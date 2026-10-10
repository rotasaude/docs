# Módulo 19 (19c) — Documentos clínicos (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** nenhum além do que já está em `origin/main` (dashboard `8a0c0de`, com o 19a e o 19b). A Task 0 confere as peças que este plano altera. O **api do 19c** precisa estar rodando na branch dele (porta **3038**) para a prova (Task 11) e é **mergeado antes** deste (contrato §11: contracts → api → dashboard → maintenance).

**Goal:** Telas dos documentos clínicos no painel da cidade (F-19.17 a F-19.23, lado dashboard): na consulta, a aba **Documentos** (emitir atestado, declaração de comparecimento, receita e requisição de exames; lista com modo, estado da assinatura, Imprimir, Cancelar e "Cancelar e emitir outro"); os **medicamentos em uso** ao lado dos problemas (suspender, reativar, "informar medicamento em uso de fora"); a receita com busca no catálogo (REMUME primeiro, controlado bloqueado), texto livre, prescrição de enfermagem por protocolo e "Renovar receita"; o atestado com a autorização do CID; a **declaração na recepção**; os documentos em **Minhas consultas**; e, para o `municipal_admin`, a tela **Documentos clínicos** (CNPJ da cidade, REMUME e protocolos de enfermagem).

**Architecture:** O cliente HTTP vai para uma seção nova no fim de `src/lib/api.ts`, com os tipos copiados do contrato do 19c (§1–§7) e as formas de entrada que ele ainda não fixa (Divergência D1). As regras de tela ficam em `src/lib/clinicalDocuments.ts` (puras: rótulos, quem emite o quê pelo CBO, rascunhos ↔ corpo do POST, validação espelho dos 422, frases das recusas, CNPJ, protocolos vigentes). Os componentes novos ficam em `src/modules/documents/` (`ConsultationDocuments`, os quatro formulários, `MedicationSearch`, `CancelDocument`, `printDocument`), em `src/modules/consultation/MedicationsBlock.tsx`, em `src/modules/attendance/DeclarationPanel.tsx` e em `src/modules/ClinicalDocumentsAdmin.tsx` + `src/modules/clinicalDocumentsAdmin/`. As telas existentes ganham o mínimo: `PatientPanel` (um encaixe ao lado dos problemas), `ConsultationWorkspace` (abas Consulta/Documentos e o bloco de medicamentos), `MyConsultations` (documentos da consulta aberta), `UnitQueue`/`Attendance` (botão "Declaração"), `modules.ts`/`App.tsx` (item do admin), `features.ts` (`clinical_documents`), `signature.ts` (rótulo do tipo de documento) e o proxy do Vite (`/clinical_documents` e `/v/`). O dashboard nunca decide o que o api garante: modo de emissão, CBO autorizado, protocolo vigente, dose máxima, controlado e CNPJ são conferidos no api; a tela só espelha para não oferecer o que ele recusaria.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("../lib/api")` e `vi.stubGlobal("fetch")`), Vite 5 (base `/dashboard/`, proxy de dev).

**Spec:** `docs/superpowers/specs/2026-10-09-module-19c-clinical-documents-design.md` (§7 é deste plano; §3–§6 e §8 dão as regras; §11 F-19.17..23), `docs/adr/0033.md` e o contrato `docs/superpowers/plans/2026-10-09-module-19c-documents-contracts.md` (§1–§7; a seção "Divergências propostas ao contrato" no fim deste plano diz o que ele ainda não fixa).

## Global Constraints

- Rotas (contrato §3–§6; sessão da cidade, cookie): `GET /attendance/consultations/:id/documents` → `{ items }`; `POST /attendance/consultations/:id/documents { kind, content, replaces_document_id? }` → 201 `<document>` (403 `not_author`, `cbo_not_allowed`; 422 `invalid_content` com `field`, `cid_requires_authorization`, `companion_cid_not_allowed`, `controlled_not_allowed`, `not_in_nursing_protocol`, `above_protocol_max_dose`, `free_text_not_allowed`, `city_cnpj_missing`, `no_exam_requests`, `invalid_item` com `index`); `POST /attendance/attendances/:id/declarations { content }` (409 `attendance_not_found_or_closed_long_ago`); `GET /attendance/documents/:id`; `GET /attendance/documents/:id/print` (PDF); `POST /attendance/documents/:id/cancel { reason }` (step-up; 422 `invalid_reason`; 409 `already_cancelled`); `GET /attendance/patients/:id/medications`; `POST /attendance/consultations/:id/medications { action, … }` (409 `already_active`); `GET /attendance/consultations/:id/renewal` → `{ items }`; `POST /attendance/medications/search { q, unit_id? }` (503 `catalog_unavailable`); `GET`/`PUT /clinical_documents/city_profile` `{ cnpj }` (step-up; 422 `invalid_cnpj`; caminho da Divergência D2); `GET`/`POST /clinical_documents/remume`, `DELETE /clinical_documents/remume/:catalog_item_id`; `GET`/`POST /clinical_documents/nursing_protocols`, `POST /clinical_documents/nursing_protocols/:id/versions`.
- Erros `{ "error": "<reason>" }`; step-up 401 `{ "error": "mfa_required" }` (quem trata é o `SensitiveAction`); interruptor desligado 403 `{ "error": "feature_disabled", "feature": "clinical_documents" }` (ou `"clinical_record"`, pré-requisito); escrita devolve o objeto puro. **Toda escrita por cookie leva `Content-Type: application/json`, inclusive `DELETE`** (corpo `{}`; contrato do 19b §13).
- Valores (contrato §1): `kind` ∈ `sick_note` | `attendance_declaration` | `prescription` | `exam_requisition`; `issue_mode` ∈ `digital` | `paper`; `status` ∈ `issued` | `cancelled`; `sick_note.type` ∈ `leave` | `companion`; `companion_reason` ∈ `clt_473_x` | `clt_473_xi` | `clt_473_xii` | `other`; `period` ∈ `morning` | `afternoon` | `full_day`; medicamento em uso `active` | `suspended`, origem `prescription` | `external`; `route` ∈ `oral` | `sublingual` | `topical` | `ophthalmic` | `otic` | `nasal` | `inhalation` | `vaginal` | `rectal` | `intramuscular` | `intravenous` | `subcutaneous` | `other`. Valor desconhecido (api mais novo) aparece cru, nunca some nem vira `undefined`.
- Quem emite (spec §4, espelho do api): atestado só CBO de médico (`2251`, `2252`, `2253`) e dentista (`2232`); receita médico, dentista e enfermeiro (`2235`) — enfermeiro só com itens de protocolo vigente, sem texto livre; declaração e requisição de exames, qualquer CBO da autora; a recepção (`citizen_verifier`) só a declaração, pelo atendimento. Só a autora emite e cancela.
- Regras da tela (spec §4–§6): CID no atestado só com a autorização marcada (e nunca no de acompanhante); motivo do cancelamento ≥ 10 caracteres (sem os espaços das pontas), com step-up; cancelar receita avisa que não desfaz medicamento entregue; controlado aparece na busca da receita **bloqueado** ("receita de controle especial — 19d"); antimicrobiano avisa antes de emitir que sai em papel, 2 vias, válido por 10 dias; o modo (`digital`/`papel`) é o que o api devolve; documento com assinatura pendente relê a lista a cada 15 s; a lista de medicamentos só muda com a consulta em rascunho; CNPJ de 14 caracteres (alfanumérico da IN RFB 2.229/2024) com DV válido.
- Segurança e LGPD (spec §8): CID, medicamentos, posologia, motivo, nomes e CPF só no corpo de POST/PUT, nunca na URL; as rotas levam só ids; nada em `console`, `localStorage`/`sessionStorage` ou nome de arquivo; leituras clínicas com `gcTime: 0`; o termo da busca vai no corpo e o cache sai com a tela (`gcTime: 0`); a recepção não chama nada além da declaração; o operador nunca vê nada disto.
- Interface tem ciclo próprio: use os componentes e estilos existentes (`Panel`, `PageHeader`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `SensitiveAction`, `SegmentedControl`, `CodeSearch`, `SignatureMarker`, `formStyles`), sem redesign. Nenhum componente comum muda.
- Testes que dependem de "agora" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(new Date(NOW19C))` (`NOW19C = "2026-10-09T10:00:00-03:00"`) e `vi.useRealTimers()` no `afterEach`; a sessão de teste é criada **depois** do `setSystemTime` (o `mfa_verified_at` dela é "agora" e abre a janela de step-up). Funções puras recebem `today` como argumento.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.

## Ambiente de execução

- Worktree `apps/dashboard/.claude/mod19c`, branch `feat/mod-19c-documents` a partir de `origin/main` (Task 0). Todos os caminhos de arquivo das tasks são relativos a ele; os comandos partem da raiz do monorepo (`/Users/eduardovrocha/Development/ioit.solutions/rota-saude`).
- Testes (script `test` = `vitest run`): `cd apps/dashboard/.claude/mod19c && npx vitest run <arquivos>`.
- Tipos antes de cada commit (`typecheck` = `tsc --noEmit`; `noUnusedLocals` ligado e o `tsconfig` inclui os testes): `cd apps/dashboard/.claude/mod19c && npx tsc --noEmit`.
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo` e não define `base`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`, `(el as HTMLButtonElement).disabled`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`.
- Os diffs abaixo têm o contexto do arquivo em `origin/main` `8a0c0de`. Confira o trecho "antes" no arquivo real; se ele mudou, aplique a mesma intenção sobre o texto que estiver lá.
- O api do 19c roda em dev na porta **3038** (plano do api do 19c). O Vite do worktree sobe **num container da rede do compose**, na **5188**, com o proxy para `http://api:3038` (Task 11).

## Review Focus

1. **O documento muda entre a leitura da lista e o clique.** Outra aba (ou a mesma médica, minutos antes) já cancelou o atestado; "Cancelar" ou "Cancelar e emitir outro" chega com 409 `already_cancelled`. A tela diz o que houve, relê a lista e **não** abre o documento novo preenchido (seria um substituto de um documento que ela não cancelou agora). Teste: Task 6, "outra aba já cancelou (409 already_cancelled): diz, relê a lista e não abre o novo".
2. **Enfermeira sem protocolo vigente, cidade sem CNPJ, item fora do protocolo ou acima da dose.** Sem protocolo vigente a tela diz que a enfermagem não prescreve sem ele (e não oferece texto livre); acima da dose máxima não manda; `city_cnpj_missing` diz quem resolve e o formulário fica preenchido. Testes: Task 5, "enfermeira: escolhe o protocolo vigente, só itens dele…", "enfermeira sem protocolo vigente…" e "CNPJ da cidade faltando…".
3. **Controlado e antimicrobiano na receita.** O controlado aparece na busca (a médica precisa saber que existe), mas o botão vem desligado com "receita de controle especial — 19d" e nada entra na receita; o antimicrobiano avisa **antes** de emitir que sai em papel, 2 vias, válido por 10 dias. Testes: Task 3, "receita: controlado aparece bloqueado com o aviso do 19d"; Task 5, "controlado não entra…" e "antimicrobiano: avisa papel, 2 vias e 10 dias antes de emitir".
4. **CID sem autorização, ou no atestado de acompanhante.** Escolher o CID não basta: sem marcar a autorização do paciente, a tela não emite e diz por quê; trocar para acompanhante tira o CID. Testes: Task 2, "atestado: CID sem autorização…"; Task 4, "CID escolhido sem marcar a autorização…" e "acompanhante: … trocar para acompanhante tira o CID".
5. **A pessoa sai da fila enquanto a recepção preenche a declaração.** A fila relê a cada poucos segundos e o atendimento encerrado some da lista; o painel da declaração continua aberto (preso ao atendimento escolhido) e emite — o api aceita atendimento encerrado há até 30 dias. Teste: Task 8, "a pessoa sai da fila enquanto a recepção preenche: o painel continua e emite".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | conferência do main, worktree, base | 0 |
| `src/lib/api.ts`, `src/lib/api.clinicalDocuments.test.ts`, `src/test/documentFixtures.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `README.md` | tipos do contrato e entradas; cliente; interruptor `clinical_documents`; proxy `/clinical_documents` e `/v/` | 1 |
| `src/lib/clinicalDocuments.ts`, `src/lib/clinicalDocuments.test.ts` | rótulos; quem emite; rascunhos ↔ corpo; validação; frases; CNPJ; protocolos | 2 |
| `src/modules/documents/MedicationSearch.tsx` (+ teste), `src/modules/consultation/MedicationsBlock.tsx` (+ teste), `src/modules/consultation/PatientPanel.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/consultation/ConsultationWorkspace.documents.test.tsx` | busca no catálogo; medicamentos em uso ao lado dos problemas | 3 |
| `src/modules/documents/SickNoteForm.tsx`, `DeclarationForm.tsx`, `ExamRequisitionForm.tsx` (+ testes), `formStyles.ts` | atestado (CID autorizado), declaração, requisição de exames | 4 |
| `src/modules/documents/PrescriptionForm.tsx` (+ teste) | receita: catálogo, texto livre, enfermagem por protocolo, antimicrobiano | 5 |
| `src/modules/documents/ConsultationDocuments.tsx` (+ teste), `CancelDocument.tsx`, `printDocument.ts`, `src/lib/api.ts`, `src/lib/signature.ts`, `src/lib/signature.test.ts`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/consultation/ConsultationWorkspace.documents.test.tsx` | aba Documentos: lista, emissão, renovação, Imprimir, Cancelar, Cancelar e emitir outro | 6 |
| `src/modules/MyConsultations.tsx`, `src/modules/MyConsultations.documents.test.tsx` | documentos em Minhas consultas | 7 |
| `src/modules/attendance/DeclarationPanel.tsx`, `src/modules/attendance/UnitQueue.tsx`, `src/modules/attendance/UnitQueue.declaration.test.tsx`, `src/modules/Attendance.tsx` | declaração de comparecimento na recepção | 8 |
| `src/modules/ClinicalDocumentsAdmin.tsx` (+ teste), `src/modules/clinicalDocumentsAdmin/CnpjPanel.tsx`, `RemumePanel.tsx`, `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx` | admin: CNPJ e REMUME | 9 |
| `src/modules/clinicalDocumentsAdmin/NursingProtocolsPanel.tsx` (+ teste), `ProtocolForm.tsx`, `src/modules/ClinicalDocumentsAdmin.tsx` (+ teste) | admin: protocolos de enfermagem | 10 |
| — | suíte, build, revisão e prova no navegador | 11 |

**Estratégia de teste:** regras puras com tabela de casos (Task 2); cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — e que CID, motivo, medicamento e CNPJ nunca vão na URL e que o `DELETE` leva JSON (Task 1); cada componente com `vi.mock("../../lib/api")` cobrindo o caminho feliz, cada recusa que muda o comportamento da tela e as bordas do Review Focus; os formulários testados soltos (o `onSubmit` é um `vi.fn()`) e o contêiner com dublês só onde o comportamento já está coberto; a consulta e a fila com dublês dos componentes novos para conferir o encaixe. Imprimir troca `window.open` e `URL.createObjectURL` por dublês. A prova final é no navegador, contra o api do 19c com o PSC simulado (Task 11).

---

### Task 0: Conferência do main, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`8a0c0de`).
- Produces: worktree `apps/dashboard/.claude/mod19c` na branch `feat/mod-19c-documents`; a base anotada (arquivos e testes verdes).

- [ ] **Step 1: Confira as peças do 19a e do 19b que este plano altera**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/dashboard fetch origin
/opt/homebrew/bin/git -C apps/dashboard log --oneline -1 origin/main
for f in src/modules/consultation/ConsultationWorkspace.tsx src/modules/consultation/PatientPanel.tsx src/modules/consultation/CodeSearch.tsx \
         src/modules/signature/SignatureMarker.tsx src/components/SensitiveAction.tsx src/shell/SegmentedControl.tsx \
         src/modules/MyConsultations.tsx src/modules/attendance/UnitQueue.tsx src/test/campaignFixtures.tsx; do
  /opt/homebrew/bin/git -C apps/dashboard cat-file -e origin/main:$f && echo "ok $f"
done
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/lib/api.ts | grep -n "const postProfessional\|^async function binaryFetch\|export interface SignatureFile\|export type SignatureDocumentType\|export function searchTerminology"
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/lib/features.ts | grep -n "export type FeatureKey"
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/modules/consultation/PatientPanel.tsx | grep -n '<strong style={sub}>Problemas ativos</strong>'
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/modules/consultation/ConsultationWorkspace.tsx | grep -n '<PatientPanel record={record.data} onOpenConsultation={setViewing} />'
```

Expected: `8a0c0de fix: send JSON content type on signature DELETE requests` (ou mais novo); nove linhas `ok …`; as cinco declarações de `api.ts`; `export type FeatureKey = "ledi_export" | "cadsus_lookup" | "clinical_record" | "digital_signature";`; e as duas linhas de `PatientPanel` e `ConsultationWorkspace` que as Tasks 3 e 6 alteram. Se o main mudou essas linhas, anote: as tasks aplicam a mesma intenção sobre o texto real.

- [ ] **Step 2: Crie o worktree e ligue o `node_modules`** (`/.claude/` já está no `.gitignore` do dashboard)

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/dashboard worktree add .claude/mod19c -b feat/mod-19c-documents origin/main
ln -s ../../node_modules apps/dashboard/.claude/mod19c/node_modules
cd apps/dashboard/.claude/mod19c && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: tudo verde e `tsc-ok`. **Anote** a base (arquivos e testes). A Task 11 espera a base mais **14 arquivos de teste novos**.

---

### Task 1: Cliente HTTP dos documentos, tipos do contrato, interruptor e proxy

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim do arquivo)
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `README.md`
- Create: `src/test/documentFixtures.ts`
- Test: `src/lib/api.clinicalDocuments.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `postProfessional`, `binaryFetch`, `SignatureFile`, `ApiError`, `ATTENDANCE_BASE`, `SignatureBlock` (já em `api.ts`); `sessionWith` (`src/test/campaignFixtures.tsx`).
- Produces:
  - tipos: `DocumentKind`, `IssueMode`, `DocumentStatus`, `SickNoteType`, `CompanionReason`, `DeclarationPeriod`, `MedicationRoute`, `MedicationStatus`, `MedicationOrigin`, `CatalogItemRef`, `SickNoteContent`, `DeclarationContent`, `PrescriptionItem`, `PrescriptionContent`, `ExamRequisitionContent`, `DocumentContent`, `DocumentAuthor`, `ClinicalDocument`, `SickNoteInput`, `DeclarationInput`, `PrescriptionItemInput`, `PrescriptionInput`, `ExamRequisitionInput`, `DocumentInput`, `PatientMedication`, `MedicationChange`, `MedicationSearchItem`, `CityDocumentsProfile`, `RemumeEntry`, `MaxDose`, `ProtocolItem`, `NursingProtocolVersion`, `NursingProtocol`, `ProtocolVersionInput`, `NursingProtocolInput`;
  - funções: `listConsultationDocuments(consultationId): Promise<ClinicalDocument[]>`, `issueDocument(consultationId, input: DocumentInput, replacesDocumentId?: string): Promise<ClinicalDocument>`, `issueAttendanceDeclaration(attendanceId, content: DeclarationInput): Promise<ClinicalDocument>`, `getClinicalDocument(id): Promise<ClinicalDocument>`, `fetchDocumentPdf(id): Promise<SignatureFile>`, `cancelDocument(id, reason): Promise<ClinicalDocument>`, `listPatientMedications(patientId): Promise<PatientMedication[]>`, `changeMedication(consultationId, change: MedicationChange): Promise<PatientMedication>`, `getPrescriptionRenewal(consultationId): Promise<PrescriptionItem[]>`, `searchMedications(q, unitId?): Promise<MedicationSearchItem[]>`, `getCityDocumentsProfile(): Promise<CityDocumentsProfile>`, `setCityCnpj(cnpj): Promise<CityDocumentsProfile>`, `listRemume(): Promise<RemumeEntry[]>`, `addRemumeItem(catalogItemId, unitIds?): Promise<RemumeEntry>`, `removeRemumeItem(catalogItemId): Promise<void>`, `listNursingProtocols(): Promise<NursingProtocol[]>`, `createNursingProtocol(input): Promise<NursingProtocol>`, `addNursingProtocolVersion(id, input): Promise<NursingProtocol>`;
  - `FeatureKey` com `"clinical_documents"` (rótulo "Documentos clínicos");
  - fixtures: `NOW19C`, `TODAY19C`, `docUser(roles?, over?)` (sessão `us1` com `clinical_record` e `clinical_documents`), `catalogItem()`, `searchItem()`, `AMOXICILLIN`, `CLONAZEPAM`, `DIPYRONE_REF`, `DIPYRONE`, `prescriptionItem()`, `sickNoteDoc()`, `prescriptionDoc()`, `declarationDoc()`, `examDoc()`, `medication()`, `remumeEntry()`, `nursingProtocol()`.

- [ ] **Step 1: Write the failing test**

Fixtures (usadas nesta e nas próximas tasks):

```ts
// src/test/documentFixtures.ts
// Dados comuns aos testes do módulo 19c (documentos clínicos). Relógio dos
// testes: sexta, 2026-10-09 10:00 em São Paulo (-03:00). A autora é `us1`,
// a mesma das fixtures da consulta do 19a (CBO 225142, médica).
import type {
  CatalogItemRef, ClinicalDocument, MedicationSearchItem, NursingProtocol, PatientMedication, PrescriptionItem, RemumeEntry, SessionUser
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const NOW19C = "2026-10-09T10:00:00-03:00";
export const TODAY19C = "2026-10-09";

// Janela de step-up aberta (sessionWith põe `mfa_verified_at` = agora): crie a
// sessão DEPOIS de fixar o relógio. `{ mfa_verified_at: null }` força o código.
export function docUser(roles: string[] = [ "health_professional" ], over: Partial<SessionUser> = {}): SessionUser {
  return sessionWith(roles, {
    id: "us1", email_address: "medica@curitiba.demo", features: [ "clinical_record", "clinical_documents" ], ...over
  });
}

export function catalogItem(over: Partial<CatalogItemRef> = {}): CatalogItemRef {
  return {
    id: "ci1", catmat_code: 267656, label: "Metformina, cloridrato 850 mg, comprimido",
    active_ingredient: "cloridrato de metformina", strength: "850 mg", dosage_form: "comprimido", ...over
  };
}

export function searchItem(over: Partial<MedicationSearchItem> = {}): MedicationSearchItem {
  return { ...catalogItem(), antimicrobial: false, controlled: false, in_network: true, ...over };
}

export const AMOXICILLIN = searchItem({
  id: "ci2", catmat_code: 267621, label: "Amoxicilina 500 mg, cápsula", active_ingredient: "amoxicilina",
  strength: "500 mg", dosage_form: "cápsula", antimicrobial: true
});
export const CLONAZEPAM = searchItem({
  id: "ci3", catmat_code: 269999, label: "Clonazepam 2 mg, comprimido", active_ingredient: "clonazepam",
  strength: "2 mg", dosage_form: "comprimido", controlled: true, in_network: false
});
export const DIPYRONE_REF = catalogItem({
  id: "ci4", catmat_code: 268144, label: "Dipirona sódica 500 mg/mL, solução oral", active_ingredient: "dipirona sódica",
  strength: "500 mg/mL", dosage_form: "solução oral"
});
export const DIPYRONE: MedicationSearchItem = { ...DIPYRONE_REF, antimicrobial: false, controlled: false, in_network: true };

export function prescriptionItem(over: Partial<PrescriptionItem> = {}): PrescriptionItem {
  return {
    position: 1, catalog_item: catalogItem(), printed_description: "Cloridrato de metformina 850 mg — comprimido",
    quantity: 60, quantity_unit: "comprimido", route: "oral", dosage_instructions: "1 comprimido após o café e 1 após o jantar",
    duration_days: 30, continuous: true, antimicrobial: false, ...over
  };
}

function documentOf(over: Partial<ClinicalDocument>): ClinicalDocument {
  return {
    id: "doc1", kind: "sick_note", status: "issued", issue_mode: "paper", issued_at: "2026-10-09T09:40:00-03:00",
    cancelled_at: null, cancel_reason: null,
    author: { id: "us1", name: "Dra. Helena Prado", council: "CRM-PR 45678", cbo_code: "225142" },
    patient: { id: "pa1", display_name: "Joana Lima" }, consultation_id: "cs1", attendance_id: "a1",
    short_code: "K7Q2M9XW4P", verification_url: "https://curitiba.rotasaude.app/v/tok1", replaces_document_id: null,
    content: { type: "leave", days: 3, start_on: "2026-10-09", cid_authorized: false, note: null }, signature: null,
    ...over
  };
}

export function sickNoteDoc(over: Partial<ClinicalDocument> = {}): ClinicalDocument {
  return documentOf(over);
}

export function prescriptionDoc(over: Partial<ClinicalDocument> = {}): ClinicalDocument {
  return documentOf({
    id: "doc2", kind: "prescription", short_code: "P3H8T2LQ6R",
    content: { items: [ prescriptionItem() ], antimicrobial: false, copies: 1 }, ...over
  });
}

export function declarationDoc(over: Partial<ClinicalDocument> = {}): ClinicalDocument {
  return documentOf({
    id: "doc3", kind: "attendance_declaration", short_code: "D9W4N7KC3M",
    content: { date: "2026-10-09", period: "morning", unit_name: "UBS Centro" }, ...over
  });
}

export function examDoc(over: Partial<ClinicalDocument> = {}): ClinicalDocument {
  return documentOf({
    id: "doc4", kind: "exam_requisition", short_code: "E2R6Y8VB4T",
    content: { exams: [ { sigtap_code: "0202010503", label: "DOSAGEM DE HEMOGLOBINA GLICOSILADA", competence: "202609" } ], note: null },
    ...over
  });
}

export function medication(over: Partial<PatientMedication> = {}): PatientMedication {
  return {
    id: "pm1", catalog_item: catalogItem(), free_text: null, label: "Metformina 850 mg", dosage_summary: "1 cp 2x/dia",
    continuous: true, status: "active", origin: "prescription", started_on: "2026-03-10", updated_at: "2026-09-10T14:30:00-03:00",
    ...over
  };
}

export function remumeEntry(over: Partial<RemumeEntry> = {}): RemumeEntry {
  return { catalog_item: catalogItem(), unit_ids: [], ...over };
}

export function nursingProtocol(over: Partial<NursingProtocol> = {}): NursingProtocol {
  return {
    id: "np1", title: "Protocolo de enfermagem na atenção primária", number: "007/2025", year: 2025,
    current_version: { id: "npv1", valid_from: "2025-03-01", valid_until: null, items: [ { catalog_item: DIPYRONE_REF, max_dose: { quantity: 2, unit: "frasco" } } ] },
    ...over
  };
}
```

```ts
// src/lib/api.clinicalDocuments.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import {
  addNursingProtocolVersion, addRemumeItem, cancelDocument, changeMedication, createNursingProtocol, fetchDocumentPdf,
  getCityDocumentsProfile, getClinicalDocument, getPrescriptionRenewal, issueAttendanceDeclaration, issueDocument,
  listConsultationDocuments, listNursingProtocols, listPatientMedications, listRemume, removeRemumeItem, searchMedications, setCityCnpj
} from "./api";
import { medication, nursingProtocol, prescriptionItem, remumeEntry, searchItem, sickNoteDoc } from "../test/documentFixtures";

afterEach(() => vi.unstubAllGlobals());

function stub(body: unknown, status = 200) {
  const fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
    new Response(body === undefined ? null : JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fn);
  return fn;
}
const call = (fn: ReturnType<typeof stub>, i = 0) => fn.mock.calls[i] as [ string, RequestInit ];
// Toda escrita por cookie exige JSON (CSRF do api): método, header e corpo.
function expectJsonWrite(fn: ReturnType<typeof stub>, method: string, body: unknown, i = 0) {
  expect(call(fn, i)[1].method).toBe(method);
  expect((call(fn, i)[1].headers as Record<string, string>)["Content-Type"]).toBe("application/json");
  expect(JSON.parse(call(fn, i)[1].body as string)).toEqual(body);
}

describe("cliente dos documentos clínicos — consulta", () => {
  it("lista os documentos da consulta pelo id", async () => {
    const fn = stub({ items: [ sickNoteDoc() ] });
    expect((await listConsultationDocuments("cs1"))[0].short_code).toBe("K7Q2M9XW4P");
    expect(call(fn)[0]).toBe("/attendance/consultations/cs1/documents");
    expect(call(fn)[1].credentials).toBe("include");
  });

  it("emite com kind e content no corpo; o CID vai no corpo, nunca na URL", async () => {
    const fn = stub(sickNoteDoc(), 201);
    await issueDocument("cs1", { kind: "sick_note", content: { type: "leave", days: 3, start_on: "2026-10-09", cid10: { code: "J069" }, cid_authorized: true } });
    expect(call(fn)[0]).toBe("/attendance/consultations/cs1/documents");
    expectJsonWrite(fn, "POST", { kind: "sick_note", content: { type: "leave", days: 3, start_on: "2026-10-09", cid10: { code: "J069" }, cid_authorized: true } });
    expect(call(fn)[0]).not.toMatch(/J069/);
  });

  it("emitir outro manda replaces_document_id", async () => {
    const fn = stub(sickNoteDoc({ id: "doc9", replaces_document_id: "doc1" }), 201);
    await issueDocument("cs1", { kind: "exam_requisition", content: {} }, "doc1");
    expectJsonWrite(fn, "POST", { kind: "exam_requisition", content: {}, replaces_document_id: "doc1" });
  });

  it("declaração da recepção vai pelo atendimento", async () => {
    const fn = stub(sickNoteDoc(), 201);
    await issueAttendanceDeclaration("a1", { period: "morning" });
    expect(call(fn)[0]).toBe("/attendance/attendances/a1/declarations");
    expectJsonWrite(fn, "POST", { content: { period: "morning" } });
  });

  it("lê um documento e cancela com o motivo no corpo", async () => {
    let fn = stub(sickNoteDoc());
    await getClinicalDocument("doc1");
    expect(call(fn)[0]).toBe("/attendance/documents/doc1");
    fn = stub(sickNoteDoc({ status: "cancelled" }));
    await cancelDocument("doc1", "emitido com dias errados");
    expect(call(fn)[0]).toBe("/attendance/documents/doc1/cancel");
    expectJsonWrite(fn, "POST", { reason: "emitido com dias errados" });
  });

  it("o PDF é lido com a sessão e Accept application/pdf", async () => {
    const fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
      new Response("%PDF-1.7", { status: 200, headers: { "Content-Type": "application/pdf" } }));
    vi.stubGlobal("fetch", fn);
    const file = await fetchDocumentPdf("doc1");
    expect(await file.blob.text()).toBe("%PDF-1.7");
    expect(fn.mock.calls[0][0]).toBe("/attendance/documents/doc1/print");
    expect(((fn.mock.calls[0][1] as RequestInit).headers as Record<string, string>).Accept).toBe("application/pdf");
  });
});

describe("cliente dos documentos clínicos — medicamentos", () => {
  it("lista os medicamentos em uso do paciente", async () => {
    const fn = stub({ items: [ medication() ] });
    expect((await listPatientMedications("pa1"))[0].label).toBe("Metformina 850 mg");
    expect(call(fn)[0]).toBe("/attendance/patients/pa1/medications");
  });

  it("muda a lista pela consulta; o nome do medicamento de fora vai no corpo", async () => {
    const fn = stub(medication());
    await changeMedication("cs1", { action: "add_external", free_text: "Losartana 50 mg", dosage_summary: "1 cp/dia", continuous: true });
    expect(call(fn)[0]).toBe("/attendance/consultations/cs1/medications");
    expectJsonWrite(fn, "POST", { action: "add_external", free_text: "Losartana 50 mg", dosage_summary: "1 cp/dia", continuous: true });
    expect(call(fn)[0]).not.toMatch(/Losartana/);
  });

  it("renovação e busca no catálogo (termo no corpo; unit_id só quando há unidade)", async () => {
    let fn = stub({ items: [ prescriptionItem() ] });
    expect((await getPrescriptionRenewal("cs1"))[0].quantity).toBe(60);
    expect(call(fn)[0]).toBe("/attendance/consultations/cs1/renewal");
    fn = stub({ items: [ searchItem() ] });
    await searchMedications("metformina", "u1");
    expect(call(fn)[0]).toBe("/attendance/medications/search");
    expectJsonWrite(fn, "POST", { q: "metformina", unit_id: "u1" });
    fn = stub({ items: [] });
    await searchMedications("dipirona");
    expectJsonWrite(fn, "POST", { q: "dipirona" });
  });
});

describe("cliente dos documentos clínicos — cidade (municipal_admin)", () => {
  it("CNPJ: lê e grava com PUT JSON", async () => {
    let fn = stub({ cnpj: null });
    expect((await getCityDocumentsProfile()).cnpj).toBeNull();
    expect(call(fn)[0]).toBe("/clinical_documents/city_profile");
    fn = stub({ cnpj: "11222333000181" });
    await setCityCnpj("11222333000181");
    expect(call(fn)[0]).toBe("/clinical_documents/city_profile");
    expectJsonWrite(fn, "PUT", { cnpj: "11222333000181" });
  });

  it("REMUME: lista, inclui (unidades só quando escolhidas) e tira com DELETE JSON", async () => {
    let fn = stub({ items: [ remumeEntry() ] });
    expect((await listRemume())[0].catalog_item.id).toBe("ci1");
    expect(call(fn)[0]).toBe("/clinical_documents/remume");
    fn = stub(remumeEntry({ unit_ids: [ "u1" ] }), 201);
    await addRemumeItem("ci1", [ "u1" ]);
    expectJsonWrite(fn, "POST", { catalog_item_id: "ci1", unit_ids: [ "u1" ] });
    fn = stub(remumeEntry(), 201);
    await addRemumeItem("ci1", []);
    expectJsonWrite(fn, "POST", { catalog_item_id: "ci1" });
    fn = stub(undefined, 204);
    await removeRemumeItem("ci1");
    expect(call(fn)[0]).toBe("/clinical_documents/remume/ci1");
    expectJsonWrite(fn, "DELETE", {});
  });

  it("protocolos: lista, cria e cria versão", async () => {
    let fn = stub({ items: [ nursingProtocol() ] });
    expect((await listNursingProtocols())[0].number).toBe("007/2025");
    expect(call(fn)[0]).toBe("/clinical_documents/nursing_protocols");
    const input = { title: "Protocolo", number: "007/2025", year: 2025, valid_from: "2026-10-09", items: [ { catalog_item_id: "ci4", max_dose: { quantity: 2, unit: "frasco" } } ] };
    fn = stub(nursingProtocol(), 201);
    await createNursingProtocol(input);
    expectJsonWrite(fn, "POST", input);
    fn = stub(nursingProtocol(), 201);
    await addNursingProtocolVersion("np1", { valid_from: "2026-10-09", items: [ { catalog_item_id: "ci4" } ] });
    expect(call(fn)[0]).toBe("/clinical_documents/nursing_protocols/np1/versions");
    expectJsonWrite(fn, "POST", { valid_from: "2026-10-09", items: [ { catalog_item_id: "ci4" } ] });
  });
});
```

Em `src/lib/features.test.ts`, no teste "rótulo conhecido em português; desconhecido sai como a chave", antes da linha de `rnds_sync`:

```ts
    expect(featureLabel("clinical_documents")).toBe("Documentos clínicos");
    expect(hasFeature({ features: [ "clinical_documents" ] }, "clinical_documents")).toBe(true);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/lib/api.clinicalDocuments.test.ts src/lib/features.test.ts`
Expected: FAIL — as funções novas não existem em `./api` e `featureLabel("clinical_documents")` devolve a chave crua (e o `tsc` acusaria `"clinical_documents"` fora de `FeatureKey`).

- [ ] **Step 3: Write minimal implementation**

No fim de `src/lib/api.ts`:

```ts
// ─── Documentos clínicos (módulo 19c, ADR 0033; contrato 2026-10-09 §1–§7) ───
// CID, medicamentos, posologia, motivo e nomes só no corpo de POST/PUT, nunca
// em URL: as rotas levam só ids. Toda escrita por cookie leva JSON, inclusive
// o DELETE (corpo `{}`). A recepção só chama a declaração. As formas de
// entrada (`*Input`) são as da Divergência D1: o api deriva o resto
// (descrição impressa, item do catálogo, antimicrobiano, vias, validade, CNPJ,
// protocolo, rótulo do CID, nome da unidade e os exames da consulta).

const CLINICAL_DOCUMENTS_BASE = import.meta.env.VITE_CLINICAL_DOCUMENTS_BASE || "/clinical_documents";

export type DocumentKind = "sick_note" | "attendance_declaration" | "prescription" | "exam_requisition";
export type IssueMode = "digital" | "paper";
export type DocumentStatus = "issued" | "cancelled";
export type SickNoteType = "leave" | "companion";
export type CompanionReason = "clt_473_x" | "clt_473_xi" | "clt_473_xii" | "other";
export type DeclarationPeriod = "morning" | "afternoon" | "full_day";
export type MedicationRoute =
  | "oral" | "sublingual" | "topical" | "ophthalmic" | "otic" | "nasal" | "inhalation" | "vaginal" | "rectal"
  | "intramuscular" | "intravenous" | "subcutaneous" | "other";
export type MedicationStatus = "active" | "suspended";
export type MedicationOrigin = "prescription" | "external";

export interface CatalogItemRef {
  id: string; catmat_code: number; label: string; active_ingredient: string; strength: string; dosage_form: string;
}
export interface SickNoteContent {
  type: SickNoteType; days?: number; start_on?: string; companion_name?: string; companion_kinship?: string;
  companion_reason?: CompanionReason; cid10?: { code: string; label: string }; cid_authorized: boolean; note?: string | null;
}
export interface DeclarationContent {
  date: string; arrived_at?: string; left_at?: string; period?: DeclarationPeriod; unit_name: string;
  companion_name?: string; issuer_registration?: string;
}
export interface PrescriptionItem {
  position: number; catalog_item?: CatalogItemRef; free_text?: string; printed_description: string; quantity: number;
  quantity_unit: string; route: MedicationRoute; dosage_instructions: string; duration_days?: number; continuous: boolean;
  antimicrobial: boolean; reason_problem_id?: string;
}
export interface PrescriptionContent {
  items: PrescriptionItem[];
  nursing_protocol?: { id: string; title: string; number: string; year: number; version_id: string };
  city_cnpj?: string; antimicrobial: boolean; copies: 1 | 2; valid_until?: string;
}
export interface ExamRequisitionContent {
  exams: { sigtap_code: string; label: string; competence: string; cid10_justification?: string }[]; note?: string | null;
}
export type DocumentContent = SickNoteContent | DeclarationContent | PrescriptionContent | ExamRequisitionContent;

export interface DocumentAuthor { id: string; name: string; council: string | null; cbo_code: string }
export interface ClinicalDocument {
  id: string; kind: DocumentKind; status: DocumentStatus; issue_mode: IssueMode; issued_at: string;
  cancelled_at: string | null; cancel_reason: string | null; author: DocumentAuthor;
  patient: { id: string; display_name: string } | null; consultation_id: string | null; attendance_id: string;
  short_code: string; verification_url: string; replaces_document_id: string | null;
  content: DocumentContent; signature: SignatureBlock | null;
}

export interface SickNoteInput {
  type: SickNoteType; days?: number; start_on?: string; companion_name?: string; companion_kinship?: string;
  companion_reason?: CompanionReason; cid10?: { code: string }; cid_authorized: boolean; note?: string;
}
export interface DeclarationInput {
  period?: DeclarationPeriod; arrived_at?: string; left_at?: string; companion_name?: string; issuer_registration?: string;
}
export interface PrescriptionItemInput {
  catalog_item?: { id: string }; free_text?: string; quantity: number; quantity_unit: string; route: MedicationRoute;
  dosage_instructions: string; duration_days?: number; continuous: boolean; reason_problem_id?: string;
}
export interface PrescriptionInput { items: PrescriptionItemInput[]; nursing_protocol?: { version_id: string } }
export interface ExamRequisitionInput { note?: string }
export type DocumentInput =
  | { kind: "sick_note"; content: SickNoteInput }
  | { kind: "attendance_declaration"; content: DeclarationInput }
  | { kind: "prescription"; content: PrescriptionInput }
  | { kind: "exam_requisition"; content: ExamRequisitionInput };

export interface PatientMedication {
  id: string; catalog_item?: CatalogItemRef | null; free_text?: string | null; label: string; dosage_summary: string | null;
  continuous: boolean; status: MedicationStatus; origin: MedicationOrigin; started_on?: string | null; updated_at: string;
}
export type MedicationChange =
  | { action: "add_external"; catalog_item_id?: string; free_text?: string; dosage_summary?: string; continuous: boolean }
  | { action: "suspend" | "reactivate"; medication_id: string };
export interface MedicationSearchItem extends CatalogItemRef { antimicrobial: boolean; controlled: boolean; in_network: boolean }

export interface CityDocumentsProfile { cnpj: string | null }
export interface RemumeEntry { catalog_item: CatalogItemRef; unit_ids: string[] }
export interface MaxDose { quantity: number; unit: string }
export interface ProtocolItem { catalog_item: CatalogItemRef; max_dose?: MaxDose | null }
export interface NursingProtocolVersion { id: string; valid_from: string; valid_until?: string | null; items: ProtocolItem[] }
export interface NursingProtocol { id: string; title: string; number: string; year: number; current_version: NursingProtocolVersion | null }
export interface ProtocolVersionInput { valid_from: string; valid_until?: string; items: { catalog_item_id: string; max_dose?: MaxDose }[] }
export interface NursingProtocolInput extends ProtocolVersionInput { title: string; number: string; year: number }

const attendanceDocPath = (...parts: string[]) => [ ATTENDANCE_BASE, ...parts.map(encodeURIComponent) ].join("/");
const cityDocPath = (...parts: string[]) => [ CLINICAL_DOCUMENTS_BASE, ...parts.map(encodeURIComponent) ].join("/");

// Ler gera a trilha no api (regras de leitura do 19a).
export async function listConsultationDocuments(consultationId: string): Promise<ClinicalDocument[]> {
  return (await jsonFetch<{ items: ClinicalDocument[] }>(attendanceDocPath("consultations", consultationId, "documents"))).items;
}

export function issueDocument(consultationId: string, input: DocumentInput, replacesDocumentId?: string): Promise<ClinicalDocument> {
  return jsonFetch(attendanceDocPath("consultations", consultationId, "documents"), postProfessional({
    kind: input.kind, content: input.content, ...(replacesDocumentId ? { replaces_document_id: replacesDocumentId } : {})
  }));
}

export function issueAttendanceDeclaration(attendanceId: string, content: DeclarationInput): Promise<ClinicalDocument> {
  return jsonFetch(attendanceDocPath("attendances", attendanceId, "declarations"), postProfessional({ content }));
}

export function getClinicalDocument(id: string): Promise<ClinicalDocument> {
  return jsonFetch(attendanceDocPath("documents", id));
}

// PDF lido com fetch (e não por navegação) para a tela mostrar a recusa.
export function fetchDocumentPdf(id: string): Promise<SignatureFile> {
  return binaryFetch(attendanceDocPath("documents", id, "print"), "application/pdf");
}

// Step-up: quem trata 401 mfa_required é o SensitiveAction.
export function cancelDocument(id: string, reason: string): Promise<ClinicalDocument> {
  return jsonFetch(attendanceDocPath("documents", id, "cancel"), postProfessional({ reason }));
}

export async function listPatientMedications(patientId: string): Promise<PatientMedication[]> {
  return (await jsonFetch<{ items: PatientMedication[] }>(attendanceDocPath("patients", patientId, "medications"))).items;
}

export function changeMedication(consultationId: string, change: MedicationChange): Promise<PatientMedication> {
  return jsonFetch(attendanceDocPath("consultations", consultationId, "medications"), postProfessional(change));
}

export async function getPrescriptionRenewal(consultationId: string): Promise<PrescriptionItem[]> {
  return (await jsonFetch<{ items: PrescriptionItem[] }>(attendanceDocPath("consultations", consultationId, "renewal"))).items;
}

export async function searchMedications(q: string, unitId?: string): Promise<MedicationSearchItem[]> {
  const body = unitId ? { q, unit_id: unitId } : { q };
  return (await jsonFetch<{ items: MedicationSearchItem[] }>(attendanceDocPath("medications", "search"), postProfessional(body))).items;
}

export function getCityDocumentsProfile(): Promise<CityDocumentsProfile> {
  return jsonFetch(cityDocPath("city_profile"));
}

export function setCityCnpj(cnpj: string): Promise<CityDocumentsProfile> {
  return jsonFetch(cityDocPath("city_profile"), { method: "PUT", body: JSON.stringify({ cnpj }) });
}

export async function listRemume(): Promise<RemumeEntry[]> {
  return (await jsonFetch<{ items: RemumeEntry[] }>(cityDocPath("remume"))).items;
}

export function addRemumeItem(catalogItemId: string, unitIds?: string[]): Promise<RemumeEntry> {
  const body = unitIds && unitIds.length > 0 ? { catalog_item_id: catalogItemId, unit_ids: unitIds } : { catalog_item_id: catalogItemId };
  return jsonFetch(cityDocPath("remume"), postProfessional(body));
}

export async function removeRemumeItem(catalogItemId: string): Promise<void> {
  await jsonFetch<void>(cityDocPath("remume", catalogItemId), { ...postProfessional({}), method: "DELETE" });
}

export async function listNursingProtocols(): Promise<NursingProtocol[]> {
  return (await jsonFetch<{ items: NursingProtocol[] }>(cityDocPath("nursing_protocols"))).items;
}

export function createNursingProtocol(input: NursingProtocolInput): Promise<NursingProtocol> {
  return jsonFetch(cityDocPath("nursing_protocols"), postProfessional(input));
}

export function addNursingProtocolVersion(id: string, input: ProtocolVersionInput): Promise<NursingProtocol> {
  return jsonFetch(cityDocPath("nursing_protocols", id, "versions"), postProfessional(input));
}
```

(`binaryFetch` é declaração de função — está içada; `postProfessional` e `ATTENDANCE_BASE` são constantes declaradas antes no arquivo.)

Em `src/lib/features.ts`:

```ts
export type FeatureKey = "ledi_export" | "cadsus_lookup" | "clinical_record" | "digital_signature" | "clinical_documents";
```

```ts
const FEATURE_LABEL: Record<string, string> = {
  ledi_export: "Envio da produção ao e-SUS (LEDI)",
  cadsus_lookup: "Consulta ao CADSUS na validação presencial",
  clinical_record: "Prontuário da atenção primária",
  digital_signature: "Assinatura digital",
  clinical_documents: "Documentos clínicos"
};
```

Em `vite.config.ts`, no comentário do topo, depois da linha de `/signature`:

```ts
//   /clinical_documents → documentos clínicos da cidade (módulo 19c: CNPJ, REMUME e protocolos de enfermagem; municipal_admin).
//   ^/v/        → página pública de conferência do documento (módulo 19c), servida pelo api no host da cidade;
//                 em dev, só para a prova do QR code pela porta do Vite (regex: não casa /vite nem /viacep).
```

e no `proxy`, depois de `"/signature": proxy(TARGET)`:

```ts
      "/signature": proxy(TARGET),
      "/clinical_documents": proxy(TARGET),
      "^/v/": proxy(TARGET)
```

No `README.md`, na tabela "## Módulos", logo antes da linha de **Conta**:

```markdown
| Atendimento → consulta (módulo 19c) | Aba Documentos (atestado, declaração, receita, requisição de exames; imprimir e cancelar) e medicamentos em uso ao lado dos problemas | `health_professional` com `clinical_documents` |
| Atendimento → fila (módulo 19c) | "Declaração" de comparecimento pelo atendimento | recepção e profissionais, com `clinical_documents` |
| Cidade → Documentos clínicos (módulo 19c) | CNPJ da cidade, REMUME e protocolos de enfermagem (step-up nas escritas) | `municipal_admin` com `clinical_documents` |
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/lib/api.clinicalDocuments.test.ts src/lib/features.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19c
/opt/homebrew/bin/git add src/lib/api.ts src/lib/api.clinicalDocuments.test.ts src/test/documentFixtures.ts \
  src/lib/features.ts src/lib/features.test.ts vite.config.ts README.md
/opt/homebrew/bin/git commit -m "feat: add the clinical documents API client, feature key and dev proxy

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela dos documentos clínicos

**Files:**
- Create: `src/lib/clinicalDocuments.ts`
- Test: `src/lib/clinicalDocuments.test.ts`

**Interfaces:**
- Consumes: os tipos e `ApiError`/`errorCode` da Task 1; `featureDisabledKey`, `hasFeature` (`features.ts`); `describeActionError` (`actionErrors.ts`); `fmtDateTime` (`format.ts`).
- Produces (todas puras; `today` = `AAAA-MM-DD` no fuso da cidade):
  ```ts
  export const DOCUMENTS_KEY = "consultationDocuments";   // [ DOCUMENTS_KEY, consultationId ]
  export const MEDICATIONS_KEY = "patientMedications";    // [ MEDICATIONS_KEY, patientId ]
  export const PROTOCOLS_KEY = "nursingProtocols";
  export const REMUME_KEY = "remume";
  export const CITY_DOCS_PROFILE_KEY = "cityDocumentsProfile";
  export const CANCEL_REASON_MIN = 10;
  export const PENDING_REFETCH_MS = 15_000;
  export const DOCUMENTS_DISABLED: string; CANCEL_WARNING: string; ANTIMICROBIAL_NOTE: string; CONTROLLED_BLOCKED: string; NO_PROTOCOL: string;
  export const ROUTES: MedicationRoute[]; ROUTE_LABEL; COMPANION_REASON_LABEL; PERIOD_LABEL; MEDICATION_STATUS_LABEL; MEDICATION_ORIGIN_LABEL;
  export function kindLabel(kind: string): string;
  export function issueModeLabel(mode: string): string;
  export function documentSituation(doc: Pick<ClinicalDocument, "status" | "cancelled_at">): string;
  export function issuedNotice(doc: ClinicalDocument): string;
  export type ProfessionalKind = "physician" | "dentist" | "nurse" | "other";
  export function professionalKind(cbo: string | null | undefined): ProfessionalKind;
  export function kindsFor(cbo: string | null | undefined): DocumentKind[];
  export function canUseDocuments(user): boolean;
  export function canCancel(doc: ClinicalDocument, userId: string | null | undefined): boolean;
  export function documentsRefetchInterval(docs: ClinicalDocument[] | undefined): number | false;
  export function withDocument(list: ClinicalDocument[], doc: ClinicalDocument): ClinicalDocument[];
  export function cancelReasonProblem(reason: string): string | null;
  export interface SickNoteDraft { type; days; startOn; companionName; companionKinship; companionReason: CompanionReason | ""; cid: CodedOption | null; cidAuthorized: boolean; note }
  export function emptySickNote(today: string): SickNoteDraft; sickNoteProblem(d): string | null; sickNoteInput(d): SickNoteInput;
  export interface DeclarationDraft { date; mode: "period" | "times"; period: DeclarationPeriod | ""; arrivedAt; leftAt; companionName; issuerRegistration }
  export function emptyDeclaration(today: string, arrivedAt?: string): DeclarationDraft; declarationProblem(d): string | null; declarationInput(d): DeclarationInput;
  export type ItemMedication = CatalogItemRef & { antimicrobial?: boolean; controlled?: boolean; in_network?: boolean; max_dose?: MaxDose | null };
  export interface ItemDraft { key; medication: ItemMedication | null; freeText; quantity; quantityUnit; route: MedicationRoute | ""; dosage; durationDays; continuous: boolean }
  export interface PrescriptionDraft { items: ItemDraft[]; protocolVersionId: string }
  export function emptyItem(): ItemDraft; itemFromMedication(m: ItemMedication): ItemDraft; itemFromPrescription(item: PrescriptionItem): ItemDraft;
  export function prescriptionProblems(d: PrescriptionDraft, nurse: boolean): string[]; prescriptionInput(d): PrescriptionInput; hasAntimicrobial(d): boolean;
  export interface ExamRequisitionDraft { note: string }
  export type ReissueDraft = { kind: "sick_note"; draft: SickNoteDraft } | { kind: "attendance_declaration"; draft: DeclarationDraft } | { kind: "prescription"; draft: PrescriptionDraft } | { kind: "exam_requisition"; draft: ExamRequisitionDraft };
  export function reissueDraft(doc: ClinicalDocument): ReissueDraft | null;
  export interface ExternalDraft { medication: CatalogItemRef | null; freeText; dosage; continuous: boolean }
  export const EMPTY_EXTERNAL: ExternalDraft; externalProblem(d): string | null; externalChange(d): MedicationChange;
  export function medicationLine(m: PatientMedication): string; sortMedications(list): PatientMedication[];
  export function vigentProtocols(list: NursingProtocol[], today: string): NursingProtocol[]; protocolLabel(p: NursingProtocol): string;
  export interface ProtocolDraft { title; number; year; validFrom; validUntil; items: { item: CatalogItemRef; maxDose: string; maxUnit: string }[] }
  export function emptyProtocol(today): ProtocolDraft; protocolDraftFrom(p, today): ProtocolDraft; addProtocolItem(d, item): ProtocolDraft;
  export function protocolProblem(d, withHeader: boolean): string | null; protocolVersionInput(d): ProtocolVersionInput; protocolInput(d): NursingProtocolInput;
  export function cnpjDigits(text): string; isValidCnpj(text): boolean; formatCnpj(digits: string | null | undefined): string;
  export function documentError(err: unknown): string;
  ```

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/clinicalDocuments.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  ANTIMICROBIAL_NOTE, CONTROLLED_BLOCKED, DOCUMENTS_DISABLED, EMPTY_EXTERNAL, PENDING_REFETCH_MS, addProtocolItem, canCancel, canUseDocuments,
  cancelReasonProblem, declarationInput, declarationProblem, documentError, documentSituation, documentsRefetchInterval, emptyDeclaration,
  emptyItem, emptyProtocol, emptySickNote, externalChange, externalProblem, formatCnpj, hasAntimicrobial, isValidCnpj, issueModeLabel,
  issuedNotice, itemFromMedication, itemFromPrescription, kindLabel, kindsFor, medicationLine, prescriptionInput, prescriptionProblems,
  professionalKind, protocolDraftFrom, protocolInput, protocolLabel, protocolProblem, protocolVersionInput, reissueDraft, sickNoteInput,
  sickNoteProblem, sortMedications, vigentProtocols, withDocument, type ItemDraft
} from "./clinicalDocuments";
import {
  AMOXICILLIN, CLONAZEPAM, DIPYRONE_REF, TODAY19C, catalogItem, declarationDoc, examDoc, medication, nursingProtocol, prescriptionDoc,
  prescriptionItem, searchItem, sickNoteDoc
} from "../test/documentFixtures";

const filled = (over: Partial<ItemDraft> = {}): ItemDraft => ({
  ...itemFromMedication(searchItem()), quantity: "60", route: "oral", dosage: "1 cp 2x/dia", ...over
});

describe("rótulos e situação", () => {
  it("tipo, modo e situação; desconhecido cru", () => {
    expect(kindLabel("sick_note")).toBe("Atestado");
    expect(kindLabel("attendance_declaration")).toBe("Declaração de comparecimento");
    expect(kindLabel("prescription")).toBe("Receita");
    expect(kindLabel("exam_requisition")).toBe("Requisição de exames");
    expect(kindLabel("referral")).toBe("referral");
    expect(issueModeLabel("digital")).toBe("digital");
    expect(issueModeLabel("paper")).toBe("papel");
    expect(issueModeLabel("hybrid")).toBe("hybrid");
    expect(documentSituation({ status: "issued", cancelled_at: null })).toBe("emitido");
    expect(documentSituation({ status: "cancelled", cancelled_at: "2026-10-09T10:05:00-03:00" })).toBe("cancelado em 09/10/2026, 10:05");
  });

  it("o aviso depois de emitir diz como o documento sai", () => {
    expect(issuedNotice(sickNoteDoc())).toBe("Documento emitido (Atestado) em modo papel: imprima e assine à mão.");
    expect(issuedNotice(sickNoteDoc({ issue_mode: "digital" })))
      .toBe("Documento emitido (Atestado) em modo digital: a assinatura entra na sua fila; o PDF assinado sai depois dela.");
    expect(issuedNotice(prescriptionDoc({ content: { items: [ prescriptionItem({ antimicrobial: true }) ], antimicrobial: true, copies: 2, valid_until: "2026-10-19" } })))
      .toBe("Documento emitido (Receita) em modo papel: imprima as 2 vias e assine à mão.");
  });
});

describe("quem emite o quê (espelho do api)", () => {
  it("CBO de médico, dentista, enfermeiro e outros", () => {
    expect(professionalKind("225142")).toBe("physician");
    expect(professionalKind("225250")).toBe("physician");
    expect(professionalKind("225320")).toBe("physician");
    expect(professionalKind("223208")).toBe("dentist");
    expect(professionalKind("223565")).toBe("nurse");
    expect(professionalKind("223605")).toBe("other");
    expect(professionalKind(null)).toBe("other");
    expect(kindsFor("225142")).toEqual([ "sick_note", "attendance_declaration", "prescription", "exam_requisition" ]);
    expect(kindsFor("223208")).toEqual([ "sick_note", "attendance_declaration", "prescription", "exam_requisition" ]);
    expect(kindsFor("223565")).toEqual([ "attendance_declaration", "prescription", "exam_requisition" ]);
    expect(kindsFor("223605")).toEqual([ "attendance_declaration", "exam_requisition" ]);
  });

  it("documentos só com clinical_documents e nunca para o operador; cancela só a autora o emitido", () => {
    expect(canUseDocuments({ operator: false, features: [ "clinical_record", "clinical_documents" ] })).toBe(true);
    expect(canUseDocuments({ operator: false, features: [ "clinical_record" ] })).toBe(false);
    expect(canUseDocuments({ operator: true, features: [ "clinical_documents" ] })).toBe(false);
    expect(canUseDocuments(null)).toBe(false);
    expect(canCancel(sickNoteDoc(), "us1")).toBe(true);
    expect(canCancel(sickNoteDoc(), "us2")).toBe(false);
    expect(canCancel(sickNoteDoc({ status: "cancelled" }), "us1")).toBe(false);
    expect(canCancel(sickNoteDoc(), undefined)).toBe(false);
  });

  it("relê a lista só enquanto há assinatura pendente", () => {
    expect(documentsRefetchInterval(undefined)).toBe(false);
    expect(documentsRefetchInterval([ sickNoteDoc() ])).toBe(false);
    expect(documentsRefetchInterval([ sickNoteDoc({ signature: { mode: "pending", request_id: "sr1" } }) ])).toBe(PENDING_REFETCH_MS);
    expect(documentsRefetchInterval([ sickNoteDoc({ status: "cancelled", signature: { mode: "pending" } }) ])).toBe(false);
  });

  it("withDocument troca o que já existe e põe o novo no topo", () => {
    const list = [ sickNoteDoc(), prescriptionDoc() ];
    expect(withDocument(list, sickNoteDoc({ status: "cancelled" }))[0].status).toBe("cancelled");
    expect(withDocument(list, examDoc()).map((d) => d.id)).toEqual([ "doc4", "doc1", "doc2" ]);
  });

  it("motivo do cancelamento com 10 ou mais caracteres, sem contar as pontas", () => {
    expect(cancelReasonProblem("   curto   ")).toBe("descreva o motivo com pelo menos 10 caracteres");
    expect(cancelReasonProblem("dias errados")).toBeNull();
  });
});

describe("atestado", () => {
  it("afastamento: dias inteiros e início; manda o código do CID só com autorização", () => {
    const d = { ...emptySickNote(TODAY19C), days: "3" };
    expect(d.startOn).toBe(TODAY19C);
    expect(sickNoteProblem({ ...d, days: "0" })).toBe("informe os dias de afastamento (número inteiro, 1 ou mais)");
    expect(sickNoteProblem({ ...d, days: "2,5" })).toBe("informe os dias de afastamento (número inteiro, 1 ou mais)");
    expect(sickNoteProblem({ ...d, startOn: "" })).toBe("informe o início do afastamento");
    expect(sickNoteProblem(d)).toBeNull();
    expect(sickNoteInput({ ...d, note: "  repouso  " }))
      .toEqual({ type: "leave", days: 3, start_on: TODAY19C, cid_authorized: false, note: "repouso" });
    const withCid = { ...d, cid: { code: "J069", label: "Infecção aguda das vias aéreas superiores" }, cidAuthorized: true };
    expect(sickNoteInput(withCid)).toEqual({ type: "leave", days: 3, start_on: TODAY19C, cid10: { code: "J069" }, cid_authorized: true });
  });

  it("atestado: CID sem autorização não passa; acompanhante nunca leva CID", () => {
    const d = { ...emptySickNote(TODAY19C), days: "3", cid: { code: "J069", label: "IVAS" }, cidAuthorized: false };
    expect(sickNoteProblem(d)).toBe("o CID só entra com a autorização do paciente — marque a autorização ou tire o CID");
    const companion = { ...d, type: "companion" as const, companionName: "Marcos", companionKinship: "filho", companionReason: "clt_473_xi" as const };
    expect(sickNoteProblem(companion)).toBeNull();
    expect(sickNoteInput(companion)).toEqual({
      type: "companion", start_on: TODAY19C, companion_name: "Marcos", companion_kinship: "filho", companion_reason: "clt_473_xi", cid_authorized: false
    });
  });

  it("acompanhante: pede nome, parentesco e motivo", () => {
    const d = { ...emptySickNote(TODAY19C), type: "companion" as const };
    expect(sickNoteProblem(d)).toBe("informe o nome do acompanhante");
    expect(sickNoteProblem({ ...d, companionName: "Marcos" })).toBe("informe o parentesco");
    expect(sickNoteProblem({ ...d, companionName: "Marcos", companionKinship: "filho" })).toBe("escolha o motivo do acompanhamento");
  });
});

describe("declaração de comparecimento", () => {
  it("período ou horário, nunca os dois", () => {
    const byPeriod = emptyDeclaration(TODAY19C);
    expect(byPeriod.mode).toBe("period");
    expect(declarationProblem(byPeriod)).toBe("escolha o período");
    expect(declarationInput({ ...byPeriod, period: "morning", arrivedAt: "08:15" })).toEqual({ period: "morning" });
    const byTime = emptyDeclaration(TODAY19C, "08:15");
    expect(byTime.mode).toBe("times");
    expect(declarationProblem({ ...byTime, arrivedAt: "8:15" })).toBe("informe a hora de chegada (HH:MM)");
    expect(declarationProblem({ ...byTime, leftAt: "08:00" })).toBe("a saída precisa ser depois da chegada");
    expect(declarationInput({ ...byTime, leftAt: "10:30", companionName: " Marcos ", issuerRegistration: " 4521 " }))
      .toEqual({ arrived_at: "08:15", left_at: "10:30", companion_name: "Marcos", issuer_registration: "4521" });
  });
});

describe("receita", () => {
  it("item do catálogo e de texto livre viram o corpo do POST", () => {
    const d = { items: [ filled({ durationDays: "30", continuous: true }),
      { ...emptyItem(), freeText: " Soro fisiológico 0,9% ", quantity: "1", quantityUnit: "frasco", route: "nasal" as const, dosage: "2 jatos" } ],
      protocolVersionId: "" };
    expect(prescriptionProblems(d, false)).toEqual([]);
    expect(prescriptionInput(d)).toEqual({ items: [
      { catalog_item: { id: "ci1" }, quantity: 60, quantity_unit: "comprimido", route: "oral", dosage_instructions: "1 cp 2x/dia", duration_days: 30, continuous: true },
      { free_text: "Soro fisiológico 0,9%", quantity: 1, quantity_unit: "frasco", route: "nasal", dosage_instructions: "2 jatos", continuous: false }
    ] });
  });

  it("diz o que falta por item, e controlado nunca entra", () => {
    expect(prescriptionProblems({ items: [], protocolVersionId: "" }, false)).toEqual([ "inclua ao menos um medicamento" ]);
    expect(prescriptionProblems({ items: [ emptyItem() ], protocolVersionId: "" }, false)).toEqual([
      "item 1: escolha um medicamento do catálogo ou escreva em texto livre",
      "item 1: informe a quantidade (número inteiro)",
      "item 1: informe a unidade da quantidade",
      "item 1: escolha a via",
      "item 1: escreva a posologia"
    ]);
    expect(prescriptionProblems({ items: [ filled({ medication: CLONAZEPAM }) ], protocolVersionId: "" }, false))
      .toEqual([ `item 1: ${CONTROLLED_BLOCKED}` ]);
    expect(prescriptionProblems({ items: [ filled({ durationDays: "sete" }) ], protocolVersionId: "" }, false))
      .toEqual([ "item 1: a duração é em dias (número inteiro)" ]);
  });

  it("enfermagem: protocolo obrigatório, sem texto livre e até a dose máxima", () => {
    const fromProtocol = filled({ medication: { ...DIPYRONE_REF, max_dose: { quantity: 2, unit: "comprimido" } }, quantity: "3" });
    expect(prescriptionProblems({ items: [ fromProtocol ], protocolVersionId: "" }, true)).toEqual([
      "escolha o protocolo de enfermagem", "item 1: acima da dose máxima do protocolo (até 2 comprimido)"
    ]);
    const free = { ...emptyItem(), freeText: "Dipirona gotas", quantity: "1", quantityUnit: "frasco", route: "oral" as const, dosage: "20 gotas" };
    expect(prescriptionProblems({ items: [ free ], protocolVersionId: "npv1" }, true)).toEqual([ "item 1: a enfermagem só prescreve itens do protocolo" ]);
    expect(prescriptionInput({ items: [ { ...fromProtocol, quantity: "1" } ], protocolVersionId: "npv1" }))
      .toEqual({ items: [ { catalog_item: { id: "ci4" }, quantity: 1, quantity_unit: "comprimido", route: "oral", dosage_instructions: "1 cp 2x/dia", continuous: false } ],
        nursing_protocol: { version_id: "npv1" } });
  });

  it("antimicrobiano é avisado; renovação e item do catálogo preenchem o rascunho", () => {
    expect(hasAntimicrobial({ items: [ itemFromMedication(AMOXICILLIN) ], protocolVersionId: "" })).toBe(true);
    expect(hasAntimicrobial({ items: [ itemFromMedication(searchItem()) ], protocolVersionId: "" })).toBe(false);
    expect(ANTIMICROBIAL_NOTE).toBe("antimicrobiano: a receita sai em papel, em 2 vias (1ª via — farmácia, 2ª via — paciente), válida por 10 dias");
    const picked = itemFromMedication(searchItem());
    expect([ picked.quantityUnit, picked.route, picked.continuous, picked.medication?.id ]).toEqual([ "comprimido", "", false, "ci1" ]);
    const renewed = itemFromPrescription(prescriptionItem());
    expect([ renewed.medication?.id, renewed.quantity, renewed.quantityUnit, renewed.route, renewed.dosage, renewed.durationDays, renewed.continuous ])
      .toEqual([ "ci1", "60", "comprimido", "oral", "1 comprimido após o café e 1 após o jantar", "30", true ]);
    const freeRenewed = itemFromPrescription(prescriptionItem({ catalog_item: undefined, free_text: "Soro", duration_days: undefined }));
    expect([ freeRenewed.medication, freeRenewed.freeText, freeRenewed.durationDays ]).toEqual([ null, "Soro", "" ]);
    expect(itemFromMedication(searchItem()).key).not.toBe(itemFromMedication(searchItem()).key);
  });
});

describe("cancelar e emitir outro", () => {
  it("o novo abre com o conteúdo do cancelado", () => {
    const sick = reissueDraft(sickNoteDoc({ content: { type: "leave", days: 3, start_on: "2026-10-09", cid10: { code: "J069", label: "IVAS" }, cid_authorized: true, note: null } }));
    expect(sick?.kind).toBe("sick_note");
    if (sick?.kind === "sick_note") expect([ sick.draft.days, sick.draft.cid?.code, sick.draft.cidAuthorized, sick.draft.note ]).toEqual([ "3", "J069", true, "" ]);
    const decl = reissueDraft(declarationDoc());
    if (decl?.kind === "attendance_declaration") expect([ decl.draft.mode, decl.draft.period ]).toEqual([ "period", "morning" ]);
    const rx = reissueDraft(prescriptionDoc());
    if (rx?.kind === "prescription") expect(rx.draft.items.map((i) => i.medication?.id)).toEqual([ "ci1" ]);
    const exam = reissueDraft(examDoc({ content: { exams: [], note: "jejum de 8 h" } }));
    if (exam?.kind === "exam_requisition") expect(exam.draft.note).toBe("jejum de 8 h");
    expect(reissueDraft(sickNoteDoc({ kind: "referral" as never }))).toBeNull();
  });
});

describe("medicamentos em uso", () => {
  it("de fora: do catálogo ou texto livre", () => {
    expect(externalProblem(EMPTY_EXTERNAL)).toBe("escolha o medicamento no catálogo ou escreva o nome");
    expect(externalChange({ ...EMPTY_EXTERNAL, medication: catalogItem(), dosage: " 1 cp/dia " }))
      .toEqual({ action: "add_external", catalog_item_id: "ci1", dosage_summary: "1 cp/dia", continuous: true });
    expect(externalChange({ ...EMPTY_EXTERNAL, freeText: " Losartana 50 mg ", continuous: false }))
      .toEqual({ action: "add_external", free_text: "Losartana 50 mg", continuous: false });
  });

  it("linha e ordem: em uso primeiro, depois por nome", () => {
    expect(medicationLine(medication())).toBe("Metformina 850 mg — 1 cp 2x/dia");
    expect(medicationLine(medication({ dosage_summary: null }))).toBe("Metformina 850 mg");
    const list = [ medication({ id: "b", label: "Sinvastatina", status: "suspended" }), medication({ id: "c", label: "Losartana" }), medication({ id: "a" }) ];
    expect(sortMedications(list).map((m) => m.id)).toEqual([ "c", "a", "b" ]);
  });
});

describe("protocolos de enfermagem", () => {
  it("vigentes na data; rótulo com número e ano", () => {
    const current = nursingProtocol();
    const expired = nursingProtocol({ id: "np2", current_version: { id: "npv2", valid_from: "2024-01-01", valid_until: "2025-12-31", items: [] } });
    const future = nursingProtocol({ id: "np3", current_version: { id: "npv3", valid_from: "2026-11-01", valid_until: null, items: [] } });
    const none = nursingProtocol({ id: "np4", current_version: null });
    expect(vigentProtocols([ current, expired, future, none ], TODAY19C).map((p) => p.id)).toEqual([ "np1" ]);
    expect(protocolLabel(current)).toBe("Protocolo de enfermagem na atenção primária (nº 007/2025, 2025)");
  });

  it("rascunho: cabeçalho, vigência, itens e dose máxima", () => {
    const d = emptyProtocol(TODAY19C);
    expect(d.year).toBe("2026");
    expect(protocolProblem(d, true)).toBe("informe o título do protocolo");
    const full = addProtocolItem({ ...d, title: "Protocolo", number: "007/2025", year: "2025" }, DIPYRONE_REF);
    expect(addProtocolItem(full, DIPYRONE_REF).items).toHaveLength(1);
    expect(protocolProblem({ ...full, year: "25" }, true)).toBe("informe o ano de publicação com 4 dígitos");
    expect(protocolProblem({ ...full, validUntil: "2026-01-01" }, true)).toBe("o fim da vigência precisa ser depois do início");
    expect(protocolProblem({ ...full, items: [] }, true)).toBe("inclua ao menos um medicamento no protocolo");
    expect(protocolProblem({ ...full, items: [ { item: DIPYRONE_REF, maxDose: "dois", maxUnit: "frasco" } ] }, true))
      .toBe("item 1: a dose máxima é um número inteiro (ou deixe em branco)");
    expect(protocolProblem({ ...full, title: "" }, false)).toBeNull();
    expect(protocolInput({ ...full, items: [ { item: DIPYRONE_REF, maxDose: "2", maxUnit: "frasco" } ] })).toEqual({
      title: "Protocolo", number: "007/2025", year: 2025, valid_from: TODAY19C, items: [ { catalog_item_id: "ci4", max_dose: { quantity: 2, unit: "frasco" } } ]
    });
    expect(protocolVersionInput({ ...full, validUntil: "2027-12-31" }))
      .toEqual({ valid_from: TODAY19C, valid_until: "2027-12-31", items: [ { catalog_item_id: "ci4" } ] });
    const next = protocolDraftFrom(nursingProtocol(), TODAY19C);
    expect([ next.title, next.validFrom, next.items[0].item.id, next.items[0].maxDose, next.items[0].maxUnit ])
      .toEqual([ "Protocolo de enfermagem na atenção primária", TODAY19C, "ci4", "2", "frasco" ]);
    expect(protocolProblem({ ...full, items: [ { item: DIPYRONE_REF, maxDose: "2", maxUnit: " " } ] }, true))
      .toBe("item 1: informe a unidade da dose máxima");
  });
});

describe("CNPJ", () => {
  it("dígito verificador, máscara e ausente", () => {
    expect(isValidCnpj("11.222.333/0001-81")).toBe(true);
    expect(isValidCnpj("11222333000181")).toBe(true);
    expect(isValidCnpj("11222333000180")).toBe(false);
    expect(isValidCnpj("00000000000000")).toBe(false);
    expect(isValidCnpj("1122233300018")).toBe(false);
    expect(isValidCnpj("12.ABC.345/01DE-35")).toBe(true);
    expect(isValidCnpj("12abc34501de35")).toBe(true);
    expect(isValidCnpj("12ABC34501DE36")).toBe(false);
    expect(formatCnpj("11222333000181")).toBe("11.222.333/0001-81");
    expect(formatCnpj("12ABC34501DE35")).toBe("12.ABC.345/01DE-35");
    expect(formatCnpj(null)).toBe("não cadastrado");
  });
});

describe("frases das recusas", () => {
  const err = (status: number, body: unknown) => new ApiError(status, body, String(status));

  it("cada recusa do contrato tem frase; item e campo apontam onde", () => {
    expect(documentError(err(403, { error: "cbo_not_allowed" }))).toBe("o seu CBO não emite este tipo de documento");
    expect(documentError(err(403, { error: "not_author" }))).toBe("só a autora da consulta emite e cancela os documentos dela");
    expect(documentError(err(422, { error: "controlled_not_allowed", index: 1 })))
      .toBe("item 2: medicamento controlado não sai nesta receita (receita de controle especial — 19d)");
    expect(documentError(err(422, { error: "invalid_content", field: "days" }))).toBe("confira o campo: dias de afastamento");
    expect(documentError(err(422, { error: "invalid_content", field: "xyz" }))).toBe("confira o campo: xyz");
    expect(documentError(err(422, { error: "city_cnpj_missing" })))
      .toBe("a cidade ainda não cadastrou o CNPJ — sem ele a receita de enfermagem não sai; avise o administrador da cidade");
    expect(documentError(err(409, { error: "attendance_not_found_or_closed_long_ago" })))
      .toBe("atendimento não encontrado ou encerrado há mais de 30 dias — a declaração não sai mais por aqui");
    expect(documentError(err(409, { error: "already_cancelled" }))).toBe("este documento já estava cancelado — a lista foi atualizada");
    expect(documentError(err(503, { error: "catalog_unavailable" }))).toBe("o catálogo de medicamentos não respondeu — tente de novo ou use texto livre");
    expect(documentError(err(409, { error: "awaiting_signature" })))
      .toBe("o documento digital ainda não foi assinado — assine (sessão ou lote) ou volte ao papel em Pendentes de assinatura para imprimir");
  });

  it("interruptor desligado: documentos ou prontuário", () => {
    expect(documentError(err(403, { error: "feature_disabled", feature: "clinical_documents" }))).toBe(DOCUMENTS_DISABLED);
    expect(documentError(err(403, { error: "feature_disabled", feature: "clinical_record" }))).toBe("o prontuário está desligado nesta cidade");
  });

  it("recusa desconhecida não mostra o código cru", () => {
    expect(documentError(err(500, ""))).not.toMatch(/500|http/);
    expect(documentError(new Error("x"))).toBe("não foi possível concluir — tente de novo");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/lib/clinicalDocuments.test.ts`
Expected: FAIL — `./clinicalDocuments` não existe.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/clinicalDocuments.ts
// Documentos clínicos da consulta (módulo 19c; ADR 0033; contrato 2026-10-09
// §1–§7). Regras fora do React: rótulos, quem emite o quê (espelho do api),
// rascunho ↔ corpo do POST, o que falta antes de emitir (espelho dos 422 —
// quem garante é o api), cancelar e emitir outro, medicamentos em uso,
// protocolos de enfermagem, CNPJ e as frases das recusas. Nenhuma frase
// repete texto clínico: só rótulos, posições ("item 2") e nomes de campo.
import {
  ApiError, errorCode, type CatalogItemRef, type ClinicalDocument, type CodedOption, type CompanionReason, type DeclarationContent,
  type DeclarationInput, type DeclarationPeriod, type DocumentKind, type ExamRequisitionContent, type MaxDose, type MedicationChange,
  type MedicationRoute, type NursingProtocol, type NursingProtocolInput, type PatientMedication, type PrescriptionContent,
  type PrescriptionInput, type PrescriptionItem, type ProtocolVersionInput, type SickNoteContent, type SickNoteInput, type SickNoteType
} from "./api";
import { describeActionError } from "./actionErrors";
import { featureDisabledKey, hasFeature } from "./features";
import { fmtDateTime } from "./format";

export const DOCUMENTS_KEY = "consultationDocuments";
export const MEDICATIONS_KEY = "patientMedications";
export const PROTOCOLS_KEY = "nursingProtocols";
export const REMUME_KEY = "remume";
export const CITY_DOCS_PROFILE_KEY = "cityDocumentsProfile";

export const CANCEL_REASON_MIN = 10;
export const PENDING_REFETCH_MS = 15_000;

export const DOCUMENTS_DISABLED = "os documentos clínicos estão desligados nesta cidade";
export const CANCEL_WARNING = "cancelar não desfaz medicamento já entregue ao paciente";
export const ANTIMICROBIAL_NOTE =
  "antimicrobiano: a receita sai em papel, em 2 vias (1ª via — farmácia, 2ª via — paciente), válida por 10 dias";
export const CONTROLLED_BLOCKED = "receita de controle especial — 19d";
export const NO_PROTOCOL = "a cidade não tem protocolo de enfermagem vigente — sem ele a enfermagem não prescreve";
const GENERIC = "não foi possível concluir — tente de novo";

// ─── Rótulos ──────────────────────────────────────────────────────────────────

const KIND_LABEL: Record<string, string> = {
  sick_note: "Atestado", attendance_declaration: "Declaração de comparecimento", prescription: "Receita",
  exam_requisition: "Requisição de exames"
};
export function kindLabel(kind: string): string {
  return KIND_LABEL[kind] ?? kind;
}

const MODE_LABEL: Record<string, string> = { digital: "digital", paper: "papel" };
export function issueModeLabel(mode: string): string {
  return MODE_LABEL[mode] ?? mode;
}

export function documentSituation(doc: Pick<ClinicalDocument, "status" | "cancelled_at">): string {
  if (doc.status === "issued") return "emitido";
  if (doc.status === "cancelled") return `cancelado em ${fmtDateTime(doc.cancelled_at)}`;
  return doc.status;
}

export function issuedNotice(doc: ClinicalDocument): string {
  const what = `Documento emitido (${kindLabel(doc.kind)})`;
  if (doc.issue_mode === "digital") return `${what} em modo digital: a assinatura entra na sua fila; o PDF assinado sai depois dela.`;
  const twoCopies = doc.kind === "prescription" && (doc.content as PrescriptionContent).copies === 2;
  return `${what} em modo papel: imprima${twoCopies ? " as 2 vias" : ""} e assine à mão.`;
}

export const ROUTE_LABEL: Record<MedicationRoute, string> = {
  oral: "oral", sublingual: "sublingual", topical: "tópica", ophthalmic: "oftálmica", otic: "otológica", nasal: "nasal",
  inhalation: "inalatória", vaginal: "vaginal", rectal: "retal", intramuscular: "intramuscular", intravenous: "intravenosa",
  subcutaneous: "subcutânea", other: "outra"
};
export const ROUTES = Object.keys(ROUTE_LABEL) as MedicationRoute[];

export const COMPANION_REASON_LABEL: Record<CompanionReason, string> = {
  clt_473_x: "acompanhar gestante em consulta ou exame de pré-natal (CLT art. 473, X)",
  clt_473_xi: "acompanhar filho de até 6 anos em consulta (CLT art. 473, XI)",
  clt_473_xii: "exame preventivo de câncer (CLT art. 473, XII)",
  other: "outro motivo"
};
export const PERIOD_LABEL: Record<DeclarationPeriod, string> = { morning: "manhã", afternoon: "tarde", full_day: "dia todo" };
export const MEDICATION_STATUS_LABEL: Record<string, string> = { active: "em uso", suspended: "suspenso" };
export const MEDICATION_ORIGIN_LABEL: Record<string, string> = { prescription: "receita", external: "de fora" };

// ─── Quem emite o quê (spec §4; o api confere) ────────────────────────────────

export type ProfessionalKind = "physician" | "dentist" | "nurse" | "other";

export function professionalKind(cbo: string | null | undefined): ProfessionalKind {
  const code = cbo ?? "";
  if (/^225[123]/.test(code)) return "physician";
  if (code.startsWith("2232")) return "dentist";
  if (code.startsWith("2235")) return "nurse";
  return "other";
}

export function kindsFor(cbo: string | null | undefined): DocumentKind[] {
  const who = professionalKind(cbo);
  const kinds: DocumentKind[] = [];
  if (who === "physician" || who === "dentist") kinds.push("sick_note");
  kinds.push("attendance_declaration");
  if (who !== "other") kinds.push("prescription");
  kinds.push("exam_requisition");
  return kinds;
}

export function canUseDocuments(user: { operator?: boolean; features?: unknown } | null | undefined): boolean {
  return !!user && !user.operator && hasFeature(user, "clinical_documents");
}

export function canCancel(doc: ClinicalDocument, userId: string | null | undefined): boolean {
  return !!userId && doc.status === "issued" && doc.author.id === userId;
}

export function documentsRefetchInterval(docs: ClinicalDocument[] | undefined): number | false {
  return (docs ?? []).some((d) => d.status === "issued" && d.signature?.mode === "pending") ? PENDING_REFETCH_MS : false;
}

export function withDocument(list: ClinicalDocument[], doc: ClinicalDocument): ClinicalDocument[] {
  return list.some((d) => d.id === doc.id) ? list.map((d) => (d.id === doc.id ? doc : d)) : [ doc, ...list ];
}

export function cancelReasonProblem(reason: string): string | null {
  return reason.trim().length < CANCEL_REASON_MIN ? `descreva o motivo com pelo menos ${CANCEL_REASON_MIN} caracteres` : null;
}

const positiveInt = (text: string): number | null => {
  const t = text.trim();
  return /^\d+$/.test(t) && Number(t) >= 1 ? Number(t) : null;
};

// ─── Atestado ─────────────────────────────────────────────────────────────────

export interface SickNoteDraft {
  type: SickNoteType; days: string; startOn: string; companionName: string; companionKinship: string;
  companionReason: CompanionReason | ""; cid: CodedOption | null; cidAuthorized: boolean; note: string;
}

export function emptySickNote(today: string): SickNoteDraft {
  return { type: "leave", days: "", startOn: today, companionName: "", companionKinship: "", companionReason: "", cid: null,
    cidAuthorized: false, note: "" };
}

export function sickNoteProblem(d: SickNoteDraft): string | null {
  if (d.type === "leave") {
    if (positiveInt(d.days) === null) return "informe os dias de afastamento (número inteiro, 1 ou mais)";
    if (!d.startOn) return "informe o início do afastamento";
    if (d.cid && !d.cidAuthorized) return "o CID só entra com a autorização do paciente — marque a autorização ou tire o CID";
    return null;
  }
  if (!d.startOn) return "informe o dia do acompanhamento";
  if (!d.companionName.trim()) return "informe o nome do acompanhante";
  if (!d.companionKinship.trim()) return "informe o parentesco";
  if (!d.companionReason) return "escolha o motivo do acompanhamento";
  return null;
}

export function sickNoteInput(d: SickNoteDraft): SickNoteInput {
  const note = d.note.trim();
  const tail = note ? { note } : {};
  if (d.type === "leave") {
    return { type: "leave", days: positiveInt(d.days) ?? 0, start_on: d.startOn,
      ...(d.cid ? { cid10: { code: d.cid.code } } : {}), cid_authorized: !!d.cid && d.cidAuthorized, ...tail };
  }
  return { type: "companion", start_on: d.startOn, companion_name: d.companionName.trim(), companion_kinship: d.companionKinship.trim(),
    companion_reason: d.companionReason as CompanionReason, cid_authorized: false, ...tail };
}

// ─── Declaração ───────────────────────────────────────────────────────────────

export interface DeclarationDraft {
  date: string; mode: "period" | "times"; period: DeclarationPeriod | ""; arrivedAt: string; leftAt: string;
  companionName: string; issuerRegistration: string;
}

const HOUR_MINUTE = /^([01]\d|2[0-3]):[0-5]\d$/;

export function emptyDeclaration(today: string, arrivedAt = ""): DeclarationDraft {
  return { date: today, mode: arrivedAt ? "times" : "period", period: "", arrivedAt, leftAt: "", companionName: "", issuerRegistration: "" };
}

export function declarationProblem(d: DeclarationDraft): string | null {
  if (d.mode === "period") return d.period ? null : "escolha o período";
  if (!HOUR_MINUTE.test(d.arrivedAt)) return "informe a hora de chegada (HH:MM)";
  if (d.leftAt && !HOUR_MINUTE.test(d.leftAt)) return "a hora de saída precisa ser HH:MM";
  if (d.leftAt && d.leftAt <= d.arrivedAt) return "a saída precisa ser depois da chegada";
  return null;
}

export function declarationInput(d: DeclarationDraft): DeclarationInput {
  const companion = d.companionName.trim();
  const registration = d.issuerRegistration.trim();
  const tail = { ...(companion ? { companion_name: companion } : {}), ...(registration ? { issuer_registration: registration } : {}) };
  if (d.mode === "period") return { period: d.period as DeclarationPeriod, ...tail };
  return { arrived_at: d.arrivedAt, ...(d.leftAt ? { left_at: d.leftAt } : {}), ...tail };
}

// ─── Receita ──────────────────────────────────────────────────────────────────

export type ItemMedication = CatalogItemRef & {
  antimicrobial?: boolean; controlled?: boolean; in_network?: boolean; max_dose?: MaxDose | null;
};
export interface ItemDraft {
  key: string; medication: ItemMedication | null; freeText: string; quantity: string; quantityUnit: string;
  route: MedicationRoute | ""; dosage: string; durationDays: string; continuous: boolean;
}
export interface PrescriptionDraft { items: ItemDraft[]; protocolVersionId: string }

let itemSeq = 0;
const nextKey = () => { itemSeq += 1; return `item-${itemSeq}`; };

export function emptyItem(): ItemDraft {
  return { key: nextKey(), medication: null, freeText: "", quantity: "", quantityUnit: "", route: "", dosage: "", durationDays: "", continuous: false };
}

export function itemFromMedication(m: ItemMedication): ItemDraft {
  return { ...emptyItem(), medication: m, quantityUnit: m.dosage_form };
}

export function itemFromPrescription(item: PrescriptionItem): ItemDraft {
  return {
    key: nextKey(),
    medication: item.catalog_item ? { ...item.catalog_item, antimicrobial: item.antimicrobial } : null,
    freeText: item.free_text ?? "", quantity: String(item.quantity), quantityUnit: item.quantity_unit, route: item.route,
    dosage: item.dosage_instructions, durationDays: item.duration_days ? String(item.duration_days) : "", continuous: item.continuous
  };
}

export function prescriptionProblems(d: PrescriptionDraft, nurse: boolean): string[] {
  const out: string[] = [];
  if (nurse && !d.protocolVersionId) out.push("escolha o protocolo de enfermagem");
  if (d.items.length === 0) out.push("inclua ao menos um medicamento");
  d.items.forEach((it, i) => {
    const n = `item ${i + 1}`;
    const quantity = positiveInt(it.quantity);
    if (!it.medication && !it.freeText.trim()) out.push(`${n}: escolha um medicamento do catálogo ou escreva em texto livre`);
    if (it.medication?.controlled) out.push(`${n}: ${CONTROLLED_BLOCKED}`);
    if (nurse && !it.medication && it.freeText.trim()) out.push(`${n}: a enfermagem só prescreve itens do protocolo`);
    if (quantity === null) out.push(`${n}: informe a quantidade (número inteiro)`);
    const max = it.medication?.max_dose;
    if (nurse && quantity !== null && max && (quantity > max.quantity || it.quantityUnit.trim() !== max.unit)) {
      out.push(`${n}: acima da dose máxima do protocolo (até ${max.quantity} ${max.unit})`);
    }
    if (!it.quantityUnit.trim()) out.push(`${n}: informe a unidade da quantidade`);
    if (!it.route) out.push(`${n}: escolha a via`);
    if (!it.dosage.trim()) out.push(`${n}: escreva a posologia`);
    if (it.durationDays.trim() && positiveInt(it.durationDays) === null) out.push(`${n}: a duração é em dias (número inteiro)`);
  });
  return out;
}

export function prescriptionInput(d: PrescriptionDraft): PrescriptionInput {
  return {
    items: d.items.map((it) => {
      const duration = positiveInt(it.durationDays);
      return {
        ...(it.medication ? { catalog_item: { id: it.medication.id } } : { free_text: it.freeText.trim() }),
        quantity: positiveInt(it.quantity) ?? 0, quantity_unit: it.quantityUnit.trim(), route: it.route as MedicationRoute,
        dosage_instructions: it.dosage.trim(), ...(duration !== null ? { duration_days: duration } : {}), continuous: it.continuous
      };
    }),
    ...(d.protocolVersionId ? { nursing_protocol: { version_id: d.protocolVersionId } } : {})
  };
}

export function hasAntimicrobial(d: PrescriptionDraft): boolean {
  return d.items.some((it) => !!it.medication?.antimicrobial);
}

// ─── Cancelar e emitir outro ─────────────────────────────────────────────────

export interface ExamRequisitionDraft { note: string }
export type ReissueDraft =
  | { kind: "sick_note"; draft: SickNoteDraft }
  | { kind: "attendance_declaration"; draft: DeclarationDraft }
  | { kind: "prescription"; draft: PrescriptionDraft }
  | { kind: "exam_requisition"; draft: ExamRequisitionDraft };

export function reissueDraft(doc: ClinicalDocument): ReissueDraft | null {
  switch (doc.kind) {
    case "sick_note": {
      const c = doc.content as SickNoteContent;
      return { kind: "sick_note", draft: {
        type: c.type, days: c.days ? String(c.days) : "", startOn: c.start_on ?? "", companionName: c.companion_name ?? "",
        companionKinship: c.companion_kinship ?? "", companionReason: c.companion_reason ?? "",
        cid: c.cid10 ? { code: c.cid10.code, label: c.cid10.label } : null, cidAuthorized: c.cid_authorized, note: c.note ?? ""
      } };
    }
    case "attendance_declaration": {
      const c = doc.content as DeclarationContent;
      return { kind: "attendance_declaration", draft: {
        date: c.date, mode: c.period ? "period" : "times", period: c.period ?? "", arrivedAt: c.arrived_at ?? "",
        leftAt: c.left_at ?? "", companionName: c.companion_name ?? "", issuerRegistration: c.issuer_registration ?? ""
      } };
    }
    case "prescription": {
      const c = doc.content as PrescriptionContent;
      return { kind: "prescription", draft: { items: c.items.map(itemFromPrescription), protocolVersionId: c.nursing_protocol?.version_id ?? "" } };
    }
    case "exam_requisition":
      return { kind: "exam_requisition", draft: { note: (doc.content as ExamRequisitionContent).note ?? "" } };
    default:
      return null;
  }
}

// ─── Medicamentos em uso ─────────────────────────────────────────────────────

export interface ExternalDraft { medication: CatalogItemRef | null; freeText: string; dosage: string; continuous: boolean }
export const EMPTY_EXTERNAL: ExternalDraft = { medication: null, freeText: "", dosage: "", continuous: true };

export function externalProblem(d: ExternalDraft): string | null {
  return !d.medication && !d.freeText.trim() ? "escolha o medicamento no catálogo ou escreva o nome" : null;
}

export function externalChange(d: ExternalDraft): MedicationChange {
  const dosage = d.dosage.trim();
  return {
    action: "add_external",
    ...(d.medication ? { catalog_item_id: d.medication.id } : { free_text: d.freeText.trim() }),
    ...(dosage ? { dosage_summary: dosage } : {}),
    continuous: d.continuous
  };
}

export function medicationLine(m: PatientMedication): string {
  return m.dosage_summary ? `${m.label} — ${m.dosage_summary}` : m.label;
}

export function sortMedications(list: PatientMedication[]): PatientMedication[] {
  const rank = (m: PatientMedication) => (m.status === "active" ? 0 : 1);
  return [ ...list ].sort((a, b) => rank(a) - rank(b) || a.label.localeCompare(b.label, "pt-BR"));
}

// ─── Protocolos de enfermagem ─────────────────────────────────────────────────

export function vigentProtocols(list: NursingProtocol[], today: string): NursingProtocol[] {
  return list.filter((p) => {
    const v = p.current_version;
    return !!v && v.valid_from <= today && (!v.valid_until || v.valid_until >= today);
  });
}

export function protocolLabel(p: NursingProtocol): string {
  return `${p.title} (nº ${p.number}, ${p.year})`;
}

export interface ProtocolDraft {
  title: string; number: string; year: string; validFrom: string; validUntil: string; items: { item: CatalogItemRef; maxDose: string; maxUnit: string }[];
}

export function emptyProtocol(today: string): ProtocolDraft {
  return { title: "", number: "", year: today.slice(0, 4), validFrom: today, validUntil: "", items: [] };
}

export function protocolDraftFrom(p: NursingProtocol, today: string): ProtocolDraft {
  return {
    title: p.title, number: p.number, year: String(p.year), validFrom: today, validUntil: "",
    items: (p.current_version?.items ?? []).map((i) => ({
      item: i.catalog_item, maxDose: i.max_dose ? String(i.max_dose.quantity) : "", maxUnit: i.max_dose?.unit ?? i.catalog_item.dosage_form
    }))
  };
}

export function addProtocolItem(d: ProtocolDraft, item: CatalogItemRef): ProtocolDraft {
  if (d.items.some((i) => i.item.id === item.id)) return d;
  return { ...d, items: [ ...d.items, { item, maxDose: "", maxUnit: item.dosage_form } ] };
}

export function protocolProblem(d: ProtocolDraft, withHeader: boolean): string | null {
  if (withHeader) {
    if (!d.title.trim()) return "informe o título do protocolo";
    if (!d.number.trim()) return "informe o número do protocolo";
    if (!/^\d{4}$/.test(d.year.trim())) return "informe o ano de publicação com 4 dígitos";
  }
  if (!d.validFrom) return "informe o início da vigência";
  if (d.validUntil && d.validUntil < d.validFrom) return "o fim da vigência precisa ser depois do início";
  if (d.items.length === 0) return "inclua ao menos um medicamento no protocolo";
  const bad = d.items.findIndex((i) => i.maxDose.trim() !== "" && positiveInt(i.maxDose) === null);
  if (bad >= 0) return `item ${bad + 1}: a dose máxima é um número inteiro (ou deixe em branco)`;
  const noUnit = d.items.findIndex((i) => i.maxDose.trim() !== "" && !i.maxUnit.trim());
  if (noUnit >= 0) return `item ${noUnit + 1}: informe a unidade da dose máxima`;
  return null;
}

export function protocolVersionInput(d: ProtocolDraft): ProtocolVersionInput {
  return {
    valid_from: d.validFrom,
    ...(d.validUntil ? { valid_until: d.validUntil } : {}),
    items: d.items.map((i) => {
      const max = positiveInt(i.maxDose);
      return { catalog_item_id: i.item.id, ...(max !== null ? { max_dose: { quantity: max, unit: i.maxUnit.trim() } } : {}) };
    })
  };
}

export function protocolInput(d: ProtocolDraft): NursingProtocolInput {
  return { title: d.title.trim(), number: d.number.trim(), year: Number(d.year.trim()), ...protocolVersionInput(d) };
}

// ─── CNPJ da cidade ───────────────────────────────────────────────────────────

// CNPJ sem máscara, maiúsculo: 12 caracteres [0-9A-Z] + 2 dígitos (IN RFB 2.229/2024; o numérico é o caso particular).
export function cnpjDigits(text: string): string {
  return text.toUpperCase().replace(/[^0-9A-Z]/g, "");
}

export function isValidCnpj(text: string): boolean {
  const d = cnpjDigits(text);
  if (!/^[0-9A-Z]{12}\d{2}$/.test(d) || /^(.)\1{13}$/.test(d)) return false;
  const digit = (length: number) => {
    const weights = length === 12 ? [ 5, 4, 3, 2, 9, 8, 7, 6, 5, 4, 3, 2 ] : [ 6, 5, 4, 3, 2, 9, 8, 7, 6, 5, 4, 3, 2 ];
    // Valor do caractere = código ASCII − 48 (dígitos 0–9; A = 17 …), como a RFB.
    const rest = weights.reduce((sum, w, i) => sum + (d.charCodeAt(i) - 48) * w, 0) % 11;
    return rest < 2 ? 0 : 11 - rest;
  };
  return digit(12) === Number(d[12]) && digit(13) === Number(d[13]);
}

export function formatCnpj(digits: string | null | undefined): string {
  if (!digits) return "não cadastrado";
  const d = cnpjDigits(digits);
  return d.length === 14 ? `${d.slice(0, 2)}.${d.slice(2, 5)}.${d.slice(5, 8)}/${d.slice(8, 12)}-${d.slice(12)}` : d;
}

// ─── Frases das recusas ───────────────────────────────────────────────────────

const FIELD_LABEL: Record<string, string> = {
  days: "dias de afastamento", start_on: "início", companion_name: "nome do acompanhante", companion_kinship: "parentesco",
  companion_reason: "motivo do acompanhamento", date: "dia", period: "período", arrived_at: "hora de chegada",
  left_at: "hora de saída", items: "medicamentos", note: "observação"
};

const ERRORS: Record<string, string> = {
  not_author: "só a autora da consulta emite e cancela os documentos dela",
  cbo_not_allowed: "o seu CBO não emite este tipo de documento",
  cid_requires_authorization: "o CID só entra com a autorização do paciente",
  companion_cid_not_allowed: "atestado de acompanhante não leva CID",
  controlled_not_allowed: `medicamento controlado não sai nesta receita (${CONTROLLED_BLOCKED})`,
  not_in_nursing_protocol: "este medicamento não está no protocolo de enfermagem vigente",
  above_protocol_max_dose: "a quantidade passa da dose máxima do protocolo",
  free_text_not_allowed: "a enfermagem só prescreve itens do protocolo — sem texto livre",
  city_cnpj_missing: "a cidade ainda não cadastrou o CNPJ — sem ele a receita de enfermagem não sai; avise o administrador da cidade",
  no_exam_requests: "esta consulta não tem exames pedidos — inclua os exames na consulta (ou num adendo) antes",
  invalid_item: "um dos itens da receita não foi aceito",
  attendance_not_found_or_closed_long_ago: "atendimento não encontrado ou encerrado há mais de 30 dias — a declaração não sai mais por aqui",
  invalid_reason: `descreva o motivo com pelo menos ${CANCEL_REASON_MIN} caracteres`,
  already_cancelled: "este documento já estava cancelado — a lista foi atualizada",
  awaiting_signature: "o documento digital ainda não foi assinado — assine (sessão ou lote) ou volte ao papel em Pendentes de assinatura para imprimir",
  already_exists: "já existe um protocolo com este número e ano",
  already_active: "este medicamento já está em uso na lista",
  not_draft: "a consulta já foi finalizada — a lista de medicamentos muda só com a consulta em rascunho",
  catalog_unavailable: "o catálogo de medicamentos não respondeu — tente de novo ou use texto livre",
  invalid_cnpj: "CNPJ inválido — confira os 14 caracteres (números e, no CNPJ novo, letras)",
  out_of_context: "fora do atendimento, abra o prontuário com motivo para ver este documento",
  opening_required: "a abertura justificada terminou — abra o prontuário de novo",
  missing_role: "seu papel não permite esta ação"
};

export function documentError(err: unknown): string {
  const feature = featureDisabledKey(err);
  if (feature === "clinical_record") return "o prontuário está desligado nesta cidade";
  if (feature !== null) return DOCUMENTS_DISABLED;
  const code = errorCode(err);
  const body = err instanceof ApiError && err.body && typeof err.body === "object" ? (err.body as { field?: unknown; index?: unknown }) : null;
  if (code === "invalid_content") {
    const field = typeof body?.field === "string" ? body.field : "";
    return `confira o campo: ${FIELD_LABEL[field] ?? (field || "do formulário")}`;
  }
  if (code && ERRORS[code]) return `${typeof body?.index === "number" ? `item ${body.index + 1}: ` : ""}${ERRORS[code]}`;
  const described = describeActionError(err);
  return "message" in described ? described.message : GENERIC;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/lib/clinicalDocuments.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro. Se `documentError(err(500, ""))` vier com o código cru, confira o que `describeActionError` devolve para 500 (`src/lib/actionErrors.ts`): a frase genérica dele é a esperada.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19c
/opt/homebrew/bin/git add src/lib/clinicalDocuments.ts src/lib/clinicalDocuments.test.ts
/opt/homebrew/bin/git commit -m "feat: add screen rules for clinical documents, prescriptions and nursing protocols

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Busca no catálogo e medicamentos em uso ao lado dos problemas

**Files:**
- Create: `src/modules/documents/MedicationSearch.tsx`, `src/modules/consultation/MedicationsBlock.tsx`
- Modify: `src/modules/consultation/PatientPanel.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`
- Test: `src/modules/documents/MedicationSearch.test.tsx`, `src/modules/consultation/MedicationsBlock.test.tsx`, `src/modules/consultation/ConsultationWorkspace.documents.test.tsx`

**Interfaces:**
- Consumes: `searchMedications`, `listPatientMedications`, `changeMedication`, `errorCode` (Task 1); `CONTROLLED_BLOCKED`, `MEDICATIONS_KEY`, `MEDICATION_STATUS_LABEL`, `MEDICATION_ORIGIN_LABEL`, `EMPTY_EXTERNAL`, `externalProblem`, `externalChange`, `medicationLine`, `sortMedications`, `documentError`, `canUseDocuments` (Task 2); `useDebouncedValue`, `Tag`, `formStyles`.
- Produces:
  - `MedicationSearch({ label, unitId?, blockControlled, onPick(item: MedicationSearchItem), delayMs? })` — busca a partir de 2 caracteres, resultados em `<ul aria-label="resultados: <label>">`, um botão por item (nome acessível = `label` do item), selos "na rede", "antimicrobiano" e, no controlado, `CONTROLLED_BLOCKED` (botão desligado quando `blockControlled`) ou "controlado";
  - `MedicationsBlock({ patientId, draftConsultationId: string | null, unitId?, searchDelayMs? })` — "Medicamentos em uso", Suspender/Reativar e "Informar medicamento em uso de fora" só com `draftConsultationId`;
  - `PatientPanel` ganha a prop opcional `besideProblems?: ReactNode`, desenhada logo depois de "Problemas ativos";
  - `ConsultationWorkspace` passa `<MedicationsBlock …>` quando a sessão tem `clinical_documents`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/documents/MedicationSearch.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, searchMedications: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { MedicationSearch } from "./MedicationSearch";
import { AMOXICILLIN, CLONAZEPAM, searchItem } from "../../test/documentFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function renderIt(blockControlled: boolean, onPick = vi.fn()) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<MedicationSearch label="Buscar medicamento" unitId="u1" blockControlled={blockControlled} onPick={onPick} delayMs={0} />, { wrapper });
  return onPick;
}

describe("MedicationSearch", () => {
  beforeEach(() => {
    mocked(api.searchMedications).mockReset();
    mocked(api.searchMedications).mockResolvedValue([ searchItem(), AMOXICILLIN, CLONAZEPAM ]);
  });

  it("busca com a unidade, mostra 'na rede' e 'antimicrobiano', escolhe e limpa o campo", async () => {
    const onPick = renderIt(true);
    const field = screen.getByLabelText("Buscar medicamento") as HTMLInputElement;
    fireEvent.change(field, { target: { value: "metf" } });
    const list = await screen.findByRole("list", { name: "resultados: Buscar medicamento" });
    expect(api.searchMedications).toHaveBeenCalledWith("metf", "u1");
    expect(within(list).getAllByText("na rede")).toHaveLength(2);
    expect(within(list).getByText("antimicrobiano")).not.toBeNull();
    fireEvent.click(within(list).getByRole("button", { name: "Metformina, cloridrato 850 mg, comprimido" }));
    expect(onPick).toHaveBeenCalledWith(searchItem());
    expect(field.value).toBe("");
  });

  it("receita: controlado aparece bloqueado com o aviso do 19d", async () => {
    const onPick = renderIt(true);
    fireEvent.change(screen.getByLabelText("Buscar medicamento"), { target: { value: "clon" } });
    const list = await screen.findByRole("list", { name: "resultados: Buscar medicamento" });
    const blocked = within(list).getByRole("button", { name: "Clonazepam 2 mg, comprimido" }) as HTMLButtonElement;
    expect(blocked.disabled).toBe(true);
    expect(within(list).getByText("receita de controle especial — 19d")).not.toBeNull();
    fireEvent.click(blocked);
    expect(onPick).not.toHaveBeenCalled();
  });

  it("lista de uso: controlado entra (não é receita)", async () => {
    const onPick = renderIt(false);
    fireEvent.change(screen.getByLabelText("Buscar medicamento"), { target: { value: "clon" } });
    const list = await screen.findByRole("list", { name: "resultados: Buscar medicamento" });
    fireEvent.click(within(list).getByRole("button", { name: "Clonazepam 2 mg, comprimido" }));
    expect(onPick).toHaveBeenCalledWith(CLONAZEPAM);
    expect(within(list).queryByText("receita de controle especial — 19d")).toBeNull();
  });

  it("um caractere não busca; catálogo fora do ar diz para usar texto livre", async () => {
    renderIt(true);
    fireEvent.change(screen.getByLabelText("Buscar medicamento"), { target: { value: "m" } });
    expect(screen.getByText("digite pelo menos 2 caracteres")).not.toBeNull();
    expect(api.searchMedications).not.toHaveBeenCalled();
    mocked(api.searchMedications).mockRejectedValue(new ApiError(503, { error: "catalog_unavailable" }, "503"));
    fireEvent.change(screen.getByLabelText("Buscar medicamento"), { target: { value: "metf" } });
    expect((await screen.findByRole("alert")).textContent).toBe("o catálogo de medicamentos não respondeu — tente de novo ou use texto livre");
  });
});
```

```tsx
// src/modules/consultation/MedicationsBlock.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listPatientMedications: vi.fn(), changeMedication: vi.fn(), searchMedications: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { MedicationsBlock } from "./MedicationsBlock";
import { renderWithProviders } from "../../test/campaignFixtures";
import { CLONAZEPAM, docUser, medication } from "../../test/documentFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const LIST = [ medication(), medication({ id: "pm2", label: "Sinvastatina 20 mg", dosage_summary: null, status: "suspended", origin: "external", continuous: false }) ];

describe("MedicationsBlock", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listPatientMedications, api.changeMedication, api.searchMedications ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(docUser());
    mocked(api.listPatientMedications).mockResolvedValue(LIST);
    mocked(api.changeMedication).mockResolvedValue(medication());
    mocked(api.searchMedications).mockResolvedValue([ CLONAZEPAM ]);
  });

  it("lista em uso e suspensos, com origem e uso contínuo; sem consulta em rascunho não oferece mudar", async () => {
    renderWithProviders(<MedicationsBlock patientId="pa1" draftConsultationId={null} />);
    const list = await screen.findByRole("list", { name: "medicamentos em uso" });
    const rows = within(list).getAllByRole("listitem");
    expect(rows[0].textContent).toMatch(/Metformina 850 mg — 1 cp 2x\/dia/);
    expect(within(rows[0]).getByText("em uso")).not.toBeNull();
    expect(within(rows[0]).getByText("receita")).not.toBeNull();
    expect(within(rows[0]).getByText("uso contínuo")).not.toBeNull();
    expect(within(rows[1]).getByText("suspenso")).not.toBeNull();
    expect(within(rows[1]).getByText("de fora")).not.toBeNull();
    expect(api.listPatientMedications).toHaveBeenCalledWith("pa1");
    expect(screen.queryByRole("button", { name: /Suspender|Reativar/ })).toBeNull();
    expect(screen.queryByRole("button", { name: "Informar medicamento em uso de fora" })).toBeNull();
  });

  it("suspende e reativa pela consulta em rascunho, e relê a lista", async () => {
    renderWithProviders(<MedicationsBlock patientId="pa1" draftConsultationId="cs1" />);
    fireEvent.click(await screen.findByRole("button", { name: "Suspender Metformina 850 mg" }));
    await waitFor(() => expect(api.changeMedication).toHaveBeenCalledWith("cs1", { action: "suspend", medication_id: "pm1" }));
    fireEvent.click(screen.getByRole("button", { name: "Reativar Sinvastatina 20 mg" }));
    await waitFor(() => expect(api.changeMedication).toHaveBeenCalledWith("cs1", { action: "reactivate", medication_id: "pm2" }));
    await waitFor(() => expect(mocked(api.listPatientMedications).mock.calls.length).toBeGreaterThanOrEqual(3));
  });

  it("informa um medicamento de fora pelo catálogo (controlado entra: não é receita)", async () => {
    renderWithProviders(<MedicationsBlock patientId="pa1" draftConsultationId="cs1" unitId="u1" searchDelayMs={0} />);
    fireEvent.click(await screen.findByRole("button", { name: "Informar medicamento em uso de fora" }));
    fireEvent.change(screen.getByLabelText("Buscar no catálogo"), { target: { value: "clon" } });
    fireEvent.click(await screen.findByRole("button", { name: "Clonazepam 2 mg, comprimido" }));
    fireEvent.change(screen.getByLabelText("Como usa (resumo)"), { target: { value: "1 cp à noite" } });
    fireEvent.click(screen.getByRole("button", { name: "Incluir na lista" }));
    await waitFor(() => expect(api.changeMedication).toHaveBeenCalledWith("cs1",
      { action: "add_external", catalog_item_id: "ci3", dosage_summary: "1 cp à noite", continuous: true }));
    await waitFor(() => expect(screen.queryByRole("region", { name: "Medicamento em uso de fora" })).toBeNull());
  });

  it("texto livre; sem nada escolhido diz o que falta", async () => {
    renderWithProviders(<MedicationsBlock patientId="pa1" draftConsultationId="cs1" searchDelayMs={0} />);
    fireEvent.click(await screen.findByRole("button", { name: "Informar medicamento em uso de fora" }));
    fireEvent.click(screen.getByRole("button", { name: "Incluir na lista" }));
    expect((await screen.findByRole("alert")).textContent).toBe("escolha o medicamento no catálogo ou escreva o nome");
    expect(api.changeMedication).not.toHaveBeenCalled();
    fireEvent.change(screen.getByLabelText("Ou escreva o nome (texto livre)"), { target: { value: "Losartana 50 mg" } });
    fireEvent.click(screen.getByLabelText("Uso contínuo"));
    fireEvent.click(screen.getByRole("button", { name: "Incluir na lista" }));
    await waitFor(() => expect(api.changeMedication).toHaveBeenCalledWith("cs1",
      { action: "add_external", free_text: "Losartana 50 mg", continuous: false }));
  });

  it("409 already_active: diz e relê a lista", async () => {
    mocked(api.changeMedication).mockRejectedValue(new ApiError(409, { error: "already_active" }, "409"));
    renderWithProviders(<MedicationsBlock patientId="pa1" draftConsultationId="cs1" searchDelayMs={0} />);
    fireEvent.click(await screen.findByRole("button", { name: "Informar medicamento em uso de fora" }));
    fireEvent.change(screen.getByLabelText("Ou escreva o nome (texto livre)"), { target: { value: "Metformina" } });
    fireEvent.click(screen.getByRole("button", { name: "Incluir na lista" }));
    expect((await screen.findByRole("alert")).textContent).toBe("este medicamento já está em uso na lista");
    await waitFor(() => expect(mocked(api.listPatientMedications).mock.calls.length).toBeGreaterThanOrEqual(2));
  });
});
```

```tsx
// src/modules/consultation/ConsultationWorkspace.documents.test.tsx
// Módulo 19c: o encaixe dos documentos na consulta (medicamentos ao lado dos
// problemas e, na Task 6, a aba Documentos). Os componentes novos são dublês:
// o comportamento deles tem teste próprio.
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen } from "@testing-library/react";
import type { Consultation } from "../../lib/api";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getAttendanceRecord: vi.fn(), getConsultationOptions: vi.fn(),
    startConsultation: vi.fn(), getConsultation: vi.fn() };
});
vi.mock("./ConsultationEditor", () => ({
  ConsultationEditor: (p: { consultation: Consultation; onFinalized(c: Consultation): void }) => (
    <div>
      <span>{`editor ${p.consultation.id}`}</span>
      <button type="button" onClick={() => p.onFinalized({ ...p.consultation, status: "finalized" })}>finalizar (dublê)</button>
    </div>
  )
}));
vi.mock("./ConsultationView", () => ({
  ConsultationView: (p: { consultation: Consultation }) => <span>{`consulta finalizada ${p.consultation.id}`}</span>,
  ConsultationLoader: (p: { id: string }) => <span>{`consulta anterior ${p.id}`}</span>
}));
vi.mock("./MedicationsBlock", () => ({
  MedicationsBlock: (p: { patientId: string; draftConsultationId: string | null; unitId?: string }) => (
    <span>{`medicamentos ${p.patientId} · ${p.draftConsultationId ? `rascunho ${p.draftConsultationId}` : "sem consulta"} · ${p.unitId}`}</span>
  )
}));

import * as api from "../../lib/api";
import { ConsultationWorkspace } from "./ConsultationWorkspace";
import { renderWithProviders } from "../../test/campaignFixtures";
import { consultation, options, record } from "../../test/consultationFixtures";
import { docUser } from "../../test/documentFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };

function renderIt() {
  renderWithProviders(<ConsultationWorkspace attendanceId="a1" unit={unit} units={[ unit ]} onClose={vi.fn()} onFinalized={vi.fn()} />);
}

describe("ConsultationWorkspace — documentos clínicos (19c)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.getAttendanceRecord, api.getConsultationOptions, api.startConsultation, api.getConsultation ]) {
      mocked(fn).mockReset();
    }
    mocked(api.fetchCurrentSession).mockResolvedValue(docUser());
    mocked(api.getAttendanceRecord).mockResolvedValue(record());
    mocked(api.getConsultationOptions).mockResolvedValue(options());
    mocked(api.startConsultation).mockResolvedValue(consultation());
  });

  it("os medicamentos aparecem ao lado dos problemas; mudar só com a consulta em rascunho", async () => {
    renderIt();
    expect(await screen.findByText("medicamentos pa1 · sem consulta · u1")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Iniciar consulta" }));
    expect(await screen.findByText("medicamentos pa1 · rascunho cs1 · u1")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "finalizar (dublê)" }));
    expect(await screen.findByText("medicamentos pa1 · sem consulta · u1")).not.toBeNull();
  });

  it("sem clinical_documents na sessão, nada novo aparece", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(docUser([ "health_professional" ], { features: [ "clinical_record" ] }));
    renderIt();
    expect(await screen.findByText("Joana Lima")).not.toBeNull();
    expect(screen.queryByText(/^medicamentos /)).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/documents/MedicationSearch.test.tsx src/modules/consultation/MedicationsBlock.test.tsx src/modules/consultation/ConsultationWorkspace.documents.test.tsx`
Expected: FAIL — os dois componentes não existem e a consulta não desenha o bloco.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/documents/MedicationSearch.tsx
// Busca no catálogo de medicamentos (módulo 19c; contrato §5): o termo vai no
// corpo de um POST e só a partir de 2 caracteres; a REMUME da unidade vem
// primeiro ("na rede"). Na receita, o controlado aparece (a médica precisa
// saber que existe) mas não entra: receita de controle especial é do 19d.
// Na lista de uso e na REMUME, entra.
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { searchMedications, type MedicationSearchItem } from "../../lib/api";
import { CONTROLLED_BLOCKED, documentError } from "../../lib/clinicalDocuments";
import { useDebouncedValue } from "../../lib/useDebouncedValue";
import { Tag } from "../../components/Tag";
import { disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";

export const MEDICATION_SEARCH_MIN_CHARS = 2;

interface Props {
  label: string;
  unitId?: string;
  blockControlled: boolean;
  onPick(item: MedicationSearchItem): void;
  delayMs?: number;
}

export function MedicationSearch({ label, unitId, blockControlled, onPick, delayMs = 300 }: Props) {
  const [ text, setText ] = useState("");
  const typed = text.trim();
  const term = useDebouncedValue(typed, delayMs);
  const enabled = term.length >= MEDICATION_SEARCH_MIN_CHARS;
  // gcTime: 0 — o termo pode dizer o que a pessoa toma; sai da memória com a tela.
  const query = useQuery({
    queryKey: [ "medicationSearch", unitId ?? null, term ], queryFn: () => searchMedications(term, unitId), enabled, gcTime: 0, retry: false
  });
  const show = enabled && typed.length >= MEDICATION_SEARCH_MIN_CHARS;

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
      <label style={labelStyle}>
        {label}
        <input value={text} style={inputStyle} onChange={(e) => setText(e.target.value)} />
      </label>
      {typed.length > 0 && typed.length < MEDICATION_SEARCH_MIN_CHARS && (
        <small style={hint}>digite pelo menos {MEDICATION_SEARCH_MIN_CHARS} caracteres</small>
      )}
      {show && query.isPending && <small className="mono" style={hint}>buscando…</small>}
      {show && query.isError && <p role="alert" style={alert}>{documentError(query.error)}</p>}
      {show && query.isSuccess && query.data.length === 0 && <small style={hint}>nenhum medicamento encontrado</small>}
      {show && query.isSuccess && query.data.length > 0 && (
        <ul aria-label={`resultados: ${label}`} style={list}>
          {query.data.map((item) => {
            const blocked = blockControlled && item.controlled;
            return (
              <li key={item.id} style={row}>
                <button type="button" disabled={blocked} aria-label={item.label}
                  style={{ ...(blocked ? disabledButtonStyle : secondaryButtonStyle), flex: 1, textAlign: "left" }}
                  onClick={() => { onPick(item); setText(""); }}>
                  {item.label}
                </button>
                {item.in_network && <Tag tone="ok" mono={false}>na rede</Tag>}
                {item.antimicrobial && <Tag tone="warn" mono={false}>antimicrobiano</Tag>}
                {item.controlled && <Tag tone="down" mono={false}>{blockControlled ? CONTROLLED_BLOCKED : "controlado"}</Tag>}
              </li>
            );
          })}
        </ul>
      )}
    </div>
  );
}

const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const list: CSSProperties = { listStyle: "none", margin: 0, padding: 0, display: "flex", flexDirection: "column", gap: 4 };
const row: CSSProperties = { display: "flex", gap: 6, alignItems: "center" };
```

```tsx
// src/modules/consultation/MedicationsBlock.tsx
// Medicamentos em uso do paciente (módulo 19c; spec §5; contrato §4), ao lado
// dos problemas: estado atual reconstruído de eventos no api. Suspender,
// reativar e "informar medicamento em uso de fora" só pela consulta em
// rascunho da autora (o api confere); a receita de uso contínuo inclui sozinha.
// Leitura com gcTime 0 (prontuário).
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { changeMedication, listPatientMedications } from "../../lib/api";
import {
  EMPTY_EXTERNAL, MEDICATIONS_KEY, MEDICATION_ORIGIN_LABEL, MEDICATION_STATUS_LABEL, documentError, externalChange, externalProblem,
  medicationLine, sortMedications, type ExternalDraft
} from "../../lib/clinicalDocuments";
import { Tag } from "../../components/Tag";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { MedicationSearch } from "../documents/MedicationSearch";

interface Props {
  patientId: string;
  // A consulta em rascunho da autora; null = só leitura.
  draftConsultationId: string | null;
  unitId?: string;
  searchDelayMs?: number;
}

export function MedicationsBlock({ patientId, draftConsultationId, unitId, searchDelayMs }: Props) {
  const queryClient = useQueryClient();
  const query = useQuery({
    queryKey: [ MEDICATIONS_KEY, patientId ], queryFn: () => listPatientMedications(patientId), gcTime: 0, retry: false
  });
  const [ busy, setBusy ] = useState<string | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const [ adding, setAdding ] = useState(false);
  const reload = () => void queryClient.invalidateQueries({ queryKey: [ MEDICATIONS_KEY, patientId ] });

  async function change(id: string, action: "suspend" | "reactivate") {
    if (!draftConsultationId || busy) return;
    setBusy(id); setError(null);
    try {
      await changeMedication(draftConsultationId, { action, medication_id: id });
    } catch (err) {
      setError(documentError(err));
    } finally {
      setBusy(null);
      reload();
    }
  }

  const items = sortMedications(query.data ?? []);

  return (
    <div style={block}>
      <strong style={sub}>Medicamentos em uso</strong>
      {query.isPending && <p className="mono" style={muted}>carregando…</p>}
      {query.isError && <p role="alert" style={alert}>{documentError(query.error)}</p>}
      {error && <p role="alert" style={alert}>{error}</p>}
      {query.isSuccess && (items.length === 0 ? <p style={muted}>nenhum medicamento registrado</p> : (
        <ul aria-label="medicamentos em uso" style={list}>
          {items.map((m) => (
            <li key={m.id} style={row}>
              <span style={{ flex: 1 }}>{medicationLine(m)}</span>
              <Tag tone={m.status === "active" ? "ok" : "neutral"} mono={false}>{MEDICATION_STATUS_LABEL[m.status] ?? m.status}</Tag>
              <Tag mono={false}>{MEDICATION_ORIGIN_LABEL[m.origin] ?? m.origin}</Tag>
              {m.continuous && <Tag mono={false}>uso contínuo</Tag>}
              {draftConsultationId && m.status === "active" && (
                <button type="button" aria-label={`Suspender ${m.label}`} disabled={busy === m.id}
                  style={busy === m.id ? disabledButtonStyle : secondaryButtonStyle} onClick={() => void change(m.id, "suspend")}>Suspender</button>
              )}
              {draftConsultationId && m.status === "suspended" && (
                <button type="button" aria-label={`Reativar ${m.label}`} disabled={busy === m.id}
                  style={busy === m.id ? disabledButtonStyle : secondaryButtonStyle} onClick={() => void change(m.id, "reactivate")}>Reativar</button>
              )}
            </li>
          ))}
        </ul>
      ))}
      {draftConsultationId && !adding && (
        <div>
          <button type="button" style={secondaryButtonStyle} onClick={() => setAdding(true)}>Informar medicamento em uso de fora</button>
        </div>
      )}
      {draftConsultationId && adding && (
        <ExternalForm consultationId={draftConsultationId} unitId={unitId} searchDelayMs={searchDelayMs}
          onDone={() => { setAdding(false); reload(); }} onRefresh={reload} onCancel={() => setAdding(false)} />
      )}
    </div>
  );
}

function ExternalForm({ consultationId, unitId, searchDelayMs, onDone, onRefresh, onCancel }: {
  consultationId: string; unitId?: string; searchDelayMs?: number; onDone(): void; onRefresh(): void; onCancel(): void;
}) {
  const [ draft, setDraft ] = useState<ExternalDraft>(EMPTY_EXTERNAL);
  const [ busy, setBusy ] = useState(false);
  const [ problem, setProblem ] = useState<string | null>(null);

  async function save() {
    if (busy) return;
    const found = externalProblem(draft);
    if (found) { setProblem(found); return; }
    setBusy(true); setProblem(null);
    try {
      await changeMedication(consultationId, externalChange(draft));
      onDone();
    } catch (err) {
      setProblem(documentError(err));
      onRefresh();
    } finally {
      setBusy(false);
    }
  }

  return (
    <section aria-label="Medicamento em uso de fora" style={panel}>
      {draft.medication ? (
        <p style={text}>
          {draft.medication.label}{" "}
          <button type="button" style={linkButton} onClick={() => setDraft((d) => ({ ...d, medication: null }))}>trocar</button>
        </p>
      ) : (
        <>
          <MedicationSearch label="Buscar no catálogo" unitId={unitId} blockControlled={false} delayMs={searchDelayMs}
            onPick={(m) => setDraft((d) => ({ ...d, medication: m, freeText: "" }))} />
          <label style={labelStyle}>
            Ou escreva o nome (texto livre)
            <input value={draft.freeText} style={inputStyle} onChange={(e) => setDraft((d) => ({ ...d, freeText: e.target.value }))} />
          </label>
        </>
      )}
      <label style={labelStyle}>
        Como usa (resumo)
        <input value={draft.dosage} style={inputStyle} onChange={(e) => setDraft((d) => ({ ...d, dosage: e.target.value }))} />
      </label>
      <label style={checkLabel}>
        <input type="checkbox" checked={draft.continuous} onChange={(e) => setDraft((d) => ({ ...d, continuous: e.target.checked }))} />
        Uso contínuo
      </label>
      {problem && <p role="alert" style={alert}>{problem}</p>}
      <div style={row}>
        <button type="button" disabled={busy} style={busy ? disabledButtonStyle : buttonStyle} onClick={() => void save()}>Incluir na lista</button>
        <button type="button" disabled={busy} style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </section>
  );
}

const block: CSSProperties = { display: "flex", flexDirection: "column", gap: 6 };
const sub: CSSProperties = { fontSize: 12.5 };
const list: CSSProperties = { margin: 0, paddingLeft: 0, listStyle: "none", fontSize: 12.5, display: "flex", flexDirection: "column", gap: 4 };
const row: CSSProperties = { display: "flex", gap: 6, alignItems: "center", flexWrap: "wrap" };
const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, padding: 12, border: "1px solid var(--rule)", borderRadius: 8 };
const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const checkLabel: CSSProperties = { display: "flex", gap: 6, alignItems: "center", fontSize: 12, color: "var(--ink2)" };
const text: CSSProperties = { margin: 0, fontSize: 12.5 };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const linkButton: CSSProperties = {
  border: "none", background: "transparent", padding: 0, cursor: "pointer", fontSize: 12, color: "var(--accent)", textDecoration: "underline"
};
```

Em `src/modules/consultation/PatientPanel.tsx`:

```tsx
import type { CSSProperties, ReactNode } from "react";
```

```tsx
// Módulo 19c: `besideProblems` = os medicamentos em uso, logo depois dos problemas.
export function PatientPanel({ record, onOpenConsultation, besideProblems }: {
  record: ClinicalRecord; onOpenConsultation(id: string): void; besideProblems?: ReactNode;
}) {
```

e, logo depois do `</div>` do bloco "Problemas ativos" (o que fecha `{active.length === 0 ? … : (<ul aria-label="problemas ativos" …>)}`):

```tsx
      {besideProblems}
```

Em `src/modules/consultation/ConsultationWorkspace.tsx`:

```tsx
import { canUseDocuments } from "../../lib/clinicalDocuments";
import { MedicationsBlock } from "./MedicationsBlock";
```

depois de `const signer = canSign(user);`:

```tsx
  // Módulo 19c: documentos clínicos e medicamentos em uso, com o interruptor ligado.
  const documentsOn = canUseDocuments(user);
```

e troque `<PatientPanel record={record.data} onOpenConsultation={setViewing} />` por:

```tsx
          <PatientPanel record={record.data} onOpenConsultation={setViewing} besideProblems={documentsOn ? (
            <MedicationsBlock patientId={record.data.patient.id} unitId={props.unit.id} searchDelayMs={props.searchDelayMs}
              draftConsultationId={consultation?.status === "draft" ? consultation.id : null} />
          ) : undefined} />
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/documents/MedicationSearch.test.tsx src/modules/consultation/ && npx tsc --noEmit`
Expected: PASS (os testes antigos da consulta e do painel do paciente continuam verdes: a prop nova é opcional) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19c
/opt/homebrew/bin/git add src/modules/documents/MedicationSearch.tsx src/modules/documents/MedicationSearch.test.tsx \
  src/modules/consultation/MedicationsBlock.tsx src/modules/consultation/MedicationsBlock.test.tsx \
  src/modules/consultation/PatientPanel.tsx src/modules/consultation/ConsultationWorkspace.tsx \
  src/modules/consultation/ConsultationWorkspace.documents.test.tsx
/opt/homebrew/bin/git commit -m "feat: show the patient's medications in use next to the problems in the consultation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Formulários do atestado, da declaração e da requisição de exames

**Files:**
- Create: `src/modules/documents/SickNoteForm.tsx`, `src/modules/documents/DeclarationForm.tsx`, `src/modules/documents/ExamRequisitionForm.tsx`, `src/modules/documents/formStyles.ts`
- Test: `src/modules/documents/SickNoteForm.test.tsx`, `src/modules/documents/DeclarationForm.test.tsx`, `src/modules/documents/ExamRequisitionForm.test.tsx`

**Interfaces:**
- Consumes: `SickNoteDraft`, `emptySickNote`, `sickNoteProblem`, `sickNoteInput`, `DeclarationDraft`, `declarationProblem`, `declarationInput`, `COMPANION_REASON_LABEL`, `PERIOD_LABEL`, `documentError` (Task 2); `searchTerminology` (19a) e `CodeSearch` (19a, `src/modules/consultation/CodeSearch.tsx`); `consultationError` (19a).
- Produces (os três só montam o corpo e chamam `onSubmit`; quem emite é o contêiner da Task 6 ou a recepção da Task 8; recusa do `onSubmit` aparece no próprio formulário, que fica preenchido):
  ```tsx
  SickNoteForm({ initial: SickNoteDraft; onSubmit(content: SickNoteInput): Promise<void>; onCancel(): void; searchDelayMs?: number; replacing?: boolean })
  DeclarationForm({ initial: DeclarationDraft; showIssuerRegistration?: boolean; onSubmit(content: DeclarationInput): Promise<void>; onCancel(): void; replacing?: boolean })
  ExamRequisitionForm({ initialNote?: string; onSubmit(content: ExamRequisitionInput): Promise<void>; onCancel(): void; replacing?: boolean })
  ```
  Cada um é um `<form aria-label="Atestado" | "Declaração de comparecimento" | "Requisição de exames">`, com os botões "Emitir atestado" / "Emitir declaração" / "Emitir requisição" e "Cancelar"; `replacing` mostra "Substitui o documento cancelado".
  `src/modules/documents/formStyles.ts` exporta `formBox`, `labelStyle`, `checkLabel`, `rowStyle`, `alertStyle`, `mutedStyle`, `linkButton` (os estilos repetidos dos formulários desta pasta).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/documents/SickNoteForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, searchTerminology: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { emptySickNote } from "../../lib/clinicalDocuments";
import { SickNoteForm } from "./SickNoteForm";
import { TODAY19C } from "../../test/documentFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function renderIt(onSubmit = vi.fn(async () => {})) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<SickNoteForm initial={emptySickNote(TODAY19C)} onSubmit={onSubmit} onCancel={vi.fn()} searchDelayMs={0} />, { wrapper });
  return onSubmit;
}

async function pickCid() {
  fireEvent.change(screen.getByLabelText("CID-10 (opcional, só com autorização do paciente)"), { target: { value: "ivas" } });
  const list = await screen.findByRole("list", { name: "resultados: CID-10 (opcional, só com autorização do paciente)" });
  fireEvent.click(within(list).getByRole("button", { name: "J069 — Infecção aguda das vias aéreas superiores" }));
}

describe("SickNoteForm", () => {
  beforeEach(() => {
    mocked(api.searchTerminology).mockReset();
    mocked(api.searchTerminology).mockResolvedValue([ { code: "J069", label: "Infecção aguda das vias aéreas superiores" } ]);
  });

  it("afastamento com CID autorizado: manda dias, início, código e a autorização", async () => {
    const onSubmit = renderIt();
    fireEvent.change(screen.getByLabelText("Dias de afastamento"), { target: { value: "3" } });
    await pickCid();
    expect(api.searchTerminology).toHaveBeenCalledWith("ivas", "cid10");
    fireEvent.click(screen.getByLabelText("O paciente autorizou o CID no atestado"));
    fireEvent.click(screen.getByRole("button", { name: "Emitir atestado" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({
      type: "leave", days: 3, start_on: TODAY19C, cid10: { code: "J069" }, cid_authorized: true
    }));
  });

  it("CID escolhido sem marcar a autorização: não emite e diz por quê", async () => {
    const onSubmit = renderIt();
    fireEvent.change(screen.getByLabelText("Dias de afastamento"), { target: { value: "3" } });
    await pickCid();
    fireEvent.click(screen.getByRole("button", { name: "Emitir atestado" }));
    expect((await screen.findByRole("alert")).textContent)
      .toBe("o CID só entra com a autorização do paciente — marque a autorização ou tire o CID");
    expect(onSubmit).not.toHaveBeenCalled();
    fireEvent.click(screen.getByRole("button", { name: "tirar o CID" }));
    fireEvent.click(screen.getByRole("button", { name: "Emitir atestado" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({ type: "leave", days: 3, start_on: TODAY19C, cid_authorized: false }));
  });

  it("acompanhante: pede nome, parentesco e motivo; trocar para acompanhante tira o CID", async () => {
    const onSubmit = renderIt();
    await pickCid();
    fireEvent.click(screen.getByLabelText("Acompanhante"));
    expect(screen.queryByText(/J069/)).toBeNull();
    expect(screen.queryByLabelText("CID-10 (opcional, só com autorização do paciente)")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Emitir atestado" }));
    expect((await screen.findByRole("alert")).textContent).toBe("informe o nome do acompanhante");
    fireEvent.change(screen.getByLabelText("Nome do acompanhante"), { target: { value: "Marcos Lima" } });
    fireEvent.change(screen.getByLabelText("Parentesco"), { target: { value: "filho" } });
    fireEvent.change(screen.getByLabelText("Motivo (CLT art. 473)"), { target: { value: "clt_473_xi" } });
    fireEvent.click(screen.getByRole("button", { name: "Emitir atestado" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({
      type: "companion", start_on: TODAY19C, companion_name: "Marcos Lima", companion_kinship: "filho",
      companion_reason: "clt_473_xi", cid_authorized: false
    }));
  });

  it("recusa do api fica no formulário com o que foi digitado", async () => {
    renderIt(vi.fn(async () => { throw new ApiError(403, { error: "cbo_not_allowed" }, "403"); }));
    fireEvent.change(screen.getByLabelText("Dias de afastamento"), { target: { value: "3" } });
    fireEvent.click(screen.getByRole("button", { name: "Emitir atestado" }));
    expect((await screen.findByRole("alert")).textContent).toBe("o seu CBO não emite este tipo de documento");
    expect((screen.getByLabelText("Dias de afastamento") as HTMLInputElement).value).toBe("3");
  });
});
```

```tsx
// src/modules/documents/DeclarationForm.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { emptyDeclaration } from "../../lib/clinicalDocuments";
import { DeclarationForm } from "./DeclarationForm";
import { TODAY19C } from "../../test/documentFixtures";

afterEach(cleanup);

describe("DeclarationForm", () => {
  it("período: manda o dia e o período", async () => {
    const onSubmit = vi.fn(async () => {});
    render(<DeclarationForm initial={emptyDeclaration(TODAY19C)} onSubmit={onSubmit} onCancel={vi.fn()} />);
    fireEvent.click(screen.getByRole("button", { name: "Emitir declaração" }));
    expect((await screen.findByRole("alert")).textContent).toBe("escolha o período");
    fireEvent.change(screen.getByLabelText("Período"), { target: { value: "morning" } });
    fireEvent.click(screen.getByRole("button", { name: "Emitir declaração" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({ period: "morning" }));
    expect(screen.queryByLabelText("Sua matrícula (sai no papel)")).toBeNull();
  });

  it("horário: chegada obrigatória, saída depois da chegada", async () => {
    const onSubmit = vi.fn(async () => {});
    render(<DeclarationForm initial={emptyDeclaration(TODAY19C, "08:15")} onSubmit={onSubmit} onCancel={vi.fn()} />);
    expect((screen.getByLabelText("Chegada") as HTMLInputElement).value).toBe("08:15");
    fireEvent.change(screen.getByLabelText("Saída"), { target: { value: "08:00" } });
    fireEvent.click(screen.getByRole("button", { name: "Emitir declaração" }));
    expect((await screen.findByRole("alert")).textContent).toBe("a saída precisa ser depois da chegada");
    fireEvent.change(screen.getByLabelText("Saída"), { target: { value: "10:30" } });
    fireEvent.change(screen.getByLabelText("Acompanhante (opcional)"), { target: { value: "Marcos Lima" } });
    fireEvent.click(screen.getByRole("button", { name: "Emitir declaração" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({
      arrived_at: "08:15", left_at: "10:30", companion_name: "Marcos Lima"
    }));
  });

  it("recepção: a matrícula de quem emite vai junto", async () => {
    const onSubmit = vi.fn(async () => {});
    render(<DeclarationForm initial={emptyDeclaration(TODAY19C)} showIssuerRegistration onSubmit={onSubmit} onCancel={vi.fn()} />);
    fireEvent.change(screen.getByLabelText("Período"), { target: { value: "full_day" } });
    fireEvent.change(screen.getByLabelText("Sua matrícula (sai no papel)"), { target: { value: "4521" } });
    fireEvent.click(screen.getByRole("button", { name: "Emitir declaração" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({ period: "full_day", issuer_registration: "4521" }));
  });
});
```

```tsx
// src/modules/documents/ExamRequisitionForm.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { ApiError } from "../../lib/api";
import { ExamRequisitionForm } from "./ExamRequisitionForm";

afterEach(cleanup);

describe("ExamRequisitionForm", () => {
  it("manda a observação; sem exames na consulta a frase diz o que fazer", async () => {
    const onSubmit = vi.fn()
      .mockRejectedValueOnce(new ApiError(422, { error: "no_exam_requests" }, "422"))
      .mockResolvedValueOnce(undefined);
    render(<ExamRequisitionForm onSubmit={onSubmit} onCancel={vi.fn()} />);
    expect(screen.getByText(/os exames pedidos na consulta/)).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Emitir requisição" }));
    expect((await screen.findByRole("alert")).textContent)
      .toBe("esta consulta não tem exames pedidos — inclua os exames na consulta (ou num adendo) antes");
    expect(onSubmit).toHaveBeenLastCalledWith({});
    fireEvent.change(screen.getByLabelText("Observação (opcional)"), { target: { value: "jejum de 8 h" } });
    fireEvent.click(screen.getByRole("button", { name: "Emitir requisição" }));
    await waitFor(() => expect(onSubmit).toHaveBeenLastCalledWith({ note: "jejum de 8 h" }));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/documents/SickNoteForm.test.tsx src/modules/documents/DeclarationForm.test.tsx src/modules/documents/ExamRequisitionForm.test.tsx`
Expected: FAIL — os três formulários não existem.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/modules/documents/formStyles.ts
// Estilos repetidos dos formulários de documento (módulo 19c). Os botões e
// campos são os de src/components/formStyles.ts; aqui só a moldura e textos.
import type { CSSProperties } from "react";

export const formBox: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 };
export const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
export const checkLabel: CSSProperties = { display: "flex", gap: 6, alignItems: "center", fontSize: 12, color: "var(--ink2)" };
export const rowStyle: CSSProperties = { display: "flex", gap: 8, alignItems: "flex-end", flexWrap: "wrap" };
export const alertStyle: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
export const mutedStyle: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
export const linkButton: CSSProperties = {
  border: "none", background: "transparent", padding: 0, cursor: "pointer", fontSize: 12, color: "var(--accent)", textDecoration: "underline"
};
```

```tsx
// src/modules/documents/SickNoteForm.tsx
// Atestado (módulo 19c; spec §4; contrato §2): afastamento (dias e início) ou
// acompanhante (nome, parentesco, motivo da CLT art. 473). O CID só entra com
// a autorização do paciente marcada aqui — ela fica registrada no documento e
// no prontuário; o atestado de acompanhante nunca leva CID. Quem emite (CBO de
// médico ou dentista) o api confere.
import { useState, type FormEvent } from "react";
import { searchTerminology, type CompanionReason, type SickNoteInput } from "../../lib/api";
import {
  COMPANION_REASON_LABEL, documentError, sickNoteInput, sickNoteProblem, type SickNoteDraft
} from "../../lib/clinicalDocuments";
import { consultationError } from "../../lib/consultation";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { CodeSearch } from "../consultation/CodeSearch";
import { alertStyle, checkLabel, formBox, labelStyle, linkButton, mutedStyle, rowStyle } from "./formStyles";

const CID_LABEL = "CID-10 (opcional, só com autorização do paciente)";

interface Props {
  initial: SickNoteDraft;
  onSubmit(content: SickNoteInput): Promise<void>;
  onCancel(): void;
  searchDelayMs?: number;
  replacing?: boolean;
}

export function SickNoteForm({ initial, onSubmit, onCancel, searchDelayMs, replacing }: Props) {
  const [ d, setD ] = useState<SickNoteDraft>(initial);
  const [ busy, setBusy ] = useState(false);
  const [ problem, setProblem ] = useState<string | null>(null);
  const set = (patch: Partial<SickNoteDraft>) => setD((prev) => ({ ...prev, ...patch }));

  async function submit(event: FormEvent) {
    event.preventDefault();
    if (busy) return;
    const found = sickNoteProblem(d);
    if (found) { setProblem(found); return; }
    setBusy(true); setProblem(null);
    try {
      await onSubmit(sickNoteInput(d));
    } catch (err) {
      setProblem(documentError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <form aria-label="Atestado" onSubmit={(e) => void submit(e)} style={formBox}>
      <strong>Atestado</strong>
      {replacing && <p style={mutedStyle}>Substitui o documento cancelado.</p>}
      <div role="radiogroup" aria-label="tipo de atestado" style={rowStyle}>
        <label style={checkLabel}>
          <input type="radio" name="sick-note-type" checked={d.type === "leave"} onChange={() => set({ type: "leave" })} />
          Afastamento
        </label>
        <label style={checkLabel}>
          <input type="radio" name="sick-note-type" checked={d.type === "companion"}
            onChange={() => set({ type: "companion", cid: null, cidAuthorized: false })} />
          Acompanhante
        </label>
      </div>

      {d.type === "leave" ? (
        <>
          <div style={rowStyle}>
            <label style={labelStyle}>
              Dias de afastamento
              <input inputMode="numeric" value={d.days} style={inputStyle} onChange={(e) => set({ days: e.target.value })} />
            </label>
            <label style={labelStyle}>
              Início
              <input type="date" value={d.startOn} style={inputStyle} onChange={(e) => set({ startOn: e.target.value })} />
            </label>
          </div>
          {d.cid ? (
            <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
              <p style={{ margin: 0, fontSize: 12.5 }}>
                <span className="mono">{d.cid.code}</span>{` — ${d.cid.label} `}
                <button type="button" style={linkButton} onClick={() => set({ cid: null, cidAuthorized: false })}>tirar o CID</button>
              </p>
              <label style={checkLabel}>
                <input type="checkbox" checked={d.cidAuthorized} onChange={(e) => set({ cidAuthorized: e.target.checked })} />
                O paciente autorizou o CID no atestado
              </label>
              <p style={mutedStyle}>A autorização fica registrada no documento e no prontuário.</p>
            </div>
          ) : (
            <CodeSearch label={CID_LABEL} queryKey="terminologySearch:cid10" search={(term) => searchTerminology(term, "cid10")}
              onPick={(item) => set({ cid: item, cidAuthorized: false })} errorText={consultationError} delayMs={searchDelayMs} />
          )}
        </>
      ) : (
        <>
          <div style={rowStyle}>
            <label style={labelStyle}>
              Nome do acompanhante
              <input value={d.companionName} style={inputStyle} onChange={(e) => set({ companionName: e.target.value })} />
            </label>
            <label style={labelStyle}>
              Parentesco
              <input value={d.companionKinship} style={inputStyle} onChange={(e) => set({ companionKinship: e.target.value })} />
            </label>
            <label style={labelStyle}>
              Dia do acompanhamento
              <input type="date" value={d.startOn} style={inputStyle} onChange={(e) => set({ startOn: e.target.value })} />
            </label>
          </div>
          <label style={labelStyle}>
            Motivo (CLT art. 473)
            <select value={d.companionReason} style={inputStyle}
              onChange={(e) => set({ companionReason: e.target.value as CompanionReason | "" })}>
              <option value="">escolha</option>
              {(Object.keys(COMPANION_REASON_LABEL) as CompanionReason[]).map((r) => (
                <option key={r} value={r}>{COMPANION_REASON_LABEL[r]}</option>
              ))}
            </select>
          </label>
        </>
      )}

      <label style={labelStyle}>
        Observação (opcional)
        <textarea value={d.note} rows={2} style={inputStyle} onChange={(e) => set({ note: e.target.value })} />
      </label>
      {problem && <p role="alert" style={alertStyle}>{problem}</p>}
      <div style={rowStyle}>
        <button type="submit" disabled={busy} style={busy ? disabledButtonStyle : buttonStyle}>Emitir atestado</button>
        <button type="button" disabled={busy} style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </form>
  );
}
```

```tsx
// src/modules/documents/DeclarationForm.tsx
// Declaração de comparecimento (módulo 19c; spec §4; contrato §2 e §3): o dia
// e o período, ou a hora de chegada (e a de saída). Usada na consulta (pela
// autora) e na recepção (pelo atendimento), onde o papel traz a matrícula de
// quem emite. A unidade sai do atendimento, no api.
import { useState, type FormEvent } from "react";
import type { DeclarationInput, DeclarationPeriod } from "../../lib/api";
import { PERIOD_LABEL, declarationInput, declarationProblem, documentError, type DeclarationDraft } from "../../lib/clinicalDocuments";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alertStyle, checkLabel, formBox, labelStyle, mutedStyle, rowStyle } from "./formStyles";

interface Props {
  initial: DeclarationDraft;
  showIssuerRegistration?: boolean;
  onSubmit(content: DeclarationInput): Promise<void>;
  onCancel(): void;
  replacing?: boolean;
}

export function DeclarationForm({ initial, showIssuerRegistration, onSubmit, onCancel, replacing }: Props) {
  const [ d, setD ] = useState<DeclarationDraft>(initial);
  const [ busy, setBusy ] = useState(false);
  const [ problem, setProblem ] = useState<string | null>(null);
  const set = (patch: Partial<DeclarationDraft>) => setD((prev) => ({ ...prev, ...patch }));

  async function submit(event: FormEvent) {
    event.preventDefault();
    if (busy) return;
    const found = declarationProblem(d);
    if (found) { setProblem(found); return; }
    setBusy(true); setProblem(null);
    try {
      await onSubmit(declarationInput(d));
    } catch (err) {
      setProblem(documentError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <form aria-label="Declaração de comparecimento" onSubmit={(e) => void submit(e)} style={formBox}>
      <strong>Declaração de comparecimento</strong>
      {replacing && <p style={mutedStyle}>Substitui o documento cancelado.</p>}
      <p style={mutedStyle}>O dia é o do atendimento; a unidade também (o api preenche os dois).</p>
      <div style={rowStyle}>
        <div role="radiogroup" aria-label="como informar" style={rowStyle}>
          <label style={checkLabel}>
            <input type="radio" name="declaration-mode" checked={d.mode === "period"} onChange={() => set({ mode: "period" })} />
            Por período
          </label>
          <label style={checkLabel}>
            <input type="radio" name="declaration-mode" checked={d.mode === "times"} onChange={() => set({ mode: "times" })} />
            Por horário
          </label>
        </div>
      </div>
      {d.mode === "period" ? (
        <label style={labelStyle}>
          Período
          <select value={d.period} style={inputStyle} onChange={(e) => set({ period: e.target.value as DeclarationPeriod | "" })}>
            <option value="">escolha</option>
            {(Object.keys(PERIOD_LABEL) as DeclarationPeriod[]).map((p) => <option key={p} value={p}>{PERIOD_LABEL[p]}</option>)}
          </select>
        </label>
      ) : (
        <div style={rowStyle}>
          <label style={labelStyle}>
            Chegada
            <input type="time" value={d.arrivedAt} style={inputStyle} onChange={(e) => set({ arrivedAt: e.target.value })} />
          </label>
          <label style={labelStyle}>
            Saída
            <input type="time" value={d.leftAt} style={inputStyle} onChange={(e) => set({ leftAt: e.target.value })} />
          </label>
        </div>
      )}
      <label style={labelStyle}>
        Acompanhante (opcional)
        <input value={d.companionName} style={inputStyle} onChange={(e) => set({ companionName: e.target.value })} />
      </label>
      {showIssuerRegistration && (
        <label style={labelStyle}>
          Sua matrícula (sai no papel)
          <input value={d.issuerRegistration} style={inputStyle} onChange={(e) => set({ issuerRegistration: e.target.value })} />
        </label>
      )}
      {problem && <p role="alert" style={alertStyle}>{problem}</p>}
      <div style={rowStyle}>
        <button type="submit" disabled={busy} style={busy ? disabledButtonStyle : buttonStyle}>Emitir declaração</button>
        <button type="button" disabled={busy} style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </form>
  );
}
```

```tsx
// src/modules/documents/ExamRequisitionForm.tsx
// Requisição de exames em PDF (módulo 19c; spec §4; contrato §3): o api monta
// a requisição com os exames pedidos na consulta (o que já está salvo); aqui
// só a observação. Sem exames, o api recusa (no_exam_requests).
import { useState, type FormEvent } from "react";
import type { ExamRequisitionInput } from "../../lib/api";
import { documentError } from "../../lib/clinicalDocuments";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alertStyle, formBox, labelStyle, mutedStyle, rowStyle } from "./formStyles";

interface Props {
  initialNote?: string;
  onSubmit(content: ExamRequisitionInput): Promise<void>;
  onCancel(): void;
  replacing?: boolean;
}

export function ExamRequisitionForm({ initialNote = "", onSubmit, onCancel, replacing }: Props) {
  const [ note, setNote ] = useState(initialNote);
  const [ busy, setBusy ] = useState(false);
  const [ problem, setProblem ] = useState<string | null>(null);

  async function submit(event: FormEvent) {
    event.preventDefault();
    if (busy) return;
    setBusy(true); setProblem(null);
    try {
      const trimmed = note.trim();
      await onSubmit(trimmed ? { note: trimmed } : {});
    } catch (err) {
      setProblem(documentError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <form aria-label="Requisição de exames" onSubmit={(e) => void submit(e)} style={formBox}>
      <strong>Requisição de exames</strong>
      {replacing && <p style={mutedStyle}>Substitui o documento cancelado.</p>}
      <p style={mutedStyle}>
        A requisição sai com os exames pedidos na consulta (o que já está salvo). Para mudar a lista, edite a consulta — ou faça um
        adendo, se ela já foi finalizada.
      </p>
      <label style={labelStyle}>
        Observação (opcional)
        <textarea value={note} rows={2} style={inputStyle} onChange={(e) => setNote(e.target.value)} />
      </label>
      {problem && <p role="alert" style={alertStyle}>{problem}</p>}
      <div style={rowStyle}>
        <button type="submit" disabled={busy} style={busy ? disabledButtonStyle : buttonStyle}>Emitir requisição</button>
        <button type="button" disabled={busy} style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </form>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/documents/ && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19c
/opt/homebrew/bin/git add src/modules/documents/formStyles.ts \
  src/modules/documents/SickNoteForm.tsx src/modules/documents/SickNoteForm.test.tsx \
  src/modules/documents/DeclarationForm.tsx src/modules/documents/DeclarationForm.test.tsx \
  src/modules/documents/ExamRequisitionForm.tsx src/modules/documents/ExamRequisitionForm.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the sick note, attendance declaration and exam requisition forms

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Formulário da receita (catálogo, texto livre, enfermagem por protocolo, antimicrobiano)

**Files:**
- Create: `src/modules/documents/PrescriptionForm.tsx`
- Test: `src/modules/documents/PrescriptionForm.test.tsx`

**Interfaces:**
- Consumes: `listNursingProtocols` (Task 1); `PrescriptionDraft`, `ItemDraft`, `emptyItem`, `itemFromMedication`, `prescriptionProblems`, `prescriptionInput`, `hasAntimicrobial`, `professionalKind`, `vigentProtocols`, `protocolLabel`, `ROUTES`, `ROUTE_LABEL`, `ANTIMICROBIAL_NOTE`, `NO_PROTOCOL`, `PROTOCOLS_KEY`, `documentError` (Task 2); `MedicationSearch` (Task 3); `src/modules/documents/formStyles.ts` (Task 4).
- Produces: `PrescriptionForm({ cboCode: string; unitId?: string; today: string; initial: PrescriptionDraft; onSubmit(content: PrescriptionInput): Promise<void>; onCancel(): void; searchDelayMs?: number; replacing?: boolean })` — `<form aria-label="Receita">`. Médico e dentista: `MedicationSearch` (controlado bloqueado) e "Incluir em texto livre". Enfermeiro (`2235xx`): escolhe o protocolo vigente (`<select>` "Protocolo de enfermagem") e inclui só os itens dele (botões "Incluir <medicamento>"), sem texto livre; sem protocolo vigente, `NO_PROTOCOL`. Cada item é um `<li aria-label="item N">` com "Quantidade N", "Unidade N", "Via N", "Duração em dias N", "Posologia N", "Uso contínuo N" e "Remover item N"; o de texto livre tem "Medicamento N (texto livre)". Antimicrobiano mostra `ANTIMICROBIAL_NOTE` (`role="note"`). Botões "Emitir receita" e "Cancelar"; o que falta aparece numa lista `role="alert"`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/documents/PrescriptionForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, searchMedications: vi.fn(), listNursingProtocols: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError, type PrescriptionInput } from "../../lib/api";
import { ANTIMICROBIAL_NOTE, NO_PROTOCOL, type PrescriptionDraft } from "../../lib/clinicalDocuments";
import { PrescriptionForm } from "./PrescriptionForm";
import { AMOXICILLIN, CLONAZEPAM, TODAY19C, nursingProtocol, searchItem } from "../../test/documentFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const EMPTY: PrescriptionDraft = { items: [], protocolVersionId: "" };

function renderIt(cboCode: string, onSubmit: (c: PrescriptionInput) => Promise<void> = vi.fn(async () => {})) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<PrescriptionForm cboCode={cboCode} unitId="u1" today={TODAY19C} initial={EMPTY} onSubmit={onSubmit} onCancel={vi.fn()}
    searchDelayMs={0} />, { wrapper });
  return onSubmit;
}

async function pick(term: string, name: string) {
  fireEvent.change(screen.getByLabelText("Buscar medicamento"), { target: { value: term } });
  const list = await screen.findByRole("list", { name: "resultados: Buscar medicamento" });
  fireEvent.click(within(list).getByRole("button", { name }));
}

function fill(n: number, values: { quantity?: string; unit?: string; route?: string; dosage?: string; duration?: string }) {
  if (values.quantity !== undefined) fireEvent.change(screen.getByLabelText(`Quantidade ${n}`), { target: { value: values.quantity } });
  if (values.unit !== undefined) fireEvent.change(screen.getByLabelText(`Unidade ${n}`), { target: { value: values.unit } });
  if (values.route !== undefined) fireEvent.change(screen.getByLabelText(`Via ${n}`), { target: { value: values.route } });
  if (values.dosage !== undefined) fireEvent.change(screen.getByLabelText(`Posologia ${n}`), { target: { value: values.dosage } });
  if (values.duration !== undefined) fireEvent.change(screen.getByLabelText(`Duração em dias ${n}`), { target: { value: values.duration } });
}

describe("PrescriptionForm", () => {
  beforeEach(() => {
    mocked(api.searchMedications).mockReset();
    mocked(api.listNursingProtocols).mockReset();
    mocked(api.searchMedications).mockResolvedValue([ searchItem(), AMOXICILLIN, CLONAZEPAM ]);
    mocked(api.listNursingProtocols).mockResolvedValue([ nursingProtocol() ]);
  });

  it("médica: inclui do catálogo (REMUME marcada) e em texto livre, e manda a receita", async () => {
    const onSubmit = renderIt("225142");
    await pick("metf", "Metformina, cloridrato 850 mg, comprimido");
    const first = screen.getByRole("listitem", { name: "item 1" });
    expect(within(first).getByText("na rede")).not.toBeNull();
    expect((screen.getByLabelText("Unidade 1") as HTMLInputElement).value).toBe("comprimido");
    fill(1, { quantity: "60", route: "oral", dosage: "1 comprimido após o café e 1 após o jantar", duration: "30" });
    fireEvent.click(screen.getByLabelText("Uso contínuo 1"));
    fireEvent.click(screen.getByRole("button", { name: "Incluir em texto livre" }));
    fireEvent.change(screen.getByLabelText("Medicamento 2 (texto livre)"), { target: { value: "Soro fisiológico 0,9%" } });
    fill(2, { quantity: "1", unit: "frasco", route: "nasal", dosage: "2 jatos em cada narina 3 vezes ao dia" });
    fireEvent.click(screen.getByRole("button", { name: "Emitir receita" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({ items: [
      { catalog_item: { id: "ci1" }, quantity: 60, quantity_unit: "comprimido", route: "oral",
        dosage_instructions: "1 comprimido após o café e 1 após o jantar", duration_days: 30, continuous: true },
      { free_text: "Soro fisiológico 0,9%", quantity: 1, quantity_unit: "frasco", route: "nasal",
        dosage_instructions: "2 jatos em cada narina 3 vezes ao dia", continuous: false }
    ] }));
    expect(api.searchMedications).toHaveBeenCalledWith("metf", "u1");
    expect(api.listNursingProtocols).not.toHaveBeenCalled();
  });

  it("controlado não entra: o botão vem desligado com o aviso do 19d", async () => {
    renderIt("225142");
    fireEvent.change(screen.getByLabelText("Buscar medicamento"), { target: { value: "clon" } });
    const list = await screen.findByRole("list", { name: "resultados: Buscar medicamento" });
    fireEvent.click(within(list).getByRole("button", { name: "Clonazepam 2 mg, comprimido" }));
    expect(screen.queryByRole("listitem", { name: "item 1" })).toBeNull();
    expect(within(list).getByText("receita de controle especial — 19d")).not.toBeNull();
  });

  it("antimicrobiano: avisa papel, 2 vias e 10 dias antes de emitir", async () => {
    renderIt("225142");
    expect(screen.queryByRole("note")).toBeNull();
    await pick("amox", "Amoxicilina 500 mg, cápsula");
    expect(screen.getByRole("note").textContent).toBe(ANTIMICROBIAL_NOTE);
    fireEvent.click(screen.getByRole("button", { name: "Remover item 1" }));
    expect(screen.queryByRole("note")).toBeNull();
  });

  it("faltando campo: lista o que falta por item e não manda", async () => {
    const onSubmit = renderIt("225142");
    fireEvent.click(screen.getByRole("button", { name: "Emitir receita" }));
    expect((await screen.findByRole("alert")).textContent).toBe("inclua ao menos um medicamento");
    fireEvent.click(screen.getByRole("button", { name: "Incluir em texto livre" }));
    fireEvent.click(screen.getByRole("button", { name: "Emitir receita" }));
    const items = within(await screen.findByRole("alert")).getAllByRole("listitem").map((li) => li.textContent);
    expect(items).toContain("item 1: escolha um medicamento do catálogo ou escreva em texto livre");
    expect(items).toContain("item 1: escolha a via");
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it("enfermeira: escolhe o protocolo vigente, só itens dele, sem texto livre; acima da dose máxima não manda", async () => {
    const onSubmit = renderIt("223565");
    const select = await screen.findByLabelText("Protocolo de enfermagem");
    expect(screen.queryByLabelText("Buscar medicamento")).toBeNull();
    expect(screen.queryByRole("button", { name: "Incluir em texto livre" })).toBeNull();
    fireEvent.change(select, { target: { value: "npv1" } });
    fireEvent.click(screen.getByRole("button", { name: "Incluir Dipirona sódica 500 mg/mL, solução oral" }));
    expect(screen.getByText("protocolo: até 2 frasco")).not.toBeNull();
    fill(1, { quantity: "3", unit: "frasco", route: "oral", dosage: "20 gotas até de 6 em 6 horas" });
    fireEvent.click(screen.getByRole("button", { name: "Emitir receita" }));
    const problems = within(await screen.findByRole("alert")).getAllByRole("listitem").map((li) => li.textContent);
    expect(problems).toEqual([ "item 1: acima da dose máxima do protocolo (até 2 frasco)" ]);
    expect(onSubmit).not.toHaveBeenCalled();
    fill(1, { quantity: "1" });
    fireEvent.click(screen.getByRole("button", { name: "Emitir receita" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({
      items: [ { catalog_item: { id: "ci4" }, quantity: 1, quantity_unit: "frasco", route: "oral", dosage_instructions: "20 gotas até de 6 em 6 horas", continuous: false } ],
      nursing_protocol: { version_id: "npv1" }
    }));
  });

  it("enfermeira sem protocolo vigente: diz que não prescreve sem ele", async () => {
    mocked(api.listNursingProtocols).mockResolvedValue([
      nursingProtocol({ current_version: { id: "npv0", valid_from: "2024-01-01", valid_until: "2025-12-31", items: [] } })
    ]);
    renderIt("223565");
    expect((await screen.findByRole("alert")).textContent).toBe(NO_PROTOCOL);
    expect(screen.queryByLabelText("Protocolo de enfermagem")).toBeNull();
  });

  it("CNPJ da cidade faltando: a frase diz quem resolve e o formulário fica preenchido", async () => {
    renderIt("223565", vi.fn(async () => { throw new ApiError(422, { error: "city_cnpj_missing" }, "422"); }));
    fireEvent.change(await screen.findByLabelText("Protocolo de enfermagem"), { target: { value: "npv1" } });
    fireEvent.click(screen.getByRole("button", { name: "Incluir Dipirona sódica 500 mg/mL, solução oral" }));
    fill(1, { quantity: "1", unit: "frasco", route: "oral", dosage: "20 gotas" });
    fireEvent.click(screen.getByRole("button", { name: "Emitir receita" }));
    expect((await screen.findByRole("alert")).textContent)
      .toBe("a cidade ainda não cadastrou o CNPJ — sem ele a receita de enfermagem não sai; avise o administrador da cidade");
    expect((screen.getByLabelText("Quantidade 1") as HTMLInputElement).value).toBe("1");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/documents/PrescriptionForm.test.tsx`
Expected: FAIL — `./PrescriptionForm` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/documents/PrescriptionForm.tsx
// Receita comum (módulo 19c; spec §5; contrato §2–§5). Médico e dentista
// buscam no catálogo (REMUME da unidade primeiro; controlado bloqueado — 19d)
// ou escrevem em texto livre. Enfermeiro prescreve só itens do protocolo
// municipal vigente (COFEN 801/2026), sem texto livre e até a dose máxima; o
// CNPJ da cidade e o Coren entram no documento pelo api. Antimicrobiano sai em
// papel, 2 vias, válido por 10 dias (o api decide; a tela avisa antes).
import { useState, type CSSProperties, type FormEvent } from "react";
import { useQuery } from "@tanstack/react-query";
import { listNursingProtocols, type MedicationRoute, type PrescriptionInput } from "../../lib/api";
import {
  ANTIMICROBIAL_NOTE, NO_PROTOCOL, PROTOCOLS_KEY, ROUTES, ROUTE_LABEL, documentError, emptyItem, hasAntimicrobial, itemFromMedication,
  prescriptionInput, prescriptionProblems, professionalKind, protocolLabel, vigentProtocols, type ItemDraft, type PrescriptionDraft
} from "../../lib/clinicalDocuments";
import { Tag } from "../../components/Tag";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { MedicationSearch } from "./MedicationSearch";
import { alertStyle, checkLabel, formBox, labelStyle, mutedStyle, rowStyle } from "./formStyles";

interface Props {
  cboCode: string;
  unitId?: string;
  today: string;
  initial: PrescriptionDraft;
  onSubmit(content: PrescriptionInput): Promise<void>;
  onCancel(): void;
  searchDelayMs?: number;
  replacing?: boolean;
}

export function PrescriptionForm({ cboCode, unitId, today, initial, onSubmit, onCancel, searchDelayMs, replacing }: Props) {
  const nurse = professionalKind(cboCode) === "nurse";
  const protocols = useQuery({ queryKey: [ PROTOCOLS_KEY ], queryFn: listNursingProtocols, enabled: nurse, retry: false });
  const [ d, setD ] = useState<PrescriptionDraft>(initial);
  const [ busy, setBusy ] = useState(false);
  const [ problems, setProblems ] = useState<string[]>([]);
  const vigent = vigentProtocols(protocols.data ?? [], today);
  const chosen = vigent.find((p) => p.current_version?.id === d.protocolVersionId) ?? null;

  const addItem = (item: ItemDraft) => setD((prev) => ({ ...prev, items: [ ...prev.items, item ] }));
  const setItem = (key: string, patch: Partial<ItemDraft>) =>
    setD((prev) => ({ ...prev, items: prev.items.map((it) => (it.key === key ? { ...it, ...patch } : it)) }));
  const removeItem = (key: string) => setD((prev) => ({ ...prev, items: prev.items.filter((it) => it.key !== key) }));

  async function submit(event: FormEvent) {
    event.preventDefault();
    if (busy) return;
    const found = prescriptionProblems(d, nurse);
    if (found.length > 0) { setProblems(found); return; }
    setBusy(true); setProblems([]);
    try {
      await onSubmit(prescriptionInput(d));
    } catch (err) {
      setProblems([ documentError(err) ]);
    } finally {
      setBusy(false);
    }
  }

  return (
    <form aria-label="Receita" onSubmit={(e) => void submit(e)} style={formBox}>
      <strong>Receita</strong>
      {replacing && <p style={mutedStyle}>Substitui o documento cancelado.</p>}

      {nurse ? (
        <>
          {protocols.isPending && <p className="mono" style={mutedStyle}>carregando os protocolos…</p>}
          {protocols.isError && <p role="alert" style={alertStyle}>{documentError(protocols.error)}</p>}
          {protocols.isSuccess && vigent.length === 0 && <p role="alert" style={alertStyle}>{NO_PROTOCOL}</p>}
          {vigent.length > 0 && (
            <label style={labelStyle}>
              Protocolo de enfermagem
              <select value={d.protocolVersionId} style={inputStyle}
                onChange={(e) => setD({ items: [], protocolVersionId: e.target.value })}>
                <option value="">escolha</option>
                {vigent.map((p) => <option key={p.id} value={p.current_version!.id}>{protocolLabel(p)}</option>)}
              </select>
            </label>
          )}
          {chosen && (
            <ul aria-label="itens do protocolo" style={list}>
              {chosen.current_version!.items.map((pi) => (
                <li key={pi.catalog_item.id}>
                  <button type="button" style={{ ...secondaryButtonStyle, width: "100%", textAlign: "left" }}
                    aria-label={`Incluir ${pi.catalog_item.label}`}
                    onClick={() => addItem(itemFromMedication({ ...pi.catalog_item, max_dose: pi.max_dose ?? null }))}>
                    {`${pi.catalog_item.label}${pi.max_dose ? ` · até ${pi.max_dose.quantity} ${pi.max_dose.unit}` : ""}`}
                  </button>
                </li>
              ))}
            </ul>
          )}
        </>
      ) : (
        <>
          <MedicationSearch label="Buscar medicamento" unitId={unitId} blockControlled delayMs={searchDelayMs}
            onPick={(m) => addItem(itemFromMedication(m))} />
          <div>
            <button type="button" style={secondaryButtonStyle} onClick={() => addItem(emptyItem())}>Incluir em texto livre</button>
          </div>
        </>
      )}

      {hasAntimicrobial(d) && <p role="note" style={{ ...mutedStyle, fontWeight: 600, color: "var(--warn)" }}>{ANTIMICROBIAL_NOTE}</p>}

      {d.items.length === 0 ? <p style={mutedStyle}>nenhum medicamento na receita</p> : (
        <ol aria-label="itens da receita" style={{ ...list, gap: 10 }}>
          {d.items.map((it, i) => (
            <ItemFields key={it.key} index={i} item={it} onChange={(patch) => setItem(it.key, patch)} onRemove={() => removeItem(it.key)} />
          ))}
        </ol>
      )}

      {problems.length > 0 && (
        <ul role="alert" style={{ ...alertStyle, paddingLeft: 18 }}>
          {problems.map((p) => <li key={p}>{p}</li>)}
        </ul>
      )}
      <div style={rowStyle}>
        <button type="submit" disabled={busy} style={busy ? disabledButtonStyle : buttonStyle}>Emitir receita</button>
        <button type="button" disabled={busy} style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </form>
  );
}

function ItemFields({ index, item, onChange, onRemove }: {
  index: number; item: ItemDraft; onChange(patch: Partial<ItemDraft>): void; onRemove(): void;
}) {
  const n = index + 1;
  return (
    <li aria-label={`item ${n}`} style={itemBox}>
      <div style={rowStyle}>
        {item.medication ? <strong style={{ fontSize: 12.5 }}>{item.medication.label}</strong> : (
          <label style={{ ...labelStyle, flex: 1 }}>
            {`Medicamento ${n} (texto livre)`}
            <input value={item.freeText} style={inputStyle} onChange={(e) => onChange({ freeText: e.target.value })} />
          </label>
        )}
        {item.medication?.in_network && <Tag tone="ok" mono={false}>na rede</Tag>}
        {item.medication?.antimicrobial && <Tag tone="warn" mono={false}>antimicrobiano</Tag>}
        {item.medication?.max_dose && <span style={mutedStyle}>{`protocolo: até ${item.medication.max_dose.quantity} ${item.medication.max_dose.unit}`}</span>}
        <button type="button" aria-label={`Remover item ${n}`} style={secondaryButtonStyle} onClick={onRemove}>Remover</button>
      </div>
      <div style={rowStyle}>
        <label style={labelStyle}>
          {`Quantidade ${n}`}
          <input inputMode="numeric" value={item.quantity} style={inputStyle} onChange={(e) => onChange({ quantity: e.target.value })} />
        </label>
        <label style={labelStyle}>
          {`Unidade ${n}`}
          <input value={item.quantityUnit} style={inputStyle} onChange={(e) => onChange({ quantityUnit: e.target.value })} />
        </label>
        <label style={labelStyle}>
          {`Via ${n}`}
          <select value={item.route} style={inputStyle} onChange={(e) => onChange({ route: e.target.value as MedicationRoute | "" })}>
            <option value="">escolha</option>
            {ROUTES.map((r) => <option key={r} value={r}>{ROUTE_LABEL[r]}</option>)}
          </select>
        </label>
        <label style={labelStyle}>
          {`Duração em dias ${n}`}
          <input inputMode="numeric" value={item.durationDays} style={inputStyle} onChange={(e) => onChange({ durationDays: e.target.value })} />
        </label>
      </div>
      <label style={labelStyle}>
        {`Posologia ${n}`}
        <textarea value={item.dosage} rows={2} style={inputStyle} onChange={(e) => onChange({ dosage: e.target.value })} />
      </label>
      <label style={checkLabel}>
        <input type="checkbox" checked={item.continuous} onChange={(e) => onChange({ continuous: e.target.checked })} />
        {`Uso contínuo ${n}`}
      </label>
    </li>
  );
}

const list: CSSProperties = { listStyle: "none", margin: 0, padding: 0, display: "flex", flexDirection: "column", gap: 4 };
const itemBox: CSSProperties = { display: "flex", flexDirection: "column", gap: 6, padding: 10, border: "1px solid var(--rule)", borderRadius: 6 };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/documents/PrescriptionForm.test.tsx && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19c
/opt/homebrew/bin/git add src/modules/documents/PrescriptionForm.tsx src/modules/documents/PrescriptionForm.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the prescription form with catalog search, free text and nursing protocols

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Aba Documentos na consulta — lista, emissão, renovação, Imprimir, Cancelar e "Cancelar e emitir outro"

**Files:**
- Create: `src/modules/documents/ConsultationDocuments.tsx`, `src/modules/documents/CancelDocument.tsx`, `src/modules/documents/printDocument.ts`
- Modify: `src/lib/api.ts` (`SignatureDocumentType`), `src/lib/signature.ts` (`DOCUMENT_LABEL`), `src/lib/signature.test.ts`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/consultation/ConsultationWorkspace.documents.test.tsx`
- Test: `src/modules/documents/ConsultationDocuments.test.tsx`

**Interfaces:**
- Consumes: `listConsultationDocuments`, `issueDocument`, `cancelDocument`, `getPrescriptionRenewal`, `fetchDocumentPdf`, `ApiError`, `errorCode` (Task 1); de `clinicalDocuments.ts` (Task 2): `DOCUMENTS_KEY`, `MEDICATIONS_KEY`, `CANCEL_WARNING`, `canCancel`, `cancelReasonProblem`, `documentError`, `documentSituation`, `documentsRefetchInterval`, `emptyDeclaration`, `emptySickNote`, `issueModeLabel`, `issuedNotice`, `itemFromPrescription`, `kindLabel`, `kindsFor`, `reissueDraft`, `withDocument`, `ReissueDraft`; os quatro formulários (Tasks 4 e 5); `SignatureMarker` (19b); `SensitiveAction`; `SegmentedControl`; `todayInCity` (`campaigns.ts`); `useAuth`.
- Produces:
  - `printDocument(id: string, errorText: (err: unknown) => string): Promise<string | null>` — abre a janela **no clique** (antes de qualquer `await`), lê o PDF, aponta a janela para um `blob:` e devolve a frase do erro (ou `null`); `POPUP_BLOCKED`;
  - `CancelDocument({ doc, reissue, onDone(cancelled), onStale(), onCancel(), onGoToSecurity? })` — `SensitiveAction` com step-up e o campo "Motivo do cancelamento";
  - `ConsultationDocuments({ consultationId, issue?: IssueContext, searchDelayMs?, onGoToSecurity? })`, `IssueContext = { cboCode: string; unitId?: string }` — sem `issue`, só a lista (Imprimir e Cancelar);
  - `SignatureDocumentType` com `"clinical_document"`; `documentLabel("clinical_document")` → "documento clínico";
  - `ConsultationWorkspace`: abas "Consulta" / "Documentos" (`SegmentedControl`) quando há consulta e `clinical_documents`; o editor fica **montado** (escondido) na aba Documentos, para o autosave não perder o que foi digitado.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/documents/ConsultationDocuments.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listConsultationDocuments: vi.fn(), issueDocument: vi.fn(), cancelDocument: vi.fn(),
    getPrescriptionRenewal: vi.fn(), fetchDocumentPdf: vi.fn(), searchMedications: vi.fn(), searchTerminology: vi.fn(),
    listNursingProtocols: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { DOCUMENTS_DISABLED } from "../../lib/clinicalDocuments";
import { ConsultationDocuments } from "./ConsultationDocuments";
import { renderWithProviders } from "../../test/campaignFixtures";
import { NOW19C, TODAY19C, declarationDoc, docUser, prescriptionDoc, prescriptionItem, sickNoteDoc } from "../../test/documentFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
let win: { location: { href: string }; close: ReturnType<typeof vi.fn> };

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });

const ISSUE = { cboCode: "225142", unitId: "u1" };
const cancelled = (over = {}) => sickNoteDoc({ status: "cancelled", cancelled_at: "2026-10-09T10:05:00-03:00", cancel_reason: "dias errados no atestado", ...over });

function renderIt(issue: typeof ISSUE | undefined = ISSUE) {
  return renderWithProviders(<ConsultationDocuments consultationId="cs1" issue={issue} searchDelayMs={0} />);
}

describe("ConsultationDocuments", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW19C));
    for (const fn of [ api.fetchCurrentSession, api.listConsultationDocuments, api.issueDocument, api.cancelDocument,
      api.getPrescriptionRenewal, api.fetchDocumentPdf, api.searchMedications, api.searchTerminology, api.listNursingProtocols ]) {
      mocked(fn).mockReset();
    }
    mocked(api.fetchCurrentSession).mockResolvedValue(docUser());
    mocked(api.listConsultationDocuments).mockResolvedValue([ sickNoteDoc() ]);
    win = { location: { href: "" }, close: vi.fn() };
    vi.spyOn(window, "open").mockImplementation(() => win as unknown as Window);
    Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:doc1") });
    Object.defineProperty(URL, "revokeObjectURL", { configurable: true, value: vi.fn() });
  });

  it("lista os documentos com modo, assinatura, situação e código; cancelado não imprime nem cancela", async () => {
    mocked(api.listConsultationDocuments).mockResolvedValue([
      sickNoteDoc(),
      prescriptionDoc({ issue_mode: "digital", signature: { mode: "pending", request_id: "sr1", reason_code: "no_session" } }),
      declarationDoc({ status: "cancelled", cancelled_at: "2026-10-09T09:50:00-03:00" })
    ]);
    renderIt();
    const rows = (await screen.findAllByRole("row")).slice(1);
    expect(rows).toHaveLength(3);
    expect(within(rows[0]).getByText("Atestado")).not.toBeNull();
    expect(within(rows[0]).getByText("papel")).not.toBeNull();
    expect(within(rows[0]).getByText("emitido")).not.toBeNull();
    expect(within(rows[0]).getByText("K7Q2M9XW4P")).not.toBeNull();
    expect(within(rows[1]).getByText("digital")).not.toBeNull();
    expect(within(rows[1]).getByText("assinatura pendente")).not.toBeNull();
    expect(within(rows[1]).getByText("sem sessão de assinatura aberta")).not.toBeNull();
    expect(within(rows[2]).getByText("cancelado em 09/10/2026, 09:50")).not.toBeNull();
    expect(within(rows[2]).queryByRole("button")).toBeNull();
    expect(api.listConsultationDocuments).toHaveBeenCalledWith("cs1");
  });

  it("médica: oferece os quatro tipos e Renovar receita; enfermeira não emite atestado", async () => {
    const { unmount } = renderIt();
    const group = await screen.findByRole("group", { name: "emitir documento" });
    expect(within(group).getAllByRole("button").map((b) => b.textContent))
      .toEqual([ "Atestado", "Declaração", "Receita", "Requisição de exames", "Renovar receita" ]);
    unmount();
    renderIt({ cboCode: "223565", unitId: "u1" });
    const nurse = await screen.findByRole("group", { name: "emitir documento" });
    expect(within(nurse).getAllByRole("button").map((b) => b.textContent))
      .toEqual([ "Declaração", "Receita", "Requisição de exames", "Renovar receita" ]);
  });

  it("emite um atestado: a lista ganha o documento e a tela diz como ele sai", async () => {
    mocked(api.listConsultationDocuments).mockResolvedValue([]);
    mocked(api.issueDocument).mockResolvedValue(sickNoteDoc({ id: "doc7" }));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Atestado" }));
    fireEvent.change(screen.getByLabelText("Dias de afastamento"), { target: { value: "3" } });
    fireEvent.click(screen.getByRole("button", { name: "Emitir atestado" }));
    await waitFor(() => expect(api.issueDocument).toHaveBeenCalledWith("cs1",
      { kind: "sick_note", content: { type: "leave", days: 3, start_on: TODAY19C, cid_authorized: false } }, undefined));
    expect((await screen.findByRole("status")).textContent).toBe("Documento emitido (Atestado) em modo papel: imprima e assine à mão.");
    expect(screen.queryByRole("form", { name: "Atestado" })).toBeNull();
    expect(screen.getAllByRole("row")).toHaveLength(2);
  });

  it("Renovar receita: preenche com os contínuos, emite e manda reler os medicamentos em uso", async () => {
    mocked(api.getPrescriptionRenewal).mockResolvedValue([ prescriptionItem() ]);
    mocked(api.issueDocument).mockResolvedValue(prescriptionDoc({ id: "doc8" }));
    const { client } = renderIt();
    const invalidate = vi.spyOn(client, "invalidateQueries");
    fireEvent.click(await screen.findByRole("button", { name: "Renovar receita" }));
    expect((await screen.findByLabelText("Quantidade 1") as HTMLInputElement).value).toBe("60");
    fireEvent.click(screen.getByRole("button", { name: "Emitir receita" }));
    await waitFor(() => expect(api.issueDocument).toHaveBeenCalledWith("cs1", { kind: "prescription", content: { items: [
      { catalog_item: { id: "ci1" }, quantity: 60, quantity_unit: "comprimido", route: "oral",
        dosage_instructions: "1 comprimido após o café e 1 após o jantar", duration_days: 30, continuous: true }
    ] } }, undefined));
    await waitFor(() => expect(invalidate).toHaveBeenCalledWith({ queryKey: [ "patientMedications" ] }));
  });

  it("Renovar sem contínuos ativos: diz que não há o que renovar", async () => {
    mocked(api.getPrescriptionRenewal).mockResolvedValue([]);
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Renovar receita" }));
    expect((await screen.findByRole("status")).textContent).toBe("nenhum medicamento de uso contínuo ativo para renovar");
    expect(screen.queryByRole("form", { name: "Receita" })).toBeNull();
  });

  it("Imprimir abre o PDF numa janela nova, pelo id", async () => {
    mocked(api.fetchDocumentPdf).mockResolvedValue({ blob: new Blob([ "%PDF" ], { type: "application/pdf" }), filename: null });
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Imprimir Atestado de 09/10/2026, 09:40" }));
    expect(window.open).toHaveBeenCalledWith("", "_blank");
    await waitFor(() => expect(win.location.href).toBe("blob:doc1"));
    expect(api.fetchDocumentPdf).toHaveBeenCalledWith("doc1");
  });

  it("Cancelar pede o motivo (10 ou mais) com step-up e a lista mostra cancelado", async () => {
    mocked(api.cancelDocument).mockResolvedValue(cancelled());
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Cancelar Atestado de 09/10/2026, 09:40" }));
    const dialog = screen.getByRole("region", { name: "Cancelar: Atestado de 09/10/2026, 09:40" });
    fireEvent.change(within(dialog).getByLabelText("Motivo do cancelamento"), { target: { value: "curto" } });
    fireEvent.click(within(dialog).getByRole("button", { name: "Confirmar cancelamento" }));
    expect((await within(dialog).findByRole("alert")).textContent).toBe("descreva o motivo com pelo menos 10 caracteres");
    expect(api.cancelDocument).not.toHaveBeenCalled();
    fireEvent.change(within(dialog).getByLabelText("Motivo do cancelamento"), { target: { value: "  dias errados no atestado " } });
    fireEvent.click(within(dialog).getByRole("button", { name: "Confirmar cancelamento" }));
    await waitFor(() => expect(api.cancelDocument).toHaveBeenCalledWith("doc1", "dias errados no atestado"));
    expect(await screen.findByText("cancelado em 09/10/2026, 10:05")).not.toBeNull();
    expect(screen.getByRole("status").textContent).toBe("Documento cancelado.");
    expect(screen.queryByRole("form", { name: "Atestado" })).toBeNull();
  });

  it("receita: o cancelamento avisa que não desfaz medicamento entregue", async () => {
    mocked(api.listConsultationDocuments).mockResolvedValue([ prescriptionDoc() ]);
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Cancelar Receita de 09/10/2026, 09:40" }));
    expect(screen.getByText("cancelar não desfaz medicamento já entregue ao paciente")).not.toBeNull();
  });

  it("Cancelar e emitir outro: cancela, abre o atestado preenchido e o novo sai com replaces_document_id", async () => {
    mocked(api.cancelDocument).mockResolvedValue(cancelled());
    mocked(api.issueDocument).mockResolvedValue(sickNoteDoc({ id: "doc9", replaces_document_id: "doc1" }));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Cancelar e emitir outro: Atestado de 09/10/2026, 09:40" }));
    const dialog = screen.getByRole("region", { name: "Cancelar e emitir outro: Atestado de 09/10/2026, 09:40" });
    fireEvent.change(within(dialog).getByLabelText("Motivo do cancelamento"), { target: { value: "dias errados no atestado" } });
    fireEvent.click(within(dialog).getByRole("button", { name: "Cancelar e abrir o novo" }));
    const form = await screen.findByRole("form", { name: "Atestado" });
    expect(within(form).getByText("Substitui o documento cancelado.")).not.toBeNull();
    expect((within(form).getByLabelText("Dias de afastamento") as HTMLInputElement).value).toBe("3");
    fireEvent.change(within(form).getByLabelText("Dias de afastamento"), { target: { value: "5" } });
    fireEvent.click(within(form).getByRole("button", { name: "Emitir atestado" }));
    await waitFor(() => expect(api.issueDocument).toHaveBeenCalledWith("cs1",
      { kind: "sick_note", content: { type: "leave", days: 5, start_on: "2026-10-09", cid_authorized: false } }, "doc1"));
  });

  it("outra aba já cancelou (409 already_cancelled): diz, relê a lista e não abre o novo", async () => {
    mocked(api.cancelDocument).mockRejectedValue(new ApiError(409, { error: "already_cancelled" }, "409"));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Cancelar e emitir outro: Atestado de 09/10/2026, 09:40" }));
    fireEvent.change(screen.getByLabelText("Motivo do cancelamento"), { target: { value: "dias errados no atestado" } });
    fireEvent.click(screen.getByRole("button", { name: "Cancelar e abrir o novo" }));
    expect((await screen.findByRole("alert")).textContent).toBe("este documento já estava cancelado — a lista foi atualizada");
    await waitFor(() => expect(mocked(api.listConsultationDocuments).mock.calls.length).toBeGreaterThanOrEqual(2));
    expect(screen.queryByRole("form", { name: "Atestado" })).toBeNull();
  });

  it("interruptor desligado entre a leitura e o clique: a frase diz e o formulário fica", async () => {
    mocked(api.issueDocument).mockRejectedValue(new ApiError(403, { error: "feature_disabled", feature: "clinical_documents" }, "403"));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Atestado" }));
    fireEvent.change(screen.getByLabelText("Dias de afastamento"), { target: { value: "3" } });
    fireEvent.click(screen.getByRole("button", { name: "Emitir atestado" }));
    expect((await screen.findByRole("alert")).textContent).toBe(DOCUMENTS_DISABLED);
    expect((screen.getByLabelText("Dias de afastamento") as HTMLInputElement).value).toBe("3");
  });

  it("Minhas consultas (sem issue): só a lista, Imprimir e Cancelar", async () => {
    renderIt(undefined);
    expect(await screen.findByRole("button", { name: "Imprimir Atestado de 09/10/2026, 09:40" })).not.toBeNull();
    expect(screen.getByRole("button", { name: "Cancelar Atestado de 09/10/2026, 09:40" })).not.toBeNull();
    expect(screen.queryByRole("group", { name: "emitir documento" })).toBeNull();
    expect(screen.queryByRole("button", { name: /Cancelar e emitir outro/ })).toBeNull();
  });

  it("documento de outra autora: sem Cancelar", async () => {
    mocked(api.listConsultationDocuments).mockResolvedValue([ sickNoteDoc({ author: { id: "us9", name: "Dr. Outro", council: null, cbo_code: "225142" } }) ]);
    renderIt();
    expect(await screen.findByRole("button", { name: "Imprimir Atestado de 09/10/2026, 09:40" })).not.toBeNull();
    expect(screen.queryByRole("button", { name: /^Cancelar/ })).toBeNull();
  });
});
```

Em `src/lib/signature.test.ts`, no teste que confere `documentLabel` (o de `signatureFileName`), acrescente:

```ts
    expect(documentLabel("clinical_document")).toBe("documento clínico");
```

Em `src/modules/consultation/ConsultationWorkspace.documents.test.tsx` (Task 3), acrescente o dublê da aba e os testes. Junto dos outros `vi.mock`:

```tsx
vi.mock("../documents/ConsultationDocuments", () => ({
  ConsultationDocuments: (p: { consultationId: string; issue?: { cboCode: string; unitId?: string } }) => (
    <span>{`documentos ${p.consultationId} · ${p.issue ? `emissão ${p.issue.cboCode} ${p.issue.unitId}` : "só lista"}`}</span>
  )
}));
```

e no `describe`:

```tsx
  it("abas Consulta e Documentos depois de iniciar; o rascunho fica montado ao trocar de aba", async () => {
    renderIt();
    expect(screen.queryByRole("tab", { name: "Documentos" })).toBeNull();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta" }));
    fireEvent.click(await screen.findByRole("tab", { name: "Documentos" }));
    expect(screen.getByText("documentos cs1 · emissão 225142 u1")).not.toBeNull();
    expect(screen.getByText("editor cs1").closest("[hidden]")).not.toBeNull();
    fireEvent.click(screen.getByRole("tab", { name: "Consulta" }));
    expect(screen.getByText("editor cs1").closest("[hidden]")).toBeNull();
    expect(screen.queryByText(/^documentos cs1/)).toBeNull();
  });

  it("consulta de outra autora: Documentos só com a lista", async () => {
    mocked(api.startConsultation).mockResolvedValue(consultation({ author: { id: "us9", name: "Dr. Outro" } }));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta" }));
    fireEvent.click(await screen.findByRole("tab", { name: "Documentos" }));
    expect(screen.getByText("documentos cs1 · só lista")).not.toBeNull();
  });

  it("sem clinical_documents: sem abas", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(docUser([ "health_professional" ], { features: [ "clinical_record" ] }));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta" }));
    expect(await screen.findByText("editor cs1")).not.toBeNull();
    expect(screen.queryByRole("tab")).toBeNull();
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/documents/ConsultationDocuments.test.tsx src/lib/signature.test.ts src/modules/consultation/ConsultationWorkspace.documents.test.tsx`
Expected: FAIL — `./ConsultationDocuments` não existe, `documentLabel("clinical_document")` devolve a chave crua e a consulta não tem abas.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/modules/documents/printDocument.ts
// Imprimir um documento clínico (módulo 19c; contrato §3): o api devolve o PDF
// assinado (digital e assinado) ou o do modo papel. Lido com a sessão e aberto
// numa janela nova por um endereço `blob:` — nada do conteúdo passa pela URL
// da aplicação. A janela abre NO CLIQUE (antes de qualquer await), senão o
// navegador a trata como pop-up.
import { fetchDocumentPdf } from "../../lib/api";

const PDF_URL_TTL_MS = 60_000;
export const POPUP_BLOCKED = "o navegador bloqueou a janela nova — permita janelas deste site para imprimir";

export async function printDocument(id: string, errorText: (err: unknown) => string): Promise<string | null> {
  const win = window.open("", "_blank");
  try {
    const { blob } = await fetchDocumentPdf(id);
    if (!win) return POPUP_BLOCKED;
    const url = URL.createObjectURL(blob);
    win.location.href = url;
    setTimeout(() => URL.revokeObjectURL(url), PDF_URL_TTL_MS);
    return null;
  } catch (err) {
    win?.close();
    return errorText(err);
  }
}
```

```tsx
// src/modules/documents/CancelDocument.tsx
// Cancelar um documento clínico (módulo 19c; spec §6; contrato §3): só a
// autora, motivo com 10 ou mais caracteres, step-up (o SensitiveAction é o
// único que conhece o código). O documento fica no histórico como cancelado e
// a página de conferência passa a dizer "cancelado"; a assinatura pendente sai
// da fila no api. 409 already_cancelled = outra aba já cancelou: quem chama relê.
import { useRef } from "react";
import { ApiError, cancelDocument, errorCode, type ClinicalDocument } from "../../lib/api";
import { CANCEL_WARNING, cancelReasonProblem, documentError, kindLabel } from "../../lib/clinicalDocuments";
import { fmtDateTime } from "../../lib/format";
import { SensitiveAction } from "../../components/SensitiveAction";

interface Props {
  doc: ClinicalDocument;
  reissue: boolean;
  onDone(cancelled: ClinicalDocument): void;
  onStale(): void;
  onCancel(): void;
  onGoToSecurity?(): void;
}

export function CancelDocument({ doc, reissue, onDone, onStale, onCancel, onGoToSecurity }: Props) {
  const result = useRef<ClinicalDocument | null>(null);
  return (
    <SensitiveAction
      title={`${reissue ? "Cancelar e emitir outro" : "Cancelar"}: ${kindLabel(doc.kind)} de ${fmtDateTime(doc.issued_at)}`}
      description={(
        <>
          <p style={text}>
            O documento fica no histórico como cancelado e a página de conferência passa a dizer "cancelado". Se a assinatura digital
            ainda estiver na fila, ela sai da fila.
          </p>
          {doc.kind === "prescription" && <p style={{ ...text, fontWeight: 600 }}>{CANCEL_WARNING}</p>}
          {reissue && <p style={text}>Depois do cancelamento, um documento novo abre preenchido com os mesmos dados, para você conferir e emitir.</p>}
        </>
      )}
      requiresStepUp
      fields={[ { name: "reason", label: "Motivo do cancelamento", required: true } ]}
      confirmLabel={reissue ? "Cancelar e abrir o novo" : "Confirmar cancelamento"}
      run={async (values) => {
        const reason = values.reason.trim();
        // Espelho do 422 invalid_reason do api: nada vai ao servidor com motivo curto.
        if (cancelReasonProblem(reason)) throw new ApiError(422, { error: "invalid_reason" }, "local");
        result.current = await cancelDocument(doc.id, reason);
      }}
      onDone={() => { if (result.current) onDone(result.current); }}
      onCancel={onCancel}
      onGoToSecurity={onGoToSecurity}
      translateError={(err) => {
        const code = errorCode(err);
        if (code === "already_cancelled") onStale();
        return code ? documentError(err) : null;
      }}
    />
  );
}

const text = { margin: 0 };
```

```tsx
// src/modules/documents/ConsultationDocuments.tsx
// Documentos da consulta (módulo 19c; spec §7 "Consulta" e "Minhas consultas";
// contrato §3–§4): lista com modo, assinatura, situação e código; emitir os
// quatro tipos (pelo CBO da autora; o api confere), "Renovar receita",
// Imprimir, Cancelar e "Cancelar e emitir outro". Sem `issue` (Minhas
// consultas), só a lista. Leitura com gcTime 0; com assinatura pendente, a
// lista relê a cada 15 s até assentar.
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { getPrescriptionRenewal, issueDocument, listConsultationDocuments, type ClinicalDocument, type DocumentInput, type DocumentKind } from "../../lib/api";
import { useAuth } from "../../lib/auth";
import { todayInCity } from "../../lib/campaigns";
import { fmtDateTime } from "../../lib/format";
import {
  DOCUMENTS_KEY, MEDICATIONS_KEY, canCancel, documentError, documentSituation, documentsRefetchInterval, emptyDeclaration, emptySickNote,
  issueModeLabel, issuedNotice, itemFromPrescription, kindLabel, kindsFor, reissueDraft, withDocument, type ReissueDraft
} from "../../lib/clinicalDocuments";
import { DataTable, type Column } from "../../components/DataTable";
import { buttonStyle, disabledButtonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { SignatureMarker } from "../signature/SignatureMarker";
import { SickNoteForm } from "./SickNoteForm";
import { DeclarationForm } from "./DeclarationForm";
import { PrescriptionForm } from "./PrescriptionForm";
import { ExamRequisitionForm } from "./ExamRequisitionForm";
import { CancelDocument } from "./CancelDocument";
import { printDocument } from "./printDocument";

export interface IssueContext { cboCode: string; unitId?: string }

interface Props {
  consultationId: string;
  // Sem `issue`: só a lista (Minhas consultas) — Imprimir e Cancelar.
  issue?: IssueContext;
  searchDelayMs?: number;
  onGoToSecurity?(): void;
}

type OpenForm = ReissueDraft & { replaces?: string };

const START_LABEL: Record<DocumentKind, string> = {
  sick_note: "Atestado", attendance_declaration: "Declaração", prescription: "Receita", exam_requisition: "Requisição de exames"
};

export function ConsultationDocuments({ consultationId, issue, searchDelayMs, onGoToSecurity }: Props) {
  const { user } = useAuth();
  const queryClient = useQueryClient();
  const key = [ DOCUMENTS_KEY, consultationId ];
  const query = useQuery({
    queryKey: key, queryFn: () => listConsultationDocuments(consultationId), gcTime: 0, retry: false,
    refetchInterval: (q) => documentsRefetchInterval(q.state.data)
  });
  const [ form, setForm ] = useState<OpenForm | null>(null);
  const [ cancelling, setCancelling ] = useState<{ doc: ClinicalDocument; reissue: boolean } | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const [ renewing, setRenewing ] = useState(false);
  const [ printing, setPrinting ] = useState<string | null>(null);
  const today = todayInCity();
  const kinds = issue ? kindsFor(issue.cboCode) : [];

  function open(next: OpenForm) {
    setNotice(null); setError(null); setCancelling(null); setForm(next);
  }

  function start(kind: DocumentKind) {
    if (kind === "sick_note") open({ kind, draft: emptySickNote(today) });
    else if (kind === "attendance_declaration") open({ kind, draft: emptyDeclaration(today) });
    else if (kind === "prescription") open({ kind, draft: { items: [], protocolVersionId: "" } });
    else open({ kind, draft: { note: "" } });
  }

  async function renew() {
    if (renewing) return;
    setRenewing(true); setNotice(null); setError(null);
    try {
      const items = await getPrescriptionRenewal(consultationId);
      if (items.length === 0) { setForm(null); setNotice("nenhum medicamento de uso contínuo ativo para renovar"); return; }
      open({ kind: "prescription", draft: { items: items.map(itemFromPrescription), protocolVersionId: "" } });
    } catch (err) {
      setError(documentError(err));
    } finally {
      setRenewing(false);
    }
  }

  // A recusa sobe para o formulário, que a mostra e fica preenchido.
  async function issueIt(input: DocumentInput) {
    const doc = await issueDocument(consultationId, input, form?.replaces);
    queryClient.setQueryData<ClinicalDocument[]>(key, (list) => withDocument(list ?? [], doc));
    if (doc.kind === "prescription") void queryClient.invalidateQueries({ queryKey: [ MEDICATIONS_KEY ] });
    setForm(null);
    setNotice(issuedNotice(doc));
  }

  async function print(doc: ClinicalDocument) {
    if (printing) return;
    setPrinting(doc.id); setError(null);
    const problem = await printDocument(doc.id, documentError);
    setPrinting(null);
    if (problem) setError(problem);
  }

  function askCancel(doc: ClinicalDocument, reissue: boolean) {
    setForm(null); setNotice(null); setError(null);
    setCancelling({ doc, reissue });
  }

  const what = (d: ClinicalDocument) => `${kindLabel(d.kind)} de ${fmtDateTime(d.issued_at)}`;
  const cols: Column<ClinicalDocument>[] = [
    { label: "Documento", w: "1.4fr", render: (d) => `${kindLabel(d.kind)}${d.replaces_document_id ? " · substitui um cancelado" : ""}` },
    { label: "Emitido em", w: "1fr", render: (d) => fmtDateTime(d.issued_at) },
    { label: "Modo", w: "0.6fr", render: (d) => issueModeLabel(d.issue_mode) },
    { label: "Assinatura", w: "1.6fr", render: (d) => d.signature
      ? <SignatureMarker label={`assinatura: ${kindLabel(d.kind)}`} block={d.signature} consultationId={consultationId} />
      : "—" },
    { label: "Situação", w: "1fr", render: (d) => documentSituation(d) },
    { label: "Código", w: "1fr", render: (d) => <span className="mono">{d.short_code}</span> },
    { label: "", w: "auto", align: "right", render: (d) => (
      <span style={actions}>
        {d.status === "issued" && (
          <button type="button" aria-label={`Imprimir ${what(d)}`} disabled={printing === d.id}
            style={printing === d.id ? disabledButtonStyle : secondaryButtonStyle} onClick={() => void print(d)}>Imprimir</button>
        )}
        {canCancel(d, user?.id) && (
          <button type="button" aria-label={`Cancelar ${what(d)}`} style={secondaryButtonStyle} onClick={() => askCancel(d, false)}>Cancelar</button>
        )}
        {issue && canCancel(d, user?.id) && (
          <button type="button" aria-label={`Cancelar e emitir outro: ${what(d)}`} style={secondaryButtonStyle}
            onClick={() => askCancel(d, true)}>Cancelar e emitir outro</button>
        )}
      </span>
    ) }
  ];

  return (
    <section aria-label="Documentos da consulta" style={panel}>
      {issue && (
        <div role="group" aria-label="emitir documento" style={row}>
          {kinds.map((k) => (
            <button key={k} type="button" style={secondaryButtonStyle} onClick={() => start(k)}>{START_LABEL[k]}</button>
          ))}
          {kinds.includes("prescription") && (
            <button type="button" disabled={renewing} style={renewing ? disabledButtonStyle : buttonStyle} onClick={() => void renew()}>
              Renovar receita
            </button>
          )}
        </div>
      )}
      {notice && <p role="status" style={statusStyle}>{notice}</p>}
      {error && <p role="alert" style={alert}>{error}</p>}

      {issue && form?.kind === "sick_note" && (
        <SickNoteForm initial={form.draft} replacing={!!form.replaces} searchDelayMs={searchDelayMs} onCancel={() => setForm(null)}
          onSubmit={(content) => issueIt({ kind: "sick_note", content })} />
      )}
      {issue && form?.kind === "attendance_declaration" && (
        <DeclarationForm initial={form.draft} replacing={!!form.replaces} onCancel={() => setForm(null)}
          onSubmit={(content) => issueIt({ kind: "attendance_declaration", content })} />
      )}
      {issue && form?.kind === "prescription" && (
        <PrescriptionForm cboCode={issue.cboCode} unitId={issue.unitId} today={today} initial={form.draft} replacing={!!form.replaces}
          searchDelayMs={searchDelayMs} onCancel={() => setForm(null)} onSubmit={(content) => issueIt({ kind: "prescription", content })} />
      )}
      {issue && form?.kind === "exam_requisition" && (
        <ExamRequisitionForm initialNote={form.draft.note} replacing={!!form.replaces} onCancel={() => setForm(null)}
          onSubmit={(content) => issueIt({ kind: "exam_requisition", content })} />
      )}

      {cancelling && (
        <CancelDocument
          key={`${cancelling.doc.id}:${cancelling.reissue}`}
          doc={cancelling.doc}
          reissue={cancelling.reissue}
          onGoToSecurity={onGoToSecurity}
          onDone={(cancelledDoc) => {
            queryClient.setQueryData<ClinicalDocument[]>(key, (list) => withDocument(list ?? [], cancelledDoc));
            const next = cancelling.reissue ? reissueDraft(cancelledDoc) : null;
            setCancelling(null);
            setNotice(next ? "Documento cancelado. Confira os dados e emita o novo." : "Documento cancelado.");
            if (next) setForm({ ...next, replaces: cancelledDoc.id });
          }}
          onStale={() => void queryClient.invalidateQueries({ queryKey: key })}
          onCancel={() => setCancelling(null)}
        />
      )}

      {query.isPending && <p className="mono" style={muted}>carregando os documentos…</p>}
      {query.isError && <p role="alert" style={alert}>{documentError(query.error)}</p>}
      {query.isSuccess && (
        <DataTable cols={cols} rows={query.data} rowKey={(d) => d.id} empty="nenhum documento emitido nesta consulta" />
      )}
    </section>
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 10 };
const row: CSSProperties = { display: "flex", gap: 8, flexWrap: "wrap" };
const actions: CSSProperties = { display: "flex", gap: 6, justifyContent: "flex-end", flexWrap: "wrap" };
const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
const muted: CSSProperties = { margin: 0, fontSize: 10.5, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

Em `src/lib/api.ts`:

```ts
export type SignatureDocumentType = "consultation" | "consultation_addendum" | "clinical_document";
```

Em `src/lib/signature.ts`:

```ts
const DOCUMENT_LABEL: Record<string, string> = {
  consultation: "consulta", consultation_addendum: "adendo", clinical_document: "documento clínico"
};
```

Em `src/modules/consultation/ConsultationWorkspace.tsx`:

```tsx
import { SegmentedControl } from "../../shell/SegmentedControl";
import { ConsultationDocuments } from "../documents/ConsultationDocuments";
```

depois de `const documentsOn = canUseDocuments(user);` (Task 3):

```tsx
  // Aba Documentos: o editor continua montado (escondido) para o autosave não
  // perder o que foi digitado; os documentos só montam quando a aba abre.
  const [ tab, setTab ] = useState<"consultation" | "documents">("consultation");
  const showTabs = documentsOn && !!consultation;
```

e troque o trecho que desenha o editor e a leitura (de `{consultation?.status === "draft" && options.isError && …}` até o fim do bloco `{consultation?.status === "finalized" && (…)}`) por:

```tsx
          {showTabs && (
            <SegmentedControl
              options={[ { key: "consultation", label: "Consulta" }, { key: "documents", label: "Documentos" } ]}
              value={tab} onChange={setTab} />
          )}

          <div hidden={showTabs && tab === "documents"} style={showTabs && tab === "documents" ? undefined : { display: "contents" }}>
            {consultation?.status === "draft" && options.isError && <p role="alert" style={alert}>{consultationError(options.error)}</p>}
            {consultation?.status === "draft" && options.data && (
              <ConsultationEditor key={consultation.id} consultation={consultation} record={record.data} options={options.data}
                referenceUnitIds={props.referenceUnitIds} unit={props.unit} units={props.units}
                autosaveDelayMs={props.autosaveDelayMs} searchDelayMs={props.searchDelayMs}
                onFinalized={(c) => { setConsultation(c); restartRereads(); props.onFinalized(); }}
                onLocked={(message) => void reload(message)} />
            )}

            {consultation?.status === "finalized" && (
              <ConsultationView consultation={consultation} options={options.data ?? null} patientProblems={record.data.problems}
                searchDelayMs={props.searchDelayMs}
                onAddendumAdded={(addendum) => { setConsultation((c) => (c ? withAddendum(c, addendum) : c)); restartRereads(); }}
                onClose={props.onClose} />
            )}
          </div>

          {showTabs && tab === "documents" && consultation && (
            <ConsultationDocuments consultationId={consultation.id} searchDelayMs={props.searchDelayMs}
              issue={user?.id === consultation.author.id ? { cboCode: consultation.cbo_code, unitId: props.unit.id } : undefined} />
          )}
```

(`display: contents` não deixa o invólucro mexer no `gap` do painel. O estilo só vale quando a aba Consulta está aberta: um `display` no `style` venceria o `hidden` e o editor apareceria por baixo da aba Documentos.)

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/documents/ src/modules/consultation/ src/lib/signature.test.ts && npx tsc --noEmit`
Expected: PASS (os testes antigos da consulta continuam verdes: sem `clinical_documents` na sessão deles, não há abas) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19c
/opt/homebrew/bin/git add src/modules/documents/ConsultationDocuments.tsx src/modules/documents/ConsultationDocuments.test.tsx \
  src/modules/documents/CancelDocument.tsx src/modules/documents/printDocument.ts src/lib/api.ts src/lib/signature.ts src/lib/signature.test.ts \
  src/modules/consultation/ConsultationWorkspace.tsx src/modules/consultation/ConsultationWorkspace.documents.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the documents tab to the consultation with issue, print, cancel and reissue

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Documentos em "Minhas consultas"

**Files:**
- Modify: `src/modules/MyConsultations.tsx`
- Test: `src/modules/MyConsultations.documents.test.tsx`

**Interfaces:**
- Consumes: `ConsultationDocuments` (Task 6, sem `issue`); `canUseDocuments` (Task 2).
- Produces: ao abrir uma consulta em "Minhas consultas", a autora vê, logo abaixo da leitura, os documentos dela (Imprimir e Cancelar), quando a sessão tem `clinical_documents`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/MyConsultations.documents.test.tsx
// Módulo 19c: documentos da consulta aberta em "Minhas consultas" (reimprimir
// e cancelar). A leitura da consulta e a lista de documentos são dublês.
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listMyConsultations: vi.fn(), getConsultationOptions: vi.fn() };
});
vi.mock("./consultation/ConsultationView", () => ({
  ConsultationLoader: (p: { id: string }) => <span>{`consulta ${p.id}`}</span>
}));
vi.mock("./documents/ConsultationDocuments", () => ({
  ConsultationDocuments: (p: { consultationId: string; issue?: unknown }) => (
    <span>{`documentos ${p.consultationId} · ${p.issue ? "com emissão" : "só lista"}`}</span>
  )
}));

import * as api from "../lib/api";
import { MyConsultations } from "./MyConsultations";
import { renderWithProviders } from "../test/campaignFixtures";
import { consultationListItem, options } from "../test/consultationFixtures";
import { docUser } from "../test/documentFixtures";

afterEach(cleanup);
const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("Minhas consultas — documentos (19c)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listMyConsultations, api.getConsultationOptions ]) m(fn).mockReset();
    m(api.fetchCurrentSession).mockResolvedValue(docUser());
    m(api.listMyConsultations).mockResolvedValue([ consultationListItem() ]);
    m(api.getConsultationOptions).mockResolvedValue(options());
  });

  it("abrir a consulta mostra os documentos dela, só a lista", async () => {
    renderWithProviders(<MyConsultations />);
    fireEvent.click(await screen.findByRole("button", { name: "Abrir consulta de 06/10/2026, 14:30" }));
    expect(screen.getByText("consulta c1")).not.toBeNull();
    expect(screen.getByText("documentos c1 · só lista")).not.toBeNull();
  });

  it("sem clinical_documents, só a consulta", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(docUser([ "health_professional" ], { features: [ "clinical_record" ] }));
    renderWithProviders(<MyConsultations />);
    fireEvent.click(await screen.findByRole("button", { name: "Abrir consulta de 06/10/2026, 14:30" }));
    expect(screen.getByText("consulta c1")).not.toBeNull();
    expect(screen.queryByText(/^documentos /)).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/MyConsultations.documents.test.tsx`
Expected: FAIL — "documentos c1 · só lista" não aparece.

- [ ] **Step 3: Write minimal implementation**

Em `src/modules/MyConsultations.tsx`:

```tsx
import { canUseDocuments } from "../lib/clinicalDocuments";
import { ConsultationDocuments } from "./documents/ConsultationDocuments";
```

e troque o ramo `viewing ? (…)`:

```tsx
      {viewing ? (
        <>
          <ConsultationLoader key={viewing} id={viewing} options={options.data ?? null} patientProblems={[]}
            onClose={() => setViewing(null)} />
          {/* Módulo 19c: reimprimir e cancelar os documentos desta consulta. */}
          {canUseDocuments(user) && <ConsultationDocuments key={`documents:${viewing}`} consultationId={viewing} />}
        </>
      ) : (
```

(o ramo `: (<Panel title="Consultas finalizadas" …>)` continua igual.)

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/MyConsultations.documents.test.tsx src/modules/MyConsultations.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes antigos de "Minhas consultas" continuam verdes: as sessões deles não têm `clinical_documents`) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19c
/opt/homebrew/bin/git add src/modules/MyConsultations.tsx src/modules/MyConsultations.documents.test.tsx
/opt/homebrew/bin/git commit -m "feat: list the consultation documents in my consultations for reprint and cancel

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Declaração de comparecimento na recepção

**Files:**
- Create: `src/modules/attendance/DeclarationPanel.tsx`
- Modify: `src/modules/attendance/UnitQueue.tsx`, `src/modules/Attendance.tsx`
- Test: `src/modules/attendance/UnitQueue.declaration.test.tsx`

**Interfaces:**
- Consumes: `issueAttendanceDeclaration`, `QueueRow`, `ClinicalDocument` (Task 1); `emptyDeclaration`, `issuedNotice`, `documentError` (Task 2); `DeclarationForm` (Task 4); `printDocument` (Task 6); `todayInCity`, `fmtHourMinute`.
- Produces:
  - `DeclarationPanel({ row: QueueRow, onClose() })` — `<section aria-label="Declaração de comparecimento do atendimento">` com o `DeclarationForm` (hora de chegada já preenchida pelo check-in; matrícula de quem emite), e, depois de emitida, o aviso do api, o código de conferência e "Imprimir";
  - `UnitQueue` ganha a prop `clinicalDocuments?: boolean`: com ela, um botão "Declaração" em cada linha de "Aguardando" e de "Em atendimento", para a recepção e para os profissionais; o painel fica preso ao atendimento escolhido, mesmo que a linha saia da fila;
  - `Attendance` passa `clinicalDocuments={hasFeature(user, "clinical_documents")}`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/attendance/UnitQueue.declaration.test.tsx
// Módulo 19c: declaração de comparecimento emitida pela recepção, pelo atendimento.
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listUnitQueue: vi.fn(), issueAttendanceDeclaration: vi.fn(), fetchDocumentPdf: vi.fn() };
});
vi.mock("../consultation/ConsultationWorkspace", () => ({ ConsultationWorkspace: () => null }));

import * as api from "../../lib/api";
import { ApiError, type QueueRow } from "../../lib/api";
import { AuthProvider } from "../../lib/auth";
import { UnitQueue } from "./UnitQueue";
import { NOW19C, declarationDoc, docUser } from "../../test/documentFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };
let win: { location: { href: string }; close: ReturnType<typeof vi.fn> };

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });

function row(over: Partial<QueueRow>): QueueRow {
  return { id: "a1", cpf_masked: "***.982.247-**", checked_in_at: "2026-10-09T09:20:00-03:00", protocol_name: null, priority: null,
    source: "triage", appointment_time: null, called_at: null, called_by_name: null, screening: null, display_name: "Joana Lima", ...over };
}
const waitingRow = row({});
const inCareRow = row({ id: "a3", display_name: "Carlos Souza", called_at: "2026-10-09T09:50:00-03:00", called_by_name: "Dra. Helena" });

function renderQueue(clinicalDocuments = true) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  render(<UnitQueue unit={unit} units={[ unit ]} canCare={false} clinicalRecord clinicalDocuments={clinicalDocuments} />, { wrapper });
  return client;
}
const section = async (title: string) => (await screen.findByText(title)).parentElement as HTMLElement;

describe("UnitQueue — declaração de comparecimento (19c)", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW19C));
    for (const fn of [ api.fetchCurrentSession, api.listUnitQueue, api.issueAttendanceDeclaration, api.fetchDocumentPdf ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(docUser([ "citizen_verifier" ], { id: "us5", email_address: "recepcao@curitiba.demo" }));
    mocked(api.listUnitQueue).mockResolvedValue({ waiting: [ waitingRow ], in_care: [ inCareRow ] });
    mocked(api.issueAttendanceDeclaration).mockResolvedValue(declarationDoc({ id: "doc3", consultation_id: null, attendance_id: "a1" }));
    mocked(api.fetchDocumentPdf).mockResolvedValue({ blob: new Blob([ "%PDF" ], { type: "application/pdf" }), filename: null });
    win = { location: { href: "" }, close: vi.fn() };
    vi.spyOn(window, "open").mockImplementation(() => win as unknown as Window);
    Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:doc3") });
    Object.defineProperty(URL, "revokeObjectURL", { configurable: true, value: vi.fn() });
  });

  it("recepção: Declaração nas duas listas; emite pelo atendimento com a chegada e a matrícula, e imprime", async () => {
    renderQueue();
    const inCare = await section("Em atendimento");
    expect(within(inCare).getByRole("button", { name: "Declaração" })).not.toBeNull();
    expect(within(inCare).queryByRole("button", { name: "Encerrar" })).toBeNull();
    const waiting = await section("Aguardando");
    fireEvent.click(within(waiting).getByRole("button", { name: "Declaração" }));
    const panel = screen.getByRole("region", { name: "Declaração de comparecimento do atendimento" });
    expect(within(panel).getByText("Joana Lima · chegou às 09:20")).not.toBeNull();
    expect((within(panel).getByLabelText("Chegada") as HTMLInputElement).value).toBe("09:20");
    fireEvent.change(within(panel).getByLabelText("Sua matrícula (sai no papel)"), { target: { value: "4521" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Emitir declaração" }));
    await waitFor(() => expect(api.issueAttendanceDeclaration).toHaveBeenCalledWith("a1",
      { arrived_at: "09:20", issuer_registration: "4521" }));
    expect((await within(panel).findByRole("status")).textContent)
      .toBe("Documento emitido (Declaração de comparecimento) em modo papel: imprima e assine à mão.");
    expect(within(panel).getByText("D9W4N7KC3M")).not.toBeNull();
    fireEvent.click(within(panel).getByRole("button", { name: "Imprimir" }));
    await waitFor(() => expect(win.location.href).toBe("blob:doc3"));
    expect(api.fetchDocumentPdf).toHaveBeenCalledWith("doc3");
  });

  it("a pessoa sai da fila enquanto a recepção preenche: o painel continua e emite", async () => {
    const client = renderQueue();
    fireEvent.click(within(await section("Aguardando")).getByRole("button", { name: "Declaração" }));
    mocked(api.listUnitQueue).mockResolvedValue({ waiting: [], in_care: [ inCareRow ] });
    await client.invalidateQueries({ queryKey: [ "unitQueue", "u1" ] });
    expect(await screen.findByText("Ninguém aguardando")).not.toBeNull();
    const panel = screen.getByRole("region", { name: "Declaração de comparecimento do atendimento" });
    fireEvent.click(within(panel).getByRole("button", { name: "Emitir declaração" }));
    await waitFor(() => expect(api.issueAttendanceDeclaration).toHaveBeenCalledWith("a1", { arrived_at: "09:20" }));
  });

  it("atendimento de mais de 30 dias: a frase diz e o formulário fica", async () => {
    mocked(api.issueAttendanceDeclaration).mockRejectedValue(new ApiError(409, { error: "attendance_not_found_or_closed_long_ago" }, "409"));
    renderQueue();
    fireEvent.click(within(await section("Aguardando")).getByRole("button", { name: "Declaração" }));
    fireEvent.click(screen.getByRole("button", { name: "Emitir declaração" }));
    expect((await screen.findByRole("alert")).textContent)
      .toBe("atendimento não encontrado ou encerrado há mais de 30 dias — a declaração não sai mais por aqui");
    expect(screen.getByRole("form", { name: "Declaração de comparecimento" })).not.toBeNull();
  });

  it("sem clinical_documents: sem Declaração", async () => {
    renderQueue(false);
    const waiting = await section("Aguardando");
    expect(within(waiting).queryByRole("button", { name: "Declaração" })).toBeNull();
    expect(within(await section("Em atendimento")).queryByRole("button", { name: "Declaração" })).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/attendance/UnitQueue.declaration.test.tsx`
Expected: FAIL — não há botão "Declaração" (e o `tsc` acusaria a prop `clinicalDocuments` desconhecida).

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/attendance/DeclarationPanel.tsx
// Declaração de comparecimento pela recepção (módulo 19c; spec §4 e §7
// "Recepção"; contrato §3): pelo atendimento, com a hora de chegada do
// check-in já preenchida e a matrícula de quem emite (sai no papel). Sai em
// papel (a recepção não assina); o api recusa atendimento de mais de 30 dias.
// O painel guarda a linha escolhida: a pessoa pode sair da fila enquanto a
// recepção preenche, e a declaração ainda sai.
import { useState, type CSSProperties } from "react";
import { issueAttendanceDeclaration, type ClinicalDocument, type QueueRow } from "../../lib/api";
import { documentError, emptyDeclaration, issuedNotice } from "../../lib/clinicalDocuments";
import { todayInCity } from "../../lib/campaigns";
import { fmtHourMinute } from "../../lib/format";
import { buttonStyle, disabledButtonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { DeclarationForm } from "../documents/DeclarationForm";
import { printDocument } from "../documents/printDocument";

export function DeclarationPanel({ row, onClose }: { row: QueueRow; onClose(): void }) {
  const [ issued, setIssued ] = useState<ClinicalDocument | null>(null);
  const [ printing, setPrinting ] = useState(false);
  const [ printError, setPrintError ] = useState<string | null>(null);
  const [ initial ] = useState(() => emptyDeclaration(todayInCity(), fmtHourMinute(row.checked_in_at)));

  async function print(doc: ClinicalDocument) {
    if (printing) return;
    setPrinting(true); setPrintError(null);
    const problem = await printDocument(doc.id, documentError);
    setPrinting(false);
    if (problem) setPrintError(problem);
  }

  return (
    <section aria-label="Declaração de comparecimento do atendimento" style={panel}>
      <p className="mono" style={muted}>{`${row.display_name ?? row.cpf_masked} · chegou às ${fmtHourMinute(row.checked_in_at)}`}</p>
      {issued ? (
        <>
          <p role="status" style={{ margin: 0, fontSize: 12.5, fontWeight: 600 }}>{issuedNotice(issued)}</p>
          <p style={muted}>Código de conferência: <span className="mono">{issued.short_code}</span></p>
          {printError && <p role="alert" style={alert}>{printError}</p>}
          <div style={{ display: "flex", gap: 8 }}>
            <button type="button" disabled={printing} style={printing ? disabledButtonStyle : buttonStyle} onClick={() => void print(issued)}>
              Imprimir
            </button>
            <button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar</button>
          </div>
        </>
      ) : (
        <DeclarationForm initial={initial} showIssuerRegistration onCancel={onClose}
          onSubmit={async (content) => { setIssued(await issueAttendanceDeclaration(row.id, content)); }} />
      )}
    </section>
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 };
const muted: CSSProperties = { margin: 0, fontSize: 11, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

Em `src/modules/attendance/UnitQueue.tsx`:

```tsx
import { DeclarationPanel } from "./DeclarationPanel";
```

Na interface `Props`, depois de `clinicalRecord?: boolean;`:

```tsx
  // Módulo 19c: `clinical_documents` ligado na sessão (declaração de comparecimento).
  clinicalDocuments?: boolean;
```

Na assinatura do componente:

```tsx
export function UnitQueue({
  unit, units, canCare, careBlocked, onClinicalRefused, now = () => new Date(), clinicalRecord = false, clinicalDocuments = false
}: Props) {
```

Depois de `const [ panel, setPanel ] = useState<ScreeningPanel | null>(null);`:

```tsx
  // Módulo 19c: a linha guardada aqui segura o painel mesmo que a pessoa saia da fila.
  const [ declaring, setDeclaring ] = useState<QueueRow | null>(null);
  const declarationButton = (r: QueueRow) => clinicalDocuments && (
    <button type="button" style={secondaryButtonStyle} onClick={() => setDeclaring(r)}>Declaração</button>
  );
```

Em "Aguardando", no `<div style={{ display: "flex", gap: 8, justifyContent: "flex-end" }}>` das ações, logo antes do botão "Saiu sem atendimento":

```tsx
                          {declarationButton(r)}
```

Em "Em atendimento", troque a coluna de ações (`...(canCare ? [ { … } ] : [])`) por:

```tsx
                    ...(canCare || clinicalDocuments ? [ {
                      label: "", w: "auto" as const, align: "right" as const, render: (r: QueueRow) => (
                        <div style={{ display: "flex", gap: 8, justifyContent: "flex-end" }}>
                          {declarationButton(r)}
                          {canCare && clinicalRecord && (
                            <button type="button" style={secondaryButtonStyle}
                              onClick={() => setPanel({ kind: "consultation", attendanceId: r.id })}>
                              Consulta
                            </button>
                          )}
                          {canCare && r.screening?.id && (
                            <button type="button" style={secondaryButtonStyle}
                              onClick={() => setPanel({ kind: "view", id: r.screening!.id })}>
                              Ver escuta
                            </button>
                          )}
                          {canCare && <button type="button" style={secondaryButtonStyle} onClick={() => setClosing(r)}>Encerrar</button>}
                        </div>
                      )
                    } ] : [])
```

E logo antes de `{closing && (`:

```tsx
        {clinicalDocuments && declaring && (
          <DeclarationPanel key={declaring.id} row={declaring} onClose={() => setDeclaring(null)} />
        )}
```

Em `src/modules/Attendance.tsx`, no `<UnitQueue …>`, depois de `clinicalRecord={hasFeature(user, "clinical_record")}`:

```tsx
          clinicalDocuments={hasFeature(user, "clinical_documents")}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/attendance/ src/modules/Attendance.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes antigos da fila continuam verdes: sem a prop nova, nada muda para quem cuida) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19c
/opt/homebrew/bin/git add src/modules/attendance/DeclarationPanel.tsx src/modules/attendance/UnitQueue.tsx \
  src/modules/attendance/UnitQueue.declaration.test.tsx src/modules/Attendance.tsx
/opt/homebrew/bin/git commit -m "feat: let the front desk issue attendance declarations from the unit queue

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Admin municipal — tela "Documentos clínicos": CNPJ da cidade e REMUME

**Files:**
- Create: `src/modules/ClinicalDocumentsAdmin.tsx`, `src/modules/clinicalDocumentsAdmin/CnpjPanel.tsx`, `src/modules/clinicalDocumentsAdmin/RemumePanel.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx`
- Test: `src/modules/ClinicalDocumentsAdmin.test.tsx`

**Interfaces:**
- Consumes: `getCityDocumentsProfile`, `setCityCnpj`, `listRemume`, `addRemumeItem`, `removeRemumeItem`, `listActiveUnits`, `ApiError`, `errorCode` (Task 1 e módulo 11); `CITY_DOCS_PROFILE_KEY`, `REMUME_KEY`, `DOCUMENTS_DISABLED`, `cnpjDigits`, `isValidCnpj`, `formatCnpj`, `documentError` (Task 2); `MedicationSearch` (Task 3); `SensitiveAction`, `Panel`, `PageHeader`, `DataTable`, `KeyValue`, `EmptyState`, `Tag`.
- Produces:
  - `ModuleId` `"clinical-documents-admin"`, item "Documentos clínicos" (ícone `℞`) no grupo **Cidade**, visível só para `municipal_admin` (não operador) com `clinical_documents` na sessão;
  - `ClinicalDocumentsAdmin({ onGoToSecurity?, searchDelayMs? })` — `PageHeader` "Documentos clínicos" e os painéis "CNPJ da cidade" e "REMUME" (e, na Task 10, "Protocolos de enfermagem");
  - `CnpjPanel({ onGoToSecurity? })` — mostra o CNPJ (ou "não cadastrado" e o efeito disso); "Cadastrar CNPJ"/"Trocar CNPJ" pelo `SensitiveAction` com step-up; DV conferido antes de mandar; manda só os dígitos;
  - `RemumePanel({ onGoToSecurity?, searchDelayMs? })` — lista (Medicamento, CATMAT, Unidades — "todas" sem unidade escolhida), "Incluir na REMUME" pela busca, com as unidades opcionais, e "Tirar", os dois com step-up.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/ClinicalDocumentsAdmin.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getCityDocumentsProfile: vi.fn(), setCityCnpj: vi.fn(), listRemume: vi.fn(),
    addRemumeItem: vi.fn(), removeRemumeItem: vi.fn(), listActiveUnits: vi.fn(), searchMedications: vi.fn(), listNursingProtocols: vi.fn(),
    createNursingProtocol: vi.fn(), addNursingProtocolVersion: vi.fn() };
});

import * as api from "../lib/api";
import { DOCUMENTS_DISABLED } from "../lib/clinicalDocuments";
import { ClinicalDocumentsAdmin } from "./ClinicalDocumentsAdmin";
import { renderWithProviders } from "../test/campaignFixtures";
import { AMOXICILLIN, DIPYRONE_REF, NOW19C, docUser, remumeEntry, searchItem } from "../test/documentFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const UNITS = [ { id: "u1", name: "UBS Centro", kind: "ubs" }, { id: "u2", name: "UBS Boqueirão", kind: "ubs" } ];
const admin = () => docUser([ "municipal_admin" ], { id: "ad1", email_address: "admin@curitiba.demo" });

afterEach(() => { cleanup(); vi.useRealTimers(); });

function renderIt() {
  return renderWithProviders(<ClinicalDocumentsAdmin searchDelayMs={0} />);
}

describe("ClinicalDocumentsAdmin", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW19C));
    for (const fn of [ api.fetchCurrentSession, api.getCityDocumentsProfile, api.setCityCnpj, api.listRemume, api.addRemumeItem,
      api.removeRemumeItem, api.listActiveUnits, api.searchMedications, api.listNursingProtocols ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(admin());
    mocked(api.getCityDocumentsProfile).mockResolvedValue({ cnpj: null });
    mocked(api.setCityCnpj).mockResolvedValue({ cnpj: "11222333000181" });
    mocked(api.listRemume).mockResolvedValue([ remumeEntry({ unit_ids: [ "u2" ] }), remumeEntry({ catalog_item: DIPYRONE_REF }) ]);
    mocked(api.addRemumeItem).mockResolvedValue(remumeEntry());
    mocked(api.removeRemumeItem).mockResolvedValue(undefined);
    mocked(api.listActiveUnits).mockResolvedValue(UNITS);
    mocked(api.searchMedications).mockResolvedValue([ searchItem(), AMOXICILLIN ]);
    mocked(api.listNursingProtocols).mockResolvedValue([]);
  });

  it("só municipal_admin com clinical_documents", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(docUser([ "health_professional" ]));
    const { unmount } = renderIt();
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
    expect(api.listRemume).not.toHaveBeenCalled();
    unmount();
    mocked(api.fetchCurrentSession).mockResolvedValue(docUser([ "municipal_admin" ], { features: [ "clinical_record" ] }));
    renderIt();
    expect(await screen.findByText(DOCUMENTS_DISABLED)).not.toBeNull();
    expect(api.getCityDocumentsProfile).not.toHaveBeenCalled();
  });

  it("CNPJ: sem cadastro diz o efeito; cadastrar com step-up recusa DV inválido e salva só os dígitos", async () => {
    renderIt();
    const panel = await screen.findByRole("region", { name: "CNPJ da cidade" });
    expect(await within(panel).findByText("não cadastrado")).not.toBeNull();
    expect(within(panel).getByText("Sem o CNPJ, a receita de enfermagem não sai (COFEN 801/2026).")).not.toBeNull();
    fireEvent.click(within(panel).getByRole("button", { name: "Cadastrar CNPJ" }));
    const dialog = screen.getByRole("region", { name: "Cadastrar o CNPJ da cidade" });
    fireEvent.change(within(dialog).getByLabelText("CNPJ (14 caracteres)"), { target: { value: "11.222.333/0001-80" } });
    fireEvent.click(within(dialog).getByRole("button", { name: "Salvar CNPJ" }));
    expect((await within(dialog).findByRole("alert")).textContent).toBe("CNPJ inválido — confira os 14 caracteres (números e, no CNPJ novo, letras)");
    expect(api.setCityCnpj).not.toHaveBeenCalled();
    fireEvent.change(within(dialog).getByLabelText("CNPJ (14 caracteres)"), { target: { value: "11.222.333/0001-81" } });
    fireEvent.click(within(dialog).getByRole("button", { name: "Salvar CNPJ" }));
    await waitFor(() => expect(api.setCityCnpj).toHaveBeenCalledWith("11222333000181"));
    expect((await within(panel).findByRole("status")).textContent).toBe("CNPJ salvo.");
  });

  it("REMUME: lista com as unidades, inclui com a unidade escolhida e tira, com step-up", async () => {
    renderIt();
    const panel = await screen.findByRole("region", { name: "REMUME" });
    const rows = (await within(panel).findAllByRole("row")).slice(1);
    expect(within(rows[0]).getByText("Metformina, cloridrato 850 mg, comprimido")).not.toBeNull();
    expect(await within(rows[0]).findByText("UBS Boqueirão")).not.toBeNull();
    expect(within(rows[1]).getByText("todas")).not.toBeNull();

    fireEvent.change(within(panel).getByLabelText("Incluir na REMUME"), { target: { value: "amox" } });
    fireEvent.click(await within(panel).findByRole("button", { name: "Amoxicilina 500 mg, cápsula" }));
    const add = within(panel).getByRole("region", { name: "Incluir: Amoxicilina 500 mg, cápsula" });
    fireEvent.click(within(add).getByLabelText("UBS Centro"));
    fireEvent.click(within(add).getByRole("button", { name: "Incluir" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar inclusão" }));
    await waitFor(() => expect(api.addRemumeItem).toHaveBeenCalledWith("ci2", [ "u1" ]));
    expect((await within(panel).findByRole("status")).textContent).toBe("Medicamento incluído na REMUME.");

    fireEvent.click(within(panel).getByRole("button", { name: "Tirar Metformina, cloridrato 850 mg, comprimido da REMUME" }));
    expect(screen.getByText("Ele deixa de aparecer como \"na rede\" na busca da receita; receitas já emitidas não mudam.")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Confirmar retirada" }));
    await waitFor(() => expect(api.removeRemumeItem).toHaveBeenCalledWith("ci1"));
    await waitFor(() => expect(mocked(api.listRemume).mock.calls.length).toBeGreaterThanOrEqual(3));
  });

  it("medicamento que já está na REMUME: diz em vez de incluir de novo", async () => {
    renderIt();
    const panel = await screen.findByRole("region", { name: "REMUME" });
    await within(panel).findAllByRole("row");
    fireEvent.change(within(panel).getByLabelText("Incluir na REMUME"), { target: { value: "metf" } });
    fireEvent.click(await within(panel).findByRole("button", { name: "Metformina, cloridrato 850 mg, comprimido" }));
    expect(within(panel).getByText("este medicamento já está na REMUME")).not.toBeNull();
    expect(within(panel).queryByRole("button", { name: "Incluir" })).toBeNull();
  });
});
```

Em `src/shell/modules.test.ts`, junto dos testes do 19b (depois de "Conta → Assinatura digital…"):

```ts
  it("Cidade → Documentos clínicos: só municipal_admin com clinical_documents, nunca operador", () => {
    const ids = (u: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(u).flatMap((g) => g.items.map((i) => i.id));
    const admin = { operator: false, memberships: [ { role: "municipal_admin" } ] };
    expect(ids({ ...admin, features: [ "clinical_record", "clinical_documents" ] })).toContain("clinical-documents-admin");
    expect(ids({ ...admin, features: [ "clinical_record" ] })).not.toContain("clinical-documents-admin");
    expect(ids({ operator: false, memberships: [ { role: "health_professional" } ], features: [ "clinical_documents" ] }))
      .not.toContain("clinical-documents-admin");
    expect(ids({ operator: true, memberships: [ { role: "municipal_admin" } ], features: [ "clinical_documents" ] }))
      .not.toContain("clinical-documents-admin");
    expect(labelFor("clinical-documents-admin")).toBe("Documentos clínicos");
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/ClinicalDocumentsAdmin.test.tsx src/shell/modules.test.ts`
Expected: FAIL — `./ClinicalDocumentsAdmin` não existe e o item não está no menu.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/ClinicalDocumentsAdmin.tsx
// Documentos clínicos da cidade (módulo 19c; spec §3 e §7 "Admin municipal";
// contrato §6): CNPJ da secretaria/fundo (sai na receita de enfermagem),
// REMUME (os medicamentos "na rede") e protocolos de enfermagem. Só
// municipal_admin, com clinical_documents; toda escrita com step-up.
import type { ReactNode } from "react";
import { useAuth } from "../lib/auth";
import { DOCUMENTS_DISABLED, canUseDocuments } from "../lib/clinicalDocuments";
import { PageHeader } from "../components/PageHeader";
import { EmptyState } from "../components/EmptyState";
import { CnpjPanel } from "./clinicalDocumentsAdmin/CnpjPanel";
import { RemumePanel } from "./clinicalDocumentsAdmin/RemumePanel";

export function ClinicalDocumentsAdmin({ onGoToSecurity, searchDelayMs }: { onGoToSecurity?(): void; searchDelayMs?: number }) {
  const { user } = useAuth();
  if (!user) return null;
  const isAdmin = !user.operator && user.memberships.some((m) => m.role === "municipal_admin");
  if (!isAdmin) return <Frame><EmptyState title="seu papel não permite ver esta tela" /></Frame>;
  if (!canUseDocuments(user)) return <Frame><EmptyState title={DOCUMENTS_DISABLED} /></Frame>;
  return (
    <Frame>
      <CnpjPanel onGoToSecurity={onGoToSecurity} />
      <RemumePanel onGoToSecurity={onGoToSecurity} searchDelayMs={searchDelayMs} />
    </Frame>
  );
}

function Frame({ children }: { children: ReactNode }) {
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Documentos clínicos" sub="cidade · CNPJ, REMUME e protocolos de enfermagem" />
      {children}
    </div>
  );
}
```

```tsx
// src/modules/clinicalDocumentsAdmin/CnpjPanel.tsx
// CNPJ da cidade (secretaria ou fundo municipal de saúde; spec §3; contrato §6
// e Divergência D2): obrigatório para a receita de enfermagem (COFEN 801/2026).
// Gravar passa pelo step-up; o dígito verificador é conferido aqui antes de ir
// ao api (que confere de novo: 422 invalid_cnpj). Vai só com os dígitos.
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { ApiError, errorCode, getCityDocumentsProfile, setCityCnpj } from "../../lib/api";
import { CITY_DOCS_PROFILE_KEY, cnpjDigits, documentError, formatCnpj, isValidCnpj } from "../../lib/clinicalDocuments";
import { Panel } from "../../components/Panel";
import { KeyValue } from "../../components/KeyValue";
import { SensitiveAction } from "../../components/SensitiveAction";
import { buttonStyle } from "../../components/formStyles";

export function CnpjPanel({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: [ CITY_DOCS_PROFILE_KEY ], queryFn: getCityDocumentsProfile, retry: false });
  const [ editing, setEditing ] = useState(false);
  const [ notice, setNotice ] = useState<string | null>(null);
  const cnpj = query.data?.cnpj ?? null;

  return (
    <Panel title="CNPJ da cidade" sub="secretaria ou fundo municipal de saúde · sai na receita de enfermagem">
      <div style={body}>
        {query.isPending && <p className="mono" style={muted}>carregando…</p>}
        {query.isError && <p role="alert" style={alert}>{documentError(query.error)}</p>}
        {query.isSuccess && <KeyValue k="CNPJ" v={formatCnpj(cnpj)} />}
        {query.isSuccess && !cnpj && <p style={muted}>Sem o CNPJ, a receita de enfermagem não sai (COFEN 801/2026).</p>}
        {notice && <p role="status" style={statusStyle}>{notice}</p>}
        {query.isSuccess && !editing && (
          <div>
            <button type="button" style={buttonStyle} onClick={() => { setNotice(null); setEditing(true); }}>
              {cnpj ? "Trocar CNPJ" : "Cadastrar CNPJ"}
            </button>
          </div>
        )}
        {editing && (
          <SensitiveAction
            title={cnpj ? "Trocar o CNPJ da cidade" : "Cadastrar o CNPJ da cidade"}
            description="O CNPJ da secretaria ou do fundo municipal de saúde sai em toda receita de enfermagem emitida daqui em diante."
            requiresStepUp
            fields={[ { name: "cnpj", label: "CNPJ (14 caracteres)", required: true } ]}
            confirmLabel="Salvar CNPJ"
            run={async (values) => {
              if (!isValidCnpj(values.cnpj)) throw new ApiError(422, { error: "invalid_cnpj" }, "local");
              await setCityCnpj(cnpjDigits(values.cnpj));
            }}
            onDone={() => {
              setEditing(false); setNotice("CNPJ salvo.");
              void queryClient.invalidateQueries({ queryKey: [ CITY_DOCS_PROFILE_KEY ] });
            }}
            onCancel={() => setEditing(false)}
            onGoToSecurity={onGoToSecurity}
            translateError={(err) => (errorCode(err) ? documentError(err) : null)}
          />
        )}
      </div>
    </Panel>
  );
}

const body: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 16, fontSize: 12.5 };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

```tsx
// src/modules/clinicalDocumentsAdmin/RemumePanel.tsx
// REMUME (módulo 19c; spec §3; contrato §6): os medicamentos do catálogo da
// plataforma que a rede municipal tem, com as unidades opcionais (nenhuma =
// todas). Na busca da receita eles vêm primeiro, com "na rede". Incluir e
// tirar passam pelo step-up. A busca no catálogo é a mesma da receita
// (Divergência D5: o api a libera ao municipal_admin).
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { addRemumeItem, errorCode, listActiveUnits, listRemume, removeRemumeItem, type MedicationSearchItem, type RemumeEntry } from "../../lib/api";
import { REMUME_KEY, documentError } from "../../lib/clinicalDocuments";
import { Panel } from "../../components/Panel";
import { DataTable, type Column } from "../../components/DataTable";
import { SensitiveAction } from "../../components/SensitiveAction";
import { buttonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { MedicationSearch } from "../documents/MedicationSearch";

type Confirming = { kind: "add"; item: MedicationSearchItem; unitIds: string[] } | { kind: "remove"; entry: RemumeEntry };

export function RemumePanel({ onGoToSecurity, searchDelayMs }: { onGoToSecurity?(): void; searchDelayMs?: number }) {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: [ REMUME_KEY ], queryFn: listRemume, retry: false });
  const units = useQuery({ queryKey: [ "activeUnits" ], queryFn: listActiveUnits });
  const [ picked, setPicked ] = useState<MedicationSearchItem | null>(null);
  const [ unitIds, setUnitIds ] = useState<string[]>([]);
  const [ confirming, setConfirming ] = useState<Confirming | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const reload = () => void queryClient.invalidateQueries({ queryKey: [ REMUME_KEY ] });
  const unitName = (id: string) => units.data?.find((u) => u.id === id)?.name ?? id;
  const inRemume = new Set((query.data ?? []).map((e) => e.catalog_item.id));

  const cols: Column<RemumeEntry>[] = [
    { label: "Medicamento", w: "2.4fr", render: (e) => e.catalog_item.label },
    { label: "CATMAT", w: "0.8fr", render: (e) => <span className="mono">{e.catalog_item.catmat_code}</span> },
    { label: "Unidades", w: "1.6fr", render: (e) => (e.unit_ids.length === 0 ? "todas" : e.unit_ids.map(unitName).join(", ")) },
    { label: "", w: "auto", align: "right", render: (e) => (
      <button type="button" aria-label={`Tirar ${e.catalog_item.label} da REMUME`} style={secondaryButtonStyle}
        onClick={() => { setNotice(null); setConfirming({ kind: "remove", entry: e }); }}>Tirar</button>
    ) }
  ];

  function toggleUnit(id: string, on: boolean) {
    setUnitIds((prev) => (on ? [ ...prev, id ] : prev.filter((u) => u !== id)));
  }

  return (
    <Panel title="REMUME" sub="relação municipal de medicamentos · aparecem primeiro na receita, como “na rede”">
      <div style={body}>
        {notice && <p role="status" style={statusStyle}>{notice}</p>}
        {query.isPending && <p className="mono" style={muted}>carregando…</p>}
        {query.isError && <p role="alert" style={alert}>{documentError(query.error)}</p>}
        {query.isSuccess && <DataTable cols={cols} rows={query.data} rowKey={(e) => e.catalog_item.id} empty="nenhum medicamento na REMUME" />}

        {!picked && !confirming && (
          <MedicationSearch label="Incluir na REMUME" blockControlled={false} delayMs={searchDelayMs}
            onPick={(item) => { setNotice(null); setPicked(item); setUnitIds([]); }} />
        )}
        {picked && !confirming && (
          <section aria-label={`Incluir: ${picked.label}`} style={box}>
            <strong>{picked.label}</strong>
            {inRemume.has(picked.id) ? (
              <p style={muted}>este medicamento já está na REMUME</p>
            ) : (
              <>
                <p style={muted}>Unidades (nenhuma marcada = todas as unidades da rede)</p>
                <div style={row}>
                  {(units.data ?? []).map((u) => (
                    <label key={u.id} style={checkLabel}>
                      <input type="checkbox" checked={unitIds.includes(u.id)} onChange={(e) => toggleUnit(u.id, e.target.checked)} />
                      {u.name}
                    </label>
                  ))}
                </div>
              </>
            )}
            <div style={row}>
              {!inRemume.has(picked.id) && (
                <button type="button" style={buttonStyle} onClick={() => setConfirming({ kind: "add", item: picked, unitIds })}>Incluir</button>
              )}
              <button type="button" style={secondaryButtonStyle} onClick={() => setPicked(null)}>Cancelar</button>
            </div>
          </section>
        )}

        {confirming?.kind === "add" && (
          <SensitiveAction
            title={`Incluir na REMUME: ${confirming.item.label}`}
            description={confirming.unitIds.length === 0 ? "Em todas as unidades da rede." : `Unidades: ${confirming.unitIds.map(unitName).join(", ")}.`}
            requiresStepUp
            confirmLabel="Confirmar inclusão"
            run={async () => { await addRemumeItem(confirming.item.id, confirming.unitIds); }}
            onDone={() => { setConfirming(null); setPicked(null); setNotice("Medicamento incluído na REMUME."); reload(); }}
            onCancel={() => setConfirming(null)}
            onGoToSecurity={onGoToSecurity}
            translateError={(err) => (errorCode(err) ? documentError(err) : null)}
          />
        )}
        {confirming?.kind === "remove" && (
          <SensitiveAction
            title={`Tirar da REMUME: ${confirming.entry.catalog_item.label}`}
            description={'Ele deixa de aparecer como "na rede" na busca da receita; receitas já emitidas não mudam.'}
            requiresStepUp
            confirmLabel="Confirmar retirada"
            run={async () => { await removeRemumeItem(confirming.entry.catalog_item.id); }}
            onDone={() => { setConfirming(null); setNotice("Medicamento retirado da REMUME."); reload(); }}
            onCancel={() => setConfirming(null)}
            onGoToSecurity={onGoToSecurity}
            translateError={(err) => (errorCode(err) ? documentError(err) : null)}
          />
        )}
      </div>
    </Panel>
  );
}

const body: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 16, fontSize: 12.5 };
const box: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, padding: 12, border: "1px solid var(--rule)", borderRadius: 8 };
const row: CSSProperties = { display: "flex", gap: 8, flexWrap: "wrap", alignItems: "center" };
const checkLabel: CSSProperties = { display: "flex", gap: 6, alignItems: "center", fontSize: 12, color: "var(--ink2)" };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

Em `src/shell/modules.ts`:

```ts
  | "signature" | "signature-pending" | "signature-overview" | "clinical-documents-admin";
```

```ts
  { label: "Cidade", items: [
    { id: "territory", label: "Território", icon: "⌖" },
    { id: "clinical-documents-admin", label: "Documentos clínicos", icon: "℞" }
  ]},
```

e, em `navGroupsFor`, junto dos outros filtros de item (antes de `if (item.id === "integrations" …`):

```ts
      // Módulo 19c: CNPJ, REMUME e protocolos de enfermagem, só do municipal_admin, com clinical_documents.
      if (item.id === "clinical-documents-admin") return !user?.operator && isAdmin && hasFeature(user, "clinical_documents");
```

Em `src/App.tsx`:

```tsx
import { ClinicalDocumentsAdmin } from "./modules/ClinicalDocumentsAdmin";
```

```tsx
    case "clinical-documents-admin": return <ClinicalDocumentsAdmin onGoToSecurity={() => setActive("security")} />;
```

(Logo depois do `case "signature-overview"`.)

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/ClinicalDocumentsAdmin.test.tsx src/shell/modules.test.ts && npx tsc --noEmit`
Expected: PASS (os testes antigos de `modules.test.ts` continuam: nenhum grupo novo, só um item no grupo Cidade, que já some para quem não é admin) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19c
/opt/homebrew/bin/git add src/modules/ClinicalDocumentsAdmin.tsx src/modules/ClinicalDocumentsAdmin.test.tsx \
  src/modules/clinicalDocumentsAdmin/CnpjPanel.tsx src/modules/clinicalDocumentsAdmin/RemumePanel.tsx \
  src/shell/modules.ts src/shell/modules.test.ts src/App.tsx
/opt/homebrew/bin/git commit -m "feat: add the municipal clinical documents screen with the city CNPJ and the REMUME

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Admin municipal — protocolos de enfermagem

**Files:**
- Create: `src/modules/clinicalDocumentsAdmin/NursingProtocolsPanel.tsx`, `src/modules/clinicalDocumentsAdmin/ProtocolForm.tsx`
- Modify: `src/modules/ClinicalDocumentsAdmin.tsx`
- Test: `src/modules/clinicalDocumentsAdmin/NursingProtocolsPanel.test.tsx`

**Interfaces:**
- Consumes: `listNursingProtocols`, `createNursingProtocol`, `addNursingProtocolVersion`, `NursingProtocol` (Task 1); `PROTOCOLS_KEY`, `ProtocolDraft`, `emptyProtocol`, `protocolDraftFrom`, `addProtocolItem`, `protocolProblem`, `protocolInput`, `protocolVersionInput`, `documentError` (Task 2); `MedicationSearch` (Task 3); `todayInCity`; `SensitiveAction`.
- Produces:
  - `ProtocolForm({ title, initial: ProtocolDraft, withHeader: boolean, searchDelayMs?, onReady(d: ProtocolDraft), onCancel() })` — `<form aria-label={title}>` com "Título", "Número", "Ano de publicação" (só `withHeader`), "Vigência — de", "Vigência — até (opcional)", a busca "Incluir medicamento no protocolo" (controlado bloqueado), por item "Dose máxima — <medicamento>" e "Tirar <medicamento>", e "Revisar e salvar" (valida com `protocolProblem` e chama `onReady`);
  - `NursingProtocolsPanel({ onGoToSecurity?, searchDelayMs? })` — `Panel` "Protocolos de enfermagem" com a lista (Protocolo, Número e ano, Vigência, Itens), "Novo protocolo" e, por linha, "Nova versão de <título>"; salvar passa pelo `SensitiveAction` com step-up ("Salvar protocolo"). Editar cria versão nova; a receita guarda a versão em que saiu.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/clinicalDocumentsAdmin/NursingProtocolsPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listNursingProtocols: vi.fn(), createNursingProtocol: vi.fn(),
    addNursingProtocolVersion: vi.fn(), searchMedications: vi.fn() };
});

import * as api from "../../lib/api";
import { NursingProtocolsPanel } from "./NursingProtocolsPanel";
import { renderWithProviders } from "../../test/campaignFixtures";
import { DIPYRONE, NOW19C, docUser, nursingProtocol } from "../../test/documentFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

afterEach(() => { cleanup(); vi.useRealTimers(); });

describe("NursingProtocolsPanel", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW19C));
    for (const fn of [ api.fetchCurrentSession, api.listNursingProtocols, api.createNursingProtocol, api.addNursingProtocolVersion,
      api.searchMedications ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(docUser([ "municipal_admin" ], { id: "ad1" }));
    mocked(api.listNursingProtocols).mockResolvedValue([ nursingProtocol() ]);
    mocked(api.createNursingProtocol).mockResolvedValue(nursingProtocol({ id: "np2" }));
    mocked(api.addNursingProtocolVersion).mockResolvedValue(nursingProtocol());
    mocked(api.searchMedications).mockResolvedValue([ DIPYRONE ]);
  });

  it("lista os protocolos com número, ano, vigência e itens", async () => {
    renderWithProviders(<NursingProtocolsPanel searchDelayMs={0} />);
    const rows = (await screen.findAllByRole("row")).slice(1);
    expect(within(rows[0]).getByText("Protocolo de enfermagem na atenção primária")).not.toBeNull();
    expect(within(rows[0]).getByText("nº 007/2025 · 2025")).not.toBeNull();
    expect(within(rows[0]).getByText("desde 01/03/2025")).not.toBeNull();
    expect(within(rows[0]).getByText("1")).not.toBeNull();
  });

  it("cria um protocolo: valida, pede step-up e manda título, número, ano, vigência e itens com dose máxima", async () => {
    renderWithProviders(<NursingProtocolsPanel searchDelayMs={0} />);
    fireEvent.click(await screen.findByRole("button", { name: "Novo protocolo" }));
    const form = screen.getByRole("form", { name: "Novo protocolo de enfermagem" });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("informe o título do protocolo");
    fireEvent.change(within(form).getByLabelText("Título"), { target: { value: "Protocolo de hipertensão" } });
    fireEvent.change(within(form).getByLabelText("Número"), { target: { value: "012/2026" } });
    fireEvent.change(within(form).getByLabelText("Incluir medicamento no protocolo"), { target: { value: "dipi" } });
    fireEvent.click(await within(form).findByRole("button", { name: "Dipirona sódica 500 mg/mL, solução oral" }));
    fireEvent.change(within(form).getByLabelText("Dose máxima — Dipirona sódica 500 mg/mL, solução oral"), { target: { value: "2" } });
    expect((within(form).getByLabelText("Unidade da dose máxima — Dipirona sódica 500 mg/mL, solução oral") as HTMLInputElement).value)
      .toBe("solução oral");
    fireEvent.change(within(form).getByLabelText("Unidade da dose máxima — Dipirona sódica 500 mg/mL, solução oral"), { target: { value: "frasco" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    const dialog = screen.getByRole("region", { name: "Criar o protocolo: Protocolo de hipertensão" });
    fireEvent.click(within(dialog).getByRole("button", { name: "Salvar protocolo" }));
    await waitFor(() => expect(api.createNursingProtocol).toHaveBeenCalledWith({
      title: "Protocolo de hipertensão", number: "012/2026", year: 2026, valid_from: "2026-10-09",
      items: [ { catalog_item_id: "ci4", max_dose: { quantity: 2, unit: "frasco" } } ]
    }));
    expect((await screen.findByRole("status")).textContent).toBe("Protocolo salvo.");
    expect(screen.queryByRole("form", { name: "Novo protocolo de enfermagem" })).toBeNull();
  });

  it("nova versão parte da vigente e não pede título", async () => {
    renderWithProviders(<NursingProtocolsPanel searchDelayMs={0} />);
    fireEvent.click(await screen.findByRole("button", { name: "Nova versão de Protocolo de enfermagem na atenção primária" }));
    const form = screen.getByRole("form", { name: "Nova versão: Protocolo de enfermagem na atenção primária" });
    expect(within(form).queryByLabelText("Título")).toBeNull();
    expect((within(form).getByLabelText("Dose máxima — Dipirona sódica 500 mg/mL, solução oral") as HTMLInputElement).value).toBe("2");
    fireEvent.change(within(form).getByLabelText("Vigência — até (opcional)"), { target: { value: "2027-12-31" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    fireEvent.click(within(screen.getByRole("region", { name: "Nova versão: Protocolo de enfermagem na atenção primária" }))
      .getByRole("button", { name: "Salvar protocolo" }));
    await waitFor(() => expect(api.addNursingProtocolVersion).toHaveBeenCalledWith("np1", {
      valid_from: "2026-10-09", valid_until: "2027-12-31", items: [ { catalog_item_id: "ci4", max_dose: { quantity: 2, unit: "frasco" } } ]
    }));
  });
});
```

Em `src/modules/ClinicalDocumentsAdmin.test.tsx` (Task 9), acrescente no `describe`:

```tsx
  it("mostra também os protocolos de enfermagem", async () => {
    renderIt();
    expect(await screen.findByRole("region", { name: "Protocolos de enfermagem" })).not.toBeNull();
    expect(await screen.findByText("nenhum protocolo cadastrado")).not.toBeNull();
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/clinicalDocumentsAdmin/NursingProtocolsPanel.test.tsx src/modules/ClinicalDocumentsAdmin.test.tsx`
Expected: FAIL — `./NursingProtocolsPanel` não existe e a tela não tem o painel.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/clinicalDocumentsAdmin/ProtocolForm.tsx
// Protocolo de enfermagem (módulo 19c; spec §3; COFEN 801/2026): título,
// número e ano de publicação; vigência; os medicamentos que a enfermagem pode
// prescrever, com dose máxima opcional (Divergência D6: limite da quantidade
// por item da receita). Só valida e entrega o rascunho; quem grava (com
// step-up) é o painel.
import { useState, type CSSProperties, type FormEvent } from "react";
import { addProtocolItem, protocolProblem, type ProtocolDraft } from "../../lib/clinicalDocuments";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { MedicationSearch } from "../documents/MedicationSearch";

interface Props {
  title: string;
  initial: ProtocolDraft;
  withHeader: boolean;
  searchDelayMs?: number;
  onReady(d: ProtocolDraft): void;
  onCancel(): void;
}

export function ProtocolForm({ title, initial, withHeader, searchDelayMs, onReady, onCancel }: Props) {
  const [ d, setD ] = useState<ProtocolDraft>(initial);
  const [ problem, setProblem ] = useState<string | null>(null);
  const set = (patch: Partial<ProtocolDraft>) => setD((prev) => ({ ...prev, ...patch }));

  function submit(event: FormEvent) {
    event.preventDefault();
    const found = protocolProblem(d, withHeader);
    setProblem(found);
    if (!found) onReady(d);
  }

  return (
    <form aria-label={title} onSubmit={submit} style={box}>
      <strong>{title}</strong>
      {withHeader && (
        <div style={row}>
          <label style={label}>Título<input value={d.title} style={inputStyle} onChange={(e) => set({ title: e.target.value })} /></label>
          <label style={label}>Número<input value={d.number} style={inputStyle} onChange={(e) => set({ number: e.target.value })} /></label>
          <label style={label}>
            Ano de publicação
            <input inputMode="numeric" value={d.year} style={inputStyle} onChange={(e) => set({ year: e.target.value })} />
          </label>
        </div>
      )}
      <div style={row}>
        <label style={label}>
          Vigência — de
          <input type="date" value={d.validFrom} style={inputStyle} onChange={(e) => set({ validFrom: e.target.value })} />
        </label>
        <label style={label}>
          Vigência — até (opcional)
          <input type="date" value={d.validUntil} style={inputStyle} onChange={(e) => set({ validUntil: e.target.value })} />
        </label>
      </div>
      <MedicationSearch label="Incluir medicamento no protocolo" blockControlled delayMs={searchDelayMs}
        onPick={(item) => setD((prev) => addProtocolItem(prev, item))} />
      {d.items.length === 0 ? <p style={muted}>nenhum medicamento no protocolo</p> : (
        <ul aria-label="medicamentos do protocolo" style={list}>
          {d.items.map((i) => (
            <li key={i.item.id} style={row}>
              <span style={{ flex: 1, fontSize: 12.5 }}>{i.item.label}</span>
              <label style={label}>
                {`Dose máxima — ${i.item.label}`}
                <input inputMode="numeric" value={i.maxDose} style={inputStyle}
                  onChange={(e) => set({ items: d.items.map((x) => (x.item.id === i.item.id ? { ...x, maxDose: e.target.value } : x)) })} />
              </label>
              <label style={label}>
                {`Unidade da dose máxima — ${i.item.label}`}
                <input value={i.maxUnit} style={inputStyle}
                  onChange={(e) => set({ items: d.items.map((x) => (x.item.id === i.item.id ? { ...x, maxUnit: e.target.value } : x)) })} />
              </label>
              <button type="button" aria-label={`Tirar ${i.item.label}`} style={secondaryButtonStyle}
                onClick={() => set({ items: d.items.filter((x) => x.item.id !== i.item.id) })}>Tirar</button>
            </li>
          ))}
        </ul>
      )}
      {problem && <p role="alert" style={alert}>{problem}</p>}
      <div style={row}>
        <button type="submit" style={buttonStyle}>Revisar e salvar</button>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </form>
  );
}

const box: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 12, border: "1px solid var(--rule)", borderRadius: 8 };
const row: CSSProperties = { display: "flex", gap: 8, flexWrap: "wrap", alignItems: "flex-end" };
const list: CSSProperties = { listStyle: "none", margin: 0, padding: 0, display: "flex", flexDirection: "column", gap: 6 };
const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

```tsx
// src/modules/clinicalDocumentsAdmin/NursingProtocolsPanel.tsx
// Protocolos de enfermagem da cidade (módulo 19c; spec §3; contrato §6): só
// municipal_admin, com step-up. Editar cria uma versão nova (a receita de
// enfermagem guarda a versão em que saiu); a enfermagem só prescreve itens do
// protocolo vigente.
import { useRef, useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { addNursingProtocolVersion, createNursingProtocol, errorCode, listNursingProtocols, type NursingProtocol } from "../../lib/api";
import {
  PROTOCOLS_KEY, documentError, emptyProtocol, protocolDraftFrom, protocolInput, protocolVersionInput, type ProtocolDraft
} from "../../lib/clinicalDocuments";
import { todayInCity } from "../../lib/campaigns";
import { Panel } from "../../components/Panel";
import { DataTable, type Column } from "../../components/DataTable";
import { SensitiveAction } from "../../components/SensitiveAction";
import { buttonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { ProtocolForm } from "./ProtocolForm";

type Editing = { mode: "new" } | { mode: "version"; protocol: NursingProtocol };

const day = (iso: string) => iso.split("-").reverse().join("/");

function validity(p: NursingProtocol): string {
  const v = p.current_version;
  if (!v) return "sem versão";
  return v.valid_until ? `de ${day(v.valid_from)} até ${day(v.valid_until)}` : `desde ${day(v.valid_from)}`;
}

export function NursingProtocolsPanel({ onGoToSecurity, searchDelayMs }: { onGoToSecurity?(): void; searchDelayMs?: number }) {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: [ PROTOCOLS_KEY ], queryFn: listNursingProtocols, retry: false });
  const [ editing, setEditing ] = useState<Editing | null>(null);
  const [ ready, setReady ] = useState<ProtocolDraft | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const formKey = useRef(0);
  // O último rascunho: cancelar o step-up volta ao formulário com o que foi digitado.
  const lastDraft = useRef<ProtocolDraft | null>(null);
  const today = todayInCity();

  function start(next: Editing) {
    formKey.current += 1;
    lastDraft.current = null;
    setNotice(null); setReady(null); setEditing(next);
  }

  const cols: Column<NursingProtocol>[] = [
    { label: "Protocolo", w: "2.2fr", render: (p) => p.title },
    { label: "Número e ano", w: "1.2fr", render: (p) => `nº ${p.number} · ${p.year}` },
    { label: "Vigência", w: "1.4fr", render: (p) => validity(p) },
    { label: "Itens", w: "0.5fr", align: "right", render: (p) => String(p.current_version?.items.length ?? 0) },
    { label: "", w: "auto", align: "right", render: (p) => (
      <button type="button" aria-label={`Nova versão de ${p.title}`} style={secondaryButtonStyle}
        onClick={() => start({ mode: "version", protocol: p })}>Nova versão</button>
    ) }
  ];

  const formTitle = editing?.mode === "version" ? `Nova versão: ${editing.protocol.title}` : "Novo protocolo de enfermagem";

  return (
    <Panel title="Protocolos de enfermagem" sub="COFEN 801/2026 · a enfermagem só prescreve itens do protocolo vigente">
      <div style={body}>
        {notice && <p role="status" style={statusStyle}>{notice}</p>}
        {query.isPending && <p className="mono" style={muted}>carregando…</p>}
        {query.isError && <p role="alert" style={alert}>{documentError(query.error)}</p>}
        {query.isSuccess && <DataTable cols={cols} rows={query.data} rowKey={(p) => p.id} empty="nenhum protocolo cadastrado" />}
        {!editing && (
          <div><button type="button" style={buttonStyle} onClick={() => start({ mode: "new" })}>Novo protocolo</button></div>
        )}
        {editing && !ready && (
          <ProtocolForm key={formKey.current} title={formTitle} withHeader={editing.mode === "new"} searchDelayMs={searchDelayMs}
            initial={lastDraft.current ?? (editing.mode === "new" ? emptyProtocol(today) : protocolDraftFrom(editing.protocol, today))}
            onReady={(d) => { lastDraft.current = d; setReady(d); }} onCancel={() => setEditing(null)} />
        )}
        {editing && ready && (
          <SensitiveAction
            title={editing.mode === "new" ? `Criar o protocolo: ${ready.title.trim()}` : `Nova versão: ${editing.protocol.title}`}
            description="Editar cria uma versão nova; as receitas já emitidas guardam a versão em que saíram."
            requiresStepUp
            confirmLabel="Salvar protocolo"
            run={async () => {
              if (editing.mode === "new") await createNursingProtocol(protocolInput(ready));
              else await addNursingProtocolVersion(editing.protocol.id, protocolVersionInput(ready));
            }}
            onDone={() => {
              lastDraft.current = null;
              setEditing(null); setReady(null); setNotice("Protocolo salvo.");
              void queryClient.invalidateQueries({ queryKey: [ PROTOCOLS_KEY ] });
            }}
            onCancel={() => setReady(null)}
            onGoToSecurity={onGoToSecurity}
            translateError={(err) => (errorCode(err) ? documentError(err) : null)}
          />
        )}
      </div>
    </Panel>
  );
}

const body: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 16, fontSize: 12.5 };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

Em `src/modules/ClinicalDocumentsAdmin.tsx`:

```tsx
import { NursingProtocolsPanel } from "./clinicalDocumentsAdmin/NursingProtocolsPanel";
```

e, depois do `<RemumePanel …/>`:

```tsx
      <NursingProtocolsPanel onGoToSecurity={onGoToSecurity} searchDelayMs={searchDelayMs} />
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run src/modules/clinicalDocumentsAdmin/ src/modules/ClinicalDocumentsAdmin.test.tsx && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19c
/opt/homebrew/bin/git add src/modules/clinicalDocumentsAdmin/NursingProtocolsPanel.tsx src/modules/clinicalDocumentsAdmin/ProtocolForm.tsx \
  src/modules/clinicalDocumentsAdmin/NursingProtocolsPanel.test.tsx src/modules/ClinicalDocumentsAdmin.tsx src/modules/ClinicalDocumentsAdmin.test.tsx
/opt/homebrew/bin/git commit -m "feat: add nursing protocols with versions to the municipal clinical documents screen

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–10 na branch `feat/mod-19c-documents`; o api do 19c rodando na **3038** com a semente do 19c, o PSC simulado e o `signer` do compose.
- Produces: branch pronta para merge (depois do merge do api do 19c).

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd apps/dashboard/.claude/mod19c && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base anotada na Task 0 mais **14 arquivos de teste novos** (`api.clinicalDocuments`, `clinicalDocuments`, `MedicationSearch`, `MedicationsBlock`, `ConsultationWorkspace.documents`, `SickNoteForm`, `DeclarationForm`, `ExamRequisitionForm`, `PrescriptionForm`, `ConsultationDocuments`, `MyConsultations.documents`, `UnitQueue.declaration`, `ClinicalDocumentsAdmin`, `NursingProtocolsPanel`). Um número de arquivos que dobra é artefato de build descoberto pelo vitest: pare e reporte. A CI roda os mesmos três; `dist/` está no `.gitignore`.

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19c status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19c log --stat origin/main..HEAD | grep -c node_modules
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19c diff origin/main --stat -- src/components
```

Expected: `status` vazio, a contagem `0` e nenhum arquivo de `src/components` mudado (interface sem redesign).

- [ ] **Step 2: Nada vaza**

```bash
cd apps/dashboard/.claude/mod19c
grep -rn "console\." src/lib/clinicalDocuments.ts src/modules/documents src/modules/consultation/MedicationsBlock.tsx \
  src/modules/attendance/DeclarationPanel.tsx src/modules/ClinicalDocumentsAdmin.tsx src/modules/clinicalDocumentsAdmin
grep -rn "localStorage\|sessionStorage" src/lib/clinicalDocuments.ts src/modules/documents src/modules/consultation/MedicationsBlock.tsx \
  src/modules/attendance/DeclarationPanel.tsx src/modules/clinicalDocumentsAdmin
sed -n '/─── Documentos clínicos (módulo 19c/,$p' src/lib/api.ts | grep -n "URLSearchParams\|?q=\|?reason\|?cnpj\|?cid"
```

Expected: os três sem saída — nenhum `console`, nenhum armazenamento do navegador, nada de termo de busca, motivo, CNPJ ou CID em URL.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§3–§8, §11), o ADR 0033 e o contrato do 19c (§1–§7). Pontos de atenção:
- toda escrita leva JSON, inclusive o `DELETE` da REMUME; nada de CID, medicamento, motivo ou CNPJ em URL; leituras clínicas com `gcTime: 0`;
- cancelar, CNPJ, REMUME e protocolos só pelo `SensitiveAction` com step-up; emitir e mudar a lista de medicamentos sem step-up (contrato §3–§4);
- a tela nunca decide o modo de emissão nem libera o que o api recusa: o espelho (CBO, protocolo, dose, controlado, CID) só evita oferecer; a recusa do api aparece no formulário, que fica preenchido;
- controlado aparece bloqueado na receita e liberado na lista de uso e na REMUME; antimicrobiano avisa antes de emitir;
- o editor da consulta continua montado na aba Documentos (o autosave não perde o que foi digitado);
- 409 `already_cancelled` relê e não abre o substituto; 409 `already_active` relê a lista de medicamentos;
- a recepção só vê "Declaração"; o operador não vê nada novo; o item do admin some sem `clinical_documents`;
- interface sem redesign: só componentes existentes; nenhum componente comum mudou.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 19c rodando na porta **3038** (plano do api do 19c: servidor do worktree no container, semente com Curitiba em `record_mode = record`, `clinical_record`, `digital_signature`, `signature_psc_mock` e `clinical_documents` ligados em dev, catálogo de medicamentos importado — a prova do maintenance do 19c importa o CATMAT —, a médica e uma enfermeira da semente com CPF, um protocolo de enfermagem vigente e o CNPJ da cidade vazio no começo). O Vite do worktree sobe num container da rede do compose, para alcançar `http://api:3038` (se o plano do api subir o servidor do 19c num container próprio, use o nome dele no lugar de `api`):

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose run -d --rm --no-deps --name dashboard-mod19c -p 5188:5188 \
  -w /app/.claude/mod19c -e VITE_API_PROXY_TARGET=http://api:3038 \
  dashboard npx vite --port 5188 --host 0.0.0.0
docker logs -f dashboard-mod19c   # espere o "ready"; Ctrl+C sai do log, o container segue
```

Abra `http://curitiba.localhost:5188/dashboard/`. O usuário faz o login; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Confira com screenshot:
- como `admin@curitiba.demo`, Cidade → **Documentos clínicos**: CNPJ "não cadastrado" → "Cadastrar CNPJ" com um DV errado (recusa na tela) e depois um válido → "CNPJ salvo."; REMUME: incluir um medicamento numa unidade e outro em todas; Protocolos: o vigente da semente, "Nova versão" com mais um item;
- como a médica, com a sessão de assinatura aberta (19b), chamar um paciente, iniciar a consulta: "Medicamentos em uso" ao lado dos problemas; "Informar medicamento em uso de fora"; aba **Documentos** → **Atestado** com CID (sem marcar a autorização: recusa na tela; marcando: emite) → "Documento emitido (Atestado) em modo digital…"; a linha mostra "assinatura pendente" e, em até 15 s, "assinada digitalmente"; **Receita** com um item da REMUME ("na rede"), um controlado (bloqueado, "receita de controle especial — 19d"), um de texto livre e um de uso contínuo → emitida; os medicamentos em uso ganham o contínuo; uma receita com amoxicilina → o aviso de papel em 2 vias antes de emitir, e o PDF com "1ª via — farmácia" / "2ª via — paciente";
- **Imprimir** cada documento: o PDF tem o QR code e o código curto; abrir `http://curitiba.localhost:5188/v/<token>` (o token do `verification_url`) mostra tipo, data, profissional, unidade, iniciais e ano de nascimento, "válido" e o modo — **nunca** CID, medicamento, dias de afastamento ou CPF;
- **Cancelar** o atestado com motivo curto (recusa) e válido (step-up) → "cancelado em …"; a página `/v/<token>` passa a dizer "cancelado"; **Cancelar e emitir outro** a receita → o aviso "cancelar não desfaz medicamento já entregue ao paciente" e a receita nova aberta preenchida; emitida, ela mostra "substitui um cancelado";
- **Renovar receita** numa consulta nova do mesmo paciente → a receita abre com os contínuos;
- **Minhas consultas** → abrir a consulta → os documentos com Imprimir e Cancelar (sem emissão);
- como a enfermeira: aba Documentos sem "Atestado"; **Receita** só com o protocolo vigente e os itens dele (sem busca e sem texto livre); acima da dose máxima, recusa na tela; com o CNPJ cadastrado, a receita sai com protocolo e CNPJ;
- como a recepção (`citizen_verifier`), Atendimento → fila → **Declaração** de quem espera → "Emitir declaração" com a matrícula → "Documento emitido (Declaração de comparecimento) em modo papel…" e "Imprimir";
- no maintenance, desligar `clinical_documents` em Curitiba e recarregar o dashboard: a aba Documentos, os medicamentos em uso, o botão "Declaração" e o item do admin somem; a página `/v/<token>` continua respondendo.

Depois: `docker rm -f dashboard-mod19c`. Religue o que a prova desligou (interruptores da cidade como estavam antes).

- [ ] **Step 5:** **Pare.** O merge do dashboard só vem depois do merge do api do 19c, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: contracts → api → dashboard → maintenance (contrato §11). Antes do push, confira `origin/main..main` no dashboard e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-19c-documents
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove .claude/mod19c
/opt/homebrew/bin/git branch -d feat/mod-19c-documents
```

---

## Divergências propostas ao contrato

Formatos que este plano precisou e que o contrato do 19c não traz (ou traz de outro jeito). Onde o plano do `api` do 19c (`2026-10-09-module-19c-documents-api.md`, "Desvios e precisões") já fixou a forma, este plano a segue e diz isso.

1. **D1 — Forma de entrada do `POST …/documents` e das declarações** (segue o desvio 14 do plano do `api`). O §3 manda `{ kind, content }`, mas o §2 descreve o `content` de **saída**. A entrada é a forma de saída sem os campos que o servidor calcula (`label`, `printed_description`, `antimicrobial`, `position`, `copies`, `city_cnpj`, `unit_name`, `date`, `exams`):
   - atestado `{ type, days?, start_on?, companion_name?, companion_kinship?, companion_reason?, cid10?: { code }, cid_authorized, note? }`;
   - declaração `{ period? | arrived_at?, left_at?, companion_name?, issuer_registration? }` — o dia e a unidade são os do atendimento (o formulário não tem campo de dia);
   - receita `{ items: [ { catalog_item?: { id } | free_text?, quantity, quantity_unit, route, dosage_instructions, duration_days?, continuous } ], nursing_protocol?: { version_id } }`;
   - requisição `{ note? }` (os exames são os `consultation_exam_requests` salvos).
   `invalid_content.field` e `invalid_item.{index, field}` usam os nomes de saída. O dashboard só aceita quantidade inteira (o `api` aceita até 2 casas); fracionar fica para quando a interface pedir.
2. **D2 — Rota do CNPJ** (igual ao desvio 12 do plano do `api`). O §6 diz `GET/PUT /admin/city_profile`, mas `/admin` é o console **só de leitura** (`config/routes.rb`: "NENHUMA rota de escrita pode ser adicionada aqui") e não tem `city_profile`. Fica `GET/PUT /clinical_documents/city_profile` (`{ name, cnpj }`; `PUT` com step-up; 422 `invalid_cnpj`), com CNPJ alfanumérico da IN RFB 2.229/2024 (a tela confere o DV com o valor ASCII − 48, como a RFB, e manda sem máscara).
3. **D3 — A enfermagem lê os protocolos** (o plano do `api` já libera o `GET /clinical_documents/nursing_protocols` ao profissional). O contrato §6 o punha só sob `municipal_admin`; a receita de enfermagem precisa do protocolo e dos itens dele. A tela filtra os vigentes na data da cidade.
4. **D4 — `max_dose = { quantity, unit }`** (desvio 10 do plano do `api`). O §6 trazia `max_dose?` sem forma. É a quantidade máxima por item de receita; unidade diferente da do protocolo também recusa (`above_protocol_max_dose`). O cadastro do protocolo pede as duas (a unidade vem preenchida com a forma farmacêutica).
5. **D5 — Busca no catálogo para o `municipal_admin`.** A REMUME e os protocolos escolhem itens do catálogo, mas `POST /attendance/medications/search` é só do profissional (`require_professional` no plano do `api`). Proposta: a mesma rota aceita o `municipal_admin` (sem `unit_id`). **Ainda em aberto com o plano do `api`**: recusada, o `api` dá uma busca em `/clinical_documents/catalog_search` com a mesma forma e muda só a função do cliente.
6. **D6 — `document_type` `clinical_document`** nos pedidos de assinatura (o mesmo do plano do `api`: no banco `ClinicalDocument`). Em "Pendentes de assinatura" aparece como "documento clínico".
7. **D7 — Declaração da recepção depois que o atendimento saiu da fila.** O §3 aceita atendimento encerrado há até 30 dias, mas a recepção não tem rota para achar um atendimento encerrado; nesta entrega o botão "Declaração" fica na fila (Aguardando e Em atendimento) e o painel aberto sobrevive à saída da linha. Proposta para depois: `GET /attendance/units/:id/attendances?date=AAAA-MM-DD` com os encerrados do dia.
8. **D8 — Hora da declaração `HH:MM`** (hora de parede da cidade), igual ao esquema canônico (`contracts`, C3) e ao plano do `api`.
9. **D9 — 409 `not_draft`** para mudar a lista de medicamentos com a consulta finalizada (está na lista de erros do plano do `api`; o contrato §4 não dava o código).
10. **D10 — Impresso de documento digital ainda não assinado → 409 `awaiting_signature`** (desvio 7 do plano do `api`; não está no contrato §3). A tela diz para assinar ou voltar ao papel em "Pendentes de assinatura"; e `already_exists` (protocolo com o mesmo número e ano) ganha frase.

## Self-review

- **Cobertura (spec §7 e §11; contrato §2–§7):**
  - consulta → aba Documentos (emitir os quatro; lista com modo, estado da assinatura, Imprimir, Cancelar, Cancelar e emitir outro) — Tasks 4, 5 e 6; medicamentos em uso ao lado dos problemas com suspender, reativar e "de fora" — Task 3 (F-19.18);
  - receita com busca (REMUME primeiro, controlado bloqueado), texto livre, enfermagem por protocolo (sem texto livre, dose máxima, CNPJ), antimicrobiano e renovação — Tasks 2, 3, 5 e 6 (F-19.19);
  - atestado com a autorização do CID e o de acompanhante — Tasks 2 e 4 (F-19.20); declaração na consulta e na recepção — Tasks 4, 6 e 8 (F-19.21); requisição de exames — Tasks 4 e 6 (F-19.22);
  - cancelamento pela autora com motivo e step-up, aviso do medicamento entregue, Imprimir com o PDF do api (QR e código no PDF; página pública servida pelo api, proxy `/v/` para a prova) — Tasks 1, 6 e 11 (F-19.23);
  - Minhas consultas — Task 7; admin: CNPJ e REMUME — Task 9, protocolos — Task 10 (F-19.17); interruptor `clinical_documents` e proxy — Task 1; LGPD na tela (spec §8) — Global Constraints e Task 11; Vitest nas telas (spec §9) — Tasks 1–10; prova no navegador — Task 11.
  - Fora desta entrega, de propósito: `reason_problem_id` (opcional; o contrato o guarda para a RNDS futura, sem tela agora); a página pública em si (é do api).
- **Placeholders:** nenhum. Arquivos novos vêm inteiros; os existentes, por trecho com contexto de `origin/main` `8a0c0de` (a Task 0 manda conferir).
- **Consistência de nomes:** chaves `DOCUMENTS_KEY`, `MEDICATIONS_KEY`, `PROTOCOLS_KEY`, `REMUME_KEY`, `CITY_DOCS_PROFILE_KEY` (Task 2) usadas nas Tasks 3, 5, 6, 9 e 10; `emptySickNote`, `emptyDeclaration`, `itemFromPrescription`, `reissueDraft`, `ReissueDraft` (Task 2) no contêiner (Task 6); `SickNoteForm`, `DeclarationForm`, `ExamRequisitionForm` (Task 4) e `PrescriptionForm` (Task 5) com `onSubmit(content)` usados pelo contêiner (Task 6) e pela recepção (Task 8, `DeclarationForm`); `printDocument` (Task 6) na recepção (Task 8); `MedicationSearch` (Task 3) nas Tasks 5, 9 e 10; `canUseDocuments` (Task 2) nas Tasks 3, 7 e 9; fixtures `NOW19C`, `TODAY19C`, `docUser`, `catalogItem`, `searchItem`, `AMOXICILLIN`, `CLONAZEPAM`, `DIPYRONE`, `DIPYRONE_REF`, `prescriptionItem`, `sickNoteDoc`, `prescriptionDoc`, `declarationDoc`, `examDoc`, `medication`, `remumeEntry`, `nursingProtocol` (Task 1) nas Tasks 2–10; id de módulo `clinical-documents-admin` igual em `modules.ts`, `App.tsx` e nos testes.
- **Review Focus:** 1 — Task 6 ("outra aba já cancelou (409 already_cancelled)…"); 2 — Task 5 ("enfermeira: escolhe o protocolo vigente…", "enfermeira sem protocolo vigente…", "CNPJ da cidade faltando…"); 3 — Task 3 ("receita: controlado aparece bloqueado…") e Task 5 ("controlado não entra…", "antimicrobiano: avisa…"); 4 — Task 2 ("atestado: CID sem autorização…") e Task 4 ("CID escolhido sem marcar…", "acompanhante: … tira o CID"); 5 — Task 8 ("a pessoa sai da fila enquanto a recepção preenche…").
