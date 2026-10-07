# Módulo 19 (19a) — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-07-module-19-consultation-design.md` · **ADR:** `adr/0031.md`

Fonte única dos formatos entre `api` e `dashboard` (o `maintenance` só ganha a
chave `clinical_record` pelo mecanismo genérico do ADR 0028). Erros
`{ "error": "<reason>" }`; step-up 401 `{ "error": "mfa_required" }`;
interruptor desligado 403 `{ "error": "feature_disabled", "feature": "clinical_record" }`;
escrita devolve o objeto puro.

## 1. Valores comuns

- Problema: `{ "id", "terminology": "ciap2"|"cid10", "code", "label", "status": "active"|"resolved", "onset_on"|null, "onset_precision": "day"|"month"|"year"|null, "resolved_on"|null }`.
- Sinais vitais: mesma forma `vitals` do contrato do módulo 18.
- Condutas e tipos de atendimento: códigos fixados pela Task 1 do plano do api
  (`config/ledi/consultation_mapping.yml`); o `api` expõe a lista em §4.
- Nome de exibição: `display_name` = nome social se houver, senão nome completo.

## 2. Validação presencial (existente, muda)

`POST /attendance/verifications` aceita `full_name` (obrigatório, 3–200),
`social_name` (opcional, ≤ 200), `mother_name` (opcional, ≤ 200). 422
`invalid_full_name`, `invalid_social_name`, `invalid_mother_name`.
`POST /attendance/lookup` devolve `citizen.names: { full_name_set: bool,
display_name }` (nunca os três valores para quem só faz balcão sem validar).
`POST /attendance/verifications/:id/names` (completar nomes de par já validado,
papel `citizen_verifier`) `{ full_name, social_name?, mother_name? }` → o par.

## 3. Paciente e prontuário

Forma do prontuário (`record`):
```json
{ "patient": { "id", "display_name", "full_name", "social_name"|null, "age", "sex", "cpf_masked" },
  "access": "in_context"|"justified",
  "problems": [ <problema> ],
  "today_screening": <screening do módulo 18>|null,
  "consultations": [ { "id", "finalized_at", "author_name", "cbo_label", "care_type_label",
                       "problems": [ <problema avaliado> ], "addenda_count" } ] }
```
- `GET /attendance/attendances/:id/record` → `<record>` em contexto; 403
  `out_of_context`, `missing_role`; 409 `citizen_not_verified`.
- `GET /clinical_record/patients/:id` → `<record>` só com abertura válida; 403
  `opening_required`.
- `POST /clinical_record/openings` (step-up) `{ cpf, reason_code, reason_note? }`
  → `{ "opening_id", "patient_id", "expires_at" }`; 404 `patient_not_found`; 422
  `invalid_reason`.
- `GET /clinical_record/openings?from=&to=&user_id=` (`municipal_admin`) →
  `{ "items": [ { "id", "user_name", "cpf_masked", "reason_code", "created_at", "expires_at" } ] }`.

## 4. Consulta

Forma (`consultation`):
```json
{ "id", "attendance_id", "patient_id", "status": "draft"|"finalized",
  "author": { "id", "name" }, "cbo_code",
  "subjective", "objective", "assessment", "plan", "vitals": {...},
  "care_type", "evaluated_problems": [ { "problem_id"|null, "terminology", "code", "label",
                                         "action": "evaluate"|"add"|"resolve"|"correct_onset",
                                         "onset_on"?, "onset_precision"? } ],
  "conducts": [ "<código>" ], "exam_requests": [ { "sigtap_code", "label", "cid10_justification"? } ],
  "started_at", "finalized_at"|null,
  "addenda": [ { "id", "author_name", "created_at", "reason", "text", "changes": {...} } ] }
```
- `GET /attendance/consultation_options` → `{ "care_types": [{code,label}], "conducts": [{code,label}], "cid10_allowed_for_cbo": bool }` (para o usuário corrente).
- `POST /attendance/attendances/:id/consultation` → 201 `<consultation>` (`draft`);
  409 `citizen_not_verified`, `not_in_care`, `not_caller`, `already_exists`; 403
  `cbo_not_allowed`, `feature_disabled`.
- `PATCH /attendance/consultations/:id` (autosave; só o autor; só `draft`) → 200
  `<consultation>`; 409 `not_draft`; 403 `not_author`; 422 `implausible_vital`
  (com `field`), `text_too_long` (com `field`; 20.000 por campo).
- `POST /attendance/consultations/:id/finalize` `{ outcome: <corpo do close existente> }`
  → 200 `<consultation>`; 422 `no_problem_evaluated`, `no_conduct`,
  `assessment_or_plan_required`, `patient_name_missing`,
  `cid10_not_allowed_for_cbo`, mais os erros do close existente.
- `GET /attendance/consultations/:id` → `<consultation>` (rascunho só para o autor;
  finalizada em contexto ou abertura); trilha `clinical_record.viewed`.
- `POST /attendance/consultations/:id/addenda` `{ reason, text, changes?, opening_id? }`
  → 201 com o adendo; 409 `not_finalized`; 403 `opening_required`; 422
  `invalid_reason`.
- `GET /attendance/consultations/:id/print` → `application/pdf`; 409
  `not_finalized`, `patient_name_missing`.

## 5. Busca de terminologia

`POST /attendance/ciap2/search` (módulo 18) passa a aceitar
`{ q, terminology: "ciap2"|"cid10" }` (padrão `ciap2`); mesma resposta.
`POST /attendance/sigtap/search { q }` → `{ items: [{ code, label }] }` (competência
ativa); 503 `terminology_unavailable`.

## 6. Produção (módulo 16/18)

Ficha `correction_pending` aparece em `GET /production` com
`status: "correction_pending"` e não conta como pendente de envio.

## 7. Eventos (só ids)

| Nome | Payload |
|---|---|
| `patient.created` / `patient.linked` | `patient_id, citizen_id` |
| `patient_problem.changed` | `patient_problem_id, kind, consultation_id|addendum_id` |
| `consultation.started` / `consultation.finalized` | `consultation_id, attendance_id` |
| `consultation.addendum_added` | `consultation_id, addendum_id` |
| `clinical_record.viewed` | `patient_id, user_id, access, reason_code` |
| `clinical_record.opened` | `opening_id, patient_id, user_id, reason_code` |

## 8. Ordem de entrega

1. `api` (porta de dev sugerida 3036).
2. `dashboard` depois do `api`.
3. Interruptor `clinical_record` ligado pelo maintenance só em dev/staging.
