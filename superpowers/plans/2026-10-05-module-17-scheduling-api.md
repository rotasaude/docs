# Módulo 17 — Agenda dos profissionais (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Tipos de atendimento (base da plataforma + cidade), modelos de agenda ligados ao turno, vagas calculadas com trava de sobreposição no banco, marcação em vaga e encaixe com limite, transição `legacy`, pedido de agendamento gerado pela triagem (fila ordenada e fila "sem unidade"), "Não posso nesse horário", lembrete da véspera (aviso + SMS) e as agendas da unidade e do profissional — o lado api de F-17.1 a F-17.9 (ADR 0029).

**Architecture:** Uma migração de cidade (`20261008100001`) cria `appointment_types` (com a base copiada de `config/scheduling/appointment_types.yml`), `schedule_templates`, `appointment_request_triages` e `appointment_notices`, acrescenta colunas em `appointments`, `appointment_requests`, `professional_shifts`, `professional_links` e `city_profile`, e põe a `EXCLUDE USING gist` dos horários `slot` (com verificação prévia que aborta listando ids). O cálculo de vagas é uma função pura (`Scheduling::Availability.compute`) sobre `Data` carregados por `Scheduling::Availability.for`; `Appointments::Book`/`FitIn` gravam sob a ordem de travas cidadão → horário vivo → pedido → unidade (FOR SHARE) → turno (FOR UPDATE, só no encaixe), e a EXCLUDE decide a corrida da mesma vaga. `Triages::Schedule` roda na conclusão da triagem; a fila e as agendas saem de apresentadores (`Scheduling::RequestJson`, `Scheduling::AppointmentPresenter`, `Scheduling::UnitAgenda`, `Scheduling::ProfessionalAgenda`). O lembrete é um job por cidade (`EachCityJob`) que só age das 17h às 20h locais e é idempotente por `appointments.reminded_at`.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade, `btree_gist`), RSpec, json_schemer, Solid Queue (recorrência por cidade).

**Spec:** `docs/.claude/ciclo2/superpowers/specs/2026-10-05-module-17-scheduling-design.md` e `docs/.claude/ciclo2/adr/0029.md` (leia os dois antes de começar). Contratos entre apps: `docs/.claude/ciclo2/superpowers/plans/2026-10-05-module-17-scheduling-contracts.md` — **inclusive §8 e §9, que valem sobre as seções anteriores**. Os planos do `contracts`, do dashboard e do wpda foram escritos contra ele: **não mude nomes, formatos nem códigos de erro**; as precisões que o código real obrigou estão no fim, em "Divergências propostas ao contrato". O schema novo nasce no repo `contracts` (tag `protocols-v1.5.0`, outro plano); a Task 1 copia o arquivo de lá.

## Desvios da spec (e precisões)

Onde a spec é omissa ou o código real obrigou a escolher:

1. **`tsrange`, não `tstzrange`.** `appointments.scheduled_at` e o novo `ends_at` são `timestamp without time zone` em UTC (o padrão `datetime` do Rails 8, igual a `professional_shifts`, cuja EXCLUDE já usa `tsrange`). A trava é `EXCLUDE USING gist (professional_id WITH =, tsrange(scheduled_at, ends_at) WITH &&) WHERE (booking_kind = 'slot' AND status IN ('scheduled','confirmed','checked_in'))`.
2. **A verificação de sobreposição herdada nada encontra no primeiro rollout:** todo horário existente vira `legacy` (sem profissional) na mesma migração. Ela existe (e é testada) para o caso de a coluna já ter sido preenchida por um rollout interrompido ou por SQL manual; aborta com `AddProfessionalSchedules::InheritedOverlap` listando os pares de ids.
3. **Guardas do banco que mudam.** `rota_appointment_request_guard` recusava qualquer mudança de `target_unit_id`; passa a aceitar **só** `NULL → unidade` (atribuição da fila "sem unidade") e acrescenta `origin_triage_id` às colunas imutáveis. `rota_appointment_guard` acrescenta `professional_id`, `appointment_type_key`, `ends_at`, `shift_id`, `booking_kind` e `fit_in_reason` às imutáveis ("mudança de turno ou modelo nunca move horário"). `professional_shifts.schedule_template_id` e `professional_links.default_appointment_type_key` são colunas novas, fora das listas dos guardas atuais (mudam enquanto o turno não está cancelado e o vínculo não está encerrado).
4. **Backfill de `due_on`** dos pedidos existentes = `created_at::date + 30` (data UTC). Hoje o desfecho **não** registra data indicada de retorno; todo pedido de atendimento novo nasce com `due_on = hoje (fuso da cidade) + 30`, tipo `retorno`, prioridade `routine`. O guarda dos pedidos recusa UPDATE em pedido `closed`, então a migração derruba `appointment_requests_guard` antes do backfill; o `city_triggers.sql` executado no fim recria.
5. **"Pedido aberto do mesmo tipo"** = pedido vivo (`open` ou `scheduled`) do cidadão com o mesmo `appointment_type_key`, de qualquer origem. A triagem que casa com ele só acrescenta a linha em `appointment_request_triages`, reduz `due_on` (o menor) e sobe a prioridade (a maior). Um índice único parcial (`citizen_id, appointment_type_key` em `kind = 'triage'` vivo) fecha a corrida entre duas conclusões; quem perde reentra pela fusão.
6. **Tipo do protocolo inexistente (ou desativado) na cidade:** o pedido nasce mesmo assim, com a chave (o nome cai para a chave); nenhuma necessidade clínica some. A recepção escolhe o tipo ao marcar (o corpo da marcação tem `appointment_type_key`). O gate só avisa (contratos §1, §9).
7. **Revogação:** fecha como `consent_revoked` o pedido de triagem `open` ligado a triagens da conversa revogada **se nenhuma outra triagem ligada a ele** pertence a conversa com consentimento ativo (pedido fundido de duas triagens não some por uma revogação só).
8. **Horário `legacy` ocupa 15 minutos** (`Appointment::LEGACY_SPAN`) só para a regra "cidadão sem dois horários ativos sobrepostos" (`citizen_busy`); não entra nas vagas (não tem profissional).
9. **Remarcar pela recepção:** `POST /attendance/requests/:id/appointments` em pedido `scheduled` encerra o horário vivo como `moved` e cria o novo ligado por `moved_from_appointment_id` (o mesmo mecanismo do esvaziamento de unidade, api#29), publicando `appointment.moved`. É assim que a recepção resolve "precisa remarcar". O esvaziamento (`HealthUnits::Drain`) continua levando o horário como `legacy` para a outra unidade (o profissional não atende lá) e passa a copiar os campos novos do pedido.
10. **Lembrete:** o job roda a cada 15 minutos e só age entre 17h e 20h no fuso da cidade (a janela do SMS termina às 20h); `reminded_at` o torna idempotente. SMS exige chave da cidade (`campaigns_sms_enabled`), provedor configurado, `sms_opt_in` **e** não ter silenciado os lembretes (`appointment_reminders_muted`, opt-out que já existe desde api#39). Falha do provedor não repete (como o lembrete de confirmação).
11. **Aviso de lembrete mora em tabela própria (`appointment_notices`):** o estado de leitura não pode ficar em `appointments`, cujo guarda congela horário encerrado (o cidadão lê depois do check-in). A exclusão do cadastro apaga as linhas (DELETE permitido pelo guarda).
12. **Limite de encaixe do turno** = `fit_in_limit` do modelo ligado (ativo ou não) ou, sem modelo, `city_profile.default_fit_in_limit` (padrão 2; sem `city_profile`, 2). A garantia é o `FOR UPDATE` do turno no `FitIn` (não há trigger que conte encaixes).
13. **Faixas na agenda e na pré-visualização** usam a forma do modelo (contratos §2: `starts`/`ends` `HH:MM`), já recortadas pelo turno no dia mostrado, com `appointment_type_name` nas `bookable` (contratos §9). Turno sem modelo vira uma faixa `bookable` do tipo resolvido; sem tipo, nenhuma faixa (só encaixe). Faixa que chega à meia-noite por recorte termina em `"24:00"` (só na saída; o modelo continua recusando `24:00`, `crosses_midnight`).
14. **`citizen.name`** de §4.4 fica de fora: o cidadão não tem nome no cadastro.
15. **`reopened_reason = citizen_reschedule`** é gravado no pedido, mas as saídas (fila e "Meus horários") continuam mostrando só `expired|no_show|null`; a remarcação pedida aparece em `reschedule_requested` (os tipos do dashboard e do wpda só conhecem os dois valores antigos).
16. **A agenda da unidade mantém a chave antiga `appointments`** (forma do módulo 08) ao lado da forma nova, para o dashboard em produção não quebrar entre o deploy do api e o do dashboard.

## Global Constraints

- Tudo de domínio no banco de cada cidade: migração em `db/city_migrate`, dump à mão em `db/city_schema.rb` (a paridade compara o schema normalizado pelo Postgres, `spec/services/city_schema_spec.rb`), triggers em `db/city_triggers.sql` (a migração termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`). Migração irreversível (`down` levanta `ActiveRecord::IrreversibleMigration`, como `20260926000001` e `20261004100001`). Rollout: publicar a imagem e rodar `city:migrate:all` **antes** de cortar tráfego; **nunca** migrar fora do rake (cidade trava em 503).
- Valores, exatamente (contratos §2): `booking_kind` ∈ `slot`, `fit_in`, `legacy`; faixa `kind` ∈ `walk_in`, `bookable`, `blocked`; `priority` ∈ `routine`, `priority`; `preferred_period` ∈ `morning`, `afternoon`, `any`; `reschedule_reason_code` ∈ `work`, `health`, `transport`, `other`; `kind` do pedido ∈ `return`, `referral`, `triage`. Tipo: `key` `^[a-z][a-z0-9_]{1,40}$`, `duration_minutes` 5–240, `cbo_prefixes` 1–20 itens de 1–6 dígitos, `name` 1–60 caracteres; modelo: `fit_in_limit` 0–20, `slot_minutes` 5–240. Frase fixa do cancelamento por remarcação: `"Remarcação pedida pelo cidadão"`. SMS: `"Secretaria de Saúde de {cidade}: você tem um compromisso de saúde amanhã. Veja em {link}"`, link = `Campaigns::SmsText.link(city)` (sem identificador).
- Base da plataforma: `consulta_medica` (20 min, `2251`, `2252`, `2253`), `consulta_enfermagem` (15, `2235`), `consulta_odontologica` (30, `2232`), `retorno` (15, todos os anteriores). Tipo `platform` nunca muda `key` nem `origin` (trigger) e não aceita `cbo_prefixes` na edição (`platform_type_locked`).
- Datas `YYYY-MM-DD` no fuso da cidade (`Time.zone` dentro de `CityConnection.with`); instantes ISO 8601 com fuso; `from`/`to` inclusivos (contratos §9).
- Jobs por cidade só via `prepend EachCityJob` (recorrente) ou `include CityScopedJob`; `Current.city` **nunca** é atribuído em `app/` ou `lib/` (`spec/architecture/current_city_assignment_spec.rb`).
- Eventos de domínio só com ids (contratos §6), declarados em `config/initializers/domain_events.rb` com `to: []` e em `spec/initializers/domain_events_bindings_spec.rb` (Task 2). Nenhum texto livre (motivo, nota, justificativa) em evento, log (`:reason` e `:note` já estão em `filter_parameters`) ou Analytics. Nenhum `Platform.audit` novo (nada em `R18_PLATFORM_EVENT_NAMES`).
- Erros `{ "error": "<reason>" }`; `render_failure` de `AttendanceAccess` para as rotas de balcão e de profissionais; papel ausente 403 `forbidden`.
- Specs de request com `type: :request`; arquivo novo em `spec/support/` precisa de `require_relative` em `spec/rails_helper.rb`. Specs com threads: `self.use_transactional_tests = false`, `Queue#pop(timeout:)`, `after` que solta as threads e apaga tudo o que commitou (com `session_replication_role = replica`, como `spec/commands/attendances/call_next_concurrency_spec.rb`).
- `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..29)`.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). `git add` sempre com caminhos explícitos (nunca `-A`).

## Ambiente de execução

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API"). Ordem de merge: `contracts` (tag `protocols-v1.5.0`) → api → dashboard e wpda.
- Worktree (a partir da raiz do monorepo, `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod17 -b feat/mod-17-scheduling origin/main
  cp apps/api/config/master.key apps/api/.claude/mod17/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod17`. Todo comando Rails/RSpec:

  ```bash
  docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec <arquivos>
  ```

- Todo `git add`/`git commit` usa `-C apps/api/.claude/mod17`.
- Depois da migração de cidade (Task 2): `DROP DATABASE` dos dois bancos de teste de cidade (`rota_saude_test_city_a`, `rota_saude_test_city_b`) e `docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod17 api bin/rails city:test_databases` (o Postgres é o do host: `psql -U rota_saude -d postgres -c "DROP DATABASE ..."`). Ao voltar para a main, repita (o banco de teste fica à frente).
- Suíte completa só com o worker parado e sem outra sessão rodando suíte:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec
  docker compose start worker
  ```

- **Conflitos com o módulo 16** (branch `feat/mod-16-record-mode`, outra sessão): os dois tocam `db/city_schema.rb`, `db/city_triggers.sql`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `config/routes.rb`, `config/recurring.yml`, `db/seeds.rb`, `spec/rails_helper.rb`, `spec/adr_pointers_spec.rb`, `app/commands/citizens/erase.rb`, `app/controllers/concerns/professional_rendering.rb` e `app/controllers/professionals_controller.rb`. O módulo 17 **não** toca a validação presencial (`citizens/verify.rb`, `attendance_controller.rb`). As migrações do 16 são `20261005200001` e `20261007100001`; a nossa é `20261008100001`, maior, para que o `define(version:)` do `city_schema.rb` fique com a nossa em qualquer ordem de merge. No rebase: manter os dois blocos (tabelas, triggers, bindings, rotas, linhas de seed, `require_relative`), `VALID_RANGE = (1..29)`, e no `erase.rb`/`professional_rendering.rb` aplicar as duas alterações (são linhas diferentes). Depois do rebase, refazer os bancos de teste e rodar `spec/services/city_schema_spec.rb`.
- Prova no navegador e planos do dashboard/wpda: o api do worktree sobe na porta **3034** (Task 22).

## Review Focus

1. **Fuso e virada do dia:** faixas do modelo em hora local (não UTC), turno que cruza a meia-noite, cidade em `America/Manaus`, e o lembrete "de amanhã" às 17h locais. Testes: Task 5 ("virada de dia", "fuso de Manaus") e Task 18 ("17h em Manaus, não em São Paulo").
2. **Duas recepções (ou dois cliques) na mesma vaga:** uma marca, a outra recebe 409 `slot_taken` — nunca 500, nunca dois horários. Encaixe no último lugar do limite: um passa, o outro recebe `fit_in_limit`. Testes: Task 11 (threads) e Task 12 ("ExclusionViolation vira 409").
3. **Turno cancelado ou modelo editado depois da marcação:** o horário fica intacto, aparece `shift_cancelled`/`outside_template`, o pedido entra na fila como `needs_reschedule` e a recepção remarca (o antigo vira `moved`). Testes: Task 12 ("remarcar move") e Task 13 ("turno cancelado e modelo editado").
4. **Pedido sem unidade:** triagem de quem não tem bairro (ou bairro sem unidade de referência) cai na fila "sem unidade"; atribuir duas vezes dá 409; nenhuma tela do cidadão quebra com `target_unit` nulo. Testes: Task 14 ("sem unidade"), Task 15 ("assign duas vezes"), Task 16 ("resultado sem unidade") e Task 17 ("lista com pedido sem unidade").
5. **"Não posso nesse horário" nas bordas:** depois do início, em horário já cancelado ou duas vezes → 409 `not_reschedulable`; o prazo não muda; a nota com dado pessoal nunca vai para evento. Testes: Task 17 (tabela de recusas) e Task 20 (invariante de texto livre).

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `config/protocols/schema.json`, `app/protocols/validation/scheduling.rb`, `app/protocols/gate.rb`, `app/services/protocols/scheduling_targets.rb`, `app/controllers/authoring/protocols_controller.rb` | schema 1.5.0, gate e aviso de tipo | 1 |
| `db/city_migrate/20261008100001_add_professional_schedules.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `config/scheduling/appointment_types.yml`, `app/models/{appointment_type,schedule_template,appointment_request_triage,appointment_notice}.rb`, `app/models/{appointment,appointment_request,professional_shift,professional_link}.rb`, `config/initializers/domain_events.rb`, `spec/support/scheduling_helpers.rb`, `spec/adr_pointers_spec.rb` | dados | 2 |
| `app/services/scheduling/appointment_types.rb`, `app/services/scheduling/availability/type.rb`, `app/jobs/provision_city_job.rb` | base de tipos e catálogo | 3 |
| `app/commands/scheduling/save_appointment_type.rb`, `app/controllers/appointment_types_controller.rb` | tipos de atendimento | 4 |
| `app/services/scheduling/availability.rb` (parte pura) | faixas e vagas | 5 |
| `app/services/scheduling/template_blocks.rb`, `app/services/scheduling/block_json.rb`, `app/commands/scheduling/save_template.rb`, `app/services/scheduling/template_preview.rb`, `app/controllers/schedule_templates_controller.rb` | modelos e pré-visualização | 6 |
| `app/commands/professionals/{schedule_shift,set_shift_template,set_link_default_type}.rb`, controllers de turno e vínculo, `professional_rendering.rb` | modelo no turno, tipo padrão | 7 |
| `app/services/scheduling/availability.rb` (carga), `app/services/scheduling/transition.rb`, `app/services/scheduling/date_range.rb` | vagas da unidade, `legacy_days` | 8 |
| `app/commands/appointments/{placement,book}.rb` | marcação em vaga | 9 |
| `app/commands/appointments/fit_in.rb`, `app/services/scheduling/fit_in_limit.rb` | encaixe | 10 |
| `spec/commands/appointments/booking_concurrency_spec.rb` | corridas | 11 |
| `app/controllers/appointment_requests_controller.rb`, `app/commands/appointments/schedule.rb`, `app/services/scheduling/appointment_presenter.rb` | marcação na API | 12 |
| `app/services/scheduling/{block_json,unit_agenda,professional_agenda}.rb`, `app/controllers/professional_agenda_controller.rb` | agendas | 13 |
| `app/commands/triages/schedule.rb`, `app/commands/complete_triage.rb`, `app/commands/appointment_requests/lifecycle.rb`, `app/commands/health_units/drain.rb` | pedido da triagem | 14 |
| `app/services/scheduling/request_json.rb`, `app/commands/appointment_requests/assign_unit.rb`, `appointment_requests_controller.rb` | fila | 15 |
| `app/commands/appointment_requests/close_revoked.rb`, `app/commands/revoke_consent.rb`, `app/commands/citizens/erase.rb`, `app/services/scheduling/triage_request.rb`, `citizen_api/triages_controller.rb` | LGPD e resultado | 16 |
| `app/commands/appointments/request_reschedule.rb`, `citizen_api/appointments_controller.rb`, `app/services/scheduling/unit_address.rb` | cidadão | 17 |
| `app/commands/appointments/remind.rb`, `app/jobs/appointments/remind_job.rb`, `config/recurring.yml` | lembrete | 18 |
| `citizen_api/notices_controller.rb` | caixa de avisos | 19 |
| `spec/invariants/scheduling_invariants_spec.rb` | invariantes do ADR 0029 | 20 |
| `lib/scheduling_crew.rb`, `lib/triage_catalog_crew.rb`, `db/seeds.rb` | semente de dev | 21 |

---

## Fatia 1 — Protocolo (F-17.5, parte do schema)

### Task 1: Schema `protocols-v1.5.0`, gate de `scheduling` e aviso de tipo inexistente

A linguagem já aceita `outcome.*`/`profile.*` e `gte`/`lte` (módulo 15, em `origin/main`); esta task só copia o schema e liga o gate.

**Files:**
- Modify: `config/protocols/schema.json`
- Create: `app/protocols/validation/scheduling.rb`
- Modify: `app/protocols/gate.rb`
- Create: `app/services/protocols/scheduling_targets.rb`
- Modify: `app/controllers/authoring/protocols_controller.rb`
- Test: `spec/protocols/schema_scheduling_spec.rb`, `spec/protocols/validation/scheduling_spec.rb`, `spec/services/protocols/scheduling_targets_spec.rb`, `spec/requests/authoring_scheduling_gate_spec.rb`

**Interfaces:**
- Consumes: `Protocols::Validation::Condition.errors(node, by_id, variables:)`, `Protocols::Validation::Offer::SUGGESTION` (módulo 15).
- Produces:
  - `Protocols::Validation::Scheduling::VARIABLES == Protocols::Validation::Offer::SUGGESTION` (`profile.age profile.sex outcome.tier outcome.score outcome.priority`);
  - `Protocols::Validation::Scheduling.call(definition) -> Array<String>` (total);
  - `Protocols::SchedulingTargets.warnings(definition) -> Array<String>` (precisa de banco; fora de `app/protocols/`);
  - `POST /authoring/protocols/gate` → 200 `{ valid: true, warnings: [...] }` quando há aviso (como o módulo 15).

- [ ] **Step 1: Confira a tag do `contracts` e copie o schema**

```bash
/opt/homebrew/bin/git -C contracts fetch --tags
/opt/homebrew/bin/git -C contracts show protocols-v1.5.0:protocols/schema.json > apps/api/.claude/mod17/config/protocols/schema.json
/opt/homebrew/bin/git -C apps/api/.claude/mod17 diff --stat config/protocols/schema.json
```
Expected: só a propriedade nova `scheduling` (abaixo de `suggestions`). Se a tag não existe, **pare** e avise: o api não mergeia antes do `contracts`. O acréscimo, para conferência:

```json
"scheduling": {
  "type": "array", "maxItems": 10,
  "items": { "type": "object", "additionalProperties": false,
    "required": ["when", "appointment_type", "priority", "due_in_days"],
    "properties": {
      "when":             { "$ref": "#/$defs/condition" },
      "appointment_type": { "type": "string", "pattern": "^[a-z][a-z0-9_]{1,40}$" },
      "priority":         { "enum": ["routine", "priority"] },
      "due_in_days":      { "type": "integer", "minimum": 1, "maximum": 365 } } }
}
```

- [ ] **Step 2: Escreva as specs que falham**

```ruby
# spec/protocols/schema_scheduling_spec.rb
require "rails_helper"
require "json_schemer"

# protocols-v1.5.0 (ADR 0029; contratos §1, §8): scheduling opcional, até 10
# regras, `when` só na forma estruturada.
RSpec.describe "protocols schema.json scheduling contract (v1.5.0)" do
  let(:schema) { JSONSchemer.schema(JSON.parse(File.read(Rails.root.join("config/protocols/schema.json")))) }

  def base(extra = {})
    {
      "name" => "saude-do-idoso", "version" => 1, "start_step_id" => "s1",
      "steps" => [ { "id" => "s1", "prompt" => "?", "answer_type" => "integer", "branches" => {}, "weights" => {} } ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 9 } }
    }.merge(extra)
  end

  def rule(extra = {})
    { "when" => { "gte" => ["outcome.score", 3] }, "appointment_type" => "consulta_medica",
      "priority" => "routine", "due_in_days" => 30 }.merge(extra)
  end

  it "aceita protocolo sem scheduling (1.4.0 continua válido) e com regras completas" do
    expect(schema.valid?(base)).to be(true)
    expect(schema.valid?(base("scheduling" => [ rule, rule("priority" => "priority", "due_in_days" => 1) ]))).to be(true)
  end

  it "recusa campo faltando, prioridade, prazo, tipo e mapa simples no when" do
    expect(schema.valid?(base("scheduling" => [ rule.except("due_in_days") ]))).to be(false)
    expect(schema.valid?(base("scheduling" => [ rule("priority" => "urgent") ]))).to be(false)
    expect(schema.valid?(base("scheduling" => [ rule("due_in_days" => 0) ]))).to be(false)
    expect(schema.valid?(base("scheduling" => [ rule("due_in_days" => 366) ]))).to be(false)
    expect(schema.valid?(base("scheduling" => [ rule("appointment_type" => "Consulta") ]))).to be(false)
    expect(schema.valid?(base("scheduling" => [ rule("when" => { "s1" => "1" }) ]))).to be(false)
    expect(schema.valid?(base("scheduling" => [ rule("extra" => 1) ]))).to be(false)
    expect(schema.valid?(base("scheduling" => Array.new(11) { rule }))).to be(false)
  end
end
```

```ruby
# spec/protocols/validation/scheduling_spec.rb
require "rails_helper"

# ADR 0029 (spec §5.1): o `when` do agendamento aceita outcome.*, profile.* e
# os passos do próprio protocolo; nada de citizen.*.
RSpec.describe Protocols::Validation::Scheduling do
  def definition(scheduling)
    { "name" => "saude-do-idoso", "version" => 1, "start_step_id" => "quedas",
      "steps" => [ { "id" => "quedas", "prompt" => "?", "answer_type" => "boolean" },
                   { "id" => "remedios", "prompt" => "?", "answer_type" => "integer" } ],
      "scheduling" => scheduling }
  end

  def rule(node) = { "when" => node, "appointment_type" => "consulta_medica", "priority" => "routine", "due_in_days" => 30 }

  it "aceita resultado, perfil e passos" do
    node = { "any" => [ { "gte" => ["outcome.score", 3] }, { "eq" => ["profile.sex", "female"] },
                        { "gte" => ["profile.age", 60] }, { "eq" => ["quedas", "true"] }, { "gte" => ["remedios", 5] } ] }
    expect(described_class.call(definition([ rule(node) ]))).to eq([])
  end

  it "recusa variável de outro lugar, passo inexistente e regra que não é objeto" do
    errors = described_class.call(definition([ rule({ "eq" => ["citizen.neighborhood_id", SecureRandom.uuid] }),
                                               rule({ "eq" => ["fantasma", "1"] }), "x" ]))
    expect(errors.size).to be >= 3
    expect(errors).to all(start_with("scheduling["))
    expect(errors.join).to include("scheduling[0].when", "scheduling[1].when", "scheduling[2] must be an object")
  end

  it "sem scheduling não há erro; scheduling que não é lista é erro; total para lixo" do
    expect(described_class.call(definition(nil).except("scheduling"))).to eq([])
    expect(described_class.call(definition("x"))).to eq(["scheduling must be an array"])
    expect(described_class.call(nil)).to eq([])
  end
end
```

```ruby
# spec/services/protocols/scheduling_targets_spec.rb
require "rails_helper"

# Contratos §1/§9: tipo inexistente (ou desativado) na cidade só avisa.
RSpec.describe Protocols::SchedulingTargets do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  def definition(*keys)
    { "scheduling" => keys.map { |k| { "when" => { "gte" => ["outcome.score", 1] }, "appointment_type" => k,
                                        "priority" => "routine", "due_in_days" => 7 } } }
  end

  it "avisa tipo inexistente e desativado; tipo ativo não avisa" do
    AppointmentType.create!(key: "consulta_medica", name: "Consulta médica", duration_minutes: 20,
                            cbo_prefixes: ["2251"], origin: "platform")
    AppointmentType.create!(key: "acupuntura", name: "Acupuntura", duration_minutes: 30, cbo_prefixes: ["2251"],
                            origin: "city", active: false)
    expect(described_class.warnings(definition("consulta_medica", "fantasma", "acupuntura"))).to eq([
      "scheduling appointment_type 'fantasma' does not exist in this city",
      "scheduling appointment_type 'acupuntura' is inactive in this city"
    ])
  end

  it "total: sem scheduling ou com lixo, nenhum aviso" do
    expect(described_class.warnings({})).to eq([])
    expect(described_class.warnings({ "scheduling" => "x" })).to eq([])
    expect(described_class.warnings(nil)).to eq([])
  end
end
```

```ruby
# spec/requests/authoring_scheduling_gate_spec.rb
require "rails_helper"

RSpec.describe "POST /authoring/protocols/gate com scheduling", type: :request do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:author) { staff_with("autoria-#{SecureRandom.hex(3)}@cidade.gov.br", "protocol_author") }

  def definition(scheduling)
    { "name" => "saude-do-idoso", "version" => 1, "start_step_id" => "s1",
      "steps" => [ { "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
                     "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 4, "false" => 0 } } ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "media" => 3 },
                     "priority_map" => { "baixa" => 9, "media" => 5 } },
      "scheduling" => scheduling }
  end

  it "válido com aviso de tipo inexistente (200 + warnings); variável proibida é 422" do
    sign_in_as(author)
    rule = { "when" => { "gte" => ["outcome.score", 3] }, "appointment_type" => "fantasma",
             "priority" => "routine", "due_in_days" => 30 }
    json_post "/authoring/protocols/gate", definition: definition([ rule ])
    expect(response).to have_http_status(:ok)
    expect(JSON.parse(response.body)).to eq("valid" => true,
                                            "warnings" => ["scheduling appointment_type 'fantasma' does not exist in this city"])

    json_post "/authoring/protocols/gate",
              definition: definition([ rule.merge("when" => { "eq" => ["citizen.neighborhood_id", SecureRandom.uuid] }) ])
    expect(response).to have_http_status(:unprocessable_entity)
    expect(JSON.parse(response.body)["errors"].join).to include("scheduling[0].when")
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/protocols/schema_scheduling_spec.rb spec/protocols/validation/scheduling_spec.rb spec/services/protocols/scheduling_targets_spec.rb spec/requests/authoring_scheduling_gate_spec.rb`
Expected: o schema já passa (Step 1); as outras falham com `uninitialized constant Protocols::Validation::Scheduling` / `Protocols::SchedulingTargets`. A spec de `SchedulingTargets` e a de request dependem de `AppointmentType` (Task 2): elas, o serviço e a mudança do controller **não entram no commit desta task** — ficam no worktree e são commitadas na Task 2 (Step 10), já verdes.

- [ ] **Step 4: Implemente**

```ruby
# app/protocols/validation/scheduling.rb
# Gate de scheduling (ADR 0029; spec 2026-10-05 §5.1; contratos §1). O `when`
# aceita as variáveis de resultado e de perfil e os passos do próprio
# protocolo — o mesmo conjunto de suggestions[].when. Total para qualquer
# entrada: o editor do dashboard chama o gate com a definição em edição.
module Protocols
  module Validation
    module Scheduling
      VARIABLES = Offer::SUGGESTION

      module_function

      def call(definition)
        return [] unless definition.is_a?(Hash) && definition.key?("scheduling")

        rules = definition["scheduling"]
        return ["scheduling must be an array"] unless rules.is_a?(Array)

        by_id = Array(definition["steps"]).select { |s| s.is_a?(Hash) }.to_h { |s| [s["id"].to_s, s] }
        rules.each_with_index.flat_map do |rule, index|
          next ["scheduling[#{index}] must be an object"] unless rule.is_a?(Hash)

          Condition.errors(rule["when"], by_id, variables: VARIABLES).map { |error| "scheduling[#{index}].when: #{error}" }
        end
      end
    end
  end
end
```

Em `app/protocols/gate.rb`, depois da linha do `Validation::Offer`:

```ruby
      errors.concat(Validation::Scheduling.call(definition)) # ADR 0029: scheduling[].when
```

```ruby
# app/services/protocols/scheduling_targets.rb
# Aviso do gate que precisa de banco (ADR 0029; contratos §1, §9): tipo de
# atendimento que não existe, ou está desativado, na cidade. Avisa, nunca
# bloqueia — o pedido nasce com a chave mesmo assim (Triages::Schedule).
module Protocols
  module SchedulingTargets
    module_function

    def warnings(definition)
      return [] unless definition.is_a?(Hash) && definition["scheduling"].is_a?(Array)

      keys = definition["scheduling"].filter_map { |r| r["appointment_type"].to_s.presence if r.is_a?(Hash) }.uniq
      return [] if keys.empty?

      active = AppointmentType.where(key: keys).pluck(:key, :active).to_h
      keys.filter_map do |key|
        if !active.key?(key) then "scheduling appointment_type '#{key}' does not exist in this city"
        elsif !active[key] then "scheduling appointment_type '#{key}' is inactive in this city"
        end
      end
    end
  end
end
```

Em `app/controllers/authoring/protocols_controller.rb`, `gate` passa a somar os dois avisos:

```ruby
    def gate
      definition = definition_param
      warnings = Protocols::SuggestionTargets.warnings(definition) + Protocols::SchedulingTargets.warnings(definition)
      render_gate(Protocols::Gate.call(definition), warnings: warnings)
    end
```

- [ ] **Step 5: Rode as specs do protocolo**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/protocols spec/invariants/protocol_engine_purity_spec.rb`
Expected: PASS (as duas specs que precisam de `AppointmentType` ficam para a Task 2).

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add config/protocols/schema.json app/protocols/validation/scheduling.rb app/protocols/gate.rb spec/protocols/schema_scheduling_spec.rb spec/protocols/validation/scheduling_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: validate protocol scheduling rules (protocols-v1.5.0)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 2 — Dados (F-17.1 a F-17.4, base)

### Task 2: Migração de cidade, guardas do banco, modelos e eventos declarados

**Files:**
- Create: `config/scheduling/appointment_types.yml`
- Create: `db/city_migrate/20261008100001_add_professional_schedules.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`
- Create: `app/models/appointment_type.rb`, `app/models/schedule_template.rb`, `app/models/appointment_request_triage.rb`, `app/models/appointment_notice.rb`
- Modify: `app/models/appointment.rb`, `app/models/appointment_request.rb`, `app/models/professional_shift.rb`, `app/models/professional_link.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Create: `spec/support/scheduling_helpers.rb`; Modify: `spec/rails_helper.rb`
- Modify: `spec/adr_pointers_spec.rb`, `spec/invariants/appointment_invariants_spec.rb`
- Test: `spec/models/scheduling_tables_guard_spec.rb`, `spec/models/add_professional_schedules_migration_spec.rb`

**Interfaces:**
- Produces:
  - Tabelas e colunas: `appointment_types (key único, name, duration_minutes, cbo_prefixes varchar[], active, origin, position)`; `schedule_templates (name, fit_in_limit, blocks jsonb, active)`; `appointment_request_triages (request_id, triage_id, created_at)`; `appointment_notices (appointment_id único, citizen_id, read_at, created_at)`; `appointments.{professional_id, appointment_type_key, ends_at, shift_id, booking_kind (padrão 'legacy'), fit_in_reason, reschedule_requested, reminded_at}`; `appointment_requests.{origin_triage_id, appointment_type_key (padrão 'retorno'), priority (padrão 'routine'), due_on, reschedule_reason_code, reschedule_note, preferred_period, reschedule_count}` e `origin_attendance_id`/`origin_unit_id`/`target_unit_id` nulos; `professional_shifts.schedule_template_id`; `professional_links.default_appointment_type_key`; `city_profile.default_fit_in_limit` (padrão 2).
  - Modelos: `AppointmentType` (`scope :active_types`), `ScheduleTemplate`, `AppointmentRequestTriage` (`belongs_to :request, :triage`), `AppointmentNotice` (`belongs_to :appointment, :citizen`); `Appointment::BOOKING_KINDS`, `::ACTIVE == %w[scheduled confirmed checked_in]`, `::LEGACY_SPAN == 15.minutes`, `::MIN_FIT_IN_REASON == 10`, `::RESCHEDULE_CANCEL_REASON`, `#fit_in?`, `#effective_ends_at`, `belongs_to :professional, :shift` (opcionais); `AppointmentRequest::KINDS` (com `triage`), `::PRIORITIES`, `::PERIODS`, `::RESCHEDULE_REASONS`, `::DUE_IN_DAYS == 30`, `#origin -> "attendance"|"triage"`, `has_many :request_triages`; `ProfessionalShift belongs_to :schedule_template` (opcional).
  - `AddProfessionalSchedules.inherited_overlaps(connection) -> [[id, id]]`, `.assert_no_inherited_overlap!(connection)` (levanta `AddProfessionalSchedules::InheritedOverlap`).
  - Helpers de spec: `shift!(link, starts_at:, ends_at: starts_at + 4.hours, template: nil)`, `triage_request!(citizen, unit:, type_key: "consulta_medica", priority: "routine", due_on: Time.zone.today + 30)`, `appointment_row!(request, shift, starts_at:, minutes: 20, kind: "slot", status: "confirmed", reason: nil)`, `type_row!(key, cbo: ["2251"], minutes: 20, origin: "city", active: true)`, `doctor_link!(unit, cbo: "225125") -> ProfessionalLink`.

- [ ] **Step 1: Confira o número da migração**

Run: `ls apps/api/.claude/mod17/db/city_migrate | tail -2`
Expected: a última é `20261005100001_add_triage_catalog.rb`. Use `20261008100001` (acima das do módulo 16, ver "Ambiente de execução"). Se houver migração mais nova em `origin/main`, use um número maior e ajuste o nome nos Steps 4, 6 e o `define(version:)`.

- [ ] **Step 2: Escreva a base de tipos**

```yaml
# config/scheduling/appointment_types.yml
# Base da plataforma (ADR 0029; spec 2026-10-05 §3.1). Copiada para cada cidade
# pela migração 20261008100001 e pelo provisionamento (só insere o que falta,
# nunca sobrescreve o que a cidade ajustou). Grupos = prefixos de CBO.
- key: consulta_medica
  name: Consulta médica
  duration_minutes: 20
  cbo_prefixes: ["2251", "2252", "2253"]
- key: consulta_enfermagem
  name: Consulta de enfermagem
  duration_minutes: 15
  cbo_prefixes: ["2235"]
- key: consulta_odontologica
  name: Consulta odontológica
  duration_minutes: 30
  cbo_prefixes: ["2232"]
- key: retorno
  name: Retorno
  duration_minutes: 15
  cbo_prefixes: ["2251", "2252", "2253", "2235", "2232"]
```

- [ ] **Step 3: Escreva os helpers de spec**

```ruby
# spec/support/scheduling_helpers.rb
# Módulo 17 (ADR 0029): turnos, pedidos de triagem e horários gravados direto
# (cenário de teste). Os caminhos reais são ScheduleShift, Triages::Schedule e
# Appointments::Book/FitIn.
module SchedulingHelpers
  def type_row!(key, cbo: ["2251"], minutes: 20, origin: "city", active: true, name: key.humanize)
    AppointmentType.create!(key: key, name: name, duration_minutes: minutes, cbo_prefixes: cbo, origin: origin,
                            active: active)
  end

  # Médica(o) com vínculo ativo na unidade (o perfil nasce pelo link_professional! do módulo 10).
  def doctor_link!(unit, cbo: "225125", email: "medica-#{SecureRandom.hex(3)}@cidade.gov.br")
    link_professional!(staff_with(email, "health_professional"), unit, cbo: cbo)
  end

  def shift!(link, starts_at:, ends_at: starts_at + 4.hours, template: nil)
    ProfessionalShift.create!(professional_link: link, professional_id: link.professional_id, starts_at: starts_at,
                              ends_at: ends_at, created_by_user: link.started_by_user, schedule_template: template)
  end

  def triage_request!(citizen, unit:, type_key: "consulta_medica", priority: "routine", due_on: Time.zone.today + 30)
    triage = completed_web_triage_for(citizen)
    AppointmentRequest.create!(kind: "triage", origin_triage: triage, root_triage: triage, citizen: citizen,
                               target_unit: unit, appointment_type_key: type_key, priority: priority, due_on: due_on)
  end

  def appointment_row!(request, shift, starts_at:, minutes: 20, kind: "slot", status: "confirmed", reason: nil)
    Appointment.create!(
      request: request, citizen_id: request.citizen_id, health_unit_id: shift.professional_link.health_unit_id,
      scheduled_at: starts_at, ends_at: starts_at + minutes.minutes, scheduled_by_user: shift.created_by_user,
      status: status, confirmed_at: status == "confirmed" ? Time.current : nil,
      confirmation_deadline_at: status == "scheduled" ? starts_at - 1.day : nil,
      professional_id: shift.professional_id, shift_id: shift.id, appointment_type_key: "consulta_medica",
      booking_kind: kind, fit_in_reason: reason
    )
  end
end

RSpec.configure { |c| c.include SchedulingHelpers }
```

Em `spec/rails_helper.rb`, depois de `require_relative "support/triage_catalog_helpers"`, acrescente `require_relative "support/scheduling_helpers"`.

- [ ] **Step 4: Escreva a spec de guarda (falha: tabelas e colunas não existem)**

```ruby
# spec/models/scheduling_tables_guard_spec.rb
require "rails_helper"

# Módulo 17 (ADR 0029; spec §3–§5): o banco garante o que o modelo não vê.
RSpec.describe "Guardas das tabelas da agenda" do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:link) { doctor_link!(unit) }
  let(:shift) { shift!(link, starts_at: 2.days.from_now.change(hour: 8)) }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:other) { Citizen.create!(cpf: "11144477735", phone: "+5541911112222") }

  def attempt(&) = ApplicationRecord.transaction(requires_new: true, &)

  describe "appointments" do
    it "slot e encaixe exigem profissional, tipo, fim e turno; legacy não" do
      req = triage_request!(citizen, unit: unit)
      row = appointment_row!(req, shift, starts_at: shift.starts_at).attributes.except("id")
      row.merge!("status" => "cancelled_by_citizen", "cancel_reason" => "não posso ir mais", "ended_at" => Time.current)
      expect { attempt { Appointment.insert_all!([ row.merge("ends_at" => nil) ]) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_appointments_booking_fields/)
      expect { attempt { Appointment.insert_all!([ row.merge("booking_kind" => "outro") ]) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_appointments_booking_kind/)
      expect { attempt { Appointment.insert_all!([ row.merge("ends_at" => row["scheduled_at"]) ]) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_appointments_ends/)
      legacy = row.merge("booking_kind" => "legacy", "professional_id" => nil, "shift_id" => nil, "ends_at" => nil,
                         "appointment_type_key" => nil)
      expect { attempt { Appointment.insert_all!([ legacy ]) } }.not_to raise_error
    end

    it "encaixe exige justificativa de 10+ caracteres; slot não pode ter justificativa" do
      expect { attempt { appointment_row!(triage_request!(citizen, unit: unit), shift, starts_at: shift.starts_at, kind: "fit_in", reason: "curta") } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_appointments_fit_in_reason/)
      expect { attempt { appointment_row!(triage_request!(other, unit: unit), shift, starts_at: shift.starts_at, reason: "gestante com dor") } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_appointments_fit_in_reason/)
    end

    it "dois slots ativos sobrepostos do mesmo profissional: o banco recusa; encaixe e cancelado não contam" do
      first = appointment_row!(triage_request!(citizen, unit: unit), shift, starts_at: shift.starts_at)
      expect { attempt { appointment_row!(triage_request!(other, unit: unit), shift, starts_at: shift.starts_at + 10.minutes) } }
        .to raise_error(ActiveRecord::ExclusionViolation, /excl_appointments_slot_overlap/)
      expect { attempt { appointment_row!(triage_request!(other, unit: unit), shift, starts_at: shift.starts_at + 10.minutes, kind: "fit_in", reason: "retorno que não espera") } }
        .not_to raise_error
      first.update!(status: "cancelled_by_citizen", cancel_reason: "não posso ir mais", ended_at: Time.current)
      third = Citizen.create!(cpf: "39053344705", phone: "+5541933334444")
      expect { attempt { appointment_row!(triage_request!(third, unit: unit), shift, starts_at: shift.starts_at) } }.not_to raise_error
    end

    it "profissional, tipo, fim, turno, modo e justificativa nunca mudam" do
      appt = appointment_row!(triage_request!(citizen, unit: unit), shift, starts_at: shift.starts_at)
      { professional_id: doctor_link!(unit).professional_id, appointment_type_key: "retorno",
        ends_at: appt.ends_at + 5.minutes, shift_id: shift!(link, starts_at: 5.days.from_now.change(hour: 8)).id,
        booking_kind: "legacy" }.each do |column, value|
        expect { attempt { Appointment.where(id: appt.id).update_all(column => value) } }
          .to raise_error(ActiveRecord::StatementInvalid, /scheduled columns never change/), column.to_s
      end
      expect { Appointment.where(id: appt.id).update_all(reminded_at: Time.current, reschedule_requested: false) }.not_to raise_error
    end
  end

  describe "appointment_requests" do
    it "exatamente uma origem; pedido de triagem tem origin_triage; sem origem nem as duas, recusa" do
      req = triage_request!(citizen, unit: unit)
      row = req.attributes.except("id")
      expect { attempt { AppointmentRequest.insert_all!([ row.merge("origin_triage_id" => nil) ]) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_appointment_requests_origin/)
      expect { attempt { AppointmentRequest.insert_all!([ row.merge("kind" => "return", "appointment_type_key" => "x") ]) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_appointment_requests_triage_kind/)
      %w[priority preferred_period reschedule_reason_code].each do |column|
        expect { attempt { AppointmentRequest.where(id: req.id).update_all(column => "lixo") } }
          .to raise_error(ActiveRecord::StatementInvalid, /ck_appointment_requests_#{column}/)
      end
      expect { attempt { AppointmentRequest.where(id: req.id).update_all(reschedule_note: "x" * 201) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_appointment_requests_reschedule_note/)
    end

    it "unidade de destino: nula → unidade uma vez; depois nunca muda" do
      req = triage_request!(citizen, unit: nil)
      expect { AppointmentRequest.where(id: req.id).update_all(target_unit_id: unit.id) }.not_to raise_error
      expect { attempt { AppointmentRequest.where(id: req.id).update_all(target_unit_id: create_unit("UBS Sul").id) } }
        .to raise_error(ActiveRecord::StatementInvalid, /origin columns never change/)
      expect { attempt { AppointmentRequest.where(id: req.id).update_all(origin_triage_id: completed_web_triage_for(other).id) } }
        .to raise_error(ActiveRecord::StatementInvalid, /origin columns never change/)
    end

    it "um pedido de triagem vivo por tipo por cidadão" do
      triage_request!(citizen, unit: unit)
      expect { attempt { triage_request!(citizen, unit: unit) } }.to raise_error(ActiveRecord::RecordNotUnique)
      expect { attempt { triage_request!(citizen, unit: unit, type_key: "retorno") } }.not_to raise_error
    end
  end

  it "tipos: key e origin nunca mudam; DELETE recusado" do
    type = type_row!("acupuntura")
    expect { attempt { AppointmentType.where(id: type.id).update_all(key: "outra") } }
      .to raise_error(ActiveRecord::StatementInvalid, /key and origin never change/)
    expect { attempt { AppointmentType.where(id: type.id).update_all(origin: "platform") } }
      .to raise_error(ActiveRecord::StatementInvalid, /key and origin never change/)
    expect { attempt { AppointmentType.where(id: type.id).delete_all } }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
    expect { attempt { type_row!("Maiuscula") } }.to raise_error(ActiveRecord::StatementInvalid, /ck_appointment_types_key/)
    expect { attempt { type_row!("longa", minutes: 241) } }.to raise_error(ActiveRecord::StatementInvalid, /ck_appointment_types_duration/)
    expect { attempt { type_row!("vazia", cbo: []) } }.to raise_error(ActiveRecord::StatementInvalid, /ck_appointment_types_cbo_prefixes/)
  end

  it "modelos: limite 0..20, blocks é lista, DELETE recusado" do
    template = ScheduleTemplate.create!(name: "Manhã", blocks: [])
    expect { attempt { ScheduleTemplate.where(id: template.id).update_all(fit_in_limit: 21) } }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_schedule_templates_fit_in_limit/)
    expect { attempt { ScheduleTemplate.where(id: template.id).update_all(blocks: {}) } }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_schedule_templates_blocks/)
    expect { attempt { ScheduleTemplate.where(id: template.id).delete_all } }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
  end

  it "ligação pedido↔triagem é só acréscimo; aviso de lembrete registra só a primeira leitura e aceita DELETE" do
    req = triage_request!(citizen, unit: unit)
    link_row = AppointmentRequestTriage.create!(request: req, triage: completed_web_triage_for(citizen), created_at: Time.current)
    expect { attempt { AppointmentRequestTriage.where(id: link_row.id).delete_all } }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)

    notice = AppointmentNotice.create!(appointment: appointment_row!(req, shift, starts_at: shift.starts_at),
                                       citizen: citizen, created_at: Time.current)
    expect { AppointmentNotice.where(id: notice.id).update_all(read_at: Time.current) }.not_to raise_error
    expect { attempt { AppointmentNotice.where(id: notice.id).update_all(read_at: 1.day.ago) } }
      .to raise_error(ActiveRecord::StatementInvalid, /only the first read/)
    expect { AppointmentNotice.where(id: notice.id).delete_all }.not_to raise_error
  end

  it "limite de encaixe padrão da cidade: 0..20" do
    CityProfile.create!(name: "Cidade") unless CityProfile.exists?
    expect(CityProfile.current.default_fit_in_limit).to eq(2)
    expect { attempt { CityProfile.update_all(default_fit_in_limit: 21) } }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_city_profile_default_fit_in_limit/)
  end
end
```

```ruby
# spec/models/add_professional_schedules_migration_spec.rb
require "rails_helper"
require Rails.root.join("db/city_migrate/20261008100001_add_professional_schedules.rb").to_s

# ADR 0029 (Consequências): a migração recusa seguir com horários `slot` ativos
# sobrepostos herdados, listando os ids. Num savepoint: tira a EXCLUDE, grava a
# sobreposição e chama a verificação que o up() roda antes de recriá-la.
RSpec.describe "Migração de cidade 20261008100001 (AddProfessionalSchedules): sobreposição herdada" do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:conn) { ApplicationRecord.connection }

  it "sem sobreposição passa; com sobreposição aborta listando os dois ids" do
    expect { AddProfessionalSchedules.assert_no_inherited_overlap!(conn) }.not_to raise_error

    ApplicationRecord.transaction(requires_new: true) do
      conn.execute("ALTER TABLE appointments DROP CONSTRAINT excl_appointments_slot_overlap")
      unit = create_unit
      shift = shift!(doctor_link!(unit), starts_at: 2.days.from_now.change(hour: 8))
      a = appointment_row!(triage_request!(Citizen.create!(cpf: "52998224725", phone: "+5541998765432"), unit: unit),
                           shift, starts_at: shift.starts_at)
      b = appointment_row!(triage_request!(Citizen.create!(cpf: "11144477735", phone: "+5541911112222"), unit: unit),
                           shift, starts_at: shift.starts_at + 5.minutes)

      expect(AddProfessionalSchedules.inherited_overlaps(conn)).to eq([ [ a.id, b.id ].sort ])
      expect { AddProfessionalSchedules.assert_no_inherited_overlap!(conn) }
        .to raise_error(AddProfessionalSchedules::InheritedOverlap, /#{a.id}.*#{b.id}|#{b.id}.*#{a.id}/)
      raise ActiveRecord::Rollback
    end
  end
end
```

- [ ] **Step 5: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/models/scheduling_tables_guard_spec.rb spec/models/add_professional_schedules_migration_spec.rb`
Expected: FAIL (`uninitialized constant AppointmentType` / arquivo de migração inexistente).

- [ ] **Step 6: Escreva a migração**

```ruby
# db/city_migrate/20261008100001_add_professional_schedules.rb
# Agenda dos profissionais (ADR 0029; spec 2026-10-05 §3–§5): tipos de
# atendimento (base copiada), modelos de agenda, colunas novas em horários,
# pedidos, turnos, vínculos e perfil da cidade; ligação pedido↔triagem; avisos
# de lembrete; e a trava de sobreposição dos horários `slot` (btree_gist).
# Todo horário existente vira `legacy` (padrão da coluna, sem UPDATE); a
# verificação de sobreposição herdada roda antes da EXCLUDE e aborta listando
# os pares. CHECKs na forma que o dump reproduz.
class AddProfessionalSchedules < ActiveRecord::Migration[8.1]
  class InheritedOverlap < StandardError; end

  ACTIVE = "status::text = ANY (ARRAY['scheduled', 'confirmed', 'checked_in']::text[])".freeze
  BASE_PATH = Rails.root.join("config/scheduling/appointment_types.yml")

  # Pares (a < b) de horários `slot` ativos do mesmo profissional que se
  # sobrepõem: exatamente o que a EXCLUDE recusaria.
  def self.inherited_overlaps(connection)
    connection.select_rows(<<~SQL)
      SELECT a.id, b.id FROM appointments a
      JOIN appointments b ON b.professional_id = a.professional_id AND a.id < b.id
       AND tsrange(a.scheduled_at, a.ends_at) && tsrange(b.scheduled_at, b.ends_at)
      WHERE a.booking_kind = 'slot' AND b.booking_kind = 'slot'
        AND a.status IN ('scheduled', 'confirmed', 'checked_in') AND b.status IN ('scheduled', 'confirmed', 'checked_in')
      ORDER BY 1, 2
    SQL
  end

  def self.assert_no_inherited_overlap!(connection)
    pairs = inherited_overlaps(connection)
    return if pairs.empty?

    raise InheritedOverlap, "horários sobrepostos herdados — decida cada par antes de migrar: " +
                            pairs.map { |a, b| "#{a} × #{b}" }.join("; ")
  end

  def up
    enable_extension "btree_gist" unless extension_enabled?("btree_gist")

    create_table :appointment_types, id: :uuid do |t|
      t.string :key, null: false
      t.string :name, null: false
      t.integer :duration_minutes, null: false
      t.string :cbo_prefixes, array: true, null: false, default: []
      t.boolean :active, null: false, default: true
      t.string :origin, null: false
      t.integer :position, null: false, default: 100
      t.timestamps
    end
    add_index :appointment_types, :key, unique: true
    add_check_constraint :appointment_types, "key::text ~ '^[a-z][a-z0-9_]{1,40}$'::text", name: "ck_appointment_types_key"
    add_check_constraint :appointment_types, "duration_minutes >= 5 AND duration_minutes <= 240",
                         name: "ck_appointment_types_duration"
    add_check_constraint :appointment_types, "origin::text = ANY (ARRAY['platform', 'city']::text[])",
                         name: "ck_appointment_types_origin"
    add_check_constraint :appointment_types, "cardinality(cbo_prefixes) >= 1 AND cardinality(cbo_prefixes) <= 20",
                         name: "ck_appointment_types_cbo_prefixes"
    copy_base_types

    create_table :schedule_templates, id: :uuid do |t|
      t.string :name, null: false
      t.integer :fit_in_limit, null: false, default: 2
      t.jsonb :blocks, null: false, default: []
      t.boolean :active, null: false, default: true
      t.timestamps
    end
    add_check_constraint :schedule_templates, "fit_in_limit >= 0 AND fit_in_limit <= 20",
                         name: "ck_schedule_templates_fit_in_limit"
    add_check_constraint :schedule_templates, "jsonb_typeof(blocks) = 'array'::text", name: "ck_schedule_templates_blocks"

    add_reference :professional_shifts, :schedule_template, type: :uuid, foreign_key: true, index: true
    add_column :professional_links, :default_appointment_type_key, :string
    add_column :city_profile, :default_fit_in_limit, :integer, null: false, default: 2
    add_check_constraint :city_profile, "default_fit_in_limit >= 0 AND default_fit_in_limit <= 20",
                         name: "ck_city_profile_default_fit_in_limit"

    change_requests
    change_appointments

    create_table :appointment_request_triages, id: :uuid do |t|
      t.references :request, type: :uuid, null: false, foreign_key: { to_table: :appointment_requests }, index: false
      t.references :triage, type: :uuid, null: false, foreign_key: true, index: true
      t.datetime :created_at, null: false
    end
    add_index :appointment_request_triages, %i[request_id triage_id], unique: true,
              name: "idx_appointment_request_triages_pair"

    create_table :appointment_notices, id: :uuid do |t|
      t.references :appointment, type: :uuid, null: false, foreign_key: true, index: { unique: true }
      t.references :citizen, type: :uuid, null: false, foreign_key: true, index: true
      t.datetime :read_at
      t.datetime :created_at, null: false
    end

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  # Pedidos e horários só aceitam acréscimo, e os guardas de db/city_triggers.sql
  # passam a citar as colunas novas.
  def down
    raise ActiveRecord::IrreversibleMigration, "horários e pedidos só aceitam acréscimo; os guardas dependem destas colunas"
  end

  private

  def change_requests
    # O guarda recusa UPDATE em pedido encerrado; o backfill de due_on passa por
    # todos. O city_triggers.sql do fim recria o trigger.
    execute "DROP TRIGGER IF EXISTS appointment_requests_guard ON appointment_requests"

    add_reference :appointment_requests, :origin_triage, type: :uuid, foreign_key: { to_table: :triages }, index: true
    change_column_null :appointment_requests, :origin_attendance_id, true
    change_column_null :appointment_requests, :origin_unit_id, true
    change_column_null :appointment_requests, :target_unit_id, true
    add_column :appointment_requests, :appointment_type_key, :string, null: false, default: "retorno"
    add_column :appointment_requests, :priority, :string, null: false, default: "routine"
    add_column :appointment_requests, :due_on, :date
    execute "UPDATE appointment_requests SET due_on = (created_at::date + 30)"
    change_column_null :appointment_requests, :due_on, false
    add_column :appointment_requests, :reschedule_reason_code, :string
    add_column :appointment_requests, :reschedule_note, :text
    add_column :appointment_requests, :preferred_period, :string
    add_column :appointment_requests, :reschedule_count, :integer, null: false, default: 0

    replace_check :appointment_requests, "ck_appointment_requests_kind",
                  "kind::text = ANY (ARRAY['return', 'referral', 'triage']::text[])"
    replace_check :appointment_requests, "ck_appointment_requests_closed_reason",
                  "closed_reason IS NULL OR closed_reason::text = ANY (ARRAY['fulfilled', 'citizen_cancelled', " \
                  "'dismissed', 'moved', 'consent_revoked']::text[])"
    replace_check :appointment_requests, "ck_appointment_requests_reopened_reason",
                  "reopened_reason IS NULL OR reopened_reason::text = ANY (ARRAY['expired', 'no_show', " \
                  "'citizen_reschedule']::text[])"
    add_check_constraint :appointment_requests, "(origin_attendance_id IS NULL) <> (origin_triage_id IS NULL)",
                         name: "ck_appointment_requests_origin"
    add_check_constraint :appointment_requests, "(kind::text = 'triage'::text) = (origin_triage_id IS NOT NULL)",
                         name: "ck_appointment_requests_triage_kind"
    add_check_constraint :appointment_requests,
                         "origin_attendance_id IS NULL OR (origin_unit_id IS NOT NULL AND target_unit_id IS NOT NULL)",
                         name: "ck_appointment_requests_attendance_units"
    add_check_constraint :appointment_requests, "priority::text = ANY (ARRAY['routine', 'priority']::text[])",
                         name: "ck_appointment_requests_priority"
    add_check_constraint :appointment_requests,
                         "preferred_period IS NULL OR preferred_period::text = ANY (ARRAY['morning', 'afternoon', 'any']::text[])",
                         name: "ck_appointment_requests_preferred_period"
    add_check_constraint :appointment_requests,
                         "reschedule_reason_code IS NULL OR reschedule_reason_code::text = ANY " \
                         "(ARRAY['work', 'health', 'transport', 'other']::text[])",
                         name: "ck_appointment_requests_reschedule_reason_code"
    add_check_constraint :appointment_requests, "reschedule_note IS NULL OR length(reschedule_note) <= 200",
                         name: "ck_appointment_requests_reschedule_note"
    add_check_constraint :appointment_requests, "reschedule_count >= 0", name: "ck_appointment_requests_reschedule_count"
    add_index :appointment_requests, %i[citizen_id appointment_type_key], unique: true,
              where: "kind::text = 'triage'::text AND status::text = ANY (ARRAY['open', 'scheduled']::text[])",
              name: "idx_appointment_requests_one_live_triage_type"
    add_index :appointment_requests, %i[target_unit_id status due_on], name: "idx_appointment_requests_queue"
  end

  def change_appointments
    add_reference :appointments, :professional, type: :uuid, foreign_key: true, index: false
    add_column :appointments, :appointment_type_key, :string
    add_column :appointments, :ends_at, :datetime
    add_reference :appointments, :shift, type: :uuid, foreign_key: { to_table: :professional_shifts }, index: true
    add_column :appointments, :booking_kind, :string, null: false, default: "legacy"
    add_column :appointments, :fit_in_reason, :text
    add_column :appointments, :reschedule_requested, :boolean, null: false, default: false
    add_column :appointments, :reminded_at, :datetime
    add_index :appointments, %i[professional_id scheduled_at], name: "idx_appointments_professional_time"
    add_index :appointments, %i[citizen_id scheduled_at], name: "idx_appointments_citizen_time"
    add_check_constraint :appointments, "booking_kind::text = ANY (ARRAY['slot', 'fit_in', 'legacy']::text[])",
                         name: "ck_appointments_booking_kind"
    add_check_constraint :appointments,
                         "booking_kind::text = 'legacy'::text OR (professional_id IS NOT NULL AND " \
                         "appointment_type_key IS NOT NULL AND ends_at IS NOT NULL AND shift_id IS NOT NULL)",
                         name: "ck_appointments_booking_fields"
    add_check_constraint :appointments, "ends_at IS NULL OR ends_at > scheduled_at", name: "ck_appointments_ends"
    add_check_constraint :appointments,
                         "((booking_kind::text = 'fit_in'::text) = (fit_in_reason IS NOT NULL)) AND " \
                         "(fit_in_reason IS NULL OR length(btrim(fit_in_reason)) >= 10)",
                         name: "ck_appointments_fit_in_reason"

    self.class.assert_no_inherited_overlap!(connection)
    add_exclusion_constraint :appointments, "professional_id WITH =, tsrange(scheduled_at, ends_at) WITH &&",
                             using: :gist, where: "booking_kind::text = 'slot'::text AND #{ACTIVE}",
                             name: "excl_appointments_slot_overlap"
  end

  # Só insere o que falta (ON CONFLICT): rodar de novo não desfaz ajuste da cidade.
  def copy_base_types
    YAML.load_file(BASE_PATH).each_with_index do |type, index|
      prefixes = type.fetch("cbo_prefixes").map { |p| connection.quote(p.to_s) }.join(", ")
      execute <<~SQL
        INSERT INTO appointment_types (key, name, duration_minutes, cbo_prefixes, active, origin, position, created_at, updated_at)
        VALUES (#{connection.quote(type.fetch('key'))}, #{connection.quote(type.fetch('name'))},
                #{Integer(type.fetch('duration_minutes'))}, ARRAY[#{prefixes}]::varchar[], true, 'platform', #{index + 1},
                now(), now())
        ON CONFLICT (key) DO NOTHING
      SQL
    end
  end

  def replace_check(table, name, expression)
    remove_check_constraint table, name: name
    add_check_constraint table, expression, name: name
  end
end
```

- [ ] **Step 7: Atualize `db/city_triggers.sql`**

Na função `rota_appointment_request_guard`, troque a linha `OR NEW.target_unit_id IS DISTINCT FROM OLD.target_unit_id` por

```sql
     OR (OLD.target_unit_id IS NOT NULL AND NEW.target_unit_id IS DISTINCT FROM OLD.target_unit_id)
     OR NEW.origin_triage_id IS DISTINCT FROM OLD.origin_triage_id
```

e acrescente ao comentário acima dela: `-- Módulo 17 (ADR 0029): a unidade de destino nula (fila "sem unidade") recebe uma unidade UMA vez; depois nunca muda.`

Na função `rota_appointment_guard`, antes de `OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN`, acrescente

```sql
     OR NEW.professional_id IS DISTINCT FROM OLD.professional_id
     OR NEW.appointment_type_key IS DISTINCT FROM OLD.appointment_type_key
     OR NEW.ends_at IS DISTINCT FROM OLD.ends_at
     OR NEW.shift_id IS DISTINCT FROM OLD.shift_id
     OR NEW.booking_kind IS DISTINCT FROM OLD.booking_kind
     OR NEW.fit_in_reason IS DISTINCT FROM OLD.fit_in_reason
```

No fim do arquivo:

```sql
-- appointment_types (ADR 0029; spec 2026-10-05 §3.1): o tipo da plataforma e o
-- da cidade nunca mudam de key nem de origem; desativar em vez de apagar
-- (pedidos e horários guardam a key). Sem trigger de TRUNCATE (limpeza de suíte).
CREATE OR REPLACE FUNCTION rota_appointment_type_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'appointment_types: DELETE refused (deactivate instead)';
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id OR NEW.key IS DISTINCT FROM OLD.key OR NEW.origin IS DISTINCT FROM OLD.origin THEN
    RAISE EXCEPTION 'appointment_types: key and origin never change';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- appointment_notices (ADR 0029 §6): o aviso de lembrete só registra a
-- primeira leitura; DELETE passa de propósito (exclusão do cadastro, ADR 0026).
CREATE OR REPLACE FUNCTION rota_appointment_notice_guard() RETURNS trigger AS $fn$
BEGIN
  IF NEW.id IS DISTINCT FROM OLD.id OR NEW.appointment_id IS DISTINCT FROM OLD.appointment_id
     OR NEW.citizen_id IS DISTINCT FROM OLD.citizen_id OR NEW.created_at IS DISTINCT FROM OLD.created_at
     OR (OLD.read_at IS NOT NULL AND NEW.read_at IS DISTINCT FROM OLD.read_at) THEN
    RAISE EXCEPTION 'appointment_notices: only the first read is recorded';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF to_regclass('public.appointment_types') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS appointment_types_guard ON appointment_types';
    EXECUTE 'CREATE TRIGGER appointment_types_guard
      BEFORE UPDATE OR DELETE ON appointment_types
      FOR EACH ROW EXECUTE FUNCTION rota_appointment_type_guard()';
  END IF;
  -- Modelo: desativar em vez de apagar (turnos apontam para ele).
  IF to_regclass('public.schedule_templates') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS schedule_templates_no_delete ON schedule_templates';
    EXECUTE 'CREATE TRIGGER schedule_templates_no_delete
      BEFORE DELETE ON schedule_templates
      FOR EACH ROW EXECUTE FUNCTION rota_append_only()';
  END IF;
  -- Ligação pedido↔triagem (fusão de triagens num pedido): só acréscimo.
  IF to_regclass('public.appointment_request_triages') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS appointment_request_triages_append_only ON appointment_request_triages';
    EXECUTE 'CREATE TRIGGER appointment_request_triages_append_only
      BEFORE UPDATE OR DELETE ON appointment_request_triages
      FOR EACH ROW EXECUTE FUNCTION rota_append_only()';
    EXECUTE 'DROP TRIGGER IF EXISTS appointment_request_triages_append_only_truncate ON appointment_request_triages';
    EXECUTE 'CREATE TRIGGER appointment_request_triages_append_only_truncate
      BEFORE TRUNCATE ON appointment_request_triages
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
  END IF;
  IF to_regclass('public.appointment_notices') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS appointment_notices_guard ON appointment_notices';
    EXECUTE 'CREATE TRIGGER appointment_notices_guard
      BEFORE UPDATE ON appointment_notices
      FOR EACH ROW EXECUTE FUNCTION rota_appointment_notice_guard()';
  END IF;
END
$do$;
```

- [ ] **Step 8: Atualize `db/city_schema.rb` à mão**

`define(version: 2026_10_08_100001)`. Tabelas novas, em ordem alfabética (`appointment_notices` antes de `appointment_reminders`; `appointment_request_triages` antes de `appointment_requests`; `appointment_types` depois de `appointment_requests`; `schedule_templates` depois de `report_snapshots`):

```ruby
  create_table "appointment_notices", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "appointment_id", null: false
    t.uuid "citizen_id", null: false
    t.datetime "created_at", null: false
    t.datetime "read_at"
    t.index ["appointment_id"], name: "index_appointment_notices_on_appointment_id", unique: true
    t.index ["citizen_id"], name: "index_appointment_notices_on_citizen_id"
  end

  create_table "appointment_request_triages", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "created_at", null: false
    t.uuid "request_id", null: false
    t.uuid "triage_id", null: false
    t.index ["request_id", "triage_id"], name: "idx_appointment_request_triages_pair", unique: true
    t.index ["triage_id"], name: "index_appointment_request_triages_on_triage_id"
  end

  create_table "appointment_types", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.boolean "active", default: true, null: false
    t.string "cbo_prefixes", default: [], null: false, array: true
    t.datetime "created_at", null: false
    t.integer "duration_minutes", null: false
    t.string "key", null: false
    t.string "name", null: false
    t.string "origin", null: false
    t.integer "position", default: 100, null: false
    t.datetime "updated_at", null: false
    t.index ["key"], name: "index_appointment_types_on_key", unique: true
    t.check_constraint "cardinality(cbo_prefixes) >= 1 AND cardinality(cbo_prefixes) <= 20", name: "ck_appointment_types_cbo_prefixes"
    t.check_constraint "duration_minutes >= 5 AND duration_minutes <= 240", name: "ck_appointment_types_duration"
    t.check_constraint "key::text ~ '^[a-z][a-z0-9_]{1,40}$'::text", name: "ck_appointment_types_key"
    t.check_constraint "origin::text = ANY (ARRAY['platform', 'city']::text[])", name: "ck_appointment_types_origin"
  end

  create_table "schedule_templates", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.boolean "active", default: true, null: false
    t.jsonb "blocks", default: [], null: false
    t.datetime "created_at", null: false
    t.integer "fit_in_limit", default: 2, null: false
    t.string "name", null: false
    t.datetime "updated_at", null: false
    t.check_constraint "fit_in_limit >= 0 AND fit_in_limit <= 20", name: "ck_schedule_templates_fit_in_limit"
    t.check_constraint "jsonb_typeof(blocks) = 'array'::text", name: "ck_schedule_templates_blocks"
  end
```

Em `appointment_requests`: `origin_attendance_id`, `origin_unit_id` e `target_unit_id` sem `null: false`; colunas `t.string "appointment_type_key", default: "retorno", null: false`, `t.date "due_on", null: false`, `t.uuid "origin_triage_id"`, `t.string "preferred_period"`, `t.string "priority", default: "routine", null: false`, `t.integer "reschedule_count", default: 0, null: false`, `t.text "reschedule_note"`, `t.string "reschedule_reason_code"` (ordem alfabética); índices `idx_appointment_requests_one_live_triage_type` e `idx_appointment_requests_queue` e `index_appointment_requests_on_origin_triage_id`; os três CHECKs trocados e os oito novos com as expressões do Step 6. Em `appointments`: `t.string "appointment_type_key"`, `t.string "booking_kind", default: "legacy", null: false`, `t.datetime "ends_at"`, `t.text "fit_in_reason"`, `t.uuid "professional_id"`, `t.datetime "reminded_at"`, `t.boolean "reschedule_requested", default: false, null: false`, `t.uuid "shift_id"`; índices `idx_appointments_professional_time`, `idx_appointments_citizen_time`, `index_appointments_on_shift_id`; os quatro CHECKs; e

```ruby
    t.exclusion_constraint "professional_id WITH =, tsrange(scheduled_at, ends_at) WITH &&", where: "(booking_kind::text = 'slot'::text AND status::text = ANY (ARRAY['scheduled', 'confirmed', 'checked_in']::text[]))", using: :gist, name: "excl_appointments_slot_overlap"
```

Em `city_profile`: `t.integer "default_fit_in_limit", default: 2, null: false` e o CHECK. Em `professional_links`: `t.string "default_appointment_type_key"`. Em `professional_shifts`: `t.uuid "schedule_template_id"` e `t.index ["schedule_template_id"], name: "index_professional_shifts_on_schedule_template_id"`. Chaves estrangeiras, na ordem alfabética do bloco final:

```ruby
  add_foreign_key "appointment_notices", "appointments"
  add_foreign_key "appointment_notices", "citizens"
  add_foreign_key "appointment_request_triages", "appointment_requests", column: "request_id"
  add_foreign_key "appointment_request_triages", "triages"
  add_foreign_key "appointment_requests", "triages", column: "origin_triage_id"
  add_foreign_key "appointments", "professional_shifts", column: "shift_id"
  add_foreign_key "appointments", "professionals"
  add_foreign_key "professional_shifts", "schedule_templates"
```

A paridade compara o schema normalizado pelo Postgres (não o texto); confira no Step 10.

- [ ] **Step 9: Modelos, eventos declarados e guardas de spec existentes**

```ruby
# app/models/appointment_type.rb
# Tipo de atendimento da cidade (ADR 0029 §3.1): base da plataforma copiada
# (origin platform) e tipos da cidade (origin city). Desativar nunca quebra
# pedido nem horário: eles guardam a key. key e origin nunca mudam (trigger).
class AppointmentType < ApplicationRecord
  ORIGINS = %w[platform city].freeze
  KEY = /\A[a-z][a-z0-9_]{1,40}\z/

  scope :active_types, -> { where(active: true) }
  scope :listed, -> { order(:position, :name) }

  def platform? = origin == "platform"
end
```

```ruby
# app/models/schedule_template.rb
# Modelo de agenda (ADR 0029 §3.2): faixas walk_in/bookable/blocked em hora
# local e o limite de encaixes do turno. Validado por Scheduling::TemplateBlocks.
class ScheduleTemplate < ApplicationRecord
  has_many :shifts, class_name: "ProfessionalShift", dependent: :restrict_with_error
end
```

```ruby
# app/models/appointment_request_triage.rb
# Triagem que caiu num pedido vivo do mesmo tipo (ADR 0029 §5.2): só acréscimo.
class AppointmentRequestTriage < ApplicationRecord
  belongs_to :request, class_name: "AppointmentRequest", inverse_of: :request_triages
  belongs_to :triage
end
```

```ruby
# app/models/appointment_notice.rb
# Aviso de lembrete na caixa do cidadão (ADR 0029 §6; contratos §5, §8). Só a
# primeira leitura é gravada (trigger); a exclusão do cadastro apaga.
class AppointmentNotice < ApplicationRecord
  belongs_to :appointment
  belongs_to :citizen
end
```

Em `app/models/appointment.rb`, depois de `MAX_AHEAD`:

```ruby
  BOOKING_KINDS = %w[slot fit_in legacy].freeze
  # "Ativo" para a trava e para as vagas (ADR 0029): o check-in também ocupa.
  ACTIVE = %w[scheduled confirmed checked_in].freeze
  # Horário livre (legacy) não tem fim; para "cidadão sem dois horários
  # sobrepostos" ele ocupa 15 minutos.
  LEGACY_SPAN = 15.minutes
  MIN_FIT_IN_REASON = 10
  RESCHEDULE_CANCEL_REASON = "Remarcação pedida pelo cidadão".freeze
```

e, depois de `has_one :attendance`:

```ruby
  belongs_to :professional, optional: true
  belongs_to :shift, class_name: "ProfessionalShift", optional: true
  has_one :notice, class_name: "AppointmentNotice", dependent: :restrict_with_error

  def fit_in? = booking_kind == "fit_in"
  def effective_ends_at = ends_at || scheduled_at + LEGACY_SPAN
```

`app/models/appointment_request.rb` inteiro:

```ruby
# Pedido de agendamento (ADR 0019, ADR 0029): nasce do desfecho return/referred
# com unidade ou da triagem (kind triage, regra do protocolo); a recepção da
# unidade de destino marca o horário. Só acréscimo (trigger); a origem nunca
# muda; a unidade de destino nula recebe uma unidade uma vez.
class AppointmentRequest < ApplicationRecord
  KINDS = %w[return referral triage].freeze
  STATUSES = %w[open scheduled closed].freeze
  CLOSED_REASONS = %w[fulfilled citizen_cancelled dismissed moved consent_revoked].freeze
  PRIORITIES = %w[routine priority].freeze
  PERIODS = %w[morning afternoon any].freeze
  RESCHEDULE_REASONS = %w[work health transport other].freeze
  DUE_IN_DAYS = 30

  belongs_to :origin_attendance, class_name: "Attendance", inverse_of: :appointment_request, optional: true
  belongs_to :origin_triage, class_name: "Triage", optional: true
  belongs_to :citizen
  belongs_to :root_triage, class_name: "Triage"
  belongs_to :origin_unit, class_name: "HealthUnit", optional: true
  belongs_to :target_unit, class_name: "HealthUnit", optional: true
  belongs_to :closed_by_user, class_name: "User", optional: true
  # Pedido movido de unidade (api#29): o novo aponta para o antigo.
  belongs_to :moved_from_request, class_name: "AppointmentRequest", optional: true
  has_many :appointments, foreign_key: :request_id, inverse_of: :request, dependent: :restrict_with_error
  has_many :request_triages, class_name: "AppointmentRequestTriage", foreign_key: :request_id, inverse_of: :request,
                             dependent: :restrict_with_error

  # Sem data indicada no desfecho (o atendimento não registra uma): +30 dias.
  attribute :due_on, :date, default: -> { Time.zone.today + DUE_IN_DAYS }

  scope :live_requests, -> { where(status: %w[open scheduled]) }

  def latest_appointment
    appointments.max_by(&:created_at)
  end

  def origin = origin_triage_id ? "triage" : "attendance"
end
```

Em `app/models/professional_shift.rb`, `belongs_to :schedule_template, optional: true`.

Em `config/initializers/domain_events.rb`, depois de `DomainEvents.bind "health_unit.drained", to: []`:

```ruby
  # Agenda dos profissionais (ADR 0029; contratos §6): trilha, só ids, sem consumidor.
  DomainEvents.bind "appointment.booked", to: []
  DomainEvents.bind "appointment.fit_in_created", to: []
  DomainEvents.bind "appointment.reschedule_requested", to: []
  DomainEvents.bind "appointment.reminded", to: []
  DomainEvents.bind "appointment_request.created_from_triage", to: []
  DomainEvents.bind "appointment_request.merged_triage", to: []
  DomainEvents.bind "appointment_request.unit_assigned", to: []
  DomainEvents.bind "appointment_type.changed", to: []
  DomainEvents.bind "schedule_template.changed", to: []
  DomainEvents.bind "professional.shift_template_set", to: []
  DomainEvents.bind "professional.link_default_type_set", to: []
```

No fim de `spec/initializers/domain_events_bindings_spec.rb`:

```ruby
# Módulo 17 (ADR 0029): eventos da agenda declarados, só trilha.
RSpec.describe "scheduling event bindings (ADR 0029)" do
  it "declares every scheduling event with no consumer" do
    names = %w[appointment.booked appointment.fit_in_created appointment.reschedule_requested appointment.reminded
               appointment_request.created_from_triage appointment_request.merged_triage
               appointment_request.unit_assigned appointment_type.changed schedule_template.changed
               professional.shift_template_set professional.link_default_type_set]
    expect(DomainEvents.registry.keys).to include(*names)
    expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
  end
end
```

Em `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..29).freeze`.

Em `spec/invariants/appointment_invariants_spec.rb`, o exemplo "o banco recusa pedido sem atendimento de origem" passa a esperar a CHECK nova (a coluna ficou nula para o pedido da triagem):

```ruby
    it "o banco recusa pedido sem nenhuma origem (nem atendimento, nem triagem)" do
      req = travel_to(t0) { request_from_return }
      row = req.attributes.except("id", "origin_attendance_id").merge("origin_attendance_id" => nil)
      expect { attempt { AppointmentRequest.insert_all!([ row ]) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_appointment_requests_origin/)
    end
```

- [ ] **Step 10: Bancos de teste, specs e commit**

```bash
psql -U rota_saude -d postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b"
docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod17 api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/models/scheduling_tables_guard_spec.rb spec/models/add_professional_schedules_migration_spec.rb spec/services/city_schema_spec.rb spec/services/protocols/scheduling_targets_spec.rb spec/requests/authoring_scheduling_gate_spec.rb spec/invariants/appointment_invariants_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/adr_pointers_spec.rb spec/models/appointment_request_spec.rb spec/models/appointment_spec.rb spec/requests/appointment_requests_spec.rb spec/commands/appointments spec/commands/health_units
```
Expected: PASS. (O Postgres é o do host, fora do compose; não encoste em `rota_saude_no_city_selected` nem nos bancos de dev.) Se `city_schema_spec` acusar diferença, o `city_schema.rb` está fora da migração: compare `\d appointments` nos dois bancos e corrija o dump, nunca a migração já testada pelo guard spec.

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add config/scheduling/appointment_types.yml db/city_migrate/20261008100001_add_professional_schedules.rb db/city_schema.rb db/city_triggers.sql app/models/appointment_type.rb app/models/schedule_template.rb app/models/appointment_request_triage.rb app/models/appointment_notice.rb app/models/appointment.rb app/models/appointment_request.rb app/models/professional_shift.rb config/initializers/domain_events.rb spec/initializers/domain_events_bindings_spec.rb spec/support/scheduling_helpers.rb spec/rails_helper.rb spec/adr_pointers_spec.rb spec/invariants/appointment_invariants_spec.rb spec/models/scheduling_tables_guard_spec.rb spec/models/add_professional_schedules_migration_spec.rb app/services/protocols/scheduling_targets.rb app/controllers/authoring/protocols_controller.rb spec/services/protocols/scheduling_targets_spec.rb spec/requests/authoring_scheduling_gate_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: add professional schedule tables and the slot overlap lock

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Base de tipos e catálogo (`Scheduling::AppointmentTypes`)

**Files:**
- Create: `app/services/scheduling/appointment_types.rb`
- Modify: `app/jobs/provision_city_job.rb`
- Modify: `spec/support/scheduling_helpers.rb`
- Test: `spec/services/scheduling/appointment_types_spec.rb`, `spec/jobs/provision_city_job_spec.rb`

**Interfaces:**
- Consumes: `AppointmentType` (Task 2), `config/scheduling/appointment_types.yml`.
- Produces:
  - `Scheduling::AppointmentTypes.base -> Array<Hash>` (yml), `.seed_platform!(now: Time.current) -> Integer` (insere as que faltam, devolve quantas), `.serves?(type, cbo_code) -> Boolean` (`type` responde a `cbo_prefixes`), `.catalog -> Catalog`.
  - `Scheduling::AppointmentTypes::Catalog#find(key) -> AppointmentType|nil`, `#name_for(key) -> String|nil` (nome, ou a própria key quando não existe; `nil` para key nula), `#active -> Hash{key => Scheduling::Availability::Type}`, `#fallback -> Array<Scheduling::Availability::Type>` (tipos `platform` ativos na ordem da base).
  - `Scheduling::Availability::Type = Data.define(:key, :duration_minutes, :cbo_prefixes)` com `#serves?(cbo)` (definido aqui, num arquivo próprio, porque a Task 5 o reaproveita): `app/services/scheduling/availability/type.rb`.
  - Helper de spec: `ensure_appointment_types!` (chama `seed_platform!`).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/scheduling/appointment_types_spec.rb
require "rails_helper"

# ADR 0029 §3.1: a base é copiada (só o que falta), a cidade ajusta e o
# catálogo resolve nome, tipos ativos e o tipo padrão por CBO.
RSpec.describe Scheduling::AppointmentTypes do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  it "copia a base uma vez e nunca sobrescreve o ajuste da cidade" do
    expect(described_class.seed_platform!).to eq(4)
    AppointmentType.find_by!(key: "consulta_medica").update!(duration_minutes: 30, active: false)
    expect(described_class.seed_platform!).to eq(0)
    expect(AppointmentType.find_by!(key: "consulta_medica")).to have_attributes(duration_minutes: 30, active: false)
    expect(AppointmentType.listed.pluck(:key, :origin, :position)).to eq([
      [ "consulta_medica", "platform", 1 ], [ "consulta_enfermagem", "platform", 2 ],
      [ "consulta_odontologica", "platform", 3 ], [ "retorno", "platform", 4 ]
    ])
  end

  it "serves? casa pelo prefixo do CBO" do
    described_class.seed_platform!
    medica = AppointmentType.find_by!(key: "consulta_medica")
    expect(described_class.serves?(medica, "225125")).to be(true)
    expect(described_class.serves?(medica, "223505")).to be(false)
    expect(described_class.serves?(AppointmentType.find_by!(key: "retorno"), "223208")).to be(true)
  end

  it "catálogo: nome (ou a key), ativos e o padrão da base na ordem do arquivo" do
    described_class.seed_platform!
    type_row!("acupuntura", cbo: ["2251"], active: false)
    catalog = described_class.catalog
    expect(catalog.name_for("consulta_enfermagem")).to eq("Consulta de enfermagem")
    expect(catalog.name_for("fantasma")).to eq("fantasma")
    expect(catalog.name_for(nil)).to be_nil
    expect(catalog.active.keys).to contain_exactly("consulta_medica", "consulta_enfermagem", "consulta_odontologica", "retorno")
    expect(catalog.fallback.map(&:key)).to eq(%w[consulta_medica consulta_enfermagem consulta_odontologica retorno])
    expect(catalog.fallback.find { |t| t.serves?("223505") }.key).to eq("consulta_enfermagem")
  end
end
```

Em `spec/jobs/provision_city_job_spec.rb`, no exemplo "creates database and role, migrates, seeds the city, activates it and audits once", dentro do `CityConnection.with(city) do`, acrescente:

```ruby
      expect(AppointmentType.where(origin: "platform").order(:position).pluck(:key))
        .to eq(%w[consulta_medica consulta_enfermagem consulta_odontologica retorno])
```

e, no exemplo "stays provisioning after a failure and resumes without duplicating anything", `expect(AppointmentType.count).to eq(4)` ao lado da contagem existente.

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/services/scheduling/appointment_types_spec.rb`
Expected: FAIL com `uninitialized constant Scheduling::AppointmentTypes`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/scheduling/availability/type.rb
# Tipo de atendimento como dado puro (ADR 0029 §4.1): o que o cálculo de vagas
# precisa saber. Grupos de CBO são prefixos.
module Scheduling
  module Availability
    Type = Data.define(:key, :duration_minutes, :cbo_prefixes) do
      def serves?(cbo_code) = cbo_prefixes.any? { |prefix| cbo_code.to_s.start_with?(prefix) }
    end
  end
end
```

```ruby
# app/services/scheduling/appointment_types.rb
# Base de tipos da plataforma e catálogo da cidade (ADR 0029 §3.1). A base vem
# de config/scheduling/appointment_types.yml; a migração 20261008100001 e o
# provisionamento a copiam só inserindo o que falta.
module Scheduling
  module AppointmentTypes
    PATH = Rails.root.join("config/scheduling/appointment_types.yml")

    class Catalog
      def initialize(records)
        @records = records
        @by_key = records.index_by(&:key)
      end

      def find(key) = @by_key[key.to_s]

      def name_for(key)
        return nil if key.nil?

        find(key)&.name || key.to_s
      end

      def active = @records.select(&:active).to_h { |r| [ r.key, data(r) ] }

      def fallback = @records.select { |r| r.active && r.platform? }.sort_by(&:position).map { |r| data(r) }

      private

      def data(record)
        Availability::Type.new(key: record.key, duration_minutes: record.duration_minutes,
                               cbo_prefixes: Array(record.cbo_prefixes))
      end
    end

    module_function

    def base = @base ||= YAML.load_file(PATH).map(&:freeze).freeze

    def seed_platform!(now: Time.current)
      rows = base.each_with_index.map do |type, index|
        { key: type.fetch("key"), name: type.fetch("name"), duration_minutes: type.fetch("duration_minutes"),
          cbo_prefixes: type.fetch("cbo_prefixes").map(&:to_s), active: true, origin: "platform",
          position: index + 1, created_at: now, updated_at: now }
      end
      AppointmentType.insert_all(rows, unique_by: :key).rows.size
    end

    def serves?(type, cbo_code) = Array(type.cbo_prefixes).any? { |prefix| cbo_code.to_s.start_with?(prefix) }

    def catalog = Catalog.new(AppointmentType.listed.to_a)
  end
end
```

`insert_all(..., unique_by: :key)` vira `ON CONFLICT (key) DO NOTHING ... RETURNING id`: `rows.size` é o número de linhas inseridas.

Em `app/jobs/provision_city_job.rb`, dentro de `seed`, depois do `SeedProtocol.call(...)`:

```ruby
          # ADR 0029 §3.1: a base de tipos (a migração já copiou; aqui é a
          # garantia idempotente do provisionamento).
          Scheduling::AppointmentTypes.seed_platform!
```

Em `spec/support/scheduling_helpers.rb`, dentro do módulo:

```ruby
  def ensure_appointment_types! = Scheduling::AppointmentTypes.seed_platform!
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/services/scheduling/appointment_types_spec.rb spec/jobs/provision_city_job_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/services/scheduling/availability/type.rb app/services/scheduling/appointment_types.rb app/jobs/provision_city_job.rb spec/support/scheduling_helpers.rb spec/services/scheduling/appointment_types_spec.rb spec/jobs/provision_city_job_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: copy the platform appointment types into each city

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 3 — Tipos, modelos e vagas (F-17.1, F-17.2)

### Task 4: Tipos de atendimento — comandos e `/professionals/appointment_types`

**Files:**
- Create: `app/commands/scheduling/save_appointment_type.rb`
- Create: `app/controllers/appointment_types_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/commands/scheduling/save_appointment_type_spec.rb`, `spec/requests/appointment_types_spec.rb`

**Interfaces:**
- Consumes: `AppointmentType` (Task 2), `ensure_appointment_types!` (Task 3).
- Produces:
  - `Scheduling::SaveAppointmentType.create(attrs:, by:) -> Result` (`payload[:type]`), `.update(type:, attrs:, by:) -> Result`; motivos `:invalid_key`, `:key_taken`, `:invalid_name`, `:invalid_duration`, `:invalid_cbo_prefixes`, `:platform_type_locked`, `:invalid`; evento `appointment_type.changed { key, user_id }`.
  - `AppointmentTypesController#type_json(type) -> { key, name, duration_minutes, cbo_prefixes, active, origin }` (contratos §3).
  - Rotas: `GET /professionals/appointment_types` (leitura: `municipal_admin`, `citizen_verifier`, `health_professional`, `protocol_author`, `protocol_reviewer` — contratos §3, §9), `POST /professionals/appointment_types`, `POST /professionals/appointment_types/:key` (só `municipal_admin`).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/commands/scheduling/save_appointment_type_spec.rb
require "rails_helper"

# ADR 0029 §3.1; contratos §3, §9: a cidade cria, ajusta e desativa; o tipo da
# plataforma não troca os grupos de CBO.
RSpec.describe Scheduling::SaveAppointmentType do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:admin) { staff_with("admin-tipos@cidade.gov.br", "municipal_admin") }
  let(:valid) { { "key" => "acupuntura", "name" => "Acupuntura", "duration_minutes" => 30, "cbo_prefixes" => ["2251", "2236"] } }

  it "cria tipo da cidade, com evento só de key e usuário" do
    type = described_class.create(attrs: valid, by: admin).payload[:type]
    expect(type).to have_attributes(key: "acupuntura", origin: "city", active: true, cbo_prefixes: ["2251", "2236"])
    expect(DomainEvent.where(name: "appointment_type.changed").last.payload).to eq("key" => "acupuntura", "user_id" => admin.id)
  end

  {
    { "key" => "Acupuntura" } => :invalid_key, { "key" => "a" } => :invalid_key, { "key" => nil } => :invalid_key,
    { "name" => "" } => :invalid_name, { "name" => "x" * 61 } => :invalid_name,
    { "duration_minutes" => 4 } => :invalid_duration, { "duration_minutes" => 241 } => :invalid_duration,
    { "duration_minutes" => "30" } => :invalid_duration,
    { "cbo_prefixes" => [] } => :invalid_cbo_prefixes, { "cbo_prefixes" => ["22a1"] } => :invalid_cbo_prefixes,
    { "cbo_prefixes" => ["1234567"] } => :invalid_cbo_prefixes, { "cbo_prefixes" => Array.new(21, "2251") } => :invalid_cbo_prefixes,
    { "cbo_prefixes" => "2251" } => :invalid_cbo_prefixes, { "active" => "sim" } => :invalid
  }.each do |change, reason|
    it "recusa #{change.inspect} com #{reason}" do
      expect(described_class.create(attrs: valid.merge(change), by: admin).reason).to eq(reason)
      expect(AppointmentType.where(key: "acupuntura")).to be_empty
    end
  end

  it "key repetida (inclusive da plataforma) é key_taken" do
    described_class.create(attrs: valid, by: admin)
    expect(described_class.create(attrs: valid, by: admin).reason).to eq(:key_taken)
    expect(described_class.create(attrs: valid.merge("key" => "retorno"), by: admin).reason).to eq(:key_taken)
  end

  it "ajusta duração, nome e ativo do tipo da plataforma; grupos de CBO travados" do
    medica = AppointmentType.find_by!(key: "consulta_medica")
    result = described_class.update(type: medica, attrs: { "duration_minutes" => 30, "active" => false, "name" => "Consulta" }, by: admin)
    expect(result.payload[:type]).to have_attributes(duration_minutes: 30, active: false, name: "Consulta", origin: "platform")
    expect(described_class.update(type: medica, attrs: { "cbo_prefixes" => ["2251"] }, by: admin).reason).to eq(:platform_type_locked)
  end

  it "tipo da cidade troca os grupos de CBO" do
    type = described_class.create(attrs: valid, by: admin).payload[:type]
    expect(described_class.update(type: type, attrs: { "cbo_prefixes" => ["2235"] }, by: admin).payload[:type].cbo_prefixes)
      .to eq(["2235"])
  end
end
```

```ruby
# spec/requests/appointment_types_spec.rb
require "rails_helper"

RSpec.describe "/professionals/appointment_types", type: :request do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:admin) { staff_with("admin-tipos@cidade.gov.br", "municipal_admin") }
  def body = JSON.parse(response.body)

  it "lista para quem lê (admin, recepção, profissional, autoria, revisão) e recusa quem não lê" do
    %w[municipal_admin citizen_verifier health_professional protocol_author protocol_reviewer].each do |role|
      sign_in_as(staff_with("#{role}-#{SecureRandom.hex(2)}@cidade.gov.br", role))
      get "/professionals/appointment_types"
      expect(response).to have_http_status(:ok), role
      expect(body["types"].first).to eq("key" => "consulta_medica", "name" => "Consulta médica", "duration_minutes" => 20,
                                        "cbo_prefixes" => ["2251", "2252", "2253"], "active" => true, "origin" => "platform")
    end
    sign_in_as(staff_with("viewer-#{SecureRandom.hex(2)}@cidade.gov.br", "viewer"))
    get "/professionals/appointment_types"
    expect(response).to have_http_status(:forbidden)
  end

  it "cria e ajusta (só admin); devolve o tipo puro; erros com o código do contrato" do
    sign_in_as(staff_with("recepcao-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier"))
    json_post "/professionals/appointment_types", key: "acupuntura", name: "Acupuntura", duration_minutes: 30, cbo_prefixes: ["2251"]
    expect(response).to have_http_status(:forbidden)

    sign_in_as(admin)
    json_post "/professionals/appointment_types", key: "acupuntura", name: "Acupuntura", duration_minutes: 30, cbo_prefixes: ["2251"]
    expect(response).to have_http_status(:created)
    expect(body).to include("key" => "acupuntura", "origin" => "city")

    json_post "/professionals/appointment_types/acupuntura", active: false
    expect(response).to have_http_status(:ok)
    expect(body["active"]).to be(false)

    json_post "/professionals/appointment_types/consulta_medica", cbo_prefixes: ["2251"]
    expect(response).to have_http_status(:unprocessable_entity)
    expect(body).to eq("error" => "platform_type_locked")

    json_post "/professionals/appointment_types", key: "acupuntura", name: "Outra", duration_minutes: 30, cbo_prefixes: ["2251"]
    expect(body).to eq("error" => "key_taken")

    json_post "/professionals/appointment_types/fantasma", active: false
    expect(response).to have_http_status(:not_found)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/scheduling/save_appointment_type_spec.rb spec/requests/appointment_types_spec.rb`
Expected: FAIL com `uninitialized constant Scheduling::SaveAppointmentType` e 404 nas rotas.

- [ ] **Step 3: Implemente o comando**

```ruby
# app/commands/scheduling/save_appointment_type.rb
# Tipos de atendimento da cidade (ADR 0029 §3.1; contratos §3, §9). Só o
# municipal_admin chega aqui (controller). O tipo da plataforma ajusta nome,
# duração e ativo; os grupos de CBO são da plataforma (platform_type_locked).
module Scheduling
  module SaveAppointmentType
    PREFIX = /\A\d{1,6}\z/
    MAX_NAME = 60
    MAX_PREFIXES = 20

    module_function

    def create(attrs:, by:)
      key = attrs["key"]
      return Result.fail(:invalid_key) unless key.is_a?(String) && key.match?(AppointmentType::KEY)

      values = { "name" => attrs["name"], "duration_minutes" => attrs["duration_minutes"],
                 "cbo_prefixes" => attrs["cbo_prefixes"], "active" => attrs.fetch("active", true) }
      reason = invalid(values)
      return Result.fail(reason) if reason

      type = ApplicationRecord.transaction(requires_new: true) do
        AppointmentType.create!(key: key, name: values["name"].squish, duration_minutes: values["duration_minutes"],
                                cbo_prefixes: values["cbo_prefixes"], active: values["active"], origin: "city")
      end
      DomainEvents.publish("appointment_type.changed", key: type.key, user_id: by.id)
      Result.ok(type: type)
    rescue ActiveRecord::RecordNotUnique
      Result.fail(:key_taken)
    end

    def update(type:, attrs:, by:)
      return Result.fail(:platform_type_locked) if type.platform? && attrs.key?("cbo_prefixes")

      values = attrs.slice("name", "duration_minutes", "cbo_prefixes", "active")
      reason = invalid(values)
      return Result.fail(reason) if reason

      changes = values.to_h { |k, v| [ k, k == "name" ? v.squish : v ] }
      ApplicationRecord.transaction do
        type.update!(changes)
        DomainEvents.publish("appointment_type.changed", key: type.key, user_id: by.id)
      end
      Result.ok(type: type)
    end

    # Só confere as chaves presentes (update parcial); create passa todas.
    def invalid(values)
      if values.key?("name")
        name = values["name"]
        return :invalid_name unless name.is_a?(String) && name.squish.length.between?(1, MAX_NAME)
      end
      if values.key?("duration_minutes")
        minutes = values["duration_minutes"]
        return :invalid_duration unless minutes.is_a?(Integer) && minutes.between?(5, 240)
      end
      if values.key?("cbo_prefixes")
        prefixes = values["cbo_prefixes"]
        ok = prefixes.is_a?(Array) && prefixes.size.between?(1, MAX_PREFIXES) &&
             prefixes.all? { |p| p.is_a?(String) && p.match?(PREFIX) }
        return :invalid_cbo_prefixes unless ok
      end
      return :invalid if values.key?("active") && ![ true, false ].include?(values["active"])

      nil
    end
  end
end
```

- [ ] **Step 4: Controller e rotas**

```ruby
# app/controllers/appointment_types_controller.rb
# Tipos de atendimento (ADR 0029 §3.1; contratos §3, §9). Leitura para quem
# monta agenda, marca, atende ou escreve protocolo; escrita só municipal_admin,
# sem step-up (tipo não muda quem pode fazer o quê).
class AppointmentTypesController < ApplicationController
  include Authentication
  include AttendanceAccess

  wrap_parameters false

  READERS = %w[municipal_admin citizen_verifier health_professional protocol_author protocol_reviewer].freeze
  ERROR_STATUS = {
    invalid_key: :unprocessable_entity, key_taken: :unprocessable_entity, invalid_name: :unprocessable_entity,
    invalid_duration: :unprocessable_entity, invalid_cbo_prefixes: :unprocessable_entity,
    platform_type_locked: :unprocessable_entity, invalid: :unprocessable_entity
  }.freeze

  before_action :require_reader, only: :index
  before_action :require_admin, except: :index

  def index
    render json: { types: AppointmentType.listed.map { |t| type_json(t) } }
  end

  def create
    result = Scheduling::SaveAppointmentType.create(attrs: body, by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: type_json(result.payload[:type]), status: :created
  end

  def update
    type = AppointmentType.find_by(key: params[:key].to_s)
    return render(json: { error: "not_found" }, status: :not_found) unless type

    result = Scheduling::SaveAppointmentType.update(type: type, attrs: body, by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: type_json(result.payload[:type])
  end

  private

  def require_reader
    forbid unless Current.user && Membership.active.where(user: Current.user, role: READERS).exists?
  end

  def body = request.request_parameters.to_h

  def type_json(t)
    { key: t.key, name: t.name, duration_minutes: t.duration_minutes, cbo_prefixes: Array(t.cbo_prefixes),
      active: t.active, origin: t.origin }
  end
end
```

Em `config/routes.rb`, no `scope "/professionals"`, **antes** de `get ":id"` (as literais vêm antes de `:id`):

```ruby
    # Agenda dos profissionais (ADR 0029; contratos §3).
    get  "appointment_types",      to: "appointment_types#index"
    post "appointment_types",      to: "appointment_types#create"
    post "appointment_types/:key", to: "appointment_types#update"
```

(`Membership.active` é o escopo que `spec/support/appointment_helpers.rb` já usa.)

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/scheduling/save_appointment_type_spec.rb spec/requests/appointment_types_spec.rb spec/requests/professionals_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/commands/scheduling/save_appointment_type.rb app/controllers/appointment_types_controller.rb config/routes.rb spec/commands/scheduling/save_appointment_type_spec.rb spec/requests/appointment_types_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: let the city manage appointment types

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: `Scheduling::Availability` — faixas e vagas, puro, com tabela de casos

**Files:**
- Create: `app/services/scheduling/availability.rb`
- Test: `spec/services/scheduling/availability_compute_spec.rb`

**Interfaces:**
- Consumes: `Scheduling::Availability::Type` (Task 3).
- Produces (tudo puro: nada de banco, recebe o fuso):
  - `Scheduling::Availability::Shift = Data.define(:id, :professional_id, :starts_at, :ends_at, :cbo_code, :default_type_key, :blocks, :cancelled)` — `blocks` é a lista do modelo (`Hash` com chaves texto) ou `nil` (sem modelo);
  - `::Busy = Data.define(:professional_id, :starts_at, :ends_at)`;
  - `::Slot = Data.define(:professional_id, :shift_id, :starts_at, :ends_at, :appointment_type_key)`;
  - `::Block = Data.define(:starts_at, :ends_at, :kind, :appointment_type_key, :slot_minutes)` (instantes já recortados pelo turno);
  - `.blocks_for(shift, types:, fallback:, zone:) -> Array<Block>` (`types` = `Hash{key => Type}` dos ativos; `fallback` = base ativa na ordem);
  - `.slice(block, minutes) -> Array<[Time, Time]>` (sobra descartada);
  - `.default_key(shift, types, fallback) -> String|nil`;
  - `.compute(shifts:, type:, types:, fallback:, busy:, now:, zone:, window: nil) -> Array<Slot>` (ordenado por início e profissional).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/scheduling/availability_compute_spec.rb
require "rails_helper"

# ADR 0029 §4.1 / spec §9: tabela de casos do cálculo de vagas, puro.
RSpec.describe Scheduling::Availability, ".compute" do
  let(:sp) { ActiveSupport::TimeZone["America/Sao_Paulo"] }
  let(:manaus) { ActiveSupport::TimeZone["America/Manaus"] }
  let(:day) { Date.new(2026, 10, 6) }

  def at(hour, min = 0, d: day, z: sp) = z.local(d.year, d.month, d.day, hour, min)
  def type(key, minutes, prefixes) = described_class::Type.new(key: key, duration_minutes: minutes, cbo_prefixes: prefixes)

  let(:medica) { type("consulta_medica", 20, %w[2251 2252 2253]) }
  let(:enfermagem) { type("consulta_enfermagem", 15, %w[2235]) }
  let(:retorno) { type("retorno", 15, %w[2251 2252 2253 2235 2232]) }
  let(:types) { [ medica, enfermagem, retorno ].index_by(&:key) }
  let(:fallback) { [ medica, enfermagem, retorno ] }

  def shift(starts:, ends:, cbo: "225125", default: nil, blocks: nil, cancelled: false, pro: "p1")
    described_class::Shift.new(id: "s-#{pro}", professional_id: pro, starts_at: starts, ends_at: ends, cbo_code: cbo,
                               default_type_key: default, blocks: blocks, cancelled: cancelled)
  end

  def block(starts, ends, kind, key = nil, slot = nil)
    { "starts" => starts, "ends" => ends, "kind" => kind, "appointment_type_key" => key, "slot_minutes" => slot }.compact
  end

  def hours(shifts, wanted, busy: [], now: at(12, d: day - 1), zone: sp, window: nil)
    described_class.compute(shifts: Array(shifts), type: wanted, types: types, fallback: fallback, busy: busy,
                            now: now, zone: zone, window: window)
                   .map { |s| s.starts_at.in_time_zone(zone).strftime("%d %H:%M") }
  end

  let(:morning) { shift(starts: at(8), ends: at(12)) }
  let(:three_blocks) do
    [ block("07:00", "09:00", "walk_in"), block("09:00", "11:00", "bookable", "consulta_medica"),
      block("11:00", "12:00", "blocked") ]
  end

  it "sem modelo, com tipo padrão do vínculo: o turno inteiro em vagas desse tipo" do
    expect(hours(shift(starts: at(8), ends: at(9), default: "retorno"), retorno))
      .to eq([ "06 08:00", "06 08:15", "06 08:30", "06 08:45" ])
  end

  it "sem modelo e sem padrão: o tipo da base pelo CBO; outro tipo não tem vaga" do
    expect(hours(shift(starts: at(8), ends: at(9)), medica)).to eq([ "06 08:00", "06 08:20", "06 08:40" ])
    expect(hours(shift(starts: at(8), ends: at(9)), retorno)).to eq([])
  end

  it "sem modelo e sem correspondência de CBO: nenhuma vaga (só encaixe)" do
    psicologo = shift(starts: at(8), ends: at(9), cbo: "251510")
    expect(described_class.blocks_for(psicologo, types: types, fallback: fallback, zone: sp)).to eq([])
    expect(hours(psicologo, retorno)).to eq([])
  end

  it "tipo padrão desativado (fora de types) cai para a base" do
    expect(hours(shift(starts: at(8), ends: at(9), default: "acupuntura"), medica)).to eq([ "06 08:00", "06 08:20", "06 08:40" ])
  end

  it "modelo com três faixas: só a agendável do tipo vira vaga" do
    expect(hours(shift(starts: at(7), ends: at(13), blocks: three_blocks), medica))
      .to eq([ "06 09:00", "06 09:20", "06 09:40", "06 10:00", "06 10:20", "06 10:40" ])
    expect(hours(shift(starts: at(7), ends: at(13), blocks: three_blocks), retorno)).to eq([])
  end

  it "faixa fora do turno é recortada pelo turno" do
    expect(hours(shift(starts: at(7), ends: at(13), blocks: [ block("06:00", "08:00", "bookable", "consulta_medica") ]), medica))
      .to eq([ "06 07:00", "06 07:20", "06 07:40" ])
  end

  it "sobra no fim da faixa é descartada; slot_minutes do modelo vale sobre a duração" do
    expect(hours(shift(starts: at(7), ends: at(13), blocks: [ block("09:00", "10:10", "bookable", "consulta_medica") ]), medica))
      .to eq([ "06 09:00", "06 09:20", "06 09:40" ])
    expect(hours(shift(starts: at(7), ends: at(13), blocks: [ block("09:00", "10:00", "bookable", "consulta_medica", 30) ]), medica))
      .to eq([ "06 09:00", "06 09:30" ])
  end

  it "CBO não servido pelo tipo: nenhuma vaga" do
    expect(hours(morning, enfermagem)).to eq([])
  end

  it "vaga cujo início já passou some" do
    expect(hours(shift(starts: at(7), ends: at(13), blocks: three_blocks), medica, now: at(9, 30)))
      .to eq([ "06 09:40", "06 10:00", "06 10:20", "06 10:40" ])
  end

  it "horário ativo do profissional (de qualquer modo) tira as vagas que cruza; o de outro profissional não" do
    busy = [ described_class::Busy.new(professional_id: "p1", starts_at: at(9, 10), ends_at: at(9, 30)),
             described_class::Busy.new(professional_id: "p2", starts_at: at(10), ends_at: at(11)) ]
    expect(hours(shift(starts: at(7), ends: at(13), blocks: three_blocks), medica, busy: busy))
      .to eq([ "06 09:40", "06 10:00", "06 10:20", "06 10:40" ])
  end

  it "turno cancelado: nenhuma vaga" do
    expect(hours(shift(starts: at(8), ends: at(9), cancelled: true), medica)).to eq([])
  end

  it "virada de dia: o modelo vale em cada dia local que o turno cobre" do
    night = shift(starts: at(22), ends: at(2, d: day + 1),
                  blocks: [ block("22:00", "23:00", "bookable", "retorno"), block("00:00", "01:00", "bookable", "retorno") ])
    expect(hours(night, retorno)).to eq([ "06 22:00", "06 22:15", "06 22:30", "06 22:45",
                                          "07 00:00", "07 00:15", "07 00:30", "07 00:45" ])
  end

  it "fuso: as horas do modelo são do fuso da cidade (Manaus, UTC−4)" do
    am = shift(starts: at(8, z: manaus), ends: at(12, z: manaus), blocks: [ block("09:00", "10:00", "bookable", "consulta_medica") ])
    slots = described_class.compute(shifts: [ am ], type: medica, types: types, fallback: fallback, busy: [],
                                    now: at(12, d: day - 1), zone: manaus)
    expect(slots.first.starts_at.utc.strftime("%H:%M")).to eq("13:00")
    expect(slots.size).to eq(3)
  end

  it "janela: só vagas que começam dentro dela; ordem por início e profissional" do
    a = shift(starts: at(8), ends: at(9), pro: "b")
    b = shift(starts: at(8), ends: at(9), pro: "a")
    slots = described_class.compute(shifts: [ a, b ], type: medica, types: types, fallback: fallback, busy: [],
                                    now: at(12, d: day - 1), zone: sp, window: at(8)..at(8, 20))
    expect(slots.map { |s| [ s.professional_id, s.starts_at.in_time_zone(sp).strftime("%H:%M") ] })
      .to eq([ [ "a", "08:00" ], [ "b", "08:00" ], [ "a", "08:20" ], [ "b", "08:20" ] ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/services/scheduling/availability_compute_spec.rb`
Expected: FAIL com `uninitialized constant Scheduling::Availability::Shift`.

- [ ] **Step 3: Implemente a parte pura**

```ruby
# app/services/scheduling/availability.rb
# Vagas calculadas (ADR 0029 §4.1). A vaga nunca é gravada: sai de turnos não
# cancelados, do modelo ligado a cada turno e dos horários ativos do
# profissional. A parte de cima é PURA (recebe dados e o fuso); a carga do
# banco fica em `for` (Task 8).
#
# 1. turno cujo vínculo tem CBO servido pelo tipo pedido;
# 2. faixas: as do modelo (horas no fuso da cidade, em cada dia local que o
#    turno cobre, recortadas pelo turno); sem modelo, o turno inteiro como
#    `bookable` do tipo padrão do vínculo → senão o da base pelo CBO → senão
#    nenhuma faixa (só encaixe);
# 3. cada faixa `bookable` do tipo é cortada na duração da vaga (slot_minutes
#    da faixa ou a do tipo); a sobra do fim é descartada;
# 4. some a vaga que cruza horário ativo do profissional (qualquer modo) e a
#    que já começou.
module Scheduling
  module Availability
    Shift = Data.define(:id, :professional_id, :starts_at, :ends_at, :cbo_code, :default_type_key, :blocks, :cancelled)
    Busy = Data.define(:professional_id, :starts_at, :ends_at)
    Slot = Data.define(:professional_id, :shift_id, :starts_at, :ends_at, :appointment_type_key)
    Block = Data.define(:starts_at, :ends_at, :kind, :appointment_type_key, :slot_minutes)

    module_function

    def compute(shifts:, type:, types:, fallback:, busy:, now:, zone:, window: nil)
      shifts.reject(&:cancelled).select { |shift| type.serves?(shift.cbo_code) }.flat_map do |shift|
        blocks_for(shift, types: types, fallback: fallback, zone: zone)
          .select { |b| b.kind == "bookable" && b.appointment_type_key == type.key }
          .flat_map { |b| slice(b, b.slot_minutes || type.duration_minutes) }
          .map do |starts, ends|
            Slot.new(professional_id: shift.professional_id, shift_id: shift.id, starts_at: starts, ends_at: ends,
                     appointment_type_key: type.key)
          end
          .reject { |slot| slot.starts_at <= now || (window && !window.cover?(slot.starts_at)) }
          .reject { |slot| overlaps_busy?(slot, busy) }
      end.sort_by { |slot| [ slot.starts_at, slot.professional_id.to_s ] }
    end

    def blocks_for(shift, types:, fallback:, zone:)
      return template_blocks(shift, zone) if shift.blocks

      key = default_key(shift, types, fallback)
      return [] unless key

      [ Block.new(starts_at: shift.starts_at, ends_at: shift.ends_at, kind: "bookable", appointment_type_key: key,
                  slot_minutes: nil) ]
    end

    def default_key(shift, types, fallback)
      own = types[shift.default_type_key]
      return own.key if own&.serves?(shift.cbo_code)

      fallback.find { |t| t.serves?(shift.cbo_code) }&.key
    end

    def slice(block, minutes)
      step = minutes.to_i.minutes
      return [] unless step.positive?

      out = []
      cursor = block.starts_at
      while cursor + step <= block.ends_at
        out << [ cursor, cursor + step ]
        cursor += step
      end
      out
    end

    def template_blocks(shift, zone)
      local_days(shift, zone).flat_map do |day|
        Array(shift.blocks).filter_map do |raw|
          starts = [ local(day, raw["starts"], zone), shift.starts_at ].max
          ends = [ local(day, raw["ends"], zone), shift.ends_at ].min
          next unless ends > starts

          Block.new(starts_at: starts, ends_at: ends, kind: raw["kind"], appointment_type_key: raw["appointment_type_key"],
                    slot_minutes: raw["slot_minutes"])
        end
      end.sort_by(&:starts_at)
    end

    def local_days(shift, zone)
      shift.starts_at.in_time_zone(zone).to_date..(shift.ends_at - 1.second).in_time_zone(zone).to_date
    end

    def local(day, hhmm, zone)
      hour, min = hhmm.to_s.split(":").map(&:to_i)
      zone.local(day.year, day.month, day.day, hour, min)
    end

    def overlaps_busy?(slot, busy)
      busy.any? do |b|
        b.professional_id == slot.professional_id && b.starts_at < slot.ends_at && b.ends_at > slot.starts_at
      end
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/services/scheduling/availability_compute_spec.rb`
Expected: PASS (14 exemplos).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/services/scheduling/availability.rb spec/services/scheduling/availability_compute_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: compute bookable slots from shifts and schedule templates

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Modelos de agenda — validação das faixas, gravação e pré-visualização

**Files:**
- Create: `app/services/scheduling/template_blocks.rb`
- Create: `app/services/scheduling/block_json.rb`
- Create: `app/commands/scheduling/save_template.rb`
- Create: `app/services/scheduling/template_preview.rb`
- Create: `app/controllers/schedule_templates_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/services/scheduling/template_blocks_spec.rb`, `spec/commands/scheduling/save_template_spec.rb`, `spec/services/scheduling/template_preview_spec.rb`, `spec/requests/schedule_templates_spec.rb`

**Interfaces:**
- Consumes: `Scheduling::AppointmentTypes.catalog` (Task 3), `Scheduling::Availability.blocks_for/.slice` (Task 5).
- Produces:
  - `Scheduling::TemplateBlocks::KINDS`, `.detail(blocks, catalog) -> Symbol|nil` (`:empty`, `:bad_block`, `:bad_time`, `:crosses_midnight`, `:missing_type`, `:unknown_type`, `:inactive_type`, `:bad_slot_minutes`, `:overlap`), `.normalize(blocks) -> Array<Hash>` (só as chaves do contrato, texto).
  - `Scheduling::BlockJson.list(blocks, zone:, catalog:) -> Array<Hash>` — faixas efetivas (`Availability::Block`) na forma do contrato §2 (`starts`/`ends` `HH:MM` locais, `"24:00"` quando o recorte chega à meia-noite), uma entrada por dia local; `bookable` com `appointment_type_key` e `appointment_type_name` (§9); `slot_minutes` quando veio do modelo.
  - `Scheduling::SaveTemplate.call(template: nil, attrs:, by:) -> Result` (`payload[:template]`; motivos `:invalid_name`, `:invalid_fit_in_limit`, `:invalid_blocks` com `details: { detail: }`, `:invalid`); evento `schedule_template.changed { template_id, user_id }`.
  - `Scheduling::TemplatePreview.call(blocks:, fit_in_limit:, sample:, catalog: AppointmentTypes.catalog, zone: Time.zone) -> Result` (`payload: { slots: [ { starts_at, ends_at, appointment_type_key } ], blocks: [...] }`; motivos de validação + `:invalid`).
  - `ScheduleTemplatesController#template_json(t) -> { id, name, fit_in_limit, blocks, active }`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/scheduling/template_blocks_spec.rb
require "rails_helper"

# Contratos §3, §9: cada recusa com o seu detail.
RSpec.describe Scheduling::TemplateBlocks do
  before { Current.city = TEST_CITY_A; ensure_appointment_types!; type_row!("acupuntura", active: false) }
  after { Current.reset }

  let(:catalog) { Scheduling::AppointmentTypes.catalog }
  def b(starts, ends, kind = "bookable", key = "consulta_medica", slot = nil)
    { "starts" => starts, "ends" => ends, "kind" => kind, "appointment_type_key" => key, "slot_minutes" => slot }.compact
  end

  it "aceita as três faixas sem sobreposição (encostadas valem)" do
    blocks = [ b("07:00", "09:00", "walk_in", nil), b("09:00", "11:00"), b("11:00", "12:00", "blocked", nil) ]
    expect(described_class.detail(blocks, catalog)).to be_nil
  end

  {
    "lista vazia" => [ [], :empty ],
    "não é lista" => [ "x", :bad_block ],
    "faixa que não é objeto" => [ [ "x" ], :bad_block ],
    "chave desconhecida" => [ [ { "starts" => "09:00", "ends" => "10:00", "kind" => "blocked", "cor" => "azul" } ], :bad_block ],
    "kind desconhecido" => [ [ { "starts" => "09:00", "ends" => "10:00", "kind" => "folga" } ], :bad_block ],
    "tipo em faixa que não é bookable" => [ [ { "starts" => "09:00", "ends" => "10:00", "kind" => "walk_in", "appointment_type_key" => "retorno" } ], :bad_block ],
    "hora malformada" => [ [ { "starts" => "9:00", "ends" => "10:00", "kind" => "blocked" } ], :bad_time ],
    "minuto inválido" => [ [ { "starts" => "09:60", "ends" => "10:00", "kind" => "blocked" } ], :bad_time ],
    "fim antes do início" => [ [ { "starts" => "10:00", "ends" => "09:00", "kind" => "blocked" } ], :crosses_midnight ],
    "fim 24:00" => [ [ { "starts" => "22:00", "ends" => "24:00", "kind" => "blocked" } ], :crosses_midnight ],
    "bookable sem tipo" => [ [ { "starts" => "09:00", "ends" => "10:00", "kind" => "bookable" } ], :missing_type ],
    "tipo inexistente" => [ [ { "starts" => "09:00", "ends" => "10:00", "kind" => "bookable", "appointment_type_key" => "fantasma" } ], :unknown_type ],
    "tipo desativado" => [ [ { "starts" => "09:00", "ends" => "10:00", "kind" => "bookable", "appointment_type_key" => "acupuntura" } ], :inactive_type ],
    "slot_minutes fora de 5–240" => [ [ { "starts" => "09:00", "ends" => "10:00", "kind" => "bookable", "appointment_type_key" => "retorno", "slot_minutes" => 4 } ], :bad_slot_minutes ],
    "slot_minutes texto" => [ [ { "starts" => "09:00", "ends" => "10:00", "kind" => "bookable", "appointment_type_key" => "retorno", "slot_minutes" => "15" } ], :bad_slot_minutes ],
    "sobreposição" => [ [ { "starts" => "09:00", "ends" => "10:00", "kind" => "blocked" }, { "starts" => "09:30", "ends" => "11:00", "kind" => "walk_in" } ], :overlap ]
  }.each do |name, (blocks, detail)|
    it("#{name} → #{detail}") { expect(described_class.detail(blocks, catalog)).to eq(detail) }
  end

  it "normalize guarda só as chaves do contrato, em texto" do
    expect(described_class.normalize([ ActiveSupport::HashWithIndifferentAccess.new(b("09:00", "10:00", "bookable", "retorno", 15)) ]))
      .to eq([ { "starts" => "09:00", "ends" => "10:00", "kind" => "bookable", "appointment_type_key" => "retorno", "slot_minutes" => 15 } ])
  end
end
```

```ruby
# spec/commands/scheduling/save_template_spec.rb
require "rails_helper"

RSpec.describe Scheduling::SaveTemplate do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:admin) { staff_with("admin-modelos@cidade.gov.br", "municipal_admin") }
  let(:blocks) { [ { "starts" => "09:00", "ends" => "11:00", "kind" => "bookable", "appointment_type_key" => "consulta_medica" } ] }

  it "cria com limite padrão 2, edita parcialmente e publica só ids" do
    template = described_class.call(attrs: { "name" => " Manhã ", "blocks" => blocks }, by: admin).payload[:template]
    expect(template).to have_attributes(name: "Manhã", fit_in_limit: 2, active: true, blocks: blocks)
    described_class.call(template: template, attrs: { "fit_in_limit" => 0, "active" => false }, by: admin)
    expect(template.reload).to have_attributes(fit_in_limit: 0, active: false, blocks: blocks)
    expect(DomainEvent.where(name: "schedule_template.changed").pluck(:payload).uniq)
      .to eq([ { "template_id" => template.id, "user_id" => admin.id } ])
  end

  it "recusa nome, limite, faixas e ativo inválidos sem gravar" do
    expect(described_class.call(attrs: { "name" => "", "blocks" => blocks }, by: admin).reason).to eq(:invalid_name)
    expect(described_class.call(attrs: { "name" => "M", "fit_in_limit" => 21, "blocks" => blocks }, by: admin).reason)
      .to eq(:invalid_fit_in_limit)
    result = described_class.call(attrs: { "name" => "M", "blocks" => [] }, by: admin)
    expect([ result.reason, result.details ]).to eq([ :invalid_blocks, { detail: "empty" } ])
    expect(described_class.call(attrs: { "name" => "M", "blocks" => blocks, "active" => "sim" }, by: admin).reason).to eq(:invalid)
    expect(ScheduleTemplate.count).to eq(0)
  end
end
```

```ruby
# spec/services/scheduling/template_preview_spec.rb
require "rails_helper"

# Spec §3.2: pré-visualização sem gravar, com faixas efetivas e vagas de cada
# faixa agendável cujo tipo serve o CBO da amostra.
RSpec.describe Scheduling::TemplatePreview do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:day) { Time.zone.today + 3 }
  let(:blocks) do
    [ { "starts" => "07:00", "ends" => "09:00", "kind" => "walk_in" },
      { "starts" => "09:00", "ends" => "10:00", "kind" => "bookable", "appointment_type_key" => "consulta_medica" },
      { "starts" => "10:00", "ends" => "10:30", "kind" => "bookable", "appointment_type_key" => "consulta_enfermagem" },
      { "starts" => "11:00", "ends" => "12:00", "kind" => "blocked" } ]
  end
  let(:sample) { { "starts_at" => day.in_time_zone.change(hour: 7).iso8601, "ends_at" => day.in_time_zone.change(hour: 13).iso8601, "cbo_code" => "225125" } }

  it "vagas só do tipo que o CBO atende; faixas na forma do contrato; nada gravado" do
    result = described_class.call(blocks: blocks, fit_in_limit: 2, sample: sample)
    expect(result.payload[:slots].map { |s| [ Time.zone.parse(s[:starts_at]).strftime("%H:%M"), s[:appointment_type_key] ] })
      .to eq([ [ "09:00", "consulta_medica" ], [ "09:20", "consulta_medica" ], [ "09:40", "consulta_medica" ] ])
    expect(result.payload[:blocks]).to eq([
      { starts: "07:00", ends: "09:00", kind: "walk_in" },
      { starts: "09:00", ends: "10:00", kind: "bookable", appointment_type_key: "consulta_medica", appointment_type_name: "Consulta médica" },
      { starts: "10:00", ends: "10:30", kind: "bookable", appointment_type_key: "consulta_enfermagem", appointment_type_name: "Consulta de enfermagem" },
      { starts: "11:00", ends: "12:00", kind: "blocked" }
    ])
    expect(ScheduleTemplate.count).to eq(0)
  end

  it "recusa faixas, limite e amostra inválidos" do
    expect(described_class.call(blocks: [], fit_in_limit: 2, sample: sample).details).to eq(detail: "empty")
    expect(described_class.call(blocks: blocks, fit_in_limit: -1, sample: sample).reason).to eq(:invalid_fit_in_limit)
    expect(described_class.call(blocks: blocks, fit_in_limit: 2, sample: sample.merge("ends_at" => sample["starts_at"])).reason).to eq(:invalid)
    expect(described_class.call(blocks: blocks, fit_in_limit: 2, sample: sample.merge("cbo_code" => "x")).reason).to eq(:invalid)
  end
end
```

```ruby
# spec/requests/schedule_templates_spec.rb
require "rails_helper"

RSpec.describe "/professionals/schedule_templates", type: :request do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:admin) { staff_with("admin-modelos@cidade.gov.br", "municipal_admin") }
  let(:blocks) { [ { starts: "09:00", ends: "11:00", kind: "bookable", appointment_type_key: "consulta_medica" } ] }
  def body = JSON.parse(response.body)

  it "lista, cria, edita e pré-visualiza (só admin); objeto puro; detail no 422" do
    sign_in_as(staff_with("recepcao-modelos@cidade.gov.br", "citizen_verifier"))
    get "/professionals/schedule_templates"
    expect(response).to have_http_status(:forbidden)

    sign_in_as(admin)
    json_post "/professionals/schedule_templates", name: "Manhã", fit_in_limit: 3, blocks: blocks
    expect(response).to have_http_status(:created)
    id = body["id"]
    expect(body).to eq("id" => id, "name" => "Manhã", "fit_in_limit" => 3, "active" => true,
                       "blocks" => [ { "starts" => "09:00", "ends" => "11:00", "kind" => "bookable", "appointment_type_key" => "consulta_medica" } ])

    json_post "/professionals/schedule_templates/#{id}", blocks: [ { starts: "09:00", ends: "08:00", kind: "blocked" } ]
    expect(response).to have_http_status(:unprocessable_entity)
    expect(body).to eq("error" => "invalid_blocks", "detail" => "crosses_midnight")

    get "/professionals/schedule_templates"
    expect(body["templates"].map { |t| t["id"] }).to eq([ id ])

    day = Time.zone.today + 3
    json_post "/professionals/schedule_templates/preview",
              blocks: blocks, fit_in_limit: 2,
              sample: { starts_at: day.in_time_zone.change(hour: 8).iso8601, ends_at: day.in_time_zone.change(hour: 12).iso8601, cbo_code: "225125" }
    expect(response).to have_http_status(:ok)
    expect(body["slots"].size).to eq(6)
    expect(ScheduleTemplate.count).to eq(1)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/services/scheduling/template_blocks_spec.rb spec/commands/scheduling/save_template_spec.rb spec/services/scheduling/template_preview_spec.rb spec/requests/schedule_templates_spec.rb`
Expected: FAIL com `uninitialized constant Scheduling::TemplateBlocks`.

- [ ] **Step 3: Implemente validação, saída das faixas, gravação e pré-visualização**

```ruby
# app/services/scheduling/template_blocks.rb
# Faixas do modelo (ADR 0029 §3.2; contratos §2, §3, §9): `HH:MM` locais,
# fim depois do início no mesmo dia (sem 24:00), sem sobreposição (encostar
# vale), `bookable` exige tipo ativo, `slot_minutes` 5–240 só em `bookable`.
module Scheduling
  module TemplateBlocks
    KINDS = %w[walk_in bookable blocked].freeze
    KEYS = %w[starts ends kind appointment_type_key slot_minutes].freeze
    TIME = /\A([01]\d|2[0-3]):[0-5]\d\z/
    MAX_BLOCKS = 24

    module_function

    def detail(blocks, catalog)
      return :bad_block unless blocks.is_a?(Array)
      return :empty if blocks.empty?
      return :bad_block if blocks.size > MAX_BLOCKS

      ranges = []
      blocks.each do |raw|
        reason = block_detail(raw, catalog)
        return reason if reason

        ranges << [ raw["starts"], raw["ends"] ]
      end
      ranges.sort.each_cons(2).any? { |(_, ends), (starts, _)| starts < ends } ? :overlap : nil
    end

    def block_detail(raw, catalog)
      return :bad_block unless raw.respond_to?(:to_h) && raw.is_a?(Hash)
      return :bad_block unless (raw.keys.map(&:to_s) - KEYS).empty? && KINDS.include?(raw["kind"])
      return :crosses_midnight if raw["ends"] == "24:00"
      return :bad_time unless TIME.match?(raw["starts"].to_s) && TIME.match?(raw["ends"].to_s)
      return :crosses_midnight unless raw["starts"] < raw["ends"]

      bookable = raw["kind"] == "bookable"
      return :bad_block if !bookable && (raw.key?("appointment_type_key") || raw.key?("slot_minutes"))
      return nil unless bookable
      return :missing_type if raw["appointment_type_key"].blank?

      type = catalog.find(raw["appointment_type_key"])
      return :unknown_type unless type
      return :inactive_type unless type.active

      minutes = raw["slot_minutes"]
      return :bad_slot_minutes unless minutes.nil? || (minutes.is_a?(Integer) && minutes.between?(5, 240))

      nil
    end

    def normalize(blocks) = blocks.map { |raw| raw.to_h.stringify_keys.slice(*KEYS) }
  end
end
```

```ruby
# app/services/scheduling/block_json.rb
# Faixas efetivas (Availability::Block, instantes) na forma do modelo (contratos
# §2): `HH:MM` no fuso da cidade, uma entrada por dia local que a faixa cobre.
# O recorte que chega à meia-noite termina em "24:00" (só na saída).
# `bookable` traz o nome do tipo (contratos §9).
module Scheduling
  module BlockJson
    module_function

    def list(blocks, zone:, catalog:)
      blocks.flat_map do |block|
        first = block.starts_at.in_time_zone(zone).to_date
        last = (block.ends_at - 1.second).in_time_zone(zone).to_date
        (first..last).map do |day|
          day_start = zone.local(day.year, day.month, day.day)
          day_end = day_start + 1.day
          starts = [ block.starts_at, day_start ].max
          ends = [ block.ends_at, day_end ].min
          json(block, starts.in_time_zone(zone).strftime("%H:%M"),
               ends >= day_end ? "24:00" : ends.in_time_zone(zone).strftime("%H:%M"), catalog)
        end
      end
    end

    def json(block, starts, ends, catalog)
      out = { starts: starts, ends: ends, kind: block.kind }
      return out unless block.kind == "bookable"

      out[:appointment_type_key] = block.appointment_type_key
      out[:appointment_type_name] = catalog.name_for(block.appointment_type_key)
      out[:slot_minutes] = block.slot_minutes if block.slot_minutes
      out
    end
  end
end
```

```ruby
# app/commands/scheduling/save_template.rb
# Modelo de agenda (ADR 0029 §3.2; contratos §3, §9). Mudar o modelo nunca
# move horário marcado: as vagas são calculadas, e o horário guarda o que foi
# marcado (o que saiu da grade aparece com outside_template na agenda).
module Scheduling
  module SaveTemplate
    MAX_NAME = 60

    module_function

    def call(attrs:, by:, template: nil)
      template ||= ScheduleTemplate.new
      name = attrs.key?("name") ? attrs["name"] : template.name
      return Result.fail(:invalid_name) unless name.is_a?(String) && name.squish.length.between?(1, MAX_NAME)

      limit = attrs.key?("fit_in_limit") ? attrs["fit_in_limit"] : template.fit_in_limit
      return Result.fail(:invalid_fit_in_limit) unless limit.is_a?(Integer) && limit.between?(0, 20)

      blocks = attrs.key?("blocks") ? attrs["blocks"] : template.blocks
      detail = TemplateBlocks.detail(blocks, AppointmentTypes.catalog)
      return Result.fail(:invalid_blocks, details: { detail: detail.to_s }) if detail

      active = attrs.key?("active") ? attrs["active"] : template.active
      return Result.fail(:invalid) unless [ true, false ].include?(active)

      ApplicationRecord.transaction do
        template.update!(name: name.squish, fit_in_limit: limit, blocks: TemplateBlocks.normalize(blocks), active: active)
        DomainEvents.publish("schedule_template.changed", template_id: template.id, user_id: by.id)
      end
      Result.ok(template: template)
    end
  end
end
```

```ruby
# app/services/scheduling/template_preview.rb
# Pré-visualização do modelo (spec §3.2; contratos §3): aplica as faixas a um
# turno de amostra e devolve as vagas de cada faixa agendável cujo tipo serve
# o CBO da amostra. Nada é gravado.
module Scheduling
  module TemplatePreview
    module_function

    def call(blocks:, fit_in_limit:, sample:, catalog: AppointmentTypes.catalog, zone: Time.zone)
      detail = TemplateBlocks.detail(blocks, catalog)
      return Result.fail(:invalid_blocks, details: { detail: detail.to_s }) if detail
      return Result.fail(:invalid_fit_in_limit) unless fit_in_limit.is_a?(Integer) && fit_in_limit.between?(0, 20)

      shift = sample_shift(sample, TemplateBlocks.normalize(blocks), zone)
      return Result.fail(:invalid) unless shift

      types = catalog.active
      effective = Availability.blocks_for(shift, types: types, fallback: catalog.fallback, zone: zone)
      slots = effective.select { |b| b.kind == "bookable" }.flat_map do |block|
        type = types[block.appointment_type_key]
        next [] unless type&.serves?(shift.cbo_code)

        Availability.slice(block, block.slot_minutes || type.duration_minutes).map do |starts, ends|
          { starts_at: starts.iso8601, ends_at: ends.iso8601, appointment_type_key: type.key }
        end
      end
      Result.ok(slots: slots.sort_by { |s| s[:starts_at] }, blocks: BlockJson.list(effective, zone: zone, catalog: catalog))
    end

    def sample_shift(sample, blocks, zone)
      return nil unless sample.is_a?(Hash) && sample["cbo_code"].to_s.match?(/\A\d{6}\z/)

      starts = zone.iso8601(sample["starts_at"].to_s)
      ends = zone.iso8601(sample["ends_at"].to_s)
      return nil unless ends > starts && ends - starts <= ProfessionalShift::MAX_DURATION

      Availability::Shift.new(id: nil, professional_id: nil, starts_at: starts, ends_at: ends,
                              cbo_code: sample["cbo_code"].to_s, default_type_key: nil, blocks: blocks, cancelled: false)
    rescue ArgumentError
      nil
    end
  end
end
```

- [ ] **Step 4: Controller e rotas**

```ruby
# app/controllers/schedule_templates_controller.rb
# Modelos de agenda (ADR 0029 §3.2; contratos §3, §9). Só municipal_admin; sem
# step-up. Escrita devolve o objeto puro.
class ScheduleTemplatesController < ApplicationController
  include Authentication
  include AttendanceAccess

  wrap_parameters false

  ERROR_STATUS = {
    invalid_name: :unprocessable_entity, invalid_fit_in_limit: :unprocessable_entity,
    invalid_blocks: :unprocessable_entity, invalid: :unprocessable_entity
  }.freeze

  before_action :require_admin

  def index
    render json: { templates: ScheduleTemplate.order(:name).map { |t| template_json(t) } }
  end

  def create
    save(nil, status: :created)
  end

  def update
    template = ScheduleTemplate.find_by(id: params[:id].to_s)
    return render(json: { error: "not_found" }, status: :not_found) unless template

    save(template, status: :ok)
  end

  def preview
    body = request.request_parameters.to_h
    result = Scheduling::TemplatePreview.call(blocks: body["blocks"], fit_in_limit: body["fit_in_limit"],
                                              sample: body["sample"].is_a?(Hash) ? body["sample"].to_h : nil)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: result.payload
  end

  private

  def save(template, status:)
    result = Scheduling::SaveTemplate.call(template: template, attrs: request.request_parameters.to_h, by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: template_json(result.payload[:template]), status: status
  end

  def template_json(t)
    { id: t.id, name: t.name, fit_in_limit: t.fit_in_limit, blocks: t.blocks, active: t.active }
  end
end
```

`request.request_parameters` de um corpo JSON chega como `HashWithIndifferentAccess` (faixas também): `raw["starts"]` funciona e `normalize` grava `Hash` simples. No `TemplateBlocks.block_detail`, `raw.is_a?(Hash)` vale para `HashWithIndifferentAccess` (subclasse de `Hash`).

Em `config/routes.rb`, no `scope "/professionals"`, depois das rotas de tipos e antes de `get ":id"`:

```ruby
    get  "schedule_templates",         to: "schedule_templates#index"
    post "schedule_templates",         to: "schedule_templates#create"
    post "schedule_templates/preview", to: "schedule_templates#preview"
    post "schedule_templates/:id",     to: "schedule_templates#update"
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/services/scheduling/template_blocks_spec.rb spec/commands/scheduling/save_template_spec.rb spec/services/scheduling/template_preview_spec.rb spec/requests/schedule_templates_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/services/scheduling/template_blocks.rb app/services/scheduling/block_json.rb app/commands/scheduling/save_template.rb app/services/scheduling/template_preview.rb app/controllers/schedule_templates_controller.rb config/routes.rb spec/services/scheduling/template_blocks_spec.rb spec/commands/scheduling/save_template_spec.rb spec/services/scheduling/template_preview_spec.rb spec/requests/schedule_templates_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: add schedule templates with block validation and preview

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Modelo no turno e tipo padrão do vínculo

**Files:**
- Modify: `app/commands/professionals/schedule_shift.rb`
- Create: `app/commands/professionals/set_shift_template.rb`, `app/commands/professionals/set_link_default_type.rb`
- Modify: `app/controllers/professional_shifts_controller.rb`, `app/controllers/professional_links_controller.rb`, `app/controllers/concerns/professional_rendering.rb`
- Modify: `config/routes.rb`
- Test: `spec/commands/professionals/shift_template_and_default_type_spec.rb`, `spec/requests/professional_schedule_settings_spec.rb`

**Interfaces:**
- Consumes: `ScheduleTemplate`, `AppointmentType` (Task 2), `Scheduling::AppointmentTypes.serves?` (Task 3).
- Produces:
  - `Professionals::ScheduleShift.call(link:, starts_at:, ends_at:, by:, schedule_template_id: nil)` (motivo novo `:invalid_template`);
  - `Professionals::SetShiftTemplate.call(shift:, schedule_template_id:, by:) -> Result` (`:invalid_template`, `:already_cancelled`); evento `professional.shift_template_set { shift_id, schedule_template_id, by_user_id }`;
  - `Professionals::SetLinkDefaultType.call(link:, appointment_type_key:, by:) -> Result` (`:type_not_served`, `:inactive_type`, `:already_ended`); evento `professional.link_default_type_set { professional_link_id, appointment_type_key, by_user_id }`;
  - `shift_json` ganha `schedule_template_id`; `link_json` ganha `default_appointment_type_key` (contratos §3, §9);
  - rotas `POST /professionals/shifts/:id/template` → `{ shift }`; `POST /professionals/links/:id/default_type` → `{ link }` (os envelopes que essas telas já usam para turno e vínculo).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/commands/professionals/shift_template_and_default_type_spec.rb
require "rails_helper"

# ADR 0029 §3.2: o modelo liga-se ao turno; o vínculo ganha um tipo padrão
# (para o turno sem modelo). Mudar nenhum dos dois move horário marcado.
RSpec.describe "Modelo no turno e tipo padrão do vínculo" do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:admin) { staff_with("admin-agenda@cidade.gov.br", "municipal_admin") }
  let(:unit) { create_unit }
  let(:link) { doctor_link!(unit) }
  let(:template) { ScheduleTemplate.create!(name: "Manhã", blocks: [ { "starts" => "09:00", "ends" => "10:00", "kind" => "blocked" } ]) }
  let(:starts) { 2.days.from_now.change(hour: 8) }

  it "lança turno com modelo ativo; modelo inativo ou inexistente é invalid_template" do
    shift = Professionals::ScheduleShift.call(link: link, starts_at: starts, ends_at: starts + 4.hours, by: admin,
                                              schedule_template_id: template.id).payload[:shift]
    expect(shift.schedule_template_id).to eq(template.id)
    template.update!(active: false)
    expect(Professionals::ScheduleShift.call(link: link, starts_at: starts + 1.day, ends_at: starts + 1.day + 4.hours,
                                             by: admin, schedule_template_id: template.id).reason).to eq(:invalid_template)
    expect(Professionals::ScheduleShift.call(link: link, starts_at: starts + 1.day, ends_at: starts + 1.day + 4.hours,
                                             by: admin, schedule_template_id: SecureRandom.uuid).reason).to eq(:invalid_template)
  end

  it "troca e tira o modelo do turno, sem mexer no horário marcado; turno cancelado recusa" do
    shift = shift!(link, starts_at: starts)
    appt = appointment_row!(triage_request!(Citizen.create!(cpf: "52998224725", phone: "+5541998765432"), unit: unit),
                            shift, starts_at: starts + 1.hour)
    expect(Professionals::SetShiftTemplate.call(shift: shift, schedule_template_id: template.id, by: admin)).to be_ok
    expect(shift.reload.schedule_template_id).to eq(template.id)
    expect(Professionals::SetShiftTemplate.call(shift: shift, schedule_template_id: nil, by: admin)).to be_ok
    expect(shift.reload.schedule_template_id).to be_nil
    expect(appt.reload).to have_attributes(scheduled_at: starts + 1.hour, shift_id: shift.id, status: "confirmed")
    expect(DomainEvent.where(name: "professional.shift_template_set").last.payload)
      .to eq("shift_id" => shift.id, "schedule_template_id" => nil, "by_user_id" => admin.id)

    shift.update!(cancelled_at: Time.current, cancelled_by_user: admin, cancel_reason: "troca de escala")
    expect(Professionals::SetShiftTemplate.call(shift: shift, schedule_template_id: template.id, by: admin).reason)
      .to eq(:already_cancelled)
  end

  it "tipo padrão do vínculo: só tipo ativo que serve o CBO; nil limpa; vínculo encerrado recusa" do
    expect(Professionals::SetLinkDefaultType.call(link: link, appointment_type_key: "retorno", by: admin)).to be_ok
    expect(link.reload.default_appointment_type_key).to eq("retorno")
    expect(Professionals::SetLinkDefaultType.call(link: link, appointment_type_key: "consulta_enfermagem", by: admin).reason)
      .to eq(:type_not_served)
    expect(Professionals::SetLinkDefaultType.call(link: link, appointment_type_key: "fantasma", by: admin).reason)
      .to eq(:type_not_served)
    AppointmentType.find_by!(key: "consulta_medica").update!(active: false)
    expect(Professionals::SetLinkDefaultType.call(link: link, appointment_type_key: "consulta_medica", by: admin).reason)
      .to eq(:inactive_type)
    expect(Professionals::SetLinkDefaultType.call(link: link, appointment_type_key: nil, by: admin)).to be_ok
    expect(link.reload.default_appointment_type_key).to be_nil

    link.update!(ended_at: Time.current, ended_by_user: admin)
    expect(Professionals::SetLinkDefaultType.call(link: link, appointment_type_key: "retorno", by: admin).reason)
      .to eq(:already_ended)
  end
end
```

```ruby
# spec/requests/professional_schedule_settings_spec.rb
require "rails_helper"

RSpec.describe "Modelo no turno e tipo padrão do vínculo (API)", type: :request do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:admin) { staff_with("admin-agenda@cidade.gov.br", "municipal_admin") }
  let(:link) { doctor_link!(create_unit) }
  let(:template) { ScheduleTemplate.create!(name: "Manhã", blocks: [ { "starts" => "09:00", "ends" => "10:00", "kind" => "blocked" } ]) }
  let(:day) { Time.zone.tomorrow.in_time_zone }
  def body = JSON.parse(response.body)

  it "lança com modelo, troca o modelo, define o tipo padrão e a ficha mostra os dois" do
    sign_in_as(admin)
    json_post "/professionals/links/#{link.id}/shifts", starts_at: day.change(hour: 8).iso8601,
                                                        ends_at: day.change(hour: 12).iso8601, schedule_template_id: template.id
    expect(response).to have_http_status(:created)
    expect(body["shift"]["schedule_template_id"]).to eq(template.id)
    shift_id = body.dig("shift", "id")

    json_post "/professionals/shifts/#{shift_id}/template", schedule_template_id: nil
    expect(response).to have_http_status(:ok)
    expect(body["shift"]).to include("id" => shift_id, "schedule_template_id" => nil)

    json_post "/professionals/shifts/#{shift_id}/template", schedule_template_id: SecureRandom.uuid
    expect(response).to have_http_status(:unprocessable_entity)
    expect(body).to eq("error" => "invalid_template")

    json_post "/professionals/links/#{link.id}/default_type", appointment_type_key: "retorno"
    expect(response).to have_http_status(:ok)
    expect(body["link"]).to include("id" => link.id, "default_appointment_type_key" => "retorno")

    get "/professionals/#{link.professional_id}"
    expect(body["links"].sole["default_appointment_type_key"]).to eq("retorno")
    get "/professionals/#{link.professional_id}/shifts"
    expect(body["shifts"].sole["schedule_template_id"]).to be_nil
  end

  it "recepção não mexe (403)" do
    sign_in_as(staff_with("recepcao-agenda@cidade.gov.br", "citizen_verifier"))
    json_post "/professionals/links/#{link.id}/default_type", appointment_type_key: "retorno"
    expect(response).to have_http_status(:forbidden)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/professionals/shift_template_and_default_type_spec.rb spec/requests/professional_schedule_settings_spec.rb`
Expected: FAIL (`unknown keyword: :schedule_template_id`, `uninitialized constant Professionals::SetShiftTemplate`).

- [ ] **Step 3: Implemente**

Em `app/commands/professionals/schedule_shift.rb`, a assinatura vira `def self.call(link:, starts_at:, ends_at:, by:, schedule_template_id: nil)`; depois da checagem de `invalid_shift`:

```ruby
      template = nil
      if schedule_template_id.present?
        template = ScheduleTemplate.find_by(id: schedule_template_id.to_s, active: true)
        return Result.fail(:invalid_template) unless template
      end
```

e o `create!` ganha `schedule_template: template`. (Um `id` que não é uuid vira `nil` no cast do Rails e o `find_by` não acha: `invalid_template`.)

```ruby
# app/commands/professionals/set_shift_template.rb
# Liga, troca ou tira o modelo de agenda do turno (ADR 0029 §3.2). As vagas
# são calculadas: nada do que já foi marcado muda (o horário fora da grade nova
# aparece com outside_template na agenda).
module Professionals
  module SetShiftTemplate
    module_function

    def call(shift:, schedule_template_id:, by:)
      template = nil
      if schedule_template_id.present?
        template = ScheduleTemplate.find_by(id: schedule_template_id.to_s, active: true)
        return Result.fail(:invalid_template) unless template
      end

      ApplicationRecord.transaction do
        shift.lock!
        next Result.fail(:already_cancelled) if shift.cancelled_at

        shift.update!(schedule_template: template)
        DomainEvents.publish("professional.shift_template_set", shift_id: shift.id, schedule_template_id: template&.id,
                                                                by_user_id: by.id)
        Result.ok(shift: shift)
      end
    end
  end
end
```

```ruby
# app/commands/professionals/set_link_default_type.rb
# Tipo padrão do vínculo (ADR 0029 §3.2): vale para o turno sem modelo. Tem de
# ser um tipo ativo que atende o CBO do vínculo.
module Professionals
  module SetLinkDefaultType
    module_function

    def call(link:, appointment_type_key:, by:)
      key = appointment_type_key.presence
      if key
        type = AppointmentType.find_by(key: key.to_s)
        return Result.fail(:type_not_served) unless type && Scheduling::AppointmentTypes.serves?(type, link.cbo_code)
        return Result.fail(:inactive_type) unless type.active
      end

      ApplicationRecord.transaction do
        link.lock!
        next Result.fail(:already_ended) unless link.active?

        link.update!(default_appointment_type_key: key)
        DomainEvents.publish("professional.link_default_type_set", professional_link_id: link.id,
                                                                   appointment_type_key: key, by_user_id: by.id)
        Result.ok(link: link)
      end
    end
  end
end
```

Em `app/controllers/professional_shifts_controller.rb`: `ERROR_STATUS` ganha `invalid_template: :unprocessable_entity`; `create` lê `scalar_body(%w[starts_at ends_at schedule_template_id])` e passa `schedule_template_id: body["schedule_template_id"]`; ação nova:

```ruby
  def template
    shift = ProfessionalShift.find_by(id: params[:id])
    return render(json: { error: "not_found" }, status: :not_found) unless shift

    body = scalar_body(%w[schedule_template_id])
    return render(json: { error: "invalid" }, status: :unprocessable_entity) if body.value?(:non_scalar)

    result = Professionals::SetShiftTemplate.call(shift: shift, schedule_template_id: body["schedule_template_id"],
                                                  by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { shift: shift_json(result.payload[:shift]) }
  end
```

Em `app/controllers/professional_links_controller.rb`: `ERROR_STATUS` ganha `type_not_served: :unprocessable_entity, inactive_type: :unprocessable_entity` (`already_ended` já está); ação nova, **sem** step-up (o tipo padrão não muda autorização clínica):

```ruby
  def default_type
    link = ProfessionalLink.find_by(id: params[:id])
    return render(json: { error: "not_found" }, status: :not_found) unless link

    body = scalar_body(%w[appointment_type_key])
    return render(json: { error: "invalid" }, status: :unprocessable_entity) if body.value?(:non_scalar)

    result = Professionals::SetLinkDefaultType.call(link: link, appointment_type_key: body["appointment_type_key"],
                                                    by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { link: link_json(result.payload[:link]) }
  end
```

Em `app/controllers/concerns/professional_rendering.rb`: `link_json` ganha `default_appointment_type_key: l.default_appointment_type_key`; `shift_json` ganha `schedule_template_id: s.schedule_template_id`.

Em `config/routes.rb`, no `scope "/professionals"`, junto de `links/:id/end` e `shifts/:id/cancel`:

```ruby
    post "links/:id/default_type", to: "professional_links#default_type"
    post "shifts/:id/template",    to: "professional_shifts#template"
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/professionals spec/requests/professional_schedule_settings_spec.rb spec/requests/professional_shifts_spec.rb spec/requests/professional_links_spec.rb spec/requests/professionals_spec.rb`
Expected: PASS. Se algum exemplo existente compara `link_json`/`shift_json` com `eq` de hash inteiro, acrescente a chave nova (`default_appointment_type_key: nil` / `schedule_template_id: nil`) à expectativa — é a única mudança permitida nessas specs.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/commands/professionals/schedule_shift.rb app/commands/professionals/set_shift_template.rb app/commands/professionals/set_link_default_type.rb app/controllers/professional_shifts_controller.rb app/controllers/professional_links_controller.rb app/controllers/concerns/professional_rendering.rb config/routes.rb spec/commands/professionals/shift_template_and_default_type_spec.rb spec/requests/professional_schedule_settings_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: attach schedule templates to shifts and default types to links

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Se alguma spec existente precisou da chave nova, acrescente o caminho dela ao `git add`.)

---

## Fatia 4 — Marcação (F-17.3, F-17.4, F-17.9)

### Task 8: Vagas da unidade (`Availability.for`), transição `legacy` e `GET /attendance/units/:id/availability`

**Files:**
- Modify: `app/services/scheduling/availability.rb` (carga)
- Create: `app/services/scheduling/transition.rb`
- Modify: `app/controllers/appointment_requests_controller.rb`, `config/routes.rb`
- Create: `app/services/scheduling/date_range.rb`
- Test: `spec/services/scheduling/availability_for_spec.rb`, `spec/requests/unit_availability_spec.rb`

**Interfaces:**
- Consumes: `Availability.compute` (Task 5), `AppointmentTypes.catalog` (Task 3).
- Produces:
  - `Scheduling::Availability.for(unit:, from:, to:, appointment_type:, now: Time.current, catalog: AppointmentTypes.catalog) -> Array<Slot>` (`from`/`to` `Date` inclusivos no fuso da cidade; tipo inexistente ou desativado → `[]`);
  - `Scheduling::Availability.shift_data(professional_shift) -> Shift`, `.shifts_in(unit_id, window, include_cancelled: false) -> Array<Shift>`, `.busy_for(professional_ids, window) -> Array<Busy>`;
  - `Scheduling::Transition.shift_days(unit_id, from, to) -> Set<Date>`, `.legacy_days(unit_id, from, to) -> Array<Date>`, `.slots_day?(unit_id, date) -> Boolean`;
  - `Scheduling::DateRange.parse(from, to, default_days:, max_days:) -> Range<Date>|nil` (inclusivo; `nil` = inválido);
  - `GET /attendance/units/:id/availability?type=&from=&to=` (até 14 dias) → `{ slots: [ { professional_id, professional_name, shift_id, starts_at, ends_at } ], legacy_days: [ "YYYY-MM-DD" ] }`; 422 `invalid_range`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/scheduling/availability_for_spec.rb
require "rails_helper"

# A carga do banco que alimenta o cálculo puro (ADR 0029 §4.1): só turnos não
# cancelados da unidade; horários ativos do profissional em QUALQUER unidade
# ocupam; tipo desativado não tem vaga; dia sem turno é legacy.
RSpec.describe Scheduling::Availability, ".for" do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:other_unit) { create_unit("UBS Sul") }
  let(:link) { doctor_link!(unit) }
  let(:day) { Time.zone.today + 2 }
  let(:medica) { AppointmentType.find_by!(key: "consulta_medica") }
  def hours(slots) = slots.map { |s| s.starts_at.strftime("%H:%M") }

  it "vagas do turno, sem as ocupadas (inclusive em outra unidade) e sem turno cancelado" do
    shift = shift!(link, starts_at: day.in_time_zone.change(hour: 8), ends_at: day.in_time_zone.change(hour: 9))
    shift!(link, starts_at: (day + 1).in_time_zone.change(hour: 8), ends_at: (day + 1).in_time_zone.change(hour: 9))
      .update!(cancelled_at: Time.current, cancelled_by_user: link.started_by_user, cancel_reason: "troca")
    appointment_row!(triage_request!(Citizen.create!(cpf: "52998224725", phone: "+5541998765432"), unit: unit), shift,
                     starts_at: shift.starts_at + 20.minutes)

    slots = described_class.for(unit: unit, from: day, to: day + 1, appointment_type: medica)
    expect(hours(slots)).to eq([ "08:00", "08:40" ])
    expect(slots.first).to have_attributes(professional_id: link.professional_id, shift_id: shift.id)

    other_link = ProfessionalLink.create!(professional: link.professional, health_unit: other_unit, cbo_code: "225125",
                                          started_at: Time.current, started_by_user: link.started_by_user)
    other_shift = shift!(other_link, starts_at: day.in_time_zone.change(hour: 9), ends_at: day.in_time_zone.change(hour: 10))
    expect { appointment_row!(triage_request!(Citizen.create!(cpf: "11144477735", phone: "+5541911112222"), unit: other_unit),
                              other_shift, starts_at: other_shift.starts_at) }.not_to raise_error
    expect(hours(described_class.for(unit: other_unit, from: day, to: day, appointment_type: medica)))
      .to eq([ "09:20", "09:40" ])
  end

  it "tipo desativado: nenhuma vaga" do
    shift!(link, starts_at: day.in_time_zone.change(hour: 8))
    medica.update!(active: false)
    expect(described_class.for(unit: unit, from: day, to: day, appointment_type: medica)).to eq([])
  end

  it "transição: dia com turno não cancelado da unidade usa vagas; os outros são legacy" do
    shift!(link, starts_at: day.in_time_zone.change(hour: 22), ends_at: (day + 1).in_time_zone.change(hour: 6))
    cancelled = shift!(link, starts_at: (day + 3).in_time_zone.change(hour: 8))
    cancelled.update!(cancelled_at: Time.current, cancelled_by_user: link.started_by_user, cancel_reason: "troca")
    expect(Scheduling::Transition.legacy_days(unit.id, day - 1, day + 3)).to eq([ day - 1, day + 2, day + 3 ])
    expect(Scheduling::Transition.slots_day?(unit.id, day + 1)).to be(true)
    expect(Scheduling::Transition.slots_day?(other_unit.id, day)).to be(false)
  end
end
```

```ruby
# spec/requests/unit_availability_spec.rb
require "rails_helper"

RSpec.describe "GET /attendance/units/:id/availability", type: :request do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:link) { doctor_link!(unit) }
  let(:day) { Time.zone.today + 2 }
  def body = JSON.parse(response.body)

  it "vagas com o nome do profissional e os dias legacy, para a recepção" do
    shift = shift!(link, starts_at: day.in_time_zone.change(hour: 8), ends_at: day.in_time_zone.change(hour: 8, min: 40))
    sign_in_as(staff_with("recepcao-vagas@cidade.gov.br", "citizen_verifier"))
    get "/attendance/units/#{unit.id}/availability", params: { type: "consulta_medica", from: day.iso8601, to: (day + 1).iso8601 }
    expect(response).to have_http_status(:ok)
    expect(body["slots"]).to eq([
      { "professional_id" => link.professional_id, "professional_name" => link.professional.professional_name,
        "shift_id" => shift.id, "starts_at" => shift.starts_at.iso8601, "ends_at" => (shift.starts_at + 20.minutes).iso8601 },
      { "professional_id" => link.professional_id, "professional_name" => link.professional.professional_name,
        "shift_id" => shift.id, "starts_at" => (shift.starts_at + 20.minutes).iso8601, "ends_at" => shift.ends_at.iso8601 }
    ])
    expect(body["legacy_days"]).to eq([ (day + 1).iso8601 ])

    get "/attendance/units/#{unit.id}/availability", params: { type: "fantasma", from: day.iso8601, to: day.iso8601 }
    expect(body).to eq("slots" => [], "legacy_days" => [])
  end

  it "intervalo inválido ou maior que 14 dias: 422 invalid_range; profissional sem papel de recepção: 403" do
    sign_in_as(staff_with("recepcao-vagas@cidade.gov.br", "citizen_verifier"))
    get "/attendance/units/#{unit.id}/availability", params: { type: "consulta_medica", from: day.iso8601, to: (day + 14).iso8601 }
    expect(response).to have_http_status(:unprocessable_entity)
    expect(body).to eq("error" => "invalid_range")
    get "/attendance/units/#{unit.id}/availability", params: { type: "consulta_medica", from: "ontem" }
    expect(body).to eq("error" => "invalid_range")

    sign_in_as(staff_with("medica-vagas@cidade.gov.br", "health_professional"))
    get "/attendance/units/#{unit.id}/availability", params: { type: "consulta_medica" }
    expect(response).to have_http_status(:forbidden)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/services/scheduling/availability_for_spec.rb spec/requests/unit_availability_spec.rb`
Expected: FAIL (`undefined method 'for'`, rota inexistente).

- [ ] **Step 3: Implemente a carga, a transição e o intervalo**

No fim de `app/services/scheduling/availability.rb`, ainda dentro de `module Availability` (abaixo de `overlaps_busy?`), a parte que lê o banco:

```ruby
    # ── Carga do banco (não é pura) ─────────────────────────────────────────
    def for(unit:, from:, to:, appointment_type:, now: Time.current, catalog: AppointmentTypes.catalog)
      types = catalog.active
      type = appointment_type && types[appointment_type.key]
      return [] unless type

      window = from.in_time_zone.beginning_of_day..to.in_time_zone.end_of_day
      shifts = shifts_in(unit.id, window)
      busy = busy_for(shifts.map(&:professional_id).uniq, window)
      compute(shifts: shifts, type: type, types: types, fallback: catalog.fallback, busy: busy, now: now,
              zone: Time.zone, window: window)
    end

    def shifts_in(unit_id, window, include_cancelled: false)
      scope = ProfessionalShift.joins(:professional_link).where(professional_links: { health_unit_id: unit_id })
                               .where("professional_shifts.starts_at <= ? AND professional_shifts.ends_at > ?",
                                      window.end, window.begin)
                               .includes(:professional_link, :schedule_template)
      scope = scope.where(cancelled_at: nil) unless include_cancelled
      scope.order(:starts_at).map { |s| shift_data(s) }
    end

    def shift_data(shift)
      Shift.new(id: shift.id, professional_id: shift.professional_id, starts_at: shift.starts_at, ends_at: shift.ends_at,
                cbo_code: shift.professional_link.cbo_code,
                default_type_key: shift.professional_link.default_appointment_type_key,
                blocks: shift.schedule_template&.blocks, cancelled: shift.cancelled_at.present?)
    end

    # Horários ativos do profissional em qualquer unidade (com folga de um dia
    # nas bordas: vaga do fim da janela pode passar da meia-noite).
    def busy_for(professional_ids, window)
      return [] if professional_ids.empty?

      Appointment.where(status: Appointment::ACTIVE, professional_id: professional_ids).where.not(ends_at: nil)
                 .where("scheduled_at < ? AND ends_at > ?", window.end + 1.day, window.begin - 1.day)
                 .pluck(:professional_id, :scheduled_at, :ends_at)
                 .map { |pro, starts, ends| Busy.new(professional_id: pro, starts_at: starts, ends_at: ends) }
    end
```

```ruby
# app/services/scheduling/transition.rb
# Transição por unidade (ADR 0029): a marcação livre de hoje (legacy) vale só
# no dia em que a unidade não tem nenhum turno não cancelado cruzando o dia
# (fuso da cidade). Turno que termina exatamente à meia-noite não conta no dia
# seguinte.
module Scheduling
  module Transition
    module_function

    def shift_days(unit_id, from, to)
      window = from.in_time_zone.beginning_of_day..to.in_time_zone.end_of_day
      ProfessionalShift.joins(:professional_link)
                       .where(professional_links: { health_unit_id: unit_id }, cancelled_at: nil)
                       .where("professional_shifts.starts_at <= ? AND professional_shifts.ends_at > ?", window.end, window.begin)
                       .pluck(:starts_at, :ends_at)
                       .flat_map { |starts, ends| (starts.in_time_zone.to_date..(ends - 1.second).in_time_zone.to_date).to_a }
                       .to_set
    end

    def legacy_days(unit_id, from, to)
      with_shift = shift_days(unit_id, from, to)
      (from..to).reject { |day| with_shift.include?(day) }
    end

    def slots_day?(unit_id, date) = shift_days(unit_id, date, date).include?(date)
  end
end
```

```ruby
# app/services/scheduling/date_range.rb
# Intervalo de datas da cidade, inclusivo (contratos §9). Sem `from`, hoje; sem
# `to`, from + (default_days − 1). nil quando inválido ou longo demais.
module Scheduling
  module DateRange
    module_function

    def parse(from, to, default_days:, max_days:)
      first = from.present? ? Date.iso8601(from.to_s) : Time.zone.today
      last = to.present? ? Date.iso8601(to.to_s) : first + (default_days - 1)
      return nil if last < first || (last - first).to_i >= max_days

      first..last
    rescue ArgumentError, Date::Error
      nil
    end
  end
end
```

- [ ] **Step 4: A rota e a ação**

Em `app/controllers/appointment_requests_controller.rb`, ação nova (o `before_action :require_verifier` já cobre):

```ruby
  AVAILABILITY_MAX_DAYS = 14

  def availability
    unit = HealthUnit.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless unit

    range = Scheduling::DateRange.parse(params[:from], params[:to], default_days: 7, max_days: AVAILABILITY_MAX_DAYS)
    return render json: { error: "invalid_range" }, status: :unprocessable_entity unless range

    type = AppointmentType.find_by(key: params[:type].to_s)
    slots = type ? Scheduling::Availability.for(unit: unit, from: range.begin, to: range.end, appointment_type: type) : []
    names = Professional.where(id: slots.map(&:professional_id).uniq).pluck(:id, :professional_name).to_h
    render json: {
      slots: slots.map do |s|
        { professional_id: s.professional_id, professional_name: names[s.professional_id], shift_id: s.shift_id,
          starts_at: s.starts_at.iso8601, ends_at: s.ends_at.iso8601 }
      end,
      legacy_days: type ? Scheduling::Transition.legacy_days(unit.id, range.begin, range.end).map(&:iso8601) : []
    }
  end
```

Em `config/routes.rb`, no `scope "/attendance"`, junto das rotas de pedidos: `get "units/:id/availability", to: "appointment_requests#availability"`.

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/services/scheduling spec/requests/unit_availability_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/services/scheduling/availability.rb app/services/scheduling/transition.rb app/services/scheduling/date_range.rb app/controllers/appointment_requests_controller.rb config/routes.rb spec/services/scheduling/availability_for_spec.rb spec/requests/unit_availability_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: list a unit's bookable slots and its legacy days

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: `Appointments::Book` — marcação em vaga (e remarcação pela recepção)

**Files:**
- Create: `app/commands/appointments/placement.rb`
- Create: `app/commands/appointments/book.rb`
- Test: `spec/commands/appointments/book_spec.rb`

**Interfaces:**
- Consumes: `Scheduling::Availability.for` (Task 8), `Scheduling::AppointmentTypes.serves?` (Task 3), `HealthUnit.lock_active!`.
- Produces:
  - `Appointments::Placement.parse(value) -> Time|nil`, `.lock!(request)` (cidadão FOR UPDATE → horários vivos do pedido FOR UPDATE → pedido FOR UPDATE: a ordem horário → pedido de `CancelByCitizen`/`Lapse`/`Drain`, com o cidadão antes, como `Citizens::Erase`), `.bookable?(request) -> Boolean` (`open` ou `scheduled`), `.citizen_busy?(request, starts, ends) -> Boolean` (ignora o horário vivo do próprio pedido), `.create!(request:, at:, by:, now:, **attrs) -> Appointment` (encerra o vivo como `moved`, liga por `moved_from_appointment`, nasce confirmado com menos de 48h, pedido vira `scheduled` e perde `reopened_reason`; publica `appointment.moved` quando moveu).
  - `Appointments::Book.call(request:, professional:, starts_at:, type:, by:, now: Time.current) -> Result` — motivos `:invalid_time`, `:wrong_unit` (pedido sem unidade), `:slot_unavailable`, `:type_not_served`, `:request_not_open`, `:citizen_busy`, `:slot_taken` (EXCLUDE), `:invalid_unit`; `payload[:appointment]`; evento `appointment.booked { appointment_id, request_id, booking_kind: "slot" }`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/commands/appointments/book_spec.rb
require "rails_helper"

# ADR 0029 §4.3: a vaga tem de existir no cálculo; o cidadão não pode ter
# outro horário ativo sobreposto; a EXCLUDE decide a corrida (slot_taken).
RSpec.describe Appointments::Book do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:link) { doctor_link!(unit) }
  let(:reception) { staff_with("recepcao-book@cidade.gov.br", "citizen_verifier") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:medica) { AppointmentType.find_by!(key: "consulta_medica") }
  let(:day) { Time.zone.today + 5 }
  let!(:shift) { shift!(link, starts_at: day.in_time_zone.change(hour: 8), ends_at: day.in_time_zone.change(hour: 10)) }
  let(:request) { triage_request!(citizen, unit: unit) }

  def book(req = request, at: shift.starts_at, type: medica, professional: link.professional)
    described_class.call(request: req, professional: professional, starts_at: at.iso8601, type: type, by: reception)
  end

  it "marca a vaga: slot com profissional, tipo, fim e turno; pedido scheduled; evento só de ids" do
    appointment = book.payload[:appointment]
    expect(appointment).to have_attributes(booking_kind: "slot", professional_id: link.professional_id, shift_id: shift.id,
                                           appointment_type_key: "consulta_medica", scheduled_at: shift.starts_at,
                                           ends_at: shift.starts_at + 20.minutes, status: "scheduled",
                                           health_unit_id: unit.id)
    expect(request.reload.status).to eq("scheduled")
    expect(DomainEvent.where(name: "appointment.booked").sole.payload)
      .to eq("appointment_id" => appointment.id, "request_id" => request.id, "booking_kind" => "slot")
  end

  it "com menos de 48h nasce confirmado" do
    near = shift!(link, starts_at: 1.day.from_now.change(min: 0), ends_at: 1.day.from_now.change(min: 0) + 1.hour)
    expect(book(at: near.starts_at).payload[:appointment].status).to eq("confirmed")
  end

  it "fora da grade é slot_unavailable; tipo que o CBO não atende é type_not_served; passado é invalid_time" do
    expect(book(at: shift.starts_at + 5.minutes).reason).to eq(:slot_unavailable)
    expect(book(type: AppointmentType.find_by!(key: "consulta_enfermagem")).reason).to eq(:type_not_served)
    expect(book(at: 1.hour.ago).reason).to eq(:invalid_time)
    expect(described_class.call(request: request, professional: nil, starts_at: shift.starts_at.iso8601, type: medica,
                                by: reception).reason).to eq(:slot_unavailable)
  end

  it "pedido sem unidade é wrong_unit; pedido encerrado é request_not_open" do
    expect(book(triage_request!(Citizen.create!(cpf: "11144477735", phone: "+5541911112222"), unit: nil)).reason).to eq(:wrong_unit)
    AppointmentRequests::Lifecycle.close!(request, reason: "dismissed", by: reception, dismiss_reason: "cidadão mudou de cidade")
    expect(book.reason).to eq(:request_not_open)
  end

  it "cidadão com outro horário ativo sobreposto (inclusive legacy de 15 min) é citizen_busy" do
    other = triage_request!(citizen, unit: unit, type_key: "retorno")
    Appointment.create!(request: other, citizen: citizen, health_unit: unit, scheduled_at: shift.starts_at + 10.minutes,
                        scheduled_by_user: reception, status: "confirmed", confirmed_at: Time.current)
    expect(book.reason).to eq(:citizen_busy)
    expect(book(at: shift.starts_at + 40.minutes)).to be_ok
  end

  it "violação da trava (corrida) vira slot_taken, sem exceção" do
    rival = triage_request!(Citizen.create!(cpf: "11144477735", phone: "+5541911112222"), unit: unit)
    slot = Scheduling::Availability.for(unit: unit, from: day, to: day, appointment_type: medica).first
    appointment_row!(rival, shift, starts_at: shift.starts_at)
    allow(Scheduling::Availability).to receive(:for).and_return([ slot ])
    expect(book.reason).to eq(:slot_taken)
    expect(request.reload.status).to eq("open")
  end

  it "remarcação pela recepção: o horário vivo vira moved e o novo aponta para ele" do
    first = book.payload[:appointment]
    second = book(at: shift.starts_at + 40.minutes).payload[:appointment]
    expect(first.reload).to have_attributes(status: "moved")
    expect(second).to have_attributes(moved_from_appointment_id: first.id, status: "scheduled")
    expect(DomainEvent.where(name: "appointment.moved").sole.payload)
      .to include("from_appointment_id" => first.id, "to_appointment_id" => second.id)
  end

  it "unidade desativada entre a leitura e a transação: invalid_unit" do
    request # o pedido nasce antes do stub de transação
    deactivate_before_transaction(unit)
    expect(book.reason).to eq(:invalid_unit)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/appointments/book_spec.rb`
Expected: FAIL com `uninitialized constant Appointments::Book`.

- [ ] **Step 3: Implemente**

```ruby
# app/commands/appointments/placement.rb
# O que Book e FitIn compartilham (ADR 0029 §4.3). Chamado DENTRO da transação.
#
# Ordem de travas: cidadão (FOR UPDATE: duas marcações do mesmo cidadão se
# enfileiram, e citizen_busy vê a anterior) → horários vivos do pedido → pedido.
# Horário antes de pedido é a ordem de CancelByCitizen, Lapse e Drain; cidadão
# antes de tudo é a de Citizens::Erase. Depois disso vêm a unidade (FOR SHARE)
# e, no encaixe, o turno (FOR UPDATE).
module Appointments
  module Placement
    module_function

    def parse(value)
      Time.zone.iso8601(value.to_s)
    rescue ArgumentError
      nil
    end

    def lock!(request)
      Citizen.lock.find(request.citizen_id)
      Appointment.where(request_id: request.id, status: Appointment::LIVE).order(:id).lock.to_a
      request.lock!
    end

    def bookable?(request) = %w[open scheduled].include?(request.status)

    def citizen_busy?(request, starts, ends)
      own = Appointment.where(request_id: request.id, status: Appointment::LIVE).select(:id)
      Appointment.where(citizen_id: request.citizen_id, status: Appointment::ACTIVE).where.not(id: own)
                 .where("scheduled_at < ? AND COALESCE(ends_at, scheduled_at + make_interval(secs => ?)) > ?",
                        ends, Appointment::LEGACY_SPAN.to_i, starts)
                 .exists?
    end

    # Remarcar = encerrar o vivo como `moved` e criar o novo ligado a ele (o
    # mesmo mecanismo do esvaziamento de unidade, api#29).
    def create!(request:, at:, by:, now:, **attrs)
      previous = request.appointments.live.first
      previous&.update!(status: "moved", ended_at: now)
      born_confirmed = at - now < Appointment::BORN_CONFIRMED_WITHIN
      appointment = Appointment.create!(
        request: request, citizen_id: request.citizen_id, health_unit_id: request.target_unit_id, scheduled_at: at,
        scheduled_by_user: by, moved_from_appointment: previous, status: born_confirmed ? "confirmed" : "scheduled",
        confirmed_at: born_confirmed ? now : nil,
        confirmation_deadline_at: born_confirmed ? nil : at - Appointment::CONFIRMATION_LEAD, **attrs
      )
      request.update!(status: "scheduled", reopened_reason: nil)
      if previous
        DomainEvents.publish("appointment.moved", from_appointment_id: previous.id, to_appointment_id: appointment.id,
                                                  to_unit_id: appointment.health_unit_id, born_confirmed: born_confirmed,
                                                  fit_in: appointment.fit_in?)
      end
      appointment
    end
  end
end
```

```ruby
# app/commands/appointments/book.rb
# Marcação em vaga (ADR 0029 §4.3): a vaga tem de existir no cálculo
# (Scheduling::Availability) para o profissional, o início e o tipo; o cidadão
# não pode ter outro horário ativo sobreposto; a EXCLUDE do banco decide a
# corrida entre duas recepções (slot_taken). Em pedido já marcado, remarca.
module Appointments
  module Book
    module_function

    def call(request:, professional:, starts_at:, type:, by:, now: Time.current)
      at = Placement.parse(starts_at)
      return Result.fail(:invalid_time) if at.nil? || at <= now || at > now + Appointment::MAX_AHEAD
      return Result.fail(:slot_unavailable) if professional.nil?
      return Result.fail(:type_not_served) if type.nil?
      return Result.fail(:wrong_unit) if request.target_unit_id.nil?

      ApplicationRecord.transaction do
        Placement.lock!(request)
        next Result.fail(:request_not_open) unless Placement.bookable?(request)

        HealthUnit.lock_active!(request.target_unit_id)
        slot = find_slot(request, professional, type, at, now)
        next Result.fail(served?(request, professional, type) ? :slot_unavailable : :type_not_served) unless slot
        next Result.fail(:citizen_busy) if Placement.citizen_busy?(request, at, slot.ends_at)

        appointment = Placement.create!(request: request, at: at, by: by, now: now, booking_kind: "slot",
                                        professional_id: professional.id, appointment_type_key: type.key,
                                        ends_at: slot.ends_at, shift_id: slot.shift_id)
        DomainEvents.publish("appointment.booked", appointment_id: appointment.id, request_id: request.id,
                                                   booking_kind: "slot")
        Result.ok(appointment: appointment)
      end
    rescue ActiveRecord::ExclusionViolation
      Result.fail(:slot_taken)
    rescue HealthUnit::Inactive
      Result.fail(:invalid_unit)
    end

    def find_slot(request, professional, type, at, now)
      day = at.to_date
      Scheduling::Availability.for(unit: request.target_unit, from: day, to: day, appointment_type: type, now: now)
                              .find { |s| s.professional_id == professional.id && s.starts_at == at }
    end

    def served?(request, professional, type)
      type.active && ProfessionalLink.active.where(professional_id: professional.id, health_unit_id: request.target_unit_id)
                                     .any? { |link| Scheduling::AppointmentTypes.serves?(type, link.cbo_code) }
    end
  end
end
```

(A vaga é procurada no dia local do início; um turno noturno que começou na véspera entra porque `shifts_in` pega o turno que cruza a janela do dia.)

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/appointments/book_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/commands/appointments/placement.rb app/commands/appointments/book.rb spec/commands/appointments/book_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: book appointments into computed slots

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: `Appointments::FitIn` — encaixe com justificativa e limite

**Files:**
- Create: `app/services/scheduling/fit_in_limit.rb`
- Create: `app/commands/appointments/fit_in.rb`
- Test: `spec/commands/appointments/fit_in_spec.rb`

**Interfaces:**
- Consumes: `Appointments::Placement` (Task 9).
- Produces:
  - `Scheduling::FitInLimit.for(shift) -> Integer` (modelo do turno; senão `city_profile.default_fit_in_limit`; senão 2), `.count(shift, except_request_id: nil) -> Integer` (encaixes ativos do turno);
  - `Appointments::FitIn.call(request:, professional:, shift:, starts_at:, type:, reason:, by:, now: Time.current) -> Result` — motivos `:invalid_reason`, `:invalid_time`, `:outside_shift`, `:type_not_served`, `:wrong_unit`, `:request_not_open`, `:fit_in_limit`, `:citizen_busy`, `:invalid_unit`; eventos `appointment.booked { appointment_id, request_id, booking_kind: "fit_in" }` e `appointment.fit_in_created { appointment_id, shift_id }` (sem a justificativa).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/commands/appointments/fit_in_spec.rb
require "rails_helper"

# ADR 0029 §4.3: encaixe é horário extra DENTRO do turno, com justificativa,
# contado contra o limite do turno; pode sobrepor vaga ocupada.
RSpec.describe Appointments::FitIn do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:link) { doctor_link!(unit) }
  let(:reception) { staff_with("recepcao-encaixe@cidade.gov.br", "citizen_verifier") }
  let(:medica) { AppointmentType.find_by!(key: "consulta_medica") }
  let(:day) { Time.zone.today + 5 }
  let(:shift) { shift!(link, starts_at: day.in_time_zone.change(hour: 8), ends_at: day.in_time_zone.change(hour: 10)) }
  let(:reason) { "gestante com sangramento leve" }
  let(:cpfs) { %w[52998224725 11144477735 39053344705 87748248800] }
  def citizen(i) = Citizen.create!(cpf: cpfs[i], phone: "+55419#{format('%08d', 30_000_000 + i)}")
  let(:requests) { Hash.new { |h, k| h[k] = triage_request!(citizen(k), unit: unit) } }

  def fit_in(req, at: shift.starts_at + 10.minutes, why: reason, type: medica, on: shift)
    described_class.call(request: req, professional: link.professional, shift: on, starts_at: at.iso8601, type: type,
                         reason: why, by: reception)
  end

  it "grava fit_in sobre vaga ocupada, com fim pela duração do tipo; eventos sem a justificativa" do
    appointment_row!(requests[1], shift, starts_at: shift.starts_at)
    appointment = fit_in(requests[0]).payload[:appointment]
    expect(appointment).to have_attributes(booking_kind: "fit_in", fit_in_reason: reason, shift_id: shift.id,
                                           ends_at: shift.starts_at + 30.minutes)
    events = DomainEvent.where(name: %w[appointment.booked appointment.fit_in_created]).pluck(:name, :payload).to_h
    expect(events["appointment.fit_in_created"]).to eq("appointment_id" => appointment.id, "shift_id" => shift.id)
    expect(events["appointment.booked"]).to include("booking_kind" => "fit_in")
    expect(events.values.to_json).not_to include("gestante")
  end

  it "conta contra o limite padrão da cidade (2) ou o do modelo; o próprio pedido remarcado não conta" do
    first = fit_in(requests[0]).payload[:appointment]
    fit_in(requests[1], at: shift.starts_at + 40.minutes)
    expect(fit_in(requests[2], at: shift.starts_at + 70.minutes).reason).to eq(:fit_in_limit)
    expect(fit_in(requests[0], at: shift.starts_at + 70.minutes)).to be_ok
    expect(first.reload.status).to eq("moved")

    template = ScheduleTemplate.create!(name: "Sem encaixe", fit_in_limit: 0,
                                        blocks: [ { "starts" => "08:00", "ends" => "10:00", "kind" => "blocked" } ])
    other = shift!(link, starts_at: (day + 1).in_time_zone.change(hour: 8), template: template)
    expect(fit_in(requests[3], at: other.starts_at + 10.minutes, on: other).reason).to eq(:fit_in_limit)
  end

  it "limite da cidade vem do perfil" do
    (CityProfile.current || CityProfile.create!(name: "Cidade")).update!(default_fit_in_limit: 0)
    expect(fit_in(requests[0]).reason).to eq(:fit_in_limit)
  end

  it "recusas: justificativa curta, fora do turno, turno cancelado, de outro profissional, tipo não servido, pedido sem unidade" do
    expect(fit_in(requests[0], why: "urgente").reason).to eq(:invalid_reason)
    expect(fit_in(requests[0], at: shift.ends_at - 10.minutes).reason).to eq(:outside_shift)
    expect(fit_in(requests[0], at: shift.starts_at - 10.minutes).reason).to eq(:outside_shift)
    expect(fit_in(requests[0], type: AppointmentType.find_by!(key: "consulta_enfermagem")).reason).to eq(:type_not_served)
    expect(described_class.call(request: requests[0], professional: doctor_link!(unit).professional, shift: shift,
                                starts_at: (shift.starts_at + 10.minutes).iso8601, type: medica, reason: reason,
                                by: reception).reason).to eq(:outside_shift)
    expect(fit_in(triage_request!(citizen(3), unit: nil)).reason).to eq(:wrong_unit)
    shift.update!(cancelled_at: Time.current, cancelled_by_user: reception, cancel_reason: "troca")
    expect(fit_in(requests[0]).reason).to eq(:outside_shift)
  end

  it "turno de outra unidade: outside_shift" do
    other_unit = create_unit("UBS Sul")
    expect(fit_in(triage_request!(citizen(1), unit: other_unit)).reason).to eq(:outside_shift)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/appointments/fit_in_spec.rb`
Expected: FAIL com `uninitialized constant Appointments::FitIn`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/scheduling/fit_in_limit.rb
# Limite de encaixes do turno (ADR 0029 §3.2, §4.3): o do modelo ligado (ativo
# ou não), senão o padrão da cidade, senão 2. Conta os encaixes ATIVOS.
module Scheduling
  module FitInLimit
    DEFAULT = 2

    module_function

    def for(shift)
      shift.schedule_template&.fit_in_limit || CityProfile.current&.default_fit_in_limit || DEFAULT
    end

    def count(shift, except_request_id: nil)
      scope = Appointment.where(shift_id: shift.id, booking_kind: "fit_in", status: Appointment::ACTIVE)
      scope = scope.where.not(request_id: except_request_id) if except_request_id
      scope.count
    end
  end
end
```

```ruby
# app/commands/appointments/fit_in.rb
# Encaixe (ADR 0029 §4.3): horário extra dentro do turno, com justificativa
# (≥ 10 caracteres, imutável, fora de evento e log), contado contra o limite do
# turno sob FOR UPDATE do turno — dois encaixes no último lugar se enfileiram e
# o segundo vê o primeiro. Pode sobrepor vaga ocupada (não entra na EXCLUDE).
module Appointments
  module FitIn
    module_function

    def call(request:, professional:, shift:, starts_at:, type:, reason:, by:, now: Time.current)
      reason = reason.to_s.strip
      return Result.fail(:invalid_reason) if reason.length < Appointment::MIN_FIT_IN_REASON

      at = Placement.parse(starts_at)
      return Result.fail(:invalid_time) if at.nil? || at <= now
      return Result.fail(:outside_shift) if shift.nil? || professional.nil? || shift.professional_id != professional.id
      return Result.fail(:type_not_served) if type.nil? || !type.active
      return Result.fail(:wrong_unit) if request.target_unit_id.nil?

      ends = at + type.duration_minutes.minutes
      ApplicationRecord.transaction do
        Placement.lock!(request)
        next Result.fail(:request_not_open) unless Placement.bookable?(request)

        HealthUnit.lock_active!(request.target_unit_id)
        shift.lock!
        link = shift.professional_link
        next Result.fail(:outside_shift) if shift.cancelled_at || link.health_unit_id != request.target_unit_id
        next Result.fail(:type_not_served) unless Scheduling::AppointmentTypes.serves?(type, link.cbo_code)
        next Result.fail(:outside_shift) unless at >= shift.starts_at && ends <= shift.ends_at
        if Scheduling::FitInLimit.count(shift, except_request_id: request.id) >= Scheduling::FitInLimit.for(shift)
          next Result.fail(:fit_in_limit)
        end
        next Result.fail(:citizen_busy) if Placement.citizen_busy?(request, at, ends)

        appointment = Placement.create!(request: request, at: at, by: by, now: now, booking_kind: "fit_in",
                                        professional_id: professional.id, appointment_type_key: type.key, ends_at: ends,
                                        shift_id: shift.id, fit_in_reason: reason)
        DomainEvents.publish("appointment.booked", appointment_id: appointment.id, request_id: request.id,
                                                   booking_kind: "fit_in")
        DomainEvents.publish("appointment.fit_in_created", appointment_id: appointment.id, shift_id: shift.id)
        Result.ok(appointment: appointment)
      end
    rescue HealthUnit::Inactive
      Result.fail(:invalid_unit)
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/appointments/fit_in_spec.rb spec/commands/appointments/book_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/services/scheduling/fit_in_limit.rb app/commands/appointments/fit_in.rb spec/commands/appointments/fit_in_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: add justified fit-in appointments with a per-shift limit

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Corridas reais (threads) — mesma vaga, último encaixe, cidadão sobreposto

**Files:**
- Create: `spec/commands/appointments/booking_concurrency_spec.rb`

**Interfaces:**
- Consumes: `Appointments::Book`, `Appointments::FitIn` (Tasks 9–10), `wait_for_lock_wait` (`spec/support/lock_wait.rb`).
- Produces: nenhuma interface nova; prova a ordem de travas e a EXCLUDE sob disputa (spec §9, "Concorrência com threads").

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/commands/appointments/booking_concurrency_spec.rb
require "rails_helper"

# ADR 0029 §4.3 sob disputa, com threads reais contra TEST_CITY_A (sem fixture
# transacional), no padrão de spec/commands/attendances/call_next_concurrency_spec.rb.
# A primeira marcação para DENTRO da transação (no publish de appointment.booked,
# depois do INSERT); a segunda tem de esperar o lock e decidir só depois do
# COMMIT da primeira. O after solta as threads e apaga tudo o que commitou.
RSpec.describe "Marcação sob disputa" do
  self.use_transactional_tests = false

  let(:release) { Queue.new }
  let(:threads) { [] }
  let(:ids) { { users: [], pros: [], links: [], shifts: [], citizens: [], conversations: [], triages: [], requests: [] } }

  before do
    CityConnection.with(TEST_CITY_A) do
      Current.set(city: TEST_CITY_A) do
        tag = SecureRandom.hex(4)
        Scheduling::AppointmentTypes.seed_platform!
        admin = User.create!(email_address: "adm-#{tag}@c.gov.br", password: "senha-segura-123")
        ids[:admin] = admin.id
        unit = HealthUnit.create!(name: "UBS Agenda #{tag}", kind: "ubs")
        ids[:unit] = unit.id
        template = ScheduleTemplate.create!(name: "Limite um #{tag}", fit_in_limit: 1, blocks: [])
        ids[:template] = template.id
        starts = (Time.zone.today + 3).in_time_zone.change(hour: 8)
        ids[:starts] = starts
        %w[a b].each_with_index do |suffix, i|
          user = User.create!(email_address: "doc-#{suffix}-#{tag}@c.gov.br", password: "senha-segura-123")
          ids[:users] << user.id
          Membership.create!(user: user, role: "health_professional", granted_at: Time.current)
          pro = Professional.create!(user: user, professional_name: "P#{suffix}", council: "CRM", council_state: "PR",
                                     registration_number: "#{tag.to_i(16).to_s[0, 7]}#{i}",
                                     cns: Professionals::Cns.generate("#{tag}#{suffix}"))
          ids[:pros] << pro.id
          ids[:"pro_#{suffix}"] = pro.id
          link = ProfessionalLink.create!(professional: pro, health_unit: unit, cbo_code: "225125",
                                          started_at: Time.current, started_by_user: admin)
          ids[:links] << link.id
          shift = ProfessionalShift.create!(professional_link: link, professional_id: pro.id, starts_at: starts,
                                            ends_at: starts + 2.hours, created_by_user: admin,
                                            schedule_template: suffix == "b" ? template : nil)
          ids[:shifts] << shift.id
          ids[:"shift_#{suffix}"] = shift.id
        end
        protocol = ProtocolDefinition.create!(
          name: "agenda-#{tag}", version: 1, status: "draft",
          definition: { "name" => "agenda-#{tag}", "version" => 1, "start_step_id" => "q1",
                        "steps" => [ { "id" => "q1", "prompt" => "?", "answer_type" => "boolean",
                                       "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 1, "false" => 0 } } ],
                        "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 9 } } }
        )
        ids[:protocol] = protocol.id
        # r0, r1, r2: três cidadãos; r3: o cidadão de r0, outro tipo (um pedido de triagem vivo por tipo).
        [ [ 0, "consulta_medica" ], [ 1, "consulta_medica" ], [ 2, "consulta_medica" ], [ 0, "retorno" ] ].each do |n, key|
          citizen = ids[:citizens][n] ? Citizen.find(ids[:citizens][n]) :
                      Citizen.create!(cpf: CampaignHistory.cpf_for("agenda-#{tag}-#{n}"), phone: "+55419#{format('%08d', 40_000_000 + n)}")
          ids[:citizens][n] ||= citizen.id
          conversation = Conversation.create!(channel: "web", citizen: citizen, phone: citizen.phone, state: "completed")
          ids[:conversations] << conversation.id
          triage = Triage.create!(conversation: conversation, protocol_definition: protocol, protocol_name: protocol.name,
                                  status: "completed", tier: "baixa", priority: 9, answers: {}, completed_at: Time.current)
          ids[:triages] << triage.id
          request = AppointmentRequest.create!(kind: "triage", origin_triage: triage, root_triage: triage, citizen: citizen,
                                               target_unit: unit, appointment_type_key: key)
          ids[:requests] << request.id
        end
      end
    end
  end

  after do
    3.times { release << true }
    threads.each do |t|
      t.join(5) || t.kill
    rescue StandardError
      nil # join relança a exceção da thread; a limpeza precisa rodar mesmo assim
    end
    CityConnection.with(TEST_CITY_A) do
      ApplicationRecord.transaction do
        ApplicationRecord.connection.execute("SET LOCAL session_replication_role = replica")
        appointment_ids = Appointment.where(request_id: ids[:requests]).pluck(:id)
        DomainEvent.where("payload->>'appointment_id' IN (?)", appointment_ids.presence || [ "" ]).delete_all
        Appointment.where(id: appointment_ids).delete_all
        AppointmentRequest.where(id: ids[:requests]).delete_all
        Triage.where(id: ids[:triages]).delete_all
        Conversation.where(id: ids[:conversations]).delete_all
        Citizen.where(id: ids[:citizens].compact).delete_all
        ProtocolDefinition.where(id: ids[:protocol]).delete_all
        ProfessionalShift.where(id: ids[:shifts]).delete_all
        ScheduleTemplate.where(id: ids[:template]).delete_all
        ProfessionalLink.where(id: ids[:links]).delete_all
        Professional.where(id: ids[:pros]).delete_all
        Membership.where(user_id: ids[:users]).delete_all
        User.where(id: ids[:users] + [ ids[:admin] ].compact).delete_all
        HealthUnit.where(id: ids[:unit]).delete_all
        AppointmentType.delete_all # base semeada aqui; nenhuma outra spec commita tipos
      end
    end
  end

  def in_city(&) = CityConnection.with(TEST_CITY_A) { Current.set(city: TEST_CITY_A, &) }

  # A thread que chama isto para DENTRO da transação, logo depois do INSERT.
  def hold_after_insert!(holding)
    holder = Thread.current
    original = DomainEvents.method(:publish)
    allow(DomainEvents).to receive(:publish) do |*args, **kwargs, &blk|
      result = original.call(*args, **kwargs, &blk)
      if Thread.current == holder && args.first == "appointment.booked"
        holding << true
        release.pop(timeout: 10) or raise "timeout esperando release"
      end
      result
    end
  end

  def book(request_index, pro, at)
    in_city do
      Appointments::Book.call(request: AppointmentRequest.find(ids[:requests][request_index]), professional: Professional.find(ids[pro]),
                              starts_at: at.iso8601, type: AppointmentType.find_by!(key: "consulta_medica"),
                              by: User.find(ids[:admin])).reason || :ok
    end
  end

  def fit_in(request_index, at)
    in_city do
      Appointments::FitIn.call(request: AppointmentRequest.find(ids[:requests][request_index]), professional: Professional.find(ids[:pro_b]),
                               shift: ProfessionalShift.find(ids[:shift_b]), starts_at: at.iso8601,
                               type: AppointmentType.find_by!(key: "consulta_medica"), reason: "retorno que não espera",
                               by: User.find(ids[:admin])).reason || :ok
    end
  end

  # Roda `first` numa thread que segura a transação; `second` noutra; prova que
  # a segunda espera um lock; solta a primeira e devolve [primeira, segunda].
  def race(first, second)
    holding = Queue.new
    outcomes = [ Queue.new, Queue.new ]
    threads << Thread.new { hold_after_insert!(holding); outcomes[0] << first.call }
    holding.pop(timeout: 5) or raise "a primeira marcação não chegou ao INSERT"
    threads << Thread.new { outcomes[1] << second.call }
    expect(wait_for_lock_wait).to be(true)
    release << true
    [ outcomes[0].pop(timeout: 10), outcomes[1].pop(timeout: 10) ]
  end

  it "duas recepções na mesma vaga: uma marca, a outra recebe slot_taken" do
    at = ids[:starts]
    expect(race(-> { book(0, :pro_a, at) }, -> { book(1, :pro_a, at) })).to eq([ :ok, :slot_taken ])
    expect(CityConnection.with(TEST_CITY_A) { Appointment.where(request_id: ids[:requests], status: Appointment::ACTIVE).count }).to eq(1)
  end

  it "dois encaixes no último lugar do limite: um passa, o outro recebe fit_in_limit" do
    at = ids[:starts] + 10.minutes
    expect(race(-> { fit_in(0, at) }, -> { fit_in(1, at + 30.minutes) })).to eq([ :ok, :fit_in_limit ])
  end

  it "o mesmo cidadão em dois profissionais na mesma hora: o segundo recebe citizen_busy" do
    at = ids[:starts]
    in_city { ProfessionalShift.find(ids[:shift_b]).update!(schedule_template_id: nil) }
    expect(race(-> { book(0, :pro_a, at) }, -> { book(3, :pro_b, at) })).to eq([ :ok, :citizen_busy ])
  end
end
```

Notas: o pedido `r3` (`retorno`) existe só para ser um segundo pedido do mesmo cidadão de `r0` (um pedido de triagem vivo por tipo); a marcação escolhe o tipo (`consulta_medica`). O turno `b` nasce com um modelo sem faixas e limite 1 (só encaixe); o exemplo do cidadão tira esse modelo antes da corrida, para o turno `b` ter vagas pelo CBO.

- [ ] **Step 2: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/appointments/booking_concurrency_spec.rb`
Expected: PASS, 3 exemplos. Se algum ficar vermelho por `wait_for_lock_wait` falso, a segunda marcação **não** esperou: a ordem de travas de `Placement.lock!` ou o `shift.lock!` do encaixe está errada — não afrouxe a spec.

Prova de que a spec morde (não commitar): troque `shift.lock!` por `shift.reload` em `FitIn` e rode só o exemplo do encaixe — deve falhar (os dois passam); desfaça. Remova o `Citizen.lock.find` de `Placement.lock!` e rode o exemplo do cidadão — deve falhar; desfaça. Relate as duas falhas no relatório da task.

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add spec/commands/appointments/booking_concurrency_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "test: race slot booking, last fit-in and overlapping citizen bookings

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Marcação na API — três formas, `use_slots` e o horário na forma única (§4.4)

**Files:**
- Create: `app/services/scheduling/appointment_presenter.rb`
- Modify: `app/commands/appointments/schedule.rb`
- Modify: `app/controllers/appointment_requests_controller.rb`
- Test: `spec/services/scheduling/appointment_presenter_spec.rb`, `spec/requests/booking_spec.rb`, `spec/commands/appointments/schedule_spec.rb`

**Interfaces:**
- Consumes: `Book`, `FitIn` (Tasks 9–10), `Scheduling::Transition.slots_day?` (Task 8), `Availability.blocks_for/.shift_data`.
- Produces:
  - `Scheduling::AppointmentPresenter.new(show_reason:, catalog: AppointmentTypes.catalog, zone: Time.zone)#call(appointment) -> Hash` na forma de contratos §4.4 + §9 (`legacy` com `ends_at`, `appointment_type_key/name`, `professional` e `shift_id` nulos; `fit_in_reason` só com `show_reason` e em encaixe; `outside_template` = `slot` que não cabe mais numa faixa `bookable` do seu tipo no modelo atual; `shift_cancelled`). Pede `includes(:citizen, :professional, shift: [:professional_link, :schedule_template])` de quem chama.
  - `Appointments::Schedule` (livre) recusa `:use_slots` em dia com turno da unidade.
  - `POST /attendance/requests/:id/appointments` com `kind` `slot` | `fit_in` | `legacy` (ausente = `legacy`), `health_unit_id` nas três (contratos §9) → 201 `{ appointment: <§4.4> + confirmation_deadline_at }`; 409 `slot_taken` (livre: com `taken`), `slot_unavailable`, `citizen_busy`, `fit_in_limit`, `use_slots`, `request_not_open`; 422 `invalid_reason`, `type_not_served`, `outside_shift`, `wrong_unit`, `invalid_time`, `invalid_kind`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/scheduling/appointment_presenter_spec.rb
require "rails_helper"

RSpec.describe Scheduling::AppointmentPresenter do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:link) { doctor_link!(unit) }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:day) { Time.zone.today + 4 }
  let(:template) do
    ScheduleTemplate.create!(name: "Manhã", blocks: [ { "starts" => "08:00", "ends" => "09:00", "kind" => "bookable",
                                                       "appointment_type_key" => "consulta_medica" } ])
  end
  let(:shift) { shift!(link, starts_at: day.in_time_zone.change(hour: 8), template: template) }

  it "slot: forma única, sem justificativa; modelo editado marca outside_template; turno cancelado marca shift_cancelled" do
    appt = appointment_row!(triage_request!(citizen, unit: unit), shift, starts_at: shift.starts_at)
    json = described_class.new(show_reason: true).call(appt)
    expect(json).to eq(
      id: appt.id, status: "confirmed", booking_kind: "slot", scheduled_at: appt.scheduled_at.iso8601,
      ends_at: appt.ends_at.iso8601, appointment_type_key: "consulta_medica", appointment_type_name: "Consulta médica",
      professional: { id: link.professional_id, name: link.professional.professional_name }, shift_id: shift.id,
      fit_in: false, outside_template: false, shift_cancelled: false,
      citizen: { id: citizen.id, cpf_masked: citizen.cpf_masked }
    )
    template.update!(blocks: [ { "starts" => "10:00", "ends" => "11:00", "kind" => "blocked" } ])
    shift.update!(cancelled_at: Time.current, cancelled_by_user: link.started_by_user, cancel_reason: "troca")
    json = described_class.new(show_reason: true).call(appt.reload)
    expect(json).to include(outside_template: true, shift_cancelled: true, scheduled_at: appt.scheduled_at.iso8601)
  end

  it "encaixe: justificativa só para quem marca; legacy com campos nulos" do
    fit = appointment_row!(triage_request!(citizen, unit: unit), shift, starts_at: shift.starts_at, kind: "fit_in",
                                                                      reason: "retorno que não espera")
    expect(described_class.new(show_reason: true).call(fit)).to include(fit_in: true, fit_in_reason: "retorno que não espera",
                                                                         outside_template: false)
    expect(described_class.new(show_reason: false).call(fit)).not_to have_key(:fit_in_reason)

    other = triage_request!(Citizen.create!(cpf: "11144477735", phone: "+5541911112222"), unit: unit)
    legacy = Appointment.create!(request: other, citizen: other.citizen, health_unit: unit, scheduled_at: shift.starts_at,
                                 scheduled_by_user: link.started_by_user, status: "confirmed", confirmed_at: Time.current)
    expect(described_class.new(show_reason: true).call(legacy))
      .to include(booking_kind: "legacy", ends_at: nil, appointment_type_key: nil, appointment_type_name: nil,
                  professional: nil, shift_id: nil, fit_in: false, outside_template: false, shift_cancelled: false)
  end
end
```

```ruby
# spec/requests/booking_spec.rb
require "rails_helper"

# Contratos §4.3, §9: três formas de marcar; health_unit_id nas três.
RSpec.describe "POST /attendance/requests/:id/appointments (agenda)", type: :request do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:link) { doctor_link!(unit) }
  let(:reception) { staff_with("recepcao-marca@cidade.gov.br", "citizen_verifier") }
  let(:day) { Time.zone.today + 5 }
  let!(:shift) { shift!(link, starts_at: day.in_time_zone.change(hour: 8), ends_at: day.in_time_zone.change(hour: 10)) }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:req) { triage_request!(citizen, unit: unit) }
  def body = JSON.parse(response.body)

  def slot_body(at = shift.starts_at)
    { kind: "slot", health_unit_id: unit.id, professional_id: link.professional_id, starts_at: at.iso8601,
      appointment_type_key: "consulta_medica" }
  end

  before { sign_in_as(reception) }

  it "vaga: 201 com o horário na forma única; remarcar move o anterior" do
    json_post "/attendance/requests/#{req.id}/appointments", slot_body
    expect(response).to have_http_status(:created)
    first_id = body.dig("appointment", "id")
    expect(body["appointment"]).to include("booking_kind" => "slot", "status" => "scheduled", "shift_id" => shift.id,
                                           "appointment_type_name" => "Consulta médica", "fit_in" => false)
    expect(body["appointment"]).to have_key("confirmation_deadline_at")

    json_post "/attendance/requests/#{req.id}/appointments", slot_body(shift.starts_at + 40.minutes)
    expect(response).to have_http_status(:created)
    expect(Appointment.find(first_id).status).to eq("moved")
  end

  it "encaixe: 201 com a justificativa visível para a recepção; justificativa curta 422 invalid_reason" do
    json_post "/attendance/requests/#{req.id}/appointments",
              slot_body.merge(kind: "fit_in", shift_id: shift.id, reason: "curta")
    expect(response).to have_http_status(:unprocessable_entity)
    expect(body).to eq("error" => "invalid_reason")
    json_post "/attendance/requests/#{req.id}/appointments",
              slot_body(shift.starts_at + 5.minutes).merge(kind: "fit_in", shift_id: shift.id, reason: "gestante com dor")
    expect(response).to have_http_status(:created)
    expect(body["appointment"]).to include("booking_kind" => "fit_in", "fit_in" => true, "fit_in_reason" => "gestante com dor")
  end

  it "livre: em dia com turno 409 use_slots; em dia sem turno (e sem kind) marca como hoje" do
    json_post "/attendance/requests/#{req.id}/appointments",
              kind: "legacy", health_unit_id: unit.id, scheduled_at: shift.starts_at.change(hour: 14).iso8601
    expect(response).to have_http_status(:conflict)
    expect(body).to eq("error" => "use_slots")

    json_post "/attendance/requests/#{req.id}/appointments",
              health_unit_id: unit.id, scheduled_at: (shift.starts_at + 1.day).change(hour: 14).iso8601
    expect(response).to have_http_status(:created)
    expect(body["appointment"]).to include("booking_kind" => "legacy", "ends_at" => nil, "professional" => nil)
  end

  it "unidade diferente da do pedido: 422 wrong_unit; vaga fora da grade 409; kind desconhecido 422" do
    json_post "/attendance/requests/#{req.id}/appointments", slot_body.merge(health_unit_id: create_unit("UBS Sul").id)
    expect(body).to eq("error" => "wrong_unit")
    json_post "/attendance/requests/#{req.id}/appointments", slot_body(shift.starts_at + 5.minutes)
    expect(response).to have_http_status(:conflict)
    expect(body).to eq("error" => "slot_unavailable")
    json_post "/attendance/requests/#{req.id}/appointments", slot_body.merge(kind: "grupo")
    expect(body).to eq("error" => "invalid_kind")
  end

  it "violação da trava vira 409 slot_taken (nunca 500)" do
    slot = Scheduling::Availability.for(unit: unit, from: day, to: day, appointment_type: AppointmentType.find_by!(key: "consulta_medica")).first
    appointment_row!(triage_request!(Citizen.create!(cpf: "11144477735", phone: "+5541911112222"), unit: unit), shift,
                     starts_at: shift.starts_at)
    allow(Scheduling::Availability).to receive(:for).and_return([ slot ])
    json_post "/attendance/requests/#{req.id}/appointments", slot_body
    expect(response).to have_http_status(:conflict)
    expect(body).to eq("error" => "slot_taken")
  end
end
```

Em `spec/commands/appointments/schedule_spec.rb`, acrescente:

```ruby
  it "dia com turno não cancelado na unidade: use_slots (ADR 0029, transição)" do
    ensure_appointment_types!
    req = request_for(in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor))
    at = 3.days.from_now.change(hour: 14)
    shift!(ProfessionalLink.find_by!(health_unit: unit), starts_at: at.change(hour: 8))
    expect(Appointments::Schedule.call(request: req, scheduled_at: at.iso8601, health_unit_id: unit.id, by: reception).reason)
      .to eq(:use_slots)
  end
```

(Se os `let` desse arquivo tiverem outros nomes, use os dele para unidade, recepção, profissional e cidadão; o vínculo do profissional vem do `link_professional!` do `before`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/services/scheduling/appointment_presenter_spec.rb spec/requests/booking_spec.rb spec/commands/appointments/schedule_spec.rb`
Expected: FAIL (`uninitialized constant Scheduling::AppointmentPresenter`; o controller ainda ignora `kind`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/scheduling/appointment_presenter.rb
# Horário na forma única (contratos §4.4, §9): agenda da unidade, fila e Minha
# agenda. `legacy` não tem fim, tipo, profissional nem turno. A justificativa do
# encaixe só vai para quem marca e para o municipal_admin (show_reason).
# Guarda as faixas por turno: uma agenda inteira calcula cada turno uma vez.
module Scheduling
  class AppointmentPresenter
    def initialize(show_reason:, catalog: AppointmentTypes.catalog, zone: Time.zone)
      @show_reason = show_reason
      @catalog = catalog
      @zone = zone
      @blocks = {}
    end

    def call(appointment)
      legacy = appointment.booking_kind == "legacy"
      professional = appointment.professional unless legacy
      json = {
        id: appointment.id, status: appointment.status, booking_kind: appointment.booking_kind,
        scheduled_at: appointment.scheduled_at.iso8601, ends_at: legacy ? nil : appointment.ends_at&.iso8601,
        appointment_type_key: legacy ? nil : appointment.appointment_type_key,
        appointment_type_name: legacy ? nil : @catalog.name_for(appointment.appointment_type_key),
        professional: professional && { id: professional.id, name: professional.professional_name },
        shift_id: legacy ? nil : appointment.shift_id, fit_in: appointment.fit_in?,
        outside_template: outside_template?(appointment), shift_cancelled: appointment.shift&.cancelled_at.present? || false,
        citizen: { id: appointment.citizen_id, cpf_masked: appointment.citizen.cpf_masked }
      }
      json[:fit_in_reason] = appointment.fit_in_reason if @show_reason && appointment.fit_in?
      json
    end

    private

    def outside_template?(appointment)
      return false unless appointment.booking_kind == "slot" && appointment.shift

      blocks = (@blocks[appointment.shift_id] ||= Availability.blocks_for(
        Availability.shift_data(appointment.shift), types: @catalog.active, fallback: @catalog.fallback, zone: @zone
      ))
      blocks.none? do |b|
        b.kind == "bookable" && b.appointment_type_key == appointment.appointment_type_key &&
          b.starts_at <= appointment.scheduled_at && b.ends_at >= appointment.ends_at
      end
    end
  end
end
```

Em `app/commands/appointments/schedule.rb`, dentro da transação, logo depois de `next Result.fail(:request_not_open) unless request.status == "open"`:

```ruby
        # Transição (ADR 0029): a marcação livre só vale em dia sem turno da unidade.
        next Result.fail(:use_slots) if Scheduling::Transition.slots_day?(request.target_unit_id, at.to_date)
```

e acrescente ao comentário do topo: `# Módulo 17 (ADR 0029): este é o caminho "legacy" — só em dia sem turno.`

Em `app/controllers/appointment_requests_controller.rb`:

```ruby
  ERROR_STATUS = {
    invalid_time: :unprocessable_entity, request_not_open: :conflict, wrong_unit: :unprocessable_entity,
    invalid_unit: :unprocessable_entity, reason_too_short: :unprocessable_entity, slot_taken: :conflict,
    slot_unavailable: :conflict, citizen_busy: :conflict, fit_in_limit: :conflict, use_slots: :conflict,
    invalid_reason: :unprocessable_entity, type_not_served: :unprocessable_entity, outside_shift: :unprocessable_entity,
    invalid_kind: :unprocessable_entity, already_assigned: :conflict
  }.freeze

  def schedule
    request = AppointmentRequest.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless request

    result = book(request, request.request_parameters)
    return render_failure(result, ERROR_STATUS) if result.failure?

    appointment = Appointment.includes(:citizen, :professional, shift: %i[professional_link schedule_template])
                             .find(result.payload[:appointment].id)
    render json: { appointment: presenter.call(appointment)
                                         .merge(confirmation_deadline_at: appointment.confirmation_deadline_at&.iso8601) },
           status: :created
  end
```

e, em `private`:

```ruby
  # Contratos §4.3, §9: health_unit_id vai nas três formas; ausência de kind = legacy.
  def book(request, body)
    kind = body["kind"].presence || "legacy"
    return Result.fail(:invalid_kind) unless Appointment::BOOKING_KINDS.include?(kind)
    if kind == "legacy"
      return Appointments::Schedule.call(request: request, scheduled_at: body["scheduled_at"],
                                         health_unit_id: body["health_unit_id"], by: Current.user,
                                         allow_overlap: body["allow_overlap"] == true)
    end
    return Result.fail(:wrong_unit) if request.target_unit_id.nil? || body["health_unit_id"].to_s != request.target_unit_id

    professional = Professional.find_by(id: body["professional_id"].to_s)
    type = AppointmentType.find_by(key: body["appointment_type_key"].to_s)
    if kind == "slot"
      Appointments::Book.call(request: request, professional: professional, starts_at: body["starts_at"], type: type,
                              by: Current.user)
    else
      Appointments::FitIn.call(request: request, professional: professional,
                               shift: ProfessionalShift.find_by(id: body["shift_id"].to_s), starts_at: body["starts_at"],
                               type: type, reason: body["reason"], by: Current.user)
    end
  end

  # Quem chega aqui marca (require_verifier): vê a justificativa do encaixe.
  def presenter = @presenter ||= Scheduling::AppointmentPresenter.new(show_reason: true)
```

O `appointment_json` antigo deixa de ser usado por `schedule`; remova-o se nada mais o chamar.

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/services/scheduling/appointment_presenter_spec.rb spec/requests/booking_spec.rb spec/requests/appointment_requests_spec.rb spec/commands/appointments spec/invariants/appointment_invariants_spec.rb`
Expected: PASS (as specs do módulo 08 continuam verdes: nenhuma cria turno, então todo dia delas é `legacy`).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/services/scheduling/appointment_presenter.rb app/commands/appointments/schedule.rb app/controllers/appointment_requests_controller.rb spec/services/scheduling/appointment_presenter_spec.rb spec/requests/booking_spec.rb spec/commands/appointments/schedule_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: book slots, fit-ins or free times from the request queue

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Agendas — da unidade (recepção) e Minha agenda (profissional)

**Files:**
- Modify: `app/services/scheduling/block_json.rb` (`for_day`)
- Create: `app/services/scheduling/unit_agenda.rb`, `app/services/scheduling/professional_agenda.rb`
- Create: `app/controllers/professional_agenda_controller.rb`
- Modify: `app/controllers/appointment_requests_controller.rb` (`agenda`), `config/routes.rb`
- Test: `spec/requests/unit_agenda_spec.rb`, `spec/requests/my_agenda_spec.rb`

**Interfaces:**
- Consumes: `AppointmentPresenter` (Task 12), `Availability.blocks_for/.shift_data`, `FitInLimit` (Task 10), `DateRange` (Task 8).
- Produces:
  - `Scheduling::BlockJson.for_day(blocks, day:, zone:, catalog:) -> Array<Hash>` (só o pedaço de cada faixa que cai no dia);
  - `Scheduling::UnitAgenda.call(unit:, date:) -> Hash` = contratos §4.5 + §9: `{ date, professionals: [ { id, name, shifts: [ { shift_id, starts_at, ends_at, cancelled_at, blocks, fit_in_count, fit_in_limit } ], appointments: [§4.4] } ], unassigned: [§4.4 legacy], appointments: [ forma antiga ] }` (a chave antiga `appointments` fica para o dashboard em produção até o novo entrar; ver Desvios);
  - `Scheduling::ProfessionalAgenda.call(professional:, from:, to:) -> { days: [ { date, shifts: [ { shift_id, starts_at, ends_at, cancelled_at, unit: { id, name }, blocks, appointments } ] } ] }` (sem `fit_in_reason`);
  - `GET /attendance/units/:id/agenda?date=` (forma nova) e `GET /professionals/me/agenda?from=&to=` (até 7 dias, inclusivos; 404 `no_profile`; 422 `invalid_range`).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/requests/unit_agenda_spec.rb
require "rails_helper"

RSpec.describe "GET /attendance/units/:id/agenda (por profissional)", type: :request do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:link) { doctor_link!(unit) }
  let(:reception) { staff_with("recepcao-agenda@cidade.gov.br", "citizen_verifier") }
  let(:day) { Time.zone.today + 4 }
  let(:template) do
    ScheduleTemplate.create!(name: "Manhã", fit_in_limit: 3, blocks: [
      { "starts" => "07:00", "ends" => "08:00", "kind" => "walk_in" },
      { "starts" => "08:00", "ends" => "09:00", "kind" => "bookable", "appointment_type_key" => "consulta_medica" }
    ])
  end
  let(:shift) { shift!(link, starts_at: day.in_time_zone.change(hour: 7), ends_at: day.in_time_zone.change(hour: 11), template: template) }
  let(:cpfs) { %w[52998224725 11144477735 39053344705] }
  def citizen(i) = Citizen.create!(cpf: cpfs[i], phone: "+55419#{format('%08d', 50_000_000 + i)}")
  def body = JSON.parse(response.body)

  it "turnos com faixas e contador de encaixe, horários por profissional, livres sem profissional" do
    slot = appointment_row!(triage_request!(citizen(0), unit: unit), shift, starts_at: shift.starts_at + 1.hour)
    fit = appointment_row!(triage_request!(citizen(1), unit: unit), shift, starts_at: shift.starts_at + 1.hour,
                           kind: "fit_in", reason: "gestante com dor")
    other = triage_request!(citizen(2), unit: unit)
    legacy = Appointment.create!(request: other, citizen: other.citizen, health_unit: unit,
                                 scheduled_at: shift.starts_at + 3.hours, scheduled_by_user: reception,
                                 status: "confirmed", confirmed_at: Time.current)

    sign_in_as(reception)
    get "/attendance/units/#{unit.id}/agenda", params: { date: day.iso8601 }
    expect(response).to have_http_status(:ok)
    pro = body["professionals"].sole
    expect(pro).to include("id" => link.professional_id, "name" => link.professional.professional_name)
    expect(pro["shifts"].sole).to eq(
      "shift_id" => shift.id, "starts_at" => shift.starts_at.iso8601, "ends_at" => shift.ends_at.iso8601,
      "cancelled_at" => nil, "fit_in_count" => 1, "fit_in_limit" => 3,
      "blocks" => [ { "starts" => "07:00", "ends" => "08:00", "kind" => "walk_in" },
                    { "starts" => "08:00", "ends" => "09:00", "kind" => "bookable", "appointment_type_key" => "consulta_medica",
                      "appointment_type_name" => "Consulta médica" } ]
    )
    expect(pro["appointments"].map { |a| a["id"] }).to contain_exactly(slot.id, fit.id)
    expect(pro["appointments"].find { |a| a["id"] == fit.id }["fit_in_reason"]).to eq("gestante com dor")
    expect(body["unassigned"].map { |a| [ a["id"], a["booking_kind"] ] }).to eq([ [ legacy.id, "legacy" ] ])
    expect(body["appointments"].map { |a| a["id"] }).to contain_exactly(slot.id, fit.id, legacy.id)
  end

  it "turno cancelado e modelo editado: o horário fica, marcado (Review Focus 3)" do
    slot = appointment_row!(triage_request!(citizen(0), unit: unit), shift, starts_at: shift.starts_at + 1.hour)
    template.update!(blocks: [ { "starts" => "07:00", "ends" => "11:00", "kind" => "walk_in" } ])
    shift.update!(cancelled_at: Time.current, cancelled_by_user: reception, cancel_reason: "troca de escala")

    sign_in_as(reception)
    get "/attendance/units/#{unit.id}/agenda", params: { date: day.iso8601 }
    pro = body["professionals"].sole
    expect(pro["shifts"].sole["cancelled_at"]).to be_present
    expect(pro["appointments"].sole).to include("id" => slot.id, "status" => "confirmed", "outside_template" => true,
                                               "shift_cancelled" => true, "scheduled_at" => slot.scheduled_at.iso8601)
  end
end
```

```ruby
# spec/requests/my_agenda_spec.rb
require "rails_helper"

RSpec.describe "GET /professionals/me/agenda", type: :request do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:link) { doctor_link!(unit) }
  let(:doctor) { link.professional.user }
  let(:day) { Time.zone.today + 1 }
  def body = JSON.parse(response.body)

  it "dias do intervalo, turnos com unidade e faixas, horários sem a justificativa do encaixe" do
    shift = shift!(link, starts_at: day.in_time_zone.change(hour: 8))
    fit = appointment_row!(triage_request!(Citizen.create!(cpf: "52998224725", phone: "+5541998765432"), unit: unit), shift,
                           starts_at: shift.starts_at, kind: "fit_in", reason: "gestante com dor")
    sign_in_as(doctor)
    get "/professionals/me/agenda", params: { from: day.iso8601, to: (day + 1).iso8601 }
    expect(response).to have_http_status(:ok)
    expect(body["days"].map { |d| d["date"] }).to eq([ day.iso8601, (day + 1).iso8601 ])
    first = body["days"].first["shifts"].sole
    expect(first).to include("shift_id" => shift.id, "unit" => { "id" => unit.id, "name" => unit.name }, "cancelled_at" => nil)
    expect(first["blocks"]).to eq([ { "starts" => "08:00", "ends" => "12:00", "kind" => "bookable",
                                      "appointment_type_key" => "consulta_medica", "appointment_type_name" => "Consulta médica" } ])
    expect(first["appointments"].sole).to include("id" => fit.id, "fit_in" => true)
    expect(first["appointments"].sole).not_to have_key("fit_in_reason")
    expect(body["days"].last["shifts"]).to eq([])
  end

  it "sem cadastro profissional 404 no_profile; mais de 7 dias 422; recepção 403" do
    sign_in_as(staff_with("medica-sem-cadastro@cidade.gov.br", "health_professional"))
    get "/professionals/me/agenda"
    expect(response).to have_http_status(:not_found)
    expect(body).to eq("error" => "no_profile")

    sign_in_as(doctor)
    get "/professionals/me/agenda", params: { from: day.iso8601, to: (day + 7).iso8601 }
    expect(body).to eq("error" => "invalid_range")

    sign_in_as(staff_with("recepcao-minha@cidade.gov.br", "citizen_verifier"))
    get "/professionals/me/agenda"
    expect(response).to have_http_status(:forbidden)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/requests/unit_agenda_spec.rb spec/requests/my_agenda_spec.rb`
Expected: FAIL (agenda na forma antiga; rota `me/agenda` inexistente).

- [ ] **Step 3: Implemente**

Em `app/services/scheduling/block_json.rb`, dentro do módulo:

```ruby
    # Só o pedaço de cada faixa que cai no dia local `day`.
    def for_day(blocks, day:, zone:, catalog:)
      day_start = zone.local(day.year, day.month, day.day)
      day_end = day_start + 1.day
      blocks.filter_map do |block|
        starts = [ block.starts_at, day_start ].max
        ends = [ block.ends_at, day_end ].min
        next unless ends > starts

        json(block, starts.in_time_zone(zone).strftime("%H:%M"),
             ends >= day_end ? "24:00" : ends.in_time_zone(zone).strftime("%H:%M"), catalog)
      end
    end
```

```ruby
# app/services/scheduling/unit_agenda.rb
# Agenda da unidade no dia (contratos §4.5, §9): por profissional, os turnos que
# cruzam o dia (cancelados também, com cancelled_at), as faixas do dia, o
# contador de encaixes e os horários; os livres (legacy) sem profissional em
# `unassigned`. `appointments` (forma antiga do módulo 08) fica até o dashboard
# novo entrar. Quem lê é a recepção: vê a justificativa do encaixe.
module Scheduling
  module UnitAgenda
    module_function

    def call(unit:, date:, catalog: AppointmentTypes.catalog, zone: Time.zone)
      window = date.in_time_zone.all_day
      shifts = ProfessionalShift.joins(:professional_link).where(professional_links: { health_unit_id: unit.id })
                                .where("professional_shifts.starts_at <= ? AND professional_shifts.ends_at > ?",
                                       window.end, window.begin)
                                .includes(:professional, :professional_link, :schedule_template).order(:starts_at).to_a
      appointments = Appointment.where(health_unit: unit, scheduled_at: window)
                                .includes(:citizen, :request, :professional, shift: %i[professional_link schedule_template])
                                .order(:scheduled_at).to_a
      presenter = AppointmentPresenter.new(show_reason: true, catalog: catalog, zone: zone)
      professionals = (shifts.map(&:professional) + appointments.filter_map(&:professional)).uniq
                                                                                          .sort_by { |p| [ p.professional_name, p.id ] }
      {
        date: date.iso8601,
        professionals: professionals.map do |p|
          { id: p.id, name: p.professional_name,
            shifts: shifts.select { |s| s.professional_id == p.id }.map { |s| shift_json(s, date, catalog, zone) },
            appointments: appointments.select { |a| a.professional_id == p.id }.map { |a| presenter.call(a) } }
        end,
        unassigned: appointments.select { |a| a.professional_id.nil? }.map { |a| presenter.call(a) },
        appointments: appointments.map { |a| legacy_json(a) }
      }
    end

    def shift_json(shift, date, catalog, zone)
      blocks = Availability.blocks_for(Availability.shift_data(shift), types: catalog.active, fallback: catalog.fallback,
                                       zone: zone)
      { shift_id: shift.id, starts_at: shift.starts_at.iso8601, ends_at: shift.ends_at.iso8601,
        cancelled_at: shift.cancelled_at&.iso8601, blocks: BlockJson.for_day(blocks, day: date, zone: zone, catalog: catalog),
        fit_in_count: FitInLimit.count(shift), fit_in_limit: FitInLimit.for(shift) }
    end

    def legacy_json(a)
      { id: a.id, scheduled_at: a.scheduled_at.iso8601, cpf_masked: a.citizen.cpf_masked, kind: a.request.kind,
        status: a.status }
    end
  end
end
```

```ruby
# app/services/scheduling/professional_agenda.rb
# Minha agenda (ADR 0029 §7; contratos §3, §9): só leitura, os turnos do
# profissional em cada dia do intervalo (cancelados também), com unidade,
# faixas do dia e os horários daquele turno no dia. Sem justificativa de encaixe.
module Scheduling
  module ProfessionalAgenda
    module_function

    def call(professional:, from:, to:, catalog: AppointmentTypes.catalog, zone: Time.zone)
      window = from.in_time_zone.beginning_of_day..to.in_time_zone.end_of_day
      shifts = ProfessionalShift.where(professional: professional)
                                .where("starts_at <= ? AND ends_at > ?", window.end, window.begin)
                                .includes(:schedule_template, professional_link: :health_unit).order(:starts_at).to_a
      appointments = Appointment.where(shift_id: shifts.map(&:id))
                                .includes(:citizen, :professional, shift: %i[professional_link schedule_template])
                                .order(:scheduled_at).to_a
      presenter = AppointmentPresenter.new(show_reason: false, catalog: catalog, zone: zone)
      { days: (from..to).map { |day| day_json(day, shifts, appointments, presenter, catalog, zone) } }
    end

    def day_json(day, shifts, appointments, presenter, catalog, zone)
      day_window = day.in_time_zone.all_day
      today = shifts.select { |s| s.starts_at <= day_window.end && s.ends_at > day_window.begin }
      { date: day.iso8601, shifts: today.map do |shift|
        unit = shift.professional_link.health_unit
        blocks = Availability.blocks_for(Availability.shift_data(shift), types: catalog.active, fallback: catalog.fallback,
                                         zone: zone)
        { shift_id: shift.id, starts_at: shift.starts_at.iso8601, ends_at: shift.ends_at.iso8601,
          cancelled_at: shift.cancelled_at&.iso8601, unit: { id: unit.id, name: unit.name },
          blocks: BlockJson.for_day(blocks, day: day, zone: zone, catalog: catalog),
          appointments: appointments.select { |a| a.shift_id == shift.id && day_window.cover?(a.scheduled_at) }
                                    .map { |a| presenter.call(a) } }
      end }
    end
  end
end
```

```ruby
# app/controllers/professional_agenda_controller.rb
# GET /professionals/me/agenda?from=&to= (ADR 0029 §7; contratos §3, §9): até 7
# dias, inclusivos; o papel health_professional e o cadastro profissional.
class ProfessionalAgendaController < ApplicationController
  include Authentication
  include AttendanceAccess

  MAX_DAYS = 7

  before_action :require_professional

  def show
    professional = Current.user.professional
    return render(json: { error: "no_profile" }, status: :not_found) unless professional

    range = Scheduling::DateRange.parse(params[:from], params[:to], default_days: MAX_DAYS, max_days: MAX_DAYS)
    return render(json: { error: "invalid_range" }, status: :unprocessable_entity) unless range

    render json: Scheduling::ProfessionalAgenda.call(professional: professional, from: range.begin, to: range.end)
  end
end
```

Em `app/controllers/appointment_requests_controller.rb`, `agenda` passa a:

```ruby
  def agenda
    unit = HealthUnit.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless unit

    day = (Date.iso8601(params[:date].to_s) rescue Time.zone.today)
    render json: Scheduling::UnitAgenda.call(unit: unit, date: day)
  end
```

(`agenda_json` sai do controller: a forma antiga mora em `UnitAgenda.legacy_json`.)

Em `config/routes.rb`, no `scope "/professionals"`, junto de `get "me"` e antes de `get ":id"`: `get "me/agenda", to: "professional_agenda#show"`.

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/requests/unit_agenda_spec.rb spec/requests/my_agenda_spec.rb spec/requests/appointment_requests_spec.rb spec/requests/professionals_spec.rb`
Expected: PASS (as specs antigas da agenda leem `appointments`, que continua).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/services/scheduling/block_json.rb app/services/scheduling/unit_agenda.rb app/services/scheduling/professional_agenda.rb app/controllers/professional_agenda_controller.rb app/controllers/appointment_requests_controller.rb config/routes.rb spec/requests/unit_agenda_spec.rb spec/requests/my_agenda_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: show the unit agenda by professional and the professional's own agenda

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 5 — Pedido da triagem e fila (F-17.5, F-17.6)

### Task 14: Pedido gerado pela triagem (`Triages::Schedule`) e campos novos nos pedidos de atendimento

**Files:**
- Create: `app/commands/triages/schedule.rb`
- Modify: `app/commands/complete_triage.rb`
- Modify: `app/commands/appointment_requests/lifecycle.rb`
- Modify: `app/commands/health_units/drain.rb`
- Modify: `spec/support/triage_catalog_helpers.rb` (`scheduling:` em `catalog_definition`)
- Test: `spec/commands/triages/schedule_spec.rb`, `spec/commands/health_units/drain_spec.rb` (acréscimo)

**Interfaces:**
- Consumes: `Protocols::ConditionContext.build`, `Protocols::Condition.eval`, `Protocols::Urgency.urgent?`, `Territory::ReferenceUnits.for`, `Citizen#profile_context` (módulos 11 e 15); `AppointmentRequestTriage` (Task 2).
- Produces:
  - `Triages::Schedule.call(triage:, outcome:, on: Time.zone.today) -> AppointmentRequest|nil` — chamado por `CompleteTriage` dentro da transação, depois de `Triages::Suggest`; eventos `appointment_request.created_from_triage { request_id, triage_id }` e `appointment_request.merged_triage { request_id, triage_id }`;
  - `AppointmentRequests::Lifecycle.open_for!` grava `appointment_type_key: "retorno"`, `priority: "routine"`, `due_on: hoje + 30`;
  - `HealthUnits::Drain` copia para o pedido novo `origin_triage_id`, `appointment_type_key`, `priority`, `due_on`, `reschedule_*`, `preferred_period`, `reschedule_count` e as linhas de `appointment_request_triages`;
  - helper `catalog_definition(name, ..., scheduling: nil)` / `active_protocol!(name, scheduling: [...])`.

- [ ] **Step 1: Ajuste o helper do módulo 15**

Em `spec/support/triage_catalog_helpers.rb`, `catalog_definition` ganha o parâmetro `scheduling: nil` e a chave `"scheduling" => scheduling` no hash (antes do `.compact`). Assinatura nova: `def catalog_definition(name, offer: nil, suggestions: nil, priority_when: nil, scheduling: nil)`.

- [ ] **Step 2: Escreva a spec que falha**

```ruby
# spec/commands/triages/schedule_spec.rb
require "rails_helper"

# ADR 0029 §5.2 (spec §9 "Pedido da triagem"): primeira regra que casa; urgente
# nunca; unidade de referência do bairro ou fila sem unidade; não duplica;
# tipo inexistente não perde o pedido.
RSpec.describe Triages::Schedule do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset; Rails.cache.clear }

  let(:rules) do
    [ { "when" => { "eq" => ["q1", "false"] }, "appointment_type" => "consulta_enfermagem", "priority" => "routine", "due_in_days" => 60 },
      { "when" => { "gte" => ["outcome.score", 4] }, "appointment_type" => "consulta_medica", "priority" => "priority", "due_in_days" => 7 },
      { "when" => { "gte" => ["outcome.score", 1] }, "appointment_type" => "retorno", "priority" => "routine", "due_in_days" => 30 } ]
  end
  let(:neighborhood) { Neighborhood.create!(name: "Batel", source: "seed") }
  let(:unit) { create_unit("UBS Batel").tap { |u| NeighborhoodCoverage.create!(neighborhood: neighborhood, health_unit: u) } }
  let(:par) { profiled_citizen!(age: 70, neighborhood: neighborhood) }

  def complete(citizen, answer, protocol = "saude-do-idoso")
    started = start_for!(citizen, protocol).payload
    Citizens::SubmitAnswer.call(conversation: started[:conversation], answer: answer, idempotency_key: SecureRandom.uuid)
    started[:triage].reload
  end

  def requests_of(citizen) = AppointmentRequest.where(citizen: citizen)

  it "a primeira regra que casa gera o pedido na unidade de referência, com evento só de ids" do
    unit
    active_protocol!("saude-do-idoso", scheduling: rules)
    triage = complete(par, "true")
    request = requests_of(par).sole
    expect(request).to have_attributes(kind: "triage", origin_triage_id: triage.id, root_triage_id: triage.id,
                                       origin_attendance_id: nil, origin_unit_id: nil, target_unit_id: unit.id,
                                       appointment_type_key: "consulta_medica", priority: "priority",
                                       due_on: Time.zone.today + 7, status: "open")
    expect(DomainEvent.where(name: "appointment_request.created_from_triage").sole.payload)
      .to eq("request_id" => request.id, "triage_id" => triage.id)
  end

  it "nenhuma regra: só orientação; resultado urgente: nunca" do
    active_protocol!("saude-do-idoso", scheduling: [ rules[1] ],
                                       priority_when: [ { "when" => { "eq" => ["q1", "true"] }, "priority" => 1 } ])
    expect(complete(par, "true").priority).to eq(1)
    expect(requests_of(par)).to be_empty
    complete(par, "false")
    expect(requests_of(par)).to be_empty
  end

  it "sem bairro (ou sem unidade de referência): fila sem unidade" do
    active_protocol!("saude-do-idoso", scheduling: rules)
    sem_bairro = profiled_citizen!(age: 70, phone: "+5541977776666")
    complete(sem_bairro, "true")
    expect(requests_of(sem_bairro).sole.target_unit_id).to be_nil
  end

  it "pedido vivo do mesmo tipo: não duplica; prazo menor, prioridade maior, triagem ligada" do
    unit
    active_protocol!("saude-do-idoso", scheduling: [ rules[1].merge("priority" => "routine", "due_in_days" => 30) ])
    active_protocol!("saude-do-idoso-2", scheduling: [ rules[1] ])
    first = complete(par, "true")
    request = requests_of(par).sole
    second = complete(par, "true", "saude-do-idoso-2")
    expect(requests_of(par).sole.id).to eq(request.id)
    expect(request.reload).to have_attributes(due_on: Time.zone.today + 7, priority: "priority", origin_triage_id: first.id)
    expect(request.request_triages.pluck(:triage_id)).to eq([ second.id ])
    expect(DomainEvent.where(name: "appointment_request.merged_triage").sole.payload)
      .to eq("request_id" => request.id, "triage_id" => second.id)
  end

  it "tipo inexistente na cidade: o pedido nasce com a key (nenhuma necessidade some)" do
    active_protocol!("saude-do-idoso", scheduling: [ rules[1].merge("appointment_type" => "geriatria") ])
    complete(par, "true")
    expect(requests_of(par).sole.appointment_type_key).to eq("geriatria")
  end

  it "conversa sem cidadão (WhatsApp) não gera pedido" do
    protocol = active_protocol!("saude-do-idoso", scheduling: rules)
    conversation = Conversation.create!(channel: "whatsapp", phone: "5541900001111", state: "consented")
    triage = Triage.create!(conversation: conversation, protocol_definition: protocol, protocol_name: protocol.name,
                            status: "completed", answers: { "q1" => "true" })
    outcome = Struct.new(:tier, :score, :priority, :terminal?).new("media", 4, 5, true)
    expect(described_class.call(triage: triage, outcome: outcome)).to be_nil
  end
end
```

A fusão é por cidadão e tipo, não por protocolo: o segundo protocolo (`saude-do-idoso-2`) cai no mesmo pedido.

No `spec/commands/health_units/drain_spec.rb` (existente), acrescente:

```ruby
  it "pedido de triagem movido leva tipo, prioridade, prazo, origem e as triagens ligadas (ADR 0029)" do
    ensure_appointment_types!
    req = triage_request!(Citizen.create!(cpf: "39053344705", phone: "+5541933334444"), unit: unit, priority: "priority",
                          due_on: Time.zone.today + 9)
    extra = completed_web_triage_for(req.citizen)
    AppointmentRequestTriage.create!(request: req, triage: extra, created_at: Time.current)
    HealthUnits::Drain.call(unit: unit, target_unit_id: target.id, reason: "reforma da unidade", by: admin)
    fresh = AppointmentRequest.find_by!(moved_from_request_id: req.id)
    expect(fresh).to have_attributes(kind: "triage", origin_triage_id: req.origin_triage_id, priority: "priority",
                                     due_on: Time.zone.today + 9, appointment_type_key: "consulta_medica",
                                     target_unit_id: target.id)
    expect(fresh.request_triages.pluck(:triage_id)).to eq([ extra.id ])
  end
```

(Use os nomes de `let` que o arquivo já tem para a unidade que esvazia, a de destino e o admin.)

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/triages/schedule_spec.rb spec/commands/health_units/drain_spec.rb`
Expected: FAIL (`uninitialized constant Triages::Schedule`; o drain não copia os campos novos — e o pedido de triagem nem é criado, pela CHECK de uma origem).

- [ ] **Step 4: Implemente**

```ruby
# app/commands/triages/schedule.rb
# Pedido de agendamento gerado pela triagem (ADR 0029 §5.2). Roda dentro da
# transação do CompleteTriage, depois de complete!. Nada se o resultado é
# urgente (Protocols::Urgency) ou se a conversa não tem cidadão (WhatsApp).
# Vale a PRIMEIRA regra de scheduling[] cujo `when` é verdadeiro (perfil +
# respostas + resultado). Unidade = a primeira de referência do bairro copiado
# na triagem; sem ela, fila "sem unidade". Havendo pedido vivo do mesmo tipo
# para o cidadão, funde: prazo menor, prioridade maior, triagem ligada. O
# índice único parcial fecha a corrida entre duas conclusões (savepoint).
module Triages
  module Schedule
    module_function

    def call(triage:, outcome:, on: Time.zone.today)
      return nil if Protocols::Urgency.urgent?(outcome)

      citizen = triage.conversation.citizen
      rules = triage.protocol_definition.definition["scheduling"]
      return nil unless citizen && rules.is_a?(Array) && rules.any?

      context = Protocols::ConditionContext.build(
        answers: triage.answers, profile: citizen.profile_context(on: on),
        outcome: { tier: outcome.tier, score: outcome.score, priority: outcome.priority }
      )
      rule = rules.find { |r| r.is_a?(Hash) && Protocols::Condition.eval(r["when"], context) }
      return nil unless rule

      attrs = { key: rule["appointment_type"].to_s,
                priority: AppointmentRequest::PRIORITIES.include?(rule["priority"]) ? rule["priority"] : "routine",
                due_on: on + rule["due_in_days"].to_i.clamp(1, 365) }
      merge(citizen, triage, attrs) || create(citizen, triage, attrs)
    end

    def merge(citizen, triage, attrs)
      request = AppointmentRequest.live_requests.where(citizen_id: citizen.id, appointment_type_key: attrs[:key])
                                  .order(:created_at).lock.first
      return nil unless request

      priority = [ request.priority, attrs[:priority] ].include?("priority") ? "priority" : "routine"
      request.update!(due_on: [ request.due_on, attrs[:due_on] ].min, priority: priority)
      AppointmentRequestTriage.create!(request: request, triage: triage, created_at: Time.current)
      DomainEvents.publish("appointment_request.merged_triage", request_id: request.id, triage_id: triage.id)
      request
    end

    def create(citizen, triage, attrs)
      unit = Territory::ReferenceUnits.for(triage.neighborhood_id).first
      request = ApplicationRecord.transaction(requires_new: true) do
        AppointmentRequest.create!(kind: "triage", origin_triage: triage, root_triage: triage, citizen: citizen,
                                   target_unit: unit, appointment_type_key: attrs[:key], priority: attrs[:priority],
                                   due_on: attrs[:due_on])
      end
      DomainEvents.publish("appointment_request.created_from_triage", request_id: request.id, triage_id: triage.id)
      request
    rescue ActiveRecord::RecordNotUnique
      merge(citizen, triage, attrs)
    end
  end
end
```

Em `app/commands/complete_triage.rb`, depois de `Triages::Suggest.call(...)`:

```ruby
          Triages::Schedule.call(triage: @triage, outcome: outcome) # ADR 0029: nunca em urgente
```

Em `app/commands/appointment_requests/lifecycle.rb`, o `create!` de `open_for!` ganha:

```ruby
        appointment_type_key: "retorno", priority: "routine",
        due_on: Time.zone.today + AppointmentRequest::DUE_IN_DAYS,
```

Em `app/commands/health_units/drain.rb`, no `create!` de `move`, acrescente os campos copiados

```ruby
        origin_triage_id: request.origin_triage_id, appointment_type_key: request.appointment_type_key,
        priority: request.priority, due_on: request.due_on, reschedule_reason_code: request.reschedule_reason_code,
        reschedule_note: request.reschedule_note, preferred_period: request.preferred_period,
        reschedule_count: request.reschedule_count,
```

e, logo depois do `create!`:

```ruby
      request.request_triages.each do |link|
        AppointmentRequestTriage.create!(request: fresh, triage_id: link.triage_id, created_at: link.created_at)
      end
```

Acrescente ao comentário do topo do `Drain`: `# Módulo 17: o pedido novo leva tipo, prioridade, prazo, origem (atendimento ou triagem) e as triagens ligadas; o horário vai como legacy (o profissional não atende no destino).`

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/triages spec/commands/complete_triage_spec.rb spec/commands/complete_triage_payload_spec.rb spec/commands/health_units spec/commands/attendances spec/requests/appointment_requests_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/commands/triages/schedule.rb app/commands/complete_triage.rb app/commands/appointment_requests/lifecycle.rb app/commands/health_units/drain.rb spec/support/triage_catalog_helpers.rb spec/commands/triages/schedule_spec.rb spec/commands/health_units/drain_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: open appointment requests from completed triages

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 15: Fila da recepção — ordem, marcas, "sem unidade", detalhe e atribuição

**Files:**
- Create: `app/services/scheduling/request_json.rb`
- Create: `app/commands/appointment_requests/assign_unit.rb`
- Modify: `app/controllers/appointment_requests_controller.rb`, `config/routes.rb`
- Modify: `spec/requests/appointment_requests_spec.rb` (a prioridade numérica vira `triage_priority`)
- Test: `spec/requests/request_queue_spec.rb`, `spec/commands/appointment_requests/assign_unit_spec.rb`

**Interfaces:**
- Consumes: `AppointmentPresenter` (Task 12), `Triages::Schedule` (Task 14, para o cenário).
- Produces:
  - `Scheduling::RequestJson.new(catalog:, presenter:, today: Time.zone.today)#call(request, detail: false) -> Hash` (contratos §4.1 + §9): `id, kind, origin, origin_unit_name (nulo em triagem), target_unit_id, created_at, cpf_masked, note, reopened_reason (expired|no_show|null), triage_priority, appointment_type_key, appointment_type_name, priority, due_on, overdue, reschedule_requested, reschedule_reason_code, preferred_period, reschedule_count, needs_reschedule, appointment (§4.4 do horário vivo | null)`; `detail: true` acrescenta `reschedule_note`;
  - `Scheduling::RequestJson.sort(requests, today:) -> Array` (atrasados, `due_on`, prioridade, criação);
  - `AppointmentRequests::AssignUnit.call(request:, unit_id:, by:) -> Result` (`:already_assigned`, `:request_not_open`, `:invalid_unit`); evento `appointment_request.unit_assigned { request_id, unit_id, by_user_id }`;
  - rotas: `GET /attendance/units/:id/requests` (abertos + `needs_reschedule`), `GET /attendance/requests/unassigned` → `{ requests }`, `GET /attendance/requests/:id` → o item + `reschedule_note` (objeto puro), `POST /attendance/requests/:id/assign_unit` `{ unit_id }` → o item (objeto puro); 409 `already_assigned`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/commands/appointment_requests/assign_unit_spec.rb
require "rails_helper"

RSpec.describe AppointmentRequests::AssignUnit do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:reception) { staff_with("recepcao-fila@cidade.gov.br", "citizen_verifier") }
  let(:unit) { create_unit }
  let(:req) { triage_request!(Citizen.create!(cpf: "52998224725", phone: "+5541998765432"), unit: nil) }

  it "atribui uma vez, com evento só de ids; a segunda é already_assigned" do
    expect(described_class.call(request: req, unit_id: unit.id, by: reception)).to be_ok
    expect(req.reload.target_unit_id).to eq(unit.id)
    expect(DomainEvent.where(name: "appointment_request.unit_assigned").sole.payload)
      .to eq("request_id" => req.id, "unit_id" => unit.id, "by_user_id" => reception.id)
    expect(described_class.call(request: req, unit_id: create_unit("UBS Sul").id, by: reception).reason).to eq(:already_assigned)
  end

  it "unidade inativa, inexistente ou id malformado: invalid_unit" do
    expect(described_class.call(request: req, unit_id: create_unit("UBS Fechada", active: false).id, by: reception).reason).to eq(:invalid_unit)
    expect(described_class.call(request: req, unit_id: SecureRandom.uuid, by: reception).reason).to eq(:invalid_unit)
    expect(described_class.call(request: req, unit_id: "x", by: reception).reason).to eq(:invalid_unit)
  end
end
```

```ruby
# spec/requests/request_queue_spec.rb
require "rails_helper"

# Contratos §4.1, §9; spec §5.3: ordem por atraso, prazo e prioridade; marcas;
# fila sem unidade; detalhe com a nota (só ali).
RSpec.describe "Fila de pedidos (agenda)", type: :request do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A; ensure_appointment_types!; sign_in_as(reception) }
  after { Current.reset }

  let(:reception) { staff_with("recepcao-fila@cidade.gov.br", "citizen_verifier") }
  let(:unit) { create_unit }
  let(:cpfs) { %w[52998224725 11144477735 39053344705 87748248800 15350946056] }
  def citizen(i) = Citizen.create!(cpf: cpfs[i], phone: "+55419#{format('%08d', 60_000_000 + i)}")
  def body = JSON.parse(response.body)
  let(:today) { Time.zone.today }

  it "ordena atrasados, depois prazo, depois prioridade; marca e mostra a origem" do
    late = triage_request!(citizen(0), unit: unit, due_on: today - 1)
    soon_routine = triage_request!(citizen(1), unit: unit, due_on: today + 5)
    soon_priority = triage_request!(citizen(2), unit: unit, due_on: today + 5, priority: "priority")
    later = triage_request!(citizen(3), unit: unit, due_on: today + 20, priority: "priority")

    get "/attendance/units/#{unit.id}/requests"
    expect(body["requests"].map { |r| r["id"] }).to eq([ late.id, soon_priority.id, soon_routine.id, later.id ])
    first = body["requests"].first
    expect(first).to include("kind" => "triage", "origin" => "triage", "origin_unit_name" => nil, "overdue" => true,
                             "appointment_type_key" => "consulta_medica", "appointment_type_name" => "Consulta médica",
                             "priority" => "routine", "due_on" => (today - 1).iso8601, "reschedule_requested" => false,
                             "reschedule_count" => 0, "needs_reschedule" => false, "appointment" => nil,
                             "triage_priority" => late.root_triage.priority)
    expect(first).not_to have_key("reschedule_note")
  end

  it "pedido marcado em turno cancelado volta à fila como needs_reschedule, com o horário" do
    link = doctor_link!(unit)
    shift = shift!(link, starts_at: (today + 3).in_time_zone.change(hour: 8))
    req = triage_request!(citizen(0), unit: unit)
    appointment = appointment_row!(req, shift, starts_at: shift.starts_at)
    req.update!(status: "scheduled")
    get "/attendance/units/#{unit.id}/requests"
    expect(body["requests"]).to eq([])

    shift.update!(cancelled_at: Time.current, cancelled_by_user: reception, cancel_reason: "troca de escala")
    get "/attendance/units/#{unit.id}/requests"
    row = body["requests"].sole
    expect(row).to include("id" => req.id, "needs_reschedule" => true)
    expect(row["appointment"]).to include("id" => appointment.id, "shift_cancelled" => true)
  end

  it "fila sem unidade, atribuição (duas vezes = 409) e detalhe com a nota" do
    orphan = triage_request!(citizen(4), unit: nil)
    orphan.update!(reschedule_note: "trabalho de manhã")
    get "/attendance/requests/unassigned"
    expect(body["requests"].map { |r| r["id"] }).to eq([ orphan.id ])
    expect(body["requests"].first).not_to have_key("reschedule_note")

    get "/attendance/requests/#{orphan.id}"
    expect(body).to include("id" => orphan.id, "reschedule_note" => "trabalho de manhã", "target_unit_id" => nil)

    json_post "/attendance/requests/#{orphan.id}/assign_unit", unit_id: unit.id
    expect(response).to have_http_status(:ok)
    expect(body).to include("id" => orphan.id, "target_unit_id" => unit.id)
    json_post "/attendance/requests/#{orphan.id}/assign_unit", unit_id: unit.id
    expect(response).to have_http_status(:conflict)
    expect(body).to eq("error" => "already_assigned")

    get "/attendance/requests/unassigned"
    expect(body["requests"]).to eq([])
  end

  it "pedido de retorno: origem atendimento, tipo retorno, prazo de 30 dias" do
    doctor = staff_with("medica-fila@cidade.gov.br", "health_professional")
    link_professional!(doctor, unit)
    a = in_care!(waiting_attendance(citizen(0), unit: unit, by: reception), by: doctor)
    Attendances::Close.call(attendance: a, outcome: "return", referral_unit_id: nil, referral_note: "reavaliar", by: doctor)
    get "/attendance/units/#{unit.id}/requests"
    expect(body["requests"].sole).to include("origin" => "attendance", "origin_unit_name" => unit.name,
                                             "appointment_type_key" => "retorno", "priority" => "routine",
                                             "due_on" => (today + 30).iso8601)
  end
end
```

Em `spec/requests/appointment_requests_spec.rb`, primeiro exemplo: `"priority" => 5` vira `"triage_priority" => 5, "priority" => "routine"` (contratos §9). Se outro exemplo do arquivo ordenar pela prioridade da triagem, ele passa a esperar a ordem nova (atrasados, prazo, prioridade do pedido, criação); com prazos iguais (todos +30 do mesmo dia), a ordem é a de criação.

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/appointment_requests/assign_unit_spec.rb spec/requests/request_queue_spec.rb`
Expected: FAIL (constantes e rotas inexistentes; a fila antiga quebra em `r.origin_unit.name` no pedido de triagem).

- [ ] **Step 3: Implemente**

```ruby
# app/services/scheduling/request_json.rb
# Item da fila de pedidos (contratos §4.1, §9). `priority` é a do pedido
# (routine|priority); a prioridade numérica da triagem é `triage_priority`.
# A nota livre do cidadão só no detalhe. `reopened_reason` mostra só
# expired/no_show: a remarcação pedida aparece em `reschedule_requested`.
module Scheduling
  class RequestJson
    def self.sort(requests, today:)
      requests.sort_by do |r|
        overdue = r.status == "open" && r.due_on < today
        [ overdue ? 0 : 1, r.due_on, r.priority == "priority" ? 0 : 1, r.created_at ]
      end
    end

    def initialize(catalog: AppointmentTypes.catalog, presenter: AppointmentPresenter.new(show_reason: true),
                   today: Time.zone.today)
      @catalog = catalog
      @presenter = presenter
      @today = today
    end

    def call(request, detail: false)
      live = request.appointments.select { |a| Appointment::LIVE.include?(a.status) }.max_by(&:created_at)
      json = {
        id: request.id, kind: request.kind, origin: request.origin, origin_unit_name: request.origin_unit&.name,
        target_unit_id: request.target_unit_id, created_at: request.created_at.iso8601,
        cpf_masked: request.citizen.cpf_masked, note: request.note,
        reopened_reason: request.reopened_reason == "citizen_reschedule" ? nil : request.reopened_reason,
        triage_priority: request.root_triage.priority, appointment_type_key: request.appointment_type_key,
        appointment_type_name: @catalog.name_for(request.appointment_type_key), priority: request.priority,
        due_on: request.due_on.iso8601, overdue: request.status == "open" && request.due_on < @today,
        reschedule_requested: request.status == "open" && request.reopened_reason == "citizen_reschedule",
        reschedule_reason_code: request.reschedule_reason_code, preferred_period: request.preferred_period,
        reschedule_count: request.reschedule_count, needs_reschedule: live&.shift&.cancelled_at.present? || false,
        appointment: live && @presenter.call(live)
      }
      json[:reschedule_note] = request.reschedule_note if detail
      json
    end
  end
end
```

```ruby
# app/commands/appointment_requests/assign_unit.rb
# Fila "sem unidade" (ADR 0029 §5.3): o pedido da triagem sem unidade de
# referência recebe uma unidade ativa UMA vez (o trigger aceita só NULL →
# unidade). FOR SHARE na unidade, como no módulo 09.
module AppointmentRequests
  module AssignUnit
    UUID = /\A\h{8}-(\h{4}-){3}\h{12}\z/

    module_function

    def call(request:, unit_id:, by:)
      return Result.fail(:invalid_unit) unless unit_id.to_s.match?(UUID)

      ApplicationRecord.transaction do
        request.lock!
        next Result.fail(:already_assigned) unless request.target_unit_id.nil?
        next Result.fail(:request_not_open) unless request.status == "open"

        unit = HealthUnit.lock_active!(unit_id.to_s)
        request.update!(target_unit: unit)
        DomainEvents.publish("appointment_request.unit_assigned", request_id: request.id, unit_id: unit.id,
                                                                  by_user_id: by.id)
        Result.ok(request: request)
      end
    rescue HealthUnit::Inactive
      Result.fail(:invalid_unit)
    end
  end
end
```

Em `app/controllers/appointment_requests_controller.rb`, `index` e as ações novas (o `before_action :require_verifier` cobre todas):

```ruby
  QUEUE_INCLUDES = [ :citizen, :origin_unit, :root_triage,
                     { appointments: [ :citizen, :professional, { shift: %i[professional_link schedule_template] } ] } ].freeze

  def index
    unit = HealthUnit.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless unit

    base = AppointmentRequest.where(target_unit: unit)
    cancelled = Appointment.live.joins(:shift).where.not(professional_shifts: { cancelled_at: nil }).select(:request_id)
    render_queue(base.where(status: "open").or(base.where(status: "scheduled", id: cancelled)))
  end

  def unassigned
    render_queue(AppointmentRequest.where(target_unit_id: nil, status: "open"))
  end

  def show
    request = AppointmentRequest.includes(*QUEUE_INCLUDES).find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless request

    render json: request_json_builder.call(request, detail: true)
  end

  def assign_unit
    request = AppointmentRequest.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless request

    result = AppointmentRequests::AssignUnit.call(request: request, unit_id: request.request_parameters["unit_id"],
                                                  by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: request_json_builder.call(AppointmentRequest.includes(*QUEUE_INCLUDES).find(request.id))
  end
```

e, em `private` (o `request_json` antigo sai):

```ruby
  def render_queue(scope)
    requests = Scheduling::RequestJson.sort(scope.includes(*QUEUE_INCLUDES).to_a, today: Time.zone.today)
    render json: { requests: requests.map { |r| request_json_builder.call(r) } }
  end

  def request_json_builder = @request_json_builder ||= Scheduling::RequestJson.new(presenter: presenter)
```

`ERROR_STATUS` já tem `already_assigned: :conflict` (Task 12); `request_not_open` e `invalid_unit` também.

Em `config/routes.rb`, no `scope "/attendance"`, junto das rotas de pedidos, **`requests/unassigned` antes de `requests/:id`**:

```ruby
    get  "requests/unassigned",       to: "appointment_requests#unassigned"
    get  "requests/:id",              to: "appointment_requests#show"
    post "requests/:id/assign_unit",  to: "appointment_requests#assign_unit"
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/appointment_requests spec/requests/request_queue_spec.rb spec/requests/appointment_requests_spec.rb spec/requests/attendance_contract_spec.rb spec/requests/booking_spec.rb`
Expected: PASS. Se `attendance_contract_spec.rb` fixa a forma antiga do item da fila (chave `priority` numérica), ajuste-a como no `appointment_requests_spec.rb` e acrescente o arquivo ao commit.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/services/scheduling/request_json.rb app/commands/appointment_requests/assign_unit.rb app/controllers/appointment_requests_controller.rb config/routes.rb spec/commands/appointment_requests/assign_unit_spec.rb spec/requests/request_queue_spec.rb spec/requests/appointment_requests_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: order the request queue by due date and add the unassigned queue

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 16: LGPD e o resultado da triagem — revogação fecha o pedido, exclusão apaga avisos, `scheduling_request`

**Files:**
- Create: `app/commands/appointment_requests/close_revoked.rb`
- Modify: `app/commands/revoke_consent.rb`, `app/commands/citizens/erase.rb`
- Create: `app/services/scheduling/triage_request.rb`
- Modify: `app/controllers/citizen_api/triages_controller.rb`
- Test: `spec/commands/appointment_requests/close_revoked_spec.rb`, `spec/requests/citizen_api/triage_scheduling_request_spec.rb`, `spec/commands/citizens/erase_spec.rb` (acréscimo)

**Interfaces:**
- Consumes: `Triages::Schedule` (Task 14), `AppointmentNotice` (Task 2).
- Produces:
  - `AppointmentRequests::CloseRevoked.call(conversation:) -> Array<AppointmentRequest>` — dentro da transação de `RevokeConsent`: fecha como `consent_revoked` cada pedido de triagem `open` ligado a triagens da conversa, salvo se outra triagem ligada a ele está em conversa com consentimento ativo; usa `Lifecycle.close!` (evento `appointment_request.closed { appointment_request_id, closed_reason }`);
  - `Citizens::Erase.erase_pair` apaga `AppointmentNotice` do par;
  - `Scheduling::TriageRequest.for(triage, catalog: AppointmentTypes.catalog) -> { unit_name, due_on, appointment_type_name } | nil` (pedido vivo, não movido, nascido da triagem ou ligado a ela);
  - `GET /citizen/triages/:id` ganha `scheduling_request` (só na triagem do próprio par; outra → `null`).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/commands/appointment_requests/close_revoked_spec.rb
require "rails_helper"

# ADR 0029 §5.2 + ADR 0026: revogar fecha o pedido de triagem aberto sem
# horário; pedido já marcado fica (o cidadão cancela se quiser); pedido fundido
# com triagem de outra conversa ainda consentida também fica.
RSpec.describe AppointmentRequests::CloseRevoked do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset; Rails.cache.clear }

  let(:rule) { [ { "when" => { "gte" => ["outcome.score", 1] }, "appointment_type" => "consulta_medica", "priority" => "routine", "due_in_days" => 30 } ] }
  let(:par) { profiled_citizen!(age: 70) }

  def complete(protocol = "saude-do-idoso")
    started = start_for!(par, protocol).payload
    Citizens::SubmitAnswer.call(conversation: started[:conversation], answer: "true", idempotency_key: SecureRandom.uuid)
    started[:triage].reload
  end

  it "revogar fecha o pedido aberto da triagem como consent_revoked, com evento" do
    active_protocol!("saude-do-idoso", scheduling: rule)
    triage = complete
    RevokeConsent.call(conversation: triage.conversation, origin: "web")
    request = AppointmentRequest.find_by!(origin_triage_id: triage.id)
    expect(request).to have_attributes(status: "closed", closed_reason: "consent_revoked")
    expect(DomainEvent.where(name: "appointment_request.closed").last.payload)
      .to eq("appointment_request_id" => request.id, "closed_reason" => "consent_revoked")
  end

  it "pedido fundido com triagem de outra conversa ainda consentida fica aberto" do
    active_protocol!("saude-do-idoso", scheduling: rule)
    active_protocol!("saude-do-idoso-2", scheduling: rule)
    first = complete
    complete("saude-do-idoso-2")
    RevokeConsent.call(conversation: first.conversation, origin: "web")
    expect(AppointmentRequest.find_by!(origin_triage_id: first.id).status).to eq("open")
  end

  it "pedido já marcado não fecha" do
    active_protocol!("saude-do-idoso", scheduling: rule)
    triage = complete
    request = AppointmentRequest.find_by!(origin_triage_id: triage.id)
    request.update!(status: "scheduled")
    RevokeConsent.call(conversation: triage.conversation, origin: "web")
    expect(request.reload.status).to eq("scheduled")
  end
end
```

```ruby
# spec/requests/citizen_api/triage_scheduling_request_spec.rb
require "rails_helper"

# Contratos §5: o resultado da triagem diz a unidade, o tipo e o prazo do
# pedido gerado (o texto é do wpda). Sem unidade, unit_name nulo.
RSpec.describe "GET /citizen/triages/:id — scheduling_request", type: :request do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset; Rails.cache.clear }

  let(:rule) { [ { "when" => { "gte" => ["outcome.score", 1] }, "appointment_type" => "consulta_medica", "priority" => "routine", "due_in_days" => 30 } ] }
  def body = JSON.parse(response.body)

  it "com e sem unidade; sem regra, null" do
    active_protocol!("saude-do-idoso", scheduling: rule)
    sign_in_citizen("+5541998765432")
    started = start_citizen_triage(protocol_name: "saude-do-idoso")
    json_post "/citizen/conversations/#{started.dig('conversation', 'id')}/answers", answer: "true", idempotency_key: SecureRandom.uuid
    triage_id = Triage.order(:created_at).last.id

    get "/citizen/triages/#{triage_id}"
    expect(body["scheduling_request"]).to eq("unit_name" => nil, "due_on" => (Time.zone.today + 30).iso8601,
                                             "appointment_type_name" => "Consulta médica")

    unit = create_unit("UBS Batel")
    AppointmentRequest.find_by!(origin_triage_id: triage_id).update!(target_unit: unit)
    get "/citizen/triages/#{triage_id}"
    expect(body["scheduling_request"]["unit_name"]).to eq("UBS Batel")
  end
end
```

(Se o corpo de `start_citizen_triage` ou a rota de resposta tiverem outra forma no `origin/main`, siga `spec/requests/citizen_api/triage_suggestions_spec.rb`, que faz exatamente este caminho no módulo 15.)

No `spec/commands/citizens/erase_spec.rb` (existente), num exemplo que confirma a exclusão de um par sem atendimento, crie antes um aviso de lembrete do par e espere que ele suma:

```ruby
    unit = create_unit("UBS Aviso")
    shift = shift!(doctor_link!(unit), starts_at: 3.days.from_now.change(hour: 8))
    notice = AppointmentNotice.create!(appointment: appointment_row!(triage_request!(citizen, unit: unit), shift, starts_at: shift.starts_at),
                                       citizen: citizen, created_at: Time.current)
    # ... confirmação da exclusão, como o exemplo já faz ...
    expect(AppointmentNotice.where(id: notice.id)).to be_empty
```

(`citizen` é o par que o exemplo exclui; ajuste o nome ao do arquivo.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/appointment_requests/close_revoked_spec.rb spec/requests/citizen_api/triage_scheduling_request_spec.rb spec/commands/citizens/erase_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implemente**

```ruby
# app/commands/appointment_requests/close_revoked.rb
# Revogação (ADR 0029 §5.2; ADR 0026): o pedido de triagem ainda aberto (sem
# horário) ligado a triagens da conversa revogada fecha como consent_revoked —
# a menos que outra triagem ligada a ele esteja numa conversa com
# consentimento ativo. Chamado DENTRO da transação de RevokeConsent, depois de
# revogar.
module AppointmentRequests
  module CloseRevoked
    module_function

    def call(conversation:)
      triage_ids = conversation.triages.select(:id)
      linked = AppointmentRequestTriage.where(triage_id: triage_ids).select(:request_id)
      base = AppointmentRequest.where(kind: "triage", status: "open")
      requests = base.where(origin_triage_id: triage_ids).or(base.where(id: linked)).order(:id).lock.to_a
      requests.reject { |request| still_consented?(request, conversation) }.each do |request|
        Lifecycle.close!(request, reason: "consent_revoked")
      end
    end

    def still_consented?(request, conversation)
      triage_ids = [ request.origin_triage_id, *request.request_triages.pluck(:triage_id) ]
      Consent.where(revoked_at: nil, conversation_id: Triage.where(id: triage_ids).select(:conversation_id))
             .where.not(conversation_id: conversation.id).exists?
    end
  end
end
```

Em `app/commands/revoke_consent.rb`, depois do `TriageSuggestion...delete_all`:

```ruby
      # ADR 0029 §5.2: o pedido de agendamento ainda sem horário fecha.
      AppointmentRequests::CloseRevoked.call(conversation: @conversation)
```

Em `app/commands/citizens/erase.rb`, em `erase_pair`, junto dos outros `delete_all`:

```ruby
      # ADR 0029: os avisos de lembrete do par (o trigger deixa o DELETE passar).
      AppointmentNotice.where(citizen_id: citizen.id).delete_all
```

```ruby
# app/services/scheduling/triage_request.rb
# O pedido que a triagem gerou (ou no qual foi fundida), para o resultado do
# cidadão (contratos §5). Só pedido vivo; o movido de unidade (api#29) é
# seguido pela cópia, que leva origin_triage_id e as ligações.
module Scheduling
  module TriageRequest
    module_function

    def for(triage, catalog: AppointmentTypes.catalog)
      linked = AppointmentRequestTriage.where(triage_id: triage.id).select(:request_id)
      base = AppointmentRequest.where(status: %w[open scheduled])
      request = base.where(origin_triage_id: triage.id).or(base.where(id: linked))
                    .includes(:target_unit).order(created_at: :desc, id: :desc).first
      return nil unless request

      { unit_name: request.target_unit&.name, due_on: request.due_on.iso8601,
        appointment_type_name: catalog.name_for(request.appointment_type_key) }
    end
  end
end
```

Em `app/controllers/citizen_api/triages_controller.rb`, `show` passa a mesclar:

```ruby
      scheduling = own_triage?(triage) ? Scheduling::TriageRequest.for(triage) : nil
      render json: summary(triage).merge(reference_units: Territory::ReferenceUnits.as_json_list(units),
                                         suggestions: suggestions, scheduling_request: scheduling)
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/appointment_requests spec/requests/citizen_api spec/commands/citizens/erase_spec.rb spec/commands/revoke_consent_spec.rb spec/jobs/anonymize_revoked_triage_job_spec.rb`
Expected: PASS. Se algum exemplo de `GET /citizen/triages/:id` compara o corpo inteiro com `eq`, acrescente `"scheduling_request" => nil`.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/commands/appointment_requests/close_revoked.rb app/commands/revoke_consent.rb app/commands/citizens/erase.rb app/services/scheduling/triage_request.rb app/controllers/citizen_api/triages_controller.rb spec/commands/appointment_requests/close_revoked_spec.rb spec/requests/citizen_api/triage_scheduling_request_spec.rb spec/commands/citizens/erase_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: close triage requests on consent revocation and show them in the result

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 6 — Cidadão (F-17.7, F-17.8)

### Task 17: "Não posso nesse horário" e "Meus horários" com os campos novos

**Files:**
- Create: `app/commands/appointments/request_reschedule.rb`
- Create: `app/services/scheduling/unit_address.rb`
- Modify: `app/controllers/citizen_api/appointments_controller.rb`, `config/routes.rb`
- Test: `spec/commands/appointments/request_reschedule_spec.rb`, `spec/requests/citizen_api/reschedule_request_spec.rb`

**Interfaces:**
- Consumes: `AppointmentRequest::RESCHEDULE_REASONS`, `::PERIODS`, `Appointment::RESCHEDULE_CANCEL_REASON` (Task 2).
- Produces:
  - `Appointments::RequestReschedule.call(appointment:, reason_code:, note:, preferred_period:, now: Time.current) -> Result` — motivos `:invalid_reason_code`, `:invalid_period`, `:note_too_long`, `:not_reschedulable`; o horário vira `cancelled_by_citizen` com a frase fixa e `reschedule_requested = true`; o pedido volta a `open` com `reopened_reason = "citizen_reschedule"`, motivo, nota, período e `reschedule_count + 1`, **prazo mantido**; evento `appointment.reschedule_requested { appointment_id, request_id }`;
  - `Scheduling::UnitAddress.call(unit) -> { street, number, complement, zip } | nil` (a forma de `reference_units`, contratos §8);
  - `GET /citizen/appointments`: `request` ganha `kind`, `target_unit_name` (nulo sem unidade), `appointment_type_name`, `due_on`; `appointment` ganha `appointment_type_name`, `professional_name`, `unit: { name, address }`, `ends_at`, `can_request_reschedule` (contratos §5, §8);
  - `POST /citizen/appointments/:id/reschedule_request` `{ reason_code, note?, preferred_period }` → 200 `{ appointment }` (como `confirm`/`cancel`, contratos §8); 409 `not_reschedulable`; 422 `invalid_reason_code`, `invalid_period`, `note_too_long`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/commands/appointments/request_reschedule_spec.rb
require "rails_helper"

# ADR 0029 §6 (spec §9 "Cidadão: reschedule"): motivo, período, contagem,
# prazo mantido; recusas nas bordas; nota nunca em evento.
RSpec.describe Appointments::RequestReschedule do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:shift) { shift!(doctor_link!(unit), starts_at: 4.days.from_now.change(hour: 8)) }
  let(:request) { triage_request!(Citizen.create!(cpf: "52998224725", phone: "+5541998765432"), unit: unit, due_on: Time.zone.today + 12) }
  let(:appointment) do
    appointment_row!(request, shift, starts_at: shift.starts_at).tap { request.update!(status: "scheduled") }
  end

  def ask(appt = appointment, reason: "work", note: "entro às 7h no serviço", period: "afternoon")
    described_class.call(appointment: appt, reason_code: reason, note: note, preferred_period: period)
  end

  it "cancela com a frase fixa, devolve o pedido marcado à fila e mantém o prazo" do
    expect(ask).to be_ok
    expect(appointment.reload).to have_attributes(status: "cancelled_by_citizen", reschedule_requested: true,
                                                  cancel_reason: "Remarcação pedida pelo cidadão")
    expect(request.reload).to have_attributes(status: "open", reopened_reason: "citizen_reschedule",
                                              reschedule_reason_code: "work", reschedule_note: "entro às 7h no serviço",
                                              preferred_period: "afternoon", reschedule_count: 1,
                                              due_on: Time.zone.today + 12)
    payload = DomainEvent.where(name: "appointment.reschedule_requested").sole.payload
    expect(payload).to eq("appointment_id" => appointment.id, "request_id" => request.id)
  end

  it "conta cada pedido de remarcação" do
    ask
    second = appointment_row!(request.reload, shift, starts_at: shift.starts_at + 40.minutes)
    request.update!(status: "scheduled")
    described_class.call(appointment: second, reason_code: "transport", note: nil, preferred_period: "any")
    expect(request.reload).to have_attributes(reschedule_count: 2, reschedule_reason_code: "transport", reschedule_note: nil)
  end

  {
    { reason: "ferias" } => :invalid_reason_code, { period: "noite" } => :invalid_period,
    { note: "x" * 201 } => :note_too_long
  }.each do |args, reason|
    it("recusa #{args.keys.first} inválido com #{reason}") { expect(ask(**args).reason).to eq(reason) }
  end

  it "depois do início, já cancelado, ou duas vezes: not_reschedulable" do
    ask
    expect(ask(appointment.reload).reason).to eq(:not_reschedulable)
    other = appointment_row!(triage_request!(Citizen.create!(cpf: "11144477735", phone: "+5541911112222"), unit: unit),
                             shift, starts_at: shift.starts_at + 20.minutes)
    travel_to(other.scheduled_at + 1.minute) { expect(ask(other).reason).to eq(:not_reschedulable) }
  end
end
```

```ruby
# spec/requests/citizen_api/reschedule_request_spec.rb
require "rails_helper"

# Contratos §5, §8.
RSpec.describe "Meus horários (agenda)", type: :request do
  before { Current.city = TEST_CITY_A; Rails.cache.clear; ensure_appointment_types! }
  after { Current.reset }

  let(:unit) { create_unit.tap { |u| u.update!(address_street: "Rua XV de Novembro", address_number: "100", address_zip: "80020310") } }
  let(:link) { doctor_link!(unit) }
  let(:shift) { shift!(link, starts_at: 4.days.from_now.change(hour: 8)) }
  let!(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  def body = JSON.parse(response.body)

  it "lista pedido e horário com os campos novos; pede outro horário; 409 na segunda vez" do
    request = triage_request!(citizen, unit: unit)
    appointment = appointment_row!(request, shift, starts_at: shift.starts_at)
    request.update!(status: "scheduled")
    orphan = triage_request!(citizen, unit: nil, type_key: "retorno")
    sign_in_citizen("+5541998765432")

    get "/citizen/appointments", params: { citizen_id: citizen.id }
    items = body["appointments"].index_by { |i| i.dig("request", "id") }
    expect(items[orphan.id]["request"]).to include("kind" => "triage", "target_unit_name" => nil,
                                                   "appointment_type_name" => "Retorno",
                                                   "due_on" => orphan.due_on.iso8601)
    expect(items[orphan.id]["appointment"]).to be_nil
    expect(items[request.id]["appointment"]).to include(
      "id" => appointment.id, "appointment_type_name" => "Consulta médica",
      "professional_name" => link.professional.professional_name, "ends_at" => appointment.ends_at.iso8601,
      "can_request_reschedule" => true,
      "unit" => { "name" => unit.name, "address" => { "street" => "Rua XV de Novembro", "number" => "100",
                                                      "complement" => nil, "zip" => "80020310" } }
    )

    json_post "/citizen/appointments/#{appointment.id}/reschedule_request",
              reason_code: "work", note: "entro às 7h", preferred_period: "afternoon"
    expect(response).to have_http_status(:ok)
    expect(body["appointment"]).to include("id" => appointment.id, "status" => "cancelled_by_citizen",
                                           "can_request_reschedule" => false)

    json_post "/citizen/appointments/#{appointment.id}/reschedule_request", reason_code: "work", preferred_period: "any"
    expect(response).to have_http_status(:conflict)
    expect(body).to eq("error" => "not_reschedulable")
  end

  it "horário de outro celular: 404; motivo inválido: 422" do
    request = triage_request!(citizen, unit: unit)
    appointment = appointment_row!(request, shift, starts_at: shift.starts_at)
    sign_in_citizen("+5541911112222")
    json_post "/citizen/appointments/#{appointment.id}/reschedule_request", reason_code: "work", preferred_period: "any"
    expect(response).to have_http_status(:not_found)

    sign_in_citizen("+5541998765432")
    json_post "/citizen/appointments/#{appointment.id}/reschedule_request", reason_code: "ferias", preferred_period: "any"
    expect(response).to have_http_status(:unprocessable_entity)
    expect(body).to eq("error" => "invalid_reason_code")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/appointments/request_reschedule_spec.rb spec/requests/citizen_api/reschedule_request_spec.rb`
Expected: FAIL (constante e rota inexistentes; a lista quebra no pedido sem unidade).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/appointments/request_reschedule.rb
# "Não posso nesse horário" (ADR 0029 §6): antes do início, o cidadão cancela o
# horário marcado ou confirmado com um motivo da lista fixa e um período
# preferido; o pedido volta à fila marcado (reopened_reason citizen_reschedule),
# com a contagem, e o PRAZO NÃO MUDA. A nota (≤ 200) fica só no pedido: nunca em
# evento nem log. Trava horário → pedido (a ordem de CancelByCitizen).
module Appointments
  module RequestReschedule
    MAX_NOTE = 200

    module_function

    def call(appointment:, reason_code:, note:, preferred_period:, now: Time.current)
      return Result.fail(:invalid_reason_code) unless AppointmentRequest::RESCHEDULE_REASONS.include?(reason_code)
      return Result.fail(:invalid_period) unless AppointmentRequest::PERIODS.include?(preferred_period)

      note = note.is_a?(String) ? note.strip.presence : nil
      return Result.fail(:note_too_long) if note && note.length > MAX_NOTE

      ApplicationRecord.transaction do
        appointment.lock!
        request = appointment.request
        request.lock!
        next Result.fail(:not_reschedulable) unless Appointment::LIVE.include?(appointment.status) && now < appointment.scheduled_at

        appointment.update!(status: "cancelled_by_citizen", cancel_reason: Appointment::RESCHEDULE_CANCEL_REASON,
                            reschedule_requested: true, ended_at: now)
        request.update!(status: "open", reopened_reason: "citizen_reschedule", reschedule_reason_code: reason_code,
                        reschedule_note: note, preferred_period: preferred_period,
                        reschedule_count: request.reschedule_count + 1)
        DomainEvents.publish("appointment.reschedule_requested", appointment_id: appointment.id, request_id: request.id)
        Result.ok(appointment: appointment)
      end
    end
  end
end
```

```ruby
# app/services/scheduling/unit_address.rb
# Endereço da unidade na forma de reference_units (contratos §8).
module Scheduling
  module UnitAddress
    module_function

    def call(unit)
      return nil unless unit

      { street: unit.address_street, number: unit.address_number, complement: unit.address_complement,
        zip: unit.address_zip }
    end
  end
end
```

Em `app/controllers/citizen_api/appointments_controller.rb`:

- `ERROR_STATUS` ganha `not_reschedulable: :conflict, invalid_reason_code: :unprocessable_entity, invalid_period: :unprocessable_entity, note_too_long: :unprocessable_entity`;
- o `rate_limit` de escrita passa a `only: %i[confirm cancel reschedule_request]`;
- `index` passa a `includes(:target_unit, moved_from_request: :target_unit, appointments: %i[health_unit professional])`;
- ação nova:

```ruby
    def reschedule_request
      with_appointment do |a|
        Appointments::RequestReschedule.call(appointment: a, reason_code: params[:reason_code].to_s,
                                             note: params[:note], preferred_period: params[:preferred_period].to_s)
      end
    end
```

- `item_json` e `appointment_json` passam a:

```ruby
    def item_json(request)
      latest = request.latest_appointment
      {
        request: { id: request.id, kind: request.kind, target_unit_name: request.target_unit&.name,
                   status: request.status, closed_reason: request.closed_reason,
                   reopened_reason: request.reopened_reason == "citizen_reschedule" ? nil : request.reopened_reason,
                   moved_from_unit_name: request.moved_from_request&.target_unit&.name,
                   appointment_type_name: catalog.name_for(request.appointment_type_key),
                   due_on: request.due_on.iso8601 },
        appointment: latest && appointment_json(latest)
      }
    end

    def appointment_json(a)
      legacy = a.booking_kind == "legacy"
      { id: a.id, scheduled_at: a.scheduled_at.iso8601, status: a.status,
        confirmation_deadline_at: a.confirmation_deadline_at&.iso8601,
        check_in_available: Attendances::AppointmentCheckInEligibility.check(a) == :ok,
        ends_at: legacy ? nil : a.ends_at&.iso8601,
        appointment_type_name: legacy ? nil : catalog.name_for(a.appointment_type_key),
        professional_name: legacy ? nil : a.professional&.professional_name,
        unit: { name: a.health_unit.name, address: Scheduling::UnitAddress.call(a.health_unit) },
        can_request_reschedule: Appointment::LIVE.include?(a.status) && Time.current < a.scheduled_at }
    end

    def catalog = @catalog ||= Scheduling::AppointmentTypes.catalog
```

Em `config/routes.rb`, no escopo do cidadão, junto das ações de horário: `post "appointments/:id/reschedule_request", to: "appointments#reschedule_request"`.

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/commands/appointments spec/requests/citizen_api/reschedule_request_spec.rb spec/requests/citizen_api/appointments_spec.rb spec/requests/citizen_api/isolation_spec.rb`
Expected: PASS (os exemplos antigos usam `include`, que aceita as chaves novas).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/commands/appointments/request_reschedule.rb app/services/scheduling/unit_address.rb app/controllers/citizen_api/appointments_controller.rb config/routes.rb spec/commands/appointments/request_reschedule_spec.rb spec/requests/citizen_api/reschedule_request_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: let citizens ask for another time without choosing it

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 18: Lembrete da véspera — `Appointments::Remind` e `Appointments::RemindJob`

**Files:**
- Create: `app/commands/appointments/remind.rb`
- Create: `app/jobs/appointments/remind_job.rb`
- Modify: `config/recurring.yml`
- Test: `spec/jobs/appointments/remind_job_spec.rb`

**Interfaces:**
- Consumes: `AppointmentNotice` (Task 2), `Campaigns::SmsSetting`, `SmsGateway`, `Campaigns::SmsText.link`, `Campaigns::SmsBatchJob::WINDOW_HOURS`, `CitizenContactPreference` (módulo 12, api#39).
- Produces:
  - `Appointments::Remind.call(appointment:, now: Time.current) -> Result` (`payload[:sms]` booleano, ou `skipped: :not_due`) — sob lock do horário: cria o aviso, tenta o SMS, grava `reminded_at`, publica `appointment.reminded { appointment_id, sms }`; `Appointments::Remind::TEXT` (o texto fixo, com `%{city}` e `%{link}`);
  - `Appointments::RemindJob` (`prepend EachCityJob`, fila `housekeeping`, a cada 15 minutos): age só das 17h às 20h no fuso da cidade, sobre horários `confirmed` de amanhã (fuso da cidade) sem `reminded_at`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/jobs/appointments/remind_job_spec.rb
require "rails_helper"

# ADR 0029 §6 (spec §9 "lembrete"): véspera às 17h no fuso da cidade, só
# confirmados, idempotente; aviso sempre; SMS só com a chave da cidade, o
# opt-in e sem o opt-out de lembretes; texto fixo, sem identificador.
RSpec.describe Appointments::RemindJob do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A; SmsGateway::Test.reset!; ensure_appointment_types! }
  after { SmsGateway::Test.reset!; Current.reset }

  let(:zone_name) { "America/Sao_Paulo" }
  let!(:city_record) { create(:city, slug: TEST_CITY_A.slug, database_url: TEST_CITY_A.database_url, time_zone: zone_name) }
  let(:zone) { ActiveSupport::TimeZone[zone_name] }
  let(:eve) { Time.zone.today + 3 }
  let(:unit) { create_unit }
  let(:shift) { shift!(doctor_link!(unit), starts_at: zone.local((eve + 1).year, (eve + 1).month, (eve + 1).day, 8)) }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }

  def confirmed!(at = shift.starts_at, status: "confirmed", who: citizen)
    appointment_row!(triage_request!(who, unit: unit), shift, starts_at: at, status: status)
  end

  def at(hour, min = 0) = zone.local(eve.year, eve.month, eve.day, hour, min)
  def run_at(time) = travel_to(time) { described_class.perform_now }
  def sms_on! = (CityProfile.current || CityProfile.create!(name: "Curitiba")).update!(campaigns_sms_enabled: true)
  def opt_in!(muted: false) = CitizenContactPreference.create!(citizen_id: citizen.id, sms_opt_in: true, appointment_reminders_muted: muted)

  it "às 17h da véspera: aviso, SMS de texto fixo, reminded_at e evento; antes das 17h e na segunda rodada, nada" do
    sms_on!
    opt_in!
    appointment = confirmed!
    run_at(at(16, 59))
    expect(AppointmentNotice.count).to eq(0)

    run_at(at(17, 0))
    run_at(at(17, 15))
    expect(AppointmentNotice.where(appointment_id: appointment.id, citizen_id: citizen.id).count).to eq(1)
    expect(appointment.reload.reminded_at).to be_present
    expect(SmsGateway::Test.deliveries.size).to eq(1)
    body = SmsGateway::Test.deliveries.first[:body]
    expect(body).to eq("Secretaria de Saúde de Curitiba: você tem um compromisso de saúde amanhã. Veja em " \
                       "#{Campaigns::SmsText.link(TEST_CITY_A)}")
    expect(body).not_to include(unit.name, appointment.id)
    expect(DomainEvent.where(name: "appointment.reminded").sole.payload).to eq("appointment_id" => appointment.id, "sms" => true)
  end

  it "sem opt-in, com opt-out de lembretes ou sem a chave: só o aviso" do
    appointment = confirmed!
    run_at(at(17, 0))
    expect(SmsGateway::Test.deliveries).to be_empty
    expect(AppointmentNotice.where(appointment_id: appointment.id).count).to eq(1)
    expect(DomainEvent.where(name: "appointment.reminded").sole.payload["sms"]).to be(false)

    other = Citizen.create!(cpf: "11144477735", phone: "+5541911112222")
    CitizenContactPreference.create!(citizen_id: other.id, sms_opt_in: true, appointment_reminders_muted: true)
    sms_on!
    confirmed!(shift.starts_at + 20.minutes, who: other)
    run_at(at(17, 30))
    expect(SmsGateway::Test.deliveries).to be_empty
  end

  it "só confirmados de amanhã; depois das 20h, nada" do
    sms_on!
    opt_in!
    unconfirmed = confirmed!(status: "scheduled")
    run_at(at(20, 0))
    run_at(at(17, 0))
    expect(unconfirmed.reload.reminded_at).to be_nil
    expect(AppointmentNotice.count).to eq(0)
  end

  context "cidade em Manaus (UTC−4)" do
    let(:zone_name) { "America/Manaus" }

    it "as 17h são de Manaus, não de São Paulo (Review Focus 1)" do
      appointment = confirmed!
      run_at(ActiveSupport::TimeZone["America/Sao_Paulo"].local(eve.year, eve.month, eve.day, 17, 30)) # 16h30 em Manaus
      expect(appointment.reload.reminded_at).to be_nil
      run_at(at(17, 0))
      expect(appointment.reload.reminded_at).to be_present
    end
  end
end
```

(Se a factory `:city` não aceitar `time_zone`, crie o `City` como `spec/support/city_request_auth.rb#use_test_city_host!` faz, passando `time_zone:`; o trigger `cities_time_zone_immutable` só impede trocar depois.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/jobs/appointments/remind_job_spec.rb`
Expected: FAIL com `uninitialized constant Appointments::RemindJob`.

- [ ] **Step 3: Implemente**

```ruby
# app/commands/appointments/remind.rb
# Lembrete da véspera do horário confirmado (ADR 0029 §6). Sob lock do
# horário: um aviso na caixa do cidadão (sempre), um SMS de texto fixo (só com
# a chave de SMS da cidade, provedor, opt-in e sem o opt-out de lembretes,
# dentro da janela 8h–20h) e reminded_at (idempotência). Falha do provedor não
# repete; o log leva só a classe do erro.
module Appointments
  module Remind
    TEXT = "Secretaria de Saúde de %{city}: você tem um compromisso de saúde amanhã. Veja em %{link}".freeze

    module_function

    def call(appointment:, now: Time.current)
      ApplicationRecord.transaction do
        appointment.lock!
        next Result.ok(skipped: :not_due) unless appointment.status == "confirmed" && appointment.reminded_at.nil?

        unless AppointmentNotice.exists?(appointment_id: appointment.id)
          AppointmentNotice.create!(appointment: appointment, citizen_id: appointment.citizen_id, created_at: now)
        end
        sms = deliver?(appointment, now)
        appointment.update!(reminded_at: now)
        DomainEvents.publish("appointment.reminded", appointment_id: appointment.id, sms: sms)
        Result.ok(sms: sms)
      end
    end

    def deliver?(appointment, now)
      return false unless Campaigns::SmsSetting.enabled? && SmsGateway.configured?
      return false unless Campaigns::SmsBatchJob::WINDOW_HOURS.cover?(now.hour)

      preference = CitizenContactPreference.find_by(citizen_id: appointment.citizen_id)
      return false unless preference&.sms_opt_in && !preference.appointment_reminders_muted

      SmsGateway.deliver(phone: appointment.citizen.phone, body: body)
      true
    rescue StandardError => e
      Rails.logger.warn("[appointment.reminded] SMS não saiu: #{e.class}")
      false
    end

    def body
      name = CityProfile.current&.name.presence || Current.city.name
      format(TEXT, city: name, link: Campaigns::SmsText.link(Current.city))
    end
  end
end
```

```ruby
# app/jobs/appointments/remind_job.rb
# Lembrete da véspera (ADR 0029 §6), cidade por cidade. Roda a cada 15
# minutos e só age das 17h às 20h no fuso da cidade (CityConnection.with usa o
# fuso dela); reminded_at faz a próxima rodada pular quem já recebeu.
module Appointments
  class RemindJob < ApplicationJob
    prepend EachCityJob
    queue_as :housekeeping

    FROM_HOUR = 17

    def perform
      now = Time.current
      return unless now.hour >= FROM_HOUR && Campaigns::SmsBatchJob::WINDOW_HOURS.cover?(now.hour)

      tomorrow = (now.to_date + 1).in_time_zone.all_day
      Appointment.where(status: "confirmed", reminded_at: nil, scheduled_at: tomorrow).find_each do |appointment|
        Remind.call(appointment: appointment, now: now)
      end
    end
  end
end
```

Em `config/recurring.yml`, no bloco `default`, depois de `mark_no_show_appointments`:

```yaml
  remind_appointments:
    class: Appointments::RemindJob
    queue: housekeeping
    schedule: "every 15 minutes"
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/jobs/appointments/remind_job_spec.rb spec/jobs/send_confirmation_reminders_job_spec.rb spec/architecture`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/commands/appointments/remind.rb app/jobs/appointments/remind_job.rb config/recurring.yml spec/jobs/appointments/remind_job_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: remind citizens of confirmed appointments the evening before

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 19: Caixa de avisos com campanhas e lembretes

**Files:**
- Modify: `app/controllers/citizen_api/notices_controller.rb`
- Modify: `spec/requests/citizen_api/notices_spec.rb` (o aviso de campanha ganha `kind`)
- Test: `spec/requests/citizen_api/appointment_notices_spec.rb`

**Interfaces:**
- Consumes: `AppointmentNotice` (Task 2), `Appointments::Remind` (Task 18), `Scheduling::UnitAddress` (Task 17).
- Produces: `GET /citizen/notices` → `{ notices: [ campanha | lembrete ], unread_count }`, mais novo primeiro entre as duas fontes; campanha = a forma de hoje + `kind: "campaign"`; lembrete = `{ kind: "appointment_reminder", id, appointment_id, appointment_type_name, unit_name, unit_address: { street, number, complement, zip }, scheduled_at, professional_name, read, cpf_masked }` (contratos §5, §8); `unread_count` conta as duas fontes, sem quem silenciou (`notices_muted`); `POST /citizen/notices/:id/read` acha o id nas duas.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/citizen_api/appointment_notices_spec.rb
require "rails_helper"

# Contratos §5, §8: a caixa une campanhas e lembretes; o id é opaco e a
# leitura vale para os dois.
RSpec.describe "Caixa de avisos com lembretes", type: :request do
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset }

  let(:phone) { "+5541998765432" }
  let!(:ana) { person!(phone: phone, cpf: "52998224725") }
  let(:unit) { create_unit.tap { |u| u.update!(address_street: "Rua das Flores", address_number: "10") } }
  let(:link) { doctor_link!(unit) }
  let(:shift) { shift!(link, starts_at: 2.days.from_now.change(hour: 8)) }
  def body = JSON.parse(response.body)

  def reminder!(citizen, at: 30.minutes.ago)
    appointment = appointment_row!(triage_request!(citizen, unit: unit), shift,
                                   starts_at: shift.starts_at + (AppointmentNotice.count * 20).minutes)
    AppointmentNotice.create!(appointment: appointment, citizen: citizen, created_at: at)
  end

  it "une as duas fontes, mais novo primeiro, com a forma do lembrete; conta e marca lido nas duas" do
    campaign = recipient!(sent_campaign!(title: "Vacinação", dispatched_at: 2.hours.ago), ana, sms_status: "not_opted_in")
    notice = reminder!(ana)
    sign_in_citizen(phone)

    get "/citizen/notices"
    expect(body["notices"].map { |n| [ n["kind"], n["id"] ] })
      .to eq([ [ "appointment_reminder", notice.id ], [ "campaign", campaign.id ] ])
    expect(body["notices"].first).to eq(
      "kind" => "appointment_reminder", "id" => notice.id, "appointment_id" => notice.appointment_id,
      "appointment_type_name" => "Consulta médica", "unit_name" => unit.name,
      "unit_address" => { "street" => "Rua das Flores", "number" => "10", "complement" => nil, "zip" => nil },
      "scheduled_at" => notice.appointment.scheduled_at.iso8601,
      "professional_name" => link.professional.professional_name, "read" => false, "cpf_masked" => nil
    )
    expect(body["unread_count"]).to eq(2)

    json_post "/citizen/notices/#{notice.id}/read"
    expect(response).to have_http_status(:ok)
    json_post "/citizen/notices/#{notice.id}/read"
    expect(response).to have_http_status(:ok)
    get "/citizen/notices"
    expect(body["unread_count"]).to eq(1)
    expect(body["notices"].first["read"]).to be(true)
  end

  it "duas pessoas no celular: cpf_masked; quem silenciou não conta; aviso de outro celular é 404" do
    bia = person!(phone: phone, cpf: "11144477735")
    reminder!(ana)
    reminder!(bia, at: 1.hour.ago)
    CitizenContactPreference.create!(citizen_id: bia.id, notices_muted: true)
    stranger = reminder!(person!, at: 2.hours.ago)
    sign_in_citizen(phone)

    get "/citizen/notices"
    expect(body["notices"].map { |n| n["cpf_masked"] }).to eq([ ana.cpf_masked, bia.cpf_masked ])
    expect(body["unread_count"]).to eq(1)
    json_post "/citizen/notices/#{stranger.id}/read"
    expect(response).to have_http_status(:not_found)
  end
end
```

(`person!`, `recipient!` e `sent_campaign!` são de `spec/support/campaign_helpers.rb`; `person!` sem argumentos cria alguém noutro celular.)

Em `spec/requests/citizen_api/notices_spec.rb`, o `eq` do primeiro exemplo ganha `"kind" => "campaign"`.

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/requests/citizen_api/appointment_notices_spec.rb`
Expected: FAIL (a caixa só lê campanhas).

- [ ] **Step 3: Implemente**

`app/controllers/citizen_api/notices_controller.rb` inteiro:

```ruby
# Caixa de avisos (ADR 0024; F-12.4; ADR 0029 §6). A sessão é do TELEFONE: a
# caixa mostra os avisos de todos os cidadãos dele (D12), com cpf_masked quando
# há mais de um. Duas fontes: campanhas (campaign_recipients) e lembretes de
# horário (appointment_notices); mais novo primeiro entre as duas. unread_count
# ignora quem silenciou; os avisos continuam na lista. O id é opaco: a leitura
# procura nas duas fontes.
module CitizenApi
  class NoticesController < BaseController
    def index
      citizens = current_citizen_session.citizens.to_a
      ids = citizens.map(&:id)
      masked = citizens.size > 1 ? citizens.to_h { |c| [ c.id, c.cpf_masked ] } : {}
      muted = CitizenContactPreference.where(citizen_id: ids, notices_muted: true).pluck(:citizen_id)
      campaigns = visible(ids).preload(:campaign).to_a.map { |row| [ row.campaign.dispatched_at, notice_json(row, masked[row.citizen_id]) ] }
      reminders = reminders(ids).to_a.map { |row| [ row.created_at, reminder_json(row, masked[row.citizen_id]) ] }
      render json: {
        notices: (campaigns + reminders).sort_by { |at, json| [ -at.to_f, json[:id] ] }.map(&:last),
        unread_count: visible(ids - muted).where(notice_read_at: nil).count +
                      AppointmentNotice.where(citizen_id: ids - muted, read_at: nil).count
      }
    end

    # Idempotente e sem corrida: só grava quando ainda está NULL (os triggers
    # recusam trocar uma leitura já gravada).
    def read
      own = current_citizen_session.citizens.select(:id)
      if (row = visible(own).find_by(id: params[:id]))
        CampaignRecipient.where(id: row.id, notice_read_at: nil).update_all(notice_read_at: Time.current)
      elsif (notice = AppointmentNotice.where(citizen_id: own).find_by(id: params[:id]))
        AppointmentNotice.where(id: notice.id, read_at: nil).update_all(read_at: Time.current)
      else
        return render_error("not_found", :not_found)
      end
      render json: { ok: true }
    end

    private

    def visible(citizen_ids)
      CampaignRecipient.joins(:campaign).where(citizen_id: citizen_ids, campaigns: { status: "sent" })
    end

    def reminders(citizen_ids)
      AppointmentNotice.where(citizen_id: citizen_ids).includes(appointment: %i[health_unit professional])
    end

    def notice_json(row, cpf_masked)
      {
        kind: "campaign", id: row.id, title: row.campaign.title, body: row.campaign.body,
        dispatched_at: row.campaign.dispatched_at.iso8601, read: row.notice_read_at.present?, cpf_masked: cpf_masked
      }
    end

    def reminder_json(row, cpf_masked)
      appointment = row.appointment
      {
        kind: "appointment_reminder", id: row.id, appointment_id: appointment.id,
        appointment_type_name: catalog.name_for(appointment.appointment_type_key),
        unit_name: appointment.health_unit.name, unit_address: Scheduling::UnitAddress.call(appointment.health_unit),
        scheduled_at: appointment.scheduled_at.iso8601, professional_name: appointment.professional&.professional_name,
        read: row.read_at.present?, cpf_masked: cpf_masked
      }
    end

    def catalog = @catalog ||= Scheduling::AppointmentTypes.catalog
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/requests/citizen_api/appointment_notices_spec.rb spec/requests/citizen_api/notices_spec.rb spec/requests/citizen_api/contact_preferences_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add app/controllers/citizen_api/notices_controller.rb spec/requests/citizen_api/appointment_notices_spec.rb spec/requests/citizen_api/notices_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "feat: show appointment reminders in the citizen notice box

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 7 — Invariantes, semente e fechamento

### Task 20: Suíte de invariantes do ADR 0029

**Files:**
- Create: `spec/invariants/scheduling_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 2–19.
- Produces: o critério de fechamento do módulo (spec §9 "Invariantes"), um `describe` por invariante do ADR 0029.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/invariants/scheduling_invariants_spec.rb
require "rails_helper"

# Módulo 17, critério de fechamento (ADR 0029, "Invariantes"). Onde o banco
# garante, o ataque é por SQL (insert_all/update_all); onde a garantia é a trava
# do comando, a prova sob disputa está em
# spec/commands/appointments/booking_concurrency_spec.rb e aqui fica a
# sequencial. Como em db/city_triggers.sql, isto NÃO defende contra o DONO da tabela.
RSpec.describe "Invariantes da agenda (ADR 0029)" do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A; ensure_appointment_types! }
  after { Current.reset; Rails.cache.clear }

  let(:unit) { create_unit }
  let(:link) { doctor_link!(unit) }
  let(:admin) { staff_with("admin-inv@cidade.gov.br", "municipal_admin") }
  let(:reception) { staff_with("recepcao-inv@cidade.gov.br", "citizen_verifier") }
  let(:shift) { shift!(link, starts_at: 4.days.from_now.change(hour: 8)) }
  let(:cpfs) { %w[52998224725 11144477735 39053344705 87748248800] }
  def citizen(i) = Citizen.create!(cpf: cpfs[i], phone: "+55419#{format('%08d', 70_000_000 + i)}")
  def attempt(&) = ApplicationRecord.transaction(requires_new: true, &)

  describe "nenhum horário slot ativo se sobrepõe a outro do mesmo profissional" do
    it "o banco recusa, mesmo por insert_all" do
      first = appointment_row!(triage_request!(citizen(0), unit: unit), shift, starts_at: shift.starts_at)
      other = triage_request!(citizen(1), unit: unit)
      row = first.attributes.except("id").merge("request_id" => other.id, "citizen_id" => other.citizen_id,
                                                "scheduled_at" => shift.starts_at + 5.minutes,
                                                "ends_at" => shift.starts_at + 25.minutes)
      expect { attempt { Appointment.insert_all!([ row ]) } }.to raise_error(ActiveRecord::ExclusionViolation)
    end
  end

  describe "nenhum turno passa do limite de encaixes" do
    it "o terceiro encaixe num turno de limite 2 é recusado" do
      results = 3.times.map do |i|
        Appointments::FitIn.call(request: triage_request!(citizen(i), unit: unit), professional: link.professional,
                                 shift: shift, starts_at: (shift.starts_at + (i * 30).minutes).iso8601,
                                 type: AppointmentType.find_by!(key: "consulta_medica"), reason: "retorno que não espera",
                                 by: reception)
      end
      expect(results.map { |r| r.reason || :ok }).to eq([ :ok, :ok, :fit_in_limit ])
      expect(Scheduling::FitInLimit.count(shift)).to eq(2)
    end
  end

  describe "resultado urgente nunca gera pedido" do
    it "Triages::Schedule com prioridade urgente não grava nada" do
      protocol = active_protocol!("saude-do-idoso", scheduling: [
        { "when" => { "gte" => ["outcome.score", 0] }, "appointment_type" => "consulta_medica", "priority" => "priority", "due_in_days" => 1 }
      ])
      par = profiled_citizen!(age: 70)
      conversation = Conversation.create!(channel: "web", citizen: par, phone: par.phone, state: "completed")
      triage = Triage.create!(conversation: conversation, protocol_definition: protocol, protocol_name: protocol.name,
                              status: "completed", answers: { "q1" => "true" }, priority: 1, tier: "media")
      urgent = Struct.new(:tier, :score, :priority, :terminal?).new("media", 4, 1, true)
      expect(Protocols::Urgency.urgent?(urgent)).to be(true)
      expect(Triages::Schedule.call(triage: triage, outcome: urgent)).to be_nil
      expect(AppointmentRequest.where(citizen: par)).to be_empty
    end
  end

  describe "mudança de turno ou modelo nunca apaga nem move horário marcado" do
    it "cancelar o turno, trocar ou editar o modelo deixam o horário igual; o banco recusa mover e apagar" do
      template = ScheduleTemplate.create!(name: "Manhã", blocks: [ { "starts" => "08:00", "ends" => "09:00", "kind" => "bookable",
                                                                     "appointment_type_key" => "consulta_medica" } ])
      Professionals::SetShiftTemplate.call(shift: shift, schedule_template_id: template.id, by: admin)
      appt = appointment_row!(triage_request!(citizen(0), unit: unit), shift, starts_at: shift.starts_at)
      before = appt.reload.attributes

      Scheduling::SaveTemplate.call(template: template, attrs: { "blocks" => [ { "starts" => "10:00", "ends" => "11:00", "kind" => "blocked" } ] }, by: admin)
      Professionals::SetShiftTemplate.call(shift: shift, schedule_template_id: nil, by: admin)
      Professionals::CancelShift.call(shift: shift.reload, reason: "troca de escala", by: admin)
      expect(appt.reload.attributes).to eq(before)

      %w[scheduled_at ends_at professional_id shift_id].each do |column|
        value = column.end_with?("_at") ? appt.public_send(column) + 1.hour : doctor_link!(unit).professional_id
        value = shift!(link, starts_at: 9.days.from_now.change(hour: 8)).id if column == "shift_id"
        expect { attempt { Appointment.where(id: appt.id).update_all(column => value) } }
          .to raise_error(ActiveRecord::StatementInvalid, /scheduled columns never change/), column
      end
      expect { attempt { Appointment.where(id: appt.id).delete_all } }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
    end
  end

  describe "nenhum texto livre (motivo, justificativa) em evento, log ou Analytics" do
    it "encaixe e remarcação pedida não deixam o texto em domain_events; os parâmetros saem filtrados do log" do
      fit = Appointments::FitIn.call(request: triage_request!(citizen(0), unit: unit), professional: link.professional,
                                     shift: shift, starts_at: (shift.starts_at + 10.minutes).iso8601,
                                     type: AppointmentType.find_by!(key: "consulta_medica"),
                                     reason: "gestante com sangramento", by: reception).payload[:appointment]
      Appointments::RequestReschedule.call(appointment: fit, reason_code: "health", note: "tenho fisioterapia às 8h",
                                           preferred_period: "afternoon")
      payloads = DomainEvent.pluck(:payload).to_json
      expect(payloads).not_to include("gestante", "fisioterapia", "Remarcação pedida")

      filter = ActiveSupport::ParameterFilter.new(Rails.application.config.filter_parameters)
      filtered = filter.filter("reason" => "a", "note" => "b", "fit_in_reason" => "c", "reschedule_note" => "d")
      expect(filtered.values).to all(eq("[FILTERED]"))
    end

    it "nada de Analytics lê as colunas de texto livre da agenda" do
      sources = Dir[Rails.root.join("app/services/analytics/**/*.rb")].map { |f| File.read(f) }.join
      expect(sources).not_to match(/fit_in_reason|reschedule_note|cancel_reason|dismiss_reason/)
    end
  end

  describe "o cidadão nunca escolhe a vaga" do
    it "nenhuma rota do cidadão cria horário; pedir outro horário ignora qualquer início enviado" do
      citizen_posts = Rails.application.routes.routes.filter_map do |route|
        path = route.path.spec.to_s
        path.delete_suffix("(.:format)") if route.verb == "POST" && path.start_with?("/citizen/appointments")
      end
      expect(citizen_posts).to contain_exactly("/citizen/appointments/:id/confirm", "/citizen/appointments/:id/cancel",
                                               "/citizen/appointments/:id/check_in_code",
                                               "/citizen/appointments/:id/reschedule_request")

      appt = appointment_row!(triage_request!(citizen(0), unit: unit), shift, starts_at: shift.starts_at)
      expect(Appointment.where(status: Appointment::LIVE).count).to eq(1)
      Appointments::RequestReschedule.call(appointment: appt, reason_code: "work", note: nil, preferred_period: "any")
      expect(Appointment.count).to eq(1) # cancelou o horário; nenhum novo nasceu
      expect(Appointment.where(status: Appointment::LIVE).count).to eq(0)
    end
  end
end
```

- [ ] **Step 2: Rode e prove que morde**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/invariants/scheduling_invariants_spec.rb`
Expected: PASS.

Mutação (não commitar; desfazer cada uma): (a) tire `OR NEW.shift_id IS DISTINCT FROM OLD.shift_id` de `rota_appointment_guard`, rode `bin/rails runner 'ActiveRecord::Base.connection.execute(File.read("db/city_triggers.sql"))'` contra o banco de teste A (ou recrie os bancos de teste) e o invariante 4 tem de falhar; (b) troque `>=` por `>` na comparação do limite em `FitIn` e o invariante 2 tem de falhar; (c) acrescente `reason: reason` ao `appointment.fit_in_created` e o invariante 5 tem de falhar. Relate as três falhas.

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add spec/invariants/scheduling_invariants_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "test: pin the ADR 0029 scheduling invariants

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 21: Semente de dev (spec §10)

**Files:**
- Create: `lib/scheduling_crew.rb`
- Modify: `lib/triage_catalog_crew.rb` (`run_cycle!` público)
- Modify: `db/seeds.rb`
- Test: `spec/lib/scheduling_crew_spec.rb`

**Interfaces:**
- Consumes: `SignatureCrew`, `ProfessionalCrew` (médica `profissional@` com vínculo `225125` e enfermeira `enfermeira@` com `223505` na "UBS Jardim das Flores"), `TriageCatalogCrew` (o `saude-do-idoso` ativo), `Scheduling::SaveTemplate`, `Professionals::ScheduleShift`, `Professionals::SetShiftTemplate`.
- Produces: `SchedulingCrew.seed_current_city(slug:) -> { types:, template:, new_shifts:, templated:, protocol: }` (idempotente); `TriageCatalogCrew.run_cycle!(slug, name, version, definition)` público.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/lib/scheduling_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/signature_crew")
require Rails.root.join("lib/professional_crew")
require Rails.root.join("lib/triage_catalog_crew")
require Rails.root.join("lib/scheduling_crew")

# Semente do módulo 17 (spec §10): base de tipos; em Curitiba, o modelo
# "Manhã" nos turnos da médica; turnos da semana para a médica e a enfermeira;
# "Saúde do idoso" com regra de agendamento (rotina, 30 dias), pelo ciclo
# assinado. Idempotente.
RSpec.describe SchedulingCrew do
  let!(:city_record) { register_test_city! }

  before do
    create_default_protocol!
    %w[Centro Batel Portão].each { |name| Neighborhood.create!(name: name, source: "seed") }
    admin = staff_with("admin@curitiba.demo", "municipal_admin")
    SignatureCrew.seed_current_city(slug: "curitiba", password: "dev-password")
    ProfessionalCrew.seed_current_city(slug: "curitiba", password: "dev-password", admin: admin)
    TriageCatalogCrew.seed_current_city(slug: "curitiba", ddd: "41")
  end
  after { Rails.cache.clear }

  def seed = described_class.seed_current_city(slug: "curitiba")

  it "tipos, modelo na médica, turnos da semana e o idoso com agendamento; a segunda rodada não cria nada" do
    result = seed
    expect(result[:types]).to eq(4)
    template = ScheduleTemplate.find_by!(name: SchedulingCrew::TEMPLATE_NAME)
    expect(template.blocks.map { |b| b.values_at("starts", "ends", "kind") })
      .to eq([ %w[07:00 09:00 walk_in], %w[09:00 11:00 bookable], %w[11:00 12:00 blocked] ])

    unit = HealthUnit.find_by!(name: SchedulingCrew::UNIT)
    medica = User.find_by!(email_address: "profissional@curitiba.demo").professional
    enfermeira = User.find_by!(email_address: "enfermeira@curitiba.demo").professional
    future = ->(p) { ProfessionalShift.valid_shifts.where(professional: p).where("starts_at > ?", Time.current) }
    expect(future.(medica).joins(:professional_link).where(professional_links: { health_unit_id: unit.id })
                 .pluck(:schedule_template_id).uniq).to eq([ template.id ])
    expect(future.(enfermeira).pluck(:schedule_template_id).uniq).to eq([ nil ])
    expect(future.(medica).count).to be >= 5

    idoso = ProtocolDefinition.find_by!(name: "saude-do-idoso", status: "active")
    expect(idoso.definition["scheduling"]).to eq(SchedulingCrew::SCHEDULING)
    expect(Protocols::Gate.call(idoso.definition)).to be_valid

    day = future.(medica).joins(:professional_link).where(professional_links: { health_unit_id: unit.id })
                .order(:starts_at).first.starts_at.to_date
    slots = Scheduling::Availability.for(unit: unit, from: day, to: day,
                                         appointment_type: AppointmentType.find_by!(key: "consulta_medica"))
    expect(slots.select { |s| s.professional_id == medica.id }.map { |s| s.starts_at.strftime("%H:%M") })
      .to eq(%w[09:00 09:20 09:40 10:00 10:20 10:40])

    counts = [ ScheduleTemplate.count, ProfessionalShift.count, ProtocolDefinition.where(name: "saude-do-idoso").count ]
    again = seed
    expect([ ScheduleTemplate.count, ProfessionalShift.count, ProtocolDefinition.where(name: "saude-do-idoso").count ]).to eq(counts)
    expect(again).to include(new_shifts: 0, templated: 0)
  end
end
```

Fora de Curitiba não há modelo (`slug == "curitiba"`): as contas `*@maringa.demo` não existem neste banco de teste, então a regra é conferida no review.

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/lib/scheduling_crew_spec.rb`
Expected: FAIL com `cannot load such file -- .../lib/scheduling_crew`.

- [ ] **Step 3: Implemente**

Em `lib/triage_catalog_crew.rb`, dentro de `class << self`, depois da definição de `run_cycle!`, acrescente `public :run_cycle!` (o ciclo assinado é reaproveitado pela semente do módulo 17).

```ruby
# lib/scheduling_crew.rb
require_relative "triage_catalog_crew"

# Semente de dev do módulo 17 (spec 2026-10-05 §10). Dev é fictício mas imita o
# real: a base de tipos copiada; em Curitiba, o modelo "Manhã" (demanda do dia
# 7–9h, consultas médicas 9–11h, bloqueio 11–12h) nos turnos da médica da
# semente na UBS Jardim das Flores; turnos da semana (dias úteis) para a médica
# (7–13h) e a enfermeira (7–19h, sem modelo: vagas de consulta de enfermagem
# pelo CBO); e "Saúde do idoso" com a regra de agendamento (rotina, 30 dias)
# numa versão nova assinada de verdade. Idempotente. Roda depois do
# TriageCatalogCrew.
class SchedulingCrew
  UNIT = "UBS Jardim das Flores"
  TEMPLATE_NAME = "Manhã: demanda do dia, consultas e bloqueio"
  BLOCKS = [
    { "starts" => "07:00", "ends" => "09:00", "kind" => "walk_in" },
    { "starts" => "09:00", "ends" => "11:00", "kind" => "bookable", "appointment_type_key" => "consulta_medica" },
    { "starts" => "11:00", "ends" => "12:00", "kind" => "blocked" }
  ].freeze
  SCHEDULING = [ { "when" => { "gte" => ["outcome.score", 3] }, "appointment_type" => "consulta_medica",
                   "priority" => "routine", "due_in_days" => 30 } ].freeze
  ELDERLY = TriageCatalogCrew::ELDERLY

  class << self
    def seed_current_city(slug:)
      Scheduling::AppointmentTypes.seed_platform!
      admin = User.find_by!(email_address: "admin@#{slug}.demo")
      unit = HealthUnit.find_by!(name: UNIT)
      medica = link_for("profissional", slug, unit, "225125")
      enfermeira = link_for("enfermeira", slug, unit, "223505")
      new_shifts = ensure_week!(medica, admin, 7, 13) + ensure_week!(enfermeira, admin, 7, 19)
      template = slug == "curitiba" ? ensure_template!(admin) : nil
      templated = template ? attach!(medica, template, admin) : 0
      protocol = ensure_protocol!(slug)
      { types: AppointmentType.count, template: template&.name, new_shifts: new_shifts, templated: templated,
        protocol: "#{protocol.name} v#{protocol.version}" }
    end

    private

    def link_for(prefix, slug, unit, cbo)
      professional = User.find_by!(email_address: "#{prefix}@#{slug}.demo").professional
      professional.links.active.find_by!(health_unit: unit, cbo_code: cbo)
    end

    def business_days = (1..9).map { |n| Time.zone.today + n }.reject { |d| d.saturday? || d.sunday? }.first(5)

    # O ProfessionalCrew já lança turnos; aqui só preenche o dia útil que ficou sem.
    def ensure_week!(link, admin, from_hour, to_hour)
      business_days.count do |day|
        starts = day.in_time_zone.change(hour: from_hour)
        ends = day.in_time_zone.change(hour: to_hour)
        next false if link.shifts.valid_shifts.where("starts_at < ? AND ends_at > ?", ends, starts).exists?

        result = Professionals::ScheduleShift.call(link: link, starts_at: starts, ends_at: ends, by: admin)
        next false if result.reason == :shift_overlap # turno noutra unidade no mesmo horário

        raise "semente da agenda: turno recusado (#{result.reason})" if result.failure?

        true
      end
    end

    def ensure_template!(admin)
      existing = ScheduleTemplate.find_by(name: TEMPLATE_NAME)
      return existing if existing

      result = Scheduling::SaveTemplate.call(attrs: { "name" => TEMPLATE_NAME, "fit_in_limit" => 2, "blocks" => BLOCKS.map(&:dup) },
                                             by: admin)
      raise "semente da agenda: modelo recusado (#{result.reason} #{result.details})" if result.failure?

      result.payload[:template]
    end

    def attach!(link, template, admin)
      link.shifts.valid_shifts.where("starts_at > ?", Time.current).where(schedule_template_id: nil).count do |shift|
        Professionals::SetShiftTemplate.call(shift: shift, schedule_template_id: template.id, by: admin).ok?
      end
    end

    # Versão nova = a ativa + scheduling; retoma a versão pendente que já o leva.
    def ensure_protocol!(slug)
      active = ProtocolDefinition.find_by!(name: ELDERLY, status: "active")
      return active if active.definition["scheduling"] == SCHEDULING

      versions = ProtocolDefinition.where(name: ELDERLY)
      pending = versions.where.not(status: %w[active retired]).find { |v| v.definition["scheduling"] == SCHEDULING }
      version = pending&.version || (versions.maximum(:version) + 1)
      TriageCatalogCrew.run_cycle!(slug, ELDERLY, version,
                                   active.definition.merge("version" => version, "scheduling" => SCHEDULING))
    end
  end
end
```

Em `db/seeds.rb`: `require Rails.root.join("lib/scheduling_crew").to_s` junto dos outros `require`; e, depois do bloco do catálogo de triagens (módulo 15):

```ruby
        # ── Agenda (módulo 17, spec 2026-10-05 §10) ───────────────────────────
        # Depois do catálogo: usa o saude-do-idoso ativo, a médica e a enfermeira.
        agenda = SchedulingCrew.seed_current_city(slug: slug)
        puts "[seeds] agenda ...... #{agenda[:types]} tipos; modelo: #{agenda[:template] || 'nenhum'}; " \
             "#{agenda[:new_shifts]} turnos novos, #{agenda[:templated]} com modelo; #{agenda[:protocol]} com agendamento"
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec spec/lib/scheduling_crew_spec.rb spec/lib/triage_catalog_crew_spec.rb spec/lib/professional_crew_spec.rb`
Expected: PASS.

A rodada em dev (`db:seed`) fica para a Task 22, depois do `city:migrate:all`, com o usuário.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod17 add lib/scheduling_crew.rb lib/triage_catalog_crew.rb db/seeds.rb spec/lib/scheduling_crew_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod17 commit -m "chore: seed dev schedules, a morning template and a scheduling rule

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 22: Revisão final, suíte completa e api na porta 3034

- [ ] **Step 1:** Confira a cópia do schema contra a tag do `contracts`:

  ```bash
  /opt/homebrew/bin/git -C contracts fetch --tags
  diff <(/opt/homebrew/bin/git -C contracts show protocols-v1.5.0:protocols/schema.json) apps/api/.claude/mod17/config/protocols/schema.json
  ```
  Expected: sem diferença. Se a tag não existe, **pare** e avise: o api não mergeia antes do `contracts`.

- [ ] **Step 2:** Um subagente revisor lê `origin/main..HEAD` do api contra a spec, o ADR 0029, os contratos (com §8 e §9) e as seções "Desvios" e "Divergências" deste plano, com atenção a:
  - nenhum `DomainEvents.publish` com texto livre (`grep -rn -A3 "DomainEvents.publish(" app lib` — as chamadas são multilinha; compare call sites × nomes distintos com a lista declarada na Task 2);
  - `Current.city` nunca atribuído em `app/`/`lib/`; `RemindJob` com `prepend EachCityJob`;
  - ordem de travas (cidadão → horário → pedido → unidade → turno) em `Placement`, `Book`, `FitIn`, `RequestReschedule` (horário → pedido) e `AssignUnit` — nenhum caminho novo trava pedido antes de horário;
  - toda leitura do cidadão passa por `current_citizen_session.citizens` (horários, avisos, triagens);
  - `fit_in_reason` só sai com `show_reason: true` (balcão), nunca em Minha agenda nem para o cidadão; `reschedule_note` só no detalhe do pedido;
  - N+1 na fila, nas agendas e na caixa de avisos (`includes` em `QUEUE_INCLUDES`, `UnitAgenda`, `ProfessionalAgenda`, `reminders`).
- [ ] **Step 3:** Corrija os achados e rode a suíte completa com o worker parado:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod17 api bundle exec rspec
  docker compose start worker
  ```
  Expected: 0 falhas. Relatório: commits, contagem e tempo (s/exemplo; > 0,15 s/exemplo com host calmo é regressão), as mutações das Tasks 11 e 20, e as specs existentes alteradas e por quê (`adr_pointers_spec`, `appointment_invariants_spec`, `domain_events_bindings_spec`, `appointment_requests_spec`, `notices_spec`, `schedule_spec`, `drain_spec`, `erase_spec`, `provision_city_job_spec`, `triage_catalog_helpers`, e as que pediram as chaves novas de vínculo/turno/resultado).
- [ ] **Step 4:** Para a prova no navegador (spec §9, com o usuário) e para os planos do dashboard e do wpda, suba o api do worktree no container, na porta **3034**, sem derrubar o servidor principal:

  ```bash
  docker compose exec -d -w /rails/.claude/mod17 api bin/rails server -b 0.0.0.0 -p 3034 -P tmp/pids/server-mod17.pid
  ```

  O Vite do worktree de cada front aponta o proxy para `http://api:3034` (`VITE_API_PROXY_TARGET`). **Antes**, com autorização do usuário: `city:migrate:all` (as cidades de dev passam a ter as tabelas da agenda — o checkout principal fica atrás até o merge) e `db:seed` (a Task 21). Roteiro: o `admin@curitiba.demo` vê o modelo "Manhã" e a pré-visualização; a avó (62) responde "Saúde do idoso" com "sim" e o resultado diz a unidade e o prazo; a recepção acha o pedido na fila, marca uma vaga da médica e tenta um terceiro encaixe (409); a avó pede outro horário no wpda e o pedido volta à fila marcado; o lembrete da véspera sai às 17h (no dev: `Appointments::RemindJob.perform_now` dentro do horário, com `travel` só em console). Login, OTP e TOTP são do usuário (senhas e TOTP da semente de dev podem ser mostrados no chat se ele pedir).
- [ ] **Step 5:** **Pare.** Merge, push, board e docs (página de status do módulo) só com autorização explícita do usuário, uma etapa de cada vez. Ordem: `contracts` → api → dashboard e wpda. Rollout por cidade: publicar a imagem nova e rodar `city:migrate:all` dela antes de cortar tráfego (a migração aborta, listando ids, se achar horário `slot` sobreposto). A forma nova da fila (`triage_priority`) e da agenda (`professionals`) é usada pelo dashboard novo; a chave antiga `appointments` da agenda segue até ele entrar. Ao voltar o checkout para a main: `DROP DATABASE` dos dois bancos de teste de cidade e `city:test_databases` de novo; derrube o servidor da 3034 (`kill $(cat tmp/pids/server-mod17.pid)` no container).

---

## Self-review (feito ao escrever o plano)

**Cobertura da spec e dos contratos:**

| Spec / contrato | Task |
|---|---|
| §3.1 tipos (base no yml, cópia na migração e no provisionamento, `key`/`origin` imutáveis, `serves?`) | 2, 3, 4 |
| §3.2 modelos (faixas, limite, `active`), modelo no turno, tipo padrão do vínculo, `default_fit_in_limit`, pré-visualização | 2, 6, 7, 10 |
| §4.1 `Availability` (puro, tabela de casos) | 5, 8 |
| §4.2 colunas, `CHECK`s, `EXCLUDE` (com verificação prévia) | 2 |
| §4.3 `Book`, `FitIn`, remarcar pela recepção, transição `legacy`, turno cancelado → `needs_reschedule` | 8, 9, 10, 12, 15 |
| §5.1 schema 1.5.0 e gate (variáveis, aviso de tipo) | 1 |
| §5.2 pedido da triagem (regra, urgente, unidade/sem unidade, fusão, revogação) e campos de todos os pedidos | 2, 14, 16 |
| §5.3 fila (ordem, marcas, sem unidade, `assign_unit`) | 15 |
| §6 "Não posso nesse horário", lembrete (17h, idempotente, SMS com chave e opt-in), caixa de avisos, resultado | 16, 17, 18, 19 |
| §7 agendas (unidade, Minha agenda) e APIs de profissionais | 4, 6, 7, 13 |
| §8 LGPD (texto livre fora de evento/log/Analytics, revogação, exclusão) | 16, 20 |
| §9 testes (tabela de casos, threads, pedido da triagem, cidadão, lembrete, transição, migração, invariantes) | 5, 11, 14, 17, 18, 12, 2, 20 |
| §10 semente | 21 |
| §11 rollout (`VALID_RANGE`, `city:migrate:all`, ordem) | 2, 22 |
| Contratos §3–§6, §8, §9 | 4, 6, 7, 8, 12, 13, 15, 17, 19 |

**Placeholders:** nenhum "TBD"/"implementar depois". Os ajustes em specs existentes dizem a linha e o texto; onde o conteúdo exato de um arquivo existente não foi lido ao planejar (`schedule_spec`, `drain_spec`, `erase_spec`, `attendance_contract_spec`, `start_citizen_triage`), a regra do ajuste está escrita.

**Tipos e nomes:** `Scheduling::Availability::{Type,Shift,Busy,Slot,Block}` (Tasks 3, 5) são os de 6, 8, 9, 12, 13; `AppointmentTypes::Catalog#{find,name_for,active,fallback}` (3) em 4, 6, 8, 12, 13, 15, 16, 17, 19; `Placement.{lock!,bookable?,citizen_busy?,create!}` (9) em 10; `AppointmentPresenter` (12) em 13, 15; `RequestJson` (15); `BlockJson.{list,for_day}` (6, 13); `FitInLimit.{for,count}` (10) em 13, 20; `DateRange.parse` (8) em 13; helpers `type_row!`, `doctor_link!`, `shift!`, `triage_request!`, `appointment_row!` (2) e `ensure_appointment_types!` (3) nas specs seguintes.

**Ordem e suíte verde:** a Task 1 só commita o que não depende de `AppointmentType`; o resto entra na Task 2. As specs do módulo 08 continuam verdes porque nenhum dia delas tem turno (`legacy`), a agenda mantém a chave antiga e a fila só renomeia a prioridade numérica (ajuste explícito na Task 15).

**Review Focus:** 1 → Tasks 5 e 18; 2 → Tasks 11 e 12; 3 → Tasks 12 e 13; 4 → Tasks 14, 15, 16, 17; 5 → Tasks 17 e 20.

---

## Divergências propostas ao contrato

Os contratos §8 e §9 já absorveram a maior parte do que este plano precisava. Ficam:

1. **Detalhe novo de `invalid_blocks`: `bad_block`** — faixa que não é objeto, chave fora do contrato, `kind` desconhecido, tipo ou `slot_minutes` em faixa que não é `bookable`, ou mais de 24 faixas. Nenhum dos detalhes do contrato (`overlap`, `missing_type`, `unknown_type`, `bad_time`, `empty`, `bad_slot_minutes`, `inactive_type`, `crosses_midnight`) descreve esses casos sem enganar a tela.
2. **Códigos novos:** `invalid_template` (422; turno com modelo inexistente ou inativo, em `POST /professionals/links/:id/shifts` e `POST /professionals/shifts/:id/template`), `invalid_kind` (422; `kind` desconhecido na marcação), `invalid` (422; `active` que não é booleano em tipo e modelo; amostra inválida na pré-visualização), `already_cancelled` (409; modelo em turno cancelado), `already_ended` (409) e `type_not_served`/`inactive_type` (422) no tipo padrão do vínculo, `invalid_unit` (422) e `request_not_open` (409) no `assign_unit`.
3. **Eventos a mais** (só ids, sem consumidor): `appointment_request.unit_assigned { request_id, unit_id, by_user_id }`, `professional.shift_template_set { shift_id, schedule_template_id, by_user_id }`, `professional.link_default_type_set { professional_link_id, appointment_type_key, by_user_id }`; e a remarcação pela recepção reaproveita `appointment.moved` (forma do api#29).
4. **Faixas em agenda e pré-visualização** saem na forma do modelo (§2), recortadas pelo turno e pelo dia; o recorte que chega à meia-noite termina em `"24:00"` (só na saída — o modelo continua recusando `24:00`). Turno sem modelo vira uma faixa `bookable` do tipo resolvido, ou nenhuma faixa.
5. **Agenda da unidade** mantém, além da forma nova, a chave antiga `appointments` (forma do módulo 08) até o dashboard novo entrar; nenhuma tela nova deve lê-la.
6. **Item da fila** traz também `target_unit_id` e `appointment` (o horário vivo, §4.4, ou `null`) — o §4.4 diz que a forma única vale "na fila"; e `reopened_reason` continua `expired|no_show|null` (a remarcação pedida aparece só em `reschedule_requested`). O mesmo mapeamento vale para `request.reopened_reason` em `GET /citizen/appointments`.
7. **`GET /attendance/units/:id/availability`** com tipo inexistente ou desativado responde 200 `{ slots: [], legacy_days: [] }` (sem erro); sem `from`/`to`, hoje + 6 dias.
8. **Leitura de `GET /professionals/appointment_types`** inclui também `citizen_verifier` e `health_professional` (a recepção escolhe o tipo no encaixe; o profissional vê os nomes na Minha agenda), além de `municipal_admin`, `protocol_author` e `protocol_reviewer`.
9. **`citizen` no horário (§4.4)** sai sem `name` (o cadastro não tem nome).
10. **Tipo do protocolo inexistente na cidade** não impede o pedido: ele nasce com a key e `appointment_type_name` cai para a própria key.
