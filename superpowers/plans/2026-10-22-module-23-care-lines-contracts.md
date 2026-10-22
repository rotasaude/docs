# Módulo 23 — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-22-module-23-care-lines-design.md` · **ADR:** `adr/0040.md`

Fonte única dos formatos entre `api`, `dashboard`, `wpda` e `maintenance`.
Convenções herdadas (contratos do 21a §15, 21b §14, 21c §11, 22 §12–§13): erros
`{ "error" }`; step-up; interruptor 403 `feature_disabled`; JSON em toda escrita por
cookie; listas com envelope e `next_before`; registro de outro escopo por id → 404,
lista com escopo alheio → 403; leitura nominal com trilha e sem polling; contadores
sem trilha; texto longo recusado; todo Data/Struct com dado pessoal sem dado em
`inspect`/`to_s`/`pretty_print`.

## 1. Nomes no código

| Spec | Código |
|---|---|
| linhas do catálogo | `child_c2`, `pregnancy_c3`, `diabetes_c4`, `hypertension_c5`, `elderly_c6`, `women_c7` |
| fonte da boa prática | `consultation`, `blood_pressure`, `weight_height`, `exam:<sigtap>`, `procedure:<sigtap>`, `acs_visit`, `vaccine:<código>` |
| situação | `ok` \| `due_soon` \| `overdue` \| `untracked` |
| cronograma | `proportional` \| `end_of_window` |
| âncora | `enrollment` \| `birth` \| `lmp` (DUM) \| `rolling` |
| origem da sugestão | `problem` \| `age_sex` \| `declared` \| `resolved_problem` (sugestão de encerramento) |
| motivo de saída | `discharge` \| `death` \| `moved` \| `pregnancy_ended` \| `other` |
| canal do contato | `phone` \| `home_visit` \| `wpda_message` \| `in_person` |
| desfecho do contato | `reached` \| `no_answer` \| `scheduled` \| `refused` |
| condição declarada | `diabetes` \| `hypertension` \| `pregnancy` \| `mental_health` \| `none` |

## 2. Formas

`<care_line>`: `{ "id", "name", "source": "template" | "custom", "template_key"|null, "template_version"|null, "indicator"|null, "eligibility", "practices": [ { "key", "label", "source", "window_days", "min_count", "schedule", "anchor", "cbo_filter"|null } ], "declared_condition"|null, "active" }`.

`<enrollment>` (nominal; trilha): `{ "id", "person": { "patient_id"|null, "citizen_id"|null, "display_name", "age" }, "care_line": { "id", "name" }, "team": { "id", "ine", "name" }, "anchor_date"|null, "status", "enrolled_at", "worst_status", "practices": [ { "key", "label", "status", "done_at"|null, "due_on"|null, "count"|null, "expected"|null } ], "last_contact": { "channel", "outcome", "at" }|null }`.

`<suggestion>`: `{ "id", "person": {…}, "care_line": { "id", "name" }, "reason", "created_at" }` (sem texto clínico; `reason` só o código).

## 3. Linhas da cidade (`/care_lines/admin`, `municipal_admin`; escrita com step-up)

- `GET /care_lines/admin/templates` (catálogo-base, só leitura).
- `GET/POST/PATCH /care_lines/admin/lines[/:id]` — adotar (`{ "template_key", "template_version" }`) ou criar própria (`{ "name", "eligibility", "practices", "declared_condition"? }`); 422 `invalid_practices`, `invalid_eligibility`, `duplicate_template`.

## 4. Sugestões e inscrição (`/care_lines`, `health_professional` com vínculo na unidade)

- `GET /care_lines/suggestions?team_id=&care_line_id=&before=` (nominal; trilha).
- `POST /care_lines/suggestions/:id/accept` — `{ "team_id", "anchor_date"? }` (só médico ou enfermeiro: CBO 225* ou 2235*; 403 `cbo_not_allowed`); cria o paciente por `Patients::Resolve` quando a sugestão é de cidadão validado; 422 `anchor_required` (gestante sem DUM).
- `POST /care_lines/suggestions/:id/dismiss` — `{ "reason" }`.
- `POST /care_lines/enrollments` (manual) — `{ "patient_id" | "citizen_id", "care_line_id", "team_id", "anchor_date"? }`; 409 `already_enrolled`, 409 `citizen_not_verified`.
- `POST /care_lines/enrollments/:id/end` — `{ "reason", "note"? }` (motivo do §1).

## 5. Painel e busca ativa

- `GET /care_lines/enrollments?team_id=&care_line_id=&status=overdue|due_soon|all&before=&limit=` → `{ "enrollments": [<enrollment>], "next_before" }` (nominal; trilha uma por consulta, `clinical_record.viewed` com `access: care_line`; sem polling).
- `GET /care_lines/enrollments/:id` → `<enrollment>` com histórico de contatos.
- `POST /care_lines/enrollments/:id/contacts` — `{ "channel", "outcome", "note"? }` (nota cifrada, até 500).
- Agendar: o dashboard abre o pedido de agendamento do módulo 17 com o paciente preenchido (rota existente).
- `GET /care_lines/suggestions/count?team_id=` e `GET /care_lines/enrollments/overdue/count?team_id=` → `{ "count" }` (sem trilha).

## 6. Prévia (`/care_lines/preview`, `municipal_admin` e `analyst`; agregado sem nome)

`GET /care_lines/preview?team_id=&indicator=C2..C7` → `{ "indicator", "team", "denominator", "score", "max_score", "practices": [ { "key", "tracked": bool, "weight", "achieved" } ], "excluded": ["acs_visit", "vaccine"], "estimate": true }`. Denominador = população elegível (problema ativo, idade/sexo) entre pacientes e cidadãos validados da equipe, inscritos ou não; pesos das práticas não rastreadas redistribuídos.

## 7. Cidadão (wpda, par validado)

- `GET/POST/DELETE /citizen/declared_conditions?citizen_id=` — `{ "condition" }`; cifrado; retirar grava `withdrawn_at`.
- `GET /citizen/care?citizen_id=` → `{ "lines": [ { "name", "items": [ { "label", "status", "due_on"|null } ] } ] }` (só linhas inscritas; linguagem simples; sem código nem motivo); 403 `not_verified`.
- Triagem vencida (depende do interruptor do módulo 15): a lista de ofertas do cidadão ganha `overdue: bool` e `last_done_at` por oferta com intervalo de repetição.

## 8. Maintenance

- Interruptor `care_lines` (`requires: ["feature:clinical_record"]`).
- Catálogo-base: `careLineTemplates` (lista) e `upsertCareLineTemplate` (nova versão) → `{ ok, errors }`; evento de plataforma `care_line.template_changed` (R18).

## 9. Eventos (só ids)

`care_line_suggestion.created`, `.accepted`, `.dismissed`, `care_line_enrollment.created`, `.ended`, `care_outreach.recorded`, `citizen_condition.declared`, `.withdrawn`, `care_line.adopted`.

## 10. Ordem de entrega

0. Prova técnica (plano do api, Task 0): notas metodológicas C2–C7 vigentes — boas práticas, janelas, contagens, códigos CIAP/CID de elegibilidade, pesos — fixadas no catálogo-base.
1. `api` (porta 3045; migrações de cidade e plataforma a partir de 20261022900001; worktree `.claude/mod23`, branch `feat/mod-23-care-lines`; testes com `bin/test api -w mod23`).
2. `dashboard` (Vite 5198) e `wpda` (Vite 5199), worktrees em `rota-saude/.worktrees/<app>-mod23`. 3. `maintenance`. 4. catálogo público.
