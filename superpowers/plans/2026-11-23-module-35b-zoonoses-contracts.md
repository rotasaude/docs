# Módulo 35b — Contratos entre apps

**Fonte única** do que os repositórios do 35b combinam entre si. As linhas do §19 prevalecem sobre o texto das seções anteriores. Os planos de cada repositório (`-contracts-repo.md`, `-api.md`, `-dashboard.md`, `-wpda.md`, `-maintenance.md`, `-campo.md`) seguem este documento. Divergência vai para a seção "Divergências propostas ao contrato" do plano, e a decisão volta aqui como Q-n.

- **Spec:** `docs/superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md`: decisões do usuário 1–7 e do coordenador 14–25 e 29–32. **ADR:** 0054. **Pesquisa:** `docs/pesquisa/2026-11-14-zoonoses-e-endemias-35.md`, §3. **Módulo:** `docs/modulos/35--zoonoses-e-endemias.md`.
- **Base:**
  - contrato do 35a, `2026-11-16-module-35a-vector-control-contracts.md`, até a Q63;
  - contratos do 24a, 24b e 24c, do 25 e do 33;
  - `field-sync-v1.3.0`.
  Nada deles muda, salvo o que este documento diz que muda.
- **F-IDs:** F-35.11 a F-35.18, do board #1.

Convenções herdadas (24a, 24b, 33 e 35a):
- erros `{ "error" }`, com `path` nas recusas de validação;
- step-up `401 mfa_required`;
- interruptor desligado `403 feature_disabled`;
- JSON em toda escrita por cookie;
- listas `{ items, next_before }`, com cursor `before` opaco;
- registro de outro escopo por id → 404;
- leitura nominal com trilha e sem polling; contagens sem trilha;
- texto longo é recusado, nunca cortado;
- dado sensível só no corpo, nunca em URL nem em log;
- concorrência otimista por `expected_version` → 409 `<entidade>_changed` (Q34–Q37 do 35a);
- `Data`/`Struct` com dado pessoal sem esse dado em `inspect`, `to_s` e `as_json`.

**Bases** (origin/main em 2026-11-23):

| Repo | Commit |
|---|---|
| api | `85dc80ee` |
| dashboard | `2e3a2a9` |
| wpda | `cfa6675` |
| maintenance | `482e53b` |
| contracts | `6ce25c2` (`field-sync-v1.3.0` em `c022e80`) |
| campo | `dbc54b3` |

**Valores fixos:**

| Item | Valor |
|---|---|
| Porta do api | 3061 |
| Vite do dashboard | 5221 |
| Vite do wpda | 5222, com `WPDA_ALLOW_DESKTOP=1` |
| maintenance | 5177, com o serviço do compose parado durante a prova |
| Migrações da cidade | a partir de `20261123900001` (a última na main é `20261116900003`); nenhuma de plataforma |
| Branch (todos os repos) | `feat/mod-35b-zoonoses` |
| Worktree do api | `apps/api/.claude/mod35b`, com `config/master.key` copiada |
| Worktrees dos fronts, do contracts e do campo | `rota-saude/.worktrees/<repo>-mod35b` |
| Testes | só `bin/test api -w mod35b …`, `bin/test <app> -w <repo>-mod35b` e `bin/test campo -w campo-mod35b`; suíte completa pela sessão "Testes" |
| Tag do contracts | `field-sync-v1.4.0` (MINOR); `session.contract` continua `"field-sync-v1"` |

Os worktrees do 35a continuam de pé até o fim do 35 (decisão do usuário, 2026-11-23). Nenhum plano do 35b os usa nem os remove.

**Mudança transversal.** O 35b toca:
- migração de cidade e `CityEncryption::CITY_KEYED_TARGETS`;
- `Platform::Features`, `Membership::ROLES` e `config/professionals/cbo_saude.yml`;
- o estoque do 27a (`Stock::Domain`, `stock_movements`);
- a ficha da notificação do 25 (`config/surveillance/sinan-form.yml`, `Surveillance::Form`) e os vínculos de eventos do 25;
- o registro e o encaminhamento da ouvidoria do 33;
- o `field-sync`.

A suíte completa do api é obrigatória antes do merge.

## 1. Nomes no código

**Interruptor** (fim do `CATALOG` de `Platform::Features`, depois de `vector_control`):

| Chave | Descrição | `requires` |
|---|---|---|
| `zoonoses` | "Centro de zoonoses: animal com tutor, vacinação antirrábica, demandas, observação do agressor, leishmaniose canina e canil" | `%w[feature:territory_registry]` |

- O rótulo curto, no maintenance e no dashboard, é "Centro de zoonoses".
- A chave entra em `CITIZEN_HIDDEN`, mas o wpda lê a chave para a ouvidoria (§4). A leitura usa o mesmo caminho pelo qual o wpda já lê `ombudsman`.
- **Gates:**
  - `Zoonoses::Gate.usable?(city)` = a chave está habilitada e `TerritoryRegistry::Gate.usable?(city)`.
  - `Zoonoses::Gate.field_usable?(city)` = `usable?` e `HomeVisits::Gate.usable?(city)`. Vale para o app (§11) e para o casa a casa.
- **Desligado:**
  - as rotas `/zoonoses` respondem `403 feature_disabled`;
  - os itens do app voltam `rejected feature_disabled`;
  - os jobs do §5 não agem;
  - o que foi gravado fica.
- **Sem cascata para o `vector_control`.** O 35b funciona com ou sem o 35a. O que ele usa do 35a fica fora dos dois gates: o catálogo de insumos (§3) e o local de estoque da UVZ.

**Papel novo** (`Membership::ROLES` + `ck_memberships_role`; vale para a cidade inteira):

| Papel | Rótulo | Privilegiado | Faz |
|---|---|---|---|
| `zoonoses_team` | "Equipe de zoonoses" | não | animal, vacinação no posto fixo e na rotina, demandas, observação do agressor, coleta e resultado da sorologia, coleira, canil (entrada, saída, castração); lê o bloco do animal e o contato do tutor da notificação (§5) |

- **Coordenação:** é o `endemic_coordinator` do 35a, que passa a ser a "Coordenação de vigilância ambiental" no rótulo (Q1). Ele faz campanha, inquérito, parâmetros do 35b e relatórios, além de tudo o que o `zoonoses_team` faz.
- **`municipal_admin`:** lê os relatórios e não escreve nada do 35b.

**Médico veterinário:**
- Entra em `config/professionals/cbo_saude.yml` como `{ code: "223305", title: "Médico veterinário", council: "CRMV" }`. O `CRMV` já está em `Professional::COUNCILS`.
- Para o 35b, veterinário é o usuário com perfil profissional de CBO `223305`, conselho CRMV completo e vínculo ativo (`Professionals::Link`) numa unidade `environmental_surveillance` ativa.
- Só ele registra eutanásia (§7). Ele também pode fazer tudo o que o `zoonoses_team` faz, sem precisar do papel.
- O método é `Zoonoses::Access.veterinarian?(user)`.

**Acesso** (`Zoonoses::Access`):

| Método | Regra |
|---|---|
| `team?(user)` | papel ativo `zoonoses_team`, ou `veterinarian?`, ou `coordinator?` |
| `coordinator?(user)` | papel ativo `endemic_coordinator` |
| `veterinarian?(user)` | ver acima |
| `reader?(user)` | `team?` ou `municipal_admin` |
| `field_agent?(user)` | `HomeVisits::Access.registers?` na microárea, com CBO `515140` ou `515105` (§12) |

**Unidade:** a mesma `environmental_surveillance` do 35a, com o rótulo "Vigilância ambiental e zoonoses". O 35b não cria tipo de unidade. O canil fica na unidade (§7).

**Tabelas novas da cidade.** Todas são `id: :uuid` e têm `created_at`. As mutáveis também têm `updated_at` e `version` (≥ 1, `+1` a cada escrita). "Cifrado" quer dizer `CityEncryption`, e a coluna entra em `CITY_KEYED_TARGETS`.

- **`animals`**:
  - `species` (`dog` | `cat` | `other`), `name` (1–60, opcional), `sex` (`male` | `female` | `unknown`), `size` (`small` | `medium` | `large` | nulo);
  - `coat` (1–40, opcional), `breed` (1–60, opcional, "SRD" aceito), `birth_estimate_on` (date, opcional; o app manda o mês e o dia vira 1);
  - `microchip` (15 dígitos, opcional; cifrado **determinístico**, único entre os não nulos), `registry_number` (1–20, opcional; único sem caixa);
  - `household_id` (opcional), `tutor_patient_id` (opcional; FK `patients`), `community` (bool: comunitário ou recolhido, sem tutor);
  - `status` (`active` | `deceased` | `missing` | `transferred`), `status_on` (date), `neutered` (bool);
  - `created_by_user_id`;
  - CHECK: `tutor_patient_id IS NULL OR community = false`.
- **`rabies_campaigns`**:
  - `name` (1–80), `year`, `starts_on`, `ends_on`;
  - `goal_dogs` e `goal_cats` (inteiros ≥ 0), `stage` (`planned` | `open` | `closed`).
  - Só pode haver uma `open` por vez.
- **`rabies_vaccinations`**:
  - `animal_id` (opcional: posto fixo sem cadastro), `species` (`dog` | `cat`; congelada, igual à do animal quando houver);
  - `campaign_id` (opcional; nulo = rotina), `modality` (`house` | `fixed_post` | `routine`), `vaccinated_on`;
  - `health_unit_id` (onde foi registrada; nulo no casa a casa), `household_id` (casa a casa);
  - `vector_product_id` (a vacina, §3), `stock_lot_id` (nulo se faltou, §3);
  - `adverse_event` (bool), `adverse_note` (cifrado, 10–500, opcional);
  - `recorded_by_user_id`, `client_uuid` (app; único entre os não nulos);
  - `cancelled_at`, `cancel_reason` (cifrado, 10–500). Só se anula uma vez, e a anulação estorna o estoque.
  - Imutável fora da anulação (trigger, como `rota_immunization_guard`).
- **`zoonoses_demands`** (1:1 com `ombudsman_forwardings`, §4):
  - `ombudsman_forwarding_id` (único), `subtype` (§4), `state` (`open` | `scheduled` | `inspected` | `closed`);
  - `scheduled_on`, `inspected_on`, `inspection_note` (cifrado, 10–2000), `outcome` (`resolved` | `guidance` | `not_found` | `referred` | nulo);
  - `assigned_user_id`.
- **`bite_observations`** (§5):
  - `disease_notification_id` (único), `species` (`dog` | `cat`), `exposure_on`, `ends_on` (= `exposure_on` + `observation_days`);
  - `place` (`home` | `shelter`), `animal_id` (opcional);
  - `state` (`open` | `closed` | `cancelled`), `outcome` (`healthy` | `died` | `missing` | `rabies_signs` | `sent_to_exam` | `euthanized` | nulo), `outcome_on`, `outcome_note` (cifrado, 10–500, opcional);
  - `closed_by_user_id`.
- **`leish_surveys`**:
  - `name` (1–80), `kind` (`sample` | `census` | `focus`), `neighborhood_id`, `city_block_ids` (uuid[], não vazio);
  - `dog_population` (inteiro > 0, informado), `target_dogs` (calculado, §6), `starts_on`, `ends_on`;
  - `stage` (`draft` | `open` | `closed`), `parameters_version`.
- **`leish_tests`**:
  - `leish_survey_id` (opcional: exame fora de inquérito), `animal_id`, `collected_on`, `label` (único; §6);
  - `dpp_result` (`reactive` | `non_reactive` | `invalid` | nulo), `dpp_on`;
  - `elisa_result` (`reactive` | `non_reactive` | `inconclusive` | nulo), `elisa_on`, `elisa_request` (1–30, opcional: o número da requisição no GAL);
  - `result_by_user_id`, `client_uuid`.
- **`leish_dog_outcomes`** (cão reagente, §6):
  - `animal_id`, `leish_test_id`, `decision` (`tutor_treatment` | `euthanasia` | `refused`), `decided_on`, `note` (cifrado, 10–500, opcional).
- **`collars`**:
  - `animal_id`, `vector_product_id`, `placed_on`, `due_on` (= `placed_on` + `collar_months`), `removed_on`, `removal_reason` (`replaced` | `lost` | `death` | `other` | nulo);
  - `client_uuid`.
  - Só pode haver uma coleira ativa (sem `removed_on`) por animal.
- **`shelter_stays`**:
  - `animal_id`, `health_unit_id` (unidade `environmental_surveillance`), `entry_kind` (`collection` | `voluntary_surrender` | `observation` | `other`), `entered_on`;
  - `exit_kind` (`returned` | `adopted` | `death` | `euthanasia` | `transferred` | nulo), `exited_on`, `adopter_patient_id` (opcional, só em `adopted`);
  - `bite_observation_id` (opcional, só em `observation`).
  - Só pode haver uma estada aberta por animal.
- **`neuterings`**: `animal_id`, `performed_on`, `professional_id` (veterinário), `shelter_stay_id` (opcional). É só acréscimo.
- **`euthanasias`** (§7), só acréscimo e imutável por trigger:
  - `animal_id`, `performed_on`, `professional_id` (veterinário, congelado);
  - `reason` (`leishmaniasis` | `rabies_suspect` | `untreatable_suffering` | `public_health_risk` | `other`), `reason_note` (cifrado, 10–1000, obrigatório);
  - `legal_basis` (`cfmv_1000_2012` | `law_14228_2021`, lista não vazia);
  - `shelter_stay_id`, `leish_dog_outcome_id`, `bite_observation_id` (opcionais).
- **`zoonoses_parameter_versions`**: igual ao `vector_parameter_versions` do 35a (§9).

**Colunas novas em tabelas existentes:**
- `ombudsman_subjects.code` (texto, opcional, único entre os não nulos). Marca os assuntos reservados do §4.
- `ombudsman_manifestations.place_address` (cifrado, 10–300, opcional; §4).
- `vector_products.uses`: a lista aceita também `rabies_vaccine` e `collar` (§3).

## 2. Animal

- **Quem cadastra:** `Zoonoses::Access.team?`, pelo dashboard, e o agente de campo pelo app (§12). O agente cadastra só animal de domicílio das suas microáreas.
- **Tutor:**
  - É sempre um `Patient`, escolhido pela busca que o dashboard já tem (CPF ou CNS no corpo, com trilha). O nome digitado não vale como tutor.
  - Sem tutor, o animal é `community: true` ou fica sem tutor e sem domicílio, como o recolhido pelo canil.
  - O tutor não precisa morar no domicílio do animal.
- **Domicílio:** opcional, e só de domicílio ativo do 24a. Animal de domicílio inativo continua, mas não aparece na folha.
- **Busca** (`POST /zoonoses/animals/search`, com o corpo `{ microchip? , registry_number?, tutor_patient_id?, household_id? }`):
  - com `microchip` ou `registry_number`, devolve no máximo 1;
  - com tutor ou domicílio, devolve a lista.
  - Nunca vai por GET: o microchip não vai em URL nem em log (decisão 23).
- **Histórico** (`GET /zoonoses/animals/:id`): devolve o animal mais `vaccinations[]`, `observations[]`, `tests[]`, `collars[]`, `stays[]`, `neuterings[]` e `euthanasia`, montados por consulta, sem tabela de eventos (Q2).
  - O tutor sai só como `{ id, display_name }`, o mesmo do 24a.
  - A leitura com tutor grava a trilha `animal.viewed` (§16).
- **Edição:** `PATCH` com `expected_version` → 409 `animal_changed`.
  - `status` vai de `active` para os outros estados e não volta.
  - `deceased` também nasce da eutanásia (§7) e da saída `death` do canil.
- **Duplicado:**
  - microchip ou registro repetido → 422 `microchip_taken` ou `registry_number_taken`, com `path`;
  - pelo app, a mesma recusa volta como `rejected` (§11).

## 3. Vacinação antirrábica, campanha e insumos

**Insumos.** A vacina antirrábica animal e a coleira de deltametrina são `vector_products` do 35a (decisão 29), com `uses` `rabies_vaccine` e `collar` e `unit` `un` (`charge_size` 1).
- Elas aparecem na tela de produtos do 35a quando a cidade tem `zoonoses` ou `vector_control` utilizável (Q3). As rotas de produto passam a aceitar qualquer um dos dois gates.
- `Stock::Domain.usable?(city, VECTOR_CONTROL)` passa a ser `vector_control` **ou** `zoonoses` utilizável.

**Baixa do estoque.**
- Toda vacinação e toda coleira baixam 1 un do local `vector_control_settings.stock_location_id` (o local da UVZ do 35a) pelo `VectorControl::Fefo`. O lote não é escolhido.
- Tipos de movimento novos, com sinais:

  | Tipo | Sinal |
  |---|---|
  | `animal_vaccination` | negativo |
  | `animal_vaccination_reversal` | positivo |
  | `collar_placement` | negativo |

- Documentos novos em `ck_stock_movements_document`: `rabies_vaccination_id` e `collar_id`.
- Falta de saldo, de lote ou de local **nunca recusa** o registro. Ela grava `vector_stock_shortfalls` com a causa do 35a e uma referência nova (`rabies_vaccination_id` ou `collar_id`). A coordenação e o `stock_keeper` conciliam a falta na tela do 35a.
- `stock_lot_id` fica nulo na vacinação que faltou.

**Campanha.**
- O `coordinator?` cria a campanha e passa de `planned` a `open` e a `closed`, sem voltar. Abrir uma segunda → 409 `campaign_already_open`.
- As metas são de cães e de gatos. O padrão sugerido é `round(estimativa × rabies_coverage_target)`, com a estimativa do §9, e o coordenador pode trocar.

**Registro.**
- **`house`:** pelo app, ou pelo dashboard com domicílio. Exige `field_usable?`, animal cadastrado e campanha `open`.
- **`fixed_post`:** pelo dashboard, numa campanha `open`. O animal é opcional: sem cadastro, a dose leva só `species` (decisão 16; Q4). Sem nome de tutor digitado.
- **`routine`:** pelo dashboard, sem campanha, com animal cadastrado.
- **Data:** `vaccinated_on` fica entre o início da campanha (ou 30 dias antes, na rotina) e hoje.
- **Repetição:** uma segunda dose do mesmo animal na mesma campanha → 422 `already_vaccinated_in_campaign`.
- **Evento adverso:** `adverse_event: true` mostra o lembrete fixo "Notifique o evento adverso no formulário do Ministério da Saúde (EATAV)." O registro não vai a nenhum sistema.
- **Anulação:** `POST …/cancel` com `reason`. É feita por quem registrou ou pelo `coordinator?`, e estorna a baixa.

**Resultado semanal** (decisão 16):
- Vem por semana epidemiológica (a mesma regra da Q22 do 35a), por campanha.
- Traz cães e gatos vacinados (sem as anuladas), o acumulado, a meta e a cobertura (acumulado ÷ meta, uma casa decimal).
- Sai em tela e em CSV (`VectorControl::Csv`, com o mesmo formato).
- Nenhuma contagem é suprimida: não há pessoa no agregado (Q5).

## 4. Demandas de zoonoses pela ouvidoria

**Assuntos reservados.**
- Ao ligar o `zoonoses`, a cidade ganha, se ainda não tiver, o assunto pai "Zoonoses" (`code` `zoonoses`) e oito filhos, com `code` `zoonoses.<subtipo>`:

  | Subtipo | Rótulo |
  |---|---|
  | `scorpion` | Escorpião |
  | `bat` | Morcego |
  | `rodent` | Rato ou outro roedor |
  | `tick` | Carrapato |
  | `aggressive_animal` | Animal que mordeu ou atacou |
  | `neurological_animal` | Animal com comportamento estranho ou paralisia |
  | `sporotrichosis` | Gato com feridas (esporotricose) |
  | `other` | Outro problema com animal |

- O assunto com `code` pode ser renomeado, mas não pode ser desativado com o `zoonoses` ligado → 409 `reserved_subject`.

**Entrada.** O registro do 33 ganha duas chaves opcionais, `zoonoses_subtype` e `place_address`.
- **wpda, manifestação identificada ou preservada:**
  - aceita com `kind` `request` (Solicitação) ou `report` (Denúncia);
  - `place_address` é obrigatório quando houver `zoonoses_subtype`.
- **wpda, denúncia anônima:**
  - aceita `zoonoses_subtype` e `place_address`;
  - continua `kind` `report`, porque o CHECK do 33 não muda (Q6).
- **Equipe da ouvidoria, no balcão, telefone ou OuvidorSUS:** aceita as mesmas duas chaves.
- **Chave sem uso:** com o `zoonoses` desligado, a chave é recusada → 422 `invalid_request`, com `path` `zoonoses_subtype`.

**Encaminhamento automático** (decisão 31). Na mesma transação do registro, quando houver `zoonoses_subtype`:
1. O `Ombudsman::Classify` roda com o ator do sistema e o assunto `zoonoses.<subtipo>`: `received` → `in_analysis`.
2. O `Ombudsman::Forward` roda com o ator do sistema para a unidade `environmental_surveillance` ativa da cidade:
   - se houver mais de uma unidade, vai para a de menor `created_at`;
   - o extrato é montado pelo api: o rótulo do subtipo, o texto do cidadão e o `place_address`, nessa ordem;
   - o prazo do encaminhamento é o do 33 (`forwarding_days`);
   - vale para os três modos, inclusive o anônimo.
3. Nasce a `zoonoses_demands` `open`, ligada ao encaminhamento.

Sem unidade de vigilância ambiental ativa, a manifestação fica `in_analysis`, já classificada, e segue a triagem normal do 33.

**Quem responde.** A unidade da UVZ conta como destino com respondente quando houver um usuário ativo com `Zoonoses::Access.team?`. O `destination_without_responder` do 33 passa a considerar esse caso, e ninguém precisa do papel `ombudsman_responder`.

**Fila do centro** (`/zoonoses/demands`):
- Mostra o subtipo, o extrato, o `place_address`, o prazo e a cor do 33, e o estado da demanda.
- Nunca mostra identidade, contato, protocolo nem anexos.
- O `place_address` é do lugar, não da pessoa (decisão 31). Ainda assim, sai só no detalhe, com trilha `zoonoses_demand.viewed`.

**Andamento:**
- `open` → `scheduled` (data), → `inspected` (data e nota), → `closed`. `closed` sempre traz o `outcome` e um texto de 20 a 4000 caracteres, que vira a resposta do encaminhamento (`Ombudsman::ReplyForwarding`).
- Em qualquer estado aberto, a demanda pode ser recusada pelo `Ombudsman::DeclineForwarding`, com texto, e fica `closed` sem `outcome`.
- O pedido de prorrogação usa o `Ombudsman::ExtendForwarding` do 33, uma vez.
- Tudo com `expected_version` → 409 `demand_changed`. A ação sobre encaminhamento cancelado → `forwarding_changed` do 33.
- A resposta ao cidadão continua com a equipe da ouvidoria, pelo fluxo e pelos prazos do 33.

## 5. Observação do animal agressor (25)

**Bloco do animal na ficha.**
- `config/surveillance/sinan-form.yml` ganha a seção `rabies_animal`, aceita só quando `disease_code` é `acidente_animal_raiva`. Os campos ficam no mesmo `form` cifrado:

  | Campo | Valores |
  |---|---|
  | `animal_species` | `dog` `cat` `bat` `primate` `fox` `livestock` `other` |
  | `animal_condition` | `healthy` `suspect` `rabid` `dead_or_missing` |
  | `animal_observable` | bool; só pode ser `true` com `dog` ou `cat` |
  | `exposure_on` | data |
  | `animal_tutor_name` | 1–120 |
  | `animal_tutor_phone` | 10–11 dígitos |
  | `animal_address` | 10–300 |

- `Surveillance::Form` recusa campo da seção com outra doença → 422 `invalid_request`, com `path` `form.<campo>`.
- O dashboard mostra a seção no detalhe da notificação pelo `formSlot` (decisão 32).

**Abertura.**
- `Zoonoses::OpenObservationJob` (`IdempotentConsumer`) fica ligado a `disease_notification.created`, `disease_notification.updated` e `disease_notification.transitioned`, como o `ProposeBlockJob` do 35a.
- Ele abre a `bite_observations` quando todas as condições valem:
  - `Zoonoses::Gate.usable?`;
  - a doença é `acidente_animal_raiva`;
  - o estado é `notified` ou `investigating`;
  - `animal_observable` é `true`, com espécie `dog` ou `cat`, e há `exposure_on`;
  - ainda não há observação para a notificação.
- `ends_on` = `exposure_on` + `observation_days` (§9).
- Uma notificação `dismissed` antes do desfecho leva a observação a `cancelled`.

**Fila do centro** (`/zoonoses/observations`):
- Mostra só o bloco: espécie, condição, `exposure_on`, `ends_on` e os dias que faltam.
- O nome, o telefone do tutor e o endereço do animal saem só no detalhe, com trilha `bite_observation.contact_viewed` (como o `address_viewed` do 35a).
- O paciente nunca aparece: nem nome, nem idade, nem unidade (decisão 32).

**Desfecho:**
- `POST …/close` com `outcome`, `outcome_on` (≥ `exposure_on`, ≤ hoje) e nota opcional, e `expected_version` → 409 `observation_changed`.
- `healthy` antes de `ends_on` → 422 `observation_not_finished`.
- `euthanized` exige a eutanásia registrada (§7), ligada pela `bite_observation_id`.
- O animal pode ser cadastrado e ligado a qualquer momento (`animal_id`). Com `place` `shelter`, a observação nasce com a estada de entrada `observation` (§7).

**Volta à notificação.**
- O JSON da notificação (`Surveillance::Json.notification`) ganha `animal_observation: { state, ends_on, outcome, outcome_on } | null`, lido da tabela do 35b.
- O 35b não escreve na `disease_notifications` (Q7).
- O detalhe da notificação mostra "Condição final do animal após a observação".
- A fila do 25 ganha o sinal `animal_outcome_pending`: a observação foi fechada e a notificação ainda está aberta. Esse sinal é o aviso à unidade que atendeu.

## 6. Leishmaniose visceral canina

**Inquérito** (`coordinator?`):
- `draft` → `open` → `closed`, sem voltar.
- **Cálculo do `target_dogs`:**
  - `sample`: pela tabela `leish_sample_table` dos parâmetros, população × prevalência esperada (`leish_expected_prevalence`, padrão 0,02). O cálculo usa a primeira linha cuja população seja ≥ `dog_population`.
  - `census`: igual a `dog_population`.
  - `focus`: `max(leish_focus_min_dogs, dog_population)`, com padrão de 100.
- Os quarteirões são os do 35a (`city_blocks`). O `focus` sugere os anéis do 35a em volta de um quarteirão escolhido, e o coordenador ajusta.
- O índice usado é o do 35a: o inquérito do 35b depende dos quarteirões cadastrados, não do interruptor `vector_control` (Q8).
- `dog_population` é informado. A tela sugere a soma dos cães ativos cadastrados nos quarteirões.

**Coleta e resultado:**
- A coleta pode ser feita no app (§12) ou no dashboard. Ela exige um animal `dog` ativo.
- **Etiqueta:** `LV-<ano>-<6 hex do id do inquérito>-<seq de 4 dígitos>`. Fora de inquérito, usa `LV-<ano>-AVUL-<seq>`. É única na cidade.
- **TR-DPP:** lançado no campo (app) ou na UVZ.
- **ELISA:**
  - só depois de um DPP `reactive`; sem ele → 422 `elisa_without_reactive_dpp`;
  - `elisa_request` guarda o número do GAL, digitado.
- Os resultados são lançados por `team?`. A correção leva `expected_version` → 409 `test_changed`.
- **Cão reagente:** DPP `reactive` e ELISA `reactive`. Ele entra na lista "Desfecho pendente". A `leish_dog_outcomes` registra a decisão:
  - `euthanasia` exige a eutanásia registrada (§7);
  - `tutor_treatment` e `refused` exigem um tutor.

**Prevalência:**
- É calculada por inquérito: reagentes ÷ examinados com resultado final, com uma casa.
- O resultado final é o ELISA; o DPP não reagente conta como negativo.

**Coleira:**
- Colocada no app ou no dashboard, ela baixa o estoque (§3).
- `due_on` = `placed_on` + `collar_months` (padrão de 6).
- Coleira nova num animal que já tem uma ativa: a anterior sai com `replaced`, na mesma transação.
- A lista "Coleiras a trocar" mostra as coleiras com `due_on` em até 15 dias. A folha do app mostra as mesmas coleiras, pelas microáreas do agente (§12).

## 7. Canil, gatil, castração e eutanásia

**Estada:**
- **Entrada:** `team?` registra, com o animal. O animal sem cadastro é criado na hora, sem tutor e sem domicílio.
  - Entrada com uma estada já aberta → 409 `stay_already_open`.
  - A entrada `observation` exige uma observação `open` com `place` `shelter`.
- **Saída:**
  - `returned` exige tutor;
  - `adopted` exige `adopter_patient_id`, que passa a ser o tutor;
  - `death` e `euthanasia` levam o animal a `deceased`;
  - `euthanasia` exige a eutanásia registrada.
  - Tudo com `expected_version` → 409 `stay_changed`.
- **Lotação:** estadas abertas por unidade, sem limite no sistema.

**Castração:** só `veterinarian?`. Marca `animals.neutered = true`, que não volta.

**Eutanásia** (decisão 20; invariante do ADR 0054):
- Só `veterinarian?`. Outro usuário → 403 `veterinarian_required`.
- É só acréscimo e imutável: trigger `rota_append_only` e nenhuma rota de edição.
- O `professional_id` é o do usuário logado, congelado.
- `reason_note` é obrigatório; `legal_basis` tem ao menos um item.
- O animal vai a `deceased` com `status_on` = `performed_on`.
- Registra no máximo uma por animal → 409 `already_euthanized`.
- Exige step-up (`mfa_required`), como as ações privilegiadas.

## 8. Relatórios (35b)

Os relatórios ficam em `Zoonoses::Reports`, com CSV pelo formato do 35a. Todos levam filtro de período.

| Nome | Conteúdo | Supressão |
|---|---|---|
| `rabies_weekly` | §3, por campanha | não |
| `rabies_modalities` | doses por modalidade e por espécie no período | não |
| `demands` | demandas por subtipo, estado e cor do prazo | sim: `Admin::SmallCount` nas células por subtipo |
| `observations` | observações por desfecho e por espécie | sim |
| `leish` | por inquérito: examinados, reagentes, prevalência, coleiras colocadas e a trocar | não |
| `shelter` | lotação atual, entradas e saídas por tipo, castrações e eutanásias por motivo | não |

Quem lê: `reader?`. O `municipal_admin` lê todos.

## 9. Parâmetros (35b)

Ficam em `zoonoses_parameter_versions` e seguem a mesma regra da Q34 do 35a: `expected_version`, 409 `parameters_changed`, versão 0 = `config/zoonoses/defaults.yml`. Só o `coordinator?` escreve.

| Chave | Padrão | Faixa |
|---|---|---|
| `observation_days` | 10 | 10–30 |
| `collar_months` | 6 | 1–12 |
| `leish_expected_prevalence` | 0.02 | 0.01–0.5 |
| `leish_sample_table` | tabela do manual LV-2026, 4.6.4.1 (população → amostra) | linhas crescentes |
| `leish_focus_min_dogs` | 100 | 10–1000 |
| `dogs_per_block` | 16 | 1–100 |
| `rabies_coverage_target` | 0.8 | 0.1–1 |
| `dog_population_estimate` | nulo | inteiro ≥ 0 ou nulo |
| `cat_population_estimate` | nulo | inteiro ≥ 0 ou nulo |

A tabela do `leish_sample_table` é copiada do manual pelo plano do api, com a fonte no cabeçalho do yml. Valor que não for conferido no manual fica marcado como "a conferir", e isso vira gate de go-live (§18).

## 10. Rotas do api

Escopo `/zoonoses`, `module: :zoonoses`, com `before_action :require_zoonoses!`. As exceções levam nota.

| Rota | Quem |
|---|---|
| `POST /zoonoses/animals`, `PATCH /zoonoses/animals/:id`, `GET /zoonoses/animals/:id`, `POST /zoonoses/animals/search` | `team?` |
| `GET/POST /zoonoses/campaigns`, `PATCH /zoonoses/campaigns/:id`, `POST /zoonoses/campaigns/:id/{open,close}` | leitura `reader?`; escrita `coordinator?` |
| `POST /zoonoses/vaccinations`, `POST /zoonoses/vaccinations/:id/cancel`, `GET /zoonoses/vaccinations` (filtros: campanha, período, modalidade) | `team?` |
| `GET /zoonoses/demands`, `GET /zoonoses/demands/:id`, `POST /zoonoses/demands/:id/{schedule,inspect,close,decline,extend}` | `team?` |
| `GET /zoonoses/observations`, `GET /zoonoses/observations/:id`, `GET /zoonoses/observations/:id/contact`, `POST /zoonoses/observations/:id/{close,link_animal}` | `team?` |
| `GET/POST /zoonoses/leish_surveys`, `PATCH …/:id`, `POST …/:id/{open,close}` | leitura `reader?`; escrita `coordinator?` |
| `POST /zoonoses/leish_tests`, `PATCH /zoonoses/leish_tests/:id`, `GET /zoonoses/leish_tests` (filtros: inquérito, pendência) | `team?` |
| `POST /zoonoses/leish_outcomes` | `team?`; `euthanasia` exige a eutanásia já feita |
| `POST /zoonoses/collars`, `POST /zoonoses/collars/:id/remove`, `GET /zoonoses/collars?due=1` | `team?` |
| `GET/POST /zoonoses/shelter_stays`, `POST /zoonoses/shelter_stays/:id/exit` | `team?` |
| `POST /zoonoses/neuterings` | `veterinarian?` |
| `POST /zoonoses/euthanasias` | `veterinarian?`, com step-up |
| `GET/PUT /zoonoses/parameters` | leitura `reader?`; escrita `coordinator?` |
| `GET /zoonoses/reports/:name(.csv)` | `reader?` |
| `GET /zoonoses/domains` | `reader?` e o agente de campo; responde mesmo com o gate desligado, como o `/vector_control/domains` (Q32 do 35a) |

As mudanças fora de `/zoonoses`: o registro do 33 (§4), a ficha e o JSON da notificação do 25 (§5) e as rotas de produto do 35a (§3).

## 11. `field-sync` v1.4.0 (MINOR)

- **`session-response`:** `features.zoonoses` (bool, opcional). Fica fora do `required`.
- **`reference-response`:**
  - `zoonoses_domains` (opcional no topo; `$def` fechado com `additionalProperties: false`);
  - traz `species`, `sexes`, `sizes`, `animal_statuses`, `dpp_results`, `products` (só `rabies_vaccine` e `collar`, com `id`, `code`, `name` e `use`), `open_campaign` (`{ id, name, starts_on, ends_on } | null`), `parameters` (`collar_months`) e `limits`.
- **`changes-response`**, tipos novos:
  - `animal`: `id`, `version`, `household_id`, `species`, `name`, `sex`, `size`, `coat`, `breed`, `birth_estimate_on`, `status`, `neutered`, `tutor` (`{ patient_id, display_name } | null`), `community`, `last_rabies_vaccination_on`, `collar_due_on`, `has_microchip` (bool). O número do microchip não vai ao app (Q9).
  - `leish_survey`: `id`, `version`, `name`, `kind`, `city_block_ids`, `starts_on`, `ends_on`. Vêm só os inquéritos `open` que cruzam os quarteirões dos domicílios das microáreas do agente.
  - O api emite os dois tipos só para cidade com `Zoonoses::Gate.field_usable?`.
  - O animal vem pelo domicílio do escopo do agente: animal sem domicílio não vai.
  - `removals` ganha `animal`, para o animal que sai do escopo ou do estado `active`.
- **`upload-request`**, itens novos, com corpos fechados:
  - `animal`: cria ou edita (com `version` na edição). Leva `household_id`/`household_client_uuid`, `tutor_patient_id`/`tutor_client_uuid` (o paciente criado no mesmo lote, 24c) ou nenhum. O `microchip` é opcional e só vai na criação.
  - `rabies_vaccination`: `animal_id`/`animal_client_uuid`, `campaign_id`, `vaccinated_on`, `vector_product_id`, `adverse_event`, `adverse_note`. A modalidade é sempre `house`.
  - `leish_collection`: `leish_survey_id` (opcional), `animal_id`/`animal_client_uuid`, `collected_on`, `dpp_result` (opcional), `dpp_on`. A etiqueta é dada pelo api e volta no `upload-response` (`label`), como a amostra do 35a.
  - `collar`: `animal_id`/`animal_client_uuid`, `vector_product_id`, `placed_on`.
- **Recusas novas no `domains-v1.json`:**
  - `microchip_taken`, `registry_number_taken`, `already_vaccinated_in_campaign`, `campaign_not_open`, `survey_not_open`;
  - `animal_not_dog` (coleta e coleira só em cão);
  - `animal_not_active`.
- **Falta de estoque:** não é recusa. O item volta `accepted` (§3).
- **Compatibilidade (regra da Q6 do 35a):**
  - O app do 35a ignora tipo desconhecido no `changes`. O `reference` e a `session` não são validados em tempo de execução, então ele convive com uma cidade com `zoonoses` ligado e só não mostra os animais.
  - O app anterior ao 35a quebra. O gate de go-live é: nenhum aparelho com app anterior ao 35a numa cidade com `zoonoses` ligado.
  - Ligar ou desligar o `zoonoses` chega a todos os aparelhos como `reset: true`.
- **Exemplos:** cada item e cada tipo novo leva um válido e os inválidos de cada `required` e de cada `additionalProperties`.
- **Docs:** README e CHANGELOG no formato da v1.3.0. Os SHA-256 vão ao api e ao campo.

## 12. App de campo (`rotasaude/campo`)

- **Esquema local** `SCHEMA_VERSION = 3`. A migração 2→3:
  - cria as tabelas `animals` e `leish_surveys` e zera o cursor;
  - não apaga linha e mantém `last_download_ok_at`, como a 1→2.
- **Quem usa:**
  - **animal, vacinação e coleira:** o agente com `registers?` na microárea, com CBO `515140` (endemias) ou `515105` (ACS);
  - **coleta do inquérito:** só o `515140`.
  - As frases de recusa local seguem o molde `AGENT_*` do 35a.
- **Telas:**
  - **Detalhe do domicílio:** seção "Animais" (cadastrar e editar) e, em cada animal, "Vacinar" (só com campanha `open` no `reference`), "Colocar coleira" (só cão) e "Coletar sangue (LV)" (só cão, só o `515140`, com inquérito `open` que cubra o quarteirão do domicílio, ou avulsa).
  - **Coleta:** a etiqueta só aparece depois do aceite. Até lá, o app mostra "Etiqueta depois de enviar", como na Q58 do 35a.
- **Folha:** as coleiras com `collar_due_on` em até 15 dias entram em "Imóveis a visitar", com o motivo "Coleira a trocar".
- **Fila:** os quatro itens novos entram na fila serial do 24c. O `discardImpact` de `animal` leva junto a vacinação, a coleta e a coleira que dependem dele, com aviso.
- **Desligado** (`features.zoonoses` falso): as telas somem, e o que está na fila espera (Q6 do 35a).
- **Maestro:** quatro fluxos novos contra o servidor falso: `animal-register-offline`, `rabies-vaccination-offline`, `leish-collection-offline` e `collar-offline`. Os do 24c e do 35a continuam verdes.

## 13. Dashboard

O grupo "Zoonoses" fica no menu (`src/shell/modules.ts`), com a guarda `canUseZoonoses(user)` = `hasFeature(user, "zoonoses")` e um papel. Ele tem estas telas:

| Id | Tela | Quem |
|---|---|---|
| `zoo-animals` | Animais: busca (microchip, registro, tutor, domicílio), cadastro, histórico | `team?` |
| `zoo-campaigns` | Campanhas e vacinação: campanha, posto fixo, rotina, resultado semanal | leitura `reader?`; posto e rotina `team?`; campanha `coordinator?` |
| `zoo-demands` | Demandas: fila do centro (§4) | `team?` |
| `zoo-observations` | Observação do agressor (§5) | `team?` |
| `zoo-leish` | Leishmaniose: inquéritos, coleta, resultados, desfecho, coleiras a trocar | `team?`; inquérito `coordinator?` |
| `zoo-shelter` | Canil e gatil: estadas, castração, eutanásia | `team?`; castração e eutanásia `veterinarian?` |
| `zoo-reports` | Relatórios (§8) | `reader?` |
| `zoo-settings` | Parâmetros (§9) | `coordinator?` |

- **409:** a regra do 35a vale aqui: relê uma vez e não reenvia (`isZoonosesStale`).
- **Fora do grupo:**
  - detalhe da notificação do 25: seção `rabies_animal` e "Condição final do animal após a observação" (§5);
  - fila do 25: sinal `animal_outcome_pending`;
  - produtos do 35a: as duas finalidades novas (§3);
  - assuntos da ouvidoria: os reservados aparecem com cadeado.
- **Eutanásia:** o formulário mostra o veterinário logado (nome e CRMV), o motivo, a nota e a base legal, e pede o step-up. Depois do envio, a tela só lê.

## 14. wpda

- **"Nova manifestação":** nos tipos Solicitação e Denúncia, com o `zoonoses` ligado na cidade, aparece a pergunta "É sobre animais, insetos ou bichos?".
  - Com "Sim", vêm o subtipo (lista do §4) e o campo obrigatório "Onde fica o problema? (endereço do local)".
  - O aviso fixo diz: "Seu pedido vai direto ao centro de zoonoses. Não escreva seu nome nem telefone no texto."
- **"Denúncia anônima":** as mesmas duas perguntas, opcionais.
- **Desligado:** com o `zoonoses` desligado, nada muda.
- **"Minhas manifestações":** mostra o assunto e o andamento do 33, e nunca a demanda nem a vistoria.
- **e2e:** os dois caminhos, nos celulares do 33.

## 15. Maintenance

- `FEATURE_LABELS.zoonoses` = "Centro de zoonoses".
- `MISSING_LABELS.territory_registry_disabled` já existe e não muda.
- README: um parágrafo "Centro de zoonoses (módulo 35b)", com a ordem de deploy (contracts → api → dashboard, wpda e maintenance → campo) e o gate de go-live dos aparelhos (§11).

## 16. LGPD, erros e eventos

**Invariantes:**
- Tutor, adotante e paciente da notificação nunca em URL, log, evento ou resposta de lista. O tutor sai só como `{ id, display_name }` no detalhe do animal.
- O contato do tutor e o endereço do animal da notificação saem só no `GET …/observations/:id/contact`, com trilha.
- O microchip só vai no corpo (busca, cadastro) e nunca ao app (Q9).
- O `place_address` da demanda sai só no detalhe, com trilha.
- A eutanásia e a vacinação não se editam.

**Trilhas novas** (`clinical_record.viewed`-like, na tabela de trilhas do 35a): `animal.viewed`, `bite_observation.contact_viewed`, `zoonoses_demand.viewed`.

**Eventos de domínio** (`Zoonoses::Catalog::DOMAIN_EVENTS`, nessa ordem; só ids no payload; ligados em `config/initializers/domain_events.rb` com `to: []`):
1. `animal.registered`
2. `animal.changed`
3. `rabies_campaign.changed`
4. `rabies_vaccination.recorded`
5. `rabies_vaccination.cancelled`
6. `zoonoses_demand.opened`
7. `zoonoses_demand.changed`
8. `bite_observation.opened`
9. `bite_observation.closed`
10. `bite_observation.contact_viewed`
11. `leish_survey.changed`
12. `leish_test.recorded`
13. `leish_dog_outcome.recorded`
14. `collar.placed`
15. `collar.removed`
16. `shelter_stay.opened`
17. `shelter_stay.closed`
18. `neutering.recorded`
19. `euthanasia.recorded`
20. `zoonoses_parameters.changed`

- A falta de estoque reaproveita `vector_stock.shortfall_recorded`.
- Nenhum `Platform.audit` novo, e o R18 não muda.

**Spec de sigilo:** `spec/security/zoonoses_secrecy_spec.rb`, no molde do `vector_control_secrecy_spec.rb`:
- `PAYLOAD_KEYS` dos 20 eventos;
- o log sem tutor, contato, microchip nem endereço;
- as respostas de lista sem tutor;
- a recusa 4xx com só `error` e `path`;
- as trilhas;
- a varredura dos `DomainEvents.publish`.

## 17. Semente de dev e roteiro de prova

**Semente** (`lib/zoonoses_crew.rb`, chamada no `db/seeds.rb` depois do `VectorControlCrew`; só em Curitiba; fictícia e coerente):
- liga o `zoonoses` pelo mantenedor de dev;
- cria a veterinária `veterinaria@curitiba.demo`, com CBO 223305, CRMV-PR com número válido de formato e vínculo na "UVZ Curitiba";
- cria a equipe `zoonoses@curitiba.demo` (`zoonoses_team`); o `endemias@` continua como coordenação;
- dá ao `ace@` os animais da área dele;
- cria dois produtos, "Vacina antirrábica animal" (`rabies_vaccine`) e "Coleira deltametrina 4%" (`collar`), com lotes no local da UVZ;
- cria oito animais em domicílios do 24a: cinco com tutor, um comunitário e dois recolhidos no canil;
- cria a campanha `open` "Campanha antirrábica 2026" com 4 semanas de doses;
- cria uma notificação `acidente_animal_raiva` com o bloco do animal e a observação em curso;
- cria um inquérito `open` com 3 testes (1 reagente com ELISA pendente) e uma coleira vencendo;
- cria uma demanda de zoonoses (escorpião) aberta pela ouvidoria;
- não cria nenhuma eutanásia.

**Roteiro de prova** (dashboard 5221, wpda 5222 e maintenance 5177 contra o api 3061; campo no emulador contra o api 3061, depois do api e dos fronts):
1. Maintenance: liga e desliga o `zoonoses`. O dashboard esconde o grupo e o wpda esconde a pergunta.
2. Animal: busca pelo microchip, cadastro com tutor, histórico.
3. Campanha: posto fixo sem cadastro, rotina, anulação com estorno, resultado semanal em tela e em CSV.
4. wpda: solicitação de escorpião com endereço e denúncia anônima de morcego. As duas chegam à fila do centro com o extrato, e a resposta volta à ouvidoria.
5. Notificação de acidente com o bloco do animal: a observação abre, o contato é lido com trilha, e o desfecho "sadio" depois do 10º dia aparece na notificação.
6. Leishmaniose: inquérito por amostra com o tamanho calculado, coleta, DPP, ELISA, desfecho e coleira.
7. Canil: entrada, castração, eutanásia pela veterinária com step-up, e a recusa ao usuário da equipe.
8. Relatórios: os seis, com a supressão em demandas e em observações.
9. Campo: Q6 (desligar, parear o APK do 35b, religar, `reset: true`); casos sem sinal: animal novo com tutor novo no mesmo lote, vacinação, coleta com etiqueta depois do aceite, coleira com falta de estoque aceita, folha com coleira a trocar.

## 18. Entrega, testes e pendências

**Ordem:**
1. contracts T0–Tn;
2. SHA-256 ao api;
3. api inteiro;
4. tag `field-sync-v1.4.0` (com autorização);
5. dashboard, wpda e maintenance;
6. campo;
7. suíte completa pela sessão "Testes";
8. provas;
9. merges com autorização.

**Testes por repo:**
- **api:** specs de cada regra numérica do §6 e do §9, inclusive as bordas:
  - população acima da última linha da tabela;
  - ELISA sem DPP;
  - 10º dia exato;
  - segunda dose na campanha;
  - coleira trocada;
  - eutanásia por quem não é veterinário;
  - eutanásia em duplicidade.
  
  Também entram os specs de request de todas as rotas do §10, o spec de sigilo, o spec do encaminhamento automático (os três modos, sem unidade, com o interruptor desligado), o do bloco da ficha com outra doença e a suíte completa.
- **dashboard e wpda:** vitest das telas; prova no navegador.
- **campo:** Jest e Maestro.
- **contracts:** `tools/check.sh` e `check_domains.py --release`.

**Gates de go-live:**
- formato da planilha estadual da campanha antirrábica da UF piloto, conferido contra o `rabies_weekly`;
- a tabela do `leish_sample_table` conferida no manual;
- os gates de aparelho do §11 e os do 24c;
- o fluxo da demanda conferido com a ouvidoria da cidade piloto.

**Fora do 35b, para cards:**
- bloqueio de foco da raiva em 72 h;
- EATAV pelo REDCap;
- envio ao GAL;
- prontuário veterinário;
- censo animal.

## 19. Decisões (Q-n)

| # | Decisão |
|---|---|
| Q1 (coordenador, contrato) | A coordenação do centro de zoonoses é o `endemic_coordinator` do 35a, com o rótulo novo "Coordenação de vigilância ambiental" (decisão 30 da spec: os papéis novos são a coordenação e a equipe). O 35b cria só o `zoonoses_team`. |
| Q2 (coordenador, contrato) | Sem a tabela `animal_events` do resumo da spec, §3. O histórico do animal é montado pelas tabelas de cada ação, para não haver duas fontes. |
| Q3 (coordenador, contrato) | A vacina antirrábica animal e a coleira são `vector_products` do 35a, com as finalidades `rabies_vaccine` e `collar` (decisão 29). O domínio `VECTOR_CONTROL` do estoque fica utilizável com `vector_control` ou com `zoonoses`. |
| Q4 (coordenador, contrato) | No posto fixo, a dose sem cadastro leva só a espécie, sem nome de tutor digitado: o resultado da campanha é agregado (pesquisa §3.2), e nome solto seria dado pessoal sem cadastro. |
| Q5 (coordenador, contrato) | O resultado da vacinação não é suprimido: o agregado de animais não tem pessoa. A supressão vale em demandas e em observações, que nascem de pessoas. |
| Q6 (coordenador, contrato) | A denúncia anônima de zoonoses continua `kind` `report`, porque o CHECK do 33 fica como está. A solicitação identificada ou preservada usa `request`. O endereço do local é permitido nos três modos (decisão 31). |
| Q7 (coordenador, contrato) | O desfecho da observação não escreve na notificação do 25. O JSON da notificação lê a observação, e a fila ganha o sinal `animal_outcome_pending`. Assim o 25 continua dono do que grava, e o desfecho fica numa fonte só. |
| Q8 (coordenador, contrato) | O inquérito e o foco de LV usam os `city_blocks` e os anéis do 35a mesmo sem o `vector_control` ligado. O cadastro de quarteirões fica disponível a quem tem `vector_control` ou `zoonoses`. |
| Q9 (coordenador, contrato) | O número do microchip não vai ao app, só `has_microchip`. O agente cadastra o número uma vez, e a busca por ele fica no dashboard. |
| Q10 (planos: wpda D1–D2, api D1 e D11, campo D10, dashboard D-2) | **Gates por superfície.** O `public_config` do wpda ganha `zoonoses: bool` = `Zoonoses::Gate.usable?` e a ouvidoria utilizável. O api aceita `zoonoses_subtype` e `place_address` nessa mesma condição. O `features.zoonoses` da sessão do app = `Zoonoses::Gate.field_usable?`. O `canUseZoonoses` do dashboard também exige `canUseTerritory`. O `/zoonoses/domains` segue o `/vector_control/domains`: com o gate desligado, 403 `feature_disabled` (corrige o §10). |
| Q11 (planos: dashboard D-6, D-7 e Task 4, api D13–D14, contracts D1) | **Peças do 35a numa cidade só com `zoonoses`.** Produtos, quarteirões (com os anéis), o local de estoque em `vector_control_settings`, as faltas de insumo, a busca de item e o `/vector_control/domains` passam a abrir com `vector_control` **ou** `zoonoses` utilizável, pelo gate novo `VectorControl::Gate.supplies_usable?`. O resto do `/vector_control` continua só com `vector_control`. Produto de zoonoses tem uma finalidade só (`rabies_vaccine` ou `collar`), com `unit` `un` e `charge_size` 1.00, e nunca mistura finalidade com as vetoriais. O `vector_control_domains.products` do `reference` lista só os produtos vetoriais, porque o enum da v1.3.0 não muda. Os de zoonoses vão em `zoonoses_domains.products`. |
| Q12 (planos: api D20, dashboard D-3, D-12, D-13) | **Papéis.** O `zoonoses_team` é **privilegiado** (`PRIVILEGED_ROLES`): ele busca tutor por CPF e lê o contato do tutor. O mantenedor não concede o papel. `veterinarian?` = `team?` e o perfil profissional 223305 com CRMV e vínculo na UVZ. A veterinária precisa do papel para usar o dashboard, e a tela a reconhece por `/professionals/me`. Ler o contato da observação grava trilha e não pede step-up a mais. O PUT dos parâmetros segue o PUT do 35a. |
| Q13 (planos: api D2–D4, D6–D7, dashboard D-11) | **Encaminhamento automático.** O encaminhamento e a classificação gravam o ator `system` (`created_by_user_id` passa a aceitar nulo, com CHECK de ator). O **extrato é só o texto do cidadão**: o subtipo e o `place_address` ficam em campos próprios, então o limite de 4000 não estoura. A lista da fila mostra subtipo, prazo, cor e estado; o extrato e o endereço saem só no detalhe, com trilha. Se a UVZ não tiver ninguém com `team?`, a manifestação fica `in_analysis` já classificada, como sem unidade. Os usuários do centro respondem, recusam e prorrogam o encaminhamento da UVZ pelas rotas `/zoonoses/demands` (`via_zoonoses`), sem precisar do `ombudsman_responder`. O encaminhamento feito à mão pela equipe da ouvidoria para a UVZ também cria a demanda: o subtipo vem do `code` do assunto, ou fica `other`. |
| Q14 (planos: api D5 e D22, dashboard D-17 e D-18) | **Assuntos.** `Ombudsman::Subjects.ensure_zoonoses!` roda sob demanda (primeiro registro com subtipo, listagem de assuntos com o gate ligado), porque ligar o interruptor não tem gancho na cidade. Se a cidade já tiver um assunto pai "Zoonoses" sem `code`, ele é adotado. O JSON do assunto passa a trazer `code`. No balcão da ouvidoria, o registro aceita as duas chaves só com `kind` `request` ou `report`, e a tela de registro do dashboard entra no plano do dashboard (Task 14). |
| Q15 (plano do wpda, D3–D6) | **wpda.** "Minhas manifestações" segue o 33 e não mostra o assunto (corrige o §14). Com subtipo, o `place_address` é obrigatório nos três modos, inclusive na anônima (corrige o §4 e o §14). Os rótulos dos subtipos ficam fixos no wpda, com a lista do §4: renomear o assunto no dashboard não muda o canal. O cidadão não vê de volta o `place_address`. |
| Q16 (planos: api D8–D10, dashboard D-9 e D-10) | **Ficha do 25 e observação.** A seção `rabies_animal` usa os tipos de texto da ficha. Data vai como `"AAAA-MM-DD"` e booleano como `"true"`/`"false"`, ambos validados. O erro é `invalid_form`, com `path` `form.<campo>` (corrige o §5), e o PATCH continua substituindo a ficha inteira. Sai o estado `cancelled` da observação: o `dismissed` do 25 só parte de `possible`, e a observação só nasce de `notified` ou `investigating`. Os estados ficam `open` e `closed`. A observação nasce `place` `home` e vira `shelter` na entrada `observation` do canil. O sinal da fila do 25 é `animal_outcome_pending: boolean`. |
| Q17 (planos: api D12, dashboard D-8) | **Faltas de insumo fora do boletim.** `vector_stock_shortfalls` ganha `source` (`inspection` \| `vaccination` \| `collar`), as referências anuláveis `rabies_vaccination_id` e `collar_id`, com CHECK de exatamente uma referência por `source`, e `occurred_on`. O `visit_on` passa a aceitar nulo fora do `inspection`. A tela de faltas do 35a mostra a origem. |
| Q18 (planos: api D15–D17, dashboard D-15) | **Leishmaniose.** A ligação vai do desfecho à eutanásia: `leish_dog_outcomes.euthanasia_id` é obrigatório com `decision` `euthanasia`, e sai `euthanasias.leish_dog_outcome_id`. Sai o parâmetro `leish_expected_prevalence`: a tabela do manual é a de 2% e é ela que se edita. População acima da última linha usa a amostra da última linha. Sugestão de `dog_population`: os cães ativos cadastrados nos quarteirões; com zero cadastrados, `quarteirões × dogs_per_block`. O foco sugere os anéis do `VectorControl::Rings` do 35a. |
| Q19 (plano do api D18; plano do campo D13) | **Cobertura antirrábica.** O `rabies_weekly` traz as duas coberturas: sobre a meta e sobre a estimativa de população (spec 16), esta só quando houver estimativa. A segunda dose na mesma campanha é barrada só pelo api: o tablet não sabe a campanha da última dose, e isso fica documentado. |
| Q20 (planos: contracts D3–D5 e D7–D11, campo D4–D6 e D11) | **Formas do `field-sync` v1.4.0.** O item `animal` leva o corpo inteiro. `animal_id` vai na edição, com `version`. `microchip` e `registry_number` vão só na criação. `status` vai só na edição, sem voltar a `active`. Tutor é proibido com `community: true`. O nascimento vai em `birth_estimate_month` (`"AAAA-MM"`). O `accepted` do `animal` traz `version`, e a edição vencida volta `stale_version` com o `current_version`, como no 24c, sem mescla. Campos obrigatórios: vacinação (animal, `campaign_id`, `vaccinated_on`, `vector_product_id`, `adverse_event`, com `adverse_note` só se `adverse_event`); coleta (animal, `collected_on`, e `dpp_result` com `dpp_on` sempre juntos); coleira (animal, `vector_product_id`, `placed_on`). Entram os códigos `date_out_of_range` e `household_not_in_survey`, com `path`. Rótulos, chaves de `limits`, o tipo de inquérito e a finalidade do produto vão na seção `zoonoses` do `domains-v1.json`. `last_rabies_vaccination_on` e `collar_due_on` mudam sem `version` nova: o api reemite o animal e o app sobrescreve (Q29 do 35a). Os exemplos inválidos são um por chave dos corpos novos, gerados em laço, e inválidos dirigidos na sessão e na referência. |
| Q21 (planos: campo D1–D3, contracts D2 e D6, api D21) | **Inquérito e etiqueta no tablet.** O `leish_survey` traz `household_ids`, os domicílios do escopo do agente nos quarteirões do inquérito, no lugar de `city_block_ids`. Assim o tablet não depende do quarteirão do 35a. O inquérito fechado e o animal que sai do escopo ou do `active` saem por `removals`, com `reason` `out_of_scope` (`TOMBSTONE_ENTITY_TYPES` ganha `animal` e `leish_survey`). A etiqueta da coleta é feita pelo **api** e volta no `accepted` e no `duplicate` como `label` (hex maiúsculo; corrige o "como a amostra do 35a" do §11). O app guarda a coleta em `local_leish_collections` até o aceite. |
| Q22 (plano do campo, D7–D8) | **Tutor no tablet.** O `tutor_client_uuid` cita o paciente criado no mesmo lote pelo item `patient` do 24c. Como só o ACS cria morador, o 515140 escolhe o tutor entre os moradores que o tablet já tem. Animal com domicílio pode ficar sem tutor e sem `community` (tutor não identificado). `community: true` quer dizer só que ninguém responde pelo animal (corrige o §2). |
| Q23 (plano do campo, D9) | **Coleira a trocar no tablet.** A coleira não entra em "Imóveis a visitar", que é do 515140 com `vector_control` e tem um motivo por imóvel. Ela ganha a lista própria "Coleiras a trocar" na área de animais, para quem usa as telas de zoonoses no tablet (corrige o §6 e o §12). |
| Q24 (plano do campo, D12 e D14) | **Frases.** A fonte é `ZOONOSES_ERRORS` em `apps/dashboard/src/lib/zoonosesErrors.ts`, e o campo copia a lista. A Task 5 do campo espera esse commit do dashboard. O "Q6" do §12 e do §17 é a Q6 do 35a. |
| Q25 (plano do dashboard, D-1, D-4, D-5, D-14, D-16) | **Formas e buscas do dashboard.** Os tipos da Task 1 do dashboard valem como proposta. O api da execução fixa as formas, e a Task 0 do dashboard confere o plano contra o api real, como no 33. A busca de tutor e de adotante é `POST /zoonoses/person_lookup` (CPF ou CNS no corpo, com trilha). O domicílio do animal sai da resposta dessa busca (o domicílio do cadastro individual ativo da pessoa, com um resumo do endereço): não há lista livre de domicílios. Os filtros dos relatórios são `from`/`to`, `campaign_id` e `survey_id`. Os rótulos do menu ficam "Relatórios de zoonoses" e "Parâmetros de zoonoses". |
| Q26 (plano do api, D19) | **Trilhas.** Seguem o molde do `transmission_block_case.address_viewed` do 35a: são eventos de domínio. `animal.viewed` e `zoonoses_demand.viewed` entram no fim da lista do §16, com os números 21 e 22, e entram no `PAYLOAD_KEYS` do spec de sigilo. |
| Q27 (plano do maintenance, DM1–DM3) | **Maintenance.** O parágrafo do 35a no README passa a dizer "Coordenação de vigilância ambiental" (Q1). A descrição do `CATALOG` fica sem o prefixo: "Animal com tutor, vacinação antirrábica, demandas da ouvidoria, observação do agressor, leishmaniose canina e canil". |
| Q28 (plano do dashboard, Task 10) | **Sugestão de quarteirões do inquérito.** O api expõe `POST /zoonoses/leish_surveys/suggest`, só para `coordinator?`. O corpo traz `{ kind, neighborhood_id, center_city_block_id?, rings? }`. A resposta é `{ city_block_ids, dog_population_suggested, target_dogs }`. No `focus`, os quarteirões vêm dos anéis do `VectorControl::Rings` em volta do central. Nos outros tipos, vêm os quarteirões ativos do bairro. A população e a amostra seguem a Q18. A rota não grava nada. |
| Q29 (plano do campo, N1) | **Tutor novo no mesmo lote.** O `tutor_client_uuid` aceita o `client_uuid` de um item `patient` **ou** de um `individual_registration` com `new_patient` do mesmo lote, como o `patient_client_uuid` da Q26 do 24c. O api resolve o paciente criado por qualquer um dos dois. Fora disso, o item volta `rejected` `tutor_not_found` (código novo no `domains-v1.json`, com `path` `tutor_client_uuid`). O caso "tutor novo no mesmo lote" fica na prova. |
| Q30 (api Task 2, questão A) | **Perfil da veterinária sem autoridade clínica.** O `zoonoses_team` entra em `Professional::PROFILE_ROLES` e `LINK_ROLES`, como os papéis próprios do CAPS (P29), e o perfil profissional e o vínculo na UVZ saem por ele, sem `health_professional`. O perfil é **opcional** para esse papel: `/professionals/pending` não lista membro do `zoonoses_team` sem perfil, porque técnico e balcão não têm CBO. `veterinarian?` continua sendo `team?` mais o perfil 223305/CRMV com vínculo ativo na UVZ. Na semente, a veterinária tem só `zoonoses_team` e o perfil. |
| Q31 (api Task 2, questão B) | **Situação do animal.** As transições são: `active` → `missing` \| `transferred` \| `deceased`; `missing` → `active` \| `deceased`; `transferred` e `deceased` são finais. Isso corrige o §2: o animal desaparecido que reaparece volta a `active`. A eutanásia e a saída `death` do canil exigem animal `active` ou `missing`; nos outros casos, 409 `animal_not_active`. A entrada no canil de animal `missing` o devolve a `active`, na mesma transação. O trigger segue estas transições. |
