# Revogação depois da triagem e exclusão do cadastro — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Revogação anonimiza a triagem concluída sem atendimento, a trilha deixa de guardar respostas, e o cadastro do cidadão pode ser excluído no posto por duas pessoas (ADR 0026).

**Architecture:** Tudo no banco da cidade (ADR 0020). Duas migrações de cidade (`triages.anonymized_at`; `citizen_erasure_requests` + `citizens.erased_at`), triggers em `db/city_triggers.sql`, comandos em `app/commands/citizens/`, rotas de escrita sob `/attendance` (o `/admin/api` é só leitura), telas no módulo Atendimento do dashboard.

**Tech Stack:** Rails 8.1 + RSpec (api, no container `api`), React + TypeScript + TanStack Query + vitest (dashboard).

**Spec:** `docs/superpowers/specs/2026-10-01-lgpd-revocation-and-erasure-design.md` (ADR `docs/adr/0026.md`). Correção sobre a spec: as rotas da exclusão ficam em `/attendance/erasure_requests…`, não em `/admin/api` (que é só leitura, `config/routes.rb`).

## Global Constraints

- Worktree do api em `apps/api/.claude/lgpd-erasure` (branch `mod-7/lgpd-erasure`, já criada em origin/main `22535ac`, `config/master.key` copiado). Specs: `docker compose exec -T -w /rails/.claude/lgpd-erasure api bundle exec rspec <paths>` a partir da raiz do monorepo.
- Suíte completa só com `docker compose stop worker` antes e `docker compose start worker` depois.
- Migração de cidade nova: aplicar `db/city_triggers.sql` e o schema em `rota_saude_test_city_a/b` (`docker compose exec -T -e RAILS_ENV=test api bin/rails city:test_databases` a partir da worktree com `-w`), e avisar as outras sessões.
- git: `/opt/homebrew/bin/git`. Commits em inglês, Conventional Commits por extenso, trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Specs de request exigem `type: :request` (inferência desligada).
- Texto de interface em português.
- Origem da revogação em `consent.revoked`: exatamente `"web"`, `"whatsapp"` ou `"erasure"`.
- Marcador da exclusão: `"erased:" + SecureRandom.uuid`.
- Motivo de recusa: ≥ 10 caracteres depois de `strip`.

## Review Focus

- Revogação chegando no mesmo instante que o check-in da mesma triagem: o atendimento vence (triagem retida) ou a anonimização vence (check-in recusado); nunca atendimento sobre triagem vazia. Teste na Task 3.
- Conversa de WhatsApp sem `citizen_id` do mesmo telefone: entra na exclusão pelo telefone. Teste na Task 6.
- Atendimento criado entre o pedido e a confirmação: confirmação vira `retained`, nada apagado. Teste na Task 6.
- Admin que também é verificador tentando confirmar o próprio pedido: 403/`own_request`. Teste na Task 6 e 7.
- Mesmo CPF e celular entrando de novo depois da exclusão: cadastro novo, sem triagens antigas. Teste na Task 6.

---

### Task 1: Trilha só com referência

**Files:**
- Modify: `app/commands/complete_triage.rb` (publish de `triage.completed`/`triage.urgent`)
- Modify: `app/jobs/generate_report_job.rb`
- Modify: `app/commands/revoke_consent.rb`, `app/commands/conversation_advance.rb:116`, `app/controllers/citizen_api/triages_controller.rb:36`
- Test: `spec/commands/complete_triage_payload_spec.rb` (novo), `spec/jobs/generate_report_job_spec.rb`, `spec/commands/revoke_consent_spec.rb`

**Interfaces:**
- Produces: `RevokeConsent.call(conversation:, origin:)` com `origin` em `%w[web whatsapp erasure]` (substitui `reason:`). `consent.revoked` payload `{conversation_id, consent_id, origin}`.
- Produces: `triage.completed`/`triage.urgent` payload `{triage_id}`.

- [ ] **Step 1: Failing tests**

```ruby
# spec/commands/complete_triage_payload_spec.rb
require "rails_helper"

# ADR 0026: a trilha (imutável por 12 meses) só leva referência.
RSpec.describe "payload de triage.completed e triage.urgent" do
  before { Current.city = TEST_CITY_A }

  it "publica só o triage_id, sem respostas nem classificação" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    triage = completed_web_triage_for(citizen)

    event = DomainEvent.where(name: "triage.completed").find_by("payload->>'triage_id' = ?", triage.id)
    expect(event.payload).to eq("triage_id" => triage.id)
  end
end
```

```ruby
# spec/jobs/generate_report_job_spec.rb — novo exemplo
it "monta o outcome do relatório a partir da triagem, não do payload (ADR 0026)" do
  citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
  triage = completed_web_triage_for(citizen)
  ReportSnapshot.where(triage_id: triage.id).delete_all

  described_class.new.handle(triage_id: triage.id)

  expect(ReportSnapshot.find_by!(triage_id: triage.id).outcome).to eq(triage.reload.outcome)
end
```

```ruby
# spec/commands/revoke_consent_spec.rb — novo exemplo
it "grava a origem da revogação, nunca o texto do cidadão" do
  RevokeConsent.call(conversation: conversation, origin: "whatsapp")
  event = DomainEvent.where(name: "consent.revoked").order(:created_at).last
  expect(event.payload).to include("origin" => "whatsapp")
  expect(event.payload).not_to have_key("reason")
end

it "recusa origem desconhecida" do
  expect { RevokeConsent.call(conversation: conversation, origin: "apagar meus dados") }
    .to raise_error(ArgumentError, /origin/)
end
```

(Ajuste o `let(:conversation)` ao que o arquivo já usa; se não houver, crie uma conversa consentida pelo mesmo caminho dos exemplos existentes.)

- [ ] **Step 2: Run** `... rspec spec/commands/complete_triage_payload_spec.rb spec/jobs/generate_report_job_spec.rb spec/commands/revoke_consent_spec.rb` → FAIL (payload tem tier/trail; `origin:` desconhecido).

- [ ] **Step 3: Implement**

```ruby
# app/commands/complete_triage.rb (dentro de `if outcome.terminal?`)
@triage.complete!(outcome)
DomainEvents.publish("triage.completed", triage_id: @triage.id)
DomainEvents.publish("triage.urgent",    triage_id: @triage.id) if Protocols::Urgency.urgent?(outcome)
```

```ruby
# app/jobs/generate_report_job.rb
def handle(triage_id:, **)
  triage = Triage.find(triage_id)
  return if triage.report_snapshot

  token = ReportSnapshot.mint_token
  ReportSnapshot.create!(
    triage: triage,
    protocol_definition: triage.protocol_definition,
    outcome: triage.outcome,
    payload: build_payload(triage),
    token: token,
    signature: ReportSnapshot.sign(token),
    expires_at: Time.current + EXPIRATION
  )
end
```

```ruby
# app/commands/revoke_consent.rb
ORIGINS = %w[web whatsapp erasure].freeze

def self.call(conversation:, origin:)
  raise ArgumentError, "origin must be one of #{ORIGINS.join(', ')}" unless ORIGINS.include?(origin)

  new(conversation: conversation, origin: origin).call
end

def initialize(conversation:, origin:)
  @conversation = conversation
  @origin = origin
end
# ...e no publish:
DomainEvents.publish("consent.revoked", conversation_id: @conversation.id, consent_id: active.id, origin: @origin)
```

`conversation_advance.rb:116` → `RevokeConsent.call(conversation: @conversation, origin: "whatsapp")`; `triages_controller.rb:36` → `origin: "web"`. Rode `grep -rn "RevokeConsent.call\|reason: \"citizen_web\"" app spec` e troque os chamadores de spec também.

- [ ] **Step 4: Run** os três arquivos + `spec/jobs/alert_municipality_job_spec.rb spec/jobs/notify_citizen_job_spec.rb spec/jobs/update_dashboard_job_spec.rb spec/commands/conversation_advance_spec.rb spec/requests/citizen_api` → PASS.

- [ ] **Step 5: Commit** `feat: keep only references in triage and revocation events` (body: `Refs rotasaude/api#30`).

---

### Task 2: Revogação anonimiza a triagem concluída sem atendimento

**Files:**
- Create: `db/city_migrate/20261001000001_add_anonymized_at_to_triages.rb`
- Modify: `db/city_schema.rb` (coluna + versão), `db/city_triggers.sql` (`rota_triage_neighborhood_guard`)
- Modify: `app/jobs/anonymize_revoked_triage_job.rb`; Create: `app/commands/triages/anonymize.rb`
- Test: `spec/jobs/anonymize_revoked_triage_job_spec.rb`, `spec/models/triage_neighborhood_guard_spec.rb` (se existir o spec do trigger, acrescente nele)

**Interfaces:**
- Produces: `Triages::Anonymize.call(triage)` → `true` se anonimizou, `false` se retida (tem atendimento) ou já anonimizada. Trava a linha (`lock!`) e reconfere `Attendance.exists?(triage_id:)`.
- Produces: coluna `triages.anonymized_at`.

- [ ] **Step 1: Failing tests**

```ruby
# spec/jobs/anonymize_revoked_triage_job_spec.rb — novos exemplos
describe "triagem concluída (ADR 0026)" do
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let!(:triage) { completed_web_triage_for(citizen) }
  let(:conversation) { triage.conversation }

  it "anonimiza a concluída sem atendimento, com bairro e anonymized_at" do
    triage.update_columns(neighborhood_id: Neighborhood.create!(name: "Centro").id) if triage.neighborhood_id.nil?
    described_class.new.handle(conversation_id: conversation.id)

    triage.reload
    expect(triage.status).to eq("completed")
    expect([triage.answers, triage.outcome, triage.tier, triage.priority, triage.neighborhood_id])
      .to eq([{}, nil, nil, nil, nil])
    expect(triage.anonymized_at).to be_present
  end

  it "não toca a concluída que virou atendimento" do
    Attendance.create!(triage: triage, citizen: citizen, health_unit: create_unit, checked_in_by_user: staff,
                       checked_in_at: Time.current, check_in_method: "code")
    expect { described_class.new.handle(conversation_id: conversation.id) }.not_to(change { triage.reload.attributes })
  end

  it "é idempotente" do
    described_class.new.handle(conversation_id: conversation.id)
    expect { described_class.new.handle(conversation_id: conversation.id) }.not_to(change { triage.reload.anonymized_at })
  end
end
```

(`staff`: `User.create!(email_address: "s-#{SecureRandom.hex(3)}@x.com", password: "secret123")`. Ajuste a criação de `Neighborhood` aos atributos reais do modelo.)

```ruby
# trigger: bairro vai a nulo com anonymized_at
it "aceita bairro nulo na concluída quando anonymized_at é preenchido, e recusa sem ele" do
  t = completed_web_triage_for(Citizen.create!(cpf: "52998224725", phone: "+5541998765432"))
  t.update_columns(neighborhood_id: Neighborhood.create!(name: "Centro").id)
  expect { ApplicationRecord.transaction(requires_new: true) { t.update_columns(neighborhood_id: nil) } }
    .to raise_error(ActiveRecord::StatementInvalid, /neighborhood_id never changes/)
  t.update_columns(neighborhood_id: nil, anonymized_at: Time.current)
  expect(t.reload.neighborhood_id).to be_nil
end
```

- [ ] **Step 2: Run** → FAIL (coluna não existe).

- [ ] **Step 3: Implement**

```ruby
# db/city_migrate/20261001000001_add_anonymized_at_to_triages.rb
# ADR 0026: a revogação anonimiza também a triagem concluída sem atendimento;
# anonymized_at marca a anonimização e libera o bairro nulo no trigger.
class AddAnonymizedAtToTriages < ActiveRecord::Migration[8.1]
  def up
    add_column :triages, :anonymized_at, :timestamptz
    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    remove_column :triages, :anonymized_at
    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end
end
```

`db/city_schema.rb`: `t.timestamptz "anonymized_at"` em `triages`; versão `2026_10_01_000001`.

`db/city_triggers.sql`, em `rota_triage_neighborhood_guard`:

```sql
IF NEW.neighborhood_id IS NULL
   AND (NEW.status = 'aborted_by_revocation' OR NEW.anonymized_at IS NOT NULL) THEN
  RETURN NEW;
END IF;
```

(Atualize o comentário acima da função: ADR 0026.) O `down` reexecuta o arquivo; como ele referencia `NEW.anonymized_at`, o `down` deve primeiro reescrever a função sem a coluna — use no `down`:

```ruby
def down
  execute <<~SQL
    CREATE OR REPLACE FUNCTION rota_triage_neighborhood_guard() RETURNS trigger AS $fn$
    BEGIN
      IF NEW.neighborhood_id IS DISTINCT FROM OLD.neighborhood_id THEN
        IF NEW.neighborhood_id IS NULL AND NEW.status = 'aborted_by_revocation' THEN
          RETURN NEW;
        END IF;
        RAISE EXCEPTION 'triages: neighborhood_id never changes after insert (only to NULL on revocation)';
      END IF;
      RETURN NEW;
    END;
    $fn$ LANGUAGE plpgsql;
  SQL
  remove_column :triages, :anonymized_at
end
```

```ruby
# app/commands/triages/anonymize.rb
# ADR 0026: apaga o conteúdo clínico de uma triagem que não virou atendimento.
# Trava a linha e reconfere o atendimento: o check-in trava a mesma linha
# (Attendances::CheckIn), então um dos dois vence, nunca os dois.
module Triages
  module Anonymize
    def self.call(triage)
      triage.with_lock do
        next false if triage.anonymized_at.present? || Attendance.exists?(triage_id: triage.id)

        triage.update_columns(answers: {}, outcome: nil, tier: nil, priority: nil, current_step: nil,
                              neighborhood_id: nil, anonymized_at: Time.current, updated_at: Time.current)
        true
      end
    end
  end
end
```

```ruby
# app/jobs/anonymize_revoked_triage_job.rb
def handle(conversation_id:, **)
  Triage.where(conversation_id: conversation_id, status: %w[aborted_by_revocation completed]).find_each do |t|
    Triages::Anonymize.call(t)
  end
  Campaigns::ForgetRevokedRecipients.call(conversation_id: conversation_id)
end
```

(Atualize o cabeçalho do job: ADR 0026.)

- [ ] **Step 4:** Aplique nos bancos de teste: `docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/lgpd-erasure api bin/rails city:test_databases`. Run os specs da task + `spec/jobs/anonymize_revoked_triage_job_spec.rb spec/services/analytics spec/architecture/database_config_parity_spec.rb` → PASS.

- [ ] **Step 5: Commit** `feat: anonymize a completed triage without attendance on revocation` (body: `Refs rotasaude/api#30`). Avise as outras sessões (migração de cidade `20261001000001`).

---

### Task 3: Check-in recusa triagem anonimizada ou revogada

**Files:**
- Modify: `app/commands/attendances/check_in_eligibility.rb`, `app/commands/attendances/check_in.rb`, `app/commands/attendances/check_in_by_exception.rb`
- Test: `spec/commands/attendances/check_in_eligibility_spec.rb` (ou o spec de check-in existente)

**Interfaces:**
- Consumes: `triages.anonymized_at` (Task 2), `Triages::Anonymize` (Task 2).
- Produces: `CheckInEligibility.check(triage)` devolve `:triage_not_eligible` para triagem anonimizada ou de conversa `revoked`.

- [ ] **Step 1: Failing tests**

```ruby
describe "revogação (ADR 0026)" do
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:triage) { completed_web_triage_for(citizen) }

  it "recusa triagem anonimizada" do
    triage.update_columns(anonymized_at: Time.current)
    expect(Attendances::CheckInEligibility.check(triage)).to eq(:triage_not_eligible)
    expect(Attendances::CheckInEligibility.eligible_for(Citizen.where(id: citizen.id))).to be_empty
  end

  it "recusa logo depois do COMMIT da revogação, antes do job anonimizar" do
    RevokeConsent.call(conversation: triage.conversation, origin: "web")
    expect(Attendances::CheckInEligibility.check(triage.reload)).to eq(:triage_not_eligible)
  end
end
```

E a corrida (Review Focus 1), no spec do `Attendances::CheckIn`: anonimize a triagem entre a leitura sem lock e a transação, como `deactivate_before_transaction` faz com a unidade, e espere `Result.fail(:triage_not_eligible)` e nenhum `Attendance`:

```ruby
it "não cria atendimento se a triagem foi anonimizada antes do lock" do
  code = Citizens::IssueCheckInCode.call(triage: triage).payload.fetch(:code) # use o emissor real do arquivo
  allow(ApplicationRecord).to receive(:transaction).and_wrap_original do |original, *args, **kwargs, &block|
    triage.update_columns(anonymized_at: Time.current)
    original.call(*args, **kwargs, &block)
  end
  result = Attendances::CheckIn.call(cpf: citizen.cpf, code: code, health_unit_id: create_unit.id,
                                     document_checked: false, by: staff)
  expect(result.reason).to eq(:triage_not_eligible)
  expect(Attendance.where(triage_id: triage.id)).to be_empty
end
```

- [ ] **Step 2: Run** → FAIL.

- [ ] **Step 3: Implement**

```ruby
# check_in_eligibility.rb
def check(triage)
  return :triage_not_eligible unless triage.status_completed? && triage.conversation.channel_web?
  return :triage_not_eligible if triage.anonymized_at.present? || triage.conversation.state == "revoked"
  return :already_checked_in if Attendance.exists?(triage_id: triage.id)
  return :triage_too_old if triage.completed_at.nil? || triage.completed_at < WINDOW.ago

  :ok
end

def eligible_for(citizens)
  Triage.joins(:conversation)
        .where(conversations: { channel: "web", citizen_id: citizens.select(:id) })
        .where.not(conversations: { state: "revoked" })
        .where(status: "completed", anonymized_at: nil).where("triages.completed_at >= ?", WINDOW.ago)
        .where.not(id: Attendance.where.not(triage_id: nil).select(:triage_id))
        .order(completed_at: :desc)
end
```

(Confira o nome real do estado/enum de `Conversation` — `state: :revoked` — e use o mesmo.)

Em `CheckIn.call`, no ramo sem `appointment`, dentro da transação: `triage.lock!` antes de `CheckInEligibility.check(triage)`. Em `CheckInByException`, dentro da transação, antes do `Attendance.create!`: `triage.lock!` e `next/return Result.fail(:triage_not_eligible) unless CheckInEligibility.check(triage) == :ok` (use o mesmo padrão `result = …` do arquivo para sair da transação).

- [ ] **Step 4: Run** `spec/commands/attendances spec/requests/check_ins* spec/requests/citizen_api` → PASS.

- [ ] **Step 5: Commit** `fix: refuse check-in of a revoked or anonymized triage` (body: `Refs rotasaude/api#30`).

---

### Task 4: Schema da exclusão

**Files:**
- Create: `db/city_migrate/20261001000002_create_citizen_erasure_requests.rb`, `app/models/citizen_erasure_request.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/citizen.rb`, `app/services/city_encryption.rb` (`CITY_KEYED_TARGETS`)
- Test: `spec/models/citizen_erasure_request_spec.rb`

**Interfaces:**
- Produces: `CitizenErasureRequest` (`STATUSES = %w[pending confirmed rejected retained]`, `belongs_to :presented_citizen, class_name: "Citizen"`, `belongs_to :requested_by_user, class_name: "User"`, `belongs_to :decided_by_user, class_name: "User", optional: true`, `encrypts :cpf, deterministic: true, key_provider: CityDeterministicKeyProvider.new`, scope `pending`).
- Produces: `citizens.erased_at`; `Citizen.not_erased` scope.

- [ ] **Step 1: Failing tests**

```ruby
require "rails_helper"

RSpec.describe CitizenErasureRequest do
  before { Current.city = TEST_CITY_A }

  def attempt = ApplicationRecord.transaction(requires_new: true) { yield }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:verifier) { User.create!(email_address: "v-#{SecureRandom.hex(3)}@x.com", password: "secret123") }
  let(:admin) { User.create!(email_address: "a-#{SecureRandom.hex(3)}@x.com", password: "secret123") }
  let!(:request) do
    described_class.create!(cpf: citizen.cpf, presented_citizen: citizen, requested_by_user: verifier,
                            document_checked: true, status: "pending")
  end

  it "recusa DELETE" do
    expect { attempt { described_class.where(id: request.id).delete_all } }
      .to raise_error(ActiveRecord::StatementInvalid, /citizen_erasure_requests is append-only/)
  end

  it "aceita uma decisão e recusa a segunda" do
    request.update_columns(status: "rejected", decided_by_user_id: admin.id, decided_at: Time.current,
                           reject_reason: "documento não confere")
    expect { attempt { request.update_columns(status: "confirmed") } }
      .to raise_error(ActiveRecord::StatementInvalid, /already decided/)
  end

  it "recusa mudar quem pediu ou o par apresentado" do
    expect { attempt { request.update_columns(requested_by_user_id: admin.id) } }
      .to raise_error(ActiveRecord::StatementInvalid, /only the decision columns/)
  end

  it "exige documento conferido e decisão coerente com o status" do
    expect { attempt { described_class.create!(cpf: "x", presented_citizen: citizen, requested_by_user: verifier, document_checked: false, status: "pending") } }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_citizen_erasure_requests_document/)
  end

  it "guarda o CPF cifrado" do
    raw = described_class.connection.select_value(described_class.sanitize_sql(["SELECT cpf FROM citizen_erasure_requests WHERE id = ?", request.id]))
    expect(raw).not_to include("52998224725")
    expect(described_class.where(cpf: "52998224725").pluck(:id)).to eq([request.id])
  end
end
```

- [ ] **Step 2: Run** → FAIL.

- [ ] **Step 3: Implement**

```ruby
# db/city_migrate/20261001000002_create_citizen_erasure_requests.rb
# ADR 0026: pedido de exclusão (Art. 18). O pedido é a prova da resposta da
# cidade: só acréscimo, decisão gravada uma vez (trigger em city_triggers.sql).
class CreateCitizenErasureRequests < ActiveRecord::Migration[8.1]
  def up
    add_column :citizens, :erased_at, :timestamptz

    create_table :citizen_erasure_requests, id: :uuid do |t|
      t.string :cpf, null: false
      t.references :presented_citizen, type: :uuid, null: false, foreign_key: { to_table: :citizens }
      t.references :requested_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }
      t.boolean :document_checked, null: false
      t.string :status, null: false
      t.references :decided_by_user, type: :uuid, foreign_key: { to_table: :users }
      t.timestamptz :decided_at
      t.text :reject_reason
      t.timestamps
    end
    add_index :citizen_erasure_requests, :cpf
    add_index :citizen_erasure_requests, :cpf, unique: true, where: "status = 'pending'",
              name: "idx_citizen_erasure_requests_one_pending"
    add_check_constraint :citizen_erasure_requests, "document_checked", name: "ck_citizen_erasure_requests_document"
    add_check_constraint :citizen_erasure_requests, "status IN ('pending','confirmed','rejected','retained')",
                         name: "ck_citizen_erasure_requests_status"
    add_check_constraint :citizen_erasure_requests, "(status = 'pending') = (decided_at IS NULL)",
                         name: "ck_citizen_erasure_requests_decision"
    add_check_constraint :citizen_erasure_requests,
                         "status <> 'rejected' OR length(btrim(coalesce(reject_reason, ''))) >= 10",
                         name: "ck_citizen_erasure_requests_reason"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    drop_table :citizen_erasure_requests
    remove_column :citizens, :erased_at
  end
end
```

(`retained` nasce decidido: `decided_at` = criação e `decided_by_user_id` nulo — o check acima aceita; a decisão foi do sistema.)

```sql
-- db/city_triggers.sql — citizen_erasure_requests (ADR 0026): só acréscimo; a
-- decisão sai de pending UMA vez; na confirmação o cpf vira o marcador.
CREATE OR REPLACE FUNCTION rota_citizen_erasure_request_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'citizen_erasure_requests is append-only: DELETE refused';
  END IF;
  IF OLD.status <> 'pending' THEN
    RAISE EXCEPTION 'citizen_erasure_requests: already decided';
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id
     OR NEW.presented_citizen_id IS DISTINCT FROM OLD.presented_citizen_id
     OR NEW.requested_by_user_id IS DISTINCT FROM OLD.requested_by_user_id
     OR NEW.document_checked IS DISTINCT FROM OLD.document_checked
     OR NEW.created_at IS DISTINCT FROM OLD.created_at
     OR (NEW.cpf IS DISTINCT FROM OLD.cpf AND NEW.status <> 'confirmed') THEN
    RAISE EXCEPTION 'citizen_erasure_requests: only the decision columns may change';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF to_regclass('public.citizen_erasure_requests') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS citizen_erasure_requests_guard ON citizen_erasure_requests';
    EXECUTE 'CREATE TRIGGER citizen_erasure_requests_guard
      BEFORE UPDATE OR DELETE ON citizen_erasure_requests
      FOR EACH ROW EXECUTE FUNCTION rota_citizen_erasure_request_guard()';
    EXECUTE 'DROP TRIGGER IF EXISTS citizen_erasure_requests_append_only_truncate ON citizen_erasure_requests';
    EXECUTE 'CREATE TRIGGER citizen_erasure_requests_append_only_truncate
      BEFORE TRUNCATE ON citizen_erasure_requests
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
  END IF;
END
$do$;
```

Atenção: o TRUNCATE recusado pode quebrar a limpeza de alguma suíte que trunca tabelas da cidade; se `city:test_databases` ou o harness truncarem, retire o trigger de TRUNCATE e documente como em `memberships`.

```ruby
# app/models/citizen_erasure_request.rb
# ADR 0026: pedido de exclusão do cadastro (Art. 18). Só acréscimo; ver
# db/city_triggers.sql (citizen_erasure_requests_guard).
class CitizenErasureRequest < ApplicationRecord
  STATUSES = %w[pending confirmed rejected retained].freeze

  encrypts :cpf, deterministic: true, key_provider: CityDeterministicKeyProvider.new

  belongs_to :presented_citizen, class_name: "Citizen"
  belongs_to :requested_by_user, class_name: "User"
  belongs_to :decided_by_user, class_name: "User", optional: true

  validates :status, inclusion: { in: STATUSES }
  scope :pending, -> { where(status: "pending") }
end
```

`app/models/citizen.rb`: `scope :not_erased, -> { where(erased_at: nil) }`. `city_encryption.rb`: `[ CitizenErasureRequest, :cpf ],` em `CITY_KEYED_TARGETS`. Schema: versão `2026_10_01_000002`, tabela e coluna como na migração.

- [ ] **Step 4:** `city:test_databases` de novo; run `spec/models/citizen_erasure_request_spec.rb spec/architecture spec/jobs/reencryption_job_spec.rb` (a lista esperada de modelos no `reencryption_job_spec` ganha `CitizenErasureRequest`) → PASS.

- [ ] **Step 5: Commit** `feat: add the citizen erasure request record` (body: `Refs rotasaude/docs#2`). Avise as sessões (migração `20261001000002`).

---

### Task 5: Pedir a exclusão (`Citizens::RequestErasure`)

**Files:**
- Create: `app/commands/citizens/request_erasure.rb`
- Test: `spec/commands/citizens/request_erasure_spec.rb`

**Interfaces:**
- Consumes: `CitizenErasureRequest` (Task 4).
- Produces: `Citizens::RequestErasure.call(cpf:, document_checked:, by:)` → `Result.ok(request:)` (status `pending` ou `retained`) ou `Result.fail(:invalid_cpf | :document_check_required | :citizen_not_found | :already_pending)`.
- Produces: `Citizens::RequestErasure.pairs_of(cpf)` → `Citizen.not_erased.where(cpf:)`; `Citizens::RequestErasure.attended?(citizens)` → bool (algum `Attendance` com `citizen_id` dos pares).

- [ ] **Step 1: Failing tests**

```ruby
require "rails_helper"

RSpec.describe Citizens::RequestErasure do
  before { Current.city = TEST_CITY_A }

  let(:verifier) { User.create!(email_address: "v-#{SecureRandom.hex(3)}@x.com", password: "secret123") }
  let!(:pair_a) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let!(:pair_b) { Citizen.create!(cpf: "52998224725", phone: "+5541998760000") }

  def request!(cpf: "529.982.247-25", checked: true) = described_class.call(cpf: cpf, document_checked: checked, by: verifier)

  it "cria pedido pendente para o CPF, apresentando um par" do
    result = request!
    expect(result.payload[:request]).to have_attributes(status: "pending", requested_by_user_id: verifier.id)
    expect(DomainEvent.where(name: "citizen.erasure_requested").last.payload.keys).to contain_exactly("request_id")
  end

  it "nasce retido quando algum par do CPF tem atendimento" do
    triage = completed_web_triage_for(pair_b)
    Attendance.create!(triage: triage, citizen: pair_b, health_unit: create_unit, checked_in_by_user: verifier,
                       checked_in_at: Time.current, check_in_method: "code")
    expect(request!.payload[:request]).to have_attributes(status: "retained", decided_at: be_present)
  end

  it "recusa sem documento conferido, CPF inválido, CPF sem cadastro e pedido já pendente" do
    expect(request!(checked: false).reason).to eq(:document_check_required)
    expect(request!(cpf: "123").reason).to eq(:invalid_cpf)
    expect(request!(cpf: "111.444.777-35").reason).to eq(:citizen_not_found)
    request!
    expect(request!.reason).to eq(:already_pending)
  end
end
```

- [ ] **Step 2: Run** → FAIL.

- [ ] **Step 3: Implement**

```ruby
# app/commands/citizens/request_erasure.rb
# ADR 0026: o citizen_verifier registra no posto, com documento conferido, o
# pedido de exclusão de TODOS os pares do CPF. CPF com par atendido nasce
# retido por base legal.
module Citizens
  module RequestErasure
    module_function

    def call(cpf:, document_checked:, by:)
      return Result.fail(:document_check_required) unless document_checked == true

      digits = CitizenIdentity::Cpf.normalize(cpf)
      return Result.fail(:invalid_cpf) unless digits

      pairs = pairs_of(digits)
      return Result.fail(:citizen_not_found) if pairs.empty?

      request = nil
      ApplicationRecord.transaction do
        retained = attended?(pairs)
        request = CitizenErasureRequest.create!(
          cpf: digits, presented_citizen: pairs.order(:created_at).first, requested_by_user: by,
          document_checked: true, status: retained ? "retained" : "pending", decided_at: (Time.current if retained)
        )
        DomainEvents.publish(retained ? "citizen.erasure_retained" : "citizen.erasure_requested", request_id: request.id)
      end
      Result.ok(request: request)
    rescue ActiveRecord::RecordNotUnique => e
      raise unless e.message.include?("idx_citizen_erasure_requests_one_pending")

      Result.fail(:already_pending)
    end

    def pairs_of(digits) = Citizen.not_erased.where(cpf: digits)

    def attended?(pairs) = Attendance.where(citizen_id: pairs.select(:id)).exists?
  end
end
```

- [ ] **Step 4: Run** → PASS. **Step 5: Commit** `feat: let a verifier request a citizen erasure at the counter` (body: `Refs rotasaude/docs#2`).

---

### Task 6: Confirmar e recusar (`Citizens::Erase`, `Citizens::RejectErasure`)

**Files:**
- Create: `app/commands/citizens/erase.rb`, `app/commands/citizens/reject_erasure.rb`
- Test: `spec/commands/citizens/erase_spec.rb`, `spec/commands/citizens/reject_erasure_spec.rb`

**Interfaces:**
- Consumes: `RevokeConsent.call(conversation:, origin: "erasure")` (Task 1), `Triages::Anonymize` (Task 2, mas aqui sem a checagem de atendimento: nenhum par tem), `RequestErasure.pairs_of/attended?` (Task 5).
- Produces: `Citizens::Erase.call(request:, by:)` → `Result.ok(request:)` com status `confirmed` ou `retained`; `Result.fail(:not_pending | :own_request)`.
- Produces: `Citizens::RejectErasure.call(request:, reason:, by:)` → `Result.ok(request:)`; `Result.fail(:not_pending | :reason_too_short)`.
- Produces: `Citizens::Erase::TOMBSTONE_PREFIX = "erased:"`.

- [ ] **Step 1: Failing tests**

```ruby
require "rails_helper"

RSpec.describe Citizens::Erase do
  before { Current.city = TEST_CITY_A }

  let(:verifier) { User.create!(email_address: "v-#{SecureRandom.hex(3)}@x.com", password: "secret123") }
  let(:admin) { User.create!(email_address: "a-#{SecureRandom.hex(3)}@x.com", password: "secret123") }
  let!(:pair) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let!(:triage) { completed_web_triage_for(pair) }
  let(:request) { Citizens::RequestErasure.call(cpf: "52998224725", document_checked: true, by: verifier).payload[:request] }

  it "deixa a casca: sem CPF, telefone, sessões, códigos nem triagem legível" do
    CitizenSession.create!(phone: pair.phone, token_digest: SecureRandom.hex, expires_at: 1.day.from_now) # ajuste aos atributos reais
    InboundMessage.create!(message_id: "wamid.#{SecureRandom.hex(4)}", from: pair.phone, kind: "text")
    whatsapp = Conversation.create!(phone: pair.phone, state: :greeting) # sem citizen_id (Review Focus 2)

    result = described_class.call(request: request, by: admin)

    expect(result.payload[:request].status).to eq("confirmed")
    pair.reload
    expect(pair.cpf).to start_with("erased:")
    expect(pair.phone).to start_with("erased:")
    expect(pair.erased_at).to be_present
    expect(Citizen.where(cpf: "52998224725")).to be_empty
    expect(CitizenSession.where(phone: "+5541998765432")).to be_empty
    expect(InboundMessage.where(from: "+5541998765432")).to be_empty
    expect(Conversation.where(phone: "+5541998765432")).to be_empty
    expect(whatsapp.reload.phone).to start_with("erased:")
    expect(triage.reload).to have_attributes(answers: {}, tier: nil, anonymized_at: be_present)
    expect(Consent.where(conversation_id: triage.conversation_id, revoked_at: nil)).to be_empty
    expect(result.payload[:request].reload.cpf).to start_with("erased:")
    expect(DomainEvent.where(name: "citizen.erased").last.payload.keys).to contain_exactly("request_id")
  end

  it "nenhuma tabela da cidade decifra o CPF ou o telefone depois" do
    described_class.call(request: request, by: admin)
    CityEncryption::CITY_KEYED_TARGETS.each do |model, attr|
      expect(model.where(attr => "52998224725").exists?).to be(false), "#{model}.#{attr} (cpf)"
      expect(model.where(attr => "+5541998765432").exists?).to be(false), "#{model}.#{attr} (phone)"
    end
  end

  it "vira retido se o atendimento chegou depois do pedido (Review Focus 3)" do
    request
    Attendance.create!(triage: triage, citizen: pair, health_unit: create_unit, checked_in_by_user: verifier,
                       checked_in_at: Time.current, check_in_method: "code")
    expect(described_class.call(request: request, by: admin).payload[:request].status).to eq("retained")
    expect(pair.reload.cpf).to eq("52998224725")
  end

  it "recusa quem pediu e pedido já decidido (Review Focus 4)" do
    expect(described_class.call(request: request, by: verifier).reason).to eq(:own_request)
    described_class.call(request: request, by: admin)
    expect(described_class.call(request: request.reload, by: admin).reason).to eq(:not_pending)
  end

  it "o mesmo CPF e celular entrando de novo viram cadastro novo (Review Focus 5)" do
    described_class.call(request: request, by: admin)
    fresh = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    expect(fresh.id).not_to eq(pair.id)
    expect(Conversation.where(citizen_id: fresh.id)).to be_empty
  end
end
```

```ruby
RSpec.describe Citizens::RejectErasure do
  before { Current.city = TEST_CITY_A }
  let(:verifier) { User.create!(email_address: "v-#{SecureRandom.hex(3)}@x.com", password: "secret123") }
  let(:admin) { User.create!(email_address: "a-#{SecureRandom.hex(3)}@x.com", password: "secret123") }
  let!(:pair) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:request) { Citizens::RequestErasure.call(cpf: "52998224725", document_checked: true, by: verifier).payload[:request] }

  it "recusa com motivo e não apaga nada" do
    expect(described_class.call(request: request, reason: "curto", by: admin).reason).to eq(:reason_too_short)
    result = described_class.call(request: request, reason: "documento com foto não confere", by: admin)
    expect(result.payload[:request]).to have_attributes(status: "rejected", decided_by_user_id: admin.id)
    expect(pair.reload.cpf).to eq("52998224725")
  end
end
```

- [ ] **Step 2: Run** → FAIL.

- [ ] **Step 3: Implement**

```ruby
# app/commands/citizens/erase.rb
# ADR 0026: o municipal_admin confirma a exclusão (step-up no controller).
# Numa transação: reconfere a retenção; senão revoga, anonimiza, apaga o
# apagável e troca CPF e telefone por um marcador que não identifica ninguém.
module Citizens
  module Erase
    TOMBSTONE_PREFIX = "erased:".freeze

    module_function

    def call(request:, by:)
      result = nil
      ApplicationRecord.transaction do
        request.lock!
        next result = Result.fail(:not_pending) unless request.status == "pending"
        next result = Result.fail(:own_request) if request.requested_by_user_id == by.id

        pairs = RequestErasure.pairs_of(request.cpf).lock.to_a
        if RequestErasure.attended?(Citizen.where(id: pairs.map(&:id)))
          request.update!(status: "retained", decided_by_user: by, decided_at: Time.current)
          DomainEvents.publish("citizen.erasure_retained", request_id: request.id)
          next result = Result.ok(request: request)
        end

        pairs.each { |citizen| erase_pair(citizen) }
        request.update!(status: "confirmed", decided_by_user: by, decided_at: Time.current, cpf: tombstone)
        DomainEvents.publish("citizen.erased", request_id: request.id)
        result = Result.ok(request: request)
      end
      result
    end

    def erase_pair(citizen)
      phone = citizen.phone
      conversations = Conversation.where(citizen_id: citizen.id).or(Conversation.where(phone: phone))

      conversations.find_each do |conversation|
        RevokeConsent.call(conversation: conversation, origin: "erasure") if conversation.active_consent
        Triage.where(conversation_id: conversation.id).find_each do |t|
          t.update_columns(answers: {}, outcome: nil, tier: nil, priority: nil, current_step: nil,
                           neighborhood_id: nil, anonymized_at: t.anonymized_at || Time.current, updated_at: Time.current)
        end
        conversation.update_columns(phone: tombstone, updated_at: Time.current)
      end

      CitizenSession.where(phone: phone).delete_all
      OtpChallenge.where(phone: phone).delete_all
      InboundMessage.where(from: phone).delete_all
      OutboundMessage.where(to: phone).delete_all
      CitizenVerificationCode.where(citizen_id: citizen.id).delete_all
      CitizenContactPreference.where(citizen_id: citizen.id).delete_all
      CampaignRecipient.where(citizen_id: citizen.id).delete_all

      citizen.update_columns(cpf: tombstone, phone: tombstone, neighborhood_id: nil, erased_at: Time.current,
                             updated_at: Time.current)
    end

    def tombstone = "#{TOMBSTONE_PREFIX}#{SecureRandom.uuid}"
  end
end
```

Notas:
- `update_columns` com atributo cifrado: confirme que o Rails cifra no `update_columns` (cifra no `serialize` do tipo — sim); o teste de varredura prova.
- `RevokeConsent` abre transação aninhada: junta-se à externa; o `consent.revoked` enfileira o job depois do COMMIT, que acha as triagens já anonimizadas e não faz nada.
- Se `CitizenVerificationCode` tiver FK de `citizen_verifications` ou outro bloqueio, o `delete_all` falha alto no teste; aí mantenha as linhas e só anule o `code_digest`.

```ruby
# app/commands/citizens/reject_erasure.rb
module Citizens
  module RejectErasure
    module_function

    def call(request:, reason:, by:)
      return Result.fail(:reason_too_short) if reason.to_s.strip.length < 10

      result = nil
      ApplicationRecord.transaction do
        request.lock!
        next result = Result.fail(:not_pending) unless request.status == "pending"

        request.update!(status: "rejected", decided_by_user: by, decided_at: Time.current, reject_reason: reason.to_s.strip)
        DomainEvents.publish("citizen.erasure_rejected", request_id: request.id)
        result = Result.ok(request: request)
      end
      result
    end
  end
end
```

- [ ] **Step 4: Run** → PASS. **Step 5: Commit** `feat: confirm or reject a citizen erasure, leaving a tombstone` (body: `Refs rotasaude/docs#2`).

---

### Task 7: Rotas, controller e política

**Files:**
- Create: `app/controllers/erasure_requests_controller.rb`
- Modify: `config/routes.rb` (escopo `/attendance`), `app/policies/citizen_verification_policy.rb` (sem mudança se `verify?`/`manage?` bastarem)
- Test: `spec/requests/erasure_requests_spec.rb`

**Interfaces:**
- Consumes: Tasks 5 e 6.
- Produces (JSON):
  - `POST /attendance/erasure_requests` `{cpf, document_checked}` → 201 `{request: {id, status, created_at}}`; 422 `invalid_cpf`/`document_check_required`; 404 `citizen_not_found`; 409 `already_pending`. Só `citizen_verifier`.
  - `GET /attendance/erasure_requests` → `{requests: [{id, created_at, requested_by, pairs, phone_masked}]}` só `pending`. Só `municipal_admin`.
  - `POST /attendance/erasure_requests/:id/confirm` → 200 `{request: {id, status}}`; 401 `mfa_required` sem step-up; 403 `own_request`; 409 `not_pending`. Só `municipal_admin`.
  - `POST /attendance/erasure_requests/:id/reject` `{reason}` → 200; 422 `reason_too_short`; 409 `not_pending`. Só `municipal_admin`.

- [ ] **Step 1: Failing tests** (`type: :request`): papel errado → 403 nas quatro rotas; verificador cria (201 pending); admin sem step-up confirma → 401 `mfa_required` e o cidadão continua com CPF; admin com step-up confirma → 200 `confirmed`; admin que também é verificador e fez o pedido → 403 `own_request`; lista não traz CPF inteiro (`expect(response.body).not_to include("52998224725")`). Use o helper de step-up que os specs de `setup#deactivate_user` usam (`grep -rn "step_up\|reauthenticat" spec/requests/setup* spec/support`).

- [ ] **Step 2: Run** → FAIL.

- [ ] **Step 3: Implement**

```ruby
# config/routes.rb, dentro de scope "/attendance"
get  "erasure_requests",             to: "erasure_requests#index"
post "erasure_requests",             to: "erasure_requests#create"
post "erasure_requests/:id/confirm", to: "erasure_requests#confirm"
post "erasure_requests/:id/reject",  to: "erasure_requests#reject"
```

```ruby
# app/controllers/erasure_requests_controller.rb
# ADR 0026: exclusão do cadastro (Art. 18), no posto, por duas pessoas.
#   POST /attendance/erasure_requests             {cpf, document_checked}  citizen_verifier
#   GET  /attendance/erasure_requests                                      municipal_admin
#   POST /attendance/erasure_requests/:id/confirm  (step-up)               municipal_admin
#   POST /attendance/erasure_requests/:id/reject   {reason}                municipal_admin
# O CPF vai no corpo, nunca na URL.
class ErasureRequestsController < ApplicationController
  include Authentication
  include AttendanceAccess
  include MfaStepUp

  ERROR_STATUS = {
    invalid_cpf: :unprocessable_entity, document_check_required: :unprocessable_entity,
    reason_too_short: :unprocessable_entity, citizen_not_found: :not_found,
    already_pending: :conflict, not_pending: :conflict, own_request: :forbidden
  }.freeze

  before_action :require_verifier, only: :create
  before_action :require_admin, only: %i[index confirm reject]

  def create
    result = Citizens::RequestErasure.call(cpf: params[:cpf], document_checked: params[:document_checked] == true,
                                           by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { request: request_json(result.payload[:request]) }, status: :created
  end

  def index
    rows = CitizenErasureRequest.pending.includes(:requested_by_user, :presented_citizen).order(:created_at)
    render json: { requests: rows.map { |r| index_json(r) } }
  end

  def confirm
    request = CitizenErasureRequest.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless request
    return require_step_up! unless reauthenticated_recently?

    result = Citizens::Erase.call(request: request, by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { request: request_json(result.payload[:request]) }
  end

  def reject
    request = CitizenErasureRequest.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless request

    result = Citizens::RejectErasure.call(request: request, reason: params[:reason], by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { request: request_json(result.payload[:request]) }
  end

  private

  def request_json(r) = { id: r.id, status: r.status, created_at: r.created_at.iso8601 }

  def index_json(r)
    {
      id: r.id, created_at: r.created_at.iso8601, requested_by: r.requested_by_user.email_address,
      pairs: Citizens::RequestErasure.pairs_of(r.cpf).count,
      phone_masked: CitizenIdentity::Phone.mask(r.presented_citizen.phone)
    }
  end
end
```

(Se `ApplicationController` exigir `allow_operator_grant_access` ou algo parecido para escrita, siga o que `AttendanceController` faz — o operador com grant é só leitura e não deve passar.)

- [ ] **Step 4: Run** `spec/requests/erasure_requests_spec.rb spec/architecture` (há guardas de rota e de grant do operador) → PASS.

- [ ] **Step 5: Commit** `feat: expose citizen erasure requests at the counter` (body: `Refs rotasaude/docs#2`).

---

### Task 8: Suíte completa e revisão do api

- [ ] `docker compose stop worker`; suíte completa da worktree; `docker compose start worker`. Esperado 0 falhas.
- [ ] Revisão do diff de produção por subagente (`git diff origin/main...HEAD`), com foco em: caminhos que agora recusam (check-in, trigger de bairro), cifra no `update_columns`, `RevokeConsent` aninhado, ordem de deploy.
- [ ] Corrigir achados com TDD e commit.

---

### Task 9: Dashboard

**Files (repo dashboard, worktree `apps/dashboard/.claude/lgpd-erasure`, branch `mod-7/lgpd-erasure`):**
- Modify: `src/lib/api.ts` (4 funções), `src/modules/Attendance.tsx`
- Create: `src/modules/attendance/ErasureRequest.tsx`, `src/modules/attendance/ErasureRequests.tsx`
- Test: `src/modules/attendance/ErasureRequest.test.tsx`, `src/modules/attendance/ErasureRequests.test.tsx`

**Interfaces:**
- Consumes: as rotas da Task 7.
- Produces (`src/lib/api.ts`):

```ts
export interface ErasureRequestResult { id: string; status: "pending" | "confirmed" | "rejected" | "retained"; created_at: string }
export interface PendingErasure { id: string; created_at: string; requested_by: string; pairs: number; phone_masked: string }
export function requestErasure(cpf: string, documentChecked: boolean): Promise<{ request: ErasureRequestResult }>
export function listPendingErasures(): Promise<{ requests: PendingErasure[] }>
export function confirmErasure(id: string): Promise<{ request: ErasureRequestResult }>
export function rejectErasure(id: string, reason: string): Promise<{ request: ErasureRequestResult }>
```

Implemente com o mesmo helper de requisição que `verifyCitizen`/`revokeVerification` usam em `api.ts` (POST com JSON, erro com `error` do corpo).

- [ ] **Step 1: Failing tests (vitest)**
  - `ErasureRequest`: CPF inválido desabilita o botão; sem marcar "Conferi o documento com foto" desabilita; envio chama `requestErasure("52998224725", true)`; resposta `pending` mostra "Pedido registrado. Um administrador precisa confirmar."; `retained` mostra "Cadastro retido por base legal: há registro de atendimento. Nada foi apagado."; 409 mostra "Já existe um pedido pendente para este CPF."; 404 "Nenhum cadastro com este CPF."
  - `ErasureRequests`: lista mostra data, quem pediu, nº de cadastros e celular mascarado; "Confirmar exclusão" pede confirmação ("A exclusão é irreversível…") e chama `confirmErasure` pelo fluxo de step-up (`useStepUp`, como em `Team.tsx`); 403 `own_request` mostra "Quem registrou o pedido não pode confirmá-lo."; "Recusar" exige motivo de 10+ caracteres.
- [ ] **Step 2: Run** `npx vitest run src/modules/attendance/Erasure*` → FAIL.
- [ ] **Step 3: Implement** os dois componentes no estilo de `Attendance.tsx` (`Panel`, `inputStyle`, `buttonStyle`, `attendanceError`). Em `Attendance.tsx`: `ErasureRequest` aparece para `canVerify`; `ErasureRequests` para `isAdmin`.
- [ ] **Step 4: Run** `npx vitest run` e `npx tsc --noEmit` → PASS.
- [ ] **Step 5: Commit** `feat: add citizen erasure requests to the attendance module`.

---

### Task 10: Docs, board e publicação

- [ ] docs (worktree `--detach` em origin/main):
  - `modulos/07--lgpd-auditoria.md`: F-07.15 (revogação anonimiza a concluída sem atendimento; trilha só com referência), F-ID novo para a exclusão (perguntar ao usuário o número/escopo antes de criar) e Histórico.
  - `operacao/atender-requisicao-lgpd.md`: seção Eliminação passa a descrever o fluxo no posto; Revogação com a regra do atendimento.
  - `adr/0014.md` e `adr/0025.md`: nota apontando o 0026.
  - Commit `docs: record revocation after triage and citizen erasure (ADR 0026)`.
- [ ] Merge `--no-ff` de api e dashboard sobre origin/main destacado, push `HEAD:main` (com autorização do usuário); docs push.
- [ ] Termo de consentimento: texto novo da spec §3.6 publicado em cada cidade de dev (`city:consent_term:publish`); produção fica para o rollout.
- [ ] Board #2: api#30 e docs#2 → Em tratamento no início da Task 1; → Resolvido no fim (fechar a issue antes de mover). Pendências novas (spec §8) como issues `ciclo:01` com o template, no board #2.
- [ ] Catálogo: se a funcionalidade de revogação/LGPD do catálogo mudar, mandar o texto (220–420 caracteres) para a sessão "Módulos planejados e status".
- [ ] Avisar a sessão "Módulos planejados e status": ADR e decisão em uma linha, commits e suíte, estado dos dois cards.
