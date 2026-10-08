# Módulo 19 (19b) — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-08-module-19b-digital-signature-design.md` · **ADR:** `adr/0032.md`

Fonte única dos formatos entre `api`, `dashboard`, `maintenance` e `signer`.
Erros `{ "error": "<reason>" }`; step-up 401 `{ "error": "mfa_required" }`;
interruptor desligado 403 `{ "error": "feature_disabled", "feature": "digital_signature" }`;
escrita devolve o objeto puro. Executa depois do 19a fechado: as formas
`<consultation>` e `<record>` são as do contrato do 19a (com o §9 dele).

## 1. Valores comuns

- `provider`: `vidaas` | `birdid` | `safeid` | `neoid` | `remoteid`.
- Estado do pedido: `pending` | `signed` | `failed` | `returned_to_paper`.
- Modo do documento: `digital` | `manual` | `pending`.
- Validação: `valid` | `invalid` | `indeterminate`.
- Motivos (`reason_code`): `no_session`, `session_expired`, `provider_unavailable`,
  `provider_rejected`, `signer_unavailable`, `verification_failed`,
  `certificate_expired`, `certificate_revoked`, `feature_disabled` (este só em
  `returned_to_paper`), `user_request` (volta ao papel pelo autor).
- `document_type`: `consultation` | `consultation_addendum`.

## 2. Bloco `signature` na consulta e no adendo

`<consultation>` e cada item de `addenda` ganham:
```json
"signature": { "mode": "digital"|"manual"|"pending",
               "request_id"?, "signature_id"?, "signed_at"?, "signer_name"?,
               "verification"?: "valid"|"invalid"|"indeterminate",
               "reason_code"? }
```
`manual` quando o interruptor está desligado, o autor não tinha certificado ativo
na finalização, ou o pedido voltou ao papel. Rascunho não tem o bloco.

## 3. Certificado (`/signature/certificates`, usuário corrente)

Forma `<certificate>`: `{ "id", "provider", "issuer", "serial_number", "not_after", "status", "expires_in_days" }`.
- `GET /signature/certificates/current` → `<certificate>` ou 404 `certificate_not_linked`.
- `POST /signature/certificates/discover` → `{ "providers": [ { "provider", "found": bool } ], "unavailable": [ "<provider>" ] }`.
- `POST /signature/certificates/link { provider, return_to }` (step-up) → `{ "authorize_url" }`; 422 `invalid_provider`.
- `DELETE /signature/certificates/current` (step-up) → 204.

## 4. OAuth e sessão

- `POST /signature/sessions { return_to }` → `{ "authorize_url" }`; 409 `certificate_not_linked`.
- `GET /signature/sessions/current` → `{ "active": bool, "expires_at"?, "provider"? }`.
- `DELETE /signature/sessions/current` → 204.
- `POST /signature/oauth/callback { state, code }` → `{ "purpose": "link"|"session"|"batch", "result": <certificate> | { "expires_at" } | <batch> }`;
  422 `invalid_state`; 409 `authorization_expired`; 403 `authorization_denied`;
  422 `certificate_cpf_mismatch`, `certificate_expired`, `certificate_revoked`;
  503 `provider_unavailable`.
- `return_to` é caminho relativo do dashboard (`/^\/[^\/]/`); a rota de retorno
  do dashboard é `/signature/callback?state=&code=` (ou `error=`).

## 5. Pendentes e lote

Forma `<request>`: `{ "id", "document_type", "document_id", "consultation_id", "patient_display_name", "finalized_at", "status", "reason_code"?, "attempts" }`.
- `GET /signature/requests?status=pending` → `{ "items": [<request>] }` (só do autor, até 200, mais antigas primeiro).
- `POST /signature/requests/:id/return_to_paper { reason }` → `<request>`; 422 `invalid_reason` (< 10); 403 `not_author`; 409 `not_pending`.
- `POST /signature/batches { request_ids?, return_to }` (sem ids = todas pendentes, até 50) → `{ "authorize_url", "count" }`; 409 `nothing_pending`, `certificate_not_linked`.
- Resultado do lote (no callback): `<batch>` = `{ "signed": n, "failed": [ { "request_id", "reason_code" } ] }`.

## 6. Assinatura

- `GET /signature/signatures/:id` → `{ "id", "document_type", "document_id", "signed_at", "signer_name", "signer_cpf_masked", "policy": "AD-RB", "verification", "verification_reasons": [], "verified_at", "content": <JSON canônico legível> }`;
  trilha `clinical_record.viewed`; 403 `out_of_context` como no 19a.
- `GET /signature/signatures/:id/pdf` → `application/pdf` (PAdES).
- `GET /signature/signatures/:id/package` → `application/zip` (`document.json` + `document.json.p7s`).
- `POST /signature/signatures/:id/verify` → mesma forma do GET, revalidada.
- Impresso do 19a: `GET /attendance/consultations/:id/print` devolve o PDF
  assinado quando o modo é `digital` (consulta sem adendo assinado depois) e o
  impresso do 19a nos demais casos.

## 7. Admin (`municipal_admin`)

`GET /signature/admin/overview?from=&to=` →
```json
{ "professionals": [ { "user_id", "name", "certificate_status": "active"|"none"|"expiring", "not_after"?, "pending_count", "oldest_pending_at"? } ],
  "documents_by_mode": { "digital", "manual", "pending" },
  "invalid_or_indeterminate": [ { "signature_id", "document_type", "signer_name", "verification", "verified_at" } ] }
```
`expiring` = vence em ≤ 30 dias; período no fuso da cidade, padrão 30 dias.

## 8. Maintenance

- Interruptor `digital_signature` pelo mecanismo genérico (`missing`:
  `clinical_record_disabled`).
- Leitura nova na API GraphQL de manutenção: `signatureProviders { key configured lastCheckAt lastCheckOk }`
  e `signerStatus { reachable version crlUpdatedAt }` (plataforma, sem segredo).

## 9. Serviço `signer` (interno; api/worker → signer)

`Authorization: Bearer <SIGNER_TOKEN>`; JSON; corpo nunca logado.
- `POST /prepare { kind: "cades"|"pades", document_base64, certificate_der_base64, policy: "AD-RB" }`
  → `{ "to_be_signed_sha256_base64", "prepared_state" }` (estado opaco, base64).
- `POST /assemble { kind, prepared_state, signature_value_base64 }` →
  `{ "signature_base64", "validation_material_base64" }` (p7s destacado ou PDF assinado).
- `POST /verify { kind, document_base64?, signature_base64 }` →
  `{ "status": "valid"|"invalid"|"indeterminate", "signer_cpf", "signer_name", "policy_oid", "signed_at", "reasons": [] }`.
- `GET /health` → `{ "version", "crl_updated_at" }`.
- Erros: 400 `invalid_request`, 401 `unauthorized`, 422 `invalid_certificate`,
  `invalid_signature_value`, 500 `internal`.

## 10. JSON canônico (`contracts`)

`schemas/clinical/consultation-v1.json` e `consultation-addendum-v1.json`
(JSON Schema 2020-12), serializados em RFC 8785. Campos: `schema`, `city`
(`ibge_code`, `name`), `unit` (`cnes`, `name`), `professional` (`name`, `cpf`,
`cbo_code`, `council`), `patient` (`display_name`, `cpf`, `birth_date`),
`consultation` (`id`, `started_at`, `finalized_at`, `care_type`, S/O/A/P,
`vitals`, `evaluated_problems` com `terminology`/`code`/`release`/`action`,
`conducts`, `exam_requests`, `outcome`); adendo: `consultation_id`, `id`,
`created_at`, `reason`, `text`, `changes`, `previous_sha256`. Tag
`clinical-v1.0.0`.

## 11. Eventos (só ids)

| Nome | Payload |
|---|---|
| `signature.certificate_linked` / `signature.certificate_unlinked` | `certificate_id, user_id, provider` |
| `signature.session_opened` | `session_id, user_id, provider` |
| `signature.signed` | `signature_id, request_id, document_type, document_id` |
| `signature.failed` | `request_id, reason_code` |
| `signature.returned_to_paper` | `request_id, reason_code` |
| `signature.verified` | `signature_id, verification` |

## 12. Ordem de entrega

0. Prova técnica (Task 0 do plano do `signer`).
1. `contracts` `clinical-v1.0.0`.
2. `signer` (repositório novo, com autorização).
3. `api` (porta de dev sugerida 3037; migrações de cidade acima das do 19a).
4. `dashboard`. 5. `maintenance`.
