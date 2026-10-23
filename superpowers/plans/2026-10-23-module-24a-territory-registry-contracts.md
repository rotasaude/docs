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
| situação do cadastro | `valid` \| `due_soon` (vence em ≤ 60 dias) \| `expired` (> 24 meses) \| `missing` \| `refused` \| `exited` |
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
