# Banco por cidade — Plano 3B: Grant assinado (operador entra na cidade) e callback único do gov.br

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deixar o operador entrar numa cidade por um grant assinado de uso único, só para leitura e auditado nos dois
bancos, e levar o login gov.br para um callback único em `auth.*` que entrega a sessão à cidade pelo mesmo grant.

**Architecture:** Um grant é uma linha `city_grants` no banco de plataforma (cidade, tipo `operator`/`user`, sujeito,
validade de 60 s, consumo atômico) mais um token assinado com a chave da plataforma que carrega o id da linha e o slug
da cidade. Quem emite o grant é o console (operador) ou o callback do gov.br (usuário); quem consome é a cidade, em
`POST /session/grant`, que cria uma `Session` local. A sessão da cidade passa a aceitar um operador no lugar de um
usuário; toda sessão de operador é negada por padrão e só alcança as ações que um controller libera explicitamente
(os painéis de leitura e a própria sessão).

**Tech Stack:** Rails 8.1.3, Ruby 3.3.6, PostgreSQL, RSpec, ROTP, `ActiveSupport::MessageVerifier`, Docker Compose.

**Spec:** `docs/superpowers/specs/2026-09-12-banco-por-cidade-design.md` — §5 ("Operador entrando numa cidade", "gov.br").
**Planos anteriores:** 1, 2 e 3, todos mergeados em `main` do repo `apps/api` (Plano 3 = merge `6f55708`).

## Decisões que este plano carrega

Tomadas com o usuário em 2026-09-14, antes da escrita:

- **Operador dentro da cidade é SÓ LEITURA.** Lê os painéis `/admin/api/*` e a própria sessão (`GET`/`DELETE /session`).
  Não convida, não revoga, não desativa, não publica, não edita protocolo, não usa MFA da cidade. Implementado como
  negação por padrão: cada ação precisa ser liberada por nome.
- **Mudança de schema da cidade aplicada recriando os bancos de dev** `rota_saude_city_curitiba` e
  `rota_saude_city_maringa` (dados de exemplo repovoados pelos seeds). Nenhuma task de migração de cidades existentes
  — `city:migrate:all` continua sendo do Plano 4.
- **Segurança: só o teto de tentativas de TOTP entra.** Replay de TOTP, tempo de resposta no login da cidade e troca do
  id de sessão após o TOTP continuam estacionados. Enrollment de MFA e reset de senha do operador ficam fora.
- **Nenhum frontend é tocado** (Plano 6). O redirecionamento para a cidade usa um template de URL por env var; a prova
  é por request spec e `curl`.

Decisões de desenho tomadas na escrita (registradas aqui para a review):

- **Grant vive 60 segundos e é de uso único**, com a linha `city_grants` como fonte da verdade: o token só aponta para
  ela. Consumo é um `UPDATE ... WHERE consumed_at IS NULL AND expires_at > now()` que precisa afetar exatamente 1 linha.
  O `kind` e o sujeito vêm da LINHA, nunca do token.
- **O token é assinado com `Rails.application.message_verifier(:city_grant)`** (chave derivada do `secret_key_base` —
  "chave da plataforma"). Chave por cidade é do Plano 6.
- **Sessão de operador na cidade vale 1 hora** a partir da criação e cai na hora se o operador for desativado na
  plataforma.
- **Auditoria da entrada do operador:** primeiro `PlatformEvent operator.city_access` (plataforma), depois, numa
  transação da cidade, a `Session` e o `DomainEvent operator.city_access`. Os dois bancos não têm transação comum; se a
  cidade falhar depois da auditoria da plataforma, sobra o registro de uma TENTATIVA sem sessão — o oposto (sessão sem
  auditoria) não pode acontecer.
- **gov.br:** o `state` é um token assinado (`message_verifier(:govbr_state)`, 10 minutos) com o slug da cidade e um
  `nonce`; o callback em `auth.*` exige que o `nonce` do `id_token` seja o mesmo. PKCE fica fora (não há cliente
  público: quem troca o `code` é o servidor com `client_secret`).
- **Redirecionamento para a cidade:** `CITY_DASHBOARD_URL_TEMPLATE` (default `http://%{slug}.localhost:5175/dashboard/`),
  com o grant em `?grant=`.

## Global Constraints

- **Cookie de sessão é host-only: NUNCA `domain:`** (spec §5). Vale para `session_id` e `operator_session_id`.
- **Resolver a cidade ANTES de autenticar** (`CityResolution` → `Authentication`); nunca `prepend_before_action` para
  autenticação.
- **Operador mora só na plataforma.** Na cidade existe só a `Session` com `operator_id` — nenhuma conta replicada.
- **Grant de uma cidade não vale em outra**: a cidade confere o slug do token contra a cidade do host ANTES de consumir.
- **Sessão de operador na cidade é negada por padrão**; só `Admin::Api::*` e `SessionsController#show/#destroy` a aceitam.
- **O banco de plataforma nunca guarda dado de cidadão.** `city_grants` guarda só ids. `PlatformEvent` recusa chave de
  payload que contenha `email`, `cpf`, `phone`, `wa_id`, `provider_uid`, `body`, `name` (salvo `phone_number_id`,
  `city_name`) ou `from`.
- **Subdomínios reservados (`admin`, `api`, `auth`, `www`) nunca resolvem cidade.** `admin.*` é o console; `auth.*` é
  o callback do gov.br.
- **`connects_to` só uma vez, no corpo de `CityRecord`.** Cidade entra por `CityConnection.with`.
- **Ponteiros de ADR em comentário só na faixa ADR-0001..ADR-0015.** Nada de constante top-level em spec.
- **Todo comando roda no container**, a partir da raiz do monorepo: `docker compose exec -T api <cmd>`; rspec com
  `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Nunca `bundle` no host.
- **Sondagens só como spec de rascunho** (apagado depois); `rails runner` só para leitura.
- **Operações destrutivas permitidas neste plano: SÓ** `dropdb` de `rota_saude_city_curitiba` e
  `rota_saude_city_maringa` na Task 1 (decisão do usuário). Nada em `rota_saude_development`, `rota_saude_test`,
  `rota_saude_platform_*`. `city:test_databases` recarrega o schema nos bancos de cidade de TESTE (é o procedimento
  normal da suíte).
- **Migração de plataforma é aditiva** e roda com `db:migrate:platform` (dev e test); o dump
  `db/platform_schema.rb` é regenerado com `bin/rails db:schema:dump:platform` em development e commitado.
- **Suíte verde ao fim de CADA task.** Nenhum spec enfraquecido; exemplo removido precisa de justificativa e do
  invariante provado em outro spec nomeado.
- **Commits em inglês**, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- **Depois de criar diretório novo em `app/`**, o `api-dev` em execução precisa de `docker compose restart api` para
  enxergá-lo (a suíte não precisa).

## Estado herdado (repo `apps/api`, `main` em `6f55708`)

| Peça | Onde | Hoje |
|---|---|---|
| `Session` | `app/models/session.rb`; `db/city_schema.rb` | `belongs_to :user`; `sessions.user_id NOT NULL`, FK para `users` |
| `Authentication` | `app/controllers/concerns/authentication.rb` | `before_action :require_authentication`; cookie `session_id` assinado, `permanent`; `allow_unauthenticated_access` |
| Controllers com `Authentication` | `admin/api/base_controller.rb`, `sessions_controller.rb`, `mfa_controller.rb`, `passwords_controller.rb`, `publications_controller.rb`, `authoring/protocols_controller.rb`, `setup_controller.rb` | todos assumem `Current.user` presente |
| `Admin::Api::BaseController` | | `require_city_membership` usa `current_user.memberships` |
| `SessionsController` | | `create`, `show`, `destroy`, `govbr_callback` (provisório, no host da cidade) |
| `Authenticator::GovBr` | `app/auth/authenticator/gov_br.rb` | `authenticate(code:)` = troca do code + `find_or_provision_user` + evento `identity.govbr_login`; `exchange_code_for_claims`, `redirect_uri` via credentials/ENV |
| Console | `app/controllers/operators/*`, `OperatorAuthentication`, `PlatformConsoleHost` | login de operador com TOTP; `PENDING_MFA_WINDOW`, `OPERATOR_SESSION_TTL = 12.hours`; `operator_sessions` sem contador de tentativas |
| `CityCatalog` | `app/models/city_catalog.rb` | `find_by_host`, `reserved_host?`, `console_host?` |
| `Platform.audit` | `app/events/platform.rb` | grava `PlatformEvent`; guarda de chaves em `app/models/platform_event.rb` e `spec/events/platform_event_payload_guard_spec.rb` (lista estática de nomes de evento) |
| Pendings | | `rotate_token_spec.rb` e `admin/api/reports_spec.rb`: `skip "Plano 3B: grant de operador ..."` |

Suíte: **490 examples, 0 failures, 2 pending**. Dev: `curitiba` e `maringa` ativas, com seeds e dataset de demo.

## File Structure

| Arquivo | Task | Responsabilidade |
|---|---|---|
| `db/city_migrate/20260914000001_allow_operator_city_sessions.rb`, `db/city_schema.rb` | 1 | `sessions.operator_id`, `user_id` opcional, CHECK de exatamente um ator |
| `app/models/session.rb` | 1 | ator: usuário OU operador; expiração da sessão de operador |
| `app/controllers/concerns/authentication.rb` | 1 | negação por padrão para sessão de operador; `allow_operator_grant_access` |
| `app/controllers/admin/api/base_controller.rb`, `app/controllers/sessions_controller.rb` | 1 | liberam a sessão de operador (leitura / própria sessão) |
| `spec/support/city_request_auth.rb` | 1 | `sign_in_operator_grant(operator)` |
| `spec/requests/operator_city_session_spec.rb`, `spec/models/session_spec.rb`, `spec/architecture/operator_grant_access_spec.rb` | 1 | só leitura, expiração, allowlist |
| `db/platform_migrate/20260914000002_create_city_grants.rb`, `db/platform_schema.rb` | 2 | tabela de grants |
| `app/models/city_grant.rb`, `app/services/city_grants.rb` | 2 | emitir e consumir grant |
| `spec/services/city_grants_spec.rb` | 2 | uso único, validade, cidade certa |
| `app/services/city_dashboard_url.rb` | 3 | URL da cidade com `?grant=` |
| `app/controllers/operators/city_grants_controller.rb`, `app/controllers/sessions_controller.rb`, `config/routes.rb` | 3 | console emite, cidade consome |
| `spec/requests/operators/city_grants_spec.rb`, `spec/requests/session_grant_spec.rb` | 3 | fluxo completo e recusas |
| `app/models/city_catalog.rb`, `app/constraints/platform_auth_host.rb`, `app/controllers/govbr/callbacks_controller.rb`, `app/auth/authenticator/gov_br.rb` | 4 | callback único em `auth.*` |
| `spec/requests/govbr_login_spec.rb`, `spec/auth/gov_br_spec.rb` | 4 | state, nonce, cidade certa |
| `db/platform_migrate/20260914000003_add_mfa_failed_attempts_to_operator_sessions.rb`, `app/controllers/operators/sessions_controller.rb` | 5 | teto de tentativas de TOTP |
| `README.md`, `deploy/production/deploy.yml`, `deploy/development/deploy.yml` | 6 | envs novos e fluxo de entrada |

---

### Task 1: Sessão da cidade aceita operador, negado por padrão

**Files:**
- Create: `db/city_migrate/20260914000001_allow_operator_city_sessions.rb`
- Modify: `db/city_schema.rb` (versão e bloco `sessions`)
- Modify: `app/models/session.rb`
- Modify: `app/controllers/concerns/authentication.rb`
- Modify: `app/controllers/admin/api/base_controller.rb`, `app/controllers/sessions_controller.rb`
- Modify: `spec/support/city_request_auth.rb`
- Create: `spec/models/session_spec.rb`, `spec/requests/operator_city_session_spec.rb`, `spec/architecture/operator_grant_access_spec.rb`

**Interfaces:**
- Consumes: `Operator` (plataforma: `active?`, `email_address`, `mfa_enrolled?`), `CityRequestAuth#sign_in_as`, `TEST_CITY_B`.
- Produces:
  - `sessions.operator_id uuid NULL`, `sessions.user_id` agora `NULL`, CHECK `ck_sessions_exactly_one_actor`.
  - `Session::OPERATOR_GRANT_TTL = 1.hour`; `Session#operator_grant? → Boolean`; `Session#operator → Operator|nil`;
    `Session#usable? → Boolean` (sessão de usuário: sempre; de operador: criada há menos de 1 h e operador ativo).
  - `Authentication.allow_operator_grant_access(only: :all | Array<Symbol>)` e o class_attribute
    `operator_grant_actions` (default `[]`). Sessão de operador numa ação não liberada → `403 {"error":"operator_read_only"}`.
  - `CityRequestAuth#sign_in_operator_grant(operator) → Session`.
  - `GET /session` com sessão de operador → `{ id, email_address, mfa_enrolled, operator: true, mfa_verified_at: nil, memberships: [] }`.

- [ ] **Step 1: Write the failing tests**

Em `spec/support/city_request_auth.rb`, dentro de `module CityRequestAuth`, acrescente depois de `sign_in_as`:

```ruby
  # Sessão de operador aberta por grant (Plano 3B): a linha vive no banco da
  # cidade corrente, sem usuário, e o cookie é o mesmo `session_id` da cidade.
  def sign_in_operator_grant(operator)
    session = Session.create!(operator_id: operator.id, user_agent: "rspec", ip_address: "127.0.0.1")
    jar = ActionDispatch::TestRequest.create.cookie_jar
    jar.signed[:session_id] = session.id
    cookies[:session_id] = jar[:session_id]
    session
  end
```

Crie `spec/models/session_spec.rb`:

```ruby
require "rails_helper"

# Sessão da cidade: exatamente UM ator — usuário da cidade ou operador da
# plataforma (entrada por grant, Plano 3B). A regra vale no modelo e no banco.
RSpec.describe Session do
  include ActiveSupport::Testing::TimeHelpers

  let(:user) { User.create!(email_address: "s-#{SecureRandom.hex(3)}@x.com", password: "secret123") }
  let(:operator) do
    Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end

  it "is valid with a user only, or with an operator only" do
    expect(described_class.new(user: user)).to be_valid
    expect(described_class.new(operator_id: operator.id)).to be_valid
  end

  it "is invalid with neither actor or with both" do
    expect(described_class.new).not_to be_valid
    expect(described_class.new(user: user, operator_id: operator.id)).not_to be_valid
  end

  it "the city database refuses a session with neither actor or with both, even skipping validation" do
    expect { described_class.new.save!(validate: false) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_sessions_exactly_one_actor/)
    expect { described_class.new(user: user, operator_id: operator.id).save!(validate: false) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_sessions_exactly_one_actor/)
  end

  describe "#usable?" do
    it "a user session is always usable" do
      expect(described_class.create!(user: user)).to be_usable
    end

    it "an operator session is usable for one hour while the operator is active" do
      session = described_class.create!(operator_id: operator.id)
      expect(session).to be_operator_grant
      expect(session.operator).to eq(operator)
      expect(session).to be_usable

      travel 59.minutes do
        expect(session).to be_usable
      end
      travel 61.minutes do
        expect(session).not_to be_usable
      end
    end

    it "an operator session stops being usable once the operator is deactivated" do
      session = described_class.create!(operator_id: operator.id)
      operator.update!(deactivated_at: Time.current)

      expect(session).not_to be_usable
    end
  end
end
```

Crie `spec/requests/operator_city_session_spec.rb`:

```ruby
require "rails_helper"

# Operador dentro da cidade (grant, Plano 3B) é SÓ LEITURA — decisão do usuário.
# Negação por padrão: toda ação que não libera explicitamente a sessão de
# operador responde 403 operator_read_only.
RSpec.describe "Operator session inside a city", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  let!(:operator) do
    Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end

  def json = JSON.parse(response.body)

  it "reads the city panels without any city membership" do
    sign_in_operator_grant(operator)

    %w[/admin/api/reports /admin/api/overview].each do |path|
      get path, params: { period: "30d" }
      expect(response).to have_http_status(:ok), "#{path} respondeu #{response.status}"
    end
  end

  it "GET /session describes the operator, with no memberships" do
    sign_in_operator_grant(operator)

    get "/session"

    expect(response).to have_http_status(:ok)
    expect(json).to eq("id" => operator.id, "email_address" => operator.email_address, "mfa_enrolled" => true,
                       "operator" => true, "mfa_verified_at" => nil, "memberships" => [])
  end

  it "is refused on every action that is not explicitly read-only" do
    sign_in_operator_grant(operator)
    requests = [
      [ :post, "/setup/invitations", { email: "x@x.com", role: "viewer" } ],
      [ :get,  "/setup/memberships", {} ],
      [ :post, "/setup/users/#{SecureRandom.uuid}/deactivate", {} ],
      [ :post, "/setup/memberships/#{SecureRandom.uuid}/revoke", {} ],
      [ :post, "/mfa/enroll", {} ],
      [ :post, "/mfa/step_up", { code: "123456" } ],
      [ :get,  "/authoring/protocols/definition", { name: "x", version: 1 } ],
      [ :post, "/authoring/protocols/draft", { definition: { name: "x" } } ],
      [ :post, "/protocols/1/publish", {} ]
    ]

    requests.each do |verb, path, params|
      send(verb, path, params: params)
      expect(response).to have_http_status(:forbidden), "#{verb.upcase} #{path} respondeu #{response.status}"
      expect(json).to eq("error" => "operator_read_only"), "#{verb.upcase} #{path} respondeu #{response.body}"
    end
  end

  it "logs out: DELETE /session destroys the operator session" do
    session = sign_in_operator_grant(operator)

    delete "/session"

    expect(response).to have_http_status(:no_content)
    expect(Session.exists?(session.id)).to be(false)
  end

  it "expires one hour after it was opened" do
    sign_in_operator_grant(operator)

    travel 61.minutes do
      get "/admin/api/reports", params: { period: "30d" }
      expect(response).to have_http_status(:unauthorized)
    end
  end

  it "stops working as soon as the operator is deactivated on the platform" do
    sign_in_operator_grant(operator)
    operator.update!(deactivated_at: Time.current)

    get "/admin/api/reports", params: { period: "30d" }

    expect(response).to have_http_status(:unauthorized)
  end

  it "only exists in the city where it was opened" do
    sign_in_operator_grant(operator)
    create(:city, slug: TEST_CITY_B.slug, status: "active", database_url: city_database_url("rota_saude_test_city_b"))

    get "/admin/api/reports", params: { period: "30d" }, headers: { "HOST" => "#{TEST_CITY_B.slug}.rotasaude.app" }

    expect(response).to have_http_status(:unauthorized)
  end

  it "does not change what a city user can do" do
    user = User.create!(email_address: "adm-#{SecureRandom.hex(3)}@x.com", password: "secret123")
    Membership.create!(user: user, role: "municipal_admin", granted_at: Time.current)
    sign_in_as(user)

    get "/setup/memberships"

    expect(response).to have_http_status(:ok)
  end
end
```

Crie `spec/architecture/operator_grant_access_spec.rb`:

```ruby
require "rails_helper"

# Operador dentro da cidade é só leitura (Plano 3B). A negação é por padrão, e
# esta guarda fixa QUEM libera: só a API de leitura da cidade e a própria sessão.
# Um controller novo que chame allow_operator_grant_access precisa passar por aqui.
RSpec.describe "Operator grant access allowlist" do
  it "only Admin::Api (all actions) and SessionsController#show/#destroy accept an operator session" do
    Rails.application.eager_load!

    allowed = ApplicationController.descendants
      .select { |klass| klass.include?(Authentication) && klass.name.present? }
      .reject { |klass| klass.operator_grant_actions == [] }
      .to_h { |klass| [ klass.name, klass.operator_grant_actions ] }

    expect(allowed).to include("Admin::Api::BaseController" => :all, "SessionsController" => %i[show destroy])
    expect(allowed.keys - [ "SessionsController" ]).to all(start_with("Admin::Api::"))
    expect(allowed.except("SessionsController").values.uniq).to eq([ :all ])
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/session_spec.rb spec/requests/operator_city_session_spec.rb spec/architecture/operator_grant_access_spec.rb`
Expected: FAIL — `unknown attribute 'operator_id' for Session` (a coluna ainda não existe) e
`undefined method 'operator_grant_actions'`.

- [ ] **Step 3: Change the city schema**

Crie `db/city_migrate/20260914000001_allow_operator_city_sessions.rb`:

```ruby
# Sessão da cidade aceita um operador da plataforma no lugar de um usuário
# (grant assinado, Plano 3B — spec banco-por-cidade §5). Exatamente um ator por
# sessão. O operador NÃO existe no banco da cidade: operator_id é só o id da
# conta de plataforma, sem chave estrangeira.
class AllowOperatorCitySessions < ActiveRecord::Migration[8.1]
  def change
    change_column_null :sessions, :user_id, true
    add_column :sessions, :operator_id, :uuid
    add_index :sessions, :operator_id, name: "index_sessions_on_operator_id"
    add_check_constraint :sessions, "(user_id IS NULL) <> (operator_id IS NULL)", name: "ck_sessions_exactly_one_actor"
  end
end
```

Em `db/city_schema.rb`, troque a versão `ActiveRecord::Schema[8.1].define(version: 2026_09_13_000010) do` por
`ActiveRecord::Schema[8.1].define(version: 2026_09_14_000001) do` e o bloco `create_table "sessions"` inteiro por:

```ruby
  create_table "sessions", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "created_at", null: false
    t.string "ip_address"
    t.datetime "mfa_verified_at"
    t.uuid "operator_id"
    t.datetime "updated_at", null: false
    t.string "user_agent"
    t.uuid "user_id"
    t.index ["operator_id"], name: "index_sessions_on_operator_id"
    t.index ["user_id"], name: "index_sessions_on_user_id"
    t.check_constraint "(user_id IS NULL) <> (operator_id IS NULL)", name: "ck_sessions_exactly_one_actor"
  end
```

`add_foreign_key "sessions", "users"` continua (FK aceita `NULL`).

Recarregue o schema nos bancos de cidade de TESTE (procedimento normal da suíte; só eles):

Run: `docker compose exec -T -e RAILS_ENV=test -e POSTGRES_PASSWORD=postgres api bin/rails city:test_databases`
Expected: `schema de cidade carregado em rota_saude_test_city_a` e `... city_b`.

Run: `psql -d rota_saude_test_city_a -Atc "select column_name, is_nullable from information_schema.columns where table_name='sessions' and column_name in ('user_id','operator_id') order by 1"`
Expected: `operator_id|YES` e `user_id|YES`.

- [ ] **Step 4: Write the implementation**

`app/models/session.rb` inteiro:

```ruby
# Sessão da cidade (ADR-0011), no banco da cidade. Exatamente um ator:
#   - usuário da cidade (login por senha ou gov.br), ou
#   - operador da plataforma que entrou por grant assinado (Plano 3B) — sem
#     conta na cidade; operator_id é o id da conta de plataforma.
# Sessão de operador é SÓ LEITURA (Authentication nega por padrão), vale
# OPERATOR_GRANT_TTL e cai quando o operador é desativado na plataforma.
class Session < ApplicationRecord
  OPERATOR_GRANT_TTL = 1.hour

  belongs_to :user, optional: true

  validate :exactly_one_actor

  def operator_grant?
    operator_id.present?
  end

  def operator
    return nil unless operator_grant?

    @operator ||= Operator.find_by(id: operator_id)
  end

  def usable?
    return true unless operator_grant?

    created_at > OPERATOR_GRANT_TTL.ago && operator&.active? == true
  end

  private

  def exactly_one_actor
    return if user_id.present? ^ operator_id.present?

    errors.add(:base, "a session belongs to exactly one of a user or an operator")
  end
end
```

`app/controllers/concerns/authentication.rb` — no bloco `included do`, depois de `before_action :require_authentication`:

```ruby
    # Sessão de operador aberta por grant (Plano 3B) é negada por padrão. Cada
    # controller libera por nome as ações de LEITURA que aceitam operador.
    class_attribute :operator_grant_actions, default: [], instance_writer: false
```

no bloco `class_methods do`, depois de `allow_unauthenticated_access`:

```ruby
    def allow_operator_grant_access(only: :all)
      self.operator_grant_actions = only == :all ? :all : Array(only).map(&:to_sym)
    end
```

e troque `require_authentication` e `find_session_by_cookie` por:

```ruby
  def require_authentication
    return request_authentication unless resume_session
    return if !Current.session.operator_grant? || operator_grant_access_allowed?

    render json: { error: "operator_read_only" }, status: :forbidden
  end

  def operator_grant_access_allowed?
    actions = self.class.operator_grant_actions
    actions == :all || actions.include?(action_name.to_sym)
  end
```

```ruby
  def find_session_by_cookie
    return nil unless cookies.signed[:session_id]

    session = Session.find_by(id: cookies.signed[:session_id])
    session if session&.usable?
  end
```

Acrescente ao comentário de cabeçalho do concern a linha:
`#   - sessão de operador (grant, Plano 3B) é negada por padrão: ver allow_operator_grant_access.`

`app/controllers/admin/api/base_controller.rb` — logo depois de `include Authentication`:

```ruby
  # Operador que entrou na cidade por grant (Plano 3B) lê todos os painéis: este
  # namespace não tem escrita (critério §10).
  allow_operator_grant_access
```

e o método do gate passa a:

```ruby
  def require_city_membership
    return if Current.session.operator_grant?
    return if current_user.memberships.active.exists?

    render json: { error: "no_city_membership" }, status: :forbidden
  end
```

`app/controllers/sessions_controller.rb` — depois de `allow_unauthenticated_access ...`:

```ruby
  # Operador dentro da cidade (grant, Plano 3B) vê e encerra a própria sessão; nada mais.
  allow_operator_grant_access only: %i[show destroy]
```

a ação `show` passa a:

```ruby
  # GET /session — quem está autenticado agora (útil para a UI inicializar).
  def show
    return render(json: serialize_operator_grant(Current.session)) if Current.session.operator_grant?
    return head :unauthorized unless current_user

    render json: serialize(current_user)
  end
```

e, em `private`, acrescente:

```ruby
  # Sessão de operador aberta por grant (Plano 3B): mesmo formato do SessionUser;
  # operador não tem membership na cidade.
  def serialize_operator_grant(session)
    operator = session.operator
    {
      id: operator.id,
      email_address: operator.email_address,
      mfa_enrolled: operator.mfa_enrolled?,
      operator: true,
      mfa_verified_at: nil,
      memberships: []
    }
  end
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/session_spec.rb spec/requests/operator_city_session_spec.rb spec/architecture/operator_grant_access_spec.rb spec/requests/city_session_isolation_spec.rb spec/requests/admin/api`
Expected: PASS, 0 failures.

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 2 pending.

- [ ] **Step 6: Recreate the two development city databases**

Decisão do usuário: o schema novo entra nos bancos de dev recriando-os. É a ÚNICA operação destrutiva do plano e só
atinge `rota_saude_city_curitiba` e `rota_saude_city_maringa` (dados de exemplo). Nada mais é apagado.

```bash
dropdb --force rota_saude_city_curitiba && dropdb --force rota_saude_city_maringa
```
Expected: sem saída (`--force` derruba as conexões abertas pelo `api-dev`).

```bash
docker compose exec -T api bin/rails city:dev_baseline && docker compose exec -T api bin/rails db:seed && docker compose exec -T api bin/rails db:seed:demo && docker compose exec -T api bin/rails db:seed:demo:verify
```
Expected: as duas cidades `criado` + `schema de cidade carregado` + `active`; os blocos de seed das duas cidades; e
`[dashboard_demo:verify] OK — all panels populated for every dev city`.

```bash
psql -d rota_saude_city_curitiba -Atc "select version from schema_migrations; select conname from pg_constraint where conname='ck_sessions_exactly_one_actor'"
```
Expected: `20260914000001` e `ck_sessions_exactly_one_actor`.

```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: curitiba.localhost" -H "Content-Type: application/json" -d '{"email_address":"admin@curitiba.demo","password":"dev-password"}' http://localhost:3030/session
```
Expected: `201` (login da cidade continua funcionando sobre o schema novo).

- [ ] **Step 7: Commit**

```bash
git -C apps/api add db/city_migrate/20260914000001_allow_operator_city_sessions.rb db/city_schema.rb app/models/session.rb app/controllers/concerns/authentication.rb app/controllers/admin/api/base_controller.rb app/controllers/sessions_controller.rb spec/support/city_request_auth.rb spec/models/session_spec.rb spec/requests/operator_city_session_spec.rb spec/architecture/operator_grant_access_spec.rb
git -C apps/api commit -m "Let a city session belong to a platform operator, read-only by default

A city session now holds exactly one actor: a city user or a platform
operator. Operator sessions are refused on every action unless a
controller allows them by name; only the read-only city admin API and
the session's own show/destroy do. They expire after one hour and die
with the operator's deactivation.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: Grant assinado de uso único

**Files:**
- Create: `db/platform_migrate/20260914000002_create_city_grants.rb`
- Modify: `db/platform_schema.rb` (regenerado pelo dump)
- Create: `app/models/city_grant.rb`
- Create: `app/services/city_grants.rb`
- Create: `spec/services/city_grants_spec.rb`

**Interfaces:**
- Consumes: `City` (plataforma: `id`, `slug`), `PlatformRecord`.
- Produces:
  - tabela `city_grants` (plataforma): `id uuid`, `city_id uuid NOT NULL` (FK `cities`), `kind string NOT NULL`
    (CHECK `operator`/`user`), `subject_id uuid NOT NULL`, `expires_at datetime NOT NULL`, `consumed_at datetime`,
    timestamps.
  - `CityGrant < PlatformRecord` com `KINDS = %w[operator user]`.
  - `CityGrants::TTL = 60.seconds`.
  - `CityGrants.issue(city:, kind:, subject_id:) → String` (token assinado; o payload tem só `jti` e o slug da cidade).
  - `CityGrants.redeem(token:, city:) → CityGrant | nil` — nil para token inválido, adulterado, vencido, de outra
    cidade ou já consumido. Consome atomicamente; o `kind` e o `subject_id` saem da linha.

- [ ] **Step 1: Write the failing test**

Crie `spec/services/city_grants_spec.rb`:

```ruby
require "rails_helper"

# Grant assinado de entrada numa cidade (spec §5, Plano 3B): 60 s, uso único,
# válido só na cidade para a qual foi emitido. A linha em city_grants é a fonte
# da verdade; o token só aponta para ela.
RSpec.describe CityGrants do
  include ActiveSupport::Testing::TimeHelpers

  let(:city) { create(:city) }
  let(:other_city) { create(:city) }
  let(:subject_id) { SecureRandom.uuid }

  def issue(kind: "operator", for_city: city)
    described_class.issue(city: for_city, kind: kind, subject_id: subject_id)
  end

  it "issues a token backed by an unconsumed grant row that expires in 60 seconds" do
    token = nil
    expect { token = issue }.to change(CityGrant, :count).by(1)

    grant = CityGrant.order(:created_at).last
    expect(token).to be_a(String)
    expect(grant).to have_attributes(city_id: city.id, kind: "operator", subject_id: subject_id, consumed_at: nil)
    expect(grant.expires_at).to be_within(2.seconds).of(60.seconds.from_now)
  end

  it "the token carries only the grant id and the city slug" do
    payload = Rails.application.message_verifier(:city_grant).verified(issue, purpose: :city_grant)

    expect(payload.keys).to contain_exactly("jti", "city")
    expect(payload["city"]).to eq(city.slug)
  end

  it "redeems once, in the city it was issued for, returning the grant from the database" do
    token = issue(kind: "user")

    grant = described_class.redeem(token: token, city: city)

    expect(grant).to have_attributes(kind: "user", subject_id: subject_id, city_id: city.id)
    expect(grant.consumed_at).to be_present
    expect(described_class.redeem(token: token, city: city)).to be_nil
  end

  it "refuses another city without consuming the grant" do
    token = issue

    expect(described_class.redeem(token: token, city: other_city)).to be_nil
    expect(CityGrant.order(:created_at).last.consumed_at).to be_nil
    expect(described_class.redeem(token: token, city: city)).to be_present
  end

  it "refuses an expired grant" do
    token = issue

    travel 61.seconds do
      expect(described_class.redeem(token: token, city: city)).to be_nil
    end
  end

  it "refuses a grant whose row expired even if the token signature is still valid" do
    token = issue
    CityGrant.order(:created_at).last.update!(expires_at: 1.second.ago)

    expect(described_class.redeem(token: token, city: city)).to be_nil
  end

  it "refuses a tampered token, a token with another purpose and a non-string token" do
    token = issue
    foreign = Rails.application.message_verifier(:city_grant).generate(
      { "jti" => CityGrant.order(:created_at).last.id, "city" => city.slug }, purpose: :other
    )

    expect(described_class.redeem(token: "#{token}x", city: city)).to be_nil
    expect(described_class.redeem(token: foreign, city: city)).to be_nil
    expect(described_class.redeem(token: [ token ], city: city)).to be_nil
    expect(described_class.redeem(token: "", city: city)).to be_nil
    expect(described_class.redeem(token: token, city: city)).to be_present
  end

  it "refuses a grant consumed by a concurrent request" do
    token = issue
    CityGrant.order(:created_at).last.update!(consumed_at: Time.current)

    expect(described_class.redeem(token: token, city: city)).to be_nil
  end

  it "rejects an unknown kind" do
    expect { issue(kind: "admin") }.to raise_error(ActiveRecord::RecordInvalid)
  end

  it "the platform database refuses an unknown kind even skipping validation" do
    grant = CityGrant.new(city: city, kind: "admin", subject_id: subject_id, expires_at: 1.minute.from_now)

    expect { grant.save!(validate: false) }.to raise_error(ActiveRecord::StatementInvalid, /ck_city_grants_kind/)
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_grants_spec.rb`
Expected: FAIL — `uninitialized constant CityGrants`.

- [ ] **Step 3: Create the platform table**

Crie `db/platform_migrate/20260914000002_create_city_grants.rb`:

```ruby
# Grants assinados de entrada numa cidade (spec banco-por-cidade §5, Plano 3B).
# Emitidos pelo console (operador) e pelo callback do gov.br (usuário); consumidos
# uma única vez pela cidade. Só ids — nenhum dado pessoal (banco de plataforma).
class CreateCityGrants < ActiveRecord::Migration[8.1]
  def change
    create_table :city_grants, id: :uuid do |t|
      t.uuid     :city_id,     null: false
      t.string   :kind,        null: false
      t.uuid     :subject_id,  null: false
      t.datetime :expires_at,  null: false
      t.datetime :consumed_at
      t.timestamps
    end
    add_index :city_grants, :city_id
    add_foreign_key :city_grants, :cities
    add_check_constraint :city_grants, "kind IN ('operator', 'user')", name: "ck_city_grants_kind"
  end
end
```

Aplique a migração (aditiva) nos bancos de plataforma de development e de test, e regenere o dump a partir de
development:

```bash
docker compose exec -T api bin/rails db:migrate:platform && docker compose exec -T -e RAILS_ENV=test api bin/rails db:migrate:platform && docker compose exec -T api bin/rails db:schema:dump:platform
```
Expected: `CreateCityGrants: migrated` duas vezes; sem erro no dump.

Run: `git -C apps/api diff --stat db/platform_schema.rb && git -C apps/api diff db/platform_schema.rb | grep "^[+-]" | grep -v "^+++\|^---"`
Expected: só a versão `2026_09_14_000002`, o bloco `create_table "city_grants"` (com o `check_constraint` e o índice) e
`add_foreign_key "city_grants", "cities"`. Se aparecer qualquer outra diferença, pare e reporte (o dump não deve
tocar as outras tabelas).

- [ ] **Step 4: Write the implementation**

Crie `app/models/city_grant.rb`:

```ruby
# Um grant de entrada numa cidade, no banco de PLATAFORMA. Ver CityGrants.
class CityGrant < PlatformRecord
  KINDS = %w[operator user].freeze

  belongs_to :city

  validates :kind, inclusion: { in: KINDS }
  validates :subject_id, :expires_at, presence: true
end
```

Crie `app/services/city_grants.rb`:

```ruby
# Grant assinado de entrada numa cidade (spec banco-por-cidade §5, Plano 3B).
#
# Quem emite: o console (operador entrando numa cidade) e o callback do gov.br
# em auth.* (usuário voltando para a cidade dele). Quem consome: a cidade, em
# POST /session/grant, que cria a Session local.
#
# Garantias:
#   - validade de TTL (60 s) no token E na linha;
#   - uso único: o consumo é um UPDATE condicional que precisa afetar 1 linha, então
#     duas requisições com o mesmo token não abrem duas sessões;
#   - vale só na cidade emitida: o slug do token é comparado com a cidade do host
#     ANTES de consumir (um grant de A apresentado em B não é gasto);
#   - kind e sujeito saem da LINHA, nunca do token.
#
# A assinatura usa a chave da plataforma (message_verifier, derivada do
# secret_key_base). Chave por cidade é do Plano 6.
module CityGrants
  TTL = 60.seconds
  PURPOSE = :city_grant

  module_function

  def issue(city:, kind:, subject_id:)
    grant = CityGrant.create!(city: city, kind: kind, subject_id: subject_id, expires_at: TTL.from_now)
    verifier.generate({ "jti" => grant.id, "city" => city.slug }, purpose: PURPOSE, expires_in: TTL)
  end

  def redeem(token:, city:)
    return nil unless token.is_a?(String) && token.present?

    payload = verifier.verified(token, purpose: PURPOSE)
    return nil unless payload.is_a?(Hash) && payload["city"] == city.slug && payload["jti"].present?

    now = Time.current
    consumed = CityGrant.where(id: payload["jti"], city_id: city.id, consumed_at: nil)
                        .where("expires_at > ?", now)
                        .update_all(consumed_at: now, updated_at: now)
    return nil unless consumed == 1

    CityGrant.find(payload["jti"])
  end

  def verifier
    Rails.application.message_verifier(PURPOSE)
  end
end
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_grants_spec.rb`
Expected: PASS, 10 examples, 0 failures.

Se `verifier.verified` levantar em vez de devolver nil para um token malformado (ex.: `"#{token}x"`), trate no
próprio `redeem` com `rescue ActiveSupport::MessageVerifier::InvalidSignature, ArgumentError` devolvendo nil, e
registre no relatório qual exceção apareceu — não afrouxe o exemplo.

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 2 pending.

- [ ] **Step 6: Commit**

```bash
git -C apps/api add db/platform_migrate/20260914000002_create_city_grants.rb db/platform_schema.rb app/models/city_grant.rb app/services/city_grants.rb spec/services/city_grants_spec.rb
git -C apps/api commit -m "Add single-use signed grants to enter a city

A grant is a platform row (city, kind, subject, 60-second expiry) plus a
token signed with the platform key that points at it. Redeeming checks
the city before consuming and consumes with a conditional update, so a
grant works once, only in its own city, and never outlives its row.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: Operador entra na cidade

**Files:**
- Create: `app/services/city_dashboard_url.rb`
- Create: `app/controllers/operators/city_grants_controller.rb`
- Modify: `app/controllers/concerns/authentication.rb` (extrai `write_session_cookie`)
- Modify: `app/controllers/sessions_controller.rb` (ação `grant`)
- Modify: `config/routes.rb`
- Modify: `spec/events/platform_event_payload_guard_spec.rb` (nome de evento novo)
- Modify: `spec/requests/admin/api/reports_spec.rb` (skip de operador vira exemplo real)
- Modify: `spec/commands/municipality_channels/rotate_token_spec.rb` (remove o skip de operador)
- Create: `spec/requests/operators/city_grants_spec.rb`, `spec/requests/session_grant_spec.rb`

**Interfaces:**
- Consumes: `CityGrants.issue/redeem`, `CityGrants::TTL` (Task 2); `Session#operator_grant?`,
  `SessionsController#serialize_operator_grant`, `sign_in_operator_grant` (Task 1); `Operators::BaseController`,
  `current_operator`, `PlatformConsoleHost` (Plano 3); `Platform.audit`; `DomainEvents.publish`.
- Produces:
  - `CityDashboardUrl.for(city, grant:) → String` — `format(ENV.fetch("CITY_DASHBOARD_URL_TEMPLATE", "http://%{slug}.localhost:5175/dashboard/"), slug:)` + `?grant=<token>`.
  - `POST /city_grants { city_slug }` no host `admin.*` (operador verificado) → `201 { redirect_url, expires_in: 60 }`;
    cidade inexistente, não servível ou slug que não é texto → `404 {"error":"unknown_city"}`; sem sessão → `401`.
  - `POST /session/grant { token }` no host da cidade → `201` com o JSON da sessão (operador: formato de
    `serialize_operator_grant`; usuário: formato de `serialize`); grant inválido, vencido, de outra cidade, consumido,
    ou sujeito inexistente/desativado → `401 {"error":"invalid_grant"}`.
  - Eventos `operator.city_access`: `PlatformEvent` com payload exatamente `{ "city_id", "operator_id" }` e
    `DomainEvent` na cidade com `{ "operator_id", "session_id" }`.
  - `Authentication#write_session_cookie(session)` — cookie `session_id` assinado, httponly, lax, secure em produção;
    `permanent` só para sessão de usuário.

- [ ] **Step 1: Write the failing tests**

Crie `spec/requests/operators/city_grants_spec.rb`:

```ruby
require "rails_helper"

# Console emite o grant de entrada numa cidade para o operador verificado
# (Plano 3B). A resposta é a URL da cidade com o grant; quem consome é a cidade.
RSpec.describe "Operator city grants on the platform console", type: :request do
  let(:password) { "s3nha-forte-1" }
  let!(:operator) do
    Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end
  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) } # registrada pelo before global de request specs

  def json = JSON.parse(response.body)

  def verified_login!
    post "/session", params: { email_address: operator.email_address, password: password }
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(operator.otp_secret).now }
    expect(response).to have_http_status(:ok)
  end

  before { host! "admin.rotasaude.app" }

  it "issues an operator grant for an active city and answers the city URL carrying it" do
    verified_login!

    expect {
      post "/city_grants", params: { city_slug: city.slug }
    }.to change(CityGrant, :count).by(1)

    expect(response).to have_http_status(:created)
    expect(json["expires_in"]).to eq(60)
    url = URI.parse(json["redirect_url"])
    expect("#{url.scheme}://#{url.host}:#{url.port}#{url.path}").to eq("http://#{city.slug}.localhost:5175/dashboard/")
    token = Rack::Utils.parse_query(url.query).fetch("grant")
    expect(CityGrant.order(:created_at).last).to have_attributes(city_id: city.id, kind: "operator", subject_id: operator.id)
    expect(CityGrants.redeem(token: token, city: city)).to be_present
  end

  it "honours CITY_DASHBOARD_URL_TEMPLATE" do
    verified_login!
    allow(ENV).to receive(:fetch).and_call_original
    allow(ENV).to receive(:fetch).with("CITY_DASHBOARD_URL_TEMPLATE", anything).and_return("https://%{slug}.rotasaude.app/dashboard/")

    post "/city_grants", params: { city_slug: city.slug }

    expect(json["redirect_url"]).to start_with("https://#{city.slug}.rotasaude.app/dashboard/?grant=")
  end

  it "requires a verified operator session" do
    post "/city_grants", params: { city_slug: city.slug }

    expect(response).to have_http_status(:unauthorized)
    expect(CityGrant.count).to eq(0)
  end

  it "answers 404 for an unknown city, a suspended city and a non-string slug, issuing nothing" do
    verified_login!
    suspended = create(:city, status: "suspended")

    [ "naoexiste", suspended.slug ].each do |slug|
      post "/city_grants", params: { city_slug: slug }
      expect(response).to have_http_status(:not_found)
      expect(json).to eq("error" => "unknown_city")
    end
    post "/city_grants", params: { city_slug: [ city.slug ] }
    expect(response).to have_http_status(:not_found)
    expect(CityGrant.count).to eq(0)
  end

  it "is not served on a city host" do
    post "/city_grants", params: { city_slug: city.slug }, headers: { "HOST" => "#{city.slug}.rotasaude.app" }

    expect(response).to have_http_status(:not_found)
  end
end
```

Crie `spec/requests/session_grant_spec.rb`:

```ruby
require "rails_helper"

RSpec::Matchers.define_negated_matcher :not_change, :change unless RSpec::Matchers.method_defined?(:not_change)

# A cidade consome o grant (Plano 3B) e abre a Session local. Operador: sessão só
# leitura, auditada na plataforma E na cidade. Usuário: sessão normal da cidade
# (é o que o callback do gov.br emite, Task 4).
RSpec.describe "POST /session/grant", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  let(:password) { "s3nha-forte-1" }
  let!(:operator) do
    Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end
  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }

  def json = JSON.parse(response.body)
  def set_cookie_header = Array(response.headers["Set-Cookie"]).join("\n")
  def operator_grant(for_city: city) = CityGrants.issue(city: for_city, kind: "operator", subject_id: operator.id)

  it "opens a read-only operator session, audited on the platform and in the city" do
    token = operator_grant

    expect {
      post "/session/grant", params: { token: token }
    }.to change(Session, :count).by(1)
      .and change { PlatformEvent.where(name: "operator.city_access").count }.by(1)
      .and change { DomainEvent.where(name: "operator.city_access").count }.by(1)

    expect(response).to have_http_status(:created)
    expect(json).to include("id" => operator.id, "operator" => true, "memberships" => [])
    session = Session.order(:created_at).last
    expect(session).to have_attributes(operator_id: operator.id, user_id: nil)
    expect(PlatformEvent.where(name: "operator.city_access").last.payload)
      .to eq("city_id" => city.id, "operator_id" => operator.id)
    expect(DomainEvent.where(name: "operator.city_access").last.payload)
      .to include("operator_id" => operator.id, "session_id" => session.id)
    expect(set_cookie_header).to match(/session_id=/)
    expect(set_cookie_header).not_to match(/domain=/i)

    get "/admin/api/reports", params: { period: "30d" }
    expect(response).to have_http_status(:ok)
    post "/setup/invitations", params: { email: "x@x.com", role: "viewer" }
    expect(response).to have_http_status(:forbidden)
    expect(json).to eq("error" => "operator_read_only")
  end

  it "works end to end from the console grant to the city session" do
    host! "admin.rotasaude.app"
    post "/session", params: { email_address: operator.email_address, password: password }
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(operator.otp_secret).now }
    post "/city_grants", params: { city_slug: city.slug }
    token = Rack::Utils.parse_query(URI.parse(json["redirect_url"]).query).fetch("grant")

    host! test_city_host
    post "/session/grant", params: { token: token }

    expect(response).to have_http_status(:created)
    expect(json["operator"]).to be(true)
  end

  it "refuses a grant used a second time" do
    token = operator_grant
    post "/session/grant", params: { token: token }

    expect {
      post "/session/grant", params: { token: token }
    }.not_to change(Session, :count)
    expect(response).to have_http_status(:unauthorized)
    expect(json).to eq("error" => "invalid_grant")
  end

  it "refuses a grant issued for another city without consuming it" do
    other = create(:city, slug: TEST_CITY_B.slug, status: "active", database_url: city_database_url("rota_saude_test_city_b"))
    token = operator_grant(for_city: other)

    post "/session/grant", params: { token: token }

    expect(response).to have_http_status(:unauthorized)
    expect(CityGrant.order(:created_at).last.consumed_at).to be_nil
  end

  it "refuses an expired grant" do
    token = operator_grant

    travel 61.seconds do
      post "/session/grant", params: { token: token }
    end

    expect(response).to have_http_status(:unauthorized)
  end

  it "refuses an operator deactivated after the grant was issued, with no session and no audit" do
    token = operator_grant
    operator.update!(deactivated_at: Time.current)

    expect {
      post "/session/grant", params: { token: token }
    }.to not_change(Session, :count).and not_change(PlatformEvent, :count)
    expect(response).to have_http_status(:unauthorized)
  end

  it "opens no session when the platform audit fails" do
    token = operator_grant
    allow(Platform).to receive(:audit).and_raise(ActiveRecord::StatementInvalid, "boom")

    expect {
      post "/session/grant", params: { token: token }
    }.to raise_error(ActiveRecord::StatementInvalid).and not_change(Session, :count)
  end

  it "opens a normal session for a user grant of this city" do
    user = User.create!(email_address: "u-#{SecureRandom.hex(3)}@x.com", password: "secret123")
    token = CityGrants.issue(city: city, kind: "user", subject_id: user.id)

    post "/session/grant", params: { token: token }

    expect(response).to have_http_status(:created)
    expect(json).to include("id" => user.id, "operator" => false)
    expect(Session.order(:created_at).last).to have_attributes(user_id: user.id, operator_id: nil)
  end

  it "refuses a user grant whose user does not exist in this city" do
    token = CityGrants.issue(city: city, kind: "user", subject_id: SecureRandom.uuid)

    post "/session/grant", params: { token: token }

    expect(response).to have_http_status(:unauthorized)
    expect(json).to eq("error" => "invalid_grant")
  end

  it "refuses a malformed or non-string token" do
    [ "lixo", [ "lixo" ], nil ].each do |token|
      post "/session/grant", params: { token: token }
      expect(response).to have_http_status(:unauthorized)
    end
  end
end
```

(`not_change` é o matcher negado declarado no topo deste arquivo: não existe nenhum em `spec/support/`.)

Em `spec/requests/admin/api/reports_spec.rb`, troque o exemplo `"an operator sees a city's reports as metadata,
without token or payload"` (o `skip "Plano 3B: grant de operador ..."`) por:

```ruby
  it "an operator who entered the city by grant sees its reports as metadata, without token or payload" do
    seed_report(tier: "alta", token: "TOK-OPERATOR-456")
    operator = Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                                otp_secret: ROTP::Base32.random, otp_enabled: true)
    sign_in_operator_grant(operator)

    get "/admin/api/reports", params: { period: "30d" }

    expect(response).to have_http_status(:ok)
    expect(JSON.parse(response.body).dig("data", "reports").size).to eq(1)
    expect(response.body).not_to include("TOK-OPERATOR-456")
    expect(response.body).not_to include("NEVER-EXPOSE")
  end
```

Em `spec/commands/municipality_channels/rotate_token_spec.rb`, apague o exemplo
`"the platform operator grant (cross-city custody) is Plan 3B"` e o comentário da linha 9 que o anuncia. Justificativa
(registre no relatório): por decisão do usuário, operador dentro da cidade é SÓ LEITURA — não haverá custódia de token
por operador. `RotateToken` não tem rota HTTP, e toda escrita de uma sessão de operador já é recusada por
`spec/requests/operator_city_session_spec.rb` ("is refused on every action that is not explicitly read-only").

Em `spec/events/platform_event_payload_guard_spec.rb`, acrescente `operator.city_access` ao fim de
`R18_PLATFORM_EVENT_NAMES`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/operators/city_grants_spec.rb spec/requests/session_grant_spec.rb spec/requests/admin/api/reports_spec.rb`
Expected: FAIL — rotas `/city_grants` e `/session/grant` inexistentes (404). O exemplo novo de `reports_spec.rb` já
passa (Task 1 liberou a leitura).

- [ ] **Step 3: Write the implementation**

Crie `app/services/city_dashboard_url.rb`:

```ruby
# URL do dashboard de uma cidade com um grant, para onde o console e o callback
# do gov.br mandam o navegador (Plano 3B). O host por cidade vem de um template
# porque os frontends ainda não têm host por cidade (Plano 6).
module CityDashboardUrl
  DEFAULT_TEMPLATE = "http://%{slug}.localhost:5175/dashboard/"

  module_function

  def for(city, grant:)
    base = format(ENV.fetch("CITY_DASHBOARD_URL_TEMPLATE", DEFAULT_TEMPLATE), slug: city.slug)
    "#{base}?#{{ grant: grant }.to_query}"
  end
end
```

Crie `app/controllers/operators/city_grants_controller.rb`:

```ruby
# Operador verificado pede para entrar numa cidade (spec §5, Plano 3B):
#
#   POST /city_grants { city_slug } → 201 { redirect_url, expires_in }
#
# Só emite o grant; a entrada (Session na cidade) e a auditoria nos dois lados
# acontecem quando a cidade o consome, em POST /session/grant.
module Operators
  class CityGrantsController < BaseController
    def create
      slug = params[:city_slug]
      city = slug.is_a?(String) ? City.find_by(slug: slug) : nil
      return render(json: { error: "unknown_city" }, status: :not_found) unless city&.servable?

      token = CityGrants.issue(city: city, kind: "operator", subject_id: current_operator.id)
      render json: { redirect_url: CityDashboardUrl.for(city, grant: token), expires_in: CityGrants::TTL.to_i },
             status: :created
    end
  end
end
```

`app/controllers/concerns/authentication.rb` — troque `start_new_session_for` por:

```ruby
  def start_new_session_for(user)
    user.sessions.create!(
      user_agent: request.user_agent,
      ip_address: request.remote_ip
    ).tap do |session|
      Current.session = session
      write_session_cookie(session)
    end
  end

  # Host-only: NUNCA `domain:` (spec §5). Sessão de operador (grant) é de sessão
  # do navegador; a de usuário é permanent, como sempre foi.
  def write_session_cookie(session)
    jar = session.operator_grant? ? cookies.signed : cookies.signed.permanent
    jar[:session_id] = {
      value: session.id,
      httponly: true,
      same_site: :lax,
      secure: Rails.env.production?
    }
  end
```

`app/controllers/sessions_controller.rb`:
- `allow_unauthenticated_access only: %i[create govbr_callback grant]`
- `rate_limit ... only: %i[create govbr_callback grant]`
- acrescente ao cabeçalho de comentário a linha
  `#   POST   /session/grant   { token }                    → 201 (entrada por grant assinado, Plano 3B)`
- a ação pública, depois de `create`:

```ruby
  # POST /session/grant { token } — entrada por grant assinado (spec §5, Plano 3B).
  # O grant de uma cidade não vale em outra, vale uma vez e por 60 s (CityGrants).
  def grant
    grant = CityGrants.redeem(token: params[:token], city: Current.city)
    return render_invalid_grant unless grant

    grant.kind == "operator" ? open_operator_grant_session(grant) : open_user_grant_session(grant)
  end
```

- e, em `private`:

```ruby
  def render_invalid_grant
    render json: { error: "invalid_grant" }, status: :unauthorized
  end

  # Auditoria primeiro na PLATAFORMA, depois Session + evento na CIDADE (transação
  # da cidade). Sem transação comum aos dois bancos, a falha que sobra é o registro
  # de uma tentativa sem sessão — nunca uma sessão sem auditoria.
  def open_operator_grant_session(grant)
    operator = Operator.find_by(id: grant.subject_id)
    return render_invalid_grant unless operator&.active?

    Platform.audit("operator.city_access", city_id: Current.city.id, operator_id: operator.id)
    session = ApplicationRecord.transaction do
      Session.create!(operator_id: operator.id, user_agent: request.user_agent, ip_address: request.remote_ip).tap do |s|
        DomainEvents.publish("operator.city_access", operator_id: operator.id, session_id: s.id)
      end
    end
    Current.session = session
    write_session_cookie(session)
    render json: serialize_operator_grant(session), status: :created
  end

  def open_user_grant_session(grant)
    user = User.find_by(id: grant.subject_id)
    return render_invalid_grant unless user&.active?

    start_new_session_for(user)
    render json: serialize(user), status: :created
  end
```

`config/routes.rb`:
- no bloco `constraints(PlatformConsoleHost)`, dentro do `scope module: :operators, as: :operator`, acrescente
  `resources :city_grants, only: :create`;
- logo abaixo de `resource :session, only: %i[create show destroy]` (o da cidade), acrescente:

```ruby
  # Entrada na cidade por grant assinado (Plano 3B): operador vindo do console ou
  # usuário vindo do callback do gov.br.
  post "/session/grant", to: "sessions#grant"
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/operators spec/requests/session_grant_spec.rb spec/requests/operator_city_session_spec.rb spec/requests/admin/api spec/events/platform_event_payload_guard_spec.rb spec/commands/municipality_channels/rotate_token_spec.rb spec/architecture`
Expected: PASS, 0 failures.

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, **0 pending** (os dois skips "Plano 3B" viraram, respectivamente, exemplo real e remoção
justificada).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/services/city_dashboard_url.rb app/controllers/operators/city_grants_controller.rb app/controllers/concerns/authentication.rb app/controllers/sessions_controller.rb config/routes.rb spec/requests/operators/city_grants_spec.rb spec/requests/session_grant_spec.rb spec/requests/admin/api/reports_spec.rb spec/commands/municipality_channels/rotate_token_spec.rb spec/events/platform_event_payload_guard_spec.rb
git -C apps/api commit -m "Let an operator enter a city through a signed grant

The console issues a grant for an active city and answers the city URL
carrying it; the city redeems it at POST /session/grant and opens a
read-only operator session, audited first on the platform and then in
the city. The same endpoint opens a normal session for a user grant.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: Callback único do gov.br em `auth.*`

**Files:**
- Modify: `app/models/city_catalog.rb` (`auth_host?`), `spec/models/city_catalog_spec.rb`
- Create: `app/constraints/platform_auth_host.rb`
- Create: `app/controllers/govbr/callbacks_controller.rb`
- Modify: `app/auth/authenticator/gov_br.rb`, `app/auth/authenticator.rb`
- Modify: `app/controllers/sessions_controller.rb` (sai `govbr_callback`, entra `govbr_start`)
- Modify: `config/routes.rb`
- Modify: `spec/auth/gov_br_spec.rb`
- Delete: `spec/controllers/sessions_controller_govbr_spec.rb`
- Create: `spec/requests/govbr_login_spec.rb`

**Interfaces:**
- Consumes: `CityGrants.issue(city:, kind: "user", subject_id:)`, `CityDashboardUrl.for(city, grant:)` (Tasks 2-3);
  `POST /session/grant` (Task 3) consome o grant de usuário; `CityConnection.with`; `Current.set(city:)`.
- Produces:
  - `CityCatalog.auth_host?(host) → Boolean` (primeiro rótulo `auth`); `PlatformAuthHost.matches?(request)`.
  - `Authenticator::GovBr::STATE_TTL = 10.minutes`, `STATE_PURPOSE = :govbr_state`, `AUTHORIZE_ENDPOINT_PATH = "/authorize"`.
  - `Authenticator::GovBr.start(city:) → String` — URL de autorização com `response_type=code`, `client_id`,
    `scope=openid email profile`, `redirect_uri`, `state` (assinado: `{ "city", "nonce" }`) e `nonce`.
  - `Authenticator::GovBr.verify_state(state) → Hash | nil`.
  - `Authenticator::GovBr.exchange_code_for_claims(code) → Hash` (levanta `IntegrationError` se `code` vazio).
  - `Authenticator::GovBr.provision_from_claims(claims) → User | nil` — roda NA conexão da cidade corrente; publica
    `identity.govbr_login`.
  - `POST /auth/govbr/start` no host da cidade → `200 { authorize_url }` (ou `502 govbr_integration_error` sem config).
  - `GET /auth/govbr/callback?code&state` SÓ no host `auth.*` → `302` para `CityDashboardUrl.for(city, grant:)`; state
    inválido/vencido → `400 invalid_state`; cidade inexistente/não servível → `404 unknown_city`; nonce diferente →
    `401 invalid_nonce`; usuário nil/desativado → `401 govbr_unauthenticated`; falha de integração → `502`.
  - Removido: `Authenticator.govbr`, `Authenticator::GovBr.authenticate`, `SessionsController#govbr_callback` e a rota
    `GET /auth/govbr/callback` do host da cidade.

Por que o callback é único (spec §5): o OIDC do gov.br valida `redirect_uri` contra o registrado, e registrar uma URI
por cidade não escala com provisionamento. `auth.*` resolve a cidade pelo `state` e devolve o navegador à cidade com o
MESMO grant do operador — a cidade abre a sessão do usuário em `POST /session/grant`.

Justificativa da remoção de `spec/controllers/sessions_controller_govbr_spec.rb` (callback provisório no host da
cidade, eliminado pela spec §5): "retorna 201 + user serializado" → `govbr_login_spec` "redirects back to the city with
a grant that opens the user's session"; "retorna 401" → "refuses a deactivated user without issuing a grant";
"retorna 502" → "answers 502 on an integration failure"; "usuário da cidade com MFA cadastrado ... sem mfa_required" →
o mesmo exemplo de ida e volta, que confere o JSON da sessão aberta pelo grant.

- [ ] **Step 1: Write the failing tests**

Em `spec/models/city_catalog_spec.rb`, dentro do `RSpec.describe CityCatalog`, acrescente:

```ruby
  describe ".auth_host?" do
    it "is true only when the first label is auth" do
      expect(described_class.auth_host?("auth.rotasaude.app")).to be(true)
      expect(described_class.auth_host?("AUTH.localhost:3030")).to be(true)
      expect(described_class.auth_host?("admin.rotasaude.app")).to be(false)
      expect(described_class.auth_host?("curitiba.rotasaude.app")).to be(false)
      expect(described_class.auth_host?(nil)).to be(false)
    end
  end
```

Substitua `spec/auth/gov_br_spec.rb` inteiro:

```ruby
require "rails_helper"

# Seam do gov.br (ADR-0011), depois do callback único em auth.* (Plano 3B):
# state assinado com cidade + nonce, troca do code, e provisionamento do usuário
# NA cidade corrente. fetch_token e decode_id_token reais usam HTTP e JWT; aqui a
# troca é mockada.
RSpec.describe Authenticator::GovBr do
  include ActiveSupport::Testing::TimeHelpers

  let(:city) { City.new(slug: TEST_CITY_A.slug) }

  before do
    Current.reset
    Current.city = TEST_CITY_A # identity.govbr_login é evento DA CIDADE (Ruling R18)
    allow(described_class).to receive_messages(
      client_id: "rota-client", redirect_uri: "https://auth.rotasaude.app/auth/govbr/callback",
      issuer_url: "https://sso.staging.acesso.gov.br"
    )
  end

  after { Current.reset }

  describe ".start and .verify_state" do
    def query_of(url) = Rack::Utils.parse_query(URI.parse(url).query)

    it "builds the authorize URL with a signed state carrying the city and the same nonce" do
      url = described_class.start(city: city)

      expect(url).to start_with("https://sso.staging.acesso.gov.br/authorize?")
      query = query_of(url)
      expect(query).to include("response_type" => "code", "client_id" => "rota-client", "scope" => "openid email profile",
                               "redirect_uri" => "https://auth.rotasaude.app/auth/govbr/callback")
      state = described_class.verify_state(query.fetch("state"))
      expect(state).to eq("city" => TEST_CITY_A.slug, "nonce" => query.fetch("nonce"))
    end

    it "uses a fresh nonce on every start" do
      first = query_of(described_class.start(city: city)).fetch("nonce")
      second = query_of(described_class.start(city: city)).fetch("nonce")

      expect(first).not_to eq(second)
    end

    it "refuses a tampered, expired, blank or non-string state" do
      state = query_of(described_class.start(city: city)).fetch("state")

      expect(described_class.verify_state("#{state}x")).to be_nil
      expect(described_class.verify_state("")).to be_nil
      expect(described_class.verify_state([ state ])).to be_nil
      travel 11.minutes do
        expect(described_class.verify_state(state)).to be_nil
      end
    end
  end

  describe ".exchange_code_for_claims" do
    it "code vazio levanta IntegrationError" do
      expect { described_class.exchange_code_for_claims("") }.to raise_error(Authenticator::GovBr::IntegrationError, /vazio/)
    end
  end

  describe ".provision_from_claims" do
    let(:claims) { { "sub" => "12345678900", "email" => "fulano@gov.br", "name" => "Fulano de Tal", "amr" => [ "ouro" ] } }

    it "cria User + Identity quando nenhum existe" do
      user = described_class.provision_from_claims(claims)

      expect(user).to be_a(User)
      expect(user.email_address).to eq("fulano@gov.br")
      expect(Identity.where(provider: "govbr", provider_uid: "12345678900").count).to eq(1)
    end

    it "reusa User existente quando o email bate (seam: 2 identidades para mesmo user)" do
      existing = User.create!(email_address: "fulano@gov.br", password: "secret123")

      user = described_class.provision_from_claims(claims)

      expect(user.id).to eq(existing.id)
      expect(Identity.where(user: existing, provider: "govbr").count).to eq(1)
    end

    it "reusa User+Identity quando provider_uid já existe (segundo login)" do
      first  = described_class.provision_from_claims(claims)
      second = described_class.provision_from_claims(claims)

      expect(first.id).to eq(second.id)
      expect(Identity.where(provider: "govbr", provider_uid: "12345678900").count).to eq(1)
    end

    it "grava um DomainEvent identity.govbr_login com assurance level, na cidade (Ruling R18), nunca na plataforma" do
      user = nil
      expect { user = described_class.provision_from_claims(claims) }.not_to change(PlatformEvent, :count)

      event = DomainEvent.find_by!(name: "identity.govbr_login")
      expect(event.payload).to include("user_id" => user.id, "provider_uid" => "12345678900", "assurance" => "ouro")
    end

    it "user desativado retorna nil" do
      described_class.provision_from_claims(claims)
      User.find_by(email_address: "fulano@gov.br").update!(deactivated_at: Time.current)

      expect(described_class.provision_from_claims(claims)).to be_nil
    end
  end

  describe ".assurance_meets?" do
    it "bronze cobre só viewer" do
      expect(described_class.assurance_meets?(assurance: "bronze", role: "viewer")).to be true
      expect(described_class.assurance_meets?(assurance: "bronze", role: "municipal_admin")).to be false
      expect(described_class.assurance_meets?(assurance: "bronze", role: "platform_operator")).to be false
    end

    it "prata cobre viewer e municipal_admin (mas não publisher/operator)" do
      expect(described_class.assurance_meets?(assurance: "prata", role: "viewer")).to be true
      expect(described_class.assurance_meets?(assurance: "prata", role: "municipal_admin")).to be true
      expect(described_class.assurance_meets?(assurance: "prata", role: "protocol_publisher")).to be false
    end

    it "ouro cobre todos" do
      expect(described_class.assurance_meets?(assurance: "ouro", role: "viewer")).to be true
      expect(described_class.assurance_meets?(assurance: "ouro", role: "protocol_publisher")).to be true
      expect(described_class.assurance_meets?(assurance: "ouro", role: "platform_operator")).to be true
    end

    it "assurance ou role nil/inválido devolve false" do
      expect(described_class.assurance_meets?(assurance: nil, role: "viewer")).to be false
      expect(described_class.assurance_meets?(assurance: "diamante", role: "viewer")).to be false
      expect(described_class.assurance_meets?(assurance: "ouro", role: nil)).to be false
    end
  end
end
```

Crie `spec/requests/govbr_login_spec.rb`:

```ruby
require "rails_helper"

# Login gov.br com callback único em auth.* (spec §5, Plano 3B):
#   cidade  POST /auth/govbr/start      → authorize_url (state assinado: cidade + nonce)
#   auth.*  GET  /auth/govbr/callback   → provisiona NA cidade do state, emite grant de usuário, 302 para a cidade
#   cidade  POST /session/grant         → sessão do usuário (Task 3)
RSpec.describe "gov.br login through the single auth callback", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  let(:city_a) { City.find_by!(slug: TEST_CITY_A.slug) }
  let(:claims_base) { { "sub" => "12345678900", "email" => "fulano@gov.br", "amr" => [ "prata" ] } }

  def json = JSON.parse(response.body)
  def query_of(url) = Rack::Utils.parse_query(URI.parse(url).query)

  before do
    allow(Authenticator::GovBr).to receive_messages(
      client_id: "rota-client", redirect_uri: "https://auth.rotasaude.app/auth/govbr/callback",
      issuer_url: "https://sso.staging.acesso.gov.br"
    )
  end

  # Começa no host da cidade e devolve [state, nonce].
  def start_on(host)
    post "/auth/govbr/start", headers: { "HOST" => host }
    expect(response).to have_http_status(:ok)
    query = query_of(json.fetch("authorize_url"))
    [ query.fetch("state"), query.fetch("nonce") ]
  end

  def callback(state:, code: "valid-code")
    get "/auth/govbr/callback", params: { code: code, state: state }, headers: { "HOST" => "auth.rotasaude.app" }
  end

  def stub_exchange(nonce)
    allow(Authenticator::GovBr).to receive(:exchange_code_for_claims).with("valid-code").and_return(claims_base.merge("nonce" => nonce))
  end

  it "redirects back to the city with a grant that opens the user's session" do
    state, nonce = start_on(test_city_host)
    stub_exchange(nonce)

    callback(state: state)

    expect(response).to have_http_status(:found)
    expect(response.location).to start_with("http://#{TEST_CITY_A.slug}.localhost:5175/dashboard/?grant=")
    user = User.find_by!(email_address: "fulano@gov.br")
    expect(DomainEvent.where(name: "identity.govbr_login").last.payload).to include("user_id" => user.id)

    post "/session/grant", params: { token: query_of(response.location).fetch("grant") }, headers: { "HOST" => test_city_host }
    expect(response).to have_http_status(:created)
    expect(json).to include("id" => user.id, "operator" => false)
    expect(json).not_to have_key("mfa_required")
  end

  it "provisions the user only in the city named by the state" do
    city_b = create(:city, slug: TEST_CITY_B.slug, status: "active", database_url: city_database_url("rota_saude_test_city_b"))
    state, nonce = start_on("#{TEST_CITY_B.slug}.rotasaude.app")
    stub_exchange(nonce)

    callback(state: state)

    expect(response).to have_http_status(:found)
    expect(response.location).to start_with("http://#{TEST_CITY_B.slug}.localhost:5175/dashboard/?grant=")
    expect(CityConnection.with(city_b) { User.where(email_address: "fulano@gov.br").count }).to eq(1)
    expect(User.where(email_address: "fulano@gov.br").count).to eq(0) # cidade A (conexão padrão)
    expect(CityGrant.order(:created_at).last).to have_attributes(city_id: city_b.id, kind: "user")
  end

  it "refuses a nonce that is not the one in the state, creating nothing" do
    state, _nonce = start_on(test_city_host)
    stub_exchange("outro-nonce")

    expect { callback(state: state) }.not_to change(CityGrant, :count)

    expect(response).to have_http_status(:unauthorized)
    expect(json).to eq("error" => "invalid_nonce")
    expect(User.where(email_address: "fulano@gov.br")).to be_empty
  end

  it "refuses a tampered or expired state without calling gov.br" do
    state, _nonce = start_on(test_city_host)
    expect(Authenticator::GovBr).not_to receive(:exchange_code_for_claims)

    callback(state: "#{state}x")
    expect(response).to have_http_status(:bad_request)
    expect(json).to eq("error" => "invalid_state")

    travel 11.minutes do
      callback(state: state)
    end
    expect(response).to have_http_status(:bad_request)
  end

  it "answers 404 when the state names a city that is not servable" do
    state, _nonce = start_on(test_city_host)
    city_a.update!(status: "suspended")
    CityCatalog.reset_cache!

    callback(state: state)

    expect(response).to have_http_status(:not_found)
    expect(json).to eq("error" => "unknown_city")
  end

  it "refuses a deactivated user without issuing a grant" do
    User.create!(email_address: "fulano@gov.br", password: "secret123", deactivated_at: Time.current)
    state, nonce = start_on(test_city_host)
    stub_exchange(nonce)

    expect { callback(state: state) }.not_to change(CityGrant, :count)

    expect(response).to have_http_status(:unauthorized)
    expect(json).to eq("error" => "govbr_unauthenticated")
  end

  it "answers 502 on an integration failure, issuing nothing" do
    state, _nonce = start_on(test_city_host)
    allow(Authenticator::GovBr).to receive(:exchange_code_for_claims).and_raise(Authenticator::GovBr::IntegrationError, "boom")

    expect { callback(state: state) }.not_to change(CityGrant, :count)

    expect(response).to have_http_status(:bad_gateway)
    expect(json).to eq("error" => "govbr_integration_error")
  end

  it "is not served on a city host nor on the console host" do
    state, _nonce = start_on(test_city_host)

    [ test_city_host, "admin.rotasaude.app" ].each do |host|
      get "/auth/govbr/callback", params: { code: "valid-code", state: state }, headers: { "HOST" => host }
      expect(response).to have_http_status(:not_found), "#{host} respondeu #{response.status}"
    end
  end

  it "start answers 502 when gov.br is not configured" do
    allow(Authenticator::GovBr).to receive(:client_id).and_raise(Authenticator::GovBr::IntegrationError, "missing GOVBR_CLIENT_ID")

    post "/auth/govbr/start"

    expect(response).to have_http_status(:bad_gateway)
    expect(json).to eq("error" => "govbr_integration_error")
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_catalog_spec.rb spec/auth/gov_br_spec.rb spec/requests/govbr_login_spec.rb`
Expected: FAIL — `undefined method 'auth_host?'`, `undefined method 'start'`/`'provision_from_claims'`, e 404 em
`/auth/govbr/start`.

- [ ] **Step 3: Write the implementation**

`app/models/city_catalog.rb` — depois de `console_host?`:

```ruby
    # Host do callback único do gov.br (auth.*). Reservado: nunca resolve cidade.
    def auth_host?(host)
      label_for(host) == "auth"
    end
```

Crie `app/constraints/platform_auth_host.rb`:

```ruby
# Constraint de rota: casa requisições dirigidas a auth.*, onde mora o callback
# único do gov.br (spec banco-por-cidade §5, Plano 3B).
class PlatformAuthHost
  def self.matches?(request)
    CityCatalog.auth_host?(request.host)
  end
end
```

`app/auth/authenticator/gov_br.rb`:
- troque os passos 2 e 3 do comentário de cabeçalho por:

```ruby
#   2. A redirect_uri registrada no gov.br é o callback ÚNICO em auth.*
#      (GET /auth/govbr/callback, Govbr::CallbacksController — Plano 3B).
#   3. O login começa no host da cidade (POST /auth/govbr/start): o `state` é
#      assinado com a cidade e um nonce; o callback exige o mesmo nonce no id_token.
```

- acrescente, depois de `JWKS_ENDPOINT_PATH`:

```ruby
    AUTHORIZE_ENDPOINT_PATH = "/authorize".freeze
    STATE_PURPOSE = :govbr_state
    STATE_TTL = 10.minutes
```

- troque `def self.authenticate(code:) ... end` por:

```ruby
    # URL de autorização do gov.br para a cidade `city`. O state (assinado, 10 min)
    # carrega a cidade e o nonce; o mesmo nonce vai para o gov.br e volta no id_token.
    def self.start(city:)
      nonce = SecureRandom.hex(16)
      state = state_verifier.generate({ "city" => city.slug, "nonce" => nonce },
                                      purpose: STATE_PURPOSE, expires_in: STATE_TTL)
      uri = URI.join(issuer_url, AUTHORIZE_ENDPOINT_PATH)
      uri.query = {
        response_type: "code", client_id: client_id, scope: "openid email profile",
        redirect_uri: redirect_uri, state: state, nonce: nonce
      }.to_query
      uri.to_s
    end

    def self.verify_state(state)
      return nil unless state.is_a?(String) && state.present?

      payload = state_verifier.verified(state, purpose: STATE_PURPOSE)
      return nil unless payload.is_a?(Hash) && payload["city"].is_a?(String) && payload["nonce"].is_a?(String)

      payload
    end

    # Roda NA conexão da cidade corrente (Govbr::CallbacksController abre
    # CityConnection.with e Current.set(city:)). Devolve nil para usuário desativado.
    def self.provision_from_claims(claims)
      uid       = claims.fetch("sub")
      assurance = claims["amr"]&.first || claims["nivel_confianca"]

      user = find_or_provision_user(uid: uid, email: claims["email"], name: claims["name"])
      return nil unless user&.active?

      annotate_identity_assurance(user, uid, assurance) if assurance.present?
      user
    end
```

- em `exchange_code_for_claims`, acrescente como primeira linha
  `raise IntegrationError, "code vazio" if code.blank?`;
- troque o comentário acima de `find_or_provision_user` por
  `# Roda na conexão da cidade corrente (ver provision_from_claims).`;
- em `# — Configuration —`, acrescente:

```ruby
    def self.state_verifier
      Rails.application.message_verifier(STATE_PURPOSE)
    end
```

`app/auth/authenticator.rb` — apague o método `self.govbr(code:)` e a linha `#   - govbr(code:) — ...` do cabeçalho;
acrescente ao cabeçalho: `# gov.br: Authenticator::GovBr, chamado pelo callback único em auth.* (Plano 3B).`

Crie `app/controllers/govbr/callbacks_controller.rb`:

```ruby
# Callback ÚNICO do gov.br, no host auth.* (spec banco-por-cidade §5, Plano 3B).
#
#   GET /auth/govbr/callback?code=…&state=…
#     → state assinado aponta a cidade → troca o code → confere o nonce →
#       provisiona o usuário NA cidade → grant de usuário → 302 para a cidade.
#
# NÃO herda de ApplicationController: auth.* é reservado e não resolve cidade pelo
# host — a cidade vem do state.
module Govbr
  class CallbacksController < ActionController::API
    before_action :require_auth_host

    rate_limit to: 20, within: 1.minute,
               with: -> { render json: { error: "too_many_requests" }, status: :too_many_requests }

    def show
      state = Authenticator::GovBr.verify_state(params[:state])
      return render(json: { error: "invalid_state" }, status: :bad_request) unless state

      city = City.find_by(slug: state["city"])
      return render(json: { error: "unknown_city" }, status: :not_found) unless city&.servable?

      claims = Authenticator::GovBr.exchange_code_for_claims(params[:code].to_s)
      return render(json: { error: "invalid_nonce" }, status: :unauthorized) unless nonce_matches?(claims, state)

      user = nil
      Current.set(city: city) do
        CityConnection.with(city) { user = Authenticator::GovBr.provision_from_claims(claims) }
      end
      return render(json: { error: "govbr_unauthenticated" }, status: :unauthorized) unless user

      token = CityGrants.issue(city: city, kind: "user", subject_id: user.id)
      redirect_to CityDashboardUrl.for(city, grant: token), allow_other_host: true, status: :found
    rescue Authenticator::GovBr::IntegrationError => e
      Rails.logger.error("[govbr_callback] #{e.class}: #{e.message}")
      render json: { error: "govbr_integration_error" }, status: :bad_gateway
    end

    private

    def require_auth_host
      head :not_found unless CityCatalog.auth_host?(request.host)
    end

    def nonce_matches?(claims, state)
      nonce = claims["nonce"]
      nonce.is_a?(String) && ActiveSupport::SecurityUtils.secure_compare(nonce, state["nonce"])
    end
  end
end
```

`app/controllers/sessions_controller.rb`:
- `allow_unauthenticated_access only: %i[create govbr_start grant]` e o mesmo trio no `rate_limit`;
- apague a ação `govbr_callback` inteira (com o comentário e o `rescue`);
- acrescente, depois de `grant`:

```ruby
  # POST /auth/govbr/start — começa o login gov.br DESTA cidade (Plano 3B). O
  # callback é único, em auth.* (Govbr::CallbacksController), e volta para cá com
  # um grant de usuário.
  def govbr_start
    render json: { authorize_url: Authenticator::GovBr.start(city: Current.city) }
  rescue Authenticator::GovBr::IntegrationError => e
    Rails.logger.error("[govbr_start] #{e.class}: #{e.message}")
    render json: { error: "govbr_integration_error" }, status: :bad_gateway
  end
```

- no cabeçalho de comentário, troque qualquer menção a `govbr_callback` por
  `#   POST   /auth/govbr/start                           → 200 { authorize_url }`.

`config/routes.rb`:
- logo DEPOIS do bloco `constraints(PlatformConsoleHost) ... end`, acrescente:

```ruby
  # Callback único do gov.br (auth.*, Plano 3B). A cidade vem do state assinado,
  # não do host. Antes das rotas de cidade: a primeira rota que casa vence.
  constraints(PlatformAuthHost) do
    get "/auth/govbr/callback", to: "govbr/callbacks#show"
  end
```

- troque o bloco da cidade

```ruby
  # gov.br OIDC callback (ADR-0011). Frontend redireciona para gov.br;
  # gov.br retorna com ?code=... → trocamos e iniciamos sessão.
  get  "/auth/govbr/callback", to: "sessions#govbr_callback"
```

por:

```ruby
  # gov.br (ADR-0011): o login começa na cidade; o callback é o de auth.*, acima.
  post "/auth/govbr/start", to: "sessions#govbr_start"
```

Apague o spec do callback provisório:

```bash
git -C apps/api rm spec/controllers/sessions_controller_govbr_spec.rb
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_catalog_spec.rb spec/auth spec/requests/govbr_login_spec.rb spec/requests/session_grant_spec.rb spec/requests/city_session_isolation_spec.rb spec/requests/operators`
Expected: PASS, 0 failures.

Run: `grep -rn "govbr_callback\|Authenticator.govbr\|GovBr.authenticate" apps/api/app apps/api/config apps/api/spec`
Expected: vazio.

Run: `docker compose exec -T api bin/rails zeitwerk:check`
Expected: `All is good!` (a pasta nova `app/controllers/govbr` vira o namespace `Govbr`).

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/models/city_catalog.rb spec/models/city_catalog_spec.rb app/constraints/platform_auth_host.rb app/controllers/govbr/callbacks_controller.rb app/auth/authenticator/gov_br.rb app/auth/authenticator.rb app/controllers/sessions_controller.rb config/routes.rb spec/auth/gov_br_spec.rb spec/requests/govbr_login_spec.rb
git -C apps/api commit -m "Move the gov.br callback to a single auth host that hands off through a grant

The login starts on the city host with a signed state carrying the city
and a nonce. The only callback lives on auth.*: it checks the state and
the nonce, provisions the user inside that city, and redirects back with
a user grant that the city redeems at POST /session/grant.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 5: Teto de tentativas de TOTP no login do operador

**Files:**
- Create: `db/platform_migrate/20260914000003_add_mfa_failed_attempts_to_operator_sessions.rb`
- Modify: `db/platform_schema.rb` (regenerado)
- Modify: `app/controllers/concerns/operator_authentication.rb` (constante)
- Modify: `app/controllers/operators/sessions_controller.rb`
- Modify: `spec/requests/operators/sessions_spec.rb`

**Interfaces:**
- Consumes: `Operators::SessionsController#challenge_totp`, `pending_session`, `OperatorAuthentication::COOKIE` (Plano 3).
- Produces:
  - `operator_sessions.mfa_failed_attempts integer NOT NULL DEFAULT 0` (plataforma).
  - `OperatorAuthentication::MAX_TOTP_ATTEMPTS = 5`.
  - Challenge com código errado: incrementa o contador de forma atômica e responde `401 {"error":"invalid_code"}`;
    no 5º erro da MESMA sessão pendente, apaga a sessão, apaga o cookie e responde
    `401 {"error":"too_many_attempts"}`. Um novo passo de senha cria sessão nova, com contador zerado.

Por que por sessão e não por operador: o rate limit atual é por IP (10 a cada 3 min). Por sessão pendente, quem troca
de IP ainda precisa refazer o passo da senha a cada 5 tentativas — o que o rate limit por IP e a senha já freiam — sem
dar a um terceiro o poder de travar a conta do operador errando o TOTP dele.

- [ ] **Step 1: Write the failing tests**

Em `spec/requests/operators/sessions_spec.rb`, dentro do `RSpec.describe` principal (fora dos `describe` aninhados),
acrescente:

```ruby
  describe "TOTP attempt limit (Plano 3B)" do
    def wrong_challenge(session_id)
      post "/session/challenge", params: { session_id: session_id, code: "nao-e-um-codigo" }
    end

    it "counts wrong codes on the pending session and still accepts the right one before the limit" do
      login!
      session_id = json["session_id"]

      4.times do
        wrong_challenge(session_id)
        expect(response).to have_http_status(:unauthorized)
        expect(json).to eq("error" => "invalid_code")
      end
      expect(OperatorSession.find(session_id).mfa_failed_attempts).to eq(4)

      post "/session/challenge", params: { session_id: session_id, code: totp }
      expect(response).to have_http_status(:ok)
    end

    it "destroys the pending session on the fifth wrong code, so the right code no longer works" do
      login!
      session_id = json["session_id"]

      4.times { wrong_challenge(session_id) }
      wrong_challenge(session_id)

      expect(response).to have_http_status(:unauthorized)
      expect(json).to eq("error" => "too_many_attempts")
      expect(OperatorSession.exists?(session_id)).to be(false)

      post "/session/challenge", params: { session_id: session_id, code: totp }
      expect(response).to have_http_status(:unauthorized)
      expect(json).to eq("error" => "invalid_session")
    end

    it "a new password step starts a fresh count" do
      login!
      first_session_id = json["session_id"]
      5.times { wrong_challenge(first_session_id) }
      expect(OperatorSession.exists?(first_session_id)).to be(false)

      login!
      expect(OperatorSession.find(json["session_id"]).mfa_failed_attempts).to eq(0)
      post "/session/challenge", params: { session_id: json["session_id"], code: totp }
      expect(response).to have_http_status(:ok)
    end
  end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/operators/sessions_spec.rb`
Expected: FAIL — `undefined method 'mfa_failed_attempts'` e o 5º erro responde `invalid_code`.

- [ ] **Step 3: Add the column**

Crie `db/platform_migrate/20260914000003_add_mfa_failed_attempts_to_operator_sessions.rb`:

```ruby
# Teto de tentativas de TOTP por sessão pendente de operador (Plano 3B).
class AddMfaFailedAttemptsToOperatorSessions < ActiveRecord::Migration[8.1]
  def change
    add_column :operator_sessions, :mfa_failed_attempts, :integer, null: false, default: 0
  end
end
```

```bash
docker compose exec -T api bin/rails db:migrate:platform && docker compose exec -T -e RAILS_ENV=test api bin/rails db:migrate:platform && docker compose exec -T api bin/rails db:schema:dump:platform
```
Expected: `AddMfaFailedAttemptsToOperatorSessions: migrated` duas vezes.

Run: `git -C apps/api diff db/platform_schema.rb | grep "^[+-]" | grep -v "^+++\|^---"`
Expected: só a versão `2026_09_14_000003` e a linha `t.integer "mfa_failed_attempts", default: 0, null: false`.

- [ ] **Step 4: Write the implementation**

`app/controllers/concerns/operator_authentication.rb` — depois de `OPERATOR_SESSION_TTL = 12.hours`:

```ruby
  # Tentativas de TOTP por sessão pendente; no limite a sessão é apagada e o
  # operador recomeça pela senha (Plano 3B).
  MAX_TOTP_ATTEMPTS = 5
```

`app/controllers/operators/sessions_controller.rb` — em `challenge_totp`, troque

```ruby
      unless Mfa::Verify.call(session.operator, code: params[:code])
        return render(json: { error: "invalid_code" }, status: :unauthorized)
      end
```

por

```ruby
      return register_failed_totp(session) unless Mfa::Verify.call(session.operator, code: params[:code])
```

e acrescente em `private`:

```ruby
    # O rate limit é por IP; trocando de IP dá para insistir no código. Por sessão
    # pendente, no MAX_TOTP_ATTEMPTS-ésimo erro a sessão é apagada e o operador
    # volta ao passo da senha. O incremento é atômico no banco, para duas
    # requisições simultâneas não contarem uma só.
    def register_failed_totp(session)
      OperatorSession.where(id: session.id).update_all("mfa_failed_attempts = mfa_failed_attempts + 1")
      return render(json: { error: "invalid_code" }, status: :unauthorized) if
        session.reload.mfa_failed_attempts < OperatorAuthentication::MAX_TOTP_ATTEMPTS

      session.destroy
      cookies.delete(OperatorAuthentication::COOKIE)
      render json: { error: "too_many_attempts" }, status: :unauthorized
    end
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/operators/sessions_spec.rb`
Expected: PASS, 0 failures.

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 0 pending.

- [ ] **Step 6: Commit**

```bash
git -C apps/api add db/platform_migrate/20260914000003_add_mfa_failed_attempts_to_operator_sessions.rb db/platform_schema.rb app/controllers/concerns/operator_authentication.rb app/controllers/operators/sessions_controller.rb spec/requests/operators/sessions_spec.rb
git -C apps/api commit -m "Cap wrong TOTP attempts per pending operator session

The per-IP rate limit alone let an attacker with the password keep
guessing codes from new addresses. Count wrong codes on the pending
session and destroy it on the fifth, so the operator starts over from
the password step.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 6: Prova em desenvolvimento, envs de deploy e documentação

**Files:**
- Modify: `README.md` (seção "Ambiente de desenvolvimento (monorepo)")
- Modify: `deploy/production/deploy.yml`, `deploy/development/deploy.yml`
- Modify: `deploy/production/secrets`, `deploy/development/secrets`, `deploy/SECRETS.md`

**Interfaces:**
- Consumes: tudo das Tasks 1-5; bancos de dev recriados na Task 1.
- Produces: envs `CITY_DASHBOARD_URL_TEMPLATE` (clear) e `GOVBR_REDIRECT_URI` (clear) nos dois `deploy.yml`;
  `GOVBR_CLIENT_ID` e `GOVBR_CLIENT_SECRET` como secrets definidos nos dois arquivos `secrets`; fluxo documentado.

- [ ] **Step 1: Restart the development api and confirm it boots**

As rotas e o autoload mudaram (`app/controllers/govbr`, `app/constraints/platform_auth_host.rb`).

```bash
docker compose restart api && sleep 8 && curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3030/up
```
Expected: `200`.

- [ ] **Step 2: Prove the operator entering a city, in development**

Todos os comandos só leem ou usam o app normalmente (login, grant, requisições). Guarde os arquivos temporários num
diretório de rascunho (ex.: `$TMPDIR` ou o scratchpad da sessão) — os caminhos abaixo usam `/tmp/rs3b-*`.

```bash
curl -s -o /tmp/rs3b-login.json -D /tmp/rs3b-login.h -H "Host: admin.localhost" -H "Content-Type: application/json" -d '{"email_address":"dev@local","password":"dev-password"}' http://localhost:3030/session && cat /tmp/rs3b-login.json
```
Expected: `{"mfa_required":true,"session_id":"..."}`.

```bash
CODE=$(docker compose exec -T api bin/rails runner 'require "rotp"; puts ROTP::TOTP.new(Operator.find_by!(email_address: "dev@local").otp_secret).now' | tail -1); SID=$(ruby -rjson -e 'puts JSON.parse(File.read("/tmp/rs3b-login.json"))["session_id"]'); OPC=$(grep -o 'operator_session_id=[^;]*' /tmp/rs3b-login.h); curl -s -o /dev/null -w "challenge=%{http_code}\n" -H "Host: admin.localhost" -H "Content-Type: application/json" -H "Cookie: $OPC" -d "{\"session_id\":\"$SID\",\"code\":\"$CODE\"}" http://localhost:3030/session/challenge; curl -s -o /tmp/rs3b-grant.json -w "city_grants=%{http_code}\n" -H "Host: admin.localhost" -H "Content-Type: application/json" -H "Cookie: $OPC" -d '{"city_slug":"curitiba"}' http://localhost:3030/city_grants; cat /tmp/rs3b-grant.json
```
Expected: `challenge=200`, `city_grants=201` e `{"redirect_url":"http://curitiba.localhost:5175/dashboard/?grant=...","expires_in":60}`.

```bash
GRANT=$(ruby -rjson -ruri -e 'puts URI.decode_www_form(URI.parse(JSON.parse(File.read("/tmp/rs3b-grant.json"))["redirect_url"]).query).to_h["grant"]'); curl -s -o /tmp/rs3b-city.json -D /tmp/rs3b-city.h -w "grant=%{http_code}\n" -H "Host: curitiba.localhost" -H "Content-Type: application/json" -d "{\"token\":\"$GRANT\"}" http://localhost:3030/session/grant; cat /tmp/rs3b-city.json; echo; curl -s -o /dev/null -w "reuse=%{http_code}\n" -H "Host: curitiba.localhost" -H "Content-Type: application/json" -d "{\"token\":\"$GRANT\"}" http://localhost:3030/session/grant
```
Expected: `grant=201`, JSON com `"operator":true` e `"email_address":"dev@local"`, `Set-Cookie: session_id=...` sem
`domain=`, e `reuse=401`.

```bash
SC=$(grep -o 'session_id=[^;]*' /tmp/rs3b-city.h | head -1); curl -s -o /dev/null -w "reports=%{http_code}\n" -H "Host: curitiba.localhost" -H "Cookie: $SC" "http://localhost:3030/admin/api/reports?period=30d"; curl -s -w " invite=%{http_code}\n" -H "Host: curitiba.localhost" -H "Content-Type: application/json" -H "Cookie: $SC" -d '{"email":"x@x.com","role":"viewer"}' http://localhost:3030/setup/invitations; curl -s -o /dev/null -w "maringa=%{http_code}\n" -H "Host: maringa.localhost" -H "Cookie: $SC" "http://localhost:3030/admin/api/reports?period=30d"
```
Expected: `reports=200`, `{"error":"operator_read_only"} invite=403`, `maringa=401`.

```bash
psql -d rota_saude_platform_development -Atc "select count(*) from platform_events where name='operator.city_access'"; psql -d rota_saude_city_curitiba -Atc "select count(*) from domain_events where name='operator.city_access'"
```
Expected: `1` ou mais nos dois.

- [ ] **Step 3: Prove the gov.br routes and the TOTP cap, in development**

Sem credenciais do gov.br em dev, o começo do login responde 502 e o callback só é servido em `auth.*`:

```bash
curl -s -w " start=%{http_code}\n" -X POST -H "Host: curitiba.localhost" http://localhost:3030/auth/govbr/start; curl -s -o /dev/null -w "callback_city=%{http_code}\n" -H "Host: curitiba.localhost" "http://localhost:3030/auth/govbr/callback?code=x&state=y"; curl -s -w " callback_auth=%{http_code}\n" -H "Host: auth.localhost" "http://localhost:3030/auth/govbr/callback?code=x&state=y"
```
Expected: `{"error":"govbr_integration_error"} start=502`, `callback_city=404`, `{"error":"invalid_state"} callback_auth=400`.

Teto de TOTP (cinco códigos errados na mesma sessão pendente):

```bash
curl -s -o /tmp/rs3b-cap.json -D /tmp/rs3b-cap.h -H "Host: admin.localhost" -H "Content-Type: application/json" -d '{"email_address":"dev@local","password":"dev-password"}' http://localhost:3030/session; SID=$(ruby -rjson -e 'puts JSON.parse(File.read("/tmp/rs3b-cap.json"))["session_id"]'); OPC=$(grep -o 'operator_session_id=[^;]*' /tmp/rs3b-cap.h); for i in 1 2 3 4 5; do curl -s -w " try$i=%{http_code}\n" -H "Host: admin.localhost" -H "Content-Type: application/json" -H "Cookie: $OPC" -d "{\"session_id\":\"$SID\",\"code\":\"000000x\"}" http://localhost:3030/session/challenge; done
```
Expected: quatro `{"error":"invalid_code"} tryN=401` e `{"error":"too_many_attempts"} try5=401`.
(O rate limit de 10 a cada 3 minutos por IP conta estas requisições; se aparecer `too_many_requests`, espere 3 minutos
e repita — não é falha do teto.)

Cole todas as saídas no relatório.

- [ ] **Step 4: Deploy envs and docs**

Nos dois `deploy.yml` (`deploy/production/deploy.yml`, `deploy/development/deploy.yml`):
- em `env.clear`, acrescente (produção com o domínio de exemplo já usado no arquivo; desenvolvimento com o de `dev`):

produção:
```yaml
    CITY_DASHBOARD_URL_TEMPLATE: "https://%{slug}.rota-saude.example/dashboard/"
    GOVBR_REDIRECT_URI: "https://auth.rota-saude.example/auth/govbr/callback"
```

desenvolvimento:
```yaml
    CITY_DASHBOARD_URL_TEMPLATE: "https://%{slug}.dev.rota-saude.example/dashboard/"
    GOVBR_REDIRECT_URI: "https://auth.dev.rota-saude.example/auth/govbr/callback"
```

- em `env.secret`, acrescente em ordem alfabética `GOVBR_CLIENT_ID` e `GOVBR_CLIENT_SECRET`.

Em `deploy/production/secrets`, antes do bloco do WhatsApp:

```bash
# gov.br OIDC — cliente registrado com redirect_uri única em auth.* (Plano 3B).
GOVBR_CLIENT_ID=$(op read "op://${OP_VAULT}/govbr/client_id")
GOVBR_CLIENT_SECRET=$(op read "op://${OP_VAULT}/govbr/client_secret")
```

Em `deploy/development/secrets`, antes do bloco do WhatsApp:

```bash
# gov.br OIDC (staging). Vazio = login gov.br responde 502 em dev.
GOVBR_CLIENT_ID=${GOVBR_CLIENT_ID:-}
GOVBR_CLIENT_SECRET=${GOVBR_CLIENT_SECRET:-}
```

Em `deploy/SECRETS.md`, no inventário, acrescente:
`- \`GOVBR_CLIENT_ID\` / \`GOVBR_CLIENT_SECRET\` — cliente OIDC do gov.br; a redirect_uri registrada é a ÚNICA de auth.* (\`GOVBR_REDIRECT_URI\`).`
e, na lista de itens do cofre, `govbr` (campos `client_id`, `client_secret`).

Run: `bash -n apps/api/deploy/production/secrets && bash -n apps/api/deploy/development/secrets && for f in production development; do for k in $(ruby -ryaml -e 'puts YAML.load_file(ARGV[0])["env"]["secret"]' apps/api/deploy/$f/deploy.yml); do grep -q "^$k=" apps/api/deploy/$f/secrets || echo "MISSING $f $k"; done; done`
Expected: sem saída.

No `README.md`, seção "Ambiente de desenvolvimento (monorepo)", acrescente um parágrafo:

```markdown
**Operador entrando numa cidade (Plano 3B).** No console (`admin.localhost`), depois do login com TOTP,
`POST /city_grants {city_slug}` devolve a URL da cidade com um grant de 60 s e uso único; a cidade o consome em
`POST /session/grant {token}`. A sessão de operador na cidade é SÓ LEITURA (painéis `/admin/api/*` e a própria sessão),
vale 1 hora e é auditada no banco de plataforma e no da cidade. Cinco códigos TOTP errados apagam a sessão pendente.

**gov.br (Plano 3B).** O login começa na cidade (`POST /auth/govbr/start`) e o callback é ÚNICO, em
`auth.<domínio>/auth/govbr/callback`, que volta para a cidade com um grant. Variáveis: `GOVBR_CLIENT_ID`,
`GOVBR_CLIENT_SECRET`, `GOVBR_REDIRECT_URI`, `GOVBR_ISSUER_URL` (default staging). Sem elas, `start` responde 502.
O destino de volta usa `CITY_DASHBOARD_URL_TEMPLATE` (default `http://%{slug}.localhost:5175/dashboard/`).
```

- [ ] **Step 5: Run the whole suite**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec --order rand:4711`
Expected: 0 failures, 0 pending.

- [ ] **Step 6: Commit**

```bash
git -C apps/api add README.md deploy/production/deploy.yml deploy/development/deploy.yml deploy/production/secrets deploy/development/secrets deploy/SECRETS.md
git -C apps/api commit -m "Document city grants and the single gov.br callback, and wire their deploy envs

Add the city dashboard URL template and the gov.br redirect URI as clear
envs, define the gov.br client credentials as secrets, and describe how
an operator enters a city and how gov.br login returns to it.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Definition of done

- `sessions` da cidade aceita exatamente um ator (usuário OU operador), no modelo e no banco.
- Sessão de operador na cidade: só lê `/admin/api/*` e `GET/DELETE /session`; qualquer outra ação → `403
  operator_read_only`; vale 1 h; cai com a desativação do operador; não existe em outra cidade. Guarda de arquitetura
  fixa a allowlist.
- Grant: 60 s, uso único (consumo atômico), válido só na cidade emitida, `kind`/sujeito vindos da linha.
- Console emite grant para cidade ativa; a cidade consome e audita nos dois bancos (`operator.city_access`).
- gov.br: `start` na cidade com `state` assinado (cidade + nonce); callback só em `auth.*`, confere nonce, provisiona na
  cidade do state, volta com grant de usuário; nenhum callback no host da cidade.
- Cinco códigos TOTP errados apagam a sessão pendente do operador.
- Bancos de dev de cidade recriados com o schema novo; prova por `curl` da Task 6 colada no relatório.
- Suíte: 0 failures, **0 pending**.

## O que este plano NÃO faz

- Replay de TOTP, tempo de resposta no login da cidade, troca do id de sessão após o TOTP (estacionados por decisão do
  usuário).
- Enrollment de MFA e reset de senha do operador; purga de sessões pendentes/expiradas.
- `city:migrate:all`, 503 por cidade atrasada, provisionamento em duas fases (Plano 4).
- Frontends: hosts por cidade, telas de "entrar na cidade" e de retorno do gov.br, CORS dinâmico, proxy do Vite
  (Plano 6). Também é do deploy/Plano 6 publicar `admin.*`, `auth.*` e `*.` de cidade no proxy do Kamal e fazer o proxy
  sobrescrever `X-Forwarded-Host`.
- Chave de assinatura/cifra por cidade (Plano 6). O grant e o state usam a chave da plataforma.
- PKCE no gov.br (a troca do `code` é feita pelo servidor com `client_secret`).

## Riscos

1. **Negação por padrão depende de toda ação passar por `require_authentication`.** Uma ação com
   `allow_unauthenticated_access` nunca vê a sessão de operador — hoje isso é intencional (login, grant, reset,
   convite público). Um controller novo que pule a autenticação e leia `Current.session` por conta própria contornaria
   a regra; a guarda de arquitetura só fixa quem LIBERA, não quem pula.
2. **A auditoria em dois bancos não é atômica.** A ordem escolhida (plataforma antes, cidade depois) garante que não
   exista sessão de operador sem auditoria; o custo é poder existir auditoria de uma tentativa sem sessão.
3. **Recriar os bancos de dev apaga os dados de exemplo** (repovoados por seeds). Qualquer dado manual que o usuário
   tenha criado nesses dois bancos se perde — é a decisão registrada.
4. **O gov.br real não é exercitado** (sem credenciais): `exchange_code_for_claims` e a verificação do `id_token` seguem
   cobertos só por mock. A integração real é verificação de operação, não deste plano.
