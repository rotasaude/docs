# Banco por cidade — Plano 5: Worker por cidade

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:**
- A fila de cada cidade passa a morar no banco dela, e os jobs de ciclo de vida numa fila de plataforma.
- Um processo gerente mantém um supervisor Solid Queue por cidade ativa, mais o da plataforma, com filas enxutas e
  restart com backoff.
- O Solid Cache vai para o banco de plataforma, e o banco compartilhado `rota_saude_<env>` deixa de ser usado.

**Architecture:**
- **Roteamento:** o `SolidQueue::Record` tem como shard padrão o banco de plataforma (`config.solid_queue.connects_to`).
  Cada cidade ganha, em runtime, um pool de fila no banco dela, registrado por `CityConnection` com
  `establish_connection`, e `CityConnection.with` usa `connected_to_many([CityRecord, SolidQueue::Record])`. Assim um job
  enfileirado dentro de uma cidade cai na fila dela, e um job enfileirado fora de cidade cai na de plataforma, que só
  aceita jobs de plataforma.
- **Worker:** `bin/city_workers` roda o `CityWorkers::Manager`, que forka um supervisor por unidade e reconcilia com o
  catálogo a cada poll. No filho de uma cidade, o padrão do `SolidQueue::Record` é trocado para o banco dela no processo
  inteiro, e as tarefas recorrentes passam a ser por cidade.

**Tech Stack:** Rails 8.1.3, Ruby 3.3.6, PostgreSQL (dev 15.13; prod `postgres:16`), Solid Queue 1.4.0, Solid Cache
1.0.10, RSpec, Docker Compose, Kamal 2.

**Spec:** `docs/superpowers/specs/2026-09-12-banco-por-cidade-design.md`:
- §1 ("Topologia");
- §2 (banco da cidade inclui Solid Queue);
- §4 ("Worker por cidade", "Solid Queue no banco de plataforma, para operações de ciclo de vida");
- "Spike 2: worker por cidade";
- "Riscos abertos" 1.

**Planos anteriores:** 1, 2, 3, 3B e 4, todos mergeados em `main` do repo `apps/api` (Plano 4 = merge `d1dc8ee`).

## Decisões que este plano carrega

**Do usuário (2026-09-15, antes da escrita):**

1. **Solid Cache vai para o banco de plataforma.**
   - É um store só; as chaves de protocolo já carregam a cidade.
   - *Custo:* as chaves de `rate_limit` guardam IP fora do banco da cidade.
2. **O banco compartilhado `rota_saude_<env>` é aposentado neste plano.**
   - Saem as configs `queue`, a env `DATABASE_URL`, `ROTA_ADMIN_PASSWORD` e as cargas de schema do `start.sh`.
   - O banco de dev **não é apagado**; só deixa de ser usado.
3. **A guarda de cidade com schema atrasado para jobs entra aqui** (pendência I3 do Plano 4).
   - O gerente não sobe supervisor de cidade atrasada.
   - `CityScopedJob` recusa e reagenda.
   - `EachCityJob` pula essa cidade.
   - A ingestão do WhatsApp segue gravando a mensagem.
4. **Jobs pendentes na fila compartilhada de dev são descartados:** as filas novas começam vazias; nada é migrado nem
   apagado.

**Tomadas na escrita:**

5. **Roteamento da fila:**
   - `config.solid_queue.connects_to = { shards: { default: { writing: :platform } } }` é o único `connects_to` fora do
     corpo de `CityRecord`.
   - O pool de fila de cada cidade é registrado por `establish_connection` junto com o pool de domínio.
   - Validado por sondagem em 2026-09-15:
     - o padrão aponta para `rota_saude_platform_test`;
     - dentro de `SolidQueue::Record.connected_to(shard:)` o `SolidQueue::Job.enqueue` grava no banco da cidade, mesmo
       aninhado em `CityConnection.with`;
     - fora do bloco volta à plataforma.
   - *Custo:* dois pools por cidade por processo (domínio e fila).
6. **No filho de uma cidade, `SolidQueue::Record.establish_connection(<config da cidade>)` troca o padrão para o
   processo inteiro.**
   - Validado na mesma sondagem: outra thread passou a ver o banco da cidade.
   - `CityWorkers::Context.city_slug` marca o processo.
7. **A fila de plataforma só aceita jobs de plataforma** (`PlatformQueue`):
   - `ProvisionCityJob`, `PurgePlatformAccessJob` e entregas do `InvitationMailer`.
   - `SolidQueue::RecurringJob` entra em qualquer fila.
   - Um job de cidade enfileirado fora de uma cidade levanta `PlatformQueue::Misplaced` em vez de gravar dado de cidadão
     no banco de plataforma, e um job de plataforma não entra na fila de uma cidade.
   - A guarda é um `before_enqueue` em `ApplicationJob` e em `CityMailDeliveryJob`, o `delivery_job` do
     `ApplicationMailer`.
   - *Custo:* todo job novo de plataforma precisa entrar na lista.
8. **`EachCityJob`:**
   - dentro do worker de uma cidade, roda só nessa cidade;
   - fora de worker de cidade (console, specs), roda em toda cidade ativa, como hoje;
   - nos dois casos pula cidade com schema atrasado.
   - Os specs multi-cidade existentes seguem válidos.
9. **Recorrência dividida:**
   - `config/recurring.yml` fica com as tarefas de cidade, agendadas pelo scheduler de cada cidade na fila dela (spec §4).
   - `config/recurring_platform.yml` fica com `purge_platform_access` e `clear_solid_queue_finished`, que também continua
     no arquivo de cidade porque cada banco limpa a sua fila.
10. **Filas enxutas.** A cidade (`config/queue.yml`) tem:
    - três workers: `urgent` isolado (ADR-0006), `realtime,default` e `reports,housekeeping`;
    - dispatcher e scheduler.

    A plataforma (`config/queue_platform.yml`) tem um worker `default,housekeeping` e o dispatcher.
    - *Custo:* ~6 processos por cidade (supervisor, dispatcher, scheduler e 3 workers) contra os 4–5 estimados no spike 2.
11. **Gerente (`CityWorkers::Manager`):**
    - lê o catálogo a cada 30 s (`CITY_WORKERS_POLL_SECONDS`): cidades `active`, fora as atrasadas;
    - sobe o que falta e manda TERM ao que saiu;
    - reinicia quem morreu com backoff exponencial de 1 s até 300 s, zerado depois de 600 s de pé;
    - no TERM/INT, repassa TERM aos filhos, espera `SolidQueue.shutdown_timeout + 5 s` e mata o grupo de processo de quem
      sobrar.
    - Cada filho é líder do próprio grupo, para o KILL não deixar workers órfãos.
    - Não usa pidfile (`SolidQueue.supervisor_pidfile` segue nil).
12. **Schema atrasado nos jobs:**
    - `CityScopedJob` levanta `CitySchemaBehind`, e `ApplicationJob` reagenda a cada 5 min por até 12 tentativas.
    - `CityMismatch` levanta quando o job é de outra cidade que não a do worker (defesa em profundidade) e não é
      descartado.
13. **`primary` aposentado.**
    - Aponta para `rota_saude_no_city_selected` com `database_tasks: false`. É verificado em Rails 8.1.3 que
      `configs_for(env_name:)` esconde configs sem database tasks da checagem de migrations pendentes do dev.
    - A config `queue` sai.
    - `cache` aponta para o banco de plataforma com `database_tasks: false`.
    - `db/schema.rb`, `db/queue_schema.rb`, `db/cache_schema.rb` e `bin/jobs` são apagados.
    - O plugin `solid_queue` do Puma sai.
    - Os nomes dos bancos aposentados seguem recusados pelo `city:load_schema`.
14. **Os painéis `/admin/api/overview` e `/admin/api/queues` passam a ler a fila da cidade** pela conexão dela. A dívida
    "500 em teste por falta de tabela de fila" vira request spec.
15. **Upgrade futuro do Solid Queue que adicionar colunas** exige migration de cidade e de plataforma (documentado no
    README).

## Global Constraints

- **Resolver a cidade ANTES de autenticar** (`CityResolution` → `Authentication`).
- **`connects_to` só em dois lugares:** uma vez no corpo de `CityRecord` e uma vez em
  `config.solid_queue.connects_to` (`config/application.rb`). Cidade entra por `CityConnection` com
  `establish_connection`, para domínio e fila.
- **O banco de plataforma nunca guarda dado de cidadão.** A fila de plataforma só recebe `PlatformQueue::JOBS` e
  entregas de `PlatformQueue::MAILERS`. `PlatformEvent` segue recusando chave de payload com `email`, `cpf`, `phone`,
  `wa_id`, `provider_uid`, `body` ou `name` (salvo `phone_number_id` e `city_name`) ou exatamente `from`.
- **A fila de uma cidade só guarda jobs dessa cidade, no banco dela.**
- **`ActiveRecord::Tasks::DatabaseTasks.with_temporary_connection` só em processo de uma thread** (rake, subprocesso).
- **Senha e `database_url` nunca vão para log, mensagem de erro, argv de processo ou payload de evento.** Erros de banco
  passam por `CitySchema.redact`.
- **Migrations de cidade e de plataforma deste plano são aditivas (expand-only).**
- **Nunca conceder CREATE em `public` a `rota_app`.**
- **Ponteiros de ADR em comentário só na faixa ADR-0001..ADR-0015.** Nada de constante top-level em spec (use métodos).
- **Todo comando roda no container**, a partir da raiz do monorepo: `docker compose exec -T api <cmd>`. O rspec roda com
  `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Nunca `bundle` no host.
- **Suíte completa:** timeout de comando ≥ 600000 ms, nunca duas suítes ao mesmo tempo, nunca `git stash`.
  - Uma execução interrompida deixa linhas comitadas por specs não transacionais.
  - Limpe só os prefixos de teste `rota_saude_test_city_prov`, `rota_test_city_prov` e `rota_saude_test_scratch_`, e as
    cidades `prov*` com seus `platform_events`/`city_channels`/`city_grants` em `rota_saude_platform_test`.
- **Sondagens só como spec de rascunho** (apagado depois). `rails runner` só para leitura.
- **Operações destrutivas permitidas neste plano: SÓ estas.**
  - Bancos e roles com os prefixos de teste acima.
  - `city:test_databases` (recarga normal da suíte).
  - Na prova da Task 6: `DROP` de `rota_saude_city_cascavel`/`rota_city_cascavel` pelo offboarding, e remoção do dump da
    prova.
  - Nada em `rota_saude_development`, `rota_saude_test`, `rota_saude_platform_*`, `rota_saude_no_city_selected` ou
    `rota_saude_city_curitiba`/`maringa`, além de migrations aditivas e `db:seed`.
  - O banco compartilhado de dev **não é apagado**.
- **Spec que registra pool de cidade cujo banco apaga chama `CityConnection.forget(city.shard)`**, que agora remove os
  dois pools.
- **Spec que cria banco de verdade usa `self.use_transactional_tests = false`** e limpa o que escreveu na plataforma.
- **Suíte verde ao fim de CADA task, 0 pending.** Nenhum spec enfraquecido.
- **Commits em inglês**, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.

## Estado herdado (repo `apps/api`, `main` em `d1dc8ee`)

| Peça | Onde | Hoje |
|---|---|---|
| Fila | `config/application.rb:38`, `bin/jobs`, `config/queue.yml`, `config/recurring.yml` | adapter `:solid_queue`, `SolidQueue.connects_to` nil → `SolidQueue::Record` usa `ActiveRecord::Base` (`primary` = banco compartilhado); `queue.yml` com 5 workers; 12 tarefas recorrentes num arquivo só |
| Tabelas da fila/cache | `db/queue_schema.rb`, `db/cache_schema.rb` | só no banco compartilhado (carregadas pelo `start.sh`); nenhum banco de cidade ou de plataforma tem `solid_queue_*`/`solid_cache_entries` |
| Configs de banco | `config/database.yml` | `primary`/`queue`/`cache` → `rota_saude_<env>` (dev/test por nome, prod por `DATABASE_URL`); `queue`/`cache` como `rota_admin`; `platform`; `city_unset` → `rota_saude_no_city_selected` (`database_tasks: false`) |
| Cache | `config/cache.yml`, envs | `solid_cache_store` em dev/prod, sem `database:` (usa `ActiveRecord::Base`); `null_store` em test; usado por `rate_limit` e `Protocols.current` (chave com o shard da cidade) |
| Conexão de cidade | `app/models/city_connection.rb` | `ensure_pool` registra só `CityRecord`; `with` usa `CityRecord.connected_to`; `forget` remove só esse pool; `db_config_for` privado |
| Jobs de cidade | `app/jobs/concerns/city_scoped_job.rb`, `each_city_job.rb`, `idempotent_consumer.rb` | `with_city(slug)`: `CityMissing`/`CityNotServable`; `EachCityJob` itera toda cidade `active` |
| Jobs de plataforma | `ProvisionCityJob`, `PurgePlatformAccessJob`, `InvitationMailer#invite` | enfileirados fora de cidade (console, worker) |
| Enfileiramento em cidade | `DomainEvents.publish`, `Whatsapp::Ingest` (dentro de `CityConnection.with`), jobs que encadeiam jobs, `PasswordMailer` | caem hoje no banco compartilhado |
| Painéis | `app/queries/admin/overview_query.rb`, `queues_query.rb` | leem `SolidQueue::*` pela conexão padrão → 500 em teste (`spec/requests/operator_city_session_spec.rb` documenta) |
| Worker dev/prod | `docker-compose.yml` (raiz, fora do git) `worker: ./bin/jobs start`; Kamal `servers.worker.cmd: ./bin/jobs` | um supervisor para tudo |
| Guardas | `spec/architecture/connection_invariants_spec.rb` | recusa `connects_to` fora de corpo de classe abstrata (pegaria `config.solid_queue.connects_to`) |
| `city:load_schema` | `lib/tasks/city.rake` | recusa bancos de `primary`/`queue`/`cache`/`platform`/`city_unset`; `spec/tasks/city_rake_spec.rb` prova recusando `rota_saude_test` e `rota_saude_development` |

Suíte: **651 examples, 0 failures**. Dev: `curitiba` e `maringa` ativas (schema `20260915000002`), `londrina` archived.

**Sondagem da escrita (2026-09-15, spec de rascunho já apagado):**
- **Enqueue por cidade:**
  - Com `SolidQueue::Record.connects_to(shards: { default: { writing: :platform } })`, o padrão resolveu para
    `rota_saude_platform_test`.
  - Num banco scratch com o `db/queue_schema.rb` carregado, registrei o pool com
    `connection_handler.establish_connection(cfg, owner_name: SolidQueue::Record, role: :writing, shard: :probecity)`.
  - Dentro de `CityConnection.with(TEST_CITY_A)`, um `SolidQueue::Record.connected_to(role: :writing,
    shard: :probecity) { SolidQueue::Job.enqueue(...) }` gravou 1 job no scratch, enquanto `CityRecord` seguia em
    `rota_saude_test_city_a`. Fora do bloco, a fila voltou a `rota_saude_platform_test`.
- **Troca de padrão no processo:** outra thread leu o shard. Depois de `SolidQueue::Record.establish_connection(cfg)`,
  outra thread sem `connected_to` passou a ver o scratch.
- **API:** `ActiveRecord::Base.connected_to_many` existe.
- **Migrations pendentes:** `ActiveRecord::Migration.check_pending_migrations` (middleware `CheckPending` do dev) usa
  `configs_for(env_name:)`, que esconde configs com `database_tasks: false`.

## File Structure

| Arquivo | Task | Responsabilidade |
|---|---|---|
| `db/solid_queue_tables.rb` | 1 | definição única das tabelas do Solid Queue 1.4 |
| `db/city_migrate/20260916000001_create_solid_queue_tables.rb`, `db/city_schema.rb` | 1 | fila no banco da cidade |
| `db/platform_migrate/20260916000002_create_platform_solid_queue_tables.rb`, `db/platform_migrate/20260916000003_create_solid_cache_entries.rb`, `db/platform_schema.rb` | 1 | fila de plataforma e cache no banco de plataforma |
| `spec/architecture/queue_tables_spec.rb` | 1 | tabelas presentes nos bancos certos |
| `config/application.rb` | 2 | `config.solid_queue.connects_to` |
| `app/models/city_connection.rb` | 2 | pool de fila por cidade; `with` nos dois; `database_config` público |
| `app/services/city_workers/context.rb` | 2 | cidade do processo worker |
| `app/services/platform_queue.rb`, `app/jobs/city_mail_delivery_job.rb`, `app/jobs/application_job.rb`, `app/mailers/application_mailer.rb` | 2 | guarda da fila de plataforma |
| `spec/support/platform_queue.rb`, `spec/rails_helper.rb` | 2 | `on_platform_queue { }` para specs |
| `spec/models/city_connection_queue_spec.rb`, `spec/services/platform_queue_spec.rb`, `spec/architecture/connection_invariants_spec.rb`, `spec/requests/operator_city_session_spec.rb` | 2 | roteamento, guarda, painéis |
| `spec/commands/provision_city_spec.rb`, `spec/jobs/provision_city_job_spec.rb`, `spec/requests/operators/cities_spec.rb` | 2 | enfileiramento de plataforma fora da cidade do harness |
| `app/jobs/concerns/each_city_job.rb`, `app/jobs/concerns/city_scoped_job.rb`, `app/jobs/application_job.rb` | 3 | cidade do worker e guarda de schema |
| `config/recurring.yml`, `config/recurring_platform.yml`, `config/queue.yml`, `config/queue_platform.yml` | 3 | recorrência dividida, filas enxutas |
| `spec/jobs/concerns/each_city_job_spec.rb`, `spec/jobs/concerns/city_scoped_job_spec.rb`, `spec/config/solid_queue_configuration_spec.rb` | 3 | comportamento e configuração |
| `app/services/city_workers/unit.rb`, `backoff.rb`, `clock.rb`, `spawner.rb`, `child.rb`, `manager.rb`, `bin/city_workers` | 4 | gerente de supervisores |
| `spec/services/city_workers/backoff_spec.rb`, `manager_spec.rb`, `child_spec.rb` | 4 | laço sem fork; filho com fork real |
| `config/database.yml`, `config/cache.yml`, `config/puma.rb`, `lib/tasks/city.rake`, `bin/docker-entrypoint` | 5 | aposentar o compartilhado; cache na plataforma |
| `db/schema.rb`, `db/queue_schema.rb`, `db/cache_schema.rb`, `bin/jobs` (apagar) | 5 | fonte antiga |
| `spec/architecture/shared_database_retired_spec.rb`, `spec/tasks/city_rake_spec.rb` | 5 | guarda da aposentadoria |
| `deploy/production/deploy.yml`, `deploy/development/deploy.yml`, `deploy/production/secrets`, `deploy/development/secrets`, `deploy/SECRETS.md`, `README.md`, `../../start.sh`, `../../docker-compose.yml` (fora do git) | 5 | deploy, docs, dev |
| `README.md` (seção do worker) | 6 | runbook do worker por cidade |

---

### Task 1: Tabelas da fila nos bancos de cidade e de plataforma, e do cache na plataforma

**Files:**
- Create: `db/solid_queue_tables.rb`
- Create: `db/city_migrate/20260916000001_create_solid_queue_tables.rb`
- Modify: `db/city_schema.rb`
- Create: `db/platform_migrate/20260916000002_create_platform_solid_queue_tables.rb`
- Create: `db/platform_migrate/20260916000003_create_solid_cache_entries.rb`
- Modify: `db/platform_schema.rb` (regenerado por dump)
- Create: `spec/architecture/queue_tables_spec.rb`

**Interfaces:**
- Consumes:
  - `CitySchema`, `city:migrate:all` e `city:test_databases` (Plano 4).
  - O spec de paridade migrations × dump (`spec/services/city_schema_spec.rb`).
- Produces:
  - Tabelas `solid_queue_*` em todo banco de cidade e no banco de plataforma, e `solid_cache_entries` no banco de
    plataforma.
  - `SolidQueueTables.create(migration)`.
  - `CitySchema.expected_version == 20260916000001`.

- [ ] **Step 1: Spec (falha)**

Criar `spec/architecture/queue_tables_spec.rb`:

```ruby
require "rails_helper"

# Plano 5 (spec banco-por-cidade §2/§4): a fila de cada cidade mora no banco dela;
# a fila de plataforma e o Solid Cache, no banco de plataforma.
RSpec.describe "Solid Queue and Solid Cache tables" do
  def solid_queue_tables
    %w[solid_queue_blocked_executions solid_queue_claimed_executions solid_queue_failed_executions
       solid_queue_jobs solid_queue_pauses solid_queue_processes solid_queue_ready_executions
       solid_queue_recurring_executions solid_queue_recurring_tasks solid_queue_scheduled_executions
       solid_queue_semaphores]
  end

  it "has the queue and the cache in the platform database" do
    expect(PlatformRecord.connection.tables).to include(*solid_queue_tables, "solid_cache_entries")
  end

  it "has the queue, and no cache, in a city database" do
    tables = CityRecord.connection.tables

    expect(tables).to include(*solid_queue_tables)
    expect(tables).not_to include("solid_cache_entries")
  end
end
```

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/architecture/queue_tables_spec.rb`
Expected: FAIL nos dois exemplos (as tabelas não existem).

- [ ] **Step 2: Definição única das tabelas do Solid Queue**

Criar `db/solid_queue_tables.rb`:

```ruby
# Tabelas do Solid Queue 1.4.0, transcritas de db/queue_schema.rb (Plano 5). Uma
# definição só, usada pela migration de cidade e pela de plataforma: a fila de
# cada cidade mora no banco dela; a de plataforma, no banco de plataforma (spec
# banco-por-cidade §1/§4). Um upgrade do Solid Queue que mude tabelas exige nova
# migration nos dois diretórios (db/city_migrate e db/platform_migrate).
module SolidQueueTables
  module_function

  def create(migration)
    migration.create_table :solid_queue_jobs do |t|
      t.string :queue_name, null: false
      t.string :class_name, null: false
      t.text :arguments
      t.integer :priority, default: 0, null: false
      t.string :active_job_id
      t.datetime :scheduled_at
      t.datetime :finished_at
      t.string :concurrency_key
      t.timestamps
      t.index :active_job_id, name: "index_solid_queue_jobs_on_active_job_id"
      t.index :class_name, name: "index_solid_queue_jobs_on_class_name"
      t.index :finished_at, name: "index_solid_queue_jobs_on_finished_at"
      t.index %i[queue_name finished_at], name: "index_solid_queue_jobs_for_filtering"
      t.index %i[scheduled_at finished_at], name: "index_solid_queue_jobs_for_alerting"
    end

    migration.create_table :solid_queue_blocked_executions do |t|
      t.bigint :job_id, null: false
      t.string :queue_name, null: false
      t.integer :priority, default: 0, null: false
      t.string :concurrency_key, null: false
      t.datetime :expires_at, null: false
      t.datetime :created_at, null: false
      t.index %i[concurrency_key priority job_id], name: "index_solid_queue_blocked_executions_for_release"
      t.index %i[expires_at concurrency_key], name: "index_solid_queue_blocked_executions_for_maintenance"
      t.index :job_id, name: "index_solid_queue_blocked_executions_on_job_id", unique: true
    end

    migration.create_table :solid_queue_claimed_executions do |t|
      t.bigint :job_id, null: false
      t.bigint :process_id
      t.datetime :created_at, null: false
      t.index :job_id, name: "index_solid_queue_claimed_executions_on_job_id", unique: true
      t.index %i[process_id job_id], name: "index_solid_queue_claimed_executions_on_process_id_and_job_id"
    end

    migration.create_table :solid_queue_failed_executions do |t|
      t.bigint :job_id, null: false
      t.text :error
      t.datetime :created_at, null: false
      t.index :job_id, name: "index_solid_queue_failed_executions_on_job_id", unique: true
    end

    migration.create_table :solid_queue_pauses do |t|
      t.string :queue_name, null: false
      t.datetime :created_at, null: false
      t.index :queue_name, name: "index_solid_queue_pauses_on_queue_name", unique: true
    end

    migration.create_table :solid_queue_processes do |t|
      t.string :kind, null: false
      t.datetime :last_heartbeat_at, null: false
      t.bigint :supervisor_id
      t.integer :pid, null: false
      t.string :hostname
      t.text :metadata
      t.datetime :created_at, null: false
      t.string :name, null: false
      t.index :last_heartbeat_at, name: "index_solid_queue_processes_on_last_heartbeat_at"
      t.index %i[name supervisor_id], name: "index_solid_queue_processes_on_name_and_supervisor_id", unique: true
      t.index :supervisor_id, name: "index_solid_queue_processes_on_supervisor_id"
    end

    migration.create_table :solid_queue_ready_executions do |t|
      t.bigint :job_id, null: false
      t.string :queue_name, null: false
      t.integer :priority, default: 0, null: false
      t.datetime :created_at, null: false
      t.index :job_id, name: "index_solid_queue_ready_executions_on_job_id", unique: true
      t.index %i[priority job_id], name: "index_solid_queue_poll_all"
      t.index %i[queue_name priority job_id], name: "index_solid_queue_poll_by_queue"
    end

    migration.create_table :solid_queue_recurring_executions do |t|
      t.bigint :job_id, null: false
      t.string :task_key, null: false
      t.datetime :run_at, null: false
      t.datetime :created_at, null: false
      t.index :job_id, name: "index_solid_queue_recurring_executions_on_job_id", unique: true
      t.index %i[task_key run_at], name: "index_solid_queue_recurring_executions_on_task_key_and_run_at", unique: true
    end

    migration.create_table :solid_queue_recurring_tasks do |t|
      t.string :key, null: false
      t.string :schedule, null: false
      t.string :command, limit: 2048
      t.string :class_name
      t.text :arguments
      t.string :queue_name
      t.integer :priority, default: 0
      t.boolean :static, default: true, null: false
      t.text :description
      t.timestamps
      t.index :key, name: "index_solid_queue_recurring_tasks_on_key", unique: true
      t.index :static, name: "index_solid_queue_recurring_tasks_on_static"
    end

    migration.create_table :solid_queue_scheduled_executions do |t|
      t.bigint :job_id, null: false
      t.string :queue_name, null: false
      t.integer :priority, default: 0, null: false
      t.datetime :scheduled_at, null: false
      t.datetime :created_at, null: false
      t.index :job_id, name: "index_solid_queue_scheduled_executions_on_job_id", unique: true
      t.index %i[scheduled_at priority job_id], name: "index_solid_queue_dispatch_all"
    end

    migration.create_table :solid_queue_semaphores do |t|
      t.string :key, null: false
      t.integer :value, default: 1, null: false
      t.datetime :expires_at, null: false
      t.timestamps
      t.index :expires_at, name: "index_solid_queue_semaphores_on_expires_at"
      t.index %i[key value], name: "index_solid_queue_semaphores_on_key_and_value"
      t.index :key, name: "index_solid_queue_semaphores_on_key", unique: true
    end

    %i[solid_queue_blocked_executions solid_queue_claimed_executions solid_queue_failed_executions
       solid_queue_ready_executions solid_queue_recurring_executions solid_queue_scheduled_executions].each do |table|
      migration.add_foreign_key table, :solid_queue_jobs, column: :job_id, on_delete: :cascade
    end
  end
end
```

- [ ] **Step 3: Migrations**

Criar `db/city_migrate/20260916000001_create_solid_queue_tables.rb`:

```ruby
require Rails.root.join("db/solid_queue_tables").to_s

# A fila da cidade mora no banco dela (spec banco-por-cidade §1: o payload de
# ProcessInboundMessageJob e SendWhatsappJob carrega telefone e mensagem).
# Plano 5. Aditiva.
class CreateSolidQueueTables < ActiveRecord::Migration[8.1]
  def change
    SolidQueueTables.create(self)
  end
end
```

Criar `db/platform_migrate/20260916000002_create_platform_solid_queue_tables.rb`:

```ruby
require Rails.root.join("db/solid_queue_tables").to_s

# Fila de PLATAFORMA (spec banco-por-cidade §4, Plano 5): jobs de ciclo de vida —
# provisionamento, e-mail do convite, purga de acesso. Só PlatformQueue::JOBS
# entram aqui; nunca dado de cidadão. Aditiva.
class CreatePlatformSolidQueueTables < ActiveRecord::Migration[8.1]
  def change
    SolidQueueTables.create(self)
  end
end
```

Criar `db/platform_migrate/20260916000003_create_solid_cache_entries.rb`:

```ruby
# Solid Cache no banco de PLATAFORMA (Plano 5, decisão do usuário): rate_limit dos
# logins e cache de protocolos (a chave carrega o shard da cidade). Transcrito de
# db/cache_schema.rb. Aditiva.
class CreateSolidCacheEntries < ActiveRecord::Migration[8.1]
  def change
    create_table :solid_cache_entries do |t|
      t.binary :key, null: false
      t.binary :value, null: false
      t.datetime :created_at, null: false
      t.bigint :key_hash, null: false
      t.integer :byte_size, null: false
      t.index :byte_size, name: "index_solid_cache_entries_on_byte_size"
      t.index %i[key_hash byte_size], name: "index_solid_cache_entries_on_key_hash_and_byte_size"
      t.index :key_hash, name: "index_solid_cache_entries_on_key_hash", unique: true
    end
  end
end
```

- [ ] **Step 4: Dump do schema de cidade**

Em `db/city_schema.rb`, trocar `ActiveRecord::Schema[8.1].define(version: 2026_09_15_000002) do` por
`ActiveRecord::Schema[8.1].define(version: 2026_09_16_000001) do`.

Imediatamente antes da linha `  create_table "triages", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|`,
inserir os blocos abaixo, cada um seguido de uma linha em branco. São os blocos de `db/queue_schema.rb`, na ordem
alfabética que o dump usa.

```ruby
  create_table "solid_queue_blocked_executions", force: :cascade do |t|
    t.string "concurrency_key", null: false
    t.datetime "created_at", null: false
    t.datetime "expires_at", null: false
    t.bigint "job_id", null: false
    t.integer "priority", default: 0, null: false
    t.string "queue_name", null: false
    t.index ["concurrency_key", "priority", "job_id"], name: "index_solid_queue_blocked_executions_for_release"
    t.index ["expires_at", "concurrency_key"], name: "index_solid_queue_blocked_executions_for_maintenance"
    t.index ["job_id"], name: "index_solid_queue_blocked_executions_on_job_id", unique: true
  end

  create_table "solid_queue_claimed_executions", force: :cascade do |t|
    t.datetime "created_at", null: false
    t.bigint "job_id", null: false
    t.bigint "process_id"
    t.index ["job_id"], name: "index_solid_queue_claimed_executions_on_job_id", unique: true
    t.index ["process_id", "job_id"], name: "index_solid_queue_claimed_executions_on_process_id_and_job_id"
  end

  create_table "solid_queue_failed_executions", force: :cascade do |t|
    t.datetime "created_at", null: false
    t.text "error"
    t.bigint "job_id", null: false
    t.index ["job_id"], name: "index_solid_queue_failed_executions_on_job_id", unique: true
  end

  create_table "solid_queue_jobs", force: :cascade do |t|
    t.string "active_job_id"
    t.text "arguments"
    t.string "class_name", null: false
    t.string "concurrency_key"
    t.datetime "created_at", null: false
    t.datetime "finished_at"
    t.integer "priority", default: 0, null: false
    t.string "queue_name", null: false
    t.datetime "scheduled_at"
    t.datetime "updated_at", null: false
    t.index ["active_job_id"], name: "index_solid_queue_jobs_on_active_job_id"
    t.index ["class_name"], name: "index_solid_queue_jobs_on_class_name"
    t.index ["finished_at"], name: "index_solid_queue_jobs_on_finished_at"
    t.index ["queue_name", "finished_at"], name: "index_solid_queue_jobs_for_filtering"
    t.index ["scheduled_at", "finished_at"], name: "index_solid_queue_jobs_for_alerting"
  end

  create_table "solid_queue_pauses", force: :cascade do |t|
    t.datetime "created_at", null: false
    t.string "queue_name", null: false
    t.index ["queue_name"], name: "index_solid_queue_pauses_on_queue_name", unique: true
  end

  create_table "solid_queue_processes", force: :cascade do |t|
    t.datetime "created_at", null: false
    t.string "hostname"
    t.string "kind", null: false
    t.datetime "last_heartbeat_at", null: false
    t.text "metadata"
    t.string "name", null: false
    t.integer "pid", null: false
    t.bigint "supervisor_id"
    t.index ["last_heartbeat_at"], name: "index_solid_queue_processes_on_last_heartbeat_at"
    t.index ["name", "supervisor_id"], name: "index_solid_queue_processes_on_name_and_supervisor_id", unique: true
    t.index ["supervisor_id"], name: "index_solid_queue_processes_on_supervisor_id"
  end

  create_table "solid_queue_ready_executions", force: :cascade do |t|
    t.datetime "created_at", null: false
    t.bigint "job_id", null: false
    t.integer "priority", default: 0, null: false
    t.string "queue_name", null: false
    t.index ["job_id"], name: "index_solid_queue_ready_executions_on_job_id", unique: true
    t.index ["priority", "job_id"], name: "index_solid_queue_poll_all"
    t.index ["queue_name", "priority", "job_id"], name: "index_solid_queue_poll_by_queue"
  end

  create_table "solid_queue_recurring_executions", force: :cascade do |t|
    t.datetime "created_at", null: false
    t.bigint "job_id", null: false
    t.datetime "run_at", null: false
    t.string "task_key", null: false
    t.index ["job_id"], name: "index_solid_queue_recurring_executions_on_job_id", unique: true
    t.index ["task_key", "run_at"], name: "index_solid_queue_recurring_executions_on_task_key_and_run_at", unique: true
  end

  create_table "solid_queue_recurring_tasks", force: :cascade do |t|
    t.text "arguments"
    t.string "class_name"
    t.string "command", limit: 2048
    t.datetime "created_at", null: false
    t.text "description"
    t.string "key", null: false
    t.integer "priority", default: 0
    t.string "queue_name"
    t.string "schedule", null: false
    t.boolean "static", default: true, null: false
    t.datetime "updated_at", null: false
    t.index ["key"], name: "index_solid_queue_recurring_tasks_on_key", unique: true
    t.index ["static"], name: "index_solid_queue_recurring_tasks_on_static"
  end

  create_table "solid_queue_scheduled_executions", force: :cascade do |t|
    t.datetime "created_at", null: false
    t.bigint "job_id", null: false
    t.integer "priority", default: 0, null: false
    t.string "queue_name", null: false
    t.datetime "scheduled_at", null: false
    t.index ["job_id"], name: "index_solid_queue_scheduled_executions_on_job_id", unique: true
    t.index ["scheduled_at", "priority", "job_id"], name: "index_solid_queue_dispatch_all"
  end

  create_table "solid_queue_semaphores", force: :cascade do |t|
    t.datetime "created_at", null: false
    t.datetime "expires_at", null: false
    t.string "key", null: false
    t.datetime "updated_at", null: false
    t.integer "value", default: 1, null: false
    t.index ["expires_at"], name: "index_solid_queue_semaphores_on_expires_at"
    t.index ["key", "value"], name: "index_solid_queue_semaphores_on_key_and_value"
    t.index ["key"], name: "index_solid_queue_semaphores_on_key", unique: true
  end
```

Imediatamente antes do `end` final do arquivo, depois das linhas `add_foreign_key` já existentes, inserir:

```ruby
  add_foreign_key "solid_queue_blocked_executions", "solid_queue_jobs", column: "job_id", on_delete: :cascade
  add_foreign_key "solid_queue_claimed_executions", "solid_queue_jobs", column: "job_id", on_delete: :cascade
  add_foreign_key "solid_queue_failed_executions", "solid_queue_jobs", column: "job_id", on_delete: :cascade
  add_foreign_key "solid_queue_ready_executions", "solid_queue_jobs", column: "job_id", on_delete: :cascade
  add_foreign_key "solid_queue_recurring_executions", "solid_queue_jobs", column: "job_id", on_delete: :cascade
  add_foreign_key "solid_queue_scheduled_executions", "solid_queue_jobs", column: "job_id", on_delete: :cascade
```

- [ ] **Step 5: Aplicar nos bancos de teste e rodar os specs**

Run: `docker compose exec -T -e RAILS_ENV=test api ./bin/rails city:test_databases`
Run: `docker compose exec -T -e RAILS_ENV=test api ./bin/rails db:migrate:platform`
Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/architecture/queue_tables_spec.rb spec/services/city_schema_spec.rb`
Expected: PASS. O exemplo de paridade de `city_schema_spec` prova que a migration de cidade e o dump editado à mão
produzem o mesmo schema; se falhar, corrija o lado errado sem afrouxar o fingerprint.

- [ ] **Step 6: Aplicar em dev e regenerar o dump de plataforma**

Run: `docker compose exec -T api ./bin/rails db:migrate:platform`
Run: `docker compose exec -T api ./bin/rails db:schema:dump:platform`
Expected: `db/platform_schema.rb` ganha as 11 tabelas `solid_queue_*`, `solid_cache_entries` e as 6 chaves estrangeiras.
Nada mais muda nele.

Run: `docker compose exec -T api ./bin/rails city:migrate:all`
Expected: `curitiba → 20260916000001`, `maringa → 20260916000001`, saída 0.

Run: `docker compose restart api` (a versão esperada é memoizada por processo).
Run: `curl -s -o /dev/null -w "%{http_code}\n" -H "Host: curitiba.localhost" http://localhost:3030/admin/api/overview`
Expected: `401`.

- [ ] **Step 7: Suíte inteira**

Run (timeout ≥ 600000 ms): `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

- [ ] **Step 8: Commit**

```bash
git add db/solid_queue_tables.rb db/city_migrate/20260916000001_create_solid_queue_tables.rb db/city_schema.rb \
  db/platform_migrate/20260916000002_create_platform_solid_queue_tables.rb \
  db/platform_migrate/20260916000003_create_solid_cache_entries.rb db/platform_schema.rb \
  spec/architecture/queue_tables_spec.rb
git commit -m "Create Solid Queue tables in city and platform databases and Solid Cache on the platform

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: Fila roteada por cidade e guarda da fila de plataforma

**Files:**
- Modify: `config/application.rb`
- Modify: `app/models/city_connection.rb`
- Create: `app/services/city_workers/context.rb`
- Create: `app/services/platform_queue.rb`
- Create: `app/jobs/city_mail_delivery_job.rb`
- Modify: `app/jobs/application_job.rb`
- Modify: `app/mailers/application_mailer.rb`
- Create: `spec/support/platform_queue.rb`
- Modify: `spec/rails_helper.rb`
- Create: `spec/models/city_connection_queue_spec.rb`
- Create: `spec/services/platform_queue_spec.rb`
- Modify: `spec/architecture/connection_invariants_spec.rb`
- Modify: `spec/requests/operator_city_session_spec.rb`
- Modify: `spec/commands/provision_city_spec.rb`, `spec/jobs/provision_city_job_spec.rb`, `spec/requests/operators/cities_spec.rb`

**Interfaces:**
- Consumes: as tabelas `solid_queue_*` (Task 1).
- Produces:
  - `SolidQueue.connects_to == { shards: { default: { writing: :platform } } }`.
  - `CityConnection.with(city)` roteia `CityRecord` e `SolidQueue::Record`.
  - `CityConnection.ensure_pool(city)` registra os dois pools, e `CityConnection.forget(shard)` remove os dois.
  - `CityConnection.database_config(city) -> ActiveRecord::DatabaseConfigurations::DatabaseConfig` (público).
  - `CityWorkers::Context.city_slug` (accessor; `nil` fora de worker de cidade).
  - `PlatformQueue::JOBS`, `PlatformQueue::MAILERS`, `PlatformQueue::ANYWHERE`, `PlatformQueue::Misplaced`.
  - `PlatformQueue.platform_target? -> Boolean`, `PlatformQueue.check!(job)`.
  - `CityMailDeliveryJob`.
  - Helper de spec `on_platform_queue { }`.

- [ ] **Step 1: Specs (falham)**

Criar `spec/support/platform_queue.rb`:

```ruby
# Enfileirar um job de PLATAFORMA dentro de um exemplo (Plano 5). O harness roda
# todo exemplo dentro de CityConnection.with(TEST_CITY_A), então a fila de destino
# num spec é a da cidade — e PlatformQueue recusa job de plataforma ali. Este
# helper leva só o SolidQueue::Record de volta ao shard padrão (a fila de
# plataforma), como acontece no console e no worker de plataforma.
module PlatformQueueHelpers
  def on_platform_queue(&block)
    SolidQueue::Record.connected_to(role: :writing, shard: SolidQueue::Record.default_shard, &block)
  end
end

RSpec.configure do |config|
  config.include PlatformQueueHelpers
end
```

Em `spec/rails_helper.rb`, depois de `require_relative "support/provisioned_cities"`, acrescentar:

```ruby
require_relative "support/platform_queue"
```

Criar `spec/models/city_connection_queue_spec.rb`:

```ruby
require "rails_helper"

# Plano 5 (spec banco-por-cidade §1/§4): a fila de cada cidade mora no banco dela;
# fora de cidade, a fila é a de plataforma. Usa o adapter real do Solid Queue —
# o de teste não grava em tabela nenhuma.
RSpec.describe "CityConnection queue routing" do
  around do |example|
    adapter_was = ActiveJob::Base.queue_adapter
    ActiveJob::Base.queue_adapter = :solid_queue
    example.run
  ensure
    ActiveJob::Base.queue_adapter = adapter_was
  end

  let(:probe_job) { stub_const("QueueRoutingProbeJob", Class.new(ApplicationJob) { def perform; end }) }

  def probe_jobs
    SolidQueue::Job.where(class_name: "QueueRoutingProbeJob")
  end

  it "enqueues a job inside a city onto that city's own database" do
    probe_job.perform_later

    expect(SolidQueue::Job.connection_db_config.database).to eq("rota_saude_test_city_a")
    expect(probe_jobs.count).to eq(1)
    expect(on_platform_queue { probe_jobs.count }).to eq(0)
  end

  it "keeps two cities' queues apart" do
    CityConnection.with(TEST_CITY_B) { probe_job.perform_later }

    expect(probe_jobs.count).to eq(0)
    expect(CityConnection.with(TEST_CITY_B) { [ SolidQueue::Job.connection_db_config.database, probe_jobs.count ] })
      .to eq([ "rota_saude_test_city_b", 1 ])
  end

  it "enqueues a platform job outside any city onto the platform database" do
    on_platform_queue { PurgePlatformAccessJob.perform_later }

    expect(on_platform_queue { [ SolidQueue::Job.connection_db_config.database,
                                 SolidQueue::Job.where(class_name: "PurgePlatformAccessJob").count ] })
      .to eq([ "rota_saude_platform_test", 1 ])
  end

  it "registers and forgets both pools of a city" do
    city = create(:city, slug: "fila#{SecureRandom.hex(3)}", database_url: city_database_url("rota_saude_test_city_b"))
    handler = ActiveRecord::Base.connection_handler

    CityConnection.ensure_pool(city)
    expect(handler.retrieve_connection_pool("SolidQueue::Record", role: :writing, shard: city.shard)).to be_present

    CityConnection.forget(city.shard)
    expect(handler.retrieve_connection_pool("CityRecord", role: :writing, shard: city.shard)).to be_nil
    expect(handler.retrieve_connection_pool("SolidQueue::Record", role: :writing, shard: city.shard)).to be_nil
  end
end
```

Criar `spec/services/platform_queue_spec.rb`:

```ruby
require "rails_helper"

# Plano 5: a fila de plataforma mora no banco de plataforma, que nunca guarda dado
# de cidadão. Só jobs de ciclo de vida entram nela; job de cidade fora de cidade
# levanta em vez de cair ali.
RSpec.describe PlatformQueue do
  include ActiveJob::TestHelper

  let(:city_job) { stub_const("PlatformQueueCityProbeJob", Class.new(ApplicationJob) { def perform; end }) }
  let(:reset_url) { "http://testcitya.localhost:5175/dashboard/?reset=tok" }
  let(:accept_url) { "http://testcitya.localhost:5175/dashboard/?invite=tok" }

  after { CityWorkers::Context.city_slug = nil }

  it "accepts a city job inside a city and refuses it on the platform queue" do
    expect { city_job.perform_later }.to have_enqueued_job(city_job)
    expect { on_platform_queue { city_job.perform_later } }
      .to raise_error(PlatformQueue::Misplaced, /PlatformQueueCityProbeJob/)
  end

  it "accepts a platform job on the platform queue and refuses it inside a city" do
    expect { on_platform_queue { PurgePlatformAccessJob.perform_later } }.to have_enqueued_job(PurgePlatformAccessJob)
    expect { PurgePlatformAccessJob.perform_later }.to raise_error(PlatformQueue::Misplaced, /PurgePlatformAccessJob/)
  end

  it "accepts only the invitation mailer on the platform queue" do
    expect { on_platform_queue { InvitationMailer.invite(email_address: "a@cidade.gov.br", accept_url: accept_url).deliver_later } }
      .to have_enqueued_job(CityMailDeliveryJob)
    expect { on_platform_queue { PasswordMailer.reset(email_address: "a@cidade.gov.br", reset_url: reset_url).deliver_later } }
      .to raise_error(PlatformQueue::Misplaced, /PasswordMailer/)
    expect { PasswordMailer.reset(email_address: "a@cidade.gov.br", reset_url: reset_url).deliver_later }
      .to have_enqueued_job(CityMailDeliveryJob)
  end

  it "treats the default shard as the city's queue inside a city worker process" do
    CityWorkers::Context.city_slug = "curitiba"

    expect(described_class.platform_target?).to be(false)
    expect { on_platform_queue { city_job.perform_later } }.to have_enqueued_job(city_job)
  end

  it "lets Solid Queue's recurring command job into any queue" do
    job = SolidQueue::RecurringJob.new("SolidQueue::Job.clear_finished_in_batches")

    expect { described_class.check!(job) }.not_to raise_error
    expect { on_platform_queue { described_class.check!(job) } }.not_to raise_error
  end
end
```

Em `spec/architecture/connection_invariants_spec.rb`, trocar o exemplo inteiro:

```ruby
  it "only calls connects_to from an abstract class body" do
    offenders = Dir.chdir(Rails.root) do
      source_files.flat_map do |path|
        lines = File.readlines(path, encoding: "UTF-8")
        real_connects_to = code_lines(lines).select { |l, _| l.include?("connects_to") }
        next [] if real_connects_to.empty?
        next [] if code_lines(lines).any? { |l, _| l.match?(/self\.abstract_class\s*=\s*true/) }

        real_connects_to.map { |_, i| "#{path}:#{i + 1}" }
      end
    end

    expect(offenders).to eq([]),
      "connects_to fora de classe abstrata (use CityConnection.ensure_pool):\n#{offenders.join("\n")}"
  end
```

por:

```ruby
  it "only calls connects_to from an abstract class body, or configures Solid Queue's once" do
    offenders = Dir.chdir(Rails.root) do
      source_files.flat_map do |path|
        lines = File.readlines(path, encoding: "UTF-8")
        real_connects_to = code_lines(lines).select { |l, _| l.include?("connects_to") }
        if path == "config/application.rb"
          real_connects_to = real_connects_to.reject { |l, _| l.match?(/\bconfig\.solid_queue\.connects_to\s*=/) }
        end
        next [] if real_connects_to.empty?
        next [] if code_lines(lines).any? { |l, _| l.match?(/self\.abstract_class\s*=\s*true/) }

        real_connects_to.map { |_, i| "#{path}:#{i + 1}" }
      end
    end

    expect(offenders).to eq([]),
      "connects_to fora de classe abstrata (use CityConnection.ensure_pool):\n#{offenders.join("\n")}"
  end

  # Plano 5: a fila de plataforma é o shard padrão do SolidQueue::Record; cada
  # cidade entra por CityConnection.ensure_pool (establish_connection), nunca por
  # outro connects_to.
  it "points Solid Queue at the platform database by default, configured only in config/application.rb" do
    expect(SolidQueue.connects_to).to eq(shards: { default: { writing: :platform } })

    elsewhere = Dir.chdir(Rails.root) do
      (source_files - [ "config/application.rb" ]).select do |path|
        code_lines(File.readlines(path, encoding: "UTF-8")).any? { |l, _| l.include?("solid_queue.connects_to") }
      end
    end
    expect(elsewhere).to eq([])
  end
```

Em `spec/requests/operator_city_session_spec.rb`, trocar:

```ruby
    # /admin/api/overview would also belong on this list, but it (and
    # /admin/api/queues) call SolidQueue::FailedExecution/ClaimedExecution
    # directly. SolidQueue::Record has no connects_to in this app, so it uses
    # whatever connection ApplicationRecord/primary resolves to — the PRIMARY
    # test database (rota_saude_test), not the city shard. rota_saude_test has
    # no solid_queue_* tables (those only exist in the `queue` role, which
    # config/database.yml doesn't even define for test), so the query 500s.
    # This reproduces for ANY signed-in actor (probed with a plain
    # municipal_admin session, not just an operator grant), so it predates
    # this task and is out of scope for it (Task 1 touches only the files
    # listed in its brief) — /admin/api/reports and /admin/api/triages already
    # cover the read-without-membership invariant this example exists for.
    %w[/admin/api/reports /admin/api/triages].each do |path|
```

por:

```ruby
    # /admin/api/overview and /admin/api/queues read SolidQueue::* tables. Since
    # Plan 5 the city's queue lives in the city's database and CityConnection.with
    # routes SolidQueue::Record there too, so these panels read the city's own
    # queue.
    %w[/admin/api/overview /admin/api/queues /admin/api/reports /admin/api/triages].each do |path|
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_connection_queue_spec.rb spec/services/platform_queue_spec.rb spec/architecture/connection_invariants_spec.rb spec/requests/operator_city_session_spec.rb`
Expected: FAIL. Os motivos esperados são:
- `uninitialized constant PlatformQueue` / `CityMailDeliveryJob` / `CityWorkers::Context`;
- fila no banco errado (`rota_saude_test`);
- `SolidQueue.connects_to` nil;
- 500 nos painéis.

- [ ] **Step 3: Solid Queue com a plataforma por padrão**

Em `config/application.rb`, trocar:

```ruby
    # ADR-0001: Solid Queue é o adapter padrão.
    config.active_job.queue_adapter = :solid_queue
```

por:

```ruby
    # ADR-0001: Solid Queue é o adapter padrão.
    config.active_job.queue_adapter = :solid_queue

    # Plano 5 (spec banco-por-cidade §1/§4): a fila padrão do Solid Queue é a de
    # PLATAFORMA (jobs de ciclo de vida). A fila de cada cidade mora no banco dela e
    # entra em runtime por CityConnection (establish_connection, nunca outro
    # connects_to); o worker de uma cidade troca o padrão para o banco dela
    # (CityWorkers::Child). O banco de plataforma nunca recebe job de cidade:
    # PlatformQueue recusa.
    config.solid_queue.connects_to = { shards: { default: { writing: :platform } } }
```

- [ ] **Step 4: `CityConnection` com o pool de fila**

Em `app/models/city_connection.rb`:

Trocar:

```ruby
    def with(city, &block)
      ensure_pool(city)
      CityRecord.connected_to(shard: city.shard, role: :writing, &block)
    end
```

por:

```ruby
    # Domínio E fila da cidade (Plano 5): um job enfileirado aqui dentro cai na fila
    # do banco da cidade, não na de plataforma.
    def with(city, &block)
      ensure_pool(city)
      ActiveRecord::Base.connected_to_many([ CityRecord, SolidQueue::Record ], role: :writing, shard: city.shard, &block)
    end
```

Trocar:

```ruby
        ActiveRecord::Base.connection_handler.establish_connection(
          db_config_for(city),
          owner_name: CityRecord,
          role: :writing,
          shard: city.shard
        )
```

por:

```ruby
        config = database_config(city)
        # A fila da cidade mora no banco dela (Plano 5). O pool de fila é registrado
        # antes do de domínio: registered? olha o de domínio, então um existe só se
        # o outro já existe.
        ActiveRecord::Base.connection_handler.establish_connection(
          config, owner_name: SolidQueue::Record, role: :writing, shard: city.shard
        )
        ActiveRecord::Base.connection_handler.establish_connection(
          config, owner_name: CityRecord, role: :writing, shard: city.shard
        )
```

Trocar:

```ruby
    def forget(shard)
      ActiveRecord::Base.connection_handler
        .remove_connection_pool(CityRecord.name, role: :writing, shard: shard)
    end

    private

    def db_config_for(city)
```

por:

```ruby
    def forget(shard)
      handler = ActiveRecord::Base.connection_handler
      handler.remove_connection_pool(CityRecord.name, role: :writing, shard: shard)
      handler.remove_connection_pool(SolidQueue::Record.name, role: :writing, shard: shard)
    end

    # Config resolvida do banco da cidade. Pública para o worker da cidade
    # (CityWorkers::Child), que liga o Solid Queue do processo inteiro a ela.
    def database_config(city)
```

No comentário de `ensure_pool` do mesmo arquivo, trocar `#                            não por db_config_for — por isso o rescue cobre`
por `#                            não por database_config — por isso o rescue cobre`. `CitySchema.db_config_for` é outro
método e não muda.

E, no mesmo arquivo, mover a linha `    private` para logo antes de `    def pool_size`. O resultado final do trecho é:

```ruby
      resolved
    end

    private

    def pool_size
```

- [ ] **Step 5: Contexto do worker, guarda e entrega de e-mail**

Criar `app/services/city_workers/context.rb`:

```ruby
module CityWorkers
  # A cidade deste processo quando ele é o worker de uma cidade (CityWorkers::Child,
  # Plano 5). nil no web, no console, nos specs e no worker de plataforma.
  module Context
    mattr_accessor :city_slug
  end
end
```

Criar `app/services/platform_queue.rb`:

```ruby
# Onde um job pode ser enfileirado (spec banco-por-cidade §1/§4, Plano 5).
#
# A fila de cada cidade mora no banco dela; a fila de PLATAFORMA, no banco de
# plataforma, e só recebe jobs de ciclo de vida — o banco de plataforma nunca
# guarda dado de cidadão, e o payload de um job de cidade (telefone, mensagem) é
# dado de cidadão. Um job de cidade enfileirado fora da conexão de uma cidade
# cairia na fila de plataforma: aqui ele levanta em vez de cair. E um job de
# plataforma não entra na fila de uma cidade.
module PlatformQueue
  # Jobs e mailers de ciclo de vida. Job novo de plataforma entra aqui.
  JOBS = %w[ProvisionCityJob PurgePlatformAccessJob].freeze
  MAILERS = %w[InvitationMailer].freeze
  # Job do próprio Solid Queue para tarefas recorrentes do tipo command: cada
  # banco limpa a sua fila.
  ANYWHERE = %w[SolidQueue::RecurringJob].freeze

  class Misplaced < StandardError; end

  module_function

  # A fila de destino é a de plataforma quando o SolidQueue::Record está no shard
  # padrão E este processo não é o worker de uma cidade — no worker de cidade o
  # padrão foi trocado para o banco dela (CityWorkers::Child).
  def platform_target?
    SolidQueue::Record.current_shard == SolidQueue::Record.default_shard && CityWorkers::Context.city_slug.nil?
  end

  def check!(job)
    return if ANYWHERE.include?(job.class.name)

    if platform_target?
      raise Misplaced, "#{label(job)} é de cidade e não entra na fila de plataforma" unless platform?(job)
    elsif platform?(job)
      raise Misplaced, "#{label(job)} é de plataforma e não entra na fila de uma cidade"
    end
  end

  def platform?(job)
    return MAILERS.include?(label(job)) if job.is_a?(ActionMailer::MailDeliveryJob)

    JOBS.include?(job.class.name)
  end

  # Para entrega de e-mail, o nome do mailer (primeiro argumento do job).
  def label(job)
    job.is_a?(ActionMailer::MailDeliveryJob) ? job.arguments.first.to_s : job.class.name
  end
end
```

Criar `app/jobs/city_mail_delivery_job.rb`:

```ruby
# Entrega de e-mail com a mesma guarda de fila dos jobs do app (Plano 5): um
# deliver_later de mailer de cidade fora da conexão de uma cidade levanta
# PlatformQueue::Misplaced em vez de gravar o e-mail na fila de plataforma.
class CityMailDeliveryJob < ActionMailer::MailDeliveryJob
  before_enqueue { |job| PlatformQueue.check!(job) }
end
```

Em `app/jobs/application_job.rb`, logo depois de `  self.enqueue_after_transaction_commit = true`, acrescentar:

```ruby

  # Plano 5: job de cidade só na fila da cidade; job de plataforma só na fila de
  # plataforma (PlatformQueue).
  before_enqueue { |job| PlatformQueue.check!(job) }
```

Substituir o conteúdo de `app/mailers/application_mailer.rb` por:

```ruby
class ApplicationMailer < ActionMailer::Base
  default from: ENV.fetch("MAIL_FROM", "rota-saude@example.com")
  layout "mailer"

  # deliver_later passa pela guarda de fila (PlatformQueue, Plano 5).
  self.delivery_job = CityMailDeliveryJob
end
```

- [ ] **Step 6: Specs que enfileiram job de plataforma saem da fila da cidade do harness**

Nos três arquivos abaixo, todo trecho que enfileira job de plataforma passa a rodar dentro de
`on_platform_queue { ... }`. Esses trechos são `ProvisionCity.call`, `POST /cities` e `ProvisionCityJob.perform_now`,
este último porque entrega o `InvitationMailer`. Nenhuma asserção muda:

- `spec/commands/provision_city_spec.rb`: todo `described_class.call(...)` vira `on_platform_queue { described_class.call(...) }`.
  Exemplo: `expect { result = described_class.call(**args) }` vira
  `expect { result = on_platform_queue { described_class.call(**args) } }`.
- `spec/requests/operators/cities_spec.rb`: todo `post "/cities", params: ...` vira
  `on_platform_queue { post "/cities", params: ... }`, inclusive dentro de `expect { }`.
- `spec/jobs/provision_city_job_spec.rb`: todo `described_class.perform_now(...)` vira
  `on_platform_queue { described_class.perform_now(...) }`.

- [ ] **Step 7: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_connection_queue_spec.rb spec/services/platform_queue_spec.rb spec/architecture/connection_invariants_spec.rb spec/requests/operator_city_session_spec.rb spec/commands/provision_city_spec.rb spec/jobs/provision_city_job_spec.rb spec/requests/operators/cities_spec.rb`
Expected: PASS.

- [ ] **Step 8: Suíte inteira**

Run (timeout ≥ 600000 ms): `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

`spec/requests/passwords_spec.rb` usa `have_enqueued_mail(PasswordMailer, :reset)`. Se esse matcher do rspec-rails não
reconhecer o `CityMailDeliveryJob` como job de e-mail (falha dizendo que nenhum e-mail foi enfileirado, embora o job
esteja na fila), troque só nesse arquivo por `have_enqueued_job(CityMailDeliveryJob).with("PasswordMailer", "reset",
"deliver_now", args: [], kwargs: hash_including(email_address: <o mesmo e-mail do exemplo>))`, e o `not_to` equivalente.
A asserção continua sendo que o e-mail de reset foi, ou não, enfileirado.

Um `PlatformQueue::Misplaced` num arquivo **fora** dos três do Step 6 não é artefato do harness. É um enfileiramento de
job de cidade fora de cidade, ou de job de plataforma dentro de cidade, e pode ser bug real. Não o embrulhe em
`on_platform_queue`: pare e reporte o arquivo, o job e o caminho de código.

- [ ] **Step 9: Prova em dev**

Run: `docker compose restart api`
Run: `curl -s -o /dev/null -w "%{http_code}\n" -H "Host: curitiba.localhost" http://localhost:3030/admin/api/overview`
Expected: `401`.

Até a Task 5, o worker de dev (`bin/jobs`) passa a olhar só a fila de plataforma, e jobs de cidade ficam parados na fila
de cada cidade. O esperado é isso: o worker por cidade chega na Task 4 e entra em uso na Task 5.

- [ ] **Step 10: Commit**

```bash
git add config/application.rb app/models/city_connection.rb app/services/city_workers/context.rb \
  app/services/platform_queue.rb app/jobs/city_mail_delivery_job.rb app/jobs/application_job.rb \
  app/mailers/application_mailer.rb spec/support/platform_queue.rb spec/rails_helper.rb \
  spec/models/city_connection_queue_spec.rb spec/services/platform_queue_spec.rb \
  spec/architecture/connection_invariants_spec.rb spec/requests/operator_city_session_spec.rb \
  spec/commands/provision_city_spec.rb spec/jobs/provision_city_job_spec.rb spec/requests/operators/cities_spec.rb
git commit -m "Route each city's jobs to its own database and guard the platform queue

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: Cidade do worker, guarda de schema nos jobs, recorrência dividida e filas enxutas

**Files:**
- Modify: `app/jobs/concerns/each_city_job.rb`
- Modify: `app/jobs/concerns/city_scoped_job.rb`
- Modify: `app/jobs/application_job.rb`
- Modify: `config/recurring.yml`
- Create: `config/recurring_platform.yml`
- Modify: `config/queue.yml`
- Create: `config/queue_platform.yml`
- Create: `spec/jobs/concerns/each_city_job_spec.rb`
- Modify: `spec/jobs/concerns/city_scoped_job_spec.rb`
- Create: `spec/config/solid_queue_configuration_spec.rb`

**Interfaces:**
- Consumes: `CityWorkers::Context.city_slug`, `PlatformQueue::JOBS` (Task 2), `CitySchema.behind?` (Plano 4).
- Produces:
  - `CityScopedJob::CitySchemaBehind` e `CityScopedJob::CityMismatch`.
  - `ApplicationJob` com `retry_on CityScopedJob::CitySchemaBehind, wait: 5.minutes, attempts: 12`.
  - `EachCityJob` visita só a cidade do worker (quando houver) e pula cidade atrasada.
  - Arquivos `config/queue.yml`, `config/queue_platform.yml`, `config/recurring.yml` e `config/recurring_platform.yml`.

- [ ] **Step 1: Specs (falham)**

Criar `spec/jobs/concerns/each_city_job_spec.rb`:

```ruby
require "rails_helper"

# Plano 5: no worker de uma cidade, EachCityJob roda só nela (a tarefa recorrente
# mora na fila dela); fora de worker de cidade, em toda cidade ativa; nos dois
# casos pula cidade com schema atrasado (spec §4).
RSpec.describe EachCityJob do
  let(:job_class) do
    stub_const("EachCityProbeJob", Class.new(ApplicationJob) do
      prepend EachCityJob
      class << self; attr_accessor :visited; end

      def perform
        self.class.visited << [ Current.city.slug, CityRecord.connection_db_config.database ]
      end
    end)
  end

  let!(:city_a) { create(:city, slug: "cadaa#{SecureRandom.hex(3)}", database_url: city_database_url("rota_saude_test_city_a")) }
  let!(:city_b) { create(:city, slug: "cadab#{SecureRandom.hex(3)}", database_url: city_database_url("rota_saude_test_city_b")) }

  before { job_class.visited = [] }
  after { CityWorkers::Context.city_slug = nil }

  it "visits every active city outside a city worker" do
    job_class.new.perform

    expect(job_class.visited).to contain_exactly([ city_a.slug, "rota_saude_test_city_a" ],
                                                 [ city_b.slug, "rota_saude_test_city_b" ])
  end

  it "visits only the worker's own city inside a city worker" do
    CityWorkers::Context.city_slug = city_b.slug

    job_class.new.perform

    expect(job_class.visited).to eq([ [ city_b.slug, "rota_saude_test_city_b" ] ])
  end

  it "skips a city whose schema is behind" do
    city_b.update!(schema_version: nil)

    job_class.new.perform

    expect(job_class.visited).to eq([ [ city_a.slug, "rota_saude_test_city_a" ] ])
  end
end
```

Em `spec/jobs/concerns/city_scoped_job_spec.rb`, acrescentar `  include ActiveJob::TestHelper` logo depois de
`RSpec.describe CityScopedJob do` e, antes do `end` final:

```ruby
  describe "schema atrasado e cidade do worker (Plano 5)" do
    after { CityWorkers::Context.city_slug = nil }

    it "raises CitySchemaBehind for a city whose schema is behind, without running the block" do
      city = create(:city, slug: "atrasadajob", status: "active", schema_version: nil,
                           database_url: city_database_url("rota_saude_test_city_b"))

      expect { job_class.new.perform(city.slug) }.to raise_error(CityScopedJob::CitySchemaBehind)
      expect(job_class.ran).to be false
    end

    it "reschedules the job instead of failing it when the city's schema is behind" do
      city = create(:city, slug: "atrasadaretry", status: "active", schema_version: nil,
                           database_url: city_database_url("rota_saude_test_city_b"))

      expect { job_class.perform_now(city.slug) }.to have_enqueued_job(job_class).with(city.slug)
      expect(job_class.ran).to be false
    end

    it "raises CityMismatch for a job of another city than this worker's, without running the block" do
      city = create(:city, slug: "outracidade", status: "active", database_url: city_database_url("rota_saude_test_city_b"))
      CityWorkers::Context.city_slug = "curitiba"

      expect { job_class.new.perform(city.slug) }.to raise_error(CityScopedJob::CityMismatch)
      expect(job_class.ran).to be false
    end

    it "runs a job of the worker's own city" do
      city = create(:city, slug: "propriacidade", status: "active", database_url: city_database_url("rota_saude_test_city_b"))
      CityWorkers::Context.city_slug = city.slug

      job_class.new.perform(city.slug)

      expect(job_class.ran).to be true
    end
  end
```

Criar `spec/config/solid_queue_configuration_spec.rb`:

```ruby
require "rails_helper"

# Plano 5 (spec banco-por-cidade §4): cada cidade roda três workers — urgent
# isolado (ADR-0006) — e agenda só tarefas de cidade; a plataforma roda um worker
# e agenda só jobs de plataforma.
RSpec.describe "Solid Queue configuration per city and platform" do
  def config(path, env)
    ActiveSupport::ConfigurationFile.parse(Rails.root.join(path)).fetch(env, {})
  end

  def task_classes(path, env)
    config(path, env).values.filter_map { |task| task["class"] }
  end

  %w[development production].each do |env|
    context env do
      it "gives each city three workers, with urgent alone" do
        queues = config("config/queue.yml", env).fetch("workers").map { |worker| Array(worker["queues"]) }

        expect(queues).to eq([ %w[urgent], %w[realtime default], %w[reports housekeeping] ])
      end

      it "gives the platform workers covering the queues of every platform job and of e-mail delivery" do
        queues = config("config/queue_platform.yml", env).fetch("workers").flat_map { |worker| Array(worker["queues"]) }

        expect(queues).to include(ProvisionCityJob.queue_name, PurgePlatformAccessJob.queue_name, "default")
      end

      it "schedules only platform jobs on the platform and no platform job in a city" do
        expect(task_classes("config/recurring_platform.yml", env)).to all(satisfy { |name| PlatformQueue::JOBS.include?(name) })
        expect(task_classes("config/recurring.yml", env) & PlatformQueue::JOBS).to eq([])
        expect(task_classes("config/recurring.yml", env).map(&:safe_constantize)).to all(be_present)
      end

      it "clears finished jobs in every queue database" do
        expect(config("config/recurring.yml", env)).to have_key("clear_solid_queue_finished")
        expect(config("config/recurring_platform.yml", env)).to have_key("clear_solid_queue_finished")
      end
    end
  end
end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/jobs/concerns/each_city_job_spec.rb spec/jobs/concerns/city_scoped_job_spec.rb spec/config/solid_queue_configuration_spec.rb`
Expected: FAIL. Os motivos esperados são:
- `EachCityJob` visita as duas cidades no worker de uma e não pula a cidade atrasada;
- `uninitialized constant CityScopedJob::CitySchemaBehind` / `CityMismatch`;
- `config/queue_platform.yml` e `config/recurring_platform.yml` inexistentes;
- `config/queue.yml` com cinco workers.

- [ ] **Step 3: `CityScopedJob` e `ApplicationJob`**

Em `app/jobs/concerns/city_scoped_job.rb`, trocar:

```ruby
  class CityMissing < StandardError; end
  class CityNotServable < StandardError; end

  private

  def with_city(slug)
    raise CityMissing, "#{self.class.name}: slug nulo" if slug.blank?

    city = City.find_by(slug: slug)
    raise CityMissing, "#{self.class.name}: cidade #{slug} não existe" if city.nil?
    unless city.servable?
      raise CityNotServable, "#{self.class.name}: cidade #{slug} não está servível (status=#{city.status})"
    end
```

por:

```ruby
  class CityMissing < StandardError; end
  class CityNotServable < StandardError; end
  # Cidade com schema atrasado (spec §4, Plano 5): o job espera o city:migrate:all
  # em vez de rodar código novo sobre schema velho. ApplicationJob reagenda.
  class CitySchemaBehind < StandardError; end
  # Job de outra cidade que não a do worker que o executa: a fila de uma cidade só
  # tem jobs dela (Plano 5). Defesa em profundidade do roteamento — não é
  # descartado, fica visível como falha.
  class CityMismatch < StandardError; end

  private

  def with_city(slug)
    raise CityMissing, "#{self.class.name}: slug nulo" if slug.blank?

    worker_slug = CityWorkers::Context.city_slug
    if worker_slug && worker_slug != slug
      raise CityMismatch, "#{self.class.name}: job da cidade #{slug} no worker da cidade #{worker_slug}"
    end

    city = City.find_by(slug: slug)
    raise CityMissing, "#{self.class.name}: cidade #{slug} não existe" if city.nil?
    unless city.servable?
      raise CityNotServable, "#{self.class.name}: cidade #{slug} não está servível (status=#{city.status})"
    end
    raise CitySchemaBehind, "#{self.class.name}: cidade #{slug} com schema atrasado" if CitySchema.behind?(city)
```

Também trocar, no comentário do topo do mesmo arquivo:

```ruby
# O job carrega o SLUG, não o objeto: o payload viaja pela fila, que ainda vive
# no banco compartilhado até o Plano 5.
```

por:

```ruby
# O job carrega o SLUG, não o objeto: o payload viaja pela fila, que mora no
# banco da própria cidade (Plano 5).
```

Em `app/jobs/application_job.rb`, logo depois de `  retry_on ActiveRecord::Deadlocked, attempts: 3, wait: :polynomially_longer`,
acrescentar:

```ruby
  # Plano 5: cidade com schema atrasado — espera o city:migrate:all (até 1 hora).
  retry_on CityScopedJob::CitySchemaBehind, wait: 5.minutes, attempts: 12
```

- [ ] **Step 4: `EachCityJob`**

Substituir o conteúdo de `app/jobs/concerns/each_city_job.rb` por:

```ruby
# Para jobs que rodam em cada cidade (tarefas recorrentes de housekeeping).
#
# Plano 5 (spec banco-por-cidade §4): as tarefas recorrentes moram na fila de
# cada cidade e são agendadas pelo scheduler dela. Por isso:
#   - no worker de uma cidade (CityWorkers::Context), o corpo roda só nessa cidade;
#   - fora de worker de cidade (console, specs), roda em toda cidade ativa;
#   - nos dois casos, cidade com schema atrasado fica de fora — código novo não
#     roda sobre schema velho.
#
# Isolamento de falha por cidade: uma cidade que levanta não impede as demais de
# rodar. Ao final, se alguma cidade falhou, levanta um erro agregado com os slugs
# e mensagens.
#
# Usar SEMPRE via `prepend EachCityJob` (NÃO include): com include, o perform do
# subclass aparece antes na cadeia de ancestrais e o wrap do módulo nunca dispara.
module EachCityJob
  class AggregatedFailure < StandardError
    def initialize(failures)
      @failures = failures
      super(failures.map { |slug, error| "#{slug}: #{error.class}: #{error.message}" }.join("; "))
    end

    attr_reader :failures
  end

  def perform(*args, **kwargs)
    failures = {}

    each_city_cities.each do |city|
      begin
        Current.city = city
        CityConnection.with(city) { super(*args, **kwargs) }
      rescue => e
        Rails.logger.error("[#{self.class.name}] city=#{city.slug} failed: #{e.class}: #{e.message}")
        failures[city.slug] = e
      end
    end

    raise AggregatedFailure, failures if failures.any?
  end

  private

  def each_city_cities
    scope = City.where(status: "active").order(:slug)
    scope = scope.where(slug: CityWorkers::Context.city_slug) if CityWorkers::Context.city_slug

    scope.to_a.reject do |city|
      next false unless CitySchema.behind?(city)

      Rails.logger.warn("[#{self.class.name}] city=#{city.slug} com schema atrasado: pulada")
      true
    end
  end
end
```

- [ ] **Step 5: Recorrência e filas**

Em `config/recurring.yml`:

Trocar a primeira linha `# Tarefas recorrentes Solid Queue. Ver ADR-0010 e ADR-0014.` por:

```yaml
# Tarefas recorrentes de CIDADE (Solid Queue). Ver ADR-0010 e ADR-0014.
# Plano 5: o scheduler de cada cidade agenda estas tarefas na fila do banco dela,
# e cada job roda só naquela cidade (EachCityJob no worker de cidade). Tarefas de
# plataforma ficam em config/recurring_platform.yml.
```

E remover o bloco inteiro, com a linha em branco que o segue:

```yaml
  purge_platform_access:
    class: PurgePlatformAccessJob
    queue: housekeeping
    schedule: "every day at 3:30am America/Sao_Paulo"

```

Criar `config/recurring_platform.yml`:

```yaml
# Tarefas recorrentes da PLATAFORMA (Plano 5): agendadas pelo supervisor de
# plataforma na fila do banco de plataforma. Só jobs de PlatformQueue::JOBS.

default: &default
  purge_platform_access:
    class: PurgePlatformAccessJob
    queue: housekeeping
    schedule: "every day at 3:30am America/Sao_Paulo"

  clear_solid_queue_finished:
    command: "SolidQueue::Job.clear_finished_in_batches(sleep_between_batches: 0.3)"
    schedule: every hour at minute 12

development:
  <<: *default

production:
  <<: *default
```

Substituir o conteúdo de `config/queue.yml` por:

```yaml
# Solid Queue de CADA CIDADE (Plano 5, spec banco-por-cidade §4): o supervisor da
# cidade roda estes processos contra o banco dela. Três workers — urgent isolado
# (alertas à secretaria, ADR-0006), realtime+default (webhook, envio, avisos,
# e-mail), reports+housekeeping — mais o dispatcher; o scheduler vem de
# config/recurring.yml. O pool do banco precisa de pelo menos (maior número de
# threads + 2) conexões: RAILS_MAX_THREADS do worker.

default: &default
  dispatchers:
    - polling_interval: 1
      batch_size: 500
  workers:
    - queues: urgent
      threads: 3
      processes: 1
      polling_interval: 0.1
    - queues: [ realtime, default ]
      threads: 5
      processes: 1
      polling_interval: 0.1
    - queues: [ reports, housekeeping ]
      threads: 2
      processes: 1
      polling_interval: 1

development:
  <<: *default

test:
  <<: *default

production:
  <<: *default
  workers:
    - queues: urgent
      threads: 5
      processes: 1
      polling_interval: 0.1
    - queues: [ realtime, default ]
      threads: 10
      processes: 1
      polling_interval: 0.1
    - queues: [ reports, housekeeping ]
      threads: 3
      processes: 1
      polling_interval: 1
```

Criar `config/queue_platform.yml`:

```yaml
# Solid Queue da PLATAFORMA (Plano 5): jobs de ciclo de vida — provisionamento,
# e-mail do convite (fila default), purga de acesso (housekeeping). Um worker
# basta.

default: &default
  dispatchers:
    - polling_interval: 1
      batch_size: 100
  workers:
    - queues: [ default, housekeeping ]
      threads: 2
      processes: 1
      polling_interval: 1

development:
  <<: *default

test:
  <<: *default

production:
  <<: *default
```

- [ ] **Step 6: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/jobs/concerns spec/config/solid_queue_configuration_spec.rb spec/jobs`
Expected: PASS. Isso inclui os specs multi-cidade existentes de `purge_domain_events`, `sweep_abandoned`, `reencryption`,
`rebuild_dashboard_metrics`, `reconcile_consents` e `purge_operator_city_sessions`.

- [ ] **Step 7: Suíte inteira**

Run (timeout ≥ 600000 ms): `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

- [ ] **Step 8: Commit**

```bash
git add app/jobs/concerns/each_city_job.rb app/jobs/concerns/city_scoped_job.rb app/jobs/application_job.rb \
  config/recurring.yml config/recurring_platform.yml config/queue.yml config/queue_platform.yml \
  spec/jobs/concerns/each_city_job_spec.rb spec/jobs/concerns/city_scoped_job_spec.rb \
  spec/config/solid_queue_configuration_spec.rb
git commit -m "Scope recurring jobs to the worker's city, hold jobs for lagging cities and slim the queues

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: Gerente de supervisores por cidade (`bin/city_workers`)

**Files:**
- Create: `app/services/city_workers/unit.rb`
- Create: `app/services/city_workers/backoff.rb`
- Create: `app/services/city_workers/clock.rb`
- Create: `app/services/city_workers/spawner.rb`
- Create: `app/services/city_workers/child.rb`
- Create: `app/services/city_workers/manager.rb`
- Create: `bin/city_workers`
- Create: `spec/services/city_workers/backoff_spec.rb`
- Create: `spec/services/city_workers/manager_spec.rb`
- Create: `spec/services/city_workers/child_spec.rb`

**Interfaces:**
- Consumes:
  - `CityWorkers::Context` e `CityConnection.database_config(city)` (Task 2).
  - `config/queue.yml`, `config/queue_platform.yml`, `config/recurring.yml` e `config/recurring_platform.yml` (Task 3).
  - `CitySchema.behind?`.
  - `SolidQueue::Supervisor.start(mode:, config_file:, recurring_schedule_file:)`.
- Produces:
  - `CityWorkers::Unit.platform`, `CityWorkers::Unit.city(slug)`, `#key`, `#config_file`, `#recurring_schedule_file`.
  - `CityWorkers::Backoff#delay_for(failures) -> Float`, `#failures_after_exit(previous, ran_for:) -> Integer`.
  - `CityWorkers::Clock#now`, `#sleep(seconds)`.
  - `CityWorkers::Spawner#spawn(unit) -> pid`, `#terminate(pid)`, `#kill(pid)`, `#reap -> [[pid, status]]`.
  - `CityWorkers::Child.prepare(unit) -> Hash` e `.run(unit)`.
  - `CityWorkers::Manager.new(spawner:, clock:, catalog:, backoff:, poll_interval:, logger:)`, com `#tick`,
    `#shutdown(timeout:)`, `#run(stop_signal:)`, `#running` e `.active_city_slugs`.
  - Executável `bin/city_workers`.

- [ ] **Step 1: Specs (falham)**

Criar `spec/services/city_workers/backoff_spec.rb`:

```ruby
require "rails_helper"

# Spike 2: um filho cujo banco sumiu morre em 0,4 s; sem espera, reiniciaria em
# loop apertado.
RSpec.describe CityWorkers::Backoff do
  subject(:backoff) { described_class.new }

  it "doubles the wait after each consecutive failure, up to the cap" do
    expect((1..10).map { |failures| backoff.delay_for(failures) })
      .to eq([ 1.0, 2.0, 4.0, 8.0, 16.0, 32.0, 64.0, 128.0, 256.0, 300.0 ])
    expect(backoff.delay_for(0)).to eq(0.0)
  end

  it "counts a failure after a short run and starts over after a stable run" do
    expect(backoff.failures_after_exit(3, ran_for: 5.0)).to eq(4)
    expect(backoff.failures_after_exit(3, ran_for: 600.0)).to eq(1)
  end
end
```

Criar `spec/services/city_workers/manager_spec.rb`:

```ruby
require "rails_helper"

# Plano 5 (spec banco-por-cidade §4, riscos abertos 1): o laço de reconciliação do
# gerente, sem fork — processo, relógio e catálogo são dublês.
RSpec.describe CityWorkers::Manager do
  let(:clock) do
    Class.new do
      attr_reader :now

      def initialize = @now = 0.0
      def sleep(seconds) = @now += seconds
      def advance(seconds) = @now += seconds
    end.new
  end

  let(:spawner) do
    Class.new do
      attr_reader :spawned, :terminated, :killed

      def initialize
        @spawned, @terminated, @killed, @exits, @next_pid = [], [], [], [], 100
      end

      def spawn(unit) = (@next_pid += 1).tap { |pid| @spawned << [ unit.key, pid ] }
      def terminate(pid) = @terminated << pid
      def kill(pid) = @killed << pid
      def reap = @exits.shift(@exits.size)
      def exit!(pid) = @exits << [ pid, :exited ]
      def pid_of(key) = @spawned.reverse.find { |spawned_key, _| spawned_key == key }&.last
      def spawn_count(key) = @spawned.count { |spawned_key, _| spawned_key == key }
    end.new
  end

  let(:slugs) { %w[curitiba maringa] }
  let(:catalog) { -> { slugs } }
  let(:manager) do
    described_class.new(spawner: spawner, clock: clock, catalog: catalog, poll_interval: 30.0, logger: Logger.new(nil))
  end

  it "starts the platform supervisor and one per catalog city on the first tick" do
    manager.tick

    expect(spawner.spawned.map(&:first)).to eq(%w[platform city:curitiba city:maringa])
    expect(manager.running.keys).to eq(%w[platform city:curitiba city:maringa])
  end

  it "restarts a crashed city with exponential backoff, without touching the others" do
    manager.tick
    spawner.exit!(spawner.pid_of("city:curitiba"))

    manager.tick
    expect(spawner.spawn_count("city:curitiba")).to eq(1)
    clock.advance(0.5)
    manager.tick
    expect(spawner.spawn_count("city:curitiba")).to eq(1)
    clock.advance(0.6)
    manager.tick
    expect(spawner.spawn_count("city:curitiba")).to eq(2)

    spawner.exit!(spawner.pid_of("city:curitiba"))
    manager.tick
    clock.advance(1.5)
    manager.tick
    expect(spawner.spawn_count("city:curitiba")).to eq(2)
    clock.advance(0.6)
    manager.tick
    expect(spawner.spawn_count("city:curitiba")).to eq(3)

    expect(spawner.spawn_count("city:maringa")).to eq(1)
    expect(spawner.spawn_count("platform")).to eq(1)
    expect(spawner.terminated).to be_empty
  end

  it "restarts right after a supervisor that crashes following a stable run" do
    manager.tick
    clock.advance(700.0)
    spawner.exit!(spawner.pid_of("city:maringa"))

    manager.tick
    clock.advance(1.1)
    manager.tick

    expect(spawner.spawn_count("city:maringa")).to eq(2)
  end

  it "stops a city that left the catalog at the next poll and does not restart it" do
    manager.tick
    curitiba = spawner.pid_of("city:curitiba")
    slugs.replace(%w[maringa])

    clock.advance(10.0)
    manager.tick
    expect(spawner.terminated).to be_empty

    clock.advance(20.0)
    manager.tick
    expect(spawner.terminated).to eq([ curitiba ])

    spawner.exit!(curitiba)
    clock.advance(400.0)
    manager.tick
    expect(spawner.spawn_count("city:curitiba")).to eq(1)
    expect(manager.running.keys).to eq(%w[platform city:maringa])
  end

  it "starts a new catalog city at the next poll without restarting the others" do
    manager.tick
    slugs.replace(%w[curitiba maringa cascavel])

    clock.advance(30.0)
    manager.tick

    expect(spawner.spawn_count("city:cascavel")).to eq(1)
    expect(spawner.spawned.size).to eq(4)
  end

  it "keeps what is running when the catalog cannot be read" do
    manager.tick
    allow(catalog).to receive(:call).and_raise(ActiveRecord::ConnectionNotEstablished)

    clock.advance(30.0)
    manager.tick

    expect(spawner.terminated).to be_empty
    expect(manager.running.keys).to eq(%w[platform city:curitiba city:maringa])
  end

  it "starts the platform supervisor even when the very first catalog read fails" do
    allow(catalog).to receive(:call).and_raise(ActiveRecord::ConnectionNotEstablished)

    manager.tick

    expect(spawner.spawned.map(&:first)).to eq(%w[platform])
  end

  it "terminates every supervisor on shutdown and kills those still alive after the timeout" do
    manager.tick
    platform = spawner.pid_of("platform")
    spawner.exit!(platform)

    manager.shutdown(timeout: 5.0)

    expect(spawner.terminated).to match_array(manager_pids = spawner.spawned.map(&:last))
    expect(spawner.killed).to match_array(manager_pids - [ platform ])
  end

  describe ".active_city_slugs" do
    it "lists active cities whose schema is current, in slug order" do
      create(:city, slug: "zzativa", status: "active")
      create(:city, slug: "aaativa", status: "active")
      create(:city, slug: "suspensa", status: "suspended")
      create(:city, slug: "atrasada", status: "active", schema_version: nil)

      expect(described_class.active_city_slugs).to eq(%w[aaativa zzativa])
    end
  end
end
```

Criar `spec/services/city_workers/child_spec.rb`:

```ruby
require "rails_helper"

# Plano 5 (spike 2): o filho de uma unidade liga o Solid Queue do PROCESSO INTEIRO
# ao banco certo antes de entrar no supervisor. Roda de verdade num fork, e o
# resultado volta por um pipe. O fork abre conexões novas, então a cidade precisa
# estar comitada (sem transação de fixture).
RSpec.describe CityWorkers::Child do
  self.use_transactional_tests = false

  let(:slug) { "filho#{SecureRandom.hex(3)}" }
  let!(:city) do
    City.create!(slug: slug, name: "Filho", status: "active", schema_version: CitySchema.expected_version.to_s,
                 database_url: city_database_url("rota_saude_test_city_b"), encryption_key: SecureRandom.hex(32))
  end

  after do
    CityConnection.forget(city.shard)
    City.where(id: city.id).delete_all
  end

  # Roda `prepare` num fork, numa thread nova (sem o connected_to que o harness
  # empilhou na thread principal), e devolve o que o processo filho enxerga.
  def prepare_in_fork(unit)
    reader, writer = IO.pipe
    pid = Process.fork do
      reader.close
      result = Thread.new do
        options = described_class.prepare(unit)
        {
          "context" => CityWorkers::Context.city_slug,
          "queue_database" => SolidQueue::Job.connection_db_config.database,
          "queue_database_other_thread" => Thread.new { SolidQueue::Job.connection_db_config.database }.value,
          "config_file" => options[:config_file].to_s.delete_prefix("#{Rails.root}/"),
          "recurring_schedule_file" => options[:recurring_schedule_file].to_s.delete_prefix("#{Rails.root}/"),
          "group_leader" => Process.getpgrp == Process.pid
        }
      rescue StandardError => e
        { "error" => e.class.name }
      end.value
      writer.write(result.to_json)
      writer.close
      exit!(0)
    end
    writer.close
    output = reader.read
    Process.wait(pid)
    JSON.parse(output)
  end

  it "binds the whole process's Solid Queue to the city's database and marks the process's city" do
    expect(prepare_in_fork(CityWorkers::Unit.city(slug))).to eq(
      "context" => slug,
      "queue_database" => "rota_saude_test_city_b",
      "queue_database_other_thread" => "rota_saude_test_city_b",
      "config_file" => "config/queue.yml",
      "recurring_schedule_file" => "config/recurring.yml",
      "group_leader" => true
    )
  end

  it "keeps the platform supervisor on the platform database, with no city" do
    expect(prepare_in_fork(CityWorkers::Unit.platform)).to include(
      "context" => nil,
      "queue_database" => "rota_saude_platform_test",
      "queue_database_other_thread" => "rota_saude_platform_test",
      "config_file" => "config/queue_platform.yml",
      "recurring_schedule_file" => "config/recurring_platform.yml"
    )
  end

  it "refuses a city that is no longer active or whose schema is behind" do
    city.update!(status: "suspended")
    expect(prepare_in_fork(CityWorkers::Unit.city(slug))).to eq("error" => "CityWorkers::Child::CityUnavailable")

    city.update!(status: "active", schema_version: nil)
    expect(prepare_in_fork(CityWorkers::Unit.city(slug))).to eq("error" => "CityWorkers::Child::CityUnavailable")
  end
end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_workers`
Expected: FAIL com `uninitialized constant CityWorkers::Backoff` / `Manager` / `Child` / `Unit`.

- [ ] **Step 3: Unidade, backoff, relógio e processos**

Criar `app/services/city_workers/unit.rb`:

```ruby
module CityWorkers
  # Um supervisor Solid Queue gerido pelo Manager: o da plataforma ou o de uma
  # cidade (Plano 5). Cada tipo tem sua configuração de filas e de recorrência.
  Unit = Data.define(:kind, :city_slug) do
    def self.platform = new(kind: :platform, city_slug: nil)
    def self.city(slug) = new(kind: :city, city_slug: slug)

    def key = kind == :platform ? "platform" : "city:#{city_slug}"
    def config_file = kind == :platform ? "config/queue_platform.yml" : "config/queue.yml"
    def recurring_schedule_file = kind == :platform ? "config/recurring_platform.yml" : "config/recurring.yml"
  end
end
```

Criar `app/services/city_workers/backoff.rb`:

```ruby
module CityWorkers
  # Espera antes de reiniciar um supervisor que morreu (spec §4, spike 2): um filho
  # cujo banco sumiu morre em 0,4 s e, sem espera, reiniciaria em loop apertado. A
  # espera dobra a cada falha seguida, até MAX; um filho que ficou de pé por
  # STABLE_AFTER volta a contar do começo.
  class Backoff
    BASE = 1.0
    MAX = 300.0
    STABLE_AFTER = 600.0

    def delay_for(consecutive_failures)
      return 0.0 if consecutive_failures <= 0

      [ BASE * (2**(consecutive_failures - 1)), MAX ].min
    end

    def failures_after_exit(previous_failures, ran_for:)
      ran_for >= STABLE_AFTER ? 1 : previous_failures + 1
    end
  end
end
```

Criar `app/services/city_workers/clock.rb`:

```ruby
module CityWorkers
  # Relógio monotônico do Manager (substituível nos specs).
  class Clock
    def now = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    def sleep(seconds) = Kernel.sleep(seconds)
  end
end
```

Criar `app/services/city_workers/spawner.rb`:

```ruby
module CityWorkers
  # Processos reais do Manager: um fork por unidade. Cada filho vira líder do
  # próprio grupo de processo (Child.prepare), e o KILL vai para o grupo inteiro —
  # sem deixar workers e dispatcher do Solid Queue órfãos. O TERM vai só para o
  # supervisor, que encerra os próprios filhos com graça.
  class Spawner
    def spawn(unit)
      Process.fork do
        Process.setproctitle("rota-city-workers #{unit.key}")
        Child.run(unit)
      end
    end

    def terminate(pid)
      signal(:TERM, pid)
    end

    def kill(pid)
      signal(:KILL, -pid)
    end

    def reap
      exits = []
      loop do
        pid, status = Process.waitpid2(-1, Process::WNOHANG)
        break unless pid

        exits << [ pid, status ]
      end
      exits
    rescue Errno::ECHILD
      exits
    end

    private

    def signal(name, pid)
      Process.kill(name, pid)
    rescue Errno::ESRCH
      nil
    end
  end
end
```

Criar `app/services/city_workers/child.rb`:

```ruby
module CityWorkers
  # O que roda dentro do fork de uma unidade (spec §4, spike 2): liga o Solid Queue
  # ao banco certo e entra no supervisor.
  #   - cidade: o padrão do SolidQueue::Record passa a ser o banco da cidade para o
  #     processo inteiro (todas as threads do supervisor e dos workers que ele
  #     forka), e CityWorkers::Context marca a cidade do processo;
  #   - plataforma: o padrão já é o banco de plataforma (config.solid_queue.connects_to).
  # Uma cidade que deixou de estar active, ou atrasou o schema, entre a leitura do
  # catálogo e o fork levanta CityUnavailable: o filho sai com erro, o Manager
  # espera o backoff e a tira no próximo poll.
  module Child
    class CityUnavailable < StandardError; end

    module_function

    def run(unit)
      SolidQueue::Supervisor.start(**prepare(unit))
    end

    def prepare(unit)
      Process.setpgid(0, 0)

      if unit.kind == :city
        city = City.find_by(slug: unit.city_slug)
        unless city&.servable? && !CitySchema.behind?(city)
          raise CityUnavailable, "cidade #{unit.city_slug} fora do ar ou com schema atrasado"
        end

        CityWorkers::Context.city_slug = city.slug
        CityConnection.ensure_pool(city)
        SolidQueue::Record.establish_connection(CityConnection.database_config(city))
      end

      {
        mode: :fork,
        config_file: Rails.root.join(unit.config_file),
        recurring_schedule_file: Rails.root.join(unit.recurring_schedule_file)
      }
    end
  end
end
```

- [ ] **Step 4: O gerente e o executável**

Criar `app/services/city_workers/manager.rb`:

```ruby
module CityWorkers
  # Mantém um supervisor Solid Queue por cidade ativa, mais o da plataforma (spec
  # banco-por-cidade §4 e riscos abertos 1, Plano 5). Um laço de reconciliação:
  #   - lê o catálogo a cada poll_interval: cidades active com schema em dia — uma
  #     cidade atrasada sai e volta sozinha depois do city:migrate:all;
  #   - sobe o que falta e manda TERM a quem saiu do catálogo;
  #   - reinicia quem morreu sem ter sido parado, com Backoff.
  # Não guarda estado em banco: cada gerente descobre as cidades sozinho. Processo,
  # relógio e catálogo são injetados, para o laço ser testável sem fork.
  class Manager
    Running = Struct.new(:unit, :pid, :started_at, keyword_init: true)

    attr_reader :running

    def self.active_city_slugs
      City.where(status: "active").order(:slug).reject { |city| CitySchema.behind?(city) }.map(&:slug)
    end

    def initialize(spawner:, clock:, catalog:, backoff: Backoff.new, poll_interval: 30.0, logger: Rails.logger)
      @spawner = spawner
      @clock = clock
      @catalog = catalog
      @backoff = backoff
      @poll_interval = poll_interval
      @logger = logger
      @running = {}
      @failures = Hash.new(0)
      @next_start_at = {}
      @stopping = {}
      @desired = nil
      @last_poll_at = nil
    end

    def run(stop_signal:)
      until stop_signal.call
        tick
        @clock.sleep(1.0)
      end
      shutdown(timeout: SolidQueue.shutdown_timeout.to_f + 5.0)
    end

    def tick
      refresh_desired if poll_due?
      reap
      stop_undesired
      start_missing
    end

    def shutdown(timeout:)
      running.each_value do |entry|
        @stopping[entry.pid] = true
        @spawner.terminate(entry.pid)
      end

      deadline = @clock.now + timeout
      until running.empty? || @clock.now >= deadline
        reap
        @clock.sleep(0.1) unless running.empty?
      end

      running.each_value do |entry|
        @logger.warn("[city_workers] #{entry.unit.key} não parou em #{timeout}s: KILL (pid #{entry.pid})")
        @spawner.kill(entry.pid)
      end
      reap
    end

    private

    def desired
      @desired || [ Unit.platform ]
    end

    def poll_due?
      @last_poll_at.nil? || @clock.now - @last_poll_at >= @poll_interval
    end

    def refresh_desired
      @last_poll_at = @clock.now
      @desired = [ Unit.platform ] + @catalog.call.map { |slug| Unit.city(slug) }
    rescue StandardError => e
      @logger.error("[city_workers] catálogo indisponível, mantendo o que roda: #{e.class}: #{CitySchema.redact(e.message)}")
    end

    def reap
      @spawner.reap.each do |pid, status|
        entry = running.values.find { |candidate| candidate.pid == pid } or next
        key = entry.unit.key
        running.delete(key)

        if @stopping.delete(pid)
          @failures.delete(key)
          @next_start_at.delete(key)
          @logger.info("[city_workers] #{key} parou (pid #{pid})")
        else
          @failures[key] = @backoff.failures_after_exit(@failures[key], ran_for: @clock.now - entry.started_at)
          delay = @backoff.delay_for(@failures[key])
          @next_start_at[key] = @clock.now + delay
          @logger.warn("[city_workers] #{key} morreu (pid #{pid}, #{status}); reinicia em #{delay}s (falha #{@failures[key]})")
        end
      end
    end

    def stop_undesired
      keys = desired.map(&:key)
      running.each_value do |entry|
        next if keys.include?(entry.unit.key) || @stopping[entry.pid]

        @logger.info("[city_workers] #{entry.unit.key} saiu do catálogo: TERM (pid #{entry.pid})")
        @stopping[entry.pid] = true
        @spawner.terminate(entry.pid)
      end
    end

    def start_missing
      desired.each do |unit|
        next if running.key?(unit.key)
        next if @next_start_at[unit.key] && @clock.now < @next_start_at[unit.key]

        pid = @spawner.spawn(unit)
        running[unit.key] = Running.new(unit: unit, pid: pid, started_at: @clock.now)
        @logger.info("[city_workers] #{unit.key} subiu (pid #{pid})")
      end
    end
  end
end
```

Criar `bin/city_workers`:

```ruby
#!/usr/bin/env ruby
# Worker do Rota Saúde (Plano 5, spec banco-por-cidade §4): um supervisor Solid
# Queue por cidade ativa, mais o da plataforma, geridos por CityWorkers::Manager.
# Substitui bin/jobs. TERM/INT: repassa TERM aos supervisores, espera
# SolidQueue.shutdown_timeout + 5 s e mata os grupos de quem sobrar.
require_relative "../config/environment"

stop = false
%w[TERM INT].each { |signal| Signal.trap(signal) { stop = true } }

manager = CityWorkers::Manager.new(
  spawner: CityWorkers::Spawner.new,
  clock: CityWorkers::Clock.new,
  catalog: CityWorkers::Manager.method(:active_city_slugs),
  poll_interval: Float(ENV.fetch("CITY_WORKERS_POLL_SECONDS", "30"))
)
manager.run(stop_signal: -> { stop })
```

Run (no host, dentro de `apps/api`): `chmod +x bin/city_workers`

- [ ] **Step 5: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_workers`
Expected: PASS, com todos os exemplos (backoff 2, manager 9, child 3) e saída sem ruído.

- [ ] **Step 6: Suíte inteira e eager load**

Run (timeout ≥ 600000 ms): `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

Run: `docker compose exec -T api bash -c 'RAILS_ENV=development bin/rails runner "Rails.application.eager_load!; puts :eager_load_ok"'`
Expected: `eager_load_ok`.

- [ ] **Step 7: Commit**

```bash
git add app/services/city_workers bin/city_workers spec/services/city_workers
git commit -m "Supervise one Solid Queue per active city plus the platform, restarting with backoff

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 5: Cache na plataforma, aposentadoria do banco compartilhado e `bin/city_workers` no deploy

**Files:**
- Modify: `config/database.yml`
- Modify: `config/cache.yml`
- Modify: `config/puma.rb`
- Modify: `bin/docker-entrypoint`
- Modify: `lib/tasks/city.rake`
- Delete: `db/schema.rb`, `db/queue_schema.rb`, `db/cache_schema.rb`, `bin/jobs`
- Create: `spec/architecture/shared_database_retired_spec.rb`
- Modify: `spec/tasks/city_rake_spec.rb`
- Modify: `spec/config/solid_queue_configuration_spec.rb`
- Modify: `deploy/production/deploy.yml`, `deploy/development/deploy.yml`
- Modify: `deploy/production/secrets`, `deploy/development/secrets`
- Modify: `deploy/SECRETS.md`
- Modify: `README.md`
- Modify: `<raiz-do-monorepo>/start.sh` e
  `<raiz-do-monorepo>/docker-compose.yml` (fora do git)

**Interfaces:**
- Consumes: tabelas de fila e cache na plataforma (Task 1), `bin/city_workers` (Task 4).
- Produces:
  - `primary` → `rota_saude_no_city_selected` (`database_tasks: false`), sem config `queue`, com `cache` → banco de
    plataforma.
  - O worker de dev e de produção roda `./bin/city_workers`.

- [ ] **Step 1: Specs (falham)**

Criar `spec/architecture/shared_database_retired_spec.rb`:

```ruby
require "rails_helper"

# Plano 5 (decisão do usuário): o banco compartilhado rota_saude_<env> não é mais
# usado. Fila: cidade e plataforma; cache: plataforma; ActiveRecord::Base: banco
# vazio que falha fechado.
RSpec.describe "Shared database retired" do
  def database_yml
    YAML.safe_load(ERB.new(Rails.root.join("config/database.yml").read).result, aliases: true)
  end

  it "declares no queue database and keeps primary on the empty database, outside database tasks" do
    %w[development test production].each do |env|
      expect(database_yml[env]).not_to have_key("queue"), "config/database.yml: queue under #{env}"
      expect(database_yml[env]["primary"]).to include("database" => "rota_saude_no_city_selected", "database_tasks" => false)
    end
  end

  it "puts Solid Cache on the platform database, outside database tasks" do
    %w[development production].each do |env|
      expect(database_yml[env]["cache"]).to include("username" => "rota_platform", "database_tasks" => false)
    end
    expect(database_yml["development"]["cache"]["database"]).to eq(database_yml["development"]["platform"]["database"])
    expect(ActiveSupport::ConfigurationFile.parse(Rails.root.join("config/cache.yml"))["production"]).to include("database" => "cache")
  end

  it "resolves ActiveRecord::Base to the empty database" do
    expect(ActiveRecord::Base.connection_db_config.database).to eq("rota_saude_no_city_selected")
  end

  it "leaves no schema dump or entry point of the shared database behind" do
    %w[db/schema.rb db/queue_schema.rb db/cache_schema.rb bin/jobs].each do |path|
      expect(Rails.root.join(path)).not_to exist
    end
  end
end
```

Em `spec/config/solid_queue_configuration_spec.rb`, acrescentar antes do `end` final:

```ruby
  # Solid Queue valida pool ≥ maior número de threads de um worker + 2 contra o
  # pool do banco do processo (RAILS_MAX_THREADS do worker).
  it "gives the Kamal worker a database pool that fits the largest worker" do
    %w[development production].each do |env|
      deploy = YAML.safe_load(Rails.root.join("deploy/#{env}/deploy.yml").read)
      pool = Integer(deploy.dig("servers", "worker", "env", "clear", "RAILS_MAX_THREADS"))
      threads = config("config/queue.yml", env).fetch("workers").map { |worker| worker["threads"] }.max

      expect(pool).to be >= threads + 2, "deploy/#{env}/deploy.yml: RAILS_MAX_THREADS #{pool} < #{threads} + 2"
      expect(deploy.dig("servers", "worker", "cmd")).to eq("./bin/city_workers")
    end
  end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/architecture/shared_database_retired_spec.rb spec/config/solid_queue_configuration_spec.rb`
Expected: FAIL. Os motivos esperados são:
- `queue` presente;
- `primary` em `rota_saude_test`;
- `cache.yml` sem `database`;
- os arquivos antigos ainda existem;
- o worker do Kamal ainda roda `./bin/jobs` sem `RAILS_MAX_THREADS`.

- [ ] **Step 3: `config/database.yml` e `config/cache.yml`**

Substituir o conteúdo de `config/database.yml` por:

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS", 5) %>
  prepared_statements: true
  variables:
    statement_timeout: 15s

# Plano 5: não existe mais banco compartilhado. Domínio de cada cidade: banco da
# cidade (CityRecord/CityConnection). Catálogo, operadores, fila de plataforma e
# Solid Cache: banco de plataforma. Fila de cada cidade: banco da cidade
# (SolidQueue::Record + CityConnection).
#
# `primary` existe porque ActiveRecord::Base precisa de uma config. Aponta para o
# banco VAZIO rota_saude_no_city_selected (o mesmo do city_unset) e fica fora das
# tasks de banco (database_tasks: false — também fora da checagem de migrations
# pendentes do dev): nada no app usa ActiveRecord::Base para dado, e uma query
# solta nele falha fechada.
development:
  primary:
    <<: *default
    database: rota_saude_no_city_selected
    username: rota_app
    password: <%= ENV.fetch("ROTA_APP_PASSWORD", "rota_app") %>
    host: <%= ENV.fetch("DATABASE_HOST", "127.0.0.1") %>
    port: <%= ENV.fetch("DATABASE_PORT", 5432) %>
    database_tasks: false
  # Banco de plataforma — catálogo de cidades, roteamento, contas de operador e fila
  # de plataforma. NUNCA guarda dado de cidadão. Ver spec 2026-09-12-banco-por-cidade-design.
  platform:
    <<: *default
    database: rota_saude_platform_development
    username: rota_platform
    password: <%= ENV.fetch("ROTA_PLATFORM_PASSWORD", "rota_platform") %>
    host: <%= ENV.fetch("DATABASE_HOST", "127.0.0.1") %>
    port: <%= ENV.fetch("DATABASE_PORT", 5432) %>
    migrations_paths: db/platform_migrate
    schema_dump: platform_schema.rb
  # Solid Cache (config/cache.yml `database: cache`) no banco de plataforma (Plano
  # 5). A tabela vem de db/platform_migrate; esta config não roda tasks.
  cache:
    <<: *default
    database: rota_saude_platform_development
    username: rota_platform
    password: <%= ENV.fetch("ROTA_PLATFORM_PASSWORD", "rota_platform") %>
    host: <%= ENV.fetch("DATABASE_HOST", "127.0.0.1") %>
    port: <%= ENV.fetch("DATABASE_PORT", 5432) %>
    database_tasks: false
  # Destino do shard `bootstrap` do CityRecord: banco que EXISTE (criado vazio por
  # city:test_databases) sem NENHUMA tabela — query de domínio fora de
  # CityConnection.with levanta PG::UndefinedTable em vez de ler dado real.
  city_unset:
    <<: *default
    database: rota_saude_no_city_selected
    username: rota_app
    password: <%= ENV.fetch("ROTA_APP_PASSWORD", "rota_app") %>
    host: <%= ENV.fetch("DATABASE_HOST", "127.0.0.1") %>
    port: <%= ENV.fetch("DATABASE_PORT", 5432) %>
    database_tasks: false

test:
  primary:
    <<: *default
    database: rota_saude_no_city_selected
    username: rota_app
    password: <%= ENV.fetch("ROTA_APP_PASSWORD", "rota_app") %>
    host: <%= ENV.fetch("DATABASE_HOST", "127.0.0.1") %>
    port: <%= ENV.fetch("DATABASE_PORT", 5432) %>
    database_tasks: false
  platform:
    <<: *default
    database: rota_saude_platform_test
    username: rota_platform
    password: <%= ENV.fetch("ROTA_PLATFORM_PASSWORD", "rota_platform") %>
    host: <%= ENV.fetch("DATABASE_HOST", "127.0.0.1") %>
    port: <%= ENV.fetch("DATABASE_PORT", 5432) %>
    migrations_paths: db/platform_migrate
    schema_dump: platform_schema.rb
  city_unset:
    <<: *default
    database: rota_saude_no_city_selected
    username: rota_app
    password: <%= ENV.fetch("ROTA_APP_PASSWORD", "rota_app") %>
    host: <%= ENV.fetch("DATABASE_HOST", "127.0.0.1") %>
    port: <%= ENV.fetch("DATABASE_PORT", 5432) %>
    database_tasks: false

production:
  primary:
    <<: *default
    url: <%= ENV.fetch("CITY_UNSET_DATABASE_URL") { "" } %>
    database: rota_saude_no_city_selected
    username: rota_app
    password: <%= ENV.fetch("ROTA_APP_PASSWORD") %>
    pool: <%= ENV.fetch("RAILS_MAX_THREADS", 10) %>
    database_tasks: false
  platform:
    <<: *default
    url: <%= ENV.fetch("PLATFORM_DATABASE_URL") { "" } %>
    username: rota_platform
    password: <%= ENV.fetch("ROTA_PLATFORM_PASSWORD") %>
    migrations_paths: db/platform_migrate
    schema_dump: platform_schema.rb
    pool: <%= ENV.fetch("RAILS_MAX_THREADS", 10) %>
  cache:
    <<: *default
    url: <%= ENV.fetch("PLATFORM_DATABASE_URL") { "" } %>
    username: rota_platform
    password: <%= ENV.fetch("ROTA_PLATFORM_PASSWORD") %>
    pool: <%= ENV.fetch("RAILS_MAX_THREADS", 10) %>
    database_tasks: false
  # NÃO reaproveita PLATFORM_DATABASE_URL nem nenhuma URL de banco real: com a URL
  # apontando para um banco de verdade, o `database` embutido nela venceria o
  # `database:` explícito abaixo (UrlConfig mescla por cima), e o shard bootstrap
  # voltaria a servir dado. CITY_UNSET_DATABASE_URL é uma env dedicada ao banco
  # vazio.
  city_unset:
    <<: *default
    url: <%= ENV.fetch("CITY_UNSET_DATABASE_URL") { "" } %>
    database: rota_saude_no_city_selected
    username: rota_app
    password: <%= ENV.fetch("ROTA_APP_PASSWORD") %>
    pool: <%= ENV.fetch("RAILS_MAX_THREADS", 10) %>
    database_tasks: false
```

Substituir o conteúdo de `config/cache.yml` por:

```yaml
# Solid Cache no banco de PLATAFORMA (Plano 5): config `cache` de config/database.yml.
# Em test o store é :null_store (config/environments/test.rb) e nada é gravado.
default: &default
  store_options:
    max_age: <%= 60.days.to_i %>
    namespace: <%= "rota-saude-#{Rails.env}" %>

development:
  <<: *default
  database: cache
test:
  <<: *default
production:
  <<: *default
  database: cache
```

- [ ] **Step 4: Arquivos antigos, Puma, entrypoint e guarda do `city:load_schema`**

Run (em `apps/api`): `git rm db/schema.rb db/queue_schema.rb db/cache_schema.rb bin/jobs`

Em `config/puma.rb`, remover a linha `plugin :solid_queue if ENV["SOLID_QUEUE_IN_PUMA"] == "true"` e a linha em branco que
a precede. Com fila por cidade, o worker é sempre `bin/city_workers`.

Em `bin/docker-entrypoint`, trocar:

```bash
# O bootstrap do zero (databases, roles, extensions, schema de fila/cache) é do
# start.sh no dev e da infra no prod.
```

por:

```bash
# O bootstrap do zero (databases, roles, extensions) é do start.sh no dev e da
# infra no prod. Fila e cache vêm por migration (plataforma e cidade), não por
# carga de schema.
```

Em `lib/tasks/city.rake`, trocar:

```ruby
  protected_database_names = lambda do
    ActiveRecord::Base.configurations.configurations
      .select { |cfg| protected_role_names.include?(cfg.name) }
      .filter_map { |cfg| cfg.respond_to?(:database) ? cfg.database.presence : nil }
      .uniq
  end
```

por:

```ruby
  # Bancos compartilhados aposentados no Plano 5: primary/queue/cache apontavam para
  # eles. Não estão mais em database.yml, mas continuam existindo em dev e test com
  # dados antigos, então city:load_schema segue recusando-os.
  retired_database_names = %w[rota_saude_development rota_saude_test rota_saude_production].freeze

  protected_database_names = lambda do
    (ActiveRecord::Base.configurations.configurations
      .select { |cfg| protected_role_names.include?(cfg.name) }
      .filter_map { |cfg| cfg.respond_to?(:database) ? cfg.database.presence : nil } + retired_database_names)
      .uniq
  end
```

Em `spec/tasks/city_rake_spec.rb`, trocar `it "refuses a target that resolves to the shared database behind primary/admin (this environment)" do`
por `it "refuses a target that resolves to the retired shared test database" do`, e trocar o comentário:

```ruby
    # rota_saude_development is primary/admin/queue/cache's database in
    # development, not in the test env this spec runs under — the guard must
    # still catch it, since it checks every environment's protected configs.
```

por:

```ruby
    # rota_saude_development was the shared database until Plan 5. It is no longer
    # in config/database.yml but still exists in dev with old data, so the guard
    # keeps refusing it by name.
```

- [ ] **Step 5: Deploy (Kamal)**

Em `deploy/production/deploy.yml`, trocar:

```yaml
  worker:
    hosts:
      - worker1.rota-saude.example
    cmd: ./bin/jobs
    # Só o worker cria e apaga banco de cidade (Plano 4): o web nunca recebe a
    # credencial de rota_provisioner. Backup e offboarding rodam aqui
    # (kamal app exec --roles=worker) e gravam em CITY_BACKUP_DIR.
    env:
      secret:
        - PROVISIONER_DATABASE_URL
    options:
      memory: 1g
```

por:

```yaml
  worker:
    hosts:
      - worker1.rota-saude.example
    # Plano 5: um supervisor Solid Queue por cidade ativa, mais o da plataforma
    # (~6 processos por cidade). Só o worker cria e apaga banco de cidade (Plano 4):
    # o web nunca recebe a credencial de rota_provisioner. Backup e offboarding
    # rodam aqui (kamal app exec --roles=worker) e gravam em CITY_BACKUP_DIR.
    cmd: ./bin/city_workers
    env:
      clear:
        # Pool do banco ≥ maior número de threads de um worker em config/queue.yml + 2.
        RAILS_MAX_THREADS: "12"
      secret:
        - PROVISIONER_DATABASE_URL
    options:
      memory: 4g
```

E, em `env.secret`, remover as linhas `    - DATABASE_URL` e `    - ROTA_ADMIN_PASSWORD`.

Em `deploy/development/deploy.yml`, trocar:

```yaml
  worker:
    hosts:
      - dev.rota-saude.example
    cmd: ./bin/jobs
    # Só o worker cria e apaga banco de cidade (Plano 4). Backup e offboarding
    # rodam aqui e gravam em CITY_BACKUP_DIR.
    env:
      secret:
        - PROVISIONER_DATABASE_URL
    options:
      memory: 384m
```

por:

```yaml
  worker:
    hosts:
      - dev.rota-saude.example
    # Plano 5: um supervisor Solid Queue por cidade ativa, mais o da plataforma.
    # Só o worker cria e apaga banco de cidade (Plano 4). Backup e offboarding rodam
    # aqui e gravam em CITY_BACKUP_DIR.
    cmd: ./bin/city_workers
    env:
      clear:
        # Pool do banco ≥ maior número de threads de um worker em config/queue.yml + 2.
        RAILS_MAX_THREADS: "12"
      secret:
        - PROVISIONER_DATABASE_URL
    options:
      memory: 2g
```

E, em `env.secret`, remover as linhas `    - DATABASE_URL` e `    - ROTA_ADMIN_PASSWORD`.

Em `deploy/production/secrets` e em `deploy/development/secrets`, remover a linha que começa com `DATABASE_URL=` e a que
começa com `ROTA_ADMIN_PASSWORD=`, sem imprimir seus valores. Um comentário imediatamente acima que fale só do banco
compartilhado, como `# Postgres do accessory (host dedicado).`, também sai, a menos que ainda descreva
`POSTGRES_PASSWORD`; nesse caso fica.

Em `deploy/SECRETS.md`:
- Trocar ``- `ROTA_APP_PASSWORD` / `ROTA_ADMIN_PASSWORD` — senhas dos papéis Postgres (`rota_admin` só para fila/cache até o Plano 5).``
  por ``- `ROTA_APP_PASSWORD` — senha do papel `rota_app` (banco vazio `rota_saude_no_city_selected`).``.
- Remover a linha ``- `DATABASE_URL` — banco compartilhado (fila e cache).``.
- Na linha de `PLATFORM_DATABASE_URL`, acrescentar ao fim, antes do ponto final: `, fila de plataforma e Solid Cache`.
- No trecho `` `postgres-roles` (campos `rota_app`,
`rota_admin`, `rota_platform`, `provisioner_url`) ``, remover `` `rota_admin`, ``.

- [ ] **Step 6: README**

Em `README.md`:
- Remover a linha da tabela ``| `ROTA_ADMIN_PASSWORD` | `rota_admin` | `queue`, `cache` (até o Plano 5) |``.
- Na linha de `ROTA_APP_PASSWORD`, trocar a última coluna por `` `primary` e `city_unset` (banco vazio) ``.
- Na linha de `ROTA_PLATFORM_PASSWORD`, trocar a última coluna por `` `platform`, `cache` e a fila de plataforma ``.
- Remover a linha da tabela de bancos ``| `rota_saude_development`, `rota_saude_test` | `rota_saude` | `start.sh` |``.
- Na seção "Bootstrap do banco (do zero)", trocar o item que começa com `- **Banco compartilhado**` (até a linha que
  termina em `conexão de cidade (`CityRecord`), sem RLS.`) por:

```markdown
- **Banco compartilhado** — aposentado no Plano 5. `rota_saude_development` e `rota_saude_test` continuam existindo
  no Postgres de dev com dados antigos, mas nada os usa: a fila de cada cidade mora no banco dela, a fila de
  plataforma e o Solid Cache no banco de plataforma (`db/platform_migrate`), e `primary` aponta para o banco vazio
  `rota_saude_no_city_selected`. `city:load_schema` segue recusando esses nomes.
```

- Trocar o item que começa com ``- `config.active_record.dump_schema_after_migration` é `false` `` (até a linha
  `` `db:migrate:platform` antes desta mudança — Task 3 contava com isso). ``) por:

```markdown
- `config.active_record.dump_schema_after_migration` é `false` em development. Depois de qualquer migration em
  `db/platform_migrate/`, rode `bin/rails db:schema:dump:platform` e commite `db/platform_schema.rb`. Depois de uma
  migration em `db/city_migrate/`, atualize `db/city_schema.rb` à mão: o spec de paridade compara os dois.
- Upgrade do Solid Queue que mude tabelas exige migration nova em `db/city_migrate/` **e** em `db/platform_migrate/`
  (as duas usam `db/solid_queue_tables.rb`).
```

- Trocar o parágrafo final da seção:

```markdown
Migrations incrementais no dev seguem via `db:migrate` (entrypoint), normalmente
— `db/migrate/` fica vazio de propósito (só domínio de cidade mudava esse
diretório, e esse domínio saiu).
```

por:

```markdown
Migrations no dev: `bin/rails db:migrate` (plataforma) e `bin/rails city:migrate:all` (cidades) — ou `bin/migrate`,
que roda os dois. `db/migrate/` fica vazio de propósito.
```

- [ ] **Step 7: `start.sh` e `docker-compose.yml` (raiz do monorepo, fora do git)**

Em `<raiz-do-monorepo>/start.sh`:

Trocar o bloco:

```bash
DB_NAME="rota_saude_${RAILS_ENV}"
DB_TEST="rota_saude_test"

if ! psql -U postgres -lqt 2>/dev/null | cut -d \| -f1 | grep -qw "$DB_NAME" \
   && ! psql -lqt 2>/dev/null | cut -d \| -f1 | grep -qw "$DB_NAME"; then
  say "criando user e banco rota_saude no postgres do host"
  createuser -d rota_saude 2>/dev/null || true
  psql -d postgres -c "ALTER USER rota_saude WITH PASSWORD '$POSTGRES_PASSWORD';" >/dev/null
  createdb -O rota_saude "$DB_NAME" 2>/dev/null || true
  createdb -O rota_saude "$DB_TEST" 2>/dev/null || true
fi
```

por:

```bash
# Plano 5: não há mais banco compartilhado. rota_saude é o superusuário de
# bootstrap que cria os bancos de plataforma e de cidade (tasks abaixo).
PLATFORM_DB="rota_saude_platform_${RAILS_ENV}"

if ! psql -d postgres -tAc "SELECT 1 FROM pg_roles WHERE rolname='rota_saude'" 2>/dev/null | grep -q 1; then
  say "criando o superusuário de bootstrap rota_saude no postgres do host"
  createuser -s rota_saude 2>/dev/null || true
  psql -d postgres -c "ALTER USER rota_saude WITH PASSWORD '$POSTGRES_PASSWORD';" >/dev/null
fi
```

Trocar o bloco:

```bash
  warn "--reset: dropando $DB_NAME e $DB_TEST do host"
  dropdb --if-exists "$DB_NAME"
  dropdb --if-exists "$DB_TEST"
  dropdb --if-exists rota_saude_city_curitiba
  dropdb --if-exists rota_saude_city_maringa
  createdb -O rota_saude "$DB_NAME"
  createdb -O rota_saude "$DB_TEST"
```

por:

```bash
  warn "--reset: dropando os bancos das cidades de dev do host"
  dropdb --if-exists rota_saude_city_curitiba
  dropdb --if-exists rota_saude_city_maringa
```

Trocar tudo desde a linha `# --- roles + schema do banco compartilhado (fila/cache) -----------------------` até a
linha `fi` que fecha o bloco `if [ "${NEEDS_CACHE// /}" = "t" ]; then` (inclusive) por:

```bash
# --- roles do app ---------------------------------------------------------------
# rota_app acessa o banco vazio rota_saude_no_city_selected (primary/city_unset).
# Fila e cache não carregam mais schema aqui: vêm por migration (plataforma e
# cidade, Plano 5).
say "garantindo roles do app"
PGPASSWORD="$POSTGRES_PASSWORD" psql -U rota_saude -h localhost -d postgres -v ON_ERROR_STOP=1 -c "
DO \$\$ BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='rota_app') THEN CREATE ROLE rota_app LOGIN PASSWORD '${ROTA_APP_PASSWORD:-rota_app}'; END IF;
END \$\$;
" >/dev/null
```

Trocar as duas linhas do resumo final:

```bash
  postgres ............... host:$POSTGRES_PORT  db=$DB_NAME  user=rota_saude
```

por:

```bash
  postgres ............... host:$POSTGRES_PORT  db=$PLATFORM_DB  user=rota_saude
```

e:

```bash
  banco .................. psql -U rota_saude -d $DB_NAME
```

por:

```bash
  banco .................. psql -U rota_saude -d $PLATFORM_DB
```

Run (leitura, na raiz do monorepo): `grep -n "DB_NAME\|DB_TEST\|rota_admin\|ROTA_ADMIN\|schema:load" start.sh`
Expected: nenhuma linha.

Em `<raiz-do-monorepo>/docker-compose.yml`:
- trocar `    command: ./bin/jobs start` por `    command: ./bin/city_workers`;
- remover a linha `  ROTA_ADMIN_PASSWORD: ${ROTA_ADMIN_PASSWORD:-rota_admin}`.

NÃO rode o `start.sh`.

- [ ] **Step 8: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/architecture spec/config spec/tasks/city_rake_spec.rb`
Expected: PASS.

- [ ] **Step 9: Suíte inteira, boot, cache e migrations**

Run (timeout ≥ 600000 ms): `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

Run: `docker compose exec -T api bash -c 'RAILS_ENV=development bin/rails runner "Rails.application.eager_load!; puts ActiveRecord::Base.connection_db_config.database; puts SolidQueue::Job.connection_db_config.database; puts SolidCache::Entry.connection_db_config.database"'`
Expected: `rota_saude_no_city_selected`, `rota_saude_platform_development`, `rota_saude_platform_development`.

Run: `docker compose exec -T api ./bin/rails db:migrate`
Expected: saída 0, só a config `platform` (nada a aplicar).

Run: `docker compose up -d --force-recreate --no-build api worker`
Run: `curl -s -o /dev/null -w "up %{http_code}\n" http://localhost:3030/up; curl -s -o /dev/null -w "curitiba %{http_code}\n" -H "Host: curitiba.localhost" http://localhost:3030/admin/api/overview`
Expected: `up 200`, `curitiba 401`.

Run (grava uma entrada de cache via rate limit, com credencial inválida de propósito):
`curl -s -o /dev/null -w "login %{http_code}\n" -H "Host: curitiba.localhost" -H "Content-Type: application/json" -d '{"email_address":"ninguem@curitiba.demo","password":"errada"}' http://localhost:3030/session`
Run (leitura): `docker compose exec -T api bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -h $DATABASE_HOST -U rota_saude -tA -d rota_saude_platform_development -c "select count(*) > 0 from solid_cache_entries"'`
Expected: `login 401` e `t`.

Run: `docker compose logs worker --since 2m | grep city_workers`
Expected: `platform subiu`, `city:curitiba subiu` e `city:maringa subiu`, sem `morreu`. A prova completa é a Task 6.

- [ ] **Step 10: Commit**

```bash
git add config/database.yml config/cache.yml config/puma.rb bin/docker-entrypoint lib/tasks/city.rake \
  spec/architecture/shared_database_retired_spec.rb spec/tasks/city_rake_spec.rb spec/config/solid_queue_configuration_spec.rb \
  deploy/production/deploy.yml deploy/development/deploy.yml deploy/production/secrets deploy/development/secrets \
  deploy/SECRETS.md README.md
git commit -m "Retire the shared database, move Solid Cache to the platform and run bin/city_workers

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

O `git rm` do Step 4 já colocou as remoções no índice. `start.sh` e `docker-compose.yml` ficam fora do git; relate as
edições no relatório.

---

### Task 6: Prova do worker por cidade em dev e runbook

**Files:**
- Modify: `README.md` (seção nova)

**Interfaces:**
- Consumes: tudo das Tasks 1–5.
- Produces: prova ponta a ponta colada no relatório e seção "Worker por cidade (Plano 5)" no README.

- [ ] **Step 1: Estado inicial do banco compartilhado (leitura)**

Run: `docker compose exec -T api bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -h $DATABASE_HOST -U rota_saude -tA -d rota_saude_development -c "select count(*), max(id) from solid_queue_jobs"'`
Anote o resultado. No Step 8, ele precisa ser o mesmo: nada mais escreve no compartilhado.

- [ ] **Step 2: Supervisores, processos e recorrência por banco (leitura)**

Run: `docker compose logs worker --since 10m | grep city_workers | tail -20`
Expected: `platform subiu`, `city:curitiba subiu` e `city:maringa subiu`.

Run (leitura):

```bash
docker compose exec -T api bash -c 'export PGPASSWORD=$POSTGRES_PASSWORD; for db in rota_saude_platform_development rota_saude_city_curitiba rota_saude_city_maringa; do echo "== $db"; psql -h $DATABASE_HOST -U rota_saude -tA -d $db -c "select kind, count(*) from solid_queue_processes group by kind order by kind" -c "select string_agg(key, '"'"','"'"' order by key) from solid_queue_recurring_tasks"; done'
```

Expected:
- plataforma: `Dispatcher|1`, `Scheduler|1`, `Supervisor|1`, `Worker|1`; tarefas `clear_solid_queue_finished` e
  `purge_platform_access`;
- curitiba e maringa: `Dispatcher|1`, `Scheduler|1`, `Supervisor|1`, `Worker|3`; tarefas de cidade (sem
  `purge_platform_access`).

- [ ] **Step 3: Job de cidade processado na fila da cidade**

Run: `curl -s -o /dev/null -w "reset %{http_code}\n" -H "Host: curitiba.localhost" -H "Content-Type: application/json" -d '{"email_address":"admin@curitiba.demo"}' http://localhost:3030/passwords`
Expected: `reset 204` (o controller responde 204 para e-mail existente ou não; aqui o usuário existe, então o e-mail é
enfileirado).

Aguarde 5 s.
Run (leitura):

```bash
docker compose exec -T api bash -c 'export PGPASSWORD=$POSTGRES_PASSWORD; for db in rota_saude_city_curitiba rota_saude_city_maringa rota_saude_platform_development; do echo "== $db"; psql -h $DATABASE_HOST -U rota_saude -tA -d $db -c "select count(*) filter (where finished_at is not null), count(*) from solid_queue_jobs where class_name = '"'"'CityMailDeliveryJob'"'"' and created_at > now() - interval '"'"'10 minutes'"'"'"; done'
```

Expected: curitiba `1|1`; maringa `0|0`; plataforma `0|0`.

- [ ] **Step 4: Job de plataforma e cidade nova assumida pelo gerente (`cascavel`)**

Run (raiz do monorepo, UMA chamada de bash):

```bash
JAR="$(mktemp -d)/op.jar"; API=http://localhost:3030
CODE=$(docker compose exec -T api ./bin/rails runner 'print ROTP::TOTP.new(ENV.fetch("DEV_OPERATOR_OTP_SECRET", "TQLRHWIAKEISPIW6YY3IAKGCLVNPF4EV")).now' | tail -1)
SID=$(curl -s -c "$JAR" -b "$JAR" -H 'Host: admin.localhost' -H 'Content-Type: application/json' \
  -d '{"email_address":"dev@local","password":"dev-password"}' "$API/session" | sed -E 's/.*"session_id":"([^"]+)".*/\1/')
curl -s -o /dev/null -w "challenge %{http_code}\n" -c "$JAR" -b "$JAR" -H 'Host: admin.localhost' \
  -H 'Content-Type: application/json' -d "{\"session_id\":\"$SID\",\"code\":\"$CODE\"}" "$API/session/challenge"
BODY=$(curl -s -b "$JAR" -H 'Host: admin.localhost' -H 'Content-Type: application/json' -w ' %{http_code}' \
  -d '{"slug":"cascavel","name":"Cascavel","uf":"PR","ibge_code":"4104808","admin_email":"prefeita@cascavel.demo","alert_email":"alertas@cascavel.demo"}' \
  "$API/cities")
echo "POST /cities → $BODY"
ID=$(echo "$BODY" | sed -E 's/.*"id":"([^"]+)".*/\1/')
for i in $(seq 1 60); do
  STATUS=$(curl -s -b "$JAR" -H 'Host: admin.localhost' "$API/cities/$ID")
  echo "$STATUS" | grep -q '"status":"active"' && break
  sleep 2
done
echo "GET /cities/:id → $STATUS"
sleep 35
docker compose logs worker --since 3m | grep "city:cascavel"
```

Expected:
- `challenge 200`, `POST /cities → {"id":"<uuid>"} 202`, `"status":"active"`;
- no log, `city:cascavel subiu` em até 30 s depois da ativação.

Run (leitura):

```bash
docker compose exec -T api bash -c 'export PGPASSWORD=$POSTGRES_PASSWORD; psql -h $DATABASE_HOST -U rota_saude -tA -d rota_saude_platform_development -c "select class_name, count(*) filter (where finished_at is not null) from solid_queue_jobs where created_at > now() - interval '"'"'10 minutes'"'"' group by class_name order by 1"; psql -h $DATABASE_HOST -U rota_saude -tA -d rota_saude_city_cascavel -c "select kind, count(*) from solid_queue_processes group by kind order by kind"'
```

Expected:
- plataforma: `CityMailDeliveryJob|1` (e-mail do convite) e `ProvisionCityJob|1`;
- cascavel: `Dispatcher|1`, `Scheduler|1`, `Supervisor|1`, `Worker|3`.

- [ ] **Step 5: Supervisor que morre é reiniciado com backoff**

Run: `docker compose logs worker --since 30m | grep "city:maringa subiu" | tail -1`
Pegue o pid da última linha (`... subiu (pid N)`).
Run: `docker compose exec -T worker bash -c 'kill -TERM <N>'`

Aguarde 5 s.
Run: `docker compose logs worker --since 1m | grep "city:maringa"`
Expected: `city:maringa morreu (pid N, ...); reinicia em 1.0s (falha 1)` seguido de `city:maringa subiu (pid M)`. A parada
não foi pedida pelo gerente, então conta como falha. Curitiba, cascavel e plataforma não aparecem.

Repita o `kill -TERM` com o pid novo em menos de 10 min.
Expected: `reinicia em 2.0s (falha 2)`.

- [ ] **Step 6: Suspensão tira a cidade do gerente; offboarding de `cascavel`**

Run: `docker compose exec -T api ./bin/rails 'city:suspend[cascavel]'`
Aguarde 35 s.
Run: `docker compose logs worker --since 1m | grep "city:cascavel"`
Expected: `city:cascavel saiu do catálogo: TERM (pid ...)` e `city:cascavel parou (pid ...)`.

Aguarde mais 30 s (o offboarding exige 60 s desde a suspensão).
Run: `docker compose exec -T -e CONFIRM=cascavel api ./bin/rails 'city:offboard[cascavel]'`
Expected: `[city:offboard] cascavel → archived; banco e role apagados; dump final: /rails/tmp/city_backups/cascavel-<timestamp>.dump`.

Run: `docker compose exec -T api bash -c 'rm -f /rails/tmp/city_backups/cascavel-*.dump'`

- [ ] **Step 7: Parada e volta do worker sem órfãos**

Run: `docker compose stop worker`
Run: `docker compose logs worker --since 1m | grep city_workers | tail -10`
Expected: `platform parou`, `city:curitiba parou` e `city:maringa parou`, sem `KILL`.

Run: `docker compose start worker`
Aguarde 10 s.
Run: `docker compose logs worker --since 30s | grep city_workers`
Expected: as três unidades `subiu`. Cascavel não aparece, porque está archived.

Run (leitura): `docker compose exec -T api bash -c 'export PGPASSWORD=$POSTGRES_PASSWORD; for db in rota_saude_platform_development rota_saude_city_curitiba rota_saude_city_maringa; do psql -h $DATABASE_HOST -U rota_saude -tA -d $db -c "select count(*) from solid_queue_processes where kind = '"'"'Supervisor'"'"'"; done'`
Expected: `1` em cada banco (supervisores velhos saíram do registro).

- [ ] **Step 8: Compartilhado intocado, suíte e boot**

Repita o comando do Step 1.
Expected: o mesmo resultado anotado.

Run (timeout ≥ 600000 ms): `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec --seed 7171`
Run (timeout ≥ 600000 ms): `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec --seed 8282`
Expected: 0 failures nas duas. Depois de cada uma, a checagem de sobras dá `0`, `0` e `0|0`:
`docker compose exec -T api bash -c 'export PGPASSWORD=$POSTGRES_PASSWORD; psql -h $DATABASE_HOST -U rota_saude -tA -d postgres -c "select count(*) from pg_database where datname like '"'"'rota_saude_test_city_prov%'"'"' or datname like '"'"'rota_saude_test_scratch_%'"'"'" -c "select count(*) from pg_roles where rolname like '"'"'rota_test_city_prov%'"'"'"; psql -h $DATABASE_HOST -U rota_saude -tA -d rota_saude_platform_test -c "select (select count(*) from cities), (select count(*) from platform_events)"'`

Run: `curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3030/up`
Expected: `200`.

- [ ] **Step 9: Runbook no README**

Em `README.md`, imediatamente antes de `## Bootstrap do banco (do zero)`, acrescentar:

```markdown
## Worker por cidade (Plano 5)

`bin/city_workers` (container `worker` no dev, papel `worker` no Kamal) roda **um supervisor Solid Queue por cidade
ativa, mais o da plataforma**:

- **Fila da cidade** — no banco dela: webhook, envio de WhatsApp, alertas, relatórios, e-mails de redefinição de senha,
  tarefas recorrentes de `config/recurring.yml` (agendadas por cidade). Workers em `config/queue.yml`: `urgent`
  isolado, `realtime,default`, `reports,housekeeping`.
- **Fila de plataforma** — no banco de plataforma: `ProvisionCityJob`, e-mail do convite, `PurgePlatformAccessJob` e
  `config/recurring_platform.yml`. Só entram jobs de `PlatformQueue::JOBS`/`MAILERS`: um job de cidade enfileirado
  fora de uma cidade levanta `PlatformQueue::Misplaced`. Job novo de plataforma precisa entrar nessa lista.
- **Catálogo** — o gerente lê as cidades `active` com schema em dia a cada `CITY_WORKERS_POLL_SECONDS` (30 s): cidade
  nova começa a processar em até 30 s; cidade suspensa, arquivada ou com schema atrasado para em até 30 s (os jobs dela
  esperam: `CitySchemaBehind` reagenda por até 1 hora).
- **Falha** — supervisor que morre é reiniciado com espera de 1 s, 2 s, 4 s… até 5 min; volta a 1 s depois de 10 min
  de pé. Log: `docker compose logs -f worker | grep city_workers`.
- **Parada** — TERM/INT repassa TERM aos supervisores, espera `SolidQueue.shutdown_timeout` + 5 s e mata o grupo de
  processo de quem sobrar.
- **Dimensionamento** — ~6 processos por cidade. `RAILS_MAX_THREADS` do worker precisa ser ≥ maior `threads` de
  `config/queue.yml` + 2 (Kamal: 12). Um host de worker roda todas as cidades; mais de um host duplica supervisores por
  cidade (seguro, mas dobra processos).
- **Painéis** — `/admin/api/queues` e `/admin/api/overview` leem a fila da cidade do host.
- `SOLID_QUEUE_IN_PUMA` não existe mais: o worker é sempre `bin/city_workers`.

```

- [ ] **Step 10: Commit**

```bash
git add README.md
git commit -m "Document the per-city worker runbook

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

Cole no relatório as saídas dos Steps 1–8, sem valores secretos.

---

## Definition of done

- A fila de cada cidade mora no banco dela e a de plataforma no banco de plataforma; o Solid Cache está no banco de
  plataforma; a migration de cidade e o dump coincidem (spec de paridade).
- `CityConnection.with` roteia domínio e fila. Um job de cidade fora de cidade, ou um de plataforma dentro de cidade,
  levanta `PlatformQueue::Misplaced`. Os painéis de fila leem a fila da cidade.
- `EachCityJob` roda só na cidade do worker (ou em todas fora de worker de cidade) e pula cidade atrasada.
  `CityScopedJob` reagenda cidade atrasada e recusa job de outra cidade.
- Recorrência dividida entre cidade e plataforma; filas enxutas, com `urgent` isolado.
- `bin/city_workers` roda um supervisor por cidade ativa, mais o da plataforma, com as seguintes garantias:
  - cidade nova entra em até 30 s e cidade suspensa sai em até 30 s;
  - supervisor que morre volta com backoff;
  - TERM encerra tudo sem órfãos;
  - cidade atrasada não sobe.
- O banco compartilhado não é mais usado: `primary` aponta para o banco vazio, sem `queue`, sem `DATABASE_URL` e sem
  `ROTA_ADMIN_PASSWORD`. `bin/jobs` e os dumps antigos saíram, e `start.sh` e compose foram ajustados. O banco de dev
  continua existindo e intocado.
- A prova em dev cobre: processos e recorrência por banco, job de cidade na fila certa, provisionamento pela fila de
  plataforma, `cascavel` assumida e depois offboardada, backoff, parada limpa, e o compartilhado inalterado.
- Suíte: 0 failures, **0 pending**, em duas seeds.

## O que este plano NÃO faz

- **Migração de jobs:** não migra jobs pendentes da fila compartilhada (decisão 4) e não apaga os bancos
  `rota_saude_development`/`rota_saude_test`.
- **Workers em vários hosts:** não reparte cidades entre hosts nem faz autoscaling. Um host roda todas as cidades, e mais
  hosts duplicam supervisores.
- **Operação:** não há painel de operação de fila da plataforma (Mission Control) nem métricas de processos.
- **Pendências do Plano 4** que continuam como tarefas separadas:
  - validação no PG 16 e drop determinístico;
  - reemissão do convite, umask do dump e `sslmode=verify-full`.
- **Plano 6:** frontends, CORS dinâmico, publicação de `admin.*`/`auth.*`/hosts de cidade no proxy do Kamal e chaves por
  cidade.
- **Deploy:** não há rollout gradual de supervisores. O deploy troca o container do worker inteiro; os jobs esperam na
  fila durante a troca.

## Riscos

1. **Memória do worker.** São ~6 processos Ruby por cidade (CoW ajuda, porque o fork é depois do boot). Os limites de
   4g em produção e 2g no deploy development cobrem poucas cidades e precisam ser medidos; crescer exige repartir
   cidades entre hosts, o que este plano não cobre.
2. **Conexões.** Cada cidade ativa soma dois pools por processo web (domínio e fila) e um pool por processo do Solid
   Queue dela, contra o `max_connections` do Postgres (risco aberto 2 da spec).
3. **Lista de jobs de plataforma.** Um job novo de plataforma esquecido em `PlatformQueue::JOBS` levanta no primeiro
   enqueue em produção. A falha é alta e não vaza dado, mas é falha.
4. **Janela de 30 s do catálogo.** Uma cidade suspensa ainda processa jobs por até 30 s. Jobs dela que começam depois
   disso levantam `CityNotServable` e ficam como falha.
5. **Upgrade do Solid Queue.** Uma mudança de tabelas precisa de migration de cidade e de plataforma e de
   `city:migrate:all` antes de subir o código novo. Com o worker novo e o banco velho, o supervisor da cidade falha e
   fica em backoff até migrar.
6. **Deploy do worker.** Parar o container para todas as cidades ao mesmo tempo; os jobs agendados atrasam alguns
   segundos, mas nada se perde porque a fila está no banco.
7. **Fork dentro de rspec** (`child_spec`). Depende do Rails descartar pools herdados depois do fork; se uma versão futura
   mudar isso, o spec fica instável antes do código de produção.
