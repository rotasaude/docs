# Plano 2 — Fundação da API de manutenção (fatia 2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** um mantenedor consegue entrar na API de manutenção com senha + TOTP, num host próprio que só existe em development e staging, e toda entrada, falha e convite fica registrada numa auditoria que nem ele consegue apagar. O GraphQL sobe com uma única query, `me`.

**Architecture:** identidade própria (`Maintainer`) no banco de plataforma, espelhando `Operator` sem herdar dele. A rota só é desenhada quando `MaintenanceApi.enabled?` permite, dentro da constraint de host `maintenance-api.*`; em produção com a chave ligada, o processo não sobe. A auditoria escreve em `platform_events` com um par tentativa/resultado e é protegida por trigger no Postgres.

**Tech Stack:** Rails 8.1.3 (API-only), `graphql-ruby`, `rotp`/`bcrypt`, Postgres 16/15, RSpec.

**Spec:** `docs/superpowers/specs/2026-09-17-maintenance-graphql-api-design.md` (§3 hosts, §5 trava de ativação, §6 identidade e sessão, §8 endpoint e limites, §9 auditoria, §10 testes, §11 fatia 2).

**Plano anterior:** Plano 1 (ambiente staging) mergeado em `main` local do `api` (merge `1452053`, suíte 916/0). Este plano começa daí.

## Decisões minhas na escrita (cada uma com o custo se estiver errada)

1. **Gerenciar mantenedores fica para o Plano 3.** A fatia 2 entrega `rake maintainer:invite` (que roda no servidor), o fluxo de aceite e a query `me`. As mutations `inviteMaintainer` / `deactivateMaintainer` vão junto com os tokens, no Plano 3, porque as três precisam exatamente da mesma mecânica de mutation auditada com step-up de TOTP. **Custo se errado:** até o Plano 3, convidar outro mantenedor exige acesso ao servidor.
2. **A auditoria grava `maintainer_id`, nunca o e-mail.** A spec §9 mostra `"login"` no payload, mas `platform_events` é governado pela Ruling R18, que recusa payload com chave contendo `email` — e o espírito da regra é não acumular dado pessoal ali. O login é resolvido na leitura, juntando com `maintainers`. **Custo se errado:** quem ler `platform_events` cru vê o id, não o e-mail, e precisa de um join para o nome.
3. **Sem mailer nesta fatia.** `rake maintainer:invite` imprime o link do convite no terminal de quem rodou (com o e-mail mascarado, como `city:invite_admin` já faz). O `InvitationMailer` de cidade não serve, e o frontend que receberia o link só existe depois. **Custo se errado:** o convite é entregue fora de banda até haver mailer próprio.
4. **`rate_limit` por IP é declarado, mas não tem spec.** O `rate_limit` do Rails usa o cache store, e em test o store é `:null_store` (`config/environments/test.rb`), então um spec não observaria nada. O bloqueio que a fatia PROVA é o por conta, que vive no banco. É o mesmo padrão do `Operators::SessionsController`, que também declara `rate_limit` sem spec. **Custo se errado:** uma regressão que apague a linha do `rate_limit` não quebra a suíte.
5. **Dois rótulos reservados, não um:** `maintenance` (frontend) e `maintenance-api` (API). Reservar só o da API deixaria alguém cadastrar a cidade `maintenance` e ficar com o host do frontend. **Custo se errado:** dois slugs a menos disponíveis para cidade.
6. **O trigger de imutabilidade não aparece em `db/platform_schema.rb`.** O dump em Ruby não representa trigger. O banco de test é construído por `db:migrate:platform` (é o que a CI faz), então ele existe onde a suíte roda; um spec confere a existência do trigger no catálogo do Postgres, para que uma reconstrução por `schema:load` apareça como falha em vez de silêncio. **Custo se errado:** quem carregar o schema puro fica sem o trigger e o spec avisa.

## Global Constraints

Valem para TODA task:

- **Nunca imprimir segredo:** `otp_secret`, token de convite em claro, `password_digest`, conteúdo de credentials, chave de cifra. Id, contagem, booleano e e-mail mascarado, sim.
- **Nada de dado de cidadão** em nenhum payload, resposta ou log desta fatia. A API não lê banco de cidade neste plano.
- **Ruling R18:** todo nome novo de `PlatformEvent` precisa entrar em `R18_PLATFORM_EVENT_NAMES` (`spec/events/platform_event_payload_guard_spec.rb`), e o payload não pode ter chave contendo `email`, `cpf`, `phone`, `name`, `body`, `wa_id`, `provider_uid`, nem a chave `from`. **Declare o nome; nunca afrouxe a guarda.**
- **Ambiente publicado pergunta `Rota.deployed?`** (`lib/rota.rb`), nunca `Rails.env.production?`. `spec/architecture/deployed_environment_guard_spec.rb` quebra a suíte se o literal voltar — ele varre `app`, `config` (fora de `config/environments/`), `lib`, `db`, `bin`, `script`, `Rakefile` e `config.ru`.
- **Commits em inglês, Conventional Commits** com tipo por extenso, terminando com a linha exata `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- **Comandos Ruby/Rails/rspec rodam no container**, a partir da raiz do monorepo: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Nunca no host (o Ruby do host é 3.4; o Gemfile exige 3.3.6).
- **Migrations de plataforma:** `docker compose exec -T api bin/rails db:migrate:platform`. Isso atualiza `db/platform_schema.rb`, que entra no commit. O banco de **test** também precisa migrar: `docker compose exec -T -e RAILS_ENV=test -e POSTGRES_PASSWORD=postgres api bin/rails db:migrate:platform`.
- **`git` do PATH está quebrado:** use `/opt/homebrew/bin/git`, a partir de `apps/api`. **Nunca dar push.** Trabalhe na branch `feat/maintenance-api-foundation`, criada de `main`.
- **Suíte completa:** `docker compose stop worker` antes, timeout ≥ 600000 ms, `docker compose start worker` depois. Baseline de entrada: **916 exemplos, 0 falhas**.
- **Nada destrutivo** em `rota_saude_platform_development` / `_test` nem nos bancos das cidades de dev (`curitiba`, `maringa` ativas; `cascavel`, `londrina` arquivadas). Este plano só acrescenta tabelas ao banco de plataforma.
- **Nunca rodar `start.sh`.**

## Fatos verificados (não re-descobrir)

- **`Operator`** (`app/models/operator.rb`) é o espelho a seguir: `has_secure_password`, `encrypts :otp_secret, key_provider: PlatformKeyProvider.new`, `normalizes :email_address`, índice único em `lower(email_address)`, `deactivate!` que apaga as sessões.
- **`OperatorAuthentication`** (`app/controllers/concerns/operator_authentication.rb`) traz o padrão de cookie e sessão: `COOKIE`, `PENDING_MFA_WINDOW = 10.minutes`, `OPERATOR_SESSION_TTL = 12.hours`, `MAX_TOTP_ATTEMPTS = 5`, `cookies.signed`, `httponly: true`, `secure: Rota.deployed?`, **nunca `domain:`** (`spec/architecture/cookie_domain_spec.rb` falha se alguém escrever `domain:` ou configurar session store).
- **`Operators::SessionsController`** mostra o fluxo senha → TOTP com os detalhes que importam: `update_all` condicional a `mfa_verified_at: nil` (corrida entre challenge e erro), auditoria na mesma transação do carimbo, `rate_limit to: 10, within: 3.minutes`, vínculo entre `session_id` do corpo e o cookie.
- **`Mfa::Enroll.call(user)`** devolve `{secret:, otpauth_uri:, recovery_codes:}` e grava `otp_secret`, `otp_enabled: false`, `otp_recovery_codes` (hash BCrypt). **`Mfa::Verify.call(user, code:)`** valida TOTP com drift de 30s e consome recovery code. Os dois esperam um objeto com `otp_secret`, `otp_recovery_codes` e `email_address`.
- **`Platform.audit(name, **payload)`** (`app/events/platform.rb`) cria `PlatformEvent` com `occurred_at`. `platform_events` tem `id`, `name`, `payload jsonb`, `occurred_at`, `published_at`, timestamps.
- **`CityCatalog`** (`app/models/city_catalog.rb`): `RESERVED = %w[admin api auth www]`, `label_for`, `console_host?`, `auth_host?`. `CityDatabase.valid_slug?` recusa slug em `RESERVED`.
- **Constraints de host** vivem em `app/constraints/` (`PlatformConsoleHost`, `PlatformAuthHost`), cada uma com um `self.matches?(request)` de uma linha.
- **`Operators::BaseController`** herda de `ActionController::API`, inclui `ActionController::Cookies`, checa o host num `before_action` **antes** de incluir a autenticação (host errado → 404, nunca 401).
- **Request specs** trocam host com `host!` (ex.: `host! "admin.rotasaude.app"`), e o `before` global aponta para o host da cidade de teste. Não há factory de operador: os specs criam com `Operator.create!`.
- **`rake city:invite_admin`** (`lib/tasks/city.rake`) é o padrão da rake de convite: valida e-mail com `URI::MailTo::EMAIL_REGEXP`, `abort` com mensagem prefixada, mascara e-mail (`p***@dominio`), audita com `Platform.audit`.
- **CORS** (`config/initializers/cors.rb`) tem um bloco `allow` com `origins` dinâmico (host de cidade) e `ALLOWED_ORIGINS`; os `resource` declaram `credentials: true` para `/session` e `/admin/api/*`.
- **Migration de plataforma** mais recente: `db/platform_migrate/20260916000003_create_solid_cache_entries.rb`. Tabelas de plataforma usam `id: :uuid, default: -> { "gen_random_uuid()" }`.
- **`config/environments/test.rb`** usa `cache_store = :null_store` (por isso `rate_limit` não é observável em spec) e `eager_load` só com `CI`.
- **`lib/` não é autoloaded**: `lib/rota.rb` é `require_relative`ado em `config/application.rb`. `config/routes.rb` roda depois, então pode usar constantes de `lib/` já requeridas ali.

---

## File Structure

**apps/api**
- Create: `lib/maintenance_api.rb`, `config/initializers/01_maintenance_api.rb`, `app/constraints/maintenance_api_host.rb`, `db/platform_migrate/20260917000001_create_maintainers.rb`, `db/platform_migrate/20260917000002_maintenance_events_immutable.rb`, `app/models/maintainer.rb`, `app/models/maintainer_session.rb`, `app/models/maintainer_invitation.rb`, `app/events/maintenance_audit.rb`, `app/controllers/concerns/maintainer_authentication.rb`, `app/controllers/maintenance/base_controller.rb`, `app/controllers/maintenance/sessions_controller.rb`, `app/controllers/maintenance/invitations_controller.rb`, `app/controllers/maintenance/graphql_controller.rb`, `app/graphql/maintenance/schema.rb`, `app/graphql/maintenance/types/{base_object,query_type,maintainer_type}.rb`, `lib/tasks/maintainer.rake`
- Create (specs): `spec/lib/maintenance_api_spec.rb`, `spec/architecture/maintenance_api_route_spec.rb`, `spec/models/maintainer_spec.rb`, `spec/events/maintenance_audit_spec.rb`, `spec/requests/maintenance/sessions_spec.rb`, `spec/requests/maintenance/invitations_spec.rb`, `spec/requests/maintenance/graphql_spec.rb`, `spec/tasks/maintainer_rake_spec.rb`
- Modify: `config/application.rb`, `config/routes.rb`, `config/initializers/cors.rb`, `app/models/city_catalog.rb`, `db/platform_schema.rb` (gerado), `Gemfile`, `Gemfile.lock`, `spec/events/platform_event_payload_guard_spec.rb`

Branch: `feat/maintenance-api-foundation`, criada de `main` (merge `1452053`).

---

### Task 1: Trava de ativação e host próprio

**Files:**
- Create: `lib/maintenance_api.rb`, `config/initializers/01_maintenance_api.rb`, `app/constraints/maintenance_api_host.rb`
- Modify: `config/application.rb` (ao lado do `require_relative "../lib/rota"`), `app/models/city_catalog.rb`
- Test: `spec/lib/maintenance_api_spec.rb`, `spec/architecture/maintenance_api_route_spec.rb`

**Interfaces:**
- Consumes: `Rota.deployed?` (`lib/rota.rb`), `CityCatalog.label_for`.
- Produces:
  - `MaintenanceApi::ALLOWED_ENVS` → `%w[development staging]`
  - `MaintenanceApi.enabled?(env: Rails.env, flag: ENV["MAINTENANCE_API_ENABLED"]) → Boolean`
  - `MaintenanceApi.check_boot!(env:, flag:) → nil`, levanta `MaintenanceApi::EnabledInProduction`
  - `MaintenanceApiHost.matches?(request) → Boolean`
  - `CityCatalog.maintenance_api_host?(host) → Boolean`; `CityCatalog::RESERVED` passa a incluir `maintenance` e `maintenance-api`

- [ ] **Step 1: Escrever os specs que falham**

`spec/lib/maintenance_api_spec.rb`:

```ruby
require "rails_helper"

# Spec da API de manutenção §5: a rota só é desenhada em development e staging,
# com a chave ligada — e em produção a chave ligada DERRUBA o boot. A decisão é
# uma função pura porque a ausência da rota em produção precisa ser provável por
# spec, e a suíte roda em test.
RSpec.describe MaintenanceApi do
  describe ".enabled?" do
    it "is true in development and staging only with the flag on" do
      expect(described_class.enabled?(env: "development", flag: "true")).to be(true)
      expect(described_class.enabled?(env: "staging", flag: "true")).to be(true)
      expect(described_class.enabled?(env: "development", flag: nil)).to be(false)
      expect(described_class.enabled?(env: "staging", flag: "false")).to be(false)
    end

    it "is never true in production, whatever the flag says" do
      expect(described_class.enabled?(env: "production", flag: "true")).to be(false)
      expect(described_class.enabled?(env: "production", flag: nil)).to be(false)
    end

    it "is always true in test, where the request specs live" do
      expect(described_class.enabled?(env: "test", flag: nil)).to be(true)
    end
  end

  describe ".check_boot!" do
    it "refuses to boot production with the flag on" do
      expect { described_class.check_boot!(env: "production", flag: "true") }
        .to raise_error(MaintenanceApi::EnabledInProduction, /MAINTENANCE_API_ENABLED/)
    end

    it "lets every other combination boot" do
      [ %w[production false], %w[staging true], %w[development true], %w[test false] ].each do |env, flag|
        expect { described_class.check_boot!(env: env, flag: flag) }.not_to raise_error
      end
    end
  end
end
```

`spec/architecture/maintenance_api_route_spec.rb`:

```ruby
require "rails_helper"

# Spec da API de manutenção §3: o host maintenance-api.* é da PLATAFORMA, e
# maintenance.* é do frontend. Nenhum dos dois pode resolver cidade — senão um
# slug cadastrado roubaria o host da ferramenta que dá acesso a todas as cidades.
RSpec.describe "Maintenance API host" do
  it "reserves both maintenance labels, so no city can take them" do
    expect(CityCatalog::RESERVED).to include("maintenance", "maintenance-api")
    expect(CityDatabase.valid_slug?("maintenance")).to be(false)
    expect(CityDatabase.valid_slug?("maintenance-api")).to be(false)
  end

  it "resolves no city on either host" do
    expect(CityCatalog.find_by_host("maintenance-api.rotasaude.app")).to be_nil
    expect(CityCatalog.find_by_host("maintenance.rotasaude.app")).to be_nil
  end

  it "matches the API host only" do
    expect(CityCatalog.maintenance_api_host?("maintenance-api.rotasaude.app")).to be(true)
    expect(CityCatalog.maintenance_api_host?("maintenance.rotasaude.app")).to be(false)
    expect(CityCatalog.maintenance_api_host?("admin.rotasaude.app")).to be(false)
    expect(CityCatalog.maintenance_api_host?("curitiba.rotasaude.app")).to be(false)
  end

  it "matches requests through the route constraint" do
    request = ActionDispatch::TestRequest.create("HTTP_HOST" => "maintenance-api.rotasaude.app")
    other   = ActionDispatch::TestRequest.create("HTTP_HOST" => "admin.rotasaude.app")

    expect(MaintenanceApiHost.matches?(request)).to be(true)
    expect(MaintenanceApiHost.matches?(other)).to be(false)
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falham**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/lib/maintenance_api_spec.rb spec/architecture/maintenance_api_route_spec.rb`
Expected: FAIL com `NameError: uninitialized constant MaintenanceApi` e `MaintenanceApiHost`.

- [ ] **Step 3: Implementar `lib/maintenance_api.rb`**

```ruby
# Quando a API de manutenção existe (spec da API de manutenção §5).
#
# Fica em lib/ e é `require`ado por config/application.rb, como lib/rota.rb:
# config/routes.rb pergunta isto para decidir se DESENHA a rota, e rota que não
# existe devolve 404 no roteamento, sem depender de nenhum controller lembrar de
# negar — a mesma regra da tela /maintenance.
module MaintenanceApi
  # Ambientes em que a ferramenta pode existir. Produção está fora por decisão do
  # usuário: lá o acesso é a dado real e a decisão é própria, futura.
  ALLOWED_ENVS = %w[development staging].freeze

  FLAG = "MAINTENANCE_API_ENABLED"

  class EnabledInProduction < StandardError; end

  module_function

  # test SEMPRE liga: cookie, CSRF, bloqueio de conta e escopo precisam de
  # request spec, e test não é ambiente publicado. A ausência em produção é
  # provada aqui, nesta função, e não pela rota.
  def enabled?(env: Rails.env, flag: ENV[FLAG])
    return true if env.to_s == "test"

    ALLOWED_ENVS.include?(env.to_s) && flag.to_s == "true"
  end

  # Falha fechada no boot: chave ligada num processo de produção é erro de
  # deploy, e erro de deploy tem que aparecer no deploy.
  def check_boot!(env: Rails.env, flag: ENV[FLAG])
    return unless env.to_s == "production" && flag.to_s == "true"

    raise EnabledInProduction,
          "#{FLAG}=true em produção — a API de manutenção não existe em produção (spec §5). " \
          "Remova a variável do deploy."
  end
end
```

- [ ] **Step 4: Carregar e travar o boot**

Em `config/application.rb`, logo depois do `require_relative "../lib/rota"`:

```ruby
# config/routes.rb pergunta MaintenanceApi.enabled? para decidir se desenha a
# rota, e as rotas são carregadas antes do Zeitwerk terminar.
require_relative "../lib/maintenance_api"
```

`config/initializers/01_maintenance_api.rb`:

```ruby
# Trava de boot da API de manutenção (spec §5). Prefixo 01_: depois da checagem
# de credentials (00_) e antes de qualquer initializer que monte rota.
MaintenanceApi.check_boot!
```

- [ ] **Step 5: Reservar os dois rótulos e reconhecer o host**

Em `app/models/city_catalog.rb`, troque a constante:

```ruby
  # maintenance.* (frontend) e maintenance-api.* (API) são da plataforma: a
  # ferramenta alcança TODAS as cidades, então nenhuma cidade pode responder
  # nesses hosts (spec da API de manutenção §3).
  RESERVED = %w[admin api auth www maintenance maintenance-api].freeze
```

E, ao lado de `console_host?`:

```ruby
    # Host da API de manutenção (maintenance-api.*). Reservado: nunca resolve cidade.
    def maintenance_api_host?(host)
      label_for(host) == "maintenance-api"
    end
```

`app/constraints/maintenance_api_host.rb`:

```ruby
# Constraint de rota: casa requisições dirigidas à API de manutenção
# (maintenance-api.*), irmã de PlatformConsoleHost. Casada com
# MaintenanceApi.enabled?, que decide se a rota chega a ser desenhada.
class MaintenanceApiHost
  def self.matches?(request)
    CityCatalog.maintenance_api_host?(request.host)
  end
end
```

- [ ] **Step 6: Rodar os specs da task e os vizinhos que a mudança toca**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/lib/maintenance_api_spec.rb spec/architecture/maintenance_api_route_spec.rb spec/requests/city_resolution_spec.rb spec/requests/cors_spec.rb spec/services/city_database_spec.rb`
Expected: PASS, 0 falhas.

- [ ] **Step 7: Suíte completa e commit**

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
```

```bash
cd apps/api
/opt/homebrew/bin/git add lib/maintenance_api.rb config/initializers/01_maintenance_api.rb \
  app/constraints/maintenance_api_host.rb config/application.rb app/models/city_catalog.rb \
  spec/lib/maintenance_api_spec.rb spec/architecture/maintenance_api_route_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: gate the maintenance API by environment and host

The route only exists in development and staging with the flag on, and
in test where the request specs live; production with the flag on fails
at boot. maintenance and maintenance-api become reserved labels, so no
city can answer on the tool's hosts.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 2: Tabelas e modelos do mantenedor

**Files:**
- Create: `db/platform_migrate/20260917000001_create_maintainers.rb`, `app/models/maintainer.rb`, `app/models/maintainer_session.rb`, `app/models/maintainer_invitation.rb`
- Modify: `db/platform_schema.rb` (gerado pela migration)
- Test: `spec/models/maintainer_spec.rb`

**Interfaces:**
- Consumes: `PlatformRecord`, `PlatformKeyProvider`, `Mfa::Enroll` / `Mfa::Verify`.
- Produces:
  - `Maintainer`: `email_address`, `password_digest`, `otp_secret` (cifrado), `otp_recovery_codes`, `otp_enabled_at`, `deactivated_at`, `failed_attempts`, `locked_until`, `invited_by_id`; `has_many :maintainer_sessions`, `has_many :maintainer_invitations`; `Maintainer::LOCKOUT_ATTEMPTS = 5`, `LOCKOUT_WINDOW = 15.minutes`; `.active` scope; `#active?`, `#enrolled?`, `#locked?`, `#register_failure!`, `#clear_failures!`, `#deactivate!`, `#last_active?`
  - `MaintainerSession`: `maintainer_id`, `mfa_verified_at`, `last_seen_at`, `totp_attempts`, `ip_address`, `user_agent`
  - `MaintainerInvitation`: `maintainer_id`, `token_digest`, `expires_at`, `used_at`; `MaintainerInvitation::TTL = 24.hours`; `.digest_for(token)`, `.issue!(maintainer:)` → `[invitation, token_em_claro]`, `#usable?`

- [ ] **Step 1: Escrever o spec que falha**

`spec/models/maintainer_spec.rb`:

```ruby
require "rails_helper"

# Spec da API de manutenção §6: identidade própria, separada de Operator, com
# poderes totais — então a força está na autenticação: TOTP obrigatório,
# bloqueio por conta e desativação que mata sessões na hora.
RSpec.describe Maintainer do
  include ActiveSupport::Testing::TimeHelpers

  def build_maintainer(email: "m-#{SecureRandom.hex(3)}@rotasaude.app")
    described_class.create!(email_address: email)
  end

  it "normalizes and refuses a duplicate e-mail, whatever the case" do
    build_maintainer(email: "Alguem@Rotasaude.APP")

    expect(described_class.find_by(email_address: "alguem@rotasaude.app")).to be_present
    expect { build_maintainer(email: "ALGUEM@rotasaude.app") }.to raise_error(ActiveRecord::RecordInvalid)
  end

  it "is not enrolled until it has a password and a confirmed TOTP" do
    maintainer = build_maintainer
    expect(maintainer.enrolled?).to be(false)

    maintainer.update!(password: "s3nha-forte-1")
    expect(maintainer.enrolled?).to be(false)

    Mfa::Enroll.call(maintainer)
    expect(maintainer.reload.enrolled?).to be(false)

    maintainer.update!(otp_enabled_at: Time.current)
    expect(maintainer.reload.enrolled?).to be(true)
  end

  it "locks the account after five failures and stays locked with the right password" do
    maintainer = build_maintainer

    (described_class::LOCKOUT_ATTEMPTS - 1).times { maintainer.register_failure! }
    expect(maintainer.reload.locked?).to be(false)

    maintainer.register_failure!
    expect(maintainer.reload.locked?).to be(true)
    expect(maintainer.locked_until).to be_within(5.seconds).of(described_class::LOCKOUT_WINDOW.from_now)

    travel_to(described_class::LOCKOUT_WINDOW.from_now + 1.second) do
      expect(maintainer.reload.locked?).to be(false)
    end
  end

  it "clears the failure count on a good login" do
    maintainer = build_maintainer
    2.times { maintainer.register_failure! }

    maintainer.clear_failures!

    expect(maintainer.reload.failed_attempts).to eq(0)
    expect(maintainer.locked_until).to be_nil
  end

  it "deactivates, killing every session at once" do
    maintainer = build_maintainer
    maintainer.maintainer_sessions.create!(mfa_verified_at: Time.current, last_seen_at: Time.current)

    maintainer.deactivate!

    expect(maintainer.reload.active?).to be(false)
    expect(MaintainerSession.where(maintainer_id: maintainer.id)).to be_empty
  end

  it "knows it is the last active maintainer" do
    first = build_maintainer
    expect(first.last_active?).to be(true)

    second = build_maintainer
    expect(first.reload.last_active?).to be(false)

    second.deactivate!
    expect(first.reload.last_active?).to be(true)
  end

  # Bloco aninhado: aqui `described_class` seria MaintainerInvitation, então o
  # mantenedor é criado pelo nome da classe, não por described_class.
  describe MaintainerInvitation do
    it "stores only a digest and is usable once, inside the window" do
      maintainer = Maintainer.create!(email_address: "inv-#{SecureRandom.hex(3)}@rotasaude.app")
      invitation, token = MaintainerInvitation.issue!(maintainer: maintainer)

      expect(token).to be_present
      expect(invitation.token_digest).not_to include(token)
      expect(MaintainerInvitation.find_by(token_digest: MaintainerInvitation.digest_for(token))).to eq(invitation)
      expect(invitation.usable?).to be(true)

      invitation.update!(used_at: Time.current)
      expect(invitation.reload.usable?).to be(false)

      other, _token = MaintainerInvitation.issue!(maintainer: maintainer)
      travel_to(MaintainerInvitation::TTL.from_now + 1.second) { expect(other.reload.usable?).to be(false) }
    end
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/maintainer_spec.rb`
Expected: FAIL com `NameError: uninitialized constant Maintainer`.

- [ ] **Step 3: Escrever a migration**

`db/platform_migrate/20260917000001_create_maintainers.rb`:

```ruby
# Identidade da API de manutenção, no banco de PLATAFORMA (spec §6). Espelha
# operators/operator_sessions: mesma cifra de otp_secret, mesmo índice único por
# lower(email). password_digest é NULO até o convite ser aceito — o mantenedor
# nasce do convite, não de um formulário de senha.
class CreateMaintainers < ActiveRecord::Migration[8.1]
  def change
    create_table :maintainers, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.string   :email_address, null: false
      t.string   :password_digest
      t.string   :otp_secret
      t.jsonb    :otp_recovery_codes, null: false, default: []
      t.datetime :otp_enabled_at
      t.datetime :deactivated_at
      t.integer  :failed_attempts, null: false, default: 0
      t.datetime :locked_until
      t.uuid     :invited_by_id
      t.timestamps
      t.index "lower(email_address)", unique: true, name: "index_maintainers_on_lower_email"
    end

    create_table :maintainer_sessions, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.uuid     :maintainer_id, null: false
      t.datetime :mfa_verified_at
      t.datetime :last_seen_at
      t.integer  :totp_attempts, null: false, default: 0
      t.string   :ip_address
      t.string   :user_agent
      t.timestamps
      t.index :maintainer_id
    end

    create_table :maintainer_invitations, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.uuid     :maintainer_id, null: false
      t.string   :token_digest, null: false
      t.datetime :expires_at, null: false
      t.datetime :used_at
      t.timestamps
      t.index :token_digest, unique: true
      t.index :maintainer_id
    end
  end
end
```

- [ ] **Step 4: Migrar development e test**

```bash
docker compose exec -T api bin/rails db:migrate:platform
docker compose exec -T -e RAILS_ENV=test -e POSTGRES_PASSWORD=postgres api bin/rails db:migrate:platform
```
Expected: as três tabelas criadas e `db/platform_schema.rb` atualizado. Confira com `git diff --stat db/platform_schema.rb`.

- [ ] **Step 5: Implementar os modelos**

`app/models/maintainer.rb`:

```ruby
# Conta da API de manutenção (spec §6). Papel único e TOTAL: não há papel a
# guardar, então o que protege é a autenticação — TOTP obrigatório, bloqueio por
# conta e desativação que mata sessão na hora.
#
# Espelha Operator sem herdar: são planos diferentes (o console opera produção;
# esta ferramenta não existe lá), e juntar os dois numa tabela só faria uma conta
# de console valer aqui.
class Maintainer < PlatformRecord
  # validations: false — o mantenedor nasce pelo convite, ainda sem senha; a
  # senha chega no aceite (Task 5), e `enrolled?` é quem decide se ele loga.
  has_secure_password validations: false

  has_many :maintainer_sessions, dependent: :destroy
  has_many :maintainer_invitations, dependent: :destroy

  # Mesma custódia de Operator#otp_secret: chave da PLATAFORMA, fixa, porque
  # este atributo pode ser lido de dentro de CityConnection.with.
  encrypts :otp_secret, key_provider: PlatformKeyProvider.new

  normalizes :email_address, with: ->(e) { e.strip.downcase }

  validates :email_address, presence: true, uniqueness: { case_sensitive: false }

  LOCKOUT_ATTEMPTS = 5
  LOCKOUT_WINDOW = 15.minutes

  scope :active, -> { where(deactivated_at: nil) }

  def active? = deactivated_at.nil?

  # Só loga quem tem senha E TOTP confirmado: um convite aceito pela metade não
  # abre sessão.
  def enrolled? = password_digest.present? && otp_secret.present? && otp_enabled_at.present?

  def locked? = locked_until.present? && locked_until > Time.current

  # Incremento atômico: duas tentativas simultâneas contam duas. O bloqueio é da
  # CONTA (o rate_limit do controller é por IP, e trocar de IP é barato).
  def register_failure!
    self.class.where(id: id).update_all(<<~SQL.squish)
      failed_attempts = failed_attempts + 1,
      locked_until = CASE WHEN failed_attempts + 1 >= #{LOCKOUT_ATTEMPTS}
                          THEN now() + interval '#{LOCKOUT_WINDOW.to_i} seconds' ELSE locked_until END,
      updated_at = now()
    SQL
  end

  def clear_failures!
    update!(failed_attempts: 0, locked_until: nil)
  end

  def deactivate!
    transaction do
      update!(deactivated_at: Time.current)
      maintainer_sessions.destroy_all
    end
  end

  # Guarda do "ninguém tranca todo mundo para fora": o último mantenedor ativo
  # não pode ser desativado (spec §6).
  def last_active? = active? && self.class.active.where.not(id: id).none?
end
```

`app/models/maintainer_session.rb`:

```ruby
class MaintainerSession < PlatformRecord
  belongs_to :maintainer
end
```

`app/models/maintainer_invitation.rb`:

```ruby
# Convite de mantenedor (spec §6): uso único, 24h. Grava só o DIGEST — o token em
# claro existe uma vez, na saída da rake, e nunca no banco nem em log.
class MaintainerInvitation < PlatformRecord
  belongs_to :maintainer

  TTL = 24.hours

  def self.digest_for(token) = OpenSSL::Digest::SHA256.hexdigest(token.to_s)

  def self.issue!(maintainer:)
    token = SecureRandom.urlsafe_base64(32)
    invitation = create!(maintainer: maintainer, token_digest: digest_for(token), expires_at: TTL.from_now)
    [ invitation, token ]
  end

  def usable? = used_at.nil? && expires_at > Time.current
end
```

- [ ] **Step 6: Rodar e confirmar que passa**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/maintainer_spec.rb`
Expected: PASS, 7 exemplos.

- [ ] **Step 7: Commit**

```bash
cd apps/api
/opt/homebrew/bin/git add db/platform_migrate/20260917000001_create_maintainers.rb db/platform_schema.rb \
  app/models/maintainer.rb app/models/maintainer_session.rb app/models/maintainer_invitation.rb \
  spec/models/maintainer_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: add the maintainer identity on the platform database

Maintainers mirror operators without inheriting from them: encrypted TOTP
secret, account lockout after five failures, deactivation that kills
every session, and single-use invitations stored as digests.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 3: Auditoria imutável

**Files:**
- Create: `app/events/maintenance_audit.rb`, `db/platform_migrate/20260917000002_maintenance_events_immutable.rb`
- Modify: `spec/events/platform_event_payload_guard_spec.rb` (lista R18), `db/platform_schema.rb`
- Test: `spec/events/maintenance_audit_spec.rb`

**Interfaces:**
- Consumes: `Platform.audit`, `PlatformEvent`.
- Produces:
  - `MaintenanceAudit::NAMES` → lista congelada dos nomes desta fatia
  - `MaintenanceAudit.record(name, outcome:, maintainer_id:, credential:, module_name:, correlation_id: SecureRandom.uuid, **fields) → String` (o `correlation_id`), levanta `ArgumentError` para nome fora de `NAMES` ou `outcome` fora de `%w[attempted ok rejected error]`
  - `MaintenanceAudit.credential_for(session: nil, token: nil) → Hash` — `{ "kind" => "session" }` ou `{ "kind" => "token", "token_id" => "…" }`
  - Trigger `platform_events_maintenance_immutable` no Postgres

- [ ] **Step 1: Escrever o spec que falha**

`spec/events/maintenance_audit_spec.rb`:

```ruby
require "rails_helper"

# Spec da API de manutenção §9: com poderes totais, o que resta é registrar quem
# fez o quê — e ninguém, nem o superusuário, reescreve esse registro.
RSpec.describe MaintenanceAudit do
  let(:maintainer) { Maintainer.create!(email_address: "aud-#{SecureRandom.hex(3)}@rotasaude.app") }

  it "records an event with actor, module, outcome and correlation id" do
    correlation_id = described_class.record(
      "maintenance.session.started", outcome: "ok", maintainer_id: maintainer.id,
      credential: described_class.credential_for(session: "s-1"), module_name: "session"
    )

    event = PlatformEvent.order(:created_at).last
    expect(event.name).to eq("maintenance.session.started")
    expect(event.payload).to include("outcome" => "ok", "module" => "session",
                                     "maintainer_id" => maintainer.id, "correlation_id" => correlation_id)
    expect(event.payload.fetch("credential")).to eq({ "kind" => "session" })
  end

  it "never carries the maintainer's e-mail, only the id (Ruling R18)" do
    expect do
      described_class.record("maintenance.session.started", outcome: "ok", maintainer_id: maintainer.id,
                             credential: { "kind" => "session" }, module_name: "session",
                             login: maintainer.email_address)
    end.to raise_error(ActiveRecord::RecordInvalid, /payload/i)
  end

  it "refuses an undeclared name and an unknown outcome" do
    expect do
      described_class.record("maintenance.whatever.done", outcome: "ok", maintainer_id: maintainer.id,
                             credential: { "kind" => "session" }, module_name: "session")
    end.to raise_error(ArgumentError, /maintenance\.whatever\.done/)

    expect do
      described_class.record("maintenance.session.started", outcome: "maybe", maintainer_id: maintainer.id,
                             credential: { "kind" => "session" }, module_name: "session")
    end.to raise_error(ArgumentError, /maybe/)
  end

  it "keeps the attempt and its outcome on the same correlation id" do
    correlation_id = described_class.record("maintenance.maintainer.invited", outcome: "attempted",
                                            maintainer_id: maintainer.id, credential: { "kind" => "session" },
                                            module_name: "maintainer")
    described_class.record("maintenance.maintainer.invited", outcome: "ok", maintainer_id: maintainer.id,
                           credential: { "kind" => "session" }, module_name: "maintainer",
                           correlation_id: correlation_id)

    outcomes = PlatformEvent.where(name: "maintenance.maintainer.invited")
                            .select { |e| e.payload["correlation_id"] == correlation_id }
                            .map { |e| e.payload["outcome"] }

    expect(outcomes).to contain_exactly("attempted", "ok")
  end

  describe "immutability in the database" do
    let!(:event) do
      described_class.record("maintenance.session.started", outcome: "ok", maintainer_id: maintainer.id,
                             credential: { "kind" => "session" }, module_name: "session")
      PlatformEvent.order(:created_at).last
    end

    it "refuses UPDATE of the payload and DELETE, and still allows published_at" do
      expect { event.update!(payload: event.payload.merge("outcome" => "rejected")) }
        .to raise_error(ActiveRecord::StatementInvalid, /immutable/i)
      expect { event.destroy! }.to raise_error(ActiveRecord::StatementInvalid, /immutable/i)

      expect { event.update!(published_at: Time.current) }.not_to raise_error
    end

    it "leaves the other platform events alone" do
      Platform.audit("operator.login", operator_id: SecureRandom.uuid)
      other = PlatformEvent.order(:created_at).last

      expect { other.destroy! }.not_to raise_error
    end

    # O trigger não aparece em db/platform_schema.rb (o dump em Ruby não
    # representa trigger): um banco reconstruído por schema:load ficaria sem ele.
    it "is installed as a trigger, not only as application code" do
      installed = PlatformRecord.connection.select_value(<<~SQL.squish)
        SELECT count(*) FROM pg_trigger
        WHERE NOT tgisinternal AND tgname = 'platform_events_maintenance_immutable'
      SQL

      expect(installed.to_i).to eq(1)
    end
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/events/maintenance_audit_spec.rb`
Expected: FAIL com `NameError: uninitialized constant MaintenanceAudit`.

- [ ] **Step 3: Implementar `app/events/maintenance_audit.rb`**

```ruby
# Auditoria da API de manutenção (spec §9), gravada em platform_events pelo mesmo
# canal do resto da plataforma (Platform.audit).
#
# Duas decisões que o arquivo existe para sustentar:
#
#   1. O ATOR é o id, nunca o e-mail. platform_events é governado pela Ruling
#      R18, que recusa chave com "email" — e a intenção da regra é não acumular
#      dado pessoal aqui. Quem lê resolve o login juntando com maintainers.
#   2. Nome fora da lista e outcome desconhecido levantam. Um typo em nome de
#      evento produziria um registro que nenhuma consulta encontra — auditoria
#      que ninguém acha é auditoria que não existe.
module MaintenanceAudit
  # Nomes desta fatia. Nome novo entra AQUI e em R18_PLATFORM_EVENT_NAMES
  # (spec/events/platform_event_payload_guard_spec.rb) — a suíte quebra até lá.
  NAMES = %w[
    maintenance.session.started
    maintenance.session.failed
    maintenance.session.locked
    maintenance.session.ended
    maintenance.maintainer.invited
    maintenance.maintainer.accepted
  ].freeze

  OUTCOMES = %w[attempted ok rejected error].freeze

  module_function

  def record(name, outcome:, maintainer_id:, credential:, module_name:, correlation_id: SecureRandom.uuid, **fields)
    raise ArgumentError, "evento de manutenção não declarado: #{name}" unless NAMES.include?(name.to_s)
    raise ArgumentError, "outcome desconhecido: #{outcome}" unless OUTCOMES.include?(outcome.to_s)

    Platform.audit(name,
                   outcome: outcome.to_s,
                   module: module_name.to_s,
                   maintainer_id: maintainer_id,
                   credential: credential,
                   correlation_id: correlation_id,
                   **fields)

    correlation_id
  end

  # Quem agiu: sessão humana ou token de serviço (o token chega no Plano 3). O id
  # da sessão não entra: ele é o valor do cookie.
  def credential_for(session: nil, token: nil)
    return { "kind" => "token", "token_id" => token.id } if token

    raise ArgumentError, "credential_for exige session: ou token:" unless session

    { "kind" => "session" }
  end
end
```

- [ ] **Step 4: Declarar os nomes na guarda R18**

Em `spec/events/platform_event_payload_guard_spec.rb`, acrescente à constante `R18_PLATFORM_EVENT_NAMES`, depois de `operator.impersonated`:

```ruby
                                maintenance.session.started maintenance.session.failed
                                maintenance.session.locked maintenance.session.ended
                                maintenance.maintainer.invited maintenance.maintainer.accepted
```

- [ ] **Step 5: Escrever a migration do trigger**

`db/platform_migrate/20260917000002_maintenance_events_immutable.rb`:

```ruby
# Imutabilidade da auditoria de manutenção (spec §9). O mantenedor pode tudo —
# menos reescrever o registro do que fez. Nenhuma mutation toca platform_events,
# e este trigger fecha o caminho que sobra: um bug no app, ou um psql aberto com
# o papel da aplicação.
#
# Só published_at pode mudar (é o outbox, ADR-0004). O filtro é por nome
# maintenance.%: o resto de platform_events segue com a purga por retenção.
#
# ATENÇÃO: trigger não entra em db/platform_schema.rb (o dump em Ruby não o
# representa). Ele existe onde as MIGRATIONS rodaram; spec/events/
# maintenance_audit_spec.rb confere a presença no pg_trigger para que um banco
# reconstruído por schema:load falhe alto.
class MaintenanceEventsImmutable < ActiveRecord::Migration[8.1]
  def up
    execute <<~SQL
      CREATE OR REPLACE FUNCTION platform_events_maintenance_immutable() RETURNS trigger AS $fn$
      BEGIN
        IF TG_OP = 'DELETE' THEN
          RAISE EXCEPTION 'maintenance audit events are immutable: DELETE refused (%)', OLD.name;
        END IF;

        IF NEW.name IS DISTINCT FROM OLD.name
           OR NEW.payload IS DISTINCT FROM OLD.payload
           OR NEW.occurred_at IS DISTINCT FROM OLD.occurred_at
           OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
          RAISE EXCEPTION 'maintenance audit events are immutable: only published_at may change (%)', OLD.name;
        END IF;

        RETURN NEW;
      END;
      $fn$ LANGUAGE plpgsql;

      CREATE TRIGGER platform_events_maintenance_immutable
        BEFORE UPDATE OR DELETE ON platform_events
        FOR EACH ROW
        WHEN (OLD.name LIKE 'maintenance.%')
        EXECUTE FUNCTION platform_events_maintenance_immutable();
    SQL
  end

  def down
    execute <<~SQL
      DROP TRIGGER IF EXISTS platform_events_maintenance_immutable ON platform_events;
      DROP FUNCTION IF EXISTS platform_events_maintenance_immutable();
    SQL
  end
end
```

- [ ] **Step 6: Migrar e rodar**

```bash
docker compose exec -T api bin/rails db:migrate:platform
docker compose exec -T -e RAILS_ENV=test -e POSTGRES_PASSWORD=postgres api bin/rails db:migrate:platform
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/events/maintenance_audit_spec.rb spec/events/platform_event_payload_guard_spec.rb
```
Expected: PASS nos dois arquivos.

Se o exemplo de imutabilidade falhar por a transação do spec abortar depois da exceção do trigger, envolva cada expectativa em `PlatformRecord.transaction(requires_new: true) { ... }` no spec — o savepoint isola o erro. Ajuste só o spec.

- [ ] **Step 7: Commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/events/maintenance_audit.rb db/platform_migrate/20260917000002_maintenance_events_immutable.rb \
  db/platform_schema.rb spec/events/maintenance_audit_spec.rb spec/events/platform_event_payload_guard_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: add the maintenance audit channel, immutable in the database

Every maintenance event carries actor id, module, outcome and a
correlation id that ties an attempt to its result. The actor is an id,
never an e-mail (Ruling R18), undeclared names raise, and a Postgres
trigger refuses UPDATE and DELETE on maintenance.% events.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 4: Sessão humana (senha → TOTP), cookie e CSRF

**Files:**
- Create: `app/controllers/concerns/maintainer_authentication.rb`, `app/controllers/maintenance/base_controller.rb`, `app/controllers/maintenance/sessions_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/cors.rb`
- Test: `spec/requests/maintenance/sessions_spec.rb`

**Interfaces:**
- Consumes: `MaintenanceApi.enabled?`, `MaintenanceApiHost`, `Maintainer`, `MaintainerSession`, `Mfa::Verify`, `MaintenanceAudit`, `Rota.deployed?`.
- Produces:
  - `MaintainerAuthentication`: `COOKIE = :maintainer_session_id`, `PENDING_MFA_WINDOW = 10.minutes`, `ABSOLUTE_TTL = 8.hours`, `IDLE_TTL = 30.minutes`, `MAX_TOTP_ATTEMPTS = 5`, `HEADER = "X-Rota-Maintenance"`; `#current_maintainer`, `#require_maintainer_authentication`, `#start_pending_session_for`, `#write_maintenance_cookie`, `#terminate_maintenance_session`; classe: `allow_unauthenticated_maintainer_access(**options)`
  - `Maintenance::BaseController` (host + CSRF + autenticação)
  - Rotas, só quando `MaintenanceApi.enabled?`: `POST /session`, `POST /session/challenge`, `GET /session`, `DELETE /session` sob `MaintenanceApiHost`
  - `ENV["MAINTENANCE_FRONTEND_ORIGIN"]` é a origem exigida no `Origin`

- [ ] **Step 1: Escrever o spec que falha**

`spec/requests/maintenance/sessions_spec.rb`:

```ruby
require "rails_helper"

# Spec da API de manutenção §6: senha abre sessão PENDENTE, TOTP verifica, e o
# cookie só autentica depois disso. Poderes totais ⇒ sessão curta, bloqueio por
# conta e CSRF por Origin + header próprio.
RSpec.describe "Maintainer session", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  let(:password) { "s3nha-forte-1" }
  let(:frontend) { "https://maintenance.rotasaude.app" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "m-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end

  def api_host = "maintenance-api.rotasaude.app"
  def json = JSON.parse(response.body)
  def totp = ROTP::TOTP.new(maintainer.otp_secret).now
  def headers = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }

  def login!(pwd = password)
    post "/session", params: { email_address: maintainer.email_address, password: pwd }, headers: headers
  end

  def verified_login!
    login!
    post "/session/challenge", params: { session_id: json["session_id"], code: totp }, headers: headers
    expect(response).to have_http_status(:ok)
  end

  before do
    host! api_host
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with("MAINTENANCE_FRONTEND_ORIGIN").and_return(frontend)
  end

  it "opens a pending session on the password step, which does not authenticate yet" do
    login!

    expect(response).to have_http_status(:ok)
    expect(json).to include("mfa_required" => true)
    expect(MaintainerSession.find(json["session_id"]).mfa_verified_at).to be_nil

    get "/session", headers: headers
    expect(response).to have_http_status(:unauthorized)
  end

  it "authenticates after TOTP and audits the login with the maintainer id, never the e-mail" do
    verified_login!

    get "/session", headers: headers
    expect(response).to have_http_status(:ok)
    expect(json).to include("email_address" => maintainer.email_address)

    event = PlatformEvent.where(name: "maintenance.session.started").last
    expect(event.payload).to include("maintainer_id" => maintainer.id, "outcome" => "ok")
    expect(event.payload.to_json).not_to include(maintainer.email_address)
  end

  it "writes a host-only, HttpOnly, SameSite=Strict cookie" do
    login!
    set_cookie = Array(response.headers["Set-Cookie"]).join("\n")

    expect(set_cookie).to include("maintainer_session_id")
    expect(set_cookie).to include("HttpOnly")
    expect(set_cookie).to match(/SameSite=Strict/i)
    expect(set_cookie).not_to include("domain")
  end

  it "expires the session eight hours after the TOTP, and after thirty idle minutes" do
    verified_login!

    travel_to(7.hours.from_now) do
      get "/session", headers: headers
      expect(response).to have_http_status(:ok)
    end

    travel_to(8.hours.from_now + 1.minute) do
      get "/session", headers: headers
      expect(response).to have_http_status(:unauthorized)
    end

    verified_login!
    travel_to(31.minutes.from_now) do
      get "/session", headers: headers
      expect(response).to have_http_status(:unauthorized)
    end
  end

  it "locks the account after five bad passwords, then refuses the right one" do
    Maintainer::LOCKOUT_ATTEMPTS.times { login!("errada") }

    expect(maintainer.reload.locked?).to be(true)
    expect(PlatformEvent.where(name: "maintenance.session.locked").count).to eq(1)

    login!
    expect(response).to have_http_status(:unauthorized)
    expect(json).to include("error" => "locked")
  end

  it "refuses a request without the exact Origin or without the header" do
    verified_login!

    get "/session", headers: { "Origin" => "https://attacker.example", "X-Rota-Maintenance" => "1" }
    expect(response).to have_http_status(:forbidden)

    get "/session", headers: { "Origin" => frontend }
    expect(response).to have_http_status(:forbidden)
  end

  it "answers 404 on any other host, never 401" do
    host! "curitiba.rotasaude.app"
    get "/session", headers: headers
    expect(response).to have_http_status(:not_found)

    host! "admin.rotasaude.app"
    post "/session", params: { email_address: maintainer.email_address, password: password }, headers: headers
    expect(response).not_to have_http_status(:ok)
  end

  it "refuses a deactivated maintainer and ends the session on logout" do
    verified_login!
    delete "/session", headers: headers
    expect(response).to have_http_status(:no_content)
    expect(PlatformEvent.where(name: "maintenance.session.ended").count).to eq(1)

    maintainer.deactivate!
    login!
    expect(response).to have_http_status(:unauthorized)
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/maintenance/sessions_spec.rb`
Expected: FAIL — sem rota, os posts respondem 404.

- [ ] **Step 3: Implementar o concern**

`app/controllers/concerns/maintainer_authentication.rb`:

```ruby
# Autenticação do mantenedor na API de manutenção (spec §6), contra
# Maintainer/MaintainerSession no banco de plataforma.
#
# Diferenças deliberadas em relação a OperatorAuthentication:
#   - sessão mais CURTA: 8h absolutas desde o TOTP e 30 min de inatividade. O
#     operador tem 12h; aqui os poderes são totais;
#   - SameSite=Strict (o operador usa :lax): nenhuma navegação de terceiro
#     carrega esta sessão;
#   - CSRF explícito: Origin exatamente igual ao frontend do ambiente MAIS o
#     header X-Rota-Maintenance, que força preflight. A API é servida em outra
#     origem que o frontend, então SameSite sozinho não basta — os dois estão no
#     mesmo site (rotasaude.com.br), e os três ambientes também;
#   - host-only, como todo cookie de sessão: NUNCA `domain:`.
module MaintainerAuthentication
  extend ActiveSupport::Concern

  COOKIE = :maintainer_session_id
  HEADER = "X-Rota-Maintenance"
  PENDING_MFA_WINDOW = 10.minutes
  ABSOLUTE_TTL = 8.hours
  IDLE_TTL = 30.minutes
  MAX_TOTP_ATTEMPTS = 5

  included do
    before_action :require_maintenance_origin
    before_action :require_maintainer_authentication
  end

  class_methods do
    def allow_unauthenticated_maintainer_access(**options)
      skip_before_action :require_maintainer_authentication, **options
    end
  end

  private

  def current_maintainer = Current.maintainer_session&.maintainer

  # Origem exata + header próprio. Sem os dois, um formulário de outro site
  # dispararia mutation com o cookie do mantenedor.
  def require_maintenance_origin
    return head(:forbidden) unless request.headers["Origin"] == ENV["MAINTENANCE_FRONTEND_ORIGIN"]

    head(:forbidden) unless request.headers[HEADER] == "1"
  end

  def require_maintainer_authentication
    resume_maintainer_session || render(json: { error: "unauthenticated" }, status: :unauthorized)
  end

  def resume_maintainer_session
    Current.maintainer_session ||= find_verified_session&.tap { |session| touch_session(session) }
  end

  def find_verified_session
    id = cookies.signed[COOKIE]
    return nil unless id

    session = MaintainerSession.find_by(id: id)
    return nil unless session&.mfa_verified_at && session.maintainer.active?
    return nil if session.mfa_verified_at <= ABSOLUTE_TTL.ago
    return nil if (session.last_seen_at || session.mfa_verified_at) <= IDLE_TTL.ago

    session
  end

  # A inatividade é medida no SERVIDOR: o cookie é de sessão do navegador, que
  # some ao fechar, mas quem decide é a linha no banco.
  def touch_session(session)
    session.update_columns(last_seen_at: Time.current)
  end

  def start_pending_session_for(maintainer)
    maintainer.maintainer_sessions.create!(user_agent: request.user_agent, ip_address: request.remote_ip)
              .tap { |session| write_maintenance_cookie(session) }
  end

  def write_maintenance_cookie(session)
    cookies.signed[COOKIE] = {
      value: session.id,
      httponly: true,
      same_site: :strict,
      secure: Rota.deployed?
    }
  end

  def terminate_maintenance_session
    MaintainerSession.find_by(id: cookies.signed[COOKIE])&.destroy
    Current.maintainer_session = nil
    cookies.delete(COOKIE)
  end
end
```

Acrescente o atributo em `app/models/current.rb`, junto dos que já existem:

```ruby
  attribute :maintainer_session
```

- [ ] **Step 4: Implementar os controllers**

`app/controllers/maintenance/base_controller.rb`:

```ruby
# Base dos controllers da API de manutenção (maintenance-api.*).
#
# NÃO herda de ApplicationController: não há cidade no host, e CityResolution
# devolveria 404. Espelha Operators::BaseController, inclusive na ordem: o host
# é checado ANTES da autenticação, para host errado receber 404 e nunca 401.
module Maintenance
  class BaseController < ActionController::API
    include ActionController::Cookies

    before_action :require_maintenance_host

    include MaintainerAuthentication

    private

    def require_maintenance_host
      head :not_found unless CityCatalog.maintenance_api_host?(request.host)
    end
  end
end
```

`app/controllers/maintenance/sessions_controller.rb`:

```ruby
# Sessão do mantenedor (spec §6) — JSON-only, no host maintenance-api.*:
#
#   POST   /session            { email_address, password } → 200 { mfa_required, session_id }
#   POST   /session/challenge  { session_id, code }        → 200 mantenedor
#   GET    /session                                        → 200 mantenedor | 401
#   DELETE /session                                        → 204
module Maintenance
  class SessionsController < BaseController
    allow_unauthenticated_maintainer_access only: %i[create challenge_totp destroy]

    rate_limit to: 10, within: 3.minutes, only: %i[create challenge_totp],
               with: -> { render json: { error: "too_many_requests" }, status: :too_many_requests }

    def create
      maintainer = Maintainer.active.find_by(email_address: params[:email_address].to_s.strip.downcase)
      return render(json: { error: "invalid_credentials" }, status: :unauthorized) unless maintainer
      return render(json: { error: "locked" }, status: :unauthorized) if maintainer.locked?

      unless maintainer.enrolled? && maintainer.authenticate(params[:password].to_s)
        return register_failed_password(maintainer)
      end

      maintainer.clear_failures!
      session = start_pending_session_for(maintainer)
      render json: { mfa_required: true, session_id: session.id }, status: :ok
    end

    def challenge_totp
      session = pending_session
      return render(json: { error: "invalid_session" }, status: :unauthorized) unless session
      return register_failed_totp(session) unless Mfa::Verify.call(session.maintainer, code: params[:code])

      # Atômico, pelo mesmo motivo de Operators::SessionsController: o cookie já
      # foi plantado no passo da senha, então um carimbo sem evento de auditoria
      # autenticaria sem registro. O update_all condicional é a guarda contra a
      # corrida com register_failed_totp.
      now = Time.current
      verified = PlatformRecord.transaction do
        stamped = MaintainerSession.where(id: session.id, mfa_verified_at: nil)
                                   .update_all(mfa_verified_at: now, last_seen_at: now, updated_at: now)
        next false unless stamped == 1

        MaintenanceAudit.record("maintenance.session.started", outcome: "ok", module_name: "session",
                                maintainer_id: session.maintainer_id,
                                credential: MaintenanceAudit.credential_for(session: session))
        true
      end
      return render(json: { error: "invalid_session" }, status: :unauthorized) unless verified

      session.assign_attributes(mfa_verified_at: now, last_seen_at: now)
      write_maintenance_cookie(session)
      Current.maintainer_session = session
      render json: serialize(session), status: :ok
    end

    def show
      render json: serialize(Current.maintainer_session)
    end

    def destroy
      session = Current.maintainer_session || MaintainerSession.find_by(id: cookies.signed[MaintainerAuthentication::COOKIE])
      if session
        MaintenanceAudit.record("maintenance.session.ended", outcome: "ok", module_name: "session",
                                maintainer_id: session.maintainer_id,
                                credential: MaintenanceAudit.credential_for(session: session))
      end
      terminate_maintenance_session
      head :no_content
    end

    private

    # O bloqueio é da conta, não do IP: o rate_limit acima atrapalha quem insiste
    # do mesmo lugar, e trocar de IP é barato demais para ser a única barreira.
    def register_failed_password(maintainer)
      maintainer.register_failure!
      locked = maintainer.reload.locked?

      MaintenanceAudit.record(locked ? "maintenance.session.locked" : "maintenance.session.failed",
                              outcome: "rejected", module_name: "session", maintainer_id: maintainer.id,
                              credential: { "kind" => "password" })

      render json: { error: locked ? "locked" : "invalid_credentials" }, status: :unauthorized
    end

    # A sessão do challenge tem de ser a MESMA cujo cookie este cliente recebeu,
    # ainda pendente, dentro da janela e de mantenedor ativo.
    def pending_session
      id = params[:session_id].to_s
      return nil if id.empty? || cookies.signed[MaintainerAuthentication::COOKIE] != id

      session = MaintainerSession.find_by(id: id, mfa_verified_at: nil)
      return nil unless session
      return nil if session.created_at <= MaintainerAuthentication::PENDING_MFA_WINDOW.ago
      return nil unless session.maintainer.active?

      session
    end

    def register_failed_totp(session)
      counted = MaintainerSession.where(id: session.id, mfa_verified_at: nil)
                                 .update_all("totp_attempts = totp_attempts + 1")
      return render(json: { error: "invalid_session" }, status: :unauthorized) unless counted == 1

      attempts = MaintainerSession.where(id: session.id).pick(:totp_attempts)
      return render(json: { error: "invalid_code" }, status: :unauthorized) if
        attempts && attempts < MaintainerAuthentication::MAX_TOTP_ATTEMPTS

      MaintainerSession.where(id: session.id, mfa_verified_at: nil).delete_all
      cookies.delete(MaintainerAuthentication::COOKIE)
      MaintenanceAudit.record("maintenance.session.failed", outcome: "rejected", module_name: "session",
                              maintainer_id: session.maintainer_id, credential: { "kind" => "totp" })
      render json: { error: "too_many_attempts" }, status: :unauthorized
    end

    def serialize(session)
      maintainer = session.maintainer
      {
        id: maintainer.id,
        email_address: maintainer.email_address,
        mfa_verified_at: session.mfa_verified_at&.iso8601,
        expires_at: (session.mfa_verified_at + MaintainerAuthentication::ABSOLUTE_TTL).iso8601
      }
    end
  end
end
```

- [ ] **Step 5: Desenhar as rotas e liberar o CORS**

Em `config/routes.rb`, no fim do `Rails.application.routes.draw`, depois do bloco `if Rails.env.development?`:

```ruby
  # API de manutenção (spec da API de manutenção §3/§5). A rota SÓ existe em
  # development e staging com MAINTENANCE_API_ENABLED=true — e em test, onde os
  # request specs vivem. Em produção não é desenhada, e a chave ligada derruba o
  # boot (config/initializers/01_maintenance_api.rb).
  if MaintenanceApi.enabled?
    constraints(MaintenanceApiHost) do
      scope module: :maintenance, as: :maintenance do
        resource :session, only: %i[create show destroy]
        post "/session/challenge", to: "sessions#challenge_totp"
      end
    end
  end
```

Em `config/initializers/cors.rb`, depois do bloco `allow` existente, acrescente:

```ruby
  # Frontend da API de manutenção: origem ÚNICA e exata, com credenciais (o
  # cookie de sessão). Bloco próprio, e não uma entrada em ALLOWED_ORIGINS,
  # porque aquela lista também libera /session e /admin/api do console.
  allow do
    origins ENV.fetch("MAINTENANCE_FRONTEND_ORIGIN", "")
    resource "/session",
             headers: :any,
             methods: %i[get post delete options],
             credentials: true
    resource "/session/challenge",
             headers: :any,
             methods: %i[post options],
             credentials: true
  end
```

- [ ] **Step 6: Rodar e confirmar que passa**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/maintenance/sessions_spec.rb spec/requests/cors_spec.rb spec/architecture/cookie_domain_spec.rb`
Expected: PASS, 0 falhas.

- [ ] **Step 7: Suíte completa e commit**

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
```

```bash
cd apps/api
/opt/homebrew/bin/git add app/controllers/concerns/maintainer_authentication.rb app/controllers/maintenance \
  app/models/current.rb config/routes.rb config/initializers/cors.rb spec/requests/maintenance/sessions_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: authenticate maintainers with password, TOTP and a strict cookie

Password opens a pending session, TOTP verifies it, and the cookie is
host-only, HttpOnly and SameSite=Strict. The session dies eight hours
after the TOTP or after thirty idle minutes, five bad passwords lock the
account, and every request needs the frontend's exact Origin plus the
X-Rota-Maintenance header.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 5: Convite e aceite

**Files:**
- Create: `app/controllers/maintenance/invitations_controller.rb`, `lib/tasks/maintainer.rake`
- Modify: `config/routes.rb`
- Test: `spec/requests/maintenance/invitations_spec.rb`, `spec/tasks/maintainer_rake_spec.rb`

**Interfaces:**
- Consumes: `Maintainer`, `MaintainerInvitation`, `Mfa::Enroll`, `Mfa::Verify`, `MaintenanceAudit`.
- Produces:
  - `GET /invitations/:token` → `{ email_address, otpauth_uri, secret, recovery_codes }` (primeiro passo do aceite: gera o TOTP)
  - `POST /invitations/:token/accept` `{ password, code }` → 204
  - `rake maintainer:invite[email]` → cria ou reconvida, imprime a URL do convite e o e-mail mascarado

- [ ] **Step 1: Escrever os specs que falham**

`spec/requests/maintenance/invitations_spec.rb`:

```ruby
require "rails_helper"

# Spec da API de manutenção §6: o mantenedor nasce de um convite de uso único,
# válido por 24h, e só passa a logar com senha DEFINIDA e TOTP CONFIRMADO.
RSpec.describe "Maintainer invitation", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:maintainer) { Maintainer.create!(email_address: "conv-#{SecureRandom.hex(3)}@rotasaude.app") }
  let(:issued) { MaintainerInvitation.issue!(maintainer: maintainer) }
  let(:invitation) { issued.first }
  let(:token) { issued.last }

  def headers = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
  def json = JSON.parse(response.body)

  before do
    host! "maintenance-api.rotasaude.app"
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with("MAINTENANCE_FRONTEND_ORIGIN").and_return(frontend)
  end

  it "hands the enrollment material once and accepts password plus a confirmed TOTP" do
    get "/invitations/#{token}", headers: headers
    expect(response).to have_http_status(:ok)
    expect(json).to include("email_address" => maintainer.email_address)
    expect(json["otpauth_uri"]).to include("otpauth://")

    code = ROTP::TOTP.new(maintainer.reload.otp_secret).now
    post "/invitations/#{token}/accept", params: { password: "s3nha-forte-1", code: code }, headers: headers

    expect(response).to have_http_status(:no_content)
    expect(maintainer.reload.enrolled?).to be(true)
    expect(invitation.reload.usable?).to be(false)
    expect(PlatformEvent.where(name: "maintenance.maintainer.accepted").count).to eq(1)
  end

  it "refuses a wrong TOTP, leaving the invitation usable and the account without a password" do
    get "/invitations/#{token}", headers: headers

    post "/invitations/#{token}/accept", params: { password: "s3nha-forte-1", code: "000000" }, headers: headers

    expect(response).to have_http_status(:unprocessable_content)
    expect(maintainer.reload.enrolled?).to be(false)
    expect(maintainer.password_digest).to be_nil
    expect(invitation.reload.usable?).to be(true)
  end

  it "refuses a used token, an expired one and an unknown one" do
    get "/invitations/#{token}", headers: headers
    code = ROTP::TOTP.new(maintainer.reload.otp_secret).now
    post "/invitations/#{token}/accept", params: { password: "s3nha-forte-1", code: code }, headers: headers
    expect(response).to have_http_status(:no_content)

    post "/invitations/#{token}/accept", params: { password: "outra-senha-9", code: code }, headers: headers
    expect(response).to have_http_status(:not_found)

    get "/invitations/token-que-nao-existe", headers: headers
    expect(response).to have_http_status(:not_found)

    other_maintainer = Maintainer.create!(email_address: "exp-#{SecureRandom.hex(3)}@rotasaude.app")
    _other, other_token = MaintainerInvitation.issue!(maintainer: other_maintainer)
    travel_to(MaintainerInvitation::TTL.from_now + 1.second) do
      get "/invitations/#{other_token}", headers: headers
      expect(response).to have_http_status(:not_found)
    end
  end
end
```

`spec/tasks/maintainer_rake_spec.rb` (siga o formato de `spec/tasks/city_rake_spec.rb` para carregar as tasks):

```ruby
require "rails_helper"
require "rake"

# Spec da API de manutenção §6: o PRIMEIRO mantenedor de cada ambiente sai daqui,
# no servidor — não há como convidá-lo pela API, porque ninguém existiria para
# conceder. A mesma task reconvida quem perdeu a janela de 24h.
RSpec.describe "maintainer rake tasks" do
  before do
    Rake::Task.clear
    Rails.application.load_tasks
  end

  def invoke(email)
    task = Rake::Task["maintainer:invite"]
    task.reenable
    output = capture_stdout { task.invoke(email) }
    Rake::Task.clear
    Rails.application.load_tasks
    output
  end

  def capture_stdout
    original = $stdout
    $stdout = StringIO.new
    yield
    $stdout.string
  ensure
    $stdout = original
  end

  it "creates the maintainer, prints a masked e-mail and a usable invitation URL" do
    output = invoke("primeiro@rotasaude.app")

    maintainer = Maintainer.find_by(email_address: "primeiro@rotasaude.app")
    expect(maintainer).to be_present
    expect(maintainer.maintainer_invitations.count).to eq(1)
    expect(output).to include("p***@rotasaude.app")
    expect(output).not_to include("primeiro@rotasaude.app")
    expect(output).to match(%r{/invitations/[A-Za-z0-9_-]{20,}})
    expect(PlatformEvent.where(name: "maintenance.maintainer.invited").count).to eq(1)
  end

  it "re-invites an existing maintainer with a brand new token" do
    invoke("segundo@rotasaude.app")
    first_digest = Maintainer.find_by(email_address: "segundo@rotasaude.app").maintainer_invitations.last.token_digest

    invoke("segundo@rotasaude.app")
    invitations = Maintainer.find_by(email_address: "segundo@rotasaude.app").maintainer_invitations.order(:created_at)

    expect(invitations.count).to eq(2)
    expect(invitations.last.token_digest).not_to eq(first_digest)
    expect(invitations.first.reload.used_at).to be_nil
  end

  it "refuses an invalid e-mail and a deactivated maintainer" do
    expect { invoke("nao-e-email") }.to raise_error(SystemExit)

    invoke("terceiro@rotasaude.app")
    Maintainer.find_by(email_address: "terceiro@rotasaude.app").deactivate!
    expect { invoke("terceiro@rotasaude.app") }.to raise_error(SystemExit)
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falham**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/maintenance/invitations_spec.rb spec/tasks/maintainer_rake_spec.rb`
Expected: FAIL — rota inexistente (404) e `Don't know how to build task 'maintainer:invite'`.

- [ ] **Step 3: Implementar o controller de convite**

`app/controllers/maintenance/invitations_controller.rb`:

```ruby
# Aceite do convite de mantenedor (spec §6), em dois passos:
#
#   GET  /invitations/:token          → gera o TOTP e devolve o material de
#                                       cadastro (uri, secret, recovery codes)
#   POST /invitations/:token/accept   → senha + código; só aqui a conta passa a
#                                       logar, e o convite é consumido
#
# Sem autenticação, por definição: quem aceita ainda não tem sessão. O que
# protege é o token de uso único, com 24h de validade, guardado só como digest.
module Maintenance
  class InvitationsController < BaseController
    allow_unauthenticated_maintainer_access

    rate_limit to: 10, within: 3.minutes,
               with: -> { render json: { error: "too_many_requests" }, status: :too_many_requests }

    def show
      invitation = usable_invitation or return head(:not_found)

      enrollment = Mfa::Enroll.call(invitation.maintainer)
      render json: {
        email_address: invitation.maintainer.email_address,
        otpauth_uri: enrollment[:otpauth_uri],
        secret: enrollment[:secret],
        recovery_codes: enrollment[:recovery_codes]
      }, status: :ok
    end

    def accept
      invitation = usable_invitation or return head(:not_found)
      maintainer = invitation.maintainer

      password = params[:password].to_s
      return render(json: { error: "weak_password" }, status: :unprocessable_content) if password.length < 12
      unless Mfa::Verify.call(maintainer, code: params[:code])
        return render(json: { error: "invalid_code" }, status: :unprocessable_content)
      end

      PlatformRecord.transaction do
        maintainer.update!(password: password, otp_enabled_at: Time.current)
        invitation.update!(used_at: Time.current)
        MaintenanceAudit.record("maintenance.maintainer.accepted", outcome: "ok", module_name: "maintainer",
                                maintainer_id: maintainer.id, credential: { "kind" => "invitation" })
      end

      head :no_content
    end

    private

    # Busca pelo DIGEST: o token em claro nunca foi gravado. Convite usado,
    # vencido, inexistente ou de conta desativada respondem igual — 404 — para
    # não distinguir "não existe" de "já usado".
    def usable_invitation
      invitation = MaintainerInvitation.find_by(token_digest: MaintainerInvitation.digest_for(params[:token]))
      return nil unless invitation&.usable?
      return nil unless invitation.maintainer.active?

      invitation
    end
  end
end
```

Em `config/routes.rb`, dentro do `scope module: :maintenance`, acrescente:

```ruby
        get  "/invitations/:token",        to: "invitations#show", as: :invitation
        post "/invitations/:token/accept", to: "invitations#accept"
```

E, no bloco novo de CORS (`config/initializers/cors.rb`), acrescente o `resource`:

```ruby
    resource "/invitations/*",
             headers: :any,
             methods: %i[get post options],
             credentials: true
```

- [ ] **Step 4: Implementar a rake**

`lib/tasks/maintainer.rake`:

```ruby
# Convite do mantenedor (spec §6). É o ÚNICO caminho fora da API, e é assim de
# propósito: o primeiro mantenedor de cada ambiente não tem quem o convide.
#
# Imprime a URL do convite porque nesta fatia não há mailer: o link é entregue
# fora de banda por quem rodou a task. O e-mail sai MASCARADO, como em
# city:invite_admin — contexto suficiente sem ecoar o endereço no terminal.
namespace :maintainer do
  mask_email = lambda do |address|
    local, _, domain = address.to_s.partition("@")
    "#{local[0]}***@#{domain}"
  end

  desc "Convida (ou reconvida) um mantenedor da API de manutenção. Uso: maintainer:invite[email]"
  task :invite, [ :email ] => :environment do |_t, args|
    unless MaintenanceApi.enabled?
      abort "[maintainer:invite] a API de manutenção não está ligada neste ambiente (#{Rails.env})"
    end

    email = args[:email].to_s.strip.downcase
    abort "uso: rails 'maintainer:invite[email]'" if email.blank?
    abort "[maintainer:invite] e-mail inválido" unless email.match?(URI::MailTo::EMAIL_REGEXP)

    maintainer = Maintainer.find_or_initialize_by(email_address: email)
    if maintainer.persisted? && !maintainer.active?
      abort "[maintainer:invite] #{mask_email.call(email)} está desativado — reative antes de reconvidar"
    end

    invitation = nil
    token = nil
    PlatformRecord.transaction do
      maintainer.save!
      # Reconvite zera senha e TOTP: quem perdeu o dispositivo recomeça o
      # cadastro inteiro, e um convite pendente antigo deixa de valer.
      maintainer.update!(password: nil, otp_secret: nil, otp_enabled_at: nil, otp_recovery_codes: [],
                         failed_attempts: 0, locked_until: nil)
      maintainer.maintainer_sessions.destroy_all
      invitation, token = MaintainerInvitation.issue!(maintainer: maintainer)
      MaintenanceAudit.record("maintenance.maintainer.invited", outcome: "ok", module_name: "maintainer",
                              maintainer_id: maintainer.id, credential: { "kind" => "rake" },
                              invitation_id: invitation.id)
    end

    origin = ENV.fetch("MAINTENANCE_FRONTEND_ORIGIN", "https://maintenance.#{Rails.env}.rotasaude.com.br")
    puts "[maintainer:invite] #{mask_email.call(email)} → convite válido por #{MaintainerInvitation::TTL.inspect}"
    puts "[maintainer:invite] #{origin}/invitations/#{token}"
  end
end
```

- [ ] **Step 5: Rodar e confirmar que passam**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/maintenance/invitations_spec.rb spec/tasks/maintainer_rake_spec.rb spec/events/platform_event_payload_guard_spec.rb`
Expected: PASS, 0 falhas.

- [ ] **Step 6: Commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/controllers/maintenance/invitations_controller.rb lib/tasks/maintainer.rake \
  config/routes.rb config/initializers/cors.rb spec/requests/maintenance/invitations_spec.rb spec/tasks/maintainer_rake_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: invite maintainers from the server and accept in two steps

rake maintainer:invite is the only way in from outside the API, which is
how the first maintainer of an environment exists at all. The invitation
is single-use, expires in 24 hours and is stored as a digest; accepting
takes a password and a confirmed TOTP before the account can log in.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 6: GraphQL mínimo (`me`) com limites

**Files:**
- Create: `app/graphql/maintenance/schema.rb`, `app/graphql/maintenance/types/base_object.rb`, `app/graphql/maintenance/types/query_type.rb`, `app/graphql/maintenance/types/maintainer_type.rb`, `app/controllers/maintenance/graphql_controller.rb`
- Modify: `Gemfile`, `Gemfile.lock`, `config/routes.rb`, `config/initializers/cors.rb`, `config/application.rb` (namespace do Zeitwerk para `app/graphql`, se necessário)
- Test: `spec/requests/maintenance/graphql_spec.rb`

**Interfaces:**
- Consumes: `MaintainerAuthentication`, `Current.maintainer_session`.
- Produces: `POST /graphql` no host da manutenção; `Maintenance::Schema` com `max_depth 10`, `max_complexity 200`, introspecção só em development; `Query.me → Maintainer!`

- [ ] **Step 1: Instalar a gem**

```bash
docker compose exec -T api bundle add graphql --version "~> 2.3" --skip-install
docker compose exec -T api bundle install
```
Expected: `Gemfile` com `gem "graphql", "~> 2.3"` e `Gemfile.lock` atualizado. **Não** rode o generator do `graphql-ruby` (ele cria um schema global `ApiSchema`, controller e rota que este plano não quer).

- [ ] **Step 2: Escrever o spec que falha**

`spec/requests/maintenance/graphql_spec.rb`:

```ruby
require "rails_helper"

# Spec da API de manutenção §8: endpoint único, POST-only, uma operação por
# requisição, com limite de profundidade, complexidade e tamanho. Nesta fatia o
# schema tem só `me` — o resto chega nas fatias de leitura e mutation.
RSpec.describe "Maintenance GraphQL", type: :request do
  let(:password) { "s3nha-forte-1" }
  let(:frontend) { "https://maintenance.rotasaude.app" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "gql-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end

  def headers = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
  def json = JSON.parse(response.body)

  def login!
    post "/session", params: { email_address: maintainer.email_address, password: password }, headers: headers
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now },
         headers: headers
    expect(response).to have_http_status(:ok)
  end

  def query!(query, **variables)
    post "/graphql", params: { query: query, variables: variables }, headers: headers
  end

  before do
    host! "maintenance-api.rotasaude.app"
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with("MAINTENANCE_FRONTEND_ORIGIN").and_return(frontend)
  end

  it "refuses an unauthenticated query" do
    query!("{ me { emailAddress } }")

    expect(response).to have_http_status(:unauthorized)
  end

  it "answers me for a verified session" do
    login!
    query!("{ me { id emailAddress } }")

    expect(response).to have_http_status(:ok)
    expect(json.dig("data", "me")).to include("id" => maintainer.id, "emailAddress" => maintainer.email_address)
  end

  it "refuses GET, because a query in the URL lands in logs and caches" do
    login!
    get "/graphql", params: { query: "{ me { id } }" }, headers: headers

    expect(response).to have_http_status(:not_found)
  end

  it "refuses a query above the size limit" do
    login!
    query!("{ me { id } } #{'#' * (Maintenance::GraphqlController::MAX_QUERY_BYTES + 1)}")

    expect(response).to have_http_status(:payload_too_large)
  end

  it "answers an error, not a crash, for an unknown field" do
    login!
    query!("{ cidades { slug } }")

    expect(response).to have_http_status(:ok)
    expect(json["errors"].first["message"]).to include("cidades")
    expect(json["data"]).to be_nil
  end

  it "keeps introspection out of the deployed environments" do
    login!
    allow(Rota).to receive(:deployed?).and_return(true)
    query!("{ __schema { types { name } } }")

    expect(json["errors"]).to be_present
  end
end
```

- [ ] **Step 3: Rodar e confirmar que falha**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/maintenance/graphql_spec.rb`
Expected: FAIL — rota inexistente, `NameError` em `Maintenance::GraphqlController`.

- [ ] **Step 4: Implementar o schema**

`app/graphql/maintenance/types/base_object.rb`:

```ruby
module Maintenance
  module Types
    class BaseObject < GraphQL::Schema::Object
    end
  end
end
```

`app/graphql/maintenance/types/maintainer_type.rb`:

```ruby
module Maintenance
  module Types
    class MaintainerType < BaseObject
      description "Uma conta da API de manutenção"

      field :id, ID, null: false
      field :email_address, String, null: false
      field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    end
  end
end
```

`app/graphql/maintenance/types/query_type.rb`:

```ruby
module Maintenance
  module Types
    class QueryType < BaseObject
      description "Consultas da API de manutenção"

      field :me, MaintainerType, null: false, description: "O mantenedor da sessão corrente"

      def me = context.fetch(:maintainer)
    end
  end
end
```

`app/graphql/maintenance/schema.rb`:

```ruby
# Schema da API de manutenção (spec §8). Nesta fatia responde só `me`.
#
# Os limites não são enfeite: o endpoint é servido na internet pública e um
# mantenedor autenticado tem poderes totais, então uma query fundo demais ou
# complexa demais é problema mesmo vinda de dentro.
module Maintenance
  class Schema < GraphQL::Schema
    query Types::QueryType

    max_depth 10
    max_complexity 200

    def self.unauthorized_object(error)
      raise GraphQL::ExecutionError, "não autorizado: #{error.type.graphql_name}"
    end
  end
end
```

Se o Zeitwerk reclamar de `app/graphql/maintenance/...` esperando a constante `Maintenance::...` como raiz, siga o padrão que `config/application.rb` já usa para `app/protocols` e `app/messaging`: pré-declare `module Maintenance; end` e registre o diretório com namespace. Se não reclamar, não mexa.

- [ ] **Step 5: Implementar o controller e a rota**

`app/controllers/maintenance/graphql_controller.rb`:

```ruby
# Endpoint único da API de manutenção (spec §8): POST, uma operação por
# requisição, sem lote. GET não existe — query em URL vai parar em log e cache.
module Maintenance
  class GraphqlController < BaseController
    MAX_QUERY_BYTES = 10_000
    INTROSPECTION = /\b__(schema|type)\b/

    def execute
      query = params[:query].to_s
      return head(:payload_too_large) if query.bytesize > MAX_QUERY_BYTES

      # Introspecção só em development: em ambiente publicado o schema é lido do
      # SDL em `contracts` (ADR-0015), não do endpoint. A checagem é aqui, e não
      # em Schema.disable_introspection_entry_points, porque aquela roda uma vez
      # no carregamento da classe — congelaria a decisão do ambiente que estava
      # ativo quando a classe carregou.
      if Rota.deployed? && query.match?(INTROSPECTION)
        return render(json: { errors: [ { message: "introspecção desligada em ambiente publicado" } ] }, status: :ok)
      end

      result = Schema.execute(
        query,
        variables: params[:variables] || {},
        operation_name: params[:operationName],
        context: {
          maintainer: current_maintainer,
          maintainer_session: Current.maintainer_session,
          request_id: request.request_id
        }
      )

      render json: result
    end
  end
end
```

Em `config/routes.rb`, dentro do `scope module: :maintenance`:

```ruby
        post "/graphql", to: "graphql#execute"
```

Em `config/initializers/cors.rb`, no bloco da manutenção:

```ruby
    resource "/graphql",
             headers: :any,
             methods: %i[post options],
             credentials: true
```

- [ ] **Step 6: Rodar, suíte completa e commit**

```bash
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/maintenance
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
```
Expected: 0 falhas.

```bash
cd apps/api
/opt/homebrew/bin/git add Gemfile Gemfile.lock app/graphql app/controllers/maintenance/graphql_controller.rb \
  config/routes.rb config/initializers/cors.rb spec/requests/maintenance/graphql_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: serve the maintenance GraphQL endpoint with a single me query

POST only, one operation per request, capped at depth 10, complexity 200
and 10 KB, with introspection off outside development. The schema starts
with me; cities, tokens and mutations arrive in the next slices.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

## Verificação final do plano

- [ ] Suíte completa com o worker parado: 0 falhas.
- [ ] `docker compose exec -T api bin/rails runner 'puts MaintenanceApi.enabled?(env: "production", flag: "true")'` imprime `false`.
- [ ] Um `curl` no host errado devolve 404: `curl -s -o /dev/null -w '%{http_code}\n' -H 'Origin: …' http://curitiba.localhost:3030/session` (o host de cidade não tem essa rota de manutenção).
- [ ] `rake maintainer:invite` em development cria o convite e imprime a URL; o aceite pelo endpoint deixa a conta `enrolled?`.
- [ ] `grep -rn "maintenance\." app/events/maintenance_audit.rb` e a lista R18 têm exatamente os mesmos nomes.
- [ ] Nenhum `staging.key`, token em claro ou `otp_secret` em diff, log ou relatório.

## Fora deste plano

- **Mutations de mantenedor** (`inviteMaintainer`, `deactivateMaintainer`) e **tokens de serviço**: Plano 3.
- **Leitura de cidades** (`cities`, `city(slug:)`), limites por cidade e spec de schema: Plano 4.
- **Mutations por módulo** e o par tentativa/resultado aplicado a escrita real: Plano 5.
- **SDL publicado em `contracts`**: Plano 6.
- **Mailer do convite** e o frontend `maintenance.<env>`: spec própria, depois do Plano 3.
- **`auditEvents` como query**: entra no Plano 3, junto com a restrição de ela ser só para sessão humana.
