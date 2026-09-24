# Validação presencial do cidadão (subprojeto 2) — plano

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** um servidor com o papel `citizen_verifier` valida no balcão da UBS o cadastro do cidadão (CPF do documento + código de 6 dígitos gerado no wpda), o par passa a `verified` e vê o histórico completo do CPF; o `municipal_admin` concede o papel e pode desfazer validações.

**Architecture:** uma migração de cidade adiciona o papel ao CHECK de `memberships` e cria `citizen_verification_codes` e `citizen_verifications` (esta com trigger que só deixa preencher a revogação). Comandos em `Citizens::` fazem gerar código, buscar, validar e desfazer; `AttendanceController` (`/attendance/*`, sessão da cidade) e `CitizenApi::VerificationCodesController` (`/citizen/verification_codes`, sessão do cidadão) expõem isso. O dashboard ganha o módulo "Atendimento" e o botão de revisor-atendente na Equipe; o wpda ganha "Validar no posto", o selo e o histórico completo.

**Tech Stack:** Rails 8.1, PostgreSQL (apps/api); Vite + React 18 + TanStack Query + Vitest + Testing Library (apps/dashboard, apps/wpda).

**Spec:** `docs/superpowers/specs/2026-09-24-citizen-presencial-verification-design.md` (commit `59151ab`). ADR de contexto: `docs/adr/0017.md`.

## Global Constraints

- Commits: Conventional Commits **em inglês**, tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Git: `/opt/homebrew/bin/git`. `apps/api`, `apps/dashboard`, `apps/wpda` e `docs` são repositórios separados; a raiz do monorepo não é git. Branch `feat/citizen-presencial-verification` em api, dashboard e wpda, a partir de `main`. Nunca em `main`, nunca push. Staging sempre explícito (nunca `git add -A`/`.`).
- **Antes de criar o branch**, confira que ninguém está usando o repositório: `git branch --show-current` deve ser `main` e `git status --short` vazio. Outra sessão ("Página /maintenance em development") trabalha em api/dashboard com `feat/revert-target`; se o repositório não estiver em `main` limpo, pare e avise.
- api: comandos no container a partir da raiz do monorepo: `docker compose exec -T api bundle exec rspec <arquivos>`. Suíte completa: `docker compose stop worker`; `docker compose exec -T api bundle exec rspec` (em primeiro plano); `docker compose start worker` (sempre). Acima de ~3 min é regressão.
- Specs de request exigem `type: :request`; `sign_in_as(user)` vem de `spec/support/city_request_auth.rb`; o cidadão usa `sign_in_citizen`/`json_post` de `spec/support/citizen_request_helpers.rb`. Não existe factory `:user`: `User.create!(email_address:, password:)`. Arquivo novo em `spec/support` precisa de `require_relative` em `spec/rails_helper.rb`. Specs com `travel` incluem `ActiveSupport::Testing::TimeHelpers`.
- **Migração de cidade:** `db/city_migrate/20260924000001_create_citizen_verifications.rb` (confira `ls db/city_migrate | tail -1`; a última é `20260923000001`). `db/city_schema.rb` é mantido à mão; o juiz é `spec/services/city_schema_spec.rb`. Triggers ficam em `db/city_triggers.sql` (executado pela migração **e** por `load_city_schema`).
- Evento novo exige `DomainEvents.bind "<nome>", to: []` em `config/initializers/domain_events.rb`.
- Valores do spec: código de **6 dígitos**, válido por **10 minutos**, **5 tentativas**, uso único, um ativo por cidadão; motivo para desfazer com **≥ 10 caracteres** (após `btrim`); limites: geração **10 por hora** por sessão do cidadão; busca/validação **30 a cada 10 minutos** por servidor.
- Papel novo: `citizen_verifier`, em `Membership::ROLES` e `Membership::PRIVILEGED_ROLES` (conceder/revogar exigem step-up; o mantenedor da API de manutenção nunca concede).
- O atendente nunca vê celular em claro nem prioridade/respostas/resultado de triagem; o lookup exige CPF **e** código.
- Nunca comparar ambiente por literal (`Rails.env == ...`); use `Rota.deployed?`. Nunca imprimir código de validação, código OTP ou TOTP em relatório ou commit.
- wpda: todo texto interativo ≥ 18 px, alvos ≥ 48 px.

## Review Focus

1. **Um CPF com dois pares, cada um com código ativo:** o código digitado identifica exatamente o par que o gerou, nunca o outro. Teste na Task 2 (`Citizens::VerificationCodeMatch`).
2. **Histórico completo não vaza:** par `declared` com o mesmo CPF continua vendo só as próprias triagens; depois de desfeita a validação, o par volta a ver só as próprias. Teste na Task 5.
3. **Dois atendentes com o mesmo código ao mesmo tempo:** exatamente uma validação; o segundo recebe `code_expired`. Teste na Task 3 (código consumido sob lock + índice único).
4. **Busca sem código:** `POST /attendance/lookup` sem `code` (ou com CPF válido e código em branco) não devolve dado nenhum do cadastro. Teste na Task 4.
5. **Revogar consentimento de triagem de outro par pelo histórico completo:** recusado. Teste na Task 5.

---

## File Structure

**apps/api**
- Create: `db/city_migrate/20260924000001_create_citizen_verifications.rb`; Modify: `db/city_schema.rb`, `db/city_triggers.sql`
- Modify: `app/models/membership.rb`, `app/models/citizen.rb`
- Create: `app/models/citizen_verification.rb`, `app/models/citizen_verification_code.rb`
- Create: `app/commands/citizens/issue_verification_code.rb`, `app/commands/citizens/verification_code_match.rb`, `app/commands/citizens/lookup_for_verification.rb`, `app/commands/citizens/verify.rb`, `app/commands/citizens/revoke_verification.rb`
- Create: `app/policies/citizen_verification_policy.rb`
- Create: `app/controllers/attendance_controller.rb`, `app/controllers/citizen_api/verification_codes_controller.rb`
- Modify: `app/controllers/citizen_api/triages_controller.rb`, `config/routes.rb`, `config/initializers/domain_events.rb`, `config/initializers/filter_parameter_logging.rb`
- Specs: `spec/models/citizen_verification_spec.rb`, `spec/models/citizen_verification_code_spec.rb`, `spec/models/membership_roles_spec.rb`, `spec/commands/citizens/*verification*_spec.rb`, `spec/requests/attendance_spec.rb`, `spec/requests/citizen_api/verification_codes_spec.rb`, `spec/requests/citizen_api/verified_history_spec.rb`, `spec/support/verification_helpers.rb`

**apps/dashboard**
- Modify: `vite.config.ts`, `src/lib/api.ts`, `src/shell/modules.ts`, `src/App.tsx`, `src/lib/team.ts`, `src/modules/Team.tsx`
- Create: `src/lib/attendance.ts` (+ test), `src/modules/Attendance.tsx` (+ test)

**apps/wpda**
- Modify: `src/lib/citizenApi.ts`, `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`
- Create: `src/modules/citizen/VerificationCodeStep.tsx` (+ test)

---

### Task 1: Papel, tabelas, trigger e modelos

**Repo:** `apps/api`. Antes: confira `main` limpo e crie `feat/citizen-presencial-verification`.

**Files:**
- Create: `db/city_migrate/20260924000001_create_citizen_verifications.rb`
- Modify: `db/city_schema.rb` (à mão), `db/city_triggers.sql`, `app/models/membership.rb`, `app/models/citizen.rb`
- Create: `app/models/citizen_verification.rb`, `app/models/citizen_verification_code.rb`
- Test: `spec/models/citizen_verification_spec.rb`, `spec/models/membership_roles_spec.rb`

**Interfaces:**
- Produces: `Membership::ROLES` inclui `"citizen_verifier"`; `Membership::PRIVILEGED_ROLES` = `%w[municipal_admin protocol_reviewer citizen_verifier]`; `CitizenVerification` (`belongs_to :citizen`, `:verified_by_user`, `:revoked_by_user` opcional; `scope :active`; `#active?`); `CitizenVerificationCode` (`belongs_to :citizen`; constantes `TTL = 10.minutes`, `MAX_ATTEMPTS = 5`); `Citizen#verifications` (`has_many`), `Citizen#active_verification`.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/models/membership_roles_spec.rb
require "rails_helper"

RSpec.describe Membership do
  it "conhece o papel citizen_verifier e o trata como privilegiado" do
    expect(described_class::ROLES).to include("citizen_verifier")
    expect(described_class::PRIVILEGED_ROLES).to include("citizen_verifier")
  end

  it "o banco aceita o papel" do
    user = User.create!(email_address: "atendente@cidade.gov.br", password: "senha-segura-123")
    expect { described_class.create!(user: user, role: "citizen_verifier", granted_at: Time.current) }.not_to raise_error
  end
end
```

```ruby
# spec/models/citizen_verification_spec.rb
require "rails_helper"

RSpec.describe CitizenVerification do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:verifier) { User.create!(email_address: "atendente@cidade.gov.br", password: "senha-segura-123") }
  let(:admin) { User.create!(email_address: "admin@cidade.gov.br", password: "senha-segura-123") }

  def verification
    described_class.create!(citizen: citizen, verified_by_user: verifier, verified_at: Time.current)
  end

  it "permite uma validação ativa por cidadão" do
    verification
    expect { verification }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "desfaz uma vez, com os três campos juntos" do
    v = verification
    v.update!(revoked_at: Time.current, revoked_by_user: admin, revoke_reason: "documento de outra pessoa")
    expect(v.reload).not_to be_active
    expect { v.update!(revoke_reason: "outro motivo qualquer") }.to raise_error(ActiveRecord::StatementInvalid, /already revoked/)
  end

  it "depois de desfeita, aceita uma validação nova" do
    verification.update!(revoked_at: Time.current, revoked_by_user: admin, revoke_reason: "documento de outra pessoa")
    expect { verification }.not_to raise_error
  end

  it "recusa motivo curto e revogação incompleta" do
    v = verification
    expect { v.update!(revoked_at: Time.current, revoked_by_user: admin, revoke_reason: "curto") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_citizen_verifications_revocation/)
    expect { v.reload.update!(revoked_at: Time.current) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_citizen_verifications_revocation/)
  end

  it "recusa apagar e recusa mudar quem validou" do
    v = verification
    expect { v.delete }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    expect { v.update_column(:verified_by_user_id, admin.id) }
      .to raise_error(ActiveRecord::StatementInvalid, /only the revocation columns/)
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/models/membership_roles_spec.rb spec/models/citizen_verification_spec.rb`
Expected: FAIL (`citizen_verifier` fora de ROLES; `uninitialized constant CitizenVerification`).

- [ ] **Step 3: Migração**

```ruby
# db/city_migrate/20260924000001_create_citizen_verifications.rb
# Validação presencial do cidadão (spec 2026-09-24-citizen-presencial-
# verification-design §3). Aditiva: papel novo no CHECK de memberships e duas
# tabelas. citizen_verifications só aceita acréscimo, exceto preencher a
# revogação uma vez — o trigger vem de db/city_triggers.sql, a mesma fonte que
# load_city_schema executa depois de carregar o dump.
class CreateCitizenVerifications < ActiveRecord::Migration[8.1]
  ROLES_BEFORE = %w[municipal_admin protocol_author protocol_publisher protocol_reviewer viewer].freeze
  ROLES_AFTER = %w[citizen_verifier municipal_admin protocol_author protocol_publisher protocol_reviewer viewer].freeze

  def up
    replace_roles_check(ROLES_AFTER)

    create_table :citizen_verification_codes, id: :uuid do |t|
      t.references :citizen, type: :uuid, null: false, foreign_key: true, index: true
      t.string :code_digest, null: false
      t.integer :attempts, null: false, default: 0
      t.datetime :expires_at, null: false
      t.datetime :consumed_at
      t.timestamps
    end

    create_table :citizen_verifications, id: :uuid do |t|
      t.references :citizen, type: :uuid, null: false, foreign_key: true, index: true
      t.references :verified_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: true
      t.datetime :verified_at, null: false
      t.datetime :revoked_at
      t.references :revoked_by_user, type: :uuid, foreign_key: { to_table: :users }, index: true
      t.text :revoke_reason
      t.datetime :created_at, null: false
    end
    add_index :citizen_verifications, :citizen_id, unique: true, where: "revoked_at IS NULL",
              name: "idx_citizen_verifications_one_active"
    add_check_constraint :citizen_verifications,
                         "(revoked_at IS NULL AND revoked_by_user_id IS NULL AND revoke_reason IS NULL) OR " \
                         "(revoked_at IS NOT NULL AND revoked_by_user_id IS NOT NULL AND revoke_reason IS NOT NULL " \
                         "AND length(btrim(revoke_reason)) >= 10)",
                         name: "ck_citizen_verifications_revocation"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    execute "DROP TRIGGER IF EXISTS citizen_verifications_guard ON citizen_verifications"
    execute "DROP TRIGGER IF EXISTS citizen_verifications_append_only_truncate ON citizen_verifications"
    execute "DROP FUNCTION IF EXISTS rota_citizen_verification_guard()"
    drop_table :citizen_verifications
    drop_table :citizen_verification_codes
    replace_roles_check(ROLES_BEFORE)
  end

  private

  # Mesma forma de 20260918000001_create_protocol_signatures.rb: ANY
  # (ARRAY[...]::text[]) é a única que sobrevive ao round-trip migração → dump.
  def replace_roles_check(roles)
    remove_check_constraint :memberships, name: "ck_memberships_role"
    add_check_constraint :memberships, "role::text = ANY (ARRAY[#{roles.map { |r| "'#{r}'" }.join(', ')}]::text[])",
                         name: "ck_memberships_role"
  end
end
```

- [ ] **Step 4: Trigger**

No fim de `db/city_triggers.sql`:

```sql
-- citizen_verifications (spec 2026-09-24-citizen-presencial-verification §3):
-- só acréscimo, exceto preencher a revogação UMA vez. Quem validou, quando e
-- para qual cidadão nunca mudam; uma linha nunca é apagada.
CREATE OR REPLACE FUNCTION rota_citizen_verification_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'citizen_verifications is append-only: DELETE refused';
  END IF;
  IF OLD.revoked_at IS NOT NULL THEN
    RAISE EXCEPTION 'citizen_verifications: already revoked';
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id
     OR NEW.citizen_id IS DISTINCT FROM OLD.citizen_id
     OR NEW.verified_by_user_id IS DISTINCT FROM OLD.verified_by_user_id
     OR NEW.verified_at IS DISTINCT FROM OLD.verified_at
     OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'citizen_verifications: only the revocation columns may change';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS citizen_verifications_guard ON citizen_verifications;
CREATE TRIGGER citizen_verifications_guard
  BEFORE UPDATE OR DELETE ON citizen_verifications
  FOR EACH ROW EXECUTE FUNCTION rota_citizen_verification_guard();

DROP TRIGGER IF EXISTS citizen_verifications_append_only_truncate ON citizen_verifications;
CREATE TRIGGER citizen_verifications_append_only_truncate
  BEFORE TRUNCATE ON citizen_verifications
  FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only();
```

- [ ] **Step 5: Dump à mão**

Em `db/city_schema.rb`: suba `define(version:)` para `2026_09_24_000001`; troque o CHECK `ck_memberships_role` para incluir `'citizen_verifier'::text` como primeiro item; acrescente, em ordem alfabética (depois de `citizen_sessions`, antes de `citizens`), `citizen_verification_codes` e `citizen_verifications`, e as `add_foreign_key` correspondentes (`citizen_verification_codes → citizens`, `citizen_verifications → citizens`, `citizen_verifications → users` com `column: "verified_by_user_id"` e `column: "revoked_by_user_id"`). O texto exato de `where:` e `check_constraint` é o que o Postgres devolve: se a paridade falhar, rode a migração num banco de dev (`docker compose exec -T api bin/rails city:migrate:all`) e copie de `pg_indexes`/`pg_get_constraintdef`.

- [ ] **Step 6: Modelos**

`app/models/membership.rb`:

```ruby
  ROLES = %w[citizen_verifier municipal_admin protocol_author protocol_publisher protocol_reviewer viewer].freeze

  # ... (comentário existente) ...
  # citizen_verifier (spec da validação presencial §2.1): marca identidade de
  # cidadão como verificada — conceder e revogar exigem step-up, e o
  # mantenedor não concede.
  PRIVILEGED_ROLES = %w[municipal_admin protocol_reviewer citizen_verifier].freeze
```

```ruby
# app/models/citizen_verification.rb
# Validação presencial de um par (CPF, celular) — spec 2026-09-24 §3. Só
# acréscimo: desfazer preenche revoked_* uma vez (trigger em city_triggers.sql).
class CitizenVerification < ApplicationRecord
  belongs_to :citizen
  belongs_to :verified_by_user, class_name: "User"
  belongs_to :revoked_by_user, class_name: "User", optional: true

  scope :active, -> { where(revoked_at: nil) }

  def active?
    revoked_at.nil?
  end
end
```

```ruby
# app/models/citizen_verification_code.rb
# Código que o cidadão mostra no balcão (spec 2026-09-24 §2.2, §3). A lógica de
# emitir e conferir mora nos comandos Citizens::IssueVerificationCode e
# Citizens::VerificationCodeMatch.
class CitizenVerificationCode < ApplicationRecord
  TTL = 10.minutes
  MAX_ATTEMPTS = 5

  belongs_to :citizen

  scope :usable, -> { where(consumed_at: nil).where("expires_at > ?", Time.current) }

  def self.digest(citizen_id, code)
    key = Rails.application.key_generator.generate_key("citizen-verification-code", 32)
    OpenSSL::HMAC.hexdigest("SHA256", key, "#{citizen_id}:#{code}")
  end
end
```

Em `app/models/citizen.rb`:

```ruby
  has_many :verifications, class_name: "CitizenVerification", dependent: :restrict_with_error
  has_many :verification_codes, class_name: "CitizenVerificationCode", dependent: :restrict_with_error

  def active_verification
    verifications.active.first
  end
```

- [ ] **Step 7: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/models/membership_roles_spec.rb spec/models/citizen_verification_spec.rb spec/services/city_schema_spec.rb spec/architecture spec/requests/setup_privileged_role_step_up_spec.rb spec/requests/setup_grant_role_spec.rb`
Expected: PASS. Se `spec/architecture/city_encrypted_attributes_guard_spec.rb` reclamar, as tabelas novas não têm `encrypts` — não acrescente nada à lista.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add db/city_migrate/20260924000001_create_citizen_verifications.rb db/city_schema.rb db/city_triggers.sql app/models/membership.rb app/models/citizen.rb app/models/citizen_verification.rb app/models/citizen_verification_code.rb spec/models/membership_roles_spec.rb spec/models/citizen_verification_spec.rb
/opt/homebrew/bin/git commit -m "feat: add the citizen_verifier role and citizen verification tables" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Código de validação (emitir e conferir)

**Repo:** `apps/api`.

**Files:**
- Create: `app/commands/citizens/issue_verification_code.rb`, `app/commands/citizens/verification_code_match.rb`
- Test: `spec/models/citizen_verification_code_spec.rb`, `spec/commands/citizens/verification_code_match_spec.rb`
- Create: `spec/support/verification_helpers.rb`; Modify: `spec/rails_helper.rb`

**Interfaces:**
- Consumes: `CitizenVerificationCode` (`TTL`, `MAX_ATTEMPTS`, `.usable`, `.digest`), `Citizen#active_verification` (Task 1).
- Produces:
  - `Citizens::IssueVerificationCode.call(citizen:) -> Result ok(code: String, expires_at: Time) | fail(:already_verified)`
  - `Citizens::VerificationCodeMatch.call(cpf:, code:, lock: false) -> Result ok(citizen:, verification_code:) | fail(:invalid_cpf | :invalid_code | :code_expired | :code_exhausted)`. Com `lock: true`, trava as linhas candidatas (`FOR UPDATE`) — chame dentro de uma transação.
  - Helper de spec `issue_code_for(citizen) -> String`.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/support/verification_helpers.rb
module VerificationHelpers
  def issue_code_for(citizen)
    Citizens::IssueVerificationCode.call(citizen: citizen).payload.fetch(:code)
  end
end

RSpec.configure { |c| c.include VerificationHelpers }
```

(Acrescente `require_relative "support/verification_helpers"` em `spec/rails_helper.rb`.)

```ruby
# spec/models/citizen_verification_code_spec.rb
require "rails_helper"

RSpec.describe Citizens::IssueVerificationCode do
  include ActiveSupport::Testing::TimeHelpers

  before { Current.city = TEST_CITY_A }
  after { Current.reset; travel_back }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }

  it "emite 6 dígitos válidos por 10 minutos e guarda só o hash" do
    result = described_class.call(citizen: citizen)
    expect(result.payload[:code]).to match(/\A\d{6}\z/)
    expect(result.payload[:expires_at]).to be_within(1.second).of(10.minutes.from_now)
    expect(CitizenVerificationCode.last.code_digest).not_to include(result.payload[:code])
  end

  it "um código novo invalida o anterior" do
    first = described_class.call(citizen: citizen)
    described_class.call(citizen: citizen)
    expect(CitizenVerificationCode.usable.where(citizen: citizen).count).to eq(1)
    expect(Citizens::VerificationCodeMatch.call(cpf: citizen.cpf, code: first.payload[:code]).reason)
      .to eq(:invalid_code).or eq(:code_expired)
  end

  it "par já verificado não gera código" do
    user = User.create!(email_address: "a@cidade.gov.br", password: "senha-segura-123")
    CitizenVerification.create!(citizen: citizen, verified_by_user: user, verified_at: Time.current)
    expect(described_class.call(citizen: citizen).reason).to eq(:already_verified)
  end
end
```

```ruby
# spec/commands/citizens/verification_code_match_spec.rb
require "rails_helper"

RSpec.describe Citizens::VerificationCodeMatch do
  include ActiveSupport::Testing::TimeHelpers

  before { Current.city = TEST_CITY_A }
  after { Current.reset; travel_back }

  let(:mine) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:family) { Citizen.create!(cpf: "52998224725", phone: "+5541911112222") }
  let(:stranger) { Citizen.create!(cpf: "11144477735", phone: "+5541933334444") }

  def wrong(code) = code == "000000" ? "111111" : "000000"

  it "identifica o par dono do código, mesmo com outro par do CPF com código ativo" do
    issue_code_for(family)
    code = issue_code_for(mine)
    result = described_class.call(cpf: "529.982.247-25", code: code)
    expect(result.payload[:citizen]).to eq(mine)
  end

  it "código de outro CPF responde igual a código errado" do
    issue_code_for(mine)
    other = issue_code_for(stranger)
    expect(described_class.call(cpf: mine.cpf, code: other).reason).to eq(:invalid_code)
  end

  it "CPF sem código nenhum responde invalid_code" do
    expect(described_class.call(cpf: mine.cpf, code: "123456").reason).to eq(:invalid_code)
  end

  it "CPF inválido" do
    expect(described_class.call(cpf: "111.111.111-11", code: "123456").reason).to eq(:invalid_cpf)
  end

  it "vence em 10 minutos" do
    code = issue_code_for(mine)
    travel 11.minutes
    expect(described_class.call(cpf: mine.cpf, code: code).reason).to eq(:code_expired)
  end

  it "esgota em 5 tentativas erradas, mesmo que depois venha o código certo" do
    code = issue_code_for(mine)
    5.times { expect(described_class.call(cpf: mine.cpf, code: wrong(code)).reason).to eq(:invalid_code) }
    expect(described_class.call(cpf: mine.cpf, code: code).reason).to eq(:code_exhausted)
  end

  it "não consome o código (quem consome é a validação)" do
    code = issue_code_for(mine)
    2.times { expect(described_class.call(cpf: mine.cpf, code: code)).to be_ok }
  end

  it "código já consumido responde code_expired" do
    code = issue_code_for(mine)
    described_class.call(cpf: mine.cpf, code: code).payload[:verification_code].update!(consumed_at: Time.current)
    expect(described_class.call(cpf: mine.cpf, code: code).reason).to eq(:code_expired)
  end

  it "código em branco não identifica ninguém" do
    issue_code_for(mine)
    expect(described_class.call(cpf: mine.cpf, code: "").reason).to eq(:invalid_code)
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/models/citizen_verification_code_spec.rb spec/commands/citizens/verification_code_match_spec.rb`
Expected: FAIL (`uninitialized constant Citizens::IssueVerificationCode`).

- [ ] **Step 3: Implementação**

```ruby
# app/commands/citizens/issue_verification_code.rb
# "Validar no posto" no wpda (spec 2026-09-24 §4): um código de 6 dígitos,
# 10 minutos, que invalida os anteriores do mesmo par. Reasons: :already_verified.
module Citizens
  class IssueVerificationCode
    def self.call(citizen:)
      return Result.fail(:already_verified) if citizen.active_verification

      code = format("%06d", SecureRandom.random_number(1_000_000))
      record = nil
      ApplicationRecord.transaction do
        CitizenVerificationCode.usable.where(citizen: citizen).update_all(expires_at: Time.current)
        record = CitizenVerificationCode.create!(
          citizen: citizen,
          code_digest: CitizenVerificationCode.digest(citizen.id, code),
          expires_at: CitizenVerificationCode::TTL.from_now
        )
      end
      Result.ok(code: code, expires_at: record.expires_at)
    end
  end
end
```

```ruby
# app/commands/citizens/verification_code_match.rb
# Confere CPF + código digitados no balcão (spec 2026-09-24 §4, §6). Não
# consome o código. Um código errado conta uma tentativa em TODOS os códigos
# utilizáveis do CPF (não dá para saber de qual par era a tentativa). Código de
# outro CPF e CPF sem código respondem igual a código errado.
# Reasons: :invalid_cpf, :invalid_code, :code_expired, :code_exhausted.
module Citizens
  class VerificationCodeMatch
    RECENT = 24.hours

    def self.call(cpf:, code:, lock: false)
      digits = CitizenIdentity::Cpf.normalize(cpf)
      return Result.fail(:invalid_cpf) unless digits

      citizens = Citizen.where(cpf: digits)
      candidates = CitizenVerificationCode.usable.where(citizen: citizens).order(:created_at)
      candidates = candidates.lock if lock
      candidates = candidates.to_a

      if candidates.empty?
        recent = CitizenVerificationCode.where(citizen: citizens).where("created_at > ?", RECENT.ago).exists?
        return Result.fail(recent ? :code_expired : :invalid_code)
      end

      hit = candidates.find do |c|
        ActiveSupport::SecurityUtils.secure_compare(c.code_digest, CitizenVerificationCode.digest(c.citizen_id, code.to_s))
      end

      if hit.nil?
        CitizenVerificationCode.where(id: candidates.map(&:id)).update_all("attempts = attempts + 1")
        return Result.fail(:invalid_code)
      end
      return Result.fail(:code_exhausted) if hit.attempts >= CitizenVerificationCode::MAX_ATTEMPTS

      Result.ok(citizen: hit.citizen, verification_code: hit)
    end
  end
end
```

Atenção ao "esgota em 5 tentativas": a 5ª errada leva `attempts` a 5; a tentativa seguinte com o código certo acha o `hit` com `attempts >= 5` e responde `code_exhausted` — é o comportamento que o teste espera.

- [ ] **Step 4: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/models/citizen_verification_code_spec.rb spec/commands/citizens/verification_code_match_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/commands/citizens/issue_verification_code.rb app/commands/citizens/verification_code_match.rb spec/models/citizen_verification_code_spec.rb spec/commands/citizens/verification_code_match_spec.rb spec/support/verification_helpers.rb spec/rails_helper.rb
/opt/homebrew/bin/git commit -m "feat: issue and match counter verification codes for citizens" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Buscar, validar e desfazer (comandos + eventos)

**Repo:** `apps/api`.

**Files:**
- Create: `app/commands/citizens/lookup_for_verification.rb`, `app/commands/citizens/verify.rb`, `app/commands/citizens/revoke_verification.rb`, `app/policies/citizen_verification_policy.rb`
- Modify: `config/initializers/domain_events.rb`
- Test: `spec/commands/citizens/verify_spec.rb`, `spec/commands/citizens/lookup_for_verification_spec.rb`, `spec/commands/citizens/revoke_verification_spec.rb`

**Interfaces:**
- Consumes: `Citizens::VerificationCodeMatch` (Task 2), `CitizenVerification`, `Membership` (Task 1), `create_default_protocol!`, `Citizens::StartConversation`, `Citizens::SubmitAnswer` (subprojeto 1).
- Produces:
  - `CitizenVerificationPolicy.new(user, nil)#verify?` (papel `citizen_verifier`) e `#manage?` (papel `municipal_admin`).
  - `Citizens::LookupForVerification.call(cpf:, code:) -> ok(citizen:, triages: [{date: Time, protocol_name: String}]) | fail(:invalid_cpf|:invalid_code|:code_expired|:code_exhausted|:already_verified)`; em `:already_verified`, `details: { verified_at: Time }`.
  - `Citizens::Verify.call(cpf:, code:, document_checked:, by:) -> ok(verification:) | fail(:document_check_required|:invalid_cpf|:invalid_code|:code_expired|:code_exhausted|:already_verified)`.
  - `Citizens::RevokeVerification.call(verification:, reason:, by:) -> ok(verification:) | fail(:already_revoked|:own_verification|:reason_too_short)`.
  - Eventos `citizen.verified` (`citizen_id`, `verification_id`, `verified_by_user_id`) e `citizen.verification_revoked` (`citizen_id`, `verification_id`, `revoked_by_user_id`).

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/commands/citizens/verify_spec.rb
require "rails_helper"

RSpec.describe Citizens::Verify do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:verifier) { User.create!(email_address: "atendente@cidade.gov.br", password: "senha-segura-123") }

  def verify(code, checked: true)
    described_class.call(cpf: "529.982.247-25", code: code, document_checked: checked, by: verifier)
  end

  it "valida o par, consome o código, atualiza o nível e publica o evento" do
    code = issue_code_for(citizen)
    result = verify(code)
    expect(result).to be_ok
    expect(citizen.reload).to be_verification_level_verified
    expect(result.payload[:verification].verified_by_user).to eq(verifier)
    expect(CitizenVerificationCode.usable.where(citizen: citizen)).to be_empty
    event = DomainEvent.where(name: "citizen.verified").sole
    expect(event.payload).to eq("citizen_id" => citizen.id, "verification_id" => result.payload[:verification].id,
                                "verified_by_user_id" => verifier.id)
  end

  it "exige a caixa 'conferi o documento'" do
    code = issue_code_for(citizen)
    expect(verify(code, checked: false).reason).to eq(:document_check_required)
    expect(citizen.reload).to be_verification_level_declared
  end

  it "o mesmo código usado duas vezes: só a primeira valida" do
    code = issue_code_for(citizen)
    expect(verify(code)).to be_ok
    expect(verify(code).reason).to eq(:code_expired)
    expect(CitizenVerification.where(citizen: citizen).count).to eq(1)
  end

  it "não mexe em triagens, consentimentos nem métricas" do
    create_default_protocol!
    started = Citizens::StartConversation.call(citizen: citizen, consent_version: Consents.current_version, session_id: "s").payload
    Citizens::SubmitAnswer.call(conversation: started[:conversation], answer: "false", idempotency_key: "k")
    snapshot = -> { [Triage.order(:id).map(&:attributes), Consent.order(:id).map(&:attributes), DashboardMetric.order(:id).map(&:attributes)] }
    before = snapshot.call
    verify(issue_code_for(citizen))
    expect(snapshot.call).to eq(before)
  end
end
```

```ruby
# spec/commands/citizens/lookup_for_verification_spec.rb
require "rails_helper"

RSpec.describe Citizens::LookupForVerification do
  before do
    Current.city = TEST_CITY_A
    create_default_protocol!
  end
  after { Current.reset; Rails.cache.clear }

  let(:mine) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:family) { Citizen.create!(cpf: "52998224725", phone: "+5541911112222") }

  def triage_for(citizen)
    started = Citizens::StartConversation.call(citizen: citizen, consent_version: Consents.current_version, session_id: "s").payload
    Citizens::SubmitAnswer.call(conversation: started[:conversation], answer: "false", idempotency_key: SecureRandom.uuid)
  end

  it "devolve o par do código e só data + protocolo das triagens de todos os pares do CPF" do
    triage_for(mine)
    triage_for(family)
    result = described_class.call(cpf: mine.cpf, code: issue_code_for(mine))
    expect(result.payload[:citizen]).to eq(mine)
    expect(result.payload[:triages].size).to eq(2)
    expect(result.payload[:triages].first.keys).to contain_exactly(:date, :protocol_name)
    expect(result.payload[:triages].map { |t| t[:protocol_name] }.uniq).to eq([StartTriage::DEFAULT_PROTOCOL_NAME])
  end

  it "sem código certo, nada volta" do
    expect(described_class.call(cpf: mine.cpf, code: "").payload).to eq({})
  end
end
```

```ruby
# spec/commands/citizens/revoke_verification_spec.rb
require "rails_helper"

RSpec.describe Citizens::RevokeVerification do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:verifier) { User.create!(email_address: "atendente@cidade.gov.br", password: "senha-segura-123") }
  let(:admin) { User.create!(email_address: "admin@cidade.gov.br", password: "senha-segura-123") }
  let(:verification) do
    Citizens::Verify.call(cpf: citizen.cpf, code: issue_code_for(citizen), document_checked: true, by: verifier)
                    .payload[:verification]
  end

  it "desfaz, volta o par a declared e publica o evento" do
    result = described_class.call(verification: verification, reason: "documento de outra pessoa", by: admin)
    expect(result).to be_ok
    expect(citizen.reload).to be_verification_level_declared
    expect(DomainEvent.where(name: "citizen.verification_revoked").sole.payload)
      .to include("citizen_id" => citizen.id, "revoked_by_user_id" => admin.id)
  end

  it "o validador não desfaz a própria validação" do
    expect(described_class.call(verification: verification, reason: "documento de outra pessoa", by: verifier).reason)
      .to eq(:own_verification)
  end

  it "motivo curto é recusado" do
    expect(described_class.call(verification: verification, reason: "  curto  ", by: admin).reason).to eq(:reason_too_short)
  end

  it "desfazer duas vezes é recusado" do
    described_class.call(verification: verification, reason: "documento de outra pessoa", by: admin)
    expect(described_class.call(verification: verification.reload, reason: "de novo, por engano", by: admin).reason)
      .to eq(:already_revoked)
  end

  it "depois de desfeita, o par pode ser validado de novo" do
    described_class.call(verification: verification, reason: "documento de outra pessoa", by: admin)
    again = Citizens::Verify.call(cpf: citizen.cpf, code: issue_code_for(citizen), document_checked: true, by: verifier)
    expect(again).to be_ok
    expect(CitizenVerification.where(citizen: citizen).count).to eq(2)
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/commands/citizens/verify_spec.rb spec/commands/citizens/lookup_for_verification_spec.rb spec/commands/citizens/revoke_verification_spec.rb`
Expected: FAIL (`uninitialized constant Citizens::Verify`).

- [ ] **Step 3: Implementação**

```ruby
# app/policies/citizen_verification_policy.rb
# Validação presencial (spec 2026-09-24 §2): quem valida e quem desfaz.
class CitizenVerificationPolicy < ApplicationPolicy
  def verify?
    role?(:citizen_verifier)
  end

  def manage?
    role?(:municipal_admin)
  end
end
```

```ruby
# app/commands/citizens/lookup_for_verification.rb
# O cartão do balcão (spec 2026-09-24 §4, §5): o par que o código identifica e
# as triagens do CPF (todos os pares) só com data e protocolo.
module Citizens
  class LookupForVerification
    def self.call(cpf:, code:)
      match = VerificationCodeMatch.call(cpf: cpf, code: code)
      return match if match.failure?

      citizen = match.payload[:citizen]
      if (active = citizen.active_verification)
        return Result.fail(:already_verified, details: { verified_at: active.verified_at })
      end

      Result.ok(citizen: citizen, triages: triages_of_cpf(citizen.cpf))
    end

    def self.triages_of_cpf(cpf)
      Triage.joins(:conversation)
            .where(conversations: { channel: "web", citizen_id: Citizen.where(cpf: cpf).select(:id) })
            .order(created_at: :desc)
            .map { |t| { date: t.completed_at || t.created_at, protocol_name: t.protocol_name } }
    end
  end
end
```

```ruby
# app/commands/citizens/verify.rb
# Valida o par no balcão (spec 2026-09-24 §2, §4). Confere e consome o código
# sob lock: dois atendentes com o mesmo código → só um valida.
module Citizens
  class Verify
    def self.call(cpf:, code:, document_checked:, by:)
      return Result.fail(:document_check_required) unless document_checked == true

      result = nil
      ApplicationRecord.transaction do
        match = VerificationCodeMatch.call(cpf: cpf, code: code, lock: true)
        next result = match if match.failure?

        citizen = match.payload[:citizen]
        if (active = citizen.active_verification)
          next result = Result.fail(:already_verified, details: { verified_at: active.verified_at })
        end

        match.payload[:verification_code].update!(consumed_at: Time.current)
        verification = CitizenVerification.create!(citizen: citizen, verified_by_user: by, verified_at: Time.current)
        citizen.update!(verification_level: "verified")
        DomainEvents.publish("citizen.verified", citizen_id: citizen.id, verification_id: verification.id,
                                                 verified_by_user_id: by.id)
        result = Result.ok(verification: verification)
      end
      result
    rescue ActiveRecord::RecordNotUnique
      Result.fail(:already_verified)
    end
  end
end
```

`VerificationCodeMatch` com `lock: true` incrementa `attempts` num código errado dentro desta transação; como a transação termina sem erro (o `next` só sai do bloco), o incremento é gravado. Confira isso com o teste de 5 tentativas da Task 2 e, se preferir, acrescente um exemplo aqui: 5 `verify` errados seguidos → o certo responde `code_exhausted`.

```ruby
# app/commands/citizens/revoke_verification.rb
# Desfaz uma validação (spec 2026-09-24 §2.6): só municipal_admin (checado no
# controller), nunca quem validou; motivo ≥ 10 caracteres.
module Citizens
  class RevokeVerification
    MIN_REASON = 10

    def self.call(verification:, reason:, by:)
      return Result.fail(:already_revoked) unless verification.active?
      return Result.fail(:own_verification) if verification.verified_by_user_id == by.id
      return Result.fail(:reason_too_short) if reason.to_s.strip.length < MIN_REASON

      ApplicationRecord.transaction do
        verification.update!(revoked_at: Time.current, revoked_by_user: by, revoke_reason: reason.to_s.strip)
        verification.citizen.update!(verification_level: "declared")
        DomainEvents.publish("citizen.verification_revoked", citizen_id: verification.citizen_id,
                                                             verification_id: verification.id,
                                                             revoked_by_user_id: by.id)
      end
      Result.ok(verification: verification)
    end
  end
end
```

Em `config/initializers/domain_events.rb`, junto dos outros `to: []`:

```ruby
  # Validação presencial (spec 2026-09-24): trilha; a prova é citizen_verifications.
  DomainEvents.bind "citizen.verified", to: []
  DomainEvents.bind "citizen.verification_revoked", to: []
```

- [ ] **Step 4: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/commands/citizens spec/events`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/commands/citizens/lookup_for_verification.rb app/commands/citizens/verify.rb app/commands/citizens/revoke_verification.rb app/policies/citizen_verification_policy.rb config/initializers/domain_events.rb spec/commands/citizens/verify_spec.rb spec/commands/citizens/lookup_for_verification_spec.rb spec/commands/citizens/revoke_verification_spec.rb
/opt/homebrew/bin/git commit -m "feat: verify citizens at the counter and let admins revoke" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Rotas `/attendance/*` e `/citizen/verification_codes`

**Repo:** `apps/api`.

**Files:**
- Create: `app/controllers/attendance_controller.rb`, `app/controllers/citizen_api/verification_codes_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/filter_parameter_logging.rb`
- Test: `spec/requests/attendance_spec.rb`, `spec/requests/citizen_api/verification_codes_spec.rb`

**Interfaces:**
- Consumes: comandos das Tasks 2–3; `CitizenApi::BaseController` (`render_error`, `current_citizen_session`, `RateLimitStore`); `Authentication` (`Current.user`), `sign_in_as`, `sign_in_citizen`, `json_post`.
- Produces (JSON):
  - `POST /citizen/verification_codes {citizen_id}` → 201 `{code, expires_at}` | 404 | 409 `already_verified`
  - `POST /attendance/lookup {cpf, code}` → 200 `{citizen: {id, cpf_masked, phone_masked, created_at, verification_level}, triages: [{date, protocol_name}]}`
  - `POST /attendance/verifications {cpf, code, document_checked}` → 201 `{verification: {id, citizen_id, verified_at}}`
  - `GET /attendance/verifications?cpf=` → 200 `{verifications: [{id, verified_at, verified_by, phone_masked, active, revoked_at, revoked_by, revoke_reason}]}` (e-mails dos servidores em `verified_by`/`revoked_by`)
  - `POST /attendance/verifications/:id/revoke {reason}` → 200 `{verification: {...}}`
  - Erros: `{error: "<code>"}` (+ `verified_at` em `already_verified`), com os status da tabela do spec §6.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/requests/citizen_api/verification_codes_spec.rb
require "rails_helper"

RSpec.describe "Citizen verification codes", type: :request do
  before { Current.city = TEST_CITY_A; Rails.cache.clear }
  after { Current.reset }

  let!(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }

  it "gera o código para um par do celular da sessão" do
    sign_in_citizen("+5541998765432")
    json_post "/citizen/verification_codes", citizen_id: citizen.id
    expect(response).to have_http_status(:created)
    expect(JSON.parse(response.body)["code"]).to match(/\A\d{6}\z/)
  end

  it "par de outro celular: 404" do
    sign_in_citizen("+5541911112222")
    json_post "/citizen/verification_codes", citizen_id: citizen.id
    expect(response).to have_http_status(:not_found)
  end

  it "sem sessão: 401" do
    json_post "/citizen/verification_codes", citizen_id: citizen.id
    expect(response).to have_http_status(:unauthorized)
  end
end
```

```ruby
# spec/requests/attendance_spec.rb
require "rails_helper"

RSpec.describe "Attendance", type: :request do
  before { Current.city = TEST_CITY_A; Rails.cache.clear }
  after { Current.reset }

  let!(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:verifier) { user_with("atendente@cidade.gov.br", "citizen_verifier") }
  let(:admin) { user_with("admin@cidade.gov.br", "municipal_admin") }
  let(:viewer) { user_with("leitura@cidade.gov.br", "viewer") }
  def body = JSON.parse(response.body)

  def user_with(email, role)
    User.create!(email_address: email, password: "senha-segura-123").tap do |u|
      Membership.create!(user: u, role: role, granted_at: Time.current)
    end
  end

  it "lookup e validação pelo atendente" do
    code = issue_code_for(citizen)
    sign_in_as(verifier)
    json_post "/attendance/lookup", cpf: "529.982.247-25", code: code
    expect(response).to have_http_status(:ok)
    expect(body["citizen"]).to include("cpf_masked" => "***.982.247-**", "phone_masked" => "(**) *****-5432",
                                       "verification_level" => "declared")
    expect(body.to_s).not_to include("998765432")

    json_post "/attendance/verifications", cpf: "529.982.247-25", code: code, document_checked: true
    expect(response).to have_http_status(:created)
    expect(citizen.reload).to be_verification_level_verified
  end

  it "lookup sem código não devolve dado do cadastro" do
    issue_code_for(citizen)
    sign_in_as(verifier)
    json_post "/attendance/lookup", cpf: "529.982.247-25"
    expect(response).to have_http_status(:unprocessable_entity)
    expect(body.keys).to eq(["error"])
  end

  it "sem a caixa 'conferi o documento': 422" do
    sign_in_as(verifier)
    json_post "/attendance/verifications", cpf: citizen.cpf, code: issue_code_for(citizen), document_checked: false
    expect(body["error"]).to eq("document_check_required")
  end

  it "par já verificado: 409 com a data" do
    code = issue_code_for(citizen)
    sign_in_as(verifier)
    json_post "/attendance/verifications", cpf: citizen.cpf, code: code, document_checked: true
    CitizenVerificationCode.create!(citizen: citizen, code_digest: CitizenVerificationCode.digest(citizen.id, "123456"),
                                    expires_at: 10.minutes.from_now)
    json_post "/attendance/lookup", cpf: citizen.cpf, code: "123456"
    expect(response).to have_http_status(:conflict)
    expect(body).to include("error" => "already_verified", "verified_at" => be_present)
  end

  it "viewer não usa o atendimento: 403" do
    sign_in_as(viewer)
    json_post "/attendance/lookup", cpf: citizen.cpf, code: issue_code_for(citizen)
    expect(response).to have_http_status(:forbidden)
  end

  it "admin lista e desfaz; o próprio validador não desfaz" do
    sign_in_as(verifier)
    json_post "/attendance/verifications", cpf: citizen.cpf, code: issue_code_for(citizen), document_checked: true
    id = body.dig("verification", "id")

    json_post "/attendance/verifications/#{id}/revoke", reason: "documento de outra pessoa"
    expect(response).to have_http_status(:forbidden)

    sign_in_as(admin)
    get "/attendance/verifications", params: { cpf: "529.982.247-25" }
    expect(body["verifications"].sole).to include("verified_by" => "atendente@cidade.gov.br", "active" => true)

    json_post "/attendance/verifications/#{id}/revoke", reason: "curto"
    expect(body["error"]).to eq("reason_too_short")
    json_post "/attendance/verifications/#{id}/revoke", reason: "documento de outra pessoa"
    expect(response).to have_http_status(:ok)
    expect(citizen.reload).to be_verification_level_declared
  end

  it "admin que também é atendente não desfaz a própria validação: 403 own_verification" do
    Membership.create!(user: admin, role: "citizen_verifier", granted_at: Time.current)
    sign_in_as(admin)
    json_post "/attendance/verifications", cpf: citizen.cpf, code: issue_code_for(citizen), document_checked: true
    json_post "/attendance/verifications/#{body.dig('verification', 'id')}/revoke", reason: "documento de outra pessoa"
    expect(response).to have_http_status(:forbidden)
    expect(body["error"]).to eq("own_verification")
  end

  it "o histórico por CPF é só do admin" do
    sign_in_as(verifier)
    get "/attendance/verifications", params: { cpf: citizen.cpf }
    expect(response).to have_http_status(:forbidden)
  end

  it "escrita sem JSON é recusada" do
    sign_in_as(verifier)
    post "/attendance/lookup", params: { cpf: citizen.cpf, code: "123456" }
    expect(response).to have_http_status(:unsupported_media_type)
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/requests/attendance_spec.rb spec/requests/citizen_api/verification_codes_spec.rb`
Expected: FAIL (404 nas rotas).

- [ ] **Step 3: Implementação**

```ruby
# app/controllers/citizen_api/verification_codes_controller.rb
# POST /citizen/verification_codes {citizen_id} — "Validar no posto" (spec
# 2026-09-24 §4). O par precisa ser do celular da sessão.
module CitizenApi
  class VerificationCodesController < BaseController
    rate_limit to: 10, within: 1.hour, only: :create, name: "citizen_verification_code",
               by: -> { current_citizen_session&.id || request.remote_ip }, store: RateLimitStore,
               with: -> { render_error("too_many_requests", :too_many_requests) }

    def create
      citizen = current_citizen_session.citizens.find_by(id: params[:citizen_id])
      return render_error("not_found", :not_found) unless citizen

      result = Citizens::IssueVerificationCode.call(citizen: citizen)
      return render_error(result.reason, :conflict) if result.failure?

      render json: { code: result.payload[:code], expires_at: result.payload[:expires_at].iso8601 }, status: :created
    end
  end
end
```

```ruby
# app/controllers/attendance_controller.rb
# Balcão da UBS (spec 2026-09-24-citizen-presencial-verification §4–§7).
#   POST /attendance/lookup                   {cpf, code}                    citizen_verifier
#   POST /attendance/verifications            {cpf, code, document_checked}  citizen_verifier
#   GET  /attendance/verifications?cpf=                                      municipal_admin
#   POST /attendance/verifications/:id/revoke {reason}                       municipal_admin
class AttendanceController < ApplicationController
  include Authentication

  # Mesmo delegador de MfaController::RateLimitStore.
  module RateLimitStore
    def self.increment(...) = Rails.cache.increment(...)
  end

  ERROR_STATUS = {
    invalid_cpf: :unprocessable_entity, invalid_code: :unprocessable_entity, code_expired: :unprocessable_entity,
    code_exhausted: :unprocessable_entity, document_check_required: :unprocessable_entity,
    reason_too_short: :unprocessable_entity, already_verified: :conflict, already_revoked: :conflict,
    own_verification: :forbidden
  }.freeze

  before_action :require_verifier, only: %i[lookup verify]
  before_action :require_admin, only: %i[index revoke]

  rate_limit to: 30, within: 10.minutes, only: %i[lookup verify], name: "attendance",
             by: -> { Current.user&.id || request.remote_ip }, store: RateLimitStore,
             with: -> { render json: { error: "too_many_requests" }, status: :too_many_requests }

  def lookup
    result = Citizens::LookupForVerification.call(cpf: params[:cpf], code: params[:code])
    return render_failure(result) if result.failure?

    citizen = result.payload[:citizen]
    render json: {
      citizen: {
        id: citizen.id, cpf_masked: citizen.cpf_masked, phone_masked: CitizenIdentity::Phone.mask(citizen.phone),
        created_at: citizen.created_at.iso8601, verification_level: citizen.verification_level
      },
      triages: result.payload[:triages].map { |t| { date: t[:date].iso8601, protocol_name: t[:protocol_name] } }
    }
  end

  def verify
    result = Citizens::Verify.call(cpf: params[:cpf], code: params[:code],
                                   document_checked: params[:document_checked] == true, by: Current.user)
    return render_failure(result) if result.failure?

    v = result.payload[:verification]
    render json: { verification: { id: v.id, citizen_id: v.citizen_id, verified_at: v.verified_at.iso8601 } },
           status: :created
  end

  def index
    digits = CitizenIdentity::Cpf.normalize(params[:cpf])
    return render json: { error: "invalid_cpf" }, status: :unprocessable_entity unless digits

    rows = CitizenVerification.joins(:citizen).where(citizens: { cpf: digits })
                              .includes(:citizen, :verified_by_user, :revoked_by_user).order(verified_at: :desc)
    render json: { verifications: rows.map { |v| verification_json(v) } }
  end

  def revoke
    verification = CitizenVerification.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless verification

    result = Citizens::RevokeVerification.call(verification: verification, reason: params[:reason], by: Current.user)
    return render_failure(result) if result.failure?

    render json: { verification: verification_json(verification.reload) }
  end

  private

  def require_verifier
    forbid unless CitizenVerificationPolicy.new(Current.user, nil).verify?
  end

  def require_admin
    forbid unless CitizenVerificationPolicy.new(Current.user, nil).manage?
  end

  def forbid
    render json: { error: "forbidden" }, status: :forbidden
  end

  def render_failure(result)
    payload = { error: result.reason.to_s }
    payload[:verified_at] = result.details[:verified_at].iso8601 if result.details[:verified_at]
    render json: payload, status: ERROR_STATUS.fetch(result.reason, :unprocessable_entity)
  end

  def verification_json(v)
    {
      id: v.id, verified_at: v.verified_at.iso8601, verified_by: v.verified_by_user.email_address,
      phone_masked: CitizenIdentity::Phone.mask(v.citizen.phone), active: v.active?,
      revoked_at: v.revoked_at&.iso8601, revoked_by: v.revoked_by_user&.email_address, revoke_reason: v.revoke_reason
    }
  end
end
```

Notas:
- O `Authentication` já recusa escrita sem JSON (`require_json_for_cookie_writes`) — o teste "escrita sem JSON" deve passar sem código novo; se não passar, investigue antes de duplicar a regra.
- `params[:document_checked] == true`: com corpo JSON o valor chega booleano. `"true"` (string) não vale — é proposital.
- `Current.user` é nil numa sessão de operador por grant; o `Authentication` já recusa grant de operador fora das ações liberadas, então `require_verifier` nunca vê nil numa sessão válida.

Em `config/routes.rb`, depois do bloco `/setup`:

```ruby
  # Balcão da UBS: validação presencial do cidadão (spec 2026-09-24). Escrita
  # de servidor da cidade — fora de /admin/api, que é só leitura.
  scope "/attendance" do
    post "lookup",                   to: "attendance#lookup"
    post "verifications",            to: "attendance#verify"
    get  "verifications",            to: "attendance#index"
    post "verifications/:id/revoke", to: "attendance#revoke"
  end
```

E dentro do `scope "/citizen"` existente: `post "verification_codes", to: "verification_codes#create"`.

Em `config/initializers/filter_parameter_logging.rb`, acrescente `:reason` (o `code` já é filtrado).

- [ ] **Step 4: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/requests/attendance_spec.rb spec/requests/citizen_api spec/architecture spec/requests/cookie_write_requires_json_spec.rb`
Expected: PASS. Se um spec de `spec/architecture` pedir registro de rota nova, siga a regra que ele protege.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/attendance_controller.rb app/controllers/citizen_api/verification_codes_controller.rb config/routes.rb config/initializers/filter_parameter_logging.rb spec/requests/attendance_spec.rb spec/requests/citizen_api/verification_codes_spec.rb
/opt/homebrew/bin/git commit -m "feat: expose counter verification to city staff and codes to citizens" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Histórico completo para o par verificado

**Repo:** `apps/api`.

**Files:**
- Modify: `app/controllers/citizen_api/triages_controller.rb`
- Test: `spec/requests/citizen_api/verified_history_spec.rb`

**Interfaces:**
- Consumes: `Citizen#verification_level_verified?`, `Citizen#active_verification` (Task 1); `Citizens::Verify` (Task 3).
- Produces: `GET /citizen/triages?citizen_id=` → `citizen` ganha `verified_at` (ISO ou `null`); cada triagem ganha `origin_phone_masked` (`null` quando é do próprio par). Para um par verificado, a lista inclui as triagens web de todos os pares do mesmo CPF. `GET /citizen/triages/:id` segue o mesmo escopo. `POST /citizen/triages/:id/revoke_consent` numa triagem de outro par → 403 `not_own_triage`; `consent_active` vem `false` para triagens de outro par.

- [ ] **Step 1: Write the failing test**

```ruby
# spec/requests/citizen_api/verified_history_spec.rb
require "rails_helper"

RSpec.describe "Verified citizen history", type: :request do
  before do
    Current.city = TEST_CITY_A
    create_default_protocol!
    Rails.cache.clear
  end
  after { Current.reset }

  let!(:mine) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let!(:family) { Citizen.create!(cpf: "52998224725", phone: "+5541911112222") }
  let(:verifier) { User.create!(email_address: "atendente@cidade.gov.br", password: "senha-segura-123") }
  let(:admin) { User.create!(email_address: "admin@cidade.gov.br", password: "senha-segura-123") }
  def body = JSON.parse(response.body)

  def triage_for(citizen)
    started = Citizens::StartConversation.call(citizen: citizen, consent_version: Consents.current_version, session_id: "s").payload
    Citizens::SubmitAnswer.call(conversation: started[:conversation], answer: "false", idempotency_key: SecureRandom.uuid)
    started[:triage]
  end

  def verify(citizen)
    Citizens::Verify.call(cpf: citizen.cpf, code: issue_code_for(citizen), document_checked: true, by: verifier)
                    .payload[:verification]
  end

  it "par verificado vê as triagens de todos os pares do CPF, marcadas com o celular de origem" do
    own = triage_for(mine)
    other = triage_for(family)
    verify(mine)
    sign_in_citizen("+5541998765432")
    get "/citizen/triages", params: { citizen_id: mine.id }
    expect(body["citizen"]["verified_at"]).to be_present
    rows = body["triages"].index_by { |t| t["id"] }
    expect(rows.keys).to contain_exactly(own.id, other.id)
    expect(rows[own.id]["origin_phone_masked"]).to be_nil
    expect(rows[other.id]["origin_phone_masked"]).to eq("(**) *****-2222")
    expect(rows[other.id]["consent_active"]).to be(false)
  end

  it "par declarado do mesmo CPF continua vendo só as próprias" do
    triage_for(mine)
    other = triage_for(family)
    verify(mine)
    sign_in_citizen("+5541911112222")
    get "/citizen/triages", params: { citizen_id: family.id }
    expect(body["triages"].map { |t| t["id"] }).to eq([other.id])
    expect(body["citizen"]["verified_at"]).to be_nil
  end

  it "depois de desfeita a validação, volta a ver só as próprias" do
    own = triage_for(mine)
    triage_for(family)
    Citizens::RevokeVerification.call(verification: verify(mine), reason: "documento de outra pessoa", by: admin)
    sign_in_citizen("+5541998765432")
    get "/citizen/triages", params: { citizen_id: mine.id }
    expect(body["triages"].map { |t| t["id"] }).to eq([own.id])
  end

  it "não revoga consentimento de triagem de outro par" do
    triage_for(mine)
    other = triage_for(family)
    verify(mine)
    sign_in_citizen("+5541998765432")
    json_post "/citizen/triages/#{other.id}/revoke_consent"
    expect(response).to have_http_status(:forbidden)
    expect(body["error"]).to eq("not_own_triage")
    expect(other.conversation.reload.active_consent).to be_present
  end

  it "o par verificado abre o detalhe de uma triagem de outro par" do
    triage_for(mine)
    other = triage_for(family)
    verify(mine)
    sign_in_citizen("+5541998765432")
    get "/citizen/triages/#{other.id}"
    expect(response).to have_http_status(:ok)
    expect(body["origin_phone_masked"]).to eq("(**) *****-2222")
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec -T api bundle exec rspec spec/requests/citizen_api/verified_history_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implementação**

Em `app/controllers/citizen_api/triages_controller.rb`:

- `index`: depois de achar `citizen`, use `triages = history_scope(citizen).order(created_at: :desc)` e devolva `citizen: { id:, cpf_masked:, verification_level:, verified_at: citizen.active_verification&.verified_at&.iso8601 }` e `triages: triages.map { |t| summary(t, viewer: citizen) }`.
- `show` e `revoke_consent`: o escopo passa a ser o de todos os cidadãos visíveis pela sessão. `revoke_consent` recusa com 403 `not_own_triage` quando `triage.conversation.citizen_id` não é de um par do celular da sessão.

```ruby
    # Par verificado: histórico completo do CPF (spec 2026-09-24 §2.3, §4).
    # Par declarado: só o próprio par (spec 2026-09-22 §2.3).
    def history_scope(citizen)
      ids = citizen.verification_level_verified? ? Citizen.where(cpf: citizen.cpf).select(:id) : [citizen.id]
      Triage.joins(:conversation).where(conversations: { channel: "web", citizen_id: ids })
    end

    # Triagens que a sessão pode ver: as dos pares do celular, mais as dos
    # outros pares do CPF de cada par verificado do celular.
    def visible_triages
      own = current_citizen_session.citizens
      verified_cpfs = own.select(&:verification_level_verified?).map(&:cpf)
      ids = Citizen.where(id: own.select(:id)).or(Citizen.where(cpf: verified_cpfs)).select(:id)
      Triage.joins(:conversation).where(conversations: { channel: "web", citizen_id: ids })
    end

    def own_triage?(triage)
      current_citizen_session.citizens.exists?(id: triage.conversation.citizen_id)
    end

    def summary(triage, viewer: nil)
      own = viewer ? triage.conversation.citizen_id == viewer.id : own_triage?(triage)
      {
        id: triage.id, status: triage.status, tier: triage.tier, priority: triage.priority,
        created_at: triage.created_at.iso8601, completed_at: triage.completed_at&.iso8601,
        report_url: triage.report_snapshot&.url,
        consent_active: own && triage.conversation.active_consent.present?,
        origin_phone_masked: own ? nil : CitizenIdentity::Phone.mask(triage.conversation.citizen.phone)
      }
    end
```

Troque `scoped_triages` por `visible_triages` em `show` e `revoke_consent`. Em `revoke_consent`, antes de chamar `RevokeConsent`: `return render_error("not_own_triage", :forbidden) unless own_triage?(triage)`.

Atenção: `Citizen.where(cpf: [...])` com `cpf` cifrado de forma determinística funciona porque o Rails cifra os valores da busca; o teste do par verificado prova isso.

- [ ] **Step 4: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/requests/citizen_api`
Expected: PASS (inclusive o `isolation_spec.rb` do subprojeto 1).

- [ ] **Step 5: Suíte completa**

`docker compose stop worker`; `docker compose exec -T api bundle exec rspec`; `docker compose start worker`. Expected: 0 falhas, ~3 min.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/citizen_api/triages_controller.rb spec/requests/citizen_api/verified_history_spec.rb
/opt/homebrew/bin/git commit -m "feat: show the full CPF history to verified citizens" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Dashboard — módulo "Atendimento"

**Repo:** `apps/dashboard`. Antes: confira `main` limpo e crie `feat/citizen-presencial-verification`. Comandos: `npm test`, `npm run typecheck`, `npm run build` em `apps/dashboard`.

**Files:**
- Modify: `vite.config.ts` (proxy `/attendance`), `src/lib/api.ts`, `src/shell/modules.ts`, `src/App.tsx`
- Create: `src/lib/attendance.ts`, `src/lib/attendance.test.ts`, `src/modules/Attendance.tsx`, `src/modules/Attendance.test.tsx`

**Interfaces:**
- Consumes: rotas da Task 4.
- Produces:
  - `src/lib/api.ts`: tipos `AttendanceCitizen {id, cpf_masked, phone_masked, created_at, verification_level}`, `AttendanceTriage {date, protocol_name}`, `VerificationRow {id, verified_at, verified_by, phone_masked, active, revoked_at, revoked_by, revoke_reason}`; funções `lookupCitizen(cpf, code)`, `verifyCitizen(cpf, code)`, `listVerifications(cpf)`, `revokeVerification(id, reason)`.
  - `src/lib/attendance.ts`: `maskCpf(s)`, `isValidCpf(s)`, `attendanceError(err: unknown): string`.
  - `ModuleId` ganha `"attendance"`; grupo de menu "Atendimento" visível para `citizen_verifier` ou `municipal_admin`.

- [ ] **Step 1: Write the failing tests**

```ts
// src/lib/attendance.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import { attendanceError, isValidCpf, maskCpf } from "./attendance";

describe("attendance helpers", () => {
  it("mascara e valida CPF", () => {
    expect(maskCpf("52998224725")).toBe("529.982.247-25");
    expect(isValidCpf("529.982.247-25")).toBe(true);
    expect(isValidCpf("111.111.111-11")).toBe(false);
  });

  it("traduz os erros do balcão sem confundir com o código do autenticador", () => {
    expect(attendanceError(new ApiError(422, { error: "invalid_code" }, "x")))
      .toBe("código não confere — confira com o cidadão");
    expect(attendanceError(new ApiError(422, { error: "code_expired" }, "x")))
      .toBe("código vencido ou já usado — peça ao cidadão para gerar outro código");
    expect(attendanceError(new ApiError(422, { error: "code_exhausted" }, "x")))
      .toBe("tentativas esgotadas — peça ao cidadão para gerar outro código");
    expect(attendanceError(new ApiError(409, { error: "already_verified", verified_at: "2026-09-24T12:00:00Z" }, "x")))
      .toMatch(/^cadastro já verificado em /);
    expect(attendanceError(new ApiError(403, { error: "own_verification" }, "x")))
      .toBe("quem validou não pode desfazer a própria validação");
    expect(attendanceError(new Error("rede"))).toBe("não foi possível concluir — tente de novo");
  });
});
```

```tsx
// src/modules/Attendance.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), lookupCitizen: vi.fn(), verifyCitizen: vi.fn(),
           listVerifications: vi.fn(), revokeVerification: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { AuthProvider } from "../lib/auth";
import { Attendance } from "./Attendance";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function session(role: string): api.SessionUser {
  return { id: "u1", email_address: "a@cidade.gov.br", operator: false, mfa_enrolled: true,
           mfa_verified_at: null,
           memberships: [ { municipality_id: "m1", municipality_name: "Curitiba", municipality_uf: "PR", role } ] };
}

function renderAttendance() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  render(<Attendance />, { wrapper });
}

const found = {
  citizen: { id: "c1", cpf_masked: "***.982.247-**", phone_masked: "(**) *****-5432",
             created_at: "2026-09-20T10:00:00Z", verification_level: "declared" },
  triages: [ { date: "2026-09-21T10:00:00Z", protocol_name: "triage-respiratoria" } ]
};

describe("Attendance", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.lookupCitizen, api.verifyCitizen, api.listVerifications, api.revokeVerification ]) {
      mocked(fn).mockReset();
    }
    mocked(api.fetchCurrentSession).mockResolvedValue(session("citizen_verifier"));
  });

  it("busca, exige a caixa do documento e valida", async () => {
    mocked(api.lookupCitizen).mockResolvedValue(found);
    mocked(api.verifyCitizen).mockResolvedValue(undefined);
    renderAttendance();
    fireEvent.change(await screen.findByLabelText("CPF"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Código do cidadão"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
    expect(await screen.findByText("(**) *****-5432")).toBeInTheDocument();
    expect(screen.getByText("triage-respiratoria")).toBeInTheDocument();
    const validate = screen.getByRole("button", { name: "Validar cadastro" });
    expect(validate).toBeDisabled();
    fireEvent.click(screen.getByLabelText("Conferi o documento com foto e o CPF confere"));
    fireEvent.click(validate);
    await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456"));
    expect(await screen.findByText("Cadastro validado")).toBeInTheDocument();
    fireEvent.click(screen.getByRole("button", { name: "Próximo atendimento" }));
    expect(screen.getByLabelText("CPF")).toHaveValue("");
  });

  it("CPF inválido não chega à API", async () => {
    renderAttendance();
    fireEvent.change(await screen.findByLabelText("CPF"), { target: { value: "11111111111" } });
    fireEvent.change(screen.getByLabelText("Código do cidadão"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
    expect(await screen.findByText("CPF inválido")).toBeInTheDocument();
    expect(api.lookupCitizen).not.toHaveBeenCalled();
  });

  it("mostra a frase do erro do balcão", async () => {
    mocked(api.lookupCitizen).mockRejectedValue(new ApiError(422, { error: "code_exhausted" }, "x"));
    renderAttendance();
    fireEvent.change(await screen.findByLabelText("CPF"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Código do cidadão"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
    expect(await screen.findByText("tentativas esgotadas — peça ao cidadão para gerar outro código")).toBeInTheDocument();
  });

  it("o histórico de validações só aparece para o admin, e desfazer exige motivo", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session("municipal_admin"));
    mocked(api.listVerifications).mockResolvedValue([
      { id: "v1", verified_at: "2026-09-24T10:00:00Z", verified_by: "atendente@cidade.gov.br",
        phone_masked: "(**) *****-5432", active: true, revoked_at: null, revoked_by: null, revoke_reason: null }
    ]);
    mocked(api.revokeVerification).mockResolvedValue(undefined);
    renderAttendance();
    fireEvent.change(await screen.findByLabelText("CPF do histórico"), { target: { value: "52998224725" } });
    fireEvent.click(screen.getByRole("button", { name: "Ver histórico" }));
    fireEvent.click(await screen.findByRole("button", { name: "Desfazer" }));
    const confirm = screen.getByRole("button", { name: "Confirmar desfazer" });
    expect(confirm).toBeDisabled();
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "documento de outra pessoa" } });
    fireEvent.click(confirm);
    await waitFor(() => expect(api.revokeVerification).toHaveBeenCalledWith("v1", "documento de outra pessoa"));
  });

  it("atendente não vê o histórico", async () => {
    renderAttendance();
    await screen.findByLabelText("CPF");
    expect(screen.queryByLabelText("CPF do histórico")).not.toBeInTheDocument();
  });
});
```

Acrescente também, no teste existente do shell (ou num novo `src/shell/modules.test.ts`): `navGroupsFor` mostra "Atendimento" para `citizen_verifier` e para `municipal_admin`, e esconde para `viewer`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- src/lib/attendance.test.ts src/modules/Attendance.test.tsx`
Expected: FAIL (módulos inexistentes).

- [ ] **Step 3: Implementação**

`vite.config.ts`: no `proxy`, `"/attendance": proxy(TARGET)`.

`src/lib/api.ts` (junto das outras funções):

```ts
const ATTENDANCE_BASE = import.meta.env.VITE_ATTENDANCE_BASE || "/attendance";

export interface AttendanceCitizen {
  id: string; cpf_masked: string; phone_masked: string; created_at: string;
  verification_level: "declared" | "verified";
}
export interface AttendanceTriage { date: string; protocol_name: string }
export interface VerificationRow {
  id: string; verified_at: string; verified_by: string; phone_masked: string; active: boolean;
  revoked_at: string | null; revoked_by: string | null; revoke_reason: string | null;
}

export async function lookupCitizen(cpf: string, code: string): Promise<{ citizen: AttendanceCitizen; triages: AttendanceTriage[] }> {
  return jsonFetch(`${ATTENDANCE_BASE}/lookup`, { method: "POST", body: JSON.stringify({ cpf, code }) });
}

export async function verifyCitizen(cpf: string, code: string): Promise<void> {
  await jsonFetch<unknown>(`${ATTENDANCE_BASE}/verifications`, {
    method: "POST", body: JSON.stringify({ cpf, code, document_checked: true })
  });
}

export async function listVerifications(cpf: string): Promise<VerificationRow[]> {
  const payload = await jsonFetch<{ verifications: VerificationRow[] }>(
    `${ATTENDANCE_BASE}/verifications?cpf=${encodeURIComponent(cpf)}`);
  return payload.verifications;
}

export async function revokeVerification(id: string, reason: string): Promise<void> {
  await jsonFetch<unknown>(`${ATTENDANCE_BASE}/verifications/${encodeURIComponent(id)}/revoke`, {
    method: "POST", body: JSON.stringify({ reason })
  });
}
```

```ts
// src/lib/attendance.ts
// Balcão da UBS (spec 2026-09-24-citizen-presencial-verification §5, §6). Os
// erros daqui têm tradução própria: `invalid_code` no balcão é o código do
// CIDADÃO, não o TOTP do servidor que describeActionError traduz.
import { ApiError } from "./api";
import { fmtDateTime } from "./format";

export const onlyDigits = (s: string) => s.replace(/\D/g, "");

export function maskCpf(input: string): string {
  const d = onlyDigits(input).slice(0, 11);
  const head = [ d.slice(0, 3), d.slice(3, 6), d.slice(6, 9) ].filter(Boolean).join(".");
  return d.length > 9 ? `${head}-${d.slice(9)}` : head;
}

export function isValidCpf(input: string): boolean {
  const d = onlyDigits(input);
  if (d.length !== 11 || /^(\d)\1{10}$/.test(d)) return false;
  const n = d.split("").map(Number);
  const check = (len: number) => {
    const sum = n.slice(0, len).reduce((acc, x, i) => acc + x * (len + 1 - i), 0);
    const rest = (sum * 10) % 11;
    return rest === 10 ? 0 : rest;
  };
  return check(9) === n[9] && check(10) === n[10];
}

const GENERIC = "não foi possível concluir — tente de novo";
const MESSAGES: Record<string, string> = {
  invalid_cpf: "CPF inválido",
  invalid_code: "código não confere — confira com o cidadão",
  code_expired: "código vencido ou já usado — peça ao cidadão para gerar outro código",
  code_exhausted: "tentativas esgotadas — peça ao cidadão para gerar outro código",
  document_check_required: "marque que conferiu o documento com foto",
  already_revoked: "esta validação já foi desfeita",
  own_verification: "quem validou não pode desfazer a própria validação",
  reason_too_short: "o motivo precisa de pelo menos 10 caracteres",
  forbidden: "seu papel não permite esta ação",
  too_many_requests: "muitas tentativas — aguarde alguns minutos"
};

export function attendanceError(err: unknown): string {
  if (!(err instanceof ApiError)) return GENERIC;
  const body = (err.body ?? {}) as { error?: string; verified_at?: string };
  if (body.error === "already_verified") {
    return body.verified_at ? `cadastro já verificado em ${fmtDateTime(body.verified_at)}` : "cadastro já verificado";
  }
  if (err.status === 401) return "sessão expirada — entre de novo";
  return (body.error && MESSAGES[body.error]) || GENERIC;
}
```

(Confira a assinatura de `fmtDateTime` em `src/lib/format.ts` e ajuste se preciso.)

`src/modules/Attendance.tsx` — componente com dois blocos, usando `PageHeader`, `Panel`, `DataTable`, `Tag`, `buttonStyle`/`inputStyle` de `components/formStyles` e `useAuth()` para saber os papéis:

- **Balcão** (visível para `citizen_verifier`): estados `form → found → done`.
  - `form`: inputs rotulados "CPF" (valor com `maskCpf`) e "Código do cidadão" (só dígitos, 6), botão "Buscar". Se `!isValidCpf(cpf)`, mostra "CPF inválido" (`role="alert"`) e não chama a API. Senão `lookupCitizen(cpf, code)`; erro → `attendanceError(err)` em `role="alert"`.
  - `found`: `KeyValue`/linhas com CPF mascarado, celular mascarado, "cadastrado em" (`fmtDateTime`), nível (`Tag`); `DataTable` com colunas "Data" e "Protocolo" (vazio: "nenhuma triagem"); checkbox rotulado "Conferi o documento com foto e o CPF confere"; botão "Validar cadastro" desabilitado até marcar; ao clicar, `verifyCitizen(cpf, code)` → `done`; erro → mensagem.
  - `done`: texto "Cadastro validado" (`role="status"`) e botão "Próximo atendimento", que volta a `form` com tudo limpo.
- **Histórico de validações** (visível só para `municipal_admin`): input rotulado "CPF do histórico", botão "Ver histórico" → `listVerifications(cpf)`; `DataTable` com Data, Servidor, Celular, Situação ("ativa" ou "desfeita em … por … — motivo"); em linhas ativas, botão "Desfazer" abre um painel com textarea rotulada "Motivo" e botão "Confirmar desfazer" desabilitado enquanto `motivo.trim().length < 10`; confirmar chama `revokeVerification(id, motivo)` e recarrega a lista; erro → `attendanceError`.

`src/shell/modules.ts`: acrescente `"attendance"` a `ModuleId`; novo grupo `{ label: "Atendimento", items: [{ id: "attendance", label: "Atendimento", icon: "☑" }] }` antes de "Equipe"; em `navGroupsFor`:

```ts
  const roles = user?.memberships?.map((m) => m.role) ?? [];
  const isAdmin = roles.includes("municipal_admin");
  const canAttend = isAdmin || roles.includes("citizen_verifier");
  // ...
    if (group.label === "Atendimento") return canAttend;
```

`src/App.tsx`: `case "attendance": return <Attendance />;` (import de `./modules/Attendance`).

- [ ] **Step 4: Run tests**

Run: `npm test && npm run typecheck && npm run build`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add vite.config.ts src/lib/api.ts src/lib/attendance.ts src/lib/attendance.test.ts src/modules/Attendance.tsx src/modules/Attendance.test.tsx src/shell/modules.ts src/App.tsx
/opt/homebrew/bin/git commit -m "feat: add the attendance module for counter verification" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Se criou `src/shell/modules.test.ts`, inclua no `git add`.)

---

### Task 7: Dashboard — conceder "atendente" na Equipe

**Repo:** `apps/dashboard`.

**Files:**
- Modify: `src/lib/team.ts`, `src/modules/Team.tsx`, `src/modules/Team.test.tsx`

**Interfaces:**
- Consumes: `grantRole`, `revokeMembership`, `SensitiveAction` (existentes).
- Produces: `VERIFIER_ROLE = "citizen_verifier"`; `TeamMember` ganha `isVerifier: boolean` e `verifierMembershipId: string | null`.

- [ ] **Step 1: Write the failing test** (em `src/modules/Team.test.tsx`, no mesmo estilo dos testes existentes)

```tsx
  it("torna atendente com step-up e remove atendente", async () => {
    mocked(api.listMemberships).mockResolvedValue([
      membership("ana@cidade.gov.br", "viewer"),
      membership("bia@cidade.gov.br", "citizen_verifier", "m-bia")
    ]);
    mocked(api.grantRole).mockResolvedValue(undefined);
    mocked(api.revokeMembership).mockResolvedValue(undefined);
    renderTeam();

    fireEvent.click(await screen.findByRole("button", { name: "Tornar atendente" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.grantRole).toHaveBeenCalledWith("u-ana@cidade.gov.br", "citizen_verifier"));

    fireEvent.click(await screen.findByRole("button", { name: "Remover atendente" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.revokeMembership).toHaveBeenCalledWith("m-bia"));
  });
```

(A sessão padrão do teste já vem com `mfa_verified_at` recente, então o `SensitiveAction` não pede código. Se o nome do botão de confirmar for outro, use o que o `SensitiveAction` mostra — `confirmLabel` padrão é "Confirmar".)

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- src/modules/Team.test.tsx`
Expected: FAIL (não há "Tornar atendente").

- [ ] **Step 3: Implementação**

`src/lib/team.ts`:

```ts
export const VERIFIER_ROLE = "citizen_verifier";
```

e, em `TeamMember`, `isVerifier: boolean; verifierMembershipId: string | null;` (inicializados `false`/`null` em `teamMembers`, preenchidos quando `row.role === VERIFIER_ROLE`, como o revisor).

`src/modules/Team.tsx`:
- `type Pending = { member: TeamMember; kind: "grant" | "revoke"; role: "reviewer" | "verifier" }`.
- Nova coluna "Atendente" (`w: "auto"`, `align: "right"`): "Remover atendente" quando `m.isVerifier`, senão "Tornar atendente".
- O bloco `pending` escolhe título, descrição, papel e id pela `role`:
  - grant verifier: título "Tornar atendente", descrição "`<email>` poderá validar cadastros de cidadãos no balcão da UBS.", `grantRole(userId, VERIFIER_ROLE)`, mensagem "`<email>` agora é atendente".
  - revoke verifier: título "Remover atendente", descrição "`<email>` deixa de validar cadastros. As validações que já fez continuam valendo.", `revokeMembership(verifierMembershipId!)`, mensagem "`<email>` não é mais atendente".
  - `requiresStepUp` em todos (o papel é privilegiado na API).
- Atualize o `sub` do `PageHeader` para "papéis · revisores de protocolo · atendentes" e o comentário de escopo no topo do arquivo.

- [ ] **Step 4: Run tests**

Run: `npm test && npm run typecheck`
Expected: PASS (os testes existentes do revisor continuam passando).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/team.ts src/modules/Team.tsx src/modules/Team.test.tsx
/opt/homebrew/bin/git commit -m "feat: let admins grant and revoke the attendant role" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: wpda — "Validar no posto", selo e histórico completo

**Repo:** `apps/wpda`. Antes: confira `main` limpo e crie `feat/citizen-presencial-verification`. Comandos: `npm test`, `npm run typecheck`, `npm run build`.

**Files:**
- Modify: `src/lib/citizenApi.ts`, `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/triage.test.tsx`
- Create: `src/modules/citizen/VerificationCodeStep.tsx`, `src/modules/citizen/VerificationCodeStep.test.tsx`

**Interfaces:**
- Consumes: `POST /citizen/verification_codes`, campos novos de `GET /citizen/triages` (Tasks 4–5).
- Produces: `citizenApi.issueVerificationCode(citizenId) -> Promise<{ code: string; expires_at: string }>`; `Person` ganha `verified_at?: string | null`; `TriageSummary` ganha `origin_phone_masked: string | null`; `HistoryStep` ganha a prop `onValidate(citizenId)`; `VerificationCodeStep({ citizenId, onBack })`; `Flow` ganha o estado `{ at: "verify-code"; citizenId; consentVersion }`.

- [ ] **Step 1: Write the failing tests**

```tsx
// src/modules/citizen/VerificationCodeStep.test.tsx
import { describe, it, expect, vi, afterEach } from "vitest";
import { act, render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { VerificationCodeStep } from "./VerificationCodeStep";
import { citizenApi } from "../../lib/citizenApi";

afterEach(() => { vi.restoreAllMocks(); vi.useRealTimers(); });

describe("VerificationCodeStep", () => {
  it("mostra o código, a instrução e a contagem, e gera outro", async () => {
    const spy = vi.spyOn(citizenApi, "issueVerificationCode")
      .mockResolvedValueOnce({ code: "123456", expires_at: new Date(Date.now() + 600_000).toISOString() })
      .mockResolvedValueOnce({ code: "654321", expires_at: new Date(Date.now() + 600_000).toISOString() });
    render(<VerificationCodeStep citizenId="p1" onBack={vi.fn()} />);
    expect(await screen.findByText("123456")).toBeInTheDocument();
    expect(screen.getByText("Mostre este código e um documento com foto ao atendente.")).toBeInTheDocument();
    expect(screen.getByText(/1\d:\d\d|10:00|9:\d\d/)).toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Gerar outro código" }));
    expect(await screen.findByText("654321")).toBeInTheDocument();
    expect(spy).toHaveBeenCalledWith("p1");
  });

  it("código vencido mostra o aviso e deixa gerar outro", async () => {
    vi.spyOn(citizenApi, "issueVerificationCode")
      .mockResolvedValue({ code: "123456", expires_at: new Date(Date.now() - 1000).toISOString() });
    render(<VerificationCodeStep citizenId="p1" onBack={vi.fn()} />);
    expect(await screen.findByText("Este código venceu.")).toBeInTheDocument();
  });
});
```

Em `src/modules/citizen/triage.test.tsx`, acrescente ao `describe("HistoryStep")`:

```tsx
  it("cadastro declarado oferece 'Validar no posto'", async () => {
    vi.spyOn(citizenApi, "triages").mockResolvedValue({
      citizen: { id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared", verified_at: null },
      triages: []
    });
    const onValidate = vi.fn();
    render(<HistoryStep citizenId="p1" onBack={vi.fn()} onValidate={onValidate} />);
    await userEvent.click(await screen.findByRole("button", { name: "Validar no posto" }));
    expect(onValidate).toHaveBeenCalledWith("p1");
  });

  it("cadastro verificado mostra o selo e a origem das triagens de outro celular, sem revogar", async () => {
    vi.spyOn(citizenApi, "triages").mockResolvedValue({
      citizen: { id: "p1", cpf_masked: "***.982.247-**", verification_level: "verified", verified_at: "2026-09-24T12:00:00Z" },
      triages: [{ id: "t2", status: "completed", tier: "baixa", priority: 9, created_at: "2026-09-20T12:00:00Z",
                  completed_at: "2026-09-20T12:05:00Z", report_url: null, consent_active: false,
                  origin_phone_masked: "(**) *****-2222" }]
    });
    render(<HistoryStep citizenId="p1" onBack={vi.fn()} onValidate={vi.fn()} />);
    expect(await screen.findByText(/Cadastro verificado em/)).toBeInTheDocument();
    expect(screen.getByText("Feita no celular (**) *****-2222")).toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Validar no posto" })).not.toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Revogar consentimento" })).not.toBeInTheDocument();
  });
```

Atualize também os usos existentes de `HistoryStep` nos testes para passar `onValidate={vi.fn()}` e as fixtures de `TriageSummary` para incluir `origin_phone_masked: null`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- src/modules/citizen`
Expected: FAIL.

- [ ] **Step 3: Implementação**

`src/lib/citizenApi.ts`: `Person` ganha `verified_at?: string | null`; `TriageSummary` ganha `origin_phone_masked: string | null`; em `citizenApi`:

```ts
  issueVerificationCode: (citizenId: string) =>
    call<{ code: string; expires_at: string }>("POST", "/verification_codes", { citizen_id: citizenId }),
```

```tsx
// src/modules/citizen/VerificationCodeStep.tsx
// "Validar no posto" (spec 2026-09-24-citizen-presencial-verification §5): o
// código que o cidadão mostra ao atendente, com a contagem até vencer.
import { useEffect, useState } from "react";
import { citizenApi } from "../../lib/citizenApi";
import { BigButton, ErrorText, Screen, messageFor } from "./ui";

function remaining(expiresAt: string): number {
  return Math.max(0, Math.floor((new Date(expiresAt).getTime() - Date.now()) / 1000));
}

export function VerificationCodeStep({ citizenId, onBack }: { citizenId: string; onBack: () => void }) {
  const [data, setData] = useState<{ code: string; expires_at: string } | null>(null);
  const [left, setLeft] = useState(0);
  const [error, setError] = useState<string | null>(null);

  async function issue() {
    setError(null);
    try {
      const d = await citizenApi.issueVerificationCode(citizenId);
      setData(d);
      setLeft(remaining(d.expires_at));
    } catch (e) {
      setError(messageFor(e));
    }
  }

  useEffect(() => { void issue(); }, [citizenId]);

  useEffect(() => {
    if (!data || left <= 0) return;
    const t = setTimeout(() => setLeft(remaining(data.expires_at)), 1000);
    return () => clearTimeout(t);
  }, [data, left]);

  const mm = Math.floor(left / 60);
  const ss = String(left % 60).padStart(2, "0");

  return (
    <Screen title="Validar no posto"
      footer={<>
        <BigButton onClick={issue}>Gerar outro código</BigButton>
        <BigButton variant="secondary" onClick={onBack}>Voltar</BigButton>
      </>}>
      {error && <ErrorText>{error}</ErrorText>}
      {data && (
        <>
          <p>Mostre este código e um documento com foto ao atendente.</p>
          <p aria-live="polite" style={{ fontSize: 48, fontWeight: 700, letterSpacing: 8, textAlign: "center", margin: "24px 0" }}>
            {data.code}
          </p>
          {left > 0
            ? <p style={{ textAlign: "center" }}>Vale por mais {mm}:{ss}</p>
            : <p role="alert" style={{ textAlign: "center" }}>Este código venceu.</p>}
        </>
      )}
    </Screen>
  );
}
```

`HistoryStep.tsx`:
- Prop nova `onValidate: (citizenId: string) => void`.
- `declared`: mantém o aviso e acrescenta `<BigButton onClick={() => onValidate(citizenId)}>Validar no posto</BigButton>`.
- `verified`: no lugar do aviso, `<p>Cadastro verificado em {fmtDateTime(data.citizen.verified_at)}</p>` (se `verified_at` vier nulo, "Cadastro verificado").
- Em cada item: se `t.origin_phone_masked`, mostra `<div>Feita no celular {t.origin_phone_masked}</div>`; o botão "Revogar consentimento" só aparece se `t.consent_active && !t.origin_phone_masked`.

`Flow.tsx`: estado novo `{ at: "verify-code"; citizenId: string; consentVersion: string | null }`; o caso `"history"` passa `onValidate={(id) => setState({ at: "verify-code", citizenId: id, consentVersion: state.consentVersion })}`; o caso `"verify-code"` renderiza `<VerificationCodeStep citizenId={state.citizenId} onBack={() => setState({ at: "history", citizenId: state.citizenId, consentVersion: state.consentVersion })} />`.

Em `ui.tsx`, `MESSAGES` ganha `already_verified: "Seu cadastro já está verificado."`.

- [ ] **Step 4: Run tests**

Run: `npm test && npm run typecheck && npm run build`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/citizenApi.ts src/modules/citizen/HistoryStep.tsx src/modules/citizen/Flow.tsx src/modules/citizen/ui.tsx src/modules/citizen/triage.test.tsx src/modules/citizen/VerificationCodeStep.tsx src/modules/citizen/VerificationCodeStep.test.tsx
/opt/homebrew/bin/git commit -m "feat: let citizens show a counter code and see their full history" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Verificação de ponta a ponta em desenvolvimento

Sem código novo, a menos que a verificação ache defeito (nesse caso, corrija na task dona, com teste).

- [ ] **Step 1:** `docker compose exec -T api bin/rails city:migrate:all` (curitiba e maringa → `20260924000001`); `docker compose restart api worker`. Se o wpda ou o dashboard do Vite não repassarem as rotas novas, `docker compose restart wpda dashboard`.
- [ ] **Step 2:** Em Curitiba, entre no dashboard como `municipal_admin` de dev (semente `SignatureCrew`, `lib/signature_crew.rb`), faça step-up e, na Equipe, torne atendente uma segunda conta de dev.
- [ ] **Step 3:** No wpda (`http://curitiba.localhost:5176/wpda/`), entre com um celular de teste, abra "Minhas triagens", toque "Validar no posto" e anote o código da tela.
- [ ] **Step 4:** No dashboard, como a conta atendente, abra "Atendimento", busque pelo CPF + código, confira que aparecem só celular mascarado e data/protocolo das triagens, marque a caixa e valide.
- [ ] **Step 5:** No wpda, recarregue "Minhas triagens": selo "Cadastro verificado em …"; se o mesmo CPF tiver triagem noutro celular de teste, ela aparece com "Feita no celular …" e sem "Revogar consentimento".
- [ ] **Step 6:** No dashboard, como `municipal_admin`, "Histórico de validações" pelo CPF → "Desfazer" com motivo; confira no wpda que o cadastro volta a "não verificado".
- [ ] **Step 7:** Relatório com o que foi testado e prints das telas principais (sem códigos em claro).
