# Módulo 27a — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-29-module-27a-pharmacy-stock-design.md` · **ADR:** `adr/0045.md`

Fonte única dos formatos entre `api`, `dashboard` e `maintenance` (o wpda não muda).
Convenções herdadas (contratos do 21a §15, 21b §14, 21c §11, 22 §12–§13, 23 §11–§12,
24a §14, 25 §15 e 26a §13): erros `{ "error" }`; step-up (TOTP); interruptor 403
`feature_disabled`; JSON em toda escrita por cookie; listas com envelope e
`next_before` (cursor opaco quando a ordem não é por id); registro de outro escopo
por id → 404, lista com escopo alheio → 403; leitura nominal com trilha e sem polling;
contadores sem trilha; texto longo recusado; dado sensível só no corpo, nunca na URL
nem no log; todo Data/Struct com dado pessoal sem dado em `inspect`/`to_s`/
`pretty_print`/`as_json`; paciente sem CPF como no 24a (`has_cpf`, `cns_masked`);
quantidades como string decimal com até 2 casas (vírgula só na tela).

## 1. Nomes no código

| Spec | Código |
|---|---|
| interruptor | `pharmacy` (sem `requires`) |
| papéis | `pharmacist` (CBO `2234*`; tudo, inclusive controlados) · `pharmacy_assistant` (dispensa não controlado) · `stock_keeper` (entradas, transferências, inventário) — vinculados a locais |
| local | `central` \| `unit` (unidade com a marca `dispenses`) |
| movimento | `opening` \| `receipt` \| `donation` \| `transfer_out` \| `transfer_in` \| `dispensing` \| `loss` \| `expiry` \| `adjustment` \| `return` |
| transferência | `sent` \| `received` \| `received_with_difference` |
| ajuste | `applied` \| `pending` (acima de 10% do saldo do lote) \| `approved` \| `rejected` |
| receita | `digital` (documento do 19c) \| `external` |

## 2. Formas

`<lot>`: `{ "id", "item": { "catalog_item_id", "catmat_code", "name", "unit", "controlled": bool, "controlled_list"|null }, "lot_number", "expires_on", "manufacturer"|null }`.

`<position>` (sem dado pessoal; sem trilha): `{ "location": { "id", "name", "kind" }, "item": {…}, "lots": [ { "lot": <lot>, "balance", "in_transit" } ], "balance", "minimum"|null, "below_minimum": bool, "expiring_soon": bool }`.

`<prescription_view>` (nominal; trilha): `{ "kind": "digital"|"external", "id", "short_code"|null, "issued_on", "valid_until", "expired": bool, "cancelled": bool, "prescriber": { "name", "council"|null }, "patient": { "id", "display_name", "has_cpf", "cns_masked"|null }, "items": [ { "id", "catalog_item_id"|null, "free_text"|null, "printed_description", "quantity", "quantity_unit", "dosage_instructions", "continuous": bool, "controlled": bool, "dispensed_total", "last_dispensed_on"|null, "next_allowed_on"|null, "completed": bool } ] }`.

`<dispensation>`: `{ "id", "location": {…}, "prescription": { "kind", "id" }, "patient": {…}, "by": { "name", "role" }, "at", "items": [ { "prescription_item_id"|null, "catalog_item_id", "lot_id", "quantity", "mapped_from_free_text": bool } ], "retention": { "notification_number"|null }|null }`.

## 3. Administração (`municipal_admin`; escrita com step-up)

- `GET/POST/PATCH /pharmacy/admin/locations[/:id]` — `{ "kind", "health_unit_id"? (só `unit`), "name", "active" }`; a unidade ganha `dispenses: true`.
- `POST/DELETE /pharmacy/admin/locations/:id/staff` — `{ "user_id", "role" }`.
- `GET/PATCH /pharmacy/admin/settings` — `{ "common_validity_days": 30, "continuous_validity_days": 180, "continuous_interval_days": 25, "expiry_alert_days": 90, "adjustment_approval_percent": 10 }`.
- `GET/PUT /pharmacy/locations/:id/minimums` — `[ { "catalog_item_id", "minimum" } ]` (`stock_keeper` do local).

## 4. Entradas e posição (`stock_keeper` do local)

- `POST /pharmacy/receipts` — `{ "location_id", "document": { "kind": "invoice"|"donation", "number", "supplier" }, "items": [ { "catalog_item_id", "lot_number", "expires_on", "manufacturer"?, "quantity", "package"?: { "units_per_package", "packages" } } ] }` → 201; 422 `invalid_item`, `expired_lot`, `invalid_quantity` (com `path`).
- `POST /pharmacy/openings` — inventário de abertura (mesma forma, `kind: opening`), uma vez por local e item.
- `GET /pharmacy/positions?location_id=&q=&expiring=true&below_minimum=true&before=&limit=` → `{ "positions": [<position>], "next_before" }` (central vê todos os locais; unidade vê o seu).
- `GET /pharmacy/lots/:id/movements?location_id=&before=` → extrato (sem nome de paciente; dispensação aparece só com o id).

## 5. Transferências

- `POST /pharmacy/transfers` (`stock_keeper` do central) — `{ "to_location_id", "items": [ { "lot_id", "quantity" } ] }` → `sent`; sai do central e fica `in_transit`.
- `POST /pharmacy/transfers/:id/receive` (`stock_keeper`, `pharmacist` ou `pharmacy_assistant` do destino) — `{ "items": [ { "lot_id", "quantity_received", "reason"? } ] }`; diferença exige `reason` (10–500) → `received_with_difference`; a diferença volta como `loss` com motivo.
- `GET /pharmacy/transfers?location_id=&status=&before=`.

## 6. Inventário e ajustes (step-up)

- `POST /pharmacy/adjustments` — `{ "location_id", "lot_id", "kind": "adjustment"|"loss"|"expiry", "counted_quantity"? | "quantity"?, "reason" }`; acima de `adjustment_approval_percent` do saldo do lote → `pending`.
- `POST /pharmacy/adjustments/:id/approve|reject` — outro usuário (`stock_keeper` ou `pharmacist`) do mesmo local; 403 `same_user`.

## 7. Dispensação

- `POST /pharmacy/prescriptions/lookup` — `{ "location_id", "short_code" }` ou `{ "location_id", "patient_id" }` → `{ "prescriptions": [<prescription_view>] }` (trilha).
- `POST /pharmacy/external_prescriptions` — `{ "patient_id" | "new_patient": {…forma do 24a}, "prescriber": { "name", "council", "council_state", "council_number" }, "issued_on", "notification_number"?, "items": [ { "catalog_item_id", "quantity", "quantity_unit", "dosage_instructions", "continuous" } ] }` → `<prescription_view>` (auditada).
- `POST /pharmacy/dispensations` — `{ "location_id", "prescription": { "kind", "id" }, "items": [ { "prescription_item_id", "catalog_item_id"?, "lot_id", "quantity" } ], "retention"?: { "notification_number" } }` → 201 `<dispensation>`. Recusas: 422 `prescription_cancelled`, `prescription_expired`, `too_early` (com `next_allowed_on`), `insufficient_balance` (com `path`), `controlled_requires_pharmacist` (403), `free_text_requires_pharmacist` (403), `retention_required`, `lot_expired`.
- `GET /pharmacy/dispensations?patient_id=&before=` (trilha).

## 8. Prontuário

- Linha do tempo (20): item `dispensation` com data, unidade e medicamentos (sem lote).
- Lista de medicamentos do paciente: cada item ganha `last_dispensed_on` (só leitura; `patient_medications` não muda).

## 9. Alertas (sem trilha)

`GET /pharmacy/alerts?location_id=` → `{ "expiring", "expired", "below_minimum", "in_transit", "pending_adjustments" }`.

## 10. Maintenance

Interruptor `pharmacy` (rótulo e descrição; mecanismo genérico).

## 11. Eventos (só ids)

`stock_receipt.recorded`, `stock_transfer.sent`, `.received`, `stock_adjustment.recorded`, `.approved`, `.rejected`, `dispensation.recorded`, `external_prescription.recorded`, `pharmacy.settings_changed`.

## 12. Ordem de entrega

1. `api` (porta 3049; migrações de cidade a partir de 20261029900001; worktree `.claude/mod27a`, branch `feat/mod-27a-pharmacy-stock`; testes com `bin/test api -w mod27a`).
2. `dashboard` (Vite 5205), worktree em `rota-saude/.worktrees/dashboard-mod27a`.
3. `maintenance` (prova na 5177 com o serviço do compose parado).
4. Catálogo público.

## 13. Acréscimos (revisão dos planos, 2026-10-29)

Reconciliação dos planos do api (P1–P18), do dashboard (D1–D25) e do maintenance (M1–M4).

- **H1 — Unidade de dispensação (P1, corrige o §2):** `medication_catalog_items.unit` é a unidade da concentração; a unidade de dispensação sai de `dosage_form` por tabela YAML do api e fica gravada no lote (`<lot>.item.dispensing_unit`). Quantidades com até 2 casas, máximo 99999,99, sempre como string `"30.00"` na saída.
- **H2 — Tabelas e rotas (P2, P3, D1–D7, D10, D12, D25):** tabelas auxiliares do plano do api (`stock_location_staff`, `pharmacy_settings`, `stock_entries`, `stock_openings`, `stock_transfer_items`, `stock_adjustments`, `dispensation_items`, `external_prescription_items`). Rotas novas: `GET /pharmacy/locations` (meus locais com `roles`; o `stock_keeper` do central vê todos), `POST /pharmacy/items/search`, `POST /pharmacy/person_lookup` (par validado vira paciente, como no 24a), `POST /pharmacy/patients/duplicates`, `GET /pharmacy/adjustments[/:id]`, `GET /pharmacy/transfers/:id`, `POST /pharmacy/returns`. `GET` dos mínimos traz `item`; posição aceita `catalog_item_id=`; `<transfer>` e `<adjustment>` com as formas do plano do api, `<adjustment>.by` com `user_id`; itens de `<dispensation>` com `id`, nome do item e número do lote. Abertura duplicada → 409 `opening_exists` (também pelo farmacêutico). Staff sem e-mail na resposta.
- **H3 — Validade e intervalo (P4, P5, P9, P15, P16):** `valid_until` e `expired` também por item; o intervalo de 25 dias conta por item da receita; receita do 19c emitida em papel também é dispensável; busca por paciente olha os últimos 365 dias, até 20 receitas; receita externa: conselho CRM, CRO ou COREN, posologia 3–500, número da notificação obrigatório nas listas A/B (`notification_required`), validade fixa de 30 dias para controlado.
- **H4 — Ajustes (P6, D17):** só o ajuste de contagem acima do limite vai para aprovação; perda e vencimento são aplicados na hora; o farmacêutico também faz abertura e ajuste.
- **H5 — Transferência (P7, D18):** diferença no recebimento = `transfer_in` do enviado + `loss` da diferença no destino, com motivo.
- **H6 — Retenção (P8, D14):** controlado (Portaria 344) só pelo farmacêutico, com retenção; **antimicrobiano** (RDC 471/2021) exige retenção da 2ª via por quem dispensa (farmacêutico ou auxiliar); `retention: {}` com o número tirado do documento quando houver.
- **H7 — Prontuário (P10, D22, D23):** a dispensação aparece na linha do tempo só para o profissional (wpda não muda), com `status: "dispensed"` e filtro `kinds[]=dispensation`, e continua visível com o interruptor desligado; `last_dispensed_on` por item da lista de medicamentos.
- **H8 — Recusas (P12, D20, D21):** recusas do plano do api, mais `prescription_not_found`; em escrita com vários itens (entrada, dispensação, transferência, receita externa), o 422 traz `problems: [ { "path", "error" } ]` com todas as recusas de uma vez.
- **H9 — Eventos e papéis (P11, P17, P18, D19):** evento novo `dispensation.returned`, payloads do plano do api (só ids); `pharmacist` e `pharmacy_assistant` em `PRIVILEGED_ROLES`, `stock_keeper` não; vínculo ao local exige a `Membership` ativa do papel e, para `pharmacist`, CBO `2234*`.
- **H10 — Listas (P14, D15, D24):** posições ordenadas por (local, item) com cursor opaco; alertas são contagens (as listas da tela vêm da posição); faixas das configurações do plano do api.
- **H11 — Maintenance (M1–M4):** interruptor "Farmácia e almoxarifado" sem `requires` nem `missing`, sem `environments`, descrição do `CATALOG` do api; catálogo com o id `"27a"`.
- **H12 — Cruzamento do §13 com as tasks (2026-10-29):**
  - **Antimicrobiano (detalha o H6):** a marca vem do catálogo da plataforma (`medication_catalog_items.antimicrobial`, alimentado pela lista Anvisa) para a receita digital **e** a externa — quem digita não marca; o api calcula `retention_required: bool` por item da `<prescription_view>` (antimicrobiano, controlado ou `category` de controle especial do 19c); retenção de antimicrobiano por farmacêutico ou auxiliar, com `notification_number: null`; o catálogo público cita a RDC 471/2021 na F-27.6.
  - **Go-live:** o interruptor vale para todo `/pharmacy/*`; o runbook manda ligar `pharmacy`, cadastrar locais, equipe, parâmetros e mínimos e fazer o inventário de abertura antes de anunciar a dispensação (sem saldo, a dispensação já recusa com `insufficient_balance`).
  - **Devolução:** entra no dashboard no 27a (histórico de retiradas → "Registrar devolução"), consumindo `POST /pharmacy/returns`.
  - **Duplicados:** `POST /pharmacy/patients/duplicates` na forma e com a trilha do 25; receita externa com `new_patient` aceita `duplicates_reviewed` ou recusa com 409 `duplicate_suspected` (24a L28).
  - **`problems` (detalha o H8):** `error` e `path` do primeiro problema no topo; `problems: [ { "path", "error", "next_allowed_on"? } ]`, incluindo recebimento de transferência e as recusas apuradas sob trava (`insufficient_balance`, `too_early`, `item_completed`); 403 de papel vem sozinho.
  - **Receita externa vencida:** `prescription_expired` (`path: "issued_on"`) só quando todos os itens já nascem vencidos.
  - **Formas (B1–B14 do relatório de execução):** `item.unit` = concentração e `item.dispensing_unit` em todo `<item>`; mínimos com envelope `{ "minimums": [...] }`; `<adjustment>` com `requested_by: { user_id, name }`, `reason`, `balance_at_request`, `created_at`; itens de `<dispensation>` com `item` e `lot` embutidos; escritas sem envelope, DELETE de equipe 204; linha do tempo com `detail_ref: null`; posição aceita `catalog_item_id` e trata `false` como ausente; quantidades `numeric(12,2)` ≤ 99999,99 em todas as colunas, embalagem 1–10000; `prescription_not_found` 404; extrato com `reason`; `complete` opcional por item da dispensação; `catmat_code` inteiro; recusas `invalid_request`, `citizen_not_verified`, `invalid_cpf`, `invalid_cns`, `cbo_not_allowed`.
- **H13 — Limites (api Task 2, detalha o H1):** o teto de 99999,99 vale para quantidade digitada e para movimento; saldos (`balance_at_request`, `counted_quantity`, posição) não têm teto (`numeric(14,2)`, `>= 0`). Ajuste com `|contado − saldo|` acima de 99999,99 → 422 `invalid_quantity`. Item em texto livre mapeado para controlado ou antimicrobiano: o api recusa sem retenção (422 `retention_required`) e a tela abre a retenção e reenvia.
- **H14 — Unidade desativada (toca o módulo 09):** com `pharmacy` utilizável, desativar a `HealthUnit` cuja farmácia tem transferência `sent` (de ou para ela) ou ajuste `pending` → 409 `pharmacy_pending`; sem farmácia ou sem o interruptor, o 09 não muda. Todo 422 do `/pharmacy/*` traz `problems`, inclusive as recusas estruturais.
