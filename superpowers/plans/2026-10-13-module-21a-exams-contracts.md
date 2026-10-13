# Módulo 21a — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-13-module-21a-exam-tracking-design.md` · **ADR:** `adr/0036.md`

Fonte única dos formatos entre `api`, `dashboard`, `wpda`, `maintenance`,
`contracts` e o laboratório simulado. Convenções do projeto: `{ "error": "<reason>" }`;
step-up 401 `mfa_required`; interruptor 403 `{ "error": "feature_disabled", "feature": "<key>" }`;
Content-Type JSON em toda escrita por cookie (inclusive DELETE); leitura clínica
por `ClinicalRecord::Access`, com trilha.

## 1. Nomes no código (a spec usa os termos em português)

| Spec | Código |
|---|---|
| papel `regulador` / `prestador` | `Membership` `exam_regulator` / `exam_provider` (o `exam_provider` liga o usuário a um `exam_providers.id` por `exam_provider_users`) |
| `kind` do prestador | `municipal` \| `accredited` \| `consortium` \| `lacen` \| `imaging` |
| `release_policy` | `immediate` \| `after_acknowledgement` \| `never_citizen` |
| `priority` | `routine` \| `urgent` |
| `origin` do exame | `consultation` \| `reception_guide` \| `pec` |
| eventos da trilha | `requested` \| `authorized` \| `denied` \| `scheduled` \| `collected` \| `resulted` \| `acknowledged` \| `escalated` \| `cancelled` |
| motivo do escalonamento | `overdue` \| `critical` \| `unlinked_requester` |
| `origin` do resultado / `source` | `order` \| `external` / `unit` \| `provider` \| `fhir` \| `consultation` |

`status` do exame (cache da trilha): `awaiting_authorization` | `denied` |
`awaiting_schedule` | `scheduled` | `collected` | `resulted` | `acknowledged` | `cancelled`.

Etapas com prazo (`stage`): `authorize` | `schedule` | `collect` | `result` | `acknowledge`.
Padrão da plataforma: 15, 15, 30, 30, 15 dias. `urgent` = `ceil(prazo / 2)`.
Prazo da etapa conta desde o evento que a abriu (`requested`/`authorized`/`scheduled`/`collected`/`resulted`).

## 2. Formas comuns

`<exam_order>` (profissional, unidade, regulador):
```json
{ "id", "short_code", "patient": { "id", "display_name" }, "sigtap_code", "sigtap_competence",
  "label", "priority", "cid10_justification"|null, "origin",
  "requester": { "user_id"|null, "name", "cns"|null }, "unit": { "id", "name" },
  "provider": { "id", "name" }|null, "scheduled_at"|null, "scheduled_place"|null,
  "external_reference"|null, "status", "status_changed_at",
  "overdue": { "stage", "due_on", "days_late" }|null, "escalated": bool,
  "result": <exam_result>|null, "events": [<exam_event>] }
```
`<exam_event>`: `{ "kind", "occurred_at", "actor": { "type": "user"|"provider", "name" }, "reason"|null }`.
`<exam_result>`: `{ "id", "origin", "source", "released_on", "laboratory_name", "critical",
"values": [ { "key"|null, "label", "value", "unit"|null, "loinc"|null } ], "pdf_url", "acknowledged_at"|null }`.
`pdf_url` é curta e assinada (5 min); o download grava trilha.

`<provider_order>` (portal do prestador; nunca prontuário):
`{ "id", "short_code", "patient": { "name", "birth_date", "cpf_last3" }, "sigtap_code", "label", "priority", "status", "scheduled_at"|null, "scheduled_place"|null, "overdue"|null }`.

## 3. Rotas do profissional e da unidade (`/exams`, sessão do dashboard)

- `GET /exams/orders?patient_id=|unit_id=&status[]=&overdue=true&stage=&provider_id=&sigtap_code=&before=&limit=` → `{ "orders": [<exam_order sem events>], "next_before"|null }`.
- `GET /exams/orders/:id` → `<exam_order>`.
- `POST /exams/orders` (guia da recepção; `health_professional` ou `citizen_verifier` da unidade) — `{ "patient_id", "sigtap_code", "priority", "cid10_justification"?, "requester": { "user_id" } | { "name", "cns" }, "external_reference"?, "guide_upload_id" }` → 201 `<exam_order>`. 422 `sigtap_not_in_catalog`, `invalid_cns`, `guide_missing`.
- `POST /exams/uploads` (multipart, PDF/JPEG/PNG ≤ 10 MB) → `{ "upload_id" }`; 422 `invalid_file`.
- `POST /exams/orders/:id/schedule` — `{ "provider_id", "scheduled_at", "scheduled_place" }`; `POST …/collect`; `POST …/cancel` — `{ "reason" }`.
- `POST /exams/orders/:id/result` — `{ "upload_id", "released_on", "laboratory_name", "critical", "values": [ { "key", "value" } ] }`.
- `POST /exams/orders/:id/acknowledge` — `{ "reason"? }` (obrigatório quando quem dá ciência não é o solicitante e o exame está escalado).
- `GET /exams/acknowledgements/pending` → exames do solicitante aguardando ciência; `GET /exams/acknowledgements/team` → fila da equipe.
- `POST /exams/external_results` — `{ "patient_id", "sigtap_code", "released_on", "laboratory_name", "upload_id", "values", "consultation_id"? }` → 201 `<exam_result>`.
- Erros de transição: 409 `invalid_transition` (`{ "error", "status" }`); 422 `reason_required`, `provider_not_enabled_for_exam`, `result_already_registered`; 403 `missing_role`, `not_requester`, `not_in_team`, `out_of_context`, `opening_required`.

## 4. Regulação (`/exams/regulation`, `exam_regulator` ou `municipal_admin`)

- `GET /exams/regulation/queue?before=&limit=` → `{ "orders": [<exam_order sem events>], "next_before" }` (urgente primeiro, depois `requested` mais antigo).
- `POST /exams/orders/:id/authorize` — `{ "provider_id" }`; `POST /exams/orders/:id/deny` — `{ "reason" }`.

## 5. Catálogo e prestadores (`/exams/admin`, `municipal_admin`; escrita com step-up)

- `GET/POST/PATCH /exams/admin/catalog_items[/:id]` — `{ "sigtap_code", "requires_authorization", "release_policy", "deadlines": { "<stage>": int|null }, "result_fields": [ { "key", "label", "unit"|null, "loinc"|null } ], "active" }`; 422 `sigtap_unknown` (fora da competência vigente do SIGTAP da plataforma), `duplicate_sigtap`, `invalid_deadline`.
- `GET/POST/PATCH /exams/admin/providers[/:id]` — `{ "name", "cnes"|null, "kind", "active", "items": [ { "catalog_item_id", "deadlines": { "<stage>": int|null } } ] }`.
- `POST /exams/admin/providers/:id/fhir_token` → `{ "token" }` (mostrado uma vez; troca o anterior); `DELETE …/fhir_token` revoga.
- `POST /exams/admin/providers/:id/users` — convite do usuário `exam_provider` (mecanismo de convite existente).
- `GET /exams/admin/report?provider_id=&month=YYYY-MM` → `{ "rows": [ { "provider": { "id", "name" }, "authorized", "collected", "resulted_on_time", "resulted_late", "pending" } ] }`; `GET …/report.csv?detail=true` → CSV com `short_code, sigtap_code, authorized_on, scheduled_on, collected_on, resulted_on, on_time`.

## 6. Portal do prestador (`/exams/provider`, `exam_provider`)

- `GET /exams/provider/orders?status[]=&overdue=true&before=&limit=` → `{ "orders": [<provider_order>], "next_before" }`, `{ "overdue_count" }` no cabeçalho de resposta `X-Overdue-Count`.
- `POST /exams/provider/orders/:id/schedule|collect|result` — mesmos corpos do §3.
- Guia de outro prestador: 404 `not_found` (sem distinguir de inexistente).

## 7. Entrada FHIR (host da cidade, sem cookie)

- `POST /fhir/DiagnosticReport`, `Authorization: Bearer <token>`, `Content-Type: application/fhir+json`.
- Perfil mínimo (`contracts/exams/fhir-diagnostic-report-inbound.json`): `resourceType`, `identifier[0].value`, `status` (`final` | `amended`), `basedOn[0].identifier.value` = `short_code`, `subject.identifier` (sistema CPF `https://saude.gov.br/fhir/r4/NamingSystem/cpf` ou CNS `…/cns`), `issued`, `performer[0].display`, `presentedForm[0]` (`contentType: application/pdf`, `data` base64), `contained[]` de `Observation` (`code.coding[]` com `system: http://loinc.org`, `valueQuantity` ou `valueString`, `interpretation`).
- Respostas: 201 `{ "result_id" }`; 200 com o mesmo `result_id` quando o `identifier` já foi recebido daquele prestador; 401 `invalid_token`; 403 `feature_disabled`; 404 `order_not_found` (inexistente ou de outro prestador); 409 `invalid_transition`; 422 `patient_mismatch`, `missing_pdf`, `invalid_resource` (com `details` só de caminho, nunca valor); 429 `rate_limited` (60/min por token).
- `interpretation` `HH`, `LL` ou `AA` em qualquer `Observation` → `critical: true`.
- `GET /fhir/ServiceRequest?status=active` (mesmo token) → `Bundle` `searchset` com as guias agendadas ou coletadas do prestador: `identifier` = `short_code`, `code` SIGTAP, `subject.identifier` (CPF ou CNS), `priority`. É o que o laboratório (e o `fake-lab`) usa para saber o que fazer.

## 8. Ponte do PEC (`exam_pec_bridge`)

- `GET /pec/exams/new?<mesmos parâmetros assinados da ponte do módulo 20>` → HTML "Solicitar exames" (sessão curta própria, cookie isolado, 15 min); `POST /pec/exams` (form, token CSRF da sessão curta) cria `exam_order` com `origin: pec`. Erros: os `pec_*` do módulo 20 + `pec_exam_bridge_disabled`.
- Só se a prova técnica (plano do api, Task 0) confirmar envio de formulário dentro da tela de sistemas externos.

## 9. Cidadão (sessão do wpda, só par validado)

- `GET /citizen/exams?before=&limit=` → `{ "verified": bool, "exams": [ { "id", "label", "status_label", "requested_at", "scheduled_at"|null, "scheduled_place"|null, "result": { "available": bool, "pickup_at_unit": bool, "released_on"|null, "pdf_available": bool }|null } ], "next_before" }`.
- `GET /citizen/exams/:id/pdf` → `application/pdf` só se a política permitir; senão 404 `not_found`. 403 `not_verified`.
- Nunca: motivo de negação, `critical`, valores fora da política, solicitante digitado.

## 10. Linha do tempo

`kind` novo `exam` em `Timeline::Sources::Exams` (profissional: título = exame, `status`, `summary` = etapa atual ou "resultado registrado"; `detail_ref: { "type": "exam_order", "id" }`). Cidadão: entrada com `status_label` e, quando a política permite, `document`-like `{ "id", "downloadable": true }` apontando para `/citizen/exams/:id/pdf`.

## 11. Maintenance (GraphQL, token humano)

- Interruptores `exam_tracking` (`requires: ["feature:clinical_record"]`), `exam_fhir_inbound` (`["feature:exam_tracking"]`), `exam_pec_bridge` (`["feature:exam_tracking", "feature:pec_external_view"]`), `fake_lab` (`["feature:exam_fhir_inbound"]`, `non_production: true`). Mensagens de `missing` pelo mecanismo genérico.

## 12. Eventos (só ids)

| Nome | Payload |
|---|---|
| `exam_order.requested` | `exam_order_id, origin, priority` |
| `exam_order.authorized` / `.denied` / `.scheduled` / `.collected` / `.cancelled` | `exam_order_id` |
| `exam_order.resulted` | `exam_order_id, source, critical` |
| `exam_order.acknowledged` | `exam_order_id, by_team` |
| `exam_order.escalated` | `exam_order_id, reason` |
| `exam_result.external_recorded` | `exam_result_id` |
| `exam_provider.fhir_token_rotated` / `_revoked` | `exam_provider_id` |
| `clinical_record.viewed` | + `access: exam_result` |

## 13. Laboratório simulado (`fake-lab`, compose de dev)

Lê `FAKE_LAB_TOKEN` (token FHIR do prestador "Laboratório Simulado" da semente) e `FAKE_LAB_CITY_HOST`; a cada 30 s pede `GET /fhir/ServiceRequest?status=active` e, para cada guia, envia um `DiagnosticReport` do §7 (hemograma ou HbA1c, PDF de exemplo). Com `fake_lab` desligado, o api responde 403 e o serviço só registra no log.

## 14. Ordem de entrega

0. Provas técnicas (plano do api, Task 0): escrita na tela de sistemas externos do PEC local; perfil do `DiagnosticReport`.
1. `contracts` — `exams/exam-order-v1.json` e `exams/fhir-diagnostic-report-inbound.json`, tag `exams-v1.0.0`.
2. `api` (porta de dev 3041; migrações de cidade e plataforma a partir de 20261013900001; worktree `.claude/mod21`, branch `feat/mod-21a-exams`, bancos `_mod21`).
3. `dashboard` (Vite 5192) e `wpda` (Vite 5193), worktrees em `rota-saude/.worktrees/`. 4. `maintenance`. 5. catálogo público.
