# Rate-limit accept_invitation (CHORE) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the house rate limit to the public `POST /setup/accept_invitation` endpoint, with a smoke request spec proving the endpoint still works.

**Architecture:** A single `rate_limit to: 10, within: 3.minutes, only: %i[accept_invitation]` on `SetupController`, mirroring `SessionsController`/`PasswordsController`.

**Tech Stack:** Rails 8.1 (RSpec request spec). Run specs in the `api-dev` container.

## Global Constraints

- **Commits in English.** `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER:** `docker exec api-dev bundle exec rspec <path>`.
- Scope the rate_limit to `only: %i[accept_invitation]` (the only public brute-force surface). No other controller change. No migration.
- A 429 is NOT testable under the `:null_store` test cache (rate_limit is a no-op) — do NOT add a 429 test; the smoke spec proves the macro doesn't break the endpoint.

---

### Task 1: rate_limit + smoke request spec

**Files:**
- Modify: `apps/api/app/controllers/setup_controller.rb`
- Test: `apps/api/spec/requests/setup_accept_invitation_spec.rb` (create)

**Interfaces:**
- Produces: `POST /setup/accept_invitation` is rate-limited (`10 / 3.minutes`) and still returns `201` for a valid invitation / `422` for an invalid token.

- [ ] **Step 1: Write the smoke test**

Create `apps/api/spec/requests/setup_accept_invitation_spec.rb`, mirroring `spec/commands/accept_invitation_spec.rb`'s fixture setup:

```ruby
require "rails_helper"

RSpec.describe "Setup accept_invitation", type: :request do
  let(:muni) { create(:municipality) }
  let!(:operator) do
    u = User.create!(email_address: "op-#{SecureRandom.hex(3)}@example.org", password: "secret123")
    Membership.create!(user: u, role: "platform_operator", granted_at: Time.current)
    u
  end
  let!(:inv) do
    ApplicationRecord.connected_to(role: :admin) do
      Invitation.create!(
        email: "new-#{SecureRandom.hex(3)}@example.org", role: "municipal_admin", municipality: muni,
        token: "tok-#{SecureRandom.hex(4)}", invited_by: operator, expires_at: 1.day.from_now
      )
    end
  end

  it "accepts a valid invitation (the rate_limit macro does not break the public endpoint)" do
    post "/setup/accept_invitation", params: { token: inv.token, password: "secretpw-1" }
    expect(response).to have_http_status(:created)
    expect(JSON.parse(response.body)["email_address"]).to eq(inv.email)
  end

  it "rejects an invalid token with 422 (endpoint reachable, not rate-limited away)" do
    post "/setup/accept_invitation", params: { token: "nope", password: "x" }
    expect(response).to have_http_status(:unprocessable_entity)
  end
end
```

If a transactional request spec trips on `AcceptInvitation`'s admin-connection + `DomainEvents.publish` under `SET LOCAL` (e.g. an RLS/connection error), switch this spec to the non-transactional admin harness used by the job/query request specs: `require Rails.root.join("spec/support/admin_rls")`, `self.use_transactional_tests = false`, `before { clean_admin_tables }` / `after { clean_admin_tables }`, and build the fixtures inside `as_admin`. Keep the two assertions unchanged.

- [ ] **Step 2: Run to verify it fails or passes**

Run: `docker exec api-dev bundle exec rspec spec/requests/setup_accept_invitation_spec.rb`
Expected: the two examples PASS **without** the rate_limit line too (the endpoint already works) — this spec is a regression guard that the rate_limit macro (added next) does not break the endpoint. If they don't pass yet, fix the fixture harness (see Step 1 note) before proceeding.

- [ ] **Step 3: Add the rate_limit**

In `apps/api/app/controllers/setup_controller.rb`, immediately after the line
`allow_unauthenticated_access only: %i[accept_invitation]`, add:

```ruby
  # Fluxo público token-as-credential — mesmo teto de sessions/passwords, para
  # não deixar superfície de brute-force sem limite. Só na ação pública.
  rate_limit to: 10, within: 3.minutes, only: %i[accept_invitation],
             with: -> { render json: { error: "too_many_requests" }, status: :too_many_requests }
```

Change nothing else.

- [ ] **Step 4: Run to verify it still passes**

Run: `docker exec api-dev bundle exec rspec spec/requests/setup_accept_invitation_spec.rb`
Expected: PASS (2 examples) — the endpoint still returns 201/422 with the rate_limit in place (429 is a no-op under `:null_store`).

- [ ] **Step 5: Regression — the auth/setup command specs**

Run: `docker exec api-dev bundle exec rspec spec/requests spec/commands/accept_invitation_spec.rb`
Expected: green (nothing else affected).

- [ ] **Step 6: Commit**

```bash
git -C apps/api add app/controllers/setup_controller.rb spec/requests/setup_accept_invitation_spec.rb
git -C apps/api commit -m "Rate-limit the public accept_invitation endpoint"
git -C apps/api log --oneline -1
```

---

## Wrap-up (after the task)

- Run `docker exec api-dev bundle exec rspec spec/requests/setup_accept_invitation_spec.rb` — expect green.
- Move the CHORE card to Done, then Verified.
- Sync is a separate explicit step (user-authorized).

## Self-Review notes

- **Spec coverage:** the rate_limit line (scoped to accept_invitation) → Step 3; the smoke spec (valid→201, invalid→422) → Step 1. The 429 is intentionally untested (null_store) — documented in the spec.
- **Placeholder scan:** none.
- **Consistency:** the `rate_limit` signature is byte-identical to Sessions/Passwords except `only: %i[accept_invitation]`.
