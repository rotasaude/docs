# Banco por cidade — Plano 2: Desmontagem do multi-tenant e identidade na cidade

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reparentear o domínio e a identidade para a conexão da cidade, remover
a maquinaria de RLS por completo, e recriar o schema limpo sem `municipality_id`.

**Architecture:** `ApplicationRecord` passa a herdar de `CityRecord` e perde seu
`connects_to` — os ~24 modelos vêm junto sem serem tocados. O shard `bootstrap`
passa a apontar para um banco inexistente, de modo que qualquer query fora de
`CityConnection.with` levante em vez de cair no banco compartilhado. As tabelas
lidas antes de saber a cidade (canais, operadores, auditoria de plataforma)
migram para o banco de plataforma. O banco compartilhado perde o domínio e
sobrevive apenas como banco de fila e cache até o Plano 5.

**Tech Stack:** Rails 8.1.3, PostgreSQL 15, RSpec + FactoryBot, Docker Compose
(serviço `api`, container `api-dev`).

**Spec:** `docs/superpowers/specs/2026-09-12-banco-por-cidade-design.md`
**Plano anterior:** `docs/superpowers/plans/2026-09-12-banco-por-cidade-01-fundacao.md` (mergeado em `main`, commit `c61a3da`)

## Decisões que este plano carrega

Tomadas com o usuário em 2026-09-12, antes da escrita:

- **A identidade entra aqui, não num plano separado.** `User`, `Session`,
  `Membership` e `Invitation` herdam de `ApplicationRecord`; reparentear leva as
  quatro junto no mesmo commit. O Plano 3 original foi absorvido.
- **Janela vermelha aceita.** A Task 5 é um corte único e grande; a suíte só
  volta a verde no fim dela. A alternativa (verde a cada passo) exigiria código
  de compatibilidade descartável.
- **`/admin/api` mantém o caminho.** Passa a significar "API administrativa da
  cidade". Nenhum frontend é tocado neste plano.
- **Dev sobe duas cidades** (`curitiba` e `maringa`), para isolamento ser
  exercitável no navegador e não só na suíte.
- **Schema recriado limpo**, não migrations de remoção. Não há produção.

## Global Constraints

- **`connects_to` exatamente uma vez, no corpo de `CityRecord`.** Verificado em
  2026-09-12: `connected_to` só é permitido na classe abstrata que estabeleceu a
  conexão, e a classe do **meio** (`ApplicationRecord`, depois de reparenteada)
  levanta `NotImplementedError` se alguém chamar `connected_to` nela. Netos
  (`Conversation < ApplicationRecord < CityRecord`) roteiam corretamente pelo
  `connected_to` de `CityRecord` — medido.
- **`NotImplementedError` herda de `ScriptError`, não de `StandardError`.** Um
  `rescue => e` não o pega. Specs que esperam essa falha precisam de
  `rescue Exception` ou `expect { }.to raise_error(NotImplementedError)`.
- **O shard `bootstrap` deve falhar FECHADO.** Medido em 2026-09-12: hoje ele
  aponta para `:primary` e uma query solta lê `rota_saude_development` em
  silêncio. Apontando para um banco inexistente, a mesma query levanta
  `ActiveRecord::NoDatabaseError`, a cidade real continua funcionando, e o boot
  não quebra porque pools são preguiçosos.
- **O banco de plataforma nunca guarda dado de cidadão.** Canais, operadores,
  catálogo e auditoria de plataforma — nada de conversa, triagem ou relatório.
- **Todo comando roda dentro do container**, nunca no host:
  `docker compose exec -T api <cmd>` a partir da raiz do monorepo. O Ruby do host
  é 3.4.4; o app é 3.3.6. Passos que precisam do superuser levam
  `-e POSTGRES_PASSWORD=postgres`.
- **Mensagens de commit em inglês.**
- **Nenhum spec pode ser enfraquecido para passar.** Um spec que hoje prova
  isolamento por RLS deve passar a provar o mesmo por conexão — não sumir.

## Estado herdado do Plano 1

| Peça | Onde | Comportamento |
|---|---|---|
| `PlatformRecord` | `app/models/platform_record.rb` | abstrata, conexão fixa `:platform` |
| `City` | `app/models/city.rb` | `#shard`, `#servable?`, `encrypts :database_url`/`:encryption_key`, valida slug como rótulo DNS e rejeita reservados |
| `CityCatalog` | `app/models/city_catalog.rb` | `find_by_host`, cache com TTL 30s e teto de 500, não memoiza miss |
| `CityRecord` | `app/models/city_record.rb` | abstrata, **um** `connects_to shards: { bootstrap: { writing: :primary } }` |
| `CityConnection` | `app/models/city_connection.rb` | `.with`, `.ensure_pool`, `.registered?`, `.forget`, `InvalidCityDatabase` |
| `CityResolution` | `app/controllers/concerns/city_resolution.rb` | `around_action`, 404/403, resolve antes da auth |
| `CityProvisioner` | `app/commands/city_provisioner.rb` | devolve `Result`, só registra no catálogo |
| `city:create`, `city:test_databases` | `lib/tasks/city.rake` | dev-only, abortam fora de development |

Suíte atual: **358 examples, 0 failures**.

---

## File Structure

| Arquivo | O que acontece |
|---|---|
| `config/database.yml` | ganha `city_unset`; `primary`/`admin` viram apenas fila+cache |
| `app/models/city_record.rb` | shard `bootstrap` passa a apontar para `city_unset` |
| `app/models/application_record.rb` | perde `connects_to`, passa a herdar de `CityRecord` |
| `db/migrate/` | 33 migrations substituídas por uma inicial limpa |
| `db/platform_migrate/` | ganha canais, operadores, sessões de operador, eventos de plataforma |
| `db/structure.sql`, `lib/tasks/bootstrap.rake`, `bin/verify-bootstrap` | deletados |
| `lib/migration_helpers/rls.rb` | deletado |
| `app/controllers/concerns/tenant_scoped_request.rb` | deletado |
| `app/jobs/concerns/tenant_scoped_job.rb` | vira `city_scoped_job.rb` |
| `app/jobs/concerns/admin_role_job.rb` | deletado |
| `app/queries/admin/scoped.rb` | deletado |
| `app/models/current.rb` | perde `municipality_id` |
| `app/events/platform.rb` | passa a escrever em `platform_events` |
| `app/models/municipality.rb`, `municipality_channel.rb`, `unknown_channel.rb` | viram `PlatformRecord` ou somem |
| `db/seeds.rb`, `start.sh` | baseline de duas cidades |
| `spec/rails_helper.rb` | estabelece cidade de teste, embrulha cada exemplo |

---

### Task 1: Infra de teste multi-cidade

**Esta task vem primeiro de propósito.** Ela contém o único desconhecido técnico
que sobrou, e ele é da mesma família que já custou 205 falhas no Plano 1: como
`use_transactional_fixtures` se comporta quando cada exemplo roda dentro de
`connected_to(shard:)`. Se isso não funcionar, o formato do plano muda — melhor
descobrir agora.

**Files:**
- Modify: `spec/rails_helper.rb`
- Create: `spec/support/city_test_databases.rb`
- Modify: `lib/tasks/city.rake`
- Test: `spec/support/city_test_databases_spec.rb`

**Interfaces:**
- Consumes: `CityConnection.with`, `CityConnection.forget`, `City` (Plano 1).
- Produces: constante `TEST_CITY_A`/`TEST_CITY_B` (objetos `City` não persistidos
  apontando para `rota_saude_test_city_a`/`_b`); `around` global que roda cada
  exemplo dentro da conexão de `TEST_CITY_A`; helper `within_city(city) { }` para
  specs que precisam da outra.

- [ ] **Step 1: Write the failing test**

Create `spec/support/city_test_databases_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "City test database harness" do
  it "runs every example inside the default test city's connection" do
    expect(CityRecord.connection_db_config.database).to eq("rota_saude_test_city_a")
  end

  it "rolls back writes between examples" do
    expect(Probe.count).to eq(0)
    Probe.create!(label: "efêmero")
    expect(Probe.count).to eq(1)
  end

  it "rolls back writes between examples (second example proves the first rolled back)" do
    expect(Probe.count).to eq(0)
  end

  it "can switch to the second city with within_city" do
    within_city(TEST_CITY_B) do
      expect(CityRecord.connection_db_config.database).to eq("rota_saude_test_city_b")
    end
    expect(CityRecord.connection_db_config.database).to eq("rota_saude_test_city_a")
  end
end
```

`Probe` aqui é a tabela `probes` que `city:test_databases` já cria nos dois
bancos. Declare-a no arquivo de support, não no spec.

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec -T api bundle exec rspec spec/support/city_test_databases_spec.rb`
Expected: FAIL — `uninitialized constant TEST_CITY_B`, ou o banco reportado é
`rota_saude_test` (o compartilhado), não `rota_saude_test_city_a`.

- [ ] **Step 3: Create `spec/support/city_test_databases.rb`**

```ruby
# Harness de banco por cidade para a suíte.
#
# Cada exemplo roda dentro da conexão de uma cidade de teste. Specs que precisam
# de duas cidades (isolamento) usam `within_city`.
#
# Pré-requisito: rails city:test_databases
module CityTestDatabases
  def self.url_for(db)
    host = ENV.fetch("DATABASE_HOST", "127.0.0.1")
    port = ENV.fetch("DATABASE_PORT", "5432")
    pwd  = ENV.fetch("POSTGRES_PASSWORD", "postgres")
    "postgres://rota_saude:#{pwd}@#{host}:#{port}/#{db}"
  end

  def self.city(slug, db)
    City.new(slug: slug, name: slug.capitalize, status: "active",
             database_url: url_for(db), encryption_key: SecureRandom.hex(32))
  end

  def within_city(city, &block)
    CityConnection.with(city, &block)
  end
end

TEST_CITY_A = CityTestDatabases.city("testcitya", "rota_saude_test_city_a").freeze
TEST_CITY_B = CityTestDatabases.city("testcityb", "rota_saude_test_city_b").freeze

class Probe < CityRecord
  self.table_name = "probes"
end

RSpec.configure do |config|
  config.include CityTestDatabases

  config.around(:each) do |example|
    CityConnection.with(TEST_CITY_A) { example.run }
  end
end
```

- [ ] **Step 4: Require it from `spec/rails_helper.rb`**

Acrescente, depois do `require_relative "support/city_probe_controller"` que já existe:

```ruby
require_relative "support/city_test_databases"
```

- [ ] **Step 5: Run test to verify it passes**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/support/city_test_databases_spec.rb`
Expected: PASS, 4 examples.

**Se falhar por conta de transação:** `use_transactional_fixtures` abre a
transação antes do `around`, ou o `connected_to` troca a conexão por baixo da
transação já aberta. Os sintomas seriam rollback não acontecendo entre exemplos,
ou `ActiveRecord::ActiveRecordError` sobre conexão trocada dentro de transação.
**Pare e reporte** — não invente um workaround. Alternativas conhecidas, na ordem
em que eu tentaria: (a) trocar o `around(:each)` por `prepend_before`/`after` com
controle manual da transação; (b) `self.use_transactional_tests = false` só nos
specs de cidade, com limpeza por `delete_all`; (c) estabelecer a conexão da
cidade de teste como a `primary` do ambiente de test, tornando o `around`
desnecessário para a maioria dos specs.

- [ ] **Step 6: Run the whole suite**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: ainda 358 + 4 = 362, 0 failures. **Nada foi reparenteado ainda**, então
os specs existentes continuam usando `ApplicationRecord` no banco compartilhado;
o `around` só afeta `CityRecord`.

Se o `around` global quebrar specs existentes, isso é informação valiosa e você
deve reportá-la antes de seguir.

- [ ] **Step 7: Commit**

```bash
git add spec/rails_helper.rb spec/support/city_test_databases.rb spec/support/city_test_databases_spec.rb lib/tasks/city.rake
git commit -m "Add per-city test database harness"
```

---

### Task 2: Shard bootstrap falha fechado

**Files:**
- Modify: `config/database.yml`
- Modify: `app/models/city_record.rb`
- Test: `spec/models/city_record_spec.rb`

**Interfaces:**
- Consumes: `CityRecord` (Plano 1).
- Produces: config `city_unset` em development e test; `CityRecord` cujo shard
  `bootstrap` aponta para ela.

- [ ] **Step 1: Write the failing test**

Create `spec/models/city_record_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe CityRecord do
  it "fails closed outside a city connection" do
    expect {
      CityRecord.connected_to(shard: :bootstrap, role: :writing) { Probe.count }
    }.to raise_error(ActiveRecord::NoDatabaseError, /no_city_selected/)
  end

  it "still serves a real city" do
    within_city(TEST_CITY_B) do
      expect(CityRecord.connection_db_config.database).to eq("rota_saude_test_city_b")
    end
  end

  it "declares connects_to exactly once" do
    source = File.read(Rails.root.join("app/models/city_record.rb"), encoding: "UTF-8")
    expect(source.scan(/^\s*connects_to\b/).size).to eq(1)
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_record_spec.rb`
Expected: FAIL no primeiro exemplo — hoje a query no shard `bootstrap` **tem
sucesso** contra o banco compartilhado, que é precisamente o defeito.

- [ ] **Step 3: Add the `city_unset` config**

Em `config/database.yml`, dentro de `development:` e de `test:`, acrescente:

```yaml
  # Destino do shard `bootstrap` do CityRecord. Aponta DE PROPÓSITO para um
  # banco que não existe: qualquer query fora de CityConnection.with levanta
  # ActiveRecord::NoDatabaseError em vez de cair no banco compartilhado.
  # Verificado em 2026-09-12 — pools são preguiçosos, então isto não quebra o boot.
  city_unset:
    <<: *default
    database: rota_saude_no_city_selected
    username: rota_app
    password: <%= ENV.fetch("ROTA_APP_PASSWORD", "rota_app") %>
    host: <%= ENV.fetch("DATABASE_HOST", "127.0.0.1") %>
    port: <%= ENV.fetch("DATABASE_PORT", 5432) %>
    database_tasks: false
```

`database_tasks: false` impede que `db:migrate` e `db:prepare` tentem criar ou
migrar esse banco fantasma.

Em `production:`, acrescente a mesma entrada com `url:` seguindo a forma das
outras entradas de produção, e `database_tasks: false`.

- [ ] **Step 4: Point the bootstrap shard at it**

Em `app/models/city_record.rb`, troque a linha do `connects_to`:

```ruby
  connects_to shards: { bootstrap: { writing: :city_unset } }
```

E acrescente ao comentário do arquivo, acima dela:

```ruby
# O shard :bootstrap aponta para `city_unset`, um banco que NÃO EXISTE. Isso é
# deliberado: sem cidade selecionada, qualquer query levanta NoDatabaseError em
# vez de ler o banco compartilhado em silêncio. Antes disto ele apontava para
# :primary e falhava ABERTO.
```

- [ ] **Step 5: Run test to verify it passes**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_record_spec.rb`
Expected: PASS, 3 examples.

- [ ] **Step 6: Run the whole suite**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 362, 0 failures.

- [ ] **Step 7: Commit**

```bash
git add config/database.yml app/models/city_record.rb spec/models/city_record_spec.rb
git commit -m "Make the bootstrap shard fail closed"
```

---

### Task 3: Tabelas de plataforma que faltam

O corte da Task 5 torna inalcançável tudo que é lido **antes** de saber a cidade.
Estas tabelas precisam existir na plataforma antes, ou o corte quebra o webhook e
o login de operador.

**Files:**
- Create: `db/platform_migrate/20260913000001_create_platform_tables.rb`
- Create: `app/models/city_channel.rb`, `app/models/unknown_channel.rb`
- Create: `app/models/operator.rb`, `app/models/operator_session.rb`
- Create: `app/models/platform_event.rb`
- Test: `spec/models/city_channel_spec.rb`, `spec/models/operator_spec.rb`

**Interfaces:**
- Consumes: `PlatformRecord`, `City` (Plano 1).
- Produces: `CityChannel.active.find_by(phone_number_id:)` → registro com
  `city_id`; `UnknownChannel.record!(phone_number_id:, change:)`;
  `Operator` com autenticação e MFA; `PlatformEvent.create!`.

- [ ] **Step 1: Write the failing tests**

Create `spec/models/city_channel_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe CityChannel do
  let(:city) { City.create!(slug: "canaltest", name: "Canal", status: "active",
                            database_url: CityTestDatabases.url_for("rota_saude_test_city_a"),
                            encryption_key: SecureRandom.hex(32)) }

  it "routes a phone_number_id to its city without a city connection" do
    described_class.create!(city: city, phone_number_id: "pn-1", waba_id: "w-1",
                            display_phone_number: "+55 41 0000-0000",
                            access_token: "segredo", active: true)

    found = described_class.active.find_by(phone_number_id: "pn-1")
    expect(found.city_id).to eq(city.id)
  end

  it "encrypts the access token at rest" do
    channel = described_class.create!(city: city, phone_number_id: "pn-2", waba_id: "w-2",
                                      display_phone_number: "+55 41 0000-0001",
                                      access_token: "segredo", active: true)
    raw = described_class.connection.select_value(
      described_class.sanitize_sql(["SELECT access_token FROM city_channels WHERE id = ?", channel.id])
    )
    expect(raw).not_to eq("segredo")
    expect(described_class.find(channel.id).access_token).to eq("segredo")
  end

  it "lives in the platform database, not a city database" do
    expect(described_class.connection_db_config.database).to match(/platform/)
  end
end
```

- [ ] **Step 2: Run to verify failure**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_channel_spec.rb`
Expected: FAIL — `uninitialized constant CityChannel`.

- [ ] **Step 3: Write the migration**

Create `db/platform_migrate/20260913000001_create_platform_tables.rb`. As colunas
vêm das tabelas existentes no banco compartilhado — leia
`db/structure.sql` para `municipality_channels`, `unknown_channels`, `users`,
`sessions` e `domain_events` antes de escrever, e preserve tipos, índices e
constraints. Renomeie `municipality_id` para `city_id` em `city_channels`.

`operators` e `operator_sessions` espelham `users` e `sessions`, menos o que é
específico de município. `platform_events` espelha `domain_events` sem a coluna
de tenant.

- [ ] **Step 4: Write the models**

Cinco modelos, todos herdando de `PlatformRecord`. `CityChannel` leva
`encrypts :access_token` (o `MunicipalityChannel` atual já faz isso —
`app/models/municipality_channel.rb:5`). `Operator` leva a mesma autenticação e
MFA que `User` tem hoje; leia `app/models/user.rb` antes de escrever.

- [ ] **Step 5: Migrate and run the tests**

```bash
docker compose exec -T api bin/rails db:migrate:platform
docker compose exec -T -e RAILS_ENV=test api bin/rails db:migrate:platform
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_channel_spec.rb spec/models/operator_spec.rb
```

Expected: PASS.

- [ ] **Step 6: Run the whole suite and commit**

A suíte deve continuar verde — nada consome esses modelos ainda.

```bash
git add db/platform_migrate app/models/city_channel.rb app/models/unknown_channel.rb app/models/operator.rb app/models/operator_session.rb app/models/platform_event.rb db/platform_schema.rb spec/models/city_channel_spec.rb spec/models/operator_spec.rb
git commit -m "Add platform tables for channels, operators and audit"
```

---

### Task 4: Schema inicial limpo

**Files:**
- Delete: `db/migrate/*` (33 arquivos), `db/structure.sql`, `lib/tasks/bootstrap.rake`, `bin/verify-bootstrap`, `lib/migration_helpers/rls.rb`
- Create: `db/migrate/20260913000010_create_city_schema.rb`
- Modify: `config/application.rb` (remove o `require_relative "../lib/migration_helpers/rls"`)

**Interfaces:**
- Produces: um schema de cidade sem `municipality_id`, sem RLS, sem split de
  ownership — carregável por `db:schema:load` em qualquer banco de cidade.

O que entra: as 16 tabelas de domínio e identidade, **menos** a coluna
`municipality_id` em todas elas, menos `municipalities` (virou `cities` na
plataforma), menos `municipality_channels` e `unknown_channels` (viraram
plataforma), menos `authors` se ele for redundante com `users` — verifique.

O que **não** entra: `solid_queue_*` e `solid_cache_entries`. Essas continuam no
banco compartilhado até o Plano 5.

- [ ] **Step 1: Inventory before deleting**

```bash
docker compose exec -T api bin/rails runner 'puts ActiveRecord::Base.connection.tables.sort'
```

Guarde a saída no relatório. Ela é a lista de verificação do que o novo schema
precisa reproduzir.

- [ ] **Step 2: Write the new migration**

Uma migration só, `create_city_schema`. Derive cada `create_table` do
`db/structure.sql` atual (leia antes de deletar), removendo `municipality_id` e
suas FKs e índices. Preserve os 15 check constraints e as duas extensões
(`citext`, `pgcrypto`).

Índices que citavam `municipality_id` precisam ser repensados, não só
desprovidos da coluna: `conversations` tinha índice parcial por tenant + telefone;
sem tenant, a unicidade passa a ser só por telefone.

- [ ] **Step 3: Delete the old machinery**

```bash
git rm -r db/migrate db/structure.sql lib/tasks/bootstrap.rake bin/verify-bootstrap lib/migration_helpers/rls.rb
```

Depois recrie `db/migrate/` com a migration nova. E remova de
`config/application.rb:19` a linha `require_relative "../lib/migration_helpers/rls"`.

- [ ] **Step 4: Rebuild the two test city databases from the new schema**

Estenda `city:test_databases` para carregar o schema em vez de só criar a tabela
`probes`, mantendo `probes` (o harness da Task 1 depende dela).

```bash
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bin/rails city:test_databases
```

- [ ] **Step 5: Verify the schema loads clean**

```bash
docker compose exec -T api bin/rails runner 'puts CityRecord.connection.tables.sort' # dentro de uma cidade
```

Expected: as tabelas de domínio, nenhuma com `municipality_id`. Confirme com:

```bash
docker compose exec -T api bin/rails runner '
  CityConnection.with(City.first) do
    cols = CityRecord.connection.tables.flat_map { |t| CityRecord.connection.columns(t).map(&:name) }
    puts cols.include?("municipality_id") ? "AINDA TEM municipality_id" : "limpo"
  end'
```

- [ ] **Step 6: Commit**

A suíte estará vermelha a partir daqui — é a janela aceita. Commite mesmo assim,
para que o corte da Task 5 tenha uma base legível.

```bash
git add -A db/ lib/ bin/ config/application.rb
git commit -m "Replace the tenant schema with a clean per-city schema"
```

---

### Task 5: O corte

**Esta é a task da janela vermelha.** Ela reparenteia, remove a maquinaria de
tenant e ajusta todo o código que dependia dela. A suíte fica vermelha do começo
ao fim e só volta a verde no Step final.

**Files:**
- Modify: `app/models/application_record.rb`, `app/models/current.rb`
- Delete: `app/controllers/concerns/tenant_scoped_request.rb`, `app/jobs/concerns/admin_role_job.rb`, `app/queries/admin/scoped.rb`
- Rename: `app/jobs/concerns/tenant_scoped_job.rb` → `city_scoped_job.rb`
- Modify: `app/events/domain_events.rb`, `app/events/platform.rb`, `app/models/conversation.rb`, `app/services/whatsapp/ingest.rb`, `app/controllers/application_controller.rb`, `app/controllers/admin/api/base_controller.rb`, as 16 queries de `app/queries/admin/`, os ~20 jobs, os commands
- Modify: ~41 arquivos de spec

**Interfaces:**
- Consumes: tudo das Tasks 1-4.
- Produces: `ApplicationRecord < CityRecord` sem `connects_to`; `CityScopedJob#with_city(slug)`; `DomainEvents.publish` sem tenant.

- [ ] **Step 1: Reparent `ApplicationRecord`**

```ruby
# Base de todo modelo de DOMÍNIO e de IDENTIDADE de uma cidade.
#
# Não declara connects_to: a conexão é a da cidade resolvida em runtime, e quem
# a estabelece é CityRecord. Chamar `connected_to` NESTA classe levanta
# NotImplementedError — só a connection class pode.
class ApplicationRecord < CityRecord
  self.abstract_class = true
end
```

- [ ] **Step 2: Strip the tenant from `Current`**

```ruby
# CurrentAttributes resetado por request e por job.
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  # Cidade resolvida pelo host. Serve para log e para o envelope de resposta.
  # NUNCA use em WHERE: o escopo é a conexão, não um valor de coluna.
  attribute :city

  delegate :user, to: :session, allow_nil: true
end
```

- [ ] **Step 3: Delete the tenant machinery**

```bash
git rm app/controllers/concerns/tenant_scoped_request.rb \
      app/jobs/concerns/admin_role_job.rb \
      app/queries/admin/scoped.rb \
      spec/rls/tenant_isolation_spec.rb \
      spec/support/admin_rls.rb
git mv app/jobs/concerns/tenant_scoped_job.rb app/jobs/concerns/city_scoped_job.rb
```

`spec/rls/tenant_isolation_spec.rb` é aposentado porque
`spec/cities/city_isolation_spec.rb` (Plano 1) já prova os mesmos invariantes
pelo novo mecanismo. **Confirme isso lendo os dois antes de deletar** — se o
antigo provar algo que o novo não prova, o novo ganha um exemplo primeiro.

- [ ] **Step 4: Rewrite `CityScopedJob`**

```ruby
# Wrapper de cidade para jobs. Todo job que toca dado de domínio inclui isto.
# Falha fechada: sem cidade, levanta — não vaza.
#
# O job carrega o SLUG, não o objeto: o payload viaja pela fila, que ainda vive
# no banco compartilhado até o Plano 5.
module CityScopedJob
  extend ActiveSupport::Concern

  class CityMissing < StandardError; end

  private

  def with_city(slug)
    raise CityMissing, "#{self.class.name}: slug nulo" if slug.blank?

    city = City.find_by(slug: slug)
    raise CityMissing, "#{self.class.name}: cidade #{slug} não existe" if city.nil?

    Current.city = city
    CityConnection.with(city) { yield }
  end
end
```

- [ ] **Step 5: Strip the tenant from the event publisher**

`DomainEvents.publish` perde `municipality_id` e a exceção `TenantMissing`; o
payload enfileirado troca `municipality_id:` por `city_slug:`. `Platform.audit`
passa a escrever em `PlatformEvent` em vez de `DomainEvent` com tenant nulo.

- [ ] **Step 6: Sweep the rest**

Use a lista para não perder nada:

```bash
grep -rln "municipality_id\|Current.municipality_id\|with_tenant\|TenantScoped\|AdminRoleJob\|Admin::Scoped\|connected_to(role: :admin)" app lib config
```

Cada arquivo dessa lista precisa de decisão. Padrões:
- `where(municipality_id: ...)` → remova a condição inteira.
- `connected_to(role: :admin) { ... }` → remova o wrapper; não há mais role admin.
- `Admin::Scoped.triages(muni)` → `Triage.all`.
- `Conversation.for(phone, municipality_id:)` → `Conversation.for(phone)`.
- `Whatsapp::Ingest.route` → resolve por `CityChannel` na plataforma, depois
  `CityConnection.with`.
- `Admin::Api::BaseController` → perde `skip_tenant_scope`,
  `with_admin_connection`, `resolve_municipality`, `cross_tenant?`,
  `first_member_municipality`, `municipality_filter`; ganha `include CityResolution`.
- `ApplicationController` → troca `include TenantScopedRequest` por
  `include CityResolution`.

- [ ] **Step 7: Sweep the specs**

```bash
grep -rln "municipality_id\|SET LOCAL\|as_admin\|clean_admin_tables\|municipality:" spec
```

São ~41 arquivos. A maioria fica **mais simples**: some o `SET LOCAL` manual,
some o `use_transactional_tests = false`, some o `as_admin`. Factories perdem
`municipality`.

**Nenhum spec pode ser deletado para a suíte fechar.** Um spec que prova
comportamento de domínio continua valendo; só o andaime de tenant sai.

- [ ] **Step 8: Green**

```bash
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
```

Expected: 0 failures. O número de exemplos vai **cair** — os que só existiam para
provar RLS somem com ela. Reporte o número antes e depois, e justifique a queda.

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "Reparent the domain to per-city connections and remove RLS"
```

---

---

## As duas seções abaixo são ESBOÇO, não plano

Tasks 1 a 5 estão em granularidade executável. As duas seguintes **não estão**, e
não devem ser despachadas como as outras.

O motivo é concreto, não preguiça: o que o login por subdomínio precisa fazer
depende da forma que `Authentication`, `SessionsController` e as factories
tiverem **depois** do corte da Task 5. Escrever código de plano em cima de um
estado que ainda não existe produziria exatamente as suposições erradas que a
execução do Plano 1 encontrou quatro vezes.

**Promova estas duas a um Plano 3 escrito depois que a Task 5 fechar verde**,
com o código real em mãos. Ficam aqui como escopo reservado, para que ninguém
suponha que o Plano 2 entrega login funcionando.

### Esboço A: Login por subdomínio

**Files:**
- Modify: `app/controllers/sessions_controller.rb`, `app/controllers/concerns/authentication.rb`
- Create: `app/controllers/platform/sessions_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/city_login_spec.rb`

Com a identidade dentro do banco da cidade, `Session.find_by(id:)` passa a
consultar a cidade do host — a inversão que torna o cookie de uma cidade inútil
na vizinha. O operador autentica num controller separado, contra `Operator` na
plataforma.

- [ ] **Step 1: Write the failing test**

```ruby
require "rails_helper"

RSpec.describe "Login por subdomínio", type: :request do
  it "a sessão criada na cidade A não autentica na cidade B" do
    # cria usuário em A, loga, e usa o cookie contra o host de B
    # espera 401 — sem nenhuma verificação de aplicação, só a conexão
  end

  it "o cookie de sessão sai sem atributo domain" do
    # host-only: não viaja para outro subdomínio
  end
end
```

Escreva os dois exemplos por extenso, usando o harness da Task 1 e o
`CityResolution` do Plano 1. O segundo exemplo é a guarda de regressão da
propriedade que o Plano 1 declarou inegociável.

- [ ] **Steps 2-6:** implemente, rode, commite. O `SessionsController` atual
  (`app/controllers/sessions_controller.rb`) já tem o fluxo de senha + MFA; o que
  muda é **onde** o usuário é procurado, não como. O caminho de operador sai dele
  para `Platform::SessionsController`.

---

### Esboço B: Baseline de duas cidades

**Files:**
- Modify: `db/seeds.rb`, `start.sh`
- Create: `db/seeds/city_demo.rb`

`db/seeds.rb` hoje (121 linhas) cria um município, dois usuários e uma demo
ponta-a-ponta. Passa a: criar **duas** cidades no catálogo, provisionar seus
bancos, e rodar o seed de demonstração dentro de cada uma. O operador
`dev@local` migra para `Operator` na plataforma, com o mesmo `otp_secret` fixo —
esse detalhe existe para o autenticador do dev sobreviver a resets e deve ser
preservado.

`start.sh` perde a chamada a `db:bootstrap` (que não existe mais) e ganha a
criação dos dois bancos de cidade.

- [ ] **Steps:** escreva o seed, rode `start.sh --reset`, prove que as duas
  cidades sobem e que `curitiba.localhost:5175` e `maringa.localhost:5175`
  respondem com dados diferentes. Commite.

---

## Definition of done

- `ApplicationRecord` herda de `CityRecord` e não declara `connects_to`.
- Uma query fora de `CityConnection.with` levanta `NoDatabaseError`.
- `grep -rn "municipality_id" app lib config db/migrate` não retorna nada.
- Não existe mais `rota_app`/`rota_admin` como papéis distintos no código.
- A suíte fecha verde, e a queda no número de exemplos está justificada.
- `start.sh --reset` sobe duas cidades com dados distintos.
- O banco compartilhado contém apenas `solid_queue_*` e `solid_cache_entries`.

## O que este plano NÃO faz

- Não sobe worker por cidade — a fila continua compartilhada (Plano 5).
- Não implementa provisionamento em duas fases nem `city:migrate:all` (Plano 4).
- Não toca os três frontends, CORS, nem `ReportSnapshot#url` (Plano 6).
- Não entrega login por subdomínio funcionando (esboço A, vira Plano 3)
- Não implementa o grant assinado de operador para entrar numa cidade (Plano 6).
- Não implementa chave de cifra por cidade em uso — a coluna existe desde o
  Plano 1, mas ninguém a consome ainda (Plano 6).

## Riscos

1. **Task 1 é o desconhecido.** Se `use_transactional_fixtures` não conviver com
   `connected_to` por exemplo, o formato da suíte muda e as Tasks 5-7 mudam com
   ele. Por isso ela é a primeira.
2. **A Task 5 é grande demais para uma review confortável.** É o preço da janela
   vermelha, escolhido conscientemente. Mitigação: os Steps 1-5 são mecânicos e
   verificáveis isoladamente; o Step 6 é onde mora o julgamento.
3. **`authors` pode ser redundante com `users`.** Se for, some na Task 4 e o
   `Author`-header de autenticação de protocolo precisa de outro caminho. Decida
   na Task 4, não na 5.
4. **A fila compartilhada carrega dado de cidadão até o Plano 5.** É um estado
   transitório conhecido e aceito, não um descuido — mas não deve chegar a
   produção assim.
