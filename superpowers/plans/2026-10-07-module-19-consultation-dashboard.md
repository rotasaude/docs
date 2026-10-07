# Módulo 19 (19a) — Consulta / prontuário da APS (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** executar **depois que o módulo 18 (acolhimento) estiver em `origin/main`** do dashboard e do api. Este plano reaproveita peças que o plano do dashboard do 18 (`docs/superpowers/plans/2026-10-07-module-18-screening-dashboard.md`) cria: `src/lib/screening.ts` (`VITALS`, `parseVitals`, `vitalsFormFrom`, `bmiOf`, `alertLabel`, `screeningError`, `COLOR_LABEL`, `COLOR_TONE`, `GLUCOSE_MOMENT_LABEL`, tipos `VitalsForm`/`VitalsProblems`), `src/modules/attendance/VitalSignsFields.tsx`, os tipos `VitalSigns`/`Screening`/`CallResult`/`QueueRow.screening` de `src/lib/api.ts`, o `panel` (`ScreeningPanel`) e o `now` da `UnitQueue`, e `src/test/screeningFixtures.ts`. Se o 18 ainda não estiver lá, **pare** (Ambiente de execução, passo 1).

**Goal:** Telas do prontuário da atenção primária no painel da cidade. No atendimento chamado, o profissional vê o painel do paciente (nome de exibição, idade, sexo, problemas ativos, escuta do dia, últimas consultas) e escreve a consulta — S, O, A, P; sinais vitais com o componente do 18; problemas com avaliar, incluir, resolver e corrigir início, buscando CIAP-2 ou CID-10 por nome; condutas e tipo de atendimento de `consultation_options`; exames com busca SIGTAP; desfecho com a mesma tela de "Encerrar" — com rascunho salvo sozinho e "Finalizar" com as recusas do contrato. A consulta finalizada abre em leitura, com Adendo (motivo e mudanças) e Imprimir (o PDF numa janela nova). Fora do atendimento, a tela Prontuário abre a ficha por CPF com motivo e step-up, marcada "abertura justificada", com contagem regressiva dos 30 minutos; o `municipal_admin` vê o relatório "Aberturas fora de contexto". A validação presencial pede nome completo, social e da mãe, e o check-in oferece completar os nomes de quem já foi validado sem eles. A Produção e-SUS mostra a ficha `correction_pending`. A recepção não vê nada do prontuário: na fila, só o nome de exibição e a cor.

**Architecture:** O cliente HTTP novo vai para o fim de `src/lib/api.ts`, com os tipos copiados do arquivo de contratos do módulo 19 (e os campos das Divergências D1–D4 marcados no comentário). As regras ficam fora do React, em funções puras: `src/lib/consultation.ts` (rascunho ↔ corpo do PATCH, o que falta para finalizar, lista de problemas, início com precisão, exames, adendo, frases das recusas), `src/lib/outcome.ts` (desfecho do atendimento, extraído do `ClosePanel`), `src/lib/clinicalRecord.ts` (abertura justificada e contagem) e `src/lib/citizenNames.ts` (nomes do documento); o salvamento automático é um hook (`src/lib/useAutosave.ts`) que nunca sobrescreve o que a pessoa digitou e que "Finalizar" espera. As telas novas ficam em `src/modules/consultation/` (`CodeSearch`, `OnsetInput`, `ProblemsEditor`, `ConductsField`, `ExamRequestsField`, `ConsultationEditor`, `PatientPanel`, `ConsultationView`, `AddendumForm`, `ConsultationWorkspace`), em `src/modules/clinicalRecord/` (`OpeningForm`, `JustifiedRecord`, `OpeningsReport`) e no módulo novo `src/modules/ClinicalRecord.tsx` (item "Prontuário" do grupo Atendimento); as existentes ganham o mínimo: `UnitQueue` (botão "Consulta", consulta aberta na chamada, coluna "Nome", `ClosePanel` usando `OutcomeFields`), `Attendance` (interruptor e nomes no balcão), `CheckIn` (completar nomes), `Production` (`correction_pending`), `modules.ts`/`App.tsx` (rota), `features.ts` (`clinical_record`) e o proxy do Vite (`/clinical_record`). O dashboard nunca decide o que o api garante: as regras na tela só avisam antes de enviar.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("../lib/api")` e `vi.stubGlobal("fetch")`), Vite 5 (proxy de dev).

**Spec:** `docs/.claude/ciclo2/superpowers/specs/2026-10-07-module-19-consultation-design.md` (§7 é deste plano; §3, §4, §5 e §6 dão as regras), `docs/.claude/ciclo2/adr/0031.md` e o arquivo de contratos `docs/.claude/ciclo2/superpowers/plans/2026-10-07-module-19-consultation-contracts.md` (inteiro). O plano do api do módulo 19 precisa estar **mergeado antes** do merge deste (contratos §8); o api do 19 roda em dev na porta **3036**.

## Global Constraints

- Rotas usadas (sessão da cidade):
  - `POST /attendance/verifications` (existente) aceita `full_name` (obrigatório, 3–200), `social_name` (≤ 200), `mother_name` (≤ 200); 422 `invalid_full_name`, `invalid_social_name`, `invalid_mother_name`; `POST /attendance/lookup` devolve `citizen.names: { full_name_set, display_name }`; `POST /attendance/verifications/:id/names` `{ full_name, social_name?, mother_name? }` (`citizen_verifier`);
  - `GET /attendance/attendances/:id/record` → `<record>` (403 `out_of_context`, `missing_role`; 409 `citizen_not_verified`); `GET /clinical_record/patients/:id` → `<record>` (403 `opening_required`); `POST /clinical_record/openings` (step-up) `{ cpf, reason_code, reason_note? }` → `{ opening_id, patient_id, expires_at }` (404 `patient_not_found`, 422 `invalid_reason`); `GET /clinical_record/openings?from=&to=&user_id=` (`municipal_admin`) → `{ items }`;
  - `GET /attendance/consultation_options` → `{ care_types, conducts, cid10_allowed_for_cbo }`; `POST /attendance/attendances/:id/consultation` → 201 `<consultation>` (409 `citizen_not_verified`, `not_in_care`, `not_caller`, `already_exists`; 403 `cbo_not_allowed`, `feature_disabled`); `PATCH /attendance/consultations/:id` → `<consultation>` (409 `not_draft`; 403 `not_author`; 422 `implausible_vital` com `field`, `text_too_long` com `field`); `POST /attendance/consultations/:id/finalize` `{ outcome: <corpo do close> }` (422 `no_problem_evaluated`, `no_conduct`, `assessment_or_plan_required`, `patient_name_missing`, `cid10_not_allowed_for_cbo` e os do close); `GET /attendance/consultations/:id`; `POST /attendance/consultations/:id/addenda` `{ reason, text, changes?, opening_id? }` → 201 o adendo (409 `not_finalized`; 403 `opening_required`; 422 `invalid_reason`); `GET /attendance/consultations/:id/print` → `application/pdf` (409 `not_finalized`, `patient_name_missing`);
  - `POST /attendance/ciap2/search { q, terminology }` (módulo 18, ampliada) e `POST /attendance/sigtap/search { q }` → `{ items: [{ code, label }] }` (503 `terminology_unavailable`);
  - `GET /production` (existente): ficha com `status: "correction_pending"`, fora da contagem de pendentes;
  - propostas deste plano (Divergências): `consultation_id` no 409 `already_exists` (D1); `names` e `verification_id` no check-in (D2); `display_name` no item da fila (D3); forma de `changes` do adendo (D4).
- Erros `{ "error": "<reason>" }`; step-up 401 `{ "error": "mfa_required" }` (quem trata é o `SensitiveAction`); interruptor desligado 403 `{ "error": "feature_disabled", "feature": "clinical_record" }`; escrita devolve o objeto puro.
- Valores (contratos §1, §3): terminologia `ciap2` | `cid10`; problema `active` | `resolved`; precisão `day` | `month` | `year`; ação `evaluate` | `add` | `resolve` | `correct_onset`; acesso `in_context` | `justified`; motivo da abertura `case_review` | `active_search` | `continuity_of_care` | `other` (nota ≥ 10 em `other`); abertura válida 30 minutos; texto clínico até 20.000 por campo; motivo do adendo ≥ 10; nome de exibição = social, senão completo.
- LGPD (spec §5; ADR 0031): texto clínico (S, O, A, P, adendo, motivo, nota da abertura) e nomes **só em corpo de POST/PATCH**, nunca em URL, `console`, `localStorage`/`sessionStorage` ou mensagem de erro (as frases são montadas só com códigos e rótulos); as rotas levam só ids; o CPF da abertura vai no corpo; consultas de leitura clínica com `gcTime: 0` (o cache some quando a tela fecha); a recepção (`citizen_verifier` sem `health_professional`) não vê "Consulta", não chama nenhuma rota do prontuário e não tem o item "Prontuário" no menu — na fila, só o nome de exibição e a cor.
- Papéis: consulta e prontuário em contexto para `health_professional` com vínculo ativo na unidade (o `canCare` de `Attendance.tsx`; CBO, chamador e contexto quem confere é o api, e as recusas viram frase); abertura justificada para `health_professional`; relatório para `municipal_admin`; completar nomes para `citizen_verifier`; tudo só com `clinical_record` na sessão (`hasFeature`).
- Interface tem ciclo próprio: use os componentes e estilos existentes (`Panel`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `FrozenTextNotice`, `SensitiveAction`, `formStyles`, `VitalSignsFields`), sem redesign. Nenhum componente comum muda.
- Testes que dependem de "agora" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` (mais `setInterval`/`clearInterval` na contagem da abertura) + `vi.setSystemTime(new Date(NOW19))` (`NOW19 = "2026-10-07T10:00:00-03:00"`) e `vi.useRealTimers()` no `afterEach`. Funções puras recebem `today`/`nowMs` como argumento. Esperas (`autosaveDelayMs`, `searchDelayMs`) são props, zeradas ou altas nos testes.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

1. Confira que o módulo 18 já está em `origin/main` (pré-requisito). A partir da raiz do monorepo (`/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

   ```bash
   /opt/homebrew/bin/git -C apps/dashboard fetch origin
   /opt/homebrew/bin/git -C apps/dashboard cat-file -e origin/main:src/lib/screening.ts && echo ok18
   /opt/homebrew/bin/git -C apps/dashboard cat-file -e origin/main:src/modules/attendance/VitalSignsFields.tsx && echo ok18
   /opt/homebrew/bin/git -C apps/dashboard log --oneline -1 origin/main
   ```

   Expected: `ok18` duas vezes. Sem isso, **pare** e avise: o 18 não foi mergeado.

2. Crie o worktree e ligue o `node_modules` do checkout principal (`/.claude/` já está no `.gitignore` do dashboard):

   ```bash
   /opt/homebrew/bin/git -C apps/dashboard worktree add .claude/mod19 -b feat/mod-19-consultation origin/main
   ln -s ../../node_modules apps/dashboard/.claude/mod19/node_modules
   cd apps/dashboard/.claude/mod19 && npx vitest run 2>&1 | tail -3
   ```

   Anote a base (arquivos e testes verdes): é a de `origin/main` depois do 18 (o plano do 18 previa 125 arquivos e 1329 testes).

- Testes (script `test` = `vitest run`): `cd apps/dashboard/.claude/mod19 && npx vitest run <arquivos>`.
- Tipos antes de cada commit (script `typecheck` = `tsc --noEmit`; `noUnusedLocals` ligado, e o `tsconfig` inclui os testes): `cd apps/dashboard/.claude/mod19 && npx tsc --noEmit`.
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`.
- **Arquivos em comum com o módulo 18** (o plano foi escrito contra o desenho do 18; confira os trechos "antes" de cada diff no arquivo real e, se o 18 mudou algo no caminho, aplique a mesma intenção sobre o texto que estiver lá): `src/lib/api.ts` (o 18 acrescenta uma seção no fim, `QueueRow.screening`, `CallResult`; este plano acrescenta a seção do 19 **depois** da do 18 e campos opcionais em `AttendanceCitizen`, `CheckInCitizen`, `QueueRow`, `FichaStatus`), `src/lib/attendance.ts` (`MESSAGES`), `src/modules/attendance/UnitQueue.tsx` (o 18 cria `ScreeningPanel`, `panel`, `now`, `colorTag` e mexe em `onCall`/`onCallNext`; este plano estende a união e troca só o `ClosePanel`), `src/modules/Attendance.tsx`, `src/lib/production.ts` e `src/modules/Production.tsx` (o 18 troca `last_error` por `last_error_codes` e põe `GenerationFailures`; este plano só acrescenta o estado `correction_pending`), `src/test/recordModeFixtures.ts` (`ficha()` com `last_error_codes`). Se o 18 for rebaseado depois de este plano começar, rebaseie a branch `feat/mod-19-consultation` sobre `origin/main` (`git rebase origin/main` no worktree) e rode a suíte inteira antes de seguir.
- Os diffs abaixo têm o contexto do arquivo depois do merge do 18 e das tasks anteriores. Aplique à mão ou com `git apply` a partir de um arquivo no scratchpad (nunca dentro do worktree).
- O api do módulo 19 roda em dev na porta **3036**; a prova no navegador aponta o proxy para ela (Task 14).

## Review Focus

1. **O que a pessoa digita enquanto o rascunho está sendo salvo.** A médica escreve no Plano, o salvamento automático sai, ela continua escrevendo e clica "Finalizar" antes da próxima pausa: a resposta do PATCH não pode apagar o que veio depois, e a finalização tem de esperar o salvamento em curso e mandar o texto mais novo antes de fechar a consulta (senão a consulta imutável fica sem as últimas linhas). Testes: Task 6, "texto digitado durante o salvamento não se perde e é salvo em seguida" e "flush espera o salvamento em curso e salva o que faltava"; Task 7, "Finalizar espera o salvamento pendente e manda o rascunho mais novo antes".
2. **A abertura justificada vence com a tela aberta.** A enfermeira abriu o prontuário às 10:00 e continua lendo às 10:30, ou o relógio do servidor fecha antes e um `GET` volta 403 `opening_required`: a leitura some (dados e cache), a tela diz que terminou e oferece nova abertura, sem erro parado nem dado clínico na tela. Testes: Task 10, "a contagem chega a zero: o prontuário some e oferece nova abertura" e "403 opening_required na leitura encerra a abertura".
3. **A recepção nunca lê o prontuário.** Com `clinical_record` ligado, quem só tem `citizen_verifier` vê na fila o nome de exibição e a cor, mas nenhum "Consulta", nenhuma chamada a `/record`, e o menu não tem "Prontuário". Testes: Task 9, "recepção vê nome e cor, sem Consulta e sem ler o prontuário"; Task 10, "Prontuário no menu: só profissional ou admin, só com clinical_record, nunca operador".
4. **Problema já ativo incluído de novo.** O paciente já tem T90 ativo e a médica busca "diabetes" e escolhe T90 em "Incluir": a tela não pode mandar um `add` duplicado (o api tem índice único de ativo); vira "avaliado" e avisa. Testes: Task 2, "incluir código já ativo marca avaliado e avisa"; Task 5, "incluir T90 que já está na lista marca avaliado".
5. **Rascunho retomado.** A página recarregou no meio da consulta, ou outra aba já finalizou: "Iniciar consulta" com 409 `already_exists` retoma o rascunho existente (sem erro), e o 409 `not_draft` no salvamento trava o editor e mostra a consulta finalizada. Testes: Task 9, "já iniciada: retoma o rascunho em vez de mostrar erro"; Task 7, "not_draft no salvamento trava o editor e avisa".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `src/lib/api.ts`, `src/lib/api.consultation.test.ts`, `src/test/consultationFixtures.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `README.md` | tipos do contrato; cliente do prontuário, consulta, adendo, impresso, buscas, abertura, relatório, nomes; interruptor `clinical_record`; proxy `/clinical_record` | 1 |
| `src/lib/consultation.ts`, `src/lib/consultation.test.ts` | rascunho, finalização, problemas, início, exames, adendo, frases | 2 |
| `src/lib/outcome.ts`, `src/lib/outcome.test.ts`, `src/modules/attendance/OutcomeFields.tsx`, `src/modules/attendance/OutcomeFields.test.tsx`, `src/modules/attendance/UnitQueue.tsx` | desfecho reaproveitável; `ClosePanel` usando-o | 3 |
| `src/modules/consultation/CodeSearch.tsx` (+ teste) | busca de código por nome (CIAP-2, CID-10, SIGTAP) | 4 |
| `src/modules/consultation/OnsetInput.tsx`, `ProblemsEditor.tsx`, `ConductsField.tsx`, `ExamRequestsField.tsx` (+ testes) | itens estruturados da consulta | 5 |
| `src/lib/useAutosave.ts` (+ teste) | salvamento automático sem perder digitação | 6 |
| `src/modules/consultation/ConsultationEditor.tsx` (+ teste) | editor do rascunho e finalização com desfecho | 7 |
| `src/modules/consultation/PatientPanel.tsx`, `ConsultationView.tsx`, `AddendumForm.tsx` (+ testes) | painel do paciente; consulta finalizada; adendo; imprimir | 8 |
| `src/modules/consultation/ConsultationWorkspace.tsx` (+ teste), `src/modules/attendance/UnitQueue.tsx`, `src/modules/attendance/UnitQueue.consultation.test.tsx`, `src/modules/Attendance.tsx`, `src/modules/Attendance.test.tsx` | consulta no atendimento chamado | 9 |
| `src/lib/clinicalRecord.ts` (+ teste), `src/modules/ClinicalRecord.tsx` (+ teste), `src/modules/clinicalRecord/OpeningForm.tsx`, `src/modules/clinicalRecord/JustifiedRecord.tsx` (+ teste), `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx` | prontuário fora de contexto | 10 |
| `src/modules/clinicalRecord/OpeningsReport.tsx` (+ teste), `src/modules/ClinicalRecord.tsx` | relatório "Aberturas fora de contexto" | 11 |
| `src/lib/citizenNames.ts` (+ teste), `src/modules/attendance/NamesFields.tsx`, `src/modules/attendance/CompleteNames.tsx`, `src/modules/Attendance.tsx`, `src/modules/Attendance.test.tsx`, `src/modules/attendance/CheckIn.tsx`, `src/modules/attendance/CheckIn.test.tsx`, `src/lib/attendance.ts` | nomes na validação presencial e completar nomes | 12 |
| `src/lib/api.ts`, `src/lib/production.ts`, `src/lib/production.test.ts`, `src/modules/Production.tsx`, `src/modules/Production.test.tsx` | ficha `correction_pending` | 13 |
| — | suíte, build, revisão e prova no navegador | 14 |

**Estratégia de teste:** regras puras com tabela de casos (Tasks 2, 3, 10, 12), cliente HTTP com `fetch` falso conferindo URL, método e corpo — e que nada clínico vai na URL (Task 1), o hook de salvamento com promessas controladas pelo teste (Task 6), e cada tela com `vi.mock("../../lib/api")` cobrindo o caminho feliz, cada recusa que muda o comportamento da tela e as bordas do Review Focus. Componentes que compõem outros já testados trocam o filho por um dublê (`vi.mock("./ConsultationEditor")`) para testar só o próprio fluxo. Relógio fixo em todo teste que depende de "agora". A prova final é no navegador, contra o api do módulo 19 com a semente da spec §9 (Task 14).

---
### Task 1: Cliente HTTP do módulo 19, tipos do contrato, interruptor e proxy

**Files:**
- Modify: `src/lib/api.ts` (`AttendanceCitizen`, `verifyCitizen`, `CheckInCitizen`, `checkIn`, `QueueRow`; seção nova no fim do arquivo, depois da do módulo 18)
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `README.md`
- Create: `src/test/consultationFixtures.ts`
- Test: `src/lib/api.consultation.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `postProfessional`, `ApiError`, `ATTENDANCE_BASE`, `Sex`, `AttendanceOutcome`, `VitalSigns`, `Screening` (todos já em `api.ts`, os dois últimos do módulo 18); `revision()`, `screening()` (`src/test/screeningFixtures.ts`, do 18).
- Produces:
  - tipos: `Terminology`, `ProblemStatus`, `OnsetPrecision`, `ProblemAction`, `RecordAccess`, `OpeningReason`, `CareOutcome`, `CodedOption`, `CitizenNames`, `CitizenNamesInput`, `PatientProblem`, `EvaluatedProblem`, `ExamRequest`, `AddendumChanges`, `Addendum`, `Consultation`, `ConsultationSummary`, `RecordPatient`, `ClinicalRecord`, `ConsultationOptions`, `ConsultationDraftInput`, `OutcomeBody`, `AddendumInput`, `OpeningInput`, `Opening`, `OpeningRow`, `OpeningsQuery`;
  - campos: `AttendanceCitizen.names?: CitizenNames`; `CheckInCitizen.names?: CitizenNames` e `CheckInCitizen.verification_id?: string | null` (D2); `checkIn(...)` devolve também `verification_id?: string | null` (D2); `QueueRow.display_name?: string | null` (D3); `verifyCitizen(cpf, code, profile: VerifiedProfile & Partial<CitizenNamesInput>, extra?)`;
  - funções: `getAttendanceRecord(attendanceId): Promise<ClinicalRecord>`, `getConsultationOptions(): Promise<ConsultationOptions>`, `startConsultation(attendanceId): Promise<Consultation>`, `saveConsultationDraft(id, input: ConsultationDraftInput): Promise<Consultation>`, `finalizeConsultation(id, outcome: OutcomeBody): Promise<Consultation>`, `getConsultation(id): Promise<Consultation>`, `addAddendum(id, input: AddendumInput): Promise<Addendum>`, `fetchConsultationPdf(id): Promise<Blob>`, `searchTerminology(q, terminology: Terminology): Promise<CodedOption[]>`, `searchSigtap(q): Promise<CodedOption[]>`, `openClinicalRecord(input: OpeningInput): Promise<Opening>`, `getJustifiedRecord(patientId): Promise<ClinicalRecord>`, `listOpenings(q: OpeningsQuery): Promise<OpeningRow[]>`, `completeCitizenNames(verificationId, names: CitizenNamesInput): Promise<void>`;
  - `FeatureKey` com `"clinical_record"` (rótulo "Prontuário da atenção primária");
  - fixtures: `NOW19`, `TODAY19`, `problem()`, `summary()`, `record()`, `consultation()`, `finalized()`, `options()`, `opening()`, `openingRow()`; a sessão dos testes é `sessionWith(roles, { id: "us1", features: [ "clinical_record" ] })` (`src/test/campaignFixtures.tsx`), e a autora das fixtures é `us1`.

- [ ] **Step 1: Write the failing test**

Fixtures (usadas nesta e nas próximas tasks):

```ts
// src/test/consultationFixtures.ts
// Dados comuns aos testes do módulo 19 (consulta). Relógio dos testes:
// quarta, 2026-10-07 10:00 em São Paulo (-03:00). Os códigos de tipo de
// atendimento e de conduta são ilustrativos: os reais vêm do api
// (consultation_options, fixados pela Task 1 do plano do api).
import type {
  ClinicalRecord, Consultation, ConsultationOptions, ConsultationSummary, Opening, OpeningRow, PatientProblem
} from "../lib/api";
import { revision, screening } from "./screeningFixtures";

export const NOW19 = "2026-10-07T10:00:00-03:00";
export const TODAY19 = "2026-10-07";

export function problem(over: Partial<PatientProblem> = {}): PatientProblem {
  return {
    id: "pp1", terminology: "ciap2", code: "T90", label: "Diabetes não insulino-dependente", status: "active",
    onset_on: "2019-03-01", onset_precision: "month", resolved_on: null, ...over
  };
}

export function summary(over: Partial<ConsultationSummary> = {}): ConsultationSummary {
  return {
    id: "cs0", finalized_at: "2026-09-10T14:30:00-03:00", author_name: "Dra. Helena Prado",
    cbo_label: "Médico da estratégia de saúde da família", care_type_label: "Consulta agendada",
    problems: [ { problem_id: "pp1", terminology: "ciap2", code: "T90", label: "Diabetes não insulino-dependente", action: "evaluate" } ],
    addenda_count: 1, ...over
  };
}

export function record(over: Partial<ClinicalRecord> = {}): ClinicalRecord {
  return {
    patient: { id: "pa1", display_name: "Joana Lima", full_name: "João Carlos Lima", social_name: "Joana Lima", age: 54,
      sex: "female", cpf_masked: "***.982.247-**" },
    access: "in_context",
    problems: [ problem() ],
    today_screening: screening({ status: "completed", destination: "same_day", current_revision: revision(), revisions_count: 1 }),
    consultations: [ summary() ],
    ...over
  };
}

export function consultation(over: Partial<Consultation> = {}): Consultation {
  return {
    id: "cs1", attendance_id: "a1", patient_id: "pa1", status: "draft", author: { id: "us1", name: "Dra. Helena Prado" },
    cbo_code: "225142", subjective: "", objective: "", assessment: "", plan: "", vitals: {}, care_type: "5",
    evaluated_problems: [], conducts: [], exam_requests: [], started_at: "2026-10-07T09:55:00-03:00", finalized_at: null,
    addenda: [], ...over
  };
}

export function finalized(over: Partial<Consultation> = {}): Consultation {
  return consultation({
    status: "finalized", finalized_at: "2026-10-07T10:20:00-03:00",
    subjective: "Refere sede e cansaço há duas semanas.", objective: "Bom estado geral.",
    assessment: "Diabetes descompensado.", plan: "Ajuste de dose; retorno em 30 dias.",
    vitals: { systolic: 150, diastolic: 95, capillary_glucose: 280, glucose_moment: "random" },
    evaluated_problems: [ { problem_id: "pp1", terminology: "ciap2", code: "T90", label: "Diabetes não insulino-dependente", action: "evaluate" } ],
    conducts: [ "9" ],
    exam_requests: [ { sigtap_code: "0202010503", label: "DOSAGEM DE HEMOGLOBINA GLICOSILADA", cid10_justification: "E119" } ],
    ...over
  });
}

export function options(over: Partial<ConsultationOptions> = {}): ConsultationOptions {
  return {
    care_types: [ { code: "2", label: "Consulta agendada" }, { code: "5", label: "Consulta no dia" } ],
    conducts: [ { code: "9", label: "Retorno para consulta agendada" }, { code: "12", label: "Alta do episódio" } ],
    cid10_allowed_for_cbo: true, ...over
  };
}

export function opening(over: Partial<Opening> = {}): Opening {
  return { opening_id: "op1", patient_id: "pa1", expires_at: "2026-10-07T10:30:00-03:00", ...over };
}

export function openingRow(over: Partial<OpeningRow> = {}): OpeningRow {
  return {
    id: "op1", user_name: "Enf. Lúcia Prado", cpf_masked: "***.982.247-**", reason_code: "case_review",
    created_at: "2026-10-06T15:10:00-03:00", expires_at: "2026-10-06T15:40:00-03:00", ...over
  };
}
```

Teste do cliente:

```ts
// src/lib/api.consultation.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import {
  ApiError, addAddendum, completeCitizenNames, fetchConsultationPdf, finalizeConsultation, getAttendanceRecord,
  getConsultation, getConsultationOptions, getJustifiedRecord, listOpenings, openClinicalRecord, saveConsultationDraft,
  searchSigtap, searchTerminology, startConsultation, verifyCitizen
} from "./api";
import { consultation, finalized, opening, openingRow, options, record } from "../test/consultationFixtures";

afterEach(() => vi.unstubAllGlobals());

function stub(body: unknown, status = 200) {
  const fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
    new Response(body === undefined ? null : JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fn);
  return fn;
}
const call = (fn: ReturnType<typeof stub>, i = 0) => fn.mock.calls[i] as [ string, RequestInit ];
const sent = (fn: ReturnType<typeof stub>, i = 0) => JSON.parse(call(fn, i)[1].body as string);
const draftInput = {
  subjective: "dor no peito", objective: "", assessment: "", plan: "", vitals: { systolic: 150, diastolic: 95 },
  care_type: "5", evaluated_problems: [], conducts: [ "9" ], exam_requests: []
};

describe("cliente do módulo 19 — prontuário e consulta", () => {
  it("prontuário em contexto e opções da consulta são GET com só o id na URL", async () => {
    let fn = stub(record());
    expect((await getAttendanceRecord("a/1")).patient.display_name).toBe("Joana Lima");
    expect(call(fn)[0]).toBe("/attendance/attendances/a%2F1/record");
    expect(call(fn)[1].method).toBeUndefined();
    expect(call(fn)[1].credentials).toBe("include");

    fn = stub(options());
    expect((await getConsultationOptions()).cid10_allowed_for_cbo).toBe(true);
    expect(call(fn)[0]).toBe("/attendance/consultation_options");
  });

  it("iniciar, ler, salvar e finalizar", async () => {
    let fn = stub(consultation(), 201);
    expect((await startConsultation("a1")).status).toBe("draft");
    expect(call(fn)[0]).toBe("/attendance/attendances/a1/consultation");
    expect(call(fn)[1].method).toBe("POST");

    fn = stub(consultation());
    await getConsultation("cs1");
    expect(call(fn)[0]).toBe("/attendance/consultations/cs1");
    expect(call(fn)[1].method).toBeUndefined();

    fn = stub(consultation({ subjective: "dor no peito" }));
    await saveConsultationDraft("cs1", draftInput);
    expect(call(fn)[0]).toBe("/attendance/consultations/cs1");
    expect(call(fn)[1].method).toBe("PATCH");
    expect(sent(fn)).toEqual(draftInput);

    fn = stub(finalized());
    await finalizeConsultation("cs1", { outcome: "referred", referral_unit_id: "u2" });
    expect(call(fn)[0]).toBe("/attendance/consultations/cs1/finalize");
    expect(sent(fn)).toEqual({ outcome: { outcome: "referred", referral_unit_id: "u2" } });
  });

  it("adendo manda motivo, texto, mudanças e a abertura no corpo", async () => {
    const fn = stub({ id: "ad1", author_name: "Enf. Lúcia Prado", created_at: "2026-10-07T10:40:00-03:00",
      reason: "correção do plano", text: "Retorno em 15 dias.", changes: { conducts: [ "12" ] } }, 201);
    const out = await addAddendum("cs1", { reason: "correção do plano", text: "Retorno em 15 dias.",
      changes: { conducts: [ "12" ] }, opening_id: "op1" });
    expect(out.id).toBe("ad1");
    expect(call(fn)[0]).toBe("/attendance/consultations/cs1/addenda");
    expect(sent(fn)).toEqual({ reason: "correção do plano", text: "Retorno em 15 dias.", changes: { conducts: [ "12" ] }, opening_id: "op1" });
  });

  it("impresso: pede PDF com a sessão e devolve o arquivo; 409 vira ApiError com o código", async () => {
    let fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
      new Response("%PDF-1.4", { status: 200, headers: { "Content-Type": "application/pdf" } }));
    vi.stubGlobal("fetch", fn);
    const blob = await fetchConsultationPdf("cs1");
    expect(blob.size).toBe(8);
    expect(fn.mock.calls[0][0]).toBe("/attendance/consultations/cs1/print");
    expect((fn.mock.calls[0][1] as RequestInit).credentials).toBe("include");
    expect(((fn.mock.calls[0][1] as RequestInit).headers as Record<string, string>).Accept).toBe("application/pdf");

    fn = stub({ error: "patient_name_missing" }, 409);
    const err = await fetchConsultationPdf("cs1").catch((e: unknown) => e);
    expect(err).toBeInstanceOf(ApiError);
    expect((err as ApiError).body).toEqual({ error: "patient_name_missing" });
  });
});

describe("cliente do módulo 19 — buscas de terminologia", () => {
  it("CIAP-2 e CID-10 pela rota do módulo 18, com a terminologia no corpo", async () => {
    const fn = stub({ items: [ { code: "E11", label: "Diabetes mellitus não insulino-dependente" } ] });
    expect((await searchTerminology("diabetes", "cid10"))[0].code).toBe("E11");
    expect(call(fn)[0]).toBe("/attendance/ciap2/search");
    expect(sent(fn)).toEqual({ q: "diabetes", terminology: "cid10" });
  });

  it("SIGTAP no corpo, nunca na URL", async () => {
    const fn = stub({ items: [ { code: "0202010503", label: "DOSAGEM DE HEMOGLOBINA GLICOSILADA" } ] });
    expect((await searchSigtap("hemoglobina glicada"))[0].code).toBe("0202010503");
    expect(call(fn)[0]).toBe("/attendance/sigtap/search");
    expect(sent(fn)).toEqual({ q: "hemoglobina glicada" });
  });
});

describe("cliente do módulo 19 — abertura justificada e relatório", () => {
  it("abrir manda CPF e motivo no corpo; ler pelo id do paciente", async () => {
    let fn = stub(opening(), 201);
    const out = await openClinicalRecord({ cpf: "529.982.247-25", reason_code: "other", reason_note: "revisão pedida pela equipe" });
    expect(out.opening_id).toBe("op1");
    expect(call(fn)[0]).toBe("/clinical_record/openings");
    expect(sent(fn)).toEqual({ cpf: "529.982.247-25", reason_code: "other", reason_note: "revisão pedida pela equipe" });

    fn = stub(record({ access: "justified" }));
    expect((await getJustifiedRecord("pa1")).access).toBe("justified");
    expect(call(fn)[0]).toBe("/clinical_record/patients/pa1");
  });

  it("relatório filtra por período e, quando escolhido, por profissional", async () => {
    let fn = stub({ items: [ openingRow() ] });
    expect(await listOpenings({ from: "2026-09-07", to: "2026-10-07" })).toHaveLength(1);
    expect(call(fn)[0]).toBe("/clinical_record/openings?from=2026-09-07&to=2026-10-07");

    fn = stub({ items: [] });
    await listOpenings({ from: "2026-09-07", to: "2026-10-07", userId: "us9" });
    expect(call(fn)[0]).toBe("/clinical_record/openings?from=2026-09-07&to=2026-10-07&user_id=us9");
  });
});

describe("cliente do módulo 19 — nomes na validação presencial", () => {
  it("validar manda os nomes junto do perfil conferido", async () => {
    const fn = stub({ verification: { id: "v1" } }, 201);
    await verifyCitizen("529.982.247-25", "123456",
      { birth_date: "1970-01-02", sex: "female", gender_identity: null, full_name: "Joana Lima", mother_name: "Maria Lima" });
    expect(sent(fn)).toEqual({ cpf: "529.982.247-25", code: "123456", document_checked: true, birth_date: "1970-01-02",
      sex: "female", gender_identity: null, full_name: "Joana Lima", mother_name: "Maria Lima" });
  });

  it("completar nomes de par já validado pela validação ativa", async () => {
    const fn = stub({ id: "c1" });
    await completeCitizenNames("v1", { full_name: "João Carlos Lima", social_name: "Joana Lima" });
    expect(call(fn)[0]).toBe("/attendance/verifications/v1/names");
    expect(call(fn)[1].method).toBe("POST");
    expect(sent(fn)).toEqual({ full_name: "João Carlos Lima", social_name: "Joana Lima" });
  });
});
```

E o rótulo do interruptor, em `src/lib/features.test.ts`:

```diff
--- a/src/lib/features.test.ts
+++ b/src/lib/features.test.ts
@@ -31,6 +31,8 @@ describe("features da sessão (contratos §1)", () => {
   it("rótulo conhecido em português; desconhecido sai como a chave", () => {
     expect(featureLabel("ledi_export")).toBe("Envio da produção ao e-SUS (LEDI)");
     expect(featureLabel("cadsus_lookup")).toBe("Consulta ao CADSUS na validação presencial");
+    expect(featureLabel("clinical_record")).toBe("Prontuário da atenção primária");
+    expect(hasFeature({ features: [ "clinical_record" ] }, "clinical_record")).toBe(true);
     expect(featureLabel("rnds_sync")).toBe("rnds_sync");
   });
 });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/api.consultation.test.ts src/lib/features.test.ts`
Expected: FAIL — `getAttendanceRecord is not a function` (e as outras funções novas) e `expected 'clinical_record' to be 'Prontuário da atenção primária'`.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`, os campos opcionais nos tipos existentes:

```diff
--- a/src/lib/api.ts
+++ b/src/lib/api.ts
@@ export interface AttendanceCitizen {
   // Módulo 15 (contratos §4.4): o perfil declarado do par, para o atendente
   // confirmar ou corrigir. Opcional porque uma API anterior omite a chave.
   profile?: CitizenProfile | null;
+  // Módulo 19 (contratos §2): se o nome completo já foi conferido e o nome de
+  // exibição; nunca os três nomes. Opcional: api anterior não manda.
+  names?: CitizenNames;
 }
@@ export interface VerifyExtra { cadsus_confirmed?: boolean }
 
 export async function verifyCitizen(
-  cpf: string, code: string, profile: VerifiedProfile, extra: VerifyExtra = {}
+  cpf: string, code: string, profile: VerifiedProfile & Partial<CitizenNamesInput>, extra: VerifyExtra = {}
 ): Promise<void> {
@@ export interface CheckInCitizen {
   id: string; cpf_masked: string; phone_masked: string;
   verification_level: "declared" | "verified";
+  // Módulo 19, Divergência D2: nomes e a validação ativa do par validado, para
+  // a recepção completar os nomes no check-in. Opcionais: api sem a D2 não manda.
+  names?: CitizenNames; verification_id?: string | null;
 }
@@ export async function checkIn(
   cpf: string, code: string, healthUnitId: string, documentChecked: boolean
-): Promise<{ attendance: Attendance; verified: boolean }> {
+): Promise<{ attendance: Attendance; verified: boolean; verification_id?: string | null }> {
@@ export interface QueueRow {
   reference_unit_ids?: string[];
+  // Módulo 19, Divergência D3: nome de exibição (social, senão completo) do
+  // paciente; a recepção vê só ele e a cor. Ausente em api sem a D3.
+  display_name?: string | null;
```

(`QueueRow` já tem a linha `screening?: QueueScreening | null;` do módulo 18 logo abaixo de `reference_unit_ids`; o campo novo entra entre as duas.)

E a seção nova no **fim** do arquivo, depois da seção do módulo 18:

```ts
// ─── Prontuário da APS (módulo 19a, ADR 0031; contratos do módulo 19) ──────
// Texto clínico (S, O, A, P, adendo, motivo, nota da abertura) e nomes só em
// corpo de POST/PATCH, nunca em URL: as rotas levam só ids, e o CPF da
// abertura vai no corpo. A recepção não chama nada daqui.

const CLINICAL_RECORD_BASE = import.meta.env.VITE_CLINICAL_RECORD_BASE || "/clinical_record";

export type Terminology = "ciap2" | "cid10";
export type ProblemStatus = "active" | "resolved";
export type OnsetPrecision = "day" | "month" | "year";
export type ProblemAction = "evaluate" | "add" | "resolve" | "correct_onset";
export type RecordAccess = "in_context" | "justified";
export type OpeningReason = "case_review" | "active_search" | "continuity_of_care" | "other";
export type CareOutcome = Exclude<AttendanceOutcome, "left">;

export interface CodedOption { code: string; label: string }
export interface CitizenNames { full_name_set: boolean; display_name: string | null }
export interface CitizenNamesInput { full_name: string; social_name?: string; mother_name?: string }

export interface PatientProblem {
  id: string; terminology: Terminology; code: string; label: string; status: ProblemStatus;
  onset_on: string | null; onset_precision: OnsetPrecision | null; resolved_on: string | null;
}
export interface EvaluatedProblem {
  problem_id: string | null; terminology: Terminology; code: string; label: string; action: ProblemAction;
  onset_on?: string | null; onset_precision?: OnsetPrecision | null;
}
export interface ExamRequest { sigtap_code: string; label: string; cid10_justification?: string | null }
// Divergência D4: `evaluated_problems` são eventos novos (mesma forma da
// consulta); `conducts` e `exam_requests` são as listas finais.
export interface AddendumChanges { evaluated_problems?: EvaluatedProblem[]; conducts?: string[]; exam_requests?: ExamRequest[] }
export interface Addendum { id: string; author_name: string; created_at: string; reason: string; text: string; changes: AddendumChanges | null }

export interface Consultation {
  id: string; attendance_id: string; patient_id: string; status: "draft" | "finalized";
  author: { id: string; name: string }; cbo_code: string;
  subjective: string | null; objective: string | null; assessment: string | null; plan: string | null;
  vitals: VitalSigns; care_type: string | null;
  evaluated_problems: EvaluatedProblem[]; conducts: string[]; exam_requests: ExamRequest[];
  started_at: string; finalized_at: string | null; addenda: Addendum[];
}
export interface ConsultationSummary {
  id: string; finalized_at: string; author_name: string; cbo_label: string; care_type_label: string;
  problems: EvaluatedProblem[]; addenda_count: number;
}
export interface RecordPatient {
  id: string; display_name: string; full_name: string; social_name: string | null; age: number; sex: Sex; cpf_masked: string;
}
export interface ClinicalRecord {
  patient: RecordPatient; access: RecordAccess; problems: PatientProblem[];
  today_screening: Screening | null; consultations: ConsultationSummary[];
}
export interface ConsultationOptions { care_types: CodedOption[]; conducts: CodedOption[]; cid10_allowed_for_cbo: boolean }
// Corpo do autosave (Divergência D5: todos os campos editáveis a cada PATCH).
export interface ConsultationDraftInput {
  subjective: string; objective: string; assessment: string; plan: string; vitals: VitalSigns; care_type: string | null;
  evaluated_problems: EvaluatedProblem[]; conducts: string[]; exam_requests: ExamRequest[];
}
// O mesmo corpo do POST /attendance/attendances/:id/close.
export interface OutcomeBody { outcome: CareOutcome; referral_unit_id?: string; referral_note?: string }
export interface AddendumInput { reason: string; text: string; changes?: AddendumChanges; opening_id?: string }
export interface OpeningInput { cpf: string; reason_code: OpeningReason; reason_note?: string }
export interface Opening { opening_id: string; patient_id: string; expires_at: string }
export interface OpeningRow {
  id: string; user_name: string; cpf_masked: string; reason_code: OpeningReason; created_at: string; expires_at: string;
}
export interface OpeningsQuery { from: string; to: string; userId?: string }

const attendancePath = (id: string, action: string) => `${ATTENDANCE_BASE}/attendances/${encodeURIComponent(id)}/${action}`;
const consultationPath = (id: string, action?: string) =>
  `${ATTENDANCE_BASE}/consultations/${encodeURIComponent(id)}${action ? `/${action}` : ""}`;

// Ler gera a trilha `clinical_record.viewed` no api.
export function getAttendanceRecord(attendanceId: string): Promise<ClinicalRecord> {
  return jsonFetch(attendancePath(attendanceId, "record"));
}

export function getConsultationOptions(): Promise<ConsultationOptions> {
  return jsonFetch(`${ATTENDANCE_BASE}/consultation_options`);
}

export function startConsultation(attendanceId: string): Promise<Consultation> {
  return jsonFetch(attendancePath(attendanceId, "consultation"), postProfessional({}));
}

export function saveConsultationDraft(id: string, input: ConsultationDraftInput): Promise<Consultation> {
  return jsonFetch(consultationPath(id), { method: "PATCH", body: JSON.stringify(input) });
}

export function finalizeConsultation(id: string, outcome: OutcomeBody): Promise<Consultation> {
  return jsonFetch(consultationPath(id, "finalize"), postProfessional({ outcome }));
}

export function getConsultation(id: string): Promise<Consultation> {
  return jsonFetch(consultationPath(id));
}

export function addAddendum(id: string, input: AddendumInput): Promise<Addendum> {
  return jsonFetch(consultationPath(id, "addenda"), postProfessional(input));
}

// O PDF é gerado na hora (não gravado). Lido com fetch, e não por navegação,
// para a tela mostrar o 409 (`patient_name_missing`, `not_finalized`) em vez
// de uma página de erro na janela nova.
export async function fetchConsultationPdf(id: string): Promise<Blob> {
  const url = consultationPath(id, "print");
  const res = await fetch(url, { credentials: "include", headers: { Accept: "application/pdf" } });
  if (!res.ok) {
    const text = await res.text().catch(() => "");
    let body: unknown = text;
    if (text) { try { body = JSON.parse(text); } catch { /* deixa string */ } }
    throw new ApiError(res.status, body, `${res.status} on ${url}`);
  }
  return res.blob();
}

// A rota de CIAP-2 do módulo 18, ampliada para CID-10 (contratos §5).
export async function searchTerminology(q: string, terminology: Terminology): Promise<CodedOption[]> {
  const payload = await jsonFetch<{ items: CodedOption[] }>(`${ATTENDANCE_BASE}/ciap2/search`, postProfessional({ q, terminology }));
  return payload.items;
}

export async function searchSigtap(q: string): Promise<CodedOption[]> {
  const payload = await jsonFetch<{ items: CodedOption[] }>(`${ATTENDANCE_BASE}/sigtap/search`, postProfessional({ q }));
  return payload.items;
}

// Step-up: quem trata 401 mfa_required é o SensitiveAction.
export function openClinicalRecord(input: OpeningInput): Promise<Opening> {
  return jsonFetch(`${CLINICAL_RECORD_BASE}/openings`, postProfessional(input));
}

export function getJustifiedRecord(patientId: string): Promise<ClinicalRecord> {
  return jsonFetch(`${CLINICAL_RECORD_BASE}/patients/${encodeURIComponent(patientId)}`);
}

// Só datas e o id do profissional na URL; o relatório já vem com CPF mascarado.
export async function listOpenings(q: OpeningsQuery): Promise<OpeningRow[]> {
  const params = new URLSearchParams({ from: q.from, to: q.to });
  if (q.userId) params.set("user_id", q.userId);
  const payload = await jsonFetch<{ items: OpeningRow[] }>(`${CLINICAL_RECORD_BASE}/openings?${params.toString()}`);
  return payload.items;
}

// `:id` é a validação ativa do par (Divergência D2).
export async function completeCitizenNames(verificationId: string, names: CitizenNamesInput): Promise<void> {
  await jsonFetch<unknown>(`${ATTENDANCE_BASE}/verifications/${encodeURIComponent(verificationId)}/names`, postProfessional(names));
}
```

Em `src/lib/features.ts`:

```diff
--- a/src/lib/features.ts
+++ b/src/lib/features.ts
@@ -5,12 +5,13 @@
 // Só o maintenance liga e desliga; o dashboard nunca escreve interruptor.
 import { ApiError } from "./api";
 
-export type FeatureKey = "ledi_export" | "cadsus_lookup";
+export type FeatureKey = "ledi_export" | "cadsus_lookup" | "clinical_record";
 
 export const FEATURE_DISABLED_MESSAGE = "esta funcionalidade está desligada para a cidade";
 
 const FEATURE_LABEL: Record<string, string> = {
   ledi_export: "Envio da produção ao e-SUS (LEDI)",
-  cadsus_lookup: "Consulta ao CADSUS na validação presencial"
+  cadsus_lookup: "Consulta ao CADSUS na validação presencial",
+  clinical_record: "Prontuário da atenção primária"
 };
```

Em `vite.config.ts` (rota nova do prontuário fora do atendimento):

```diff
--- a/vite.config.ts
+++ b/vite.config.ts
@@ -17,6 +17,7 @@ import react from "@vitejs/plugin-react";
 //   /triage_catalog → catálogo de triagens da cidade (módulo 15; leitura para papéis de protocolo, escrita do municipal_admin).
 //   /integrations, /cnes, /production → módulo 16 (credenciais, CNES e produção e-SUS).
+//   /clinical_record → prontuário fora do atendimento (módulo 19: abertura justificada e relatório).
 //
@@ -50,7 +51,8 @@ export default defineConfig({
       "/integrations": proxy(TARGET),
       "/cnes": proxy(TARGET),
-      "/production": proxy(TARGET)
+      "/production": proxy(TARGET),
+      "/clinical_record": proxy(TARGET)
     }
   }
 });
```

Em `README.md`:

```diff
--- a/README.md
+++ b/README.md
@@ -105,7 +105,7 @@ Fora do Docker:
 O Vite proxa `/up`, `/admin/api`, `/authoring`, `/session`, `/passwords`,
 `/auth`, `/setup`, `/protocols`, `/mfa`, `/attendance`, `/professionals`, `/territory`, `/campaigns`, `/triage_catalog`,
-`/integrations`, `/cnes` e `/production` para
+`/integrations`, `/cnes`, `/production` e `/clinical_record` para
 `VITE_API_PROXY_TARGET` com `changeOrigin: false`. **Não troque para `true`**:
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/api.consultation.test.ts src/lib/features.test.ts && npx tsc --noEmit`
Expected: PASS (10 testes no cliente; `features.test.ts` verde); `tsc` limpo (o `Counter` continua mandando só o perfil, que cabe em `VerifiedProfile & Partial<CitizenNamesInput>`).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/lib/api.ts src/lib/api.consultation.test.ts src/test/consultationFixtures.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts README.md
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "feat: add the clinical record api client and contract types

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 2: Regras puras da consulta e frases das recusas

**Files:**
- Create: `src/lib/consultation.ts`
- Test: `src/lib/consultation.test.ts`

**Interfaces:**
- Consumes: tipos e `ApiError` da Task 1; `parseVitals`, `vitalsFormFrom`, `screeningError`, `VitalsForm`, `VitalsProblems` (`src/lib/screening.ts`, módulo 18); `fmtDay` (`src/lib/audiencePhrase.ts`).
- Produces (`src/lib/consultation.ts`):
  - chaves de cache `RECORD_KEY = "attendanceRecord"`, `CONSULTATION_KEY = "consultation"`, `OPTIONS_KEY = "consultationOptions"`, `JUSTIFIED_KEY = "justifiedRecord"`;
  - `TEXT_MAX = 20_000`, `TEXT_MAX_LABEL = "20.000"`, `ADDENDUM_REASON_MIN = 10`; `type SoapField`; `SOAP_FIELDS: { key; label }[]` ("Subjetivo (S)", "Objetivo (O)", "Avaliação (A)", "Plano (P)"); `textLabel(field)`; `TERMINOLOGY_LABEL`; `ACTION_LABEL` (`avaliado`, `incluído`, `resolvido`, `início corrigido`); `PRECISIONS`; `PRECISION_LABEL`;
  - `interface Onset { onset_on: string; onset_precision: OnsetPrecision }`; `interface ConsultationDraft { soap: Record<SoapField, string>; vitals: VitalsForm; careType: string; problems: EvaluatedProblem[]; conducts: string[]; exams: ExamRequest[] }`; `draftFrom(c)`; `interface DraftCheck { input: ConsultationDraftInput; vitalsProblems: VitalsProblems; tooLong: SoapField[] }`; `checkDraft(d)`; `blockedReason(check): string | null`; `finalizeProblems(check): string[]`;
  - `ageLabel(age)`, `codedLabel(list, code)`;
  - problemas: `problemKey(item)`, `markProblem(items, problem, "evaluate" | "resolve")`, `correctOnset(items, problem, onset)`, `addProblem(items, patientProblems, terminology, ref): { items; notice }`, `setItemOnset(items, key, onset)`, `removeItem(items, key)`;
  - início: `parseOnset(precision, text, today): Onset | { problem: string }`, `onsetLabel(on, precision)`, `onsetInputValue(on, precision)`;
  - exames: `normalizeCid10(text)`, `cid10Problem(text)`, `addExam(exams, ref)`, `setJustification(exams, code, text)`, `examsProblem(exams)`;
  - adendo: `addendumProblem(reason, text)`, `interface AddendumEdit { problems; conducts; exams }`, `addendumChanges(base, edit): AddendumChanges | undefined`, `changesLines(changes, conductLabel): string[]`;
  - recusas: `consultationError(err): string`, `existingConsultationId(err): string | null` (Divergência D1).

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/consultation.test.ts
import { describe, expect, it } from "vitest";
import { ApiError, type EvaluatedProblem } from "./api";
import {
  addExam, addProblem, addendumChanges, addendumProblem, ageLabel, blockedReason, changesLines, checkDraft, cid10Problem,
  codedLabel, consultationError, correctOnset, draftFrom, examsProblem, existingConsultationId, finalizeProblems, markProblem,
  normalizeCid10, onsetInputValue, onsetLabel, parseOnset, problemKey, removeItem, setItemOnset, setJustification
} from "./consultation";
import { consultation, finalized, options, problem } from "../test/consultationFixtures";

const T90 = problem();
const K86 = problem({ id: "pp2", code: "K86", label: "Hipertensão sem complicações", onset_on: null, onset_precision: null });
const err = (status: number, body: unknown) => new ApiError(status, body, String(status));

describe("rascunho ↔ corpo do PATCH", () => {
  it("draftFrom e checkDraft fazem o caminho de volta, com vírgula decimal e tipo vazio como null", () => {
    const d = draftFrom(consultation({ subjective: null, vitals: { temperature_c: 37.8 }, care_type: null }));
    expect(d.soap.subjective).toBe("");
    expect(d.vitals.temperature_c).toBe("37,8");
    const check = checkDraft(d);
    expect(check.input.vitals).toEqual({ temperature_c: 37.8 });
    expect(check.input.care_type).toBeNull();
    expect(check.tooLong).toEqual([]);
  });

  it("sinal fora do plausível fica fora do corpo e aparece em vitalsProblems", () => {
    const d = draftFrom(consultation());
    const check = checkDraft({ ...d, vitals: { ...d.vitals, systolic: "400", diastolic: "90" } });
    expect(check.input.vitals.systolic).toBeUndefined();
    expect(check.vitalsProblems.systolic).toBe("use de 50 a 300 mmHg");
  });

  it("texto acima de 20.000 bloqueia o salvamento e diz qual campo", () => {
    const d = draftFrom(consultation());
    const check = checkDraft({ ...d, soap: { ...d.soap, plan: "x".repeat(20_001) } });
    expect(check.tooLong).toEqual([ "plan" ]);
    expect(blockedReason(check)).toBe("Plano (P) passa de 20.000 caracteres");
    expect(blockedReason(checkDraft(d))).toBeNull();
  });

  it("justificativa do exame vai normalizada; vazia sai do corpo", () => {
    const d = draftFrom(consultation({ exam_requests: [
      { sigtap_code: "0202010503", label: "HbA1c", cid10_justification: "e11.9" },
      { sigtap_code: "0202010295", label: "Glicose", cid10_justification: "" }
    ] }));
    expect(checkDraft(d).input.exam_requests).toEqual([
      { sigtap_code: "0202010503", label: "HbA1c", cid10_justification: "E119" },
      { sigtap_code: "0202010295", label: "Glicose" }
    ]);
  });
});

describe("o que falta para finalizar (espelho dos 422)", () => {
  it("rascunho vazio pede problema, conduta e A ou P, nessa ordem", () => {
    expect(finalizeProblems(checkDraft(draftFrom(consultation())))).toEqual([
      "avalie, inclua ou resolva ao menos um problema",
      "marque ao menos uma conduta",
      "escreva a avaliação (A) ou o plano (P)"
    ]);
  });

  it("só o plano basta; sinais com problema e CID-10 inválido também travam", () => {
    const base = draftFrom(consultation({ plan: "retorno em 30 dias", conducts: [ "9" ],
      evaluated_problems: [ { problem_id: "pp1", terminology: "ciap2", code: "T90", label: "Diabetes", action: "evaluate" } ] }));
    expect(finalizeProblems(checkDraft(base))).toEqual([]);
    const bad = { ...base, vitals: { ...base.vitals, systolic: "150" },
      exams: [ { sigtap_code: "0202010503", label: "HbA1c", cid10_justification: "E1" } ] };
    expect(finalizeProblems(checkDraft(bad))).toEqual([
      "corrija os sinais vitais marcados", "confira o CID-10 da justificativa do exame 0202010503"
    ]);
  });
});

describe("lista de problemas", () => {
  it("avaliar e resolver trocam a ação do mesmo problema, sem duplicar", () => {
    let items = markProblem([], T90, "evaluate");
    items = markProblem(items, T90, "resolve");
    expect(items).toEqual([ { problem_id: "pp1", terminology: "ciap2", code: "T90", label: T90.label, action: "resolve" } ]);
  });

  it("corrigir início leva a data e a precisão", () => {
    const items = correctOnset([], K86, { onset_on: "2018-01-01", onset_precision: "year" });
    expect(items[0]).toMatchObject({ problem_id: "pp2", action: "correct_onset", onset_on: "2018-01-01", onset_precision: "year" });
  });

  it("incluir código novo vira add", () => {
    const out = addProblem([], [ T90 ], "ciap2", { code: "K86", label: "Hipertensão sem complicações" });
    expect(out.notice).toBeNull();
    expect(out.items).toEqual([ { problem_id: null, terminology: "ciap2", code: "K86", label: "Hipertensão sem complicações", action: "add" } ]);
  });

  it("incluir código já ativo marca avaliado e avisa", () => {
    const out = addProblem([], [ T90 ], "ciap2", { code: "T90", label: "Diabetes não insulino-dependente" });
    expect(out.items).toEqual([ { problem_id: "pp1", terminology: "ciap2", code: "T90", label: T90.label, action: "evaluate" } ]);
    expect(out.notice).toBe("T90 já está na lista do paciente — marcado como avaliado");
    const again = addProblem(out.items, [ T90 ], "ciap2", { code: "T90", label: T90.label });
    expect(again.items).toBe(out.items);
    expect(again.notice).toBe("T90 já está na lista do paciente e já foi marcado nesta consulta");
  });

  it("mesmo código em outra terminologia, ou resolvido, entra como add; repetido na consulta não duplica", () => {
    expect(addProblem([], [ T90 ], "cid10", { code: "T90", label: "Sequelas de traumatismo" }).items[0].action).toBe("add");
    const resolved = problem({ status: "resolved" });
    expect(addProblem([], [ resolved ], "ciap2", { code: "T90", label: T90.label }).items[0].action).toBe("add");
    const once = addProblem([], [], "ciap2", { code: "K86", label: "Hipertensão" }).items;
    const twice = addProblem(once, [], "ciap2", { code: "K86", label: "Hipertensão" });
    expect(twice.items).toBe(once);
    expect(twice.notice).toBe("K86 já foi incluído nesta consulta");
  });

  it("chave, início do incluído e remover", () => {
    const items: EvaluatedProblem[] = [
      ...markProblem([], T90, "evaluate"),
      { problem_id: null, terminology: "ciap2", code: "K86", label: "Hipertensão", action: "add" }
    ];
    expect(items.map(problemKey)).toEqual([ "pp1", "ciap2:K86" ]);
    const withOnset = setItemOnset(items, "ciap2:K86", { onset_on: "2020-05-01", onset_precision: "month" });
    expect(withOnset[1]).toMatchObject({ action: "add", onset_on: "2020-05-01", onset_precision: "month" });
    expect(removeItem(withOnset, "pp1").map(problemKey)).toEqual([ "ciap2:K86" ]);
  });
});

describe("início com precisão", () => {
  const today = "2026-10-07";
  it.each([
    [ "year", "2019", { onset_on: "2019-01-01", onset_precision: "year" } ],
    [ "month", "2019-03", { onset_on: "2019-03-01", onset_precision: "month" } ],
    [ "day", "2019-03-12", { onset_on: "2019-03-12", onset_precision: "day" } ],
    [ "month", "2026-10", { onset_on: "2026-10-01", onset_precision: "month" } ],
    [ "day", "2026-10-07", { onset_on: "2026-10-07", onset_precision: "day" } ]
  ] as const)("%s %s é aceito", (precision, text, expected) => {
    expect(parseOnset(precision, text, today)).toEqual(expected);
  });

  it.each([
    [ "year", "19", "informe o ano com 4 dígitos" ],
    [ "year", "1899", "use um ano a partir de 1900" ],
    [ "year", "2027", "o início não pode ser no futuro" ],
    [ "month", "2026-11", "o início não pode ser no futuro" ],
    [ "month", "2019-13", "informe o mês e o ano" ],
    [ "day", "2026-10-08", "o início não pode ser no futuro" ],
    [ "day", "2026-02-30", "informe uma data válida" ],
    [ "day", "", "informe uma data válida" ]
  ] as const)("%s %s é recusado: %s", (precision, text, message) => {
    expect(parseOnset(precision, text, today)).toEqual({ problem: message });
  });

  it("rótulo e valor do campo por precisão", () => {
    expect(onsetLabel("2019-03-12", "day")).toBe("desde 12/03/2019");
    expect(onsetLabel("2019-03-01", "month")).toBe("desde 03/2019");
    expect(onsetLabel("2019-01-01", "year")).toBe("desde 2019");
    expect(onsetLabel(null, null)).toBe("início não informado");
    expect(onsetInputValue("2019-03-01", "month")).toBe("2019-03");
    expect(onsetInputValue("2019-01-01", "year")).toBe("2019");
    expect(onsetInputValue("2019-03-12", "day")).toBe("2019-03-12");
  });
});

describe("exames", () => {
  it("não repete exame e guarda a justificativa como digitada", () => {
    const once = addExam([], { code: "0202010503", label: "HbA1c" });
    expect(addExam(once, { code: "0202010503", label: "HbA1c" })).toBe(once);
    expect(setJustification(once, "0202010503", "e11")[0].cid10_justification).toBe("e11");
  });

  it("CID-10: normaliza e confere o formato", () => {
    expect(normalizeCid10(" e11.9 ")).toBe("E119");
    expect(cid10Problem("E11")).toBeNull();
    expect(cid10Problem("e11.9")).toBeNull();
    expect(cid10Problem("")).toBeNull();
    expect(cid10Problem("E1")).toBe("use um código CID-10, ex.: E11 ou E119");
    expect(examsProblem([ { sigtap_code: "X", label: "x", cid10_justification: "11E" } ])).toBe("confira o CID-10 da justificativa do exame X");
  });
});

describe("adendo", () => {
  it("motivo com 10 caracteres e texto obrigatório", () => {
    expect(addendumProblem("curto", "texto")).toBe("o motivo do adendo precisa de pelo menos 10 caracteres");
    expect(addendumProblem("correção do plano", "  ")).toBe("escreva o texto do adendo");
    expect(addendumProblem("correção do plano", "x".repeat(20_001))).toBe("Texto do adendo passa de 20.000 caracteres");
    expect(addendumProblem("correção do plano", "Retorno em 15 dias.")).toBeNull();
  });

  it("só manda o que mudou", () => {
    const base = finalized();
    expect(addendumChanges(base, { problems: [], conducts: [ "9" ], exams: base.exam_requests })).toBeUndefined();
    expect(addendumChanges(base, { problems: [], conducts: [ "12" ], exams: base.exam_requests })).toEqual({ conducts: [ "12" ] });
    expect(addendumChanges(base, { problems: markProblem([], problem(), "resolve"), conducts: [ "9" ], exams: [] })).toEqual({
      evaluated_problems: [ { problem_id: "pp1", terminology: "ciap2", code: "T90", label: problem().label, action: "resolve" } ],
      exam_requests: []
    });
  });

  it("resumo das mudanças só com códigos e rótulos", () => {
    const label = (code: string) => codedLabel(options().conducts, code);
    expect(changesLines({ evaluated_problems: markProblem([], problem(), "resolve"), conducts: [ "12" ], exam_requests: [] }, label))
      .toEqual([ "Problemas: T90 resolvido", "Condutas: Alta do episódio", "Exames: nenhum" ]);
    expect(changesLines(null, label)).toEqual([]);
  });
});

describe("rótulos e recusas", () => {
  it("idade e rótulo de código", () => {
    expect(ageLabel(1)).toBe("1 ano");
    expect(ageLabel(54)).toBe("54 anos");
    expect(codedLabel(options().care_types, "5")).toBe("Consulta no dia");
    expect(codedLabel(options().care_types, "99")).toBe("99");
    expect(codedLabel(undefined, null)).toBe("—");
  });

  it.each([
    [ 409, { error: "citizen_not_verified" }, "o cadastro desta pessoa não foi validado no balcão — a consulta exige a validação presencial; encerre o atendimento só com o desfecho" ],
    [ 409, { error: "not_caller" }, "só quem chamou o atendimento registra a consulta" ],
    [ 403, { error: "cbo_not_allowed" }, "sua ocupação (CBO) não registra consulta" ],
    [ 403, { error: "feature_disabled", feature: "clinical_record" }, "o prontuário está desligado nesta cidade" ],
    [ 403, { error: "out_of_context" }, "este atendimento não está com você — para ler o prontuário fora do atendimento, use Prontuário com o motivo" ],
    [ 409, { error: "not_draft" }, "esta consulta já foi finalizada — a tela foi atualizada" ],
    [ 422, { error: "patient_name_missing" }, "falta o nome completo do paciente — peça à recepção para completar os nomes no check-in e tente de novo" ],
    [ 422, { error: "cid10_not_allowed_for_cbo" }, "sua ocupação não pode usar CID-10 — troque o problema por um código CIAP-2" ],
    [ 422, { error: "text_too_long", field: "assessment" }, "Avaliação (A) passa de 20.000 caracteres" ],
    [ 422, { error: "implausible_vital", field: "systolic" }, "Pressão sistólica: valor fora do plausível — confira" ],
    [ 403, { error: "opening_required" }, "a abertura justificada terminou ou não existe — abra o prontuário de novo com o motivo" ],
    [ 503, { error: "terminology_unavailable" }, "a terminologia não está disponível agora — tente de novo em instantes" ],
    [ 422, { error: "referral_required" }, "informe a unidade de destino ou a descrição do encaminhamento" ],
    [ 500, "boom", "não foi possível concluir — tente de novo" ]
  ])("%s %j → frase", (status, body, phrase) => {
    expect(consultationError(err(status, body))).toBe(phrase);
  });

  it("already_exists com o id retoma; sem id (api sem a D1), não", () => {
    expect(existingConsultationId(err(409, { error: "already_exists", consultation_id: "cs1" }))).toBe("cs1");
    expect(existingConsultationId(err(409, { error: "already_exists" }))).toBeNull();
    expect(existingConsultationId(err(409, { error: "not_caller" }))).toBeNull();
    expect(existingConsultationId(new Error("x"))).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/consultation.test.ts`
Expected: FAIL — `Failed to resolve import "./consultation"`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/consultation.ts
// Consulta do prontuário da APS (módulo 19a; ADR 0031; contratos do módulo 19
// §1 e §4). Regras fora do React: rascunho ↔ corpo do PATCH, o que falta para
// finalizar (espelho dos 422 — quem garante é o api), lista de problemas,
// início com precisão, exames, adendo e as frases das recusas. Nenhuma frase
// repete texto clínico: só códigos e rótulos.
import {
  ApiError, type AddendumChanges, type CodedOption, type Consultation, type ConsultationDraftInput,
  type EvaluatedProblem, type ExamRequest, type OnsetPrecision, type PatientProblem, type ProblemAction, type Terminology
} from "./api";
import { fmtDay } from "./audiencePhrase";
import { parseVitals, screeningError, vitalsFormFrom, type VitalsForm, type VitalsProblems } from "./screening";

export const RECORD_KEY = "attendanceRecord";
export const CONSULTATION_KEY = "consultation";
export const OPTIONS_KEY = "consultationOptions";
export const JUSTIFIED_KEY = "justifiedRecord";

export const TEXT_MAX = 20_000;
export const TEXT_MAX_LABEL = "20.000";
export const ADDENDUM_REASON_MIN = 10;

export type SoapField = "subjective" | "objective" | "assessment" | "plan";
export const SOAP_FIELDS: { key: SoapField; label: string }[] = [
  { key: "subjective", label: "Subjetivo (S)" },
  { key: "objective", label: "Objetivo (O)" },
  { key: "assessment", label: "Avaliação (A)" },
  { key: "plan", label: "Plano (P)" }
];
const TEXT_LABEL: Record<string, string> = {
  subjective: "Subjetivo (S)", objective: "Objetivo (O)", assessment: "Avaliação (A)", plan: "Plano (P)",
  text: "Texto do adendo", reason: "Motivo do adendo"
};
export function textLabel(field: string): string {
  return TEXT_LABEL[field] ?? "O texto";
}

export const TERMINOLOGY_LABEL: Record<Terminology, string> = { ciap2: "CIAP-2", cid10: "CID-10" };
export const ACTION_LABEL: Record<ProblemAction, string> = {
  evaluate: "avaliado", add: "incluído", resolve: "resolvido", correct_onset: "início corrigido"
};
export const PRECISIONS: OnsetPrecision[] = [ "day", "month", "year" ];
export const PRECISION_LABEL: Record<OnsetPrecision, string> = { day: "dia", month: "mês", year: "ano" };

export interface Onset { onset_on: string; onset_precision: OnsetPrecision }

export interface ConsultationDraft {
  soap: Record<SoapField, string>;
  vitals: VitalsForm;
  careType: string;
  problems: EvaluatedProblem[];
  conducts: string[];
  exams: ExamRequest[];
}

export function draftFrom(c: Consultation): ConsultationDraft {
  return {
    soap: { subjective: c.subjective ?? "", objective: c.objective ?? "", assessment: c.assessment ?? "", plan: c.plan ?? "" },
    vitals: vitalsFormFrom(c.vitals ?? {}),
    careType: c.care_type ?? "",
    problems: c.evaluated_problems ?? [],
    conducts: c.conducts ?? [],
    exams: c.exam_requests ?? []
  };
}

export interface DraftCheck { input: ConsultationDraftInput; vitalsProblems: VitalsProblems; tooLong: SoapField[] }

// Sinal com problema fica fora do corpo (o api recusaria o PATCH inteiro).
export function checkDraft(d: ConsultationDraft): DraftCheck {
  const { vitals, problems } = parseVitals(d.vitals);
  return {
    input: {
      subjective: d.soap.subjective, objective: d.soap.objective, assessment: d.soap.assessment, plan: d.soap.plan,
      vitals, care_type: d.careType === "" ? null : d.careType,
      evaluated_problems: d.problems, conducts: d.conducts, exam_requests: d.exams.map(examPayload)
    },
    vitalsProblems: problems,
    tooLong: SOAP_FIELDS.map((f) => f.key).filter((k) => d.soap[k].length > TEXT_MAX)
  };
}

// Texto longo demais não vai para o api (422 text_too_long): o rascunho para de salvar e diz qual campo.
export function blockedReason(check: DraftCheck): string | null {
  return check.tooLong.length > 0 ? `${textLabel(check.tooLong[0])} passa de ${TEXT_MAX_LABEL} caracteres` : null;
}

export function finalizeProblems(check: DraftCheck): string[] {
  const out: string[] = [];
  if (Object.keys(check.vitalsProblems).length > 0) out.push("corrija os sinais vitais marcados");
  for (const field of check.tooLong) out.push(`${textLabel(field)} passa de ${TEXT_MAX_LABEL} caracteres`);
  if (check.input.evaluated_problems.length === 0) out.push("avalie, inclua ou resolva ao menos um problema");
  if (check.input.conducts.length === 0) out.push("marque ao menos uma conduta");
  if (check.input.assessment.trim() === "" && check.input.plan.trim() === "") out.push("escreva a avaliação (A) ou o plano (P)");
  const exam = examsProblem(check.input.exam_requests);
  if (exam) out.push(exam);
  return out;
}

export function ageLabel(age: number): string {
  return `${age} ${age === 1 ? "ano" : "anos"}`;
}

export function codedLabel(list: CodedOption[] | null | undefined, code: string | null | undefined): string {
  if (!code) return "—";
  return list?.find((o) => o.code === code)?.label ?? code;
}

// ─── Problemas ───────────────────────────────────────────────────────────────
// Um item por problema da lista (pelo id); incluído novo pela terminologia e código.

export function problemKey(p: Pick<EvaluatedProblem, "problem_id" | "terminology" | "code">): string {
  return p.problem_id ?? `${p.terminology}:${p.code}`;
}

function fromPatient(problem: PatientProblem, action: ProblemAction): EvaluatedProblem {
  return { problem_id: problem.id, terminology: problem.terminology, code: problem.code, label: problem.label, action };
}

export function markProblem(items: EvaluatedProblem[], problem: PatientProblem, action: "evaluate" | "resolve"): EvaluatedProblem[] {
  return [ ...items.filter((i) => i.problem_id !== problem.id), fromPatient(problem, action) ];
}

export function correctOnset(items: EvaluatedProblem[], problem: PatientProblem, onset: Onset): EvaluatedProblem[] {
  return [ ...items.filter((i) => i.problem_id !== problem.id), { ...fromPatient(problem, "correct_onset"), ...onset } ];
}

// O api tem um ativo por (paciente, terminologia, código): incluir o que já
// está ativo vira "avaliado" (nunca um add duplicado).
export function addProblem(
  items: EvaluatedProblem[], patientProblems: PatientProblem[], terminology: Terminology, ref: CodedOption
): { items: EvaluatedProblem[]; notice: string | null } {
  const active = patientProblems.find((p) => p.status === "active" && p.terminology === terminology && p.code === ref.code);
  if (active) {
    if (items.some((i) => i.problem_id === active.id)) {
      return { items, notice: `${ref.code} já está na lista do paciente e já foi marcado nesta consulta` };
    }
    return { items: markProblem(items, active, "evaluate"), notice: `${ref.code} já está na lista do paciente — marcado como avaliado` };
  }
  if (items.some((i) => i.problem_id === null && i.terminology === terminology && i.code === ref.code)) {
    return { items, notice: `${ref.code} já foi incluído nesta consulta` };
  }
  return { items: [ ...items, { problem_id: null, terminology, code: ref.code, label: ref.label, action: "add" } ], notice: null };
}

export function setItemOnset(items: EvaluatedProblem[], key: string, onset: Onset): EvaluatedProblem[] {
  return items.map((i) => (problemKey(i) === key ? { ...i, ...onset } : i));
}

export function removeItem(items: EvaluatedProblem[], key: string): EvaluatedProblem[] {
  return items.filter((i) => problemKey(i) !== key);
}

// ─── Início com precisão ─────────────────────────────────────────────────────
// Datas YYYY-MM-DD comparadas como texto (nenhum fuso desloca o dia); mês e
// ano viram o primeiro dia do período.

function validDay(text: string): boolean {
  if (!/^\d{4}-\d{2}-\d{2}$/.test(text)) return false;
  const d = new Date(`${text}T12:00:00Z`);
  return !Number.isNaN(d.getTime()) && d.toISOString().slice(0, 10) === text;
}

export function parseOnset(precision: OnsetPrecision, text: string, today: string): Onset | { problem: string } {
  const t = text.trim();
  let on: string;
  if (precision === "year") {
    if (!/^\d{4}$/.test(t)) return { problem: "informe o ano com 4 dígitos" };
    on = `${t}-01-01`;
  } else if (precision === "month") {
    if (!/^\d{4}-(0[1-9]|1[0-2])$/.test(t)) return { problem: "informe o mês e o ano" };
    on = `${t}-01`;
  } else {
    if (!validDay(t)) return { problem: "informe uma data válida" };
    on = t;
  }
  if (Number(on.slice(0, 4)) < 1900) return { problem: "use um ano a partir de 1900" };
  const limit = precision === "year" ? today.slice(0, 4) : precision === "month" ? today.slice(0, 7) : today;
  if (on.slice(0, limit.length) > limit) return { problem: "o início não pode ser no futuro" };
  return { onset_on: on, onset_precision: precision };
}

export function onsetLabel(on: string | null | undefined, precision: OnsetPrecision | null | undefined): string {
  if (!on || !precision) return "início não informado";
  if (precision === "year") return `desde ${on.slice(0, 4)}`;
  if (precision === "month") return `desde ${on.slice(5, 7)}/${on.slice(0, 4)}`;
  return `desde ${fmtDay(on)}`;
}

export function onsetInputValue(on: string, precision: OnsetPrecision): string {
  if (precision === "year") return on.slice(0, 4);
  if (precision === "month") return on.slice(0, 7);
  return on;
}

// ─── Exames (SIGTAP) ─────────────────────────────────────────────────────────

export function normalizeCid10(text: string): string {
  return text.toUpperCase().replace(/[.\s]/g, "");
}

export function cid10Problem(text: string | null | undefined): string | null {
  const n = normalizeCid10(text ?? "");
  if (n === "") return null;
  return /^[A-Z]\d{2}[0-9A-Z]?$/.test(n) ? null : "use um código CID-10, ex.: E11 ou E119";
}

function examPayload(e: ExamRequest): ExamRequest {
  const j = normalizeCid10(e.cid10_justification ?? "");
  return j === "" ? { sigtap_code: e.sigtap_code, label: e.label } : { sigtap_code: e.sigtap_code, label: e.label, cid10_justification: j };
}

export function addExam(exams: ExamRequest[], ref: CodedOption): ExamRequest[] {
  if (exams.some((e) => e.sigtap_code === ref.code)) return exams;
  return [ ...exams, { sigtap_code: ref.code, label: ref.label } ];
}

export function setJustification(exams: ExamRequest[], code: string, text: string): ExamRequest[] {
  return exams.map((e) => (e.sigtap_code === code ? { ...e, cid10_justification: text } : e));
}

export function examsProblem(exams: ExamRequest[]): string | null {
  const bad = exams.find((e) => cid10Problem(e.cid10_justification) !== null);
  return bad ? `confira o CID-10 da justificativa do exame ${bad.sigtap_code}` : null;
}

// ─── Adendo ──────────────────────────────────────────────────────────────────

export function addendumProblem(reason: string, text: string): string | null {
  if (reason.trim().length < ADDENDUM_REASON_MIN) return `o motivo do adendo precisa de pelo menos ${ADDENDUM_REASON_MIN} caracteres`;
  if (text.trim() === "") return "escreva o texto do adendo";
  if (text.length > TEXT_MAX) return `Texto do adendo passa de ${TEXT_MAX_LABEL} caracteres`;
  return null;
}

export interface AddendumEdit { problems: EvaluatedProblem[]; conducts: string[]; exams: ExamRequest[] }

// Divergência D4: problemas são eventos novos; condutas e exames, a lista final
// (só vão quando mudaram).
export function addendumChanges(base: Consultation, edit: AddendumEdit): AddendumChanges | undefined {
  const changes: AddendumChanges = {};
  if (edit.problems.length > 0) changes.evaluated_problems = edit.problems;
  const sameConducts = edit.conducts.length === base.conducts.length && edit.conducts.every((c) => base.conducts.includes(c));
  if (!sameConducts) changes.conducts = edit.conducts;
  const examKey = (list: ExamRequest[]) =>
    JSON.stringify(list.map((e) => `${e.sigtap_code}|${normalizeCid10(e.cid10_justification ?? "")}`).sort());
  if (examKey(edit.exams) !== examKey(base.exam_requests)) changes.exam_requests = edit.exams.map(examPayload);
  return Object.keys(changes).length > 0 ? changes : undefined;
}

export function changesLines(changes: AddendumChanges | null | undefined, conductLabel: (code: string) => string): string[] {
  if (!changes) return [];
  const lines: string[] = [];
  if (changes.evaluated_problems?.length) {
    lines.push(`Problemas: ${changes.evaluated_problems.map((p) => `${p.code} ${ACTION_LABEL[p.action]}`).join(", ")}`);
  }
  if (changes.conducts) lines.push(`Condutas: ${changes.conducts.map(conductLabel).join(" · ") || "nenhuma"}`);
  if (changes.exam_requests) lines.push(`Exames: ${changes.exam_requests.map((e) => e.sigtap_code).join(", ") || "nenhum"}`);
  return lines;
}

// ─── Recusas ─────────────────────────────────────────────────────────────────

const MESSAGES: Record<string, string> = {
  citizen_not_verified: "o cadastro desta pessoa não foi validado no balcão — a consulta exige a validação presencial; encerre o atendimento só com o desfecho",
  not_in_care: "este atendimento não está mais em atendimento — a fila foi atualizada",
  not_caller: "só quem chamou o atendimento registra a consulta",
  already_exists: "a consulta deste atendimento já foi iniciada, mas não foi possível retomá-la — recarregue a página",
  cbo_not_allowed: "sua ocupação (CBO) não registra consulta",
  feature_disabled: "o prontuário está desligado nesta cidade",
  out_of_context: "este atendimento não está com você — para ler o prontuário fora do atendimento, use Prontuário com o motivo",
  not_draft: "esta consulta já foi finalizada — a tela foi atualizada",
  not_author: "só quem escreveu a consulta pode editá-la",
  no_problem_evaluated: "avalie, inclua ou resolva ao menos um problema",
  no_conduct: "marque ao menos uma conduta",
  assessment_or_plan_required: "escreva a avaliação (A) ou o plano (P)",
  patient_name_missing: "falta o nome completo do paciente — peça à recepção para completar os nomes no check-in e tente de novo",
  cid10_not_allowed_for_cbo: "sua ocupação não pode usar CID-10 — troque o problema por um código CIAP-2",
  not_finalized: "a consulta ainda não foi finalizada",
  opening_required: "a abertura justificada terminou ou não existe — abra o prontuário de novo com o motivo",
  invalid_reason: `o motivo do adendo precisa de pelo menos ${ADDENDUM_REASON_MIN} caracteres`,
  terminology_unavailable: "a terminologia não está disponível agora — tente de novo em instantes"
};

export function consultationError(err: unknown): string {
  if (err instanceof ApiError) {
    const body = (err.body ?? {}) as { error?: string; field?: string };
    if (body.error === "text_too_long") return `${textLabel(body.field ?? "")} passa de ${TEXT_MAX_LABEL} caracteres`;
    if (body.error && MESSAGES[body.error]) return MESSAGES[body.error];
  }
  // implausible_vital (com field), os do close e o genérico.
  return screeningError(err);
}

// Divergência D1: o 409 already_exists traz o id do rascunho existente.
export function existingConsultationId(err: unknown): string | null {
  if (!(err instanceof ApiError) || err.status !== 409) return null;
  const body = (err.body ?? {}) as { error?: unknown; consultation_id?: unknown };
  return body.error === "already_exists" && typeof body.consultation_id === "string" ? body.consultation_id : null;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/consultation.test.ts && npx tsc --noEmit`
Expected: PASS; `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/lib/consultation.ts src/lib/consultation.test.ts
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "feat: add consultation draft, problem list and refusal rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 3: Desfecho do atendimento reaproveitável (a mesma tela do "Encerrar")

A finalização da consulta manda o mesmo corpo do `close` (contratos §4) e a spec pede a tela de desfecho existente. Os campos do `ClosePanel` saem para `OutcomeFields` e as regras para `src/lib/outcome.ts`, sem mudar nada do que a pessoa vê nem do corpo enviado: os 34 testes de `UnitQueue.test.tsx` continuam verdes sem alteração.

**Files:**
- Create: `src/lib/outcome.ts`, `src/modules/attendance/OutcomeFields.tsx`
- Modify: `src/modules/attendance/UnitQueue.tsx` (`ClosePanel`, `OUTCOME_LABEL` e imports)
- Test: `src/lib/outcome.test.ts`, `src/modules/attendance/OutcomeFields.test.tsx` (e `src/modules/attendance/UnitQueue.test.tsx` sem mudança)

**Interfaces:**
- Consumes: `CareOutcome`, `OutcomeBody`, `HealthUnit` (Task 1 e existentes); `splitReferenceUnits` (`src/lib/attendance.ts`); `FrozenTextNotice`, `inputStyle`.
- Produces:
  - `src/lib/outcome.ts`: `CARE_OUTCOMES`, `OUTCOME_LABEL`, `interface OutcomeDraft { outcome: CareOutcome; referralChoice: string | null; note: string }`, `EMPTY_OUTCOME`, `outcomeView(draft, referenceIds, unit, units): { referenceUnits; otherUnits; referralUnitId; targetUnitName }`, `outcomeProblem(draft, referralUnitId): string | null`, `outcomeBody(draft, referralUnitId): OutcomeBody`;
  - `OutcomeFields({ value: OutcomeDraft; onChange(next): void; referenceIds?: string[]; unit: HealthUnit; units: HealthUnit[]; idPrefix?: string })` — os mesmos rótulos de hoje ("Desfecho", "Unidade de destino", "Descrição", "Nota (opcional)") e a linha "Gera pedido de agendamento na <unidade>"; os ids dos avisos são `${idPrefix}referral-note-notice` e `${idPrefix}return-note-notice` (sem prefixo no `ClosePanel`, como hoje).

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/outcome.test.ts
import { describe, expect, it } from "vitest";
import { EMPTY_OUTCOME, outcomeBody, outcomeProblem, outcomeView } from "./outcome";

const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };
const units = [ unit, { id: "u2", name: "UBS Bairro Alto", kind: "ubs" }, { id: "u3", name: "Ambulatório de Especialidades", kind: "other" } ];

describe("desfecho do atendimento", () => {
  it("referência do bairro vem escolhida; a própria unidade nunca é referência", () => {
    const v = outcomeView({ ...EMPTY_OUTCOME, outcome: "referred" }, [ "u3", "u1" ], unit, units);
    expect(v.referenceUnits.map((u) => u.id)).toEqual([ "u3" ]);
    expect(v.referralUnitId).toBe("u3");
    expect(v.targetUnitName).toBe("Ambulatório de Especialidades");
  });

  it("escolher '—' vale mais que a sugestão; retorno é na própria unidade", () => {
    expect(outcomeView({ outcome: "referred", referralChoice: "", note: "" }, [ "u3" ], unit, units).referralUnitId).toBe("");
    expect(outcomeView({ ...EMPTY_OUTCOME, outcome: "return" }, [], unit, units).targetUnitName).toBe("UBS Centro");
    expect(outcomeView(EMPTY_OUTCOME, [], unit, units).targetUnitName).toBeUndefined();
  });

  it("encaminhar pede unidade ou descrição", () => {
    expect(outcomeProblem({ outcome: "referred", referralChoice: null, note: "" }, "")).toBe(
      "informe a unidade de destino ou a descrição do encaminhamento");
    expect(outcomeProblem({ outcome: "referred", referralChoice: null, note: "cardiologia" }, "")).toBeNull();
    expect(outcomeProblem(EMPTY_OUTCOME, "")).toBeNull();
  });

  it("corpo igual ao do close de hoje", () => {
    expect(outcomeBody(EMPTY_OUTCOME, "u3")).toEqual({ outcome: "discharged" });
    expect(outcomeBody({ outcome: "referred", referralChoice: null, note: "" }, "u3")).toEqual({ outcome: "referred", referral_unit_id: "u3" });
    expect(outcomeBody({ outcome: "return", referralChoice: null, note: "trazer exames" }, "")).toEqual({ outcome: "return", referral_note: "trazer exames" });
  });
});
```

```tsx
// src/modules/attendance/OutcomeFields.test.tsx
import { afterEach, describe, expect, it } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { useState } from "react";
import { EMPTY_OUTCOME, type OutcomeDraft } from "../../lib/outcome";
import { OutcomeFields } from "./OutcomeFields";
import { expectFrozenNotice } from "../../test/frozenNotice";

afterEach(cleanup);
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };
const units = [ unit, { id: "u2", name: "UBS Bairro Alto", kind: "ubs" } ];

function Harness({ idPrefix }: { idPrefix?: string }) {
  const [ value, setValue ] = useState<OutcomeDraft>(EMPTY_OUTCOME);
  return (
    <>
      <OutcomeFields value={value} onChange={setValue} referenceIds={[ "u2" ]} unit={unit} units={units} idPrefix={idPrefix} />
      <pre data-testid="value">{JSON.stringify(value)}</pre>
    </>
  );
}

describe("OutcomeFields", () => {
  it("encaminhado: referência escolhida, descrição com o aviso e a linha do pedido", () => {
    render(<Harness />);
    fireEvent.change(screen.getByLabelText("Desfecho"), { target: { value: "referred" } });
    expect((screen.getByLabelText("Unidade de destino") as HTMLSelectElement).value).toBe("u2");
    expect(screen.getByRole("option", { name: "UBS Bairro Alto · referência" })).not.toBeNull();
    expectFrozenNotice(screen.getByLabelText("Descrição"));
    expect(screen.getByText("Gera pedido de agendamento na UBS Bairro Alto")).not.toBeNull();
  });

  it("retorno: nota opcional; o prefixo separa os ids dos avisos", () => {
    render(<Harness idPrefix="consultation-" />);
    fireEvent.change(screen.getByLabelText("Desfecho"), { target: { value: "return" } });
    const note = screen.getByLabelText("Nota (opcional)");
    expect(note.getAttribute("aria-describedby")).toBe("consultation-return-note-notice");
    expectFrozenNotice(note);
    fireEvent.change(note, { target: { value: "trazer exames" } });
    expect(JSON.parse(screen.getByTestId("value").textContent ?? "{}")).toEqual({ outcome: "return", referralChoice: null, note: "trazer exames" });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/outcome.test.ts src/modules/attendance/OutcomeFields.test.tsx`
Expected: FAIL — `Failed to resolve import "./outcome"` e `"./OutcomeFields"`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/outcome.ts
// Desfecho do atendimento (spec 2026-09-24 §6; módulo 11 D4; contratos do
// módulo 19 §4): o mesmo corpo do POST /attendance/attendances/:id/close serve
// ao "Encerrar" da fila e ao "Finalizar" da consulta.
import type { CareOutcome, HealthUnit, OutcomeBody } from "./api";
import { splitReferenceUnits } from "./attendance";

export const CARE_OUTCOMES: CareOutcome[] = [ "discharged", "referred", "return" ];
export const OUTCOME_LABEL: Record<CareOutcome, string> = {
  discharged: "Atendido e liberado",
  referred: "Encaminhado",
  return: "Retorno"
};

// `referralChoice` null = a pessoa ainda não mexeu, e vale a primeira unidade
// de referência (que pode chegar depois, com `units`); "" = escolheu "—".
export interface OutcomeDraft { outcome: CareOutcome; referralChoice: string | null; note: string }
export const EMPTY_OUTCOME: OutcomeDraft = { outcome: "discharged", referralChoice: null, note: "" };

export function outcomeView(draft: OutcomeDraft, referenceIds: string[] | undefined, unit: HealthUnit, units: HealthUnit[]) {
  const { referenceUnits, otherUnits } = splitReferenceUnits(units, referenceIds, unit.id);
  const referralUnitId = draft.referralChoice ?? referenceUnits[0]?.id ?? "";
  const targetUnitName = draft.outcome === "return"
    ? unit.name
    : (draft.outcome === "referred" && referralUnitId ? units.find((u) => u.id === referralUnitId)?.name : undefined);
  return { referenceUnits, otherUnits, referralUnitId, targetUnitName };
}

export function outcomeProblem(draft: OutcomeDraft, referralUnitId: string): string | null {
  return draft.outcome === "referred" && !referralUnitId && !draft.note.trim()
    ? "informe a unidade de destino ou a descrição do encaminhamento"
    : null;
}

// Igual ao ClosePanel de antes: unidade só no encaminhamento; nota sempre que preenchida.
export function outcomeBody(draft: OutcomeDraft, referralUnitId: string): OutcomeBody {
  const body: OutcomeBody = { outcome: draft.outcome };
  if (draft.outcome === "referred" && referralUnitId) body.referral_unit_id = referralUnitId;
  if (draft.note) body.referral_note = draft.note;
  return body;
}
```

```tsx
// src/modules/attendance/OutcomeFields.tsx
// Campos do desfecho (os do "Encerrar" de antes), usados pelo ClosePanel e pela
// finalização da consulta (módulo 19). `idPrefix` separa os ids dos avisos
// quando os dois estão na tela.
import type { CareOutcome, HealthUnit } from "../../lib/api";
import { CARE_OUTCOMES, OUTCOME_LABEL, outcomeView, type OutcomeDraft } from "../../lib/outcome";
import { inputStyle } from "../../components/formStyles";
import { FrozenTextNotice } from "../../components/FrozenTextNotice";

interface Props {
  value: OutcomeDraft;
  onChange(next: OutcomeDraft): void;
  referenceIds?: string[];
  unit: HealthUnit;
  units: HealthUnit[];
  idPrefix?: string;
}

export function OutcomeFields({ value, onChange, referenceIds, unit, units, idPrefix = "" }: Props) {
  const { referenceUnits, otherUnits, referralUnitId, targetUnitName } = outcomeView(value, referenceIds, unit, units);
  const set = (patch: Partial<OutcomeDraft>) => onChange({ ...value, ...patch });

  return (
    <>
      <label style={labelStyle}>
        Desfecho
        <select value={value.outcome} onChange={(e) => set({ outcome: e.target.value as CareOutcome })} style={inputStyle}>
          {CARE_OUTCOMES.map((o) => <option key={o} value={o}>{OUTCOME_LABEL[o]}</option>)}
        </select>
      </label>

      {value.outcome === "referred" && (
        <>
          <label style={labelStyle}>
            Unidade de destino
            <select value={referralUnitId} onChange={(e) => set({ referralChoice: e.target.value })} style={inputStyle}>
              <option value="">—</option>
              {referenceUnits.map((u) => <option key={u.id} value={u.id}>{`${u.name} · referência`}</option>)}
              {otherUnits.map((u) => <option key={u.id} value={u.id}>{u.name}</option>)}
            </select>
          </label>
          <label style={labelStyle}>
            Descrição
            <input value={value.note} onChange={(e) => set({ note: e.target.value })} style={inputStyle}
              aria-describedby={`${idPrefix}referral-note-notice`} />
          </label>
          <FrozenTextNotice id={`${idPrefix}referral-note-notice`} />
        </>
      )}

      {value.outcome === "return" && (
        <>
          <label style={labelStyle}>
            Nota (opcional)
            <input value={value.note} onChange={(e) => set({ note: e.target.value })} style={inputStyle}
              aria-describedby={`${idPrefix}return-note-notice`} />
          </label>
          <FrozenTextNotice id={`${idPrefix}return-note-notice`} />
        </>
      )}

      {targetUnitName && (
        <p className="mono" style={{ margin: 0, fontSize: 11, color: "var(--ink3)" }}>
          Gera pedido de agendamento na {targetUnitName}
        </p>
      )}
    </>
  );
}

const labelStyle = { display: "flex", flexDirection: "column" as const, gap: 4, fontSize: 12, color: "var(--ink2)" };
```

Em `src/modules/attendance/UnitQueue.tsx`, os imports e a constante saem (o resto do arquivo, inclusive o que o módulo 18 pôs, não muda):

```diff
--- a/src/modules/attendance/UnitQueue.tsx
+++ b/src/modules/attendance/UnitQueue.tsx
@@ -1,22 +1,23 @@
 import { useState } from "react";
 import { useQuery, useQueryClient } from "@tanstack/react-query";
 import {
   callAttendance, callNext, closeAttendance, errorCode, getScreening, listUnitQueue,
-  type AppointmentRequestSummary, type AttendanceOutcome, type HealthUnit, type QueueRow, type Screening
+  type AppointmentRequestSummary, type HealthUnit, type QueueRow, type Screening
 } from "../../lib/api";
-import { ATTENDANCE_REFETCH_MS, attendanceError, splitReferenceUnits } from "../../lib/attendance";
+import { ATTENDANCE_REFETCH_MS, attendanceError } from "../../lib/attendance";
+import { EMPTY_OUTCOME, outcomeBody, outcomeProblem, outcomeView, type OutcomeDraft } from "../../lib/outcome";
 import { fmtDateTime, fmtHourMinute } from "../../lib/format";
 import { useAuth } from "../../lib/auth";
 import { Panel } from "../../components/Panel";
 import { DataTable } from "../../components/DataTable";
 import { EmptyState } from "../../components/EmptyState";
-import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
-import { FrozenTextNotice } from "../../components/FrozenTextNotice";
+import { buttonStyle, disabledButtonStyle, secondaryButtonStyle } from "../../components/formStyles";
 import { Tag } from "../../components/Tag";
 import { COLOR_LABEL, COLOR_TONE, screeningError, waitLabel, waitedMinutes } from "../../lib/screening";
 import { ScreeningDetail, ScreeningDetailLoader } from "./ScreeningDetail";
 import { ScreeningForm } from "./ScreeningForm";
+import { OutcomeFields } from "./OutcomeFields";
@@
-const OUTCOME_LABEL: Record<Exclude<AttendanceOutcome, "left">, string> = {
-  discharged: "Atendido e liberado",
-  referred: "Encaminhado",
-  return: "Retorno"
-};
-
```

E a função `ClosePanel` inteira (de `function ClosePanel(` até o fim do seu `return`) é substituída por esta; a constante `labelStyle` do fim do arquivo, que só o `ClosePanel` usava, sai (o `noUnusedLocals` recusaria); `colorTag` (do módulo 18) fica:

```tsx
function ClosePanel(
  { row, unit, units, onClinicalRefused, onCancel, onDone }: {
    row: QueueRow; unit: HealthUnit; units: HealthUnit[];
    onClinicalRefused?(): void;
    onCancel(): void; onDone(appointmentRequest: AppointmentRequestSummary | null): void;
  }
) {
  const auth = useAuth();
  // Campos e regras em OutcomeFields/outcome.ts (módulo 19: a finalização da
  // consulta usa os mesmos). A primeira unidade de referência vem escolhida.
  const [ draft, setDraft ] = useState<OutcomeDraft>(EMPTY_OUTCOME);
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const { referralUnitId } = outcomeView(draft, row.reference_unit_ids, unit, units);
  const disabled = busy || outcomeProblem(draft, referralUnitId) !== null;

  async function confirm() {
    if (disabled) return;
    setBusy(true); setError(null);
    try {
      const body = outcomeBody(draft, referralUnitId);
      const result = await closeAttendance(row.id, body.outcome, body.referral_unit_id, body.referral_note);
      onDone(result.appointmentRequest);
    } catch (err) {
      const code = errorCode(err);
      if (code === "already_closed") { onDone(null); return; }
      handleClinicalRefusal(code, () => void auth.reload(), onClinicalRefused);
      setError(attendanceError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <section style={{ display: "flex", flexDirection: "column", gap: 10, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 }}>
      <strong>Encerrar atendimento</strong>
      <p className="mono" style={{ margin: 0, fontSize: 11, color: "var(--ink3)" }}>
        {row.cpf_masked} · {row.protocol_name ?? "—"} · chegou às {fmtDateTime(row.checked_in_at)}
      </p>
      {error && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{error}</p>}

      <OutcomeFields value={draft} onChange={setDraft} referenceIds={row.reference_unit_ids} unit={unit} units={units} />

      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={disabled} onClick={() => void confirm()} style={disabled ? disabledButtonStyle : buttonStyle}>
          Confirmar encerramento
        </button>
        <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>
          Cancelar
        </button>
      </div>
    </section>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/outcome.test.ts src/modules/attendance/OutcomeFields.test.tsx src/modules/attendance/UnitQueue.test.tsx src/modules/attendance/UnitQueue.screening.test.tsx && npx tsc --noEmit`
Expected: PASS (4 + 2 novos; `UnitQueue.test.tsx` com os 34 de antes e `UnitQueue.screening.test.tsx` do 18, sem mudança); `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/lib/outcome.ts src/lib/outcome.test.ts src/modules/attendance/OutcomeFields.tsx src/modules/attendance/OutcomeFields.test.tsx src/modules/attendance/UnitQueue.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "refactor: extract the attendance outcome fields for reuse

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 4: Busca de código por nome (CIAP-2, CID-10 e SIGTAP)

O `Ciap2Search` do 18 guarda **uma** queixa; a consulta precisa de uma busca que **acrescenta** itens a uma lista (problemas e exames) e troca de terminologia. Um componente só, `CodeSearch`, serve aos dois usos e não mexe no do 18.

**Files:**
- Create: `src/modules/consultation/CodeSearch.tsx`
- Test: `src/modules/consultation/CodeSearch.test.tsx`

**Interfaces:**
- Consumes: `CodedOption` (Task 1); `useDebouncedValue` (`src/lib/useDebouncedValue.ts`).
- Produces: `CODE_SEARCH_MIN_CHARS = 2`; `CodeSearch({ label: string; placeholder?: string; queryKey: string; search(term: string): Promise<CodedOption[]>; onPick(item: CodedOption): void; errorText(err: unknown): string; delayMs?: number })` — campo com o rótulo dado; um botão `"<código> — <nome>"` por resultado dentro da lista `resultados: <rótulo>`; escolher chama `onPick` e limpa o campo; chave de cache `[ queryKey, termo ]`. O termo vai pelo `search` (POST), nunca na URL.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/consultation/CodeSearch.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";
import { ApiError, type CodedOption } from "../../lib/api";
import { consultationError } from "../../lib/consultation";
import { CodeSearch } from "./CodeSearch";

afterEach(cleanup);

function renderIt(search: (q: string) => Promise<CodedOption[]>, onPick = vi.fn()) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<CodeSearch label="Incluir problema (CIAP-2)" queryKey="terminologySearch:ciap2" search={search}
    onPick={onPick} errorText={consultationError} delayMs={0} />, { wrapper });
  return onPick;
}

describe("CodeSearch", () => {
  it("busca por nome, escolhe e limpa o campo", async () => {
    const search = vi.fn(async () => [ { code: "T90", label: "Diabetes não insulino-dependente" } ]);
    const onPick = renderIt(search);
    const field = screen.getByLabelText("Incluir problema (CIAP-2)") as HTMLInputElement;
    fireEvent.change(field, { target: { value: "diabetes" } });
    const list = await screen.findByRole("list", { name: "resultados: Incluir problema (CIAP-2)" });
    fireEvent.click(within(list).getByRole("button", { name: "T90 — Diabetes não insulino-dependente" }));
    expect(search).toHaveBeenCalledWith("diabetes");
    expect(onPick).toHaveBeenCalledWith({ code: "T90", label: "Diabetes não insulino-dependente" });
    expect(field.value).toBe("");
    expect(screen.queryByRole("list", { name: "resultados: Incluir problema (CIAP-2)" })).toBeNull();
  });

  it("um caractere não busca", async () => {
    const search = vi.fn(async () => []);
    renderIt(search);
    fireEvent.change(screen.getByLabelText("Incluir problema (CIAP-2)"), { target: { value: "d" } });
    expect(screen.getByText("digite pelo menos 2 caracteres")).not.toBeNull();
    await new Promise((r) => setTimeout(r, 20));
    expect(search).not.toHaveBeenCalled();
  });

  it("nada encontrado e terminologia indisponível", async () => {
    const search = vi.fn()
      .mockResolvedValueOnce([])
      .mockRejectedValueOnce(new ApiError(503, { error: "terminology_unavailable" }, "503"));
    renderIt(search);
    fireEvent.change(screen.getByLabelText("Incluir problema (CIAP-2)"), { target: { value: "xyzw" } });
    expect(await screen.findByText("nenhum código encontrado")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Incluir problema (CIAP-2)"), { target: { value: "febre" } });
    expect((await screen.findByRole("alert")).textContent).toBe("a terminologia não está disponível agora — tente de novo em instantes");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/modules/consultation/CodeSearch.test.tsx`
Expected: FAIL — `Failed to resolve import "./CodeSearch"`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/consultation/CodeSearch.tsx
// Busca de código por nome ou código (módulo 19; contratos §5): problemas em
// CIAP-2 ou CID-10 e exames em SIGTAP. O termo vai no corpo de um POST (pode
// descrever a queixa) e só a partir de 2 caracteres. Escolher acrescenta o
// item à lista de quem usa (onPick) e limpa o campo para a próxima busca.
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import type { CodedOption } from "../../lib/api";
import { useDebouncedValue } from "../../lib/useDebouncedValue";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";

export const CODE_SEARCH_MIN_CHARS = 2;

interface Props {
  label: string;
  placeholder?: string;
  queryKey: string;
  search(term: string): Promise<CodedOption[]>;
  onPick(item: CodedOption): void;
  errorText(err: unknown): string;
  delayMs?: number;
}

export function CodeSearch({ label, placeholder, queryKey, search, onPick, errorText, delayMs = 300 }: Props) {
  const [ text, setText ] = useState("");
  const typed = text.trim();
  const term = useDebouncedValue(typed, delayMs);
  const enabled = term.length >= CODE_SEARCH_MIN_CHARS;
  const query = useQuery({ queryKey: [ queryKey, term ], queryFn: () => search(term), enabled, staleTime: 5 * 60_000 });
  // Depois de escolher, o campo limpa na hora; o termo "atrasado" não pode
  // manter a lista velha na tela.
  const show = enabled && typed.length >= CODE_SEARCH_MIN_CHARS;

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
      <label style={labelStyle}>
        {label}
        <input value={text} placeholder={placeholder} style={inputStyle} onChange={(e) => setText(e.target.value)} />
      </label>
      {typed.length > 0 && typed.length < CODE_SEARCH_MIN_CHARS && (
        <small style={hint}>digite pelo menos {CODE_SEARCH_MIN_CHARS} caracteres</small>
      )}
      {show && query.isPending && <small className="mono" style={hint}>buscando…</small>}
      {show && query.isError && <p role="alert" style={alert}>{errorText(query.error)}</p>}
      {show && query.isSuccess && query.data.length === 0 && <small style={hint}>nenhum código encontrado</small>}
      {show && query.isSuccess && query.data.length > 0 && (
        <ul aria-label={`resultados: ${label}`} style={{ listStyle: "none", margin: 0, padding: 0, display: "flex", flexDirection: "column", gap: 4 }}>
          {query.data.map((item) => (
            <li key={item.code}>
              <button type="button" style={{ ...secondaryButtonStyle, width: "100%", textAlign: "left" }}
                onClick={() => { onPick(item); setText(""); }}>
                {`${item.code} — ${item.label}`}
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/modules/consultation/CodeSearch.test.tsx`
Expected: PASS (3 testes).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/modules/consultation/CodeSearch.tsx src/modules/consultation/CodeSearch.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "feat: search problem and exam codes by name

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 5: Itens estruturados — problemas, condutas e exames

**Files:**
- Create: `src/modules/consultation/OnsetInput.tsx`, `src/modules/consultation/ProblemsEditor.tsx`, `src/modules/consultation/ConductsField.tsx`, `src/modules/consultation/ExamRequestsField.tsx`
- Test: `src/modules/consultation/ProblemsEditor.test.tsx`, `src/modules/consultation/StructuredFields.test.tsx`

**Interfaces:**
- Consumes: `searchTerminology`, `searchSigtap`, tipos (Task 1); `ACTION_LABEL`, `TERMINOLOGY_LABEL`, `PRECISIONS`, `PRECISION_LABEL`, `addProblem`, `markProblem`, `correctOnset`, `setItemOnset`, `removeItem`, `problemKey`, `parseOnset`, `onsetLabel`, `onsetInputValue`, `addExam`, `setJustification`, `cid10Problem`, `consultationError`, `Onset` (Task 2); `CodeSearch` (Task 4); `Tag`, `formStyles`.
- Produces:
  - `OnsetInput({ code: string; today: string; initial?: Onset | null; onApply(onset: Onset): void; onCancel(): void })` — grupo "Início de <código>" com "Precisão" (dia/mês/ano), "Início" (`date`, `month` ou texto com 4 dígitos) e "Aplicar início";
  - `ProblemsEditor({ patientProblems: PatientProblem[]; items: EvaluatedProblem[]; onChange(next): void; cid10Allowed: boolean; today: string; searchDelayMs?: number })` — fieldset "Problemas e condições": lista "problemas ativos do paciente" com botões "Avaliar <código>", "Resolver <código>", "Corrigir início <código>", "Desfazer <código>"; lista "problemas incluídos nesta consulta" com "Informar início <código>" e "Remover <código>"; select "Terminologia" (CID-10 só com `cid10Allowed`) e a busca "Incluir problema (CIAP-2|CID-10)"; aviso `role="status"` de `addProblem`;
  - `ConductsField({ options: CodedOption[]; value: string[]; onChange(next): void })` — fieldset "Condutas", uma caixa por opção;
  - `ExamRequestsField({ value: ExamRequest[]; onChange(next): void; cid10Allowed: boolean; searchDelayMs?: number })` — fieldset "Exames solicitados", busca "Solicitar exame (SIGTAP)", "Remover exame <código>" e, com `cid10Allowed`, "CID-10 de justificativa (<código>)" com o problema sob o campo.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/consultation/ProblemsEditor.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState, type ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, searchTerminology: vi.fn() };
});

import * as api from "../../lib/api";
import type { EvaluatedProblem } from "../../lib/api";
import { ProblemsEditor } from "./ProblemsEditor";
import { TODAY19, problem } from "../../test/consultationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const T90 = problem();
const K86 = problem({ id: "pp2", code: "K86", label: "Hipertensão sem complicações", onset_on: null, onset_precision: null });

function Harness({ cid10Allowed = true }: { cid10Allowed?: boolean }) {
  const [ items, setItems ] = useState<EvaluatedProblem[]>([]);
  return (
    <>
      <ProblemsEditor patientProblems={[ T90, K86 ]} items={items} onChange={setItems} cid10Allowed={cid10Allowed}
        today={TODAY19} searchDelayMs={0} />
      <pre data-testid="items">{JSON.stringify(items)}</pre>
    </>
  );
}
function renderIt(cid10Allowed?: boolean) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<Harness cid10Allowed={cid10Allowed} />, { wrapper });
}
const items = (): EvaluatedProblem[] => JSON.parse(screen.getByTestId("items").textContent ?? "[]");

describe("ProblemsEditor", () => {
  beforeEach(() => mocked(api.searchTerminology).mockReset());

  it("lista os ativos com o início e marca avaliar, resolver e desfazer", () => {
    renderIt();
    const list = screen.getByRole("list", { name: "problemas ativos do paciente" });
    expect(within(list).getByText(/Diabetes não insulino-dependente · desde 03\/2019/)).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Avaliar T90" }));
    expect(items()).toEqual([ { problem_id: "pp1", terminology: "ciap2", code: "T90", label: T90.label, action: "evaluate" } ]);
    expect(within(list).getByText("avaliado")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Resolver T90" }));
    expect(items().map((i) => i.action)).toEqual([ "resolve" ]);
    fireEvent.click(screen.getByRole("button", { name: "Desfazer T90" }));
    expect(items()).toEqual([]);
  });

  it("corrigir início com precisão de ano; futuro é recusado sob o campo", () => {
    renderIt();
    fireEvent.click(screen.getByRole("button", { name: "Corrigir início K86" }));
    const group = screen.getByRole("group", { name: "Início de K86" });
    fireEvent.change(within(group).getByLabelText("Precisão"), { target: { value: "year" } });
    fireEvent.change(within(group).getByLabelText("Início"), { target: { value: "2027" } });
    fireEvent.click(within(group).getByRole("button", { name: "Aplicar início" }));
    expect(within(group).getByRole("alert").textContent).toBe("o início não pode ser no futuro");
    fireEvent.change(within(group).getByLabelText("Início"), { target: { value: "2018" } });
    fireEvent.click(within(group).getByRole("button", { name: "Aplicar início" }));
    expect(items()).toEqual([ { problem_id: "pp2", terminology: "ciap2", code: "K86", label: K86.label, action: "correct_onset",
      onset_on: "2018-01-01", onset_precision: "year" } ]);
    expect(screen.queryByRole("group", { name: "Início de K86" })).toBeNull();
  });

  it("incluir T90 que já está na lista marca avaliado e avisa", async () => {
    mocked(api.searchTerminology).mockResolvedValue([ { code: "T90", label: "Diabetes não insulino-dependente" } ]);
    renderIt();
    fireEvent.change(screen.getByLabelText("Incluir problema (CIAP-2)"), { target: { value: "diabetes" } });
    fireEvent.click(await screen.findByRole("button", { name: "T90 — Diabetes não insulino-dependente" }));
    expect(items().map((i) => [ i.problem_id, i.action ])).toEqual([ [ "pp1", "evaluate" ] ]);
    expect(screen.getByRole("status").textContent).toBe("T90 já está na lista do paciente — marcado como avaliado");
  });

  it("incluir em CID-10 busca na terminologia escolhida e pede o início do novo", async () => {
    mocked(api.searchTerminology).mockResolvedValue([ { code: "E11", label: "Diabetes mellitus não insulino-dependente" } ]);
    renderIt();
    fireEvent.change(screen.getByLabelText("Terminologia"), { target: { value: "cid10" } });
    fireEvent.change(screen.getByLabelText("Incluir problema (CID-10)"), { target: { value: "diabetes" } });
    fireEvent.click(await screen.findByRole("button", { name: "E11 — Diabetes mellitus não insulino-dependente" }));
    expect(api.searchTerminology).toHaveBeenCalledWith("diabetes", "cid10");
    const added = screen.getByRole("list", { name: "problemas incluídos nesta consulta" });
    expect(within(added).getByText(/CID-10 · início não informado/)).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Informar início E11" }));
    const group = screen.getByRole("group", { name: "Início de E11" });
    fireEvent.change(within(group).getByLabelText("Início"), { target: { value: "2024-05" } });
    fireEvent.click(within(group).getByRole("button", { name: "Aplicar início" }));
    expect(items()).toEqual([ { problem_id: null, terminology: "cid10", code: "E11", label: "Diabetes mellitus não insulino-dependente",
      action: "add", onset_on: "2024-05-01", onset_precision: "month" } ]);
    fireEvent.click(screen.getByRole("button", { name: "Remover E11" }));
    expect(items()).toEqual([]);
  });

  it("sem CID-10 para o CBO, a terminologia só oferece CIAP-2", () => {
    renderIt(false);
    const select = screen.getByLabelText("Terminologia") as HTMLSelectElement;
    expect(Array.from(select.options).map((o) => o.value)).toEqual([ "ciap2" ]);
  });
});
```

```tsx
// src/modules/consultation/StructuredFields.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState, type ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, searchSigtap: vi.fn() };
});

import * as api from "../../lib/api";
import type { ExamRequest } from "../../lib/api";
import { ConductsField } from "./ConductsField";
import { ExamRequestsField } from "./ExamRequestsField";
import { options } from "../../test/consultationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function Conducts() {
  const [ value, setValue ] = useState<string[]>([]);
  return (<><ConductsField options={options().conducts} value={value} onChange={setValue} /><pre data-testid="v">{JSON.stringify(value)}</pre></>);
}
function Exams({ cid10Allowed }: { cid10Allowed: boolean }) {
  const [ value, setValue ] = useState<ExamRequest[]>([]);
  return (<><ExamRequestsField value={value} onChange={setValue} cid10Allowed={cid10Allowed} searchDelayMs={0} /><pre data-testid="v">{JSON.stringify(value)}</pre></>);
}
function withQuery(ui: ReactNode) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}
const value = () => JSON.parse(screen.getByTestId("v").textContent ?? "null");

describe("condutas e exames", () => {
  it("condutas pelas opções do api", () => {
    render(<Conducts />);
    fireEvent.click(screen.getByLabelText("Alta do episódio"));
    fireEvent.click(screen.getByLabelText("Retorno para consulta agendada"));
    expect(value()).toEqual([ "12", "9" ]);
    fireEvent.click(screen.getByLabelText("Alta do episódio"));
    expect(value()).toEqual([ "9" ]);
  });

  it("exame pela busca SIGTAP, sem repetir, com a justificativa CID-10 conferida", async () => {
    mocked(api.searchSigtap).mockResolvedValue([ { code: "0202010503", label: "DOSAGEM DE HEMOGLOBINA GLICOSILADA" } ]);
    withQuery(<Exams cid10Allowed />);
    fireEvent.change(screen.getByLabelText("Solicitar exame (SIGTAP)"), { target: { value: "hemoglobina" } });
    fireEvent.click(await screen.findByRole("button", { name: "0202010503 — DOSAGEM DE HEMOGLOBINA GLICOSILADA" }));
    fireEvent.change(screen.getByLabelText("Solicitar exame (SIGTAP)"), { target: { value: "hemoglobina" } });
    fireEvent.click(await screen.findByRole("button", { name: "0202010503 — DOSAGEM DE HEMOGLOBINA GLICOSILADA" }));
    expect(value()).toEqual([ { sigtap_code: "0202010503", label: "DOSAGEM DE HEMOGLOBINA GLICOSILADA" } ]);
    const just = screen.getByLabelText("CID-10 de justificativa (0202010503)");
    fireEvent.change(just, { target: { value: "E1" } });
    expect(screen.getByRole("alert").textContent).toBe("use um código CID-10, ex.: E11 ou E119");
    fireEvent.change(just, { target: { value: "e11.9" } });
    expect(screen.queryByRole("alert")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Remover exame 0202010503" }));
    expect(value()).toEqual([]);
  });

  it("sem CID-10 para o CBO, não há justificativa", async () => {
    mocked(api.searchSigtap).mockResolvedValue([ { code: "0202010503", label: "DOSAGEM DE HEMOGLOBINA GLICOSILADA" } ]);
    withQuery(<Exams cid10Allowed={false} />);
    fireEvent.change(screen.getByLabelText("Solicitar exame (SIGTAP)"), { target: { value: "hemoglobina" } });
    fireEvent.click(await screen.findByRole("button", { name: "0202010503 — DOSAGEM DE HEMOGLOBINA GLICOSILADA" }));
    expect(screen.queryByLabelText("CID-10 de justificativa (0202010503)")).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/modules/consultation/ProblemsEditor.test.tsx src/modules/consultation/StructuredFields.test.tsx`
Expected: FAIL — `Failed to resolve import "./ProblemsEditor"` (e `./ConductsField`, `./ExamRequestsField`).

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/consultation/OnsetInput.tsx
// Início de um problema com precisão (dia, mês ou ano; spec §3). "Hoje" é o da
// cidade, recebido como texto: nenhum fuso desloca o dia.
import { useState, type CSSProperties } from "react";
import type { OnsetPrecision } from "../../lib/api";
import { PRECISIONS, PRECISION_LABEL, onsetInputValue, parseOnset, type Onset } from "../../lib/consultation";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";

interface Props { code: string; today: string; initial?: Onset | null; onApply(onset: Onset): void; onCancel(): void }

export function OnsetInput({ code, today, initial, onApply, onCancel }: Props) {
  const [ precision, setPrecision ] = useState<OnsetPrecision>(initial?.onset_precision ?? "month");
  const [ text, setText ] = useState(initial ? onsetInputValue(initial.onset_on, initial.onset_precision) : "");
  const [ problem, setProblem ] = useState<string | null>(null);

  function apply() {
    const result = parseOnset(precision, text, today);
    if ("problem" in result) { setProblem(result.problem); return; }
    onApply(result);
  }

  return (
    <div role="group" aria-label={`Início de ${code}`} style={box}>
      <label style={label}>
        Precisão
        <select value={precision} style={inputStyle}
          onChange={(e) => { setPrecision(e.target.value as OnsetPrecision); setText(""); setProblem(null); }}>
          {PRECISIONS.map((p) => <option key={p} value={p}>{PRECISION_LABEL[p]}</option>)}
        </select>
      </label>
      <label style={label}>
        Início
        <input type={precision === "day" ? "date" : precision === "month" ? "month" : "text"}
          inputMode={precision === "year" ? "numeric" : undefined} placeholder={precision === "year" ? "ex.: 2019" : undefined}
          value={text} max={precision === "day" ? today : undefined} style={inputStyle}
          onChange={(e) => { setText(e.target.value); setProblem(null); }} />
      </label>
      {problem && <small role="alert" style={alert}>{problem}</small>}
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" style={buttonStyle} onClick={apply}>Aplicar início</button>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </div>
  );
}

const box: CSSProperties = { display: "flex", flexDirection: "column", gap: 6, padding: 10, border: "1px dashed var(--rule)", borderRadius: 6, maxWidth: 320 };
const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const alert: CSSProperties = { margin: 0, fontSize: 12, color: "var(--down)" };
```

```tsx
// src/modules/consultation/ProblemsEditor.tsx
// Lista de problemas na consulta (módulo 19; spec §3 e §7; ADR 0031): o
// profissional avalia, resolve ou corrige o início dos problemas ativos e
// inclui novos pela busca CIAP-2 (ou CID-10, quando o CBO pode). A lista do
// paciente só muda pelo api, na finalização ou no adendo; aqui se monta o
// pedido (`evaluated_problems`).
import { useState, type CSSProperties } from "react";
import { searchTerminology, type CodedOption, type EvaluatedProblem, type PatientProblem, type Terminology } from "../../lib/api";
import {
  ACTION_LABEL, TERMINOLOGY_LABEL, addProblem, consultationError, correctOnset, markProblem, onsetLabel, problemKey,
  removeItem, setItemOnset
} from "../../lib/consultation";
import { Tag } from "../../components/Tag";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { CodeSearch } from "./CodeSearch";
import { OnsetInput } from "./OnsetInput";

interface Props {
  patientProblems: PatientProblem[];
  items: EvaluatedProblem[];
  onChange(next: EvaluatedProblem[]): void;
  cid10Allowed: boolean;
  today: string;
  searchDelayMs?: number;
}

export function ProblemsEditor({ patientProblems, items, onChange, cid10Allowed, today, searchDelayMs }: Props) {
  const [ terminology, setTerminology ] = useState<Terminology>("ciap2");
  const [ editingOnset, setEditingOnset ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const active = patientProblems.filter((p) => p.status === "active");
  const added = items.filter((i) => i.problem_id === null);

  function pick(ref: CodedOption) {
    const result = addProblem(items, patientProblems, terminology, ref);
    setNotice(result.notice);
    if (result.items !== items) onChange(result.items);
  }

  return (
    <fieldset style={fieldset}>
      <legend style={legend}>Problemas e condições</legend>
      {notice && <p role="status" style={muted}>{notice}</p>}

      <strong style={sub}>Lista do paciente</strong>
      {active.length === 0 ? <p style={muted}>nenhum problema ativo</p> : (
        <ul aria-label="problemas ativos do paciente" style={list}>
          {active.map((p) => {
            const item = items.find((i) => i.problem_id === p.id);
            return (
              <li key={p.id} style={row}>
                <span><span className="mono">{p.code}</span>{` — ${p.label} · ${onsetLabel(p.onset_on, p.onset_precision)}`}</span>
                {item && <Tag tone="info">{ACTION_LABEL[item.action]}</Tag>}
                {item?.action === "correct_onset" && (
                  <span style={muted}>{`novo início: ${onsetLabel(item.onset_on, item.onset_precision)}`}</span>
                )}
                <span style={actions}>
                  <button type="button" aria-label={`Avaliar ${p.code}`} style={secondaryButtonStyle}
                    onClick={() => onChange(markProblem(items, p, "evaluate"))}>Avaliar</button>
                  <button type="button" aria-label={`Resolver ${p.code}`} style={secondaryButtonStyle}
                    onClick={() => onChange(markProblem(items, p, "resolve"))}>Resolver</button>
                  <button type="button" aria-label={`Corrigir início ${p.code}`} style={secondaryButtonStyle}
                    onClick={() => setEditingOnset(p.id)}>Corrigir início</button>
                  {item && (
                    <button type="button" aria-label={`Desfazer ${p.code}`} style={secondaryButtonStyle}
                      onClick={() => onChange(removeItem(items, problemKey(item)))}>Desfazer</button>
                  )}
                </span>
                {editingOnset === p.id && (
                  <OnsetInput code={p.code} today={today}
                    initial={p.onset_on && p.onset_precision ? { onset_on: p.onset_on, onset_precision: p.onset_precision } : null}
                    onApply={(onset) => { onChange(correctOnset(items, p, onset)); setEditingOnset(null); }}
                    onCancel={() => setEditingOnset(null)} />
                )}
              </li>
            );
          })}
        </ul>
      )}

      <strong style={sub}>Incluídos nesta consulta</strong>
      {added.length === 0 ? <p style={muted}>nenhum problema novo</p> : (
        <ul aria-label="problemas incluídos nesta consulta" style={list}>
          {added.map((i) => {
            const key = problemKey(i);
            return (
              <li key={key} style={row}>
                <span>
                  <span className="mono">{i.code}</span>
                  {` — ${i.label} · ${TERMINOLOGY_LABEL[i.terminology]} · ${onsetLabel(i.onset_on, i.onset_precision)}`}
                </span>
                <span style={actions}>
                  <button type="button" aria-label={`Informar início ${i.code}`} style={secondaryButtonStyle}
                    onClick={() => setEditingOnset(key)}>Informar início</button>
                  <button type="button" aria-label={`Remover ${i.code}`} style={secondaryButtonStyle}
                    onClick={() => onChange(removeItem(items, key))}>Remover</button>
                </span>
                {editingOnset === key && (
                  <OnsetInput code={i.code} today={today}
                    onApply={(onset) => { onChange(setItemOnset(items, key, onset)); setEditingOnset(null); }}
                    onCancel={() => setEditingOnset(null)} />
                )}
              </li>
            );
          })}
        </ul>
      )}

      <label style={{ ...labelStyle, maxWidth: 200 }}>
        Terminologia
        <select value={terminology} style={inputStyle} onChange={(e) => setTerminology(e.target.value as Terminology)}>
          <option value="ciap2">CIAP-2</option>
          {cid10Allowed && <option value="cid10">CID-10</option>}
        </select>
      </label>
      <CodeSearch key={terminology} label={`Incluir problema (${TERMINOLOGY_LABEL[terminology]})`}
        placeholder="nome ou código, ex.: diabetes, T90" queryKey={`terminologySearch:${terminology}`}
        search={(q) => searchTerminology(q, terminology)} errorText={consultationError} onPick={pick} delayMs={searchDelayMs} />
    </fieldset>
  );
}

const fieldset: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, border: "1px solid var(--rule)", borderRadius: 8, padding: 12 };
const legend: CSSProperties = { fontSize: 12.5, fontWeight: 600 };
const sub: CSSProperties = { fontSize: 12 };
const list: CSSProperties = { listStyle: "none", margin: 0, padding: 0, display: "flex", flexDirection: "column", gap: 6 };
const row: CSSProperties = { display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap", fontSize: 12.5 };
const actions: CSSProperties = { display: "flex", gap: 6, marginLeft: "auto", flexWrap: "wrap" };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
```

```tsx
// src/modules/consultation/ConductsField.tsx
// Condutas da consulta: códigos LEDI vindos de consultation_options (contratos §4).
import type { CSSProperties } from "react";
import type { CodedOption } from "../../lib/api";

interface Props { options: CodedOption[]; value: string[]; onChange(next: string[]): void }

export function ConductsField({ options, value, onChange }: Props) {
  return (
    <fieldset style={fieldset}>
      <legend style={legend}>Condutas</legend>
      {options.map((o) => (
        <label key={o.code} style={{ display: "flex", gap: 8, alignItems: "center", fontSize: 12.5 }}>
          <input type="checkbox" checked={value.includes(o.code)}
            onChange={(e) => onChange(e.target.checked ? [ ...value, o.code ] : value.filter((c) => c !== o.code))} />
          {o.label}
        </label>
      ))}
    </fieldset>
  );
}

const fieldset: CSSProperties = { display: "flex", flexDirection: "column", gap: 6, border: "1px solid var(--rule)", borderRadius: 8, padding: 12 };
const legend: CSSProperties = { fontSize: 12.5, fontWeight: 600 };
```

```tsx
// src/modules/consultation/ExamRequestsField.tsx
// Exames solicitados (SIGTAP da competência ativa; contratos §4 e §5). A
// justificativa CID-10 só aparece para o CBO que pode usar CID-10 (Divergência D9).
import type { CSSProperties } from "react";
import { searchSigtap, type ExamRequest } from "../../lib/api";
import { addExam, cid10Problem, consultationError, setJustification } from "../../lib/consultation";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { CodeSearch } from "./CodeSearch";

interface Props { value: ExamRequest[]; onChange(next: ExamRequest[]): void; cid10Allowed: boolean; searchDelayMs?: number }

export function ExamRequestsField({ value, onChange, cid10Allowed, searchDelayMs }: Props) {
  return (
    <fieldset style={fieldset}>
      <legend style={legend}>Exames solicitados</legend>
      {value.length > 0 && (
        <ul aria-label="exames solicitados" style={{ listStyle: "none", margin: 0, padding: 0, display: "flex", flexDirection: "column", gap: 8 }}>
          {value.map((e) => {
            const problem = cid10Allowed ? cid10Problem(e.cid10_justification) : null;
            return (
              <li key={e.sigtap_code} style={{ display: "flex", gap: 8, alignItems: "flex-end", flexWrap: "wrap", fontSize: 12.5 }}>
                <span style={{ flex: 1, minWidth: 200 }}><span className="mono">{e.sigtap_code}</span>{` — ${e.label}`}</span>
                {cid10Allowed && (
                  <div style={{ display: "flex", flexDirection: "column", gap: 4, maxWidth: 220 }}>
                    <label style={labelStyle}>
                      {`CID-10 de justificativa (${e.sigtap_code})`}
                      <input value={e.cid10_justification ?? ""} placeholder="opcional, ex.: E11" style={inputStyle}
                        onChange={(ev) => onChange(setJustification(value, e.sigtap_code, ev.target.value))} />
                    </label>
                    {problem && <small role="alert" style={{ fontSize: 12, color: "var(--down)" }}>{problem}</small>}
                  </div>
                )}
                <button type="button" aria-label={`Remover exame ${e.sigtap_code}`} style={secondaryButtonStyle}
                  onClick={() => onChange(value.filter((x) => x.sigtap_code !== e.sigtap_code))}>Remover</button>
              </li>
            );
          })}
        </ul>
      )}
      <CodeSearch label="Solicitar exame (SIGTAP)" placeholder="nome ou código do procedimento" queryKey="sigtapSearch"
        search={searchSigtap} errorText={consultationError} onPick={(ref) => onChange(addExam(value, ref))} delayMs={searchDelayMs} />
    </fieldset>
  );
}

const fieldset: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, border: "1px solid var(--rule)", borderRadius: 8, padding: 12 };
const legend: CSSProperties = { fontSize: 12.5, fontWeight: 600 };
const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/modules/consultation/ProblemsEditor.test.tsx src/modules/consultation/StructuredFields.test.tsx`
Expected: PASS (5 + 3 testes).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/modules/consultation/OnsetInput.tsx src/modules/consultation/ProblemsEditor.tsx src/modules/consultation/ConductsField.tsx src/modules/consultation/ExamRequestsField.tsx src/modules/consultation/ProblemsEditor.test.tsx src/modules/consultation/StructuredFields.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "feat: edit problems, conducts and exam requests in a consultation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 6: Salvamento automático que não perde digitação

**Files:**
- Create: `src/lib/useAutosave.ts`
- Test: `src/lib/useAutosave.test.tsx`

**Interfaces:**
- Consumes: `fmtHourMinute` (`src/lib/format.ts`).
- Produces:
  - `type SaveStatus = { kind: "idle" } | { kind: "saving" } | { kind: "saved"; at: Date } | { kind: "error"; message: string }`;
  - `interface AutosaveOptions<T> { value: T; initialKey: string; blockedReason: string | null; enabled: boolean; delayMs: number; save(value: T): Promise<unknown>; describe(err: unknown): string; onError?(err: unknown): void; now?(): Date }`;
  - `useAutosave<T>(options): { status: SaveStatus; flush(): Promise<boolean> }` — compara pelo `JSON.stringify(value)`; salva depois de `delayMs` sem mudança; um salvamento por vez, e ao terminar salva de novo se o valor mudou no meio; a resposta nunca volta para o formulário; `flush()` espera o que estiver em curso e salva o que faltar (true = o servidor tem o valor atual); `blockedReason` ou `enabled: false` impedem salvar; ao desmontar, salva o que estiver pendente;
  - `saveStatusLabel(status, blockedReason): string` — "rascunho", "salvando…", "salvo às HH:MM", "não salvo — <frase>".

- [ ] **Step 1: Write the failing test**

```tsx
// src/lib/useAutosave.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { useState } from "react";
import { saveStatusLabel, useAutosave } from "./useAutosave";
import { NOW19 } from "../test/consultationFixtures";

afterEach(cleanup);

function deferred() {
  let resolve!: () => void;
  let reject!: (err: unknown) => void;
  const promise = new Promise<void>((res, rej) => { resolve = res; reject = rej; });
  return { promise, resolve, reject };
}

function Harness({ save, delayMs = 0, blocked = null }: { save(v: { text: string }): Promise<unknown>; delayMs?: number; blocked?: string | null }) {
  const [ text, setText ] = useState("a");
  const [ flushed, setFlushed ] = useState("");
  const autosave = useAutosave({
    value: { text }, initialKey: JSON.stringify({ text: "a" }), blockedReason: blocked, enabled: true, delayMs,
    save, describe: () => "falhou", now: () => new Date(NOW19)
  });
  return (
    <>
      <input aria-label="texto" value={text} onChange={(e) => setText(e.target.value)} />
      <span data-testid="status">{saveStatusLabel(autosave.status, blocked)}</span>
      <button type="button" onClick={() => void autosave.flush().then((ok) => setFlushed(String(ok)))}>flush</button>
      <span data-testid="flushed">{flushed}</span>
    </>
  );
}
const type = (value: string) => fireEvent.change(screen.getByLabelText("texto"), { target: { value } });
const status = () => screen.getByTestId("status").textContent;

describe("useAutosave", () => {
  it("o valor que o servidor já tem não é salvo", async () => {
    const save = vi.fn(async () => undefined);
    render(<Harness save={save} />);
    await new Promise((r) => setTimeout(r, 20));
    expect(save).not.toHaveBeenCalled();
    expect(status()).toBe("rascunho");
  });

  it("salva depois da pausa e diz a hora", async () => {
    const save = vi.fn(async () => undefined);
    render(<Harness save={save} />);
    type("ab");
    await waitFor(() => expect(save).toHaveBeenCalledWith({ text: "ab" }));
    await waitFor(() => expect(status()).toBe("salvo às 10:00"));
  });

  it("texto digitado durante o salvamento não se perde e é salvo em seguida", async () => {
    const first = deferred();
    const save = vi.fn().mockReturnValueOnce(first.promise).mockResolvedValue(undefined);
    render(<Harness save={save} />);
    type("ab");
    await waitFor(() => expect(save).toHaveBeenCalledTimes(1));
    expect(status()).toBe("salvando…");
    type("abc");
    first.resolve();
    await waitFor(() => expect(save).toHaveBeenCalledTimes(2));
    expect(save.mock.calls[1][0]).toEqual({ text: "abc" });
    await waitFor(() => expect(status()).toBe("salvo às 10:00"));
    expect((screen.getByLabelText("texto") as HTMLInputElement).value).toBe("abc");
  });

  it("flush espera o salvamento em curso e salva o que faltava", async () => {
    const first = deferred();
    const save = vi.fn().mockReturnValueOnce(first.promise).mockResolvedValue(undefined);
    render(<Harness save={save} delayMs={60_000} />);
    type("ab");
    fireEvent.click(screen.getByRole("button", { name: "flush" }));
    await waitFor(() => expect(save).toHaveBeenCalledTimes(1));
    type("abc");
    fireEvent.click(screen.getByRole("button", { name: "flush" }));
    first.resolve();
    await waitFor(() => expect(screen.getByTestId("flushed").textContent).toBe("true"));
    expect(save).toHaveBeenCalledTimes(2);
    expect(save.mock.calls[1][0]).toEqual({ text: "abc" });
  });

  it("bloqueado não salva e diz o motivo", async () => {
    const save = vi.fn(async () => undefined);
    render(<Harness save={save} blocked="Plano (P) passa de 20.000 caracteres" />);
    type("ab");
    fireEvent.click(screen.getByRole("button", { name: "flush" }));
    await waitFor(() => expect(screen.getByTestId("flushed").textContent).toBe("false"));
    expect(save).not.toHaveBeenCalled();
    expect(status()).toBe("não salvo — Plano (P) passa de 20.000 caracteres");
  });

  it("erro mostra a frase e a próxima mudança tenta de novo", async () => {
    const save = vi.fn().mockRejectedValueOnce(new Error("x")).mockResolvedValue(undefined);
    render(<Harness save={save} />);
    type("ab");
    await waitFor(() => expect(status()).toBe("não salvo — falhou"));
    type("abc");
    await waitFor(() => expect(status()).toBe("salvo às 10:00"));
    expect(save).toHaveBeenLastCalledWith({ text: "abc" });
  });

  it("fechar a tela com mudança pendente salva", async () => {
    const save = vi.fn(async () => undefined);
    const { unmount } = render(<Harness save={save} delayMs={60_000} />);
    type("ab");
    unmount();
    await waitFor(() => expect(save).toHaveBeenCalledWith({ text: "ab" }));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/useAutosave.test.tsx`
Expected: FAIL — `Failed to resolve import "./useAutosave"`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/useAutosave.ts
// Salvamento automático do rascunho da consulta (módulo 19; spec §4). Regras:
// - um salvamento por vez; se o valor mudou enquanto salvava, salva de novo
//   com o mais novo (nunca manda um valor velho por último);
// - a resposta do servidor nunca volta para o formulário (o que a pessoa
//   digitou durante o salvamento fica);
// - `flush()` espera o que estiver em curso e salva o que faltar — a
//   finalização chama antes de fechar a consulta;
// - `blockedReason` (texto longo demais) e `enabled: false` (consulta travada)
//   impedem salvar; ao desmontar, salva o pendente.
// O valor não é guardado em nenhum outro lugar (nem localStorage).
import { useCallback, useEffect, useRef, useState } from "react";
import { fmtHourMinute } from "./format";

export type SaveStatus =
  | { kind: "idle" }
  | { kind: "saving" }
  | { kind: "saved"; at: Date }
  | { kind: "error"; message: string };

export interface AutosaveOptions<T> {
  value: T;
  initialKey: string;
  blockedReason: string | null;
  enabled: boolean;
  delayMs: number;
  save(value: T): Promise<unknown>;
  describe(err: unknown): string;
  onError?(err: unknown): void;
  now?(): Date;
}

export function useAutosave<T>(options: AutosaveOptions<T>): { status: SaveStatus; flush(): Promise<boolean> } {
  const key = JSON.stringify(options.value);
  const latest = useRef({ key, value: options.value });
  latest.current = { key, value: options.value };
  const opts = useRef(options);
  opts.current = options;
  const savedKey = useRef(options.initialKey);
  const running = useRef<Promise<boolean> | null>(null);
  const [ status, setStatus ] = useState<SaveStatus>({ kind: "idle" });

  const pump = useCallback((): Promise<boolean> => {
    if (running.current) return running.current;
    const run = (async () => {
      // Cede a vez antes de olhar o valor: `running.current` já aponta para
      // esta execução quando o corpo roda (senão um retorno imediato limparia
      // a referência antes de ela ser gravada).
      await null;
      try {
        for (;;) {
          const { enabled, blockedReason, save, describe, onError, now } = opts.current;
          if (!enabled || blockedReason) return false;
          const { key: k, value: v } = latest.current;
          if (k === savedKey.current) return true;
          setStatus({ kind: "saving" });
          try {
            await save(v);
          } catch (err) {
            setStatus({ kind: "error", message: describe(err) });
            onError?.(err);
            return false;
          }
          savedKey.current = k;
          setStatus({ kind: "saved", at: now ? now() : new Date() });
        }
      } finally {
        running.current = null;
      }
    })();
    running.current = run;
    return run;
  }, []);

  useEffect(() => {
    if (!options.enabled || options.blockedReason || key === savedKey.current) return;
    const id = setTimeout(() => { void pump(); }, options.delayMs);
    return () => clearTimeout(id);
  }, [ key, options.enabled, options.blockedReason, options.delayMs, pump ]);

  // Fechar a tela (ou trocar de atendimento) não perde o que ficou pendente.
  useEffect(() => () => { void pump(); }, [ pump ]);

  return { status, flush: pump };
}

export function saveStatusLabel(status: SaveStatus, blockedReason: string | null): string {
  if (blockedReason) return `não salvo — ${blockedReason}`;
  if (status.kind === "saving") return "salvando…";
  if (status.kind === "saved") return `salvo às ${fmtHourMinute(status.at.toISOString())}`;
  if (status.kind === "error") return `não salvo — ${status.message}`;
  return "rascunho";
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/useAutosave.test.tsx`
Expected: PASS (7 testes).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/lib/useAutosave.ts src/lib/useAutosave.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "feat: autosave drafts without losing text typed during a save

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 7: Editor da consulta — S, O, A, P, sinais, itens estruturados, salvamento e Finalizar

**Files:**
- Create: `src/modules/consultation/ConsultationEditor.tsx`
- Test: `src/modules/consultation/ConsultationEditor.test.tsx`

**Interfaces:**
- Consumes: `saveConsultationDraft`, `finalizeConsultation`, `errorCode`, tipos (Task 1); `SOAP_FIELDS`, `TEXT_MAX_LABEL`, `draftFrom`, `checkDraft`, `blockedReason`, `finalizeProblems`, `consultationError`, `ConsultationDraft` (Task 2); `EMPTY_OUTCOME`, `outcomeBody`, `outcomeProblem`, `outcomeView`, `OutcomeDraft`, `OutcomeFields` (Task 3); `ProblemsEditor`, `ConductsField`, `ExamRequestsField` (Task 5); `useAutosave`, `saveStatusLabel` (Task 6); `VitalSignsFields`, `bmiOf` (módulo 18); `todayInCity` (`src/lib/campaigns.ts`).
- Produces: `interface ConsultationEditorProps { consultation: Consultation; record: ClinicalRecord; options: ConsultationOptions; referenceUnitIds?: string[]; unit: HealthUnit; units: HealthUnit[]; autosaveDelayMs?: number; searchDelayMs?: number; onFinalized(c: Consultation): void; onLocked(message: string): void }` e `ConsultationEditor(props)` — seção "Consulta" com a situação do rascunho (`role="status"`), as quatro áreas de texto ("Subjetivo (S)" … "Plano (P)"), os `VitalSignsFields` do 18, "Tipo de atendimento", os itens estruturados e "Finalizar consulta", que abre a seção "Finalizar consulta" com o `OutcomeFields` (`idPrefix="consultation-"`), a lista "o que falta para finalizar" e "Confirmar finalização"/"Voltar ao rascunho". 409 `not_draft` e 403 `not_author` (no salvamento ou na finalização) travam o editor e chamam `onLocked` com a frase. Espera padrão do salvamento: 1500 ms.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/consultation/ConsultationEditor.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, saveConsultationDraft: vi.fn(), finalizeConsultation: vi.fn(), searchTerminology: vi.fn(), searchSigtap: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { ConsultationEditor } from "./ConsultationEditor";
import { NOW19, consultation, finalized, options, record } from "../../test/consultationFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };
const units = [ unit, { id: "u2", name: "Ambulatório de Especialidades", kind: "other" } ];

function renderEditor(autosaveDelayMs = 0) {
  const onFinalized = vi.fn();
  const onLocked = vi.fn();
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<ConsultationEditor consultation={consultation()} record={record()} options={options()} unit={unit} units={units}
    autosaveDelayMs={autosaveDelayMs} searchDelayMs={0} onFinalized={onFinalized} onLocked={onLocked} />, { wrapper });
  return { onFinalized, onLocked };
}
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });
function fillMinimum() {
  fireEvent.click(screen.getByRole("button", { name: "Avaliar T90" }));
  fireEvent.click(screen.getByLabelText("Retorno para consulta agendada"));
  text("Plano (P)", "Ajuste de dose; retorno em 30 dias.");
}

describe("ConsultationEditor", () => {
  beforeEach(() => {
    for (const fn of [ api.saveConsultationDraft, api.finalizeConsultation, api.searchTerminology, api.searchSigtap ]) mocked(fn).mockReset();
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW19));
    mocked(api.saveConsultationDraft).mockImplementation(async () => consultation());
    mocked(api.finalizeConsultation).mockResolvedValue(finalized());
  });

  it("escrever no Subjetivo salva sozinho e mostra a hora", async () => {
    renderEditor();
    expect(screen.getByRole("status").textContent).toBe("rascunho");
    text("Subjetivo (S)", "Refere sede e cansaço há duas semanas.");
    await waitFor(() => expect(api.saveConsultationDraft).toHaveBeenCalledWith("cs1",
      expect.objectContaining({ subjective: "Refere sede e cansaço há duas semanas.", care_type: "5" })));
    await waitFor(() => expect(screen.getByRole("status").textContent).toBe("salvo às 10:00"));
  });

  it("Finalizar lista o que falta e não chama a API", () => {
    renderEditor();
    fireEvent.click(screen.getByRole("button", { name: "Finalizar consulta" }));
    const section = screen.getByRole("region", { name: "Finalizar consulta" });
    const missing = within(section).getByRole("list", { name: "o que falta para finalizar" });
    expect(within(missing).getAllByRole("listitem").map((li) => li.textContent)).toEqual([
      "avalie, inclua ou resolva ao menos um problema", "marque ao menos uma conduta", "escreva a avaliação (A) ou o plano (P)"
    ]);
    expect((within(section).getByRole("button", { name: "Confirmar finalização" }) as HTMLButtonElement).disabled).toBe(true);
    expect(api.finalizeConsultation).not.toHaveBeenCalled();
  });

  it("Finalizar espera o salvamento pendente e manda o rascunho mais novo antes", async () => {
    const { onFinalized } = renderEditor(60_000);
    fillMinimum();
    fireEvent.click(screen.getByRole("button", { name: "Finalizar consulta" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar finalização" }));
    await waitFor(() => expect(onFinalized).toHaveBeenCalledWith(finalized()));
    expect(api.saveConsultationDraft).toHaveBeenCalledTimes(1);
    expect(mocked(api.saveConsultationDraft).mock.calls[0][1]).toMatchObject({
      plan: "Ajuste de dose; retorno em 30 dias.", conducts: [ "9" ],
      evaluated_problems: [ { problem_id: "pp1", action: "evaluate" } ]
    });
    expect(mocked(api.saveConsultationDraft).mock.invocationCallOrder[0])
      .toBeLessThan(mocked(api.finalizeConsultation).mock.invocationCallOrder[0]);
    expect(api.finalizeConsultation).toHaveBeenCalledWith("cs1", { outcome: "discharged" });
  });

  it("o desfecho vai como no Encerrar: encaminhamento com a unidade", async () => {
    renderEditor(60_000);
    fillMinimum();
    fireEvent.click(screen.getByRole("button", { name: "Finalizar consulta" }));
    const section = screen.getByRole("region", { name: "Finalizar consulta" });
    fireEvent.change(within(section).getByLabelText("Desfecho"), { target: { value: "referred" } });
    expect(within(section).getByRole("list", { name: "o que falta para finalizar" }).textContent)
      .toBe("informe a unidade de destino ou a descrição do encaminhamento");
    fireEvent.change(within(section).getByLabelText("Unidade de destino"), { target: { value: "u2" } });
    fireEvent.click(within(section).getByRole("button", { name: "Confirmar finalização" }));
    await waitFor(() => expect(api.finalizeConsultation).toHaveBeenCalledWith("cs1", { outcome: "referred", referral_unit_id: "u2" }));
  });

  it("422 patient_name_missing vira a frase com o caminho", async () => {
    mocked(api.finalizeConsultation).mockRejectedValue(new ApiError(422, { error: "patient_name_missing" }, "422"));
    const { onFinalized } = renderEditor(60_000);
    fillMinimum();
    fireEvent.click(screen.getByRole("button", { name: "Finalizar consulta" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar finalização" }));
    expect((await screen.findByRole("alert")).textContent)
      .toBe("falta o nome completo do paciente — peça à recepção para completar os nomes no check-in e tente de novo");
    expect(onFinalized).not.toHaveBeenCalled();
  });

  it("not_draft no salvamento trava o editor e avisa", async () => {
    mocked(api.saveConsultationDraft).mockRejectedValue(new ApiError(409, { error: "not_draft" }, "409"));
    const { onLocked } = renderEditor();
    text("Objetivo (O)", "Bom estado geral.");
    await waitFor(() => expect(onLocked).toHaveBeenCalledWith("esta consulta já foi finalizada — a tela foi atualizada"));
    expect((screen.getByLabelText("Objetivo (O)") as HTMLTextAreaElement).disabled).toBe(true);
  });

  it("sinal vital fora do plausível não vai no rascunho e trava a finalização", async () => {
    renderEditor();
    text("Pressão sistólica (mmHg)", "400");
    fillMinimum();
    await waitFor(() => expect(api.saveConsultationDraft).toHaveBeenCalled());
    const last = mocked(api.saveConsultationDraft).mock.calls.at(-1)?.[1];
    expect(last.vitals.systolic).toBeUndefined();
    fireEvent.click(screen.getByRole("button", { name: "Finalizar consulta" }));
    expect(screen.getByRole("list", { name: "o que falta para finalizar" }).textContent).toContain("corrija os sinais vitais marcados");
  });

  it("texto acima de 20.000 não é salvo e diz qual campo", async () => {
    renderEditor();
    text("Plano (P)", "x".repeat(20_001));
    expect(screen.getByRole("status").textContent).toBe("não salvo — Plano (P) passa de 20.000 caracteres");
    await new Promise((r) => setTimeout(r, 20));
    expect(api.saveConsultationDraft).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/modules/consultation/ConsultationEditor.test.tsx`
Expected: FAIL — `Failed to resolve import "./ConsultationEditor"`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/consultation/ConsultationEditor.tsx
// Rascunho da consulta (módulo 19; spec §4 e §7; contratos §4): S, O, A, P em
// texto, sinais vitais (o componente do acolhimento), tipo de atendimento,
// problemas, condutas e exames estruturados. O rascunho se salva sozinho e só
// a autora o vê. "Finalizar" pede o desfecho com a mesma tela do "Encerrar",
// espera o salvamento em curso e manda; a consulta finalizada não muda mais.
import { useState, type CSSProperties } from "react";
import {
  errorCode, finalizeConsultation, saveConsultationDraft,
  type ClinicalRecord, type Consultation, type ConsultationOptions, type HealthUnit
} from "../../lib/api";
import {
  SOAP_FIELDS, TEXT_MAX_LABEL, blockedReason, checkDraft, consultationError, draftFrom, finalizeProblems, type ConsultationDraft
} from "../../lib/consultation";
import { EMPTY_OUTCOME, outcomeBody, outcomeProblem, outcomeView, type OutcomeDraft } from "../../lib/outcome";
import { bmiOf } from "../../lib/screening";
import { todayInCity } from "../../lib/campaigns";
import { saveStatusLabel, useAutosave } from "../../lib/useAutosave";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { VitalSignsFields } from "../attendance/VitalSignsFields";
import { OutcomeFields } from "../attendance/OutcomeFields";
import { ProblemsEditor } from "./ProblemsEditor";
import { ConductsField } from "./ConductsField";
import { ExamRequestsField } from "./ExamRequestsField";

export interface ConsultationEditorProps {
  consultation: Consultation;
  record: ClinicalRecord;
  options: ConsultationOptions;
  referenceUnitIds?: string[];
  unit: HealthUnit;
  units: HealthUnit[];
  autosaveDelayMs?: number;
  searchDelayMs?: number;
  onFinalized(c: Consultation): void;
  onLocked(message: string): void;
}

// A consulta mudou de dono ou de estado fora desta tela: o editor para.
const LOCKING = new Set([ "not_draft", "not_author" ]);

export function ConsultationEditor(props: ConsultationEditorProps) {
  const { consultation, record, options, unit, units } = props;
  const [ draft, setDraft ] = useState<ConsultationDraft>(() => draftFrom(consultation));
  const [ initialKey ] = useState(() => JSON.stringify(checkDraft(draftFrom(consultation)).input));
  const [ locked, setLocked ] = useState(false);
  const [ finalizing, setFinalizing ] = useState(false);
  const [ outcome, setOutcome ] = useState<OutcomeDraft>(EMPTY_OUTCOME);
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);

  function lock(err: unknown): boolean {
    const code = errorCode(err);
    if (!code || !LOCKING.has(code)) return false;
    setLocked(true);
    props.onLocked(consultationError(err));
    return true;
  }

  const check = checkDraft(draft);
  const blocked = blockedReason(check);
  const autosave = useAutosave({
    value: check.input, initialKey, blockedReason: blocked, enabled: !locked, delayMs: props.autosaveDelayMs ?? 1500,
    save: (input) => saveConsultationDraft(consultation.id, input),
    describe: consultationError,
    onError: (err) => { lock(err); }
  });

  const today = todayInCity();
  const { referralUnitId } = outcomeView(outcome, props.referenceUnitIds, unit, units);
  const outcomeIssue = outcomeProblem(outcome, referralUnitId);
  const missing = [ ...finalizeProblems(check), ...(outcomeIssue ? [ outcomeIssue ] : []) ];
  const set = (patch: Partial<ConsultationDraft>) => setDraft((d) => ({ ...d, ...patch }));

  async function confirm() {
    if (busy || locked || missing.length > 0) return;
    setBusy(true); setError(null);
    try {
      if (!(await autosave.flush())) {
        setError("o rascunho não foi salvo — veja o aviso no topo da consulta e tente de novo");
        return;
      }
      props.onFinalized(await finalizeConsultation(consultation.id, outcomeBody(outcome, referralUnitId)));
    } catch (err) {
      if (!lock(err)) setError(consultationError(err));
    } finally {
      setBusy(false);
    }
  }

  const blockedFinal = busy || locked || missing.length > 0;

  return (
    <section aria-label="Consulta" style={panel}>
      <div style={{ display: "flex", gap: 12, alignItems: "baseline", flexWrap: "wrap" }}>
        <strong>Consulta — rascunho</strong>
        <span role="status" style={muted}>{saveStatusLabel(autosave.status, blocked)}</span>
      </div>
      <p style={muted}>
        O rascunho é salvo sozinho e só você o vê. Finalizada, a consulta não muda mais: correção é por adendo.
      </p>

      {SOAP_FIELDS.map((f) => (
        <label key={f.key} style={labelStyle}>
          {f.label}
          <textarea value={draft.soap[f.key]} rows={3} disabled={locked} style={inputStyle}
            onChange={(e) => { const v = e.target.value; setDraft((d) => ({ ...d, soap: { ...d.soap, [f.key]: v } })); }} />
          {check.tooLong.includes(f.key) && (
            <small style={alert}>{`${f.label} passa de ${TEXT_MAX_LABEL} caracteres`}</small>
          )}
        </label>
      ))}

      <VitalSignsFields form={draft.vitals} problems={check.vitalsProblems} alerts={[]}
        bmi={bmiOf(check.input.vitals.weight_kg, check.input.vitals.height_cm)} onChange={(vitals) => set({ vitals })} />

      <label style={{ ...labelStyle, maxWidth: 320 }}>
        Tipo de atendimento
        <select value={draft.careType} style={inputStyle} onChange={(e) => set({ careType: e.target.value })}>
          <option value="">—</option>
          {options.care_types.map((o) => <option key={o.code} value={o.code}>{o.label}</option>)}
        </select>
      </label>

      <ProblemsEditor patientProblems={record.problems} items={draft.problems} onChange={(problems) => set({ problems })}
        cid10Allowed={options.cid10_allowed_for_cbo} today={today} searchDelayMs={props.searchDelayMs} />
      <ConductsField options={options.conducts} value={draft.conducts} onChange={(conducts) => set({ conducts })} />
      <ExamRequestsField value={draft.exams} onChange={(exams) => set({ exams })} cid10Allowed={options.cid10_allowed_for_cbo}
        searchDelayMs={props.searchDelayMs} />

      {!finalizing ? (
        <div>
          <button type="button" disabled={locked} style={locked ? disabledButtonStyle : buttonStyle} onClick={() => setFinalizing(true)}>
            Finalizar consulta
          </button>
        </div>
      ) : (
        <section aria-label="Finalizar consulta" style={box}>
          <strong>Finalizar consulta</strong>
          <p style={muted}>
            Finalizar encerra o atendimento com o desfecho abaixo e gera a ficha do e-SUS. Encaminhamento e retorno geram o pedido de agendamento.
          </p>
          <OutcomeFields idPrefix="consultation-" value={outcome} onChange={setOutcome}
            referenceIds={props.referenceUnitIds} unit={unit} units={units} />
          {missing.length > 0 && (
            <ul aria-label="o que falta para finalizar" style={{ margin: 0, paddingLeft: 18 }}>
              {missing.map((m) => <li key={m} style={alert}>{m}</li>)}
            </ul>
          )}
          {error && <p role="alert" style={alert}>{error}</p>}
          <div style={{ display: "flex", gap: 8 }}>
            <button type="button" disabled={blockedFinal} style={blockedFinal ? disabledButtonStyle : buttonStyle}
              onClick={() => void confirm()}>
              Confirmar finalização
            </button>
            <button type="button" disabled={busy} style={secondaryButtonStyle} onClick={() => setFinalizing(false)}>
              Voltar ao rascunho
            </button>
          </div>
        </section>
      )}
    </section>
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 12, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 };
const box: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 12, border: "1px solid var(--rule2)", borderRadius: 8 };
const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/modules/consultation/ConsultationEditor.test.tsx`
Expected: PASS (8 testes).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/modules/consultation/ConsultationEditor.tsx src/modules/consultation/ConsultationEditor.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "feat: write and finalize a SOAP consultation with autosave

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 8: Painel do paciente, consulta finalizada, Adendo e Imprimir

**Files:**
- Create: `src/modules/consultation/PatientPanel.tsx`, `src/modules/consultation/ConsultationView.tsx`, `src/modules/consultation/AddendumForm.tsx`
- Test: `src/modules/consultation/PatientPanel.test.tsx`, `src/modules/consultation/ConsultationView.test.tsx`, `src/modules/consultation/AddendumForm.test.tsx`

**Interfaces:**
- Consumes: `getConsultation`, `addAddendum`, `fetchConsultationPdf`, `errorCode`, tipos (Task 1); `ACTION_LABEL`, `CONSULTATION_KEY`, `SOAP_FIELDS`, `ageLabel`, `codedLabel`, `onsetLabel`, `changesLines`, `consultationError`, `addendumProblem`, `addendumChanges`, `examsProblem` (Task 2); `ProblemsEditor`, `ConductsField`, `ExamRequestsField` (Task 5); `VITALS`, `GLUCOSE_MOMENT_LABEL`, `COLOR_LABEL`, `COLOR_TONE`, `alertLabel` (módulo 18); `sexLabel` (`src/lib/profile.ts`); `todayInCity`; `KeyValue`, `Tag`, `DataTable`, `formStyles`.
- Produces:
  - `VitalsList({ vitals: VitalSigns; label: string })` — `<ul>` com "Pressão sistólica: 185 mmHg", momento da glicemia e IMC (nada quando não há sinal);
  - `PatientPanel({ record: ClinicalRecord; onOpenConsultation(id: string): void })` — seção "Paciente": "Nome" (só o nome de exibição), "Idade", "Sexo", "CPF"; "Problemas ativos"; "Escuta de hoje" (cor, CIAP-2, queixa, sinais, alertas) ou "sem escuta concluída hoje"; "Últimas consultas" com o botão "Abrir consulta de <data e hora>";
  - `ConsultationView({ consultation; options: ConsultationOptions | null; patientProblems: PatientProblem[]; canAddendum: boolean; openingId?: string; searchDelayMs?: number; onAddendumAdded(): void; onClose(): void; onOpeningRequired?(): void })` — seção "Consulta finalizada" com S/O/A/P, sinais, "problemas avaliados", condutas, "exames solicitados", "adendos" em ordem; botões "Imprimir", "Adendo" (só com `canAddendum`) e "Fechar consulta";
  - `ConsultationLoader({ id; canAddendum(c: Consultation): boolean; …as demais props da View sem consultation/canAddendum/onAddendumAdded })` — lê pelo `GET` (chave `[ CONSULTATION_KEY, id ]`, `gcTime: 0`), relê depois do adendo, e chama `onOpeningRequired` no 403 `opening_required`;
  - `AddendumForm({ consultation; options; patientProblems; openingId?; searchDelayMs?; onDone(): void; onCancel(): void; onOpeningRequired?(): void })` — seção "Adendo" com "Motivo do adendo", "Texto do adendo", a caixa "Mudar problemas, condutas ou exames" (abre `ProblemsEditor`, `ConductsField` e `ExamRequestsField`, partindo das condutas e exames da consulta) e "Registrar adendo"; manda `changes` só com o que mudou e `opening_id` quando recebido.
- Imprimir: abre uma janela em branco **no clique** (o navegador não bloqueia), busca o PDF com `fetchConsultationPdf`, aponta a janela para um `blob:` e revoga o endereço depois de 60 s; na recusa, fecha a janela e mostra a frase.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/consultation/PatientPanel.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
import { PatientPanel } from "./PatientPanel";
import { record } from "../../test/consultationFixtures";

afterEach(cleanup);

describe("PatientPanel", () => {
  it("nome de exibição, idade, sexo, problemas, escuta de hoje e últimas consultas", () => {
    const onOpen = vi.fn();
    render(<PatientPanel record={record()} onOpenConsultation={onOpen} />);
    const panel = screen.getByRole("region", { name: "Paciente" });
    expect(within(panel).getByText("Joana Lima")).not.toBeNull();
    expect(within(panel).queryByText("João Carlos Lima")).toBeNull();
    expect(within(panel).getByText("54 anos")).not.toBeNull();
    expect(within(panel).getByText("feminino")).not.toBeNull();
    expect(within(panel).getByText(/Diabetes não insulino-dependente · desde 03\/2019/)).not.toBeNull();
    expect(within(panel).getByText("vermelho")).not.toBeNull();
    expect(within(panel).getByText("Pressão sistólica: 185 mmHg")).not.toBeNull();
    fireEvent.click(within(panel).getByRole("button", { name: "Abrir consulta de 10/09/2026, 14:30" }));
    expect(onOpen).toHaveBeenCalledWith("cs0");
  });

  it("sem escuta nem consultas anteriores", () => {
    render(<PatientPanel record={record({ today_screening: null, consultations: [], problems: [] })} onOpenConsultation={vi.fn()} />);
    expect(screen.getByText("sem escuta concluída hoje")).not.toBeNull();
    expect(screen.getByText("nenhum problema ativo")).not.toBeNull();
    expect(screen.getByText("nenhuma consulta anterior")).not.toBeNull();
  });
});
```

```tsx
// src/modules/consultation/ConsultationView.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchConsultationPdf: vi.fn(), getConsultation: vi.fn(), addAddendum: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { ConsultationLoader, ConsultationView } from "./ConsultationView";
import { finalized, options, problem } from "../../test/consultationFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
let win: { location: { href: string }; close: ReturnType<typeof vi.fn> };

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

function wrap(ui: ReactNode) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}
const addendum = { id: "ad1", author_name: "Enf. Lúcia Prado", created_at: "2026-10-07T11:00:00-03:00",
  reason: "correção do plano", text: "Retorno em 15 dias.", changes: { conducts: [ "12" ] } };

describe("ConsultationView", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchConsultationPdf, api.getConsultation, api.addAddendum ]) mocked(fn).mockReset();
    win = { location: { href: "" }, close: vi.fn() };
    vi.spyOn(window, "open").mockImplementation(() => win as unknown as Window);
    // O jsdom não tem createObjectURL/revokeObjectURL.
    Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:pdf-1") });
    Object.defineProperty(URL, "revokeObjectURL", { configurable: true, value: vi.fn() });
  });

  it("mostra o registro finalizado e os adendos em ordem", () => {
    wrap(<ConsultationView consultation={finalized({ addenda: [ addendum ] })} options={options()} patientProblems={[ problem() ]}
      canAddendum={false} onAddendumAdded={vi.fn()} onClose={vi.fn()} />);
    const view = screen.getByRole("region", { name: "Consulta finalizada" });
    expect(within(view).getByText("Consulta de 07/10/2026, 10:20")).not.toBeNull();
    expect(within(view).getByText("Diabetes descompensado.")).not.toBeNull();
    expect(within(view).getByText("Glicemia capilar: 280 mg/dL")).not.toBeNull();
    expect(within(view).getByText("T90 — Diabetes não insulino-dependente · avaliado")).not.toBeNull();
    expect(within(view).getByText("Condutas: Retorno para consulta agendada")).not.toBeNull();
    const list = within(view).getByRole("list", { name: "adendos" });
    expect(within(list).getByText("Adendo de Enf. Lúcia Prado em 07/10/2026, 11:00")).not.toBeNull();
    expect(within(list).getByText("Condutas: Alta do episódio")).not.toBeNull();
    expect(within(view).queryByRole("button", { name: "Adendo" })).toBeNull();
  });

  it("Imprimir abre o PDF numa janela nova", async () => {
    mocked(api.fetchConsultationPdf).mockResolvedValue(new Blob([ "%PDF" ], { type: "application/pdf" }));
    wrap(<ConsultationView consultation={finalized()} options={options()} patientProblems={[]} canAddendum onAddendumAdded={vi.fn()} onClose={vi.fn()} />);
    fireEvent.click(screen.getByRole("button", { name: "Imprimir" }));
    expect(window.open).toHaveBeenCalledWith("", "_blank");
    await waitFor(() => expect(win.location.href).toBe("blob:pdf-1"));
    expect(api.fetchConsultationPdf).toHaveBeenCalledWith("cs1");
  });

  it("Imprimir sem o nome do paciente fecha a janela e diz o caminho", async () => {
    mocked(api.fetchConsultationPdf).mockRejectedValue(new ApiError(409, { error: "patient_name_missing" }, "409"));
    wrap(<ConsultationView consultation={finalized()} options={options()} patientProblems={[]} canAddendum onAddendumAdded={vi.fn()} onClose={vi.fn()} />);
    fireEvent.click(screen.getByRole("button", { name: "Imprimir" }));
    expect((await screen.findByRole("alert")).textContent)
      .toBe("falta o nome completo do paciente — peça à recepção para completar os nomes no check-in e tente de novo");
    expect(win.close).toHaveBeenCalled();
  });

  it("o loader lê pelo id e avisa quando a abertura acabou", async () => {
    mocked(api.getConsultation).mockRejectedValue(new ApiError(403, { error: "opening_required" }, "403"));
    const onOpeningRequired = vi.fn();
    wrap(<ConsultationLoader id="cs1" canAddendum={() => true} options={options()} patientProblems={[]} onClose={vi.fn()}
      onOpeningRequired={onOpeningRequired} />);
    await waitFor(() => expect(onOpeningRequired).toHaveBeenCalled());
    expect(api.getConsultation).toHaveBeenCalledWith("cs1");
  });
});
```

```tsx
// src/modules/consultation/AddendumForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, addAddendum: vi.fn(), searchTerminology: vi.fn(), searchSigtap: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { AddendumForm } from "./AddendumForm";
import { finalized, options, problem } from "../../test/consultationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function renderForm(openingId?: string) {
  const onDone = vi.fn();
  const onOpeningRequired = vi.fn();
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<AddendumForm consultation={finalized()} options={options()} patientProblems={[ problem() ]} openingId={openingId}
    searchDelayMs={0} onDone={onDone} onCancel={vi.fn()} onOpeningRequired={onOpeningRequired} />, { wrapper });
  return { onDone, onOpeningRequired };
}
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

describe("AddendumForm", () => {
  beforeEach(() => {
    mocked(api.addAddendum).mockReset();
    mocked(api.addAddendum).mockResolvedValue({ id: "ad1" });
  });

  it("motivo curto trava o envio", () => {
    renderForm();
    text("Motivo do adendo", "curto");
    text("Texto do adendo", "Retorno em 15 dias.");
    expect(screen.getByRole("alert").textContent).toBe("o motivo do adendo precisa de pelo menos 10 caracteres");
    expect((screen.getByRole("button", { name: "Registrar adendo" }) as HTMLButtonElement).disabled).toBe(true);
  });

  it("sem mudanças, só motivo e texto", async () => {
    const { onDone } = renderForm();
    text("Motivo do adendo", "correção do plano");
    text("Texto do adendo", "Retorno em 15 dias.");
    fireEvent.click(screen.getByRole("button", { name: "Registrar adendo" }));
    await waitFor(() => expect(onDone).toHaveBeenCalled());
    expect(api.addAddendum).toHaveBeenCalledWith("cs1", { reason: "correção do plano", text: "Retorno em 15 dias." });
  });

  it("com mudanças: só vai o que mudou, e a abertura quando há", async () => {
    renderForm("op1");
    text("Motivo do adendo", "correção do plano");
    text("Texto do adendo", "Problema resolvido; alta.");
    fireEvent.click(screen.getByLabelText("Mudar problemas, condutas ou exames"));
    fireEvent.click(screen.getByRole("button", { name: "Resolver T90" }));
    fireEvent.click(screen.getByLabelText("Alta do episódio"));
    fireEvent.click(screen.getByRole("button", { name: "Registrar adendo" }));
    await waitFor(() => expect(api.addAddendum).toHaveBeenCalled());
    expect(mocked(api.addAddendum).mock.calls[0][1]).toEqual({
      reason: "correção do plano", text: "Problema resolvido; alta.", opening_id: "op1",
      changes: {
        evaluated_problems: [ { problem_id: "pp1", terminology: "ciap2", code: "T90", label: problem().label, action: "resolve" } ],
        conducts: [ "9", "12" ]
      }
    });
  });

  it("403 opening_required avisa quem abriu a leitura", async () => {
    mocked(api.addAddendum).mockRejectedValue(new ApiError(403, { error: "opening_required" }, "403"));
    const { onOpeningRequired } = renderForm("op1");
    text("Motivo do adendo", "correção do plano");
    text("Texto do adendo", "Retorno em 15 dias.");
    fireEvent.click(screen.getByRole("button", { name: "Registrar adendo" }));
    expect((await screen.findByRole("alert")).textContent)
      .toBe("a abertura justificada terminou ou não existe — abra o prontuário de novo com o motivo");
    expect(onOpeningRequired).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/modules/consultation/PatientPanel.test.tsx src/modules/consultation/ConsultationView.test.tsx src/modules/consultation/AddendumForm.test.tsx`
Expected: FAIL — `Failed to resolve import "./PatientPanel"` (e `./ConsultationView`, `./AddendumForm`).

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/consultation/PatientPanel.tsx
// Painel do paciente no atendimento e na abertura justificada (módulo 19;
// spec §7; contratos §3). Mostra só o nome de exibição (social, senão o
// completo): o nome de registro não aparece aqui.
import type { CSSProperties } from "react";
import type { ClinicalRecord, ConsultationSummary, VitalSigns } from "../../lib/api";
import { ageLabel, onsetLabel } from "../../lib/consultation";
import { COLOR_LABEL, COLOR_TONE, GLUCOSE_MOMENT_LABEL, VITALS, alertLabel } from "../../lib/screening";
import { sexLabel } from "../../lib/profile";
import { fmtDateTime, fmtNumber } from "../../lib/format";
import { KeyValue } from "../../components/KeyValue";
import { Tag } from "../../components/Tag";
import { DataTable } from "../../components/DataTable";
import { secondaryButtonStyle } from "../../components/formStyles";

export function VitalsList({ vitals, label }: { vitals: VitalSigns; label: string }) {
  const measured = VITALS.filter((v) => typeof vitals[v.key] === "number");
  if (measured.length === 0 && !vitals.glucose_moment && typeof vitals.bmi !== "number") return null;
  return (
    <ul aria-label={label} style={{ margin: 0, paddingLeft: 18, fontSize: 12.5 }}>
      {measured.map((v) => (
        <li key={v.key}>{`${v.label}: ${fmtNumber(vitals[v.key] as number)}${v.unit ? ` ${v.unit}` : ""}`}</li>
      ))}
      {vitals.glucose_moment && <li>{`Momento da glicemia: ${GLUCOSE_MOMENT_LABEL[vitals.glucose_moment]}`}</li>}
      {typeof vitals.bmi === "number" && <li>{`IMC: ${fmtNumber(vitals.bmi)}`}</li>}
    </ul>
  );
}

export function PatientPanel({ record, onOpenConsultation }: { record: ClinicalRecord; onOpenConsultation(id: string): void }) {
  const { patient, problems, consultations } = record;
  const active = problems.filter((p) => p.status === "active");
  const rev = record.today_screening?.current_revision ?? null;

  return (
    <section aria-label="Paciente" style={panel}>
      <div style={{ display: "flex", gap: 24, flexWrap: "wrap" }}>
        <KeyValue k="Nome" v={patient.display_name} mono={false} />
        <KeyValue k="Idade" v={ageLabel(patient.age)} />
        <KeyValue k="Sexo" v={sexLabel(patient.sex)} />
        <KeyValue k="CPF" v={patient.cpf_masked} />
      </div>

      <div style={block}>
        <strong style={sub}>Problemas ativos</strong>
        {active.length === 0 ? <p style={muted}>nenhum problema ativo</p> : (
          <ul aria-label="problemas ativos" style={{ margin: 0, paddingLeft: 18, fontSize: 12.5 }}>
            {active.map((p) => (
              <li key={p.id}><span className="mono">{p.code}</span>{` — ${p.label} · ${onsetLabel(p.onset_on, p.onset_precision)}`}</li>
            ))}
          </ul>
        )}
      </div>

      <div style={block}>
        <strong style={sub}>Escuta de hoje</strong>
        {rev ? (
          <div style={{ display: "flex", flexDirection: "column", gap: 4, fontSize: 12.5 }}>
            <span style={{ display: "flex", gap: 8, alignItems: "center" }}>
              <Tag tone={COLOR_TONE[rev.final_color]}>{COLOR_LABEL[rev.final_color]}</Tag>
              <span><span className="mono">{rev.ciap2.code}</span>{` — ${rev.ciap2.label}`}</span>
            </span>
            {rev.complaint_note && <span>{`Queixa: ${rev.complaint_note}`}</span>}
            <VitalsList vitals={rev.vitals} label="sinais da escuta de hoje" />
            {rev.alerts.length > 0 && (
              <span style={{ color: "var(--down)", fontWeight: 600 }}>{rev.alerts.map(alertLabel).join(" · ")}</span>
            )}
          </div>
        ) : <p style={muted}>sem escuta concluída hoje</p>}
      </div>

      <div style={block}>
        <strong style={sub}>Últimas consultas</strong>
        <DataTable<ConsultationSummary>
          cols={[
            { label: "Data", w: "1fr", render: (c) => fmtDateTime(c.finalized_at) },
            { label: "Profissional", w: "1.6fr", render: (c) => `${c.author_name} · ${c.cbo_label}` },
            { label: "Tipo", w: "1fr", render: (c) => c.care_type_label },
            { label: "Problemas", w: "1fr", render: (c) => c.problems.map((p) => p.code).join(", ") || "—" },
            { label: "Adendos", w: "0.6fr", align: "right", render: (c) => String(c.addenda_count) },
            {
              label: "", w: "auto", align: "right", render: (c) => (
                <button type="button" aria-label={`Abrir consulta de ${fmtDateTime(c.finalized_at)}`} style={secondaryButtonStyle}
                  onClick={() => onOpenConsultation(c.id)}>
                  Abrir
                </button>
              )
            }
          ]}
          rows={consultations}
          rowKey={(c) => c.id}
          empty="nenhuma consulta anterior"
        />
      </div>
    </section>
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 12, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 };
const block: CSSProperties = { display: "flex", flexDirection: "column", gap: 6 };
const sub: CSSProperties = { fontSize: 12.5 };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
```

```tsx
// src/modules/consultation/ConsultationView.tsx
// Consulta finalizada (módulo 19; spec §4 e §7; contratos §4): leitura, adendos
// em ordem, Adendo e Imprimir. Ler pelo GET gera a trilha no api. O PDF é lido
// com a sessão e aberto numa janela nova por um endereço `blob:` (nada do
// conteúdo passa pela URL da aplicação).
import { useEffect, useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import {
  errorCode, fetchConsultationPdf, getConsultation,
  type Consultation, type ConsultationOptions, type PatientProblem
} from "../../lib/api";
import { ACTION_LABEL, CONSULTATION_KEY, SOAP_FIELDS, changesLines, codedLabel, consultationError } from "../../lib/consultation";
import { fmtDateTime } from "../../lib/format";
import { buttonStyle, disabledButtonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { VitalsList } from "./PatientPanel";
import { AddendumForm } from "./AddendumForm";

export interface ConsultationViewProps {
  consultation: Consultation;
  options: ConsultationOptions | null;
  patientProblems: PatientProblem[];
  canAddendum: boolean;
  openingId?: string;
  searchDelayMs?: number;
  onAddendumAdded(): void;
  onClose(): void;
  onOpeningRequired?(): void;
}

const PDF_URL_TTL_MS = 60_000;
const POPUP_BLOCKED = "o navegador bloqueou a janela nova — permita janelas deste site para imprimir";

export function ConsultationView(props: ConsultationViewProps) {
  const { consultation: c, options } = props;
  const [ adding, setAdding ] = useState(false);
  const [ printing, setPrinting ] = useState(false);
  const [ printError, setPrintError ] = useState<string | null>(null);
  const conductLabel = (code: string) => codedLabel(options?.conducts, code);
  const addenda = [ ...c.addenda ].sort((a, b) => a.created_at.localeCompare(b.created_at));

  async function print() {
    if (printing) return;
    setPrinting(true); setPrintError(null);
    // Aberta no clique: depois de um await, o navegador trataria como pop-up.
    const win = window.open("", "_blank");
    try {
      const blob = await fetchConsultationPdf(c.id);
      if (!win) { setPrintError(POPUP_BLOCKED); return; }
      const url = URL.createObjectURL(blob);
      win.location.href = url;
      setTimeout(() => URL.revokeObjectURL(url), PDF_URL_TTL_MS);
    } catch (err) {
      win?.close();
      if (errorCode(err) === "opening_required") props.onOpeningRequired?.();
      setPrintError(consultationError(err));
    } finally {
      setPrinting(false);
    }
  }

  return (
    <section aria-label="Consulta finalizada" style={panel}>
      <div style={{ display: "flex", gap: 12, alignItems: "baseline", flexWrap: "wrap" }}>
        <strong>{`Consulta de ${fmtDateTime(c.finalized_at)}`}</strong>
        <span style={muted}>{`${c.author.name} · ${codedLabel(options?.care_types, c.care_type)}`}</span>
      </div>

      {SOAP_FIELDS.map((f) => c[f.key] ? (
        <div key={f.key} style={block}>
          <strong style={sub}>{f.label}</strong>
          <p style={textStyle}>{c[f.key]}</p>
        </div>
      ) : null)}

      <VitalsList vitals={c.vitals} label="sinais vitais da consulta" />

      {c.evaluated_problems.length > 0 && (
        <ul aria-label="problemas avaliados" style={list}>
          {c.evaluated_problems.map((p) => (
            <li key={`${p.terminology}:${p.code}`}>{`${p.code} — ${p.label} · ${ACTION_LABEL[p.action]}`}</li>
          ))}
        </ul>
      )}
      <p style={textStyle}>{`Condutas: ${c.conducts.map(conductLabel).join(" · ") || "—"}`}</p>
      {c.exam_requests.length > 0 && (
        <ul aria-label="exames solicitados" style={list}>
          {c.exam_requests.map((e) => (
            <li key={e.sigtap_code}>
              {`${e.sigtap_code} — ${e.label}${e.cid10_justification ? ` · CID-10 ${e.cid10_justification}` : ""}`}
            </li>
          ))}
        </ul>
      )}

      {addenda.length > 0 && (
        <ol aria-label="adendos" style={{ margin: 0, paddingLeft: 18, display: "flex", flexDirection: "column", gap: 8 }}>
          {addenda.map((a) => (
            <li key={a.id} style={{ fontSize: 12.5 }}>
              <strong>{`Adendo de ${a.author_name} em ${fmtDateTime(a.created_at)}`}</strong>
              <p style={textStyle}>{`Motivo: ${a.reason}`}</p>
              <p style={textStyle}>{a.text}</p>
              {changesLines(a.changes, conductLabel).map((line) => <p key={line} style={textStyle}>{line}</p>)}
            </li>
          ))}
        </ol>
      )}

      {printError && <p role="alert" style={alert}>{printError}</p>}
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={printing} style={printing ? disabledButtonStyle : buttonStyle} onClick={() => void print()}>
          Imprimir
        </button>
        {props.canAddendum && !adding && (
          <button type="button" style={secondaryButtonStyle} onClick={() => setAdding(true)}>Adendo</button>
        )}
        <button type="button" style={secondaryButtonStyle} onClick={props.onClose}>Fechar consulta</button>
      </div>

      {adding && (
        <AddendumForm consultation={c} options={options} patientProblems={props.patientProblems} openingId={props.openingId}
          searchDelayMs={props.searchDelayMs}
          onDone={() => { setAdding(false); props.onAddendumAdded(); }}
          onCancel={() => setAdding(false)} onOpeningRequired={props.onOpeningRequired} />
      )}
    </section>
  );
}

type LoaderProps = Omit<ConsultationViewProps, "consultation" | "canAddendum" | "onAddendumAdded"> & {
  id: string;
  canAddendum(c: Consultation): boolean;
};

export function ConsultationLoader({ id, canAddendum, ...rest }: LoaderProps) {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: [ CONSULTATION_KEY, id ], queryFn: () => getConsultation(id), gcTime: 0, staleTime: 0 });
  const openingGone = query.isError && errorCode(query.error) === "opening_required";
  const { onOpeningRequired } = rest;

  useEffect(() => {
    if (openingGone) onOpeningRequired?.();
  }, [ openingGone, onOpeningRequired ]);

  if (query.isPending) return <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando a consulta…</p>;
  if (query.isError) return <p role="alert" style={alert}>{consultationError(query.error)}</p>;
  return (
    <ConsultationView consultation={query.data} canAddendum={canAddendum(query.data)}
      onAddendumAdded={() => void queryClient.invalidateQueries({ queryKey: [ CONSULTATION_KEY, id ] })} {...rest} />
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 };
const block: CSSProperties = { display: "flex", flexDirection: "column", gap: 2 };
const sub: CSSProperties = { fontSize: 12 };
const list: CSSProperties = { margin: 0, paddingLeft: 18, fontSize: 12.5 };
const textStyle: CSSProperties = { margin: 0, fontSize: 12.5, whiteSpace: "pre-wrap" };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

```tsx
// src/modules/consultation/AddendumForm.tsx
// Adendo (módulo 19; spec §4; ADR 0031): só acréscimo, com motivo, podendo
// mudar problemas, condutas e exames. Quem não é a autora só chega aqui com
// uma abertura justificada (opening_id).
import { useState, type CSSProperties } from "react";
import {
  addAddendum, errorCode, type Consultation, type ConsultationOptions, type EvaluatedProblem, type ExamRequest, type PatientProblem
} from "../../lib/api";
import { addendumChanges, addendumProblem, consultationError, examsProblem } from "../../lib/consultation";
import { todayInCity } from "../../lib/campaigns";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { ProblemsEditor } from "./ProblemsEditor";
import { ConductsField } from "./ConductsField";
import { ExamRequestsField } from "./ExamRequestsField";

interface Props {
  consultation: Consultation;
  options: ConsultationOptions | null;
  patientProblems: PatientProblem[];
  openingId?: string;
  searchDelayMs?: number;
  onDone(): void;
  onCancel(): void;
  onOpeningRequired?(): void;
}

export function AddendumForm({ consultation, options, patientProblems, openingId, searchDelayMs, onDone, onCancel, onOpeningRequired }: Props) {
  const [ reason, setReason ] = useState("");
  const [ text, setText ] = useState("");
  const [ withChanges, setWithChanges ] = useState(false);
  const [ problems, setProblems ] = useState<EvaluatedProblem[]>([]);
  const [ conducts, setConducts ] = useState<string[]>(consultation.conducts);
  const [ exams, setExams ] = useState<ExamRequest[]>(consultation.exam_requests);
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);

  const touched = reason !== "" || text !== "";
  const problem = addendumProblem(reason, text) ?? (withChanges ? examsProblem(exams) : null);
  const disabled = busy || problem !== null;

  async function submit() {
    if (disabled) return;
    setBusy(true); setError(null);
    try {
      const changes = withChanges ? addendumChanges(consultation, { problems, conducts, exams }) : undefined;
      await addAddendum(consultation.id, {
        reason: reason.trim(), text,
        ...(changes ? { changes } : {}),
        ...(openingId ? { opening_id: openingId } : {})
      });
      onDone();
    } catch (err) {
      if (errorCode(err) === "opening_required") onOpeningRequired?.();
      setError(consultationError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <section aria-label="Adendo" style={panel}>
      <strong>Adendo</strong>
      <p style={muted}>O adendo não muda a consulta: fica junto dela, com seu nome e a hora, e também não muda depois.</p>
      <label style={labelStyle}>
        Motivo do adendo
        <input value={reason} onChange={(e) => setReason(e.target.value)} style={inputStyle} />
      </label>
      <label style={labelStyle}>
        Texto do adendo
        <textarea value={text} rows={3} onChange={(e) => setText(e.target.value)} style={inputStyle} />
      </label>
      <label style={{ ...labelStyle, flexDirection: "row", alignItems: "center", gap: 8 }}>
        <input type="checkbox" checked={withChanges} onChange={(e) => setWithChanges(e.target.checked)} />
        Mudar problemas, condutas ou exames
      </label>
      {withChanges && (
        <>
          <ProblemsEditor patientProblems={patientProblems} items={problems} onChange={setProblems}
            cid10Allowed={options?.cid10_allowed_for_cbo ?? false} today={todayInCity()} searchDelayMs={searchDelayMs} />
          <ConductsField options={options?.conducts ?? []} value={conducts} onChange={setConducts} />
          <ExamRequestsField value={exams} onChange={setExams} cid10Allowed={options?.cid10_allowed_for_cbo ?? false}
            searchDelayMs={searchDelayMs} />
        </>
      )}
      {touched && problem && <p role="alert" style={alert}>{problem}</p>}
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={disabled} style={disabled ? disabledButtonStyle : buttonStyle} onClick={() => void submit()}>
          Registrar adendo
        </button>
        <button type="button" disabled={busy} style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </section>
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 12, border: "1px solid var(--rule2)", borderRadius: 8 };
const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/modules/consultation/PatientPanel.test.tsx src/modules/consultation/ConsultationView.test.tsx src/modules/consultation/AddendumForm.test.tsx`
Expected: PASS (2 + 4 + 4 testes).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/modules/consultation/PatientPanel.tsx src/modules/consultation/ConsultationView.tsx src/modules/consultation/AddendumForm.tsx src/modules/consultation/PatientPanel.test.tsx src/modules/consultation/ConsultationView.test.tsx src/modules/consultation/AddendumForm.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "feat: show the patient panel and finalized consultations with addenda and print

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 9: Consulta no atendimento chamado

**Files:**
- Create: `src/modules/consultation/ConsultationWorkspace.tsx`
- Modify: `src/modules/attendance/UnitQueue.tsx` (prop `clinicalRecord`, painel `consultation`, coluna "Nome", botão "Consulta", chamada abre a consulta), `src/modules/Attendance.tsx` (passa o interruptor)
- Test: `src/modules/consultation/ConsultationWorkspace.test.tsx`, `src/modules/attendance/UnitQueue.consultation.test.tsx`, `src/modules/Attendance.test.tsx`

**Interfaces:**
- Consumes: `getAttendanceRecord`, `getConsultationOptions`, `startConsultation`, `getConsultation`, tipos (Task 1); `RECORD_KEY`, `OPTIONS_KEY`, `consultationError`, `existingConsultationId` (Task 2); `ConsultationEditor` (Task 7); `PatientPanel`, `ConsultationView`, `ConsultationLoader` (Task 8); `useAuth`; `hasFeature`; da `UnitQueue` do módulo 18: `ScreeningPanel`, `panel`/`setPanel`, `CallResult`.
- Produces:
  - `ConsultationWorkspace({ attendanceId: string; referenceUnitIds?: string[]; unit: HealthUnit; units: HealthUnit[]; autosaveDelayMs?: number; searchDelayMs?: number; onClose(): void; onFinalized(): void })` — seção "Consulta do atendimento": lê o prontuário em contexto (chave `[ RECORD_KEY, attendanceId ]`, `gcTime: 0`) e as opções (chave `[ OPTIONS_KEY, user.id ]`); `PatientPanel`; "Iniciar consulta" (que também retoma: 409 `already_exists` com `consultation_id` → `GET` do rascunho); editor para o rascunho, `ConsultationView` para a finalizada (Adendo só para a autora); consulta anterior pelo `ConsultationLoader`; "Fechar painel da consulta";
  - `UnitQueue` ganha `clinicalRecord?: boolean` (padrão `false`): com ele e `canCare`, "Consulta" em cada linha de "Em atendimento", e "Chamar"/"Chamar próximo" abrem a consulta do atendimento chamado no lugar da escuta solta (a escuta do dia está no painel do paciente); com ele, a coluna "Nome" (`display_name`, Divergência D3) nas duas tabelas, também para a recepção; finalizar invalida `[ "unitQueue", unit.id ]` e `[ "unitRequests" ]` e mostra "Consulta finalizada e atendimento encerrado.";
  - `Attendance` passa `clinicalRecord={hasFeature(user, "clinical_record")}`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/consultation/ConsultationWorkspace.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen } from "@testing-library/react";
import type { Consultation } from "../../lib/api";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getAttendanceRecord: vi.fn(), getConsultationOptions: vi.fn(),
    startConsultation: vi.fn(), getConsultation: vi.fn() };
});
vi.mock("./ConsultationEditor", () => ({
  ConsultationEditor: (p: { consultation: Consultation; onFinalized(c: Consultation): void; onLocked(m: string): void }) => (
    <div>
      <span>{`editor ${p.consultation.id}`}</span>
      <button type="button" onClick={() => p.onFinalized({ ...p.consultation, status: "finalized" })}>finalizar (dublê)</button>
      <button type="button" onClick={() => p.onLocked("esta consulta já foi finalizada — a tela foi atualizada")}>travar (dublê)</button>
    </div>
  )
}));
vi.mock("./ConsultationView", () => ({
  ConsultationView: (p: { consultation: Consultation; canAddendum: boolean }) =>
    <span>{`consulta finalizada ${p.consultation.id} · adendo ${p.canAddendum ? "sim" : "não"}`}</span>,
  ConsultationLoader: (p: { id: string }) => <span>{`consulta anterior ${p.id}`}</span>
}));

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { ConsultationWorkspace } from "./ConsultationWorkspace";
import { renderWithProviders, sessionWith } from "../../test/campaignFixtures";
import { consultation, finalized, options, record } from "../../test/consultationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };

function renderIt() {
  const onFinalized = vi.fn();
  renderWithProviders(<ConsultationWorkspace attendanceId="a1" unit={unit} units={[ unit ]} onClose={vi.fn()} onFinalized={onFinalized} />);
  return { onFinalized };
}

describe("ConsultationWorkspace", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.getAttendanceRecord, api.getConsultationOptions, api.startConsultation, api.getConsultation ]) {
      mocked(fn).mockReset();
    }
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "clinical_record" ] }));
    mocked(api.getAttendanceRecord).mockResolvedValue(record());
    mocked(api.getConsultationOptions).mockResolvedValue(options());
    mocked(api.startConsultation).mockResolvedValue(consultation());
  });

  it("mostra o paciente e inicia a consulta", async () => {
    renderIt();
    expect(await screen.findByText("Joana Lima")).not.toBeNull();
    expect(api.getAttendanceRecord).toHaveBeenCalledWith("a1");
    fireEvent.click(screen.getByRole("button", { name: "Iniciar consulta" }));
    expect(await screen.findByText("editor cs1")).not.toBeNull();
    expect(api.startConsultation).toHaveBeenCalledWith("a1");
  });

  it("já iniciada: retoma o rascunho em vez de mostrar erro", async () => {
    mocked(api.startConsultation).mockRejectedValue(new ApiError(409, { error: "already_exists", consultation_id: "cs1" }, "409"));
    mocked(api.getConsultation).mockResolvedValue(consultation({ subjective: "Refere sede." }));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta" }));
    expect(await screen.findByText("editor cs1")).not.toBeNull();
    expect(api.getConsultation).toHaveBeenCalledWith("cs1");
    expect(screen.queryByRole("alert")).toBeNull();
  });

  it("cadastro não validado: diz a frase e não oferece iniciar", async () => {
    mocked(api.getAttendanceRecord).mockRejectedValue(new ApiError(409, { error: "citizen_not_verified" }, "409"));
    renderIt();
    expect((await screen.findByRole("alert")).textContent).toBe(
      "o cadastro desta pessoa não foi validado no balcão — a consulta exige a validação presencial; encerre o atendimento só com o desfecho");
    expect(screen.queryByRole("button", { name: "Iniciar consulta" })).toBeNull();
  });

  it("finalizada: leitura com adendo da autora, e avisa a fila", async () => {
    const { onFinalized } = renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta" }));
    fireEvent.click(await screen.findByRole("button", { name: "finalizar (dublê)" }));
    expect(await screen.findByText("consulta finalizada cs1 · adendo sim")).not.toBeNull();
    expect(onFinalized).toHaveBeenCalled();
  });

  it("travou (outra aba finalizou): relê a consulta e avisa", async () => {
    mocked(api.getConsultation).mockResolvedValue(finalized());
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta" }));
    fireEvent.click(await screen.findByRole("button", { name: "travar (dublê)" }));
    expect(await screen.findByText("consulta finalizada cs1 · adendo sim")).not.toBeNull();
    expect(screen.getByRole("status").textContent).toBe("esta consulta já foi finalizada — a tela foi atualizada");
  });

  it("abre uma consulta anterior pelo painel do paciente", async () => {
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Abrir consulta de 10/09/2026, 14:30" }));
    expect(screen.getByText("consulta anterior cs0")).not.toBeNull();
  });
});
```

```tsx
// src/modules/attendance/UnitQueue.consultation.test.tsx
// Módulo 19: consulta no atendimento chamado; nome de exibição e cor para a recepção.
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listUnitQueue: vi.fn(), callAttendance: vi.fn(), callNext: vi.fn(),
    closeAttendance: vi.fn(), getScreening: vi.fn(), getAttendanceRecord: vi.fn() };
});
vi.mock("../consultation/ConsultationWorkspace", () => ({
  ConsultationWorkspace: (p: { attendanceId: string; onFinalized(): void; onClose(): void }) => (
    <div>
      <span>{`consulta de ${p.attendanceId}`}</span>
      <button type="button" onClick={p.onFinalized}>finalizar (dublê)</button>
    </div>
  )
}));

import * as api from "../../lib/api";
import type { QueueRow } from "../../lib/api";
import { AuthProvider } from "../../lib/auth";
import { UnitQueue } from "./UnitQueue";
import { revision, screening } from "../../test/screeningFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };

function row(over: Partial<QueueRow>): QueueRow {
  return { id: "a1", cpf_masked: "***.982.247-**", checked_in_at: "2026-10-07T09:20:00-03:00", protocol_name: null, priority: null,
    source: "triage", appointment_time: null, called_at: null, called_by_name: null, screening: null, display_name: null, ...over };
}
const waitingRow = row({ id: "a1", display_name: "Joana Lima", screening: { id: "sc1", color: "red", destination: "same_day", waited_minutes: 25 } });
const inCareRow = row({ id: "a3", cpf_masked: "***.333.444-**", display_name: "Carlos Souza", called_at: "2026-10-07T09:50:00-03:00",
  called_by_name: "Dra. Helena" });

function session(role: string) {
  return { id: "us1", email_address: "x@cidade.gov.br", operator: false, mfa_enrolled: true, mfa_verified_at: null,
    memberships: [ { city_slug: "m1", city_name: "Curitiba", city_uf: "PR", role } ], features: [ "clinical_record" ] };
}
function renderQueue(canCare: boolean, clinicalRecord = true) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  render(<UnitQueue unit={unit} units={[ unit ]} canCare={canCare} clinicalRecord={clinicalRecord} />, { wrapper });
}
const section = async (title: string) => (await screen.findByText(title)).parentElement as HTMLElement;

describe("UnitQueue — consulta (módulo 19)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listUnitQueue, api.callAttendance, api.callNext, api.closeAttendance,
      api.getScreening, api.getAttendanceRecord ]) {
      mocked(fn).mockReset();
    }
    mocked(api.fetchCurrentSession).mockResolvedValue(session("health_professional"));
    mocked(api.listUnitQueue).mockResolvedValue({ waiting: [ waitingRow ], in_care: [ inCareRow ] });
  });

  it("quem cuida vê Consulta em Em atendimento e abre a consulta daquele atendimento", async () => {
    renderQueue(true);
    const inCare = await section("Em atendimento");
    fireEvent.click(await within(inCare).findByRole("button", { name: "Consulta" }));
    expect(screen.getByText("consulta de a3")).not.toBeNull();
  });

  it("Chamar abre a consulta do atendimento chamado, no lugar da escuta solta", async () => {
    mocked(api.callAttendance).mockResolvedValue({ attendance: { id: "a1" },
      screening: screening({ status: "completed", destination: "same_day", current_revision: revision(), revisions_count: 1 }) });
    renderQueue(true);
    const waiting = await section("Aguardando");
    fireEvent.click(await within(waiting).findByRole("button", { name: "Chamar" }));
    expect(await screen.findByText("consulta de a1")).not.toBeNull();
    expect(screen.queryByRole("region", { name: "Escuta inicial do atendimento" })).toBeNull();
  });

  it("recepção vê nome e cor, sem Consulta e sem ler o prontuário", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session("citizen_verifier"));
    renderQueue(false);
    const waiting = await section("Aguardando");
    expect(await within(waiting).findByText("Joana Lima")).not.toBeNull();
    expect(within(waiting).getByText("vermelho")).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Consulta" })).toBeNull();
    expect(screen.queryByText(/consulta de/)).toBeNull();
    expect(api.getAttendanceRecord).not.toHaveBeenCalled();
  });

  it("sem o interruptor: nem Consulta nem coluna Nome", async () => {
    renderQueue(true, false);
    const inCare = await section("Em atendimento");
    await within(inCare).findByRole("button", { name: "Encerrar" });
    expect(within(inCare).queryByRole("button", { name: "Consulta" })).toBeNull();
    expect(screen.queryByText("Nome")).toBeNull();
  });

  it("finalizar pela consulta recarrega a fila e avisa", async () => {
    renderQueue(true);
    const inCare = await section("Em atendimento");
    fireEvent.click(await within(inCare).findByRole("button", { name: "Consulta" }));
    fireEvent.click(screen.getByRole("button", { name: "finalizar (dublê)" }));
    expect(await screen.findByText("Consulta finalizada e atendimento encerrado.")).not.toBeNull();
    await waitFor(() => expect(api.listUnitQueue).toHaveBeenCalledTimes(2));
  });
});
```

E em `src/modules/Attendance.test.tsx`, um teste no `describe("Attendance")` (o mock de `listScreeningQueue` já está no arquivo desde o módulo 18; acrescente `getAttendanceRecord: vi.fn()` à lista do `vi.mock`):

```tsx
  it("com clinical_record, o profissional com vínculo vê Consulta em quem está em atendimento", async () => {
    const unit = { id: "un1", name: "UBS Centro", kind: "ubs" };
    mocked(api.fetchCurrentSession).mockResolvedValue({ ...session("health_professional"), features: [ "clinical_record" ] });
    localStorage.setItem(currentUnitKey("u1"), unit.id);
    mocked(api.listActiveUnits).mockResolvedValue([ unit ]);
    mocked(api.getMyProfessional).mockResolvedValue({
      professional: {} as api.Professional, shifts: [],
      links: [ { id: "l1", health_unit_id: "un1", unit_name: "UBS Centro", cbo_code: "225142", cbo_title: null,
        started_at: "x", started_by: "a", ended_at: null, ended_by: null } ]
    });
    mocked(api.listUnitQueue).mockResolvedValue({ waiting: [], in_care: [ {
      id: "a3", cpf_masked: "***.333.444-**", checked_in_at: "2026-10-07T09:20:00-03:00", protocol_name: null, priority: null,
      source: "triage", appointment_time: null, called_at: "2026-10-07T09:50:00-03:00", called_by_name: "Dra. Helena",
      screening: null, display_name: "Carlos Souza"
    } ] });
    renderAttendance();
    expect(await screen.findByRole("button", { name: "Consulta" })).not.toBeNull();
    expect(screen.getByText("Carlos Souza")).not.toBeNull();
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/modules/consultation/ConsultationWorkspace.test.tsx src/modules/attendance/UnitQueue.consultation.test.tsx src/modules/Attendance.test.tsx`
Expected: FAIL — `Failed to resolve import "./ConsultationWorkspace"`; na fila, `Unable to find role="button" and name "Consulta"`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/consultation/ConsultationWorkspace.tsx
// Consulta no atendimento chamado (módulo 19; spec §7; contratos §3 e §4): o
// prontuário em contexto (o api confere chamador, CBO e par validado e grava a
// trilha), o rascunho da consulta e a leitura da finalizada. "Iniciar consulta"
// também retoma: o 409 already_exists traz o id do rascunho (Divergência D1).
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import {
  getAttendanceRecord, getConsultation, getConsultationOptions, startConsultation, type Consultation, type HealthUnit
} from "../../lib/api";
import { OPTIONS_KEY, RECORD_KEY, consultationError, existingConsultationId } from "../../lib/consultation";
import { useAuth } from "../../lib/auth";
import { buttonStyle, disabledButtonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { PatientPanel } from "./PatientPanel";
import { ConsultationEditor } from "./ConsultationEditor";
import { ConsultationLoader, ConsultationView } from "./ConsultationView";

interface Props {
  attendanceId: string;
  referenceUnitIds?: string[];
  unit: HealthUnit;
  units: HealthUnit[];
  autosaveDelayMs?: number;
  searchDelayMs?: number;
  onClose(): void;
  onFinalized(): void;
}

export function ConsultationWorkspace(props: Props) {
  const { user } = useAuth();
  const record = useQuery({
    queryKey: [ RECORD_KEY, props.attendanceId ], queryFn: () => getAttendanceRecord(props.attendanceId), gcTime: 0, staleTime: 0
  });
  const options = useQuery({ queryKey: [ OPTIONS_KEY, user?.id ?? null ], queryFn: getConsultationOptions, staleTime: 5 * 60_000 });
  const [ consultation, setConsultation ] = useState<Consultation | null>(null);
  const [ viewing, setViewing ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const isMine = (c: Consultation) => c.author.id === user?.id;

  async function open() {
    if (busy) return;
    setBusy(true); setError(null);
    try {
      setConsultation(await startConsultation(props.attendanceId));
    } catch (err) {
      const existing = existingConsultationId(err);
      if (!existing) { setError(consultationError(err)); return; }
      try { setConsultation(await getConsultation(existing)); } catch (again) { setError(consultationError(again)); }
    } finally {
      setBusy(false);
    }
  }

  async function reload(message?: string) {
    if (message) setNotice(message);
    if (!consultation) return;
    try { setConsultation(await getConsultation(consultation.id)); } catch (err) { setError(consultationError(err)); }
  }

  return (
    <section aria-label="Consulta do atendimento" style={panel}>
      <div style={{ display: "flex", gap: 8, alignItems: "center" }}>
        <strong style={{ flex: 1 }}>Consulta do atendimento</strong>
        <button type="button" style={secondaryButtonStyle} onClick={props.onClose}>Fechar painel da consulta</button>
      </div>
      {notice && <p role="status" style={statusStyle}>{notice}</p>}
      {error && <p role="alert" style={alert}>{error}</p>}
      {record.isPending && <p className="mono" style={loading}>carregando o prontuário…</p>}
      {record.isError && <p role="alert" style={alert}>{consultationError(record.error)}</p>}

      {record.data && (
        <>
          <PatientPanel record={record.data} onOpenConsultation={setViewing} />
          {viewing && (
            <ConsultationLoader key={viewing} id={viewing} canAddendum={isMine} options={options.data ?? null}
              patientProblems={record.data.problems} searchDelayMs={props.searchDelayMs} onClose={() => setViewing(null)} />
          )}

          {!consultation && (
            <div>
              <button type="button" disabled={busy} style={busy ? disabledButtonStyle : buttonStyle} onClick={() => void open()}>
                Iniciar consulta
              </button>
            </div>
          )}

          {consultation?.status === "draft" && options.isError && <p role="alert" style={alert}>{consultationError(options.error)}</p>}
          {consultation?.status === "draft" && options.data && (
            <ConsultationEditor key={consultation.id} consultation={consultation} record={record.data} options={options.data}
              referenceUnitIds={props.referenceUnitIds} unit={props.unit} units={props.units}
              autosaveDelayMs={props.autosaveDelayMs} searchDelayMs={props.searchDelayMs}
              onFinalized={(c) => { setConsultation(c); props.onFinalized(); }}
              onLocked={(message) => void reload(message)} />
          )}

          {consultation?.status === "finalized" && (
            <ConsultationView consultation={consultation} options={options.data ?? null} patientProblems={record.data.problems}
              canAddendum={isMine(consultation)} searchDelayMs={props.searchDelayMs}
              onAddendumAdded={() => void reload()} onClose={props.onClose} />
          )}
        </>
      )}
    </section>
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 12, padding: 16, border: "1px solid var(--rule2)", borderRadius: 8 };
const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const loading: CSSProperties = { margin: 0, fontSize: 10.5, color: "var(--ink3)" };
```

Em `src/modules/attendance/UnitQueue.tsx` (sobre o estado do módulo 18 e da Task 3):

```diff
--- a/src/modules/attendance/UnitQueue.tsx
+++ b/src/modules/attendance/UnitQueue.tsx
@@ imports
 import { ScreeningForm } from "./ScreeningForm";
 import { OutcomeFields } from "./OutcomeFields";
+import { ConsultationWorkspace } from "../consultation/ConsultationWorkspace";
@@ interface Props {
   onClinicalRefused?(): void;
   now?(): Date;
+  // Módulo 19: `clinical_record` ligado na sessão (consulta e nome de exibição).
+  clinicalRecord?: boolean;
 }
@@ type ScreeningPanel =
   | { kind: "called"; screening: Screening }
   | { kind: "view"; id: string }
-  | { kind: "reassess"; row: QueueRow; screening: Screening };
+  | { kind: "reassess"; row: QueueRow; screening: Screening }
+  // Módulo 19: a consulta do atendimento chamado (traz a escuta do dia no painel do paciente).
+  | { kind: "consultation"; attendanceId: string };
@@
-export function UnitQueue({ unit, units, canCare, careBlocked, onClinicalRefused, now = () => new Date() }: Props) {
+export function UnitQueue({
+  unit, units, canCare, careBlocked, onClinicalRefused, now = () => new Date(), clinicalRecord = false
+}: Props) {
@@ async function onCallNext() {
       const result = await callNext(unit.id);
-      if (result.screening) setPanel({ kind: "called", screening: result.screening });
+      openCalled(result);
       invalidate();
@@ async function onCall(row: QueueRow) {
       const result = await callAttendance(row.id, unit.id);
-      if (result.screening) setPanel({ kind: "called", screening: result.screening });
+      openCalled(result);
       invalidate();
```

E, logo depois de `function invalidate() { … }`, a função que decide o que abre na chamada:

```tsx
  // Com o prontuário ligado, a chamada abre a consulta (a escuta do dia está
  // no painel do paciente); sem ele, a escuta abre como no módulo 18.
  function openCalled(result: CallResult) {
    if (clinicalRecord) setPanel({ kind: "consultation", attendanceId: result.attendance.id });
    else if (result.screening) setPanel({ kind: "called", screening: result.screening });
  }

  // Módulo 19 (Divergência D3): nome de exibição, também para a recepção.
  const nameCol = clinicalRecord
    ? [ { label: "Nome", w: "1.5fr", render: (r: QueueRow) => r.display_name ?? "—" } ]
    : [];
```

(`CallResult` entra no import de `../../lib/api`, ao lado de `type Screening`.)

Nas duas tabelas, a coluna "Nome" entra logo depois de "CPF":

```diff
                 <DataTable<QueueRow>
                   rowStyle={redRow}
                   cols={[
                     { label: "CPF", w: "1.5fr", render: (r) => r.cpf_masked },
+                    ...nameCol,
                     { label: "Cor", w: "0.8fr", render: (r) => colorTag(r) },
@@
                 <DataTable<QueueRow>
                   cols={[
                     { label: "CPF", w: "1.5fr", render: (r) => r.cpf_masked },
+                    ...nameCol,
                     { label: "Cor", w: "0.8fr", render: (r) => colorTag(r) },
```

Em "Em atendimento", o botão "Consulta" antes de "Ver escuta":

```diff
                         <div style={{ display: "flex", gap: 8, justifyContent: "flex-end" }}>
+                          {clinicalRecord && (
+                            <button type="button" style={secondaryButtonStyle}
+                              onClick={() => setPanel({ kind: "consultation", attendanceId: r.id })}>
+                              Consulta
+                            </button>
+                          )}
                           {r.screening?.id && (
```

E o painel, junto dos painéis da escuta (antes de `{closing && (`):

```diff
+        {canCare && clinicalRecord && panel?.kind === "consultation" && (
+          <ConsultationWorkspace
+            key={panel.attendanceId}
+            attendanceId={panel.attendanceId}
+            referenceUnitIds={inCare.find((r) => r.id === panel.attendanceId)?.reference_unit_ids}
+            unit={unit}
+            units={units}
+            onClose={() => setPanel(null)}
+            onFinalized={() => {
+              invalidate();
+              void queryClient.invalidateQueries({ queryKey: [ "unitRequests" ] });
+              setDone("Consulta finalizada e atendimento encerrado.");
+            }}
+          />
+        )}
+
         {closing && (
```

Em `src/modules/Attendance.tsx`:

```diff
--- a/src/modules/Attendance.tsx
+++ b/src/modules/Attendance.tsx
@@ export function Attendance({ onNavigate }: { onNavigate(id: ModuleId): void }) {
           canCare={canCare}
           careBlocked={careBlocked}
+          clinicalRecord={hasFeature(user, "clinical_record")}
           onClinicalRefused={() => void queryClient.invalidateQueries({ queryKey: [ "myProfessional", user.id ] })}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/modules/consultation/ConsultationWorkspace.test.tsx src/modules/attendance/UnitQueue.consultation.test.tsx src/modules/attendance/UnitQueue.test.tsx src/modules/attendance/UnitQueue.screening.test.tsx src/modules/Attendance.test.tsx && npx tsc --noEmit`
Expected: PASS (6 + 5 novos; `UnitQueue.test.tsx`, `UnitQueue.screening.test.tsx` e o resto de `Attendance.test.tsx` sem mudança de resultado — sem `clinicalRecord`, a fila é a do módulo 18); `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/modules/consultation/ConsultationWorkspace.tsx src/modules/consultation/ConsultationWorkspace.test.tsx src/modules/attendance/UnitQueue.tsx src/modules/attendance/UnitQueue.consultation.test.tsx src/modules/Attendance.tsx src/modules/Attendance.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "feat: open the consultation in the called attendance

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 10: Prontuário fora do atendimento — abertura justificada com step-up e contagem dos 30 minutos

**Files:**
- Create: `src/lib/clinicalRecord.ts`, `src/modules/ClinicalRecord.tsx`, `src/modules/clinicalRecord/OpeningForm.tsx`, `src/modules/clinicalRecord/JustifiedRecord.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx`
- Test: `src/lib/clinicalRecord.test.ts`, `src/modules/ClinicalRecord.test.tsx`, `src/modules/clinicalRecord/JustifiedRecord.test.tsx`

**Interfaces:**
- Consumes: `openClinicalRecord`, `getJustifiedRecord`, `getConsultationOptions`, `errorCode`, tipos (Task 1); `JUSTIFIED_KEY`, `CONSULTATION_KEY`, `OPTIONS_KEY`, `consultationError` (Task 2); `PatientPanel`, `ConsultationLoader` (Task 8); `SensitiveAction`, `FrozenTextNotice`, `Panel`, `PageHeader`, `EmptyState`, `Tag`; `isValidCpf`, `maskCpf` (`src/lib/attendance.ts`); `hasFeature`, `useAuth`.
- Produces:
  - `src/lib/clinicalRecord.ts`: `OPENING_REASONS` (`case_review` "Revisão de caso", `active_search` "Busca ativa", `continuity_of_care` "Continuidade do cuidado", `other` "Outro motivo"), `OPENING_NOTE_MIN = 10`, `reasonLabel(code)`, `interface OpeningDraft { cpf: string; reason: OpeningReason | ""; note: string }`, `EMPTY_OPENING`, `openingProblem(draft)`, `openingBody(draft): OpeningInput`, `remainingMs(expiresAt, nowMs)`, `countdownLabel(ms)` ("29:59"), `OPENING_ENDED`, `clinicalRecordError(err)`;
  - `OpeningForm({ onOpened(o: Opening): void; onGoToSecurity?(): void })` — `Panel` "Abrir prontuário fora do atendimento": "CPF do paciente", "Motivo", "Descrição do motivo" (com o `FrozenTextNotice`), "Continuar" → `SensitiveAction` "Confirmar abertura justificada" com step-up e `confirmLabel` "Abrir prontuário";
  - `JustifiedRecord({ opening: Opening; onEnd(): void; searchDelayMs?: number })` — `Panel` "Prontuário (abertura justificada)" com a marca "abertura justificada", "Abertura justificada · expira em mm:ss" (relógio de 1 s), `PatientPanel` e a consulta escolhida (`ConsultationLoader` com `canAddendum` sempre e o `opening_id`); quando a contagem chega a zero ou o api responde 403 `opening_required`, apaga o cache (`JUSTIFIED_KEY` e `CONSULTATION_KEY`), mostra `OPENING_ENDED` e "Nova abertura"; "Encerrar leitura" volta ao formulário;
  - `ClinicalRecord({ onGoToSecurity?(): void })` — página "Prontuário": interruptor desligado → "o prontuário está desligado nesta cidade"; sem `health_professional` nem `municipal_admin` → "seu papel não permite abrir o prontuário"; o profissional vê o formulário ou a leitura (o relatório entra na Task 11);
  - `ModuleId` com `"clinical-record"`; item "Prontuário" no grupo Atendimento, só para quem não é operador, tem `health_professional` ou `municipal_admin` e `clinical_record` na sessão.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/clinicalRecord.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  EMPTY_OPENING, clinicalRecordError, countdownLabel, openingBody, openingProblem, reasonLabel, remainingMs
} from "./clinicalRecord";

describe("abertura justificada", () => {
  it("CPF, motivo e a nota obrigatória só em 'Outro motivo'", () => {
    expect(openingProblem({ ...EMPTY_OPENING, cpf: "111.111.111-11", reason: "case_review" })).toBe("CPF inválido");
    expect(openingProblem({ ...EMPTY_OPENING, cpf: "529.982.247-25" })).toBe("escolha o motivo");
    expect(openingProblem({ cpf: "529.982.247-25", reason: "other", note: "curta" }))
      .toBe("em 'Outro motivo', descreva com pelo menos 10 caracteres");
    expect(openingProblem({ cpf: "529.982.247-25", reason: "other", note: "revisão pedida pela equipe" })).toBeNull();
    expect(openingProblem({ cpf: "529.982.247-25", reason: "active_search", note: "" })).toBeNull();
  });

  it("corpo: nota só quando escrita", () => {
    expect(openingBody({ cpf: "529.982.247-25", reason: "active_search", note: "  " }))
      .toEqual({ cpf: "529.982.247-25", reason_code: "active_search" });
    expect(openingBody({ cpf: "529.982.247-25", reason: "other", note: " revisão pedida pela equipe " }))
      .toEqual({ cpf: "529.982.247-25", reason_code: "other", reason_note: "revisão pedida pela equipe" });
  });

  it("contagem regressiva", () => {
    const now = Date.parse("2026-10-07T10:00:00-03:00");
    expect(remainingMs("2026-10-07T10:30:00-03:00", now)).toBe(30 * 60_000);
    expect(remainingMs("2026-10-07T09:59:00-03:00", now)).toBe(0);
    expect(remainingMs("lixo", now)).toBe(0);
    expect(countdownLabel(30 * 60_000)).toBe("30:00");
    expect(countdownLabel(30 * 60_000 - 1000)).toBe("29:59");
    expect(countdownLabel(59_500)).toBe("1:00");
    expect(countdownLabel(0)).toBe("0:00");
  });

  it("rótulos e recusas", () => {
    expect(reasonLabel("continuity_of_care")).toBe("Continuidade do cuidado");
    expect(clinicalRecordError(new ApiError(404, { error: "patient_not_found" }, "404")))
      .toBe("nenhum prontuário para este CPF — o prontuário nasce na primeira consulta de um cadastro validado");
    expect(clinicalRecordError(new ApiError(422, { error: "invalid_reason" }, "422")))
      .toBe("escolha o motivo; em 'Outro motivo', descreva com pelo menos 10 caracteres");
    expect(clinicalRecordError(new ApiError(403, { error: "opening_required" }, "403")))
      .toBe("a abertura justificada terminou ou não existe — abra o prontuário de novo com o motivo");
  });
});
```

```tsx
// src/modules/ClinicalRecord.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), openClinicalRecord: vi.fn(), getJustifiedRecord: vi.fn(),
    getConsultationOptions: vi.fn(), listOpenings: vi.fn(), listMemberships: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { ClinicalRecord } from "./ClinicalRecord";
import { renderWithProviders, sessionWith } from "../test/campaignFixtures";
import { opening, options, record } from "../test/consultationFixtures";

afterEach(cleanup);
const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const pro = () => sessionWith([ "health_professional" ], { id: "us1", features: [ "clinical_record" ] });

describe("Prontuário fora do atendimento", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.stepUpMfa, api.openClinicalRecord, api.getJustifiedRecord,
      api.getConsultationOptions, api.listOpenings, api.listMemberships ]) m(fn).mockReset();
    m(api.fetchCurrentSession).mockResolvedValue(pro());
    m(api.openClinicalRecord).mockResolvedValue(opening({ expires_at: new Date(Date.now() + 30 * 60_000).toISOString() }));
    m(api.getJustifiedRecord).mockResolvedValue(record({ access: "justified" }));
    m(api.getConsultationOptions).mockResolvedValue(options());
    m(api.listOpenings).mockResolvedValue([]);
    m(api.listMemberships).mockResolvedValue([]);
  });

  it("interruptor desligado: diz e não oferece nada", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ]));
    renderWithProviders(<ClinicalRecord />);
    expect(await screen.findByText("o prontuário está desligado nesta cidade")).not.toBeNull();
    expect(screen.queryByLabelText("CPF do paciente")).toBeNull();
  });

  it("recepção não abre prontuário", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "citizen_verifier" ], { features: [ "clinical_record" ] }));
    renderWithProviders(<ClinicalRecord />);
    expect(await screen.findByText("seu papel não permite abrir o prontuário")).not.toBeNull();
    expect(api.getJustifiedRecord).not.toHaveBeenCalled();
  });

  it("CPF, motivo e step-up abrem a leitura marcada como abertura justificada", async () => {
    renderWithProviders(<ClinicalRecord />);
    fireEvent.change(await screen.findByLabelText("CPF do paciente"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "active_search" } });
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    const confirm = screen.getByRole("region", { name: "Confirmar abertura justificada" });
    expect(confirm.textContent).toContain("CPF 529.982.247-25 · Busca ativa");
    fireEvent.click(screen.getByRole("button", { name: "Abrir prontuário" }));
    await waitFor(() => expect(api.openClinicalRecord).toHaveBeenCalledWith({ cpf: "529.982.247-25", reason_code: "active_search" }));
    expect(await screen.findByText("Joana Lima")).not.toBeNull();
    expect(api.getJustifiedRecord).toHaveBeenCalledWith("pa1");
    expect(screen.getByText(/^Abertura justificada · expira em (30:00|29:5\d)$/)).not.toBeNull();
  });

  it("'Outro motivo' sem descrição não continua", async () => {
    renderWithProviders(<ClinicalRecord />);
    fireEvent.change(await screen.findByLabelText("CPF do paciente"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "other" } });
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    expect(screen.getByRole("alert").textContent).toBe("em 'Outro motivo', descreva com pelo menos 10 caracteres");
    expect(screen.queryByRole("region", { name: "Confirmar abertura justificada" })).toBeNull();
  });

  it("CPF sem prontuário: a frase aparece na confirmação", async () => {
    m(api.openClinicalRecord).mockRejectedValue(new ApiError(404, { error: "patient_not_found" }, "404"));
    renderWithProviders(<ClinicalRecord />);
    fireEvent.change(await screen.findByLabelText("CPF do paciente"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "case_review" } });
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    fireEvent.click(screen.getByRole("button", { name: "Abrir prontuário" }));
    expect(await screen.findByText("nenhum prontuário para este CPF — o prontuário nasce na primeira consulta de um cadastro validado"))
      .not.toBeNull();
  });
});
```

```tsx
// src/modules/clinicalRecord/JustifiedRecord.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, screen } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getJustifiedRecord: vi.fn(), getConsultationOptions: vi.fn() };
});
vi.mock("../consultation/ConsultationView", () => ({
  ConsultationLoader: (p: { id: string; openingId?: string; canAddendum(c: unknown): boolean; onOpeningRequired?(): void }) => (
    <div>
      <span>{`consulta ${p.id} · abertura ${p.openingId} · adendo ${p.canAddendum({}) ? "sim" : "não"}`}</span>
      <button type="button" onClick={() => p.onOpeningRequired?.()}>abertura acabou (dublê)</button>
    </div>
  )
}));

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { JustifiedRecord } from "./JustifiedRecord";
import { OPENING_ENDED } from "../../lib/clinicalRecord";
import { renderWithProviders, sessionWith } from "../../test/campaignFixtures";
import { NOW19, opening, options, record } from "../../test/consultationFixtures";

const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(() => { cleanup(); vi.useRealTimers(); });

describe("JustifiedRecord", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.getJustifiedRecord, api.getConsultationOptions ]) m(fn).mockReset();
    vi.useFakeTimers({ toFake: [ "Date", "setInterval", "clearInterval" ] });
    vi.setSystemTime(new Date(NOW19));
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "clinical_record" ] }));
    m(api.getJustifiedRecord).mockResolvedValue(record({ access: "justified" }));
    m(api.getConsultationOptions).mockResolvedValue(options());
  });

  it("a contagem chega a zero: o prontuário some e oferece nova abertura", async () => {
    const onEnd = vi.fn();
    renderWithProviders(<JustifiedRecord opening={opening()} onEnd={onEnd} />);
    expect(await screen.findByText("Joana Lima")).not.toBeNull();
    expect(screen.getByText("Abertura justificada · expira em 30:00")).not.toBeNull();
    act(() => { vi.advanceTimersByTime(1000); });
    expect(screen.getByText("Abertura justificada · expira em 29:59")).not.toBeNull();
    act(() => { vi.advanceTimersByTime(30 * 60_000); });
    expect(screen.getByText(OPENING_ENDED)).not.toBeNull();
    expect(screen.queryByText("Joana Lima")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Nova abertura" }));
    expect(onEnd).toHaveBeenCalled();
  });

  it("403 opening_required na leitura encerra a abertura", async () => {
    m(api.getJustifiedRecord).mockRejectedValue(new ApiError(403, { error: "opening_required" }, "403"));
    renderWithProviders(<JustifiedRecord opening={opening()} onEnd={vi.fn()} />);
    expect(await screen.findByText(OPENING_ENDED)).not.toBeNull();
  });

  it("consulta anterior com a abertura (adendo permitido) e fim da abertura vindo dela", async () => {
    renderWithProviders(<JustifiedRecord opening={opening()} onEnd={vi.fn()} />);
    fireEvent.click(await screen.findByRole("button", { name: "Abrir consulta de 10/09/2026, 14:30" }));
    expect(screen.getByText("consulta cs0 · abertura op1 · adendo sim")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "abertura acabou (dublê)" }));
    expect(screen.getByText(OPENING_ENDED)).not.toBeNull();
  });
});
```

E em `src/shell/modules.test.ts`, um `describe` novo dentro do `describe("modules")`:

```ts
  describe("módulo 19 na navegação", () => {
    const user = (roles: string[], features: string[] = [], operator = false) =>
      ({ operator, memberships: roles.map((role) => ({ role })), features });
    const ids = (u: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(u).flatMap((g) => g.items.map((i) => i.id));

    it("Prontuário no menu: só profissional ou admin, só com clinical_record, nunca operador", () => {
      expect(ids(user([ "health_professional" ], [ "clinical_record" ]))).toContain("clinical-record");
      expect(ids(user([ "municipal_admin" ], [ "clinical_record" ]))).toContain("clinical-record");
      expect(ids(user([ "health_professional" ]))).not.toContain("clinical-record");
      expect(ids(user([ "citizen_verifier" ], [ "clinical_record" ]))).not.toContain("clinical-record");
      expect(ids(user([ "municipal_admin" ], [ "clinical_record" ], true))).not.toContain("clinical-record");
      expect(labelFor("clinical-record")).toBe("Prontuário");
    });
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/clinicalRecord.test.ts src/modules/ClinicalRecord.test.tsx src/modules/clinicalRecord/JustifiedRecord.test.tsx src/shell/modules.test.ts`
Expected: FAIL — `Failed to resolve import "./clinicalRecord"` (e `./ClinicalRecord`, `./JustifiedRecord`); no menu, `expected [...] to include 'clinical-record'`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/clinicalRecord.ts
// Abertura justificada do prontuário (módulo 19; spec §5; ADR 0031): fora do
// atendimento, a leitura pede CPF, motivo de lista (nota ≥ 10 em "Outro
// motivo") e step-up, vale 30 minutos para aquele paciente e usuário e fica no
// relatório. O CPF e a nota vão no corpo do POST.
import { ApiError, type OpeningInput, type OpeningReason } from "./api";
import { isValidCpf } from "./attendance";
import { consultationError } from "./consultation";

export const OPENING_REASONS: { value: OpeningReason; label: string }[] = [
  { value: "case_review", label: "Revisão de caso" },
  { value: "active_search", label: "Busca ativa" },
  { value: "continuity_of_care", label: "Continuidade do cuidado" },
  { value: "other", label: "Outro motivo" }
];
export const OPENING_NOTE_MIN = 10;
export const OPENING_ENDED = "A abertura de 30 minutos terminou. Para continuar, abra de novo com o motivo.";

export function reasonLabel(code: string): string {
  return OPENING_REASONS.find((r) => r.value === code)?.label ?? code;
}

export interface OpeningDraft { cpf: string; reason: OpeningReason | ""; note: string }
export const EMPTY_OPENING: OpeningDraft = { cpf: "", reason: "", note: "" };

export function openingProblem(d: OpeningDraft): string | null {
  if (!isValidCpf(d.cpf)) return "CPF inválido";
  if (d.reason === "") return "escolha o motivo";
  if (d.reason === "other" && d.note.trim().length < OPENING_NOTE_MIN) {
    return `em 'Outro motivo', descreva com pelo menos ${OPENING_NOTE_MIN} caracteres`;
  }
  return null;
}

// Só chamado depois de openingProblem === null (o motivo já foi escolhido).
export function openingBody(d: OpeningDraft): OpeningInput {
  const note = d.note.trim();
  return { cpf: d.cpf, reason_code: d.reason as OpeningReason, ...(note ? { reason_note: note } : {}) };
}

export function remainingMs(expiresAt: string, nowMs: number): number {
  const t = Date.parse(expiresAt);
  return Number.isNaN(t) ? 0 : Math.max(0, t - nowMs);
}

export function countdownLabel(ms: number): string {
  const total = Math.ceil(Math.max(0, ms) / 1000);
  return `${Math.floor(total / 60)}:${String(total % 60).padStart(2, "0")}`;
}

export function clinicalRecordError(err: unknown): string {
  const code = err instanceof ApiError ? (err.body as { error?: string } | null)?.error : undefined;
  if (code === "patient_not_found") return "nenhum prontuário para este CPF — o prontuário nasce na primeira consulta de um cadastro validado";
  if (code === "invalid_reason") return `escolha o motivo; em 'Outro motivo', descreva com pelo menos ${OPENING_NOTE_MIN} caracteres`;
  return consultationError(err);
}
```

```tsx
// src/modules/clinicalRecord/OpeningForm.tsx
// Pedido de abertura justificada (módulo 19; spec §5 e §7). O step-up usa o
// padrão existente (SensitiveAction): a tela diz o quê, ele pede o código.
import { useRef, useState, type CSSProperties } from "react";
import { errorCode, openClinicalRecord, type Opening, type OpeningReason } from "../../lib/api";
import { maskCpf } from "../../lib/attendance";
import {
  EMPTY_OPENING, OPENING_REASONS, clinicalRecordError, openingBody, openingProblem, reasonLabel, type OpeningDraft
} from "../../lib/clinicalRecord";
import { Panel } from "../../components/Panel";
import { SensitiveAction } from "../../components/SensitiveAction";
import { FrozenTextNotice } from "../../components/FrozenTextNotice";
import { buttonStyle, inputStyle } from "../../components/formStyles";

const OWN_REFUSALS = new Set([ "patient_not_found", "invalid_reason", "feature_disabled", "missing_role" ]);

export function OpeningForm({ onOpened, onGoToSecurity }: { onOpened(o: Opening): void; onGoToSecurity?(): void }) {
  const [ draft, setDraft ] = useState<OpeningDraft>(EMPTY_OPENING);
  const [ problem, setProblem ] = useState<string | null>(null);
  const [ confirming, setConfirming ] = useState(false);
  const opened = useRef<Opening | null>(null);

  function next() {
    const p = openingProblem(draft);
    setProblem(p);
    if (!p) setConfirming(true);
  }

  return (
    <Panel title="Abrir prontuário fora do atendimento" sub="abertura justificada · válida por 30 minutos">
      <div style={{ display: "flex", flexDirection: "column", gap: 10, maxWidth: 420 }}>
        <p style={muted}>
          Use só quando precisar ler o prontuário de alguém que não está em atendimento com você. A abertura fica
          registrada, com o motivo, e aparece no relatório da administração.
        </p>
        {!confirming ? (
          <>
            <label style={labelStyle}>
              CPF do paciente
              <input value={draft.cpf} inputMode="numeric" style={inputStyle}
                onChange={(e) => setDraft({ ...draft, cpf: maskCpf(e.target.value) })} />
            </label>
            <label style={labelStyle}>
              Motivo
              <select value={draft.reason} style={inputStyle}
                onChange={(e) => setDraft({ ...draft, reason: e.target.value as OpeningReason | "" })}>
                <option value="">—</option>
                {OPENING_REASONS.map((r) => <option key={r.value} value={r.value}>{r.label}</option>)}
              </select>
            </label>
            <label style={labelStyle}>
              Descrição do motivo
              <textarea value={draft.note} rows={2} style={inputStyle} aria-describedby="opening-note-notice"
                onChange={(e) => setDraft({ ...draft, note: e.target.value })} />
            </label>
            <FrozenTextNotice id="opening-note-notice" />
            <small style={muted}>Obrigatória em "Outro motivo" (pelo menos 10 caracteres).</small>
            {problem && <p role="alert" style={alert}>{problem}</p>}
            <div><button type="button" style={buttonStyle} onClick={next}>Continuar</button></div>
          </>
        ) : (
          <SensitiveAction
            title="Confirmar abertura justificada"
            description={`CPF ${draft.cpf} · ${reasonLabel(draft.reason)}. A leitura vale por 30 minutos e fica no relatório da administração.`}
            requiresStepUp
            confirmLabel="Abrir prontuário"
            run={async () => { opened.current = await openClinicalRecord(openingBody(draft)); }}
            onDone={() => { if (opened.current) onOpened(opened.current); }}
            onCancel={() => setConfirming(false)}
            onGoToSecurity={onGoToSecurity}
            translateError={(err) => (OWN_REFUSALS.has(errorCode(err) ?? "") ? clinicalRecordError(err) : null)}
          />
        )}
      </div>
    </Panel>
  );
}

const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

```tsx
// src/modules/clinicalRecord/JustifiedRecord.tsx
// Leitura com abertura justificada (módulo 19; spec §5 e §7): marcada, com a
// contagem dos 30 minutos. Quando acaba (relógio ou 403 opening_required do
// api), os dados saem da tela e do cache e a pessoa pode abrir de novo.
import { useEffect, useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { errorCode, getConsultationOptions, getJustifiedRecord, type Opening } from "../../lib/api";
import { CONSULTATION_KEY, JUSTIFIED_KEY, OPTIONS_KEY } from "../../lib/consultation";
import { OPENING_ENDED, clinicalRecordError, countdownLabel, remainingMs } from "../../lib/clinicalRecord";
import { useAuth } from "../../lib/auth";
import { Panel } from "../../components/Panel";
import { Tag } from "../../components/Tag";
import { buttonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { PatientPanel } from "../consultation/PatientPanel";
import { ConsultationLoader } from "../consultation/ConsultationView";

interface Props { opening: Opening; onEnd(): void; searchDelayMs?: number }

export function JustifiedRecord({ opening, onEnd, searchDelayMs }: Props) {
  const { user } = useAuth();
  const queryClient = useQueryClient();
  const [ nowMs, setNowMs ] = useState(() => Date.now());
  const [ ended, setEnded ] = useState(false);
  const [ viewing, setViewing ] = useState<string | null>(null);

  useEffect(() => {
    const id = setInterval(() => setNowMs(Date.now()), 1000);
    return () => clearInterval(id);
  }, []);

  const left = remainingMs(opening.expires_at, nowMs);
  const record = useQuery({
    queryKey: [ JUSTIFIED_KEY, opening.opening_id ], queryFn: () => getJustifiedRecord(opening.patient_id),
    enabled: !ended && left > 0, gcTime: 0, staleTime: 0
  });
  const options = useQuery({ queryKey: [ OPTIONS_KEY, user?.id ?? null ], queryFn: getConsultationOptions, staleTime: 5 * 60_000 });
  const over = ended || left <= 0 || (record.isError && errorCode(record.error) === "opening_required");

  useEffect(() => {
    if (!over) return;
    queryClient.removeQueries({ queryKey: [ JUSTIFIED_KEY, opening.opening_id ] });
    queryClient.removeQueries({ queryKey: [ CONSULTATION_KEY ] });
  }, [ over, queryClient, opening.opening_id ]);

  if (over) {
    return (
      <Panel title="Prontuário (abertura justificada)">
        <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
          <p role="status" style={{ margin: 0, fontSize: 13, fontWeight: 600 }}>{OPENING_ENDED}</p>
          <div><button type="button" style={buttonStyle} onClick={onEnd}>Nova abertura</button></div>
        </div>
      </Panel>
    );
  }

  return (
    <Panel title="Prontuário (abertura justificada)" sub="leitura fora do atendimento · fica registrada"
      right={<Tag tone="warn">abertura justificada</Tag>}>
      <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
        <p style={countdown}>{`Abertura justificada · expira em ${countdownLabel(left)}`}</p>
        {record.isPending && <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando o prontuário…</p>}
        {record.isError && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{clinicalRecordError(record.error)}</p>}
        {record.data && (
          <>
            <PatientPanel record={record.data} onOpenConsultation={setViewing} />
            {viewing && (
              <ConsultationLoader key={viewing} id={viewing} canAddendum={() => true} openingId={opening.opening_id}
                options={options.data ?? null} patientProblems={record.data.problems} searchDelayMs={searchDelayMs}
                onClose={() => setViewing(null)} onOpeningRequired={() => setEnded(true)} />
            )}
          </>
        )}
        <div><button type="button" style={secondaryButtonStyle} onClick={onEnd}>Encerrar leitura</button></div>
      </div>
    </Panel>
  );
}

const countdown: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600, color: "var(--warn)" };
```

```tsx
// src/modules/ClinicalRecord.tsx
// Prontuário fora do atendimento (módulo 19; spec §5 e §7): abertura
// justificada para o profissional e, para o municipal_admin, o relatório das
// aberturas (Task 11). Só com `clinical_record` na sessão; a recepção não entra.
import { useState, type ReactNode } from "react";
import type { Opening } from "../lib/api";
import { useAuth } from "../lib/auth";
import { hasFeature } from "../lib/features";
import { PageHeader } from "../components/PageHeader";
import { EmptyState } from "../components/EmptyState";
import { OpeningForm } from "./clinicalRecord/OpeningForm";
import { JustifiedRecord } from "./clinicalRecord/JustifiedRecord";

export function ClinicalRecord({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  const { user } = useAuth();
  const [ opening, setOpening ] = useState<Opening | null>(null);
  if (!user) return null;
  const roles = user.memberships.map((m) => m.role);
  const isProfessional = roles.includes("health_professional");
  const isAdmin = roles.includes("municipal_admin");

  if (!hasFeature(user, "clinical_record")) return <Frame><EmptyState title="o prontuário está desligado nesta cidade" /></Frame>;
  if (user.operator || (!isProfessional && !isAdmin)) {
    return <Frame><EmptyState title="seu papel não permite abrir o prontuário" /></Frame>;
  }

  return (
    <Frame>
      {isProfessional && (opening
        ? <JustifiedRecord key={opening.opening_id} opening={opening} onEnd={() => setOpening(null)} />
        : <OpeningForm onOpened={setOpening} onGoToSecurity={onGoToSecurity} />)}
    </Frame>
  );
}

function Frame({ children }: { children: ReactNode }) {
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Prontuário" sub="leitura fora do atendimento · abertura justificada" />
      {children}
    </div>
  );
}
```

Em `src/shell/modules.ts`:

```diff
--- a/src/shell/modules.ts
+++ b/src/shell/modules.ts
@@ -9,7 +9,7 @@ export type ModuleId =
   | "queues" | "health" | "protocol-editor" | "security" | "team" | "attendance"
   | "professionals" | "my-profile" | "territory" | "campaigns" | "analytics"
-  | "integrations" | "cnes" | "production" | "my-agenda";
+  | "integrations" | "cnes" | "production" | "my-agenda" | "clinical-record";
@@ -35,7 +35,8 @@ export const NAV_GROUPS: NavGroupDef[] = [
   { label: "Atendimento", items: [
     { id: "attendance", label: "Atendimento", icon: "☑" },
-    { id: "my-agenda", label: "Minha agenda", icon: "◷" }
+    { id: "my-agenda", label: "Minha agenda", icon: "◷" },
+    { id: "clinical-record", label: "Prontuário", icon: "⚕" }
   ]},
@@ export function navGroupsFor(
   const canProduction = !user?.operator && (isAdmin || roles.includes("analyst")) && hasFeature(user, "ledi_export");
+  // Módulo 19 (ADR 0031): abertura justificada do profissional e relatório do
+  // municipal_admin, só com `clinical_record` ligado; a recepção nunca.
+  const canClinicalRecord = !user?.operator && (isProfessional || isAdmin) && hasFeature(user, "clinical_record");
@@
       if (item.id === "my-agenda") return isProfessional;
+      if (item.id === "clinical-record") return canClinicalRecord;
       if (item.id === "integrations" || item.id === "cnes") return canIntegrations;
```

Em `src/App.tsx`:

```diff
--- a/src/App.tsx
+++ b/src/App.tsx
@@ -29,6 +29,7 @@ import { Integrations } from "./modules/Integrations";
 import { Cnes } from "./modules/Cnes";
 import { Production } from "./modules/Production";
+import { ClinicalRecord } from "./modules/ClinicalRecord";
@@ function renderModule(active: ModuleId, setActive: (id: ModuleId) => void) {
     case "production":     return <Production onGoToSecurity={() => setActive("security")} />;
+    case "clinical-record": return <ClinicalRecord onGoToSecurity={() => setActive("security")} />;
     default:               return <Placeholder title={labelFor(active)} />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/clinicalRecord.test.ts src/modules/ClinicalRecord.test.tsx src/modules/clinicalRecord/JustifiedRecord.test.tsx src/shell/modules.test.ts && npx tsc --noEmit`
Expected: PASS (4 + 5 + 3 novos; `modules.test.ts` com o teste novo e os de antes — "Minha agenda" continua `[ "attendance", "my-agenda" ]` sem o interruptor); `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/lib/clinicalRecord.ts src/lib/clinicalRecord.test.ts src/modules/ClinicalRecord.tsx src/modules/ClinicalRecord.test.tsx src/modules/clinicalRecord/OpeningForm.tsx src/modules/clinicalRecord/JustifiedRecord.tsx src/modules/clinicalRecord/JustifiedRecord.test.tsx src/shell/modules.ts src/shell/modules.test.ts src/App.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "feat: open the clinical record out of context with a justified opening

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 11: Relatório "Aberturas fora de contexto" (municipal_admin)

**Files:**
- Create: `src/modules/clinicalRecord/OpeningsReport.tsx`
- Modify: `src/modules/ClinicalRecord.tsx`, `src/modules/ClinicalRecord.test.tsx`
- Test: `src/modules/clinicalRecord/OpeningsReport.test.tsx`

**Interfaces:**
- Consumes: `listOpenings`, `listMemberships`, `OpeningRow`, `MembershipRow` (Task 1 e existentes); `reasonLabel` (Task 10); `addDays`, `todayInCity` (`src/lib/campaigns.ts`); `fmtDateTime`, `fmtHourMinute`; `consultationError` (Task 2).
- Produces: `OPENINGS_KEY = "clinicalOpenings"`; `OpeningsReport({ today?: string })` — `Panel` "Aberturas fora de contexto": "De" e "Até" (padrão: os últimos 30 dias, no fuso da cidade), "Profissional" (todos ou um usuário de `listMemberships`, por e-mail), "Buscar"; tabela Quando, Quem, CPF (mascarado), Motivo, Válida até; "nenhuma abertura no período"; período invertido não busca e diz por quê. `ClinicalRecord` mostra o relatório para `municipal_admin`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/clinicalRecord/OpeningsReport.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, listOpenings: vi.fn(), listMemberships: vi.fn() };
});

import * as api from "../../lib/api";
import { OpeningsReport } from "./OpeningsReport";
import { TODAY19, openingRow } from "../../test/consultationFixtures";

afterEach(cleanup);
const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const member = (id: string, email: string, role = "health_professional") =>
  ({ id: `m-${id}-${role}`, user: { id, email_address: email }, role, granted_at: "2026-09-01T00:00:00Z" });

function renderIt() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  render(<QueryClientProvider client={client}><OpeningsReport today={TODAY19} /></QueryClientProvider>);
}

describe("OpeningsReport", () => {
  beforeEach(() => {
    m(api.listOpenings).mockReset();
    m(api.listMemberships).mockReset();
    m(api.listOpenings).mockResolvedValue([ openingRow() ]);
    m(api.listMemberships).mockResolvedValue([
      member("us9", "lucia@curitiba.demo"), member("us9", "lucia@curitiba.demo", "citizen_verifier"), member("us2", "helena@curitiba.demo")
    ]);
  });

  it("lista os últimos 30 dias: quem, quando, CPF mascarado e motivo", async () => {
    renderIt();
    const table = await screen.findByRole("table");
    expect(within(table).getByText("Enf. Lúcia Prado")).not.toBeNull();
    expect(within(table).getByText("***.982.247-**")).not.toBeNull();
    expect(within(table).getByText("Revisão de caso")).not.toBeNull();
    expect(within(table).getByText("06/10/2026, 15:10")).not.toBeNull();
    expect(api.listOpenings).toHaveBeenCalledWith({ from: "2026-09-07", to: "2026-10-07", userId: "" });
  });

  it("filtra por profissional (cada pessoa uma vez) e período", async () => {
    renderIt();
    await screen.findByRole("table");
    const select = screen.getByLabelText("Profissional") as HTMLSelectElement;
    await waitFor(() => expect(select.options).toHaveLength(3));
    fireEvent.change(select, { target: { value: "us9" } });
    fireEvent.change(screen.getByLabelText("De"), { target: { value: "2026-10-01" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
    await waitFor(() => expect(api.listOpenings).toHaveBeenLastCalledWith({ from: "2026-10-01", to: "2026-10-07", userId: "us9" }));
  });

  it("período invertido não busca", async () => {
    renderIt();
    await screen.findByRole("table");
    fireEvent.change(screen.getByLabelText("De"), { target: { value: "2026-10-08" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
    expect(screen.getByRole("alert").textContent).toBe("a data inicial precisa ser igual ou anterior à final");
    expect(api.listOpenings).toHaveBeenCalledTimes(1);
  });

  it("sem aberturas no período", async () => {
    m(api.listOpenings).mockResolvedValue([]);
    renderIt();
    expect(await screen.findByText("nenhuma abertura no período")).not.toBeNull();
  });
});
```

E em `src/modules/ClinicalRecord.test.tsx`, dois testes no `describe` existente:

```tsx
  it("o municipal_admin vê o relatório, sem o formulário de abertura", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "municipal_admin" ], { features: [ "clinical_record" ] }));
    renderWithProviders(<ClinicalRecord />);
    expect(await screen.findByRole("region", { name: "Aberturas fora de contexto" })).not.toBeNull();
    expect(screen.queryByLabelText("CPF do paciente")).toBeNull();
  });

  it("o profissional não vê o relatório", async () => {
    renderWithProviders(<ClinicalRecord />);
    await screen.findByLabelText("CPF do paciente");
    expect(screen.queryByRole("region", { name: "Aberturas fora de contexto" })).toBeNull();
    expect(api.listOpenings).not.toHaveBeenCalled();
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/modules/clinicalRecord/OpeningsReport.test.tsx src/modules/ClinicalRecord.test.tsx`
Expected: FAIL — `Failed to resolve import "./OpeningsReport"`; no `ClinicalRecord`, `Unable to find role="region" and name "Aberturas fora de contexto"`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/clinicalRecord/OpeningsReport.tsx
// Relatório das aberturas fora de contexto (módulo 19; spec §5; contratos §3):
// para o municipal_admin, quem abriu prontuário fora do atendimento, quando,
// de qual CPF (mascarado pelo api) e por quê. A nota da abertura nunca vem.
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { listMemberships, listOpenings, type OpeningRow } from "../../lib/api";
import { reasonLabel } from "../../lib/clinicalRecord";
import { consultationError } from "../../lib/consultation";
import { addDays, todayInCity } from "../../lib/campaigns";
import { fmtDateTime, fmtHourMinute } from "../../lib/format";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { EmptyState } from "../../components/EmptyState";
import { buttonStyle, inputStyle } from "../../components/formStyles";

export const OPENINGS_KEY = "clinicalOpenings";
const DEFAULT_DAYS = 30;

export function OpeningsReport({ today = todayInCity() }: { today?: string }) {
  const [ from, setFrom ] = useState(() => addDays(today, -DEFAULT_DAYS));
  const [ to, setTo ] = useState(today);
  const [ userId, setUserId ] = useState("");
  const [ params, setParams ] = useState(() => ({ from: addDays(today, -DEFAULT_DAYS), to: today, userId: "" }));
  const [ problem, setProblem ] = useState<string | null>(null);
  const members = useQuery({ queryKey: [ "memberships" ], queryFn: listMemberships, staleTime: 60_000 });
  const query = useQuery({
    queryKey: [ OPENINGS_KEY, params.from, params.to, params.userId ], queryFn: () => listOpenings(params), gcTime: 0
  });

  // Uma linha por papel em /setup/memberships: cada pessoa aparece uma vez.
  const users = Array.from(new Map((members.data ?? []).map((row) => [ row.user.id, row.user.email_address ])))
    .sort((a, b) => a[1].localeCompare(b[1], "pt-BR"));

  function apply() {
    if (from > to) { setProblem("a data inicial precisa ser igual ou anterior à final"); return; }
    setProblem(null);
    setParams({ from, to, userId });
  }

  return (
    <Panel title="Aberturas fora de contexto" sub="quem abriu prontuário fora do atendimento, e por quê">
      <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
        <div style={{ display: "flex", gap: 8, alignItems: "flex-end", flexWrap: "wrap" }}>
          <label style={labelStyle}>De<input type="date" value={from} max={today} style={inputStyle} onChange={(e) => setFrom(e.target.value)} /></label>
          <label style={labelStyle}>Até<input type="date" value={to} max={today} style={inputStyle} onChange={(e) => setTo(e.target.value)} /></label>
          <label style={{ ...labelStyle, minWidth: 220 }}>
            Profissional
            <select value={userId} style={inputStyle} onChange={(e) => setUserId(e.target.value)}>
              <option value="">todos</option>
              {users.map(([ id, email ]) => <option key={id} value={id}>{email}</option>)}
            </select>
          </label>
          <button type="button" style={buttonStyle} onClick={apply}>Buscar</button>
        </div>
        {problem && <p role="alert" style={alert}>{problem}</p>}
        {query.isError && <p role="alert" style={alert}>{consultationError(query.error)}</p>}
        {query.isPending && <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando…</p>}
        {query.isSuccess && (query.data.length === 0 ? <EmptyState title="nenhuma abertura no período" /> : (
          <DataTable<OpeningRow>
            cols={[
              { label: "Quando", w: "1fr", render: (r) => fmtDateTime(r.created_at) },
              { label: "Quem", w: "1.4fr", render: (r) => r.user_name },
              { label: "CPF", w: "1fr", render: (r) => <span className="mono">{r.cpf_masked}</span> },
              { label: "Motivo", w: "1.2fr", render: (r) => reasonLabel(r.reason_code) },
              { label: "Válida até", w: "0.8fr", render: (r) => fmtHourMinute(r.expires_at) }
            ]}
            rows={query.data}
            rowKey={(r) => r.id}
          />
        ))}
      </div>
    </Panel>
  );
}

const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

(O `DataTable` já é `role="table"`, com linhas `role="row"`; o teste o acha por `findByRole("table")`.)

Em `src/modules/ClinicalRecord.tsx`:

```diff
--- a/src/modules/ClinicalRecord.tsx
+++ b/src/modules/ClinicalRecord.tsx
@@
 import { OpeningForm } from "./clinicalRecord/OpeningForm";
 import { JustifiedRecord } from "./clinicalRecord/JustifiedRecord";
+import { OpeningsReport } from "./clinicalRecord/OpeningsReport";
@@
       {isProfessional && (opening
         ? <JustifiedRecord key={opening.opening_id} opening={opening} onEnd={() => setOpening(null)} />
         : <OpeningForm onOpened={setOpening} onGoToSecurity={onGoToSecurity} />)}
+      {isAdmin && <OpeningsReport />}
     </Frame>
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/modules/clinicalRecord/OpeningsReport.test.tsx src/modules/ClinicalRecord.test.tsx`
Expected: PASS (4 + 7 testes).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/modules/clinicalRecord/OpeningsReport.tsx src/modules/clinicalRecord/OpeningsReport.test.tsx src/modules/ClinicalRecord.tsx src/modules/ClinicalRecord.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "feat: report out-of-context clinical record openings to the municipal admin

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 12: Nomes na validação presencial e "completar nomes" no check-in

O nome completo (obrigatório), o nome social e o nome da mãe passam a ser conferidos no documento no balcão de validação (contratos §2). Quem já foi validado sem eles — inclusive quem é validado **no check-in**, que hoje valida sem nome (`Attendances::CheckIn.verify_once`) — tem os nomes completados no próprio check-in: a busca de validação recusa par já validado (409 `already_verified`) e o wpda não emite código de validação para ele, então o balcão de validação não alcança esse par (Divergência D2). Sem os nomes, a consulta não finaliza nem imprime (`patient_name_missing`).

**Files:**
- Create: `src/lib/citizenNames.ts`, `src/modules/attendance/NamesFields.tsx`, `src/modules/attendance/CompleteNames.tsx`
- Modify: `src/modules/Attendance.tsx` (`Counter`), `src/modules/attendance/CheckIn.tsx` (`CodeFlow`), `src/lib/attendance.ts` (`MESSAGES`)
- Test: `src/lib/citizenNames.test.ts`, `src/modules/Attendance.test.tsx`, `src/modules/attendance/CheckIn.test.tsx`

**Interfaces:**
- Consumes: `verifyCitizen` (com `Partial<CitizenNamesInput>`), `completeCitizenNames`, `CheckInCitizen.names`/`verification_id`, `checkIn(...).verification_id`, `CitizenNamesInput` (Task 1); `attendanceError`.
- Produces:
  - `src/lib/citizenNames.ts`: `NAME_MAX = 200`, `FULL_NAME_MIN = 3`, `interface NamesDraft { fullName: string; socialName: string; motherName: string }`, `EMPTY_NAMES`, `namesProblem(d): string | null`, `namesBody(d): CitizenNamesInput` (espaços repetidos viram um; social e mãe só quando preenchidos);
  - `NamesFields({ value: NamesDraft; onChange(next): void; problem: string | null })` — fieldset "Nomes (como no documento)" com "Nome completo (documento)", "Nome social (opcional)", "Nome da mãe (opcional)" e o problema sob os campos (`role="alert"`, fora dos rótulos); sem `FrozenTextNotice` (o aviso diz para não escrever nomes);
  - `CompleteNames({ verificationId: string })` — seção "Completar nomes do cadastro" com os `NamesFields` e "Salvar nomes"; depois, "Nomes registrados no cadastro";
  - `Counter`: "Validar cadastro" também espera o nome completo, e o corpo leva `full_name` (e `social_name`/`mother_name` quando preenchidos);
  - `CheckIn` (código): par validado com `names.full_name_set === false` e `verification_id` mostra `CompleteNames` antes de "Iniciar atendimento"; check-in que validou agora (`verified` com `verification_id`) mostra `CompleteNames` depois de "Atendimento iniciado".

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/citizenNames.test.ts
import { describe, expect, it } from "vitest";
import { EMPTY_NAMES, namesBody, namesProblem } from "./citizenNames";

describe("nomes do documento", () => {
  it("nome completo obrigatório (3 a 200); social e mãe até 200", () => {
    expect(namesProblem(EMPTY_NAMES)).toBe("informe o nome completo como no documento");
    expect(namesProblem({ ...EMPTY_NAMES, fullName: " Jo " })).toBe("informe o nome completo como no documento");
    expect(namesProblem({ ...EMPTY_NAMES, fullName: "a".repeat(201) })).toBe("o nome completo pode ter até 200 caracteres");
    expect(namesProblem({ ...EMPTY_NAMES, fullName: "João Carlos Lima", socialName: "a".repeat(201) }))
      .toBe("o nome social pode ter até 200 caracteres");
    expect(namesProblem({ ...EMPTY_NAMES, fullName: "João Carlos Lima", motherName: "a".repeat(201) }))
      .toBe("o nome da mãe pode ter até 200 caracteres");
    expect(namesProblem({ fullName: "João Carlos Lima", socialName: "Joana Lima", motherName: "" })).toBeNull();
  });

  it("corpo: espaços normalizados e opcionais só quando preenchidos", () => {
    expect(namesBody({ fullName: "  João   Carlos Lima ", socialName: " ", motherName: "Maria  Lima" }))
      .toEqual({ full_name: "João Carlos Lima", mother_name: "Maria Lima" });
    expect(namesBody({ fullName: "João Carlos Lima", socialName: "Joana Lima", motherName: "" }))
      .toEqual({ full_name: "João Carlos Lima", social_name: "Joana Lima" });
  });
});
```

Em `src/modules/Attendance.test.tsx` (o nome completo agora é exigido; os testes que validam passam a preenchê-lo, e o corpo esperado ganha `full_name`):

```diff
--- a/src/modules/Attendance.test.tsx
+++ b/src/modules/Attendance.test.tsx
@@ it("busca, exige a caixa do documento e o perfil conferido, e valida", async () => {
     fireEvent.change(screen.getByLabelText("Data de nascimento (documento)"), { target: { value: "1990-05-10" } });
     fireEvent.change(screen.getByLabelText("Sexo (documento)"), { target: { value: "male" } });
+    expect(validate.disabled).toBe(true);
+    expect(screen.getByText("informe o nome completo como no documento")).not.toBeNull();
+    fireEvent.change(screen.getByLabelText("Nome completo (documento)"), { target: { value: "Carlos Souza" } });
     expect(validate.disabled).toBe(false);
     fireEvent.click(validate);
     await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456",
-      { birth_date: "1990-05-10", sex: "male", gender_identity: null }));
+      { birth_date: "1990-05-10", sex: "male", gender_identity: null, full_name: "Carlos Souza" }));
@@ describe("Attendance — perfil conferido no documento", () => {
     await screen.findByText("(**) *****-5432");
     fireEvent.click(screen.getByLabelText("Conferi o documento com foto e o CPF confere"));
+    fireEvent.change(screen.getByLabelText("Nome completo (documento)"), { target: { value: "Maria Aparecida Souza" } });
     return screen.getByRole("button", { name: "Validar cadastro" }) as HTMLButtonElement;
   }
@@ it("mostra o declarado, já preenche os campos e envia o conferido", async () => {
     await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456",
-      { birth_date: "1963-04-02", sex: "female", gender_identity: "cis_woman" }));
+      { birth_date: "1963-04-02", sex: "female", gender_identity: "cis_woman", full_name: "Maria Aparecida Souza" }));
@@ it("o atendente corrige o sexo e tira a identidade", async () => {
     await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456",
-      { birth_date: "1963-04-02", sex: "male", gender_identity: null }));
+      { birth_date: "1963-04-02", sex: "male", gender_identity: null, full_name: "Maria Aparecida Souza" }));
@@ describe("Attendance — CADSUS no balcão (módulo 16)", () => {
-  const profileBody = { birth_date: "1963-04-02", sex: "female", gender_identity: null };
+  const profileBody = { birth_date: "1963-04-02", sex: "female", gender_identity: null, full_name: "Maria Aparecida Souza" };
@@   async function search() {
     fireEvent.change(screen.getByLabelText("Sexo (documento)"), { target: { value: "female" } });
+    fireEvent.change(screen.getByLabelText("Nome completo (documento)"), { target: { value: "Maria Aparecida Souza" } });
     fireEvent.click(screen.getByLabelText("Conferi o documento com foto e o CPF confere"));
   }
```

E um `describe` novo no fim do arquivo:

```tsx
describe("Attendance — nomes no balcão (módulo 19)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.lookupCitizen, api.verifyCitizen, api.listActiveUnits, api.getMyProfessional ]) {
      mocked(fn).mockReset();
    }
    mocked(api.fetchCurrentSession).mockResolvedValue(session("citizen_verifier"));
    mocked(api.listActiveUnits).mockResolvedValue([]);
    mocked(api.getMyProfessional).mockResolvedValue(null);
    mocked(api.lookupCitizen).mockResolvedValue(found);
    mocked(api.verifyCitizen).mockResolvedValue(undefined);
  });

  async function openForm() {
    renderAttendance();
    fireEvent.change(await screen.findByLabelText("CPF do cidadão (validação)"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Código de validação"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar validação" }));
    await screen.findByText("(**) *****-5432");
    fireEvent.change(screen.getByLabelText("Data de nascimento (documento)"), { target: { value: "1970-01-02" } });
    fireEvent.change(screen.getByLabelText("Sexo (documento)"), { target: { value: "female" } });
    fireEvent.click(screen.getByLabelText("Conferi o documento com foto e o CPF confere"));
  }

  it("social e mãe vão quando preenchidos", async () => {
    await openForm();
    fireEvent.change(screen.getByLabelText("Nome completo (documento)"), { target: { value: "João Carlos Lima" } });
    fireEvent.change(screen.getByLabelText("Nome social (opcional)"), { target: { value: "Joana Lima" } });
    fireEvent.change(screen.getByLabelText("Nome da mãe (opcional)"), { target: { value: "Maria Lima" } });
    fireEvent.click(screen.getByRole("button", { name: "Validar cadastro" }));
    await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456", {
      birth_date: "1970-01-02", sex: "female", gender_identity: null,
      full_name: "João Carlos Lima", social_name: "Joana Lima", mother_name: "Maria Lima"
    }));
  });

  it("recusa do nome pelo api é traduzida", async () => {
    mocked(api.verifyCitizen).mockRejectedValue(new ApiError(422, { error: "invalid_full_name" }, "422"));
    await openForm();
    fireEvent.change(screen.getByLabelText("Nome completo (documento)"), { target: { value: "João Carlos Lima" } });
    fireEvent.click(screen.getByRole("button", { name: "Validar cadastro" }));
    expect(await screen.findByText("confira o nome completo no documento (3 a 200 caracteres)")).not.toBeNull();
  });
});
```

Em `src/modules/attendance/CheckIn.test.tsx`, o mock ganha `completeCitizenNames: vi.fn()` (e o `beforeEach` o zera), e dois testes no `describe("CheckIn")`:

```tsx
  it("par validado sem nome completo: completar nomes no check-in", async () => {
    mocked(api.lookupCheckIn).mockResolvedValue({ ...foundVerified,
      citizen: { ...foundVerified.citizen, names: { full_name_set: false, display_name: null }, verification_id: "v1" } });
    mocked(api.completeCitizenNames).mockResolvedValue(undefined);
    renderCheckIn();
    fireEvent.change(screen.getByLabelText("CPF do cidadão (check-in)"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Código de check-in"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar check-in" }));
    const box = await screen.findByRole("region", { name: "Completar nomes do cadastro" });
    fireEvent.change(within(box).getByLabelText("Nome completo (documento)"), { target: { value: "João Carlos Lima" } });
    fireEvent.change(within(box).getByLabelText("Nome social (opcional)"), { target: { value: "Joana Lima" } });
    fireEvent.click(within(box).getByRole("button", { name: "Salvar nomes" }));
    await waitFor(() => expect(api.completeCitizenNames).toHaveBeenCalledWith("v1", { full_name: "João Carlos Lima", social_name: "Joana Lima" }));
    expect(await screen.findByText("Nomes registrados no cadastro")).not.toBeNull();
  });

  it("check-in que validou agora oferece completar os nomes; com nome, nada aparece", async () => {
    mocked(api.lookupCheckIn).mockResolvedValue(foundDeclared);
    mocked(api.checkIn).mockResolvedValue({ attendance: { id: "a1" }, verified: true, verification_id: "v2" });
    renderCheckIn();
    fireEvent.change(screen.getByLabelText("CPF do cidadão (check-in)"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Código de check-in"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar check-in" }));
    expect(screen.queryByRole("region", { name: "Completar nomes do cadastro" })).toBeNull();
    fireEvent.click(await screen.findByLabelText("Conferi o documento com foto e o CPF confere"));
    fireEvent.click(screen.getByRole("button", { name: "Iniciar atendimento" }));
    expect(await screen.findByRole("region", { name: "Completar nomes do cadastro" })).not.toBeNull();
  });
```

(`within` entra no import de `@testing-library/react` do arquivo.)

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/citizenNames.test.ts src/modules/Attendance.test.tsx src/modules/attendance/CheckIn.test.tsx`
Expected: FAIL — `Failed to resolve import "./citizenNames"`; no balcão, `Unable to find a label with the text of: Nome completo (documento)`; no check-in, `Unable to find role="region" and name "Completar nomes do cadastro"`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/citizenNames.ts
// Nomes conferidos no documento (módulo 19; spec §3; contratos §2): completo
// obrigatório (3–200), social e mãe opcionais (até 200). Nome de exibição =
// social, senão completo (quem decide é o api). Nunca em URL nem em log.
import type { CitizenNamesInput } from "./api";

export const NAME_MAX = 200;
export const FULL_NAME_MIN = 3;

export interface NamesDraft { fullName: string; socialName: string; motherName: string }
export const EMPTY_NAMES: NamesDraft = { fullName: "", socialName: "", motherName: "" };

const clean = (s: string) => s.trim().replace(/\s+/g, " ");

export function namesProblem(d: NamesDraft): string | null {
  const full = clean(d.fullName);
  if (full.length < FULL_NAME_MIN) return "informe o nome completo como no documento";
  if (full.length > NAME_MAX) return `o nome completo pode ter até ${NAME_MAX} caracteres`;
  if (clean(d.socialName).length > NAME_MAX) return `o nome social pode ter até ${NAME_MAX} caracteres`;
  if (clean(d.motherName).length > NAME_MAX) return `o nome da mãe pode ter até ${NAME_MAX} caracteres`;
  return null;
}

export function namesBody(d: NamesDraft): CitizenNamesInput {
  const social = clean(d.socialName);
  const mother = clean(d.motherName);
  return { full_name: clean(d.fullName), ...(social ? { social_name: social } : {}), ...(mother ? { mother_name: mother } : {}) };
}
```

```tsx
// src/modules/attendance/NamesFields.tsx
// Nomes do documento no balcão (módulo 19). Sem FrozenTextNotice: o aviso
// padrão pede para não escrever nomes, e aqui o nome é o dado conferido.
import type { CSSProperties } from "react";
import type { NamesDraft } from "../../lib/citizenNames";
import { inputStyle } from "../../components/formStyles";

interface Props { value: NamesDraft; onChange(next: NamesDraft): void; problem: string | null }

export function NamesFields({ value, onChange, problem }: Props) {
  const set = (patch: Partial<NamesDraft>) => onChange({ ...value, ...patch });
  return (
    <fieldset style={fieldset}>
      <legend style={{ fontSize: 12.5, fontWeight: 600 }}>Nomes (como no documento)</legend>
      <label style={labelStyle}>
        Nome completo (documento)
        <input value={value.fullName} autoComplete="off" style={inputStyle} onChange={(e) => set({ fullName: e.target.value })} />
      </label>
      <label style={labelStyle}>
        Nome social (opcional)
        <input value={value.socialName} autoComplete="off" style={inputStyle} onChange={(e) => set({ socialName: e.target.value })} />
      </label>
      <label style={labelStyle}>
        Nome da mãe (opcional)
        <input value={value.motherName} autoComplete="off" style={inputStyle} onChange={(e) => set({ motherName: e.target.value })} />
      </label>
      {problem && <small role="alert" style={{ fontSize: 12, color: "var(--down)" }}>{problem}</small>}
    </fieldset>
  );
}

const fieldset: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, border: "1px solid var(--rule)", borderRadius: 8, padding: 12, maxWidth: 420 };
const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
```

```tsx
// src/modules/attendance/CompleteNames.tsx
// Completar os nomes de um par já validado (módulo 19; contratos §2 e
// Divergência D2), no check-in. Sem o nome completo, a consulta não finaliza.
import { useState, type CSSProperties } from "react";
import { completeCitizenNames } from "../../lib/api";
import { attendanceError } from "../../lib/attendance";
import { EMPTY_NAMES, namesBody, namesProblem, type NamesDraft } from "../../lib/citizenNames";
import { buttonStyle, disabledButtonStyle } from "../../components/formStyles";
import { NamesFields } from "./NamesFields";

export function CompleteNames({ verificationId }: { verificationId: string }) {
  const [ names, setNames ] = useState<NamesDraft>(EMPTY_NAMES);
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const [ done, setDone ] = useState(false);
  const problem = namesProblem(names);

  async function save() {
    if (busy || problem) return;
    setBusy(true); setError(null);
    try {
      await completeCitizenNames(verificationId, namesBody(names));
      setDone(true);
    } catch (err) {
      setError(attendanceError(err));
    } finally {
      setBusy(false);
    }
  }

  if (done) return <p role="status" style={{ margin: 0, fontSize: 13, fontWeight: 600 }}>Nomes registrados no cadastro</p>;

  return (
    <section aria-label="Completar nomes do cadastro" style={box}>
      <strong>Completar nomes do cadastro</strong>
      <p style={{ margin: 0, fontSize: 12, color: "var(--ink3)" }}>
        Cadastro validado sem o nome completo: confira no documento. Sem ele, a consulta não pode ser finalizada.
      </p>
      <NamesFields value={names} onChange={setNames} problem={problem} />
      {error && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{error}</p>}
      <div>
        <button type="button" disabled={busy || problem !== null} style={busy || problem !== null ? disabledButtonStyle : buttonStyle}
          onClick={() => void save()}>
          Salvar nomes
        </button>
      </div>
    </section>
  );
}

const box: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 12, border: "1px solid var(--rule)", borderRadius: 8 };
```

Em `src/lib/attendance.ts` (as frases das recusas do contrato §2):

```diff
--- a/src/lib/attendance.ts
+++ b/src/lib/attendance.ts
@@ const MESSAGES: Record<string, string> = {
   invalid_screening_scope: "escolha uma das opções de acolhimento",
-  terminology_unavailable: "a CIAP-2 não está disponível agora — tente de novo em instantes"
+  terminology_unavailable: "a CIAP-2 não está disponível agora — tente de novo em instantes",
+  // Módulo 19 (contratos §2): nomes conferidos no documento.
+  invalid_full_name: "confira o nome completo no documento (3 a 200 caracteres)",
+  invalid_social_name: "o nome social pode ter até 200 caracteres",
+  invalid_mother_name: "o nome da mãe pode ter até 200 caracteres"
 };
```

(A última linha do `MESSAGES` depois do módulo 18 é a de `terminology_unavailable`; se for outra, acrescente as três depois dela.)

Em `src/modules/Attendance.tsx` (`Counter`):

```diff
--- a/src/modules/Attendance.tsx
+++ b/src/modules/Attendance.tsx
@@ imports
 import { CadsusCheck } from "./attendance/CadsusCheck";
+import { NamesFields } from "./attendance/NamesFields";
+import { EMPTY_NAMES, namesBody, namesProblem, type NamesDraft } from "../lib/citizenNames";
 import { hasFeature } from "../lib/features";
@@ function Counter({ cadsusOn }: { cadsusOn: boolean }) {
   const [ profile, setProfile ] = useState<ProfileCheckValue>(() => initialProfileCheck(null));
+  const [ names, setNames ] = useState<NamesDraft>(EMPTY_NAMES);
   const [ found, setFound ] = useState<Found | null>(null);
@@   function reset() {
     setState("form"); setCpf(""); setCode(""); setChecked(false); setFound(null); setError(null);
     setProfile(initialProfileCheck(null));
+    setNames(EMPTY_NAMES);
     setCadsusConfirmed(false);
   }
@@
   // "Hoje" no fuso da cidade: às 23h30 de São Paulo, amanhã ainda é futuro.
   const profileProblem = profileCheckProblem(profile, todayInCity());
+  // Módulo 19: nome completo obrigatório; social e mãe opcionais.
+  const namesIssue = namesProblem(names);
+  const blocked = !checked || busy || profileProblem !== null || namesIssue !== null;
 
   async function validate() {
-    if (busy || !checked || profileProblem) return;
+    if (blocked) return;
     setError(null);
@@
       const profileBody = {
-        birth_date: profile.birthDate, sex: profile.sex as Sex, gender_identity: profile.genderIdentity || null
+        birth_date: profile.birthDate, sex: profile.sex as Sex, gender_identity: profile.genderIdentity || null,
+        ...namesBody(names)
       };
@@
             <ProfileCheck declared={found.citizen.profile ?? null} value={profile} today={todayInCity()} onChange={setProfile} />
+            <NamesFields value={names} onChange={setNames} problem={namesIssue} />
             {cadsusOn && (
@@
               <button
                 type="button"
-                disabled={!checked || busy || profileProblem !== null}
+                disabled={blocked}
                 onClick={() => void validate()}
-                style={(!checked || busy || profileProblem !== null) ? disabledButtonStyle : buttonStyle}
+                style={blocked ? disabledButtonStyle : buttonStyle}
               >
                 Validar cadastro
```

Em `src/modules/attendance/CheckIn.tsx` (`CodeFlow`):

```diff
--- a/src/modules/attendance/CheckIn.tsx
+++ b/src/modules/attendance/CheckIn.tsx
@@ imports
 import { FrozenTextNotice } from "../../components/FrozenTextNotice";
+import { CompleteNames } from "./CompleteNames";
@@ function CodeFlow({ unit, onUnitInvalid }: Props) {
   const [ verified, setVerified ] = useState(false);
+  // Módulo 19 (Divergência D2): validação ativa criada agora pelo check-in, para completar os nomes.
+  const [ newVerificationId, setNewVerificationId ] = useState<string | null>(null);
@@   function reset() {
-    setState("form"); setCpf(""); setCode(""); setChecked(false); setFound(null); setVerified(false); setError(null);
+    setState("form"); setCpf(""); setCode(""); setChecked(false); setFound(null); setVerified(false); setError(null);
+    setNewVerificationId(null);
   }
@@ async function start() {
       const result = await checkIn(cpf, code, unit.id, checked);
       setVerified(result.verified);
+      setNewVerificationId(result.verified ? result.verification_id ?? null : null);
       setState("done");
@@
           {found.citizen.verification_level === "declared" && (
             <label style={{ ...labelStyle, flexDirection: "row", alignItems: "center", gap: 8 }}>
               <input type="checkbox" checked={checked} onChange={(e) => setChecked(e.target.checked)} />
               Conferi o documento com foto e o CPF confere
             </label>
           )}
+
+          {found.citizen.verification_level === "verified" && found.citizen.names?.full_name_set === false
+            && found.citizen.verification_id && (
+            <CompleteNames verificationId={found.citizen.verification_id} />
+          )}
@@
           {verified && <p role="status" style={{ margin: 0, fontSize: 13, fontWeight: 600 }}>cadastro validado</p>}
+          {newVerificationId && <CompleteNames verificationId={newVerificationId} />}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/citizenNames.test.ts src/modules/Attendance.test.tsx src/modules/attendance/CheckIn.test.tsx && npx tsc --noEmit`
Expected: PASS (2 novos na regra; `Attendance.test.tsx` com os de antes ajustados e 2 novos; `CheckIn.test.tsx` com 2 novos); `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/lib/citizenNames.ts src/lib/citizenNames.test.ts src/modules/attendance/NamesFields.tsx src/modules/attendance/CompleteNames.tsx src/modules/Attendance.tsx src/modules/Attendance.test.tsx src/modules/attendance/CheckIn.tsx src/modules/attendance/CheckIn.test.tsx src/lib/attendance.ts
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "feat: check full, social and mother names at in-person verification

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 13: Produção e-SUS — ficha com correção pendente

Adendo que muda dado estruturado de consulta com ficha **já aceita** gera uma linha `correction_pending` que não é enviada enquanto o reenvio depois do aceite não for confirmado (api#41; spec §6; contratos §6). A Produção mostra a situação, não oferece "Reenviar" e explica por quê.

**Files:**
- Modify: `src/lib/api.ts` (`FichaStatus`), `src/lib/production.ts` (`FICHA_STATUS`, `CORRECTION_PENDING_NOTE`), `src/modules/Production.tsx` (aviso no painel "Fichas")
- Test: `src/lib/production.test.ts`, `src/modules/Production.test.tsx`

**Interfaces:**
- Consumes: `ficha()`, `productionFixture()` (`src/test/recordModeFixtures.ts`, já com `last_error_codes` do módulo 18); `canResend`.
- Produces: `FichaStatus` com `"correction_pending"`; `FICHA_STATUS.correction_pending = { label: "correção pendente — não enviada", tone: "info" }`; `CORRECTION_PENDING_NOTE`; `canResend` continua só para `rejected`.

- [ ] **Step 1: Write the failing test**

Em `src/lib/production.test.ts`, no `describe` de papéis (o import de `ficha` já existe no arquivo; se não, venha de `../test/recordModeFixtures`):

```ts
  it("correção pendente (módulo 19) tem rótulo e nunca se reenvia", () => {
    expect(FICHA_STATUS.correction_pending).toEqual({ label: "correção pendente — não enviada", tone: "info" });
    expect(canResend([ "municipal_admin" ], ficha({ status: "correction_pending" }))).toBe(false);
  });
```

Em `src/modules/Production.test.tsx`, no `describe("Produção e-SUS (módulo 16)")`:

```tsx
  it("ficha com correção pendente: situação, aviso e sem Reenviar", async () => {
    m(api.getProduction).mockResolvedValue(productionFixture({
      fichas: [ ficha({ id: "f9", ficha_type: "atendimento_individual", status: "correction_pending", accepted_at: null }) ],
      fichas_total: 1
    }));
    renderWithProviders(<Production />);
    expect(await screen.findByText("correção pendente — não enviada")).not.toBeNull();
    expect(screen.getByText(CORRECTION_PENDING_NOTE)).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Reenviar ficha f9" })).toBeNull();
  });

  it("sem correção pendente na página, sem o aviso", async () => {
    renderWithProviders(<Production />);
    await screen.findByText("recusada");
    expect(screen.queryByText(CORRECTION_PENDING_NOTE)).toBeNull();
  });
```

(`CORRECTION_PENDING_NOTE` entra no import de `../lib/production` do arquivo de teste.)

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/production.test.ts src/modules/Production.test.tsx`
Expected: FAIL — `CORRECTION_PENDING_NOTE` não exportado e `expected undefined to deeply equal { label: 'correção pendente — não enviada', … }`.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`:

```diff
--- a/src/lib/api.ts
+++ b/src/lib/api.ts
@@
 export type CompetenceAlert = "none" | "attention" | "critical";
-export type FichaStatus = "pending" | "sending" | "accepted" | "rejected" | "failed";
+// `correction_pending` (módulo 19, contratos §6): correção de ficha já aceita,
+// guardada e não enviada até o reenvio depois do aceite ser confirmado (api#41).
+export type FichaStatus = "pending" | "sending" | "accepted" | "rejected" | "failed" | "correction_pending";
```

Em `src/lib/production.ts`:

```diff
--- a/src/lib/production.ts
+++ b/src/lib/production.ts
@@ export const FICHA_STATUS: Record<string, { label: string; tone: "neutral" | "info" | "ok" | "down" }> = {
   rejected: { label: "recusada", tone: "down" },
-  failed: { label: "falhou — sem novas tentativas", tone: "down" }
+  failed: { label: "falhou — sem novas tentativas", tone: "down" },
+  correction_pending: { label: "correção pendente — não enviada", tone: "info" }
 };
+
+// Módulo 19 (spec §6): adendo depois do aceite gera a correção, que fica parada.
+export const CORRECTION_PENDING_NOTE =
+  "Correções de fichas já aceitas ficam guardadas e não são enviadas enquanto o reenvio depois do aceite não for confirmado com o PEC. Não contam como pendentes.";
```

Em `src/modules/Production.tsx` (import e o aviso dentro do painel "Fichas", antes da tabela):

```diff
--- a/src/modules/Production.tsx
+++ b/src/modules/Production.tsx
@@
-  FICHA_STATUS, PRODUCTION_KEY, RESEND_STALE, alertBanner, canReadProduction, canResend, deadlinePhrase, errorCodeLabel,
+  CORRECTION_PENDING_NOTE, FICHA_STATUS, PRODUCTION_KEY, RESEND_STALE, alertBanner, canReadProduction, canResend, deadlinePhrase, errorCodeLabel,
@@
           <Panel title="Fichas" sub={`competência ${competenceLabel(data.competence)} · página ${page}`}>
             <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
+              {data.fichas.some((f) => f.status === "correction_pending") && (
+                <p role="status" style={deadlineStyle}>{CORRECTION_PENDING_NOTE}</p>
+              )}
               <DataTable<LediFicha> cols={cols} rows={data.fichas} rowKey={(f) => f.id} empty="nenhuma ficha nesta página" />
```

(A primeira linha do import é a que o módulo 18 deixou; acrescente `CORRECTION_PENDING_NOTE` no começo dela, como acima, mantendo o resto.)

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run src/lib/production.test.ts src/modules/Production.test.tsx && npx tsc --noEmit`
Expected: PASS (1 + 2 novos; os de antes sem mudança); `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod19 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 add src/lib/api.ts src/lib/production.ts src/lib/production.test.ts src/modules/Production.tsx src/modules/Production.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 commit -m "feat: show pending corrections of accepted fichas in e-SUS production

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 14: Suíte, build, revisão e prova no navegador

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd apps/dashboard/.claude/mod19 && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde (a base anotada no Ambiente de execução mais os 19 arquivos de teste novos deste plano). A CI roda os mesmos três; `dist/` está no `.gitignore`.

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19 log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e a contagem `0`. Se o symlink aparecer, **não** o adicione.

- [ ] **Step 2: Nada clínico vaza**

```bash
cd apps/dashboard/.claude/mod19
grep -rn "console\." src/lib/consultation.ts src/lib/clinicalRecord.ts src/lib/citizenNames.ts src/lib/outcome.ts src/lib/useAutosave.ts src/modules/consultation src/modules/clinicalRecord src/modules/ClinicalRecord.tsx src/modules/attendance/NamesFields.tsx src/modules/attendance/CompleteNames.tsx
grep -rn "localStorage\|sessionStorage" src/lib/consultation.ts src/lib/useAutosave.ts src/modules/consultation src/modules/clinicalRecord
grep -n "cpf=\|?q=\|subjective=\|reason=" src/lib/api.ts
grep -rn "full_name\b" src/modules/consultation src/modules/attendance/UnitQueue.tsx
```

Expected: os quatro sem saída — nenhum `console`, nenhum armazenamento do navegador no prontuário, nenhum texto ou CPF em URL, e o nome completo (de registro) não aparece no painel do paciente nem na fila (só `display_name`).

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§3, §4, §5, §6, §7), o ADR 0031 e o arquivo de contratos do módulo 19 (inteiro). Pontos de atenção:
- a recepção nunca recebe ação nem leitura do prontuário ("Consulta" atrás de `canCare && clinicalRecord`; o item "Prontuário" só para profissional e admin; nenhuma rota do prontuário chamada sem esses papéis); a fila mostra só nome de exibição e cor;
- texto clínico e nomes só em corpo de POST/PATCH; consultas de leitura clínica com `gcTime: 0`; o cache da abertura é apagado quando ela acaba;
- o salvamento automático nunca sobrescreve o formulário com a resposta, nunca manda valor velho por último, e "Finalizar" espera o salvamento em curso;
- o desfecho da finalização é o mesmo corpo e a mesma tela do "Encerrar" (`OutcomeFields`), e o `ClosePanel` não mudou de comportamento;
- incluir problema já ativo vira "avaliado"; CID-10 só aparece para o CBO que pode (busca e justificativa de exame);
- step-up só pelo `SensitiveAction`; a contagem dos 30 minutos e o 403 `opening_required` encerram a leitura;
- interface sem redesign: só componentes existentes; nenhum componente comum mudou.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Rode o api da branch do módulo 19 na porta **3036**, com a semente da spec §9 (Curitiba com `record_mode = record` e `clinical_record` ligados em dev; cidadã validada com nome completo e social; médica e enfermeira da semente com vínculo na UBS; uma consulta finalizada com diabetes (CIAP-2 T90) ativo e um adendo). Depois o Vite do worktree apontando para ele:

```bash
cd apps/dashboard/.claude/mod19 && VITE_API_PROXY_TARGET=http://localhost:3036 npx vite --port 5186 --host 0.0.0.0
```

Abra `http://curitiba.localhost:5186/dashboard/`. O usuário faz o login; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Confira com screenshot:
- como a recepção (`admin@curitiba.demo`, que tem `citizen_verifier` em dev): validação presencial pedindo "Nome completo (documento)"; check-in da cidadã da semente; na fila, nome de exibição e cor, sem "Consulta"; o menu sem "Prontuário" se o usuário só tiver `citizen_verifier` (com `municipal_admin`, o item aparece só com o relatório);
- como a médica: "Chamar" abre "Consulta do atendimento" com o nome social, idade, sexo, T90 ativo "desde …", a escuta do dia e a consulta anterior; "Abrir" a anterior mostra o adendo; "Iniciar consulta"; S/O/A/P com "salvo às HH:MM"; PA 150/95; T90 avaliado; incluir K86 pela busca "hipertensão"; buscar "diabetes" e escolher T90 (vira "avaliado"); conduta; exame "hemoglobina glicada" com CID-10 E11; "Finalizar consulta" com "Retorno" → a pessoa sai da fila, o pedido aparece em Pedidos;
- na consulta finalizada: "Imprimir" abre o PDF numa janela nova; "Adendo" com motivo e mudança de conduta;
- recarregar a página no meio de outro rascunho e "Iniciar consulta" retoma o texto;
- como a enfermeira, em Prontuário: CPF da cidadã, "Busca ativa", step-up com o autenticador → leitura marcada "abertura justificada" com a contagem; abrir a consulta e fazer um adendo (com a abertura);
- como `admin@curitiba.demo` em Prontuário: o relatório "Aberturas fora de contexto" lista a abertura da enfermeira com CPF mascarado e motivo;
- em Produção e-SUS (com `ledi_export` ligado): a ficha de atendimento individual da consulta; depois do adendo numa ficha já aceita (o api da semente marca uma como aceita, se houver), a linha "correção pendente — não enviada" com o aviso.

- [ ] **Step 5:** **Pare.** O merge do dashboard só vem depois do merge do api do módulo 19, e só com autorização explícita do usuário. A ordem de deploy é api → dashboard (contratos §8; spec §10); o interruptor `clinical_record` é ligado pelo maintenance, só em dev/staging.

---

## Divergências propostas ao contrato

Formatos que este plano precisou e que o contrato não traz (ou traz de outro jeito). O plano já está escrito com a proposta; se uma for recusada, o ajuste fica dito.

1. **D1 — `consultation_id` no 409 `already_exists`.** `POST /attendance/attendances/:id/consultation` com rascunho já existente responde só `{ error: "already_exists" }`, e nenhuma rota devolve o rascunho de um atendimento (o `<record>` lista só consultas finalizadas). Sem o id, quem recarregou a página no meio da consulta perde o caminho de volta ao rascunho. Proposta: `{ "error": "already_exists", "consultation_id": "<id>" }` (só para a autora; para outra pessoa, o 409 sem o id, e o `GET` daria 403 `not_author`). Recusada, a tela diz "a consulta deste atendimento já foi iniciada, mas não foi possível retomá-la — recarregue a página" (`existingConsultationId` devolve `null`).
2. **D2 — onde completar os nomes de par já validado.** O contrato pede `POST /attendance/verifications/:id/names` (`citizen_verifier`) e põe `names` em `POST /attendance/lookup`, mas a busca de validação recusa par já validado (409 `already_verified`, `Citizens::LookupForVerification`), o wpda não emite código de validação para ele (`Citizens::IssueVerificationCode`), e o histórico com os ids das validações é só do `municipal_admin` — a recepção não tem como chegar ao `:id`. E o check-in **valida sem nome** (`Attendances::CheckIn.verify_once`), criando pares validados sem nome todo dia. Proposta: `:id` é a validação ativa; `POST /attendance/check_ins/lookup` devolve, para par validado, `citizen.names: { full_name_set, display_name }` e `citizen.verification_id`; `POST /attendance/check_ins` devolve `verification_id` quando validou. O dashboard oferece "Completar nomes do cadastro" no check-in (antes de iniciar, ou logo depois de validar). Recusada, só o balcão de validação pede os nomes (pares novos), e a consulta de quem foi validado sem nome para em `patient_name_missing`.
3. **D3 — `display_name` no item da fila.** A spec §5 diz "fila mostra só nome de exibição e cor", mas o contrato não acrescenta o nome ao `GET /attendance/units/:id/queue`. Proposta: `display_name: string | null` em cada item (nome social, senão completo; `null` sem nome), para os dois papéis. Recusada, a coluna "Nome" mostra "—" (o tipo já é opcional).
4. **D4 — forma de `changes` do adendo.** O contrato dá `"changes": {...}`. Proposta: `{ "evaluated_problems"?: [<item avaliado, mesma forma da consulta>], "conducts"?: [<código>], "exam_requests"?: [{ sigtap_code, label, cid10_justification? }] }`, em que `evaluated_problems` são eventos novos sobre a lista do paciente e `conducts`/`exam_requests` são as **listas finais** (só vêm quando mudaram); a mesma forma volta em `addenda[].changes` (ou `null`). E `opening_id` aceito também quando a autora faz adendo pela abertura (o api ignora).
5. **D5 — corpo do `PATCH /attendance/consultations/:id`.** O contrato não diz o corpo do autosave. Proposta: todos os campos editáveis a cada salvamento (`subjective`, `objective`, `assessment`, `plan`, `vitals`, `care_type` ou `null`, `evaluated_problems`, `conducts`, `exam_requests`), substituindo o rascunho; `vitals` como no módulo 18 (string vazia/ausente = não medido).
6. **D6 — janela do deploy com o nome completo obrigatório.** Com o api novo e o dashboard antigo no ar (rollout api → dashboard, spec §10), o balcão antigo não manda `full_name` e toda validação presencial cairia em 422 `invalid_full_name`. Proposta: o api aceitar validação sem `full_name` (o par fica sem nome, como os de antes) até o dashboard novo estar no ar, ou api e dashboard no mesmo deploy.
7. **D7 — `<record>.consultations` e `cid10_justification`.** Este plano supõe que `consultations` traz só consultas finalizadas (`finalized_at` não nulo) e que a justificativa CID-10 de exame segue a mesma regra de CBO dos problemas (`cid10_allowed_for_cbo`; sem ela, o campo nem aparece). Proposta: o contrato dizer as duas coisas.
8. **D8 — impresso com recusa em JSON.** O dashboard lê o PDF com `fetch` (para mostrar o 409 em vez de uma página de erro na janela nova): `GET …/print` precisa responder `Content-Type: application/pdf` no sucesso e `{ "error": … }` em JSON nas recusas (como o resto do contrato), com `Content-Disposition: inline`.

## Self-review

- **Cobertura (spec §7 e contrato §2–§6):**
  - painel do paciente (nome de exibição, idade, sexo, problemas ativos, escuta do dia, últimas consultas) — Task 8; prontuário em contexto, iniciar/retomar, recusas (`citizen_not_verified`, `not_caller`, `cbo_not_allowed`, `feature_disabled`, `out_of_context`) — Tasks 2 e 9;
  - editor: S/O/A/P — Task 7; sinais com o componente do 18 — Task 7; problemas com avaliar/incluir/resolver/corrigir início — Tasks 2 e 5; busca CIAP-2/CID-10 e SIGTAP — Tasks 1, 4 e 5; condutas e tipo de atendimento de `consultation_options` — Tasks 5 e 7; exames — Task 5; encaminhamento e desfecho pela tela existente — Tasks 3 e 7; autosave com indicação — Tasks 6 e 7; Finalizar com as recusas — Tasks 2 e 7;
  - consulta finalizada: leitura, Adendo (motivo e mudanças), Imprimir — Task 8;
  - prontuário fora de contexto: CPF, motivo, step-up, "abertura justificada" com contagem dos 30 minutos — Task 10; relatório (`municipal_admin`) — Task 11;
  - validação presencial com nome completo, social e da mãe; completar nomes — Task 12; Produção `correction_pending` — Task 13;
  - recepção sem acesso — Tasks 9 e 10; `features` (`clinical_record`) — Tasks 1, 9 e 10; proxy `/clinical_record` — Task 1; spec §5 (LGPD na tela) — Global Constraints e Task 14; spec §8 (front com relógio fixo; prova no navegador) — Tasks 6–10 e 14.
- **Placeholders:** nenhum. Arquivos novos vêm inteiros; os existentes, por diff com contexto do estado depois do módulo 18 (os trechos que dependem do 18 dizem o que conferir).
- **Consistência de nomes:** `RECORD_KEY`, `CONSULTATION_KEY`, `OPTIONS_KEY`, `JUSTIFIED_KEY` (Task 2) usados nas Tasks 8–10; `consultationError`/`existingConsultationId` (Task 2) nas Tasks 4–11; `OutcomeDraft`/`outcomeView`/`outcomeBody`/`OutcomeFields` (Task 3) nas Tasks 3 e 7; `CodeSearch` (Task 4) na Task 5; `useAutosave`/`saveStatusLabel` (Task 6) na Task 7; `ConsultationEditor` (Task 7), `PatientPanel`/`ConsultationView`/`ConsultationLoader`/`AddendumForm`/`VitalsList` (Task 8) nas Tasks 9 e 10; `clinicalRecordError`/`OPENING_ENDED`/`reasonLabel` (Task 10) nas Tasks 10 e 11; `NamesFields`/`CompleteNames`/`namesBody` (Task 12); fixtures `NOW19`, `TODAY19`, `problem`, `summary`, `record`, `consultation`, `finalized`, `options`, `opening`, `openingRow` (Task 1) nas Tasks 2–11.
- **Review Focus:** 1 — Task 6 ("texto digitado durante o salvamento…", "flush espera…") e Task 7 ("Finalizar espera o salvamento pendente…"); 2 — Task 10 ("a contagem chega a zero…", "403 opening_required…"); 3 — Task 9 ("recepção vê nome e cor…") e Task 10 ("Prontuário no menu…"); 4 — Task 2 ("incluir código já ativo…") e Task 5 ("incluir T90 que já está na lista…"); 5 — Task 9 ("já iniciada: retoma…") e Task 7 ("not_draft no salvamento…").
