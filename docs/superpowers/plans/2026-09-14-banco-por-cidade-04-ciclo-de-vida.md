# Banco por cidade — Plano 4: Ciclo de vida da cidade

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Dar à cidade um ciclo de vida completo. Ela é provisionada em duas fases (catálogo → worker cria banco e role,
migra, semeia e ativa) e o schema passa a ser aplicado em todas as cidades por uma task de deploy, com 503 só para a
cidade atrasada. A cidade também pode ser suspensa, ter backup e ser desligada, e grants e sessões vencidos passam a ser
purgados.

**Architecture:**
- **Schema:** o schema de cidade passa a ter versão: `db/city_migrate` define a versão esperada pelo código, e
  `cities.schema_version` guarda a versão aplicada em cada cidade.
- **Migração:** `city:migrate:all` aplica e registra essa versão, e o resolver compara as duas.
- **Provisionamento:** `POST /cities` no console só grava `provisioning` e enfileira `ProvisionCityJob`. No worker, o
  job conecta como `rota_provisioner` (CREATEDB/CREATEROLE, sem superusuário) e cria um role dono do banco da cidade,
  com `CONNECT` revogado de `PUBLIC`. Depois migra num subprocesso, semeia o banco da cidade e ativa a cidade.
- **Suspensão e offboarding:** são commands chamados por rake. O offboarding faz dump final, marca `archived` e roda
  `DROP DATABASE`/`DROP ROLE`.

**Tech Stack:** Rails 8.1.3, Ruby 3.3.6, PostgreSQL (dev 15.13; prod `postgres:16`), `pg` 1.6.3, Solid Queue 1.4.0,
RSpec, Docker Compose, Kamal 2.

**Spec:** `docs/superpowers/specs/2026-09-12-banco-por-cidade-design.md` — §4 ("Ciclo de vida da cidade") e os casos
nomeados de §Verification ("provisionamento idempotente; cidade em `provisioning` não é servida", "`city:migrate:all`
falha alto quando uma cidade quebra", "offboarding de A não altera nada em B", "schema atrasado → 503 só naquela
cidade").
**Planos anteriores:** 1, 2, 3 e 3B, todos mergeados em `main` do repo `apps/api` (Plano 3B = merge `8fe4021`).

## Decisões que este plano carrega

Tomadas na escrita, sem consulta ao usuário (a sessão pediu continuidade sem perguntas). **Revise antes de executar**:
cada uma diz o custo se estiver errada.

1. **Jobs de ciclo de vida rodam na fila compartilhada atual**, não numa fila de plataforma. A fila de plataforma (spec
   §4) nasce no Plano 5, junto com o gerente de supervisores. *Custo:* o payload (slug, nome, e-mails de servidor
   público) fica no banco compartilhado, que até o Plano 5 já guarda o payload de todo job.
2. **No provisionamento, a migração roda num subprocesso** (`bin/rails city:migrate[slug]`). O Rails 8.1 sempre migra
   pela conexão de `ActiveRecord::Base`, e o Solid Queue não tem `connects_to`: usa essa mesma conexão. Trocá-la dentro
   do worker desviaria as threads da fila. `city:migrate:all` roda em processo, porque é rake de uma thread. *Custo:*
   alguns segundos de boot por cidade provisionada.
3. **Um role por cidade, dono do banco dela, usado em runtime e nas migrations.**
   - O `CONNECT` de `PUBLIC` é revogado.
   - `rota_provisioner` tem LOGIN, CREATEDB e CREATEROLE, não é superusuário e vira membro de cada role de cidade.
   - Validado por sondagem em 2026-09-14 no PG 15.13: provisionar, migrar do zero como o role da cidade, recusar
     `rota_app`, `pg_dump`, `DROP DATABASE ... WITH (FORCE)` e `DROP ROLE`.
   - *Custo:* o runtime da cidade pode fazer DDL no próprio banco, nunca no de outra cidade.
4. **Nomes derivados do slug**, e o slug provisionável tem no máximo 40 caracteres (identificadores do Postgres têm 63
   bytes).
   - Fora de test: banco `rota_saude_city_<slug>`, role `rota_city_<slug>`.
   - Em test: banco `rota_saude_test_city_<slug>`, role `rota_test_city_<slug>`.
5. **Guarda 503 pelo catálogo.** `cities.schema_version` (a coluna já existe) é comparada com a maior versão em
   `db/city_migrate`; versão nula conta como atrasada. Quem grava a versão são `city:migrate`, `city:migrate:all`,
   `city:dev_up` e o provisionamento. *Custo:* uma migração aplicada à mão, fora das tasks, deixa a cidade em 503 até o
   próximo `city:migrate:all`.
6. **`city:migrate:all` migra cidades `active` e `suspended`.** `provisioning` fica com o job e `archived` não tem
   banco. Uma cidade que falha não impede as demais; no fim a task sai com status ≠ 0 listando as que ficaram para trás.
   O lock por cidade é o advisory lock que o Migrator do Rails já pega por banco.
7. **Provisionamento pelo console:** `POST /cities` em `admin.*` (operador verificado) → `202 { id }`, e
   `GET /cities/:id` → status. `POST /setup/municipalities` (501) e `ProvisionMunicipality` saem. O registro do canal
   WhatsApp, que era parte deles, vira `MunicipalityChannels::Register` + `channels:register`.
8. **O provisionamento NÃO semeia `consent_terms`** (desvio da spec §4).
   - O que impede: `Consents.current_version` devolve `ConsentTerm.maximum(:version)`, uma string (`"v1"`), ou `1`
     quando não há termo. O texto do termo vem das credentials por número (`policy.v<n>.text`).
   - Por isso, semear um termo mudaria o consentimento das cidades novas em relação às de dev, que não têm termo.
   - O provisionamento semeia `city_profile`, um destinatário de alerta por e-mail, o protocolo template em rascunho e
     o convite do primeiro `municipal_admin`.
   - *Custo:* o termo por cidade continua pendente, junto com a dívida de tipo da versão.
9. **O convite do primeiro admin vem da plataforma:** `invitations.invited_by_id` passa a aceitar nulo (migração
   expand-only).
   - O convite chega por e-mail (`InvitationMailer`), com link `CITY_DASHBOARD_URL_TEMPLATE` + `?invite=<token>`.
   - A tela que lê `invite` é do Plano 6; até lá, o aceite funciona pela API (`POST /setup/accept_invitation`).
10. **`city_profile` é uma tabela singleton** (nome, UF, IBGE, settings). O destino do alerta continua sendo o
    `AlertRecipient` (R37): `city_profile` não carrega destino, e nenhum leitor atual muda.
11. **Suspensão, retomada, backup e offboarding são commands com rake**, sem endpoint; o console só ganha tela no
    Plano 6.
    - O offboarding exige cidade `suspended` e segue esta ordem: dump final → canais inativos e grants apagados →
      `archived` → `DROP DATABASE`/`DROP ROLE`.
    - Rodar de novo numa cidade `archived` só repete o drop, que é idempotente.
    - `curitiba` e `maringa`, criadas por `city:dev_up` com banco do superusuário, não são apagáveis por
      `rota_provisioner`; está documentado.
12. **Purga diária:** grants vencidos há mais de 1 dia, sessões de operador pendentes além de `PENDING_MFA_WINDOW`,
    sessões verificadas além de `OPERATOR_SESSION_TTL` e, em cada cidade, sessões de operador por grant além de
    `OPERATOR_GRANT_TTL`.
13. **O entrypoint deixa de migrar.** `bin/migrate` roda `db:migrate` + `city:migrate:all` com a imagem nova, antes de
    subir o código.
14. **Dockerfile com `postgresql-client-16`** (repositório PGDG): o cliente 15 do bookworm não faz dump do
    `postgres:16` de produção.
15. **Dev:** `curitiba` e `maringa` continuam como estão (superusuário, `city:dev_up`).
    - O `schema_migrations` delas é completado só com INSERT, e as migrations novas chegam por `city:migrate:all`.
    - A prova do fluxo real provisiona e depois desliga uma terceira cidade, `londrina`.

## Global Constraints

- **Resolver a cidade ANTES de autenticar** (`CityResolution` → `Authentication`). A guarda 503 fica dentro do resolver,
  antes de abrir a conexão da cidade.
- **Cookie de sessão é host-only: NUNCA `domain:`** (spec §5).
- **O banco de plataforma nunca guarda dado de cidadão.** `PlatformEvent` recusa chave de payload que contenha `email`,
  `cpf`, `phone`, `wa_id`, `provider_uid`, `body` ou `name` (salvo `phone_number_id` e `city_name`) ou que seja
  exatamente `from`. Todo evento novo entra na lista estática de `spec/events/platform_event_payload_guard_spec.rb`.
- **`connects_to` só uma vez, no corpo de `CityRecord`.** Cidade entra por `CityConnection.with`.
- **`ActiveRecord::Tasks::DatabaseTasks.with_temporary_connection`** (usado por `CitySchema.migrate!`,
  `current_version`, `backfill_versions!` e `city:load_schema`) troca a conexão de `ActiveRecord::Base` durante o bloco.
  Por isso só roda em processo de uma thread (rake, subprocesso), nunca dentro de job ou request.
- **Senha e `database_url` nunca vão para log, mensagem de erro, argv de processo ou payload de evento.** Mensagens de
  erro de banco passam por `CitySchema.redact`. O `pg_dump` recebe a senha por `PGPASSWORD`, e o role recebe a senha já
  cifrada (`encrypt_password`), nunca em texto no SQL.
- **As migrations de cidade deste plano são aditivas (expand-only).** Toda migração destrutiva futura exige
  expand/contract (spec §4).
- **Nunca conceder CREATE em `public` a `rota_app`.**
- **Ponteiros de ADR em comentário só na faixa ADR-0001..ADR-0015.** Nada de constante top-level em spec.
- **Todo comando roda no container**, a partir da raiz do monorepo: `docker compose exec -T api <cmd>`. O rspec roda com
  `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Nunca `bundle` no host.
- **Sondagens só como spec de rascunho** (apagado depois). `rails runner` só para leitura.
- **Operações destrutivas permitidas neste plano: SÓ estas.**
  - `DROP DATABASE`/`DROP ROLE` de bancos e roles criados pelos próprios specs deste plano (prefixos
    `rota_saude_test_city_prov`, `rota_test_city_prov`, `rota_saude_test_scratch_`).
  - `DROP DATABASE`/`DROP ROLE` da cidade de prova `londrina` (`rota_saude_city_londrina`, `rota_city_londrina`), na
    Task 9.
  - Nada em `rota_saude_development`, `rota_saude_test`, `rota_saude_platform_*` ou `rota_saude_no_city_selected`.
  - Em `rota_saude_city_curitiba`/`maringa`, só INSERT em `schema_migrations`, as migrations aditivas deste plano e
    `db:seed`.
  - `city:test_databases` recarrega o schema nos bancos de cidade de TESTE; é o procedimento normal da suíte.
- **Spec que registra pool de uma cidade cujo banco depois apaga precisa chamar `CityConnection.forget(city.shard)`
  antes do fim do exemplo.** Um pool registrado para banco inexistente derruba todo exemplo transacional seguinte, porque
  `setup_transactional_fixtures` pina todo pool registrado.
- **Spec que cria banco de verdade usa `self.use_transactional_tests = false`** e apaga o que escreveu na plataforma
  (`City`, `CityChannel`, `CityGrant`, `PlatformEvent`).
- **Suíte verde ao fim de CADA task, 0 pending.** Nenhum spec enfraquecido; exemplo removido precisa de justificativa e
  do invariante provado em outro spec nomeado.
- **Commits em inglês**, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.

## Estado herdado (repo `apps/api`, `main` em `8fe4021`)

| Peça | Onde | Hoje |
|---|---|---|
| Catálogo | `app/models/city.rb`, `db/platform_schema.rb` | `STATUSES = provisioning active suspended archived`; coluna `cities.schema_version` (string) existe e ninguém grava |
| Resolução | `app/controllers/concerns/city_resolution.rb` | nil/`provisioning`/`archived` → 404 `unknown_city`; `suspended` → 403 `city_suspended`; sem 503 |
| Cidades de dev | `CityProvisioner`, `lib/tasks/city.rake` (`create`, `dev_up`, `dev_baseline`) | URL com o superusuário de bootstrap; `dev_up` carrega `db/city_schema.rb` quando `users` não existe |
| Carga do schema | `city.rake` → `load_city_schema` | `DatabaseTasks.load_schema` com a config resolvida da URL, **sem `migrations_paths`**: registra só a versão do dump |
| Migrations de cidade | `db/city_migrate/` | `20260913000010_create_city_schema`, `20260914000001_allow_operator_city_sessions`; nenhuma task aplica |
| `schema_migrations` de dev | `rota_saude_city_curitiba`, `rota_saude_city_maringa` | só `20260914000001` (falta `20260913000010`) |
| Provisionamento | `SetupController#provision_municipality` (501), `app/commands/provision_municipality.rb` | o command cria `CityChannel` + convite/termo/alerta/protocolo numa cidade JÁ ativa; não atômico |
| Entrypoint | `bin/docker-entrypoint` | o papel web roda `db:create` (best-effort) e `db:migrate` no boot |
| Solid Queue | `config/application.rb`, `bin/jobs`, `config/recurring.yml` | adapter `:solid_queue` **sem `connects_to`**: usa a conexão de `ActiveRecord::Base` (banco compartilhado) |
| Migração no Rails 8.1.3 | `activerecord/lib/active_record/migration.rb` | `Migrator`/`Migration` usam `DatabaseTasks.migration_connection(_pool)` = `ActiveRecord::Base` |
| Postgres | host / imagem | dev 15.13 (Homebrew); prod `postgres:16` (accessory); imagem com `postgresql-client` 15 (bookworm) |
| Roles | cluster | `rota_saude` (superusuário), `rota_app`, `rota_admin`, `rota_platform`; **não existe `rota_provisioner`** |
| Consentimento | `app/services/consents.rb` | `current_version = ConsentTerm.maximum(:version) \|\| 1`; texto em credentials; os seeds de dev não criam `ConsentTerm` |
| Convite | `db/city_schema.rb`, `InviteMember`, `AcceptInvitation` | `invitations.invited_by_id NOT NULL` (FK users); nenhum mailer de convite; `memberships.granted_by_id` já é nulável |
| Purga | — | nada purga `city_grants`, `operator_sessions` ou sessões de operador na cidade |

Suíte: **553 examples, 0 failures**. Dev: `curitiba` e `maringa` ativas, com seeds.

**Sondagens da escrita (2026-09-14, specs de rascunho já apagados):**
- Um role com CREATEDB/CREATEROLE, sem superusuário, conseguiu criar role de cidade, `GRANT` do role para si próprio
  (duas vezes, sem erro), `CREATE DATABASE ... OWNER`, `REVOKE ALL ... FROM PUBLIC`,
  `DROP DATABASE ... WITH (FORCE)` e `DROP ROLE`.
- `rota_app` foi recusado com `permission denied for database`, e o `pg_dump` como role da cidade funcionou.
- `CitySchema`-style: `HashConfig` com `migrations_paths: db/city_migrate` + `with_temporary_connection` +
  `pool.migration_context.migrate` migrou um banco vazio do zero (2 migrations, 20 tabelas) e devolveu
  `ActiveRecord::Base` ao banco original.
- O schema migrado do zero é **idêntico** ao carregado pelo dump em `rota_saude_test_city_b`: 143 colunas, 65 índices e
  33 constraints.

## File Structure

| Arquivo | Task | Responsabilidade |
|---|---|---|
| `app/services/city_schema.rb` | 1, 2 | versão esperada, config ad-hoc de cidade, `migrate!`, `current_version`, `backfill_versions!`, `redact`, `behind?` |
| `app/services/city_migrations.rb` | 1, 5 | `run(city)`, `run_all`, `Failed`; `Subprocess` (Task 5) |
| `lib/tasks/city.rake` | 1, 7 | `load_schema` com versões; `dev_up` migra e grava versão; `migrate`, `migrate:all`; `suspend`, `resume`, `backup`, `offboard` |
| `spec/support/scratch_databases.rb`, `spec/rails_helper.rb` | 1 | bancos descartáveis de spec |
| `spec/services/city_schema_spec.rb`, `spec/services/city_migrations_spec.rb`, `spec/tasks/city_rake_spec.rb` | 1, 2, 7 | migração do zero, paridade com o dump, falha alta |
| `app/controllers/concerns/city_resolution.rb`, `spec/requests/city_resolution_spec.rb` | 2 | 503 só para a cidade atrasada |
| `spec/factories/cities.rb`, `spec/support/city_request_auth.rb` | 2 | cidades de spec em dia com o schema |
| `bin/docker-entrypoint`, `bin/migrate`, `spec/architecture/boot_does_not_migrate_spec.rb` | 2 | boot não migra; passo de deploy |
| `db/city_migrate/20260915000001_create_city_profile.rb`, `db/city_migrate/20260915000002_allow_platform_invitations.rb`, `db/city_schema.rb` | 3 | `city_profile`; convite sem `invited_by` |
| `app/models/city_profile.rb`, `app/models/invitation.rb`, `db/seeds.rb`, `app/jobs/dispatch_municipality_alert_job.rb` | 3 | singleton; convite opcional; perfil nas cidades de dev |
| `spec/models/city_profile_spec.rb`, `spec/commands/accept_invitation_spec.rb` | 3 | singleton; aceite de convite da plataforma |
| `lib/tasks/platform.rake` | 4 | role `rota_provisioner` |
| `app/services/city_database.rb`, `spec/services/city_database_spec.rb`, `spec/support/provisioned_cities.rb` | 4 | banco e role por cidade, isolamento, limpeza |
| `app/commands/provision_city.rb`, `app/jobs/provision_city_job.rb` | 5 | fase 1 e fase 2 do provisionamento |
| `app/services/city_templates.rb`, `config/city_templates/triage_respiratoria.json` | 5 | protocolo template (também usado pelos seeds) |
| `app/mailers/invitation_mailer.rb`, `app/views/invitation_mailer/invite.text.erb`, `app/views/invitation_mailer/invite.html.erb`, `app/services/city_dashboard_url.rb` | 5 | e-mail do convite |
| `spec/commands/provision_city_spec.rb`, `spec/jobs/provision_city_job_spec.rb`, `spec/services/city_migrations_subprocess_spec.rb`, `spec/mailers/invitation_mailer_spec.rb` | 5 | idempotência, retomada, sem duplicar convite |
| `app/controllers/operators/cities_controller.rb`, `config/routes.rb`, `spec/requests/operators/cities_spec.rb` | 6 | console provisiona e consulta status |
| `app/controllers/setup_controller.rb`, `app/commands/provision_municipality.rb` (apagar), `spec/commands/provision_municipality_spec.rb` (apagar) | 6 | aposenta o endpoint 501 e o command antigo |
| `app/commands/municipality_channels/register.rb`, `lib/tasks/channels.rake`, `spec/commands/municipality_channels/register_spec.rb` | 6 | registro do canal WhatsApp |
| `app/commands/city_lifecycle/suspend.rb`, `resume.rb`, `backup.rb`, `offboard.rb` | 7 | ciclo de vida depois de ativa |
| `spec/commands/city_lifecycle/suspend_resume_spec.rb`, `backup_spec.rb`, `offboard_spec.rb` | 7 | backup restaurável sozinho; offboarding de A não toca B |
| `app/jobs/purge_platform_access_job.rb`, `app/jobs/purge_operator_city_sessions_job.rb`, `config/recurring.yml`, specs | 8 | purga |
| `spec/events/platform_event_payload_guard_spec.rb`, `app/events/platform.rb` | 6, 7 | eventos novos de plataforma |
| `Dockerfile`, `deploy/production/deploy.yml`, `deploy/development/deploy.yml`, `deploy/SECRETS.md`, `README.md`, `../../start.sh` (fora do git) | 9 | cliente 16, envs, runbook, prova em dev |

---

### Task 1: Versões de schema da cidade, `city:migrate` e `city:migrate:all`

**Files:**
- Create: `app/services/city_schema.rb`
- Create: `app/services/city_migrations.rb`
- Create: `spec/support/scratch_databases.rb`
- Create: `spec/services/city_schema_spec.rb`
- Create: `spec/services/city_migrations_spec.rb`
- Modify: `spec/rails_helper.rb` (require do suporte)
- Modify: `lib/tasks/city.rake` (`load_city_schema`, `dev_up`, tasks novas)
- Modify: `spec/tasks/city_rake_spec.rb` (bloco novo)

**Interfaces:**
- Consumes: `City#database_url`, `City#schema_version`, `CityDatabaseUrls.city_database_url(db, user:, password:)`
  (spec), `rota_saude_test_city_b` carregado pelo dump (`city:test_databases`).
- Produces:
  - `CitySchema.migrations_paths -> Array<String>`, `CitySchema.expected_version -> Integer`.
  - `CitySchema.db_config_for(url) -> ActiveRecord::DatabaseConfigurations::HashConfig`.
  - `CitySchema.migrate!(url) -> Integer`, `CitySchema.current_version(url) -> Integer`.
  - `CitySchema.backfill_versions!(url) -> Integer`, `CitySchema.redact(text) -> String`.
  - `CityMigrations::STATUSES`, `CityMigrations.run(city) -> Integer` (grava `schema_version`),
    `CityMigrations.run_all(out: $stdout)` (levanta `CityMigrations::Failed` com `#failures` = `{slug => mensagem}`).
  - Rake `city:migrate[slug]` e `city:migrate:all`.
  - Spec: `ScratchDatabases::PREFIX`, `.new_name`, `.url(name)`, `.create!(name)`, `.drop!(name)`, `.exists?(name)`,
    `.superuser(database = "postgres") { |conn| }`.

- [ ] **Step 1: Suporte de bancos descartáveis**

Criar `spec/support/scratch_databases.rb`:

```ruby
# Bancos descartáveis criados de verdade por specs que precisam de DDL comitado,
# de pg_dump ou de um banco que ainda não tem schema (Plano 4). Criados e apagados
# com a credencial de superusuário de bootstrap e SEMPRE com o prefixo PREFIX —
# nunca um banco da suíte, de dev ou de plataforma.
#
# Quem usa precisa de `self.use_transactional_tests = false` (DDL dentro da
# transação de fixture seria desfeito e invisível a outra conexão) e de apagar o
# banco no `after`.
module ScratchDatabases
  PREFIX = "rota_saude_test_scratch_"

  module_function

  def new_name
    "#{PREFIX}#{SecureRandom.hex(4)}"
  end

  def url(name)
    CityDatabaseUrls.city_database_url(name)
  end

  def create!(name)
    guard!(name)
    superuser { |conn| conn.exec("CREATE DATABASE #{PG::Connection.quote_ident(name)}") }
    name
  end

  def drop!(name)
    guard!(name)
    superuser { |conn| conn.exec("DROP DATABASE IF EXISTS #{PG::Connection.quote_ident(name)} WITH (FORCE)") }
  end

  def exists?(name)
    superuser { |conn| conn.exec_params("SELECT 1 FROM pg_database WHERE datname = $1", [ name ]).ntuples == 1 }
  end

  def superuser(database = "postgres")
    conn = PG.connect(CityDatabaseUrls.city_database_url(database))
    conn.set_notice_receiver { |_| }
    yield conn
  ensure
    conn&.close
  end

  def guard!(name)
    return if name.to_s.start_with?(PREFIX)

    raise ArgumentError, "scratch database must start with #{PREFIX}: #{name.inspect}"
  end
end
```

Em `spec/rails_helper.rb`, logo depois de `require_relative "support/city_database_urls"`, acrescentar:

```ruby
require_relative "support/scratch_databases"
```

- [ ] **Step 2: Specs de `CitySchema` (falham)**

Criar `spec/services/city_schema_spec.rb`:

```ruby
require "rails_helper"

# Schema dos bancos de cidade (spec banco-por-cidade §4, Plano 4). Os exemplos que
# migram usam um banco descartável DE VERDADE: DDL dentro da transação de fixture
# seria desfeito e invisível às outras conexões.
RSpec.describe CitySchema do
  self.use_transactional_tests = false

  let(:scratch) { ScratchDatabases.new_name }

  after { ScratchDatabases.drop!(scratch) }

  def versions_on_disk
    Dir[Rails.root.join("db/city_migrate/*.rb").to_s].map { |f| File.basename(f).to_i }.sort
  end

  def recorded_versions(database)
    ScratchDatabases.superuser(database) { |c| c.exec("SELECT version FROM schema_migrations ORDER BY 1").column_values(0).map(&:to_i) }
  end

  it "expects the highest version in db/city_migrate" do
    expect(described_class.expected_version).to eq(versions_on_disk.max)
  end

  it "migrates an empty database from zero to the expected version, and the second run is a no-op" do
    ScratchDatabases.create!(scratch)
    url = ScratchDatabases.url(scratch)

    expect(described_class.migrate!(url)).to eq(described_class.expected_version)
    expect(described_class.migrate!(url)).to eq(described_class.expected_version)
    expect(described_class.current_version(url)).to eq(described_class.expected_version)
    expect(recorded_versions(scratch)).to eq(versions_on_disk)
  end

  it "gives ActiveRecord::Base back its own database after migrating a city" do
    ScratchDatabases.create!(scratch)
    original = ActiveRecord::Base.connection_db_config.database

    described_class.migrate!(ScratchDatabases.url(scratch))

    expect(ActiveRecord::Base.connection_db_config.database).to eq(original)
  end

  # O dump db/city_schema.rb (city:test_databases, city:dev_up) e as migrations
  # (provisionamento, rollout) precisam produzir o MESMO schema.
  # rota_saude_test_city_b é carregado pelo dump.
  it "produces from the migrations exactly the schema of db/city_schema.rb" do
    ScratchDatabases.create!(scratch)
    described_class.migrate!(ScratchDatabases.url(scratch))

    expect(schema_fingerprint(scratch)).to eq(schema_fingerprint("rota_saude_test_city_b"))
  end

  it "backfills versions below the highest recorded one and never records a higher one" do
    ScratchDatabases.create!(scratch)
    url = ScratchDatabases.url(scratch)
    described_class.migrate!(url)
    lowest, highest = versions_on_disk.first, versions_on_disk.last

    ScratchDatabases.superuser(scratch) { |c| c.exec("DELETE FROM schema_migrations WHERE version <> '#{highest}'") }
    expect(described_class.backfill_versions!(url)).to eq(highest)
    expect(recorded_versions(scratch)).to eq(versions_on_disk)

    ScratchDatabases.superuser(scratch) { |c| c.exec("DELETE FROM schema_migrations WHERE version <> '#{lowest}'") }
    expect(described_class.backfill_versions!(url)).to eq(lowest)
    expect(recorded_versions(scratch)).to eq([ lowest ])
  end

  it "redacts the credentials of any database URL in a message" do
    text = "falhou em postgres://rota_city_x:s3gr3d0@db:5432/rota_saude_city_x e postgresql://u:p@h/d"

    expect(described_class.redact(text)).to eq("falhou em postgres://***@db:5432/rota_saude_city_x e postgresql://***@h/d")
  end

  def schema_fingerprint(database)
    ignored = "('probes', 'schema_migrations', 'ar_internal_metadata')"
    ScratchDatabases.superuser(database) do |conn|
      {
        columns: conn.exec(<<~SQL).values,
          SELECT table_name, column_name, data_type, is_nullable, column_default
          FROM information_schema.columns
          WHERE table_schema = 'public' AND table_name NOT IN #{ignored}
          ORDER BY 1, 2
        SQL
        indexes: conn.exec(<<~SQL).values,
          SELECT tablename, indexname, indexdef FROM pg_indexes
          WHERE schemaname = 'public' AND tablename NOT IN #{ignored}
          ORDER BY 1, 2
        SQL
        constraints: conn.exec(<<~SQL).values
          SELECT rel.relname, con.conname, pg_get_constraintdef(con.oid)
          FROM pg_constraint con
          JOIN pg_class rel ON rel.oid = con.conrelid
          JOIN pg_namespace ns ON ns.oid = rel.relnamespace
          WHERE ns.nspname = 'public' AND rel.relname NOT IN #{ignored}
          ORDER BY 1, 2
        SQL
      }
    end
  end
end
```

- [ ] **Step 3: Specs de `CityMigrations` (falham)**

Criar `spec/services/city_migrations_spec.rb`:

```ruby
require "rails_helper"

# city:migrate:all (spec banco-por-cidade §4): migra toda cidade active/suspended,
# registra a versão no catálogo e falha ALTO no fim se alguma ficar para trás —
# sem deixar uma cidade quebrada impedir as outras.
RSpec.describe CityMigrations do
  self.use_transactional_tests = false

  let(:scratch) { ScratchDatabases.new_name }
  let(:created_slugs) { [] }

  after do
    City.where(slug: created_slugs).delete_all
    ScratchDatabases.drop!(scratch)
  end

  def catalog_city(status:, database:)
    slug = "mig#{SecureRandom.hex(4)}"
    created_slugs << slug
    City.create!(slug: slug, name: "Migra", status: status, database_url: ScratchDatabases.url(database),
                 encryption_key: SecureRandom.hex(32))
  end

  it "migrates a city and records the version in the catalog" do
    ScratchDatabases.create!(scratch)
    city = catalog_city(status: "active", database: scratch)

    expect(described_class.run(city)).to eq(CitySchema.expected_version)
    expect(city.reload.schema_version).to eq(CitySchema.expected_version.to_s)
  end

  it "keeps going after a broken city and fails loud at the end, naming only the cities left behind" do
    ScratchDatabases.create!(scratch)
    healthy  = catalog_city(status: "suspended", database: scratch)
    broken   = catalog_city(status: "active", database: "#{ScratchDatabases::PREFIX}missing#{SecureRandom.hex(3)}")
    skipped  = catalog_city(status: "provisioning", database: "#{ScratchDatabases::PREFIX}missing#{SecureRandom.hex(3)}")
    archived = catalog_city(status: "archived", database: "#{ScratchDatabases::PREFIX}missing#{SecureRandom.hex(3)}")
    out = StringIO.new

    expect { described_class.run_all(out: out) }.to raise_error(CityMigrations::Failed) { |error|
      expect(error.failures.keys).to eq([ broken.slug ])
      expect(error.message).to include(broken.slug)
    }

    expect(healthy.reload.schema_version).to eq(CitySchema.expected_version.to_s)
    expect([ broken, skipped, archived ].map { |c| c.reload.schema_version }).to eq([ nil, nil, nil ])
    expect(out.string).to include("#{healthy.slug} → #{CitySchema.expected_version}").and include("#{broken.slug} FALHOU")
  end

  it "does not raise when every city migrates" do
    ScratchDatabases.create!(scratch)
    catalog_city(status: "active", database: scratch)

    expect { described_class.run_all(out: StringIO.new) }.not_to raise_error
  end
end
```

Acrescentar ao fim de `spec/tasks/city_rake_spec.rb`:

```ruby
# city:migrate e city:migrate:all (Plano 4). A migração de verdade é provada em
# spec/services/city_migrations_spec.rb; aqui só o contrato da task.
RSpec.describe "city:migrate and city:migrate:all rake tasks" do
  before(:all) do
    Rails.application.load_tasks unless Rake::Task.task_defined?("city:migrate:all")
  end

  before do
    %w[city:migrate city:migrate:all].each { |name| Rake::Task[name].reenable }
  end

  def invoke_silently(name, *args)
    original_stdout, $stdout = $stdout, StringIO.new
    original_stderr, $stderr = $stderr, StringIO.new
    Rake::Task[name].invoke(*args)
  ensure
    $stdout = original_stdout
    $stderr = original_stderr
  end

  it "city:migrate aborts for an unknown slug" do
    expect { invoke_silently("city:migrate", "naoexiste#{SecureRandom.hex(3)}") }.to raise_error(SystemExit)
  end

  it "city:migrate refuses an archived city without touching any database" do
    city = create(:city, status: "archived")
    expect(CityMigrations).not_to receive(:run)

    expect { invoke_silently("city:migrate", city.slug) }.to raise_error(SystemExit)
  end

  it "city:migrate migrates the city through CityMigrations" do
    city = create(:city, status: "provisioning")
    expect(CityMigrations).to receive(:run).with(city).and_return(CitySchema.expected_version)

    expect { invoke_silently("city:migrate", city.slug) }.not_to raise_error
  end

  it "city:migrate exits non-zero when the migration raises" do
    city = create(:city, status: "active")
    allow(CityMigrations).to receive(:run).and_raise(ActiveRecord::NoDatabaseError)

    expect { invoke_silently("city:migrate", city.slug) }.to raise_error(SystemExit) { |e| expect(e.status).not_to eq(0) }
  end

  it "city:migrate:all exits non-zero when a city is left behind" do
    allow(CityMigrations).to receive(:run_all).and_raise(CityMigrations::Failed.new("quebrada" => "PG::Error: boom"))

    expect { invoke_silently("city:migrate:all") }.to raise_error(SystemExit) { |e| expect(e.status).not_to eq(0) }
  end
end
```

- [ ] **Step 4: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_schema_spec.rb spec/services/city_migrations_spec.rb spec/tasks/city_rake_spec.rb`
Expected: FAIL com `NameError: uninitialized constant CitySchema` / `CityMigrations` e task `city:migrate` inexistente.

- [ ] **Step 5: `CitySchema`**

Criar `app/services/city_schema.rb`:

```ruby
# Schema dos bancos de CIDADE: onde moram as migrations, qual versão este código
# espera e como aplicar (spec banco-por-cidade §4, Plano 4).
#
# As migrations de cidade ficam em db/city_migrate, fora de toda config de
# config/database.yml: nenhum db:migrate genérico as alcança.
#
# ATENÇÃO — migrate!, current_version e backfill_versions! usam
# DatabaseTasks.with_temporary_connection, que troca a conexão de
# ActiveRecord::Base durante o bloco (o Rails 8.1 migra sempre por ela). O Solid
# Queue usa essa mesma conexão. Só chame em processo de uma thread (rake,
# subprocesso) — nunca dentro de job ou request. O provisionamento migra por
# subprocesso (CityMigrations::Subprocess).
module CitySchema
  MIGRATIONS_PATH = "db/city_migrate"

  class << self
    def migrations_paths
      [ Rails.root.join(MIGRATIONS_PATH).to_s ]
    end

    # Maior versão presente em db/city_migrate: o schema que ESTE código espera.
    def expected_version
      @expected_version ||= Dir[File.join(migrations_paths.first, "*.rb")].map { |f| File.basename(f).to_i }.max.to_i
    end

    # Config ad-hoc de um banco de cidade com as migrations de cidade. Nunca é
    # registrada em database.yml.
    def db_config_for(url)
      resolved = ActiveRecord::Base.configurations.resolve(url.to_s)
      ActiveRecord::DatabaseConfigurations::HashConfig.new(
        Rails.env, "city", resolved.configuration_hash.merge(migrations_paths: migrations_paths)
      )
    end

    # Aplica as migrations pendentes e devolve a versão resultante. O Migrator do
    # Rails pega um advisory lock por banco: duas execuções na mesma cidade não
    # correm juntas (a segunda levanta ActiveRecord::ConcurrentMigrationError).
    def migrate!(url)
      with_city_connection(url) do |conn|
        conn.pool.migration_context.migrate
        conn.pool.migration_context.current_version
      end
    end

    def current_version(url)
      with_city_connection(url) { |conn| conn.pool.migration_context.current_version }
    end

    # Completa schema_migrations ABAIXO da maior versão já registrada, sem nunca
    # declarar aplicada uma versão maior. Conserta bancos carregados pelo dump
    # antes de load_schema conhecer db/city_migrate — só a versão do dump ficava
    # registrada, e city:migrate tentaria recriar o schema inteiro.
    def backfill_versions!(url)
      with_city_connection(url) do |conn|
        version = conn.pool.migration_context.current_version
        conn.assume_migrated_upto_version(version) if version.positive?
        conn.pool.migration_context.current_version
      end
    end

    # Tira usuário e senha de toda URL de banco num texto (mensagem de erro, saída
    # de subprocesso) antes de ir para log ou exceção.
    def redact(text)
      text.to_s.gsub(%r{://[^/\s@]+@}, "://***@")
    end

    private

    def with_city_connection(url, &block)
      verbose_was = ActiveRecord::Migration.verbose
      ActiveRecord::Migration.verbose = false
      ActiveRecord::Tasks::DatabaseTasks.with_temporary_connection(db_config_for(url), &block)
    ensure
      ActiveRecord::Migration.verbose = verbose_was
    end
  end
end
```

- [ ] **Step 6: `CityMigrations`**

Criar `app/services/city_migrations.rb`:

```ruby
# Aplica o schema de cidade nas cidades do catálogo e registra a versão em
# cities.schema_version, que a guarda de runtime compara com a versão esperada
# pelo código (spec banco-por-cidade §4, Plano 4).
#
# Roda em processo de uma thread (rake city:migrate / city:migrate:all, ou o
# subprocesso do provisionamento) — ver CitySchema.
module CityMigrations
  # Estados com banco a migrar. provisioning é migrado pelo próprio job de
  # provisionamento; archived não tem mais banco.
  STATUSES = %w[active suspended].freeze

  class Failed < StandardError
    attr_reader :failures

    def initialize(failures)
      @failures = failures
      super("cidades não migradas: #{failures.map { |slug, message| "#{slug} (#{message})" }.join('; ')}")
    end
  end

  module_function

  def run(city)
    version = CitySchema.migrate!(city.database_url)
    city.update!(schema_version: version.to_s)
    version
  end

  # Migra cada cidade; uma falha não impede as demais. No fim levanta Failed com o
  # slug e a mensagem (sem credencial) de cada cidade que ficou para trás.
  def run_all(out: $stdout)
    failures = {}

    City.where(status: STATUSES).order(:slug).each do |city|
      out.puts "[city:migrate:all] #{city.slug} → #{run(city)}"
    rescue StandardError => e
      failures[city.slug] = "#{e.class}: #{CitySchema.redact(e.message)}"
      out.puts "[city:migrate:all] #{city.slug} FALHOU — #{failures[city.slug]}"
    end

    raise Failed, failures if failures.any?
  end
end
```

- [ ] **Step 7: Tasks de rake**

Em `lib/tasks/city.rake`, dentro de `load_city_schema`, trocar:

```ruby
    url = database_name_or_url.to_s.include?("://") ? database_name_or_url : city_database_url.call(database_name_or_url)
    db_config = ActiveRecord::Base.configurations.resolve(url)
```

por:

```ruby
    url = database_name_or_url.to_s.include?("://") ? database_name_or_url : city_database_url.call(database_name_or_url)
    # Com migrations_paths de cidade: o `define(version:)` do dump registra em
    # schema_migrations TODAS as versões de db/city_migrate até a do dump, não só
    # a última (sem isso, city:migrate tentaria recriar o schema).
    db_config = CitySchema.db_config_for(url)
```

Ainda em `city:dev_up`, trocar o bloco que vai de `if empty.strip == "t"` até `puts "[city:dev_up] #{city.slug} → #{city.status} (#{database})"` por:

```ruby
    if empty.strip == "t"
      load_city_schema.call(database)
      puts "[city:dev_up] schema de cidade carregado em #{database}"
    else
      # Banco carregado antes de load_schema registrar todas as versões: completa
      # as anteriores à maior registrada (só INSERT) antes de migrar.
      CitySchema.backfill_versions!(city.database_url)
      puts "[city:dev_up] #{database} já tem o schema de cidade — versões anteriores registradas"
    end

    version = CitySchema.migrate!(city.database_url)
    city.update!(status: "active", schema_version: version.to_s)
    CityCatalog.reset_cache!
    puts "[city:dev_up] #{city.slug} → #{city.status} (#{database}, schema #{version})"
```

Acrescentar, antes do `end` final de `namespace :city`:

```ruby
  desc "Aplica as migrations de cidade numa cidade do catálogo e registra a versão. Uso: city:migrate[slug]"
  task :migrate, %i[slug] => :environment do |_t, args|
    abort "uso: rails 'city:migrate[slug]'" if args[:slug].blank?
    city = City.find_by(slug: args[:slug])
    abort "[city:migrate] cidade #{args[:slug]} não existe" unless city
    abort "[city:migrate] cidade #{city.slug} está archived — não tem banco" if city.status == "archived"

    begin
      version = CityMigrations.run(city)
    rescue StandardError => e
      abort "[city:migrate] #{city.slug} falhou — #{e.class}: #{CitySchema.redact(e.message)}"
    end
    puts "[city:migrate] #{city.slug} → #{version}"
  end

  namespace :migrate do
    desc "Aplica as migrations de cidade em toda cidade active/suspended; sai com erro listando as que ficarem para trás."
    task all: :environment do
      CityMigrations.run_all
    rescue CityMigrations::Failed => e
      abort "[city:migrate:all] #{e.message}"
    end
  end
```

- [ ] **Step 8: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_schema_spec.rb spec/services/city_migrations_spec.rb spec/tasks/city_rake_spec.rb`
Expected: PASS. Se o exemplo de paridade falhar, a diferença impressa diz qual lado está errado. O dump é o que a suíte
e o dev carregam; corrija a migration ou o dump até os dois coincidirem, nunca afrouxando o fingerprint.

- [ ] **Step 9: Bancos de teste e de dev com versões completas**

Run: `docker compose exec -T -e RAILS_ENV=test api ./bin/rails city:test_databases`
Expected: `schema de cidade carregado` nos dois bancos de teste.

Run: `docker compose exec -T api ./bin/rails city:dev_baseline`
Expected: para `curitiba` e `maringa`, `já tem o schema de cidade — versões anteriores registradas` e
`→ active (..., schema 20260914000001)`. Só INSERT em `schema_migrations` e UPDATE no catálogo.

Run (leitura): `docker compose exec -T api bash -c 'for db in rota_saude_city_curitiba rota_saude_city_maringa; do PGPASSWORD=$POSTGRES_PASSWORD psql -h $DATABASE_HOST -U rota_saude -d $db -tA -c "select string_agg(version, \",\" order by version) from schema_migrations"; done; PGPASSWORD=$POSTGRES_PASSWORD psql -h $DATABASE_HOST -U rota_saude -d rota_saude_platform_development -tA -c "select slug, schema_version from cities order by slug"'`
Expected: `20260913000010,20260914000001` duas vezes; `curitiba|20260914000001` e `maringa|20260914000001`.

Run: `docker compose exec -T api ./bin/rails city:migrate:all`
Expected: `curitiba → 20260914000001`, `maringa → 20260914000001`, saída 0.

- [ ] **Step 10: Suíte inteira**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

- [ ] **Step 11: Commit**

```bash
git add app/services/city_schema.rb app/services/city_migrations.rb lib/tasks/city.rake \
  spec/support/scratch_databases.rb spec/rails_helper.rb spec/services/city_schema_spec.rb \
  spec/services/city_migrations_spec.rb spec/tasks/city_rake_spec.rb
git commit -m "Version city schemas and roll migrations out with city:migrate:all

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: Guarda 503 para a cidade atrasada e boot sem migração

**Files:**
- Modify: `app/services/city_schema.rb` (`behind?`)
- Modify: `app/controllers/concerns/city_resolution.rb`
- Modify: `spec/factories/cities.rb`
- Modify: `spec/support/city_request_auth.rb`
- Modify: `spec/requests/city_resolution_spec.rb`
- Modify: `spec/services/city_schema_spec.rb`
- Modify: `bin/docker-entrypoint`
- Create: `bin/migrate`
- Create: `spec/architecture/boot_does_not_migrate_spec.rb`

**Interfaces:**
- Consumes: `CitySchema.expected_version` (Task 1), `City#schema_version`.
- Produces:
  - `CitySchema.behind?(city) -> Boolean`.
  - Resposta `503 { "error": "city_schema_behind" }`.
  - A factory `:city` passa a nascer com `schema_version` em dia, e `use_test_city_host!` também.
  - `bin/migrate`.

- [ ] **Step 1: Specs (falham)**

Em `spec/services/city_schema_spec.rb`, acrescentar antes do `def schema_fingerprint`:

```ruby
  describe ".behind?" do
    it "is true when the catalog records a lower version, or none, and false when it is current" do
      expected = described_class.expected_version

      expect(described_class.behind?(City.new(schema_version: (expected - 1).to_s))).to be(true)
      expect(described_class.behind?(City.new(schema_version: nil))).to be(true)
      expect(described_class.behind?(City.new(schema_version: expected.to_s))).to be(false)
    end
  end
```

Em `spec/requests/city_resolution_spec.rb`, acrescentar antes do `end` final:

```ruby
  it "returns 503 only for the city whose schema is behind" do
    create(:city, slug: "atrasada", status: "active", database_url: city_a_url,
                  schema_version: (CitySchema.expected_version - 1).to_s)
    create(:city, slug: "emdia", status: "active", database_url: city_a_url)

    get "/_probe", headers: { "HOST" => "atrasada.rotasaude.app" }
    expect(response).to have_http_status(:service_unavailable)
    expect(JSON.parse(response.body)["error"]).to eq("city_schema_behind")

    get "/_probe", headers: { "HOST" => "emdia.rotasaude.app" }
    expect(response).to have_http_status(:ok)
  end

  it "treats an active city with no recorded schema version as behind" do
    create(:city, slug: "semversao", status: "active", database_url: city_a_url, schema_version: nil)

    get "/_probe", headers: { "HOST" => "semversao.rotasaude.app" }
    expect(response).to have_http_status(:service_unavailable)
  end

  it "still answers 403 for a suspended city that is also behind" do
    create(:city, slug: "suspatrasada", status: "suspended", database_url: city_a_url, schema_version: nil)

    get "/_probe", headers: { "HOST" => "suspatrasada.rotasaude.app" }
    expect(response).to have_http_status(:forbidden)
  end
```

Criar `spec/architecture/boot_does_not_migrate_spec.rb`:

```ruby
require "rails_helper"

# Spec banco-por-cidade §4: com um banco por cidade, réplicas migrando no boot
# correriam entre si e o boot cresceria com o número de cidades. Migração é passo
# explícito de deploy (bin/migrate).
RSpec.describe "Boot does not migrate" do
  def code_of(path)
    File.readlines(Rails.root.join(path)).reject { |line| line.strip.start_with?("#") }.join
  end

  it "keeps every migration task out of bin/docker-entrypoint" do
    expect(code_of("bin/docker-entrypoint")).not_to match(/db:(create|migrate|prepare|schema:load)|city:/)
  end

  it "migrates the platform databases and then every city in bin/migrate" do
    code = code_of("bin/migrate")

    expect(code.index("db:migrate")).to be < code.index("city:migrate:all")
    expect(File.executable?(Rails.root.join("bin/migrate"))).to be(true)
  end
end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_schema_spec.rb spec/requests/city_resolution_spec.rb spec/architecture/boot_does_not_migrate_spec.rb`
Expected: FAIL. Os motivos são `NoMethodError: behind?`, `200` onde se espera `503` e `bin/migrate` inexistente.

- [ ] **Step 3: `behind?` e a guarda no resolver**

Em `app/services/city_schema.rb`, acrescentar depois de `expected_version`:

```ruby
    # Cidade cujo schema registrado no catálogo é menor que o esperado por este
    # código. Versão nula (nunca migrada pelas tasks) conta como atrasada. Não abre
    # conexão: compara só o catálogo.
    def behind?(city)
      city.schema_version.to_i < expected_version
    end
```

Em `app/controllers/concerns/city_resolution.rb`, trocar:

```ruby
    return render(json: { error: "unknown_city" }, status: :not_found) unless city.servable?
```

por:

```ruby
    return render(json: { error: "unknown_city" }, status: :not_found) unless city.servable?
    # Deploy não é atômico (spec §4): código novo pode encontrar uma cidade que
    # city:migrate:all ainda não alcançou. Só ESSA cidade fica fora do ar.
    return render(json: { error: "city_schema_behind" }, status: :service_unavailable) if CitySchema.behind?(city)
```

- [ ] **Step 4: Cidades de spec em dia com o schema**

Em `spec/factories/cities.rb`, depois de `encryption_key { SecureRandom.hex(32) }`:

```ruby
    schema_version { CitySchema.expected_version.to_s }
```

Em `spec/support/city_request_auth.rb`, trocar:

```ruby
      City.create!(slug: TEST_CITY_A.slug, name: TEST_CITY_A.name, status: "active",
                   database_url: TEST_CITY_A.database_url, encryption_key: TEST_CITY_A.encryption_key)
```

por:

```ruby
      City.create!(slug: TEST_CITY_A.slug, name: TEST_CITY_A.name, status: "active",
                   database_url: TEST_CITY_A.database_url, encryption_key: TEST_CITY_A.encryption_key,
                   schema_version: CitySchema.expected_version.to_s)
```

- [ ] **Step 5: Entrypoint sem migração e `bin/migrate`**

Substituir o conteúdo de `bin/docker-entrypoint` por:

```bash
#!/bin/bash -e
# Entrypoint da imagem única (ADR-0002). NÃO migra (spec banco-por-cidade §4):
# com um banco por cidade, réplicas web migrariam em corrida entre si e o boot
# cresceria com o número de cidades. Migração é passo explícito de deploy, com a
# imagem nova e ANTES de trocar o código em execução: bin/migrate.
#
# O bootstrap do zero (databases, roles, extensions, schema de fila/cache) é do
# start.sh no dev e da infra no prod.

exec "${@}"
```

Criar `bin/migrate`:

```bash
#!/bin/bash -e
# Passo de deploy (spec banco-por-cidade §4), rodado com a imagem NOVA antes de
# trocar o código em execução:
#   1. db:migrate — primary/queue/cache/platform;
#   2. city:migrate:all — toda cidade active/suspended, com lock por cidade; sai
#      com status != 0 listando as cidades que ficaram para trás.
# O código antigo segue no ar enquanto isto roda: migração destrutiva exige
# expand/contract. Cidade que falhar responde 503 até ser migrada.
./bin/rails db:migrate
./bin/rails city:migrate:all
```

Run: `chmod +x bin/migrate` (no host, dentro de `apps/api`; é só modo de arquivo).

- [ ] **Step 6: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_schema_spec.rb spec/requests/city_resolution_spec.rb spec/architecture/boot_does_not_migrate_spec.rb`
Expected: PASS.

- [ ] **Step 7: Suíte inteira**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending. Um request spec que responder `503` cria cidade ativa sem passar pela factory nem por
`use_test_city_host!`. Nesse caso, grave `schema_version: CitySchema.expected_version.to_s` na criação; não relaxe a
guarda.

- [ ] **Step 8: Prova em dev**

Run: `docker compose restart api`
Run: `curl -s -o /dev/null -w "%{http_code}\n" -H "Host: curitiba.localhost" http://localhost:3030/admin/api/overview`
Expected: `401` (servida e em dia: o schema_version foi gravado na Task 1).

- [ ] **Step 9: Commit**

```bash
git add app/services/city_schema.rb app/controllers/concerns/city_resolution.rb spec/factories/cities.rb \
  spec/support/city_request_auth.rb spec/requests/city_resolution_spec.rb spec/services/city_schema_spec.rb \
  bin/docker-entrypoint bin/migrate spec/architecture/boot_does_not_migrate_spec.rb
git commit -m "Answer 503 only for a city behind the schema and stop migrating on boot

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: `city_profile` e convite da plataforma — primeiras migrations entregues por `city:migrate:all`

**Files:**
- Create: `db/city_migrate/20260915000001_create_city_profile.rb`
- Create: `db/city_migrate/20260915000002_allow_platform_invitations.rb`
- Create: `app/models/city_profile.rb`
- Create: `spec/models/city_profile_spec.rb`
- Modify: `db/city_schema.rb`
- Modify: `app/models/invitation.rb`
- Modify: `spec/commands/accept_invitation_spec.rb`
- Modify: `db/seeds.rb`
- Modify: `app/jobs/dispatch_municipality_alert_job.rb` (comentário)

**Interfaces:**
- Consumes: `CitySchema` e `city:migrate:all` (Task 1); a guarda 503 (Task 2).
- Produces:
  - Tabela `city_profile`, com singleton, `name`, `uf`, `ibge_code` e `settings`.
  - `CityProfile.current -> CityProfile | nil`.
  - `Invitation` com `invited_by` opcional.
  - `CitySchema.expected_version == 20260915000002`.

- [ ] **Step 1: Specs (falham)**

Criar `spec/models/city_profile_spec.rb`:

```ruby
require "rails_helper"

# Identidade da cidade no banco dela (spec banco-por-cidade §3, Plano 4): um dump
# restaurado sozinho continua sabendo de que cidade é.
RSpec.describe CityProfile do
  it "holds at most one row per city database" do
    described_class.create!(name: "Cidade A", uf: "PR", ibge_code: "4106902")

    expect { described_class.create!(name: "Outra") }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "refuses a row that is not the singleton" do
    expect { described_class.create!(name: "Cidade A", singleton: false) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_city_profile_singleton/)
  end

  it "returns the row as .current, and nil before provisioning" do
    expect(described_class.current).to be_nil

    profile = described_class.create!(name: "Cidade A", uf: "PR", ibge_code: "4106902")

    expect(described_class.current).to eq(profile)
  end

  it "validates name, UF and IBGE code" do
    expect(described_class.new(name: "", uf: "PR")).not_to be_valid
    expect(described_class.new(name: "X", uf: "pr")).not_to be_valid
    expect(described_class.new(name: "X", ibge_code: "123")).not_to be_valid
    expect(described_class.new(name: "X", uf: "PR", ibge_code: "4106902")).to be_valid
  end
end
```

Em `spec/commands/accept_invitation_spec.rb`, acrescentar antes do `end` final:

```ruby
  # Provisionamento (Plano 4): o convite do primeiro municipal_admin vem da
  # plataforma — ainda não existe usuário na cidade para ser invited_by.
  it "aceita convite da plataforma (sem invited_by) e grava a membership sem granted_by" do
    Invitation.create!(email: "primeira@example.org", role: "municipal_admin", token: "plataforma-1",
                       invited_by: nil, expires_at: 1.day.from_now)

    res = described_class.call(token: "plataforma-1", password: "secretpw")

    expect(res.ok?).to be true
    expect(Membership.find_by!(user: res.payload[:user], role: "municipal_admin").granted_by_id).to be_nil
  end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_profile_spec.rb spec/commands/accept_invitation_spec.rb`
Expected: FAIL com `uninitialized constant CityProfile` e `Validation failed: Invited by must exist`, ou
`PG::NotNullViolation` em `invited_by_id`.

- [ ] **Step 3: Migrations**

Criar `db/city_migrate/20260915000001_create_city_profile.rb`:

```ruby
# Identidade da cidade DENTRO do banco dela (spec banco-por-cidade §3): nome, UF,
# código IBGE e settings, gravados no provisionamento (Plano 4). Um dump
# restaurado sozinho continua sabendo de que cidade é, sem ler o catálogo.
#
# Singleton: a coluna `singleton` só aceita true e é única — no máximo uma linha.
# Migração aditiva (expand-only).
class CreateCityProfile < ActiveRecord::Migration[8.1]
  def change
    create_table :city_profile, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.boolean :singleton, null: false, default: true
      t.string :name, null: false
      t.string :uf, limit: 2
      t.string :ibge_code, limit: 7
      t.jsonb :settings, null: false, default: {}
      t.timestamps
    end
    add_index :city_profile, :singleton, unique: true, name: "index_city_profile_singleton"
    add_check_constraint :city_profile, "singleton", name: "ck_city_profile_singleton"
  end
end
```

Criar `db/city_migrate/20260915000002_allow_platform_invitations.rb`:

```ruby
# O convite do primeiro municipal_admin de uma cidade nova vem da plataforma
# (provisionamento, Plano 4): ainda não existe usuário na cidade para ser
# invited_by. A FK para users continua; só a obrigatoriedade sai. Aditiva.
class AllowPlatformInvitations < ActiveRecord::Migration[8.1]
  def change
    change_column_null :invitations, :invited_by_id, true
  end
end
```

- [ ] **Step 4: Dump `db/city_schema.rb`**

Trocar `ActiveRecord::Schema[8.1].define(version: 2026_09_14_000001) do` por
`ActiveRecord::Schema[8.1].define(version: 2026_09_15_000002) do`.

Imediatamente antes de `  create_table "consent_terms", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|`,
inserir (seguido de uma linha em branco):

```ruby
  create_table "city_profile", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "created_at", null: false
    t.string "ibge_code", limit: 7
    t.string "name", null: false
    t.jsonb "settings", default: {}, null: false
    t.boolean "singleton", default: true, null: false
    t.string "uf", limit: 2
    t.datetime "updated_at", null: false
    t.index ["singleton"], name: "index_city_profile_singleton", unique: true
    t.check_constraint "singleton", name: "ck_city_profile_singleton"
  end
```

No bloco `create_table "invitations"`, trocar `    t.uuid "invited_by_id", null: false` por `    t.uuid "invited_by_id"`.

- [ ] **Step 5: Modelos**

Criar `app/models/city_profile.rb`:

```ruby
# Identidade da cidade no banco dela — linha única (ver CreateCityProfile).
# Escrita pelo provisionamento (Plano 4) e pelos seeds de dev. O destino do alerta
# NÃO mora aqui: continua sendo o AlertRecipient (R37).
class CityProfile < ApplicationRecord
  self.table_name = "city_profile"

  validates :name, presence: true
  validates :uf, format: { with: /\A[A-Z]{2}\z/ }, allow_nil: true
  validates :ibge_code, format: { with: /\A\d{7}\z/ }, allow_nil: true

  def self.current = first
end
```

Em `app/models/invitation.rb`, trocar `  belongs_to :invited_by, class_name: "User"` por:

```ruby
  # Nulo quando o convite vem da plataforma: o primeiro municipal_admin de uma
  # cidade recém-provisionada (Plano 4).
  belongs_to :invited_by, class_name: "User", optional: true
```

- [ ] **Step 6: Carregar o dump nos bancos de teste e rodar os specs**

Run: `docker compose exec -T -e RAILS_ENV=test api ./bin/rails city:test_databases`
Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_profile_spec.rb spec/commands/accept_invitation_spec.rb spec/services/city_schema_spec.rb`
Expected: PASS. Em `city_schema_spec`, o exemplo de paridade prova que o dump editado à mão bate com as migrations.

- [ ] **Step 7: Seeds e comentário do R37**

Em `db/seeds.rb`:

Trocar:

```ruby
#   - CADA CIDADE (curitiba, maringa), dentro da conexão dela: admin@<slug>.demo /
#     dev-password como municipal_admin, um AlertRecipient de e-mail ativo,
```

por:

```ruby
#   - CADA CIDADE (curitiba, maringa), dentro da conexão dela: admin@<slug>.demo /
#     dev-password como municipal_admin, o city_profile, um AlertRecipient de e-mail ativo,
```

Trocar `  { "curitiba" => "41", "maringa" => "44" }.each do |slug, ddd|` por:

```ruby
  { "curitiba" => %w[41 4106902], "maringa" => %w[44 4115200] }.each do |slug, (ddd, ibge_code)|
```

Trocar:

```ruby
        # ── Destinatário de alerta urgente (R37) ──────────────────────────────
        # DispatchMunicipalityAlertJob entrega ao primeiro AlertRecipient de email
        # ativo da cidade e levanta NoAlertRecipient sem nenhum. Um por cidade, no
        # banco dela; o city_profile (Plano 4) substitui.
```

por:

```ruby
        # ── Identidade da cidade no banco dela (city_profile, Plano 4) ────────
        profile = CityProfile.current || CityProfile.new
        profile.update!(name: city.name, uf: city.uf, ibge_code: ibge_code)

        # ── Destinatário de alerta urgente (R37) ──────────────────────────────
        # DispatchMunicipalityAlertJob entrega ao primeiro AlertRecipient de email
        # ativo da cidade e levanta NoAlertRecipient sem nenhum. Um por cidade, no
        # banco dela. city_profile não carrega destino de alerta (Plano 4).
```

Trocar `        puts "  municipal ... #{muni_admin.email_address} / #{password}  → dashboard"` por:

```ruby
        puts "  perfil ...... #{profile.name}/#{profile.uf} IBGE #{profile.ibge_code}"
        puts "  municipal ... #{muni_admin.email_address} / #{password}  → dashboard"
```

Em `app/jobs/dispatch_municipality_alert_job.rb`, trocar:

```ruby
# (alert_email/alert_webhook); a tabela municipalities não existe no mundo por
# cidade e o city_profile da spec §3 ainda não existe no schema de cidade — o
# provisionamento (Plano 4) é quem o cria. Sem destinatário, levanta: o alerta
# falha visível em vez de sumir.
```

por:

```ruby
# (alert_email/alert_webhook); a tabela municipalities não existe no mundo por
# cidade, e o city_profile (spec §3, Plano 4) guarda a identidade da cidade, não
# o destino do alerta. Sem destinatário, levanta: o alerta falha visível em vez
# de sumir.
```

- [ ] **Step 8: Suíte inteira**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

- [ ] **Step 9: Rollout em dev pelo caminho de deploy**

Run: `docker compose exec -T api ./bin/rails city:migrate:all`
Expected: `curitiba → 20260915000002`, `maringa → 20260915000002`, saída 0 (migrations aditivas sobre os dados de
exemplo).

Run: `docker compose restart api worker` (a versão esperada é memoizada por processo).
Run: `docker compose exec -T api ./bin/rails db:seed`
Expected: `perfil ...... Curitiba/PR IBGE 4106902` e `perfil ...... Maringá/PR IBGE 4115200`.

Run: `curl -s -o /dev/null -w "%{http_code}\n" -H "Host: curitiba.localhost" http://localhost:3030/admin/api/overview`
Expected: `401` (servida e em dia).

- [ ] **Step 10: Commit**

```bash
git add db/city_migrate/20260915000001_create_city_profile.rb db/city_migrate/20260915000002_allow_platform_invitations.rb \
  db/city_schema.rb app/models/city_profile.rb app/models/invitation.rb spec/models/city_profile_spec.rb \
  spec/commands/accept_invitation_spec.rb db/seeds.rb app/jobs/dispatch_municipality_alert_job.rb
git commit -m "Add the city_profile singleton and allow platform-issued invitations

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: `rota_provisioner` e banco + role por cidade

**Files:**
- Modify: `lib/tasks/platform.rake`
- Create: `app/services/city_database.rb`
- Create: `spec/services/city_database_spec.rb`
- Create: `spec/support/provisioned_cities.rb`
- Modify: `spec/rails_helper.rb` (require do suporte)

**Interfaces:**
- Consumes: `CityCatalog::RESERVED`, `CitySchema.redact`, `CityMigrations.run` (Task 1), `ScratchDatabases.superuser`
  (Task 1).
- Produces:
  - `CityDatabase::MAX_SLUG_LENGTH = 40`, `CityDatabase::InvalidSlug`, `CityDatabase::ProvisionerMissing`.
  - `CityDatabase.valid_slug?(slug) -> Boolean`, `CityDatabase.database_name(slug) -> String`,
    `CityDatabase.role_name(slug) -> String`.
  - `CityDatabase.url_for(slug:, password:) -> String`, `CityDatabase.provisioner_url -> String`.
  - `CityDatabase.ensure!(slug:, password:)`, `CityDatabase.drop!(slug:)`, `CityDatabase.exists?(slug:) -> Boolean`.
  - Spec: `provision_city!(status: "active") -> City` e `cleanup_provisioned_city!(city)`.

- [ ] **Step 1: Criar o role `rota_provisioner` pelo bootstrap**

Em `lib/tasks/platform.rake`, logo depois de `abort "[platform:bootstrap] falha no role:\n#{out}" unless st.success?`, inserir:

```ruby
    # Papel que cria e apaga banco e role de cada cidade (Plano 4): CREATEDB e
    # CREATEROLE, sem superusuário. Existe uma vez no cluster; o provisionamento
    # conecta com ele por PROVISIONER_DATABASE_URL (em dev/test, CityDatabase monta
    # a URL a partir de ROTA_PROVISIONER_PASSWORD).
    provisioner_pwd = ENV.fetch("ROTA_PROVISIONER_PASSWORD", "rota_provisioner")
    provisioner_sql = <<~SQL
      DO $$ BEGIN
        IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='rota_provisioner') THEN
          CREATE ROLE rota_provisioner LOGIN CREATEDB CREATEROLE PASSWORD '#{provisioner_pwd}';
        ELSE
          ALTER ROLE rota_provisioner LOGIN CREATEDB CREATEROLE NOSUPERUSER;
        END IF;
      END $$;
    SQL
    out, st = Open3.capture2e(env, *base, "-d", "postgres", "-c", provisioner_sql)
    abort "[platform:bootstrap] falha no role rota_provisioner:\n#{out}" unless st.success?
```

Run: `docker compose exec -T api ./bin/rails platform:bootstrap`
Expected: termina sem erro (cria `rota_provisioner`; o resto é idempotente).

Run (leitura): `docker compose exec -T api bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -h $DATABASE_HOST -U rota_saude -d postgres -tA -c "select rolname, rolcreatedb, rolcreaterole, rolsuper from pg_roles where rolname = '"'"'rota_provisioner'"'"'"'`
Expected: `rota_provisioner|t|t|f`.

- [ ] **Step 2: Specs de `CityDatabase` (falham)**

Criar `spec/services/city_database_spec.rb`:

```ruby
require "rails_helper"

# Banco e role por cidade (spec banco-por-cidade §4, Plano 4). Cria bancos e roles
# DE VERDADE com rota_provisioner — sem transação de fixture — e apaga tudo no
# `after`. Slugs sempre com prefixo "prov".
RSpec.describe CityDatabase do
  self.use_transactional_tests = false

  let(:slug_a) { "prov#{SecureRandom.hex(4)}" }
  let(:slug_b) { "prov#{SecureRandom.hex(4)}" }
  let(:pwd_a) { SecureRandom.hex(24) }
  let(:pwd_b) { SecureRandom.hex(24) }

  after { [ slug_a, slug_b ].each { |slug| described_class.drop!(slug: slug) } }

  def superuser_value(sql, *params)
    ScratchDatabases.superuser { |conn| conn.exec_params(sql, params).getvalue(0, 0) }
  end

  it "names databases and roles per environment and refuses slugs that are unsafe as identifiers" do
    expect(described_class.database_name("curitiba")).to eq("rota_saude_test_city_curitiba")
    expect(described_class.role_name("curitiba")).to eq("rota_test_city_curitiba")

    [ "a", "x" * 41, "Maiuscula", "com espaco", "-hifen", "admin", %w[lista], nil ].each do |bad|
      expect { described_class.database_name(bad) }.to raise_error(CityDatabase::InvalidSlug)
      expect(described_class.valid_slug?(bad)).to be(false)
    end
    expect(described_class.valid_slug?("x" * 40)).to be(true)
  end

  it "builds the city URL with the city's own role, on the provisioner's host" do
    url = URI.parse(described_class.url_for(slug: "curitiba", password: "abc123"))
    provisioner = URI.parse(described_class.provisioner_url)

    expect([ url.scheme, url.user, url.password, url.host, url.port, url.path ])
      .to eq([ "postgres", "rota_test_city_curitiba", "abc123", provisioner.host, provisioner.port, "/rota_saude_test_city_curitiba" ])
  end

  it "creates a role that owns its database, with CONNECT revoked from PUBLIC, idempotently" do
    2.times { described_class.ensure!(slug: slug_a, password: pwd_a) }

    database = described_class.database_name(slug_a)
    expect(described_class.exists?(slug: slug_a)).to be(true)
    expect(superuser_value("SELECT pg_get_userbyid(datdba) FROM pg_database WHERE datname = $1", database))
      .to eq(described_class.role_name(slug_a))
    expect(superuser_value("SELECT datacl::text FROM pg_database WHERE datname = $1", database)).not_to match(/[{,]=/)
    expect(superuser_value("SELECT rolpassword FROM pg_authid WHERE rolname = $1", described_class.role_name(slug_a)))
      .to start_with("SCRAM-SHA-256$")
    PG.connect(described_class.url_for(slug: slug_a, password: pwd_a)).close
  end

  it "keeps a city's role out of another city's database, and rota_app out of both" do
    described_class.ensure!(slug: slug_a, password: pwd_a)
    described_class.ensure!(slug: slug_b, password: pwd_b)

    cross = described_class.url_for(slug: slug_a, password: pwd_a)
                           .sub("/#{described_class.database_name(slug_a)}", "/#{described_class.database_name(slug_b)}")
    expect { PG.connect(cross) }.to raise_error(PG::ConnectionBad, /permission denied/)

    rota_app = CityDatabaseUrls.city_database_url(described_class.database_name(slug_a),
                                                  user: "rota_app", password: ENV.fetch("ROTA_APP_PASSWORD", "rota_app"))
    expect { PG.connect(rota_app) }.to raise_error(PG::ConnectionBad, /permission denied/)
  end

  it "realigns the role password with the catalog when run again" do
    role = described_class.role_name(slug_a)
    described_class.ensure!(slug: slug_a, password: "antiga#{pwd_a}")
    before = superuser_value("SELECT rolpassword FROM pg_authid WHERE rolname = $1", role)

    described_class.ensure!(slug: slug_a, password: pwd_a)

    expect(superuser_value("SELECT rolpassword FROM pg_authid WHERE rolname = $1", role)).not_to eq(before)
    PG.connect(described_class.url_for(slug: slug_a, password: pwd_a)).close
  end

  it "drops database and role, and dropping again is a no-op" do
    described_class.ensure!(slug: slug_a, password: pwd_a)

    2.times { described_class.drop!(slug: slug_a) }

    expect(described_class.exists?(slug: slug_a)).to be(false)
    expect(superuser_value("SELECT count(*) FROM pg_roles WHERE rolname = $1", described_class.role_name(slug_a))).to eq("0")
  end

  it "requires PROVISIONER_DATABASE_URL in production" do
    allow(Rails.env).to receive(:production?).and_return(true)
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with("PROVISIONER_DATABASE_URL").and_return(nil)

    expect { described_class.provisioner_url }.to raise_error(CityDatabase::ProvisionerMissing)
  end
end
```

- [ ] **Step 3: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_database_spec.rb`
Expected: FAIL com `uninitialized constant CityDatabase`.

- [ ] **Step 4: `CityDatabase`**

Criar `app/services/city_database.rb`:

```ruby
# Banco e role de uma cidade no Postgres (spec banco-por-cidade §4, Plano 4).
#
# Tudo aqui conecta como rota_provisioner — CREATEDB e CREATEROLE, sem
# superusuário — por PROVISIONER_DATABASE_URL. Em produção só o worker recebe essa
# URL: o processo web não cria nem apaga banco.
#
# Cada cidade tem um role próprio, DONO do seu banco, usado em runtime e nas
# migrations. O CONNECT de PUBLIC é revogado: o role de uma cidade (e rota_app)
# não abre o banco de outra. rota_provisioner vira membro de cada role de cidade —
# no Postgres 16 quem cria um role não herda os privilégios dele, e sem essa
# associação não poderia dar o banco a esse dono nem apagá-lo depois.
#
# A senha do role vai cifrada (SCRAM) no SQL, nunca em texto. Nomes derivam do
# slug, passam por quote_ident, e o slug precisa ser rótulo DNS de até
# MAX_SLUG_LENGTH caracteres, para o nome caber nos 63 bytes de identificador.
class CityDatabase
  class InvalidSlug < ArgumentError; end
  class ProvisionerMissing < StandardError; end

  MAX_SLUG_LENGTH = 40
  SLUG = /\A[a-z0-9]([a-z0-9-]*[a-z0-9])?\z/

  class << self
    def valid_slug?(slug)
      slug.is_a?(String) && slug.length.between?(2, MAX_SLUG_LENGTH) && slug.match?(SLUG) &&
        !CityCatalog::RESERVED.include?(slug)
    end

    def database_name(slug)
      check!(slug)
      Rails.env.test? ? "rota_saude_test_city_#{slug}" : "rota_saude_city_#{slug}"
    end

    def role_name(slug)
      check!(slug)
      Rails.env.test? ? "rota_test_city_#{slug}" : "rota_city_#{slug}"
    end

    # URL que vai para cities.database_url: role e banco da cidade, no mesmo
    # servidor do provisioner.
    def url_for(slug:, password:)
      server = URI.parse(provisioner_url)
      URI::Generic.build(scheme: "postgres", userinfo: "#{role_name(slug)}:#{password}",
                         host: server.host, port: server.port, path: "/#{database_name(slug)}").to_s
    end

    # Idempotente: cria o que falta e realinha a senha do role com a do catálogo
    # (um retry depois de uma falha no meio não deixa senha divergente).
    def ensure!(slug:, password:)
      role = role_name(slug)
      database = database_name(slug)

      with_provisioner do |conn|
        secret = conn.escape_literal(conn.encrypt_password(password, role, "scram-sha-256"))
        verb = role_exists?(conn, role) ? "ALTER" : "CREATE"
        conn.exec("#{verb} ROLE #{quote(role)} WITH LOGIN PASSWORD #{secret}")
        conn.exec("GRANT #{quote(role)} TO #{quote(conn.user)}")
        conn.exec("CREATE DATABASE #{quote(database)} OWNER #{quote(role)}") unless database_exists?(conn, database)
        conn.exec("REVOKE ALL ON DATABASE #{quote(database)} FROM PUBLIC")
      end
    end

    # Apaga banco e role (offboarding; limpeza de specs). Idempotente. FORCE derruba
    # as conexões ainda abertas no banco.
    def drop!(slug:)
      with_provisioner do |conn|
        conn.exec("DROP DATABASE IF EXISTS #{quote(database_name(slug))} WITH (FORCE)")
        conn.exec("DROP ROLE IF EXISTS #{quote(role_name(slug))}")
      end
    end

    def exists?(slug:)
      with_provisioner { |conn| database_exists?(conn, database_name(slug)) }
    end

    def provisioner_url
      ENV["PROVISIONER_DATABASE_URL"].presence || local_provisioner_url
    end

    private

    def check!(slug)
      raise InvalidSlug, "slug inválido para banco de cidade: #{slug.inspect}" unless valid_slug?(slug)
    end

    def local_provisioner_url
      raise ProvisionerMissing, "PROVISIONER_DATABASE_URL ausente" if Rails.env.production?

      host = ENV.fetch("DATABASE_HOST", "127.0.0.1")
      port = ENV.fetch("DATABASE_PORT", "5432")
      pwd  = ENV.fetch("ROTA_PROVISIONER_PASSWORD", "rota_provisioner")
      "postgres://rota_provisioner:#{pwd}@#{host}:#{port}/postgres"
    end

    def with_provisioner
      conn = PG.connect(provisioner_url)
      conn.set_notice_receiver { |_| }
      yield conn
    ensure
      conn&.close
    end

    def role_exists?(conn, role)
      conn.exec_params("SELECT 1 FROM pg_roles WHERE rolname = $1", [ role ]).ntuples == 1
    end

    def database_exists?(conn, database)
      conn.exec_params("SELECT 1 FROM pg_database WHERE datname = $1", [ database ]).ntuples == 1
    end

    def quote(identifier)
      PG::Connection.quote_ident(identifier)
    end
  end
end
```

- [ ] **Step 5: Suporte de cidade provisionada para os specs seguintes**

Criar `spec/support/provisioned_cities.rb`:

```ruby
# Cidade provisionada DE VERDADE para specs de ciclo de vida (Plano 4): linha no
# catálogo, banco e role próprios (CityDatabase) e schema migrado. Exige
# `self.use_transactional_tests = false`.
#
# Sempre limpe com cleanup_provisioned_city!: um pool registrado para um banco
# apagado derruba todo exemplo transacional seguinte (setup_transactional_fixtures
# pina todo pool registrado).
module ProvisionedCities
  def provision_city!(status: "active")
    slug = "prov#{SecureRandom.hex(4)}"
    password = SecureRandom.hex(24)
    CityDatabase.ensure!(slug: slug, password: password)
    city = City.create!(slug: slug, name: "Cidade #{slug}", uf: "PR", status: "provisioning",
                        database_url: CityDatabase.url_for(slug: slug, password: password),
                        encryption_key: SecureRandom.hex(32))
    CityMigrations.run(city)
    city.update!(status: status)
    city
  end

  def cleanup_provisioned_city!(city)
    CityConnection.forget(city.shard)
    CityDatabase.drop!(slug: city.slug)
    CityChannel.where(city_id: city.id).delete_all
    CityGrant.where(city_id: city.id).delete_all
    PlatformEvent.where("payload->>'city_id' = ?", city.id).delete_all
    City.where(id: city.id).delete_all
  end
end

RSpec.configure do |config|
  config.include ProvisionedCities
end
```

Em `spec/rails_helper.rb`, depois de `require_relative "support/scratch_databases"`, acrescentar:

```ruby
require_relative "support/provisioned_cities"
```

- [ ] **Step 6: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_database_spec.rb`
Expected: PASS.

Run (leitura): `docker compose exec -T api bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -h $DATABASE_HOST -U rota_saude -d postgres -tA -c "select count(*) from pg_database where datname like '"'"'rota_saude_test_city_prov%'"'"'" -c "select count(*) from pg_roles where rolname like '"'"'rota_test_city_prov%'"'"'"'`
Expected: `0` e `0` (nada sobrou).

- [ ] **Step 7: Suíte inteira**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

- [ ] **Step 8: Commit**

```bash
git add lib/tasks/platform.rake app/services/city_database.rb spec/services/city_database_spec.rb \
  spec/support/provisioned_cities.rb spec/rails_helper.rb
git commit -m "Create and drop a least-privilege database and role per city

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 5: Provisionamento em duas fases — `ProvisionCity` e `ProvisionCityJob`

**Files:**
- Create: `app/commands/provision_city.rb`
- Create: `app/jobs/provision_city_job.rb`
- Modify: `app/services/city_migrations.rb` (acrescentar `Subprocess`)
- Create: `app/services/city_templates.rb`
- Create: `config/city_templates/triage_respiratoria.json`
- Create: `app/mailers/invitation_mailer.rb`
- Create: `app/views/invitation_mailer/invite.text.erb`
- Create: `app/views/invitation_mailer/invite.html.erb`
- Modify: `app/services/city_dashboard_url.rb`
- Modify: `db/seeds.rb` (protocolo vem do template)
- Create: `spec/commands/provision_city_spec.rb`
- Create: `spec/jobs/provision_city_job_spec.rb`
- Create: `spec/services/city_migrations_subprocess_spec.rb`
- Create: `spec/mailers/invitation_mailer_spec.rb`

**Interfaces:**
- Consumes:
  - `CityDatabase.valid_slug?`, `url_for`, `ensure!` (Task 4).
  - `CityMigrations.run` (Task 1).
  - `CityProfile` e `Invitation` sem `invited_by` (Task 3).
  - `InviteMember.call(email:, role:, invited_by:)`, `SeedProtocol.call(template:)`, `Platform.audit`.
  - `ProvisionedCities#cleanup_provisioned_city!` (Task 4).
- Produces:
  - `ProvisionCity.call(slug:, name:, uf:, ibge_code:, admin_email:, alert_email:, by:) -> Result`: ok com
    `payload[:city]`; falha com `:invalid` ou `:city_exists`.
  - `ProvisionCityJob#perform(city_id:, ibge_code:, admin_email:, alert_email:, operator_id:)` e
    `ProvisionCityJob.migrator` (callable `call(city)`).
  - `CityMigrations::Subprocess.call(city) -> City` e `CityMigrations::Subprocess::Failed`.
  - `CityTemplates.protocol -> { name: String, definition: Hash }`.
  - `InvitationMailer.invite(email_address:, accept_url:)`.
  - `CityDashboardUrl.invitation(city, token:) -> String` e `CityDashboardUrl.base(city) -> String`.

- [ ] **Step 1: Specs (falham)**

Criar `spec/commands/provision_city_spec.rb`:

```ruby
require "rails_helper"

# Fase 1 do provisionamento em duas fases (spec banco-por-cidade §4, Plano 4): só
# registra no catálogo e enfileira. Quem cria banco é o worker (ProvisionCityJob).
RSpec.describe ProvisionCity do
  include ActiveJob::TestHelper

  let(:operator) do
    Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end
  let(:args) do
    { slug: "novacidade", name: "Nova Cidade", uf: "PR", ibge_code: "4113700",
      admin_email: "prefeita@novacidade.gov.br", alert_email: "alertas@novacidade.gov.br", by: operator }
  end

  it "registers the city as provisioning, with its own role and database, and enqueues phase two" do
    result = nil
    expect { result = described_class.call(**args) }.to have_enqueued_job(ProvisionCityJob).with(
      city_id: kind_of(String), ibge_code: "4113700", admin_email: "prefeita@novacidade.gov.br",
      alert_email: "alertas@novacidade.gov.br", operator_id: operator.id
    )

    expect(result.ok?).to be(true)
    city = result.payload[:city]
    expect(city).to have_attributes(slug: "novacidade", name: "Nova Cidade", uf: "PR", status: "provisioning",
                                    schema_version: nil)
    url = URI.parse(city.database_url)
    expect([ url.user, url.path ]).to eq([ "rota_test_city_novacidade", "/rota_saude_test_city_novacidade" ])
    expect(url.password).to match(/\A\h{48}\z/)
  end

  it "re-enqueues phase two for a city still provisioning, without a second catalog row or a new password" do
    first = described_class.call(**args).payload[:city]
    original_url = first.database_url

    expect { described_class.call(**args) }
      .to have_enqueued_job(ProvisionCityJob).with(hash_including(city_id: first.id))
    expect(City.where(slug: "novacidade").pluck(:id)).to eq([ first.id ])
    expect(first.reload.database_url).to eq(original_url)
  end

  it "refuses a slug that belongs to a city past provisioning, enqueuing nothing" do
    create(:city, slug: "novacidade", status: "active")

    result = nil
    expect { result = described_class.call(**args) }.not_to have_enqueued_job(ProvisionCityJob)
    expect(result.reason).to eq(:city_exists)
  end

  {
    slug: [ "x" * 41, "admin", "Maiuscula", %w[lista], nil ],
    name: [ "", %w[lista], nil ],
    uf: [ "pr", "PRX", nil ],
    ibge_code: [ "123", "41137000", nil ],
    admin_email: [ "nao-e-email", [ "a@b.co" ], nil ],
    alert_email: [ "nao-e-email", nil ]
  }.each do |field, bad_values|
    bad_values.each do |bad|
      it "refuses #{field}=#{bad.inspect.truncate(20)} without touching the catalog or the queue" do
        result = nil
        expect { result = described_class.call(**args.merge(field => bad)) }.not_to have_enqueued_job
        expect(result.reason).to eq(:invalid)
        expect(City.count).to eq(City.where(slug: TEST_CITY_A.slug).count)
      end
    end
  end
end
```

Criar `spec/jobs/provision_city_job_spec.rb`:

```ruby
require "rails_helper"

# Fase 2 do provisionamento (spec banco-por-cidade §4, Plano 4), contra bancos e
# roles DE VERDADE. A migração roda em processo aqui (a suíte não tem threads do
# Solid Queue); o subprocesso usado no worker é provado em
# city_migrations_subprocess_spec.rb.
RSpec.describe ProvisionCityJob, type: :job do
  self.use_transactional_tests = false

  let(:slug) { "prov#{SecureRandom.hex(4)}" }
  let(:operator_id) { SecureRandom.uuid }
  let(:args) do
    { ibge_code: "4113700", admin_email: "Prefeita@Cidade.gov.br", alert_email: "alertas@cidade.gov.br",
      operator_id: operator_id }
  end
  let!(:city) do
    City.create!(slug: slug, name: "Cidade Nova", uf: "PR", status: "provisioning",
                 database_url: CityDatabase.url_for(slug: slug, password: SecureRandom.hex(24)),
                 encryption_key: SecureRandom.hex(32))
  end

  around do |example|
    migrator_was = described_class.migrator
    described_class.migrator = ->(c) { CityMigrations.run(c) }
    example.run
  ensure
    described_class.migrator = migrator_was
  end

  before { allow(InvitationMailer).to receive(:invite).and_call_original }

  after { cleanup_provisioned_city!(city) }

  def provisioned_events
    PlatformEvent.where(name: "municipality.provisioned").where("payload->>'city_id' = ?", city.id)
  end

  it "creates database and role, migrates, seeds the city, activates it and audits once" do
    described_class.perform_now(city_id: city.id, **args)

    city.reload
    expect(city.status).to eq("active")
    expect(city.schema_version).to eq(CitySchema.expected_version.to_s)
    expect(URI.parse(city.database_url).user).to eq(CityDatabase.role_name(slug))

    CityConnection.with(city) do
      expect(CityProfile.current).to have_attributes(name: "Cidade Nova", uf: "PR", ibge_code: "4113700")
      expect(AlertRecipient.pluck(:channel, :destination, :active)).to eq([ [ "email", "alertas@cidade.gov.br", true ] ])
      expect(ProtocolDefinition.pluck(:name, :status)).to eq([ %w[triage-respiratoria draft] ])
      expect(Invitation.pluck(:email, :role, :invited_by_id)).to eq([ [ "prefeita@cidade.gov.br", "municipal_admin", nil ] ])
      expect(ConsentTerm.count).to eq(0)
    end

    expect(provisioned_events.count).to eq(1)
    expect(provisioned_events.first.payload.keys).to contain_exactly("city_id", "ibge_code", "by")
    expect(provisioned_events.first.payload).to include("ibge_code" => "4113700", "by" => operator_id)
  end

  it "e-mails the invitation link to the first admin" do
    described_class.perform_now(city_id: city.id, **args)

    token = CityConnection.with(city) { Invitation.sole.token }
    expect(InvitationMailer).to have_received(:invite)
      .with(email_address: "Prefeita@Cidade.gov.br", accept_url: CityDashboardUrl.invitation(city, token: token)).once
  end

  it "stays provisioning after a failure and resumes without duplicating anything" do
    described_class.migrator = ->(_c) { raise "migração caiu" }
    described_class.perform_now(city_id: city.id, **args) # retry_on engole e reagenda

    expect(city.reload.status).to eq("provisioning")
    expect(CityDatabase.exists?(slug: slug)).to be(true)

    described_class.migrator = ->(c) { CityMigrations.run(c) }
    2.times { described_class.perform_now(city_id: city.id, **args) }

    expect(city.reload.status).to eq("active")
    CityConnection.with(city) do
      expect([ CityProfile.count, AlertRecipient.count, ProtocolDefinition.count, Invitation.count ]).to eq([ 1, 1, 1, 1 ])
    end
    expect(provisioned_events.count).to eq(1)
    expect(InvitationMailer).to have_received(:invite).once
  end

  it "does not e-mail the invitation twice when activation fails after seeding" do
    allow(Platform).to receive(:audit).and_raise(ActiveRecord::StatementInvalid, "plataforma caiu")
    described_class.perform_now(city_id: city.id, **args)

    expect(city.reload.status).to eq("provisioning")
    expect(InvitationMailer).to have_received(:invite).once

    allow(Platform).to receive(:audit).and_call_original
    described_class.perform_now(city_id: city.id, **args)

    expect(city.reload.status).to eq("active")
    expect(InvitationMailer).to have_received(:invite).once
    CityConnection.with(city) { expect(Invitation.count).to eq(1) }
  end

  it "ignores a city that is not provisioning, and an unknown id" do
    city.update!(status: "suspended")
    expect(CityDatabase).not_to receive(:ensure!)

    described_class.perform_now(city_id: city.id, **args)
    described_class.perform_now(city_id: SecureRandom.uuid, **args)

    expect(city.reload.status).to eq("suspended")
  end
end
```

Criar `spec/services/city_migrations_subprocess_spec.rb`:

```ruby
require "rails_helper"

# Dentro do worker a migração da cidade não pode trocar a conexão de
# ActiveRecord::Base (o Solid Queue usa a mesma): roda em outro processo.
RSpec.describe CityMigrations::Subprocess do
  let(:city) { create(:city, slug: "subproc#{SecureRandom.hex(3)}", status: "provisioning") }

  it "runs city:migrate for the slug in a separate rails process and returns the reloaded city" do
    status = instance_double(Process::Status, success?: true, exitstatus: 0)
    expect(Open3).to receive(:capture2e)
      .with({ "RAILS_ENV" => Rails.env }, Rails.root.join("bin/rails").to_s, "city:migrate[#{city.slug}]",
            chdir: Rails.root.to_s)
      .and_return([ "[city:migrate] ok", status ])

    expect(described_class.call(city)).to eq(city)
  end

  it "raises with the tail of the output, credentials redacted, when the task fails" do
    status = instance_double(Process::Status, success?: false, exitstatus: 1)
    allow(Open3).to receive(:capture2e).and_return([ "boom em postgres://rota_city_x:s3gr3d0@db/x\n", status ])

    expect { described_class.call(city) }.to raise_error(CityMigrations::Subprocess::Failed) { |error|
      expect(error.message).to include(city.slug).and include("://***@")
      expect(error.message).not_to include("s3gr3d0")
    }
  end
end
```

Criar `spec/mailers/invitation_mailer_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe InvitationMailer, type: :mailer do
  let(:accept_url) { "http://novacidade.localhost:5175/dashboard/?invite=tok-123" }

  # R42: só valores simples — roda no worker, sem conexão de cidade.
  it "addresses the given e-mail and carries the given invitation link" do
    mail = described_class.invite(email_address: "prefeita@novacidade.gov.br", accept_url: accept_url)

    expect(mail.to).to eq([ "prefeita@novacidade.gov.br" ])
    expect(mail.subject).to be_present
    expect(mail.text_part.body.decoded).to include(accept_url)
    expect(mail.html_part.body.decoded).to include(accept_url)
  end
end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/commands/provision_city_spec.rb spec/jobs/provision_city_job_spec.rb spec/services/city_migrations_subprocess_spec.rb spec/mailers/invitation_mailer_spec.rb`
Expected: FAIL com `uninitialized constant` para `ProvisionCity`, `ProvisionCityJob`, `CityMigrations::Subprocess` e
`InvitationMailer`.

- [ ] **Step 3: Template de protocolo e seeds**

Criar `config/city_templates/triage_respiratoria.json`:

```json
{
  "name": "triage-respiratoria",
  "version": 1,
  "start_step_id": "tosse",
  "steps": [
    {
      "id": "tosse",
      "prompt": "Você está com tosse?",
      "answer_type": "boolean",
      "branches": { "true": "febre", "false": null },
      "weights": { "true": 3, "false": 0 }
    },
    {
      "id": "febre",
      "prompt": "Está com febre alta?",
      "answer_type": "boolean",
      "branches": { "true": null, "false": null },
      "weights": { "true": 5, "false": 0 }
    }
  ],
  "scoring": {
    "type": "weighted",
    "thresholds": { "baixa": 0, "alta": 5 },
    "priority_map": { "baixa": 9, "alta": 1 }
  },
  "recommendations": {
    "alta": {
      "title": "Procure atendimento hoje",
      "body": "Prioridade alta. Vá à UPA/unidade mais próxima ainda hoje. Falta de ar, dor no peito ou lábios roxos → 192."
    },
    "baixa": {
      "title": "Cuidados em casa",
      "body": "Repouso e hidratação. Se piorar ou persistir por mais de 3 dias, procure sua unidade de saúde."
    }
  }
}
```

Criar `app/services/city_templates.rb`:

```ruby
# Conteúdo inicial de uma cidade nova (Plano 4): o protocolo template que o
# provisionamento semeia como rascunho, e que os seeds de dev ativam.
module CityTemplates
  PROTOCOL_PATH = "config/city_templates/triage_respiratoria.json"

  module_function

  # { name:, definition: } — o formato de SeedProtocol.call(template:).
  def protocol
    definition = JSON.parse(Rails.root.join(PROTOCOL_PATH).read)
    { name: definition.fetch("name"), definition: definition }
  end
end
```

Em `db/seeds.rb`, trocar o literal inteiro que começa em `  protocol_defn = {` e termina no `  }` que fecha
`"recommendations"` por:

```ruby
  # Mesmo protocolo que o provisionamento semeia em rascunho (Plano 4); aqui ativo.
  protocol_defn = CityTemplates.protocol.fetch(:definition)
```

- [ ] **Step 4: Mailer e URL do convite**

Substituir o conteúdo de `app/services/city_dashboard_url.rb` por:

```ruby
# URLs do dashboard de uma cidade (Planos 3B e 4): a entrada com grant, para onde o
# console e o callback do gov.br mandam o navegador, e o link do convite do
# primeiro municipal_admin. O host por cidade vem de um template porque os
# frontends ainda não têm host por cidade (Plano 6).
module CityDashboardUrl
  DEFAULT_TEMPLATE = "http://%{slug}.localhost:5175/dashboard/"

  module_function

  def for(city, grant:)
    "#{base(city)}?#{{ grant: grant }.to_query}"
  end

  # A tela que lê `invite` é do Plano 6; até lá o aceite é POST /setup/accept_invitation.
  def invitation(city, token:)
    "#{base(city)}?#{{ invite: token }.to_query}"
  end

  def base(city)
    format(ENV.fetch("CITY_DASHBOARD_URL_TEMPLATE", DEFAULT_TEMPLATE), slug: city.slug)
  end
end
```

Criar `app/mailers/invitation_mailer.rb`:

```ruby
# Convite do primeiro municipal_admin de uma cidade recém-provisionada (Plano 4).
#
# Recebe SÓ valores simples (R42): deliver_later roda no worker, sem conexão de
# cidade. O ProvisionCityJob monta o endereço e o link.
class InvitationMailer < ApplicationMailer
  def invite(email_address:, accept_url:)
    @accept_url = accept_url
    mail(to: email_address, subject: "[rota-saúde] Convite para administrar sua cidade")
  end
end
```

Criar `app/views/invitation_mailer/invite.text.erb`:

```erb
Olá,

Sua cidade foi cadastrada no Rota Saúde e você foi convidado(a) para administrá-la.

Crie sua senha e aceite o convite: <%= @accept_url %>

O convite expira em 7 dias. Se você não esperava este e-mail, ignore-o.
```

Criar `app/views/invitation_mailer/invite.html.erb`:

```erb
<p>Olá,</p>
<p>Sua cidade foi cadastrada no Rota Saúde e você foi convidado(a) para administrá-la.</p>
<p><a href="<%= @accept_url %>">Crie sua senha e aceite o convite</a>.</p>
<p>O convite expira em 7 dias. Se você não esperava este e-mail, ignore-o.</p>
```

- [ ] **Step 5: Migração em subprocesso**

Em `app/services/city_migrations.rb`, acrescentar `require "open3"` na primeira linha do arquivo e, antes do `end`
final do módulo:

```ruby
  # Migra a cidade num processo à parte (bin/rails city:migrate[slug]) e devolve a
  # City recarregada, com o schema_version gravado pela task. É o migrador do
  # ProvisionCityJob: dentro do worker, CitySchema.migrate! trocaria a conexão de
  # ActiveRecord::Base, que as threads do Solid Queue usam.
  module Subprocess
    class Failed < StandardError; end

    module_function

    def call(city)
      out, status = Open3.capture2e({ "RAILS_ENV" => Rails.env }, Rails.root.join("bin/rails").to_s,
                                    "city:migrate[#{city.slug}]", chdir: Rails.root.to_s)
      unless status.success?
        raise Failed, "city:migrate[#{city.slug}] saiu com #{status.exitstatus}: " \
                      "#{CitySchema.redact(out.lines.last(5).join).strip}"
      end

      city.reload
    end
  end
```

- [ ] **Step 6: Fase 1 — `ProvisionCity`**

Criar `app/commands/provision_city.rb`:

```ruby
# Fase 1 do provisionamento em duas fases (spec banco-por-cidade §4, Plano 4):
# registra a cidade no catálogo como `provisioning` e enfileira o ProvisionCityJob,
# que cria banco e role, migra, semeia e ativa. O processo web não cria banco —
# só o worker, com a credencial de rota_provisioner.
#
# A senha do role da cidade nasce aqui e fica no catálogo (database_url cifrada):
# todo retry do job usa a mesma.
#
# Idempotente: repetir com o slug de uma cidade ainda em provisioning reenfileira
# o job para ela — é assim que se retoma um provisionamento que falhou. Slug de
# cidade em qualquer outro estado → :city_exists.
class ProvisionCity
  UF = /\A[A-Z]{2}\z/
  IBGE_CODE = /\A\d{7}\z/

  def self.call(slug:, name:, uf:, ibge_code:, admin_email:, alert_email:, by:)
    new(slug: slug, name: name, uf: uf, ibge_code: ibge_code, admin_email: admin_email,
        alert_email: alert_email, by: by).call
  end

  def initialize(slug:, name:, uf:, ibge_code:, admin_email:, alert_email:, by:)
    @slug, @name, @uf, @ibge_code = slug, name, uf, ibge_code
    @admin_email, @alert_email, @by = admin_email, alert_email, by
  end

  def call
    errors = validation_errors
    return Result.fail(:invalid, message: errors.join(", ")) if errors.any?

    city = City.find_by(slug: @slug)
    if city && city.status != "provisioning"
      return Result.fail(:city_exists, message: "cidade #{@slug} já existe (status=#{city.status})")
    end

    city ||= City.create!(slug: @slug, name: @name, uf: @uf, status: "provisioning",
                          database_url: CityDatabase.url_for(slug: @slug, password: SecureRandom.hex(24)),
                          encryption_key: SecureRandom.hex(32))

    ProvisionCityJob.perform_later(city_id: city.id, ibge_code: @ibge_code, admin_email: @admin_email,
                                   alert_email: @alert_email, operator_id: @by.id)
    Result.ok(city: city)
  rescue ActiveRecord::RecordInvalid => e
    Result.fail(:invalid, message: e.record.errors.full_messages.join(", "))
  rescue ActiveRecord::RecordNotUnique
    Result.fail(:city_exists, message: "cidade #{@slug} já existe")
  end

  private

  def validation_errors
    [].tap do |errors|
      errors << "slug inválido" unless CityDatabase.valid_slug?(@slug)
      errors << "name obrigatório" unless @name.is_a?(String) && @name.present?
      errors << "uf inválida" unless string_matching?(@uf, UF)
      errors << "ibge_code inválido" unless string_matching?(@ibge_code, IBGE_CODE)
      errors << "admin_email inválido" unless string_matching?(@admin_email, URI::MailTo::EMAIL_REGEXP)
      errors << "alert_email inválido" unless string_matching?(@alert_email, URI::MailTo::EMAIL_REGEXP)
    end
  end

  def string_matching?(value, pattern)
    value.is_a?(String) && value.match?(pattern)
  end
end
```

- [ ] **Step 7: Fase 2 — `ProvisionCityJob`**

Criar `app/jobs/provision_city_job.rb`:

```ruby
# Fase 2 do provisionamento (spec banco-por-cidade §4, Plano 4). Roda no worker,
# o único processo com PROVISIONER_DATABASE_URL:
#   1. banco e role da cidade (CityDatabase.ensure!);
#   2. migrations, em subprocesso (CityMigrations::Subprocess), que grava
#      cities.schema_version;
#   3. no banco da cidade, numa transação: city_profile, destinatário de alerta,
#      protocolo template em rascunho e convite do primeiro municipal_admin
#      (convidado pela plataforma: invited_by nulo);
#   4. e-mail do convite, SÓ quando o convite nasceu nesta execução;
#   5. cidade → active e municipality.provisioned, numa transação de plataforma.
#
# Cada passo é idempotente: um retry (ou um novo POST /cities com o mesmo slug)
# retoma de onde parou sem duplicar nada. O e-mail vem antes da ativação: se a
# ativação falhar, o retry encontra o convite e não o reenvia. Cidade fora de
# provisioning é ignorada — ativa, suspensa ou arquivada não volta a ser
# provisionada.
#
# NÃO semeia consent_terms (Plano 4, decisão 8): Consents.current_version lê
# ConsentTerm.maximum(:version) e o texto do termo vem das credentials.
class ProvisionCityJob < ApplicationJob
  queue_as :default

  retry_on StandardError, attempts: 3, wait: :polynomially_longer

  class_attribute :migrator, default: CityMigrations::Subprocess

  class SeedFailed < StandardError; end

  def perform(city_id:, ibge_code:, admin_email:, alert_email:, operator_id:)
    city = City.find_by(id: city_id)
    return unless city&.status == "provisioning"

    CityDatabase.ensure!(slug: city.slug, password: URI.parse(city.database_url).password)
    migrator.call(city)
    city.reload

    token = seed(city, ibge_code: ibge_code, admin_email: admin_email, alert_email: alert_email)
    if token
      InvitationMailer.invite(email_address: admin_email,
                              accept_url: CityDashboardUrl.invitation(city, token: token)).deliver_later
    end

    PlatformRecord.transaction do
      city.update!(status: "active")
      Platform.audit("municipality.provisioned", city_id: city.id, ibge_code: ibge_code, by: operator_id)
    end
    CityCatalog.reset_cache!
  end

  private

  # Devolve o token do convite quando ele foi criado AGORA; nil se já existia.
  def seed(city, ibge_code:, admin_email:, alert_email:)
    token = nil

    Current.set(city: city) do
      CityConnection.with(city) do
        ApplicationRecord.transaction do
          CityProfile.create!(name: city.name, uf: city.uf, ibge_code: ibge_code) unless CityProfile.exists?

          unless AlertRecipient.exists?
            AlertRecipient.create!(channel: "email", destination: alert_email, escalation_order: 0, active: true)
          end

          template = CityTemplates.protocol
          SeedProtocol.call(template: template) unless ProtocolDefinition.exists?(name: template.fetch(:name))

          unless Invitation.exists?(email: admin_email.downcase, role: "municipal_admin")
            invited = InviteMember.call(email: admin_email, role: "municipal_admin", invited_by: nil)
            raise SeedFailed, "convite do primeiro municipal_admin: #{invited.message}" if invited.failure?

            token = invited.payload[:invitation].token
          end
        end
      end
    end

    token
  end
end
```

- [ ] **Step 8: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/commands/provision_city_spec.rb spec/jobs/provision_city_job_spec.rb spec/services/city_migrations_subprocess_spec.rb spec/mailers/invitation_mailer_spec.rb spec/requests/operators/city_grants_spec.rb`
Expected: PASS. `city_grants_spec` confirma que `CityDashboardUrl.for` não mudou.

Run (leitura): `docker compose exec -T api bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -h $DATABASE_HOST -U rota_saude -d postgres -tA -c "select count(*) from pg_database where datname like '"'"'rota_saude_test_city_prov%'"'"'"'`
Expected: `0`.

- [ ] **Step 9: Suíte inteira e seeds**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

Run: `docker compose exec -T api ./bin/rails db:seed`
Expected: `protocolo ... triage-respiratoria v1 (active)` nas duas cidades (o template lido do JSON é o mesmo protocolo).

- [ ] **Step 10: Commit**

```bash
git add app/commands/provision_city.rb app/jobs/provision_city_job.rb app/services/city_migrations.rb \
  app/services/city_templates.rb config/city_templates/triage_respiratoria.json app/mailers/invitation_mailer.rb \
  app/views/invitation_mailer app/services/city_dashboard_url.rb db/seeds.rb spec/commands/provision_city_spec.rb \
  spec/jobs/provision_city_job_spec.rb spec/services/city_migrations_subprocess_spec.rb spec/mailers/invitation_mailer_spec.rb
git commit -m "Provision a city in two phases: register it, then create, migrate, seed and activate on the worker

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 6: Console provisiona (`POST /cities`, `GET /cities/:id`) e o fluxo antigo sai

**Files:**
- Create: `app/controllers/operators/cities_controller.rb`
- Create: `spec/requests/operators/cities_spec.rb`
- Modify: `config/routes.rb`
- Modify: `app/controllers/setup_controller.rb`
- Modify: `app/controllers/application_controller.rb` (comentário)
- Delete: `app/commands/provision_municipality.rb`
- Delete: `spec/commands/provision_municipality_spec.rb`
- Create: `app/commands/municipality_channels/register.rb`
- Create: `spec/commands/municipality_channels/register_spec.rb`
- Modify: `lib/tasks/channels.rake`
- Modify: `spec/events/platform_event_payload_guard_spec.rb`

**Interfaces:**
- Consumes: `ProvisionCity.call` (Task 5), `Operators::BaseController` e `current_operator` (Plano 3), `Platform.audit`.
- Produces:
  - `POST /cities` (host `admin.*`) → `202 { "id" }` | `409 { "error": "city_exists" }` |
    `422 { "error": "invalid", "message" }` | `401`.
  - `GET /cities/:id` (host `admin.*`) → `200 { "id", "slug", "status", "schema_version" }` | `404`.
  - `MunicipalityChannels::Register.call(city:, phone_number_id:, waba_id:, display_phone_number:, access_token:)`
    → `Result`: ok com `payload[:channel]`; falha com `:city_not_servable` ou `:invalid`.
  - Rake `channels:register`.
  - Evento de plataforma `channel.registered`.

**Por que apagar `ProvisionMunicipality` e o spec dele**, e onde fica provado cada invariante:

| Invariante | Onde passa a ser provado |
|---|---|
| canal da cidade criado na plataforma | `spec/commands/municipality_channels/register_spec.rb` |
| convite, alerta e protocolo gravados juntos no banco da cidade | `spec/jobs/provision_city_job_spec.rb` (Task 5) |
| `municipality.provisioned` com payload exato `city_id`, `ibge_code`, `by` | `spec/jobs/provision_city_job_spec.rb` (Task 5) |
| cidade não servível não recebe nada | `register_spec.rb` (canal); `provision_city_job_spec.rb` ("ignores a city that is not provisioning") |

O termo de consentimento deixa de ser semeado (decisão 8).

- [ ] **Step 1: Specs (falham)**

Criar `spec/requests/operators/cities_spec.rb`:

```ruby
require "rails_helper"

# Provisionamento pelo console de plataforma (spec banco-por-cidade §4, Plano 4).
RSpec.describe "City provisioning on the platform console", type: :request do
  let(:password) { "s3nha-forte-1" }
  let!(:operator) do
    Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end
  let(:params) do
    { slug: "novacidade", name: "Nova Cidade", uf: "PR", ibge_code: "4113700",
      admin_email: "prefeita@novacidade.gov.br", alert_email: "alertas@novacidade.gov.br" }
  end

  def json = JSON.parse(response.body)

  def verified_login!
    post "/session", params: { email_address: operator.email_address, password: password }
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(operator.otp_secret).now }
    expect(response).to have_http_status(:ok)
  end

  before { host! "admin.rotasaude.app" }

  it "registers the city as provisioning, enqueues phase two and answers only the id" do
    verified_login!

    expect { post "/cities", params: params }
      .to change(City, :count).by(1).and have_enqueued_job(ProvisionCityJob)

    expect(response).to have_http_status(:accepted)
    city = City.find_by!(slug: "novacidade")
    expect(json).to eq("id" => city.id)
    expect(city.status).to eq("provisioning")
  end

  it "does not serve the city while it is provisioning" do
    verified_login!
    post "/cities", params: params
    CityCatalog.reset_cache!

    host! "novacidade.rotasaude.app"
    get "/admin/api/overview"

    expect(response).to have_http_status(:not_found)
  end

  it "reports the provisioning status by id, and 404 for an unknown id" do
    verified_login!
    post "/cities", params: params
    id = json["id"]

    get "/cities/#{id}"
    expect(response).to have_http_status(:ok)
    expect(json).to eq("id" => id, "slug" => "novacidade", "status" => "provisioning", "schema_version" => nil)

    get "/cities/#{SecureRandom.uuid}"
    expect(response).to have_http_status(:not_found)
    get "/cities/nao-e-uuid"
    expect(response).to have_http_status(:not_found)
  end

  it "answers 409 for a slug of an active city and 422 for invalid input, enqueuing nothing" do
    verified_login!

    expect { post "/cities", params: params.merge(slug: TEST_CITY_A.slug) }.not_to have_enqueued_job
    expect(response).to have_http_status(:conflict)
    expect(json["error"]).to eq("city_exists")

    expect { post "/cities", params: params.merge(uf: "pr") }.not_to have_enqueued_job
    expect(response).to have_http_status(:unprocessable_entity)
    expect(json["error"]).to eq("invalid")
  end

  it "requires a verified operator session" do
    expect { post "/cities", params: params }.not_to change(City, :count)
    expect(response).to have_http_status(:unauthorized)

    get "/cities/#{City.find_by!(slug: TEST_CITY_A.slug).id}"
    expect(response).to have_http_status(:unauthorized)
  end

  it "is not reachable on a city host, and POST /setup/municipalities is gone" do
    host! test_city_host

    post "/cities", params: params
    expect(response).to have_http_status(:not_found)

    post "/setup/municipalities", params: params
    expect(response).to have_http_status(:not_found)
    expect(City.where(slug: "novacidade")).to be_empty
  end
end
```

Criar `spec/commands/municipality_channels/register_spec.rb`:

```ruby
require "rails_helper"

# Registro do canal WhatsApp de uma cidade (Plano 4) — antes, parte do
# ProvisionMunicipality.
RSpec.describe MunicipalityChannels::Register do
  let(:city) { create(:city, status: "active") }
  let(:attrs) do
    { phone_number_id: "PN-#{SecureRandom.hex(3)}", waba_id: "WABA-1", display_phone_number: "+55 41 99999-0000",
      access_token: "tok-secreto" }
  end

  it "registers an active channel on the platform and audits it without token or phone" do
    result = described_class.call(city: city, **attrs)

    expect(result.ok?).to be(true)
    expect(result.payload[:channel]).to have_attributes(city_id: city.id, phone_number_id: attrs[:phone_number_id], active: true)
    expect(PlatformEvent.find_by!(name: "channel.registered").payload)
      .to eq("city_id" => city.id, "phone_number_id" => attrs[:phone_number_id])
  end

  it "refuses a city that is not active and writes nothing" do
    provisioning = create(:city, status: "provisioning")

    result = nil
    expect { result = described_class.call(city: provisioning, **attrs) }.not_to change(PlatformEvent, :count)
    expect(result.reason).to eq(:city_not_servable)
    expect(CityChannel.where(city: provisioning)).to be_empty
  end

  it "refuses an empty token and a phone_number_id that is already registered" do
    expect(described_class.call(city: city, **attrs.merge(access_token: "")).reason).to eq(:invalid)

    described_class.call(city: city, **attrs)
    expect(described_class.call(city: create(:city), **attrs).reason).to eq(:invalid)
    expect(CityChannel.where(phone_number_id: attrs[:phone_number_id]).count).to eq(1)
  end
end
```

Em `spec/events/platform_event_payload_guard_spec.rb`, trocar:

```ruby
  R18_PLATFORM_EVENT_NAMES = %w[municipality.provisioned channel.token_rotated channel.unknown_seen operator.login operator.city_access].freeze
```

por:

```ruby
  R18_PLATFORM_EVENT_NAMES = %w[municipality.provisioned channel.registered channel.token_rotated channel.unknown_seen
                                operator.login operator.city_access].freeze
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/operators/cities_spec.rb spec/commands/municipality_channels/register_spec.rb spec/events/platform_event_payload_guard_spec.rb`
Expected: FAIL. Os motivos são rota `/cities` inexistente, `uninitialized constant MunicipalityChannels::Register` e
`/setup/municipalities` respondendo 501.

- [ ] **Step 3: Controller e rotas**

Criar `app/controllers/operators/cities_controller.rb`:

```ruby
# Provisionamento de cidade pelo console (spec banco-por-cidade §4, Plano 4):
#
#   POST /cities { slug, name, uf, ibge_code, admin_email, alert_email } → 202 { id }
#   GET  /cities/:id                                                     → 200 { id, slug, status, schema_version }
#
# O POST só registra e enfileira (ProvisionCity): quem cria o banco é o worker. A
# resposta é só o id — o token do convite nunca volta para o console, vai por
# e-mail para o primeiro municipal_admin.
module Operators
  class CitiesController < BaseController
    def create
      result = ProvisionCity.call(
        slug: params[:slug], name: params[:name], uf: params[:uf], ibge_code: params[:ibge_code],
        admin_email: params[:admin_email], alert_email: params[:alert_email], by: current_operator
      )
      return render(json: { id: result.payload[:city].id }, status: :accepted) if result.ok?

      status = result.reason == :city_exists ? :conflict : :unprocessable_entity
      render json: { error: result.reason.to_s, message: result.message }, status: status
    end

    def show
      city = City.find_by(id: params[:id].to_s)
      return head(:not_found) unless city

      render json: { id: city.id, slug: city.slug, status: city.status, schema_version: city.schema_version }
    end
  end
end
```

Em `config/routes.rb`, dentro do bloco `constraints(PlatformConsoleHost)`, trocar:

```ruby
      resources :city_grants, only: :create
```

por:

```ruby
      resources :city_grants, only: :create
      # Provisionamento em duas fases (Plano 4).
      resources :cities, only: %i[create show]
```

E remover a linha:

```ruby
    post "/municipalities",              to: "setup#provision_municipality"
```

- [ ] **Step 4: Aposentar `provision_municipality`**

Em `app/controllers/setup_controller.rb`, trocar:

```ruby
#   - provision_municipality é ação de operador sobre o catálogo, servida no
#     host de plataforma: pula a resolução de cidade. Sem o grant de operador
#     (Plano 3B) nem provisionamento de banco (Plano 4), responde 501 sem
#     tocar dado nenhum.
```

por:

```ruby
#   - provisionar cidade não é mais daqui: é POST /cities no console de
#     plataforma (Operators::CitiesController, Plano 4).
```

Trocar:

```ruby
class SetupController < ApplicationController
  skip_city_resolution only: %i[provision_municipality]

  include Authentication

  # provision_municipality não autentica porque não faz nada além de responder
  # 501: falta o grant de operador (Plano 3B) para autorizar a ação fora de
  # uma cidade.
  allow_unauthenticated_access only: %i[accept_invitation provision_municipality]
```

por:

```ruby
class SetupController < ApplicationController
  include Authentication

  allow_unauthenticated_access only: %i[accept_invitation]
```

Remover o bloco inteiro:

```ruby
  # POST /setup/municipalities — desligado até o Plano 3B (grant de operador
  # para agir sobre o catálogo) e o Plano 4 (POST /setup/cities, provisionamento
  # em duas fases). O command ProvisionMunicipality segue utilizável para uma
  # cidade já registrada e servível, fora do HTTP.
  def provision_municipality
    render json: {
      error: "provisioning_unavailable",
      message: "provisionamento de cidade passa para a plataforma (Planos 3B e 4)"
    }, status: :not_implemented
  end

```

Em `app/controllers/application_controller.rb`, trocar:

```ruby
#     cidade aplicam skip_city_resolution (Webhooks::WhatsappController,
#     SetupController).
```

por:

```ruby
#     cidade aplicam skip_city_resolution (Webhooks::WhatsappController).
```

Run: `git rm app/commands/provision_municipality.rb spec/commands/provision_municipality_spec.rb`

- [ ] **Step 5: Registro de canal**

Criar `app/commands/municipality_channels/register.rb`:

```ruby
# Registra o canal WhatsApp de uma cidade (CityChannel, na PLATAFORMA). Era parte
# do antigo ProvisionMunicipality; no provisionamento em duas fases (Plano 4) o
# canal é um passo à parte, feito quando a Meta libera o número. Só cidade ativa.
# Auditoria sem o token nem o telefone (ADR-0012/0013).
module MunicipalityChannels
  module Register
    def self.call(city:, phone_number_id:, waba_id:, display_phone_number:, access_token:)
      unless city.servable?
        return Result.fail(:city_not_servable, message: "cidade #{city.slug} não está ativa (status=#{city.status})")
      end
      return Result.fail(:invalid, message: "access_token vazio") if access_token.blank?

      channel = nil
      PlatformRecord.transaction do
        channel = CityChannel.create!(city: city, phone_number_id: phone_number_id, waba_id: waba_id,
                                      display_phone_number: display_phone_number, access_token: access_token,
                                      active: true)
        Platform.audit("channel.registered", city_id: city.id, phone_number_id: channel.phone_number_id)
      end
      Result.ok(channel: channel)
    rescue ActiveRecord::RecordInvalid => e
      Result.fail(:invalid, message: e.record.errors.full_messages.join(", "))
    rescue ActiveRecord::RecordNotUnique
      Result.fail(:invalid, message: "phone_number_id já registrado")
    end
  end
end
```

Em `lib/tasks/channels.rake`, antes do `end` final de `namespace :channels`, acrescentar:

```ruby
  desc "Register a city's WhatsApp channel. ENV: CITY_SLUG, PHONE_NUMBER_ID, WABA_ID, DISPLAY_PHONE_NUMBER, ACCESS_TOKEN"
  task register: :environment do
    city = City.find_by!(slug: ENV.fetch("CITY_SLUG"))
    result = MunicipalityChannels::Register.call(
      city: city, phone_number_id: ENV.fetch("PHONE_NUMBER_ID"), waba_id: ENV.fetch("WABA_ID"),
      display_phone_number: ENV.fetch("DISPLAY_PHONE_NUMBER"), access_token: ENV.fetch("ACCESS_TOKEN")
    )
    abort("[channels:register] failed: #{result.reason} #{result.message}") if result.failure?
    puts "[channels:register] #{city.slug} → phone_number_id=#{result.payload[:channel].phone_number_id}"
  end
```

- [ ] **Step 6: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/operators/cities_spec.rb spec/commands/municipality_channels spec/events/platform_event_payload_guard_spec.rb spec/requests/setup_accept_invitation_spec.rb`
Expected: PASS.

Run: `grep -rn "provision_municipality\|provisioning_unavailable\|ProvisionMunicipality\|setup/municipalities" app lib config spec`
Expected: nenhuma linha.

- [ ] **Step 7: Suíte inteira**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

- [ ] **Step 8: Commit**

```bash
git add app/controllers/operators/cities_controller.rb spec/requests/operators/cities_spec.rb config/routes.rb \
  app/controllers/setup_controller.rb app/controllers/application_controller.rb \
  app/commands/municipality_channels/register.rb spec/commands/municipality_channels/register_spec.rb \
  lib/tasks/channels.rake spec/events/platform_event_payload_guard_spec.rb
git commit -m "Provision cities from the platform console and retire the old setup endpoint

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

(O `git rm` do Step 4 já colocou as duas remoções no índice.)

---

### Task 7: Suspensão, retomada, backup e offboarding

**Files:**
- Create: `app/commands/city_lifecycle/suspend.rb`
- Create: `app/commands/city_lifecycle/resume.rb`
- Create: `app/commands/city_lifecycle/backup.rb`
- Create: `app/commands/city_lifecycle/offboard.rb`
- Create: `spec/commands/city_lifecycle/suspend_resume_spec.rb`
- Create: `spec/commands/city_lifecycle/backup_spec.rb`
- Create: `spec/commands/city_lifecycle/offboard_spec.rb`
- Modify: `lib/tasks/city.rake`
- Modify: `spec/tasks/city_rake_spec.rb`
- Modify: `spec/events/platform_event_payload_guard_spec.rb`
- Modify: `app/events/platform.rb` (comentário)

**Interfaces:**
- Consumes:
  - `CityConnection.forget` e `CityConnection.registered?`, `CityCatalog.reset_cache!`.
  - `CityDatabase.drop!` e `exists?` (Task 4).
  - `CitySchema.redact` (Task 1).
  - `ProvisionedCities` (Task 4), `ScratchDatabases` (Task 1), `CityGrants.issue`.
- Produces:
  - `CityLifecycle::Suspend.call(city:) -> Result` e `CityLifecycle::Resume.call(city:) -> Result`.
  - `CityLifecycle::Backup.call(city:, dir:) -> Result` (`payload[:path]`).
  - `CityLifecycle::Offboard.call(city:, backup_dir:) -> Result` (`payload[:city]`, `payload[:backup_path]`).
  - Motivos de falha: `:invalid_status`, `:backup_failed`, `:drop_failed`.
  - Rake `city:suspend[slug]`, `city:resume[slug]`, `city:backup[slug]` e `city:offboard[slug]` (este exige
    `CONFIRM=<slug>`). O diretório de backup vem de `CITY_BACKUP_DIR` (default `tmp/city_backups`).
  - Eventos `city.suspended`, `city.resumed`, `city.backed_up`, `city.archived`.

- [ ] **Step 1: Specs (falham)**

Criar `spec/commands/city_lifecycle/suspend_resume_spec.rb`:

```ruby
require "rails_helper"

# Suspender tira a cidade do ar sem apagar nada (spec banco-por-cidade §4). O 403
# no host está em spec/requests/city_resolution_spec.rb.
RSpec.describe "CityLifecycle::Suspend and CityLifecycle::Resume" do
  let(:city) { create(:city, status: "active", database_url: city_database_url("rota_saude_test_city_b")) }

  def events(name) = PlatformEvent.where(name: name).where("payload->>'city_id' = ?", city.id)

  it "suspends an active city, drops its pool in this process and audits" do
    CityConnection.ensure_pool(city)

    result = CityLifecycle::Suspend.call(city: city)

    expect(result.ok?).to be(true)
    expect(city.reload.status).to eq("suspended")
    expect(CityConnection.registered?(city.shard)).to be(false)
    expect(events("city.suspended").pluck(:payload)).to eq([ { "city_id" => city.id } ])
  end

  it "refuses to suspend a city that is not active, writing nothing" do
    city.update!(status: "provisioning")

    result = nil
    expect { result = CityLifecycle::Suspend.call(city: city) }.not_to change(PlatformEvent, :count)
    expect(result.reason).to eq(:invalid_status)
    expect(city.reload.status).to eq("provisioning")
  end

  it "resumes a suspended city and audits, and refuses a city that is not suspended" do
    city.update!(status: "suspended")

    expect(CityLifecycle::Resume.call(city: city).ok?).to be(true)
    expect(city.reload.status).to eq("active")
    expect(events("city.resumed").count).to eq(1)

    expect(CityLifecycle::Resume.call(city: city).reason).to eq(:invalid_status)
  end
end
```

Criar `spec/commands/city_lifecycle/backup_spec.rb`:

```ruby
require "rails_helper"
require "open3"
require "tmpdir"

# Backup é pg_dump por cidade, restaurável sozinho (spec banco-por-cidade §4).
RSpec.describe CityLifecycle::Backup do
  self.use_transactional_tests = false

  let!(:city) { provision_city! }
  let(:dir) { Dir.mktmpdir("city-backups") }
  let(:scratch) { ScratchDatabases.new_name }

  after do
    cleanup_provisioned_city!(city)
    ScratchDatabases.drop!(scratch)
    FileUtils.rm_rf(dir)
  end

  it "dumps one city into a file that restores alone into an empty database" do
    CityConnection.with(city) { User.create!(email_address: "servidora@cidade.gov.br", password: "secret123") }

    result = described_class.call(city: city, dir: dir)

    expect(result.ok?).to be(true)
    path = result.payload[:path]
    expect(File.basename(path)).to match(/\A#{city.slug}-\d{8}T\d{6}Z\.dump\z/)

    ScratchDatabases.create!(scratch)
    out, status = Open3.capture2e(
      { "PGPASSWORD" => ENV.fetch("POSTGRES_PASSWORD", "postgres") },
      "pg_restore", "--no-owner", "--no-acl", "--host", ENV.fetch("DATABASE_HOST", "127.0.0.1"),
      "--port", ENV.fetch("DATABASE_PORT", "5432"), "--username", "rota_saude", "--dbname", scratch, path
    )
    expect(status.success?).to be(true), out
    ScratchDatabases.superuser(scratch) do |conn|
      expect(conn.exec("SELECT email_address FROM users").column_values(0)).to eq([ "servidora@cidade.gov.br" ])
      expect(conn.exec("SELECT max(version) FROM schema_migrations").getvalue(0, 0)).to eq(CitySchema.expected_version.to_s)
    end
    expect(PlatformEvent.where(name: "city.backed_up").where("payload->>'city_id' = ?", city.id).pluck(:payload))
      .to eq([ { "city_id" => city.id, "file" => File.basename(path) } ])
  end

  it "fails without leaving a file or echoing the password when the database is unreachable" do
    ghost_slug = "provghost#{SecureRandom.hex(3)}"
    ghost = City.new(slug: ghost_slug, status: "active",
                     database_url: CityDatabase.url_for(slug: ghost_slug, password: "s3gr3d0s3gr3d0"))

    result = described_class.call(city: ghost, dir: dir)

    expect(result.reason).to eq(:backup_failed)
    expect(result.message).not_to include("s3gr3d0s3gr3d0")
    expect(Dir.children(dir)).to be_empty
  end

  it "refuses a city that is provisioning or archived" do
    %w[provisioning archived].each do |status|
      expect(described_class.call(city: City.new(slug: city.slug, status: status), dir: dir).reason).to eq(:invalid_status)
    end
  end
end
```

Criar `spec/commands/city_lifecycle/offboard_spec.rb`:

```ruby
require "rails_helper"
require "tmpdir"

# Offboarding (spec banco-por-cidade §4): suspensa → dump final → archived → DROP.
# Caso nomeado da spec: offboarding de A não altera nada em B.
RSpec.describe CityLifecycle::Offboard do
  self.use_transactional_tests = false

  let!(:city_a) { provision_city!(status: "suspended") }
  let!(:city_b) { provision_city! }
  let(:dir) { Dir.mktmpdir("city-backups") }

  after do
    [ city_a, city_b ].each { |city| cleanup_provisioned_city!(city) }
    FileUtils.rm_rf(dir)
  end

  def role_can_connect?(city)
    PG.connect(city.database_url).close
    true
  rescue PG::ConnectionBad
    false
  end

  def channel_for(city)
    CityChannel.create!(city: city, phone_number_id: "PN-#{SecureRandom.hex(4)}", waba_id: "WABA",
                        display_phone_number: "+55 41 90000-0000", access_token: "tok", active: true)
  end

  it "dumps, archives and drops A, leaving B untouched" do
    CityConnection.with(city_b) { User.create!(email_address: "b@cidade-b.gov.br", password: "secret123") }
    channel_a, channel_b = channel_for(city_a), channel_for(city_b)
    CityGrants.issue(city: city_a, kind: "operator", subject_id: SecureRandom.uuid)
    CityGrants.issue(city: city_b, kind: "operator", subject_id: SecureRandom.uuid)

    result = described_class.call(city: city_a, backup_dir: dir)

    expect(result.ok?).to be(true)
    expect(File.exist?(result.payload[:backup_path])).to be(true)
    expect(city_a.reload.status).to eq("archived")
    expect(CityDatabase.exists?(slug: city_a.slug)).to be(false)
    expect(role_can_connect?(city_a)).to be(false)
    expect(channel_a.reload.active).to be(false)
    expect(CityGrant.where(city_id: city_a.id)).to be_empty
    expect(PlatformEvent.where(name: "city.archived").where("payload->>'city_id' = ?", city_a.id).pluck(:payload))
      .to eq([ { "city_id" => city_a.id, "backup" => File.basename(result.payload[:backup_path]) } ])

    expect(city_b.reload.status).to eq("active")
    expect(CityDatabase.exists?(slug: city_b.slug)).to be(true)
    expect(role_can_connect?(city_b)).to be(true)
    expect(channel_b.reload.active).to be(true)
    expect(CityGrant.where(city_id: city_b.id).count).to eq(1)
    CityConnection.with(city_b) { expect(User.pluck(:email_address)).to eq([ "b@cidade-b.gov.br" ]) }
  end

  it "refuses a city that is not suspended, dropping and dumping nothing" do
    result = described_class.call(city: city_b, backup_dir: dir)

    expect(result.reason).to eq(:invalid_status)
    expect(CityDatabase.exists?(slug: city_b.slug)).to be(true)
    expect(Dir.children(dir)).to be_empty
  end

  it "changes nothing when the final dump fails" do
    allow(CityLifecycle::Backup).to receive(:call).and_return(Result.fail(:backup_failed, message: "disco cheio"))

    result = described_class.call(city: city_a, backup_dir: dir)

    expect(result.reason).to eq(:backup_failed)
    expect(city_a.reload.status).to eq("suspended")
    expect(CityDatabase.exists?(slug: city_a.slug)).to be(true)
  end

  it "only repeats the drop for a city already archived" do
    described_class.call(city: city_a, backup_dir: dir)
    expect(CityLifecycle::Backup).not_to receive(:call)

    result = described_class.call(city: city_a.reload, backup_dir: dir)

    expect(result.ok?).to be(true)
    expect(result.payload[:backup_path]).to be_nil
    expect(CityDatabase.exists?(slug: city_a.slug)).to be(false)
  end
end
```

Acrescentar ao fim de `spec/tasks/city_rake_spec.rb`:

```ruby
# Tasks de ciclo de vida (Plano 4). O comportamento está nos specs de
# CityLifecycle; aqui só o contrato — em especial a confirmação do offboarding.
RSpec.describe "city lifecycle rake tasks" do
  before(:all) do
    Rails.application.load_tasks unless Rake::Task.task_defined?("city:offboard")
  end

  before do
    %w[city:suspend city:resume city:backup city:offboard].each { |name| Rake::Task[name].reenable }
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:fetch).and_call_original
  end

  def invoke_silently(name, *args)
    original_stdout, $stdout = $stdout, StringIO.new
    original_stderr, $stderr = $stderr, StringIO.new
    Rake::Task[name].invoke(*args)
  ensure
    $stdout = original_stdout
    $stderr = original_stderr
  end

  it "city:offboard refuses to run without CONFIRM equal to the slug, calling nothing" do
    city = create(:city, status: "suspended")
    expect(CityLifecycle::Offboard).not_to receive(:call)

    [ nil, "outra-cidade" ].each do |confirm|
      Rake::Task["city:offboard"].reenable
      allow(ENV).to receive(:[]).with("CONFIRM").and_return(confirm)
      expect { invoke_silently("city:offboard", city.slug) }.to raise_error(SystemExit)
    end
  end

  it "city:offboard with CONFIRM=<slug> offboards into CITY_BACKUP_DIR" do
    city = create(:city, status: "suspended")
    allow(ENV).to receive(:[]).with("CONFIRM").and_return(city.slug)
    allow(ENV).to receive(:fetch).with("CITY_BACKUP_DIR").and_return("/tmp/city-backups-spec")
    expect(CityLifecycle::Offboard).to receive(:call).with(city: city, backup_dir: "/tmp/city-backups-spec")
      .and_return(Result.ok(city: city, backup_path: "/tmp/city-backups-spec/x.dump"))

    expect { invoke_silently("city:offboard", city.slug) }.not_to raise_error
  end

  it "city:suspend exits non-zero with the command's reason when it fails" do
    city = create(:city, status: "provisioning")

    expect { invoke_silently("city:suspend", city.slug) }.to raise_error(SystemExit) { |e| expect(e.status).not_to eq(0) }
    expect(city.reload.status).to eq("provisioning")
  end

  it "every lifecycle task aborts for an unknown slug" do
    %w[city:suspend city:resume city:backup city:offboard].each do |name|
      expect { invoke_silently(name, "naoexiste#{SecureRandom.hex(3)}") }.to raise_error(SystemExit)
    end
  end
end
```

Em `spec/events/platform_event_payload_guard_spec.rb`, trocar:

```ruby
  R18_PLATFORM_EVENT_NAMES = %w[municipality.provisioned channel.registered channel.token_rotated channel.unknown_seen
                                operator.login operator.city_access].freeze
```

por:

```ruby
  R18_PLATFORM_EVENT_NAMES = %w[municipality.provisioned city.suspended city.resumed city.backed_up city.archived
                                channel.registered channel.token_rotated channel.unknown_seen
                                operator.login operator.city_access].freeze
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/commands/city_lifecycle spec/tasks/city_rake_spec.rb spec/events/platform_event_payload_guard_spec.rb`
Expected: FAIL com `uninitialized constant CityLifecycle` e tasks `city:suspend`/`city:offboard` inexistentes.

- [ ] **Step 3: Commands**

Criar `app/commands/city_lifecycle/suspend.rb`:

```ruby
# Suspende uma cidade (spec banco-por-cidade §4): o resolver passa a responder 403
# city_suspended, EachCityJob deixa de rodá-la e CityScopedJob levanta
# CityNotServable. Nada é apagado; CityLifecycle::Resume desfaz.
#
# Este processo derruba o pool na hora. Os demais convergem pelo TTL do
# CityCatalog (30 s): até lá ainda servem a cidade.
module CityLifecycle
  module Suspend
    def self.call(city:)
      unless city.status == "active"
        return Result.fail(:invalid_status, message: "cidade #{city.slug} não está active (status=#{city.status})")
      end

      PlatformRecord.transaction do
        city.update!(status: "suspended")
        Platform.audit("city.suspended", city_id: city.id)
      end
      CityConnection.forget(city.shard)
      CityCatalog.reset_cache!
      Result.ok(city: city)
    end
  end
end
```

Criar `app/commands/city_lifecycle/resume.rb`:

```ruby
# Retoma uma cidade suspensa (spec banco-por-cidade §4). Se o schema dela ficou
# para trás durante a suspensão, o resolver responde 503 até city:migrate:all.
module CityLifecycle
  module Resume
    def self.call(city:)
      unless city.status == "suspended"
        return Result.fail(:invalid_status, message: "cidade #{city.slug} não está suspended (status=#{city.status})")
      end

      PlatformRecord.transaction do
        city.update!(status: "active")
        Platform.audit("city.resumed", city_id: city.id)
      end
      CityCatalog.reset_cache!
      Result.ok(city: city)
    end
  end
end
```

Criar `app/commands/city_lifecycle/backup.rb`:

```ruby
require "open3"

# Dump de UMA cidade (spec banco-por-cidade §4): pg_dump em formato custom, sem dono
# nem ACL, restaurável sozinho num banco vazio com pg_restore --no-owner.
#
# Conecta com o role da própria cidade (só enxerga o banco dela). A senha vai por
# PGPASSWORD, nunca na linha de comando. Os dados cifrados (AR encryption) seguem
# cifrados no dump: restaurar exige as chaves — por cidade no Plano 6.
module CityLifecycle
  module Backup
    STATUSES = %w[active suspended].freeze

    def self.call(city:, dir:)
      unless STATUSES.include?(city.status)
        return Result.fail(:invalid_status, message: "cidade #{city.slug} sem banco para dump (status=#{city.status})")
      end

      url = URI.parse(city.database_url)
      FileUtils.mkdir_p(dir)
      path = File.join(dir.to_s, "#{city.slug}-#{Time.current.utc.strftime('%Y%m%dT%H%M%SZ')}.dump")

      out, status = Open3.capture2e(
        { "PGPASSWORD" => URI::DEFAULT_PARSER.unescape(url.password.to_s) },
        "pg_dump", "--format=custom", "--no-owner", "--no-acl",
        "--host", url.host.to_s, "--port", (url.port || 5432).to_s,
        "--username", URI::DEFAULT_PARSER.unescape(url.user.to_s), "--dbname", url.path.delete_prefix("/"),
        "--file", path
      )
      unless status.success?
        FileUtils.rm_f(path)
        return Result.fail(:backup_failed, message: CitySchema.redact(out.lines.last(3).join).strip)
      end

      Platform.audit("city.backed_up", city_id: city.id, file: File.basename(path))
      Result.ok(path: path)
    end
  end
end
```

Criar `app/commands/city_lifecycle/offboard.rb`:

```ruby
# Desliga uma cidade (spec banco-por-cidade §4): suspensa → dump final → canais
# inativos e grants apagados → archived → DROP DATABASE e DROP ROLE.
#
# O dump vem antes de tudo: se falhar, nada muda. archived vem antes do drop: se
# o drop falhar, a cidade já não é servida nem migrada, e rodar de novo numa
# cidade archived só repete o drop (idempotente). Entregar o dump à prefeitura
# fica fora do sistema.
#
# Cidades de dev criadas por city:dev_up (banco do superusuário de bootstrap) não
# são apagáveis por rota_provisioner: o resultado é :drop_failed, com a cidade já
# archived e o banco intacto.
module CityLifecycle
  module Offboard
    def self.call(city:, backup_dir:)
      return drop(city, backup_path: nil) if city.status == "archived"

      unless city.status == "suspended"
        return Result.fail(:invalid_status, message: "cidade #{city.slug} precisa estar suspended (status=#{city.status})")
      end

      backup = Backup.call(city: city, dir: backup_dir)
      return backup if backup.failure?

      PlatformRecord.transaction do
        CityChannel.where(city_id: city.id).update_all(active: false, updated_at: Time.current)
        CityGrant.where(city_id: city.id).delete_all
        city.update!(status: "archived")
        Platform.audit("city.archived", city_id: city.id, backup: File.basename(backup.payload[:path]))
      end
      CityCatalog.reset_cache!

      drop(city, backup_path: backup.payload[:path])
    end

    def self.drop(city, backup_path:)
      CityConnection.forget(city.shard)
      CityDatabase.drop!(slug: city.slug)
      Result.ok(city: city, backup_path: backup_path)
    rescue PG::Error => e
      Result.fail(:drop_failed, message: CitySchema.redact(e.message))
    end
    private_class_method :drop
  end
end
```

- [ ] **Step 4: Tasks de rake**

Em `lib/tasks/city.rake`, antes do `end` final de `namespace :city`, acrescentar:

```ruby
  # Ciclo de vida depois de ativa (Plano 4). Sem endpoint: o console só ganha tela
  # no Plano 6. Em produção rodam no papel worker (kamal app exec --roles=worker),
  # que tem PROVISIONER_DATABASE_URL e o volume de CITY_BACKUP_DIR.
  lifecycle_city = lambda do |task_name, slug|
    abort "uso: rails '#{task_name}[slug]'" if slug.blank?
    City.find_by(slug: slug) || abort("[#{task_name}] cidade #{slug} não existe")
  end
  city_backup_dir = -> { ENV.fetch("CITY_BACKUP_DIR") { Rails.root.join("tmp/city_backups").to_s } }

  desc "Suspende uma cidade (o host dela responde 403). Uso: city:suspend[slug]"
  task :suspend, %i[slug] => :environment do |_t, args|
    city = lifecycle_city.call("city:suspend", args[:slug])
    result = CityLifecycle::Suspend.call(city: city)
    abort "[city:suspend] #{result.reason}: #{result.message}" if result.failure?
    puts "[city:suspend] #{city.slug} → suspended"
  end

  desc "Retoma uma cidade suspensa. Uso: city:resume[slug]"
  task :resume, %i[slug] => :environment do |_t, args|
    city = lifecycle_city.call("city:resume", args[:slug])
    result = CityLifecycle::Resume.call(city: city)
    abort "[city:resume] #{result.reason}: #{result.message}" if result.failure?
    puts "[city:resume] #{city.slug} → active"
  end

  desc "Dump de uma cidade em CITY_BACKUP_DIR (default tmp/city_backups). Uso: city:backup[slug]"
  task :backup, %i[slug] => :environment do |_t, args|
    city = lifecycle_city.call("city:backup", args[:slug])
    result = CityLifecycle::Backup.call(city: city, dir: city_backup_dir.call)
    abort "[city:backup] #{result.reason}: #{result.message}" if result.failure?
    puts "[city:backup] #{city.slug} → #{result.payload[:path]}"
  end

  desc "IRREVERSÍVEL: dump final, archived, DROP DATABASE e DROP ROLE de uma cidade suspensa. Uso: CONFIRM=<slug> city:offboard[slug]"
  task :offboard, %i[slug] => :environment do |_t, args|
    city = lifecycle_city.call("city:offboard", args[:slug])
    abort "[city:offboard] irreversível: confirme com CONFIRM=#{city.slug}" unless ENV["CONFIRM"] == city.slug

    result = CityLifecycle::Offboard.call(city: city, backup_dir: city_backup_dir.call)
    abort "[city:offboard] #{result.reason}: #{result.message}" if result.failure?
    puts "[city:offboard] #{city.slug} → archived; banco e role apagados; dump final: " \
         "#{result.payload[:backup_path] || 'feito na execução anterior'}"
  end
```

- [ ] **Step 5: Comentário de `Platform`**

Em `app/events/platform.rb`, trocar:

```ruby
# de PLATAFORMA. Só para eventos sobre objetos de plataforma: a cidade
# (municipality.provisioned), canais (channel.token_rotated,
# channel.unknown_seen) e operadores (operator.login). Eventos sobre usuários,
# memberships e convites de uma cidade usam DomainEvents.publish dentro da
# conexão da cidade (Ruling R18). PlatformEvent recusa payload com chave de dado
# pessoal.
```

por:

```ruby
# de PLATAFORMA. Só para eventos sobre objetos de plataforma: a cidade
# (municipality.provisioned, city.suspended, city.resumed, city.backed_up,
# city.archived), canais (channel.registered, channel.token_rotated,
# channel.unknown_seen) e operadores (operator.login, operator.city_access).
# Eventos sobre usuários, memberships e convites de uma cidade usam
# DomainEvents.publish dentro da conexão da cidade (Ruling R18). PlatformEvent
# recusa payload com chave de dado pessoal.
```

- [ ] **Step 6: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/commands/city_lifecycle spec/tasks/city_rake_spec.rb spec/events/platform_event_payload_guard_spec.rb`
Expected: PASS.

Run (leitura): `docker compose exec -T api bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -h $DATABASE_HOST -U rota_saude -d postgres -tA -c "select count(*) from pg_database where datname like '"'"'rota_saude_test_%prov%'"'"' or datname like '"'"'rota_saude_test_scratch_%'"'"'" -c "select count(*) from pg_roles where rolname like '"'"'rota_test_city_prov%'"'"'"'`
Expected: `0` e `0`.

- [ ] **Step 7: Suíte inteira**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

- [ ] **Step 8: Commit**

```bash
git add app/commands/city_lifecycle spec/commands/city_lifecycle lib/tasks/city.rake spec/tasks/city_rake_spec.rb \
  spec/events/platform_event_payload_guard_spec.rb app/events/platform.rb
git commit -m "Suspend, resume, back up and offboard a city

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 8: Purga de grants e sessões vencidos

**Files:**
- Create: `app/jobs/purge_platform_access_job.rb`
- Create: `app/jobs/purge_operator_city_sessions_job.rb`
- Create: `spec/jobs/purge_platform_access_job_spec.rb`
- Create: `spec/jobs/purge_operator_city_sessions_job_spec.rb`
- Modify: `config/recurring.yml`

**Interfaces:**
- Consumes:
  - `CityGrant`, `OperatorSession`, `OperatorAuthentication::PENDING_MFA_WINDOW` e `OPERATOR_SESSION_TTL`.
  - `Session::OPERATOR_GRANT_TTL`, `EachCityJob`.
- Produces:
  - `PurgePlatformAccessJob` (`RETENTION = 1.day`) e `PurgeOperatorCitySessionsJob`.
  - Tarefas recorrentes `purge_platform_access` e `purge_operator_city_sessions`.

- [ ] **Step 1: Specs (falham)**

Criar `spec/jobs/purge_platform_access_job_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe PurgePlatformAccessJob, type: :job do
  let(:city) { create(:city) }
  let(:operator) do
    Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end

  it "deletes grants expired for more than a day and keeps the others" do
    old    = CityGrant.create!(city: city, kind: "operator", subject_id: operator.id, expires_at: 25.hours.ago)
    recent = CityGrant.create!(city: city, kind: "operator", subject_id: operator.id, expires_at: 23.hours.ago)
    live   = CityGrant.create!(city: city, kind: "user", subject_id: SecureRandom.uuid, expires_at: 1.minute.from_now)

    described_class.perform_now

    expect(CityGrant.where(id: [ old, recent, live ].map(&:id)).pluck(:id)).to contain_exactly(recent.id, live.id)
  end

  it "deletes operator sessions that can no longer authenticate and keeps the usable ones" do
    stale_pending = operator.operator_sessions.create!(created_at: (OperatorAuthentication::PENDING_MFA_WINDOW + 1.minute).ago)
    fresh_pending = operator.operator_sessions.create!
    expired       = operator.operator_sessions.create!(mfa_verified_at: (OperatorAuthentication::OPERATOR_SESSION_TTL + 1.minute).ago)
    verified      = operator.operator_sessions.create!(mfa_verified_at: 1.hour.ago)

    described_class.perform_now

    expect(OperatorSession.where(id: [ stale_pending, fresh_pending, expired, verified ].map(&:id)).pluck(:id))
      .to contain_exactly(fresh_pending.id, verified.id)
  end
end
```

Criar `spec/jobs/purge_operator_city_sessions_job_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe PurgeOperatorCitySessionsJob, type: :job do
  before { Current.city = TEST_CITY_A }

  # EachCityJob está prepended: o corpo roda direto na conexão que o harness abriu,
  # como em purge_domain_events_job_spec.rb.
  def call_body
    described_class.instance_method(:perform).super_method.bind_call(described_class.new)
  end

  it "deletes operator grant sessions past their TTL and never touches user sessions" do
    user = User.create!(email_address: "u-#{SecureRandom.hex(3)}@cidade.gov.br", password: "secret123")
    old_user_session = user.sessions.create!(created_at: 30.days.ago)
    expired = Session.create!(operator_id: SecureRandom.uuid, created_at: (Session::OPERATOR_GRANT_TTL + 1.minute).ago)
    live    = Session.create!(operator_id: SecureRandom.uuid)

    call_body

    expect(Session.where(id: [ old_user_session, expired, live ].map(&:id)).pluck(:id))
      .to contain_exactly(old_user_session.id, live.id)
  end

  it "runs in every active city" do
    expect(described_class.ancestors.first).to eq(EachCityJob)
  end
end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/jobs/purge_platform_access_job_spec.rb spec/jobs/purge_operator_city_sessions_job_spec.rb`
Expected: FAIL com `uninitialized constant PurgePlatformAccessJob` / `PurgeOperatorCitySessionsJob`.

- [ ] **Step 3: Jobs**

Criar `app/jobs/purge_platform_access_job.rb`:

```ruby
# Purga diária do que expira na PLATAFORMA (Plano 4): grants de entrada vencidos há
# mais de RETENTION e sessões de operador que não autenticam mais — pendentes além
# de PENDING_MFA_WINDOW, verificadas além de OPERATOR_SESSION_TTL. A trilha de
# acesso fica em platform_events; estas linhas são só estado de curta duração.
class PurgePlatformAccessJob < ApplicationJob
  queue_as :housekeeping

  RETENTION = 1.day

  def perform
    now = Time.current
    CityGrant.where("expires_at < ?", now - RETENTION).delete_all
    OperatorSession.where(mfa_verified_at: nil)
                   .where("created_at < ?", now - OperatorAuthentication::PENDING_MFA_WINDOW).delete_all
    OperatorSession.where("mfa_verified_at < ?", now - OperatorAuthentication::OPERATOR_SESSION_TTL).delete_all
  end
end
```

Criar `app/jobs/purge_operator_city_sessions_job.rb`:

```ruby
# Purga, em cada cidade ativa, as sessões de operador abertas por grant que já
# passaram de Session::OPERATOR_GRANT_TTL (Plano 4) — não autenticam mais
# (Session#usable?). Sessão de usuário não é tocada.
class PurgeOperatorCitySessionsJob < ApplicationJob
  prepend EachCityJob
  queue_as :housekeeping

  def perform
    Session.where.not(operator_id: nil).where("created_at < ?", Session::OPERATOR_GRANT_TTL.ago).delete_all
  end
end
```

Em `config/recurring.yml`, dentro de `default: &default`, logo antes de `  resend_pending_alerts:`, inserir:

```yaml
  purge_platform_access:
    class: PurgePlatformAccessJob
    queue: housekeeping
    schedule: "every day at 3:30am America/Sao_Paulo"

  purge_operator_city_sessions:
    class: PurgeOperatorCitySessionsJob
    queue: housekeeping
    schedule: "every day at 3:45am America/Sao_Paulo"

```

- [ ] **Step 4: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/jobs/purge_platform_access_job_spec.rb spec/jobs/purge_operator_city_sessions_job_spec.rb`
Expected: PASS.

- [ ] **Step 5: Suíte inteira**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

- [ ] **Step 6: Commit**

```bash
git add app/jobs/purge_platform_access_job.rb app/jobs/purge_operator_city_sessions_job.rb \
  spec/jobs/purge_platform_access_job_spec.rb spec/jobs/purge_operator_city_sessions_job_spec.rb config/recurring.yml
git commit -m "Purge expired city grants and operator sessions daily

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 9: Cliente Postgres 16, deploy, documentação e prova em dev

**Files:**
- Modify: `Dockerfile`
- Modify: `deploy/production/deploy.yml`
- Modify: `deploy/development/deploy.yml`
- Modify: `deploy/production/secrets`
- Modify: `deploy/development/secrets`
- Modify: `deploy/SECRETS.md`
- Modify: `README.md`
- Modify: `../../start.sh` (raiz do monorepo, fora do git)

**Interfaces:**
- Consumes: tudo das Tasks 1–8.
- Produces:
  - Imagem com `pg_dump` 16.
  - `PROVISIONER_DATABASE_URL` só no papel worker.
  - `CITY_BACKUP_DIR` com volume.
  - Runbook de deploy (`bin/migrate` antes do `kamal deploy`).
  - Prova do ciclo completo em dev.

- [ ] **Step 1: Dockerfile**

Em `Dockerfile`, no stage `base`, trocar:

```dockerfile
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      curl \
      libjemalloc2 \
      libvips \
      postgresql-client \
      tzdata && \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*
```

por:

```dockerfile
# postgresql-client-16 do repositório PGDG (Plano 4): o pg_dump precisa ser da
# mesma versão major do servidor ou mais nova, o Postgres de produção é o 16 e o
# postgresql-client do Debian é o 15.
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y ca-certificates curl && \
    install -d /usr/share/postgresql-common/pgdg && \
    curl -fsSLo /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc https://www.postgresql.org/media/keys/ACCC4CF8.asc && \
    . /etc/os-release && \
    echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt ${VERSION_CODENAME}-pgdg main" \
      > /etc/apt/sources.list.d/pgdg.list && \
    apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      libjemalloc2 \
      libvips \
      postgresql-client-16 \
      tzdata && \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*
```

Run (raiz do monorepo): `docker build --target base -t rota-saude/api:pgclient-check apps/api`
Run: `docker run --rm --entrypoint pg_dump rota-saude/api:pgclient-check --version`
Expected: `pg_dump (PostgreSQL) 16.` seguido da versão menor.
Run: `docker image rm rota-saude/api:pgclient-check`

A imagem de dev (`rota-saude/api:dev`) NÃO é reconstruída nesta task. O `pg_dump` 15 dela atende o Postgres 15 do host;
um `start.sh --rebuild` futuro pega o cliente 16, que também faz dump do 15.

- [ ] **Step 2: Deploy (Kamal)**

Em `deploy/production/deploy.yml`, trocar:

```yaml
  worker:
    hosts:
      - worker1.rota-saude.example
    cmd: ./bin/jobs
    options:
      memory: 1g
```

por:

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
      volume: /var/lib/rota-saude/city_backups:/rails/storage/city_backups
```

E, em `env.clear`, depois de `GOVBR_ISSUER_URL: "https://sso.acesso.gov.br"`, acrescentar:

```yaml
    CITY_BACKUP_DIR: "/rails/storage/city_backups"
```

Em `deploy/development/deploy.yml`, trocar:

```yaml
  worker:
    hosts:
      - dev.rota-saude.example
    cmd: ./bin/jobs
    options:
      memory: 384m
```

por:

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
      volume: /var/lib/rota-saude/city_backups:/rails/storage/city_backups
```

E, em `env.clear`, depois de `GOVBR_ISSUER_URL: "https://sso.staging.acesso.gov.br"`, acrescentar:

```yaml
    CITY_BACKUP_DIR: "/rails/storage/city_backups"
```

Em `deploy/production/secrets` e `deploy/development/secrets`, logo depois da linha `ROTA_PLATFORM_PASSWORD=...` de cada
arquivo, acrescentar uma linha no MESMO formato dela, lendo o campo `provisioner_url` do item `postgres-roles`, e o
comentário acima dela:

```bash
# URL do papel rota_provisioner (CREATEDB/CREATEROLE, sem superusuário) — só o
# papel worker recebe (Plano 4).
PROVISIONER_DATABASE_URL=$(op read "op://${OP_VAULT}/postgres-roles/provisioner_url")
```

Se a linha `ROTA_PLATFORM_PASSWORD` do arquivo fechar a expressão de outro jeito, copie esse fechamento. Nunca imprima
valores desses arquivos.

- [ ] **Step 3: `deploy/SECRETS.md`**

Depois do item que começa com `` - `ROTA_PLATFORM_PASSWORD` `` (senha do papel `rota_platform`), acrescentar:

```markdown
- `PROVISIONER_DATABASE_URL` — `postgres://rota_provisioner:<senha>@<host>:5432/postgres`. Só o papel **worker** recebe:
  cria e apaga banco e role de cada cidade (Plano 4). Cada cidade provisionada ganha o role `rota_city_<slug>`, dono do
  banco `rota_saude_city_<slug>`, com senha gerada no provisionamento e guardada cifrada em `cities.database_url` —
  nenhuma senha de cidade entra no cofre.
```

Trocar `` `postgres-roles` (campos `rota_app`,
`rota_admin`, `rota_platform`) `` por `` `postgres-roles` (campos `rota_app`,
`rota_admin`, `rota_platform`, `provisioner_url`) ``. O trecho quebra linha exatamente entre `rota_app`, e `rota_admin`
no arquivo.

Antes de `## Rotação`, acrescentar:

```markdown
## Papel `rota_provisioner` (uma vez por cluster)

Criado pela infra, com o superusuário do Postgres, antes do primeiro provisionamento:

    CREATE ROLE rota_provisioner LOGIN CREATEDB CREATEROLE PASSWORD '<senha do cofre>';

Nunca `SUPERUSER`. Em dev e test, `rails platform:bootstrap` cria o mesmo papel com `ROTA_PROVISIONER_PASSWORD`
(default `rota_provisioner`).

## Backup e offboarding

`CITY_BACKUP_DIR` (volume do worker) recebe os dumps de `city:backup` e o dump final de `city:offboard`. O dump contém
dados cifrados com as chaves de AR Encryption acima: guardar o dump sem as chaves não permite restaurar.
```

- [ ] **Step 4: README**

Em `README.md`, na tabela de variáveis, depois da linha de `ROTA_PLATFORM_PASSWORD`, acrescentar:

```markdown
| `ROTA_PROVISIONER_PASSWORD` | `rota_provisioner` | `platform:bootstrap` (cria o papel) e `CityDatabase` em dev/test |
| `PROVISIONER_DATABASE_URL` | montada a partir da anterior | papel worker: cria e apaga banco/role de cidade (obrigatória em produção) |
| `CITY_BACKUP_DIR` | `tmp/city_backups` | `city:backup`, `city:offboard` |
```

Na tabela de bancos, depois da linha de `rota_saude_city_curitiba`, acrescentar:

```markdown
| `rota_saude_city_<slug>` (cidades provisionadas) | `rota_city_<slug>` | `ProvisionCityJob` (worker), a partir de `POST /cities` |
```

Imediatamente antes de `## Bootstrap do banco (do zero)`, acrescentar:

```markdown
## Ciclo de vida da cidade (Plano 4)

**Provisionar.** No console (`admin.*`, operador com TOTP):

- `POST /cities {slug, name, uf, ibge_code, admin_email, alert_email}` grava a cidade como `provisioning` e responde
  `202 {id}`.
- O worker (`ProvisionCityJob`) cria o role `rota_city_<slug>` e o banco dele, com `CONNECT` revogado de `PUBLIC`.
  Depois migra (subprocesso `city:migrate[slug]`), grava `city_profile`, o destinatário de alerta, o protocolo template
  em rascunho e o convite do primeiro `municipal_admin`, e marca a cidade `active`.
- O convite vai por e-mail (`?invite=<token>` no dashboard; a tela é do Plano 6).
- `GET /cities/:id` mostra o status. Repetir o POST com o mesmo slug retoma um provisionamento que falhou.
- O canal WhatsApp é outro passo: `CITY_SLUG=... PHONE_NUMBER_ID=... WABA_ID=... DISPLAY_PHONE_NUMBER=... ACCESS_TOKEN=... rails channels:register`.
- O provisionamento não semeia termo de consentimento.

**Migrar (deploy).** O boot NÃO migra. Com a imagem nova, antes de trocar o código em execução, rode `bin/migrate`
(`db:migrate` + `city:migrate:all`). Por exemplo: `kamal app exec --roles=worker --version=<nova> bin/migrate` e só
então `kamal deploy`.
- `city:migrate:all` migra toda cidade `active`/`suspended`, com lock por cidade, e sai com erro listando as que
  ficaram para trás.
- A cidade atrasada responde `503 city_schema_behind`; as outras seguem no ar.
- Toda migração destrutiva é expand/contract: o código antigo roda sobre o schema novo durante o deploy.
- Migração de cidade mora em `db/city_migrate/`, e `db/city_schema.rb` precisa acompanhar. O spec de paridade em
  `spec/services/city_schema_spec.rb` compara os dois.

**Suspender, backup, desligar** (rake; em produção no papel worker):

- `rails 'city:suspend[slug]'` → o host responde 403 em até 30 s. `rails 'city:resume[slug]'` desfaz.
- `rails 'city:backup[slug]'` → `pg_dump` da cidade em `CITY_BACKUP_DIR`, restaurável sozinho com
  `pg_restore --no-owner`.
- `CONFIRM=<slug> rails 'city:offboard[slug]'` (IRREVERSÍVEL, só cidade suspensa) → dump final, canais inativos,
  `archived`, `DROP DATABASE` e `DROP ROLE`.
- `curitiba` e `maringa` (criadas por `city:dev_up`, banco do superusuário) não são apagáveis pelo `rota_provisioner`.

**Purga.** Diariamente, `PurgePlatformAccessJob` apaga grants vencidos há mais de 1 dia e sessões de operador que não
autenticam mais. `PurgeOperatorCitySessionsJob` apaga, em cada cidade, as sessões de operador por grant além de 1 hora.

```

- [ ] **Step 5: `start.sh` (raiz do monorepo, fora do git)**

Em `<raiz-do-monorepo>/start.sh`, trocar:

```bash
# cria, carrega e ativa as duas cidades de dev (curitiba, maringa).
```

por:

```bash
# cria, carrega e ativa as duas cidades de dev (curitiba, maringa).
# platform:bootstrap também cria o papel rota_provisioner (Plano 4), e
# city:migrate:all aplica as migrations de cidade pendentes — o boot do api não
# migra mais.
```

E trocar:

```bash
docker compose run --rm --no-deps api ./bin/rails city:dev_baseline
```

por:

```bash
docker compose run --rm --no-deps api ./bin/rails city:dev_baseline
docker compose run --rm --no-deps api ./bin/rails city:migrate:all
```

NÃO rode o `start.sh`.

- [ ] **Step 6: Prova do ciclo completo em dev (cidade `londrina`)**

Run: `docker compose restart api worker`

Run (raiz do monorepo, UMA chamada de bash):

```bash
JAR="$(mktemp -d)/op.jar"; API=http://localhost:3030
CODE=$(docker compose exec -T api ./bin/rails runner 'print ROTP::TOTP.new(ENV.fetch("DEV_OPERATOR_OTP_SECRET", "TQLRHWIAKEISPIW6YY3IAKGCLVNPF4EV")).now' | tail -1)
SID=$(curl -s -c "$JAR" -b "$JAR" -H 'Host: admin.localhost' -H 'Content-Type: application/json' \
  -d '{"email_address":"dev@local","password":"dev-password"}' "$API/session" | sed -E 's/.*"session_id":"([^"]+)".*/\1/')
curl -s -o /dev/null -w "challenge %{http_code}\n" -c "$JAR" -b "$JAR" -H 'Host: admin.localhost' \
  -H 'Content-Type: application/json' -d "{\"session_id\":\"$SID\",\"code\":\"$CODE\"}" "$API/session/challenge"
BODY=$(curl -s -b "$JAR" -H 'Host: admin.localhost' -H 'Content-Type: application/json' -w ' %{http_code}' \
  -d '{"slug":"londrina","name":"Londrina","uf":"PR","ibge_code":"4113700","admin_email":"prefeita@londrina.demo","alert_email":"alertas@londrina.demo"}' \
  "$API/cities")
echo "POST /cities → $BODY"
ID=$(echo "$BODY" | sed -E 's/.*"id":"([^"]+)".*/\1/')
for i in $(seq 1 60); do
  STATUS=$(curl -s -b "$JAR" -H 'Host: admin.localhost' "$API/cities/$ID")
  echo "$STATUS" | grep -q '"status":"active"' && break
  sleep 2
done
echo "GET /cities/:id → $STATUS"
curl -s -o /dev/null -w "londrina overview %{http_code}\n" -H 'Host: londrina.localhost' "$API/admin/api/overview"
```

Expected:
- `challenge 200`.
- `POST /cities → {"id":"<uuid>"} 202`.
- `GET /cities/:id → {...,"status":"active","schema_version":"20260915000002"}`.
- `londrina overview 401`: servida, sem sessão.
- Se o status não chegar a `active`, `docker compose logs worker --since 5m` mostra o erro do `ProvisionCityJob`.

Run (leitura): `docker compose exec -T api bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -h $DATABASE_HOST -U rota_saude -d postgres -tA -c "select pg_get_userbyid(datdba), datacl from pg_database where datname = '"'"'rota_saude_city_londrina'"'"'"; PGPASSWORD=$POSTGRES_PASSWORD psql -h $DATABASE_HOST -U rota_saude -d rota_saude_city_londrina -tA -c "select name, uf, ibge_code from city_profile" -c "select email, role, invited_by_id is null from invitations" -c "select name, status from protocol_definitions"'`
Expected:
- `rota_city_londrina|{rota_city_londrina=CTc/rota_city_londrina}`: dono é o role da cidade, sem entrada de PUBLIC.
- `Londrina|PR|4113700`.
- `prefeita@londrina.demo|municipal_admin|t`.
- `triage-respiratoria|draft`.

Run: `docker compose exec -T api ./bin/rails 'city:suspend[londrina]'`
Aguarde 30 s (TTL do `CityCatalog` no processo do `api`).
Run: `curl -s -o /dev/null -w "%{http_code}\n" -H 'Host: londrina.localhost' http://localhost:3030/admin/api/overview`
Expected: `403`.

Run: `docker compose exec -T -e CONFIRM=londrina api ./bin/rails 'city:offboard[londrina]'`
Expected: `[city:offboard] londrina → archived; banco e role apagados; dump final: /rails/tmp/city_backups/londrina-<timestamp>.dump`.

Run (leitura): `docker compose exec -T api bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -h $DATABASE_HOST -U rota_saude -d postgres -tA -c "select count(*) from pg_database where datname = '"'"'rota_saude_city_londrina'"'"'" -c "select count(*) from pg_roles where rolname = '"'"'rota_city_londrina'"'"'"; PGPASSWORD=$POSTGRES_PASSWORD psql -h $DATABASE_HOST -U rota_saude -d rota_saude_platform_development -tA -c "select status from cities where slug = '"'"'londrina'"'"'" -c "select name from platform_events where payload->>'"'"'city_id'"'"' = (select id::text from cities where slug = '"'"'londrina'"'"') order by occurred_at"'`
Expected: `0`, `0`, `archived`, e os eventos `municipality.provisioned`, `city.suspended`, `city.backed_up`,
`city.archived`, nessa ordem.

Aguarde 30 s.
Run: `curl -s -o /dev/null -w "londrina %{http_code}\n" -H 'Host: londrina.localhost' http://localhost:3030/admin/api/overview; curl -s -o /dev/null -w "curitiba %{http_code}\n" -H 'Host: curitiba.localhost' http://localhost:3030/admin/api/overview`
Expected: `londrina 404`, `curitiba 401`.

Run: `docker compose exec -T api ./bin/rails city:migrate:all`
Expected: só `curitiba` e `maringa`, saída 0.

Run: `rm apps/api/tmp/city_backups/londrina-*.dump` (o dump da prova contém só dados de exemplo; `tmp/` é ignorado pelo
git).

Cole as saídas desta etapa no relatório da task.

- [ ] **Step 7: Suíte inteira e boot**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

Run: `docker compose exec -T api bash -c 'RAILS_ENV=development bin/rails runner "Rails.application.eager_load!; puts :eager_load_ok"'`
Expected: `eager_load_ok`.

Run: `curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3030/up`
Expected: `200`.

- [ ] **Step 8: Commit**

```bash
git add Dockerfile deploy/production/deploy.yml deploy/development/deploy.yml deploy/production/secrets \
  deploy/development/secrets deploy/SECRETS.md README.md
git commit -m "Ship pg_dump 16, the provisioner credential for workers and the city lifecycle runbook

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Definition of done

- O schema de cidade tem versão.
  - `db/city_migrate` define a versão esperada, e as migrations do zero produzem exatamente o dump (spec de paridade).
  - `load_schema` registra todas as versões.
  - `city:migrate[slug]` e `city:migrate:all` aplicam as migrations e gravam `cities.schema_version`.
  - `city:migrate:all` continua depois de uma falha e sai com erro listando as cidades atrasadas.
- A cidade atrasada responde `503 city_schema_behind` e as outras seguem servidas. O entrypoint não migra; `bin/migrate`
  é o passo de deploy.
- `city_profile` existe em toda cidade e o convite da plataforma é aceito sem `invited_by`. O rollout chegou a
  `curitiba`/`maringa` por `city:migrate:all`, sem apagar dados.
- `rota_provisioner` (sem superusuário) cria role e banco por cidade, com `CONNECT` revogado de `PUBLIC`. O role de uma
  cidade e o `rota_app` não abrem o banco de outra.
- Provisionamento em duas fases:
  - `POST /cities` → 202 `{id}`.
  - O worker cria, migra (subprocesso), semeia e ativa.
  - É idempotente e retomável, e não reenvia o convite.
  - Cidade em `provisioning` não é servida.
  - `POST /setup/municipalities` e `ProvisionMunicipality` não existem mais, e o canal WhatsApp tem
    `channels:register`.
- Suspensão, retomada, backup restaurável sozinho e offboarding (dump → archived → drop). Offboarding de A não altera B.
- Grants e sessões de operador vencidos são purgados diariamente.
- A imagem traz `pg_dump` 16, `PROVISIONER_DATABASE_URL` só no worker e o runbook está documentado.
- A prova em dev provisionou, suspendeu e desligou `londrina`; as saídas estão coladas no relatório.
- Suíte: 0 failures, **0 pending**.

## O que este plano NÃO faz

- Fila Solid Queue de plataforma e worker por cidade: gerente de supervisores, `queue.yml` enxuto, restart com backoff
  (Plano 5). Os jobs de ciclo de vida rodam na fila compartilhada até lá.
- Telas do console para provisionar, suspender e desligar; tela de aceite de convite (`?invite=`); CORS dinâmico; hosts
  por cidade nos frontends e no proxy do Kamal (Plano 6).
- Chave de cifra e de assinatura por cidade (Plano 6). O dump de uma cidade continua cifrado com as chaves globais.
- Termo de consentimento por cidade e a dívida de tipo da versão (`ConsentTerm.version` string × `Consent` numérico ×
  texto nas credentials).
- Reenvio de convite; entrega do dump final à prefeitura; backup agendado; task de restore.
- Migrar `curitiba`/`maringa` para role próprio (seguem com o superusuário de bootstrap).
- 503 para jobs de cidade atrasada (só o resolver HTTP recusa); guarda contra migração destrutiva sem expand/contract
  (é disciplina de review, não código).
- Automatizar `bin/migrate` em hook do Kamal; publicar `admin.*`, `auth.*` e hosts de cidade no proxy.
- Contratos fora do repo `apps/api` (`contracts/`, OpenAPI) que ainda citem `POST /setup/municipalities`.
- Política de descarte de job com `CityNotServable` (job de cidade suspensa segue falhando e reagendando) e a lista de
  cidades de dev repetida em `city.rake`, `seeds.rb` e `lib/dashboard_demo.rb` — dívidas herdadas, não tocadas aqui.

## Riscos

1. **Deploy não atômico.** Entre `bin/migrate` e a troca do código, o código antigo roda sobre o schema novo. O
   expand/contract é disciplina, não é imposto por código; uma migração destrutiva sem ele derruba a versão antiga
   durante o deploy.
2. **Versão pelo catálogo.** Uma migração aplicada fora das tasks não atualiza `cities.schema_version`, e a cidade fica
   em 503 até o próximo `city:migrate:all`. O inverso (catálogo adiantado em relação ao banco) só acontece com edição
   manual do catálogo.
3. **Provisionamento em dois bancos sem transação comum.** Os passos são idempotentes e a cidade só é servida quando
   `active`. Mesmo assim, apagar à mão uma linha `provisioning` do catálogo deixa banco e role órfãos, que precisam de
   `DROP` manual.
4. **Offboarding é irreversível.** O dump final é o único caminho de volta, e restaurá-lo exige as chaves de AR
   Encryption. A custódia do arquivo em `CITY_BACKUP_DIR` fica com a operação.
5. **Convergência de 30 s.** Suspensão e offboarding só tiram a cidade do ar nos outros processos quando o cache do
   `CityCatalog` expira; os pools ociosos desses processos só caem no restart.
6. **O role da cidade é dono do banco.** O runtime pode fazer DDL no próprio banco; uma injeção de SQL na cidade não
   alcança outra cidade, mas alcança o schema dela.
7. **O build depende do apt.postgresql.org** para o `postgresql-client-16`; uma indisponibilidade do PGDG quebra o build
   da imagem.
8. **Subprocesso de migração.** Ele herda o ambiente do worker; uma variável ausente só no subprocesso aparece como
   `CityMigrations::Subprocess::Failed` com o fim da saída, e o job reagenda até três vezes.
