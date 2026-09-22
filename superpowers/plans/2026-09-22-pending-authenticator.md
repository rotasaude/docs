# Troca de autenticador com segredo pendente — plano

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** trocar o autenticador do usuário da cidade deixa de derrubar o antigo: `POST /mfa/enroll` grava um segredo **pendente**, e só `POST /mfa/confirm` promove. Junto, um código TOTP passa a valer uma vez só.

**Architecture:** quatro colunas novas em `users`, no banco de cada cidade (`otp_pending_secret`, `otp_pending_recovery_codes`, `otp_pending_at`, `last_otp_step`). Um serviço novo, `Mfa::PendingEnrollment`, faz propor e promover; o `MfaController` passa a chamá-lo e a recusar código repetido. `Mfa::Enroll` fica intocado (mantenedor e operador). O dashboard perde os textos que deixam de ser verdade e ganha duas mensagens.

**Tech Stack:** Rails 8.1, PostgreSQL, ROTP, BCrypt (apps/api); Vite + React 18 + Vitest (apps/dashboard).

**Spec:** `docs/superpowers/specs/2026-09-22-pending-authenticator-design.md` (commit `470dd41`). Este plano implementa §3 a §7 dele.

## Global Constraints

- Commits: Conventional Commits **em inglês**, tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Git: `/opt/homebrew/bin/git` (o `git` do PATH aborta). `apps/api` e `apps/dashboard` são repositórios separados; a raiz do monorepo não é git. Branch `feat/pending-authenticator` em cada um. Nunca em `main`, nunca push.
- Staging de commit sempre explícito: nunca `git add -A` nem `git add .` (há um diretório de rascunho `.superpowers/` fora do git em apps/dashboard).
- Suíte completa do api: na raiz do monorepo, `docker compose stop worker`, depois `docker compose exec -T api bundle exec rspec`, depois `docker compose start worker` (sempre religar, mesmo em falha). Acima de ~3 min é regressão.
- Specs de request exigem `type: :request`; `sign_in_as(user)` vem de `spec/support/city_request_auth.rb`.
- **Migração de cidade:** arquivo em `db/city_migrate`, e `db/city_schema.rb` atualizado **à mão** (é assim que as migrações de cidade anteriores fizeram). A spec `spec/services/city_schema_spec.rb` ("produces from the migrations exactly the schema of db/city_schema.rb") é quem prova que os dois batem.
- **BCrypt sempre pelo custo de `Mfa::Enroll.recovery_code_cost`** (custo mínimo em teste). Custo fixo 12 já deixou a suíte em 10 minutos uma vez.
- Nunca imprima segredo, chave TOTP ou código de recuperação em log, relatório ou mensagem de commit.
- Nunca rode `start.sh`. Nunca desligue trigger de imutabilidade.
- Prazo do pendente: **15 minutos**. Janela de step-up: 5 minutos (`MfaStepUp::STEP_UP_WINDOW`), inalterada.

## File Structure

**apps/api**
- Create: `db/city_migrate/20260922000001_add_pending_authenticator_to_users.rb`
- Modify: `db/city_schema.rb` (à mão)
- Modify: `app/models/user.rb`, com `encrypts :otp_pending_secret` e o consumo de passo
- Create: `spec/models/user_totp_spec.rb`
- Create: `app/services/mfa/pending_enrollment.rb`
- Modify: `app/services/mfa/verify.rb`, com `step_for_secret`
- Create: `spec/services/mfa/pending_enrollment_spec.rb`
- Modify: `app/controllers/mfa_controller.rb`
- Modify/Create: `spec/requests/mfa_enroll_step_up_spec.rb` e `spec/requests/mfa_pending_enrollment_spec.rb`

**apps/dashboard**
- Modify: `src/lib/actionErrors.ts` e `src/lib/actionErrors.test.ts`
- Modify: `src/modules/Security.tsx` e `src/modules/Security.test.tsx`

---

### Task 1: colunas de cidade e consumo de passo do TOTP

**Files:**
- Create: `apps/api/db/city_migrate/20260922000001_add_pending_authenticator_to_users.rb`
- Modify: `apps/api/db/city_schema.rb` (à mão), `apps/api/app/models/user.rb`
- Test: `apps/api/spec/models/user_totp_spec.rb`

**Interfaces:**
- Consumes: `Mfa::Verify.totp_step_for(user, code)` (`app/services/mfa/verify.rb`), que devolve o número do passo de 30 s ou `nil`; o desenho de `Maintainer#consume_totp!` (`app/models/maintainer.rb:75-83`).
- Produces:
  ```ruby
  # colunas novas em users (banco da cidade)
  otp_pending_secret :string          # cifrada por `encrypts`
  otp_pending_recovery_codes :jsonb   # default [], null: false
  otp_pending_at :datetime
  last_otp_step :integer

  User#consume_totp_step!(step) -> true/false   # grava o passo; false se repetido ou anterior
  User#consume_totp!(code)      -> true/false   # passo do segredo ATIVO + consume_totp_step!
  ```

- [ ] **Step 1: Branch**

```bash
cd apps/api && /opt/homebrew/bin/git checkout -b feat/pending-authenticator
```

- [ ] **Step 2: Escrever a spec que falha**

`spec/models/user_totp_spec.rb`:

```ruby
require "rails_helper"

# Um código de TOTP vale UMA vez para a conta, em qualquer endpoint (spec do
# autenticador pendente §3). Mesmo desenho de Maintainer#consume_totp!: guarda
# o PASSO consumido, nunca o código, e a própria gravação condicional é o
# teste — duas requisições simultâneas com o mesmo código não passam as duas.
RSpec.describe User, "consumo de passo do TOTP", type: :model do
  let(:user) do
    u = User.create!(email_address: "ana-#{SecureRandom.hex(3)}@example.org", password: "secret123")
    Mfa::Enroll.call(u)
    u.update!(otp_enabled: true)
    u.reload
  end

  def current_code = ROTP::TOTP.new(user.otp_secret).now

  it "aceita um código novo e recusa o mesmo código de novo" do
    code = current_code

    expect(user.consume_totp!(code)).to be(true)
    expect(user.reload.last_otp_step).to be_present
    expect(user.consume_totp!(code)).to be(false)
  end

  it "recusa um passo anterior ao último consumido" do
    step = Mfa::Verify.totp_step_for(user, current_code)
    user.update!(last_otp_step: step)

    expect(user.consume_totp_step!(step - 1)).to be(false)
    expect(user.reload.last_otp_step).to eq(step)
  end

  it "aceita o passo seguinte" do
    step = Mfa::Verify.totp_step_for(user, current_code)
    user.update!(last_otp_step: step)

    expect(user.consume_totp_step!(step + 1)).to be(true)
    expect(user.reload.last_otp_step).to eq(step + 1)
  end

  it "recusa um código que não é do segredo da conta" do
    expect(user.consume_totp!(ROTP::TOTP.new(ROTP::Base32.random).now)).to be(false)
  end

  it "só um vencedor quando o mesmo passo chega duas vezes pelo mesmo registro" do
    step = Mfa::Verify.totp_step_for(user, current_code)
    other = User.find(user.id)   # segunda instância, como duas requisições

    expect([ user.consume_totp_step!(step), other.consume_totp_step!(step) ].count(true)).to eq(1)
  end

  it "guarda o pendente sem tocar no segredo ativo" do
    active = user.otp_secret
    user.update!(otp_pending_secret: ROTP::Base32.random, otp_pending_at: Time.current)

    expect(user.reload.otp_secret).to eq(active)
    expect(user.otp_enabled).to be(true)
    expect(user.otp_pending_secret).to be_present
  end
end
```

- [ ] **Step 3: Rodar e ver falhar**

Da raiz do monorepo:
```bash
docker compose exec -T api bundle exec rspec spec/models/user_totp_spec.rb
```
Esperado: FAIL — `undefined method 'consume_totp!'` e coluna `otp_pending_secret` inexistente.

- [ ] **Step 4: Migração**

`db/city_migrate/20260922000001_add_pending_authenticator_to_users.rb`:

```ruby
# Autenticador pendente (spec 2026-09-22-pending-authenticator-design §3).
#
# Aditiva e nula por padrão: o segredo ATIVO (otp_secret, otp_recovery_codes,
# otp_enabled) não é tocado, e conta já cadastrada segue funcionando sem nada
# a migrar. `last_otp_step` é o mesmo campo que Maintainer usa para recusar a
# repetição de um código dentro da janela de drift.
class AddPendingAuthenticatorToUsers < ActiveRecord::Migration[8.1]
  def change
    add_column :users, :otp_pending_secret, :string
    add_column :users, :otp_pending_recovery_codes, :jsonb, null: false, default: []
    add_column :users, :otp_pending_at, :datetime
    add_column :users, :last_otp_step, :integer
  end
end
```

- [ ] **Step 5: Atualizar o dump à mão**

`db/city_schema.rb` é mantido **à mão** neste projeto (foi assim nas migrações de cidade anteriores; a lambda `load_city_schema` de `lib/tasks/city.rake` só o carrega). Duas edições:

1. no bloco `create_table "users"`, acrescente as quatro colunas na **ordem alfabética** que o dump já usa nas outras colunas:

```ruby
    t.integer "last_otp_step"
    t.datetime "otp_pending_at"
    t.jsonb "otp_pending_recovery_codes", default: [], null: false
    t.string "otp_pending_secret"
```

2. no topo, suba a versão de `define(version: ...)` para `2026_09_22_000001`.

Quem prova que ficou igual é a spec de paridade (`spec/services/city_schema_spec.rb`, "produces from the migrations exactly the schema of db/city_schema.rb"), que compara um banco migrado com um carregado do dump. Ela roda no Step 7.

- [ ] **Step 6: Modelo**

Em `app/models/user.rb`, ao lado de `encrypts :otp_secret`:

```ruby
  encrypts :otp_pending_secret
```

e, junto dos outros métodos públicos:

```ruby
  # Um código de TOTP vale uma vez só para esta conta, em qualquer endpoint
  # (spec do autenticador pendente §3/§4). Guarda o PASSO de 30 s consumido, e
  # nunca o código. A gravação condicional É o teste: `update_all` devolve 1
  # só quando o passo é maior que o último, então duas requisições com o mesmo
  # código não passam as duas, e um passo mais velho (ainda válido pela
  # tolerância de relógio) também é recusado. Mesmo desenho de
  # Maintainer#consume_totp!.
  def consume_totp_step!(step)
    return false if step.nil?

    self.class.where(id: id)
        .where("last_otp_step IS NULL OR last_otp_step < ?", step)
        .update_all(last_otp_step: step, updated_at: Time.current) == 1
  end

  # Consome um código do segredo ATIVO. A confirmação de matrícula usa o
  # PENDENTE e passa por Mfa::PendingEnrollment.
  def consume_totp!(code)
    consume_totp_step!(Mfa::Verify.totp_step_for(self, code))
  end
```

- [ ] **Step 7: Rodar e ver passar**

```bash
docker compose exec -T api bundle exec rspec spec/models/user_totp_spec.rb spec/services/city_schema_spec.rb
```
Esperado: PASS. O exemplo de paridade do dump é o que prova o Step 5.

- [ ] **Step 8: Suíte completa** (worker parado; ver Global Constraints). Esperado: 0 falhas.

- [ ] **Step 9: Commit**

```bash
/opt/homebrew/bin/git add db/city_migrate/20260922000001_add_pending_authenticator_to_users.rb db/city_schema.rb app/models/user.rb spec/models/user_totp_spec.rb
/opt/homebrew/bin/git commit -m "feat: add pending authenticator columns and single-use TOTP steps" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: `Mfa::PendingEnrollment` e o `MfaController`

**Files:**
- Create: `apps/api/app/services/mfa/pending_enrollment.rb`
- Modify: `apps/api/app/services/mfa/verify.rb`, `apps/api/app/controllers/mfa_controller.rb`
- Test: `apps/api/spec/services/mfa/pending_enrollment_spec.rb`, `apps/api/spec/requests/mfa_pending_enrollment_spec.rb`, e ajuste de `apps/api/spec/requests/mfa_enroll_step_up_spec.rb`

**Interfaces:**
- Consumes:
  - `User#consume_totp_step!` (Task 1);
  - `Mfa::Enroll.recovery_code_cost` (já público em `app/services/mfa/enroll.rb`), que respeita `ActiveModel::SecurePassword.min_cost`;
  - `Mfa::Verify.consume_recovery_code(user, code)` (já público);
  - `MfaStepUp#require_step_up!` / `#reauthenticated_recently?`.
- Produces:
  ```ruby
  Mfa::Verify.step_for_secret(secret, code) -> Integer | nil   # passo de um segredo qualquer
  Mfa::PendingEnrollment::TTL = 15.minutes
  Mfa::PendingEnrollment.start(user) -> { otpauth_uri:, recovery_codes: [10 strings] }
  Mfa::PendingEnrollment.confirm(user, code:) -> :ok | :no_pending_enrollment | :enrollment_expired | :invalid_code | :code_reused
  ```
  Respostas HTTP: `POST /mfa/confirm` responde `200 { ok: true }` ou `422 { error: <o símbolo acima> }`; `POST /mfa/step_up` ganha `422 { error: "code_reused" }`.

- [ ] **Step 1: Escrever a spec do serviço (falha)**

`spec/services/mfa/pending_enrollment_spec.rb`:

```ruby
require "rails_helper"

# Propor e promover (spec do autenticador pendente §4): `start` nunca toca no
# segredo ativo, e só `confirm` promove. É isso que permite abandonar uma troca
# no meio sem ficar sem segundo fator.
RSpec.describe Mfa::PendingEnrollment do
  let(:user) do
    u = User.create!(email_address: "bia-#{SecureRandom.hex(3)}@example.org", password: "secret123")
    Mfa::Enroll.call(u)
    u.update!(otp_enabled: true)
    u.reload
  end

  def pending_code(u) = ROTP::TOTP.new(u.reload.otp_pending_secret).now

  describe ".start" do
    it "grava o pendente e não toca no ativo" do
      active_secret = user.otp_secret
      active_codes = user.otp_recovery_codes

      out = described_class.start(user)

      expect(out[:otpauth_uri]).to start_with("otpauth://totp/")
      expect(out[:recovery_codes].length).to eq(10)
      user.reload
      expect(user.otp_secret).to eq(active_secret)
      expect(user.otp_recovery_codes).to eq(active_codes)
      expect(user.otp_enabled).to be(true)
      expect(user.otp_pending_secret).to be_present
      expect(user.otp_pending_recovery_codes.length).to eq(10)
      expect(user.otp_pending_at).to be_within(5.seconds).of(Time.current)
    end

    it "o otpauth_uri leva o segredo pendente, não o ativo" do
      out = described_class.start(user)

      secret = URI.decode_www_form(URI.parse(out[:otpauth_uri]).query).to_h["secret"]
      expect(secret).to eq(user.reload.otp_pending_secret)
      expect(secret).not_to eq(user.otp_secret)
    end

    it "chamar de novo substitui o pendente" do
      first = described_class.start(user)[:recovery_codes]
      described_class.start(user)

      expect(user.reload.otp_pending_recovery_codes.length).to eq(10)
      expect(first.any? { |c| user.otp_pending_recovery_codes.any? { |h| BCrypt::Password.new(h) == c } }).to be(false)
    end
  end

  describe ".confirm" do
    it "promove o pendente e limpa" do
      codes = described_class.start(user)[:recovery_codes]
      pending_secret = user.reload.otp_pending_secret

      expect(described_class.confirm(user, code: pending_code(user))).to eq(:ok)

      user.reload
      expect(user.otp_secret).to eq(pending_secret)
      expect(user.otp_enabled).to be(true)
      expect(user.otp_pending_secret).to be_nil
      expect(user.otp_pending_recovery_codes).to eq([])
      expect(user.otp_pending_at).to be_nil
      expect(user.otp_recovery_codes.any? { |h| BCrypt::Password.new(h) == codes.first }).to be(true)
    end

    it "recusa sem pendente" do
      expect(described_class.confirm(user, code: "123456")).to eq(:no_pending_enrollment)
    end

    it "recusa e limpa um pendente vencido" do
      described_class.start(user)
      user.update!(otp_pending_at: 16.minutes.ago)

      expect(described_class.confirm(user, code: pending_code(user))).to eq(:enrollment_expired)
      expect(user.reload.otp_pending_secret).to be_nil
    end

    it "recusa o código do segredo ATIVO" do
      described_class.start(user)

      expect(described_class.confirm(user, code: ROTP::TOTP.new(user.otp_secret).now)).to eq(:invalid_code)
      expect(user.reload.otp_pending_secret).to be_present
    end

    it "recusa um código já usado" do
      described_class.start(user)
      code = pending_code(user)
      expect(described_class.confirm(user, code: code)).to eq(:ok)
      described_class.start(user)

      # O passo já foi consumido na promoção acima; o mesmo instante não serve
      # duas vezes, nem para um pendente novo.
      expect(described_class.confirm(user, code: pending_code(user))).to eq(:code_reused)
    end

    it "recusa um código de recuperação pendente (confirmar prova o autenticador novo)" do
      codes = described_class.start(user)[:recovery_codes]

      expect(described_class.confirm(user, code: codes.first)).to eq(:invalid_code)
    end
  end
end
```

O exemplo "recusa um código já usado" depende de o segundo `pending_code` cair no mesmo passo de 30 s do primeiro. Se ele oscilar na virada do passo, envolva as duas confirmações em `travel_to(Time.current)` (`ActiveSupport::Testing::TimeHelpers`, como `spec/requests/protocol_lifecycle_spec.rb` já usa) em vez de afrouxar a asserção.

- [ ] **Step 2: Rodar e ver falhar**

```bash
docker compose exec -T api bundle exec rspec spec/services/mfa/pending_enrollment_spec.rb
```
Esperado: FAIL — `uninitialized constant Mfa::PendingEnrollment`.

- [ ] **Step 3: `step_for_secret` em `Mfa::Verify`**

Em `app/services/mfa/verify.rb`, acrescente o método e faça `totp_step_for` delegar, sem duplicar a regra de drift:

```ruby
    # O passo de um segredo QUALQUER — a confirmação de matrícula verifica
    # contra o segredo PENDENTE, que não está em nenhum atributo do modelo.
    def self.step_for_secret(secret, code)
      return nil if secret.blank? || code.blank?

      totp = ROTP::TOTP.new(secret)
      at = totp.verify(code.to_s.gsub(/\s+/, ""), drift_behind: DRIFT, drift_ahead: DRIFT)
      at && (at.to_i / totp.interval)
    end
```

e troque o corpo de `totp_step_for` por `step_for_secret(user.otp_secret, code)`, mantendo o comentário que já existe lá.

- [ ] **Step 4: Implementar o serviço**

`app/services/mfa/pending_enrollment.rb`:

```ruby
require "rotp"

# Matrícula em duas etapas do usuário da cidade (spec 2026-09-22-pending-
# authenticator-design §4): `start` PROPÕE um segredo, `confirm` PROMOVE.
#
# Enquanto não houver confirmação, o autenticador ativo continua valendo — é o
# que permite abandonar uma troca no meio sem ficar sem segundo fator, e o que
# impede quem tem só a senha de plantar um segredo e confirmá-lo (numa conta já
# cadastrada, MfaController#enroll exige step-up).
#
# Mantenedor e operador continuam em Mfa::Enroll: lá a matrícula nasce de um
# convite, e reenviar o convite É o caminho de recuperação.
module Mfa
  module PendingEnrollment
    RECOVERY_COUNT = 10
    RECOVERY_LEN   = 10
    TTL = 15.minutes

    def self.start(user)
      secret = ROTP::Base32.random
      codes  = Array.new(RECOVERY_COUNT) { SecureRandom.alphanumeric(RECOVERY_LEN).downcase }

      user.update!(
        otp_pending_secret: secret,
        # Mesmo custo de Mfa::Enroll (mínimo em teste): custo fixo 12 já levou
        # a suíte inteira a 10 minutos.
        otp_pending_recovery_codes: codes.map { |c| BCrypt::Password.create(c, cost: Mfa::Enroll.recovery_code_cost).to_s },
        otp_pending_at: Time.current
      )

      {
        otpauth_uri: ROTP::TOTP.new(secret, issuer: "Rota Saúde").provisioning_uri(user.email_address),
        recovery_codes: codes
      }
    end

    # :ok | :no_pending_enrollment | :enrollment_expired | :invalid_code | :code_reused
    def self.confirm(user, code:)
      return :no_pending_enrollment if user.otp_pending_secret.blank?

      if user.otp_pending_at.nil? || user.otp_pending_at <= TTL.ago
        clear!(user)
        return :enrollment_expired
      end

      # SÓ TOTP do pendente: um recovery code (do ativo ou do pendente) não
      # prova que o autenticador novo foi lido.
      step = Mfa::Verify.step_for_secret(user.otp_pending_secret, code)
      return :invalid_code unless step
      return :code_reused unless user.consume_totp_step!(step)

      user.update!(
        otp_secret: user.otp_pending_secret,
        otp_recovery_codes: user.otp_pending_recovery_codes,
        otp_enabled: true,
        otp_pending_secret: nil,
        otp_pending_recovery_codes: [],
        otp_pending_at: nil
      )
      :ok
    end

    def self.clear!(user)
      user.update!(otp_pending_secret: nil, otp_pending_recovery_codes: [], otp_pending_at: nil)
    end
  end
end
```

- [ ] **Step 5: Rodar o serviço e ver passar**

```bash
docker compose exec -T api bundle exec rspec spec/services/mfa/pending_enrollment_spec.rb
```
Esperado: PASS.

- [ ] **Step 6: Escrever a spec de requisição (falha)**

`spec/requests/mfa_pending_enrollment_spec.rb`:

```ruby
require "rails_helper"

# Os endpoints de MFA do usuário da cidade (spec do autenticador pendente §4).
RSpec.describe "MFA pending enrollment", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  def json = JSON.parse(response.body)

  let!(:user) { User.create!(email_address: "cau-#{SecureRandom.hex(3)}@example.org", password: "secret123") }

  def enroll!(stepped_up: false)
    session = sign_in_as(user)
    session.update!(mfa_verified_at: Time.current) if stepped_up
    post "/mfa/enroll", as: :json
    session
  end

  def pending_code = ROTP::TOTP.new(user.reload.otp_pending_secret).now

  it "primeiro cadastro: enroll propõe e confirm promove" do
    enroll!
    expect(response).to have_http_status(:ok)
    expect(user.reload.otp_secret).to be_nil
    expect(user.otp_pending_secret).to be_present

    post "/mfa/confirm", params: { code: pending_code }, as: :json

    expect(response).to have_http_status(:ok)
    expect(user.reload.otp_enabled).to be(true)
    expect(user.otp_secret).to be_present
    expect(user.otp_pending_secret).to be_nil
  end

  context "conta já cadastrada" do
    before do
      Mfa::Enroll.call(user)
      user.update!(otp_enabled: true)
    end

    it "enroll sem step-up continua recusado, e nada muda" do
      secret = user.reload.otp_secret
      enroll!

      expect(response).to have_http_status(:unauthorized)
      expect(json).to eq("error" => "mfa_required")
      expect(user.reload.otp_secret).to eq(secret)
      expect(user.otp_pending_secret).to be_nil
    end

    it "enroll com step-up propõe e o autenticador antigo continua valendo" do
      active_secret = user.reload.otp_secret
      enroll!(stepped_up: true)

      expect(response).to have_http_status(:ok)
      user.reload
      expect(user.otp_secret).to eq(active_secret)
      expect(user.otp_enabled).to be(true)
      expect(user.otp_pending_secret).to be_present
    end

    it "confirm promove e o segredo antigo para de valer" do
      enroll!(stepped_up: true)
      old_secret = user.reload.otp_secret

      post "/mfa/confirm", params: { code: pending_code }, as: :json

      expect(response).to have_http_status(:ok)
      expect(user.reload.otp_secret).not_to eq(old_secret)
    end

    it "confirm sem pendente responde no_pending_enrollment" do
      sign_in_as(user)

      post "/mfa/confirm", params: { code: "123456" }, as: :json

      expect(response).to have_http_status(:unprocessable_entity)
      expect(json).to eq("error" => "no_pending_enrollment")
    end

    it "confirm de um pendente vencido responde enrollment_expired" do
      enroll!(stepped_up: true)
      code = pending_code
      user.update!(otp_pending_at: 16.minutes.ago)

      post "/mfa/confirm", params: { code: code }, as: :json

      expect(response).to have_http_status(:unprocessable_entity)
      expect(json).to eq("error" => "enrollment_expired")
    end

    it "confirm com código errado responde invalid_code" do
      enroll!(stepped_up: true)

      post "/mfa/confirm", params: { code: "000000" }, as: :json

      expect(response).to have_http_status(:unprocessable_entity)
      expect(json).to eq("error" => "invalid_code")
    end
  end

  describe "step_up" do
    before do
      Mfa::Enroll.call(user)
      user.update!(otp_enabled: true)
    end

    it "aceita o TOTP e carimba a sessão" do
      session = sign_in_as(user)

      post "/mfa/step_up", params: { code: ROTP::TOTP.new(user.reload.otp_secret).now }, as: :json

      expect(response).to have_http_status(:ok)
      expect(session.reload.mfa_verified_at).to be_within(5.seconds).of(Time.current)
    end

    it "recusa o MESMO código na segunda vez" do
      sign_in_as(user)
      code = ROTP::TOTP.new(user.reload.otp_secret).now
      post "/mfa/step_up", params: { code: code }, as: :json
      expect(response).to have_http_status(:ok)

      post "/mfa/step_up", params: { code: code }, as: :json

      expect(response).to have_http_status(:unprocessable_entity)
      expect(json).to eq("error" => "code_reused")
    end

    it "continua aceitando um código de recuperação, uma vez só" do
      codes = Mfa::Enroll.call(user)[:recovery_codes]
      user.update!(otp_enabled: true)
      sign_in_as(user)

      post "/mfa/step_up", params: { code: codes.first }, as: :json
      expect(response).to have_http_status(:ok)

      post "/mfa/step_up", params: { code: codes.first }, as: :json
      expect(response).to have_http_status(:unprocessable_entity)
      expect(json).to eq("error" => "invalid_code")
    end
  end
end
```

Ajuste também `spec/requests/mfa_enroll_step_up_spec.rb`, que hoje afirma coisas do modelo antigo: os exemplos "primeiro cadastro não pede step-up" e "com a janela aberta troca o autenticador" continuam, mas o segundo passa a esperar que `otp_secret` **não** mude e que `otp_pending_secret` fique presente. O exemplo "cadastro começado e nunca confirmado recomeça sem step-up" muda de sentido: com o pendente, uma conta ativa com cadastro em andamento **continua ativa**, então `enroll` sem step-up é recusado. Reescreva-o assim, com esse nome novo: "conta ativa com cadastro pendente continua exigindo step-up".

- [ ] **Step 7: Rodar e ver falhar**

```bash
docker compose exec -T api bundle exec rspec spec/requests/mfa_pending_enrollment_spec.rb spec/requests/mfa_enroll_step_up_spec.rb
```
Esperado: FAIL nos exemplos novos (o controller ainda usa `Mfa::Enroll` e não conhece os símbolos).

- [ ] **Step 8: Implementar o controller**

Em `app/controllers/mfa_controller.rb`, mantenha o `rate_limit` e a guarda de step-up como estão, e troque os três corpos:

```ruby
  def enroll
    # Spec do dashboard §4.2: trocar o autenticador de uma conta que JÁ tem
    # TOTP exige step-up — senão a senha sozinha (ou uma sessão roubada)
    # substituiria o segundo fator. O primeiro cadastro não tem fator anterior
    # a pedir e segue só com a sessão.
    return require_step_up! if Current.user.mfa_enrolled? && !reauthenticated_recently?

    payload = Mfa::PendingEnrollment.start(Current.user)
    render json: {
      otpauth_uri: payload[:otpauth_uri],
      recovery_codes: payload[:recovery_codes]   # mostrar uma vez, nunca mais
    }
  end

  # A matrícula só vale depois daqui: é `confirm` que promove o pendente. O
  # autenticador anterior vale até esta linha passar.
  def confirm
    outcome = Mfa::PendingEnrollment.confirm(Current.user, code: params[:code])
    return render(json: { ok: true }) if outcome == :ok

    # :no_pending_enrollment | :enrollment_expired | :invalid_code | :code_reused
    render json: { error: outcome.to_s }, status: :unprocessable_entity
  end

  def step_up
    return render(json: { error: "code_reused" }, status: :unprocessable_entity) if reused_totp?

    if stepped_up?
      Current.session.update!(mfa_verified_at: Time.current)
      render json: { ok: true }
    else
      render json: { error: "invalid_code" }, status: :unprocessable_entity
    end
  end

  private

  # TOTP do segredo ativo, consumido uma vez (User#consume_totp_step!), ou um
  # código de recuperação, consumido como sempre.
  def reused_totp?
    step = Mfa::Verify.totp_step_for(Current.user, params[:code])
    step.present? && !Current.user.consume_totp_step!(step)
  end

  def stepped_up?
    Mfa::Verify.totp_step_for(Current.user, params[:code]).present? ||
      Mfa::Verify.consume_recovery_code(Current.user, params[:code])
  end
```

**Atenção:** `reused_totp?` já consome o passo quando o código é válido, então `stepped_up?` não pode consumir de novo — por isso ele só repete `totp_step_for`, que é leitura pura. Se preferir, escreva os dois num método só; o que não pode é consumir duas vezes nem carimbar a sessão sem consumir.

- [ ] **Step 9: Rodar e ver passar**

```bash
docker compose exec -T api bundle exec rspec spec/requests/mfa_pending_enrollment_spec.rb spec/requests/mfa_enroll_step_up_spec.rb spec/controllers/mfa_controller_spec.rb spec/services/mfa
```
Esperado: PASS. `spec/controllers/mfa_controller_spec.rb` tem o exemplo do rate limit e um de confirmação — ajuste-o ao modelo novo se ele falhar por afirmar o comportamento antigo, sem afrouxar o que ele prova.

- [ ] **Step 10: Suíte completa** (worker parado). Esperado: 0 falhas.

- [ ] **Step 11: Commit**

```bash
/opt/homebrew/bin/git add app/services/mfa/pending_enrollment.rb app/services/mfa/verify.rb app/controllers/mfa_controller.rb spec/services/mfa/pending_enrollment_spec.rb spec/requests/mfa_pending_enrollment_spec.rb spec/requests/mfa_enroll_step_up_spec.rb spec/controllers/mfa_controller_spec.rb
/opt/homebrew/bin/git commit -m "feat: keep the active authenticator until the new one is confirmed" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: dashboard — textos e mensagens

**Files:**
- Modify: `apps/dashboard/src/lib/actionErrors.ts`, `apps/dashboard/src/lib/actionErrors.test.ts`
- Modify: `apps/dashboard/src/modules/Security.tsx`, `apps/dashboard/src/modules/Security.test.tsx`

**Interfaces:**
- Consumes: os códigos de erro da Task 2 (`enrollment_expired`, `code_reused`, `no_pending_enrollment`, `invalid_code`), todos em `422`.
- Produces: `ActionError` ganha o campo opcional `code?: string` nas variantes `invalid_code` e `rejected`, com o código cru da API, para a tela escolher a frase sem reler status HTTP.

**Comportamento:**
- `describeActionError` passa a mapear, em `422`:
  - `invalid_code` → `{ kind: "invalid_code", code: "invalid_code", message: "código inválido" }`;
  - `code_reused` → `{ kind: "invalid_code", code: "code_reused", message: "código já usado — espere o próximo" }` (é erro do campo do código, como o inválido);
  - `enrollment_expired` → `{ kind: "rejected", code: "enrollment_expired", message: "cadastro expirado — comece de novo" }`;
  - qualquer outro `422`/`409` segue como está (mensagem da API, ou a frase genérica).
- `Security.tsx`:
  - some o aviso "Seu autenticador anterior já não vale — confirme o novo para voltar a aprovar ações." e o estado `viaReplace` que só existia para ele (e o parâmetro `replace` de `start`);
  - a descrição da troca volta a ser: "O autenticador atual continua valendo até você confirmar o novo.";
  - sai o `void auth.reload()` de dentro do `run` da troca, e o comentário que o explicava: o enroll não muda mais a sessão;
  - no `catch` da confirmação, a frase do relógio fica **só** para `code === "invalid_code"`; `code_reused` e `enrollment_expired` mostram a mensagem que vem de `describeActionError`.

- [ ] **Step 1: Branch**

```bash
cd apps/dashboard && /opt/homebrew/bin/git checkout -b feat/pending-authenticator
```

- [ ] **Step 2: Testes que falham — mapeamento**

Acrescente a `src/lib/actionErrors.test.ts`:

```ts
  it("422 code_reused é erro do campo do código", () => {
    expect(describeActionError(apiError(422, { error: "code_reused" })))
      .toEqual({ kind: "invalid_code", code: "code_reused", message: "código já usado — espere o próximo" });
  });

  it("422 enrollment_expired pede recomeçar", () => {
    expect(describeActionError(apiError(422, { error: "enrollment_expired" })))
      .toEqual({ kind: "rejected", code: "enrollment_expired", message: "cadastro expirado — comece de novo" });
  });
```

e atualize o exemplo que já existe de `invalid_code` para esperar também `code: "invalid_code"`.

- [ ] **Step 3: Testes que falham — tela**

Em `src/modules/Security.test.tsx`:
- no teste da troca, troque a asserção do aviso removido por `expect(screen.queryByText(/anterior já não vale/)).toBeNull()` e acrescente `expect(await screen.findByText("O autenticador atual continua valendo até você confirmar o novo.")).not.toBeNull();` antes de confirmar o step-up;
- acrescente dois exemplos, no molde do que já existe para o código inválido:

```tsx
  it("código já usado: mensagem própria, campo limpo", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session());
    mocked(api.enrollMfa).mockResolvedValue(ENROLLMENT);
    mocked(api.confirmMfa).mockRejectedValue(new ApiError(422, { error: "code_reused" }, "422"));
    renderSecurity();

    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar autenticador" }));
    await screen.findByAltText("QR do autenticador");
    fireEvent.click(screen.getByLabelText("guardei os códigos"));
    fireEvent.change(screen.getByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar cadastro" }));

    expect(await screen.findByText("código já usado — espere o próximo")).not.toBeNull();
    expect((screen.getByLabelText("Código do autenticador") as HTMLInputElement).value).toBe("");
  });

  it("cadastro expirado: pede recomeçar", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session());
    mocked(api.enrollMfa).mockResolvedValue(ENROLLMENT);
    mocked(api.confirmMfa).mockRejectedValue(new ApiError(422, { error: "enrollment_expired" }, "422"));
    renderSecurity();

    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar autenticador" }));
    await screen.findByAltText("QR do autenticador");
    fireEvent.click(screen.getByLabelText("guardei os códigos"));
    fireEvent.change(screen.getByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar cadastro" }));

    expect(await screen.findByText("cadastro expirado — comece de novo")).not.toBeNull();
  });
```

- [ ] **Step 4: Rodar e ver falhar**

Run: `npx vitest run src/lib/actionErrors.test.ts src/modules/Security.test.tsx`
Esperado: FAIL nos exemplos novos.

- [ ] **Step 5: Implementar**

Em `src/lib/actionErrors.ts`:

```ts
export type ActionError =
  | { kind: "mfa_required" }
  | { kind: "invalid_code"; code: string; message: string }
  | { kind: "forbidden"; message: string }
  | { kind: "rejected"; code?: string; message: string }
  | { kind: "session_expired"; message: string }
  | { kind: "rate_limited"; message: string }
  | { kind: "failed"; message: string };
```

e, dentro de `describeActionError`, antes do tratamento genérico de 422/409:

```ts
  // Erros do campo do código (a matrícula em duas etapas da API: spec do
  // autenticador pendente §4). `code` viaja junto para a tela escolher a
  // frase sem reler status HTTP.
  if (err.status === 422 && code === "invalid_code") {
    return { kind: "invalid_code", code, message: "código inválido" };
  }
  if (err.status === 422 && code === "code_reused") {
    return { kind: "invalid_code", code, message: "código já usado — espere o próximo" };
  }
  if (err.status === 422 && code === "enrollment_expired") {
    return { kind: "rejected", code, message: "cadastro expirado — comece de novo" };
  }
```

(a linha antiga de `invalid_code` some; o resto da função fica igual.)

Em `src/modules/Security.tsx`:
- apague o estado `viaReplace`, o parâmetro `replace` de `start` e o bloco `{viaReplace && (...)}`;
- na `SensitiveAction` da troca: `description="O autenticador atual continua valendo até você confirmar o novo."` e `run={async () => { start(await enrollMfa()); }}` (sem o `auth.reload()` e sem o comentário dele);
- no `catch` de `confirm`:

```tsx
      const described = describeActionError(err);
      setError(described.kind === "invalid_code" && described.code === "invalid_code"
        ? "código inválido — confira se o relógio do celular está certo"
        : "message" in described ? described.message : "não foi possível concluir — tente de novo");
```

- [ ] **Step 6: Rodar e ver passar**

```bash
npx vitest run src/lib/actionErrors.test.ts src/modules/Security.test.tsx && npm run typecheck && npm test && npm run build
```
Esperado: verde.

- [ ] **Step 7: Verificação no navegador** (stack de dev de pé)

1. Na raiz do monorepo, aplique a migração nas cidades de dev: `docker compose exec -T api bin/rails city:migrate:all`.
2. Abra `http://curitiba.localhost:5175/dashboard/` com uma conta de dev da cidade (as contas estão nos READMEs; não invente). A conta `admin@curitiba.demo` pode estar sem autenticador por causa de uma troca inacabada anterior — se estiver, cadastre.
3. Com o autenticador ativo, clique em "Trocar autenticador", faça o step-up e **saia da tela sem confirmar**. Volte: a página deve continuar mostrando "Autenticador ativo.", e um step-up com o autenticador **antigo** deve funcionar (ex.: abra a troca de novo e informe o código antigo).
4. Agora complete uma troca: step-up, leia o QR novo, confirme. O novo passa a valer.

Registre no relatório o que foi visto, sem chave e sem códigos.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add src/lib/actionErrors.ts src/lib/actionErrors.test.ts src/modules/Security.tsx src/modules/Security.test.tsx
/opt/homebrew/bin/git commit -m "fix: tell the truth about replacing an authenticator" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Fora deste plano

- Cadastro obrigatório de TOTP ao aceitar convite e aviso por e-mail ao cadastrar ou trocar (spec §4 "o limite que continua").
- Mantenedor e operador, que seguem com `Mfa::Enroll`.
- Login da cidade com TOTP.
- A fatia 2 das telas de assinatura (Equipe), que este plano destrava.
