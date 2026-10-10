# Módulo 19 (19d) — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-10-module-19d-controlled-prescriptions-design.md` · **ADR:** `adr/0034.md`

Estende o contrato do 19c (`2026-10-09-module-19c-documents-contracts.md`, §12
prevalece). Pré-requisito de execução: **19c entregue em origin/main**. Mesmas
convenções: `{ "error": "<reason>" }` (com `field`/`index`), step-up 401
`mfa_required`, interruptor 403 `{ "error": "feature_disabled", "feature":
"controlled_prescriptions" }`, Content-Type JSON em toda escrita por cookie
(inclusive DELETE), leitura por `ClinicalRecord::Access`.

## 1. Valores comuns

- `category` (receita): `common` | `special_control` | `antimicrobial`.
- `sncr.kind`: `rce` | `ret`. Estado do número: `free` | `used` | `voided`.
- `notification_type`: `A` | `B` | `B2`.
- Lista Portaria 344 no item do catálogo: `controlled_list` ∈ `A1`,`A2`,`A3`,`B1`,`B2`,`C1`,`C2`,`C3`,`C5`, mais `anticonvulsant: bool`.

## 2. Receita (estende o §2/§3 do 19c)

`content` da `prescription` ganha:
```json
{ "category": "special_control"|"antimicrobial"|"common",
  "sncr": { "kind": "rce"|"ret", "number": "<string>", "simulated": bool }|null,
  "patient_identification": { "cpf": "<11 dígitos>"|null, "no_cpf": bool, "passport"?: "<string>",
                               "address": { "street", "number", "complement"?, "district", "city", "uf", "zip" } }|null,
  "prescriber_contact": { "address": {…}, "phone": "<string>" }|null }
```
- `category` é calculada pelo api (a entrada não a envia; se enviar, é ignorada).
- `special_control`: `patient_identification` (com `address`; `cpf` ou `no_cpf`) e `prescriber_contact` obrigatórios; `antimicrobial` digital: `prescriber_contact` obrigatório, `patient_identification` opcional.
- `sncr` presente só no modo `digital` de `special_control`/`antimicrobial`; `null` em papel.
- Validade impressa: `valid_until` = emissão + 30 dias (`special_control`) / + 10 dias (`antimicrobial`).
- Erros novos em `POST /attendance/consultations/:id/documents`: 422
  `mixed_categories`, `requires_notification` (com `index`), `not_supported`
  (C2/C3, com `index`), `too_many_c1_substances`, `duration_exceeded` (com
  `index`), `patient_identification_required` (com `field`),
  `prescriber_address_missing`; 403 `cbo_not_allowed` (enfermeiro em
  `special_control`).
- Resposta da emissão traz `issue_mode` e, quando caiu para papel, `paper_reason`:
  `no_certificate` | `signature_unavailable` | `no_sncr_number` | `nurse_antimicrobial` | `feature_disabled`.

## 3. Registro da Notificação

- `POST /attendance/consultations/:id/documents { kind: "controlled_notification_record", content }`
  com `content`: `{ notification_type, paper_number, numbering_uf, item: { catalog_item: { id }, quantity, quantity_unit, route, dosage_instructions, duration_days } }`
  → 201 `<document>` (sem `short_code` nem `verification_url`; `issue_mode: "paper"`;
  `signature: null`). 422 `item_not_in_notification_list`,
  `notification_type_mismatch` (item A com tipo B etc.), `duration_exceeded`
  (A ≤ 30; B/B2 ≤ 60), `invalid_content` (com `field`); 403 `cbo_not_allowed`.
- Cancelamento pelo `POST /attendance/documents/:id/cancel` do 19c.

## 4. Estoque SNCR (usuário corrente)

- `GET /sncr/stock` → `{ "rce": { "free", "used", "voided", "requests_this_month" }, "ret": { … }, "low_threshold": 50, "simulated": bool }`.
- `POST /sncr/requests { kind, return_to }` → `{ "authorize_url" }`; 409
  `monthly_limit_reached`; 403 `cbo_not_allowed`; 409 `professional_cpf_missing`.
- `POST /sncr/oauth/callback { state, code }` (ou `{ state, error }`) →
  `{ "kind", "received": n, "batch_id", "return_to" }`; 422 `invalid_state`;
  409 `authorization_expired`; 403 `authorization_denied`; 503
  `sncr_unavailable`, `sncr_exhausted`.
- Rota de retorno do dashboard: `/dashboard/sncr/callback`.

## 5. Perfil do profissional

- `GET/PUT /attendance/professional_profile/contact` → `{ address: {…}, phone }`
  (o próprio usuário); 422 `invalid_address` (com `field`), `invalid_phone`.

## 6. Admin (`municipal_admin`)

- `GET /sncr/admin/overview` → `{ "professionals": [ { user_id, name, rce_free, ret_free, low: bool, last_request_at } ], "simulated": bool }`.

## 7. Página pública (estende o §7 do 19c)

- Ganha `category` e `sncr: { kind, number }` quando houver; nunca medicamento ou CPF.

## 8. Maintenance (GraphQL, token humano)

- Interruptores `controlled_prescriptions` (missing `clinical_documents_disabled`) e
  `sncr_mock` (missing `controlled_prescriptions_disabled`; só fora de produção —
  em produção não aparece no catálogo e a mutation responde `unknown_feature`).
- `sncrStatus { configured: Boolean!, reachable: Boolean!, lastCheckAt: ISO8601DateTime, simulatedAvailable: Boolean! }`.

## 9. JSON canônico (`contracts`)

`clinical/clinical-document-v1.json` ganha, na receita, `category`, `sncr`,
`patient_identification` e `prescriber_contact` (§2) — MINOR, tag
`clinical-v1.2.0`; `controlled_notification_record` não é assinável (sem
canônico). Vetor JCS com uma RCE e uma RET.

## 10. Eventos (só ids)

| Nome | Payload |
|---|---|
| `sncr.numbers_requested` | `user_id, kind, count, simulated` |
| `sncr.number_used` | `document_id, kind` |
| `sncr.number_voided` | `document_id, kind` |
| `controlled_notification.recorded` | `document_id, notification_type` |
| `clinical_document.issued` / `.cancelled` | + `category` |

## 11. Ordem de entrega

0. Prova técnica (troca do token gov.br do SNCR no servidor dentro de 30 s).
1. `contracts` `clinical-v1.2.0`.
2. `api` (porta de dev sugerida 3039; migrações a partir de 20261010700001; `fake-sncr` no compose, porta 8092).
3. `dashboard` (Vite 5189). 4. `maintenance`.
