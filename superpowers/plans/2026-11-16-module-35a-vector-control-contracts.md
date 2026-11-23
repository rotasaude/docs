# Módulo 35a — Contratos entre apps

**Fonte única** do que os repositórios do 35a combinam entre si. As linhas do §20 prevalecem sobre o texto das seções anteriores. Os planos de cada repositório (`-contracts-repo.md`, `-api.md`, `-dashboard.md`, `-maintenance.md`, `-campo.md`) seguem este documento; divergência vai para a seção "Divergências propostas ao contrato" do plano, e a decisão volta aqui como Q-n.

- **Spec:** `docs/superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md` (decisões do usuário 1–7; do coordenador 1–13 e 23–30). **ADR:** 0054. **Pesquisa:** `docs/pesquisa/2026-11-14-zoonoses-e-endemias-35.md`. **Módulo:** `docs/modulos/35--zoonoses-e-endemias.md`.
- **Base:** contratos do 24a, 24b (§1–§16) e 24c (§1–§13, até a Q86); `field-sync-v1.2.0`. Nada deles muda, salvo o que este documento diz que muda.
- **Fica no 35b:** animal, vacinação antirrábica, ouvidoria, agressor, leishmaniose, canil, papel do veterinário, interruptor `zoonoses`.

Convenções herdadas (24a, 24b, 33):
- erros `{ "error" }`, com `path` nas recusas de validação;
- step-up `401 mfa_required`;
- interruptor desligado `403 feature_disabled`;
- JSON em toda escrita por cookie;
- listas `{ items, next_before }`, com cursor `before` opaco;
- filtros repetidos com `[]`;
- registro de outro escopo por id → 404; lista de escopo alheio → 403;
- leitura nominal com trilha e sem polling; contagens sem trilha;
- texto longo é recusado, nunca cortado;
- dado sensível só no corpo, nunca em URL nem em log;
- `Data`/`Struct` com dado pessoal sem esse dado em `inspect`/`to_s`/`as_json`.

**Bases** (origin/main em 2026-11-16):

| Repo | Commit |
|---|---|
| api | `469e2e68` |
| dashboard | `677a511` |
| maintenance | `fea9c29` |
| contracts | `f9d6df2` (`field-sync-v1.2.0`) |
| campo | `8429bd5` |

**Valores fixos:**

| Item | Valor |
|---|---|
| Porta do api | 3059 |
| Vite do dashboard | 5220 |
| maintenance | 5177, com o serviço do compose parado durante a prova |
| Migrações da cidade | a partir de `20261116900001` (a última na main é `20261113900001`); nenhuma de plataforma |
| Branch (todos os repos) | `feat/mod-35a-vector-control` |
| Worktree do api | `apps/api/.claude/mod35a`, com `config/master.key` copiada |
| Worktrees dos fronts, do contracts e do campo | `rota-saude/.worktrees/<repo>-mod35a` |
| Testes | só `bin/test api -w mod35a …`, `bin/test <app> -w <repo>-mod35a` e `bin/test campo -w campo-mod35a`; suíte completa pela sessão "Testes" |
| Tag do contracts | `field-sync-v1.3.0` (MINOR); `session.contract` continua `"field-sync-v1"` |

**Mudança transversal.** O 35a toca:
- migração de cidade e `CityEncryption::CITY_KEYED_TARGETS`;
- `Platform::Features` e `Membership::ROLES`;
- `HealthUnit::KINDS` e o estoque do 27a (`Stock::Domain`, `Pharmacy::Ledger::DOCUMENTS`, `stock_movements`);
- `HomeVisits::Record` e os vínculos de eventos do 25;
- o `field-sync`.

A suíte completa do api é obrigatória antes do merge.

## 1. Nomes no código

**Interruptor** (fim do `CATALOG` de `Platform::Features`; os specs que fixam a ordem são atualizados):

| Chave | Descrição | `requires` |
|---|---|---|
| `vector_control` | "Vigilância ambiental de endemias: boletim do agente na visita, quarteirões, ciclos, ovitrampas, LIRAa e bloqueio" | `%w[feature:home_visits]` |

- O rótulo curto, no maintenance e no dashboard, é "Vigilância ambiental (endemias)".
- A chave entra em `CITIZEN_HIDDEN`.
- Gate: `VectorControl::Gate.usable?(city)` = habilitado e `HomeVisits::Gate.usable?(city)`.
- Desligado: as rotas respondem `403 feature_disabled`, o item do app é `rejected feature_disabled`, e o que foi gravado fica.
- O job de bloqueio (§8) não age com o gate desligado.
- `home_visits` já exige `territory_registry`, então a decisão 1 da spec continua valendo (Q1).

**Papéis novos** (`Membership::ROLES` + `ck_memberships_role`, como o 33 fez em `20261111900001`; valem para a cidade inteira):

| Papel | Rótulo | Privilegiado | Faz |
|---|---|---|---|
| `endemic_coordinator` | "Coordenação de endemias" | sim (`PRIVILEGED_ROLES`: lê o endereço do caso de arbovirose, §8) | quarteirões e perfil do imóvel, cadência e ciclos, produtos e parâmetros, LIRAa, bloqueio, contagem de ovitrampa, resultado de amostra, tarefas, relatórios |
| `vector_lab` | "Laboratório de entomologia" | não | contagem de ovos e resultado de amostra; lê a fila de amostras e de palhetas |

`municipal_admin` lê os relatórios e não escreve nada do 35a. O acesso mora em `VectorControl::Access`:

| Método | Regra |
|---|---|
| `coordinator?(user)` | papel ativo `endemic_coordinator` |
| `lab?(user)` | papel ativo `vector_lab` |
| `reader?(user)` | um dos dois, ou `municipal_admin` |
| `field_agent?(user, team)` | `HomeVisits::Access.role(member) == :field` com o CBO `515140` (`TerritoryRegistry::Catalog::ENDEMIC_AGENT_CBO`) |

**Quem registra o boletim:** só o agente de combate às endemias (CBO `515140`), pela visita do 24b, nas microáreas atribuídas a ele (`HomeVisits::Access.registers?`). Visita com boletim de outro CBO → `403 cbo_not_allowed`, com `path` `vector_inspection`. Nada muda para o ACS. O `352210` fica fora: não está na Tabela 7 do LEDI (api#171).

**Unidade:** `HealthUnit::KINDS` + `environmental_surveillance`, rótulo "Vigilância ambiental e zoonoses" (UVZ, coordenação de endemias; decisão 30). Não dispensa, não tem fila e não tem leito. Os bloqueios de troca de tipo do `HealthUnitsController` valem como para `other`.

**Tabelas novas da cidade.** Todas `id: :uuid`, com `created_at`; as mutáveis também têm `updated_at` e `version` (≥ 1, `+1` a cada escrita, para o feed do app). "Cifrado" = `CityEncryption`, e entra em `CITY_KEYED_TARGETS`.

- **`city_blocks`** (quarteirão):
  - `neighborhood_id` (não nulo), `number` (texto 1–10, `/\A[0-9A-Za-z-]{1,10}\z/`), `active`, `created_by_user_id`;
  - único (`neighborhood_id`, `lower(number)`).
- **`city_block_neighbors`:** `city_block_id`, `neighbor_block_id`. Guarda o par uma vez, com `CHECK (city_block_id < neighbor_block_id)`, e é único pelo par. Pode cruzar bairros. Nunca é apagada: o par desfeito ganha `ended_at`.
- **`household_vector_profiles`** (1:1 com `households`, único por `household_id`):
  - `city_block_id` (nulo até alguém atribuir), `risk_class` (`green` | `yellow` | `red` | `gray` | `purple` | nulo), `risk_assessed_on` (date);
  - `last_inspection_id`, `last_worked_on` (date), `last_situation`, denormalizados pelo §3;
  - `updated_by_user_id`;
  - CHECK: `risk_assessed_on IS NOT NULL` ⇔ `risk_class IS NOT NULL`.
  - Não é versão do 24a: mudar quarteirão ou risco **não** cria `household_versions` nem ficha LEDI 3 (Q2).
- **`vector_inspections`** (1:1 com a visita, único por `home_visit_id`):
  - `activity`, `work_type`, `situation`, `recovered` (derivado), `educational_action` (bool);
  - `entered_at`, `left_at` (timestamptz, opcionais juntos);
  - `liraa_draw_item_id`, `transmission_block_id` (opcionais);
  - `city_block_id`: o quarteirão do imóvel na hora da visita, congelado.
- **`vector_inspection_deposits`:** `vector_inspection_id`, `group` (`A1` `A2` `B` `C` `D1` `D2` `E`), `existing`, `with_water`, `with_larvae`, `eliminated`, `treated` (inteiros 0–999). Único por (inspeção, grupo).
- **`vector_inspection_measures`:** `vector_inspection_id`, `method` (`mechanical` | `focal` | `perifocal` | `spatial`), `vector_product_id` (nulo só em `mechanical`), `charges` (numeric 8,2, > 0 e ≤ 999, nulo só em `mechanical`), `quantity` (numeric 12,2 = `charges × charge_size` do produto na hora). Único por (inspeção, método, produto).
- **`vector_samples`:**
  - `vector_inspection_id`, `number` (1–99, sequencial na inspeção), `deposit_group`, `larvae_collected` (1–10), `label` (único, §9);
  - resultado: `examined_larvae`, `aegypti_larvae`, `albopictus_larvae`, `result_at`, `result_by_user_id`, `result_note` (cifrado, 10–500, só na correção).
- **`vector_products`:** `code` (1–10, único sem caixa), `name` (1–80), `unit` (`g` | `ml` | `un`), `charge_size` (numeric 10,2, > 0), `uses` (lista não vazia de `focal` `perifocal` `spatial`), `active`.
- **`vector_cadences`:**
  - `neighborhood_id` (nulo = cidade), `risk_class` (nulo = qualquer), `strategic_point` (bool), `days` (7–365);
  - único por (`neighborhood_id`, `risk_class`, `strategic_point`), tratando os nulos como valor.
- **`vector_cycles`:** `year`, `number` (1–12), `starts_on`, `ends_on`. Único por (`year`, `number`), sem sobreposição de datas (exclusion constraint `daterange`).
- **`vector_parameter_versions`:** `version` (int, único), `values` (jsonb, §11), `created_by_user_id`. A vigente é a maior; a semente cria a versão 1 com o padrão de fábrica.
- **`vector_control_settings`** (singleton id 1): `stock_location_id` (o local do 27a que o boletim baixa; nulo = sem baixa, §4).
- **`ovitraps`:**
  - `number` (texto 1–12; único entre as ativas, sem caixa), `household_id`, `city_block_id` (congelado na instalação);
  - `consent_at`, `installed_on`, `removed_on`, `status` (`active` | `removed` | `lost`), `installed_by_user_id`, `client_uuid` (único, opcional);
  - `field_device_id`, `origin`.
- **`ovitrap_readings`:**
  - `ovitrap_id`, `installed_on` (palheta), `collected_on`, `condition` (`ok` | `damaged` | `lost` | `dry` | nulo até a coleta), `epi_week` (`YYYYWW`, da semana epidemiológica de `installed_on`);
  - `egg_count` (≥ 0, nulo até a contagem), `counted_at`, `counted_by_user_id`, `count_note` (cifrado, 10–500, só na correção);
  - no máximo uma leitura aberta (`collected_on IS NULL`) por ovitrampa.
- **`vector_tasks`:**
  - `kind` (`ovitrap_intensification` | `case_resident_elsewhere` | `block_case_without_place`);
  - `state` (`open` | `done` | `dismissed`), `ovitrap_reading_id` | `disease_notification_id` | `transmission_block_id`;
  - `suggested_block_ids` (uuid[]), `closed_by_user_id`, `closed_note` (cifrado, 10–500).
- **`liraa_surveys`:** `name` (1–80), `starts_on`, `ends_on`, `status` (`draft` | `drawn` | `closed`), `parameters_version`, `drawn_at`, `closed_at`.
- **`liraa_strata`:** `liraa_survey_id`, `name` (1–60), `mode` (`standard` | `half` | `census`), `households_count` (N na hora do sorteio), `sample_target` (n), `block_interval`, `random_start`.
- **`liraa_stratum_blocks`:** `liraa_stratum_id`, `city_block_id`, `drawn` (bool), `household_start` (1–5, nulo se não sorteado). Um quarteirão aparece em no máximo um estrato por levantamento.
- **`liraa_draw_items`:** `liraa_stratum_id`, `city_block_id`, `household_id`, `position`, `status` (`pending` | `inspected` | `substituted`), `substitute_of_id`.
- **`transmission_blocks`:**
  - `status` (`proposed` | `active` | `closed` | `dismissed`), `disease_group` (`arbovirus`), `center_block_id` (nulo até a coordenação escolher);
  - `confirmed_at`, `confirmed_by_user_id`, `closed_at`, `dismissed_reason` (cifrado, 10–500).
- **`transmission_block_areas`:** `transmission_block_id`, `city_block_id`. Único pelo par.
- **`transmission_block_cases`:** `transmission_block_id`, `disease_notification_id` (único), `case_on` (o `suspected_on` da notificação), `added_by` (`job` | `user`).
- **`transmission_block_cycles`:** `transmission_block_id`, `number` (1–3), `starts_on`, `ends_on`. Único por (bloqueio, número).
- **`vector_stock_shortfalls`:**
  - `vector_inspection_measure_id` (único), `stock_location_id` (nulo ⇔ `no_location`), `vector_product_id`, `quantity`;
  - `cause` (`no_location` | `insufficient_balance` | `no_lot`);
  - `reconciled_at`, `reconciled_by_user_id`, `reconciliation_note` (cifrado, 10–500).

**Colunas novas em tabelas existentes:**

| Tabela | Muda |
|---|---|
| `health_units` | `ck_health_units_kind` + `environmental_surveillance` |
| `memberships` | `ck_memberships_role` + `endemic_coordinator`, `vector_lab` |
| `stock_locations` | `ck_stock_locations_kind` + `vector_control`, com `health_unit_id` obrigatório e unidade do tipo `environmental_surveillance` (validado no comando, §4) |
| `stock_lots` | `item_kind` + `vector_control`; coluna `vector_product_id` (uuid); `ck_stock_lots_item`: o lote `vector_control` tem `vector_product_id`, não tem `catalog_item_id` nem `immunobiological_code`, e `dispensing_unit` = `unit` do produto |
| `stock_movements` | `kind` + `vector_application` (negativo); coluna `vector_inspection_measure_id`; o `ck_stock_movements_document` passa a contar essa coluna; `vector_product_id` ao lado de `catalog_item_id`/`immunobiological_code`, com `num_nonnulls(...) = 1`, como o 29 fez |
| mínimos, aberturas e afins do 27a que têm `num_nonnulls(catalog_item_id, immunobiological_code)` | passam a ter três colunas, como no 29 |

**Migrações da cidade** (irreversíveis, como as do 24a, 24b e 33: o `down` levanta `ActiveRecord::IrreversibleMigration` e o `up` termina executando `db/city_triggers.sql`; sem dado):

| Timestamp | Conteúdo |
|---|---|
| `20261116900001_add_vector_control_territory.rb` | `city_blocks`, `city_block_neighbors`, `household_vector_profiles`, `vector_cadences`, `vector_cycles`, `vector_parameter_versions`, `vector_control_settings`, papéis, tipo de unidade |
| `20261116900002_add_vector_inspections.rb` | `vector_products`, `vector_inspections`, `vector_inspection_deposits`, `vector_inspection_measures`, `vector_samples`, `vector_stock_shortfalls`, mudanças do estoque |
| `20261116900003_add_vector_surveillance.rb` | `ovitraps`, `ovitrap_readings`, `vector_tasks`, `liraa_*`, `transmission_blocks`, `transmission_block_*` |

**Mudanças em código existente:**

| Onde | Mudança |
|---|---|
| `HomeVisits::Record` | aceita `vector_inspection` (§3); `ALLOWED` + `vector_inspection`; deriva os motivos 34–37; grava a inspeção, as medidas, as amostras e a baixa de estoque na mesma transação |
| `HomeVisits::Json` | a visita ganha `vector_inspection` (§12) para quem lê a visita |
| `Stock::Domain` | `VECTOR_CONTROL = "vector_control"`, `FEATURES["vector_control"] = "vector_control"`, local `vector_control` |
| `Pharmacy::Ledger::DOCUMENTS` | + `VectorInspectionMeasure => :vector_inspection_measure_id` |
| `Pharmacy::Catalog` | `MOVEMENT_KINDS`/`NEGATIVE_KINDS` + `vector_application` |
| 27a: entrada, transferência, ajuste, perda | aceitam lote `vector_control` (produto do `vector_products`, sem CATMAT), só em local `vector_control`; fora dele → `item_not_allowed_here` |
| `config/initializers/domain_events.rb` | `disease_notification.created` e `disease_notification.transitioned` ganham `VectorControl::ProposeBlockJob` (§8); os eventos novos do §17 |
| `FieldSync::*` | §13 |
| `config/recurring.yml` | nada novo (os índices são lidos na hora) |

## 2. Quarteirão e perfil do imóvel

- A coordenação cadastra o quarteirão: bairro e número, e depois os vizinhos.
  - Vizinho do mesmo quarteirão → `422 invalid_neighbor`.
  - Quarteirão inativo não recebe imóvel novo, mas segue nos vizinhos e no histórico.
  - Desativar quarteirão com imóvel ativo → `409 block_in_use`.
- **Atribuir o imóvel:** a coordenação faz, no dashboard, em lote (`household_ids[]` até 200 + `city_block_id`); o agente de endemias faz pelo boletim (§3).
  - O imóvel tem de ter `neighborhood_id` na versão atual do 24a (`household_versions.neighborhood_id`). Sem ele → `422 household_without_neighborhood`. O imóvel só com `district` livre espera o ACS corrigir o cadastro: o 35a não escreve no 24a.
  - Quarteirão de outro bairro → `422 block_not_in_household_neighborhood`.
  - Domicílio que troca de bairro no 24a: o perfil fica, e a tela marca "quarteirão de outro bairro" até alguém atribuir de novo. O feed do app manda o perfil como está.
- **Classe de risco** (DN-2025, 5.3): `green` "Verde (baixo)", `yellow` "Amarelo (médio)", `red` "Vermelho (alto)", `gray` "Cinza (difícil acesso, com agendamento)", `purple` "Roxo (difícil acesso, exige intervenção legal ou VISA)". Quem define: a coordenação, ou o agente pelo boletim. `risk_assessed_on` é o dia da visita ou da edição.
- **Raio em anéis** (decisão 26): `VectorControl::Rings.around(block_ids, rings:)` devolve os quarteirões a até `rings` saltos pelo grafo de vizinhos ativos, incluindo os de partida, e ignora inativos. É a única função de raio do 35a. O raio geométrico pela coordenada do 24c fica para depois (§19).

## 3. Boletim na visita

O corpo da visita do 24b (`POST /home_visits` e item `home_visit` do app) ganha `vector_inspection`, opcional:

```json
"vector_inspection": {
  "activity": "li" | "li_t" | "pe" | "ie" | "transmission_block" | "liraa",
  "work_type": "routine" | "pending",
  "situation": "worked" | "closed" | "refused" | "vacant",
  "educational_action": true,
  "entered_at": "<iso8601>", "left_at": "<iso8601>",
  "city_block_id": "<uuid>"?, "risk_class": "<classe>"?,
  "liraa_draw_item_id": "<uuid>"?, "transmission_block_id": "<uuid>"?,
  "deposits": [ { "group": "A1", "existing": 3, "with_water": 2, "with_larvae": 1, "eliminated": 1, "treated": 1 } ],
  "measures": [ { "method": "focal", "vector_product_id": "<uuid>", "charges": 2 } ],
  "samples":  [ { "number": 1, "deposit_group": "A1", "larvae_collected": 8 } ]
}
```

**Regras** (`VectorControl::InspectionRules.check(input, visit:, household:, profile:, today:)`, pura, devolve `[{ error:, path: }]`):

1. Quem registra é agente de endemias (§1), e o gate está utilizável. Senão: `cbo_not_allowed` ou `feature_disabled`, com `path` `vector_inspection`.
2. **Situação e desfecho:** `worked` ⇔ `outcome` 1; `refused` ⇔ 2; `closed` e `vacant` ⇔ 3. Divergência → `situation_outcome_mismatch`.
   - Com 2 ou 3: `deposits`, `measures` e `samples` vazios, e `educational_action` false. Senão → `inspection_not_worked`.
3. **Atividade e imóvel:**
   - `pe` só com `property_type` 4;
   - `ie` só com `property_type` 5–13;
   - `li`, `li_t`, `transmission_block` e `liraa` com qualquer tipo;
   - divergência → `activity_property_mismatch`.
4. **Vínculos:**
   - `liraa` exige `liraa_draw_item_id` de item `pending` do mesmo imóvel, em levantamento `drawn`. Outro caso → `liraa_item_not_found`.
   - `transmission_block` exige `transmission_block_id` `active`, com o quarteirão do imóvel na área. Senão → `transmission_block_not_active`.
   - As outras atividades não levam vínculo. Vínculo enviado → `vector_inspection_invalid`, com `path` do campo.
5. **Depósitos:** cada grupo no máximo uma vez; `with_water` ≤ `existing`; `with_larvae` ≤ `with_water`; `eliminated` ≤ `existing`; `treated` ≤ `with_water`. Violação → `deposit_counts_inconsistent`, com `path` `vector_inspection.deposits[i].<campo>`.
6. **Medidas:**
   - `mechanical` sem produto e sem cargas.
   - As outras exigem produto ativo cujo `uses` contém o método. Senão → `product_not_allowed`.
   - `charges` > 0 e ≤ 999, com duas casas.
7. **Amostras:**
   - `number` de 1 a N, sem buraco;
   - o grupo tem `with_larvae` > 0 naquela inspeção;
   - nº de amostras do grupo ≤ `with_larvae` do grupo (um tubito por depósito positivo);
   - violação → `samples_exceed_larvae`, com `path` `vector_inspection.samples[i]`.
8. **Horários:** `entered_at` ≤ `occurred_at` ≤ `left_at`; `left_at` − `entered_at` ≤ 12 h; os dois no dia da visita. Senão → `inspection_times_invalid`. Os dois vêm juntos ou nenhum (o dashboard pode omitir; o app sempre manda).
9. **Motivos derivados** (decisão 4; a ficha 8 não muda de código):
   - O `reasons` do corpo não pode trazer 34–37 quando há `vector_inspection` → `reason_derived`, com `path` `reasons`.
   - O api acrescenta ao `reasons` gravado, em ordem crescente e sem repetir:
     - 34, se `educational_action`;
     - 35, se algum `with_larvae` > 0;
     - 36, se algum `eliminated` > 0 ou houver medida `mechanical`;
     - 37, se houver medida `focal`.
   - Depois disso, as regras do 24b rodam sobre a lista final: não residencial, `family_responsible` nos motivos 34–37 e `reasons_max`.
10. **Quarteirão e risco no boletim:**
    - `city_block_id` só quando o perfil ainda não tem quarteirão. Senão → `block_already_set`: a troca é da coordenação. O quarteirão segue as regras do §2.
    - `risk_class` atualiza o perfil, com `risk_assessed_on` = dia da visita.
11. **Derivados gravados:**
    - `recovered` = `work_type = pending` ∧ `situation = worked` ∧ a inspeção anterior do mesmo imóvel com `situation` ∈ {`closed`, `refused`};
    - o perfil ganha `last_inspection_id`, `last_situation` e, se `worked`, `last_worked_on`;
    - o item do LIRAa vira `inspected` com `worked`. Com `closed` ou `refused`, vira `substituted`, e nasce o substituto (§7).

A visita retirada (24b): a inspeção fica, marcada pela visita. Os índices, os ciclos e o perfil a ignoram, e o perfil volta à inspeção válida anterior. A baixa de estoque é estornada com `vector_application` de sinal inverso; o falta-estoque da medida é marcado `reconciled_at` pelo sistema, sem nota. O item do LIRAa volta a `pending`; se o substituto já estiver `inspected`, fica como está.

**Ficha LEDI tipo 8:** não muda. Os motivos derivados estão em `home_visits.reasons`, e a ficha os lê como hoje.

## 4. Produtos, estoque e consumo

- A coordenação cadastra os produtos. `unit` e `charge_size` não mudam depois que há lote ou medida → `409 product_in_use`; para isso se cria outro produto. `name`, `uses` e `active` mudam a qualquer tempo.
- Estoque (decisão 29):
  - O admin cria o local `vector_control` numa unidade `environmental_surveillance`, pela tela de locais do 27a.
  - A equipe do local (`stock_location_staff`, `stock_keeper`) dá entrada, transfere e ajusta pelas telas do 27a, com o produto no lugar do CATMAT.
  - A coordenação escolhe em `vector_control_settings.stock_location_id` o local de onde o boletim baixa.
- **Baixa:**
  - Acontece na transação da visita, num savepoint, pela medida com produto. O lote é o primeiro por `expires_on`, `lot_number`, `id`, do produto, no local, não vencido, sem bloqueio e com saldo ≥ `quantity`.
  - A baixa usa um lote só e nunca divide a quantidade, como o `Urgency::DeductStock`.
  - Grava `vector_application` pelo `Pharmacy::Ledger.record!`.
- **Nunca recusa a visita** (invariante do ADR 0054):
  - sem local → `vector_stock_shortfalls` `no_location`;
  - sem lote com saldo → `insufficient_balance` (há lote, falta saldo) ou `no_lot`;
  - em qualquer caso, publica `vector_stock.shortfall_recorded`.
- **Conciliação:** a coordenação ou o `stock_keeper` do local, com nota de 10–500 caracteres, sem movimento (como o 32b). Repetir → `409 already_reconciled`.

## 5. Cadência, ciclos e imóveis vencidos

- **Cadência do imóvel** (`VectorControl::Cadence.days_for(profile:, household:, neighborhood_id:)`), da mais específica para a mais geral:
  1. PE (`property_type` 4): (bairro, PE) → (cidade, PE) → `strategic_point_days` dos parâmetros (padrão 15);
  2. demais: (bairro, risco) → (bairro, qualquer) → (cidade, risco) → `default_days` (padrão 60).
- **Vencido:** imóvel ativo com `last_worked_on` nulo, ou com `last_worked_on + days ≤ hoje`.
- **Motivos na lista**, nesta ordem, com um motivo por imóvel:
  - `transmission_block`: no bloqueio ativo;
  - `liraa`: com item `pending`;
  - `pending`: a última situação foi `closed` ou `refused`;
  - `due`.
- **Ciclo:** período da coordenação. Por ciclo e por bairro:
  - a visitar: os imóveis ativos com microárea;
  - visitados: com inspeção válida no período;
  - trabalhados, fechados, recusados, desocupados, recuperados;
  - pendência % = (imóveis cuja última inspeção do ciclo é `closed` ou `refused`) ÷ visitados × 100, com uma casa;
  - cobertura: imóveis cadastrados por bairro, ao lado (risco da decisão 7).
- **Folha do agente de endemias:** "Imóveis a visitar", em `GET /vector_control/due`, nas microáreas que ele lê pelo 24b, paginada. Cada item:
  - `household_id`, `address_short` (o do 24a), `property_type`;
  - `block` {`id`, `number`} | `null`, `risk_class`;
  - `last_worked_on`, `due_since`, `reason`;
  - `liraa_draw_item_id` | `transmission_block_id`.
  - Nunca traz morador. O "quem visitar" do 24b não muda.

## 6. Ovitrampas

- **Instalação:** pelo agente de endemias, no app ou no dashboard.
  - O imóvel precisa ter quarteirão no perfil (senão `household_without_block`).
  - Exige `consent: true` (o TCLE do morador; senão `consent_required`) e `number` livre entre as ativas (senão `ovitrap_number_taken`).
  - Abre a primeira leitura com `installed_on`.
- **Coleta da palheta:**
  - Fecha a leitura aberta com `collected_on` e `condition`.
  - `collected_on − installed_on` fica entre `ovitrap_collection_days_min` e `_max` (padrão 3 e 10; o alvo de 5 dias é orientação na tela). Senão → `collection_out_of_range`.
  - Sem leitura aberta → `reading_not_open`.
  - Com `reinstall: true` (padrão), abre a leitura seguinte com `installed_on = collected_on`.
  - `remove: true` muda o status para `removed`; `condition: lost` com `remove: true` muda para `lost`.
- **Contagem:**
  - Feita pela coordenação ou pelo `vector_lab`, só em leitura coletada com `condition = ok`. Outra condição → `422 reading_not_countable`.
  - `egg_count` vai de 0 a 9999, e 0 vale (palheta negativa).
  - Corrigir exige `count_note`, e o evento leva `corrected: true`.
- **Índices** (`VectorControl::OvitrapIndex.for(neighborhood_ids:, epi_weeks:)`), por bairro e semana epidemiológica (domingo a sábado, a do MS):
  - examinadas = leituras `ok` e contadas;
  - positivas = `egg_count` > 0;
  - IPO = positivas ÷ examinadas × 100;
  - IDO = ovos ÷ positivas;
  - IDV = ovos ÷ examinadas;
  - uma casa decimal; sem examinadas → `null`.
- **Alerta:**
  - Dispara quando a contagem gravada é ≥ `ovitrap_alert_eggs` (padrão 100; a conferir com a cidade piloto, §19).
  - Abre `vector_tasks` `ovitrap_intensification`, com `suggested_block_ids = Rings.around([quarteirão da ovitrampa], rings: ovitrap_radius_rings)` (padrão 1; 200 m da NT 3/2025).
  - Uma tarefa aberta por ovitrampa: a contagem nova soma à tarefa aberta e não abre outra.
- **Mapa:** pontos das ovitrampas pela `location` do domicílio (24c); sem ponto, só a lista. A camada leva número, contagem e semana, nunca morador.

## 7. LIRAa

- O levantamento nasce `draft`. A coordenação monta os estratos com quarteirões, cada quarteirão em um estrato só (senão `stratum_overlap`).
- **Sorteio** (`POST .../draw`, step-up, só de `draft`; senão `survey_not_draft`):
  - Por estrato, N = imóveis ativos dos quarteirões.
  - Faixas por `mode`:
    - `standard`: 8.100 ≤ N ≤ 12.000;
    - `half`: 2.000 ≤ N < 8.100;
    - `census`: N < 2.000.
  - Fora da faixa → `422 stratum_out_of_range`, com `path` `strata[i]`. Os limites ficam nos parâmetros.
  - **Amostra:**
    - `standard`: n = ⌈450 ÷ (1 + 450 ÷ N)⌉;
    - `half`: n = ⌈0,5 × N⌉ × 0,2, ou seja, metade dos quarteirões (intervalo 2) e 20% dos imóveis de cada. A leitura de "50%" no manual de 2013 é **conferida na Task 1 do api**, e a divergência volta como Q;
    - `census`: todos os imóveis de todos os quarteirões.
  - **Quarteirões:**
    - Ordenados por (nome do bairro, `number` natural).
    - B = nº de quarteirões; h = ⌈0,2 × N ÷ B⌉ (imóveis por quarteirão); b = ⌈n ÷ h⌉; k = max(1, ⌊B ÷ b⌋).
    - Início `r` aleatório em 1..k (`SecureRandom`, gravado).
    - Sorteados: r, r+k, r+2k… até b quarteirões ou o fim da lista. Em `half`, k = 2; em `census`, todos.
  - **Imóveis:**
    - No quarteirão sorteado, os imóveis ativos são ordenados por (`registered_at`, `id`).
    - Inspeciona-se ⌈0,2 × imóveis⌉ (em `census`, todos), por intervalo 5 a partir de `household_start` aleatório em 1..5 (gravado). Ao chegar ao fim da lista, volta ao começo.
- **Substituição:**
  - Item com inspeção `closed` ou `refused` vira `substituted`.
  - O substituto é o próximo imóvel da ordem do quarteirão ainda não sorteado, e herda `substitute_of_id`.
  - Se não houver, o quarteirão fica com um imóvel a menos, e o relatório mostra.
- Os itens chegam à folha (§5) e ao app (§13).
- **Fechamento** (`POST .../close`, step-up, de `drawn`): congela os índices.
- **Índices** por estrato e município, das inspeções `worked` ligadas aos itens:
  - IIP = imóveis positivos ÷ inspecionados × 100;
  - IB = recipientes positivos ÷ inspecionados × 100;
  - ITR(g) = positivos do grupo g ÷ positivos × 100.
  - "Positivo" é a amostra com `aegypti_larvae` > 0 (um tubito por depósito). O imóvel é positivo quando tem alguma amostra positiva.
  - Enquanto houver amostra sem resultado, o índice é **provisório**, com `provisional: true`.
  - Classe do IIP: < 1 "Satisfatório", 1 a 3,9 "Alerta", ≥ 4 "Risco"; parametrizada (`iip_bands`).
- **Relatório:** PDF (A4, `Consultations::Print.document`) e CSV (UTF-8 com BOM, `;`, `\r\n`, como o `Surveillance::Csv`).

## 8. Bloqueio de transmissão

- **Gatilho** (decisão 28): `VectorControl::ProposeBlockJob` (`IdempotentConsumer`), ligado a `disease_notification.created` e a `disease_notification.transitioned`.
  - Age quando a notificação está `notified`, a doença está em `block_diseases` (padrão `dengue`, `chikungunya` e `zika`) e o gate está utilizável.
  - A notificação `possible` da rede de segurança só age quando passa a `notified`.
- **Caso de fora** (`resident_elsewhere`): só uma tarefa `case_resident_elsewhere`, sem bloqueio.
- **Quarteirão do caso** (decisão 27): o cadastro individual ativo do paciente → o domicílio → `household_vector_profiles.city_block_id`. Faltando qualquer elo, o caso entra sem lugar, e nasce a tarefa `block_case_without_place`.
- **Agrupamento:**
  - O caso com quarteirão entra no bloqueio `proposed` ou `active` cuja área já tem o quarteirão do caso. Se o bloqueio ainda é proposta sem área, conta `Rings.around([centro], rings: block_radius_rings)`.
  - Vale também a janela: o caso mais recente do bloqueio tem `case_on` ≥ `case_on` do novo − `block_grouping_days` (padrão 7).
  - Se não houver bloqueio assim, nasce a proposta, com `center_block_id` e área sugerida.
  - O caso sem lugar sempre nasce proposta própria. A coordenação junta pela rota de mover caso.
- **Confirmação** (`POST .../confirm`, step-up): só de `proposed`, com `center_block_id` e área (lista de quarteirões, ≥ 1; a sugestão vem marcada). Vira `active`.
- **Descarte:** de `proposed` para `dismissed`, com motivo.
- **Fechamento:** de `active` para `closed`.
- **Ciclos de UBV:** até `ubv_cycles_max` (padrão 3).
  - Cada ciclo tem de `ubv_cycle_days_min` a `_max` dias (padrão 3 a 5), e `starts_on` fica ≥ `ends_on` do anterior + `ubv_cycle_interval_days` (padrão 5).
  - Violação → `422 cycle_invalid`, com `path`.
- **Endereço do caso:** só a coordenação lê, para escolher o quarteirão de caso sem lugar.
  - Rota: `GET /vector_control/transmission_blocks/:id/cases/:case_id/address`. Devolve o endereço do paciente e o do domicílio, se houver.
  - Grava a trilha com `access: "transmission_block_case"`, pelo mesmo mecanismo do 24a; nunca há polling.
  - Nenhuma outra resposta do 35a traz paciente: o caso aparece como doença, data e "caso 1, 2…".
- **Mapa:** os pontos dos imóveis da área (24c), sem o caso.
- **Indicador:** tempo da notificação (`created_at` da notificação) até a confirmação, em horas, no relatório.

## 9. Amostras

- A etiqueta é calculada igual no app e no api, sem rede: `label = upcase(client_uuid_da_visita[0,8]) + "-" + format("%02d", number)`.
  - A visita do dashboard também tem `client_uuid` (24b).
  - Colisão de `label` (índice único) → a visita é recusada com `client_uuid_conflict`, e o app gera outro uuid.
- **Fila de resultado:** amostras sem resultado, por data da visita, para a coordenação e o `vector_lab`.
- **Resultado:**
  - `examined_larvae` ≤ `larvae_collected`; `aegypti_larvae + albopictus_larvae` ≤ `examined_larvae`. Senão → `sample_result_invalid`.
  - Corrigir exige `result_note`.
  - O resultado volta ao imóvel (detalhe da visita) e aos índices.

## 10. Relatórios

`GET /vector_control/reports/<nome>` em JSON, e com `.csv` no fim do caminho. Leem a coordenação, o `vector_lab` (só ovitrampas e amostras) e o `municipal_admin`. São contagens, sem trilha.

| Nome | Filtros | Linhas |
|---|---|---|
| `situations` | `cycle_id`, `neighborhood_ids[]` | por classe de imóvel (R = 1, 7–11, 99; C = 2; TB = 3; PE = 4; O = 5, 6, 12, 13): a visitar, visitados, trabalhados, fechados, recusados, desocupados, recuperados, pendência %; cobertura por bairro |
| `strategic_points` | `cycle_id` | PE por situação e por última visita (quinzena) |
| `production` | `from`, `to`, `professional_ids[]` | por agente e por dia: imóveis, trabalhados, depósitos eliminados e tratados, amostras, cargas por produto |
| `consumption` | `from`, `to` | por produto: cargas, quantidade, baixas, falta-estoque por causa |
| `ovitraps` | `epi_weeks[]`, `neighborhood_ids[]` | IPO, IDO e IDV por bairro e semana; perdidas e avariadas |
| `liraa` | `survey_id` | índices por estrato e município, classe do IIP, amostra planejada × inspecionada, substituições; também `.pdf` |
| `blocks` | `from`, `to` | bloqueios por status, casos (com `Admin::SmallCount` para 1–4), horas até a confirmação, ciclos |

## 11. Parâmetros técnicos

`vector_parameter_versions.values`. A coordenação edita com step-up, e cada edição cria uma versão nova (decisão 24). O padrão de fábrica fica em `config/vector_control/defaults.yml`:

```yaml
default_days: 60
strategic_point_days: 15
block_radius_rings: 1
block_grouping_days: 7
block_diseases: [dengue, chikungunya, zika]
ubv_cycles_max: 3
ubv_cycle_days_min: 3
ubv_cycle_days_max: 5
ubv_cycle_interval_days: 5
ovitrap_collection_days_min: 3
ovitrap_collection_days_max: 10
ovitrap_alert_eggs: 100
ovitrap_radius_rings: 1
liraa_standard_min: 8100
liraa_standard_max: 12000
liraa_half_min: 2000
liraa_base_sample: 450
liraa_household_fraction: 0.2
iip_bands: [1.0, 4.0]
```

- O mesmo spec confere as faixas de cada chave e o PUT (`422 invalid_parameter`, com `path`).
- O levantamento do LIRAa grava a `parameters_version` do sorteio. O bloqueio e a ovitrampa usam a versão vigente.

## 12. Rotas do api

Prefixo `/vector_control`, sessão do dashboard. As escritas por cookie são JSON e passam por `require_vector_control!`. Onde está marcado "step-up", vale o `401 mfa_required`.

| Rota | Quem | Nota |
|---|---|---|
| `GET /blocks?neighborhood_id=` · `POST /blocks` · `PATCH /blocks/:id` · `POST /blocks/:id/neighbors` · `DELETE /blocks/:id/neighbors/:neighbor_id` | coordenação (leitura: também `vector_lab` e agente) | o DELETE grava `ended_at` |
| `GET /households?neighborhood_id=&city_block_id=&without_block=` · `POST /households/assign_block` · `PATCH /households/:id/profile` | coordenação | lista com `address_short`, tipo, quarteirão, risco, última visita; trilha `territory_registry` |
| `GET /due?micro_area_id=` | agente de endemias | §5 |
| `GET /cadences` · `PUT /cadences` · `GET /cycles` · `POST /cycles` · `PATCH /cycles/:id` | coordenação | o PUT troca a lista inteira, com step-up |
| `GET /parameters` · `PUT /parameters` | coordenação; o GET também para o agente | step-up |
| `GET /products` · `POST /products` · `PATCH /products/:id` · `GET /settings` · `PUT /settings` | coordenação | |
| `GET /ovitraps` · `POST /ovitraps` · `POST /ovitraps/:id/collect` · `GET /ovitrap_readings?state=to_count` · `POST /ovitrap_readings/:id/count` | instalação e coleta: agente; contagem: coordenação e `vector_lab` | |
| `GET /tasks` · `POST /tasks/:id/close` | coordenação | `state` `done` ou `dismissed`, com nota |
| `GET/POST /liraa_surveys` · `PATCH /liraa_surveys/:id` · `POST /liraa_surveys/:id/draw` · `POST /liraa_surveys/:id/close` · `GET /liraa_surveys/:id/items` | coordenação | draw e close com step-up |
| `GET /transmission_blocks` · `GET /transmission_blocks/:id` · `POST /transmission_blocks/:id/confirm` · `POST .../dismiss` · `POST .../close` · `POST .../cycles` · `POST .../cases/:case_id/move` · `GET .../cases/:case_id/address` | coordenação | confirm e address com step-up |
| `GET /samples?state=pending` · `POST /samples/:id/result` | coordenação e `vector_lab` | |
| `GET /stock_shortfalls` · `POST /stock_shortfalls/:id/reconcile` | coordenação e `stock_keeper` do local | |
| `GET /reports/:name` (+ `.csv`, e `.pdf` no `liraa`) | §10 | |

- A visita (`GET /home_visits/:id`, folha e lista) ganha `vector_inspection`:
  - os campos do corpo do §3;
  - os derivados (`recovered`, e `samples[].label` e `samples[].result`);
  - os produtos com nome e código.
- A visita só tem esse campo quando há inspeção, e ele vale para quem já lê a visita pelo 24b.

## 13. `field-sync` v1.3.0 (MINOR)

| Arquivo | Muda |
|---|---|
| `session-response-v1.json` | `features` + `vector_control` (bool, **opcional**; ausente = false) |
| `upload-request-v1.json` | `home_visit_body` + `vector_inspection` (§3, `$defs/vector_inspection` fechado); tipos novos de item `ovitrap_installation` e `ovitrap_collection` |
| `upload-response-v1.json` | `accepted` do `home_visit` pode trazer `vector_inspection_id`; do `ovitrap_installation`, `ovitrap_id` |
| `changes-response-v1.json` | `<household>` + `vector` opcional: { `city_block_id`, `risk_class`, `risk_assessed_on`, `last_worked_on`, `last_situation` } \| `null`. `change_item.type` + `city_block`, `ovitrap`, `liraa_item`, `transmission_block` |
| `reference-response-v1.json` | + `vector_control_domains`, opcional (só com o gate utilizável) |
| `domains-v1.json` | `vector_control` (domínios e limites do §3 e §6) e os códigos novos em `errors.item` |
| `examples/` | válidos e inválidos de cada novidade; e os dois achados do contracts#2: `street_type` com o código (`"081"`), e o vocabulário de `rules` da atividade na forma que o api serve (Q3) |

**Itens novos do envio:**

```json
{ "client_uuid", "type": "ovitrap_installation",
  "body": { "household_id" | "household_client_uuid", "number", "consent": true, "installed_on": "YYYY-MM-DD" } }
{ "client_uuid", "type": "ovitrap_collection",
  "body": { "ovitrap_id" | "ovitrap_client_uuid", "collected_on": "YYYY-MM-DD",
            "condition": "ok" | "damaged" | "lost" | "dry", "reinstall": true, "remove": false } }
```

- Os dois são só do agente de endemias com o gate utilizável. Outro CBO → `rejected cbo_not_allowed`.
- `ovitrap_client_uuid` resolve pelo livro de idempotência, como as outras citações do 24c. A falta da citação → `rejected not_found`, com `path` `ovitrap_client_uuid`.

**Entidades novas do download.** Cada uma tem `id` e `version` e sai pelo mesmo cursor (o plano do api escolhe a mecânica; a regra é que toda escrita de `version` reemita a entidade):

| Tipo | Corpo | Quando vai |
|---|---|---|
| `city_block` | `id`, `neighborhood_id`, `number`, `active`, `neighbor_ids[]`, `version` | quarteirões dos bairros dos imóveis do escopo |
| `ovitrap` | `id`, `number`, `household_id`, `status`, `open_reading` { `installed_on` } \| `null`, `last_reading` { `collected_on`, `condition`, `egg_count` } \| `null`, `version` | ovitrampas de imóveis do escopo |
| `liraa_item` | `id`, `survey_id`, `survey_name`, `household_id`, `city_block_id`, `status`, `version` | itens de levantamentos `drawn` com imóvel do escopo; quando o levantamento fecha, vai com `status: "closed"` |
| `transmission_block` | `id`, `status`, `block_ids[]`, `cycles[]` { `number`, `starts_on`, `ends_on` }, `version` | bloqueios `active` com quarteirão de imóvel do escopo; ao fechar, vai uma vez como `closed`. **Nunca leva caso nem notificação** |

- Mudança de perfil do imóvel reemite o `<household>`.
- O `removal` não ganha tipo novo: o fim vem pelo status.

**`vector_control_domains`** (referência):
- `activities`, `work_types`, `situations`, `deposit_groups`, `measure_methods` e `risk_classes`, cada um com `{ code, label }`;
- `products` com `{ id, code, name, unit, charge_size, uses }`, só os ativos;
- `cadences`, as linhas da tabela;
- `parameters`: `default_days`, `strategic_point_days`, `ovitrap_collection_days_min` e `_max`;
- `limits`: depósitos 999, cargas 999, amostras 99 e larvas 10.

**Compatibilidade:**
- O app da v1.2.0 ignora o que não conhece. Esse é o contrato do README; o plano do campo confere que o `applyPage` ignora tipo desconhecido em vez de falhar.
- O api aceita o envio da v1.2.0 sem mudança.
- Nada fica obrigatório.

## 14. App de campo (`rotasaude/campo`)

- **Esquema local v2:**
  - `SCHEMA_VERSION = 2`;
  - `migrate()` sobe de 1 para 2 sem apagar nada: cria `city_blocks`, `ovitraps`, `liraa_items` e `transmission_blocks`, e os índices;
  - o `vector` do domicílio fica no `body`;
  - a versão desconhecida continua indo para `UnknownSchemaError`;
  - um teste parte de um banco v1 com dado e sobe.
- **Boletim:**
  - Aparece na `VisitForm` só com `session.features.vector_control` e o CBO do vínculo `515140`.
  - Ordem: atividade, tipo de trabalho, situação, horários (entrada ao abrir a visita, saída ao salvar, editáveis), depósitos por grupo (contadores), medidas (produto e cargas), amostras (etiqueta mostrada na hora, §9), ação educativa, quarteirão (só se faltar) e risco.
  - As regras do §3 são portadas para `src/rules/vectorInspection.ts`, com fábrica como as do 24c (Q76 do 24c). As frases de recusa são as do dashboard.
- **Folha:** "Imóveis a visitar" calculado no aparelho (§5, com a cadência e os parâmetros da referência), com os itens do LIRAa e do bloqueio por cima.
- **Ovitrampas:**
  - instalar a partir do domicílio;
  - lista com "Coletar palheta" (condição, reinstalar ou retirar);
  - nada de contagem no app.
- **Offline:** tudo entra na fila do 24c. A ordem de envio respeita as citações (ovitrampa nova antes da coleta).
- **Testes:** Jest das regras, da migração 1→2 e do envio; fluxos do Maestro para o boletim offline e para instalar e coletar a ovitrampa.

## 15. Dashboard

- **Grupo novo "Endemias"** (`NAV_GROUPS`, `CATALOG`, `renderModule`), para `endemic_coordinator` e, onde indicado, para `vector_lab`. Telas:
  - `vector-blocks`: quarteirões, vizinhos, imóveis sem quarteirão e atribuição em lote;
  - `vector-cycles`: ciclos e cadência;
  - `vector-ovitraps`: lista, mapa e contagem (também `vector_lab`);
  - `vector-liraa`: estratos, sorteio, itens e índices;
  - `vector-transmission`: propostas, área, ciclos e casos sem paciente;
  - `vector-samples`: fila de resultado (também `vector_lab`);
  - `vector-tasks`: tarefas;
  - `vector-settings`: produtos, parâmetros e local de estoque;
  - `vector-reports`: relatórios, com download de CSV e PDF pelo `binaryFetch` e o `saveFile`.
- **Agente de endemias:**
  - a `VisitForm` ganha a seção "Boletim" (§3, mesmas regras);
  - o `HomeVisitDay` ganha "Imóveis a visitar" (§5);
  - o `VisitDetail` mostra o boletim, as etiquetas e os resultados;
  - "Ovitrampas" para instalar e coletar.
- **Unidades (09):** tipo "Vigilância ambiental e zoonoses".
- **Estoque (27a):** o domínio `vector_control` nas telas de local, entrada, transferência e ajuste, com o produto no lugar do CATMAT; e "Falta de estoque" para conciliar.
- **Mapa:** o `createFieldMap` do 24c, com pontos coloridos por risco (`vector-blocks`), por contagem (`vector-ovitraps`) ou por área (`vector-transmission`). Sem coordenada, só a lista.
- `FeatureKey` e `FEATURE_LABEL` ganham `vector_control`.

## 16. Maintenance

- `FEATURE_LABELS.vector_control` = "Vigilância ambiental (endemias)".
- `MISSING_LABELS.home_visits_disabled` = "precisa da visita domiciliar ligada".
- Testes nos dois arquivos e o parágrafo do README, como no 24b (commit 55b680b).
- Nenhuma mutation nova.

## 17. LGPD, erros e eventos

- **Sigilo:**
  - Nenhuma resposta do 35a, salvo a do endereço do caso (§8) e as rotas do 24b/24a que já existem, traz nome, CPF, CNS, telefone ou endereço de pessoa.
  - `address_short` é o do domicílio, como no 24a.
  - A coordenada nunca sai ligada a caso.
  - Microchip não é do 35a.
- **Eventos novos** (`DomainEvents.publish`, só ids; ligados em `config/initializers/domain_events.rb` e fixados num spec de sigilo novo, `spec/security/vector_control_secrecy_spec.rb`, no molde do `home_visit_secrecy_spec`):

| Evento | Payload |
|---|---|
| `city_block.changed` | `city_block_id`, `user_id` |
| `household_vector_profile.changed` | `household_id`, `user_id` |
| `vector_stock.shortfall_recorded` | `vector_stock_shortfall_id` |
| `vector_stock.shortfall_reconciled` | `vector_stock_shortfall_id`, `user_id` |
| `ovitrap.installed` / `ovitrap.collected` | `ovitrap_id`, `user_id` |
| `ovitrap_reading.counted` | `ovitrap_reading_id`, `user_id`, `corrected` |
| `vector_task.opened` / `vector_task.closed` | `vector_task_id` (+ `user_id` no closed) |
| `liraa_survey.drawn` / `liraa_survey.closed` | `liraa_survey_id`, `user_id` |
| `transmission_block.proposed` | `transmission_block_id` |
| `transmission_block.confirmed` / `.dismissed` / `.closed` | `transmission_block_id`, `user_id` |
| `transmission_block_case.address_viewed` | `transmission_block_id`, `user_id` |
| `vector_sample.result_recorded` | `vector_sample_id`, `user_id`, `corrected` |
| `vector_parameters.changed` | `version`, `user_id` |
| `vector_product.changed` | `vector_product_id`, `user_id` |

- A inspeção não tem evento próprio: vai no `home_visit.recorded`.
- Nenhum nome novo de `Platform.audit`.
- **Códigos novos** (422 salvo indicado; os de item também em `domains-v1.json`):
  - `vector_inspection_invalid`, `situation_outcome_mismatch`, `inspection_not_worked`, `activity_property_mismatch`, `liraa_item_not_found`, `transmission_block_not_active`, `deposit_counts_inconsistent`, `product_not_allowed`, `samples_exceed_larvae`, `inspection_times_invalid`, `reason_derived`, `block_already_set`;
  - `household_without_neighborhood`, `block_not_in_household_neighborhood`, `household_without_block`, `consent_required`, `ovitrap_number_taken`, `collection_out_of_range`, `reading_not_open`, `reading_not_countable`, `ovitrap_not_active`;
  - `invalid_neighbor`, `block_in_use` (409), `product_in_use` (409), `already_reconciled` (409);
  - `stratum_overlap`, `stratum_out_of_range`, `survey_not_draft` (409), `cycle_invalid`, `cycle_overlap`, `invalid_parameter`, `sample_result_invalid`, `block_number_taken`.

## 18. Semente de dev e roteiro de prova

- `lib/vector_control_crew.rb`, chamado pelo `db:seed`, só em Curitiba, que tem `home_visits`. Liga `vector_control` em dev. É fictício e imita o real (memória "semente de dev realista").
- **Pessoas:**
  - `ace@curitiba.demo`: agente de endemias, CBO 515140, numa eSF, com duas microáreas atribuídas;
  - `endemias@curitiba.demo`: `endemic_coordinator`;
  - `entomologia@curitiba.demo`: `vector_lab`;
  - a senha e o TOTP são os das outras contas da semente.
- **Unidade e estoque:** unidade "UVZ Curitiba" (`environmental_surveillance`) com local `vector_control`, produtos (dois larvicidas e um adulticida, com cargas de catálogo) e lotes com saldo.
- **Território:**
  - quarteirões com vizinhos em dois bairros, e o perfil de todos os imóveis das duas microáreas;
  - risco variado e dois PE;
  - quatro semanas de ovitrampas contadas, com uma acima do alerta;
  - um LIRAa sorteado em `census`, com metade inspecionada;
  - um caso de dengue numa paciente da semente, com o bloqueio `active` e um ciclo.
- **Roteiro de prova** (dashboard e app; um caminho por requisito do edital):
  1. boletim completo na visita, com a ficha 8 derivada;
  2. baixa de estoque e falta conciliada;
  3. vencidos por risco;
  4. instalação, coleta e contagem de ovitrampa, com IPO no mapa;
  5. LIRAa sorteado, substituição e relatório PDF;
  6. notificação de dengue que propõe o bloqueio, a confirmação e o ciclo;
  7. amostra com resultado que muda o IIP;
  8. relatórios em CSV;
  9. boletim offline no app.

## 19. Entrega, testes e pendências

- **Ordem:**
  1. contracts (v1.3.0, tag com autorização);
  2. api;
  3. dashboard e maintenance;
  4. campo;
  5. suíte completa do api pela "Testes";
  6. prova no navegador e no emulador;
  7. merge e push com autorização, repo por repo.
- **Testes do api:**
  - specs de cada número das regras, com as bordas:
    - estrato com N = 8.099, 8.100, 12.000, 12.001, 1.999 e 2.000;
    - substituição sem vizinho;
    - caso no 7º e no 8º dia;
    - contagem 0;
    - leitura com 2 e com 11 dias;
    - ciclo de UBV com intervalo de 4 dias;
  - a ficha 8 com os motivos derivados;
  - a baixa que nunca recusa;
  - o estorno na retirada;
  - o spec de sigilo;
  - o feed do app com as entidades novas;
  - o envio da v1.2.0 sem mudança.
- **Gates de go-live** (no board #2, no fechamento):
  - diretriz vigente conferida com a vigilância ambiental de uma cidade piloto (cadência, raio em anéis, limiar da ovitrampa, faixa do IIP, modo `half`);
  - ficha 8 do agente de endemias aceita no PEC real.
- **Fora do 35a** (card no fechamento):
  - raio geométrico pela coordenada;
  - território de endemias sem microárea;
  - Conta Ovos;
  - exportação do LIRAa estadual;
  - suspensão de Wolbachia/TIE no bloqueio.

## 20. Decisões (Q-n)

| Q | Decisão |
|---|---|
| Q1 (coordenador, ao escrever; aprovada pelo usuário em 2026-11-16) | `vector_control` exige `feature:home_visits`, e não só `territory_registry`: o boletim é um bloco da visita, e sem a visita não há onde registrá-lo. `home_visits` já exige o cadastro, então a decisão 1 da spec fica coberta. |
| Q2 (coordenador, ao escrever; aprovada pelo usuário em 2026-11-16) | Quarteirão e risco ficam em `household_vector_profiles`, fora de `household_versions`. Mudá-los não cria versão do 24a, não gera ficha 3 e não depende do ACS. A spec dizia "colunas novas no 24a"; a tabela 1:1 é a forma disso. |
| Q3 (coordenador, ao escrever; aprovada pelo usuário em 2026-11-16) | Os dois achados do contracts#2 (exemplos com `street_type` `"RUA"` e o vocabulário de `rules` do `domains-v1.json`) entram na v1.3.0, que é a "próxima versão" decidida pelo usuário em 2026-11-16. Os seis itens adiados do contracts#2 continuam adiados. |
| Q4 (coordenador, ao escrever; aprovada pelo usuário em 2026-11-16) | Insumo de campo é um terceiro domínio do estoque do 27a (`item_kind` `vector_control`, produto do `vector_products`), não item do CATMAT nem do almoxarifado do laboratório (21b). Segue o caminho que o 29 abriu para o imunobiológico. |
| Q5 (coordenador, ao escrever; aprovada pelo usuário em 2026-11-16) | Sem geometria de quarteirão: os mapas do 35a usam os pontos dos imóveis do 24c, e o raio conta anéis de vizinhança. Quarteirão é lista, não polígono. |
| Q6 (planos do campo D1, do contracts CV-1/CV-2 e do api W-14; compatibilidade) | O app da v1.2.0 **não** ignora tipo desconhecido no download (`applyPage` manda tudo o que não é `micro_area`/`household` para `putRegistration`, e a página falha). O §13 "Compatibilidade" muda: (1) o api só emite as entidades novas e o `<household>.vector` para cidade com `VectorControl::Gate.usable?`; ausente = gate desligado, `null` = imóvel sem perfil; (2) o interruptor entra no `scope_digest` do `FieldSync::Scope`, e ligar ou desligar `vector_control` chega ao aparelho como `reset: true`; (3) o campo passa a ignorar tipo desconhecido (Task 1 do campo, sobre a v1.2.0), e a migração 1→2 do esquema local zera o cursor; (4) **gate de go-live:** a cidade só liga `vector_control` depois de todos os aparelhos estarem no app do 35a; em dev, o APK do 35a vai ao emulador antes do `db:seed` que liga o interruptor. O feed anda por `updated_at`, e toda escrita de `version` reemite a entidade. |
| Q7 (campo D2/D3/D4/D5, contracts CV-3/CV-5/CV-9/CV-13, api W-9; ids, códigos e limites) | (1) Colisão de etiqueta: código novo `sample_label_taken`, `path` `vector_inspection.samples[i]` (não `client_uuid_conflict`). O app troca etiquetas só por ação explícita do agente ("Gerar etiquetas novas"); no dashboard, a etiqueta é provisória até o 201, porque o 24b renova o `client_uuid` depois de um 4xx (dashboard D-12). (2) No `accepted`/`duplicate` de `ovitrap_installation` e `ovitrap_collection`, `id` é o da ovitrampa; não há `ovitrap_id`. O `home_visit` pode trazer `vector_inspection_id`. (3) `limits` do `vector_control_domains`: `deposit_count_max` 999, `charges_max` 999, `samples_max` 99, `larvae_collected_max` 10, `measures_max` 20; `deposits` tem no máximo 7 itens. (4) Chaves obrigatórias do boletim: `activity`, `work_type`, `situation`, `educational_action`; listas ausentes valem vazias. (5) `consent` é boolean; `false` dá `consent_required`. `reinstall` é true por padrão e é ignorado com `remove: true`; o app manda `reinstall: !remove`. (6) O número da ovitrampa segue `^[0-9A-Za-z-]{1,12}$`. (7) Erro de forma dentro do boletim → `vector_inspection_invalid` com o `path` do campo; as relações do §3 ficam fora do esquema. |
| Q8 (dashboard D-1..D-20; rotas e formas que faltavam no §12) | Valem os tipos e corpos da Task 1 do plano do dashboard, com decimais como texto (27a). Rotas novas do api: `GET /vector_control/domains` (exatamente o `vector_control_domains` do app, mesmo serializador); `GET /vector_control/households/:id` (o perfil na forma do `<household>.vector`, para o agente nas microáreas dele e para a coordenação; o `TerritoryRegistry::Json.household` do 24a **não** muda, api W-10); `GET /vector_control/map_points?layer=risk\|ovitraps\|transmission&…` → `{ map, points: [{ id, latitude, longitude, category, label }] }`, sem pessoa, e a camada do bloqueio sem o caso; `GET /vector_control/cycles/:id` com o resumo do §5; `CityBlock.households_count`; `GET /ovitraps?micro_area_id=` (agente) ou `?neighborhood_id=` (coordenação e laboratório) e `status`; `POST /pharmacy/items/search` aceita `domain: "vector_control"` e devolve o produto como item com `vector_product_id`; `GET /stock_shortfalls?state=&location_id=` (coordenação lê todas, inclusive `no_location`; o `stock_keeper`, as do seu local); `cases[].number` estável depois de mover o caso. Relatório em JSON: `{ name, provisional?, columns, rows, notes? }`, com `SmallCount`. O `/due` ordena pelo motivo do §5 e depois por `due_since`, com os nunca trabalhados (`due_since = null`) primeiro. O agente pode salvar visita sem boletim (chave "Registrar boletim", ligada por padrão). As atividades `liraa` e `transmission_block` só são oferecidas a partir de "Imóveis a visitar". O tipo de unidade `environmental_surveillance` só é oferecido com o gate utilizável e não sai dos seletores. O agente é reconhecido pelo CBO da equipe em `GET /home_visits/context`; a sessão não traz CBO. |
| Q9 (códigos de transição e de nota; dashboard D-10, api A-14) | Transição fora do estado (confirmar o que não é proposta, fechar levantamento que não está sorteado, fechar tarefa já fechada, ciclo em bloqueio que não está ativo): `409 invalid_transition` (o código do 25), `path` `status`. O dashboard troca o seu `status_changed` por `invalid_transition` e mantém a frase. `survey_not_draft` fica para o sorteio e para a edição. Nota obrigatória ausente: `422 invalid_request`, `path` `count_note`, `result_note` ou `note`. Os `path` dos códigos do §3 e do §6 são os da Task 2 do plano do dashboard (fonte única); o api e os exemplos do contracts se alinham a eles, e a divergência vira Q. A tabela de frases do dashboard ganha `sample_label_taken` ("esta etiqueta já foi usada — gere etiquetas novas para este boletim") e `invalid_transition`. |
| Q10 (plano do api A-1..A-15 e W-1..W-13) | Aceitos todos. Em particular: estorno da baixa por `vector_application_reversal` (positivo), como o `immunization_reversal` do 29; `quantity = (charges × charge_size).ceil(2)`, com mínimo 0,01; os motivos derivados entram **antes** da conversão Q43 do 24b, e o erro do 24b que cair num motivo derivado volta com `path` `subject`; sem versão gravada, valem os padrões do `defaults.yml` (versão 0); as faixas do PUT de parâmetros são as da tabela do plano do api, que o §11 passa a citar; vizinho com único parcial `WHERE ended_at IS NULL`; `liraa_stratum_blocks.liraa_survey_id` (conferido por trigger) com único `(liraa_survey_id, city_block_id)`; `vector_tasks.ovitrap_id` com único parcial nas abertas; a semana epidemiológica pelo `Surveillance::EpiWeek`, gravada como inteiro `YYYYWW` e exposta como `"YYYY-WW"` nas rotas, nos filtros, no JSON e no app (dashboard D-14); boletim sem domicílio → `vector_inspection_invalid`, `path` `household_id` (campo D13, contracts CV-6); a cadência ganha o degrau (cidade, qualquer risco) antes de `default_days` (campo D11); a `Pharmacy::SaveLocation` aceita `vector_control` só em unidade `environmental_surveillance`; unidade com esse local não troca de tipo (`409 unit_has_vector_control_location`); a leitura do endereço do caso grava a trilha **e** o evento `transmission_block_case.address_viewed`; o CHECK do livro de idempotência ganha os dois itens novos; o estorno na retirada é síncrono, dentro do `HomeVisits::Withdraw`; a duração do ciclo de UBV conta os dois dias; o `endemic_coordinator` segue a regra dos papéis privilegiados. O api aplica os itens de um lote em ordem, cada um vendo o que o anterior gravou (campo D12). O modo `half` do LIRAa espera a prova técnica da Task 1 do api (A-15). |
| Q11 (plano do campo D6–D10, D14, D15; plano do contracts CV-4, CV-7, CV-8, CV-10..CV-14) | Aceitos. A hora de entrada padrão é o minuto do `occurred_at`, e "dia" é o dia da cidade. A folha do app porta a regra do api (`vectorDue.ts`), com os casos do spec dele. `local_visits.summary` ganha `vector { activity, situation, city_block_id?, risk_class? }`, sem o que foi dito na casa (estende a Q79(3) do 24c). Sem remoção nova, o app apaga o item do LIRAa que não está pendente, o bloqueio fechado e a ovitrampa inativa, em cascata quando o domicílio sai; quarteirão fora do escopo só sai no `reset`. Um motivo e um vínculo por imóvel na folha: o bloqueio vence o LIRAa. Valem os nomes, `path` e frases do dashboard; o que faltar no dashboard vai ao coordenador. "Corrigir número" só para `ovitrap_installation` com `ovitrap_number_taken`, reenviando o mesmo `client_uuid` (estende a Q81(1) do 24c). Os rótulos de todos os domínios ficam no `domains-v1.json`. O esquema de `street_type` **não** é restringido ao código, para não apertar o pedido numa MINOR. `participant_measurements` perde o `ledi` para ficar igual ao api. O `proof` fica `fcf23c87`, e o `domains_spec` do 35a no api confere a seção `vector_control`. |
| Q12 (maintenance MV-1, MV-2) | A frase de `home_visits_disabled` é "visita domiciliar e territorial (home_visits) desligada ou incompleta", como a Q29 do 24b, porque o api manda o mesmo código quando a visita está ligada e o cadastro não. A descrição do `vector_control` no `CATALOG` passa a ser "Boletim do agente na visita, quarteirões, ciclos, ovitrampas, LIRAa e bloqueio de transmissão", sem repetir o nome (como a Q24 do 24b). Papel e tipo de unidade não pedem nada no maintenance. |
| Q13 (segunda rodada dos planos: dashboard, campo N1, api N-1..N-10, contracts) | (1) **Imóvel não residencial pelo agente de endemias** (conflito do dashboard; decisão 3 da spec: "cadastrado no fluxo do 24a … na microárea de quem cadastra"; o `515140` está na Tabela 2 do LEDI): com `vector_control` utilizável, o agente de endemias cadastra e edita domicílio do 24a **só** de tipo não residencial (`2 3 4 5 6 12 13`), sem família nem morador, nas microáreas atribuídas a ele, pelo `TerritoryRegistry::SaveHousehold` (dashboard e item `household` do app). Tipo residencial, família, cadastro individual e troca para tipo residencial → `403 cbo_not_allowed` (no app, `rejected cbo_not_allowed`), como hoje. A ficha 3 sai com o CBO dele. O ACS não muda. Isso estende a Q11 do 24b só para esse caso. (2) `path` das recusas da ovitrampa: `household_without_block` → `household_id`; `consent_required` → `consent`; `ovitrap_number_taken` → `number`; `reading_not_open` e `ovitrap_not_active` → `ovitrap_id`; `collection_out_of_range` → `collected_on`. (3) `sample_label_taken` é 422, e a frase fica uma só para o app e para o dashboard. (4) Carga inicial do app: `cursor === null && last_download_ok_at === null`. A migração 1→2 mantém `last_download_ok_at`, e o tablet migrado não espera Wi-Fi para baixar de novo (N1 do campo). (5) `GET /vector_control/households/:id` → `{ id, neighborhood_id, vector }`, com `vector` na forma do `<household>.vector`. (6) `transmission_block_cases.number`: sequência da cidade, gravada e imutável; vai em `cases[].number` (entra no §1). (7) As categorias das camadas do `map_points` são as do plano do api, e o adaptador de cor do dashboard as segue. Ler os pontos grava a trilha `territory_map` do 24c. (8) O `created_at` da versão 0 dos parâmetros é `null`, e o tipo do dashboard é `string \| null`. (9) Mover caso para bloqueio fechado → `409 invalid_transition`. (10) O api devolve só o primeiro erro, como no 24b; a lista do dashboard começa pelo mesmo erro. |

| Q14 (terceira rodada: api e dashboard D-21) | (1) `GET /territory/teams` (24a; é o que o `getTerritoryContext` do dashboard lê) lista as microáreas atribuídas ao agente de endemias com `can_register: true` e `property_types: ["2","3","4","5","6","12","13"]` quando o `vector_control` está utilizável. `property_types` ausente ou `null` = todos os tipos, e o ACS não muda. O dashboard usa esse campo, não o CBO do contexto da visita. (2) Microárea não atribuída ao agente: `403 out_of_scope` no dashboard e `rejected not_found` com `path` `household_id` no app, como a visita do 24b trata os papéis de campo. |
| Q15 (contracts, revisão final da branch antes da Task 6) | `vector_control_domains.products[].charge_size` é **texto** no `reference-response-v1.json`: `{ "type": "string", "pattern": "^[0-9]{1,8}\\.[0-9]{2}$" }` (numeric 10,2), como os decimais da Q8 e do 27a, porque o `GET /vector_control/domains` usa o mesmo serializador. O "> 0" fica com o api. Exemplos `"1.00"` e `"12.50"`. A entrada `measures[].charges` do envio continua **número** (2 casas, > 0, ≤ 999), como nos planos do api, do dashboard (`measureQuantity(charges: number, chargeSize: string)`) e do campo. |
| Q16 (api Task 1, prova técnica; Q-T1b..f) | Vale a fonte oficial (decisão 2 do usuário: só o que os editais repetem). (b) Dentro do quarteirão o LIRAa começa **sempre no 1º imóvel**, no sentido horário (manual LIRAa 2013 p.28): `household_start = 1`, sem sorteio; o sorteio fica só no início casual dos quarteirões. A ordem dos imóveis segue (`registered_at`, `id`) até haver ordem de percurso. (c) A substituição tenta **primeiro o imóvel imediatamente anterior** na ordem, depois o posterior, alternando para fora, só entre os não sorteados (p.30). (d) **Ponto estratégico (tipo `4`) fica fora da base do sorteio** e do N do estrato (p.28; DN-2009 p.74). (e) O IIP é **arredondado a uma casa** (meio para cima) antes de classificar e é exibido com uma casa; faixas < 1,0 / 1,0–3,9 / ≥ 4,0, iguais ao Quadro 5 da DN-2009. (f) O rótulo de `B` passa a **"B — Depósito móvel"** (manual 2013 p.35, DN-2009, DN-2025): muda o `domains-v1.json` do contracts antes da tag, com SHA-256 novo para o api. O `census` (N < 2.000) fica como está. O modo `half` (Q-T1a) é decidido à parte. |
| Q17 (api Task 1, Q-T1a; decidida pelo usuário em 2026-11-17) | O modo `half` do LIRAa segue o manual de 2013 (p.13): os quarteirões são sorteados como no `standard`, com `n = ⌈450 ÷ (1 + 450 ÷ N)⌉` (N sem PE, Q16), e dentro de cada quarteirão sorteado se inspeciona **50% dos imóveis**, um a cada dois (intervalo 2, a partir do 1º, Q16). Por analogia com a p.15, `h = ⌈0,5 · N ÷ B⌉`, `b = ⌈n ÷ h⌉` e `k = max(1, ⌊B ÷ b⌋)`. Nos parâmetros (§11), `liraa_household_fraction` (0,2) fica sendo a do `standard` e entra a chave nova `liraa_half_household_fraction` (padrão 0,5, faixa 0,01–1,0 com duas casas, como a outra); o dashboard ganha o campo no tipo e na tela de parâmetros. O intervalo dos imóveis é `round(1 ÷ fração)` do modo (`standard` 5, `half` 2), e `HOUSEHOLD_INTERVAL` deixa de ser constante. Substitui a leitura do §7 (metade dos quarteirões, 20% dos imóveis, `n = ⌈0,5 × N⌉ × 0,2`). |
| Q18 (api Task 5; plano do dashboard) | `GET /vector_control/blocks?neighborhood_id=` devolve **todos** os quarteirões do bairro, sem página e sem `next_before`, na ordem natural do `number` (dígitos como inteiro, depois o resto, como o sorteio do LIRAa). O `neighborhood_id` é obrigatório (sem ele, `422` `path` `neighborhood_id`), e `before` é recusado como chave desconhecida. Nenhum bairro é cortado em silêncio; o escopo `CityBlock.natural_number_order` serve também ao sorteio do LIRAa. |
| Q19 (api Task 5b; detalha Q13(1) e Q14(2)) | (1) No dashboard, **criar** imóvel em microárea não atribuída ao agente de endemias → `403 out_of_scope` (Q14(2)); **editar** ou ler um imóvel fora das microáreas dele → `404`, como o "não visível" do 24a, para não revelar que o imóvel existe. No app continua `rejected not_found`, `path` `household_id`. (2) Imóvel não residencial com **morador ativo** não é editado pelo agente de endemias: `cbo_not_allowed` no dashboard e no app, conferido sob trava (a edição regravaria o endereço do morador). (3) A rota de família responde `403 cbo_not_allowed` ao 515140 nas microáreas dele; os outros papéis sem acesso seguem com `404`. |
| Q20 (api Task 6) | (1) No LIRAa, `vacant` conta como **fechado**: o item vira `substituted` e gera substituto, como `closed` e `refused`. O boletim do manual de 2013 (p.80) só tem as situações F, R e T, e o imóvel desocupado entra como fechado, com o motivo no rodapé. (2) A retirada do boletim desfaz só o que é da visita (`last_worked_on`, `last_situation`, a inspeção válida). `city_block_id` e `risk_class` gravados por aquele boletim **ficam**: são atributos do imóvel, e a coordenação os corrige pela rota de perfil. |
| Q21 (api Task 7) | (1) O estoque de insumo **não exige** o `pharmacy`: como o local de vacina do 29 (`StockGate`, `staff_controller`), as rotas de local e de equipe (`/pharmacy/admin/locations`, o vínculo de equipe ao local e `GET /pharmacy/locations`) conferem o interruptor **do domínio do local**: `vector_control` → `VectorControl::Gate`; os outros domínios seguem como estão. Com `pharmacy` desligado, a lista e o admin mostram só os locais `vector_control`. O `requires` do `vector_control` continua só `feature:home_visits`. No dashboard, a administração do local `vector_control`, a equipe dele, a entrada de nota e as faltas de insumo aparecem com `vector_control` utilizável mesmo sem `pharmacy`. (2) Causa da falta: `no_lot` só quando **não existe lote nenhum** do produto no local; havendo lote, mas nenhum válido com saldo > 0 (zerado, inclusive na corrida, ou vencido), a causa é `insufficient_balance`. Substitui a leitura da W-7. |
| Q22 (api Task 8) | (1) `vector_cadences` é a única tabela do território com `DELETE` liberado (o `PUT /cadences` troca a lista inteira, §12); `TRUNCATE` segue recusado e as outras tabelas continuam sem `DELETE`. (2) O `year` do ciclo tem de ser o **ano civil ou o ano epidemiológico** (`Surveillance::EpiWeek`) de `starts_on`; fora disso, `422 cycle_invalid` `path` `year`. Assim, o ciclo 1 que começa em 29/12/2024 (SE 1 de 2025) e o que começa em 02/01/2026 (ainda SE 2025-53) são aceitos com o ano que a cidade usa. (3) `ck_liraa_surveys_parameters_version` aceita `≥ 0` (versão 0 = padrões, A-4), ajustado na `20261116900003` antes de publicar. |
| Q23 (api Task 9) | (1) Correção de leitura de ovitrampa é **mudar a contagem**: a mesma `egg_count` com só a `count_note` nova é recusada com `422` `path` `egg_count` (no api e no `countProblems` do dashboard). (2) `ovitrap_number_taken` é `422` com `path` `number` (tabela do §17); o `409` da linha do teste do plano do dashboard estava errado. |
| Q24 (api Task 10) | (1) O N do estrato mostrado no editor do LIRAa é o do api: o estrato do levantamento em `draft` traz `households_count` (sem PE, Q16(d)) calculado a cada gravação, e o dashboard mostra esse número, não a soma de `CityBlock.households_count` (que continua contando PE). (2) "Fechar congela os índices" quer dizer congelar **quais inspeções contam**. Resultado de laboratório que chega depois do fechamento ainda entra: muda positivos e o índice fica `provisional` enquanto houver amostra do levantamento sem resultado; com todas lidas, o índice é final. |
| Q25 (api Task 10, notas da revisão) | (1) O substituto sai só da base do sorteio: imóvel cadastrado depois de `drawn_at` (ou que entrou no quarteirão depois) não substitui ninguém. Implementação: o sorteio grava a base de cada quarteirão sorteado em `liraa_stratum_blocks.base_household_ids uuid[]` (congelada pelo trigger depois do sorteio, nunca em resposta; CHECK: sorteado ⇔ base presente); a substituição usa essa base ∩ a base de hoje (ativo, no quarteirão, sem PE); sem elegível, o item fica `substituted` sem substituto. (2) Os valores do `defaults.yml` (versão 0) ficam presos pelo spec da prova técnica; mudar um padrão exige antes gravar a versão 1 com os valores antigos nas cidades que têm levantamento na versão 0, para nenhum índice fechado ser reclassificado. Sem código agora. (3) A retirada durante um fechamento fica como está (o índice congelado não muda). (4) O custo da contagem do N no `SaveSurvey` vai para a revisão final. |
| Q26 (api Task 11) | (1) O `ProposeBlockJob` age quando a notificação está `notified` **ou** `investigating` (a intenção do §8 é "chegou a `notified`"; a fila pode atrasar); `possible` e `closed` não agem, e a idempotência segue pelo caso único por notificação. (2) Mover caso grava quem moveu: o trigger da Task 4 passa a aceitar `added_by` `job` → `user` no move, e entra o evento `transmission_block_case.moved` (`transmission_block_id`, `from_transmission_block_id`, `user_id`; só ids), o 20º do §17. (3) O endereço do caso só abre em bloqueio `proposed` ou `active` (fechado ou descartado → `409 invalid_transition`); vale para caso com e sem lugar. A resposta vai com `Cache-Control: no-store` e grava também a trilha do domicílio. (4) **Revista na re-revisão da Task 11:** o `confirm` leva `seen_case_ids` (os casos que a tela mostrou) e recusa com `409 proposal_changed` se a proposta tiver caso fora dessa lista (entrou depois da leitura); o dashboard relê, avisa e confirma de novo com a lista nova. A área não é conferida contra o quarteirão dos casos (a tela não sabe o lugar de cada caso, §17), e caso movido, de cadastro alterado ou de quarteirão desativado não trava a confirmação. |
| Q27 (api Task 12) | Fila de amostras: `pending` na ordem (dia da coleta, etiqueta) **crescente** (a mais antiga primeiro); `done` pelo **resultado mais recente primeiro** (`resulted_at` decrescente, depois `id`). Correção de resultado segue a Q23: mesmos números com só a nota nova → `422` `path` `examined_larvae`. |
| Q28 (api Task 13) | (1) Na camada `transmission` do `map_points`, o rótulo do ponto é só a categoria, **sem o número do quarteirão** (o ponto único do quarteirão central não pode virar a casa do caso sem o step-up e a trilha do endereço). As camadas `risk` e `ovitraps` seguem como estão. (2) Diferença entre janelas sobrepostas do relatório `blocks` fica anotada como risco aceito: o relatório não tem lugar e quem o lê já vê os casos nos bloqueios. (3) O `vector_lab` lê os relatórios `ovitraps` **e** `liraa` (os dois dependem do resultado do laboratório e são agregados); o dashboard mostra os dois ao laboratório. (4) O CSV do `liraa` leva a coluna "Provisório" e as notas no fim; nenhum índice provisório sai como final. |
| Q29 (api Task 15) | (1) Entidade que entra no escopo do aparelho por junção (quarteirão do bairro de um domicílio novo ou movido, bloqueio que passa a cobrir quarteirão de perfil do escopo, ovitrampa e item de LIRAa de domicílio que muda para microárea do escopo) chega no próximo download: o `t` da entidade inclui o momento de entrada do membro que a traz, e o conjunto de quarteirões inclui os quarteirões dos perfis do escopo (nunca `vector.city_block_id` sem o `city_block`). (2) O app sobrescreve `city_block`, `ovitrap`, `liraa_item` e `transmission_block` qualquer que seja a `version` (o api reemite com a mesma `version` na contagem de ovos, no fechamento do LIRAa e no ciclo novo de UBV); vai uma linha no `field-sync/README.md` da v1.3.0, fora dos SHA-256. (3) A carga completa traz o histórico fechado, que o aparelho apaga ao receber: custo só de transferência, fica como está no 35a. |
| Q30 (api, revisão final) | O índice de levantamento do LIRAa **fechado** é o retrato oficial em `closed_at`: visita retirada depois do fechamento continua contando (o índice já pode ter sido informado ao estado). "Os índices ignoram visitas retiradas" vale para retiradas antes do fechamento. Resultado de laboratório que chega depois segue a Q24(2). |
| Q31 (dashboard Task 3/17) | Quem lê um relatório consegue escolher o filtro dele: o **index** de `GET /vector_control/cycles` abre para quem lê `situations` ou `strategic_points` (coordenação e `municipal_admin`), e o **index** de `GET /vector_control/liraa_surveys` abre para quem lê `liraa` (coordenação, `municipal_admin` e `vector_lab`). Só a listagem: detalhe, criação, edição, sorteio e fechamento continuam da coordenação. As duas listas não trazem pessoa. Mudança no api, em branch de correção do 35a, com specs de acesso; entra na suíte completa e no merge do fim do 35a, com autorização do usuário. |
| Q32 (dashboard Task 5) | `GET /vector_control/domains` é dado de referência, sem pessoa: abre para **quem lê a visita domiciliar** (24b: enfermeiro, médico, ACS, TACS e o agente de endemias) além da coordenação, do `vector_lab` e do `municipal_admin`, com o gate utilizável. Os rótulos do boletim no dashboard vêm sempre dessa rota (fonte única: `domains-v1.json` → `domains.yml` → rota), sem cópia estática no dashboard. Mudança no api na branch de correção do 35a (Q31). |
| Q33 (dashboard Task 7; revista para caber no texto publicado da v1.3.0) | Coleta de ovitrampa que não reinstala nem retira deixaria a armadilha ativa sem leitura aberta, presa ao número. O esquema e o texto da `field-sync-v1.3.0` ficam como publicados: `reinstall` é `true` por padrão e **ignorado com `remove: true`** (CV-9). A única recusa nova, regra do api: `remove` falso ou ausente **com `reinstall: false`** → `422 invalid_request` `path` `reinstall` no dashboard e `rejected invalid_request` `path` `reinstall` no app (retirar é sempre pedido explícito, nunca deduzido). `true/true` retira, como o texto publicado diz. Uma linha no `field-sync/README.md` registra a recusa (fora dos SHA-256, sem tag nova). Dashboard e campo mandam sempre `reinstall: !remove`. |
| Q34 (dashboard Task 10) | `PUT /vector_control/parameters` leva `expected_version` (a versão que a tela leu; `0` = padrões), como o `revert` de protocolo: ausente → `422 expected_version_required` `path` `expected_version`; diferente da versão vigente, conferida sob a trava → `409 parameters_changed`, sem gravar. O dashboard manda a versão lida e, no `409`, relê e mostra "Outra pessoa alterou os parâmetros; confira e salve de novo", mantendo o que foi digitado. Correção na branch do api do 35a (Q31). |
| Q35 (dashboard Task 11) | `PUT /vector_control/cadences` (troca a lista inteira) leva `expected_digest`: o `GET /vector_control/cadences` devolve `{ items, digest }`, com `digest` = SHA-256 da lista vigente em forma canônica (linhas ordenadas por bairro, risco e dias); ausente → `422 expected_digest_required` `path` `expected_digest`; diferente do digest vigente, calculado sob a trava → `409 cadences_changed`, sem gravar. Não usa `max(updated_at)`: lista vazia e lista apagada teriam o mesmo token. Sem migração. O dashboard manda o digest lido e, no `409`, relê e mantém o que foi digitado, como na Q34. Correção na branch do api do 35a (Q31). |
| Q36 (dashboard Task 13) | (1) Frases de sucesso do resultado de amostra: "resultado lançado" na primeira vez e "resultado corrigido" na correção (na convenção das frases de sucesso do 35a: minúscula e sem ponto); " O índice do LIRAa usa o resultado novo." só quando a amostra é de inspeção do LIRAa e a resposta já diz isso; sem esse dado, a frase fica sem a segunda parte (nada de campo novo no api só para isso). (2) **Correção** de contagem de ovos e de resultado de amostra leva `expected_version` (a `version` da leitura ou da amostra que a tela mostrou), como a Q34: ausente na correção → `422 expected_version_required`; diferente da vigente, conferida sob a trava → `409 reading_changed` / `409 sample_changed`, sem gravar. A primeira contagem e o primeiro resultado não levam versão (a corrida ali já é recusada). A `version` sobe a cada escrita da linha. Correção na branch do api do 35a (Q31). |
| Q37 (dashboard Task 14) | (1) Nome e período do levantamento do LIRAa são editáveis na tela **enquanto `draft`**, pelo `PATCH` que já existe (não há `DELETE`). (2) O `PATCH /vector_control/liraa_surveys/:id` leva `expected_version` (a `version` do levantamento, que já existe), como a Q34: ausente → `422 expected_version_required`; diferente da vigente sob a trava → `409 survey_changed`, sem gravar. Toda gravação do rascunho sobe a `version`, inclusive a que só muda os estratos ou o N (nota da revisão da Task 10 do api). Correção na branch do api do 35a (Q31). |
| Q38 (dashboard Task 15) | Rótulo de doença no 35a: rota nova `GET /vector_control/diseases` → `{ items: [{ code, name }], groups: [{ code, label }] }`: `items` = `Surveillance::Diseases.all` (lista vigente da plataforma mais as locais ativas, por nome), `groups` = os grupos do bloqueio com rótulo, no catálogo do 35a ao lado de `TransmissionBlock::DISEASE_GROUPS` (o 25 não tem grupo e não muda); só rótulo, sem pessoa; lê quem lê bloqueio ou edita parâmetros (coordenação e `municipal_admin`), com o gate do 35a utilizável, sem exigir o gate nem os papéis da vigilância. A tela do bloqueio e a de parâmetros (`block_diseases`) leem dali; código sem rótulo aparece cru. **Não** entra no `/vector_control/domains`: ele usa o mesmo serializador do `vector_control_domains` do app, que é fechado (`additionalProperties: false`) na `field-sync-v1.3.0`, e mudar exigiria tag nova. Correção na branch do api do 35a (Q31). |
| Q39 (dashboard Task 17) | O relatório `production` fica **sem o filtro de profissional na tela**: ele já lista uma linha por profissional, e nenhuma rota do 35a entrega `professional_id` à coordenação. O filtro `professional_ids` continua no api (CSV e chamadas diretas). |
| Q40 (dashboard Task 18) | (1) O almoxarife de um local `vector_control` recebe em `GET /pharmacy/locations` os **outros** locais `vector_control` ativos da cidade com `roles: []`, como no 27a (central) e no 29, para transferir entre bases; ler ou mexer no estoque deles continua exigindo vínculo. Correção na branch do api do 35a (Q31). (2) A falta conciliada mostra "conciliada por <nome>" (o `reconciled_by.name` que o api já devolve) ou "conciliada pelo sistema" (`reconciled_by: null`, estorno W-6), como o 32b. "Nenhuma linha com pessoa" do §17 é sobre cidadão e paciente, não sobre o profissional que agiu. |
| Q41 (dashboard Task 18A; **retirada** em 2026-11-18) | A primeira redação (tirar nome de responsável e moradores do que o 515140 recebe) estava errada e foi revertida (api `020799c3`, sem push): o agente de endemias visita domicílio **residencial** (LI), e a visita do 24b precisa do responsável e dos moradores para o `subject` (Q44 do 24b); o `field-sync-v1.3.0` também exige `family.responsible` e `who_item.display_name`. Vale a leitura de campo do 24b como está. A Q13(1) ("sem família nem morador") é sobre o que o agente **cadastra** (só imóvel não residencial, sem criar família nem morador), não sobre o que ele lê. Nenhuma mudança no api nem no app. |
| Q42 (campo Task 3; completa a Q6) | O app trata a **primeira página de um download sem cursor** exatamente como uma página `reset: true`: pelo mesmo caminho de código, dentro da transação dessa página (nunca antes do pedido), com o apagamento **só do território** (microárea, domicílio, família, cadastro e as quatro entidades do 35a; mantém a fila, o relógio confiável e as `local_visits`/`local_activities` já aceitas, que são as marcas de "visitado hoje" da folha; revisto na Task 4 do campo). O `reset: true` por troca de escopo segue apagando como no 24c. Fecha o caso em que o escopo mudou entre o último download da v1 e o primeiro da v2 (a migração zera o cursor e o api, sem cursor, não sabe mandar `reset`), e também o primeiro download depois de qualquer `wipe`. A regra da carga inicial da Q13(4) é avaliada antes do pedido e não muda. O api e o contrato não mudam. |
| Q43 (campo, antes da Task 8) | Domicílio com dois itens do LIRAa pendentes só existe com dois levantamentos sorteados ao mesmo tempo na cidade. O api escolhe o mais antigo (`created_at`, `id`); o app, sem essa data no feed, escolhe pelo menor `id`. A divergência fica **aceita e documentada** no 35a: não perde dado (a inspeção leva o `liraa_draw_item_id` escolhido) e o caso é raro. Trazer `drawn_at` no feed exigiria versão nova do `field-sync`. Também vale: a recontagem da Q42 mantém o `last_download_ok_at` (a Q13(4) não muda se a 2ª página falhar). |
| Q44 (campo Task 5) | (1) Código desconhecido **só** no `vector_control_domains` não recusa a referência inteira (a Q71 do 24c continua valendo para os outros dicionários): o app guarda a referência com `vectorDomains = null`, o boletim fica indisponível só para o agente de endemias, e o ACS e os outros CBOs seguem normais. (2) Frase do boletim indisponível (interruptor ligado sem domínios utilizáveis): "o boletim está indisponível neste aparelho — sincronize ou atualize o app". |
| Q45 (campo Task 7) | (1) Frases do boletim no app: "Entrada (hh:mm)", "Saída (hh:mm)" e "Saída em branco: a hora em que a visita for registrada." (o app preenche a saída ao salvar, Q11); "Retirar amostra" (o mesmo texto do dashboard, no lugar de "Apagar"); "Diminuir <rótulo>" / "Aumentar <rótulo>" como rótulos de acessibilidade dos contadores; "O boletim precisa de um imóvel cadastrado.". (2) Com o imóvel trabalhado, o app **não poda nem renumera** amostras (renumerar mudaria a etiqueta já escrita no tubito): `samples_exceed_larvae` se resolve subindo "Com larva" ou retirando a partir da última amostra, e grupo com amostra não sai. Divergência aceita com o dashboard. (3) No formulário da visita do app, as recusas do servidor aparecem sob o campo também para o ACS (padrão dos outros formulários). O `path` cru da tela "precisa de atenção" vira o rótulo da seção (Task 10). |
| Q46 (campo Task 8) | Frase da folha vazia: "Nenhum imóvel vencido nas suas microáreas." Divergências do app seguidas pelo api/dashboard ficam aceitas: imóvel em bloqueio ativo continua na folha depois da visita (`due.rb`); `startFor` igual ao do dashboard (pendência em PE → `{ activity: "pe", workType: "pending" }`); "vence em" antes do prazo; domicílio criado no tablet = "nunca trabalhado". |
| Q47 (campo Task 8) | O resumo local da visita aceita (Q11) guarda também o `liraa_draw_item_id`, para o imóvel não voltar à folha como LIRAa quando o boletim já foi aceito e o download seguinte falhou. Só no app; o contrato não muda. |
| Q48 (campo Task 8) | Fica a comparação como está: `accepted_at` (relógio do aparelho, `dispatch.ts` do 24c) recebe o `clock_skew_ms` antes de ser comparado com o `last_download_ok_at` (relógio confiável). O `dispatch.ts` do 24c não muda no 35a. |
| Q49–Q52 (campo Task 9) | (49) Coleta recusada continua por cima da lista, para a coleta seguinte não nascer presa atrás dela. (50) `acceptCollection` atualiza a linha local no aceite (retirada some; senão a palheta abre na data da coleta). (51) Sem os domínios do vetor, a instalação segue liberada e a coleta não confere a janela no app, como no dashboard; o api confere e recusa. (52) Descartar a instalação apaga a linha criada no tablet e as coletas em cascata. Frases: "Imóvel", "Termo (TCLE)", "Ovitrampa", títulos "Ovitrampas" e "Corrigir número", cabeçalho "código · nome". Sem caixa "Reinstalar palheta" (Q33) e sem campo de data (hoje da cidade). |
| Q53–Q55 (campo Task 9b) | (53) Rótulo "Novo imóvel não residencial" para o agente no app. (54) Na recusa do servidor ao agente, `phraseForItem` usa as frases do dashboard: `AGENT_CBO_NOT_ALLOWED` para tipo residencial ou criação de família/morador, `AGENT_HAS_RESIDENT` para imóvel com morador; `out_of_scope` fica com a frase do 24c. (55) Imóvel não residencial com morador ativo no tablet esconde "Editar domicílio" e mostra `AGENT_HAS_RESIDENT`, com a mesma guarda no `saveHousehold` (o api segue como autoridade); microárea fora da sessão é recusada na tela com `out_of_scope` em `micro_area_id`. |
| Q56–Q57 (campo Task 10) | (56) "Gerar etiquetas novas" é o botão da Q7(1); depois dele o app mostra "Etiquete os tubitos: …" com as etiquetas novas. (57) No "precisa de atenção", `liraa_draw_item_id` e `transmission_block_id` → "Atividade"; a raiz do boletim e paths desconhecidos dentro dele → "Boletim do agente de endemias"; o path cru nunca aparece. |
| Q58 (campo Task 10) | Toda visita com amostra ainda não aceita mostra "Etiquete os tubitos: …" na sua linha em Hoje, com as etiquetas **vigentes** (as do primeiro salvamento ou as geradas por "Gerar etiquetas novas"), calculadas do `client_uuid` e dos números das amostras; some no aceite. Assim nada depende de o agente não sair da tela. A etiqueta antiga, depois de trocada, não aparece em lugar nenhum (Q56). |
| Q59 (campo Task 11) | No app, o detalhe do imóvel ganha "Registrar visita" para o agente de endemias com `canRecordInspection`, **quando o imóvel não tem morador ativo** (PE, comércio, terreno; `subject` sem pessoa), com o `startFor` do imóvel (PE → atividade `pe`; senão vazio). Cobre revisita antes do prazo e denúncia, fora da folha. Imóvel com morador continua recebendo a visita pelo morador, como no 24c. Sem frase nova. Entra antes da prova da Task 12. |
