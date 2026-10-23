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
