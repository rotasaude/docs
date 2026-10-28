# Módulo 26a — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-28-module-26a-care-regulation-design.md` · **ADR:** `adr/0044.md`

Fonte única dos formatos entre `api`, `dashboard`, `wpda` e `maintenance`.
Convenções herdadas (contratos do 21a §15, 21b §14, 21c §11, 22 §12–§13, 23 §11–§12,
24a §14 e 25 §15): erros `{ "error" }`; step-up; interruptor 403 `feature_disabled`;
JSON em toda escrita por cookie; listas com envelope e `next_before` (cursor opaco
quando a ordem não é por id); registro de outro escopo por id → 404, lista com escopo
alheio → 403; leitura nominal com trilha e sem polling; contadores sem trilha; texto
longo recusado; dado sensível (justificativa, motivos, CID) só no corpo, nunca na URL
nem no log; todo Data/Struct com dado pessoal sem dado em `inspect`/`to_s`/
`pretty_print`/`as_json`; paciente sem CPF como no 24a (`has_cpf`, `cns_masked`).

## 1. Nomes no código

| Spec | Código |
|---|---|
| interruptor | `care_regulation` (`requires: ["feature:clinical_record"]`) |
| papel | `care_regulator` (privilegiado, step-up para conceder) |
| tipo do pedido | `specialty` (CBO) \| `procedure` (SIGTAP) |
| cor de risco | `red` \| `yellow` \| `green` \| `blue` (prazos padrão 7/30/90/180 dias) |
| situação | `draft` (rascunho na consulta) \| `pending` \| `returned` \| `authorized` \| `scheduled` \| `completed` \| `missed` \| `denied` \| `cancelled` \| `closed` (segunda falta) \| `external` (regulado pelo estado) |
| tipo de prestador | `own` (unidade da rede) \| `contracted` \| `consortium` \| `ppi` |
| sistema externo | `sisreg` \| `care` \| `cross` \| `other` |
| quem pede | médico `225*`; enfermeiro `2235*` (só a lista municipal); dentista `2232*` (só especialidades `2232*`) |

## 2. Formas

`<regulation_request>` (nominal; trilha): `{ "id", "patient": { "id", "display_name", "has_cpf", "cns_masked"|null }, "type", "target": { "code", "name" }, "cid10", "justification", "suggested_color", "color", "color_reason"|null, "status", "requested_at", "due_on", "overdue": bool, "requester": { "name", "cbo", "unit": { "id", "name" } }, "consultation_id", "authorization": { "offer_id", "provider": { "id", "name", "kind" }, "month", "skip_reason"|null, "at", "by": { "name" } }|null, "appointment_request_id"|null, "schedule": { "at", "place", "notes"|null }|null, "outcome"|null, "misses", "external": { "system", "number" }|null, "state_regulated": bool, "version", "events": [ { "status", "at", "by": { "name" }|null, "reason"|null } ] }`.

`<offer>`: `{ "id", "provider": { "id", "name", "kind" }, "target": { "type", "code", "name" }, "month": "YYYY-MM", "quantity", "used", "available" }`.

## 3. Na consulta (`health_professional` na consulta)

- `GET/POST /attendance/consultations/:id/regulation_requests`; `PATCH/DELETE .../regulation_requests/:rid` só em `draft`. Corpo: `{ "type", "target_code", "cid10", "justification", "suggested_color", "state_regulated"? }`. 403 `cbo_not_allowed` (§1); 422 `invalid_target`, `invalid_cid10`, `invalid_request` (com `path`).
- Finalizar a consulta (ou adendo) passa os `draft` para `pending` (ou `external` quando `state_regulated`), na mesma transação.
- `GET /regulation/requests/:id/guide.pdf` — guia de encaminhamento (solicitante e regulador; trilha; sem cache; `Content-Disposition` sem dado pessoal).

## 4. Fila do regulador (`care_regulator`)

- `GET /regulation/requests?status=&type=&target_code=&color=&overdue=&before=&limit=` → `{ "requests": [<regulation_request> sem `events`], "next_before" }` (ordem: cor, depois data do pedido; trilha uma por consulta).
- `GET /regulation/requests/:id` → `<regulation_request>`.
- `POST /regulation/requests/:id/authorize` — `{ "offer_id", "color"?, "color_reason"?, "skip_reason"?, "version" }`; 409 `quota_exhausted`, 422 `skip_reason_required` (há pedido elegível mais antigo de cor igual ou mais grave), 422 `color_reason_required`, 409 `stale_version`. Oferta própria cria pedido de agendamento do 17 (`kind: regulated`) na unidade.
- `POST /regulation/requests/:id/return` — `{ "reason", "version" }`; `deny` — `{ "reason", "version" }` (só antes de autorizar); `cancel` — `{ "reason", "version" }` (devolve a cota se ainda não atendido).
- `GET /regulation/requests/count` → `{ "pending", "overdue", "returned", "authorized", "external" }` (sem trilha).

## 5. Solicitante (unidade que pediu)

- `GET /regulation/returned?unit_id=` → devolvidos da unidade (trilha).
- `POST /regulation/requests/:id/complement` — `{ "justification", "cid10"?, "suggested_color"?, "version" }` → volta a `pending` mantendo a data do pedido.

## 6. Prestadores e ofertas (`municipal_admin`, escrita com step-up; leitura também `care_regulator`)

- `GET/POST/PATCH /regulation/admin/providers[/:id]` — `{ "name", "cnes", "kind", "health_unit_id"? (só `own`), "active" }`.
- `GET/POST/PATCH /regulation/admin/offers[/:id]` — `{ "provider_id", "target_type", "target_code", "month", "quantity" }`; `GET ?month=&target_code=` devolve `<offer>` com `used`/`available`. 422 `quantity_below_used`.
- `GET/PATCH /regulation/admin/deadlines` — `{ "red", "yellow", "green", "blue" }` (dias).
- `GET/POST/DELETE /regulation/admin/nurse_items` — `{ "target_type", "target_code" }`.

## 7. Agendamento e desfecho

- Rede própria: o pedido segue o pedido de agendamento do 17 (horário marcado → `scheduled`; atendimento → `completed`; falta/lapse → `missed`).
- Externo: `POST /regulation/requests/:id/schedule` — `{ "at", "place", "notes"?, "version" }`; `POST /regulation/requests/:id/outcome` — `{ "result": "completed"|"missed", "version" }`.
- Falta: primeira → volta a `pending` com `misses: 1` (cota não devolvida); segunda → `closed`.

## 8. Convivência com o estado (`care_regulator`)

- `POST /regulation/requests/:id/external` — `{ "system", "number", "version" }`.
- `external`: não entra na fila nem consome cota; o regulador lança `schedule` e `outcome`.
- `GET /regulation/exports/state.csv?status=` — CSV livre para digitar no sistema do estado (trilha; CNS só 4 últimos dígitos; sem CPF).

## 9. Cidadão (wpda, par validado)

`GET /citizen/regulation_requests?citizen_id=` → `{ "requests": [ { "id", "label", "status": "waiting"|"scheduled"|"returned"|"denied"|"closed"|"completed", "scheduled_at"|null, "place"|null, "updated_at" } ] }` — sem cor, posição, CID ou justificativa; 403 `not_verified`.

## 10. Maintenance

Interruptor `care_regulation` (rótulo e dependência, mecanismo genérico).

## 11. Eventos (só ids)

`regulation_request.created`, `.authorized`, `.returned`, `.denied`, `.cancelled`, `.scheduled`, `.completed`, `.missed`, `.closed`, `care_offer.changed`, `care_provider.changed`.

## 12. Ordem de entrega

0. Catálogo de alvos reguláveis (plano do api, Task 0): especialidades médicas e odontológicas (CBO 2002, `225*` e `2232*`) e procedimentos SIGTAP reguláveis (grupos 02, 03 e 04, sem os de APAC/OCI), conferidos com as tabelas já vendorizadas.
1. `api` (porta 3048; migrações de cidade a partir de 20261028900001; worktree `.claude/mod26a`, branch `feat/mod-26a-care-regulation`; testes com `bin/test api -w mod26a`).
2. `dashboard` (Vite 5203) e `wpda` (Vite 5204), worktrees em `rota-saude/.worktrees/<app>-mod26a`.
3. `maintenance` (prova na 5177 com o serviço do compose parado).
4. Catálogo público.
