# Módulo 24a — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-23-module-24a-territory-registry-design.md` · **ADR:** `adr/0042.md` (divisão: `adr/0041.md`)

Fonte única dos formatos entre `api`, `dashboard` e `maintenance` (o wpda não muda).
Convenções herdadas (contratos do 21a §15, 21b §14, 21c §11, 22 §12–§13 e 23 §11–§12):
erros `{ "error" }`; step-up; interruptor 403 `feature_disabled`; JSON em toda escrita
por cookie; listas com envelope e `next_before`; registro de outro escopo por id → 404,
lista com escopo alheio → 403; leitura nominal com trilha e sem polling; contadores sem
trilha; texto longo recusado; dado sensível (CPF, CNS, condições, recusa, situação de
rua, moradia) só no corpo, nunca na URL nem no log (P20 do 23); todo Data/Struct com
dado pessoal sem dado em `inspect`/`to_s`/`pretty_print`/`as_json`.

## 1. Nomes no código

| Spec | Código |
|---|---|
| interruptor | `territory_registry` (`requires: ["feature:clinical_record"]`) |
| situação do cadastro | `valid` \| `due_soon` (vence em ≤ 60 dias) \| `expired` (≥ 24 meses desde a última versão, no dia da cidade) \| `missing` \| `refused` \| `exited` |
| motivo de saída | `moved` \| `death` |
| sem CPF | `no_cpf_reason`: códigos da ficha (fixados na Task 0 do api) |
| profissionais de campo | CBO `515105` (ACS), `515140` (agente de endemias) |
| quem cadastra | ACS responsável pela microárea; membro da equipe com CBO `2235*` (enfermeiro) ou `2251*`/`2252*`/`2253*` (médico) |
| fichas LEDI | `cadastro_individual`, `cadastro_domiciliar` (códigos de `tipoDadoSerializado` fixados na Task 0) |

Os dicionários das fichas (sociodemográfico, condições de saúde, situação de rua,
condições de moradia, tipo de imóvel, renda) usam as chaves dos campos do IDL LEDI
8.7.0 em `snake_case` e os domínios oficiais. A Task 0 do api publica a tabela
completa como acréscimo §11 deste contrato **antes** das tasks de formulário do
dashboard.

## 2. Formas

`<micro_area>`: `{ "id", "team": { "id", "ine", "name" }, "code": "01".."99", "name", "acs": { "professional_id", "name" }|null, "active" }`.

`<household>` (nominal; trilha): `{ "id", "micro_area": { "id", "code" }, "address": { "zip", "street", "number", "complement"|null, "neighborhood_id"|null, "reference"|null }, "property_type", "living_conditions": {…}, "contact_phone"|null, "refused": bool, "status": "active" | "inactive", "version", "registered_at", "registered_by": { "name"|null }, "families": [<family>] }`.

`<family>`: `{ "id", "responsible": { "patient_id", "display_name" }, "income_band"|null, "members_count", "living_since"|null, "moved_out": bool }`.

`<registration>` (nominal; trilha): `{ "id", "patient": { "id", "display_name", "birth_date", "sex", "has_cpf": bool, "no_cpf_reason"|null }, "household_id"|null, "family_id"|null, "is_responsible": bool, "sociodemographic": {…}, "health_conditions": {…}, "street_situation": {…}|null, "refused": bool, "exit": { "reason", "on" }|null, "status", "version", "registered_at", "registered_by": { "name"|null }, "identity_changed_at"|null }`.

`<person_row>` (validade): `{ "patient_id"|null, "citizen_id"|null, "display_name", "age", "micro_area_code"|null, "status", "registered_at"|null, "registration_id"|null }`.

## 3. Microáreas (`/territory/admin/micro_areas`, `municipal_admin`; escrita com step-up)

- `GET /territory/admin/micro_areas?team_id=` → `{ "micro_areas": [<micro_area>] }`.
- `POST` — `{ "team_id", "code", "name", "acs_professional_id"? }`; `PATCH /:id` — `{ "name"?, "acs_professional_id"?, "active"? }`.
- 422 `invalid_code`, 409 `duplicate_code`, 422 `acs_not_in_team`, 422 `acs_without_cns`, 422 `team_kind_not_allowed` (só equipes 70/76).

## 4. Domicílios e famílias (`/territory`, quem cadastra do §1, escopo da equipe)

- `GET /territory/households?micro_area_id=&before=&limit=` → `{ "households": [<household> sem `living_conditions`], "next_before" }` (trilha uma por consulta).
- `GET /territory/households/:id` → `<household>` (trilha).
- `POST /territory/households` e `PATCH /territory/households/:id` — corpo `<household>` sem `id`/`version`/`registered_*`; cada gravação é nova versão + ficha `cadastro_domiciliar`.
- `POST /territory/households/:id/families`, `PATCH /territory/families/:id` — corpo `<family>`; gera nova versão do domicílio + ficha.
- 422 `invalid_household` (com `path`), 422 `invalid_family`, 409 `stale_version` (`version` esperada no corpo).

## 5. Pessoas e cadastro individual

- `POST /territory/patients/duplicates` — `{ "full_name", "birth_date", "mother_name"? }` → `{ "candidates": [ { "patient_id", "display_name", "birth_date", "has_cpf" } ] }` (corpo, nunca query; trilha com os ids mostrados; no máximo 10).
- `POST /territory/registrations` — `{ "patient_id" | "citizen_id" | "new_patient": { "full_name", "social_name"?, "birth_date", "sex", "mother_name"?, "cpf"? | "no_cpf_reason", "cns"? }, "household_id"?, "family_id"?, "is_responsible"?, "sociodemographic", "health_conditions", "street_situation"?, "refused"? }`. `citizen_id` de par validado sem paciente cria o paciente por `Patients::Resolve`; `new_patient` cria o paciente direto (decisão 13).
- `PATCH /territory/registrations/:id` — mesmo corpo, com `version`; informar `cpf` de paciente sem CPF → 409 `cpf_taken` se outro paciente tem esse CPF.
- `POST /territory/registrations/:id/exit` — `{ "reason", "on" }`.
- `GET /territory/registrations/:id` → `<registration>` (trilha); `GET /territory/registrations?patient_id=` (trilha).
- 409 `already_registered` (cadastro ativo do paciente), 422 `invalid_registration` (com `path`), 422 `cpf_or_reason_required`, 409 `stale_version`, 403 `cbo_not_allowed`, 403 `out_of_scope`.

## 6. Validade e quem falta cadastrar

- `GET /territory/registry_status?team_id=&micro_area_id=&status=expired|due_soon|missing&before=&limit=` → `{ "people": [<person_row>], "next_before" }` (nominal; trilha; população = `TeamPopulation` do 23, pacientes e pares validados).
- `GET /territory/registry_status/count?team_id=&micro_area_id=` → `{ "valid", "due_soon", "expired", "missing" }` (sem trilha).

## 7. Profissionais (módulo 10)

CBO `515105` e `515140` dispensam registro em conselho; CNS continua obrigatório. Demais CBOs sem mudança.

## 8. LEDI

- `Ledi::FichaTypes` ganha `cadastro_individual` e `cadastro_domiciliar`; os tipos Thrift já gerados passam a ser carregados.
- Fonte da ficha: a versão imutável (`individual_registration_versions`, `household_versions`); atualização aponta a ficha originadora.
- Cabeçalho: CNS e CBO de quem gravou, INE da equipe, microárea do domicílio, CNES da unidade da equipe.
- Paciente sem CPF nas fichas de atendimento e no BPA-I: CNS no lugar do CPF; sem os dois, cai em `ledi_generation_failures` / pendência do faturamento.

## 9. Equipe da pessoa e óbito

- `CareLines::TeamPopulation`: cadastro individual ativo (sem saída) → equipe da microárea do domicílio; sem cadastro, a regra atual (inscrição, consulta recente com equipe, bairro — P18 e P22 do 23).
- `CareLines::Facts.deceased`: saída do cadastro por `death` OU inscrição encerrada por `death`.

## 10. Telas afetadas fora do 24a

- Faturamento (22): endereço do paciente com cadastro ativo só leitura, com o rótulo "do cadastro domiciliar".
- Recepção e prontuário: paciente sem CPF aparece com "sem CPF" e o CNS quando houver; nada quebra quando `cpf` é nulo.

## 11. Maintenance

Interruptor `territory_registry` (rótulo e dependência, mecanismo genérico).

## 12. Eventos (só ids)

`micro_area.changed`, `household.saved`, `territory_registration.saved`, `.exited`, `patient.cpf_added`.

## 13. Ordem de entrega

0. Prova técnica (plano do api, Task 0): regras oficiais das fichas de cadastro individual e domiciliar (LEDI 8.7, `/ledi/documentacao/regras/`), conferidas com o IDL vendorizado; códigos de tipo de ficha; tabela dos dicionários publicada como §11.
1. `api` (porta 3046; migrações de cidade a partir de 20261023900001; worktree `.claude/mod24a`, branch `feat/mod-24a-territory-registry`; testes com `bin/test api -w mod24a`).
2. `dashboard` (Vite 5200), worktree em `rota-saude/.worktrees/dashboard-mod24a`.
3. `maintenance` (Vite 5201), worktree em `rota-saude/.worktrees/maintenance-mod24a`.
4. Catálogo público.

## 14. Acréscimos (revisão dos planos, 2026-10-23)

Reconciliação dos planos do api (R1–R10, desvios), do dashboard (T1–T15) e do maintenance (N1–N3).

- **L1 — Dicionários (R1, T6):** a tabela dos dicionários das fichas, entregue pela Task 0 do api, vira o **§15** deste contrato (o §11 é Maintenance). Códigos das seções são **inteiros** como no IDL (T13). A tela usa os códigos de raça/cor do paciente (22: `"01".."05"`) e o api traduz para o LEDI (parda 03→4, amarela 04→3); o §15 traz a tabela de tradução (T15).
- **L2 — Equipes:** `GET /territory/admin/teams` (admin) → equipes 70/76 com os candidatos a ACS (CBO 515105, `has_cns`) (T1). `GET /territory/teams` (profissional) → equipes do vínculo com `micro_areas` e `can_register` (T2, como P22 do 23).
- **L3 — Moradores e busca:** `GET /territory/registrations?household_id=` (trilha) (T3). `POST /territory/person_lookup` — `{ "cpf" }` só no corpo → `{ "patient_id"|null, "citizen_id"|null, "display_name", "registration_id"|null }` (T4).
- **L4 — Identidade (R5, T5):** o corpo do cadastro ganha `identification` (nacionalidade, município de nascimento, nome do pai, celular, e-mail etc., chaves do §15) e `patient_demographics` (`race_color`, `ethnicity`, gravados no paciente). `full_name`, `birth_date`, `sex` e `mother_name` de paciente com par validado são só leitura: valor diferente → 422 `identity_locked`. 409 `already_registered` devolve `registration_id`. A validação posterior no posto gera nova versão e ficha em nome de quem gravou a versão anterior.
- **L5 — Sem CPF e sem CNS (T7):** o LEDI exige CNS quando não há CPF, e CPF e CNS nunca vão juntos. O cadastro grava mesmo sem os dois; a ficha fica "não gerada" (`patient_without_identifier`), e a validade mostra a pessoa como `missing_identifier` para a equipe obter o CNS. 409 `cns_taken` quando o CNS já é de outro paciente.
- **L6 — Endereço (R3, T8, R6, T9):** `address` ganha `street_type` (DNE), `no_number` e `district`; limites = o menor entre LEDI e BPA (logradouro 30, número 5, complemento 10, bairro 30). Imóveis tipo 07–11 exigem instituição de permanência (§15). Recepção e faturamento recebem `address_from_household: bool`; gravar endereço de paciente com cadastro ativo → 422 `address_from_household`.
- **L7 — Fora de área e domicílio (R2, R4):** cadastro sem domicílio exige `team_id` e vai com fora de área; `<registration>` ganha `team` e `micro_area`. `PATCH` do domicílio aceita `status` (`active` \| `inactive`).
- **L8 — Paciente nas outras telas (T10):** formas existentes que mostram paciente ganham `has_cpf` e `cns_masked` (só os 4 últimos dígitos).
- **L9 — Saída (T11, T14):** `exit` ganha `death_certificate` (número da DO, opcional) e `death_on`; saída de cadastro recusado → 422 `invalid_transition`. `moved`/`death` são traduzidos pelo api para os motivos da ficha.
- **L10 — Quem lê a validade (T12):** lista nominal para os profissionais com vínculo na equipe (ACS, enfermeiro, médico); contagens também para `municipal_admin` e `analyst`.
- **L11 — Trilha (R7):** listas e detalhe de domicílio gravam `clinical_record.viewed` com `access: territory_registry` (uma por consulta; ids mostrados).
- **L12 — Códigos (R8, R9):** acrescentam-se `cns_taken`, `citizen_not_verified`, `invalid_transition`, `invalid_request`, `invalid_limit`, `identity_locked`, `address_from_household`. Cursor de `registry_status` = `person_ref`. BPA-I sem CPF: CNS, CPF em branco e o indicador de sem CPF do layout, a confirmar na Task 0; muda `contracts/billing/bpa-interface.md` (tag `billing-v1.1.0`).
- **L13 — Retenção LGPD (R10):** retém quem tem qualquer versão de cadastro gravada ou é responsável de família (superconjunto de "ficha enfileirada"). Paciente sem CPF não tem pedido de exclusão pelo celular; o fluxo do posto é por CPF (runbook).
- **L14 — Exportação:** sem `ledi_export` a versão grava e nenhuma ficha nasce; não há varredura retroativa. O runbook manda ligar `ledi_export` antes de `territory_registry`.
- **L15 — Índice de duplicados:** usa a mesma cifra determinística do CPF sobre nome normalizado + nascimento (sem texto claro; a rotação de chave da cidade cobre o índice).
- **L16 — Maintenance (N1–N3):** rótulo do interruptor "Cadastro individual e domiciliar", sem `environments`; o catálogo chama o módulo de "Cadastro individual e domiciliar" até o 24b; a prova do maintenance usa a 5177 com o serviço do compose parado (o proxy exige o Origin constante), não a 5201.

- **L17 — Etnia sem código LEDI:** a etnia só se aplica a raça/cor indígena (`05`). Com raça/cor `05` e etnia sem código LEDI (tradução do §15.2), o cadastro recusa com 422 `invalid_ethnicity` (`path` `patient_demographics.ethnicity`); com outra raça/cor, a etnia não vai na ficha.
- **L18 — Situação de rua no BPA-I:** fora do 24a; o campo `prd_situacao_rua` continua em branco (pendência para o faturamento).

- **L19 — Pré-voo da execução (2026-10-23):** `GET /territory/registry_status/count` → `{ "valid", "due_soon", "expired", "missing", "missing_identifier" }`, e a lista aceita `status=missing_identifier`. A validade de 24 meses conta da **última** versão (gravação ou atualização de identidade). Cadastro com `household_id` copia o endereço do domicílio para o morador. Saída do cadastro só por quem cadastra (o cabeçalho da ficha exige CNS e CBO); o admin não registra saída. Quem cadastra precisa ter CBO das tabelas do LEDI (403 `cbo_not_allowed`).

- **L20 — UF e IBGE:** `territory_registry` declara a exigência de `CityProfile.uf` e `ibge_code` (mesmo mecanismo do `procedure_billing`: o maintenance mostra `usable: false` com `ibge_code_missing`/`uf_missing`, sem bloquear as rotas); enquanto faltarem, a versão grava e a ficha fica "não gerada" (`city_profile_incomplete`), nunca 500, e o reenvio gera a ficha depois de completar o perfil. Nomes vão para a ficha no conjunto oficial do "Validar nome" (Ç→C, diacríticos fora de agudo/circunflexo/til/trema removidos, hífen→espaço, apóstrofo simples); o paciente guarda o nome como digitado.

- **L21 — Forma do cadastro (Task 10 do api):** `<registration>.patient` ganha `full_name`, `mother_name`, `cns_masked` e `identity_locked`; `<registration>` ganha `patient_demographics` e `identification`; `exit` = `{ "reason", "on", "death_on"|null, "death_certificate"|null }` (`death_on` obrigatório no óbito). `social_name` também fica só leitura com par validado. `GET /territory/registrations?patient_id=` de paciente sem cadastro na equipe do usuário → 200 com lista vazia. `person_lookup` responde sem envelope. `patients:backfill_match_keys` é passo obrigatório do rollout.

- **L22 — Equipe na linha inscrita (ajusta o §9):** inscrição ativa numa linha de cuidado mantém a equipe da inscrição **naquela linha**; o cadastro define a equipe em todo o resto (demais linhas, população da equipe, prévia, validade). Cadastro com recusa e sem saída também define a equipe. Óbito registrado pela saída do cadastro conta como óbito mesmo com o interruptor desligado depois.

- **L23 — Validade (Task 12 do api):** exatamente 24 meses já é `expired` (conservador: "atualizado nos últimos 24 meses"). A lista de `registry_status` aceita também `status=valid` e `status=refused`. O ACS vê só a própria microárea; óbito sai da população; saída `moved` volta a pessoa para `missing`.

- **L24 — Revisão final do api (2026-10-24), ajusta o L4:** a validação posterior no posto só marca `identity_changed_at` (não gera versão nem ficha); `<registration>` e `<person_row>` ganham `identity_review_pending: bool` ("identidade atualizada no posto — revisar e salvar"), e a ficha de atualização nasce no próximo salvamento, em nome de quem salvou. Informar CPF junto com identidade digitada: os dados validados substituem os digitados (decisão 13). Duplicados incluem candidatos com mãe nula, depois dos exatos, com rate limit. Nome do logradouro não pode começar pelo tipo (422). Tipo de logradouro DNE no BPA-I (campo 30) fica fora do 24a, como pendência do faturamento (com o L18).

- **L25 — Fechamento do api (17175a8):** lista de domicílios em ordem `registered_at DESC, id DESC` com `next_before` opaco; `ocupacao_codigo_cbo2002` no formato `[0-9A-Z]{6}`; `POST /territory/patients/duplicates` limitado a 30 por minuto por usuário (429); chave `uf_missing` no maintenance (L20).

- **L26 — Endereço travado (Task 7 do dashboard):** `address_from_household` é calculado pelo mesmo lookup da gravação (par vinculado ou CPF) e vem também em `attendance#lookup` e `check_ins#exception`, além do check-in com código e da pendência do faturamento. Depois de um 422 `address_from_household`, a tela trava o endereço. Fora da raça/cor `05`, o PATCH de dados do paciente do faturamento manda `ethnicity: ""` (apaga a etnia gravada; o L17 vale para a ficha).

## 15. Dicionários das fichas (prova técnica, Task 0 do api)

Fonte: documentação LEDI 8.7.0 baixada em 2026-10-23 para `apps/api/vendor/ledi/8.7.0/docs/` (SHA-256 em `vendor/ledi/8.7.0/SOURCE`), conferida campo a campo contra o IDL vendorizado (`9316f280f5`). O arquivo que manda é `apps/api/config/ledi/registration_dictionaries.yml` (api `c1665bf`, guardado por `spec/services/ledi/registration_dictionaries_spec.rb`); as tabelas abaixo são geradas dele. Chave = nome do campo do IDL em `snake_case` (`underscore` do ActiveSupport); **quando a documentação e o IDL divergem no nome, vale o IDL** (é o que vai no binário). Códigos de domínio são **inteiros** (L1), exceto os dois domínios de campo texto do fim.

### 15.1 Tipos de ficha e cabeçalho

- `tipoDadoSerializado`: **2** = Cadastro Individual (`cadastro_individual`), **3** = Cadastro Domiciliar e Territorial (`cadastro_domiciliar`). Registrados no `Ledi::FichaTypes`.
- Comuns: `tpCdsOrigem = 3`; `uuid` com o CNES na frente (até 44); `fichaAtualizada` obrigatório; `uuidFichaOriginadora` obrigatório — igual ao `uuid` no cadastro novo, o `uuid` da originadora na atualização. Domiciliar: `tipoEndereco = 3`.
- `headerTransport` = `UnicaLotacaoHeader`: `profissionalCNS` (15, obrigatório), `cboCodigo_2002` (só os CBOs da Tabela 1 — individual, 124 códigos — ou da Tabela 2 — domiciliar, 115 códigos; em `config/ledi/registration_cbos.yml`), `cnes` (7), `ine` (10, opcional), `dataAtendimento` (epoch ms, não futuro), `codigoIbgeMunicipio` (7). `515105`, `515140`, `223505`, `223565`, `225125`, `225130`, `225142` e `225170` estão nas duas tabelas. As tabelas 1 e 2 diferem em poucos códigos (ex.: `515124` só na 1, `515125` só na 2).

### 15.2 Tradução dos códigos do paciente (módulo 22 → LEDI)

| campo do paciente | paciente (BPA) | ficha LEDI |
|---|---|---|
| `race_color` | `"01"` Branca, `"02"` Preta, `"03"` Parda, `"04"` Amarela, `"05"` Indígena | `racaCorCidadao` 1 Branca, 2 Preta, **4 Parda**, **3 Amarela**, 5 Indígena |
| `ethnicity` (só com `"05"`) | código CADSUS de 4 dígitos | `etnia` = o número do código CADSUS de `0001` a `0264`; **`0304` (PURI) → 406** e **`0315` (WARAO) → 407** (no LEDI, 304 e 315 são JENIPAPO-KANINDE e KATUKINA PANO). Os códigos LEDI 265–405 têm código CADSUS `X265`…`X405` e não existem no BPA. Etnia do paciente fora de `0001`–`0264`, `0304` e `0315` não tem código LEDI → a ficha não pode sair (`TerritoryRegistry::Catalog.ethnicity_to_ledi` devolve `nil`). |
| `sex` | `male` / `female` | `sexoCidadao` **0 Masculino**, **1 Feminino** (4 Ignorado e 5 Indeterminado não são usados) |
| saída | `death` / `moved` | `motivoSaidaCidadao` **135 Óbito**, **136 Mudança de território**; `dataObito` só e sempre com 135; `numeroDO` (9) só com 135 |

### 15.3 Identificação que mora no paciente (não está em `identification`)

`nomeCidadao` (`full_name`, 3–70, regra "Validar nome": ≥ 2 termos, sem espaço duplo, só letras romanas com acento e apóstrofo, maiúsculas, não "A A …", não dois termos de 2 letras), `nomeSocial` (`social_name`, ≤ 70, texto e apóstrofo), `dataNascimentoCidadao` (`birth_date`, nunca depois do atendimento nem 130 anos antes), `sexoCidadao`, `nomeMaeCidadao` (`mother_name`, 3–70 salvo `desconhece_nome_mae`), `racaCorCidadao`/`etnia` (§15.2), `cpfCidadao`/`cnsCidadao` (**nunca juntos**; CPF obrigatório se não há CNS e `stNaoPossuiCpf = false`; **CNS obrigatório com `stNaoPossuiCpf = true`**), `stNaoPossuiCpf` (obrigatório) e `justificativaNaoPossuiCpf` (`no_cpf_reason`, domínio `justificativa_nao_possui_cpf`, obrigatória com `stNaoPossuiCpf = true`). Derivados pelo api (não vão no corpo): `statusEhResponsavel` (`is_responsible`), `cpfResponsavelFamiliar`/`cnsResponsavelFamiliar` (só com `statusEhResponsavel = false`, um só), `microArea` (proibida com `stForaArea`) e `stForaArea` (L7). Com `statusTermoRecusaCadastroIndividualAtencaoBasica = true` (`refused`) ficam **proibidos** `condicoesDeSaude`, `emSituacaoDeRua`, `informacoesSocioDemograficas`, `informacoesSocioEconomicas` e `saidaCidadaoCadastro`, e a identificação fica opcional.

### 15.4 Domicílio e família (corpo do §4, não são seções do dicionário)

- `tipoDeImovel` (`property_type`, obrigatório, domínio `tipo_de_imovel`). `familias`, animais e `telefoneResidencia` só com tipo 1. `condicaoMoradia` proibida para 2, 3, 4, 5, 6, 12, 13 e 99, e com recusa. `instituicaoPermanencia` só para 7–11 (`nomeResponsavelTecnico` obrigatório). Para 7–11 ficam proibidos `areaProducaoRural`, `materialPredominanteParedesExtDomicilio`, `nuComodos`, `tipoAcessoDomicilio`, `tipoDomicilio` e `situacaoMoradiaPosseTerra` (este só para 7–11; os demais também para 13).
- `enderecoLocalPermanencia` obrigatório sem recusa: `bairro` (≤ 72), `cep` (8 dígitos), `codigoIbgeMunicipio` (7), `nomeLogradouro` (≤ 72), `numero` (1–10, letras e dígitos; proibido com `stSemNumero`), `numeroDneUf` (domínio texto `uf`), `tipoLogradouroNumeroDne` (domínio texto `tipo_logradouro`), `complemento` ≤ 30, `pontoReferencia` ≤ 40, `telefoneContato`/`telefoneResidencia` 10–11 dígitos, `microArea` (2; obrigatória salvo `stForaArea`). Os limites do Rota Saúde são os menores entre LEDI e BPA (L6).
- `FamiliaRow`: `numeroCnsResponsavel` (15) **ou** `cpfResponsavel` (11), nunca os dois; `dataNascimentoResponsavel`; `numeroMembrosFamilia` (≠ 0, até 3 dígitos); `numeroProntuario` (≤ 30, letras e dígitos); `rendaFamiliar` (domínio `renda_familiar`); `resideDesde` (epoch ms, não futuro); `stMudanca`. Famílias com o mesmo responsável são recusadas; `nuMoradores` ≥ número de famílias e ≥ Σ membros (1 quando a família não informa).
- `latitude`/`longitude` (um exige o outro) ficam para o 24c.

#### Identificação (`identification`) — `IdentificacaoUsuarioCidadaoThrift`, só os campos que não moram no paciente

| chave | campo IDL | tipo | domínio | obrigatório / condição (texto oficial) |
|---|---|---|---|---|
| `nome_pai_cidadao` | `nomePaiCidadao` | name (mín. 3, máx. 70) |  | As regras de validação de um nome estão descritas em Validar nome; Não deve ser preenchido se o campo desconheceNomePai = true. Caso contrário, o preenchimento é obrigatório. |
| `desconhece_nome_pai` | `desconheceNomePai` | bool |  |  |
| `desconhece_nome_mae` | `desconheceNomeMae` | bool |  |  |
| `nacionalidade_cidadao` | `nacionalidadeCidadao` | code | `nacionalidade` | **obrigatório** |
| `codigo_ibge_municipio_nascimento` | `codigoIbgeMunicipioNascimento` | digits (mín. 7, máx. 7) |  | Só pode ser preenchido se o campo nacionalidadeCidadao = 1 (Brasileiro). Neste caso é de preenchimento obrigatório. |
| `pais_nascimento` | `paisNascimento` | code | `pais` | Só pode ser preenchido se o campo nacionalidadeCidadao = 3 (Estrangeiro). Neste caso o preenchimento é obrigatório; Se o campo nacionalidadeCidadao for igual a 1 (Brasileira), este campo deve ser preenchido com 31 (BRASIL). |
| `dt_naturalizacao` | `dtNaturalizacao` | date |  | Só deve ser preenchido se o campo nacionalidadeCidadao = 2. Neste caso, é de preenchimento obrigatório; Não pode ser posterior a dataAtendimento; Não pode ser anterior a dataNascimentoCidadao. |
| `portaria_naturalizacao` | `portariaNaturalizacao` | text (máx. 16) |  | Só deve ser preenchido se o campo nacionalidadeCidadao = 2. Neste caso o preenchimento é obrigatório. |
| `dt_entrada_brasil` | `dtEntradaBrasil` | date |  | Só deve ser preenchido se o campo nacionalidadeCidadao = 3. Neste caso o preenchimento é obrigatório; Não pode ser posterior a dataAtendimento; Não pode ser anterior a dataNascimentoCidadao. |
| `telefone_celular` | `telefoneCelular` | digits (mín. 10, máx. 11) |  | Apenas números; O preenchimento do campo é obrigatório apenas se o campo nacionalidadeCidadao estiver preenchido como "brasileira" ou "naturalizado". |
| `email_cidadao` | `emailCidadao` | text (mín. 6, máx. 100) |  | Requerido seguir o padrão endereco@domínio.extensao. |
| `numero_nis_pis_pasep` | `numeroNisPisPasep` | digits (mín. 11, máx. 11) |  | Apenas números. |
| `estado_civil` | `estadoCivil` | code | `estado_civil` |  |
| `tipo_sanguineo` | `tipoSanguineo` | code | `tipo_sanguineo` |  |
| `dnv` | `dnv` | text (mín. 11, máx. 15) |  |  |

#### Sociodemográfico (`sociodemographic`) — `InformacoesSocioDemograficasThrift` + `InformacoesSocioEconomicasThrift`

| chave | campo IDL | tipo | domínio | obrigatório / condição (texto oficial) |
|---|---|---|---|---|
| `deficiencias_cidadao` | `deficienciasCidadao` | codes (até 6 itens) | `deficiencia_cidadao` | Requerido preenchimento de pelo menos um item se o campo statusTemAlgumaDeficiencia = true; Não deve ser preenchido se o campo statusTemAlgumaDeficiencia = false. |
| `grau_instrucao_cidadao` | `grauInstrucaoCidadao` | code | `curso_mais_elevado` |  |
| `ocupacao_codigo_cbo2002` | `ocupacaoCodigoCbo2002` | text (mín. 6, máx. 6) |  | Deve ser um código de CBO válido. |
| `orientacao_sexual_cidadao` | `orientacaoSexualCidadao` | code | `orientacao_sexual` | Não pode ser preenchido se o campo statusDesejaInformarOrientacaoSexual = false; É de preenchimento obrigatório se o campo statusDesejaInformarOrientacaoSexual = true. |
| `relacao_parentesco_cidadao` | `relacaoParentescoCidadao` | code | `relacao_parentesco` | Só pode ser preenchido se o campo statusEhResponsavel = false. |
| `situacao_mercado_trabalho_cidadao` | `situacaoMercadoTrabalhoCidadao` | code | `situacao_mercado_trabalho` |  |
| `status_deseja_informar_orientacao_sexual` | `statusDesejaInformarOrientacaoSexual` | bool |  | **obrigatório** |
| `status_frequenta_benzedeira` | `statusFrequentaBenzedeira` | bool |  |  |
| `status_frequenta_escola` | `statusFrequentaEscola` | bool |  |  |
| `status_membro_povo_comunidade_tradicional` | `statusMembroPovoComunidadeTradicional` | bool |  |  |
| `status_participa_grupo_comunitario` | `statusParticipaGrupoComunitario` | bool |  |  |
| `status_possui_plano_saude_privado` | `statusPossuiPlanoSaudePrivado` | bool |  |  |
| `status_tem_alguma_deficiencia` | `statusTemAlgumaDeficiencia` | bool |  |  |
| `identidade_genero_cidadao` | `identidadeGeneroCidadao` | code | `identidade_genero_cidadao` | Não pode ser preenchido se o campo statusDesejaInformarIdentidadeGenero = false; É de preenchimento obrigatório se o campo statusDesejaInformarIdentidadeGenero = true. |
| `status_deseja_informar_identidade_genero` | `statusDesejaInformarIdentidadeGenero` | bool |  | **obrigatório** |
| `responsavel_por_crianca` | `responsavelPorCrianca` | codes (até 6 itens) | `responsavel_crianca` | Não pode ser preenchido se a dataNascimentoCidadao for maior ou igual a 10 anos a partir da dataAtendimento. |
| `co_povo_comunidade_tradicional` | `coPovoComunidadeTradicional` | code | `povo_comunidade_tradicional` | Não deve ser preenchido se o campo statusMembroPovoComunidadeTradicional = false. |
| `alimentos_acabaram_antes_ter_dinheiro_comprar_mais` | `alimentosAcabaramAntesTerDinheiroComprarMais` | bool |  | Só pode ser preenchido se o campo comeuAlgunsAlimentosQueTinhaDinheiroAcabou for preenchido. Neste caso o preenchimento é obrigatório; Não deve ser preenchido se o campo statusTermoRecusaCadastroIndividualAtencaoBasica = true. |
| `comeu_alguns_alimentos_que_tinha_dinheiro_acabou` | `comeuAlgunsAlimentosQueTinhaDinheiroAcabou` | bool |  | Só pode ser preenchido se o campo alimentosAcabaramAntesTerDinheiroComprarMais for preenchido. Neste caso o preenchimento é obrigatório; Não deve ser preenchido se o campo statusTermoRecusaCadastroIndividualAtencaoBasica = true. |

#### Condições de saúde (`health_conditions`) — `CondicoesDeSaudeThrift`

| chave | campo IDL | tipo | domínio | obrigatório / condição (texto oficial) |
|---|---|---|---|---|
| `descricao_causa_internacao_em12_meses` | `descricaoCausaInternacaoEm12Meses` | text (máx. 100) |  | Não deve ser preenchido se o campo statusTeveInternadoem12Meses = false. |
| `descricao_outra_condicao1` | `descricaoOutraCondicao1` | text (máx. 100) |  |  |
| `descricao_outra_condicao2` | `descricaoOutraCondicao2` | text (máx. 100) |  |  |
| `descricao_outra_condicao3` | `descricaoOutraCondicao3` | text (máx. 100) |  |  |
| `descricao_plantas_medicinais_usadas` | `descricaoPlantasMedicinaisUsadas` | text (máx. 100) |  | Não deve ser preenchido se o campo statusUsaPlantasMedicinais = false. |
| `doenca_cardiaca` | `doencaCardiaca` | codes (até 3 itens) | `doenca_cardiaca` | Requerido preenchimento de pelo menos um item se o campo statusTeveDoencaCardiaca = true; Não deve ser preenchido se o campo statusTeveDoencaCardiaca = false. |
| `doenca_respiratoria` | `doencaRespiratoria` | codes (até 4 itens) | `doenca_respiratoria` | Requerido preenchimento de pelo menos um item se o campo statusTemDoencaRespiratoria = true; Não deve ser preenchido o campo statusTemDoencaRespiratoria = false. |
| `doenca_rins` | `doencaRins` | codes (até 3 itens) | `problema_rins` | Requerido preenchimento de pelo menos um item se o campo statusTemTeveDoencasRins = true; Não deve ser preenchido se o campo statusTemTeveDoencasRins = false. |
| `maternidade_de_referencia` | `maternidadeDeReferencia` | text (máx. 100) |  | Não pode ser preenchido se o campo statusEhGestante = false. |
| `situacao_peso` | `situacaoPeso` | code | `consideracao_peso` |  |
| `status_eh_dependente_alcool` | `statusEhDependenteAlcool` | bool |  |  |
| `status_eh_dependente_outras_drogas` | `statusEhDependenteOutrasDrogas` | bool |  |  |
| `status_eh_fumante` | `statusEhFumante` | bool |  |  |
| `status_eh_gestante` | `statusEhGestante` | bool |  | Não pode ser preenchido se: Campo sexoCidadao = 0 - Masculino; Campo dataNascimentoCidadao for menor que 9 anos ou maior que 60 anos a partir da dataAtendimento. |
| `status_esta_acamado` | `statusEstaAcamado` | bool |  |  |
| `status_esta_domiciliado` | `statusEstaDomiciliado` | bool |  |  |
| `status_tem_diabetes` | `statusTemDiabetes` | bool |  |  |
| `status_tem_doenca_respiratoria` | `statusTemDoencaRespiratoria` | bool |  |  |
| `status_tem_hanseniase` | `statusTemHanseniase` | bool |  |  |
| `status_tem_hipertensao_arterial` | `statusTemHipertensaoArterial` | bool |  |  |
| `status_tem_teve_cancer` | `statusTemTeveCancer` | bool |  |  |
| `status_tem_teve_doencas_rins` | `statusTemTeveDoencasRins` | bool |  |  |
| `status_tem_tuberculose` | `statusTemTuberculose` | bool |  |  |
| `status_teve_avc_derrame` | `statusTeveAvcDerrame` | bool |  |  |
| `status_teve_doenca_cardiaca` | `statusTeveDoencaCardiaca` | bool |  |  |
| `status_teve_infarto` | `statusTeveInfarto` | bool |  |  |
| `status_teve_internado_em12_meses` | `statusTeveInternadoEm12Meses` | bool |  |  |
| `status_usa_outras_praticas_integrativas_ou_complementares` | `statusUsaOutrasPraticasIntegrativasOuComplementares` | bool |  |  |
| `status_usa_planta_medicinais` | `statusUsaPlantaMedicinais` | bool |  |  |
| `status_diagnostico_mental` | `statusDiagnosticoMental` | bool |  |  |

#### Situação de rua (`street_situation`) — `EmSituacaoDeRuaThrift`

| chave | campo IDL | tipo | domínio | obrigatório / condição (texto oficial) |
|---|---|---|---|---|
| `grau_parentesco_familiar_frequentado` | `grauParentescoFamiliarFrequentado` | text (máx. 100) |  | Não pode ser preenchido se o campo statusSituacaoRua = false; Não pode ser preenchido se o campo statusVisitaFamiliarFrequentemente = false. |
| `higiene_pessoal_situacao_rua` | `higienePessoalSituacaoRua` | codes (até 4 itens) | `acesso_higiene` | Não pode ser preenchido se o campo statusSituacaoRua = false; Requerido preenchimento de pelo menos um item se o campo statusTemAcessoHigienePessoalSituacaoRua = true. |
| `origem_alimento_situacao_rua` | `origemAlimentoSituacaoRua` | codes (até 5 itens) | `origem_alimentacao` | Não pode ser preenchido se o campo statusSituacaoRua = false. |
| `outra_instituicao_que_acompanha` | `outraInstituicaoQueAcompanha` | text (máx. 100) |  | Não pode ser preenchido se o campo statusSituacaoRua = false; Não pode ser preenchido se o campo statusAcompanhadoPorOutraInstituição = false. |
| `quantidade_alimentacoes_ao_dia_situacao_rua` | `quantidadeAlimentacoesAoDiaSituacaoRua` | code | `quantas_vezes_alimentacao` | Não pode ser preenchido se o campo statusSituacaoRua = false. |
| `status_acompanhado_por_outra_instituicao` | `statusAcompanhadoPorOutraInstituicao` | bool |  | Não pode ser preenchido se o campo statusSituacaoRua = false. |
| `status_possui_referencia_familiar` | `statusPossuiReferenciaFamiliar` | bool |  | Não pode ser preenchido se o campo statusSituacaoRua = false. |
| `status_recebe_beneficio` | `statusRecebeBeneficio` | bool |  | Não pode ser preenchido se o campo statusSituacaoRua = false. |
| `status_situacao_rua` | `statusSituacaoRua` | bool |  | Obrigatório se o campo statusTermoRecusaCadastroIndividualAtencaoBasica = false. Não deve ser preenchido se o campo statusTermoRecusaCadastroIndividualAtencaoBasica = true. |
| `status_tem_acesso_higiene_pessoal_situacao_rua` | `statusTemAcessoHigienePessoalSituacaoRua` | bool |  | Não pode ser preenchido se o campo statusSituacaoRua = false. |
| `status_visita_familiar_frequentemente` | `statusVisitaFamiliarFrequentemente` | bool |  | Não pode ser preenchido se o campo statusSituacaoRua = false. |
| `tempo_situacao_rua` | `tempoSituacaoRua` | code | `tempo_situacao_de_rua` | Não pode ser preenchido se o campo statusSituacaoRua = false. |

#### Condições de moradia (`living_conditions`) — `CondicaoMoradiaThrift` + animais + `instituicaoPermanencia`

| chave | campo IDL | tipo | domínio | obrigatório / condição (texto oficial) |
|---|---|---|---|---|
| `abastecimento_agua` | `abastecimentoAgua` | code | `abastecimento_de_agua` |  |
| `area_producao_rural` | `areaProducaoRural` | code | `condicao_de_posse_e_uso_da_terra` | Não deve ser preenchido se o campo tipoDeImovel = 07 Abrigo, 08 Instituição de longa permanência para idosos, 09 Unidade prisional, 10 Unidade de medida socioeducativa, 11 Delegacia ou 13 CASAI; Não deve ser preenchido se o campo localizacao = 83 - Urbana ou 205 - Periurbana. |
| `destino_lixo` | `destinoLixo` | code | `destino_do_lixo` |  |
| `forma_escoamento_banheiro` | `formaEscoamentoBanheiro` | code | `forma_de_escoamento_do_banheiro_ou_sanitario` |  |
| `localizacao` | `localizacao` | code | `localizacao_da_moradia` | **obrigatório** |
| `material_predominante_paredes_ext_domicilio` | `materialPredominanteParedesExtDomicilio` | code | `material_predominante_na_construcao` | Não deve ser preenchido se o campo tipoDeImovel = 07 Abrigo, 08 Instituição de longa permanência para idosos, 09 Unidade prisional, 10 Unidade de medida socioeducativa, 11 Delegacia ou 13 CASAI. |
| `nu_comodos` | `nuComodos` | count |  | Não deve ser preenchido se o campo tipoDeImovel = 07 Abrigo, 08 Instituição de longa permanência para idosos, 09 Unidade prisional, 10 Unidade de medida socioeducativa, 11 Delegacia ou 13 CASAI. Não pode ser preenchido com o valor "0" (zero). |
| `nu_moradores` | `nuMoradores` | count |  | Não pode ser menor que o número de familias cadastradas; Não pode ser menor que o somatório dos valores preenchidos no campo numeroMembrosFamilia, considerando o valor 1 quando este campo não foi informado no cadastro de uma família. |
| `situacao_moradia_posse_terra` | `situacaoMoradiaPosseTerra` | code | `situacao_de_moradia` | **obrigatório** — Não devem ser preenchidos se o campo tipoDeImovel = 07 Abrigo, 08 Instituição de longa permanência para idosos, 09 Unidade prisional, 10 Unidade de medida socioeducativa ou 11 Delegacia. |
| `st_disponibilidade_energia_eletrica` | `stDisponibilidadeEnergiaEletrica` | bool |  |  |
| `tipo_acesso_domicilio` | `tipoAcessoDomicilio` | code | `tipo_de_acesso_ao_domicilio` | Não deve ser preenchido se o campo tipoDeImovel = 07 Abrigo, 08 Instituição de longa permanência para idosos, 09 Unidade prisional, 10 Unidade de medida socioeducativa, 11 Delegacia ou 13 CASAI. |
| `tipo_domicilio` | `tipoDomicilio` | code | `tipo_de_domicilio` | Não deve ser preenchido se o campo tipoDeImovel = 07 Abrigo, 08 Instituição de longa permanência para idosos, 09 Unidade prisional, 10 Unidade de medida socioeducativa, 11 Delegacia ou 13 CASAI. |
| `agua_consumo_domicilio` | `aguaConsumoDomicilio` | code | `agua_consumo_domicilio` |  |
| `tipo_origem_energia_eletrica` | `tipoOrigemEnergiaEletrica` | code | `tipo_origem_energia_eletrica` | Não deve ser preenchido se o campo stDisponibilidadeEnergiaEletrica = false. |
| `animais_no_domicilio` | `animaisNoDomicilio` | codes (até 8 itens) | `animal_no_domicilio` | Só pode ser preenchido se o campo tipoDeImovel = 01 Domicílio; Não pode ser preenchido se o campo stAnimaisNoDomicilio = false; Não pode ser preenchido se o campo statusTermoRecusa = true. |
| `st_animais_no_domicilio` | `stAnimaisNoDomicilio` | bool |  | Só pode ser preenchido se o campo tipoDeImovel = 01 Domicílio; Não pode ser preenchido se o campo statusTermoRecusa = true. |
| `quantos_animais_no_domicilio` | `quantosAnimaisNoDomicilio` | count |  | Só pode ser preenchido se o campo tipoDeImovel = 01 Domicílio; Não pode ser preenchido se o campo stAnimaisNoDomicilio = false; Não pode ser preenchido se o campo statusTermoRecusa = true; Não pode ser preenchido com o valor "0" (zero); Deve ser maior ou igual ao número de opções selecionadas no campo animaisNoDomicilio. |
| `instituicao_permanencia` | `instituicaoPermanencia` | object |  | Só pode ser preenchido quando o campo tipoDeImovel = 07 Abrigo, 08 Instituição de longa permanência para idosos, 09 Unidade prisional, 10 Unidade de medida socioeducativa ou 11 Delegacia; Não deve ser preenchido se o campo statusTermoRecusa = true. |
| `instituicao_permanencia.nome_instituicao_permanencia` | `nomeInstituicaoPermanencia` | text (máx. 100) |  |  |
| `instituicao_permanencia.st_outros_profissionais_vinculados` | `stOutrosProfissionaisVinculados` | bool |  |  |
| `instituicao_permanencia.nome_responsavel_tecnico` | `nomeResponsavelTecnico` | name (mín. 3, máx. 70) |  | **obrigatório** — As regras de validação de um nome estão descritas em Validar nome. |
| `instituicao_permanencia.cns_responsavel_tecnico` | `cnsResponsavelTecnico` | digits (mín. 15, máx. 15) |  |  |
| `instituicao_permanencia.cargo_instituicao` | `cargoInstituicao` | text (máx. 100) |  |  |
| `instituicao_permanencia.telefone_responsavel_tecnico` | `telefoneResponsavelTecnico` | digits (mín. 10, máx. 11) |  | Mínimo 10 dígitos, máximo 11 (DDD + 8 ou 9). |

#### Domínios (códigos inteiros, como no IDL)

| domínio | códigos |
|---|---|
| `abastecimento_de_agua` | 117 Rede encanada até o domicílio - Sistema de abastecimento público (municipal ou estadual); 214 Rede encanada até o domicílio - Sistema de abastecimento próprio da aldeia (SESAI); 118 Poço / Nascente no domicílio; 119 Cisterna - Água de chuva; 120 Carro pipa; 215 Captação direta de água do rio; 216 Captação direta de poço coletivo; 217 Ponto de abastecimento coletivo - chafariz; 121 Outro |
| `acesso_higiene` | 42 Banho; 43 Acesso a sanitário; 44 Higiene bucal; 45 Outros |
| `agua_consumo_domicilio` | 98 Fervida; 99 Clorada; 152 Mineral; 218 Filtrada com filtro de barro; 219 Filtrada por outro tipo de filtro; 220 Clorada intradomiciliar com hipoclorito de sódio; 100 Sem tratamento |
| `animal_no_domicilio` | 128 Gato; 129 Cachorro; 130 Pássaro; 223 Macaco; 224 Galinha; 225 Porco; 226 Répteis; 132 Outros |
| `condicao_de_posse_e_uso_da_terra` | 101 Proprietário; 102 Parceiro(a) / Meeiro(a); 103 Assentado(a); 104 Posseiro; 105 Arrendatário(a); 106 Comodatário(a); 107 Beneficiário(a) do banco da terra; 208 Terra indígena demarcada; 209 Terra não demarcada; 108 Não se aplica |
| `consideracao_peso` | 21 Abaixo do peso; 22 Peso adequado; 23 Acima do peso |
| `curso_mais_elevado` | 51 Creche; 52 Pré-escola (exceto CA); 53 Classe de alfabetização - CA; 54 Ensino fundamental 1ª a 4ª séries; 55 Ensino fundamental 5ª a 8ª séries; 56 Ensino fundamental completo; 61 Ensino fundamental especial; 58 Ensino fundamental EJA - séries iniciais (supletivo 1ª a 4ª); 59 Ensino fundamental EJA - séries finais (supletivo 5ª a 8ª); 60 Ensino médio, médio 2º ciclo (científico, técnico e etc); 57 Ensino médio especial; 62 Ensino médio EJA (supletivo); 63 Superior, aperfeiçoamento, especialização, mestrado, doutorado; 64 Alfabetização para adultos (Mobral, etc); 65 Nenhum |
| `deficiencia_cidadao` | 12 Auditiva; 13 Visual; 14 Intelectual / Cognitiva; 15 Física; 232 TEA; 16 Outra |
| `destino_do_lixo` | 93 Coletado; 95 Céu aberto; 221 Queimado; 222 Enterrado; 96 Outro |
| `doenca_cardiaca` | 24 Insuficiência cardíaca; 25 Outro; 26 Não sabe |
| `doenca_respiratoria` | 30 Asma; 31 DPOC / Enfisema; 32 Outro; 33 Não sabe |
| `estado_civil` | 1 Solteiro; 2 Casado; 3 Viúvo; 4 Separado jucidialmente; 5 União consensual |
| `etnia` | 406 códigos — ver `config/ledi/registration_dictionaries.yml` do api |
| `forma_de_escoamento_do_banheiro_ou_sanitario` | 122 Rede coletora de esgoto ou pluvial; 123 Fossa séptica; 124 Fossa rudimentar; 125 Direto para um rio, lago ou mar; 126 Céu aberto; 127 Outra forma |
| `identidade_genero_cidadao` | 149 Homem transgênero; 150 Mulher transgênero; 156 Travesti; 200 Homem cisgênero; 201 Mulher cisgênero; 203 Não-binário; 151 Outro |
| `justificativa_nao_possui_cpf` | 2 Paciente desacordado, sem documentos; 3 Recém-nascidos; 4 Indígenas sem CPF; 5 Pessoa em situação de rua sem documentos; 6 Recusa cultural ou religiosa; 7 Estrangeiro em trânsito (turista/visitante); 8 Pessoa indocumentada; 9 Crianças sem documentação emitida; 10 Estrangeiro residente sem CPF; 11 Internação compulsória (saúde mental ou ordem judicial); 12 População migrante/refugiada em processo de regularização; 13 Situações de calamidade ou desastre |
| `localizacao_da_moradia` | 83 Urbana; 84 Rural; 205 Periurbana |
| `material_predominante_na_construcao` | 109 Alvenaria com revestimento; 110 Alvenaria sem revestimento; 111 Taipa com revestimento; 112 Taipa sem revestimento; 113 Madeira emparelhada; 114 Material aproveitado; 115 Palha; 210 Lona; 211 Misto/Diferentes materiais; 212 Caule de palmeira; 213 Sem parede; 116 Outro material |
| `motivo_saida` | 135 Óbito; 136 Mudança de território |
| `nacionalidade` | 1 Brasileira; 2 Naturalizado; 3 Estrangeiro |
| `orientacao_sexual` | 148 Heterossexual; 154 Bissexual; 196 Gay; 197 Lésbica; 198 Assexual; 199 Panssexual; 155 Outro |
| `origem_alimentacao` | 37 Restaurante popular; 38 Doação grupo religioso; 39 Doação restaurante; 40 Doação popular; 41 Outros |
| `pais` | 346 códigos — ver `config/ledi/registration_dictionaries.yml` do api |
| `povo_comunidade_tradicional` | 35 códigos — ver `config/ledi/registration_dictionaries.yml` do api |
| `problema_rins` | 27 Insuficiência renal; 28 Outro; 29 Não sabe |
| `quantas_vezes_alimentacao` | 34 1 vez; 35 2 ou 3 vezes; 36 mais de 3 vezes |
| `raca_cor` | 1 Branca; 2 Preta; 4 Parda; 3 Amarela; 5 Indígena |
| `relacao_parentesco` | 137 Cônjuge / Companheiro(a); 138 Filho(a); 139 Enteado(a); 140 Neto(a) / Bisneto(a); 141 Pai / Mãe; 142 Sogro(a); 143 Irmão / Irmã; 144 Genro / Nora; 145 Outro parente; 146 Não parente |
| `renda_familiar` | 1 1/4 de salário mínimo; 2 Meio salário mínimo; 3 Um salário mínimo; 4 Dois salários mínimos; 7 Três salários mínimos; 5 Quatro salários mínimos; 6 Acima de quatro salários mínimos; 8 Ausência de renda |
| `responsavel_crianca` | 1 Adulto responsável; 2 Outra(s) criança(s); 133 Adolescente; 3 Sozinha; 134 Creche; 4 Outro |
| `sexo` | 0 Masculino; 1 Feminino; 4 Ignorado; 5 Indeterminado |
| `situacao_de_moradia` | 75 Próprio; 76 Financiado; 77 Alugado; 78 Arrendado; 79 Cedido; 80 Ocupação; 81 Situação de rua; 204 Temporária; 82 Outra |
| `situacao_mercado_trabalho` | 66 Empregador; 67 Assalariado com carteira de trabalho; 68 Assalariado sem carteira de trabalho; 69 Autônomo com previdência social; 70 Autônomo sem previdência social; 71 Aposentado / Pensionista; 72 Desempregado; 73 Não trabalha; 147 Servidor público / Militar; 74 Outro |
| `tempo_situacao_de_rua` | 17 Menos de 6 meses; 18 6 a 12 meses; 19 1 a 5 anos; 20 Mais de 5 anos |
| `tipo_de_acesso_ao_domicilio` | 89 Pavimento; 90 Chão batido; 91 Fluvial; 92 Outro |
| `tipo_de_domicilio` | 85 Casa; 86 Apartamento; 87 Cômodo; 206 Maloca; 207 Improvisado; 88 Outro |
| `tipo_de_imovel` | 1 Domicílio; 2 Comércio; 3 Terreno baldio; 4 Ponto Estratégico (cemitério, borracharia, ferro-velho, depósito de sucata ou materiais de construção, garagem de ônibus ou veículo de grande porte, casa de farinha); 5 Escola; 6 Creche; 7 Abrigo; 8 Instituição de longa permanência para idosos; 9 Unidade prisional; 10 Unidade de medida sócio educativa; 11 Delegacia; 12 Estabelecimento religioso; 13 CASAI; 99 Outros |
| `tipo_origem_energia_eletrica` | 227 Gerador comunitário; 228 Gerador individual; 229 Fotovoltaica individual; 230 Concessionária; 231 Fotovoltaica comunitária |
| `tipo_sanguineo` | 1 A+; 2 A-; 3 B+; 4 B-; 5 AB+; 6 AB-; 7 O+; 8 O- |

Domínios de campo **texto** do IDL (código com zeros à esquerda, nunca inteiro): `uf` (27 UFs, `numeroDneUf`; Paraná = `"17"`) e `tipo_logradouro` (393 tipos do DNE, `tipoLogradouroNumeroDne`; Rua = `"081"`, Avenida = `"008"`) — listas completas em `string_domains` do YAML.

### 15.5 Achados da prova

1. **`stAtivo`** está na documentação do `CadastroDomiciliar` (só em atualização; `false` só sem famílias ou com todas em `stMudanca`), mas **não existe no IDL vendorizado** (vai até o id 20 sem ele). Domicílio inativo sai como atualização com todas as famílias em `stMudanca = true` (Desvio 9 do plano do api).
2. **Nomes que divergem entre documentação e IDL** (vale o IDL): `statusTeveInternadoem12Meses` → `statusTeveInternadoEm12Meses`; `statusUsaPlantasMedicinais` → `statusUsaPlantaMedicinais`; `microarea` (identificação) → `microArea`. As regras condicionais citam os nomes da documentação.
3. A página do `CadastroIndividual` não lista `informacoesSocioEconomicas` (id 16 do IDL), mas documenta a struct; entra na seção `sociodemographic`.
4. **Etnia** precisa de tradução (§15.2), como raça/cor — cópia direta troca PURI por JENIPAPO-KANINDE.
5. **BPA-I sem CPF** (PDF `Layout_Exportacao_BPA.pdf` 05.00, SHA-256 `f318bc03…f9f1f`, baixado de novo e conferido): paciente pelo **CNS** em `Prd-cnspac` (60–74), `prd_cpf_pcnte` (339–349) em **brancos** e `prd_sem_cpf` (351) = **`S`** ("Pessoa sem CPF/Registro Civil", desde 07/2026; exceção ao atributo SIGTAP `058 — Obrigatório CPF`). Nota 2 do PDF: um documento só. Confirma o L12.
6. O BPA-I também tem `prd_situacao_rua` (350, `N`/`S`/branco, desde 12/2024), que o módulo 22 deixa em branco. Com o cadastro individual, a situação de rua passa a ser conhecida — **proposta: preencher o campo 350 fica fora do 24a**, como pendência; a coordenação decide.
7. **Município de nascimento** (`codigoIbgeMunicipioNascimento`) e **ocupação** (`ocupacaoCodigoCbo2002`) usam tabelas externas grandes (5.570 municípios; CBO 2002 inteira). As páginas foram vendorizadas (`municipios.html`, `cbo_disponiveis.html`); o YAML valida só o formato (7 dígitos; 6 caracteres). A escolha na tela (lista ou busca) é do plano do dashboard.
