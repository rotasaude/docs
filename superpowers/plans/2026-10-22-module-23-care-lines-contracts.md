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

## 11. Acréscimos (revisão dos planos, 2026-10-22)

Aceitos pelo coordenador; prevalecem sobre as seções acima. Detalhe nas seções finais de `2026-10-22-module-23-care-lines-{api,dashboard,wpda,maintenance}.md`.

**Notas metodológicas (Task 0 do api).** As seis notas C2–C7 (assinadas em 19/06/2026) e a NT 8/2026 (29/05/2026) estão fixadas no plano do api com SHA-256; pesos C2 2, C3 2, C4 1, C5 1, C6 1, C7 2; conceitos Regular 0,25, Suficiente 0,50, Bom 0,75, Ótimo 1,00. Mudou um SHA-256 → a execução para.

**api (L1–L14), aceitas**
- **L1 — Boa prática:** soma `window_start_days`, `code_filter`, `applies_to`, `all_of` (E de OUs), `points`, `citizen_label`.
- **L2 — Elegibilidade:** problemas e exclusões por prefixo CIAP/CID, idade em dias ou anos, sexo e identidade de gênero (C7 inclui homem trans e exclui mulher trans, como a nota), condição declarada, `requires_anchor`.
- **L3 — CBO de quem confirma ou inscreve:** médico 2231*, 2251*, 2252*, 2253* e enfermeiro 2235* (substitui 225*/2235* do §4).
- **L4 — Códigos:** `invalid_anchor` (DUM entre hoje − 336 dias e hoje), `line_inactive`, `invalid_transition`, `reason_required` (1–500)/`too_long`, `out_of_scope`, `immutable_rules`, `duplicate_declared_condition` e os demais do plano do api.
- **L5 — Sugestão:** `team_id` obrigatório e `enrollment_id`; pessoa sem equipe atribuível não recebe sugestão; aceitar a sugestão de encerramento (`{}`) encerra com `discharge`.
- **L6 — Busca ativa paginada:** a página pode vir curta com `next_before` não nulo ("continue"); a tela carrega mais só sob demanda.
- **L7 — Trilha:** um `clinical_record.viewed` (`access: care_line`) por pessoa mostrada.
- **L8 — Prévia:** + `template_version`, `concept`; por prática `denominator`, `points`, `label`; `score` e `weight` strings com uma casa (tela com vírgula); `excluded` com as famílias não rastreadas.
- **L9 — Triagem vencida:** o módulo 15 não tem interruptor; `overdue`/`last_done_at` vêm sempre no catálogo do cidadão; "Meu acompanhamento" ganha `overdue_triages`.
- **L10 — Apoio:** `GET /care_lines/lines`, `GET /care_lines/teams`.
- **L11 — Condição declarada:** forma de resposta do plano do api; `none` retira as demais; só par validado declara.
- **L12 — `<enrollment>`:** + `ended_reason`, `ended_at`; `contacts` no detalhe; `last_contact` sem nota.
- **L13 — Eventos:** levam `user_id` ou `citizen_id`; evento de plataforma `care_line.template_changed` e auditoria `maintenance.care_line_template.changed` (R18).

**Telas (dashboard, wpda, maintenance) — acrescentar no api (não estão no plano do api)**
- **T1 — Condição declarada fora da URL:** `DELETE /citizen/declared_conditions?citizen_id=` leva `{ "condition" }` no **corpo** (dado de saúde nunca na URL nem no log de requisição); o spec do api testa assim.
- **T2 — `can_confirm`:** `GET /care_lines/teams` traz, por equipe, `can_confirm` (o usuário tem CBO de quem confirma com vínculo na unidade da equipe).
- **T3 — Busca de pessoa:** `POST /care_lines/person_lookup { "cpf" }` → paciente ou cidadão validado (como `patient_lookup` do 21a/22; trilha; limite de taxa; par só declarado → 409 `citizen_not_verified`).
- **T4 — Agendar pela linha:** `POST /care_lines/enrollments/:id/appointment_request` → `{ "request", "unit" }` cria o pedido de agendamento do módulo 17 para o paciente, na unidade da equipe.
- **T5 — Versão nova disponível:** `<care_line>` + `latest_template_version` (opcional; a tela também cruza com o catálogo-base).
- **T6 — Sessão do cidadão:** `care_lines` em `features` da sessão do wpda (conferir o mecanismo genérico).

**maintenance**
- **T7:** o catálogo-base não exige step-up (padrão do maintenance para o que só vale na próxima adoção); rótulo "Linhas de cuidado"; chave em todos os ambientes, sem simulado; `createdBy` opcional na versão; `all_of`, `code_filter`, `applies_to`, identidades de gênero e `declared` não são editáveis na tela — vêm da versão de partida e voltam iguais (mudar é pelo YAML do api).

## 12. Acréscimos (pré-voo da execução, 2026-10-22)

Prevalecem sobre as seções acima.

- **P1 — Busca de pessoa (forma de T3):** `POST /care_lines/person_lookup { "cpf" }` → `{ "person": { "patient_id"|null, "citizen_id"|null, "display_name", "cpf_masked", "verified": true } }`, **sem** `Patients::Resolve` (buscar não cria paciente; ele nasce na inscrição); 422 `invalid_cpf`, 409 `citizen_not_verified`, 404 `person_not_found`, 403 `missing_role`, 429 `too_many_requests` (30/min por usuário); trilha uma por pessoa mostrada.
- **P3 — Âncora e equipe na sugestão:** `<suggestion>` e `<enrollment>` trazem `care_line.requires_anchor`; `<suggestion>` traz `team_id`.
- **P4 — `createdBy`** (e-mail do mantenedor; nulo na carga do YAML) na versão do catálogo-base no GraphQL.
- **P5 — Detalhes:** (a) `GET /citizen/care` nunca traz item `untracked`; (b) fonte `oral_health` (não rastreada) e `exam:` restrito a SIGTAP do grupo 02; (c) `POST /citizen/declared_conditions` com `citizen_id` e `condition` no corpo; `DELETE` com `citizen_id` na URL e `condition` no corpo (T1); condição não vigente ou par alheio → 404 `not_found`; (d) pontos da boa prática inteiros de 0 a 100; (e) lista vazia vale como ausente nas regras (`problems: []`, `declared: []`); (f) profissional sem vínculo → 403 `missing_role` em `/care_lines/teams`.
- **P6 — Consulta puerperal (`c3_i`):** consulta com código de puerpério **ou** procedimento 0301010129 (como a nota); `consultation` é permitida como alternativa dentro de `all_of`.
- **P7 — Catálogo público:** testes do catálogo rodam fora do `bin/test` (exceção, como E13 do 22).
- **P8 — Condição declarada no wpda:** vale para todo par com `verification_level = verified` (inclusive validado por check-in); a seção aparece também abaixo do formulário de perfil.
- **P9 — Menu do dashboard:** o grupo "Linhas de cuidado" só aparece com `care_lines` **e** `clinical_record` utilizáveis na sessão.
- **P10 — Prova do maintenance:** a versão nova da C4 publicada na prova é permanente (imutabilidade); publicar com as mesmas regras, mudando só o nome, por último, avisando.
- **P11 — "Estou grávida":** opção visível para todos os sexos (homem trans pode gestar; C3/C7 já tratam identidade de gênero).
- **P2 — Agendar pela linha (decisão do usuário, 2026-10-22; forma de T4):** migração de cidade `20261022900003` no módulo 17: `appointment_requests` ganha `origin_care_line_enrollment_id`, `root_triage_id` passa a aceitar nulo e o check de origem aceita `care_line`; no máximo um pedido vivo por inscrição. `POST /care_lines/enrollments/:id/appointment_request` → `{ "request", "unit", "can_book" }` (`can_book` = `CitizenVerificationPolicy#verify?`; sem ele a tela diz "pedido enviado à recepção"); 409 `already_scheduled`, `invalid_transition`, `citizen_not_verified`; 422 `invalid_unit`. Pedidos existentes do 17 não mudam; a mudança exige a suíte completa do api.
