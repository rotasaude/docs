# Módulo 19 (19c) — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-09-module-19c-clinical-documents-design.md` · **ADR:** `adr/0033.md`

Fonte única dos formatos entre `api`, `dashboard`, `maintenance` e `contracts`.
Erros `{ "error": "<reason>" }` (com `field`/`index` quando couber); step-up
401 `{ "error": "mfa_required" }`; interruptor desligado 403
`{ "error": "feature_disabled", "feature": "clinical_documents" }`; escrita
devolve o objeto puro; toda escrita por cookie exige `Content-Type:
application/json`, inclusive DELETE (contrato do 19b, §13). Leitura de
documento segue `ClinicalRecord::Access` do 19a (autora sempre; outro
profissional em contexto ou abertura, senão 403 `opening_required`/`out_of_context`
como na consulta; `municipal_admin` só conteúdo, com step-up, trilha
`administrative`). Assinatura: formas do contrato do 19b (bloco `signature`).

## 1. Valores comuns

- `kind`: `sick_note` | `attendance_declaration` | `prescription` | `exam_requisition`.
- `issue_mode`: `digital` | `paper`. `status`: `issued` | `cancelled`.
- `sick_note.type`: `leave` | `companion`. `companion_reason`: `clt_473_x` (gestante,
  consultas pré-natal) | `clt_473_xi` (filho até 6 anos) | `clt_473_xii`
  (preventivo de câncer) | `other`.
- Medicamento em uso: `status` `active` | `suspended`; `origin` `prescription` | `external`.
- Via (`route`): `oral` | `sublingual` | `topical` | `ophthalmic` | `otic` |
  `nasal` | `inhalation` | `vaginal` | `rectal` | `intramuscular` |
  `intravenous` | `subcutaneous` | `other`.

## 2. Formas

`<document>`:
```json
{ "id", "kind", "status", "issue_mode", "issued_at", "cancelled_at"|null, "cancel_reason"|null,
  "author": { "id", "name", "council", "cbo_code" },
  "patient": { "id", "display_name" }|null, "consultation_id"|null, "attendance_id",
  "short_code", "verification_url", "replaces_document_id"|null,
  "content": <conteúdo do tipo>, "signature": <bloco do 19b>|null }
```
Conteúdo por tipo:
- `sick_note`: `{ type, days?, start_on?, companion_name?, companion_kinship?, companion_reason?, cid10?: {code,label}, cid_authorized: bool, note? }`.
- `attendance_declaration`: `{ date, arrived_at?, left_at?, period?: "morning"|"afternoon"|"full_day", unit_name, companion_name?, issuer_registration? }`.
- `prescription`: `{ items: [<item>], nursing_protocol?: { id, title, number, year, version_id }, city_cnpj?, antimicrobial: bool, copies: 1|2, valid_until? }`.
- `exam_requisition`: `{ exams: [ { sigtap_code, label, competence, cid10_justification? } ], note? }`.

`<item>` (receita): `{ position, catalog_item?: { id, catmat_code, label, active_ingredient, strength, dosage_form }, free_text?, printed_description, quantity, quantity_unit, route, dosage_instructions, duration_days?, continuous: bool, antimicrobial: bool, reason_problem_id? }`.

`<medication>` (em uso): `{ id, catalog_item?: {…}, free_text?, label, dosage_summary, continuous, status, origin, started_on?, updated_at }`.

## 3. Documentos (`/attendance`)

- `GET /attendance/consultations/:id/documents` → `{ "items": [<document>] }`.
- `POST /attendance/consultations/:id/documents { kind, content, replaces_document_id? }`
  (autora; rascunho ou finalizada) → 201 `<document>`; 403 `not_author`,
  `cbo_not_allowed`; 422 por tipo: `invalid_content` (com `field`),
  `cid_requires_authorization`, `companion_cid_not_allowed`,
  `controlled_not_allowed`, `not_in_nursing_protocol`,
  `above_protocol_max_dose`, `free_text_not_allowed`, `city_cnpj_missing`,
  `no_exam_requests`, `invalid_item` (com `index`).
- `POST /attendance/attendances/:id/declarations { content }` (recepção ou
  profissional com vínculo na unidade) → 201 `<document>`; 409
  `attendance_not_found_or_closed_long_ago` (atendimento de mais de 30 dias).
- `GET /attendance/documents/:id` → `<document>` (trilha).
- `GET /attendance/documents/:id/print` → `application/pdf` (assinado quando
  `digital` e `signed`; senão o PDF do modo papel).
- `POST /attendance/documents/:id/cancel { reason }` (autora; step-up) →
  `<document>`; 422 `invalid_reason` (< 10); 409 `already_cancelled`.

## 4. Medicamentos em uso

- `GET /attendance/patients/:id/medications` → `{ "items": [<medication>] }` (mesma regra de leitura do prontuário).
- `POST /attendance/consultations/:id/medications { action: "add_external"|"suspend"|"reactivate", medication_id?, catalog_item_id?, free_text?, dosage_summary?, continuous? }`
  (autora, consulta em rascunho) → `<medication>`; 409 `already_active`.
- Receita com item `continuous` inclui/atualiza a lista na emissão.
- `GET /attendance/consultations/:id/renewal` → `{ "items": [<item>] }` (contínuos ativos, para preencher a receita).

## 5. Catálogo e buscas

- `POST /attendance/medications/search { q, unit_id? }` → `{ "items": [ { id, catmat_code, label, active_ingredient, strength, dosage_form, antimicrobial, controlled, in_network: bool } ] }` (REMUME primeiro; controlados com `controlled: true` e bloqueados na emissão); 503 `catalog_unavailable`.

## 6. Cidade (`municipal_admin`, step-up nas escritas)

- `GET/PUT /admin/city_profile` ganha `cnpj` (14 dígitos, DV válido; 422 `invalid_cnpj`).
- REMUME: `GET /clinical_documents/remume` → `{ "items": [ { catalog_item, unit_ids: [] } ] }`;
  `POST /clinical_documents/remume { catalog_item_id, unit_ids? }`;
  `DELETE /clinical_documents/remume/:catalog_item_id`.
- Protocolos: `GET /clinical_documents/nursing_protocols` → `{ "items": [ { id, title, number, year, current_version: { id, valid_from, valid_until?, items: [ { catalog_item, max_dose? } ] } } ] }`;
  `POST /clinical_documents/nursing_protocols { title, number, year, valid_from, valid_until?, items }`;
  `POST /clinical_documents/nursing_protocols/:id/versions { valid_from, valid_until?, items }`.

## 7. Página pública de conferência (sem login, host da cidade)

- `GET /v/:token` (HTML simples servido pelo api, ou JSON com `Accept: application/json`) →
  `{ "kind", "issued_at", "professional": { name, council }, "unit", "patient": { initials, birth_year }, "status": "valid"|"cancelled"|"awaiting_signature", "cancelled_at"?, "mode": "digital"|"paper", "simulated": bool, "signed_pdf_url"? }`.
- `POST /v/lookup { short_code, birth_year }` → mesma forma; 404 `not_found` para
  qualquer combinação errada (sem distinguir); 429 `rate_limited`.
- `GET /v/:token/signed.pdf` → PDF assinado (só modo digital e assinado).
- Nunca: CID, medicamentos, dias de afastamento, CPF.

## 8. Maintenance (GraphQL, token humano)

- Interruptor `clinical_documents` (mecanismo genérico; missing `clinical_record_disabled`).
- `medicationCatalog { currentRelease { id importedAt itemsCount hiddenCount } reviewItems(first) { catmatCode sourceDescription parseStatus } }`.
- `importMedicationCatalog` (mutation) → `{ ok, releaseId, added, changed, removed, errors }`;
  `importAnvisaLists` → `{ ok, antimicrobials, controlled, errors }`.

## 9. JSON canônico (`contracts`)

`clinical/clinical-document-v1.json` (JSON Schema 2020-12, RFC 8785), tag
`clinical-v1.1.0`: cabeçalho (`schema: "rotasaude.clinical_document.v1"`,
`city`, `unit`, `professional`, `patient`, `document`: `id`, `kind`,
`issued_at`, `replaces_document_id`) + `content` por tipo (§2), receita com
`catmat_code`, `catalog_release` e protocolo; mesmas convenções do
`consultation-v1` (UTC com `Z`, CPF só dígitos, texto vazio `null`, inteiros
onde for código). Vetor de canonicalização em `clinical/examples/canonical/`.

## 10. Eventos (só ids)

| Nome | Payload |
|---|---|
| `clinical_document.issued` | `document_id, kind, consultation_id|attendance_id, issue_mode` |
| `clinical_document.cancelled` | `document_id, kind` |
| `patient_medication.changed` | `patient_medication_id, kind, document_id|consultation_id` |
| `medication_catalog.imported` (plataforma) | `release_id, added, changed, removed` |

## 11. Ordem de entrega

1. `contracts` `clinical-v1.1.0`.
2. `api` (porta de dev sugerida 3038; migração de cidade a partir de 20261009600001; plataforma idem).
3. `dashboard` (Vite 5188). 4. `maintenance`.
