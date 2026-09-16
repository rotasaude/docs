# F-01.9 — WhatsApp channel access_token rotation (custody, zero-downtime)

**Date:** 2026-07-02
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-01
**Board:** F-01.9 (In Progress)
**Touches:** `apps/api` only.

## Problem

The WhatsApp Cloud API bearer token lives on `municipality_channels.access_token`
(AR-encrypted). It is written only at provisioning (`ProvisionMunicipality`) and
the dev setup path — there is no controlled way to rotate it when a city's token
must be replaced. F-01.9 adds an audited, authorized rotation with clear custody,
without downtime.

## Current state (verified)

- `MunicipalityChannel` (RLS-exempt control-plane table): `encrypts :access_token`;
  `scope :active`. Read at send time by `Whatsapp::Outbound` —
  `"Authorization" => "Bearer #{@channel.access_token}"` — fetched fresh per send
  (`SendWhatsappJob` looks up `MunicipalityChannel.find_by!(active: true)` each time).
  So an in-place token update is inherently zero-downtime.
- `Platform.audit(name, **payload)` (`app/events/platform.rb`): writes a
  control-plane `DomainEvent` (`municipality_id: nil`) under the admin connection
  — the audit channel for identity/provisioning events. No tenant needed.
- Command pattern (`Result`): `Result.ok(payload_hash)` / `Result.fail(reason,
  message:)`; `result.ok?`/`failure?`/`reason`/`payload`. Authorization is done
  inside the command (e.g. `Protocols::Publish` → `ProtocolPolicy`, `Result.fail(:forbidden)`).
- `Membership` (control-plane): `user.memberships.active`; `role` ∈
  {platform_operator, municipal_admin, …}; `municipality_id` (nil for operator).
  Per ADR-0012, `municipal_admin` "configura canal" for its city.
- `Admin::Api::*` is **read-only** (acceptance criterion §10) — a write endpoint
  does NOT belong there.

## Decisions (from brainstorming)

1. **Manual, audited rotation with custody** (chosen). Not auto-refresh via Meta,
   not pre-swap Graph verification (both deferred).
2. Zero-downtime by construction — an in-place `access_token` update takes effect
   on the next send (token read fresh each time).
3. **Custody = role**: `platform_operator` (any city) or `municipal_admin` of the
   channel's municipality. Enforced inside the command, surface-independent.
4. **Audit** via `Platform.audit("channel.token_rotated", …)` — municipality_id,
   phone_number_id, actor id; **never the token value**.
5. Trigger surface for MVP = the command + a rake task. An authenticated HTTP
   endpoint (with step-up MFA) is a follow-up (Admin::Api is read-only).
6. No migration.

## Design

### 1. `MunicipalityChannels::RotateToken` command

`app/commands/municipality_channels/rotate_token.rb`:

```ruby
# Rotaciona o access_token do canal WhatsApp de um município (F-01.9).
# Custody: platform_operator (qualquer cidade) ou municipal_admin da cidade.
# Zero-downtime: Outbound lê o token fresco a cada envio. Auditoria via
# Platform.audit — SEM o valor do token (ADR-0011/0023).
module MunicipalityChannels
  module RotateToken
    def self.call(municipality_id:, new_token:, by:)
      return Result.fail(:invalid, message: "new_token vazio") if new_token.blank?
      return Result.fail(:forbidden) unless authorized?(by, municipality_id)

      ApplicationRecord.connected_to(role: :admin) do
        channel = MunicipalityChannel.active.find_by(municipality_id: municipality_id)
        return Result.fail(:not_found) unless channel

        channel.update!(access_token: new_token)
        Platform.audit(
          "channel.token_rotated",
          municipality_id: municipality_id,
          phone_number_id: channel.phone_number_id,
          by: by&.id
        )
        Result.ok(channel: channel)
      end
    end

    def self.authorized?(user, municipality_id)
      return false unless user
      user.memberships.active.any? do |m|
        m.role == "platform_operator" ||
          (m.role == "municipal_admin" && m.municipality_id == municipality_id)
      end
    end
    private_class_method :authorized?
  end
end
```

Reasons: `:invalid` (blank token), `:forbidden` (custody), `:not_found` (no active
channel). `authorized?` reads `memberships` (control-plane, readable under the
admin connection).

### 2. `channels:rotate_token` rake task

`lib/tasks/channels.rake` — a thin operator wrapper; the new token comes from an
ENV var (not a rake argument) so it doesn't land in shell history/process args:

```ruby
namespace :channels do
  desc "Rotate a municipality's WhatsApp access_token. ENV: MUNICIPALITY_SLUG, ROTATE_TOKEN, ACTOR_EMAIL"
  task rotate_token: :environment do
    slug  = ENV.fetch("MUNICIPALITY_SLUG")
    token = ENV.fetch("ROTATE_TOKEN")
    actor = ApplicationRecord.connected_to(role: :admin) { User.find_by!(email: ENV.fetch("ACTOR_EMAIL")) }
    muni  = ApplicationRecord.connected_to(role: :admin) { Municipality.find_by!(slug: slug) }

    result = MunicipalityChannels::RotateToken.call(municipality_id: muni.id, new_token: token, by: actor)
    abort("[channels:rotate_token] failed: #{result.reason} #{result.message}") if result.failure?
    puts "[channels:rotate_token] rotated channel for #{slug} (phone_number_id=#{result.payload[:channel].phone_number_id})"
  end
end
```

### 3. Audit event

`channel.token_rotated` (platform-scope, `municipality_id: nil` on the DomainEvent
row, payload has the municipality_id/phone_number_id/actor). It flows into the
existing `domain_events` audit trail / events panel. The token value is never in
the payload.

## Testing (RSpec, run in the container)

Command spec (`spec/commands/municipality_channels/rotate_token_spec.rb`), admin
harness (`use_transactional_tests = false` + `as_admin` + `clean_admin_tables`,
since channels/memberships are control-plane):

- **platform_operator rotates**: token changes to the new value; a
  `channel.token_rotated` DomainEvent exists whose payload does **NOT** contain the
  token; `Result.ok`.
- **municipal_admin of the channel's city rotates**: `Result.ok`, token changed.
- **municipal_admin of another city**: `Result.fail(:forbidden)`; token unchanged.
- **a user with no qualifying membership**: `:forbidden`.
- **no active channel for the municipality**: `:not_found`.
- **blank/nil new_token**: `:invalid`; token unchanged.

(Build the user + memberships + channel via `as_admin`. Assert the token by
reloading the channel under `as_admin` and comparing `access_token`.)

The rake task is a thin wrapper (ENV → command); covered indirectly by the command
spec. An optional focused task test may stub ENV and assert it calls the command.

## Out of scope / follow-ups

- Authenticated HTTP endpoint for municipal_admin self-service rotation, with
  step-up MFA (Admin::Api is read-only; a municipal write surface is separate).
- Pre-swap verification of the new token against the Graph API.
- Automated refresh via the Meta token API.
- Keeping a short grace window with the previous token (not needed — single
  in-place value, read fresh per send).

## Workflow

Card F-01.9 In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`.
