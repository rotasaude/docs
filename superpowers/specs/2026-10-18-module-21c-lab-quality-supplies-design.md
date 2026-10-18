# Módulo 21 — Exames e laboratório, subprojeto 21c (qualidade e insumos do laboratório municipal) — design

**Data:** 2026-10-18
**Status:** aprovado (2026-10-18)
**Afeta:**
- `apps/api`: cidade — `lab_instruments`, `lab_qc_materials`, `lab_qc_lots`, `lab_qc_targets`, `lab_qc_runs`, `lab_qc_values`, `lab_qc_blocks`, `lab_qc_exceptions`, `lab_qc_review_items`, `lab_qc_rules`, `lab_eqa_rounds`, `lab_supplies`, `lab_supply_lots`, `lab_supply_movements`, `lab_supply_in_use`; vínculo exame × aparelho nas versões do exame; lote de reagente em `lab_results`; interruptores `lab_quality_control` e `lab_inventory`; controle pela HL7; bloqueio na liberação do 21b; `fake-middleware` mandando controles.
- `apps/dashboard`: aparelhos, CIQ (materiais, lotes, referências, corridas, Levey-Jennings, bloqueios, exceções, revisões, regras), CEQ, estoque (itens, lotes, movimentos, lote em uso, alertas, rastreio, consumo).
- `apps/maintenance`: interruptores (mecanismo genérico).
- `contracts`: seção de controle no perfil HL7 — tag `lab-v1.1.0`.

**ADR:** `docs/adr/0038.md` · **Módulo:** `docs/modulos/21--exames.md` · **Base:** spec do 21b (`superpowers/specs/2026-10-15-module-21b-municipal-lab-design.md`) e contrato do 21b (§14) · **Pesquisa:** `docs/pesquisa/2026-10-15-laboratorio-municipal-21b.md` (CIQ/CEQ, RDC 978 arts. 171–188)

**Fora desta entrega:** importação de controles por CSV com colunas configuráveis (rotasaude/api#87); baixa automática de estoque por teste; compras, pedidos e fornecedores; estoque fora do laboratório (módulo 27); calibração e manutenção de aparelhos além do registro de corrida; integração com portais de provedores de CEQ.

## 1. Ponto de partida

- 21b (ADR 0037): fluxo analítico do laboratório municipal — catálogo com versões, amostras, resultados digitados ou pela HL7 (`OUL^R22`, `instrument_test_code`), liberação por `lab_releaser` com RT e substituto, retificação, críticos.
- RDC Anvisa 978/2025 (pesquisa de 2026-10-15): gestão da qualidade obrigatória com CIQ e CEQ (arts. 171–172), em todos os equipamentos e analitos; registro dos controles (art. 174), avaliação e registro dos resultados e relatório anual do CEQ (art. 176); liberação ou rejeição das análises após avaliar as amostras controle, com registro de inadequações e ações (art. 180, III–IV); CIQ no mínimo a cada corrida e manutenção (art. 183, §2º). A RDC exige o registro, não que esteja no LIS.
- Editais: CIQ com Westgard/Levey-Jennings em 2 de 9; estoque de reagentes no laboratório em 1 de 9.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | CIQ completo (materiais, lotes, níveis, referências, Levey-Jennings, Westgard automático, bloqueio) e registro do CEQ. |
| 2 | Estoque com lotes, lote em uso ligado a cada resultado e valor de controle, e relatório de consumo estimado, sem baixa automática. |
| 3 | Valores de controle digitados e pela HL7; importação CSV configurável vira rotasaude/api#87. |
| 4 | CIQ reprovado bloqueia a liberação do analito no aparelho; RT ou substituto libera exceção com justificativa; resultados liberados desde a última corrida aprovada vão para revisão. |
| 5 | Referência começa pela bula; após 20 corridas aprovadas o sistema sugere a do laboratório e o RT aceita (nova versão). |
| 6 | Abordagem 1: tabelas da cidade, avaliação de Westgard na gravação, bloqueio gravado. |

## 3. Interruptores

- `lab_quality_control` (`requires: ["feature:municipal_lab"]`): CIQ e CEQ.
- `lab_inventory` (`requires: ["feature:municipal_lab"]`): estoque.
Independentes entre si.

## 4. Dados (cidade)

**Aparelhos**
- `lab_instruments`: `exam_provider_id`, `name`, `manufacturer`, `model`, `serial_number`, `sector`, `hl7_equipment_id` (para identificar na HL7), `active`.
- As versões do exame do 21b ganham `lab_instrument_id` (opcional), ligando exame e aparelho.

**CIQ**
- `lab_qc_materials`: `exam_provider_id`, `name`, `manufacturer`, `levels` (lista de nomes), `analyte_keys` (com `lab_exam_id`), `active`.
- `lab_qc_lots`: `lab_qc_material_id`, `lot_number`, `expires_on`, `level_barcodes` (nível → código de barras, únicos na cidade e distintos dos códigos de amostra), `status` (`in_use` | `closed`), `opened_on`, `closed_on`.
- `lab_qc_targets` (imutável): `lab_qc_lot_id`, `level`, `analyte_key`, `lab_instrument_id`, `version`, `source` (`package_insert` | `laboratory`), `mean`, `sd`, `accepted_by`, `accepted_at`, `based_on_runs` (contagem, quando `laboratory`).
- `lab_qc_runs`: `lab_instrument_id`, `sector`, `started_at`, `status` (`approved` | `warning` | `rejected` | `open`), `evaluated_at`.
- `lab_qc_values` (só acréscimo): `lab_qc_run_id`, `lab_qc_lot_id`, `level`, `analyte_key`, `value`, `source` (`typed` | `hl7`), `hl7_receipt_id`, `z_score`, `target_version`, `rules_fired` (lista), `outcome` (`ok` | `warning` | `reject`), `reagent_lot_id`, `entered_by`, `entered_at`.
- `lab_qc_rules`: por `lab_instrument_id` + `analyte_key` (ausente = padrão): regras ligadas entre `1_2s` (aviso), `1_3s`, `2_2s`, `R_4s`, `4_1s`, `10x` (reprovação); desligar exige justificativa e RT.
- `lab_qc_blocks`: `lab_instrument_id`, `analyte_key`, `opened_by_run_id`, `opened_at`, `corrective_action`, `closed_by_run_id`, `closed_by`, `closed_at`.
- `lab_qc_exceptions` (só acréscimo): `lab_qc_block_id`, `lab_requisition_item_id`, `justification`, `granted_by` (RT ou substituto), `granted_at`.
- `lab_qc_review_items`: `lab_qc_block_id`, `lab_requisition_item_id`, `decision` (`pending` | `kept` | `rectify`), `decided_by`, `decided_at`, `note`.
- `lab_eqa_rounds`: `exam_provider_id`, `program` (`PNCQ` | `Controllab` | texto), `round_code`, `round_date`, `items` (por analito: enviado, alvo, aceitável, `adequate` | `inadequate`, ação), `annual_report` (PDF cifrado, opcional), `recorded_by`.

**Estoque**
- `lab_supplies`: `exam_provider_id`, `name`, `kind` (`reagent` | `calibrator` | `control` | `consumable`), `unit`, `tests_per_package` (reagente), `minimum_stock`, `expiry_alert_days` (padrão 30), `lab_exam_ids`, `active`.
- `lab_supply_lots`: `lab_supply_id`, `lot_number`, `expires_on`, `supplier`, `origin` (`purchase` | `donation` | `loan`).
- `lab_supply_movements` (só acréscimo): `lab_supply_lot_id`, `kind` (`receipt` | `opening` | `loss` | `expiry` | `adjustment`), `quantity` (com sinal), `reason`, `document`, `recorded_by`, `recorded_at`.
- `lab_supply_in_use`: `lab_supply_lot_id`, `lab_instrument_id`, `lab_exam_id`, `started_at`, `ended_at`.
- `lab_results` (21b) ganha `reagent_lot_id` (nulo = sem rastreio).

## 5. CIQ

1. **Cadastro** (admin do laboratório ou RT): material, lote com níveis e códigos de barras, referência da bula por nível, analito e aparelho.
2. **Corrida.** Valor de controle entra digitado (tela da corrida, por nível) ou pela HL7 (§7). A corrida do aparelho no dia abre sozinha no primeiro valor; uma corrida nova abre após uma reprovada ou a pedido.
3. **Avaliação na gravação.** z = (valor − média) / dp contra a referência vigente; regras de Westgard ligadas, sobre o histórico do mesmo lote, nível, analito e aparelho (e entre níveis na mesma corrida para R-4s e 2-2s entre níveis). Só `1_2s` → corrida `warning`. Regra de reprovação → corrida `rejected` e bloqueio aberto para aparelho e analito.
4. **Bloqueio na liberação (21b).** Item cujo analito tem bloqueio ativo no aparelho do exame não é liberado (`qc_blocked`), inclusive resultados que entraram antes da reprovação e ainda não liberados.
5. **Exceção.** RT ou substituto libera o item bloqueado com justificativa (a liberação usa a assinatura de sempre); a exceção fica registrada e no relatório; o laudo não muda.
6. **Revisão.** Na reprovação, os itens daquele aparelho e analito liberados desde a última corrida aprovada entram na lista de revisão; o RT decide `kept` ou `rectify` (abre a retificação do 21b).
7. **Fechamento.** Ação corretiva registrada + nova corrida aprovada com todos os níveis do analito fecham o bloqueio.
8. **Referência do laboratório.** Com 20 corridas aprovadas no lote (por nível, analito, aparelho), o sistema sugere média e dp calculados; o RT aceita e vira nova versão vigente.
9. **Telas:** corridas do dia por aparelho; Levey-Jennings (lote, nível, analito, aparelho; ±1/2/3 dp; regras marcadas; versão de referência visível); bloqueios ativos; revisões pendentes; exceções; regras por analito.
10. **CEQ.** RT registra cada rodada (enviado, alvo, adequação, ação para inadequado) e anexa o relatório anual; tela de rodadas e taxa de adequação por analito.

## 6. Estoque

1. **Cadastro** de itens (unidade, mínimo, rendimento em testes, exames ligados).
2. **Entrada** de lote (número, validade, fornecedor, origem, quantidade, documento); lote vencido recusado.
3. **Abertura** (saída) e **lote em uso** por aparelho e exame; marcar um novo encerra o anterior do mesmo aparelho e exame.
4. **Perda, vencimento, ajuste** com motivo; ajuste só admin.
5. **Saldo** = soma dos movimentos; saída maior que o saldo recusada.
6. **Alertas** (tela): vencimento dentro de `expiry_alert_days`; abaixo do mínimo; resultado de exame/aparelho sem lote em uso entra marcado "sem rastreio de reagente".
7. **Rastreio por lote:** requisições, exames e datas dos resultados e controles com aquele lote; CSV.
8. **Consumo estimado** (mês, item, aparelho): testes realizados (resultados + repetições + controles dos exames ligados) × testes das aberturas (embalagens abertas × rendimento); desvio em %; tela e CSV; nada é baixado.

## 7. HL7 do controle

- `POST /hl7/v2` (21b). Amostra cujo `SPM-2` é código de barras de nível de lote de controle cadastrado é tratada como controle: valor vai para a corrida de CIQ, avaliado, nunca vira resultado de paciente. Aparelho identificado por `EQU-1` ou `OBX-18` contra `lab_instruments.hl7_equipment_id`.
- Recusas `AE` (códigos no perfil): lote de controle vencido ou encerrado, analito sem referência, aparelho desconhecido.
- A convenção exata (marca de controle em `SPM`/`SAC`/`INV`/`OBX`, identificação do aparelho) é prova técnica (§11) e entra no perfil `lab/hl7-v2-profile.md`.
- `fake-middleware` passa a mandar corridas de controle (aprovada e uma reprovada 2-2s).

## 8. Papéis

- `lab_staff`: corridas, entradas, aberturas, perdas, lote em uso.
- `lab_releaser`: o de `lab_staff` + Levey-Jennings e CEQ.
- RT e substituto: aceitar referência do laboratório, exceção, revisão, desligar regra, fechar bloqueio.
- `municipal_admin`: cadastros (aparelhos, materiais, itens), ajuste de inventário, relatórios.

## 9. Segurança e trilha

- Corridas, valores, referências, exceções, revisões e movimentos só de acréscimo (trigger).
- Exceção, revisão, regra desligada, ajuste e troca de referência exigem motivo e ficam na trilha.
- Eventos só com ids: `lab_qc_run.evaluated`, `lab_qc_block.opened`, `.closed`, `lab_qc_exception.granted`, `lab_qc_review.decided`, `lab_qc_target.accepted`, `lab_qc_rule.changed`, `lab_eqa_round.recorded`, `lab_supply_lot.received`, `lab_supply_movement.recorded`, `lab_supply_in_use.changed`.
- Rastreio por lote mostra número da requisição e código do exame; nome do paciente só para quem já tem acesso ao laboratório.

## 10. Testes

- **api:** cada regra de Westgard com séries conhecidas (dispara e não dispara); bloqueio abre e fecha; liberação bloqueada e exceção do RT; revisão desde a última aprovada; sugestão de referência após 20 corridas e nova versão; controle pela HL7 (aceite, recusas) e controle nunca vira paciente; saldo nunca negativo; lote em uso trocado sozinho; resultado e controle guardando o lote; rastreio e consumo; triggers; corrida real no bloqueio × liberação; guarda R18.
- **dashboard:** Vitest, incluindo o Levey-Jennings.
- **Prova no navegador** (`fake-middleware` mandando controles): lote com referência da bula → corrida aprovada → corrida reprovada (2-2s) → liberação bloqueada, exceção do RT e revisão → ação corretiva e corrida que fecha o bloqueio → sugestão de referência (semente com 20 corridas) → CEQ → entrada de reagente, abertura e lote em uso → resultado com lote e rastreio → consumo.

## 11. Entrega

0. Provas técnicas: (a) convenção HL7 v2.5.1 para resultado de controle e identificação do aparelho; (b) Levey-Jennings com a biblioteca de gráficos já usada no dashboard.
1. `contracts`: seção de controle no perfil HL7 — tag `lab-v1.1.0`.
2. `api` (migrações acima das do 21b; `fake-middleware` com controles).
3. `dashboard`. 4. `maintenance`. 5. catálogo público.

## 12. F-IDs (propostos)

| F-ID | Funcionalidade | Apps |
|---|---|---|
| F-21.25 | Interruptores e aparelhos do laboratório | api, dashboard, maintenance |
| F-21.26 | Materiais, lotes e referências de controle (bula e laboratório) | api, dashboard |
| F-21.27 | Corridas de CIQ e regras de Westgard | api, dashboard |
| F-21.28 | Controle pela HL7 | api, contracts |
| F-21.29 | Bloqueio da liberação, exceção do RT e revisão | api, dashboard |
| F-21.30 | Gráfico de Levey-Jennings | dashboard |
| F-21.31 | Registro do CEQ | api, dashboard |
| F-21.32 | Estoque de reagentes e insumos com lotes | api, dashboard |
| F-21.33 | Lote em uso e rastreabilidade do reagente | api, dashboard |
| F-21.34 | Relatório de consumo estimado | api, dashboard |

## 13. Riscos e pendências de go-live

- Middleware real mandando controles (convenção de cada fornecedor).
- Validação do CIQ pelo RT do laboratório (entra no dossiê do art. 106).
- Laboratórios que mantêm o CIQ no software do provedor de controle: o 21c convive digitando os valores das corridas até a importação CSV (api#87).
- Volume de valores de controle por banco de cidade (vários níveis × analitos × corridas por dia).
