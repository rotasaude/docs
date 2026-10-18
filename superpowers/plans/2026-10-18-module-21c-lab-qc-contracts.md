# Módulo 21c — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-18-module-21c-lab-quality-supplies-design.md` · **ADR:** `adr/0038.md` · **Base:** contrato do 21b (`superpowers/plans/2026-10-15-module-21b-lab-contracts.md`, inclusive §14 B1–B64) e do 21a (§15).

Fonte única dos formatos entre `api`, `dashboard`, `maintenance`, `contracts` e o
middleware simulado. Convenções herdadas: `{ "error": "<reason>" }`; step-up 401
`mfa_required`; interruptor 403 `feature_disabled`; JSON em toda escrita por
cookie; números digitados com vírgula decimal (ponto recusado, D11 do 21b);
polling nunca em rota que grava trilha; lock na transição e ordem de travas
estável.

## 1. Nomes no código

| Spec | Código |
|---|---|
| situação da corrida | `open` \| `approved` \| `warning` \| `rejected` |
| origem da referência | `package_insert` \| `laboratory` |
| regras | `1_2s` (aviso) \| `1_3s` \| `2_2s` \| `R_4s` \| `4_1s` \| `10x` |
| desfecho do valor | `ok` \| `warning` \| `reject` |
| decisão da revisão | `pending` \| `kept` \| `rectify` |
| adequação do CEQ | `adequate` \| `inadequate` |
| tipo do item | `reagent` \| `calibrator` \| `control` \| `consumable` |
| origem do lote | `purchase` \| `donation` \| `loan` |
| movimento | `receipt` \| `opening` \| `loss` \| `expiry` \| `adjustment` |
| situação do lote de controle | `in_use` \| `closed` |

## 2. Aparelhos (`/lab/admin/instruments`, `municipal_admin`; leitura para membros do laboratório em `/lab/instruments`)

- `GET/POST/PATCH /lab/admin/instruments[/:id]` — `{ "provider_id", "name", "manufacturer", "model", "serial_number", "sector", "hl7_equipment_id"|null, "active" }`; 422 `invalid_name`, `invalid_sector`, `duplicate_hl7_equipment_id`.
- `POST /lab/admin/exams/:id/versions` (21b) aceita `lab_instrument_id`|null.

## 3. CIQ (`/lab/qc`)

- Materiais e lotes (admin ou RT/substituto): `GET/POST/PATCH /lab/qc/materials[/:id]` — `{ "provider_id", "name", "manufacturer", "levels": [string], "analytes": [ { "lab_exam_id", "analyte_key" } ], "active" }`; `GET/POST /lab/qc/lots`, `POST /lab/qc/lots/:id/close` — `{ "material_id", "lot_number", "expires_on", "level_barcodes": { "<level>": "<barcode>" } }`; 422 `barcode_taken` (colide com controle ou amostra), `invalid_levels`, `expired_lot`.
- Referências: `GET /lab/qc/targets?lot_id=` → versões vigentes e histórico; `POST /lab/qc/targets` (bula) — `{ "lot_id", "level", "analyte_key", "instrument_id", "mean", "sd" }`; `GET /lab/qc/targets/suggestions` → sugestões com ≥ 20 corridas aprovadas (`{ lot_id, level, analyte_key, instrument_id, mean, sd, runs }`); `POST /lab/qc/targets/suggestions/accept` (RT/substituto) — mesma chave → nova versão `laboratory`.
- Corridas: `GET /lab/qc/runs?instrument_id=&date=` ; `POST /lab/qc/runs` (abrir a pedido) — `{ "instrument_id" }`; `POST /lab/qc/values` — `{ "run_id"|null, "instrument_id", "lot_id", "level", "analyte_key", "value" }` → `{ "value": <qc_value>, "run": <qc_run>, "block": <qc_block>|null }`; 422 `no_target`, `expired_lot`, `lot_closed`, `invalid_value`.
- `<qc_value>`: `{ "id", "level", "analyte_key", "value", "z_score", "rules_fired", "outcome", "source", "reagent_lot_id"|null, "entered_at", "entered_by" }`. `<qc_run>`: `{ "id", "instrument_id", "status", "started_at", "evaluated_at"|null, "values": [<qc_value>] }`.
- Levey-Jennings: `GET /lab/qc/chart?lot_id=&level=&analyte_key=&instrument_id=&from=&to=` → `{ "target": { "mean", "sd", "version", "source" }, "points": [ { "at", "value", "z_score", "outcome", "rules_fired", "target_version" } ] }` (até 200 pontos; sem trilha).
- Regras (RT/substituto): `GET /lab/qc/rules?instrument_id=&analyte_key=`; `PUT /lab/qc/rules` — `{ "instrument_id", "analyte_key", "rules": { "<regra>": bool }, "justification" }`; 422 `justification_required`.
- Bloqueios: `GET /lab/qc/blocks?status=active|closed` → `{ "blocks": [ { "id", "instrument_id", "analyte_key", "opened_at", "opened_by_run_id", "corrective_action"|null, "closed_at"|null } ] }`; `GET /lab/qc/blocks/active/count` (sem trilha); `POST /lab/qc/blocks/:id/corrective_action` — `{ "text" }`; o fechamento é automático na corrida aprovada com todos os níveis depois da ação (422 `corrective_action_required` se a corrida aprovar sem ação registrada: o bloqueio continua).
- Exceção (RT/substituto): `POST /lab/qc/exceptions` — `{ "block_id", "item_id", "justification" }` → autoriza a liberação daquele item (a liberação do 21b segue normal); 422 `justification_required`, `not_blocked`.
- Revisão: `GET /lab/qc/reviews?status=pending`; `POST /lab/qc/reviews/:id` — `{ "decision": "kept" | "rectify", "note"? }` (`rectify` devolve o laudo a retificar; a retificação é a do 21b).
- Liberação do 21b: item com bloqueio ativo e sem exceção → 409 `qc_blocked` com `{ "error", "items": [ { "item_id", "block_id", "analyte_key", "instrument_id" } ] }`.

## 4. CEQ (`/lab/eqa`, `lab_releaser`, RT, admin)

- `GET/POST /lab/eqa/rounds`, `GET /lab/eqa/rounds/:id` — `{ "provider_id", "program", "round_code", "round_date", "items": [ { "analyte_key", "lab_exam_id", "submitted", "target", "acceptable_low"|null, "acceptable_high"|null, "adequacy", "action"|null } ], "annual_report_upload_id"|null }`; 422 `action_required` (item `inadequate` sem ação).
- `GET /lab/eqa/summary?provider_id=&year=` → adequação por analito.

## 5. Estoque (`/lab/inventory`)

- Itens (admin): `GET/POST/PATCH /lab/inventory/supplies[/:id]` — `{ "provider_id", "name", "kind", "unit", "tests_per_package"|null, "minimum_stock", "expiry_alert_days", "lab_exam_ids", "active" }`.
- Lotes e movimentos (`lab_staff`, `lab_releaser`, admin): `GET /lab/inventory/lots?supply_id=&with_balance=true`; `POST /lab/inventory/lots` (entrada) — `{ "supply_id", "lot_number", "expires_on", "supplier", "origin", "quantity", "document"? }`; `POST /lab/inventory/movements` — `{ "lot_id", "kind": "opening"|"loss"|"expiry"|"adjustment", "quantity", "reason" }` (`adjustment` só admin; aceita sinal); 422 `expired_lot`, `insufficient_balance`, `reason_required`.
- Lote em uso: `GET /lab/inventory/in_use?instrument_id=`; `POST /lab/inventory/in_use` — `{ "lot_id", "instrument_id", "lab_exam_id" }` (encerra o anterior do par); 422 `lot_not_reagent`, `expired_lot`, `exam_not_linked`.
- Alertas: `GET /lab/inventory/alerts` → `{ "expiring": [...], "below_minimum": [...], "untraced_results": <count> }` (sem trilha).
- Rastreio: `GET /lab/inventory/lots/:id/trace` (+ `.csv`) → `{ "results": [ { "requisition_number", "sigtap_code", "entered_at" } ], "qc_values": [ { "run_id", "analyte_key", "entered_at" } ] }` (sem nome de paciente; trilha de leitura).
- Consumo: `GET /lab/inventory/consumption?provider_id=&month=` (+ `.csv`) → linhas `{ supply_id, name, instrument_id|null, tests_performed, tests_opened, deviation_pct }`.

## 6. HL7 do controle

- `POST /hl7/v2` (21b): `OUL^R22` cujo `SPM-2` é código de barras de nível de lote de controle → valor de controle; aparelho por `EQU-1` (ou `OBX-18`) contra `hl7_equipment_id`. Convenção exata da marca de controle (prova técnica, Task 0) documentada em `lab/hl7-v2-profile.md` (seção "Controle"). Recusas `AE` (`ERR-5`): `qc_lot_expired`, `qc_lot_closed`, `qc_no_target`, `unknown_instrument`. O controle nunca cria `lab_results`.
- `fake-middleware`: manda também corridas de controle (níveis normal e patológico, uma corrida 2-2s reprovada).

## 7. Lote no resultado (21b)

- `lab_results.reagent_lot_id` e `lab_qc_values.reagent_lot_id` preenchidos com o lote em uso do par aparelho × exame no momento da gravação; sem lote → `null` e marca "sem rastreio". `<lab_result_row>` ganha `reagent_lot`: `{ "id", "lot_number" }`|null (só membros do laboratório).

## 8. Maintenance

- Interruptores `lab_quality_control` e `lab_inventory` (`requires: ["feature:municipal_lab"]`); `missing` `municipal_lab_disabled`.

## 9. Eventos (só ids)

`lab_qc_run.evaluated` (`status`), `lab_qc_block.opened`, `lab_qc_block.closed`, `lab_qc_exception.granted`, `lab_qc_review.decided` (`decision`), `lab_qc_target.accepted` (`source`), `lab_qc_rule.changed`, `lab_eqa_round.recorded`, `lab_supply_lot.received`, `lab_supply_movement.recorded` (`kind`), `lab_supply_in_use.changed`.

## 10. Ordem de entrega

0. Provas técnicas (plano do api, Task 0): convenção HL7 v2.5.1 de resultado de controle e identificação do aparelho; Levey-Jennings com a biblioteca de gráficos do dashboard.
1. `contracts` — seção "Controle" no perfil HL7, exemplos; tag `lab-v1.1.0`.
2. `api` (porta 3043; migrações de cidade a partir de 20261018900001; worktree `.claude/mod21c`, branch `feat/mod-21c-lab-qc`, bancos `_mod21c`).
3. `dashboard` (Vite 5196; worktree `rota-saude/.worktrees/dashboard-mod21c`). 4. `maintenance`. 5. catálogo público.
