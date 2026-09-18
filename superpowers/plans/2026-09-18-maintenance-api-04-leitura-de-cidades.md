# Plano 4 — Leitura de cidades (fatia 4)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** a API de manutenção passa a responder o que a tela `/maintenance` responde, por GraphQL e para todas as cidades: catálogo, configuração, contas, canal e sinais de operação — sem nenhum segredo, sem nenhum conteúdo de cidadão, e respeitando o escopo por cidade dos tokens de serviço.

**Architecture:** `cities` lê só o banco de plataforma. `city(slug:)` é o **único** caminho para dentro do banco de uma cidade, e é onde o escopo do token é aplicado. Cada subárvore de cidade resolve dentro de `CityConnection.with`, com a falha isolada naquele campo. Um analisador limita quantas cidades uma operação pode tocar.

**Tech Stack:** Rails 8.1.3, `graphql-ruby` 2.6, Postgres, RSpec.

**Spec:** `docs/superpowers/specs/2026-09-17-maintenance-graphql-api-design.md` (§7 escopo por cidade, §8 schema, acesso às cidades e limites, §10 testes, §11 fatia 4).

**Planos anteriores:** 1, 2 e 3 mergeados e pusheados (merges `1452053`, `b7263b0`, `e75af36`). Baseline de entrada: **1083 exemplos, 0 falhas**.

## Decisões minhas na escrita (cada uma com o custo se estiver errada)

1. **A inteligência de leitura é do GraphQL, não do `CityInventory`.** A tela `/maintenance` continua com o dela, intocada. Unificar os dois agora acoplaria uma tela dev-only a um schema público e obrigaria a mexer no que já funciona e tem spec. **Custo se errado:** duas leituras do mesmo assunto convivem até alguém decidir aposentar a tela; a duplicação é de *consulta*, não de regra — a regra ("sem segredo, sem cidadão") é guardada por spec nos dois lados.
2. **`counts` é o único lugar onde dado de cidadão aparece, e como número.** Conversa, mensagem, triagem e consentimento entram por contagem. É a mesma regra da tela, e o adiamento do mascaramento continua valendo. **Custo se errado:** quem precisa depurar conteúdo continua indo ao console.
3. **O teto é de 5 cidades por operação, contado por analisador**, junto dos que já existem. Alias, fragmento e segunda operação contam igual. **Custo se errado:** uma tela que queira comparar 6 cidades faz duas chamadas.
4. **Cidade inalcançável é erro de campo, não da resposta.** A operação devolve as cidades que responderam e, no lugar da que falhou, um erro com `extensions.code = "CITY_UNREACHABLE"` e a mensagem passada por `CitySchema.redact`. **Custo se errado:** um cliente ingênuo que ignore `errors` mostra a cidade como vazia em vez de quebrada.
5. **Cidade arquivada não abre conexão.** Os campos de plataforma respondem; os de dentro do banco devolvem `CITY_ARCHIVED`. É o que a tela já faz, pelo mesmo motivo: o banco não existe mais. **Custo se errado:** quem quiser inspecionar uma cidade arquivada precisa do backup, que é onde ela está.
6. **O escopo por cidade é aplicado em `city(slug:)` e no filtro de `cities`.** Nenhum resolver de subárvore repete a regra: sem passar por `city(slug:)`, não se chega ao banco de cidade nenhum. **Custo se errado:** um campo futuro que abra conexão por outro caminho escapa — e é isso que a guarda da Task 2 passa a vigiar.

## Global Constraints

Valem para TODA task:

- **Nunca expor segredo:** `database_url`, `encryption_key`, `access_token` do canal, `password_digest`, `otp_secret`, `token_digest`, digest de convite. Nem mascarados: ausentes.
- **Nunca expor conteúdo de cidadão:** telefone, texto de mensagem, `raw` do webhook, evidência de consentimento, contexto e resposta de triagem. Só contagem e metadado. **Nenhum campo novo de cidadão entra sem passar pelo documento de adiamento do mascaramento.**
- **Toda mensagem de erro que venha de uma conexão de cidade passa por `CitySchema.redact`** — `PG::ConnectionBad` traz a URL com a senha do role.
- **Ruling R18** continua valendo para todo `PlatformEvent` novo (nome em `MaintenanceAudit::NAMES`, no `case` e em `R18_PLATFORM_EVENT_NAMES`; nenhuma chave com `email`, `name`, `phone`, `cpf`, `body`, `wa_id`, `provider_uid`, nem `from`).
- **Leitura não é auditada** (spec §9): não acrescente evento de auditoria para consulta.
- **Ambiente publicado pergunta `Rota.deployed?`**; a guarda de arquitetura varre `app`, `config` (fora de `config/environments/`), `lib`, `db`, `bin`, `script`, `Rakefile`, `config.ru`.
- **Commits em inglês, Conventional Commits**, terminando com a linha exata `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>` — copiada literalmente, **nunca** o nome do próprio modelo. Confira com `/opt/homebrew/bin/git log -1 --format=%B | tail -1`.
- **Comandos Ruby/Rails/rspec rodam no container**, a partir da raiz do monorepo: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Nunca no host.
- **Suíte completa uma vez, por task, em primeiro plano:** `docker compose stop worker`, suíte com timeout de 600000 ms, `docker compose start worker`. **Nada notifica ninguém** — nunca espere por execução em segundo plano.
- **Não mexa no `Gemfile`.** Gems moram na imagem, que o `worker` compartilha; mudança ali exige reconstruir a imagem e recriar os dois containers.
- **Nunca desabilite o trigger `platform_events_maintenance_immutable`.** Se uma sondagem escrever evento de manutenção que você não consegue apagar, diga isso no relatório.
- **As cidades de dev são `curitiba` e `maringa` (ativas), `cascavel` e `londrina` (arquivadas, sem banco).** Não crie, não apague e não migre cidade de dev. Cidade de teste vem do harness (`TEST_CITY_A`).
- **`git` do PATH está quebrado:** use `/opt/homebrew/bin/git`, a partir de `apps/api`. **Nunca dar push.** Branch `feat/maintenance-api-cities`, criada de `main`.

## Fatos verificados (não re-descobrir)

- **`CityConnection.with(city) { ... }`** entra no shard da cidade, seta `Current.city` e o contexto de cifra. Cidade sem pool registrado é registrada na hora; URL inválida levanta `CityConnection::InvalidCityDatabase`, cuja mensagem **nunca** interpola o erro original (a URL tem credencial).
- **`City`** (banco de plataforma): `slug`, `name`, `uf`, `status` (`provisioning`, `active`, `suspended`, `archived`), `schema_version`, `ibge_code`, `created_at`; `encrypts :database_url, :encryption_key`; `#shard`; `#servable?` (só `active`); scope `.active`.
- **`CitySchema`**: `expected_version`, `behind?(city)`, e **`redact(text)`**, que troca `://user:pass@` por `://***@`.
- **`CityChannel`** (plataforma): `phone_number_id`, `waba_id`, `display_phone_number`, `active`, e `access_token` **cifrado** — nunca sai.
- **Dentro do banco da cidade:** `CityProfile.current` (`name`, `uf`, `ibge_code`), `AlertRecipient.active` (`channel`, `destination`, `escalation_order`), `ConsentTerm` (`version`), `ProtocolDefinition.active` (`name`, `version`, `status`), `Membership.active` (`role`), `User` (`email_address`, `active?`, `mfa_enrolled?`), `Conversation`, `Triage`, `InboundMessage`, `ReportSnapshot`, `DashboardMetric`, `DomainEvent`, `SolidQueue::Job`.
- **`CityInventory`** (`app/services/city_inventory.rb`) é a leitura equivalente da tela dev-only: `SKIPPED_STATUSES = %w[archived]`, sondagem com `rescue StandardError` que registra a classe do erro, contagens em `counts`, e o comentário que fixa as duas regras. **Não altere este arquivo neste plano.**
- **Schema de manutenção:** `Maintenance::Schema` com `max_depth 10`, `max_complexity 200`, `use GraphQL::Schema::Timeout, max_seconds: 10`, `query_analyzer` de `WriteScope` e `HumanOnly`, e `mutation Types::MutationType`. Tipos publicados hoje: `Query` (`me`, `maintenanceTokens`, `auditEvents`), `Mutation` (4 mutations), `Maintainer`, `MaintenanceToken`, `AuditEvent`, `UserError` e os payloads.
- **`Maintenance::Credential`** (`app/services/maintenance/credential.rb`): `human?`, `token?`, `read_only?`, **`allows_city?(slug)`** (humano sempre true; token respeita `city_slugs`, lista vazia = todas), `maintainer`, `audit_payload`.
- **`Maintenance::Analyzers::Refusal`** é o contrato entre analisador e auditoria: o analisador **etiqueta** o erro com `extensions.code = "TOKEN_SCOPE_REFUSED"` e `refusedFields`; `Maintenance::GraphqlController` lê a etiqueta e grava **um** `maintenance.token.refused`. Analisador não escreve.
- **`spec/graphql/maintenance/analyzers_spec.rb`** tem hoje o exemplo `"has no root field with a city-scoped argument yet"`, que falha assim que um campo de raiz ganhar argumento `slug`/`citySlug`. **Ele é um alarme, e a Task 2 o substitui pela guarda definitiva — nunca apague sem substituir.**
- **`spec/architecture/maintenance_schema_spec.rb`**: `EXPECTED_TYPES` (tipo → campos, camelCase), `FORBIDDEN_FRAGMENTS` (`phone body raw evidence response context digest secret token key url`), `ALLOWED_NAMES` (allowlist **nominal e exata**) e um auto-teste do padrão.
- **Harness de teste de cidade:** os specs que precisam de banco de cidade usam as cidades de teste do harness (ver `spec/support`), e a suíte roda com o worker parado para não esgotar conexões.

---

## File Structure

**apps/api**
- Create: `app/graphql/maintenance/types/city_summary_type.rb`, `app/graphql/maintenance/types/city_type.rb`, `app/graphql/maintenance/types/city_channel_type.rb`, `app/graphql/maintenance/types/city_profile_type.rb`, `app/graphql/maintenance/types/alert_recipient_type.rb`, `app/graphql/maintenance/types/protocol_definition_type.rb`, `app/graphql/maintenance/types/membership_type.rb`, `app/graphql/maintenance/types/city_account_type.rb`, `app/graphql/maintenance/types/city_counts_type.rb`, `app/graphql/maintenance/types/city_operations_type.rb`, `app/graphql/maintenance/analyzers/city_budget.rb`, `app/queries/maintenance/city_catalog_query.rb`, `app/queries/maintenance/city_reader.rb`
- Create (specs): `spec/requests/maintenance/cities_spec.rb`, `spec/requests/maintenance/city_spec.rb`, `spec/requests/maintenance/city_operations_spec.rb`, `spec/queries/maintenance/city_reader_spec.rb`
- Modify: `app/graphql/maintenance/types/query_type.rb`, `app/graphql/maintenance/schema.rb`, `spec/architecture/maintenance_schema_spec.rb`, `spec/graphql/maintenance/analyzers_spec.rb`

Branch: `feat/maintenance-api-cities`, de `main` (merge `e75af36`).

---

### Task 1: `cities` — o catálogo, sem tocar em banco de cidade

**Files:**
- Create: `app/queries/maintenance/city_catalog_query.rb`, `app/graphql/maintenance/types/city_summary_type.rb`
- Modify: `app/graphql/maintenance/types/query_type.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/cities_spec.rb`

**Interfaces:**
- Consumes: `City`, `CitySchema`, `Maintenance::Credential#allows_city?`.
- Produces:
  - `Maintenance::CityCatalogQuery.call(status: nil, credential:)` → `[City]` já filtrado pelo escopo da credencial, ordenado por `slug`
  - `Query.cities(status: CityStatus)` → `[CitySummary!]!`
  - `CitySummary`: `slug`, `name`, `uf`, `status`, `schemaVersion`, `schemaBehind`, `createdAt`
  - `CityStatus` (enum): `PROVISIONING`, `ACTIVE`, `SUSPENDED`, `ARCHIVED`

- [ ] **Step 1: Escrever o spec que falha**

`spec/requests/maintenance/cities_spec.rb`:

```ruby
require "rails_helper"

# Spec §8: `cities` lê SÓ o banco de plataforma — nenhuma conexão de cidade é
# aberta — e um token só enxerga as cidades do seu escopo.
RSpec.describe "Maintenance cities", type: :request do
  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:password) { "s3nha-forte-1" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "ct-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end

  def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
  def json = JSON.parse(response.body)

  def login!
    post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now },
         headers: browser
    expect(response).to have_http_status(:ok)
  end

  QUERY = <<~GQL
    query($status: CityStatus) {
      cities(status: $status) { slug name uf status schemaVersion schemaBehind createdAt }
    }
  GQL

  def cities!(headers: browser, **variables)
    post "/graphql", params: { query: QUERY, variables: variables.to_json }, headers: headers
    json.dig("data", "cities")
  end

  before do
    host! "maintenance-api.rotasaude.app"
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
    login!
  end

  it "lists every registered city, ordered by slug, without opening a city connection" do
    expect(CityConnection).not_to receive(:with)

    listed = cities!

    expect(listed.map { |c| c["slug"] }).to eq(City.order(:slug).pluck(:slug))
    expect(listed.first.keys)
      .to contain_exactly(*%w[slug name uf status schemaVersion schemaBehind createdAt])
  end

  it "filters by status" do
    archived = City.where(status: "archived").pluck(:slug)
    skip "nenhuma cidade arquivada no harness" if archived.empty?

    listed = cities!(status: "ARCHIVED")

    expect(listed.map { |c| c["slug"] }).to match_array(archived)
  end

  it "never exposes a secret of the catalog" do
    cities!

    expect(response.body).not_to include("database_url")
    expect(response.body).not_to include("encryption_key")
    City.find_each { |city| expect(response.body).not_to include(city.database_url) }
  end

  it "shows a token only the cities of its scope" do
    scoped_slug = City.order(:slug).first.slug
    _token, secret = MaintenanceToken.issue!(maintainer: maintainer, name: "ci", access: "read",
                                             city_slugs: [ scoped_slug ], expires_at: 5.days.from_now)

    listed = cities!(headers: { "Authorization" => "Bearer #{secret}", "Cookie" => "" })

    expect(listed.map { |c| c["slug"] }).to eq([ scoped_slug ])
  end

  it "shows a token with an empty scope every city" do
    _token, secret = MaintenanceToken.issue!(maintainer: maintainer, name: "ci", access: "read",
                                             city_slugs: [], expires_at: 5.days.from_now)

    listed = cities!(headers: { "Authorization" => "Bearer #{secret}", "Cookie" => "" })

    expect(listed.map { |c| c["slug"] }).to eq(City.order(:slug).pluck(:slug))
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/maintenance/cities_spec.rb`
Expected: FAIL — o schema não tem `cities`, então a validação recusa a query.

- [ ] **Step 3: Implementar a consulta**

`app/queries/maintenance/city_catalog_query.rb`:

```ruby
# Catálogo de cidades para a API de manutenção (spec §8).
#
# Lê SÓ o banco de plataforma: nenhuma conexão de cidade é aberta aqui, e é por
# isso que listar 40 cidades custa uma query, não 40 conexões. O caminho para
# dentro de uma cidade é `city(slug:)`, e só ele.
#
# O escopo do token é aplicado AQUI e em `city(slug:)` — os dois pontos onde
# cidade é escolhida. Nenhum resolver de subárvore repete a regra.
module Maintenance
  class CityCatalogQuery
    def self.call(credential:, status: nil)
      scope = City.order(:slug)
      scope = scope.where(status: status) if status.present?

      scope.select { |city| credential.allows_city?(city.slug) }
    end
  end
end
```

- [ ] **Step 4: Implementar o tipo e o campo**

`app/graphql/maintenance/types/city_summary_type.rb`:

```ruby
module Maintenance
  module Types
    class CityStatusEnum < GraphQL::Schema::Enum
      graphql_name "CityStatus"
      City::STATUSES.each { |status| value status.upcase, value: status }
    end

    class CitySummaryType < BaseObject
      description "Uma cidade do catálogo. Nada aqui vem do banco da cidade."

      field :slug, String, null: false
      field :name, String, null: false
      field :uf, String, null: false
      field :status, String, null: false
      field :schema_version, String, null: true
      field :schema_behind, Boolean, null: false
      field :created_at, GraphQL::Types::ISO8601DateTime, null: false

      def schema_behind = CitySchema.behind?(object)
    end
  end
end
```

Em `QueryType`:

```ruby
      field :cities, [ Types::CitySummaryType ], null: false,
            description: "Catálogo de cidades, sem abrir conexão com nenhuma delas" do
        argument :status, Types::CityStatusEnum, required: false
      end

      def cities(status: nil)
        CityCatalogQuery.call(credential: context.fetch(:credential), status: status)
      end
```

- [ ] **Step 5: Atualizar a guarda de schema**

Em `spec/architecture/maintenance_schema_spec.rb`, acrescente a `EXPECTED_TYPES`:

```ruby
    "CitySummary" => %w[slug name uf status schemaVersion schemaBehind createdAt],
```

e acrescente `cities` à lista de campos de `Query`. Se o `graphql-ruby` publicar o enum como tipo com campos, declare-o também — use exatamente o que ele publica.

- [ ] **Step 6: Rodar, suíte completa e commit**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/maintenance spec/architecture/maintenance_schema_spec.rb spec/graphql/maintenance/analyzers_spec.rb`

Depois a suíte completa (worker parado antes, religado depois).

```bash
cd apps/api
/opt/homebrew/bin/git add app/queries/maintenance/city_catalog_query.rb app/graphql/maintenance/types/city_summary_type.rb \
  app/graphql/maintenance/types/query_type.rb spec/architecture/maintenance_schema_spec.rb \
  spec/requests/maintenance/cities_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: list the city catalogue on the maintenance API

cities reads the platform database only — listing forty cities costs one
query, not forty connections — and a service token sees exactly the
cities of its scope.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 2: `city(slug:)` — a porta única, com o escopo aplicado

**Files:**
- Create: `app/graphql/maintenance/types/city_type.rb`, `app/graphql/maintenance/types/city_channel_type.rb`, `app/graphql/maintenance/analyzers/city_budget.rb`
- Modify: `app/graphql/maintenance/types/query_type.rb`, `app/graphql/maintenance/schema.rb`, `spec/architecture/maintenance_schema_spec.rb`, `spec/graphql/maintenance/analyzers_spec.rb` (substituir o alarme)
- Test: `spec/requests/maintenance/city_spec.rb`

**Interfaces:**
- Produces:
  - `Query.city(slug: String!)` → `City` (nulo quando o escopo não permite ou o slug não existe)
  - `City`: os campos de `CitySummary` mais `ibgeCode`, `channel`, e (nas Tasks 3 e 4) `profile`, `consentTermVersion`, `protocols`, `alertRecipients`, `accounts`, `counts`, `operations`
  - `CityChannel`: `phoneNumberId`, `wabaId`, `displayPhoneNumber`, `active` — **nunca** `accessToken`
  - `Maintenance::Analyzers::CityBudget::MAX_CITIES = 5`, que recusa a operação com `extensions.code = "CITY_BUDGET_EXCEEDED"`

- [ ] **Step 1: Escrever o spec que falha**

`spec/requests/maintenance/city_spec.rb` — o mesmo arranjo de login das outras (extraia para um helper compartilhado em `spec/support` se preferir, e diga no relatório):

```ruby
require "rails_helper"

# Spec §8: `city(slug:)` é o ÚNICO caminho para dentro do banco de uma cidade, e
# é onde o escopo do token é aplicado. Teto de 5 cidades por operação.
RSpec.describe "Maintenance city", type: :request do
  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:password) { "s3nha-forte-1" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "cy-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end
  let(:city) { City.active.order(:slug).first }

  def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
  def json = JSON.parse(response.body)

  def login!
    post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now },
         headers: browser
    expect(response).to have_http_status(:ok)
  end

  def gql!(query, headers: browser, **variables)
    post "/graphql", params: { query: query, variables: variables.to_json }, headers: headers
  end

  CITY = <<~GQL
    query($slug: String!) {
      city(slug: $slug) {
        slug name uf status ibgeCode schemaVersion schemaBehind
        channel { phoneNumberId wabaId displayPhoneNumber active }
      }
    }
  GQL

  before do
    host! "maintenance-api.rotasaude.app"
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
    login!
  end

  it "answers the platform side of a city, never the channel token" do
    gql!(CITY, slug: city.slug)

    answered = json.dig("data", "city")
    expect(answered).to include("slug" => city.slug, "status" => city.status)
    expect(answered["channel"]&.keys).to satisfy { |keys| keys.nil? || keys.exclude?("accessToken") }
    expect(response.body).not_to include("access_token")
  end

  it "answers nil for a slug that does not exist" do
    gql!(CITY, slug: "cidade-que-nao-existe")

    expect(json.dig("data", "city")).to be_nil
    expect(json["errors"]).to be_nil
  end

  it "refuses a city outside a token's scope, and answers the one inside it" do
    other = City.where.not(id: city.id).order(:slug).first
    skip "harness com uma cidade só" if other.nil?

    _token, secret = MaintenanceToken.issue!(maintainer: maintainer, name: "ci", access: "read",
                                             city_slugs: [ city.slug ], expires_at: 5.days.from_now)
    bearer = { "Authorization" => "Bearer #{secret}", "Cookie" => "" }

    gql!(CITY, slug: other.slug, headers: bearer)
    expect(json.dig("data", "city")).to be_nil
    expect(json["errors"].first["extensions"]["code"]).to eq("CITY_OUT_OF_SCOPE")

    gql!(CITY, slug: city.slug, headers: bearer)
    expect(json.dig("data", "city", "slug")).to eq(city.slug)
  end

  it "refuses an operation that touches more than five cities, before executing it" do
    slugs = City.order(:slug).limit(6).pluck(:slug)
    skip "harness com menos de 6 cidades" if slugs.size < 6

    query = "{ " + slugs.each_with_index.map { |s, i| "c#{i}: city(slug: \"#{s}\") { slug }" }.join(" ") + " }"
    expect(CityConnection).not_to receive(:with)

    gql!(query)

    expect(json["errors"].first["extensions"]["code"]).to eq("CITY_BUDGET_EXCEEDED")
    expect(json["data"]).to be_nil
  end

  it "counts aliases and fragments toward the same budget" do
    slug = city.slug
    query = <<~GQL
      { a: city(slug: "#{slug}") { ...s } b: city(slug: "#{slug}") { ...s } c: city(slug: "#{slug}") { ...s }
        d: city(slug: "#{slug}") { ...s } e: city(slug: "#{slug}") { ...s } f: city(slug: "#{slug}") { ...s } }
      fragment s on City { slug }
    GQL

    gql!(query)

    expect(json["errors"].first["extensions"]["code"]).to eq("CITY_BUDGET_EXCEEDED")
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

- [ ] **Step 3: Implementar o tipo, o campo e o analisador**

`app/graphql/maintenance/types/city_channel_type.rb`:

```ruby
module Maintenance
  module Types
    class CityChannelType < BaseObject
      description "Canal de WhatsApp da cidade. O access_token é cifrado e NUNCA sai daqui."

      field :phone_number_id, String, null: false
      field :waba_id, String, null: true
      field :display_phone_number, String, null: false
      field :active, Boolean, null: false
    end
  end
end
```

`app/graphql/maintenance/types/city_type.rb` (as Tasks 3 e 4 acrescentam campos aqui):

```ruby
module Maintenance
  module Types
    class CityType < BaseObject
      description "Uma cidade. Só este tipo alcança o banco da cidade, e só por city(slug:)."

      field :slug, String, null: false
      field :name, String, null: false
      field :uf, String, null: false
      field :status, String, null: false
      field :ibge_code, String, null: true
      field :schema_version, String, null: true
      field :schema_behind, Boolean, null: false
      field :created_at, GraphQL::Types::ISO8601DateTime, null: false
      field :channel, Types::CityChannelType, null: true

      def schema_behind = CitySchema.behind?(object)

      # Canal mora na PLATAFORMA, ao lado do catálogo: sai sem abrir conexão de
      # cidade (mesma escolha de CityInventory#channel_for).
      def channel
        CityChannel.where(city_id: object.id).order(active: :desc, created_at: :desc).first
      end
    end
  end
end
```

Em `QueryType`:

```ruby
      field :city, Types::CityType, null: true,
            description: "Uma cidade. Único caminho para dentro do banco dela." do
        argument :slug, String, required: true
      end

      # O escopo do token é aplicado AQUI (spec §7): é um dos dois pontos em que
      # uma cidade é escolhida, e o único que abre conexão. Slug inexistente
      # responde nulo sem erro; slug fora do escopo responde erro explícito —
      # são coisas diferentes, e confundi-las esconderia a recusa.
      def city(slug:)
        credential = context.fetch(:credential)
        unless credential.allows_city?(slug)
          raise GraphQL::ExecutionError.new("cidade fora do escopo do token",
                                            extensions: { "code" => "CITY_OUT_OF_SCOPE" })
        end

        City.find_by(slug: slug)
      end
```

`app/graphql/maintenance/analyzers/city_budget.rb`:

```ruby
# Teto de cidades por operação (spec §8: "até 5 cidades por operação").
#
# Cada `city(slug:)` abre UMA conexão com o banco daquela cidade. Sem teto, uma
# query de vinte campos de cidade abre vinte conexões numa requisição — foi
# esgotamento de conexões que derrubou a suíte inteira no Plano 5 do banco por
# cidade, e ali era um worker, não a internet pública.
#
# Conta na ANÁLISE, antes de executar: alias e fragmento contam igual, porque o
# custo é por campo resolvido, não por slug distinto.
module Maintenance
  module Analyzers
    class CityBudget < GraphQL::Analysis::Analyzer
      MAX_CITIES = 5
      CODE = "CITY_BUDGET_EXCEEDED"

      def initialize(subject)
        super
        @city_fields = 0
      end

      def on_enter_field(node, _parent, visitor)
        return unless node.name == "city"
        return unless visitor.query.schema.query == visitor.parent_type_definition

        @city_fields += 1
      end

      def result
        return if @city_fields <= MAX_CITIES

        GraphQL::AnalysisError.new("operação toca #{@city_fields} cidades; o teto é #{MAX_CITIES}",
                                   extensions: { "code" => CODE })
      end
    end
  end
end
```

**Nota para o implementador:** confira na API real do `graphql-ruby` 2.6 como `GraphQL::AnalysisError` aceita `extensions` (os analisadores existentes em `app/graphql/maintenance/analyzers/` já fazem isso para `TOKEN_SCOPE_REFUSED` — siga o mesmo padrão deles). Registre o analisador em `schema.rb` junto dos outros.

- [ ] **Step 4: Substituir o alarme do Plano 3 pela guarda definitiva**

Em `spec/graphql/maintenance/analyzers_spec.rb`, troque o exemplo `"has no root field with a city-scoped argument yet"` por:

```ruby
  # O Plano 3 deixou aqui um alarme: "nenhum campo de raiz tem argumento de
  # cidade ainda". O Plano 4 criou `city(slug:)`, então o alarme cumpriu o
  # papel e vira a guarda definitiva — todo campo de raiz com argumento de
  # cidade PRECISA passar por Credential#allows_city?.
  it "routes every city-scoped root field through the credential's city scope" do
    root_fields = Maintenance::Schema.query.fields.merge(Maintenance::Schema.mutation.fields)
    scoped = root_fields.select { |_name, field| (field.arguments.keys & %w[slug citySlug]).any? }

    expect(scoped.keys).to contain_exactly("city")

    scoped.each_key do |name|
      source = File.read(Rails.root.join("app/graphql/maintenance/types/query_type.rb"))
      expect(source).to match(/def #{name}\b.*?allows_city\?/m),
                        "#{name} aceita argumento de cidade e não consulta allows_city?"
    end
  end
```

Se um campo de cidade nascer em outro arquivo que não `query_type.rb`, a guarda precisa procurar nele também — ajuste e diga no relatório.

- [ ] **Step 5: Atualizar a guarda de schema, rodar, suíte completa e commit**

Acrescente `City`, `CityChannel` e o campo `city` a `EXPECTED_TYPES`.

```bash
cd apps/api
/opt/homebrew/bin/git add app/graphql/maintenance app/queries/maintenance spec/architecture/maintenance_schema_spec.rb \
  spec/graphql/maintenance/analyzers_spec.rb spec/requests/maintenance/city_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: open one city at a time through city(slug:)

city(slug:) is the only way into a city's database, so it is where a
token's city scope is enforced, and an analyzer caps an operation at
five cities — one connection each — before it executes.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 3: O lado de dentro do banco da cidade

**Files:**
- Create: `app/queries/maintenance/city_reader.rb`, `app/graphql/maintenance/types/city_profile_type.rb`, `app/graphql/maintenance/types/alert_recipient_type.rb`, `app/graphql/maintenance/types/protocol_definition_type.rb`, `app/graphql/maintenance/types/city_account_type.rb`
- Modify: `app/graphql/maintenance/types/city_type.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/queries/maintenance/city_reader_spec.rb`, mais exemplos em `spec/requests/maintenance/city_spec.rb`

**Interfaces:**
- Produces:
  - `Maintenance::CityReader.call(city) { ... }` — abre `CityConnection.with`, devolve o valor do bloco, e converte falha em `Maintenance::CityReader::Unreachable` (mensagem já redigida) ou `Archived`
  - `City.profile` → `CityProfile` (`name`, `uf`, `ibgeCode`)
  - `City.consentTermVersion` → `Int`
  - `City.protocols` → `[ProtocolDefinition!]!` (`name`, `version`, `status`)
  - `City.alertRecipients` → `[AlertRecipient!]!` (`channel`, `destination`, `escalationOrder`)
  - `City.accounts` → `[CityAccount!]!` (`login`, `roles`, `active`, `mfaEnrolled`)

- [ ] **Step 1: Escrever os specs que falham**

`spec/queries/maintenance/city_reader_spec.rb` — prova o isolamento de falha **sem** passar por GraphQL:

```ruby
require "rails_helper"

# A leitura de dentro da cidade é sondagem: erro daqui é diagnóstico, não
# motivo para a resposta inteira falhar (mesma regra de CityInventory).
RSpec.describe Maintenance::CityReader do
  let(:city) { City.active.order(:slug).first }

  it "runs the block inside the city's connection" do
    result = described_class.call(city) { CityProfile.current&.name || :sem_perfil }

    expect(result).not_to be_nil
  end

  it "answers Archived for a city whose database is gone, without connecting" do
    archived = City.where(status: "archived").order(:slug).first
    skip "nenhuma cidade arquivada no harness" if archived.nil?
    expect(CityConnection).not_to receive(:with)

    expect { described_class.call(archived) { CityProfile.current } }
      .to raise_error(described_class::Archived)
  end

  it "wraps a connection failure and redacts the credential from the message" do
    allow(CityConnection).to receive(:with)
      .and_raise(PG::ConnectionBad, "connection to postgres://rota_city_x:s3nha@db:5432/rota_saude_city_x failed")

    expect { described_class.call(city) { CityProfile.current } }
      .to raise_error(described_class::Unreachable) { |e|
        expect(e.message).to include("PG::ConnectionBad")
        expect(e.message).to include("://***@")
        expect(e.message).not_to include("s3nha")
      }
  end
end
```

E, em `spec/requests/maintenance/city_spec.rb`, acrescente:

```ruby
  it "answers the configuration inside the city's database" do
    query = <<~GQL
      query($slug: String!) {
        city(slug: $slug) {
          profile { name uf ibgeCode }
          consentTermVersion
          protocols { name version status }
          alertRecipients { channel destination escalationOrder }
          accounts { login roles active mfaEnrolled }
        }
      }
    GQL

    gql!(query, slug: city.slug)

    answered = json.dig("data", "city")
    expect(json["errors"]).to be_nil
    expect(answered).to have_key("profile")
    expect(answered["accounts"]).to be_an(Array)
  end

  it "reports an unreachable city as a field error, redacted, without failing the operation" do
    other = City.active.where.not(id: city.id).order(:slug).first
    skip "harness com uma cidade ativa só" if other.nil?

    allow(Maintenance::CityReader).to receive(:call).and_call_original
    allow(Maintenance::CityReader).to receive(:call).with(having_attributes(slug: other.slug))
      .and_raise(Maintenance::CityReader::Unreachable, "PG::ConnectionBad: connection to ://***@db failed")

    query = <<~GQL
      { ok: city(slug: "#{city.slug}") { profile { name } }
        bad: city(slug: "#{other.slug}") { profile { name } } }
    GQL
    gql!(query)

    expect(json.dig("data", "ok", "profile")).not_to be_nil
    expect(json.dig("data", "bad", "profile")).to be_nil
    error = json["errors"].find { |e| e["path"]&.include?("bad") }
    expect(error["extensions"]["code"]).to eq("CITY_UNREACHABLE")
    expect(error["message"]).to include("://***@")
  end

  it "answers CITY_ARCHIVED for the inner fields of an archived city, and still answers the platform ones" do
    archived = City.where(status: "archived").order(:slug).first
    skip "nenhuma cidade arquivada no harness" if archived.nil?

    gql!('query($slug: String!) { city(slug: $slug) { slug status profile { name } } }', slug: archived.slug)

    expect(json.dig("data", "city", "status")).to eq("archived")
    expect(json.dig("data", "city", "profile")).to be_nil
    expect(json["errors"].first["extensions"]["code"]).to eq("CITY_ARCHIVED")
  end

  it "never answers citizen content or a secret from inside the city" do
    query = <<~GQL
      query($slug: String!) { city(slug: $slug) { accounts { login } alertRecipients { destination } } }
    GQL

    gql!(query, slug: city.slug)

    %w[password_digest otp_secret database_url encryption_key access_token].each do |forbidden|
      expect(response.body).not_to include(forbidden)
    end
  end
```

- [ ] **Step 2: Rodar e confirmar que falham**

- [ ] **Step 3: Implementar o leitor**

`app/queries/maintenance/city_reader.rb`:

```ruby
# Entrada única no banco de uma cidade, para a API de manutenção (spec §8).
#
# Duas regras que este arquivo existe para sustentar:
#
#   1. Cidade inalcançável NÃO derruba a operação. É justamente quando algo
#      quebrou que a ferramenta precisa responder — as cidades que respondem
#      respondem, e a que falhou vira erro NAQUELE campo.
#   2. A mensagem passa por CitySchema.redact. PG::ConnectionBad traz a URL de
#      conexão inteira, com a senha do role da cidade; sem isto, o caminho de
#      erro — o menos exercitado — publicaria o segredo que todo o resto esconde.
module Maintenance
  class CityReader
    # Status sem banco para conectar: tentar é garantia de erro, e erro previsto
    # não é diagnóstico (mesma lista de CityInventory::SKIPPED_STATUSES).
    ARCHIVED_STATUSES = %w[archived].freeze

    class Archived < StandardError; end
    class Unreachable < StandardError; end

    def self.call(city)
      raise Archived, "cidade #{city.status}: sem banco para conectar" if ARCHIVED_STATUSES.include?(city.status)

      CityConnection.with(city) { yield }
    rescue Archived
      raise
    rescue StandardError => e
      raise Unreachable, "#{e.class}: #{CitySchema.redact(e.message)}"
    end
  end
end
```

- [ ] **Step 4: Implementar os tipos e os campos**

Os tipos `CityProfileType` (`name`, `uf`, `ibgeCode`), `AlertRecipientType` (`channel`, `destination`, `escalationOrder`), `ProtocolDefinitionType` (`name`, `version`, `status`) e `CityAccountType` (`login`, `roles`, `active`, `mfaEnrolled`) são objetos simples, no formato dos tipos já existentes.

**`CityAccountType` é o ponto sensível:** `login` é o e-mail de uma conta de **staff da prefeitura**, não de cidadão — a mesma informação que a tela `/maintenance` já mostra. `password_digest`, `otp_secret` e recovery codes ficam de fora por regra; o que serve para operar é saber **se** a conta exige MFA.

Em `CityType`, cada campo de dentro da cidade resolve por `CityReader`, convertendo as duas exceções em erro de campo:

```ruby
      field :profile, Types::CityProfileType, null: true
      field :consent_term_version, Integer, null: true
      field :protocols, [ Types::ProtocolDefinitionType ], null: false
      field :alert_recipients, [ Types::AlertRecipientType ], null: false
      field :accounts, [ Types::CityAccountType ], null: false

      def profile = inside { CityProfile.current }
      def consent_term_version = inside { ConsentTerm.maximum(:version) }
      def protocols = inside { ProtocolDefinition.active.order(:name).to_a }
      def alert_recipients = inside { AlertRecipient.active.order(:escalation_order).to_a }

      def accounts
        inside do
          User.order(:email_address).map do |user|
            { login: user.email_address, roles: user.memberships.active.order(:role).pluck(:role),
              active: user.active?, mfa_enrolled: user.mfa_enrolled? }
          end
        end
      end

      private

      # Um só ponto converte as duas falhas previstas em erro de CAMPO: a
      # operação segue, e o cliente vê exatamente qual cidade não respondeu.
      def inside(&block)
        CityReader.call(object, &block)
      rescue CityReader::Archived => e
        raise GraphQL::ExecutionError.new(e.message, extensions: { "code" => "CITY_ARCHIVED" })
      rescue CityReader::Unreachable => e
        raise GraphQL::ExecutionError.new(e.message, extensions: { "code" => "CITY_UNREACHABLE" })
      end
```

**Atenção ao custo:** cada campo destes abre a conexão de novo. Com cinco campos numa cidade são cinco entradas em `CityConnection.with` — barato (o pool já está registrado), mas não gratuito. Se preferir resolver a subárvore inteira numa entrada só, faça, prove por spec que a conexão é aberta uma vez por cidade e diga no relatório; **não** troque a regra do teto de 5.

- [ ] **Step 5: Atualizar a guarda de schema, rodar, suíte completa e commit**

Acrescente os quatro tipos novos e os cinco campos a `EXPECTED_TYPES`. Confira que nenhum nome novo bate nos `FORBIDDEN_FRAGMENTS` — `destination` e `login` não batem; se algum bater, **pare e reporte** em vez de acrescentar à allowlist por conta própria.

```bash
cd apps/api
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: read a city's configuration through the maintenance API

Profile, consent term, protocols, alert recipients and staff accounts
come from inside the city's database, one entry point, with an
unreachable city reported as a field error whose message is redacted —
a connection failure carries the role's password.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 4: Sinais de operação e contagens

**Files:**
- Create: `app/graphql/maintenance/types/city_counts_type.rb`, `app/graphql/maintenance/types/city_operations_type.rb`
- Modify: `app/graphql/maintenance/types/city_type.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/city_operations_spec.rb`

**Interfaces:**
- Produces:
  - `City.counts` → `CityCounts`: `users`, `conversations`, `triages`, `inboundMessages`, `reportSnapshots`, `consents` — **só números**
  - `City.operations` → `CityOperations`: `domainEvents` (últimos N: `name`, `occurredAt`, `publishedAt` — **nunca** payload), `reportSnapshots` (`id`, `createdAt`, `expiresAt` — nunca conteúdo), `dashboardMetrics` (`name`, `value`, `computedAt`), `failedJobs` (`className`, `failedAt`, `error` redigido)

- [ ] **Step 1: Escrever o spec que falha**

`spec/requests/maintenance/city_operations_spec.rb`, cobrindo:

```ruby
require "rails_helper"

# Spec §8: dados operacionais entram como metadado e contagem. O payload de um
# evento de domínio é dado de cidadão — ele nunca sai daqui.
RSpec.describe "Maintenance city operations", type: :request do
  # (mesmo arranjo de login dos outros specs de manutenção)

  it "counts citizen data instead of showing it" do
    gql!('query($slug: String!) { city(slug: $slug) { counts { users conversations triages inboundMessages reportSnapshots consents } } }',
         slug: city.slug)

    counts = json.dig("data", "city", "counts")
    expect(counts.values).to all(be_a(Integer))
  end

  it "lists domain events as metadata, never their payload" do
    gql!('query($slug: String!) { city(slug: $slug) { operations { domainEvents { name occurredAt publishedAt } } } }',
         slug: city.slug)

    expect(json["errors"]).to be_nil
    expect(response.body).not_to include("payload")
  end

  it "lists report snapshots and dashboard metrics without their content" do
    gql!('query($slug: String!) { city(slug: $slug) { operations { reportSnapshots { id createdAt expiresAt } dashboardMetrics { name value computedAt } } } }',
         slug: city.slug)

    expect(json["errors"]).to be_nil
    %w[payload body signature token].each { |forbidden| expect(response.body).not_to include(forbidden) }
  end

  it "lists failed jobs with a redacted error" do
    gql!('query($slug: String!) { city(slug: $slug) { operations { failedJobs { className failedAt error } } } }',
         slug: city.slug)

    expect(json["errors"]).to be_nil
    json.dig("data", "city", "operations", "failedJobs").to_a.each do |job|
      expect(job["error"].to_s).not_to match(%r{://[^/\s@]+:[^/\s@]+@})
    end
  end

  it "reports an unreachable city on the operations subtree too" do
    # mesmo padrão do spec de configuração: stub de CityReader levantando
    # Unreachable, e o erro sai com CITY_UNREACHABLE naquele campo
  end
end
```

Complete o arranjo de login e o último exemplo seguindo `spec/requests/maintenance/city_spec.rb`; eles são os mesmos.

- [ ] **Step 2: Rodar e confirmar que falha**

- [ ] **Step 3: Implementar**

`CityCountsType` e `CityOperationsType` são objetos simples cujos resolvers rodam dentro de `CityReader` (reaproveite o `inside` da Task 3, extraindo-o para um módulo se ficar mais limpo — diga no relatório o que escolheu).

Limites que o implementador precisa respeitar:
- **`domainEvents` devolve no máximo 50**, os mais recentes por `occurred_at`, e **nunca** o `payload`.
- **`reportSnapshots`** devolve id e datas; nunca o conteúdo nem a assinatura.
- **`failedJobs`** lê as execuções falhas do Solid Queue **do banco da cidade**, no máximo 50, com a mensagem passada por `CitySchema.redact`.
- **`counts`** usa `count` direto, sem carregar linha.

- [ ] **Step 4: Atualizar a guarda de schema, rodar, suíte completa e commit**

---

### Task 5: Guardas de leitura

**Files:**
- Modify: `spec/architecture/maintenance_schema_spec.rb`
- Test: o mesmo arquivo, mais `spec/requests/maintenance/city_spec.rb`

Esta task não acrescenta comportamento: ela fecha as guardas que impedem a fatia de crescer errado.

- [ ] **Step 1: Guarda "sem conteúdo de cidadão"**

Acrescente ao spec de schema um exemplo que percorre **todos** os tipos publicados e recusa qualquer campo cujo tipo seja `String` e cujo nome esteja numa lista de conteúdo (`body`, `raw`, `evidence`, `response`, `context`, `phone`, `message`, `text`). Os `FORBIDDEN_FRAGMENTS` já cobrem parte disso; o que falta é a afirmação explícita de que **nenhum tipo novo de cidade** publica conteúdo, com o comentário dizendo que o mascaramento está adiado e que é este spec que segura a porta.

- [ ] **Step 2: Guarda do orçamento de conexão**

Um exemplo que executa uma operação com 5 cidades e conta quantas vezes `CityConnection.with` é chamado, afirmando que é no máximo uma por cidade **por campo resolvido** (ou uma por cidade, se a Task 3 tiver consolidado). O número exato vem do que a Task 3 decidiu — escreva o que for verdade e comente por quê.

- [ ] **Step 3: Guarda do caminho único**

Um exemplo que afirma que `CityConnection.with` e `CityReader.call` **não** são chamados de nenhum resolver fora de `CityType` — varra `app/graphql/maintenance/` e recuse ocorrências em outros arquivos, com o comentário explicando que o escopo do token é aplicado em `city(slug:)` e que um segundo caminho para o banco de cidade o contornaria.

- [ ] **Step 4: Rodar, suíte completa e commit**

---

## Verificação final do plano

- [ ] Suíte completa com o worker parado: 0 falhas.
- [ ] `cities` não abre nenhuma conexão de cidade (provado por spec).
- [ ] Uma operação com 6 cidades é recusada antes de executar.
- [ ] Uma cidade inalcançável não derruba a operação, e a mensagem sai redigida.
- [ ] Um token com escopo enxerga só as suas cidades, em `cities` **e** em `city(slug:)`.
- [ ] Nenhum campo novo publica segredo ou conteúdo de cidadão — as três guardas da Task 5 passam.
- [ ] `CityInventory` e a tela `/maintenance` seguem intocados: `git diff main -- app/services/city_inventory.rb app/controllers/maintenance_controller.rb` vazio.

## Fora deste plano

- **Mutations por módulo** (perfil, canal, protocolos, destinatários, jobs, projeções): Plano 5.
- **SDL publicado em `contracts`**: Plano 6.
- **Conteúdo de cidadão e mascaramento:** adiado, ver o registro de 2026-09-17.
- **Aposentar `CityInventory`** em favor do GraphQL: decisão própria, depois que a ferramenta tiver frontend.
- **Job de CI do boot de produção**, chave de staging no cofre e mailer do convite: pendências abertas dos planos anteriores.
