# Purge domain_events (F-07.3) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a recurring housekeeping job that hard-deletes `domain_events` rows older than a 12-month retention window.

**Architecture:** A single `PurgeDomainEventsJob` mirroring `PurgeProcessedEventsJob` (`prepend AdminRoleJob`, `queue_as :housekeeping`, bulk `delete_all` by the indexed `occurred_at`), scheduled daily in `config/recurring.yml`.

**Tech Stack:** Rails 8.1, Solid Queue (RSpec, run in the `api-dev` container).

## Global Constraints

- **Commits in English.** `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER:** `docker exec api-dev bundle exec rspec <path>`.
- Cross-tenant job MUST use `prepend AdminRoleJob` (NOT `include` — `include` silently skips the admin-connection wrap).
- Cut by `occurred_at` (indexed); retention `older_than_months: 12`; hard `delete_all`; no migration; no domain event emitted.

---

### Task 1: `PurgeDomainEventsJob` + recurring schedule

**Files:**
- Create: `apps/api/app/jobs/purge_domain_events_job.rb`
- Modify: `apps/api/config/recurring.yml`
- Test: `apps/api/spec/jobs/purge_domain_events_job_spec.rb` (create)

**Interfaces:**
- Produces: `PurgeDomainEventsJob#perform(older_than_months: 12)` — deletes `DomainEvent` rows with `occurred_at < older_than_months.months.ago`, logs `[purge_domain_events] deleted=<n> cutoff=<iso8601>`.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/jobs/purge_domain_events_job_spec.rb`. The job runs under `AdminRoleJob` (admin/BYPASSRLS), so use the admin-connection harness (mirrors `spec/jobs/purge_processed_events_job_spec.rb` / `sweep_abandoned_conversations_job_spec.rb`):

```ruby
require "rails_helper"
require Rails.root.join("spec/support/admin_rls")

RSpec.describe PurgeDomainEventsJob, type: :job do
  self.use_transactional_tests = false

  before { clean_admin_tables }
  after  { clean_admin_tables }

  # Cria um domain_event com occurred_at explícito (default seria Time.current).
  def make_event(muni_id, occurred_at:, name: "triage.completed")
    as_admin do
      ev = DomainEvent.create!(name: name, payload: {}, municipality_id: muni_id, occurred_at: occurred_at)
      ev.id
    end
  end

  let(:muni_id) do
    as_admin { Municipality.create!(name: "Purge City", slug: "purge-city", ibge_code: "3500070").id }
  end

  it "deletes events older than the 12-month window and keeps recent ones" do
    old_id    = make_event(muni_id, occurred_at: 13.months.ago)
    recent_id = make_event(muni_id, occurred_at: 1.month.ago)

    described_class.new.perform(older_than_months: 12)

    as_admin do
      expect(DomainEvent.exists?(old_id)).to be(false)
      expect(DomainEvent.exists?(recent_id)).to be(true)
    end
  end

  it "keeps an event just inside the window (strict < cutoff)" do
    just_inside_id = make_event(muni_id, occurred_at: 11.months.ago)
    described_class.new.perform(older_than_months: 12)
    as_admin { expect(DomainEvent.exists?(just_inside_id)).to be(true) }
  end

  it "honors a custom window" do
    two_months_id = make_event(muni_id, occurred_at: 2.months.ago)
    one_week_id    = make_event(muni_id, occurred_at: 1.week.ago)
    described_class.new.perform(older_than_months: 1)
    as_admin do
      expect(DomainEvent.exists?(two_months_id)).to be(false)
      expect(DomainEvent.exists?(one_week_id)).to be(true)
    end
  end
end
```

If `Municipality.create!`/`DomainEvent.create!` attributes don't match the real models, align them (check the factory and `app/models/domain_event.rb`) — the asserted behavior (old deleted, recent kept, custom window) is the requirement.

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/jobs/purge_domain_events_job_spec.rb`
Expected: FAIL — `uninitialized constant PurgeDomainEventsJob`.

- [ ] **Step 3: Create the job**

Create `apps/api/app/jobs/purge_domain_events_job.rb`:

```ruby
# Purga domain_events além da janela de retenção (auditoria: 12 meses).
# Ver ADR-0005/0014. delete_all cross-tenant sob rota_admin (BYPASSRLS).
class PurgeDomainEventsJob < ApplicationJob
  prepend AdminRoleJob
  queue_as :housekeeping

  def perform(older_than_months: 12)
    cutoff = older_than_months.months.ago
    count = DomainEvent.where("occurred_at < ?", cutoff).delete_all
    Rails.logger.info("[purge_domain_events] deleted=#{count} cutoff=#{cutoff.iso8601}")
  end
end
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/jobs/purge_domain_events_job_spec.rb`
Expected: PASS (3 examples, 0 failures).

- [ ] **Step 5: Add the recurring schedule**

In `apps/api/config/recurring.yml`, inside `default: &default` (e.g. after the `purge_processed_events` entry), add:

```yaml
  purge_domain_events:
    class: PurgeDomainEventsJob
    queue: housekeeping
    schedule: "every day at 4:45am America/Sao_Paulo"
    args: { older_than_months: 12 }
```

- [ ] **Step 6: Verify recurring.yml parses**

Run: `docker exec api-dev bundle exec ruby -ryaml -e "YAML.load_file('config/recurring.yml'); puts 'recurring.yml OK'"`
Expected: `recurring.yml OK` (no parse error; the new key is present under `default`).

- [ ] **Step 7: Commit**

```bash
git -C apps/api add app/jobs/purge_domain_events_job.rb config/recurring.yml spec/jobs/purge_domain_events_job_spec.rb
git -C apps/api commit -m "Add recurring purge of domain_events beyond 12-month retention"
git -C apps/api log --oneline -1
```

---

## Wrap-up (after the task)

- Run `docker exec api-dev bundle exec rspec spec/jobs/purge_domain_events_job_spec.rb` — expect green.
- Move the board card (F-07.3) to Done, then Verified.
- Sync is a separate explicit step (user-authorized).

## Self-Review notes

- **Spec coverage:** the job (occurred_at cut, 12-month default, delete_all, log) + the recurring entry → Task 1 Steps 3 & 5; retention/boundary/custom-window behavior → Task 1 Step 1 tests. All spec sections mapped.
- **Placeholder scan:** none.
- **Type consistency:** `perform(older_than_months:)` signature is consistent between the job (Step 3), the tests (Step 1), and the recurring args (Step 5).
- **Pattern fidelity:** `prepend AdminRoleJob` (not include), `queue_as :housekeeping`, `delete_all` + single log line — matches `PurgeProcessedEventsJob` exactly.
