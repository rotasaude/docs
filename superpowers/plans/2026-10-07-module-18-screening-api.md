# Módulo 18 — Acolhimento (escuta inicial) (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A escuta inicial entre o check-in e a chamada — queixa em CIAP-2, sinais vitais com plausibilidade e alerta, cor CAB 28 sugerida por protocolo assinado (`kind: "screening"`), destino (`same_day`/`schedule`/`oriented`/`referred`) com a exceção na trava do módulo 13, filas por cor, reavaliação, trilha de leitura, as duas fichas LEDI (Atendimento Individual de escuta inicial e Procedimentos das aferições) com "fichas não geradas" e "gerar de novo", e o api#43 (fila LEDI só com códigos, purga de 90 dias, exclusão limpando a fila) — o lado api de F-18.1 a F-18.8 (ADR 0030).

**Architecture:** A Task 1 confirma o layout LEDI 8.7.0 contra os IDLs vendorizados e a documentação oficial e grava o resultado, linha a linha com a fonte, em `config/ledi/screening_mapping.yml` (lido por `Ledi::ScreeningMapping`); contradição com a spec para tudo. Duas migrações de cidade: `20261007300001` (escutas, revisões, escopo da unidade, desfechos e travas do atendimento, pedido `kind = screening`, `ledi_generation_failures`) e `20261007300002` (`ledi_outbox.last_error` → `last_error_codes`, `last_attempted_at`, `replaces_outbox_id`, índice único só das não recusadas). A parte clínica pura (`Screenings::VitalSigns`, `Screenings::RiskSuggestion`) é testada em tabela; os comandos (`Screenings::Start/Abandon/Complete/Reassess`) travam sempre atendimento → escuta, conferem papel, vínculo e CBO (`Screenings::Authorization`) e fecham o atendimento na mesma transação, com o trigger `attendances_screening_close_guard` como última palavra. As fichas implementam a interface `Ledi::Ficha` do módulo 16; `Ledi::ScreeningFicha` decide entre ficha e "não gerada" e é chamada pelo `Ledi::ScreeningFichaJob` (fechamento) e pelo `Ledi::ScreeningFichaSweepJob` (23h no fuso da cidade).

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade), RSpec, json_schemer, Thrift 0.17 (LEDI 8.7.0 vendorizado), Solid Queue (recorrência por cidade).

**Spec:** `docs/.claude/ciclo2/superpowers/specs/2026-10-07-module-18-screening-design.md` e `docs/.claude/ciclo2/adr/0030.md` (leia os dois antes de começar). Contratos entre apps: `docs/.claude/ciclo2/superpowers/plans/2026-10-07-module-18-screening-contracts.md` — fonte única dos formatos; os planos do `contracts` e do dashboard foram escritos contra ele: **não mude nomes, formatos nem códigos de erro**. O que o código real obrigou a precisar está em "Desvios da spec" e, quando toca formato, em "Divergências propostas ao contrato" (fim do arquivo). O schema novo nasce no repo `contracts` (tag `protocols-v1.6.0`, outro plano); a Task 2 copia o arquivo de lá. A revisão do ADR 0028 é do plano do docs, não deste.

## Desvios da spec (e precisões)

Onde a spec é omissa ou o código real obrigou a escolher:

1. **Layout confirmado (Task 1, 2026-10-07, ao escrever o plano).** O que a pesquisa achou e a Task 1 reconfere: no Atendimento Individual (MIAI) `tipoAtendimento = 4` ("Escuta inicial / Orientação"); o MIAI **não** aceita CBO `3222xx` (aceita 223505); a Ficha de Procedimentos (MIP) aceita `3222xx`, tem `statusEscutaInicialOrientacao` e `medicoes`, e **proíbe** o procedimento `03.01.04.007-9` (escuta inicial) na lista — a escuta do técnico vai pela marca, nunca pelo código. CPF e CNS do cidadão são **mutuamente exclusivos** nas duas fichas: vai só o CPF. MIAI exige `condutas` (1–12), `problemasCondicoes` (≥ 1, com `isAvaliado`) e `stCidadaoNaoPossuiCpf`. Glicemia capilar no LEDI vai até 800.
2. **CBO permitido = grupos da spec ∩ fichas possíveis.** `Screenings::Cbos.allowed?(cbo)` exige o prefixo de `config/scheduling/screening_cbos.yml` **e** que a ficha exista para ele (`Ledi::ScreeningMapping.exportable_cbo?`: está na tabela de CBOs do MIAI ou começa por `3222`). Ex.: `223293` (dentista da ESF) não está na tabela do MIAI → `cbo_not_allowed`. Assim nenhuma escuta concluída fica sem ficha possível por causa do CBO, e "não gerada" só tem os motivos da spec.
3. **Glicemia capilar plausível de 10 a 800 mg/dL** (a spec dá 10–1000 como exemplo; o LEDI recusa acima de 800). Os outros limites são os da spec. Alertas: sistólica ≥ 140, diastólica ≥ 90, FC > 100 ou < 50, FR > 24, temperatura ≥ 37,8, SpO2 < 95, glicemia < 70 ou ≥ 200, dor ≥ 7.
4. **Texto livre não é cifrado** (`complaint_note`, `color_change_reason`, `orientation_note`), como `attendances.referral_note` e `exception_reason`: as tabelas têm guarda de imutabilidade e a re-cifra regravaria as linhas. A proteção é de saída: nunca em evento, log (`filter_parameters`) nem Analytics, e a recepção nunca recebe.
5. **Quem fica registrado na escuta** é o último autor clínico: `Start` grava vínculo e CBO de quem iniciou; `Complete` e `Reassess` regravam `professional_link_id`/`cbo_code` com os de quem concluiu/reavaliou. A ficha usa esses. Qualquer profissional autorizado na unidade pode concluir, abandonar ou reavaliar.
6. **Escuta abandonada é reaproveitada.** `screenings.attendance_id` é único; `Start` sobre uma `abandoned` a devolve a `in_progress` com o novo autor (o trigger aceita só `abandoned → in_progress`).
7. **Chamada e saída com escuta em curso.** `Attendances::Call` (e `CallNext`) e `Attendances::Close` abandonam a escuta `in_progress` na mesma transação (ordem de travas atendimento → escuta); quem tentar concluir depois recebe 409 `not_in_progress`. `CallNext` segue a mesma ordem da fila (contratos §4) e **não** pula quem ainda espera escuta: cidade que não usa o acolhimento continua chamando como hoje.
8. **Ordem da fila do profissional:** primeiro quem tem escuta concluída (`same_day`), por cor (red, yellow, green, blue) e chegada; depois os demais na ordem de hoje (prioridade da triagem raiz, chegada). `waited_minutes` = minutos inteiros desde o check-in.
9. **Pedido `kind = screening`** leva `origin_attendance_id` (o atendimento) **e** `origin_screening_id`; a CHECK `ck_appointment_requests_origin` não muda. O índice único de `origin_screening_id` ignora pedidos `moved` (o esvaziamento de unidade copia a origem).
10. **Trava do módulo 13.** Além do trigger novo, `rota_attendance_guard` passa a aceitar `waiting → closed` com `scheduled_from_screening`, `oriented` ou `referred`, e `in_care → closed` só com desfecho fora de `left`, `scheduled_from_screening`, `oriented`; `ck_attendances_closing` passa a exigir chamada para desfecho clínico que não seja de escuta.
11. **`ck_ledi_outbox_rejected_error`** vira `status <> 'rejected' OR jsonb_array_length(last_error_codes) > 0 OR payload IS NULL`: a exclusão (e a purga) apagam conteúdo de recusada sem violar a CHECK.
12. **"Última tentativa"** dos 90 dias = coluna nova `ledi_outbox.last_attempted_at` (gravada em cada `accept!`/`reject!`/`retry_later!`; preenchida com `updated_at` onde `attempts > 0`). `updated_at` não serve: a re-cifra o move.
13. **Códigos de erro (`Ledi::ErrorCodes`)**: campo = último segmento conhecido da chave de `errosValidacao` (lista fechada; desconhecido → `other`); código por palavras da mensagem (`required`, `invalid`, `not_allowed`, `out_of_range`, `duplicate`, `unknown`) — a mensagem é lida em memória e descartada. Falhas de transporte: campo `transport`, códigos `http_error`, `unreachable`, `invalid_url`, `login_failed`, `internal_error`. `Ledi::ErrorText` e `Ledi::Outcome.message` saem.
14. **Uma ficha por escuta.** O fechamento e o varredor só geram se não existe linha na fila para `("Screening", screening_id)` (qualquer tipo de ficha). Ficha recusada só se regera pelo "Reenviar" do painel (`POST /production/fichas/:id/resend`), que para fonte `Screening` monta a ficha de novo a partir da escuta (uuid novo, `replaces_outbox_id`) em vez de reembrulhar o conteúdo antigo.
15. **Varredor** = `Ledi::ScreeningFichaSweepJob` (classe separada: recorrente por cidade com `EachCityJob`), agendado a cada hora e agindo só quando a hora local é 23. `Ledi::ScreeningFichaJob` (`CityScopedJob`) é o do fechamento.
16. **Ficha do técnico só com aferição** (spec §5). O LEDI aceitaria MIP só com a marca de escuta; ver "Divergências" (D10).
17. **Exclusão (`Citizens::Erase`)** chama `Ledi::CitizenSources.scrub!` para as linhas da fila cujas fontes são do par. Na prática é quase sempre vazio (quem tem escuta foi atendido e fica retido, ADR 0026); o teste é unitário no `scrub!` e de chamada no `Erase`.
18. **Nome reservado.** Protocolo `kind: "screening"` só com nome `acolhimento`; protocolo de triagem nunca com esse nome. O catálogo de triagens, a oferta ao cidadão e `StartTriage` ignoram protocolos `screening`.
19. **`schedule.due_in_days` ausente** usa o padrão da cor (yellow 7, green 15, blue 30); red sem padrão → `invalid_schedule`. Tipo do pedido tem de existir e estar ativo na cidade.
20. **`screening.viewed`** também é publicado quando a escuta sai no detalhe da chamada (`call`/`call_next`) — é leitura do dado clínico pelo profissional.

## Valores fixados por este plano (para o contrato §9)

Lugar único para copiar para o contrato. Decisões já aceitas pelo coordenador (contrato §9) estão incorporadas às tasks indicadas.

1. **Alertas de sinais vitais** (`revision.alerts` e `suggest.alerts`, ordem fixa; Task 6): `systolic_high` (sistólica ≥ 140), `diastolic_high` (diastólica ≥ 90), `heart_rate_high` (FC > 100), `heart_rate_low` (FC < 50), `respiratory_rate_high` (FR > 24), `temperature_high` (temperatura ≥ 37,8), `spo2_low` (SpO2 < 95), `glucose_low` (glicemia < 70), `glucose_high` (glicemia ≥ 200), `pain_severe` (dor ≥ 7).
2. **Plausibilidade** (422 `implausible_vital` com `field`; Task 6): sistólica 50–300, diastólica 20–200 e menor que a sistólica (`field: "diastolic"`), FC 20–250, FR 4–80, temperatura 30–45 (1 casa), SpO2 50–100, **glicemia 10–800 com `glucose_moment` obrigatório** (glicemia sem momento → `field: "glucose_moment"`; momento sem glicemia → `field: "capillary_glucose"`), peso 0,5–400 (2 casas), altura 30–250, dor 0–10; inteiros não aceitam fração; vírgula ou ponto decimal; string vazia = não medido; `vitals` que não é objeto → `field: "vitals"`; só uma das pressões → 422 `bp_incomplete`. `bmi` = peso/(altura/100)², 1 casa, nulo sem um dos dois; chave `bmi` enviada pelo cliente é ignorada.
3. **`last_error_codes`** (`[{ field, code }]`, até 20; Task 4): `field` ∈ `uuidFicha`, `headerTransport`, `profissionalCNS`, `cboCodigo_2002`, `cnes`, `ine`, `dataAtendimento`, `codigoIbgeMunicipio`, `cpfCidadao`, `cnsCidadao`, `cns`, `dataNascimento`, `dtNascimento`, `sexo`, `turno`, `localDeAtendimento`, `localAtendimento`, `tipoAtendimento`, `condutas`, `problemasCondicoes`, `ciap`, `medicoes`, `procedimentos`, `dataHoraInicialAtendimento`, `dataHoraFinalAtendimento`, `transport`, `other`; `code` ∈ `required`, `invalid`, `not_allowed`, `out_of_range`, `duplicate`, `http_error`, `unreachable`, `invalid_url`, `login_failed`, `internal_error`, `unknown`. Desconhecido → `{ "field": "other", "code": "unknown" }`. Falha de transporte → `field: "transport"`.
4. **`GET /production`** (Task 4): cada ficha `{ id, ficha_type, status, attempts, last_error_codes, created_at, accepted_at }`; `rejections: [{ field, code, count }]` (mais frequente primeiro, depois `field`, `code`); `ficha_type` ganha `atendimento_individual`.
5. **Pedido `kind: "screening"` na fila de pedidos do módulo 17** (`Scheduling::RequestJson`, sem mudança de código): `kind: "screening"`, `origin: "attendance"` (o pedido leva `origin_attendance_id`; a escuta fica em `origin_screening_id`, fora da forma), `origin_unit_name` = unidade do atendimento, `target_unit_id` = a mesma, `note: null`, `appointment_type_key`/`priority`/`due_on` do destino `schedule` (prazo padrão pela cor: yellow 7, green 15, blue 30; red exige `due_in_days`).
6. **Fila do profissional** (Task 11–12): `screening: { id, color, destination, waited_minutes } | null` (só escuta concluída; `waited_minutes` = minutos inteiros desde o check-in); a recepção recebe só esse bloco.
7. **Sugestão** (Task 12): `POST /attendance/screenings/suggest { ciap2_code?, vitals, attendance_id }` → `{ suggested_color, matched_rules: [{ index, text }], alerts, bmi }`; 404 `not_found` (atendimento), 403 `missing_role`/`missing_link` (vínculo ativo na unidade do atendimento), 422 `implausible_vital`/`bp_incomplete`/`invalid_ciap2`.
8. **CIAP-2** (Task 12): `POST /attendance/ciap2/search { q }` → `{ items: [{ code, label }] }` (até 20; código por prefixo ou nome sem acento/caixa; `q` vazio ou não texto → `[]`); 503 `terminology_unavailable` sem release ativa; só `health_professional`.
9. **Simulador do editor** (Task 12): `POST /authoring/protocols/simulate_screening { definition, vitals, ciap2_code, profile: { age, sex } }` → sempre 200 `{ suggested_color, matched_rules: [{ index, text }], errors, warnings }`; erros: `"schema: (root) object"`, `"definition is not a screening protocol"`, os do schema e do gate da variante, `"vitals: <reason> <field>"`; `warnings` sempre `[]` nesta entrega; autor e revisor (403 `missing_role` aos demais).
10. **Schema** (Task 2): `schema_version` inteiro (como hoje); a variante proíbe `start_step_id`, `steps`, `scoring`, `recommendations`, `priority_when`, `offer`, `suggestions`, `scheduling`; nome `acolhimento` só na variante (o gate recusa triagem com esse nome).
11. **Motivos de "não gerada"**: `unit_without_cnes`, `professional_without_team`, `professional_without_cns`, `citizen_without_birth_date`, `citizen_without_sex`, `unknown_ciap2` (na ordem em que aparecem em `reason_codes`).

## Global Constraints

- Tudo de domínio no banco de cada cidade: migração em `db/city_migrate`, dump à mão em `db/city_schema.rb` (a paridade compara o schema normalizado pelo Postgres, `spec/services/city_schema_spec.rb`), triggers em `db/city_triggers.sql` (a migração termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`; o dump não representa trigger). Migrações irreversíveis (`down` levanta `ActiveRecord::IrreversibleMigration`). Números `20261007300001` e `20261007300002`, maiores que todos os de `origin/main` (o último é `20261006210001`). Rollout: publicar a imagem e rodar `city:migrate:all` **antes** de cortar tráfego; **nunca** migrar fora do rake.
- Valores, exatamente (contratos §2): cor `red` | `yellow` | `green` | `blue` (gravidade nessa ordem); destino `same_day` | `schedule` | `oriented` | `referred`; escuta `in_progress` | `completed` | `abandoned`; `screening_scope` `walk_in` (padrão) | `all`; desfechos novos `scheduled_from_screening`, `oriented`; `glucose_moment` `fasting` | `postprandial` | `random`; `appointment_requests.kind` ganha `screening`. Nome reservado do protocolo: `acolhimento`; `risk_rules` 1–50.
- CBOs (spec §3.2): grupos `2251`, `2252`, `2253`, `2235`, `2234`, `2516`, `2237`, `2236`, `2232` e `3222`, em `config/scheduling/screening_cbos.yml` (com o recorte do Desvio 2).
- Limites (spec §3.1): sistólica 50–300, diastólica 20–200 e < sistólica, FC 20–250, FR 4–80, temperatura 30–45 (`decimal(3,1)`), SpO2 50–100, glicemia 10–800 (Desvio 3), peso 0,5–400 (`decimal(5,2)`), altura 30–250, dor 0–10; pressão "ambos ou nenhum"; glicemia exige momento. Textos: `complaint_note`, `orientation_note` ≤ 500; `color_change_reason` 10–500 quando a cor final difere da sugerida.
- Prazo padrão do `schedule` pela cor: yellow 7, green 15, blue 30; red sem padrão.
- Motivos de "não gerada" (lista fechada): `unit_without_cnes`, `professional_without_team`, `professional_without_cns`, `citizen_without_birth_date`, `citizen_without_sex`, `unknown_ciap2`.
- Erros `{ "error": "<reason>" }` (com `field` quando dito); `render_failure` de `AttendanceAccess`; step-up 401 `{ "error": "mfa_required" }`; papel ausente 403.
- Eventos de domínio só com ids (contratos §7), declarados em `config/initializers/domain_events.rb` com `to: []` e em `spec/initializers/domain_events_bindings_spec.rb`: `screening.started`, `screening.abandoned`, `screening.completed`, `screening.reassessed`, `screening.viewed`, `ledi.generation_failed`, `ledi.generation_retried`, `ledi.payload_purged`. Nenhum texto livre em evento, log ou Analytics. Nenhum `Platform.audit` novo (nada em `R18_PLATFORM_EVENT_NAMES`).
- Jobs por cidade só via `prepend EachCityJob` (recorrente) ou `include CityScopedJob`; `Current.city` **nunca** é atribuído em `app/` ou `lib/` (`spec/architecture/current_city_assignment_spec.rb`).
- Specs de request com `type: :request`; arquivo novo em `spec/support/` precisa de `require_relative` em `spec/rails_helper.rb`. Specs com threads: `self.use_transactional_tests = false`, `Queue#pop(timeout:)`, `after` que solta as threads e apaga tudo o que commitou (com `session_replication_role = replica`, como `spec/commands/attendances/call_next_concurrency_spec.rb`).
- `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..30)`.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). `git add` sempre com caminhos explícitos (nunca `-A`).

## Ambiente de execução

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API"). Ordem de merge: `contracts` (tag `protocols-v1.6.0`) → api → dashboard.
- Worktree (a partir da raiz do monorepo, `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod18 -b feat/mod-18-screening origin/main
  cp apps/api/config/master.key apps/api/.claude/mod18/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod18`. Todo comando Rails/RSpec:

  ```bash
  docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec <arquivos>
  ```

- Todo `git add`/`git commit` usa `-C apps/api/.claude/mod18`.
- Depois de cada migração de cidade (Tasks 3 e 4): `DROP DATABASE` dos dois bancos de teste de cidade e `city:test_databases` de novo:

  ```bash
  psql -U rota_saude -d postgres -c "DROP DATABASE rota_saude_test_city_a" -c "DROP DATABASE rota_saude_test_city_b"
  docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod18 api bin/rails city:test_databases
  ```

  Ao voltar para a main, repita (o banco de teste fica à frente).
- Suíte completa só com o worker parado e sem outra sessão rodando suíte:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec
  docker compose start worker
  ```

- A Task 1 pode usar WebFetch (só leitura) na documentação oficial `https://integracao.esusaps.bridge.ufsc.tech`.
- Prova no navegador e plano do dashboard: o api do worktree sobe na porta **3035** (Task 18).

## Review Focus

1. **Duas profissionais (ou a escuta e a chamada) na mesma pessoa ao mesmo tempo:** só uma escuta nasce; a outra recebe 409 `already_screening`; se o médico chama a pessoa no meio da escuta, a escuta vira `abandoned` e quem tenta concluir recebe 409 `not_in_progress` — nunca 500, nunca atendimento fechado duas vezes. Testes: Task 10 ("duas iniciam juntas", "a chamada vence a conclusão").
2. **Sinais vitais nas bordas:** vírgula decimal (`"37,9"`), string vazia, só a sistólica, `0`, o limite exato, inteiro com fração (`120.5` na pressão), glicemia sem momento, diastólica maior que a sistólica, chave desconhecida, `vitals` que não é objeto → 422 `implausible_vital` (com `field`) ou `bp_incomplete`, nunca 500. Testes: Task 6 (tabela de casos) e Task 12 ("vitals lixo vira 422").
3. **Sem protocolo ativo, sinal não medido, protocolo de triagem chamado `acolhimento`:** sem sugestão a cor final é livre e sem justificativa; regra que usa sinal não medido é falsa (não casa); o nome reservado é recusado no gate; o acolhimento nunca aparece no catálogo de triagens do cidadão. Testes: Task 7 ("sem protocolo", "sinal ausente") e Task 2 ("acolhimento fora do catálogo", "nome reservado").
4. **Atendimento encerrado `left` (ou chamado) com escuta em curso, e fechado depois pelo profissional:** a escuta vira `abandoned`; só escuta `completed` gera ficha; o fechamento e o varredor das 23h nunca geram duas fichas. Testes: Task 8 ("left abandona a escuta") e Task 14 ("fechamento depois do varredor não duplica").
5. **Exportação mudando entre a conclusão e o fechamento, e a recusada regenerada:** desligada no fechamento → nenhuma ficha (nem "não gerada"); ligada no varredor → nasce; "Reenviar" de recusada já regenerada → 409; o varredor age às 23h no fuso da cidade (Manaus), não no de São Paulo. Testes: Task 14 ("exportação desligada no fechamento", "23h em Manaus") e Task 15 ("regenerar duas vezes").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `config/ledi/screening_mapping.yml`, `app/services/ledi/screening_mapping.rb` | layout LEDI confirmado, com fonte por linha | 1 |
| `config/protocols/schema.json`, `app/protocols/validation/screening.rb`, `app/protocols/validation/condition.rb`, `app/protocols/condition_context.rb`, `app/protocols/gate.rb`, `app/protocols/validator.rb`, `app/models/protocol_definition.rb`, `app/services/triages/{offer,catalog_admin}.rb`, `app/commands/start_triage.rb`, `app/controllers/authoring/protocols_controller.rb`, `app/services/protocols/condition_text.rb` | protocolo `screening` e variáveis | 2 |
| `db/city_migrate/20261007300001_add_screenings.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `config/scheduling/screening_cbos.yml`, `app/models/{screening,screening_revision,ledi_generation_failure}.rb`, `app/models/{attendance,appointment_request,health_unit}.rb`, `app/commands/health_units/drain.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/support/screening_helpers.rb`, `spec/adr_pointers_spec.rb` | dados das escutas | 3 |
| `db/city_migrate/20261007300002_ledi_outbox_error_codes.rb`, `app/services/ledi/{error_codes,outcome,delivery}.rb`, `app/models/ledi_outbox_entry.rb`, `app/commands/ledi/{enqueue,resend}.rb`, `app/queries/ledi/production_summary.rb`, `app/controllers/production_controller.rb`, `lib/ledi_crew.rb` | api#43: códigos na fila | 4 |
| `app/jobs/ledi/purge_stale_payloads_job.rb`, `app/services/ledi/citizen_sources.rb`, `app/commands/citizens/erase.rb`, `config/recurring.yml` | api#43: purga e exclusão | 5 |
| `app/services/screenings/vital_signs.rb` | sinais vitais (puro) | 6 |
| `app/services/screenings/{risk_suggestion,active_protocol,suggest,ciap2}.rb` | cor sugerida | 7 |
| `app/services/screenings/{cbos,scope,authorization}.rb`, `app/commands/screenings/{start,abandon}.rb`, `app/commands/attendances/{call,close}.rb` | iniciar e abandonar | 8 |
| `app/services/screenings/revision_input.rb`, `app/commands/screenings/{complete,reassess}.rb` | concluir, destino, reavaliar | 9 |
| `spec/commands/screenings/screening_concurrency_spec.rb` | corridas | 10 |
| `app/services/screenings/{queue,json}.rb`, `app/commands/attendances/unit_queue.rb` | filas | 11 |
| `app/controllers/{screenings,ciap2_codes,attendances,health_units}_controller.rb`, `config/routes.rb` | API | 12 |
| `app/services/ledi/fichas/{initial_listening,screening_procedures,medicoes}.rb`, `app/services/ledi/{ficha_types,version}.rb` | as duas fichas | 13 |
| `app/services/ledi/screening_ficha.rb`, `app/jobs/ledi/{screening_ficha_job,screening_ficha_sweep_job}.rb`, `app/commands/ledi/enqueue.rb`, `app/commands/screenings/complete.rb`, `app/commands/attendances/close.rb`, `config/recurring.yml` | geração e "não gerada" | 14 |
| `app/controllers/production_controller.rb`, `app/commands/ledi/resend.rb`, `config/routes.rb` | "não geradas", "gerar de novo", regenerar recusada | 15 |
| `spec/invariants/screening_invariants_spec.rb` | invariantes do ADR 0030 | 16 |
| `lib/screening_crew.rb`, `db/seeds.rb` | semente de dev | 17 |
| — | revisão final, suíte, porta 3035 | 18 |

---
## Fatia 0 — Layout LEDI (F-18.5, pré-condição)

### Task 1: Confirmação do layout LEDI e `config/ledi/screening_mapping.yml`

Nada de ficha é escrito antes desta task passar. Ela fixa, com a fonte de cada valor, o que as Tasks 13 e 14 usam. **Critério de parada:** se qualquer uma das confirmações do Step 2 contradisser a coluna "Esperado" (que é o desenho da spec §5 e o que a pesquisa achou em 2026-10-07), **pare**, não escreva o mapeamento e reporte ao coordenador com a URL e o trecho — a spec diz que a confirmação "pode mudar o mapeamento CBO → ficha".

**Files:**
- Create: `config/ledi/screening_mapping.yml`
- Create: `app/services/ledi/screening_mapping.rb`
- Test: `spec/config/ledi/screening_mapping_spec.rb`

**Interfaces:**
- Produces (`Ledi::ScreeningMapping`, módulo de funções, lê o YAML uma vez por processo):
  - `value(key) -> Object` (levanta `Ledi::ScreeningMapping::Missing` para chave inexistente), `entry(key) -> { "value", "source" }`;
  - `miai_cbos -> Array<String>`, `miai_cbo?(cbo) -> bool`, `procedures_cbo?(cbo) -> bool` (prefixo `3222`), `exportable_cbo?(cbo) -> bool`;
  - `conduta(destination) -> Integer`, `sex_code("female"|"male") -> Integer`, `glucose_code(moment) -> Integer`, `measurement_field(column) -> String` (nome do campo em `MedicoesThrift`), `procedure(kind) -> String` (SIGTAP, 10 dígitos), `turno(time) -> Integer`.
  - Chaves usadas pelas Tasks 13–14: `tipo_dado_serializado.atendimento_individual`, `initial_listening.tipo_atendimento`, `initial_listening.local_de_atendimento`, `initial_listening.conduta.<destino>`, `screening_procedures.status_escuta_inicial_orientacao`, `screening_procedures.local_atendimento`, `screening_procedures.forbidden_procedure`, `procedure.{blood_pressure,capillary_glucose,temperature,weight,height}`, `measurement.<coluna>`, `measurement_limit.capillary_glucose_max`, `glucose_moment.<momento>`, `sex.<sexo>`, `turno.{morning,afternoon,night}`, `citizen_identifier`.

- [ ] **Step 1: Leia os IDLs vendorizados**

```bash
cd apps/api/.claude/mod18
cat vendor/ledi/8.7.0/SOURCE | head -3
sed -n '/struct FichaAtendimentoIndividualChildThrift/,/^}/p' vendor/ledi/8.7.0/idl/ras/ficha_atendimento_individual.thrift
sed -n '/struct FichaProcedimentoChildThrift/,/^}/p' vendor/ledi/8.7.0/idl/ras/ficha_atendimento_procedimento.thrift
sed -n '/struct MedicoesThrift/,/^}/p;/struct ProblemaCondicaoThrift/,/^}/p;/struct VariasLotacoesHeaderThrift/,/^}/p' vendor/ledi/8.7.0/idl/ras/common.thrift
```
Expected: `ledi_version: 8.7.0`; o filho do MIAI tem `7:optional i64 tipoAtendimento`, `22:optional list<i64> condutas`, `30:optional string cpfCidadao`, `39:optional common.MedicoesThrift medicoes`, `40:optional list<common.ProblemaCondicaoThrift> problemasCondicoes`, `47:optional bool stCidadaoNaoPossuiCpf`; o mestre do MIAI usa `common.VariasLotacoesHeaderThrift headerTransport`; o filho do MIP tem `7:optional bool statusEscutaInicialOrientacao`, `8:optional list<string> procedimentos`, `16:optional common.MedicoesThrift medicoes`; `MedicoesThrift` tem `pressaoArterialSistolica`(3), `pressaoArterialDiastolica`(4), `frequenciaRespiratoria`(5), `frequenciaCardiaca`(6), `temperatura`(7), `saturacaoO2`(8), `glicemiaCapilar`(9), `tipoGlicemiaCapilar`(10), `peso`(11), `altura`(12); `ProblemaCondicaoThrift` tem `ciap`(4) e `isAvaliado`(9).

- [ ] **Step 2: Confirme na documentação oficial (WebFetch, só leitura)**

| # | Pergunta | URL | Esperado (achado em 2026-10-07) |
|---|---|---|---|
| a | Código de tipo de atendimento da escuta inicial no MIAI | `https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/referencias/dicionario.html` (TipoDeAtendimento) e `.../estrutura_arquivos/dicionario-fai.html` (#7 só aceita 1, 2, 4, 5, 6) | `4` = "Escuta inicial / Orientação" |
| b | Campos de medição (nome, unidade, limite) | `.../estrutura_arquivos/dicionario-fai.html` e `.../dicionario-fp.html` (medicoes) | os nomes do Step 1; temperatura 20,0–45,0 (1 casa); peso 0,5–500 (3 casas); altura 20–250 (1 casa); glicemia 0–800 com `tipoGlicemiaCapilar` obrigatório junto; sistólica e diastólica 0–999, uma exige a outra; SpO2 0–100 |
| c | O MIAI aceita CBO 3222xx? | `.../documentacao/regras/cbo.html` (Grupos de CBOs por Modelo de Informação) | **não** (nem 322205, 322210, 322230, 322245, 322250); 223505, 225125, 225142 **sim**; a tabela do MIP tem 322205, 322230, 322245 |
| d | SIGTAP das aferições no MIP | `.../dicionario-fp.html` (Procedimentos do Modelo de Informação) e a Tabela Unificada (`http://sigtap.datasus.gov.br`) | temperatura `0301100250` (03.01.10.025-0, ABPG034), peso `0101040083` (01.01.04.008-3, ABPG039), altura `0101040075` (01.01.04.007-5, ABPG038), PA `0301100039` (03.01.10.003-9, já usado pela ficha sintética do módulo 16), glicemia capilar `0214010015` (02.14.01.001-5); `0301040079` (escuta inicial) **proibido** na lista |
| e | Escuta inicial no MIP | `.../dicionario-fp.html` | `statusEscutaInicialOrientacao` "identifica se o registro de procedimentos foi utilizado para representar uma escuta inicial"; com ela `true`, `procedimentos` deixa de ser obrigatório |
| f | CPF × CNS do cidadão | `.../dicionario-fai.html` e `.../dicionario-fp.html` | mutuamente exclusivos; `stCidadaoNaoPossuiCpf` obrigatório no MIAI |
| g | Condutas do MIAI | `.../referencias/dicionario.html` (CondutaEncaminhamento) | `11` encaminhamento interno no dia, `1` retorno para consulta agendada, `9` alta do episódio, `4` encaminhamento para serviço especializado |
| h | Tabelas de códigos | `.../referencias/dicionario.html` | Sexo: `0` masculino, `1` feminino; Turno: `1` manhã, `2` tarde, `3` noite; LocalDeAtendimento `1` = UBS; TipoGlicemiaCapilar `0` jejum, `1` pós-prandial, `3` não especificado; TipoDadoSerializado `4` = Atendimento Individual, `7` = Procedimentos |

Transcreva também, da tabela "Modelo de Informação de Atendimento Individual" de `regras/cbo.html`, **todos** os CBOs da tabela que comecem por `2251`, `2252`, `2253`, `2232`, `2234`, `2235`, `2236`, `2237`, `2516` para `miai_cbos` (o Step 4 traz os que a pesquisa já leu; confira cada um e acrescente os que faltarem). **Pare** se: (a) não for `4`; (c) a tabela do MIAI tiver algum `3222xx` ou não tiver `223505`; (d) algum dos cinco códigos não existir na competência SIGTAP vigente ou estiver fora dos grupos aceitos pelo MIP; (e) o MIP não aceitar a escuta sem procedimento quando há aferição; (f) CPF e CNS puderem ir juntos (aí a spec "CPF (e CNS)" vale e o Step 4 muda — reporte).

- [ ] **Step 3: Escreva a spec que falha (arquivo ainda não existe)**

```ruby
# spec/config/ledi/screening_mapping_spec.rb
require "rails_helper"

# ADR 0030 / spec §5 (Tarefa 1): o mapeamento da escuta para o LEDI 8.7.0 tem
# fonte em cada linha e bate com os IDLs vendorizados.
RSpec.describe Ledi::ScreeningMapping do
  def idl_fields(file, struct)
    text = Rails.root.join("vendor/ledi/8.7.0/idl", file).read
    body = text[/struct\s+#{struct}\s*\{(.*?)\n\}/m, 1] or raise "#{struct} não está em #{file}"
    body.scan(/^\s*\d+:\s*(?:required|optional)\s+[\w.<>]+\s+(\w+)/).flatten
  end

  it "é da versão ativa e toda entrada tem valor e fonte" do
    expect(described_class.data.fetch("ledi_version")).to eq(Ledi::Version::ACTIVE)
    described_class.data.fetch("entries").each do |key, entry|
      expect(entry.keys).to contain_exactly("value", "source"), key
      expect(entry["source"].to_s.strip).not_to be_empty, key
    end
    expect(described_class.data.dig("miai_cbos", "source")).to be_present
    expect(described_class.data.dig("procedures_cbo_prefixes", "source")).to be_present
  end

  it "escuta inicial no MIAI é tipo 4, local UBS, e cada destino tem conduta" do
    expect(described_class.value("initial_listening.tipo_atendimento")).to eq(4)
    expect([ 1, 2, 4, 5, 6 ]).to include(described_class.value("initial_listening.tipo_atendimento"))
    expect(described_class.value("initial_listening.local_de_atendimento")).to eq(1)
    expect(%w[same_day schedule oriented referred].map { |d| described_class.conduta(d) }).to eq([ 11, 1, 9, 4 ])
    expect(described_class.value("tipo_dado_serializado.atendimento_individual")).to eq(4)
  end

  it "os campos de medição existem em MedicoesThrift" do
    fields = idl_fields("ras/common.thrift", "MedicoesThrift")
    columns = %w[systolic diastolic heart_rate respiratory_rate temperature_c spo2 capillary_glucose glucose_moment weight_kg height_cm]
    expect(columns.map { |c| described_class.measurement_field(c) }).to all(satisfy { |f| fields.include?(f) })
    expect(described_class.value("measurement_limit.capillary_glucose_max")).to eq(800)
  end

  it "o MIP marca a escuta, usa só SIGTAP de 10 dígitos e nunca o código da escuta" do
    expect(idl_fields("ras/ficha_atendimento_procedimento.thrift", "FichaProcedimentoChildThrift"))
      .to include("statusEscutaInicialOrientacao", "procedimentos", "medicoes")
    expect(described_class.value("screening_procedures.status_escuta_inicial_orientacao")).to be(true)
    codes = %w[blood_pressure capillary_glucose temperature weight height].map { |k| described_class.procedure(k) }
    expect(codes).to all(match(/\A\d{10}\z/))
    expect(codes).not_to include(described_class.value("screening_procedures.forbidden_procedure"))
  end

  it "CBO: o MIAI tem enfermeiro e médicos e nenhum técnico; 3222 vai para o MIP" do
    expect(described_class.miai_cbos).to include("223505", "225125", "225142")
    expect(described_class.miai_cbos).to all(match(/\A[0-9A-Z]{6}\z/))
    expect(described_class.miai_cbos.grep(/\A3222/)).to be_empty
    expect(described_class.miai_cbo?("322205")).to be(false)
    expect(described_class.procedures_cbo?("322205")).to be(true)
    expect(described_class.exportable_cbo?("223505")).to be(true)
    expect(described_class.exportable_cbo?("223293")).to eq(described_class.miai_cbos.include?("223293"))
  end

  it "códigos de sexo, glicemia e turno; CPF é o identificador" do
    expect([ described_class.sex_code("male"), described_class.sex_code("female") ]).to eq([ 0, 1 ])
    expect(%w[fasting postprandial random].map { |m| described_class.glucose_code(m) }).to eq([ 0, 1, 3 ])
    Time.use_zone("America/Sao_Paulo") do
      expect([ 8, 13, 19 ].map { |h| described_class.turno(Time.zone.parse("2026-10-07 #{h}:00")) }).to eq([ 1, 2, 3 ])
    end
    expect(described_class.value("citizen_identifier")).to eq("cpf")
  end

  it "chave inexistente levanta" do
    expect { described_class.value("nao.existe") }.to raise_error(described_class::Missing)
  end
end
```

- [ ] **Step 4: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/config/ledi/screening_mapping_spec.rb`
Expected: FAIL com `uninitialized constant Ledi::ScreeningMapping`.

- [ ] **Step 5: Escreva o mapeamento (com a fonte de cada linha) e o leitor**

```yaml
# config/ledi/screening_mapping.yml
# Escuta inicial (ADR 0030; spec 2026-10-07 §5, Tarefa 1). Cada valor tem a
# fonte em que foi conferido. Mudar uma linha = reconferir a fonte e rodar
# spec/config/ledi/screening_mapping_spec.rb. Base: https://integracao.esusaps.bridge.ufsc.tech
ledi_version: "8.7.0"
checked_on: "2026-10-07"
entries:
  tipo_dado_serializado.atendimento_individual:
    value: 4
    source: "ledi/documentacao/referencias/dicionario.html — TipoDadoSerializado: 4 Modelo de Informação de Atendimento Individual"
  tipo_dado_serializado.procedimento:
    value: 7
    source: "ledi/documentacao/referencias/dicionario.html — TipoDadoSerializado: 7 Modelo de Informação de Procedimentos"
  initial_listening.tipo_atendimento:
    value: 4
    source: "referencias/dicionario.html — TipoDeAtendimento 4 'Escuta inicial / Orientação'; estrutura_arquivos/dicionario-fai.html #7 aceita 1, 2, 4, 5, 6"
  initial_listening.local_de_atendimento:
    value: 1
    source: "referencias/dicionario.html — LocalDeAtendimento 1 UBS"
  initial_listening.conduta.same_day:
    value: 11
    source: "referencias/dicionario.html — CondutaEncaminhamento 11 'Encaminhamento interno no dia'"
  initial_listening.conduta.schedule:
    value: 1
    source: "referencias/dicionario.html — CondutaEncaminhamento 1 'Retorno para consulta agendada'"
  initial_listening.conduta.oriented:
    value: 9
    source: "referencias/dicionario.html — CondutaEncaminhamento 9 'Alta do episódio'"
  initial_listening.conduta.referred:
    value: 4
    source: "referencias/dicionario.html — CondutaEncaminhamento 4 'Encaminhamento para serviço especializado'"
  screening_procedures.status_escuta_inicial_orientacao:
    value: true
    source: "estrutura_arquivos/dicionario-fp.html — statusEscutaInicialOrientacao: identifica registro de procedimentos que representa uma escuta inicial"
  screening_procedures.local_atendimento:
    value: 1
    source: "referencias/dicionario.html — LocalDeAtendimento 1 UBS"
  screening_procedures.forbidden_procedure:
    value: "0301040079"
    source: "estrutura_arquivos/dicionario-fp.html — procedimentos: não pode ser preenchido com 03.01.04.007-9"
  procedure.blood_pressure:
    value: "0301100039"
    source: "SIGTAP 03.01.10.003-9 AFERIÇÃO DE PRESSÃO ARTERIAL (Tabela Unificada; já usado por Ledi::Fichas::Synthetic, módulo 16)"
  procedure.capillary_glucose:
    value: "0214010015"
    source: "SIGTAP 02.14.01.001-5 GLICEMIA CAPILAR (Tabela Unificada, grupo 02 aceito pelo MIP)"
  procedure.temperature:
    value: "0301100250"
    source: "estrutura_arquivos/dicionario-fp.html — Procedimentos do Modelo de Informação: 03.01.10.025-0 Aferição de temperatura (ABPG034)"
  procedure.weight:
    value: "0101040083"
    source: "estrutura_arquivos/dicionario-fp.html — 01.01.04.008-3 Medição de peso (ABPG039)"
  procedure.height:
    value: "0101040075"
    source: "estrutura_arquivos/dicionario-fp.html — 01.01.04.007-5 Medição de altura (ABPG038)"
  measurement.systolic:
    value: pressaoArterialSistolica
    source: "vendor/ledi/8.7.0/idl/ras/common.thrift MedicoesThrift #3 (i32, mmHg)"
  measurement.diastolic:
    value: pressaoArterialDiastolica
    source: "common.thrift MedicoesThrift #4 (i32, mmHg); dicionário: uma exige a outra"
  measurement.respiratory_rate:
    value: frequenciaRespiratoria
    source: "common.thrift MedicoesThrift #5 (i32, 0–200)"
  measurement.heart_rate:
    value: frequenciaCardiaca
    source: "common.thrift MedicoesThrift #6 (i32, 0–999)"
  measurement.temperature_c:
    value: temperatura
    source: "common.thrift MedicoesThrift #7 (double, 20.0–45.0, 1 casa)"
  measurement.spo2:
    value: saturacaoO2
    source: "common.thrift MedicoesThrift #8 (i32, 0–100)"
  measurement.capillary_glucose:
    value: glicemiaCapilar
    source: "common.thrift MedicoesThrift #9 (i32, 0–800; exige tipoGlicemiaCapilar)"
  measurement.glucose_moment:
    value: tipoGlicemiaCapilar
    source: "common.thrift MedicoesThrift #10 (i64)"
  measurement.weight_kg:
    value: peso
    source: "common.thrift MedicoesThrift #11 (double, 0.5–500, 3 casas)"
  measurement.height_cm:
    value: altura
    source: "common.thrift MedicoesThrift #12 (double, 20–250, 1 casa)"
  measurement_limit.capillary_glucose_max:
    value: 800
    source: "estrutura_arquivos/dicionario-fai.html e dicionario-fp.html — glicemiaCapilar: valor máximo 800"
  glucose_moment.fasting:
    value: 0
    source: "referencias/dicionario.html — TipoGlicemiaCapilar 0 Jejum"
  glucose_moment.postprandial:
    value: 1
    source: "referencias/dicionario.html — TipoGlicemiaCapilar 1 Pós-prandial"
  glucose_moment.random:
    value: 3
    source: "referencias/dicionario.html — TipoGlicemiaCapilar 3 Não especificado"
  sex.male:
    value: 0
    source: "referencias/dicionario.html — Sexo 0 Masculino"
  sex.female:
    value: 1
    source: "referencias/dicionario.html — Sexo 1 Feminino"
  turno.morning:
    value: 1
    source: "referencias/dicionario.html — Turno 1 Manhã (antes das 12h, fuso da cidade)"
  turno.afternoon:
    value: 2
    source: "referencias/dicionario.html — Turno 2 Tarde (12h–18h)"
  turno.night:
    value: 3
    source: "referencias/dicionario.html — Turno 3 Noite (a partir das 18h)"
  citizen_identifier:
    value: cpf
    source: "dicionario-fai.html e dicionario-fp.html — cpfCidadao e cnsCidadao não podem ir juntos; CPF é o identificador primário desde o LEDI 8.4.0"
miai_cbos:
  source: "ledi/documentacao/regras/cbo.html — tabela 'Modelo de Informação de Atendimento Individual' (só os grupos da escuta, spec §3.2)"
  codes: [
    "225103", "225105", "225109", "225110", "225112", "225118", "225120", "225121", "225124", "225125", "225127",
    "225130", "225133", "225135", "225136", "225139", "225140", "225142", "225154", "225155", "225160", "225165",
    "225170", "225180", "225185", "225195", "225250", "225255", "225265", "225270", "225275", "225285", "225350",
    "223208", "223405", "223415", "223425", "223430", "223445", "223505", "223530", "223545", "223550", "223555",
    "223560", "223565", "223605", "223650", "223710", "251505", "251510", "251530", "251540", "251545", "251550",
    "251555", "251605"
  ]
procedures_cbo_prefixes:
  source: "ledi/documentacao/regras/cbo.html — tabela 'Modelo de Informação de Procedimentos' contém 322205, 322210, 322230, 322245, 322250"
  prefixes: [ "3222" ]
```

(`miai_cbos` acima é o que a pesquisa leu da tabela; no Step 2 confira cada código e acrescente os que a tabela tiver nos grupos da escuta e faltarem aqui — a spec só exige 223505/225125/225142 presentes e nenhum `3222`.)

```ruby
# app/services/ledi/screening_mapping.rb
# Mapeamento da escuta inicial para o LEDI (ADR 0030; spec §5, Tarefa 1). O
# YAML guarda valor e fonte de cada linha; quem monta ficha lê daqui, nunca de
# literal solto. Carregado uma vez por processo.
module Ledi
  module ScreeningMapping
    PATH = Rails.root.join("config/ledi/screening_mapping.yml")

    class Missing < StandardError; end

    module_function

    def data = (@data ||= YAML.load_file(PATH).freeze)

    def entry(key) = data.fetch("entries").fetch(key.to_s) { raise Missing, key.to_s }

    def value(key) = entry(key).fetch("value")

    def miai_cbos = data.fetch("miai_cbos").fetch("codes")

    def miai_cbo?(cbo) = miai_cbos.include?(cbo.to_s)

    def procedures_cbo?(cbo)
      data.fetch("procedures_cbo_prefixes").fetch("prefixes").any? { |prefix| cbo.to_s.start_with?(prefix) }
    end

    def exportable_cbo?(cbo) = miai_cbo?(cbo) || procedures_cbo?(cbo)

    def conduta(destination) = value("initial_listening.conduta.#{destination}")
    def sex_code(sex) = value("sex.#{sex}")
    def glucose_code(moment) = value("glucose_moment.#{moment}")
    def measurement_field(column) = value("measurement.#{column}")
    def procedure(kind) = value("procedure.#{kind}")

    # Turno no fuso da cidade (Time.zone dentro de CityConnection.with).
    def turno(time)
      hour = time.in_time_zone.hour
      value(if hour < 12 then "turno.morning" elsif hour < 18 then "turno.afternoon" else "turno.night" end)
    end
  end
end
```

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/config/ledi/screening_mapping_spec.rb`
Expected: PASS (7 exemplos).

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add config/ledi/screening_mapping.yml app/services/ledi/screening_mapping.rb spec/config/ledi/screening_mapping_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: confirm the LEDI layout for the initial listening

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 1 — Protocolo de acolhimento (F-18.2, parte do schema)

### Task 2: Schema `protocols-v1.6.0`, gate de `screening`, variáveis `vitals.*`/`complaint.ciap2` e o acolhimento fora do catálogo de triagens

**Files:**
- Modify: `config/protocols/schema.json`
- Create: `app/protocols/validation/screening.rb`
- Modify: `app/protocols/validation/condition.rb`, `app/protocols/condition_context.rb`, `app/protocols/gate.rb`, `app/protocols/validator.rb`
- Modify: `app/models/protocol_definition.rb`, `app/services/triages/offer.rb`, `app/services/triages/catalog_admin.rb`, `app/commands/start_triage.rb`, `app/controllers/authoring/protocols_controller.rb`, `app/services/protocols/condition_text.rb`
- Modify: `spec/protocols/gate_spec.rb:120`, `spec/protocols/condition_context_spec.rb`
- Test: `spec/protocols/schema_screening_spec.rb`, `spec/protocols/validation/screening_spec.rb`, `spec/protocols/condition_context_screening_spec.rb`, `spec/services/triages/offer_screening_spec.rb`

**Interfaces:**
- Consumes: `Protocols::Validation::Condition.errors(node, by_id, variables:)`, `Protocols::ConditionContext.build(...)` (módulo 15).
- Produces:
  - `Protocols::Validation::Screening::NAME == "acolhimento"`, `::COLORS == %w[red yellow green blue]`, `::MAX_RULES == 50`, `::VARIABLES` (as 15 do contrato §1), `.screening?(definition) -> bool`, `.call(definition) -> Array<String>` (total), `.reserved_name_errors(definition) -> Array<String>`;
  - `Protocols::Validation::Condition::VARIABLES` com `vitals.*` (`:number`, exceto `vitals.glucose_moment` `:glucose_moment`) e `complaint.ciap2` (`:ciap2`);
  - `Protocols::ConditionContext.build(answers: {}, profile: {}, outcome: {}, citizen: {}, vitals: {}, complaint: {})` (chaves `vitals.<campo>` e `complaint.ciap2`); `RESERVED_PREFIXES` com `vitals.` e `complaint.`;
  - `ProtocolDefinition.triage_protocols` e `.screening_protocols` (scopes);
  - `Protocols::ConditionText::LABELS` com os rótulos dos sinais.

- [ ] **Step 1: Confira a tag do `contracts` e copie o schema**

```bash
/opt/homebrew/bin/git -C contracts fetch --tags
/opt/homebrew/bin/git -C contracts show protocols-v1.6.0:protocols/schema.json > apps/api/.claude/mod18/config/protocols/schema.json
/opt/homebrew/bin/git -C apps/api/.claude/mod18 diff --stat config/protocols/schema.json
```
Expected: a raiz passa a escolher entre a definição de triagem (a de hoje) e a variante `kind: "screening"` (`required: [name, version, kind, risk_rules]`, `kind: { const: "screening" }`, `risk_rules` 1–50 itens `{ when: $ref condition, color: enum red|yellow|green|blue }`, `schema_version` inteiro como hoje, `additionalProperties: false` — logo proíbe `start_step_id`, `steps`, `scoring`, `recommendations`, `priority_when`, `offer`, `suggestions`, `scheduling`; contrato §9). Se a tag não existe, **pare** e avise: o api não mergeia antes do `contracts`.

- [ ] **Step 2: Escreva as specs que falham**

```ruby
# spec/protocols/schema_screening_spec.rb
require "rails_helper"
require "json_schemer"

# protocols-v1.6.0 (ADR 0030; contratos §1): variante kind "screening".
RSpec.describe "protocols schema.json screening contract (v1.6.0)" do
  let(:schema) { JSONSchemer.schema(JSON.parse(File.read(Rails.root.join("config/protocols/schema.json")))) }

  def screening(extra = {})
    { "name" => "acolhimento", "version" => 1, "kind" => "screening",
      "risk_rules" => [ { "when" => { "gte" => ["vitals.systolic", 180] }, "color" => "red" } ] }.merge(extra)
  end

  it "aceita a variante e continua aceitando triagem sem kind" do
    expect(schema.valid?(screening)).to be(true)
    triage = { "name" => "t", "version" => 1, "start_step_id" => "s1",
               "steps" => [ { "id" => "s1", "prompt" => "?", "answer_type" => "boolean" } ] }
    expect(schema.valid?(triage)).to be(true)
  end

  it "recusa cor fora da escala, regras vazias ou demais, campos de triagem e regra sem when" do
    expect(schema.valid?(screening("risk_rules" => [ { "when" => { "gte" => ["vitals.spo2", 1] }, "color" => "orange" } ]))).to be(false)
    expect(schema.valid?(screening("risk_rules" => []))).to be(false)
    expect(schema.valid?(screening("risk_rules" => Array.new(51) { { "when" => { "gte" => ["vitals.spo2", 1] }, "color" => "blue" } }))).to be(false)
    expect(schema.valid?(screening("steps" => []))).to be(false)
    expect(schema.valid?(screening("scoring" => {}))).to be(false)
    expect(schema.valid?(screening("start_step_id" => "s1"))).to be(false)
    expect(schema.valid?(screening("recommendations" => {}))).to be(false)
    expect(schema.valid?(screening("priority_when" => []))).to be(false)
    expect(schema.valid?(screening("schema_version" => 6))).to be(true)
    expect(schema.valid?(screening("schema_version" => "1.6.0"))).to be(false)
    expect(schema.valid?(screening("risk_rules" => [ { "color" => "red" } ]))).to be(false)
    expect(schema.valid?(screening.except("risk_rules"))).to be(false)
  end
end
```

```ruby
# spec/protocols/validation/screening_spec.rb
require "rails_helper"

# ADR 0030 (spec §3.3): o when das regras de cor aceita vitals.*, complaint.ciap2
# e profile.*; o nome acolhimento é reservado à variante.
RSpec.describe Protocols::Validation::Screening do
  def definition(rules, name: "acolhimento")
    { "name" => name, "version" => 1, "kind" => "screening", "risk_rules" => rules }
  end

  def rule(node, color = "red") = { "when" => node, "color" => color }

  it "aceita sinais, queixa e perfil, inclusive momento da glicemia" do
    node = { "any" => [ { "gte" => ["vitals.systolic", 180] }, { "lt" => ["vitals.spo2", 90] },
                        { "eq" => ["complaint.ciap2", "K86"] }, { "eq" => ["vitals.glucose_moment", "fasting"] },
                        { "gte" => ["profile.age", 60] }, { "eq" => ["profile.sex", "female"] },
                        { "gte" => ["vitals.bmi", 40] } ] }
    expect(described_class.call(definition([ rule(node) ]))).to eq([])
  end

  it "recusa variável de outro lugar, valores inválidos, passo e regra malformada" do
    errors = described_class.call(definition([
      rule({ "gte" => ["outcome.score", 3] }), rule({ "eq" => ["complaint.ciap2", "k86"] }),
      rule({ "eq" => ["vitals.glucose_moment", "noite"] }), rule({ "gte" => ["vitals.glucose_moment", 1] }),
      rule({ "eq" => ["tosse", "true"] }), "x", { "when" => { "gte" => ["vitals.spo2", 1] }, "color" => "orange" }
    ]))
    expect(errors.join("\n")).to include("risk_rules[0].when", "risk_rules[1].when", "risk_rules[2].when",
                                         "risk_rules[3].when", "risk_rules[4].when", "risk_rules[5] must be an object",
                                         "risk_rules[6].color")
  end

  it "nome reservado: screening só com acolhimento; triagem nunca com acolhimento" do
    expect(described_class.call(definition([ rule({ "gte" => ["vitals.spo2", 1] }) ], name: "outro")))
      .to include("screening protocol must be named 'acolhimento'")
    expect(described_class.reserved_name_errors({ "name" => "acolhimento", "steps" => [] }))
      .to eq([ "name 'acolhimento' is reserved for the screening protocol" ])
  end

  it "total para lixo e para triagem" do
    expect(described_class.call(nil)).to eq([])
    expect(described_class.call({ "name" => "t" })).to eq([])
    expect(described_class.call(definition("x"))).to eq([ "risk_rules must be an array" ])
    expect(described_class.call(definition([]))).to eq([ "risk_rules must have 1 to 50 rules" ])
  end

  it "o gate completo responde à variante sem rodar os linters de triagem" do
    result = Protocols::Gate.call(definition([ rule({ "gte" => ["vitals.systolic", 180] }) ]))
    expect(result.valid?).to be(true)
    expect(Protocols::Gate.call(definition([ rule({ "gte" => ["outcome.score", 1] }) ])).valid?).to be(false)
  end

  it "o validador do save aceita a variante (rascunho) e recusa sem risk_rules" do
    expect(Protocols::Validator.call(definition([ rule({ "gte" => ["vitals.spo2", 1] }) ])).valid?).to be(true)
    expect(Protocols::Validator.call(definition(nil).except("risk_rules")).errors).to include("missing :risk_rules")
  end
end
```

```ruby
# spec/protocols/condition_context_screening_spec.rb
require "rails_helper"

# ADR 0030: as variáveis da escuta entram no contexto plano como texto;
# sinal ausente não entra (condição falsa).
RSpec.describe Protocols::ConditionContext do
  it "monta vitals.* e complaint.ciap2, sem nil, e reserva os prefixos" do
    context = described_class.build(vitals: { systolic: 185, temperature_c: BigDecimal("38.5"), spo2: nil, bmi: 31.2 },
                                    complaint: { ciap2: "K86" }, profile: { age: 61, sex: "female" })
    expect(context).to eq("vitals.systolic" => "185", "vitals.temperature_c" => "38.5", "vitals.bmi" => "31.2",
                          "complaint.ciap2" => "K86", "profile.age" => "61", "profile.sex" => "female")
    expect(described_class.reserved?("vitals.systolic")).to be(true)
    expect(described_class.reserved?("complaint.ciap2")).to be(true)
    expect(Protocols::Condition.eval({ "gte" => ["vitals.systolic", 180] }, context)).to be(true)
    expect(Protocols::Condition.eval({ "lt" => ["vitals.spo2", 90] }, context)).to be(false)
  end

  it "ignora campo desconhecido de vitals" do
    expect(described_class.build(vitals: { "pressao" => 1 })).to eq({})
  end
end
```

```ruby
# spec/services/triages/offer_screening_spec.rb
require "rails_helper"

# ADR 0030: o protocolo de acolhimento é da escuta, nunca uma triagem do cidadão.
RSpec.describe "Acolhimento fora do catálogo de triagens" do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let!(:acolhimento) do
    ProtocolDefinition.create!(name: "acolhimento", version: 1, status: "active",
                               definition: { "name" => "acolhimento", "version" => 1, "kind" => "screening",
                                             "risk_rules" => [ { "when" => { "gte" => ["vitals.systolic", 180] }, "color" => "red" } ] })
  end

  it "não é oferecido, não entra no catálogo do admin e não começa como triagem" do
    citizen = profiled_citizen!(age: 40)
    expect(Triages::Offer.for(citizen: citizen).map(&:protocol_name)).not_to include("acolhimento")
    expect(Triages::CatalogAdmin.index.map { |i| i[:protocol_name] }).not_to include("acolhimento")
    expect(ProtocolDefinition.triage_protocols).not_to include(acolhimento)
    expect(ProtocolDefinition.screening_protocols).to eq([ acolhimento ])
    started = start_for!(citizen, "acolhimento")
    expect(started).to be_failure
    expect(%i[not_offered no_protocol]).to include(started.reason)
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/protocols/schema_screening_spec.rb spec/protocols/validation/screening_spec.rb spec/protocols/condition_context_screening_spec.rb spec/services/triages/offer_screening_spec.rb`
Expected: FAIL (`uninitialized constant Protocols::Validation::Screening`, `unknown keyword: :vitals`, e o `ProtocolDefinition.create!` recusado pelo validador do save com `missing :start_step_id`).

- [ ] **Step 4: Variáveis da escuta na linguagem de condição**

Em `app/protocols/condition_context.rb`, troque `RESERVED_PREFIXES` e `build` (o resto fica):

```ruby
    RESERVED_PREFIXES = %w[profile. outcome. citizen. vitals. complaint.].freeze
    # ADR 0030: sinais vitais da escuta (spec §3.1) e o IMC calculado.
    VITALS = %w[systolic diastolic heart_rate respiratory_rate temperature_c spo2 capillary_glucose glucose_moment
                weight_kg height_cm bmi pain_score].freeze

    module_function

    def reserved?(name) = name.to_s.start_with?(*RESERVED_PREFIXES)

    def build(answers: {}, profile: {}, outcome: {}, citizen: {}, vitals: {}, complaint: {})
      context = {}
      (answers.is_a?(Hash) ? answers : {}).each { |key, value| context[key.to_s] = value unless reserved?(key) }
      profile = symbolize(profile)
      outcome = symbolize(outcome)
      citizen = symbolize(citizen)
      vitals = symbolize(vitals)
      complaint = symbolize(complaint)
      put(context, "profile.age", profile[:age])
      put(context, "profile.sex", profile[:sex])
      put(context, "outcome.tier", outcome[:tier])
      put(context, "outcome.score", outcome[:score])
      put(context, "outcome.priority", outcome[:priority])
      put(context, "citizen.neighborhood_id", citizen[:neighborhood_id])
      VITALS.each { |field| put(context, "vitals.#{field}", vitals[field.to_sym]) }
      put(context, "complaint.ciap2", complaint[:ciap2])
      context
    end
```

`put` usa `value.to_s`: `BigDecimal("38.5").to_s` é `"0.385e2"`, que `Float()` lê como 38.5, mas o teste espera `"38.5"`. Troque `put` por:

```ruby
    def put(context, key, value)
      return if value.nil?

      context[key] = value.is_a?(BigDecimal) ? value.to_s("F") : value.to_s
    end
```

Em `app/protocols/validation/condition.rb`, troque `VARIABLES` e acrescente as constantes e os ramos de valor:

```ruby
      VARIABLES = {
        "profile.age" => :number, "profile.sex" => :sex, "outcome.tier" => :text,
        "outcome.score" => :number, "outcome.priority" => :number, "citizen.neighborhood_id" => :uuid,
        # ADR 0030 (contratos §1): variáveis da escuta.
        "vitals.systolic" => :number, "vitals.diastolic" => :number, "vitals.heart_rate" => :number,
        "vitals.respiratory_rate" => :number, "vitals.temperature_c" => :number, "vitals.spo2" => :number,
        "vitals.capillary_glucose" => :number, "vitals.glucose_moment" => :glucose_moment,
        "vitals.weight_kg" => :number, "vitals.height_cm" => :number, "vitals.bmi" => :number,
        "vitals.pain_score" => :number, "complaint.ciap2" => :ciap2
      }.freeze
      SEXES = %w[female male].freeze
      GLUCOSE_MOMENTS = %w[fasting postprandial random].freeze
      CIAP2 = /\A[A-Z]\d{2}\z/
```

e em `variable_value_errors`:

```ruby
        valid = case VARIABLES[name]
                when :sex then ->(v) { SEXES.include?(v.to_s) }
                when :uuid then ->(v) { v.to_s.match?(UUID) }
                when :number then ->(v) { numeric?(v) }
                when :glucose_moment then ->(v) { GLUCOSE_MOMENTS.include?(v.to_s) }
                when :ciap2 then ->(v) { v.to_s.match?(CIAP2) }
                else ->(_v) { true }
                end
```

e a mensagem de `reserved_prefix_errors`:

```ruby
          "step id '#{s["id"]}' uses a reserved prefix (profile., outcome., citizen., vitals., complaint.)"
```

Atualize `spec/protocols/gate_spec.rb:120` para a mesma frase e, em `spec/protocols/condition_context_spec.rb` ("knows the reserved prefixes"), acrescente `expect(described_class.reserved?("vitals.spo2")).to be(true)`.

Em `app/services/protocols/condition_text.rb`, troque `LABELS`:

```ruby
    LABELS = {
      "profile.age" => "idade", "profile.sex" => "sexo", "outcome.tier" => "faixa", "outcome.score" => "pontuação",
      "outcome.priority" => "prioridade", "citizen.neighborhood_id" => "bairro",
      "vitals.systolic" => "pressão sistólica", "vitals.diastolic" => "pressão diastólica",
      "vitals.heart_rate" => "frequência cardíaca", "vitals.respiratory_rate" => "frequência respiratória",
      "vitals.temperature_c" => "temperatura", "vitals.spo2" => "saturação", "vitals.capillary_glucose" => "glicemia",
      "vitals.glucose_moment" => "momento da glicemia", "vitals.weight_kg" => "peso", "vitals.height_cm" => "altura",
      "vitals.bmi" => "IMC", "vitals.pain_score" => "dor", "complaint.ciap2" => "queixa (CIAP-2)"
    }.freeze
```

- [ ] **Step 5: Gate da variante**

```ruby
# app/protocols/validation/screening.rb
# Gate da variante kind "screening" (ADR 0030; spec §3.3; contratos §1): só
# regras { when, color }, até 50; o when aceita vitals.*, complaint.ciap2 e
# profile.*. Nome reservado: acolhimento. Total para qualquer entrada.
module Protocols
  module Validation
    module Screening
      NAME = "acolhimento".freeze
      COLORS = %w[red yellow green blue].freeze
      MAX_RULES = 50
      VARIABLES = (Condition::VARIABLES.keys.grep(/\A(vitals|complaint)\./) + %w[profile.age profile.sex]).freeze

      module_function

      def screening?(definition) = definition.is_a?(Hash) && definition["kind"] == "screening"

      def call(definition)
        return [] unless screening?(definition)

        errors = []
        errors << "screening protocol must be named '#{NAME}'" unless definition["name"] == NAME
        rules = definition["risk_rules"]
        return errors << "risk_rules must be an array" unless rules.is_a?(Array)
        return errors << "risk_rules must have 1 to #{MAX_RULES} rules" unless rules.size.between?(1, MAX_RULES)

        rules.each_with_index do |rule, index|
          next errors << "risk_rules[#{index}] must be an object" unless rule.is_a?(Hash)

          errors << "risk_rules[#{index}].color must be one of #{COLORS.join(', ')}" unless COLORS.include?(rule["color"])
          errors.concat(Condition.errors(rule["when"], {}, variables: VARIABLES).map { |e| "risk_rules[#{index}].when: #{e}" })
        end
        errors
      end

      # Protocolo de triagem nunca usa o nome do acolhimento.
      def reserved_name_errors(definition)
        return [] unless definition.is_a?(Hash) && !screening?(definition) && definition["name"] == NAME

        [ "name '#{NAME}' is reserved for the screening protocol" ]
      end
    end
  end
end
```

Em `app/protocols/gate.rb`, logo depois do retorno de `schema_errors`:

```ruby
      # ADR 0030: a variante de escuta só tem regras de cor; os linters de
      # triagem (passos, pontuação, oferta) não se aplicam.
      return Validator::Result.new(errors: Validation::Screening.call(definition)) if Validation::Screening.screening?(definition)
```

e no fim da lista de erros da triagem: `errors.concat(Validation::Screening.reserved_name_errors(definition))`.

Em `app/protocols/validator.rb`, no começo de `call`:

```ruby
    def call
      return screening_result if definition.is_a?(Hash) && definition["kind"] == "screening"

      errors = []
      errors.concat(schema_errors)
      errors.concat(linter_errors) if errors.empty?
      Result.new(errors: errors)
    end
```

e, entre os privados:

```ruby
    # ADR 0030: a variante de escuta não tem passos; o save só exige a forma
    # mínima (o gate completo roda em publicar e na prévia).
    def screening_result
      errors = []
      errors << "missing :name" unless definition["name"].is_a?(String)
      errors << "missing :version" unless definition["version"].is_a?(Integer)
      errors << "missing :risk_rules" unless definition["risk_rules"].is_a?(Array)
      Result.new(errors: errors)
    end
```

Em `app/controllers/authoring/protocols_controller.rb#preview`, antes de `Definitions.build`:

```ruby
      # ADR 0030: a escuta não tem passos para simular resposta.
      if Protocols::Validation::Screening.screening?(definition_param)
        return render(json: { valid: false, errors: [ "preview is not available for screening protocols" ] },
                      status: :unprocessable_entity)
      end
```

- [ ] **Step 6: O acolhimento fora do catálogo de triagens**

Em `app/models/protocol_definition.rb`, depois de `scope :published`:

```ruby
  # ADR 0030: a variante de escuta (acolhimento) não é triagem do cidadão.
  scope :screening_protocols, -> { where("definition->>'kind' = 'screening'") }
  scope :triage_protocols, -> { where("definition->>'kind' IS NULL OR definition->>'kind' <> 'screening'") }
```

Troque `ProtocolDefinition.active` por `ProtocolDefinition.triage_protocols.active` em `Triages::Offer.for` (`app/services/triages/offer.rb:63`), `Triages::CatalogAdmin.index` e `.item_for` (`app/services/triages/catalog_admin.rb:9` e `:19`). Em `app/commands/start_triage.rb`, troque `ProtocolDefinition.find_by(name: name, status: "active")` por `ProtocolDefinition.triage_protocols.find_by(name: name, status: "active")`.

- [ ] **Step 7: Rode e veja passar (e a suíte de protocolos)**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/protocols spec/services/triages spec/services/protocols spec/commands/protocols spec/requests/authoring* spec/commands/start_triage_spec.rb`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add config/protocols/schema.json app/protocols/validation/screening.rb app/protocols/validation/condition.rb app/protocols/condition_context.rb app/protocols/gate.rb app/protocols/validator.rb app/models/protocol_definition.rb app/services/triages/offer.rb app/services/triages/catalog_admin.rb app/commands/start_triage.rb app/controllers/authoring/protocols_controller.rb app/services/protocols/condition_text.rb spec/protocols/gate_spec.rb spec/protocols/condition_context_spec.rb spec/protocols/schema_screening_spec.rb spec/protocols/validation/screening_spec.rb spec/protocols/condition_context_screening_spec.rb spec/services/triages/offer_screening_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: accept the screening protocol variant with vital sign variables

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 2 — Dados (F-18.1, F-18.3, F-18.6, F-18.7)

### Task 3: Migração das escutas, travas do atendimento, modelos e eventos declarados

**Files:**
- Create: `config/scheduling/screening_cbos.yml`
- Create: `db/city_migrate/20261007300001_add_screenings.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`
- Create: `app/models/screening.rb`, `app/models/screening_revision.rb`, `app/models/ledi_generation_failure.rb`
- Modify: `app/models/attendance.rb`, `app/models/appointment_request.rb`, `app/models/health_unit.rb`, `app/commands/health_units/drain.rb:74-82`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `config/initializers/filter_parameter_logging.rb`
- Create: `spec/support/screening_helpers.rb`; Modify: `spec/rails_helper.rb`
- Modify: `spec/adr_pointers_spec.rb`
- Test: `spec/models/screening_tables_guard_spec.rb`

**Interfaces:**
- Produces:
  - Tabelas e colunas: `health_units.screening_scope` (`walk_in` padrão | `all`); `screenings (attendance_id único, status, started_by_user_id, professional_link_id, cbo_code, started_at, completed_at, current_revision_id, destination, orientation_note, appointment_request_id, timestamps)`; `screening_revisions (screening_id, by_user_id, created_at, ciap2_code, ciap2_release_id, complaint_note, systolic, diastolic, heart_rate, respiratory_rate, temperature_c decimal(3,1), spo2, capillary_glucose, glucose_moment, weight_kg decimal(5,2), height_cm, pain_score, suggested_color, final_color, color_change_reason, rule_protocol_definition_id, matched_rules int[])`; `appointment_requests.origin_screening_id`; `ledi_generation_failures (source_type, source_id, reason_codes jsonb, resolved_at, timestamps)`.
  - Triggers: `screenings_guard`, `screening_revisions_append_only` (+ TRUNCATE), `attendances_screening_close_guard`; `rota_attendance_guard` e `rota_appointment_request_guard` atualizados.
  - Modelos: `Screening` (`STATUSES`, `DESTINATIONS`, `belongs_to :attendance, :started_by_user, :professional_link, :current_revision (opcional), :appointment_request (opcional)`, `has_many :revisions`, scopes `completed_screenings`, `in_progress_screenings`, `#completed?`, `#in_progress?`); `ScreeningRevision` (`COLORS`, `VITAL_COLUMNS`, `belongs_to :screening, :by_user`); `LediGenerationFailure` (`REASONS`, `scope :unresolved`, `#resolved?`); `Attendance::OUTCOMES` com os dois novos, `::CLOSE_OUTCOMES == %w[discharged referred return left]`, `::SCREENING_OUTCOMES == { "schedule" => "scheduled_from_screening", "oriented" => "oriented", "referred" => "referred" }`, `has_one :screening`; `AppointmentRequest::KINDS` com `screening`, `belongs_to :origin_screening` (opcional); `HealthUnit::SCREENING_SCOPES == %w[walk_in all]`.
  - Helpers de spec (`spec/support/screening_helpers.rb`): `ciap2_release!(codes = ScreeningHelpers::CIAP2)`, `screening_citizen!(n, age: 40 + n, sex: "female")`, `reception!`, `screener!(unit, cbo: "223505")`, `walk_in_attendance!(unit, citizen:, checked_in_at: Time.current)`, `scheduled_attendance!(unit, citizen:, checked_in_at: Time.current)`, `appointment_type!(key = "consulta_enfermagem")`, `acolhimento!(rules = ScreeningHelpers::RULES, version: 1)`, `revision_params(**over)`, `exportable_unit!(unit, *users)`.

- [ ] **Step 1: Confira o número da migração**

Run: `ls apps/api/.claude/mod18/db/city_migrate | tail -2`
Expected: a última é `20261006210001_add_professional_schedules.rb`. Use `20261007300001` (esta) e `20261007300002` (Task 4). Se houver migração mais nova em `origin/main`, use números maiores e ajuste os nomes nos Steps 5–6 e o `define(version:)`.

- [ ] **Step 2: Lista de CBOs da escuta**

```yaml
# config/scheduling/screening_cbos.yml
# Quem faz a escuta inicial (ADR 0030; spec 2026-10-07 §3.2): nível superior
# da equipe e técnico/auxiliar de enfermagem. Prefixos de CBO. A ficha LEDI
# recorta (Screenings::Cbos.allowed?): o CBO precisa ter ficha possível.
- { prefix: "2251", group: "Médicos clínicos" }
- { prefix: "2252", group: "Médicos em especialidades cirúrgicas" }
- { prefix: "2253", group: "Médicos em medicina diagnóstica e terapêutica" }
- { prefix: "2235", group: "Enfermeiros" }
- { prefix: "2234", group: "Farmacêuticos" }
- { prefix: "2516", group: "Assistentes sociais" }
- { prefix: "2237", group: "Nutricionistas" }
- { prefix: "2236", group: "Fisioterapeutas" }
- { prefix: "2232", group: "Cirurgiões-dentistas" }
- { prefix: "3222", group: "Técnicos e auxiliares de enfermagem" }
```

- [ ] **Step 3: Helpers de spec**

```ruby
# spec/support/screening_helpers.rb
# Módulo 18 (ADR 0030): CIAP-2 na plataforma, profissionais da escuta,
# atendimentos aguardando (demanda espontânea ou horário) e o protocolo de
# acolhimento ativo, gravados direto (cenário de teste). Os caminhos reais são
# Terminology::Import, Attendances::CheckIn e o ciclo assinado.
module ScreeningHelpers
  CIAP2 = { "K86" => "Hipertensão sem complicações", "R05" => "Tosse", "A03" => "Febre",
            "N01" => "Cefaleia", "T90" => "Diabetes não insulino-dependente" }.freeze
  RULES = [
    { "when" => { "any" => [ { "gte" => ["vitals.systolic", 180] }, { "lt" => ["vitals.spo2", 90] } ] }, "color" => "red" },
    { "when" => { "any" => [ { "gte" => ["vitals.temperature_c", 39] }, { "gte" => ["vitals.capillary_glucose", 300] } ] },
      "color" => "yellow" },
    { "when" => { "eq" => ["complaint.ciap2", "R05"] }, "color" => "green" }
  ].freeze

  def ciap2_release!(codes = CIAP2)
    TerminologyRelease.active.find_by(kind: "ciap2") || begin
      release = TerminologyRelease.create!(kind: "ciap2", version: "2026.1", source_sha256: "c" * 64,
                                           imported_by: "rspec", imported_at: Time.current, status: "importing")
      codes.each { |code, description| Ciap2Code.create!(release: release, code: code, description: description) }
      release.update!(status: "active", activated_at: Time.current)
      release
    end
  end

  # Cidadão com perfil (nascimento e sexo), CPF e telefone únicos por n.
  def screening_citizen!(n, age: 40 + n, sex: "female")
    profiled_citizen!(age: age, sex: sex, phone: format("+55419%08d", 31_000_000 + n))
  end

  def reception! = (@reception ||= staff_with("recepcao-#{SecureRandom.hex(3)}@cidade.gov.br", "citizen_verifier"))

  # Profissional da escuta com vínculo ativo na unidade (perfil pelo link_professional!).
  def screener!(unit, cbo: "223505", email: "escuta-#{SecureRandom.hex(3)}@cidade.gov.br")
    staff_with(email, "health_professional").tap { |user| link_professional!(user, unit, cbo: cbo) }
  end

  # Demanda espontânea: origem triagem, sem horário.
  def walk_in_attendance!(unit, citizen:, checked_in_at: Time.current)
    triage = completed_web_triage_for(citizen)
    Attendance.create!(triage: triage, citizen: citizen, health_unit: unit, checked_in_by_user: reception!,
                       checked_in_at: checked_in_at, check_in_method: "code")
  end

  # Origem horário marcado (fora do escopo walk_in).
  def scheduled_attendance!(unit, citizen:, checked_in_at: Time.current)
    request = triage_request!(citizen, unit: unit)
    shift = shift!(doctor_link!(unit), starts_at: 1.hour.from_now)
    appointment = appointment_row!(request, shift, starts_at: shift.starts_at)
    Attendance.create!(appointment: appointment, citizen: citizen, health_unit: unit, checked_in_by_user: reception!,
                       checked_in_at: checked_in_at, check_in_method: "code")
  end

  def appointment_type!(key = "consulta_enfermagem")
    AppointmentType.find_by(key: key) || type_row!(key, cbo: ["2235"], minutes: 15, origin: "platform")
  end

  def acolhimento!(rules = RULES, version: 1)
    ProtocolDefinition.create!(name: "acolhimento", version: version, status: "active",
                               definition: { "name" => "acolhimento", "version" => version, "kind" => "screening",
                                             "risk_rules" => rules })
  end

  def revision_params(**over)
    { "ciap2_code" => "K86", "vitals" => { "systolic" => 130, "diastolic" => 85 }, "final_color" => "green" }
      .merge(over.transform_keys(&:to_s))
  end

  # CNES na unidade, equipe com INE e os profissionais dados na equipe: a ficha pode nascer.
  def exportable_unit!(unit, *users)
    unit.update!(cnes: "1234567") if unit.cnes.blank?
    team = HealthTeam.find_by(health_unit: unit) ||
           HealthTeam.create!(health_unit: unit, ine: "0000123456", kind: "70", name: "ESF Centro")
    users.each do |user|
      link = user.professional.links.active.find_by!(health_unit: unit)
      HealthTeamMember.create!(professional: user.professional, health_team: team, cbo_code: link.cbo_code,
                               started_on: Time.zone.today - 30)
    end
    team
  end
end

RSpec.configure { |c| c.include ScreeningHelpers }
```

Em `spec/rails_helper.rb`, depois de `require_relative "support/ledi_helpers"`, acrescente `require_relative "support/screening_helpers"`.

- [ ] **Step 4: Escreva a spec de guarda (falha: tabelas e colunas não existem)**

```ruby
# spec/models/screening_tables_guard_spec.rb
require "rails_helper"

# Módulo 18 (ADR 0030; spec §3–§4): o banco garante o que o modelo não vê.
RSpec.describe "Guardas das tabelas da escuta" do
  before { Current.city = TEST_CITY_A; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  let(:attendance) { walk_in_attendance!(unit, citizen: screening_citizen!(1)) }
  let(:link) { nurse.professional.links.active.sole }

  def attempt(&) = ApplicationRecord.transaction(requires_new: true, &)

  def screening!(status: "in_progress")
    Screening.create!(attendance: attendance, status: "in_progress", started_by_user: nurse, professional_link: link,
                      cbo_code: link.cbo_code, started_at: Time.current).tap do |s|
      s.update!(status: "abandoned") if status == "abandoned"
    end
  end

  def revision!(screening, **over)
    ScreeningRevision.create!({ screening: screening, by_user: nurse, ciap2_code: "K86",
                                ciap2_release_id: TerminologyRelease.active.find_by!(kind: "ciap2").id,
                                systolic: 130, diastolic: 85, final_color: "green" }.merge(over))
  end

  def complete!(screening, destination: "same_day", **over)
    revision = revision!(screening)
    screening.update!({ status: "completed", completed_at: Time.current, destination: destination,
                        current_revision: revision }.merge(over))
    revision
  end

  describe "health_units.screening_scope" do
    it "nasce walk_in e só aceita walk_in ou all" do
      expect(unit.screening_scope).to eq("walk_in")
      expect { attempt { unit.update_columns(screening_scope: "todos") } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_health_units_screening_scope/)
    end
  end

  describe "screenings" do
    it "uma por atendimento; DELETE recusado; atendimento nunca muda" do
      screening = screening!
      expect { attempt { screening! } }.to raise_error(ActiveRecord::RecordNotUnique)
      expect { attempt { screening.delete } }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
      other = walk_in_attendance!(unit, citizen: screening_citizen!(2))
      expect { attempt { screening.update_columns(attendance_id: other.id) } }
        .to raise_error(ActiveRecord::StatementInvalid, /identity columns never change/)
    end

    it "transições: em curso → concluída/abandonada; abandonada → em curso; concluída só ganha revisão" do
      screening = screening!(status: "abandoned")
      screening.update!(status: "in_progress", started_at: Time.current)
      revision = complete!(screening)
      expect { attempt { screening.update_columns(status: "abandoned") } }
        .to raise_error(ActiveRecord::StatementInvalid, /only gains revisions/)
      expect { attempt { screening.update_columns(destination: "oriented", orientation_note: "beber água e voltar") } }
        .to raise_error(ActiveRecord::StatementInvalid, /only gains revisions/)
      newer = revision!(screening, final_color: "yellow")
      expect { screening.update!(current_revision: newer) }.not_to raise_error
      expect(revision.reload).to be_persisted
    end

    it "CHECKs: concluída exige destino, revisão e hora; orientação só com oriented; schedule exige pedido" do
      screening = screening!
      expect { attempt { screening.update_columns(status: "completed", completed_at: Time.current) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_screenings_completion/)
      expect { attempt { complete!(screening, destination: "oriented") } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_screenings_orientation/)
      expect { attempt { complete!(screening, destination: "schedule") } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_screenings_schedule/)
      expect { attempt { screening.update_columns(destination: "agora") } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_screenings_destination/)
    end
  end

  describe "screening_revisions" do
    it "só acréscimo; plausibilidade, pressão aos pares, glicemia com momento, cor e justificativa" do
      screening = screening!
      revision = revision!(screening)
      expect { attempt { revision.update_columns(final_color: "red") } }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { attempt { revision.delete } }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      {
        { systolic: 120, diastolic: nil } => /ck_screening_revisions_bp/,
        { systolic: 120, diastolic: 130 } => /ck_screening_revisions_bp/,
        { systolic: 400, diastolic: 80 } => /ck_screening_revisions_systolic/,
        { spo2: 101 } => /ck_screening_revisions_spo2/,
        { capillary_glucose: 120 } => /ck_screening_revisions_glucose/,
        { capillary_glucose: 900, glucose_moment: "random" } => /ck_screening_revisions_glucose/,
        { pain_score: 11 } => /ck_screening_revisions_pain_score/,
        { final_color: "orange" } => /ck_screening_revisions_final_color/,
        { suggested_color: "red", final_color: "green" } => /ck_screening_revisions_color_change/,
        { suggested_color: "red", final_color: "green", color_change_reason: "curta" } => /ck_screening_revisions_color_change_reason/,
        { ciap2_code: "k86" } => /ck_screening_revisions_ciap2/,
        { complaint_note: "x" * 501 } => /ck_screening_revisions_complaint_note/
      }.each do |attrs, error|
        expect { attempt { revision!(screening, **attrs) } }.to raise_error(ActiveRecord::StatementInvalid, error), attrs.inspect
      end
      expect { attempt { revision!(screening, suggested_color: "red", final_color: "yellow",
                                              color_change_reason: "dor torácica já avaliada") } }.not_to raise_error
    end
  end

  describe "attendances (exceção na trava do módulo 13)" do
    def close_from_waiting(outcome, **extra)
      attempt { attendance.update!({ status: "closed", outcome: outcome, closed_by_user: nurse, closed_at: Time.current }.merge(extra)) }
    end

    it "fechar de waiting como scheduled_from_screening/oriented/referred exige escuta concluída com o destino" do
      expect { close_from_waiting("oriented") }.to raise_error(ActiveRecord::StatementInvalid, /requires a completed screening/)
      expect { close_from_waiting("referred", referral_note: "UPA") }
        .to raise_error(ActiveRecord::StatementInvalid, /requires a completed screening/)
      screening = screening!
      complete!(screening, destination: "same_day")
      expect { close_from_waiting("oriented") }.to raise_error(ActiveRecord::StatementInvalid, /requires a completed screening/)
    end

    it "com a escuta oriented concluída, fecha oriented de waiting; desfecho clínico comum ainda exige chamada" do
      complete!(screening!, destination: "oriented", orientation_note: "hidratação e retorno se piorar")
      expect { close_from_waiting("discharged") }.to raise_error(ActiveRecord::StatementInvalid, /invalid transition/)
      expect { close_from_waiting("oriented") }.not_to raise_error
      expect(attendance.reload.outcome).to eq("oriented")
    end

    it "oriented e scheduled_from_screening nunca saem de in_care" do
      attendance.update!(status: "in_care", called_by_user: nurse, called_at: Time.current)
      expect { attempt { attendance.update!(status: "closed", outcome: "oriented", closed_by_user: nurse, closed_at: Time.current) } }
        .to raise_error(ActiveRecord::StatementInvalid, /invalid transition/)
    end
  end

  describe "appointment_requests kind screening" do
    it "exige a escuta de origem e o atendimento; a origem nunca muda" do
      screening = screening!
      base = { kind: "screening", origin_attendance: attendance, citizen: attendance.citizen,
               root_triage: attendance.root_triage, origin_unit: unit, target_unit: unit,
               appointment_type_key: appointment_type!.key, priority: "routine", due_on: Time.zone.today + 7 }
      expect { attempt { AppointmentRequest.create!(base) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_appointment_requests_screening_kind/)
      request = AppointmentRequest.create!(base.merge(origin_screening: screening))
      other = Screening.create!(attendance: walk_in_attendance!(unit, citizen: screening_citizen!(3)), status: "in_progress",
                                started_by_user: nurse, professional_link: link, cbo_code: link.cbo_code, started_at: Time.current)
      expect { attempt { request.update_columns(origin_screening_id: other.id) } }
        .to raise_error(ActiveRecord::StatementInvalid, /origin columns never change/)
    end
  end

  describe "ledi_generation_failures" do
    it "um aberto por fonte; motivos não vazios" do
      LediGenerationFailure.create!(source_type: "Screening", source_id: SecureRandom.uuid, reason_codes: [ "unit_without_cnes" ])
      id = SecureRandom.uuid
      LediGenerationFailure.create!(source_type: "Screening", source_id: id, reason_codes: [ "citizen_without_sex" ])
      expect { attempt { LediGenerationFailure.create!(source_type: "Screening", source_id: id, reason_codes: [ "citizen_without_sex" ]) } }
        .to raise_error(ActiveRecord::RecordNotUnique)
      expect { attempt { LediGenerationFailure.create!(source_type: "Screening", source_id: SecureRandom.uuid, reason_codes: []) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_ledi_generation_failures_reason_codes/)
    end
  end
end
```

- [ ] **Step 5: Escreva a migração**

```ruby
# db/city_migrate/20261007300001_add_screenings.rb
# Módulo 18 (ADR 0030; spec 2026-10-07 §3–§5): escuta inicial com revisões só
# de acréscimo, escopo por unidade, desfechos de escuta e a exceção na trava
# do módulo 13, pedido de agendamento com origem na escuta e as fichas que não
# puderam ser geradas. Triggers em db/city_triggers.sql.
class AddScreenings < ActiveRecord::Migration[8.1]
  SCREENING_OUTCOMES = "'scheduled_from_screening'::text, 'oriented'::text".freeze

  def up
    add_column :health_units, :screening_scope, :string, null: false, default: "walk_in"
    add_check_constraint :health_units, "screening_scope::text = ANY (ARRAY['walk_in'::text, 'all'::text])",
                         name: "ck_health_units_screening_scope"

    create_table :screenings, id: :uuid do |t|
      t.uuid :attendance_id, null: false
      t.string :status, null: false, default: "in_progress"
      t.uuid :started_by_user_id, null: false
      t.uuid :professional_link_id, null: false
      t.string :cbo_code, null: false
      t.datetime :started_at, null: false
      t.datetime :completed_at
      t.uuid :current_revision_id
      t.string :destination
      t.text :orientation_note
      t.uuid :appointment_request_id
      t.timestamps
    end
    add_index :screenings, :attendance_id, unique: true
    add_index :screenings, :started_by_user_id
    add_index :screenings, :professional_link_id
    add_index :screenings, :appointment_request_id
    add_index :screenings, :status
    add_foreign_key :screenings, :attendances
    add_foreign_key :screenings, :users, column: :started_by_user_id
    add_foreign_key :screenings, :professional_links
    add_foreign_key :screenings, :appointment_requests
    add_check_constraint :screenings, "status::text = ANY (ARRAY['in_progress'::text, 'completed'::text, 'abandoned'::text])",
                         name: "ck_screenings_status"
    add_check_constraint :screenings,
                         "destination IS NULL OR destination::text = ANY (ARRAY['same_day'::text, 'schedule'::text, 'oriented'::text, 'referred'::text])",
                         name: "ck_screenings_destination"
    add_check_constraint :screenings,
                         "(status::text = 'completed'::text) = (completed_at IS NOT NULL AND destination IS NOT NULL AND current_revision_id IS NOT NULL)",
                         name: "ck_screenings_completion"
    add_check_constraint :screenings,
                         "(destination IS DISTINCT FROM 'oriented' AND orientation_note IS NULL) OR " \
                         "(destination = 'oriented' AND orientation_note IS NOT NULL AND length(btrim(orientation_note)) BETWEEN 1 AND 500)",
                         name: "ck_screenings_orientation"
    add_check_constraint :screenings, "(destination IS NOT DISTINCT FROM 'schedule') = (appointment_request_id IS NOT NULL)",
                         name: "ck_screenings_schedule"
    add_check_constraint :screenings, "cbo_code::text ~ '^[0-9]{6}$'::text", name: "ck_screenings_cbo_code"

    create_table :screening_revisions, id: :uuid do |t|
      t.uuid :screening_id, null: false
      t.uuid :by_user_id, null: false
      t.datetime :created_at, null: false
      t.string :ciap2_code, limit: 3, null: false
      t.uuid :ciap2_release_id, null: false
      t.text :complaint_note
      t.integer :systolic
      t.integer :diastolic
      t.integer :heart_rate
      t.integer :respiratory_rate
      t.decimal :temperature_c, precision: 3, scale: 1
      t.integer :spo2
      t.integer :capillary_glucose
      t.string :glucose_moment
      t.decimal :weight_kg, precision: 5, scale: 2
      t.integer :height_cm
      t.integer :pain_score
      t.string :suggested_color
      t.string :final_color, null: false
      t.text :color_change_reason
      t.uuid :rule_protocol_definition_id
      t.integer :matched_rules, array: true, null: false, default: []
    end
    add_index :screening_revisions, :screening_id
    add_index :screening_revisions, :by_user_id
    add_foreign_key :screening_revisions, :screenings
    add_foreign_key :screening_revisions, :users, column: :by_user_id
    add_foreign_key :screening_revisions, :protocol_definitions, column: :rule_protocol_definition_id
    add_foreign_key :screenings, :screening_revisions, column: :current_revision_id
    colors = "ARRAY['red'::text, 'yellow'::text, 'green'::text, 'blue'::text]"
    {
      "ck_screening_revisions_ciap2" => "ciap2_code::text ~ '^[A-Z][0-9]{2}$'::text",
      "ck_screening_revisions_complaint_note" => "complaint_note IS NULL OR length(complaint_note) <= 500",
      "ck_screening_revisions_bp" => "(systolic IS NULL AND diastolic IS NULL) OR (systolic IS NOT NULL AND diastolic IS NOT NULL AND diastolic < systolic)",
      "ck_screening_revisions_systolic" => "systolic IS NULL OR systolic BETWEEN 50 AND 300",
      "ck_screening_revisions_diastolic" => "diastolic IS NULL OR diastolic BETWEEN 20 AND 200",
      "ck_screening_revisions_heart_rate" => "heart_rate IS NULL OR heart_rate BETWEEN 20 AND 250",
      "ck_screening_revisions_respiratory_rate" => "respiratory_rate IS NULL OR respiratory_rate BETWEEN 4 AND 80",
      "ck_screening_revisions_temperature" => "temperature_c IS NULL OR temperature_c BETWEEN 30 AND 45",
      "ck_screening_revisions_spo2" => "spo2 IS NULL OR spo2 BETWEEN 50 AND 100",
      "ck_screening_revisions_glucose" => "(capillary_glucose IS NULL AND glucose_moment IS NULL) OR (capillary_glucose BETWEEN 10 AND 800 AND glucose_moment::text = ANY (ARRAY['fasting'::text, 'postprandial'::text, 'random'::text]))",
      "ck_screening_revisions_weight" => "weight_kg IS NULL OR weight_kg BETWEEN 0.5 AND 400",
      "ck_screening_revisions_height" => "height_cm IS NULL OR height_cm BETWEEN 30 AND 250",
      "ck_screening_revisions_pain_score" => "pain_score IS NULL OR pain_score BETWEEN 0 AND 10",
      "ck_screening_revisions_suggested_color" => "suggested_color IS NULL OR suggested_color::text = ANY (#{colors})",
      "ck_screening_revisions_final_color" => "final_color::text = ANY (#{colors})",
      "ck_screening_revisions_color_change" => "suggested_color IS NULL OR final_color::text = suggested_color::text OR color_change_reason IS NOT NULL",
      "ck_screening_revisions_color_change_reason" => "color_change_reason IS NULL OR length(btrim(color_change_reason)) BETWEEN 10 AND 500"
    }.each { |name, expression| add_check_constraint :screening_revisions, expression, name: name }

    # Desfechos de escuta e a exceção na trava (spec §4).
    remove_check_constraint :attendances, name: "ck_attendances_outcome"
    add_check_constraint :attendances,
                         "outcome IS NULL OR outcome::text = ANY (ARRAY['discharged'::text, 'referred'::text, 'return'::text, 'left'::text, #{SCREENING_OUTCOMES}])",
                         name: "ck_attendances_outcome"
    remove_check_constraint :attendances, name: "ck_attendances_referral"
    add_check_constraint :attendances,
                         "((outcome IS NULL OR outcome::text = ANY (ARRAY['discharged'::text, 'left'::text, #{SCREENING_OUTCOMES}])) " \
                         "AND referral_unit_id IS NULL AND referral_note IS NULL) OR " \
                         "(outcome = 'referred' AND (referral_unit_id IS NOT NULL OR (referral_note IS NOT NULL AND length(btrim(referral_note)) > 0))) OR " \
                         "(outcome = 'return' AND referral_unit_id IS NULL)",
                         name: "ck_attendances_referral"
    remove_check_constraint :attendances, name: "ck_attendances_closing"
    add_check_constraint :attendances,
                         "(status::text = 'waiting'::text AND called_at IS NULL AND outcome IS NULL AND closed_by_user_id IS NULL AND closed_at IS NULL AND referral_unit_id IS NULL AND referral_note IS NULL) OR " \
                         "(status::text = 'in_care'::text AND called_at IS NOT NULL AND outcome IS NULL AND closed_by_user_id IS NULL AND closed_at IS NULL AND referral_unit_id IS NULL AND referral_note IS NULL) OR " \
                         "(status::text = 'closed'::text AND outcome IS NOT NULL AND closed_by_user_id IS NOT NULL AND closed_at IS NOT NULL " \
                         "AND (called_at IS NOT NULL OR outcome::text = ANY (ARRAY['left'::text, 'referred'::text, #{SCREENING_OUTCOMES}])) " \
                         "AND (called_at IS NULL OR outcome::text <> ALL (ARRAY[#{SCREENING_OUTCOMES}])))",
                         name: "ck_attendances_closing"

    # Pedido com origem na escuta (ADR 0029 + 0030).
    add_column :appointment_requests, :origin_screening_id, :uuid
    add_foreign_key :appointment_requests, :screenings, column: :origin_screening_id
    add_index :appointment_requests, :origin_screening_id, unique: true,
                                                           where: "(closed_reason)::text IS DISTINCT FROM 'moved'::text"
    remove_check_constraint :appointment_requests, name: "ck_appointment_requests_kind"
    add_check_constraint :appointment_requests,
                         "kind::text = ANY (ARRAY['return'::text, 'referral'::text, 'triage'::text, 'screening'::text])",
                         name: "ck_appointment_requests_kind"
    add_check_constraint :appointment_requests, "(kind::text = 'screening'::text) = (origin_screening_id IS NOT NULL)",
                         name: "ck_appointment_requests_screening_kind"
    add_check_constraint :appointment_requests, "origin_screening_id IS NULL OR origin_attendance_id IS NOT NULL",
                         name: "ck_appointment_requests_screening_origin"

    # Fichas que não puderam ser geradas por falta de identificação (spec §5).
    create_table :ledi_generation_failures, id: :uuid do |t|
      t.string :source_type, null: false
      t.uuid :source_id, null: false
      t.jsonb :reason_codes, null: false, default: []
      t.datetime :resolved_at
      t.timestamps
    end
    add_index :ledi_generation_failures, %i[source_type source_id], unique: true, where: "(resolved_at IS NULL)",
                                                                     name: "idx_ledi_generation_failures_open"
    add_index :ledi_generation_failures, :created_at
    add_check_constraint :ledi_generation_failures,
                         "jsonb_typeof(reason_codes) = 'array'::text AND jsonb_array_length(reason_codes) > 0",
                         name: "ck_ledi_generation_failures_reason_codes"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

- [ ] **Step 6: Triggers**

Em `db/city_triggers.sql`, troque o teste de transição de `rota_attendance_guard`:

```sql
  IF NEW.status IS DISTINCT FROM OLD.status
     AND NOT ((OLD.status = 'waiting' AND NEW.status = 'in_care')
          OR (OLD.status = 'waiting' AND NEW.status = 'closed'
              AND NEW.outcome IN ('left', 'scheduled_from_screening', 'oriented', 'referred'))
          OR (OLD.status = 'in_care' AND NEW.status = 'closed'
              AND NEW.outcome NOT IN ('left', 'scheduled_from_screening', 'oriented'))) THEN
    RAISE EXCEPTION 'attendances: invalid transition % -> %', OLD.status, NEW.status;
  END IF;
```

Em `rota_appointment_request_guard`, acrescente à lista de colunas imutáveis `OR NEW.origin_screening_id IS DISTINCT FROM OLD.origin_screening_id`. No fim do arquivo:

```sql
-- screenings (ADR 0030; spec 2026-10-07 §3): uma por atendimento; nunca some.
-- Em curso → concluída ou abandonada; abandonada pode voltar a em curso
-- (outra profissional retoma); concluída só troca a revisão corrente e o
-- autor clínico (reavaliação).
CREATE OR REPLACE FUNCTION rota_screening_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'screenings is append-only: DELETE refused';
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id OR NEW.attendance_id IS DISTINCT FROM OLD.attendance_id
     OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'screenings: identity columns never change';
  END IF;
  IF OLD.status = 'completed'
     AND (NEW.status IS DISTINCT FROM OLD.status OR NEW.completed_at IS DISTINCT FROM OLD.completed_at
          OR NEW.destination IS DISTINCT FROM OLD.destination
          OR NEW.orientation_note IS DISTINCT FROM OLD.orientation_note
          OR NEW.appointment_request_id IS DISTINCT FROM OLD.appointment_request_id
          OR NEW.started_at IS DISTINCT FROM OLD.started_at
          OR NEW.started_by_user_id IS DISTINCT FROM OLD.started_by_user_id) THEN
    RAISE EXCEPTION 'screenings: a completed screening only gains revisions';
  END IF;
  IF NEW.status IS DISTINCT FROM OLD.status
     AND NOT ((OLD.status = 'in_progress' AND NEW.status IN ('completed', 'abandoned'))
          OR (OLD.status = 'abandoned' AND NEW.status = 'in_progress')) THEN
    RAISE EXCEPTION 'screenings: invalid transition % -> %', OLD.status, NEW.status;
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- Exceção na trava do módulo 13 (ADR 0030, Invariantes): fechar de waiting
-- com desfecho de escuta exige escuta concluída com aquele destino.
CREATE OR REPLACE FUNCTION rota_attendance_screening_close_guard() RETURNS trigger AS $fn$
BEGIN
  IF OLD.status = 'waiting' AND NEW.status = 'closed'
     AND NEW.outcome IN ('scheduled_from_screening', 'oriented', 'referred')
     AND NOT EXISTS (
       SELECT 1 FROM screenings s
       WHERE s.attendance_id = NEW.id AND s.status = 'completed'
         AND s.destination = CASE NEW.outcome WHEN 'scheduled_from_screening' THEN 'schedule' ELSE NEW.outcome END) THEN
    RAISE EXCEPTION 'attendances: closing from waiting with % requires a completed screening with that destination', NEW.outcome;
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF to_regclass('public.screenings') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS screenings_guard ON screenings';
    EXECUTE 'CREATE TRIGGER screenings_guard
      BEFORE UPDATE OR DELETE ON screenings
      FOR EACH ROW EXECUTE FUNCTION rota_screening_guard()';
    EXECUTE 'DROP TRIGGER IF EXISTS screenings_append_only_truncate ON screenings';
    EXECUTE 'CREATE TRIGGER screenings_append_only_truncate
      BEFORE TRUNCATE ON screenings
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
    EXECUTE 'DROP TRIGGER IF EXISTS attendances_screening_close_guard ON attendances';
    EXECUTE 'CREATE TRIGGER attendances_screening_close_guard
      BEFORE UPDATE ON attendances
      FOR EACH ROW EXECUTE FUNCTION rota_attendance_screening_close_guard()';
  END IF;
  IF to_regclass('public.screening_revisions') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS screening_revisions_append_only ON screening_revisions';
    EXECUTE 'CREATE TRIGGER screening_revisions_append_only
      BEFORE UPDATE OR DELETE ON screening_revisions
      FOR EACH ROW EXECUTE FUNCTION rota_append_only()';
    EXECUTE 'DROP TRIGGER IF EXISTS screening_revisions_append_only_truncate ON screening_revisions';
    EXECUTE 'CREATE TRIGGER screening_revisions_append_only_truncate
      BEFORE TRUNCATE ON screening_revisions
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
  END IF;
END
$do$;
```

(Confira a mensagem de `rota_append_only` no topo do arquivo: as expectativas `/append-only/` do Step 4 dependem dela conter "append-only"; se o texto for outro, ajuste o regex da spec, não a função.)

- [ ] **Step 7: Dump à mão em `db/city_schema.rb`**

Troque `define(version: 2026_10_06_210001)` por `define(version: 2026_10_07_300001)`. Em `appointment_requests`, acrescente `t.uuid "origin_screening_id"` (ordem alfabética, depois de `origin_attendance_id`), o índice `t.index ["origin_screening_id"], name: "index_appointment_requests_on_origin_screening_id", unique: true, where: "((closed_reason)::text IS DISTINCT FROM 'moved'::text)"`, troque `ck_appointment_requests_kind` pela lista com `'screening'` e acrescente `ck_appointment_requests_screening_kind` e `ck_appointment_requests_screening_origin` com as expressões da migração. Em `attendances`, troque `ck_attendances_closing`, `ck_attendances_outcome` e `ck_attendances_referral` pelas da migração. Em `health_units`, `t.string "screening_scope", default: "walk_in", null: false` e `t.check_constraint "screening_scope::text = ANY (ARRAY['walk_in'::text, 'all'::text])", name: "ck_health_units_screening_scope"`. Tabelas novas (em ordem alfabética entre as existentes):

```ruby
  create_table "ledi_generation_failures", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "created_at", null: false
    t.jsonb "reason_codes", default: [], null: false
    t.datetime "resolved_at"
    t.uuid "source_id", null: false
    t.string "source_type", null: false
    t.datetime "updated_at", null: false
    t.index ["created_at"], name: "index_ledi_generation_failures_on_created_at"
    t.index ["source_type", "source_id"], name: "idx_ledi_generation_failures_open", unique: true, where: "(resolved_at IS NULL)"
    t.check_constraint "jsonb_typeof(reason_codes) = 'array'::text AND jsonb_array_length(reason_codes) > 0", name: "ck_ledi_generation_failures_reason_codes"
  end

  create_table "screening_revisions", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "by_user_id", null: false
    t.integer "capillary_glucose"
    t.string "ciap2_code", limit: 3, null: false
    t.uuid "ciap2_release_id", null: false
    t.text "color_change_reason"
    t.text "complaint_note"
    t.datetime "created_at", null: false
    t.integer "diastolic"
    t.string "final_color", null: false
    t.string "glucose_moment"
    t.integer "heart_rate"
    t.integer "height_cm"
    t.integer "matched_rules", default: [], null: false, array: true
    t.integer "pain_score"
    t.integer "respiratory_rate"
    t.uuid "rule_protocol_definition_id"
    t.uuid "screening_id", null: false
    t.integer "spo2"
    t.string "suggested_color"
    t.integer "systolic"
    t.decimal "temperature_c", precision: 3, scale: 1
    t.decimal "weight_kg", precision: 5, scale: 2
    t.index ["by_user_id"], name: "index_screening_revisions_on_by_user_id"
    t.index ["screening_id"], name: "index_screening_revisions_on_screening_id"
    t.check_constraint "(systolic IS NULL AND diastolic IS NULL) OR (systolic IS NOT NULL AND diastolic IS NOT NULL AND diastolic < systolic)", name: "ck_screening_revisions_bp"
    t.check_constraint "ciap2_code::text ~ '^[A-Z][0-9]{2}$'::text", name: "ck_screening_revisions_ciap2"
    t.check_constraint "suggested_color IS NULL OR final_color::text = suggested_color::text OR color_change_reason IS NOT NULL", name: "ck_screening_revisions_color_change"
    t.check_constraint "color_change_reason IS NULL OR length(btrim(color_change_reason)) BETWEEN 10 AND 500", name: "ck_screening_revisions_color_change_reason"
    t.check_constraint "complaint_note IS NULL OR length(complaint_note) <= 500", name: "ck_screening_revisions_complaint_note"
    t.check_constraint "diastolic IS NULL OR diastolic BETWEEN 20 AND 200", name: "ck_screening_revisions_diastolic"
    t.check_constraint "final_color::text = ANY (ARRAY['red'::text, 'yellow'::text, 'green'::text, 'blue'::text])", name: "ck_screening_revisions_final_color"
    t.check_constraint "(capillary_glucose IS NULL AND glucose_moment IS NULL) OR (capillary_glucose BETWEEN 10 AND 800 AND glucose_moment::text = ANY (ARRAY['fasting'::text, 'postprandial'::text, 'random'::text]))", name: "ck_screening_revisions_glucose"
    t.check_constraint "heart_rate IS NULL OR heart_rate BETWEEN 20 AND 250", name: "ck_screening_revisions_heart_rate"
    t.check_constraint "height_cm IS NULL OR height_cm BETWEEN 30 AND 250", name: "ck_screening_revisions_height"
    t.check_constraint "pain_score IS NULL OR pain_score BETWEEN 0 AND 10", name: "ck_screening_revisions_pain_score"
    t.check_constraint "respiratory_rate IS NULL OR respiratory_rate BETWEEN 4 AND 80", name: "ck_screening_revisions_respiratory_rate"
    t.check_constraint "spo2 IS NULL OR spo2 BETWEEN 50 AND 100", name: "ck_screening_revisions_spo2"
    t.check_constraint "suggested_color IS NULL OR suggested_color::text = ANY (ARRAY['red'::text, 'yellow'::text, 'green'::text, 'blue'::text])", name: "ck_screening_revisions_suggested_color"
    t.check_constraint "systolic IS NULL OR systolic BETWEEN 50 AND 300", name: "ck_screening_revisions_systolic"
    t.check_constraint "temperature_c IS NULL OR temperature_c BETWEEN 30 AND 45", name: "ck_screening_revisions_temperature"
    t.check_constraint "weight_kg IS NULL OR weight_kg BETWEEN 0.5 AND 400", name: "ck_screening_revisions_weight"
  end

  create_table "screenings", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "appointment_request_id"
    t.uuid "attendance_id", null: false
    t.string "cbo_code", null: false
    t.datetime "completed_at"
    t.datetime "created_at", null: false
    t.uuid "current_revision_id"
    t.string "destination"
    t.text "orientation_note"
    t.uuid "professional_link_id", null: false
    t.datetime "started_at", null: false
    t.uuid "started_by_user_id", null: false
    t.string "status", default: "in_progress", null: false
    t.datetime "updated_at", null: false
    t.index ["appointment_request_id"], name: "index_screenings_on_appointment_request_id"
    t.index ["attendance_id"], name: "index_screenings_on_attendance_id", unique: true
    t.index ["professional_link_id"], name: "index_screenings_on_professional_link_id"
    t.index ["started_by_user_id"], name: "index_screenings_on_started_by_user_id"
    t.index ["status"], name: "index_screenings_on_status"
    t.check_constraint "cbo_code::text ~ '^[0-9]{6}$'::text", name: "ck_screenings_cbo_code"
    t.check_constraint "(status::text = 'completed'::text) = (completed_at IS NOT NULL AND destination IS NOT NULL AND current_revision_id IS NOT NULL)", name: "ck_screenings_completion"
    t.check_constraint "destination IS NULL OR destination::text = ANY (ARRAY['same_day'::text, 'schedule'::text, 'oriented'::text, 'referred'::text])", name: "ck_screenings_destination"
    t.check_constraint "(destination IS DISTINCT FROM 'oriented' AND orientation_note IS NULL) OR (destination = 'oriented' AND orientation_note IS NOT NULL AND length(btrim(orientation_note)) BETWEEN 1 AND 500)", name: "ck_screenings_orientation"
    t.check_constraint "(destination IS NOT DISTINCT FROM 'schedule') = (appointment_request_id IS NOT NULL)", name: "ck_screenings_schedule"
    t.check_constraint "status::text = ANY (ARRAY['in_progress'::text, 'completed'::text, 'abandoned'::text])", name: "ck_screenings_status"
  end
```

Nas `add_foreign_key` do fim, em ordem alfabética: `add_foreign_key "appointment_requests", "screenings", column: "origin_screening_id"`, `add_foreign_key "screening_revisions", "protocol_definitions", column: "rule_protocol_definition_id"`, `add_foreign_key "screening_revisions", "screenings"`, `add_foreign_key "screening_revisions", "users", column: "by_user_id"`, `add_foreign_key "screenings", "appointment_requests"`, `add_foreign_key "screenings", "attendances"`, `add_foreign_key "screenings", "professional_links"`, `add_foreign_key "screenings", "screening_revisions", column: "current_revision_id"`, `add_foreign_key "screenings", "users", column: "started_by_user_id"`.

- [ ] **Step 8: Modelos**

```ruby
# app/models/screening.rb
# Escuta inicial (ADR 0030; spec 2026-10-07 §3): uma por atendimento, com
# revisões só de acréscimo. O atendimento não ganha estado: continua waiting
# durante e depois da escuta. Transições guardadas por screenings_guard.
class Screening < ApplicationRecord
  STATUSES = %w[in_progress completed abandoned].freeze
  DESTINATIONS = %w[same_day schedule oriented referred].freeze

  belongs_to :attendance
  belongs_to :started_by_user, class_name: "User"
  belongs_to :professional_link
  belongs_to :current_revision, class_name: "ScreeningRevision", optional: true
  belongs_to :appointment_request, optional: true
  has_many :revisions, class_name: "ScreeningRevision", dependent: :restrict_with_error

  scope :completed_screenings, -> { where(status: "completed") }
  scope :in_progress_screenings, -> { where(status: "in_progress") }

  def completed? = status == "completed"
  def in_progress? = status == "in_progress"
end
```

```ruby
# app/models/screening_revision.rb
# Uma revisão da escuta (ADR 0030; spec §3.1): queixa CIAP-2 (com a release),
# sinais vitais, cor sugerida e final. Só acréscimo (trigger).
class ScreeningRevision < ApplicationRecord
  COLORS = %w[red yellow green blue].freeze
  VITAL_COLUMNS = %w[systolic diastolic heart_rate respiratory_rate temperature_c spo2 capillary_glucose glucose_moment
                     weight_kg height_cm pain_score].freeze

  belongs_to :screening
  belongs_to :by_user, class_name: "User"

  def vitals = VITAL_COLUMNS.to_h { |column| [ column, self[column] ] }.compact
end
```

```ruby
# app/models/ledi_generation_failure.rb
# Ficha que não pôde nascer por falta de identificação (ADR 0030; spec §5).
# Uma aberta por fonte; "gerar de novo" tenta e resolve.
class LediGenerationFailure < ApplicationRecord
  REASONS = %w[unit_without_cnes professional_without_team professional_without_cns citizen_without_birth_date
               citizen_without_sex unknown_ciap2].freeze

  scope :unresolved, -> { where(resolved_at: nil) }

  validates :reason_codes, presence: true

  def resolved? = resolved_at.present?
end
```

Em `app/models/attendance.rb`:

```ruby
  OUTCOMES = %w[discharged referred return left scheduled_from_screening oriented].freeze
  # ADR 0030: destino da escuta → desfecho que fecha o atendimento de waiting.
  SCREENING_OUTCOMES = { "schedule" => "scheduled_from_screening", "oriented" => "oriented", "referred" => "referred" }.freeze
  # O que a rota de desfecho (Attendances::Close) aceita: os de escuta só saem de Screenings::Complete.
  CLOSE_OUTCOMES = %w[discharged referred return left].freeze
```
e, depois de `has_one :appointment_request ...`: `has_one :screening, dependent: :restrict_with_error`.

**Atenção:** `Attendances::Close` valida o desfecho com `Attendance::OUTCOMES.include?` — com os dois novos na lista, o dashboard poderia pedir `oriented` pela rota de desfecho. Em `app/commands/attendances/close.rb`, troque a primeira linha útil por:

```ruby
      return Result.fail(:invalid_outcome) unless Attendance::CLOSE_OUTCOMES.include?(outcome)
```

Em `app/models/appointment_request.rb`: `KINDS = %w[return referral triage screening].freeze` e `belongs_to :origin_screening, class_name: "Screening", optional: true`. Em `app/models/health_unit.rb`: `SCREENING_SCOPES = %w[walk_in all].freeze` e `validates :screening_scope, inclusion: { in: SCREENING_SCOPES }`. Em `app/commands/health_units/drain.rb`, no `AppointmentRequest.create!` do pedido novo, acrescente `origin_screening_id: request.origin_screening_id,`.

- [ ] **Step 9: Eventos, filtro de log e faixa de ADR**

Em `config/initializers/domain_events.rb`, no fim do bloco:

```ruby
  # Módulo 18 (ADR 0030; contratos §7): escuta e fichas não geradas; só trilha,
  # só ids. Nenhum texto livre da escuta entra em evento.
  DomainEvents.bind "screening.started", to: []
  DomainEvents.bind "screening.abandoned", to: []
  DomainEvents.bind "screening.completed", to: []
  DomainEvents.bind "screening.reassessed", to: []
  DomainEvents.bind "screening.viewed", to: []
  DomainEvents.bind "ledi.generation_failed", to: []
  DomainEvents.bind "ledi.generation_retried", to: []
  DomainEvents.bind "ledi.payload_purged", to: []
```

Em `spec/initializers/domain_events_bindings_spec.rb`:

```ruby
# Módulo 18 (ADR 0030): escuta e fichas não geradas, só trilha.
RSpec.describe "screening event bindings (ADR 0030)" do
  it "declares every module 18 city event with no consumer" do
    names = %w[screening.started screening.abandoned screening.completed screening.reassessed screening.viewed
               ledi.generation_failed ledi.generation_retried ledi.payload_purged]
    expect(DomainEvents.registry.keys).to include(*names)
    expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
  end
end
```

Em `config/initializers/filter_parameter_logging.rb`, depois da linha do ADR 0027:

```ruby
  # ADR 0030: escuta inicial — sinais vitais e queixa são dado de saúde
  # (`:note` e `:reason` já cobrem complaint_note, orientation_note e
  # color_change_reason).
  :vitals, :ciap2
```

(acrescente a vírgula depois de `:gender_identity`). Em `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..30).freeze`.

- [ ] **Step 10: Rode a migração nos bancos de teste e a spec**

```bash
psql -U rota_saude -d postgres -c "DROP DATABASE rota_saude_test_city_a" -c "DROP DATABASE rota_saude_test_city_b"
docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod18 api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/models/screening_tables_guard_spec.rb spec/services/city_schema_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/adr_pointers_spec.rb spec/models spec/commands/attendances spec/commands/health_units
```
Expected: PASS. Se `city_schema_spec` acusar diferença, a expressão do dump difere da migração: copie a da migração, não a do erro.

- [ ] **Step 11: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add config/scheduling/screening_cbos.yml db/city_migrate/20261007300001_add_screenings.rb db/city_schema.rb db/city_triggers.sql app/models/screening.rb app/models/screening_revision.rb app/models/ledi_generation_failure.rb app/models/attendance.rb app/models/appointment_request.rb app/models/health_unit.rb app/commands/attendances/close.rb app/commands/health_units/drain.rb config/initializers/domain_events.rb config/initializers/filter_parameter_logging.rb spec/initializers/domain_events_bindings_spec.rb spec/support/screening_helpers.rb spec/rails_helper.rb spec/adr_pointers_spec.rb spec/models/screening_tables_guard_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: add screenings, their revisions and the screening close guard

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 4: api#43 — a fila LEDI só com códigos (`last_error_codes`)

**Files:**
- Create: `db/city_migrate/20261007300002_ledi_outbox_error_codes.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql` (`rota_ledi_outbox_guard`)
- Create: `app/services/ledi/error_codes.rb`
- Delete: `app/services/ledi/error_text.rb`, `spec/services/ledi/error_text_spec.rb`
- Modify: `app/services/ledi/outcome.rb`, `app/services/ledi/delivery.rb`, `app/models/ledi_outbox_entry.rb`, `app/commands/ledi/resend.rb`, `app/queries/ledi/production_summary.rb`, `app/controllers/production_controller.rb`, `lib/ledi_crew.rb`
- Modify (specs que usavam `last_error`): `spec/services/ledi/outcome_spec.rb`, `spec/jobs/ledi/deliver_job_spec.rb`, `spec/models/ledi_outbox_entry_spec.rb`, `spec/models/create_ledi_outbox_migration_spec.rb`, `spec/queries/ledi/production_summary_spec.rb`, `spec/requests/production_spec.rb`, `spec/commands/ledi/resend_spec.rb`, `spec/jobs/ledi/publish_production_job_spec.rb`, `spec/lib/ledi_crew_spec.rb`, `spec/invariants/record_mode_invariants_spec.rb`
- Test: `spec/services/ledi/error_codes_spec.rb`, `spec/models/ledi_outbox_error_codes_migration_spec.rb`

**Interfaces:**
- Produces:
  - Colunas: `ledi_outbox.last_error_codes jsonb NOT NULL DEFAULT []`, `ledi_outbox.last_attempted_at`, `ledi_outbox.replaces_outbox_id` (FK para `ledi_outbox`, único); `last_error` removida; índice `idx_ledi_outbox_source` único só `WHERE status <> 'rejected'`.
  - `Ledi::ErrorCodes::FIELDS`, `::CODES`, `::UNKNOWN`, `.from_rejection(body) -> Array<Hash{"field","code"}>`, `.transport(code) -> [{ "field" => "transport", "code" => code }]`, `.valid?(codes) -> bool`, `.field_for(path)`, `.code_for(message)`.
  - `LediOutboxEntry#reject!(codes)`, `#retry_later!(codes:, wait:, give_up_after:, now:)`, `#accept!` (todos gravam `last_attempted_at`).
  - `Ledi::ProductionSummary.rejections(competence) -> [{ field:, code:, count: }]`.
  - `GET /production`: cada ficha tem `last_error_codes` no lugar de `last_error`; `rejections` com `field`/`code`/`count`.
  - `LediOutboxErrorCodes.codes_for(text) -> Array<Hash>` (classe da migração, regras congeladas): conversão do texto antigo.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/ledi/error_codes_spec.rb
require "rails_helper"

# api#43 (ADR 0030; spec §6): da resposta do PEC só ficam campo e código de
# listas fechadas — nunca valor, nome ou data do cidadão.
RSpec.describe Ledi::ErrorCodes do
  it "achata errosValidacao em campo (último segmento conhecido) e código pela mensagem" do
    body = { descricaoErro: "Erro de validação",
             errosValidacao: { "atendimentosIndividuais[0]" => { cpfCidadao: "CPF 12345678909 inválido",
                                                                  dataNascimento: "Data 10/05/1980 posterior ao atendimento" },
                               "headerTransport.ine" => "INE obrigatório",
                               "lotacao.cboCodigo_2002" => "CBO 322205 não permitido para o modelo",
                               "nomeEstranho" => "MARIA DA SILVA duplicada" } }.to_json
    expect(described_class.from_rejection(body)).to eq([
      { "field" => "cpfCidadao", "code" => "invalid" }, { "field" => "dataNascimento", "code" => "out_of_range" },
      { "field" => "ine", "code" => "required" }, { "field" => "cboCodigo_2002", "code" => "not_allowed" },
      { "field" => "other", "code" => "duplicate" }
    ])
  end

  it "sem errosValidacao usa a descrição só para classificar; corpo que não é JSON vira desconhecido" do
    expect(described_class.from_rejection({ descricaoErro: "Campo obrigatório ausente" }.to_json))
      .to eq([ { "field" => "other", "code" => "required" } ])
    expect(described_class.from_rejection("CNES 1234567 não pertence ao município")).to eq([ described_class::UNKNOWN ])
    expect(described_class.from_rejection(nil)).to eq([ described_class::UNKNOWN ])
  end

  it "nunca devolve valor: só chaves das listas fechadas, e no máximo 20" do
    body = { errosValidacao: (1..30).to_h { |i| [ "campo#{i}", "valor 529.982.247-25 inválido" ] } }.to_json
    codes = described_class.from_rejection(body)
    expect(codes.size).to be <= 20
    expect(described_class.valid?(codes)).to be(true)
    expect(codes.to_json).not_to include("529", "valor")
  end

  it "transporte e validação" do
    expect(described_class.transport("unreachable")).to eq([ { "field" => "transport", "code" => "unreachable" } ])
    expect(described_class.transport("qualquer")).to eq([ { "field" => "transport", "code" => "unknown" } ])
    expect(described_class.valid?([ { "field" => "cpfCidadao", "code" => "invalid" } ])).to be(true)
    expect(described_class.valid?([ { "field" => "cpfCidadao", "code" => "invalid", "value" => "x" } ])).to be(false)
    expect(described_class.valid?([ { "field" => "Maria", "code" => "invalid" } ])).to be(false)
  end
end
```

```ruby
# spec/models/ledi_outbox_error_codes_migration_spec.rb
require "rails_helper"
require Rails.root.join("db/city_migrate/20261007300002_ledi_outbox_error_codes.rb").to_s

# api#43 (spec §6): a migração converte o texto antigo no que der e zera o
# resto; o texto nunca sobrevive.
RSpec.describe "Migração de cidade 20261007300002 (LediOutboxErrorCodes): conversão do last_error" do
  {
    "Erro de validação; cpfCidadao: CPF [número] inválido" => [ { "field" => "cpfCidadao", "code" => "invalid" } ],
    "Erro de validação; headerTransport.ine: obrigatório; cnes: não pertence ao município" =>
      [ { "field" => "ine", "code" => "required" }, { "field" => "cnes", "code" => "unknown" } ],
    "HTTP 503" => [ { "field" => "transport", "code" => "http_error" } ],
    "PEC inacessível" => [ { "field" => "transport", "code" => "unreachable" } ],
    "endereço do PEC inválido" => [ { "field" => "transport", "code" => "invalid_url" } ],
    "login no PEC respondeu 500" => [ { "field" => "transport", "code" => "login_failed" } ],
    "erro interno (RuntimeError)" => [ { "field" => "transport", "code" => "internal_error" } ],
    "CNES 9999991 não pertence ao município da instalação." => [ { "field" => "other", "code" => "unknown" } ],
    "MARIA DA SILVA nascida em 10/05/1980" => [ { "field" => "other", "code" => "unknown" } ]
  }.each do |text, codes|
    it("#{text.truncate(40)} → #{codes.map { |c| c.values.join('/') }.join(', ')}") do
      expect(LediOutboxErrorCodes.codes_for(text)).to eq(codes)
    end
  end

  it "texto vazio não vira código" do
    expect(LediOutboxErrorCodes.codes_for(nil)).to eq([])
    expect(LediOutboxErrorCodes.codes_for("  ")).to eq([])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/ledi/error_codes_spec.rb spec/models/ledi_outbox_error_codes_migration_spec.rb`
Expected: FAIL (`uninitialized constant Ledi::ErrorCodes`; `cannot load such file ... 20261007300002`).

- [ ] **Step 3: `Ledi::ErrorCodes`**

```ruby
# app/services/ledi/error_codes.rb
# api#43 (ADR 0030; spec §6): o que a fila guarda de um erro do PEC é só
# [{ field, code }], de listas fechadas. A resposta crua é lida em memória para
# classificar e descartada: nada de mensagem, valor, nome ou data. Chave de
# errosValidacao fora da lista vira "other" (uma chave também poderia carregar
# dado). O código sai de palavras da mensagem, nunca do texto.
module Ledi
  module ErrorCodes
    FIELDS = %w[uuidFicha headerTransport profissionalCNS cboCodigo_2002 cnes ine dataAtendimento codigoIbgeMunicipio
                cpfCidadao cnsCidadao cns dataNascimento dtNascimento sexo turno localDeAtendimento localAtendimento
                tipoAtendimento condutas problemasCondicoes ciap medicoes procedimentos dataHoraInicialAtendimento
                dataHoraFinalAtendimento transport other].freeze
    CODES = %w[required invalid not_allowed out_of_range duplicate http_error unreachable invalid_url login_failed
               internal_error unknown].freeze
    UNKNOWN = { "field" => "other", "code" => "unknown" }.freeze
    MAX = 20

    module_function

    def from_rejection(body)
      parsed = parse(body)
      return [ UNKNOWN.dup ] unless parsed.is_a?(Hash)

      codes = pairs(parsed["errosValidacao"], nil).map { |path, message| { "field" => field_for(path), "code" => code_for(message) } }
      codes = [ { "field" => "other", "code" => code_for(parsed["descricaoErro"]) } ] if codes.empty?
      codes.uniq.first(MAX)
    end

    def transport(code) = [ { "field" => "transport", "code" => CODES.include?(code.to_s) ? code.to_s : "unknown" } ]

    def valid?(codes)
      codes.is_a?(Array) && codes.all? do |c|
        c.is_a?(Hash) && c.keys.map(&:to_s).sort == %w[code field] &&
          FIELDS.include?(c.with_indifferent_access[:field]) && CODES.include?(c.with_indifferent_access[:code])
      end
    end

    # "atendimentosIndividuais[0].cpfCidadao" → "cpfCidadao"; nada conhecido → "other".
    def field_for(path)
      path.to_s.split(/[.\[\]]+/).reverse.find { |segment| FIELDS.include?(segment) } || "other"
    end

    def code_for(message)
      text = I18n.transliterate(message.to_s.downcase)
      case text
      when /obrigatori|requerid|nao (foi )?(informad|preenchid)|ausente/ then "required"
      when /duplicad|ja (foi )?(enviad|recebid|cadastrad)/ then "duplicate"
      when /maxim|minim|fora (do|da) (faixa|intervalo)|superior|inferior|posterior|anterior/ then "out_of_range"
      when /nao (e |eh )?permitid|nao pode|nao aceit/ then "not_allowed"
      when /invalid|incorret|formato/ then "invalid"
      else "unknown"
      end
    end

    def parse(body)
      JSON.parse(body.to_s)
    rescue JSON::ParserError
      nil
    end

    def pairs(node, prefix)
      case node
      when Hash then node.flat_map { |k, v| pairs(v, [ prefix, k ].compact.join(".")) }
      when Array then node.flat_map { |v| pairs(v, prefix) }
      when nil then []
      else [ [ prefix, node ] ]
      end
    end
    private_class_method :parse, :pairs
  end
end
```

- [ ] **Step 4: A migração (com a conversão congelada)**

```ruby
# db/city_migrate/20261007300002_ledi_outbox_error_codes.rb
# api#43 (ADR 0030; spec 2026-10-07 §5–§6): a fila LEDI deixa de guardar texto
# do PEC. last_error vira last_error_codes ([{ field, code }]); a conversão
# do que existe é feita aqui, com regras congeladas (não chama o app), e o
# texto some. last_attempted_at mede os 90 dias da purga; replaces_outbox_id
# liga a ficha regerada à recusada; o índice único passa a ignorar recusadas.
class LediOutboxErrorCodes < ActiveRecord::Migration[8.1]
  FIELDS = %w[uuidFicha headerTransport profissionalCNS cboCodigo_2002 cnes ine dataAtendimento codigoIbgeMunicipio
              cpfCidadao cnsCidadao cns dataNascimento dtNascimento sexo turno localDeAtendimento localAtendimento
              tipoAtendimento condutas problemasCondicoes ciap medicoes procedimentos dataHoraInicialAtendimento
              dataHoraFinalAtendimento].freeze
  TRANSPORT = [ [ /\AHTTP \d+\z/, "http_error" ], [ /\APEC inacess/, "unreachable" ],
                [ /\Aendereço do PEC inválido\z/, "invalid_url" ], [ /\Alogin no PEC respondeu/, "login_failed" ],
                [ /\Aerro interno \(/, "internal_error" ] ].freeze

  # Texto antigo (Ledi::ErrorText.sanitize de "descrição; campo: msg; …") → códigos.
  def self.codes_for(text)
    text = text.to_s.strip
    return [] if text.empty?

    TRANSPORT.each { |pattern, code| return [ { "field" => "transport", "code" => code } ] if text.match?(pattern) }
    codes = text.split("; ").filter_map do |part|
      key, message = part.split(": ", 2)
      next unless message

      field = key.split(/[.\[\]]+/).reverse.find { |segment| FIELDS.include?(segment) }
      field && { "field" => field, "code" => code_for(message) }
    end
    codes.empty? ? [ { "field" => "other", "code" => "unknown" } ] : codes.uniq.first(20)
  end

  def self.code_for(message)
    text = I18n.transliterate(message.to_s.downcase)
    return "required" if text.match?(/obrigatori|requerid|ausente/)
    return "not_allowed" if text.match?(/nao (e |eh )?permitid|nao pode|nao aceit/)
    return "invalid" if text.match?(/invalid|incorret|formato/)

    "unknown"
  end

  def up
    add_column :ledi_outbox, :last_error_codes, :jsonb, null: false, default: []
    add_column :ledi_outbox, :last_attempted_at, :datetime
    add_column :ledi_outbox, :replaces_outbox_id, :uuid
    add_foreign_key :ledi_outbox, :ledi_outbox, column: :replaces_outbox_id
    add_index :ledi_outbox, :replaces_outbox_id, unique: true, name: "idx_ledi_outbox_replaces"

    # O guarda recusa qualquer UPDATE em accepted; o backfill de
    # last_attempted_at passa por todas as linhas.
    execute "ALTER TABLE ledi_outbox DISABLE TRIGGER ledi_outbox_guard"
    select_rows("SELECT id, last_error FROM ledi_outbox WHERE last_error IS NOT NULL").each do |id, text|
      execute "UPDATE ledi_outbox SET last_error_codes = #{quote(self.class.codes_for(text).to_json)}::jsonb WHERE id = #{quote(id)}"
    end
    execute "UPDATE ledi_outbox SET last_attempted_at = updated_at WHERE attempts > 0"
    execute "ALTER TABLE ledi_outbox ENABLE TRIGGER ledi_outbox_guard"

    remove_check_constraint :ledi_outbox, name: "ck_ledi_outbox_rejected_error"
    remove_column :ledi_outbox, :last_error
    add_check_constraint :ledi_outbox, "jsonb_typeof(last_error_codes) = 'array'::text", name: "ck_ledi_outbox_error_codes"
    add_check_constraint :ledi_outbox,
                         "status::text <> 'rejected'::text OR jsonb_array_length(last_error_codes) > 0 OR payload IS NULL",
                         name: "ck_ledi_outbox_rejected_error"
    remove_index :ledi_outbox, name: "idx_ledi_outbox_source"
    add_index :ledi_outbox, %i[source_type source_id ficha_type], unique: true, name: "idx_ledi_outbox_source",
                                                                  where: "(status)::text <> 'rejected'::text"
    add_index :ledi_outbox, %i[source_type source_id], name: "idx_ledi_outbox_source_lookup"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end

  private

  def quote(value) = connection.quote(value)
end
```

Em `rota_ledi_outbox_guard` (`db/city_triggers.sql`), acrescente à lista de identidade `OR NEW.replaces_outbox_id IS DISTINCT FROM OLD.replaces_outbox_id`.

Em `db/city_schema.rb`: `define(version: 2026_10_07_300002)`; em `ledi_outbox`, tire `t.string "last_error", limit: 500`, acrescente `t.jsonb "last_error_codes", default: [], null: false`, `t.datetime "last_attempted_at"`, `t.uuid "replaces_outbox_id"`, troque o índice `idx_ledi_outbox_source` por `t.index ["source_type", "source_id", "ficha_type"], name: "idx_ledi_outbox_source", unique: true, where: "((status)::text <> 'rejected'::text)"`, acrescente `t.index ["source_type", "source_id"], name: "idx_ledi_outbox_source_lookup"` e `t.index ["replaces_outbox_id"], name: "idx_ledi_outbox_replaces", unique: true`, troque `ck_ledi_outbox_rejected_error` pela expressão nova e acrescente `t.check_constraint "jsonb_typeof(last_error_codes) = 'array'::text", name: "ck_ledi_outbox_error_codes"`; nas FKs, `add_foreign_key "ledi_outbox", "ledi_outbox", column: "replaces_outbox_id"`.

Em `spec/models/create_ledi_outbox_migration_spec.rb`, a migração antiga recria a tabela sem as colunas novas: no `migrate(:up)`, rode as duas:

```ruby
  def migrate(direction)
    ActiveRecord::Migration.suppress_messages do
      CreateLediOutbox.new.exec_migration(conn, direction)
      LediOutboxErrorCodes.new.exec_migration(conn, :up) if direction == :up
    end
    LediOutboxEntry.reset_column_information
  end
```
e acrescente `require Rails.root.join("db/city_migrate/20261007300002_ledi_outbox_error_codes.rb").to_s` no topo.

- [ ] **Step 5: O modelo, o envio, o reenvio e o painel com códigos**

Em `app/models/ledi_outbox_entry.rb`, troque `accept!`, `reject!` e `retry_later!`:

```ruby
  def accept!
    transaction do
      update!(status: "accepted", accepted_at: Time.current, payload: nil, last_error_codes: [], attempts: attempts + 1,
              last_attempted_at: Time.current)
      DomainEvents.publish("ledi.ficha_accepted", outbox_id: id, ficha_type: ficha_type, competence: competence)
    end
  end

  # api#43: só códigos de lista fechada; qualquer outra coisa vira desconhecido.
  def reject!(codes)
    codes = Ledi::ErrorCodes.valid?(codes) && codes.any? ? codes : [ Ledi::ErrorCodes::UNKNOWN.dup ]
    transaction do
      update!(status: "rejected", last_error_codes: codes, attempts: attempts + 1, last_attempted_at: Time.current)
      DomainEvents.publish("ledi.ficha_rejected", outbox_id: id, ficha_type: ficha_type, competence: competence)
    end
  end

  # Falha transitória: nova tentativa depois de `wait`, ou failed quando a
  # primeira tentativa já passou de `give_up_after`.
  def retry_later!(codes:, wait:, give_up_after:, now: Time.current)
    started = first_attempt_at || now
    status = started <= now - give_up_after ? "failed" : "pending"
    update!(status: status, attempts: attempts + 1, last_error_codes: codes, last_attempted_at: now,
            next_attempt_at: now + wait)
  end
```

Em `app/services/ledi/outcome.rb`, apague `message` e `flatten` (e o comentário deles); fica só `classify`.

Em `app/services/ledi/delivery.rb`: a linha de comentário "nem log, nem evento, nem last_error" passa a "nem log, nem evento, nem a fila"; `LoginDown` carrega um código; troque `deliver`, `cookie`, `reject` e `retry_later`:

```ruby
    def deliver(entry)
      reply = post(entry)
      return pause! if reply == :unauthorized

      case Ledi::Outcome.classify(reply.status, reply.body)
      when :accepted then entry.accept!
      # api#43: o corpo do 400 vira códigos aqui e é descartado.
      when :rejected then entry.reject!(Ledi::ErrorCodes.from_rejection(reply.body))
      else retry_later(entry, "http_error")
      end
    rescue LoginDown => e
      retry_later(entry, e.message)
      :halted
    rescue Ledi::PecClient::InvalidUrl
      retry_later(entry, "invalid_url")
      :halted
    rescue Ledi::PecClient::Unreachable
      retry_later(entry, "unreachable")
    rescue StandardError => e
      # R32: erro inesperado numa ficha não trava o lote. Só a classe vai para
      # o relatório de erro; a fila guarda só o código.
      Rails.error.report(RuntimeError.new("ledi delivery: #{e.class.name}"), handled: true, severity: :error)
      retry_later(entry, "internal_error")
    end
```

(o relatório leva uma exceção nova só com o nome da classe: a mensagem original pode carregar dado da ficha.)

```ruby
    def cookie
      Ledi::SessionCache.fetch(@cache_key) { @client.login.cookie }
    rescue Ledi::PecClient::InvalidUrl
      raise LoginDown, "invalid_url"
    rescue Ledi::PecClient::Unreachable
      raise LoginDown, "unreachable"
    rescue Ledi::PecClient::Failed
      raise LoginDown, "login_failed"
    end

    def retry_later(entry, code)
      entry.retry_later!(codes: Ledi::ErrorCodes.transport(code), wait: Ledi::Backoff.wait(entry.attempts + 1),
                         give_up_after: Ledi::Backoff::GIVE_UP_AFTER)
    end
```

Apague o método `reject` e as constantes `INVALID_URL_MESSAGE` (sem uso). Em `app/commands/ledi/resend.rb`, `last_error: nil` → `last_error_codes: []`.

Em `app/queries/ledi/production_summary.rb`:

```ruby
    # api#43: recusas agrupadas por campo e código (nunca texto do PEC).
    def rejections(competence)
      LediOutboxEntry.for_competence(competence).where(status: "rejected")
                     .joins("CROSS JOIN LATERAL jsonb_array_elements(ledi_outbox.last_error_codes) AS error_code")
                     .group(Arel.sql("error_code->>'field'"), Arel.sql("error_code->>'code'")).count
                     .map { |(field, code), count| { field: field, code: code, count: count } }
                     .sort_by { |row| [ -row[:count], row[:field], row[:code] ] }
    end
```
(e o comentário do topo: "Recusas agrupadas por campo e código — nunca texto do PEC.").

Em `app/controllers/production_controller.rb#ficha_json`: `last_error: entry.last_error` → `last_error_codes: entry.last_error_codes`.

Em `lib/ledi_crew.rb`:

```ruby
  REJECTIONS = [ [ { "field" => "cnes", "code" => "not_allowed" } ],
                 [ { "field" => "cboCodigo_2002", "code" => "not_allowed" } ] ].freeze
```
e, em `create_entry`, `attrs[:last_error_codes] = REJECTIONS[index % REJECTIONS.size] if status == "rejected"` e `attrs[:last_error_codes] = Ledi::ErrorCodes.transport("http_error") if status == "failed"`.

- [ ] **Step 6: Atualize as specs que liam `last_error`**

- `spec/services/ledi/outcome_spec.rb`: apague o `describe ".message"` inteiro (coberto por `error_codes_spec.rb`).
- `spec/services/ledi/error_text_spec.rb` e `app/services/ledi/error_text.rb`: `git rm`.
- `spec/jobs/ledi/deliver_job_spec.rb`: troque cada `:last_error`/`"last_error"` por `:last_error_codes`/`"last_error_codes"` e os valores:
  - 400 → `"last_error_codes" => [ { "field" => "cpfCidadao", "code" => "invalid" } ]` e o título "400: rejected só com códigos, sem nova tentativa sozinha"; acrescente `expect(entry.reload.last_error_codes.to_json).not_to include("12345678909")`;
  - 5xx → `[ { "field" => "transport", "code" => "http_error" } ]`; timeout → `[ { "field" => "transport", "code" => "unreachable" } ]`;
  - erro inesperado → `[ { "field" => "transport", "code" => "internal_error" } ]` e `expect(second.last_error_codes.to_json).not_to include("segredo")`;
  - login inacessível → `"unreachable"`, login com erro → `"login_failed"` (a tabela do `each` passa a `[ error, code ]` e a expectativa a `Ledi::ErrorCodes.transport(code)`);
  - URL inválida (login e envio) → `Ledi::ErrorCodes.transport("invalid_url")`;
  - os "resto do lote" → `"last_error_codes" => []`.
- `spec/models/ledi_outbox_entry_spec.rb`: `entry.update_columns(last_error: "x")` → `entry.update_columns(last_error_codes: [ { "field" => "other", "code" => "unknown" } ])`; `other.reject!("CNES inválido")` → `other.reject!([ { "field" => "cnes", "code" => "invalid" } ])`.
- `spec/queries/ledi/production_summary_spec.rb`: o parâmetro `error:` vira `codes:` (`last_error_codes: codes || []`); as recusas `[ { "field" => "cnes", "code" => "not_allowed" } ]` (duas vezes) e `[ { "field" => "cboCodigo_2002", "code" => "not_allowed" } ]`; o failed `Ledi::ErrorCodes.transport("http_error")`; a expectativa de agrupamento `[ { field: "cnes", code: "not_allowed", count: 2 }, { field: "cboCodigo_2002", code: "not_allowed", count: 1 } ]` (título "agrupa recusas por campo e código").
- `spec/requests/production_spec.rb`: `entry!` com `codes:`; chaves das fichas `%w[id ficha_type status attempts last_error_codes created_at accepted_at]`; a recusa de exemplo `[ { "field" => "cpfCidadao", "code" => "invalid" } ]` e a expectativa sobre ela; o teste de agrupamento usa `[ { "field" => "cnsCidadao", "code" => "not_allowed" } ]` e espera `[ { "field" => "cnsCidadao", "code" => "not_allowed", "count" => 2 } ]`; no reenvio, `body.slice("id", "status", "last_error_codes")` com `"last_error_codes" => []`; `entry!("rejected", error: "x")` → `entry!("rejected", codes: Ledi::ErrorCodes.transport("unknown"))`.
- `spec/commands/ledi/resend_spec.rb`: `last_error: "CNES inválido"` → `last_error_codes: [ { "field" => "cnes", "code" => "invalid" } ]`; expectativa `"last_error_codes" => []`.
- `spec/jobs/ledi/publish_production_job_spec.rb`: `attrs[:last_error] = "x"` → `attrs[:last_error_codes] = [ Ledi::ErrorCodes::UNKNOWN ]`.
- `spec/lib/ledi_crew_spec.rb`: `distinct.pluck(:last_error)` → `distinct.pluck(:last_error_codes)`.
- `spec/invariants/record_mode_invariants_spec.rb:234`: `LediOutboxEntry.pluck(:last_error).to_json` → `LediOutboxEntry.pluck(:last_error_codes).to_json`.

- [ ] **Step 7: Bancos de teste, specs e varredura**

```bash
psql -U rota_saude -d postgres -c "DROP DATABASE rota_saude_test_city_a" -c "DROP DATABASE rota_saude_test_city_b"
docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod18 api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/ledi spec/jobs/ledi spec/commands/ledi spec/models/ledi_outbox_entry_spec.rb spec/models/create_ledi_outbox_migration_spec.rb spec/models/ledi_outbox_error_codes_migration_spec.rb spec/queries/ledi spec/requests/production_spec.rb spec/lib/ledi_crew_spec.rb spec/invariants/record_mode_invariants_spec.rb spec/services/city_schema_spec.rb
grep -rn "last_error\b\|ErrorText\|Outcome.message" apps/api/.claude/mod18/app apps/api/.claude/mod18/lib apps/api/.claude/mod18/spec | grep -v analytics
```
Expected: PASS; o grep só acha `last_error` em código de Analytics (`AnalyticsRun`, outra coisa) — nada da fila LEDI.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 rm app/services/ledi/error_text.rb spec/services/ledi/error_text_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add db/city_migrate/20261007300002_ledi_outbox_error_codes.rb db/city_schema.rb db/city_triggers.sql app/services/ledi/error_codes.rb app/services/ledi/outcome.rb app/services/ledi/delivery.rb app/models/ledi_outbox_entry.rb app/commands/ledi/resend.rb app/queries/ledi/production_summary.rb app/controllers/production_controller.rb lib/ledi_crew.rb spec/services/ledi/error_codes_spec.rb spec/models/ledi_outbox_error_codes_migration_spec.rb spec/services/ledi/outcome_spec.rb spec/jobs/ledi/deliver_job_spec.rb spec/models/ledi_outbox_entry_spec.rb spec/models/create_ledi_outbox_migration_spec.rb spec/queries/ledi/production_summary_spec.rb spec/requests/production_spec.rb spec/commands/ledi/resend_spec.rb spec/jobs/ledi/publish_production_job_spec.rb spec/lib/ledi_crew_spec.rb spec/invariants/record_mode_invariants_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "fix: keep only error codes in the LEDI outbox, never PEC text (api#43)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: api#43 — purga de 90 dias e exclusão limpando a fila

**Files:**
- Create: `app/jobs/ledi/purge_stale_payloads_job.rb`, `app/services/ledi/citizen_sources.rb`
- Modify: `app/commands/citizens/erase.rb`, `config/recurring.yml`
- Test: `spec/jobs/ledi/purge_stale_payloads_job_spec.rb`, `spec/services/ledi/citizen_sources_spec.rb`, `spec/commands/citizens/erase_ledi_spec.rb`

**Interfaces:**
- Consumes: `LediOutboxEntry.last_attempted_at` (Task 4), `Screening` (Task 3).
- Produces: `Ledi::PurgeStalePayloadsJob::RETENTION_DAYS == 90`, `#perform(older_than_days: 90)`; `Ledi::CitizenSources.entries_for(citizen_ids) -> Relation`, `.scrub!(citizen_ids) -> Integer`; evento `ledi.payload_purged { count }`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/jobs/ledi/purge_stale_payloads_job_spec.rb
require "rails_helper"

# api#43 (ADR 0030, Invariantes): nenhum payload de ficha rejected/failed com
# última tentativa há mais de 90 dias.
RSpec.describe Ledi::PurgeStalePayloadsJob do
  include ActiveSupport::Testing::TimeHelpers

  let!(:city) { register_test_city! }

  def entry!(status, attempted_at:)
    attrs = { uuid: "1234567-#{SecureRandom.uuid}", ficha_type: "procedimento", competence: "202606",
              source_type: "synthetic", source_id: SecureRandom.uuid, ledi_version: "8.7.0", status: status,
              next_attempt_at: Time.current, attempts: 1, last_attempted_at: attempted_at, bytes: "x".b }
    attrs[:last_error_codes] = [ { "field" => "cnes", "code" => "invalid" } ] if status == "rejected"
    CityConnection.with(city) { LediOutboxEntry.create!(attrs) }
  end

  def payload_of(entry) = CityConnection.with(city) { entry.reload.payload }

  it "apaga o conteúdo de recusada/falha com 90+ dias; o resto fica; evento só com a contagem" do
    old_rejected = entry!("rejected", attempted_at: 91.days.ago)
    old_failed = entry!("failed", attempted_at: 120.days.ago)
    recent = entry!("rejected", attempted_at: 89.days.ago)
    pending = entry!("pending", attempted_at: 200.days.ago)

    described_class.perform_now

    expect([ old_rejected, old_failed ].map { |e| payload_of(e) }).to eq([ nil, nil ])
    expect([ recent, pending ].map { |e| payload_of(e) }).to all(be_present)
    events = CityConnection.with(city) { DomainEvent.where(name: "ledi.payload_purged").pluck(:payload) }
    expect(events).to eq([ { "count" => 2 } ])
  end

  it "sem nada vencido não publica evento; o recurring.yml usa os mesmos 90 dias" do
    entry!("rejected", attempted_at: 10.days.ago)
    described_class.perform_now
    expect(CityConnection.with(city) { DomainEvent.where(name: "ledi.payload_purged").count }).to eq(0)
    task = YAML.load_file(Rails.root.join("config/recurring.yml")).dig("default", "ledi_purge_stale_payloads")
    expect(task).to include("class" => "Ledi::PurgeStalePayloadsJob", "args" => { "older_than_days" => 90 })
  end
end
```

```ruby
# spec/services/ledi/citizen_sources_spec.rb
require "rails_helper"

# api#43 (spec §6): a exclusão confirmada apaga payload e códigos das linhas
# da fila cujas fontes são do cidadão. Aceita não muda (já sem conteúdo).
RSpec.describe Ledi::CitizenSources do
  before { Current.city = TEST_CITY_A; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }

  def screening_for(citizen)
    attendance = walk_in_attendance!(unit, citizen: citizen)
    link = nurse.professional.links.active.sole
    Screening.create!(attendance: attendance, status: "in_progress", started_by_user: nurse, professional_link: link,
                      cbo_code: link.cbo_code, started_at: Time.current)
  end

  def entry!(source_id, status)
    attrs = { uuid: "1234567-#{SecureRandom.uuid}", ficha_type: "procedimento", competence: "202610",
              source_type: "Screening", source_id: source_id, ledi_version: "8.7.0", status: status,
              next_attempt_at: Time.current, bytes: "x".b }
    attrs[:last_error_codes] = [ { "field" => "cnes", "code" => "invalid" } ] if status == "rejected"
    LediOutboxEntry.create!(attrs)
  end

  it "limpa as linhas do par (pendente vira failed) e não toca as dos outros" do
    mine = screening_for(screening_citizen!(1))
    theirs = screening_for(screening_citizen!(2))
    rejected = entry!(mine.id, "rejected")
    pending = LediOutboxEntry.create!(uuid: "1234567-#{SecureRandom.uuid}", ficha_type: "atendimento_individual",
                                      competence: "202610", source_type: "Screening", source_id: mine.id,
                                      ledi_version: "8.7.0", status: "pending", next_attempt_at: Time.current, bytes: "x".b)
    other = entry!(theirs.id, "rejected")

    expect(described_class.scrub!([ mine.attendance.citizen_id ])).to eq(2)
    expect(rejected.reload.slice(:payload, :last_error_codes, :status))
      .to eq("payload" => nil, "last_error_codes" => [], "status" => "rejected")
    expect(pending.reload.slice(:payload, :status)).to eq("payload" => nil, "status" => "failed")
    expect(other.reload.payload).to be_present
  end
end
```

```ruby
# spec/commands/citizens/erase_ledi_spec.rb
require "rails_helper"

# api#43: a exclusão confirmada chama a limpeza da fila para os pares do CPF.
RSpec.describe Citizens::Erase, "fila LEDI" do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  it "chama Ledi::CitizenSources.scrub! com os ids dos pares" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    completed_web_triage_for(citizen)
    verifier = User.create!(email_address: "v-#{SecureRandom.hex(3)}@x.com", password: "secret123")
    admin = User.create!(email_address: "a-#{SecureRandom.hex(3)}@x.com", password: "secret123")
    request = Citizens::RequestErasure.call(cpf: citizen.cpf, document_checked: true, by: verifier).payload[:request]
    allow(Ledi::CitizenSources).to receive(:scrub!).and_call_original

    expect(described_class.call(request: request, by: admin)).to be_ok
    expect(Ledi::CitizenSources).to have_received(:scrub!).with([ citizen.id ])
  end
end
```

(Mesmo arranjo de `spec/commands/citizens/erase_spec.rb`: pedido por um servidor, confirmação por outro.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/jobs/ledi/purge_stale_payloads_job_spec.rb spec/services/ledi/citizen_sources_spec.rb spec/commands/citizens/erase_ledi_spec.rb`
Expected: FAIL (`uninitialized constant Ledi::PurgeStalePayloadsJob`, `Ledi::CitizenSources`).

- [ ] **Step 3: Implemente**

```ruby
# app/jobs/ledi/purge_stale_payloads_job.rb
# api#43 (ADR 0030; spec §6): o conteúdo de ficha recusada ou que desistiu é
# apagado 90 dias depois da última tentativa. Corrigir uma recusada regera a
# ficha da escuta (não reaproveita o conteúdo), então nada se perde. Diário,
# por cidade; o evento leva só a contagem.
module Ledi
  class PurgeStalePayloadsJob < ApplicationJob
    prepend EachCityJob
    queue_as :housekeeping

    RETENTION_DAYS = 90

    def perform(older_than_days: RETENTION_DAYS)
      cutoff = older_than_days.days.ago
      count = LediOutboxEntry.where(status: %w[rejected failed]).where.not(payload: nil)
                             .where("COALESCE(last_attempted_at, created_at) < ?", cutoff)
                             .update_all(payload: nil, updated_at: Time.current)
      DomainEvents.publish("ledi.payload_purged", count: count) if count.positive?
    end
  end
end
```

```ruby
# app/services/ledi/citizen_sources.rb
# Linhas da fila LEDI cujas fontes são de um cidadão (api#43; spec §6). Hoje a
# única fonte real é a escuta (Screening → atendimento → cidadão). A exclusão
# confirmada apaga conteúdo e códigos; pendente vira failed para não ir ao PEC
# sem conteúdo. Aceita é imutável e já não tem conteúdo.
module Ledi
  module CitizenSources
    module_function

    def entries_for(citizen_ids)
      screenings = Screening.joins(:attendance).where(attendances: { citizen_id: citizen_ids }).select(:id)
      LediOutboxEntry.where(source_type: "Screening", source_id: screenings)
    end

    def scrub!(citizen_ids)
      entries_for(citizen_ids).where.not(status: "accepted").update_all(
        "payload = NULL, last_error_codes = '[]'::jsonb, " \
        "status = CASE WHEN status = 'pending' THEN 'failed' ELSE status END, updated_at = now()"
      )
    end
  end
end
```

Em `app/commands/citizens/erase.rb#erase_pair`, depois de `AppointmentNotice.where(...).delete_all`:

```ruby
      # ADR 0030 (api#43): conteúdo e códigos da fila LEDI das fontes do par.
      Ledi::CitizenSources.scrub!([ citizen.id ])
```

Em `config/recurring.yml`, depois de `ledi_publish_production`:

```yaml
  ledi_purge_stale_payloads:
    class: Ledi::PurgeStalePayloadsJob
    queue: housekeeping
    schedule: "every day at 5:15am America/Sao_Paulo"
    args: { older_than_days: 90 }
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/jobs/ledi/purge_stale_payloads_job_spec.rb spec/services/ledi/citizen_sources_spec.rb spec/commands/citizens spec/config`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add app/jobs/ledi/purge_stale_payloads_job.rb app/services/ledi/citizen_sources.rb app/commands/citizens/erase.rb config/recurring.yml spec/jobs/ledi/purge_stale_payloads_job_spec.rb spec/services/ledi/citizen_sources_spec.rb spec/commands/citizens/erase_ledi_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: purge stale LEDI payloads after 90 days and on erasure (api#43)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 3 — A escuta (F-18.1 a F-18.4)

### Task 6: `Screenings::VitalSigns` — plausibilidade, alerta e IMC (puro)

**Files:**
- Create: `app/services/screenings/vital_signs.rb`
- Test: `spec/services/screenings/vital_signs_spec.rb`

**Interfaces:**
- Produces (`Screenings::VitalSigns`, sem banco):
  - `FIELDS` (as 11 colunas de `ScreeningRevision::VITAL_COLUMNS`), `PLAUSIBLE`, `GLUCOSE_MOMENTS`, `ALERT_RULES`;
  - `.parse(raw) -> Result` — ok: `payload { values: Hash<String, Integer|BigDecimal|String>, alerts: Array<String>, bmi: Float|nil }`; falha: `:implausible_vital` com `details: { field: String }` ou `:bp_incomplete`. Total: nunca levanta;
  - `.alerts(values) -> Array<String>` (ordem fixa de `ALERT_RULES`), `.bmi(values) -> Float|nil` (1 casa);
  - `.json(values) -> Hash<String, Integer|Float|String>` (BigDecimal vira Float, para a resposta).

- [ ] **Step 1: Escreva a spec (tabela de casos) que falha**

```ruby
# spec/services/screenings/vital_signs_spec.rb
require "rails_helper"

# ADR 0030 (spec §3.1): limite de plausibilidade recusa; faixa de alerta
# destaca; IMC na leitura. Review Focus 2: entrada da tela nas bordas.
RSpec.describe Screenings::VitalSigns do
  def parse(raw) = described_class.parse(raw)

  describe "valores aceitos" do
    {
      {} => {},
      nil => {},
      { "systolic" => 120, "diastolic" => 80 } => { "systolic" => 120, "diastolic" => 80 },
      { systolic: "120", diastolic: "80" } => { "systolic" => 120, "diastolic" => 80 },
      { "temperature_c" => "37,85" } => { "temperature_c" => BigDecimal("37.9") },
      { "temperature_c" => 36.5 } => { "temperature_c" => BigDecimal("36.5") },
      { "weight_kg" => "72.456" } => { "weight_kg" => BigDecimal("72.46") },
      { "spo2" => "", "heart_rate" => " " } => {},
      { "pain_score" => 0 } => { "pain_score" => 0 },
      { "systolic" => "120.0", "diastolic" => 80 } => { "systolic" => 120, "diastolic" => 80 },
      { "capillary_glucose" => 95, "glucose_moment" => "fasting" } => { "capillary_glucose" => 95, "glucose_moment" => "fasting" },
      { "systolic" => 300, "diastolic" => 200, "heart_rate" => 20, "respiratory_rate" => 80, "spo2" => 50,
        "temperature_c" => 45, "weight_kg" => "0.5", "height_cm" => 250, "capillary_glucose" => 800,
        "glucose_moment" => "random", "pain_score" => 10 } =>
        { "systolic" => 300, "diastolic" => 200, "heart_rate" => 20, "respiratory_rate" => 80, "spo2" => 50,
          "temperature_c" => BigDecimal("45"), "weight_kg" => BigDecimal("0.5"), "height_cm" => 250,
          "capillary_glucose" => 800, "glucose_moment" => "random", "pain_score" => 10 },
      { "bmi" => 99, "outro" => 1 } => {}
    }.each do |raw, values|
      it("#{raw.inspect} → #{values.inspect}") do
        result = parse(raw)
        expect(result).to be_ok
        expect(result.payload[:values]).to eq(values)
      end
    end
  end

  describe "recusas" do
    {
      { "systolic" => 120 } => [ :bp_incomplete, nil ],
      { "diastolic" => 80 } => [ :bp_incomplete, nil ],
      { "systolic" => 301, "diastolic" => 80 } => [ :implausible_vital, "systolic" ],
      { "systolic" => 49, "diastolic" => 30 } => [ :implausible_vital, "systolic" ],
      { "systolic" => 120, "diastolic" => 120 } => [ :implausible_vital, "diastolic" ],
      { "systolic" => "120.5", "diastolic" => 80 } => [ :implausible_vital, "systolic" ],
      { "systolic" => "cento e vinte", "diastolic" => 80 } => [ :implausible_vital, "systolic" ],
      { "systolic" => 0, "diastolic" => 0 } => [ :implausible_vital, "systolic" ],
      { "heart_rate" => 251 } => [ :implausible_vital, "heart_rate" ],
      { "respiratory_rate" => 3 } => [ :implausible_vital, "respiratory_rate" ],
      { "temperature_c" => "29,9" } => [ :implausible_vital, "temperature_c" ],
      { "spo2" => 101 } => [ :implausible_vital, "spo2" ],
      { "capillary_glucose" => 801, "glucose_moment" => "fasting" } => [ :implausible_vital, "capillary_glucose" ],
      { "capillary_glucose" => 120 } => [ :implausible_vital, "glucose_moment" ],
      { "glucose_moment" => "fasting" } => [ :implausible_vital, "capillary_glucose" ],
      { "capillary_glucose" => 120, "glucose_moment" => "noite" } => [ :implausible_vital, "glucose_moment" ],
      { "weight_kg" => "0.4" } => [ :implausible_vital, "weight_kg" ],
      { "height_cm" => 251 } => [ :implausible_vital, "height_cm" ],
      { "pain_score" => -1 } => [ :implausible_vital, "pain_score" ],
      { "spo2" => [ 98 ] } => [ :implausible_vital, "spo2" ],
      "lixo" => [ :implausible_vital, "vitals" ],
      [ 1, 2 ] => [ :implausible_vital, "vitals" ]
    }.each do |raw, (reason, field)|
      it("#{raw.inspect} → #{reason} #{field}") do
        result = parse(raw)
        expect(result).to be_failure
        expect(result.reason).to eq(reason)
        expect(result.details[:field]).to eq(field)
      end
    end
  end

  describe "alertas" do
    {
      { "systolic" => 139, "diastolic" => 89 } => [],
      { "systolic" => 140, "diastolic" => 90 } => %w[systolic_high diastolic_high],
      { "heart_rate" => 101 } => %w[heart_rate_high],
      { "heart_rate" => 49 } => %w[heart_rate_low],
      { "respiratory_rate" => 25 } => %w[respiratory_rate_high],
      { "temperature_c" => "37.7" } => [],
      { "temperature_c" => "37.8" } => %w[temperature_high],
      { "spo2" => 95 } => [],
      { "spo2" => 94 } => %w[spo2_low],
      { "capillary_glucose" => 69, "glucose_moment" => "random" } => %w[glucose_low],
      { "capillary_glucose" => 200, "glucose_moment" => "postprandial" } => %w[glucose_high],
      { "pain_score" => 7 } => %w[pain_severe]
    }.each do |raw, alerts|
      it("#{raw.inspect} → #{alerts.inspect}") { expect(parse(raw).payload[:alerts]).to eq(alerts) }
    end
  end

  it "IMC com peso e altura, nil sem um deles; json devolve número" do
    result = parse("weight_kg" => "80", "height_cm" => 175)
    expect(result.payload[:bmi]).to eq(26.1)
    expect(parse("weight_kg" => 80).payload[:bmi]).to be_nil
    expect(described_class.json(result.payload[:values])).to eq("weight_kg" => 80.0, "height_cm" => 175)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/screenings/vital_signs_spec.rb`
Expected: FAIL com `uninitialized constant Screenings::VitalSigns`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/screenings/vital_signs.rb
# Sinais vitais da escuta (ADR 0030; spec §3.1). Puro e total: aceita o que a
# tela manda (número, texto com vírgula ou ponto, vazio = não medido), recusa
# o implausível dizendo o campo, destaca o que está em faixa de alerta e
# calcula o IMC. Os mesmos limites estão nas CHECKs de screening_revisions.
module Screenings
  module VitalSigns
    FIELDS = %w[systolic diastolic heart_rate respiratory_rate temperature_c spo2 capillary_glucose glucose_moment
                weight_kg height_cm pain_score].freeze
    INTEGER_FIELDS = %w[systolic diastolic heart_rate respiratory_rate spo2 capillary_glucose height_cm pain_score].freeze
    DECIMAL_SCALE = { "temperature_c" => 1, "weight_kg" => 2 }.freeze
    PLAUSIBLE = {
      "systolic" => 50..300, "diastolic" => 20..200, "heart_rate" => 20..250, "respiratory_rate" => 4..80,
      "temperature_c" => 30..45, "spo2" => 50..100, "capillary_glucose" => 10..800,
      "weight_kg" => BigDecimal("0.5")..400, "height_cm" => 30..250, "pain_score" => 0..10
    }.freeze
    GLUCOSE_MOMENTS = %w[fasting postprandial random].freeze
    NUMBER = /\A\d+(?:[.,]\d+)?\z/
    ALERT_RULES = [
      [ "systolic_high", "systolic", ->(v) { v >= 140 } ],
      [ "diastolic_high", "diastolic", ->(v) { v >= 90 } ],
      [ "heart_rate_high", "heart_rate", ->(v) { v > 100 } ],
      [ "heart_rate_low", "heart_rate", ->(v) { v < 50 } ],
      [ "respiratory_rate_high", "respiratory_rate", ->(v) { v > 24 } ],
      [ "temperature_high", "temperature_c", ->(v) { v >= BigDecimal("37.8") } ],
      [ "spo2_low", "spo2", ->(v) { v < 95 } ],
      [ "glucose_low", "capillary_glucose", ->(v) { v < 70 } ],
      [ "glucose_high", "capillary_glucose", ->(v) { v >= 200 } ],
      [ "pain_severe", "pain_score", ->(v) { v >= 7 } ]
    ].freeze

    module_function

    def parse(raw)
      raw = {} if raw.nil?
      raw = raw.to_unsafe_h if raw.respond_to?(:to_unsafe_h)
      return implausible("vitals") unless raw.is_a?(Hash)

      raw = raw.transform_keys(&:to_s)
      values = {}
      FIELDS.each do |field|
        value = raw[field]
        next if value.nil? || (value.is_a?(String) && value.strip.empty?)

        parsed = field == "glucose_moment" ? moment(value) : measure(field, value)
        return implausible(field) if parsed.nil?

        values[field] = parsed
      end
      return Result.fail(:bp_incomplete) if values.key?("systolic") != values.key?("diastolic")
      return implausible("diastolic") if values.key?("systolic") && values["diastolic"] >= values["systolic"]
      return implausible("glucose_moment") if values.key?("capillary_glucose") && !values.key?("glucose_moment")
      return implausible("capillary_glucose") if values.key?("glucose_moment") && !values.key?("capillary_glucose")

      Result.ok(values: values, alerts: alerts(values), bmi: bmi(values))
    end

    def alerts(values)
      ALERT_RULES.filter_map { |name, field, rule| name if values[field] && rule.call(values[field]) }
    end

    def bmi(values)
      weight = values["weight_kg"]
      height = values["height_cm"]
      return nil unless weight && height

      (weight.to_f / ((height.to_f / 100)**2)).round(1)
    end

    def json(values) = values.transform_values { |v| v.is_a?(BigDecimal) ? v.to_f : v }

    def implausible(field) = Result.fail(:implausible_vital, details: { field: field })

    def moment(value) = GLUCOSE_MOMENTS.include?(value.to_s) ? value.to_s : nil

    def measure(field, value)
      number = to_decimal(value)
      return nil if number.nil?

      number = if INTEGER_FIELDS.include?(field)
                 number.frac.zero? ? number.to_i : nil
               else
                 number.round(DECIMAL_SCALE.fetch(field), BigDecimal::ROUND_HALF_UP)
               end
      number && PLAUSIBLE.fetch(field).cover?(number) ? number : nil
    end

    def to_decimal(value)
      case value
      when Integer then BigDecimal(value)
      when Float, BigDecimal then BigDecimal(value.to_s)
      when String then value.strip.match?(NUMBER) ? BigDecimal(value.strip.tr(",", ".")) : nil
      end
    end
    private_class_method :implausible, :moment, :measure, :to_decimal
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/screenings/vital_signs_spec.rb`
Expected: PASS. (Se `"pain_score" => -1` passar como aceito, o `NUMBER` ganhou sinal: ele não aceita `-`, e `Integer` negativo cai fora de `PLAUSIBLE`.)

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add app/services/screenings/vital_signs.rb spec/services/screenings/vital_signs_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: validate screening vital signs with plausibility, alerts and BMI

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Cor sugerida — `Screenings::RiskSuggestion` (puro), protocolo ativo, CIAP-2 e `Screenings::Suggest`

**Files:**
- Create: `app/services/screenings/risk_suggestion.rb`, `app/services/screenings/active_protocol.rb`, `app/services/screenings/ciap2.rb`, `app/services/screenings/suggest.rb`
- Test: `spec/services/screenings/risk_suggestion_spec.rb`, `spec/services/screenings/suggest_spec.rb`, `spec/services/screenings/ciap2_spec.rb`

**Interfaces:**
- Consumes: `Protocols::ConditionContext.build(vitals:, complaint:, profile:)` e `Protocols::Condition.eval` (Task 2), `Screenings::VitalSigns` (Task 6).
- Produces:
  - `Screenings::RiskSuggestion::COLORS == %w[red yellow green blue]`, `.call(revision, profile, rules:) -> { color: String|nil, matched: Array<Integer> }` (puro; `revision` = `{ vitals: Hash, bmi: Float|nil, ciap2_code: String|nil }`; `matched` = índices de **todas** as regras que casam, em ordem; `color` = a mais grave entre elas);
  - `Screenings::ActiveProtocol.current -> ProtocolDefinition|nil` (versão `active` de `acolhimento` com `kind: "screening"`);
  - `Screenings::Ciap2::Code = Data(:code, :label, :release_id)`, `.find(code) -> Code|nil` (release ativa), `.label(code, release_id) -> String|nil`, `.search(query, limit: 20) -> Array<Code>`;
  - `Screenings::Suggest.call(citizen:, ciap2_code:, vitals:, bmi:, on: Time.zone.today) -> { color:, matched:, protocol_definition_id: }` (sem protocolo ativo: `{ color: nil, matched: [], protocol_definition_id: nil }`).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/screenings/risk_suggestion_spec.rb
require "rails_helper"

# ADR 0030 (spec §3.3): vale a mais grave que casar (red > yellow > green >
# blue); regra com sinal não medido não casa (Review Focus 3).
RSpec.describe Screenings::RiskSuggestion do
  let(:rules) do
    [ { "when" => { "any" => [ { "gte" => ["vitals.systolic", 180] }, { "lt" => ["vitals.spo2", 90] } ] }, "color" => "red" },
      { "when" => { "gte" => ["vitals.temperature_c", 39] }, "color" => "yellow" },
      { "when" => { "eq" => ["complaint.ciap2", "R05"] }, "color" => "green" },
      { "when" => { "all" => [ { "gte" => ["profile.age", 60] }, { "gte" => ["vitals.bmi", 35] } ] }, "color" => "yellow" },
      { "when" => { "eq" => ["complaint.ciap2", "A97"] }, "color" => "blue" } ]
  end

  def suggest(vitals: {}, bmi: nil, ciap2: nil, age: 40, sex: "female", list: rules)
    described_class.call({ vitals: vitals, bmi: bmi, ciap2_code: ciap2 }, { age: age, sex: sex }, rules: list)
  end

  {
    "PA 185/110 → red" => [ { vitals: { "systolic" => 185, "diastolic" => 110 } }, "red", [ 0 ] ],
    "SpO2 88 com febre 39,5 → red (a mais grave)" =>
      [ { vitals: { "spo2" => 88, "temperature_c" => BigDecimal("39.5") } }, "red", [ 0, 1 ] ],
    "febre 39 e tosse → yellow" => [ { vitals: { "temperature_c" => BigDecimal("39") }, ciap2: "R05" }, "yellow", [ 1, 2 ] ],
    "tosse sem sinais → green" => [ { ciap2: "R05" }, "green", [ 2 ] ],
    "idosa com IMC 36 → yellow" => [ { bmi: 36.0, age: 61 }, "yellow", [ 3 ] ],
    "A97 → blue" => [ { ciap2: "A97" }, "blue", [ 4 ] ],
    "nada casa → sem sugestão" => [ { vitals: { "systolic" => 120, "diastolic" => 80 }, ciap2: "K86" }, nil, [] ],
    "SpO2 não medida não casa lt" => [ { vitals: {} }, nil, [] ],
    "sem regras → sem sugestão" => [ { ciap2: "R05", list: [] }, nil, [] ]
  }.each do |label, (input, color, matched)|
    it(label) { expect(suggest(**input)).to eq(color: color, matched: matched) }
  end

  it "regra malformada não casa nem levanta" do
    expect(suggest(ciap2: "R05", list: [ "x", { "when" => nil, "color" => "red" }, { "color" => "red" } ]))
      .to eq(color: nil, matched: [])
  end
end
```

```ruby
# spec/services/screenings/ciap2_spec.rb
require "rails_helper"

RSpec.describe Screenings::Ciap2 do
  before { ciap2_release! }

  it "acha o código da release ativa, com rótulo e release; desconhecido é nil" do
    code = described_class.find("k86")
    expect([ code.code, code.label ]).to eq([ "K86", "Hipertensão sem complicações" ])
    expect(code.release_id).to eq(TerminologyRelease.active.find_by!(kind: "ciap2").id)
    expect(described_class.find("Z99")).to be_nil
    expect(described_class.find(nil)).to be_nil
    expect(described_class.label("K86", code.release_id)).to eq("Hipertensão sem complicações")
  end

  it "busca por código ou por nome, sem acento e sem caixa, até o limite" do
    expect(described_class.search("tos").map(&:code)).to eq([ "R05" ])
    expect(described_class.search("hipertensao").map(&:code)).to eq([ "K86" ])
    expect(described_class.search("k8").map(&:code)).to eq([ "K86" ])
    expect(described_class.search("").map(&:code)).to eq([])
    expect(described_class.search("e", limit: 2).size).to eq(2)
  end
end
```

```ruby
# spec/services/screenings/suggest_spec.rb
require "rails_helper"

# Sugestão com o protocolo ativo de acolhimento e o perfil do par.
RSpec.describe Screenings::Suggest do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:citizen) { screening_citizen!(1, age: 61) }

  it "sem protocolo ativo: sem sugestão (Review Focus 3)" do
    expect(described_class.call(citizen: citizen, ciap2_code: "R05", vitals: {}, bmi: nil))
      .to eq(color: nil, matched: [], protocol_definition_id: nil)
  end

  it "com o acolhimento ativo, sugere e diz qual versão decidiu" do
    protocol = acolhimento!
    result = described_class.call(citizen: citizen, ciap2_code: "K86", vitals: { "systolic" => 185, "diastolic" => 110 }, bmi: nil)
    expect(result).to eq(color: "red", matched: [ 0 ], protocol_definition_id: protocol.id)
  end

  it "um protocolo de triagem ativo nunca é usado para cor" do
    active_protocol!("saude-do-idoso")
    expect(Screenings::ActiveProtocol.current).to be_nil
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/screenings/risk_suggestion_spec.rb spec/services/screenings/ciap2_spec.rb spec/services/screenings/suggest_spec.rb`
Expected: FAIL (`uninitialized constant Screenings::RiskSuggestion`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/screenings/risk_suggestion.rb
# Cor sugerida (ADR 0030; spec §3.3). Puro: recebe a revisão (sinais, IMC,
# CIAP-2), o perfil e as regras do protocolo assinado; devolve a cor mais
# grave entre as regras que casam e os índices de todas elas. Sinal ausente
# não entra no contexto, então a condição sobre ele é falsa.
module Screenings
  module RiskSuggestion
    COLORS = %w[red yellow green blue].freeze

    module_function

    def call(revision, profile, rules:)
      context = Protocols::ConditionContext.build(
        vitals: (revision[:vitals] || {}).merge("bmi" => revision[:bmi]),
        complaint: { ciap2: revision[:ciap2_code] }, profile: profile || {}
      )
      matched = Array(rules).each_with_index.filter_map do |rule, index|
        next unless rule.is_a?(Hash) && COLORS.include?(rule["color"])

        index if Protocols::Condition.eval(rule["when"], context)
      end
      color = matched.map { |index| rules[index]["color"] }.min_by { |c| COLORS.index(c) }
      { color: color, matched: matched }
    end
  end
end
```

```ruby
# app/services/screenings/active_protocol.rb
# O protocolo de acolhimento em vigor na cidade (ADR 0030): a versão active
# do nome reservado, da variante screening. Sem ele, não há sugestão.
module Screenings
  module ActiveProtocol
    module_function

    def current
      ProtocolDefinition.screening_protocols.find_by(name: Protocols::Validation::Screening::NAME, status: "active")
    end
  end
end
```

```ruby
# app/services/screenings/ciap2.rb
# CIAP-2 da plataforma (ADR 0028) para a queixa da escuta: busca na release
# ativa; a revisão guarda o código e a release (a leitura usa a release
# gravada, mesmo depois de outra ser ativada).
module Screenings
  module Ciap2
    Code = Data.define(:code, :label, :release_id)

    module_function

    def release = TerminologyRelease.active.find_by(kind: "ciap2")

    def find(code)
      current = release
      return nil if current.nil? || code.blank?

      row = Ciap2Code.find_by(release_id: current.id, code: code.to_s.strip.upcase)
      row && Code.new(code: row.code, label: row.description, release_id: current.id)
    end

    def label(code, release_id) = Ciap2Code.find_by(release_id: release_id, code: code)&.description

    def search(query, limit: 20)
      current = release
      text = query.to_s.strip
      return [] if current.nil? || text.empty?

      folded = I18n.transliterate(text).downcase
      Ciap2Code.where(release_id: current.id).order(:code).select do |row|
        row.code.downcase.start_with?(folded) || I18n.transliterate(row.description).downcase.include?(folded)
      end.first(limit).map { |row| Code.new(code: row.code, label: row.description, release_id: current.id) }
    end
  end
end
```

(A CIAP-2 tem ~700 códigos: filtrar em Ruby é barato e acerta acento sem extensão `unaccent`.)

```ruby
# app/services/screenings/suggest.rb
# Cor sugerida com o protocolo ativo e o perfil do par (ADR 0030). O
# servidor sempre recalcula: a cor sugerida gravada nunca vem do cliente.
module Screenings
  module Suggest
    NONE = { color: nil, matched: [], protocol_definition_id: nil }.freeze

    module_function

    def call(citizen:, ciap2_code:, vitals:, bmi:, on: Time.zone.today)
      protocol = ActiveProtocol.current
      return NONE.dup unless protocol

      result = RiskSuggestion.call({ vitals: vitals, bmi: bmi, ciap2_code: ciap2_code },
                                   citizen.profile_context(on: on), rules: protocol.definition["risk_rules"])
      result.merge(protocol_definition_id: protocol.id)
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/screenings`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add app/services/screenings/risk_suggestion.rb app/services/screenings/active_protocol.rb app/services/screenings/ciap2.rb app/services/screenings/suggest.rb spec/services/screenings/risk_suggestion_spec.rb spec/services/screenings/ciap2_spec.rb spec/services/screenings/suggest_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: suggest the CAB 28 risk colour from the signed screening protocol

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 8: Quem faz a escuta, iniciar e abandonar (e a chamada/saída que abandonam)

**Files:**
- Create: `app/services/screenings/cbos.rb`, `app/services/screenings/scope.rb`, `app/services/screenings/authorization.rb`
- Create: `app/commands/screenings/start.rb`, `app/commands/screenings/abandon.rb`
- Modify: `app/commands/attendances/call.rb`, `app/commands/attendances/close.rb`
- Test: `spec/services/screenings/authorization_spec.rb`, `spec/commands/screenings/start_spec.rb`, `spec/commands/screenings/abandon_spec.rb`

**Interfaces:**
- Consumes: `Ledi::ScreeningMapping.exportable_cbo?` e `.miai_cbo?` (Task 1); `Screening` (Task 3).
- Produces:
  - `Screenings::Cbos.prefixes -> Array<String>`, `.allowed?(cbo) -> bool`;
  - `Screenings::Scope.required?(attendance) -> bool` (`all`, ou `walk_in` e sem horário);
  - `Screenings::Authorization.check(user:, health_unit_id:) -> [Symbol, ProfessionalLink|nil]` — `[:ok, link]`, `[:missing_role, nil]`, `[:missing_link, nil]`, `[:cbo_not_allowed, nil]`; chame **dentro** da transação (FOR SHARE no vínculo); prefere vínculo de nível superior (MIAI) quando há mais de um;
  - `Screenings::Start.call(attendance:, by:) -> Result` (ok `{ screening: }`; falhas `:missing_role`, `:missing_link`, `:cbo_not_allowed`, `:not_waiting`, `:screening_not_required`, `:already_screening`);
  - `Screenings::Abandon.call(screening:, by:) -> Result` (ok `{ screening: }`; falhas de autorização e `:not_in_progress`); `Screenings::Abandon.release!(attendance, by:) -> Integer` (dentro da transação de quem já travou o atendimento; abandona a escuta em curso, se houver).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/screenings/authorization_spec.rb
require "rails_helper"

# ADR 0030 (spec §3.2; Desvio 2): papel, vínculo ativo na unidade e CBO dos
# grupos da escuta com ficha possível.
RSpec.describe Screenings::Authorization do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }

  def check(user) = ApplicationRecord.transaction { described_class.check(user: user, health_unit_id: unit.id) }

  {
    "223505" => :ok, "225125" => :ok, "322205" => :ok, "322245" => :ok, "251605" => :ok,
    "223293" => :cbo_not_allowed, "322405" => :cbo_not_allowed, "221205" => :cbo_not_allowed
  }.each do |cbo, expected|
    it("CBO #{cbo} → #{expected}") { expect(check(screener!(unit, cbo: cbo)).first).to eq(expected) }
  end

  it "sem papel, sem vínculo na unidade, vínculo encerrado" do
    expect(check(staff_with("recepcao@cidade.gov.br", "citizen_verifier"))).to eq([ :missing_role, nil ])
    elsewhere = screener!(create_unit("UBS Outra"))
    expect(check(elsewhere)).to eq([ :missing_link, nil ])
    ended = screener!(unit)
    Professionals::EndLink.call(link: ended.professional.links.active.sole,
                                by: staff_with("adm-#{SecureRandom.hex(3)}@cidade.gov.br", "municipal_admin"))
    expect(check(ended)).to eq([ :missing_link, nil ])
  end

  it "com dois vínculos na unidade, prefere o de nível superior" do
    user = screener!(unit, cbo: "322205")
    link_professional!(user, unit, cbo: "223505")
    status, link = check(user)
    expect([ status, link.cbo_code ]).to eq([ :ok, "223505" ])
  end
end
```

```ruby
# spec/commands/screenings/start_spec.rb
require "rails_helper"

# ADR 0030 (spec §3.2): atendimento waiting, escopo da unidade, vínculo e CBO.
RSpec.describe Screenings::Start do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  let(:attendance) { walk_in_attendance!(unit, citizen: screening_citizen!(1)) }

  it "inicia: em curso, autor, vínculo e CBO; evento só com ids" do
    result = described_class.call(attendance: attendance, by: nurse)
    expect(result).to be_ok
    screening = result.payload[:screening]
    link = nurse.professional.links.active.sole
    expect(screening).to have_attributes(status: "in_progress", started_by_user_id: nurse.id,
                                         professional_link_id: link.id, cbo_code: "223505", attendance_id: attendance.id)
    expect(DomainEvent.where(name: "screening.started").sole.payload)
      .to eq("screening_id" => screening.id, "attendance_id" => attendance.id, "user_id" => nurse.id)
  end

  it "já em curso ou concluída → already_screening; abandonada é retomada por outra" do
    first = described_class.call(attendance: attendance, by: nurse).payload[:screening]
    expect(described_class.call(attendance: attendance, by: nurse).reason).to eq(:already_screening)
    Screenings::Abandon.call(screening: first, by: nurse)
    tech = screener!(unit, cbo: "322205")
    resumed = described_class.call(attendance: attendance, by: tech)
    expect(resumed).to be_ok
    expect(resumed.payload[:screening].id).to eq(first.id)
    expect(first.reload).to have_attributes(status: "in_progress", started_by_user_id: tech.id, cbo_code: "322205")
  end

  it "escopo: walk_in não exige escuta de quem tem horário; all exige" do
    scheduled = scheduled_attendance!(unit, citizen: screening_citizen!(2))
    expect(described_class.call(attendance: scheduled, by: nurse).reason).to eq(:screening_not_required)
    unit.update!(screening_scope: "all")
    expect(described_class.call(attendance: scheduled.reload, by: nurse)).to be_ok
  end

  it "atendimento que não está aguardando; autorização" do
    attendance.update!(status: "in_care", called_by_user: nurse, called_at: Time.current)
    expect(described_class.call(attendance: attendance, by: nurse).reason).to eq(:not_waiting)
    other = walk_in_attendance!(unit, citizen: screening_citizen!(3))
    expect(described_class.call(attendance: other, by: reception!).reason).to eq(:missing_role)
    expect(described_class.call(attendance: other, by: screener!(create_unit("UBS Outra"))).reason).to eq(:missing_link)
    expect(described_class.call(attendance: other, by: screener!(unit, cbo: "322405")).reason).to eq(:cbo_not_allowed)
  end
end
```

```ruby
# spec/commands/screenings/abandon_spec.rb
require "rails_helper"

# ADR 0030 (spec §3.2): abandonar devolve a pessoa à fila do acolhimento. A
# chamada e a saída sem atendimento abandonam a escuta em curso (Desvio 7;
# Review Focus 4).
RSpec.describe Screenings::Abandon do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  let(:attendance) { walk_in_attendance!(unit, citizen: screening_citizen!(1)) }
  let(:screening) { Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening] }

  it "abandona a escuta em curso; de novo → not_in_progress" do
    expect(described_class.call(screening: screening, by: nurse)).to be_ok
    expect(screening.reload.status).to eq("abandoned")
    expect(DomainEvent.where(name: "screening.abandoned").sole.payload)
      .to eq("screening_id" => screening.id, "attendance_id" => attendance.id, "user_id" => nurse.id)
    expect(described_class.call(screening: screening, by: nurse).reason).to eq(:not_in_progress)
  end

  it "a chamada do profissional abandona a escuta em curso" do
    doctor = screener!(unit, cbo: "225125")
    screening
    expect(Attendances::Call.call(attendance: attendance, health_unit_id: unit.id, by: doctor)).to be_ok
    expect(screening.reload.status).to eq("abandoned")
  end

  it "left abandona a escuta em curso (Review Focus 4)" do
    screening
    result = Attendances::Close.call(attendance: attendance, outcome: "left", referral_unit_id: nil, referral_note: nil,
                                     by: reception!)
    expect(result).to be_ok
    expect(screening.reload.status).to eq("abandoned")
  end

  it "a rota de desfecho não aceita os desfechos de escuta" do
    attendance.update!(status: "in_care", called_by_user: nurse, called_at: Time.current)
    %w[oriented scheduled_from_screening].each do |outcome|
      expect(Attendances::Close.call(attendance: attendance, outcome: outcome, referral_unit_id: nil, referral_note: nil,
                                     by: nurse).reason).to eq(:invalid_outcome)
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/screenings/authorization_spec.rb spec/commands/screenings`
Expected: FAIL (`uninitialized constant Screenings::Authorization`, `Screenings::Start`).

- [ ] **Step 3: Implemente os serviços**

```ruby
# app/services/screenings/cbos.rb
# Quem pode fazer a escuta (ADR 0030; spec §3.2; Desvio 2): CBO dos grupos de
# config/scheduling/screening_cbos.yml E com ficha LEDI possível (tabela do
# MIAI ou técnico/auxiliar de enfermagem no MIP).
module Screenings
  module Cbos
    PATH = Rails.root.join("config/scheduling/screening_cbos.yml")

    module_function

    def prefixes = (@prefixes ||= YAML.load_file(PATH).map { |row| row.fetch("prefix").to_s }.freeze)

    def allowed?(cbo)
      cbo = cbo.to_s
      prefixes.any? { |prefix| cbo.start_with?(prefix) } && Ledi::ScreeningMapping.exportable_cbo?(cbo)
    end
  end
end
```

```ruby
# app/services/screenings/scope.rb
# A unidade escolhe para quem a escuta é obrigatória (ADR 0030; spec §3.1):
# walk_in (padrão) = só quem chegou sem horário marcado; all = todos.
module Screenings
  module Scope
    module_function

    def required?(attendance)
      attendance.health_unit.screening_scope == "all" || attendance.appointment_id.nil?
    end
  end
end
```

```ruby
# app/services/screenings/authorization.rb
# Regra da escuta (ADR 0030; spec §3.2): papel health_professional, vínculo
# ativo com a unidade do atendimento e CBO permitido. Chame DENTRO da
# transação do comando: o FOR SHARE no vínculo faz o encerramento esperar,
# como Professionals::ClinicalAuthorization.
module Screenings
  module Authorization
    module_function

    def check(user:, health_unit_id:)
      return [ :missing_role, nil ] unless user&.has_role?("health_professional")

      links = ProfessionalLink.active.joins(:professional)
                              .where(professionals: { user_id: user.id }, health_unit_id: health_unit_id.to_s)
                              .lock("FOR SHARE OF professional_links").order(:started_at, :id).to_a
      return [ :missing_link, nil ] if links.empty?

      allowed = links.select { |link| Cbos.allowed?(link.cbo_code) }
      return [ :cbo_not_allowed, nil ] if allowed.empty?

      [ :ok, allowed.find { |link| Ledi::ScreeningMapping.miai_cbo?(link.cbo_code) } || allowed.first ]
    end
  end
end
```

- [ ] **Step 4: Os comandos**

```ruby
# app/commands/screenings/start.rb
# Iniciar a escuta (ADR 0030; spec §3.2). Trava o atendimento: duas
# profissionais ao mesmo tempo, uma inicia e a outra recebe already_screening.
# Escuta abandonada do mesmo atendimento é retomada (uma por atendimento).
module Screenings
  class Start
    def self.call(attendance:, by:)
      ApplicationRecord.transaction do
        attendance.lock!
        status, link = Authorization.check(user: by, health_unit_id: attendance.health_unit_id)
        next Result.fail(status) unless status == :ok
        next Result.fail(:not_waiting) unless attendance.status == "waiting"
        next Result.fail(:screening_not_required) unless Scope.required?(attendance)

        screening = Screening.lock.find_by(attendance_id: attendance.id)
        next Result.fail(:already_screening) if screening && screening.status != "abandoned"

        attrs = { status: "in_progress", started_by_user: by, professional_link: link, cbo_code: link.cbo_code,
                  started_at: Time.current }
        screening ? screening.update!(attrs) : (screening = Screening.create!(attrs.merge(attendance: attendance)))
        DomainEvents.publish("screening.started", screening_id: screening.id, attendance_id: attendance.id, user_id: by.id)
        Result.ok(screening: screening)
      end
    end
  end
end
```

```ruby
# app/commands/screenings/abandon.rb
# Abandonar a escuta (ADR 0030; spec §3.2): a pessoa volta à fila do
# acolhimento. release! é o mesmo efeito por dentro de outro comando que já
# travou o atendimento (chamada, saída): ordem de travas atendimento → escuta.
module Screenings
  class Abandon
    def self.call(screening:, by:)
      ApplicationRecord.transaction do
        attendance = Attendance.lock.find(screening.attendance_id)
        screening.lock!
        status, _link = Authorization.check(user: by, health_unit_id: attendance.health_unit_id)
        next Result.fail(status) unless status == :ok
        next Result.fail(:not_in_progress) unless screening.status == "in_progress"

        abandon!(screening, by)
        Result.ok(screening: screening)
      end
    end

    def self.release!(attendance, by:)
      Screening.where(attendance_id: attendance.id, status: "in_progress").lock.each { |s| abandon!(s, by) }.size
    end

    def self.abandon!(screening, by)
      screening.update!(status: "abandoned")
      DomainEvents.publish("screening.abandoned", screening_id: screening.id, attendance_id: screening.attendance_id,
                                                  user_id: by.id)
    end
    private_class_method :abandon!
  end
end
```

Em `app/commands/attendances/call.rb`, logo antes de `attendance.update!(status: "in_care", …)`:

```ruby
        # ADR 0030 (Desvio 7): chamar no meio da escuta encerra a escuta.
        Screenings::Abandon.release!(attendance, by: by)
```

Em `app/commands/attendances/close.rb`, logo antes de `attendance.update!(status: "closed", …)`:

```ruby
        # ADR 0030 (Desvio 7): quem sai (ou é encerrado) com escuta em curso
        # não deixa a escuta pendurada.
        Screenings::Abandon.release!(attendance, by: by)
```

- [ ] **Step 5: Rode e veja passar (e o que já existia do atendimento)**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/screenings spec/commands/screenings spec/commands/attendances`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add app/services/screenings/cbos.rb app/services/screenings/scope.rb app/services/screenings/authorization.rb app/commands/screenings/start.rb app/commands/screenings/abandon.rb app/commands/attendances/call.rb app/commands/attendances/close.rb spec/services/screenings/authorization_spec.rb spec/commands/screenings/start_spec.rb spec/commands/screenings/abandon_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: start and abandon the initial listening with link and CBO checks

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 9: Concluir (destino e fechamento) e reavaliar

**Files:**
- Create: `app/services/screenings/revision_input.rb`
- Create: `app/commands/screenings/complete.rb`, `app/commands/screenings/reassess.rb`
- Test: `spec/services/screenings/revision_input_spec.rb`, `spec/commands/screenings/complete_spec.rb`, `spec/commands/screenings/reassess_spec.rb`

**Interfaces:**
- Consumes: `Screenings::VitalSigns.parse`, `Screenings::Ciap2.find`, `Screenings::Suggest.call`, `Screenings::Authorization.check`, `AppointmentRequests::Lifecycle.open_for!`, `HealthUnit.lock_active!`, `AppointmentType.active_types`.
- Produces:
  - `Screenings::RevisionInput::MAX_NOTE == 500`, `::MIN_REASON == 10`, `.call(params, citizen:) -> Result` — ok `payload { attrs: Hash (colunas de ScreeningRevision, sem screening/by_user), alerts:, bmi: }`; falhas `:invalid_ciap2`, `:implausible_vital` (`field`), `:bp_incomplete`, `:invalid_color`, `:color_change_reason_required`, `:note_too_long` (`field`);
  - `Screenings::Complete::DEFAULT_DUE_IN_DAYS == { "yellow" => 7, "green" => 15, "blue" => 30 }`, `.call(screening:, revision_params:, destination:, destination_params:, by:) -> Result` (ok `{ screening:, attendance:, appointment_request: }`; falhas da revisão, `:invalid_destination`, `:orientation_required`, `:invalid_schedule`, `:referral_required`, `:invalid_unit`, `:note_too_long`, autorização, `:not_in_progress`, `:attendance_not_waiting`); `destination_params` = `{ "orientation_note", "schedule" => { "appointment_type_key", "priority", "due_in_days" }, "referral" => { "referral_unit_id", "referral_note" } }`;
  - `Screenings::Reassess.call(screening:, revision_params:, by:) -> Result` (ok `{ screening:, revision: }`; falhas da revisão, autorização, `:not_reassessable`).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/screenings/revision_input_spec.rb
require "rails_helper"

# ADR 0030 (spec §3.1, §3.3): queixa CIAP-2 obrigatória, sinais, cor final e
# justificativa quando a cor final difere da sugerida (recalculada aqui).
RSpec.describe Screenings::RevisionInput do
  before { Current.city = TEST_CITY_A; ciap2_release! }
  after { Current.reset }

  let(:citizen) { screening_citizen!(1) }

  def input(**over) = described_class.call(revision_params(**over), citizen: citizen)

  it "monta as colunas da revisão com a sugestão do servidor" do
    protocol = acolhimento!
    result = input(vitals: { "systolic" => "185", "diastolic" => "110" }, final_color: "red", complaint_note: "  cefaleia forte  ")
    expect(result).to be_ok
    expect(result.payload[:attrs]).to include(
      "ciap2_code" => "K86", "ciap2_release_id" => TerminologyRelease.active.find_by!(kind: "ciap2").id,
      "complaint_note" => "cefaleia forte", "systolic" => 185, "diastolic" => 110, "suggested_color" => "red",
      "final_color" => "red", "color_change_reason" => nil, "rule_protocol_definition_id" => protocol.id, "matched_rules" => [ 0 ]
    )
    expect(result.payload[:alerts]).to eq(%w[systolic_high diastolic_high])
  end

  it "cor diferente da sugerida exige justificativa de 10+; sem sugestão a cor é livre e a justificativa some" do
    acolhimento!
    red = { "systolic" => 185, "diastolic" => 110 }
    expect(input(vitals: red, final_color: "yellow").reason).to eq(:color_change_reason_required)
    expect(input(vitals: red, final_color: "yellow", color_change_reason: "curta").reason).to eq(:color_change_reason_required)
    ok = input(vitals: red, final_color: "yellow", color_change_reason: "PA medida após esforço, repetida 150/95")
    expect(ok.payload[:attrs]["color_change_reason"]).to eq("PA medida após esforço, repetida 150/95")
    free = input(vitals: { "systolic" => 120, "diastolic" => 80 }, final_color: "blue", color_change_reason: "qualquer coisa aqui")
    expect(free.payload[:attrs].values_at("suggested_color", "final_color", "color_change_reason")).to eq([ nil, "blue", nil ])
  end

  {
    { ciap2_code: "Z99" } => [ :invalid_ciap2, nil ],
    { ciap2_code: nil } => [ :invalid_ciap2, nil ],
    { vitals: { "systolic" => 120 } } => [ :bp_incomplete, nil ],
    { vitals: { "spo2" => 30 } } => [ :implausible_vital, "spo2" ],
    { final_color: "orange" } => [ :invalid_color, nil ],
    { final_color: nil } => [ :invalid_color, nil ],
    { complaint_note: "x" * 501 } => [ :note_too_long, "complaint_note" ]
  }.each do |over, (reason, field)|
    it("#{over.inspect} → #{reason}") do
      result = input(**over)
      expect([ result.reason, result.details[:field] ]).to eq([ reason, field ])
    end
  end
end
```

```ruby
# spec/commands/screenings/complete_spec.rb
require "rails_helper"

# ADR 0030 (spec §4): cada destino e o desfecho que fecha o atendimento.
RSpec.describe Screenings::Complete do
  before { Current.city = TEST_CITY_A; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  let(:attendance) { walk_in_attendance!(unit, citizen: screening_citizen!(1)) }
  let(:screening) { Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening] }

  def complete(destination, params = {}, **revision)
    described_class.call(screening: screening, revision_params: revision_params(**revision), destination: destination,
                         destination_params: params, by: nurse)
  end

  it "same_day: escuta concluída, atendimento segue aguardando; evento só com ids" do
    result = complete("same_day", final_color: "yellow")
    expect(result).to be_ok
    screening.reload
    expect(screening).to have_attributes(status: "completed", destination: "same_day")
    expect(screening.current_revision).to have_attributes(final_color: "yellow", by_user_id: nurse.id, ciap2_code: "K86")
    expect(attendance.reload.status).to eq("waiting")
    expect(DomainEvent.where(name: "screening.completed").sole.payload)
      .to eq("screening_id" => screening.id, "attendance_id" => attendance.id, "destination" => "same_day",
             "final_color" => "yellow")
  end

  it "schedule: pedido do módulo 17 com origem na escuta, prazo pela cor; atendimento fecha scheduled_from_screening" do
    type = appointment_type!
    result = complete("schedule", { "schedule" => { "appointment_type_key" => type.key, "priority" => "routine" } },
                      final_color: "yellow")
    expect(result).to be_ok
    request = result.payload[:appointment_request]
    expect(request).to have_attributes(kind: "screening", origin_screening_id: screening.id,
                                       origin_attendance_id: attendance.id, target_unit_id: unit.id,
                                       appointment_type_key: type.key, priority: "routine",
                                       due_on: Time.zone.today + 7, status: "open")
    expect(screening.reload.appointment_request_id).to eq(request.id)
    expect(attendance.reload).to have_attributes(status: "closed", outcome: "scheduled_from_screening",
                                                 closed_by_user_id: nurse.id)
    expect(DomainEvent.where(name: "appointment_request.created").sole.payload).to include("kind" => "screening")
  end

  it "schedule: prazo dado vence o padrão; red sem prazo, tipo inexistente ou prioridade inválida → invalid_schedule" do
    type = appointment_type!
    expect(complete("schedule", { "schedule" => { "appointment_type_key" => type.key, "priority" => "routine" } },
                    final_color: "red").reason).to eq(:invalid_schedule)
    expect(complete("schedule", { "schedule" => { "appointment_type_key" => "fantasma", "priority" => "routine" } },
                    final_color: "green").reason).to eq(:invalid_schedule)
    expect(complete("schedule", { "schedule" => { "appointment_type_key" => type.key, "priority" => "urgente" } },
                    final_color: "green").reason).to eq(:invalid_schedule)
    expect(complete("schedule", { "schedule" => { "appointment_type_key" => type.key, "priority" => "priority",
                                                  "due_in_days" => 3 } }, final_color: "red")).to be_ok
    expect(screening.reload.appointment_request.due_on).to eq(Time.zone.today + 3)
  end

  it "oriented exige orientação (até 500) e fecha oriented" do
    expect(complete("oriented").reason).to eq(:orientation_required)
    expect(complete("oriented", { "orientation_note" => "x" * 501 }).reason).to eq(:note_too_long)
    expect(complete("oriented", { "orientation_note" => "Hidratação, retorno se piorar" })).to be_ok
    expect(attendance.reload.outcome).to eq("oriented")
    expect(screening.reload.orientation_note).to eq("Hidratação, retorno se piorar")
  end

  it "referred: unidade gera pedido como hoje; só descrição fecha sem pedido; nada → referral_required" do
    expect(complete("referred").reason).to eq(:referral_required)
    expect(complete("referred", { "referral" => { "referral_unit_id" => "nao-e-uuid" } }).reason).to eq(:invalid_unit)
    upa = create_unit("UPA Norte", kind: "upa")
    upa.update!(active: false)
    expect(complete("referred", { "referral" => { "referral_unit_id" => upa.id } }).reason).to eq(:invalid_unit)
    upa.update!(active: true)
    result = complete("referred", { "referral" => { "referral_unit_id" => upa.id } })
    expect(result).to be_ok
    expect(attendance.reload).to have_attributes(outcome: "referred", referral_unit_id: upa.id)
    expect(result.payload[:appointment_request]).to have_attributes(kind: "referral", target_unit_id: upa.id)
  end

  it "referred só com descrição fecha sem pedido" do
    result = complete("referred", { "referral" => { "referral_note" => "CAPS" } })
    expect(result).to be_ok
    expect(result.payload[:appointment_request]).to be_nil
    expect(attendance.reload).to have_attributes(outcome: "referred", referral_note: "CAPS")
  end

  it "destino inválido, escuta que não está em curso e atendimento que não aguarda" do
    expect(complete("agora").reason).to eq(:invalid_destination)
    Screenings::Abandon.call(screening: screening, by: nurse)
    expect(complete("same_day").reason).to eq(:not_in_progress)
  end

  it "atendimento chamado por fora (sem passar pelo Call) → attendance_not_waiting" do
    screening
    attendance.update!(status: "in_care", called_by_user: nurse, called_at: Time.current)
    expect(complete("same_day").reason).to eq(:attendance_not_waiting)
  end

  it "conclui com o vínculo e CBO de quem conclui" do
    screening
    tech = screener!(unit, cbo: "322205")
    result = described_class.call(screening: screening, revision_params: revision_params, destination: "same_day",
                                  destination_params: {}, by: tech)
    expect(result).to be_ok
    expect(screening.reload).to have_attributes(cbo_code: "322205", started_by_user_id: nurse.id)
  end
end
```

```ruby
# spec/commands/screenings/reassess_spec.rb
require "rails_helper"

# ADR 0030 (spec §3.2, §4): reavaliar enquanto a pessoa espera, com destino
# same_day; cada reavaliação é uma revisão nova.
RSpec.describe Screenings::Reassess do
  before { Current.city = TEST_CITY_A; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  let(:attendance) { walk_in_attendance!(unit, citizen: screening_citizen!(1)) }
  let(:screening) do
    started = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
    Screenings::Complete.call(screening: started, revision_params: revision_params(final_color: "green"),
                              destination: "same_day", destination_params: {}, by: nurse)
    started.reload
  end

  it "nova revisão vira a corrente; a anterior fica; evento só com ids" do
    first = screening.current_revision
    result = described_class.call(screening: screening, by: nurse,
                                  revision_params: revision_params(final_color: "yellow", vitals: { "spo2" => 93 }))
    expect(result).to be_ok
    revision = result.payload[:revision]
    expect(screening.reload.current_revision_id).to eq(revision.id)
    expect(screening.revisions.count).to eq(2)
    expect(first.reload.final_color).to eq("green")
    expect(DomainEvent.where(name: "screening.reassessed").sole.payload)
      .to eq("screening_id" => screening.id, "revision_id" => revision.id, "final_color" => "yellow")
  end

  it "não reavalia escuta em curso, com outro destino, nem atendimento que saiu da espera" do
    other = walk_in_attendance!(unit, citizen: screening_citizen!(2))
    in_progress = Screenings::Start.call(attendance: other, by: nurse).payload[:screening]
    expect(described_class.call(screening: in_progress, by: nurse, revision_params: revision_params).reason)
      .to eq(:not_reassessable)
    screening.attendance.update!(status: "in_care", called_by_user: nurse, called_at: Time.current)
    expect(described_class.call(screening: screening.reload, by: nurse, revision_params: revision_params).reason)
      .to eq(:not_reassessable)
  end

  it "entrada inválida responde antes do estado" do
    expect(described_class.call(screening: screening, by: nurse, revision_params: revision_params(ciap2_code: "Z99")).reason)
      .to eq(:invalid_ciap2)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/screenings/revision_input_spec.rb spec/commands/screenings/complete_spec.rb spec/commands/screenings/reassess_spec.rb`
Expected: FAIL (`uninitialized constant Screenings::RevisionInput`).

- [ ] **Step 3: `Screenings::RevisionInput`**

```ruby
# app/services/screenings/revision_input.rb
# Corpo de uma revisão (ADR 0030; spec §3.1, §3.3; contratos §3) → colunas de
# screening_revisions. A cor sugerida é recalculada aqui com o protocolo
# ativo (nunca vem do cliente); a justificativa só fica quando a cor final
# difere de uma sugestão.
module Screenings
  module RevisionInput
    MAX_NOTE = 500
    MIN_REASON = 10

    module_function

    def call(params, citizen:)
      params = normalize(params)
      ciap = Ciap2.find(params["ciap2_code"])
      return Result.fail(:invalid_ciap2) unless ciap

      vitals = VitalSigns.parse(params["vitals"])
      return vitals if vitals.failure?

      note = params["complaint_note"].to_s.strip.presence
      return too_long("complaint_note") if note && note.length > MAX_NOTE

      final = params["final_color"].to_s
      return Result.fail(:invalid_color) unless RiskSuggestion::COLORS.include?(final)

      values = vitals.payload[:values]
      suggestion = Suggest.call(citizen: citizen, ciap2_code: ciap.code, vitals: values, bmi: vitals.payload[:bmi])
      reason = nil
      if suggestion[:color] && final != suggestion[:color]
        reason = params["color_change_reason"].to_s.strip
        return Result.fail(:color_change_reason_required) if reason.length < MIN_REASON
        return too_long("color_change_reason") if reason.length > MAX_NOTE
      end

      attrs = ScreeningRevision::VITAL_COLUMNS.to_h { |column| [ column, values[column] ] }.merge(
        "ciap2_code" => ciap.code, "ciap2_release_id" => ciap.release_id, "complaint_note" => note,
        "suggested_color" => suggestion[:color], "final_color" => final, "color_change_reason" => reason,
        "rule_protocol_definition_id" => suggestion[:protocol_definition_id], "matched_rules" => suggestion[:matched]
      )
      Result.ok(attrs: attrs, alerts: vitals.payload[:alerts], bmi: vitals.payload[:bmi])
    end

    def normalize(params)
      params = params.to_unsafe_h if params.respond_to?(:to_unsafe_h)
      params.is_a?(Hash) ? params.deep_stringify_keys : {}
    end

    def too_long(field) = Result.fail(:note_too_long, details: { field: field })
    private_class_method :normalize, :too_long
  end
end
```

- [ ] **Step 4: `Screenings::Complete` e `Screenings::Reassess`**

```ruby
# app/commands/screenings/complete.rb
# Concluir a escuta (ADR 0030; spec §4): grava a revisão, define o destino e,
# fora do same_day, fecha o atendimento a partir de waiting — exceção da trava
# do módulo 13 que o trigger attendances_screening_close_guard confere.
# Ordem de travas: atendimento → escuta → unidade (FOR SHARE).
module Screenings
  class Complete
    DESTINATIONS = Screening::DESTINATIONS
    DEFAULT_DUE_IN_DAYS = { "yellow" => 7, "green" => 15, "blue" => 30 }.freeze
    UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/

    def self.call(screening:, revision_params:, destination:, destination_params:, by:)
      attendance = screening.attendance
      input = RevisionInput.call(revision_params, citizen: attendance.citizen)
      return input if input.failure?

      destination = destination.to_s
      plan = plan_for(destination, destination_params, input.payload[:attrs]["final_color"])
      return plan if plan.failure?

      ApplicationRecord.transaction do
        attendance.lock!
        screening.lock!
        status, link = Authorization.check(user: by, health_unit_id: attendance.health_unit_id)
        next Result.fail(status) unless status == :ok
        next Result.fail(:not_in_progress) unless screening.status == "in_progress"
        next Result.fail(:attendance_not_waiting) unless attendance.status == "waiting"

        referral_unit = plan.payload[:referral_unit_id] && HealthUnit.lock_active!(plan.payload[:referral_unit_id])
        revision = ScreeningRevision.create!(input.payload[:attrs].merge(screening: screening, by_user: by))
        request = destination == "schedule" ? schedule_request!(attendance, screening, plan.payload) : nil
        screening.update!(status: "completed", completed_at: Time.current, destination: destination,
                          orientation_note: plan.payload[:orientation_note], current_revision: revision,
                          appointment_request: request, professional_link: link, cbo_code: link.cbo_code)
        request = close!(attendance, destination, referral_unit, plan.payload, by) || request unless destination == "same_day"
        DomainEvents.publish("screening.completed", screening_id: screening.id, attendance_id: attendance.id,
                                                    destination: destination, final_color: revision.final_color)
        Result.ok(screening: screening, attendance: attendance, appointment_request: request)
      end
    rescue HealthUnit::Inactive
      Result.fail(:invalid_unit)
    end

    def self.plan_for(destination, params, color)
      params = params.respond_to?(:to_unsafe_h) ? params.to_unsafe_h : params
      params = params.is_a?(Hash) ? params.deep_stringify_keys : {}
      case destination
      when "same_day" then Result.ok({})
      when "schedule" then schedule_plan(params["schedule"], color)
      when "oriented" then oriented_plan(params["orientation_note"])
      when "referred" then referral_plan(params["referral"])
      else Result.fail(:invalid_destination)
      end
    end

    def self.schedule_plan(raw, color)
      raw = raw.is_a?(Hash) ? raw : {}
      type = AppointmentType.active_types.find_by(key: raw["appointment_type_key"].to_s)
      priority = raw["priority"].to_s
      days = raw["due_in_days"].nil? ? DEFAULT_DUE_IN_DAYS[color] : Integer(raw["due_in_days"].to_s, 10, exception: false)
      valid = type && AppointmentRequest::PRIORITIES.include?(priority) && days&.between?(1, 365)
      valid ? Result.ok(appointment_type_key: type.key, priority: priority, due_in_days: days) : Result.fail(:invalid_schedule)
    end

    def self.oriented_plan(raw)
      note = raw.to_s.strip
      return Result.fail(:orientation_required) if note.empty?
      return Result.fail(:note_too_long, details: { field: "orientation_note" }) if note.length > RevisionInput::MAX_NOTE

      Result.ok(orientation_note: note)
    end

    def self.referral_plan(raw)
      raw = raw.is_a?(Hash) ? raw : {}
      unit_id = raw["referral_unit_id"].presence
      note = raw["referral_note"].to_s.strip.presence
      return Result.fail(:referral_required) if unit_id.nil? && note.nil?
      return Result.fail(:invalid_unit) if unit_id && !(unit_id.to_s.match?(UUID) && HealthUnit.where(active: true).exists?(id: unit_id))

      Result.ok(referral_unit_id: unit_id, referral_note: note)
    end

    # Pedido do módulo 17 com origem na escuta, na unidade do atendimento.
    def self.schedule_request!(attendance, screening, plan)
      HealthUnit.lock_active!(attendance.health_unit_id)
      request = AppointmentRequest.create!(
        kind: "screening", origin_attendance: attendance, origin_screening: screening, citizen: attendance.citizen,
        root_triage: attendance.root_triage, origin_unit: attendance.health_unit, target_unit: attendance.health_unit,
        appointment_type_key: plan[:appointment_type_key], priority: plan[:priority],
        due_on: Time.zone.today + plan[:due_in_days]
      )
      DomainEvents.publish("appointment_request.created", appointment_request_id: request.id,
                                                          origin_attendance_id: attendance.id,
                                                          target_unit_id: attendance.health_unit_id, kind: request.kind)
      request
    end

    # Fecha de waiting com o desfecho do destino; encaminhamento com unidade
    # abre o pedido como o desfecho do módulo 13. Devolve o pedido aberto.
    def self.close!(attendance, destination, referral_unit, plan, by)
      outcome = Attendance::SCREENING_OUTCOMES.fetch(destination)
      attendance.update!(status: "closed", outcome: outcome, closed_by_user: by, closed_at: Time.current,
                         referral_unit: referral_unit,
                         referral_note: destination == "referred" ? plan[:referral_note] : nil)
      request = destination == "referred" ? AppointmentRequests::Lifecycle.open_for!(attendance, outcome: "referred", unit: referral_unit) : nil
      DomainEvents.publish("attendance.closed", attendance_id: attendance.id, outcome: outcome, closed_by_user_id: by.id)
      request
    end
    private_class_method :plan_for, :schedule_plan, :oriented_plan, :referral_plan, :schedule_request!, :close!
  end
end
```

```ruby
# app/commands/screenings/reassess.rb
# Reavaliar enquanto a pessoa espera (ADR 0030; spec §3.2): só escuta
# concluída com destino same_day e atendimento ainda waiting. Nova revisão
# vira a corrente; a cor nova reordena a fila do profissional.
module Screenings
  class Reassess
    def self.call(screening:, revision_params:, by:)
      attendance = screening.attendance
      input = RevisionInput.call(revision_params, citizen: attendance.citizen)
      return input if input.failure?

      ApplicationRecord.transaction do
        attendance.lock!
        screening.lock!
        status, link = Authorization.check(user: by, health_unit_id: attendance.health_unit_id)
        next Result.fail(status) unless status == :ok
        unless screening.status == "completed" && screening.destination == "same_day" && attendance.status == "waiting"
          next Result.fail(:not_reassessable)
        end

        revision = ScreeningRevision.create!(input.payload[:attrs].merge(screening: screening, by_user: by))
        screening.update!(current_revision: revision, professional_link: link, cbo_code: link.cbo_code)
        DomainEvents.publish("screening.reassessed", screening_id: screening.id, revision_id: revision.id,
                                                     final_color: revision.final_color)
        Result.ok(screening: screening, revision: revision)
      end
    end
  end
end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/screenings spec/commands/screenings spec/models/screening_tables_guard_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add app/services/screenings/revision_input.rb app/commands/screenings/complete.rb app/commands/screenings/reassess.rb spec/services/screenings/revision_input_spec.rb spec/commands/screenings/complete_spec.rb spec/commands/screenings/reassess_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: complete the initial listening with its destination and reassess it

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 10: Corridas reais (threads) — duas iniciam juntas; a chamada vence a conclusão

**Files:**
- Test: `spec/commands/screenings/screening_concurrency_spec.rb`

**Interfaces:**
- Consumes: `Screenings::Start`, `Screenings::Complete` (Tasks 8–9), `Attendances::Call` (com `Screenings::Abandon.release!`), `purge_committed_rows(ids)` (chaves `:admin`, `:doctor`, `:doc_user`, `:unit`), `wait_for_lock_wait`.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/commands/screenings/screening_concurrency_spec.rb
require "rails_helper"

# Review Focus 1 (ADR 0030): duas profissionais na mesma pessoa ao mesmo
# tempo — uma escuta só; a chamada do médico no meio da conclusão — a escuta
# vira abandonada e a conclusão recebe not_in_progress, nunca 500 nem
# atendimento fechado duas vezes. Threads reais contra TEST_CITY_A (sem
# fixture transacional), como spec/commands/attendances/call_next_concurrency_spec.rb.
RSpec.describe "Escuta inicial sob disputa" do
  self.use_transactional_tests = false

  let(:release) { Queue.new }
  let(:threads) { [] }
  let(:ids) { {} }

  before do
    CityConnection.with(TEST_CITY_A) do
      Current.set(city: TEST_CITY_A) do
        tag = SecureRandom.hex(4)
        admin = User.create!(email_address: "adm-#{tag}@c.gov.br", password: "senha-segura-123")
        ids[:admin] = admin.id
        unit = HealthUnit.create!(name: "UBS Escuta #{tag}", kind: "ubs")
        ids[:unit] = unit.id
        { doctor: "223505", doc_user: "225125" }.each do |key, cbo|
          user = User.create!(email_address: "#{key}-#{tag}@c.gov.br", password: "senha-segura-123")
          ids[key] = user.id
          Membership.create!(user: user, role: "health_professional", granted_at: Time.current)
          pro = Professional.create!(user: user, professional_name: "P#{key}", council: "COREN", council_state: "PR",
                                     registration_number: "#{tag.to_i(16).to_s[0, 6]}#{cbo[-1]}",
                                     cns: Professionals::Cns.generate("#{tag}#{key}"))
          ProfessionalLink.create!(professional: pro, health_unit: unit, cbo_code: cbo, started_at: Time.current,
                                   started_by_user: admin)
        end
        citizen = Citizen.create!(cpf: CampaignHistory.cpf_for("escuta-#{tag}"), phone: "+55419#{tag.to_i(16).to_s[0, 8].rjust(8, '1')}")
        ids[:citizen] = citizen.id
        definition = { "name" => "escuta-#{tag}", "version" => 1, "start_step_id" => "tosse",
                       "steps" => [ { "id" => "tosse", "prompt" => "?", "answer_type" => "boolean",
                                      "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 5, "false" => 0 } } ],
                       "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 9 } } }
        protocol = ProtocolDefinition.create!(name: definition["name"], version: 1, status: "draft", definition: definition)
        ids[:protocol] = protocol.id
        conversation = Conversation.create!(channel: "web", citizen: citizen, phone: citizen.phone, state: "completed")
        ids[:conversation] = conversation.id
        triage = Triage.create!(conversation: conversation, protocol_definition: protocol, protocol_name: protocol.name,
                                status: "completed", tier: "baixa", priority: 9, answers: {}, completed_at: Time.current)
        ids[:triage] = triage.id
        attendance = Attendance.create!(triage: triage, citizen: citizen, health_unit: unit, checked_in_by_user: admin,
                                        checked_in_at: Time.current, check_in_method: "code")
        ids[:attendance] = attendance.id
      end
    end
    allow(Screenings::Ciap2).to receive(:find)
      .and_return(Screenings::Ciap2::Code.new(code: "K86", label: "Hipertensão", release_id: SecureRandom.uuid))
  end

  after do
    3.times { release << true }
    threads.each do |t|
      t.join(5) || t.kill
    rescue StandardError
      nil
    end
    CityConnection.with(TEST_CITY_A) do
      ApplicationRecord.transaction do
        ApplicationRecord.connection.execute("SET LOCAL session_replication_role = replica")
        screening_ids = Screening.where(attendance_id: ids[:attendance]).pluck(:id)
        DomainEvent.where("payload->>'attendance_id' = ? OR payload->>'screening_id' IN (?)", ids[:attendance].to_s,
                          screening_ids.presence || [ "" ]).delete_all
        Screening.where(id: screening_ids).update_all(current_revision_id: nil)
        ScreeningRevision.where(screening_id: screening_ids).delete_all
        Screening.where(id: screening_ids).delete_all
        Attendance.where(id: ids[:attendance]).delete_all
        Triage.where(id: ids[:triage]).delete_all
        Conversation.where(id: ids[:conversation]).delete_all
        Citizen.where(id: ids[:citizen]).delete_all
        ProtocolDefinition.where(id: ids[:protocol]).delete_all
      end
    end
    purge_committed_rows(ids)
  end

  def in_city(&) = CityConnection.with(TEST_CITY_A) { Current.set(city: TEST_CITY_A, &) }
  def attendance = Attendance.find(ids[:attendance])

  def hold_on(event_name)
    holder = nil
    holding = Queue.new
    original = DomainEvents.method(:publish)
    allow(DomainEvents).to receive(:publish) do |*args, **kwargs, &blk|
      result = original.call(*args, **kwargs, &blk)
      if Thread.current == holder && args.first == event_name
        holding << true
        release.pop(timeout: 10) or raise "timeout esperando release"
      end
      result
    end
    [ holding, ->(thread) { holder = thread } ]
  end

  it "duas iniciam juntas: a segunda espera o lock e recebe already_screening; uma escuta só" do
    holding, mark = hold_on("screening.started")
    first = Queue.new
    go = Queue.new
    threads << (holder = Thread.new do
      go.pop(timeout: 5)
      first << in_city { Screenings::Start.call(attendance: attendance, by: User.find(ids[:doctor])) }
    end)
    mark.call(holder)
    go << true
    holding.pop(timeout: 5) or raise "a primeira não travou o atendimento"

    second = Queue.new
    threads << other = Thread.new { second << in_city { Screenings::Start.call(attendance: attendance, by: User.find(ids[:doc_user])) } }
    expect(wait_for_lock_wait).to be(true)

    release << true
    expect(holder.join(5)).to be(holder)
    expect(other.join(5)).to be(other)
    expect(first.pop(timeout: 1)).to be_ok
    expect(second.pop(timeout: 1).reason).to eq(:already_screening)
    expect(in_city { Screening.where(attendance_id: ids[:attendance]).count }).to eq(1)
  end

  it "a chamada vence a conclusão: escuta abandonada, conclusão recebe not_in_progress, atendimento em atendimento" do
    screening = in_city { Screenings::Start.call(attendance: attendance, by: User.find(ids[:doctor])).payload[:screening] }
    holding, mark = hold_on("attendance.called")
    called = Queue.new
    go = Queue.new
    threads << (holder = Thread.new do
      go.pop(timeout: 5)
      called << in_city { Attendances::Call.call(attendance: attendance, health_unit_id: ids[:unit], by: User.find(ids[:doc_user])) }
    end)
    mark.call(holder)
    go << true
    holding.pop(timeout: 5) or raise "a chamada não travou o atendimento"

    completed = Queue.new
    threads << completer = Thread.new do
      completed << in_city do
        Screenings::Complete.call(screening: Screening.find(screening.id), revision_params: revision_params,
                                  destination: "oriented", destination_params: { "orientation_note" => "repouso e água" },
                                  by: User.find(ids[:doctor]))
      end
    end
    expect(wait_for_lock_wait).to be(true)

    release << true
    expect(holder.join(5)).to be(holder)
    expect(completer.join(5)).to be(completer)
    expect(called.pop(timeout: 1)).to be_ok
    expect(completed.pop(timeout: 1).reason).to eq(:not_in_progress)
    in_city do
      expect(Screening.find(screening.id).status).to eq("abandoned")
      expect(ScreeningRevision.where(screening_id: screening.id)).to be_empty
      expect(attendance.status).to eq("in_care")
    end
  end
end
```

(`revision_params` vem de `ScreeningHelpers`. Cada thread espera o `go` antes de chamar o comando: o `holder` já está marcado quando ela chega ao `publish`.)

- [ ] **Step 2: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/commands/screenings/screening_concurrency_spec.rb`
Expected: PASS. Mutação para conferir que o teste morde: tire `attendance.lock!` de `Screenings::Start` — o primeiro exemplo passa a criar duas escutas (o índice único levanta `RecordNotUnique` na segunda: vermelho); tire `Screenings::Abandon.release!` de `Attendances::Call` — o segundo exemplo conclui e fecha um atendimento já chamado (vermelho: `attendance_not_waiting` ≠ `not_in_progress`).

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add spec/commands/screenings/screening_concurrency_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "test: race two screenings and a call against a completion

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 11: Filas — acolhimento por chegada e profissional por cor; formas JSON da escuta

**Files:**
- Create: `app/services/screenings/queue.rb`, `app/services/screenings/json.rb`
- Modify: `app/commands/attendances/unit_queue.rb`, `app/models/attendance.rb` (nada além do `has_one :screening` da Task 3)
- Test: `spec/services/screenings/queue_spec.rb`, `spec/commands/attendances/unit_queue_screening_spec.rb`, `spec/services/screenings/json_spec.rb`

**Interfaces:**
- Consumes: `Screening`, `ScreeningRevision`, `Screenings::VitalSigns.json/.alerts/.bmi`, `Screenings::Ciap2.label`, `Protocols::ConditionText.call`.
- Produces:
  - `Screenings::Queue.requiring(unit) -> Relation<Attendance>`, `.pending(unit) -> Relation` (aguardando, exigem escuta, sem escuta `completed`), `.items(unit) -> Array<Attendance>` (por chegada, depois id);
  - `Attendances::UnitQueue::ORDER` novo (escuta concluída primeiro, por cor e chegada; depois a ordem de hoje); `UnitQueue::INCLUDES` com `screening: :current_revision`;
  - `Screenings::Json.screening(screening, with_revisions: false) -> Hash` (contratos §3, `<screening>`), `.revision(revision) -> Hash` (`<revision>`), `.queue_item(attendance) -> Hash` (item de `screening_queue`), `.queue_block(attendance, now: Time.current) -> Hash|nil` (`{ id, color, destination, waited_minutes }`, contrato §9), `.staff_name(user) -> String|nil`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/screenings/queue_spec.rb
require "rails_helper"

# ADR 0030 (spec §4): a fila do acolhimento são os atendimentos aguardando
# que exigem escuta pelo escopo e não têm escuta concluída, por chegada.
RSpec.describe Screenings::Queue do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  let(:t0) { Time.zone.parse("2026-10-07 08:00") }

  it "walk_in: só demanda espontânea, sem escuta concluída, por chegada; em curso e abandonada continuam" do
    late = walk_in_attendance!(unit, citizen: screening_citizen!(1), checked_in_at: t0 + 20.minutes)
    early = walk_in_attendance!(unit, citizen: screening_citizen!(2), checked_in_at: t0)
    scheduled = scheduled_attendance!(unit, citizen: screening_citizen!(3), checked_in_at: t0 - 5.minutes)
    done = walk_in_attendance!(unit, citizen: screening_citizen!(4), checked_in_at: t0 - 10.minutes)
    started = Screenings::Start.call(attendance: done, by: nurse).payload[:screening]
    Screenings::Complete.call(screening: started, revision_params: revision_params, destination: "same_day",
                              destination_params: {}, by: nurse)
    Screenings::Start.call(attendance: late, by: nurse)
    elsewhere = walk_in_attendance!(create_unit("UBS Outra"), citizen: screening_citizen!(5), checked_in_at: t0)

    expect(described_class.items(unit).map(&:id)).to eq([ early.id, late.id ])
    expect(described_class.items(unit).map(&:id)).not_to include(scheduled.id, done.id, elsewhere.id)
    unit.update!(screening_scope: "all")
    expect(described_class.items(unit.reload).map(&:id)).to eq([ scheduled.id, early.id, late.id ])
  end
end
```

```ruby
# spec/commands/attendances/unit_queue_screening_spec.rb
require "rails_helper"

# ADR 0030 (spec §4; contratos §4): com escuta same_day primeiro, por cor
# (red, yellow, green, blue) e chegada; depois os demais como hoje (prioridade
# da triagem raiz, chegada). Sem escuta nenhuma, a ordem é a do módulo 13.
RSpec.describe Attendances::UnitQueue, "com escuta" do
  before { Current.city = TEST_CITY_A; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  let(:t0) { Time.zone.parse("2026-10-07 08:00") }

  def screened!(n, color, at:)
    attendance = walk_in_attendance!(unit, citizen: screening_citizen!(n), checked_in_at: at)
    screening = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
    Screenings::Complete.call(screening: screening, revision_params: revision_params(final_color: color),
                              destination: "same_day", destination_params: {}, by: nurse)
    attendance
  end

  it "cor antes de chegada; quem não tem escuta vem depois na ordem de hoje" do
    plain_urgent = walk_in_attendance!(unit, citizen: screening_citizen!(1), checked_in_at: t0)
    plain_urgent.triage.update_columns(priority: 1)
    plain_later = walk_in_attendance!(unit, citizen: screening_citizen!(2), checked_in_at: t0 + 1.minute)
    plain_later.triage.update_columns(priority: 9)
    green = screened!(3, "green", at: t0 + 2.minutes)
    red = screened!(4, "red", at: t0 + 30.minutes)
    yellow_early = screened!(5, "yellow", at: t0 + 3.minutes)
    yellow_late = screened!(6, "yellow", at: t0 + 10.minutes)
    blue = screened!(7, "blue", at: t0 + 1.minute)

    expect(described_class.waiting(unit.id).map(&:id))
      .to eq([ red.id, yellow_early.id, yellow_late.id, green.id, blue.id, plain_urgent.id, plain_later.id ])
    expect(ApplicationRecord.transaction { described_class.lock_next_waiting(unit.id) }.id).to eq(red.id)
  end

  it "reavaliar muda a cor e a posição" do
    green = screened!(1, "green", at: t0)
    yellow = screened!(2, "yellow", at: t0 + 5.minutes)
    Screenings::Reassess.call(screening: green.screening, by: nurse, revision_params: revision_params(final_color: "red"))
    expect(described_class.waiting(unit.id).map(&:id)).to eq([ green.id, yellow.id ])
  end
end
```

```ruby
# spec/services/screenings/json_spec.rb
require "rails_helper"

# Contratos §3–§4: formas da escuta, da revisão, do item da fila do
# acolhimento e do bloco que a fila do profissional (e a recepção) recebe.
RSpec.describe Screenings::Json do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  let(:attendance) { walk_in_attendance!(unit, citizen: screening_citizen!(1), checked_in_at: 25.minutes.ago) }

  def complete!(destination: "same_day", params: {}, **revision)
    screening = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
    Screenings::Complete.call(screening: screening, revision_params: revision_params(**revision), destination: destination,
                              destination_params: params, by: nurse)
    screening.reload
  end

  it "escuta concluída com a revisão corrente: chaves do contrato, números como número, opcionais só quando há" do
    acolhimento!
    screening = complete!(final_color: "red", complaint_note: "dor de cabeça forte",
                          vitals: { "systolic" => 185, "diastolic" => 110, "temperature_c" => "38,2",
                                    "weight_kg" => "80", "height_cm" => 175 })
    json = described_class.screening(screening).deep_stringify_keys
    expect(json.keys).to match_array(%w[id attendance_id status started_at completed_at destination current_revision revisions_count])
    expect(json.values_at("status", "destination", "revisions_count")).to eq([ "completed", "same_day", 1 ])
    revision = json["current_revision"]
    expect(revision.keys).to match_array(%w[id created_at by ciap2 complaint_note vitals alerts suggested_color final_color matched_rules])
    expect(revision["by"]).to eq("id" => nurse.id, "name" => nurse.professional.professional_name)
    expect(revision["ciap2"]).to eq("code" => "K86", "label" => "Hipertensão sem complicações")
    expect(revision["vitals"]).to eq("systolic" => 185, "diastolic" => 110, "temperature_c" => 38.2, "weight_kg" => 80.0,
                                     "height_cm" => 175, "bmi" => 26.1)
    expect(revision["alerts"]).to eq(%w[systolic_high diastolic_high temperature_high])
    expect(revision["matched_rules"]).to eq([ { "index" => 0, "text" => "pressão sistólica ≥ 180 ou saturação < 90" } ])
    expect(described_class.screening(screening, with_revisions: true)[:revisions].size).to eq(1)
  end

  it "oriented leva orientation_note; schedule leva appointment_request_id" do
    oriented = complete!(destination: "oriented", params: { "orientation_note" => "repouso" })
    expect(described_class.screening(oriented)).to include(orientation_note: "repouso")
    expect(described_class.screening(oriented)).not_to have_key(:appointment_request_id)
  end

  it "bloco da fila: cor, destino e espera em minutos; nada sem escuta concluída; sem queixa nem sinais" do
    expect(described_class.queue_block(attendance)).to be_nil
    complete!(final_color: "yellow")
    block = described_class.queue_block(attendance.reload)
    expect(block).to eq(id: attendance.screening.id, color: "yellow", destination: "same_day", waited_minutes: 25)
  end

  it "item da fila do acolhimento" do
    Screenings::Start.call(attendance: attendance, by: nurse)
    item = described_class.queue_item(attendance.reload).deep_stringify_keys
    expect(item.keys).to match_array(%w[attendance_id citizen checked_in_at triage_priority screening])
    expect(item["citizen"]).to eq("id" => attendance.citizen_id, "cpf_masked" => attendance.citizen.cpf_masked)
    expect(item["screening"]).to include("status" => "in_progress", "started_by_name" => nurse.professional.professional_name)
  end
end
```

(O texto de `matched_rules` usa `Protocols::ConditionText` com os rótulos da Task 2; a regra 0 de `ScreeningHelpers::RULES` é `any` de duas condições, que no topo vira "… ou …" sem parênteses.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/screenings/queue_spec.rb spec/commands/attendances/unit_queue_screening_spec.rb spec/services/screenings/json_spec.rb`
Expected: FAIL (`uninitialized constant Screenings::Queue`; ordem antiga).

- [ ] **Step 3: Fila do acolhimento**

```ruby
# app/services/screenings/queue.rb
# Fila do acolhimento (ADR 0030; spec §4; contratos §3): atendimentos
# aguardando na unidade que exigem escuta pelo escopo e não têm escuta
# concluída, por chegada. Em curso e abandonada continuam aqui (com o bloco
# da escuta), para a equipe ver quem já está sendo escutado.
module Screenings
  module Queue
    INCLUDES = [ :citizen, :triage, { appointment: { request: :root_triage } }, { screening: { started_by_user: :professional } } ].freeze

    module_function

    def requiring(unit)
      scope = Attendance.where(health_unit_id: unit.id)
      unit.screening_scope == "all" ? scope : scope.where(appointment_id: nil)
    end

    def pending(unit)
      requiring(unit).waiting.where.not(id: Screening.completed_screenings.select(:attendance_id))
    end

    def items(unit) = pending(unit).includes(*INCLUDES).order(:checked_in_at, :id).to_a
  end
end
```

- [ ] **Step 4: Ordem da fila do profissional**

Em `app/commands/attendances/unit_queue.rb`, troque `INCLUDES`, `ROOT_TRIAGE_JOINS` e `ORDER` (e o comentário do topo):

```ruby
# Fila da unidade (spec 2026-09-25 §6; ADR 0030 §4): primeiro quem tem escuta
# concluída (same_day — os outros destinos já fecharam o atendimento), pela
# cor da revisão corrente (red, yellow, green, blue) e pela chegada; depois os
# demais pela prioridade da triagem raiz (sem prioridade por último), pela
# chegada e, no empate, pelo id; em atendimento por hora da chamada. A ordem
# mora no SQL para o "chamar próximo" travar o primeiro disponível com FOR
# UPDATE SKIP LOCKED na MESMA ordem que a tela mostra.
module Attendances
  module UnitQueue
    INCLUDES = [ :citizen, { called_by_user: :professional }, :triage, { appointment: { request: :root_triage } },
                 { screening: :current_revision } ].freeze

    ROOT_TRIAGE_JOINS = <<~SQL.squish.freeze
      LEFT JOIN triages queue_triages ON queue_triages.id = attendances.triage_id
      LEFT JOIN appointments queue_appointments ON queue_appointments.id = attendances.appointment_id
      LEFT JOIN appointment_requests queue_requests ON queue_requests.id = queue_appointments.request_id
      LEFT JOIN triages queue_root_triages ON queue_root_triages.id = queue_requests.root_triage_id
      LEFT JOIN screenings queue_screenings ON queue_screenings.attendance_id = attendances.id
        AND queue_screenings.status = 'completed'
      LEFT JOIN screening_revisions queue_revisions ON queue_revisions.id = queue_screenings.current_revision_id
    SQL
    ORDER = Arel.sql(
      "(queue_screenings.id IS NULL) ASC, " \
      "CASE queue_revisions.final_color WHEN 'red' THEN 0 WHEN 'yellow' THEN 1 WHEN 'green' THEN 2 WHEN 'blue' THEN 3 END ASC NULLS LAST, " \
      "CASE WHEN queue_screenings.id IS NULL THEN COALESCE(queue_triages.priority, queue_root_triages.priority) END ASC NULLS LAST, " \
      "attendances.checked_in_at ASC, attendances.id ASC"
    ).freeze
```

(`ordered_waiting`, `waiting`, `lock_next_waiting` e `in_care` ficam como estão: o `FOR UPDATE OF attendances` já restringe o lock à tabela do atendimento.)

- [ ] **Step 5: Formas JSON**

```ruby
# app/services/screenings/json.rb
# Formas da escuta (contratos §3–§4). A recepção só recebe queue_block (cor,
# destino, espera) — nunca queixa nem sinais. Números saem como número
# (BigDecimal vira Float); chaves opcionais só aparecem quando há valor.
module Screenings
  module Json
    module_function

    def screening(screening, with_revisions: false)
      revision = screening.current_revision
      json = {
        id: screening.id, attendance_id: screening.attendance_id, status: screening.status,
        started_at: screening.started_at&.iso8601, completed_at: screening.completed_at&.iso8601,
        destination: screening.destination, current_revision: revision && revision(revision),
        revisions_count: screening.revisions.size
      }
      json[:orientation_note] = screening.orientation_note if screening.orientation_note
      json[:appointment_request_id] = screening.appointment_request_id if screening.appointment_request_id
      json[:revisions] = screening.revisions.order(:created_at, :id).map { |r| revision(r) } if with_revisions
      json
    end

    def revision(revision)
      vitals = revision.vitals
      json = {
        id: revision.id, created_at: revision.created_at.iso8601,
        by: { id: revision.by_user_id, name: staff_name(revision.by_user) },
        ciap2: { code: revision.ciap2_code, label: Ciap2.label(revision.ciap2_code, revision.ciap2_release_id) },
        vitals: VitalSigns.json(vitals).merge("bmi" => VitalSigns.bmi(vitals)).compact,
        alerts: VitalSigns.alerts(vitals), suggested_color: revision.suggested_color,
        final_color: revision.final_color, matched_rules: matched_rules(revision)
      }
      json[:complaint_note] = revision.complaint_note if revision.complaint_note
      json[:color_change_reason] = revision.color_change_reason if revision.color_change_reason
      json
    end

    def matched_rules(revision)
      return [] if revision.matched_rules.blank?

      rules = ProtocolDefinition.find_by(id: revision.rule_protocol_definition_id)&.definition&.dig("risk_rules") || []
      revision.matched_rules.map { |index| { index: index, text: Protocols::ConditionText.call(rules.dig(index, "when")) } }
    end

    def queue_item(attendance)
      screening = attendance.screening
      {
        attendance_id: attendance.id, citizen: { id: attendance.citizen_id, cpf_masked: attendance.citizen.cpf_masked },
        checked_in_at: attendance.checked_in_at.iso8601, triage_priority: attendance.priority,
        screening: screening && { id: screening.id, status: screening.status,
                                  started_by_name: staff_name(screening.started_by_user) }
      }
    end

    def queue_block(attendance, now: Time.current)
      screening = attendance.screening
      return nil unless screening&.completed? && screening.current_revision

      { id: screening.id, color: screening.current_revision.final_color, destination: screening.destination,
        waited_minutes: ((now - attendance.checked_in_at) / 60).floor }
    end

    # Como a fila do módulo 13: nome profissional; sem perfil, o e-mail.
    def staff_name(user)
      return nil unless user

      user.professional&.professional_name || user.email_address
    end
  end
end
```

- [ ] **Step 6: Rode e veja passar (e a fila de sempre)**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/screenings spec/commands/attendances spec/requests/attendances_spec.rb spec/requests/attendance_reference_units_spec.rb`
Expected: PASS (as specs antigas da fila não têm escuta: ordem igual à de hoje).

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add app/services/screenings/queue.rb app/services/screenings/json.rb app/commands/attendances/unit_queue.rb spec/services/screenings/queue_spec.rb spec/commands/attendances/unit_queue_screening_spec.rb spec/services/screenings/json_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: order the professional queue by risk colour and list the screening queue

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 12: API — rotas da escuta, sugestão, leitura auditada, fila com cor, escopo da unidade e busca de CIAP-2

**Files:**
- Create: `app/controllers/screenings_controller.rb`, `app/controllers/ciap2_codes_controller.rb`, `app/services/protocols/simulate_screening.rb`
- Modify: `app/controllers/attendances_controller.rb`, `app/controllers/health_units_controller.rb`, `app/controllers/authoring/protocols_controller.rb`, `config/routes.rb`
- Test: `spec/requests/screenings_spec.rb`, `spec/requests/screening_queue_and_units_spec.rb`, `spec/requests/authoring/simulate_screening_spec.rb`

**Interfaces:**
- Consumes: Tasks 7–11 (`Screenings::Start/Abandon/Complete/Reassess`, `Screenings::Suggest`, `Screenings::RiskSuggestion`, `Screenings::VitalSigns`, `Screenings::Ciap2`, `Screenings::Queue`, `Screenings::Json`), `Professionals::ClinicalAuthorization.check`, `Protocols::Validation::Screening` (Task 2), `ProtocolPolicy#simulate?`.
- Produces também: `Protocols::SimulateScreening.call(definition:, vitals: {}, ciap2_code: nil, profile: {}) -> Hash` (total; nunca grava; erros de schema/gate/sinais vão em `errors`, sempre 200).
- Produces (contratos §3–§5; formatos exatos lá):
  - `GET /attendance/units/:id/screening_queue` → `{ items: [...] }` (balcão e profissionais);
  - `POST /attendance/attendances/:id/screening` → 201 `<screening>`;
  - `POST /attendance/screenings/:id/abandon` | `/complete` | `/reassess` → 200 `<screening>`;
  - `GET /attendance/screenings/:id` → `<screening>` com `revisions`, publica `screening.viewed`;
  - `POST /attendance/screenings/suggest { ciap2_code, vitals, attendance_id }` → `{ suggested_color, matched_rules, alerts, bmi }` (contrato §9: `attendance_id`, vínculo ativo na unidade do atendimento, 403 `missing_link`);
  - `GET /attendance/units/:id/queue`: cada item ganha `screening: { id, color, destination, waited_minutes } | null` (o `id` vem do contrato §9);
  - `POST /attendance/attendances/:id/call` e `POST /attendance/units/:id/call_next`: `attendance` ganha `screening: <screening> | null` (só escuta concluída; publica `screening.viewed`);
  - `POST /attendance/units` e `/units/:id` aceitam `screening_scope` (422 `invalid_screening_scope`); `GET /attendance/units` e `units/all` devolvem o campo;
  - `POST /attendance/ciap2/search { q }` → `{ items: [ { code, label } ] }` (até 20); 503 `terminology_unavailable` sem release CIAP-2 ativa (contrato §9);
  - `POST /authoring/protocols/simulate_screening { definition, vitals, ciap2_code, profile }` → `{ suggested_color, matched_rules, errors, warnings }` (contrato §9; usa a definição em edição, não a ativa).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/requests/screenings_spec.rb
require "rails_helper"

# Contratos §3: escuta inicial em /attendance.
RSpec.describe "Escuta inicial", type: :request do
  before { Current.city = TEST_CITY_A; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  let(:attendance) { walk_in_attendance!(unit, citizen: screening_citizen!(1)) }
  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]

  def start!
    sign_in_as(nurse)
    json_post "/attendance/attendances/#{attendance.id}/screening"
    body
  end

  it "inicia (201 com a forma do contrato); de novo 409; recepção 403; outra unidade 403; CBO fora 403" do
    expect(start!.keys).to match_array(%w[id attendance_id status started_at completed_at destination current_revision revisions_count])
    expect(response).to have_http_status(:created)
    expect(body.values_at("status", "current_revision", "revisions_count")).to eq([ "in_progress", nil, 0 ])
    json_post "/attendance/attendances/#{attendance.id}/screening"
    expect(status_and_error).to eq([ 409, "already_screening" ])

    other = walk_in_attendance!(unit, citizen: screening_citizen!(2))
    sign_in_as(reception!)
    json_post "/attendance/attendances/#{other.id}/screening"
    expect(status_and_error).to eq([ 403, "missing_role" ])
    sign_in_as(screener!(create_unit("UBS Outra")))
    json_post "/attendance/attendances/#{other.id}/screening"
    expect(status_and_error).to eq([ 403, "missing_link" ])
    sign_in_as(screener!(unit, cbo: "322405"))
    json_post "/attendance/attendances/#{other.id}/screening"
    expect(status_and_error).to eq([ 403, "cbo_not_allowed" ])
    json_post "/attendance/attendances/#{SecureRandom.uuid}/screening"
    expect(status_and_error).to eq([ 404, "not_found" ])
  end

  it "conclui com destino; erros de entrada 422 com field; escuta já concluída 409" do
    id = start!["id"]
    json_post "/attendance/screenings/#{id}/complete", revision_params(vitals: { "spo2" => 30 }).merge(destination: "same_day")
    expect([ response.status, body["error"], body["field"] ]).to eq([ 422, "implausible_vital", "spo2" ])
    json_post "/attendance/screenings/#{id}/complete", revision_params(vitals: "lixo").merge(destination: "same_day")
    expect([ response.status, body["error"], body["field"] ]).to eq([ 422, "implausible_vital", "vitals" ])
    json_post "/attendance/screenings/#{id}/complete", revision_params.merge(destination: "amanhã")
    expect(status_and_error).to eq([ 422, "invalid_destination" ])
    json_post "/attendance/screenings/#{id}/complete", revision_params.merge(destination: "oriented")
    expect(status_and_error).to eq([ 422, "orientation_required" ])

    json_post "/attendance/screenings/#{id}/complete",
              revision_params(complaint_note: "dor de cabeça").merge(destination: "oriented", orientation_note: "repouso")
    expect(response).to have_http_status(:ok)
    expect(body.values_at("status", "destination", "orientation_note")).to eq([ "completed", "oriented", "repouso" ])
    expect(body.dig("current_revision", "complaint_note")).to eq("dor de cabeça")
    json_post "/attendance/screenings/#{id}/complete", revision_params.merge(destination: "same_day")
    expect(status_and_error).to eq([ 409, "not_in_progress" ])
  end

  it "schedule pelo corpo do contrato abre o pedido" do
    id = start!["id"]
    json_post "/attendance/screenings/#{id}/complete",
              revision_params(final_color: "green").merge(destination: "schedule",
                                                          schedule: { appointment_type_key: appointment_type!.key, priority: "routine" })
    expect(response).to have_http_status(:ok)
    expect(AppointmentRequest.find(body["appointment_request_id"])).to have_attributes(kind: "screening", due_on: Time.zone.today + 15)
  end

  it "reavalia (200) só same_day e aguardando; abandona (200) só em curso" do
    id = start!["id"]
    json_post "/attendance/screenings/#{id}/reassess", revision_params
    expect(status_and_error).to eq([ 409, "not_reassessable" ])
    json_post "/attendance/screenings/#{id}/complete", revision_params.merge(destination: "same_day")
    json_post "/attendance/screenings/#{id}/reassess", revision_params(final_color: "yellow")
    expect([ response.status, body["revisions_count"], body.dig("current_revision", "final_color") ]).to eq([ 200, 2, "yellow" ])
    json_post "/attendance/screenings/#{id}/abandon"
    expect(status_and_error).to eq([ 409, "not_in_progress" ])
  end

  it "leitura: profissional da unidade lê com as revisões e deixa trilha; recepção e outra unidade 403" do
    id = start!["id"]
    json_post "/attendance/screenings/#{id}/complete", revision_params.merge(destination: "same_day")
    doctor = screener!(unit, cbo: "225125")
    sign_in_as(doctor)
    get "/attendance/screenings/#{id}"
    expect(response).to have_http_status(:ok)
    expect(body["revisions"].size).to eq(1)
    expect(DomainEvent.where(name: "screening.viewed").pluck(:payload)).to eq([ { "screening_id" => id, "user_id" => doctor.id } ])
    sign_in_as(reception!)
    get "/attendance/screenings/#{id}"
    expect(status_and_error).to eq([ 403, "missing_role" ])
    sign_in_as(screener!(create_unit("UBS Outra"), cbo: "225125"))
    get "/attendance/screenings/#{id}"
    expect(status_and_error).to eq([ 403, "missing_link" ])
  end

  it "sugestão sem gravar: cor, regras, alertas e IMC; atendimento desconhecido 404; outra unidade 403; sinais lixo 422" do
    acolhimento!
    sign_in_as(nurse)
    json_post "/attendance/screenings/suggest", ciap2_code: "K86", attendance_id: attendance.id,
                                                vitals: { systolic: 185, diastolic: 110, weight_kg: "80", height_cm: 175 }
    expect(body).to eq("suggested_color" => "red",
                       "matched_rules" => [ { "index" => 0, "text" => "pressão sistólica ≥ 180 ou saturação < 90" } ],
                       "alerts" => %w[systolic_high diastolic_high], "bmi" => 26.1)
    expect(ScreeningRevision.count).to eq(0)
    json_post "/attendance/screenings/suggest", ciap2_code: "K86", attendance_id: SecureRandom.uuid, vitals: {}
    expect(status_and_error).to eq([ 404, "not_found" ])
    json_post "/attendance/screenings/suggest", ciap2_code: "K86", attendance_id: attendance.id, vitals: { spo2: "x" }
    expect(status_and_error).to eq([ 422, "implausible_vital" ])
    json_post "/attendance/screenings/suggest", ciap2_code: "Z99", attendance_id: attendance.id, vitals: {}
    expect(status_and_error).to eq([ 422, "invalid_ciap2" ])
    sign_in_as(screener!(create_unit("UBS Outra")))
    json_post "/attendance/screenings/suggest", ciap2_code: "K86", attendance_id: attendance.id, vitals: {}
    expect(status_and_error).to eq([ 403, "missing_link" ])
  end

  it "busca de CIAP-2 por nome ou código, só para profissionais; sem release ativa 503" do
    sign_in_as(nurse)
    json_post "/attendance/ciap2/search", q: "tosse"
    expect(body).to eq("items" => [ { "code" => "R05", "label" => "Tosse" } ])
    json_post "/attendance/ciap2/search", q: [ "x" ]
    expect(body).to eq("items" => [])
    allow(Screenings::Ciap2).to receive(:release).and_return(nil)
    json_post "/attendance/ciap2/search", q: "tosse"
    expect(status_and_error).to eq([ 503, "terminology_unavailable" ])
    sign_in_as(reception!)
    json_post "/attendance/ciap2/search", q: "tosse"
    expect(status_and_error).to eq([ 403, "missing_role" ])
  end
end
```

```ruby
# spec/requests/screening_queue_and_units_spec.rb
require "rails_helper"

# Contratos §3–§5: fila do acolhimento, fila do profissional com cor (a
# recepção só vê cor, destino e espera), detalhe da chamada e escopo da unidade.
RSpec.describe "Filas com escuta e escopo da unidade", type: :request do
  before { Current.city = TEST_CITY_A; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  def body = JSON.parse(response.body)

  def screened!(n, color)
    attendance = walk_in_attendance!(unit, citizen: screening_citizen!(n), checked_in_at: 15.minutes.ago)
    screening = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
    Screenings::Complete.call(screening: screening,
                              revision_params: revision_params(final_color: color, complaint_note: "MARCADOR-QUEIXA",
                                                               vitals: { "systolic" => 150, "diastolic" => 95 }),
                              destination: "same_day", destination_params: {}, by: nurse)
    attendance
  end

  it "fila do acolhimento para a recepção: quem espera escuta, por chegada" do
    waiting = walk_in_attendance!(unit, citizen: screening_citizen!(1))
    screened!(2, "green")
    sign_in_as(reception!)
    get "/attendance/units/#{unit.id}/screening_queue"
    expect(response).to have_http_status(:ok)
    expect(body["items"].map { |i| i["attendance_id"] }).to eq([ waiting.id ])
    expect(body["items"].first.keys).to match_array(%w[attendance_id citizen checked_in_at triage_priority screening])
  end

  it "fila do profissional: vermelho no topo; a recepção recebe só cor, destino e espera" do
    plain = walk_in_attendance!(unit, citizen: screening_citizen!(1), checked_in_at: 1.hour.ago)
    red = screened!(2, "red")
    sign_in_as(reception!)
    get "/attendance/units/#{unit.id}/queue"
    expect(body["waiting"].map { |i| i["id"] }).to eq([ red.id, plain.id ])
    expect(body["waiting"].first["screening"])
      .to eq("id" => red.screening.id, "color" => "red", "destination" => "same_day", "waited_minutes" => 15)
    expect(body["waiting"].last["screening"]).to be_nil
    expect(response.body).not_to include("MARCADOR-QUEIXA", "systolic", "complaint")
  end

  it "o detalhe da chamada traz a escuta para o profissional e deixa trilha" do
    red = screened!(1, "red")
    doctor = screener!(unit, cbo: "225125")
    sign_in_as(doctor)
    json_post "/attendance/attendances/#{red.id}/call", health_unit_id: unit.id
    expect(response).to have_http_status(:ok)
    expect(body.dig("attendance", "screening", "current_revision", "final_color")).to eq("red")
    expect(DomainEvent.where(name: "screening.viewed").pluck(:payload))
      .to eq([ { "screening_id" => red.screening.id, "user_id" => doctor.id } ])
  end

  it "o admin muda o escopo; valor inválido 422; as listas devolvem o campo" do
    admin = staff_with("admin-escopo@cidade.gov.br", "municipal_admin")
    sign_in_as(admin)
    json_post "/attendance/units/#{unit.id}", name: unit.name, kind: unit.kind, screening_scope: "all"
    expect(response).to have_http_status(:ok)
    expect(body.dig("unit", "screening_scope")).to eq("all")
    json_post "/attendance/units/#{unit.id}", name: unit.name, kind: unit.kind, screening_scope: "todos"
    expect([ response.status, body["error"] ]).to eq([ 422, "invalid_screening_scope" ])
    json_post "/attendance/units/#{unit.id}", name: unit.name, kind: unit.kind
    expect(unit.reload.screening_scope).to eq("all")
    get "/attendance/units/all"
    expect(body["units"].first).to include("screening_scope" => "all")
    sign_in_as(nurse)
    get "/attendance/units"
    expect(body["units"].first).to include("screening_scope" => "all")
  end
end
```

```ruby
# spec/requests/authoring/simulate_screening_spec.rb
require "rails_helper"

# Contrato §9: simulador do editor para a variante screening. Usa a definição
# em edição (não a ativa), nunca grava, sempre 200 com errors.
RSpec.describe "Simulador do protocolo de acolhimento", type: :request do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  def body = JSON.parse(response.body)
  def simulate(params) = post("/authoring/protocols/simulate_screening", params: params, as: :json)
  def definition(rules = ScreeningHelpers::RULES)
    { "name" => "acolhimento", "version" => 3, "kind" => "screening", "risk_rules" => rules }
  end

  it "autor e revisor simulam a definição em edição; os demais 403 missing_role" do
    sign_in_as(staff_with("autor-sim@cidade.gov.br", "protocol_author"))
    simulate(definition: definition, vitals: { systolic: 120, diastolic: 80, temperature_c: "39,2" }, ciap2_code: "R05",
             profile: { age: 30, sex: "male" })
    expect(body).to eq("suggested_color" => "yellow",
                       "matched_rules" => [ { "index" => 1, "text" => "temperatura ≥ 39 ou glicemia ≥ 300" },
                                            { "index" => 2, "text" => "queixa (CIAP-2) = R05" } ],
                       "errors" => [], "warnings" => [])
    expect(ProtocolDefinition.count).to eq(0)
    sign_in_as(staff_with("admin-sim@cidade.gov.br", "municipal_admin"))
    simulate(definition: definition, vitals: {})
    expect([ response.status, body["error"] ]).to eq([ 403, "missing_role" ])
  end

  it "definição inválida, de triagem ou sinais implausíveis: 200 com errors e sem cor" do
    sign_in_as(staff_with("revisora-sim@cidade.gov.br", "protocol_reviewer"))
    simulate(definition: definition([ { "when" => { "gte" => ["outcome.score", 1] }, "color" => "red" } ]), vitals: {})
    expect(response).to have_http_status(:ok)
    expect(body["suggested_color"]).to be_nil
    expect(body["errors"].join).to include("risk_rules[0].when")
    simulate(definition: { "name" => "t", "version" => 1 }, vitals: {})
    expect(body["errors"]).to include("definition is not a screening protocol")
    simulate(definition: definition, vitals: { spo2: 20 })
    expect(body["errors"]).to eq([ "vitals: implausible_vital spo2" ])
    simulate(definition: "x", vitals: {})
    expect(body["errors"]).to eq([ Protocols::SimulateOffer::NOT_AN_OBJECT ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/requests/screenings_spec.rb spec/requests/screening_queue_and_units_spec.rb spec/requests/authoring/simulate_screening_spec.rb`
Expected: FAIL (`No route matches`).

- [ ] **Step 3: Rotas**

Em `config/routes.rb`, dentro de `scope "/attendance"`, depois de `post "attendances/:id/close", …`:

```ruby
    # Escuta inicial (ADR 0030; contratos §3). `screenings/suggest` PRECISA vir
    # antes de `screenings/:id` (a primeira rota que casa vence).
    get  "units/:id/screening_queue",   to: "screenings#queue"
    post "attendances/:id/screening",   to: "screenings#create"
    post "screenings/suggest",          to: "screenings#suggest"
    get  "screenings/:id",              to: "screenings#show"
    post "screenings/:id/abandon",      to: "screenings#abandon"
    post "screenings/:id/complete",     to: "screenings#complete"
    post "screenings/:id/reassess",     to: "screenings#reassess"
    post "ciap2/search",                to: "ciap2_codes#search"
```

- [ ] **Step 4: Controllers**

```ruby
# app/controllers/screenings_controller.rb
# Escuta inicial (ADR 0030; spec §7; contratos §3). Profissional com vínculo e
# CBO permitido (conferidos nos comandos); a fila do acolhimento também para o
# balcão. Ler a escuta deixa trilha (screening.viewed); a recepção nunca lê.
class ScreeningsController < ApplicationController
  include Authentication
  include AttendanceAccess

  wrap_parameters false

  ERROR_STATUS = {
    missing_role: :forbidden, missing_link: :forbidden, cbo_not_allowed: :forbidden,
    already_screening: :conflict, not_waiting: :conflict, screening_not_required: :conflict,
    not_in_progress: :conflict, attendance_not_waiting: :conflict, not_reassessable: :conflict,
    invalid_ciap2: :unprocessable_entity, implausible_vital: :unprocessable_entity, bp_incomplete: :unprocessable_entity,
    invalid_color: :unprocessable_entity, color_change_reason_required: :unprocessable_entity,
    invalid_destination: :unprocessable_entity, orientation_required: :unprocessable_entity,
    invalid_schedule: :unprocessable_entity, referral_required: :unprocessable_entity,
    invalid_unit: :unprocessable_entity, note_too_long: :unprocessable_entity
  }.freeze
  REVISION_KEYS = %w[ciap2_code complaint_note vitals final_color color_change_reason].freeze
  DESTINATION_KEYS = %w[orientation_note schedule referral].freeze

  before_action :require_attendance_staff, only: :queue
  before_action :require_professional, except: :queue
  before_action :set_screening, only: %i[show abandon complete reassess]

  def queue
    unit = HealthUnit.find_by(id: params[:id])
    return not_found unless unit

    render json: { items: Screenings::Queue.items(unit).map { |a| Screenings::Json.queue_item(a) } }
  end

  def create
    attendance = Attendance.find_by(id: params[:id])
    return not_found unless attendance

    respond(Screenings::Start.call(attendance: attendance, by: Current.user), status: :created)
  end

  def abandon = respond(Screenings::Abandon.call(screening: @screening, by: Current.user))

  def complete
    respond(Screenings::Complete.call(screening: @screening, revision_params: body_slice(REVISION_KEYS),
                                      destination: params[:destination], destination_params: body_slice(DESTINATION_KEYS),
                                      by: Current.user))
  end

  def reassess
    respond(Screenings::Reassess.call(screening: @screening, revision_params: body_slice(REVISION_KEYS), by: Current.user))
  end

  def show
    status = ApplicationRecord.transaction do
      Professionals::ClinicalAuthorization.check(user: Current.user, health_unit_id: @screening.attendance.health_unit_id)
    end
    return forbid(status.to_s) unless status == :ok

    DomainEvents.publish("screening.viewed", screening_id: @screening.id, user_id: Current.user.id)
    render json: Screenings::Json.screening(@screening, with_revisions: true)
  end

  # Cor sugerida sem gravar (contratos §3 e §9): o simulador da tela de escuta.
  # Pelo atendimento (não pelo cidadão): só quem tem vínculo ativo na unidade
  # dele lê o perfil do par.
  def suggest
    attendance = Attendance.find_by(id: params[:attendance_id])
    return not_found unless attendance

    status = ApplicationRecord.transaction do
      Professionals::ClinicalAuthorization.check(user: Current.user, health_unit_id: attendance.health_unit_id)
    end
    return forbid(status.to_s) unless status == :ok

    citizen = attendance.citizen

    vitals = Screenings::VitalSigns.parse(body_slice(%w[vitals])["vitals"])
    return render_failure(vitals, ERROR_STATUS) if vitals.failure?

    code = params[:ciap2_code].presence
    ciap = code && Screenings::Ciap2.find(code)
    return render json: { error: "invalid_ciap2" }, status: :unprocessable_entity if code && ciap.nil?

    suggestion = Screenings::Suggest.call(citizen: citizen, ciap2_code: ciap&.code, vitals: vitals.payload[:values],
                                          bmi: vitals.payload[:bmi])
    rules = suggestion[:protocol_definition_id] ? ProtocolDefinition.find(suggestion[:protocol_definition_id]).definition["risk_rules"] : []
    render json: {
      suggested_color: suggestion[:color],
      matched_rules: suggestion[:matched].map { |i| { index: i, text: Protocols::ConditionText.call(rules.dig(i, "when")) } },
      alerts: vitals.payload[:alerts], bmi: vitals.payload[:bmi]
    }
  end

  private

  def set_screening
    @screening = Screening.find_by(id: params[:id])
    not_found unless @screening
  end

  def respond(result, status: :ok)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: Screenings::Json.screening(result.payload[:screening].reload), status: status
  end

  def body_slice(keys) = params.to_unsafe_h.slice(*keys)

  def not_found = render(json: { error: "not_found" }, status: :not_found)
end
```

```ruby
# app/controllers/ciap2_codes_controller.rb
# Busca de CIAP-2 por nome ou código para a queixa da escuta (ADR 0030; spec
# §8; contrato §9). Release ativa da plataforma; até 20. O termo vai no corpo
# (POST), nunca na URL. Sem release ativa: 503 terminology_unavailable.
class Ciap2CodesController < ApplicationController
  include Authentication
  include AttendanceAccess

  wrap_parameters false

  before_action :require_professional

  def search
    return render(json: { error: "terminology_unavailable" }, status: :service_unavailable) unless Screenings::Ciap2.release

    items = Screenings::Ciap2.search(params[:q].is_a?(String) ? params[:q] : "")
    render json: { items: items.map { |c| { code: c.code, label: c.label } } }
  end
end
```

Em `app/controllers/attendances_controller.rb`:
- `queue_json`: acrescente `screening: Screenings::Json.queue_block(a)` ao hash.
- `call` e `call_next`: troque `render json: { attendance: attendance_json(result.payload[:attendance]) }` por `render json: { attendance: called_json(result.payload[:attendance]) }` e acrescente:

```ruby
  # ADR 0030 (contratos §4): quem chamou recebe a escuta concluída (leitura
  # auditada). Só a rota da chamada; o balcão não passa por aqui.
  def called_json(attendance)
    screening = attendance.screening
    screening = nil unless screening&.completed?
    DomainEvents.publish("screening.viewed", screening_id: screening.id, user_id: Current.user.id) if screening
    attendance_json(attendance).merge(screening: screening && Screenings::Json.screening(screening))
  end
```

Em `app/controllers/health_units_controller.rb`:
- `create` e `update`: `error = assign_address(unit) || assign_cnes(unit) || assign_screening_scope(unit)` (com `@unit` no update);
- privado novo:

```ruby
  # ADR 0030: para quem a escuta é obrigatória. Só muda quando a chave vem.
  def assign_screening_scope(unit)
    return nil unless params.key?("screening_scope")

    value = params["screening_scope"]
    return "invalid_screening_scope" unless value.is_a?(String) && HealthUnit::SCREENING_SCOPES.include?(value)

    unit.screening_scope = value
    nil
  end
```
- `unit_json`: `json = { id: unit.id, name: unit.name, kind: unit.kind, cnes: unit.cnes, screening_scope: unit.screening_scope }.merge(…)`.

- [ ] **Step 4b: Simulador do editor (contrato §9)**

Em `config/routes.rb`, no `scope "/authoring/protocols"`, depois de `simulate_offer`: `post "simulate_screening", to: "authoring/protocols#simulate_screening"`.

Em `app/controllers/authoring/protocols_controller.rb`: `before_action :require_author!, except: %i[simulate_offer simulate_screening]`, `before_action :require_simulator!, only: %i[simulate_offer simulate_screening]` e a ação:

```ruby
    # ADR 0030 (contrato §9): simula a cor da definição em edição; sempre 200.
    def simulate_screening
      render json: Protocols::SimulateScreening.call(definition: hash_param(:definition, nil), vitals: hash_param(:vitals),
                                                     ciap2_code: params[:ciap2_code].is_a?(String) ? params[:ciap2_code] : nil,
                                                     profile: hash_param(:profile))
    end
```

```ruby
# app/services/protocols/simulate_screening.rb
# Simulador do editor para o protocolo de acolhimento (ADR 0030; contrato §9):
# a cor que a definição EM EDIÇÃO sugeriria para sinais, queixa e perfil de
# exemplo. Não grava; definição ou sinais inválidos voltam em errors (200).
module Protocols
  module SimulateScreening
    NOT_SCREENING = "definition is not a screening protocol".freeze

    module_function

    def call(definition:, vitals: {}, ciap2_code: nil, profile: {})
      empty = { suggested_color: nil, matched_rules: [], errors: [], warnings: [] }
      return empty.merge(errors: [ SimulateOffer::NOT_AN_OBJECT ]) unless definition.is_a?(Hash)
      return empty.merge(errors: [ NOT_SCREENING ]) unless Validation::Screening.screening?(definition)

      errors = Validation::Schema.call(definition)
      errors = Validation::Screening.call(definition) if errors.empty?
      return empty.merge(errors: errors) if errors.any?

      parsed = Screenings::VitalSigns.parse(vitals)
      return empty.merge(errors: [ "vitals: #{parsed.reason} #{parsed.details[:field]}".strip ]) if parsed.failure?

      profile = ConditionContext.symbolize(profile)
      rules = definition["risk_rules"]
      result = Screenings::RiskSuggestion.call(
        { vitals: parsed.payload[:values], bmi: parsed.payload[:bmi], ciap2_code: ciap2_code.to_s.strip.upcase.presence },
        { age: Integer(profile[:age].to_s, 10, exception: false), sex: profile[:sex] }, rules: rules
      )
      empty.merge(suggested_color: result[:color],
                  matched_rules: result[:matched].map { |i| { index: i, text: ConditionText.call(rules[i]["when"]) } })
    end
  end
end
```

- [ ] **Step 5: Rode e veja passar (e as specs de unidade e atendimento)**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/requests/screenings_spec.rb spec/requests/screening_queue_and_units_spec.rb spec/requests/authoring spec/requests/attendances_spec.rb spec/requests/attendance_contract_spec.rb spec/requests/health_units* spec/requests/attendance_reference_units_spec.rb`
Expected: PASS. Se `attendance_contract_spec.rb` fixa as chaves do item da fila ou da unidade, acrescente `screening` / `screening_scope` à lista esperada (o contrato §4–§5 manda).

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add app/controllers/screenings_controller.rb app/controllers/ciap2_codes_controller.rb app/controllers/attendances_controller.rb app/controllers/health_units_controller.rb app/controllers/authoring/protocols_controller.rb app/services/protocols/simulate_screening.rb config/routes.rb spec/requests/screenings_spec.rb spec/requests/screening_queue_and_units_spec.rb spec/requests/authoring/simulate_screening_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: expose the initial listening routes, colour queue and unit scope

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Se `attendance_contract_spec.rb` ou alguma spec de unidade mudou no Step 5, inclua o caminho no `git add`.)

---
## Fatia 4 — Fichas LEDI da escuta (F-18.5, F-18.6)

### Task 13: `Ledi::Fichas::InitialListening` e `Ledi::Fichas::ScreeningProcedures`

**Files:**
- Create: `app/services/ledi/fichas/screening_identity.rb`, `app/services/ledi/fichas/medicoes.rb`, `app/services/ledi/fichas/initial_listening.rb`, `app/services/ledi/fichas/screening_procedures.rb`
- Modify: `app/services/ledi/ficha_types.rb`, `app/services/ledi/version.rb`, `spec/services/ledi/contract_spec.rb`
- Test: `spec/services/ledi/fichas/initial_listening_spec.rb`, `spec/services/ledi/fichas/screening_procedures_spec.rb`

**Interfaces:**
- Consumes: `Ledi::ScreeningMapping` (Task 1), `Ledi::Ficha.assert!` (módulo 16), `ScreeningRevision` (Task 3).
- Produces:
  - `Ledi::Fichas::ScreeningIdentity = Data.define(:cnes, :ine, :professional_cns, :cbo, :citizen_cpf, :birth_date, :sex, :started_at, :ended_at, :ibge_code)` (`birth_date` é `Date`; `sex` `"female"|"male"`);
  - `Ledi::Fichas::Medicoes.build(revision) -> MedicoesThrift|nil` (nil sem medida);
  - `Ledi::Fichas::InitialListening.new(identity:, revision:, destination:, source_id:)` — `type == "atendimento_individual"`, `competence`, `cnes`, `ine`, `source == { type: "Screening", id: source_id }`, `to_thrift(uuid:) -> FichaAtendimentoIndividualMasterThrift`;
  - `Ledi::Fichas::ScreeningProcedures.new(identity:, revision:, source_id:)` — `type == "procedimento"`, mesma interface, `to_thrift(uuid:) -> FichaProcedimentoMasterThrift`; `.procedures(revision) -> Array<String>` (SIGTAP das aferições feitas); `.applicable?(revision) -> bool`;
  - `Ledi::FichaTypes::REGISTRY["atendimento_individual"] == { code: 4, klass: "Br::Gov::Saude::Esusab::Ras::Atendindividual::FichaAtendimentoIndividualMasterThrift", uuid_field: :uuidFicha }`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/ledi/fichas/initial_listening_spec.rb
require "rails_helper"

# ADR 0030 (spec §5; Task 1): escuta por nível superior → Atendimento
# Individual, tipo 4 (escuta inicial), contra os IDLs 8.7.0.
RSpec.describe Ledi::Fichas::InitialListening do
  let(:started) { Time.zone.parse("2026-10-07 09:10") }
  let(:identity) do
    Ledi::Fichas::ScreeningIdentity.new(cnes: "1234567", ine: "0000123456", professional_cns: "700000000000005",
                                        cbo: "223505", citizen_cpf: "52998224725", birth_date: Date.new(1980, 5, 10),
                                        sex: "female", started_at: started, ended_at: started + 12.minutes,
                                        ibge_code: "4106902")
  end
  let(:revision) do
    ScreeningRevision.new(ciap2_code: "K86", systolic: 185, diastolic: 110, heart_rate: 92, respiratory_rate: 18,
                          temperature_c: BigDecimal("37.2"), spo2: 97, capillary_glucose: 110, glucose_moment: "fasting",
                          weight_kg: BigDecimal("80.5"), height_cm: 175, pain_score: 6, final_color: "red")
  end
  let(:ficha) { described_class.new(identity: identity, revision: revision, destination: "same_day", source_id: "f0f0f0f0-0000-4000-8000-000000000001") }
  def ms(time) = (time.to_f * 1000).to_i

  it "cumpre a interface Ledi::Ficha" do
    expect { Ledi::Ficha.assert!(ficha) }.not_to raise_error
    expect([ ficha.type, ficha.competence, ficha.cnes, ficha.ine ]).to eq([ "atendimento_individual", "202610", "1234567", "0000123456" ])
    expect(ficha.source).to eq(type: "Screening", id: "f0f0f0f0-0000-4000-8000-000000000001")
    expect(Ledi::FichaTypes.code(ficha.type)).to eq(4)
  end

  it "monta o MIAI: cabeçalho da lotação, escuta inicial, conduta pelo destino, CIAP-2 e medições; só CPF" do
    master = ficha.to_thrift(uuid: "1234567-u")
    expect([ master.uuidFicha, master.tpCdsOrigem ]).to eq([ "1234567-u", 3 ])
    lotacao = master.headerTransport.lotacaoFormPrincipal
    expect([ lotacao.profissionalCNS, lotacao.cboCodigo_2002, lotacao.cnes, lotacao.ine ])
      .to eq(%w[700000000000005 223505 1234567 0000123456])
    expect([ master.headerTransport.dataAtendimento, master.headerTransport.codigoIbgeMunicipio ]).to eq([ ms(started), "4106902" ])
    child = master.atendimentosIndividuais.sole
    expect([ child.tipoAtendimento, child.localDeAtendimento, child.turno, child.sexo ]).to eq([ 4, 1, 1, 1 ])
    expect(child.condutas).to eq([ 11 ])
    expect([ child.cpfCidadao, child.cns, child.stCidadaoNaoPossuiCpf ]).to eq([ "52998224725", nil, false ])
    expect(child.dataNascimento).to eq(ms(Date.new(1980, 5, 10).in_time_zone))
    expect([ child.dataHoraInicialAtendimento, child.dataHoraFinalAtendimento ]).to eq([ ms(started), ms(started + 12.minutes) ])
    problem = child.problemasCondicoes.sole
    expect([ problem.ciap, problem.isAvaliado ]).to eq([ "K86", true ])
    m = child.medicoes
    expect([ m.pressaoArterialSistolica, m.pressaoArterialDiastolica, m.frequenciaCardiaca, m.frequenciaRespiratoria,
             m.temperatura, m.saturacaoO2, m.glicemiaCapilar, m.tipoGlicemiaCapilar, m.peso, m.altura ])
      .to eq([ 185, 110, 92, 18, 37.2, 97, 110, 0, 80.5, 175.0 ])
    expect { master.validate }.not_to raise_error
    bytes = Ledi::Version.serialize(master)
    expect(Ledi::Version.deserialize(master.class, bytes)).to eq(master)
  end

  it "conduta por destino; turno da tarde; sem medida não há medicoes" do
    { "schedule" => 1, "oriented" => 9, "referred" => 4 }.each do |destination, code|
      other = described_class.new(identity: identity, revision: revision, destination: destination, source_id: SecureRandom.uuid)
      expect(other.to_thrift(uuid: "x").atendimentosIndividuais.sole.condutas).to eq([ code ])
    end
    afternoon = identity.with(started_at: Time.zone.parse("2026-10-07 14:00"), ended_at: Time.zone.parse("2026-10-07 14:10"))
    bare = ScreeningRevision.new(ciap2_code: "R05", final_color: "green")
    child = described_class.new(identity: afternoon, revision: bare, destination: "same_day", source_id: SecureRandom.uuid)
                           .to_thrift(uuid: "x").atendimentosIndividuais.sole
    expect([ child.turno, child.medicoes ]).to eq([ 2, nil ])
  end
end
```

```ruby
# spec/services/ledi/fichas/screening_procedures_spec.rb
require "rails_helper"

# ADR 0030 (spec §5; Task 1): escuta por técnico/auxiliar de enfermagem →
# Ficha de Procedimentos com a marca de escuta, só com aferição; o código
# 03.01.04.007-9 nunca vai na lista.
RSpec.describe Ledi::Fichas::ScreeningProcedures do
  let(:started) { Time.zone.parse("2026-10-07 19:30") }
  let(:identity) do
    Ledi::Fichas::ScreeningIdentity.new(cnes: "1234567", ine: nil, professional_cns: "700000000000005", cbo: "322205",
                                        citizen_cpf: "52998224725", birth_date: Date.new(1950, 1, 2), sex: "male",
                                        started_at: started, ended_at: started + 8.minutes, ibge_code: "4106902")
  end

  def revision(**attrs) = ScreeningRevision.new({ ciap2_code: "K86", final_color: "yellow" }.merge(attrs))

  it "procedimentos das aferições feitas, na ordem do mapeamento; sem aferição não se aplica" do
    full = revision(systolic: 150, diastolic: 95, capillary_glucose: 210, glucose_moment: "random",
                    temperature_c: BigDecimal("38"), weight_kg: BigDecimal("70"), height_cm: 160, spo2: 96, pain_score: 3)
    expect(described_class.procedures(full)).to eq(%w[0301100039 0214010015 0301100250 0101040083 0101040075])
    expect(described_class.applicable?(revision(spo2: 96, pain_score: 2))).to be(false)
    expect(described_class.applicable?(revision(systolic: 120, diastolic: 80))).to be(true)
  end

  it "monta o MIP com a marca de escuta, medições, sem o código da escuta e só CPF" do
    rev = revision(systolic: 150, diastolic: 95)
    ficha = described_class.new(identity: identity, revision: rev, source_id: "f0f0f0f0-0000-4000-8000-000000000002")
    expect { Ledi::Ficha.assert!(ficha) }.not_to raise_error
    expect([ ficha.type, ficha.ine ]).to eq([ "procedimento", nil ])
    master = ficha.to_thrift(uuid: "1234567-p")
    header = master.headerTransport
    expect([ header.profissionalCNS, header.cboCodigo_2002, header.cnes, header.ine ]).to eq([ "700000000000005", "322205", "1234567", nil ])
    child = master.atendProcedimentos.sole
    expect([ child.statusEscutaInicialOrientacao, child.procedimentos, child.localAtendimento, child.turno, child.sexo ])
      .to eq([ true, [ "0301100039" ], 1, 3, 0 ])
    expect([ child.cpfCidadao, child.cnsCidadao, child.stCidadaoNaoPossuiCpf ]).to eq([ "52998224725", nil, false ])
    expect([ child.medicoes.pressaoArterialSistolica, child.medicoes.pressaoArterialDiastolica ]).to eq([ 150, 95 ])
    expect(child.procedimentos).not_to include(Ledi::ScreeningMapping.value("screening_procedures.forbidden_procedure"))
    expect { master.validate }.not_to raise_error
  end
end
```

Em `spec/services/ledi/contract_spec.rb`, acrescente ao hash de structs:

```ruby
    [ "ras/ficha_atendimento_individual.thrift", "FichaAtendimentoIndividualMasterThrift" ] =>
      "Br::Gov::Saude::Esusab::Ras::Atendindividual::FichaAtendimentoIndividualMasterThrift",
    [ "ras/ficha_atendimento_individual.thrift", "FichaAtendimentoIndividualChildThrift" ] =>
      "Br::Gov::Saude::Esusab::Ras::Atendindividual::FichaAtendimentoIndividualChildThrift",
    [ "ras/common.thrift", "VariasLotacoesHeaderThrift" ] => "Br::Gov::Saude::Esusab::Ras::Common::VariasLotacoesHeaderThrift",
    [ "ras/common.thrift", "LotacaoHeaderThrift" ] => "Br::Gov::Saude::Esusab::Ras::Common::LotacaoHeaderThrift",
    [ "ras/common.thrift", "MedicoesThrift" ] => "Br::Gov::Saude::Esusab::Ras::Common::MedicoesThrift",
    [ "ras/common.thrift", "ProblemaCondicaoThrift" ] => "Br::Gov::Saude::Esusab::Ras::Common::ProblemaCondicaoThrift"
```

(O regex de `idl_fields` lê `1:optional string uuidProblema` sem ponto e vírgula, como os campos de `ProblemaCondicaoThrift`; se ele não casar alguma linha, a comparação de ids acusa — ajuste o regex para `;?` opcional, não o IDL.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/ledi/fichas spec/services/ledi/contract_spec.rb`
Expected: FAIL (`uninitialized constant Ledi::Fichas::InitialListening`; `Atendindividual` não carregado).

- [ ] **Step 3: Carregar o MIAI e registrar o tipo**

Em `app/services/ledi/version.rb`: `TYPES = %w[dado_transporte_types ficha_atendimento_procedimento_types ficha_atendimento_individual_types].freeze`. Em `app/services/ledi/ficha_types.rb`, no `REGISTRY`:

```ruby
      # ADR 0030 (Task 1): escuta inicial por nível superior.
      "atendimento_individual" => { code: 4, klass: "Br::Gov::Saude::Esusab::Ras::Atendindividual::FichaAtendimentoIndividualMasterThrift",
                                    uuid_field: :uuidFicha }
```

- [ ] **Step 4: As fichas**

```ruby
# app/services/ledi/fichas/screening_identity.rb
# Identificação que as fichas da escuta exigem (ADR 0030; spec §5), já
# conferida por Ledi::ScreeningFicha. Valores, não registros: a ficha é pura.
module Ledi
  module Fichas
    ScreeningIdentity = Data.define(:cnes, :ine, :professional_cns, :cbo, :citizen_cpf, :birth_date, :sex,
                                    :started_at, :ended_at, :ibge_code)
  end
end
```

```ruby
# app/services/ledi/fichas/medicoes.rb
# Sinais da revisão → MedicoesThrift (Task 1: nomes e tipos do IDL). Dor e
# IMC não têm campo no LEDI. Sem medida, nil (o campo é opcional).
module Ledi
  module Fichas
    module Medicoes
      INTEGER = %w[systolic diastolic heart_rate respiratory_rate spo2 capillary_glucose].freeze
      DOUBLE = %w[temperature_c weight_kg height_cm].freeze

      module_function

      def build(revision)
        Ledi::Version.load!
        attrs = {}
        INTEGER.each { |column| attrs[field(column)] = revision[column].to_i unless revision[column].nil? }
        DOUBLE.each { |column| attrs[field(column)] = revision[column].to_f unless revision[column].nil? }
        if revision.glucose_moment
          attrs[field("glucose_moment")] = Ledi::ScreeningMapping.glucose_code(revision.glucose_moment)
        end
        attrs.empty? ? nil : Br::Gov::Saude::Esusab::Ras::Common::MedicoesThrift.new(**attrs)
      end

      def field(column) = Ledi::ScreeningMapping.measurement_field(column).to_sym
    end
  end
end
```

```ruby
# app/services/ledi/fichas/initial_listening.rb
# Escuta inicial por nível superior como Atendimento Individual (ADR 0030;
# spec §5; Task 1): tipoAtendimento 4, local UBS, conduta pelo destino,
# queixa CIAP-2 como problema avaliado, medições, CPF do cidadão (CNS não vai
# junto: são exclusivos). Implementa Ledi::Ficha.
module Ledi
  module Fichas
    class InitialListening
      ORIGIN_THIRD_PARTY = 3
      M = Ledi::ScreeningMapping

      attr_reader :identity, :revision, :destination, :source_id

      def initialize(identity:, revision:, destination:, source_id:)
        @identity, @revision, @destination, @source_id = identity, revision, destination, source_id
      end

      def type = "atendimento_individual"
      def competence = identity.started_at.in_time_zone.strftime("%Y%m")
      def cnes = identity.cnes
      def ine = identity.ine
      def source = { type: "Screening", id: source_id }

      def to_thrift(uuid:)
        Ledi::Version.load!
        common = Br::Gov::Saude::Esusab::Ras::Common
        ai = Br::Gov::Saude::Esusab::Ras::Atendindividual
        header = common::VariasLotacoesHeaderThrift.new(
          lotacaoFormPrincipal: common::LotacaoHeaderThrift.new(profissionalCNS: identity.professional_cns,
                                                                cboCodigo_2002: identity.cbo, cnes: identity.cnes,
                                                                ine: identity.ine),
          dataAtendimento: ms(identity.started_at), codigoIbgeMunicipio: identity.ibge_code
        )
        child = ai::FichaAtendimentoIndividualChildThrift.new(
          dataNascimento: ms(identity.birth_date.in_time_zone), localDeAtendimento: M.value("initial_listening.local_de_atendimento"),
          sexo: M.sex_code(identity.sex), turno: M.turno(identity.started_at),
          tipoAtendimento: M.value("initial_listening.tipo_atendimento"), condutas: [ M.conduta(destination) ],
          dataHoraInicialAtendimento: ms(identity.started_at), dataHoraFinalAtendimento: ms(identity.ended_at),
          cpfCidadao: identity.citizen_cpf, stCidadaoNaoPossuiCpf: false, ficouEmObservacao: false,
          medicoes: Medicoes.build(revision),
          problemasCondicoes: [ common::ProblemaCondicaoThrift.new(ciap: revision.ciap2_code, isAvaliado: true) ]
        )
        ai::FichaAtendimentoIndividualMasterThrift.new(uuidFicha: uuid, tpCdsOrigem: ORIGIN_THIRD_PARTY,
                                                       headerTransport: header, atendimentosIndividuais: [ child ])
      end

      private

      def ms(time) = (time.to_f * 1000).to_i
    end
  end
end
```

```ruby
# app/services/ledi/fichas/screening_procedures.rb
# Escuta por técnico/auxiliar de enfermagem como Ficha de Procedimentos (ADR
# 0030; spec §5; Task 1): marca de escuta inicial, SIGTAP das aferições feitas
# (nunca 03.01.04.007-9) e medições. Só nasce com aferição (spec §5).
module Ledi
  module Fichas
    class ScreeningProcedures
      ORIGIN_THIRD_PARTY = 3
      M = Ledi::ScreeningMapping
      # Aferição → coluna que prova que ela foi feita.
      MEASURED = { "blood_pressure" => "systolic", "capillary_glucose" => "capillary_glucose",
                   "temperature" => "temperature_c", "weight" => "weight_kg", "height" => "height_cm" }.freeze

      def self.procedures(revision)
        MEASURED.filter_map { |kind, column| M.procedure(kind) unless revision[column].nil? }
      end

      def self.applicable?(revision) = procedures(revision).any?

      attr_reader :identity, :revision, :source_id

      def initialize(identity:, revision:, source_id:)
        @identity, @revision, @source_id = identity, revision, source_id
      end

      def type = "procedimento"
      def competence = identity.started_at.in_time_zone.strftime("%Y%m")
      def cnes = identity.cnes
      def ine = identity.ine
      def source = { type: "Screening", id: source_id }

      def to_thrift(uuid:)
        Ledi::Version.load!
        ras = Br::Gov::Saude::Esusab::Ras
        header = ras::Common::UnicaLotacaoHeaderThrift.new(
          profissionalCNS: identity.professional_cns, cboCodigo_2002: identity.cbo, cnes: identity.cnes,
          ine: identity.ine, dataAtendimento: ms(identity.started_at), codigoIbgeMunicipio: identity.ibge_code
        )
        child = ras::Atendprocedimentos::FichaProcedimentoChildThrift.new(
          dtNascimento: ms(identity.birth_date.in_time_zone), sexo: M.sex_code(identity.sex),
          localAtendimento: M.value("screening_procedures.local_atendimento"), turno: M.turno(identity.started_at),
          statusEscutaInicialOrientacao: M.value("screening_procedures.status_escuta_inicial_orientacao"),
          procedimentos: self.class.procedures(revision), dataHoraInicialAtendimento: ms(identity.started_at),
          dataHoraFinalAtendimento: ms(identity.ended_at), cpfCidadao: identity.citizen_cpf,
          stCidadaoNaoPossuiCpf: false, medicoes: Medicoes.build(revision)
        )
        ras::Atendprocedimentos::FichaProcedimentoMasterThrift.new(uuidFicha: uuid, tpCdsOrigem: ORIGIN_THIRD_PARTY,
                                                                   headerTransport: header, atendProcedimentos: [ child ])
      end

      private

      def ms(time) = (time.to_f * 1000).to_i
    end
  end
end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/ledi spec/config/ledi`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add app/services/ledi/fichas/screening_identity.rb app/services/ledi/fichas/medicoes.rb app/services/ledi/fichas/initial_listening.rb app/services/ledi/fichas/screening_procedures.rb app/services/ledi/ficha_types.rb app/services/ledi/version.rb spec/services/ledi/contract_spec.rb spec/services/ledi/fichas/initial_listening_spec.rb spec/services/ledi/fichas/screening_procedures_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: build the initial listening and screening procedures LEDI fichas

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 14: Geração da ficha — `Ledi::ScreeningFicha`, "não gerada", job do fechamento e varredor das 23h

**Files:**
- Create: `app/services/ledi/screening_ficha.rb`
- Create: `app/jobs/ledi/screening_ficha_job.rb`, `app/jobs/ledi/screening_ficha_sweep_job.rb`
- Modify: `app/commands/ledi/enqueue.rb`, `app/commands/screenings/complete.rb`, `app/commands/attendances/close.rb`, `config/recurring.yml`
- Test: `spec/services/ledi/screening_ficha_spec.rb`, `spec/jobs/ledi/screening_ficha_jobs_spec.rb`

**Interfaces:**
- Consumes: Tasks 1, 3, 4, 13; `Ledi::Enqueue.call`, `Platform::Features.usable?/settings`, `HealthTeamMember`, `Professionals::Cns.valid?`, `Citizen::SEXES`.
- Produces:
  - `Ledi::ScreeningFicha::SOURCE_TYPE == "Screening"`, `.exportable?(city) -> bool`, `.build(screening) -> [ficha|nil, Array<String>]` (motivos de `LediGenerationFailure::REASONS`), `.generate(screening, city: Current.city) -> :enqueued | :nothing | :failed | :exists | :unusable | :skipped`, `.record_failure!(screening, reasons) -> :failed`, `.resolve_failure!(screening)`;
  - `Ledi::Enqueue.call(ficha, city:, replaces: nil)` — com `replaces` (uma `LediOutboxEntry` recusada) cria sempre uma linha nova com `replaces_outbox_id`;
  - `Ledi::ScreeningFichaJob.enqueue_for(screening)`, `#perform(city_slug:, screening_id:)`;
  - `Ledi::ScreeningFichaSweepJob::SWEEP_HOUR == 23`, `#perform` (age só às 23h locais; escutas `completed` de atendimentos abertos sem linha na fila);
  - eventos `ledi.generation_failed { failure_id, source_type, source_id }`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/ledi/screening_ficha_spec.rb
require "rails_helper"

# ADR 0030 (spec §5): ficha por CBO, só com a exportação utilizável; sem
# identificação completa, "não gerada" com motivo; uma ficha por escuta.
RSpec.describe Ledi::ScreeningFicha do
  let(:city) { ledi_ready!(register_test_city!, pec_url: "https://pec.a.test", record_mode: "record", ibge_code: "4106902") }
  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }

  around { |ex| CityConnection.with(city) { ex.run } }
  before do
    ciap2_release!
    allow(Ledi::DeliverJob).to receive(:perform_later)
  end

  def screening_for(citizen, by: nurse, destination: "oriented", **revision)
    attendance = walk_in_attendance!(unit, citizen: citizen)
    started = Screenings::Start.call(attendance: attendance, by: by).payload[:screening]
    params = destination == "oriented" ? { "orientation_note" => "repouso" } : {}
    Screenings::Complete.call(screening: started, revision_params: revision_params(**revision), destination: destination,
                              destination_params: params, by: by)
    started.reload
  end

  it "nível superior: Atendimento Individual na fila, uma vez só; transporte tipo 4" do
    exportable_unit!(unit, nurse)
    screening = screening_for(screening_citizen!(1))
    expect(described_class.generate(screening)).to eq(:enqueued)
    entry = LediOutboxEntry.sole
    expect(entry).to have_attributes(source_type: "Screening", source_id: screening.id, ficha_type: "atendimento_individual",
                                     status: "pending")
    expect(Ledi::Transport.read(entry.bytes).tipoDadoSerializado).to eq(4)
    expect(described_class.generate(screening)).to eq(:exists)
    expect(LediOutboxEntry.count).to eq(1)
  end

  it "técnico com aferição: Procedimentos; técnico sem aferição: nada e sem 'não gerada'" do
    tech = screener!(unit, cbo: "322205")
    exportable_unit!(unit, tech)
    with_bp = screening_for(screening_citizen!(1), by: tech)
    expect(described_class.generate(with_bp)).to eq(:enqueued)
    expect(LediOutboxEntry.sole.ficha_type).to eq("procedimento")
    bare = screening_for(screening_citizen!(2), by: tech, vitals: { "spo2" => 97 })
    expect(described_class.generate(bare)).to eq(:nothing)
    expect(LediOutboxEntry.count).to eq(1)
    expect(LediGenerationFailure.count).to eq(0)
  end

  it "sem identificação: 'não gerada' com os motivos, evento só com ids; corrigido, gera e resolve" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    screening = screening_for(citizen)
    expect(described_class.generate(screening)).to eq(:failed)
    failure = LediGenerationFailure.sole
    expect(failure.reason_codes).to eq(%w[unit_without_cnes professional_without_team citizen_without_birth_date citizen_without_sex])
    expect(DomainEvent.where(name: "ledi.generation_failed").sole.payload)
      .to eq("failure_id" => failure.id, "source_type" => "Screening", "source_id" => screening.id)
    expect(described_class.generate(screening)).to eq(:failed)
    expect(DomainEvent.where(name: "ledi.generation_failed").count).to eq(1)

    exportable_unit!(unit, nurse)
    citizen.update!(birth_date: "1980-05-10", sex: "female", profile_source: "declared")
    expect(described_class.generate(screening)).to eq(:enqueued)
    expect(failure.reload.resolved_at).to be_present
  end

  it "CIAP-2 que não existe na release gravada → unknown_ciap2" do
    exportable_unit!(unit, nurse)
    screening = screening_for(screening_citizen!(1))
    # A terminologia da plataforma é imutável (trigger); simula a release gravada sem o código.
    allow(Ciap2Code).to receive(:exists?).and_return(false)
    expect(described_class.generate(screening)).to eq(:failed)
    expect(LediGenerationFailure.sole.reason_codes).to eq(%w[unknown_ciap2])
  end

  it "exportação desligada, record_mode off ou credencial recusada: nada nasce (nem 'não gerada')" do
    exportable_unit!(unit, nurse)
    screening = screening_for(screening_citizen!(1))
    ledi_off!(city)
    expect(described_class.generate(screening, city: city)).to eq(:unusable)
    ledi_ready!(city, pec_url: "https://pec.a.test", record_mode: "off")
    expect(described_class.generate(screening, city: city)).to eq(:unusable)
    ledi_ready!(city, pec_url: "https://pec.a.test", record_mode: "record")
    IntegrationCredential.find_by!(kind: "ledi").update!(last_check_status: "unauthorized")
    expect(described_class.generate(screening, city: city)).to eq(:unusable)
    expect([ LediOutboxEntry.count, LediGenerationFailure.count ]).to eq([ 0, 0 ])
  end

  it "escuta em curso ou abandonada não gera" do
    attendance = walk_in_attendance!(unit, citizen: screening_citizen!(1))
    started = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
    expect(described_class.generate(started)).to eq(:skipped)
  end
end
```

```ruby
# spec/jobs/ledi/screening_ficha_jobs_spec.rb
require "rails_helper"

# ADR 0030 (spec §5): a ficha nasce no fechamento do atendimento e no
# varredor das 23h (fuso da cidade) para escutas concluídas de atendimentos
# ainda abertos; nunca duas (Review Focus 4 e 5).
RSpec.describe "Jobs da ficha da escuta" do
  include ActiveSupport::Testing::TimeHelpers

  let(:city) do
    record = City.find_by(slug: TEST_CITY_A.slug) ||
             create(:city, slug: TEST_CITY_A.slug, database_url: TEST_CITY_A.database_url, time_zone: "America/Manaus")
    ledi_ready!(record, pec_url: "https://pec.a.test", record_mode: "record", ibge_code: "1302603")
  end
  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }

  around { |ex| CityConnection.with(city) { ex.run } }
  before do
    ciap2_release!
    allow(Ledi::DeliverJob).to receive(:perform_later)
    exportable_unit!(unit, nurse)
  end

  def same_day!(n)
    attendance = walk_in_attendance!(unit, citizen: screening_citizen!(n))
    started = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
    Screenings::Complete.call(screening: started, revision_params: revision_params, destination: "same_day",
                              destination_params: {}, by: nurse)
    started.reload
  end

  def enqueued_ficha_jobs
    ActiveJob::Base.queue_adapter.enqueued_jobs.select { |j| j[:job] == Ledi::ScreeningFichaJob }
  end

  def run_enqueued!
    enqueued_ficha_jobs.each { |job| Ledi::ScreeningFichaJob.perform_now(**ActiveJob::Arguments.deserialize(job[:args]).first) }
  end

  it "o destino que fecha o atendimento enfileira a ficha; o job a gera" do
    attendance = walk_in_attendance!(unit, citizen: screening_citizen!(1))
    started = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
    Screenings::Complete.call(screening: started, revision_params: revision_params, destination: "oriented",
                              destination_params: { "orientation_note" => "repouso" }, by: nurse)
    expect(enqueued_ficha_jobs.size).to eq(1)
    run_enqueued!
    expect(LediOutboxEntry.where(source_type: "Screening", source_id: started.id).count).to eq(1)
  end

  it "same_day só enfileira quando o profissional encerra o atendimento" do
    screening = same_day!(1)
    expect(enqueued_ficha_jobs).to be_empty
    doctor = screener!(unit, cbo: "225125")
    Attendances::Call.call(attendance: screening.attendance, health_unit_id: unit.id, by: doctor)
    Attendances::Close.call(attendance: screening.attendance.reload, outcome: "discharged", referral_unit_id: nil,
                            referral_note: nil, by: doctor)
    expect(enqueued_ficha_jobs.size).to eq(1)
  end

  it "varredor às 23h em Manaus (não em São Paulo); depois o fechamento não duplica (Review Focus 4 e 5)" do
    screening = same_day!(1)
    travel_to(Time.utc(2026, 10, 8, 2, 10)) do # 22h10 em Manaus, 23h10 em São Paulo
      Ledi::ScreeningFichaSweepJob.perform_now
    end
    expect(LediOutboxEntry.count).to eq(0)
    travel_to(Time.utc(2026, 10, 8, 3, 10)) do # 23h10 em Manaus
      Ledi::ScreeningFichaSweepJob.perform_now
    end
    expect(LediOutboxEntry.where(source_id: screening.id).count).to eq(1)

    doctor = screener!(unit, cbo: "225125")
    Attendances::Call.call(attendance: screening.attendance, health_unit_id: unit.id, by: doctor)
    Attendances::Close.call(attendance: screening.attendance.reload, outcome: "discharged", referral_unit_id: nil,
                            referral_note: nil, by: doctor)
    run_enqueued!
    expect(LediOutboxEntry.where(source_id: screening.id).count).to eq(1)
  end

  it "exportação desligada no fechamento: nada; ligada no varredor (atendimento ainda aberto): nasce" do
    screening = same_day!(1)
    ledi_off!(city)
    travel_to(Time.utc(2026, 10, 8, 3, 10)) { Ledi::ScreeningFichaSweepJob.perform_now }
    expect(LediOutboxEntry.count).to eq(0)
    ledi_ready!(city, pec_url: "https://pec.a.test", record_mode: "record", ibge_code: "1302603")
    travel_to(Time.utc(2026, 10, 8, 3, 20)) { Ledi::ScreeningFichaSweepJob.perform_now }
    expect(LediOutboxEntry.where(source_id: screening.id).count).to eq(1)
  end

  it "recurring.yml agenda o varredor a cada hora" do
    task = YAML.load_file(Rails.root.join("config/recurring.yml")).dig("default", "ledi_screening_ficha_sweep")
    expect(task).to include("class" => "Ledi::ScreeningFichaSweepJob", "schedule" => "every hour at minute 10")
  end
end
```

(Se `TEST_CITY_A` já tiver linha na plataforma com outro fuso neste ponto da suíte, o `let(:city)` a reaproveita: o fuso é imutável por trigger. Nesse caso troque o `travel_to` para as horas do fuso que ela tem — a asserção é "às 23h locais", não "em Manaus". Rode `City.find_by(slug: TEST_CITY_A.slug)&.time_zone` no console de teste para conferir antes.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/ledi/screening_ficha_spec.rb spec/jobs/ledi/screening_ficha_jobs_spec.rb`
Expected: FAIL (`uninitialized constant Ledi::ScreeningFicha`).

- [ ] **Step 3: `Ledi::Enqueue` com `replaces:`**

Em `app/commands/ledi/enqueue.rb`, troque `call` e `find_existing`:

```ruby
    # replaces: a recusada que esta ficha substitui (regeneração a partir da
    # origem, ADR 0030 §5). Sem ela, a mesma fonte entra uma vez.
    def call(ficha, city:, replaces: nil)
      Ledi::Ficha.assert!(ficha)
      return nil unless accepting?(city)

      source = ficha.source
      unless replaces
        existing = find_existing(source, ficha)
        return existing if existing
      end

      uuid = "#{ficha.cnes}-#{SecureRandom.uuid}"
      entry = begin
        ApplicationRecord.transaction(requires_new: true) do
          LediOutboxEntry.create!(uuid: uuid, ficha_type: ficha.type, competence: ficha.competence,
                                  source_type: source[:type], source_id: source[:id],
                                  ledi_version: Ledi::Version::ACTIVE, next_attempt_at: Time.current,
                                  replaces_outbox_id: replaces&.id,
                                  bytes: Ledi::Transport.wrap(ficha, city: city, uuid: uuid))
        end
      rescue ActiveRecord::RecordNotUnique
        return find_existing(source, ficha) || raise
      end
      Ledi::DeliverJob.perform_later
      entry
    end
```

```ruby
    # A mais recente da fonte (com recusadas regeneradas há mais de uma).
    def find_existing(source, ficha)
      LediOutboxEntry.where(source_type: source[:type], source_id: source[:id], ficha_type: ficha.type)
                     .order(created_at: :desc, id: :desc).first
    end
```

- [ ] **Step 4: `Ledi::ScreeningFicha`**

```ruby
# app/services/ledi/screening_ficha.rb
# A ficha da escuta (ADR 0030; spec §5). Nasce só de escuta concluída, só com
# a exportação utilizável e record_mode diferente de off, e uma vez por
# escuta (qualquer tipo de ficha). Sem identificação completa, não nasce:
# fica em ledi_generation_failures com os motivos (lista fechada), e a próxima
# tentativa que der certo resolve. CBO de nível superior (tabela do MIAI) →
# Atendimento Individual; técnico/auxiliar de enfermagem → Procedimentos,
# só com aferição.
module Ledi
  module ScreeningFicha
    SOURCE_TYPE = "Screening".freeze

    module_function

    def exportable?(city)
      Platform::Features.usable?(city, :ledi_export) && Platform::Features.settings(city)[:record_mode] != "off"
    end

    def generate(screening, city: Current.city)
      return :skipped unless screening.completed?
      return :unusable unless exportable?(city)
      return :exists if LediOutboxEntry.exists?(source_type: SOURCE_TYPE, source_id: screening.id)

      ficha, reasons = build(screening)
      return record_failure!(screening, reasons) if reasons.any?

      resolve_failure!(screening)
      return :nothing unless ficha

      Ledi::Enqueue.call(ficha, city: city) ? :enqueued : :unusable
    end

    def build(screening)
      revision = screening.current_revision
      attendance = screening.attendance
      unit = attendance.health_unit
      professional = screening.professional_link.professional
      citizen = attendance.citizen
      miai = Ledi::ScreeningMapping.miai_cbo?(screening.cbo_code)
      ine = team_ine(professional, unit)
      birth = birth_date(citizen)

      reasons = []
      reasons << "unit_without_cnes" if unit.cnes.blank?
      reasons << "professional_without_team" if ine.nil?
      reasons << "professional_without_cns" unless Professionals::Cns.valid?(professional.cns)
      reasons << "citizen_without_birth_date" if birth.nil?
      reasons << "citizen_without_sex" unless Citizen::SEXES.include?(citizen.sex)
      reasons << "unknown_ciap2" if miai && !Ciap2Code.exists?(release_id: revision.ciap2_release_id, code: revision.ciap2_code)
      return [ nil, reasons ] if reasons.any?

      identity = Ledi::Fichas::ScreeningIdentity.new(
        cnes: unit.cnes, ine: ine, professional_cns: professional.cns, cbo: screening.cbo_code,
        citizen_cpf: citizen.cpf, birth_date: birth, sex: citizen.sex, started_at: screening.started_at,
        ended_at: [ revision.created_at, screening.started_at ].max, ibge_code: CityProfile.current&.ibge_code
      )
      ficha = if miai
                Ledi::Fichas::InitialListening.new(identity: identity, revision: revision,
                                                   destination: screening.destination, source_id: screening.id)
              elsif Ledi::Fichas::ScreeningProcedures.applicable?(revision)
                Ledi::Fichas::ScreeningProcedures.new(identity: identity, revision: revision, source_id: screening.id)
              end
      [ ficha, [] ]
    end

    def record_failure!(screening, reasons)
      failure = LediGenerationFailure.unresolved.find_by(source_type: SOURCE_TYPE, source_id: screening.id)
      if failure
        failure.update!(reason_codes: reasons) unless failure.reason_codes == reasons
      else
        failure = ApplicationRecord.transaction(requires_new: true) do
          LediGenerationFailure.create!(source_type: SOURCE_TYPE, source_id: screening.id, reason_codes: reasons)
        end
        DomainEvents.publish("ledi.generation_failed", failure_id: failure.id, source_type: SOURCE_TYPE,
                                                       source_id: screening.id)
      end
      :failed
    rescue ActiveRecord::RecordNotUnique
      :failed # outra execução (fechamento × varredor) registrou no mesmo instante
    end

    def resolve_failure!(screening)
      LediGenerationFailure.unresolved.where(source_type: SOURCE_TYPE, source_id: screening.id)
                           .update_all(resolved_at: Time.current, updated_at: Time.current)
    end

    # INE da equipe ativa da unidade em que o profissional está (eSF/eAP).
    def team_ine(professional, unit)
      HealthTeamMember.active.joins(:health_team)
                      .where(professional_id: professional.id, health_teams: { health_unit_id: unit.id, active: true })
                      .order(:started_on, :id).pick("health_teams.ine")
    end

    def birth_date(citizen)
      citizen.birth_date.present? ? Date.iso8601(citizen.birth_date) : nil
    rescue Date::Error
      nil
    end
  end
end
```

- [ ] **Step 5: Os jobs e os ganchos**

```ruby
# app/jobs/ledi/screening_ficha_job.rb
# Ficha da escuta no fechamento do atendimento (ADR 0030; spec §5). Leva só
# ids; a decisão (exportável, identificação, uma vez só) é do
# Ledi::ScreeningFicha, relida na hora em que roda.
module Ledi
  class ScreeningFichaJob < ApplicationJob
    include CityScopedJob
    queue_as :default

    def self.enqueue_for(screening)
      perform_later(city_slug: Current.city.slug, screening_id: screening.id)
    end

    def perform(city_slug:, screening_id:)
      with_city(city_slug) do
        screening = Screening.find_by(id: screening_id)
        Ledi::ScreeningFicha.generate(screening, city: Current.city) if screening
      end
    end
  end
end
```

```ruby
# app/jobs/ledi/screening_ficha_sweep_job.rb
# Varredor diário (ADR 0030; spec §5): às 23h no fuso da cidade, gera a ficha
# das escutas concluídas cujos atendimentos ainda estão abertos (a pessoa
# esperou o dia todo, ou ninguém encerrou). Agendado a cada hora; só age na
# hora 23 local (CityConnection.with usa o fuso da cidade).
module Ledi
  class ScreeningFichaSweepJob < ApplicationJob
    prepend EachCityJob
    queue_as :housekeeping

    SWEEP_HOUR = 23

    def perform
      return unless Time.current.hour == SWEEP_HOUR

      Screening.completed_screenings.joins(:attendance).merge(Attendance.open_attendances)
               .where.not(id: LediOutboxEntry.where(source_type: Ledi::ScreeningFicha::SOURCE_TYPE).select(:source_id))
               .find_each { |screening| Ledi::ScreeningFicha.generate(screening, city: Current.city) }
    end
  end
end
```

Em `app/commands/screenings/complete.rb`, logo depois de `DomainEvents.publish("screening.completed", …)`:

```ruby
        # ADR 0030 (spec §5): a ficha nasce no fechamento (o job relê tudo).
        Ledi::ScreeningFichaJob.enqueue_for(screening) unless destination == "same_day"
```

Em `app/commands/attendances/close.rb`, logo depois de `DomainEvents.publish("attendance.closed", …)`:

```ruby
        # ADR 0030 (spec §5): escuta concluída (same_day) gera a ficha no fechamento.
        Ledi::ScreeningFichaJob.enqueue_for(attendance.screening) if attendance.screening&.completed?
```

Em `config/recurring.yml`, depois de `ledi_purge_stale_payloads`:

```yaml
  ledi_screening_ficha_sweep:
    class: Ledi::ScreeningFichaSweepJob
    queue: housekeeping
    schedule: "every hour at minute 10"
```

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/services/ledi spec/jobs/ledi spec/commands/ledi spec/commands/screenings spec/commands/attendances`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add app/services/ledi/screening_ficha.rb app/jobs/ledi/screening_ficha_job.rb app/jobs/ledi/screening_ficha_sweep_job.rb app/commands/ledi/enqueue.rb app/commands/screenings/complete.rb app/commands/attendances/close.rb config/recurring.yml spec/services/ledi/screening_ficha_spec.rb spec/jobs/ledi/screening_ficha_jobs_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: generate the screening ficha on close and in the nightly sweep

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 15: Produção — "fichas não geradas", "gerar de novo" e a recusada regenerada da origem

**Files:**
- Modify: `app/services/ledi/screening_ficha.rb`, `app/commands/ledi/resend.rb`, `app/controllers/production_controller.rb`, `config/routes.rb`
- Test: `spec/requests/production_generation_failures_spec.rb`, `spec/commands/ledi/resend_screening_spec.rb`

**Interfaces:**
- Consumes: `Ledi::ScreeningFicha.generate/build/record_failure!/resolve_failure!/exportable?` (Task 14), `Ledi::Enqueue.call(…, replaces:)`.
- Produces:
  - `Ledi::ScreeningFicha.retry!(failure, by:) -> LediGenerationFailure` (levanta `Ledi::ScreeningFicha::AlreadyResolved`); publica `ledi.generation_retried`;
  - `Ledi::ScreeningFicha.regenerate(entry, by:, city: Current.city) -> [Symbol, LediOutboxEntry|nil]` — `:ok`, `:not_rejected`, `:export_unusable`, `:generation_failed`;
  - `Ledi::Resend::ExportUnusable`, `Ledi::Resend::NotRegenerated` (novos); `Ledi::Resend.call` regenera fontes `Screening`;
  - `GET /production/generation_failures?resolved=false|true` → `{ items: [ { id, source_type, source_id, attendance_id, reason_codes, created_at, resolved_at } ] }` (até 200, mais novas primeiro; sem o parâmetro = não resolvidas);
  - `POST /production/generation_failures/:id/retry` (step-up) → 200 com o item; 409 `already_resolved`; 404;
  - `POST /production/fichas/:id/resend` para ficha de escuta → 200 com a ficha **nova**; 409 `not_rejected` (inclusive já regenerada), `export_unusable`, `generation_failed` (Divergência D5).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/requests/production_generation_failures_spec.rb
require "rails_helper"

# Contratos §6 (ADR 0030; spec §5): fichas que não puderam ser geradas e
# "gerar de novo" (step-up, só municipal_admin).
RSpec.describe "Produção — fichas não geradas", type: :request do
  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }
  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  let(:admin) do
    staff_with("prod-admin@cidade.gov.br", "municipal_admin").tap do |u|
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
    end
  end
  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]

  before do
    Current.city = TEST_CITY_A
    ledi_ready!(city, pec_url: "https://pec.a.test", record_mode: "record")
    allow(Ledi::DeliverJob).to receive(:perform_later)
    ciap2_release!
  end
  after { Current.reset }

  def failed_screening!
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    attendance = walk_in_attendance!(unit, citizen: citizen)
    started = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
    Screenings::Complete.call(screening: started, revision_params: revision_params, destination: "oriented",
                              destination_params: { "orientation_note" => "repouso" }, by: nurse)
    Ledi::ScreeningFicha.generate(started.reload, city: city)
    [ started, citizen ]
  end

  it "lista as não resolvidas com o atendimento e os motivos; resolved=true lista as resolvidas" do
    screening, = failed_screening!
    sign_in_as(admin)
    get "/production/generation_failures", params: { resolved: "false" }
    expect(response).to have_http_status(:ok)
    item = body["items"].sole
    expect(item.keys).to match_array(%w[id source_type source_id attendance_id reason_codes created_at resolved_at])
    expect(item.values_at("source_type", "source_id", "attendance_id", "resolved_at"))
      .to eq([ "Screening", screening.id, screening.attendance_id, nil ])
    expect(item["reason_codes"]).to include("unit_without_cnes", "citizen_without_sex")
    get "/production/generation_failures"
    expect(body["items"].size).to eq(1)
    get "/production/generation_failures", params: { resolved: "true" }
    expect(body["items"]).to eq([])
  end

  it "gerar de novo: step-up; ainda faltando atualiza os motivos; corrigido resolve e a ficha nasce; de novo 409" do
    screening, citizen = failed_screening!
    failure = LediGenerationFailure.sole
    sign_in_as(admin)
    json_post "/production/generation_failures/#{failure.id}/retry"
    expect(status_and_error).to eq([ 401, "mfa_required" ])

    sign_in_as(admin).update!(mfa_verified_at: Time.current)
    exportable_unit!(unit, nurse)
    json_post "/production/generation_failures/#{failure.id}/retry"
    expect(response).to have_http_status(:ok)
    expect(body.values_at("reason_codes", "resolved_at")).to eq([ %w[citizen_without_birth_date citizen_without_sex], nil ])

    citizen.update!(birth_date: "1980-05-10", sex: "female", profile_source: "declared")
    json_post "/production/generation_failures/#{failure.id}/retry"
    expect(body["resolved_at"]).to be_present
    expect(LediOutboxEntry.where(source_id: screening.id).count).to eq(1)
    expect(DomainEvent.where(name: "ledi.generation_retried").count).to eq(2)
    json_post "/production/generation_failures/#{failure.id}/retry"
    expect(status_and_error).to eq([ 409, "already_resolved" ])
    json_post "/production/generation_failures/#{SecureRandom.uuid}/retry"
    expect(status_and_error).to eq([ 404, "not_found" ])
  end

  it "analyst lê e não tenta de novo; viewer 403; interruptor desligado 403" do
    failed_screening!
    sign_in_as(staff_with("analista@cidade.gov.br", "analyst")).update!(mfa_verified_at: Time.current)
    get "/production/generation_failures"
    expect(response).to have_http_status(:ok)
    json_post "/production/generation_failures/#{LediGenerationFailure.sole.id}/retry"
    expect(status_and_error).to eq([ 403, "missing_role" ])
    sign_in_as(staff_with("viewer@cidade.gov.br", "viewer"))
    get "/production/generation_failures"
    expect(status_and_error).to eq([ 403, "missing_role" ])
    ledi_off!(city)
    sign_in_as(admin)
    get "/production/generation_failures"
    expect(status_and_error).to eq([ 403, "feature_disabled" ])
  end
end
```

```ruby
# spec/commands/ledi/resend_screening_spec.rb
require "rails_helper"

# ADR 0030 (spec §5): ficha de escuta recusada e corrigida na origem é regerada
# — linha nova com outro uuid e replaces_outbox_id; a antiga fica recusada.
# Review Focus 5: regenerar duas vezes a mesma recusada → not_rejected.
RSpec.describe Ledi::Resend, "fonte Screening" do
  let(:city) { ledi_ready!(register_test_city!, pec_url: "https://pec.a.test", record_mode: "record") }
  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  let(:by) { ledi_admin! }

  around { |ex| CityConnection.with(city) { ex.run } }
  before do
    ciap2_release!
    allow(Ledi::DeliverJob).to receive(:perform_later)
    exportable_unit!(unit, nurse)
  end

  let(:rejected) do
    attendance = walk_in_attendance!(unit, citizen: screening_citizen!(1))
    started = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
    Screenings::Complete.call(screening: started, revision_params: revision_params, destination: "oriented",
                              destination_params: { "orientation_note" => "repouso" }, by: nurse)
    Ledi::ScreeningFicha.generate(started.reload, city: city)
    LediOutboxEntry.sole.tap { |e| e.reject!([ { "field" => "cpfCidadao", "code" => "invalid" } ]) }
  end

  it "regera da origem: nova pendente com outro uuid e o vínculo; a recusada fica; de novo → not_rejected" do
    fresh = described_class.call(entry: rejected, by: by)
    expect(fresh.id).not_to eq(rejected.id)
    expect(fresh).to have_attributes(status: "pending", replaces_outbox_id: rejected.id, source_id: rejected.source_id)
    expect(fresh.uuid).not_to eq(rejected.uuid)
    expect(rejected.reload.status).to eq("rejected")
    expect { described_class.call(entry: rejected.reload, by: by) }.to raise_error(described_class::NotRejected)
    expect(LediOutboxEntry.where(source_id: rejected.source_id).count).to eq(2)
  end

  it "identificação quebrada desde então: 'não gerada' registrada e NotRegenerated; exportação inutilizável: ExportUnusable" do
    entry = rejected
    unit.update!(cnes: nil)
    expect { described_class.call(entry: entry, by: by) }.to raise_error(described_class::NotRegenerated)
    expect(LediGenerationFailure.unresolved.sole.reason_codes).to eq(%w[unit_without_cnes])
    unit.update!(cnes: "1234567")
    ledi_off!(city)
    expect { described_class.call(entry: entry.reload, by: by) }.to raise_error(described_class::ExportUnusable)
    expect(entry.reload.status).to eq("rejected")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/requests/production_generation_failures_spec.rb spec/commands/ledi/resend_screening_spec.rb`
Expected: FAIL (`No route matches`; `Ledi::Resend` reembrulha em vez de regerar).

- [ ] **Step 3: "Gerar de novo" e regenerar**

Em `app/services/ledi/screening_ficha.rb`, depois de `SOURCE_TYPE`:

```ruby
    class AlreadyResolved < StandardError; end
```

e, antes de `team_ine`:

```ruby
    # "Gerar de novo" (contratos §6): tenta agora; sucesso (ou ficha que já
    # existe) resolve; falta de identificação atualiza os motivos;
    # exportação inutilizável deixa como está.
    def retry!(failure, by:)
      raise AlreadyResolved if failure.resolved?

      screening = Screening.find_by(id: failure.source_id)
      outcome = screening ? generate(screening) : :skipped
      resolve_failure!(screening) if outcome == :exists
      DomainEvents.publish("ledi.generation_retried", failure_id: failure.id, source_type: failure.source_type,
                                                      source_id: failure.source_id)
      failure.reload
    end

    # Recusada corrigida na origem (spec §5): monta a ficha de novo da escuta,
    # com uuid novo e replaces_outbox_id; a linha antiga fica recusada. Não
    # levanta dentro da transação: a "não gerada" registrada aqui precisa ficar.
    def regenerate(entry, by:, city: Current.city)
      ApplicationRecord.transaction do
        entry.lock!
        next [ :not_rejected, nil ] unless entry.status == "rejected"
        next [ :not_rejected, nil ] if LediOutboxEntry.exists?(replaces_outbox_id: entry.id)
        next [ :export_unusable, nil ] unless exportable?(city)

        screening = Screening.find(entry.source_id)
        ficha, reasons = build(screening)
        if reasons.any?
          record_failure!(screening, reasons)
          next [ :generation_failed, nil ]
        end
        next [ :generation_failed, nil ] unless ficha

        fresh = Ledi::Enqueue.call(ficha, city: city, replaces: entry)
        next [ :export_unusable, nil ] unless fresh

        DomainEvents.publish("ledi.ficha_resent", outbox_id: entry.id, user_id: by.id)
        [ :ok, fresh ]
      end
    end
```

Em `app/commands/ledi/resend.rb`:

```ruby
    class NotRejected < StandardError; end
    class ExportUnusable < StandardError; end
    class NotRegenerated < StandardError; end

    module_function

    def call(entry:, by:)
      return regenerate(entry, by) if entry.source_type == Ledi::ScreeningFicha::SOURCE_TYPE

      # … corpo atual, sem mudança …
    end

    # ADR 0030 (spec §5): ficha de escuta não reaproveita o conteúdo antigo.
    def regenerate(entry, by)
      status, fresh = Ledi::ScreeningFicha.regenerate(entry, by: by)
      case status
      when :ok then fresh
      when :not_rejected then raise NotRejected
      when :export_unusable then raise ExportUnusable
      else raise NotRegenerated
      end
    end
```

(O corpo atual de `call` — `entry.with_lock …` até `entry` — continua igual, só depois do `return regenerate(…)`.)

- [ ] **Step 4: Rotas e controller**

Em `config/routes.rb`, junto de `/production`:

```ruby
  # Fichas que não puderam ser geradas (ADR 0030; contratos §6).
  get  "/production/generation_failures",           to: "production#generation_failures"
  post "/production/generation_failures/:id/retry", to: "production#retry_generation"
```

Em `app/controllers/production_controller.rb`:

```ruby
  FAILURES_LIMIT = 200

  before_action :require_read, only: %i[show generation_failures]
  before_action :require_resend, only: %i[resend retry_generation]
```
(substituindo os dois `before_action` atuais), em `resend` acrescente aos `rescue`:

```ruby
  rescue Ledi::Resend::ExportUnusable
    render json: { error: "export_unusable" }, status: :conflict
  rescue Ledi::Resend::NotRegenerated
    render json: { error: "generation_failed" }, status: :conflict
```

e as ações:

```ruby
  def generation_failures
    resolved = optional_scalar_param(:resolved).to_s
    scope = resolved == "true" ? LediGenerationFailure.where.not(resolved_at: nil) : LediGenerationFailure.unresolved
    failures = scope.order(created_at: :desc, id: :desc).limit(FAILURES_LIMIT).to_a
    attendances = Screening.where(id: failures.select { |f| f.source_type == "Screening" }.map(&:source_id))
                           .pluck(:id, :attendance_id).to_h
    render json: { items: failures.map { |f| failure_json(f, attendances[f.source_id]) } }
  end

  def retry_generation
    return require_step_up! unless reauthenticated_recently?

    failure = LediGenerationFailure.find_by(id: params[:id])
    return render(json: { error: "not_found" }, status: :not_found) unless failure

    failure = Ledi::ScreeningFicha.retry!(failure, by: Current.user)
    render json: failure_json(failure, Screening.where(id: failure.source_id).pick(:attendance_id))
  rescue Ledi::ScreeningFicha::AlreadyResolved
    render json: { error: "already_resolved" }, status: :conflict
  end
```

e, entre os privados:

```ruby
  def failure_json(failure, attendance_id)
    { id: failure.id, source_type: failure.source_type, source_id: failure.source_id, attendance_id: attendance_id,
      reason_codes: failure.reason_codes, created_at: failure.created_at.iso8601, resolved_at: failure.resolved_at&.iso8601 }
  end
```

- [ ] **Step 5: Rode e veja passar (e a produção de antes)**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/requests/production_generation_failures_spec.rb spec/commands/ledi spec/requests/production_spec.rb spec/services/ledi`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add app/services/ledi/screening_ficha.rb app/commands/ledi/resend.rb app/controllers/production_controller.rb config/routes.rb spec/requests/production_generation_failures_spec.rb spec/commands/ledi/resend_screening_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "feat: list fichas that could not be generated and regenerate rejected screening fichas

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 5 — Invariantes, semente e fechamento (F-18.7, F-18.8)

### Task 16: Suíte de invariantes do ADR 0030

**Files:**
- Create: `spec/invariants/screening_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 3–15; `ledi_ready!`, `stub_pec!`, `FakePec` (módulo 16).

- [ ] **Step 1: Escreva a spec (cada bloco diz a mutação que precisa deixá-lo vermelho)**

```ruby
# spec/invariants/screening_invariants_spec.rb
require "rails_helper"

# Módulo 18, critério de fechamento (ADR 0030, "Invariantes"). Cada bloco tem
# a mutação que precisa deixá-lo vermelho.
RSpec.describe "Invariantes do acolhimento (ADR 0030)", type: :request do
  before { Current.city = TEST_CITY_A; ciap2_release!; allow(Ledi::DeliverJob).to receive(:perform_later) }
  after { Current.reset }

  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }
  let(:unit) { create_unit }
  let(:nurse) { screener!(unit) }
  let(:marker) { "MARCADOR-#{SecureRandom.hex(4)}" }
  def body = JSON.parse(response.body)

  def capture_log
    log = StringIO.new
    capture = ActiveSupport::Logger.new(log).tap { |l| l.level = Logger::DEBUG }
    Rails.logger.broadcast_to(capture)
    yield
    log.string
  ensure
    Rails.logger.stop_broadcasting_to(capture)
  end

  def completed!(attendance, destination: "same_day", params: {}, **revision)
    started = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
    Screenings::Complete.call(screening: started, revision_params: revision_params(**revision), destination: destination,
                              destination_params: params, by: nurse)
    started.reload
  end

  # Mutação: tirar o trigger attendances_screening_close_guard (ou a condição
  # de destino dele) de db/city_triggers.sql.
  it "atendimento só fecha de waiting com desfecho de escuta se houver escuta concluída com aquele destino" do
    attendance = walk_in_attendance!(unit, citizen: screening_citizen!(1))
    close = lambda do |outcome, note = nil|
      ApplicationRecord.transaction(requires_new: true) do
        attendance.reload.update_columns(status: "closed", outcome: outcome, closed_by_user_id: nurse.id,
                                         closed_at: Time.current, referral_note: note)
      end
    end
    %w[scheduled_from_screening oriented].each do |outcome|
      expect { close.call(outcome) }.to raise_error(ActiveRecord::StatementInvalid, /requires a completed screening/)
    end
    expect { close.call("referred", "CAPS") }.to raise_error(ActiveRecord::StatementInvalid, /requires a completed screening/)
    completed!(attendance)
    expect { close.call("oriented") }.to raise_error(ActiveRecord::StatementInvalid, /requires a completed screening/)
  end

  # Mutação: tirar :note/:reason/:vitals de filter_parameters, ou pôr
  # complaint_note/orientation_note no payload de screening.completed.
  it "nenhum texto livre da escuta em evento, log ou Analytics" do
    acolhimento!
    attendance = walk_in_attendance!(unit, citizen: screening_citizen!(1))
    log = capture_log do
      sign_in_as(nurse)
      json_post "/attendance/attendances/#{attendance.id}/screening"
      id = body["id"]
      json_post "/attendance/screenings/#{id}/complete",
                revision_params(complaint_note: "queixa #{marker}", final_color: "yellow",
                                vitals: { systolic: 185, diastolic: 110 },
                                color_change_reason: "justificativa #{marker}")
                  .merge(destination: "oriented", orientation_note: "orientação #{marker}")
      expect(response).to have_http_status(:ok)
    end
    surfaces = [ log, DomainEvent.pluck(:payload).to_json, ActiveJob::Base.queue_adapter.enqueued_jobs.to_json ]
    surfaces.each { |text| expect(text).not_to include(marker) }
    analytics = Dir[Rails.root.join("app/services/analytics/**/*.rb")].map { |f| File.read(f) }.join
    expect(analytics).not_to match(/complaint_note|color_change_reason|orientation_note|screening_revisions/)
  end

  # Mutação: devolver queixa ou sinais em Screenings::Json.queue_block, ou
  # trocar require_professional por require_attendance_staff no show.
  it "a recepção nunca recebe queixa nem sinais vitais" do
    attendance = walk_in_attendance!(unit, citizen: screening_citizen!(1))
    screening = completed!(attendance, complaint_note: "queixa #{marker}", vitals: { systolic: 150, diastolic: 95 })
    sign_in_as(reception!)
    responses = []
    get "/attendance/units/#{unit.id}/queue"
    responses << response.body
    get "/attendance/units/#{unit.id}/screening_queue"
    responses << response.body
    get "/attendance/screenings/#{screening.id}"
    expect(response).to have_http_status(:forbidden)
    responses << response.body
    responses.each do |text|
      expect(text).not_to include(marker, "systolic", "vitals", "complaint", "ciap2")
    end
  end

  # Mutação: gravar Ledi::Outcome/corpo do 400 em qualquer coluna, ou um
  # Rails.logger com o corpo em Ledi::Delivery.
  it "nenhuma resposta crua do PEC é persistida nem logada; last_error_codes nunca contém valor" do
    stub_pec!
    allow(Ledi::Observations).to receive(:duplicate_marker).and_return(nil)
    allow(Ledi::DeliverJob).to receive(:perform_later).and_call_original
    ledi_ready!(city, pec_url: "https://pec.a.test", record_mode: "record")
    exportable_unit!(unit, nurse)
    screening = completed!(walk_in_attendance!(unit, citizen: screening_citizen!(1)), destination: "oriented",
                           params: { "orientation_note" => "repouso" })
    Ledi::ScreeningFicha.generate(screening, city: city)
    FakePec.for("https://pec.a.test").delivery_replies = [
      [ 400, { descricaoErro: "MARIA #{marker} nascida em 10/05/1980",
               errosValidacao: { cpfCidadao: "CPF 529.982.247-25 de MARIA #{marker} inválido" } }.to_json ]
    ]
    log = capture_log { Ledi::DeliverJob.perform_now }
    entry = LediOutboxEntry.sole
    expect(entry.status).to eq("rejected")
    expect(Ledi::ErrorCodes.valid?(entry.last_error_codes)).to be(true)
    stored = ApplicationRecord.connection.select_all("SELECT * FROM ledi_outbox").to_a.to_json
    [ stored, log, DomainEvent.pluck(:payload).to_json ].each do |text|
      expect(text).not_to include(marker, "MARIA", "529.982.247-25", "10/05/1980")
    end
  end

  # Mutação: trocar 90 por 900 em Ledi::PurgeStalePayloadsJob ou tirar
  # "failed" do where.
  it "nenhum payload de recusada/falha com última tentativa há mais de 90 dias depois da purga" do
    %w[rejected failed].each do |status|
      LediOutboxEntry.create!(uuid: "1234567-#{SecureRandom.uuid}", ficha_type: "procedimento", competence: "202606",
                              source_type: "synthetic", source_id: SecureRandom.uuid, ledi_version: "8.7.0",
                              status: status, next_attempt_at: Time.current, attempts: 1, bytes: "x".b,
                              last_attempted_at: 91.days.ago,
                              last_error_codes: [ { "field" => "cnes", "code" => "invalid" } ])
    end
    CityConnection.with(city) { Ledi::PurgeStalePayloadsJob.perform_now }
    stale = LediOutboxEntry.where(status: %w[rejected failed]).where.not(payload: nil)
                           .where("COALESCE(last_attempted_at, created_at) < ?", 90.days.ago)
    expect(stale).to be_empty
  end

  # Mutação: tirar o `return :unusable unless exportable?(city)` de
  # Ledi::ScreeningFicha.generate.
  it "a ficha da escuta nunca nasce com a exportação inutilizável" do
    exportable_unit!(unit, nurse)
    ledi_ready!(city, pec_url: "https://pec.a.test", record_mode: "record")
    IntegrationCredential.find_by!(kind: "ledi").update!(last_check_status: "unauthorized")
    first = completed!(walk_in_attendance!(unit, citizen: screening_citizen!(1)), destination: "oriented",
                       params: { "orientation_note" => "repouso" })
    Ledi::ScreeningFichaJob.perform_now(city_slug: city.slug, screening_id: first.id) # credencial recusada
    ledi_off!(city)
    second = completed!(walk_in_attendance!(unit, citizen: screening_citizen!(2)), destination: "oriented",
                        params: { "orientation_note" => "repouso" })
    Ledi::ScreeningFichaJob.perform_now(city_slug: city.slug, screening_id: second.id) # interruptor desligado
    expect(LediOutboxEntry.where(source_type: "Screening")).to be_empty
    expect(LediGenerationFailure.count).to eq(0)
  end
end
```

- [ ] **Step 2: Rode e confira as mutações**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/invariants/screening_invariants_spec.rb`
Expected: PASS. Aplique cada mutação descrita no comentário (uma por vez, sem commitar), rode o bloco e veja vermelho; desfaça.

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add spec/invariants/screening_invariants_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "test: pin the ADR 0030 invariants for the initial listening

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 17: Semente de dev (spec §10)

**Files:**
- Create: `lib/screening_crew.rb`
- Modify: `db/seeds.rb`
- Test: `spec/lib/screening_crew_spec.rb`

**Interfaces:**
- Consumes: `TriageCatalogCrew.run_cycle!(slug, name, version, definition)` (público), `Protocols::SaveDraft`, `Professionals::OpenLink`, `Citizens::RegisterPerson/StartConversation/SubmitAnswer/IssueCheckInCode`, `Attendances::CheckIn`, `ProfessionalCrew::UNITS`.
- Produces: `ScreeningCrew::RULES` (regras iniciais CAB 28), `ScreeningCrew.seed_current_city(slug:, ddd:) -> { protocol: String, technician_link: String, walk_ins: Integer }`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/lib/screening_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/screening_crew").to_s

# Spec §10: em Curitiba o acolhimento nasce assinado e ativo; a técnica de
# enfermagem ganha vínculo na UBS da semente; a unidade fica em walk_in; dois
# cidadãos de demanda espontânea com check-in. Idempotente.
RSpec.describe ScreeningCrew do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  it "as regras iniciais passam no gate da variante" do
    definition = { "name" => "acolhimento", "version" => 1, "kind" => "screening", "risk_rules" => described_class::RULES }
    expect(Protocols::Gate.call(definition).errors).to eq([])
    red = Screenings::RiskSuggestion.call({ vitals: { "systolic" => 185, "diastolic" => 110 }, bmi: nil, ciap2_code: "K86" },
                                          { age: 50, sex: "female" }, rules: described_class::RULES)
    expect(red[:color]).to eq("red")
  end
end
```

(A semente inteira roda no `db:seed` de dev, que depende do elenco do `SignatureCrew`, do `ProfessionalCrew` e do catálogo; o teste fixa só o que pode quebrar em silêncio — as regras —, e o Step 4 roda a semente de verdade.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/lib/screening_crew_spec.rb`
Expected: FAIL (`cannot load such file … lib/screening_crew`).

- [ ] **Step 3: A semente**

```ruby
# lib/screening_crew.rb
require_relative "triage_catalog_crew"

# Semente de dev do módulo 18 (spec 2026-10-07 §10). Dev é fictício mas imita
# o real: regras iniciais baseadas no CAB 28 (ponto de partida; a cidade
# revisa e assina), em Curitiba assinadas de verdade pelo elenco do
# SignatureCrew e ativas; nas demais, só o rascunho. A UBS da semente fica em
# walk_in; a técnica de enfermagem (tecnico@, CBO 322205) ganha vínculo nela
# (a enfermeira já tem); dois cidadãos de demanda espontânea com perfil,
# triagem e check-in de hoje aguardam a escuta. Idempotente. Roda depois do
# SchedulingCrew.
class ScreeningCrew
  UNIT = "UBS Jardim das Flores"
  NAME = Protocols::Validation::Screening::NAME
  RULES = [
    { "when" => { "any" => [ { "gte" => ["vitals.systolic", 180] }, { "gte" => ["vitals.diastolic", 120] },
                             { "lt" => ["vitals.spo2", 90] }, { "gte" => ["vitals.respiratory_rate", 30] },
                             { "gte" => ["vitals.heart_rate", 130] }, { "lt" => ["vitals.capillary_glucose", 50] } ] },
      "color" => "red" },
    { "when" => { "any" => [ { "gte" => ["vitals.temperature_c", 39] }, { "gte" => ["vitals.capillary_glucose", 300] },
                             { "gte" => ["vitals.systolic", 160] }, { "lt" => ["vitals.spo2", 94] },
                             { "gte" => ["vitals.pain_score", 8] } ] },
      "color" => "yellow" },
    { "when" => { "any" => [ { "gte" => ["vitals.temperature_c", 37.8] }, { "gte" => ["vitals.pain_score", 4] } ] },
      "color" => "green" }
  ].freeze
  WALK_INS = [ [ "1958-03-14", "female" ], [ "1991-11-02", "male" ] ].freeze

  class << self
    def seed_current_city(slug:, ddd:)
      admin = User.find_by!(email_address: "admin@#{slug}.demo")
      unit = HealthUnit.find_by!(name: UNIT)
      unit.update!(screening_scope: "walk_in") unless unit.screening_scope == "walk_in"
      link = ensure_technician_link(slug, unit, admin)
      protocol = slug == "curitiba" ? ensure_active!(slug) : ensure_draft!(slug)
      walk_ins = WALK_INS.each_with_index.count { |(birth, sex), i| ensure_walk_in(slug, ddd, unit, i, birth, sex) }
      { protocol: "#{protocol.name} v#{protocol.version} (#{protocol.status})", technician_link: link.cbo_code,
        walk_ins: walk_ins }
    end

    private

    def definition(version) = { "name" => NAME, "version" => version, "kind" => "screening", "risk_rules" => RULES.map(&:deep_dup) }

    def ensure_technician_link(slug, unit, admin)
      professional = User.find_by!(email_address: "tecnico@#{slug}.demo").professional
      existing = professional.links.active.find_by(health_unit: unit, cbo_code: "322205")
      return existing if existing

      result = Professionals::OpenLink.call(professional: professional, health_unit_id: unit.id, cbo_code: "322205", by: admin)
      raise "semente do acolhimento: vínculo da técnica recusado (#{result.reason})" if result.failure?

      result.payload[:link]
    end

    # Curitiba: a versão ativa com as regras; retoma uma pendente com as mesmas.
    def ensure_active!(slug)
      active = ProtocolDefinition.find_by(name: NAME, status: "active")
      return active if active && active.definition["risk_rules"] == RULES

      versions = ProtocolDefinition.where(name: NAME)
      pending = versions.where.not(status: %w[active retired]).find { |v| v.definition["risk_rules"] == RULES }
      version = pending&.version || ((versions.maximum(:version) || 0) + 1)
      TriageCatalogCrew.run_cycle!(slug, NAME, version, definition(version))
    end

    def ensure_draft!(slug)
      existing = ProtocolDefinition.where(name: NAME).order(:version).last
      return existing if existing

      author = User.find_by!(email_address: "autor@#{slug}.demo")
      result = Protocols::SaveDraft.call(definition: definition(1), by: author)
      raise "semente do acolhimento: rascunho recusado (#{result.reason})" if result.failure?

      result.payload[:protocol_definition]
    end

    # true quando fez o check-in agora; quem já está aguardando hoje fica.
    def ensure_walk_in(slug, ddd, unit, index, birth, sex)
      registered = Citizens::RegisterPerson.call(phone: format("+55%s96666%04d", ddd, index + 1),
                                                 cpf: cpf_for("#{slug}:screening:#{index}"),
                                                 profile: { birth_date: birth, sex: sex, gender_identity: nil })
      raise "semente do acolhimento: cidadão recusado (#{registered.reason})" if registered.failure?

      citizen = registered.payload[:citizen]
      return false if Attendance.waiting.where(citizen: citizen, health_unit: unit).exists?

      started = Citizens::StartConversation.call(citizen: citizen, consent_version: Consents.current_version,
                                                 session_id: "seed-screening")
      raise "semente do acolhimento: conversa recusada (#{started.reason})" if started.failure?

      %w[true true].each_with_index do |answer, step|
        Citizens::SubmitAnswer.call(conversation: started.payload[:conversation], answer: answer,
                                    idempotency_key: "seed-screening-#{slug}-#{index}-#{Time.zone.today}-#{step}")
      end
      triage = started.payload[:triage].reload
      code = Citizens::IssueCheckInCode.call(citizen: citizen, triage: triage).payload.fetch(:code)
      reception = User.find_by!(email_address: "recepcao@#{slug}.demo")
      result = Attendances::CheckIn.call(cpf: citizen.cpf, code: code, health_unit_id: unit.id, document_checked: false,
                                         by: reception)
      raise "semente do acolhimento: check-in recusado (#{result.reason})" if result.failure?

      true
    end

    def cpf_for(seed)
      base = Digest::SHA256.hexdigest(seed).scan(/\d/).join[0, 9].ljust(9, "7")
      nums = base.chars.map(&:to_i)
      first = CitizenIdentity::Cpf.check_digit(nums)
      "#{base}#{first}#{CitizenIdentity::Cpf.check_digit(nums + [ first ])}"
    end
  end
end
```

Em `db/seeds.rb`, depois de `require Rails.root.join("lib/scheduling_crew").to_s` (linha 42), `require Rails.root.join("lib/screening_crew").to_s`, e depois do bloco da agenda:

```ruby
        # ── Acolhimento (módulo 18, spec 2026-10-07 §10) ──────────────────────
        # Depois da agenda: usa o elenco do ciclo assinado, a UBS, a técnica e a recepção.
        screening = ScreeningCrew.seed_current_city(slug: slug, ddd: ddd)
        puts "[seeds] acolhimento . #{screening[:protocol]}; técnica na UBS (#{screening[:technician_link]}); " \
             "#{screening[:walk_ins]} check-ins novos de demanda espontânea"
```

- [ ] **Step 4: Rode a spec e a semente de verdade**

```bash
docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec spec/lib/screening_crew_spec.rb
```
Expected: PASS. Confira a sintaxe da semente: `docker compose exec -T -w /rails/.claude/mod18 api ruby -c db/seeds.rb lib/screening_crew.rb` → `Syntax OK`. A semente de verdade roda no banco de dev só na Task 18, com autorização do usuário (exige `city:migrate:all` com as migrações deste branch; o elenco que ela usa não existe no banco de teste).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod18 add lib/screening_crew.rb db/seeds.rb spec/lib/screening_crew_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod18 commit -m "chore: seed the signed acolhimento protocol, technician link and walk-ins for dev

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 18: Revisão final, suíte completa e api na porta 3035

- [ ] **Step 1:** Varredura de nomes: `grep -rn "last_error\b\|ErrorText\|ciap2_codes#index\|citizen_id" apps/api/.claude/mod18/app/controllers/screenings_controller.rb apps/api/.claude/mod18/app/services/ledi apps/api/.claude/mod18/app/commands/ledi` — nada (o `suggest` usa `attendance_id`).
- [ ] **Step 2:** Bancos de teste refeitos (Ambiente) e suíte completa com o worker parado:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod18 api bundle exec rspec
  docker compose start worker
  ```
  Expected: verde. Falha em spec antiga que fixa chaves de unidade, fila ou ficha → acrescente `screening_scope`, `screening`, `last_error_codes` à lista (é o contrato); nunca afrouxe asserção de texto livre.
- [ ] **Step 3:** `bundle exec rubocop` nos arquivos tocados (`docker compose exec -T -w /rails/.claude/mod18 api bundle exec rubocop <arquivos>`); corrija só o que a regra do projeto aponta.
- [ ] **Step 4:** Para a prova no navegador (spec §9) e para o plano do dashboard, suba o api do worktree na porta **3035**, sem derrubar o principal:

  ```bash
  docker compose exec -d -w /rails/.claude/mod18 api bin/rails server -b 0.0.0.0 -p 3035 -P tmp/pids/server-mod18.pid
  ```
  O Vite do worktree do dashboard aponta `VITE_API_PROXY_TARGET` para `http://api:3035`. **Antes**, com autorização do usuário: `city:migrate:all` (as cidades de dev ganham as tabelas da escuta e a fila LEDI perde `last_error`) e `db:seed` (Task 17). Roteiro (spec §9): o acolhimento v-n ativo em Curitiba; a enfermeira abre a fila do acolhimento da UBS Jardim das Flores, escuta um dos dois de demanda espontânea com PA 185/110 → sugestão vermelha, cor final vermelha, destino `same_day` → a pessoa vai para o topo da fila do profissional; o segundo com destino `schedule` → o pedido aparece na fila de agendamento do módulo 17 (`kind: "screening"`); em Produção e-SUS aparece a ficha gerada ou a "não gerada" com o motivo (a UBS da semente não tem CNES confirmado nem equipe local, então o esperado é "não gerada"). Login, OTP e TOTP são do usuário (senhas e TOTP da semente de dev podem ser mostrados no chat se ele pedir).
- [ ] **Step 5:** **Pare.** Merge, push, board e docs (página de status do módulo) só com autorização explícita do usuário, uma etapa de cada vez. Ordem: `contracts` (`protocols-v1.6.0`) → api → dashboard. Rollout por cidade: publicar a imagem nova e rodar `city:migrate:all` dela antes de cortar tráfego (a 300002 converte o texto antigo da fila em códigos e o apaga: irreversível). Ao voltar o checkout para a main: `DROP DATABASE` dos dois bancos de teste e `city:test_databases`; derrube o servidor da 3035 (`kill $(cat tmp/pids/server-mod18.pid)` no container).

---
## Self-review (feito ao escrever o plano)

**Cobertura da spec:**
- §3.1 dados e sinais → Tasks 3, 6; §3.2 comandos e CBOs → Tasks 8, 9 (lock com threads na Task 10); §3.3 classificação, schema 1.6.0, variáveis, semente do rascunho → Tasks 2, 7, 17.
- §4 destinos, trava com trigger, filas → Tasks 3 (trigger e CHECKs), 9 (destinos, pedido ADR 0029, fechamento), 11 (filas), 12 (rotas).
- §5 Tarefa 1 → Task 1 (com critério de parada); fichas por CBO → Task 13; job do fechamento + varredor 23h + só com exportação utilizável → Task 14; "não gerada" e "gerar de novo" → Tasks 14–15; regeneração de recusada com `replaces_outbox_id` e índice só das não recusadas → Tasks 4 (índice/coluna), 14 (`Enqueue … replaces:`), 15 (Reenviar).
- §6 LGPD/api#43 → Task 4 (códigos, migração de limpeza, CHECK), Task 5 (purga 90 dias, exclusão), Task 3 (`filter_parameters`), Task 11–12 (recepção só cor), Task 12 (`screening.viewed`), Task 16 (invariantes). Revisão do ADR 0028: plano do docs.
- §7 API → Task 12 (+ Task 15 para produção). §9 testes → cada task + Task 16. §10 semente → Task 17. §11 rollout → Task 18 e Global Constraints; `VALID_RANGE` → Task 3; `recurring.yml` → Tasks 5 e 14.

**Placeholders:** nenhum "TBD"/"similar à Task N"; as listas copiadas da migração para o dump estão por extenso; a única transcrição deixada ao executor é a tabela oficial de CBOs do MIAI (Task 1, Step 2), com fonte, lista inicial já lida e asserções que a prendem.

**Consistência de nomes:** `Screenings::{VitalSigns,RiskSuggestion,ActiveProtocol,Suggest,Ciap2,Cbos,Scope,Authorization,RevisionInput,Queue,Json}`, `Screenings::{Start,Abandon,Complete,Reassess}`, `Ledi::{ScreeningMapping,ErrorCodes,CitizenSources,ScreeningFicha}`, `Ledi::Fichas::{ScreeningIdentity,Medicoes,InitialListening,ScreeningProcedures}`, `Ledi::{ScreeningFichaJob,ScreeningFichaSweepJob,PurgeStalePayloadsJob}`, `Protocols::SimulateScreening`, `Attendance::{OUTCOMES,CLOSE_OUTCOMES,SCREENING_OUTCOMES}` — conferidos entre as tasks. `Screenings::Abandon.release!` é usado por `Attendances::Call`/`Close` (Task 8) e pelo teste de corrida (Task 10).

**Review Focus:** as cinco linhas têm teste na task dona (Tasks 10, 6/12, 7/2, 8/14, 14/15).

## Divergências propostas ao contrato

As decisões do coordenador (contrato §9: busca de CIAP-2 por `POST /attendance/ciap2/search` com 503 `terminology_unavailable`; `id` no bloco `screening` da fila; `suggest` por `attendance_id` com vínculo na unidade; `simulate_screening` no editor; `schema_version` inteiro; a variante proíbe `start_step_id`/`recommendations`/`priority_when`) já estão no plano. O que ainda falta no contrato, para os planos do dashboard e do contracts:

- **D1 — `invalid_unit` (422) em `POST /attendance/screenings/:id/complete`** com `destination: "referred"` e `referral_unit_id` inexistente, inativa ou não-uuid (mesmo código do desfecho do módulo 13). O contrato só lista `referral_required`.
- **D2 — `note_too_long` (422, com `field` ∈ `complaint_note`, `orientation_note`, `color_change_reason`)** em `complete` e `reassess` quando o texto passa de 500 caracteres.
- **D3 — `GET /production/generation_failures`**: sem `resolved` = não resolvidas (como `resolved=false`); `resolved=true` lista as resolvidas; até 200, mais novas primeiro, sem paginação.
- **D4 — `screening.viewed` também no detalhe da chamada** (`call`/`call_next`, que devolvem `attendance.screening` só quando a escuta está concluída; `null` em curso/abandonada).
- **D5 — "Reenviar" ficha de escuta recusada regera da origem**: `POST /production/fichas/:id/resend` devolve 200 com a ficha **nova** (outro `id` e `uuid`); 409 `not_rejected` também quando a recusada já foi regenerada; 409 novos `export_unusable` (exportação inutilizável agora) e `generation_failed` (identificação incompleta — a falta vai para "não geradas"). Sugestão: acrescentar `replaces_outbox_id` à forma da ficha em `GET /production` para o painel ligar a nova à antiga (não implementado neste plano).
- **D6 — `ledi.payload_purged { count }`** só é publicado quando `count > 0`.
- **D7 — CBO permitido** = grupos da spec ∩ fichas possíveis (tabela do MIAI ou `3222xx`): ex. `223293` recebe 403 `cbo_not_allowed`. Comportamento, não formato; o dashboard deve mostrar a mensagem desse código.
- **D8 — `POST /attendance/attendances/:id/screening` sobre escuta abandonada** devolve 201 com o **mesmo** `id` (uma escuta por atendimento), `status: "in_progress"` e o novo autor.
- **D9 — "Chamar próximo" não pula quem ainda espera escuta** (a ordem é a da fila, contrato §4); a chamada (direta ou próxima) de quem está com escuta em curso a abandona.
- **D10 — Técnico sem aferição:** pela spec §5 não há ficha. O LEDI aceita a Ficha de Procedimentos só com `statusEscutaInicialOrientacao = true` (sem procedimento); se a coordenação quiser registrar toda escuta do técnico, é uma linha em `Ledi::Fichas::ScreeningProcedures.applicable?` — decisão de produto, não implementada.
