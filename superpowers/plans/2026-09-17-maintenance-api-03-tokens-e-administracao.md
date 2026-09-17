# Plano 3 — Tokens de serviço e administração (fatia 3)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** a API de manutenção passa a ter **escrita**: um mantenedor convida e desativa outros mantenedores, cria e revoga tokens de serviço para automação, e lê a própria auditoria. Automação autentica por bearer token, com poderes restringíveis, e nunca alcança mantenedores, tokens nem auditoria.

**Architecture:** a autenticação deixa de ser "cookie ou nada" e passa por um objeto de credencial único (`Maintenance::Credential`), que responde se quem age é pessoa ou token, se pode escrever e quais cidades alcança. As mutations nascem com o par tentativa/resultado da auditoria embutido na base, e dois analisadores de query recusam, antes da execução, o que um token não pode fazer.

**Tech Stack:** Rails 8.1.3, `graphql-ruby` 2.6, `rotp`/`bcrypt`, Postgres, RSpec.

**Spec:** `docs/superpowers/specs/2026-09-17-maintenance-graphql-api-design.md` (§6 ciclo de vida da conta, §7 tokens de serviço, §8 schema e limites, §9 auditoria e `auditEvents`, §10 testes, §11 fatia 3).

**Planos anteriores:** Plano 1 (staging) e Plano 2 (fundação) mergeados e pusheados em `main` (merges `1452053` e `b7263b0`). Baseline de entrada: **998 exemplos, 0 falhas**.

## Decisões minhas na escrita (cada uma com o custo se estiver errada)

1. **O escopo por cidade é definido agora e aplicado no Plano 4.** `city_slugs` é gravado, validado e exposto, e `Credential#allows_city?` existe com spec própria — mas não há nenhum campo de cidade no schema ainda, então não há onde aplicá-lo. O Plano 4 liga o predicado no `city(slug:)`. **Custo se errado:** um token com escopo de cidade não restringe nada até o Plano 4; por isso a Task 5 acrescenta uma guarda que quebra a suíte quando um campo de cidade aparecer sem passar pelo predicado.
2. **A guarda de schema ganha uma allowlist de nomes, em vez de eu renomear o conceito.** `spec/architecture/maintenance_schema_spec.rb` recusa nomes que contenham `token`, `secret`, `key` — e esta fatia publica `MaintenanceToken`, `maintenanceTokens` e `createMaintenanceToken`. Renomear para "credencial" esconderia o que a coisa é. A allowlist é nominal e exata (nunca por fragmento), e cada entrada carrega o comentário de por que foi revisada. **Custo se errado:** a guarda passa a ter exceções, e quem acrescentar `accessToken` pode tentar incluí-lo na lista — o code review é quem barra.
3. **O segredo do token sai uma vez, num campo chamado `secretOnce`.** Nome explícito, na allowlist, com a razão declarada no schema. **Custo se errado:** um cliente que ignore a resposta perde o token e precisa criar outro.
4. **O digest do token é HMAC com o `secret_key_base` do ambiente.** Isso dá, de graça, uma terceira barreira de isolamento: um token de staging não valida em development nem que o prefixo seja adulterado, porque as credentials são próprias de cada ambiente (Plano 1). **Custo se errado:** rotacionar `secret_key_base` invalida todos os tokens — o que é o comportamento desejado, mas precisa estar documentado.
5. **O par tentativa/resultado da auditoria mora na base das mutations**, não em cada mutation. A spec §9 exige que a tentativa seja gravada **antes** e que a falha na gravação impeça a execução. Numa base, uma mutation nova não tem como esquecer. **Custo se errado:** uma mutation que precise de auditoria diferente terá de sobrescrever o método.
6. **`auditEvents` devolve lista simples com teto**, não connection paginada. São 6 filtros e um `limit` de no máximo 200. **Custo se errado:** quando o volume crescer, vira connection — mudança aditiva no schema.
7. **Um token nunca cria nem revoga token, nem gerencia mantenedor, nem lê auditoria** (spec §7). A recusa é por analisador, numa lista única, e não campo a campo. **Custo se errado:** um campo novo restrito precisa entrar nessa lista, e a guarda de schema (Task 5) é quem cobra.

## Global Constraints

Valem para TODA task:

- **Nunca imprimir nem persistir segredo:** token em claro (só na resposta de criação), `token_digest`, `otp_secret`, `password_digest`, conteúdo de credentials. Id, prefixo, contagem, booleano e e-mail de mantenedor, sim.
- **Nada de dado de cidadão.** Esta fatia não abre conexão com banco de cidade.
- **Ruling R18:** todo nome novo de `PlatformEvent` entra em `MaintenanceAudit::NAMES`, no `case` de dispatch **e** em `R18_PLATFORM_EVENT_NAMES` (`spec/events/platform_event_payload_guard_spec.rb`). O `case` tem `else` que levanta — mantenha. Nenhuma chave de payload pode conter `email`, `cpf`, `phone`, `name`, `body`, `wa_id`, `provider_uid`, nem ser `from`.
- **Ambiente publicado pergunta `Rota.deployed?`**; `spec/architecture/deployed_environment_guard_spec.rb` varre `app`, `config` (fora de `config/environments/`), `lib`, `db`, `bin`, `script`, `Rakefile` e `config.ru`.
- **Cookie de sessão nunca leva `domain:`** (`spec/architecture/cookie_domain_spec.rb`).
- **Commits em inglês, Conventional Commits** com tipo por extenso, terminando com a linha exata `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>` — copiada literalmente, **nunca** o nome do seu próprio modelo. Confira com `/opt/homebrew/bin/git log -1 --format=%B | tail -1`.
- **Comandos Ruby/Rails/rspec rodam no container**, a partir da raiz do monorepo: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Nunca no host.
- **Migrations:** `docker compose exec -T api bin/rails db:migrate` (development) e o mesmo com `-e RAILS_ENV=test -e POSTGRES_PASSWORD=postgres` (test). `db:migrate:platform` **não existe**. Commite `db/platform_schema.rb`.
- **`git` do PATH está quebrado:** use `/opt/homebrew/bin/git`, a partir de `apps/api`. **Nunca dar push.** Branch `feat/maintenance-api-tokens`, criada de `main`.
- **Suíte completa:** `docker compose stop worker` antes, timeout ≥ 600000 ms, `docker compose start worker` depois. Baseline: **998 exemplos, 0 falhas**.
- **Nada destrutivo** nos bancos de development e test.
- **Nunca rodar `start.sh`.**

## Fatos verificados (não re-descobrir)

- **`MaintenanceAudit.record(name, outcome:, maintainer_id:, credential:, module_name:, correlation_id: SecureRandom.uuid, **fields)`** (`app/events/maintenance_audit.rb`) devolve o `correlation_id`, valida nome e outcome (`attempted ok rejected error`) e despacha por `case` com `else` que levanta. `credential_for(session:)` devolve `{"kind" => "session"}` e `credential_for(token:)` devolve `{"kind" => "token", "token_id" => token.id}` — **o ramo de token já existe e ainda não tem chamador**.
- **Nomes já declarados:** `maintenance.session.{started,failed,locked,ended}`, `maintenance.maintainer.{invited,enrolled,accepted}`.
- **`MaintainerAuthentication`** (`app/controllers/concerns/maintainer_authentication.rb`): `COOKIE = :maintainer_session_id`, `HEADER = "X-Rota-Maintenance"`, `ABSOLUTE_TTL = 8.hours`, `IDLE_TTL = 30.minutes`, `PENDING_MFA_WINDOW`, `MAX_TOTP_ATTEMPTS`. Dois `before_action`: `require_maintenance_origin` (falha fechada quando `ENV[MaintenanceApi::ORIGIN]` é vazia) e `require_maintainer_authentication`. `current_maintainer` lê `Current.maintainer_session&.maintainer`.
- **`Maintenance::BaseController`** checa o host antes de incluir a autenticação. `Maintenance::GraphqlController` tem `MAX_QUERY_BYTES = 10_000`, `MAX_BODY_BYTES = 64_000`, o bloqueio de introspecção por regex em ambiente publicado, `query_variables` e um `context` com `maintainer`, `maintainer_session` e `request_id`.
- **`Maintenance::Schema`**: `query Types::QueryType`, `max_depth 10`, `max_complexity 200`, `use GraphQL::Schema::Timeout, max_seconds: 10`. **Não há `mutation`**. `Types::BaseObject < GraphQL::Schema::Object`; `QueryType` só tem `me`.
- **`spec/architecture/maintenance_schema_spec.rb`** tem `EXPECTED_TYPES` (`Query => %w[me]`, `Maintainer => %w[id emailAddress createdAt]`), `FORBIDDEN_FRAGMENTS` (`phone body raw evidence response context digest secret token key url`) e um auto-teste do padrão. **Ele vai falhar assim que esta fatia publicar `MaintenanceToken`** — a Task 3 acerta a guarda.
- **`Maintainer`**: `active?`, `enrolled?`, `locked?`, `register_failure!`, `clear_failures!`, `deactivate!` (apaga sessões), `last_active?` (existe e **não** é consultado por `deactivate!`), `LOCKOUT_ATTEMPTS`, `LOCKOUT_WINDOW`, scope `.active`.
- **`MaintainerInvitation`**: `TTL = 24.hours`, `.digest_for`, `.issue!(maintainer:)` → `[invitation, token]`, `.invalidate_pending_for!(maintainer)`, `#usable?`.
- **`lib/tasks/maintainer.rake`** faz convite e reconvite fora da API, zerando senha, TOTP e sessões e invalidando convites pendentes.
- **CORS** (`config/initializers/cors.rb`): o bloco de manutenção já é restrito ao host `maintenance-api.*`, lê a origem por requisição e lista `/session`, `/session/challenge`, `/invitations/enroll`, `/invitations/accept` e `/graphql`, todos com `credentials: true`.
- **Rotas de manutenção** ficam num `if MaintenanceApi.enabled?` + `constraints(MaintenanceApiHost)`, **antes** das rotas de cidade.
- **`MaintenanceApi`** (`lib/maintenance_api.rb`): `ALLOWED_ENVS`, `FLAG`, `ORIGIN`, `enabled?`, `check_boot!` (que também exige a origem em ambiente publicado).

---

## File Structure

**apps/api**
- Create: `db/platform_migrate/20260918000001_create_maintenance_tokens.rb`, `app/models/maintenance_token.rb`, `app/services/maintenance/credential.rb`, `app/graphql/maintenance/types/mutation_type.rb`, `app/graphql/maintenance/types/user_error_type.rb`, `app/graphql/maintenance/types/maintenance_token_type.rb`, `app/graphql/maintenance/types/audit_event_type.rb`, `app/graphql/maintenance/mutations/base_mutation.rb`, `app/graphql/maintenance/mutations/invite_maintainer.rb`, `app/graphql/maintenance/mutations/deactivate_maintainer.rb`, `app/graphql/maintenance/mutations/create_maintenance_token.rb`, `app/graphql/maintenance/mutations/revoke_maintenance_token.rb`, `app/graphql/maintenance/analyzers/write_scope.rb`, `app/graphql/maintenance/analyzers/human_only.rb`, `app/queries/maintenance/audit_events_query.rb`
- Create (specs): `spec/models/maintenance_token_spec.rb`, `spec/services/maintenance/credential_spec.rb`, `spec/requests/maintenance/token_auth_spec.rb`, `spec/requests/maintenance/maintainer_mutations_spec.rb`, `spec/requests/maintenance/token_mutations_spec.rb`, `spec/requests/maintenance/audit_events_spec.rb`, `spec/graphql/maintenance/analyzers_spec.rb`
- Modify: `app/controllers/concerns/maintainer_authentication.rb`, `app/controllers/maintenance/graphql_controller.rb`, `app/models/current.rb`, `app/models/maintainer.rb`, `app/events/maintenance_audit.rb`, `app/graphql/maintenance/schema.rb`, `app/graphql/maintenance/types/query_type.rb`, `db/platform_schema.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/architecture/maintenance_schema_spec.rb`, `deploy/SECRETS.md`

Branch: `feat/maintenance-api-tokens`, de `main` (merge `b7263b0`).

---

### Task 1: Tabela e modelo do token de serviço

**Files:**
- Create: `db/platform_migrate/20260918000001_create_maintenance_tokens.rb`, `app/models/maintenance_token.rb`
- Modify: `db/platform_schema.rb` (gerado), `deploy/SECRETS.md`
- Test: `spec/models/maintenance_token_spec.rb`

**Interfaces:**
- Produces:
  - `MaintenanceToken`: `maintainer_id`, `name`, `token_digest`, `token_prefix`, `access` (`read` | `read_write`), `city_slugs` (array), `expires_at`, `revoked_at`, `last_used_at`, `last_used_ip`
  - `MaintenanceToken::ACCESS = %w[read read_write]`, `MAX_TTL = 90.days`
  - `MaintenanceToken.prefix` → `"rsm_dev_"` / `"rsm_stg_"` / `"rsm_test_"` conforme `Rails.env`
  - `MaintenanceToken.digest_for(token)` → HMAC-SHA256 hex com `Rails.application.secret_key_base`
  - `MaintenanceToken.issue!(maintainer:, name:, access:, city_slugs:, expires_at:)` → `[token_record, secret_em_claro]`
  - `MaintenanceToken.authenticate(secret)` → registro utilizável ou `nil` (prefixo de outro ambiente, digest desconhecido, expirado, revogado ou dono inativo devolvem `nil`)
  - `#usable?`, `#read_only?`, `#allows_city?(slug)`, `#touch_use!(ip:)`, `#revoke!`

- [ ] **Step 1: Escrever o spec que falha**

`spec/models/maintenance_token_spec.rb`:

```ruby
require "rails_helper"

# Spec da API de manutenção §7: o token é um login que não expira em 8h, então
# ele nasce com validade obrigatória, escopo restringível e um segredo que
# existe uma única vez — no banco fica só o digest.
RSpec.describe MaintenanceToken do
  include ActiveSupport::Testing::TimeHelpers

  let(:maintainer) { Maintainer.create!(email_address: "tok-#{SecureRandom.hex(3)}@rotasaude.app") }

  def issue(**overrides)
    described_class.issue!(**{ maintainer: maintainer, name: "ci", access: "read_write",
                               city_slugs: [], expires_at: 30.days.from_now }.merge(overrides))
  end

  it "returns the secret once, stores only a digest, and carries the environment prefix" do
    token, secret = issue

    expect(secret).to start_with(described_class.prefix)
    expect(token.token_digest).not_to include(secret)
    expect(token.token_prefix).to eq(described_class.prefix)
    expect(described_class.where(token_digest: secret).count).to eq(0)
    expect(described_class.digest_for(secret)).to eq(token.token_digest)
  end

  it "authenticates a usable secret and refuses every unusable one" do
    token, secret = issue

    expect(described_class.authenticate(secret)).to eq(token)
    expect(described_class.authenticate("#{described_class.prefix}nao-existe")).to be_nil
    expect(described_class.authenticate("rsm_other_#{secret.split('_').last}")).to be_nil
    expect(described_class.authenticate(nil)).to be_nil

    token.revoke!
    expect(described_class.authenticate(secret)).to be_nil
  end

  it "refuses an expired secret and one whose owner was deactivated" do
    _expired, expired_secret = issue(expires_at: 1.hour.from_now)
    travel_to(2.hours.from_now) { expect(described_class.authenticate(expired_secret)).to be_nil }

    _live, live_secret = issue
    maintainer.deactivate!
    expect(described_class.authenticate(live_secret)).to be_nil
  end

  it "requires an expiry inside the ceiling and a known access level" do
    expect { issue(expires_at: nil) }.to raise_error(ActiveRecord::RecordInvalid)
    expect { issue(expires_at: (described_class::MAX_TTL + 1.day).from_now) }.to raise_error(ActiveRecord::RecordInvalid)
    expect { issue(access: "admin") }.to raise_error(ActiveRecord::RecordInvalid)
    expect { issue(name: " ") }.to raise_error(ActiveRecord::RecordInvalid)
  end

  it "answers what its scope allows" do
    read_token, _ = issue(access: "read")
    scoped, _ = issue(city_slugs: %w[curitiba])

    expect(read_token.read_only?).to be(true)
    expect(scoped.read_only?).to be(false)
    expect(scoped.allows_city?("curitiba")).to be(true)
    expect(scoped.allows_city?("maringa")).to be(false)
    expect(read_token.allows_city?("qualquer")).to be(true)   # lista vazia = todas
  end

  it "records use without touching updated_at semantics" do
    token, _ = issue

    token.touch_use!(ip: "10.0.0.9")

    expect(token.reload.last_used_at).to be_present
    expect(token.last_used_ip).to eq("10.0.0.9")
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/maintenance_token_spec.rb`
Expected: FAIL com `NameError: uninitialized constant MaintenanceToken`.

- [ ] **Step 3: Migration**

`db/platform_migrate/20260918000001_create_maintenance_tokens.rb`:

```ruby
# Tokens de serviço da API de manutenção (spec §7), no banco de PLATAFORMA.
# Só o digest é guardado: o segredo em claro existe uma vez, na resposta da
# mutation que o cria.
class CreateMaintenanceTokens < ActiveRecord::Migration[8.1]
  def change
    create_table :maintenance_tokens, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.uuid     :maintainer_id, null: false
      t.string   :name, null: false
      t.string   :token_digest, null: false
      t.string   :token_prefix, null: false
      t.string   :access, null: false
      t.string   :city_slugs, array: true, null: false, default: []
      t.datetime :expires_at, null: false
      t.datetime :revoked_at
      t.datetime :last_used_at
      t.string   :last_used_ip
      t.timestamps
      t.index :token_digest, unique: true
      t.index :maintainer_id
      t.check_constraint "access IN ('read', 'read_write')", name: "ck_maintenance_tokens_access"
    end
  end
end
```

- [ ] **Step 4: Migrar development e test**

```bash
docker compose exec -T api bin/rails db:migrate
docker compose exec -T -e RAILS_ENV=test -e POSTGRES_PASSWORD=postgres api bin/rails db:migrate
```
Expected: tabela criada e `db/platform_schema.rb` atualizado (`git diff --stat db/platform_schema.rb`).

- [ ] **Step 5: Implementar o modelo**

`app/models/maintenance_token.rb`:

```ruby
# Token de serviço da API de manutenção (spec §7): a credencial da AUTOMAÇÃO.
#
# Três decisões que este arquivo sustenta:
#
#   1. O segredo nunca é gravado. `issue!` devolve o par [registro, segredo] e
#      o segredo some do processo depois da resposta; a busca é sempre por
#      digest.
#   2. O digest é HMAC com o secret_key_base DO AMBIENTE. Como cada ambiente tem
#      credentials próprias (Plano 1), um token de staging não valida em
#      development nem que alguém troque o prefixo — o isolamento não depende só
#      do prefixo, que é texto. Rotacionar secret_key_base invalida todos os
#      tokens, de propósito.
#   3. Validade é obrigatória e tem teto. Um token sem expiração é uma conta de
#      superusuário que ninguém revoga porque ninguém lembra que ela existe.
class MaintenanceToken < PlatformRecord
  belongs_to :maintainer

  ACCESS = %w[read read_write].freeze
  MAX_TTL = 90.days
  SECRET_BYTES = 32

  validates :name, presence: true
  validates :access, inclusion: { in: ACCESS }
  validates :expires_at, presence: true
  validate :expiry_within_ceiling

  scope :live, -> { where(revoked_at: nil).where("expires_at > ?", Time.current) }

  class << self
    # Prefixo por ambiente: dá para reconhecer o token num incidente e cadastrar
    # o padrão no secret scanning do GitHub (os repositórios são públicos).
    def prefix = "rsm_#{Rails.env.to_s.first(4)}_"

    def digest_for(secret)
      OpenSSL::HMAC.hexdigest("SHA256", Rails.application.secret_key_base.to_s, secret.to_s)
    end

    def issue!(maintainer:, name:, access:, city_slugs:, expires_at:)
      secret = "#{prefix}#{SecureRandom.urlsafe_base64(SECRET_BYTES)}"
      token = create!(maintainer: maintainer, name: name.to_s.strip, access: access,
                      city_slugs: Array(city_slugs).map(&:to_s), expires_at: expires_at,
                      token_digest: digest_for(secret), token_prefix: prefix)
      [ token, secret ]
    end

    # Recusa o prefixo de outro ambiente ANTES de consultar o banco: um token de
    # staging apresentado em development nem chega a virar query.
    def authenticate(secret)
      secret = secret.to_s
      return nil unless secret.start_with?(prefix)

      token = find_by(token_digest: digest_for(secret))
      return nil unless token&.usable?

      token
    end
  end

  def usable? = revoked_at.nil? && expires_at > Time.current && maintainer.active?

  def read_only? = access == "read"

  # Lista vazia = todas as cidades. O predicado existe desde já, mas só ganha
  # chamador no Plano 4, quando o schema tiver `city(slug:)`.
  def allows_city?(slug) = city_slugs.empty? || city_slugs.include?(slug.to_s)

  def touch_use!(ip:)
    update_columns(last_used_at: Time.current, last_used_ip: ip)
  end

  def revoke!
    update!(revoked_at: Time.current)
  end

  private

  def expiry_within_ceiling
    return if expires_at.blank?

    errors.add(:expires_at, "além do teto de #{MAX_TTL.inspect}") if expires_at > MAX_TTL.from_now
    errors.add(:expires_at, "já passou") if expires_at <= Time.current
  end
end
```

- [ ] **Step 6: Documentar a custódia**

Em `deploy/SECRETS.md`, na seção "Staging", acrescente:

```markdown
Tokens de serviço da API de manutenção (`maintenance_tokens`) são HMAC do `secret_key_base` do ambiente: rotacionar essa
chave **invalida todos os tokens** daquele ambiente, de propósito. O segredo em claro (`rsm_<env>_…`) aparece uma única
vez, na resposta da mutation que o cria, e nunca é gravado — quem perder, cria outro e revoga o antigo.
```

- [ ] **Step 7: Rodar, suíte completa e commit**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/maintenance_token_spec.rb`
Expected: PASS, 6 exemplos.

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
```

```bash
cd apps/api
/opt/homebrew/bin/git add db/platform_migrate/20260918000001_create_maintenance_tokens.rb db/platform_schema.rb \
  app/models/maintenance_token.rb deploy/SECRETS.md spec/models/maintenance_token_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: add service tokens for the maintenance API

A token carries a mandatory expiry under a ninety-day ceiling, a read or
read_write access level and an optional list of cities. Only the HMAC
digest is stored, keyed by the environment's secret_key_base, so a token
of one environment cannot authenticate in another.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 2: Credencial única e autenticação por bearer

**Files:**
- Create: `app/services/maintenance/credential.rb`
- Modify: `app/controllers/concerns/maintainer_authentication.rb`, `app/models/current.rb`, `app/events/maintenance_audit.rb` (nome novo), `spec/events/platform_event_payload_guard_spec.rb`
- Test: `spec/services/maintenance/credential_spec.rb`, `spec/requests/maintenance/token_auth_spec.rb`

**Interfaces:**
- Consumes: `MaintenanceToken.authenticate`, `MaintainerSession`, `MaintenanceAudit`.
- Produces:
  - `Maintenance::Credential.session(session)` / `.token(token)`
  - `#maintainer`, `#human?`, `#token?`, `#read_only?`, `#allows_city?(slug)`, `#audit_payload`
  - `Current.maintenance_credential`
  - Autenticação: cookie **ou** `Authorization: Bearer`; os dois juntos → 401; token não passa pela checagem de `Origin`/header; token inválido audita `maintenance.token.refused`
  - Nome novo: `maintenance.token.refused`

- [ ] **Step 1: Escrever os specs que falham**

`spec/services/maintenance/credential_spec.rb`:

```ruby
require "rails_helper"

# Spec §7: o escopo é aplicado em PONTO ÚNICO. Este objeto é esse ponto — quem
# pergunta "pode escrever?" ou "alcança esta cidade?" pergunta aqui, e não
# reimplementa a regra no resolver.
RSpec.describe Maintenance::Credential do
  let(:maintainer) { Maintainer.create!(email_address: "cred-#{SecureRandom.hex(3)}@rotasaude.app") }
  let(:session) { maintainer.maintainer_sessions.create!(mfa_verified_at: Time.current, last_seen_at: Time.current) }

  def token(**overrides)
    record, _secret = MaintenanceToken.issue!(**{ maintainer: maintainer, name: "ci", access: "read_write",
                                                  city_slugs: [], expires_at: 10.days.from_now }.merge(overrides))
    record
  end

  it "describes a human session" do
    credential = described_class.session(session)

    expect(credential.human?).to be(true)
    expect(credential.token?).to be(false)
    expect(credential.read_only?).to be(false)
    expect(credential.maintainer).to eq(maintainer)
    expect(credential.allows_city?("curitiba")).to be(true)
    expect(credential.audit_payload).to eq({ "kind" => "session" })
  end

  it "describes a token, carrying its access level and city scope" do
    scoped = token(access: "read", city_slugs: %w[curitiba])
    credential = described_class.token(scoped)

    expect(credential.token?).to be(true)
    expect(credential.human?).to be(false)
    expect(credential.read_only?).to be(true)
    expect(credential.allows_city?("curitiba")).to be(true)
    expect(credential.allows_city?("maringa")).to be(false)
    expect(credential.audit_payload).to eq({ "kind" => "token", "token_id" => scoped.id })
  end
end
```

`spec/requests/maintenance/token_auth_spec.rb`:

```ruby
require "rails_helper"

# Spec §7: automação autentica por bearer, sem cookie e sem CORS. Cookie e
# bearer juntos são recusados: não pode haver dúvida sobre quem agiu.
RSpec.describe "Maintenance token authentication", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:password) { "s3nha-forte-1" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "ta-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end
  let(:issued) do
    MaintenanceToken.issue!(maintainer: maintainer, name: "ci", access: "read_write",
                            city_slugs: [], expires_at: 10.days.from_now)
  end
  let(:token) { issued.first }
  let(:secret) { issued.last }

  def json = JSON.parse(response.body)
  def bearer(value = secret) = { "Authorization" => "Bearer #{value}" }
  def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
  def query!(headers) = post("/graphql", params: { query: "{ me { id } }" }, headers: headers)

  before do
    host! "maintenance-api.rotasaude.app"
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
  end

  it "authenticates a token without Origin, header or cookie" do
    query!(bearer)

    expect(response).to have_http_status(:ok)
    expect(json.dig("data", "me", "id")).to eq(maintainer.id)
    expect(token.reload.last_used_at).to be_present
  end

  it "refuses a cookie and a bearer in the same request" do
    post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now },
         headers: browser

    query!(browser.merge(bearer))

    expect(response).to have_http_status(:unauthorized)
    expect(json).to include("error" => "ambiguous_credentials")
  end

  it "refuses an expired, revoked, unknown or foreign-environment token, and audits the refusal" do
    query!(bearer("#{MaintenanceToken.prefix}nao-existe"))
    expect(response).to have_http_status(:unauthorized)

    token.revoke!
    query!(bearer)
    expect(response).to have_http_status(:unauthorized)

    expect(PlatformEvent.where(name: "maintenance.token.refused").count).to eq(2)
    expect(PlatformEvent.where(name: "maintenance.token.refused").last.payload).to include("outcome" => "rejected")
  end

  it "refuses a token whose owner was deactivated" do
    maintainer.deactivate!

    query!(bearer)

    expect(response).to have_http_status(:unauthorized)
  end

  it "keeps the browser path unchanged: no bearer means Origin and header are still required" do
    query!({ "Origin" => "https://attacker.example", "X-Rota-Maintenance" => "1" })

    expect(response).to have_http_status(:forbidden)
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falham**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/maintenance/credential_spec.rb spec/requests/maintenance/token_auth_spec.rb`
Expected: FAIL — constante inexistente e 403 (o bearer ainda bate na trava de `Origin`).

- [ ] **Step 3: Implementar a credencial**

`app/services/maintenance/credential.rb`:

```ruby
# Quem está agindo na API de manutenção (spec §7).
#
# Existe para que "pode escrever?" e "alcança esta cidade?" tenham UMA resposta,
# em vez de cada resolver reimplementar a regra — foi assim que a spec escreveu
# o escopo do token, e é o que torna a guarda de analisador (Task 5) possível.
module Maintenance
  class Credential
    attr_reader :maintainer, :session, :token

    def self.session(session) = new(maintainer: session.maintainer, session: session)
    def self.token(token) = new(maintainer: token.maintainer, token: token)

    def initialize(maintainer:, session: nil, token: nil)
      @maintainer = maintainer
      @session = session
      @token = token
    end

    def human? = session.present?
    def token? = token.present?

    # Pessoa nunca é read-only: o mantenedor tem poderes totais (spec §5). O
    # recorte existe só para token.
    def read_only? = token? && token.read_only?

    def allows_city?(slug) = human? || token.allows_city?(slug)

    def audit_payload
      return MaintenanceAudit.credential_for(token: token) if token?

      MaintenanceAudit.credential_for(session: session)
    end
  end
end
```

- [ ] **Step 4: Autenticar bearer no concern**

Em `app/models/current.rb`, acrescente `attribute :maintenance_credential`.

Em `app/controllers/concerns/maintainer_authentication.rb`:

1. Troque os dois `before_action` do `included do` por três, nesta ordem:

```ruby
  included do
    before_action :resolve_maintenance_credential
    before_action :require_maintenance_origin
    before_action :require_maintainer_authentication
  end
```

2. Acrescente, na seção privada:

```ruby
  BEARER = /\ABearer (.+)\z/

  # Cookie OU bearer, nunca os dois: com as duas credenciais presentes não há
  # resposta honesta para "quem agiu", e a auditoria é o que resta quando os
  # poderes são totais (spec §7).
  def resolve_maintenance_credential
    bearer = request.headers["Authorization"].to_s[BEARER, 1]
    cookie = cookies.signed[COOKIE]

    return render(json: { error: "ambiguous_credentials" }, status: :unauthorized) if bearer && cookie

    return resolve_token_credential(bearer) if bearer

    session = find_verified_session
    return unless session

    touch_session(session)
    Current.maintainer_session = session
    Current.maintenance_credential = Maintenance::Credential.session(session)
  end

  def resolve_token_credential(secret)
    token = MaintenanceToken.authenticate(secret)
    unless token
      # Sem maintainer_id: um segredo recusado não identifica ninguém. O evento
      # existe para que uma enxurrada de recusas apareça na auditoria.
      MaintenanceAudit.record("maintenance.token.refused", outcome: "rejected", module_name: "token",
                              maintainer_id: nil, credential: { "kind" => "token" },
                              token_prefix: secret.to_s.split("_").first(2).join("_"))
      return render(json: { error: "unauthenticated" }, status: :unauthorized)
    end

    token.touch_use!(ip: request.remote_ip)
    Current.maintenance_credential = Maintenance::Credential.token(token)
  end
```

3. Troque `require_maintenance_origin` e `require_maintainer_authentication` por:

```ruby
  # A trava de CSRF é do NAVEGADOR. Um token não tem Origin nem cookie, então
  # exigir os dois dele recusaria toda automação; o que protege o token é ele
  # próprio ser secreto e não viajar sozinho como o cookie viaja.
  def require_maintenance_origin
    return if Current.maintenance_credential&.token?

    expected = ENV[MaintenanceApi::ORIGIN].to_s
    return head(:forbidden) if expected.blank?
    return head(:forbidden) unless request.headers["Origin"] == expected

    head(:forbidden) unless request.headers[HEADER] == "1"
  end

  def require_maintainer_authentication
    return if Current.maintenance_credential

    render(json: { error: "unauthenticated" }, status: :unauthorized)
  end
```

4. Ajuste `current_maintainer` para `Current.maintenance_credential&.maintainer`, e **mantenha** `resume_maintainer_session`, `find_verified_session`, `touch_session`, `start_pending_session_for`, `write_maintenance_cookie` e `terminate_maintenance_session` como estão (o `SessionsController` os usa).

5. Em `app/events/maintenance_audit.rb`, acrescente `maintenance.token.refused` a `NAMES` e um `when` correspondente; em `spec/events/platform_event_payload_guard_spec.rb`, acrescente o nome a `R18_PLATFORM_EVENT_NAMES`.

- [ ] **Step 5: Passar a credencial ao GraphQL**

Em `app/controllers/maintenance/graphql_controller.rb`, no `context`, acrescente `credential: Current.maintenance_credential` (mantendo `maintainer`, `maintainer_session` e `request_id`).

- [ ] **Step 6: Rodar, suíte completa e commit**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/maintenance spec/requests/maintenance`
Expected: PASS, 0 falhas — inclusive os specs de sessão do Plano 2, que não podem regredir.

Suíte completa (worker parado antes, religado depois), e commit:

```bash
cd apps/api
/opt/homebrew/bin/git add app/services/maintenance/credential.rb app/controllers/concerns/maintainer_authentication.rb \
  app/controllers/maintenance/graphql_controller.rb app/models/current.rb app/events/maintenance_audit.rb \
  spec/events/platform_event_payload_guard_spec.rb spec/services/maintenance/credential_spec.rb \
  spec/requests/maintenance/token_auth_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: authenticate automation with bearer service tokens

One credential object answers who is acting and what they may do, so a
resolver never reimplements the rule. A token authenticates without the
browser's Origin and header checks, a cookie and a bearer in the same
request are refused, and every refused secret lands in the audit.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 3: Mutations, com o par tentativa/resultado na base

**Files:**
- Create: `app/graphql/maintenance/types/mutation_type.rb`, `app/graphql/maintenance/types/user_error_type.rb`, `app/graphql/maintenance/mutations/base_mutation.rb`, `app/graphql/maintenance/mutations/invite_maintainer.rb`, `app/graphql/maintenance/mutations/deactivate_maintainer.rb`
- Modify: `app/graphql/maintenance/schema.rb`, `app/models/maintainer.rb` (trava do último ativo), `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/maintainer_mutations_spec.rb`

**Interfaces:**
- Produces:
  - `Maintenance::Mutations::BaseMutation` com `audited(module_name:, operation:, **fields) { ... }`: grava `attempted` **antes**, executa o bloco, grava `ok`/`rejected`/`error` com o mesmo `correlation_id`
  - `Types::MutationType` com `inviteMaintainer(emailAddress:)` e `deactivateMaintainer(id:)`
  - Payload comum: `{ ok: Boolean!, errors: [UserError!]! }`, `UserError { path: String, message: String! }`
  - `Maintainer#deactivate!` levanta `Maintainer::LastActive` quando é o último ativo
  - Nomes novos: `maintenance.maintainer.deactivated`

- [ ] **Step 1: Escrever o spec que falha**

`spec/requests/maintenance/maintainer_mutations_spec.rb` — cobrir, com sessão humana autenticada (reaproveite o helper de login do spec de sessão do Plano 2):

```ruby
require "rails_helper"

# Spec §6: conceder e revogar acesso é só por sessão humana, o mantenedor não
# desativa a si mesmo nem o último ativo, e a desativação mata sessões e tokens.
RSpec.describe "Maintainer mutations", type: :request do
  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:password) { "s3nha-forte-1" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "mm-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end
  let!(:other) do
    Maintainer.create!(email_address: "other-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
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

  def mutate!(query, **variables)
    post "/graphql", params: { query: query, variables: variables }, headers: browser
  end

  before do
    host! "maintenance-api.rotasaude.app"
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
    login!
  end

  it "invites a maintainer and audits the attempt and its outcome on one correlation id" do
    mutate!('mutation($e: String!) { inviteMaintainer(emailAddress: $e) { ok errors { message } } }',
            e: "novo@rotasaude.app")

    expect(json.dig("data", "inviteMaintainer", "ok")).to be(true)
    invited = Maintainer.find_by(email_address: "novo@rotasaude.app")
    expect(invited.maintainer_invitations.count).to eq(1)

    events = PlatformEvent.where(name: "maintenance.maintainer.invited").last(2)
    expect(events.map { |e| e.payload["outcome"] }).to eq(%w[attempted ok])
    expect(events.map { |e| e.payload["correlation_id"] }.uniq.size).to eq(1)
    expect(events.last.payload["maintainer_id"]).to eq(maintainer.id)
  end

  it "never returns the invitation token" do
    mutate!('mutation($e: String!) { inviteMaintainer(emailAddress: $e) { ok } }', e: "outro@rotasaude.app")

    invitation = Maintainer.find_by(email_address: "outro@rotasaude.app").maintainer_invitations.sole
    expect(response.body).not_to include(invitation.token_digest)
    expect(json.dig("data", "inviteMaintainer").keys).to contain_exactly("ok")
  end

  it "refuses an invalid e-mail as a user error, not a crash" do
    mutate!('mutation($e: String!) { inviteMaintainer(emailAddress: $e) { ok errors { path message } } }', e: "nao-e-email")

    expect(json.dig("data", "inviteMaintainer", "ok")).to be(false)
    expect(json.dig("data", "inviteMaintainer", "errors").first["path"]).to eq("emailAddress")
    expect(PlatformEvent.where(name: "maintenance.maintainer.invited").last.payload["outcome"]).to eq("rejected")
  end

  it "deactivates another maintainer, killing sessions and tokens" do
    other.maintainer_sessions.create!(mfa_verified_at: Time.current, last_seen_at: Time.current)
    MaintenanceToken.issue!(maintainer: other, name: "ci", access: "read", city_slugs: [], expires_at: 5.days.from_now)

    mutate!('mutation($id: ID!) { deactivateMaintainer(id: $id) { ok errors { message } } }', id: other.id)

    expect(json.dig("data", "deactivateMaintainer", "ok")).to be(true)
    expect(other.reload.active?).to be(false)
    expect(MaintainerSession.where(maintainer_id: other.id)).to be_empty
    expect(MaintenanceToken.where(maintainer_id: other.id).live).to be_empty
    expect(PlatformEvent.where(name: "maintenance.maintainer.deactivated").last.payload["outcome"]).to eq("ok")
  end

  it "refuses deactivating myself and refuses leaving no active maintainer" do
    mutate!('mutation($id: ID!) { deactivateMaintainer(id: $id) { ok errors { message } } }', id: maintainer.id)
    expect(json.dig("data", "deactivateMaintainer", "ok")).to be(false)
    expect(maintainer.reload.active?).to be(true)

    mutate!('mutation($id: ID!) { deactivateMaintainer(id: $id) { ok } }', id: other.id)
    other_two = Maintainer.where(deactivated_at: nil).where.not(id: maintainer.id)
    expect(other_two).to be_empty

    # Agora `maintainer` é o último ativo: nem outro mantenedor poderia removê-lo.
    expect { maintainer.deactivate! }.to raise_error(Maintainer::LastActive)
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

Expected: FAIL — o schema não tem `mutation`, então toda operação responde erro de validação.

- [ ] **Step 3: Implementar a base e os tipos**

`app/graphql/maintenance/types/user_error_type.rb`:

```ruby
module Maintenance
  module Types
    class UserErrorType < BaseObject
      description "Erro de regra de negócio, já esperado — não é falha de sistema"

      field :path, String, null: true, description: "Campo de entrada que causou o erro, quando há um"
      field :message, String, null: false
    end
  end
end
```

`app/graphql/maintenance/mutations/base_mutation.rb`:

```ruby
# Base das mutations da API de manutenção.
#
# O par tentativa/resultado da auditoria (spec §9) mora AQUI, não em cada
# mutation: a escrita acontece no banco de plataforma e a auditoria também, mas
# uma mutation nova não pode depender de alguém lembrar de auditá-la. A tentativa
# é gravada ANTES de executar, e se essa gravação falhar o bloco não roda —
# nenhuma alteração sem registro.
module Maintenance
  module Mutations
    # GraphQL::Schema::Mutation, não RelayClassicMutation: os argumentos ficam no
    # próprio campo (`inviteMaintainer(emailAddress: ...)`), sem o invólucro
    # `input:` nem o `clientMutationId` do estilo Relay, que este frontend não usa.
    class BaseMutation < GraphQL::Schema::Mutation
      class Rejected < StandardError
        attr_reader :path

        def initialize(message, path: nil)
          super(message)
          @path = path
        end
      end

      private

      def credential = context.fetch(:credential)

      def audited(event:, module_name:, **fields)
        correlation_id = MaintenanceAudit.record(event, outcome: "attempted", module_name: module_name,
                                                 maintainer_id: credential.maintainer.id,
                                                 credential: credential.audit_payload, **fields)

        result = yield
        record_outcome(event, "ok", module_name, correlation_id, fields)
        { ok: true, errors: [] }
      rescue Rejected => e
        record_outcome(event, "rejected", module_name, correlation_id, fields)
        { ok: false, errors: [ { path: e.path, message: e.message } ] }
      rescue StandardError
        record_outcome(event, "error", module_name, correlation_id, fields) if correlation_id
        raise
      end

      def record_outcome(event, outcome, module_name, correlation_id, fields)
        MaintenanceAudit.record(event, outcome: outcome, module_name: module_name,
                                maintainer_id: credential.maintainer.id,
                                credential: credential.audit_payload,
                                correlation_id: correlation_id, **fields)
      end
    end
  end
end
```

`app/graphql/maintenance/mutations/invite_maintainer.rb`:

```ruby
module Maintenance
  module Mutations
    class InviteMaintainer < BaseMutation
      description "Convida (ou reconvida) um mantenedor. O token do convite NUNCA volta na resposta."

      argument :email_address, String, required: true

      field :ok, Boolean, null: false
      field :errors, [ Types::UserErrorType ], null: false

      def resolve(email_address:)
        email = email_address.to_s.strip.downcase

        audited(event: "maintenance.maintainer.invited", module_name: "maintainer") do
          raise Rejected.new("e-mail inválido", path: "emailAddress") unless email.match?(URI::MailTo::EMAIL_REGEXP)

          invited = Maintainer.find_or_initialize_by(email_address: email)
          raise Rejected.new("mantenedor desativado", path: "emailAddress") if invited.persisted? && !invited.active?

          PlatformRecord.transaction do
            invited.save!
            invited.update!(password: nil, otp_secret: nil, otp_enabled_at: nil, otp_recovery_codes: [],
                            failed_attempts: 0, locked_until: nil, invited_by_id: credential.maintainer.id)
            invited.maintainer_sessions.destroy_all
            MaintainerInvitation.invalidate_pending_for!(invited)
            MaintainerInvitation.issue!(maintainer: invited)
          end
        end
      end
    end
  end
end
```

**Nota para o implementador:** o token do convite é descartado de propósito — nesta fatia não há mailer, e a rake continua sendo o caminho que imprime o link. Se o spec exigir o link, pare e reporte NEEDS_CONTEXT em vez de devolvê-lo na resposta.

`app/graphql/maintenance/mutations/deactivate_maintainer.rb`:

```ruby
module Maintenance
  module Mutations
    class DeactivateMaintainer < BaseMutation
      description "Desativa um mantenedor, encerrando sessões e tokens na hora"

      argument :id, ID, required: true

      field :ok, Boolean, null: false
      field :errors, [ Types::UserErrorType ], null: false

      def resolve(id:)
        audited(event: "maintenance.maintainer.deactivated", module_name: "maintainer", target_id: id) do
          target = Maintainer.find_by(id: id)
          raise Rejected.new("mantenedor não encontrado", path: "id") unless target
          raise Rejected.new("ninguém desativa a si mesmo", path: "id") if target.id == credential.maintainer.id

          begin
            target.deactivate!
          rescue Maintainer::LastActive
            raise Rejected.new("é o último mantenedor ativo", path: "id")
          end
        end
      end
    end
  end
end
```

`app/graphql/maintenance/types/mutation_type.rb`:

```ruby
module Maintenance
  module Types
    class MutationType < BaseObject
      description "Escritas da API de manutenção"

      field :invite_maintainer, mutation: Mutations::InviteMaintainer
      field :deactivate_maintainer, mutation: Mutations::DeactivateMaintainer
    end
  end
end
```

Em `app/graphql/maintenance/schema.rb`, acrescente `mutation Types::MutationType` logo abaixo de `query`.

- [ ] **Step 4: Fechar a trava do último ativo e matar tokens na desativação**

Em `app/models/maintainer.rb`:

```ruby
  class LastActive < StandardError; end
```

e, em `deactivate!`:

```ruby
  # A trava vive AQUI, não na mutation: `last_active?` sozinho era consultivo, e
  # qualquer chamador novo (rake, console, mutation futura) trancaria todo mundo
  # para fora sem perceber.
  def deactivate!
    transaction do
      raise LastActive, "último mantenedor ativo" if last_active?

      update!(deactivated_at: Time.current)
      maintainer_sessions.destroy_all
      maintenance_tokens.update_all(revoked_at: Time.current, updated_at: Time.current)
    end
  end
```

e declare `has_many :maintenance_tokens, dependent: :destroy`.

**Atenção:** o spec do Plano 2 (`spec/models/maintainer_spec.rb`) desativa um mantenedor que pode ser o único ativo. Ajuste aquele spec criando um segundo mantenedor ativo antes da desativação — o comportamento novo é o correto; o spec antigo é que assumia o anterior.

- [ ] **Step 5: Declarar nome novo e atualizar a guarda de schema**

- `maintenance.maintainer.deactivated` em `MaintenanceAudit::NAMES`, no `case` e em `R18_PLATFORM_EVENT_NAMES`.
- Em `spec/architecture/maintenance_schema_spec.rb`, acrescente a `EXPECTED_TYPES`:

```ruby
    "Mutation" => %w[inviteMaintainer deactivateMaintainer],
    "InviteMaintainerPayload" => %w[ok errors],
    "DeactivateMaintainerPayload" => %w[ok errors],
    "UserError" => %w[path message]
```

(Confira os nomes de payload que o `graphql-ruby` publica de verdade e use exatamente esses — não invente nem remova campos que ele gere.)

- [ ] **Step 6: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/graphql/maintenance app/models/maintainer.rb app/events/maintenance_audit.rb \
  spec/events/platform_event_payload_guard_spec.rb spec/architecture/maintenance_schema_spec.rb \
  spec/models/maintainer_spec.rb spec/requests/maintenance/maintainer_mutations_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: manage maintainers through audited mutations

Mutations record the attempt before they run and the outcome after, on
one correlation id, so a write without a record is impossible. Inviting
resets the account and supersedes pending invitations; deactivating
kills sessions and tokens, and refuses both self-deactivation and
removing the last active maintainer.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 4: Criar e revogar token, e listar tokens

**Files:**
- Create: `app/graphql/maintenance/types/maintenance_token_type.rb`, `app/graphql/maintenance/mutations/create_maintenance_token.rb`, `app/graphql/maintenance/mutations/revoke_maintenance_token.rb`
- Modify: `app/graphql/maintenance/types/mutation_type.rb`, `app/graphql/maintenance/types/query_type.rb`, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/token_mutations_spec.rb`

**Interfaces:**
- Produces:
  - `createMaintenanceToken(name:, access:, citySlugs:, expiresAt:, code:)` → `{ ok, errors, secretOnce }`
  - `revokeMaintenanceToken(id:)` → `{ ok, errors }`
  - `maintenanceTokens` → `[MaintenanceToken!]!` (metadado, nunca o segredo)
  - Nomes novos: `maintenance.token.created`, `maintenance.token.revoked`

- [ ] **Step 1: Escrever o spec que falha**

`spec/requests/maintenance/token_mutations_spec.rb`:

```ruby
require "rails_helper"

# Spec §7: o token é criado por PESSOA, com TOTP na hora, o segredo volta uma
# única vez e a listagem carrega só metadado. Revogar vale na hora.
RSpec.describe "Maintenance token mutations", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:password) { "s3nha-forte-1" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "tm-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end

  def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
  def json = JSON.parse(response.body)
  def totp = ROTP::TOTP.new(maintainer.otp_secret).now

  def login!
    post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
    post "/session/challenge", params: { session_id: json["session_id"], code: totp }, headers: browser
    expect(response).to have_http_status(:ok)
  end

  def gql!(query, headers: browser, **variables)
    post "/graphql", params: { query: query, variables: variables }, headers: headers
  end

  CREATE = <<~GQL
    mutation($name: String!, $access: String!, $expiresAt: ISO8601DateTime!, $code: String!, $citySlugs: [String!]) {
      createMaintenanceToken(name: $name, access: $access, expiresAt: $expiresAt, code: $code, citySlugs: $citySlugs) {
        ok
        secretOnce
        errors { path message }
      }
    }
  GQL

  REVOKE = <<~GQL
    mutation($id: ID!) { revokeMaintenanceToken(id: $id) { ok errors { path message } } }
  GQL

  LIST = <<~GQL
    { maintenanceTokens { id name access citySlugs expiresAt lastUsedAt revokedAt } }
  GQL

  def create_token!(name: "ci", access: "read_write", code: nil, expires_at: 30.days.from_now, city_slugs: [])
    gql!(CREATE, name: name, access: access, code: code || totp,
                 expiresAt: expires_at.iso8601, citySlugs: city_slugs)
  end

  before do
    host! "maintenance-api.rotasaude.app"
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
    login!
  end

  it "creates a token with a fresh TOTP, returning the secret exactly once" do
    create_token!

    payload = json.dig("data", "createMaintenanceToken")
    expect(payload["ok"]).to be(true)
    expect(payload["errors"]).to be_empty
    expect(payload["secretOnce"]).to start_with(MaintenanceToken.prefix)

    token = MaintenanceToken.sole
    expect(token.token_digest).to eq(MaintenanceToken.digest_for(payload["secretOnce"]))
    expect(token.maintainer_id).to eq(maintainer.id)

    events = PlatformEvent.where(name: "maintenance.token.created").last(2)
    expect(events.map { |e| e.payload["outcome"] }).to eq(%w[attempted ok])
    expect(events.map { |e| e.payload["correlation_id"] }.uniq.size).to eq(1)
    expect(events.last.payload).to include("token_label" => "ci", "module" => "token")
  end

  it "refuses a wrong TOTP as a user error, creating nothing and auditing the rejection" do
    create_token!(code: "000000")

    payload = json.dig("data", "createMaintenanceToken")
    expect(payload["ok"]).to be(false)
    expect(payload["secretOnce"]).to be_nil
    expect(payload["errors"].first).to include("path" => "code")
    expect(MaintenanceToken.count).to eq(0)
    expect(PlatformEvent.where(name: "maintenance.token.created").last.payload["outcome"]).to eq("rejected")
  end

  it "refuses an expiry beyond the ceiling and an unknown access level" do
    create_token!(expires_at: MaintenanceToken::MAX_TTL.from_now + 1.day)
    expect(json.dig("data", "createMaintenanceToken", "ok")).to be(false)
    expect(json.dig("data", "createMaintenanceToken", "errors").first["path"]).to eq("expiresAt")

    create_token!(access: "admin")
    expect(json.dig("data", "createMaintenanceToken", "ok")).to be(false)
    expect(MaintenanceToken.count).to eq(0)
  end

  it "lists metadata only — never the secret, on this or any later request" do
    create_token!
    secret = json.dig("data", "createMaintenanceToken", "secretOnce")

    gql!(LIST)

    listed = json.dig("data", "maintenanceTokens").sole
    expect(listed.keys).to contain_exactly(*%w[id name access citySlugs expiresAt lastUsedAt revokedAt])
    expect(response.body).not_to include(secret)
    expect(response.body).not_to include(MaintenanceToken.sole.token_digest)
  end

  it "revokes a token, and the revoked secret stops authenticating" do
    create_token!
    secret = json.dig("data", "createMaintenanceToken", "secretOnce")
    token = MaintenanceToken.sole

    gql!(REVOKE, id: token.id)

    expect(json.dig("data", "revokeMaintenanceToken", "ok")).to be(true)
    expect(token.reload.revoked_at).to be_present
    expect(PlatformEvent.where(name: "maintenance.token.revoked").last.payload["outcome"]).to eq("ok")

    post "/graphql", params: { query: "{ me { id } }" }, headers: { "Authorization" => "Bearer #{secret}" }
    expect(response).to have_http_status(:unauthorized)
  end

  it "answers a user error, not a crash, for an unknown token id" do
    gql!(REVOKE, id: SecureRandom.uuid)

    expect(json.dig("data", "revokeMaintenanceToken", "ok")).to be(false)
    expect(json.dig("data", "revokeMaintenanceToken", "errors").first["path"]).to eq("id")
  end

  # A recusa vem do analisador da Task 5. Enquanto ela não estiver aplicada este
  # exemplo falha — é o RED que a Task 5 fecha; não o marque pending.
  it "never lets a token manage tokens, whatever its access level" do
    _record, secret = MaintenanceToken.issue!(maintainer: maintainer, name: "ci", access: "read_write",
                                              city_slugs: [], expires_at: 10.days.from_now)
    bearer = { "Authorization" => "Bearer #{secret}" }

    post "/graphql", params: { query: CREATE, variables: { name: "outro", access: "read",
                                                           expiresAt: 5.days.from_now.iso8601, code: totp,
                                                           citySlugs: [] } }, headers: bearer
    expect(json["errors"]).to be_present
    expect(json.dig("data", "createMaintenanceToken")).to be_nil

    post "/graphql", params: { query: LIST }, headers: bearer
    expect(json["errors"]).to be_present

    expect(MaintenanceToken.count).to eq(1)
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

- [ ] **Step 3: Implementar o tipo e as mutations**

`app/graphql/maintenance/types/maintenance_token_type.rb`:

```ruby
module Maintenance
  module Types
    class MaintenanceTokenType < BaseObject
      description "Metadado de um token de serviço. O segredo NUNCA aparece aqui."

      field :id, ID, null: false
      field :name, String, null: false
      field :access, String, null: false
      field :city_slugs, [ String ], null: false
      field :expires_at, GraphQL::Types::ISO8601DateTime, null: false
      field :revoked_at, GraphQL::Types::ISO8601DateTime, null: true
      field :last_used_at, GraphQL::Types::ISO8601DateTime, null: true
    end
  end
end
```

`app/graphql/maintenance/mutations/create_maintenance_token.rb`:

```ruby
module Maintenance
  module Mutations
    class CreateMaintenanceToken < BaseMutation
      description "Cria um token de serviço. O segredo volta UMA vez, em secretOnce."

      argument :name, String, required: true
      argument :access, String, required: true
      argument :city_slugs, [ String ], required: false, default_value: []
      argument :expires_at, GraphQL::Types::ISO8601DateTime, required: true
      argument :code, String, required: true, description: "TOTP do momento: um token é um login que não expira em 8h"

      field :ok, Boolean, null: false
      field :errors, [ Types::UserErrorType ], null: false
      # Nome explícito e declarado na guarda de schema: o valor existe uma vez e
      # nunca é gravado. Chamá-lo de `token` ou `secret` esbarraria — com razão —
      # na guarda de nomes proibidos.
      field :secret_once, String, null: true

      def resolve(name:, access:, city_slugs:, expires_at:, code:)
        secret = nil

        # `token_label`, não `token_name`: a Ruling R18 recusa QUALQUER chave de
        # payload que contenha "name" (o fragmento existe para barrar nome de
        # pessoa), e PlatformEvent levantaria na gravação da tentativa.
        result = audited(event: "maintenance.token.created", module_name: "token", token_label: name.to_s.strip) do
          # Step-up: a sessão já está verificada, mas criar token é emitir uma
          # credencial de longa vida (spec §7).
          raise Rejected.new("código inválido", path: "code") unless Mfa::Verify.totp_valid?(credential.maintainer, code)

          record, secret = MaintenanceToken.issue!(maintainer: credential.maintainer, name: name, access: access,
                                                   city_slugs: city_slugs, expires_at: expires_at)
          record
        rescue ActiveRecord::RecordInvalid => e
          raise Rejected.new(e.record.errors.full_messages.to_sentence, path: e.record.errors.attribute_names.first.to_s.camelize(:lower))
        end

        result.merge(secret_once: secret)
      end
    end
  end
end
```

`app/graphql/maintenance/mutations/revoke_maintenance_token.rb`:

```ruby
module Maintenance
  module Mutations
    class RevokeMaintenanceToken < BaseMutation
      description "Revoga um token de serviço na hora"

      argument :id, ID, required: true

      field :ok, Boolean, null: false
      field :errors, [ Types::UserErrorType ], null: false

      def resolve(id:)
        audited(event: "maintenance.token.revoked", module_name: "token", target_id: id) do
          token = MaintenanceToken.find_by(id: id)
          raise Rejected.new("token não encontrado", path: "id") unless token

          token.revoke!
        end
      end
    end
  end
end
```

Registre as duas em `MutationType`, e acrescente a `QueryType`:

```ruby
      field :maintenance_tokens, [ Types::MaintenanceTokenType ], null: false,
            description: "Tokens de serviço, metadado apenas"

      def maintenance_tokens = MaintenanceToken.order(created_at: :desc)
```

- [ ] **Step 4: Declarar nomes e atualizar a guarda de schema**

- `maintenance.token.created` e `maintenance.token.revoked` em `NAMES`, no `case` e em `R18_PLATFORM_EVENT_NAMES`.
- Em `spec/architecture/maintenance_schema_spec.rb`, acrescente os tipos e campos novos a `EXPECTED_TYPES` **e** a allowlist nominal:

```ruby
  # Nomes que CONTÊM fragmento proibido e mesmo assim são publicados, cada um
  # revisado: esta fatia administra tokens, e chamá-los de outra coisa esconderia
  # o que são. A lista é NOMINAL e exata — nunca por fragmento.
  ALLOWED_NAMES = %w[
    MaintenanceToken maintenanceTokens createMaintenanceToken revokeMaintenanceToken
    CreateMaintenanceTokenPayload RevokeMaintenanceTokenPayload secretOnce
  ].freeze
```

e no exemplo dos fragmentos, ignore os nomes da allowlist antes de casar o padrão. **Não** transforme a allowlist em prefixo ou regex: um `accessToken` novo tem de quebrar.

- [ ] **Step 5: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/graphql/maintenance app/events/maintenance_audit.rb \
  spec/events/platform_event_payload_guard_spec.rb spec/architecture/maintenance_schema_spec.rb \
  spec/requests/maintenance/token_mutations_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: create, list and revoke service tokens

Creating a token re-asks for the TOTP, because a token is a login that
does not expire in eight hours. The secret comes back once, in
secretOnce, and never again; the listing carries metadata only.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 5: O que um token não pode fazer

**Files:**
- Create: `app/graphql/maintenance/analyzers/write_scope.rb`, `app/graphql/maintenance/analyzers/human_only.rb`
- Modify: `app/graphql/maintenance/schema.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/graphql/maintenance/analyzers_spec.rb`, e remover o `pending` do item 6 da Task 4

**Interfaces:**
- Produces:
  - `Maintenance::Analyzers::WriteScope`: operação `mutation` com credencial `read_only?` é recusada antes da execução
  - `Maintenance::Analyzers::HumanOnly::RESTRICTED` — lista única dos campos de raiz que só sessão humana alcança: `maintainers`, `maintenanceTokens`, `auditEvents`, `inviteMaintainer`, `deactivateMaintainer`, `createMaintenanceToken`, `revokeMaintenanceToken`
  - Guarda: um campo de raiz novo que não esteja nem em `RESTRICTED` nem numa lista explícita de "pode token" quebra a suíte

- [ ] **Step 1: Escrever o spec que falha**

`spec/graphql/maintenance/analyzers_spec.rb`, executando o schema direto (sem HTTP) com um `context` montado à mão:

1. Token `read` + `mutation` → erro citando o escopo, e **nenhuma** escrita acontece (conte `PlatformEvent`, conte `MaintenanceToken`).
2. Token `read_write` + `mutation` de token/mantenedor → recusado por `HumanOnly`.
3. Token `read_write` + `mutation` que não seja restrita → permitido (use uma das mutations existentes se houver alguma não restrita; se **todas** forem restritas nesta fatia, declare isso no spec com um comentário e prove só o caminho negativo — e diga no relatório).
4. Token + `auditEvents` ou `maintenanceTokens` → recusado.
5. Sessão humana → tudo permitido.
6. **Guarda de cobertura:** todo campo de raiz de `Query` e de `Mutation` está em `RESTRICTED` ou em `TOKEN_ALLOWED`; um campo novo em qualquer um dos dois tipos quebra este exemplo até ser classificado.
7. **Guarda do escopo por cidade (decisão 1):** nenhum campo de raiz aceita argumento chamado `slug` ou `citySlug` — quando o Plano 4 acrescentar `city(slug:)`, este exemplo falha e obriga quem o escrever a ligar `Credential#allows_city?`. Deixe o comentário dizendo exatamente isso.

- [ ] **Step 2: Rodar e confirmar que falha**

- [ ] **Step 3: Implementar os analisadores**

```ruby
# Recusa ANTES de executar (spec §7): um token `read` não roda mutation. A
# checagem é na análise da query, não no resolver, porque "não escreve" tem de
# valer para toda mutation — inclusive a que alguém acrescentar amanhã.
module Maintenance
  module Analyzers
    class WriteScope < GraphQL::Analysis::Analyzer
      def initialize(subject)
        super
        @credential = subject.context[:credential]
      end

      def analyze? = query.selected_operation&.operation_type == "mutation"

      def result
        return unless @credential&.read_only?

        GraphQL::AnalysisError.new("token de leitura não executa mutation")
      end
    end
  end
end
```

```ruby
# Campos que SÓ sessão humana alcança (spec §7): gerenciar mantenedores,
# gerenciar tokens e ler auditoria. Lista única — um resolver não repete a regra.
module Maintenance
  module Analyzers
    class HumanOnly < GraphQL::Analysis::Analyzer
      RESTRICTED = %w[
        maintainers maintenanceTokens auditEvents
        inviteMaintainer deactivateMaintainer createMaintenanceToken revokeMaintenanceToken
      ].freeze

      # Campos de raiz que um token PODE usar. Existe para a guarda de cobertura
      # do spec: campo de raiz novo tem de entrar aqui ou em RESTRICTED.
      TOKEN_ALLOWED = %w[me].freeze

      def initialize(subject)
        super
        @credential = subject.context[:credential]
        @touched = []
      end

      def on_enter_field(node, _parent, visitor)
        @touched << node.name if visitor.query.schema.query == visitor.parent_type_definition ||
                                 visitor.query.schema.mutation == visitor.parent_type_definition
      end

      def result
        return unless @credential&.token?

        forbidden = @touched & RESTRICTED
        return if forbidden.empty?

        GraphQL::AnalysisError.new("token de serviço não alcança: #{forbidden.uniq.join(', ')}")
      end
    end
  end
end
```

**Nota para o implementador:** a API de analisador do `graphql-ruby` 2.6 pode diferir do esboço acima (nomes de hook, como obter o tipo pai). Use a API real da versão instalada, mantendo o comportamento e a lista única; descreva no relatório o que precisou mudar.

Em `schema.rb`: `query_analyzer Analyzers::WriteScope` e `query_analyzer Analyzers::HumanOnly`.

- [ ] **Step 4: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/graphql/maintenance spec/graphql/maintenance/analyzers_spec.rb \
  spec/architecture/maintenance_schema_spec.rb spec/requests/maintenance/token_mutations_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: refuse what a service token may not do, before execution

A read token never runs a mutation, and no token reaches maintainers,
tokens or the audit. Both rules live in one analyzer list each, and a
coverage spec fails when a new root field is neither restricted nor
explicitly allowed.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 6: `auditEvents`

**Files:**
- Create: `app/graphql/maintenance/types/audit_event_type.rb`, `app/queries/maintenance/audit_events_query.rb`
- Modify: `app/graphql/maintenance/types/query_type.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/audit_events_spec.rb`

**Interfaces:**
- Produces:
  - `auditEvents(since:, until:, maintainerId:, module:, outcome:, limit:)` → `[AuditEvent!]!`, só sessão humana. **`until` e `module` são palavras reservadas do Ruby:** declare os argumentos com `as: :until_time` e `as: :module_filter`, mantendo os nomes públicos `until` e `module` no schema.
  - `AuditEvent`: `occurredAt`, `name`, `module`, `outcome`, `maintainerId`, `login`, `correlationId`
  - `Maintenance::AuditEventsQuery.call(since:, until_time:, maintainer_id:, module_filter:, outcome:, limit:)` — a consulta, fora do resolver
  - `Maintenance::AuditEventsQuery::LIMIT_MAX = 200`, `LIMIT_DEFAULT = 50`

- [ ] **Step 1: Escrever o spec que falha**

`spec/requests/maintenance/audit_events_spec.rb`:

```ruby
require "rails_helper"

# Spec §9: a auditoria é lida por PESSOA, filtrada, com teto. O payload guarda
# id (Ruling R18); o login é resolvido na leitura, juntando com maintainers.
RSpec.describe "Maintenance audit events", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:password) { "s3nha-forte-1" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "ae-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end
  let!(:other) { Maintainer.create!(email_address: "ae2-#{SecureRandom.hex(3)}@rotasaude.app") }

  def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
  def json = JSON.parse(response.body)
  def totp = ROTP::TOTP.new(maintainer.otp_secret).now

  def login!
    post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
    post "/session/challenge", params: { session_id: json["session_id"], code: totp }, headers: browser
    expect(response).to have_http_status(:ok)
  end

  QUERY = <<~GQL
    query($since: ISO8601DateTime, $until: ISO8601DateTime, $maintainerId: ID, $module: String,
          $outcome: String, $limit: Int) {
      auditEvents(since: $since, until: $until, maintainerId: $maintainerId, module: $module,
                  outcome: $outcome, limit: $limit) {
        name module outcome occurredAt maintainerId login correlationId
      }
    }
  GQL

  def audit_events!(headers: browser, **variables)
    post "/graphql", params: { query: QUERY, variables: variables }, headers: headers
    json.dig("data", "auditEvents")
  end

  def record!(name, outcome:, who: maintainer, module_name: "session", correlation_id: SecureRandom.uuid)
    MaintenanceAudit.record(name, outcome: outcome, module_name: module_name, maintainer_id: who.id,
                            credential: { "kind" => "session" }, correlation_id: correlation_id)
  end

  before do
    host! "maintenance-api.rotasaude.app"
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
    login!   # já grava maintenance.session.started
  end

  it "lists maintenance events newest first, resolving the login from the id" do
    record!("maintenance.session.failed", outcome: "rejected")

    events = audit_events!

    expect(events.first["name"]).to eq("maintenance.session.failed")
    expect(events.first["login"]).to eq(maintainer.email_address)
    expect(events.first["maintainerId"]).to eq(maintainer.id)
    expect(events.map { |e| e["occurredAt"] }).to eq(events.map { |e| e["occurredAt"] }.sort.reverse)

    # O evento em si continua sem e-mail: a Ruling R18 vale para o que é gravado.
    expect(PlatformEvent.where(name: "maintenance.session.failed").last.payload.to_json)
      .not_to include(maintainer.email_address)
  end

  it "keeps an attempt and its outcome together on one correlation id" do
    correlation_id = SecureRandom.uuid
    record!("maintenance.maintainer.invited", outcome: "attempted", module_name: "maintainer",
            correlation_id: correlation_id)
    record!("maintenance.maintainer.invited", outcome: "ok", module_name: "maintainer",
            correlation_id: correlation_id)

    events = audit_events!(module: "maintainer")

    expect(events.map { |e| e["outcome"] }).to contain_exactly("attempted", "ok")
    expect(events.map { |e| e["correlationId"] }.uniq).to eq([ correlation_id ])
  end

  it "filters by time, maintainer, module and outcome" do
    travel_to(3.days.ago) { record!("maintenance.session.ended", outcome: "ok") }
    record!("maintenance.session.failed", outcome: "rejected", who: other)
    record!("maintenance.maintainer.enrolled", outcome: "ok", module_name: "maintainer")

    recent = audit_events!(since: 1.day.ago.iso8601)
    expect(recent.map { |e| e["name"] }).not_to include("maintenance.session.ended")

    old_only = audit_events!(until: 2.days.ago.iso8601)
    expect(old_only.map { |e| e["name"] }).to eq([ "maintenance.session.ended" ])

    by_maintainer = audit_events!(maintainerId: other.id)
    expect(by_maintainer.map { |e| e["maintainerId"] }.uniq).to eq([ other.id ])
    expect(by_maintainer.first["login"]).to eq(other.email_address)

    by_module = audit_events!(module: "maintainer")
    expect(by_module.map { |e| e["module"] }.uniq).to eq([ "maintainer" ])

    rejected = audit_events!(outcome: "rejected")
    expect(rejected.map { |e| e["outcome"] }.uniq).to eq([ "rejected" ])
  end

  it "caps the limit instead of refusing it" do
    (Maintenance::AuditEventsQuery::LIMIT_MAX + 5).times { record!("maintenance.session.failed", outcome: "rejected") }

    events = audit_events!(limit: Maintenance::AuditEventsQuery::LIMIT_MAX + 100)

    expect(json["errors"]).to be_nil
    expect(events.size).to eq(Maintenance::AuditEventsQuery::LIMIT_MAX)
  end

  it "never shows platform events that are not maintenance events" do
    Platform.audit("operator.login", operator_id: SecureRandom.uuid)

    names = audit_events!(limit: Maintenance::AuditEventsQuery::LIMIT_MAX).map { |e| e["name"] }

    expect(names).to all(start_with("maintenance."))
  end

  # A recusa vem do analisador da Task 5: é o RED que aquela task fecha.
  it "refuses a service token, whatever its access level" do
    _record, secret = MaintenanceToken.issue!(maintainer: maintainer, name: "ci", access: "read_write",
                                              city_slugs: [], expires_at: 10.days.from_now)

    post "/graphql", params: { query: QUERY, variables: {} }, headers: { "Authorization" => "Bearer #{secret}" }

    expect(json["errors"]).to be_present
    expect(json.dig("data", "auditEvents")).to be_nil
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

- [ ] **Step 3: Implementar**

`app/queries/maintenance/audit_events_query.rb`:

```ruby
# Leitura da auditoria de manutenção (spec §9). Fica fora do resolver porque a
# parte difícil é SQL sobre jsonb mais a resolução dos logins, e isso merece
# spec próprio sem passar por GraphQL.
#
# O login não está no evento (Ruling R18: o payload guarda id). Ele é resolvido
# aqui, numa consulta só — um por evento seria N+1 numa tela que lista 200.
module Maintenance
  class AuditEventsQuery
    LIMIT_MAX = 200
    LIMIT_DEFAULT = 50

    Row = Struct.new(:name, :module_name, :outcome, :occurred_at, :maintainer_id, :login, :correlation_id,
                     keyword_init: true)

    def self.call(since: nil, until_time: nil, maintainer_id: nil, module_filter: nil, outcome: nil, limit: nil)
      scope = PlatformEvent.where("name LIKE 'maintenance.%'")
      scope = scope.where(occurred_at: since..) if since
      scope = scope.where(occurred_at: ..until_time) if until_time
      scope = scope.where("payload->>'maintainer_id' = ?", maintainer_id.to_s) if maintainer_id
      scope = scope.where("payload->>'module' = ?", module_filter.to_s) if module_filter
      scope = scope.where("payload->>'outcome' = ?", outcome.to_s) if outcome

      events = scope.order(occurred_at: :desc, created_at: :desc).limit(capped(limit)).to_a
      logins = Maintainer.where(id: events.filter_map { |e| e.payload["maintainer_id"] }.uniq)
                         .pluck(:id, :email_address).to_h

      events.map do |event|
        Row.new(name: event.name, module_name: event.payload["module"], outcome: event.payload["outcome"],
                occurred_at: event.occurred_at, maintainer_id: event.payload["maintainer_id"],
                login: logins[event.payload["maintainer_id"]], correlation_id: event.payload["correlation_id"])
      end
    end

    # Teto, não erro: um cliente que peça 10.000 recebe 200, e não uma falha que
    # ele teria de tratar.
    def self.capped(limit)
      [ (limit || LIMIT_DEFAULT).to_i, LIMIT_MAX ].min.clamp(1, LIMIT_MAX)
    end
  end
end
```

`app/graphql/maintenance/types/audit_event_type.rb`:

```ruby
module Maintenance
  module Types
    class AuditEventType < BaseObject
      description "Um registro da auditoria de manutenção"

      field :name, String, null: false
      field :module, String, null: false, method: :module_name
      field :outcome, String, null: false
      field :occurred_at, GraphQL::Types::ISO8601DateTime, null: false
      field :maintainer_id, ID, null: true
      field :login, String, null: true, description: "Resolvido na leitura: o evento guarda id, nunca e-mail"
      field :correlation_id, String, null: true
    end
  end
end
```

Em `QueryType`:

```ruby
      field :audit_events, [ Types::AuditEventType ], null: false,
            description: "Auditoria de manutenção, só para sessão humana" do
        argument :since, GraphQL::Types::ISO8601DateTime, required: false
        # `until` e `module` são palavras reservadas do Ruby: o nome público
        # continua o da spec, e o `as:` dá ao resolver um kwarg utilizável.
        argument :until, GraphQL::Types::ISO8601DateTime, required: false, as: :until_time
        argument :maintainer_id, ID, required: false
        argument :module, String, required: false, as: :module_filter
        argument :outcome, String, required: false
        argument :limit, Integer, required: false
      end

      def audit_events(**filters) = AuditEventsQuery.call(**filters)
```

- [ ] **Step 4: Atualizar a guarda de schema, rodar, suíte completa e commit**

Acrescente `AuditEvent` e o campo `auditEvents` a `EXPECTED_TYPES`.

```bash
cd apps/api
/opt/homebrew/bin/git add app/graphql/maintenance app/queries/maintenance spec/architecture/maintenance_schema_spec.rb \
  spec/requests/maintenance/audit_events_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: read the maintenance audit through auditEvents

Human sessions only, filtered by time, maintainer, module and outcome,
capped at two hundred rows. The event payload still carries an id and
never an e-mail; the login is resolved on read, in one query.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

## Verificação final do plano

- [ ] Suíte completa com o worker parado: 0 falhas.
- [ ] `MaintenanceAudit::NAMES`, o `case` de dispatch e `R18_PLATFORM_EVENT_NAMES` têm exatamente os mesmos nomes (compare os três).
- [ ] Nenhum segredo em diff, log ou relatório: o token em claro só aparece em `secretOnce`, e `grep -rn "secret_once\|secretOnce" app/` não devolve nada que grave o valor.
- [ ] Um bearer `read` recebe erro em qualquer mutation; um bearer `read_write` recebe erro em `maintenanceTokens`, `auditEvents` e nas quatro mutations restritas.
- [ ] `spec/architecture/maintenance_schema_spec.rb` lista todos os tipos e campos publicados, e a allowlist nominal tem só os nomes desta fatia.

## Fora deste plano

- **Leitura de cidades** (`cities`, `city(slug:)`), o limite de 5 cidades por operação e a aplicação de `Credential#allows_city?`: Plano 4.
- **Mutations por módulo** (perfil, canal, protocolos, destinatários, jobs, projeções): Plano 5.
- **SDL publicado em `contracts`**: Plano 6.
- **Mailer do convite** e o frontend: spec própria.
- **Reativar mantenedor desativado** e **preencher `invited_by_id` fora do convite**: ficam para quando houver tela.
- **Job de CI que prova a falha de boot em produção** (spec §10): pendência aberta desde o Plano 1.
