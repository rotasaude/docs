# Módulo 17 — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-05-module-17-scheduling-design.md` · **ADR:** `adr/0029.md`

Fonte única dos formatos usados por mais de um app. Mudar um formato começa
aqui. Erros: `{ "error": "<reason>" }`. Step-up: 401 `{ "error": "mfa_required" }`.
Respostas 200 de escrita são o objeto puro, sem envelope.

## 1. Schema de protocolo `protocols-v1.5.0` (MINOR)

```json
"scheduling": {
  "type": "array", "maxItems": 10,
  "items": { "type": "object", "additionalProperties": false,
    "required": ["when", "appointment_type", "priority", "due_in_days"],
    "properties": {
      "when": { "$ref": "#/$defs/condition" },
      "appointment_type": { "type": "string", "pattern": "^[a-z][a-z0-9_]{1,40}$" },
      "priority": { "enum": ["routine", "priority"] },
      "due_in_days": { "type": "integer", "minimum": 1, "maximum": 365 } } }
}
```
Variáveis do `when`: `outcome.*`, `profile.*`, ids de passo (gate do `api`).
Tipo inexistente na cidade: aviso do gate, sem bloquear.

## 2. Valores comuns

- `booking_kind`: `slot` | `fit_in` | `legacy`.
- Faixa do modelo: `{ "starts": "HH:MM", "ends": "HH:MM", "kind": "walk_in"|"bookable"|"blocked", "appointment_type_key"?: string, "slot_minutes"?: int }`.
- `priority`: `routine` | `priority`; `preferred_period`: `morning` | `afternoon` | `any`;
  `reschedule_reason_code`: `work` | `health` | `transport` | `other`.
- Datas `YYYY-MM-DD` no fuso da cidade; instantes ISO 8601 com fuso.

## 3. Profissionais (`/professionals`, `municipal_admin`; leitura também para quem lê profissionais hoje)

- `GET /professionals/appointment_types` → `{ "types": [ { "key", "name", "duration_minutes", "cbo_prefixes", "active", "origin" } ] }`.
- `POST /professionals/appointment_types` (criar, `origin=city`) e
  `POST /professionals/appointment_types/:key` (`name`?, `duration_minutes`?, `cbo_prefixes`? só em `city`, `active`?) → o tipo. 422 `invalid_key`, `key_taken`,
  `invalid_duration`, `invalid_cbo_prefixes`, `platform_type_locked`.
- `GET /professionals/schedule_templates` → `{ "templates": [ { "id", "name", "fit_in_limit", "blocks": [...], "active" } ] }`;
  `POST /professionals/schedule_templates` e `POST /professionals/schedule_templates/:id` → o modelo. 422 `invalid_blocks` (com `detail`: `overlap`, `missing_type`, `unknown_type`, `bad_time`), `invalid_fit_in_limit`.
- `POST /professionals/schedule_templates/preview` `{ blocks, fit_in_limit, sample: { starts_at, ends_at, cbo_code } }` →
  `{ "slots": [ { "starts_at", "ends_at", "appointment_type_key" } ], "blocks": [...] }` (nada gravado).
- Turno: `POST /professionals/links/:id/shifts` (existente) aceita `schedule_template_id`; `POST /professionals/shifts/:id/template` `{ schedule_template_id | null }`; `GET /professionals/:id/shifts` devolve `schedule_template_id`.
- Vínculo: `POST /professionals/links/:id/default_type` `{ appointment_type_key | null }`.
- Minha agenda (`health_professional`): `GET /professionals/me/agenda?from=&to=` (até 7 dias) → `{ "days": [ { "date", "shifts": [ { "shift_id", "unit", "blocks": [...], "appointments": [ <horário §4.4> ] } ] } ] }`.

## 4. Atendimento (`/attendance`, recepção — papéis que hoje marcam)

### 4.1 Fila
`GET /attendance/units/:id/requests` (existente) — cada pedido ganha
`appointment_type_key`, `appointment_type_name`, `priority`, `due_on`, `overdue`,
`origin` (`attendance`|`triage`), `reschedule_requested`, `reschedule_reason_code`,
`preferred_period`, `reschedule_count`, `needs_reschedule`; ordenação: atrasados,
`due_on`, prioridade. Nota livre do cidadão (`reschedule_note`) só aparece no
detalhe: `GET /attendance/requests/:id`.

`GET /attendance/requests/unassigned` → mesma forma, pedidos sem unidade.
`POST /attendance/requests/:id/assign_unit` `{ unit_id }` → o pedido; 409
`already_assigned`.

### 4.2 Vagas
`GET /attendance/units/:id/availability?type=<key>&from=YYYY-MM-DD&to=YYYY-MM-DD` (até 14 dias) →
`{ "slots": [ { "professional_id", "professional_name", "shift_id", "starts_at", "ends_at" } ],
   "legacy_days": ["YYYY-MM-DD"] }` (`legacy_days` = dias sem nenhum turno na unidade).

### 4.3 Marcar
`POST /attendance/requests/:id/appointments` (existente) aceita três formas:
- vaga: `{ "kind": "slot", "professional_id", "starts_at", "appointment_type_key" }`;
- encaixe: `{ "kind": "fit_in", "professional_id", "shift_id", "starts_at", "appointment_type_key", "reason" }` (`reason` ≥ 10);
- livre: `{ "kind": "legacy", "scheduled_at" }` (só em `legacy_days`).
Ausência de `kind` = `legacy` (compatibilidade). Erros: 409 `slot_taken`,
`slot_unavailable`, `citizen_busy`, `fit_in_limit`, `use_slots` (livre em dia com
turno); 422 `invalid_reason`, `type_not_served`, `outside_shift`.

### 4.4 Horário (forma única, em agenda, fila e Minha agenda)
```json
{ "id", "status", "booking_kind", "scheduled_at", "ends_at",
  "appointment_type_key", "appointment_type_name",
  "professional": { "id", "name" } | null, "shift_id", "fit_in": bool,
  "fit_in_reason"?, "outside_template": bool, "shift_cancelled": bool,
  "citizen": { "id", "cpf_masked", "name"? } }
```
`fit_in_reason` só para quem marca e para o `municipal_admin`.

### 4.5 Agenda da unidade
`GET /attendance/units/:id/agenda?date=` (existente) → `{ "date", "professionals": [ { "id", "name", "shifts": [ { "shift_id", "starts_at", "ends_at", "blocks": [...], "fit_in_count", "fit_in_limit" } ], "appointments": [ <§4.4> ] } ], "unassigned": [ <§4.4 legacy> ] }`.

## 5. Cidadão (`/citizen`)

- `GET /citizen/appointments` (existente): cada horário ganha `appointment_type_name`,
  `professional_name`, `unit: { name, address }`, `ends_at`,
  `can_request_reschedule`.
- `POST /citizen/appointments/:id/reschedule_request`
  `{ "reason_code", "note"? (≤ 200), "preferred_period" }` → 200 com o horário
  (`status = cancelled_by_citizen`); 409 `not_reschedulable`; 422
  `invalid_reason_code`, `invalid_period`, `note_too_long`.
- `GET /citizen/triages/:id` (existente): ganha
  `scheduling_request: { "unit_name": string|null, "due_on", "appointment_type_name" } | null`.
- `GET /citizen/notices` (existente): cada aviso ganha `kind`
  (`campaign`|`appointment_reminder`); lembrete =
  `{ "kind": "appointment_reminder", "id", "appointment_id", "appointment_type_name", "unit_name", "unit_address", "scheduled_at", "professional_name", "read_at" }`.
  `POST /citizen/notices/:id/read` vale para os dois.

## 6. Eventos (só ids)

| Nome | Payload |
|---|---|
| `appointment.booked` | `appointment_id, request_id, booking_kind` |
| `appointment.fit_in_created` | `appointment_id, shift_id` |
| `appointment.reschedule_requested` | `appointment_id, request_id` |
| `appointment.reminded` | `appointment_id, sms: bool` |
| `appointment_request.created_from_triage` | `request_id, triage_id` |
| `appointment_request.merged_triage` | `request_id, triage_id` |
| `appointment_type.changed` / `schedule_template.changed` | `key` / `template_id`, `user_id` |

## 7. Ordem de entrega

1. `contracts` `protocols-v1.5.0` (push com autorização).
2. `api` (porta de dev sugerida 3034).
3. `dashboard` e `wpda` depois do `api`.
