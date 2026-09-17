# Password Reset by Email (F-06.2) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add secure, email-based password reset over the JSON API: a signed single-use token, a mailer with a frontend link, and two rate-limited endpoints.

**Architecture:** `User.generates_token_for :password_reset` (no DB table, 15-min, single-use via `password_salt`); `PasswordMailer#reset` emails a `PUBLIC_DASHBOARD_URL?reset=<token>` link; `PasswordsController` exposes `POST /passwords` (request; always 204, no enumeration) and `PUT /passwords/:token` (reset; destroys sessions on success).

**Tech Stack:** Rails 8.1 (RSpec request + mailer specs). Run specs in the `api-dev` container.

## Global Constraints

- **Commits in English** (UI/email copy stays Portuguese). `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER:** `docker exec api-dev bundle exec rspec <path>`.
- No user enumeration: `POST /passwords` always `204`; mail only for an existing **active** user (`deactivated_at` nil).
- Token: `generates_token_for :password_reset, expires_in: 15.minutes` with a `password_salt&.last(10)` block → single-use (dies when the password changes). No migration.
- On successful reset: `user.sessions.destroy_all`. Both endpoints `rate_limit to: 10, within: 3.minutes` (mirror `SessionsController`).
- Email link: `#{ENV["PUBLIC_DASHBOARD_URL"] || "http://localhost:5174/dashboard/"}?reset=#{token}`. From = `ENV["MAIL_FROM"]` (via `ApplicationMailer`).

---

### Task 1: `User` reset token + `PasswordMailer`

**Files:**
- Modify: `apps/api/app/models/user.rb`
- Create: `apps/api/app/mailers/password_mailer.rb`, `apps/api/app/views/password_mailer/reset.html.erb`, `apps/api/app/views/password_mailer/reset.text.erb`
- Test: `apps/api/spec/models/user_reset_token_spec.rb` (create), `apps/api/spec/mailers/password_mailer_spec.rb` (create)

**Interfaces:**
- Produces: `user.generate_token_for(:password_reset)` / `User.find_by_token_for(:password_reset, token)` (single-use, 15-min); `PasswordMailer.reset(user)` → an email to `user.email_address` whose body contains `?reset=<token>`.

- [ ] **Step 1: Write the failing tests**

Create `apps/api/spec/models/user_reset_token_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "User password_reset token (F-06.2)", type: :model do
  let(:user) { User.create!(email_address: "reset-#{SecureRandom.hex(3)}@x.com", password: "old-password-1") }

  it "round-trips a password_reset token" do
    token = user.generate_token_for(:password_reset)
    expect(User.find_by_token_for(:password_reset, token)).to eq(user)
  end

  it "invalidates the token once the password changes (single-use)" do
    token = user.generate_token_for(:password_reset)
    user.update!(password: "new-password-2")
    expect(User.find_by_token_for(:password_reset, token)).to be_nil
  end

  it "returns nil for a garbage token" do
    expect(User.find_by_token_for(:password_reset, "not-a-real-token")).to be_nil
  end
end
```

Create `apps/api/spec/mailers/password_mailer_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe PasswordMailer, type: :mailer do
  let(:user) { User.create!(email_address: "mailer-#{SecureRandom.hex(3)}@x.com", password: "secret123") }

  it "addresses the user and includes a ?reset= link, not the raw password" do
    mail = described_class.reset(user)
    expect(mail.to).to eq([user.email_address])
    expect(mail.subject).to be_present
    body = mail.body.encoded
    expect(body).to include("?reset=")
    expect(body).not_to include("secret123")
  end
end
```

- [ ] **Step 2: Run to verify they fail**

Run: `docker exec api-dev bundle exec rspec spec/models/user_reset_token_spec.rb spec/mailers/password_mailer_spec.rb`
Expected: FAIL — no `password_reset` token purpose defined; `uninitialized constant PasswordMailer`.

- [ ] **Step 3: Add the token to `User`**

In `apps/api/app/models/user.rb`, inside the class (e.g. after `has_secure_password`), add:

```ruby
  generates_token_for :password_reset, expires_in: 15.minutes do
    password_salt&.last(10)
  end
```

- [ ] **Step 4: Create the mailer + views**

`apps/api/app/mailers/password_mailer.rb`:

```ruby
# E-mail de redefinição de senha (F-06.2, ADR-0022). Link aponta para o
# frontend (PUBLIC_DASHBOARD_URL); o token expira em 15 min e é de uso único.
class PasswordMailer < ApplicationMailer
  def reset(user)
    @user = user
    token = user.generate_token_for(:password_reset)
    base = ENV["PUBLIC_DASHBOARD_URL"] || "http://localhost:5174/dashboard/"
    @reset_url = "#{base}?reset=#{token}"
    mail(to: user.email_address, subject: "[rota-saúde] Redefinição de senha")
  end
end
```

`apps/api/app/views/password_mailer/reset.html.erb`:

```erb
<p>Olá,</p>
<p>Recebemos um pedido para redefinir sua senha do Rota Saúde.</p>
<p><a href="<%= @reset_url %>">Clique aqui para redefinir sua senha</a>.</p>
<p>O link expira em 15 minutos. Se você não fez este pedido, ignore este e-mail.</p>
```

`apps/api/app/views/password_mailer/reset.text.erb`:

```erb
Olá,

Recebemos um pedido para redefinir sua senha do Rota Saúde.

Redefina sua senha: <%= @reset_url %>

O link expira em 15 minutos. Se você não fez este pedido, ignore este e-mail.
```

- [ ] **Step 5: Run to verify they pass**

Run: `docker exec api-dev bundle exec rspec spec/models/user_reset_token_spec.rb spec/mailers/password_mailer_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git -C apps/api add app/models/user.rb app/mailers/password_mailer.rb app/views/password_mailer spec/models/user_reset_token_spec.rb spec/mailers/password_mailer_spec.rb
git -C apps/api commit -m "Add password_reset token and PasswordMailer"
git -C apps/api log --oneline -1
```

---

### Task 2: `PasswordsController` + routes

**Files:**
- Create: `apps/api/app/controllers/passwords_controller.rb`
- Modify: `apps/api/config/routes.rb`
- Test: `apps/api/spec/requests/passwords_spec.rb` (create)

**Interfaces:**
- Consumes: `PasswordMailer.reset` + the `:password_reset` token (Task 1).
- Produces: `POST /passwords` (always 204, mail only for active user) and `PUT /passwords/:token` (reset + session destroy, or 422).

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/requests/passwords_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Passwords (F-06.2)", type: :request do
  def make_user(active: true, password: "old-password-1")
    u = User.create!(email_address: "u-#{SecureRandom.hex(3)}@x.com", password: password)
    u.update!(deactivated_at: Time.current) unless active
    u
  end

  describe "POST /passwords (request reset)" do
    it "enqueues the reset mail for an active user and returns 204" do
      user = make_user
      expect {
        post "/passwords", params: { email_address: user.email_address }
      }.to have_enqueued_mail(PasswordMailer, :reset)
      expect(response).to have_http_status(:no_content)
    end

    it "does not enumerate: unknown email returns 204 with no mail" do
      expect {
        post "/passwords", params: { email_address: "nobody@nowhere.com" }
      }.not_to have_enqueued_mail(PasswordMailer, :reset)
      expect(response).to have_http_status(:no_content)
    end

    it "does not email a deactivated user (still 204)" do
      user = make_user(active: false)
      expect {
        post "/passwords", params: { email_address: user.email_address }
      }.not_to have_enqueued_mail(PasswordMailer, :reset)
      expect(response).to have_http_status(:no_content)
    end
  end

  describe "PUT /passwords/:token (perform reset)" do
    it "resets the password, destroys sessions, and returns 204" do
      user = make_user
      session = user.sessions.create!(user_agent: "rspec", ip_address: "127.0.0.1")
      token = user.generate_token_for(:password_reset)

      put "/passwords/#{token}", params: { password: "new-password-2", password_confirmation: "new-password-2" }
      expect(response).to have_http_status(:no_content)

      expect(Authenticator.password(email: user.email_address, password: "new-password-2")).to eq(user)
      expect(Authenticator.password(email: user.email_address, password: "old-password-1")).to be_nil
      expect(Session.exists?(session.id)).to be(false)
    end

    it "returns 422 for an invalid token" do
      put "/passwords/garbage-token", params: { password: "whatever-123", password_confirmation: "whatever-123" }
      expect(response).to have_http_status(:unprocessable_entity)
      expect(JSON.parse(response.body)["error"]).to eq("invalid_token")
    end

    it "returns 422 for a mismatched/blank password" do
      user = make_user
      token = user.generate_token_for(:password_reset)
      put "/passwords/#{token}", params: { password: "a", password_confirmation: "b" }
      expect(response).to have_http_status(:unprocessable_entity)
      expect(JSON.parse(response.body)).to have_key("errors")
    end

    it "is single-use: replaying a consumed token returns 422" do
      user = make_user
      token = user.generate_token_for(:password_reset)
      put "/passwords/#{token}", params: { password: "new-password-2", password_confirmation: "new-password-2" }
      expect(response).to have_http_status(:no_content)

      put "/passwords/#{token}", params: { password: "again-password-3", password_confirmation: "again-password-3" }
      expect(response).to have_http_status(:unprocessable_entity)
    end
  end
end
```

Before running, confirm the `Session` columns used in `user.sessions.create!` match the real `sessions` schema (check `db/schema.rb` / the `Session` model) — adjust the create attributes (or use the app's session-creation helper) if they differ; the asserted behavior (sessions destroyed on reset) is the requirement. Also confirm `have_enqueued_mail` is available (rspec-rails + `:test` queue adapter — it is, per `config/environments/test.rb`).

- [ ] **Step 2: Run to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/requests/passwords_spec.rb`
Expected: FAIL — routing error / `uninitialized constant PasswordsController`.

- [ ] **Step 3: Create the controller**

`apps/api/app/controllers/passwords_controller.rb`:

```ruby
# Reset de senha por e-mail (F-06.2, ADR-0022). JSON-only, sem autenticação.
# create: sempre 204 (sem enumeração de usuários). update: consome o token de
# uso único e destrói as sessões do usuário.
class PasswordsController < ApplicationController
  skip_tenant_scope
  include Authentication

  allow_unauthenticated_access only: %i[create update]

  rate_limit to: 10, within: 3.minutes, only: %i[create update],
             with: -> { render json: { error: "too_many_requests" }, status: :too_many_requests }

  def create
    user = User.where("lower(email_address) = ?", params[:email_address].to_s.downcase).first
    PasswordMailer.reset(user).deliver_later if user&.active?
    head :no_content
  end

  def update
    user = User.find_by_token_for(:password_reset, params[:token])
    return render(json: { error: "invalid_token" }, status: :unprocessable_entity) unless user

    if user.update(password: params[:password], password_confirmation: params[:password_confirmation])
      user.sessions.destroy_all
      head :no_content
    else
      render json: { errors: user.errors.full_messages }, status: :unprocessable_entity
    end
  end
end
```

NOTE: if `ApplicationController`/`Authentication`/`skip_tenant_scope` require anything extra for an unauthenticated JSON controller (compare with `SessionsController`, which uses the same `skip_tenant_scope` + `include Authentication` + `allow_unauthenticated_access`), mirror it. Do not add tenant scoping — these endpoints are pre-auth.

- [ ] **Step 4: Add the routes**

In `apps/api/config/routes.rb`, replace the comment line `# Sessão de admin (ADR-0022). Reset de senha fica para ADR de mailer.` context by adding, near the `resource :session` line:

```ruby
  resources :passwords, only: %i[create update], param: :token
```
(→ `POST /passwords`, `PUT/PATCH /passwords/:token`.) Keep the rest of the routes unchanged.

- [ ] **Step 5: Run to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/requests/passwords_spec.rb`
Expected: PASS (all examples).

- [ ] **Step 6: Regression — auth request specs**

Run: `docker exec api-dev bundle exec rspec spec/requests`
Expected: green (sessions/mfa/etc. unaffected).

- [ ] **Step 7: Commit**

```bash
git -C apps/api add app/controllers/passwords_controller.rb config/routes.rb spec/requests/passwords_spec.rb
git -C apps/api commit -m "Add password reset endpoints (request + token-based reset)"
git -C apps/api log --oneline -1
```

---

## Wrap-up (after all tasks)

- Run `docker exec api-dev bundle exec rspec spec/requests/passwords_spec.rb spec/mailers/password_mailer_spec.rb spec/models/user_reset_token_spec.rb` — expect green.
- Move the board card (F-06.2) to Done, then Verified.
- Sync is a separate explicit step (user-authorized). Frontend reset page is a follow-up.

## Self-Review notes

- **Spec coverage:** token (round-trip, single-use) → Task 1; mailer (address, link, no-password) → Task 1; endpoints (204-no-enumeration, active-only mail, reset+session-destroy, invalid-token 422, bad-password 422, single-use) → Task 2. All spec sections mapped.
- **Type consistency:** `generate_token_for(:password_reset)` / `find_by_token_for(:password_reset, …)` used identically in User (Task 1), the mailer (Task 1), and the controller/tests (Task 2).
- **Security:** always-204 in `create`; `deliver_later` only for `user&.active?`; `find_by_token_for` returns nil for invalid/expired/consumed tokens → 422; `sessions.destroy_all` on success; both endpoints rate-limited.
- **No migration:** signed token, no table.
