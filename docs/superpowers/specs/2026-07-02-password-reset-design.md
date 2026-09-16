# F-06.2 — Password reset by email

**Date:** 2026-07-02
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-06
**Board:** F-06.2 (In Progress)
**Touches:** `apps/api` only. (The frontend reset page is a dashboard-app follow-up.)

## Problem

Staff auth (ADR-0022) has login/MFA/sessions but no way to recover a forgotten
password — `routes.rb` even notes "Reset de senha fica para ADR de mailer." A
user who forgets their password is locked out. F-06.2 adds a secure,
email-based password reset over the existing JSON API.

## Current state (verified)

- `User`: `has_secure_password` (`password_digest`, and `password_salt` via Rails
  8.1); `email_address` normalized (`strip.downcase`) + unique; `active?` =
  `deactivated_at.nil?`. No reset token yet. `generates_token_for` is unused.
- `SessionsController` (JSON-only): `allow_unauthenticated_access only:`,
  `rate_limit to: 10, within: 3.minutes`, cookie session. `Authenticator.password(
  email:, password:)` returns the user iff active + password matches.
- Mailers: `ApplicationMailer` (`default from: ENV["MAIL_FROM"]`, `layout
  "mailer"`); `AlertMailer` sends real mail. Layouts `app/views/layouts/mailer.{html,text}.erb` exist.
- Frontend link config: `PUBLIC_DASHBOARD_URL` (used by the invitation flow —
  `setup_controller#setup_accept_url` builds `"#{base}?invite=#{token}"`).
- Test env: `action_mailer.delivery_method = :test`, `active_job.queue_adapter = :test`
  (so `deliver_later` is assertable via `have_enqueued_mail`).

## Decisions (from brainstorming)

1. Rails 8 idiom: `generates_token_for :password_reset` (signed, no DB table),
   15-minute expiry, **single-use** (block returns a slice of `password_salt`, so
   the token dies once the password changes).
2. **No user enumeration**: `POST /passwords` always responds `204`, whether or
   not the email exists. Mail is sent only for an existing **active** user.
3. On a successful reset, **destroy the user's sessions** (force re-login).
4. Both endpoints **rate-limited** (10 / 3 min), like sessions.
5. Email links to the **frontend** dashboard reset page via `PUBLIC_DASHBOARD_URL`
   (`?reset=<token>`); the page itself is out of scope (dashboard app).
6. No migration. MFA is unchanged (the emailed token proves email possession;
   operators still do TOTP at next login).

## Design

### 1. `User` — reset token

```ruby
generates_token_for :password_reset, expires_in: 15.minutes do
  password_salt&.last(10)
end
```

### 2. `PasswordsController` (JSON API)

`app/controllers/passwords_controller.rb`:
- `include Authentication`; `allow_unauthenticated_access only: %i[create update]`;
  `skip_tenant_scope`; `rate_limit to: 10, within: 3.minutes, only: %i[create update],
  with: -> { render json: { error: "too_many_requests" }, status: :too_many_requests }`.
- **`create` — POST /passwords** `{ email_address }`:
  - `user = User.where("lower(email_address) = ?", email.to_s.downcase).first`
  - `PasswordMailer.reset(user).deliver_later if user&.active?`
  - always `head :no_content` (204). No body, no enumeration.
- **`update` — PUT /passwords/:token** `{ password, password_confirmation }`:
  - `user = User.find_by_token_for(:password_reset, params[:token])`
  - `return render(json: { error: "invalid_token" }, status: :unprocessable_entity) unless user`
  - `if user.update(password: params[:password], password_confirmation: params[:password_confirmation])`
    → `user.sessions.destroy_all; head :no_content` (204)
  - `else` → `render json: { errors: user.errors.full_messages }, status: :unprocessable_entity`

### 3. `PasswordMailer`

`app/mailers/password_mailer.rb`:
```ruby
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
Views `app/views/password_mailer/reset.html.erb` + `reset.text.erb`: a short pt-BR
message with `@reset_url` and a note that the link expires in 15 minutes and can
be ignored if not requested.

### 4. Routes

`config/routes.rb`: `resources :passwords, only: %i[create update], param: :token`
(→ `POST /passwords`, `PUT/PATCH /passwords/:token`). Replace the "reset de senha
fica para ADR de mailer" comment.

## Testing (RSpec request specs, run in the container)

`spec/requests/passwords_spec.rb` (`type: :request`):

- **POST with an active user's email** → `204`; enqueues `PasswordMailer.reset`
  (`have_enqueued_mail(PasswordMailer, :reset)`).
- **POST with an unknown email** → `204`; **no** mail enqueued (no enumeration —
  identical response).
- **POST with a deactivated user's email** → `204`; no mail.
- **PUT with a valid token + valid password** → `204`; the user can now
  `Authenticator.password` with the new password and not the old; the user's
  sessions were destroyed.
- **PUT with an invalid/garbage token** → `422 invalid_token`.
- **PUT with a valid token but mismatched/blank password** → `422` with `errors`.
- **Single-use**: after a successful reset, replaying the same token → `422`
  (the salt changed, token invalid).

`spec/mailers/password_mailer_spec.rb` (optional but recommended): `reset(user)`
addresses `user.email_address`, subject present, body contains a `?reset=` link
and does not contain the raw password.

## Out of scope / follow-ups

- The frontend reset page (dashboard app) that reads `?reset=<token>` and calls
  `PUT /passwords/:token`.
- Notifying the user that their password changed (a confirmation email).
- Throttling per-account (beyond the IP rate-limit) / captcha.

## Workflow

Card F-06.2 In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`.
