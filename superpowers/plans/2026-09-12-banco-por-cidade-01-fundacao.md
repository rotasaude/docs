# Banco por cidade — Plano 1: Fundação (catálogo e resolução)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir o banco de plataforma, o catálogo de cidades e o mecanismo que
resolve `request.host` → conexão do banco daquela cidade, sem alterar o
comportamento atual do app.

**Architecture:** Uma classe abstrata `PlatformRecord` liga-se a um banco de
plataforma fixo que guarda o catálogo (`cities`). Uma segunda abstrata,
`CityRecord`, não tem conexão fixa: `CityConnection` registra um pool por cidade
sob demanda via `connection_handler.establish_connection` e executa blocos dentro
de `connected_to(shard:)`. Um concern de controller resolve o host contra o
catálogo antes de qualquer query. Nada do domínio é reparentado neste plano — o
app continua rodando sobre `ApplicationRecord` e o banco compartilhado.

**Tech Stack:** Rails 8.1.3, PostgreSQL 15, RSpec + FactoryBot, Docker Compose
(serviço `api`, container `api-dev`).

**Spec:** `docs/superpowers/specs/2026-09-12-banco-por-cidade-design.md`

## Global Constraints

- **`connects_to` exatamente uma vez, no corpo de `CityRecord`.** Ele declara um
  único shard `:bootstrap` — o que **é obrigatório**, porque `connected_to` só é
  permitido na classe abstrata que estabeleceu a conexão: sem esse `connects_to`
  inicial, `establish_connection` sozinho não torna `CityRecord` uma connection
  class e `connected_to` levanta `NotImplementedError` (verificado em 2026-09-12).
  Depois disso, cidade nova entra **exclusivamente** por
  `ActiveRecord::Base.connection_handler.establish_connection(config, owner_name:, role:, shard:)`.
  Medido no spike 1: re-chamar `connects_to` sob tráfego causou 171.620
  interrupções em cidades já ativas; `establish_connection` custou 0,11 ms com
  zero interrupções.
- **Config de conexão vem de `ActiveRecord::Base.configurations.resolve(url)`**
  (API pública, devolve `UrlConfig`). URL malformada levanta
  `ActiveRecord::DatabaseConfigurations::InvalidConfigurationError`; URL sem
  database devolve config com `database` nulo — os dois casos precisam de guarda.
- **Registro preguiçoso por processo.** Provisionar não avisa processos; cada um
  registra o pool na primeira requisição daquela cidade.
- **O banco de plataforma nunca guarda dado de cidadão.** Só catálogo, roteamento
  e contas de operador.
- **Todo comando roda dentro do container**, nunca no host:
  `docker compose exec api <cmd>` a partir da raiz do monorepo.
  O Ruby do host é 3.4.4; o app é 3.3.6.
- **Mensagens de commit em inglês.**
- Slug de cidade: `/\A[a-z0-9]([a-z0-9-]*[a-z0-9])?\z/`, 2 a 63 caracteres —
  é rótulo de DNS, porque vira subdomínio.

---

## File Structure

| Arquivo | Responsabilidade |
|---|---|
| `config/database.yml` | ganha a config `platform` em development e test |
| `app/models/platform_record.rb` | abstrata ligada ao banco de plataforma |
| `app/models/city.rb` | modelo do catálogo (`PlatformRecord`) |
| `db/platform_migrate/*.rb` | migrations do banco de plataforma |
| `lib/tasks/platform.rake` | `platform:bootstrap` (cria DB e role como superuser) |
| `app/models/city_record.rb` | abstrata do domínio da cidade — sem conexão fixa |
| `app/models/city_connection.rb` | registry de pools por cidade |
| `app/controllers/concerns/city_resolution.rb` | host → cidade → conexão |
| `lib/tasks/city.rake` | `city:create`, `city:test_databases` |
| `spec/cities/city_isolation_spec.rb` | prova de isolamento entre dois bancos |
| `spec/architecture/connection_invariants_spec.rb` | guarda do `connects_to` |

---

### Task 1: Banco de plataforma e `PlatformRecord`

**Files:**
- Modify: `config/database.yml`
- Create: `app/models/platform_record.rb`
- Create: `db/platform_migrate/20260912000001_create_cities.rb`
- Create: `lib/tasks/platform.rake`
- Test: `spec/models/platform_record_spec.rb`

**Interfaces:**
- Consumes: nada.
- Produces: `PlatformRecord` (abstract, `connects_to database: { writing: :platform }`);
  tabela `cities` com colunas `id:uuid`, `slug:string`, `name:string`,
  `uf:string(2)`, `status:string`, `database_url:text`, `encryption_key:text`,
  `schema_version:string`, timestamps; rake `platform:bootstrap`.

- [ ] **Step 1: Write the failing test**

Create `spec/models/platform_record_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe PlatformRecord do
  it "is abstract and connects to the platform database" do
    expect(described_class.abstract_class?).to be(true)
    name = described_class.connection_db_config.database
    expect(name).to match(/platform/)
  end

  it "does not share a connection with ApplicationRecord" do
    expect(described_class.connection_db_config.database)
      .not_to eq(ApplicationRecord.connection_db_config.database)
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec api bundle exec rspec spec/models/platform_record_spec.rb`
Expected: FAIL com `uninitialized constant PlatformRecord`.

- [ ] **Step 3: Add the platform config to `config/database.yml`**

Em `development:`, depois do bloco `cache:`, acrescente:

```yaml
  # Banco de plataforma — catálogo de cidades, roteamento e contas de operador.
  # NUNCA guarda dado de cidadão. Ver spec 2026-09-12-banco-por-cidade-design.
  platform:
    <<: *default
    database: rota_saude_platform_development
    username: rota_platform
    password: <%= ENV.fetch("ROTA_PLATFORM_PASSWORD", "rota_platform") %>
    host: <%= ENV.fetch("DATABASE_HOST", "127.0.0.1") %>
    port: <%= ENV.fetch("DATABASE_PORT", 5432) %>
    migrations_paths: db/platform_migrate
    schema_dump: platform_schema.rb
```

Em `test:`, depois do bloco `admin:`, acrescente o mesmo bloco trocando o
`database` para `rota_saude_platform_test`.

- [ ] **Step 4: Create `app/models/platform_record.rb`**

```ruby
# Base das tabelas do banco de PLATAFORMA (catálogo de cidades, roteamento,
# contas de operador). Conexão fixa — a plataforma não é resolvida por host.
#
# INVARIANTE: nenhuma tabela sob PlatformRecord guarda dado de cidadão.
# Ver docs/superpowers/specs/2026-09-12-banco-por-cidade-design.md
class PlatformRecord < ActiveRecord::Base
  self.abstract_class = true

  connects_to database: { writing: :platform, reading: :platform }
end
```

- [ ] **Step 5: Create the `cities` migration**

Create `db/platform_migrate/20260912000001_create_cities.rb`:

```ruby
# Catálogo de cidades. `slug` é o subdomínio e o nome do shard.
class CreateCities < ActiveRecord::Migration[8.1]
  def change
    create_table :cities, id: :uuid do |t|
      t.string :slug,           null: false
      t.string :name,           null: false
      t.string :uf,             limit: 2
      t.string :status,         null: false, default: "provisioning"
      t.text   :database_url,   null: false
      t.text   :encryption_key, null: false
      t.string :schema_version
      t.timestamps
    end

    add_index :cities, :slug, unique: true

    add_check_constraint :cities,
      "status IN ('provisioning','active','suspended','archived')",
      name: "ck_cities_status"

    add_check_constraint :cities,
      "slug ~ '^[a-z0-9]([a-z0-9-]*[a-z0-9])?$' AND length(slug) BETWEEN 2 AND 63",
      name: "ck_cities_slug_is_dns_label"
  end
end
```

- [ ] **Step 6: Create `lib/tasks/platform.rake`**

O role `rota_app` não tem `CREATEDB` (menor privilégio, ADR-0019), então o
banco de plataforma é criado pelo superuser, no mesmo padrão de
`lib/tasks/bootstrap.rake`.

```ruby
# Bootstrap do banco de PLATAFORMA. Roda como superuser porque criar database
# e role exige privilégio que rota_app não tem — e não deve ter.
require "open3"

namespace :platform do
  def platform_conn_params
    cfg = ActiveRecord::Base.configurations
                            .configs_for(env_name: Rails.env, name: "platform")
                            .configuration_hash
    {
      db:      cfg[:database],
      host:    ENV.fetch("DATABASE_HOST", cfg[:host] || "127.0.0.1").to_s,
      port:    ENV.fetch("DATABASE_PORT", cfg[:port] || 5432).to_s,
      su_user: ENV.fetch("BOOTSTRAP_SUPERUSER", "rota_saude"),
      su_pwd:  ENV.fetch("POSTGRES_PASSWORD") { abort "[platform:bootstrap] POSTGRES_PASSWORD ausente." }
    }
  end

  desc "Cria o database e o role da plataforma (idempotente)."
  task bootstrap: :environment do
    p   = platform_conn_params
    pwd = ENV.fetch("ROTA_PLATFORM_PASSWORD", "rota_platform")
    env = { "PGPASSWORD" => p[:su_pwd] }
    base = ["psql", "-h", p[:host], "-p", p[:port], "-U", p[:su_user], "-v", "ON_ERROR_STOP=1"]

    role_sql = <<~SQL
      DO $$ BEGIN
        IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='rota_platform') THEN
          CREATE ROLE rota_platform LOGIN PASSWORD '#{pwd}';
        END IF;
      END $$;
    SQL
    out, st = Open3.capture2e(env, *base, "-d", "postgres", "-c", role_sql)
    abort "[platform:bootstrap] falha no role:\n#{out}" unless st.success?

    exists, = Open3.capture2e(env, *base, "-tA", "-d", "postgres",
                              "-c", "SELECT 1 FROM pg_database WHERE datname='#{p[:db]}'")
    if exists.strip == "1"
      puts "[platform:bootstrap] #{p[:db]} já existe"
    else
      out, st = Open3.capture2e(env, *base, "-d", "postgres",
                                "-c", "CREATE DATABASE #{p[:db]} OWNER rota_platform")
      abort "[platform:bootstrap] falha ao criar #{p[:db]}:\n#{out}" unless st.success?
      puts "[platform:bootstrap] #{p[:db]} criado"
    end
  end
end
```

- [ ] **Step 7: Bootstrap and migrate the platform database**

```bash
docker compose exec -e POSTGRES_PASSWORD=postgres api bin/rails platform:bootstrap
docker compose exec api bin/rails db:migrate:platform
docker compose exec -e RAILS_ENV=test -e POSTGRES_PASSWORD=postgres api bin/rails platform:bootstrap
docker compose exec -e RAILS_ENV=test api bin/rails db:migrate:platform
```

Expected: `rota_saude_platform_development` e `rota_saude_platform_test` criados
e com a tabela `cities`.

- [ ] **Step 8: Run test to verify it passes**

Run: `docker compose exec api bundle exec rspec spec/models/platform_record_spec.rb`
Expected: PASS, 2 examples.

- [ ] **Step 9: Verify the existing suite still passes**

Run: `docker compose exec api bundle exec rspec`
Expected: mesma contagem de antes — este plano não toca o domínio.

- [ ] **Step 10: Commit**

```bash
git add config/database.yml app/models/platform_record.rb db/platform_migrate lib/tasks/platform.rake spec/models/platform_record_spec.rb
git commit -m "Add platform database with city catalog table"
```

---

### Task 2: Modelo `City` e `CityCatalog`

**Files:**
- Create: `app/models/city.rb`
- Create: `app/models/city_catalog.rb`
- Create: `spec/factories/cities.rb`
- Test: `spec/models/city_catalog_spec.rb`

**Interfaces:**
- Consumes: `PlatformRecord`, tabela `cities` (Task 1).
- Produces: `City < PlatformRecord` com `#shard → Symbol`, `#servable? → Boolean`,
  scopes `active`; `CityCatalog.find_by_host(host) → City | nil`,
  `CityCatalog.reserved_host?(host) → Boolean`, `CityCatalog.reset_cache!`.

- [ ] **Step 1: Write the failing test**

Create `spec/models/city_catalog_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe CityCatalog do
  before do
    City.delete_all
    described_class.reset_cache!
  end

  let!(:city) { create(:city, slug: "saopaulo", status: "active") }

  it "finds a city by the host's first label" do
    expect(described_class.find_by_host("saopaulo.rotasaude.app")).to eq(city)
  end

  it "finds a city by host with a port" do
    expect(described_class.find_by_host("saopaulo.localhost:5175")).to eq(city)
  end

  it "returns nil for an unknown host" do
    expect(described_class.find_by_host("naoexiste.rotasaude.app")).to be_nil
  end

  it "treats reserved subdomains as platform hosts" do
    %w[admin api auth www].each do |label|
      expect(described_class.reserved_host?("#{label}.rotasaude.app")).to be(true)
    end
    expect(described_class.reserved_host?("saopaulo.rotasaude.app")).to be(false)
  end

  it "caches lookups and refreshes after reset" do
    described_class.find_by_host("saopaulo.rotasaude.app")
    city.update!(name: "Renomeada")
    expect(described_class.find_by_host("saopaulo.rotasaude.app").name).not_to eq("Renomeada")
    described_class.reset_cache!
    expect(described_class.find_by_host("saopaulo.rotasaude.app").name).to eq("Renomeada")
  end

  it "exposes the shard name as a symbol" do
    expect(city.shard).to eq(:saopaulo)
  end

  it "is servable only when active" do
    expect(city).to be_servable
    city.update!(status: "suspended")
    expect(city).not_to be_servable
  end
end
```

- [ ] **Step 2: Write the factory**

Create `spec/factories/cities.rb`:

```ruby
FactoryBot.define do
  factory :city do
    sequence(:slug) { |n| "cidade#{n}" }
    name { "Cidade Exemplo" }
    uf { "SP" }
    status { "active" }
    database_url { "postgres://rota_city:rota_city@127.0.0.1:5432/rota_saude_test_city_a" }
    encryption_key { SecureRandom.hex(32) }
  end
end
```

- [ ] **Step 3: Run test to verify it fails**

Run: `docker compose exec api bundle exec rspec spec/models/city_catalog_spec.rb`
Expected: FAIL com `uninitialized constant City`.

- [ ] **Step 4: Create `app/models/city.rb`**

```ruby
# Uma cidade no catálogo de plataforma. O `slug` é, ao mesmo tempo, o
# subdomínio que a resolve e o nome do shard no connection handler.
class City < PlatformRecord
  STATUSES = %w[provisioning active suspended archived].freeze

  scope :active, -> { where(status: "active") }

  validates :slug, presence: true, uniqueness: true,
                   format: { with: /\A[a-z0-9]([a-z0-9-]*[a-z0-9])?\z/ },
                   length: { in: 2..63 }
  validates :name, :database_url, :encryption_key, presence: true
  validates :status, inclusion: { in: STATUSES }

  def shard = slug.to_sym

  def servable? = status == "active"
end
```

- [ ] **Step 5: Create `app/models/city_catalog.rb`**

```ruby
# Resolve host → City. Lê o catálogo de plataforma e mantém um cache em
# processo, porque isso roda antes de toda requisição.
#
# Subdomínios reservados não são cidades: pertencem à plataforma.
class CityCatalog
  RESERVED = %w[admin api auth www].freeze

  class << self
    def find_by_host(host)
      label = label_for(host)
      return nil if label.nil? || RESERVED.include?(label)

      cache.fetch(label) { cache[label] = City.find_by(slug: label) }
    end

    def reserved_host?(host)
      RESERVED.include?(label_for(host))
    end

    def reset_cache!
      @cache = {}
    end

    private

    def cache
      @cache ||= {}
    end

    def label_for(host)
      return nil if host.blank?
      host.to_s.split(":").first.to_s.split(".").first.presence&.downcase
    end
  end
end
```

- [ ] **Step 6: Run test to verify it passes**

Run: `docker compose exec api bundle exec rspec spec/models/city_catalog_spec.rb`
Expected: PASS, 7 examples.

- [ ] **Step 7: Commit**

```bash
git add app/models/city.rb app/models/city_catalog.rb spec/factories/cities.rb spec/models/city_catalog_spec.rb
git commit -m "Add City model and host-to-city catalog lookup"
```

---

### Task 3: `CityRecord` e o registry de conexões

**Files:**
- Create: `app/models/city_record.rb`
- Create: `app/models/city_connection.rb`
- Test: `spec/models/city_connection_spec.rb`

**Interfaces:**
- Consumes: `City#shard`, `City#database_url` (Task 2).
- Produces: `CityRecord` (abstract, com **um único** `connects_to shards: { bootstrap: ... }`);
  `CityConnection.with(city) { ... }`, `CityConnection.registered?(shard) → Boolean`,
  `CityConnection.ensure_pool(city) → void`, `CityConnection::InvalidCityDatabase`.

- [ ] **Step 1: Write the failing test**

Create `spec/models/city_connection_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe CityConnection do
  let(:city) do
    build(:city, slug: "conncity",
          database_url: ENV.fetch("TEST_CITY_A_URL",
            "postgres://rota_saude:postgres@127.0.0.1:5432/rota_saude_test_city_a"))
  end

  it "registers a pool on first use and reuses it afterwards" do
    expect(described_class.registered?(city.shard)).to be(false)
    described_class.ensure_pool(city)
    expect(described_class.registered?(city.shard)).to be(true)

    pool = ActiveRecord::Base.connection_handler
             .retrieve_connection_pool(CityRecord.name, role: :writing, shard: city.shard)
    described_class.ensure_pool(city)
    expect(ActiveRecord::Base.connection_handler
             .retrieve_connection_pool(CityRecord.name, role: :writing, shard: city.shard))
      .to equal(pool)
  end

  it "runs the block against the city's database" do
    result = described_class.with(city) { CityRecord.connection_db_config.database }
    expect(result).to eq("rota_saude_test_city_a")
  end

  it "raises for a city whose pool cannot be built" do
    broken = build(:city, slug: "quebrada", database_url: "not-a-url")
    expect { described_class.with(broken) { 1 } }
      .to raise_error(CityConnection::InvalidCityDatabase, /quebrada/)
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec api bundle exec rspec spec/models/city_connection_spec.rb`
Expected: FAIL com `uninitialized constant CityConnection`.

- [ ] **Step 3: Create `app/models/city_record.rb`**

```ruby
# Base do domínio de uma cidade. A conexão real é resolvida em runtime por
# CityConnection, um shard por cidade.
#
# O `connects_to` abaixo é OBRIGATÓRIO e roda uma única vez, no boot: sem ele
# CityRecord não é uma "connection class", e connected_to levanta
# NotImplementedError ("only allowed on the abstract class that established the
# connection"). O shard :bootstrap nunca é usado para servir cidade — ele só
# existe para registrar CityRecord no connection handler.
#
# NUNCA chame connects_to de novo: ele reconstrói o mapa inteiro de shards e
# derruba cidades em voo (spike 1: 171.620 interrupções sob tráfego).
#
# Enquanto o Plano 2 não reparenta o domínio, só os specs de isolamento
# herdam daqui.
class CityRecord < ActiveRecord::Base
  self.abstract_class = true

  connects_to shards: { bootstrap: { writing: :primary } }
end
```

- [ ] **Step 4: Create `app/models/city_connection.rb`**

```ruby
# Registry de pools, um por cidade.
#
# INVARIANTE (spike 1, 2026-09-12): `connects_to` reconstrói o mapa inteiro de
# shards e derruba cidades já ativas — 171.620 interrupções sob tráfego. Só
# `establish_connection` adiciona um pool isolado (0,11 ms, zero interrupções).
# NUNCA chame connects_to em CityRecord.
#
# O registro é preguiçoso POR PROCESSO: com Puma multi-worker e réplicas,
# provisionar não avisa ninguém — cada processo descobre a cidade ao servi-la.
class CityConnection
  class InvalidCityDatabase < StandardError; end

  MUTEX = Mutex.new

  class << self
    def with(city, &block)
      ensure_pool(city)
      CityRecord.connected_to(shard: city.shard, role: :writing, &block)
    end

    def ensure_pool(city)
      return if registered?(city.shard)

      MUTEX.synchronize do
        next if registered?(city.shard)

        ActiveRecord::Base.connection_handler.establish_connection(
          db_config_for(city),
          owner_name: CityRecord,
          role: :writing,
          shard: city.shard
        )
      end
    end

    def registered?(shard)
      !ActiveRecord::Base.connection_handler
        .retrieve_connection_pool(CityRecord.name, role: :writing, shard: shard)
        .nil?
    end

    private

    # Verificado em 2026-09-12 contra o Rails 8.1.3:
    #   URL malformada  -> InvalidConfigurationError
    #   URL sem database -> UrlConfig com database nulo
    # Os dois viram InvalidCityDatabase, para o chamador ter um erro só.
    def db_config_for(city)
      url = city.database_url.to_s
      url += (url.include?("?") ? "&" : "?") + "pool=#{pool_size}"

      resolved = ActiveRecord::Base.configurations.resolve(url)
      if resolved.database.blank?
        raise InvalidCityDatabase, "cidade #{city.slug}: database_url sem database"
      end

      resolved
    rescue ActiveRecord::DatabaseConfigurations::InvalidConfigurationError => e
      raise InvalidCityDatabase, "cidade #{city.slug}: #{e.message}"
    end

    def pool_size
      ENV.fetch("RAILS_MAX_THREADS", 5).to_i
    end
  end
end
```

- [ ] **Step 5: Create the test city databases**

Add to `lib/tasks/city.rake` (arquivo novo):

```ruby
require "open3"

namespace :city do
  TEST_CITY_DATABASES = %w[rota_saude_test_city_a rota_saude_test_city_b].freeze

  desc "Cria os bancos de cidade usados pelos specs de isolamento (idempotente)."
  task test_databases: :environment do
    su   = ENV.fetch("BOOTSTRAP_SUPERUSER", "rota_saude")
    pwd  = ENV.fetch("POSTGRES_PASSWORD") { abort "[city:test_databases] POSTGRES_PASSWORD ausente." }
    host = ENV.fetch("DATABASE_HOST", "127.0.0.1")
    port = ENV.fetch("DATABASE_PORT", "5432").to_s
    env  = { "PGPASSWORD" => pwd }
    base = ["psql", "-h", host, "-p", port, "-U", su, "-v", "ON_ERROR_STOP=1"]

    TEST_CITY_DATABASES.each do |db|
      exists, = Open3.capture2e(env, *base, "-tA", "-d", "postgres",
                                "-c", "SELECT 1 FROM pg_database WHERE datname='#{db}'")
      if exists.strip == "1"
        puts "[city:test_databases] #{db} já existe"
        next
      end
      out, st = Open3.capture2e(env, *base, "-d", "postgres", "-c", "CREATE DATABASE #{db} OWNER #{su}")
      abort "[city:test_databases] falha ao criar #{db}:\n#{out}" unless st.success?
      puts "[city:test_databases] #{db} criado"
    end

    TEST_CITY_DATABASES.each do |db|
      out, st = Open3.capture2e(env, *base, "-d", db, "-c",
        "CREATE TABLE IF NOT EXISTS probes (id serial PRIMARY KEY, label text NOT NULL)")
      abort "[city:test_databases] falha ao criar probes em #{db}:\n#{out}" unless st.success?
    end
  end
end
```

Run:

```bash
docker compose exec -e POSTGRES_PASSWORD=postgres api bin/rails city:test_databases
```

Expected: os dois bancos criados, cada um com a tabela `probes`.

- [ ] **Step 6: Run test to verify it passes**

Run: `docker compose exec api bundle exec rspec spec/models/city_connection_spec.rb`
Expected: PASS, 3 examples.

- [ ] **Step 7: Commit**

```bash
git add app/models/city_record.rb app/models/city_connection.rb lib/tasks/city.rake spec/models/city_connection_spec.rb
git commit -m "Add per-city connection registry using establish_connection"
```

---

### Task 4: Resolução por host no controller

**Files:**
- Create: `app/controllers/concerns/city_resolution.rb`
- Modify: `app/models/current.rb`
- Test: `spec/requests/city_resolution_spec.rb`

**Interfaces:**
- Consumes: `CityCatalog.find_by_host`, `CityCatalog.reserved_host?` (Task 2),
  `CityConnection.with` (Task 3).
- Produces: concern `CityResolution` com `around_action :within_city` e o
  class method `skip_city_resolution(**options)`; `Current.city`.

- [ ] **Step 1: Write the failing test**

Create `spec/requests/city_resolution_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "City resolution", type: :request do
  before do
    City.delete_all
    CityCatalog.reset_cache!
  end

  # Controller de teste, montado só neste spec.
  before(:all) do
    Rails.application.routes.disable_clear_and_finalize = true
    Rails.application.routes.draw do
      get "/_probe", to: "city_probe#show"
    end
  end

  after(:all) do
    Rails.application.routes.disable_clear_and_finalize = false
    Rails.application.reload_routes!
  end

  let(:city_a_url) do
    ENV.fetch("TEST_CITY_A_URL",
      "postgres://rota_saude:postgres@127.0.0.1:5432/rota_saude_test_city_a")
  end

  it "serves an active city and exposes it on Current" do
    create(:city, slug: "cidadeviva", status: "active", database_url: city_a_url)
    get "/_probe", headers: { "HOST" => "cidadeviva.rotasaude.app" }

    expect(response).to have_http_status(:ok)
    expect(JSON.parse(response.body)).to include(
      "city" => "cidadeviva", "database" => "rota_saude_test_city_a"
    )
  end

  it "returns 404 for an unknown host" do
    get "/_probe", headers: { "HOST" => "inexistente.rotasaude.app" }
    expect(response).to have_http_status(:not_found)
    expect(JSON.parse(response.body)["error"]).to eq("unknown_city")
  end

  it "returns 403 for a suspended city" do
    create(:city, slug: "suspensa", status: "suspended", database_url: city_a_url)
    get "/_probe", headers: { "HOST" => "suspensa.rotasaude.app" }
    expect(response).to have_http_status(:forbidden)
    expect(JSON.parse(response.body)["error"]).to eq("city_suspended")
  end

  it "returns 404 for a city still provisioning" do
    create(:city, slug: "nascendo", status: "provisioning", database_url: city_a_url)
    get "/_probe", headers: { "HOST" => "nascendo.rotasaude.app" }
    expect(response).to have_http_status(:not_found)
  end

  it "returns 404 for a reserved subdomain" do
    get "/_probe", headers: { "HOST" => "admin.rotasaude.app" }
    expect(response).to have_http_status(:not_found)
  end
end
```

Create `spec/support/city_probe_controller.rb`:

```ruby
# Controller usado apenas pelo spec de resolução. Fica em support para não
# poluir app/.
class CityProbeController < ActionController::API
  include CityResolution

  def show
    render json: {
      city: Current.city&.slug,
      database: CityRecord.connection_db_config.database
    }
  end
end
```

Add to `spec/rails_helper.rb`, logo após `require 'factory_bot_rails'`:

```ruby
require_relative "support/city_probe_controller"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec api bundle exec rspec spec/requests/city_resolution_spec.rb`
Expected: FAIL com `uninitialized constant CityResolution`.

- [ ] **Step 3: Add `city` to `Current`**

Modify `app/models/current.rb` — acrescente o atributo sem remover nada:

```ruby
# CurrentAttributes resetado por request e por job (ver ADR-0003).
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  attribute :municipality_id
  # Cidade resolvida pelo host. Serve para log e para o envelope de resposta.
  # NUNCA use em WHERE: o escopo é a conexão, não um valor de coluna.
  attribute :city

  delegate :user, to: :session, allow_nil: true
end
```

- [ ] **Step 4: Create `app/controllers/concerns/city_resolution.rb`**

```ruby
# Resolve a cidade pelo host ANTES de qualquer query, e executa a ação dentro
# da conexão daquela cidade.
#
# A ordem importa e é o inverso da que TenantScopedRequest usava: primeiro
# resolve a cidade, depois autentica. É isso que torna um cookie de uma cidade
# inútil na vizinha — a sessão é procurada no banco da cidade do host.
module CityResolution
  extend ActiveSupport::Concern

  included do
    around_action :within_city
  end

  class_methods do
    def skip_city_resolution(**options)
      skip_around_action :within_city, **options
    end
  end

  private

  def within_city(&block)
    city = CityCatalog.find_by_host(request.host)

    return render(json: { error: "unknown_city" }, status: :not_found) if city.nil?
    return render(json: { error: "city_suspended" }, status: :forbidden) if city.status == "suspended"
    return render(json: { error: "unknown_city" }, status: :not_found) unless city.servable?

    Current.city = city
    CityConnection.with(city, &block)
  end
end
```

- [ ] **Step 5: Run test to verify it passes**

Run: `docker compose exec api bundle exec rspec spec/requests/city_resolution_spec.rb`
Expected: PASS, 5 examples.

- [ ] **Step 6: Verify the existing suite still passes**

Run: `docker compose exec api bundle exec rspec`
Expected: sem regressão — nenhum controller do app inclui `CityResolution` ainda.

- [ ] **Step 7: Commit**

```bash
git add app/controllers/concerns/city_resolution.rb app/models/current.rb spec/requests/city_resolution_spec.rb spec/support/city_probe_controller.rb spec/rails_helper.rb
git commit -m "Resolve city from request host before any query"
```

---

### Task 5: `city:create` para desenvolvimento

**Files:**
- Modify: `lib/tasks/city.rake`
- Test: `spec/tasks/city_create_spec.rb`

**Interfaces:**
- Consumes: `City` (Task 2).
- Produces: rake `city:create[slug,name,uf]`; `CityProvisioner.call(slug:, name:, uf:) → City`.

O provisionamento completo (migrations, seeds, primeiro admin) é do Plano 4.
Aqui o objetivo é só conseguir criar uma cidade servível em dev.

- [ ] **Step 1: Write the failing test**

Create `spec/tasks/city_create_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe CityProvisioner do
  before { City.delete_all }

  it "registers a city in the catalog as provisioning" do
    city = described_class.call(slug: "novacidade", name: "Nova Cidade", uf: "SP")

    expect(city).to be_persisted
    expect(city.status).to eq("provisioning")
    expect(city.database_url).to include("rota_saude_city_novacidade")
    expect(city.encryption_key.length).to eq(64)
  end

  it "is idempotent on slug" do
    first  = described_class.call(slug: "repetida", name: "Repetida", uf: "SP")
    second = described_class.call(slug: "repetida", name: "Repetida", uf: "SP")
    expect(second.id).to eq(first.id)
    expect(City.where(slug: "repetida").count).to eq(1)
  end

  it "rejects a slug that is not a DNS label" do
    expect { described_class.call(slug: "Nao Vale", name: "X", uf: "SP") }
      .to raise_error(ActiveRecord::RecordInvalid)
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec api bundle exec rspec spec/tasks/city_create_spec.rb`
Expected: FAIL com `uninitialized constant CityProvisioner`.

- [ ] **Step 3: Create `app/models/city_provisioner.rb`**

```ruby
# Registra uma cidade no catálogo. NÃO cria o banco nem roda migrations —
# isso é o Plano 4 (provisionamento em duas fases). Aqui só existe o suficiente
# para desenvolvimento.
class CityProvisioner
  def self.call(slug:, name:, uf: nil)
    existing = City.find_by(slug: slug)
    return existing if existing

    City.create!(
      slug: slug,
      name: name,
      uf: uf,
      status: "provisioning",
      database_url: database_url_for(slug),
      encryption_key: SecureRandom.hex(32)
    )
  end

  def self.database_url_for(slug)
    host = ENV.fetch("DATABASE_HOST", "127.0.0.1")
    port = ENV.fetch("DATABASE_PORT", "5432")
    user = ENV.fetch("BOOTSTRAP_SUPERUSER", "rota_saude")
    pwd  = ENV.fetch("POSTGRES_PASSWORD", "postgres")
    "postgres://#{user}:#{pwd}@#{host}:#{port}/rota_saude_city_#{slug}"
  end
end
```

- [ ] **Step 4: Run test to verify it passes**

Run: `docker compose exec api bundle exec rspec spec/tasks/city_create_spec.rb`
Expected: PASS, 3 examples.

- [ ] **Step 5: Add the rake wrapper**

Append to `lib/tasks/city.rake`, dentro do `namespace :city do`:

```ruby
  desc "Registra uma cidade no catálogo (dev). Uso: city:create[slug,nome,uf]"
  task :create, %i[slug name uf] => :environment do |_t, args|
    abort "uso: rails 'city:create[slug,nome,uf]'" if args[:slug].blank? || args[:name].blank?
    city = CityProvisioner.call(slug: args[:slug], name: args[:name], uf: args[:uf])
    puts "[city:create] #{city.slug} → #{city.status} (#{city.database_url.sub(/:[^:@]+@/, ':***@')})"
  end
```

- [ ] **Step 6: Verify the task runs**

```bash
docker compose exec -e POSTGRES_PASSWORD=postgres api bin/rails 'city:create[curitiba,Curitiba,PR]'
```

Expected: `[city:create] curitiba → provisioning (postgres://rota_saude:***@...)`.

- [ ] **Step 7: Commit**

```bash
git add app/models/city_provisioner.rb lib/tasks/city.rake spec/tasks/city_create_spec.rb
git commit -m "Add city:create rake task for development"
```

---

### Task 6: Spec de isolamento entre cidades

**Files:**
- Create: `spec/cities/city_isolation_spec.rb`

**Interfaces:**
- Consumes: `CityConnection.with` (Task 3), `city:test_databases` (Task 3),
  tabela `probes` nos dois bancos de teste.
- Produces: nada — é o herdeiro do `spec/rls/tenant_isolation_spec.rb`, provando
  os mesmos invariantes pelo novo mecanismo.

- [ ] **Step 1: Write the test**

Create `spec/cities/city_isolation_spec.rb`:

```ruby
require "rails_helper"

# Herdeiro de spec/rls/tenant_isolation_spec.rb. Os invariantes são os mesmos;
# o mecanismo mudou de policy do Postgres para conexão por cidade.
#
# Pré-requisito: rails city:test_databases
RSpec.describe "City isolation", type: :model do
  class Probe < CityRecord
    self.table_name = "probes"
  end

  def url_for(db)
    host = ENV.fetch("DATABASE_HOST", "127.0.0.1")
    port = ENV.fetch("DATABASE_PORT", "5432")
    pwd  = ENV.fetch("POSTGRES_PASSWORD", "postgres")
    "postgres://rota_saude:#{pwd}@#{host}:#{port}/#{db}"
  end

  let(:city_a) { build(:city, slug: "isoa", database_url: url_for("rota_saude_test_city_a")) }
  let(:city_b) { build(:city, slug: "isob", database_url: url_for("rota_saude_test_city_b")) }

  before do
    [city_a, city_b].each do |c|
      CityConnection.with(c) { Probe.delete_all }
    end
  end

  it "does not see rows written in another city" do
    CityConnection.with(city_a) { Probe.create!(label: "de-a") }
    CityConnection.with(city_b) { Probe.create!(label: "de-b") }

    expect(CityConnection.with(city_a) { Probe.pluck(:label) }).to eq(["de-a"])
    expect(CityConnection.with(city_b) { Probe.pluck(:label) }).to eq(["de-b"])
  end

  it "writes land in the city that is connected" do
    CityConnection.with(city_a) { Probe.create!(label: "so-em-a") }
    expect(CityConnection.with(city_b) { Probe.count }).to eq(0)
  end

  it "keeps concurrent readers on their own city" do
    CityConnection.with(city_a) { Probe.create!(label: "de-a") }
    CityConnection.with(city_b) { Probe.create!(label: "de-b") }

    errors = Queue.new
    threads = 20.times.map do |i|
      Thread.new do
        city, want = i.even? ? [city_a, "de-a"] : [city_b, "de-b"]
        20.times do
          got = CityConnection.with(city) { Probe.pluck(:label) }
          errors << "#{city.slug} leu #{got.inspect}" unless got == [want]
        end
      rescue => e
        errors << "#{city.slug} -> #{e.class}: #{e.message}"
      end
    end
    threads.each(&:join)

    expect(errors.size).to eq(0), "cruzamento entre cidades: #{errors.pop unless errors.empty?}"
  end

  it "fails closed for a city that was never registered" do
    ghost = build(:city, slug: "fantasma", database_url: url_for("banco_que_nao_existe"))
    expect { CityConnection.with(ghost) { Probe.count } }
      .to raise_error(ActiveRecord::NoDatabaseError)
  end
end
```

- [ ] **Step 2: Run the test**

Run: `docker compose exec -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/cities/city_isolation_spec.rb`
Expected: PASS, 4 examples.

Se falhar com `PG::UndefinedTable: relation "probes" does not exist`, rode
`docker compose exec -e POSTGRES_PASSWORD=postgres api bin/rails city:test_databases` primeiro.

- [ ] **Step 3: Commit**

```bash
git add spec/cities/city_isolation_spec.rb
git commit -m "Prove city isolation across separate databases"
```

---

### Task 7: Guarda de arquitetura

**Files:**
- Create: `spec/architecture/connection_invariants_spec.rb`

**Interfaces:**
- Consumes: nada — lê o código-fonte, no estilo de `spec/adr_pointers_spec.rb`.
- Produces: nada.

- [ ] **Step 1: Write the test**

Create `spec/architecture/connection_invariants_spec.rb`:

```ruby
require "rails_helper"

# Guardas do modelo de conexão (spec 2026-09-12-banco-por-cidade-design).
#
# O invariante do connects_to não é estilo: medido no spike 1, re-chamar
# connects_to sob tráfego causou 171.620 interrupções em cidades já ativas.
RSpec.describe "Connection invariants" do
  SOURCE_ROOTS = %w[app lib config].freeze
  SELF_PATH = "spec/architecture/connection_invariants_spec.rb"

  def source_files
    Dir.chdir(Rails.root) do
      SOURCE_ROOTS.flat_map { |r| Dir.glob("#{r}/**/*.rb") }.sort - [SELF_PATH]
    end
  end

  it "only calls connects_to from an abstract class body" do
    offenders = Dir.chdir(Rails.root) do
      source_files.flat_map do |path|
        lines = File.readlines(path, encoding: "UTF-8")
        next [] unless lines.any? { |l| l.include?("connects_to") }
        next [] if lines.any? { |l| l.match?(/self\.abstract_class\s*=\s*true/) }

        lines.each_with_index
             .select { |l, _| l.include?("connects_to") }
             .map { |_, i| "#{path}:#{i + 1}" }
      end
    end

    expect(offenders).to eq([]),
      "connects_to fora de classe abstrata (use CityConnection.ensure_pool):\n#{offenders.join("\n")}"
  end

  it "never calls connects_to on CityRecord from outside its own class body" do
    offenders = Dir.chdir(Rails.root) do
      (source_files - ["app/models/city_record.rb"]).select do |path|
        File.read(path, encoding: "UTF-8").match?(/CityRecord\.connects_to/)
      end
    end

    expect(offenders).to eq([]),
      "re-chamar connects_to derruba cidades em voo — use CityConnection.ensure_pool:\n#{offenders.join("\n")}"
  end

  it "declares connects_to exactly once in CityRecord" do
    expect(CityRecord.abstract_class?).to be(true)

    source = File.read(Rails.root.join("app/models/city_record.rb"), encoding: "UTF-8")
    occurrences = source.scan(/^\s*connects_to\b/).size

    expect(occurrences).to eq(1),
      "CityRecord precisa de exatamente um connects_to: zero quebra connected_to " \
      "(NotImplementedError), mais de um reconstrói o mapa de shards."
  end

  it "registers cities through establish_connection, not connects_to" do
    source = File.read(Rails.root.join("app/models/city_connection.rb"), encoding: "UTF-8")
    expect(source).to match(/establish_connection/)
    expect(source).not_to match(/connects_to/)
  end

  it "keeps the platform database free of citizen data tables" do
    citizen_tables = %w[conversations triages inbound_messages outbound_messages
                        consents report_snapshots].freeze
    present = PlatformRecord.connection.tables & citizen_tables

    expect(present).to eq([]),
      "tabelas de dado de cidadão no banco de plataforma: #{present.join(', ')}"
  end
end
```

- [ ] **Step 2: Run the test**

Run: `docker compose exec api bundle exec rspec spec/architecture/connection_invariants_spec.rb`
Expected: PASS, 5 examples.

- [ ] **Step 3: Run the whole suite**

Run: `docker compose exec -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: verde, com os novos examples somados e nenhuma regressão.

- [ ] **Step 4: Commit**

```bash
git add spec/architecture/connection_invariants_spec.rb
git commit -m "Guard connection invariants with an architecture spec"
```

---

## Definition of done

- `rota_saude_platform_development` e `rota_saude_platform_test` existem com a
  tabela `cities`.
- `rails 'city:create[curitiba,Curitiba,PR]'` registra a cidade no catálogo.
- Um request em `<slug>.rotasaude.app` num controller que inclui `CityResolution`
  responde com a conexão daquela cidade; host desconhecido dá 404, cidade
  suspensa dá 403.
- `spec/cities/city_isolation_spec.rb` prova que A não enxerga B, inclusive sob
  20 threads concorrentes.
- A suíte existente segue verde: o domínio não foi tocado.

## O que este plano NÃO faz

- Não reparenta nenhum modelo do domínio para `CityRecord` (Plano 2).
- Não remove RLS, roles, `Current.municipality_id` nem `Admin::Scoped` (Plano 2).
- Não move `users`/`sessions` para o banco da cidade (Plano 3).
- Não cria banco de cidade nem roda migrations nele (Plano 4).
- Não trata guarda de `schema_version` desatualizada com 503 (Plano 4, junto com
  `city:migrate:all`).
- Não sobe worker por cidade (Plano 5).
- Não toca frontends, CORS nem chaves por cidade (Plano 6).
