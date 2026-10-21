# Módulo 22 — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-21-module-22-procedures-billing-design.md` · **ADR:** `adr/0039.md`

Fonte única dos formatos entre `api`, `dashboard`, `maintenance` e `contracts`.
Convenções herdadas (contratos do 21a §15, 21b §14, 21c §11): `{ "error": "<reason>" }`;
step-up 401 `mfa_required`; interruptor 403 `feature_disabled`; JSON em toda escrita
por cookie; listas com envelope e `next_before`; registro de outro escopo por id → 404,
lista/criação com escopo alheio → 403; contadores sem trilha; ordem de travas estável;
texto longo recusado (nunca cortado); decimais como string com ponto.

## 1. Nomes no código

| Spec | Código |
|---|---|
| papel | `Membership` `billing_clerk` (privilegiado) |
| origem | `attendance` \| `consultation` \| `lab_report` \| `manual` |
| instrumento | `bpa_i` \| `bpa_c` \| `not_bpa` |
| caráter | `elective` \| `urgent` |
| situação do procedimento | `recorded` \| `cancelled` |
| situação da competência | `open` \| `closed` |
| estabelecimento | `{ "type": "health_unit" \| "lab_provider", "id", "cnes", "name" }` |
| motivos de crítica | `not_in_sigtap`, `cbo_not_allowed`, `age_out_of_range`, `sex_not_allowed`, `cid_required`, `cid_not_allowed`, `not_bpa_instrument`, `establishment_is_aps`, `missing_professional_cns`, `missing_professional_cbo`, `missing_patient_id`, `missing_required_patient_field`, `missing_establishment_cnes` |

## 2. Formas

`<performed_procedure>`:
```json
{ "id", "patient": { "id", "display_name" }, "establishment": <establishment>,
  "professional": { "user_id", "name", "cbo_code" }, "performed_on", "sigtap_code", "label",
  "sigtap_competence", "quantity", "cid10"|null, "character", "source", "source_ref"|null,
  "instrument", "billable", "blocking_reasons": [string], "status", "cancel_reason"|null,
  "billing_competence"|null, "sigtap_fallback": bool }
```
`sigtap_fallback: true` quando a crítica usou a última release ativa por falta da release da competência.

`<billing_period>`: `{ "id", "establishment", "competence", "status", "version", "closed_at"|null, "closed_by"|null, "reopened_reason"|null, "summary": { "bpa_i_lines", "bpa_c_lines", "not_billable": { "<motivo>": n }, "pending": n } }`.

## 3. Procedimentos (`/billing/procedures`)

- `POST /billing/procedures` — `{ "patient_id", "establishment": { "type", "id" }, "performed_on", "sigtap_code", "quantity", "cid10"?, "character"?, "attendance_id"?, "consultation_id"? }` → 201 `<performed_procedure>` (sempre grava; a crítica vai nos campos). Quem: profissional com vínculo na unidade (atendimento/consulta/avulso) ou `billing_clerk` (`manual`). 422 `invalid_sigtap_code`, `invalid_quantity` (1..999), `invalid_date` (futuro ou antes da competência mais antiga aberta), `invalid_cid10`, `invalid_establishment`.
- `GET /billing/procedures?establishment_type=&establishment_id=&competence=&billable=&reason=&patient_id=&before=&limit=` → `{ "procedures": [...], "next_before" }` (`billing_clerk`, admin; profissional só os próprios por `patient_id`).
- `POST /billing/procedures/:id/cancel` — `{ "reason" }`; 409 se já incluído em arquivo de competência fechada.
- Atendimento e consulta: o JSON de `attendance`/`consultation` ganha `procedures: [<performed_procedure resumido>]` (só com `procedure_billing` ligado).
- SIGTAP: `GET /billing/sigtap/search?q=&competence=` → `{ "procedures": [ { "code", "name", "instruments", "requires_cid", "sex", "age_min_months", "age_max_months" } ] }` (reaproveita a plataforma).

## 4. Laboratório (21b → 22)

- Liberação (`released`) de laudo do laboratório municipal cria um `performed_procedure` por item liberado: `source: lab_report`, estabelecimento = `lab_provider` do laboratório, profissional = liberador (CBO do perfil), `performed_on` = data da coleta, `quantity: 1`. Retificação não cria procedimento; laudo `withdrawn` não cria. Idempotente por item.

## 5. Competência (`/billing/periods`, `billing_clerk`; admin lê)

- `GET /billing/periods?competence=` → `{ "periods": [<billing_period>] }` por estabelecimento faturante (não-APS + laboratório municipal) com produção.
- `GET /billing/periods/:id/pending` → `{ "items": [ { "procedure_id", "reasons", "fix": "professional" | "patient" | "establishment" | "procedure" } ] }` (críticas recalculadas na hora; grava trilha por conter paciente).
- `POST /billing/periods/:id/close` (step-up) → recalcula críticas, gera `billing_export` (versão), `status: closed`; 422 `nothing_to_export`, `settings_missing`.
- `POST /billing/periods/:id/reopen` (step-up) — `{ "reason" }` → `status: open`, próxima geração = nova versão.
- `GET /billing/periods/:id/exports` → versões `{ id, version, sha256, lines_bpa_i, lines_bpa_c, generated_at, generated_by }`; `GET /billing/exports/:id/file` → `text/plain` no layout (só `billing_clerk`; evento `billing_export.downloaded` com trilha).
- Procedimento com data de competência já fechada entra na competência aberta seguinte do estabelecimento, com `late: true` no item.

## 6. Configuração (`/billing/settings`, `municipal_admin`, step-up)

`GET/PUT /billing/settings` — `{ "origin_name", "origin_cnpj", "destination_body", "management": "M" | "E" }`; 422 `invalid_cnpj`, `invalid_management`.

## 7. Arquivo (`contracts/billing/bpa-interface.md`)

Perfil do layout de interface do BPA Magnético vigente (lido na Task 0 do plano do api a partir do PDF do DATASUS de 09/07/2026), com: cabeçalho `01`, linhas `02` (BPA-C) e `03` (BPA-I), larguras, preenchimento, CR+LF, codificação, folhas de 20 linhas, campo de controle (soma módulo 1111 + 1111), regras de "sem informação" (raça/cor, etnia), IBGE de residência = município da cidade, endereço/telefone do cidadão quando houver. Exemplos golden em `contracts/billing/examples/`.

## 8. Maintenance

Interruptor `procedure_billing` (sem `requires`).

## 9. Linha do tempo

`kind` novo `procedure` em `Timeline::Sources::Procedures`: profissional vê nome, data, unidade e profissional; cidadão vê nome e data (sem crítica, sem faturamento).

## 10. Eventos (só ids)

`performed_procedure.recorded` (`source`, `billable`), `performed_procedure.cancelled`, `billing_period.closed`, `billing_period.reopened`, `billing_export.generated`, `billing_export.downloaded`, `billing_settings.changed`.

## 11. Ordem de entrega

0. Prova técnica (plano do api, Task 0): ler `Layout_Exportacao_BPA.pdf` e `BPA_LEIAME.txt` de 09/07/2026; registrar mudanças desde 2020 e campos obrigatórios.
1. `contracts` — perfil e exemplos; tag `billing-v1.0.0`.
2. `api` (porta 3044; migrações de cidade a partir de 20261021900001; worktree `.claude/mod22`, branch `feat/mod-22-billing`, bancos `_mod22`).
3. `dashboard` (Vite 5197; worktree `rota-saude/.worktrees/dashboard-mod22`). 4. `maintenance`. 5. catálogo público.

## 12. Acréscimos (revisão dos planos, 2026-10-21)

**Decisão do usuário após a Task 0 (layout 05.00)**
- **D-U1 — Dados do paciente para o BPA-I:** `patients` ganha `race_color` (código do layout, sem `99`), `ethnicity` (obrigatória quando indígena), `address_zip`, `address_street`, `address_number`, `address_complement`, `address_district` (cifrados) — município de residência continua o da cidade. Captados (a) na recepção: check-in e cadastro do paciente, campos opcionais e incentivados; (b) pelo faturista na pendência do fechamento: `PATCH /billing/patients/:id/demographics` (`billing_clerk`, trilha, evento `billing_patient.demographics_updated` só com id). No fechamento, falta de raça/cor ou de CEP/endereço → pendência (`missing_race_color`, `missing_address`, `fix: "patient"`).

**Layout (Task 0 do api)**
- **D-L1:** BPA Magnético 05.00 (09/07/2026): cabeçalho `01` 130 posições, BPA-C `02` 48, BPA-I `03` 351, CR+LF; campo de controle = soma (procedimento + quantidade) módulo 1111 + 1111; CPF e CNS nunca juntos; "pessoa sem CPF" (posição 351); sexo `I` aceito; data de realização na competência atual e nas 3 anteriores.
- **D-L2:** `invalid_date` = fora da janela de 3 competências anteriores (substitui "antes da competência mais antiga aberta"); `sex_not_allowed` mantido como crítica do sistema; procedimento com BPA-I e BPA-C na SIGTAP vai como BPA-I; folhas de 20 também no BPA-I, série única, acima de 999 folhas → `too_many_sheets`; nome do paciente cortado em 30 (layout); `prd-org` = `BPA` (a conferir na importação real).

**dashboard (B1–B13), adotadas pelo api**
- B1 `patient_id` opcional com `attendance_id`/`consultation_id` (par validado sem paciente via `Patients::Resolve`; não validado → `citizen_not_verified`); B2 `GET /billing/procedures?attendance_id=`; B3 `POST /billing/patient_lookup {cpf}` (`invalid_cpf`); B4 `GET /billing/establishments`; B5 `professional_user_id`/`cbo_code` só do `billing_clerk` (`source: manual`; não faturista → 403; CBO do vínculo; no laboratório, liberador ativo), `GET /billing/professionals`, `invalid_professional`; B6 pendências com `performed_on`, `sigtap_code`, `label`, `patient`, `professional`; B7 `late` no procedimento; B8 409 `already_exported`, `reason_required` (10–500), 409 `invalid_transition`; B9 `{ exports }`, arquivo `bpa-<cnes>-<AAAAMM>-v<n>.txt`, versões para faturista e admin, arquivo só faturista; B10 `origin_acronym` (6, `cbc-sgl`), `too_long`, `field_required`, `GET` 200 com nulls, step-up só no `PUT`; B11 competência com `id`, `version` 0 antes do primeiro fechamento, admin lê; B12 profissional cancela só os próprios (quem fez); B13 a tela não oferece `ubs`.

**maintenance (M1–M3)**
- `procedure_billing` exige `ibge_code` (`ibge_code_missing`); todos os ambientes; rótulo "Faturamento de procedimentos (BPA)".

**api (F3, F7–F15)**
- Avulso do profissional e lançamento do faturista → `source: manual`; eventos novos `billing_period.pending_viewed` e `billing_patient.looked_up`; fornecedor `Timeline::Sources::ProcedureSource`; rotas liberadas pela chave ligada (`enabled?`); demais códigos e formas conforme a seção final do plano do api.
