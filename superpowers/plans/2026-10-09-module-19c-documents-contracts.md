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

## 12. Acréscimos da escrita dos planos (2026-10-10)

Valem sobre as seções acima e sobre os planos.

**Entrada e formas**
- Entrada do `content` = forma de saída do §2 sem os campos calculados: item com
  `catalog_item: { id }` ou `free_text`; `nursing_protocol: { version_id }`;
  `cid10: { code }`. A declaração não recebe dia nem unidade (vêm do
  atendimento). Horas `HH:MM` locais.
- A receita gravada e devolvida pela API traz `catalog_release` (id da release
  do catálogo na emissão; `null` só com texto livre), uma vez por receita.
- `max_dose` = `{ quantity, unit }` por item; unidade diferente da do protocolo →
  `above_protocol_max_dose`. Protocolo entra com `items: [{ catalog_item_id, max_dose? }]`;
  `current_version` ganha `version`.
- `short_code` formatado `XXXXX-XXXXX`; a entrada aceita minúsculas, espaço e
  hífen; `birth_year` com 4 dígitos.
- Declaração pela rota do atendimento sai sempre em papel; `signature: null` no
  papel puro; `author.cbo_code` nulo quando quem emite é a recepção. A recepção
  emite pela fila do dia (rota de atendimentos encerrados fica para depois).

**Rotas**
- CNPJ em `GET/PUT /clinical_documents/city_profile` → `{ name, cnpj }` (o
  `/admin` é só leitura); CNPJ numérico ou alfanumérico (IN RFB 2.229/2024),
  com DV.
- `POST /attendance/medications/search` aceita também `municipal_admin` (REMUME e
  protocolos). A enfermagem lê `GET /clinical_documents/nursing_protocols`.
- Leitura administrativa em `GET /clinical_record/consultations/:id/documents`.
- Página pública: `GET /v` (formulário), HTML sem `Accept: application/json`,
  `signed_pdf_url` só quando assinado e não cancelado; limites de tentativas
  10/60/30 por 10 min por IP.
- Com o interruptor desligado, cancelar continua permitido; REMUME, protocolos e
  CNPJ ficam só atrás do `clinical_record`.

**Erros novos**
- 503 `catalog_unavailable` também na emissão de receita; 409
  `awaiting_signature` e `already_cancelled` no impresso; 422
  `invalid_catalog_item` e `invalid_unit` na REMUME; 422 `invalid_medication` e
  409 `not_draft` nos medicamentos (consulta finalizada); `invalid_item` com
  `field` além de `index`; 409 `already_exists` para protocolo com número e ano
  repetidos; `invalid_content` com `field: "replaces_document_id"`.
- 19b: `reason_code` novo `document_cancelled` (pedido de documento cancelado
  sai da fila); a volta ao papel passa o documento a `paper`.

**Maintenance**
- Tipos `MedicationCatalog`, `MedicationCatalogRelease`, `MedicationCatalogItem`,
  `reviewItems(first: Int = 50)`; revisão só leitura. Importações síncronas, com
  `import_in_progress` (release `importing` presa há mais de 1 h vira `failed`).
  Em produção (sem maintenance), carga pelos rakes `medications:import_anvisa`
  e `medications:import_catmat`.
- Eventos novos: `anvisa_lists.imported`, `maintenance.medication_catalog.imported`,
  `maintenance.anvisa_lists.imported`; `patient_medication.changed` aceita
  também `addendum_id`.

**JSON canônico (`clinical-v1.1.0`)**
- Caminho `clinical/clinical-document-v1.json`; `patient` e `council`
  obrigatórios; campo que não se aplica fica ausente; `note` sempre (`null`
  vazio); `catmat_code` inteiro; `free_text` string; `nursing_protocol { id,
  title, number, year, version_id }`; declaração da recepção não é assinada nem
  tem canônico; vetor JCS com dois arquivos novos em `clinical/examples/canonical/`.
- **Decidido pelo usuário (2026-10-10):** `catalog_item.dosage_form` aceita
  `null` já na `clinical-v1.1.0` (o CATMAT não traz a forma em boa parte dos
  itens); a chave continua presente; a receita assina normalmente e o impresso
  usa a descrição original.

**Base e go-live**
- CATMAT conferido em chamada real: envelope `{ resultado, totalRegistros,
  totalPaginas, paginasRestantes }`, `tamanhoPagina=500`, 6.778 itens; o
  analisador entende ~4.100 e manda ~2.680 para revisão.
- QR code com `rqrcode_core` (Ruby puro, desenhado em vetor no Prawn).
- As listas Anvisa (antimicrobianos e controlados) não têm API aberta: entram
  como arquivos transcritos da fonte oficial e precisam de conferência por
  farmacêutico antes de ligar o interruptor (controlado faltando na lista passa
  na receita).
- **contracts entregue (ea08b05, antes da tag):** vetor JCS com **três**
  arquivos novos (o terceiro, `prescription-dosage-form-null.jcs`, mantém os
  hashes fixados no plano do api); `catalog_release` segue string livre (o api
  grava o uuid da release; exemplos usam "2026-10-09"); atestado de
  acompanhante proíbe `days`; 51 exemplos (12 válidos, 39 inválidos).
