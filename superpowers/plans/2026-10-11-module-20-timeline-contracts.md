# Módulo 20 — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-11-module-20-patient-timeline-design.md` · **ADR:** `adr/0035.md`

Fonte única dos formatos entre `api`, `dashboard`, `wpda` e `maintenance`.
Convenções do projeto: `{ "error": "<reason>" }`; step-up 401 `mfa_required`;
interruptor 403 `{ "error": "feature_disabled", "feature": "patient_timeline" }`;
Content-Type JSON em toda escrita por cookie (inclusive DELETE); leitura do
profissional por `ClinicalRecord::Access` (generalizado para a pessoa).

## 1. Valores comuns

- `kind`: `triage` | `attendance` | `screening` | `appointment` | `consultation` |
  `clinical_document` | `controlled_notification` | `campaign`.
- `access` da trilha `clinical_record.viewed`: + `timeline` | `citizen` | `pec_external`.

## 2. Entrada

Profissional (`<entry>`):
```json
{ "id": "<kind>:<source_id>", "kind", "occurred_at", "unit": { "id", "name" }|null,
  "professional": { "name", "cbo_code" }|null, "title", "summary"|null, "status"|null,
  "detail_ref": { "type", "id" } }
```
Cidadão (`<citizen_entry>`): `{ "id", "kind", "occurred_at", "unit": { "name" }|null, "professional_name"|null, "title", "summary"|null, "status"|null, "document"?: { "id", "short_code", "verification_url", "downloadable": bool } }`.
Nunca no cidadão: texto clínico livre, CID sem nome, número SNCR, dado de outra pessoa.

## 3. Profissional (`/attendance`)

- `GET /attendance/timeline?patient_id=|attendance_id=|opening_id=&before=&limit=&kinds[]=&from=&to=`
  → `{ "person": { "display_name", "partial": bool, "patient_id"|null }, "entries": [<entry>], "next_before"|null }`.
  `limit` padrão 50, máximo 100; `before` é cursor opaco (`occurred_at|kind|id`).
  403 `out_of_context`, `opening_required`, `missing_role`; 422 `invalid_cursor`,
  `invalid_period`; 404 `not_found`.
- `partial: true` quando a cidade não está no modo `record`.

## 4. Cidadão (sessão do wpda)

- `GET /citizen/history?before=&limit=` → `{ "verified": bool, "partial": bool, "entries": [<citizen_entry>], "next_before"|null }`.
  Par só declarado: `verified: false`, só `triage` e `appointment` do próprio par.
- `GET /citizen/health_summary` → `{ "problems": [ { "terminology", "code", "label", "since"|null } ], "medications": [ { "label", "dosage_summary"|null, "since"|null } ], "exam_requests": [ { "sigtap_code", "label", "requested_at" } ], "vitals": { "blood_pressure"?: { "systolic", "diastolic", "measured_at" }, "weight_kg"?: { "value", "measured_at" }, "capillary_glucose"?: { "value", "moment", "measured_at" } } }`;
  403 `not_verified`; 404 `not_available` (sem paciente ou sem `clinical_record`).
- `GET /citizen/documents/:id/pdf` → `application/pdf` (assinado quando digital e
  assinado; senão o do papel); 404 `not_found` (documento de outra pessoa,
  cancelado ou inexistente — sem distinguir); 403 `not_verified`.

## 5. Página do PEC (sem sessão do Rota Saúde, host da cidade)

- `GET /pec/timeline?documentoCidadao=&documentoProfissional=&cnes=&pec_timestamp=&<assinatura>`
  (nomes exatos conferidos na Task 0 contra a documentação oficial) → HTML só
  leitura; cria sessão curta (cookie isolado, 15 min, `SameSite=None; Secure`).
  Falha → HTML genérico com código: `pec_disabled`, `pec_signature_invalid`,
  `pec_expired`, `pec_replayed`, `pec_unit_unknown`, `pec_professional_not_linked`,
  `pec_citizen_not_verified`. `Content-Security-Policy: frame-ancestors <pec_origin>`.
- `GET /pec/timeline/more?before=` (com a sessão curta) → fragmento HTML com a próxima página.

## 6. Maintenance (GraphQL, token humano)

- Interruptores `patient_timeline` e `pec_external_view` (missing:
  `patient_timeline_disabled`, `record_mode_not_integrated`).
- `cityPecConfig(citySlug) { pecOrigin, hmacKeyConfigured: Boolean!, hmacKeyUpdatedAt }`;
  mutations `setCityPecOrigin(citySlug, pecOrigin)` e
  `rotateCityPecHmacKey(citySlug, key)` → `{ ok, errors }` (a chave nunca volta).

## 7. Eventos (só ids)

| Nome | Payload |
|---|---|
| `clinical_record.viewed` | + `access: timeline/citizen/pec_external`, `person_ref` |
| `pec.external_view_opened` | `user_id, unit_id, patient_id|null` |
| `pec.external_view_denied` | `reason_code` |
| `city.pec_config_changed` (plataforma) | `city_id, field` |

## 8. Ordem de entrega

0. Conferir na documentação oficial dos "sistemas externos" do PEC o formato exato da chamada e da assinatura.
1. `api` (porta de dev sugerida 3040; migrações a partir de 20261011800001).
2. `dashboard` (Vite 5190) e `wpda` (Vite 5191). 3. `maintenance`.
