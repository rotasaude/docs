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

- Na página `/pec/timeline` do módulo 20 (aberta pela chamada assinada), botão "Solicitar exames": `POST /pec/exams/form` (token da sessão curta) → HTML do formulário; `POST /pec/exams` (form, token CSRF da sessão curta) cria `exam_order` com `origin: pec`. Sem chamada assinada nova (cada cadastro no PEC tem chave própria; a cidade guarda uma). Erros: os `pec_*` do módulo 20 + `pec_exam_bridge_disabled`.
- Só se a prova técnica (plano do api, Task 0a) confirmar envio de formulário dentro da tela de sistemas externos; falhou → esta seção sai.

## 9. Cidadão (sessão do wpda, só par validado)

- `GET /citizen/exams?before=&limit=` → `{ "verified": bool, "exams": [ { "id", "label", "status_label", "requested_at", "scheduled_at"|null, "scheduled_place"|null, "result": { "available": bool, "pickup_at_unit": bool, "released_on"|null, "pdf_available": bool }|null } ], "next_before" }`.
- `GET /citizen/exams/:id/pdf` → `application/pdf` só se a política permitir; senão 404 `not_found`. 403 `not_verified`.
- Nunca: motivo de negação, `critical`, valores fora da política, solicitante digitado.

## 10. Linha do tempo

`kind` novo `exam` em `Timeline::Sources::Exams` (profissional: título = exame, `status`, `summary` = etapa atual ou "resultado registrado"; `detail_ref: { "type": "exam_order", "id" }`). Cidadão: entrada com `status_label` e, quando a política permite, `document`-like `{ "id", "downloadable": true }` apontando para `/citizen/exams/:id/pdf`.

## 11. Maintenance (GraphQL, token humano)

- Interruptores `exam_tracking` (`requires: ["record_mode"]`, missing `record_mode_off`), `exam_fhir_inbound` (`["feature:exam_tracking"]`), `exam_pec_bridge` (`["feature:exam_tracking", "feature:pec_external_view"]`), `fake_lab` (`["feature:exam_fhir_inbound"]`, `environments: SIMULATION_ENVS`). Mensagens de `missing` pelo mecanismo genérico: `record_mode_off`, `exam_tracking_disabled`, `exam_fhir_inbound_disabled`, `pec_external_view_disabled`.

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

## 15. Acréscimos (revisão dos planos, 2026-10-14)

Aprovados pelo usuário: A1 (D1 do api), A8 (D8), A10 (D10), A15 (D15). Os demais são detalhe dos planos, aceitos pelo coordenador. Prevalecem sobre as seções acima.

**Decisões**
- **A1 — `exam_tracking` nos dois modos.** `requires: ["record_mode"]` (record ou integrated). No `record` o pedido também nasce da consulta; no `integrated`, pela guia e pelo PEC.
- **A8 — Botão no PEC na página do módulo 20** (§8 reescrito).
- **A10 — Equipe de referência** = equipes ativas (`health_teams`) da unidade do exame; dá ciência médico ou enfermeiro dessas equipes (`not_in_team`).
- **A15 — `fake_lab` em `SIMULATION_ENVS`** (dev e staging). `exam_providers.simulated`: prestador simulado só recebe com `fake_lab` ligado.

**api**
- **A2 — Guia:** aceita `citizen_id` (via `Patients::Resolve`) ou `patient_id`; exige `unit_id`; erros `not_unit_staff`, `invalid_requester`, `invalid_cid10`.
- **A3 — Upload em JSON:** `POST /exams/uploads { "content_type", "data" (base64) }` → `{ "upload_id" }`; sem nome de arquivo; 422 `invalid_file`. Multipart não existe (a guarda de escrita por cookie só aceita JSON).
- **A4 — Arquivos:** `pdf_url`/`guide_url` = `/exams/files/<token>`, 5 min, presos ao usuário; vencido → 410 `link_expired`.
- **A5 — Duas visões do exame:** clínica (com resultado) e administrativa (sem `values`, `critical` e PDF) conforme quem lê.
- **A6 — Listas com recorte obrigatório:** `consultation_id`, `patient_id`, `requester=me`, `unit_id`, ou `overdue=true` sozinho (só admin/regulador, cidade inteira); 422 `filter_required`, `invalid_filter`.
- **A7 — FHIR:** sistemas de identificador da RNDS `http://rnds.saude.gov.br/fhir/r4/NamingSystem/cpf` e `.../cns` (substituem os do §7); `code` obrigatório; cada `Observation` contida referenciada em `result` (dom-3); só `application/fhir+json` (415); corpo até 15 MB (413); `details` só com caminhos; `ServiceRequest` exige `status=active`.
- **A11 — Papéis:** `exam_regulator` e `exam_provider` em `PRIVILEGED_ROLES`; convite do prestador `POST /exams/admin/providers/:id/users { "email" }` → `<invitation>`; relatório também para `exam_regulator`.
- **A12 — Transições:** reagendar permitido; resultado só de `scheduled` ou `collected`; 422 `provider_not_enabled_for_exam`, `invalid_schedule`.
- **A14 — fake-lab:** `FAKE_LAB_API_URL`, `FAKE_LAB_CITY_HOST`, `FAKE_LAB_TOKEN`, `FAKE_LAB_INTERVAL`.
- **A16 — Ator `system`** nos eventos do job; `reason` do escalonamento é código (`overdue` | `critical` | `unlinked_requester`).
- **A17 — Resultado:** 422 `missing_pdf`, `invalid_values`, `invalid_released_on`, `invalid_laboratory`; crítico ou solicitante digitado nasce escalado.
- **A18 — Migrações** a partir de 20261013900001; recorte SIGTAP de dev com hemograma `0202020380`, HbA1c `0202010503`, anti-HIV `0202030300`, anatomopatológico `0203020030`.
- **A19 — Rotas de apoio:** `POST /exams/patient_lookup { "cpf" }`; `GET /exams/catalog` (itens ativos com `label` e `result_fields`); `GET /exams/providers?sigtap_code=`; `GET /exams/requesters?unit_id=`; `GET /exams/external_results?patient_id=`. Evento `exam.patient_looked_up`.
- **A20 — Formas:** `<exam_order>` + `consultation_id` (filtro em `/exams/orders`) e `result_fields` no detalhe; `<provider_order>` + `result_fields`; lista do portal com `overdue_count` no corpo; `acknowledgements/pending` → `{ "orders", "denied" }`, `team` → `{ "orders" }`; prestador com `has_fhir_token`; no portal, agendar sem `provider_id`; `exam_requests[].priority` no corpo da consulta.
- **A21 — Exame aberto pela linha do tempo:** só em contexto ou com abertura justificada válida (regra do 19a); nada novo no api.

**wpda**
- **A22 — `citizen_id`** em `GET /citizen/exams` e `/citizen/exams/:id/pdf` (celular com várias pessoas; molde do módulo 20); `exam_tracking` em `features` da sessão do cidadão.
- **A23 — `result` por política:** `null` antes de o cidadão poder saber; `never_citizen` → `pickup_at_unit: true`, sem PDF; `requested_at`/`scheduled_at` ISO com fuso, `released_on` `AAAA-MM-DD`. Par declarado: 200 com `verified: false` e `exams: []`. `status_label` pronto, sem motivo; em `after_acknowledgement`, antes da ciência mostra a etapa anterior ao resultado. Resultado externo entra na lista com `kind`.
- **A24 — Linha do tempo do cidadão:** `status_label` opcional em `<citizen_entry>`; para exame, `document` = `{ "id": <exam_order id>, "short_code": null, "verification_url": null, "downloadable": true }`, PDF por `/citizen/exams/:id/pdf`, mesmo 429 dos documentos.

**maintenance**
- **A25 — Aviso de laboratório simulado** na aba de interruptores, no padrão do PSC e do SNCR simulados.

**Execução (pré-voo do api, 2026-10-14)**
- **A26 — `priority` na leitura da consulta:** `Consultations::Json` devolve `exam_requests[].priority` no rascunho e na consulta finalizada (complementa A20).
- **A27 — Trilha do laudo:** o download por `/exams/files/<token>` de um PDF de resultado grava `clinical_record.viewed` com `access: exam_result` (§12); a guia da recepção não grava.
- **A28 — Ator do evento:** `<exam_event>.actor` = `{ "type": "user" | "provider" | "system", "name": string | null }`; `system` (job diário) vem com `name: null` (complementa A16). O dashboard aceita os três.
- **A29 — `result` do cidadão:** a A23 vale para a lista e para a linha do tempo (A24); a D9 do plano do api fica superada.
- **A30 — Erros do catálogo (§5):** além de `sigtap_unknown`, `duplicate_sigtap` e `invalid_deadline`, 422 `invalid_policy`, `invalid_result_fields`, `invalid_active` e `invalid_requires_authorization`.
- **A31 — Escalonamento por motivo:** no máximo um `escalated` por motivo e por pedido. Exame já escalado por `overdue` que recebe resultado crítico ganha também `escalated` com `reason: critical`; a fila da equipe mostra o crítico. `escalated_at` guarda o primeiro.
- **A32 — Motivo na visão administrativa:** o `reason` de `denied` e `cancelled` aparece na visão administrativa (regulador, recepção, unidade) e na clínica; o motivo de `escalated` e o da ciência pela equipe só na clínica. O cidadão continua sem motivo algum.
- **A33 — Prioridade fixa:** a `priority` do exame não muda depois de criado o pedido; adendo que só troca a prioridade não altera o exame (para trocar: remover o exame e incluir de novo). O dashboard não oferece troca de prioridade no adendo.
- **A34 — Prestador simulado e reagendamento:** prestador `simulated` com `fake_lab` desligado → 422 `provider_not_enabled_for_exam` (agendar, autorizar). A unidade pode trocar o prestador ao reagendar; no portal, o prestador não troca (A12).
- **A35 — Quem agiu no portal fica na trilha:** evento feito pelo portal grava `actor_provider_id` **e** `actor_user_id` (o funcionário do laboratório); o check passa a ser "pelo menos um ator, ou nenhum para `system`". Pela entrada FHIR, só `actor_provider_id`. `<exam_event>.actor` continua `{ "type": "provider", "name": <prestador> }` para quem lê na unidade; a pessoa aparece só na trilha (consulta administrativa), nunca para o cidadão. Sem `Platform.audit` (dado da cidade fica no banco da cidade).
- **A36 — Reenvio, correção e código na entrada FHIR (§7):**
  - `identifier` repetido do mesmo prestador, mesma guia e `status: final` → 200 com o mesmo `result_id` (idempotência).
  - `identifier` repetido para outra guia → 409 `identifier_conflict`.
  - `status: amended` → 422 `amendment_not_supported` (sem fluxo de retificação no 21a; o laboratório fala com a unidade). Primeiro envio só com `final`.
  - `code` com coding SIGTAP diferente do SIGTAP da guia → 422 `invalid_resource` com `details: ["/code"]`.
  - `missing_pdf` cobre também PDF acima de 10 MB.
- **A37 — `never_citizen` só depois da ciência:** o cidadão vê "Retire o resultado na sua unidade" (`pickup_at_unit: true`) só a partir de `acknowledged`; antes, o rótulo da etapa anterior, como em `after_acknowledgement`. Resultado externo `never_citizen` nasce com ciência e mostra o aviso.
- **A38 — Resultado externo sem unidade:** qualquer unidade da cidade pode registrar (o corpo do §3 não traz `unit_id`); o paciente é da cidade, não da unidade.
- **A39 — `document.id` do resultado externo:** na linha do tempo do cidadão, o resultado externo leva `document.id` = id do `exam_result` (não há pedido); `/citizen/exams/:id/pdf` aceita id de pedido ou de resultado externo (complementa A24).
- **A40 — Status do cidadão:** valores de status do exame no wpda incluem `external`, `scheduled`, `collected` e `acknowledged` (com `status_label` pronto do `Exams::Release`); o wpda trata valor desconhecido mostrando só o `status_label`.
- **A41 — Deduplicação no PEC:** `POST /pec/exams` com mesmo paciente, SIGTAP e solicitante de um pedido `pec` dos últimos 15 minutos devolve o pedido existente com o aviso "já registrado" (proteção contra duplo envio do formulário; lock por CPF).
- **A42 — Regulador vê o CID:** quem lê com `exam_regulator` (ou `municipal_admin` na fila de regulação) recebe `cid10_justification` na fila (§4) e no detalhe do exame; `requester.cns` continua fora da visão administrativa. As demais leituras administrativas (recepção, unidade, prestador) seguem sem CID e sem CNS (complementa A5). O esquema `exam-order-v1` acompanha antes da tag `exams-v1.0.0`.
- **A43 — Regras de tela do dashboard:** `municipal_admin` não é equipe da unidade (403 `not_unit_staff`) e não vê "Exames da unidade"; laudo só PDF (`missing_pdf`), guia PDF, JPEG ou PNG; link vencido 410 `link_expired`; fila da equipe sem equipe = estado vazio explicado; contador de ciência mostra "50+" (lista paginada em 50); reagendar disponível para a unidade em `scheduled`.
