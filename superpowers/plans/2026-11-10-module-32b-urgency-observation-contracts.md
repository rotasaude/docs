# Módulo 32b — Contratos entre apps

**Spec:** `superpowers/specs/2026-11-05-module-32-urgency-design.md`. Valem as decisões 2, 3 e 20–23, a R7 e o §3 "Cidade (32b)".
**ADR:** `adr/0050.md`.
**Contrato do 32a:** `superpowers/plans/2026-11-05-module-32a-urgency-contracts.md`. Continua valendo inteiro, de §1 a Q31.

Este contrato é a **fonte única** dos planos do api, do dashboard e do maintenance para o 32b. Quando um plano divergir dele, a divergência vai para a seção final "Divergências propostas ao contrato" do próprio plano. O coordenador reconcilia aqui, na §12.

**Bases** (origin/main em 2026-11-10):

| Repo | Commit |
|---|---|
| api | `69d283c4` |
| dashboard | `979e881` |
| maintenance | `bbff803` |

O 32b não mexe no contracts nem no wpda.

**Valores fixos:**

| Item | Valor |
|---|---|
| Porta do api | 3055 |
| Vite do dashboard | 5215 |
| maintenance | 5177, com o serviço do compose parado durante a prova |
| Migrações da cidade | a partir de `20261110900001`; nenhuma de plataforma |
| Branch | `feat/mod-32b-urgency-observation` |
| Worktree do api | `apps/api/.claude/mod32b`, com `config/master.key` copiada |
| Worktrees dos fronts | `rota-saude/.worktrees/<repo>-mod32b` |

**Mudança transversal.** O 32b toca `stock_movements` e o `Pharmacy::Ledger` (27a), o fechamento do encontro (32a), o BPA (22) e a lista de interruptores. A suíte completa do api é obrigatória antes do merge.

## 1. Nomes no código

**Interruptores:**
- A observação e a prescrição fazem parte de `urgency_care`. Não há interruptor novo para elas.
- O interruptor novo é `urgency_stock_deduction`, com `requires: %w[feature:urgency_care feature:pharmacy]`. A descrição é "Atendimentos da UPA com prescrição de medicação atualizam o estoque".
- Desligado, a administração é registrada sem lote e sem movimento de estoque. A observação e a prescrição funcionam igual.
- A chave nova entra no fim do `CATALOG` de `Platform::Features`. Os specs que fixam a ordem das chaves são atualizados.

**Papéis.** Nenhum papel novo. Toda ação clínica exige vínculo ativo na UPA, como no 32a (`Urgency::Access`):

| Ação | Quem |
|---|---|
| Cadastro de leitos | `municipal_admin`, com step-up |
| Mapa de observação (leitura) | recepção, enfermagem (2235\*, 3222\*) e médico (225\*) da UPA |
| Entrada, troca de leito e saída da observação | médico (225\*) ou enfermeiro (2235\*) |
| Prescrição, alteração e suspensão | médico (225\*) |
| Checagem (administração) | enfermeiro (2235\*), técnico de enfermagem (3222\*) ou médico (225\*) |
| Faltas da UPA (leitura e conciliação) | farmacêutico e almoxarife do local `unit` da UPA, pelo `Pharmacy::Access` |

O técnico de enfermagem (3222\*) entra só na checagem. Ele continua fora da classificação (Q2 do 32a).

**Tabelas novas da cidade.** Nenhuma tem `patient_id`, porque o paciente é sempre o do `attendance` do encontro. Por isso a fusão do provisório (Q5/Q17/Q21/Q24/Q25 do 32a) não precisa mexer nelas, e a guarda `urgency_merge_tables_spec` não muda.

- **`urgency_beds`:**
  - colunas: `health_unit_id` (só UPA), `label` (até 20 caracteres, único por unidade), `active`, `updated_by_user_id`;
  - nunca é apagada (`rota_no_delete`).
- **`urgency_bed_stays`:**
  - colunas: `urgency_encounter_id`, `urgency_bed_id`, `health_unit_id`, `started_at`, `ended_at`, `end_kind` (`bed_change` | `observation_ended` | `outcome`), `started_by_user_id`, `ended_by_user_id`;
  - exclusion por leito com `tsrange(started_at, COALESCE(ended_at, 'infinity'))`, no padrão do `excl_psychosocial_night_stays_bed`;
  - índice único parcial de uma estadia aberta por encontro.
- **`urgency_prescriptions`:**
  - colunas: `urgency_encounter_id`, `version` (1, 2, …), `status` (`active` | `superseded` | `suspended` | `ended`), `supersedes_id`, `prescribed_by_user_id`, `professional_link_id`, `cbo_code`, `prescribed_at`, `suspension_reason` (cifrado), `status_changed_at`;
  - índice único parcial de uma `active` por encontro.
- **`urgency_prescription_items`:**
  - colunas: `urgency_prescription_id`, `position`, `catalog_item_id` (do `medication_catalog_items` da plataforma, sem FK), `catmat_code`, `dose` (numeric), `dose_unit`, `route` (`PrescriptionItem::ROUTES`), `frequency_kind` (`once` | `interval` | `if_needed`), `interval_hours` (1–24, só em `interval`), `max_per_day` (só em `if_needed`), `first_dose_at`, `instructions` (cifrado, opcional), `stock_quantity_per_dose` (numeric > 0, padrão 1, na `dispensing_unit` do item);
  - só itens do catálogo. Não há texto livre, porque o estoque precisa do `catalog_item_id`.
- **`urgency_administrations`:**
  - colunas: `urgency_prescription_item_id`, `urgency_encounter_id`, `scheduled_at` (nulo em `if_needed`), `status` (`administered` | `not_administered` | `refused`), `reason` (cifrado; obrigatório fora de `administered`), `administered_at`, `stock_lot_id`, `stock_quantity`, `recorded_by_user_id`, `professional_link_id`, `cbo_code`, `recorded_at`;
  - índice único parcial (`urgency_prescription_item_id`, `scheduled_at`) quando `scheduled_at` não é nulo.
- **`urgency_stock_shortfalls`** (R7):
  - colunas: `urgency_administration_id` (único), `stock_location_id` (nulo quando a UPA não tem local), `catalog_item_id`, `quantity`, `cause` (`no_location` | `insufficient_balance` | `lot_unavailable`), `reconciled_at`, `reconciled_by_user_id`, `reconciliation_note` (cifrado).

**Mudanças em tabelas existentes:**

| Tabela | Mudança |
|---|---|
| `stock_movements` | `kind` ganha `administration` (negativo) e uma FK nova, `urgency_administration_id`. Os CHECKs `_kind`, `_sign` e `_document` são ajustados |
| `Pharmacy::Catalog` | `MOVEMENT_KINDS` e `NEGATIVE_KINDS` ganham `administration` |
| `Pharmacy::Ledger` | `DOCUMENTS` ganha `UrgencyAdministration`; o modelo a seguir é o do 29 (`record_immunization`) |
| `config/urgency/sigtap.yml` | ganha `observation_code` (padrão `0301060029`), com `review: true` |
| `Urgency::Config::KEYS` | ganha `observation_code` |

**Estado do encontro.** Não há estado novo. A observação acontece com o encontro em `in_care`. Quem está em observação é quem tem uma estadia aberta. A guarda `rota_urgency_encounter_guard` não muda.

## 2. Formas

Valem as formas do 32a (§2):
- erros `{error, path}` mais `problems[]`;
- datas ISO 8601 no fuso da cidade;
- `<patient-ref>`;
- escritas que recusam chave desconhecida (Q23).

O detalhe do encontro (`GET /urgency/encounters/:id`) ganha três blocos, sob as regras de acesso clínico do Q19 e com trilha:
- `observation`: a estadia aberta e as anteriores;
- `prescription`: a versão ativa com os itens;
- `administrations`: as últimas 48 h.

## 3. Leitos (admin)

- `GET/POST/PATCH /urgency/beds {health_unit_id, label, active}`, para `municipal_admin` com step-up.
- Unidade que não é UPA → 422 `not_an_upa`.
- Rótulo repetido na unidade → 422 `label_taken`.
- Desativar um leito ocupado → 409 `bed_occupied`.
- Desativar uma UPA com leito ativo → 422 `upa_has_beds`, no padrão do `caps_has_beds`.

## 4. Observação

**Mapa.** `GET /urgency/observation?health_unit_id` devolve:

```
{beds: [{bed_id, label, active, stay: null | {stay_id, encounter_id, ticket, patient: <patient-ref>, started_at, hours, alert}}], waiting_bed: []}
```

- `hours` conta desde a **primeira** entrada em observação do encontro. A troca de leito não zera a conta.
- `alert` vale `null`, `"near_limit"` (a partir de 20 h) ou `"over_limit"` (a partir de 24 h).
- O mapa não traz texto clínico e não grava trilha (é polling, como a fila do 32a). O dashboard atualiza a cada 30 s, com `clearTimeout` ao desmontar.

**Alertas de 20 h e 24 h.** São calculados na leitura do mapa e da fila, sem tabela, job ou evento. A fila do 32a ganha, em `in_care[]`, os campos `observation: {bed_label, hours, alert} | null`. Passar de 24 h não bloqueia nada: a decisão é clínica, e o painel do gestor conta esses casos.

**Entrada e troca de leito.** `POST /urgency/encounters/:id/observation {bed_id}`.
- Exige o encontro em `in_care`. Fora dele → 422 `not_in_care`.
- Leito inativo ou de outra unidade → 422 `bed_unavailable`.
- Leito ocupado → 409 `bed_taken`. A conferência é feita em Ruby e garantida pelo exclusion, como no 31.
- Com uma estadia já aberta, a chamada é uma **troca de leito**: fecha a estadia atual com `bed_change` e abre a nova, na mesma transação.

**Saída sem desfecho.** `POST /urgency/encounters/:id/observation/end` fecha a estadia com `observation_ended`, e o paciente volta ao atendimento. Sem estadia aberta → 422 `not_in_observation`.

**Saída com desfecho.** `Urgency::RecordOutcome` fecha a estadia aberta com `outcome`, no mesmo instante e na mesma transação do desfecho. A ordem de lock é attendance → encontro → estadia → prescrição.

**Reavaliação e evolução.** Não há tabela nova. A evolução durante a observação é um **adendo da consulta `urgency`** do 19a, que já existe depois da finalização. A tela de observação dá acesso direto ao adendo.

## 5. Prescrição interna

**Emitir ou alterar.** `POST /urgency/encounters/:id/prescriptions {items: [...]}`.
- Só o médico (225\*) com vínculo na UPA.
- Exige o encontro aberto e a consulta `urgency` iniciada. Sem consulta → 422 `consultation_required`.
- De 1 a 20 itens por versão.
- Cada item: `{catalog_item_id, dose, dose_unit, route, frequency: {kind, interval_hours?, max_per_day?}, first_dose_at?, instructions?, stock_quantity_per_dose?}`. `first_dose_at` vale "agora" quando omitido.
- Se já existe uma versão `active`, ela passa a `superseded` e a nova nasce `active` com `supersedes_id`. Alterar sempre cria uma versão nova; a anterior não muda.
- Item fora do catálogo ou escondido → 422 `catalog_item_invalid`.
- Controlado: aceito em todas as listas, com `controlled_list` copiado do catálogo (§12, P1).

**Suspender.** `POST /urgency/prescriptions/:id/suspend {reason}`, com motivo de 10 caracteres ou mais. Só a versão `active` pode ser suspensa.

**Fim.** O desfecho leva a versão `active` a `ended`. A transição de `status` só anda para a frente (trigger):
- `active` → `superseded`, `suspended` ou `ended`;
- as demais são finais.

**Aprazamento.** `GET /urgency/encounters/:id/medication_schedule` devolve, por item da versão ativa:
- `slots`: os horários de `first_dose_at` até 24 h à frente, cada um com a administração, se houver;
- em `if_needed`, as administrações das últimas 24 h e quantas ainda cabem no `max_per_day`.

Os horários são calculados na leitura, sem tabela. Grava trilha.

## 6. Checagem (administração)

`POST /urgency/administrations {prescription_item_id, scheduled_at?, status, reason?, administered_at?, stock_lot_id?, stock_quantity?}`.

**Quem e quando:**
- Só enfermagem (2235\*, 3222\*) ou médico (225\*) com vínculo na UPA.
- Item de versão que não está `active` → 422 `prescription_not_active`.
- Encontro fechado → 409 `encounter_closed`.

**Horário e status:**
- `interval` e `once` exigem um `scheduled_at` que seja um dos horários do aprazamento. Fora dele → 422 `slot_invalid`. Horário já checado → 409 `slot_taken`.
- `if_needed` não aceita `scheduled_at`. Acima de `max_per_day` nas últimas 24 h → 422 `max_per_day_reached`.
- `not_administered` e `refused` exigem `reason` de 10 caracteres ou mais.

**Imutável.** A administração não pode ser alterada nem apagada (trigger). Não há correção no 32b; um registro errado fica como está, e a correção vira pendência de ciclo (§11).

**Estoque.** Só quando `urgency_stock_deduction` está utilizável e `status = administered`:
1. **Local:** o `stock_location` `unit` ativo da UPA. Sem local → a administração é gravada e nasce uma falta `no_location`.
2. **Lote:** `stock_lot_id`, se veio no corpo. Senão, o api escolhe por **FEFO**: o lote do `catalog_item_id` não vencido, com saldo, de menor `expires_on`, e então `lot_number` e `id`. Um lote vencido, de outro item ou sem saldo, vindo do corpo → 422 `lot_invalid`.
3. **Quantidade:** `stock_quantity`, ou `stock_quantity_per_dose` do item, na `dispensing_unit` do lote.
4. **Movimento:** o `Pharmacy::Ledger.record!` grava `kind: administration`, com o documento `urgency_administration_id`, na mesma transação da administração. A ordem de lock é a do Ledger: lotes → local → documento.
5. **Sem saldo:** se o FEFO não achar saldo suficiente, ou o Ledger levantar `Insufficient`, a administração é gravada **sem** movimento, e nasce uma falta (`insufficient_balance` ou `lot_unavailable`). O razão nunca fica negativo, e a checagem nunca é recusada por falta de estoque.

**Interruptor desligado:** `stock_lot_id` e `stock_quantity` no corpo → 422 `stock_deduction_disabled`. Não há movimento nem falta.

## 7. Faltas da UPA (farmácia, 27a)

- `GET /pharmacy/urgency_shortfalls?stock_location_id&state=open|reconciled` devolve `{shortfall_id, created_at, catalog_item, quantity, cause, administration: {administered_at, ticket}}`. Não traz o nome do paciente nem dado clínico, e não grava trilha.
- `POST /pharmacy/urgency_shortfalls/:id/reconcile {note}`, com nota de 10 caracteres ou mais. A falta já conciliada → 409 `already_reconciled`.
  - A farmácia resolve no estoque com o que o 27a já tem (ajuste ou transferência) e só então marca a falta.
  - Conciliar não grava movimento.
- `Pharmacy::Alerts` ganha a contagem `urgency_shortfalls` das faltas abertas do local.
- Exige `pharmacy` utilizável, mas não `urgency_stock_deduction`: as faltas antigas continuam visíveis depois de desligar o interruptor.

## 8. BPA (22)

`Billing::FromUrgency` ganha a observação:
- Se o encontro teve estadia em observação, a linha do **atendimento médico** sai com `observation_code` (`0301060029`, atendimento de urgência com observação até 24 h) **no lugar** do `care_code`, com o médico da consulta. O acolhimento com classificação não muda.
- As outras regras do 32a continuam: provisório pendente, `NON_CARE_OUTCOMES` sem linha.
- A administração de medicamentos (`0301100012`) não gera linha no 32b. A conferência entra no gate SIGTAP, rotasaude/api#136.

## 9. Linha do tempo e painel do gestor

**Linha do tempo:**
- Na visão profissional e na do PEC (Q27), a entrada `urgency` ganha `observation_hours` (inteiro ou nulo).
- Prescrição e administração não entram na linha do tempo.
- O cidadão não vê nada novo.

**Painel do gestor** (`GET /urgency/dashboard`), bloco novo `observation`:
- `stays`: encontros com observação;
- `over_24h`;
- `median_hours` e `p90_hours`;
- `bed_hours`.

As contagens seguem a supressão "lt5" do 32a.

## 10. Eventos (só ids)

| Evento | Payload |
|---|---|
| `urgency.observation_started` | `encounter_id`, `stay_id` |
| `urgency.observation_ended` | `encounter_id`, `stay_id` |
| `urgency.prescription_issued` | `encounter_id`, `prescription_id` |
| `urgency.prescription_suspended` | `encounter_id`, `prescription_id` |
| `urgency.medication_checked` | `encounter_id`, `administration_id` |
| `urgency.stock_shortfall_recorded` | `shortfall_id` |
| `pharmacy.urgency_shortfall_reconciled` | `shortfall_id` |

- Todos ficam ligados em `config/initializers/domain_events.rb` com `to: []` (só trilha).
- Entram na lista exata do `urgency_secrecy_spec`.
- Não há `Platform.audit` novo e nada muda no R18.
- A troca de leito publica `observation_ended` e `observation_started`.

## 11. LGPD, imutabilidade e ordem de entrega

**Cifra:** `instructions`, `reason`, `suspension_reason` e `reconciliation_note`. Todos entram no `CityEncryption::CITY_KEYED_TARGETS`.

**Nunca em URL, log ou evento:** nome, CPF, CNS, CID, queixa, medicamento e motivo.

**Imutabilidade** (bloco "Módulo 32b" em `db/city_triggers.sql`, e TRUNCATE recusado nas tabelas novas):

| Tabela | Regra |
|---|---|
| itens e administrações | só acréscimos, mais a recifragem |
| estadia | só recebe o fechamento uma vez |
| prescrição | só a transição de status para a frente |
| falta | só recebe a conciliação uma vez |
| leitos | nunca apagados |

**Exclusão (ADR 0026):** o encontro com observação ou prescrição é registro de saúde e fica retido, como no 32a. Nada muda no `Citizens::Erase`.

**Pendências de ciclo já conhecidas** (viram cards só com autorização no fechamento):
- correção de uma checagem errada;
- BPA da administração de medicamentos (P2, no api#136).

**Ordem de entrega:** api → dashboard → maintenance.

**Dashboard:**
- **Leitos:** o cadastro fica em "Senhas, locais e TVs", renomeado para "Configuração da UPA" (ou numa tela irmã no grupo "Urgência"). A decisão é do plano, registrada na §12.
- **Mapa e prescrição:**
  - o mapa de observação é um painel novo na mesa da UPA (`UrgencyDesk`);
  - a prescrição e a checagem ficam no painel de atendimento (`UrgencyCare`), entre a consulta e o desfecho;
  - a checagem da enfermagem também aparece no painel do enfermeiro.
- **Faltas da UPA:** tela nova no grupo "Farmácia".
- **Interruptor:** `FeatureKey` ganha `urgency_stock_deduction`.

**Maintenance:**
- rótulo "Baixa de estoque na medicação da UPA" em `FEATURE_LABELS`;
- `MISSING_LABELS` para `urgency_care_disabled` e `pharmacy_disabled`, se o api os devolver;
- teste e parágrafo no README.

## 12. Reconciliação dos planos

**Regra:** onde um plano diverge desta seção, vale esta seção. As origens são: A = api, D = dashboard, M = maintenance.

### Decisões do usuário (2026-11-10)

**P1 — Controlados na prescrição interna (opção A).**
- A prescrição interna da UPA aceita **todas** as listas da Portaria 344 (A, B e C), porque é uso interno, registrado no prontuário, e não exige notificação de receita.
- A regra do 19d (`requires_notification`, `not_supported`, `controlled_not_allowed`) **não** se aplica à prescrição interna. Continua valendo para a receita do 19, que é para levar.
- Com `urgency_stock_deduction` ligado, a administração de um controlado gera o movimento `administration` como qualquer item.
- A escrituração oficial dos controlados continua no gate do SNCR, rotasaude/api#63.
- O item guarda `controlled_list` como cópia do catálogo, e o detalhe e a checagem mostram a marca "controlado".

**P2 — BPA da administração (opção A).** O 32b fatura só a observação (§8). O `0301100012` entra na conferência do gate SIGTAP, rotasaude/api#136, e pode ser ligado depois.

### Reconciliação dos planos (2026-11-10)

Planos: api (Tasks 0–19, A1–A32), dashboard (Tasks 0–12, D1–D19) e maintenance (Tasks 0–2, M1–M4). Onde esta seção não fala de uma divergência, vale a proposta do plano.

**Q1 — Leitos** (D1, A2, A3).
- **Tela:** o cadastro fica numa tela irmã, "Leitos de observação", no grupo Urgência. Ela aparece com `urgency_care` e para `municipal_admin`. Fica fora de "Senhas, locais e TVs", que depende de `waiting_room_display`.
- **Rótulo repetido:** 409 `bed_label_taken`, no padrão do 31. Isso corrige o §3.
- **`upa_has_beds`:** 409, tanto ao desativar a UPA quanto ao trocar o `kind` dela.

**Q2 — Técnico de enfermagem (3222\*)** (D2, A17).
- **O que ele lê:** o mapa, a fila e o aprazamento. Também registra a checagem.
- **O que ele não lê:** o detalhe do encontro.
- **No código:** o `Urgency::Access` ganha um vínculo de enfermagem (2235\* ou 3222\*) para essas rotas. A classificação continua só com o 2235\* (Q2 do 32a).

**Q3 — Mapa e fila** (A1, D3, A25, A29, D19).
- **`waiting_bed`:** são os encontros em `in_care` sem estadia aberta.
- **Contagem de horas:** começa na primeira entrada do encontro, mesmo depois de sair e voltar. O limite de 24 h é da observação do atendimento, não de cada estadia.
- **Recepção:** lê o mapa de qualquer UPA, como a fila (Q26 do 32a). Isso já está no card api#134.
- **Quem perde o vínculo:** recebe 403 `missing_link` ou `cbo_not_allowed`, nunca 404.

**Q4 — Prescrição** (A8, A9, A14, A32, D6, D7, D13, D14, A31).
- **Novas colunas:** o item ganha `controlled_list`, copiado do catálogo (P1). A prescrição ganha `status_changed_by_user_id`.
- **`first_dose_at`:** aceito de agora − 1 h até agora + 24 h.
- **`dose_unit`:** domínio fechado, conforme o plano do api.
- **Limites, códigos e caminhos (`items.<n>.<campo>`):** seguem os planos do api e do dashboard.
- **Busca de medicamento:** rota nova, `POST /urgency/medication_search {query}`, sob `urgency_care` e sem exigir `clinical_documents`. Devolve `{catalog_item_id, catmat_code, name, strength, dosage_form, unit, controlled_list}`. O dashboard usa essa rota, e não o `MedicationSearch` do 19.
- **"Encerrar observação" sem desfecho:** a prescrição continua ativa, e a checagem segue.

**Q5 — Checagem e aprazamento** (A13, A15, A16, A19, D5, D8, D9, D18).
- **Antecipação:** um horário pode ser checado até **60 min antes** de vencer. Mais cedo que isso → 422 `slot_not_due`. Horários passados podem ser checados enquanto o encontro estiver aberto. Como a checagem é imutável, isso impede registrar como dada uma dose que ainda não foi dada.
- **`administered_at` e `max_per_day`:** seguem a A15 e a A16. O `max_per_day` conta só `administered`.
- **Dose em dobro entre versões:** o mesmo `catalog_item_id` no mesmo horário, em qualquer versão da prescrição → 409 `slot_taken`. O aprazamento traz `last_administration` de cada item.
- **Forma do aprazamento:** traz `stock_deduction`, que diz se o interruptor está utilizável. Quando estiver, traz também `lots[]` em ordem FEFO, sem saldo de outros locais. A enfermagem não lê o estoque da farmácia.
- **Resposta da checagem:** traz `shortfall: null | {cause}`. A falta não é erro.
- **Trilha:** a leitura do aprazamento grava trilha com o recurso novo `medication_schedule`.

**Q6 — Estoque** (A6, A7, A10, A11, A12, M3).
- **Formato das quantidades:** `dose` e as quantidades de estoque vão como texto decimal, como no `Pharmacy::Quantity`.
- **Unidade:** o item grava `stock_unit`, e o FEFO só considera lotes nessa unidade.
- **Lote:**
  - uma administração usa um lote só, sem fracionar;
  - o FEFO ignora lote bloqueado (`StockLotBlock`);
  - um lote bloqueado enviado no corpo → 422 `lot_invalid`.
- **Causas da falta:** seguem a A11.
- **Interruptor:** onde o contrato diz "desligado", leia "desligado ou não utilizável", tanto na regra do estoque quanto no 422 `stock_deduction_disabled` (§1 e §6). O plano do api cobre isso com um request spec.

**Q7 — Faltas da UPA** (A4, A5, A24, D16, D17).
- **Filtro:** o parâmetro é `location_id`, como no 27a.
- **Falta `no_location`:** é lida e conciliada por quem tem papel no local **central**. A lista do central mostra essas faltas de todas as UPAs.
- **Alertas:** `/pharmacy/alerts` ganha a 6ª chave, e as listas fixas dos specs são atualizadas.
- **Menu:** a tela aparece mesmo sem a urgência ligada. Isso está aceito.

**Q8 — Evolução em plantões diferentes** (D10). Enquanto o encontro estiver aberto, **qualquer médico (225\*) com vínculo na UPA** pode:
- abrir o prontuário do encontro;
- escrever adendos na consulta `urgency`.

Uma observação de até 24 h passa por mais de um plantão, e o médico seguinte precisa evoluir. A regra vale só para o `kind: urgency` e só com o encontro aberto, e cada leitura e cada adendo gravam trilha. Fechado o encontro, volta a regra do 19a.

**Q9 — Detalhe, desfecho e fechamento pelo 13** (A18, A20, A21, A28).
- **Sem acesso clínico** (Q19 do 32a): o detalhe traz `observation`, mas não `prescription` nem `administrations`.
- **Administrações no detalhe:** as das últimas 48 h, de qualquer versão.
- **Eventos:** a saída com desfecho também publica `urgency.observation_ended`.
- **Fechamento pelo 13:** quando `Attendances::Close` fecha um `walk_in` de UPA com `urgency_care` desligado, ele fecha a estadia aberta com `outcome` e leva a prescrição ativa a `ended`, na mesma transação (passo condicional da Task 12 do api, aceito).

**Q10 — BPA, linha do tempo, painel e eventos** (A22, A23, A26, A27, A30, D11, D12, D15).
- **BPA:** a linha da observação nasce com a consulta finalizada, porque o desfecho que fecha já a exige. A amostra SIGTAP de dev aceita só alguns CIDs no `0301060029`, e a semente usa um deles.
- **PEC:** na página do PEC, `observation_hours` vai como texto no resumo. No JSON, vai como chave só nas entradas `urgency`.
- **Extrato do 27a:**
  - ganha o documento `urgency_administration`;
  - o dashboard mostra "administração na UPA" para o `kind` e para o `document.type`;
  - os kinds do 29 sem rótulo ficam fora do 32b.
- **Painel:**
  - as horas saem com uma casa decimal;
  - a supressão `lt5` vale para o bloco inteiro, pelo número de estadias, inclusive `bed_hours`. A A30 prevalece sobre a D12.
- **Eventos:** são duas listas, a do registro em `domain_events_bindings_spec` e um cenário novo do 32b no `urgency_secrecy_spec`.

**Q11 — Maintenance** (M1–M4).
- **Faltas devolvidas:** o api devolve `urgency_care_disabled` e `pharmacy_disabled`, na ordem do `requires`, e nunca a falta do pré-requisito do pré-requisito.
- **Frases:** são as da Task 1 do maintenance, e o dashboard usa as mesmas.
- **Comentário do 32a em `features.ts`:** perde a frase sobre o 32b.
