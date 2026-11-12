# Módulo 24b — Contratos entre apps

**Spec:** `superpowers/specs/2026-11-12-module-24b-home-visit-collective-activity-design.md` (decisões 1A–7A do usuário e 1–23 do coordenador).
**ADR:** `adr/0052.md` · **Pesquisa:** `pesquisa/2026-11-12-visita-e-coletiva-24b.md` · **Módulo:** `modulos/24--domiciliar-e-coletivo.md`.

Este contrato é a **fonte única** dos planos do contracts, do api, do dashboard, do wpda e do maintenance para o 24b. Quando um plano divergir dele, a divergência vai para a seção final "Divergências propostas ao contrato" do próprio plano. O coordenador reconcilia aqui, na §16.

Convenções herdadas (24a §1–§14, 23, 33): erros `{ "error" }`, com `path` nas recusas de validação; step-up `401 mfa_required`; interruptor desligado `403 feature_disabled`; JSON em toda escrita por cookie; listas `{ items, next_before }` com cursor `before` opaco; filtros repetidos sempre com `[]`; registro de outro escopo por id → 404, lista de escopo alheio → 403; leitura nominal com trilha e sem polling; contagens sem trilha; texto longo recusado (nunca cortado); dado sensível só no corpo, nunca na URL nem no log; todo `Data`/`Struct` com dado pessoal sem dado em `inspect`/`to_s`/`as_json`.

**Bases** (origin/main em 2026-11-12):

| Repo | Commit |
|---|---|
| api | `d36aef3` |
| dashboard | `4524082` |
| wpda | `7482397` |
| maintenance | `9269b55` |
| contracts | `b0d1f87` |

**Valores fixos:**

| Item | Valor |
|---|---|
| Porta do api | 3057 |
| Vite do dashboard | 5218 |
| Vite do wpda | 5219 |
| maintenance | 5177, com o serviço do compose parado durante a prova |
| Migrações da cidade | a partir de `20261112900001` (a última na main é `20261111900002`); nenhuma de plataforma |
| Branch (todos os repos) | `feat/mod-24b-home-visits` |
| Worktree do api | `apps/api/.claude/mod24b`, com `config/master.key` copiada |
| Worktrees dos fronts e do contracts | `rota-saude/.worktrees/<repo>-mod24b` |
| Testes | só `bin/test api -w mod24b …` e `bin/test <app> -w <repo>-mod24b`; suíte completa pela sessão "Testes" |
| Tag do contracts | `field-sync-v1.0.0` (domínio novo `field-sync/`) |

**Mudança transversal.** O 24b toca migração de cidade, `CityEncryption::CITY_KEYED_TARGETS`, `Platform::Features`, `Ledi::FichaTypes`/`FichaSources`, `CareLines::{Catalog,Sources,Rules,Evaluate}`, `Timeline::KINDS`, a retenção da exclusão LGPD e uma autenticação nova (token de aparelho). A suíte completa do api é obrigatória antes do merge.

## 1. Nomes no código

**Interruptores** (no fim do `CATALOG` de `Platform::Features`; os specs que fixam a ordem são atualizados):

| Chave | Descrição | `requires` |
|---|---|---|
| `home_visits` | "Folha do dia, ficha LEDI de visita, quem visitar e sincronização do app de campo" (Q24) | `%w[feature:territory_registry]` |
| `collective_activities` | "Atividade coletiva com ficha LEDI, sem PSE" | `%w[feature:clinical_record]` |

- Rótulos curtos (maintenance e dashboard): `home_visits` → "Visita domiciliar e territorial"; `collective_activities` → "Atividade coletiva".
- `home_visits` vai ao cidadão em `GET /citizen/session` (o wpda não depende dele); `collective_activities` entra em `CITIZEN_HIDDEN` (a atividade nunca chega ao cidadão) (Q24).
- Gates: `HomeVisits::Gate.usable?(city)` e `CollectiveActivities::Gate.usable?(city)`. `HomeVisits::Gate` **não** copia a exceção de perfil do `TerritoryRegistry::Gate` (que ignora `ibge_code_missing`/`uf_missing`): sem IBGE ou UF, as rotas do 24a respondem e as da visita não (C1, Q1). A sessão da equipe só lista `home_visits` quando utilizável (Q16). Rotas: `403 feature_disabled`. Desligado, o que foi gravado fica; o job de fechamento roda mesmo assim (§2).

**Quem registra** (`config/ledi/registration_cbos.yml` ganha duas chaves; conferido em `vendor/ledi/8.7.0/docs/cbo.html`):

| Chave | Tabela LEDI | Códigos |
|---|---|---|
| `home_visit` | Tabela 7 (MIVDT) | 7: `515105` ACS, `515310` agente de ação social, `515140` ACE, `515130` agente indígena de saneamento, `515125` agente indígena de saúde, `322255` técnico em ACS, `515120` visitador sanitário |
| `collective_activity` | Tabela 5 (MIAC) | 117 códigos, **strings** (115 numéricos e os alfanuméricos `2231F9` e `1312C1`; nenhum campo supõe CBO numérico, Q1), copiados da tabela (inclui `352210` agente de saúde pública, enfermeiros, médicos, dentistas, ACS e ACE) |

`TerritoryRegistry::Access.registrar_cbo?(cbo, ficha:)` aceita `ficha: :home_visit | :collective_activity` pelo YAML, sem mudança de código (Q1). O acesso do 24b mora em `HomeVisits::Access` e `CollectiveActivities::Access`.

**Acesso à visita** (sempre: papel `health_professional`, vínculo ativo `HealthTeamMember` na equipe, equipe 70 ou 76):

| Quem | Registra visita | Lê visitas e "quem visitar" | Aparelho de campo |
|---|---|---|---|
| ACS (`515105`) | na microárea em que é `acs_professional_id`, ou fora de área | as suas microáreas | sim; baixa as suas microáreas |
| Outro CBO da Tabela 7 | nas microáreas atribuídas a ele (6A) | as microáreas atribuídas | sim; baixa as atribuídas |
| Enfermeiro (`2235*`) | não (`403 cbo_not_allowed`) | a equipe toda | não |
| Médico (`2251*`, `2252*`, `2253*`) | não | a equipe toda | não |
| `municipal_admin` | não | não | só revoga |

- `TerritoryRegistry::Catalog::REGISTRAR_CBO_PREFIXES` é a fonte de "enfermeiro e médico".
- O TACS (`322255`) não é ACS para a microárea: entra pela atribuição como os outros.
- Atribuição de microárea (6A): só o enfermeiro da equipe atribui e encerra, com step-up. Uma microárea pode ter vários atribuídos; o ACS responsável não precisa de atribuição.
- **Leitura do 24a pelo CBO de campo que não é ACS (Q11):** nas microáreas atribuídas a ele, só leitura, pelas rotas do 24a (`GET /territory/households?micro_area_id=`, `GET /territory/households/:id`, `GET /territory/registrations?household_id=`), com o escopo da atribuição e a trilha do 24a (`access: "territory_registry"`). Microárea não atribuída → `403 out_of_scope`; domicílio fora dela por id → 404. A escrita do 24a (rotas do dashboard e itens `household`/`individual_registration` do app) continua só do ACS: os outros recebem `403 cbo_not_allowed` (no app, item `rejected cbo_not_allowed`). As buscas de pessoa do 24a continuam recusando esses CBOs (C19).

**Tabelas novas da cidade** (todas `id: :uuid`, `created_at`; "cifrado" = `CityEncryption`, entra em `CITY_KEYED_TARGETS`):

- **`micro_area_field_assignments`:** `micro_area_id`, `professional_id`, `assigned_by_user_id`, `ended_at`, `ended_by_user_id`. Único ativo por (`micro_area_id`, `professional_id`) `WHERE ended_at IS NULL`. Nunca apagada.
- **`home_visit_days`** (folha):
  - `professional_id`, `user_id`, `team_id`, `cbo` (CBO do vínculo na abertura, 6 caracteres), `day` (date, dia da cidade), `sequence` (int ≥ 1; 1 = folha principal);
  - `state` (`open` | `closed`), `closed_at`, `closed_by` (`user` | `job` | `full`, Q6), `closed_by_user_id`;
  - único por (`professional_id`, `team_id`, `day`, `sequence`); no máximo uma `open` por (`professional_id`, `team_id`, `day`) (índice parcial);
  - CHECK: `(state = 'closed') = (closed_at IS NOT NULL) = (closed_by IS NOT NULL)`; `closed_by_user_id` só com `closed_by = 'user'`.
- **`home_visits`:**
  - `home_visit_day_id`, `client_uuid` (uuid, único), `occurred_at` (timestamptz), `shift` (1, 2, 3; derivado de `occurred_at`, §3);
  - `household_id`, `family_id`, `patient_id` (todos opcionais), `subject` (`resident` | `family_responsible` | `none`), `property_type` (texto, códigos de `TerritoryRegistry::Catalog::PROPERTY_TYPES`), `micro_area_id`, `out_of_area` (bool);
  - `outcome` (1 Realizada, 2 Recusada, 3 Ausente), `has_reason` (bool, sem cifra), `reasons` (cifrado; lista de inteiros), `shared` (bool), `measurements` (cifrado; objeto do §3), `note` (cifrado, até 500);
  - `withdrawn_at`, `withdrawn_by_user_id`, `withdrawal_reason` (cifrado, 10–500);
  - `origin` (`dashboard` | `field_app`), `field_device_id` (só em `field_app`), `recorded_by_user_id`;
  - `patient_id` é sempre a pessoa cujo documento vai na ficha: o morador em `resident`, o responsável da família em `family_responsible`;
  - CHECK: `(micro_area_id IS NULL) = out_of_area`; `subject = 'none'` ⇔ `patient_id IS NULL`; `subject = 'family_responsible'` ⇒ `family_id IS NOT NULL`; `outcome IN (2,3)` ⇒ `NOT has_reason`; `(withdrawn_at IS NULL) = (withdrawn_by_user_id IS NULL)`; imóvel não residencial (`2 3 4 5 6 12 13`) ⇒ `patient_id IS NULL AND family_id IS NULL`;
  - índices: (`patient_id`, `occurred_at`) parcial `WHERE withdrawn_at IS NULL AND outcome = 1`; (`micro_area_id`, `occurred_at`); (`home_visit_day_id`).
- **`collective_activities`:**
  - `client_uuid` (único), `team_id`, `health_unit_id` (unidade da equipe, do CNES do cabeçalho), `professional_id` e `cbo` (quem registra, o cabeçalho), `recorded_by_user_id`, `held_on` (date), `shift` (1, 2, 3, informado);
  - `activity_type` (1–7), `place_health_unit_id` ou `place_text` (até 250; CHECK: exatamente um);
  - `participants_count` (1–999), `altered_count` (0–999, ≤ `participants_count`);
  - `meeting_topics`, `target_audience`, `health_topics`, `health_practices` (cifrados; listas de inteiros), `procedure_code` (cifrado; SIGTAP de 10 dígitos);
  - `mistaken_at`, `mistaken_by_user_id`, `mistaken_reason` (cifrado, 10–500);
  - `origin`, `field_device_id`.
- **`collective_activity_professionals`:** `collective_activity_id`, `professional_id`, `cbo`. Único por (atividade, profissional). Não inclui quem registra (já é o cabeçalho).
- **`collective_activity_participants`:** `collective_activity_id`, `patient_id`, `altered_evaluation` (cifrado; bool), `measurements` (cifrado; `{ weight_kg, height_cm }`), `smoking` (cifrado; `{ quit, dropped_out }`). Único por (atividade, paciente).
- **`field_devices`:** `user_id`, `professional_id`, `name` (1–60), `token_digest` (SHA-256 hex, 64, único), `last_used_at`, `revoked_at`, `revoked_by_user_id`, `revoked_reason` (`user` | `admin` | `membership_ended` | `user_disabled`). No máximo 3 ativos por usuário (`422 device_limit`).
- **`field_sync_receipts`:** `client_uuid` (único), `field_device_id`, `user_id`, `item_type`, `result_type`, `result_id`. Só itens aceitos. É o livro de idempotência do envio (§9).
- **`field_sync_tombstones`:** `entity_type` (`household` | `registration`), `entity_id`, `micro_area_id`, `reason` (`moved` | `erased`), índice (`micro_area_id`, `created_at`). `moved` é gravada na mesma transação em que um domicílio troca de microárea ou um cadastro individual passa a um domicílio de outra microárea. Saída do cadastro (mudança, óbito) e domicílio inativo **não** são remoção: seguem como item, com o estado. `erased` fica reservado, sem produtor no 24b (a exclusão LGPD retém quem tem cadastro; §13) (Q9).

**Migrações da cidade** (irreversíveis, como as do 24a e do 33: o `down` levanta `ActiveRecord::IrreversibleMigration` e o `up` termina executando `db/city_triggers.sql`; sem dado; Q2):

| Timestamp | Conteúdo |
|---|---|
| `20261112900001_add_home_visits.rb` | `micro_area_field_assignments`, `home_visit_days`, `home_visits` |
| `20261112900002_add_collective_activities.rb` | `collective_activities`, `collective_activity_professionals`, `collective_activity_participants` |
| `20261112900003_add_field_devices.rb` | `field_devices`, `field_sync_receipts`, `field_sync_tombstones` |

Triggers em `db/city_triggers.sql` (§13).

**Mudanças em código existente:**

| Onde | Mudança |
|---|---|
| `Ledi::FichaTypes::REGISTRY` | `visita_domiciliar` → `{ code: 8, klass: "Br::Gov::Saude::Esusab::Ras::Visitadomiciliar::FichaVisitaDomiciliarMasterThrift", uuid_field: :uuidFicha }`; `atividade_coletiva` → `{ code: 6, klass: "Br::Gov::Saude::Esusab::Ras::Atividadecoletiva::FichaAtividadeColetivaThrift", uuid_field: :uuidFicha }` |
| `Ledi::FichaSources::REGISTRY` | `"HomeVisitDay" => "Ledi::HomeVisitDayFicha"`, `"CollectiveActivity" => "Ledi::CollectiveActivityFicha"` (mesmo contrato de `generate`, `retry!`, `regenerate` do `Ledi::RegistrationFicha`) |
| `CareLines` | §12 |
| `Timeline` | §12 |
| `Citizens::RequestErasure` | §13 |
| `config/recurring.yml` | `HomeVisits::CloseDaysJob` "every day at 3:20am America/Sao_Paulo" |
| `vendor/ledi/8.7.0/docs` | `dicionario-fvd.html` e `dicionario-fac.html` com SHA-256 e data no `SOURCE` (task 1) |

**Eventos de domínio** (`DomainEvents.publish`, só ids): `home_visit.recorded` (`home_visit_id`, `home_visit_day_id`, `user_id`), `home_visit.withdrawn` (idem), `home_visit_day.closed` (`home_visit_day_id`, `closed_by`), `collective_activity.recorded` (`collective_activity_id`, `user_id`), `collective_activity.marked_mistaken` (idem), `field_device.registered` (`field_device_id`, `user_id`), `field_device.revoked` (`field_device_id`, `user_id`, `reason`), `micro_area.field_assignment_changed` (`micro_area_id`, `professional_id`, `user_id`, `ended`: bool). Um spec de sigilo fixa a lista exata e confere que nenhum payload traz motivo, medida, observação, tema, prática, nome, CPF ou CNS.

**`Platform.audit`:** o 24b não cria nome novo (a troca de interruptor é `city.feature_changed`; a versão nova do catálogo-base é `care_line.template_changed`). Se um plano precisar de um, o nome entra em `R18_PLATFORM_EVENT_NAMES` (`spec/events/platform_event_payload_guard_spec.rb`) no mesmo commit.

## 2. Estados e ações

**Folha (`home_visit_days.state`):** `open` → `closed`. Nunca volta.

| Ação | De | Para | Quem | Regras |
|---|---|---|---|---|
| abrir (implícito) | — | `open` | a primeira visita do (profissional, equipe, dia) | sem folha aberta para o dia: cria `sequence = max + 1` (1 na primeira) |
| `close` | `open` | `closed` | o dono da folha | corpo `{ expected_state: "open" }`; outro estado → `409 day_changed`; gera a ficha na transação (§10) |
| fechamento pelo job | `open` | `closed` | `HomeVisits::CloseDaysJob` | folhas com `day < hoje` (dia da cidade); roda com `home_visits` ligado ou não (decisão 22); `closed_by = job` |

- **Folha complementar:** visita para um dia cuja folha está `closed` abre a próxima `sequence`. A 100ª visita não retirada de um (profissional, equipe, dia) fecha a folha aberta cheia na mesma transação (`closed_by = full`, ficha gerada) e abre a próxima `sequence` (Q6). A complementar fecha pelo dono, pelo job ou cheia, como a principal.
- **Prazo do SIAPS no fechamento:** se `Ledi::Deadline.on(competência do day) < hoje`, a folha fecha e a ficha fica "não gerada" com `siaps_deadline_passed` (decisão 22). Nunca é descartada.
- **Folha sem visita válida** (todas retiradas): fecha sem ficha e sem "não gerada"; a situação da ficha é `empty`.

**Visita:** `recorded` | `withdrawn` (de `withdrawn_at`).

| Ação | De | Para | Quem | Regras |
|---|---|---|---|---|
| registrar | — | `recorded` | §1 | §3; folha escolhida pelo api |
| `withdraw` | `recorded` | `withdrawn` | o dono da folha | corpo `{ expected_state: "recorded", reason }`; já retirada → `409 visit_changed`; folha fechada → `409 day_closed`; `reason` 10–500 |

A retirada tira a visita das fontes na hora (decisão 23). A linha não muda além dos três campos de retirada. Para corrigir, registra-se de novo.

**Atividade coletiva:** `recorded` | `mistaken` (de `mistaken_at`).

| Ação | De | Para | Quem | Regras |
|---|---|---|---|---|
| registrar | — | `recorded` | membro da equipe com CBO da Tabela 5 | §4; gera a ficha na hora |
| `mark_mistaken` | `recorded` | `mistaken` | quem registrou | corpo `{ expected_state: "recorded", reason }`; outro → `409 activity_changed`; tira a atividade das telas e das fontes; **não** desfaz a ficha já enfileirada (fica pendência do §15) |

**Aparelho:** ativo | inativo | revogado. Revogar é final. Não há evento de fim de vínculo no api (Q7): o token é conferido em **toda** requisição do app contra usuário ativo, papel `health_professional` e vínculo ativo em equipe 70/76 com CBO da Tabela 7; sem isso, a requisição é recusada com o erro uniforme `401 device_revoked` e o aparelho é revogado nesse momento (`membership_ended` ou `user_disabled`). Antes dessa requisição, as listas do §9.1 mostram o aparelho como `inactive` (calculado na leitura).

**Atribuição de microárea:** ativa | encerrada (`ended_at`). Encerrar é final; reatribuir cria linha nova.

## 3. Regras da visita

Dia de referência = data local da cidade de `occurred_at`.

- **Data:** `occurred_at` ≤ agora; e `Ledi::Deadline.on(competência) > hoje` (a ficha precisa sair antes do prazo; o job roda de madrugada). Fora disso → `422 visit_date_out_of_range` (`path` `occurred_at`).
- **Turno:** derivado, nunca no corpo: antes de 12h = 1 Manhã; antes de 18h = 2 Tarde; senão 3 Noite.
- **Imóvel e microárea:**
  - com `household_id`: herda `property_type` e `micro_area_id` do domicílio (o corpo não manda os dois; mandar → `422`, `path` `property_type`); domicílio inativo → `422 household_inactive` (Q21);
  - microárea inativa (herdada ou escolhida) → `422 micro_area_inactive` (Q21);
  - `property_type` é texto (`"1"`…`"99"`, como o 24a grava e devolve) no dashboard e no app; o inteiro só na ficha (Q26);
  - sem domicílio: `property_type` obrigatório e `micro_area_id` ou `out_of_area: true`;
  - `out_of_area` só para o ACS (`403 out_of_scope` para os outros).
- **Pessoa:**
  - imóvel não residencial (`2 3 4 5 6 12 13`): nenhum `patient_id`, `family_id` nem `new_patient` → `422 non_residential_with_person`; `subject = none`;
  - residencial (`1 7 8 9 10 11 99`): `subject` é `resident` (`patient_id` ou `new_patient`), `family_responsible` (`family_id`; o api põe o responsável em `patient_id`) ou `none` (visita ao imóvel sem pessoa);
  - `resident` com domicílio: o paciente tem cadastro ativo nesse domicílio ou não tem cadastro ativo (inclusive o recém-registrado do §5); cadastro ativo em outro domicílio → `422 patient_in_other_household`;
  - `family_responsible`: a família é do domicílio e não está `moved_out` (`422 family_moved_out`, Q21); responsável que saiu → `422 family_without_responsible`.
- **Desfecho × motivos × medidas:**
  - `outcome` obrigatório (1, 2, 3);
  - `outcome` 2 ou 3: `reasons` vazio (`422 reasons_forbidden`) e sem `measurements` (`422 measurements_forbidden`);
  - não residencial: só os motivos `1 27 28 31 34 35 36 37` (`422 reason_not_allowed`, `path` `reasons[i]`) e sem medidas;
  - sexo masculino (`male`): motivos 5 e 6 proibidos (`422 reason_not_allowed`);
  - `has_reason = reasons.any?`, gravado pelo api.
- **Medidas** (`measurements`, todas opcionais; limites do dicionário, a confirmar na task 1):

| Chave | Faixa | Campo LEDI |
|---|---|---|
| `weight_kg` | 0,5–500, 3 casas | `pesoAcompanhamentoNutricional` |
| `height_cm` | 20–250, 1 casa | `alturaAcompanhamentoNutricional` |
| `systolic`, `diastolic` | 0–999, inteiros; um exige o outro | `pressaoSistolica`, `pressaoDiastolica` |
| `temperature_c` | 20–45, 1 casa | `temperatura` |
| `glucose_mg_dl`, `glucose_type` | 0–800; um exige o outro | `glicemia`, `tipoGlicemia` |

- **Observação:** `note` até 500 (code points depois do `strip`); nunca vai ao LEDI, à linha do tempo nem ao cidadão.
- **Microárea do ACS:** a microárea da visita (herdada ou escolhida) precisa estar no escopo de quem registra (§1), senão `403 out_of_scope`.
- **Travas:** paciente → domicílio → folha → visita.

## 4. Regras da atividade coletiva

Regras cruzadas da ficha (a confirmar na task 1 contra o `dicionario-fac.html`); o api recusa com `422 invalid_collective_activity` e o `path`, e o dashboard espelha:

| Campo | Tipos 1–3 (reuniões) | Tipo 4 | Tipo 5 | Tipo 6 | Tipo 7 |
|---|---|---|---|---|---|
| `meeting_topics` | obrigatório | proibido | proibido | proibido | proibido |
| `target_audience` | proibido | obrigatório | obrigatório | obrigatório | obrigatório |
| `health_topics` | proibido, salvo com o tema de reunião 6 | obrigatório | obrigatório | opcional | obrigatório |
| `health_practices` | proibido | proibido | opcional | obrigatório | proibido |
| `participants` | opcional | opcional | **obrigatório** | **obrigatório** | opcional |

- Práticas 2, 9 e 33 só no tipo 6. `procedure_code` (SIGTAP, 10 dígitos) obrigatório com a prática 30 e proibido sem ela.
- `smoking` do participante só com uma das práticas 25–28.
- `participants_count` ≥ número de participantes; `altered_count` = participantes com `altered_evaluation` quando a lista vem, senão ≤ `participants_count`.
- Local: `place_health_unit_id` (unidade ativa com CNES) **ou** `place_text`; nunca INEP.
- `held_on` ≤ hoje e `Ledi::Deadline.on(competência) > hoje` (`422 activity_date_out_of_range`).
- Tema 18 (Semana saúde na escola) não é aceito (`422`, `path` `health_topics[i]`). PSE sempre `false` na ficha.
- `professionals`: lista opcional de `{ professional_id }` com vínculo ativo em qualquer equipe da cidade; o api grava o CBO do vínculo; sem CNS válido → `422 professional_without_cns`.
- Participante: `patient_id` ou `new_patient` (§5); repetido → `422 duplicate_participant`.
- Uma gravação é a atividade inteira; sem rascunho, sem edição.

## 5. Registro rápido (7A)

Na visita (`subject: resident`) e no participante, `new_patient` no lugar de `patient_id`:

```json
{ "new_patient": { "full_name", "birth_date", "sex", "mother_name"?, "cpf"? | "no_cpf_reason", "cns"? },
  "duplicates_reviewed": false }
```

- Cria pelo `Patients::CreateReviewed.call(new_patient:, reviewed:, by:)`, na transação da gravação (savepoint): a recusa do paciente desfaz o item inteiro.
- `duplicates_reviewed` acompanha cada `new_patient`: na visita, no corpo; na atividade, **dentro de cada participante** (Q15).
- Sem `duplicates_reviewed: true` e com `match_key` existente → `409 duplicate_suspected` com `path` (`new_patient` na visita; `participants[i].new_patient` na atividade) e `candidates: [{ patient_id, display_name, birth_date, sex, has_cpf, has_cns }]` (máximo 10, trilha com os ids mostrados; Q8). Nos itens do 24a no app, o candidato traz também `registration_id` (uuid ou `null`, L31; Q26). A tela oferece abrir o existente ou "é outra pessoa" (reenvia com `true`).
- `409 cpf_taken` / `409 cns_taken` (com `path`): a tela oferece a busca por documento.
- O registro rápido **não** cria cadastro individual; a pessoa aparece como "sem cadastro" na validade do 24a.
- Buscas da tela (corpo, nunca query; trilha; 30 por minuto por usuário):
  - `POST /home_visits/people/lookup` e `POST /collective_activities/people/lookup` — `{ cpf }` ou `{ cns }` → `{ patient_id, display_name, birth_date, sex, has_cpf, cns_masked }`, no padrão do `Vaccination::PersonLookupsController` (27a): ninguém → `404 person_not_found`; par validado sem paciente vira paciente (`Patients::LookupByIdentifier`) e volta na mesma forma; par só declarado → `409 citizen_not_verified`; documento inválido → `422 invalid_cpf` / `invalid_cns` (Q8);
  - `POST /home_visits/people/duplicates` e `POST /collective_activities/people/duplicates` — `{ full_name, birth_date, mother_name? }` → `{ candidates: [...] }` (`Patients::Duplicates`).

## 6. API da equipe: visitas (dashboard)

Base `/home_visits`, sessão de equipe, gate `home_visits`, `HomeVisits::Access`.

**Formas:**

`<home_visit>` (nominal; trilha): `{ id, client_uuid, day_id, occurred_at, shift, origin, property_type, micro_area: { id, code } | null, out_of_area, household: { id, address_short } | null, family_id | null, subject, patient: { id, display_name, has_cpf, cns_masked } | null, outcome, reasons: [int], shared, measurements | null, state: "recorded" | "withdrawn", withdrawn: { at, reason: string | null, by: { name } } | null, recorded_by: { name }, can_withdraw }`. `note` só no `GET /home_visits/:id`; `withdrawn.reason` preenchido só para o dono e o enfermeiro/médico da equipe, `null` para os outros leitores (Q17). `can_withdraw` = dono, `recorded` e folha `open` (Q14; o api continua recusando).

`<home_visit_day>`: `{ id, professional: { id, name }, team: { id, ine, name }, day, sequence, state, closed_at, closed_by, visits_count, withdrawn_count, own, ficha: { status, reasons: [code] }, blockers: [{ code, visit_id | null }] }`. `own` = a folha é de quem lê (Q14).
- `ficha.status`: `open` (folha aberta) | `empty` | `not_exported` (fechada com visita válida e exportação não utilizável: sem ficha e sem falha; Q5) | `not_generated` (com `reasons` do §10) | o status do `ledi_outbox` (`pending`, `sending`, `accepted`, `rejected`, `failed`, `correction_pending`).
- `blockers` (só em `open`): o que faria a ficha sair "não gerada" se a folha fechasse agora, pelos mesmos códigos do §10 (ex.: `professional_without_cns`, `patient_without_identifier` com o `visit_id`).

**Rotas:**

| Rota | Quem | Notas |
|---|---|---|
| `GET /home_visits/context` | Tabela 7, enfermeiro, médico | `{ today, min_date, teams: [{ id, ine, name, cbo, can_register, can_assign, can_register_device, reads_whole_team, out_of_area_allowed, micro_areas: [{ id, code, name, scope: "responsible" \| "assigned" \| "team" }], professionals: [{ id, name, cbo }] }], domains }`. `min_date` = o primeiro dia aceito pelo prazo do SIAPS. `can_assign` = enfermeiro da equipe; `can_register_device` = Tabela 7 com vínculo em equipe 70/76; `professionals` = os da Tabela 7 com vínculo ativo, só quando `reads_whole_team` (vazio para os outros) (Q14). `domains` = a forma `home_visit` do §7 (Q13) |
| `GET /home_visits/days/today?team_id=` | quem registra | a folha aberta de hoje (`<home_visit_day>` com `visits: [<home_visit> sem note]`) ou `null`; trilha só quando há visita com pessoa. É atalho: a tela pode usar `GET /home_visits/days?from=<hoje>&to=<hoje>` + `GET /home_visits/days/:id` para mostrar também as folhas já fechadas de hoje (Q16) |
| `GET /home_visits/days?team_id=&professional_id=&from=&to=&state[]=&before=` | dono (as suas); enfermeiro e médico (a equipe) | `{ items: [<home_visit_day>], next_before }`, sem pessoa, sem trilha |
| `GET /home_visits/days/:id` | idem | `<home_visit_day>` + `visits` (trilha) |
| `POST /home_visits/days/:id/close` | dono | `{ expected_state }` → 200 `<home_visit_day>` |
| `POST /home_visits` | §1 | `{ client_uuid, team_id, occurred_at, household_id?, micro_area_id?, out_of_area?, property_type?, subject, patient_id? \| new_patient?, family_id?, duplicates_reviewed?, outcome, reasons, shared, measurements?, note? }` → 201 `{ visit: <home_visit>, day: <home_visit_day> }`. Mesmo `client_uuid` de visita já gravada → 200 com a gravada (idempotente; o dashboard gera o uuid por formulário) |
| `GET /home_visits/:id` | dono; enfermeiro e médico da equipe; quem lê a microárea | `<home_visit>` com `note` (trilha) |
| `POST /home_visits/:id/withdraw` | dono | `{ expected_state, reason }` → 200 `<home_visit>` |
| `GET /home_visits?team_id=&micro_area_id=&professional_id=&from=&to=&outcome[]=&before=` | quem lê a microárea ou a equipe | "Visitas da equipe": `{ items: [<home_visit> sem note], next_before }` (trilha com os ids) |
| `GET /home_visits/count?team_id=&micro_area_id=&from=&to=` | idem | `{ done, refused, absent, withdrawn }` (sem trilha) |
| `GET /home_visits/assignments?team_id=` | enfermeiro da equipe | `{ items: [{ id, micro_area: { id, code, name }, professional: { id, name, cbo }, assigned_at, assigned_by: { name } }] }`, só ativas; `candidates: [{ professional_id, name, cbo }]` (Tabela 7 sem ser ACS responsável) |
| `POST /home_visits/assignments` | enfermeiro, step-up | `{ micro_area_id, professional_id }` → 201; `409 already_assigned`; `422 cbo_not_allowed` (fora da Tabela 7 ou ACS responsável pela própria) |
| `POST /home_visits/assignments/:id/end` | enfermeiro, step-up | 200; já encerrada → `409 assignment_already_ended` (Q21). O escopo do aparelho muda e o próximo download responde `reset: true` (§9.3; Q9) |

## 7. API da equipe: atividade coletiva (dashboard)

Base `/collective_activities`, gate `collective_activities`, `CollectiveActivities::Access` (membro ativo da equipe com CBO da Tabela 5; enfermeiro e médico da equipe leem).

`<collective_activity>` (nominal com participantes; trilha): `{ id, client_uuid, team: { id, ine, name }, unit: { id, name }, held_on, shift, activity_type, place: { health_unit: { id, name } } | { text }, participants_count, altered_count, meeting_topics, target_audience, health_topics, health_practices, procedure_code | null, professionals: [{ id, name, cbo }], participants: [{ patient: { id, display_name, has_cpf, cns_masked }, altered_evaluation, measurements, smoking }], recorded_by: { name }, state: "recorded" | "mistaken", mistaken: { at, reason: string | null, by: { name } } | null, ficha: { status, reasons }, can_mark_mistaken }`. Na resposta, as quatro listas sempre vêm (Q15); `mistaken.reason` como `withdrawn.reason` (Q17); `can_mark_mistaken` = quem registrou, `recorded` (Q14).

| Rota | Quem | Notas |
|---|---|---|
| `GET /collective_activities/options` | Tabela 5 | `{ today, min_date, teams: [{ id, ine, name, cbo, can_register }], units: [{ id, name, has_cnes }], professionals: [{ id, name, cbo, has_cns, team_ids }], domains }`. `domains` = a forma `collective_activity` do §7, com as regras do §4 em dado (o dashboard não as copia à mão; Q13) |
| `POST /collective_activities` | Tabela 5 | corpo explícito (Q15): `{ client_uuid, team_id, held_on, shift, activity_type, place_health_unit_id? \| place_text?, participants_count, altered_count, meeting_topics?, target_audience?, health_topics?, health_practices?, procedure_code?, professionals?: [{ professional_id }], participants?: [{ patient_id \| patient_client_uuid (só no app) \| new_patient, duplicates_reviewed?, altered_evaluation?, measurements?, smoking? }] }`; exatamente um dos dois de local; lista ausente = vazia (o dashboard manda `[]` no campo proibido pelo tipo) → 201 `<collective_activity>`. Mesmo `client_uuid` → 200 com a gravada |
| `GET /collective_activities?team_id=&from=&to=&activity_type[]=&state[]=&before=` | equipe | `{ items: [ <collective_activity> sem participants nem temas ], next_before }`, sem trilha |
| `GET /collective_activities/:id` | equipe | `<collective_activity>` (trilha com os pacientes) |
| `POST /collective_activities/:id/mark_mistaken` | quem registrou | `{ expected_state, reason }` → 200 |

Atividade `mistaken` sai da lista por padrão (`state[]=mistaken` a mostra).

**Forma dos `domains` (Q13)** — a de `context.domains` e `options.domains`, servida pelo api a partir do mesmo YAML da task 1 que preenche `domains-v1.json`. O `domains-v1.json` (do app) mantém a estrutura do plano do contracts, mais rica (erros, limites, paciente); um spec do api confere que os códigos e as regras das duas formas são os mesmos:
- `home_visit`: `motivo_visita: [{ code, label, group, group_label }]`, `desfecho`, `turno`, `tipo_glicemia`, `tipo_de_imovel` (todos `[{ code, label }]`), `non_residential_property_types`, `non_residential_reasons`, `male_forbidden_reasons`, `measurement_ranges: { <chave>: { min, max, decimals } }`;
- `collective_activity`: `activity_types`, `meeting_topics`, `target_audience`, `health_topics` (sem o 18), `health_practices` (todos `[{ code, label }]`) e `rules: { by_type: { "<tipo>": { meeting_topics | target_audience | health_topics | health_practices | participants: "required" | "forbidden" | "optional" } }, health_topics_with_meeting_topic: [6], practices_only_in_type: [{ type: 6, practices: [2, 9, 33] }], procedure_practices: [30], smoking_practices: [25, 26, 27, 28], participant_measurements: { weight_kg, height_cm } }`.
- O esquema não confere as relações; elas ficam em `rules` e no api, com os códigos do §13 (Q26).

## 8. Quem visitar

`GET /home_visits/who_to_visit?micro_area_id=&reason[]=&before=&limit=` (quem lê a microárea; nominal, trilha `clinical_record.viewed` com `access: "home_visits"` e os ids mostrados).

```json
{ "items": [ { "patient_id", "display_name", "age", "household": { "id", "address_short" } | null,
               "family_id" | null, "micro_area": { "id", "code" }, "last_visit_on" | null,
               "reasons": [ { "code": "care_line", "care_line": "<nome da linha>", "practice": "<label>",
                              "status": "overdue" | "due_soon", "due_on" },
                            { "code": "registration_due", "status": "due_soon" | "expired", "due_on" },
                            { "code": "never_visited" } ] } ],
  "next_before" }
```

- População: cadastro individual ativo (24a) com domicílio na microárea; óbito fora.
- `care_line`: inscrição ativa com prática de fonte `acs_visit` em `overdue` ou `due_soon` (avaliação do 23 com `home_visits: true`); só com `care_lines` utilizável. Nunca CID, problema ou condição.
- `registration_due`: `TerritoryRegistry::Validity` em `due_soon` (≤ 60 dias) ou `expired`.
- `never_visited`: nenhuma visita `recorded` com `outcome = 1` para o `patient_id`.
- Só entra quem tem ao menos um motivo. Ordem `display_name`, `patient_id`; cursor = `person_ref` opaco. `limit` 1–100 (padrão 50).
- `GET /home_visits/who_to_visit/count?micro_area_id=` → `{ care_line, registration_due, never_visited, people }` (sem trilha).

## 9. Aparelho de campo e sincronização v1

### 9.1 Aparelho (dashboard)

| Rota | Quem | Notas |
|---|---|---|
| `GET /field_devices` | o usuário | `{ items: [{ id, name, created_at, last_used_at, status: "active" \| "inactive" \| "revoked", revoked_at, revoked_reason }] }`; `inactive` = não revogado, mas o dono perdeu o vínculo da Tabela 7 ou foi desativado (Q7) |
| `POST /field_devices` | Tabela 7 com vínculo ativo em equipe 70/76; **step-up** | `{ name }` → 201 `{ device, token, qr_payload }`. `token` só nesta resposta; `qr_payload` = `"rotasaude-field:v1:" + base64url(JSON { host, token })`. Enfermeiro e médico → `403 cbo_not_allowed` |
| `POST /field_devices/:id/revoke` | dono | 200; sem step-up; já revogado → `409 device_already_revoked` (Q21) |
| `GET /field_devices/admin?user_id=` | `municipal_admin` | todos os aparelhos da cidade, com `user: { id, name }` e o mesmo `status` |
| `POST /field_devices/admin/:id/revoke` | `municipal_admin`, **step-up** | 200 |

- Token: 32 bytes aleatórios em base64url com prefixo `rsf_`. O banco guarda só `token_digest = SHA-256`; busca pelo índice único, nunca comparação do texto.
- A tela mostra **só** o QR, uma vez, e não guarda o token; o texto do token nunca vai ao DOM (quem perder o QR revoga e cadastra de novo; Q20). O token nunca vai em URL, log ou evento.

### 9.2 Autenticação do app

- Host da cidade, sem cookie, `Authorization: Bearer rsf_…`. Ausente ou desconhecido → `401 invalid_token`; revogado → `401 device_revoked` (com `WWW-Authenticate: Bearer`).
- Em cada requisição: usuário ativo, papel `health_professional`, vínculo ativo em equipe 70/76 com CBO da Tabela 7; senão revoga (`membership_ended` ou `user_disabled`) e responde o erro uniforme `401 device_revoked` (sem dizer o motivo; Q7). Não há gancho em `Cnes::Apply` nem em `User#deactivate!`.
- Ordem na borda (padrão do `Hl7::BaseController`; Q1): `rate_limit` por `token_digest` (por IP sem token), 120 por minuto → `429 too_many_requests`; depois o gate `home_visits` (`403 feature_disabled`); depois o token. `last_used_at` atualizado no máximo uma vez por minuto.
- Sem sessão: `Current.user` fica nil; todo comando recebe `by:` explícito (nada de `Current` novo; Q1).
- Base `/field/v1`, só JSON (415 fora disso), corpo até 2 MB (`413 payload_too_large`).

### 9.3 Rotas

**`GET /field/v1/session`** → `{ contract: "field-sync-v1", server_time, user: { id, name }, professional: { id, cbo, has_cns }, teams: [{ id, ine, name, micro_areas: [{ id, code, name, scope: "responsible" | "assigned" }], out_of_area_allowed }], features: { home_visits, collective_activities, territory_registry }, min_date }`. Sem trilha. No aparelho, `scope` só tem `responsible` e `assigned` (Q26).

**`POST /field/v1/uploads`** — `{ items: [{ client_uuid, type, body }] }`, de 1 a 50 itens.

| `type` | `body` | Resultado (`id`) |
|---|---|---|
| `home_visit` | corpo do `POST /home_visits` sem `client_uuid` (vai no item) e sem `shift`; `patient_client_uuid` aceito no lugar de `patient_id`, `household_client_uuid` no lugar de `household_id` (Q27); `occurred_at` **com** fuso | visita |
| `home_visit_withdrawal` | `{ visit_client_uuid \| visit_id, reason }` | visita |
| `collective_activity` | corpo do `POST /collective_activities`; `patient_client_uuid` aceito nos participantes | atividade |
| `patient` | `{ new_patient, duplicates_reviewed? }` (§5) | paciente |
| `household` | corpo do 24a (`POST`/`PATCH /territory/households`), com `household_id` e `version` na atualização | domicílio |
| `individual_registration` | corpo do 24a (`POST`/`PATCH /territory/registrations`), com `registration_id` e `version` na atualização; `household_client_uuid` e `patient_client_uuid` aceitos no lugar de `household_id` e `patient_id` (Q27) | cadastro |

- Itens processados **em ordem**, cada um na sua transação: um recusado não derruba o lote; um item pode citar o `client_uuid` de um `patient` ou `household` aceito antes (no mesmo lote ou em outro); citação de item não aceito → `rejected` `not_found` (Q27).
- Família não é item no v1 (Q27): offline, o app não cria família; visita `family_responsible` e cadastro com `family_id` só com família já baixada. Um item `family` entra num MINOR quando o 24c pedir.
- `household` e `individual_registration` só do ACS (regra do 24a): os outros CBOs da Tabela 7 recebem `rejected cbo_not_allowed`, e o 24c não oferece cadastro a eles (Q11).
- Idempotência: `client_uuid` já em `field_sync_receipts` → `duplicate` com o mesmo `id`, sem regravar. Mesmo `client_uuid` com outro `type` → `rejected` `client_uuid_conflict`. Recusado e `stale_version` não vão ao livro: o reenvio com o mesmo `client_uuid` é processado de novo.
- Visita e atividade gravam `origin = field_app` e `field_device_id`. A visita de dia já fechado vai para a folha complementar (§2).
- `household` e `individual_registration` exigem `territory_registry` utilizável e as regras do 24a; `collective_activity` exige `collective_activities` (senão o item volta `rejected` `feature_disabled`).

Resposta 200 (o envelope inválido é `422 invalid_request` com `path`):

```json
{ "results": [
  { "client_uuid", "status": "accepted" | "duplicate", "type", "id" },
  { "client_uuid", "status": "rejected", "type", "error", "path"?, "candidates"? },
  { "client_uuid", "status": "stale_version", "type", "id", "current_version", "current": <household> | <registration> }
] }
```

`error` vem do §13; `candidates` só em `duplicate_suspected`.

**`GET /field/v1/changes?cursor=&limit=`** — download incremental do escopo do aparelho (§1: microáreas do ACS ou atribuídas).

```json
{ "reset": false,
  "items": [ { "type": "micro_area", "id", "body": <micro_area> },
             { "type": "household", "id", "version", "body": <household com families e living_conditions> },
             { "type": "registration", "id", "version", "body": <registration> } ],
  "removals": [ { "type": "household" | "registration", "id", "reason": "moved" | "erased" | "out_of_scope" } ],   // erased e out_of_scope reservados, nunca emitidos no 24b (Q9)
  "who_to_visit": null | { "generated_at", "micro_areas": [ { "micro_area_id", "people": [ <item do §8> ] } ] },
  "next_cursor", "has_more" }
```

- Formas `<micro_area>`, `<household>` e `<registration>` = as do 24a (§2 e acréscimos L4–L31).
- Ordem por (`updated_at`, `id`) com corte em `agora − 2 s`; `limit` 1–500 (padrão 200).
- `cursor` opaco (base64url de `{ v: 1, scope_digest, since, after_id }`), sem dado pessoal. Inválido → `422 invalid_cursor`. Sem `cursor` = carga completa.
- Escopo mudou desde o cursor (atribuição nova ou encerrada, ACS trocado na microárea) → `reset: true` e a carga recomeça do zero: o app apaga o que baixou (nunca o que falta enviar) e aplica as páginas.
- `removals` vêm de `field_sync_tombstones` das microáreas do escopo depois de `since` (só `moved` no 24b; Q9). Saída do cadastro e domicílio inativo chegam como item, com `exit`/`status`.
- `who_to_visit` vem só na última página, sempre completo: é `null` se e só se `has_more: true`; o app substitui o que tinha. `next_cursor` vem sempre, inclusive na última página (o app o guarda para a próxima sincronização) (Q26).
- **Trilha por página:** `clinical_record.viewed` com `access: "field_sync"`, `user_id`, `field_device_id`, `household_ids` e `patient_ids` da página. Página sem pessoa não grava.

### 9.4 Contrato em `rotasaude/contracts`

Domínio novo `field-sync/` (tag `field-sync-v1.0.0`; o README do contracts ganha a linha):

| Arquivo | O que é |
|---|---|
| `README.md`, `CHANGELOG.md` | regras do §9 em texto curto |
| `session-response-v1.json` | `GET /field/v1/session` |
| `upload-request-v1.json`, `upload-response-v1.json` | `POST /field/v1/uploads` |
| `changes-response-v1.json` | `GET /field/v1/changes` |
| `domains-v1.json` | os domínios do §10 e do §11 e as regras do §3 e do §4, em dado |
| `examples/` + `examples/manifest.json` | um válido por tipo de item e por resultado (`accepted`, `duplicate`, `rejected`, `stale_version`), página com `removals`, página com `reset`, e inválidos |

- JSON Schema 2020-12, fechado (`additionalProperties: false`), salvo as cinco seções de dicionário do 24a (`living_conditions`, `identification`, `sociodemographic`, `health_conditions`, `street_situation`: `type: object`, a fonte é o `registration_dictionaries.yml` do api). `$id` em URL, como os outros domínios (`https://rota-saude/schemas/field-sync/<arquivo>`), e `title` = `rotasaude.field_sync.<nome>.v1` (Q26).
- Instantes com segundos e fuso (`Z` ou `±hh:mm`); o `occurred_at` enviado pelo app **exige** fuso. Códigos do 24a (`property_type`, `no_cpf_reason`) em texto (Q26).
- `error` por padrão (`^[a-z][a-z0-9_]*$`), não por `enum`; a lista conhecida fica em `domains-v1.json` (`errors.http`, `errors.item`). Os erros HTTP seguem `{ "error" }` sem esquema próprio (Q26).
- O api guarda cópia em `config/field_sync/` presa por SHA-256 num spec, como o `config/lab/lab-report-v1.json`, e testa as respostas contra ela.
- `domains-v1.json` é preenchido da tabela da task 1 (§10, §11); a tag só sai depois dela. O api publica a tabela neste contrato; quem escreve `domains-v1.json` é a sessão do contracts (Task 4 do plano do contracts), que devolve os SHA-256 novos ao api (Q13).

## 10. Ficha LEDI de visita (tipo 8)

`Ledi::HomeVisitDayFicha`, fonte `HomeVisitDay`, uma ficha por folha fechada, gerada **dentro** da transação do fechamento, só com a exportação utilizável (mesmo `generate` do 24a, inclusive `city_profile_incomplete` com a exportação aceitando).

**Mestre e cabeçalho:**

| Campo LEDI | Fonte |
|---|---|
| `uuidFicha` | gerado pelo `Ledi::Enqueue` (CNES na frente, até 44) |
| `tpCdsOrigem` | 3 |
| `headerTransport.profissionalCNS` | `professionals.cns` do dono da folha |
| `headerTransport.cboCodigo_2002` | `home_visit_days.cbo` |
| `headerTransport.cnes` | CNES da unidade da equipe |
| `headerTransport.ine` | INE da equipe |
| `headerTransport.dataAtendimento` | `day` às 00h no fuso da cidade, em epoch ms, pelo `Ledi::Fichas::RegistrationHeader` (a confirmar na task 1; Q3) |
| `headerTransport.codigoIbgeMunicipio` | `CityProfile.ibge_code` |
| `visitasDomiciliares` | as visitas `recorded` da folha, em ordem de `occurred_at`, 1 a 99 |

**Visita (`FichaVisitaDomiciliarChildThrift`):**

| # | Campo LEDI | Fonte | Regra |
|---|---|---|---|
| 1 | `turno` | `shift` | |
| 2 | `numProntuario` | nunca enviado | |
| 3 / 14 | `cnsCidadao` / `cpfCidadao` | paciente de `patient_id` | CPF se houver; senão CNS; nunca os dois; nenhum em `subject = none` |
| 24 / 25 | `stCidadaoNaoPossuiCpf` / `justificativaCidadaoNaoPossuiCpf` | paciente | sem CPF: `true` e `no_cpf_reason`; vai com o CNS quando houver |
| 4 / 5 | `dtNascimento` / `sexo` | paciente (`SEX_TO_LEDI`) | só com pessoa; nunca em não residencial |
| 6 | `statusVisitaCompartilhadaOutroProfissional` | `shared` | |
| 7 | `motivosVisita` | `reasons` | |
| 8 | `desfecho` | `outcome` | |
| 9 / 10 | `microArea` / `stForaArea` | código da microárea / `out_of_area` | `microArea` nulo com fora de área |
| 11 | `tipoDeImovel` | `property_type` (inteiro) | |
| 12, 13, 15–19 | medidas | `measurements` | §3 |
| 20 / 21 | `latitude` / `longitude` | nunca enviados (24c) | |
| 22 | `uuidOrigemCadastroDomiciliar` | `Ledi::HouseholdRegistrationFicha.originating_uuid(<versão atual do domicílio>)` | sem domicílio ou sem ficha utilizável: ausente, nunca inventado |
| 23 | `ivcf` | nunca enviado | |

Os nomes que valem são os do IDL (`justificativaCidadaoNaoPossuiCpf`, `tipoGlicemia` = 18, `glicemia` = 19).

**"Não gerada" (lista fechada, `LediGenerationFailure.reason_codes`):** `unit_without_cnes`, `professional_without_cns`, `cbo_not_in_ledi_table`, `city_profile_incomplete`, `siaps_deadline_passed`, `patient_without_identifier` (sem CPF, sem CNS e sem `no_cpf_reason`), `citizen_without_birth_date`, `citizen_without_sex`. Os de nascimento e sexo reusam os que já existem em `LediGenerationFailure::REASONS` (com rótulo no dashboard); o único código novo da ficha 8 é `siaps_deadline_passed` (Q4). Uma visita com problema segura a folha inteira; os `blockers` do §6 mostram qual antes do fechamento.

Ficha recusada: "Reenviar" regera da folha (uuid novo, `replaces_outbox_id`), como o 24a, **na Produção** (`municipal_admin`, step-up); "Visitas da equipe" e o detalhe da atividade só mostram a situação (Q12). A folha não muda. `siaps_deadline_passed` nunca se resolve: a Produção não oferece "Gerar de novo" para ele (Q12).

**Domínios** (do `dicionario.html` vendorizado; códigos inteiros; conferir na task 1 contra o `dicionario-fvd.html` e o `gen-rb`):

| Domínio | Códigos |
|---|---|
| `motivo_visita` | TIPO_VISITA 1, 29 · BUSCA_ATIVA 2, 3, 4, 30 · ACOMPANHAMENTO 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 32, 33, 19, 20, 21, 22, 23, 24, 38 · CONTROLE_AMBIENTAL_E_VETORIAL 34, 35, 36, 37 · OUTROS 25, 27, 31, 28 (rótulos na pesquisa §2; 21 e 30 têm o mesmo rótulo no dicionário, a tela mostra o grupo) |
| `desfecho` | 1 Realizada, 2 Recusada, 3 Ausente |
| `tipo_de_imovel` | o do 24a (contrato do 24a, §15); não residencial = 2, 3, 4, 5, 6, 12, 13 |
| `turno` | 1 Manhã, 2 Tarde, 3 Noite |
| `tipo_glicemia` | 0 Jejum, 1 Pós-prandial, 2 Pré-prandial, 3 Não especificado (`tipoGlicemiaCapilar` do `dicionario.html`; vale salvo o `dicionario-fvd.html` dizer outro na task 1; Q3) |
| `justificativa_nao_possui_cpf` | o do 24a (contrato do 24a, §15) |

**A confirmar na prova técnica (task 1):** `dataAtendimento` da folha; obrigatoriedade do CNS com `stCidadaoNaoPossuiCpf = true` na MIVDT; quando nascimento e sexo são obrigatórios em residencial sem pessoa; o domínio de `tipoGlicemia`; mais de uma ficha do mesmo profissional no mesmo dia (gate de go-live). A task 1 publica o resultado como acréscimo nesta seção (e em `domains-v1.json`) antes das tasks do dashboard.

## 11. Ficha LEDI de atividade coletiva (tipo 6)

`Ledi::CollectiveActivityFicha`, fonte `CollectiveActivity`, gerada na transação da gravação.

| # IDL | Campo LEDI | Fonte |
|---|---|---|
| 1 | `uuidFicha` | `Ledi::Enqueue` |
| 24 | `headerTransport` | quem registrou: CNS, `cbo`, CNES de `health_unit_id`, INE da equipe, `held_on` (00h local, ms), IBGE |
| 4 / 21 | `outraLocalidade` / `cnesLocalAtividade` | `place_text` / CNES de `place_health_unit_id` (um só) |
| 7 | `inep` | nunca |
| 11 / 12 | `numParticipantes` / `numAvaliacoesAlteradas` | `participants_count` / `altered_count` |
| 13 | `profissionais` | `collective_activity_professionals` (`cnsProfissional`, `codigoCbo2002`) |
| 14 | `atividadeTipo` | `activity_type` |
| 15 / 16 / 25 / 26 | temas, público, temas para saúde, práticas | listas do §4 |
| 18 | `participantes` | `collective_activity_participants` |
| 19 | `tbCdsOrigem` | 3 |
| 22 | `procedimento` | `procedure_code` |
| 23 | `turno` | `shift` |
| 27 / 28 | `pseEducacao` / `pseSaude` | sempre `false`, sempre enviados |

**Participante:** `cpfParticipante` se houver CPF; senão `stCidadaoNaoPossuiCpf = true`, `justificativaCidadaoNaoPossuiCpf` e `cnsParticipante`; `dataNascimento`, `sexo`, `avaliacaoAlterada`, `peso`, `altura`, `cessouHabitoFumar`, `abandonouGrupo`.
- Tipos 5 e 6: participante sem CPF e sem CNS → ficha "não gerada" `participant_without_identifier`.
- Outros tipos: participante sem CPF e sem CNS fica fora da lista da ficha e segue contado em `numParticipantes`; a tela avisa antes de gravar.

**"Não gerada" (lista fechada):** `unit_without_cnes`, `professional_without_cns`, `cbo_not_in_ledi_table`, `city_profile_incomplete`, `siaps_deadline_passed`, `participant_without_identifier`, `participant_without_birth_date`, `participant_without_sex` (Q4), `participant_professional_without_cns`, `place_without_cnes`. Entram em `LediGenerationFailure::REASONS` (com o `siaps_deadline_passed`, seis códigos novos no módulo).

**Domínios** (códigos inteiros; conferir na task 1 contra o `dicionario-fac.html`):

| Domínio | Códigos |
|---|---|
| `tipo_atividade_coletiva` | 1 Reunião de equipe, 2 Reunião com outras equipes de saúde, 3 Reunião intersetorial/conselho local/controle social, 4 Educação em saúde, 5 Atendimento em grupo, 6 Avaliação/procedimento coletivo, 7 Mobilização social |
| `temas_para_reuniao` | 1, 2, 3, 4, 5, 6, 7, 8 |
| `temas_para_saude` | 29, 19, 1, 4, 5, 7, 8, 10, 13, 14, 15, 6, 16, 17, 31, 32, 21 (18 fora: exige PSE) |
| `praticas_em_saude` | 20, 2, 23, 9, 11, 25, 26, 27, 28, 34, 22, 3, 24, 12, 30, 33 |
| `publico_alvo` | 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 12, 13, 14, 15, 16, 17, 18 "Outros", 19 e 20 (adolescente em medida socioeducativa, meio aberto e fechado); o 74 não existe no `dicionario.html` (vale salvo o `dicionario-fac.html` dizer outro na task 1; Q3) |

Rótulos na pesquisa §3. **A confirmar na task 1:** cada regra cruzada do §4 (em especial `health_topics` nos tipos 1–3 com o tema 6 e no tipo 6), o "Outros" do público-alvo (Q3), e se `smoking` vale também na prática 34.

## 12. Linhas de cuidado e linha do tempo

**Linhas de cuidado (23):**
- `CareLines::Catalog.untracked?(source, vaccination: false, home_visits: false)`: `acs_visit` é não rastreada só sem `home_visits`. Quem chama passa `home_visits: HomeVisits::Gate.usable?(Current.city)`, lido fora de transação (padrão do `vaccination:` do 29): `Evaluation`, `Preview`, `CitizenView`, o `CareLines::BaseController` (quadro, inscrições, sugestões, prévia), o `CitizenApi::CareController` e "quem visitar" (Q1).
- `CareLines::Sources.home_visits!(result, ids, from)`, uma consulta: visitas com `withdrawn_at IS NULL`, `outcome = 1`, `has_reason`, imóvel residencial, folha com `cbo IN ('515105','322255')`, `patient_id` na lista, `occurred_at ≥ from`. Evento `Event.new(on: data local, cbo: folha.cbo, codes: [])` na fonte `acs_visit`.
- `blood_pressure` e `weight_height` ganham as visitas `recorded` com `outcome = 1` e a medida (pressão com os dois valores; peso e altura juntos), de qualquer CBO da Tabela 7, **só com `home_visits` utilizável**, pelo mesmo `home_visits:` do `acs_visit` (uma leitura; desligado, a visita não entra em fonte nenhuma; Q10).
- `CareLines::Rules::PRACTICE_KEYS` ganha `min_gap_days` (inteiro 1–365, opcional). No `Evaluate`, os eventos contam em ordem de data e só entra o que dista ≥ `min_gap_days` do último que entrou.
- **Onde se edita (Q19):** o formulário de prática do catálogo-base no maintenance ganha o campo `min_gap_days`; o editor da linha própria da cidade no dashboard **preserva** e mostra o atributo ("intervalo mínimo de N dias entre registros"), sem oferecê-lo nesta entrega.
- **Catálogo-base, versão nova:** `config/care_lines/catalog-base.yml` ganha `min_gap_days: 30` na `c5_d` e `citizen_label: "Visitas do agente de saúde"` nas práticas `acs_visit` (`c2_d`, `c3_e`, `c3_j`, `c4_d`, `c5_d`, `c6_c`). Cidade nova recebe pela carga; cidade existente recebe pelo maintenance (versão nova de `child_c2`, `pregnancy_c3`, `diabetes_c4`, `hypertension_c5`, `elderly_c6`, `care_line.template_changed`), passo do runbook.
- `CareLines::Catalog::CHANNELS` (`home_visit` do contato da busca ativa) não vira visita nem conta.

**Linha do tempo (20):**
- `Timeline::KINDS` ganha `home_visit` e `collective_activity`; `Timeline::Query::SOURCES` ganha `Sources::HomeVisitSource` e `Sources::CollectiveActivitySource` (Q1).
- `collective_activity` entra em `Timeline::Query::PROFESSIONAL_ONLY_KINDS`; `home_visit` entra em `Timeline::Query::CITIZEN_FILTERED_KINDS` e **não** entra em `DECLARED_KINDS` (só o par validado vê a visita, como vacina, viagem e urgência; Q23).
- `PecTimelinesController::KINDS` exclui os dois (o PEC já recebe a ficha).
- **Profissional:** `home_visit` com todos os desfechos e a retirada: `title` "Visita domiciliar" (ou "Visita ao imóvel" sem pessoa), `summary` com desfecho, rótulos dos motivos e medidas, `professional` = dono da folha, `status` `recorded` | `withdrawn`, `detail_ref` `{ type: "home_visit", id }` (o dashboard não abre o detalhe nesta entrega: o `GET /home_visits/:id` é só de quem lê a microárea ou a equipe; Q18). Nunca a observação. `collective_activity`: `title` = rótulo do tipo, `summary` = "Participou" (e "avaliação alterada" quando houver), sem temas; `mistaken` some.
- **Cidadão:** só `outcome = 1` e não retirada; a entrada é exatamente `{ id, kind: "home_visit", occurred_at, unit: null, professional_name: null, title: "Visita do agente de saúde", summary: null, status: null, document: null }`, sem `status_label` nem `return_guidance` (Q22). `occurred_at` do cidadão é o **meio-dia local** do dia da visita (precedente da passagem do TFD, 30): a hora real não sai do servidor (Q22). Recusa, ausência, retirada, motivo, medida e observação nunca.
- Com `home_visits` desligado, a visita realizada continua no histórico do cidadão (registro retido, como a viagem do 30); no acompanhamento a fonte vira não rastreada e a prática sai (Q23).
- Pessoa da entrada = `patient_id` (o responsável, na visita à família); os outros moradores não veem a visita (Q23).

**wpda:**
- `GET /citizen/history`: o wpda não muda o pedido (vai sem `kinds[]`, que o api lê como todos); a visita chega porque `home_visit` entra em `Timeline::KINDS` e passa por `Timeline::Query.citizen_kinds` (Q22). O cartão mostra o rótulo de tipo "Visita" (o wpda não tem ícones), o título como constante do canal e **só o dia**, no fuso da cidade. Defesa do cliente: a entrada some se vier com `status` não nulo ou sem data (Q22).
- `GET /citizen/care`: sem rota nova; com `home_visits` utilizável, a prática de visita passa a aparecer (rastreada) com o `citizen_label` e o status, com as frases de sempre do `careItemView` (Pendente do usuário na Q25).

## 13. LGPD, cifra, imutabilidade, eventos e códigos de erro

- **Cifra:** `home_visits.{reasons, measurements, note, withdrawal_reason}`, `collective_activities.{meeting_topics, target_audience, health_topics, health_practices, procedure_code, mistaken_reason}`, `collective_activity_participants.{altered_evaluation, measurements, smoking}` em `CITY_KEYED_TARGETS`. `has_reason`, `outcome`, `property_type` e `activity_type` ficam sem cifra (regra e filtro).
- **Imutabilidade** (`db/city_triggers.sql`, exceção `rota.reencrypting`; nenhuma tabela do módulo aceita DELETE):
  - `home_visit_days`: só `open → closed` (com `closed_at`, `closed_by`, `closed_by_user_id`); fechada não muda;
  - `home_visits`: só os três campos de retirada, de nulo a preenchido, e só com a folha `open`;
  - `collective_activities`: só os três campos de engano, de nulo a preenchido;
  - `collective_activity_professionals`, `collective_activity_participants`, `field_sync_receipts`, `field_sync_tombstones`: `rota_append_only`;
  - `micro_area_field_assignments`: só `ended_at`/`ended_by_user_id`, de nulo a preenchido;
  - `field_devices`: `token_digest`, `user_id` e `professional_id` não mudam; `revoked_*` só de nulo a preenchido.
- **Exclusão LGPD (ADR 0026):** `Citizens::RequestErasure` retém o paciente com qualquer visita (inclusive retirada) ou participação (inclusive por engano), como o L13 do 24a. A exclusão nunca apaga cadastro nem domicílio (quem tem cadastro é retido), então o 24b não grava tombstone `erased` nem toca no `Citizens::Erase` (Q9).
- **Fora do sistema:** nada de nome, CPF, CNS, motivo, medida, observação, tema, prática, procedimento ou token em URL, log, evento ou SMS. `filter_parameters` ganha os campos que ainda não filtra (`measurements meeting_topics target_audience health_topics health_practices altered_evaluation smoking qr_payload place_text candidates withdrawal_reason mistaken_reason`); o Rails não loga cabeçalhos, e o spec de sigilo prova que o `Authorization` não aparece (Q1).
- **Trilha:** `clinical_record.viewed` com `access: "home_visits"` (telas do §6 e §8), `"collective_activities"` (§7) e `"field_sync"` (§9), publicados como `HomeVisits::Catalog::TRAIL_ACCESS`, `CollectiveActivities::Catalog::TRAIL_ACCESS` e `FIELD_TRAIL_ACCESS` (o api não tem lista fechada de `access`; Q1). Contagens (relidas a cada minuto pelo `COUNT_QUERY` do 23/24a; Q16), contexto, opções e listas sem pessoa não gravam.
- **Escopo:** registro de outro escopo por id → 404; lista de microárea ou equipe alheia → 403 `out_of_scope`.

**Códigos de erro** (além dos herdados `feature_disabled`, `mfa_required`, `not_found`, `invalid_request`, `invalid_cursor`, `invalid_limit`, `too_many_requests`, `payload_too_large`, e dos do 24a nos itens de cadastro):

| Código | HTTP | Onde |
|---|---|---|
| `cbo_not_allowed` | 403 / 422 | quem registra, aparelho, atribuição |
| `out_of_scope` | 403 | microárea, equipe, fora de área |
| `invalid_home_visit` | 422 | visita, com `path` |
| `visit_date_out_of_range` | 422 | §3 |
| `non_residential_with_person` | 422 | §3 |
| `reasons_forbidden`, `reason_not_allowed`, `measurements_forbidden` | 422 | §3, com `path` |
| `patient_in_other_household`, `family_without_responsible` | 422 | §3 |
| `day_changed`, `day_closed`, `visit_changed` | 409 | §2 |
| `invalid_collective_activity` | 422 | §4, com `path` |
| `activity_date_out_of_range`, `duplicate_participant`, `professional_without_cns` | 422 | §4 |
| `activity_changed` | 409 | §2 |
| `duplicate_suspected` | 409 | §5, com `candidates` |
| `cpf_taken`, `cns_taken` | 409 | §5, com `path` |
| `already_assigned`, `assignment_already_ended` | 409 | §6 (Q21) |
| `household_inactive`, `family_moved_out`, `micro_area_inactive` | 422 | §3 (Q21) |
| `person_not_found` | 404 | §5 (Q8) |
| `citizen_not_verified` | 409 | §5 (Q8) |
| `invalid_cpf`, `invalid_cns`, `identity_locked` | 422 / 409 | herdados do 24a, na busca e no registro rápido (Q21) |
| `device_limit` | 422 | §9.1 |
| `device_already_revoked` | 409 | §9.1 (Q21) |
| `invalid_token`, `device_revoked` | 401 | §9.2 |
| `client_uuid_conflict`, `stale_version` | item | §9.3 |

## 14. Telas

**dashboard** (grupo novo "Visita e atividade coletiva" no `NAV_GROUPS` de `src/shell/modules.ts`, depois de "Cadastro territorial"; cada item aparece com o seu interruptor e o vínculo; celular primeiro nas telas de quem registra):
- **Folha do dia:** visitas de hoje; nova visita pela microárea (domicílio → família ou morador) ou por imóvel sem cadastro; desfecho, motivos (por grupo, com as regras do §3), medidas, observação; registro rápido com a busca de duplicado; retirar com motivo; "Fechar o dia"; os `blockers` em linguagem simples ("Seu CNS não está no cadastro de profissionais").
- **Visitas da equipe:** filtros por microárea, profissional, período e desfecho; folhas com a situação da ficha; detalhe da visita.
- **Atribuição de microáreas** (enfermeiro): atribuir e encerrar, com step-up.
- **Quem visitar:** por microárea, com os motivos e o atalho "Registrar visita".
- **Atividade coletiva:** nova (o formulário segue as regras do §4 vindas de `options.domains`), lista, detalhe, "Registrada por engano".
- **Aparelhos de campo:** cadastrar (step-up, só o QR, uma vez; Q20), listar com o `status` (`inativo` quando o vínculo acabou; Q7), revogar; e a lista do administrador.
- **Linha do tempo do paciente:** os kinds `home_visit` e `collective_activity`; a visita não abre detalhe (Q18).
- **Produção:** as fontes `HomeVisitDay` ("folha de visitas") e `CollectiveActivity` ("atividade coletiva"), os motivos novos de "não gerada" e o "Reenviar" (Q12).
- **Linhas de cuidado:** o `min_gap_days` preservado e mostrado no editor (Q19).
- O domicílio e o morador da visita vêm das rotas de leitura do 24a, também para o CBO de campo que não é ACS (Q11).

**wpda:** entrada "Visita do agente de saúde" na linha do tempo, só com a data; "Meu acompanhamento" com a prática de visita.

**maintenance:** em `src/lib/features.ts`, os rótulos de `home_visits` e `collective_activities` e o `missing` `territory_registry_disabled` ("cadastro individual e domiciliar (territory_registry) desligado ou incompleto", com o nome do interruptor no maintenance; Q29). A versão nova do catálogo-base usa a tela que já existe, com o campo novo `min_gap_days` no formulário da prática (Q19) e o rótulo da fonte `acs_visit` "visita do ACS (acompanhada quando a visita domiciliar está ligada na cidade)" (Q29).

## 15. Entrega, testes e pendências

**Ordem:** contracts (schemas, sem a tag) → api (task 1 = prova técnica, que publica §10–§11; o contracts atualiza `domains-v1.json` e depois sai a tag `field-sync-v1.0.0`; Q13) → dashboard → wpda → maintenance → catálogo público (F-24.9..F-24.17). As Tasks 1 e 2 do maintenance podem rodar antes da prova do dashboard: a sessão do maintenance publica a versão nova do catálogo-base em dev (5177), com autorização do usuário, antes das provas do dashboard e do wpda (Q19). F-24.14 inclui o maintenance (Q19).

**Testes (estratégia):**
- api, por task, com `bin/test api -w mod24b <specs>`: cada regra do §3 e do §4 como spec de serviço; folha (99 + complementar, fechamento pelo dono e pelo job, interruptor desligado, prazo vencido); fichas 8 e 6 comparadas campo a campo com um exemplo de referência; "não gerada" por motivo; acesso pela tabela do §1; linhas de cuidado (`untracked?`, `home_visits!`, `min_gap_days`, pressão e peso); linha do tempo nas três visões; "quem visitar"; sincronização (idempotência, item recusado, `stale_version`, `reset`, tombstones, trilha por página) validada contra a cópia dos schemas; sigilo de log, URL e evento; triggers; retenção LGPD.
- **Suíte completa** do api pela sessão "Testes" antes do merge (migração de cidade, `CareLines`, `Timeline`, `Ledi`, cifra).
- dashboard e wpda: `bin/test <app> -w <repo>-mod24b`, componentes na largura de celular.
- contracts: exemplos validam contra o schema.

**Prova no navegador:** folha do dia no celular (visita a morador, à família e a imóvel não residencial; recusa; retirada; registro rápido com duplicado; fechar o dia); visitas da equipe pelo enfermeiro; atribuição; quem visitar; atividade tipo 5 com participantes e tipo 1 sem; "registrada por engano"; aparelho com QR e revogação; envio e download pelo contrato com `curl` (sem o app); linha do tempo no dashboard e no wpda; "Meu acompanhamento"; interruptores no maintenance.

**Runbook:** `operacao/rollout-visita-e-coletiva.md` (spec §7), com a publicação da versão nova do catálogo-base e "aparelhos só quando o 24c existir".

**Gates de go-live:** prova no PEC real das fichas 8 e 6, inclusive duas fichas do mesmo profissional no mesmo dia; SIAPS contando visita e atividade no Acompanhamento; regra da C5 contra o relatório do SIAPS.

**Pendências a abrir no fechamento:** correção ou anulação de ficha enviada (visita retirada depois do fechamento; atividade por engano já enfileirada); `min_gap_days` de C2, C3, C4 e C6; PSE; IVCF-20; agendamento de visita; latitude e longitude (24c); item `family` no envio do app (24c; Q27); rótulos "não acompanhada" de `vaccine` e `oral_health` no maintenance (`careLineTemplates.ts`), card no board #2 (Q29).

## 16. Decisões

**Do usuário (spec §2):**

| # | Decisão |
|---|---|
| U1 (1A) | O ACS registra a própria visita no dashboard, inclusive pelo celular, numa folha do dia com várias visitas, até o app de campo. Sem digitador. |
| U2 (2A) | Atividade coletiva com os 7 tipos, sem PSE (escolas com INEP no próximo ciclo). |
| U3 (3A) | No wpda, só a visita realizada, com a data e "Visita do agente de saúde"; recusa e ausência nunca; "Meu acompanhamento" conta a visita. |
| U4 (4A) | O 24b entrega o contrato de sincronização em `rotasaude/contracts` e o lado do api, testados sem o app; o 24c só consome. |
| U5 (5A) | "Quem visitar" é só leitura, por microárea, com três fontes (linhas de cuidado, cadastro a vencer, nunca visitado). Sem agendamento. |
| U6 (6A) | O ACS baixa só as microáreas em que é responsável; os outros CBOs de campo da Tabela 7, só as que o enfermeiro atribui (com trilha). Médico e enfermeiro não cadastram aparelho. Download e remoções seguem esse escopo. |
| U7 (7A) | Quem não é paciente entra por registro rápido (nome, nascimento, sexo e CPF, CNS ou a justificativa do 24a), no dashboard e no envio do app, com a guarda de duplicado do 24a. |

**Fixados na redação deste contrato** (para a revisão do coordenador; valem até a reconciliação):

| # | Ponto |
|---|---|
| C1 | `home_visits` exige `territory_registry` **utilizável**; sem IBGE ou UF, a visita fica indisponível (`territory_registry_disabled`, com rótulo "desligado ou incompleto" no maintenance). |
| C2 | Data aceita só com `Ledi::Deadline.on(competência) > hoje` (um dia de folga para o job de madrugada gerar a ficha). |
| C3 | Fora de área só para o ACS; os outros CBOs da Tabela 7 só nas microáreas atribuídas. O TACS entra por atribuição. |
| C4 | Atribuição de microárea pelo enfermeiro da equipe, com step-up; tabela `micro_area_field_assignments` (não estava no §3 da spec). |
| C5 | `home_visits.patient_id` é sempre a pessoa da ficha (morador ou responsável); `subject` diz qual. |
| C6 | Morador da visita: cadastro ativo no domicílio ou sem cadastro ativo; cadastro em outro domicílio é recusado. |
| C7 | Uma visita com problema segura a ficha da folha inteira ("não gerada"); `blockers` avisam antes. Folha só com visitas retiradas fecha `empty`, sem ficha. |
| C8 | Retirar só pelo dono da folha; "por engano" só por quem registrou; motivo de 10–500, cifrado, nos dois. |
| C9 | Participante sem CPF e sem CNS nos tipos 1–4 e 7 fica fora da lista da ficha e conta em `numParticipantes`. |
| C10 | Até 3 aparelhos ativos por usuário; token `rsf_` + 32 bytes; QR `rotasaude-field:v1:` com host e token; revogar o próprio sem step-up; o administrador revoga com step-up. |
| C11 | Envio: até 50 itens por lote, item `patient` próprio e `patient_client_uuid` para citar o paciente criado offline; só aceitos vão ao livro de idempotência. |
| C12 | Download: mudança de escopo responde `reset: true` (recarga completa); remoções por `field_sync_tombstones`; "quem visitar" só na última página, completo. |
| C13 | `PecTimelinesController::KINDS` exclui `home_visit` e `collective_activity`. |
| C14 | Catálogo-base novo: além do `min_gap_days: 30` na `c5_d`, `citizen_label` "Visitas do agente de saúde" nas seis práticas `acs_visit` (versões novas de cinco linhas). |
| C15 | Pressão e peso/altura da visita contam para qualquer CBO da Tabela 7; a fonte `acs_visit` só para 515105 e 322255. |
| C16 | `HomeVisits::CloseDaysJob` às 3h20 de Brasília (1h20 no Acre). |
| C17 | Nenhum nome novo de `Platform.audit`; o catálogo `events/` do contracts não muda. |
| C18 | Portas 3057, 5218 e 5219; o maintenance entra pela falta do rótulo `territory_registry_disabled` e das duas chaves. |
| C19 | Grupo novo "Visita e atividade coletiva" no menu do dashboard; buscas de pessoa próprias em `/home_visits/people/*` e `/collective_activities/people/*` (as do 24a recusam os CBOs de campo que não são ACS). |

**Da reconciliação dos planos** (Q-n; revisão de 2026-11-12):

| # | Decisão |
|---|---|
| Q1 (W-1, W-2, W-3, W-4, W-5, W-6, W-7, W-8, W-13) | Precisões aceitas como propostas, sem mudar comportamento: listas da linha do tempo em `Timeline::Query::…` (§12); Tabela 5 com 117 CBOs **strings**, dois alfanuméricos (§1); `registrar_cbo?` aceita a ficha pelo YAML, sem código novo (§1); `HomeVisits::Gate` não copia a exceção de perfil do 24a (§1, C1); `CareLines::BaseController` e `CitizenApi::CareController` também passam `home_visits:` (§12); os três `access` são constantes dos módulos, sem lista fechada (§13); `filter_parameters` só ganha o que falta e o spec de sigilo prova que o `Authorization` não aparece (§13); sem `Current.session` no app, `by:` explícito (§9.2); ordem na borda limite → interruptor → token (§9.2). |
| Q2 (A-1) | Migrações **irreversíveis**, como 24a e 33; o `up` termina executando `db/city_triggers.sql` (§1). |
| Q3 (A-2, A-3, W-10) | Até a task 1: `tipo_glicemia` = `0 Jejum, 1 Pós-prandial, 2 Pré-prandial, 3 Não especificado` (dicionário vendorizado); público-alvo com `18 Outros`, `19` e `20`, sem o 74; `dataAtendimento` pelo `RegistrationHeader` com o `day` às 00h. A task 1 confirma ou corrige contra `dicionario-fvd.html`/`dicionario-fac.html` e publica o acréscimo no §10–§11 (§10, §11). |
| Q4 (A-4, D-10) | Ficha 8 **reusa** `citizen_without_birth_date` e `citizen_without_sex` (já em `LediGenerationFailure::REASONS`, com rótulo); somem `patient_without_birth_date`/`patient_without_sex`. Ficha 6 ganha `participant_without_sex`. Códigos novos no módulo: `siaps_deadline_passed` e os cinco `participant_*`/`place_*` (§10, §11). |
| Q5 (A-5) | `ficha.status = not_exported` para folha fechada com visita válida e exportação não utilizável (§6). |
| Q6 (A-6) | A 100ª visita não retirada fecha a folha cheia na mesma transação, `closed_by = full`, ficha gerada, e abre a próxima `sequence`; `closed_by IN (user, job, full)` (§1, §2). |
| Q7 (A-7) | **Decisão do coordenador.** Não há evento de fim de vínculo: o token é conferido em toda requisição do app contra vínculo ativo em equipe 70/76 com CBO da Tabela 7 (e usuário ativo); sem isso, erro uniforme `401 device_revoked` e revogação nesse momento (`membership_ended`/`user_disabled`). As listas do §9.1 ganham `status: active \| inactive \| revoked` (inativo = vínculo perdido, ainda não revogado). Sem gancho em `Cnes::Apply` nem `User#deactivate!` (§2, §9.1, §9.2). |
| Q8 (A-8, D-6, D-7) | Busca de pessoa no padrão do 27a (`404 person_not_found`, `409 citizen_not_verified`, par validado vira paciente pelo `Patients::LookupByIdentifier`), com `birth_date` e `sex` na forma; candidatos do duplicado com `sex` e `has_cns`. Assim o par validado sem paciente nunca chega ao registro rápido (a proposta de "`new_patient` com CPF de par validado cria ligado" da D-6 fica rejeitada por desnecessária). Os mesmos viewers de antes; nenhum dado a quem não via (§5). |
| Q9 (A-9, A-10, A-11) | Tombstone só `moved` (domicílio troca de microárea; cadastro passa a domicílio de outra microárea). `erased` e `out_of_scope` ficam no enum e no schema, reservados, sem produtor no 24b; a exclusão LGPD não toca no `Citizens::Erase`. Saída do cadastro e domicílio inativo vão como item com o estado. Fim de atribuição → `reset: true` no próximo download (§1, §6, §9.3, §13). |
| Q10 (A-12) | Pressão e peso/altura da visita entram em `blood_pressure`/`weight_height` só com `home_visits` utilizável, pela mesma leitura do `acs_visit` (§12). |
| Q11 (D-3, W-9) | **Decisão do coordenador.** O CBO da Tabela 7 que não é ACS lê, **só leitura**, domicílios, famílias e moradores das microáreas atribuídas a ele (6A) pelas rotas de leitura do 24a (`GET /territory/households`, `/territory/households/:id`, `/territory/registrations?household_id=`), com o escopo da atribuição e a trilha do 24a. Rotas novas `/home_visits/households*` da D-3: rejeitadas. Escrita do 24a segue só do ACS (`cbo_not_allowed`; no app, item `rejected cbo_not_allowed`, e o 24c não oferece cadastro a esses CBOs) (§1, §9.3, §14). |
| Q12 (W-11, D-16) | Reenvio das fichas 8 e 6 só na Produção (`municipal_admin`, step-up); as telas do módulo só mostram a situação. "Gerar de novo" não é oferecido para `siaps_deadline_passed` (§10, §14). |
| Q13 (D-1, D-2, K-11, K-12) | `context.domains` (visita) e `options.domains` (atividade) com a forma fixada no §7 (a da D-1/D-2). O `domains-v1.json` mantém a estrutura do plano do contracts; as duas saem do mesmo YAML da task 1, e um spec do api confere que códigos e regras coincidem (duas formas, um dado). O api publica a tabela da task 1 no contrato; a sessão do contracts escreve `domains-v1.json` e devolve os SHA-256 ao api. Relações cruzadas só em `rules` e no api, com exceção dos pares de medida e de `id`+`version` no schema (§6, §7, §9.4, §15). |
| Q14 (D-4, D-5, D-20) | Capacidades por registro: `context.teams[].can_assign`, `can_register_device`, `professionals` (só com `reads_whole_team`); `<home_visit_day>.own`; `<home_visit>.can_withdraw`; `<collective_activity>.can_mark_mistaken`. O api continua recusando. O atalho "Registrar visita" do "quem visitar" usa `can_register` e o `scope`, sem campo novo (§6, §7). |
| Q15 (D-8, D-9, K-6, K-7) | Corpo explícito do `POST /collective_activities` com local **plano** (`place_health_unit_id` \| `place_text`, exatamente um; K-6 rejeitada para o pedido, a resposta segue com `place` aninhado); listas opcionais no pedido (ausente = vazia), sempre presentes na resposta; `duplicates_reviewed` por participante e o 409 com `path` `participants[i].new_patient` (§5, §7). O mesmo corpo vale no item `collective_activity` do app. |
| Q16 (D-11, D-17, D-18) | `days/today` é atalho; a tela pode listar as folhas de hoje por `GET /home_visits/days`. Contagens relidas a cada minuto (`COUNT_QUERY`), sem trilha. A sessão da equipe só lista `home_visits` quando utilizável (§1, §6, §13). |
| Q17 (D-12) | `withdrawn.reason` e `mistaken.reason` são `string \| null`, nulos para quem não pode ver (§6, §7). |
| Q18 (D-13) | A entrada `home_visit` da linha do tempo do profissional não abre detalhe nesta entrega; o `detail_ref` fica na resposta (§12, §14). |
| Q19 (M-2, D-14, M-4, M-5) | **Decisão do coordenador.** O formulário de prática do maintenance ganha o campo `min_gap_days` (proposta do M-2); o editor do dashboard preserva e mostra o atributo, sem oferecê-lo. F-24.14 inclui o maintenance. As Tasks 1–2 do maintenance podem rodar antes da prova do dashboard, e a sessão do maintenance publica a versão nova do catálogo-base em dev, com autorização do usuário, antes das provas do dashboard e do wpda (§12, §14, §15). |
| Q20 (D-15) | Aparelho: a tela mostra só o QR; o texto do token nunca vai ao DOM. Código curto de pareamento, se o 24c precisar, é versão nova do contrato (§9.1). |
| Q21 (D-19) | Códigos nomeados: `household_inactive`, `family_moved_out`, `micro_area_inactive` (422), `assignment_already_ended`, `device_already_revoked` (409); herdados do 24a na busca e no registro rápido: `invalid_cpf`, `invalid_cns`, `identity_locked` (§3, §6, §9.1, §13). |
| Q22 (W1, W3, W4, W5) | wpda: sem `kinds[]` novo (a visita chega por `Timeline::KINDS`); rótulo de tipo "Visita", sem ícone; forma exata da entrada do cidadão fixada no §12; o api manda o **meio-dia local** em `occurred_at` (a hora real não sai do servidor; precedente do 30) e o wpda mostra só o dia; título como constante do canal e entrada descartada se vier com `status` ou sem data. É exatamente o 3A (§12). |
| Q23 (W2, W8, W9) | `home_visit` fora de `DECLARED_KINDS` (só par validado); com `home_visits` desligado a visita realizada continua no histórico (registro retido); visita à família aparece só para o responsável (`patient_id`) (§12). |
| Q24 (W6, M-6) | `collective_activities` entra em `CITIZEN_HIDDEN`; `home_visits` segue na sessão do cidadão, sem uso no wpda. Descrição de `home_visits` começa pelo conteúdo (§1). |
| Q25 (W7) | **Pendente do usuário:** a prática de visita em "Meu acompanhamento" diria "Faça até <dia>" / "atrasada", como as outras, embora quem age seja o agente. A alternativa do plano do wpda (item com `by_team: true` e frases "O agente de saúde deve passar até <dia>" / "A visita está atrasada — fale com a sua unidade de saúde") muda o que o cidadão lê além do 3A. Até a decisão, vale o plano: frases de sempre, sem campo novo (§12). |
| Q26 (K-1, K-2, K-3, K-8, K-9, K-10, K-13, K-14, K-15) | Schemas: `$id` em URL e `title` = `rotasaude.field_sync.<nome>.v1`; fechados salvo as cinco seções de dicionário do 24a; instantes com fuso (`occurred_at` do app exige); `error` por padrão, lista em `domains-v1.json`; `registration_id` opcional no candidato dos itens do 24a; `next_cursor` sempre presente e `who_to_visit` nulo ⇔ `has_more`; `scope` do aparelho `responsible \| assigned`; `property_type` e `no_cpf_reason` em texto; erros HTTP sem esquema (§3, §5, §9.3, §9.4). |
| Q27 (K-4, K-5) | Família não é item no v1 (documentado; `family` num MINOR quando o 24c pedir). `household_client_uuid` em `home_visit` e `individual_registration`, e `patient_client_uuid` em `individual_registration`, com a regra do item aceito antes; citação de item não aceito → `rejected not_found` (§9.3). |
| Q28 (K-16) | Trivial: o plano do contracts corrige o README da raiz no Step 6 da Task 2 (`lab-v1.1.0`, `billing-v1.1.0` e o prefixo `billing-vX.Y.Z`), no mesmo commit da linha do `field-sync`. Sem card. |
| Q29 (M-1, M-3) | Frase do pré-requisito com o nome do interruptor do maintenance: "cadastro individual e domiciliar (territory_registry) desligado ou incompleto". Rótulo de `acs_visit`: "visita do ACS (acompanhada quando a visita domiciliar está ligada na cidade)". **Decisão do coordenador:** os rótulos "não acompanhada" de `vaccine` e `oral_health` ficam fora do 24b; card no board #2 a abrir no fechamento (§14, §15). |
| Q30 (W-12) | Só redação do plano: o api segue os 8 eventos do §1 (a spec §5 lista 7; vale o contrato). |

**Pendente do usuário:** Q25 (frase da prática de visita em "Meu acompanhamento" do wpda).
