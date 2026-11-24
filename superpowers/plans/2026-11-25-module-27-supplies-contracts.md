# Módulo 27 — Insumos — Contratos entre apps

**Fonte única** do que o api, o dashboard, o maintenance e o catalog combinam nos insumos. As linhas do §11 prevalecem sobre o texto anterior. Os planos (`-api.md`, `-dashboard.md`, `-maintenance.md`) seguem este documento. Uma divergência vai para "Divergências propostas ao contrato" e volta aqui como Q-n.

- **Spec:** `docs/superpowers/specs/2026-11-25-module-27-supplies-design.md`, aprovada em 2026-11-25, decisões 1–18.
- **Pesquisa:** `docs/pesquisa/2026-11-23-insumos-e-bnafar-27.md`.
- **Base:** 27a (ADR 0045), 27b (contrato de 2026-11-24), 19 (consulta), 20 (linha do tempo) e 22 (faturamento). Nada muda, salvo o que este documento diz.

**Convenções herdadas:**
- erros `{ "error" }` com `path`;
- step-up `401 mfa_required`;
- `403 feature_disabled`;
- listas `{ items, next_before }`;
- `expected_version` → 409 `<entidade>_changed`;
- CID, perfil clínico e dados de quem retira por terceiro só no corpo, cifrados, nunca em log, URL ou evento.

**Ordem e bases:** é executado depois do 26c. Usa o origin/main no dia da Task 0.

**Valores fixos:**

| Item | Valor |
|---|---|
| Porta do api | 3065 |
| Vite do dashboard | 5226 |
| maintenance | 5177 |
| Migrações | as seguintes às do 26c, de cidade e de plataforma (confira na Task 0) |
| Branch | `feat/mod-27-supplies` |
| Worktrees | `apps/api/.claude/mod27s` (com `master.key`) e `rota-saude/.worktrees/<repo>-mod27s` |
| Testes | `bin/test api -w mod27s …`, `bin/test <app> -w <repo>-mod27s …`; suíte completa pela sessão "Testes" |

**Mudança transversal.** Os insumos tocam:
- o estoque (quarta chave), a dispensação, a devolução e o `Stock::Domain`;
- a consulta (`Consultations::Finalize`);
- a linha do tempo;
- o faturamento;
- o catálogo da plataforma.

A suíte completa do api é obrigatória.

## 1. Interruptor e papéis

**Interruptores** (fim do `CATALOG`):
- `pharmacy_supplies`, com o rótulo "Insumos na farmácia" e `requires: %w[feature:pharmacy]`.
- `supply_programs`, com o rótulo "Programas de insumos (diabetes e estomia)" e `requires: %w[feature:pharmacy_supplies feature:clinical_record]`.
- A bolsa de estomia no faturamento exige também o interruptor de faturamento do 22. A chave exata é conferida na Task 0.

**Papéis:** os do 27a e os da consulta, sem papel novo (decisão 11). O farmacêutico aprova a exceção de cota (§4).

## 2. Catálogo de materiais e estoque

**Plataforma, `supply_catalog_items`**, no molde de `medication_catalog_items`:
- `catmat_code` (único), `class_code` (6515 | 6550), `pdm_code`, `pdm_name`, `description`, `supply_unit` (unidade de fornecimento);
- `sigtap_code` (opcional: as OPM de estomia, preenchidas pela correspondência da Task 0);
- `group` (`glucose_strip` | `lancet` | `insulin_syringe` | `pen_needle` | `ostomy_bag` | `ostomy_accessory` | `other`, atribuído na carga por regra sobre o PDM, revisável);
- `status`, `hidden`, as releases.
- **Carga:** `Medications::Catmat::Client` passa a receber `class_code:`, com o padrão 6505. `Supplies::CatalogImport.call(by:, class_codes: [6515, 6550])` cria a release `source: catmat_6515_6550`. A carga também roda pela rake `supplies:import_catmat`, que é como entra em produção, e pela mutation `importSupplyCatalog` do maintenance, fora de produção.

**Estoque:**
- `stock_lots.item_kind` ganha `supply`, com a chave `supply_catalog_item_id` e o `catmat_code`.
- A migração troca os CHECKs de lote, de mínimo, de abertura e de movimento, no molde do 35a.
- `Stock::Domain.of_lot`: `supply` → `MEDICATION`. O ramo de medicamento é reescrito com a lista explícita de chaves.
- A unidade de dispensação do insumo é a `supply_unit`.
- **27b:** o insumo vai com CATMAT pelo interruptor `bnafar_medications`, e entra na regra de pendência de código do 27b.

## 3. Inscrição

**Tabela da cidade `supply_enrollments`:**
- `program` (`diabetes` | `ostomy`), `patient_id`, `preferred_unit_id`;
- `cid10` (cifrado);
- `profile` (jsonb cifrado):
  - diabetes: `insulin_type` (`nph` | `regular` | `analog` | `none`), `applications_per_day` (0–10), `measurements_per_day` (0–10);
  - estomia: `stoma_type` (`colostomy` | `ileostomy` | `urostomy` | `other`).
- `quotas` (jsonb, congelado: `[{ rule_id, group, quantity, period_days }]`), `override_reason` (cifrado, opcional);
- `valid_until` (3 a 12 meses; laudo externo até 3);
- `status` (`active` | `inactive` | `closed`), `origin` (`consultation` | `external_report` | `import`);
- `opened_in_consultation_id`, `renewed_in_consultation_id`;
- laudo externo: `external_report` (jsonb cifrado: profissional, conselho, número, UF, data);
- `last_withdrawal_at`, `version`.
- No máximo uma inscrição `active` por (paciente, programa).

**Na consulta** (para não colidir com a coluna `consultations.supplies` da odontologia, o bloco se chama `supply_programs`):
- O rascunho vai em `draft_items["supply_programs"]`, pelo `PATCH consultations/:id` (`Consultations::SaveDraft`), validado por `SupplyPrograms::DraftInput`.
- Na finalização, `SupplyPrograms::Materialize.call` roda dentro do `Consultations::Finalize`, depois do `DentalCare::Materialize` e antes do `update!` de finalizado. Ele abre, renova ou encerra a inscrição.
- Quem pode inscrever: o médico (225*) e o enfermeiro (2235*).

**Pela farmácia** (`POST /pharmacy/supply_enrollments`):
- Quem pode: o farmacêutico.
- O pedido traz o laudo externo e `valid_until` de no máximo 3 meses.

**Importação dos pacientes atuais** (`supply_enrollment_imports`, no molde da Q11 do 27b):
- O CSV é enviado no dashboard pelo `municipal_admin`. Fica numa área de conferência e só entra com aplicação em step-up.
- O paciente fora do cadastro é criado pela busca de CPF/CNS. O que não casa vai para o relatório.

**Inativação:** `SupplyPrograms::InactivateJob` roda todo dia e passa a `inactive` quem está sem retirada há 180 dias. A reativação é feita na consulta ou pela farmácia, com laudo.

## 4. Cota e retirada

**Regras da cidade, `supply_quota_rules`:**
- `program`, `profile_match` (jsonb, por exemplo `{ insulin_type: ["nph","regular"] }`), `group`;
- `kind` (`fixed` | `per_prescription`), `quantity` (no `fixed`), `factor_field` (`measurements_per_day` | `applications_per_day`, no `per_prescription`);
- `period_days` (padrão 30), `cap` (opcional), `version`.
- Quem grava é o `municipal_admin`, em `PUT /pharmacy/admin/supply_quota_rules`.
- O cálculo de `per_prescription` é `factor × period_days`, limitado pelo `cap`.

**Saldo** (`SupplyPrograms::Balance.for(enrollment, group, now:)`):
- `quantity − (entregue − devolvido)` desde o início do período, contado a partir da primeira retirada do ciclo atual.
- O que não é retirado no período não acumula.
- O saldo soma todas as unidades.

**Dispensação:**
- `dispensations` ganha `supply_enrollment_id`. O CHECK passa a `num_nonnulls(clinical_document_id, external_prescription_id, supply_enrollment_id) = 1`.
- `dispensations` ganha `collector_name` e `collector_document`, cifrados e opcionais juntos.
- `dispensation_items`:
  - `catalog_item_id` passa a aceitar nulo, e entra `supply_catalog_item_id`;
  - `ck_dispensation_items_source` aceita a linha sem item de receita quando a dispensação é por inscrição;
  - entram `quota_override_reason` (cifrado) e `override_by_user_id`.
- **`Pharmacy::DispenseSupplies.call(location:, enrollment:, items:, collector:, by:, role:, now:)`**, comando novo:
  - confere a inscrição `active` e válida e o saldo;
  - acima do saldo, recusa com 422 `quota_exceeded`, salvo `override` do farmacêutico;
  - se quem atende é o auxiliar, a liberação pede step-up de um farmacêutico. A rota recebe `override_by` com o código.
  - Ele publica `dispensation.recorded`.
- **Rotas:** `GET /pharmacy/supply_enrollments?patient_id=` (com trilha), `POST /pharmacy/supply_dispensations`.

**Devolução:**
- `Pharmacy::RecordReturn` aceita o item de insumo, e o saldo considera a devolução.
- Bolsa de estomia → 422 `ostomy_return_not_allowed`.

## 5. Comodato

**Tabelas da cidade:**
- `device_models`: nome, tipo (`glucometer` | `cgm_reader`), fornecedor, tira compatível (`supply_catalog_item_id`), ativo.
- `device_loans`: `device_model_id`, `serial_number` (único entre os ativos), `patient_id`, `health_unit_id`, `term_accepted_at`, `loaned_at`, `returned_at`, `status` (`active` | `returned` | `replaced` | `lost`), `replaced_by_loan_id`, `police_report_number` (cifrado, só na perda), `version`.

**Rotas** `/pharmacy/device_loans` (criar, devolver, trocar, perda), pelo farmacêutico e pelo auxiliar.

**Na retirada:** a tela mostra a tira compatível do aparelho ativo do paciente.

## 6. Estomia no faturamento

- **`health_units`** ganha `ostomy_service` (bool; CHECK `kind <> 'ubs'`) e `ostomy_responsible_professional_id`.
- **Entrega da bolsa:** só numa unidade com `ostomy_service`. Fora dela, 422 `not_ostomy_service`.
- **Faturamento:**
  - `Billing::FromSupply` (job em `dispensation.recorded`) cria o `performed_procedure` com `source` `supply` (nova, com FK `dispensation_item_id`) e `sigtap_code` do item.
  - Os demais campos são o CNS e o CID da inscrição, o CBO do vínculo do responsável e a unidade.
  - Sem CNS do paciente, a entrega é recusada antes (422 `patient_without_cns`).
- **Limite do SIGTAP:** quantidade acima do `max_quantity` do SIGTAP no mês, somada à do paciente → 422 `sigtap_max_quantity`. O `Billing::Critic` não confere isso hoje.

## 7. Linha do tempo e prontuário

- `Timeline::Query::SOURCES` ganha `supply_enrollment` (aberta, renovada, encerrada), que só o profissional vê.
- `DispensationSource` mostra os rótulos do insumo.
- Na consulta, o painel do paciente mostra as inscrições ativas.

## 8. Telas

**Dashboard:**
- **Consulta:** "Programas de insumos" no `ConsultationEditor` (inscrever, renovar, encerrar), com a cota calculada à vista.
- **Farmácia:**
  - "Retirada de insumos": busca do paciente, inscrições e saldos, entrega por lote, exceção e retirada por terceiro;
  - "Inscrição com laudo externo";
  - "Aparelhos em comodato";
  - nas configurações, "Regras de cota";
  - "Importar pacientes dos programas".
- **Estoque:** entradas, transferências, posição e inventário passam a buscar o insumo.
- **Unidades:** os campos de ponto de estomia.
- **Frases:** `SUPPLIES_ERRORS` em `src/lib/suppliesErrors.ts`.

**Maintenance:**
- os dois interruptores;
- `MISSING_LABELS` com `pharmacy_supplies_disabled`;
- "Importar catálogo de materiais", fora de produção.

**Catalog:** o teste do catálogo público que proíbe "insumo|material" muda quando os insumos entrarem no catálogo, no fechamento.

## 9. Eventos e LGPD

**Eventos** (só ids):
- `supply_enrollment.opened`
- `supply_enrollment.renewed`
- `supply_enrollment.closed`
- `supply_enrollment.inactivated`
- `device_loan.changed`
- `supply_enrollment_import.applied`

A dispensação reaproveita `dispensation.recorded` e `dispensation.returned`.

**Sigilo:**
- O CID, o perfil, o laudo externo e quem retira ficam cifrados e nunca vão em log, URL nem lista.
- O CSV de importação é cifrado e apagado depois.
- Spec de sigilo: `spec/security/supplies_secrecy_spec.rb`.

## 10. Testes

**Api:**
- a quarta chave em todos os CHECKs e no `of_lot`;
- a carga CATMAT 6515/6550;
- a inscrição na finalização (abrir, renovar, encerrar), pela farmácia e por importação;
- a cota `fixed` e `per_prescription` com teto;
- o período a partir da primeira retirada;
- a exceção com o farmacêutico;
- a devolução que volta à cota e a da bolsa recusada;
- o comodato;
- a estomia no faturamento (ponto de estomia, CNS, limite do SIGTAP);
- a inativação em 180 dias;
- o sigilo;
- a suíte completa.

**Fronts:** vitest e prova no navegador.

## 11. Decisões (Q-n)

| # | Decisão |
|---|---|
| Q1 (coordenador, contrato) | O bloco da consulta se chama `supply_programs`, porque `consultations.supplies` já existe na odontologia. |
| Q2 (coordenador, contrato) | A inscrição nasce na finalização, pelo `SupplyPrograms::Materialize` dentro do `Consultations::Finalize`, no molde do plano odontológico. A farmácia só vê consulta finalizada (decisão 18). |
| Q3 (coordenador, contrato) | O ponto de estomia é um atributo da unidade (`ostomy_service`, nunca UBS), e o faturamento sai com o CBO do responsável configurado (decisão 13). |
| Q4 (coordenador, contrato) | O limite do SIGTAP para a bolsa é conferido na entrega, pelo comando dos insumos, porque o `Billing::Critic` não confere quantidade máxima. |
| Q5 (coordenador, contrato) | A carga do CATMAT de materiais em produção é pela rake, e fora de produção também pela mutation do maintenance, como na Q12 do 27b. |
| Q6 (api D1–D3; maintenance DM1–DM2) | **Catálogo de materiais.**<br>- As releases ficam em tabela própria, `supply_catalog_releases`, para não derrubar a release ativa do 6505.<br>- A unidade de fornecimento vem da PDM (`consultarMaterialUnidadeFornecimento`). Sem `UN`, o item fica com `unit_review`.<br>- O grupo do item sai de regras num YAML. Para revisar, edita-se o YAML e recarrega-se.<br>- No maintenance, a revisão é só leitura, e a carga não pede TOTP, como a de medicamentos. |
| Q7 (api D4–D5, D18; dashboard D1–D3, D10) | **Consulta e prontuário.**<br>- O bloco `supply_programs` segue a forma do Valor 5 do plano do api. O `PATCH` do rascunho devolve a cota calculada. A inscrição ganha `closed_in_consultation_id`.<br>- O painel do paciente lê `GET /patients/:id/supply_enrollments`, e o detalhe traz o CID, para o profissional em contexto.<br>- A lista de CIDs do diabetes é fixa em YAML: E10–E14 e O24.<br>- Os códigos da linha do tempo são os do plano do api.<br>- A linha BPA da bolsa não aparece para o cidadão. A `supply_enrollment` não vai para o PEC externo. |
| Q8 (api D6–D8; dashboard D8) | **Histórico e cota.**<br>- Entram a tabela `supply_enrollment_events` e a coluna `activated_at`, para a linha do tempo e para os 180 dias.<br>- `dispensation_items` copia o `supply_group` do item.<br>- As regras de cota são só de acréscimo (`replaced_at`), com uma versão única do conjunto. O `GET` e o `PUT` levam `expected_version`, e o `PUT` pede step-up. |
| Q9 (api D9; dashboard D6) | **Estoque.** A busca de item aceita `item_kind: "supply"`. Entrada, posição e mínimo aceitam `supply_catalog_item_id`. A tela trata insumo como um domínio próprio de busca, separado da receita. |
| Q10 (api D10; dashboard D4–D5) | **Retirada.**<br>- A exceção vai como `override { reason, pharmacist_user_id, code }`, com `GET /pharmacy/locations/:id/pharmacists` para escolher o farmacêutico.<br>- As formas da lista com saldos, da retirada e do laudo, os códigos de recusa e a dispensação com receita nula são os do plano do api. |
| Q11 (api D11–D12) | **Estomia.**<br>- O CNS e o limite do SIGTAP só são exigidos com o `procedure_billing` ligado. Sem o SIGTAP no item, a recusa é `supply_without_sigtap`. `max_quantity` é o `QT_MAXIMA_EXECUCAO`.<br>- O responsável do ponto de estomia precisa de vínculo ativo na unidade, e a unidade precisa de CNES. |
| Q12 (api D13–D15; dashboard D7, D9) | **Aparelhos e importação.**<br>- `device_models` é gravado pelo admin, com step-up, e lido por quem entrega.<br>- O CSV de importação tem cabeçalho fixo, o do plano do api. A conferência roda em `dry_run`: casa pelo CPF/CNS sem criar paciente, e o paciente só é criado na aplicação. Não há "desfazer".<br>- A planilha de saldo inicial do 27b ganha o `tipo_produto` `insumo`, acrescentado na execução dos insumos. |
| Q13 (api D16–D17; maintenance DM3) | **Convenções.**<br>- O "jsonb cifrado" segue o molde do código: coluna `text` com `serialize` e `encrypts`.<br>- Entram no R18 `supply_catalog.imported` e `maintenance.supply_catalog.imported`.<br>- O pré-requisito do `supply_programs` usa o motivo `clinical_record_disabled`, que já tem frase. |
