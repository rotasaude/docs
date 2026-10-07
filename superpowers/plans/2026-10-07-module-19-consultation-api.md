# Módulo 19 (19a) — Consulta / prontuário da APS (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: execute este plano só DEPOIS que o módulo 18 (acolhimento, `feat/mod-18-screening`) estiver em `origin/main` do api.** O plano usa código do 18 pelos nomes do plano do api do 18 (`docs/superpowers/plans/2026-10-07-module-18-screening-api.md`, seção "Valores fixados") e do branch dele: `Screenings::VitalSigns`, `Screenings::Ciap2`, `Screenings::Json`, `ScreeningRevision::VITAL_COLUMNS`, `Ledi::ScreeningMapping` (`miai_cbo?`, `sex_code`, `turno`, `measurement_field`), `Ledi::Fichas::{ScreeningIdentity,Medicoes}`, `Ledi::ScreeningFicha` (`retry!`, `regenerate`, `AlreadyResolved`), `Ledi::Enqueue.call(ficha, city:, replaces:)`, `LediGenerationFailure`, `ledi_outbox.{last_error_codes,replaces_outbox_id,last_attempted_at}`, `POST /attendance/ciap2/search`, `screening.viewed`, `ScreeningHelpers` (`ciap2_release!`, `screening_citizen!`, `screener!`, `walk_in_attendance!`, `reception!`, `exportable_unit!`). A Task 0 confere isso antes de tudo.

**Goal:** O prontuário da APS no modo `record` — paciente por CPF ligado aos pares validados (com nome completo, social e da mãe conferidos na validação presencial), lista de problemas por eventos, consulta SOAP com rascunho → finalizada imutável → adendos, leitura em contexto e abertura justificada com step-up, trilha de toda leitura, impresso PDF para assinatura manual e a ficha LEDI de Atendimento Individual (com `correction_pending` depois do aceite) — o lado api de F-19.1 a F-19.7 (ADR 0031).

**Architecture:** A Task 1 confirma o layout LEDI 8.7.0 (IDLs vendorizados + documentação oficial) e grava `config/ledi/consultation_mapping.yml` com fonte por linha (lido por `Ledi::ConsultationMapping`). Três migrações de cidade: `20261007400001` (nomes e `patient_id` em `citizens`, `patients`, `patient_profile_divergences`, `patient_problems`, `patient_problem_events`), `20261007400002` (consultas, itens, adendos, aberturas) e `20261007400003` (`ledi_outbox` com `correction_pending`). O banco é a última palavra: a lista de problemas só muda com evento da mesma transação (trigger `patient_problems_event_required`), consulta finalizada não muda (trigger, com a única exceção da re-cifra sob `rota.reencrypting`), itens e adendos são só acréscimos, par declarado nunca se liga a paciente. Os comandos (`Patients::Resolve`, `Patients::ApplyProblemEvent`, `Consultations::{Start,SaveDraft,Finalize,AddAddendum}`, `ClinicalRecord::Open`) travam sempre atendimento → consulta → paciente; `Consultations::Finalize` chama o `Attendances::Close` existente na mesma transação. A leitura passa sempre por `ClinicalRecord::Access` e publica `clinical_record.viewed`. A ficha é `Ledi::Fichas::IndividualCare` (interface `Ledi::Ficha` do módulo 16), gerada por `Ledi::ConsultationFicha` no `Ledi::ConsultationFichaJob`.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade), RSpec, Thrift (LEDI 8.7.0 vendorizado), Solid Queue, Active Record Encryption (chave por cidade, ADR 0007), **Prawn** (PDF, gem nova) e **pdf-reader** (só teste).

**Spec:** `docs/.claude/ciclo2/superpowers/specs/2026-10-07-module-19-consultation-design.md` e `docs/.claude/ciclo2/adr/0031.md` (leia os dois antes de começar). Contratos entre apps: `docs/.claude/ciclo2/superpowers/plans/2026-10-07-module-19-consultation-contracts.md` — fonte única dos formatos; o plano do dashboard foi escrito contra ele: **não mude nomes, formatos nem códigos de erro**. O que o código real obrigou a precisar está em "Desvios da spec" e, quando toca formato, em "Divergências propostas ao contrato" (fim do arquivo).

## Desvios da spec (e precisões)

1. **Layout confirmado (Task 1; lido ao escrever o plano, 2026-10-07).** `tipoAtendimento` do MIAI aceita só `1, 2, 4, 5, 6`; a consulta usa `1` (agendada programada/cuidado continuado), `2` (agendada), `5` (no dia) e `6` (urgência) — `4` é a escuta do módulo 18. `condutas` 1–12 sem repetição (CondutaEncaminhamento: 1, 2, 4, 5, 6, 7, 8, 9, 10, 11, 12, 14). `exame` (`ExameThrift`): `codigoExame` SIGTAP sem pontuação **do grupo 02** (ou da ListaExames, que não usamos), sem repetição, até 100; `solicitadoAvaliado` = `["S"]` (SituacaoExame `S` Solicitado, `A` Avaliado). `ProblemaCondicaoThrift`: `ciap` ou `cid10` (um dos dois), `situacao` 0 Ativo / 1 Latente / 2 Resolvido; preencher `situacao` exige `uuidProblema`, `uuidEvolucaoProblema` e `coSequencialEvolucao` (e vice-versa); `dataInicioProblema` entre o nascimento e o atendimento; `dataFimProblema` obrigatória com `situacao` 2. **Nenhuma regra de CID-10 por CBO** foi achada no dicionário do MIAI nem em `regras/cbo.html`: o único recorte é a Tabela 3 (CBOs que registram MIAI), que já é o recorte da consulta. O mapeamento guarda a regra (`cid10_cbos.rule: miai_table`) e o código continua conferindo `cid10_not_allowed_for_cbo` (vira lista de prefixos com uma linha no YAML se o produto quiser restringir a médicos — "Decisão em aberto" no fim).
2. **Quem registra consulta** = CBO na tabela do MIAI (`Ledi::ScreeningMapping.miai_cbo?`, do módulo 18) **e** fora de `2232` (dentista, módulo 28). Técnico `3222` fica de fora (não está no MIAI).
3. **Encaminhamento** vai à ficha só como conduta (4, 5, 6, 7, 8 ou 10, escolhida pelo profissional). `encaminhamentos` (`EncaminhamentoExternoThrift`) exige `especialidade` de tabela que o projeto não tem; o desfecho `referred` do atendimento continua gerando o pedido do módulo 17 como hoje.
4. **Texto clínico cifrado e imutável ao mesmo tempo.** `consultations.{subjective,objective,assessment,plan}`, `consultation_addenda.text` e `clinical_record_openings.reason_note` são `encrypts` (chave da cidade, não determinístico) e entram em `CityEncryption::CITY_KEYED_TARGETS`. Os triggers de imutabilidade aceitam uma única exceção: UPDATE que muda **só** essas colunas, com `SET LOCAL rota.reencrypting = 'on'` na transação — o que `CityEncryption.allowing_reencryption` faz para `ReencryptionJob` e `CityRekey`. Sem isso a rotação de chave da cidade quebraria na primeira consulta finalizada (Review Focus 1).
5. **Rascunho guarda os itens em `consultations.draft_items` (jsonb).** As tabelas de itens (`consultation_problems`, `consultation_conducts`, `consultation_exam_requests`) são só acréscimos: recebem linhas na finalização (consulta ainda `draft` dentro da transação) e depois só com `addendum_id`. `draft_items` vai a `{}` na finalização (CHECK).
6. **Adendo muda itens por acréscimo (contrato §9).** `changes` = `{ "evaluated_problems": [<item avaliado>], "conducts": [lista final], "exam_requests": [lista final de { sigtap_code, cid10_justification? }] }` — `evaluated_problems` são eventos novos; as listas finais só entram em `changes` quando diferem do estado efetivo. O banco recebe a diferença como linhas novas: conduta removida = linha `action: "remove"`; exame retirado (ou com justificativa trocada) = linha `status: "cancelled"` (e, se trocada, nova `requested`). O estado efetivo (para a ficha e o impresso) é `Consultations::Effective`. `opening_id` é aceito também do autor (fica registrado se válido).
7. **"Só muda por evento" no banco.** `patient_problem_events` ganha `txid` (`txid_current()`); o trigger `patient_problems_event_required` (BEFORE INSERT OR UPDATE) exige um evento da mesma transação com os mesmos valores novos. As FKs do evento (para o problema, a consulta e o adendo) são `DEFERRABLE INITIALLY DEFERRED` (o evento nasce antes do problema novo; o commit confere). `ApplyProblemEvent` gera o id do problema novo em Ruby.
8. **`add` sobre problema resolvido reativa** (evento `reactivated`, mesma linha); `add` sobre ativo igual é `evaluate` (sem evento). `resolve` grava `resolved_on` = dia da consulta (`dataFimProblema`).
9. **`uuidEvolucaoProblema`** = id da linha de `consultation_problems`; `coSequencialEvolucao` = posição dessa linha entre as do mesmo problema (ordem `created_at, id`); `uuidProblema` = id de `patient_problems`.
10. **Paciente ainda não criado.** `GET /attendance/attendances/:id/record` de par validado sem paciente devolve `patient.id: null`, `problems: []`, `consultations: []` (o paciente nasce na primeira consulta — spec §3) e não publica `clinical_record.viewed` (não há prontuário); a escuta do dia publica `screening.viewed`.
11. **Contexto sem `attendance_id`.** `GET /attendance/consultations/:id` e o impresso procuram contexto em qualquer atendimento aberto de par validado do mesmo CPF (`in_care` chamado pelo usuário, ou `waiting` em unidade de vínculo ativo dele com CBO permitido); fora disso, abertura válida; senão 403 `out_of_context`.
12. **Revogação da validação** não desfaz `citizens.patient_id` (ADR 0026/0031: "revogação não afeta o prontuário"); o trigger `citizens_patient_link_guard` impede **ligar** par declarado e impede trocar a ligação. Consulta nova exige o par validado de novo.
13. **Rascunho aberto e a rota antiga de desfecho.** `Attendances::Close` recusa desfecho com consulta `draft` no atendimento: 409 `consultation_in_progress` (Review Focus 4). `Finalize` vira a consulta para `finalized` antes de chamar o `Close`, na mesma transação.
14. **Pressão incompleta no autosave** vira 422 `implausible_vital` com `field` do lado que falta (o contrato do PATCH só tem esse código); o validador é o `Screenings::VitalSigns` do módulo 18.
15. **Regeneração por adendo**: ficha `pending`/`failed` → conteúdo regravado com o mesmo uuid; `rejected` (não regerada) → linha nova com `replaces_outbox_id`; `sending` → o job tenta de novo em 1 min; `accepted` → uma linha `correction_pending` por ficha aceita (uuid novo, `replaces_outbox_id` = a aceita), regravada a cada adendo e nunca enviada (`claim!` só pega `pending`). `ProductionSummary.counts` continua só com os cinco estados (o console não muda); a ficha aparece em `GET /production` com `status: "correction_pending"`.
16. **Semente (spec §9) liga o interruptor e o modo `record` em Curitiba (dev)**, ao contrário das sementes 16–18 — a spec manda. Com o mantenedor de dev (`dev@local`); sem ele, avisa e não liga.

## Valores fixados por este plano (para o contrato)

1. **Tipos de atendimento** (`GET /attendance/consultation_options`, `care_type`): `1` "Consulta agendada programada / Cuidado continuado", `2` "Consulta agendada", `5` "Consulta no dia", `6` "Atendimento de urgência". Sugestão: atendimento com horário (`appointment_id`) → `2`; demais (demanda espontânea, com ou sem escuta `same_day`) → `5`.
2. **Condutas**: `1` Retorno para consulta agendada, `2` Retorno para cuidado continuado/programado, `4` Encaminhamento para serviço especializado, `5` Encaminhamento para CAPS, `6` Encaminhamento para internação hospitalar, `7` Encaminhamento para urgência, `8` Encaminhamento para serviço de atenção domiciliar, `9` Alta do episódio, `10` Encaminhamento intersetorial, `11` Encaminhamento interno no dia, `12` Agendamento para grupos, `14` Agendamento para eMulti. 1–12 por consulta, sem repetir.
3. **Erros novos (422)**: `invalid_text` (com `field`; texto que não é string), `invalid_problem` (com `index`), `invalid_onset` (com `index`), `cid10_sex_incompatible` (com `index`), `invalid_conduct`, `invalid_exam` (com `index`), `invalid_care_type`, `text_required` (adendo sem texto), `invalid_changes`, `invalid_period` (relatório). 409 novos: `consultation_in_progress` (rota de desfecho com rascunho aberto). `already_exists` leva `consultation_id`.
4. **Limites**: S, O, A, P e texto do adendo até 20.000 caracteres; motivo do adendo 10–500; nota da abertura 10–500 (só com `other`); até 50 problemas, 12 condutas e 100 exames por consulta; nomes: completo 3–200, social e mãe ≤ 200.
5. **Abertura**: válida 30 min (`expires_at = created_at + 30 min`); relatório até 500 linhas, mais novas primeiro; `from`/`to` `AAAA-MM-DD` no fuso da cidade.
6. **Interruptor**: `clinical_record` com `requires: ["record_mode:record"]`; faltando, `missing` traz `record_mode_not_record`.
7. **Decisões do coordenador (contrato §9), incorporadas:** 409 `already_exists` com `consultation_id`; nomes de par já validado completados no check-in (`check_ins/lookup` → `citizen.names` e `citizen.verification_id`; `check_ins` → `verification_id` quando valida; `POST /attendance/verifications/:id/names` com a validação ativa); `display_name` em todo item da fila; `changes` do adendo com `evaluated_problems` (eventos novos) e `conducts`/`exam_requests` (listas finais, só quando mudam), `opening_id` aceito também do autor; o PATCH do rascunho recebe todos os campos editáveis a cada salvamento (o api aceita corpo parcial do mesmo jeito); `POST /attendance/verifications` sem a chave `full_name` continua aceito e não grava nome (com a chave, 3–200 obrigatório); `<record>.consultations` só finalizadas; `cid10_justification` segue a regra de CID-10 por CBO; impresso `application/pdf` inline no sucesso e JSON `{ error }` nas recusas.

## Global Constraints

- Tudo de domínio no banco de cada cidade: migração em `db/city_migrate`, dump à mão em `db/city_schema.rb` (a paridade compara o schema normalizado, `spec/services/city_schema_spec.rb`), triggers em `db/city_triggers.sql` (a migração termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`; blocos novos guardados por `to_regclass`). Migrações irreversíveis (`down` levanta `ActiveRecord::IrreversibleMigration`). Números `20261007400001`, `20261007400002`, `20261007400003` — maiores que os do módulo 18 (`20261007300002`). Rollout: publicar a imagem e rodar `city:migrate:all` **antes** de cortar tráfego; **nunca** migrar fora do rake.
- Cifra por cidade (ADR 0007): `encrypts` sem `key_provider` (não determinístico) para texto clínico, nomes, nascimento e sexo; `deterministic: true, key_provider: CityDeterministicKeyProvider.new` só para `patients.cpf`. Todo `encrypts` novo entra em `CityEncryption::CITY_KEYED_TARGETS` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`).
- Valores, exatamente: consulta `draft` | `finalized`; problema `ciap2` | `cid10`, `active` | `resolved`, precisão `day` | `month` | `year`; evento `added` | `resolved` | `reactivated` | `onset_corrected`; ação do item `evaluate` | `add` | `resolve` | `correct_onset`; motivo da abertura `case_review` | `active_search` | `continuity_of_care` | `other`; acesso `in_context` | `justified`; exame `requested` | `cancelled`; conduta `add` | `remove`; fila LEDI ganha `correction_pending`.
- Erros `{ "error": "<reason>" }` (com `field`/`index` quando dito); step-up 401 `{ "error": "mfa_required" }`; interruptor 403 `{ "error": "feature_disabled", "feature": "clinical_record" }`; papel ausente 403 `missing_role`; escrita devolve o objeto puro.
- Eventos só com ids (contratos §7), declarados em `config/initializers/domain_events.rb` (`to: []`) e em `spec/initializers/domain_events_bindings_spec.rb`: `patient.created`, `patient.linked`, `patient_problem.changed`, `consultation.started`, `consultation.finalized`, `consultation.addendum_added`, `clinical_record.viewed`, `clinical_record.opened`. Nenhum texto clínico (S, O, A, P, adendo, nota) nem nome em evento, log (`filter_parameters`), Analytics ou mensagem de erro; sem busca por texto. Nenhum `Platform.audit` novo.
- Jobs por cidade só com `include CityScopedJob` (ou `prepend EachCityJob`); `Current.city` **nunca** é atribuído em `app/` ou `lib/` (`spec/architecture/current_city_assignment_spec.rb`).
- Specs de request com `type: :request` (`infer_spec_type_from_file_location!` está desligado); arquivo novo em `spec/support/` precisa de `require_relative` em `spec/rails_helper.rb`. Specs com threads: `self.use_transactional_tests = false`, `Queue#pop(timeout:)`, `after` que solta as threads e apaga o que commitou (`session_replication_role = replica`, como `spec/commands/screenings/screening_concurrency_spec.rb`).
- `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..31)`.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). `git add` sempre com caminhos explícitos (nunca `-A`).

## Ambiente de execução

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API") e a sessão do módulo 18. Ordem de merge: api → dashboard (o `maintenance` não muda: o interruptor entra pelo mecanismo genérico).
- Worktree (a partir da raiz do monorepo, `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod19 -b feat/mod-19-consultation origin/main
  cp apps/api/config/master.key apps/api/.claude/mod19/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod19`. Todo comando Rails/RSpec:

  ```bash
  docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec <arquivos>
  ```

- Todo `git add`/`git commit` usa `-C apps/api/.claude/mod19`.
- Depois de cada migração de cidade (Tasks 3, 7 e 15): `DROP DATABASE` dos dois bancos de teste de cidade e `city:test_databases` de novo:

  ```bash
  psql -U rota_saude -d postgres -c "DROP DATABASE rota_saude_test_city_a" -c "DROP DATABASE rota_saude_test_city_b"
  docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod19 api bin/rails city:test_databases
  ```

  Ao voltar para a main, repita (o banco de teste fica à frente).
- Gem nova (Task 14): `bundle install` dentro do container (`docker compose exec -T -w /rails/.claude/mod19 api bundle install`); o `Gemfile.lock` do worktree muda. A imagem de produção precisa ser reconstruída (rollout).
- Suíte completa só com o worker parado e sem outra sessão rodando suíte:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec
  docker compose start worker
  ```

- A Task 1 pode usar WebFetch (só leitura) na documentação oficial `https://integracao.esusaps.bridge.ufsc.tech`.
- Prova no navegador e plano do dashboard: o api do worktree sobe na porta **3036** (Task 20). O prefixo novo `/clinical_record` precisa de entrada no proxy de dev do dashboard (plano do dashboard).

### Arquivos em comum com o módulo 18 (e como rebasear)

Se o 18 ainda receber correções depois de mergeado, ou se este branch nascer antes do merge final, rebase sobre `origin/main` e resolva **somando** os dois lados nestes arquivos (nenhum dos dois apaga o do outro): `db/city_schema.rb` (o `define(version:)` fica com o maior número; tabelas em ordem alfabética), `db/city_triggers.sql` (blocos novos no fim), `config/initializers/domain_events.rb` e `spec/initializers/domain_events_bindings_spec.rb` (um bloco por módulo), `config/initializers/filter_parameter_logging.rb`, `spec/adr_pointers_spec.rb` (fica `1..31`), `config/routes.rb` (rotas novas depois das da escuta), `app/controllers/attendance_controller.rb`, `app/controllers/attendances_controller.rb` (`queue_json`), `app/commands/attendances/close.rb`, `app/controllers/ciap2_codes_controller.rb`, `app/controllers/production_controller.rb`, `app/commands/ledi/resend.rb`, `app/services/city_encryption.rb`, `db/seeds.rb` (bloco do 19 depois do acolhimento). Depois do rebase: DROP + `city:test_databases` e a Task 0 de novo.

## Review Focus

1. **Rotação de chave da cidade com consulta finalizada e adendo no banco:** `ReencryptionJob` e `CityRekey` regravam o texto cifrado e o conteúdo continua legível; um UPDATE comum (sem a marca) continua recusado, e com a marca só as colunas cifradas podem mudar. Nunca uma rotação que para no meio por causa do trigger. Testes: Task 7 ("re-cifra de consulta finalizada").
2. **Texto livre que a fonte do PDF não tem** (emoji, "≥", aspas tipográficas, quebra de linha `\r\n`, 20.000 caracteres): o impresso sai, com `?` no lugar do que não cabe, nunca 500; nome no lugar certo (nome social, se houver). Testes: Task 14 ("caracteres fora do WinAnsi").
3. **Autosave nas bordas:** só a sistólica digitada, vírgula decimal, texto de 20.001 caracteres, chave desconhecida, lista que não é lista, PATCH que chega depois da finalização ou de outra pessoa → 422 com `field`/409 `not_draft`/403 `not_author`, nunca 500 nem dado parcial gravado. Testes: Task 8 (tabela de entradas), Task 9 (nada gravado) e Task 13 (rota).
4. **Profissional encerra pela rota antiga de desfecho com a consulta em rascunho**, ou a finalização falha no desfecho (unidade de encaminhamento inativa): nunca atendimento fechado com rascunho órfão, nunca consulta finalizada com atendimento aberto, nenhum evento de problema sobra. Testes: Task 10 ("rascunho aberto bloqueia o desfecho", "falha do desfecho desfaz tudo").
5. **Abertura justificada nas bordas:** exatamente 30 minutos depois, abertura de outro paciente ou de outro usuário usada no adendo, CPF com máscara, CPF de par só declarado (sem paciente) → 403 `opening_required`/404 `patient_not_found`, nunca leitura sem trilha. Testes: Task 11 ("validade e dono da abertura") e Task 13 (rotas).

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | conferir o pré-requisito (módulo 18 na main) | 0 |
| `config/ledi/consultation_mapping.yml`, `app/services/ledi/consultation_mapping.rb` | layout LEDI confirmado, com fonte por linha | 1 |
| `app/services/platform/features.rb`, `app/services/clinical_record/gate.rb`, `app/controllers/concerns/clinical_record_gate.rb`, `spec/adr_pointers_spec.rb` | interruptor `clinical_record` | 2 |
| `db/city_migrate/20261007400001_add_patients.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{patient,patient_problem,patient_problem_event,patient_profile_divergence,citizen}.rb`, `app/services/city_encryption.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/support/clinical_record_helpers.rb` | paciente, nomes e lista de problemas (dados) | 3 |
| `app/commands/citizens/{name_values,names_json,verify,complete_names}.rb`, `app/controllers/{attendance,check_ins,attendances}_controller.rb`, `config/routes.rb` | nomes na validação presencial e no check-in | 4 |
| `app/commands/patients/resolve.rb` | paciente por CPF, ligação e divergência | 5 |
| `app/commands/patients/apply_problem_event.rb`, `app/services/patients/problem_replay.rb` | único caminho de escrita da lista | 6 |
| `db/city_migrate/20261007400002_add_consultations.rb`, `app/models/{consultation,consultation_problem,consultation_conduct,consultation_exam_request,consultation_addendum,clinical_record_opening}.rb`, `app/jobs/reencryption_job.rb`, `app/commands/city_rekey.rb` | consulta, itens, adendos, aberturas (dados) e re-cifra | 7 |
| `app/services/consultations/{cbos,authorization,care_type,items_input}.rb`, `app/services/clinical_terms.rb`, `app/services/clinical_terms/sigtap_exams.rb` | quem registra e validação dos itens | 8 |
| `app/commands/consultations/{start,save_draft}.rb` | iniciar e autosave | 9 |
| `app/commands/consultations/finalize.rb`, `app/services/consultations/effective.rb`, `app/commands/attendances/close.rb` | finalizar com o desfecho | 10 |
| `app/services/clinical_record/access.rb`, `app/commands/clinical_record/open.rb`, `app/commands/consultations/add_addendum.rb` | leitura, abertura, adendo | 11 |
| `app/services/consultations/json.rb`, `app/services/clinical_record/json.rb`, `app/services/clinical_record/trail.rb` | formas do contrato e trilha | 12 |
| `app/controllers/{consultations,clinical_records,clinical_record_openings,sigtap_procedures,ciap2_codes}_controller.rb`, `config/routes.rb` | rotas | 13 |
| `Gemfile`, `Gemfile.lock`, `app/services/consultations/print.rb`, `app/controllers/consultations_controller.rb` | impresso PDF | 14 |
| `db/city_migrate/20261007400003_add_ledi_correction_pending.rb`, `app/models/ledi_outbox_entry.rb`, `app/services/ledi/fichas/individual_care.rb` | ficha de Atendimento Individual | 15 |
| `app/services/ledi/{consultation_ficha,ficha_sources}.rb`, `app/jobs/ledi/consultation_ficha_job.rb`, `app/commands/ledi/resend.rb`, `app/controllers/production_controller.rb` | geração, regeneração, `correction_pending` | 16 |
| `spec/commands/patients/resolve_concurrency_spec.rb` | corrida no mesmo CPF | 17 |
| `spec/invariants/clinical_record_invariants_spec.rb` | invariantes do ADR 0031 | 18 |
| `lib/clinical_record_crew.rb`, `db/seeds.rb` | semente de dev | 19 |
| — | revisão final, suíte, porta 3036 | 20 |

---
## Fatia 0 — Pré-requisito e layout (F-19.6, pré-condição)

### Task 0: Conferir que o módulo 18 está na main

Nada deste plano compila sem o 18. Esta task não escreve código.

- [ ] **Step 1: Confira os nomes do 18 em `origin/main`**

```bash
/opt/homebrew/bin/git -C apps/api fetch origin
/opt/homebrew/bin/git -C apps/api log --oneline -1 origin/main
for f in app/services/screenings/vital_signs.rb app/services/screenings/ciap2.rb app/services/screenings/json.rb \
         app/services/ledi/screening_mapping.rb app/services/ledi/fichas/medicoes.rb app/services/ledi/fichas/screening_identity.rb \
         app/services/ledi/screening_ficha.rb app/models/ledi_generation_failure.rb app/controllers/ciap2_codes_controller.rb \
         spec/support/screening_helpers.rb db/city_migrate/20261007300002_ledi_outbox_error_codes.rb; do
  /opt/homebrew/bin/git -C apps/api cat-file -e "origin/main:$f" && echo "ok $f" || echo "FALTA $f"
done
/opt/homebrew/bin/git -C apps/api grep -n "def retry!\|def regenerate\|class AlreadyResolved" origin/main -- app/services/ledi/screening_ficha.rb
/opt/homebrew/bin/git -C apps/api grep -n "def call(ficha, city:, replaces: nil)" origin/main -- app/commands/ledi/enqueue.rb
```
Expected: todos `ok`; os três métodos de `Ledi::ScreeningFicha` e a assinatura do `Ledi::Enqueue` aparecem. **Pare** e reporte ao coordenador se algo `FALTA` (o 18 ainda não foi mergeado) ou se um nome mudou (ajuste este plano antes de seguir).

- [ ] **Step 2: Crie o worktree (Ambiente de execução) e confira a última migração de cidade**

Run: `ls apps/api/.claude/mod19/db/city_migrate | tail -2`
Expected: a última é `20261007300002_ledi_outbox_error_codes.rb`. Se houver mais nova, use números maiores que ela nas Tasks 3, 7 e 15 (e no `define(version:)`).

---

### Task 1: Confirmação do layout LEDI da consulta e `config/ledi/consultation_mapping.yml`

Nada de ficha é escrito antes desta task passar. Ela fixa, com a fonte de cada valor, o que as Tasks 8, 15 e 16 usam. **Critério de parada:** se qualquer confirmação do Step 2 contradisser a coluna "Esperado" (o desenho da spec §6 e o que foi lido ao escrever o plano), **pare**, não escreva o mapeamento e reporte ao coordenador com a URL e o trecho. Em particular: (a) `tipoAtendimento` sem `2` ou `5`; (b) `condutas` com outra faixa; (c) exame que não aceite SIGTAP do grupo 02 ou `solicitadoAvaliado` que não seja `S`; (d) `situacao` que não aceite 0 e 2, ou que exija campo que o projeto não tem; (e) **uma regra de CID-10 por CBO** que exclua médicos ou enfermeiros da tabela do MIAI (se a regra existir e só restringir, escreva `rule: prefixes` com a lista e a fonte — não é parada).

**Files:**
- Create: `config/ledi/consultation_mapping.yml`
- Create: `app/services/ledi/consultation_mapping.rb`
- Test: `spec/config/ledi/consultation_mapping_spec.rb`

**Interfaces:**
- Consumes: `Ledi::ScreeningMapping.miai_cbo?` (módulo 18), `Ledi::Version::ACTIVE`.
- Produces (`Ledi::ConsultationMapping`, módulo de funções, lê o YAML uma vez por processo):
  - `data`, `entry(key) -> { "value", "source" }`, `value(key)` (levanta `Ledi::ConsultationMapping::Missing`);
  - `care_types -> Array<{ code: Integer, label: String }>`, `care_type?(code) -> bool`, `care_type_label(code) -> String|nil`;
  - `conducts -> Array<{ code: Integer, label: String }>`, `conduct?(code) -> bool`, `conduct_label(code) -> String|nil`;
  - `situation(status) -> Integer` (`"active"` → 0, `"resolved"` → 2); `exam_requested -> "S"`; `exam_group_prefix -> "02"`; `local_de_atendimento -> 1`; `max_conducts -> 12`; `max_exams -> 100`;
  - `cid10_allowed?(cbo) -> bool` (regra `miai_table` ou `prefixes`).

- [ ] **Step 1: Leia os IDLs vendorizados**

```bash
cd apps/api/.claude/mod19
sed -n '/struct FichaAtendimentoIndividualChildThrift/,/^}/p' vendor/ledi/8.7.0/idl/ras/ficha_atendimento_individual.thrift
sed -n '/struct ExameThrift/,/^}/p;/struct ProblemaCondicaoThrift/,/^}/p;/struct EncaminhamentoExternoThrift/,/^}/p' vendor/ledi/8.7.0/idl/ras/common.thrift
```
Expected: o filho do MIAI tem `7:optional i64 tipoAtendimento`, `17:optional list<common.ExameThrift> exame`, `22:optional list<i64> condutas`, `28`/`29` `dataHoraInicialAtendimento`/`dataHoraFinalAtendimento`, `30:optional string cpfCidadao`, `39:optional common.MedicoesThrift medicoes`, `40:optional list<common.ProblemaCondicaoThrift> problemasCondicoes`, `47:optional bool stCidadaoNaoPossuiCpf`; `ExameThrift` tem `codigoExame`(1) e `solicitadoAvaliado`(2, `list<string>`); `ProblemaCondicaoThrift` tem `uuidProblema`(1), `uuidEvolucaoProblema`(2), `coSequencialEvolucao`(3), `ciap`(4), `cid10`(5), `situacao`(6, `i64`), `dataInicioProblema`(7), `dataFimProblema`(8), `isAvaliado`(9); `EncaminhamentoExternoThrift` exige `especialidade`(1).

- [ ] **Step 2: Confirme na documentação oficial (WebFetch, só leitura)**

Base: `https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao`.

| # | Pergunta | URL | Esperado (lido em 2026-10-07) |
|---|---|---|---|
| a | Tipos aceitos no MIAI | `.../estrutura_arquivos/dicionario-fai.html` (#7) e `.../referencias/dicionario.html` (TipoDeAtendimento) | só `1, 2, 4, 5, 6`; 1 "Consulta agendada programada / Cuidado continuado", 2 "Consulta agendada", 4 "Escuta inicial / Orientação", 5 "Consulta no dia", 6 "Atendimento de urgência" |
| b | Condutas | `dicionario-fai.html` (#22) e `dicionario.html` (CondutaEncaminhamento) | 1–12 por ficha, sem repetir; 4 obrigatória se houver `encaminhamentos`; códigos 1, 2, 4, 5, 6, 7, 8, 9, 10, 11, 12, 14 com os rótulos de "Valores fixados" |
| c | Exames | `dicionario-fai.html` (bloco Exame) e `dicionario.html` (SituacaoExame) | `codigoExame` SIGTAP sem pontuação do grupo `02 - Procedimentos com finalidade diagnóstica` (ou ListaExames), sem repetir, até 100; `solicitadoAvaliado` lista de 1–2 com `S` (Solicitado) e/ou `A` (Avaliado) |
| d | Problemas | `dicionario-fai.html` (bloco ProblemaCondicao) e `dicionario.html` (SituacaoProblemasCondicoes) | `ciap` ou `cid10` (um obrigatório na falta do outro), sem repetir; `situacao` 0 Ativo, 1 Latente, 2 Resolvido; `situacao`, `uuidProblema`, `uuidEvolucaoProblema`, `coSequencialEvolucao` se exigem mutuamente; `dataInicioProblema` ≥ nascimento e ≤ atendimento; `dataFimProblema` obrigatória com `situacao` 2; `isAvaliado` obrigatório; CID-10 da família Z34 no máximo uma |
| e | CID-10 por CBO | `dicionario-fai.html` (cid10 de ProblemaCondicao; hipoteseDiagnosticoCid10) e `.../regras/cbo.html` (Tabela 3) | **nenhuma** regra de CID-10 por CBO; só a Tabela 3 (CBOs do MIAI); a hipótese do encaminhamento exige CID-10 compatível com o sexo |
| f | Medições e identificação | os mesmos de `config/ledi/screening_mapping.yml` (módulo 18) | iguais: `MedicoesThrift`, sexo 0/1, turno 1/2/3, local 1 UBS, só CPF (`stCidadaoNaoPossuiCpf: false`) |

Anote no YAML (campo `source`) a URL e a seção de cada valor.

- [ ] **Step 3: Escreva a spec que falha**

```ruby
# spec/config/ledi/consultation_mapping_spec.rb
require "rails_helper"

# ADR 0031 / spec §6 (Tarefa 1): o mapeamento da consulta para o LEDI 8.7.0 tem
# fonte em cada linha e bate com os IDLs vendorizados.
RSpec.describe Ledi::ConsultationMapping do
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
    %w[care_types conducts cid10_cbos].each { |k| expect(described_class.data.dig(k, "source")).to be_present, k }
  end

  it "os campos usados existem nos IDLs" do
    expect(idl_fields("ras/ficha_atendimento_individual.thrift", "FichaAtendimentoIndividualChildThrift"))
      .to include("tipoAtendimento", "exame", "condutas", "problemasCondicoes", "medicoes", "cpfCidadao",
                  "stCidadaoNaoPossuiCpf", "dataHoraInicialAtendimento", "dataHoraFinalAtendimento")
    expect(idl_fields("ras/common.thrift", "ExameThrift")).to eq(%w[codigoExame solicitadoAvaliado])
    expect(idl_fields("ras/common.thrift", "ProblemaCondicaoThrift"))
      .to eq(%w[uuidProblema uuidEvolucaoProblema coSequencialEvolucao ciap cid10 situacao dataInicioProblema dataFimProblema isAvaliado])
  end

  it "tipos de atendimento da consulta: 1, 2, 5 e 6 (nunca 4, que é a escuta)" do
    expect(described_class.care_types.map { |t| t[:code] }).to eq([ 1, 2, 5, 6 ])
    expect(described_class.care_types).to all(include(:label))
    expect(described_class.care_type?(4)).to be(false)
    expect(described_class.care_type?("5")).to be(false) # só inteiro
    expect(described_class.care_type_label(2)).to eq("Consulta agendada")
  end

  it "condutas do dicionário, até 12" do
    expect(described_class.conducts.map { |c| c[:code] }).to eq([ 1, 2, 4, 5, 6, 7, 8, 9, 10, 11, 12, 14 ])
    expect(described_class.conduct?(3)).to be(false)
    expect(described_class.conduct_label(9)).to eq("Alta do episódio")
    expect(described_class.max_conducts).to eq(12)
  end

  it "situação, exame e local" do
    expect([ described_class.situation("active"), described_class.situation("resolved") ]).to eq([ 0, 2 ])
    expect([ described_class.exam_requested, described_class.exam_group_prefix, described_class.max_exams ]).to eq([ "S", "02", 100 ])
    expect(described_class.local_de_atendimento).to eq(1)
    expect(described_class.value("tipo_dado_serializado.atendimento_individual")).to eq(Ledi::FichaTypes.code("atendimento_individual"))
  end

  it "CID-10 pela regra do YAML: miai_table segue a tabela do MIAI; prefixes restringe" do
    expect(described_class.data.dig("cid10_cbos", "rule")).to eq("miai_table")
    expect(described_class.cid10_allowed?("225125")).to be(true)
    expect(described_class.cid10_allowed?("223505")).to eq(Ledi::ScreeningMapping.miai_cbo?("223505"))
    expect(described_class.cid10_allowed?("322205")).to be(false)
    restricted = described_class.data.merge("cid10_cbos" => { "rule" => "prefixes", "prefixes" => [ "2251" ], "source" => "x" })
    allow(described_class).to receive(:data).and_return(restricted)
    expect([ described_class.cid10_allowed?("225125"), described_class.cid10_allowed?("223505") ]).to eq([ true, false ])
  end

  it "chave inexistente levanta" do
    expect { described_class.value("nao.existe") }.to raise_error(described_class::Missing)
  end
end
```

- [ ] **Step 4: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/config/ledi/consultation_mapping_spec.rb`
Expected: FAIL com `uninitialized constant Ledi::ConsultationMapping`.

- [ ] **Step 5: Escreva o mapeamento (com a fonte de cada linha) e o leitor**

```yaml
# config/ledi/consultation_mapping.yml
# Consulta da APS → Ficha de Atendimento Individual (ADR 0031; spec 2026-10-07
# §6, Tarefa 1). Cada valor tem a fonte em que foi conferido. Mudar uma linha =
# reconferir a fonte e rodar spec/config/ledi/consultation_mapping_spec.rb.
# Base: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao
# Medições, sexo, turno e CPF: config/ledi/screening_mapping.yml (módulo 18).
ledi_version: "8.7.0"
checked_on: "2026-10-07"
entries:
  tipo_dado_serializado.atendimento_individual:
    value: 4
    source: "referencias/dicionario.html — TipoDadoSerializado 4 Modelo de Informação de Atendimento Individual"
  local_de_atendimento:
    value: 1
    source: "referencias/dicionario.html — LocalDeAtendimento 1 UBS"
  problem_situation.active:
    value: 0
    source: "referencias/dicionario.html — SituacaoProblemasCondicoes 0 Ativo"
  problem_situation.resolved:
    value: 2
    source: "referencias/dicionario.html — SituacaoProblemasCondicoes 2 Resolvido; dicionario-fai.html #8 dataFimProblema obrigatória com situacao 2"
  exam.requested:
    value: "S"
    source: "referencias/dicionario.html — SituacaoExame S Solicitado; dicionario-fai.html bloco Exame, solicitadoAvaliado lista de 1 a 2"
  exam.sigtap_group_prefix:
    value: "02"
    source: "dicionario-fai.html bloco Exame, codigoExame — só exames do grupo 02 Procedimentos com finalidade diagnóstica (ou ListaExames)"
  exam.max_per_ficha:
    value: 100
    source: "dicionario-fai.html #17 exame — máximo 100"
  conducts.max:
    value: 12
    source: "dicionario-fai.html #22 condutas — mínimo 1, máximo 12, sem repetir"
care_types:
  source: "dicionario-fai.html #7 tipoAtendimento — aceita 1, 2, 4, 5, 6; referencias/dicionario.html TipoDeAtendimento. 4 é a escuta inicial (módulo 18), fora da consulta"
  codes:
    - { code: 1, label: "Consulta agendada programada / Cuidado continuado" }
    - { code: 2, label: "Consulta agendada" }
    - { code: 5, label: "Consulta no dia" }
    - { code: 6, label: "Atendimento de urgência" }
conducts:
  source: "referencias/dicionario.html — CondutaEncaminhamento (tabela completa em 2026-10-07)"
  codes:
    - { code: 1, label: "Retorno para consulta agendada" }
    - { code: 2, label: "Retorno para cuidado continuado / programado" }
    - { code: 4, label: "Encaminhamento para serviço especializado" }
    - { code: 5, label: "Encaminhamento para CAPS" }
    - { code: 6, label: "Encaminhamento para internação hospitalar" }
    - { code: 7, label: "Encaminhamento para urgência" }
    - { code: 8, label: "Encaminhamento para serviço de atenção domiciliar" }
    - { code: 9, label: "Alta do episódio" }
    - { code: 10, label: "Encaminhamento intersetorial" }
    - { code: 11, label: "Encaminhamento interno no dia" }
    - { code: 12, label: "Agendamento para grupos" }
    - { code: 14, label: "Agendamento para eMulti" }
cid10_cbos:
  # miai_table: quem pode registrar o MIAI pode informar CID-10 (o dicionário
  # não restringe por CBO). prefixes: lista explícita em `prefixes`.
  rule: miai_table
  source: "dicionario-fai.html — cid10 (ProblemaCondicao #5) sem regra de CBO; regras/cbo.html Tabela 3 (CBOs do MIAI). Lido em 2026-10-07"
```

```ruby
# app/services/ledi/consultation_mapping.rb
# Mapeamento da consulta da APS para o LEDI (ADR 0031; spec §6, Tarefa 1). O
# YAML guarda valor e fonte de cada linha; quem valida itens e monta a ficha lê
# daqui, nunca de literal solto. Carregado uma vez por processo.
module Ledi
  module ConsultationMapping
    PATH = Rails.root.join("config/ledi/consultation_mapping.yml")

    class Missing < StandardError; end

    module_function

    def data = (@data ||= YAML.load_file(PATH).freeze)

    def entry(key) = data.fetch("entries").fetch(key.to_s) { raise Missing, key.to_s }

    def value(key) = entry(key).fetch("value")

    def care_types = coded("care_types")
    def conducts = coded("conducts")

    def care_type?(code) = code.is_a?(Integer) && care_types.any? { |t| t[:code] == code }
    def conduct?(code) = code.is_a?(Integer) && conducts.any? { |c| c[:code] == code }
    def care_type_label(code) = care_types.find { |t| t[:code] == code }&.dig(:label)
    def conduct_label(code) = conducts.find { |c| c[:code] == code }&.dig(:label)

    def situation(status) = value("problem_situation.#{status}")
    def exam_requested = value("exam.requested")
    def exam_group_prefix = value("exam.sigtap_group_prefix")
    def max_exams = value("exam.max_per_ficha")
    def max_conducts = value("conducts.max")
    def local_de_atendimento = value("local_de_atendimento")

    def cid10_allowed?(cbo)
      rule = data.fetch("cid10_cbos")
      case rule.fetch("rule")
      when "miai_table" then Ledi::ScreeningMapping.miai_cbo?(cbo)
      when "prefixes" then Array(rule["prefixes"]).any? { |prefix| cbo.to_s.start_with?(prefix) }
      else raise Missing, "cid10_cbos.rule"
      end
    end

    def coded(key)
      data.fetch(key).fetch("codes").map { |row| { code: row.fetch("code"), label: row.fetch("label") } }
    end
    private_class_method :coded
  end
end
```

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/config/ledi/consultation_mapping_spec.rb`
Expected: PASS (7 exemplos).

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add config/ledi/consultation_mapping.yml app/services/ledi/consultation_mapping.rb spec/config/ledi/consultation_mapping_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: confirm the LEDI layout for the primary care consultation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Interruptor `clinical_record` (requer `record_mode = record`)

**Files:**
- Modify: `app/services/platform/features.rb`
- Create: `app/services/clinical_record/gate.rb`, `app/controllers/concerns/clinical_record_gate.rb`
- Modify: `spec/adr_pointers_spec.rb`
- Test: `spec/services/platform/clinical_record_feature_spec.rb`

**Interfaces:**
- Produces: `Platform::Features::CATALOG` com `clinical_record` (`requires: ["record_mode:record"]`, ausência → `"record_mode_not_record"`); `ClinicalRecord::Gate::KEY == "clinical_record"`, `ClinicalRecord::Gate.usable?(city) -> bool` (nil ou cidade sem id → false); concern `ClinicalRecordGate` com `before_action :require_clinical_record!` (403 `{ error: "feature_disabled", feature: "clinical_record" }` quando não utilizável).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/platform/clinical_record_feature_spec.rb
require "rails_helper"

# ADR 0031 (spec §4, §10): o prontuário fica atrás do interruptor
# clinical_record, que só é utilizável no modo record. O maintenance liga pelo
# mecanismo genérico (catálogo), sem código novo.
RSpec.describe "Interruptor clinical_record" do
  let(:city) { register_test_city! }

  def set(enabled) = Platform::Features.set!(city: city, key: "clinical_record", enabled: enabled, maintainer: ledi_maintainer!)

  it "está no catálogo e exige record_mode = record" do
    expect(Platform::Features::KEYS).to include("clinical_record")
    set(true)
    { "off" => [ "record_mode_not_record" ], "integrated" => [ "record_mode_not_record" ], "record" => [] }.each do |mode, missing|
      city.update!(record_mode: mode)
      expect(Platform::Features.missing(city, "clinical_record")).to eq(missing), mode
      expect(ClinicalRecord::Gate.usable?(city)).to eq(missing.empty?), mode
    end
  end

  it "desligado não é utilizável; sem cidade também não" do
    city.update!(record_mode: "record")
    expect(ClinicalRecord::Gate.usable?(city)).to be(false)
    expect(ClinicalRecord::Gate.usable?(nil)).to be(false)
    expect(ClinicalRecord::Gate.usable?(TEST_CITY_A)).to be(false) # sem linha na plataforma
  end

  it "o resumo do maintenance mostra o interruptor e o que falta" do
    set(true)
    row = Platform::Features.summary(city).find { |r| r[:key] == "clinical_record" }
    expect(row).to include(enabled: true, usable: false, missing: [ "record_mode_not_record" ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/platform/clinical_record_feature_spec.rb`
Expected: FAIL (`interruptor fora do catálogo: clinical_record` / `uninitialized constant ClinicalRecord`).

- [ ] **Step 3: Implemente**

Em `app/services/platform/features.rb`, acrescente ao `CATALOG` (depois de `cadsus_lookup`):

```ruby
      # ADR 0031: prontuário da APS (módulo 19a), só no modo record.
      Entry.new(key: "clinical_record",
                description: "Prontuário da atenção primária (consulta SOAP, lista de problemas, adendos)",
                requires: %w[record_mode:record])
```

e, em `missing_for`, logo depois da linha `when "record_mode" …`:

```ruby
      when "record_mode:record" then "record_mode_not_record" unless platform[:record_mode] == "record"
```

```ruby
# app/services/clinical_record/gate.rb
# O prontuário só existe com o interruptor clinical_record LIGADO e
# UTILIZÁVEL (ADR 0031; modo record). Relido da plataforma a cada chamada
# (Platform::Features relê record_mode por id).
module ClinicalRecord
  module Gate
    KEY = "clinical_record".freeze

    module_function

    def usable?(city)
      return false if city.nil? || city.id.nil?

      Platform::Features.usable?(city, KEY)
    end
  end
end
```

```ruby
# app/controllers/concerns/clinical_record_gate.rb
# Rotas do prontuário (ADR 0031; contratos): interruptor desligado ou sem
# record_mode = record → 403 feature_disabled, antes de qualquer leitura.
module ClinicalRecordGate
  extend ActiveSupport::Concern

  private

  def require_clinical_record!
    return if ClinicalRecord::Gate.usable?(Current.city)

    render json: { error: "feature_disabled", feature: ClinicalRecord::Gate::KEY }, status: :forbidden
  end
end
```

Em `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..31).freeze`.

- [ ] **Step 4: Rode e veja passar (e as specs do catálogo e do maintenance)**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/platform spec/requests/maintenance spec/adr_pointers_spec.rb`
Expected: PASS. Se alguma spec do maintenance fixa a lista de chaves do catálogo (`%w[ledi_export cadsus_lookup]`), acrescente `clinical_record` (é o contrato: o mecanismo é genérico).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add app/services/platform/features.rb app/services/clinical_record/gate.rb app/controllers/concerns/clinical_record_gate.rb spec/adr_pointers_spec.rb spec/services/platform/clinical_record_feature_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: add the clinical_record switch that requires record mode

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Se alguma spec do maintenance mudou no Step 4, inclua o caminho no `git add`.)

---
## Fatia 1 — Paciente, nome e lista de problemas (F-19.1, F-19.2)

### Task 3: Migração do paciente, nomes em `citizens`, lista de problemas por eventos; modelos, cifra, eventos e helpers

**Files:**
- Create: `db/city_migrate/20261007400001_add_patients.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`
- Create: `app/models/patient.rb`, `app/models/patient_problem.rb`, `app/models/patient_problem_event.rb`, `app/models/patient_profile_divergence.rb`
- Modify: `app/models/citizen.rb`, `app/services/city_encryption.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `config/initializers/filter_parameter_logging.rb`
- Create: `spec/support/clinical_record_helpers.rb`; Modify: `spec/rails_helper.rb`
- Test: `spec/models/patient_tables_guard_spec.rb`

**Interfaces:**
- Produces:
  - Colunas: `citizens.{full_name,social_name,mother_name}` (texto cifrado), `citizens.patient_id` (FK `patients`); tabelas `patients (cpf único, full_name, social_name, mother_name, birth_date, sex, timestamps)`, `patient_profile_divergences (patient_id, citizen_id, fields text[], created_at)`, `patient_problems (patient_id, terminology, code, terminology_release_id, status, onset_on, onset_precision, resolved_on, timestamps)`, `patient_problem_events (patient_problem_id, kind, consultation_id, addendum_id, user_id, status_after, onset_on, onset_precision, resolved_on, terminology_release_id, txid, created_at)`.
  - Triggers: `patients_guard`, `citizens_patient_link_guard`, `patient_problems_guard` (com a exigência de evento), `patient_problem_events_append_only` (+ TRUNCATE), `patient_profile_divergences_append_only` (+ TRUNCATE).
  - Modelos: `Patient` (`#display_name`, `#cpf_masked`, `#age(on:)`, `has_many :citizens, :problems`); `PatientProblem` (`TERMINOLOGIES`, `STATUSES`, `PRECISIONS`, `scope :active_problems`); `PatientProblemEvent` (`KINDS`); `PatientProfileDivergence` (`FIELDS`); `Citizen#display_name`, `Citizen::NAME_LIMITS`, `belongs_to :patient (opcional)`.
  - Helpers de spec (`spec/support/clinical_record_helpers.rb`): `ClinicalRecordHelpers::CID10`, `::SIGTAP`, `cid10_release!`, `sigtap_release!(competence = …)`, `clinical_city!(record_mode: "record", enabled: true) -> City`, `verifier!`, `verified_citizen!(n, full_name:, social_name:, mother_name:, age:, sex:)`, `doctor!(unit, cbo: "225125")`, `consulting_attendance!(unit, citizen:, doctor:)`, `started_consultation!(unit:, doctor:, citizen:)`, `draft_body(**over)`, `finalized_consultation!(unit:, doctor:, citizen:, outcome: …, **over)`, `capture_log { }`.

- [ ] **Step 1: Helpers de spec**

```ruby
# spec/support/clinical_record_helpers.rb
# Módulo 19 (ADR 0031): terminologias da plataforma (CID-10 e SIGTAP mínimas),
# a cidade com o prontuário ligado, pares validados com nome e o caminho até a
# consulta. Cenário de teste: os caminhos reais são Terminology::Import,
# Citizens::Verify e Attendances::Call.
module ClinicalRecordHelpers
  CID10 = {
    "E119" => [ "Diabetes mellitus não-insulino-dependente - sem complicações", nil ],
    "I10" => [ "Hipertensão essencial (primária)", nil ],
    "N390" => [ "Infecção do trato urinário de localização não especificada", nil ],
    "C61" => [ "Neoplasia maligna da próstata", "M" ]
  }.freeze
  SIGTAP = {
    "0202010503" => "DOSAGEM DE HEMOGLOBINA GLICOSILADA",
    "0202010317" => "DOSAGEM DE CREATININA",
    "0301010064" => "CONSULTA MEDICA EM ATENCAO PRIMARIA"
  }.freeze

  def cid10_release!
    TerminologyRelease.active.find_by(kind: "cid10") || begin
      release = TerminologyRelease.create!(kind: "cid10", version: "2008", source_sha256: "d" * 64,
                                           imported_by: "rspec", imported_at: Time.current, status: "importing")
      CID10.each { |code, (description, sex)| Cid10Code.create!(release: release, code: code, description: description, sex_restriction: sex) }
      release.update!(status: "active", activated_at: Time.current)
      release
    end
  end

  def sigtap_release!(competence = Time.zone.today.strftime("%Y%m"))
    TerminologyRelease.active.find_by(kind: "sigtap", version: competence) || begin
      release = TerminologyRelease.create!(kind: "sigtap", version: competence, source_sha256: "e" * 64,
                                           imported_by: "rspec", imported_at: Time.current, status: "importing")
      SIGTAP.each { |code, name| SigtapProcedure.create!(release: release, code: code, name: name) }
      release.update!(status: "active", activated_at: Time.current)
      release
    end
  end

  # A linha da TEST_CITY_A na plataforma (mesma chave de cifra do banco de
  # teste, como use_test_city_host!), com o modo e o interruptor pedidos.
  def clinical_city!(record_mode: "record", enabled: true)
    city = City.find_by(slug: TEST_CITY_A.slug) ||
           City.create!(slug: TEST_CITY_A.slug, name: TEST_CITY_A.name, status: "active", time_zone: "America/Sao_Paulo",
                        database_url: TEST_CITY_A.database_url, encryption_key: TEST_CITY_A.encryption_key,
                        schema_version: CitySchema.expected_version.to_s)
    city.update!(record_mode: record_mode)
    Platform::Features.set!(city: city, key: "clinical_record", enabled: enabled, maintainer: ledi_maintainer!)
    CityCatalog.reset_cache!
    city
  end

  def verifier! = (@verifier ||= staff_with("validador-#{SecureRandom.hex(3)}@cidade.gov.br", "citizen_verifier"))

  # Par validado com nome (o caminho real é Citizens::Verify).
  def verified_citizen!(n, full_name: "Maria Aparecida da Silva", social_name: nil, mother_name: "Joana da Silva",
                        age: 40 + n, sex: "female")
    citizen = screening_citizen!(n, age: age, sex: sex)
    CitizenVerification.create!(citizen: citizen, verified_by_user: verifier!, verified_at: Time.current)
    citizen.update!(verification_level: "verified", profile_source: "verified", full_name: full_name,
                    social_name: social_name, mother_name: mother_name)
    citizen
  end

  def doctor!(unit, cbo: "225125") = screener!(unit, cbo: cbo)

  def consulting_attendance!(unit, citizen:, doctor:)
    in_care!(walk_in_attendance!(unit, citizen: citizen), by: doctor)
  end

  def started_consultation!(unit:, doctor:, citizen:)
    attendance = consulting_attendance!(unit, citizen: citizen, doctor: doctor)
    result = Consultations::Start.call(attendance: attendance, by: doctor)
    raise "consulta não iniciou: #{result.reason}" if result.failure?

    result.payload.fetch(:consultation)
  end

  def draft_body(**over)
    { "subjective" => "Refere sede e poliúria há dois meses", "objective" => "Bom estado geral",
      "assessment" => "Diabetes mellitus tipo 2", "plan" => "Metformina 500 mg; retorno em 30 dias",
      "vitals" => { "systolic" => 130, "diastolic" => 85, "weight_kg" => "82.5", "height_cm" => 170 },
      "care_type" => 5,
      "evaluated_problems" => [ { "terminology" => "ciap2", "code" => "T90", "action" => "add",
                                  "onset_on" => "2025-08-01", "onset_precision" => "month" } ],
      "conducts" => [ 1 ], "exam_requests" => [ { "sigtap_code" => "0202010503" } ] }.merge(over.transform_keys(&:to_s))
  end

  def finalized_consultation!(unit:, doctor:, citizen:, outcome: { "outcome" => "discharged" }, **over)
    consultation = started_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    saved = Consultations::SaveDraft.call(consultation: consultation, params: draft_body(**over), by: doctor)
    raise "rascunho recusado: #{saved.reason} #{saved.details}" if saved.failure?

    result = Consultations::Finalize.call(consultation: consultation.reload, outcome_params: outcome, by: doctor)
    raise "finalização recusada: #{result.reason} #{result.details}" if result.failure?

    consultation.reload
  end

  def capture_log
    log = StringIO.new
    capture = ActiveSupport::Logger.new(log).tap { |l| l.level = Logger::DEBUG }
    Rails.logger.broadcast_to(capture)
    yield
    log.string
  ensure
    Rails.logger.stop_broadcasting_to(capture)
  end
end

RSpec.configure { |c| c.include ClinicalRecordHelpers }
```

Em `spec/rails_helper.rb`, depois de `require_relative "support/screening_helpers"`, acrescente `require_relative "support/clinical_record_helpers"`. (`Consultations::*` só existe a partir das Tasks 9–10; os helpers que os usam só são chamados por specs dessas tasks em diante.)

- [ ] **Step 2: Escreva a spec de guarda (falha: tabelas e colunas não existem)**

```ruby
# spec/models/patient_tables_guard_spec.rb
require "rails_helper"

# Módulo 19 (ADR 0031; spec §3): o banco garante o que o modelo não vê — par
# declarado nunca se liga, a ligação não troca, a lista só muda com evento da
# mesma transação, eventos e divergências são só acréscimo, sem dois ativos iguais.
RSpec.describe "Guardas das tabelas do paciente" do
  before { Current.city = TEST_CITY_A; ciap2_release! }
  after { Current.reset }

  let(:doctor) { User.create!(email_address: "medica-#{SecureRandom.hex(3)}@cidade.gov.br", password: "senha-segura-123") }
  let(:citizen) { verified_citizen!(1) }
  let(:patient) { Patient.create!(cpf: citizen.cpf, full_name: "Maria Aparecida da Silva") }

  def attempt(&) = ApplicationRecord.transaction(requires_new: true, &)

  # Evento com uma consulta qualquer (a FK para consultations nasce na Task 7).
  def event!(problem_id, kind:, status_after:, **values)
    PatientProblemEvent.create!({ patient_problem_id: problem_id, kind: kind, consultation_id: SecureRandom.uuid,
                                  user: doctor, status_after: status_after }.merge(values))
  end

  def problem!(code: "T90", status: "active", **values)
    id = SecureRandom.uuid
    event!(id, kind: "added", status_after: status, **values)
    PatientProblem.create!({ id: id, patient: patient, terminology: "ciap2", code: code, status: status,
                             terminology_release_id: TerminologyRelease.active.find_by!(kind: "ciap2").id }.merge(values))
  end

  describe "citizens" do
    it "nome cifrado com a chave da cidade; nome de exibição prefere o social" do
      citizen.update!(social_name: "Mariana")
      raw = ApplicationRecord.connection.select_value("SELECT full_name FROM citizens WHERE id = #{ApplicationRecord.connection.quote(citizen.id)}")
      expect(raw).not_to include("Maria")
      expect(citizen.reload.display_name).to eq("Mariana")
      citizen.update!(social_name: nil)
      expect(citizen.display_name).to eq("Maria Aparecida da Silva")
    end

    it "par declarado nunca se liga; a ligação nunca troca" do
      declared = screening_citizen!(2)
      expect { attempt { declared.update_columns(patient_id: patient.id) } }
        .to raise_error(ActiveRecord::StatementInvalid, /only a verified pair links to a patient/)
      citizen.update_columns(patient_id: patient.id)
      other = Patient.create!(cpf: screening_citizen!(3).cpf)
      expect { attempt { citizen.update_columns(patient_id: other.id) } }
        .to raise_error(ActiveRecord::StatementInvalid, /patient link never changes/)
    end

    it "revogar a validação não desliga o par (ADR 0031)" do
      citizen.update_columns(patient_id: patient.id)
      expect { citizen.update!(verification_level: "declared") }.not_to raise_error
      expect(citizen.reload.patient_id).to eq(patient.id)
    end
  end

  describe "patients" do
    it "um por CPF; nunca some; CPF nunca muda" do
      patient
      expect { attempt { Patient.create!(cpf: citizen.cpf) } }.to raise_error(ActiveRecord::RecordNotUnique)
      expect { attempt { patient.delete } }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
      expect { attempt { patient.update_columns(cpf: "11144477735") } }
        .to raise_error(ActiveRecord::StatementInvalid, /identity columns never change/)
    end
  end

  describe "patient_problems" do
    it "só nasce e só muda com evento da mesma transação e com os mesmos valores" do
      expect do
        attempt do
          PatientProblem.create!(patient: patient, terminology: "ciap2", code: "K86", status: "active",
                                 terminology_release_id: TerminologyRelease.active.find_by!(kind: "ciap2").id)
        end
      end.to raise_error(ActiveRecord::StatementInvalid, /changes only through an event/)

      problem = problem!
      expect { attempt { problem.update!(status: "resolved", resolved_on: Time.zone.today) } }
        .to raise_error(ActiveRecord::StatementInvalid, /changes only through an event/)
      expect do
        attempt do
          event!(problem.id, kind: "resolved", status_after: "active")
          problem.update!(status: "resolved", resolved_on: Time.zone.today)
        end
      end.to raise_error(ActiveRecord::StatementInvalid, /changes only through an event/)

      event!(problem.id, kind: "resolved", status_after: "resolved", resolved_on: Time.zone.today)
      expect { problem.update!(status: "resolved", resolved_on: Time.zone.today) }.not_to raise_error
    end

    it "sem dois ativos iguais por paciente; identidade fixa; DELETE recusado; CHECKs" do
      problem = problem!
      expect { attempt { problem!(code: "T90") } }.to raise_error(ActiveRecord::RecordNotUnique)
      expect { attempt { problem.update_columns(code: "K86") } }
        .to raise_error(ActiveRecord::StatementInvalid, /identity columns never change/)
      expect { attempt { problem.delete } }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
      {
        { code: "t90" } => /ck_patient_problems_code/,
        { status: "resolved" } => /ck_patient_problems_resolution/,
        { onset_on: Date.new(2025, 1, 1) } => /ck_patient_problems_onset/,
        { onset_on: Date.new(2025, 1, 1), onset_precision: "week" } => /ck_patient_problems_onset_precision/
      }.each do |values, error|
        expect { attempt { problem!(code: "K86", **values) } }.to raise_error(ActiveRecord::StatementInvalid, error), values.inspect
      end
    end
  end

  describe "patient_problem_events e patient_profile_divergences" do
    it "só acréscimo; consulta XOR adendo; tipo da lista" do
      problem = problem!
      event = PatientProblemEvent.where(patient_problem_id: problem.id).sole
      expect { attempt { event.update_columns(kind: "resolved") } }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { attempt { event.delete } }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect do
        attempt do
          PatientProblemEvent.create!(patient_problem_id: problem.id, kind: "added", user: doctor, status_after: "active")
        end
      end.to raise_error(ActiveRecord::StatementInvalid, /ck_patient_problem_events_source/)
      expect { attempt { event!(problem.id, kind: "edited", status_after: "active") } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_patient_problem_events_kind/)
      divergence = PatientProfileDivergence.create!(patient: patient, citizen: citizen, fields: %w[sex])
      expect { attempt { divergence.delete } }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { attempt { PatientProfileDivergence.create!(patient: patient, citizen: citizen, fields: %w[cpf]) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_patient_profile_divergences_fields/)
    end
  end
end
```

- [ ] **Step 3: Escreva a migração**

```ruby
# db/city_migrate/20261007400001_add_patients.rb
# Módulo 19 (ADR 0031; spec 2026-10-07 §3): nome completo, social e da mãe no
# par (cifrados), o paciente por CPF ligado aos pares validados, divergências
# de perfil entre pares e a lista de problemas como resultado de eventos só de
# acréscimo. As FKs dos eventos para consulta/adendo nascem na 20261007400002.
# Triggers em db/city_triggers.sql.
class AddPatients < ActiveRecord::Migration[8.1]
  def up
    create_table :patients, id: :uuid do |t|
      t.string :cpf, null: false
      t.text :full_name
      t.text :social_name
      t.text :mother_name
      t.text :birth_date
      t.text :sex
      t.timestamps
    end
    add_index :patients, :cpf, unique: true

    add_column :citizens, :full_name, :text
    add_column :citizens, :social_name, :text
    add_column :citizens, :mother_name, :text
    add_column :citizens, :patient_id, :uuid
    add_index :citizens, :patient_id
    add_foreign_key :citizens, :patients

    create_table :patient_profile_divergences, id: :uuid do |t|
      t.uuid :patient_id, null: false
      t.uuid :citizen_id, null: false
      t.text :fields, array: true, null: false
      t.datetime :created_at, null: false
    end
    add_index :patient_profile_divergences, :patient_id
    add_index :patient_profile_divergences, :citizen_id
    add_foreign_key :patient_profile_divergences, :patients
    add_foreign_key :patient_profile_divergences, :citizens
    add_check_constraint :patient_profile_divergences,
                         "cardinality(fields) > 0 AND fields <@ ARRAY['birth_date'::text, 'sex'::text]",
                         name: "ck_patient_profile_divergences_fields"

    create_table :patient_problems, id: :uuid do |t|
      t.uuid :patient_id, null: false
      t.string :terminology, null: false
      t.string :code, limit: 4, null: false
      t.uuid :terminology_release_id, null: false
      t.string :status, null: false
      t.date :onset_on
      t.string :onset_precision
      t.date :resolved_on
      t.timestamps
    end
    add_index :patient_problems, :patient_id
    add_index :patient_problems, %i[patient_id terminology code], unique: true, where: "((status)::text = 'active'::text)",
                                                                   name: "idx_patient_problems_one_active"
    add_foreign_key :patient_problems, :patients
    {
      "ck_patient_problems_terminology" => "terminology::text = ANY (ARRAY['ciap2'::text, 'cid10'::text])",
      "ck_patient_problems_status" => "status::text = ANY (ARRAY['active'::text, 'resolved'::text])",
      "ck_patient_problems_code" => "(terminology::text = 'ciap2'::text AND code::text ~ '^[A-Z][0-9]{2}$'::text) OR " \
                                    "(terminology::text = 'cid10'::text AND code::text ~ '^[A-Z][0-9]{2}[0-9X]?$'::text)",
      "ck_patient_problems_onset" => "(onset_on IS NULL) = (onset_precision IS NULL)",
      "ck_patient_problems_onset_precision" => "onset_precision IS NULL OR onset_precision::text = ANY (ARRAY['day'::text, 'month'::text, 'year'::text])",
      "ck_patient_problems_resolution" => "(status::text = 'resolved'::text) = (resolved_on IS NOT NULL)"
    }.each { |name, expression| add_check_constraint :patient_problems, expression, name: name }

    create_table :patient_problem_events, id: :uuid do |t|
      t.uuid :patient_problem_id, null: false
      t.string :kind, null: false
      t.uuid :consultation_id
      t.uuid :addendum_id
      t.uuid :user_id, null: false
      t.string :status_after, null: false
      t.date :onset_on
      t.string :onset_precision
      t.date :resolved_on
      t.uuid :terminology_release_id
      t.bigint :txid, null: false, default: -> { "txid_current()" }
      t.datetime :created_at, null: false
    end
    add_index :patient_problem_events, :patient_problem_id
    add_index :patient_problem_events, :consultation_id
    add_index :patient_problem_events, :addendum_id
    add_index :patient_problem_events, :user_id
    add_foreign_key :patient_problem_events, :patient_problems, deferrable: :deferred
    add_foreign_key :patient_problem_events, :users
    {
      "ck_patient_problem_events_kind" => "kind::text = ANY (ARRAY['added'::text, 'resolved'::text, 'reactivated'::text, 'onset_corrected'::text])",
      "ck_patient_problem_events_source" => "(consultation_id IS NULL) <> (addendum_id IS NULL)",
      "ck_patient_problem_events_status" => "status_after::text = ANY (ARRAY['active'::text, 'resolved'::text])"
    }.each { |name, expression| add_check_constraint :patient_problem_events, expression, name: name }

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

- [ ] **Step 4: Triggers (no fim de `db/city_triggers.sql`)**

```sql
-- patients (ADR 0031; spec 2026-10-07 §3): um por CPF; nunca some; a
-- identidade não muda. Nome, nascimento e sexo seguem o par validado mais
-- recente (Patients::Resolve).
CREATE OR REPLACE FUNCTION rota_patient_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'patients is append-only: DELETE refused';
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id OR NEW.cpf IS DISTINCT FROM OLD.cpf OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'patients: identity columns never change';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- citizens.patient_id (ADR 0031, Invariantes): só par validado se liga a
-- paciente; ligado, nunca troca nem desliga. A revogação (verification_level
-- volta a declared) não mexe na ligação: "revogação não afeta o prontuário".
CREATE OR REPLACE FUNCTION rota_citizen_patient_link_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'UPDATE' AND OLD.patient_id IS NOT NULL AND NEW.patient_id IS DISTINCT FROM OLD.patient_id THEN
    RAISE EXCEPTION 'citizens: the patient link never changes';
  END IF;
  IF NEW.patient_id IS NOT NULL AND (TG_OP = 'INSERT' OR OLD.patient_id IS NULL)
     AND NEW.verification_level IS DISTINCT FROM 'verified' THEN
    RAISE EXCEPTION 'citizens: only a verified pair links to a patient';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- patient_problems (ADR 0031, Invariantes): o estado só nasce e só muda com
-- um evento da MESMA transação com os mesmos valores novos
-- (Patients::ApplyProblemEvent grava o evento antes; a FK do evento é
-- DEFERRABLE). Identidade fixa; nunca some.
CREATE OR REPLACE FUNCTION rota_patient_problem_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'patient_problems is append-only: DELETE refused';
  END IF;
  IF TG_OP = 'UPDATE' AND (NEW.id IS DISTINCT FROM OLD.id OR NEW.patient_id IS DISTINCT FROM OLD.patient_id
     OR NEW.terminology IS DISTINCT FROM OLD.terminology OR NEW.code IS DISTINCT FROM OLD.code
     OR NEW.created_at IS DISTINCT FROM OLD.created_at) THEN
    RAISE EXCEPTION 'patient_problems: identity columns never change';
  END IF;
  IF NOT EXISTS (
    SELECT 1 FROM patient_problem_events e
    WHERE e.patient_problem_id = NEW.id AND e.txid = txid_current()
      AND e.status_after = NEW.status
      AND e.onset_on IS NOT DISTINCT FROM NEW.onset_on
      AND e.onset_precision IS NOT DISTINCT FROM NEW.onset_precision
      AND e.resolved_on IS NOT DISTINCT FROM NEW.resolved_on) THEN
    RAISE EXCEPTION 'patient_problems: changes only through an event of the same transaction';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF to_regclass('public.patients') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS patients_guard ON patients';
    EXECUTE 'CREATE TRIGGER patients_guard
      BEFORE UPDATE OR DELETE ON patients
      FOR EACH ROW EXECUTE FUNCTION rota_patient_guard()';
    EXECUTE 'DROP TRIGGER IF EXISTS citizens_patient_link_guard ON citizens';
    EXECUTE 'CREATE TRIGGER citizens_patient_link_guard
      BEFORE INSERT OR UPDATE ON citizens
      FOR EACH ROW EXECUTE FUNCTION rota_citizen_patient_link_guard()';
  END IF;
  IF to_regclass('public.patient_problems') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS patient_problems_guard ON patient_problems';
    EXECUTE 'CREATE TRIGGER patient_problems_guard
      BEFORE INSERT OR UPDATE OR DELETE ON patient_problems
      FOR EACH ROW EXECUTE FUNCTION rota_patient_problem_guard()';
  END IF;
  IF to_regclass('public.patient_problem_events') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS patient_problem_events_append_only ON patient_problem_events';
    EXECUTE 'CREATE TRIGGER patient_problem_events_append_only
      BEFORE UPDATE OR DELETE ON patient_problem_events
      FOR EACH ROW EXECUTE FUNCTION rota_append_only()';
    EXECUTE 'DROP TRIGGER IF EXISTS patient_problem_events_append_only_truncate ON patient_problem_events';
    EXECUTE 'CREATE TRIGGER patient_problem_events_append_only_truncate
      BEFORE TRUNCATE ON patient_problem_events
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
  END IF;
  IF to_regclass('public.patient_profile_divergences') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS patient_profile_divergences_append_only ON patient_profile_divergences';
    EXECUTE 'CREATE TRIGGER patient_profile_divergences_append_only
      BEFORE UPDATE OR DELETE ON patient_profile_divergences
      FOR EACH ROW EXECUTE FUNCTION rota_append_only()';
    EXECUTE 'DROP TRIGGER IF EXISTS patient_profile_divergences_append_only_truncate ON patient_profile_divergences';
    EXECUTE 'CREATE TRIGGER patient_profile_divergences_append_only_truncate
      BEFORE TRUNCATE ON patient_profile_divergences
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
  END IF;
END
$do$;
```

(`rota_append_only` levanta `'% is append-only: % refused'` — as expectativas `/append-only/` e `/DELETE refused/` dependem disso.)

- [ ] **Step 5: Dump à mão em `db/city_schema.rb`**

Troque `define(version: 2026_10_07_300002)` por `define(version: 2026_10_07_400001)`. Em `citizens`, acrescente (ordem alfabética das colunas): `t.text "full_name"`, `t.text "mother_name"`, `t.uuid "patient_id"`, `t.text "social_name"` e o índice `t.index ["patient_id"], name: "index_citizens_on_patient_id"`. Tabelas novas (em ordem alfabética entre as existentes):

```ruby
  create_table "patient_problem_events", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "addendum_id"
    t.uuid "consultation_id"
    t.datetime "created_at", null: false
    t.string "kind", null: false
    t.date "onset_on"
    t.string "onset_precision"
    t.uuid "patient_problem_id", null: false
    t.date "resolved_on"
    t.string "status_after", null: false
    t.uuid "terminology_release_id"
    t.bigint "txid", default: -> { "txid_current()" }, null: false
    t.uuid "user_id", null: false
    t.index ["addendum_id"], name: "index_patient_problem_events_on_addendum_id"
    t.index ["consultation_id"], name: "index_patient_problem_events_on_consultation_id"
    t.index ["patient_problem_id"], name: "index_patient_problem_events_on_patient_problem_id"
    t.index ["user_id"], name: "index_patient_problem_events_on_user_id"
    t.check_constraint "kind::text = ANY (ARRAY['added'::text, 'resolved'::text, 'reactivated'::text, 'onset_corrected'::text])", name: "ck_patient_problem_events_kind"
    t.check_constraint "(consultation_id IS NULL) <> (addendum_id IS NULL)", name: "ck_patient_problem_events_source"
    t.check_constraint "status_after::text = ANY (ARRAY['active'::text, 'resolved'::text])", name: "ck_patient_problem_events_status"
  end

  create_table "patient_problems", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.string "code", limit: 4, null: false
    t.datetime "created_at", null: false
    t.date "onset_on"
    t.string "onset_precision"
    t.uuid "patient_id", null: false
    t.date "resolved_on"
    t.string "status", null: false
    t.string "terminology", null: false
    t.uuid "terminology_release_id", null: false
    t.datetime "updated_at", null: false
    t.index ["patient_id", "terminology", "code"], name: "idx_patient_problems_one_active", unique: true, where: "((status)::text = 'active'::text)"
    t.index ["patient_id"], name: "index_patient_problems_on_patient_id"
    t.check_constraint "(terminology::text = 'ciap2'::text AND code::text ~ '^[A-Z][0-9]{2}$'::text) OR (terminology::text = 'cid10'::text AND code::text ~ '^[A-Z][0-9]{2}[0-9X]?$'::text)", name: "ck_patient_problems_code"
    t.check_constraint "(onset_on IS NULL) = (onset_precision IS NULL)", name: "ck_patient_problems_onset"
    t.check_constraint "onset_precision IS NULL OR onset_precision::text = ANY (ARRAY['day'::text, 'month'::text, 'year'::text])", name: "ck_patient_problems_onset_precision"
    t.check_constraint "(status::text = 'resolved'::text) = (resolved_on IS NOT NULL)", name: "ck_patient_problems_resolution"
    t.check_constraint "status::text = ANY (ARRAY['active'::text, 'resolved'::text])", name: "ck_patient_problems_status"
    t.check_constraint "terminology::text = ANY (ARRAY['ciap2'::text, 'cid10'::text])", name: "ck_patient_problems_terminology"
  end

  create_table "patient_profile_divergences", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "citizen_id", null: false
    t.datetime "created_at", null: false
    t.text "fields", null: false, array: true
    t.uuid "patient_id", null: false
    t.index ["citizen_id"], name: "index_patient_profile_divergences_on_citizen_id"
    t.index ["patient_id"], name: "index_patient_profile_divergences_on_patient_id"
    t.check_constraint "cardinality(fields) > 0 AND fields <@ ARRAY['birth_date'::text, 'sex'::text]", name: "ck_patient_profile_divergences_fields"
  end

  create_table "patients", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.text "birth_date"
    t.string "cpf", null: false
    t.datetime "created_at", null: false
    t.text "full_name"
    t.text "mother_name"
    t.text "sex"
    t.text "social_name"
    t.datetime "updated_at", null: false
    t.index ["cpf"], name: "index_patients_on_cpf", unique: true
  end
```

Nas `add_foreign_key` do fim, em ordem alfabética: `add_foreign_key "citizens", "patients"`, `add_foreign_key "patient_problem_events", "patient_problems", deferrable: :deferred`, `add_foreign_key "patient_problem_events", "users"`, `add_foreign_key "patient_problems", "patients"`, `add_foreign_key "patient_profile_divergences", "citizens"`, `add_foreign_key "patient_profile_divergences", "patients"`.

- [ ] **Step 6: Modelos e cifra**

```ruby
# app/models/patient.rb
# O paciente do prontuário (ADR 0031; spec §3): um por CPF, criado na primeira
# consulta de um par VALIDADO; os pares validados do CPF se ligam a ele. Nome,
# nascimento e sexo vêm do par validado mais recente (Patients::Resolve). Tudo
# cifrado com a chave da cidade; o CPF determinístico (é chave de busca).
class Patient < ApplicationRecord
  encrypts :cpf, deterministic: true, key_provider: CityDeterministicKeyProvider.new
  encrypts :full_name
  encrypts :social_name
  encrypts :mother_name
  encrypts :birth_date
  encrypts :sex

  has_many :citizens, dependent: :restrict_with_error
  has_many :problems, class_name: "PatientProblem", dependent: :restrict_with_error

  validates :cpf, presence: true

  def display_name = social_name.presence || full_name

  def cpf_masked = CitizenIdentity::Cpf.mask(cpf)

  def age(on: Time.zone.today)
    return nil if birth_date.blank?

    Citizen.age_between(Date.iso8601(birth_date), on)
  rescue Date::Error
    nil
  end
end
```

```ruby
# app/models/patient_problem.rb
# Estado atual de um problema do paciente (ADR 0031; spec §3). Resultado de
# patient_problem_events: só Patients::ApplyProblemEvent escreve aqui, e o
# trigger patient_problems_guard exige o evento na mesma transação.
class PatientProblem < ApplicationRecord
  TERMINOLOGIES = %w[ciap2 cid10].freeze
  STATUSES = %w[active resolved].freeze
  PRECISIONS = %w[day month year].freeze

  belongs_to :patient
  has_many :events, class_name: "PatientProblemEvent", dependent: :restrict_with_error

  scope :active_problems, -> { where(status: "active") }

  def active? = status == "active"
end
```

```ruby
# app/models/patient_problem_event.rb
# Um evento da lista de problemas (ADR 0031; spec §3): só acréscimo, com a
# consulta OU o adendo, o profissional e os valores novos. O txid amarra o
# evento à transação que muda o estado (trigger).
class PatientProblemEvent < ApplicationRecord
  KINDS = %w[added resolved reactivated onset_corrected].freeze

  belongs_to :patient_problem
  belongs_to :user
end
```

```ruby
# app/models/patient_profile_divergence.rb
# Dois pares validados do mesmo CPF com nascimento ou sexo diferentes (ADR
# 0031; spec §3). Só acréscimo; o paciente segue o par validado mais recente.
class PatientProfileDivergence < ApplicationRecord
  FIELDS = %w[birth_date sex].freeze

  belongs_to :patient
  belongs_to :citizen
end
```

Em `app/models/citizen.rb`, depois de `encrypts :cadsus_pending_cns`:

```ruby
  # ADR 0031 (spec §3): nome conferido no documento na validação presencial.
  # Cifrados com a chave da cidade, não determinísticos (nenhum é chave de
  # busca). Nome de exibição = social, se houver; senão o completo.
  encrypts :full_name
  encrypts :social_name
  encrypts :mother_name
  NAME_LIMITS = { "full_name" => 3..200, "social_name" => 1..200, "mother_name" => 1..200 }.freeze

  # ADR 0031: o paciente do CPF, só para par validado (trigger).
  belongs_to :patient, optional: true

  def display_name = social_name.presence || full_name
```

Em `app/services/city_encryption.rb`, no fim de `CITY_KEYED_TARGETS` (antes do `].freeze`):

```ruby
    [ LediOutboxEntry, :payload ],
    # ADR 0031: nome no par e o paciente.
    [ Citizen,        :full_name ],
    [ Citizen,        :social_name ],
    [ Citizen,        :mother_name ],
    [ Patient,        :cpf ],
    [ Patient,        :full_name ],
    [ Patient,        :social_name ],
    [ Patient,        :mother_name ],
    [ Patient,        :birth_date ],
    [ Patient,        :sex ]
```

(a linha `[ LediOutboxEntry, :payload ]` já existe; só ganha a vírgula.)

- [ ] **Step 7: Eventos e filtro de log**

Em `config/initializers/domain_events.rb`, no fim do bloco:

```ruby
  # Módulo 19 (ADR 0031; contratos §7): prontuário; só trilha, só ids. Nenhum
  # texto clínico, nota de abertura ou nome entra em evento.
  DomainEvents.bind "patient.created", to: []
  DomainEvents.bind "patient.linked", to: []
  DomainEvents.bind "patient_problem.changed", to: []
  DomainEvents.bind "consultation.started", to: []
  DomainEvents.bind "consultation.finalized", to: []
  DomainEvents.bind "consultation.addendum_added", to: []
  DomainEvents.bind "clinical_record.viewed", to: []
  DomainEvents.bind "clinical_record.opened", to: []
```

Em `spec/initializers/domain_events_bindings_spec.rb`:

```ruby
# Módulo 19 (ADR 0031): prontuário, só trilha.
RSpec.describe "clinical record event bindings (ADR 0031)" do
  it "declares every module 19 city event with no consumer" do
    names = %w[patient.created patient.linked patient_problem.changed consultation.started consultation.finalized
               consultation.addendum_added clinical_record.viewed clinical_record.opened]
    expect(DomainEvents.registry.keys).to include(*names)
    expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
  end
end
```

Em `config/initializers/filter_parameter_logging.rb`, depois de `/\Aq\z/` (com vírgula nele):

```ruby
  # ADR 0031: texto clínico da consulta e do adendo, e os nomes da pessoa.
  # (`:reason` e `:note` já cobrem o motivo do adendo e a nota da abertura.)
  :subjective, :objective, :assessment, :plan, :text, :full_name, :social_name, :mother_name
```

- [ ] **Step 8: Rode a migração nos bancos de teste e as specs**

```bash
psql -U rota_saude -d postgres -c "DROP DATABASE rota_saude_test_city_a" -c "DROP DATABASE rota_saude_test_city_b"
docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod19 api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/models/patient_tables_guard_spec.rb spec/services/city_schema_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/architecture spec/models spec/commands/citizens
```
Expected: PASS. Se `city_schema_spec` acusar diferença, copie a expressão da migração (não a do erro); `deferrable: :deferred` precisa estar no dump.

- [ ] **Step 9: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add db/city_migrate/20261007400001_add_patients.rb db/city_schema.rb db/city_triggers.sql app/models/patient.rb app/models/patient_problem.rb app/models/patient_problem_event.rb app/models/patient_profile_divergence.rb app/models/citizen.rb app/services/city_encryption.rb config/initializers/domain_events.rb config/initializers/filter_parameter_logging.rb spec/initializers/domain_events_bindings_spec.rb spec/support/clinical_record_helpers.rb spec/rails_helper.rb spec/models/patient_tables_guard_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: add patients, names on citizens and the event-sourced problem list

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 4: Nomes na validação presencial (verificar, completar no check-in, fila)

Decisões do coordenador (contrato §9) já aplicadas aqui: completar os nomes de par já validado acontece **no check-in** (`check_ins/lookup` traz `citizen.names` e `citizen.verification_id`; `check_ins` devolve `verification_id` quando valida); `POST /attendance/verifications` **sem a chave** `full_name` continua aceito (cliente antigo, compatibilidade de deploy) e não grava nome — a consulta depois para em `patient_name_missing` até completar; **com a chave**, `full_name` é obrigatório (3–200); todo item da fila ganha `display_name`.

**Files:**
- Create: `app/commands/citizens/name_values.rb`, `app/commands/citizens/complete_names.rb`, `app/commands/citizens/names_json.rb`
- Modify: `app/commands/citizens/verify.rb`, `app/controllers/attendance_controller.rb`, `app/controllers/check_ins_controller.rb`, `app/controllers/attendances_controller.rb`, `config/routes.rb`
- Test: `spec/commands/citizens/name_values_spec.rb`, `spec/requests/attendance_names_spec.rb`; Modify: `spec/commands/citizens/verify_spec.rb`

**Interfaces:**
- Consumes: `Citizen::NAME_LIMITS`, `Citizen#display_name` (Task 3).
- Produces:
  - `Citizens::NameValues::ABSENT` (marcador de "chave ausente"), `.call(full_name:, social_name: nil, mother_name: nil) -> Result` (ok `{ full_name:, social_name:, mother_name: }` com espaços normalizados, opcionais vazios → nil; com `full_name: ABSENT`, ok `{}` — nada a gravar; falhas `:invalid_full_name`, `:invalid_social_name`, `:invalid_mother_name`);
  - `Citizens::Verify.call(..., full_name: Citizens::NameValues::ABSENT, social_name: nil, mother_name: nil, ...)` (nome conferido junto do perfil, antes de consumir o código);
  - `Citizens::CompleteNames.call(verification:, full_name:, social_name:, mother_name:, by:) -> Result` (ok `{ citizen: }`; falhas de `NameValues` e `:already_revoked`);
  - `Citizens::NamesJson.call(citizen) -> { full_name_set: bool, display_name: String|nil }`;
  - `POST /attendance/verifications` aceita `full_name`, `social_name`, `mother_name`; `POST /attendance/lookup` e `POST /attendance/check_ins/lookup` devolvem `citizen.names`; `check_ins/lookup` também `citizen.verification_id` (validação ativa ou `null`); `POST /attendance/check_ins` devolve `verification_id` quando valida; `POST /attendance/verifications/:id/names` (`:id` = a validação ativa) → `{ citizen: { id, cpf_masked, verification_level, names } }`; item da fila ganha `display_name`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/commands/citizens/name_values_spec.rb
require "rails_helper"

# ADR 0031 (contratos §2 e §9): com a chave, nome completo 3–200 obrigatório;
# social e mãe opcionais até 200; sem a chave (cliente antigo), nada a gravar.
RSpec.describe Citizens::NameValues do
  def call(**names) = described_class.call(full_name: "Maria Aparecida da Silva", **names)

  it "normaliza e aceita opcionais vazios" do
    result = call(full_name: "  Maria   Aparecida  da Silva ", social_name: " ", mother_name: "Joana  da Silva")
    expect(result.payload).to eq(full_name: "Maria Aparecida da Silva", social_name: nil, mother_name: "Joana da Silva")
  end

  it "sem a chave: ok e nada a gravar (mesmo com social ou mãe)" do
    expect(described_class.call(full_name: described_class::ABSENT, social_name: "Mariana").payload).to eq({})
  end

  {
    { full_name: nil } => :invalid_full_name, { full_name: "Ma" } => :invalid_full_name,
    { full_name: "x" * 201 } => :invalid_full_name, { full_name: [ "Maria" ] } => :invalid_full_name,
    { social_name: "x" * 201 } => :invalid_social_name, { social_name: { "a" => 1 } } => :invalid_social_name,
    { mother_name: 42 } => :invalid_mother_name
  }.each do |names, reason|
    it("#{names.inspect.truncate(60)} → #{reason}") { expect(call(**names).reason).to eq(reason) }
  end
end
```

```ruby
# spec/requests/attendance_names_spec.rb
require "rails_helper"

# ADR 0031 (spec §3; contratos §2 e §9): nomes conferidos no documento na
# validação presencial; cliente antigo sem a chave continua validando;
# completar nomes de par já validado no check-in; a fila só com o nome de
# exibição. Nome nunca em log nem em evento.
RSpec.describe "Nomes na validação presencial", type: :request do
  let(:unit) { create_unit }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:marker) { "Marcadora #{SecureRandom.hex(3)}" }
  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]

  def verify!(code, **names)
    json_post "/attendance/verifications", { cpf: citizen.cpf, code: code, document_checked: true,
                                            birth_date: "1980-05-10", sex: "female" }.merge(names)
  end

  before { sign_in_as(verifier!) }

  it "valida com nomes (cifrados); nome de exibição é o social; nada em log nem evento" do
    log = capture_log do
      verify!(issue_code_for(citizen), full_name: "#{marker} da Silva", social_name: "Mariana", mother_name: "Joana")
    end
    expect(response).to have_http_status(:created)
    expect(citizen.reload.slice(:full_name, :social_name, :mother_name))
      .to eq("full_name" => "#{marker} da Silva", "social_name" => "Mariana", "mother_name" => "Joana")
    expect(citizen.display_name).to eq("Mariana")
    expect(log).not_to include(marker)
    expect(DomainEvent.pluck(:payload).to_json).not_to include(marker)
  end

  it "com a chave, nome inválido é 422 e o código continua valendo; sem a chave (cliente antigo), valida sem nome" do
    code = issue_code_for(citizen)
    verify!(code, full_name: nil)
    expect(status_and_error).to eq([ 422, "invalid_full_name" ])
    verify!(code, full_name: "Maria", social_name: "x" * 201)
    expect(status_and_error).to eq([ 422, "invalid_social_name" ])
    verify!(code, full_name: "Maria Aparecida", mother_name: [ "Joana" ])
    expect(status_and_error).to eq([ 422, "invalid_mother_name" ])
    verify!(code)
    expect(response).to have_http_status(:created)
    expect(citizen.reload.full_name).to be_nil
  end

  it "cartão do balcão e do check-in: nomes sem os valores e a validação ativa" do
    json_post "/attendance/lookup", cpf: citizen.cpf, code: issue_code_for(citizen)
    expect(body.dig("citizen", "names")).to eq("full_name_set" => false, "display_name" => nil)

    validated = verified_citizen!(8, full_name: "Maria Aparecida da Silva", social_name: nil)
    validated.update_columns(full_name: nil)
    triage = completed_web_triage_for(validated)
    code = Citizens::IssueCheckInCode.call(citizen: validated, triage: triage).payload.fetch(:code)
    json_post "/attendance/check_ins/lookup", cpf: validated.cpf, code: code, health_unit_id: unit.id
    expect(body["citizen"]).to include("names" => { "full_name_set" => false, "display_name" => nil },
                                       "verification_id" => validated.active_verification.id)
  end

  it "check-in que valida devolve a validação, para completar os nomes na sequência" do
    triage = completed_web_triage_for(citizen)
    code = Citizens::IssueCheckInCode.call(citizen: citizen, triage: triage).payload.fetch(:code)
    json_post "/attendance/check_ins", cpf: citizen.cpf, code: code, health_unit_id: unit.id, document_checked: true
    expect(response).to have_http_status(:created)
    expect(body["verification_id"]).to eq(citizen.reload.active_verification.id)
  end

  it "completar nomes: só citizen_verifier; revogada 409; inválido 422; inexistente 404" do
    verification = CitizenVerification.create!(citizen: citizen, verified_by_user: verifier!, verified_at: Time.current)
    citizen.update!(verification_level: "verified")
    json_post "/attendance/verifications/#{verification.id}/names", full_name: "Maria Aparecida da Silva", social_name: "Mariana"
    expect(response).to have_http_status(:ok)
    expect(body["citizen"]).to include("id" => citizen.id, "cpf_masked" => citizen.cpf_masked,
                                       "names" => { "full_name_set" => true, "display_name" => "Mariana" })
    expect(body.to_json).not_to include("Aparecida")
    json_post "/attendance/verifications/#{verification.id}/names", full_name: ""
    expect(status_and_error).to eq([ 422, "invalid_full_name" ])
    json_post "/attendance/verifications/#{SecureRandom.uuid}/names", full_name: "Maria Aparecida"
    expect(status_and_error).to eq([ 404, "not_found" ])
    verification.update!(revoked_at: Time.current, revoked_by_user: staff_with("adm-#{SecureRandom.hex(3)}@x.gov.br", "municipal_admin"),
                         revoke_reason: "documento de outra pessoa")
    json_post "/attendance/verifications/#{verification.id}/names", full_name: "Maria Aparecida"
    expect(status_and_error).to eq([ 409, "already_revoked" ])
    sign_in_as(staff_with("medica-#{SecureRandom.hex(3)}@x.gov.br", "health_professional"))
    json_post "/attendance/verifications/#{verification.id}/names", full_name: "Maria Aparecida"
    expect(response).to have_http_status(:forbidden)
  end

  it "todo item da fila traz só o nome de exibição (recepção incluída)" do
    verified = verified_citizen!(7, full_name: "Maria Aparecida da Silva", social_name: "Mariana")
    walk_in_attendance!(unit, citizen: verified)
    sign_in_as(reception!)
    get "/attendance/units/#{unit.id}/queue"
    expect(body["waiting"].sole["display_name"]).to eq("Mariana")
    expect(response.body).not_to include("Aparecida", "Joana")
  end
end
```

(Se `Citizens::IssueCheckInCode` tiver outro nome de chave no payload, use o de `spec/support/appointment_helpers.rb#waiting_attendance`, que faz o mesmo caminho.)

Em `spec/commands/citizens/verify_spec.rb`, acrescente:

```ruby
  describe "nomes conferidos no documento (ADR 0031)" do
    it "grava os nomes cifrados; nome inválido não gasta o código; sem a chave, valida sem nome" do
      code = issue_code_for(citizen)
      expect(verify(code, full_name: "x").reason).to eq(:invalid_full_name)
      expect(verify(code, full_name: "Maria Aparecida da Silva", social_name: "Mariana")).to be_ok
      expect(citizen.reload.slice(:full_name, :social_name, :mother_name))
        .to eq("full_name" => "Maria Aparecida da Silva", "social_name" => "Mariana", "mother_name" => nil)
    end
  end
```

(O helper `verify` existente não passa `full_name`: o cliente antigo continua validando — as specs existentes de validação não mudam.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/commands/citizens/name_values_spec.rb spec/requests/attendance_names_spec.rb spec/commands/citizens/verify_spec.rb`
Expected: FAIL (`uninitialized constant Citizens::NameValues`; `No route matches`).

- [ ] **Step 3: Os comandos**

```ruby
# app/commands/citizens/name_values.rb
# Nome conferido no documento (ADR 0031; contratos §2 e §9): com a chave,
# completo 3–200 obrigatório; social e da mãe opcionais, até 200. Espaços
# normalizados; texto vazio = não informado. Sem a chave (ABSENT; cliente
# antigo durante o deploy), nada a gravar — a consulta pede o nome depois.
module Citizens
  module NameValues
    FIELDS = %w[full_name social_name mother_name].freeze
    ABSENT = Object.new.freeze

    module_function

    def call(full_name:, social_name: nil, mother_name: nil)
      return Result.ok({}) if full_name.equal?(ABSENT)

      raw = { "full_name" => full_name, "social_name" => social_name, "mother_name" => mother_name }
      values = {}
      FIELDS.each do |field|
        value = raw[field]
        return invalid(field) unless value.nil? || value.is_a?(String)

        text = value.to_s.squish
        if field == "full_name"
          return invalid(field) unless Citizen::NAME_LIMITS[field].cover?(text.length)
        elsif text.length > Citizen::NAME_LIMITS[field].max
          return invalid(field)
        end
        values[field.to_sym] = text.presence
      end
      Result.ok(values)
    end

    def invalid(field) = Result.fail(:"invalid_#{field}")
    private_class_method :invalid
  end
end
```

```ruby
# app/commands/citizens/names_json.rb
# Os nomes como o balcão os vê (contratos §2): se o completo existe e o de
# exibição — nunca os três valores para quem só faz balcão.
module Citizens
  module NamesJson
    module_function

    def call(citizen) = { full_name_set: citizen.full_name.present?, display_name: citizen.display_name }
  end
end
```

```ruby
# app/commands/citizens/complete_names.rb
# Completar os nomes de par já validado (ADR 0031; contratos §2 e §9; spec
# §12: pares validados antes do deploy ou por cliente antigo). Feito no
# check-in, com a validação ativa. O paciente ligado passa a usar os nomes
# (Patients::Resolve.refresh!, Task 5).
module Citizens
  module CompleteNames
    module_function

    def call(verification:, full_name:, social_name:, mother_name:, by:)
      names = NameValues.call(full_name: full_name, social_name: social_name, mother_name: mother_name)
      return names if names.failure?

      ApplicationRecord.transaction do
        verification.lock!
        next Result.fail(:already_revoked) unless verification.active?

        citizen = verification.citizen
        citizen.lock!
        citizen.update!(names.payload)
        DomainEvents.publish("citizen.profile_changed", citizen_id: citizen.id)
        Result.ok(citizen: citizen)
      end
    end
  end
end
```

(Rota de completar sem a chave `full_name` chega com `full_name: nil` do controller — e é 422: completar exige o nome.)

Em `app/commands/citizens/verify.rb`:
- assinatura: `def self.call(cpf:, code:, document_checked:, by:, birth_date:, sex:, full_name: NameValues::ABSENT, social_name: nil, mother_name: nil, gender_identity: UNCHANGED, cadsus_confirmed: false, session_id: nil)`;
- logo depois de `return values if values.failure?`:

```ruby
      # ADR 0031: o nome conferido no documento, também antes de gastar o
      # código. Sem a chave (cliente antigo), nada a gravar.
      names = NameValues.call(full_name: full_name, social_name: social_name, mother_name: mother_name)
      return names if names.failure?
```
- logo depois de `apply_profile!(citizen, values.payload, keep_identity: keep_identity)`: `citizen.update!(names.payload) if names.payload.any?`;
- no comentário do topo: `# ADR 0031: também o nome completo (com a chave, obrigatório), o social e o da mãe.`

- [ ] **Step 4: Controllers e rota**

Em `app/controllers/attendance_controller.rb`:
- no comentário do topo: `#   POST /attendance/verifications/:id/names  {full_name, social_name?, mother_name?}  citizen_verifier`;
- `ERROR_STATUS` ganha `invalid_full_name: :unprocessable_entity, invalid_social_name: :unprocessable_entity, invalid_mother_name: :unprocessable_entity` (`already_revoked: :conflict` já existe);
- `before_action :require_verifier, only: %i[lookup verify cadsus_lookup names]` e o `rate_limit` com `only: %i[lookup verify cadsus_lookup names]`;
- em `lookup`, no hash do `citizen`: `names: Citizens::NamesJson.call(citizen),`;
- em `verify`, na chamada: `full_name: params.key?(:full_name) ? params[:full_name] : Citizens::NameValues::ABSENT, social_name: params[:social_name], mother_name: params[:mother_name],`;
- a ação nova:

```ruby
  # ADR 0031 (contratos §2 e §9): completar os nomes de par já validado (no
  # check-in). :id é a validação ativa.
  def names
    verification = CitizenVerification.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless verification

    result = Citizens::CompleteNames.call(verification: verification, full_name: params[:full_name],
                                          social_name: params[:social_name], mother_name: params[:mother_name],
                                          by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    citizen = result.payload[:citizen]
    render json: { citizen: { id: citizen.id, cpf_masked: citizen.cpf_masked, verification_level: citizen.verification_level,
                              names: Citizens::NamesJson.call(citizen) } }
  end
```

Em `app/controllers/check_ins_controller.rb`:
- em `lookup`, no hash do `citizen`: `names: Citizens::NamesJson.call(citizen), verification_id: citizen.active_verification&.id`;
- em `create`, troque o `render` por:

```ruby
    json = { attendance: attendance_json(result.payload[:attendance]), verified: result.payload[:verified] }
    # ADR 0031 (contrato §9): validou agora → o balcão completa os nomes com esta validação.
    json[:verification_id] = result.payload[:attendance].citizen.active_verification&.id if result.payload[:verified]
    render json: json, status: :created
```

Em `config/routes.rb`, no `scope "/attendance"`, depois de `post "verifications/:id/revoke", …`: `post "verifications/:id/names", to: "attendance#names"`.

Em `app/controllers/attendances_controller.rb#queue_json`, acrescente ao hash: `display_name: a.citizen.display_name,` (ADR 0031: a fila mostra só o nome de exibição; todo item).

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/commands/citizens spec/requests/attendance_names_spec.rb spec/requests/attendance_spec.rb spec/requests/attendance_cadsus_spec.rb spec/requests/check_ins_spec.rb spec/requests/citizen_api spec/invariants spec/commands/attendances spec/requests/attendances_spec.rb`
Expected: PASS. Se alguma spec fixa as chaves do item da fila, do cartão do balcão ou do check-in (ex.: `spec/requests/attendance_contract_spec.rb`), acrescente `display_name`/`names`/`verification_id` (é o contrato) e inclua o arquivo no `git add`.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add app/commands/citizens/name_values.rb app/commands/citizens/names_json.rb app/commands/citizens/complete_names.rb app/commands/citizens/verify.rb app/controllers/attendance_controller.rb app/controllers/check_ins_controller.rb app/controllers/attendances_controller.rb config/routes.rb spec/commands/citizens/name_values_spec.rb spec/requests/attendance_names_spec.rb spec/commands/citizens/verify_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: check full, social and mother names at the in-person verification

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: `Patients::Resolve` — paciente por CPF, ligação dos pares validados e divergência

**Files:**
- Create: `app/commands/patients/resolve.rb`
- Modify: `app/commands/citizens/verify.rb`, `app/commands/citizens/complete_names.rb`
- Test: `spec/commands/patients/resolve_spec.rb`

**Interfaces:**
- Consumes: `Patient`, `PatientProfileDivergence`, `Citizen#active_verification` (Tasks 3–4).
- Produces: `Patients::Resolve.call(citizen) -> Result` (ok `{ patient:, created: bool }`; falha `:citizen_not_verified`); `Patients::Resolve.refresh!(patient) -> Patient` (dentro de transação, com o paciente travado); `Patients::Resolve.lock_key(cpf) -> Integer`. Eventos `patient.created` / `patient.linked` `{ patient_id, citizen_id }`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/commands/patients/resolve_spec.rb
require "rails_helper"

# ADR 0031 (spec §3): o paciente nasce na primeira consulta de um par
# VALIDADO; os pares validados do CPF se ligam a ele; o perfil segue o par
# validado mais recente; divergência de nascimento/sexo fica registrada.
RSpec.describe Patients::Resolve do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:citizen) { verified_citizen!(1, full_name: "Maria Aparecida da Silva", social_name: "Mariana") }

  def second_pair(of, phone:, **profile)
    other = Citizen.create!({ cpf: of.cpf, phone: phone, birth_date: of.birth_date, sex: of.sex, profile_source: "verified" }.merge(profile))
    CitizenVerification.create!(citizen: other, verified_by_user: verifier!, verified_at: Time.current)
    other.update!(verification_level: "verified", full_name: "Maria A. da Silva")
    other
  end

  it "cria o paciente do CPF com nome, nascimento e sexo do par; liga o par; evento só com ids" do
    result = described_class.call(citizen)
    expect(result).to be_ok
    patient = result.payload[:patient]
    expect(result.payload[:created]).to be(true)
    expect(patient.slice(:cpf, :full_name, :social_name, :mother_name, :birth_date, :sex))
      .to eq("cpf" => citizen.cpf, "full_name" => "Maria Aparecida da Silva", "social_name" => "Mariana",
             "mother_name" => "Joana da Silva", "birth_date" => citizen.birth_date, "sex" => "female")
    expect(citizen.reload.patient_id).to eq(patient.id)
    expect(DomainEvent.where(name: "patient.created").sole.payload).to eq("patient_id" => patient.id, "citizen_id" => citizen.id)
  end

  it "de novo: o mesmo paciente, sem evento novo" do
    first = described_class.call(citizen).payload[:patient]
    again = described_class.call(citizen.reload)
    expect([ again.payload[:patient].id, again.payload[:created] ]).to eq([ first.id, false ])
    expect(DomainEvent.where(name: %w[patient.created patient.linked]).count).to eq(1)
  end

  it "par declarado nunca é ligado nem cria paciente" do
    declared = screening_citizen!(2)
    expect(described_class.call(declared).reason).to eq(:citizen_not_verified)
    expect([ Patient.count, declared.reload.patient_id ]).to eq([ 0, nil ])
  end

  it "segundo par validado do CPF liga ao mesmo paciente; o perfil segue o mais recente; divergência uma vez" do
    patient = described_class.call(citizen).payload[:patient]
    other = travel_to(1.minute.from_now) { second_pair(citizen, phone: "+5541990000099", sex: "male") }
    result = described_class.call(other)
    expect(result.payload[:patient].id).to eq(patient.id)
    expect(DomainEvent.where(name: "patient.linked").sole.payload).to eq("patient_id" => patient.id, "citizen_id" => other.id)
    expect(patient.reload.sex).to eq("male")
    expect(PatientProfileDivergence.sole).to have_attributes(patient_id: patient.id, citizen_id: citizen.id, fields: %w[sex])
    described_class.call(other.reload)
    expect(PatientProfileDivergence.count).to eq(1)
  end

  it "o nome vem do par validado mais recente que TEM nome (par antigo sem nome não apaga)" do
    patient = described_class.call(citizen).payload[:patient]
    other = travel_to(1.minute.from_now) { second_pair(citizen, phone: "+5541990000098") }
    other.update_columns(full_name: nil)
    described_class.call(other.reload)
    expect(patient.reload.full_name).to eq("Maria Aparecida da Silva")
  end

  it "revogar a validação não desliga; consulta nova pede validar de novo" do
    described_class.call(citizen)
    citizen.active_verification.update!(revoked_at: Time.current, revoke_reason: "documento rasurado",
                                        revoked_by_user: staff_with("adm-#{SecureRandom.hex(3)}@x.gov.br", "municipal_admin"))
    citizen.update!(verification_level: "declared")
    expect(citizen.reload.patient_id).to be_present
    expect(described_class.call(citizen).reason).to eq(:citizen_not_verified)
  end

  it "completar nomes de par ligado atualiza o paciente" do
    patient = described_class.call(citizen).payload[:patient]
    Citizens::CompleteNames.call(verification: citizen.active_verification, full_name: "Maria Aparecida Souza",
                                 social_name: nil, mother_name: nil, by: verifier!)
    expect(patient.reload.slice(:full_name, :social_name)).to eq("full_name" => "Maria Aparecida Souza", "social_name" => nil)
  end

  it "a chave do lock não contém o CPF" do
    expect(described_class.lock_key(citizen.cpf).to_s).not_to include(citizen.cpf)
    expect(described_class.lock_key(citizen.cpf)).to eq(described_class.lock_key(citizen.cpf))
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/commands/patients/resolve_spec.rb`
Expected: FAIL com `uninitialized constant Patients::Resolve`.

- [ ] **Step 3: Implemente**

```ruby
# app/commands/patients/resolve.rb
# Paciente do par (ADR 0031; spec §3): exige par VALIDADO; acha ou cria o
# paciente pelo CPF e liga o par. Duas consultas do mesmo CPF ao mesmo tempo
# (pares diferentes) criam UM paciente: lock consultivo por CPF (a chave é um
# hash — nunca o CPF no SQL nem no log), com o índice único de patients.cpf
# como última palavra. Ordem de travas de quem chama: atendimento → CPF →
# par → paciente. O perfil (nascimento, sexo) segue o par validado mais
# recente; o nome, o par validado mais recente que tem nome.
module Patients
  module Resolve
    module_function

    def call(citizen)
      return Result.fail(:citizen_not_verified) unless eligible?(citizen)

      ApplicationRecord.transaction do
        ApplicationRecord.connection.select_value("SELECT pg_advisory_xact_lock(#{lock_key(citizen.cpf)})")
        citizen.lock!
        next Result.fail(:citizen_not_verified) unless eligible?(citizen)

        patient = citizen.patient_id ? Patient.lock.find(citizen.patient_id) : Patient.lock.find_by(cpf: citizen.cpf)
        created = patient.nil?
        patient ||= Patient.create!(cpf: citizen.cpf)
        unless citizen.patient_id == patient.id
          citizen.update!(patient_id: patient.id)
          DomainEvents.publish(created ? "patient.created" : "patient.linked", patient_id: patient.id, citizen_id: citizen.id)
        end
        refresh!(patient)
        Result.ok(patient: patient, created: created)
      end
    end

    # Recalcula nome e perfil a partir dos pares validados ligados e registra
    # divergência de nascimento/sexo (uma vez por par e campos).
    def refresh!(patient)
      linked = Citizen.not_erased.verification_level_verified.where(patient_id: patient.id).includes(:verifications).to_a
      return patient if linked.empty?

      latest = linked.max_by { |c| recency(c) }
      named = linked.select { |c| c.full_name.present? }.max_by { |c| recency(c) }
      values = { birth_date: latest.birth_date, sex: latest.sex }
      values.merge!(full_name: named.full_name, social_name: named.social_name, mother_name: named.mother_name) if named
      patient.update!(values) if values.any? { |key, value| patient.public_send(key) != value }
      (linked - [ latest ]).each { |other| record_divergence!(patient, other, latest) }
      patient
    end

    def lock_key(cpf) = Digest::SHA256.hexdigest("patients:#{cpf}")[0, 15].to_i(16)

    def eligible?(citizen) = citizen.verification_level_verified? && citizen.erased_at.nil?

    def recency(citizen)
      verified_at = citizen.verifications.select(&:active?).map(&:verified_at).max
      [ verified_at || Time.zone.at(0), citizen.updated_at ]
    end

    def record_divergence!(patient, other, reference)
      fields = PatientProfileDivergence::FIELDS.select { |field| other.public_send(field) != reference.public_send(field) }
      return if fields.empty?
      return if PatientProfileDivergence.where(patient_id: patient.id, citizen_id: other.id)
                                        .where("fields = ARRAY[?]::text[]", fields).exists?

      PatientProfileDivergence.create!(patient: patient, citizen: other, fields: fields)
    end
    private_class_method :eligible?, :recency, :record_divergence!
  end
end
```

Em `app/commands/citizens/verify.rb`, logo depois de `citizen.update!(names.payload)`:

```ruby
        # ADR 0031: par revalidado já ligado a paciente — nome e perfil seguem.
        Patients::Resolve.refresh!(Patient.lock.find(citizen.patient_id)) if citizen.patient_id
```

Em `app/commands/citizens/complete_names.rb`, logo depois de `citizen.update!(names.payload)`:

```ruby
        Patients::Resolve.refresh!(Patient.lock.find(citizen.patient_id)) if citizen.patient_id
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/commands/patients spec/commands/citizens spec/requests/attendance_names_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add app/commands/patients/resolve.rb app/commands/citizens/verify.rb app/commands/citizens/complete_names.rb spec/commands/patients/resolve_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: resolve the patient by CPF from verified pairs only

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: `Patients::ApplyProblemEvent` — o único caminho de escrita da lista de problemas

**Files:**
- Create: `app/commands/patients/apply_problem_event.rb`, `app/services/patients/problem_replay.rb`
- Modify: `app/models/patient_problem_event.rb` (`belongs_to :patient_problem, optional: true`)
- Test: `spec/commands/patients/apply_problem_event_spec.rb`

**Interfaces:**
- Consumes: `PatientProblem`, `PatientProblemEvent` (Task 3).
- Produces:
  - `Patients::ApplyProblemEvent::ACTIONS == %w[evaluate add resolve correct_onset]`;
  - `.call(patient:, action:, by:, source:, terminology: nil, code: nil, release_id: nil, problem: nil, onset_on: nil, onset_precision: nil, on: Time.zone.today) -> Result` — `source` é `{ consultation: <obj com id> }` ou `{ addendum: <obj com id> }`; ok `{ problem:, event: PatientProblemEvent|nil }` (nil = avaliação sem mudança); falha `:invalid_problem`. Quem chama está numa transação e travou o paciente;
  - `Patients::ProblemReplay.state(problem) -> { status:, onset_on:, onset_precision:, resolved_on: }|nil`, `.history(problem) -> Array<Hash>`;
  - evento `patient_problem.changed { patient_problem_id, kind, consultation_id | addendum_id }`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/commands/patients/apply_problem_event_spec.rb
require "rails_helper"

# ADR 0031 (spec §3, §8): a lista muda só por evento ligado a consulta ou
# adendo; os eventos reconstroem o estado; sem dois ativos iguais; incluir o
# que foi resolvido reativa a mesma linha.
RSpec.describe Patients::ApplyProblemEvent do
  before { Current.city = TEST_CITY_A; ciap2_release!; cid10_release! }
  after { Current.reset }

  let(:doctor) { User.create!(email_address: "medica-#{SecureRandom.hex(3)}@cidade.gov.br", password: "senha-segura-123") }
  let(:patient) { Patient.create!(cpf: verified_citizen!(1).cpf) }
  let(:consultation) { Struct.new(:id).new(SecureRandom.uuid) }
  let(:ciap_release) { TerminologyRelease.active.find_by!(kind: "ciap2").id }

  def apply(action, **args)
    described_class.call(patient: patient, action: action, by: doctor, source: { consultation: consultation }, **args)
  end

  def add(code = "T90", **args) = apply("add", terminology: "ciap2", code: code, release_id: ciap_release, **args)

  it "inclui: problema ativo com início e precisão; evento com a consulta; trilha só com ids" do
    result = add(onset_on: Date.new(2025, 8, 1), onset_precision: "month")
    problem = result.payload[:problem]
    expect(problem).to have_attributes(status: "active", code: "T90", onset_on: Date.new(2025, 8, 1), onset_precision: "month")
    expect(result.payload[:event]).to have_attributes(kind: "added", consultation_id: consultation.id, user_id: doctor.id)
    expect(DomainEvent.where(name: "patient_problem.changed").sole.payload)
      .to eq("patient_problem_id" => problem.id, "kind" => "added", "consultation_id" => consultation.id)
  end

  it "incluir de novo o ativo é avaliar (sem evento); resolver; incluir o resolvido reativa a mesma linha" do
    problem = add.payload[:problem]
    again = add
    expect([ again.payload[:problem].id, again.payload[:event] ]).to eq([ problem.id, nil ])
    resolved = apply("resolve", problem: problem, on: Date.new(2026, 10, 7))
    expect(resolved.payload[:problem]).to have_attributes(status: "resolved", resolved_on: Date.new(2026, 10, 7))
    reactivated = add
    expect(reactivated.payload[:problem].id).to eq(problem.id)
    expect(reactivated.payload[:event].kind).to eq("reactivated")
    expect(problem.reload).to have_attributes(status: "active", resolved_on: nil)
    expect(PatientProblem.where(patient: patient).count).to eq(1)
  end

  it "os eventos reconstroem o estado" do
    problem = add(onset_on: Date.new(2020, 1, 1), onset_precision: "year").payload[:problem]
    apply("correct_onset", problem: problem, onset_on: Date.new(2019, 3, 1), onset_precision: "month")
    apply("resolve", problem: problem, on: Date.new(2026, 1, 2))
    add
    problem.reload
    expect(Patients::ProblemReplay.state(problem))
      .to eq(status: problem.status, onset_on: problem.onset_on, onset_precision: problem.onset_precision,
             resolved_on: problem.resolved_on)
    expect(Patients::ProblemReplay.history(problem).map { |e| e[:kind] }).to eq(%w[added onset_corrected resolved reactivated])
  end

  it "CID-10 convive com a CIAP-2 do mesmo problema clínico (terminologias diferentes)" do
    add("T90")
    cid = apply("add", terminology: "cid10", code: "E119", release_id: TerminologyRelease.active.find_by!(kind: "cid10").id)
    expect(cid).to be_ok
    expect(PatientProblem.where(patient: patient).pluck(:terminology).sort).to eq(%w[cid10 ciap2])
  end

  it "recusas: avaliar sem problema, resolver o resolvido, problema de outro paciente, ação desconhecida" do
    problem = add.payload[:problem]
    apply("resolve", problem: problem)
    other = Patient.create!(cpf: verified_citizen!(2).cpf)
    foreign = described_class.call(patient: other, action: "add", by: doctor, source: { consultation: consultation },
                                   terminology: "ciap2", code: "K86", release_id: ciap_release).payload[:problem]
    expect(apply("evaluate").reason).to eq(:invalid_problem)
    expect(apply("resolve", problem: problem.reload).reason).to eq(:invalid_problem)
    expect(apply("evaluate", problem: foreign).reason).to eq(:invalid_problem)
    expect(apply("apagar", problem: problem).reason).to eq(:invalid_problem)
    expect { described_class.call(patient: patient, action: "add", by: doctor, source: {}) }.to raise_error(ArgumentError)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/commands/patients/apply_problem_event_spec.rb`
Expected: FAIL com `uninitialized constant Patients::ApplyProblemEvent`.

- [ ] **Step 3: Implemente**

Em `app/models/patient_problem_event.rb`, troque `belongs_to :patient_problem` por:

```ruby
  # Opcional no modelo: o evento do problema NOVO é gravado antes dele (a FK
  # é DEFERRABLE e o trigger de patient_problems exige o evento).
  belongs_to :patient_problem, optional: true
```

```ruby
# app/commands/patients/apply_problem_event.rb
# Único caminho de escrita da lista de problemas (ADR 0031; spec §3). Grava o
# evento ANTES de mudar o estado — o trigger patient_problems_guard exige um
# evento da mesma transação com os mesmos valores. Quem chama abriu a
# transação e travou o paciente (Finalize, AddAddendum).
#   evaluate      — avaliado sem mudança (sem evento)
#   add           — ativo igual: avaliar; resolvido igual: reativar; senão incluir
#   resolve       — ativo → resolvido em `on`
#   correct_onset — corrige início e precisão
module Patients
  module ApplyProblemEvent
    ACTIONS = %w[evaluate add resolve correct_onset].freeze

    module_function

    def call(patient:, action:, by:, source:, terminology: nil, code: nil, release_id: nil, problem: nil,
             onset_on: nil, onset_precision: nil, on: Time.zone.today)
      consultation, addendum = source.values_at(:consultation, :addendum)
      raise ArgumentError, "source: consulta OU adendo" unless consultation.nil? ^ addendum.nil?
      return Result.fail(:invalid_problem) unless ACTIONS.include?(action.to_s)
      return Result.fail(:invalid_problem) if problem && problem.patient_id != patient.id

      origin = { by: by, consultation: consultation, addendum: addendum }
      case action.to_s
      when "evaluate"
        problem ? Result.ok(problem: problem, event: nil) : Result.fail(:invalid_problem)
      when "add"
        add(patient, terminology, code, release_id, onset_on, onset_precision, origin)
      when "resolve"
        return Result.fail(:invalid_problem) unless problem&.active?

        write(problem.lock!, "resolved", { status: "resolved", resolved_on: on }, origin)
      when "correct_onset"
        return Result.fail(:invalid_problem) unless problem

        write(problem.lock!, "onset_corrected", { onset_on: onset_on, onset_precision: onset_precision }, origin)
      end
    end

    def add(patient, terminology, code, release_id, onset_on, onset_precision, origin)
      scope = PatientProblem.where(patient_id: patient.id, terminology: terminology, code: code)
      active = scope.where(status: "active").lock.first
      return Result.ok(problem: active, event: nil) if active

      resolved = scope.where(status: "resolved").order(updated_at: :desc, id: :desc).lock.first
      if resolved
        attrs = { status: "active", resolved_on: nil, terminology_release_id: release_id }
        attrs.merge!(onset_on: onset_on, onset_precision: onset_precision) if onset_on
        return write(resolved, "reactivated", attrs, origin)
      end

      problem = PatientProblem.new(id: SecureRandom.uuid, patient: patient, terminology: terminology, code: code,
                                   terminology_release_id: release_id, status: "active",
                                   onset_on: onset_on, onset_precision: onset_precision)
      write(problem, "added", {}, origin)
    end

    def write(problem, kind, attrs, origin)
      problem.assign_attributes(attrs)
      event = PatientProblemEvent.create!(
        patient_problem_id: problem.id, kind: kind, consultation_id: origin[:consultation]&.id,
        addendum_id: origin[:addendum]&.id, user: origin[:by], status_after: problem.status,
        onset_on: problem.onset_on, onset_precision: problem.onset_precision, resolved_on: problem.resolved_on,
        terminology_release_id: problem.terminology_release_id
      )
      problem.save!
      link = origin[:consultation] ? { consultation_id: origin[:consultation].id } : { addendum_id: origin[:addendum].id }
      DomainEvents.publish("patient_problem.changed", patient_problem_id: problem.id, kind: kind, **link)
      Result.ok(problem: problem, event: event)
    end
    private_class_method :add, :write
  end
end
```

```ruby
# app/services/patients/problem_replay.rb
# "Quem disse que ele é diabético, e quando" (ADR 0031): o estado do problema
# reconstruído dos eventos, e o histórico com profissional e origem.
module Patients
  module ProblemReplay
    module_function

    def state(problem)
      last = problem.events.order(:created_at, :id).last
      last && { status: last.status_after, onset_on: last.onset_on, onset_precision: last.onset_precision,
                resolved_on: last.resolved_on }
    end

    def history(problem)
      problem.events.order(:created_at, :id).map do |event|
        { kind: event.kind, user_id: event.user_id, consultation_id: event.consultation_id,
          addendum_id: event.addendum_id, created_at: event.created_at }
      end
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/commands/patients spec/models/patient_tables_guard_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add app/commands/patients/apply_problem_event.rb app/services/patients/problem_replay.rb app/models/patient_problem_event.rb spec/commands/patients/apply_problem_event_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: change the problem list only through recorded events

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 2 — Consulta (F-19.3, F-19.4, F-19.5)

### Task 7: Migração da consulta, itens, adendos e aberturas; imutabilidade e a exceção da re-cifra

**Files:**
- Create: `db/city_migrate/20261007400002_add_consultations.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`
- Create: `app/models/consultation.rb`, `app/models/consultation_problem.rb`, `app/models/consultation_conduct.rb`, `app/models/consultation_exam_request.rb`, `app/models/consultation_addendum.rb`, `app/models/clinical_record_opening.rb`
- Modify: `app/models/patient.rb`, `app/models/attendance.rb`, `app/services/city_encryption.rb`, `app/jobs/reencryption_job.rb`, `app/commands/city_rekey.rb`
- Test: `spec/models/consultation_tables_guard_spec.rb`, `spec/commands/clinical_record_reencryption_spec.rb`

**Interfaces:**
- Produces:
  - Tabelas: `consultations (attendance_id único, patient_id, author_user_id, professional_link_id, cbo_code, status, subjective, objective, assessment, plan, <sinais vitais do módulo 18>, care_type, draft_items jsonb, started_at, finalized_at, timestamps)`; `consultation_problems (consultation_id, addendum_id, patient_problem_id, action, terminology, code, terminology_release_id, status_after, onset_on, onset_precision, resolved_on, created_at)`; `consultation_conducts (consultation_id, addendum_id, code, action, created_at)`; `consultation_exam_requests (consultation_id, addendum_id, sigtap_code, sigtap_competence, cid10_justification, status, created_at)`; `consultation_addenda (consultation_id, author_user_id, text, reason, changes jsonb, opening_id, created_at)`; `clinical_record_openings (patient_id, user_id, reason_code, reason_note, created_at, expires_at)`; FKs `patient_problem_events.consultation_id`/`addendum_id` (DEFERRABLE).
  - Triggers: `consultations_guard`, `consultation_{problems,conducts,exam_requests}_items_guard` (+ append-only, TRUNCATE), `consultation_addenda_born_finalized`, `consultation_addenda_reencryption_only`, `clinical_record_openings_reencryption_only` (+ TRUNCATE).
  - Modelos: `Consultation` (`STATUSES`, `TEXT_FIELDS`, `MAX_TEXT == 20_000`, `VITAL_COLUMNS`, `#draft?`, `#finalized?`, `#vitals`, `has_many :problem_items, :conducts, :exam_requests, :addenda`); `ConsultationProblem` (`ACTIONS`); `ConsultationConduct` (`ACTIONS`); `ConsultationExamRequest` (`STATUSES`); `ConsultationAddendum` (`MIN_REASON == 10`, `MAX_REASON == 500`); `ClinicalRecordOpening` (`REASONS`, `VALIDITY == 30.minutes`, `MIN_NOTE == 10`, `MAX_NOTE == 500`, `scope :valid_for(user_id:, patient_id:, now:)`); `Patient has_many :consultations`; `Attendance has_one :consultation`.
  - `CityEncryption.allowing_reencryption { }` (transação com `SET LOCAL rota.reencrypting = 'on'`).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/models/consultation_tables_guard_spec.rb
require "rails_helper"

# Módulo 19 (ADR 0031; spec §4–§5): consulta finalizada não muda; itens e
# adendos só por acréscimo (itens de consulta finalizada só com adendo);
# aberturas só acréscimo; texto clínico cifrado.
RSpec.describe "Guardas das tabelas da consulta" do
  before { Current.city = TEST_CITY_A; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:citizen) { verified_citizen!(1) }
  let(:patient) { Patient.create!(cpf: citizen.cpf) }
  let(:attendance) { consulting_attendance!(unit, citizen: citizen, doctor: doctor) }
  let(:link) { doctor.professional.links.active.sole }

  def attempt(&) = ApplicationRecord.transaction(requires_new: true, &)

  def consultation!(**over)
    Consultation.create!({ attendance: attendance, patient: patient, author_user: doctor, professional_link: link,
                           cbo_code: link.cbo_code, status: "draft", started_at: Time.current,
                           subjective: "MARCADOR-S", assessment: "avaliação" }.merge(over))
  end

  def finalize!(consultation) = consultation.update!(status: "finalized", finalized_at: Time.current, care_type: 5)

  def problem_row!(consultation, addendum: nil)
    id = SecureRandom.uuid
    PatientProblemEvent.create!(patient_problem_id: id, kind: "added", consultation_id: consultation.id, user: doctor,
                                status_after: "active")
    problem = PatientProblem.create!(id: id, patient: patient, terminology: "ciap2", code: "T90", status: "active",
                                     terminology_release_id: TerminologyRelease.active.find_by!(kind: "ciap2").id)
    ConsultationProblem.create!(consultation: consultation, addendum: addendum, patient_problem: problem, action: "add",
                                terminology: "ciap2", code: "T90", terminology_release_id: problem.terminology_release_id,
                                status_after: "active")
  end

  def addendum!(consultation, **over)
    ConsultationAddendum.create!({ consultation: consultation, author_user: doctor, text: "MARCADOR-ADENDO",
                                   reason: "correção do exame pedido" }.merge(over))
  end

  describe "consultations" do
    it "uma por atendimento; nunca some; identidade fixa; texto cifrado" do
      consultation = consultation!
      expect { attempt { consultation! } }.to raise_error(ActiveRecord::RecordNotUnique)
      expect { attempt { consultation.delete } }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
      expect { attempt { consultation.update_columns(author_user_id: User.create!(email_address: "x-#{SecureRandom.hex(3)}@x.br", password: "senha-segura-123").id) } }
        .to raise_error(ActiveRecord::StatementInvalid, /identity columns never change/)
      raw = ApplicationRecord.connection.select_value("SELECT subjective FROM consultations WHERE id = #{ApplicationRecord.connection.quote(consultation.id)}")
      expect(raw).not_to include("MARCADOR")
    end

    it "rascunho muda; finalizada nunca muda (nem volta a rascunho)" do
      consultation = consultation!
      expect { consultation.update!(plan: "plano novo", systolic: 120, diastolic: 80) }.not_to raise_error
      finalize!(consultation)
      expect { attempt { consultation.update!(plan: "mudei depois") } }
        .to raise_error(ActiveRecord::StatementInvalid, /finalized consultation never changes/)
      expect { attempt { consultation.update_columns(status: "draft", finalized_at: nil) } }
        .to raise_error(ActiveRecord::StatementInvalid, /finalized consultation never changes/)
    end

    it "a marca da re-cifra só deixa mudar o texto cifrado" do
      consultation = consultation!
      finalize!(consultation)
      expect do
        attempt do
          ApplicationRecord.connection.execute("SET LOCAL rota.reencrypting = 'on'")
          consultation.update_columns(care_type: 6)
        end
      end.to raise_error(ActiveRecord::StatementInvalid, /finalized consultation never changes/)
      expect { CityEncryption.allowing_reencryption { consultation.encrypt } }.not_to raise_error
      expect(consultation.reload.subjective).to eq("MARCADOR-S")
    end

    it "CHECKs: finalizada exige tipo e itens do rascunho vazios; tipo 4 não; pressão aos pares" do
      expect { attempt { consultation!(care_type: 4) } }.to raise_error(ActiveRecord::StatementInvalid, /ck_consultations_care_type/)
      expect { attempt { consultation!(systolic: 120) } }.to raise_error(ActiveRecord::StatementInvalid, /ck_consultations_bp/)
      consultation = consultation!(draft_items: { "conducts" => [ 1 ] })
      expect { attempt { consultation.update!(status: "finalized", finalized_at: Time.current, care_type: 5) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_consultations_draft_items/)
      expect { attempt { consultation.update!(status: "finalized", finalized_at: Time.current, draft_items: {}) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_consultations_finalized_care_type/)
    end
  end

  describe "itens" do
    it "entram com a consulta em rascunho; depois só com adendo da mesma consulta; nunca mudam" do
      consultation = consultation!
      row = problem_row!(consultation)
      conduct = ConsultationConduct.create!(consultation: consultation, code: 9, action: "add")
      finalize!(consultation)
      expect { attempt { ConsultationConduct.create!(consultation: consultation, code: 1, action: "add") } }
        .to raise_error(ActiveRecord::StatementInvalid, /only come with an addendum/)
      other = Consultation.create!(attendance: consulting_attendance!(unit, citizen: verified_citizen!(2), doctor: doctor),
                                   patient: Patient.create!(cpf: verified_citizen!(3).cpf), author_user: doctor,
                                   professional_link: link, cbo_code: link.cbo_code, status: "draft", started_at: Time.current)
      finalize!(other)
      foreign = addendum!(other)
      expect { attempt { ConsultationConduct.create!(consultation: consultation, addendum: foreign, code: 1, action: "add") } }
        .to raise_error(ActiveRecord::StatementInvalid, /addendum of another consultation/)
      mine = addendum!(consultation)
      expect { ConsultationConduct.create!(consultation: consultation, addendum: mine, code: 9, action: "remove") }.not_to raise_error
      expect { attempt { row.update_columns(action: "resolve") } }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { attempt { conduct.delete } }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    end

    it "CHECKs: conduta da lista; remover só com adendo; exame do grupo 02; cancelar só com adendo" do
      consultation = consultation!
      { { code: 3, action: "add" } => /ck_consultation_conducts_code/,
        { code: 9, action: "remove" } => /ck_consultation_conducts_removal/ }.each do |attrs, error|
        expect { attempt { ConsultationConduct.create!(consultation: consultation, **attrs) } }
          .to raise_error(ActiveRecord::StatementInvalid, error)
      end
      base = { consultation: consultation, sigtap_competence: "202610", status: "requested" }
      { { sigtap_code: "0301010064" } => /ck_consultation_exam_requests_sigtap/,
        { sigtap_code: "0202010503", status: "cancelled" } => /ck_consultation_exam_requests_cancel/,
        { sigtap_code: "0202010503", cid10_justification: "e11" } => /ck_consultation_exam_requests_cid10/ }.each do |attrs, error|
        expect { attempt { ConsultationExamRequest.create!(base.merge(attrs)) } }
          .to raise_error(ActiveRecord::StatementInvalid, error), attrs.inspect
      end
    end
  end

  describe "consultation_addenda" do
    it "só em consulta finalizada; nunca muda nem some; motivo 10–500; texto cifrado" do
      consultation = consultation!
      expect { attempt { addendum!(consultation) } }.to raise_error(ActiveRecord::StatementInvalid, /only a finalized consultation/)
      finalize!(consultation)
      addendum = addendum!(consultation)
      expect { attempt { addendum.update_columns(reason: "outro motivo qualquer") } }
        .to raise_error(ActiveRecord::StatementInvalid, /UPDATE refused/)
      expect { attempt { addendum.delete } }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
      expect { attempt { addendum!(consultation, reason: "curto") } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_consultation_addenda_reason/)
      expect { CityEncryption.allowing_reencryption { addendum.encrypt } }.not_to raise_error
      raw = ApplicationRecord.connection.select_value("SELECT text FROM consultation_addenda WHERE id = #{ApplicationRecord.connection.quote(addendum.id)}")
      expect(raw).not_to include("MARCADOR")
    end
  end

  describe "clinical_record_openings" do
    it "só acréscimo; nota só com other; validade positiva" do
      now = Time.current
      opening = ClinicalRecordOpening.create!(patient: patient, user: doctor, reason_code: "case_review",
                                              created_at: now, expires_at: now + 30.minutes)
      expect { attempt { opening.update_columns(expires_at: now + 1.day) } }.to raise_error(ActiveRecord::StatementInvalid, /UPDATE refused/)
      expect { attempt { opening.delete } }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
      { { reason_code: "other" } => /ck_clinical_record_openings_note/,
        { reason_code: "case_review", reason_note: "nota sem other" } => /ck_clinical_record_openings_note/,
        { reason_code: "curiosidade" } => /ck_clinical_record_openings_reason/,
        { reason_code: "case_review", expires_at: now } => /ck_clinical_record_openings_expiry/ }.each do |attrs, error|
        expect do
          attempt { ClinicalRecordOpening.create!({ patient: patient, user: doctor, created_at: now, expires_at: now + 30.minutes }.merge(attrs)) }
        end.to raise_error(ActiveRecord::StatementInvalid, error), attrs.inspect
      end
      expect(ClinicalRecordOpening.valid_for(user_id: doctor.id, patient_id: patient.id, now: now + 29.minutes)).to eq([ opening ])
      expect(ClinicalRecordOpening.valid_for(user_id: doctor.id, patient_id: patient.id, now: now + 30.minutes)).to be_empty
    end
  end
end
```

```ruby
# spec/commands/clinical_record_reencryption_spec.rb
require "rails_helper"

# Review Focus 1 (ADR 0031; Desvio 4): a rotação de chave da cidade regrava o
# texto cifrado da consulta finalizada, do adendo e da nota de abertura sem
# esbarrar na imutabilidade — e o conteúdo continua legível.
RSpec.describe "Re-cifra do prontuário" do
  let!(:city) { clinical_city! }

  def in_city(&) = CityConnection.with(city) { Current.set(city: city, &) }

  def call_job_body(**kwargs)
    ReencryptionJob.instance_method(:perform).super_method.bind_call(ReencryptionJob.new, **kwargs)
  end

  let!(:rows) do
    in_city do
      ciap2_release!
      unit = create_unit
      doctor = doctor!(unit)
      citizen = verified_citizen!(1)
      patient = Patient.create!(cpf: citizen.cpf, full_name: "Maria Aparecida da Silva")
      link = doctor.professional.links.active.sole
      consultation = Consultation.create!(attendance: consulting_attendance!(unit, citizen: citizen, doctor: doctor),
                                          patient: patient, author_user: doctor, professional_link: link, cbo_code: link.cbo_code,
                                          status: "draft", started_at: Time.current, subjective: "texto S", objective: "texto O",
                                          assessment: "texto A", plan: "texto P")
      consultation.update!(status: "finalized", finalized_at: Time.current, care_type: 5)
      addendum = ConsultationAddendum.create!(consultation: consultation, author_user: doctor, text: "texto do adendo",
                                              reason: "acréscimo de informação")
      now = Time.current
      opening = ClinicalRecordOpening.create!(patient: patient, user: doctor, reason_code: "other",
                                              reason_note: "revisão pedida pela coordenação", created_at: now, expires_at: now + 30.minutes)
      { consultation: consultation, addendum: addendum, opening: opening }
    end
  end

  it "ReencryptionJob regrava e o texto continua o mesmo" do
    stats = in_city { call_job_body(only: %i[consultation consultation_addendum clinical_record_opening]) }
    expect(stats).to include("Consultation" => 1, "ConsultationAddendum" => 1, "ClinicalRecordOpening" => 1)
    in_city do
      expect(rows[:consultation].reload.slice(:subjective, :plan)).to eq("subjective" => "texto S", "plan" => "texto P")
      expect(rows[:addendum].reload.text).to eq("texto do adendo")
      expect(rows[:opening].reload.reason_note).to eq("revisão pedida pela coordenação")
    end
  end

  it "CityRekey regrava tudo numa transação com a marca" do
    result = CityRekey.call(city: city)
    expect(result).to be_ok
    expect(result.payload[:counts]).to include("Consultation" => 4, "ConsultationAddendum" => 1, "ClinicalRecordOpening" => 1)
    in_city { expect(rows[:addendum].reload.text).to eq("texto do adendo") }
  end

  it "sem a marca, a consulta finalizada continua recusando a regravação" do
    in_city do
      expect { ApplicationRecord.transaction(requires_new: true) { rows[:consultation].encrypt } }
        .to raise_error(ActiveRecord::StatementInvalid, /finalized consultation never changes/)
    end
  end
end
```

(`CityRekey` conta por atributo não nulo — as quatro colunas de texto preenchidas → `"Consultation" => 4`; o `ReencryptionJob` conta linhas → `1`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/models/consultation_tables_guard_spec.rb spec/commands/clinical_record_reencryption_spec.rb`
Expected: FAIL (`uninitialized constant Consultation`).

- [ ] **Step 3: Escreva a migração**

```ruby
# db/city_migrate/20261007400002_add_consultations.rb
# Módulo 19 (ADR 0031; spec 2026-10-07 §4–§5): a consulta SOAP (texto
# cifrado, sinais vitais com os limites do módulo 18), os itens estruturados
# (só acréscimo; os do rascunho ficam em draft_items até a finalização), os
# adendos e as aberturas justificadas. As FKs dos eventos de problema são
# DEFERRABLE (a ordem de escrita na finalização é livre; o commit confere).
# Triggers em db/city_triggers.sql.
class AddConsultations < ActiveRecord::Migration[8.1]
  VITALS = {
    "ck_consultations_bp" => "(systolic IS NULL AND diastolic IS NULL) OR (systolic IS NOT NULL AND diastolic IS NOT NULL AND diastolic < systolic)",
    "ck_consultations_systolic" => "systolic IS NULL OR systolic BETWEEN 50 AND 300",
    "ck_consultations_diastolic" => "diastolic IS NULL OR diastolic BETWEEN 20 AND 200",
    "ck_consultations_heart_rate" => "heart_rate IS NULL OR heart_rate BETWEEN 20 AND 250",
    "ck_consultations_respiratory_rate" => "respiratory_rate IS NULL OR respiratory_rate BETWEEN 4 AND 80",
    "ck_consultations_temperature" => "temperature_c IS NULL OR temperature_c BETWEEN 30 AND 45",
    "ck_consultations_spo2" => "spo2 IS NULL OR spo2 BETWEEN 50 AND 100",
    "ck_consultations_glucose" => "(capillary_glucose IS NULL AND glucose_moment IS NULL) OR (capillary_glucose BETWEEN 10 AND 800 AND glucose_moment::text = ANY (ARRAY['fasting'::text, 'postprandial'::text, 'random'::text]))",
    "ck_consultations_weight" => "weight_kg IS NULL OR weight_kg BETWEEN 0.5 AND 400",
    "ck_consultations_height" => "height_cm IS NULL OR height_cm BETWEEN 30 AND 250",
    "ck_consultations_pain_score" => "pain_score IS NULL OR pain_score BETWEEN 0 AND 10"
  }.freeze
  CONDUCT_CODES = "ARRAY[1, 2, 4, 5, 6, 7, 8, 9, 10, 11, 12, 14]".freeze

  def up
    create_table :consultations, id: :uuid do |t|
      t.uuid :attendance_id, null: false
      t.uuid :patient_id, null: false
      t.uuid :author_user_id, null: false
      t.uuid :professional_link_id, null: false
      t.string :cbo_code, null: false
      t.string :status, null: false, default: "draft"
      t.text :subjective
      t.text :objective
      t.text :assessment
      t.text :plan
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
      t.integer :care_type
      t.jsonb :draft_items, null: false, default: {}
      t.datetime :started_at, null: false
      t.datetime :finalized_at
      t.timestamps
    end
    add_index :consultations, :attendance_id, unique: true
    add_index :consultations, :patient_id
    add_index :consultations, :author_user_id
    add_index :consultations, :professional_link_id
    add_foreign_key :consultations, :attendances
    add_foreign_key :consultations, :patients
    add_foreign_key :consultations, :users, column: :author_user_id
    add_foreign_key :consultations, :professional_links
    {
      "ck_consultations_status" => "status::text = ANY (ARRAY['draft'::text, 'finalized'::text])",
      "ck_consultations_finalization" => "(status::text = 'finalized'::text) = (finalized_at IS NOT NULL)",
      "ck_consultations_draft_items" => "jsonb_typeof(draft_items) = 'object'::text AND (status::text = 'draft'::text OR draft_items = '{}'::jsonb)",
      "ck_consultations_care_type" => "care_type IS NULL OR care_type = ANY (ARRAY[1, 2, 5, 6])",
      "ck_consultations_finalized_care_type" => "status::text = 'draft'::text OR care_type IS NOT NULL",
      "ck_consultations_cbo_code" => "cbo_code::text ~ '^[0-9A-Z]{6}$'::text"
    }.merge(VITALS).each { |name, expression| add_check_constraint :consultations, expression, name: name }

    create_table :clinical_record_openings, id: :uuid do |t|
      t.uuid :patient_id, null: false
      t.uuid :user_id, null: false
      t.string :reason_code, null: false
      t.text :reason_note
      t.datetime :created_at, null: false
      t.datetime :expires_at, null: false
    end
    add_index :clinical_record_openings, %i[user_id patient_id expires_at], name: "idx_clinical_record_openings_valid"
    add_index :clinical_record_openings, :patient_id
    add_index :clinical_record_openings, :created_at
    add_foreign_key :clinical_record_openings, :patients
    add_foreign_key :clinical_record_openings, :users
    {
      "ck_clinical_record_openings_reason" => "reason_code::text = ANY (ARRAY['case_review'::text, 'active_search'::text, 'continuity_of_care'::text, 'other'::text])",
      "ck_clinical_record_openings_note" => "(reason_code::text = 'other'::text) = (reason_note IS NOT NULL)",
      "ck_clinical_record_openings_expiry" => "expires_at > created_at"
    }.each { |name, expression| add_check_constraint :clinical_record_openings, expression, name: name }

    create_table :consultation_addenda, id: :uuid do |t|
      t.uuid :consultation_id, null: false
      t.uuid :author_user_id, null: false
      t.text :text, null: false
      t.text :reason, null: false
      t.jsonb :changes, null: false, default: {}
      t.uuid :opening_id
      t.datetime :created_at, null: false
    end
    add_index :consultation_addenda, :consultation_id
    add_index :consultation_addenda, :author_user_id
    add_index :consultation_addenda, :opening_id
    add_foreign_key :consultation_addenda, :consultations
    add_foreign_key :consultation_addenda, :users, column: :author_user_id
    add_foreign_key :consultation_addenda, :clinical_record_openings, column: :opening_id
    add_check_constraint :consultation_addenda, "length(btrim(reason)) BETWEEN 10 AND 500", name: "ck_consultation_addenda_reason"
    add_check_constraint :consultation_addenda, "jsonb_typeof(changes) = 'object'::text", name: "ck_consultation_addenda_changes"

    create_table :consultation_problems, id: :uuid do |t|
      t.uuid :consultation_id, null: false
      t.uuid :addendum_id
      t.uuid :patient_problem_id, null: false
      t.string :action, null: false
      t.string :terminology, null: false
      t.string :code, limit: 4, null: false
      t.uuid :terminology_release_id, null: false
      t.string :status_after, null: false
      t.date :onset_on
      t.string :onset_precision
      t.date :resolved_on
      t.datetime :created_at, null: false
    end
    add_index :consultation_problems, :consultation_id
    add_index :consultation_problems, :addendum_id
    add_index :consultation_problems, :patient_problem_id
    add_foreign_key :consultation_problems, :consultations
    add_foreign_key :consultation_problems, :consultation_addenda, column: :addendum_id
    add_foreign_key :consultation_problems, :patient_problems
    {
      "ck_consultation_problems_action" => "action::text = ANY (ARRAY['evaluate'::text, 'add'::text, 'resolve'::text, 'correct_onset'::text])",
      "ck_consultation_problems_terminology" => "terminology::text = ANY (ARRAY['ciap2'::text, 'cid10'::text])",
      "ck_consultation_problems_status" => "status_after::text = ANY (ARRAY['active'::text, 'resolved'::text])"
    }.each { |name, expression| add_check_constraint :consultation_problems, expression, name: name }

    create_table :consultation_conducts, id: :uuid do |t|
      t.uuid :consultation_id, null: false
      t.uuid :addendum_id
      t.integer :code, null: false
      t.string :action, null: false, default: "add"
      t.datetime :created_at, null: false
    end
    add_index :consultation_conducts, :consultation_id
    add_index :consultation_conducts, :addendum_id
    add_foreign_key :consultation_conducts, :consultations
    add_foreign_key :consultation_conducts, :consultation_addenda, column: :addendum_id
    add_check_constraint :consultation_conducts, "code = ANY (#{CONDUCT_CODES})", name: "ck_consultation_conducts_code"
    add_check_constraint :consultation_conducts, "action::text = ANY (ARRAY['add'::text, 'remove'::text])",
                         name: "ck_consultation_conducts_action"
    add_check_constraint :consultation_conducts, "action::text = 'add'::text OR addendum_id IS NOT NULL",
                         name: "ck_consultation_conducts_removal"

    create_table :consultation_exam_requests, id: :uuid do |t|
      t.uuid :consultation_id, null: false
      t.uuid :addendum_id
      t.string :sigtap_code, limit: 10, null: false
      t.string :sigtap_competence, limit: 6, null: false
      t.string :cid10_justification, limit: 4
      t.string :status, null: false, default: "requested"
      t.datetime :created_at, null: false
    end
    add_index :consultation_exam_requests, :consultation_id
    add_index :consultation_exam_requests, :addendum_id
    add_foreign_key :consultation_exam_requests, :consultations
    add_foreign_key :consultation_exam_requests, :consultation_addenda, column: :addendum_id
    {
      "ck_consultation_exam_requests_sigtap" => "sigtap_code::text ~ '^02[0-9]{8}$'::text",
      "ck_consultation_exam_requests_competence" => "sigtap_competence::text ~ '^[0-9]{4}(0[1-9]|1[0-2])$'::text",
      "ck_consultation_exam_requests_cid10" => "cid10_justification IS NULL OR cid10_justification::text ~ '^[A-Z][0-9]{2}[0-9X]?$'::text",
      "ck_consultation_exam_requests_status" => "status::text = ANY (ARRAY['requested'::text, 'cancelled'::text])",
      "ck_consultation_exam_requests_cancel" => "status::text = 'requested'::text OR addendum_id IS NOT NULL"
    }.each { |name, expression| add_check_constraint :consultation_exam_requests, expression, name: name }

    add_foreign_key :patient_problem_events, :consultations, deferrable: :deferred
    add_foreign_key :patient_problem_events, :consultation_addenda, column: :addendum_id, deferrable: :deferred

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

- [ ] **Step 4: Triggers (no fim de `db/city_triggers.sql`)**

```sql
-- consultations (ADR 0031, Invariantes; spec §4): uma por atendimento; nunca
-- some; a identidade não muda; o rascunho muda à vontade; finalizada nunca
-- muda. Única exceção: a re-cifra (CityEncryption.allowing_reencryption marca
-- a transação com rota.reencrypting) pode regravar SÓ o texto cifrado.
CREATE OR REPLACE FUNCTION rota_consultation_guard() RETURNS trigger AS $fn$
DECLARE
  texts text[] := ARRAY['subjective', 'objective', 'assessment', 'plan'];
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'consultations is append-only: DELETE refused';
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id OR NEW.attendance_id IS DISTINCT FROM OLD.attendance_id
     OR NEW.patient_id IS DISTINCT FROM OLD.patient_id OR NEW.author_user_id IS DISTINCT FROM OLD.author_user_id
     OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'consultations: identity columns never change';
  END IF;
  IF OLD.status = 'finalized' THEN
    IF current_setting('rota.reencrypting', true) = 'on' AND (to_jsonb(NEW) - texts) = (to_jsonb(OLD) - texts) THEN
      RETURN NEW;
    END IF;
    RAISE EXCEPTION 'consultations: a finalized consultation never changes; corrections are addenda';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- Itens da consulta (ADR 0031; spec §4): só acréscimo. Sem adendo, só com a
-- consulta ainda em rascunho (a finalização grava os itens antes de virar o
-- status); depois, só com um adendo DESTA consulta.
CREATE OR REPLACE FUNCTION rota_consultation_item_guard() RETURNS trigger AS $fn$
BEGIN
  IF NEW.addendum_id IS NULL THEN
    IF NOT EXISTS (SELECT 1 FROM consultations c WHERE c.id = NEW.consultation_id AND c.status = 'draft') THEN
      RAISE EXCEPTION '%: items of a finalized consultation only come with an addendum', TG_TABLE_NAME;
    END IF;
  ELSIF NOT EXISTS (SELECT 1 FROM consultation_addenda a WHERE a.id = NEW.addendum_id AND a.consultation_id = NEW.consultation_id) THEN
    RAISE EXCEPTION '%: addendum of another consultation', TG_TABLE_NAME;
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- Adendo só em consulta finalizada (ADR 0031).
CREATE OR REPLACE FUNCTION rota_consultation_addendum_insert_guard() RETURNS trigger AS $fn$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM consultations c WHERE c.id = NEW.consultation_id AND c.status = 'finalized') THEN
    RAISE EXCEPTION 'consultation_addenda: only a finalized consultation gains addenda';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- Só acréscimo, com a exceção da re-cifra: as colunas passadas em TG_ARGV (as
-- cifradas) podem ser regravadas com rota.reencrypting; nada mais muda.
CREATE OR REPLACE FUNCTION rota_reencryption_only() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION '% is append-only: DELETE refused', TG_TABLE_NAME;
  END IF;
  IF current_setting('rota.reencrypting', true) = 'on' AND (to_jsonb(NEW) - TG_ARGV) = (to_jsonb(OLD) - TG_ARGV) THEN
    RETURN NEW;
  END IF;
  RAISE EXCEPTION '% is append-only: UPDATE refused', TG_TABLE_NAME;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
DECLARE
  item text;
BEGIN
  IF to_regclass('public.consultations') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS consultations_guard ON consultations';
    EXECUTE 'CREATE TRIGGER consultations_guard
      BEFORE UPDATE OR DELETE ON consultations
      FOR EACH ROW EXECUTE FUNCTION rota_consultation_guard()';
    EXECUTE 'DROP TRIGGER IF EXISTS consultations_append_only_truncate ON consultations';
    EXECUTE 'CREATE TRIGGER consultations_append_only_truncate
      BEFORE TRUNCATE ON consultations
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
  END IF;
  FOREACH item IN ARRAY ARRAY['consultation_problems', 'consultation_conducts', 'consultation_exam_requests'] LOOP
    IF to_regclass('public.' || item) IS NOT NULL THEN
      EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_items_guard', item);
      EXECUTE format('CREATE TRIGGER %I BEFORE INSERT ON %I FOR EACH ROW EXECUTE FUNCTION rota_consultation_item_guard()',
                     item || '_items_guard', item);
      EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_append_only', item);
      EXECUTE format('CREATE TRIGGER %I BEFORE UPDATE OR DELETE ON %I FOR EACH ROW EXECUTE FUNCTION rota_append_only()',
                     item || '_append_only', item);
      EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_append_only_truncate', item);
      EXECUTE format('CREATE TRIGGER %I BEFORE TRUNCATE ON %I FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()',
                     item || '_append_only_truncate', item);
    END IF;
  END LOOP;
  IF to_regclass('public.consultation_addenda') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS consultation_addenda_born_finalized ON consultation_addenda';
    EXECUTE 'CREATE TRIGGER consultation_addenda_born_finalized
      BEFORE INSERT ON consultation_addenda
      FOR EACH ROW EXECUTE FUNCTION rota_consultation_addendum_insert_guard()';
    EXECUTE 'DROP TRIGGER IF EXISTS consultation_addenda_reencryption_only ON consultation_addenda';
    EXECUTE 'CREATE TRIGGER consultation_addenda_reencryption_only
      BEFORE UPDATE OR DELETE ON consultation_addenda
      FOR EACH ROW EXECUTE FUNCTION rota_reencryption_only(''text'')';
    EXECUTE 'DROP TRIGGER IF EXISTS consultation_addenda_append_only_truncate ON consultation_addenda';
    EXECUTE 'CREATE TRIGGER consultation_addenda_append_only_truncate
      BEFORE TRUNCATE ON consultation_addenda
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
  END IF;
  IF to_regclass('public.clinical_record_openings') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS clinical_record_openings_reencryption_only ON clinical_record_openings';
    EXECUTE 'CREATE TRIGGER clinical_record_openings_reencryption_only
      BEFORE UPDATE OR DELETE ON clinical_record_openings
      FOR EACH ROW EXECUTE FUNCTION rota_reencryption_only(''reason_note'')';
    EXECUTE 'DROP TRIGGER IF EXISTS clinical_record_openings_append_only_truncate ON clinical_record_openings';
    EXECUTE 'CREATE TRIGGER clinical_record_openings_append_only_truncate
      BEFORE TRUNCATE ON clinical_record_openings
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
  END IF;
END
$do$;
```

(`to_jsonb(NEW) - TG_ARGV` usa o operador `jsonb - text[]`; `TG_ARGV` já é `text[]`.)

- [ ] **Step 5: Dump à mão em `db/city_schema.rb`**

Troque `define(version: 2026_10_07_400001)` por `define(version: 2026_10_07_400002)`. Tabelas novas (ordem alfabética entre as existentes; colunas em ordem alfabética, como o Rails despeja):

```ruby
  create_table "clinical_record_openings", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "created_at", null: false
    t.datetime "expires_at", null: false
    t.uuid "patient_id", null: false
    t.string "reason_code", null: false
    t.text "reason_note"
    t.uuid "user_id", null: false
    t.index ["created_at"], name: "index_clinical_record_openings_on_created_at"
    t.index ["patient_id"], name: "index_clinical_record_openings_on_patient_id"
    t.index ["user_id", "patient_id", "expires_at"], name: "idx_clinical_record_openings_valid"
    t.check_constraint "expires_at > created_at", name: "ck_clinical_record_openings_expiry"
    t.check_constraint "(reason_code::text = 'other'::text) = (reason_note IS NOT NULL)", name: "ck_clinical_record_openings_note"
    t.check_constraint "reason_code::text = ANY (ARRAY['case_review'::text, 'active_search'::text, 'continuity_of_care'::text, 'other'::text])", name: "ck_clinical_record_openings_reason"
  end

  create_table "consultation_addenda", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "author_user_id", null: false
    t.jsonb "changes", default: {}, null: false
    t.uuid "consultation_id", null: false
    t.datetime "created_at", null: false
    t.uuid "opening_id"
    t.text "reason", null: false
    t.text "text", null: false
    t.index ["author_user_id"], name: "index_consultation_addenda_on_author_user_id"
    t.index ["consultation_id"], name: "index_consultation_addenda_on_consultation_id"
    t.index ["opening_id"], name: "index_consultation_addenda_on_opening_id"
    t.check_constraint "jsonb_typeof(changes) = 'object'::text", name: "ck_consultation_addenda_changes"
    t.check_constraint "length(btrim(reason)) >= 10 AND length(btrim(reason)) <= 500", name: "ck_consultation_addenda_reason"
  end

  create_table "consultation_conducts", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.string "action", default: "add", null: false
    t.uuid "addendum_id"
    t.integer "code", null: false
    t.uuid "consultation_id", null: false
    t.datetime "created_at", null: false
    t.index ["addendum_id"], name: "index_consultation_conducts_on_addendum_id"
    t.index ["consultation_id"], name: "index_consultation_conducts_on_consultation_id"
    t.check_constraint "action::text = ANY (ARRAY['add'::text, 'remove'::text])", name: "ck_consultation_conducts_action"
    t.check_constraint "code = ANY (ARRAY[1, 2, 4, 5, 6, 7, 8, 9, 10, 11, 12, 14])", name: "ck_consultation_conducts_code"
    t.check_constraint "action::text = 'add'::text OR addendum_id IS NOT NULL", name: "ck_consultation_conducts_removal"
  end

  create_table "consultation_exam_requests", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "addendum_id"
    t.string "cid10_justification", limit: 4
    t.uuid "consultation_id", null: false
    t.datetime "created_at", null: false
    t.string "sigtap_code", limit: 10, null: false
    t.string "sigtap_competence", limit: 6, null: false
    t.string "status", default: "requested", null: false
    t.index ["addendum_id"], name: "index_consultation_exam_requests_on_addendum_id"
    t.index ["consultation_id"], name: "index_consultation_exam_requests_on_consultation_id"
    t.check_constraint "status::text = 'requested'::text OR addendum_id IS NOT NULL", name: "ck_consultation_exam_requests_cancel"
    t.check_constraint "cid10_justification IS NULL OR cid10_justification::text ~ '^[A-Z][0-9]{2}[0-9X]?$'::text", name: "ck_consultation_exam_requests_cid10"
    t.check_constraint "sigtap_competence::text ~ '^[0-9]{4}(0[1-9]|1[0-2])$'::text", name: "ck_consultation_exam_requests_competence"
    t.check_constraint "sigtap_code::text ~ '^02[0-9]{8}$'::text", name: "ck_consultation_exam_requests_sigtap"
    t.check_constraint "status::text = ANY (ARRAY['requested'::text, 'cancelled'::text])", name: "ck_consultation_exam_requests_status"
  end

  create_table "consultation_problems", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.string "action", null: false
    t.uuid "addendum_id"
    t.string "code", limit: 4, null: false
    t.uuid "consultation_id", null: false
    t.datetime "created_at", null: false
    t.date "onset_on"
    t.string "onset_precision"
    t.uuid "patient_problem_id", null: false
    t.date "resolved_on"
    t.string "status_after", null: false
    t.string "terminology", null: false
    t.uuid "terminology_release_id", null: false
    t.index ["addendum_id"], name: "index_consultation_problems_on_addendum_id"
    t.index ["consultation_id"], name: "index_consultation_problems_on_consultation_id"
    t.index ["patient_problem_id"], name: "index_consultation_problems_on_patient_problem_id"
    t.check_constraint "action::text = ANY (ARRAY['evaluate'::text, 'add'::text, 'resolve'::text, 'correct_onset'::text])", name: "ck_consultation_problems_action"
    t.check_constraint "status_after::text = ANY (ARRAY['active'::text, 'resolved'::text])", name: "ck_consultation_problems_status"
    t.check_constraint "terminology::text = ANY (ARRAY['ciap2'::text, 'cid10'::text])", name: "ck_consultation_problems_terminology"
  end

  create_table "consultations", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.text "assessment"
    t.uuid "attendance_id", null: false
    t.uuid "author_user_id", null: false
    t.integer "capillary_glucose"
    t.integer "care_type"
    t.string "cbo_code", null: false
    t.datetime "created_at", null: false
    t.integer "diastolic"
    t.jsonb "draft_items", default: {}, null: false
    t.datetime "finalized_at"
    t.string "glucose_moment"
    t.integer "heart_rate"
    t.integer "height_cm"
    t.text "objective"
    t.integer "pain_score"
    t.uuid "patient_id", null: false
    t.text "plan"
    t.uuid "professional_link_id", null: false
    t.integer "respiratory_rate"
    t.integer "spo2"
    t.datetime "started_at", null: false
    t.string "status", default: "draft", null: false
    t.text "subjective"
    t.integer "systolic"
    t.decimal "temperature_c", precision: 3, scale: 1
    t.datetime "updated_at", null: false
    t.decimal "weight_kg", precision: 5, scale: 2
    t.index ["attendance_id"], name: "index_consultations_on_attendance_id", unique: true
    t.index ["author_user_id"], name: "index_consultations_on_author_user_id"
    t.index ["patient_id"], name: "index_consultations_on_patient_id"
    t.index ["professional_link_id"], name: "index_consultations_on_professional_link_id"
    t.check_constraint "(systolic IS NULL AND diastolic IS NULL) OR (systolic IS NOT NULL AND diastolic IS NOT NULL AND diastolic < systolic)", name: "ck_consultations_bp"
    t.check_constraint "care_type IS NULL OR care_type = ANY (ARRAY[1, 2, 5, 6])", name: "ck_consultations_care_type"
    t.check_constraint "cbo_code::text ~ '^[0-9A-Z]{6}$'::text", name: "ck_consultations_cbo_code"
    t.check_constraint "diastolic IS NULL OR diastolic BETWEEN 20 AND 200", name: "ck_consultations_diastolic"
    t.check_constraint "jsonb_typeof(draft_items) = 'object'::text AND (status::text = 'draft'::text OR draft_items = '{}'::jsonb)", name: "ck_consultations_draft_items"
    t.check_constraint "(status::text = 'finalized'::text) = (finalized_at IS NOT NULL)", name: "ck_consultations_finalization"
    t.check_constraint "status::text = 'draft'::text OR care_type IS NOT NULL", name: "ck_consultations_finalized_care_type"
    t.check_constraint "(capillary_glucose IS NULL AND glucose_moment IS NULL) OR (capillary_glucose BETWEEN 10 AND 800 AND glucose_moment::text = ANY (ARRAY['fasting'::text, 'postprandial'::text, 'random'::text]))", name: "ck_consultations_glucose"
    t.check_constraint "heart_rate IS NULL OR heart_rate BETWEEN 20 AND 250", name: "ck_consultations_heart_rate"
    t.check_constraint "height_cm IS NULL OR height_cm BETWEEN 30 AND 250", name: "ck_consultations_height"
    t.check_constraint "pain_score IS NULL OR pain_score BETWEEN 0 AND 10", name: "ck_consultations_pain_score"
    t.check_constraint "respiratory_rate IS NULL OR respiratory_rate BETWEEN 4 AND 80", name: "ck_consultations_respiratory_rate"
    t.check_constraint "spo2 IS NULL OR spo2 BETWEEN 50 AND 100", name: "ck_consultations_spo2"
    t.check_constraint "status::text = ANY (ARRAY['draft'::text, 'finalized'::text])", name: "ck_consultations_status"
    t.check_constraint "systolic IS NULL OR systolic BETWEEN 50 AND 300", name: "ck_consultations_systolic"
    t.check_constraint "temperature_c IS NULL OR temperature_c BETWEEN 30 AND 45", name: "ck_consultations_temperature"
    t.check_constraint "weight_kg IS NULL OR weight_kg BETWEEN 0.5 AND 400", name: "ck_consultations_weight"
  end
```

(O Postgres normaliza `BETWEEN 10 AND 500` de `ck_consultation_addenda_reason` para `>= 10 AND <= 500`; se a paridade acusar o contrário, copie a forma do erro **só para esta expressão** — é a normalização do banco, não diferença de regra.)

Nas `add_foreign_key` do fim, em ordem alfabética: `add_foreign_key "clinical_record_openings", "patients"`, `add_foreign_key "clinical_record_openings", "users"`, `add_foreign_key "consultation_addenda", "clinical_record_openings", column: "opening_id"`, `add_foreign_key "consultation_addenda", "consultations"`, `add_foreign_key "consultation_addenda", "users", column: "author_user_id"`, `add_foreign_key "consultation_conducts", "consultation_addenda", column: "addendum_id"`, `add_foreign_key "consultation_conducts", "consultations"`, `add_foreign_key "consultation_exam_requests", "consultation_addenda", column: "addendum_id"`, `add_foreign_key "consultation_exam_requests", "consultations"`, `add_foreign_key "consultation_problems", "consultation_addenda", column: "addendum_id"`, `add_foreign_key "consultation_problems", "consultations"`, `add_foreign_key "consultation_problems", "patient_problems"`, `add_foreign_key "consultations", "attendances"`, `add_foreign_key "consultations", "patients"`, `add_foreign_key "consultations", "professional_links"`, `add_foreign_key "consultations", "users", column: "author_user_id"`, `add_foreign_key "patient_problem_events", "consultation_addenda", column: "addendum_id", deferrable: :deferred`, `add_foreign_key "patient_problem_events", "consultations", deferrable: :deferred`.

- [ ] **Step 6: Modelos, cifra e a marca da re-cifra**

```ruby
# app/models/consultation.rb
# A consulta da APS (ADR 0031; spec §4): uma por atendimento, do profissional
# que chamou. Rascunho (salvo automaticamente, só do autor; itens em
# draft_items) → finalizada, imutável (trigger); correção é adendo. S, O, A, P
# cifrados com a chave da cidade; sinais vitais com os limites do módulo 18.
class Consultation < ApplicationRecord
  STATUSES = %w[draft finalized].freeze
  TEXT_FIELDS = %w[subjective objective assessment plan].freeze
  MAX_TEXT = 20_000
  VITAL_COLUMNS = ScreeningRevision::VITAL_COLUMNS

  encrypts :subjective, :objective, :assessment, :plan

  belongs_to :attendance
  belongs_to :patient
  belongs_to :author_user, class_name: "User"
  belongs_to :professional_link
  has_many :problem_items, class_name: "ConsultationProblem", dependent: :restrict_with_error
  has_many :conducts, class_name: "ConsultationConduct", dependent: :restrict_with_error
  has_many :exam_requests, class_name: "ConsultationExamRequest", dependent: :restrict_with_error
  has_many :addenda, class_name: "ConsultationAddendum", dependent: :restrict_with_error

  scope :finalized_consultations, -> { where(status: "finalized") }

  def draft? = status == "draft"
  def finalized? = status == "finalized"

  def vitals = VITAL_COLUMNS.to_h { |column| [ column, self[column] ] }.compact
end
```

```ruby
# app/models/consultation_problem.rb
# Problema avaliado na consulta (ou mudado por adendo), com a situação no
# momento (ADR 0031; spec §4). Só acréscimo.
class ConsultationProblem < ApplicationRecord
  ACTIONS = %w[evaluate add resolve correct_onset].freeze

  belongs_to :consultation
  belongs_to :addendum, class_name: "ConsultationAddendum", optional: true
  belongs_to :patient_problem
end
```

```ruby
# app/models/consultation_conduct.rb
# Conduta LEDI da consulta (ADR 0031; Task 1). Adendo acrescenta ou remove
# (linha `remove`); o efetivo é Consultations::Effective. Só acréscimo.
class ConsultationConduct < ApplicationRecord
  ACTIONS = %w[add remove].freeze

  belongs_to :consultation
  belongs_to :addendum, class_name: "ConsultationAddendum", optional: true
end
```

```ruby
# app/models/consultation_exam_request.rb
# Exame solicitado (SIGTAP grupo 02, com a competência) e a justificativa
# CID-10 opcional (ADR 0031; Task 1). Adendo cancela com linha `cancelled`.
class ConsultationExamRequest < ApplicationRecord
  STATUSES = %w[requested cancelled].freeze

  belongs_to :consultation
  belongs_to :addendum, class_name: "ConsultationAddendum", optional: true
end
```

```ruby
# app/models/consultation_addendum.rb
# Adendo de consulta finalizada (ADR 0031; spec §4): só acréscimo, com motivo
# (10–500) e texto cifrado; pode mudar problemas, condutas e exames (changes).
# De outro autor, só com abertura justificada válida (opening).
class ConsultationAddendum < ApplicationRecord
  self.table_name = "consultation_addenda"

  MIN_REASON = 10
  MAX_REASON = 500

  encrypts :text

  belongs_to :consultation
  belongs_to :author_user, class_name: "User"
  belongs_to :opening, class_name: "ClinicalRecordOpening", optional: true
end
```

```ruby
# app/models/clinical_record_opening.rb
# Abertura justificada do prontuário fora de contexto (ADR 0031; spec §5):
# motivo de lista, nota (cifrada) só com `other`, válida 30 minutos para
# aquele usuário e paciente. Só acréscimo.
class ClinicalRecordOpening < ApplicationRecord
  REASONS = %w[case_review active_search continuity_of_care other].freeze
  VALIDITY = 30.minutes
  MIN_NOTE = 10
  MAX_NOTE = 500

  encrypts :reason_note

  belongs_to :patient
  belongs_to :user

  scope :valid_for, ->(user_id:, patient_id:, now: Time.current) {
    where(user_id: user_id, patient_id: patient_id).where("expires_at > ?", now)
  }
end
```

Em `app/models/patient.rb`, depois de `has_many :problems …`: `has_many :consultations, dependent: :restrict_with_error`. Em `app/models/attendance.rb`, depois de `has_one :screening …`: `has_one :consultation, dependent: :restrict_with_error`.

Em `app/services/city_encryption.rb`, no fim de `CITY_KEYED_TARGETS`:

```ruby
    [ Patient,        :sex ],
    # ADR 0031: texto clínico (imutável; a re-cifra passa pela marca abaixo).
    [ Consultation,   :subjective ],
    [ Consultation,   :objective ],
    [ Consultation,   :assessment ],
    [ Consultation,   :plan ],
    [ ConsultationAddendum, :text ],
    [ ClinicalRecordOpening, :reason_note ]
```

e, depois de `module_function`:

```ruby
  # ADR 0031 (Desvio 4): consulta finalizada, adendo e abertura são imutáveis
  # no banco; a única escrita aceita é regravar a coluna cifrada sob esta marca
  # (SET LOCAL vale só para a transação). Usado por ReencryptionJob e CityRekey.
  def allowing_reencryption
    ApplicationRecord.transaction do
      ApplicationRecord.connection.execute("SET LOCAL rota.reencrypting = 'on'")
      yield
    end
  end
```

Em `app/jobs/reencryption_job.rb#reencrypt`, troque `record.encrypt` por `CityEncryption.allowing_reencryption { record.encrypt }`. Em `app/commands/city_rekey.rb#call`, troque `ApplicationRecord.transaction do` (o que envolve `TARGETS.each`) por `CityEncryption.allowing_reencryption do` e acrescente ao comentário "Tudo-ou-nada": `# ADR 0031: a transação leva a marca rota.reencrypting (consulta finalizada, adendo e abertura só aceitam a regravação do texto cifrado sob ela).`

- [ ] **Step 7: Rode a migração nos bancos de teste e as specs**

```bash
psql -U rota_saude -d postgres -c "DROP DATABASE rota_saude_test_city_a" -c "DROP DATABASE rota_saude_test_city_b"
docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod19 api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/models/consultation_tables_guard_spec.rb spec/commands/clinical_record_reencryption_spec.rb spec/models/patient_tables_guard_spec.rb spec/services/city_schema_spec.rb spec/architecture spec/jobs/reencryption_job_spec.rb spec/commands/city_rekey_spec.rb spec/tasks/city_rekey_rake_spec.rb spec/commands/patients
```
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add db/city_migrate/20261007400002_add_consultations.rb db/city_schema.rb db/city_triggers.sql app/models/consultation.rb app/models/consultation_problem.rb app/models/consultation_conduct.rb app/models/consultation_exam_request.rb app/models/consultation_addendum.rb app/models/clinical_record_opening.rb app/models/patient.rb app/models/attendance.rb app/services/city_encryption.rb app/jobs/reencryption_job.rb app/commands/city_rekey.rb spec/models/consultation_tables_guard_spec.rb spec/commands/clinical_record_reencryption_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: add consultations, addenda and openings with immutability guards

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 8: Quem registra, terminologias da consulta e a validação dos itens (`Consultations::ItemsInput`)

**Files:**
- Create: `app/services/consultations/cbos.rb`, `app/services/consultations/authorization.rb`, `app/services/consultations/care_type.rb`, `app/services/consultations/items_input.rb`
- Create: `app/services/clinical_terms.rb`, `app/services/clinical_terms/sigtap_exams.rb`
- Test: `spec/services/consultations/authorization_spec.rb`, `spec/services/clinical_terms_spec.rb`, `spec/services/consultations/items_input_spec.rb`

**Interfaces:**
- Consumes: `Ledi::ScreeningMapping.miai_cbo?` (18), `Ledi::ConsultationMapping` (Task 1), `Screenings::VitalSigns.parse` (18), `Terminology::Sigtap.release_for` (16), `Consultation::{TEXT_FIELDS,MAX_TEXT,VITAL_COLUMNS}`, `PatientProblem` (Tasks 3, 7).
- Produces:
  - `Consultations::Cbos::EXCLUDED_PREFIXES == %w[2232]`, `.allowed?(cbo) -> bool`;
  - `Consultations::Authorization.link_for(user:, health_unit_id:) -> [Symbol, ProfessionalLink|nil]` (`:ok`, `:missing_role`, `:missing_link`, `:cbo_not_allowed`; chame dentro de transação — FOR SHARE), `.any_link(user:) -> Symbol` (mesmos códigos, em qualquer unidade);
  - `Consultations::CareType::SCHEDULED == 2`, `::SAME_DAY == 5`, `.suggest(attendance) -> Integer`;
  - `ClinicalTerms::Code = Data(:terminology, :code, :label, :release_id)`, `ClinicalTerms.release(terminology)`, `.find(terminology, code) -> Code|nil` (código sem ponto, maiúsculo), `.label(terminology, code, release_id) -> String|nil`, `.search(terminology, query, limit: 20) -> Array<Code>`, `.cid10_sex(code, release_id) -> "F"|"M"|nil`;
  - `ClinicalTerms::SigtapExams::Exam = Data(:code, :label, :competence)`, `.release(on:)`, `.find(code, on:) -> Exam|nil` (só grupo 02), `.label(code, competence) -> String|nil`, `.search(query, on:, limit: 20) -> Array<Exam>`;
  - `Consultations::ItemsInput::MAX_PROBLEMS == 50`, `.call(params, patient:, cbo:, on: Time.zone.today) -> Result` — ok `{ attrs: Hash (colunas presentes: textos, sinais, care_type), draft_items: Hash (chaves presentes: "evaluated_problems", "conducts", "exam_requests", já normalizadas) }`; falhas `:invalid_text`/`:text_too_long` (`field`), `:implausible_vital` (`field`), `:invalid_care_type`, `:invalid_problem`/`:invalid_onset`/`:cid10_not_allowed_for_cbo`/`:cid10_sex_incompatible` (`index`), `:invalid_conduct`, `:invalid_exam` (`index`); e `.problems(list, patient:, cbo:, on:)`, `.conducts(list)`, `.exams(list, cbo:, on:)` (mesmos resultados, para o adendo). Item de problema normalizado: `{ "problem_id", "terminology", "code", "release_id", "action", "onset_on" ("AAAA-MM-DD"|nil), "onset_precision" }`; exame: `{ "sigtap_code", "sigtap_competence", "cid10_justification" }`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/consultations/authorization_spec.rb
require "rails_helper"

# ADR 0031 (spec §4; Desvio 2): consulta por profissional com papel, vínculo
# ativo na unidade e CBO da tabela do MIAI fora de 2232.
RSpec.describe Consultations::Authorization do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }

  def check(user) = ApplicationRecord.transaction { described_class.link_for(user: user, health_unit_id: unit.id) }

  { "225125" => :ok, "223505" => :ok, "225142" => :ok, "223293" => :cbo_not_allowed, "322205" => :cbo_not_allowed,
    "223208" => :cbo_not_allowed }.each do |cbo, expected|
    it("CBO #{cbo} → #{expected}") { expect(check(doctor!(unit, cbo: cbo)).first).to eq(expected) }
  end

  it "sem papel, sem vínculo na unidade; dois vínculos, vale o permitido" do
    expect(check(reception!)).to eq([ :missing_role, nil ])
    expect(check(doctor!(create_unit("UBS Outra"))).first).to eq(:missing_link)
    user = doctor!(unit, cbo: "322205")
    link_professional!(user, unit, cbo: "225125")
    status, link = check(user)
    expect([ status, link.cbo_code ]).to eq([ :ok, "225125" ])
    expect(described_class.any_link(user: user)).to eq(:ok)
    expect(described_class.any_link(user: reception!)).to eq(:missing_role)
  end

  it "tipo sugerido: horário marcado → consulta agendada; demanda espontânea → consulta no dia" do
    walk_in = walk_in_attendance!(unit, citizen: screening_citizen!(1))
    scheduled = scheduled_attendance!(unit, citizen: screening_citizen!(2))
    expect([ Consultations::CareType.suggest(walk_in), Consultations::CareType.suggest(scheduled) ]).to eq([ 5, 2 ])
    expect([ 2, 5 ]).to all(satisfy { |code| Ledi::ConsultationMapping.care_type?(code) })
  end
end
```

(`223208` — cirurgião-dentista clínico geral — está na tabela do MIAI e começa por `2232`: prova o recorte da spec, não o do layout. Se a Task 1 mostrar que ele não está no MIAI, troque por outro `2232xx` da tabela.)

```ruby
# spec/services/clinical_terms_spec.rb
require "rails_helper"

# ADR 0031 (spec §7; contratos §5): CIAP-2 e CID-10 da release ativa da
# plataforma; SIGTAP da competência ativa, só exames (grupo 02).
RSpec.describe ClinicalTerms do
  before { ciap2_release!; cid10_release!; sigtap_release! }

  it "acha por código com ou sem ponto, com rótulo e release; desconhecido é nil" do
    code = described_class.find("cid10", "e11.9")
    expect([ code.code, code.label, code.release_id ])
      .to eq([ "E119", ClinicalRecordHelpers::CID10["E119"].first, TerminologyRelease.active.find_by!(kind: "cid10").id ])
    expect(described_class.find("ciap2", "t90").code).to eq("T90")
    expect(described_class.find("cid10", "Z999")).to be_nil
    expect(described_class.find("loinc", "1")).to be_nil
    expect(described_class.cid10_sex("C61", code.release_id)).to eq("M")
  end

  it "busca CID-10 por código ou nome, sem acento e sem caixa, até o limite" do
    expect(described_class.search("cid10", "hipertensao").map(&:code)).to eq([ "I10" ])
    expect(described_class.search("cid10", "e11").map(&:code)).to eq([ "E119" ])
    expect(described_class.search("cid10", "").map(&:code)).to eq([])
    expect(described_class.search("cid10", "a", limit: 2).size).to eq(2)
  end

  it "SIGTAP: só grupo 02, com a competência; busca por código ou nome" do
    exam = ClinicalTerms::SigtapExams.find("0202010503", on: Time.zone.today)
    expect([ exam.code, exam.label, exam.competence ])
      .to eq([ "0202010503", "DOSAGEM DE HEMOGLOBINA GLICOSILADA", Time.zone.today.strftime("%Y%m") ])
    expect(ClinicalTerms::SigtapExams.find("0301010064", on: Time.zone.today)).to be_nil
    expect(ClinicalTerms::SigtapExams.find("02.02.01.050-3", on: Time.zone.today)&.code).to eq("0202010503")
    expect(ClinicalTerms::SigtapExams.search("glicosilada", on: Time.zone.today).map(&:code)).to eq([ "0202010503" ])
    expect(ClinicalTerms::SigtapExams.search("consulta", on: Time.zone.today)).to eq([])
  end
end
```

```ruby
# spec/services/consultations/items_input_spec.rb
require "rails_helper"

# ADR 0031 (spec §4; Task 1): o corpo do autosave e do adendo → colunas e
# itens normalizados. Review Focus 3: entrada nas bordas vira 422 com
# field/index, nunca 500 nem dado parcial.
RSpec.describe Consultations::ItemsInput do
  before { Current.city = TEST_CITY_A; ciap2_release!; cid10_release!; sigtap_release! }
  after { Current.reset }

  let(:citizen) { verified_citizen!(1, age: 46, sex: "female") }
  let(:patient) { Patient.create!(cpf: citizen.cpf, birth_date: citizen.birth_date, sex: "female") }
  let(:today) { Time.zone.today }

  def input(params, cbo: "225125") = described_class.call(params, patient: patient, cbo: cbo, on: today)

  def failure(params, **opts)
    result = input(params, **opts)
    [ result.reason, result.details[:field] || result.details[:index] ]
  end

  it "só as chaves presentes; texto preservado, vazio vira nil; sinais substituem; itens normalizados" do
    result = input("subjective" => "  dor\r\nlombar ", "plan" => "", "vitals" => { "systolic" => "130", "diastolic" => 85 },
                   "care_type" => 5, "conducts" => [ 1, 9 ],
                   "evaluated_problems" => [ { "terminology" => "cid10", "code" => "e11.9", "action" => "add",
                                               "onset_on" => "2025-08-17", "onset_precision" => "month" } ],
                   "exam_requests" => [ { "sigtap_code" => "0202010503", "cid10_justification" => "E119" } ],
                   "desconhecida" => 1)
    expect(result).to be_ok
    attrs = result.payload[:attrs]
    expect(attrs.slice("subjective", "plan", "care_type", "systolic", "diastolic", "spo2"))
      .to eq("subjective" => "  dor\r\nlombar ", "plan" => nil, "care_type" => 5, "systolic" => 130, "diastolic" => 85, "spo2" => nil)
    expect(attrs).not_to have_key("objective")
    expect(result.payload[:draft_items]).to eq(
      "conducts" => [ 1, 9 ],
      "evaluated_problems" => [ { "problem_id" => nil, "terminology" => "cid10", "code" => "E119",
                                  "release_id" => TerminologyRelease.active.find_by!(kind: "cid10").id, "action" => "add",
                                  "onset_on" => "2025-08-01", "onset_precision" => "month" } ],
      "exam_requests" => [ { "sigtap_code" => "0202010503", "sigtap_competence" => today.strftime("%Y%m"),
                             "cid10_justification" => "E119" } ]
    )
  end

  it "avaliar e resolver usam o problema do paciente" do
    problem = ApplicationRecord.transaction do
      Patients::ApplyProblemEvent.call(patient: patient, action: "add", by: verifier!, source: { consultation: Struct.new(:id).new(SecureRandom.uuid) },
                                       terminology: "ciap2", code: "T90", release_id: TerminologyRelease.active.find_by!(kind: "ciap2").id)
                                 .payload[:problem]
    end
    result = input("evaluated_problems" => [ { "problem_id" => problem.id, "action" => "resolve" } ])
    expect(result.payload[:draft_items]["evaluated_problems"].sole)
      .to include("problem_id" => problem.id, "terminology" => "ciap2", "code" => "T90", "action" => "resolve")
    other = Patient.create!(cpf: verified_citizen!(2).cpf)
    expect(described_class.call({ "evaluated_problems" => [ { "problem_id" => problem.id, "action" => "evaluate" } ] },
                                patient: other, cbo: "225125", on: today).reason).to eq(:invalid_problem)
  end

  {
    { "subjective" => "x" * 20_001 } => [ :text_too_long, "subjective" ],
    { "plan" => [ "a" ] } => [ :invalid_text, "plan" ],
    { "vitals" => { "systolic" => 130 } } => [ :implausible_vital, "diastolic" ],
    { "vitals" => { "diastolic" => 80 } } => [ :implausible_vital, "systolic" ],
    { "vitals" => { "spo2" => "37,5" } } => [ :implausible_vital, "spo2" ],
    { "vitals" => "lixo" } => [ :implausible_vital, "vitals" ],
    { "care_type" => 4 } => [ :invalid_care_type, nil ],
    { "care_type" => "5" } => [ :invalid_care_type, nil ],
    { "conducts" => [ 3 ] } => [ :invalid_conduct, nil ],
    { "conducts" => [ 1, 1 ] } => [ :invalid_conduct, nil ],
    { "conducts" => (1..13).to_a } => [ :invalid_conduct, nil ],
    { "conducts" => "9" } => [ :invalid_conduct, nil ],
    { "evaluated_problems" => "T90" } => [ :invalid_problem, nil ],
    { "evaluated_problems" => [ "T90" ] } => [ :invalid_problem, 0 ],
    { "evaluated_problems" => [ { "terminology" => "ciap2", "code" => "Z99", "action" => "add" } ] } => [ :invalid_problem, 0 ],
    { "evaluated_problems" => [ { "terminology" => "ciap2", "code" => "T90", "action" => "add" },
                                { "terminology" => "ciap2", "code" => "t90", "action" => "add" } ] } => [ :invalid_problem, 1 ],
    { "evaluated_problems" => [ { "action" => "evaluate", "problem_id" => "nao-e-uuid" } ] } => [ :invalid_problem, 0 ],
    { "evaluated_problems" => [ { "terminology" => "ciap2", "code" => "T90", "action" => "add", "onset_on" => "2999-01-01",
                                  "onset_precision" => "day" } ] } => [ :invalid_onset, 0 ],
    { "evaluated_problems" => [ { "terminology" => "ciap2", "code" => "T90", "action" => "add", "onset_on" => "1900-01-01",
                                  "onset_precision" => "year" } ] } => [ :invalid_onset, 0 ],
    { "evaluated_problems" => [ { "terminology" => "ciap2", "code" => "T90", "action" => "add", "onset_on" => "2025-02-30",
                                  "onset_precision" => "day" } ] } => [ :invalid_onset, 0 ],
    { "evaluated_problems" => [ { "terminology" => "ciap2", "code" => "T90", "action" => "add", "onset_on" => "2025-02-01" } ] } =>
      [ :invalid_onset, 0 ],
    { "evaluated_problems" => [ { "terminology" => "cid10", "code" => "C61", "action" => "add" } ] } => [ :cid10_sex_incompatible, 0 ],
    { "exam_requests" => [ { "sigtap_code" => "0301010064" } ] } => [ :invalid_exam, 0 ],
    { "exam_requests" => [ { "sigtap_code" => "0202010503" }, { "sigtap_code" => "0202010503" } ] } => [ :invalid_exam, 1 ],
    { "exam_requests" => [ { "sigtap_code" => "0202010503", "cid10_justification" => "Z999" } ] } => [ :invalid_exam, 0 ],
    { "exam_requests" => Array.new(101) { { "sigtap_code" => "0202010503" } } } => [ :invalid_exam, nil ]
  }.each do |params, expected|
    it("#{params.inspect.truncate(90)} → #{expected.inspect}") { expect(failure(params)).to eq(expected) }
  end

  it "CID-10 recusado ao CBO quando a regra restringe" do
    allow(Ledi::ConsultationMapping).to receive(:cid10_allowed?).and_return(false)
    expect(failure("evaluated_problems" => [ { "terminology" => "cid10", "code" => "E119", "action" => "add" } ]))
      .to eq([ :cid10_not_allowed_for_cbo, 0 ])
    expect(failure("exam_requests" => [ { "sigtap_code" => "0202010503", "cid10_justification" => "E119" } ]))
      .to eq([ :cid10_not_allowed_for_cbo, 0 ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/consultations spec/services/clinical_terms_spec.rb`
Expected: FAIL (`uninitialized constant Consultations::Authorization`).

- [ ] **Step 3: Quem registra e o tipo sugerido**

```ruby
# app/services/consultations/cbos.rb
# Quem registra consulta (ADR 0031; spec §4; Desvio 2): CBO de nível superior
# que o layout aceita no Atendimento Individual (tabela do MIAI, módulo 18) e
# fora da odontologia (2232, módulo 28).
module Consultations
  module Cbos
    EXCLUDED_PREFIXES = %w[2232].freeze

    module_function

    def allowed?(cbo)
      cbo = cbo.to_s
      Ledi::ScreeningMapping.miai_cbo?(cbo) && EXCLUDED_PREFIXES.none? { |prefix| cbo.start_with?(prefix) }
    end
  end
end
```

```ruby
# app/services/consultations/authorization.rb
# Papel health_professional, vínculo ativo e CBO permitido (ADR 0031). Chame
# DENTRO da transação do comando: o FOR SHARE no vínculo faz o encerramento
# esperar (como Professionals::ClinicalAuthorization).
module Consultations
  module Authorization
    module_function

    def link_for(user:, health_unit_id:)
      return [ :missing_role, nil ] unless user&.has_role?("health_professional")

      links = active_links(user).where(health_unit_id: health_unit_id.to_s).lock("FOR SHARE OF professional_links").to_a
      pick(links)
    end

    # Para a abertura justificada: um vínculo permitido em qualquer unidade.
    def any_link(user:)
      return :missing_role unless user&.has_role?("health_professional")

      pick(active_links(user).to_a).first
    end

    def active_links(user)
      ProfessionalLink.active.joins(:professional).where(professionals: { user_id: user.id }).order(:started_at, :id)
    end

    def pick(links)
      return [ :missing_link, nil ] if links.empty?

      allowed = links.find { |link| Cbos.allowed?(link.cbo_code) }
      allowed ? [ :ok, allowed ] : [ :cbo_not_allowed, nil ]
    end
    private_class_method :active_links, :pick
  end
end
```

```ruby
# app/services/consultations/care_type.rb
# Tipo de atendimento sugerido (spec §4; Task 1): horário marcado → consulta
# agendada (2); o resto (demanda espontânea, com ou sem escuta same_day) →
# consulta no dia (5). O profissional edita.
module Consultations
  module CareType
    SCHEDULED = 2
    SAME_DAY = 5

    module_function

    def suggest(attendance) = attendance.appointment_id ? SCHEDULED : SAME_DAY
  end
end
```

- [ ] **Step 4: Terminologias**

```ruby
# app/services/clinical_terms.rb
# CIAP-2 e CID-10 da plataforma (ADR 0028/0031) para problemas da consulta:
# a release ativa; o item guarda o código e a release (a leitura usa a
# gravada). CID-10 sem ponto e maiúsculo ("E11.9" → "E119"). A busca filtra em
# Ruby um índice dobrado (sem acento, minúsculo) por release, em memória.
module ClinicalTerms
  Code = Data.define(:terminology, :code, :label, :release_id)
  MODELS = { "ciap2" => "Ciap2Code", "cid10" => "Cid10Code" }.freeze

  module_function

  def release(terminology)
    return nil unless MODELS.key?(terminology.to_s)

    TerminologyRelease.active.where(kind: terminology.to_s).order(activated_at: :desc, id: :desc).first
  end

  def find(terminology, code)
    current = release(terminology)
    return nil if current.nil? || !code.is_a?(String) || code.blank?

    row = model(terminology).find_by(release_id: current.id, code: normalize(code))
    row && Code.new(terminology: terminology.to_s, code: row.code, label: row.description, release_id: current.id)
  end

  def label(terminology, code, release_id)
    return nil unless MODELS.key?(terminology.to_s)

    model(terminology).find_by(release_id: release_id, code: code)&.description
  end

  def cid10_sex(code, release_id) = Cid10Code.find_by(release_id: release_id, code: code)&.sex_restriction

  def search(terminology, query, limit: 20)
    current = release(terminology)
    text = query.is_a?(String) ? query.strip : ""
    return [] if current.nil? || text.empty?

    folded = fold(text)
    plain = folded.delete(".")
    index(terminology, current).select { |code, _label, label_folded| code.downcase.start_with?(plain) || label_folded.include?(folded) }
                               .first(limit)
                               .map { |code, label, _| Code.new(terminology: terminology.to_s, code: code, label: label, release_id: current.id) }
  end

  def normalize(code) = code.to_s.strip.upcase.delete(".")
  def fold(text) = I18n.transliterate(text.to_s).downcase
  def model(terminology) = MODELS.fetch(terminology.to_s).constantize

  def index(terminology, release)
    @index ||= {}
    @index[release.id] ||= model(terminology).where(release_id: release.id).order(:code).pluck(:code, :description)
                                             .map { |code, label| [ code, label, fold(label) ] }.freeze
  end
  private_class_method :model, :index
end
```

```ruby
# app/services/clinical_terms/sigtap_exams.rb
# Exames solicitáveis (ADR 0031; Task 1): procedimento SIGTAP do grupo 02 na
# release ativa da competência corrente (Terminology::Sigtap.release_for). O
# pedido guarda o código e a competência.
module ClinicalTerms
  module SigtapExams
    Exam = Data.define(:code, :label, :competence)

    module_function

    def release(on:) = Terminology::Sigtap.release_for(on.strftime("%Y%m"))

    def find(code, on:)
      current = release(on: on)
      return nil if current.nil? || !code.is_a?(String)

      digits = code.delete("^0-9")
      return nil unless digits.match?(/\A\d{10}\z/) && digits.start_with?(Ledi::ConsultationMapping.exam_group_prefix)

      row = SigtapProcedure.find_by(release_id: current.id, code: digits)
      row && Exam.new(code: row.code, label: row.name, competence: current.version)
    end

    def label(code, competence)
      current = Terminology::Sigtap.release_for(competence)
      current && SigtapProcedure.find_by(release_id: current.id, code: code)&.name
    end

    def search(query, on:, limit: 20)
      current = release(on: on)
      text = query.is_a?(String) ? query.strip : ""
      return [] if current.nil? || text.empty?

      folded = ClinicalTerms.fold(text)
      digits = text.delete("^0-9")
      SigtapProcedure.where(release_id: current.id).where("code LIKE ?", "#{Ledi::ConsultationMapping.exam_group_prefix}%")
                     .order(:code).pluck(:code, :name)
                     .select { |code, name| (digits.present? && code.start_with?(digits)) || ClinicalTerms.fold(name).include?(folded) }
                     .first(limit).map { |code, name| Exam.new(code: code, label: name, competence: current.version) }
    end
  end
end
```

(`ClinicalTerms.fold` é público: o `private_class_method` só cobre `model` e `index`.)

- [ ] **Step 5: `Consultations::ItemsInput`**

```ruby
# app/services/consultations/items_input.rb
# Corpo do autosave (e do adendo) → colunas e itens normalizados (ADR 0031;
# spec §4; contratos §4; Task 1). Só as chaves presentes mudam; texto clínico
# fica como veio (vazio = nil); sinais com o validador do módulo 18 (pressão
# incompleta = implausible_vital com o lado que falta); itens contra a release
# ativa. Nunca levanta: entrada ruim vira Result.fail com field/index.
module Consultations
  module ItemsInput
    MAX_PROBLEMS = 50
    DATE = /\A\d{4}-\d{2}-\d{2}\z/

    module_function

    def call(params, patient:, cbo:, on: Time.zone.today)
      params = normalize(params)
      attrs = {}
      items = {}

      Consultation::TEXT_FIELDS.each do |field|
        next unless params.key?(field)

        value = params[field]
        return Result.fail(:invalid_text, details: { field: field }) unless value.nil? || value.is_a?(String)
        return Result.fail(:text_too_long, details: { field: field }) if value.to_s.length > Consultation::MAX_TEXT

        attrs[field] = value.presence && (value.strip.empty? ? nil : value)
      end

      if params.key?("vitals")
        vitals = vitals(params["vitals"])
        return vitals if vitals.failure?

        attrs.merge!(vitals.payload)
      end

      if params.key?("care_type")
        value = params["care_type"]
        return Result.fail(:invalid_care_type) unless value.nil? || Ledi::ConsultationMapping.care_type?(value)

        attrs["care_type"] = value
      end

      { "evaluated_problems" => -> { problems(params["evaluated_problems"], patient: patient, cbo: cbo, on: on) },
        "conducts" => -> { conducts(params["conducts"]) },
        "exam_requests" => -> { exams(params["exam_requests"], cbo: cbo, on: on) } }.each do |key, check|
        next unless params.key?(key)

        result = check.call
        return result if result.failure?

        items[key] = result.payload[:items]
      end

      Result.ok(attrs: attrs, draft_items: items)
    end

    def vitals(raw)
      parsed = Screenings::VitalSigns.parse(raw)
      if parsed.failure?
        return parsed unless parsed.reason == :bp_incomplete

        side = raw.is_a?(Hash) && raw.transform_keys(&:to_s)["systolic"].to_s.strip.present? ? "diastolic" : "systolic"
        return Result.fail(:implausible_vital, details: { field: side })
      end
      Result.ok(Consultation::VITAL_COLUMNS.to_h { |column| [ column, parsed.payload[:values][column] ] })
    end

    def problems(list, patient:, cbo:, on:)
      return fail_index(:invalid_problem, nil) unless list.is_a?(Array) && list.size <= MAX_PROBLEMS

      seen = []
      items = list.each_with_index.map do |raw, index|
        item = problem(raw, patient: patient, cbo: cbo, on: on, index: index)
        return item if item.is_a?(Result)

        key = item["problem_id"] || [ item["terminology"], item["code"] ]
        return fail_index(:invalid_problem, index) if seen.include?(key)

        seen << key
        item
      end
      Result.ok(items: items)
    end

    def conducts(list)
      valid = list.is_a?(Array) && list.size <= Ledi::ConsultationMapping.max_conducts && list.uniq.size == list.size &&
              list.all? { |code| Ledi::ConsultationMapping.conduct?(code) }
      valid ? Result.ok(items: list) : Result.fail(:invalid_conduct)
    end

    def exams(list, cbo:, on:)
      return fail_index(:invalid_exam, nil) unless list.is_a?(Array) && list.size <= Ledi::ConsultationMapping.max_exams

      seen = []
      items = list.each_with_index.map do |raw, index|
        raw = normalize(raw)
        exam = ClinicalTerms::SigtapExams.find(raw["sigtap_code"], on: on)
        return fail_index(:invalid_exam, index) if exam.nil? || seen.include?(exam.code)

        seen << exam.code
        justification = nil
        if raw["cid10_justification"].present?
          cid = ClinicalTerms.find("cid10", raw["cid10_justification"])
          return fail_index(:invalid_exam, index) unless cid
          return fail_index(:cid10_not_allowed_for_cbo, index) unless Ledi::ConsultationMapping.cid10_allowed?(cbo)

          justification = cid.code
        end
        { "sigtap_code" => exam.code, "sigtap_competence" => exam.competence, "cid10_justification" => justification }
      end
      Result.ok(items: items)
    end

    def problem(raw, patient:, cbo:, on:, index:)
      return fail_index(:invalid_problem, index) unless raw.is_a?(Hash) || raw.respond_to?(:to_unsafe_h)

      raw = normalize(raw)
      action = raw["action"].to_s
      return fail_index(:invalid_problem, index) unless Patients::ApplyProblemEvent::ACTIONS.include?(action)

      onset = onset(raw, patient: patient, on: on, required: action == "correct_onset")
      return fail_index(:invalid_onset, index) if onset == :invalid

      if action == "add"
        terminology = raw["terminology"].to_s
        code = ClinicalTerms.find(terminology, raw["code"])
        return fail_index(:invalid_problem, index) unless code

        if terminology == "cid10"
          return fail_index(:cid10_not_allowed_for_cbo, index) unless Ledi::ConsultationMapping.cid10_allowed?(cbo)

          sex = ClinicalTerms.cid10_sex(code.code, code.release_id)
          return fail_index(:cid10_sex_incompatible, index) if sex && Terminology::Sigtap::SEX[patient.sex.to_s] != sex
        end
        return item(nil, terminology, code.code, code.release_id, action, onset)
      end

      found = raw["problem_id"].is_a?(String) && PatientProblem.find_by(id: raw["problem_id"], patient_id: patient.id)
      return fail_index(:invalid_problem, index) unless found
      return fail_index(:invalid_problem, index) if action == "resolve" && !found.active?

      item(found.id, found.terminology, found.code, found.terminology_release_id, action, onset)
    end

    # nil (sem início) | [Date, precisão] | :invalid. Precisão mês/ano grava o
    # primeiro dia; nunca no futuro nem antes do nascimento (dataInicioProblema).
    def onset(raw, patient:, on:, required:)
      date, precision = raw["onset_on"], raw["onset_precision"]
      return (required ? :invalid : nil) if date.nil? && precision.nil?
      return :invalid unless date.is_a?(String) && date.match?(DATE) && PatientProblem::PRECISIONS.include?(precision)

      parsed = Date.iso8601(date)
      parsed = parsed.beginning_of_month if precision == "month"
      parsed = parsed.beginning_of_year if precision == "year"
      birth = patient.birth_date.present? ? Date.iso8601(patient.birth_date) : nil
      return :invalid if parsed > on || (birth && parsed < (precision == "day" ? birth : birth.beginning_of_year))

      [ parsed, precision ]
    rescue Date::Error
      :invalid
    end

    def item(problem_id, terminology, code, release_id, action, onset)
      { "problem_id" => problem_id, "terminology" => terminology, "code" => code, "release_id" => release_id,
        "action" => action, "onset_on" => onset&.first&.iso8601, "onset_precision" => onset&.last }
    end

    def normalize(params)
      params = params.to_unsafe_h if params.respond_to?(:to_unsafe_h)
      params.is_a?(Hash) ? params.deep_stringify_keys : {}
    end

    def fail_index(reason, index) = Result.fail(reason, details: { index: index }.compact)
    private_class_method :vitals, :problem, :onset, :item, :normalize, :fail_index
  end
end
```

(`attrs[field] = value.presence && …`: `nil` e `""` viram `nil`, só espaços também; o resto fica como veio — formatação do texto clínico é do profissional.)

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/consultations spec/services/clinical_terms_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add app/services/consultations/cbos.rb app/services/consultations/authorization.rb app/services/consultations/care_type.rb app/services/consultations/items_input.rb app/services/clinical_terms.rb app/services/clinical_terms/sigtap_exams.rb spec/services/consultations/authorization_spec.rb spec/services/clinical_terms_spec.rb spec/services/consultations/items_input_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: validate consultation items against the LEDI layout and terminologies

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 9: `Consultations::Start` e `Consultations::SaveDraft` (iniciar e autosave)

**Files:**
- Create: `app/commands/consultations/start.rb`, `app/commands/consultations/save_draft.rb`
- Test: `spec/commands/consultations/start_spec.rb`, `spec/commands/consultations/save_draft_spec.rb`

**Interfaces:**
- Consumes: `ClinicalRecord::Gate.usable?` (Task 2), `Patients::Resolve` (Task 5), `Consultations::{Authorization,CareType,ItemsInput}` (Task 8).
- Produces:
  - `Consultations::Start.call(attendance:, by:, city: Current.city) -> Result` (ok `{ consultation: }`; falhas `:feature_disabled`, `:missing_role`, `:missing_link`, `:cbo_not_allowed`, `:not_in_care`, `:not_caller`, `:already_exists` (com `consultation_id`), `:citizen_not_verified`); evento `consultation.started { consultation_id, attendance_id }`;
  - `Consultations::SaveDraft.call(consultation:, params:, by:) -> Result` (ok `{ consultation: }`; falhas `:not_author`, `:not_draft` e as de `ItemsInput`). Itens presentes no corpo substituem a chave em `draft_items`; ausentes ficam.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/commands/consultations/start_spec.rb
require "rails_helper"

# ADR 0031 (spec §4): interruptor utilizável; atendimento in_care chamado por
# quem inicia; CBO permitido; par validado; paciente resolvido; uma por
# atendimento.
RSpec.describe Consultations::Start do
  before { Current.city = clinical_city!; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:citizen) { verified_citizen!(1) }
  let(:attendance) { consulting_attendance!(unit, citizen: citizen, doctor: doctor) }

  it "inicia o rascunho do autor, com vínculo, CBO e tipo sugerido; o paciente nasce; evento só com ids" do
    result = described_class.call(attendance: attendance, by: doctor)
    expect(result).to be_ok
    consultation = result.payload[:consultation]
    link = doctor.professional.links.active.sole
    expect(consultation).to have_attributes(status: "draft", author_user_id: doctor.id, professional_link_id: link.id,
                                            cbo_code: "225125", care_type: 5, attendance_id: attendance.id)
    expect(consultation.patient.cpf).to eq(citizen.cpf)
    expect(citizen.reload.patient_id).to eq(consultation.patient_id)
    expect(DomainEvent.where(name: "consultation.started").sole.payload)
      .to eq("consultation_id" => consultation.id, "attendance_id" => attendance.id)
  end

  it "de novo → already_exists com o id; atendimento com horário sugere consulta agendada" do
    first = described_class.call(attendance: attendance, by: doctor).payload[:consultation]
    again = described_class.call(attendance: attendance, by: doctor)
    expect([ again.reason, again.details ]).to eq([ :already_exists, { consultation_id: first.id } ])
    scheduled = in_care!(scheduled_attendance!(unit, citizen: verified_citizen!(2)), by: doctor)
    expect(described_class.call(attendance: scheduled, by: doctor).payload[:consultation].care_type).to eq(2)
  end

  it "interruptor desligado ou modo diferente de record → feature_disabled" do
    clinical_city!(enabled: false)
    expect(described_class.call(attendance: attendance, by: doctor).reason).to eq(:feature_disabled)
    clinical_city!(record_mode: "integrated")
    expect(described_class.call(attendance: attendance, by: doctor).reason).to eq(:feature_disabled)
    expect(Consultation.count).to eq(0)
  end

  it "par não validado → citizen_not_verified; nada nasce" do
    declared = consulting_attendance!(unit, citizen: screening_citizen!(3), doctor: doctor)
    expect(described_class.call(attendance: declared, by: doctor).reason).to eq(:citizen_not_verified)
    expect([ Consultation.count, Patient.count ]).to eq([ 0, 0 ])
  end

  it "atendimento aguardando, chamado por outra pessoa, papel, vínculo e CBO" do
    waiting = walk_in_attendance!(unit, citizen: verified_citizen!(4))
    expect(described_class.call(attendance: waiting, by: doctor).reason).to eq(:not_in_care)
    colleague = doctor!(unit, cbo: "223505")
    expect(described_class.call(attendance: attendance, by: colleague).reason).to eq(:not_caller)
    expect(described_class.call(attendance: attendance, by: reception!).reason).to eq(:missing_role)
    expect(described_class.call(attendance: attendance, by: doctor!(create_unit("UBS Outra"))).reason).to eq(:missing_link)
    technician = doctor!(unit, cbo: "322205")
    in_care_by_tech = consulting_attendance!(unit, citizen: verified_citizen!(5), doctor: technician)
    expect(described_class.call(attendance: in_care_by_tech, by: technician).reason).to eq(:cbo_not_allowed)
  end
end
```

```ruby
# spec/commands/consultations/save_draft_spec.rb
require "rails_helper"

# ADR 0031 (spec §4): autosave só do autor e só do rascunho; só o que veio
# muda. Review Focus 3: entrada ruim não grava nada.
RSpec.describe Consultations::SaveDraft do
  before { Current.city = clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:consultation) { started_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def save(params, by: doctor) = described_class.call(consultation: consultation, params: params, by: by)

  it "grava texto, sinais, tipo e itens; um autosave parcial não apaga o resto" do
    expect(save(draft_body)).to be_ok
    expect(save("plan" => "Metformina 850 mg")).to be_ok
    consultation.reload
    expect(consultation.slice(:subjective, :plan, :systolic, :care_type))
      .to eq("subjective" => "Refere sede e poliúria há dois meses", "plan" => "Metformina 850 mg", "systolic" => 130, "care_type" => 5)
    expect(consultation.draft_items.keys).to match_array(%w[evaluated_problems conducts exam_requests])
    expect(save("conducts" => [ 9 ])).to be_ok
    expect(consultation.reload.draft_items["conducts"]).to eq([ 9 ])
    expect(consultation.draft_items["evaluated_problems"].sole["code"]).to eq("T90")
  end

  it "entrada inválida não grava nada (nem o texto válido do mesmo corpo)" do
    save(draft_body)
    result = save("subjective" => "novo texto", "vitals" => { "systolic" => 120 })
    expect([ result.reason, result.details ]).to eq([ :implausible_vital, { field: "diastolic" } ])
    expect(consultation.reload.subjective).to eq("Refere sede e poliúria há dois meses")
  end

  it "só o autor; só rascunho" do
    colleague = doctor!(unit, cbo: "223505")
    expect(save({ "plan" => "x" }, by: colleague).reason).to eq(:not_author)
    consultation.update!(status: "finalized", finalized_at: Time.current, care_type: 5, draft_items: {})
    expect(save("plan" => "depois").reason).to eq(:not_draft)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/commands/consultations`
Expected: FAIL (`uninitialized constant Consultations::Start`).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/consultations/start.rb
# Iniciar a consulta (ADR 0031; spec §4): interruptor utilizável; atendimento
# in_care chamado por quem inicia; CBO permitido; par validado; paciente
# resolvido pelo CPF (Patients::Resolve). Uma por atendimento: o lock do
# atendimento serializa duas abas do mesmo profissional; o índice único é a
# última palavra. Ordem de travas: atendimento → CPF → par → paciente.
module Consultations
  class Start
    def self.call(attendance:, by:, city: Current.city)
      return Result.fail(:feature_disabled) unless ClinicalRecord::Gate.usable?(city)

      ApplicationRecord.transaction do
        attendance.lock!
        status, link = Authorization.link_for(user: by, health_unit_id: attendance.health_unit_id)
        next Result.fail(status) unless status == :ok
        next Result.fail(:not_in_care) unless attendance.status == "in_care"
        next Result.fail(:not_caller) unless attendance.called_by_user_id == by.id

        existing = Consultation.find_by(attendance_id: attendance.id)
        next Result.fail(:already_exists, details: { consultation_id: existing.id }) if existing

        resolved = Patients::Resolve.call(attendance.citizen)
        next resolved if resolved.failure?

        consultation = Consultation.create!(
          attendance: attendance, patient: resolved.payload[:patient], author_user: by, professional_link: link,
          cbo_code: link.cbo_code, status: "draft", care_type: CareType.suggest(attendance), started_at: Time.current
        )
        DomainEvents.publish("consultation.started", consultation_id: consultation.id, attendance_id: attendance.id)
        Result.ok(consultation: consultation)
      end
    end
  end
end
```

```ruby
# app/commands/consultations/save_draft.rb
# Autosave do rascunho (ADR 0031; spec §4; contratos §4): só o autor, só
# draft. Valida tudo antes de gravar qualquer coisa (Consultations::ItemsInput);
# itens do corpo substituem a mesma chave de draft_items.
module Consultations
  class SaveDraft
    def self.call(consultation:, params:, by:)
      ApplicationRecord.transaction do
        consultation.lock!
        next Result.fail(:not_author) unless consultation.author_user_id == by.id
        next Result.fail(:not_draft) unless consultation.draft?

        input = ItemsInput.call(params, patient: consultation.patient, cbo: consultation.cbo_code)
        next input if input.failure?

        consultation.update!(input.payload[:attrs].merge("draft_items" => consultation.draft_items.merge(input.payload[:draft_items])))
        Result.ok(consultation: consultation)
      end
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/commands/consultations spec/commands/patients`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add app/commands/consultations/start.rb app/commands/consultations/save_draft.rb spec/commands/consultations/start_spec.rb spec/commands/consultations/save_draft_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: start a consultation for a verified pair and autosave its draft

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: `Consultations::Finalize` — itens, lista de problemas e desfecho numa transação

**Files:**
- Create: `app/commands/consultations/finalize.rb`, `app/services/consultations/effective.rb`
- Modify: `app/commands/attendances/close.rb`, `app/controllers/attendances_controller.rb`
- Test: `spec/commands/consultations/finalize_spec.rb`

**Interfaces:**
- Consumes: `Patients::ApplyProblemEvent` (Task 6), `Attendances::Close.call(attendance:, outcome:, referral_unit_id:, referral_note:, by:)` (existente), `Ledi::ConsultationMapping.cid10_allowed?`.
- Produces:
  - `Consultations::Finalize.call(consultation:, outcome_params:, by:) -> Result` — `outcome_params` = `{ "outcome", "referral_unit_id", "referral_note" }` (o corpo do close existente); ok `{ consultation:, attendance:, appointment_request: }`; falhas `:not_author`, `:not_draft`, `:no_problem_evaluated`, `:no_conduct`, `:assessment_or_plan_required`, `:patient_name_missing`, `:invalid_care_type`, `:cid10_not_allowed_for_cbo` (`index`), `:invalid_problem` (`index`) e as do `Attendances::Close` (`:invalid_outcome`, `:referral_required`, `:invalid_unit`, `:already_closed`, `:invalid_transition`, `:missing_role`, `:missing_link`). Falha = nada muda (savepoint);
  - `Consultations::Effective.call(consultation) -> { problems: Array<ConsultationProblem> (a última linha de cada problema), conducts: Array<Integer>, exam_requests: Array<ConsultationExamRequest> }`;
  - `Attendances::Close` falha com `:consultation_in_progress` (409) quando o atendimento tem consulta `draft`; evento `consultation.finalized { consultation_id, attendance_id }`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/commands/consultations/finalize_spec.rb
require "rails_helper"

# ADR 0031 (spec §4, §8): finalizar exige problema avaliado, conduta, A ou P,
# nome do paciente e CID-10 permitido; numa transação aplica os eventos de
# problema, grava os itens, vira finalized e fecha o atendimento com o
# desfecho (retorno/encaminhamento geram o pedido). Review Focus 4.
RSpec.describe Consultations::Finalize do
  before { Current.city = clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:consultation) { started_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def draft!(**over) = Consultations::SaveDraft.call(consultation: consultation, params: draft_body(**over), by: doctor)

  def finalize(outcome = { "outcome" => "discharged" }, by: doctor)
    described_class.call(consultation: consultation.reload, outcome_params: outcome, by: by)
  end

  it "finaliza: itens, problema com evento da consulta, atendimento fechado; eventos só com ids" do
    draft!
    result = finalize
    expect(result).to be_ok
    consultation.reload
    expect(consultation).to have_attributes(status: "finalized", draft_items: {})
    expect(consultation.finalized_at).to be_present
    problem = PatientProblem.where(patient_id: consultation.patient_id).sole
    expect(problem).to have_attributes(code: "T90", status: "active", onset_on: Date.new(2025, 8, 1), onset_precision: "month")
    expect(problem.events.sole).to have_attributes(kind: "added", consultation_id: consultation.id)
    expect(consultation.problem_items.sole).to have_attributes(patient_problem_id: problem.id, action: "add", status_after: "active")
    expect(consultation.conducts.pluck(:code)).to eq([ 1 ])
    expect(consultation.exam_requests.sole).to have_attributes(sigtap_code: "0202010503", status: "requested")
    expect(consultation.attendance.reload).to have_attributes(status: "closed", outcome: "discharged", closed_by_user_id: doctor.id)
    expect(DomainEvent.where(name: "consultation.finalized").sole.payload)
      .to eq("consultation_id" => consultation.id, "attendance_id" => consultation.attendance_id)
  end

  it "retorno gera o pedido do módulo 17 na mesma transação" do
    draft!
    result = finalize("outcome" => "return")
    expect(result).to be_ok
    expect(result.payload[:appointment_request]).to have_attributes(kind: "return", origin_attendance_id: consultation.attendance_id)
  end

  {
    { "evaluated_problems" => [] } => :no_problem_evaluated,
    { "conducts" => [] } => :no_conduct,
    { "assessment" => "", "plan" => " " } => :assessment_or_plan_required,
    { "care_type" => nil } => :invalid_care_type
  }.each do |over, reason|
    it("#{over.inspect} → #{reason}") do
      draft!(**over.transform_keys(&:to_sym))
      expect(finalize.reason).to eq(reason)
      expect(consultation.reload.status).to eq("draft")
    end
  end

  it "paciente sem nome → patient_name_missing; CID-10 que a regra recusa → cid10_not_allowed_for_cbo" do
    draft!
    consultation.patient.update_columns(full_name: nil)
    expect(finalize.reason).to eq(:patient_name_missing)
    consultation.patient.update_columns(full_name: "Maria Aparecida da Silva")
    draft!(evaluated_problems: [ { "terminology" => "cid10", "code" => "E119", "action" => "add" } ])
    allow(Ledi::ConsultationMapping).to receive(:cid10_allowed?).and_return(false)
    result = finalize
    expect([ result.reason, result.details ]).to eq([ :cid10_not_allowed_for_cbo, { index: 0 } ])
  end

  it "só o autor; finalizada não finaliza de novo" do
    draft!
    expect(finalize(by: doctor!(unit, cbo: "223505")).reason).to eq(:not_author)
    expect(finalize).to be_ok
    expect(finalize.reason).to eq(:not_draft)
  end

  it "falha do desfecho desfaz tudo: nada de item, problema, evento ou atendimento fechado (Review Focus 4)" do
    draft!
    inactive = create_unit("UPA Norte", kind: "upa", active: false)
    result = finalize("outcome" => "referred", "referral_unit_id" => inactive.id)
    expect(result.reason).to eq(:invalid_unit)
    expect(consultation.reload).to have_attributes(status: "draft")
    expect([ ConsultationProblem.count, ConsultationConduct.count, PatientProblem.count, PatientProblemEvent.count ]).to eq([ 0, 0, 0, 0 ])
    expect(DomainEvent.where(name: %w[consultation.finalized patient_problem.changed attendance.closed]).count).to eq(0)
    expect(consultation.attendance.reload.status).to eq("in_care")
    expect(finalize("outcome" => "oriented").reason).to eq(:invalid_outcome)
    expect(consultation.reload.status).to eq("draft")
  end

  it "problema resolvido por outra consulta entre o rascunho e a finalização → invalid_problem, nada muda" do
    draft!
    finalize
    problem = PatientProblem.sole
    second = started_consultation!(unit: unit, doctor: doctor, citizen: consultation.attendance.citizen.reload)
    Consultations::SaveDraft.call(consultation: second, by: doctor,
                                  params: draft_body(evaluated_problems: [ { "problem_id" => problem.id, "action" => "resolve" } ]))
    ApplicationRecord.transaction do
      Patients::ApplyProblemEvent.call(patient: problem.patient, action: "resolve", problem: problem, by: doctor,
                                       source: { consultation: consultation })
    end
    result = described_class.call(consultation: second.reload, outcome_params: { "outcome" => "discharged" }, by: doctor)
    expect([ result.reason, result.details ]).to eq([ :invalid_problem, { index: 0 } ])
    expect(second.reload.status).to eq("draft")
  end

  it "rascunho aberto bloqueia a rota antiga de desfecho (Review Focus 4)" do
    draft!
    result = Attendances::Close.call(attendance: consultation.attendance, outcome: "discharged", referral_unit_id: nil,
                                     referral_note: nil, by: doctor)
    expect(result.reason).to eq(:consultation_in_progress)
    expect(consultation.attendance.reload.status).to eq("in_care")
  end

  it "estado efetivo: última linha de cada problema, condutas e exames" do
    draft!(conducts: [ 1, 9 ])
    finalize
    effective = Consultations::Effective.call(consultation.reload)
    expect(effective[:problems].map(&:code)).to eq([ "T90" ])
    expect(effective[:conducts]).to eq([ 1, 9 ])
    expect(effective[:exam_requests].map(&:sigtap_code)).to eq([ "0202010503" ])
  end
end
```

(O segundo atendimento do mesmo par usa a mesma pessoa — `consultation.attendance.citizen` — depois do primeiro fechado: `walk_in_attendance!` cria outra triagem. Se o check-in recusar dois atendimentos do mesmo par no dia, troque por um segundo par validado do mesmo CPF.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/commands/consultations/finalize_spec.rb`
Expected: FAIL (`uninitialized constant Consultations::Finalize`).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/consultations/finalize.rb
# Finalizar a consulta (ADR 0031; spec §4): requisitos do registro, e numa
# transação só (savepoint — falha não deixa nada): eventos de problema
# (Patients::ApplyProblemEvent), itens só de acréscimo, finalized, e o
# fechamento do atendimento pelo comando existente (retorno/encaminhamento
# geram o pedido). A consulta vira finalized ANTES do Close: o Close recusa
# rascunho aberto. Ordem de travas: atendimento → consulta → paciente.
module Consultations
  class Finalize
    OUTCOME_KEYS = %w[outcome referral_unit_id referral_note].freeze

    def self.call(consultation:, outcome_params:, by:)
      outcome = normalize(outcome_params)
      result = nil
      ApplicationRecord.transaction(requires_new: true) do
        attendance = Attendance.lock.find(consultation.attendance_id)
        consultation.lock!
        next result = Result.fail(:not_author) unless consultation.author_user_id == by.id
        next result = Result.fail(:not_draft) unless consultation.draft?

        patient = Patient.lock.find(consultation.patient_id)
        ready = requirements(consultation, patient)
        next result = ready if ready.failure?

        failure = materialize!(consultation, patient, by)
        if failure
          result = failure
          raise ActiveRecord::Rollback
        end
        consultation.update!(status: "finalized", finalized_at: Time.current, draft_items: {})
        closed = Attendances::Close.call(attendance: attendance, outcome: outcome["outcome"],
                                         referral_unit_id: outcome["referral_unit_id"],
                                         referral_note: outcome["referral_note"], by: by)
        if closed.failure?
          result = closed
          raise ActiveRecord::Rollback
        end
        DomainEvents.publish("consultation.finalized", consultation_id: consultation.id, attendance_id: attendance.id)
        result = Result.ok(consultation: consultation, attendance: closed.payload[:attendance],
                           appointment_request: closed.payload[:appointment_request])
      end
      consultation.reload if result.failure?
      result
    end

    def self.requirements(consultation, patient)
      items = consultation.draft_items
      return Result.fail(:no_problem_evaluated) if Array(items["evaluated_problems"]).empty?
      return Result.fail(:no_conduct) if Array(items["conducts"]).empty?
      return Result.fail(:assessment_or_plan_required) if consultation.assessment.blank? && consultation.plan.blank?
      return Result.fail(:patient_name_missing) if patient.full_name.blank?
      return Result.fail(:invalid_care_type) unless Ledi::ConsultationMapping.care_type?(consultation.care_type)

      Array(items["evaluated_problems"]).each_with_index do |item, index|
        if item["terminology"] == "cid10" && !Ledi::ConsultationMapping.cid10_allowed?(consultation.cbo_code)
          return Result.fail(:cid10_not_allowed_for_cbo, details: { index: index })
        end
      end
      Array(items["exam_requests"]).each_with_index do |exam, index|
        if exam["cid10_justification"] && !Ledi::ConsultationMapping.cid10_allowed?(consultation.cbo_code)
          return Result.fail(:cid10_not_allowed_for_cbo, details: { index: index })
        end
      end
      Result.ok
    end

    # Devolve nil, ou o Result de falha (com o índice do problema).
    def self.materialize!(consultation, patient, by)
      today = Time.zone.today
      items = consultation.draft_items
      Array(items["evaluated_problems"]).each_with_index do |item, index|
        applied = Patients::ApplyProblemEvent.call(
          patient: patient, action: item["action"], by: by, source: { consultation: consultation },
          terminology: item["terminology"], code: item["code"], release_id: item["release_id"],
          problem: item["problem_id"] && PatientProblem.find_by(id: item["problem_id"]),
          onset_on: item["onset_on"] && Date.iso8601(item["onset_on"]), onset_precision: item["onset_precision"], on: today
        )
        return Result.fail(applied.reason, details: { index: index }) if applied.failure?

        record_problem!(consultation, applied.payload[:problem], item["action"])
      end
      Array(items["conducts"]).each { |code| ConsultationConduct.create!(consultation: consultation, code: code, action: "add") }
      Array(items["exam_requests"]).each do |exam|
        ConsultationExamRequest.create!(consultation: consultation, sigtap_code: exam["sigtap_code"],
                                        sigtap_competence: exam["sigtap_competence"],
                                        cid10_justification: exam["cid10_justification"], status: "requested")
      end
      nil
    end

    def self.record_problem!(consultation, problem, action, addendum: nil)
      ConsultationProblem.create!(consultation: consultation, addendum: addendum, patient_problem: problem, action: action,
                                  terminology: problem.terminology, code: problem.code,
                                  terminology_release_id: problem.terminology_release_id, status_after: problem.status,
                                  onset_on: problem.onset_on, onset_precision: problem.onset_precision,
                                  resolved_on: problem.resolved_on)
    end

    def self.normalize(params)
      params = params.to_unsafe_h if params.respond_to?(:to_unsafe_h)
      params.is_a?(Hash) ? params.deep_stringify_keys.slice(*OUTCOME_KEYS) : {}
    end
    private_class_method :requirements, :materialize!, :normalize
  end
end
```

(`record_problem!` fica público: o adendo — Task 11 — grava a linha do problema com `addendum:`.)

```ruby
# app/services/consultations/effective.rb
# O que vale da consulta depois dos adendos (ADR 0031; Desvio 6): a última
# linha de cada problema; condutas acrescentadas menos as removidas; exames
# pedidos menos os cancelados — na ordem em que entraram. É o que a ficha leva.
module Consultations
  module Effective
    module_function

    def call(consultation)
      problems = consultation.problem_items.order(:created_at, :id).to_a.group_by(&:patient_problem_id).values.map(&:last)
      conducts = consultation.conducts.order(:created_at, :id).each_with_object([]) do |row, acc|
        row.action == "add" ? (acc << row.code unless acc.include?(row.code)) : acc.delete(row.code)
      end
      exams = consultation.exam_requests.order(:created_at, :id).each_with_object({}) do |row, acc|
        row.status == "requested" ? acc[row.sigtap_code] = row : acc.delete(row.sigtap_code)
      end
      { problems: problems, conducts: conducts, exam_requests: exams.values }
    end
  end
end
```

Em `app/commands/attendances/close.rb`, logo antes de `next Result.fail(:already_closed) unless attendance.open?`:

```ruby
        # ADR 0031 (Desvio 13): com a consulta em rascunho, o desfecho sai pela
        # finalização (Consultations::Finalize vira a consulta antes de chamar aqui).
        next Result.fail(:consultation_in_progress) if Consultation.exists?(attendance_id: attendance.id, status: "draft")
```

Em `app/controllers/attendances_controller.rb`, `ERROR_STATUS` ganha `consultation_in_progress: :conflict`.

- [ ] **Step 4: Rode e veja passar (e o desfecho de sempre)**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/commands/consultations spec/commands/attendances spec/requests/attendances_spec.rb spec/commands/screenings`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add app/commands/consultations/finalize.rb app/services/consultations/effective.rb app/commands/attendances/close.rb app/controllers/attendances_controller.rb spec/commands/consultations/finalize_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: finalize a consultation with its problems and the attendance outcome in one transaction

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 11: Leitura (`ClinicalRecord::Access`), abertura justificada (`ClinicalRecord::Open`) e adendo (`Consultations::AddAddendum`)

**Files:**
- Create: `app/services/clinical_record/access.rb`, `app/commands/clinical_record/open.rb`, `app/commands/consultations/add_addendum.rb`
- Modify: `app/services/consultations/authorization.rb` (`any_allowed_link`)
- Test: `spec/services/clinical_record/access_spec.rb`, `spec/commands/clinical_record/open_spec.rb`, `spec/commands/consultations/add_addendum_spec.rb`

**Interfaces:**
- Consumes: `Consultations::{Authorization,ItemsInput,Effective,Finalize.record_problem!}`, `Patients::ApplyProblemEvent`, `ClinicalRecordOpening` (Tasks 6–10).
- Produces:
  - `ClinicalRecord::Access::Grant = Data(:kind, :opening, :reason)` com `#allowed?`; `ClinicalRecord::Access.call(user:, patient:, attendance: nil, now: Time.current) -> Grant` — `kind` `:in_context` | `:justified` | `:denied` (`reason` `:missing_role` | `:out_of_context`). Contexto: atendimento `in_care` chamado pelo usuário, ou `waiting` em unidade onde ele tem vínculo ativo com CBO permitido; atendimento de par validado do mesmo CPF do paciente (sem `attendance`, procura entre os atendimentos abertos desses pares);
  - `Consultations::Authorization.any_allowed_link(user:) -> [Symbol, ProfessionalLink|nil]`;
  - `ClinicalRecord::Open.call(user:, cpf:, reason_code:, reason_note: nil) -> Result` (ok `{ opening: }`; falhas `:missing_role`, `:missing_link`, `:cbo_not_allowed`, `:invalid_reason`, `:patient_not_found`); evento `clinical_record.opened { opening_id, patient_id, user_id, reason_code }`;
  - `Consultations::AddAddendum::CHANGE_KEYS == %w[evaluated_problems conducts exam_requests]` (contrato §9: `evaluated_problems` = eventos novos; `conducts` e `exam_requests` = listas finais, guardadas em `changes` só quando mudam), `.call(consultation:, by:, reason:, text:, changes: nil, opening_id: nil) -> Result` (ok `{ addendum:, structured: bool }`; falhas `:invalid_reason`, `:text_required`, `:text_too_long` (`field: "text"`), `:invalid_changes`, `:not_finalized`, `:missing_role`, `:opening_required`, `:no_conduct` e as de `ItemsInput` (`:invalid_conduct`, `:invalid_exam`, `:invalid_problem`… com `index`)); `opening_id` aceito também do autor (guardado se válido, ignorado se não); evento `consultation.addendum_added { consultation_id, addendum_id }`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/clinical_record/access_spec.rb
require "rails_helper"

# ADR 0031 (spec §5): em contexto, abertura justificada ou nada. A recepção
# nunca lê.
RSpec.describe ClinicalRecord::Access do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = clinical_city!; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:citizen) { verified_citizen!(1) }
  let(:patient) { Patients::Resolve.call(citizen).payload[:patient] }

  def access(user, attendance: nil) = described_class.call(user: user, patient: patient, attendance: attendance)

  it "in_care chamado pelo usuário: em contexto; outro profissional: fora" do
    attendance = consulting_attendance!(unit, citizen: citizen, doctor: doctor)
    expect(access(doctor).kind).to eq(:in_context)
    expect(access(doctor, attendance: attendance).kind).to eq(:in_context)
    expect(access(doctor!(unit, cbo: "223505")).then { |g| [ g.kind, g.reason ] }).to eq([ :denied, :out_of_context ])
  end

  it "waiting: quem tem vínculo permitido na unidade lê; técnico e outra unidade não" do
    walk_in_attendance!(unit, citizen: citizen)
    expect(access(doctor!(unit, cbo: "223505")).kind).to eq(:in_context)
    expect(access(doctor!(unit, cbo: "322205")).kind).to eq(:denied)
    expect(access(doctor!(create_unit("UBS Outra"))).kind).to eq(:denied)
  end

  it "atendimento fechado não é contexto; atendimento de outro CPF também não" do
    attendance = consulting_attendance!(unit, citizen: citizen, doctor: doctor)
    other = consulting_attendance!(unit, citizen: verified_citizen!(2), doctor: doctor)
    expect(access(doctor, attendance: other).kind).to eq(:denied)
    attendance.update!(status: "closed", outcome: "discharged", closed_by_user: doctor, closed_at: Time.current)
    expect(access(doctor).kind).to eq(:denied)
  end

  it "abertura válida do próprio usuário: justificada, por 30 minutos" do
    nurse = doctor!(unit, cbo: "223505")
    now = Time.current
    opening = ClinicalRecordOpening.create!(patient: patient, user: nurse, reason_code: "case_review", created_at: now,
                                            expires_at: now + 30.minutes)
    grant = access(nurse)
    expect([ grant.kind, grant.opening ]).to eq([ :justified, opening ])
    expect(access(doctor).kind).to eq(:denied)
    travel_to(now + 30.minutes) { expect(access(nurse).kind).to eq(:denied) }
  end

  it "recepção e papel ausente: missing_role" do
    expect(access(reception!).then { |g| [ g.kind, g.reason ] }).to eq([ :denied, :missing_role ])
  end
end
```

```ruby
# spec/commands/clinical_record/open_spec.rb
require "rails_helper"

# ADR 0031 (spec §5; contratos §3): abertura por CPF, motivo de lista (nota
# de 10+ com other), válida 30 minutos; trilha só com ids e o código do motivo.
RSpec.describe ClinicalRecord::Open do
  before { Current.city = clinical_city!; ciap2_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:nurse) { doctor!(unit, cbo: "223505") }
  let(:citizen) { verified_citizen!(1) }
  let!(:patient) { Patients::Resolve.call(citizen).payload[:patient] }

  def open(**args) = described_class.call(user: nurse, cpf: CitizenIdentity::Cpf.mask(citizen.cpf), reason_code: "case_review", **args)

  it "abre por 30 minutos (CPF com máscara); evento sem a nota" do
    result = open(reason_code: "other", reason_note: "pedido da coordenação MARCADOR-NOTA")
    opening = result.payload[:opening]
    expect(opening).to have_attributes(patient_id: patient.id, user_id: nurse.id, reason_code: "other")
    expect(opening.expires_at - opening.created_at).to eq(30.minutes)
    expect(DomainEvent.where(name: "clinical_record.opened").sole.payload)
      .to eq("opening_id" => opening.id, "patient_id" => patient.id, "user_id" => nurse.id, "reason_code" => "other")
    expect(DomainEvent.pluck(:payload).to_json).not_to include("MARCADOR-NOTA")
  end

  it "nota só com other (e descartada nos demais motivos)" do
    expect(open(reason_note: "não precisava").payload[:opening].reason_note).to be_nil
    { { reason_code: "other" } => :invalid_reason, { reason_code: "other", reason_note: "curta" } => :invalid_reason,
      { reason_code: "other", reason_note: "x" * 501 } => :invalid_reason, { reason_code: "curiosidade" } => :invalid_reason,
      { reason_code: [ "case_review" ] } => :invalid_reason }.each do |args, reason|
      expect(open(**args).reason).to eq(reason), args.inspect
    end
  end

  it "CPF sem paciente (só declarado ou inexistente) → patient_not_found" do
    declared = screening_citizen!(2)
    expect(described_class.call(user: nurse, cpf: declared.cpf, reason_code: "active_search").reason).to eq(:patient_not_found)
    expect(described_class.call(user: nurse, cpf: "lixo", reason_code: "active_search").reason).to eq(:patient_not_found)
  end

  it "recepção, sem vínculo e técnico não abrem" do
    expect(described_class.call(user: reception!, cpf: citizen.cpf, reason_code: "case_review").reason).to eq(:missing_role)
    loose = staff_with("solto-#{SecureRandom.hex(3)}@x.gov.br", "health_professional")
    expect(described_class.call(user: loose, cpf: citizen.cpf, reason_code: "case_review").reason).to eq(:missing_link)
    expect(described_class.call(user: doctor!(unit, cbo: "322205"), cpf: citizen.cpf, reason_code: "case_review").reason)
      .to eq(:cbo_not_allowed)
  end
end
```

```ruby
# spec/commands/consultations/add_addendum_spec.rb
require "rails_helper"

# ADR 0031 (spec §4): adendo só em consulta finalizada, com motivo de 10+,
# texto cifrado; de terceiro, só com abertura válida dele para o paciente;
# pode mudar problemas, condutas e exames (eventos com o adendo).
# Review Focus 5: abertura de outro paciente/usuário ou vencida → opening_required.
RSpec.describe Consultations::AddAddendum do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def add(by: doctor, reason: "correção do pedido de exame", text: "Texto do adendo", **args)
    described_class.call(consultation: consultation, by: by, reason: reason, text: text, **args)
  end

  def opening_for(user, patient: consultation.patient, at: Time.current)
    ClinicalRecordOpening.create!(patient: patient, user: user, reason_code: "case_review", created_at: at, expires_at: at + 30.minutes)
  end

  it "o autor acrescenta texto e motivo; evento só com ids; sem mudança estruturada" do
    result = add(text: "MARCADOR-ADENDO")
    addendum = result.payload[:addendum]
    expect([ addendum.text, addendum.reason, addendum.changes, result.payload[:structured] ])
      .to eq([ "MARCADOR-ADENDO", "correção do pedido de exame", {}, false ])
    expect(DomainEvent.where(name: "consultation.addendum_added").sole.payload)
      .to eq("consultation_id" => consultation.id, "addendum_id" => addendum.id)
    expect(DomainEvent.pluck(:payload).to_json).not_to include("MARCADOR")
  end

  it "recusas de entrada e de estado" do
    { { reason: "curto" } => :invalid_reason, { reason: [ "x" * 20 ] } => :invalid_reason, { text: " " } => :text_required,
      { text: "x" * 20_001 } => :text_too_long, { changes: { "apagar" => [] } } => :invalid_changes,
      { changes: "lixo" } => :invalid_changes }.each do |args, reason|
      expect(add(**args).reason).to eq(reason), args.inspect
    end
    draft = started_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2))
    expect(described_class.call(consultation: draft, by: doctor, reason: "motivo suficiente", text: "x").reason).to eq(:not_finalized)
  end

  it "terceiro só com abertura válida DELE para ESTE paciente (Review Focus 5)" do
    nurse = doctor!(unit, cbo: "223505")
    expect(add(by: nurse).reason).to eq(:opening_required)
    other_patient = Patients::Resolve.call(verified_citizen!(3)).payload[:patient]
    expect(add(by: nurse, opening_id: opening_for(nurse, patient: other_patient).id).reason).to eq(:opening_required)
    expect(add(by: nurse, opening_id: opening_for(doctor).id).reason).to eq(:opening_required)
    stale = opening_for(nurse, at: 31.minutes.ago)
    expect(add(by: nurse, opening_id: stale.id).reason).to eq(:opening_required)
    valid = opening_for(nurse)
    result = add(by: nurse, opening_id: valid.id)
    expect(result.payload[:addendum]).to have_attributes(author_user_id: nurse.id, opening_id: valid.id)
    expect(add(by: reception!).reason).to eq(:missing_role)
  end

  it "o autor pode mandar a própria abertura (contrato §9): guardada se válida, ignorada se não" do
    mine = opening_for(doctor)
    expect(add(opening_id: mine.id).payload[:addendum].opening_id).to eq(mine.id)
    expect(add(opening_id: SecureRandom.uuid).payload[:addendum].opening_id).to be_nil
  end

  it "evaluated_problems são eventos novos; conducts e exam_requests são as listas finais; o efetivo reflete" do
    problem = PatientProblem.where(patient_id: consultation.patient_id).sole
    result = add(changes: { "evaluated_problems" => [ { "problem_id" => problem.id, "action" => "resolve" },
                                                      { "terminology" => "cid10", "code" => "I10", "action" => "add" } ],
                            "conducts" => [ 9 ],
                            "exam_requests" => [ { "sigtap_code" => "0202010317" } ] })
    expect(result.payload[:structured]).to be(true)
    addendum = result.payload[:addendum]
    expect(problem.reload).to have_attributes(status: "resolved", resolved_on: Time.zone.today)
    expect(problem.events.order(:created_at).last).to have_attributes(kind: "resolved", addendum_id: addendum.id)
    effective = Consultations::Effective.call(consultation.reload)
    expect(effective[:conducts]).to eq([ 9 ])
    expect(effective[:exam_requests].map(&:sigtap_code)).to eq([ "0202010317" ])
    expect(effective[:problems].map { |p| [ p.code, p.status_after ] }).to match_array([ [ "T90", "resolved" ], [ "I10", "active" ] ])
    expect(addendum.changes.keys).to match_array(%w[evaluated_problems conducts exam_requests])
    expect(addendum.changes["conducts"]).to eq([ 9 ])
  end

  it "lista final igual à atual não entra em changes; trocar a justificativa do exame recancela e repede" do
    same = add(changes: { "conducts" => [ 1 ], "exam_requests" => [ { "sigtap_code" => "0202010503" } ] })
    expect([ same.payload[:addendum].changes, same.payload[:structured] ]).to eq([ {}, false ])
    justified = add(changes: { "exam_requests" => [ { "sigtap_code" => "0202010503", "cid10_justification" => "E119" } ] })
    expect(justified.payload[:addendum].changes.keys).to eq([ "exam_requests" ])
    expect(Consultations::Effective.call(consultation.reload)[:exam_requests].sole.cid10_justification).to eq("E119")
  end

  it "lista de condutas vazia → no_conduct; chave desconhecida ou lista que não é lista → invalid_changes / 422; nada fica" do
    expect(add(changes: { "conducts" => [] }).reason).to eq(:no_conduct)
    expect(add(changes: { "conducts_added" => [ 9 ] }).reason).to eq(:invalid_changes)
    expect(add(changes: { "conducts" => "9" }).reason).to eq(:invalid_conduct)
    expect(add(changes: { "exam_requests" => [ { "sigtap_code" => "0301010064" } ] }).reason).to eq(:invalid_exam)
    expect(ConsultationAddendum.count).to eq(0)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/clinical_record spec/commands/clinical_record spec/commands/consultations/add_addendum_spec.rb`
Expected: FAIL (`uninitialized constant ClinicalRecord::Access`).

- [ ] **Step 3: Implemente**

Em `app/services/consultations/authorization.rb`, troque `any_link` por:

```ruby
    # Para a abertura e o adendo de terceiro: um vínculo permitido em qualquer unidade.
    def any_allowed_link(user:)
      return [ :missing_role, nil ] unless user&.has_role?("health_professional")

      pick(active_links(user).to_a)
    end

    def any_link(user:) = any_allowed_link(user: user).first
```

```ruby
# app/services/clinical_record/access.rb
# Quem lê o prontuário (ADR 0031; spec §5): em contexto — atendimento in_care
# chamado pelo usuário, ou waiting na unidade de um vínculo ativo dele com CBO
# permitido, de um par VALIDADO do mesmo CPF —, com abertura justificada
# válida, ou ninguém. A recepção (sem health_professional) nunca lê. Toda
# leitura permitida deixa trilha (ClinicalRecord::Trail).
module ClinicalRecord
  module Access
    Grant = Data.define(:kind, :opening, :reason) do
      def allowed? = kind != :denied
    end

    module_function

    def call(user:, patient:, attendance: nil, now: Time.current)
      return deny(:missing_role) unless user&.has_role?("health_professional")

      candidates = attendance ? [ attendance ] : open_attendances_of(patient)
      return Grant.new(kind: :in_context, opening: nil, reason: nil) if candidates.any? { |a| in_context?(user, a, patient) }

      opening = patient && ClinicalRecordOpening.valid_for(user_id: user.id, patient_id: patient.id, now: now)
                                                .order(created_at: :desc).first
      opening ? Grant.new(kind: :justified, opening: opening, reason: nil) : deny(:out_of_context)
    end

    def open_attendances_of(patient)
      return [] unless patient

      pairs = Citizen.not_erased.verification_level_verified.where(cpf: patient.cpf).select(:id)
      Attendance.open_attendances.where(citizen_id: pairs).includes(:citizen).to_a
    end

    def in_context?(user, attendance, patient)
      citizen = attendance.citizen
      return false if patient && !(citizen.verification_level_verified? && citizen.cpf == patient.cpf)

      case attendance.status
      when "in_care" then attendance.called_by_user_id == user.id
      when "waiting"
        ApplicationRecord.transaction do
          Consultations::Authorization.link_for(user: user, health_unit_id: attendance.health_unit_id).first == :ok
        end
      else false
      end
    end

    def deny(reason) = Grant.new(kind: :denied, opening: nil, reason: reason)
    private_class_method :open_attendances_of, :in_context?, :deny
  end
end
```

```ruby
# app/commands/clinical_record/open.rb
# Abertura justificada (ADR 0031; spec §5; contratos §3): profissional com
# vínculo permitido, motivo de lista (nota de 10–500 só com other), válida 30
# minutos para aquele usuário e paciente. O step-up é do controller. CPF que
# não tem paciente (nunca consultou, ou só par declarado) = patient_not_found.
module ClinicalRecord
  module Open
    module_function

    def call(user:, cpf:, reason_code:, reason_note: nil)
      status, _link = Consultations::Authorization.any_allowed_link(user: user)
      return Result.fail(status) unless status == :ok

      code = reason_code.is_a?(String) ? reason_code : nil
      return Result.fail(:invalid_reason) unless ClinicalRecordOpening::REASONS.include?(code)

      note = nil
      if code == "other"
        note = reason_note.is_a?(String) ? reason_note.strip : ""
        return Result.fail(:invalid_reason) unless note.length.between?(ClinicalRecordOpening::MIN_NOTE, ClinicalRecordOpening::MAX_NOTE)
      end

      digits = CitizenIdentity::Cpf.normalize(cpf)
      patient = digits && Patient.find_by(cpf: digits)
      return Result.fail(:patient_not_found) unless patient

      now = Time.current
      opening = ClinicalRecordOpening.create!(patient: patient, user: user, reason_code: code, reason_note: note,
                                              created_at: now, expires_at: now + ClinicalRecordOpening::VALIDITY)
      DomainEvents.publish("clinical_record.opened", opening_id: opening.id, patient_id: patient.id, user_id: user.id,
                                                     reason_code: code)
      Result.ok(opening: opening)
    end
  end
end
```

```ruby
# app/commands/consultations/add_addendum.rb
# Adendo de consulta finalizada (ADR 0031; spec §4; contrato §9; Desvio 6): só
# acréscimo, motivo 10–500, texto cifrado (1–20.000). O autor sempre (a
# abertura dele, se mandada e válida, fica registrada); outro profissional só
# com abertura justificada válida DELE para ESTE paciente. `changes`:
# `evaluated_problems` = eventos novos de problema; `conducts` e
# `exam_requests` = listas FINAIS (só entram em `changes` quando mudam). O
# banco recebe as diferenças como linhas novas (add/remove; requested/
# cancelled); a conduta efetiva nunca fica vazia. Tudo num savepoint.
module Consultations
  class AddAddendum
    CHANGE_KEYS = %w[evaluated_problems conducts exam_requests].freeze

    def self.call(consultation:, by:, reason:, text:, changes: nil, opening_id: nil)
      reason = reason.is_a?(String) ? reason.strip : ""
      unless reason.length.between?(ConsultationAddendum::MIN_REASON, ConsultationAddendum::MAX_REASON)
        return Result.fail(:invalid_reason)
      end
      return Result.fail(:text_required) unless text.is_a?(String) && text.strip.present?
      return Result.fail(:text_too_long, details: { field: "text" }) if text.length > Consultation::MAX_TEXT

      changes = normalize(changes)
      return Result.fail(:invalid_changes) unless changes

      result = nil
      ApplicationRecord.transaction(requires_new: true) do
        consultation.lock!
        next result = Result.fail(:not_finalized) unless consultation.finalized?

        patient = Patient.lock.find(consultation.patient_id)
        authorized = authorize(consultation, patient, by, opening_id)
        next result = authorized if authorized.failure?

        cbo, opening = authorized.payload.values_at(:cbo, :opening)
        plan = prepare(consultation, patient, cbo, changes)
        next result = plan if plan.failure?

        addendum = ConsultationAddendum.create!(consultation: consultation, author_user: by, text: text, reason: reason,
                                                changes: plan.payload[:stored], opening: opening)
        failure = apply!(consultation, patient, addendum, plan.payload, by)
        if failure
          result = failure
          raise ActiveRecord::Rollback
        end
        DomainEvents.publish("consultation.addendum_added", consultation_id: consultation.id, addendum_id: addendum.id)
        result = Result.ok(addendum: addendum, structured: plan.payload[:stored].any?)
      end
      result
    end

    def self.normalize(changes)
      changes = changes.to_unsafe_h if changes.respond_to?(:to_unsafe_h)
      return {} if changes.nil?
      return nil unless changes.is_a?(Hash)

      changes = changes.deep_stringify_keys
      (changes.keys - CHANGE_KEYS).empty? ? changes : nil
    end

    def self.authorize(consultation, patient, by, opening_id)
      return Result.fail(:missing_role) unless by&.has_role?("health_professional")

      opening = opening_id.is_a?(String) && ClinicalRecordOpening.valid_for(user_id: by.id, patient_id: patient.id).find_by(id: opening_id)
      return Result.ok(cbo: consultation.cbo_code, opening: opening || nil) if consultation.author_user_id == by.id
      return Result.fail(:opening_required) unless opening

      status, link = Authorization.any_allowed_link(user: by)
      status == :ok ? Result.ok(cbo: link.cbo_code, opening: opening) : Result.fail(status)
    end

    # Valida tudo antes de gravar; devolve as diferenças a gravar e o que fica em `changes`.
    def self.prepare(consultation, patient, cbo, changes)
      today = Time.zone.today
      effective = Effective.call(consultation)
      stored = {}
      plan = { problems: [], conducts_added: [], conducts_removed: [], exams_added: [], exams_removed: [] }

      if changes.key?("evaluated_problems")
        problems = ItemsInput.problems(changes["evaluated_problems"], patient: patient, cbo: cbo, on: today)
        return problems if problems.failure?

        plan[:problems] = problems.payload[:items]
        stored["evaluated_problems"] = plan[:problems] if plan[:problems].any?
      end

      if changes.key?("conducts")
        conducts = ItemsInput.conducts(changes["conducts"])
        return conducts if conducts.failure?

        final = conducts.payload[:items]
        return Result.fail(:no_conduct) if final.empty?

        plan[:conducts_added] = final - effective[:conducts]
        plan[:conducts_removed] = effective[:conducts] - final
        stored["conducts"] = final if (plan[:conducts_added] + plan[:conducts_removed]).any?
      end

      if changes.key?("exam_requests")
        exams = ItemsInput.exams(changes["exam_requests"], cbo: cbo, on: today)
        return exams if exams.failure?

        final = exams.payload[:items]
        current = effective[:exam_requests].to_h { |row| [ row.sigtap_code, row.cid10_justification ] }
        wanted = final.to_h { |e| [ e["sigtap_code"], e["cid10_justification"] ] }
        plan[:exams_removed] = current.keys.select { |code| !wanted.key?(code) || wanted[code] != current[code] }
        plan[:exams_added] = final.select { |e| !current.key?(e["sigtap_code"]) || plan[:exams_removed].include?(e["sigtap_code"]) }
        stored["exam_requests"] = final if (plan[:exams_added] + plan[:exams_removed]).any?
      end

      Result.ok(plan.merge(stored: stored))
    end

    # Devolve nil, ou o Result de falha (problema mudou desde a validação).
    def self.apply!(consultation, patient, addendum, plan, by)
      plan[:problems].each_with_index do |item, index|
        applied = Patients::ApplyProblemEvent.call(
          patient: patient, action: item["action"], by: by, source: { addendum: addendum },
          terminology: item["terminology"], code: item["code"], release_id: item["release_id"],
          problem: item["problem_id"] && PatientProblem.find_by(id: item["problem_id"]),
          onset_on: item["onset_on"] && Date.iso8601(item["onset_on"]), onset_precision: item["onset_precision"]
        )
        return Result.fail(applied.reason, details: { index: index }) if applied.failure?

        Finalize.record_problem!(consultation, applied.payload[:problem], item["action"], addendum: addendum)
      end
      plan[:conducts_added].each { |code| ConsultationConduct.create!(consultation: consultation, addendum: addendum, code: code, action: "add") }
      plan[:conducts_removed].each { |code| ConsultationConduct.create!(consultation: consultation, addendum: addendum, code: code, action: "remove") }
      plan[:exams_removed].each do |code|
        previous = consultation.exam_requests.where(sigtap_code: code, status: "requested").order(:created_at, :id).last
        ConsultationExamRequest.create!(consultation: consultation, addendum: addendum, sigtap_code: code,
                                        sigtap_competence: previous.sigtap_competence, status: "cancelled")
      end
      plan[:exams_added].each do |exam|
        ConsultationExamRequest.create!(consultation: consultation, addendum: addendum, sigtap_code: exam["sigtap_code"],
                                        sigtap_competence: exam["sigtap_competence"],
                                        cid10_justification: exam["cid10_justification"], status: "requested")
      end
      nil
    end
    private_class_method :normalize, :authorize, :prepare, :apply!
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/clinical_record spec/commands/clinical_record spec/commands/consultations spec/services/consultations`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add app/services/clinical_record/access.rb app/commands/clinical_record/open.rb app/commands/consultations/add_addendum.rb app/services/consultations/authorization.rb spec/services/clinical_record/access_spec.rb spec/commands/clinical_record/open_spec.rb spec/commands/consultations/add_addendum_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: read the clinical record in context or by justified opening, and add addenda

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 12: Formas do contrato (`Consultations::Json`, `ClinicalRecord::Json`) e a trilha de leitura

**Files:**
- Create: `app/services/consultations/json.rb`, `app/services/clinical_record/json.rb`, `app/services/clinical_record/trail.rb`
- Test: `spec/services/consultations/json_spec.rb`, `spec/services/clinical_record/json_spec.rb`

**Interfaces:**
- Consumes: `Screenings::Json.{staff_name,screening}`, `Screenings::VitalSigns.{json,bmi}` (18), `ClinicalTerms`, `ClinicalTerms::SigtapExams`, `Ledi::ConsultationMapping`, `Professionals::Cbo.find`, `ClinicalRecord::Access::Grant`.
- Produces (contratos §1, §3, §4; formatos exatos lá):
  - `Consultations::Json.consultation(consultation) -> Hash` (`<consultation>`), `.addendum(addendum) -> Hash`, `.summary(consultation) -> Hash` (item de `record.consultations`), `.evaluated_problems(consultation) -> Array<Hash>`;
  - `ClinicalRecord::Json.record(patient:, citizen:, grant:, screening: nil) -> Hash` (`<record>`; `patient` nil → bloco do par com `id: nil`, listas vazias), `.problem(patient_problem) -> Hash` (`<problema>`), `.patient_block(patient, citizen) -> Hash`;
  - `ClinicalRecord::Trail.viewed!(patient:, user:, grant:)` (publica `clinical_record.viewed { patient_id, user_id, access, reason_code }`; nada sem paciente).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/consultations/json_spec.rb
require "rails_helper"

# Contratos §4: a forma <consultation> no rascunho (itens de draft_items, com
# rótulo) e finalizada (itens gravados, adendos em ordem); opcionais só com valor.
RSpec.describe Consultations::Json do
  before { Current.city = clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:keys) do
    %w[id attendance_id patient_id status author cbo_code subjective objective assessment plan vitals care_type
       evaluated_problems conducts exam_requests started_at finalized_at addenda]
  end

  it "rascunho: chaves do contrato, itens do rascunho com rótulo, sinais com IMC" do
    consultation = started_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    Consultations::SaveDraft.call(consultation: consultation, params: draft_body, by: doctor)
    json = described_class.consultation(consultation.reload).deep_stringify_keys
    expect(json.keys).to match_array(keys)
    expect(json["author"]).to eq("id" => doctor.id, "name" => doctor.professional.professional_name)
    expect(json["vitals"]).to eq("systolic" => 130, "diastolic" => 85, "weight_kg" => 82.5, "height_cm" => 170, "bmi" => 28.5)
    expect(json["evaluated_problems"]).to eq([ { "problem_id" => nil, "terminology" => "ciap2", "code" => "T90",
                                                 "label" => "Diabetes não insulino-dependente", "action" => "add",
                                                 "onset_on" => "2025-08-01", "onset_precision" => "month" } ])
    expect(json["exam_requests"]).to eq([ { "sigtap_code" => "0202010503", "label" => "DOSAGEM DE HEMOGLOBINA GLICOSILADA" } ])
    expect(json.values_at("status", "conducts", "finalized_at", "addenda")).to eq([ "draft", [ 1 ], nil, [] ])
  end

  it "finalizada: itens gravados (sem os do adendo), adendos em ordem com autor e mudanças" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "exame adicional pedido",
                                    text: "Pedido creatinina", changes: { "exam_requests" => [ { "sigtap_code" => "0202010503" }, { "sigtap_code" => "0202010317" } ] })
    json = described_class.consultation(consultation.reload).deep_stringify_keys
    expect(json["evaluated_problems"].sole).to include("problem_id" => PatientProblem.sole.id, "code" => "T90", "action" => "add")
    expect(json["exam_requests"].map { |e| e["sigtap_code"] }).to eq([ "0202010503" ])
    addendum = json["addenda"].sole
    expect(addendum.keys).to match_array(%w[id author_name created_at reason text changes])
    expect(addendum.values_at("author_name", "reason", "text")).to eq([ doctor.professional.professional_name, "exame adicional pedido", "Pedido creatinina" ])
    expect(addendum["changes"]["exam_requests"].map { |e| e["sigtap_code"] }).to eq(%w[0202010503 0202010317])
    expect(described_class.summary(consultation).deep_stringify_keys)
      .to include("id" => consultation.id, "author_name" => doctor.professional.professional_name,
                  "care_type_label" => "Consulta no dia", "addenda_count" => 1)
  end
end
```

```ruby
# spec/services/clinical_record/json_spec.rb
require "rails_helper"

# Contratos §3: <record> com o paciente (nome de exibição; idade e sexo),
# problemas, escuta do dia e consultas; trilha só com ids, acesso e motivo.
RSpec.describe ClinicalRecord::Json do
  before { Current.city = clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:grant) { ClinicalRecord::Access::Grant.new(kind: :in_context, opening: nil, reason: nil) }

  it "paciente com consulta: bloco do paciente, problemas, consultas" do
    citizen = verified_citizen!(1, social_name: "Mariana", age: 46)
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    json = described_class.record(patient: consultation.patient, citizen: citizen, grant: grant).deep_stringify_keys
    expect(json.keys).to match_array(%w[patient access problems today_screening consultations])
    expect(json["patient"]).to eq("id" => consultation.patient_id, "display_name" => "Mariana",
                                  "full_name" => "Maria Aparecida da Silva", "social_name" => "Mariana",
                                  "age" => 46, "sex" => "female", "cpf_masked" => citizen.cpf_masked)
    expect(json["access"]).to eq("in_context")
    expect(json["problems"].sole).to eq("id" => PatientProblem.sole.id, "terminology" => "ciap2", "code" => "T90",
                                        "label" => "Diabetes não insulino-dependente", "status" => "active",
                                        "onset_on" => "2025-08-01", "onset_precision" => "month", "resolved_on" => nil)
    expect(json["consultations"].sole["id"]).to eq(consultation.id)
    expect(json["today_screening"]).to be_nil
  end

  it "par validado sem paciente ainda: id nulo, listas vazias" do
    citizen = verified_citizen!(2)
    json = described_class.record(patient: nil, citizen: citizen, grant: grant).deep_stringify_keys
    expect(json["patient"]).to include("id" => nil, "display_name" => "Maria Aparecida da Silva")
    expect(json.values_at("problems", "consultations")).to eq([ [], [] ])
  end

  it "trilha: só ids, acesso e motivo; nada sem paciente" do
    patient = Patients::Resolve.call(verified_citizen!(3)).payload[:patient]
    nurse = doctor!(unit, cbo: "223505")
    now = Time.current
    opening = ClinicalRecordOpening.create!(patient: patient, user: nurse, reason_code: "other", reason_note: "nota MARCADOR",
                                            created_at: now, expires_at: now + 30.minutes)
    ClinicalRecord::Trail.viewed!(patient: patient, user: nurse,
                                  grant: ClinicalRecord::Access::Grant.new(kind: :justified, opening: opening, reason: nil))
    ClinicalRecord::Trail.viewed!(patient: nil, user: nurse, grant: grant)
    expect(DomainEvent.where(name: "clinical_record.viewed").pluck(:payload))
      .to eq([ { "patient_id" => patient.id, "user_id" => nurse.id, "access" => "justified", "reason_code" => "other" } ])
  end
end
```

(O rótulo de T90 é o de `ScreeningHelpers::CIAP2`; se o helper do 18 tiver outro texto, use o dele.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/consultations/json_spec.rb spec/services/clinical_record/json_spec.rb`
Expected: FAIL (`uninitialized constant Consultations::Json`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/consultations/json.rb
# A forma <consultation> (contratos §4). Rascunho: itens de draft_items;
# finalizada: os itens gravados na finalização (os dos adendos ficam em
# `addenda[].changes`). Rótulos da release gravada; sinais como no módulo 18
# (número como número, IMC calculado). Opcionais só aparecem com valor.
module Consultations
  module Json
    module_function

    def consultation(c)
      { id: c.id, attendance_id: c.attendance_id, patient_id: c.patient_id, status: c.status,
        author: { id: c.author_user_id, name: Screenings::Json.staff_name(c.author_user) }, cbo_code: c.cbo_code,
        subjective: c.subjective, objective: c.objective, assessment: c.assessment, plan: c.plan, vitals: vitals(c),
        care_type: c.care_type, evaluated_problems: evaluated_problems(c), conducts: conducts(c),
        exam_requests: exam_requests(c), started_at: c.started_at.iso8601, finalized_at: c.finalized_at&.iso8601,
        addenda: c.addenda.order(:created_at, :id).map { |a| addendum(a) } }
    end

    def addendum(a)
      { id: a.id, author_name: Screenings::Json.staff_name(a.author_user), created_at: a.created_at.iso8601,
        reason: a.reason, text: a.text, changes: a.changes }
    end

    def summary(c)
      { id: c.id, finalized_at: c.finalized_at&.iso8601, author_name: Screenings::Json.staff_name(c.author_user),
        cbo_label: Professionals::Cbo.find(c.cbo_code)&.title, care_type_label: Ledi::ConsultationMapping.care_type_label(c.care_type),
        problems: evaluated_problems(c), addenda_count: c.addenda.size }
    end

    def evaluated_problems(c)
      if c.draft?
        Array(c.draft_items["evaluated_problems"]).map do |i|
          problem_item(i["problem_id"], i["terminology"], i["code"], i["release_id"], i["action"], i["onset_on"], i["onset_precision"])
        end
      else
        c.problem_items.where(addendum_id: nil).order(:created_at, :id).map do |row|
          problem_item(row.patient_problem_id, row.terminology, row.code, row.terminology_release_id, row.action,
                       row.onset_on&.iso8601, row.onset_precision)
        end
      end
    end

    def vitals(c) = Screenings::VitalSigns.json(c.vitals).merge("bmi" => Screenings::VitalSigns.bmi(c.vitals)).compact

    def conducts(c) = c.draft? ? Array(c.draft_items["conducts"]) : c.conducts.where(addendum_id: nil).order(:created_at, :id).pluck(:code)

    def exam_requests(c)
      rows = if c.draft?
               Array(c.draft_items["exam_requests"]).map { |e| e.values_at("sigtap_code", "sigtap_competence", "cid10_justification") }
             else
               c.exam_requests.where(addendum_id: nil).order(:created_at, :id).pluck(:sigtap_code, :sigtap_competence, :cid10_justification)
             end
      rows.map do |code, competence, cid|
        { sigtap_code: code, label: ClinicalTerms::SigtapExams.label(code, competence), cid10_justification: cid }.compact
      end
    end

    def problem_item(problem_id, terminology, code, release_id, action, onset_on, onset_precision)
      { problem_id: problem_id, terminology: terminology, code: code, label: ClinicalTerms.label(terminology, code, release_id),
        action: action, onset_on: onset_on, onset_precision: onset_precision }.reject { |k, v| v.nil? && %i[onset_on onset_precision].include?(k) }
    end
    private_class_method :vitals, :conducts, :exam_requests, :problem_item
  end
end
```

```ruby
# app/services/clinical_record/json.rb
# A forma <record> (contratos §3). Sem paciente ainda (par validado que nunca
# consultou), o bloco vem do par com id nulo e as listas vazias (Desvio 10).
# Problemas: ativos primeiro; consultas: as 20 finalizadas mais recentes.
module ClinicalRecord
  module Json
    CONSULTATIONS_LIMIT = 20

    module_function

    def record(patient:, citizen:, grant:, screening: nil)
      { patient: patient_block(patient, citizen), access: grant.kind.to_s,
        problems: patient ? patient.problems.order(Arel.sql("status = 'active' DESC"), :code).map { |p| problem(p) } : [],
        today_screening: screening && Screenings::Json.screening(screening),
        consultations: patient ? recent(patient).map { |c| Consultations::Json.summary(c) } : [] }
    end

    def patient_block(patient, citizen)
      source = patient || citizen
      { id: patient&.id, display_name: source.display_name, full_name: source.full_name, social_name: source.social_name,
        age: source.age, sex: source.sex, cpf_masked: source.cpf_masked }
    end

    def problem(p)
      { id: p.id, terminology: p.terminology, code: p.code, label: ClinicalTerms.label(p.terminology, p.code, p.terminology_release_id),
        status: p.status, onset_on: p.onset_on&.iso8601, onset_precision: p.onset_precision, resolved_on: p.resolved_on&.iso8601 }
    end

    def recent(patient)
      patient.consultations.finalized_consultations.includes(:author_user, :addenda).order(finalized_at: :desc, id: :desc)
             .limit(CONSULTATIONS_LIMIT)
    end
    private_class_method :recent
  end
end
```

```ruby
# app/services/clinical_record/trail.rb
# Toda leitura de prontuário deixa trilha (ADR 0031, Invariantes): ids, o
# acesso (in_context | justified) e o código do motivo — nunca a nota.
module ClinicalRecord
  module Trail
    module_function

    def viewed!(patient:, user:, grant:)
      return unless patient

      DomainEvents.publish("clinical_record.viewed", patient_id: patient.id, user_id: user.id, access: grant.kind.to_s,
                                                     reason_code: grant.opening&.reason_code)
    end
  end
end
```

(`Citizen#age` e `Patient#age` existem; `Citizen#cpf_masked` e `Patient#cpf_masked` também — o bloco serve aos dois.)

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/consultations spec/services/clinical_record`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add app/services/consultations/json.rb app/services/clinical_record/json.rb app/services/clinical_record/trail.rb spec/services/consultations/json_spec.rb spec/services/clinical_record/json_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: shape the consultation and clinical record payloads and their read trail

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: API — consulta, prontuário, aberturas, relatório e buscas de terminologia

**Files:**
- Create: `app/controllers/consultations_controller.rb`, `app/controllers/clinical_records_controller.rb`, `app/controllers/clinical_record_openings_controller.rb`, `app/controllers/sigtap_procedures_controller.rb`
- Modify: `app/controllers/ciap2_codes_controller.rb`, `config/routes.rb`
- Test: `spec/requests/consultations_spec.rb`, `spec/requests/clinical_record_spec.rb`, `spec/requests/clinical_terms_search_spec.rb`

**Interfaces:**
- Consumes: Tasks 2, 8–12.
- Produces (contratos §3–§5; formatos exatos lá):
  - `GET /attendance/consultation_options` → `{ care_types: [{ code, label }], conducts: [{ code, label }], cid10_allowed_for_cbo }`;
  - `POST /attendance/attendances/:id/consultation` → 201 `<consultation>`; `GET|PATCH /attendance/consultations/:id` → `<consultation>`; `POST /attendance/consultations/:id/finalize { outcome: {…} }` → `<consultation>`; `POST /attendance/consultations/:id/addenda { reason, text, changes?, opening_id? }` → 201 `<addendum>`;
  - `GET /attendance/attendances/:id/record` → `<record>`; `GET /clinical_record/patients/:id` → `<record>`; `POST /clinical_record/openings` (step-up) → 201 `{ opening_id, patient_id, expires_at }`; `GET /clinical_record/openings?from=&to=&user_id=` (`municipal_admin`) → `{ items: [{ id, user_name, cpf_masked, reason_code, created_at, expires_at }] }`;
  - `POST /attendance/ciap2/search { q, terminology? }` (`ciap2` padrão, `cid10`; outro valor → 422 `invalid_terminology`); `POST /attendance/sigtap/search { q }` → `{ items: [{ code, label }] }` (503 `terminology_unavailable`).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/requests/consultations_spec.rb
require "rails_helper"

# Contratos §4: a consulta em /attendance — iniciar, autosave, finalizar,
# ler (trilha), adendo. Review Focus 3: autosave nas bordas.
RSpec.describe "Consulta", type: :request do
  before { clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:citizen) { verified_citizen!(1) }
  let(:attendance) { consulting_attendance!(unit, citizen: citizen, doctor: doctor) }
  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]
  def json_patch(path, params) = patch(path, params: params.to_json, headers: { "CONTENT_TYPE" => "application/json" })

  def start!
    sign_in_as(doctor)
    json_post "/attendance/attendances/#{attendance.id}/consultation"
    body
  end

  it "opções da consulta para o usuário" do
    sign_in_as(doctor)
    get "/attendance/consultation_options"
    expect(body["care_types"].map { |t| t["code"] }).to eq([ 1, 2, 5, 6 ])
    expect(body["conducts"].first).to eq("code" => 1, "label" => "Retorno para consulta agendada")
    expect(body["cid10_allowed_for_cbo"]).to be(true)
  end

  it "inicia (201), de novo 409 com o id; par não validado 409; interruptor desligado 403" do
    id = start!["id"]
    expect(response).to have_http_status(:created)
    expect(body["status"]).to eq("draft")
    json_post "/attendance/attendances/#{attendance.id}/consultation"
    expect([ response.status, body["error"], body["consultation_id"] ]).to eq([ 409, "already_exists", id ])
    declared = consulting_attendance!(unit, citizen: screening_citizen!(5), doctor: doctor)
    json_post "/attendance/attendances/#{declared.id}/consultation"
    expect(status_and_error).to eq([ 409, "citizen_not_verified" ])
    clinical_city!(enabled: false)
    json_post "/attendance/attendances/#{declared.id}/consultation"
    expect([ response.status, body["error"], body["feature"] ]).to eq([ 403, "feature_disabled", "clinical_record" ])
  end

  it "autosave nas bordas: 422 com field, 403 de outro, 409 depois de finalizar (Review Focus 3)" do
    id = start!["id"]
    json_patch "/attendance/consultations/#{id}", draft_body
    expect(response).to have_http_status(:ok)
    expect(body["evaluated_problems"].sole["code"]).to eq("T90")
    { { "vitals" => { "systolic" => "130" } } => [ 422, "implausible_vital", "diastolic" ],
      { "subjective" => "x" * 20_001 } => [ 422, "text_too_long", "subjective" ],
      { "objective" => { "a" => 1 } } => [ 422, "invalid_text", "objective" ],
      { "conducts" => "9" } => [ 422, "invalid_conduct", nil ],
      { "evaluated_problems" => [ { "action" => "add", "terminology" => "ciap2", "code" => "XYZ" } ] } => [ 422, "invalid_problem", nil ] }
      .each do |params, (status, error, field)|
      json_patch "/attendance/consultations/#{id}", params
      expect([ response.status, body["error"], body["field"] ]).to eq([ status, error, field ]), params.inspect.truncate(80)
    end
    sign_in_as(doctor!(unit, cbo: "223505"))
    json_patch "/attendance/consultations/#{id}", "plan" => "x"
    expect(status_and_error).to eq([ 403, "not_author" ])
    sign_in_as(doctor)
    json_post "/attendance/consultations/#{id}/finalize", outcome: { outcome: "discharged" }
    expect(response).to have_http_status(:ok)
    json_patch "/attendance/consultations/#{id}", "plan" => "tarde demais"
    expect(status_and_error).to eq([ 409, "not_draft" ])
  end

  it "finaliza com o desfecho; requisitos 422; erros do close passam" do
    id = start!["id"]
    json_patch "/attendance/consultations/#{id}", draft_body(conducts: [])
    json_post "/attendance/consultations/#{id}/finalize", outcome: { outcome: "discharged" }
    expect(status_and_error).to eq([ 422, "no_conduct" ])
    json_patch "/attendance/consultations/#{id}", "conducts" => [ 1 ]
    json_post "/attendance/consultations/#{id}/finalize", outcome: { outcome: "referred" }
    expect(status_and_error).to eq([ 422, "referral_required" ])
    json_post "/attendance/consultations/#{id}/finalize", outcome: { outcome: "return" }
    expect(response).to have_http_status(:ok)
    expect(body.values_at("status", "conducts")).to eq([ "finalized", [ 1 ] ])
    expect(attendance.reload.outcome).to eq("return")
  end

  it "leitura: rascunho só do autor; finalizada em contexto ou com abertura; trilha" do
    id = start!["id"]
    nurse = doctor!(unit, cbo: "223505")
    sign_in_as(nurse)
    get "/attendance/consultations/#{id}"
    expect(status_and_error).to eq([ 403, "not_author" ])
    sign_in_as(doctor)
    json_patch "/attendance/consultations/#{id}", draft_body
    json_post "/attendance/consultations/#{id}/finalize", outcome: { outcome: "discharged" }
    sign_in_as(nurse)
    get "/attendance/consultations/#{id}"
    expect(status_and_error).to eq([ 403, "out_of_context" ])
    patient_id = Consultation.find(id).patient_id
    now = Time.current
    ClinicalRecordOpening.create!(patient_id: patient_id, user: nurse, reason_code: "case_review", created_at: now, expires_at: now + 30.minutes)
    get "/attendance/consultations/#{id}"
    expect(response).to have_http_status(:ok)
    expect(DomainEvent.where(name: "clinical_record.viewed").pluck(:payload).last)
      .to eq("patient_id" => patient_id, "user_id" => nurse.id, "access" => "justified", "reason_code" => "case_review")
  end

  it "adendo (201) do autor; de terceiro sem abertura 403; em rascunho 409" do
    id = start!["id"]
    json_post "/attendance/consultations/#{id}/addenda", reason: "acréscimo de dados", text: "texto"
    expect(status_and_error).to eq([ 409, "not_finalized" ])
    json_patch "/attendance/consultations/#{id}", draft_body
    json_post "/attendance/consultations/#{id}/finalize", outcome: { outcome: "discharged" }
    json_post "/attendance/consultations/#{id}/addenda", reason: "acréscimo de dados", text: "texto",
                                                         changes: { conducts: [ 1, 9 ] }
    expect(response).to have_http_status(:created)
    expect(body.keys).to match_array(%w[id author_name created_at reason text changes])
    sign_in_as(doctor!(unit, cbo: "223505"))
    json_post "/attendance/consultations/#{id}/addenda", reason: "acréscimo de dados", text: "texto"
    expect(status_and_error).to eq([ 403, "opening_required" ])
    json_post "/attendance/consultations/#{id}/addenda", reason: "curto", text: "texto"
    expect(status_and_error).to eq([ 422, "invalid_reason" ])
  end
end
```

```ruby
# spec/requests/clinical_record_spec.rb
require "rails_helper"

# Contratos §3: prontuário em contexto, abertura justificada (step-up, 30
# min), leitura justificada, relatório (municipal_admin). A recepção nunca
# lê: 403 em toda rota do prontuário. Review Focus 5.
RSpec.describe "Prontuário", type: :request do
  include ActiveSupport::Testing::TimeHelpers
  before { clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:citizen) { verified_citizen!(1, social_name: "Mariana") }
  let(:nurse) do
    doctor!(unit, cbo: "223505").tap do |u|
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
    end
  end
  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]
  def step_up!(user) = sign_in_as(user).update!(mfa_verified_at: Time.current)

  it "em contexto: o prontuário do atendimento chamado, com a escuta do dia e trilha" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    attendance = consulting_attendance!(unit, citizen: citizen.reload, doctor: doctor)
    sign_in_as(doctor)
    get "/attendance/attendances/#{attendance.id}/record"
    expect(response).to have_http_status(:ok)
    expect(body.dig("patient", "display_name")).to eq("Mariana")
    expect(body["access"]).to eq("in_context")
    expect(body["consultations"].sole["id"]).to eq(consultation.id)
    expect(DomainEvent.where(name: "clinical_record.viewed").pluck(:payload).last)
      .to eq("patient_id" => consultation.patient_id, "user_id" => doctor.id, "access" => "in_context", "reason_code" => nil)
    sign_in_as(doctor!(unit, cbo: "225142"))
    get "/attendance/attendances/#{attendance.id}/record"
    expect(status_and_error).to eq([ 403, "out_of_context" ])
  end

  it "par não validado → 409; par validado sem paciente → id nulo e sem trilha de prontuário" do
    declared = consulting_attendance!(unit, citizen: screening_citizen!(5), doctor: doctor)
    sign_in_as(doctor)
    get "/attendance/attendances/#{declared.id}/record"
    expect(status_and_error).to eq([ 409, "citizen_not_verified" ])
    fresh = consulting_attendance!(unit, citizen: verified_citizen!(6), doctor: doctor)
    get "/attendance/attendances/#{fresh.id}/record"
    expect(body.dig("patient", "id")).to be_nil
    expect(DomainEvent.where(name: "clinical_record.viewed").count).to eq(0)
  end

  it "abertura: step-up, motivo, 30 min; leitura justificada; vencida 403 (Review Focus 5)" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    sign_in_as(nurse)
    json_post "/clinical_record/openings", cpf: citizen.cpf, reason_code: "case_review"
    expect(status_and_error).to eq([ 401, "mfa_required" ])
    step_up!(nurse)
    get "/clinical_record/patients/#{consultation.patient_id}"
    expect(status_and_error).to eq([ 403, "opening_required" ])
    json_post "/clinical_record/openings", cpf: "123.456.789-09", reason_code: "case_review"
    expect(status_and_error).to eq([ 404, "patient_not_found" ])
    json_post "/clinical_record/openings", cpf: citizen.cpf, reason_code: "other", reason_note: "curta"
    expect(status_and_error).to eq([ 422, "invalid_reason" ])
    json_post "/clinical_record/openings", cpf: citizen.cpf, reason_code: "active_search"
    expect(response).to have_http_status(:created)
    expect(body.keys).to match_array(%w[opening_id patient_id expires_at])
    get "/clinical_record/patients/#{consultation.patient_id}"
    expect([ response.status, body["access"] ]).to eq([ 200, "justified" ])
    travel_to(30.minutes.from_now + 1.second) do
      sign_in_as(nurse)
      get "/clinical_record/patients/#{consultation.patient_id}"
      expect(status_and_error).to eq([ 403, "opening_required" ])
    end
  end

  it "relatório das aberturas: municipal_admin, filtros, CPF mascarado, nunca a nota" do
    patient = Patients::Resolve.call(citizen).payload[:patient]
    now = Time.current
    ClinicalRecordOpening.create!(patient: patient, user: nurse, reason_code: "other", reason_note: "nota MARCADOR",
                                  created_at: now, expires_at: now + 30.minutes)
    sign_in_as(staff_with("adm-rel@cidade.gov.br", "municipal_admin"))
    get "/clinical_record/openings", params: { from: Time.zone.today.iso8601, to: Time.zone.today.iso8601, user_id: nurse.id }
    item = body["items"].sole
    expect(item.keys).to match_array(%w[id user_name cpf_masked reason_code created_at expires_at])
    expect(item.values_at("user_name", "cpf_masked", "reason_code")).to eq([ nurse.professional.professional_name, patient.cpf_masked, "other" ])
    expect(response.body).not_to include("MARCADOR")
    get "/clinical_record/openings", params: { from: "ontem" }
    expect(status_and_error).to eq([ 422, "invalid_period" ])
    get "/clinical_record/openings", params: { to: (Time.zone.today - 1).iso8601 }
    expect(body["items"]).to eq([])
    sign_in_as(doctor)
    get "/clinical_record/openings"
    expect(status_and_error).to eq([ 403, "missing_role" ])
  end

  it "a recepção recebe 403 em toda rota do prontuário" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    attendance = consulting_attendance!(unit, citizen: citizen.reload, doctor: doctor)
    sign_in_as(reception!)
    [ [ :get, "/attendance/attendances/#{attendance.id}/record" ], [ :get, "/attendance/consultation_options" ],
      [ :post, "/attendance/attendances/#{attendance.id}/consultation" ], [ :get, "/attendance/consultations/#{consultation.id}" ],
      [ :patch, "/attendance/consultations/#{consultation.id}" ], [ :post, "/attendance/consultations/#{consultation.id}/finalize" ],
      [ :post, "/attendance/consultations/#{consultation.id}/addenda" ],
      [ :get, "/clinical_record/patients/#{consultation.patient_id}" ], [ :post, "/clinical_record/openings" ],
      [ :post, "/attendance/sigtap/search" ] ].each do |verb, path|
      send(verb, path, params: {}.to_json, headers: { "CONTENT_TYPE" => "application/json" })
      expect(response).to have_http_status(:forbidden), "#{verb} #{path}"
      expect(response.body).not_to include("Mariana", "Maria Aparecida", "Diabetes")
    end
  end
end
```

```ruby
# spec/requests/clinical_terms_search_spec.rb
require "rails_helper"

# Contratos §5: a busca do módulo 18 aceita CID-10; SIGTAP só exames da
# competência ativa. Termo no corpo, nunca na URL.
RSpec.describe "Busca de terminologia da consulta", type: :request do
  before { clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:doctor) { doctor!(create_unit) }
  def body = JSON.parse(response.body)

  it "CIAP-2 (padrão) e CID-10; terminologia desconhecida 422" do
    sign_in_as(doctor)
    json_post "/attendance/ciap2/search", q: "diabetes"
    expect(body["items"].map { |i| i["code"] }).to eq([ "T90" ])
    json_post "/attendance/ciap2/search", q: "diabetes", terminology: "cid10"
    expect(body).to eq("items" => [ { "code" => "E119", "label" => ClinicalRecordHelpers::CID10["E119"].first } ])
    json_post "/attendance/ciap2/search", q: "x", terminology: "loinc"
    expect([ response.status, body["error"] ]).to eq([ 422, "invalid_terminology" ])
  end

  it "SIGTAP: exames por nome ou código; sem competência ativa 503; interruptor desligado 403" do
    sign_in_as(doctor)
    json_post "/attendance/sigtap/search", q: "creatinina"
    expect(body).to eq("items" => [ { "code" => "0202010317", "label" => "DOSAGEM DE CREATININA" } ])
    allow(ClinicalTerms::SigtapExams).to receive(:release).and_return(nil)
    json_post "/attendance/sigtap/search", q: "creatinina"
    expect([ response.status, body["error"] ]).to eq([ 503, "terminology_unavailable" ])
    clinical_city!(enabled: false)
    json_post "/attendance/sigtap/search", q: "creatinina"
    expect(response).to have_http_status(:forbidden)
  end
end
```

(`ScreeningHelpers::CIAP2` traz T90 "Diabetes não insulino-dependente": a busca por "diabetes" acha só ele.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/requests/consultations_spec.rb spec/requests/clinical_record_spec.rb spec/requests/clinical_terms_search_spec.rb`
Expected: FAIL (`No route matches`).

- [ ] **Step 3: Rotas**

Em `config/routes.rb`, no `scope "/attendance"`, depois de `post "ciap2/search", …` (módulo 18):

```ruby
    # Consulta e prontuário em contexto (ADR 0031; contratos §3–§5).
    # `consultation_options` é literal: não colide com `consultations/:id`.
    get   "consultation_options",            to: "consultations#options"
    post  "attendances/:id/consultation",    to: "consultations#create"
    get   "attendances/:id/record",          to: "clinical_records#context"
    get   "consultations/:id",               to: "consultations#show"
    patch "consultations/:id",               to: "consultations#update"
    post  "consultations/:id/finalize",      to: "consultations#finalize"
    post  "consultations/:id/addenda",       to: "consultations#addenda"
    post  "sigtap/search",                   to: "sigtap_procedures#search"
```

e, depois do bloco `scope "/attendance"`:

```ruby
  # Prontuário fora de contexto (ADR 0031; contratos §3). Prefixo próprio no
  # proxy de dev do dashboard. A abertura exige step-up; o relatório é do
  # municipal_admin.
  scope "/clinical_record" do
    post "openings",     to: "clinical_record_openings#create"
    get  "openings",     to: "clinical_record_openings#index"
    get  "patients/:id", to: "clinical_records#patient"
  end
```

- [ ] **Step 4: Controllers**

```ruby
# app/controllers/consultations_controller.rb
# Consulta da APS (ADR 0031; spec §4; contratos §4). Interruptor clinical_record
# utilizável e papel health_professional em toda rota (a recepção recebe 403);
# vínculo, CBO e autoria nos comandos. Ler a consulta deixa trilha.
class ConsultationsController < ApplicationController
  include Authentication
  include AttendanceAccess
  include ClinicalRecordGate

  wrap_parameters false

  ERROR_STATUS = {
    missing_role: :forbidden, missing_link: :forbidden, cbo_not_allowed: :forbidden, not_author: :forbidden,
    opening_required: :forbidden, out_of_context: :forbidden, feature_disabled: :forbidden,
    not_in_care: :conflict, not_caller: :conflict, already_exists: :conflict, citizen_not_verified: :conflict,
    not_draft: :conflict, not_finalized: :conflict, already_closed: :conflict, consultation_in_progress: :conflict
  }.freeze
  DRAFT_KEYS = (Consultation::TEXT_FIELDS + %w[vitals care_type evaluated_problems conducts exam_requests]).freeze

  before_action :require_clinical_record!
  before_action :require_professional
  before_action :set_consultation, except: %i[options create]

  def options
    status, link = Consultations::Authorization.any_allowed_link(user: Current.user)
    render json: { care_types: Ledi::ConsultationMapping.care_types, conducts: Ledi::ConsultationMapping.conducts,
                   cid10_allowed_for_cbo: status == :ok && Ledi::ConsultationMapping.cid10_allowed?(link.cbo_code) }
  end

  def create
    attendance = Attendance.find_by(id: params[:id])
    return not_found unless attendance

    result = Consultations::Start.call(attendance: attendance, by: Current.user)
    return failure(result) if result.failure?

    render json: Consultations::Json.consultation(result.payload[:consultation]), status: :created
  end

  def show
    return forbid("not_author") if @consultation.draft? && @consultation.author_user_id != Current.user.id

    grant = read_grant
    return forbid(grant.reason.to_s) unless grant.allowed?

    ClinicalRecord::Trail.viewed!(patient: @consultation.patient, user: Current.user, grant: grant)
    render json: Consultations::Json.consultation(@consultation)
  end

  def update
    result = Consultations::SaveDraft.call(consultation: @consultation, params: body.slice(*DRAFT_KEYS), by: Current.user)
    respond(result)
  end

  def finalize
    outcome = body["outcome"].is_a?(Hash) ? body["outcome"] : {}
    respond(Consultations::Finalize.call(consultation: @consultation, outcome_params: outcome, by: Current.user))
  end

  def addenda
    result = Consultations::AddAddendum.call(consultation: @consultation, by: Current.user, reason: body["reason"],
                                             text: body["text"], changes: body["changes"], opening_id: body["opening_id"])
    return failure(result) if result.failure?

    render json: Consultations::Json.addendum(result.payload[:addendum]), status: :created
  end

  private

  def set_consultation
    @consultation = Consultation.find_by(id: params[:id])
    not_found unless @consultation
  end

  # Rascunho: só o autor chega aqui, e ele está em contexto (o rascunho só
  # existe com o atendimento in_care chamado por ele). Finalizada: contexto ou
  # abertura (ClinicalRecord::Access).
  def read_grant
    return ClinicalRecord::Access::Grant.new(kind: :in_context, opening: nil, reason: nil) if @consultation.draft?

    ClinicalRecord::Access.call(user: Current.user, patient: @consultation.patient)
  end

  def respond(result)
    return failure(result) if result.failure?

    render json: Consultations::Json.consultation(result.payload[:consultation].reload)
  end

  def failure(result)
    return render(json: { error: "feature_disabled", feature: ClinicalRecord::Gate::KEY }, status: :forbidden) if result.reason == :feature_disabled

    render_failure(result, ERROR_STATUS)
  end

  def body = params.to_unsafe_h.except("controller", "action", "id")

  def not_found = render(json: { error: "not_found" }, status: :not_found)
end
```

(`read_grant` fica privado e é reusado pelo impresso na Task 14.)

```ruby
# app/controllers/clinical_records_controller.rb
# O prontuário (ADR 0031; spec §5; contratos §3): em contexto pelo atendimento,
# ou fora dele só com abertura justificada válida. Toda leitura deixa trilha;
# a escuta do dia (módulo 18) publica screening.viewed.
class ClinicalRecordsController < ApplicationController
  include Authentication
  include AttendanceAccess
  include ClinicalRecordGate

  before_action :require_clinical_record!
  before_action :require_professional

  def context
    attendance = Attendance.find_by(id: params[:id])
    return not_found unless attendance

    citizen = attendance.citizen
    return render(json: { error: "citizen_not_verified" }, status: :conflict) unless citizen.verification_level_verified?

    patient = (citizen.patient_id && Patient.find_by(id: citizen.patient_id)) || Patient.find_by(cpf: citizen.cpf)
    grant = ClinicalRecord::Access.call(user: Current.user, patient: patient, attendance: attendance)
    return forbid(grant.reason.to_s) unless grant.allowed?

    ClinicalRecord::Trail.viewed!(patient: patient, user: Current.user, grant: grant)
    screening = attendance.screening
    screening = nil unless screening&.completed?
    DomainEvents.publish("screening.viewed", screening_id: screening.id, user_id: Current.user.id) if screening
    render json: ClinicalRecord::Json.record(patient: patient, citizen: citizen, grant: grant, screening: screening)
  end

  def patient
    patient = Patient.find_by(id: params[:id])
    return not_found unless patient

    opening = ClinicalRecordOpening.valid_for(user_id: Current.user.id, patient_id: patient.id).order(created_at: :desc).first
    return forbid("opening_required") unless opening

    grant = ClinicalRecord::Access::Grant.new(kind: :justified, opening: opening, reason: nil)
    ClinicalRecord::Trail.viewed!(patient: patient, user: Current.user, grant: grant)
    citizen = Citizen.not_erased.where(cpf: patient.cpf).order(:created_at).first
    render json: ClinicalRecord::Json.record(patient: patient, citizen: citizen, grant: grant)
  end

  private

  def not_found = render(json: { error: "not_found" }, status: :not_found)
end
```

```ruby
# app/controllers/clinical_record_openings_controller.rb
# Abertura justificada (step-up) e o relatório das aberturas (municipal_admin)
# (ADR 0031; spec §5; contratos §3). O relatório nunca mostra a nota.
class ClinicalRecordOpeningsController < ApplicationController
  include Authentication
  include AttendanceAccess
  include ClinicalRecordGate
  include MfaStepUp

  REPORT_LIMIT = 500
  ERROR_STATUS = { missing_role: :forbidden, missing_link: :forbidden, cbo_not_allowed: :forbidden,
                   patient_not_found: :not_found, invalid_reason: :unprocessable_entity }.freeze
  DATE = /\A\d{4}-\d{2}-\d{2}\z/

  before_action :require_clinical_record!
  before_action :require_professional, only: :create
  before_action :require_report_role, only: :index

  def create
    return require_step_up! unless reauthenticated_recently?

    result = ClinicalRecord::Open.call(user: Current.user, cpf: params[:cpf], reason_code: params[:reason_code],
                                       reason_note: params[:reason_note])
    return render_failure(result, ERROR_STATUS) if result.failure?

    opening = result.payload[:opening]
    render json: { opening_id: opening.id, patient_id: opening.patient_id, expires_at: opening.expires_at.iso8601 },
           status: :created
  end

  def index
    from, to = period
    return render(json: { error: "invalid_period" }, status: :unprocessable_entity) if from == :invalid || to == :invalid

    scope = ClinicalRecordOpening.includes(:patient, user: :professional).order(created_at: :desc, id: :desc).limit(REPORT_LIMIT)
    scope = scope.where(created_at: from..) if from
    scope = scope.where(created_at: ..to) if to
    scope = scope.where(user_id: params[:user_id]) if params[:user_id].is_a?(String) && params[:user_id].present?
    render json: { items: scope.map { |o| item(o) } }
  end

  private

  def require_report_role
    forbid("missing_role") unless CitizenVerificationPolicy.new(Current.user, nil).manage?
  end

  # from/to "AAAA-MM-DD" no fuso da cidade (Time.zone dentro do request).
  def period
    [ [ :from, :beginning_of_day ], [ :to, :end_of_day ] ].map do |key, edge|
      value = params[key]
      next nil if value.blank?
      next :invalid unless value.is_a?(String) && value.match?(DATE)

      Date.iso8601(value).in_time_zone.public_send(edge)
    rescue Date::Error
      :invalid
    end
  end

  def item(opening)
    { id: opening.id, user_name: Screenings::Json.staff_name(opening.user), cpf_masked: opening.patient.cpf_masked,
      reason_code: opening.reason_code, created_at: opening.created_at.iso8601, expires_at: opening.expires_at.iso8601 }
  end
end
```

```ruby
# app/controllers/sigtap_procedures_controller.rb
# Busca de exame SIGTAP para o pedido da consulta (ADR 0031; contratos §5):
# grupo 02 da competência ativa, até 20; termo no corpo. Sem release: 503.
class SigtapProceduresController < ApplicationController
  include Authentication
  include AttendanceAccess
  include ClinicalRecordGate

  wrap_parameters false

  before_action :require_clinical_record!
  before_action :require_professional

  def search
    today = Time.zone.today
    return render(json: { error: "terminology_unavailable" }, status: :service_unavailable) unless ClinicalTerms::SigtapExams.release(on: today)

    items = ClinicalTerms::SigtapExams.search(params[:q], on: today)
    render json: { items: items.map { |e| { code: e.code, label: e.label } } }
  end
end
```

Em `app/controllers/ciap2_codes_controller.rb`, troque `search` por:

```ruby
  # ADR 0031 (contratos §5): `terminology` cid10 busca na CID-10 ativa; padrão ciap2.
  def search
    terminology = params.key?(:terminology) ? params[:terminology] : "ciap2"
    return render(json: { error: "invalid_terminology" }, status: :unprocessable_entity) unless %w[ciap2 cid10].include?(terminology)

    query = params[:q].is_a?(String) ? params[:q] : ""
    if terminology == "cid10"
      return render(json: { error: "terminology_unavailable" }, status: :service_unavailable) unless ClinicalTerms.release("cid10")

      return render json: { items: ClinicalTerms.search("cid10", query).map { |c| { code: c.code, label: c.label } } }
    end
    return render(json: { error: "terminology_unavailable" }, status: :service_unavailable) unless Screenings::Ciap2.release

    render json: { items: Screenings::Ciap2.search(query).map { |c| { code: c.code, label: c.label } } }
  end
```

e o comentário do topo ganha `# ADR 0031: também CID-10 (terminology: "cid10").`

- [ ] **Step 5: Rode e veja passar (e as specs da escuta e do atendimento)**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/requests/consultations_spec.rb spec/requests/clinical_record_spec.rb spec/requests/clinical_terms_search_spec.rb spec/requests/screenings_spec.rb spec/requests/attendances_spec.rb`
Expected: PASS. (A rota e a linha do impresso na spec da recepção entram na Task 14.)

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add app/controllers/consultations_controller.rb app/controllers/clinical_records_controller.rb app/controllers/clinical_record_openings_controller.rb app/controllers/sigtap_procedures_controller.rb app/controllers/ciap2_codes_controller.rb config/routes.rb spec/requests/consultations_spec.rb spec/requests/clinical_record_spec.rb spec/requests/clinical_terms_search_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: expose the consultation, clinical record, openings and terminology search routes

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 14: Impresso PDF para assinatura manual (Prawn)

**Escolha da gem.** O `Gemfile` não tem gem de PDF. A mais simples que resolve é **Prawn** (`prawn ~> 2.5`): Ruby puro, sem binário de sistema (ao contrário de `wicked_pdf`, que exige o `wkhtmltopdf`, ou `grover`, que exige Chromium na imagem), paginação automática de texto longo e fonte embutida (Helvetica, WinAnsi) que cobre o português. O teste lê o PDF com **pdf-reader** (`~> 2.12`, só no grupo `test`, também Ruby puro). O PDF é gerado na hora e nunca gravado (spec §4).

**Files:**
- Modify: `Gemfile`, `Gemfile.lock`
- Create: `app/services/consultations/print.rb`
- Modify: `app/controllers/consultations_controller.rb`, `config/routes.rb`, `spec/requests/clinical_record_spec.rb`
- Test: `spec/services/consultations/print_spec.rb`, `spec/requests/consultation_print_spec.rb`

**Interfaces:**
- Consumes: `Consultations::Effective`, `Ledi::ConsultationMapping`, `Professionals::Cbo`, `ClinicalTerms`, `ClinicalTerms::SigtapExams`, `ConsultationsController#read_grant` (Task 13).
- Produces: `Consultations::Print.call(consultation) -> String` (bytes do PDF; levanta `Consultations::Print::NotPrintable` se a consulta não está finalizada ou o paciente não tem nome); `Consultations::Print.safe(text) -> String` (texto que a fonte embutida aceita: o que ela não tem vira `?`); `GET /attendance/consultations/:id/print` → `application/pdf`, `Cache-Control: no-store`; 409 `not_finalized`, `patient_name_missing`; 403 como a leitura; trilha `clinical_record.viewed`.

- [ ] **Step 1: A gem**

Em `Gemfile`, depois de `gem "thrift", "~> 0.22"`:

```ruby
gem "prawn", "~> 2.5"     # ADR 0031: impresso da consulta, gerado na hora (Ruby puro, sem binário)
```

e, no `group :test do`, depois de `gem "vcr"`:

```ruby
  gem "pdf-reader", "~> 2.12"   # ADR 0031: lê o impresso nas specs
```

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle install`
Expected: `Bundle complete!`; o `Gemfile.lock` ganha `prawn`, `pdf-core`, `ttfunk` e `pdf-reader` (com as dependências deles).

- [ ] **Step 2: Escreva as specs que falham**

```ruby
# spec/services/consultations/print_spec.rb
require "rails_helper"

# ADR 0031 (spec §4): consulta finalizada, adendos em ordem, unidade,
# profissional (nome, conselho, CBO), paciente (nome de exibição, CPF,
# nascimento) e espaço para assinatura e carimbo. Review Focus 2: texto que a
# fonte não tem nunca derruba o impresso.
RSpec.describe Consultations::Print do
  before { Current.city = clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }
  after { Current.reset }

  let(:unit) { create_unit("UBS Jardim das Flores") }
  let(:doctor) { doctor!(unit) }
  let(:citizen) { verified_citizen!(1, social_name: "Mariana", age: 46) }

  def text_of(bytes) = PDF::Reader.new(StringIO.new(bytes)).pages.map(&:text).join("\n").squeeze(" ")

  it "traz o registro, o paciente, o profissional, os adendos em ordem e a assinatura" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "primeiro adendo aqui", text: "Texto UM")
    Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "segundo adendo aqui", text: "Texto DOIS")
    text = text_of(described_class.call(consultation.reload))
    professional = doctor.professional
    cpf = citizen.cpf.sub(/\A(\d{3})(\d{3})(\d{3})(\d{2})\z/, '\1.\2.\3-\4')
    expect(text).to include("UBS Jardim das Flores", "Mariana", cpf, Date.iso8601(citizen.birth_date).strftime("%d/%m/%Y"),
                            professional.professional_name, "#{professional.council}-#{professional.council_state}",
                            "225125", "Refere sede e poliúria", "Diabetes mellitus tipo 2", "Metformina",
                            "T90", "Retorno para consulta agendada", "0202010503", "Assinatura e carimbo")
    expect(text.index("Texto UM")).to be < text.index("Texto DOIS")
    expect(text).not_to include("Maria Aparecida") # nome de exibição = social
  end

  it "caracteres fora da fonte, quebras e 20.000 caracteres não derrubam o impresso (Review Focus 2)" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen,
                                           subjective: "PA ≥ 140 😷 “aspas” \r\nlinha nova", plan: "y" * 20_000)
    bytes = described_class.call(consultation)
    expect(bytes).to start_with("%PDF")
    expect(text_of(bytes)).to include("PA ? 140 ? \"aspas\"").or include("PA ? 140 ? “aspas”")
    expect(described_class.safe("ç ã é ≥ 🙂")).to eq("ç ã é ? ?")
  end

  it "rascunho ou paciente sem nome não imprime" do
    draft = started_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2))
    expect { described_class.call(draft) }.to raise_error(described_class::NotPrintable)
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    consultation.patient.update_columns(full_name: nil, social_name: nil)
    expect { described_class.call(consultation.reload) }.to raise_error(described_class::NotPrintable)
  end
end
```

```ruby
# spec/requests/consultation_print_spec.rb
require "rails_helper"

# Contratos §4: GET /attendance/consultations/:id/print — PDF na hora, sem
# cache, com trilha; 409 not_finalized / patient_name_missing.
RSpec.describe "Impresso da consulta", type: :request do
  before { clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:citizen) { verified_citizen!(1) }
  def body = JSON.parse(response.body)

  it "PDF para quem lê; sem cache; trilha; rascunho 409; sem nome 409; fora de contexto 403" do
    draft = started_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    sign_in_as(doctor)
    get "/attendance/consultations/#{draft.id}/print"
    expect([ response.status, body["error"] ]).to eq([ 409, "not_finalized" ])

    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2))
    consulting_attendance!(unit, citizen: consultation.attendance.citizen, doctor: doctor)
    get "/attendance/consultations/#{consultation.id}/print"
    expect(response).to have_http_status(:ok)
    expect(response.media_type).to eq("application/pdf")
    expect(response.headers["Cache-Control"]).to include("no-store")
    expect(response.headers["Content-Disposition"]).to include("consulta.pdf")
    expect(response.headers["Content-Disposition"]).not_to include("Maria")
    expect(DomainEvent.where(name: "clinical_record.viewed").pluck(:payload).last)
      .to include("patient_id" => consultation.patient_id, "access" => "in_context")

    consultation.patient.update_columns(full_name: nil)
    get "/attendance/consultations/#{consultation.id}/print"
    expect([ response.status, body["error"] ]).to eq([ 409, "patient_name_missing" ])

    sign_in_as(doctor!(unit, cbo: "223505"))
    get "/attendance/consultations/#{consultation.id}/print"
    expect([ response.status, body["error"] ]).to eq([ 403, "out_of_context" ])
  end
end
```

Em `spec/requests/clinical_record_spec.rb`, no exemplo "a recepção recebe 403 em toda rota do prontuário", acrescente à lista `[ :get, "/attendance/consultations/#{consultation.id}/print" ],`.

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/consultations/print_spec.rb spec/requests/consultation_print_spec.rb`
Expected: FAIL (`uninitialized constant Consultations::Print`; `No route matches`).

- [ ] **Step 4: Implemente**

```ruby
# app/services/consultations/print.rb
# Impresso da consulta finalizada para assinatura e carimbo (ADR 0031; spec
# §4; até o 19b não há assinatura digital). Gerado na hora, nunca gravado. A
# fonte embutida (Helvetica/WinAnsi) não tem emoji nem símbolos como "≥": o
# texto passa por `safe`, que troca o que falta por "?" — o impresso nunca cai.
require "prawn"

module Consultations
  module Print
    class NotPrintable < StandardError; end

    OUTCOMES = { "discharged" => "Alta", "referred" => "Encaminhado", "return" => "Retorno",
                 "left" => "Saiu sem atendimento" }.freeze
    VITALS = { "systolic" => "PA sistólica (mmHg)", "diastolic" => "PA diastólica (mmHg)", "heart_rate" => "FC (bpm)",
               "respiratory_rate" => "FR (irpm)", "temperature_c" => "Temperatura (°C)", "spo2" => "SpO2 (%)",
               "capillary_glucose" => "Glicemia capilar (mg/dL)", "weight_kg" => "Peso (kg)", "height_cm" => "Altura (cm)",
               "pain_score" => "Dor (0-10)" }.freeze
    SECTIONS = { "subjective" => "Subjetivo", "objective" => "Objetivo", "assessment" => "Avaliação", "plan" => "Plano" }.freeze

    module_function

    def call(consultation)
      patient = consultation.patient
      raise NotPrintable, "not_finalized" unless consultation.finalized?
      raise NotPrintable, "patient_name_missing" if patient.full_name.blank?

      pdf = Prawn::Document.new(page_size: "A4", margin: 40, info: { Title: "Registro de consulta", Producer: "Rota Saúde" })
      pdf.font_size(10)
      header(pdf, consultation)
      patient_block(pdf, patient)
      professional_block(pdf, consultation)
      record(pdf, consultation)
      addenda(pdf, consultation)
      signature(pdf)
      pdf.render
    end

    def safe(text)
      text.to_s.gsub(/\r\n?/, "\n").encode("Windows-1252", invalid: :replace, undef: :replace, replace: "?").encode("UTF-8")
    end

    def line(pdf, label, value) = pdf.text(safe("#{label}: #{value}"))

    def title(pdf, text)
      pdf.move_down 8
      pdf.text safe(text), style: :bold, size: 11
    end

    def header(pdf, c)
      unit = c.attendance.health_unit
      pdf.text safe(CityProfile.current&.name.to_s), style: :bold, size: 12
      pdf.text safe("#{unit.name}#{unit.cnes ? " — CNES #{unit.cnes}" : ''}")
      pdf.text safe("Registro de atendimento individual — consulta"), size: 12, style: :bold
      line(pdf, "Atendimento", "#{c.started_at.in_time_zone.strftime('%d/%m/%Y %H:%M')} a #{c.finalized_at.in_time_zone.strftime('%H:%M')}")
      line(pdf, "Tipo", Ledi::ConsultationMapping.care_type_label(c.care_type))
    end

    def patient_block(pdf, patient)
      title(pdf, "Paciente")
      line(pdf, "Nome", patient.display_name)
      line(pdf, "CPF", patient.cpf.to_s.sub(/\A(\d{3})(\d{3})(\d{3})(\d{2})\z/, '\1.\2.\3-\4'))
      birth = patient.birth_date.present? ? Date.iso8601(patient.birth_date).strftime("%d/%m/%Y") : "não informado"
      line(pdf, "Nascimento", birth)
    end

    def professional_block(pdf, c)
      professional = c.author_user.professional
      title(pdf, "Profissional")
      line(pdf, "Nome", professional&.professional_name || c.author_user.email_address)
      line(pdf, "Conselho", professional && "#{professional.council}-#{professional.council_state} #{professional.registration_number}")
      line(pdf, "CBO", "#{c.cbo_code} #{Professionals::Cbo.find(c.cbo_code)&.title}")
    end

    def record(pdf, c)
      SECTIONS.each do |field, label|
        title(pdf, label)
        pdf.text safe(c.public_send(field).presence || "—")
      end
      vitals = c.vitals.except("glucose_moment")
      if vitals.any?
        title(pdf, "Sinais vitais")
        vitals.each { |column, value| line(pdf, VITALS.fetch(column), value.is_a?(BigDecimal) ? value.to_s("F") : value) }
      end
      effective = Effective.call(c)
      title(pdf, "Problemas avaliados")
      effective[:problems].each do |row|
        label = ClinicalTerms.label(row.terminology, row.code, row.terminology_release_id)
        pdf.text safe("#{row.terminology.upcase} #{row.code} #{label} — #{row.status_after == 'active' ? 'ativo' : 'resolvido'}")
      end
      title(pdf, "Condutas")
      effective[:conducts].each { |code| pdf.text safe(Ledi::ConsultationMapping.conduct_label(code)) }
      if effective[:exam_requests].any?
        title(pdf, "Exames solicitados")
        effective[:exam_requests].each do |exam|
          pdf.text safe("#{exam.sigtap_code} #{ClinicalTerms::SigtapExams.label(exam.sigtap_code, exam.sigtap_competence)}" \
                        "#{exam.cid10_justification ? " (CID-10 #{exam.cid10_justification})" : ''}")
        end
      end
      outcome(pdf, c.attendance)
    end

    def outcome(pdf, attendance)
      title(pdf, "Desfecho")
      text = OUTCOMES.fetch(attendance.outcome.to_s, attendance.outcome.to_s)
      text += " — #{attendance.referral_unit.name}" if attendance.referral_unit
      text += " — #{attendance.referral_note}" if attendance.referral_note.present?
      pdf.text safe(text)
    end

    def addenda(pdf, c)
      rows = c.addenda.order(:created_at, :id).to_a
      return if rows.empty?

      title(pdf, "Adendos")
      rows.each do |a|
        author = a.author_user.professional&.professional_name || a.author_user.email_address
        pdf.text safe("#{a.created_at.in_time_zone.strftime('%d/%m/%Y %H:%M')} — #{author} — motivo: #{a.reason}"), style: :bold
        pdf.text safe(a.text)
      end
    end

    def signature(pdf)
      pdf.move_down 40
      pdf.stroke_horizontal_line 0, 250
      pdf.move_down 4
      pdf.text safe("Assinatura e carimbo do profissional")
      pdf.move_down 8
      pdf.text safe("Impresso em #{Time.current.strftime('%d/%m/%Y %H:%M')}. Sem assinatura digital: vale com a assinatura manual."),
               size: 8
    end
    private_class_method :line, :title, :header, :patient_block, :professional_block, :record, :outcome, :addenda, :signature
  end
end
```

Em `config/routes.rb`, no `scope "/attendance"`, depois de `post "consultations/:id/addenda", …`: `get "consultations/:id/print", to: "consultations#print"`.

Em `app/controllers/consultations_controller.rb`, a ação (antes de `private`):

```ruby
  # Spec §4: PDF na hora, nunca gravado nem em cache; o nome do arquivo não
  # leva dado da pessoa.
  def print
    return render(json: { error: "not_finalized" }, status: :conflict) unless @consultation.finalized?

    grant = read_grant
    return forbid(grant.reason.to_s) unless grant.allowed?
    return render(json: { error: "patient_name_missing" }, status: :conflict) if @consultation.patient.full_name.blank?

    ClinicalRecord::Trail.viewed!(patient: @consultation.patient, user: Current.user, grant: grant)
    response.headers["Cache-Control"] = "no-store"
    send_data Consultations::Print.call(@consultation), type: "application/pdf", disposition: "inline", filename: "consulta.pdf"
  end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/consultations/print_spec.rb spec/requests/consultation_print_spec.rb spec/requests/clinical_record_spec.rb`
Expected: PASS. (Se o `pdf-reader` devolver as aspas tipográficas como as próprias — Windows-1252 tem “ ” —, o `or` da expectativa cobre os dois.)

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add Gemfile Gemfile.lock app/services/consultations/print.rb app/controllers/consultations_controller.rb config/routes.rb spec/services/consultations/print_spec.rb spec/requests/consultation_print_spec.rb spec/requests/clinical_record_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: print the finalized consultation as a PDF for manual signature

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 3 — Ficha LEDI (F-19.6)

### Task 15: `ledi_outbox` com `correction_pending` e a ficha `Ledi::Fichas::IndividualCare`

**Files:**
- Create: `db/city_migrate/20261007400003_add_ledi_correction_pending.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/ledi_outbox_entry.rb`
- Create: `app/services/ledi/fichas/individual_care.rb`
- Test: `spec/models/ledi_correction_pending_guard_spec.rb`, `spec/services/ledi/fichas/individual_care_spec.rb`

**Interfaces:**
- Consumes: `Ledi::ConsultationMapping` (Task 1), `Ledi::ScreeningMapping.{sex_code,turno}`, `Ledi::Fichas::{ScreeningIdentity,Medicoes}`, `Ledi::Ficha.assert!`, `Ledi::FichaTypes` (`atendimento_individual`, do 18).
- Produces:
  - `ledi_outbox.status` aceita `correction_pending` (exige `replaces_outbox_id`); `idx_ledi_outbox_source` ignora `rejected` e `correction_pending`; trigger `ledi_outbox_correction_guard`: linha que substitui uma ACEITA só nasce e só fica `correction_pending`; `LediOutboxEntry::STATUSES` com `correction_pending`;
  - `Ledi::Fichas::IndividualCare::Problem = Data(:uuid, :evolution_uuid, :sequence, :ciap, :cid10, :situation, :onset_on, :resolved_on)`, `::Care = Data(:care_type, :problems, :conducts, :exams, :measurements)` (`measurements` responde a `[]` e `glucose_moment`, ou nil); `Ledi::Fichas::IndividualCare.new(identity:, care:, source_id:)` — `type == "atendimento_individual"`, `competence`, `cnes`, `ine`, `source == { type: "Consultation", id: source_id }`, `to_thrift(uuid:) -> FichaAtendimentoIndividualMasterThrift`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/models/ledi_correction_pending_guard_spec.rb
require "rails_helper"

# ADR 0031 (Invariantes; spec §6): nenhuma correção de ficha aceita é enviada
# enquanto o reenvio após aceite não for confirmado (api#41): a linha que
# substitui uma aceita só existe como correction_pending e não sai dele.
RSpec.describe "ledi_outbox correction_pending" do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  def attempt(&) = ApplicationRecord.transaction(requires_new: true, &)

  def entry!(status: "pending", **attrs)
    LediOutboxEntry.create!({ uuid: "1234567-#{SecureRandom.uuid}", ficha_type: "atendimento_individual", competence: "202610",
                              source_type: "Consultation", source_id: SecureRandom.uuid, ledi_version: "8.7.0",
                              status: status, next_attempt_at: Time.current, bytes: "x".b }.merge(attrs))
  end

  let(:accepted) { entry!.tap(&:accept!) }

  it "a correção de uma aceita nasce correction_pending, uma por aceita, e o claim nunca a pega" do
    correction = entry!(status: "correction_pending", source_id: accepted.source_id, replaces_outbox_id: accepted.id)
    expect(LediOutboxEntry.claim!(limit: 10)).to be_empty
    expect(correction.reload.status).to eq("correction_pending")
    expect { attempt { entry!(status: "correction_pending", source_id: accepted.source_id, replaces_outbox_id: accepted.id) } }
      .to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "não nasce pending nem vira pending/sending; correction_pending exige a substituída" do
    expect { attempt { entry!(source_id: accepted.source_id, replaces_outbox_id: accepted.id) } }
      .to raise_error(ActiveRecord::StatementInvalid, /correction of an accepted ficha stays correction_pending/)
    correction = entry!(status: "correction_pending", source_id: accepted.source_id, replaces_outbox_id: accepted.id)
    expect { attempt { correction.update_columns(status: "pending") } }
      .to raise_error(ActiveRecord::StatementInvalid, /correction of an accepted ficha stays correction_pending/)
    expect { attempt { entry!(status: "correction_pending") } }.to raise_error(ActiveRecord::StatementInvalid, /ck_ledi_outbox_correction/)
    expect { correction.update!(bytes: "y".b) }.not_to raise_error
  end
end
```

```ruby
# spec/services/ledi/fichas/individual_care_spec.rb
require "rails_helper"

# ADR 0031 (spec §6; Task 1): consulta → Atendimento Individual contra os
# IDLs 8.7.0 — tipo, problemas com situação e uuids, condutas, exames
# solicitados, medições; só CPF; sem medicamento nem texto.
RSpec.describe Ledi::Fichas::IndividualCare do
  let(:started) { Time.zone.parse("2026-10-07 09:10") }
  let(:identity) do
    Ledi::Fichas::ScreeningIdentity.new(cnes: "1234567", ine: "0000123456", professional_cns: "700000000000005",
                                        cbo: "225125", citizen_cpf: "52998224725", birth_date: Date.new(1980, 5, 10),
                                        sex: "female", started_at: started, ended_at: started + 20.minutes, ibge_code: "4106902")
  end
  let(:problems) do
    [ described_class::Problem.new(uuid: "p-1", evolution_uuid: "e-1", sequence: 1, ciap: "T90", cid10: nil, situation: 0,
                                   onset_on: Date.new(2025, 8, 1), resolved_on: nil),
      described_class::Problem.new(uuid: "p-2", evolution_uuid: "e-2", sequence: 3, ciap: nil, cid10: "N390", situation: 2,
                                   onset_on: nil, resolved_on: Date.new(2026, 10, 7)) ]
  end
  let(:measurements) { ScreeningRevision.new(systolic: 130, diastolic: 85, weight_kg: BigDecimal("82.5"), height_cm: 170) }
  let(:care) { described_class::Care.new(care_type: 5, problems: problems, conducts: [ 1, 9 ], exams: [ "0202010503" ], measurements: measurements) }
  let(:ficha) { described_class.new(identity: identity, care: care, source_id: "f0f0f0f0-0000-4000-8000-000000000009") }
  def ms(time) = (time.to_f * 1000).to_i

  it "cumpre a interface Ledi::Ficha" do
    expect { Ledi::Ficha.assert!(ficha) }.not_to raise_error
    expect([ ficha.type, ficha.competence, ficha.cnes, ficha.ine ]).to eq([ "atendimento_individual", "202610", "1234567", "0000123456" ])
    expect(ficha.source).to eq(type: "Consultation", id: "f0f0f0f0-0000-4000-8000-000000000009")
  end

  it "monta o MIAI da consulta" do
    master = ficha.to_thrift(uuid: "1234567-c")
    expect([ master.uuidFicha, master.tpCdsOrigem ]).to eq([ "1234567-c", 3 ])
    lotacao = master.headerTransport.lotacaoFormPrincipal
    expect([ lotacao.profissionalCNS, lotacao.cboCodigo_2002, lotacao.cnes, lotacao.ine ]).to eq(%w[700000000000005 225125 1234567 0000123456])
    child = master.atendimentosIndividuais.sole
    expect([ child.tipoAtendimento, child.localDeAtendimento, child.turno, child.sexo, child.condutas ]).to eq([ 5, 1, 1, 1, [ 1, 9 ] ])
    expect([ child.cpfCidadao, child.cns, child.stCidadaoNaoPossuiCpf, child.medicamentos ]).to eq([ "52998224725", nil, false, nil ])
    expect([ child.dataHoraInicialAtendimento, child.dataHoraFinalAtendimento ]).to eq([ ms(started), ms(started + 20.minutes) ])
    first, second = child.problemasCondicoes
    expect([ first.uuidProblema, first.uuidEvolucaoProblema, first.coSequencialEvolucao, first.ciap, first.cid10, first.situacao,
             first.dataInicioProblema, first.dataFimProblema, first.isAvaliado ])
      .to eq([ "p-1", "e-1", 1, "T90", nil, 0, ms(Date.new(2025, 8, 1).in_time_zone), nil, true ])
    expect([ second.ciap, second.cid10, second.situacao, second.dataFimProblema ])
      .to eq([ nil, "N390", 2, ms(Date.new(2026, 10, 7).in_time_zone) ])
    exam = child.exame.sole
    expect([ exam.codigoExame, exam.solicitadoAvaliado ]).to eq([ "0202010503", [ "S" ] ])
    expect([ child.medicoes.pressaoArterialSistolica, child.medicoes.peso ]).to eq([ 130, 82.5 ])
    expect { master.validate }.not_to raise_error
    expect(Ledi::Version.deserialize(master.class, Ledi::Version.serialize(master))).to eq(master)
  end

  it "sem exame nem medição: os campos ficam fora" do
    bare = described_class::Care.new(care_type: 2, problems: problems.first(1), conducts: [ 9 ], exams: [], measurements: nil)
    child = described_class.new(identity: identity, care: bare, source_id: SecureRandom.uuid).to_thrift(uuid: "x").atendimentosIndividuais.sole
    expect([ child.exame, child.medicoes, child.tipoAtendimento ]).to eq([ nil, nil, 2 ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/ledi/fichas/individual_care_spec.rb`
Expected: FAIL (`uninitialized constant Ledi::Fichas::IndividualCare`). (A spec de guarda só roda depois da migração, no Step 5.)

- [ ] **Step 3: A migração e o trigger**

```ruby
# db/city_migrate/20261007400003_add_ledi_correction_pending.rb
# Módulo 19 (ADR 0031; spec §6): adendo que muda dado estruturado de uma
# ficha JÁ ACEITA vira uma linha correction_pending (com replaces_outbox_id =
# a aceita), que nunca é enviada até a regra de reenvio após aceite ser
# confirmada (api#41). O índice único por fonte passa a ignorá-la.
class AddLediCorrectionPending < ActiveRecord::Migration[8.1]
  def up
    remove_check_constraint :ledi_outbox, name: "ck_ledi_outbox_status"
    add_check_constraint :ledi_outbox,
                         "status::text = ANY (ARRAY['pending'::text, 'sending'::text, 'accepted'::text, 'rejected'::text, 'failed'::text, 'correction_pending'::text])",
                         name: "ck_ledi_outbox_status"
    add_check_constraint :ledi_outbox, "status::text <> 'correction_pending'::text OR replaces_outbox_id IS NOT NULL",
                         name: "ck_ledi_outbox_correction"
    remove_index :ledi_outbox, name: "idx_ledi_outbox_source"
    add_index :ledi_outbox, %i[source_type source_id ficha_type], unique: true, name: "idx_ledi_outbox_source",
                                                                  where: "(status)::text <> ALL (ARRAY['rejected'::text, 'correction_pending'::text])"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

No fim de `db/city_triggers.sql`:

```sql
-- Correção de ficha aceita (ADR 0031, Invariantes; spec §6; api#41): a linha
-- que substitui uma ficha ACEITA nasce e fica correction_pending — nunca
-- pending/sending, logo nunca vai ao PEC — até o reenvio após aceite ser
-- confirmado. O conteúdo pode ser regravado (novo adendo).
CREATE OR REPLACE FUNCTION rota_ledi_outbox_correction_guard() RETURNS trigger AS $fn$
BEGIN
  IF NEW.replaces_outbox_id IS NOT NULL AND NEW.status <> 'correction_pending'
     AND EXISTS (SELECT 1 FROM ledi_outbox o WHERE o.id = NEW.replaces_outbox_id AND o.status = 'accepted') THEN
    RAISE EXCEPTION 'ledi_outbox: the correction of an accepted ficha stays correction_pending (api#41)';
  END IF;
  IF TG_OP = 'UPDATE' AND OLD.status = 'correction_pending' AND NEW.status IS DISTINCT FROM OLD.status THEN
    RAISE EXCEPTION 'ledi_outbox: the correction of an accepted ficha stays correction_pending (api#41)';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF to_regclass('public.ledi_outbox') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS ledi_outbox_correction_guard ON ledi_outbox';
    EXECUTE 'CREATE TRIGGER ledi_outbox_correction_guard
      BEFORE INSERT OR UPDATE ON ledi_outbox
      FOR EACH ROW EXECUTE FUNCTION rota_ledi_outbox_correction_guard()';
  END IF;
END
$do$;
```

(O trigger roda em toda migração que reexecuta o arquivo; entre a 300002 do 18 e esta, `NEW.replaces_outbox_id` já existe.)

Em `db/city_schema.rb`: `define(version: 2026_10_07_400003)`; em `ledi_outbox`, troque o índice `idx_ledi_outbox_source` por `t.index ["source_type", "source_id", "ficha_type"], name: "idx_ledi_outbox_source", unique: true, where: "((status)::text <> ALL (ARRAY[('rejected'::character varying)::text, ('correction_pending'::character varying)::text]))"`, troque `ck_ledi_outbox_status` pela lista com `'correction_pending'::text` e acrescente `t.check_constraint "status::text <> 'correction_pending'::text OR replaces_outbox_id IS NOT NULL", name: "ck_ledi_outbox_correction"`. (O `where` do índice é o que o Postgres devolve; se a paridade acusar outra forma, copie a do erro — é normalização do banco.)

Em `app/models/ledi_outbox_entry.rb`: `STATUSES = %w[pending sending accepted rejected failed correction_pending].freeze` (comentário: `# correction_pending: correção de ficha aceita, nunca enviada até api#41 (ADR 0031).`).

- [ ] **Step 4: A ficha**

```ruby
# app/services/ledi/fichas/individual_care.rb
# Consulta da APS como Atendimento Individual (ADR 0031; spec §6; Task 1):
# tipo de atendimento, problemas avaliados com situação (uuid do problema,
# da evolução e sequência), condutas, exames solicitados (SIGTAP grupo 02,
# "S"), medições, início e fim; só o CPF do cidadão. Sem medicamento nem texto
# SOAP. Pura: recebe valores (Ledi::ConsultationFicha monta). Implementa
# Ledi::Ficha.
module Ledi
  module Fichas
    class IndividualCare
      ORIGIN_THIRD_PARTY = 3
      CM = Ledi::ConsultationMapping
      SM = Ledi::ScreeningMapping

      Problem = Data.define(:uuid, :evolution_uuid, :sequence, :ciap, :cid10, :situation, :onset_on, :resolved_on)
      Care = Data.define(:care_type, :problems, :conducts, :exams, :measurements)

      attr_reader :identity, :care, :source_id

      def initialize(identity:, care:, source_id:)
        @identity, @care, @source_id = identity, care, source_id
      end

      def type = "atendimento_individual"
      def competence = identity.started_at.in_time_zone.strftime("%Y%m")
      def cnes = identity.cnes
      def ine = identity.ine
      def source = { type: "Consultation", id: source_id }

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
          dataNascimento: ms(identity.birth_date.in_time_zone), localDeAtendimento: CM.local_de_atendimento,
          sexo: SM.sex_code(identity.sex), turno: SM.turno(identity.started_at), tipoAtendimento: care.care_type,
          condutas: care.conducts, dataHoraInicialAtendimento: ms(identity.started_at),
          dataHoraFinalAtendimento: ms(identity.ended_at), cpfCidadao: identity.citizen_cpf, stCidadaoNaoPossuiCpf: false,
          ficouEmObservacao: false, medicoes: care.measurements && Medicoes.build(care.measurements),
          problemasCondicoes: care.problems.map { |p| problem(common, p) },
          exame: care.exams.empty? ? nil : care.exams.map { |code| common::ExameThrift.new(codigoExame: code, solicitadoAvaliado: [ CM.exam_requested ]) }
        )
        ai::FichaAtendimentoIndividualMasterThrift.new(uuidFicha: uuid, tpCdsOrigem: ORIGIN_THIRD_PARTY,
                                                       headerTransport: header, atendimentosIndividuais: [ child ])
      end

      private

      def problem(common, p)
        common::ProblemaCondicaoThrift.new(
          uuidProblema: p.uuid, uuidEvolucaoProblema: p.evolution_uuid, coSequencialEvolucao: p.sequence, ciap: p.ciap,
          cid10: p.cid10, situacao: p.situation, dataInicioProblema: p.onset_on && ms(p.onset_on.in_time_zone),
          dataFimProblema: p.resolved_on && ms(p.resolved_on.in_time_zone), isAvaliado: true
        )
      end

      def ms(time) = (time.to_f * 1000).to_i
    end
  end
end
```

- [ ] **Step 5: Rode a migração nos bancos de teste e as specs**

```bash
psql -U rota_saude -d postgres -c "DROP DATABASE rota_saude_test_city_a" -c "DROP DATABASE rota_saude_test_city_b"
docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod19 api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/models/ledi_correction_pending_guard_spec.rb spec/services/ledi spec/models/ledi_outbox_entry_spec.rb spec/services/city_schema_spec.rb spec/jobs/ledi spec/commands/ledi spec/requests/production_spec.rb
```
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add db/city_migrate/20261007400003_add_ledi_correction_pending.rb db/city_schema.rb db/city_triggers.sql app/models/ledi_outbox_entry.rb app/services/ledi/fichas/individual_care.rb spec/models/ledi_correction_pending_guard_spec.rb spec/services/ledi/fichas/individual_care_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: build the individual care LEDI ficha and hold accepted-ficha corrections

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 16: Geração da ficha (`Ledi::ConsultationFicha`), regeneração por adendo, `correction_pending` e a Produção

**Files:**
- Create: `app/services/ledi/consultation_ficha.rb`, `app/services/ledi/ficha_sources.rb`, `app/jobs/ledi/consultation_ficha_job.rb`
- Modify: `app/commands/consultations/finalize.rb`, `app/commands/consultations/add_addendum.rb`, `app/commands/ledi/resend.rb`, `app/controllers/production_controller.rb`
- Test: `spec/services/ledi/consultation_ficha_spec.rb`, `spec/jobs/ledi/consultation_ficha_job_spec.rb`, `spec/requests/production_consultation_spec.rb`

**Interfaces:**
- Consumes: Tasks 10, 11, 15; `Ledi::ScreeningFicha.{exportable?,team_ine}` e `::AlreadyResolved` (18), `Ledi::Enqueue.call(ficha, city:, replaces:)`, `Ledi::Transport.{wrap,read}`, `LediGenerationFailure`.
- Produces:
  - `Ledi::ConsultationFicha::SOURCE_TYPE == "Consultation"`, `::InFlight` (erro), `.build(consultation) -> [IndividualCare|nil, Array<String>]`, `.generate(consultation, city: Current.city) -> :enqueued | :exists | :failed | :unusable | :skipped`, `.refresh!(consultation, city: Current.city) -> :rewritten | :regenerated | :correction_pending | :enqueued | :failed | :unusable` (levanta `InFlight` com a ficha em `sending`), `.retry!(failure, by:)`, `.regenerate(entry, by:, city: Current.city) -> [Symbol, LediOutboxEntry|nil]`;
  - `Ledi::FichaSources.for(source_type) -> Module|nil` (`Screening` → `Ledi::ScreeningFicha`, `Consultation` → `Ledi::ConsultationFicha`), `.attendance_ids(failures) -> Hash{source_id => attendance_id}`;
  - `Ledi::ConsultationFichaJob.enqueue_for(consultation, reason: "finalized" | "addendum")`, `#perform(city_slug:, consultation_id:, reason:)` (`retry_on InFlight`, 1 min, 10 tentativas);
  - `GET /production/generation_failures` e `/retry` passam a valer para `Consultation`; `POST /production/fichas/:id/resend` regera a ficha de consulta recusada da origem.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/ledi/consultation_ficha_spec.rb
require "rails_helper"

# ADR 0031 (spec §6, §8): a ficha nasce na finalização (exportação
# utilizável); falta de identificação → "não gerada"; duas fichas no mesmo
# atendimento (escuta e consulta); adendo regera antes do aceite e vira
# correction_pending depois — nunca enviada.
RSpec.describe Ledi::ConsultationFicha do
  include ActiveSupport::Testing::TimeHelpers

  let(:city) { ledi_ready!(clinical_city!, pec_url: "https://pec.a.test", record_mode: "record", ibge_code: "4106902") }
  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }

  before do
    Current.city = city
    ciap2_release!; cid10_release!; sigtap_release!
    allow(Ledi::DeliverJob).to receive(:perform_later)
  end
  after { Current.reset }

  def ficha_of(entry)
    Ledi::Version.deserialize(Ledi::FichaTypes.klass(entry.ficha_type), Ledi::Transport.read(entry.bytes).dadoSerializado)
  end

  def finalized!(n = 1, **over)
    exportable_unit!(unit, doctor)
    finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(n), **over)
  end

  def addendum!(consultation, changes) =
    Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "correção estruturada", text: "ajuste", changes: changes)

  it "gera o Atendimento Individual da consulta uma vez; transporte tipo 4; problemas, condutas e exames" do
    consultation = finalized!
    expect(described_class.generate(consultation)).to eq(:enqueued)
    entry = LediOutboxEntry.sole
    expect(entry).to have_attributes(source_type: "Consultation", source_id: consultation.id, ficha_type: "atendimento_individual")
    expect(Ledi::Transport.read(entry.bytes).tipoDadoSerializado).to eq(4)
    child = ficha_of(entry).atendimentosIndividuais.sole
    problem = PatientProblem.sole
    expect([ child.tipoAtendimento, child.condutas, child.exame.map(&:codigoExame) ]).to eq([ 5, [ 1 ], [ "0202010503" ] ])
    item = child.problemasCondicoes.sole
    expect([ item.uuidProblema, item.uuidEvolucaoProblema, item.coSequencialEvolucao, item.ciap, item.situacao ])
      .to eq([ problem.id, ConsultationProblem.sole.id, 1, "T90", 0 ])
    expect(child.medicoes.pressaoArterialSistolica).to eq(130)
    expect(described_class.generate(consultation)).to eq(:exists)
  end

  it "duas fichas no mesmo atendimento: a da escuta (enfermeira) e a da consulta (médica)" do
    nurse = screener!(unit)
    exportable_unit!(unit, nurse, doctor)
    citizen = verified_citizen!(1)
    attendance = walk_in_attendance!(unit, citizen: citizen)
    screening = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
    Screenings::Complete.call(screening: screening, revision_params: revision_params, destination: "same_day", destination_params: {}, by: nurse)
    Attendances::Call.call(attendance: attendance, health_unit_id: unit.id, by: doctor)
    consultation = Consultations::Start.call(attendance: attendance.reload, by: doctor).payload[:consultation]
    Consultations::SaveDraft.call(consultation: consultation, params: draft_body(vitals: {}), by: doctor)
    Consultations::Finalize.call(consultation: consultation.reload, outcome_params: { "outcome" => "discharged" }, by: doctor)
    Ledi::ScreeningFicha.generate(screening.reload)
    described_class.generate(consultation.reload)
    expect(LediOutboxEntry.pluck(:source_type).sort).to eq(%w[Consultation Screening])
    # Sem sinais na consulta, as medições vêm da escuta do mesmo atendimento.
    entry = LediOutboxEntry.find_by!(source_type: "Consultation")
    expect(ficha_of(entry).atendimentosIndividuais.sole.medicoes.pressaoArterialSistolica).to eq(130)
  end

  it "sem identificação: 'não gerada' com os motivos; corrigido, gera e resolve" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    consultation.patient.update_columns(sex: nil)
    expect(described_class.generate(consultation)).to eq(:failed)
    failure = LediGenerationFailure.sole
    expect(failure.reason_codes).to eq(%w[unit_without_cnes professional_without_team citizen_without_sex])
    expect(DomainEvent.where(name: "ledi.generation_failed").sole.payload)
      .to eq("failure_id" => failure.id, "source_type" => "Consultation", "source_id" => consultation.id)
    exportable_unit!(unit, doctor)
    consultation.patient.update_columns(sex: "female")
    expect(described_class.retry!(failure, by: ledi_admin!).resolved_at).to be_present
    expect(LediOutboxEntry.where(source_id: consultation.id).count).to eq(1)
  end

  it "exportação desligada ou record_mode off: nada nasce" do
    consultation = finalized!
    ledi_off!(city)
    expect(described_class.generate(consultation)).to eq(:unusable)
    expect([ LediOutboxEntry.count, LediGenerationFailure.count ]).to eq([ 0, 0 ])
  end

  it "adendo antes do aceite: pendente regravada com o mesmo uuid; recusada regerada com replaces" do
    consultation = finalized!
    described_class.generate(consultation)
    entry = LediOutboxEntry.sole
    addendum!(consultation, "conducts" => [ 1, 9 ])
    expect(described_class.refresh!(consultation.reload)).to eq(:rewritten)
    expect(entry.reload.uuid).to eq(entry.uuid)
    expect(ficha_of(entry).atendimentosIndividuais.sole.condutas).to eq([ 1, 9 ])

    entry.reject!([ { "field" => "condutas", "code" => "invalid" } ])
    addendum!(consultation, "conducts" => [ 9 ])
    expect(described_class.refresh!(consultation.reload)).to eq(:regenerated)
    fresh = LediOutboxEntry.find_by!(replaces_outbox_id: entry.id)
    expect([ fresh.status, ficha_of(fresh).atendimentosIndividuais.sole.condutas ]).to eq([ "pending", [ 9 ] ])
  end

  it "adendo depois do aceite: uma correction_pending por aceita, regravada e nunca enviada (api#41)" do
    consultation = finalized!
    described_class.generate(consultation)
    accepted = LediOutboxEntry.sole.tap(&:accept!)
    addendum!(consultation, "conducts" => [ 1, 9 ])
    expect(described_class.refresh!(consultation.reload)).to eq(:correction_pending)
    addendum!(consultation, "conducts" => [ 9 ])
    expect(described_class.refresh!(consultation.reload)).to eq(:correction_pending)
    correction = LediOutboxEntry.find_by!(replaces_outbox_id: accepted.id)
    expect([ correction.status, LediOutboxEntry.count ]).to eq([ "correction_pending", 2 ])
    expect(ficha_of(correction).atendimentosIndividuais.sole.condutas).to eq([ 9 ])
    expect(LediOutboxEntry.claim!(limit: 10)).to be_empty
  end

  it "ficha em envio: tenta de novo depois (InFlight)" do
    consultation = finalized!
    described_class.generate(consultation)
    LediOutboxEntry.update_all(status: "sending")
    addendum!(consultation, "conducts" => [ 1, 9 ])
    expect { described_class.refresh!(consultation.reload) }.to raise_error(described_class::InFlight)
  end

  it "problema resolvido por adendo depois do atendimento: dataFimProblema não passa do dia da consulta" do
    consultation = finalized!
    described_class.generate(consultation)
    LediOutboxEntry.sole.reject!([ { "field" => "other", "code" => "unknown" } ])
    travel_to(3.days.from_now) do
      addendum!(consultation, "evaluated_problems" => [ { "problem_id" => PatientProblem.sole.id, "action" => "resolve" } ])
      described_class.refresh!(consultation.reload)
    end
    fresh = LediOutboxEntry.where(status: "pending").sole
    problem = ficha_of(fresh).atendimentosIndividuais.sole.problemasCondicoes.sole
    expect(problem.situacao).to eq(2)
    expect(problem.dataFimProblema).to eq((consultation.started_at.in_time_zone.to_date.in_time_zone.to_f * 1000).to_i)
  end
end
```

```ruby
# spec/jobs/ledi/consultation_ficha_job_spec.rb
require "rails_helper"

# ADR 0031 (spec §6): a finalização enfileira a ficha; adendo com mudança
# estruturada enfileira a regeneração; só texto, nada.
RSpec.describe Ledi::ConsultationFichaJob do
  let(:city) { ledi_ready!(clinical_city!, pec_url: "https://pec.a.test", record_mode: "record", ibge_code: "4106902") }
  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }

  before do
    Current.city = city
    ciap2_release!; cid10_release!; sigtap_release!
    allow(Ledi::DeliverJob).to receive(:perform_later)
    exportable_unit!(unit, doctor)
  end
  after { Current.reset }

  def enqueued = ActiveJob::Base.queue_adapter.enqueued_jobs.select { |j| j[:job] == described_class }

  it "finalizar enfileira; o job gera; adendo estruturado enfileira refresh; adendo só texto não" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    expect(enqueued.size).to eq(1)
    args = ActiveJob::Arguments.deserialize(enqueued.first[:args]).first
    expect(args).to eq(city_slug: city.slug, consultation_id: consultation.id, reason: "finalized")
    described_class.perform_now(**args)
    expect(LediOutboxEntry.where(source_id: consultation.id).count).to eq(1)
    Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "só texto acrescentado", text: "x")
    expect(enqueued.size).to eq(1)
    Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "conduta acrescentada", text: "x",
                                    changes: { "conducts" => [ 1, 9 ] })
    expect(ActiveJob::Arguments.deserialize(enqueued.last[:args]).first).to include(reason: "addendum")
  end

  it "o job leva só ids" do
    finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1, full_name: "Nome MARCADOR"))
    expect(enqueued.to_json).not_to include("MARCADOR", "Diabetes")
  end
end
```

```ruby
# spec/requests/production_consultation_spec.rb
require "rails_helper"

# Contratos §6: correction_pending aparece na Produção e não conta como
# pendente; "não geradas" e "gerar de novo" valem para a consulta; recusada de
# consulta é regerada da origem.
RSpec.describe "Produção — fichas da consulta", type: :request do
  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }
  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:admin) do
    staff_with("prod-adm@cidade.gov.br", "municipal_admin").tap do |u|
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
    end
  end
  def body = JSON.parse(response.body)

  before do
    clinical_city!
    ledi_ready!(city, pec_url: "https://pec.a.test", record_mode: "record")
    ciap2_release!; cid10_release!; sigtap_release!
    allow(Ledi::DeliverJob).to receive(:perform_later)
  end

  it "correction_pending na lista, fora da contagem de pendentes" do
    exportable_unit!(unit, doctor)
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    Current.set(city: city) { Ledi::ConsultationFicha.generate(consultation) }
    accepted = LediOutboxEntry.sole.tap(&:accept!)
    Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "conduta acrescentada", text: "x",
                                    changes: { "conducts" => [ 1, 9 ] })
    Current.set(city: city) { Ledi::ConsultationFicha.refresh!(consultation.reload) }
    sign_in_as(admin)
    get "/production", params: { competence: accepted.competence }
    expect(body["fichas"].map { |f| f["status"] }).to match_array(%w[accepted correction_pending])
    expect(body["counts"]).to include("pending" => 0, "accepted" => 1)
  end

  it "não gerada da consulta lista o atendimento; gerar de novo resolve; recusada é regerada" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    Current.set(city: city) { Ledi::ConsultationFicha.generate(consultation) }
    sign_in_as(admin).update!(mfa_verified_at: Time.current)
    get "/production/generation_failures"
    item = body["items"].sole
    expect(item.values_at("source_type", "source_id", "attendance_id")).to eq([ "Consultation", consultation.id, consultation.attendance_id ])
    exportable_unit!(unit, doctor)
    json_post "/production/generation_failures/#{item['id']}/retry"
    expect(body["resolved_at"]).to be_present
    entry = LediOutboxEntry.sole
    entry.reject!([ { "field" => "cpfCidadao", "code" => "invalid" } ])
    json_post "/production/fichas/#{entry.id}/resend"
    expect(response).to have_http_status(:ok)
    expect(body.values_at("status", "replaces_outbox_id")).to eq([ "pending", entry.id ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/ledi/consultation_ficha_spec.rb spec/jobs/ledi/consultation_ficha_job_spec.rb spec/requests/production_consultation_spec.rb`
Expected: FAIL (`uninitialized constant Ledi::ConsultationFicha`).

- [ ] **Step 3: `Ledi::ConsultationFicha` e `Ledi::FichaSources`**

```ruby
# app/services/ledi/consultation_ficha.rb
# A ficha da consulta (ADR 0031; spec §6). Nasce da consulta finalizada, só com
# a exportação utilizável (a regra do módulo 18), uma vez por consulta. Sem
# identificação completa: "não gerada" com motivos de lista fechada.
# Identificação como no 18: só o CPF; nascimento e sexo do PACIENTE. Adendo
# com mudança estruturada (refresh!): pendente/falhou → regrava o conteúdo
# com o mesmo uuid; recusada → linha nova com replaces_outbox_id; em envio →
# InFlight (o job tenta de novo); aceita → uma correction_pending por aceita,
# nunca enviada (api#41; trigger ledi_outbox_correction_guard).
module Ledi
  module ConsultationFicha
    SOURCE_TYPE = "Consultation".freeze
    AlreadyResolved = Ledi::ScreeningFicha::AlreadyResolved
    class InFlight < StandardError; end

    module_function

    def exportable?(city) = Ledi::ScreeningFicha.exportable?(city)

    def generate(consultation, city: Current.city)
      return :skipped unless consultation.finalized?
      return :unusable unless exportable?(city)
      return :exists if LediOutboxEntry.exists?(source_type: SOURCE_TYPE, source_id: consultation.id)

      ficha, reasons = build(consultation)
      return record_failure!(consultation, reasons) if reasons.any?

      resolve_failure!(consultation)
      Ledi::Enqueue.call(ficha, city: city) ? :enqueued : :unusable
    end

    def refresh!(consultation, city: Current.city)
      return :unusable unless exportable?(city)

      latest = LediOutboxEntry.where(source_type: SOURCE_TYPE, source_id: consultation.id)
                              .where.not(status: "correction_pending").order(created_at: :desc, id: :desc).first
      return generate(consultation, city: city) unless latest

      ficha, reasons = build(consultation)
      return record_failure!(consultation, reasons) if reasons.any?

      ApplicationRecord.transaction do
        latest.lock!
        case latest.status
        when "sending" then raise InFlight
        when "pending", "failed"
          latest.update!(bytes: Ledi::Transport.wrap(ficha, city: city, uuid: latest.uuid))
          :rewritten
        when "rejected" then Ledi::Enqueue.call(ficha, city: city, replaces: latest) ? :regenerated : :unusable
        when "accepted" then correction!(latest, ficha, city)
        end
      end
    end

    def build(consultation)
      attendance = consultation.attendance
      unit = attendance.health_unit
      professional = consultation.professional_link.professional
      patient = consultation.patient
      ine = Ledi::ScreeningFicha.team_ine(professional, unit)
      birth = birth_date(patient)
      effective = Consultations::Effective.call(consultation)

      reasons = []
      reasons << "unit_without_cnes" if unit.cnes.blank?
      reasons << "professional_without_team" if ine.nil?
      reasons << "professional_without_cns" unless Professionals::Cns.valid?(professional.cns)
      reasons << "citizen_without_birth_date" if birth.nil?
      reasons << "citizen_without_sex" unless Citizen::SEXES.include?(patient.sex)
      unknown = effective[:problems].any? do |row|
        row.terminology == "ciap2" && !Ciap2Code.exists?(release_id: row.terminology_release_id, code: row.code)
      end
      reasons << "unknown_ciap2" if unknown
      return [ nil, reasons ] if reasons.any?

      started = consultation.started_at
      identity = Ledi::Fichas::ScreeningIdentity.new(
        cnes: unit.cnes, ine: ine, professional_cns: professional.cns, cbo: consultation.cbo_code,
        citizen_cpf: patient.cpf, birth_date: birth, sex: patient.sex, started_at: started,
        ended_at: [ consultation.finalized_at, started ].max, ibge_code: CityProfile.current&.ibge_code
      )
      care = Ledi::Fichas::IndividualCare::Care.new(
        care_type: consultation.care_type, problems: effective[:problems].map { |row| problem(row, started) },
        conducts: effective[:conducts], exams: effective[:exam_requests].map(&:sigtap_code),
        measurements: measurements(consultation, attendance)
      )
      [ Ledi::Fichas::IndividualCare.new(identity: identity, care: care, source_id: consultation.id), [] ]
    end

    def record_failure!(consultation, reasons)
      failure = LediGenerationFailure.unresolved.find_by(source_type: SOURCE_TYPE, source_id: consultation.id)
      if failure
        failure.update!(reason_codes: reasons) unless failure.reason_codes == reasons
      else
        failure = ApplicationRecord.transaction(requires_new: true) do
          LediGenerationFailure.create!(source_type: SOURCE_TYPE, source_id: consultation.id, reason_codes: reasons)
        end
        DomainEvents.publish("ledi.generation_failed", failure_id: failure.id, source_type: SOURCE_TYPE,
                                                       source_id: consultation.id)
      end
      :failed
    rescue ActiveRecord::RecordNotUnique
      :failed
    end

    def resolve_failure!(consultation)
      LediGenerationFailure.unresolved.where(source_type: SOURCE_TYPE, source_id: consultation.id)
                           .update_all(resolved_at: Time.current, updated_at: Time.current)
    end

    # "Gerar de novo" (contratos §6 do 18), como Ledi::ScreeningFicha.retry!.
    def retry!(failure, by:)
      failure.with_lock do
        raise AlreadyResolved if failure.resolved?

        consultation = Consultation.find_by(id: failure.source_id)
        outcome = consultation ? generate(consultation) : :skipped
        resolve_failure!(consultation) if outcome == :exists
        DomainEvents.publish("ledi.generation_retried", failure_id: failure.id, source_type: failure.source_type,
                                                        source_id: failure.source_id)
      end
      failure.reload
    end

    # "Reenviar" recusada: regera da consulta (uuid novo, replaces_outbox_id).
    def regenerate(entry, by:, city: Current.city)
      ApplicationRecord.transaction do
        entry.lock!
        next [ :not_rejected, nil ] unless entry.status == "rejected"
        next [ :not_rejected, nil ] if LediOutboxEntry.exists?(replaces_outbox_id: entry.id)
        next [ :export_unusable, nil ] unless exportable?(city)

        consultation = Consultation.find(entry.source_id)
        ficha, reasons = build(consultation)
        if reasons.any?
          record_failure!(consultation, reasons)
          next [ :generation_failed, nil ]
        end
        fresh = Ledi::Enqueue.call(ficha, city: city, replaces: entry)
        next [ :export_unusable, nil ] unless fresh

        resolve_failure!(consultation)
        DomainEvents.publish("ledi.ficha_resent", outbox_id: entry.id, user_id: by.id)
        [ :ok, fresh ]
      end
    end

    def correction!(accepted, ficha, city)
      existing = LediOutboxEntry.lock.find_by(replaces_outbox_id: accepted.id)
      if existing
        existing.update!(bytes: Ledi::Transport.wrap(ficha, city: city, uuid: existing.uuid))
      else
        uuid = "#{ficha.cnes}-#{SecureRandom.uuid}"
        LediOutboxEntry.create!(uuid: uuid, ficha_type: ficha.type, competence: ficha.competence, source_type: SOURCE_TYPE,
                                source_id: accepted.source_id, ledi_version: Ledi::Version::ACTIVE, status: "correction_pending",
                                next_attempt_at: Time.current, replaces_outbox_id: accepted.id,
                                bytes: Ledi::Transport.wrap(ficha, city: city, uuid: uuid))
      end
      :correction_pending
    end

    # uuidEvolucaoProblema = a linha; coSequencialEvolucao = a posição dela
    # entre as do mesmo problema; dataFimProblema nunca depois do atendimento.
    def problem(row, started)
      day = started.in_time_zone.to_date
      sequence = ConsultationProblem.where(patient_problem_id: row.patient_problem_id)
                                    .where("created_at < ? OR (created_at = ? AND id <= ?)", row.created_at, row.created_at, row.id)
                                    .count
      Ledi::Fichas::IndividualCare::Problem.new(
        uuid: row.patient_problem_id, evolution_uuid: row.id, sequence: sequence,
        ciap: row.terminology == "ciap2" ? row.code : nil, cid10: row.terminology == "cid10" ? row.code : nil,
        situation: Ledi::ConsultationMapping.situation(row.status_after), onset_on: row.onset_on,
        resolved_on: row.resolved_on && [ row.resolved_on, day ].min
      )
    end

    # Medições da consulta; sem nenhuma, as da escuta concluída do mesmo atendimento.
    def measurements(consultation, attendance)
      return consultation if consultation.vitals.any?

      screening = attendance.screening
      screening&.completed? ? screening.current_revision : nil
    end

    def birth_date(patient)
      patient.birth_date.present? ? Date.iso8601(patient.birth_date) : nil
    rescue Date::Error
      nil
    end
    private_class_method :correction!, :problem, :measurements, :birth_date
  end
end
```

```ruby
# app/services/ledi/ficha_sources.rb
# As origens de ficha que regeram da fonte (ADR 0030/0031): a Produção e o
# "Reenviar" despacham por source_type.
module Ledi
  module FichaSources
    REGISTRY = { "Screening" => "Ledi::ScreeningFicha", "Consultation" => "Ledi::ConsultationFicha" }.freeze

    module_function

    def for(source_type) = REGISTRY[source_type.to_s]&.constantize

    def attendance_ids(failures)
      ids = failures.group_by(&:source_type).transform_values { |rows| rows.map(&:source_id) }
      Screening.where(id: ids.fetch("Screening", [])).pluck(:id, :attendance_id).to_h
               .merge(Consultation.where(id: ids.fetch("Consultation", [])).pluck(:id, :attendance_id).to_h)
    end
  end
end
```

- [ ] **Step 4: O job e os ganchos**

```ruby
# app/jobs/ledi/consultation_ficha_job.rb
# Ficha da consulta (ADR 0031; spec §6): na finalização (generate) e depois de
# adendo com mudança estruturada (refresh!). Só ids; a decisão é relida na
# hora em que roda. Ficha em envio: tenta de novo em 1 minuto.
module Ledi
  class ConsultationFichaJob < ApplicationJob
    include CityScopedJob
    queue_as :default

    retry_on Ledi::ConsultationFicha::InFlight, wait: 1.minute, attempts: 10

    def self.enqueue_for(consultation, reason: "finalized")
      perform_later(city_slug: Current.city.slug, consultation_id: consultation.id, reason: reason)
    end

    def perform(city_slug:, consultation_id:, reason: "finalized")
      with_city(city_slug) do
        consultation = Consultation.find_by(id: consultation_id)
        if consultation
          reason == "addendum" ? Ledi::ConsultationFicha.refresh!(consultation, city: Current.city)
                               : Ledi::ConsultationFicha.generate(consultation, city: Current.city)
        end
      end
    end
  end
end
```

Em `app/commands/consultations/finalize.rb`, logo depois de `DomainEvents.publish("consultation.finalized", …)`:

```ruby
        # ADR 0031 (spec §6): a ficha nasce na finalização (o job relê tudo).
        Ledi::ConsultationFichaJob.enqueue_for(consultation)
```

Em `app/commands/consultations/add_addendum.rb`, logo depois de `DomainEvents.publish("consultation.addendum_added", …)`:

```ruby
        # ADR 0031 (spec §6): mudança estruturada regera (ou vira correção pendente).
        Ledi::ConsultationFichaJob.enqueue_for(consultation, reason: "addendum") if plan.payload[:stored].any?
```

Em `app/commands/ledi/resend.rb`, troque a primeira linha de `call` e o `regenerate`:

```ruby
    def call(entry:, by:)
      return regenerate(entry, by) if Ledi::FichaSources.for(entry.source_type)
```

```ruby
    # ADR 0030/0031: ficha de escuta ou de consulta não reaproveita o conteúdo
    # antigo — é regerada da origem (linha nova, outro uuid, replaces_outbox_id).
    def regenerate(entry, by)
      status, fresh = Ledi::FichaSources.for(entry.source_type).regenerate(entry, by: by)
```
(o `case` que segue fica igual).

Em `app/controllers/production_controller.rb`:

```ruby
  def generation_failures
    resolved = optional_scalar_param(:resolved).to_s
    scope = resolved == "true" ? LediGenerationFailure.where.not(resolved_at: nil) : LediGenerationFailure.unresolved
    failures = scope.order(created_at: :desc, id: :desc).limit(FAILURES_LIMIT).to_a
    attendances = Ledi::FichaSources.attendance_ids(failures)
    render json: { items: failures.map { |f| failure_json(f, attendances[f.source_id]) } }
  end

  def retry_generation
    return require_step_up! unless reauthenticated_recently?

    failure = LediGenerationFailure.find_by(id: params[:id])
    return render(json: { error: "not_found" }, status: :not_found) unless failure

    failure = Ledi::FichaSources.for(failure.source_type).retry!(failure, by: Current.user)
    render json: failure_json(failure, Ledi::FichaSources.attendance_ids([ failure ])[failure.source_id])
  rescue Ledi::ScreeningFicha::AlreadyResolved
    render json: { error: "already_resolved" }, status: :conflict
  end
```

(`Ledi::ConsultationFicha::AlreadyResolved` é a mesma classe: o `rescue` cobre as duas origens.)

- [ ] **Step 5: Rode e veja passar (e a produção e o 18 de sempre)**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/services/ledi spec/jobs/ledi spec/commands/ledi spec/requests/production_consultation_spec.rb spec/requests/production_spec.rb spec/requests/production_generation_failures_spec.rb spec/commands/consultations`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add app/services/ledi/consultation_ficha.rb app/services/ledi/ficha_sources.rb app/jobs/ledi/consultation_ficha_job.rb app/commands/consultations/finalize.rb app/commands/consultations/add_addendum.rb app/commands/ledi/resend.rb app/controllers/production_controller.rb spec/services/ledi/consultation_ficha_spec.rb spec/jobs/ledi/consultation_ficha_job_spec.rb spec/requests/production_consultation_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "feat: generate the consultation ficha, regenerate it on addenda and hold accepted corrections

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 4 — Corrida, invariantes, semente e fechamento (F-19.1 a F-19.7)

### Task 17: Corrida real — dois pares do mesmo CPF criam um paciente só

**Files:**
- Test: `spec/commands/patients/resolve_concurrency_spec.rb`

**Interfaces:**
- Consumes: `Patients::Resolve` (Task 5), `purge_committed_rows(ids)` (chave `:admin`), `wait_for_lock_wait`.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/commands/patients/resolve_concurrency_spec.rb
require "rails_helper"

# ADR 0031 (spec §8): duas consultas do mesmo CPF ao mesmo tempo, por pares
# diferentes (o telefone da família e o da pessoa), criam UM paciente — o lock
# por CPF segura a segunda até a primeira commitar. Threads reais contra
# TEST_CITY_A (sem fixture transacional), como screening_concurrency_spec.
RSpec.describe "Paciente sob disputa" do
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
        cpf = CampaignHistory.cpf_for("paciente-#{tag}")
        ids[:citizens] = %w[1 2].map do |suffix|
          citizen = Citizen.create!(cpf: cpf, phone: "+55419#{tag.to_i(16).to_s[0, 7].rjust(7, '1')}#{suffix}",
                                    birth_date: "1980-05-10", sex: "female", profile_source: "verified",
                                    full_name: "Maria Aparecida da Silva")
          CitizenVerification.create!(citizen: citizen, verified_by_user: admin, verified_at: Time.current)
          citizen.update!(verification_level: "verified")
          citizen.id
        end
      end
    end
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
        patient_ids = Citizen.where(id: ids[:citizens]).pluck(:patient_id).compact.uniq
        DomainEvent.where("payload->>'patient_id' IN (?)", patient_ids.presence || [ "" ]).delete_all
        CitizenVerification.where(citizen_id: ids[:citizens]).delete_all
        Citizen.where(id: ids[:citizens]).delete_all
        PatientProfileDivergence.where(patient_id: patient_ids).delete_all
        Patient.where(id: patient_ids).delete_all
      end
    end
    purge_committed_rows(ids.slice(:admin))
  end

  def in_city(&) = CityConnection.with(TEST_CITY_A) { Current.set(city: TEST_CITY_A, &) }

  it "a segunda espera o lock do CPF e liga ao mesmo paciente" do
    holder = nil
    holding = Queue.new
    original = DomainEvents.method(:publish)
    allow(DomainEvents).to receive(:publish) do |*args, **kwargs, &blk|
      result = original.call(*args, **kwargs, &blk)
      if Thread.current == holder && args.first == "patient.created"
        holding << true
        release.pop(timeout: 10) or raise "timeout esperando release"
      end
      result
    end

    first = Queue.new
    go = Queue.new
    threads << (holder = Thread.new do
      go.pop(timeout: 5)
      first << in_city { Patients::Resolve.call(Citizen.find(ids[:citizens][0])) }
    end)
    go << true
    holding.pop(timeout: 5) or raise "a primeira não segurou o lock"

    second = Queue.new
    threads << other = Thread.new { second << in_city { Patients::Resolve.call(Citizen.find(ids[:citizens][1])) } }
    expect(wait_for_lock_wait).to be(true)

    release << true
    expect(holder.join(5)).to be(holder)
    expect(other.join(5)).to be(other)
    a = first.pop(timeout: 1)
    b = second.pop(timeout: 1)
    expect([ a.payload[:created], b.payload[:created] ]).to eq([ true, false ])
    expect(b.payload[:patient].id).to eq(a.payload[:patient].id)
    in_city { expect(Citizen.where(id: ids[:citizens]).distinct.pluck(:patient_id)).to eq([ a.payload[:patient].id ]) }
  end
end
```

- [ ] **Step 2: Rode e confira a mutação**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/commands/patients/resolve_concurrency_spec.rb`
Expected: PASS. Mutação: tire o `pg_advisory_xact_lock` de `Patients::Resolve` — a segunda não espera (`wait_for_lock_wait` falso) e bate no índice único de `patients.cpf` (`RecordNotUnique`): vermelho. Desfaça.

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add spec/commands/patients/resolve_concurrency_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "test: race two pairs of the same CPF into a single patient

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 18: Suíte de invariantes do ADR 0031

**Files:**
- Create: `spec/invariants/clinical_record_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 3–17; `ledi_ready!`, `stub_pec!`, `FakePec` (módulo 16), `capture_log` (Task 3).

- [ ] **Step 1: Escreva a spec (cada bloco diz a mutação que precisa deixá-lo vermelho)**

```ruby
# spec/invariants/clinical_record_invariants_spec.rb
require "rails_helper"

# Módulo 19, critério de fechamento (ADR 0031, "Invariantes"). Cada bloco tem
# a mutação que precisa deixá-lo vermelho.
RSpec.describe "Invariantes do prontuário (ADR 0031)", type: :request do
  before do
    clinical_city!
    ciap2_release!; cid10_release!; sigtap_release!
    allow(Ledi::DeliverJob).to receive(:perform_later)
  end

  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }
  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:marker) { "MARCADOR#{SecureRandom.hex(4)}" }
  def body = JSON.parse(response.body)
  def attempt(&) = ApplicationRecord.transaction(requires_new: true, &)

  # Mutação: tirar o ramo `OLD.status = 'finalized'` de rota_consultation_guard.
  it "consulta finalizada não muda; correção só por adendo" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    expect { attempt { consultation.update_columns(plan: "reescrito", care_type: 6) } }
      .to raise_error(ActiveRecord::StatementInvalid, /finalized consultation never changes/)
    expect { attempt { ConsultationConduct.create!(consultation: consultation, code: 9, action: "add") } }
      .to raise_error(ActiveRecord::StatementInvalid, /only come with an addendum/)
    expect(Consultations::SaveDraft.call(consultation: consultation, params: { "plan" => "x" }, by: doctor).reason).to eq(:not_draft)
  end

  # Mutação: tirar o NOT EXISTS de rota_patient_problem_guard, ou gravar o
  # problema antes do evento em Patients::ApplyProblemEvent.
  it "patient_problems só muda por evento ligado a consulta ou adendo" do
    finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    problem = PatientProblem.sole
    expect { attempt { problem.update!(status: "resolved", resolved_on: Time.zone.today) } }
      .to raise_error(ActiveRecord::StatementInvalid, /changes only through an event/)
    expect(PatientProblemEvent.where(patient_problem_id: problem.id).pluck(:consultation_id, :addendum_id))
      .to all(satisfy { |consultation_id, addendum_id| consultation_id.present? ^ addendum_id.present? })
  end

  # Mutação: tirar ClinicalRecord::Trail.viewed! de qualquer ação de leitura.
  it "nenhuma leitura de prontuário sem trilha" do
    citizen = verified_citizen!(1)
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    attendance = consulting_attendance!(unit, citizen: citizen.reload, doctor: doctor)
    nurse = doctor!(unit, cbo: "223505")
    now = Time.current
    ClinicalRecordOpening.create!(patient: consultation.patient, user: nurse, reason_code: "case_review", created_at: now,
                                  expires_at: now + 30.minutes)
    reads = [ [ doctor, "/attendance/attendances/#{attendance.id}/record" ], [ doctor, "/attendance/consultations/#{consultation.id}" ],
              [ doctor, "/attendance/consultations/#{consultation.id}/print" ], [ nurse, "/clinical_record/patients/#{consultation.patient_id}" ] ]
    reads.each do |user, path|
      sign_in_as(user)
      expect { get path }.to change { DomainEvent.where(name: "clinical_record.viewed").count }.by(1), path
      expect(response).to have_http_status(:ok), path
    end
  end

  # Mutação: tirar :subjective/:plan/:text/:full_name de filter_parameters, pôr
  # texto ou nome em payload de evento ou argumento de job, ou devolver o texto
  # recebido numa mensagem de erro.
  it "nenhum texto clínico nem nome em evento, log, job, Analytics ou mensagem de erro" do
    citizen = verified_citizen!(1, full_name: "#{marker} Nome")
    attendance = consulting_attendance!(unit, citizen: citizen, doctor: doctor)
    log = capture_log do
      sign_in_as(doctor)
      json_post "/attendance/attendances/#{attendance.id}/consultation"
      id = body["id"]
      patch "/attendance/consultations/#{id}", params: draft_body(subjective: "S #{marker}", plan: "P #{marker}").to_json,
                                               headers: { "CONTENT_TYPE" => "application/json" }
      patch "/attendance/consultations/#{id}", params: { "objective" => "#{marker} " * 3000 }.to_json,
                                               headers: { "CONTENT_TYPE" => "application/json" }
      expect(response).to have_http_status(:unprocessable_entity)
      expect(response.body).not_to include(marker)
      json_post "/attendance/consultations/#{id}/finalize", outcome: { outcome: "discharged" }
      json_post "/attendance/consultations/#{id}/addenda", reason: "motivo #{marker}", text: "adendo #{marker}"
      expect(response).to have_http_status(:created)
    end
    [ log, DomainEvent.pluck(:payload).to_json, ActiveJob::Base.queue_adapter.enqueued_jobs.to_json ].each do |text|
      expect(text).not_to include(marker)
    end
    analytics = Dir[Rails.root.join("app/services/analytics/**/*.rb")].map { |f| File.read(f) }.join
    expect(analytics).not_to match(/consultations|consultation_addenda|clinical_record_openings|full_name|social_name|mother_name/)
  end

  # Mutação: trocar require_professional por require_attendance_staff em
  # qualquer controller do prontuário.
  it "a recepção nunca lê o prontuário" do
    citizen = verified_citizen!(1, full_name: "#{marker} Nome")
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen, subjective: "S #{marker}")
    attendance = consulting_attendance!(unit, citizen: citizen.reload, doctor: doctor)
    sign_in_as(reception!)
    [ "/attendance/attendances/#{attendance.id}/record", "/attendance/consultations/#{consultation.id}",
      "/attendance/consultations/#{consultation.id}/print", "/clinical_record/patients/#{consultation.patient_id}" ].each do |path|
      get path
      expect(response).to have_http_status(:forbidden), path
      expect(response.body).not_to include(marker)
    end
  end

  # Mutação: tirar a checagem de verification_level de Patients::Resolve, ou o
  # NEW.verification_level de rota_citizen_patient_link_guard.
  it "consulta só para par validado; par declarado nunca ligado a paciente" do
    declared = screening_citizen!(9)
    attendance = consulting_attendance!(unit, citizen: declared, doctor: doctor)
    expect(Consultations::Start.call(attendance: attendance, by: doctor).reason).to eq(:citizen_not_verified)
    patient = Patients::Resolve.call(verified_citizen!(2)).payload[:patient]
    expect { attempt { declared.update_columns(patient_id: patient.id) } }
      .to raise_error(ActiveRecord::StatementInvalid, /only a verified pair links/)
    expect(Citizen.where.not(patient_id: nil).where(verification_level: "declared")).to be_empty
  end

  # Mutação: tirar ledi_outbox_correction_guard, ou fazer correction! gravar
  # status "pending".
  it "nenhuma correção de ficha aceita é enviada enquanto o reenvio após aceite não for confirmado" do
    stub_pec!
    allow(Ledi::Observations).to receive(:duplicate_marker).and_return(nil)
    ledi_ready!(city, pec_url: "https://pec.a.test", record_mode: "record")
    exportable_unit!(unit, doctor)
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    Current.set(city: city) { Ledi::ConsultationFicha.generate(consultation) }
    accepted = LediOutboxEntry.sole.tap(&:accept!)
    Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "conduta acrescentada", text: "x",
                                    changes: { "conducts" => [ 1, 9 ] })
    Current.set(city: city) { Ledi::ConsultationFicha.refresh!(consultation.reload) }
    allow(Ledi::DeliverJob).to receive(:perform_later).and_call_original
    CityConnection.with(city) { Ledi::DeliverJob.perform_now }
    expect(FakePec.for("https://pec.a.test").deliveries).to be_empty
    expect(LediOutboxEntry.find_by!(replaces_outbox_id: accepted.id).status).to eq("correction_pending")
    expect { attempt { LediOutboxEntry.find_by!(replaces_outbox_id: accepted.id).update_columns(status: "pending") } }
      .to raise_error(ActiveRecord::StatementInvalid, /stays correction_pending/)
  end

  # ADR 0026 + 0031: paciente com consulta é retido na exclusão; a revogação
  # não afeta o prontuário. Mutação: tirar Attendance de RequestErasure.attended?.
  it "paciente com consulta é retido; revogação não desliga" do
    citizen = verified_citizen!(1)
    finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    request = Citizens::RequestErasure.call(cpf: citizen.cpf, document_checked: true, by: verifier!).payload[:request]
    expect(request.status).to eq("retained")
    Citizens::RevokeVerification.call(verification: citizen.reload.active_verification, reason: "documento rasurado",
                                      by: staff_with("adm-#{SecureRandom.hex(3)}@x.gov.br", "municipal_admin"))
    expect(citizen.reload.patient_id).to be_present
  end
end
```

- [ ] **Step 2: Rode e confira as mutações**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/invariants/clinical_record_invariants_spec.rb`
Expected: PASS. Aplique cada mutação descrita no comentário (uma por vez, sem commitar), rode o bloco e veja vermelho; desfaça.

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add spec/invariants/clinical_record_invariants_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "test: pin the ADR 0031 invariants for the primary care clinical record

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 19: Semente de dev (spec §9)

**Files:**
- Create: `lib/clinical_record_crew.rb`
- Modify: `db/seeds.rb`
- Test: `spec/lib/clinical_record_crew_spec.rb`

**Interfaces:**
- Consumes: `Platform::Features.set!`, `Terminology::Import`, `Citizens::{RegisterPerson,IssueVerificationCode,Verify,StartConversation,SubmitAnswer,IssueCheckInCode}`, `Attendances::{CheckIn,Call}`, `Consultations::{Start,SaveDraft,Finalize,AddAddendum}`, `ProfessionalCrew::UNITS`, `ScreeningCrew` (convenção de telefone).
- Produces: `ClinicalRecordCrew::PHONE_PREFIX == "95555"`, `::CIAP2_SAMPLE`, `::CID10_SAMPLE`, `.seed_platform! -> { ciap2: bool, cid10: bool }` (importa recortes só se não houver release ativa), `.seed_current_city(slug:, ddd:) -> { switch: String, patient: String, consultation: String }`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/lib/clinical_record_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/clinical_record_crew").to_s

# Spec §9: interruptor e modo record em Curitiba (dev); cidadã validada com
# nome completo e social; consulta finalizada com T90 ativo e um adendo. O
# teste fixa o que quebra em silêncio: o prefixo de telefone, os recortes de
# terminologia e o corpo do rascunho; a semente inteira roda na Task 20.
RSpec.describe ClinicalRecordCrew do
  it "o prefixo do telefone não aparece em nenhum outro lib/*_crew.rb" do
    others = Dir[Rails.root.join("lib/*_crew.rb")].reject { |f| f.end_with?("/clinical_record_crew.rb") }
    expect(others.select { |f| File.read(f).include?(described_class::PHONE_PREFIX) }).to eq([])
  end

  it "os recortes de CIAP-2 e CID-10 importam pelo caminho real e trazem T90 e E11.9" do
    result = described_class.seed_platform!
    expect(result.values).to all(be(true).or(be(false)))
    expect(ClinicalTerms.find("ciap2", "T90")).to be_present
    expect(ClinicalTerms.find("cid10", "E11.9")).to be_present
    expect(described_class.seed_platform!).to eq(ciap2: false, cid10: false)
  end

  it "o rascunho da semente passa na validação dos itens" do
    Current.city = TEST_CITY_A
    described_class.seed_platform!
    patient = Patient.create!(cpf: verified_citizen!(1).cpf, birth_date: "1979-04-12", sex: "male")
    expect(Consultations::ItemsInput.call(described_class::DRAFT, patient: patient, cbo: "225125")).to be_ok
  ensure
    Current.reset
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/lib/clinical_record_crew_spec.rb`
Expected: FAIL (`cannot load such file … lib/clinical_record_crew`).

- [ ] **Step 3: A semente**

Antes de escrever, confira o formato dos arquivos que o `Terminology::Import` lê para CIAP-2 e CID-10 (cabeçalho e separador) em `spec/services/terminology/import_spec.rb` ou nos fixtures dele; os recortes abaixo seguem `Ciap2Reader` (`ciap2.csv`, `CODIGO;TITULO`, UTF-8) e `Cid10Reader` (`CID-10-CATEGORIAS.CSV` com `CAT;DESCRICAO` e `CID-10-SUBCATEGORIAS.CSV` com `SUBCAT;DESCRICAO;RESTRSEXO`, ISO-8859-1).

```ruby
# lib/clinical_record_crew.rb
require "tmpdir"

# Semente de dev do módulo 19 (spec 2026-10-07 §9). Dev é fictício mas imita o
# real: recortes de CIAP-2 e CID-10 (só se não houver release ativa); em
# Curitiba, record_mode = record e o interruptor clinical_record LIGADOS pelo
# mantenedor de dev (a spec manda — as sementes 16–18 não ligam nada); uma
# cidadã validada no balcão com nome completo e social; atendida pela médica
# da semente (profissional@, 225125) na UBS Jardim das Flores com diabetes
# (CIAP-2 T90) ativo, consulta finalizada e um adendo; e um segundo
# atendimento dela aguardando na mesma UBS, para a enfermeira (enfermeira@,
# 223505) ler o prontuário em contexto. Idempotente. Roda depois do ScreeningCrew.
class ClinicalRecordCrew
  UNIT = "UBS Jardim das Flores"
  PHONE_PREFIX = "95555"
  CIAP2_SAMPLE = { "T90" => "Diabetes não insulino-dependente", "K86" => "Hipertensão sem complicações",
                   "R05" => "Tosse", "A03" => "Febre", "N01" => "Cefaleia" }.freeze
  CID10_SAMPLE = { "E11" => [ "Diabetes mellitus não-insulino-dependente", nil ],
                   "E119" => [ "Diabetes mellitus não-insulino-dependente - sem complicações", nil ],
                   "I10" => [ "Hipertensão essencial (primária)", nil ] }.freeze
  NAMES = { full_name: "Luiz Fernando Alves Moreira", social_name: "Luíza Alves", mother_name: "Rosana Alves" }.freeze
  PROFILE = { birth_date: "1979-04-12", sex: "male", gender_identity: "trans_woman" }.freeze
  DRAFT = {
    "subjective" => "Sede e poliúria há três meses; nega perda de peso.", "objective" => "Bom estado geral, eupneica.",
    "assessment" => "Diabetes mellitus tipo 2 recém-diagnosticado.", "plan" => "Metformina 500 mg 2x/dia; orientação alimentar; retorno em 30 dias.",
    "vitals" => { "systolic" => 132, "diastolic" => 84, "weight_kg" => "88.4", "height_cm" => 168 },
    "care_type" => 5,
    "evaluated_problems" => [ { "terminology" => "ciap2", "code" => "T90", "action" => "add",
                                "onset_on" => "2026-07-01", "onset_precision" => "month" } ],
    "conducts" => [ 1 ], "exam_requests" => []
  }.freeze

  class << self
    def seed_platform!
      { ciap2: import_unless_active("ciap2") { |dir| write_ciap2(dir) },
        cid10: import_unless_active("cid10") { |dir| write_cid10(dir) } }
    end

    def seed_current_city(slug:, ddd:)
      switch = slug == "curitiba" ? enable!(Current.city) : "desligado (só Curitiba liga)"
      unit = HealthUnit.find_by!(name: UNIT)
      citizen = ensure_citizen(slug, ddd)
      patient = citizen.patient || Patient.find_by(cpf: citizen.cpf)
      consultation = patient&.consultations&.finalized_consultations&.first
      consultation ||= (switch == "ligado" ? consult!(slug, unit, citizen) : nil)
      waiting!(slug, unit, citizen) if consultation
      { switch: switch, patient: citizen.display_name.to_s, consultation: consultation ? "finalizada com adendo" : "sem consulta" }
    end

    private

    def import_unless_active(kind)
      return false if TerminologyRelease.active.exists?(kind: kind)

      Dir.mktmpdir do |dir|
        yield Pathname(dir)
        result = Terminology::Import.call(kind: kind, version: "dev-#{Time.zone.today.year}", path: dir, by: "db:seed")
        raise "semente do prontuário: #{kind} recusada (#{result.reason} #{result.message})" if result.failure?
      end
      true
    end

    def write_ciap2(dir)
      dir.join("ciap2.csv").write("CODIGO;TITULO\n" + CIAP2_SAMPLE.map { |code, title| "#{code};#{title}" }.join("\n") + "\n")
    end

    def write_cid10(dir)
      categories = CID10_SAMPLE.select { |code, _| code.length == 3 }
      subcategories = CID10_SAMPLE.reject { |code, _| code.length == 3 }
      dir.join("CID-10-CATEGORIAS.CSV").binwrite(("CAT;DESCRICAO\n" + categories.map { |c, (d, _)| "#{c};#{d}" }.join("\n") + "\n").encode("ISO-8859-1"))
      dir.join("CID-10-SUBCATEGORIAS.CSV").binwrite(("SUBCAT;DESCRICAO;RESTRSEXO\n" +
        subcategories.map { |c, (d, s)| "#{c};#{d};#{s}" }.join("\n") + "\n").encode("ISO-8859-1"))
    end

    def enable!(city)
      maintainer = Maintainer.find_by(email_address: "dev@local")
      unless maintainer
        warn "[seeds] prontuário: sem mantenedor de dev (dev@local) — interruptor não ligado"
        return "desligado (sem mantenedor)"
      end
      city.update!(record_mode: "record") unless city.record_mode == "record"
      Platform::Features.set!(city: city, key: "clinical_record", enabled: true, maintainer: maintainer)
      "ligado"
    end

    def ensure_citizen(slug, ddd)
      registered = Citizens::RegisterPerson.call(phone: format("+55%s#{PHONE_PREFIX}%04d", ddd, 1),
                                                 cpf: ScreeningCrew.send(:cpf_for, "#{slug}:clinical_record:1"),
                                                 profile: PROFILE)
      raise "semente do prontuário: cidadã recusada (#{registered.reason})" if registered.failure?

      citizen = registered.payload[:citizen]
      return citizen if citizen.verification_level_verified?

      reception = User.find_by!(email_address: "recepcao@#{slug}.demo")
      code = Citizens::IssueVerificationCode.call(citizen: citizen).payload.fetch(:code)
      result = Citizens::Verify.call(cpf: citizen.cpf, code: code, document_checked: true, by: reception,
                                     birth_date: PROFILE[:birth_date], sex: PROFILE[:sex],
                                     gender_identity: PROFILE[:gender_identity], **NAMES)
      raise "semente do prontuário: validação recusada (#{result.reason})" if result.failure?

      citizen.reload
    end

    def checked_in!(slug, unit, citizen, step_key)
      started = Citizens::StartConversation.call(citizen: citizen, consent_version: Consents.current_version,
                                                 session_id: "seed-clinical-record")
      raise "semente do prontuário: conversa recusada (#{started.reason})" if started.failure?

      %w[true true].each_with_index do |answer, step|
        Citizens::SubmitAnswer.call(conversation: started.payload[:conversation], answer: answer,
                                    idempotency_key: "seed-clinical-#{slug}-#{step_key}-#{Time.zone.today}-#{step}")
      end
      triage = started.payload[:triage].reload
      code = Citizens::IssueCheckInCode.call(citizen: citizen, triage: triage).payload.fetch(:code)
      reception = User.find_by!(email_address: "recepcao@#{slug}.demo")
      result = Attendances::CheckIn.call(cpf: citizen.cpf, code: code, health_unit_id: unit.id, document_checked: false,
                                         by: reception)
      raise "semente do prontuário: check-in recusado (#{result.reason})" if result.failure?

      result.payload[:attendance]
    end

    def consult!(slug, unit, citizen)
      doctor = User.find_by!(email_address: "profissional@#{slug}.demo")
      attendance = checked_in!(slug, unit, citizen, "consulta")
      called = Attendances::Call.call(attendance: attendance, health_unit_id: unit.id, by: doctor)
      raise "semente do prontuário: chamada recusada (#{called.reason})" if called.failure?

      started = Consultations::Start.call(attendance: attendance.reload, by: doctor)
      raise "semente do prontuário: consulta recusada (#{started.reason})" if started.failure?

      consultation = started.payload[:consultation]
      saved = Consultations::SaveDraft.call(consultation: consultation, params: DRAFT, by: doctor)
      raise "semente do prontuário: rascunho recusado (#{saved.reason} #{saved.details})" if saved.failure?

      finalized = Consultations::Finalize.call(consultation: consultation.reload, outcome_params: { "outcome" => "return" }, by: doctor)
      raise "semente do prontuário: finalização recusada (#{finalized.reason} #{finalized.details})" if finalized.failure?

      addendum = Consultations::AddAddendum.call(consultation: consultation.reload, by: doctor,
                                                 reason: "resultado de glicemia trazido pela paciente",
                                                 text: "Glicemia de jejum de 168 mg/dL (laboratório externo), confirma o diagnóstico.")
      raise "semente do prontuário: adendo recusado (#{addendum.reason})" if addendum.failure?

      consultation.reload
    end

    # Um atendimento dela aguardando hoje, para a leitura em contexto da enfermeira.
    def waiting!(slug, unit, citizen)
      return if Attendance.waiting.where(citizen: citizen, health_unit: unit).exists?

      checked_in!(slug, unit, citizen, "espera")
    end
  end
end
```

(`ScreeningCrew.cpf_for` é privado na classe do 18: o `send` reusa o mesmo gerador de CPF válido; se o 18 o tiver tornado público, troque por chamada direta.)

Em `db/seeds.rb`, depois de `require Rails.root.join("lib/screening_crew").to_s`, `require Rails.root.join("lib/clinical_record_crew").to_s`; depois de `sigtap = RecordModeCrew.seed_platform!` e do `puts` dele:

```ruby
  # ── Terminologias da consulta (módulo 19, spec 2026-10-07 §9) ─────────────────
  terms = ClinicalRecordCrew.seed_platform!
  puts "[seeds] CIAP-2/CID-10 #{terms.map { |kind, imported| "#{kind} #{imported ? 'importada (recorte de dev)' : 'já ativa'}" }.join('; ')}"
```

e depois do bloco do acolhimento:

```ruby
        # ── Prontuário (módulo 19, spec 2026-10-07 §9) ────────────────────────
        # Depois do acolhimento: usa a UBS, a médica, a enfermeira e a recepção.
        record = ClinicalRecordCrew.seed_current_city(slug: slug, ddd: ddd)
        puts "[seeds] prontuário ... interruptor #{record[:switch]}; #{record[:patient]}: #{record[:consultation]}"
```

- [ ] **Step 4: Rode a spec**

Run: `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec spec/lib/clinical_record_crew_spec.rb`
Expected: PASS. Sintaxe: `docker compose exec -T -w /rails/.claude/mod19 api ruby -c db/seeds.rb lib/clinical_record_crew.rb` → `Syntax OK`. A semente de verdade roda no banco de dev só na Task 20, com autorização do usuário.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19 add lib/clinical_record_crew.rb db/seeds.rb spec/lib/clinical_record_crew_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19 commit -m "chore: seed the clinical record switch, a verified patient and a finalized consultation for dev

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 20: Revisão final, suíte completa e api na porta 3036

- [ ] **Step 1:** Varredura de vazamento: `grep -rn "subjective\|objective\|assessment\|\.plan\b\|full_name\|social_name\|mother_name\|reason_note" apps/api/.claude/mod19/app/services/analytics apps/api/.claude/mod19/app/events apps/api/.claude/mod19/config/initializers/domain_events.rb` — nada. `grep -rn "DomainEvents.publish(\"clinical_record\|DomainEvents.publish(\"consultation\|DomainEvents.publish(\"patient" -A2 apps/api/.claude/mod19/app` — só ids, `kind`, `access`, `reason_code` (lembre: `publish` é multilinha — confira cada chamada inteira).
- [ ] **Step 2:** Bancos de teste refeitos (Ambiente) e suíte completa com o worker parado:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod19 api bundle exec rspec
  docker compose start worker
  ```
  Expected: verde. Falha em spec antiga que fixa chaves (item da fila, cartão do balcão/check-in, catálogo de interruptores, status da fila LEDI) → acrescente `display_name`, `names`, `verification_id`, `clinical_record`, `correction_pending` (é o contrato); nunca afrouxe asserção de texto livre.
- [ ] **Step 3:** `docker compose exec -T -w /rails/.claude/mod19 api bundle exec rubocop <arquivos tocados>`; corrija só o que a regra do projeto aponta.
- [ ] **Step 4:** Para a prova no navegador e o plano do dashboard, suba o api do worktree na porta **3036**, sem derrubar o principal:

  ```bash
  docker compose exec -d -w /rails/.claude/mod19 api bin/rails server -b 0.0.0.0 -p 3036 -P tmp/pids/server-mod19.pid
  ```
  O Vite do worktree do dashboard aponta `VITE_API_PROXY_TARGET` para `http://api:3036` (com o prefixo novo `/clinical_record` no proxy). **Antes**, com autorização do usuário: `bundle install` na imagem de dev (Prawn), `city:migrate:all` (as cidades de dev ganham as tabelas do prontuário) e `db:seed` (Task 19 — liga o interruptor e o modo `record` em Curitiba). Roteiro: a médica (`profissional@curitiba.demo`) chama da fila a cidadã da semente (nome de exibição "Luíza Alves"), vê o painel com T90 ativo e a consulta anterior com o adendo, abre a consulta nova, salva, finaliza com retorno; imprime a anterior; a enfermeira (`enfermeira@curitiba.demo`) lê o prontuário em contexto pela fila de espera e, para outro CPF, abre com motivo e step-up; o admin vê o relatório de aberturas. Login, OTP e TOTP são do usuário (senhas e TOTP da semente de dev podem ser mostrados no chat se ele pedir).
- [ ] **Step 5:** **Pare.** Merge, push, board e docs (página de status do módulo) só com autorização explícita do usuário, uma etapa de cada vez. Ordem: api → dashboard. Rollout por cidade: publicar a imagem nova (com Prawn) e rodar `city:migrate:all` dela antes de cortar tráfego (as três migrações são irreversíveis); tudo nasce desligado — `record_mode = record` e `clinical_record` por cidade, pelo maintenance. Durante a janela entre o api e o dashboard novos, o balcão antigo continua validando sem nome (decisão §9 do contrato): esses pares completam o nome no check-in. Ao voltar o checkout para a main: `DROP DATABASE` dos dois bancos de teste e `city:test_databases`; derrube o servidor da 3036 (`kill $(cat tmp/pids/server-mod19.pid)` no container).

---
## Self-review (feito ao escrever o plano)

**Cobertura da spec:**
- §3 paciente, nomes, lista → Tasks 3 (dados, triggers, cifra), 4 (nomes na validação e no check-in, fila), 5 (Resolve, divergência, lock por CPF; corrida na Task 17), 6 (eventos, replay).
- §4 consulta → Tasks 2 (interruptor), 7 (dados, imutabilidade, re-cifra), 8 (CBO, terminologias, itens), 9 (Start/SaveDraft, tipo sugerido), 10 (Finalize com o Close e o pedido), 11 (adendo), 14 (impresso).
- §5 leitura e LGPD → Tasks 11 (Access, Open), 12 (trilha), 13 (rotas, recepção 403, relatório), 18 (invariantes); ADR 0026 retenção e revogação → Tasks 5 e 18.
- §6 ficha → Tasks 1 (layout com parada), 15 (ficha, `correction_pending`), 16 (geração, regeneração, Produção).
- §8 testes → cada task + Task 17 (threads) + Task 18 (invariantes). §9 semente → Task 19. §10 rollout e `VALID_RANGE` → Tasks 2 e 20.

**Placeholders:** nenhum "TBD"/"similar à Task N"; os dumps estão por extenso; as transcrições deixadas ao executor são as conferências da documentação oficial (Task 1, com o "Esperado" lido em 2026-10-07 e critério de parada) e o formato exato dos CSV de terminologia (Task 19, com o reader como referência).

**Consistência de nomes:** `Patients::{Resolve,ApplyProblemEvent,ProblemReplay}`, `Consultations::{Start,SaveDraft,Finalize,AddAddendum,ItemsInput,Effective,Json,Print,Cbos,Authorization,CareType}`, `ClinicalRecord::{Gate,Access,Open,Json,Trail}`, `ClinicalTerms`, `ClinicalTerms::SigtapExams`, `Citizens::{NameValues,NamesJson,CompleteNames}`, `Ledi::{ConsultationMapping,ConsultationFicha,FichaSources,ConsultationFichaJob}`, `Ledi::Fichas::IndividualCare`, `CityEncryption.allowing_reencryption` — conferidos entre as tasks. `Consultations::Finalize.record_problem!` é usado pelo adendo; `ConsultationsController#read_grant` pelo impresso.

**Review Focus:** as cinco linhas têm teste na task dona (Tasks 7, 14, 8/13, 10, 11/13).

## Decisões em aberto (para o usuário)

1. **CID-10 por CBO.** O layout não restringe (Task 1): hoje médico e enfermeiro informam CID-10. Se o produto quiser o comportamento do PEC (CID-10 só para médicos), é uma linha em `config/ledi/consultation_mapping.yml` (`cid10_cbos.rule: prefixes`, `prefixes: ["2251", "2252", "2253", "2231"]`) — sem código.
2. **uuid da correção após aceite.** A `correction_pending` nasce com uuid novo (o índice único de `uuid` não deixa repetir o da aceita). Quando a api#41 provar o reenvio após aceite, decide-se se a correção vai com o uuid da aceita (aí a linha troca o uuid na hora de sair) ou com o novo.

## Divergências propostas ao contrato

As decisões do coordenador (contrato §9: `already_exists` com `consultation_id`; nomes completados no check-in com `citizen.names`/`citizen.verification_id` e `verification_id` no check-in; `display_name` em todo item da fila; `changes` do adendo com `evaluated_problems`/`conducts`/`exam_requests` e `opening_id` também do autor; PATCH com todos os campos; `verifications` sem `full_name` aceito; `record.consultations` só finalizadas; `cid10_justification` com a regra de CBO; impresso PDF/JSON) já estão no plano. O que ainda falta no contrato, para o plano do dashboard:

- **D1 — Código de pré-requisito `record_mode_not_record`** no `missing` do interruptor `clinical_record` (o dashboard/maintenance precisam do rótulo, ex.: "modo de prontuário diferente de record").
- **D2 — Erros 422 novos** (com `index` nos itens): `invalid_text` (`field`), `invalid_problem`, `invalid_onset`, `cid10_sex_incompatible`, `invalid_conduct`, `invalid_exam`, `invalid_care_type` (PATCH e finalize), `text_required` e `invalid_changes` (adendo), `invalid_period` (relatório), `invalid_terminology` (busca). A finalização devolve `invalid_problem` com `index` quando um problema mudou desde o rascunho.
- **D3 — Pressão incompleta no autosave** responde `implausible_vital` com `field` do lado que falta (não `bp_incomplete`).
- **D4 — 409 `consultation_in_progress`** em `POST /attendance/attendances/:id/close` quando há consulta em rascunho no atendimento (o desfecho sai pela finalização).
- **D5 — `GET /attendance/attendances/:id/record` de par validado sem paciente**: `patient.id: null`, `problems: []`, `consultations: []`, sem trilha de prontuário.
- **D6 — Leitura fora de contexto em `/attendance`**: `GET /attendance/consultations/:id` e `/print` de consulta finalizada sem contexto nem abertura → 403 `out_of_context`; rascunho de outro autor → 403 `not_author`. Em `/clinical_record/patients/:id` o código continua `opening_required`.
- **D7 — `POST /attendance/attendances/:id/consultation`** também responde 403 `missing_role` e `missing_link` (como as outras rotas clínicas).
- **D8 — Abertura**: `POST /clinical_record/openings` responde **201**; CPF inválido também é 404 `patient_not_found` (não enumera); só profissional com vínculo de CBO permitido (403 `missing_role`/`missing_link`/`cbo_not_allowed`). Relatório: `from`/`to` `AAAA-MM-DD` no fuso da cidade, até 500 linhas, mais novas primeiro.
- **D9 — Limites e listas**: exames só SIGTAP do grupo 02 (busca e pedido), até 100; até 50 problemas e 12 condutas; tipos `1, 2, 5, 6`; condutas `1, 2, 4–12, 14` com os rótulos de "Valores fixados".
- **D10 — `cid10_allowed_for_cbo`** em `consultation_options` é calculado pelo vínculo permitido do usuário (qualquer unidade).
- **D11 — `POST /attendance/verifications/:id/names`** responde `{ citizen: { id, cpf_masked, verification_level, names } }`.
- **D12 — Forma do item avaliado**: `onset_on`/`onset_precision` só aparecem com valor; o problema da lista (`record.problems`) leva também `resolved_on`; `exam_requests[].cid10_justification` só com valor.
- **D13 — Encaminhamento na ficha só como conduta** (4, 5, 6, 7, 8 ou 10): `encaminhamentos` do MIAI exige especialidade de tabela que o projeto não tem.
- **D14 — Produção**: `correction_pending` aparece em `fichas` (com `replaces_outbox_id` apontando a aceita) e fica fora de `counts` (que continua com os cinco estados de hoje); "não geradas", "gerar de novo" e "Reenviar" valem para `source_type: "Consultation"`.
