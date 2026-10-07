# Módulo 18 — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-07-module-18-screening-design.md` · **ADR:** `adr/0030.md`

Fonte única dos formatos usados por mais de um app; mudar formato começa aqui.
Erros `{ "error": "<reason>" }`; step-up 401 `{ "error": "mfa_required" }`;
escrita 200/201 devolve o objeto puro, salvo onde dito. Correção à spec §7: a
unidade é editada em `POST /attendance/units/:id` (rota existente), não em
`/health_units/:id`.

## 1. Schema de protocolo `protocols-v1.6.0` (MINOR)

Nova variante na raiz (via `oneOf` com a definição de triagem existente):

```json
{ "name": "acolhimento", "version": 1, "schema_version": "1.6.0",
  "kind": "screening",
  "risk_rules": [ { "when": <condição>, "color": "red"|"yellow"|"green"|"blue" } ] }
```
- `kind` ausente = triagem (compatível). `kind: "screening"` exige `risk_rules`
  (1–50) e proíbe `steps`, `scoring`, `offer`, `suggestions`, `scheduling`.
- Variáveis aceitas no `when` de `risk_rules` (o gate do `api` confere):
  `vitals.systolic`, `vitals.diastolic`, `vitals.heart_rate`,
  `vitals.respiratory_rate`, `vitals.temperature_c`, `vitals.spo2`,
  `vitals.capillary_glucose`, `vitals.glucose_moment`, `vitals.weight_kg`,
  `vitals.height_cm`, `vitals.bmi`, `vitals.pain_score`, `complaint.ciap2`,
  `profile.age`, `profile.sex`.
- Nome reservado da cidade: `acolhimento` (uma versão `active`).

## 2. Valores comuns

- Cor: `red` | `yellow` | `green` | `blue` (ordem de gravidade nessa ordem).
- Destino: `same_day` | `schedule` | `oriented` | `referred`.
- Estado da escuta: `in_progress` | `completed` | `abandoned`.
- `screening_scope` da unidade: `walk_in` | `all`.
- Desfechos novos do atendimento: `scheduled_from_screening`, `oriented`.
- `glucose_moment`: `fasting` | `postprandial` | `random`.

## 3. Escuta (`/attendance`, profissionais com vínculo e CBO permitido)

Forma da revisão (`revision`):
```json
{ "id", "created_at", "by": { "id", "name" },
  "ciap2": { "code", "label" }, "complaint_note"?,
  "vitals": { "systolic"?, "diastolic"?, "heart_rate"?, "respiratory_rate"?,
              "temperature_c"?, "spo2"?, "capillary_glucose"?, "glucose_moment"?,
              "weight_kg"?, "height_cm"?, "pain_score"?, "bmi"? },
  "alerts": [ "systolic_high", ... ],
  "suggested_color": cor|null, "final_color": cor, "color_change_reason"?,
  "matched_rules": [ { "index", "text" } ] }
```
Forma da escuta (`screening`):
`{ "id", "attendance_id", "status", "started_at", "completed_at", "destination", "orientation_note"?, "appointment_request_id"?, "current_revision": <revision>|null, "revisions_count" }`.

- `GET /attendance/units/:id/screening_queue` →
  `{ "items": [ { "attendance_id", "citizen": { "id", "cpf_masked" }, "checked_in_at", "triage_priority"|null, "screening": { "id", "status", "started_by_name" }|null } ] }`
  (por chegada; só quem precisa de escuta pelo escopo e não tem escuta `completed`).
- `POST /attendance/attendances/:id/screening` → 201 `<screening>` (`in_progress`). 409
  `already_screening`, `not_waiting`, `screening_not_required`; 403 `missing_link`,
  `cbo_not_allowed`.
- `POST /attendance/screenings/:id/abandon` → 200 `<screening>`; 409 `not_in_progress`.
- `POST /attendance/screenings/suggest` `{ ciap2_code, vitals, citizen_id }` →
  `{ "suggested_color", "matched_rules", "alerts", "bmi" }` (não grava).
- `POST /attendance/screenings/:id/complete`
  `{ ciap2_code, complaint_note?, vitals, final_color, color_change_reason?, destination,
     orientation_note?, schedule?: { appointment_type_key, priority, due_in_days },
     referral?: { referral_unit_id? , referral_note? } }` → 200 `<screening>`.
  422: `invalid_ciap2`, `implausible_vital` (com `field`), `bp_incomplete`,
  `invalid_color`, `color_change_reason_required`, `invalid_destination`,
  `orientation_required`, `invalid_schedule`, `referral_required`; 409
  `not_in_progress`, `attendance_not_waiting`.
- `POST /attendance/screenings/:id/reassess` `{ ciap2_code, complaint_note?, vitals, final_color, color_change_reason? }`
  → 200 `<screening>`; 409 `not_reassessable` (não `completed`, destino ≠
  `same_day` ou atendimento não `waiting`).
- `GET /attendance/screenings/:id` → `<screening>` com `revisions: [<revision>]`;
  publica `screening.viewed`. Recepção sem vínculo clínico: 403.

## 4. Fila do profissional

`GET /attendance/units/:id/queue` (existente): cada item ganha
`screening: { "color", "destination", "waited_minutes" } | null`; ordem: com
escuta `same_day` por cor e chegada, depois os demais como hoje. A recepção
recebe só esse bloco (sem queixa nem sinais). O detalhe do atendimento chamado
(rota existente da chamada) ganha `screening: <screening>` para o profissional.

## 5. Unidade

`POST /attendance/units/:id` (existente, `municipal_admin`) aceita
`screening_scope`; `GET /attendance/units` e `units/all` devolvem o campo.
422 `invalid_screening_scope`.

## 6. Produção (módulo 16)

- `GET /production/generation_failures?resolved=false` →
  `{ "items": [ { "id", "source_type", "source_id", "attendance_id", "reason_codes": [...], "created_at", "resolved_at" } ] }`.
- `POST /production/generation_failures/:id/retry` (step-up) → 200 com o item
  (resolvido ou com `reason_codes` atualizado); 409 `already_resolved`.
- `GET /production` (existente): cada ficha passa a ter `last_error_codes:
  [{ field, code }]` no lugar de `last_error`; `rejections` agrupa por
  `field`/`code`.

## 7. Eventos (só ids)

| Nome | Payload |
|---|---|
| `screening.started` / `screening.abandoned` | `screening_id, attendance_id, user_id` |
| `screening.completed` | `screening_id, attendance_id, destination, final_color` |
| `screening.reassessed` | `screening_id, revision_id, final_color` |
| `screening.viewed` | `screening_id, user_id` |
| `ledi.generation_failed` / `ledi.generation_retried` | `failure_id, source_type, source_id` |
| `ledi.payload_purged` | `count` (por execução, por cidade) |

## 8. Ordem de entrega

1. `contracts` `protocols-v1.6.0` (push com autorização).
2. `api` (porta de dev sugerida 3035).
3. `dashboard` depois do `api`.
