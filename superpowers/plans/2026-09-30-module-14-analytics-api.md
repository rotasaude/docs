# Módulo 14 — Analytics (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Consolidação diária anônima por cidade (`analytics_daily_facts` + `analytics_runs`), papel `analyst`, as quatro frentes em `GET /admin/api/analytics/:front` com supressão de 1 a 4 sempre, `analytic` no schema de protocolo, seis indicadores semanais publicados no banco de plataforma e lidos pelo console do operador (`GET /city_analytics`) e pelo GraphQL de manutenção — o lado api de F-14.1 a F-14.9 (ADR 0025).

**Architecture:** Uma migração de cidade só de expansão cria `analytics_daily_facts` (índice único `NULLS NOT DISTINCT`) e `analytics_runs` e acrescenta `analyst` ao CHECK de papéis; uma migração de plataforma cria `city_analytics_indicators`. `Analytics::Consolidate` apaga os fatos da janela e chama um consolidador por frente (`Demand`, `Quality`, `Calibration`, `Epidemiology`), cada métrica um `INSERT ... SELECT ... GROUP BY` sobre o cru, sem linha de pessoa no Ruby. `Analytics::Run` segura um advisory lock por cidade, grava o `analytics_runs`, refaz a janela numa transação, publica (`Analytics::Publish`) depois do commit e purga; `ConsolidateAnalyticsJob` (recorrente de cidade) e `Analytics::Rebuild` (rake) só chamam o `Run`. A leitura (`app/queries/analytics/*`) soma período e recorte no SQL e só então suprime (`Analytics::Suppression`, sobre `Admin::SmallCount`).

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + banco de plataforma), Solid Queue, RSpec, graphql-ruby, json_schemer.

**Spec:** `docs/.claude/mod14/superpowers/specs/2026-09-30-module-14-analytics-design.md` e `docs/.claude/mod14/adr/0025.md` (leia os dois antes de começar). Contratos entre apps: `docs/.claude/mod14/superpowers/plans/2026-09-30-module-14-analytics-contracts.md` — os planos do dashboard, do admin e do maintenance foram escritos contra ele: **não mude nomes, formatos nem códigos de erro**. A mudança do schema de protocolo nasce no repo `contracts` (plano `2026-09-30-module-14-analytics-contracts-repo.md`); a Task 7 copia o arquivo de lá.

## Desvios da spec (e precisões de contrato)

Onde a spec é omissa ou o código real obrigou a escolher, a escolha está aqui:

1. **Revogação de verdade não aborta a triagem concluída.** `RevokeConsent` (`app/commands/revoke_consent.rb:20`) só passa a `aborted_by_revocation` a triagem `in_progress`; o wpda revoga **depois** de concluir (`CitizenApi::TriagesController#revoke_consent`), e a triagem segue `completed`, com respostas e bairro (o `AnonymizeRevokedTriageJob` só limpa as abortadas). A spec §4.2 supõe que "o cru muda". Para cumprir a intenção ("dentro da janela a pessoa sai na próxima consolidação"), o Analytics define **triagem revogada** = `status = 'aborted_by_revocation'` **ou** a conversa da própria triagem tem consentimento revogado. Triagem revogada conta em `triage.started` e em `triage.aborted` (`dim = 'revocation'`), as duas **sem bairro**, e nunca em `triage.completed`, `calibration.outcome` ou `epi.answer`.
2. **Um atendimento por triagem.** `attendances.triage_id` é único (`index_attendances_on_triage_id`), então "mais de um atendimento, o mais recente" (§3.4) não acontece. O atendimento do retorno nasce do horário (`appointment_id`, sem `triage_id`) e **não** conta na calibração: vale o desfecho do atendimento da própria triagem. O teste "0, 1 e 2 atendimentos" (§10.1) vira "sem atendimento, com atendimento, e com atendimento + atendimento do retorno".
3. **Recusa do operador com grant.** `Authentication#require_authentication` devolveria `operator_read_only`; o contrato pede `403 forbidden_role`. `Admin::Api::AnalyticsController` declara zero ações para grant (`allow_operator_grant_access(only: [])`, a spec §6.1 pede "sem `allow_operator_grant_access`") e sobrescreve `require_authentication` para devolver `forbidden_role` à sessão de grant. Usuário sem **nenhum** vínculo ativo continua recebendo `403 no_city_membership` do `BaseController` (não tem papel; o contrato fala de "outro papel").
4. **Envelope.** `Admin::Api::BaseController#render_envelope` exige o `@period` dos painéis ao vivo e acrescenta `scope`. O Analytics pula `resolve_scope` e devolve exatamente `{ data, as_of, stale }` (contratos §1), com `as_of` em UTC (`...Z`). Calibração, que não tem série, devolve `granularity: null` e `periods: []`.
5. **Parâmetros.** `granularity` fora de `week`/`month` → `422 invalid_range`. Recorte que não vale para a frente (ex.: `neighborhood_id` em `quality`) é ignorado e volta `null` em `filter`. Calibração limita o intervalo a 60 meses (a mesma régua de `month`). `from` depois do `to` já truncado para ontem → `invalid_range`.
6. **Horário do job: 2h30, não 2h.** `sweep_abandoned` roda às 2h e é quem transforma a triagem abandonada de ontem em `aborted_by_timeout`; rodando depois dele, o D-1 já nasce certo (a execução seguinte corrigiria de qualquer forma, pela janela de 30 dias).
7. **Publicação = apagar a semana e gravar de novo**, numa transação da plataforma, em vez de upsert linha a linha: quando uma taxa passa a "sem dado" (denominador 0 depois de um rebuild), a linha antiga precisa sumir, e o upsert a deixaria. Continua idempotente por (cidade, semana, indicador).
8. **Purga de `analytics_runs` preserva o último run `succeeded`.** Sem isso, uma cidade parada há mais de 90 dias diria "nunca consolidou" (`as_of: null`) com fatos no banco.
9. **Rake por argumento, não por env:** `city:analytics:rebuild[slug,from,to]` e `city:analytics:rebuild:all[from,to]`, no mesmo molde de `city:territory:seed[slug]`/`:all`. Sem `from`: desde o cru mais antigo da cidade; sem `to`: ontem.
10. **Protocolo de dev novo, não versão nova do respiratório.** O template `triage-respiratoria` só tem perguntas `boolean` (a spec §11 quer uma `enum`) e o `SignatureCrew` já deixa a próxima versão dele em rascunho para a demonstração do ciclo assinado; ativar outra versão aposentaria a v1 que as outras sementes usam. A semente cria `triagem-arbovirose` v1 (febre `boolean` e sintoma `enum` marcados; gestação `boolean` **sem** marca) e a leva pelo ciclo assinado inteiro por comandos (`SaveDraft` → `SubmitForReview` → 2 assinaturas → `Publish` → 2 assinaturas → `Activate`).
11. **"Versão mais recente que marcou" (§6.1) considera só versões que passaram pelo ciclo** (`published`, `active`, `retired`): rascunho e revisão não dão nome nem opções a uma pergunta.
12. **Faixas de espera:** `[0,15)` → `0-15`, `[15,30)` → `15-30`, `[30,60)` → `30-60`, `[60,120)` → `60-120`, `≥ 120` → `120+` (minutos entre check-in e chamada). Quem saiu sem ser chamado (`left`) não tem espera.
13. **Contagens que o contrato não nomeia:** `attendances_by_unit` (Demanda) soma `attendance.checked_in` (chegadas); `by_unit[].attendances` (Qualidade) soma `attendance.closed`. `requests_opened`/`requests_closed` trazem sempre todos os tipos/motivos, inclusive zerados, e, como as demais listas da Demanda, vêm por total desc (suprimido conta 0), depois nome. As listas fixas da Qualidade (faixas, estados, desfechos) seguem a ordem do contrato.
14. **Rebuild sobre período antigo** refaz os fatos a partir do cru que ainda existe: o que foi anonimizado desde então sai. A invariante "a revogação não altera fato fora da janela" vale para a consolidação agendada; o rebuild é ação explícita do operador, com aviso no runbook.
15. **GraphQL:** `analyticsIndicators` com `from > to` ou mais de 104 semanas → erro de campo com `extensions.code = "INVALID_RANGE"`. `analyticsStatus` é **anulável** (contratos §3, revisado): falha ao abrir a cidade anula só o campo, pelo mesmo `inside` de `counts`/`operations`.
16. **`analyticsStatus.lastError`** é o `error` do run mais recente: preenchido quando ele falhou ou quando a publicação dele falhou (`"publish: ..."`). O texto é classe + primeira linha da mensagem, até 500 caracteres (a linha `DETAIL` do PostgreSQL pode trazer valores da linha).

## Global Constraints

- Tudo de domínio no banco de cada cidade (`db/city_migrate`, `db/city_schema.rb` à mão); o banco de plataforma recebe **só** `city_analytics_indicators` (`db/platform_migrate`, `db/platform_schema.rb` gerado). Migrações só de expansão e reversíveis.
- `analytics_daily_facts`, `analytics_runs` e `city_analytics_indicators` **não têm coluna de pessoa** (cidadão, usuário, CPF, telefone, conversa, triagem).
- Métricas (`metric`), exatamente: `triage.started`, `triage.completed`, `triage.aborted`, `attendance.checked_in`, `attendance.closed`, `attendance.wait`, `appointment.ended`, `request.opened`, `request.closed`, `calibration.outcome`, `epi.answer`.
- Indicadores da plataforma, exatamente: `triages_started`, `triages_completed`, `attendances_closed`, `wait_within_30_pct`, `no_show_pct`, `left_pct`.
- Supressão **1–4 sempre**, depois de somar período e recorte, com `Admin::SmallCount`; zero continua `0`. Taxa: suprimida se numerador **ou** denominador em 1..4; `null` com denominador 0; 1 casa decimal. Na plataforma, suprimido = `value NULL, suppressed true`.
- Fuso fixo `America/Sao_Paulo` para o "dia" (api#27). O dia corrente **nunca** é consolidado (dado até D-1). Janela agendada: de hoje − 30 a ontem.
- Retenção: fatos 5 anos (`day < hoje − 5 anos` é purgado); `analytics_runs` 90 dias (menos o último `succeeded`); indicadores da plataforma 5 anos.
- Papel `analyst` em `Membership::ROLES`, **fora** de `PRIVILEGED_ROLES` (sem step-up). Leem `/admin/api/analytics/*` só `analyst` e `municipal_admin` ativos na cidade; o operador, com ou sem grant, nunca.
- O console do operador e o maintenance **nunca** abrem banco de cidade para ler indicador (`analyticsStatus` lê `analytics_runs` da cidade, como os outros campos operacionais).
- Nenhum `Platform.audit` novo e nenhum `DomainEvents.publish` novo: a trilha do Analytics é `analytics_runs`.
- `Current.city` **nunca** é atribuído em `app/` ou `lib/` (`spec/architecture/current_city_assignment_spec.rb`); o job usa `EachCityJob`, o rake usa `CityConnection.with`.
- Specs de request precisam de `type: :request` (inferência desligada). Arquivo novo em `spec/support/` precisa de `require_relative` em `spec/rails_helper.rb`. Nenhuma data fixa contra o relógio real: datas derivadas de `Time.zone.today`/`Time.zone.now`.
- **Branch com migração deixa os bancos de teste à frente da main:** ao voltar para a main, `DROP DATABASE rota_saude_test_city_a` e `rota_saude_test_city_b` e rode `city:test_databases` de novo. A migração de plataforma é aditiva e fica aplicada nos bancos de plataforma de dev e de test (compartilhados); não quebra a main.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API"). Ordem de merge: `contracts` → api → dashboard → admin → maintenance.
- Crie o worktree (a partir da raiz do monorepo, `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod14 -b feat/mod-14-analytics origin/main
  cp apps/api/config/master.key apps/api/.claude/mod14/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container; o worktree é `/rails/.claude/mod14`. Todo comando Rails/RSpec roda no container, a partir da raiz do monorepo:

  ```bash
  docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec <arquivos>
  ```

- Todo `git add`/`git commit` usa `-C apps/api/.claude/mod14` (caminhos relativos ao worktree).
- Depois da migração de cidade (Task 1): `docker compose exec -T -w /rails/.claude/mod14 api bin/rails city:test_databases`. Depois da de plataforma (Task 2): `bin/rails db:migrate` em development e em test (o `db:migrate:platform` **não existe**; ver Task 2).
- Suíte completa só com o worker parado e sem outra sessão rodando suíte completa:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec
  docker compose start worker
  ```

- Prova no navegador e extração do schema GraphQL pelos outros planos: o api do worktree sobe na porta **3031** (Task 22), sem derrubar o servidor principal.

## Review Focus

1. **Revogação depois de concluir a triagem (o caminho real do wpda):** dentro da janela, a pessoa sai de `triage.completed`, `calibration.outcome` e `epi.answer` na próxima execução e aparece em `triage.aborted/revocation` sem bairro; fora da janela, o fato fica. Testes: Task 4 ("revogada depois de concluir"), Task 6, Task 8 e Task 19 (invariante 5).
2. **Soma antes de suprimir, entre dias e entre recortes:** dois dias com 3 na mesma semana mostram 6; um dia com 3 mostra "oculto"; o mesmo em `month`; taxa com numerador pequeno é oculta mesmo com denominador grande. Testes: Task 13 ("soma antes de suprimir"), Task 14 ("taxa com numerador pequeno"), Task 19 (invariante 2).
3. **Resposta que o autor não previu:** `"sim"` numa pergunta boolean, opção fora da lista, `true` gravado como JSON, pergunta `integer` com `analytic: true` gravada direto no banco — só as duas primeiras formas válidas viram fato. Testes: Task 8 ("ignora…" e "boolean gravado como JSON true").
4. **Borda do dia no fuso:** triagem às 23h30 (02h30 UTC do dia seguinte) conta no dia local; chamada à 0h10 de quem chegou às 23h50 conta no dia da chamada. Testes: Task 4 ("dia de criação no fuso") e Task 5 ("faixas").
5. **Duas consolidações ao mesmo tempo (rebuild manual durante a agendada):** uma só roda; a outra sai sem gravar `analytics_runs`; o rake diz "outra consolidação em curso". Testes: Task 10 (spec com threads), Task 11 ("outra consolidação em curso").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `db/city_migrate/20260930200001_create_analytics.rb`, `db/city_schema.rb` | fatos, runs, papel no CHECK | 1 |
| `app/models/analytics_daily_fact.rb`, `app/models/analytics_run.rb` | modelos da cidade | 1 |
| `spec/adr_pointers_spec.rb` | `VALID_RANGE` 1..25 | 1 |
| `db/platform_migrate/20260930200002_create_city_analytics_indicators.rb`, `db/platform_schema.rb`, `app/models/city_analytics_indicator.rb` | indicadores na plataforma | 2 |
| `lib/analytics_history.rb`, `spec/support/analytics_helpers.rb`, `spec/rails_helper.rb` | histórico no passado (specs e semente) | 3 |
| `app/services/analytics.rb`, `app/services/analytics/consolidate.rb`, `app/services/analytics/consolidate/base.rb` | fuso, janela, esqueleto | 4 |
| `app/services/analytics/consolidate/demand.rb` | triagens, chegadas, pedidos | 4 |
| `app/services/analytics/consolidate/quality.rb` | desfechos, espera, horários | 5 |
| `app/services/analytics/consolidate/calibration.rb` | tier × desfecho | 6 |
| `config/protocols/schema.json` | `analytic` (cópia do `contracts`) | 7 |
| `app/services/analytics/consolidate/epidemiology.rb` | respostas marcadas | 8 |
| `app/services/analytics/suppression.rb`, `app/services/analytics/publish.rb` | regra 1–4 e publicação | 9 |
| `app/services/analytics/run.rb`, `app/jobs/consolidate_analytics_job.rb`, `config/recurring.yml` | execução, trava, purga | 10 |
| `app/services/analytics/rebuild.rb`, `lib/tasks/analytics.rake` | rebuild | 11 |
| `app/models/membership.rb`, `app/services/analytics/params.rb`, `app/services/analytics/status.rb` | papel, parâmetros, estado | 12 |
| `app/controllers/admin/api/analytics_controller.rb`, `app/queries/analytics/base_query.rb`, `app/queries/analytics/demand_query.rb`, `config/routes.rb` | rota e Demanda | 13 |
| `app/queries/analytics/quality_query.rb` | Qualidade | 14 |
| `app/queries/analytics/calibration_query.rb` | Calibração | 15 |
| `app/queries/analytics/epidemiology_query.rb` | Epidemiologia | 16 |
| `app/queries/analytics/city_indicators_query.rb`, `app/controllers/operators/city_analytics_controller.rb` | console do operador | 17 |
| `app/graphql/maintenance/types/analytics_*_type.rb`, `app/graphql/maintenance/types/city_type.rb` | GraphQL | 18 |
| `spec/invariants/analytics_invariants_spec.rb` | invariantes do ADR 0025 | 19 |
| `lib/analytics_crew.rb`, `db/seeds.rb` | semente de dev | 20 |
| `docs/.claude/mod14/operacao/analytics.md`, `docs/.claude/mod14/operacao/README.md` (repo docs) | runbook | 21 |

---

## Fatia 1 — F-14.1 (dados, consolidação, publicação, rebuild)

### Task 1: Migração de cidade, modelos, papel `analyst` no CHECK e guarda de ADR (F-14.1, F-14.2)

**Files:**
- Create: `db/city_migrate/20260930200001_create_analytics.rb`
- Modify: `db/city_schema.rb`
- Create: `app/models/analytics_daily_fact.rb`, `app/models/analytics_run.rb`
- Modify: `spec/adr_pointers_spec.rb`
- Test: `spec/models/analytics_tables_guard_spec.rb`, `spec/models/create_analytics_migration_spec.rb`

**Interfaces:**
- Produces:
  - tabelas `analytics_daily_facts` (id bigint; `day`, `metric`, `health_unit_id`, `neighborhood_id`, `protocol_name`, `protocol_version`, `tier`, `question_id`, `dim` (default `''`), `value` ≥ 1, `consolidated_at`) e `analytics_runs` (uuid; `window_from`, `window_to`, `kind`, `status`, `started_at`, `finished_at`, `published_at`, `error` até 500); papel `analyst` aceito por `ck_memberships_role`.
  - `AnalyticsDailyFact::METRICS` (as 11 métricas), `AnalyticsDailyFact::WAIT_BUCKETS` (`%w[0-15 15-30 30-60 60-120 120+]`).
  - `AnalyticsRun::KINDS` (`%w[scheduled rebuild]`), `AnalyticsRun::STATUSES` (`%w[running succeeded failed]`).

- [ ] **Step 1: Confira o número da migração**

Run: `ls apps/api/.claude/mod14/db/city_migrate | tail -2`
Expected: a última é `20260930100001_guard_attendance_insert.rb`. Se houver outra mais nova, use um número maior que ela (e ajuste o nome do arquivo nos Steps 4, 8 e 10 e o `define(version:)` no Step 7).

- [ ] **Step 2: Escreva a spec de guarda (falha: tabelas não existem)**

```ruby
# spec/models/analytics_tables_guard_spec.rb
require "rails_helper"

# Módulo 14 (ADR 0025; spec 2026-09-30 §3.1–§3.2): o banco recusa fato de
# métrica desconhecida, com valor < 1, ou repetido na mesma célula — inclusive
# com recortes nulos (índice único NULLS NOT DISTINCT) —, e run com tipo,
# estado ou janela inválidos. Tudo por SQL direto, sem passar pelo modelo.
RSpec.describe "Guardas das tabelas de Analytics" do
  let(:day) { Time.zone.today - 3 }

  def fact!(**attrs)
    ApplicationRecord.transaction(requires_new: true) do
      AnalyticsDailyFact.create!({ day: day, metric: "triage.started", value: 3, consolidated_at: Time.current }.merge(attrs))
    end
  end

  it "métrica fora da lista e valor menor que 1 são recusados por CHECK" do
    expect do
      sql_in_savepoint("INSERT INTO analytics_daily_facts (day, metric, dim, value, consolidated_at) " \
                       "VALUES ('#{day.iso8601}', 'lixo', '', 3, now())")
    end.to raise_error(ActiveRecord::StatementInvalid, /ck_analytics_facts_metric/)
    expect do
      sql_in_savepoint("INSERT INTO analytics_daily_facts (day, metric, dim, value, consolidated_at) " \
                       "VALUES ('#{day.iso8601}', 'triage.started', '', 0, now())")
    end.to raise_error(ActiveRecord::StatementInvalid, /ck_analytics_facts_value/)
  end

  it "uma linha por célula, com recortes nulos tratados como iguais" do
    fact!
    expect { fact! }.to raise_error(ActiveRecord::RecordNotUnique)
    expect { fact!(neighborhood_id: SecureRandom.uuid) }.not_to raise_error
    expect { fact!(dim: "timeout", metric: "triage.aborted") }.not_to raise_error
  end

  it "run: tipo, estado e janela garantidos por CHECK" do
    run = AnalyticsRun.create!(kind: "scheduled", status: "running", window_from: day - 30, window_to: day,
                               started_at: Time.current)
    {
      "status = 'lixo'" => /ck_analytics_runs_status/,
      "kind = 'lixo'" => /ck_analytics_runs_kind/,
      "window_from = window_to + 1" => /ck_analytics_runs_window/
    }.each do |assignment, error|
      expect { sql_in_savepoint("UPDATE analytics_runs SET #{assignment} WHERE id = '#{run.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, error), assignment
    end
  end

  it "o banco aceita o papel analyst" do
    user = staff_with("analise-#{SecureRandom.hex(3)}@cidade.gov.br")
    expect do
      sql_in_savepoint("INSERT INTO memberships (id, user_id, role, granted_at, created_at, updated_at) " \
                       "VALUES (gen_random_uuid(), '#{user.id}', 'analyst', now(), now(), now())")
    end.not_to raise_error
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/models/analytics_tables_guard_spec.rb`
Expected: FAIL com `uninitialized constant AnalyticsDailyFact`.

- [ ] **Step 4: Escreva a migração**

```ruby
# db/city_migrate/20260930200001_create_analytics.rb
# Analytics (ADR 0025; spec 2026-09-30-module-14-analytics §3.1, §3.2): a
# contagem diária anônima (analytics_daily_facts), o registro de cada
# consolidação (analytics_runs) e o papel analyst no CHECK de memberships.
# Só expansão. Sem FK para unidade e bairro: o fato sobrevive à desativação
# (unidade e bairro nunca se apagam, só se desativam).
class CreateAnalytics < ActiveRecord::Migration[8.1]
  ROLES_BEFORE = %w[campaign_manager citizen_verifier health_professional municipal_admin protocol_author
                    protocol_publisher protocol_reviewer viewer].freeze
  ROLES_AFTER = %w[analyst campaign_manager citizen_verifier health_professional municipal_admin protocol_author
                   protocol_publisher protocol_reviewer viewer].freeze
  METRICS = %w[triage.started triage.completed triage.aborted attendance.checked_in attendance.closed
               attendance.wait appointment.ended request.opened request.closed calibration.outcome
               epi.answer].freeze

  def up
    replace_roles_check(ROLES_AFTER)

    create_table :analytics_daily_facts do |t|
      t.date :day, null: false
      t.string :metric, null: false
      t.uuid :health_unit_id
      t.uuid :neighborhood_id
      t.string :protocol_name
      t.integer :protocol_version
      t.string :tier
      t.string :question_id
      t.string :dim, null: false, default: ""
      t.integer :value, null: false
      t.datetime :consolidated_at, null: false
    end
    add_index :analytics_daily_facts,
              %i[day metric health_unit_id neighborhood_id protocol_name protocol_version tier question_id dim],
              unique: true, nulls_not_distinct: true, name: "idx_analytics_facts_cell"
    add_index :analytics_daily_facts, %i[metric day], name: "idx_analytics_facts_metric_day"
    add_index :analytics_daily_facts, %i[metric neighborhood_id day], name: "idx_analytics_facts_metric_neighborhood_day"
    add_index :analytics_daily_facts, %i[metric health_unit_id day], name: "idx_analytics_facts_metric_unit_day"
    add_check_constraint :analytics_daily_facts, "metric::text = ANY (ARRAY[#{quoted(METRICS)}]::text[])",
                         name: "ck_analytics_facts_metric"
    add_check_constraint :analytics_daily_facts, "value >= 1", name: "ck_analytics_facts_value"

    create_table :analytics_runs, id: :uuid do |t|
      t.date :window_from, null: false
      t.date :window_to, null: false
      t.string :kind, null: false
      t.string :status, null: false
      t.datetime :started_at, null: false
      t.datetime :finished_at
      t.datetime :published_at
      t.string :error, limit: 500
    end
    add_index :analytics_runs, :started_at, name: "idx_analytics_runs_started_at"
    add_check_constraint :analytics_runs, "kind::text = ANY (ARRAY['scheduled', 'rebuild']::text[])",
                         name: "ck_analytics_runs_kind"
    add_check_constraint :analytics_runs, "status::text = ANY (ARRAY['running', 'succeeded', 'failed']::text[])",
                         name: "ck_analytics_runs_status"
    add_check_constraint :analytics_runs, "window_from <= window_to", name: "ck_analytics_runs_window"
  end

  def down
    drop_table :analytics_runs
    drop_table :analytics_daily_facts
    # Falha se já houver membership analyst (memberships não se apagam).
    replace_roles_check(ROLES_BEFORE)
  end

  private

  def quoted(list) = list.map { |value| "'#{value}'" }.join(", ")

  # Mesma forma de 20260929100001: ANY (ARRAY[...]::text[]) sobrevive ao round-trip.
  def replace_roles_check(roles)
    remove_check_constraint :memberships, name: "ck_memberships_role"
    add_check_constraint :memberships, "role::text = ANY (ARRAY[#{quoted(roles)}]::text[])",
                         name: "ck_memberships_role"
  end
end
```

- [ ] **Step 5: Escreva os modelos**

```ruby
# app/models/analytics_daily_fact.rb
# Contagem de um dia num recorte (ADR 0025; spec 2026-09-30 §3.1). Nenhuma
# coluna de pessoa. Escrita só por Analytics::Consolidate (INSERT ... SELECT);
# a contagem é gravada crua e a supressão de 1 a 4 é da leitura, depois de
# somar período e recorte.
class AnalyticsDailyFact < ApplicationRecord
  METRICS = %w[triage.started triage.completed triage.aborted attendance.checked_in attendance.closed
               attendance.wait appointment.ended request.opened request.closed calibration.outcome
               epi.answer].freeze
  # Espera entre check-in e chamada, em minutos (desvio 12 do plano).
  WAIT_BUCKETS = %w[0-15 15-30 30-60 60-120 120+].freeze
end
```

```ruby
# app/models/analytics_run.rb
# Uma consolidação (ADR 0025; spec 2026-09-30 §3.2): a trilha do Analytics.
# `error` é classe + primeira linha da mensagem, nunca payload.
class AnalyticsRun < ApplicationRecord
  KINDS = %w[scheduled rebuild].freeze
  STATUSES = %w[running succeeded failed].freeze
end
```

- [ ] **Step 6: Suba a guarda de ADR**

Em `spec/adr_pointers_spec.rb`:
- no comentário do topo, a lista que termina em `0024 campanhas)` passa a terminar em `0024 campanhas, 0025 analytics)`, e `0001 a 0024` passa a `0001 a 0025`;
- `VALID_RANGE = (1..24).freeze` passa a `VALID_RANGE = (1..25).freeze`;
- `it "only points at ADRs that exist in the v2 corpus (0001..0024)"` passa a `(0001..0025)`.

- [ ] **Step 7: Migre os bancos de dev e faça o dump à mão**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod14 api bin/rails "city:migrate[curitiba]"
docker compose exec -T -w /rails/.claude/mod14 api bin/rails "city:migrate[maringa]"
```
Expected: `[city:migrate] curitiba → 20260930200001` (e maringa).

Em `db/city_schema.rb`:
- `define(version: 2026_09_30_100001)` passa a `define(version: 2026_09_30_200001)`;
- em `create_table "memberships"`, a linha do CHECK passa a:

```ruby
    t.check_constraint "role::text = ANY (ARRAY['analyst'::text, 'campaign_manager'::text, 'citizen_verifier'::text, 'health_professional'::text, 'municipal_admin'::text, 'protocol_author'::text, 'protocol_publisher'::text, 'protocol_reviewer'::text, 'viewer'::text])", name: "ck_memberships_role"
```

- entre `alert_recipients` e `appointment_requests` (ordem alfabética), as duas tabelas novas:

```ruby
  create_table "analytics_daily_facts", force: :cascade do |t|
    t.datetime "consolidated_at", null: false
    t.date "day", null: false
    t.string "dim", default: "", null: false
    t.uuid "health_unit_id"
    t.string "metric", null: false
    t.uuid "neighborhood_id"
    t.string "protocol_name"
    t.integer "protocol_version"
    t.string "question_id"
    t.string "tier"
    t.integer "value", null: false
    t.index ["day", "metric", "health_unit_id", "neighborhood_id", "protocol_name", "protocol_version", "tier", "question_id", "dim"], name: "idx_analytics_facts_cell", unique: true, nulls_not_distinct: true
    t.index ["metric", "day"], name: "idx_analytics_facts_metric_day"
    t.index ["metric", "health_unit_id", "day"], name: "idx_analytics_facts_metric_unit_day"
    t.index ["metric", "neighborhood_id", "day"], name: "idx_analytics_facts_metric_neighborhood_day"
    t.check_constraint "metric::text = ANY (ARRAY['triage.started', 'triage.completed', 'triage.aborted', 'attendance.checked_in', 'attendance.closed', 'attendance.wait', 'appointment.ended', 'request.opened', 'request.closed', 'calibration.outcome', 'epi.answer']::text[])", name: "ck_analytics_facts_metric"
    t.check_constraint "value >= 1", name: "ck_analytics_facts_value"
  end

  create_table "analytics_runs", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.string "error", limit: 500
    t.datetime "finished_at"
    t.string "kind", null: false
    t.datetime "published_at"
    t.datetime "started_at", null: false
    t.string "status", null: false
    t.date "window_from", null: false
    t.date "window_to", null: false
    t.index ["started_at"], name: "idx_analytics_runs_started_at"
    t.check_constraint "kind::text = ANY (ARRAY['scheduled', 'rebuild']::text[])", name: "ck_analytics_runs_kind"
    t.check_constraint "status::text = ANY (ARRAY['running', 'succeeded', 'failed']::text[])", name: "ck_analytics_runs_status"
    t.check_constraint "window_from <= window_to", name: "ck_analytics_runs_window"
  end
```

O juiz é a paridade (Step 9). Se ela acusar diferença, compare com o banco migrado —

```bash
docker compose exec -T db psql -U postgres -d <banco_de_curitiba> -c "\d+ analytics_daily_facts" -c "\d+ analytics_runs"
```

(o nome do banco sai de `bin/rails runner 'puts URI(City.find_by(slug: "curitiba").database_url).path.delete_prefix("/")'`; não cole a URL em lugar nenhum) — e ajuste o dump, nunca a migração, salvo erro de verdade nela.

- [ ] **Step 8: Escreva a spec de down e up da migração**

```ruby
# spec/models/create_analytics_migration_spec.rb
require "rails_helper"
require Rails.root.join("db/city_migrate/20260930200001_create_analytics.rb").to_s

# F-14.1: o down() da migração de cidade 20260930200001 desfaz o que o up()
# cria (duas tabelas e o papel analyst no CHECK de memberships), e o up()
# seguinte restaura o schema idêntico. O down recusa quando já existe
# membership analyst: memberships não se apagam.
#
# Mesmo desenho de create_campaigns_migration_spec.rb (módulo 12): o DDL do
# Postgres é transacional, então down e up rodam num savepoint desfeito no fim
# e o banco de teste compartilhado nunca fica alterado.
RSpec.describe "Migração de cidade 20260930200001 (CreateAnalytics): down e up" do
  let(:new_tables) { %w[analytics_daily_facts analytics_runs] }
  let(:models) { [ AnalyticsDailyFact, AnalyticsRun, Membership ] }

  def conn = ApplicationRecord.connection

  def migrate(direction)
    ActiveRecord::Migration.suppress_messages { CreateAnalytics.new.exec_migration(conn, direction) }
    models.each(&:reset_column_information)
  end

  def rows(sql) = conn.select_rows(sql)

  def fingerprint
    ignored = "('schema_migrations', 'ar_internal_metadata')"
    {
      columns: rows(<<~SQL),
        SELECT table_name, column_name, data_type, character_maximum_length::text, is_nullable, column_default
        FROM information_schema.columns
        WHERE table_schema = 'public' AND table_name NOT IN #{ignored} ORDER BY 1, 2
      SQL
      indexes: rows(<<~SQL),
        SELECT tablename, indexname, indexdef FROM pg_indexes
        WHERE schemaname = 'public' AND tablename NOT IN #{ignored} ORDER BY 1, 2
      SQL
      constraints: rows(<<~SQL)
        SELECT rel.relname, con.conname, pg_get_constraintdef(con.oid)
        FROM pg_constraint con
        JOIN pg_class rel ON rel.oid = con.conrelid
        JOIN pg_namespace ns ON ns.oid = rel.relnamespace
        WHERE ns.nspname = 'public' AND rel.relname NOT IN #{ignored} ORDER BY 1, 2
      SQL
    }
  end

  def roles_check(fp)
    fp[:constraints].find { |(table, name, _)| table == "memberships" && name == "ck_memberships_role" }&.third
  end

  def in_rolled_back_savepoint
    ApplicationRecord.transaction(requires_new: true) do
      yield
      raise ActiveRecord::Rollback
    end
  ensure
    models.each(&:reset_column_information)
  end

  it "down remove as tabelas e o papel; up seguinte restaura o schema idêntico" do
    in_rolled_back_savepoint do
      before = fingerprint
      expect(conn.tables).to include(*new_tables)
      expect(roles_check(before)).to include("'analyst'")
      expect(before[:indexes].find { |(_, name, _)| name == "idx_analytics_facts_cell" }&.third)
        .to include("NULLS NOT DISTINCT")

      migrate(:down)
      down = fingerprint

      expect(conn.tables & new_tables).to be_empty
      expect(roles_check(down)).not_to include("analyst")
      expect(roles_check(down)).to include("'campaign_manager'", "'viewer'")

      untouched = lambda do |fp|
        fp.transform_values do |list|
          list.reject { |row| row.join(" ").include?("analytics") || row.second == "ck_memberships_role" }
        end
      end
      expect(untouched.call(down)).to eq(untouched.call(before))

      migrate(:up)

      expect(fingerprint).to eq(before)
    end
  end

  it "down falha com membership analyst existente e não desfaz nada" do
    in_rolled_back_savepoint do
      staff_with("analise-#{SecureRandom.hex(3)}@cidade.gov.br").tap do |user|
        sql_in_savepoint("INSERT INTO memberships (id, user_id, role, granted_at, created_at, updated_at) " \
                         "VALUES (gen_random_uuid(), '#{user.id}', 'analyst', now(), now(), now())")
      end
      before = fingerprint

      expect do
        ApplicationRecord.transaction(requires_new: true) { migrate(:down) }
      end.to raise_error(ActiveRecord::StatementInvalid, /ck_memberships_role/)

      models.each(&:reset_column_information)
      expect(fingerprint).to eq(before)
    end
  end
end
```

(Aqui o papel entra por SQL: `Membership::ROLES` só ganha `analyst` na Task 12.)

- [ ] **Step 9: Recarregue os bancos de teste e rode paridade, guarda de ADR, guarda das tabelas e down/up**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod14 api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/city_schema_spec.rb spec/adr_pointers_spec.rb spec/models/analytics_tables_guard_spec.rb spec/models/create_analytics_migration_spec.rb
```
Expected: PASS.

- [ ] **Step 10: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add db/city_migrate/20260930200001_create_analytics.rb db/city_schema.rb app/models/analytics_daily_fact.rb app/models/analytics_run.rb spec/adr_pointers_spec.rb spec/models/analytics_tables_guard_spec.rb spec/models/create_analytics_migration_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: add analytics daily facts and runs tables with the analyst role

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 2: Migração de plataforma `city_analytics_indicators` (F-14.8)

**Files:**
- Create: `db/platform_migrate/20260930200002_create_city_analytics_indicators.rb`
- Modify: `db/platform_schema.rb` (gerado pelo `db:migrate`)
- Create: `app/models/city_analytics_indicator.rb`
- Test: `spec/models/city_analytics_indicator_spec.rb`, `spec/models/create_city_analytics_indicators_migration_spec.rb`

**Interfaces:**
- Produces:
  - tabela de plataforma `city_analytics_indicators` (uuid; `city_id` FK `cities`, `week_start` segunda-feira, `indicator`, `value` decimal(8,2) nulo quando suprimido, `suppressed`, `published_at`), única por (`city_id`, `week_start`, `indicator`).
  - `CityAnalyticsIndicator < PlatformRecord`; `CityAnalyticsIndicator::INDICATORS` (os seis, na ordem do contrato); `CityAnalyticsIndicator::COUNT_INDICATORS` (`%w[triages_started triages_completed attendances_closed]`); `belongs_to :city`.

- [ ] **Step 1: Escreva a spec do modelo (falha: tabela não existe)**

```ruby
# spec/models/city_analytics_indicator_spec.rb
require "rails_helper"

# Módulo 14 (ADR 0025; spec 2026-09-30 §3.3): o conjunto fixo da plataforma.
# O número de 1 a 4 nunca sai da cidade: suprimido é value NULL, e o banco
# recusa as duas combinações incoerentes.
RSpec.describe CityAnalyticsIndicator do
  let(:city) { create(:city) }
  let(:monday) { (Time.zone.today - 14).beginning_of_week }

  def indicator!(**attrs)
    PlatformRecord.transaction(requires_new: true) do
      described_class.create!({ city: city, week_start: monday, indicator: "triages_started", value: 12,
                                suppressed: false, published_at: Time.current }.merge(attrs))
    end
  end

  it "suprimido = value nulo, e o contrário também" do
    expect { indicator!(value: nil, suppressed: true) }.not_to raise_error
    expect { indicator!(indicator: "no_show_pct", value: 3, suppressed: true) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_city_analytics_indicators_suppressed/)
    expect { indicator!(indicator: "left_pct", value: nil, suppressed: false) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_city_analytics_indicators_suppressed/)
  end

  it "só os seis indicadores, só segunda-feira, uma linha por cidade × semana × indicador" do
    expect { indicator!(indicator: "triagens_de_tier_alto") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_city_analytics_indicators_indicator/)
    expect { indicator!(week_start: monday + 1) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_city_analytics_indicators_monday/)
    indicator!
    expect { indicator! }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "declara os seis indicadores na ordem do contrato e as três contagens" do
    expect(described_class::INDICATORS).to eq(%w[triages_started triages_completed attendances_closed
                                                 wait_within_30_pct no_show_pct left_pct])
    expect(described_class::COUNT_INDICATORS).to eq(%w[triages_started triages_completed attendances_closed])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/models/city_analytics_indicator_spec.rb`
Expected: FAIL com `uninitialized constant CityAnalyticsIndicator`.

- [ ] **Step 3: Escreva a migração e o modelo**

```ruby
# db/platform_migrate/20260930200002_create_city_analytics_indicators.rb
# Conjunto fixo de indicadores semanais da cidade inteira (ADR 0025; spec
# 2026-09-30 §3.3, §5.2), no banco de PLATAFORMA. Sem bairro, unidade,
# protocolo ou pergunta; o suprimido chega nulo — o número de 1 a 4 nunca sai
# do banco da cidade.
class CreateCityAnalyticsIndicators < ActiveRecord::Migration[8.1]
  INDICATORS = %w[triages_started triages_completed attendances_closed wait_within_30_pct no_show_pct
                  left_pct].freeze

  def change
    create_table :city_analytics_indicators, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.uuid :city_id, null: false
      t.date :week_start, null: false
      t.string :indicator, null: false
      t.decimal :value, precision: 8, scale: 2
      t.boolean :suppressed, null: false
      t.datetime :published_at, null: false
      t.index %i[city_id week_start indicator], unique: true, name: "idx_city_analytics_indicators_cell"
      t.index :week_start, name: "idx_city_analytics_indicators_week"
      t.check_constraint "indicator IN (#{INDICATORS.map { |i| "'#{i}'" }.join(', ')})",
                         name: "ck_city_analytics_indicators_indicator"
      t.check_constraint "suppressed = (value IS NULL)", name: "ck_city_analytics_indicators_suppressed"
      t.check_constraint "EXTRACT(ISODOW FROM week_start) = 1", name: "ck_city_analytics_indicators_monday"
    end
    add_foreign_key :city_analytics_indicators, :cities
  end
end
```

```ruby
# app/models/city_analytics_indicator.rb
# Indicador semanal publicado de uma cidade (ADR 0025; spec 2026-09-30 §3.3).
# Escrito só por Analytics::Publish; lido pelo console do operador e pelo
# GraphQL de manutenção, que nunca abrem o banco da cidade para isso.
class CityAnalyticsIndicator < PlatformRecord
  INDICATORS = %w[triages_started triages_completed attendances_closed wait_within_30_pct no_show_pct
                  left_pct].freeze
  COUNT_INDICATORS = %w[triages_started triages_completed attendances_closed].freeze

  belongs_to :city
end
```

- [ ] **Step 4: Migre a plataforma em development e em test**

`db:migrate:platform` não existe (só há um banco com `database_tasks` por ambiente); `db:migrate` aponta para o banco de plataforma e regrava `db/platform_schema.rb`. O banco de plataforma é compartilhado com a main: a tabela nova é aditiva e fica.

Run:
```bash
docker compose exec -T -w /rails/.claude/mod14 api bin/rails db:migrate
docker compose exec -T -w /rails/.claude/mod14 -e RAILS_ENV=test -e POSTGRES_PASSWORD=postgres api bin/rails db:migrate
/opt/homebrew/bin/git -C apps/api/.claude/mod14 diff --stat db/platform_schema.rb
```
Expected: `db/platform_schema.rb` com `define(version: 2026_09_30_200002)`, o `create_table "city_analytics_indicators"` (colunas, dois índices, três CHECKs) e `add_foreign_key "city_analytics_indicators", "cities"`. Se o diff trouxer outra mudança que não seja da tabela nova, **pare** e descubra de onde veio antes de commitar.

- [ ] **Step 5: Escreva a spec de down e up da migração de plataforma**

```ruby
# spec/models/create_city_analytics_indicators_migration_spec.rb
require "rails_helper"
require Rails.root.join("db/platform_migrate/20260930200002_create_city_analytics_indicators.rb").to_s

# F-14.8: a migração de plataforma é reversível — down remove a tabela (e a FK),
# up a restaura idêntica. Num savepoint da conexão de plataforma, desfeito no
# fim: o banco de plataforma de teste nunca fica alterado.
RSpec.describe "Migração de plataforma 20260930200002 (CreateCityAnalyticsIndicators): down e up" do
  def conn = PlatformRecord.connection

  def migrate(direction)
    ActiveRecord::Migration.suppress_messages { CreateCityAnalyticsIndicators.new.exec_migration(conn, direction) }
    CityAnalyticsIndicator.reset_column_information
  end

  def fingerprint
    ignored = "('schema_migrations', 'ar_internal_metadata')"
    {
      columns: conn.select_rows(<<~SQL),
        SELECT table_name, column_name, data_type, numeric_precision::text, numeric_scale::text, is_nullable, column_default
        FROM information_schema.columns WHERE table_schema = 'public' AND table_name NOT IN #{ignored} ORDER BY 1, 2
      SQL
      indexes: conn.select_rows(<<~SQL),
        SELECT tablename, indexname, indexdef FROM pg_indexes
        WHERE schemaname = 'public' AND tablename NOT IN #{ignored} ORDER BY 1, 2
      SQL
      constraints: conn.select_rows(<<~SQL)
        SELECT rel.relname, con.conname, pg_get_constraintdef(con.oid)
        FROM pg_constraint con JOIN pg_class rel ON rel.oid = con.conrelid
        JOIN pg_namespace ns ON ns.oid = rel.relnamespace
        WHERE ns.nspname = 'public' AND rel.relname NOT IN #{ignored} ORDER BY 1, 2
      SQL
    }
  end

  it "down remove a tabela e a FK; up seguinte restaura o schema idêntico" do
    PlatformRecord.transaction(requires_new: true) do
      before = fingerprint
      expect(conn.table_exists?(:city_analytics_indicators)).to be(true)

      migrate(:down)
      expect(conn.table_exists?(:city_analytics_indicators)).to be(false)

      migrate(:up)
      expect(fingerprint).to eq(before)
      raise ActiveRecord::Rollback
    end
  ensure
    CityAnalyticsIndicator.reset_column_information
  end
end
```

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/models/city_analytics_indicator_spec.rb spec/models/create_city_analytics_indicators_migration_spec.rb spec/architecture/database_config_parity_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add db/platform_migrate/20260930200002_create_city_analytics_indicators.rb db/platform_schema.rb app/models/city_analytics_indicator.rb spec/models/city_analytics_indicator_spec.rb spec/models/create_city_analytics_indicators_migration_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: add the platform table for published city analytics indicators

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 3: Histórico no passado e helpers de spec do Analytics (F-14.1)

**Files:**
- Create: `lib/analytics_history.rb`
- Create: `spec/support/analytics_helpers.rb`; Modify: `spec/rails_helper.rb`
- Test: `spec/lib/analytics_history_spec.rb`

**Interfaces:**
- Consumes: `CampaignHistory.cpf_for`, `CampaignHistory.citizen!`, `CampaignHistory::CONVERSATION_STATE` (`lib/campaign_history.rb`); `create_default_protocol!`, `create_unit`, `staff_with`, `register_test_city!`.
- Produces:
  - `AnalyticsHistory.citizen!(cpf:, phone:, neighborhood: nil) → Citizen`
  - `AnalyticsHistory.triage!(citizen:, protocol:, created_at:, status: "completed", tier: "alta", priority: 1, answers: {}, revoked: false) → Triage` — concluída termina 6 min depois; `revoked: true` revoga o consentimento depois de concluir (a triagem segue `completed`); `aborted_by_revocation` nasce sem respostas e sem bairro.
  - `AnalyticsHistory.attendance!(citizen:, unit:, by:, checked_in_at:, triage: nil, appointment: nil, wait_minutes: 20, outcome: "discharged", check_in_method: "code", referral_unit: nil, stage: :closed) → Attendance` — `stage` ∈ `:waiting`, `:in_care`, `:closed`; `left` fecha sem chamada, `wait_minutes` depois do check-in; os demais fecham 15 min depois da chamada.
  - `AnalyticsHistory.request!(origin:, kind:, by:, target: origin.health_unit, created_at: origin.closed_at, status: "open", closed_reason: nil, closed_at: nil, reopened_reason: nil) → AppointmentRequest`
  - `AnalyticsHistory.ended_at(status, scheduled_at) → Time` e `AnalyticsHistory.appointment!(request:, status:, scheduled_at:, by:) → Appointment` (horário já encerrado).
  - helpers de spec: `local_at(day, hour = 10, minute = 0)`, `analytics_citizen!(neighborhood = nil)`, `a_triage!(day:, hour: 10, minute: 0, neighborhood: nil, protocol: nil, **opts)`, `an_attendance!(triage:, unit:, checked_in_at: nil, **opts)`, `analytics_staff`, `analytics_definition(name:, version:, marks:)`, `analytics_protocol!(version: 1, status: "active", marks: %w[febre sintoma idade], name: "triagem-arbovirose")`, `fact!(metric:, day:, value:, **attrs)`, `consolidated_run!(finished_at: 1.hour.ago)`, `scheduled_run!` (usa `Analytics::Run`, da Task 10).

- [ ] **Step 1: Escreva os helpers de spec e registre-os**

```ruby
# spec/support/analytics_helpers.rb
require Rails.root.join("lib/analytics_history").to_s

# Módulo 14 (ADR 0025): cenários para os consolidadores, o job e as leituras.
# O cru é gravado no passado pelo AnalyticsHistory (os comandos só aceitam
# "agora"); fatos e runs prontos servem às specs de leitura.
module AnalyticsHelpers
  # Protocolo de teste com as quatro formas de pergunta: boolean e enum
  # marcadas, boolean sensível SEM marca e integer marcado — inválido pelo
  # schema, mas gravável direto no banco: o consolidador precisa ignorar
  # mesmo assim.
  def analytics_definition(name: "triagem-arbovirose", version: 1, marks: %w[febre sintoma idade])
    steps = [
      { "id" => "febre", "prompt" => "Teve febre?", "answer_type" => "boolean",
        "branches" => { "true" => "sintoma", "false" => "sintoma" }, "weights" => { "true" => 3, "false" => 0 } },
      { "id" => "sintoma", "prompt" => "Qual o sintoma mais forte?", "answer_type" => "enum",
        "options" => [ "Manchas", "Dor nas juntas", "Nenhum" ],
        "branches" => { "Manchas" => "gestante", "Dor nas juntas" => "gestante", "Nenhum" => "gestante" },
        "weights" => { "Manchas" => 3, "Dor nas juntas" => 2, "Nenhum" => 0 } },
      { "id" => "gestante", "prompt" => "Está gestante?", "answer_type" => "boolean",
        "branches" => { "true" => "idade", "false" => "idade" }, "weights" => { "true" => 2, "false" => 0 } },
      { "id" => "idade", "prompt" => "Qual a sua idade?", "answer_type" => "integer", "branches" => {} }
    ]
    {
      "name" => name, "version" => version, "start_step_id" => "febre",
      "steps" => steps.map { |step| marks.include?(step["id"]) ? step.merge("analytic" => true) : step },
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "alta" => 5 },
                     "priority_map" => { "baixa" => 9, "alta" => 1 } }
    }
  end

  def analytics_protocol!(version: 1, status: "active", marks: %w[febre sintoma idade], name: "triagem-arbovirose")
    ProtocolDefinition.create!(name: name, version: version, status: status,
                               definition: analytics_definition(name: name, version: version, marks: marks))
  end

  # Instante local (fuso da cidade) de um dia.
  def local_at(day, hour = 10, minute = 0)
    Time.zone.local(day.year, day.month, day.day, hour, minute)
  end

  def analytics_citizen!(neighborhood = nil)
    @analytics_citizen_seq = (@analytics_citizen_seq || 0) + 1
    AnalyticsHistory.citizen!(cpf: CampaignHistory.cpf_for("analytics-spec:#{@analytics_citizen_seq}:#{SecureRandom.hex(3)}"),
                              phone: format("+554195555%04d", @analytics_citizen_seq), neighborhood: neighborhood)
  end

  # Triagem de um cidadão novo, iniciada em `day` às `hour`:`minute` (hora local).
  def a_triage!(day:, hour: 10, minute: 0, neighborhood: nil, protocol: nil, **opts)
    protocol ||= ProtocolDefinition.find_by(name: StartTriage::DEFAULT_PROTOCOL_NAME, status: "active") ||
                 create_default_protocol!
    AnalyticsHistory.triage!(citizen: analytics_citizen!(neighborhood), protocol: protocol,
                             created_at: local_at(day, hour, minute), **opts)
  end

  def analytics_staff
    @analytics_staff ||= staff_with("recepcao-#{SecureRandom.hex(3)}@cidade.gov.br", "citizen_verifier")
  end

  # Atendimento da própria triagem; por padrão chega 30 min depois da conclusão.
  def an_attendance!(triage:, unit:, checked_in_at: nil, **opts)
    AnalyticsHistory.attendance!(citizen: triage.conversation.citizen, triage: triage, unit: unit, by: analytics_staff,
                                 checked_in_at: checked_in_at || triage.completed_at + 30.minutes, **opts)
  end

  def fact!(metric:, day:, value:, **attrs)
    AnalyticsDailyFact.create!({ metric: metric, day: day, value: value, dim: "", consolidated_at: Time.current }.merge(attrs))
  end

  def consolidated_run!(finished_at: 1.hour.ago)
    AnalyticsRun.create!(kind: "scheduled", status: "succeeded", window_from: Time.zone.today - 30,
                         window_to: Time.zone.today - 1, started_at: finished_at - 1.minute,
                         finished_at: finished_at, published_at: finished_at)
  end

  def scheduled_run!
    from, to = Analytics::Run.scheduled_window
    Analytics::Run.call(kind: "scheduled", from: from, to: to)
  end
end

RSpec.configure { |c| c.include AnalyticsHelpers }
```

Em `spec/rails_helper.rb`, logo depois de `require_relative "support/campaign_helpers"`:

```ruby
require_relative "support/analytics_helpers"
```

- [ ] **Step 2: Escreva a spec do histórico (falha: arquivo não existe)**

```ruby
# spec/lib/analytics_history_spec.rb
require "rails_helper"

# O histórico do Analytics passa pelos CHECKs e triggers de cada tabela (nada
# é gravado "já pronto" onde o banco exige transição): se estas specs passam,
# as linhas são as que o domínio produziria.
RSpec.describe AnalyticsHistory do
  let(:unit) { create_unit("UBS Histórico") }
  let(:day) { Time.zone.today - 5 }
  let!(:protocol) { create_default_protocol! }

  it "concluída e revogada depois: segue completed, com o consentimento revogado e a conversa revoked" do
    triage = a_triage!(day: day, revoked: true, answers: { "tosse" => "true" })
    expect(triage.reload).to have_attributes(status: "completed", completed_at: local_at(day) + 6.minutes,
                                             answers: { "tosse" => "true" })
    expect(triage.conversation.reload.state).to eq("revoked")
    expect(triage.conversation.consents.sole.revoked_at).to be_present
  end

  it "abortada por revogação nasce anonimizada, sem respostas e sem bairro" do
    centro = Neighborhood.create!(name: "Centro", source: "manual")
    triage = a_triage!(day: day, neighborhood: centro, status: "aborted_by_revocation", answers: { "tosse" => "true" })
    expect(triage.reload).to have_attributes(answers: {}, neighborhood_id: nil, tier: nil)
    expect(triage.conversation.consents.sole.revoked_at).to be_present
  end

  it "em curso: conversa ativa, sem conclusão" do
    triage = a_triage!(day: day, status: "in_progress")
    expect(triage.reload).to have_attributes(status: "in_progress", completed_at: nil, tier: nil)
    expect(triage.conversation.state).to eq("consented")
  end

  it "atendimento percorre as transições reais; quem sai não é chamado" do
    closed = an_attendance!(triage: a_triage!(day: day), unit: unit, wait_minutes: 42, outcome: "referred")
    expect(closed.reload).to have_attributes(status: "closed", outcome: "referred", referral_unit_id: unit.id)
    expect(closed.called_at - closed.checked_in_at).to eq(42 * 60)
    expect(closed.closed_at - closed.called_at).to eq(15 * 60)

    left = an_attendance!(triage: a_triage!(day: day), unit: unit, wait_minutes: 50, outcome: "left")
    expect(left.reload).to have_attributes(status: "closed", called_at: nil)
    expect(left.closed_at - left.checked_in_at).to eq(50 * 60)

    expect(an_attendance!(triage: a_triage!(day: day), unit: unit, stage: :in_care).reload.status).to eq("in_care")
    expect(an_attendance!(triage: a_triage!(day: day), unit: unit, stage: :waiting).reload.status).to eq("waiting")
  end

  it "pedido e horários encerrados, coerentes com os CHECKs" do
    origin = an_attendance!(triage: a_triage!(day: day - 10), unit: unit, outcome: "return")
    request = AnalyticsHistory.request!(origin: origin, kind: "return", by: analytics_staff, status: "closed",
                                        closed_reason: "fulfilled", closed_at: local_at(day, 9, 10))
    checked_in = AnalyticsHistory.appointment!(request: request, status: "checked_in", scheduled_at: local_at(day, 9),
                                               by: analytics_staff)
    expect(checked_in).to have_attributes(status: "checked_in", ended_at: local_at(day, 9, 10), health_unit_id: unit.id)
    %w[no_show expired cancelled_by_citizen].each do |status|
      appointment = AnalyticsHistory.appointment!(request: request, status: status, scheduled_at: local_at(day, 14),
                                                  by: analytics_staff)
      expect(appointment.ended_at).to eq(AnalyticsHistory.ended_at(status, local_at(day, 14))), status
    end
    expect(AnalyticsHistory.ended_at("no_show", local_at(day, 14)).to_date).to eq(day)
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/lib/analytics_history_spec.rb`
Expected: FAIL com `cannot load such file -- /rails/.claude/mod14/lib/analytics_history`.

- [ ] **Step 4: Escreva o histórico**

```ruby
# lib/analytics_history.rb
require_relative "campaign_history"

# Histórico no passado para o Analytics (módulo 14): specs e semente de dev.
# Mesma razão do CampaignHistory: os comandos do domínio só aceitam "agora", e
# o Analytics olha seis meses para trás. INSERT direto e UPDATE pelas
# transições reais — os CHECKs e triggers de cada tabela continuam valendo
# (attendances nasce waiting; horário e pedido não mudam depois de encerrados).
# Dá controle fino do que o Analytics lê: início, protocolo, respostas,
# espera, desfecho, fim do horário.
module AnalyticsHistory
  CONVERSATION_STATE = CampaignHistory::CONVERSATION_STATE.merge("in_progress" => "consented").freeze
  DISMISS_REASON = "Contato sem resposta após três tentativas".freeze
  CANCEL_REASON = "Não consigo ir neste horário".freeze

  module_function

  def citizen!(cpf:, phone:, neighborhood: nil)
    CampaignHistory.citizen!(cpf: cpf, phone: phone, neighborhood: neighborhood)
  end

  # Triagem web com conversa e consentimento próprios. `created_at` é o
  # início; a concluída termina 6 min depois. `revoked: true` revoga o
  # consentimento depois de concluir (caminho real do wpda: a triagem segue
  # completed). aborted_by_revocation nasce como o AnonymizeRevokedTriageJob a
  # deixa: sem respostas e sem bairro.
  def triage!(citizen:, protocol:, created_at:, status: "completed", tier: "alta", priority: 1, answers: {},
              revoked: false)
    completed = status == "completed"
    revocation = status == "aborted_by_revocation"
    conversation = Conversation.create!(channel: "web", citizen: citizen, phone: citizen.phone,
                                        state: CONVERSATION_STATE.fetch(status), created_at: created_at)
    consent = Consent.create!(conversation: conversation, version: 1, policy_text_sha: "sha-analytics-dev",
                              channel: "web", given_at: created_at)
    completed_at = if completed then created_at + 6.minutes
                   elsif revocation then created_at + 10.minutes
                   end
    triage = Triage.create!(conversation: conversation, protocol_definition: protocol, protocol_name: protocol.name,
                            status: status, tier: completed ? tier : nil, priority: completed ? priority : nil,
                            answers: revocation ? {} : answers, created_at: created_at, completed_at: completed_at,
                            neighborhood_id: revocation ? nil : citizen.neighborhood_id)
    if revocation || revoked
      consent.revoke!(at: (completed_at || created_at) + 1.hour)
      conversation.update!(state: "revoked") unless conversation.state_revoked?
    end
    triage
  end

  # Atendimento: nasce waiting (trigger attendances_born_waiting) e percorre
  # as transições. `wait_minutes` é a espera entre check-in e chamada; "left"
  # sai sem chamada, `wait_minutes` depois do check-in; os demais desfechos
  # fecham 15 min depois da chamada.
  def attendance!(citizen:, unit:, by:, checked_in_at:, triage: nil, appointment: nil, wait_minutes: 20,
                  outcome: "discharged", check_in_method: "code", referral_unit: nil, stage: :closed)
    attendance = Attendance.create!(
      triage: triage, appointment: appointment, citizen: citizen, health_unit: unit, checked_in_by_user: by,
      checked_in_at: checked_in_at, check_in_method: check_in_method,
      exception_reason: check_in_method == "cpf_exception" ? "Documento conferido no balcão" : nil
    )
    return attendance if stage == :waiting

    called_at = checked_in_at + wait_minutes.minutes
    left = outcome == "left" && stage == :closed
    attendance.update!(status: "in_care", called_by_user: by, called_at: called_at) unless left
    return attendance if stage == :in_care

    attendance.update!(status: "closed", outcome: outcome, closed_by_user: by,
                       closed_at: left ? called_at : called_at + 15.minutes,
                       referral_unit: outcome == "referred" ? (referral_unit || unit) : nil)
    attendance
  end

  # Pedido nascido do desfecho: return volta à mesma unidade; referral vai
  # para `target`. Gravado já no estado final (o trigger só guarda UPDATE).
  def request!(origin:, kind:, by:, target: origin.health_unit, created_at: origin.closed_at, status: "open",
               closed_reason: nil, closed_at: nil, reopened_reason: nil)
    dismissed = closed_reason == "dismissed"
    AppointmentRequest.create!(
      origin_attendance: origin, citizen: origin.citizen, root_triage: origin.triage, origin_unit: origin.health_unit,
      target_unit: kind == "return" ? origin.health_unit : target, kind: kind, status: status,
      closed_reason: closed_reason, closed_at: closed_at, reopened_reason: reopened_reason,
      closed_by_user: dismissed ? by : nil, dismiss_reason: dismissed ? DISMISS_REASON : nil, created_at: created_at
    )
  end

  # Quando o horário termina, por estado: comparecimento 10 min depois do
  # horário; falta no fim do dia (MarkNoShowAppointmentsJob); expiração no
  # prazo de confirmação (véspera); cancelamento dois dias antes.
  def ended_at(status, scheduled_at)
    case status
    when "checked_in" then scheduled_at + 10.minutes
    when "no_show" then scheduled_at.end_of_day
    when "expired" then scheduled_at - 1.day
    when "cancelled_by_citizen" then scheduled_at - 2.days
    else raise ArgumentError, "horário #{status} não termina"
    end
  end

  def appointment!(request:, status:, scheduled_at:, by:)
    Appointment.create!(
      request: request, citizen: request.citizen, health_unit: request.target_unit, scheduled_by_user: by,
      scheduled_at: scheduled_at, status: status, confirmation_deadline_at: scheduled_at - 1.day,
      confirmed_at: %w[checked_in no_show].include?(status) ? scheduled_at - 2.days : nil,
      cancel_reason: status == "cancelled_by_citizen" ? CANCEL_REASON : nil,
      ended_at: ended_at(status, scheduled_at), created_at: scheduled_at - 7.days
    )
  end
end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/lib/analytics_history_spec.rb spec/lib/campaign_crew_spec.rb`
Expected: PASS (o `campaign_crew_spec` prova que o `CampaignHistory` compartilhado não mudou).

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add lib/analytics_history.rb spec/support/analytics_helpers.rb spec/rails_helper.rb spec/lib/analytics_history_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "test: add backdated history writer and helpers for analytics specs

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 4: Consolidação — esqueleto e Demanda (F-14.1, F-14.3)

**Files:**
- Create: `app/services/analytics.rb`, `app/services/analytics/consolidate.rb`, `app/services/analytics/consolidate/base.rb`, `app/services/analytics/consolidate/demand.rb`
- Test: `spec/services/analytics/consolidate/demand_spec.rb`, `spec/services/analytics/consolidate_spec.rb`

**Interfaces:**
- Consumes: tabelas da Task 1; helpers da Task 3.
- Produces:
  - `Analytics::TZ` (`"America/Sao_Paulo"`); `Analytics.to_date(value) → Date`.
  - `Analytics::Consolidate.call(from:, to:, at: Time.current)` — apaga os fatos com `day` em `from..to` e chama `Analytics::Consolidate.fronts` (nesta task, `[ Demand ]`; as Tasks 5, 6 e 8 acrescentam `Quality`, `Calibration`, `Epidemiology`). Roda na transação de quem chama.
  - `Analytics::Consolidate::Base.call(from:, to:, at:)` (classe; subclasses implementam `#call`); constantes `Base::TRIAGES` (`FROM` de triagens com `pd` e a coluna `t.consent_revoked`), `Base::REVOKED` e `Base::REVOCATION` (fragmentos SQL sobre o alias `t`).
  - Métricas gravadas por `Demand`: `triage.started`, `triage.completed`, `triage.aborted`, `attendance.checked_in`, `request.opened`, `request.closed`.

- [ ] **Step 1: Escreva a spec da Demanda (falha: constante não existe)**

```ruby
# spec/services/analytics/consolidate/demand_spec.rb
require "rails_helper"

# Spec 2026-09-30 §3.4 (demanda) e desvio 1 do plano (revogação). Cada
# métrica: entra, não entra, borda do dia no fuso da cidade.
RSpec.describe Analytics::Consolidate::Demand do
  let(:day) { Time.zone.today - 3 }
  let(:centro) { Neighborhood.create!(name: "Centro", source: "manual") }
  let(:unit) { create_unit("UBS Centro") }
  let(:other_unit) { create_unit("UPA Norte", kind: "upa") }
  let!(:protocol) { create_default_protocol! }

  def run!(from = day, to = day) = described_class.call(from: from, to: to, at: Time.current)

  def rows(metric)
    AnalyticsDailyFact.where(metric: metric)
                      .pluck(:day, :health_unit_id, :neighborhood_id, :protocol_name, :protocol_version, :tier, :dim, :value)
  end

  it "triage.started: dia do início no fuso da cidade, por bairro, protocolo e versão" do
    2.times { a_triage!(day: day, hour: 9, neighborhood: centro) }
    a_triage!(day: day, hour: 23, minute: 30, neighborhood: centro) # 02h30 UTC do dia seguinte: conta em `day`
    a_triage!(day: day, hour: 11, status: "in_progress")             # sem bairro
    a_triage!(day: day + 1, hour: 0, minute: 10, neighborhood: centro)
    a_triage!(day: day - 1, hour: 23, minute: 50, neighborhood: centro)

    run!

    expect(rows("triage.started")).to contain_exactly(
      [ day, nil, centro.id, protocol.name, 1, nil, "", 3 ],
      [ day, nil, nil, protocol.name, 1, nil, "", 1 ]
    )
  end

  it "triage.completed: dia da conclusão, com tier; revogada depois de concluir não entra" do
    a_triage!(day: day, neighborhood: centro, tier: "alta")
    a_triage!(day: day, neighborhood: centro, tier: "baixa")
    a_triage!(day: day, neighborhood: centro, tier: "alta", revoked: true)
    a_triage!(day: day, hour: 23, minute: 57, neighborhood: centro) # conclui 00h03 do dia seguinte

    run!

    expect(rows("triage.completed")).to contain_exactly(
      [ day, nil, centro.id, protocol.name, 1, "alta", "", 1 ],
      [ day, nil, centro.id, protocol.name, 1, "baixa", "", 1 ]
    )
  end

  it "triage.aborted: tempo esgotado, cancelamento e revogação; a revogação perde o bairro" do
    a_triage!(day: day, neighborhood: centro, status: "aborted_by_timeout")
    a_triage!(day: day, neighborhood: centro, status: "aborted_by_cancellation")
    a_triage!(day: day, neighborhood: centro, status: "aborted_by_revocation")
    a_triage!(day: day, neighborhood: centro, revoked: true) # revogada depois de concluir

    run!

    expect(rows("triage.aborted")).to contain_exactly(
      [ day, nil, centro.id, protocol.name, 1, nil, "timeout", 1 ],
      [ day, nil, centro.id, protocol.name, 1, nil, "cancellation", 1 ],
      [ day, nil, nil, protocol.name, 1, nil, "revocation", 2 ]
    )
    expect(rows("triage.started")).to contain_exactly(
      [ day, nil, centro.id, protocol.name, 1, nil, "", 2 ],
      [ day, nil, nil, protocol.name, 1, nil, "", 2 ]
    )
  end

  it "attendance.checked_in: por unidade e método, no dia do check-in" do
    late = a_triage!(day: day - 1, hour: 22)
    an_attendance!(triage: late, unit: unit, checked_in_at: local_at(day, 0, 20))
    an_attendance!(triage: a_triage!(day: day), unit: unit, check_in_method: "cpf_exception")
    an_attendance!(triage: a_triage!(day: day), unit: other_unit, stage: :waiting)
    an_attendance!(triage: a_triage!(day: day - 1), unit: unit) # chega em day - 1: fora

    run!

    expect(rows("attendance.checked_in")).to contain_exactly(
      [ day, unit.id, nil, nil, nil, nil, "code", 1 ],
      [ day, unit.id, nil, nil, nil, nil, "cpf_exception", 1 ],
      [ day, other_unit.id, nil, nil, nil, nil, "code", 1 ]
    )
  end

  it "request.opened e request.closed: pela unidade-alvo, por tipo e por motivo" do
    back = an_attendance!(triage: a_triage!(day: day - 2), unit: unit, outcome: "return")
    sent = an_attendance!(triage: a_triage!(day: day - 2), unit: unit, outcome: "referred", referral_unit: other_unit)
    AnalyticsHistory.request!(origin: back, kind: "return", by: analytics_staff, created_at: local_at(day, 9))
    AnalyticsHistory.request!(origin: sent, kind: "referral", target: other_unit, by: analytics_staff,
                              created_at: local_at(day - 2, 16), status: "closed", closed_reason: "dismissed",
                              closed_at: local_at(day, 15))

    run!

    expect(rows("request.opened")).to contain_exactly([ day, unit.id, nil, nil, nil, nil, "return", 1 ])
    expect(rows("request.closed")).to contain_exactly([ day, other_unit.id, nil, nil, nil, nil, "dismissed", 1 ])
  end
end
```

```ruby
# spec/services/analytics/consolidate_spec.rb
require "rails_helper"

# Spec §4.1: a consolidação apaga só os fatos da janela e os regrava do cru.
RSpec.describe Analytics::Consolidate do
  let(:day) { Time.zone.today - 3 }
  let!(:protocol) { create_default_protocol! }

  it "apaga os fatos da janela (e só eles) antes de regravar" do
    outside = fact!(metric: "triage.started", day: day - 1, value: 7)
    fact!(metric: "triage.started", day: day, value: 99, protocol_name: "fantasma")
    a_triage!(day: day)

    described_class.call(from: day, to: day)

    expect(AnalyticsDailyFact.where(day: day).pluck(:metric, :protocol_name, :value))
      .to include([ "triage.started", protocol.name, 1 ])
    expect(AnalyticsDailyFact.where(protocol_name: "fantasma")).to be_empty
    expect(outside.reload.value).to eq(7)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics`
Expected: FAIL com `uninitialized constant Analytics`.

- [ ] **Step 3: Escreva o namespace, o esqueleto e a Demanda**

```ruby
# app/services/analytics.rb
# Analytics (ADR 0025): consolidação diária anônima por cidade, leitura
# suprimida e publicação do conjunto fixo na plataforma.
module Analytics
  # Fuso fixo do "dia" (api#27; spec 2026-09-30 §14): o mesmo de config.time_zone.
  TZ = "America/Sao_Paulo"

  def self.to_date(value) = value.is_a?(Date) ? value : Date.iso8601(value.to_s)
end
```

```ruby
# app/services/analytics/consolidate.rb
module Analytics
  # Refaz os fatos da janela [from, to] a partir do cru (spec §4.1): apaga os
  # dias da janela e regrava, um consolidador por frente. Roda na transação de
  # quem chama (Analytics::Run) — sozinho, uma falha no meio deixaria a janela
  # pela metade.
  module Consolidate
    def self.fronts = [ Demand ]

    def self.call(from:, to:, at: Time.current)
      AnalyticsDailyFact.where(day: from..to).delete_all
      fronts.each { |front| front.call(from: from, to: to, at: at) }
    end
  end
end
```

```ruby
# app/services/analytics/consolidate/base.rb
module Analytics
  module Consolidate
    # Cada métrica é um INSERT ... SELECT ... GROUP BY sobre o cru da janela
    # (dias no fuso da cidade), direto no SQL: nenhuma linha de pessoa passa
    # pelo Ruby, e só a contagem sai de cada grupo.
    class Base
      COLUMNS = "day, metric, health_unit_id, neighborhood_id, protocol_name, protocol_version, tier, " \
                "question_id, dim, value, consolidated_at"

      # Triagens com a versão do protocolo (`pd`) e a marca de consentimento
      # revogado já calculada numa coluna (`t.consent_revoked`): assim os CASE
      # que agrupam por ela não carregam subconsulta correlacionada no GROUP BY.
      # O filtro de janela aplicado em `t` desce para dentro da subconsulta.
      TRIAGES = <<~SQL.squish
        FROM (
          SELECT tr.*, EXISTS (SELECT 1 FROM consents c
                               WHERE c.conversation_id = tr.conversation_id AND c.revoked_at IS NOT NULL) AS consent_revoked
          FROM triages tr
        ) t
        JOIN protocol_definitions pd ON pd.id = t.protocol_definition_id
      SQL
      # Triagem revogada (desvio 1 do plano): abortada por revogação, ou com o
      # consentimento da PRÓPRIA conversa revogado — o wpda revoga depois de
      # concluir, e a triagem segue completed. Sobre o alias `t` de TRIAGES.
      REVOKED = "t.consent_revoked"
      REVOCATION = "(t.status = 'aborted_by_revocation' OR t.consent_revoked)"

      def self.call(from:, to:, at:) = new(from: from, to: to, at: at).call

      def initialize(from:, to:, at:)
        @binds = { lower: from.in_time_zone.utc, upper: (to + 1).in_time_zone.utc, at: at, tz: Analytics::TZ }
      end

      private

      def insert!(select_sql)
        ApplicationRecord.connection.execute(
          ApplicationRecord.sanitize_sql_array([ "INSERT INTO analytics_daily_facts (#{COLUMNS}) #{select_sql}", @binds ])
        )
      end

      # As colunas datetime são `timestamp without time zone` em UTC: rotula
      # como UTC e só então leva ao relógio local (mesma lição de
      # Admin::Api::Period#group_expr).
      def day(column) = "((#{column} AT TIME ZONE 'UTC') AT TIME ZONE :tz)::date"

      def window(column) = "#{column} >= :lower AND #{column} < :upper"
    end
  end
end
```

```ruby
# app/services/analytics/consolidate/demand.rb
module Analytics
  module Consolidate
    # Demanda por território (spec §3.4): triagens por bairro, protocolo e
    # versão (e tier na concluída), chegadas por unidade e pedidos pela
    # unidade-alvo. Atendimento não tem bairro: o território é o da triagem.
    class Demand < Base
      # Triagem revogada perde o território no Analytics (desvio 1).
      TERRITORY = "CASE WHEN #{REVOCATION} THEN NULL ELSE t.neighborhood_id END"

      def call
        triages_started
        triages_completed
        triages_aborted
        attendances_checked_in
        requests_opened
        requests_closed
      end

      private

      def triages_started
        insert!(<<~SQL)
          SELECT #{day('t.created_at')}, 'triage.started', NULL, #{TERRITORY}, t.protocol_name, pd.version,
                 NULL, NULL, '', COUNT(*), :at
          #{TRIAGES}
          WHERE #{window('t.created_at')}
          GROUP BY 1, 4, 5, 6
        SQL
      end

      def triages_completed
        insert!(<<~SQL)
          SELECT #{day('t.completed_at')}, 'triage.completed', NULL, t.neighborhood_id, t.protocol_name, pd.version,
                 t.tier, NULL, '', COUNT(*), :at
          #{TRIAGES}
          WHERE t.status = 'completed' AND NOT #{REVOKED} AND #{window('t.completed_at')}
          GROUP BY 1, 4, 5, 6, 7
        SQL
      end

      def triages_aborted
        insert!(<<~SQL)
          SELECT #{day('t.created_at')}, 'triage.aborted', NULL, #{TERRITORY}, t.protocol_name, pd.version,
                 NULL, NULL,
                 CASE WHEN #{REVOCATION} THEN 'revocation'
                      WHEN t.status = 'aborted_by_timeout' THEN 'timeout'
                      ELSE 'cancellation' END,
                 COUNT(*), :at
          #{TRIAGES}
          WHERE (t.status IN ('aborted_by_timeout', 'aborted_by_cancellation', 'aborted_by_revocation')
                 OR (t.status = 'completed' AND #{REVOKED}))
            AND #{window('t.created_at')}
          GROUP BY 1, 4, 5, 6, 9
        SQL
      end

      def attendances_checked_in
        insert!(<<~SQL)
          SELECT #{day('a.checked_in_at')}, 'attendance.checked_in', a.health_unit_id, NULL, NULL, NULL,
                 NULL, NULL, a.check_in_method, COUNT(*), :at
          FROM attendances a
          WHERE #{window('a.checked_in_at')}
          GROUP BY 1, 3, 9
        SQL
      end

      def requests_opened
        insert!(<<~SQL)
          SELECT #{day('r.created_at')}, 'request.opened', r.target_unit_id, NULL, NULL, NULL,
                 NULL, NULL, r.kind, COUNT(*), :at
          FROM appointment_requests r
          WHERE #{window('r.created_at')}
          GROUP BY 1, 3, 9
        SQL
      end

      def requests_closed
        insert!(<<~SQL)
          SELECT #{day('r.closed_at')}, 'request.closed', r.target_unit_id, NULL, NULL, NULL,
                 NULL, NULL, r.closed_reason, COUNT(*), :at
          FROM appointment_requests r
          WHERE r.closed_at IS NOT NULL AND #{window('r.closed_at')}
          GROUP BY 1, 3, 9
        SQL
      end
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/services/analytics.rb app/services/analytics/consolidate.rb app/services/analytics/consolidate/base.rb app/services/analytics/consolidate/demand.rb spec/services/analytics/consolidate/demand_spec.rb spec/services/analytics/consolidate_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: consolidate daily demand facts per city in SQL

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 5: Consolidação — Qualidade (F-14.4)

**Files:**
- Create: `app/services/analytics/consolidate/quality.rb`
- Modify: `app/services/analytics/consolidate.rb`
- Test: `spec/services/analytics/consolidate/quality_spec.rb`

**Interfaces:**
- Consumes: `Analytics::Consolidate::Base` (Task 4); `AnalyticsHistory` (Task 3).
- Produces: `Analytics::Consolidate::Quality` com as métricas `attendance.closed` (dim = desfecho), `attendance.wait` (dim = faixa de `AnalyticsDailyFact::WAIT_BUCKETS`, dia da chamada), `appointment.ended` (dim = estado final; dia de `ended_at`). `Analytics::Consolidate.fronts` passa a `[ Demand, Quality ]`.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/services/analytics/consolidate/quality_spec.rb
require "rails_helper"

# Spec §3.4 (qualidade operacional) e desvio 12 do plano (faixas de espera).
RSpec.describe Analytics::Consolidate::Quality do
  let(:day) { Time.zone.today - 3 }
  let(:unit) { create_unit("UBS Centro") }
  let(:other_unit) { create_unit("UPA Norte", kind: "upa") }
  let!(:protocol) { create_default_protocol! }

  def run!(from = day, to = day) = described_class.call(from: from, to: to, at: Time.current)
  def rows(metric) = AnalyticsDailyFact.where(metric: metric).pluck(:day, :health_unit_id, :dim, :value)

  it "attendance.closed: dia do encerramento, por unidade e desfecho; em atendimento não entra" do
    2.times { an_attendance!(triage: a_triage!(day: day), unit: unit) }
    an_attendance!(triage: a_triage!(day: day), unit: unit, outcome: "left")
    an_attendance!(triage: a_triage!(day: day), unit: other_unit, outcome: "referred", referral_unit: unit)
    an_attendance!(triage: a_triage!(day: day), unit: unit, stage: :in_care)

    run!

    expect(rows("attendance.closed")).to contain_exactly(
      [ day, unit.id, "discharged", 2 ], [ day, unit.id, "left", 1 ], [ day, other_unit.id, "referred", 1 ]
    )
  end

  it "attendance.wait: faixa pela espera até a chamada, no dia da chamada; quem saiu sem chamada não entra" do
    [ 14, 15, 29, 30, 59, 60, 119, 120 ].each do |minutes|
      an_attendance!(triage: a_triage!(day: day, hour: 8), unit: unit, wait_minutes: minutes)
    end
    night = a_triage!(day: day - 1, hour: 23)
    an_attendance!(triage: night, unit: unit, checked_in_at: local_at(day - 1, 23, 50), wait_minutes: 20) # chamada 00h10
    an_attendance!(triage: a_triage!(day: day), unit: unit, outcome: "left", wait_minutes: 5)

    run!

    expect(rows("attendance.wait")).to contain_exactly(
      [ day, unit.id, "0-15", 1 ], [ day, unit.id, "15-30", 3 ], [ day, unit.id, "30-60", 2 ],
      [ day, unit.id, "60-120", 2 ], [ day, unit.id, "120+", 1 ]
    )
  end

  it "appointment.ended: por unidade e estado final, no dia do fim; horário vivo não entra" do
    origin = an_attendance!(triage: a_triage!(day: day - 20), unit: unit, outcome: "return")
    request = AnalyticsHistory.request!(origin: origin, kind: "return", by: analytics_staff)
    AnalyticsHistory.appointment!(request: request, status: "checked_in", scheduled_at: local_at(day, 9), by: analytics_staff)
    AnalyticsHistory.appointment!(request: request, status: "no_show", scheduled_at: local_at(day, 10), by: analytics_staff)
    AnalyticsHistory.appointment!(request: request, status: "expired", scheduled_at: local_at(day + 1, 9), by: analytics_staff)
    AnalyticsHistory.appointment!(request: request, status: "cancelled_by_citizen", scheduled_at: local_at(day + 2, 9),
                                  by: analytics_staff)
    Appointment.create!(request: request, citizen: request.citizen, health_unit: unit, scheduled_by_user: analytics_staff,
                        scheduled_at: local_at(day, 11), status: "scheduled", confirmation_deadline_at: local_at(day, 8))

    run!

    expect(rows("appointment.ended")).to contain_exactly(
      [ day, unit.id, "checked_in", 1 ], [ day, unit.id, "no_show", 1 ],
      [ day, unit.id, "expired", 1 ], [ day, unit.id, "cancelled_by_citizen", 1 ]
    )
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics/consolidate/quality_spec.rb`
Expected: FAIL com `uninitialized constant Analytics::Consolidate::Quality`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/analytics/consolidate/quality.rb
module Analytics
  module Consolidate
    # Qualidade operacional (spec §3.4): desfechos por unidade, espera em
    # faixas (D11: faixas somam entre dias e recortes; mediana não) e fim dos
    # horários. Faixas: desvio 12 do plano.
    class Quality < Base
      WAIT = "a.called_at - a.checked_in_at"

      def call
        attendances_closed
        attendances_wait
        appointments_ended
      end

      private

      def attendances_closed
        insert!(<<~SQL)
          SELECT #{day('a.closed_at')}, 'attendance.closed', a.health_unit_id, NULL, NULL, NULL,
                 NULL, NULL, a.outcome, COUNT(*), :at
          FROM attendances a
          WHERE a.status = 'closed' AND #{window('a.closed_at')}
          GROUP BY 1, 3, 9
        SQL
      end

      def attendances_wait
        insert!(<<~SQL)
          SELECT #{day('a.called_at')}, 'attendance.wait', a.health_unit_id, NULL, NULL, NULL, NULL, NULL,
                 CASE WHEN #{WAIT} < interval '15 minutes' THEN '0-15'
                      WHEN #{WAIT} < interval '30 minutes' THEN '15-30'
                      WHEN #{WAIT} < interval '60 minutes' THEN '30-60'
                      WHEN #{WAIT} < interval '120 minutes' THEN '60-120'
                      ELSE '120+' END,
                 COUNT(*), :at
          FROM attendances a
          WHERE a.called_at IS NOT NULL AND #{window('a.called_at')}
          GROUP BY 1, 3, 9
        SQL
      end

      def appointments_ended
        insert!(<<~SQL)
          SELECT #{day('ap.ended_at')}, 'appointment.ended', ap.health_unit_id, NULL, NULL, NULL,
                 NULL, NULL, ap.status, COUNT(*), :at
          FROM appointments ap
          WHERE ap.ended_at IS NOT NULL
            AND ap.status IN ('checked_in', 'no_show', 'expired', 'cancelled_by_citizen')
            AND #{window('ap.ended_at')}
          GROUP BY 1, 3, 9
        SQL
      end
    end
  end
end
```

Em `app/services/analytics/consolidate.rb`, `def self.fronts = [ Demand ]` passa a `def self.fronts = [ Demand, Quality ]`.

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/services/analytics/consolidate/quality.rb app/services/analytics/consolidate.rb spec/services/analytics/consolidate/quality_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: consolidate outcomes, wait buckets and ended appointments per unit

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 6: Consolidação — Calibração (F-14.5)

**Files:**
- Create: `app/services/analytics/consolidate/calibration.rb`
- Modify: `app/services/analytics/consolidate.rb`
- Test: `spec/services/analytics/consolidate/calibration_spec.rb`

**Interfaces:**
- Consumes: `Base` (Task 4).
- Produces: `Analytics::Consolidate::Calibration` com a métrica `calibration.outcome` (dia da conclusão da triagem; protocolo, versão, tier; dim = desfecho do atendimento **encerrado** da própria triagem, ou `none`). `fronts` passa a `[ Demand, Quality, Calibration ]`.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/services/analytics/consolidate/calibration_spec.rb
require "rails_helper"

# Spec §3.4 (calibração) e desvios 1 e 2 do plano: um atendimento por triagem
# (triage_id é único); o atendimento do retorno, que nasce do horário, não conta.
RSpec.describe Analytics::Consolidate::Calibration do
  let(:day) { Time.zone.today - 3 }
  let(:unit) { create_unit("UBS Centro") }
  let!(:protocol) { create_default_protocol! }

  def run!(from = day, to = day) = described_class.call(from: from, to: to, at: Time.current)

  def rows
    AnalyticsDailyFact.where(metric: "calibration.outcome").pluck(:day, :protocol_name, :protocol_version, :tier, :dim, :value)
  end

  it "sem atendimento é none; com atendimento encerrado, o desfecho; em atendimento ainda é none; revogada não entra" do
    a_triage!(day: day, tier: "alta")
    an_attendance!(triage: a_triage!(day: day, tier: "alta"), unit: unit, outcome: "discharged")
    an_attendance!(triage: a_triage!(day: day, tier: "baixa"), unit: unit, stage: :in_care)
    an_attendance!(triage: a_triage!(day: day, tier: "alta", revoked: true), unit: unit, outcome: "left")
    a_triage!(day: day, status: "aborted_by_timeout")

    run!

    expect(rows).to contain_exactly(
      [ day, protocol.name, 1, "alta", "none", 1 ],
      [ day, protocol.name, 1, "alta", "discharged", 1 ],
      [ day, protocol.name, 1, "baixa", "none", 1 ]
    )
  end

  it "vale o atendimento da própria triagem, não o do retorno que veio do horário" do
    triage = a_triage!(day: day, tier: "alta")
    first = an_attendance!(triage: triage, unit: unit, outcome: "return")
    request = AnalyticsHistory.request!(origin: first, kind: "return", by: analytics_staff)
    appointment = AnalyticsHistory.appointment!(request: request, status: "checked_in",
                                                scheduled_at: local_at(day + 1, 9), by: analytics_staff)
    AnalyticsHistory.attendance!(citizen: request.citizen, appointment: appointment, unit: unit, by: analytics_staff,
                                 checked_in_at: appointment.ended_at, outcome: "referred", referral_unit: unit)

    run!

    expect(rows).to contain_exactly([ day, protocol.name, 1, "alta", "return", 1 ])
  end

  it "versões diferentes do mesmo protocolo ficam em linhas separadas" do
    v2 = ProtocolDefinition.create!(name: protocol.name, version: 2, status: "published", definition: protocol.definition.merge("version" => 2))
    a_triage!(day: day, tier: "alta")
    a_triage!(day: day, tier: "alta", protocol: v2)

    run!

    expect(rows).to contain_exactly([ day, protocol.name, 1, "alta", "none", 1 ], [ day, protocol.name, 2, "alta", "none", 1 ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics/consolidate/calibration_spec.rb`
Expected: FAIL com `uninitialized constant Analytics::Consolidate::Calibration`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/analytics/consolidate/calibration.rb
module Analytics
  module Consolidate
    # Calibração de protocolo (spec §3.4): versão × tier × desfecho do
    # atendimento da própria triagem (desvio 2 do plano). Por versão porque
    # `tier` é texto de cada protocolo (D13): renomear a faixa separa as séries.
    # Desfecho que chega depois de 30 dias não entra (D10).
    class Calibration < Base
      def call
        insert!(<<~SQL)
          SELECT #{day('t.completed_at')}, 'calibration.outcome', NULL, NULL, t.protocol_name, pd.version,
                 t.tier, NULL, COALESCE(a.outcome, 'none'), COUNT(*), :at
          #{TRIAGES}
          LEFT JOIN attendances a ON a.triage_id = t.id AND a.status = 'closed'
          WHERE t.status = 'completed' AND NOT #{REVOKED} AND #{window('t.completed_at')}
          GROUP BY 1, 5, 6, 7, 9
        SQL
      end
    end
  end
end
```

Em `app/services/analytics/consolidate.rb`, `fronts` passa a `[ Demand, Quality, Calibration ]`.

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/services/analytics/consolidate/calibration.rb app/services/analytics/consolidate.rb spec/services/analytics/consolidate/calibration_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: consolidate protocol calibration by version, tier and outcome

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 7: `analytic` no schema de protocolo (F-14.6)

**Files:**
- Modify: `config/protocols/schema.json` (cópia de `contracts/protocols/schema.json`, `protocols-v1.2.0`)
- Test: `spec/protocols/schema_analytic_spec.rb`

**Interfaces:**
- Consumes: plano `2026-09-30-module-14-analytics-contracts-repo.md` (Task 1 dele) — o arquivo de lá é a fonte.
- Produces: pergunta (`$defs/step`) aceita `"analytic": true|false`; `analytic: true` só com `answer_type` `boolean` ou `enum` — `Protocols::Gate` (enviar para revisão e publicar) recusa o resto. Rascunho continua sem o portão completo (`Protocols::SaveDraft`), como hoje.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/protocols/schema_analytic_spec.rb
require "rails_helper"
require "json_schemer"

# F-14.6 (ADR 0025; spec 2026-09-30 §7): a marca `analytic` é da pergunta, só
# em boolean/enum, e o portão do ciclo assinado recusa o resto.
RSpec.describe "protocols schema.json analytic contract (F-14.6)" do
  let(:schema) { JSONSchemer.schema(JSON.parse(File.read(Rails.root.join("config/protocols/schema.json")))) }

  def definition(step)
    {
      "name" => "arbo-contract", "version" => 1, "start_step_id" => "s1",
      "steps" => [ { "id" => "s1", "prompt" => "?", "branches" => {}, "weights" => {} }.merge(step) ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 9 } }
    }
  end

  it "aceita analytic em boolean e em enum, e analytic: false em qualquer tipo" do
    expect(schema.valid?(definition("answer_type" => "boolean", "analytic" => true))).to be(true)
    expect(schema.valid?(definition("answer_type" => "enum", "options" => %w[a b], "analytic" => true))).to be(true)
    expect(schema.valid?(definition("answer_type" => "integer", "analytic" => false))).to be(true)
    expect(schema.valid?(definition("answer_type" => "text"))).to be(true)
  end

  it "recusa analytic: true em integer e em text, e analytic que não é booleano" do
    expect(schema.valid?(definition("answer_type" => "integer", "analytic" => true))).to be(false)
    expect(schema.valid?(definition("answer_type" => "text", "analytic" => true))).to be(false)
    expect(schema.valid?(definition("answer_type" => "boolean", "analytic" => "sim"))).to be(false)
  end

  it "o portão do ciclo assinado aponta a pergunta marcada errada" do
    errors = Protocols::Gate.call(definition("answer_type" => "integer", "analytic" => true)).errors
    expect(errors).to include(a_string_including("/steps/0/answer_type"))
    expect(Protocols::Gate.call(definition("answer_type" => "boolean", "analytic" => true,
                                           "branches" => { "true" => nil, "false" => nil })).errors).to be_empty
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/protocols/schema_analytic_spec.rb`
Expected: FAIL — o primeiro exemplo recusa `analytic` (`additionalProperties: false` em `$defs/step`).

- [ ] **Step 3: Copie o schema do `contracts`**

Com o plano do `contracts` já executado (branch `feat/protocols-analytic`, worktree `contracts/.claude/mod14`), a partir da raiz do monorepo:

```bash
cp contracts/.claude/mod14/protocols/schema.json apps/api/.claude/mod14/config/protocols/schema.json
/opt/homebrew/bin/git -C apps/api/.claude/mod14 diff config/protocols/schema.json
```

Expected: o diff é exatamente a troca abaixo em `$defs/step` (se o `contracts` ainda não foi executado, faça esta mesma edição à mão e confira com `diff` quando ele estiver pronto — os dois arquivos precisam ficar idênticos):

Antes:
```json
        "weights": {
          "type": "object",
          "additionalProperties": { "type": "integer" },
          "description": "answer_value -> peso numérico (usado por scoring weighted)"
        }
      }
    },
    "scoring_weighted": {
```

Depois:
```json
        "weights": {
          "type": "object",
          "additionalProperties": { "type": "integer" },
          "description": "answer_value -> peso numérico (usado por scoring weighted)"
        },
        "analytic": {
          "type": "boolean",
          "description": "Usar em Analytics (ADR 0025): respostas agregadas por bairro, nunca por pessoa. Só boolean e enum."
        }
      },
      "if": {
        "required": ["analytic"],
        "properties": { "analytic": { "const": true } }
      },
      "then": {
        "properties": { "answer_type": { "enum": ["boolean", "enum"] } }
      }
    },
    "scoring_weighted": {
```

- [ ] **Step 4: Rode e veja passar, com as specs de schema existentes**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/protocols`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add config/protocols/schema.json spec/protocols/schema_analytic_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: accept the analytic flag on boolean and enum protocol questions

Copied from contracts protocols-v1.2.0.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 8: Consolidação — Epidemiologia (F-14.7)

**Files:**
- Create: `app/services/analytics/consolidate/epidemiology.rb`
- Modify: `app/services/analytics/consolidate.rb`
- Test: `spec/services/analytics/consolidate/epidemiology_spec.rb`

**Interfaces:**
- Consumes: `Base` (Task 4); `analytics_protocol!`/`analytics_definition` (Task 3).
- Produces: `Analytics::Consolidate::Epidemiology` com a métrica `epi.answer` (dia da conclusão; bairro, protocolo, versão, `question_id`; dim = a opção respondida: `"true"`/`"false"` ou a string da opção do `enum`). `fronts` passa a `[ Demand, Quality, Calibration, Epidemiology ]` — completo.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/services/analytics/consolidate/epidemiology_spec.rb
require "rails_helper"

# Spec §3.4 (epi.answer) e D5: só perguntas marcadas `analytic` NA VERSÃO da
# triagem, só boolean/enum, só respostas que batem com uma opção declarada.
RSpec.describe Analytics::Consolidate::Epidemiology do
  let(:day) { Time.zone.today - 3 }
  let(:centro) { Neighborhood.create!(name: "Centro", source: "manual") }
  let!(:v1) { analytics_protocol!(version: 1, status: "retired", marks: %w[febre]) }
  let!(:v2) { analytics_protocol!(version: 2, status: "active") } # febre, sintoma e idade (integer) marcadas

  def run!(from = day, to = day) = described_class.call(from: from, to: to, at: Time.current)

  def rows
    AnalyticsDailyFact.where(metric: "epi.answer")
                      .pluck(:day, :neighborhood_id, :protocol_name, :protocol_version, :question_id, :dim, :value)
  end

  it "conta respostas marcadas de boolean e enum por bairro, protocolo, versão, pergunta e opção" do
    a_triage!(day: day, protocol: v2, neighborhood: centro,
              answers: { "febre" => "true", "sintoma" => "Manchas", "gestante" => "true", "idade" => "34" })
    a_triage!(day: day, protocol: v2, neighborhood: centro,
              answers: { "febre" => "true", "sintoma" => "Nenhum", "gestante" => "false", "idade" => "51" })
    a_triage!(day: day, protocol: v2, answers: { "febre" => "false", "sintoma" => "Manchas" })

    run!

    expect(rows).to contain_exactly(
      [ day, centro.id, v2.name, 2, "febre", "true", 2 ],
      [ day, centro.id, v2.name, 2, "sintoma", "Manchas", 1 ],
      [ day, centro.id, v2.name, 2, "sintoma", "Nenhum", 1 ],
      [ day, nil, v2.name, 2, "febre", "false", 1 ],
      [ day, nil, v2.name, 2, "sintoma", "Manchas", 1 ]
    )
  end

  it "a marca é da versão da triagem: na v1 só febre conta" do
    a_triage!(day: day, protocol: v1, answers: { "febre" => "true", "sintoma" => "Manchas" })

    run!

    expect(rows).to contain_exactly([ day, nil, v1.name, 1, "febre", "true", 1 ])
  end

  it "ignora pergunta sem marca, integer marcado à mão, resposta fora das opções e resposta em formato estranho" do
    a_triage!(day: day, protocol: v2,
              answers: { "febre" => "sim", "sintoma" => "Febre alta", "gestante" => "true", "idade" => "34" })

    run!

    expect(rows).to be_empty
  end

  it "boolean gravado como JSON true conta como \"true\"" do
    a_triage!(day: day, protocol: v2, answers: { "febre" => true })

    run!

    expect(rows).to contain_exactly([ day, nil, v2.name, 2, "febre", "true", 1 ])
  end

  it "triagem não concluída ou revogada não entra" do
    a_triage!(day: day, protocol: v2, status: "aborted_by_timeout", answers: { "febre" => "true" })
    a_triage!(day: day, protocol: v2, revoked: true, answers: { "febre" => "true" })

    run!

    expect(rows).to be_empty
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics/consolidate/epidemiology_spec.rb`
Expected: FAIL com `uninitialized constant Analytics::Consolidate::Epidemiology`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/analytics/consolidate/epidemiology.rb
module Analytics
  module Consolidate
    # Epidemiologia (spec §3.4, D5): respostas das perguntas que a VERSÃO da
    # triagem marca `analytic`, só boolean/enum, e só a resposta que bate com
    # uma opção declarada. Pergunta sem marca, integer, text e resposta fora da
    # lista nunca viram agregado — mesmo numa definição gravada direto no
    # banco, sem passar pelo schema.
    class Epidemiology < Base
      ANSWER = "t.answers ->> (s.step ->> 'id')"

      def call
        insert!(<<~SQL)
          SELECT #{day('t.completed_at')}, 'epi.answer', NULL, t.neighborhood_id, t.protocol_name, pd.version,
                 NULL, s.step ->> 'id', #{ANSWER}, COUNT(*), :at
          #{TRIAGES}
          CROSS JOIN LATERAL jsonb_array_elements(pd.definition -> 'steps') AS s(step)
          WHERE t.status = 'completed' AND NOT #{REVOKED} AND #{window('t.completed_at')}
            AND s.step -> 'analytic' = 'true'::jsonb
            AND (
              (s.step ->> 'answer_type' = 'boolean' AND #{ANSWER} IN ('true', 'false'))
              OR (s.step ->> 'answer_type' = 'enum' AND jsonb_typeof(s.step -> 'options') = 'array'
                  AND (s.step -> 'options') @> jsonb_build_array(#{ANSWER}))
            )
          GROUP BY 1, 4, 5, 6, 8, 9
        SQL
      end
    end
  end
end
```

Em `app/services/analytics/consolidate.rb`, `fronts` passa a `[ Demand, Quality, Calibration, Epidemiology ]`.

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/services/analytics/consolidate/epidemiology.rb app/services/analytics/consolidate.rb spec/services/analytics/consolidate/epidemiology_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: consolidate answers of analytic questions by neighborhood

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 9: Supressão e publicação na plataforma (F-14.8)

**Files:**
- Create: `app/services/analytics/suppression.rb`, `app/services/analytics/publish.rb`
- Test: `spec/services/analytics/suppression_spec.rb`, `spec/services/analytics/publish_spec.rb`

**Interfaces:**
- Consumes: `Admin::SmallCount` (`SUPPRESSED`, `small?`, `wrap`); `CityAnalyticsIndicator` (Task 2); `AnalyticsDailyFact`.
- Produces:
  - `Analytics::Suppression::SUPPRESSED` (`{ suppressed: true }`), `.cell(n) → Integer | SUPPRESSED`, `.rate(numerator, denominator) → Float | SUPPRESSED | nil` (1 casa; `nil` com denominador 0), `.sort_value(cell_or_rate) → Numeric` (suprimido e nulo = 0).
  - `Analytics::Publish.call(from:, to:, at: Time.current) → Integer` (linhas gravadas) — refaz na plataforma as semanas (segunda a domingo) tocadas por `from..to`, para a cidade `City.find_by!(slug: Current.city.slug)`; purga indicadores da cidade com `week_start` anterior a hoje − 5 anos.

- [ ] **Step 1: Escreva as specs**

```ruby
# spec/services/analytics/suppression_spec.rb
require "rails_helper"

# Contratos §0: Cell e Rate. Supressão depois de somar; zero continua zero.
RSpec.describe Analytics::Suppression do
  it "contagem de 1 a 4 vira { suppressed: true }; 0 e 5+ ficam" do
    expect([ 0, 1, 4, 5, 120 ].map { |n| described_class.cell(n) })
      .to eq([ 0, { suppressed: true }, { suppressed: true }, 5, 120 ])
  end

  it "taxa: nula com denominador 0, suprimida com numerador ou denominador pequenos, senão 1 casa" do
    expect(described_class.rate(0, 0)).to be_nil
    expect(described_class.rate(3, 200)).to eq(described_class::SUPPRESSED)
    expect(described_class.rate(3, 4)).to eq(described_class::SUPPRESSED)
    expect(described_class.rate(0, 3)).to eq(described_class::SUPPRESSED)
    expect(described_class.rate(0, 50)).to eq(0.0)
    expect(described_class.rate(10, 30)).to eq(33.3)
  end

  it "ordena suprimido e nulo como 0" do
    expect([ { suppressed: true }, nil, 7 ].map { |v| described_class.sort_value(v) }).to eq([ 0, 0, 7 ])
  end
end
```

```ruby
# spec/services/analytics/publish_spec.rb
require "rails_helper"

# Spec §5 (ADR 0025): seis indicadores semanais da cidade inteira, já
# suprimidos, no banco de plataforma. Idempotente; taxa sem denominador não é
# gravada; o número de 1 a 4 nunca sai.
RSpec.describe Analytics::Publish do
  let!(:city_record) { register_test_city! }
  let(:monday) { (Time.zone.today - 21).beginning_of_week }

  def published(week = monday)
    CityAnalyticsIndicator.where(city_id: city_record.id, week_start: week).order(:indicator)
                          .pluck(:indicator, :value, :suppressed).map { |i, v, s| [ i, v&.to_f, s ] }
  end

  before do
    fact!(metric: "triage.started", day: monday, value: 3)
    fact!(metric: "triage.started", day: monday + 1, value: 3)          # 6: soma antes de suprimir
    fact!(metric: "triage.completed", day: monday + 2, value: 3, tier: "alta")
    fact!(metric: "attendance.closed", day: monday, value: 10, dim: "discharged", health_unit_id: SecureRandom.uuid)
    fact!(metric: "attendance.closed", day: monday + 6, value: 2, dim: "left", health_unit_id: SecureRandom.uuid)
    fact!(metric: "attendance.wait", day: monday, value: 6, dim: "0-15")
    fact!(metric: "attendance.wait", day: monday, value: 5, dim: "15-30")
    fact!(metric: "attendance.wait", day: monday + 3, value: 11, dim: "30-60")
    fact!(metric: "appointment.ended", day: monday + 7, value: 9, dim: "checked_in") # semana seguinte
  end

  it "grava os seis indicadores da semana: contagem, suprimido e taxa; sem denominador, nada" do
    described_class.call(from: monday, to: monday + 6)

    expect(published).to eq([
      [ "attendances_closed", 12.0, false ],
      [ "left_pct", nil, true ],               # numerador 2
      [ "triages_completed", nil, true ],      # 3
      [ "triages_started", 6.0, false ],
      [ "wait_within_30_pct", 50.0, false ]    # 11 ÷ 22
    ])
    expect(CityAnalyticsIndicator.where(indicator: "no_show_pct")).to be_empty
  end

  it "publica todas as semanas tocadas pela janela" do
    described_class.call(from: monday + 5, to: monday + 8)

    expect(published(monday).map(&:first)).to include("triages_started")
    expect(published(monday + 7)).to eq([
      [ "attendances_closed", 0.0, false ], [ "no_show_pct", 0.0, false ],
      [ "triages_completed", 0.0, false ], [ "triages_started", 0.0, false ]
    ])
  end

  it "é idempotente e tira a linha que deixou de ter dado" do
    stale = CityAnalyticsIndicator.create!(city: city_record, week_start: monday, indicator: "no_show_pct", value: 40,
                                           suppressed: false, published_at: 2.days.ago)
    2.times { described_class.call(from: monday, to: monday + 6) }

    expect(CityAnalyticsIndicator.where(city_id: city_record.id, week_start: monday).count).to eq(5)
    expect(CityAnalyticsIndicator.exists?(stale.id)).to be(false)
  end

  it "purga só os indicadores da cidade com mais de 5 anos" do
    old_week = (Time.zone.today - 5.years - 7).beginning_of_week
    kept_week = (Time.zone.today - 5.years + 7).beginning_of_week
    other_city = create(:city)
    [ [ city_record, old_week ], [ city_record, kept_week ], [ other_city, old_week ] ].each do |city, week|
      CityAnalyticsIndicator.create!(city: city, week_start: week, indicator: "triages_started", value: 10,
                                     suppressed: false, published_at: Time.current)
    end

    described_class.call(from: monday, to: monday + 6)

    expect(CityAnalyticsIndicator.where(week_start: old_week).pluck(:city_id)).to eq([ other_city.id ])
    expect(CityAnalyticsIndicator.where(city_id: city_record.id, week_start: kept_week)).to exist
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics/suppression_spec.rb spec/services/analytics/publish_spec.rb`
Expected: FAIL com `uninitialized constant Analytics::Suppression`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/analytics/suppression.rb
module Analytics
  # Supressão de 1 a 4 SEMPRE no Analytics (D7; spec §6.3; contratos §0),
  # aplicada depois de somar período e recorte. Mesma regra de
  # Admin::SmallCount, que no módulo 11 só valia com filtro.
  module Suppression
    SUPPRESSED = Admin::SmallCount::SUPPRESSED

    module_function

    def cell(count) = Admin::SmallCount.wrap(count.to_i)

    # nil = "sem dado" (denominador 0). Numerador OU denominador em 1..4 →
    # oculto: com os dois à mostra, a taxa devolveria a contagem pequena.
    def rate(numerator, denominator)
      numerator = numerator.to_i
      denominator = denominator.to_i
      return nil if denominator.zero?
      return SUPPRESSED if Admin::SmallCount.small?(numerator) || Admin::SmallCount.small?(denominator)

      (numerator * 100.0 / denominator).round(1)
    end

    # Ordenação sem vazar a ordem das contagens pequenas (contratos §1.1).
    def sort_value(value) = value.is_a?(Numeric) ? value : 0
  end
end
```

```ruby
# app/services/analytics/publish.rb
module Analytics
  # Conjunto fixo da plataforma (spec §5; ADR 0025): seis indicadores
  # semanais da cidade inteira, já suprimidos. Refaz as semanas tocadas pela
  # janela apagando e gravando numa transação da plataforma (desvio 7 do
  # plano): idempotente, e a taxa que passou a "sem dado" some. O suprimido
  # vai como value NULL — o número de 1 a 4 nunca sai do banco da cidade.
  class Publish
    METRICS = %w[triage.started triage.completed attendance.closed attendance.wait appointment.ended].freeze
    RETENTION = 5.years

    def self.call(from:, to:, at: Time.current) = new(from: from, to: to, at: at).call

    def initialize(from:, to:, at:)
      @weeks = (from.beginning_of_week..to.beginning_of_week).step(7).to_a
      @at = at
    end

    def call
      city = City.find_by!(slug: Current.city.slug)
      rows = @weeks.flat_map { |week| indicators(week).map { |name, value| row(city, week, name, value) } }
      PlatformRecord.transaction(requires_new: true) do
        CityAnalyticsIndicator.where(city_id: city.id, week_start: @weeks).delete_all
        CityAnalyticsIndicator.insert_all!(rows) if rows.any?
        CityAnalyticsIndicator.where(city_id: city.id).where(week_start: ...(Time.zone.today - RETENTION)).delete_all
      end
      rows.size
    end

    private

    # { [semana, métrica, dim] => soma } das semanas inteiras (segunda a domingo).
    def sums
      @sums ||= AnalyticsDailyFact.where(metric: METRICS, day: @weeks.first..(@weeks.last + 6))
                                  .group(Arel.sql("date_trunc('week', day)::date"), :metric, :dim).sum(:value)
                                  .each_with_object(Hash.new(0)) do |((week, metric, dim), value), acc|
                                    acc[[ Analytics.to_date(week), metric, dim ]] += value
                                  end
    end

    def total(week, metric, dims = nil)
      sums.sum { |(w, m, d), value| w == week && m == metric && (dims.nil? || dims.include?(d)) ? value : 0 }
    end

    # nil (taxa sem denominador) = nenhuma linha naquela semana.
    def indicators(week)
      closed = total(week, "attendance.closed")
      {
        "triages_started" => Suppression.cell(total(week, "triage.started")),
        "triages_completed" => Suppression.cell(total(week, "triage.completed")),
        "attendances_closed" => Suppression.cell(closed),
        "wait_within_30_pct" => Suppression.rate(total(week, "attendance.wait", %w[0-15 15-30]),
                                                 total(week, "attendance.wait")),
        "no_show_pct" => Suppression.rate(total(week, "appointment.ended", %w[no_show]),
                                          total(week, "appointment.ended", %w[checked_in no_show])),
        "left_pct" => Suppression.rate(total(week, "attendance.closed", %w[left]), closed)
      }.compact
    end

    def row(city, week, indicator, value)
      suppressed = value == Suppression::SUPPRESSED
      { city_id: city.id, week_start: week, indicator: indicator, value: suppressed ? nil : value,
        suppressed: suppressed, published_at: @at }
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics/suppression_spec.rb spec/services/analytics/publish_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/services/analytics/suppression.rb app/services/analytics/publish.rb spec/services/analytics/suppression_spec.rb spec/services/analytics/publish_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: publish six suppressed weekly city indicators to the platform

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 10: `Analytics::Run` e `ConsolidateAnalyticsJob` (F-14.1)

**Files:**
- Create: `app/services/analytics/run.rb`, `app/jobs/consolidate_analytics_job.rb`
- Modify: `config/recurring.yml`
- Test: `spec/services/analytics/run_spec.rb`, `spec/services/analytics/run_lock_spec.rb`, `spec/jobs/consolidate_analytics_job_spec.rb`

**Interfaces:**
- Consumes: `Analytics::Consolidate.call` (Tasks 4–8), `Analytics::Publish.call` (Task 9), `EachCityJob`.
- Produces:
  - `Analytics::Run.call(kind:, from:, to:) → AnalyticsRun | nil` — `nil` quando outra consolidação da cidade está em curso (nada é gravado); `ArgumentError` com `from > to` ou `to >= hoje`. Falha de consolidação → run `failed`, fatos anteriores intactos, sem exceção. Falha de publicação → run `succeeded`, `published_at` nulo, `error` `"publish: <classe>: <1ª linha>"`.
  - `Analytics::Run.scheduled_window(today = Time.zone.today) → [today - 30, today - 1]`; `Analytics::Run.try_lock → true|false`; `Analytics::Run.unlock`; `Analytics::Run.describe(error) → String` (até 500); `Analytics::Run::Failed`; constantes `WINDOW_DAYS` (30), `FACT_RETENTION` (5 anos), `RUN_RETENTION` (90 dias).
  - `ConsolidateAnalyticsJob.perform_now` (recorrente de cidade, fila `housekeeping`, `every day at 2:30am America/Sao_Paulo`): run `scheduled` da janela; run `failed` → `Analytics::Run::Failed` (vira `EachCityJob::AggregatedFailure`).

- [ ] **Step 1: Escreva as specs**

```ruby
# spec/services/analytics/run_spec.rb
require "rails_helper"

# Spec §4.1 e §10.1 (job): janela, idempotência, atraso dentro e fora da
# janela, falha num consolidador, dia corrente, publicação e purgas.
RSpec.describe Analytics::Run do
  let!(:city_record) { register_test_city! }
  let(:today) { Time.zone.today }
  let(:unit) { create_unit("UBS Centro") }
  let!(:protocol) { create_default_protocol! }

  def snapshot = AnalyticsDailyFact.order(:day, :metric, :dim, :tier, :neighborhood_id, :health_unit_id)
                                   .pluck(:day, :metric, :health_unit_id, :neighborhood_id, :protocol_name,
                                          :protocol_version, :tier, :question_id, :dim, :value)

  it "a janela agendada vai de hoje − 30 a ontem" do
    expect(described_class.scheduled_window(today)).to eq([ today - 30, today - 1 ])
  end

  it "grava o run, consolida, publica e marca succeeded" do
    a_triage!(day: today - 2)

    run = scheduled_run!

    expect(run.reload).to have_attributes(kind: "scheduled", status: "succeeded", window_from: today - 30,
                                          window_to: today - 1, error: nil)
    expect(run.finished_at).to be_present
    expect(run.published_at).to be_present
    expect(AnalyticsDailyFact.where(metric: "triage.started", day: today - 2).sum(:value)).to eq(1)
    expect(CityAnalyticsIndicator.where(city_id: city_record.id)).to exist
  end

  it "duas execuções produzem os mesmos fatos" do
    an_attendance!(triage: a_triage!(day: today - 4), unit: unit, wait_minutes: 40, outcome: "return")
    a_triage!(day: today - 9, status: "aborted_by_timeout")

    scheduled_run!
    first = snapshot
    scheduled_run!

    expect(snapshot).to eq(first)
    expect(AnalyticsRun.pluck(:status)).to eq(%w[succeeded succeeded])
  end

  it "desfecho atrasado dentro da janela entra; fora da janela, não" do
    inside = a_triage!(day: today - 10, tier: "alta")
    outside = a_triage!(day: today - 40, tier: "alta")
    described_class.call(kind: "rebuild", from: today - 40, to: today - 31)
    scheduled_run!
    [ inside, outside ].each { |t| an_attendance!(triage: t, unit: unit, checked_in_at: Time.current - 2.hours) }

    scheduled_run!

    dims = ->(day) { AnalyticsDailyFact.where(metric: "calibration.outcome", day: day).pluck(:dim) }
    expect(dims.call(today - 10)).to eq(%w[discharged])
    expect(dims.call(today - 40)).to eq(%w[none])
  end

  it "falha num consolidador: run failed, rollback, fatos anteriores intactos" do
    a_triage!(day: today - 3)
    scheduled_run!
    before = snapshot
    a_triage!(day: today - 3)
    allow(Analytics::Consolidate::Quality).to receive(:call).and_raise(RuntimeError, "boom\nDETAIL: linha 42")

    run = scheduled_run!

    expect(run.reload).to have_attributes(status: "failed", error: "RuntimeError: boom", published_at: nil)
    expect(snapshot).to eq(before)
  end

  it "nunca consolida o dia corrente nem janela invertida" do
    expect { described_class.call(kind: "rebuild", from: today - 3, to: today) }.to raise_error(ArgumentError)
    expect { described_class.call(kind: "rebuild", from: today - 1, to: today - 2) }.to raise_error(ArgumentError)
    expect(AnalyticsRun.count).to eq(0)
  end

  it "com outra consolidação em curso, sai sem gravar nada" do
    allow(described_class).to receive(:try_lock).and_return(false)

    expect(scheduled_run!).to be_nil
    expect(AnalyticsRun.count).to eq(0)
  end

  it "falha da plataforma não derruba o run: succeeded, sem published_at, com o erro" do
    a_triage!(day: today - 2)
    allow(Analytics::Publish).to receive(:call).and_raise(ActiveRecord::ConnectionNotEstablished, "plataforma fora")

    run = scheduled_run!

    expect(run.reload).to have_attributes(status: "succeeded", published_at: nil,
                                          error: "publish: ActiveRecord::ConnectionNotEstablished: plataforma fora")
    expect(AnalyticsDailyFact.where(day: today - 2)).to exist
  end

  it "purga fatos com mais de 5 anos e runs com mais de 90 dias, guardando o último succeeded" do
    old = fact!(metric: "triage.started", day: today - 5.years - 1, value: 9)
    edge = fact!(metric: "triage.started", day: today - 5.years, value: 9)
    kept_run = consolidated_run!(finished_at: 100.days.ago)
    old_failed = AnalyticsRun.create!(kind: "scheduled", status: "failed", window_from: today - 130, window_to: today - 101,
                                      started_at: 95.days.ago, finished_at: 95.days.ago, error: "RuntimeError: x")
    allow(Analytics::Consolidate).to receive(:call).and_raise("boom") # nenhum succeeded novo

    scheduled_run!
    expect(AnalyticsRun.exists?(kept_run.id)).to be(true) # o run failed não purga

    allow(Analytics::Consolidate).to receive(:call).and_call_original
    scheduled_run!

    expect(AnalyticsDailyFact.exists?(old.id)).to be(false)
    expect(AnalyticsDailyFact.exists?(edge.id)).to be(true)
    expect(AnalyticsRun.exists?(old_failed.id)).to be(false)
    expect(AnalyticsRun.exists?(kept_run.id)).to be(false) # já há um succeeded mais novo
  end
end
```

```ruby
# spec/services/analytics/run_lock_spec.rb
require "rails_helper"

# Spec §4.1/§10.1: advisory lock por cidade — dois runs concorrentes, um só
# roda. Só threads reais contra o banco real de TEST_CITY_A (sem fixture
# transacional) exercitam o lock de sessão do Postgres; mesmo padrão de
# spec/models/health_unit_lock_spec.rb.
RSpec.describe "Analytics::Run: uma consolidação por cidade" do
  self.use_transactional_tests = false

  let(:release) { Queue.new }
  let(:threads) { [] }

  # Os writes commitam de verdade: solta e encerra as threads mesmo com a
  # expectativa falhando no meio, e só então apaga o que o run gravou.
  after do
    2.times { release << true }
    threads.each { |t| t.join(5) || t.kill }
    CityConnection.with(TEST_CITY_A) do
      AnalyticsDailyFact.delete_all
      AnalyticsRun.delete_all
    end
  end

  it "o segundo run concorrente sai sem gravar analytics_runs" do
    allow(Analytics::Publish).to receive(:call)
    inside = Queue.new
    allow(Analytics::Consolidate).to receive(:call).and_wrap_original do |original, **kwargs|
      inside << true
      release.pop
      original.call(**kwargs)
    end
    from, to = Analytics::Run.scheduled_window

    threads << Thread.new do
      CityConnection.with(TEST_CITY_A) { Analytics::Run.call(kind: "scheduled", from: from, to: to) }
    end
    inside.pop(timeout: 5) or raise "o primeiro run não entrou na consolidação"

    second = CityConnection.with(TEST_CITY_A) { Analytics::Run.call(kind: "rebuild", from: from, to: to) }

    expect(second).to be_nil
    release << true
    threads.each { |t| t.join(10) }
    expect(CityConnection.with(TEST_CITY_A) { AnalyticsRun.pluck(:kind, :status) }).to eq([ %w[scheduled succeeded] ])
  end
end
```

```ruby
# spec/jobs/consolidate_analytics_job_spec.rb
require "rails_helper"

RSpec.describe ConsolidateAnalyticsJob do
  let!(:city_record) { register_test_city! }

  before { create_default_protocol! }

  it "consolida a janela agendada da cidade e publica" do
    a_triage!(day: Time.zone.today - 2)

    described_class.perform_now

    expect(AnalyticsRun.sole).to have_attributes(kind: "scheduled", status: "succeeded",
                                                 window_from: Time.zone.today - 30, window_to: Time.zone.today - 1)
    expect(CityAnalyticsIndicator.where(city_id: city_record.id)).to exist
  end

  it "run que falha vira falha do job (fica visível no Solid Queue)" do
    allow(Analytics::Consolidate).to receive(:call).and_raise(RuntimeError, "boom")

    expect { described_class.perform_now }
      .to raise_error(EachCityJob::AggregatedFailure, /Analytics::Run::Failed: RuntimeError: boom/)
    expect(AnalyticsRun.sole.status).to eq("failed")
  end

  it "está no recurring.yml, às 2h30 de São Paulo, na fila housekeeping" do
    task = YAML.load_file(Rails.root.join("config/recurring.yml"), aliases: true).dig("production", "consolidate_analytics")
    expect(task).to eq("class" => "ConsolidateAnalyticsJob", "queue" => "housekeeping",
                       "schedule" => "every day at 2:30am America/Sao_Paulo")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics/run_spec.rb spec/services/analytics/run_lock_spec.rb spec/jobs/consolidate_analytics_job_spec.rb`
Expected: FAIL com `uninitialized constant Analytics::Run`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/analytics/run.rb
module Analytics
  # Uma consolidação (spec §4.1): trava por cidade, registro em
  # analytics_runs, fatos da janela refeitos numa transação só, publicação
  # depois do commit e purgas. Devolve o AnalyticsRun, ou nil quando outra
  # consolidação da mesma cidade está em curso (sai sem gravar nada).
  #
  # Falha de consolidação vira run failed com os fatos anteriores intactos, não
  # exceção: quem chama decide (o job levanta; o rebuild para).
  class Run
    # Uma chave por banco: cada cidade tem o seu, então a trava já é por cidade.
    LOCK_KEY = "analytics_consolidation"
    WINDOW_DAYS = 30
    FACT_RETENTION = 5.years
    RUN_RETENTION = 90.days
    ERROR_LIMIT = 500

    class Failed < StandardError; end

    def self.scheduled_window(today = Time.zone.today) = [ today - WINDOW_DAYS, today - 1 ]

    def self.call(kind:, from:, to:)
      raise ArgumentError, "janela invertida: #{from}..#{to}" if from > to
      raise ArgumentError, "o dia corrente nunca é consolidado (to=#{to})" if to >= Time.zone.today
      return nil unless try_lock

      begin
        new(kind: kind, from: from, to: to).call
      ensure
        unlock
      end
    end

    # Trava de SESSÃO (não de transação): cobre consolidação, publicação e
    # purga, que rodam em transações diferentes. Liberada no ensure.
    def self.try_lock
      ApplicationRecord.connection.select_value("SELECT pg_try_advisory_lock(hashtext('#{LOCK_KEY}'))")
    end

    def self.unlock
      ApplicationRecord.connection.select_value("SELECT pg_advisory_unlock(hashtext('#{LOCK_KEY}'))")
    end

    # Classe e primeira linha da mensagem: a linha DETAIL do PostgreSQL pode
    # trazer valores da linha que falhou.
    def self.describe(error)
      "#{error.class}: #{error.message.to_s.lines.first.to_s.strip}".truncate(ERROR_LIMIT)
    end

    def initialize(kind:, from:, to:)
      @kind = kind
      @from = from
      @to = to
    end

    def call
      run = AnalyticsRun.create!(kind: @kind, status: "running", window_from: @from, window_to: @to,
                                 started_at: Time.current)
      begin
        # requires_new: transação de verdade em produção, savepoint dentro de
        # outra (specs) — nos dois casos a falha desfaz só a janela.
        ApplicationRecord.transaction(requires_new: true) do
          Consolidate.call(from: @from, to: @to, at: run.started_at)
        end
      rescue StandardError => e
        run.update!(status: "failed", finished_at: Time.current, error: self.class.describe(e))
        return run
      end

      run.update!(status: "succeeded", finished_at: Time.current)
      publish(run)
      purge
      run
    end

    private

    # A próxima execução republica a própria janela (spec §4.1).
    def publish(run)
      Publish.call(from: run.window_from, to: run.window_to, at: Time.current)
      run.update!(published_at: Time.current)
    rescue StandardError => e
      run.update!(error: "publish: #{self.class.describe(e)}".truncate(ERROR_LIMIT))
    end

    # Guarda o último succeeded mesmo velho (desvio 8 do plano): sem ele,
    # as_of voltaria nulo com fatos no banco.
    def purge
      AnalyticsDailyFact.where(day: ...(Time.zone.today - FACT_RETENTION)).delete_all
      keep = AnalyticsRun.where(status: "succeeded").order(finished_at: :desc).limit(1).pluck(:id)
      AnalyticsRun.where(started_at: ...RUN_RETENTION.ago).where.not(id: keep).delete_all
    end
  end
end
```

```ruby
# app/jobs/consolidate_analytics_job.rb
# Consolidação diária do Analytics (ADR 0025; spec 2026-09-30 §4.1).
# Recorrente de cidade: no worker de cada cidade roda só nela (EachCityJob).
# Janela: de hoje − 30 a ontem. Run failed vira falha do job, para aparecer
# nas falhas do Solid Queue além de analytics_runs.
class ConsolidateAnalyticsJob < ApplicationJob
  prepend EachCityJob
  queue_as :housekeeping

  def perform
    from, to = Analytics::Run.scheduled_window
    run = Analytics::Run.call(kind: "scheduled", from: from, to: to)
    if run.nil?
      Rails.logger.info("[consolidate_analytics] city=#{Current.city.slug} outra consolidação em curso: nada a fazer")
    elsif run.status == "failed"
      raise Analytics::Run::Failed, run.error
    end
  end
end
```

Em `config/recurring.yml`, dentro de `default: &default`, depois de `sweep_abandoned` (desvio 6 do plano: depois dele):

```yaml

  consolidate_analytics:
    class: ConsolidateAnalyticsJob
    queue: housekeeping
    schedule: "every day at 2:30am America/Sao_Paulo"
```

- [ ] **Step 4: Rode e veja passar, com a guarda da configuração da fila**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics spec/jobs/consolidate_analytics_job_spec.rb spec/config/solid_queue_configuration_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/services/analytics/run.rb app/jobs/consolidate_analytics_job.rb config/recurring.yml spec/services/analytics/run_spec.rb spec/services/analytics/run_lock_spec.rb spec/jobs/consolidate_analytics_job_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: run the daily analytics consolidation per city with a lock and purges

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 11: Rebuild e rake `city:analytics:rebuild` (F-14.1)

**Files:**
- Create: `app/services/analytics/rebuild.rb`, `lib/tasks/analytics.rake`
- Test: `spec/services/analytics/rebuild_spec.rb`, `spec/tasks/analytics_rake_spec.rb`

**Interfaces:**
- Consumes: `Analytics::Run` (Task 10).
- Produces:
  - `Analytics::Rebuild.call(from: nil, to: nil) → Analytics::Rebuild::Report` (`from`, `to`, `runs` (Array de `AnalyticsRun`), `failed` (`AnalyticsRun` ou nil), `busy` (bool)) — blocos de 30 dias, `kind: "rebuild"`; `from` padrão = dia local do cru mais antigo; `to` padrão/limite = ontem; para no primeiro bloco `failed` ou ocupado.
  - `Analytics::Rebuild.earliest_raw_day → Date | nil`.
  - rake `city:analytics:rebuild[slug,from,to]` e `city:analytics:rebuild:all[from,to]`.

- [ ] **Step 1: Escreva as specs**

```ruby
# spec/services/analytics/rebuild_spec.rb
require "rails_helper"

# Spec §4.3: reconsolida em blocos de 30 dias, limitado ao cru que existe.
RSpec.describe Analytics::Rebuild do
  let!(:city_record) { register_test_city! }
  let(:today) { Time.zone.today }

  before { create_default_protocol! }

  def windows(report) = report.runs.map { |r| [ r.window_from, r.window_to, r.kind, r.status ] }

  it "sem cru: nada a fazer" do
    report = described_class.call
    expect(report.runs).to be_empty
    expect(report.busy).to be(false)
  end

  it "do cru mais antigo até ontem, em blocos de 30 dias" do
    a_triage!(day: today - 70, hour: 23, minute: 30)

    report = described_class.call

    expect(described_class.earliest_raw_day).to eq(today - 70)
    expect(windows(report)).to eq([
      [ today - 70, today - 41, "rebuild", "succeeded" ],
      [ today - 40, today - 11, "rebuild", "succeeded" ],
      [ today - 10, today - 1, "rebuild", "succeeded" ]
    ])
    expect(AnalyticsDailyFact.where(metric: "triage.started", day: today - 70).sum(:value)).to eq(1)
  end

  it "to depois de ontem é truncado; from explícito vale" do
    report = described_class.call(from: today - 5, to: today + 3)
    expect(windows(report)).to eq([ [ today - 5, today - 1, "rebuild", "succeeded" ] ])
  end

  it "outra consolidação em curso: para e informa" do
    a_triage!(day: today - 5)
    allow(Analytics::Run).to receive(:try_lock).and_return(false)

    report = described_class.call

    expect(report.busy).to be(true)
    expect(report.runs).to be_empty
  end

  it "bloco que falha interrompe os seguintes" do
    a_triage!(day: today - 70)
    allow(Analytics::Consolidate).to receive(:call).and_raise(RuntimeError, "boom")

    report = described_class.call

    expect(report.runs.size).to eq(1)
    expect(report.failed.error).to eq("RuntimeError: boom")
  end
end
```

```ruby
# spec/tasks/analytics_rake_spec.rb
require "rails_helper"
require "rake"

RSpec.describe "city:analytics:rebuild rake task" do
  before(:all) do
    Rails.application.load_tasks unless Rake::Task.task_defined?("city:analytics:rebuild")
  end

  before do
    Rake::Task["city:analytics:rebuild"].reenable
    Rake::Task["city:analytics:rebuild:all"].reenable
    create_default_protocol!
  end

  after { CityCatalog.reset_cache! }

  # A mesma linha de catálogo que os request specs usam: CityConnection.with(city)
  # cai na conexão de TEST_CITY_A da transação.
  let!(:city) { register_test_city! }

  def run(task, *args)
    out = StringIO.new
    original_stdout, $stdout = $stdout, out
    original_stderr, $stderr = $stderr, StringIO.new
    Rake::Task[task].invoke(*args)
    out.string
  ensure
    $stdout = original_stdout
    $stderr = original_stderr
  end

  it "reconsolida a cidade do cru mais antigo até ontem e relata os blocos" do
    a_triage!(day: Time.zone.today - 40)

    expect(run("city:analytics:rebuild", city.slug)).to include("#{city.slug}: 2 blocos de #{Time.zone.today - 40}")
    expect(AnalyticsRun.where(kind: "rebuild").count).to eq(2)
  end

  it "aceita from e to" do
    from = (Time.zone.today - 10).iso8601
    to = (Time.zone.today - 3).iso8601
    expect(run("city:analytics:rebuild", city.slug, from, to)).to include("1 blocos de #{from} a #{to}")
  end

  it "cidade sem cru: mensagem e saída sem erro" do
    expect(run("city:analytics:rebuild", city.slug)).to include("sem dado cru a consolidar")
  end

  it "slug ausente, cidade inexistente ou data inválida: aborta" do
    expect { run("city:analytics:rebuild") }.to raise_error(SystemExit)
    Rake::Task["city:analytics:rebuild"].reenable
    expect { run("city:analytics:rebuild", "nao-existe") }.to raise_error(SystemExit)
    Rake::Task["city:analytics:rebuild"].reenable
    expect { run("city:analytics:rebuild", city.slug, "2026-02-30") }.to raise_error(SystemExit)
  end

  it ":all passa por toda cidade ativa e aborta se alguma falhar" do
    a_triage!(day: Time.zone.today - 3)
    allow(Analytics::Consolidate).to receive(:call).and_raise(RuntimeError, "boom")

    expect { run("city:analytics:rebuild:all") }.to raise_error(SystemExit)
    expect(AnalyticsRun.where(status: "failed")).to exist
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics/rebuild_spec.rb spec/tasks/analytics_rake_spec.rb`
Expected: FAIL com `uninitialized constant Analytics::Rebuild` (e `Don't know how to build task 'city:analytics:rebuild'`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/analytics/rebuild.rb
module Analytics
  # Reconsolida um período longo (spec §4.3): primeiro deploy e correção de bug
  # de consolidação. Blocos de 30 dias com o mesmo Analytics::Run do job
  # (kind rebuild), cada um publicando as próprias semanas. Limitado ao cru que
  # ainda existe: o que foi anonimizado ou purgado não volta, e um período
  # antigo refeito perde o que saiu do cru desde então (desvio 14 do plano).
  class Rebuild
    BLOCK_DAYS = 30
    Report = Struct.new(:from, :to, :runs, :failed, :busy, keyword_init: true)

    # Primeiro dia local com dado cru que o Analytics lê.
    EARLIEST_SQL = <<~SQL.squish
      SELECT MIN(((moment AT TIME ZONE 'UTC') AT TIME ZONE '#{Analytics::TZ}')::date)::text FROM (
        SELECT MIN(created_at) AS moment FROM triages
        UNION ALL SELECT MIN(checked_in_at) FROM attendances
        UNION ALL SELECT MIN(created_at) FROM appointment_requests
        UNION ALL SELECT MIN(ended_at) FROM appointments
      ) raw
    SQL

    def self.earliest_raw_day
      value = ApplicationRecord.connection.select_value(EARLIEST_SQL)
      value && Date.iso8601(value)
    end

    def self.call(from: nil, to: nil)
      yesterday = Time.zone.yesterday
      to = [ to || yesterday, yesterday ].min
      from ||= earliest_raw_day
      runs = []
      return Report.new(from: from, to: to, runs: runs, failed: nil, busy: false) if from.nil? || from > to

      cursor = from
      while cursor <= to
        block_to = [ cursor + BLOCK_DAYS - 1, to ].min
        run = Run.call(kind: "rebuild", from: cursor, to: block_to)
        return Report.new(from: from, to: to, runs: runs, failed: nil, busy: true) if run.nil?

        runs << run
        return Report.new(from: from, to: to, runs: runs, failed: run, busy: false) if run.status == "failed"

        cursor = block_to + 1
      end
      Report.new(from: from, to: to, runs: runs, failed: nil, busy: false)
    end
  end
end
```

```ruby
# lib/tasks/analytics.rake
# Rebuild do Analytics por cidade (ADR 0025; spec 2026-09-30 §4.3, §12). Roda
# na conexão da cidade (CityConnection.with — nunca Current.city solto). Sem
# `from`: desde o cru mais antigo; sem `to`: ontem. Rollout: depois de
# city:migrate:all, city:analytics:rebuild:all.
namespace :city do
  namespace :analytics do
    # Lambda (não método): um `def` dentro de `namespace` vaza para o escopo
    # top-level do processo Rake (mesma razão de lib/tasks/city.rake).
    analytics_rebuild = lambda do |city, from, to|
      parse = ->(value) { value.presence && Date.iso8601(value) }
      report = CityConnection.with(city) { Analytics::Rebuild.call(from: parse.call(from), to: parse.call(to)) }
      raise "#{city.slug}: outra consolidação em curso — tente de novo" if report.busy
      if report.runs.empty?
        puts "[city:analytics:rebuild] #{city.slug}: sem dado cru a consolidar"
        next
      end

      puts "[city:analytics:rebuild] #{city.slug}: #{report.runs.size} blocos de #{report.from} a #{report.to}, " \
           "#{report.runs.count { |run| run.published_at.nil? }} sem publicação"
      if report.failed
        raise "#{city.slug}: bloco #{report.failed.window_from}..#{report.failed.window_to} falhou — " \
              "#{report.failed.error}"
      end
    end

    desc "Reconsolida o Analytics de uma cidade. Uso: city:analytics:rebuild[slug,from,to] (datas AAAA-MM-DD, opcionais)"
    task :rebuild, %i[slug from to] => :environment do |_t, args|
      abort "uso: rails 'city:analytics:rebuild[slug,from,to]'" if args[:slug].blank?
      city = City.find_by(slug: args[:slug]) || abort("[city:analytics:rebuild] cidade #{args[:slug]} não existe")
      abort "[city:analytics:rebuild] cidade #{city.slug} não está active" unless city.servable?
      abort "[city:analytics:rebuild] #{city.slug} com schema atrasado — rode city:migrate:all" if CitySchema.behind?(city)

      analytics_rebuild.call(city, args[:from], args[:to])
    rescue Date::Error
      abort "[city:analytics:rebuild] data inválida (use AAAA-MM-DD)"
    rescue RuntimeError => e
      abort "[city:analytics:rebuild] #{e.message}"
    end

    namespace :rebuild do
      desc "Reconsolida o Analytics de toda cidade active. Uso: city:analytics:rebuild:all[from,to]"
      task :all, %i[from to] => :environment do |_t, args|
        failed = []
        City.where(status: "active").order(:slug).each do |city|
          raise "schema atrasado — rode city:migrate:all" if CitySchema.behind?(city)

          analytics_rebuild.call(city, args[:from], args[:to])
        rescue StandardError => e
          failed << city.slug
          warn "[city:analytics:rebuild:all] #{city.slug} falhou — #{e.class}: #{e.message}"
        end
        abort "[city:analytics:rebuild:all] falharam: #{failed.join(', ')}" if failed.any?
      end
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/services/analytics spec/tasks/analytics_rake_spec.rb spec/tasks/territory_rake_spec.rb`
Expected: PASS.

- [ ] **Step 5: Rode o rebuild em dev**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod14 api bin/rails "city:analytics:rebuild[curitiba]"
```
Expected: `[city:analytics:rebuild] curitiba: N blocos de <data> a <ontem>, 0 sem publicação`.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/services/analytics/rebuild.rb lib/tasks/analytics.rake spec/services/analytics/rebuild_spec.rb spec/tasks/analytics_rake_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: add the analytics rebuild service and rake tasks

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

## Fatia 2 — F-14.2 a F-14.7 (leitura na cidade)

### Task 12: Papel `analyst`, parâmetros e estado do pipeline (F-14.2)

**Files:**
- Modify: `app/models/membership.rb`, `spec/models/membership_roles_spec.rb`
- Create: `app/services/analytics/params.rb`, `app/services/analytics/status.rb`
- Test: `spec/services/analytics/params_spec.rb`, `spec/services/analytics/status_spec.rb`, `spec/requests/analyst_grant_spec.rb`

**Interfaces:**
- Consumes: CHECK de papéis (Task 1); `AnalyticsRun`.
- Produces:
  - `"analyst"` em `Membership::ROLES` (fora de `PRIVILEGED_ROLES`).
  - `Analytics::Params.new(front, raw, today: Time.zone.today)` — levanta `Analytics::Params::Invalid` (`#code` ∈ `invalid_range`, `invalid_neighborhood`, `invalid_unit`, `invalid_protocol`). Leitores: `front`, `from` (Date), `to` (Date, ≤ ontem), `granularity` (`"week"`/`"month"`; `nil` em calibração), `neighborhood_id` (uuid, `"none"` ou nil), `health_unit_id`, `protocol_name`, `protocol_version` (Integer ou nil), `#periods → [Date]`, `#period_sql → String`, `#neighborhood? → bool`, `#neighborhood_value → uuid|nil`, `#filter → Hash` (4 chaves). Constantes `FRONTS`, `DATE`, `UUID`, `NONE`, `MAX_PERIODS`.
  - `Analytics::Status.call(now: Time.current) → Analytics::Status::Snapshot` (`last_run_status`, `last_succeeded_at`, `last_published_at`, `last_error`, `stale`; `#to_h` com chaves símbolo); `Analytics::Status::STALE_AFTER` (36 h).

- [ ] **Step 1: Escreva as specs**

Ao fim do `RSpec.describe` de `spec/models/membership_roles_spec.rb`, antes do `end`:

```ruby

  it "conhece o papel analyst e NÃO o trata como privilegiado (ADR 0025)" do
    expect(described_class::ROLES).to include("analyst")
    expect(described_class::PRIVILEGED_ROLES).not_to include("analyst")
    user = User.create!(email_address: "analise@cidade.gov.br", password: "senha-segura-123")
    expect { described_class.create!(user: user, role: "analyst", granted_at: Time.current) }.not_to raise_error
  end
```

```ruby
# spec/requests/analyst_grant_spec.rb
require "rails_helper"

# ADR 0025 (D6): analyst é só leitura analítica — conceder não pede step-up.
RSpec.describe "Conceder analyst", type: :request do
  let(:admin) do
    staff_with("admin-#{SecureRandom.hex(3)}@cidade.gov.br", "municipal_admin").tap do |u|
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
    end
  end
  let(:person) { staff_with("pessoa-#{SecureRandom.hex(3)}@cidade.gov.br", "viewer") }

  it "municipal_admin concede sem janela de step-up" do
    sign_in_as(admin)
    post "/setup/memberships", params: { user_id: person.id, role: "analyst" }, as: :json
    expect(response).to have_http_status(:created)
    expect(person.reload.has_role?(:analyst)).to be(true)
  end
end
```

```ruby
# spec/services/analytics/params_spec.rb
require "rails_helper"

# Contratos §1 (parâmetros) e desvio 5 do plano.
RSpec.describe Analytics::Params do
  let(:today) { Time.zone.today }
  let(:yesterday) { today - 1 }
  let(:base) { { from: (today - 20).iso8601, to: yesterday.iso8601 } }
  let(:centro) { Neighborhood.create!(name: "Centro", source: "manual", active: false) }
  let(:unit) { create_unit("UBS Centro") }
  let!(:protocol) { create_default_protocol! }

  def parse(front = "demand", **raw) = described_class.new(front, ActionController::Parameters.new(raw), today: today)

  def code(front = "demand", **raw)
    parse(front, **raw)
    nil
  rescue described_class::Invalid => e
    e.code
  end

  it "datas obrigatórias em AAAA-MM-DD, válidas e em ordem; to depois de ontem vira ontem" do
    expect(code(to: yesterday.iso8601)).to eq("invalid_range")
    expect(code(from: "2026-02-30", to: yesterday.iso8601)).to eq("invalid_range")
    expect(code(from: (today - 3).strftime("%d/%m/%Y"), to: yesterday.iso8601)).to eq("invalid_range")
    expect(code(from: (today - 3).iso8601, to: (today - 5).iso8601)).to eq("invalid_range")
    expect(code(from: today.iso8601, to: (today + 5).iso8601)).to eq("invalid_range")
    expect(parse(from: (today - 10).iso8601, to: (today + 5).iso8601).to).to eq(yesterday)
  end

  it "granularidade: week por padrão, month, nada mais; teto de 104 semanas e de 60 meses" do
    expect(parse(**base).granularity).to eq("week")
    expect(code(**base, granularity: "day")).to eq("invalid_range")

    first_week = yesterday.beginning_of_week - (103 * 7)
    expect(code(from: first_week.iso8601, to: yesterday.iso8601)).to be_nil
    expect(code(from: (first_week - 7).iso8601, to: yesterday.iso8601)).to eq("invalid_range")

    first_month = yesterday.beginning_of_month << 59
    expect(code(from: first_month.iso8601, to: yesterday.iso8601, granularity: "month")).to be_nil
    expect(code(from: (first_month << 1).iso8601, to: yesterday.iso8601, granularity: "month")).to eq("invalid_range")
  end

  it "períodos: segunda-feira (week) ou dia 1 (month), do período do from ao do to" do
    week = parse(**base)
    expect(week.periods).to eq(((today - 20).beginning_of_week..yesterday.beginning_of_week).step(7).to_a)
    expect(week.period_sql).to eq("date_trunc('week', day)::date")

    month = parse(from: (yesterday << 2).iso8601, to: yesterday.iso8601, granularity: "month")
    expect(month.periods).to eq([ (yesterday << 2).beginning_of_month, (yesterday << 1).beginning_of_month,
                                  yesterday.beginning_of_month ])
  end

  it "recortes válidos: bairro (inclusive inativo) ou none, unidade, protocolo e versão" do
    expect(parse(**base, neighborhood_id: centro.id)).to have_attributes(neighborhood?: true, neighborhood_value: centro.id)
    expect(parse(**base, neighborhood_id: "none")).to have_attributes(neighborhood?: true, neighborhood_value: nil)
    expect(parse(**base)).to have_attributes(neighborhood?: false)
    expect(parse(**base, health_unit_id: unit.id).health_unit_id).to eq(unit.id)
    calibration = parse("calibration", **base, protocol_name: protocol.name, protocol_version: "1")
    expect(calibration.filter).to eq(neighborhood_id: nil, health_unit_id: nil, protocol_name: protocol.name,
                                     protocol_version: 1)
  end

  it "recortes inválidos devolvem o código do contrato" do
    expect(code(**base, neighborhood_id: SecureRandom.uuid)).to eq("invalid_neighborhood")
    expect(code(**base, neighborhood_id: "centro")).to eq("invalid_neighborhood")
    expect(code(**base, health_unit_id: SecureRandom.uuid)).to eq("invalid_unit")
    expect(code(**base, protocol_name: "nao-existe")).to eq("invalid_protocol")
    expect(code("calibration", **base, protocol_version: "1")).to eq("invalid_protocol")
    expect(code("calibration", **base, protocol_name: protocol.name, protocol_version: "9")).to eq("invalid_protocol")
    expect(code("calibration", **base, protocol_name: protocol.name, protocol_version: "1x")).to eq("invalid_protocol")
  end

  it "recorte que não vale para a frente é ignorado e volta nulo" do
    quality = parse("quality", **base, neighborhood_id: "lixo", protocol_name: "lixo", health_unit_id: unit.id)
    expect(quality.filter).to eq(neighborhood_id: nil, health_unit_id: unit.id, protocol_name: nil, protocol_version: nil)
  end

  it "calibração: sem granularidade nem períodos, intervalo de até 60 meses" do
    calibration = parse("calibration", **base, granularity: "lixo")
    expect(calibration).to have_attributes(granularity: nil, periods: [])
    first_month = yesterday.beginning_of_month << 60
    expect(code("calibration", from: first_month.iso8601, to: yesterday.iso8601)).to eq("invalid_range")
  end
end
```

```ruby
# spec/services/analytics/status_spec.rb
require "rails_helper"

# Contratos §1 (as_of, stale) e §3 (analyticsStatus).
RSpec.describe Analytics::Status do
  it "nunca rodou: tudo nulo e stale" do
    expect(described_class.call.to_h).to eq(last_run_status: nil, last_succeeded_at: nil, last_published_at: nil,
                                            last_error: nil, stale: true)
  end

  it "último run, último succeeded e último publicado; stale depois de 36 h" do
    succeeded = consolidated_run!(finished_at: 40.hours.ago)
    AnalyticsRun.create!(kind: "scheduled", status: "failed", window_from: Time.zone.today - 30,
                         window_to: Time.zone.today - 1, started_at: 1.hour.ago, finished_at: 1.hour.ago,
                         error: "RuntimeError: boom")

    status = described_class.call

    expect(status).to have_attributes(last_run_status: "failed", last_error: "RuntimeError: boom", stale: true)
    expect(status.last_succeeded_at).to be_within(1.second).of(succeeded.finished_at)
    expect(status.last_published_at).to be_within(1.second).of(succeeded.published_at)
    expect(described_class.call(now: succeeded.finished_at + 35.hours).stale).to be(false)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/models/membership_roles_spec.rb spec/requests/analyst_grant_spec.rb spec/services/analytics/params_spec.rb spec/services/analytics/status_spec.rb`
Expected: FAIL (`ROLES` sem `analyst` — a concessão volta 422 `invalid_role`; `uninitialized constant Analytics::Params`).

- [ ] **Step 3: Implemente**

Em `app/models/membership.rb`, troque `ROLES` por:

```ruby
  ROLES = %w[analyst campaign_manager citizen_verifier health_professional municipal_admin protocol_author
             protocol_publisher protocol_reviewer viewer].freeze
```

e, no comentário de `PRIVILEGED_ROLES`, depois do parágrafo de `campaign_manager`:

```ruby
  # analyst (ADR 0025) NÃO entra: só lê agregados já suprimidos — sem step-up.
```

```ruby
# app/services/analytics/params.rb
module Analytics
  # Parâmetros de GET /admin/api/analytics/:front (contratos §1; desvio 5 do
  # plano). Recusa com o código do contrato (Invalid#code). Recorte que não
  # vale para a frente é ignorado e volta nulo em `filter`.
  class Params
    class Invalid < StandardError
      attr_reader :code

      def initialize(code)
        @code = code
        super(code)
      end
    end

    FRONTS = %w[demand quality calibration epidemiology].freeze
    GRANULARITIES = %w[week month].freeze
    MAX_PERIODS = { "week" => 104, "month" => 60 }.freeze
    FILTERS = {
      "demand" => %i[neighborhood unit protocol],
      "quality" => %i[unit],
      "calibration" => %i[protocol version],
      "epidemiology" => %i[neighborhood protocol version]
    }.freeze
    DATE = /\A\d{4}-\d{2}-\d{2}\z/
    UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/
    NONE = "none"

    attr_reader :front, :from, :to, :granularity, :neighborhood_id, :health_unit_id, :protocol_name, :protocol_version

    def initialize(front, raw, today: Time.zone.today)
      @front = front
      allowed = FILTERS.fetch(front)
      @from = date(raw[:from])
      @to = [ date(raw[:to]), today - 1 ].min
      raise Invalid, "invalid_range" if @from > @to

      @granularity = front == "calibration" ? nil : granularity_from(raw[:granularity])
      raise Invalid, "invalid_range" if period_count > MAX_PERIODS.fetch(@granularity || "month")

      @neighborhood_id = neighborhood(raw[:neighborhood_id]) if allowed.include?(:neighborhood)
      @health_unit_id = unit(raw[:health_unit_id]) if allowed.include?(:unit)
      @protocol_name = protocol(raw[:protocol_name]) if allowed.include?(:protocol)
      @protocol_version = version(raw[:protocol_version]) if allowed.include?(:version)
    end

    # Início de cada período, do período do `from` ao do `to` (contratos §0).
    def periods
      return [] if @granularity.nil?

      list = []
      cursor = period_start(@from)
      while cursor <= @to
        list << cursor
        cursor = @granularity == "week" ? cursor + 7 : cursor.next_month
      end
      list
    end

    def period_sql = "date_trunc('#{@granularity}', day)::date"

    def neighborhood? = !@neighborhood_id.nil?

    def neighborhood_value = @neighborhood_id == NONE ? nil : @neighborhood_id

    def filter
      { neighborhood_id: @neighborhood_id, health_unit_id: @health_unit_id, protocol_name: @protocol_name,
        protocol_version: @protocol_version }
    end

    private

    def period_start(date) = @granularity == "month" ? date.beginning_of_month : date.beginning_of_week

    def period_count
      if @granularity == "week"
        ((@to.beginning_of_week - @from.beginning_of_week).to_i / 7) + 1
      else
        (@to.year * 12 + @to.month) - (@from.year * 12 + @from.month) + 1
      end
    end

    def date(value)
      raise Invalid, "invalid_range" unless value.is_a?(String) && value.match?(DATE)

      Date.strptime(value, "%Y-%m-%d")
    rescue Date::Error
      raise Invalid, "invalid_range"
    end

    def granularity_from(value)
      return "week" if value.nil? || value == ""
      raise Invalid, "invalid_range" unless GRANULARITIES.include?(value)

      value
    end

    def neighborhood(value)
      return nil if value.nil? || value == ""
      raise Invalid, "invalid_neighborhood" unless value.is_a?(String)
      return NONE if value == NONE
      raise Invalid, "invalid_neighborhood" unless value.match?(UUID) && Neighborhood.exists?(id: value)

      value
    end

    def unit(value)
      return nil if value.nil? || value == ""
      raise Invalid, "invalid_unit" unless value.is_a?(String) && value.match?(UUID) && HealthUnit.exists?(id: value)

      value
    end

    def protocol(value)
      return nil if value.nil? || value == ""
      raise Invalid, "invalid_protocol" unless value.is_a?(String) && ProtocolDefinition.exists?(name: value)

      value
    end

    def version(value)
      return nil if value.nil? || value == ""
      raise Invalid, "invalid_protocol" unless @protocol_name && value.is_a?(String) && value.match?(/\A\d{1,9}\z/)
      raise Invalid, "invalid_protocol" unless ProtocolDefinition.exists?(name: @protocol_name, version: value.to_i)

      value.to_i
    end
  end
end
```

```ruby
# app/services/analytics/status.rb
module Analytics
  # Estado do pipeline da cidade (contratos §1 e §3), lido de analytics_runs.
  # `as_of` do Analytics = last_succeeded_at; stale = nunca consolidou ou há
  # mais de 36 h. last_error é o do run mais recente (falha dele ou da
  # publicação dele).
  class Status
    STALE_AFTER = 36.hours
    Snapshot = Struct.new(:last_run_status, :last_succeeded_at, :last_published_at, :last_error, :stale,
                          keyword_init: true)

    def self.call(now: Time.current)
      last = AnalyticsRun.order(started_at: :desc, id: :desc).first
      succeeded_at = AnalyticsRun.where(status: "succeeded").maximum(:finished_at)
      Snapshot.new(last_run_status: last&.status, last_succeeded_at: succeeded_at,
                   last_published_at: AnalyticsRun.maximum(:published_at), last_error: last&.error,
                   stale: succeeded_at.nil? || succeeded_at < now - STALE_AFTER)
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar, com as regressões de papéis**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/models/membership_roles_spec.rb spec/requests/analyst_grant_spec.rb spec/services/analytics/params_spec.rb spec/services/analytics/status_spec.rb spec/commands/grant_role_spec.rb spec/requests/setup_grant_role_spec.rb spec/requests/admin/api/membership_gate_spec.rb spec/requests/admin/api/neighborhood_filter_spec.rb spec/policies spec/requests/territory_spec.rb spec/requests/campaigns_spec.rb`
Expected: PASS (as specs que iteram `Membership::ROLES` passam a provar que `analyst` lê os painéis ao vivo, como todo papel ativo, e não escreve território nem campanha).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/models/membership.rb spec/models/membership_roles_spec.rb app/services/analytics/params.rb app/services/analytics/status.rb spec/services/analytics/params_spec.rb spec/services/analytics/status_spec.rb spec/requests/analyst_grant_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: add the analyst role, analytics params and pipeline status

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 13: `GET /admin/api/analytics/:front` e a Demanda (F-14.2, F-14.3)

**Files:**
- Create: `app/controllers/admin/api/analytics_controller.rb`, `app/queries/analytics/base_query.rb`, `app/queries/analytics/demand_query.rb`
- Modify: `config/routes.rb`, `spec/architecture/operator_grant_access_spec.rb`
- Test: `spec/requests/admin/api/analytics_spec.rb`, `spec/requests/admin/api/analytics_demand_spec.rb`

**Interfaces:**
- Consumes: `Analytics::Params`, `Analytics::Status` (Task 12); `Analytics::Suppression` (Task 9).
- Produces:
  - `GET /admin/api/analytics/:front` (nesta task só `demand`; as Tasks 14–16 abrem `quality`, `calibration`, `epidemiology`), envelope `{ data: { front, granularity, from, to, filter, periods, ...frente }, as_of, stale }`.
  - `Admin::Api::AnalyticsController::ROLES` (`%w[analyst municipal_admin]`), `::QUERIES` (frente → nome da classe).
  - `Analytics::BaseQuery.new(params, periods:)` com os privados `sums(metric, keys: [], filters: [])` (`{ [period, *keys] => soma }`), `totals(metric, keys:, filters: [])` (`{ keys => soma }`, período inteiro), `scoped`, `series`, `split`, `flat`, `row(by_period) → { series:, total: }`, `ordered(rows, name:)`, `units → [{ health_unit_id, name, active }]`.
  - `Analytics::DemandQuery#call → Hash` com `triages`, `triages_total`, `by_tier`, `by_protocol`, `by_neighborhood`, `attendances_by_unit`, `requests_opened`, `requests_closed`, `units`.

- [ ] **Step 1: Escreva as specs**

```ruby
# spec/requests/admin/api/analytics_spec.rb
require "rails_helper"

# ADR 0025 (D6, D12) e contratos §0/§1: quem lê, o envelope, os parâmetros e
# a soma antes da supressão. O conteúdo de cada frente tem spec própria.
RSpec.describe "Admin::Api analytics: acesso, envelope e parâmetros", type: :request do
  let(:today) { Time.zone.today }
  let(:monday) { (today - 14).beginning_of_week }
  let(:range) { { from: monday.iso8601, to: (monday + 6).iso8601 } }
  let(:hidden) { { "suppressed" => true } }

  def json = JSON.parse(response.body)
  def analyst = staff_with("analise-#{SecureRandom.hex(3)}@cidade.gov.br", "analyst")

  it "analyst e municipal_admin leem; todo outro papel recebe 403 forbidden_role" do
    consolidated_run!
    %w[analyst municipal_admin].each do |role|
      sign_in_as(staff_with("ok-#{role}-#{SecureRandom.hex(2)}@cidade.gov.br", role))
      get "/admin/api/analytics/demand", params: range
      expect(response).to have_http_status(:ok), role
    end
    (Membership::ROLES - %w[analyst municipal_admin]).each do |role|
      sign_in_as(staff_with("no-#{role}-#{SecureRandom.hex(2)}@cidade.gov.br", role))
      get "/admin/api/analytics/demand", params: range
      expect(response).to have_http_status(:forbidden), role
      expect(json).to eq("error" => "forbidden_role")
    end
  end

  it "operador com grant recebe 403 forbidden_role — e segue lendo o resto de /admin/api" do
    operator = Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                                otp_secret: ROTP::Base32.random, otp_enabled: true)
    sign_in_operator_grant(operator)

    get "/admin/api/analytics/demand", params: range
    expect(response).to have_http_status(:forbidden)
    expect(json).to eq("error" => "forbidden_role")

    get "/admin/api/overview", params: { period: "7d" }
    expect(response).to have_http_status(:ok)
  end

  it "sem sessão: 401; sem vínculo ativo: 403 no_city_membership; frente desconhecida: 404" do
    get "/admin/api/analytics/demand", params: range
    expect(response).to have_http_status(:unauthorized)

    sign_in_as(User.create!(email_address: "sem-vinculo-#{SecureRandom.hex(3)}@x.com", password: "senha-segura-123"))
    get "/admin/api/analytics/demand", params: range
    expect(json).to eq("error" => "no_city_membership")

    sign_in_as(analyst)
    get "/admin/api/analytics/lixo", params: range
    expect(response).to have_http_status(:not_found)
  end

  it "nunca consolidado: as_of nulo, stale, períodos e séries vazios" do
    sign_in_as(analyst)
    get "/admin/api/analytics/demand", params: range

    expect(json).to include("as_of" => nil, "stale" => true)
    expect(json["data"]).to include("front" => "demand", "granularity" => "week", "periods" => [],
                                    "from" => monday.iso8601, "to" => (monday + 6).iso8601)
    expect(json.dig("data", "triages")).to eq("started" => [], "completed" => [], "aborted" => [])
  end

  it "as_of é o fim do último run succeeded, em UTC; stale depois de 36 h" do
    run = consolidated_run!(finished_at: 2.hours.ago)
    sign_in_as(analyst)

    get "/admin/api/analytics/demand", params: range
    expect(json).to include("as_of" => run.finished_at.utc.iso8601, "stale" => false)

    run.update!(finished_at: 37.hours.ago)
    get "/admin/api/analytics/demand", params: range
    expect(json["stale"]).to be(true)
  end

  it "parâmetros inválidos: 422 com o código do contrato; to depois de ontem é truncado" do
    consolidated_run!
    sign_in_as(analyst)
    {
      {} => "invalid_range",
      range.merge(to: (monday - 1).iso8601) => "invalid_range",
      range.merge(granularity: "day") => "invalid_range",
      range.merge(from: (monday - 7 * 110).iso8601) => "invalid_range",
      range.merge(neighborhood_id: SecureRandom.uuid) => "invalid_neighborhood",
      range.merge(health_unit_id: "nao-e-uuid") => "invalid_unit",
      range.merge(protocol_name: "nao-existe") => "invalid_protocol"
    }.each do |params, error|
      get "/admin/api/analytics/demand", params: params
      expect(response).to have_http_status(:unprocessable_entity), params.inspect
      expect(json).to eq("error" => error), params.inspect
    end

    get "/admin/api/analytics/demand", params: range.merge(to: (today + 10).iso8601)
    expect(json.dig("data", "to")).to eq((today - 1).iso8601)
  end

  it "soma antes de suprimir: dois dias com 3 na semana mostram 6; um dia com 3 fica oculto; o mesmo por mês" do
    consolidated_run!
    sign_in_as(analyst)
    fact!(metric: "triage.started", day: monday, value: 3, protocol_name: "resp", protocol_version: 1)
    fact!(metric: "triage.started", day: monday + 1, value: 3, protocol_name: "resp", protocol_version: 1)
    fact!(metric: "triage.started", day: monday + 7, value: 3, protocol_name: "resp", protocol_version: 1)

    get "/admin/api/analytics/demand", params: { from: monday.iso8601, to: (monday + 13).iso8601 }
    expect(json.dig("data", "triages", "started")).to eq([ 6, hidden ])
    expect(json.dig("data", "triages_total", "started")).to eq(9)

    month = (today << 3).beginning_of_month
    fact!(metric: "triage.completed", day: month + 1, value: 3, tier: "alta", protocol_name: "resp", protocol_version: 1)
    fact!(metric: "triage.completed", day: month + 20, value: 3, tier: "alta", protocol_name: "resp", protocol_version: 1)
    get "/admin/api/analytics/demand", params: { from: month.iso8601, to: (month + 27).iso8601, granularity: "month" }
    expect(json.dig("data", "triages", "completed")).to eq([ 6 ])
    get "/admin/api/analytics/demand", params: { from: month.iso8601, to: (month + 27).iso8601 }
    expect(json.dig("data", "triages", "completed")).to all(eq(0).or(eq(hidden)))
  end
end
```

```ruby
# spec/requests/admin/api/analytics_demand_spec.rb
require "rails_helper"

# Contratos §1.1 (demanda): séries, totais do período, listas ordenadas por
# total (suprimido conta 0), recortes que valem só onde o contrato diz, e a
# lista de todas as unidades para o seletor.
RSpec.describe "GET /admin/api/analytics/demand", type: :request do
  let(:monday) { (Time.zone.today - 21).beginning_of_week }
  let(:range) { { from: monday.iso8601, to: (monday + 13).iso8601 } } # duas semanas
  let(:hidden) { { "suppressed" => true } }
  let(:centro) { Neighborhood.create!(name: "Centro", source: "manual") }
  let(:batel) { Neighborhood.create!(name: "Batel", source: "manual", active: false) }
  let(:unit) { create_unit("UBS Centro") }
  let(:upa) { create_unit("UPA Norte", kind: "upa", active: false) }

  def data = JSON.parse(response.body)["data"]

  def triage_fact!(metric, day, value, neighborhood: nil, protocol: "resp", version: 1, **attrs)
    fact!(metric: metric, day: day, value: value, neighborhood_id: neighborhood&.id, protocol_name: protocol,
          protocol_version: version, **attrs)
  end

  before do
    # O recorte protocol_name exige nome existente (Analytics::Params).
    ProtocolDefinition.create!(name: "resp", version: 1, status: "active", definition: analytics_definition(name: "resp"))
    ProtocolDefinition.create!(name: "arbo", version: 2, status: "active",
                               definition: analytics_definition(name: "arbo", version: 2))
    consolidated_run!
    sign_in_as(staff_with("analise@cidade.gov.br", "analyst"))
  end

  it "séries de triagem, totais do período e listas por tier, protocolo e bairro" do
    triage_fact!("triage.started", monday, 8, neighborhood: centro)
    triage_fact!("triage.started", monday + 8, 3)
    triage_fact!("triage.completed", monday + 1, 6, neighborhood: centro, tier: "alta")
    triage_fact!("triage.completed", monday + 3, 7, tier: "alta")
    triage_fact!("triage.completed", monday + 9, 2, neighborhood: batel, protocol: "arbo", version: 2, tier: "baixa")
    triage_fact!("triage.aborted", monday + 2, 5, dim: "timeout")

    get "/admin/api/analytics/demand", params: range

    expect(data["periods"]).to eq([ monday.iso8601, (monday + 7).iso8601 ])
    expect(data["triages"]).to eq("started" => [ 8, hidden ], "completed" => [ 13, hidden ], "aborted" => [ 5, 0 ])
    expect(data["triages_total"]).to eq("started" => 11, "completed" => 15, "aborted" => 5)
    expect(data["by_tier"]).to eq([
      { "tier" => "alta", "series" => [ 13, 0 ], "total" => 13 },
      { "tier" => "baixa", "series" => [ 0, hidden ], "total" => hidden }
    ])
    expect(data["by_protocol"]).to eq([
      { "protocol_name" => "resp", "series" => [ 13, 0 ], "total" => 13 },
      { "protocol_name" => "arbo", "series" => [ 0, hidden ], "total" => hidden }
    ])
    expect(data["by_neighborhood"]).to eq([
      { "neighborhood_id" => nil, "name" => "Sem bairro", "total" => 7 },
      { "neighborhood_id" => centro.id, "name" => "Centro", "total" => 6 },
      { "neighborhood_id" => batel.id, "name" => "Batel", "total" => hidden }
    ])
  end

  it "chegadas por unidade, pedidos por tipo e motivo (todos, mesmo zerados) e a lista de todas as unidades" do
    fact!(metric: "attendance.checked_in", day: monday, value: 9, health_unit_id: unit.id, dim: "code")
    fact!(metric: "attendance.checked_in", day: monday + 7, value: 2, health_unit_id: upa.id, dim: "cpf_exception")
    fact!(metric: "request.opened", day: monday, value: 5, health_unit_id: unit.id, dim: "return")
    fact!(metric: "request.closed", day: monday + 8, value: 6, health_unit_id: unit.id, dim: "fulfilled")

    get "/admin/api/analytics/demand", params: range

    expect(data["attendances_by_unit"]).to eq([
      { "health_unit_id" => unit.id, "name" => "UBS Centro", "series" => [ 9, 0 ], "total" => 9 },
      { "health_unit_id" => upa.id, "name" => "UPA Norte", "series" => [ 0, hidden ], "total" => hidden }
    ])
    expect(data["requests_opened"]).to eq([
      { "kind" => "return", "series" => [ 5, 0 ], "total" => 5 },
      { "kind" => "referral", "series" => [ 0, 0 ], "total" => 0 }
    ])
    expect(data["requests_closed"]).to eq([
      { "reason" => "fulfilled", "series" => [ 0, 6 ], "total" => 6 },
      { "reason" => "citizen_cancelled", "series" => [ 0, 0 ], "total" => 0 },
      { "reason" => "dismissed", "series" => [ 0, 0 ], "total" => 0 }
    ])
    expect(data["units"]).to eq([
      { "health_unit_id" => unit.id, "name" => "UBS Centro", "active" => true },
      { "health_unit_id" => upa.id, "name" => "UPA Norte", "active" => false }
    ])
  end

  it "bairro e protocolo recortam só as triagens; unidade só chegadas e pedidos; units não muda" do
    triage_fact!("triage.started", monday, 8, neighborhood: centro)
    triage_fact!("triage.started", monday, 6)
    triage_fact!("triage.started", monday, 9, neighborhood: centro, protocol: "arbo")
    fact!(metric: "attendance.checked_in", day: monday, value: 9, health_unit_id: unit.id, dim: "code")
    fact!(metric: "attendance.checked_in", day: monday, value: 7, health_unit_id: upa.id, dim: "code")

    get "/admin/api/analytics/demand", params: range.merge(neighborhood_id: centro.id, protocol_name: "resp",
                                                           health_unit_id: unit.id)
    expect(data["filter"]).to eq("neighborhood_id" => centro.id, "health_unit_id" => unit.id,
                                 "protocol_name" => "resp", "protocol_version" => nil)
    expect(data["triages"]["started"]).to eq([ 8, 0 ])
    expect(data["attendances_by_unit"].map { |r| r["health_unit_id"] }).to eq([ unit.id ])
    expect(data["units"].size).to eq(2)

    get "/admin/api/analytics/demand", params: range.merge(neighborhood_id: "none")
    expect(data["triages"]["started"]).to eq([ 6, 0 ])
    expect(data["attendances_by_unit"].map { |r| r["health_unit_id"] }).to contain_exactly(unit.id, upa.id)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/requests/admin/api/analytics_spec.rb spec/requests/admin/api/analytics_demand_spec.rb`
Expected: FAIL — a rota não existe (404 onde se espera 200/403).

- [ ] **Step 3: Escreva o controller, a rota e as queries**

```ruby
# app/controllers/admin/api/analytics_controller.rb
# GET /admin/api/analytics/:front (ADR 0025; contratos §1). Só analyst e
# municipal_admin da cidade (D6, D12): o operador com grant NÃO lê — é a
# única leitura de /admin/api fechada a ele (desvio 3 do plano). Lê só
# analytics_daily_facts e analytics_runs; toda contagem sai por
# Analytics::Suppression, depois de somar.
class Admin::Api::AnalyticsController < Admin::Api::BaseController
  ROLES = %w[analyst municipal_admin].freeze
  QUERIES = {
    "demand" => "Analytics::DemandQuery"
  }.freeze

  # Zero ações para sessão de grant (a base libera todas).
  allow_operator_grant_access(only: [])
  # O envelope do Analytics não usa o período dos painéis ao vivo (desvio 4).
  skip_before_action :resolve_scope
  before_action :require_analytics_role

  rescue_from Analytics::Params::Invalid, with: :render_invalid_params

  def show
    front = params[:front].to_s
    parsed = Analytics::Params.new(front, params)
    status = Analytics::Status.call
    # Nunca consolidou: séries vazias (contratos §1).
    periods = status.last_succeeded_at ? parsed.periods : []
    data = { front: front, granularity: parsed.granularity, from: parsed.from.iso8601, to: parsed.to.iso8601,
             filter: parsed.filter, periods: periods.map(&:iso8601) }
    data.merge!(QUERIES.fetch(front).constantize.new(parsed, periods: periods).call)
    render json: { data: data, as_of: status.last_succeeded_at&.utc&.iso8601, stale: status.stale }
  end

  private

  # Sessão de operador por grant recebe a recusa do contrato, não o
  # operator_read_only genérico de Authentication.
  def require_authentication
    return request_authentication unless resume_session

    render json: { error: "forbidden_role" }, status: :forbidden if Current.session.operator_grant?
  end

  def require_analytics_role
    return if current_user&.memberships&.active&.exists?(role: ROLES)

    render json: { error: "forbidden_role" }, status: :forbidden
  end

  def render_invalid_params(error) = render(json: { error: error.code }, status: :unprocessable_entity)
end
```

Em `config/routes.rb`, no `namespace :admin do namespace :api do`, depois de `get "neighborhoods", ...`:

```ruby
      # Analytics (ADR 0025; contratos §1): só leitura, sem grant de operador.
      get "analytics/:front", to: "analytics#show", constraints: { front: /demand/ }
```

```ruby
# app/queries/analytics/base_query.rb
module Analytics
  # Leitura de analytics_daily_facts para uma frente (contratos §1). Soma
  # período e recorte no SQL e só então suprime (Analytics::Suppression).
  # Com `periods` vazio (nunca consolidou), as séries saem vazias.
  class BaseQuery
    def initialize(params, periods:)
      @params = params
      @periods = periods
    end

    private

    attr_reader :params, :periods

    # { [período, *chaves] => soma } de uma métrica no intervalo e nos recortes.
    def sums(metric, keys: [], filters: [])
      return {} if periods.empty?

      scoped(AnalyticsDailyFact.where(metric: metric, day: params.from..params.to), filters)
        .group(Arel.sql(params.period_sql), *keys).sum(:value)
        .to_h { |key, value| parts = Array(key); [ [ Analytics.to_date(parts.first), *parts.drop(1) ], value ] }
    end

    # { chaves => soma } do período inteiro (sem série).
    def totals(metric, keys:, filters: [])
      scoped(AnalyticsDailyFact.where(metric: metric, day: params.from..params.to), filters).group(*keys).sum(:value)
    end

    def scoped(relation, filters)
      relation = relation.where(neighborhood_id: params.neighborhood_value) if filters.include?(:neighborhood) && params.neighborhood?
      relation = relation.where(health_unit_id: params.health_unit_id) if filters.include?(:unit) && params.health_unit_id
      relation = relation.where(protocol_name: params.protocol_name) if filters.include?(:protocol) && params.protocol_name
      relation = relation.where(protocol_version: params.protocol_version) if filters.include?(:version) && params.protocol_version
      relation
    end

    # sums com uma chave → { chave => { período => soma } }.
    def split(sums)
      sums.each_with_object(Hash.new { |hash, key| hash[key] = Hash.new(0) }) do |((period, key), value), acc|
        acc[key][period] += value
      end
    end

    # sums sem chave (ou com chaves a juntar) → { período => soma }.
    def flat(sums)
      sums.each_with_object(Hash.new(0)) { |((period, *), value), acc| acc[period] += value }
    end

    def series(by_period) = periods.map { |period| Suppression.cell(by_period.fetch(period, 0)) }

    def row(by_period) = { series: series(by_period), total: Suppression.cell(by_period.values.sum) }

    def ordered(rows, name:) = rows.sort_by { |row| [ -Suppression.sort_value(row[:total]), row[name].to_s ] }

    # Todas as unidades da cidade, para o seletor (contratos §1): o analyst
    # não lê /attendance/units. Independe do recorte e do período.
    def units
      HealthUnit.order(:name).map { |unit| { health_unit_id: unit.id, name: unit.name, active: unit.active } }
    end

    def unit_names(ids) = HealthUnit.where(id: ids.compact).pluck(:id, :name).to_h
  end
end
```

```ruby
# app/queries/analytics/demand_query.rb
module Analytics
  # Demanda por território (contratos §1.1). Bairro e protocolo recortam só
  # as métricas de triagem; unidade, só chegadas e pedidos.
  class DemandQuery < BaseQuery
    TRIAGE = %i[neighborhood protocol].freeze
    UNIT = %i[unit].freeze
    REQUEST_KINDS = %w[return referral].freeze
    CLOSED_REASONS = %w[fulfilled citizen_cancelled dismissed].freeze
    NO_NEIGHBORHOOD = "Sem bairro"

    def call
      started = flat(sums("triage.started", filters: TRIAGE))
      completed = flat(sums("triage.completed", filters: TRIAGE))
      aborted = flat(sums("triage.aborted", filters: TRIAGE))
      {
        triages: { started: series(started), completed: series(completed), aborted: series(aborted) },
        triages_total: { started: Suppression.cell(started.values.sum), completed: Suppression.cell(completed.values.sum),
                         aborted: Suppression.cell(aborted.values.sum) },
        by_tier: keyed_rows("triage.completed", :tier),
        by_protocol: keyed_rows("triage.completed", :protocol_name),
        by_neighborhood: ordered(neighborhood_rows, name: :name),
        attendances_by_unit: ordered(unit_rows, name: :name),
        requests_opened: ordered(fixed_rows("request.opened", REQUEST_KINDS, :kind), name: :kind),
        requests_closed: ordered(fixed_rows("request.closed", CLOSED_REASONS, :reason), name: :reason),
        units: units
      }
    end

    private

    def keyed_rows(metric, column)
      rows = split(sums(metric, keys: [ column ], filters: TRIAGE)).map { |key, by_period| { column => key, **row(by_period) } }
      ordered(rows, name: column)
    end

    def neighborhood_rows
      by_neighborhood = split(sums("triage.completed", keys: [ :neighborhood_id ], filters: TRIAGE))
      names = Neighborhood.where(id: by_neighborhood.keys.compact).pluck(:id, :name).to_h
      by_neighborhood.map do |id, by_period|
        { neighborhood_id: id, name: id ? names.fetch(id, id) : NO_NEIGHBORHOOD,
          total: Suppression.cell(by_period.values.sum) }
      end
    end

    def unit_rows
      by_unit = split(sums("attendance.checked_in", keys: [ :health_unit_id ], filters: UNIT))
      names = unit_names(by_unit.keys)
      by_unit.map { |id, by_period| { health_unit_id: id, name: names.fetch(id, id), **row(by_period) } }
    end

    def fixed_rows(metric, values, name)
      by_dim = split(sums(metric, keys: [ :dim ], filters: UNIT))
      values.map { |value| { name => value, **row(by_dim.fetch(value, {})) } }
    end
  end
end
```

Em `spec/architecture/operator_grant_access_spec.rb`, no primeiro exemplo, depois do último `expect`:

```ruby

    # ADR 0025 (D12): o Analytics da cidade é a exceção dentro de Admin::Api.
    expect(Admin::Api::AnalyticsController.operator_grant_actions).to eq([])
```

- [ ] **Step 4: Rode e veja passar, com as guardas de /admin/api**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/requests/admin/api spec/architecture/operator_grant_access_spec.rb`
Expected: PASS (`module_05_invariants_spec` segue provando que nenhuma rota de `/admin/api` escreve).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/controllers/admin/api/analytics_controller.rb app/queries/analytics/base_query.rb app/queries/analytics/demand_query.rb config/routes.rb spec/architecture/operator_grant_access_spec.rb spec/requests/admin/api/analytics_spec.rb spec/requests/admin/api/analytics_demand_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: serve the demand analytics front to analysts with suppression

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 14: Frente Qualidade (F-14.4)

**Files:**
- Create: `app/queries/analytics/quality_query.rb`
- Modify: `app/controllers/admin/api/analytics_controller.rb` (`QUERIES`), `config/routes.rb` (constraint)
- Test: `spec/requests/admin/api/analytics_quality_spec.rb`

**Interfaces:**
- Consumes: `Analytics::BaseQuery` (Task 13).
- Produces: `Analytics::QualityQuery#call → Hash` com `wait` (`buckets` — as 5 faixas, na ordem —, `within_30_pct`, `within_30_pct_total`), `appointments` (4 estados, na ordem do contrato), `no_show_pct`, `no_show_pct_total`, `attendance_outcomes` (4 desfechos), `left_pct`, `left_pct_total`, `by_unit`, `units`.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/requests/admin/api/analytics_quality_spec.rb
require "rails_helper"

# Contratos §1.2 (qualidade): faixas fixas, taxas somadas no período e só
# então suprimidas, por unidade, e a lista de todas as unidades.
RSpec.describe "GET /admin/api/analytics/quality", type: :request do
  let(:monday) { (Time.zone.today - 21).beginning_of_week }
  let(:range) { { from: monday.iso8601, to: (monday + 13).iso8601 } }
  let(:hidden) { { "suppressed" => true } }
  let(:unit) { create_unit("UBS Centro") }
  let(:upa) { create_unit("UPA Norte", kind: "upa") }

  def data = JSON.parse(response.body)["data"]
  def unit_fact!(metric, day, value, dim, at: unit) = fact!(metric: metric, day: day, value: value, dim: dim, health_unit_id: at.id)

  before do
    consolidated_run!
    sign_in_as(staff_with("analise@cidade.gov.br", "analyst"))
  end

  it "espera: sempre as cinco faixas, e a taxa de até 30 min por período e no total" do
    unit_fact!("attendance.wait", monday, 10, "0-15")
    unit_fact!("attendance.wait", monday, 10, "30-60")
    unit_fact!("attendance.wait", monday + 7, 3, "15-30")
    unit_fact!("attendance.wait", monday + 7, 20, "120+")

    get "/admin/api/analytics/quality", params: range

    expect(data["wait"]["buckets"]).to eq([
      { "bucket" => "0-15", "series" => [ 10, 0 ], "total" => 10 },
      { "bucket" => "15-30", "series" => [ 0, hidden ], "total" => hidden },
      { "bucket" => "30-60", "series" => [ 10, 0 ], "total" => 10 },
      { "bucket" => "60-120", "series" => [ 0, 0 ], "total" => 0 },
      { "bucket" => "120+", "series" => [ 0, 20 ], "total" => 20 }
    ])
    expect(data["wait"]["within_30_pct"]).to eq([ 50.0, hidden ]) # 2ª semana: numerador 3
    expect(data["wait"]["within_30_pct_total"]).to eq(30.2)       # 13 ÷ 43
  end

  it "faltas: no_show ÷ (checked_in + no_show); saiu sem atendimento: left ÷ desfechos" do
    unit_fact!("appointment.ended", monday, 30, "checked_in")
    unit_fact!("appointment.ended", monday, 10, "no_show")
    unit_fact!("appointment.ended", monday, 7, "expired")
    unit_fact!("appointment.ended", monday + 7, 2, "no_show")
    unit_fact!("attendance.closed", monday, 40, "discharged")
    unit_fact!("attendance.closed", monday, 10, "left")
    unit_fact!("attendance.closed", monday + 7, 5, "referred")

    get "/admin/api/analytics/quality", params: range

    expect(data["appointments"]).to eq([
      { "status" => "checked_in", "series" => [ 30, 0 ], "total" => 30 },
      { "status" => "no_show", "series" => [ 10, hidden ], "total" => 12 },
      { "status" => "expired", "series" => [ 7, 0 ], "total" => 7 },
      { "status" => "cancelled_by_citizen", "series" => [ 0, 0 ], "total" => 0 }
    ])
    expect(data["no_show_pct"]).to eq([ 25.0, hidden ])
    expect(data["no_show_pct_total"]).to eq(28.6) # 12 ÷ 42
    expect(data["attendance_outcomes"].map { |r| r["outcome"] }).to eq(%w[discharged referred return left])
    expect(data["left_pct"]).to eq([ 20.0, 0.0 ])
    expect(data["left_pct_total"]).to eq(18.2) # 10 ÷ 55
  end

  it "sem denominador a taxa é nula" do
    get "/admin/api/analytics/quality", params: range

    expect(data["wait"]).to include("within_30_pct" => [ nil, nil ], "within_30_pct_total" => nil)
    expect(data).to include("no_show_pct_total" => nil, "left_pct_total" => nil)
  end

  it "por unidade: atendimentos encerrados e as três taxas; recorte de unidade; units com todas" do
    unit_fact!("attendance.closed", monday, 40, "discharged")
    unit_fact!("attendance.closed", monday, 10, "left")
    unit_fact!("attendance.wait", monday, 30, "0-15")
    unit_fact!("attendance.wait", monday, 20, "60-120")
    unit_fact!("attendance.closed", monday, 3, "discharged", at: upa)

    get "/admin/api/analytics/quality", params: range

    expect(data["by_unit"]).to eq([
      { "health_unit_id" => unit.id, "name" => "UBS Centro", "attendances" => 50,
        "wait_within_30_pct" => 60.0, "no_show_pct" => nil, "left_pct" => 20.0 },
      { "health_unit_id" => upa.id, "name" => "UPA Norte", "attendances" => hidden,
        "wait_within_30_pct" => nil, "no_show_pct" => nil, "left_pct" => hidden }
    ])

    get "/admin/api/analytics/quality", params: range.merge(health_unit_id: upa.id)
    expect(data["by_unit"].map { |r| r["health_unit_id"] }).to eq([ upa.id ])
    expect(data["attendance_outcomes"].first).to eq("outcome" => "discharged", "series" => [ hidden, 0 ], "total" => hidden)
    expect(data["units"].map { |u| u["health_unit_id"] }).to eq([ unit.id, upa.id ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/requests/admin/api/analytics_quality_spec.rb`
Expected: FAIL com 404 (a rota ainda só aceita `demand`).

- [ ] **Step 3: Implemente**

```ruby
# app/queries/analytics/quality_query.rb
module Analytics
  # Qualidade operacional (contratos §1.2). Taxa: numerador e denominador
  # somados no período (ou na unidade) e só então a regra de supressão.
  class QualityQuery < BaseQuery
    UNIT = %i[unit].freeze
    APPOINTMENT_STATUSES = %w[checked_in no_show expired cancelled_by_citizen].freeze
    OUTCOMES = %w[discharged referred return left].freeze
    WITHIN_30 = %w[0-15 15-30].freeze
    SHOWN = %w[checked_in no_show].freeze

    def call
      wait = split(sums("attendance.wait", keys: [ :dim ], filters: UNIT))
      appointments = split(sums("appointment.ended", keys: [ :dim ], filters: UNIT))
      outcomes = split(sums("attendance.closed", keys: [ :dim ], filters: UNIT))
      buckets = AnalyticsDailyFact::WAIT_BUCKETS
      {
        wait: {
          buckets: buckets.map { |bucket| { bucket: bucket, **row(wait.fetch(bucket, {})) } },
          within_30_pct: rate_series(wait, WITHIN_30, buckets),
          within_30_pct_total: rate_total(wait, WITHIN_30, buckets)
        },
        appointments: APPOINTMENT_STATUSES.map { |status| { status: status, **row(appointments.fetch(status, {})) } },
        no_show_pct: rate_series(appointments, %w[no_show], SHOWN),
        no_show_pct_total: rate_total(appointments, %w[no_show], SHOWN),
        attendance_outcomes: OUTCOMES.map { |outcome| { outcome: outcome, **row(outcomes.fetch(outcome, {})) } },
        left_pct: rate_series(outcomes, %w[left], OUTCOMES),
        left_pct_total: rate_total(outcomes, %w[left], OUTCOMES),
        by_unit: by_unit,
        units: units
      }
    end

    private

    def rate_series(by_dim, numerator, denominator)
      periods.map { |period| Suppression.rate(sum_at(by_dim, numerator, period), sum_at(by_dim, denominator, period)) }
    end

    def rate_total(by_dim, numerator, denominator)
      Suppression.rate(sum_all(by_dim, numerator), sum_all(by_dim, denominator))
    end

    def sum_at(by_dim, dims, period) = dims.sum { |dim| by_dim.fetch(dim, {}).fetch(period, 0) }

    def sum_all(by_dim, dims) = dims.sum { |dim| by_dim.fetch(dim, {}).values.sum }

    def by_unit
      return [] if periods.empty?

      wait = totals("attendance.wait", keys: %i[health_unit_id dim], filters: UNIT)
      appointments = totals("appointment.ended", keys: %i[health_unit_id dim], filters: UNIT)
      outcomes = totals("attendance.closed", keys: %i[health_unit_id dim], filters: UNIT)
      ids = (wait.keys + appointments.keys + outcomes.keys).map(&:first).uniq
      names = unit_names(ids)
      rows = ids.map do |id|
        at = ->(source, dims) { dims.sum { |dim| source.fetch([ id, dim ], 0) } }
        closed = at.call(outcomes, OUTCOMES)
        { health_unit_id: id, name: names.fetch(id, id), attendances: Suppression.cell(closed),
          wait_within_30_pct: Suppression.rate(at.call(wait, WITHIN_30), at.call(wait, AnalyticsDailyFact::WAIT_BUCKETS)),
          no_show_pct: Suppression.rate(at.call(appointments, %w[no_show]), at.call(appointments, SHOWN)),
          left_pct: Suppression.rate(at.call(outcomes, %w[left]), closed) }
      end
      rows.sort_by { |row| [ -Suppression.sort_value(row[:attendances]), row[:name].to_s ] }
    end
  end
end
```

Em `Admin::Api::AnalyticsController::QUERIES`, acrescente `"quality" => "Analytics::QualityQuery"`; na rota, `constraints: { front: /demand|quality/ }`.

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/requests/admin/api/analytics_quality_spec.rb spec/requests/admin/api/analytics_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/queries/analytics/quality_query.rb app/controllers/admin/api/analytics_controller.rb config/routes.rb spec/requests/admin/api/analytics_quality_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: serve the operational quality analytics front

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 15: Frente Calibração (F-14.5)

**Files:**
- Create: `app/queries/analytics/calibration_query.rb`
- Modify: `app/controllers/admin/api/analytics_controller.rb` (`QUERIES`), `config/routes.rb` (constraint)
- Test: `spec/requests/admin/api/analytics_calibration_spec.rb`

**Interfaces:**
- Consumes: `Analytics::BaseQuery#totals` (Task 13).
- Produces: `Analytics::CalibrationQuery#call → { versions: [{ protocol_name, protocol_version, rows: [{ tier, total, outcomes, shares }] }] }`; `Analytics::CalibrationQuery::OUTCOMES` (`%w[discharged referred return left none]`).

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/requests/admin/api/analytics_calibration_spec.rb
require "rails_helper"

# Contratos §1.3 (calibração): período inteiro, versão × tier × desfecho, com a
# proporção por linha; célula e proporção suprimidas pela mesma regra.
RSpec.describe "GET /admin/api/analytics/calibration", type: :request do
  let(:monday) { (Time.zone.today - 21).beginning_of_week }
  let(:range) { { from: monday.iso8601, to: (monday + 13).iso8601 } }
  let(:hidden) { { "suppressed" => true } }

  def data = JSON.parse(response.body)["data"]

  def outcome!(protocol, version, tier, dim, value, day: monday)
    fact!(metric: "calibration.outcome", day: day, value: value, protocol_name: protocol, protocol_version: version,
          tier: tier, dim: dim)
  end

  before do
    ProtocolDefinition.create!(name: "resp", version: 1, status: "retired", definition: analytics_definition(name: "resp"))
    ProtocolDefinition.create!(name: "resp", version: 2, status: "active", definition: analytics_definition(name: "resp", version: 2))
    consolidated_run!
    sign_in_as(staff_with("analise@cidade.gov.br", "municipal_admin"))
    outcome!("resp", 1, "alta", "discharged", 6)
    outcome!("resp", 1, "alta", "discharged", 4, day: monday + 9) # soma no período
    outcome!("resp", 1, "alta", "referred", 5)
    outcome!("resp", 1, "alta", "none", 5)
    outcome!("resp", 1, "baixa", "discharged", 3)
    outcome!("resp", 2, "alta", "none", 6)
    outcome!("arbo", 1, "media", "left", 8)
  end

  it "versões por nome e versão desc; linhas por total desc; outcomes e shares com as cinco chaves" do
    get "/admin/api/analytics/calibration", params: range

    expect(data).to include("granularity" => nil, "periods" => [])
    expect(data["versions"].map { |v| [ v["protocol_name"], v["protocol_version"] ] })
      .to eq([ [ "arbo", 1 ], [ "resp", 2 ], [ "resp", 1 ] ])
    resp_v1 = data["versions"].last
    expect(resp_v1["rows"]).to eq([
      { "tier" => "alta", "total" => 20,
        "outcomes" => { "discharged" => 10, "referred" => 5, "return" => 0, "left" => 0, "none" => 5 },
        "shares" => { "discharged" => 50.0, "referred" => 25.0, "return" => 0.0, "left" => 0.0, "none" => 25.0 } },
      { "tier" => "baixa", "total" => hidden,
        "outcomes" => { "discharged" => hidden, "referred" => 0, "return" => 0, "left" => 0, "none" => 0 },
        "shares" => { "discharged" => hidden, "referred" => hidden, "return" => hidden, "left" => hidden, "none" => hidden } }
    ])
  end

  it "recorta por protocolo e versão" do
    get "/admin/api/analytics/calibration", params: range.merge(protocol_name: "resp", protocol_version: "2")

    expect(data["filter"]).to include("protocol_name" => "resp", "protocol_version" => 2)
    expect(data["versions"]).to eq([ { "protocol_name" => "resp", "protocol_version" => 2, "rows" => [
      { "tier" => "alta", "total" => 6,
        "outcomes" => { "discharged" => 0, "referred" => 0, "return" => 0, "left" => 0, "none" => 6 },
        "shares" => { "discharged" => 0.0, "referred" => 0.0, "return" => 0.0, "left" => 0.0, "none" => 100.0 } }
    ] } ])
  end
end
```

(`arbo` não precisa existir em `protocol_definitions`: o recorte não é usado com ele.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/requests/admin/api/analytics_calibration_spec.rb`
Expected: FAIL com 404.

- [ ] **Step 3: Implemente**

```ruby
# app/queries/analytics/calibration_query.rb
module Analytics
  # Calibração de protocolo (contratos §1.3): período inteiro, sem série. Por
  # versão porque o tier é texto de cada protocolo (D13).
  class CalibrationQuery < BaseQuery
    OUTCOMES = %w[discharged referred return left none].freeze
    FILTERS = %i[protocol version].freeze

    def call
      facts = totals("calibration.outcome", keys: %i[protocol_name protocol_version tier dim], filters: FILTERS)
      versions = facts.group_by { |(name, version, _tier, _dim), _value| [ name, version ] }
      {
        versions: versions.sort_by { |(name, version), _| [ name.to_s, -version.to_i ] }.map do |(name, version), entries|
          by_tier = entries.group_by { |(_name, _version, tier, _dim), _value| tier }
          rows = by_tier.map do |tier, cells|
            tier_row(tier, cells.to_h { |(_name, _version, _tier, dim), value| [ dim, value ] })
          end
          { protocol_name: name, protocol_version: version,
            rows: rows.sort_by { |row| [ -Suppression.sort_value(row[:total]), row[:tier].to_s ] } }
        end
      }
    end

    private

    def tier_row(tier, by_outcome)
      total = OUTCOMES.sum { |outcome| by_outcome.fetch(outcome, 0) }
      { tier: tier, total: Suppression.cell(total),
        outcomes: OUTCOMES.to_h { |outcome| [ outcome, Suppression.cell(by_outcome.fetch(outcome, 0)) ] },
        shares: OUTCOMES.to_h { |outcome| [ outcome, Suppression.rate(by_outcome.fetch(outcome, 0), total) ] } }
    end
  end
end
```

Em `QUERIES`, acrescente `"calibration" => "Analytics::CalibrationQuery"`; na rota, `constraints: { front: /demand|quality|calibration/ }`.

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/requests/admin/api/analytics_calibration_spec.rb spec/requests/admin/api/analytics_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/queries/analytics/calibration_query.rb app/controllers/admin/api/analytics_controller.rb config/routes.rb spec/requests/admin/api/analytics_calibration_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: serve the protocol calibration analytics front

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 16: Frente Epidemiologia (F-14.7)

**Files:**
- Create: `app/queries/analytics/epidemiology_query.rb`
- Modify: `app/controllers/admin/api/analytics_controller.rb` (`QUERIES`), `config/routes.rb` (constraint)
- Test: `spec/requests/admin/api/analytics_epidemiology_spec.rb`

**Interfaces:**
- Consumes: `Analytics::BaseQuery` (Task 13); `analytics_definition` (Task 3).
- Produces: `Analytics::EpidemiologyQuery#call → { questions: [{ protocol_name, question_id, prompt, answer_type, options: [{ value, label, series, total }] }] }`; `Analytics::EpidemiologyQuery::CYCLED` (`%w[published active retired]`).

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/requests/admin/api/analytics_epidemiology_spec.rb
require "rails_helper"

# Contratos §1.4 e desvio 11 do plano: perguntas marcadas nas versões que
# passaram pelo ciclo; prompt e opções da maior dessas versões.
RSpec.describe "GET /admin/api/analytics/epidemiology", type: :request do
  let(:monday) { (Time.zone.today - 21).beginning_of_week }
  let(:range) { { from: monday.iso8601, to: (monday + 13).iso8601 } }
  let(:hidden) { { "suppressed" => true } }
  let(:centro) { Neighborhood.create!(name: "Centro", source: "manual") }

  def data = JSON.parse(response.body)["data"]

  def answer!(version, question, dim, value, day: monday, neighborhood: nil)
    fact!(metric: "epi.answer", day: day, value: value, protocol_name: "triagem-arbovirose", protocol_version: version,
          question_id: question, dim: dim, neighborhood_id: neighborhood&.id)
  end

  before do
    analytics_protocol!(version: 1, status: "retired", marks: %w[febre])
    definition = analytics_definition(version: 2)
    definition["steps"][0]["prompt"] = "Teve febre nos últimos 7 dias?"
    ProtocolDefinition.create!(name: "triagem-arbovirose", version: 2, status: "active", definition: definition)
    analytics_protocol!(version: 3, status: "draft", marks: %w[febre sintoma gestante]) # rascunho não nomeia
    create_default_protocol! # triage-respiratoria, sem marca
    consolidated_run!
    sign_in_as(staff_with("analise@cidade.gov.br", "analyst"))
    answer!(2, "febre", "true", 12)
    answer!(2, "febre", "false", 3)
    answer!(2, "sintoma", "Manchas", 7, day: monday + 8)
    answer!(1, "febre", "true", 5, neighborhood: centro)
  end

  it "uma entrada por pergunta marcada, com prompt e opções da versão mais recente do ciclo" do
    get "/admin/api/analytics/epidemiology", params: range

    expect(data["questions"]).to eq([
      { "protocol_name" => "triagem-arbovirose", "question_id" => "febre", "prompt" => "Teve febre nos últimos 7 dias?",
        "answer_type" => "boolean", "options" => [
          { "value" => "true", "label" => "Sim", "series" => [ 17, 0 ], "total" => 17 },
          { "value" => "false", "label" => "Não", "series" => [ hidden, 0 ], "total" => hidden }
        ] },
      { "protocol_name" => "triagem-arbovirose", "question_id" => "sintoma", "prompt" => "Qual o sintoma mais forte?",
        "answer_type" => "enum", "options" => [
          { "value" => "Manchas", "label" => "Manchas", "series" => [ 0, 7 ], "total" => 7 },
          { "value" => "Dor nas juntas", "label" => "Dor nas juntas", "series" => [ 0, 0 ], "total" => 0 },
          { "value" => "Nenhum", "label" => "Nenhum", "series" => [ 0, 0 ], "total" => 0 }
        ] }
    ])
  end

  it "recorte de versão: só as perguntas marcadas nela, ainda com o texto da mais recente" do
    get "/admin/api/analytics/epidemiology",
        params: range.merge(protocol_name: "triagem-arbovirose", protocol_version: "1")

    expect(data["questions"].map { |q| [ q["question_id"], q["prompt"] ] })
      .to eq([ [ "febre", "Teve febre nos últimos 7 dias?" ] ])
    expect(data["questions"].first["options"].first).to include("series" => [ 5, 0 ], "total" => 5)
  end

  it "recorte de bairro e protocolo sem pergunta marcada" do
    get "/admin/api/analytics/epidemiology", params: range.merge(neighborhood_id: centro.id)
    expect(data["questions"].first["options"].first["total"]).to eq(5)

    get "/admin/api/analytics/epidemiology", params: range.merge(protocol_name: "triage-respiratoria")
    expect(data["questions"]).to eq([])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/requests/admin/api/analytics_epidemiology_spec.rb`
Expected: FAIL com 404.

- [ ] **Step 3: Implemente**

```ruby
# app/queries/analytics/epidemiology_query.rb
module Analytics
  # Epidemiologia (contratos §1.4): uma entrada por pergunta marcada
  # `analytic` (boolean/enum) nas versões que passaram pelo ciclo assinado
  # (desvio 11 do plano); prompt e opções da maior dessas versões.
  class EpidemiologyQuery < BaseQuery
    FILTERS = %i[neighborhood protocol version].freeze
    CYCLED = %w[published active retired].freeze
    BOOLEAN_OPTIONS = [ %w[true Sim], %w[false Não] ].freeze

    def call
      facts = sums("epi.answer", keys: %i[protocol_name question_id dim], filters: FILTERS)
      by_option = facts.each_with_object(Hash.new { |hash, key| hash[key] = Hash.new(0) }) do |((period, name, question, dim), value), acc|
        acc[[ name, question, dim ]][period] += value
      end
      { questions: catalog.map { |question| question_json(question, by_option) } }
    end

    private

    def catalog
      scope = ProtocolDefinition.where(status: CYCLED)
      scope = scope.where(name: params.protocol_name) if params.protocol_name
      latest = {}
      marked_in_version = Set.new
      scope.order(:name, :version).each do |definition|
        Array(definition.definition["steps"]).each_with_index do |step, position|
          next unless analytic?(step)

          key = [ definition.name, step["id"] ]
          latest[key] = { protocol_name: definition.name, question_id: step["id"], prompt: step["prompt"],
                          answer_type: step["answer_type"], options: step["options"], position: position }
          marked_in_version << key if params.protocol_version.nil? || definition.version == params.protocol_version
        end
      end
      latest.select { |key, _| marked_in_version.include?(key) }.values
            .sort_by { |question| [ question[:protocol_name], question[:position] ] }
    end

    def analytic?(step)
      step.is_a?(Hash) && step["analytic"] == true &&
        (step["answer_type"] == "boolean" || (step["answer_type"] == "enum" && step["options"].is_a?(Array)))
    end

    def question_json(question, by_option)
      options = if question[:answer_type] == "boolean"
                  BOOLEAN_OPTIONS
                else
                  question[:options].map { |option| [ option.to_s, option.to_s ] }
                end
      { protocol_name: question[:protocol_name], question_id: question[:question_id], prompt: question[:prompt],
        answer_type: question[:answer_type],
        options: options.map do |value, label|
          key = [ question[:protocol_name], question[:question_id], value ]
          { value: value, label: label, **row(by_option.fetch(key, {})) }
        end }
    end
  end
end
```

Em `QUERIES`, acrescente `"epidemiology" => "Analytics::EpidemiologyQuery"`; na rota, `constraints: { front: /demand|quality|calibration|epidemiology/ }`.

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/requests/admin/api`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/queries/analytics/epidemiology_query.rb app/controllers/admin/api/analytics_controller.rb config/routes.rb spec/requests/admin/api/analytics_epidemiology_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: serve the epidemiology analytics front for marked questions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

## Fatia 3 — F-14.8 e F-14.9 (plataforma: console do operador e maintenance)

### Task 17: `GET /city_analytics` no console do operador (F-14.8)

**Files:**
- Create: `app/queries/analytics/city_indicators_query.rb`, `app/controllers/operators/city_analytics_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/operators/city_analytics_spec.rb`

**Interfaces:**
- Consumes: `CityAnalyticsIndicator` (Task 2); `Analytics::Params::Invalid`, `Analytics::Params::DATE` (Task 12); `Analytics::Suppression::SUPPRESSED` (Task 9); `Operators::BaseController`.
- Produces:
  - `GET /city_analytics?from&to` (host do console) → `{ data: { weeks, indicators, cities: [{ id, slug, name, uf, last_published_at, values: { indicador => [number | {suppressed: true} | null] } }] } }`; `422 { error: "invalid_range" }`.
  - `Analytics::CityIndicatorsQuery.call(from: nil, to: nil, today: Time.zone.today) → Hash`; `Analytics::CityIndicatorsQuery::MAX_WEEKS` (104), `::DEFAULT_WEEKS` (12); `Analytics::CityIndicatorsQuery.weeks(from, to, today) → [Date]` (usado também pelo GraphQL, Task 18).

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/requests/operators/city_analytics_spec.rb
require "rails_helper"

# Contratos §2 (ADR 0025): o console do operador lê só city_analytics_indicators
# e o catálogo de cidades — cidades × semanas × indicadores —, nunca o banco de
# uma cidade.
RSpec.describe "GET /city_analytics (console do operador)", type: :request do
  let(:password) { "s3nha-forte-1" }
  let!(:operator) do
    Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end
  let(:today) { Time.zone.today }
  let(:last_week) { today.beginning_of_week - 7 }
  let!(:curitiba) { create(:city, name: "Curitiba", uf: "PR") }
  let!(:maringa) { create(:city, name: "Maringá", uf: "PR") }
  let!(:archived) { create(:city, name: "Arquivada", status: "archived") }

  def json = JSON.parse(response.body)

  def verified_login!
    post "/session", params: { email_address: operator.email_address, password: password }
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(operator.otp_secret).now }
    expect(response).to have_http_status(:ok)
  end

  def indicator!(city, week, indicator, value, suppressed: false)
    CityAnalyticsIndicator.create!(city: city, week_start: week, indicator: indicator, value: value,
                                   suppressed: suppressed, published_at: Time.current)
  end

  def city_json(city) = json["data"]["cities"].find { |row| row["slug"] == city.slug }

  before { host! "admin.rotasaude.app" }

  it "sem from/to: as 12 semanas que terminam na anterior à atual; só cidades ativas, por nome; valores alinhados" do
    verified_login!
    indicator!(curitiba, last_week, "triages_started", 128)
    indicator!(curitiba, last_week, "no_show_pct", 12.5)
    indicator!(curitiba, last_week - 7, "triages_started", nil, suppressed: true)
    indicator!(curitiba, last_week - 7 * 20, "triages_started", 99) # fora das 12 semanas
    indicator!(maringa, last_week, "wait_within_30_pct", 70)

    get "/city_analytics"

    expect(response).to have_http_status(:ok)
    data = json["data"]
    expect(data["weeks"]).to eq(((last_week - 77)..last_week).step(7).map(&:iso8601))
    expect(data["indicators"]).to eq(CityAnalyticsIndicator::INDICATORS)
    slugs = data["cities"].map { |row| row["slug"] }
    expect(slugs).to include(curitiba.slug, maringa.slug)
    expect(slugs).not_to include(archived.slug)
    expect(slugs.index(curitiba.slug)).to be < slugs.index(maringa.slug)
    expect(city_json(curitiba)).to include("id" => curitiba.id, "name" => "Curitiba", "uf" => "PR")
    expect(city_json(curitiba)["values"].keys).to eq(CityAnalyticsIndicator::INDICATORS)
    expect(city_json(curitiba)["values"]["triages_started"]).to eq([ nil ] * 10 + [ { "suppressed" => true }, 128 ])
    expect(city_json(curitiba)["values"]["no_show_pct"].last).to eq(12.5)
    expect(city_json(curitiba)["last_published_at"]).to match(/\A\d{4}-\d{2}-\d{2}T.*Z\z/)
    expect(city_json(maringa)["values"]["wait_within_30_pct"].last).to eq(70.0)
    expect(city_json(maringa)["values"]["triages_started"]).to all(be_nil)
  end

  it "from/to escolhem as semanas pela segunda-feira; até 104; inválido é 422 invalid_range" do
    verified_login!

    get "/city_analytics", params: { from: (last_week + 2).iso8601, to: (last_week + 3).iso8601 }
    expect(json["data"]["weeks"]).to eq([ last_week.iso8601 ])

    [
      { from: last_week.iso8601 },
      { from: "2026-02-30", to: last_week.iso8601 },
      { from: last_week.iso8601, to: (last_week - 7).iso8601 },
      { from: (last_week - 7 * 104).iso8601, to: last_week.iso8601 }
    ].each do |params|
      get "/city_analytics", params: params
      expect(response).to have_http_status(:unprocessable_entity), params.inspect
      expect(json).to eq("error" => "invalid_range")
    end
  end

  it "sem sessão de operador: 401; no host de uma cidade: 404" do
    get "/city_analytics"
    expect(response).to have_http_status(:unauthorized)

    host! test_city_host
    get "/city_analytics"
    expect(response).to have_http_status(:not_found)
  end

  it "nunca abre o banco de uma cidade" do
    verified_login!
    indicator!(curitiba, last_week, "triages_started", 128)
    allow(CityConnection).to receive(:with).and_call_original

    get "/city_analytics"

    expect(response).to have_http_status(:ok)
    expect(CityConnection).not_to have_received(:with)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/requests/operators/city_analytics_spec.rb`
Expected: FAIL (404 na rota).

- [ ] **Step 3: Implemente**

```ruby
# app/queries/analytics/city_indicators_query.rb
module Analytics
  # GET /city_analytics (contratos §2): cidades ativas × semanas × os seis
  # indicadores, só do banco de plataforma — nunca abre banco de cidade (ADR
  # 0025, invariante). Célula: número (contagem inteira ou % com 1 casa),
  # { suppressed: true } ou null (sem linha = sem dado).
  class CityIndicatorsQuery
    DEFAULT_WEEKS = 12
    MAX_WEEKS = 104

    def self.call(from: nil, to: nil, today: Time.zone.today) = new(weeks(from, to, today)).call

    # Sem from/to: as 12 semanas que terminam na anterior à atual.
    def self.weeks(from, to, today)
      if from.blank? && to.blank?
        last = today.beginning_of_week - 7
        first = last - (DEFAULT_WEEKS - 1) * 7
      else
        first = parse(from).beginning_of_week
        last = parse(to).beginning_of_week
      end
      raise Params::Invalid, "invalid_range" if first > last || ((last - first).to_i / 7) + 1 > MAX_WEEKS

      (first..last).step(7).to_a
    end

    def self.parse(value)
      raise Params::Invalid, "invalid_range" unless value.is_a?(String) && value.match?(Params::DATE)

      Date.strptime(value, "%Y-%m-%d")
    rescue Date::Error
      raise Params::Invalid, "invalid_range"
    end

    def initialize(weeks)
      @weeks = weeks
    end

    def call
      cities = City.where(status: "active").order(:name).to_a
      ids = cities.map(&:id)
      index = CityAnalyticsIndicator.where(city_id: ids, week_start: @weeks).index_by { |row| [ row.city_id, row.week_start, row.indicator ] }
      published = CityAnalyticsIndicator.where(city_id: ids).group(:city_id).maximum(:published_at)
      {
        weeks: @weeks.map(&:iso8601),
        indicators: CityAnalyticsIndicator::INDICATORS,
        cities: cities.map do |city|
          { id: city.id, slug: city.slug, name: city.name, uf: city.uf,
            last_published_at: published[city.id]&.utc&.iso8601,
            values: CityAnalyticsIndicator::INDICATORS.to_h do |indicator|
              [ indicator, @weeks.map { |week| cell(index[[ city.id, week, indicator ]]) } ]
            end }
        end
      }
    end

    private

    # decimal vira número JSON (BigDecimal sairia como string).
    def cell(row)
      return nil if row.nil?
      return Suppression::SUPPRESSED if row.suppressed

      CityAnalyticsIndicator::COUNT_INDICATORS.include?(row.indicator) ? row.value.to_i : row.value.to_f.round(1)
    end
  end
end
```

```ruby
# app/controllers/operators/city_analytics_controller.rb
# GET /city_analytics (ADR 0025; contratos §2): indicadores publicados das
# cidades, no console do operador. Só plataforma.
module Operators
  class CityAnalyticsController < BaseController
    def index
      render json: { data: Analytics::CityIndicatorsQuery.call(from: params[:from], to: params[:to]) }
    rescue Analytics::Params::Invalid => e
      render json: { error: e.code }, status: :unprocessable_entity
    end
  end
end
```

Em `config/routes.rb`, dentro de `constraints(PlatformConsoleHost) do scope module: :operators, as: :operator do`, depois de `resources :unknown_channels, only: :index`:

```ruby
      # Indicadores publicados das cidades (ADR 0025; contratos §2). Só plataforma.
      get "/city_analytics", to: "city_analytics#index"
```

- [ ] **Step 4: Rode e veja passar, com as guardas do console**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/requests/operators spec/architecture/host_authorization_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/queries/analytics/city_indicators_query.rb app/controllers/operators/city_analytics_controller.rb config/routes.rb spec/requests/operators/city_analytics_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: list published city indicators on the platform console

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 18: Analytics no GraphQL de manutenção (F-14.9)

**Files:**
- Create: `app/graphql/maintenance/types/analytics_indicator_type.rb`, `app/graphql/maintenance/types/analytics_status_type.rb`
- Modify: `app/graphql/maintenance/types/city_type.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/city_analytics_spec.rb`

**Interfaces:**
- Consumes: `CityAnalyticsIndicator`; `Analytics::Status.call` (Task 12); `Analytics::CityIndicatorsQuery.weeks` e `::MAX_WEEKS` (Task 17); `CityType#inside`.
- Produces (contratos §3): `AnalyticsIndicator { weekStart: ISO8601Date!, indicator: String!, value: Float, suppressed: Boolean! }`; `AnalyticsStatus { lastRunStatus, lastSucceededAt, lastPublishedAt, lastError, stale: Boolean! }`; `City.analyticsIndicators(from:, to:): [AnalyticsIndicator!]!` (plataforma; `INVALID_RANGE` com `from > to` ou mais de 104 semanas); `City.analyticsStatus: AnalyticsStatus` (anulável, pelo `inside`).

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/requests/maintenance/city_analytics_spec.rb
require "rails_helper"

# Contratos §3 (F-14.9): analyticsIndicators lê o banco de PLATAFORMA sem abrir a
# cidade; analyticsStatus lê analytics_runs da cidade e é anulável — cidade
# inalcançável anula só ele, nunca `city`.
RSpec.describe "Maintenance city analytics", type: :request do
  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:password) { "s3nha-forte-1" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "cy-an-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end
  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }
  let(:monday) { (Time.zone.today - 14).beginning_of_week }

  def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
  def json = JSON.parse(response.body)

  # Mesmo arranjo de login de spec/requests/maintenance/city_operations_spec.rb.
  def login!
    post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now },
         headers: browser
    expect(response).to have_http_status(:ok)
  end

  def gql!(query, **variables)
    post "/graphql", params: { query: query, variables: variables.to_json }, headers: browser
  end

  # Método, não constante: constante dentro de `describe` vaza para o escopo global.
  def indicators_query
    <<~GQL
      query($slug: String!, $from: ISO8601Date!, $to: ISO8601Date!) {
        city(slug: $slug) { slug analyticsIndicators(from: $from, to: $to) { weekStart indicator value suppressed } }
      }
    GQL
  end

  before do
    unless City.exists?(slug: TEST_CITY_A.slug)
      City.create!(slug: TEST_CITY_A.slug, name: TEST_CITY_A.name, status: "active",
                   database_url: TEST_CITY_A.database_url, encryption_key: TEST_CITY_A.encryption_key,
                   schema_version: CitySchema.expected_version.to_s)
    end
    host! "maintenance-api.rotasaude.app"
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
    login!
  end

  it "analyticsIndicators: linhas da plataforma no intervalo, suprimido com value nulo, sem abrir a cidade" do
    CityAnalyticsIndicator.create!(city: city, week_start: monday, indicator: "triages_started", value: 40,
                                   suppressed: false, published_at: Time.current)
    CityAnalyticsIndicator.create!(city: city, week_start: monday, indicator: "no_show_pct", value: nil,
                                   suppressed: true, published_at: Time.current)
    CityAnalyticsIndicator.create!(city: city, week_start: monday - 70, indicator: "triages_started", value: 9,
                                   suppressed: false, published_at: Time.current)
    allow(Maintenance::CityReader).to receive(:call).and_call_original

    gql!(indicators_query, slug: city.slug, from: (monday - 7).iso8601, to: (monday + 6).iso8601)

    expect(json["errors"]).to be_nil
    expect(json.dig("data", "city", "analyticsIndicators")).to eq([
      { "weekStart" => monday.iso8601, "indicator" => "no_show_pct", "value" => nil, "suppressed" => true },
      { "weekStart" => monday.iso8601, "indicator" => "triages_started", "value" => 40.0, "suppressed" => false }
    ])
    expect(Maintenance::CityReader).not_to have_received(:call)
  end

  it "analyticsIndicators com from depois de to ou mais de 104 semanas: INVALID_RANGE" do
    gql!(indicators_query, slug: city.slug, from: monday.iso8601, to: (monday - 7).iso8601)
    expect(json["errors"].first.dig("extensions", "code")).to eq("INVALID_RANGE")

    gql!(indicators_query, slug: city.slug, from: (monday - 7 * 104).iso8601, to: monday.iso8601)
    expect(json["errors"].first.dig("extensions", "code")).to eq("INVALID_RANGE")
  end

  it "analyticsStatus: estado do pipeline lido de analytics_runs da cidade" do
    run = consolidated_run!(finished_at: 2.hours.ago)

    gql!('query($slug: String!) { city(slug: $slug) { analyticsStatus { lastRunStatus lastSucceededAt lastPublishedAt lastError stale } } }',
         slug: city.slug)

    expect(json["errors"]).to be_nil
    status = json.dig("data", "city", "analyticsStatus")
    expect(status).to include("lastRunStatus" => "succeeded", "lastError" => nil, "stale" => false)
    expect(Time.iso8601(status["lastSucceededAt"])).to be_within(1.second).of(run.finished_at)
    expect(Time.iso8601(status["lastPublishedAt"])).to be_within(1.second).of(run.published_at)
  end

  it "cidade inalcançável: analyticsStatus nulo com erro de campo, e o resto de `city` responde" do
    allow(Maintenance::CityReader).to receive(:call).and_raise(Maintenance::CityReader::Unreachable, "PG::ConnectionBad: x")

    gql!('query($slug: String!) { city(slug: $slug) { slug analyticsStatus { stale } } }', slug: city.slug)

    expect(json.dig("data", "city", "slug")).to eq(city.slug)
    expect(json.dig("data", "city", "analyticsStatus")).to be_nil
    expect(json["errors"].first.dig("extensions", "code")).to eq("CITY_UNREACHABLE")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/requests/maintenance/city_analytics_spec.rb`
Expected: FAIL com erro GraphQL `Field 'analyticsIndicators' doesn't exist on type 'City'`.

- [ ] **Step 3: Implemente**

```ruby
# app/graphql/maintenance/types/analytics_indicator_type.rb
module Maintenance
  module Types
    class AnalyticsIndicatorType < BaseObject
      description "Indicador semanal publicado da cidade inteira (ADR 0025), lido do banco de plataforma. " \
                  "Suprimido (1 a 4, ou taxa com numerador/denominador nessa faixa) = value nulo."

      field :week_start, GraphQL::Types::ISO8601Date, null: false
      field :indicator, String, null: false
      field :value, Float, null: true
      field :suppressed, Boolean, null: false
    end
  end
end
```

```ruby
# app/graphql/maintenance/types/analytics_status_type.rb
module Maintenance
  module Types
    class AnalyticsStatusType < BaseObject
      description "Estado da consolidação do Analytics da cidade (analytics_runs). lastError é classe e primeira " \
                  "linha da mensagem, nunca payload."

      field :last_run_status, String, null: true
      field :last_succeeded_at, GraphQL::Types::ISO8601DateTime, null: true
      field :last_published_at, GraphQL::Types::ISO8601DateTime, null: true
      field :last_error, String, null: true
      field :stale, Boolean, null: false
    end
  end
end
```

Em `app/graphql/maintenance/types/city_type.rb`, depois de `field :operations, Types::CityOperationsType, null: true`:

```ruby

      # Módulo 14 (ADR 0025; contratos §3). analyticsIndicators lê o banco de
      # PLATAFORMA — nunca abre a cidade; analyticsStatus lê analytics_runs da
      # cidade pelo `inside`, anulável como counts/operations: cidade
      # inalcançável anula só ele.
      field :analytics_indicators, [ Types::AnalyticsIndicatorType ], null: false do
        argument :from, GraphQL::Types::ISO8601Date, required: true
        argument :to, GraphQL::Types::ISO8601Date, required: true
      end
      field :analytics_status, Types::AnalyticsStatusType, null: true
```

e, depois de `def operations ... end`:

```ruby

      def analytics_indicators(from:, to:)
        weeks = Analytics::CityIndicatorsQuery.weeks(from.iso8601, to.iso8601, Time.zone.today)
        CityAnalyticsIndicator.where(city_id: object.id, week_start: weeks).order(:week_start, :indicator).map do |row|
          { week_start: row.week_start, indicator: row.indicator, value: row.value&.to_f, suppressed: row.suppressed }
        end
      rescue Analytics::Params::Invalid
        raise GraphQL::ExecutionError.new("intervalo inválido (from <= to, até 104 semanas)",
                                          extensions: { "code" => "INVALID_RANGE" })
      end

      def analytics_status = inside { Analytics::Status.call.to_h }
```

Em `spec/architecture/maintenance_schema_spec.rb`, em `EXPECTED_TYPES`:
- `"City"` ganha `analyticsIndicators analyticsStatus` ao fim da lista;
- duas entradas novas:

```ruby
    "AnalyticsIndicator" => %w[weekStart indicator value suppressed],
    "AnalyticsStatus" => %w[lastRunStatus lastSucceededAt lastPublishedAt lastError stale],
```

- [ ] **Step 4: Rode e veja passar, com as guardas do GraphQL**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/requests/maintenance spec/architecture/maintenance_schema_spec.rb spec/graphql`
Expected: PASS (nenhum nome novo cai nos fragmentos proibidos da guarda).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add app/graphql/maintenance/types/analytics_indicator_type.rb app/graphql/maintenance/types/analytics_status_type.rb app/graphql/maintenance/types/city_type.rb spec/architecture/maintenance_schema_spec.rb spec/requests/maintenance/city_analytics_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "feat: expose analytics indicators and pipeline status on the maintenance city type

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

## Fechamento — invariantes, semente, runbook e revisão

### Task 19: Suíte de invariantes com teste de mutação (ADR 0025)

**Files:**
- Create: `spec/invariants/analytics_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 1–18.
- Produces: uma seção por invariante do ADR 0025, cada uma com a mutação que precisa deixá-la vermelha.

- [ ] **Step 1: Escreva a suíte**

```ruby
# spec/invariants/analytics_invariants_spec.rb
# Módulo 14, critério de fechamento (ADR 0025 "Invariantes"; spec 2026-09-30
# §10.2). Cada bloco tem a mutação que precisa deixá-lo vermelho (registrada
# no relatório da entrega).
require "rails_helper"

RSpec.describe "Invariantes do Analytics (ADR 0025)", type: :request do
  let!(:city_record) { register_test_city! }
  let(:today) { Time.zone.today }
  let(:centro) { Neighborhood.create!(name: "Centro", source: "manual") }
  let(:unit) { create_unit("UBS Invariante") }
  let(:operator_password) { "s3nha-forte-1" }

  before { create_default_protocol! }

  def json = JSON.parse(response.body)

  def snapshot
    AnalyticsDailyFact.order(:day, :metric, :dim, :tier, :neighborhood_id, :health_unit_id, :question_id)
                      .pluck(:day, :metric, :health_unit_id, :neighborhood_id, :protocol_name, :protocol_version,
                             :tier, :question_id, :dim, :value)
  end

  # Inteiros do JSON que são contagem: protocol_version e o eco do filtro não são.
  def counts_in(node, &block)
    case node
    when Hash then node.each { |key, value| counts_in(value, &block) unless %w[protocol_version filter].include?(key) }
    when Array then node.each { |value| counts_in(value, &block) }
    when Integer then yield node
    end
  end

  def operator!
    Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: operator_password,
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end

  # Mutação: acrescentar `t.uuid :citizen_id` ao create_table de analytics_daily_facts
  # (migração e dump) e recarregar os bancos de teste.
  it "analytics_daily_facts, analytics_runs e city_analytics_indicators não têm coluna de pessoa" do
    expect(AnalyticsDailyFact.column_names).to match_array(%w[id day metric health_unit_id neighborhood_id protocol_name
                                                              protocol_version tier question_id dim value consolidated_at])
    expect(AnalyticsRun.column_names).to match_array(%w[id window_from window_to kind status started_at finished_at
                                                        published_at error])
    expect(CityAnalyticsIndicator.column_names).to match_array(%w[id city_id week_start indicator value suppressed
                                                                  published_at])
  end

  # Mutação: em Analytics::Suppression.cell, devolver `count.to_i` sem o wrap; ou, em
  # Analytics::Publish#indicators, trocar `Suppression.cell(total(week, "triage.started"))`
  # por `total(week, "triage.started")`.
  it "nenhuma resposta do Analytics e nenhuma linha publicada traz contagem de 1 a 4" do
    monday = (today - 21).beginning_of_week
    ProtocolDefinition.create!(name: "arbo", version: 1, status: "active", definition: analytics_definition(name: "arbo"))
    (0..13).each do |offset|
      day = monday + offset
      small = (offset % 4) + 1
      triage = { protocol_name: "arbo", protocol_version: 1 }
      fact!(metric: "triage.started", day: day, value: small, neighborhood_id: offset.even? ? centro.id : nil, **triage)
      fact!(metric: "triage.completed", day: day, value: small, neighborhood_id: centro.id, tier: "alta", **triage)
      fact!(metric: "triage.aborted", day: day, value: small, dim: "timeout", **triage)
      fact!(metric: "calibration.outcome", day: day, value: small, tier: "alta", dim: "none", **triage)
      fact!(metric: "epi.answer", day: day, value: small, neighborhood_id: centro.id, question_id: "febre", dim: "true", **triage)
      %w[attendance.closed attendance.wait appointment.ended request.opened attendance.checked_in].zip(%w[left 0-15 no_show return code])
        .each { |metric, dim| fact!(metric: metric, day: day, value: small, health_unit_id: unit.id, dim: dim) }
    end
    fact!(metric: "triage.started", day: monday - 7, value: 2, protocol_name: "arbo", protocol_version: 1)
    consolidated_run!
    sign_in_as(staff_with("analise@cidade.gov.br", "analyst"))

    found = []
    base = { from: (monday - 7).iso8601, to: (monday + 13).iso8601 }
    filters = [ {}, { neighborhood_id: centro.id }, { neighborhood_id: "none" }, { health_unit_id: unit.id },
                { protocol_name: "arbo", protocol_version: "1" } ]
    [ {}, { granularity: "month" } ].product(filters).each do |granularity, filter|
      %w[demand quality calibration epidemiology].each do |front|
        get "/admin/api/analytics/#{front}", params: base.merge(granularity).merge(filter)
        expect(response).to have_http_status(:ok), "#{front} #{granularity} #{filter}: #{response.body}"
        counts_in(json["data"]) { |n| found << [ front, granularity, filter, n ] if (1..4).cover?(n) }
      end
    end
    expect(found).to be_empty

    Analytics::Publish.call(from: monday - 7, to: monday + 13)
    rows = CityAnalyticsIndicator.where(city_id: city_record.id)
    expect(rows.where(indicator: CityAnalyticsIndicator::COUNT_INDICATORS, value: 1..4)).to be_empty
    expect(rows.where(suppressed: true).where.not(value: nil)).to be_empty
    expect(rows.find_by!(week_start: monday - 7, indicator: "triages_started")).to have_attributes(suppressed: true, value: nil)
  end

  # Mutação: em Analytics::Consolidate::Epidemiology, tirar
  # `AND s.step -> 'analytic' = 'true'::jsonb`; ou trocar a condição de answer_type por `TRUE`.
  it "nenhum fato epidemiológico vem de pergunta sem analytic, nem de integer ou text" do
    definition = analytics_definition(marks: %w[febre sintoma idade])
    definition["steps"] << { "id" => "obs", "prompt" => "Algo mais?", "answer_type" => "text", "analytic" => true,
                             "branches" => {} }
    protocol = ProtocolDefinition.create!(name: "triagem-arbovirose", version: 1, status: "active", definition: definition)
    a_triage!(day: today - 2, protocol: protocol,
              answers: { "febre" => "true", "sintoma" => "Manchas", "gestante" => "true", "idade" => "34", "obs" => "true" })

    scheduled_run!

    expect(AnalyticsDailyFact.where(metric: "epi.answer").distinct.pluck(:question_id)).to match_array(%w[febre sintoma])
  end

  # Mutação: em Analytics::Consolidate.call, apagar a linha
  # `AnalyticsDailyFact.where(day: from..to).delete_all`.
  it "consolidar a mesma janela duas vezes produz os mesmos fatos" do
    protocol = analytics_protocol!
    an_attendance!(triage: a_triage!(day: today - 4, protocol: protocol, neighborhood: centro,
                                     answers: { "febre" => "true" }), unit: unit, wait_minutes: 45)
    a_triage!(day: today - 6, status: "aborted_by_cancellation")

    scheduled_run!
    first = snapshot
    scheduled_run!

    expect(AnalyticsRun.pluck(:status)).to eq(%w[succeeded succeeded])
    expect(snapshot).to eq(first)
    expect(first).not_to be_empty
  end

  # Mutação: em Analytics::Consolidate.call, trocar
  # `AnalyticsDailyFact.where(day: from..to).delete_all` por `AnalyticsDailyFact.delete_all`.
  it "a revogação não altera fato fora da janela; dentro dela, a pessoa sai na próxima execução" do
    protocol = analytics_protocol!
    old = a_triage!(day: today - 40, protocol: protocol, neighborhood: centro, answers: { "febre" => "true" })
    recent = a_triage!(day: today - 5, protocol: protocol, neighborhood: centro, answers: { "febre" => "true" })
    Analytics::Run.call(kind: "rebuild", from: today - 40, to: today - 31)
    scheduled_run!
    outside = AnalyticsDailyFact.where(day: today - 40).pluck(:metric, :dim, :neighborhood_id, :value).sort
    expect(outside).to include([ "epi.answer", "true", centro.id, 1 ])

    [ old, recent ].each do |triage|
      RevokeConsent.call(conversation: triage.conversation, reason: "citizen_web")
      AnonymizeRevokedTriageJob.new.handle(conversation_id: triage.conversation_id)
    end
    scheduled_run!

    expect(AnalyticsDailyFact.where(day: today - 40).pluck(:metric, :dim, :neighborhood_id, :value).sort).to eq(outside)
    expect(AnalyticsDailyFact.where(day: today - 5, metric: %w[triage.completed calibration.outcome epi.answer])).to be_empty
    expect(AnalyticsDailyFact.where(day: today - 5, metric: "triage.aborted").pluck(:dim, :neighborhood_id))
      .to eq([ [ "revocation", nil ] ])
  end

  # Mutação: em Analytics::Run#purge, trocar `...(Time.zone.today - FACT_RETENTION)`
  # por `..(Time.zone.today - FACT_RETENTION)`; ou FACT_RETENTION = 1.year.
  it "a purga apaga só fatos com mais de 5 anos" do
    older = fact!(metric: "triage.started", day: today - 5.years - 1, value: 9)
    edge = fact!(metric: "triage.started", day: today - 5.years, value: 9)
    year_ago = fact!(metric: "triage.started", day: today - 400, value: 9)

    scheduled_run!

    expect(AnalyticsDailyFact.where(id: [ older.id, edge.id, year_ago.id ]).pluck(:id))
      .to contain_exactly(edge.id, year_ago.id)
  end

  # Mutação: acrescentar "viewer" a Admin::Api::AnalyticsController::ROLES; ou, no
  # AnalyticsController, apagar o `render ... if Current.session.operator_grant?` de
  # require_authentication e fazer require_analytics_role aceitar `current_user.nil?`.
  it "só analyst e municipal_admin da cidade leem /admin/api/analytics; o operador, com ou sem grant, não" do
    consolidated_run!
    range = { from: (today - 14).iso8601, to: (today - 1).iso8601 }
    Membership::ROLES.each do |role|
      sign_in_as(staff_with("#{role}-#{SecureRandom.hex(2)}@cidade.gov.br", role))
      get "/admin/api/analytics/demand", params: range
      expect(response.status).to eq(%w[analyst municipal_admin].include?(role) ? 200 : 403), role
    end

    sign_in_operator_grant(operator!)
    get "/admin/api/analytics/demand", params: range
    expect(response).to have_http_status(:forbidden)

    host! "admin.rotasaude.app"
    get "/admin/api/analytics/demand", params: range
    expect(response).to have_http_status(:not_found)
  end

  # Mutação: em Analytics::CityIndicatorsQuery#call, calcular last_published_at com
  # `CityConnection.with(city) { AnalyticsRun.maximum(:published_at) }`; ou, em
  # CityType#analytics_indicators, envolver a leitura em `inside { ... }`.
  it "o console do operador e o maintenance nunca abrem o banco da cidade para ler indicador" do
    CityAnalyticsIndicator.create!(city: city_record, week_start: (today - 14).beginning_of_week,
                                   indicator: "triages_started", value: 40, suppressed: false, published_at: Time.current)
    operator = operator!
    host! "admin.rotasaude.app"
    post "/session", params: { email_address: operator.email_address, password: operator_password }
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(operator.otp_secret).now }
    maintainer = Maintainer.create!(email_address: "mt-#{SecureRandom.hex(3)}@rotasaude.app", password: operator_password,
                                    otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
    frontend = "https://maintenance.rotasaude.app"
    browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
    allow(CityConnection).to receive(:with).and_call_original

    get "/city_analytics"
    expect(response).to have_http_status(:ok)

    host! "maintenance-api.rotasaude.app"
    post "/session", params: { email_address: maintainer.email_address, password: operator_password }, headers: browser
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now },
         headers: browser
    query = 'query($slug: String!, $from: ISO8601Date!, $to: ISO8601Date!) { city(slug: $slug) { analyticsIndicators(from: $from, to: $to) { value } } }'
    post "/graphql", params: { query: query, variables: { slug: city_record.slug, from: (today - 21).iso8601,
                                                          to: (today - 1).iso8601 }.to_json }, headers: browser
    expect(json["errors"]).to be_nil
    expect(json.dig("data", "city", "analyticsIndicators")).to eq([ { "value" => 40.0 } ])

    expect(CityConnection).not_to have_received(:with)
  end
end
```

- [ ] **Step 2: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/invariants/analytics_invariants_spec.rb`
Expected: PASS.

- [ ] **Step 3: Teste de mutação**

Para cada comentário `# Mutação:` da suíte: aplique a mutação, rode só aquele bloco, confirme que fica **vermelho** e desfaça (`/opt/homebrew/bin/git -C apps/api/.claude/mod14 checkout -- <arquivo>`; para a coluna nova, recarregue com `bin/rails city:test_databases` antes e depois). Registre no relatório uma linha por mutação: arquivo, mudança, exemplo que falhou, mensagem. Mutação que passar verde = a spec não prova a invariante: conserte a spec antes de seguir.

- [ ] **Step 4: Regressão, suíte completa e custo por exemplo**

```bash
docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/invariants spec/architecture spec/events
docker compose stop worker
docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec --profile 10
docker compose start worker
```
Expected: `spec/invariants`, `spec/architecture` (inclusive `current_city_assignment_spec.rb`, `operator_grant_access_spec.rb`, `maintenance_schema_spec.rb`) e `spec/events` (guarda R18 — nenhum `Platform.audit` novo) verdes; suíte completa com 0 falhas. Custo: "Finished in N seconds" ÷ exemplos perto de **0,10 s/exemplo** com o host calmo; acima de ~0,12 s, investigue pelo `--profile` (candidatos: `staff_with` em laço de papéis, `an_attendance!` em laço).

- [ ] **Step 5: Busca por data fixa contra o relógio real**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 diff origin/main --name-only -- spec | sed 's#^#apps/api/.claude/mod14/#' | xargs grep -nE "Time\.zone\.parse\(\"20|Date\.new\(20|travel_to\(\"20" || true
```
Expected: nenhuma ocorrência (a única data literal permitida é o `"2026-02-30"` inválido de propósito).

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add spec/invariants/analytics_invariants_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "test: add the analytics invariants suite with mutation evidence

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 20: Semente de dev realista (spec §11)

**Files:**
- Create: `lib/analytics_crew.rb`
- Modify: `db/seeds.rb`
- Test: `spec/lib/analytics_crew_spec.rb`

**Interfaces:**
- Consumes: `AnalyticsHistory` (Task 3); `CampaignHistory.cpf_for`; `SignatureCrew.ensure_totp/.otpauth_uri` e as contas `autor`, `revisora1`, `revisora2`, `publisher`, `recepcao` que ele cria; `Protocols::SaveDraft`, `SubmitForReview`, `Sign`, `Publish`, `Activate`; bairros da `Territory::Seed` e unidades do `ProfessionalCrew` (rodam antes no `db/seeds.rb`); `Analytics::Rebuild` (Task 11).
- Produces: `AnalyticsCrew.seed_current_city(slug:, ddd:, password:, days: AnalyticsCrew::DAYS) → { account: { email:, role:, otpauth_uri: }, protocol: { name:, version:, status: }, new_triages: Integer, runs: Integer, failed: String | nil }`. Conta `analise@<slug>.demo` (`analyst`, TOTP fixo, env `DEV_ANALYST_OTP_SECRET`); protocolo `triagem-arbovirose` v1 ativo pelo ciclo assinado (febre `boolean` e sintoma `enum` marcados; gestação sem marca); ~6 meses de triagens, atendimentos, pedidos e horários no passado (mais procura em dia útil e na estação das arboviroses; 10 bairros, os três primeiros mais cheios; 1 em 10 sem bairro; ~2% revogadas depois de concluir); ao fim, `Analytics::Rebuild` do período.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/lib/analytics_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/signature_crew")
require Rails.root.join("lib/analytics_crew")

RSpec.describe AnalyticsCrew do
  let!(:city_record) { register_test_city! }

  before do
    create_default_protocol!
    %w[Centro Batel Portão Boqueirão].each { |name| Neighborhood.create!(name: name, source: "seed") }
    [ [ "UBS Jardim das Flores", "ubs" ], [ "UBS Vila Esperança", "ubs" ], [ "UPA 24h Centro", "upa" ] ]
      .each { |name, kind| create_unit(name, kind: kind) }
    SignatureCrew.seed_current_city(slug: "curitiba", password: "dev-password")
  end

  def seed = described_class.seed_current_city(slug: "curitiba", ddd: "41", password: "dev-password", days: 21)

  it "cria a conta analyst, leva o protocolo marcado pelo ciclo assinado e consolida três semanas de histórico" do
    result = seed

    user = User.find_by!(email_address: "analise@curitiba.demo")
    expect(user).to be_mfa_enrolled
    expect(user.has_role?(:analyst)).to be(true)
    expect(result[:account]).to include(email: "analise@curitiba.demo", role: "analyst")
    expect(result[:protocol]).to eq(name: "triagem-arbovirose", version: 1, status: "active")
    protocol = ProtocolDefinition.find_by!(name: "triagem-arbovirose", version: 1)
    expect(protocol.definition["steps"].select { |s| s["analytic"] }.map { |s| s["answer_type"] }).to eq(%w[boolean enum])
    expect(ProtocolSignature.where(protocol_definition: protocol).pluck(:purpose).tally)
      .to eq("publication" => 2, "activation" => 2)

    expect(result[:new_triages]).to be > 50
    expect(result[:failed]).to be_nil
    expect(AnalyticsRun.where(kind: "rebuild", status: "succeeded")).to exist
    expect(AnalyticsDailyFact.where(metric: "epi.answer").distinct.pluck(:question_id)).to match_array(%w[febre sintoma])
    %w[triage.started triage.completed attendance.checked_in attendance.closed attendance.wait calibration.outcome]
      .each { |metric| expect(AnalyticsDailyFact.where(metric: metric)).to exist, metric }
    expect(CityAnalyticsIndicator.where(city_id: city_record.id)).to exist
    expect(Triage.where("created_at >= ?", Time.zone.today.beginning_of_day)).to be_empty
    expect(Citizen.all.map(&:cpf)).to all(satisfy { |cpf| CitizenIdentity::Cpf.normalize(cpf) == cpf })
  end

  it "é idempotente" do
    seed
    counts = -> { [ Triage.count, Attendance.count, AppointmentRequest.count, Appointment.count, ProtocolDefinition.count ] }
    before = counts.call

    expect(seed[:new_triages]).to eq(0)
    expect(counts.call).to eq(before)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/lib/analytics_crew_spec.rb`
Expected: FAIL com `cannot load such file -- /rails/.claude/mod14/lib/analytics_crew`.

- [ ] **Step 3: Implemente a semente**

```ruby
# lib/analytics_crew.rb
require "zlib"
require_relative "signature_crew"
require_relative "campaign_history"
require_relative "analytics_history"

# Semente de dev do módulo 14 (spec 2026-09-30 §11). Dev é fictício, mas imita
# o real: bairros da semente de território, unidades do ProfessionalCrew, CPF
# com dígito válido, respostas que seguem os ramos do protocolo, tier pelos
# pesos, desfecho, pedido e horário encadeados. Seis meses para trás, com mais
# procura em dia útil e na estação das arboviroses (março a maio).
#
# O protocolo com perguntas marcadas passa pelo ciclo assinado de verdade
# (desvio 10 do plano): SaveDraft → SubmitForReview → 2 assinaturas → Publish
# → 2 assinaturas → Activate. O histórico no passado é gravado pelo
# AnalyticsHistory (os comandos só aceitam "agora"); ao fim, Analytics::Rebuild
# consolida o período. Idempotente; nada termina no dia corrente.
class AnalyticsCrew
  SECRET_ENV = "DEV_ANALYST_OTP_SECRET"
  DEFAULT_SECRET = "MFXGC3DJON2GKZLOMFXGC3DJON2GKZLO"
  DAYS = 182
  CITIZENS = 240
  PHONE_PREFIX = "95555"
  PROTOCOL_NAME = "triagem-arbovirose"
  SYMPTOMS = [ "Dor atrás dos olhos", "Manchas vermelhas na pele", "Dor nas articulações", "Nenhum destes" ].freeze
  PROTOCOL = {
    "name" => PROTOCOL_NAME, "version" => 1, "start_step_id" => "febre",
    "steps" => [
      { "id" => "febre", "prompt" => "Teve febre nos últimos 7 dias?", "answer_type" => "boolean", "analytic" => true,
        "branches" => { "true" => "sintoma", "false" => "sintoma" }, "weights" => { "true" => 3, "false" => 0 } },
      { "id" => "sintoma", "prompt" => "Qual destes sintomas está mais forte?", "answer_type" => "enum",
        "analytic" => true, "options" => SYMPTOMS, "branches" => SYMPTOMS.to_h { |symptom| [ symptom, "gestante" ] },
        "weights" => { "Dor atrás dos olhos" => 2, "Manchas vermelhas na pele" => 3, "Dor nas articulações" => 2,
                       "Nenhum destes" => 0 } },
      # Pergunta sensível SEM marca: nunca vira agregado (ADR 0025).
      { "id" => "gestante", "prompt" => "Está gestante?", "answer_type" => "boolean",
        "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 3, "false" => 0 } }
    ],
    "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "media" => 3, "alta" => 6 },
                   "priority_map" => { "baixa" => 9, "media" => 5, "alta" => 1 } },
    "recommendations" => {
      "alta" => { "title" => "Procure atendimento hoje",
                  "body" => "Vá à unidade de saúde mais próxima ainda hoje e beba bastante líquido. Sangramento, dor forte na barriga ou tontura → 192." },
      "media" => { "title" => "Procure sua unidade de saúde",
                   "body" => "Procure sua UBS em até 24 horas. Beba bastante líquido e evite anti-inflamatórios." },
      "baixa" => { "title" => "Cuidados em casa",
                   "body" => "Repouso e hidratação. Se aparecerem febre ou manchas na pele, procure sua UBS." }
    }
  }.freeze

  # Procura por dia da semana (domingo = 0).
  WEEKDAY = [ 4, 14, 12, 11, 11, 10, 5 ].freeze
  BOOLEAN_TRUE = { "febre" => 0.55, "gestante" => 0.06, "tosse" => 0.6 }.freeze
  SYMPTOM_WEIGHTS = [ 3, 2, 3, 4 ].freeze
  # [minutos de espera base, peso]; a espera final ganha 0 a 9 min.
  WAIT_MINUTES = [ [ 6, 35 ], [ 20, 25 ], [ 40, 20 ], [ 80, 12 ], [ 140, 8 ] ].freeze
  OUTCOMES = [ [ "discharged", 60 ], [ "referred", 12 ], [ "return", 18 ], [ "left", 10 ] ].freeze
  APPOINTMENT_ENDINGS = [ [ "checked_in", 65 ], [ "no_show", 15 ], [ "expired", 10 ],
                          [ "cancelled_by_citizen", 10 ] ].freeze

  class << self
    def seed_current_city(slug:, ddd:, password:, days: DAYS)
      account = ensure_analyst(slug: slug, password: password)
      protocol = ensure_protocol!(slug)
      new_triages = seed_history!(slug: slug, ddd: ddd, days: days)
      report = Analytics::Rebuild.call(from: Time.zone.today - days)
      { account: account, protocol: { name: protocol.name, version: protocol.version, status: protocol.status },
        new_triages: new_triages, runs: report.runs.size,
        failed: report.failed&.error || (report.busy ? "outra consolidação em curso" : nil) }
    end

    private

    def ensure_analyst(slug:, password:)
      user = User.find_or_initialize_by(email_address: "analise@#{slug}.demo")
      user.password = password
      user.save!
      Membership.find_or_create_by!(user: user, role: "analyst") { |m| m.granted_at = Time.current }
      SignatureCrew.ensure_totp(user, secret_env: SECRET_ENV, default_secret: DEFAULT_SECRET)
      { email: user.email_address, role: "analyst", otpauth_uri: SignatureCrew.otpauth_uri(user) }
    end

    # Retoma de onde parou: cada passo só roda se a versão estiver no estado dele.
    def ensure_protocol!(slug)
      record = ProtocolDefinition.find_by(name: PROTOCOL_NAME, version: 1)
      return record if record&.status == "active"

      author, first, second, publisher = %w[autor revisora1 revisora2 publisher]
                                         .map { |prefix| User.find_by!(email_address: "#{prefix}@#{slug}.demo") }
      status = -> { ProtocolDefinition.find_by!(name: PROTOCOL_NAME, version: 1).status }
      check!(Protocols::SaveDraft.call(definition: PROTOCOL.deep_dup, by: author), "rascunho") if record.nil?
      check!(Protocols::SubmitForReview.call(name: PROTOCOL_NAME, version: 1, by: author), "revisão") if status.call == "draft"
      if status.call == "in_review"
        [ first, second ].each { |reviewer| sign!(reviewer, "publication") }
        check!(Protocols::Publish.call(name: PROTOCOL_NAME, version: 1, by: publisher), "publicação")
      end
      if status.call == "published"
        [ first, second ].each { |reviewer| sign!(reviewer, "activation") }
        check!(Protocols::Activate.call(name: PROTOCOL_NAME, version: 1, by: publisher), "ativação")
      end
      ProtocolDefinition.find_by!(name: PROTOCOL_NAME, version: 1)
    end

    def sign!(reviewer, purpose)
      result = Protocols::Sign.call(name: PROTOCOL_NAME, version: 1, purpose: purpose, by: reviewer)
      check!(result, "assinatura de #{purpose}") unless result.reason == :already_signed
    end

    def check!(result, step)
      return result if result.ok?

      raise "semente do Analytics: #{step} de #{PROTOCOL_NAME} falhou — #{result.reason}: #{result.message}"
    end

    # 0 quando o histórico já existe (idempotência pelo protocolo da semente).
    def seed_history!(slug:, ddd:, days:)
      return 0 if Triage.where(protocol_name: PROTOCOL_NAME).exists?

      rng = Random.new(Zlib.crc32("analytics:#{slug}"))
      by = User.find_by!(email_address: "recepcao@#{slug}.demo")
      units = HealthUnit.where(active: true).order(:name).to_a
      raise "semente do Analytics: nenhuma unidade ativa (rode o ProfessionalCrew antes)" if units.empty?

      neighborhoods = Neighborhood.where(active: true).order(:name).to_a.sample(10, random: rng)
      raise "semente do Analytics: nenhum bairro (rode a Territory::Seed antes)" if neighborhoods.empty?

      protocols = [ ProtocolDefinition.find_by!(name: PROTOCOL_NAME, status: "active"),
                    ProtocolDefinition.find_by!(name: StartTriage::DEFAULT_PROTOCOL_NAME, status: "active") ]
      weighted_places = neighborhoods.each_with_index.map { |place, index| [ place, index < 3 ? 4 : 1 ] }
      citizens = Array.new(CITIZENS) do |index|
        neighborhood = rng.rand < 0.1 ? nil : weighted(rng, weighted_places)
        AnalyticsHistory.citizen!(cpf: CampaignHistory.cpf_for("#{slug}:analytics:#{index}"),
                                  phone: format("+55%s#{PHONE_PREFIX}%04d", ddd, index + 1), neighborhood: neighborhood)
      end

      count = 0
      ApplicationRecord.transaction do
        days.downto(1) do |ago|
          date = Time.zone.today - ago
          daily_count(date, rng).times do
            one_triage!(date, rng: rng, citizens: citizens, protocols: protocols, units: units, by: by)
            count += 1
          end
        end
      end
      count
    end

    def daily_count(date, rng)
      season = if date.month.between?(3, 5) then 1.5
               elsif date.month == 6 then 1.2
               else 0.9
               end
      (WEEKDAY[date.wday] * season * (0.8 + 0.4 * rng.rand)).round
    end

    def one_triage!(date, rng:, citizens:, protocols:, units:, by:)
      citizen = citizens[rng.rand(citizens.size)]
      protocol = rng.rand < 0.6 ? protocols.first : protocols.last
      at = Time.zone.local(date.year, date.month, date.day, 7 + rng.rand(13), rng.rand(60))
      answers = walk(protocol.definition, rng)
      roll = rng.rand
      status, revoked = if roll < 0.84 then [ "completed", false ]
                        elsif roll < 0.91 then [ "aborted_by_timeout", false ]
                        elsif roll < 0.97 then [ "aborted_by_cancellation", false ]
                        elsif roll < 0.99 then [ "completed", true ]
                        else [ "aborted_by_revocation", false ]
                        end
      tier = tier_for(protocol.definition, answers)
      given = status == "completed" ? answers : answers.first(rng.rand(answers.size)).to_h
      triage = AnalyticsHistory.triage!(citizen: citizen, protocol: protocol, created_at: at, status: status,
                                        revoked: revoked, tier: tier, answers: given,
                                        priority: protocol.definition.dig("scoring", "priority_map", tier) || 5)
      attend!(triage, rng: rng, units: units, by: by) if status == "completed" && !revoked && rng.rand < 0.7
    end

    def attend!(triage, rng:, units:, by:)
      unit = weighted(rng, units.each_with_index.map { |place, index| [ place, index.zero? ? 3 : 2 ] })
      checked_in_at = triage.completed_at + (20 + rng.rand(240)).minutes
      wait = weighted(rng, WAIT_MINUTES) + rng.rand(10)
      return if checked_in_at + (wait + 30).minutes >= Time.zone.now.beginning_of_day # nada termina hoje

      outcome = weighted(rng, OUTCOMES)
      other = units.find { |place| place != unit } || unit
      attendance = AnalyticsHistory.attendance!(citizen: triage.conversation.citizen, triage: triage, unit: unit, by: by,
                                                checked_in_at: checked_in_at, wait_minutes: wait, outcome: outcome,
                                                referral_unit: outcome == "referred" ? other : nil)
      follow_up!(attendance, rng: rng, by: by, other: other) if %w[return referred].include?(outcome)
    end

    # Pedido do desfecho e, quando o horário já passou, o fim dele: comparecimento
    # fecha o pedido (e gera o atendimento do retorno); cancelamento fecha; falta
    # e expiração reabrem, e parte dos reabertos é dispensada três dias depois.
    def follow_up!(attendance, rng:, by:, other:)
      kind = attendance.outcome == "return" ? "return" : "referral"
      scheduled_at = (attendance.closed_at + (5 + rng.rand(15)).days).change(hour: 8 + rng.rand(9), min: 0)
      today_start = Time.zone.now.beginning_of_day
      if scheduled_at + 1.day >= today_start
        AnalyticsHistory.request!(origin: attendance, kind: kind, target: other, by: by)
        return
      end

      ending = weighted(rng, APPOINTMENT_ENDINGS)
      ended_at = AnalyticsHistory.ended_at(ending, scheduled_at)
      closed_reason = { "checked_in" => "fulfilled", "cancelled_by_citizen" => "citizen_cancelled" }[ending]
      closed_at = closed_reason && ended_at
      if closed_reason.nil? && rng.rand < 0.4 && ended_at + 3.days < today_start
        closed_reason = "dismissed"
        closed_at = ended_at + 3.days
      end
      request = AnalyticsHistory.request!(origin: attendance, kind: kind, target: other, by: by,
                                          status: closed_reason ? "closed" : "open", closed_reason: closed_reason,
                                          closed_at: closed_at,
                                          reopened_reason: %w[no_show expired].include?(ending) ? ending : nil)
      appointment = AnalyticsHistory.appointment!(request: request, status: ending, scheduled_at: scheduled_at, by: by)
      return unless ending == "checked_in"

      AnalyticsHistory.attendance!(citizen: request.citizen, appointment: appointment, unit: request.target_unit, by: by,
                                   checked_in_at: appointment.ended_at, wait_minutes: weighted(rng, WAIT_MINUTES))
    end

    # Percorre o protocolo como o cidadão: resposta sorteada, próximo passo pelo ramo.
    def walk(definition, rng)
      steps = definition.fetch("steps").index_by { |step| step["id"] }
      answers = {}
      id = definition["start_step_id"]
      while id && (step = steps[id]) && !answers.key?(id)
        answer = if step["answer_type"] == "enum"
                   weighted(rng, step["options"].each_with_index.map { |option, i| [ option, SYMPTOM_WEIGHTS[i] || 1 ] })
                 else
                   rng.rand < BOOLEAN_TRUE.fetch(id, 0.5) ? "true" : "false"
                 end
        answers[id] = answer
        id = step.fetch("branches", {})[answer]
      end
      answers
    end

    # Scoring weighted: a maior faixa cujo mínimo a soma dos pesos alcança.
    def tier_for(definition, answers)
      score = definition.fetch("steps").sum do |step|
        answers.key?(step["id"]) ? step.fetch("weights", {}).fetch(answers[step["id"]], 0).to_i : 0
      end
      definition.dig("scoring", "thresholds").to_h.select { |_tier, min| score >= min }.max_by { |_tier, min| min }&.first
    end

    def weighted(rng, pairs)
      mark = rng.rand * pairs.sum(&:last)
      pairs.each { |value, weight| return value if (mark -= weight).negative? }
      pairs.last.first
    end
  end
end
```

Em `db/seeds.rb`:
- no comentário do topo, depois do parágrafo do `SignatureCrew`, acrescente: `#     O analyst (analise@<slug>.demo) e ~6 meses de histórico consolidado para o Analytics vêm de \`lib/analytics_crew.rb\` (módulo 14).`;
- depois de `require Rails.root.join("lib/campaign_crew").to_s`: `require Rails.root.join("lib/analytics_crew").to_s`;
- depois do bloco das campanhas (antes de `puts "[seeds] cidade ...`):

```ruby

        # ── Analytics (módulo 14, spec 2026-09-30 §11) ────────────────────────
        # Depois do território, dos profissionais e do elenco do ciclo assinado:
        # usa bairros, unidades, recepção, autor, revisoras e publisher. A
        # primeira vez leva uns segundos por cidade (~6 meses de histórico).
        analytics = AnalyticsCrew.seed_current_city(slug: slug, ddd: ddd, password: password)
        puts "[seeds] analyst .......... #{analytics[:account][:email]} / #{password} + MFA → #{analytics[:account][:otpauth_uri]}"
        puts "[seeds] analytics .. #{analytics[:protocol][:name]} v#{analytics[:protocol][:version]} " \
             "(#{analytics[:protocol][:status]}), #{analytics[:new_triages]} triagens novas, " \
             "#{analytics[:runs]} blocos consolidados#{analytics[:failed] ? " — FALHOU: #{analytics[:failed]}" : ''}"
```

- [ ] **Step 4: Rode e veja passar; semeie o dev**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod14 api bundle exec rspec spec/lib/analytics_crew_spec.rb spec/lib/campaign_crew_spec.rb spec/lib/territory_crew_spec.rb
docker compose exec -T -w /rails/.claude/mod14 api bin/rails db:seed
```
Expected: PASS; o `db:seed` imprime a linha `analyst` e a de analytics para curitiba e maringa, sem `FALHOU`. Rode o `db:seed` de novo: `0 triagens novas`.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod14 add lib/analytics_crew.rb db/seeds.rb spec/lib/analytics_crew_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod14 commit -m "chore: seed an analyst, a marked protocol and six months of analytics history

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 21: Runbook `operacao/analytics.md` (repo docs, spec §12)

**Files (repo docs, worktree `docs/.claude/mod14`, branch `docs/mod-14-analytics`):**
- Create: `operacao/analytics.md`
- Modify: `operacao/README.md` (tabela de runbooks)

**Interfaces:**
- Consumes: nomes reais das Tasks 1, 2, 10, 11 e 18 (migrações, job, rake, campos do GraphQL).
- Produces: o runbook de rollout e operação do Analytics.

- [ ] **Step 1: Escreva o runbook**

`docs/.claude/mod14/operacao/analytics.md`:

````markdown
# Analytics — rollout e operação (módulo 14, ADR 0025)

## Quando aplicar

- No primeiro deploy do módulo 14 (api, dashboard, admin, maintenance).
- Quando a consolidação atrasar (`analyticsStatus.stale` no maintenance) ou falhar.
- Para corrigir um bug de consolidação (rebuild).

## Pré-requisitos

- Imagem do api com a migração de cidade `20260930200001_create_analytics` e a de
  plataforma `20260930200002_create_city_analytics_indicators`. As duas são **só
  aditivas**: tabelas novas e o papel `analyst` no CHECK de `memberships`.
- O worker de cada cidade na mesma imagem (o `ConsolidateAnalyticsJob` é
  recorrente de cidade, fila `housekeeping`, todo dia às 2h30 de São Paulo).
- `contracts` em `protocols-v1.2.0` (`analytic` na pergunta).

## Passos

1. **Migrar antes do tráfego.** Publicar a imagem nova do api e rodar, dela,
   `bin/rails db:migrate` (plataforma) e `bin/rails city:migrate:all` (cidades)
   **antes** de cortar o tráfego. Nunca migrar fora do rake: a cidade trava em 503.
2. **Ordem de deploy:** api → dashboard → admin → maintenance. Os três frontends
   leem rotas e campos novos (`/admin/api/analytics/*`, `GET /city_analytics`,
   `City.analyticsIndicators`/`analyticsStatus`); um frontend novo contra api
   antigo mostra erro nas telas de Analytics.
3. **Rebuild inicial**, depois do deploy do api:
   `bin/rails "city:analytics:rebuild:all"` — consolida cada cidade desde o dado
   cru mais antigo, em blocos de 30 dias, e publica as semanas na plataforma. O
   que já foi purgado do cru (eventos com mais de 12 meses, conteúdo de triagem
   revogada) não volta.
4. **Conceder o papel** "Análise" (`analyst`) em Equipe a quem vai ler o
   Analytics. Não pede step-up. O `municipal_admin` já lê.
5. **Marcar perguntas** para a Epidemiologia: o autor marca "Usar em Analytics"
   (só `boolean`/`enum`) numa versão nova do protocolo, que passa pelo ciclo
   assinado. Versão já publicada não muda.

## Validação

- No maintenance, na ficha da cidade: `analyticsStatus.lastRunStatus =
  succeeded`, `stale = false` e `lastPublishedAt` preenchido depois da primeira
  execução agendada.
- No dashboard, como `analyst`: as quatro abas com o carimbo "dados até
  <ontem>"; filtrando por bairro, células de 1 a 4 aparecem como "oculto".
- No console do operador: "Analytics das cidades" com as últimas 12 semanas.
- `analytics_runs` da cidade: um `scheduled` por dia; `error` vazio.

## Operação

- **Atraso (`stale`)**: o último run `succeeded` tem mais de 36 h. Veja
  `lastError` no maintenance e as falhas do Solid Queue do worker da cidade
  (`ConsolidateAnalyticsJob` levanta `Analytics::Run::Failed` quando o run falha).
  Corrigida a causa, a próxima execução refaz os últimos 30 dias sozinha; para
  não esperar, `bin/rails "city:analytics:rebuild[<slug>,<hoje-30>]"`.
- **Publicação falhou** (`lastError` começa com `publish:`): os fatos da cidade
  estão certos; a próxima execução republica a própria janela. Se o bloco era de
  um rebuild antigo, rode o rebuild daquele período de novo.
- **"Outra consolidação em curso"**: o rake encontrou o job (ou outro rebuild)
  rodando na cidade. Espere terminar e rode de novo; não há fila.
- **Bug de consolidação**: corrigido o código, `city:analytics:rebuild[<slug>,<from>,<to>]`
  do período afetado. **Atenção:** o rebuild refaz a partir do cru que existe
  hoje; triagem anonimizada depois da consolidação original sai do número (a
  consolidação agendada nunca mexe fora dos últimos 30 dias — o rebuild é a
  exceção explícita).
- **Retenção**: fatos 5 anos, `analytics_runs` 90 dias (o último `succeeded`
  sempre fica), indicadores da plataforma 5 anos — purgados pelo próprio job.
- **Revogação**: dentro dos últimos 30 dias, a pessoa sai dos números na
  próxima execução (vira triagem abortada por revogação, sem bairro); fora
  deles, o agregado anônimo fica (LGPD art. 12).

## Rollback

- Frontends: voltar a imagem anterior; nada no api depende deles.
- api: a imagem anterior ignora as tabelas novas. O `down` da migração de cidade
  **falha** depois que alguém recebeu `analyst` (memberships não se apagam) —
  rollback do papel é revogar, não migrar. As tabelas novas podem ficar.

## ADRs relacionados

0010, 0014, 0016, 0020, 0022, 0023, 0025.
````

- [ ] **Step 2: Registre o runbook no índice**

Em `docs/.claude/mod14/operacao/README.md`, na tabela "Runbooks", depois da linha de `atendimento.md`:

```markdown
| `analytics.md` | Rollout do módulo 14 (migrações aditivas, ordem api → dashboard → admin → maintenance, rebuild inicial), atraso e falha da consolidação, rebuild, retenção e revogação | 0014, 0020, 0023, 0025 | Vigente |
```

- [ ] **Step 3: Commit (repo docs)**

```bash
/opt/homebrew/bin/git -C docs/.claude/mod14 add operacao/analytics.md operacao/README.md
/opt/homebrew/bin/git -C docs/.claude/mod14 commit -m "docs: add the analytics rollout and operation runbook

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 22: Revisão final do api e prova no navegador

- [ ] **Step 1:** Um subagente revisor lê `origin/main..HEAD` do api (código de produção, migrações, dump, semente) contra a spec inteira, o ADR 0025, os contratos e a seção "Desvios da spec", com atenção a:
  - nenhuma coluna de pessoa nas três tabelas; nenhum `Platform.audit` nem `DomainEvents.publish` novos (`grep -rn -A3 "Platform.audit(\|DomainEvents.publish(" app lib` — as chamadas são multilinha);
  - todo SQL dos consolidadores: só binds (`:lower`, `:upper`, `:at`, `:tz`) e fragmentos constantes; nenhum valor de usuário interpolado;
  - toda contagem que sai de `/admin/api/analytics/*` passa por `Analytics::Suppression` depois de somar; nenhuma taxa calculada com número já suprimido;
  - `Admin::Api::AnalyticsController` sem ação para grant; `Operators::CityAnalyticsController` e `CityType#analytics_indicators` sem `CityConnection`/`inside`;
  - nenhuma atribuição de `Current.city` em `app/` ou `lib/`;
  - N+1 nas leituras (uma consulta por métrica, nomes de unidade/bairro em lote);
  - `config/protocols/schema.json` idêntico ao do `contracts` (`diff contracts/.claude/mod14/protocols/schema.json apps/api/.claude/mod14/config/protocols/schema.json`).
- [ ] **Step 2:** Corrija os achados, rode de novo a suíte completa (worker parado) e escreva o relatório da entrega: commits, contagem e tempo da suíte (s/exemplo), evidência de mutação (Task 19), specs existentes alteradas e por quê (`adr_pointers_spec`, `membership_roles_spec`, `operator_grant_access_spec`, `maintenance_schema_spec`).
- [ ] **Step 3:** Para a prova no navegador (spec §10.4, com o usuário; planos do dashboard, do admin e do maintenance) e para a extração do schema GraphQL pelo plano do maintenance, suba o api do worktree no container, na porta **3031**, sem derrubar o servidor principal:

  ```bash
  docker compose exec -d -w /rails/.claude/mod14 api bin/rails server -b 0.0.0.0 -p 3031 -P tmp/pids/server-mod14.pid
  ```

  O Vite do worktree de cada front aponta o proxy para `http://api:3031` (`VITE_API_PROXY_TARGET`). Login e TOTP são feitos pelo usuário (senhas e TOTP da semente de dev podem ser mostrados no chat se ele pedir). Roteiro: `analise@curitiba.demo` vê as quatro abas com a semente e, filtrando por bairro, "oculto"; o autor marca uma pergunta num rascunho no editor; o operador `dev@local` vê "Analytics das cidades" no console; o mantenedor vê `analyticsStatus` e os indicadores na ficha de Curitiba. Para forçar atraso: `docker compose exec -T -w /rails/.claude/mod14 api bin/rails runner 'CityConnection.with(City.find_by!(slug: "curitiba")) { AnalyticsRun.update_all(finished_at: 2.days.ago) }'` (e `city:analytics:rebuild[curitiba]` para voltar).
- [ ] **Step 4:** **Pare.** Merge, push, board e docs (página de status do módulo) só com autorização explícita do usuário, uma etapa de cada vez. Ordem: `contracts` → api → dashboard → admin → maintenance. Ao voltar o checkout para a main: `DROP DATABASE` dos dois bancos de teste de cidade e `city:test_databases` de novo; derrube o servidor da 3031 (`kill $(cat tmp/pids/server-mod14.pid)` no container).

---

## Self-review (feito ao escrever o plano)

**Cobertura da spec:**

| Spec | Task |
|---|---|
| §3.1 `analytics_daily_facts` (colunas, `NULLS NOT DISTINCT`, índices de leitura, sem FK, contagem crua) | 1 |
| §3.2 `analytics_runs`; purga de 90 dias | 1, 10 |
| §3.3 `city_analytics_indicators` (plataforma, CHECK suprimido ⇔ nulo, purga 5 anos) | 2, 9 |
| §3.4 métricas de demanda / qualidade / calibração / epidemiologia | 4 / 5 / 6 / 8 |
| §4.1 job (janela 30 dias, advisory lock, transação única, runs, publicação depois do commit, purgas, falha preserva fatos) | 10 |
| §4.2 revogação | 4, 6, 8 (desvio 1), 19 |
| §4.3 rebuild e rake | 11 |
| §5 publicação e o conjunto fixo | 9 |
| §6.1 rotas da cidade (papéis, sem grant, parâmetros, limites, envelope, `stale`, supressão sempre) | 12, 13, 14, 15, 16 |
| §6.2 console do operador | 17 |
| §6.3 supressão (cidade e plataforma) | 9, 13, 19 |
| §6.4 GraphQL (`analyticsIndicators`, `analyticsStatus` anulável) | 18 |
| §7 `analytic` no schema (cópia do `contracts`) | 7 |
| §10.1 testes do api; §10.2 suíte de invariantes | todas; 19 |
| §11 semente de dev | 20 |
| §12 rollout (runbook) | 21 |
| contratos §1 `units` (demand, quality) e `triages_total` (demand) | 13, 14 |
| guarda `adr_pointers_spec` 1..25 | 1 |

**Placeholders:** nenhum "TBD"/"implemente depois"; cada passo de código traz o código.

**Tipos e nomes:** `AnalyticsDailyFact::METRICS/WAIT_BUCKETS`, `AnalyticsRun::KINDS/STATUSES` (1); `CityAnalyticsIndicator::INDICATORS/COUNT_INDICATORS` (2, 9, 17, 19); `AnalyticsHistory.triage!/attendance!/request!/appointment!/ended_at` (3, 4–8, 20); helpers `a_triage!`, `an_attendance!`, `analytics_protocol!`, `analytics_definition`, `local_at`, `analytics_staff`, `fact!`, `consolidated_run!`, `scheduled_run!` (3; usados de 4 a 19); `Analytics::Consolidate.call/.fronts` e `Consolidate::Base.call(from:, to:, at:)` (4–8, 10); `Analytics::Suppression.cell/.rate/.sort_value/::SUPPRESSED` (9, 13–17); `Analytics::Publish.call(from:, to:, at:)` (9, 10, 19); `Analytics::Run.call(kind:, from:, to:)`, `.scheduled_window`, `.try_lock`, `::Failed` (10, 11, 19); `Analytics::Rebuild.call(from:, to:)` → `Report(from, to, runs, failed, busy)` (11, 20); `Analytics::Params.new(front, raw, today:)` e `::Invalid#code`, `::DATE` (12, 13, 17); `Analytics::Status.call` → `Snapshot#to_h` (12, 13, 18); `Analytics::BaseQuery.new(params, periods:)` (13–16); `Analytics::CityIndicatorsQuery.call/.weeks/::MAX_WEEKS` (17, 18).

**Review Focus:** as cinco linhas têm teste na task dona (Tasks 4, 5, 6, 8, 10, 11, 13, 14, 19).

**Contratos revisados pelo coordenador (2026-09-30), incorporados:** `analyticsStatus` anulável via `inside` (Task 18, desvio 15); `data.units` em `demand` e `quality` (Tasks 13, 14); `triages_total` em `demand` (Task 13); porta 3031 para a prova e a extração do schema (Ambiente, Task 22).

## Fora deste plano

- Mudança no repo `contracts` (`analytic` no schema, `protocols-v1.2.0`): plano `2026-09-30-module-14-analytics-contracts-repo.md` — executado **antes** da Task 7.
- Telas do dashboard, do admin e do maintenance (planos próprios, contra os mesmos contratos).
- Cards do board (F-14.1 a F-14.9), página de status do módulo 14, e emenda no ADR 0025 se o usuário aprovar os desvios 1 (definição de "triagem revogada") e 2 (um atendimento por triagem).
