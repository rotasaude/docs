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

## 11. Acréscimos (revisão dos planos, 2026-10-18)

Aceitos pelo coordenador; prevalecem sobre as seções acima. Detalhe nas seções finais de `2026-10-18-module-21c-lab-qc-{api,dashboard,maintenance}.md`.

**Números (corrige o cabeçalho deste contrato).** No JSON, decimais (valor, média, dp, CEQ) são **string com ponto**, como no 21b; `z_score` é número. A recusa do ponto é regra de **entrada nas telas** (o dashboard lê vírgula como decimal e recusa ponto, como na digitação do 21b); o api aceita a string com ponto que o dashboard envia.

**maintenance (M1–M3)**
- **C-M1:** `lab_quality_control` e `lab_inventory` sem `environments` (todos os ambientes, produção inclusive) e independentes.
- **C-M2:** o aviso do `fake-middleware` diz que ele manda uma corrida reprovada que abre bloqueio (também em staging).
- **C-M3:** `missing` reaproveita `municipal_lab_disabled` do 21b.

**dashboard (C1–C17), adotadas pelo api com estes ajustes**
- **C-1:** `GET /lab/context` ganha `technical_lead_of` (laboratórios onde o usuário é RT ou substituto).
- **C-2:** listas do 21c filtram por `provider_id` e têm envelope (`{ "<recurso>": [...], "next_before"? }`); `/lab/qc/runs` filtra por `instrument_id`; `/lab/instruments` também para o admin.
- **C-3:** formas de lote de controle, referências `{ current, history }`, item de revisão (com `requisition_id`), saldos, lote em uso, alertas e resumo do CEQ como no plano do dashboard (o api pode mandar campos a mais).
- **C-5:** `/lab/exams` com `id` e `analytes`, também para liberador e admin.
- **C-6:** `reason` obrigatório ao lançar referência que troca a vigente e ao aceitar a sugestão (coluna `reason` na referência).
- **C-7:** `GET /lab/qc/exceptions` (membros do laboratório, inclusive `lab_staff`).
- **C-8:** `GET /lab/inventory/movements?lot_id=`.
- **C-9:** itens do 409 `qc_blocked` trazem `instrument_name`.
- **C-10:** `<qc_value>` traz `lot_id` e `reagent_lot { id, lot_number }`|null.
- **C-11:** `GET /lab/qc/blocks/active/count` → `{ "count" }`, sem trilha, para membros do laboratório e admin.
- **C-12:** `POST /lab/eqa/uploads` e `GET /lab/eqa/rounds/:id/annual_report.pdf` (sem trilha; sem dado de paciente).
- **C-13:** aparelhos respondem com qualquer dos dois interruptores ligados (403 leva `feature: "lab_quality_control"`); a versão do exame devolve `lab_instrument_id`.
- **C-14:** `lab_staff` e `lab_releaser` registram a ação corretiva.
- **C-15:** quantidade de estoque inteira; o api dá o sinal, exceto no ajuste.
- **C-16:** leitura das regras com `is_default`; `justification_required` só quando alguma regra fica desligada.
- **C-17:** gráfico exige `from` e `to` (até 366 dias, 422 `invalid_period`); `target` `null` sem referência vigente; eixo em z contra a versão com que cada ponto foi avaliado.

**api (Q1–Q17)**
- **C-Q1/Q2 — HL7 do controle:** o `SPM-2` (código de barras de nível cadastrado) decide que é controle; `SPM-11` = `Q` recomendado (IHE LAW LAB-28/29); `Q` com código de paciente → `AE` `unknown_barcode` (controle nunca vira paciente); `INV` aceito e ignorado; `SPM-4` pode vir vazio; aparelho pela repetição de `OBX-18` que casa com `hl7_equipment_id` (`EQU` não faz parte do `OUL^R22` 2.5.1). Recusa nova `qc_disabled`. Corrida reprovada responde `AA` (é desfecho do CIQ, não erro da mensagem). Confirmado na Task 0 do api; se as fontes contradisserem, para.
- **C-Q3 — Fechamento:** corrida aprovada sem ação corretiva registrada grava o valor (201) com `notice: "corrective_action_required"` e o bloqueio continua; fecha só com valores de todos os níveis gravados depois da ação.
- **C-Q4/Q5/Q6 — Corridas e série:** "aprovada" = `approved` ou `warning`; `run_id` de corrida reprovada → 409; após reprovada a próxima corrida é nova; pela HL7, uma corrida por mensagem e aparelho; a série de Westgard ignora corridas reprovadas; limites estritos (|z| = 2 não dispara 1-2s; = 3 não dispara 1-3s).
- **C-Q7/Q9 — Versões:** regras guardadas como versões só de acréscimo; o valor guarda o id da referência usada; o aceite da sugestão recalcula no servidor.
- **C-Q8 — Códigos de barras de controle** em tabela própria; 10 dígitos (espaço das amostras) → `barcode_taken`; formato inválido → `invalid_barcode`.
- **C-Q10 — Revisão e exceção:** revisão cobre itens liberados desde a última corrida aprovada daquele aparelho e analito; exceção só para item ainda não liberado.
- **C-Q11 — Eventos a mais:** `lab_qc_block.action_recorded`, `lab_supply_lot.traced`.
- **C-Q12 — Estoque idempotente:** entrada com mesmo lote e mesma validade soma no lote existente; marcar lote já em uso não faz nada.
- **C-Q13 — "Teste" no consumo:** resultado, repetição e valor de controle dos exames ligados ao item (definição no plano do api).
- **C-Q14 — Erros:** a lista de códigos 422 novos do plano do api passa a valer como parte deste contrato.
- **C-Q16:** Levey-Jennings não abre para `lab_staff` (spec §8).
- **C-Q17:** CEQ com um item por analito em cada rodada.
- **C-Q15:** ver "Números" acima.

**Dev:** sem contas novas — `tecnica-lab@`, `biomedica@` (RT), `patologista@` (substituto) e o admin de Curitiba; semente `lib/lab_qc_crew.rb`: aparelhos `BIO-01` (Mindray BS-240) e `HEM-01` (Sysmex XN-350), controle de dois níveis (lote `CQB-2611`, códigos `CQB2611N1`/`CQB2611N2`), 20 corridas aprovadas de glicose e creatinina, estoque com um lote vencendo e um item abaixo do mínimo; `fake-middleware` com uma corrida aprovada por dia e a reprovada 2-2s sob demanda (`run.rb qc reject_2_2s`).

**Execução (pré-voo do api, 2026-10-18)**
- **C-E1 — Relatório anual do CEQ:** o JSON da rodada sempre traz `annual_report_upload_id` (null sem relatório).
- **C-E2 — Justificativa das regras (corrige C-16):** `justification_required` só quando a mudança desliga uma regra que estava ligada; religar ou manter não pede.
- **C-E3 — Ação corretiva:** texto em branco → 422 `invalid_text`; `corrective_action_required` é só o `notice` do 201 do valor (C-Q3), que o dashboard lê e mostra como aviso.
- **C-E4 — Quem registra a ação corretiva:** qualquer membro do laboratório (`lab_staff`, `lab_releaser`, C-14); o fechamento do bloqueio é automático (C-Q3). Exceção, revisão, regras e aceite de referência continuam só do RT ou substituto.
- **C-E5 — Período do Levey-Jennings (corrige C-17):** `from`/`to` opcionais, padrão últimos 31 dias; máximo 366 (`invalid_period`).
- **C-E6 — Lote em uso repetido:** marcar o lote que já está em uso → 200 com o registro atual (nada é criado).
- **C-E7 — Identificador HL7 do aparelho:** até 50 caracteres, no api e no dashboard.
- **C-E8 — Consumo sem rendimento:** `tests_opened` pode ser null (item sem `tests_per_package`); a tela mostra "—" e não calcula desvio.
- **C-E9 — Formato do decimal no api:** `^-?\d{1,9}(\.\d{1,4})?$`; vírgula ou formato malformado → 422 `invalid_value` (substitui a Q15 e o `TYPED` do plano do api).
- **C-E10 — Código de barras de controle:** `^[A-Za-z0-9._-]{3,30}$`, nunca só 10 dígitos (espaço das amostras, C-Q8); formato inválido → `invalid_barcode`. O perfil HL7 registra esta regra, o limite de C-E7 e o formato de C-E9 antes da tag `lab-v1.1.0`.
- **C-E11 — Motivo longo:** motivo ou justificativa com mais de 1000 caracteres → 422 `invalid_reason` (recusado, nunca cortado); em branco continua `reason_required` / `justification_required`.
- **C-E12 — Analito único no material:** um material de controle não pode ter a mesma `analyte_key` em dois exames (422 `duplicate_analyte_key` ao salvar o material); assim referência e valor ficam chaveados por `analyte_key` sem ambiguidade.
- **C-E13 — Código de barras de controle sem caixa:** unicidade e busca (inclusive do `SPM-2`) sem diferenciar maiúsculas de minúsculas.
- **C-E14 — 4-1s e 10x também entre níveis (Westgard clássico):** além da série do mesmo nível, 4-1s dispara com os últimos 4 valores do analito no aparelho somando níveis (ex.: 2 níveis × 2 corridas) do mesmo lado de 1 dp, e 10x com os últimos 10 (ex.: 2 níveis × 5 corridas) do mesmo lado da média; a série entre níveis usa a mesma ordem cronológica e ignora corridas reprovadas (C-Q6).
- **C-E15 — Série entre níveis limitada ao lote (corrige C-E14):** a série entre níveis do 4-1s e do 10x junta só os níveis do **mesmo lote de controle**, no mesmo analito e aparelho; trocar de lote recomeça a série (a referência muda com o lote).
- **C-E16 — Ação corretiva longa:** acima de 1000 caracteres → 422 `invalid_text`, o mesmo código do texto em branco (C-E3); `invalid_reason` (C-E11) fica para motivos e justificativas.
- **C-E17 — Data da lista de corridas:** `date` malformada → 422 `invalid_date` (não cai em hoje).
- **C-E18 — Corrida do dia:** o valor digitado vai para a última corrida do aparelho no dia; se ela estiver `rejected` ou não existir, abre-se uma nova (detalha C-Q5). Assim, depois de uma reprovação, ação corretiva e nova corrida aprovada fecham o bloqueio.
- **C-E19 — Retificação com analito bloqueado:** a retificação de item já liberado também fica `qc_blocked`, mas o RT ou substituto pode conceder exceção para ela (mesma rota e justificativa); assim um laudo errado pode ser corrigido com valor medido de forma confiável (outro aparelho, repetição validada), com registro.
- **C-E20 — Nota da revisão:** acima de 1000 caracteres → 422 `invalid_note` (nota é opcional; `invalid_reason` fica para motivos e justificativas obrigatórios).
- **C-E21 — "Desde quando" da revisão:** o marco é o `started_at` da última corrida aceita (`approved` ou `warning`) daquele aparelho e analito — conservador, nunca deixa liberação de fora.
- **C-E22 — Laudo ICP pendente na revisão:** itens de laudos `pending_signature` liberados depois do marco (C-E21) também entram na revisão quando o CIQ reprova; a finalização ICP não consulta o CIQ (uma assinatura feita não é descartada) — o RT decide na revisão se mantém ou retifica.
- **C-E23 — HL7 com paciente e controle:** numa mensagem com amostras de paciente e de controle, os controles são processados antes das amostras (ordem de travas aparelho → corrida → requisição), sem inverter a ordem da liberação.
- **C-E24 — Relatório anual do CEQ por programa e ano (substitui C-E1 e C-12):** o relatório anual é registro próprio, só de acréscimo, `lab_eqa_annual_reports` (`provider_id`, `program`, `year`, upload cifrado, `recorded_by`), um por laboratório, programa e ano (409 `already_recorded` no repetido); a rodada perde `annual_report_upload_id`. Rotas: `POST /lab/eqa/uploads` (como antes), `POST /lab/eqa/annual_reports` `{ "provider_id", "program", "year", "upload_id" }`, `GET /lab/eqa/annual_reports?provider_id=&year=`, `GET /lab/eqa/annual_reports/:id/pdf` (sem trilha). O resumo do ano (`/lab/eqa/summary`) indica se há relatório do programa. A migração do 21c ainda não publicada é ajustada no próprio arquivo.
- **C-E25 — Upload com propósito:** o relatório anual do CEQ só aceita upload criado por `POST /lab/eqa/uploads` (marcado com o propósito do CEQ); qualquer outro upload (laudo, guia) → 422 `invalid_file`. Rodada ou relatório de outro laboratório → 404 `not_found` (não revela existência). Ação de item do CEQ acima de 500 caracteres → 422 `invalid_items`.
- **C-E26 — Propósito do arquivo:** `exam_files.purpose` (nulo para os arquivos do 21a; `lab_eqa_annual_report` para o CEQ), na migração do 21c; `store`/`claim!` exigem o mesmo propósito, então upload de um fluxo nunca é reivindicado por outro. Lista ou criação com `provider_id` de outro laboratório → 403 `missing_role` (não revela registro); leitura de registro de outro laboratório → 404 (C-E25).
- **C-E27 — 404 também no estoque:** item, lote ou movimento de outro laboratório (lista de lotes por `supply_id`, movimentos por `lot_id`, criação de lote ou movimento em item alheio) → 404 `not_found`; lista filtrada por `provider_id` de outro laboratório → 403 `missing_role` (C-E26).
- **C-E28 — Saldo útil:** "abaixo do mínimo" (na lista de itens e no alerta) usa o saldo dos lotes **não vencidos**; lote vencido aparece no seu próprio alerta e não conta como estoque disponível.
- **C-E29 — Ano do CEQ:** `year` não inteiro (inclusive string) → 422 `invalid_year`.
- **C-E30 — Lote do valor calculado:** valor calculado (fórmula) herda o `reagent_lot_id` das medidas que o geraram; se elas têm lotes diferentes ou alguma não tem lote, o calculado fica sem lote (não conta como "sem rastreio" no alerta, pois não é medida). O rastreio por lote alcança o exame pelas medidas de origem.
- **C-E31 — Detalhes do lote em uso e rastreio:** aparelho ou lote de outro laboratório → 404 no `in_use` e no `trace`; na criação do lote em uso, aparelho desconhecido, inativo ou de outro laboratório → 422 `invalid_instrument`; o rastreio deduplica antes de paginar (ordem estável).
- **C-E32 — Consumo sem aparelho:** testes de exames cuja versão não tem aparelho entram numa linha própria `instrument_id: null` ("sem aparelho") por item, para a soma das linhas bater com o total; `tests_opened` e `deviation_pct` podem ser null (C-E8) e `deviation_pct` é número no JSON (vírgula só no CSV).
