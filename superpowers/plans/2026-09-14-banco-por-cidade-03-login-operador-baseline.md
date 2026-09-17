# Banco por cidade — Plano 3: Login na cidade, operador na plataforma e baseline de duas cidades

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fechar a identidade do banco por cidade do lado do servidor: usuário da cidade loga só na cidade dele,
operador loga no console de plataforma (`admin.*`) com TOTP, os painéis exigem vínculo ativo, e o dev sobe duas
cidades reais com dados distintos.

**Architecture:** O login da cidade já roda dentro da conexão da cidade (Planos 1-2); este plano tira dele o ramo
morto de operador e prova o isolamento com specs de request e uma guarda de arquitetura contra `domain:` em cookie.
O operador ganha um fluxo próprio — `Operators::SessionsController`, contra `Operator`/`OperatorSession` no banco
de plataforma — servido nos MESMOS caminhos (`/session`, `/session/challenge`) mas só quando o host é `admin.*`,
por uma constraint de rota. O dev ganha `city:dev_up`/`city:dev_baseline`, que criam banco, carregam schema e
ativam `curitiba` e `maringa`; o seed e o dataset de demonstração passam a semear cada cidade na conexão dela.

**Tech Stack:** Rails 8.1.3, Ruby 3.3.6, PostgreSQL, RSpec + FactoryBot, ROTP, Docker Compose (serviço `api`).

**Spec:** `docs/superpowers/specs/2026-09-12-banco-por-cidade-design.md` (§2 resolução, §5 identidade)
**Planos anteriores:** `2026-09-12-banco-por-cidade-01-fundacao.md` e `2026-09-12-banco-por-cidade-02-desmontagem.md`
(ambos mergeados em `main` do repo `apps/api`; Plano 2 = merge `f27cfd1`). Este plano promove os Esboços A e B do Plano 2.

## Decisões que este plano carrega

Tomadas com o usuário em 2026-09-14, antes da escrita:

- **Escopo:** login da cidade endurecido + login do operador na plataforma + baseline de duas cidades + dataset de
  demonstração reescrito por cidade.
- **Grant assinado e gov.br ficam para o plano seguinte (nome de trabalho: Plano 3B), juntos** — a spec §5 usa o
  mesmo mecanismo de grant para os dois. Consequências aceitas: `SetupController#deactivate_user` segue 403 para
  todos; os exemplos de operador em `spec/commands/municipality_channels/rotate_token_spec.rb` e
  `spec/requests/admin/api/reports_spec.rb` seguem em `skip`, apontando para o 3B; `/auth/govbr/callback` segue
  provisório, na cidade do host.
- **Nenhum frontend é tocado.** Tudo é provado por request spec e por `curl` com cabeçalho `Host`. Consequência
  conhecida: no navegador, os frontends chamam a API via proxy do Vite com `changeOrigin: true` (Host `api:3000`),
  que não resolve cidade nem console — isso é do Plano 6.
- **Cidades de dev:** `curitiba` (Curitiba/PR) e `maringa` (Maringá/PR) — decisão do Plano 2.
- **O operador usa os mesmos caminhos e o mesmo formato de resposta da sessão da cidade** (`POST /session` →
  `{ mfa_required, session_id }`; `POST /session/challenge` → `SessionUser`), para o frontend do admin não precisar
  mudar de rota no Plano 6.
- **Cookie do operador é de sessão do navegador** (sem `permanent`), com nome próprio `operator_session_id`. Conta
  privilegiada não fica logada indefinidamente; custo: relogar ao fechar o navegador.

## Global Constraints

- **Cookie de sessão é host-only: NUNCA `domain:`** (spec §5, "inegociável"). Vale para `session_id` e
  `operator_session_id`.
- **Resolver a cidade ANTES de autenticar.** Usuário e `Session` de cidade só são lidos dentro da conexão da cidade
  do host (`CityResolution` → `Authentication`, nessa ordem — nunca `prepend_before_action` para autenticação).
- **Operador mora só na plataforma** (`operators`, `operator_sessions`). Nenhuma conta de operador replicada em banco
  de cidade.
- **Subdomínios reservados (`admin`, `api`, `auth`, `www`) nunca resolvem cidade.** `admin.*` é o console.
- **O banco de plataforma nunca guarda dado de cidadão.** `PlatformEvent` recusa chave de payload que contenha
  `email`, `cpf`, `phone`, `wa_id`, `provider_uid`, `body`, `name` (salvo `phone_number_id`, `city_name`) ou `from`.
- **`connects_to` só uma vez, no corpo de `CityRecord`.** Cidade entra por `CityConnection.with`.
- **Ponteiros de ADR em comentário só na faixa ADR-0001..ADR-0015** (`spec/adr_pointers_spec.rb` falha fora dela).
- **Nada de constante top-level em spec** (`X = ...` dentro de `describe` vaza para o topo); use método ou `let`.
- **Todo comando roda no container**, a partir da raiz do monorepo: `docker compose exec -T api <cmd>`; rspec com
  `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Nunca `bundle` no host.
- **Sondagens só como spec de rascunho** (apagado depois): `rails runner` com `ActiveRecord::Base.transaction` NÃO
  cobre pool de cidade e já commitou linha em banco de teste de cidade. `rails runner` só leitura é permitido.
- **Nenhuma operação destrutiva** em `rota_saude_development`, `rota_saude_test`, `rota_saude_platform_*` ou bancos de
  cidade de teste. `city:load_schema` usa `force: :cascade` e só pode rodar num banco SEM o schema de cidade.
- **Suíte verde ao fim de CADA task** (este plano não tem janela vermelha).
- **Nenhum spec enfraquecido para passar.** Exemplo removido precisa de justificativa: o comportamento deixou de
  existir por decisão da spec, e o invariante tem de estar provado em outro spec nomeado.
- **Contrato JSON dos frontends preservado:** `id`, `email_address`, `mfa_enrolled`, `operator`, `mfa_verified_at`,
  `memberships[].municipality_id|municipality_name|municipality_uf|role`, `mfa_required`, `session_id`, e o envelope
  `{ data:, as_of: }` com `scope.municipality`.
- **Commits em inglês**, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.

## Estado herdado (repo `apps/api`, `main` em `b4cac07`)

| Peça | Onde | Comportamento hoje |
|---|---|---|
| `CityResolution` | `app/controllers/concerns/city_resolution.rb` | `around_action :within_city`; 404 host desconhecido/reservado/não servível, 403 suspensa |
| `Authentication` | `app/controllers/concerns/authentication.rb` | `before_action :require_authentication` dentro da cidade; cookie `session_id` assinado, `permanent`, sem `domain` |
| `SessionsController` | `app/controllers/sessions_controller.rb` | login de usuário da cidade; ainda tem ramos `user.operator?` (sempre `false`) e `challenge_totp` |
| `User#operator?` | `app/models/user.rb:44` | sempre `false` |
| `Operator`, `OperatorSession` | `app/models/operator.rb`, `operator_session.rb` | modelos de plataforma, com `has_secure_password`, `encrypts :otp_secret`, `mfa_enrolled?`; **nenhum controller os usa** |
| `Mfa::Verify` | `app/services/mfa/verify.rb` | `call(obj, code:)` — funciona com qualquer objeto com `otp_secret`/`otp_recovery_codes`/`update!` |
| `Admin::Api::BaseController` | `app/controllers/admin/api/base_controller.rb` | exige só sessão; **sem gate de membership** (revisão 5b M3) |
| `CityCatalog` | `app/models/city_catalog.rb` | `find_by_host`, `reserved_host?`, `RESERVED = %w[admin api auth www]` |
| `CityProvisioner` | `app/commands/city_provisioner.rb` | só registra no catálogo (`provisioning`), URL `rota_saude_city_<slug>` com superusuário |
| `city.rake` | `lib/tasks/city.rake` | `load_schema`, `test_databases`, `create` — nenhum cria banco de cidade de dev nem ativa |
| `db/seeds.rb` | | operador `dev@local` + UMA cidade (`SEED_CITY_SLUG`), pulada se não estiver ativa |
| `db/seeds/dashboard_demo.rb` | | levanta na linha 5 (schema pré-corte) |
| `start.sh` (raiz do monorepo, **fora de git**) | | roda `platform:bootstrap`, `db:migrate:platform`, `city:test_databases`, `db:seed` |

Suíte atual: **447 examples, 0 failures, 3 pending**. Pendings: gate de membership (`admin/api/reports_spec.rb`),
operador em `admin/api/reports_spec.rb`, operador em `rotate_token_spec.rb`.

Banco de dev hoje: catálogo com `curitiba` em `provisioning`, sem banco `rota_saude_city_curitiba`.

## File Structure

| Arquivo | Task | Responsabilidade |
|---|---|---|
| `app/controllers/admin/api/base_controller.rb` | 1 | gate `require_city_membership` |
| `spec/requests/admin/api/membership_gate_spec.rb` | 1 | gate em todo endpoint |
| `app/models/city_catalog.rb` | 2 | `console_host?(host)` |
| `app/constraints/platform_console_host.rb` | 2 | constraint de rota `admin.*` |
| `app/controllers/concerns/operator_authentication.rb` | 2 | sessão/cookie de operador |
| `app/controllers/operators/base_controller.rb` | 2 | base do console, sem `CityResolution` |
| `app/controllers/operators/sessions_controller.rb` | 2 | senha → sessão pendente → TOTP |
| `app/models/current.rb` | 2 | `Current.operator_session` |
| `config/routes.rb` | 2, 3 | rotas do console antes das de cidade; remove challenge da cidade |
| `spec/requests/operators/sessions_spec.rb` | 2 | login de operador e isolamento de host |
| `app/controllers/sessions_controller.rb` | 3 | sem ramo de operador |
| `spec/requests/city_session_isolation_spec.rb` | 3 | cookie de A não autentica em B |
| `spec/architecture/cookie_domain_spec.rb` | 3 | guarda contra `domain:` |
| `lib/tasks/city.rake` | 4 | `city:dev_up`, `city:dev_baseline` |
| `db/seeds.rb` | 4 | duas cidades |
| `README.md`, `start.sh` (fora de git) | 4 | baseline de dev |
| `lib/dashboard_demo.rb` | 5 | dataset por cidade + verificação |
| `db/seeds/dashboard_demo.rb`, `lib/tasks/seed_demo.rake` | 5 | carregam/verificam o dataset |
| `spec/lib/dashboard_demo_spec.rb` | 5 | dataset popula o que o verify confere |

---

### Task 1: Gate de membership nos painéis da cidade

**Files:**
- Modify: `app/controllers/admin/api/base_controller.rb`
- Modify: `spec/requests/admin/api/reports_spec.rb:66-79` (tira o `pending`)
- Create: `spec/requests/admin/api/membership_gate_spec.rb`

**Interfaces:**
- Consumes: `Authentication#current_user` (User da cidade do host), `Membership.active`, `Membership::ROLES`,
  `CityRequestAuth#sign_in_as(user)` (spec/support/city_request_auth.rb).
- Produces: todo endpoint de `Admin::Api::*` responde `403 {"error":"no_city_membership"}` a usuário autenticado sem
  membership ativo. Sem sessão continua `401`.

Contexto: revisão 5b M3. Um usuário da cidade sem vínculo — criado pelo gov.br, ou com o membership revogado
(`RevokeMembership` não derruba sessões) — lê todos os painéis da própria cidade.

- [ ] **Step 1: Write the failing test**

Crie `spec/requests/admin/api/membership_gate_spec.rb`:

```ruby
require "rails_helper"

# Revisão 5b M3: Admin::Api só exigia sessão. Um usuário da cidade sem
# membership ativo — auto-provisionado pelo gov.br, ou com o membership revogado
# (RevokeMembership não derruba sessões) — lia todos os painéis da cidade.
RSpec.describe "Admin::Api membership gate", type: :request do
  def endpoints
    %w[
      /admin/api/overview /admin/api/ingestion /admin/api/conversations
      /admin/api/consent /admin/api/triages /admin/api/reports
      /admin/api/classification /admin/api/protocols /admin/api/queues
      /admin/api/events /admin/api/health /admin/api/municipalities
    ] + [ "/admin/api/triages/#{SecureRandom.uuid}/trail", "/admin/api/protocols/#{SecureRandom.uuid}" ]
  end

  def user_with(role: nil, revoked: false)
    user = User.create!(email_address: "gate-#{SecureRandom.hex(4)}@x.com", password: "secret123")
    if role
      Membership.create!(user: user, role: role, granted_at: 2.days.ago, revoked_at: (revoked ? 1.day.ago : nil))
    end
    user
  end

  it "refuses every endpoint to a city user with no membership" do
    sign_in_as(user_with)

    endpoints.each do |path|
      get path, params: { period: "30d" }
      expect(response).to have_http_status(:forbidden), "#{path} respondeu #{response.status}"
      expect(JSON.parse(response.body)).to eq("error" => "no_city_membership")
    end
  end

  it "refuses a user whose only membership was revoked, even with a live session" do
    sign_in_as(user_with(role: "viewer", revoked: true))

    get "/admin/api/reports", params: { period: "30d" }

    expect(response).to have_http_status(:forbidden)
  end

  it "lets every active local role through (positive control)" do
    Membership::ROLES.each do |role|
      sign_in_as(user_with(role: role))
      get "/admin/api/reports", params: { period: "30d" }
      expect(response).to have_http_status(:ok), "#{role} respondeu #{response.status}"
    end
  end

  it "still answers 401, not 403, without a session" do
    get "/admin/api/reports", params: { period: "30d" }
    expect(response).to have_http_status(:unauthorized)
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/admin/api/membership_gate_spec.rb`
Expected: FAIL nos dois primeiros exemplos (`expected 403 ... got 200` ou `422`); os dois últimos passam.

- [ ] **Step 3: Write minimal implementation**

Em `app/controllers/admin/api/base_controller.rb`, troque o bloco do topo da classe e acrescente o método privado.
O cabeçalho de comentário ganha a responsabilidade nova:

```ruby
# Base de todos os controllers do namespace Admin:: (read-only).
# Auth real via cookie de sessão (ADR-0011).
#
# Responsabilidades:
#  - fronteira de auth (Authentication concern → require_authentication), que
#    roda DENTRO da conexão da cidade do host (CityResolution);
#  - gate de vínculo: sessão não basta, o usuário precisa de ao menos um
#    membership ATIVO nesta cidade (revisão 5b M3);
#  - período e timezone do escopo;
#  - envelope universal { data:, as_of: }, com o descritor da cidade.
#
# Escopo = o banco da cidade do host. Não há município a resolver nem visão
# cross-tenant (spec banco-por-cidade §5): as queries deste namespace leem o
# banco inteiro da cidade, sem filtro. O parâmetro de município que os
# frontends ainda enviam é ignorado.
#
# Nenhuma rota de escrita é permitida neste namespace (critério de aceite §10).
class Admin::Api::BaseController < ApplicationController
  include Authentication

  TZ = ActiveSupport::TimeZone["America/Sao_Paulo"]

  # Depois de require_authentication (incluído acima) e antes de qualquer
  # leitura. Papel específico por painel não é deste plano: hoje qualquer papel
  # local lê os painéis da própria cidade.
  before_action :require_city_membership
  before_action :resolve_scope
```

E, logo depois de `private`:

```ruby
  def require_city_membership
    return if current_user.memberships.active.exists?

    render json: { error: "no_city_membership" }, status: :forbidden
  end
```

Não mexa no resto do arquivo.

- [ ] **Step 4: Remove the pending in reports_spec**

Em `spec/requests/admin/api/reports_spec.rb`, o exemplo `"a city user with no active membership must not see the
city's reports"` perde as linhas do `pending "Plano 3: gate de membership — ..."` e ganha a checagem do corpo.
Fica assim:

```ruby
  it "a city user with no active membership must not see the city's reports" do
    seed_report(tier: "alta")
    homeless = User.create!(email_address: "no-membership-#{SecureRandom.hex(3)}@x.com", password: "secret123")
    sign_in_as(homeless)

    get "/admin/api/reports", params: { period: "30d" }

    expect(response).to have_http_status(:forbidden)
    expect(JSON.parse(response.body)).to eq("error" => "no_city_membership")
  end
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/admin spec/controllers/admin`
Expected: PASS, 0 failures; o único pending restante nesse diretório é o `skip` de operador em `reports_spec.rb`.

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: `451 examples, 0 failures, 2 pending` (447 + 4 novos; o pending do gate virou exemplo verde).

- [ ] **Step 6: Commit**

```bash
git -C apps/api add app/controllers/admin/api/base_controller.rb spec/requests/admin/api/membership_gate_spec.rb spec/requests/admin/api/reports_spec.rb
git -C apps/api commit -m "Require an active city membership for the city admin API

A signed-in city user with no active membership (self-provisioned via
gov.br, or whose membership was revoked) could read every panel of the
city. Refuse with 403 no_city_membership after authentication.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: Login do operador no console de plataforma (`admin.*`)

**Files:**
- Modify: `app/models/city_catalog.rb` (método público `console_host?`)
- Modify: `spec/models/city_catalog_spec.rb`
- Create: `app/constraints/platform_console_host.rb`
- Create: `app/controllers/concerns/operator_authentication.rb`
- Create: `app/controllers/operators/base_controller.rb`
- Create: `app/controllers/operators/sessions_controller.rb`
- Modify: `app/models/current.rb`
- Modify: `config/routes.rb` (bloco do console no TOPO do `draw`)
- Create: `spec/requests/operators/sessions_spec.rb`

**Interfaces:**
- Consumes: `Operator` (`authenticate`, `active?`, `mfa_enrolled?`, `operator_sessions`, `otp_secret`),
  `OperatorSession` (`operator_id`, `mfa_verified_at`, `user_agent`, `ip_address`, `created_at`),
  `Mfa::Verify.call(obj, code:) → true|false`, `Platform.audit(name, **payload)`,
  `CityRequestAuth#test_city_host` (spec).
- Produces:
  - `CityCatalog.console_host?(host) → Boolean` — `true` só quando o primeiro rótulo do host é `admin`.
  - `PlatformConsoleHost.matches?(request) → Boolean` (constraint de rota).
  - `OperatorAuthentication` com `COOKIE = :operator_session_id`, `PENDING_MFA_WINDOW = 10.minutes`,
    `.allow_unauthenticated_operator_access(**opts)`, e privados `current_operator`,
    `require_operator_authentication`, `resume_operator_session`, `start_pending_operator_session_for(operator)`,
    `write_operator_cookie(session)`, `terminate_operator_session`.
  - `Current.operator_session` (OperatorSession verificada por TOTP, ou nil).
  - Rotas no host `admin.*`: `POST /session`, `POST /session/challenge`, `GET /session`, `DELETE /session`
    → `Operators::SessionsController`.
  - `PlatformEvent` `operator.login` com payload exatamente `{ "operator_id", "operator_session_id" }`.

Fluxo (ADR-0011: operador exige TOTP a cada login):

1. `POST /session {email_address, password}` no host `admin.*`. Operador inexistente, senha errada ou desativado →
   `401 invalid_credentials`. Sem MFA → `403 mfa_enrollment_required`, sem criar sessão. Senão cria
   `OperatorSession` PENDENTE (`mfa_verified_at: nil`), planta o cookie `operator_session_id` e responde
   `200 { mfa_required: true, session_id }`. Sessão pendente NÃO autentica.
2. `POST /session/challenge {session_id, code}`. A sessão tem de ser a do cookie deste cliente, pendente, criada há
   menos de 10 minutos, de operador ativo; senão `401 invalid_session`. TOTP errado → `401 invalid_code`. Certo →
   carimba `mfa_verified_at`, audita `operator.login` e responde `200` no formato `SessionUser`.
3. `GET /session` → `200` com o operador, ou `401`. `DELETE /session` → `204`, destrói a sessão e o cookie.

- [ ] **Step 1: Write the failing tests**

Em `spec/models/city_catalog_spec.rb`, acrescente dentro do `RSpec.describe CityCatalog`:

```ruby
  describe ".console_host?" do
    it "is true only when the first label is admin" do
      expect(described_class.console_host?("admin.rotasaude.app")).to be(true)
      expect(described_class.console_host?("ADMIN.rotasaude.app")).to be(true)
      expect(described_class.console_host?("admin.localhost:5174")).to be(true)
      expect(described_class.console_host?("curitiba.rotasaude.app")).to be(false)
      expect(described_class.console_host?("api.rotasaude.app")).to be(false)
      expect(described_class.console_host?("")).to be(false)
      expect(described_class.console_host?(nil)).to be(false)
    end
  end
```

Crie `spec/requests/operators/sessions_spec.rb`:

```ruby
require "rails_helper"

# Login do operador no console de plataforma (admin.*), contra Operator no banco
# de plataforma. Operador exige TOTP a cada login (ADR-0011): a senha só abre uma
# sessão PENDENTE; o cookie só autentica depois do challenge.
RSpec.describe "Operator session on the platform console", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  let(:password) { "s3nha-forte-1" }
  let!(:operator) { create_operator("op") }

  def create_operator(prefix)
    Operator.create!(email_address: "#{prefix}-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end

  def console_host = "admin.rotasaude.app"
  def json = JSON.parse(response.body)
  def set_cookie_header = Array(response.headers["Set-Cookie"]).join("\n")
  def totp(op = operator) = ROTP::TOTP.new(op.otp_secret).now

  def login!(op = operator, pwd = password)
    post "/session", params: { email_address: op.email_address, password: pwd }
  end

  def verified_login!
    login!
    post "/session/challenge", params: { session_id: json["session_id"], code: totp }
    expect(response).to have_http_status(:ok)
  end

  # O before global de request specs aponta para o host da cidade de teste; este
  # before roda depois e troca para o console.
  before { host! console_host }

  it "answers the password step with mfa_required and a pending session that does not authenticate yet" do
    login!

    expect(response).to have_http_status(:ok)
    expect(json).to include("mfa_required" => true)
    expect(OperatorSession.find(json["session_id"]).mfa_verified_at).to be_nil

    get "/session"
    expect(response).to have_http_status(:unauthorized)
  end

  it "completes the login with a valid TOTP and audits it on the platform" do
    login!
    session_id = json["session_id"]

    expect {
      post "/session/challenge", params: { session_id: session_id, code: totp }
    }.to change { PlatformEvent.where(name: "operator.login").count }.by(1)

    expect(response).to have_http_status(:ok)
    expect(json).to include("id" => operator.id, "email_address" => operator.email_address,
                            "operator" => true, "mfa_enrolled" => true, "memberships" => [])
    expect(json["mfa_verified_at"]).to be_present
    expect(PlatformEvent.where(name: "operator.login").order(:occurred_at).last.payload)
      .to eq("operator_id" => operator.id, "operator_session_id" => session_id)

    get "/session"
    expect(response).to have_http_status(:ok)
    expect(json["id"]).to eq(operator.id)
  end

  it "refuses a wrong TOTP and keeps the session unauthenticated" do
    login!

    post "/session/challenge", params: { session_id: json["session_id"], code: "nao-e-um-codigo" }

    expect(response).to have_http_status(:unauthorized)
    expect(json).to eq("error" => "invalid_code")
    get "/session"
    expect(response).to have_http_status(:unauthorized)
  end

  it "refuses a challenge for a pending session that is not the one in this client's cookie" do
    other = create_operator("outro")
    foreign = other.operator_sessions.create!(user_agent: "outro navegador")
    login! # planta o cookie da sessão pendente DESTE cliente

    post "/session/challenge", params: { session_id: foreign.id, code: totp(other) }

    expect(response).to have_http_status(:unauthorized)
    expect(json).to eq("error" => "invalid_session")
    expect(foreign.reload.mfa_verified_at).to be_nil
  end

  it "refuses a challenge after the pending window" do
    login!
    session_id = json["session_id"]

    travel 11.minutes do
      post "/session/challenge", params: { session_id: session_id, code: totp }
    end

    expect(response).to have_http_status(:unauthorized)
    expect(json).to eq("error" => "invalid_session")
  end

  it "refuses wrong password, unknown email and a deactivated operator alike" do
    login!(operator, "errada")
    expect(response).to have_http_status(:unauthorized)
    expect(json).to eq("error" => "invalid_credentials")

    post "/session", params: { email_address: "ninguem@rotasaude.app", password: password }
    expect(response).to have_http_status(:unauthorized)
    expect(json).to eq("error" => "invalid_credentials")

    operator.update!(deactivated_at: Time.current)
    login!
    expect(response).to have_http_status(:unauthorized)
    expect(OperatorSession.where(operator: operator)).to be_empty
  end

  it "refuses an operator without MFA before creating any session" do
    operator.update!(otp_enabled: false)

    login!

    expect(response).to have_http_status(:forbidden)
    expect(json).to eq("error" => "mfa_enrollment_required")
    expect(OperatorSession.where(operator: operator)).to be_empty
  end

  it "logs out: destroys the session and the cookie stops authenticating" do
    verified_login!
    session_id = OperatorSession.where(operator: operator).sole.id

    delete "/session"

    expect(response).to have_http_status(:no_content)
    expect(OperatorSession.exists?(session_id)).to be(false)
    get "/session"
    expect(response).to have_http_status(:unauthorized)
  end

  it "sets a host-only operator cookie (never a Domain attribute)" do
    login!

    expect(set_cookie_header).to match(/operator_session_id=/)
    expect(set_cookie_header).not_to match(/domain=/i)
  end

  it "operator controllers never resolve a city" do
    expect(Operators::BaseController.ancestors).not_to include(CityResolution)
    expect(Operators::BaseController.ancestors).not_to include(Authentication)
  end

  describe "host isolation" do
    def verified_cookie
      verified_login!
      set_cookie_header[/operator_session_id=[^;]+/]
    end

    it "a verified operator cookie does not authenticate on a city host" do
      cookie = verified_cookie

      get "/session", headers: { "HOST" => test_city_host, "Cookie" => cookie }

      expect(response).to have_http_status(:unauthorized)
    end

    it "operator credentials are not city credentials" do
      post "/session", params: { email_address: operator.email_address, password: password },
                       headers: { "HOST" => test_city_host }

      expect(response).to have_http_status(:unauthorized)
      expect(json).to eq("error" => "invalid_credentials")
    end

    it "the console host serves no city API, even to a verified operator" do
      cookie = verified_cookie

      get "/admin/api/overview", headers: { "Cookie" => cookie }

      expect(response).to have_http_status(:not_found)
    end
  end

  # Defesa em profundidade: se uma rota nova esquecer a constraint do console,
  # o controller de operador continua recusando host de cidade.
  describe "an operator route drawn without the console constraint" do
    before(:all) do
      Rails.application.routes.disable_clear_and_finalize = true
      Rails.application.routes.draw do
        get "/_operator_session_without_constraint", to: "operators/sessions#show"
      end
    end

    after(:all) do
      Rails.application.routes.disable_clear_and_finalize = false
      Rails.application.reload_routes!
    end

    it "answers 404 on a city host" do
      get "/_operator_session_without_constraint", headers: { "HOST" => test_city_host }

      expect(response).to have_http_status(:not_found)
    end
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_catalog_spec.rb spec/requests/operators/sessions_spec.rb`
Expected: FAIL — `NoMethodError: undefined method 'console_host?'` e, no request spec, `NameError: uninitialized
constant Operators` / respostas `404 unknown_city` no host `admin.*`.

- [ ] **Step 3: Write minimal implementation**

`app/models/city_catalog.rb` — acrescente o método público logo depois de `reserved_host?`:

```ruby
    # Host do console de plataforma (admin.*). Reservado: nunca resolve cidade.
    def console_host?(host)
      label_for(host) == "admin"
    end
```

`app/constraints/platform_console_host.rb`:

```ruby
# Constraint de rota: casa requisições dirigidas ao console de plataforma
# (admin.*). Em config/routes.rb manda /session desse host para Operators::,
# e não para o SessionsController da cidade.
class PlatformConsoleHost
  def self.matches?(request)
    CityCatalog.console_host?(request.host)
  end
end
```

`app/models/current.rb`:

```ruby
# CurrentAttributes resetado por request e por job (ver ADR-0003).
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  # Cidade resolvida pelo host. Serve para log e para o envelope de resposta.
  # NUNCA use em WHERE: o escopo é a conexão, não um valor de coluna.
  attribute :city
  # Sessão de operador JÁ verificada por TOTP, no console de plataforma (admin.*).
  # Nunca coexiste com uma cidade resolvida: o console não resolve cidade.
  attribute :operator_session

  delegate :user, to: :session, allow_nil: true
end
```

`app/controllers/concerns/operator_authentication.rb`:

```ruby
# Autenticação de operador no console de plataforma (admin.*), contra
# Operator/OperatorSession no banco de plataforma. Ver ADR-0011.
#
# Diferenças deliberadas em relação a Authentication (usuário da cidade):
#   - operador exige TOTP a cada login: a senha só cria uma sessão PENDENTE, e só
#     sessão com mfa_verified_at autentica;
#   - cookie com nome próprio (operator_session_id) e de sessão do navegador
#     (sem `permanent`): conta privilegiada não fica logada indefinidamente;
#   - host-only, como todo cookie de sessão: NUNCA `domain:` (spec §5).
module OperatorAuthentication
  extend ActiveSupport::Concern

  COOKIE = :operator_session_id
  PENDING_MFA_WINDOW = 10.minutes

  included do
    before_action :require_operator_authentication
  end

  class_methods do
    def allow_unauthenticated_operator_access(**options)
      skip_before_action :require_operator_authentication, **options
    end
  end

  private

  def current_operator
    Current.operator_session&.operator
  end

  def require_operator_authentication
    resume_operator_session || render(json: { error: "unauthenticated" }, status: :unauthorized)
  end

  def resume_operator_session
    Current.operator_session ||= find_verified_operator_session
  end

  def find_verified_operator_session
    id = cookies.signed[COOKIE]
    return nil unless id

    session = OperatorSession.find_by(id: id)
    return nil unless session&.mfa_verified_at && session.operator.active?

    session
  end

  def start_pending_operator_session_for(operator)
    operator.operator_sessions.create!(user_agent: request.user_agent, ip_address: request.remote_ip).tap do |session|
      write_operator_cookie(session)
    end
  end

  def write_operator_cookie(session)
    cookies.signed[COOKIE] = {
      value: session.id,
      httponly: true,
      same_site: :lax,
      secure: Rails.env.production?
    }
  end

  def terminate_operator_session
    OperatorSession.find_by(id: cookies.signed[COOKIE])&.destroy
    Current.operator_session = nil
    cookies.delete(COOKIE)
  end
end
```

`app/controllers/operators/base_controller.rb`:

```ruby
# Base dos controllers do console de plataforma (admin.*).
#
# NÃO herda de ApplicationController: não há cidade a resolver aqui, e
# CityResolution devolveria 404 para o host reservado admin.*. A autenticação é
# contra Operator/OperatorSession, no banco de plataforma — nunca contra
# User/Session de uma cidade.
module Operators
  class BaseController < ActionController::API
    include ActionController::Cookies

    # Defesa em profundidade: as rotas já exigem PlatformConsoleHost. Declarado
    # ANTES de OperatorAuthentication para rodar primeiro: host de cidade recebe
    # 404, nunca 401.
    before_action :require_console_host

    include OperatorAuthentication

    private

    def require_console_host
      head :not_found unless CityCatalog.console_host?(request.host)
    end
  end
end
```

`app/controllers/operators/sessions_controller.rb`:

```ruby
# Sessão de operador no console de plataforma — JSON-only. Ver ADR-0011.
#
# Mesmos caminhos e mesmo formato de resposta da sessão da cidade, servidos só
# no host admin.* (config/routes.rb, PlatformConsoleHost):
#
#   POST   /session            { email_address, password } → 200 { mfa_required, session_id }
#   POST   /session/challenge  { session_id, code }        → 200 operador
#   GET    /session                                        → 200 operador | 401
#   DELETE /session                                        → 204
module Operators
  class SessionsController < BaseController
    allow_unauthenticated_operator_access only: %i[create challenge_totp]

    rate_limit to: 10, within: 3.minutes, only: %i[create challenge_totp],
               with: -> { render json: { error: "too_many_requests" }, status: :too_many_requests }

    def create
      operator = authenticate_operator(params[:email_address], params[:password])
      return render(json: { error: "invalid_credentials" }, status: :unauthorized) unless operator
      return render(json: { error: "mfa_enrollment_required" }, status: :forbidden) unless operator.mfa_enrolled?

      session = start_pending_operator_session_for(operator)
      render json: { mfa_required: true, session_id: session.id }, status: :ok
    end

    def challenge_totp
      session = pending_session
      return render(json: { error: "invalid_session" }, status: :unauthorized) unless session

      unless Mfa::Verify.call(session.operator, code: params[:code])
        return render(json: { error: "invalid_code" }, status: :unauthorized)
      end

      session.update!(mfa_verified_at: Time.current)
      write_operator_cookie(session)
      Current.operator_session = session
      Platform.audit("operator.login", operator_id: session.operator_id, operator_session_id: session.id)
      render json: serialize(session), status: :ok
    end

    def show
      render json: serialize(Current.operator_session)
    end

    def destroy
      terminate_operator_session
      head :no_content
    end

    private

    def authenticate_operator(email, password)
      return nil if email.blank? || password.blank?

      operator = Operator.find_by(email_address: email)
      return nil unless operator&.active?

      operator.authenticate(password) || nil
    end

    # A sessão do challenge tem de ser a MESMA cujo cookie este cliente recebeu no
    # passo da senha, ainda sem TOTP, dentro da janela e de operador ativo. Sem o
    # vínculo com o cookie, quem soubesse um session_id pendente de outro
    # operador poderia completar o login dele com um TOTP próprio.
    def pending_session
      id = params[:session_id].to_s
      return nil if id.empty? || cookies.signed[OperatorAuthentication::COOKIE] != id

      session = OperatorSession.find_by(id: id, mfa_verified_at: nil)
      return nil unless session
      return nil if session.created_at <= OperatorAuthentication::PENDING_MFA_WINDOW.ago
      return nil unless session.operator.active?

      session
    end

    # Mesmo formato do SessionUser que o frontend do admin já lê.
    def serialize(session)
      operator = session.operator
      {
        id: operator.id,
        email_address: operator.email_address,
        mfa_enrolled: operator.mfa_enrolled?,
        operator: true,
        mfa_verified_at: session.mfa_verified_at&.iso8601,
        memberships: []
      }
    end
  end
end
```

`config/routes.rb` — insira como PRIMEIRO bloco dentro de `Rails.application.routes.draw do`:

```ruby
  # Console de plataforma (admin.*): operador autentica contra Operator, no banco
  # de plataforma (Operators::SessionsController). Mesmos caminhos da sessão da
  # cidade, para o frontend do admin não mudar de rota. PRECISA vir antes das
  # rotas de cidade: a primeira rota que casa vence.
  constraints(PlatformConsoleHost) do
    scope module: :operators, as: :operator do
      resource :session, only: %i[create show destroy]
      post "/session/challenge", to: "sessions#challenge_totp"
    end
  end
```

Nesta task, a rota `post "/session/challenge", to: "sessions#challenge_totp"` da cidade continua existindo — sai na
Task 3, junto com a ação.

- [ ] **Step 4: Run tests to verify they pass**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_catalog_spec.rb spec/requests/operators/sessions_spec.rb`
Expected: PASS, 0 failures.

Run: `docker compose exec -T api bin/rails runner 'Rails.application.eager_load!; puts Operators::SessionsController.name'`
Expected: `Operators::SessionsController` (Zeitwerk carrega `app/constraints` e `app/controllers/operators`).

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 2 pending; o número de exemplos sobe exatamente pelos exemplos novos desta task.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/models/city_catalog.rb spec/models/city_catalog_spec.rb app/constraints/platform_console_host.rb app/controllers/concerns/operator_authentication.rb app/controllers/operators app/models/current.rb config/routes.rb spec/requests/operators/sessions_spec.rb
git -C apps/api commit -m "Log operators in on the platform console host

Operators authenticate against Operator in the platform database, on the
admin.* host only, through the same /session paths the admin frontend
already calls. The password step opens a pending session bound to this
client's cookie; only a TOTP-verified session authenticates, and the
login is audited as a platform event.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: Login da cidade endurecido

**Files:**
- Modify: `app/controllers/sessions_controller.rb` (sem ramo de operador, sem `challenge_totp`)
- Modify: `config/routes.rb` (remove `post "/session/challenge", to: "sessions#challenge_totp"` da cidade)
- Delete: `spec/controllers/sessions_controller_mfa_spec.rb`
- Modify: `spec/controllers/sessions_controller_govbr_spec.rb` (troca os dois contextos de operador)
- Create: `spec/requests/city_session_isolation_spec.rb`
- Create: `spec/architecture/cookie_domain_spec.rb`
- Modify (só comentário/texto): os arquivos da tabela do Step 6

**Interfaces:**
- Consumes: `Authentication#start_new_session_for(user)`, `CityRequestAuth#test_city_host`, factory `:city`,
  `CityDatabaseUrls#city_database_url(db)`, `TEST_CITY_B`, `Mfa::Enroll.call(user)`, rotas do console da Task 2.
- Produces: `POST /session` da cidade responde sempre `201` com `operator: false` (nunca `mfa_required`);
  `POST /session/challenge` num host de cidade não tem rota (`404`).

Por que remover o ramo em vez de deixar: `User#operator?` é `false` desde o Plano 2, então os ramos
`if user.operator?` do `SessionsController` e a ação `challenge_totp` da cidade são código morto — e código morto
num controller de autenticação é exatamente onde alguém "reativa" um bypass. O comportamento que eles protegiam
(operador exige TOTP a cada login) agora vive e é testado em `Operators::SessionsController` (Task 2).

Justificativa de remoção de spec (regra "nenhum spec enfraquecido"):
- `spec/controllers/sessions_controller_mfa_spec.rb` — seus dois exemplos ("create devolve mfa_required quando user é
  operador", "challenge_totp com código válido finaliza o login") testavam operador logando pela cidade, que a spec §5
  elimina. Os invariantes vivem em `spec/requests/operators/sessions_spec.rb`: "answers the password step with
  mfa_required..." e "completes the login with a valid TOTP...".
- Os contextos "operador (operator? stubado)" de `sessions_controller_govbr_spec.rb` — mesma razão; são substituídos
  por um exemplo do comportamento atual (usuário da cidade com MFA recebe 201, sem `mfa_required`).

- [ ] **Step 1: Write the failing tests**

Crie `spec/requests/city_session_isolation_spec.rb`:

```ruby
require "rails_helper"

# Spec §5: resolver a cidade ANTES de autenticar é o que torna o cookie de uma
# cidade inútil na vizinha — sem nenhuma verificação de aplicação, só a conexão.
RSpec.describe "City session isolation", type: :request do
  let(:password) { "secret123" }
  let!(:user_a) { User.create!(email_address: "pessoa@cidade.gov.br", password: password) }
  # Mesmo slug de TEST_CITY_B: a requisição ao host de B e o CityConnection.with
  # deste spec compartilham a sessão pinada daquele shard (ver o harness).
  let!(:city_b) do
    create(:city, slug: TEST_CITY_B.slug, status: "active", database_url: city_database_url("rota_saude_test_city_b"))
  end

  def city_b_host = "#{TEST_CITY_B.slug}.rotasaude.app"
  def set_cookie_header = Array(response.headers["Set-Cookie"]).join("\n")

  def login_on_city_a
    post "/session", params: { email_address: user_a.email_address, password: password }
    expect(response).to have_http_status(:created)
    set_cookie_header[/session_id=[^;]+/]
  end

  it "the session cookie is host-only (no Domain attribute)" do
    login_on_city_a

    expect(set_cookie_header).to match(/session_id=/)
    expect(set_cookie_header).not_to match(/domain=/i)
  end

  it "a session created in city A authenticates in A and not in B" do
    cookie = login_on_city_a

    get "/session", headers: { "Cookie" => cookie }
    expect(response).to have_http_status(:ok) # controle positivo: mesma cidade

    get "/session", headers: { "HOST" => city_b_host, "Cookie" => cookie }
    expect(response).to have_http_status(:unauthorized)
  end

  it "the same email existing in city B does not make A's cookie valid there" do
    CityConnection.with(city_b) { User.create!(email_address: user_a.email_address, password: password) }
    cookie = login_on_city_a

    get "/session", headers: { "HOST" => city_b_host, "Cookie" => cookie }

    expect(response).to have_http_status(:unauthorized)
  end

  it "city A's credentials do not log in on city B" do
    post "/session", params: { email_address: user_a.email_address, password: password },
                     headers: { "HOST" => city_b_host }

    expect(response).to have_http_status(:unauthorized)
    expect(JSON.parse(response.body)).to eq("error" => "invalid_credentials")
  end

  it "logging in on the city answers 201 with operator: false, never mfa_required, even with MFA enrolled" do
    Mfa::Enroll.call(user_a)
    user_a.update!(otp_enabled: true)

    post "/session", params: { email_address: user_a.email_address, password: password }

    expect(response).to have_http_status(:created)
    body = JSON.parse(response.body)
    expect(body).to include("operator" => false, "email_address" => user_a.email_address, "mfa_enrolled" => true)
    expect(body).not_to have_key("mfa_required")
  end

  it "the city no longer offers the operator TOTP challenge" do
    post "/session/challenge", params: { session_id: SecureRandom.uuid, code: "123456" }

    expect(response).to have_http_status(:not_found)
  end
end
```

Crie `spec/architecture/cookie_domain_spec.rb`:

```ruby
require "rails_helper"

# Spec §5: o cookie de sessão é host-only — NUNCA `domain:`. Com domain, o cookie
# de uma cidade (ou do console) viajaria para todos os subdomínios. Os request
# specs provam o Set-Cookie em runtime; esta guarda pega a regressão no código,
# inclusive num controller que nenhum spec exercita.
RSpec.describe "Cookie domain guard" do
  def pattern
    /\bdomain:|:domain\s*=>|session_store/
  end

  def offending_lines
    Dir.chdir(Rails.root) do
      %w[app config lib].flat_map { |root| Dir.glob("#{root}/**/*.rb") }.sort.flat_map do |path|
        File.readlines(path, encoding: "UTF-8").each_with_index.filter_map do |line, i|
          next if line.lstrip.start_with?("#")

          "#{path}:#{i + 1}: #{line.strip}" if line.match?(pattern)
        end
      end
    end
  end

  it "no source file sets a cookie Domain or configures a session store" do
    expect(offending_lines).to eq([])
  end

  it "the pattern catches the forms it exists for" do
    [
      'cookies.signed[:session_id] = { value: id, domain: :all }',
      'cookies[:operator_session_id] = { :domain => ".rotasaude.app" }',
      'config.session_store :cookie_store, key: "_rota"'
    ].each { |sample| expect(sample).to match(pattern) }
  end
end
```

Em `spec/controllers/sessions_controller_govbr_spec.rb`, substitua o bloco de comentário e os DOIS contextos
`"operador (operator? stubado) ..."` (do comentário `# No city user can be a platform operator any more` até o fim
do segundo contexto) por:

```ruby
  # Operador de plataforma não loga pela cidade: loga no console, host admin.*
  # (Operators::SessionsController, spec/requests/operators/sessions_spec.rb). O
  # que resta a garantir aqui é que um usuário da cidade com MFA cadastrado NÃO
  # recebe mfa_required no callback — o TOTP da cidade é step-up (publicação),
  # não login.
  context "usuário da cidade com MFA cadastrado" do
    let!(:user) do
      u = User.create!(email_address: "mfa@gov.br", password: SecureRandom.base58(16))
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
      u
    end

    before { allow(Authenticator).to receive(:govbr).and_return(user) }

    it "retorna 201 com a sessão, sem mfa_required" do
      get "/auth/govbr/callback", params: { code: "valid" }

      expect(response).to have_http_status(:created)
      body = JSON.parse(response.body)
      expect(body).not_to have_key("mfa_required")
      expect(body).to include("operator" => false, "mfa_enrolled" => true)
    end
  end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/city_session_isolation_spec.rb spec/architecture/cookie_domain_spec.rb spec/controllers/sessions_controller_govbr_spec.rb`
Expected: FAIL só em "the city no longer offers the operator TOTP challenge" (a rota ainda existe: responde `401
invalid_session`, não `404`). Os demais já passam — são guarda de regressão do que os Planos 1-2 entregaram, e têm de
continuar passando depois do Step 3.

- [ ] **Step 3: Write minimal implementation**

`app/controllers/sessions_controller.rb` inteiro:

```ruby
# Sessões de usuário DA CIDADE — JSON-only (API). Ver ADR-0011.
#
# Roda dentro da conexão da cidade do host (CityResolution): usuário e sessão são
# procurados no banco dessa cidade, então o cookie de uma cidade não autentica
# na vizinha. Operador de plataforma NÃO loga aqui — ver
# Operators::SessionsController, no host admin.*.
#
#   POST   /session   { email_address, password }  → 201 + set-cookie
#   GET    /session                                 → 200 | 401
#   DELETE /session                                 → 204 + clear-cookie
class SessionsController < ApplicationController
  include Authentication

  allow_unauthenticated_access only: %i[create govbr_callback]

  rate_limit to: 10, within: 3.minutes, only: %i[create govbr_callback],
             with: -> { render json: { error: "too_many_requests" }, status: :too_many_requests }

  def create
    user = Authenticator.password(email: params[:email_address], password: params[:password])
    return render(json: { error: "invalid_credentials" }, status: :unauthorized) unless user

    start_new_session_for(user)
    render json: serialize(user), status: :created
  end

  # GET /auth/govbr/callback?code=…&state=…  (ADR-0011 gov.br seam)
  #
  # Provisório: roda na cidade do host, como as demais ações — a identidade gov.br
  # e a sessão são gravadas no banco dessa cidade. O callback único em auth.*,
  # resolvendo a cidade pelo `state` com grant assinado, é do Plano 3B.
  #
  # state opcional aqui — backend não armazena state em sessão (API JSON).
  # Frontend SPA é quem gera/verifica state via storage local + envia ao
  # gov.br. Este endpoint só completa o exchange e cria a sessão.
  def govbr_callback
    user = Authenticator.govbr(code: params[:code])
    return render(json: { error: "govbr_unauthenticated" }, status: :unauthorized) unless user

    start_new_session_for(user)
    render json: serialize(user), status: :created
  rescue Authenticator::GovBr::IntegrationError => e
    Rails.logger.error("[govbr_callback] #{e.class}: #{e.message}")
    render json: { error: "govbr_integration_error" }, status: :bad_gateway
  end

  def destroy
    terminate_session
    head :no_content
  end

  # GET /session — quem está autenticado agora (útil para a UI inicializar).
  def show
    return head :unauthorized unless current_user
    render json: serialize(current_user)
  end

  private

  def serialize(user)
    {
      id: user.id,
      email_address: user.email_address,
      mfa_enrolled: user.mfa_enrolled?,
      # Chave do contrato que dashboard e admin já leem. Usuário de cidade nunca
      # é operador; operador loga no console (Operators::SessionsController).
      operator: false,
      mfa_verified_at: Current.session&.mfa_verified_at&.iso8601,
      memberships: serialize_memberships(user)
    }
  end

  # Memberships ativos na cidade do host. As chaves municipality_* seguem o
  # contrato que dashboard e admin já leem (apps/*/src/lib/api.ts), mas os
  # valores vêm da cidade resolvida — a chave de id carrega o slug. Renomear o
  # contrato é dos frontends (Plano 6).
  def serialize_memberships(user)
    city = Current.city
    user.memberships.active.map do |m|
      {
        municipality_id: city.slug,
        municipality_name: city.name,
        municipality_uf: city.uf,
        role: m.role
      }
    end
  end
end
```

`config/routes.rb` — apague a linha da cidade (a do console, dentro do `constraints(PlatformConsoleHost)`, fica):

```ruby
  post "/session/challenge", to: "sessions#challenge_totp"
```

e troque o comentário `# Sessão de admin (ADR-0011).` acima de `resource :session` por
`# Sessão de usuário da cidade (ADR-0011). Operador: bloco do console, acima.`

Apague `spec/controllers/sessions_controller_mfa_spec.rb`:

```bash
git -C apps/api rm spec/controllers/sessions_controller_mfa_spec.rb
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/city_session_isolation_spec.rb spec/architecture/cookie_domain_spec.rb spec/controllers spec/requests/operators`
Expected: PASS, 0 failures.

- [ ] **Step 5: Confirm no operator branch is left in city controllers**

Run: `grep -rn "operator?" apps/api/app/controllers`
Expected: uma única linha — `app/controllers/setup_controller.rb` (`deactivate_user`, 403 até o Plano 3B).

- [ ] **Step 6: Re-point the stale "Plano 3" references**

O que este plano entrega deixa de ser "Plano 3" nos comentários. Edite cada linha para o dono real, sem mudar
código. A tabela vem de `grep -rnE "Plano 3([^B]|$)" app lib config db spec` em `b4cac07`:

| Arquivo:linha (em `b4cac07`) | Novo texto aponta para |
|---|---|
| `app/auth/authenticator/gov_br.rb:111` | callback único em auth.* com grant assinado → **Plano 3B** |
| `app/models/platform_event.rb:2` e `app/events/platform.rb:4` | operadores já auditam aqui (`operator.login`, Plano 3) — reescrever no presente, sem número de plano |
| `app/models/user.rb:43` | `SetupController#deactivate_user` falha fechado até o grant de operador → **Plano 3B** |
| `app/commands/provision_municipality.rb:15` | operador convidando por grant → **Plano 3B** |
| `app/commands/municipality_channels/rotate_token.rb:4` | operador com grant de entrada na cidade → **Plano 3B** |
| `app/controllers/setup_controller.rb:10,13,23,31` | grant de operador → **Plano 3B**; provisionamento → **Plano 4** (mantém) |
| `app/controllers/admin/api/base_controller.rb:59` | renomear contrato dos frontends → **Plano 6** |
| `app/channels/application_cable/connection.rb:7` | quem montar ActionCable decide → **Plano 6** |
| `lib/tasks/channels.rake:1` | ator deveria ser Operator via grant → **Plano 3B** |
| `spec/requests/admin/api/reports_spec.rb:28` e `spec/commands/municipality_channels/rotate_token_spec.rb:43` | texto do `skip`: `"Plano 3B: grant de operador — ..."` (resto do texto igual) |

`db/seeds.rb`, `db/seeds/dashboard_demo.rb` e o `pending` de `reports_spec.rb:67-71` saem nas Tasks 4, 5 e 1.

Run: `grep -rnE "Plano 3([^B]|$)" apps/api/app apps/api/lib apps/api/config apps/api/spec`
Expected: vazio (`db/` só fica vazio depois das Tasks 4 e 5).

- [ ] **Step 7: Run the whole suite**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 2 pending (os dois `skip` "Plano 3B: grant de operador").

- [ ] **Step 8: Commit**

```bash
git -C apps/api add app/controllers/sessions_controller.rb config/routes.rb spec/requests/city_session_isolation_spec.rb spec/architecture/cookie_domain_spec.rb spec/controllers/sessions_controller_govbr_spec.rb app/auth/authenticator/gov_br.rb app/models/platform_event.rb app/events/platform.rb app/models/user.rb app/commands/provision_municipality.rb app/commands/municipality_channels/rotate_token.rb app/controllers/setup_controller.rb app/controllers/admin/api/base_controller.rb app/channels/application_cable/connection.rb lib/tasks/channels.rake spec/requests/admin/api/reports_spec.rb spec/commands/municipality_channels/rotate_token_spec.rb
git -C apps/api commit -m "Drop the dead operator branch from city login and guard session isolation

City users are never operators; operators log in on the platform console.
Remove the operator branches and the TOTP challenge from the city
SessionsController, prove that a session cookie from one city does not
authenticate in another, and guard against a cookie Domain attribute.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: Baseline de duas cidades em dev

**Files:**
- Modify: `lib/tasks/city.rake` (tasks `city:dev_up` e `city:dev_baseline`)
- Modify: `spec/tasks/city_rake_spec.rb`
- Modify: `db/seeds.rb` (duas cidades)
- Modify: `README.md` (seção "Ambiente de desenvolvimento (monorepo)")
- Modify, **fora de git**: `start.sh` na raiz do monorepo

**Interfaces:**
- Consumes: `CityProvisioner.call(slug:, name:, uf:) → Result` (payload `city:`), a lambda `load_city_schema` já
  definida em `lib/tasks/city.rake`, `CityCatalog.reset_cache!`, `City#servable?`, `GenerateReportJob#handle`.
- Produces:
  - `rails 'city:dev_up[slug,nome,uf]'` — só em development; registra a cidade (se faltar), cria o banco
    `rota_saude_city_<slug>` (se faltar), carrega o schema de cidade SÓ se o banco ainda não o tiver, e ativa.
    Idempotente.
  - `rails city:dev_baseline` — `city:dev_up` para `curitiba` (Curitiba/PR) e `maringa` (Maringá/PR).
  - `db:seed` semeando as duas cidades, com e-mails `admin@<slug>.demo`, DDD 41 e 44, canal
    `PNID-<SLUG>-DEV`.

Por que "só se o banco não tiver o schema": o dump de cidade usa `force: :cascade`. Recarregar num banco com dados
apagaria tudo. A checagem é `to_regclass('public.users') IS NULL`; se ela própria falhar, a task aborta em vez de
carregar às cegas.

`.localhost` já é host permitido em development (verificado: `Rails.application.config.hosts` inclui `".localhost"`),
então nada muda em `config/environments/development.rb`.

- [ ] **Step 1: Write the failing test**

Em `spec/tasks/city_rake_spec.rb`, acrescente ao fim do arquivo:

```ruby
# city:dev_up cria banco e carrega schema com credencial de superusuário de
# bootstrap (CityProvisioner#database_url_for) — mesma razão de city:create para
# nunca rodar fora de development. Em test ela precisa abortar ANTES de tocar o
# catálogo.
RSpec.describe "city:dev_up and city:dev_baseline rake tasks" do
  before(:all) do
    Rails.application.load_tasks unless Rake::Task.task_defined?("city:dev_up")
  end

  before do
    %w[city:dev_up city:dev_baseline].each { |name| Rake::Task[name].reenable }
  end

  def invoke_silently(name, *args)
    original_stderr, $stderr = $stderr, StringIO.new
    Rake::Task[name].invoke(*args)
  ensure
    $stderr = original_stderr
  end

  it "city:dev_up aborts outside development, before touching the catalog" do
    slug = "naosobe#{SecureRandom.hex(3)}"

    expect { invoke_silently("city:dev_up", slug, "Nao Sobe", "SP") }.to raise_error(SystemExit)
    expect(City.where(slug: slug)).to be_empty
  end

  it "city:dev_baseline aborts outside development, before touching the catalog" do
    expect { invoke_silently("city:dev_baseline") }.to raise_error(SystemExit)
    expect(City.where(slug: %w[curitiba maringa])).to be_empty
  end
end
```

Atenção ao `load_tasks` duplicado: o primeiro `describe` do arquivo checa `city:create`. Se ele já carregou as tasks,
`city:dev_up` estará definida e este `before(:all)` não recarrega — é o que evita ações de task registradas duas vezes.

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/tasks/city_rake_spec.rb`
Expected: FAIL — `Don't know how to build task 'city:dev_up'`.

- [ ] **Step 3: Write minimal implementation**

Em `lib/tasks/city.rake`, dentro de `namespace :city do`, depois da task `:create` (as lambdas
`load_city_schema` e as variáveis locais já estão definidas acima), acrescente:

```ruby
  # Cidades de desenvolvimento: duas, para o isolamento ser exercitável fora da
  # suíte (decisão do Plano 2). [slug, nome, uf].
  dev_cities = [ %w[curitiba Curitiba PR], [ "maringa", "Maringá", "PR" ] ].freeze

  desc "Dev: registra, cria o banco, carrega o schema e ativa uma cidade (idempotente). Uso: city:dev_up[slug,nome,uf]"
  task :dev_up, %i[slug name uf] => :environment do |_t, args|
    # Mesma razão de city:create: a database_url usa credencial de superusuário de
    # bootstrap. Provisionamento real (rota_provisioner) é o Plano 4.
    abort "[city:dev_up] só roda em development." unless Rails.env.development?
    abort "uso: rails 'city:dev_up[slug,nome,uf]'" if args[:slug].blank? || args[:name].blank?

    result = CityProvisioner.call(slug: args[:slug], name: args[:name], uf: args[:uf])
    abort "[city:dev_up] falhou: #{result.message}" if result.failure?
    city = result.payload[:city]

    database = ActiveRecord::Base.configurations.resolve(city.database_url).database.to_s
    abort "[city:dev_up] #{city.slug}: database_url sem database" if database.empty?

    su   = ENV.fetch("BOOTSTRAP_SUPERUSER", "rota_saude")
    pwd  = ENV.fetch("POSTGRES_PASSWORD") { abort "[city:dev_up] POSTGRES_PASSWORD ausente." }
    host = ENV.fetch("DATABASE_HOST", "127.0.0.1")
    port = ENV.fetch("DATABASE_PORT", "5432").to_s
    env  = { "PGPASSWORD" => pwd }
    base = [ "psql", "-h", host, "-p", port, "-U", su, "-v", "ON_ERROR_STOP=1", "-tA" ]

    exists, st = Open3.capture2e(env, *base, "-d", "postgres",
                                 "-c", "SELECT 1 FROM pg_database WHERE datname='#{database}'")
    abort "[city:dev_up] não consegui consultar pg_database:\n#{exists}" unless st.success?
    if exists.strip == "1"
      puts "[city:dev_up] #{database} já existe"
    else
      # Identificador entre aspas: slug pode ter hífen (rótulo DNS).
      out, st = Open3.capture2e(env, *base, "-d", "postgres", "-c", %(CREATE DATABASE "#{database}" OWNER #{su}))
      abort "[city:dev_up] falha ao criar #{database}:\n#{out}" unless st.success?
      puts "[city:dev_up] #{database} criado"
    end

    # O dump de cidade usa force: :cascade — só carrega num banco SEM o schema.
    # Se a checagem falhar, aborta: carregar às cegas apagaria dados.
    empty, st = Open3.capture2e(env, *base, "-d", database, "-c", "SELECT to_regclass('public.users') IS NULL")
    abort "[city:dev_up] não consegui checar o schema de #{database}:\n#{empty}" unless st.success?
    if empty.strip == "t"
      load_city_schema.call(database)
      puts "[city:dev_up] schema de cidade carregado em #{database}"
    else
      puts "[city:dev_up] #{database} já tem o schema de cidade — nada carregado"
    end

    city.update!(status: "active") unless city.status == "active"
    CityCatalog.reset_cache!
    puts "[city:dev_up] #{city.slug} → #{city.status} (#{database})"
  end

  desc "Dev: sobe as cidades de desenvolvimento (curitiba, maringa). Idempotente."
  task dev_baseline: :environment do
    abort "[city:dev_baseline] só roda em development." unless Rails.env.development?

    dev_cities.each do |slug, name, uf|
      Rake::Task["city:dev_up"].reenable
      Rake::Task["city:dev_up"].invoke(slug, name, uf)
    end
  end
```

- [ ] **Step 4: Run test to verify it passes**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/tasks/city_rake_spec.rb`
Expected: PASS, 0 failures.

- [ ] **Step 5: Rewrite db/seeds.rb for both cities**

Substitua `db/seeds.rb` inteiro:

```ruby
# Seeds de desenvolvimento. Idempotente: `bin/rails db:seed` pode rodar N vezes.
#
#   - PLATAFORMA: operador dev@local / dev-password (Operator + MFA, otp_secret
#     fixo) — loga no console, host admin.* (Operators::SessionsController); e um
#     canal WhatsApp por cidade (CityChannel).
#   - CADA CIDADE (curitiba, maringa), dentro da conexão dela: admin@<slug>.demo /
#     dev-password como municipal_admin, um AlertRecipient de e-mail ativo,
#     protocolo ATIVO (triage-respiratoria), uma triagem completa e o relatório.
#     DDD, telefones, e-mails e canal diferem por cidade, para o isolamento ficar
#     visível fora da suíte.
#
# Pré-requisito: as cidades ativas no catálogo, com banco e schema —
# `bin/rails city:dev_baseline` (o start.sh já roda). Cidade ausente ou inativa é
# pulada com aviso.
#
# Operador exige MFA/TOTP a cada login (ADR-0011). Em dev usamos um `otp_secret`
# FIXO (override por env) para que a entrada no seu autenticador continue válida
# após cada reset — caso contrário você teria que re-enrolar toda vez.
#
# NUNCA roda em produção — senhas e segredo fixos são só para ambiente local.
if Rails.env.production?
  warn "[seeds] pulando: seeds de dev não rodam em produção"
else
  password = ENV.fetch("DEV_USER_PASSWORD", "dev-password")

  # ── Operador de plataforma + MFA ──────────────────────────────────────────────
  # otp_secret fixo (dev) para o autenticador sobreviver a resets. Só é setado
  # quando o operador ainda não tem MFA (não clobbera um segredo já existente).
  operator = Operator.find_or_initialize_by(email_address: "dev@local")
  operator.password = password
  unless operator.otp_enabled? && operator.otp_secret.present?
    operator.otp_secret  = ENV.fetch("DEV_OPERATOR_OTP_SECRET", "TQLRHWIAKEISPIW6YY3IAKGCLVNPF4EV")
    operator.otp_enabled = true
  end
  operator.save!
  puts "[seeds] operador .... #{operator.email_address} / #{password} + MFA (otp_secret fixo) → console admin.*"

  protocol_defn = {
    "name" => "triage-respiratoria", "version" => 1, "start_step_id" => "tosse",
    "steps" => [
      { "id" => "tosse", "prompt" => "Você está com tosse?", "answer_type" => "boolean",
        "branches" => { "true" => "febre", "false" => nil }, "weights" => { "true" => 3, "false" => 0 } },
      { "id" => "febre", "prompt" => "Está com febre alta?", "answer_type" => "boolean",
        "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 5, "false" => 0 } }
    ],
    "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "alta" => 5 },
                   "priority_map" => { "baixa" => 9, "alta" => 1 } },
    "recommendations" => {
      "alta"  => { "title" => "Procure atendimento hoje",
                   "body" => "Prioridade alta. Vá à UPA/unidade mais próxima ainda hoje. Falta de ar, dor no peito ou lábios roxos → 192." },
      "baixa" => { "title" => "Cuidados em casa",
                   "body" => "Repouso e hidratação. Se piorar ou persistir por mais de 3 dias, procure sua unidade de saúde." }
    }
  }

  { "curitiba" => "41", "maringa" => "44" }.each do |slug, ddd|
    city = City.find_by(slug: slug)
    if city.nil? || !city.servable?
      warn "[seeds] cidade '#{slug}' ausente ou não ativa no catálogo — pulada (rode bin/rails city:dev_baseline)"
      next
    end

    # ── Canal WhatsApp (plataforma) ─────────────────────────────────────────────
    tag = slug.upcase
    channel = CityChannel.find_or_create_by!(phone_number_id: "PNID-#{tag}-DEV") do |c|
      c.city                 = city
      c.waba_id              = "WABA-#{tag}-DEV"
      c.display_phone_number = "+55#{ddd}999990000"
      c.access_token         = "DEV-WHATSAPP-TOKEN-#{tag}"
      c.active               = true
    end

    Current.set(city: city) do
      CityConnection.with(city) do
        # ── Usuário municipal (Dashboard) ─────────────────────────────────────
        muni_admin = User.find_or_initialize_by(email_address: "admin@#{slug}.demo")
        muni_admin.password = password
        muni_admin.save!
        Membership.find_or_create_by!(user: muni_admin, role: "municipal_admin") do |m|
          m.granted_at = Time.current
        end

        # ── Destinatário de alerta urgente (R37) ──────────────────────────────
        # DispatchMunicipalityAlertJob entrega ao primeiro AlertRecipient de e-mail
        # ativo da cidade e levanta NoAlertRecipient sem nenhum. Um por cidade, no
        # banco dela; o city_profile (Plano 4) substitui.
        alert_recipient = AlertRecipient.find_or_initialize_by(
          channel: "email", destination: ENV.fetch("DEV_ALERT_EMAIL", "alertas@#{slug}.demo")
        )
        alert_recipient.active = true
        alert_recipient.save!

        # ── Demo ponta-a-ponta ────────────────────────────────────────────────
        protocol = ProtocolDefinition.find_or_create_by!(name: "triage-respiratoria", version: 1) do |p|
          p.status     = "active"
          p.definition = protocol_defn
        end

        # state "consented": a pessoa consentiu e concluiu a triagem — é o estado
        # que os painéis live/funil de Conversas contam. created_at ~5 min antes
        # de completed_at para o KPI avgToCompleteMin exibir uma duração realista.
        convo  = Conversation.find_or_create_by!(phone: "+55#{ddd}999990001") { |c| c.state = "consented" }
        triage = Triage.where(conversation_id: convo.id, protocol_definition_id: protocol.id).first
        triage ||= Triage.create!(
          conversation: convo, protocol_definition: protocol, protocol_name: "triage-respiratoria",
          status: "completed", tier: "alta", priority: 1,
          created_at: 5.minutes.ago, completed_at: Time.current,
          answers: { "tosse" => "true", "febre" => "true" },
          outcome: { "status" => "terminal", "tier" => "alta", "priority" => 1,
                     "trail" => [ { "step" => "tosse", "answer" => "true" }, { "step" => "febre", "answer" => "true" } ] }
        )
        GenerateReportJob.new.handle(triage_id: triage.id, status: "terminal", tier: "alta", priority: 1)
        report = ReportSnapshot.find_by(triage_id: triage.id)

        puts "[seeds] cidade ...... #{city.name} (#{city.slug}/#{city.uf}, #{city.status})"
        puts "  municipal ... #{muni_admin.email_address} / #{password}  → dashboard"
        puts "  alerta ...... #{alert_recipient.destination} (email, active)"
        puts "  canal ....... #{channel.phone_number_id} (active)"
        puts "  protocolo ... #{protocol.name} v#{protocol.version} (#{protocol.status})"
        puts "  relatório ... #{report&.url}"
      end
    end
  end
end

# Dataset opcional e pesado dos painéis (todas as cidades de dev). Fora por
# padrão; o seed base fica enxuto. Ligue com SEED_DASHBOARD_DEMO=1 bin/rails db:seed
# (ou bin/rails db:seed:demo). Ver lib/dashboard_demo.rb.
if ENV["SEED_DASHBOARD_DEMO"] == "1" && !Rails.env.production?
  load Rails.root.join("db/seeds/dashboard_demo.rb")
end
```

- [ ] **Step 6: Bring the two cities up in the development database and prove isolation**

Estes comandos agem no banco de DESENVOLVIMENTO. Eles só criam o que falta: bancos `rota_saude_city_curitiba` e
`rota_saude_city_maringa`, schema num banco vazio, linhas de seed idempotentes. Nenhum drop.

```bash
docker compose exec -T api bin/rails city:dev_baseline
```
Expected: para cada cidade, `criado` (ou `já existe`), `schema de cidade carregado` (ou `já tem o schema`), e
`curitiba → active` / `maringa → active`. Rodar de novo: nenhum `criado`, nenhum `carregado`.

```bash
docker compose exec -T api bin/rails db:seed
```
Expected: bloco `[seeds] cidade ...... Curitiba (curitiba/PR, active)` e outro `Maringá (maringa/PR, active)`.

Login na cidade A e uso do cookie nas duas cidades (a API escuta em `localhost:3030`; o `Host` escolhe a cidade):

```bash
curl -s -o /tmp/rs-cwb-login.json -D /tmp/rs-cwb-headers.txt -H "Host: curitiba.localhost" -H "Content-Type: application/json" -d '{"email_address":"admin@curitiba.demo","password":"dev-password"}' http://localhost:3030/session
```
Expected: `/tmp/rs-cwb-login.json` com `"municipality_name":"Curitiba"` e `"operator":false`; o cabeçalho
`Set-Cookie: session_id=...` sem `domain=`.

```bash
COOKIE=$(grep -o 'session_id=[^;]*' /tmp/rs-cwb-headers.txt); curl -s -o /dev/null -w "%{http_code}\n" -H "Host: curitiba.localhost" -H "Cookie: $COOKIE" "http://localhost:3030/admin/api/reports?period=30d"; curl -s -o /dev/null -w "%{http_code}\n" -H "Host: maringa.localhost" -H "Cookie: $COOKIE" "http://localhost:3030/admin/api/reports?period=30d"
```
Expected: `200` e depois `401`.

```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: maringa.localhost" -H "Content-Type: application/json" -d '{"email_address":"admin@curitiba.demo","password":"dev-password"}' http://localhost:3030/session
```
Expected: `401`.

Login do operador no console (três comandos: senha, código TOTP, challenge):

```bash
curl -s -o /tmp/rs-op-login.json -D /tmp/rs-op-headers.txt -H "Host: admin.localhost" -H "Content-Type: application/json" -d '{"email_address":"dev@local","password":"dev-password"}' http://localhost:3030/session; cat /tmp/rs-op-login.json
```
Expected: `{"mfa_required":true,"session_id":"..."}` e `Set-Cookie: operator_session_id=...` sem `domain=`.

```bash
docker compose exec -T api bin/rails runner 'require "rotp"; puts ROTP::TOTP.new(Operator.find_by!(email_address: "dev@local").otp_secret).now' | tail -1 > /tmp/rs-op-code.txt
```
Expected: `/tmp/rs-op-code.txt` com 6 dígitos. (Este `rails runner` só lê.)

```bash
SID=$(ruby -rjson -e 'puts JSON.parse(File.read("/tmp/rs-op-login.json"))["session_id"]'); COOKIE=$(grep -o 'operator_session_id=[^;]*' /tmp/rs-op-headers.txt); curl -s -H "Host: admin.localhost" -H "Content-Type: application/json" -H "Cookie: $COOKIE" -d "{\"session_id\":\"$SID\",\"code\":\"$(cat /tmp/rs-op-code.txt)\"}" http://localhost:3030/session/challenge
```
Expected: JSON com `"operator":true` e `"email_address":"dev@local"` (rode em até 30 s depois de gerar o código).

Cole as saídas no relatório da task.

- [ ] **Step 7: Update start.sh (outside git) and the README**

`start.sh` na raiz do monorepo não está em nenhum repositório: edite, valide e NÃO tente commitar.

No bloco `# --- bancos de plataforma e "nenhuma cidade selecionada"`, acrescente como última linha, depois de
`city:test_databases`:

```bash
docker compose run --rm --no-deps api ./bin/rails city:dev_baseline
```

e, no comentário desse bloco, troque a linha `# fechado. city:test_databases também cria os bancos de cidade da suíte.`
por:

```bash
# fechado. city:test_databases cria os bancos de cidade da suíte; city:dev_baseline
# cria, carrega e ativa as duas cidades de dev (curitiba, maringa).
```

No bloco `if [ "$RESET_DB" = "1" ]; then`, depois de `dropdb --if-exists "$DB_TEST"`, acrescente:

```bash
  dropdb --if-exists rota_saude_city_curitiba
  dropdb --if-exists rota_saude_city_maringa
```

(`city:dev_up` recria o banco e recarrega o schema; a linha do catálogo continua e é reaproveitada.)

Run: `bash -n start.sh`
Expected: sem saída.

No `README.md` do api, seção "Ambiente de desenvolvimento (monorepo)":
- na tabela de bancos, acrescente a linha
  `| \`rota_saude_city_curitiba\`, \`rota_saude_city_maringa\` | \`rota_saude\` | \`rails city:dev_baseline\` |`;
- troque o parágrafo que começa em `` `start.sh` chama as três tasks antes do `db:seed`. `` por:

```markdown
`start.sh` chama essas tasks e `city:dev_baseline` antes do `db:seed`. Contas de dev:
`admin@curitiba.demo` e `admin@maringa.demo` (senha `dev-password`) em cada cidade, e o operador `dev@local`
(mesma senha + TOTP) no console. Hosts: `curitiba.localhost`, `maringa.localhost`, `admin.localhost`.
No navegador os frontends ainda não resolvem cidade — o proxy do Vite troca o Host por `api:3000` até o Plano 6;
para exercitar hoje, use `curl -H "Host: curitiba.localhost" http://localhost:3030/...`.
```

- [ ] **Step 8: Run the whole suite**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 2 pending.

- [ ] **Step 9: Commit**

```bash
git -C apps/api add lib/tasks/city.rake spec/tasks/city_rake_spec.rb db/seeds.rb README.md
git -C apps/api commit -m "Bring up two development cities with their own databases

city:dev_up registers a city, creates its database, loads the city schema
only into a database that does not have it yet, and activates it;
city:dev_baseline does that for curitiba and maringa. The seed now fills
both cities with distinct data inside each city's connection.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 5: Dataset de demonstração por cidade

**Files:**
- Create: `lib/dashboard_demo.rb` (o módulo, reescrito sem município)
- Modify: `db/seeds/dashboard_demo.rb` (vira só carregador)
- Modify: `lib/tasks/seed_demo.rake` (verificação delega ao módulo)
- Create: `spec/lib/dashboard_demo_spec.rb`

**Interfaces:**
- Consumes: `City#servable?`, `CityConnection.with(city)`, `Admin::Api::Period.parse(key:, from:, to:, tz:)`,
  `Admin::ConversationsQuery.call(period:)`, `Admin::ConsentQuery.call(period:)`, `Admin::IngestionQuery.call(period:)`,
  `Admin::TriagesQuery.call(period:)`, `Admin::ClassificationQuery.call(period:)`, `Admin::ReportsQuery.call(period:)`,
  `Admin::ProtocolsQuery.index`, `Admin::EventsQuery.call(name:, from:, to:, period:)`, `ReportSnapshot.sign(token)`.
- Produces:
  - `DashboardDemo::CITIES` — `[{ slug: "curitiba", scale: 1.0, code: "CWB", ddd: "41" }, { slug: "maringa", scale: 0.4, code: "MGA", ddd: "44" }]`.
  - `DashboardDemo.seed_current_city(cfg) → Hash` — semeia a cidade da conexão CORRENTE; devolve contagens.
  - `DashboardDemo.verify_current_city(slug) → Array<String>` — falhas (vazio = ok) na conexão corrente.
  - `DashboardDemo.run! → Array` — cada cidade de `CITIES` ativa, na conexão dela.
  - `DashboardDemo.verify! → Array<String>` — cidade ausente/inativa vira a falha
    `"<slug>: city missing or not active in the catalog"`.

O conteúdo do dataset (distribuições, ciclos de tier/status/ack, eventos de protocolo e de trilha) NÃO muda: só saem
`Municipality`, `municipality_id` e a conexão `role: :admin`. Se o spec do Step 1 acusar um painel vazio por causa de
drift entre dataset e query anterior ao corte (confira com `git -C apps/api show c61a3da:db/seeds/dashboard_demo.rb`),
ajuste o DATASET para a query atual e registre no relatório — nunca afrouxe a checagem.

- [ ] **Step 1: Write the failing test**

Crie `spec/lib/dashboard_demo_spec.rb`:

```ruby
require "rails_helper"
require Rails.root.join("lib/dashboard_demo").to_s

# O dataset de demonstração precisa popular exatamente o que db:seed:demo:verify
# confere, DENTRO da conexão da cidade corrente e sem coluna de município. Roda na
# conexão padrão da suíte (TEST_CITY_A).
RSpec.describe DashboardDemo do
  it "populates every panel the verify task checks, in the current city" do
    cfg = described_class::CITIES.first

    described_class.seed_current_city(cfg)

    expect(described_class.verify_current_city(cfg[:slug])).to eq([])
  end

  it "is idempotent: a second run adds nothing" do
    cfg = described_class::CITIES.last

    first = described_class.seed_current_city(cfg)

    expect(described_class.seed_current_city(cfg)).to eq(first)
  end

  it "writes only into the connected city" do
    city_b = create(:city, slug: TEST_CITY_B.slug, database_url: city_database_url("rota_saude_test_city_b"))

    described_class.seed_current_city(described_class::CITIES.first)

    expect(CityConnection.with(city_b) { [ Conversation.count, DomainEvent.count, ReportSnapshot.count ] })
      .to eq([ 0, 0, 0 ])
  end

  it "verify! reports a dev city missing from the catalog instead of raising" do
    expect(described_class.verify!).to include("curitiba: city missing or not active in the catalog",
                                               "maringa: city missing or not active in the catalog")
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/lib/dashboard_demo_spec.rb`
Expected: FAIL — `cannot load such file -- /rails/lib/dashboard_demo`.

- [ ] **Step 3: Write the module**

Crie `lib/dashboard_demo.rb`:

```ruby
# Dataset de demonstração dos painéis do dashboard (opt-in): popula cada visão
# (Aquisição/Triagem/Governança) de cada cidade de dev, de forma idempotente,
# espalhado pelos últimos 30 dias.
#
# Por cidade: tudo roda dentro da conexão da cidade (CityConnection.with) — não há
# coluna de município nem conexão privilegiada. Chaves naturais determinísticas
# (message_id, idempotency_key, payload.demo_id, token) tornam a re-execução um
# no-op.
#
# Rodar:     bin/rails db:seed:demo          (pré-requisito: bin/rails city:dev_baseline)
# Verificar: bin/rails db:seed:demo:verify
module DashboardDemo
  module_function

  CITIES = [
    { slug: "curitiba", scale: 1.0, code: "CWB", ddd: "41" },
    { slug: "maringa",  scale: 0.4, code: "MGA", ddd: "44" }
  ].freeze

  # Distribuição de estados da FSM (base Curitiba; escalada por cidade). Cobre
  # todo bucket do funil (greeting/awaiting_consent/consented), o live
  # (awaiting_consent + consented) e as saídas (revoked). "completed" é terminal e
  # fica fora do funil por desenho.
  STATE_DIST = { "greeting" => 4, "awaiting_consent" => 4, "consented" => 12,
                 "completed" => 8, "revoked" => 3, "abandoned" => 3 }.freeze

  # Status de outbound é inteiro literal 0–5: {0,1,2}=ok, {3}=warn, {4,5}=err
  # (ingestion_query).
  ACK_STATUS_CYCLE = [ 0, 1, 2, 0, 1, 2, 3, 4, 5, 2, 0, 3, 4 ].freeze

  TIER_CYCLE = %w[low medium high medium low high medium high].freeze
  # Prioridade na faixa do contrato (1..9, menor = mais urgente).
  SEED_PRIORITY = { "high" => 1, "medium" => 5, "low" => 9 }.freeze
  MODE_CYCLE = %w[weighted weighted decision_table].freeze
  # Conversas consentidas só têm triagem completed ou in_progress; os in_progress
  # mantêm a taxa de conclusão abaixo de 100%.
  TRIAGE_STATUS_CYCLE = %w[completed completed completed completed in_progress completed in_progress completed].freeze

  # Timestamp determinístico `days` atrás, em hora/minuto fixos.
  def at_days_ago(days, hour: 10)
    (Time.current - days.to_i.days).change(hour: hour, min: (days.to_i * 7) % 60, sec: 0)
  end

  # Índice i em [0, n) → deslocamento em dias em [0, 29].
  def spread_days(i, n)
    n <= 1 ? 0 : ((i * 29.0) / (n - 1)).round
  end

  def scaled(base, cfg)
    [ (base * cfg[:scale]).round, 1 ].max
  end

  def actor(cfg, who)
    "#{who}@#{cfg[:slug]}.demo"
  end

  # Definição válida (passa Protocols::Validator), mesmo formato do seed base.
  def demo_definition(name, version)
    {
      "name" => name, "version" => version, "start_step_id" => "tosse",
      "steps" => [
        { "id" => "tosse", "prompt" => "Você está com tosse?", "answer_type" => "boolean",
          "branches" => { "true" => "febre", "false" => nil }, "weights" => { "true" => 3, "false" => 0 } },
        { "id" => "febre", "prompt" => "Está com febre alta?", "answer_type" => "boolean",
          "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 5, "false" => 0 } }
      ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "alta" => 5 },
                     "priority_map" => { "baixa" => 9, "alta" => 1 } },
      "recommendations" => {
        "alta"  => { "title" => "Procure atendimento hoje", "body" => "Prioridade alta. Procure a unidade mais próxima." },
        "baixa" => { "title" => "Cuidados em casa", "body" => "Repouso e hidratação; se piorar, procure sua unidade." }
      }
    }
  end

  def upsert_protocol(name, version, status)
    ProtocolDefinition.find_or_create_by!(name: name, version: version) do |p|
      p.status = status
      p.definition = demo_definition(name, version)
    end
  end

  # Várias versões/status para o painel Protocolos. v1 ativa reaproveita a linha
  # do seed base, se existir.
  def build_protocols
    resp1 = upsert_protocol("triage-respiratoria", 1, "active")
    upsert_protocol("triage-respiratoria", 2, "published")
    upsert_protocol("triage-respiratoria", 3, "draft")
    dengue1 = upsert_protocol("triagem-dengue", 1, "active")
    upsert_protocol("triagem-dengue", 2, "retired")
    { "triage-respiratoria" => resp1, "triagem-dengue" => dengue1 }
  end

  def build_conversations_and_consents(cfg)
    triage_convos = []
    seq = 0
    STATE_DIST.each do |state, base|
      scaled(base, cfg).times do
        seq += 1
        phone = format("+55%s9%08d", cfg[:ddd], seq)
        # Espalhamento coprimo sobre [0,29]: intercala estados pela janela de 30d.
        days  = (seq * 13) % 30
        convo = Conversation.find_or_create_by!(phone: phone) do |c|
          c.state = state
          c.created_at = at_days_ago(days, hour: 9)
        end

        if %w[consented completed revoked].include?(state)
          version = seq.even? ? 2 : 1
          Consent.where(conversation_id: convo.id).first || Consent.create!(
            conversation: convo, version: version,
            channel: "whatsapp", policy_text_sha: "demo-policy-sha-v#{version}",
            given_at: at_days_ago(days, hour: 11),
            revoked_at: (state == "revoked" ? at_days_ago([ days - 1, 0 ].max, hour: 12) : nil)
          )
        end

        triage_convos << { convo: convo, state: state, days: days, seq: seq } if %w[consented completed].include?(state)
      end
    end
    triage_convos
  end

  # Inbound em 30d (dia 0 → <24h pendente; dias ≥ 2 → backlog >24h) e log de ack.
  def build_ingestion(cfg)
    n_in = scaled(40, cfg)
    n_in.times do |i|
      InboundMessage.find_or_create_by!(message_id: "IN-#{cfg[:code]}-#{format('%04d', i + 1)}") do |m|
        m.from = format("+55%s9%08d", cfg[:ddd], i + 1)
        m.kind = "text"
        m.raw = "demo inbound message #{i + 1}"
        m.created_at = at_days_ago(spread_days(i, n_in), hour: (i % 12) + 8)
      end
    end

    n_out = scaled(30, cfg)
    n_out.times do |i|
      OutboundMessage.find_or_create_by!(idempotency_key: "OUT-#{cfg[:code]}-#{format('%04d', i + 1)}") do |m|
        m.to = format("+55%s9%08d", cfg[:ddd], i + 1)
        m.status = ACK_STATUS_CYCLE[i % ACK_STATUS_CYCLE.length]
        m.template = { "name" => "rota_saude_ask", "language" => { "code" => "pt_BR" } }
        m.created_at = at_days_ago(spread_days(i, n_out), hour: (i % 12) + 8)
      end
    end
  end

  def build_triages_and_reports(cfg, citizens, protocols)
    protos = protocols.values
    built = []
    citizens.each_with_index do |c, i|
      convo = c[:convo]
      days  = c[:days]
      proto = protos[i % protos.size]
      status = c[:state] == "completed" ? "completed" : TRIAGE_STATUS_CYCLE[i % TRIAGE_STATUS_CYCLE.size]
      completed = status == "completed"
      tier = TIER_CYCLE[i % TIER_CYCLE.size]
      mode = MODE_CYCLE[i % MODE_CYCLE.size]
      priority = SEED_PRIORITY.fetch(tier)
      started_at = at_days_ago(days, hour: 9)
      completed_at = completed ? started_at + (3 + (i % 8)).minutes : nil

      triage = Triage.where(conversation_id: convo.id, protocol_definition_id: proto.id).first
      triage ||= Triage.create!(
        conversation: convo, protocol_definition: proto, protocol_name: proto.name,
        status: status,
        tier: (completed ? tier : nil), priority: (completed ? priority : nil),
        current_step: "febre",
        answers: { "tosse" => "true", "febre" => (tier == "high" ? "true" : "false") },
        created_at: started_at, completed_at: completed_at,
        outcome: (completed ? {
          "status" => "terminal", "tier" => tier, "priority" => priority,
          "scoring" => { "mode" => mode, "score" => (tier == "high" ? 8 : tier == "medium" ? 4 : 1) },
          "trail" => [ { "step" => "tosse", "answer" => "true" },
                       { "step" => "febre", "answer" => (tier == "high" ? "true" : "false") } ]
        } : {})
      )

      built << { triage: triage, tier: tier, mode: mode, priority: priority, days: days, completed: completed }

      next unless completed

      expired = (i % 4).zero?
      upsert_report(cfg, triage, tier,
                    created_at: completed_at,
                    expires_at: (expired ? at_days_ago(days + 2, hour: 9) : Time.current + 20.days))
    end
    built
  end

  # Snapshot montado direto (não via GenerateReportJob) para retroagir created_at e
  # expires_at e ter mistura de relatórios vivos e expirados.
  def upsert_report(cfg, triage, tier, created_at:, expires_at:)
    return if ReportSnapshot.where(triage_id: triage.id).exists?

    token = "RPT-#{cfg[:code]}-#{triage.id.to_s[0, 8]}"
    ReportSnapshot.create!(
      triage: triage, protocol_definition: triage.protocol_definition,
      outcome: { "tier" => tier, "priority" => triage.priority, "status" => "terminal" },
      payload: { "tier" => tier, "priority" => triage.priority, "completed_at" => triage.completed_at&.iso8601 },
      token: token, signature: ReportSnapshot.sign(token),
      created_at: created_at, expires_at: expires_at
    )
  end

  # Idempotente por payload.demo_id sintético (domain_events não tem chave natural).
  def upsert_event(demo_id, name:, occurred_at:, payload:)
    existing = DomainEvent.where("payload ->> 'demo_id' = ?", demo_id).first
    return existing if existing

    DomainEvent.create!(
      name: name, occurred_at: occurred_at, published_at: occurred_at, created_at: occurred_at,
      payload: payload.merge("demo_id" => demo_id)
    )
  end

  def build_events(cfg, protocols, citizens, triages)
    code = cfg[:code]
    ana = actor(cfg, "ana")
    bruno = actor(cfg, "bruno")

    # Eventos de auditoria de protocolo (four-eyes + protocol_events).
    respv2 = ProtocolDefinition.find_by!(name: "triage-respiratoria", version: 2)
    dengue1 = protocols["triagem-dengue"]
    dengue2 = ProtocolDefinition.find_by!(name: "triagem-dengue", version: 2)

    # respiratoria v2: criada por A, publicada por B → fourEyes = true
    upsert_event("#{code}-P-RESP2-C", name: "protocol.created", occurred_at: at_days_ago(20),
                 payload: { "protocol_definition_id" => respv2.id, "name" => "triage-respiratoria", "version" => 2, "actor" => ana })
    upsert_event("#{code}-P-RESP2-P", name: "protocol.published", occurred_at: at_days_ago(18),
                 payload: { "protocol_definition_id" => respv2.id, "name" => "triage-respiratoria", "version" => 2, "actor" => bruno })
    # dengue v1: criada e publicada pelo MESMO ator → fourEyes = false
    upsert_event("#{code}-P-DENG1-C", name: "protocol.created", occurred_at: at_days_ago(25),
                 payload: { "protocol_definition_id" => dengue1.id, "name" => "triagem-dengue", "version" => 1, "actor" => ana })
    upsert_event("#{code}-P-DENG1-P", name: "protocol.published", occurred_at: at_days_ago(24),
                 payload: { "protocol_definition_id" => dengue1.id, "name" => "triagem-dengue", "version" => 1, "actor" => ana })
    # dengue v2 aposentada
    upsert_event("#{code}-P-DENG2-R", name: "protocol.retired", occurred_at: at_days_ago(10),
                 payload: { "protocol_definition_id" => dengue2.id, "name" => "triagem-dengue", "version" => 2, "actor" => bruno })

    # conversation.* e consent.* (prefixos do filtro de Eventos)
    citizens.each_with_index do |c, i|
      convo = c[:convo]
      days = c[:days]
      upsert_event("#{code}-CV-#{i}", name: "conversation.consented", occurred_at: at_days_ago(days, hour: 10),
                   payload: { "conversation_id" => convo.id, "actor" => "sistema" })
      upsert_event("#{code}-CO-#{i}", name: "consent.given", occurred_at: at_days_ago(days, hour: 11),
                   payload: { "conversation_id" => convo.id, "actor" => "cidadão" })
    end

    # Eventos de trilha (nomes crus, por triagem completa) + triage./priority.
    triages.each_with_index do |t, i|
      next unless t[:completed]

      tri = t[:triage]
      base = at_days_ago(t[:days], hour: 9)
      upsert_event("#{code}-T-SC-#{i}", name: "scored", occurred_at: base + 1.minute,
                   payload: { "triage_id" => tri.id, "rule" => "weighted", "ref" => "mode:#{t[:mode]}", "out" => t[:tier], "actor" => "sistema" })
      upsert_event("#{code}-T-TA-#{i}", name: "tier_assigned", occurred_at: base + 2.minutes,
                   payload: { "triage_id" => tri.id, "rule" => "threshold", "ref" => "tier", "out" => t[:tier], "actor" => "sistema" })
      upsert_event("#{code}-T-DONE-#{i}", name: "triage.completed", occurred_at: base + 3.minutes,
                   payload: { "triage_id" => tri.id, "actor" => "sistema" })
      next unless t[:priority] == 1

      upsert_event("#{code}-T-PR-#{i}", name: "priority_rule", occurred_at: base + 2.minutes,
                   payload: { "triage_id" => tri.id, "rule" => "escalate", "ref" => "priority", "out" => "1", "actor" => "sistema" })
      upsert_event("#{code}-PRI-#{i}", name: "priority.escalated", occurred_at: base + 3.minutes,
                   payload: { "triage_id" => tri.id, "actor" => "sistema" })
    end
  end

  # Semeia a cidade da conexão CORRENTE. Quem chama escolhe a cidade.
  def seed_current_city(cfg)
    protocols = build_protocols
    citizens = build_conversations_and_consents(cfg)
    build_ingestion(cfg)
    triages = build_triages_and_reports(cfg, citizens, protocols)
    build_events(cfg, protocols, citizens, triages)

    {
      conversations: Conversation.count, consents: Consent.count, inbound: InboundMessage.count,
      outbound: OutboundMessage.count, triages: Triage.count, reports: ReportSnapshot.count,
      protocols: ProtocolDefinition.count, events: DomainEvent.count
    }
  end

  def run!
    CITIES.filter_map do |cfg|
      city = City.find_by(slug: cfg[:slug])
      unless city&.servable?
        warn "[dashboard_demo] #{cfg[:slug]} ausente ou inativa no catálogo — pulada (rode bin/rails city:dev_baseline)"
        next
      end

      counts = nil
      Current.set(city: city) { CityConnection.with(city) { counts = seed_current_city(cfg) } }
      puts "[dashboard_demo] #{cfg[:slug]} #{counts.inspect}"
      counts
    end
  end

  # Falhas da cidade da conexão CORRENTE (vazio = todos os painéis populados).
  def verify_current_city(slug)
    tz = ActiveSupport::TimeZone["America/Sao_Paulo"]
    p = Admin::Api::Period.parse(key: "30d", from: nil, to: nil, tz: tz)
    failures = []
    check = ->(cond, msg) { failures << "#{slug}: #{msg}" unless cond }

    cv = Admin::ConversationsQuery.call(period: p)
    check.call(cv[:funnel].sum { |f| f[:count] }.positive?, "conversations funnel empty")
    check.call(cv[:live].to_i.positive?, "no live conversations")

    co = Admin::ConsentQuery.call(period: p)
    check.call(co[:given].to_i.positive?, "no consents given")
    check.call(co[:revoked].to_i.positive?, "no consents revoked")

    ig = Admin::IngestionQuery.call(period: p)
    check.call(ig[:inboundTotal].to_i.positive?, "no inbound messages")
    check.call(ig[:ack].sum { |a| a[:count] }.positive?, "ack breakdown empty")

    tr = Admin::TriagesQuery.call(period: p)
    check.call(tr[:started].to_i.positive?, "no triages started")

    cl = Admin::ClassificationQuery.call(period: p)
    check.call(cl[:tiers].all? { |t| t[:count].to_i.positive? }, "a tier bucket is empty")
    check.call(cl[:priorityTrue].to_i.positive?, "no priority triages")
    check.call(cl[:byMode].size >= 2, "<2 scoring modes")

    rp = Admin::ReportsQuery.call(period: p)
    check.call(rp[:reports].any? { |r| r[:live] }, "no live reports")
    check.call(rp[:reports].any? { |r| !r[:live] }, "no expired reports")

    pr = Admin::ProtocolsQuery.index
    check.call(pr[:list].size >= 5, "<5 protocol rows")

    ev = Admin::EventsQuery.call(name: nil, from: nil, to: nil, period: p)
    names = ev[:byType].map { |x| x[:name] }
    %w[triage. consent. conversation. protocol. priority.].each do |pre|
      check.call(names.any? { |n| n.start_with?(pre) }, "no events for prefix #{pre}")
    end

    failures
  end

  def verify!
    CITIES.flat_map do |cfg|
      city = City.find_by(slug: cfg[:slug])
      next [ "#{cfg[:slug]}: city missing or not active in the catalog" ] unless city&.servable?

      CityConnection.with(city) { verify_current_city(cfg[:slug]) }
    end
  end
end
```

- [ ] **Step 4: Point the seed file and the rake tasks at the module**

`db/seeds/dashboard_demo.rb` inteiro:

```ruby
# Dataset opt-in dos painéis, em cada cidade de dev. Ver lib/dashboard_demo.rb.
require Rails.root.join("lib/dashboard_demo").to_s

DashboardDemo.run!
```

`lib/tasks/seed_demo.rake` inteiro:

```ruby
namespace :db do
  namespace :seed do
    desc "Load the dashboard demo dataset into every dev city (idempotent)"
    task demo: :environment do
      load Rails.root.join("db/seeds/dashboard_demo.rb")
    end

    # Cada cidade é verificada dentro da conexão dela (DashboardDemo.verify!).
    desc "Verify the dashboard demo dataset populates every panel (every dev city)"
    task "demo:verify": :environment do
      require Rails.root.join("lib/dashboard_demo").to_s

      failures = DashboardDemo.verify!
      if failures.empty?
        puts "[dashboard_demo:verify] OK — all panels populated for every dev city"
      else
        abort "[dashboard_demo:verify] FAILURES:\n- #{failures.join("\n- ")}"
      end
    end
  end
end
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/lib/dashboard_demo_spec.rb`
Expected: PASS, 4 examples, 0 failures.

Run: `grep -rn "Municipality\|municipality_id\|role: :admin" apps/api/lib/dashboard_demo.rb apps/api/db/seeds apps/api/lib/tasks/seed_demo.rake`
Expected: vazio.

- [ ] **Step 6: Load and verify the dataset in the development database**

Pré-requisito: Task 4 Step 6 feita (duas cidades ativas). Só grava linhas idempotentes.

```bash
docker compose exec -T api bin/rails db:seed:demo
docker compose exec -T api bin/rails db:seed:demo:verify
```
Expected: duas linhas `[dashboard_demo] curitiba {...}` / `maringa {...}` com contagens diferentes (escala 1.0 e 0.4),
e `[dashboard_demo:verify] OK — all panels populated for every dev city`. Rodar `db:seed:demo` de novo: mesmas
contagens.

- [ ] **Step 7: Run the whole suite and the stale-reference check**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`
Expected: 0 failures, 2 pending.

Run: `grep -rnE "Plano 3([^B]|$)" apps/api/app apps/api/lib apps/api/config apps/api/db apps/api/spec`
Expected: vazio.

- [ ] **Step 8: Commit**

```bash
git -C apps/api add lib/dashboard_demo.rb db/seeds/dashboard_demo.rb lib/tasks/seed_demo.rake spec/lib/dashboard_demo_spec.rb
git -C apps/api commit -m "Seed the dashboard demo dataset per city

Move the dataset into lib/dashboard_demo.rb without the municipality
column or the privileged connection: it seeds and verifies each dev
city inside that city's connection, with a spec proving it fills every
panel the verify task checks.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Definition of done

- `Admin::Api` responde `403 no_city_membership` a usuário da cidade sem membership ativo, em todo endpoint.
- Operador loga no host `admin.*` com senha + TOTP; sessão pendente não autentica; login auditado em `platform_events`;
  cookie `operator_session_id` host-only; cookie de operador não autentica em host de cidade e vice-versa.
- `SessionsController` da cidade sem ramo de operador; `POST /session/challenge` não existe em host de cidade.
- Sessão da cidade A não autentica na cidade B (request spec) e nenhum código seta `domain:` em cookie (guarda).
- `rails city:dev_baseline` sobe `curitiba` e `maringa` com bancos próprios; `db:seed` semeia as duas; o `curl` da
  Task 4 prova `200` na cidade certa e `401` na outra.
- `db:seed:demo` + `db:seed:demo:verify` passam nas duas cidades de dev.
- Suíte: 0 failures, 2 pending — os dois `skip "Plano 3B: grant de operador"`.
- `grep -rnE "Plano 3([^B]|$)" app lib config db spec` vazio.

## O que este plano NÃO faz

- Grant assinado de operador entrando numa cidade e callback único do gov.br em `auth.*` (Plano 3B).
- Enrollment de MFA e reset de senha de operador (Plano 3B, junto do grant).
- Nenhum frontend: proxy do Vite, hosts por cidade, CORS dinâmico, renomear `municipality_*` (Plano 6).
- Autorização por papel em cada painel (hoje qualquer papel local lê os painéis da própria cidade).
- Provisionamento em duas fases, `city:migrate:all`, 503 por cidade atrasada (Plano 4).
- Worker/fila por cidade (Plano 5). Chave de cifra por cidade (Plano 6).

## Riscos

1. **Ordem das rotas é o que separa operador de usuário.** A primeira rota que casa vence; alguém que declarar
   `resource :session` acima do bloco do console mandaria o `admin.*` para o login da cidade. Mitigação: o bloco é o
   primeiro do `draw` com comentário, os request specs de isolamento de host, e `require_console_host` no controller.
2. **`city:dev_up` só carrega schema em banco sem `users`.** Um banco de cidade parcialmente carregado (com `users`,
   sem outras tabelas) não é reparado — isso é trabalho do `city:migrate:all` do Plano 4.
3. **Drift entre o dataset de demo e as queries** anterior ao corte pode aparecer no spec da Task 5. A regra é ajustar
   o dataset, nunca a checagem.
4. **Navegador segue sem login** até o Plano 6 (proxy do Vite troca o Host). A prova deste plano é por spec e `curl`.
