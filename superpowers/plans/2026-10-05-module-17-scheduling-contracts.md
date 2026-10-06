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

## 8. Acréscimos da escrita dos planos (2026-10-05)

Valem sobre as seções acima e sobre os planos.

**Cidadão (§5)**
- `GET /citizen/appointments`: os pedidos listados ganham `kind` (`return`|`referral`|`triage`),
  `target_unit_name: string|null`, `appointment_type_name`, `due_on`. O `api` não
  pode quebrar com pedido sem unidade (`item_json` hoje faz `request.target_unit.name`).
- Endereço da unidade (`unit.address` e `unit_address`) tem a mesma forma de
  `reference_units`: `{ street, number, complement, zip }`.
- `POST /citizen/appointments/:id/reschedule_request` responde
  `{ "appointment": ... }`, como `confirm` e `cancel` (exceção à regra do objeto
  puro, por consistência com as ações do cidadão que já existem).
- Lembrete na caixa de avisos: `read: boolean` (não `read_at`), `cpf_masked`
  quando o celular tem mais de uma pessoa; `unread_count` conta os lembretes com
  a mesma regra de `notices_muted`. O id do aviso é opaco; `POST
  /citizen/notices/:id/read` procura nas duas fontes.
- `when` de `scheduling` aceita só a condição estruturada (o mapa legado é
  recusado pelo schema).

## 9. Acréscimos do plano do dashboard (2026-10-06)

**Fila e pedido (§4.1)**
- O número de prioridade da triagem que a fila já devolve passa a se chamar
  `triage_priority`; `priority` é só `routine|priority`.
- `origin_unit_name` é nulo em pedido `kind = triage` (o `api` não pode fazer
  `r.origin_unit.name` sem guarda).
- `GET /attendance/requests/:id` = o item da fila + `reschedule_note`.

**Marcar (§4.3)**
- `health_unit_id` vai nas três formas (a recusa `wrong_unit` de hoje continua).
- Marcação livre mantém o comportamento atual: 409 `slot_taken` com `taken` e o
  `allow_overlap` (api#26).
- Resposta: 201 `{ "appointment": ... }`, como hoje.

**Horário e agenda (§4.4, §4.5, §3)**
- Em `legacy`: `ends_at`, `appointment_type_key/name`, `professional`, `shift_id` vêm `null`.
- Turno na agenda e em Minha agenda: `starts_at`, `ends_at`, `cancelled_at`.
- Faixas `bookable` trazem `appointment_type_name`.
- Minha agenda: `unit: { id, name }`; sem cadastro profissional → 404
  `no_profile`; `from`/`to` inclusivos (vale também para `availability`).

**Profissionais (§3)**
- Leitura de `GET /professionals/appointment_types` também para
  `protocol_author` e `protocol_reviewer`.
- `POST /professionals/shifts/:id/template` devolve o turno;
  `POST /professionals/links/:id/default_type` devolve o vínculo;
  `GET /professionals/:id` traz `default_appointment_type_key` em cada vínculo.
- 422 a mais: `invalid_name` (tipo e modelo), `invalid_blocks` com `detail`
  `empty`, `bad_slot_minutes` (fora de 5–240), `inactive_type`, `crosses_midnight`
  (fim depois do início, sem `24:00`); `cbo_prefixes`: 1 a 20 itens, cada um com 1
  a 6 dígitos.

**Gate (§1)**
- O aviso de tipo inexistente sai como no módulo 15: 200 com `warnings: [string]`.

## 10. Acréscimos do plano do api (2026-10-06)

- `invalid_blocks` ganha `detail: bad_block` (faixa malformada, chave ou `kind`
  desconhecido, tipo/`slot_minutes` em faixa não `bookable`, mais de 24 faixas).
- Códigos a mais: 422 `invalid_template` (turno com modelo inválido/inativo),
  `invalid_kind` (marcação), `invalid` (`active` não booleano; amostra inválida
  na pré-visualização); 409 `already_cancelled` (modelo de turno cancelado);
  `already_ended`, `type_not_served`, `inactive_type` (tipo padrão do vínculo);
  `invalid_unit`, `request_not_open` (`assign_unit`).
- Eventos a mais (só ids): `appointment_request.unit_assigned`,
  `professional.shift_template_set`, `professional.link_default_type_set`; a
  remarcação pela recepção usa o `appointment.moved` existente.
- Faixas devolvidas (agenda, pré-visualização) já recortadas pelo turno e pelo
  dia; recorte que chega à meia-noite sai como `"24:00"` (só na saída). Turno
  sem modelo = uma faixa `bookable` do tipo resolvido, ou nenhuma.
- A agenda da unidade mantém também a chave antiga `appointments` até o
  dashboard novo entrar.
- Item da fila ganha `target_unit_id` e `appointment` (§4.4 ou `null`);
  `reopened_reason` continua `expired|no_show|null`; a remarcação pedida aparece em
  `reschedule_requested`.
- `availability` com tipo inexistente/inativo: 200 `{ slots: [], legacy_days: [] }`;
  sem `from`/`to`: hoje + 6 dias.
- Leitura dos tipos também para `citizen_verifier` e `health_professional`.
- `citizen` no horário (§4.4) sem `name` (o cadastro não tem nome).
- Tipo do protocolo inexistente na cidade não impede o pedido: nasce com a
  chave, e o nome exibido é a própria chave.

## 11. Acréscimos da onda pré-merge (2026-10-06)

- `GET /citizen/triages/:id` → `scheduling_request` ganha `status` e
  `scheduled_at`:
  `{ unit_name: string|null, due_on: "YYYY-MM-DD", appointment_type_name: string, status: "open"|"scheduled", scheduled_at: string (ISO 8601 com fuso)|null } | null`.
  `scheduled_at` é o horário vivo mais recente do pedido (`scheduled` ou
  `confirmed`), ou `null`. Pedido encerrado continua `scheduling_request: null`.
  Depois de uma fusão que encurtou o prazo, `scheduled_at` pode cair depois de
  `due_on`: com `status: "scheduled"` o wpda não mostra prazo nem alerta de prazo.
  Api anterior sem os campos: o wpda trata como `status: "open"`, `scheduled_at: null`.
- Fusão de triagem em pedido `scheduled` cujo `due_on` passa a cair antes da data
  local do horário vivo: o pedido vira `needs_reschedule` e entra na fila (o
  horário não muda; a recepção decide). `overdue` também vale para pedido que
  precisa remarcar com `due_on` vencido. A forma do item da fila não muda.
- Exclusão do cadastro (`erase_pair`): cancela os horários vivos do par como
  `cancelled_by_citizen` com a frase fixa `"Exclusão do cadastro pedida pelo
  cidadão"` (evento `appointment.cancelled` com `by: "erasure"`, sem texto),
  fecha os pedidos vivos como `consent_revoked` e anula `reschedule_note` de todos
  os pedidos do par, inclusive encerrados (exceção mínima no guarda: só
  `reschedule_note → NULL` com as demais colunas iguais).
