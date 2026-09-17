# CHORE — Rate-limit SetupController#accept_invitation

**Date:** 2026-07-02
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-06 (auth hardening)
**Board:** CHORE card (In Progress). Flagged by the F-06.2 whole-branch security review.
**Touches:** `apps/api` only.

## Problem

`POST /setup/accept_invitation` is a **public, token-as-credential** endpoint
(`allow_unauthenticated_access only: %i[accept_invitation]`; it calls
`AcceptInvitation.call(token:, password:)`, looking the user up by an invitation
token). Unlike `SessionsController` and `PasswordsController` — the other public
credential endpoints, both `rate_limit to: 10, within: 3.minutes` — it has **no
rate limit**, leaving a brute-force / token-guessing surface. F-06.2's security
review flagged this once the house rate-limit pattern was established twice.

## Current state (verified)

- `SetupController` (`include Authentication`, `skip_tenant_scope`): actions
  `provision_municipality`/`invite_member`/`revoke_membership`/`deactivate_user`
  (authenticated operator/municipal_admin) + `accept_invitation` (public). No
  `rate_limit`.
- `SessionsController` / `PasswordsController`:
  `rate_limit to: 10, within: 3.minutes, only: %i[…], with: -> { render json: {
  error: "too_many_requests" }, status: :too_many_requests }`.
- Test env: `config.cache_store = :null_store`. `rate_limit` counts via the cache,
  so it is a **no-op under `:null_store`** — a 429 cannot be observed in test.
  Consistent with the repo: `rate_limit` is not tested anywhere
  (Sessions/Passwords rate limits are untested for the same reason).
- `AcceptInvitation` (control-plane): finds the Invitation by token; on success
  creates the user + identity + membership and starts a session. `Invitation`
  (control-plane) needs `email`, `role`, `token`, `invited_by`, `expires_at`,
  `municipality_id`.

## Decisions (from brainstorming)

1. Add the **house `rate_limit`** to `SetupController`, scoped `only:
   %i[accept_invitation]` — the only public brute-force surface; the
   authenticated operator/admin actions don't need it.
2. **Do not** test the 429 directly (untestable under `:null_store`; a global
   cache-store change is out of scope). Document the limitation, consistent with
   Sessions/Passwords.
3. Add a **smoke request spec** (the first `setup` request spec): a valid
   invitation accept → `201`, an invalid token → `422` — proving the
   `only: %i[accept_invitation]` macro is valid and doesn't break the endpoint.
4. No migration.

## Design

`app/controllers/setup_controller.rb`: after `allow_unauthenticated_access only:
%i[accept_invitation]`, add:

```ruby
  # Fluxo público token-as-credential — mesmo teto de sessions/passwords, para
  # não deixar superfície de brute-force sem limite. Só na ação pública.
  rate_limit to: 10, within: 3.minutes, only: %i[accept_invitation],
             with: -> { render json: { error: "too_many_requests" }, status: :too_many_requests }
```

No other change to the controller or its actions.

## Testing (RSpec request spec, run in the container)

`spec/requests/setup_accept_invitation_spec.rb` (`type: :request`) — mirror the
`spec/commands/accept_invitation_spec.rb` fixture setup (`create(:municipality)`;
operator user + `Invitation` via `connected_to(role: :admin)`):

- **valid token + password → `201`**, body `email_address` == the invitation's
  email (the rate_limit macro doesn't break the happy path).
- **invalid token → `422`** (the endpoint is still reachable, not rate-limited
  away in the first request).

(If a transactional request spec trips on the command's admin-connection +
`DomainEvents.publish` under `SET LOCAL`, fall back to the
`use_transactional_tests = false` + `as_admin` + `clean_admin_tables` harness, as
the command/job specs do.) A 429 test is intentionally omitted — untestable under
`:null_store`, documented above.

## Out of scope / follow-ups

- Testing the actual 429 (needs a counting test cache — a global change).
- Per-account throttling / captcha beyond the IP rate limit.
- Rate-limiting the authenticated setup actions (not a brute-force surface).

## Workflow

CHORE card In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`.
