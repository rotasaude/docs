# Módulo 25 — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-27-module-25-disease-surveillance-design.md` · **ADR:** `adr/0043.md`

Fonte única dos formatos entre `api`, `dashboard` e `maintenance` (o wpda não muda).
Convenções herdadas (contratos do 21a §15, 21b §14, 21c §11, 22 §12–§13, 23 §11–§12 e
24a §14): erros `{ "error" }`; step-up; interruptor 403 `feature_disabled`; JSON em toda
escrita por cookie; listas com envelope e `next_before`; registro de outro escopo por
id → 404, lista com escopo alheio → 403; leitura nominal com trilha e sem polling;
contadores sem trilha; texto longo recusado; dado sensível só no corpo, nunca na URL
nem no log; todo Data/Struct com dado pessoal sem dado em `inspect`/`to_s`/
`pretty_print`/`as_json`; paciente sem CPF como no 24a (`has_cpf`, `cns_masked`).

## 1. Nomes no código

| Spec | Código |
|---|---|
| interruptor | `surveillance` (sem `requires`; a sugestão na consulta exige também `clinical_record`) |
| papel | `surveillance` (privilegiado, step-up para conceder) |
| prazo da doença | `immediate` (24 h) \| `weekly` (7 dias) |
| origem | `consultation` \| `manual` \| `safety_net` |
| situação | `possible` (rede de segurança) \| `notified` \| `investigating` \| `closed` \| `entered` (digitada no sistema oficial) \| `dismissed` (possível descartada) |
| classificação final | `confirmed` \| `discarded` \| `inconclusive` |
| semana epidemiológica | `YYYY-WW` pela regra do Sinan (domingo a sábado; semana 1 = a que contém o primeiro sábado do ano — confirmar na Task 0) |

## 2. Formas

`<disease>`: `{ "code", "name", "deadline": "immediate"|"weekly", "cid10": [..], "ciap2": [..], "repeat_after_days", "sinan_form"|null, "local": bool, "list_version" }`.

`<notification>` (nominal; trilha): `{ "id", "patient": { "id", "display_name", "has_cpf", "cns_masked"|null }, "disease": { "code", "name", "deadline" }, "origin", "status", "classification"|null, "suspected_on", "symptoms_on"|null, "unit": { "id", "cnes", "name" }, "notifier": { "name" }|null, "due_at", "overdue": bool, "official_number"|null, "resident_elsewhere": bool, "residence_ibge"|null, "neighborhood": { "id", "name" }|null, "form": {…}, "version", "events": [ { "status", "at", "by": { "name" }|null, "reason"|null } ] }`.

`form` traz os campos da ficha de notificação individual do Sinan que não vêm do paciente; as chaves são fixadas pela Task 0 do api e publicadas como §14 deste contrato antes das tasks de formulário do dashboard.

## 3. Lista de doenças

- Plataforma (maintenance, GraphQL): `notifiableDiseases` (versão vigente e histórico) e `publishNotifiableDiseaseList(input)` → `{ ok, errors }`; publicar cria versão nova; evento de plataforma `notifiable_disease_list.published` (R18).
- Cidade: `GET /surveillance/diseases` → `{ "diseases": [<disease>] }` (plataforma vigente + locais; `surveillance` e `health_professional`).
- Doenças locais: `GET/POST/PATCH /surveillance/admin/local_diseases[/:id]` (`surveillance`, escrita com step-up); 422 `invalid_disease`, 409 `duplicate_code`.

## 4. Na consulta (`health_professional` na consulta, `clinical_record` e `surveillance` ligados)

- `GET /consultations/:id/notification_suggestions` → `{ "suggestions": [ { "disease": <disease>, "matched_code" } ] }` (problemas `add` da consulta cujo código está na lista, sem notificação aberta ou recente do paciente para a doença).
- `POST /surveillance/notifications` — `{ "consultation_id", "disease_code", "symptoms_on"? }` → 201 `<notification>`; 409 `already_notified` (com `notification_id` se visível); 403 `cbo_not_allowed` (só médico `225*` e enfermeiro `2235*`). Nunca bloqueia a finalização.

## 5. Fila da vigilância (`surveillance`)

- `GET /surveillance/notifications?status=&disease=&overdue=&unit_id=&before=&limit=` → `{ "notifications": [<notification> sem `form` e `events`], "next_before" }` (trilha uma por consulta; imediatas e vencidas primeiro).
- `GET /surveillance/notifications/:id` → `<notification>` (trilha).
- `POST /surveillance/notifications` (manual) — `{ "patient_id" | "citizen_id" | "new_patient": {…forma do 24a}, "disease_code", "suspected_on", "symptoms_on"?, "unit_id", "resident_elsewhere"?, "residence_ibge"? }`.
- `PATCH /surveillance/notifications/:id` — `{ "form", "version" }`; 409 `stale_version`; 422 `invalid_form` (com `path`).
- `POST /surveillance/notifications/:id/transition` — `{ "to", "classification"?, "official_number"?, "reason"?, "version" }`; 422 `invalid_transition`, `classification_required`, `official_number_required`.
- `POST /surveillance/notifications/:id/dismiss` — `{ "reason", "version" }` (só `possible`).
- `GET /surveillance/notifications/count` → `{ "possible", "notified", "investigating", "overdue", "immediate_open", "without_official_number" }` (sem trilha).

## 6. Notificação negativa

- `health_units.notifying` (bool; padrão verdadeiro para unidades de APS ativas), editável pelo `municipal_admin`.
- `GET /surveillance/weekly_negatives?week=YYYY-WW` → `{ "week", "units": [ { "unit": {…}, "status": "with_cases"|"negative"|"pending" } ] }`.
- `POST /surveillance/weekly_negatives` — `{ "unit_id", "week" }`; 409 `unit_has_notifications`, 409 `already_recorded`, 422 `future_week`.

## 7. Saídas (`surveillance`; trilha; sem cache)

- `GET /surveillance/notifications/:id/form.pdf` — ficha de notificação individual no formato do Sinan.
- `GET /surveillance/exports/week.csv?week=YYYY-WW` — uma linha por notificação da semana (sem CPF; CNS mascarado).
- Relatório sem número oficial = lista com `status=closed` e sem `official_number`.

## 8. Painel (`surveillance` e `analyst`; agregado sem nome)

`GET /surveillance/panel?from_week=&to_week=&disease=` → `{ "weeks": ["YYYY-WW"…], "rows": [ { "neighborhood": { "id"|null, "name" }, "counts": [n|null], "signal": [bool] } ], "suppressed": true }`. Contagens 1–4 viram `null` (regra do analytics); sinal = semana acima do dobro da média das 4 anteriores com pelo menos 5 casos; sempre rotulado como sinal para avaliar.

## 9. Alertas e jobs

- Notificação imediata criada → e-mail para todos os membros ativos `surveillance`: "nova notificação imediata: <doença>, <unidade>" (sem dado do paciente).
- Job diário: vencidas → e-mail resumo (só contagens).
- Job semanal (segunda): unidades notificadoras sem negativa nem caso na semana anterior → e-mail resumo.
- Rede de segurança: consumidor de `consultation.finalized` (e `consultation.addendum_added`) cria `possible` idempotente.

## 10. Bairro e residência

Bairro: domicílio do cadastro territorial (24a) → `citizens.neighborhood_id` → null ("não informado"). `resident_elsewhere` com `residence_ibge` (7 dígitos) para quem mora em outro município.

## 11. Maintenance

- Interruptor `surveillance`.
- Lista de doenças da plataforma: listar, editar e publicar versão (mantenedor com step-up).

## 12. Eventos (só ids)

`disease_notification.created`, `.updated`, `.transitioned`, `.dismissed`, `weekly_negative.recorded`, `local_disease.changed`; plataforma `notifiable_disease_list.published`.

## 13. Ordem de entrega

0. Prova técnica (plano do api, Task 0): Portaria 11.211/2026 (lista e prazos), CID-10 de cada doença nas fichas do Sinan e CIAP-2 equivalentes, ficha de notificação individual do Sinan (campos), regra da semana epidemiológica; carga inicial com fonte por linha; tabela da ficha publicada como §14.
1. `api` (porta 3047; migrações de cidade e plataforma a partir de 20261027900001; worktree `.claude/mod25`, branch `feat/mod-25-disease-surveillance`; testes com `bin/test api -w mod25`).
2. `dashboard` (Vite 5202), worktree em `rota-saude/.worktrees/dashboard-mod25`.
3. `maintenance` (prova na 5177 com o serviço do compose parado), worktree em `rota-saude/.worktrees/maintenance-mod25`.
4. Catálogo público.
