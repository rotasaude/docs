# Channel Token Rotation (F-01.9) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an audited, authorized, zero-downtime rotation of a municipality's WhatsApp `access_token`, via a `MunicipalityChannels::RotateToken` command and a thin `channels:rotate_token` rake task.

**Architecture:** A `Result`-returning command enforces custody (platform_operator, or municipal_admin of the channel's city), updates `access_token` in place under the admin connection (control-plane, RLS-exempt), and audits via `Platform.audit` without the token value. A rake task wraps it for operators (new token from ENV).

**Tech Stack:** Rails 8.1, RSpec. Run specs in the `api-dev` container.

## Global Constraints

- **Commits in English.** `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER:** `docker exec api-dev bundle exec rspec <path>`.
- Custody = role: `platform_operator` (any city) OR `municipal_admin` of the channel's `municipality_id`. Enforced in the command.
- Zero-downtime: in-place `access_token` update (Outbound reads the token fresh per send).
- Audit `channel.token_rotated` via `Platform.audit` — payload has municipality_id/phone_number_id/actor id, **NEVER the token value**.
- `MunicipalityChannel`/`Membership`/`User` are control-plane/RLS-exempt — read/write them under `ApplicationRecord.connected_to(role: :admin)`.
- `Result` API: `Result.ok(payload_hash)` / `Result.fail(reason, message:)`; `result.ok?`/`failure?`/`reason`/`payload`.
- No migration; no HTTP endpoint (deferred).

---

### Task 1: `MunicipalityChannels::RotateToken` + rake task

**Files:**
- Create: `apps/api/app/commands/municipality_channels/rotate_token.rb`
- Create: `apps/api/lib/tasks/channels.rake`
- Test: `apps/api/spec/commands/municipality_channels/rotate_token_spec.rb` (create)

**Interfaces:**
- Produces: `MunicipalityChannels::RotateToken.call(municipality_id:, new_token:, by:) -> Result` — `ok(channel:)` on success; `fail(:invalid)` (blank token), `fail(:forbidden)` (custody), `fail(:not_found)` (no active channel). Rake task `channels:rotate_token` (ENV: MUNICIPALITY_SLUG, ROTATE_TOKEN, ACTOR_EMAIL).

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/commands/municipality_channels/rotate_token_spec.rb`. Channels/memberships are control-plane, so use the admin harness:

```ruby
require "rails_helper"
require Rails.root.join("spec/support/admin_rls")

RSpec.describe MunicipalityChannels::RotateToken, type: :model do
  self.use_transactional_tests = false
  before { clean_admin_tables }
  after  { clean_admin_tables }

  # Builds a municipality + active channel + a user with the given membership.
  # role/for_muni control the membership; returns a struct of ids/objects.
  def scenario(role:, for_muni: :same)
    as_admin do
      muni  = Municipality.create!(name: "Tok City", slug: "tok-city-#{SecureRandom.hex(3)}", ibge_code: "3500#{rand(100..999)}")
      other = Municipality.create!(name: "Other City", slug: "other-#{SecureRandom.hex(3)}", ibge_code: "3501#{rand(100..999)}")
      channel = MunicipalityChannel.create!(municipality: muni, phone_number_id: "PN#{SecureRandom.hex(3)}",
                                            waba_id: "WABA", display_phone_number: "+5511", access_token: "OLD", active: true)
      user = User.create!(email: "actor-#{SecureRandom.hex(3)}@x.com", password: "dev-password-123")
      unless role.nil?
        muni_id = role == "platform_operator" ? nil : (for_muni == :same ? muni.id : other.id)
        Membership.create!(user: user, role: role, municipality_id: muni_id, granted_at: Time.current)
      end
      OpenStruct.new(muni: muni, channel: channel, user: user)
    end
  end

  def token_of(channel_id)
    as_admin { MunicipalityChannel.find(channel_id).access_token }
  end

  it "rotates for a platform_operator and audits without the token value" do
    s = scenario(role: "platform_operator")
    result = described_class.call(municipality_id: s.muni.id, new_token: "NEW-TOKEN", by: s.user)
    expect(result.ok?).to be(true)
    expect(token_of(s.channel.id)).to eq("NEW-TOKEN")
    event = as_admin { DomainEvent.where(name: "channel.token_rotated").order(:occurred_at).last }
    expect(event).to be_present
    expect(event.payload.to_json).not_to include("NEW-TOKEN")
    expect(event.payload["phone_number_id"]).to eq(s.channel.phone_number_id)
  end

  it "rotates for a municipal_admin of the channel's city" do
    s = scenario(role: "municipal_admin", for_muni: :same)
    expect(described_class.call(municipality_id: s.muni.id, new_token: "NEW", by: s.user).ok?).to be(true)
    expect(token_of(s.channel.id)).to eq("NEW")
  end

  it "forbids a municipal_admin of another city and leaves the token unchanged" do
    s = scenario(role: "municipal_admin", for_muni: :other)
    result = described_class.call(municipality_id: s.muni.id, new_token: "NEW", by: s.user)
    expect(result.failure?).to be(true)
    expect(result.reason).to eq(:forbidden)
    expect(token_of(s.channel.id)).to eq("OLD")
  end

  it "forbids a user with no qualifying membership" do
    s = scenario(role: nil)
    expect(described_class.call(municipality_id: s.muni.id, new_token: "NEW", by: s.user).reason).to eq(:forbidden)
  end

  it "returns not_found when the municipality has no active channel" do
    s = scenario(role: "platform_operator")
    as_admin { s.channel.update!(active: false) }
    expect(described_class.call(municipality_id: s.muni.id, new_token: "NEW", by: s.user).reason).to eq(:not_found)
  end

  it "returns invalid for a blank token and leaves the token unchanged" do
    s = scenario(role: "platform_operator")
    expect(described_class.call(municipality_id: s.muni.id, new_token: "", by: s.user).reason).to eq(:invalid)
    expect(token_of(s.channel.id)).to eq("OLD")
  end
end
```

Adjust `User.create!`/`Membership.create!`/`Municipality.create!` attributes to the real models if they differ (check `spec/factories` and the models) — the asserted behavior (rotate / custody / not_found / invalid / no-token-in-audit) is the requirement. `require "ostruct"` at the top if `OpenStruct` isn't autoloaded.

- [ ] **Step 2: Run to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/commands/municipality_channels/rotate_token_spec.rb`
Expected: FAIL — `uninitialized constant MunicipalityChannels::RotateToken`.

- [ ] **Step 3: Implement the command**

Create `apps/api/app/commands/municipality_channels/rotate_token.rb`:

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

Note: `app/commands/` autoloads (`MunicipalityChannels::RotateToken` from `app/commands/municipality_channels/rotate_token.rb`). If `Platform.audit` opening its own admin connection while already inside `connected_to(role: :admin)` causes a nested-connection issue in the test, it is still correct (nested `connected_to` to the same role is a no-op); if a spec fails for that reason, report it rather than removing the audit.

- [ ] **Step 4: Run to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/commands/municipality_channels/rotate_token_spec.rb`
Expected: PASS (6 examples, 0 failures).

- [ ] **Step 5: Add the rake task**

Create `apps/api/lib/tasks/channels.rake`:

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

- [ ] **Step 6: Verify the rake task loads**

Run: `docker exec api-dev bundle exec rails -e "Rails.application.load_tasks; abort('missing') unless Rake::Task.task_defined?('channels:rotate_token'); puts 'channels:rotate_token OK'"`
Expected: `channels:rotate_token OK` (the task is defined and the file parses). (Do not actually invoke it — it needs ENV + real records.)

- [ ] **Step 7: Commit**

```bash
git -C apps/api add app/commands/municipality_channels/rotate_token.rb lib/tasks/channels.rake spec/commands/municipality_channels/rotate_token_spec.rb
git -C apps/api commit -m "Add audited zero-downtime rotation of channel access_token"
git -C apps/api log --oneline -1
```

---

## Wrap-up (after the task)

- Run `docker exec api-dev bundle exec rspec spec/commands/municipality_channels/rotate_token_spec.rb` — expect green.
- Move the board card (F-01.9) to Done, then Verified.
- Sync is a separate explicit step (user-authorized).

## Self-Review notes

- **Spec coverage:** command (custody, in-place swap, audit-without-token, reasons) → Task 1 Step 3 + tests; rake wrapper → Step 5. All spec sections mapped; endpoint/verify/auto-refresh explicitly deferred.
- **Custody correctness:** `authorized?` allows platform_operator (any) and municipal_admin of the target municipality only; the "another city" and "no membership" cases are asserted `:forbidden`.
- **No token leak:** the audit test asserts the serialized payload does not contain the new token; only municipality_id/phone_number_id/actor are audited.
- **Placeholder scan:** none.
