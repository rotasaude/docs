# Check-in na unidade e desfecho do atendimento (subprojeto 3) — plano

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** o cidadão faz check-in numa unidade de saúde (código gerado a partir da triagem no wpda, ou exceção por CPF com motivo), o que abre um atendimento ligado à triagem; o atendente registra o desfecho (liberado, encaminhado, saiu sem atendimento) e o cidadão acompanha no wpda; o `municipal_admin` mantém o cadastro mínimo de unidades.

**Architecture:** uma migração de cidade cria `health_units` e `attendances` (com trigger que só deixa encerrar uma vez) e dá finalidade (`purpose`) e `triage_id` ao código do balcão. Comandos em `Citizens::` (código) e `Attendances::` (check-in, exceção, desfecho) reaproveitam a conferência do código e a validação do subprojeto 2. Controllers novos sob `/attendance/*` (sessão da cidade) e uma rota nova do cidadão. O dashboard amplia o módulo "Atendimento"; o wpda ganha "Cheguei na unidade" e a situação do atendimento.

**Tech Stack:** Rails 8.1, PostgreSQL (apps/api); Vite + React 18 + TanStack Query + Vitest (apps/dashboard sem jest-dom; apps/wpda com jest-dom e user-event).

**Spec:** `docs/superpowers/specs/2026-09-24-citizen-attendance-check-in-design.md` (commit `41a0d2d`). ADR: `docs/adr/0018.md`.

## Global Constraints

- Commits: Conventional Commits **em inglês**, tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Git: `/opt/homebrew/bin/git`. Repositórios separados. Branch `feat/citizen-attendance-check-in` em api, dashboard e wpda, a partir de `main`. Nunca em `main`, nunca push. Staging explícito.
- **Antes de criar cada branch:** `git branch --show-current` = `main` e `git status --short` vazio. Outra sessão ("Página /maintenance em development") usa api/dashboard/maintenance; se o repo não estiver em `main` limpo, pare e avise.
- api: comandos no container a partir da raiz do monorepo (`docker compose exec -T api bundle exec rspec <arquivos>`); suíte completa em primeiro plano com `docker compose stop worker` antes e `docker compose start worker` depois (sempre). Acima de ~3 min é regressão. Specs de request com `type: :request`; `sign_in_as`, `sign_in_citizen`, `json_post`, `issue_code_for` já existem; sem factory `:user`; `travel` exige `include ActiveSupport::Testing::TimeHelpers`; arquivo novo em `spec/support` precisa de `require_relative` em `spec/rails_helper.rb`.
- **Migração de cidade:** `db/city_migrate/20260925000001_create_attendances.rb` (confira `ls db/city_migrate | tail -1`; a última é `20260924000001`). Dump à mão em `db/city_schema.rb`; juiz: `spec/services/city_schema_spec.rb`. Triggers novos em `db/city_triggers.sql`, **protegidos por `to_regclass(...) IS NOT NULL`** (o arquivo é re-executado por migrações antigas). Depois da migração, recarregue os bancos de teste com `docker compose exec -T api bin/rails city:test_databases` se a paridade reclamar de estado velho.
- Evento novo: `DomainEvents.bind "<nome>", to: []`.
- Valores do spec: janela de check-in **3 dias** depois de `triages.completed_at`; código **6 dígitos**, **10 min**, **5 tentativas**, um ativo por cidadão (qualquer finalidade); motivo da exceção **≥ 10 caracteres** após `btrim`; limite **30 a cada 10 minutos** por servidor em `check_ins/lookup`, `check_ins`, `check_ins/search`, `check_ins/exception`; desfechos `discharged` | `referred` | `left`; tipos de unidade `ubs` | `upa` | `hospital` | `other`.
- Papéis: `citizen_verifier` faz check-in, exceção, lista de abertos e desfecho; `municipal_admin` mantém unidades. O CPF nunca vai na URL.
- Nunca imprimir código de balcão/OTP/TOTP em relatório ou commit. Nunca comparar ambiente por literal.
- dashboard: testes **sem jest-dom** e sem dependência nova (asserções simples: `getBy*`, `(el as HTMLButtonElement).disabled`, `queryBy*(...)).toBeNull()`). wpda: texto interativo ≥ 18 px, alvos ≥ 48 px.

## Review Focus

1. **Código de finalidade trocada:** o código gerado por "Validar no posto" no check-in, e o de "Cheguei na unidade" na validação, respondem `invalid_code` (e não `code_expired`). Teste na Task 2.
2. **Triagem de outro par/CPF na exceção:** `check_ins/exception` com `triage_id` de outro CPF, ou de triagem não elegível, responde `triage_not_eligible` sem criar nada. Teste na Task 3.
3. **Dois atendentes com o mesmo código de check-in:** um atendimento só; o segundo recebe `code_expired`. Teste na Task 3.
4. **Unidade desativada entre a escolha e o check-in** (ou como destino de encaminhamento): `invalid_unit`, sem criar nada; o dashboard volta para a escolha de unidade. Testes nas Tasks 3 e 6.
5. **Check-in de cidadão declarado com a caixa marcada:** validação e atendimento na mesma transação — se o check-in falhar (ex.: `already_checked_in`), a validação também não é gravada. Teste na Task 3.

---

## File Structure

**apps/api**
- Create: `db/city_migrate/20260925000001_create_attendances.rb`; Modify: `db/city_schema.rb`, `db/city_triggers.sql`
- Create: `app/models/health_unit.rb`, `app/models/attendance.rb`; Modify: `app/models/citizen_verification_code.rb`, `app/models/triage.rb`
- Create: `app/commands/citizens/issue_counter_code.rb`, `app/commands/citizens/issue_check_in_code.rb`; Modify: `app/commands/citizens/issue_verification_code.rb`, `app/commands/citizens/verification_code_match.rb`, `app/commands/citizens/verify.rb`
- Create: `app/commands/attendances/check_in_eligibility.rb`, `lookup_for_check_in.rb`, `check_in.rb`, `eligible_triages.rb`, `check_in_by_exception.rb`, `close.rb`
- Create: `app/controllers/concerns/attendance_access.rb`, `app/controllers/health_units_controller.rb`, `app/controllers/check_ins_controller.rb`, `app/controllers/attendances_controller.rb`, `app/controllers/citizen_api/check_in_codes_controller.rb`; Modify: `app/controllers/attendance_controller.rb`, `app/controllers/citizen_api/triages_controller.rb`, `config/routes.rb`, `config/initializers/domain_events.rb`, `config/initializers/filter_parameter_logging.rb`
- Specs em `spec/models/`, `spec/commands/citizens/`, `spec/commands/attendances/`, `spec/requests/`, `spec/requests/citizen_api/`; helper `spec/support/attendance_helpers.rb`

**apps/dashboard**
- Modify: `src/lib/api.ts`, `src/lib/attendance.ts`, `src/modules/Attendance.tsx`
- Create: `src/modules/attendance/UnitPicker.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/attendance/CheckIn.tsx`, `src/modules/attendance/OpenAttendances.tsx` (+ testes)

**apps/wpda**
- Create: `src/modules/citizen/CounterCodeStep.tsx`; Modify: `src/modules/citizen/VerificationCodeStep.tsx`, `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `src/lib/citizenApi.ts` (+ testes)

---

### Task 1: Tabelas, trigger e modelos

**Repo:** `apps/api`. Antes: confira `main` limpo e crie `feat/citizen-attendance-check-in`.

**Files:**
- Create: `db/city_migrate/20260925000001_create_attendances.rb`, `app/models/health_unit.rb`, `app/models/attendance.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/citizen_verification_code.rb`, `app/models/triage.rb`
- Test: `spec/models/health_unit_spec.rb`, `spec/models/attendance_spec.rb`, `spec/models/citizen_verification_code_purpose_spec.rb`; Create: `spec/support/attendance_helpers.rb` (+ `require_relative`)

**Interfaces:**
- Produces: `HealthUnit` (`KINDS = %w[ubs upa hospital other]`, `scope :active_units`, validação de nome único sem maiúsculas); `Attendance` (`belongs_to :triage, :citizen, :health_unit`, `:checked_in_by_user`, `:referral_unit` e `:closed_by_user` opcionais; `METHODS = %w[code cpf_exception]`; `OUTCOMES = %w[discharged referred left]`; `scope :open_attendances`; `#open?`); `Triage has_one :attendance`; `CitizenVerificationCode` com `purpose` (`verification` padrão | `check_in`), `belongs_to :triage, optional: true`, `scope :for_purpose(p)`; helpers de spec `create_unit(name = "UBS Centro", kind: "ubs")` e `completed_web_triage_for(citizen, completed_at: Time.current)`.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/support/attendance_helpers.rb
module AttendanceHelpers
  def create_unit(name = "UBS Centro", kind: "ubs", active: true)
    HealthUnit.create!(name: name, kind: kind, active: active)
  end

  # Triagem web concluída do cidadão. Usa o protocolo padrão; a data de
  # conclusão pode ser deslocada para testar a janela de 3 dias.
  def completed_web_triage_for(citizen, completed_at: Time.current)
    create_default_protocol! unless ProtocolDefinition.exists?(name: StartTriage::DEFAULT_PROTOCOL_NAME, status: "active")
    started = Citizens::StartConversation.call(citizen: citizen, consent_version: Consents.current_version, session_id: "s").payload
    Citizens::SubmitAnswer.call(conversation: started[:conversation], answer: "false", idempotency_key: SecureRandom.uuid)
    triage = started[:triage].reload
    triage.update_columns(completed_at: completed_at)
    triage
  end
end

RSpec.configure { |c| c.include AttendanceHelpers }
```

```ruby
# spec/models/health_unit_spec.rb
require "rails_helper"

RSpec.describe HealthUnit do
  it "nome único sem distinguir maiúsculas" do
    described_class.create!(name: "UBS Centro", kind: "ubs")
    dup = described_class.new(name: "ubs centro", kind: "upa")
    expect(dup).not_to be_valid
    expect { dup.save!(validate: false) }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "tipo precisa ser conhecido" do
    expect(described_class.new(name: "X", kind: "clinica")).not_to be_valid
  end

  it "lista só as ativas" do
    a = described_class.create!(name: "A", kind: "ubs")
    described_class.create!(name: "B", kind: "upa", active: false)
    expect(described_class.active_units).to eq([a])
  end
end
```

```ruby
# spec/models/attendance_spec.rb
require "rails_helper"

RSpec.describe Attendance do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:staff) { User.create!(email_address: "atendente@cidade.gov.br", password: "senha-segura-123") }
  let(:unit) { create_unit }
  let(:triage) { completed_web_triage_for(citizen) }

  def open!(**attrs)
    described_class.create!({ triage: triage, citizen: citizen, health_unit: unit, checked_in_by_user: staff,
                              checked_in_at: Time.current, check_in_method: "code" }.merge(attrs))
  end

  it "um atendimento por triagem" do
    open!
    expect { open! }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "exceção exige motivo com 10 caracteres" do
    expect { open!(check_in_method: "cpf_exception", exception_reason: "curto") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_attendances_exception_reason/)
    expect { open!(check_in_method: "cpf_exception", exception_reason: "cidadão sem celular") }.not_to raise_error
  end

  it "encerra uma vez; encaminhado exige destino ou descrição" do
    a = open!
    expect { a.update!(status: "closed", outcome: "referred", closed_by_user: staff, closed_at: Time.current) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_attendances_referral/)
    a.reload.update!(status: "closed", outcome: "referred", referral_note: "cardiologia", closed_by_user: staff,
                     closed_at: Time.current)
    expect { a.update!(outcome: "left") }.to raise_error(ActiveRecord::StatementInvalid, /already closed/)
  end

  it "não apaga e não muda o check-in" do
    a = open!
    expect { described_class.transaction(requires_new: true) { a.delete } }
      .to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    other = create_unit("UPA Norte", kind: "upa")
    expect { described_class.transaction(requires_new: true) { a.update_column(:health_unit_id, other.id) } }
      .to raise_error(ActiveRecord::StatementInvalid, /check-in columns/)
  end
end
```

```ruby
# spec/models/citizen_verification_code_purpose_spec.rb
require "rails_helper"

RSpec.describe CitizenVerificationCode do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }

  it "check_in exige triagem e verification não aceita triagem" do
    base = { citizen: citizen, code_digest: "x", expires_at: 10.minutes.from_now }
    expect { described_class.create!(base.merge(purpose: "check_in")) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_citizen_verification_codes_purpose_triage/)
    expect(described_class.create!(base).purpose).to eq("verification")
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/models/health_unit_spec.rb spec/models/attendance_spec.rb spec/models/citizen_verification_code_purpose_spec.rb`
Expected: FAIL (`uninitialized constant HealthUnit`).

- [ ] **Step 3: Migração**

```ruby
# db/city_migrate/20260925000001_create_attendances.rb
# Check-in na unidade e desfecho do atendimento (spec 2026-09-24-citizen-
# attendance-check-in-design §3; ADR 0018). Aditiva. attendances só aceita
# acréscimo, exceto encerrar uma vez — trigger em db/city_triggers.sql.
class CreateAttendances < ActiveRecord::Migration[8.1]
  def up
    create_table :health_units, id: :uuid do |t|
      t.string :name, null: false
      t.string :kind, null: false
      t.boolean :active, null: false, default: true
      t.timestamps
    end
    add_index :health_units, "lower((name)::text)", unique: true, name: "idx_health_units_name_ci"
    add_check_constraint :health_units, "kind::text = ANY (ARRAY['ubs', 'upa', 'hospital', 'other']::text[])",
                         name: "ck_health_units_kind"

    add_column :citizen_verification_codes, :purpose, :string, null: false, default: "verification"
    add_reference :citizen_verification_codes, :triage, type: :uuid, foreign_key: true, index: true
    add_check_constraint :citizen_verification_codes,
                         "purpose::text = ANY (ARRAY['verification', 'check_in']::text[])",
                         name: "ck_citizen_verification_codes_purpose"
    add_check_constraint :citizen_verification_codes,
                         "(purpose::text = 'check_in'::text) = (triage_id IS NOT NULL)",
                         name: "ck_citizen_verification_codes_purpose_triage"

    create_table :attendances, id: :uuid do |t|
      t.references :triage, type: :uuid, null: false, foreign_key: true, index: { unique: true }
      t.references :citizen, type: :uuid, null: false, foreign_key: true, index: true
      t.references :health_unit, type: :uuid, null: false, foreign_key: true, index: true
      t.references :checked_in_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: true
      t.datetime :checked_in_at, null: false
      t.string :check_in_method, null: false
      t.text :exception_reason
      t.string :status, null: false, default: "open"
      t.string :outcome
      t.references :referral_unit, type: :uuid, foreign_key: { to_table: :health_units }, index: true
      t.text :referral_note
      t.references :closed_by_user, type: :uuid, foreign_key: { to_table: :users }, index: true
      t.datetime :closed_at
      t.datetime :created_at, null: false
    end
    add_check_constraint :attendances, "check_in_method::text = ANY (ARRAY['code', 'cpf_exception']::text[])",
                         name: "ck_attendances_method"
    add_check_constraint :attendances,
                         "(check_in_method::text = 'code'::text AND exception_reason IS NULL) OR " \
                         "(check_in_method::text = 'cpf_exception'::text AND exception_reason IS NOT NULL " \
                         "AND length(btrim(exception_reason)) >= 10)",
                         name: "ck_attendances_exception_reason"
    add_check_constraint :attendances, "status::text = ANY (ARRAY['open', 'closed']::text[])",
                         name: "ck_attendances_status"
    add_check_constraint :attendances,
                         "(status::text = 'open'::text AND outcome IS NULL AND closed_by_user_id IS NULL " \
                         "AND closed_at IS NULL AND referral_unit_id IS NULL AND referral_note IS NULL) OR " \
                         "(status::text = 'closed'::text AND outcome IS NOT NULL AND closed_by_user_id IS NOT NULL " \
                         "AND closed_at IS NOT NULL)",
                         name: "ck_attendances_closing"
    add_check_constraint :attendances,
                         "outcome IS NULL OR outcome::text = ANY (ARRAY['discharged', 'referred', 'left']::text[])",
                         name: "ck_attendances_outcome"
    add_check_constraint :attendances,
                         "(outcome IS DISTINCT FROM 'referred' AND referral_unit_id IS NULL AND referral_note IS NULL) OR " \
                         "(outcome = 'referred' AND (referral_unit_id IS NOT NULL OR " \
                         "(referral_note IS NOT NULL AND length(btrim(referral_note)) > 0)))",
                         name: "ck_attendances_referral"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    execute "DROP TRIGGER IF EXISTS attendances_guard ON attendances"
    execute "DROP TRIGGER IF EXISTS attendances_append_only_truncate ON attendances"
    execute "DROP FUNCTION IF EXISTS rota_attendance_guard()"
    drop_table :attendances
    remove_check_constraint :citizen_verification_codes, name: "ck_citizen_verification_codes_purpose_triage"
    remove_check_constraint :citizen_verification_codes, name: "ck_citizen_verification_codes_purpose"
    remove_reference :citizen_verification_codes, :triage, foreign_key: true, index: true
    remove_column :citizen_verification_codes, :purpose
    drop_table :health_units
  end
end
```

- [ ] **Step 4: Trigger** (no fim de `db/city_triggers.sql`, no mesmo formato protegido do bloco de `citizen_verifications`)

```sql
-- attendances (spec 2026-09-24-citizen-attendance-check-in §3; ADR 0018): só
-- acréscimo, exceto encerrar UMA vez. O check-in nunca muda.
CREATE OR REPLACE FUNCTION rota_attendance_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'attendances is append-only: DELETE refused';
  END IF;
  IF OLD.status = 'closed' THEN
    RAISE EXCEPTION 'attendances: already closed';
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id
     OR NEW.triage_id IS DISTINCT FROM OLD.triage_id
     OR NEW.citizen_id IS DISTINCT FROM OLD.citizen_id
     OR NEW.health_unit_id IS DISTINCT FROM OLD.health_unit_id
     OR NEW.checked_in_by_user_id IS DISTINCT FROM OLD.checked_in_by_user_id
     OR NEW.checked_in_at IS DISTINCT FROM OLD.checked_in_at
     OR NEW.check_in_method IS DISTINCT FROM OLD.check_in_method
     OR NEW.exception_reason IS DISTINCT FROM OLD.exception_reason
     OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'attendances: the check-in columns never change';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF to_regclass('public.attendances') IS NOT NULL THEN
    DROP TRIGGER IF EXISTS attendances_guard ON attendances;
    CREATE TRIGGER attendances_guard
      BEFORE UPDATE OR DELETE ON attendances
      FOR EACH ROW EXECUTE FUNCTION rota_attendance_guard();
    DROP TRIGGER IF EXISTS attendances_append_only_truncate ON attendances;
    CREATE TRIGGER attendances_append_only_truncate
      BEFORE TRUNCATE ON attendances
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only();
  END IF;
END
$do$;
```

Siga exatamente a forma do guard `to_regclass` que o subprojeto 2 deixou no arquivo (leia o bloco de `citizen_verifications` antes); se lá a forma for outra, copie a de lá.

- [ ] **Step 5: Dump à mão** em `db/city_schema.rb`: versão `2026_09_25_000001`; tabelas `attendances` e `health_units` em ordem alfabética; colunas `purpose` e `triage_id` (+ índice e CHECKs) em `citizen_verification_codes`; `add_foreign_key` novos em ordem. O texto exato vem do Postgres: se a paridade falhar, migre um banco de dev (`docker compose exec -T api bin/rails city:migrate:all`) e copie de `pg_indexes`/`pg_get_constraintdef`.

- [ ] **Step 6: Modelos**

```ruby
# app/models/health_unit.rb
# Unidade de saúde da cidade (ADR 0018; começo do módulo 09): cadastro mínimo
# mantido pelo municipal_admin. Desativar some das listas; atendimentos
# antigos continuam apontando para ela.
class HealthUnit < ApplicationRecord
  KINDS = %w[ubs upa hospital other].freeze

  has_many :attendances, dependent: :restrict_with_error

  validates :name, presence: true, uniqueness: { case_sensitive: false }
  validates :kind, inclusion: { in: KINDS }

  scope :active_units, -> { where(active: true).order(:name) }
end
```

```ruby
# app/models/attendance.rb
# Atendimento numa unidade, aberto pelo check-in a partir de uma triagem
# (ADR 0018). Só acréscimo, exceto encerrar uma vez (trigger).
class Attendance < ApplicationRecord
  METHODS = %w[code cpf_exception].freeze
  OUTCOMES = %w[discharged referred left].freeze

  belongs_to :triage
  belongs_to :citizen
  belongs_to :health_unit
  belongs_to :checked_in_by_user, class_name: "User"
  belongs_to :referral_unit, class_name: "HealthUnit", optional: true
  belongs_to :closed_by_user, class_name: "User", optional: true

  scope :open_attendances, -> { where(status: "open") }

  def open?
    status == "open"
  end
end
```

Em `app/models/citizen_verification_code.rb`: `belongs_to :triage, optional: true`; `PURPOSES = %w[verification check_in].freeze`; `scope :for_purpose, ->(purpose) { where(purpose: purpose.to_s) }`. Em `app/models/triage.rb`: `has_one :attendance, dependent: :restrict_with_error`.

- [ ] **Step 7: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/models/health_unit_spec.rb spec/models/attendance_spec.rb spec/models/citizen_verification_code_purpose_spec.rb spec/services/city_schema_spec.rb spec/architecture spec/commands/citizens`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add db/city_migrate/20260925000001_create_attendances.rb db/city_schema.rb db/city_triggers.sql app/models/health_unit.rb app/models/attendance.rb app/models/citizen_verification_code.rb app/models/triage.rb spec/models/health_unit_spec.rb spec/models/attendance_spec.rb spec/models/citizen_verification_code_purpose_spec.rb spec/support/attendance_helpers.rb spec/rails_helper.rb
/opt/homebrew/bin/git commit -m "feat: add health units, attendances and counter code purposes" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Código do balcão com finalidade

**Repo:** `apps/api`.

**Files:**
- Create: `app/commands/citizens/issue_counter_code.rb`, `app/commands/citizens/issue_check_in_code.rb`, `app/commands/attendances/check_in_eligibility.rb`
- Modify: `app/commands/citizens/issue_verification_code.rb`, `app/commands/citizens/verification_code_match.rb`
- Test: `spec/commands/citizens/counter_code_purpose_spec.rb`, `spec/commands/attendances/check_in_eligibility_spec.rb`

**Interfaces:**
- Consumes: Task 1.
- Produces:
  - `Citizens::IssueCounterCode.call(citizen:, purpose:, triage: nil) -> Result ok(code:, expires_at:)` — invalida todos os códigos utilizáveis do cidadão (qualquer finalidade) e cria o novo.
  - `Citizens::IssueVerificationCode.call(citizen:)` — inalterado por fora (delegando a `IssueCounterCode` com `purpose: "verification"`).
  - `Citizens::IssueCheckInCode.call(citizen:, triage:) -> ok(code:, expires_at:) | fail(:triage_too_old | :already_checked_in | :triage_not_eligible)`.
  - `Citizens::VerificationCodeMatch.call(cpf:, code:, lock: false, purpose: "verification")` — só considera códigos daquela finalidade (inclusive na checagem de "recente"); payload ganha `triage:` (a triagem do código, `nil` na validação).
  - `Attendances::CheckInEligibility::WINDOW = 3.days`; `.check(triage) -> :ok | :triage_too_old | :already_checked_in | :triage_not_eligible`; `.eligible_for(citizens_scope) -> Triage relation` (web, `completed`, `completed_at >= 3.days.ago`, sem atendimento, mais recentes primeiro).

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/commands/citizens/counter_code_purpose_spec.rb
require "rails_helper"

RSpec.describe "Counter code purposes" do
  include ActiveSupport::Testing::TimeHelpers

  before { Current.city = TEST_CITY_A }
  after { Current.reset; travel_back; Rails.cache.clear }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:triage) { completed_web_triage_for(citizen) }

  def check_in_code
    Citizens::IssueCheckInCode.call(citizen: citizen, triage: triage).payload.fetch(:code)
  end

  it "o código de check-in aponta para a triagem" do
    code = check_in_code
    match = Citizens::VerificationCodeMatch.call(cpf: citizen.cpf, code: code, purpose: "check_in")
    expect(match.payload[:triage]).to eq(triage)
  end

  it "código de check-in na validação, e o contrário, dá invalid_code" do
    code = check_in_code
    expect(Citizens::VerificationCodeMatch.call(cpf: citizen.cpf, code: code).reason).to eq(:invalid_code)
    v = issue_code_for(citizen)
    expect(Citizens::VerificationCodeMatch.call(cpf: citizen.cpf, code: v, purpose: "check_in").reason).to eq(:invalid_code)
  end

  it "um código novo invalida o anterior de qualquer finalidade" do
    check_in_code
    issue_code_for(citizen)
    expect(CitizenVerificationCode.usable.where(citizen: citizen).pluck(:purpose)).to eq(["verification"])
  end

  it "só nasce para triagem concluída há 3 dias ou menos e sem atendimento" do
    old = completed_web_triage_for(Citizen.create!(cpf: "11144477735", phone: "+5541911112222"), completed_at: 4.days.ago)
    expect(Citizens::IssueCheckInCode.call(citizen: old.conversation.citizen, triage: old).reason).to eq(:triage_too_old)
  end
end
```

```ruby
# spec/commands/attendances/check_in_eligibility_spec.rb
require "rails_helper"

RSpec.describe Attendances::CheckInEligibility do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:staff) { User.create!(email_address: "a@cidade.gov.br", password: "senha-segura-123") }

  it "elegível: concluída há até 3 dias e sem atendimento" do
    fresh = completed_web_triage_for(citizen, completed_at: 71.hours.ago)
    expect(described_class.check(fresh)).to eq(:ok)
    expect(described_class.eligible_for(Citizen.where(id: citizen.id))).to eq([fresh])
  end

  it "antiga e já atendida não são elegíveis" do
    old = completed_web_triage_for(citizen, completed_at: 73.hours.ago)
    expect(described_class.check(old)).to eq(:triage_too_old)
    fresh = completed_web_triage_for(Citizen.create!(cpf: "11144477735", phone: "+5541911112222"))
    Attendance.create!(triage: fresh, citizen: fresh.conversation.citizen, health_unit: create_unit,
                       checked_in_by_user: staff, checked_in_at: Time.current, check_in_method: "code")
    expect(described_class.check(fresh)).to eq(:already_checked_in)
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/commands/citizens/counter_code_purpose_spec.rb spec/commands/attendances/check_in_eligibility_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implementação**

```ruby
# app/commands/citizens/issue_counter_code.rb
# Código do balcão (ADR 0017/0018): 6 dígitos, 10 minutos, um ativo por
# cidadão — gerar qualquer código invalida o anterior, de qualquer finalidade.
module Citizens
  class IssueCounterCode
    def self.call(citizen:, purpose:, triage: nil)
      code = format("%06d", SecureRandom.random_number(1_000_000))
      record = nil
      ApplicationRecord.transaction do
        CitizenVerificationCode.usable.where(citizen: citizen).update_all(expires_at: Time.current)
        record = CitizenVerificationCode.create!(
          citizen: citizen, purpose: purpose.to_s, triage: triage,
          code_digest: CitizenVerificationCode.digest(citizen.id, code),
          expires_at: CitizenVerificationCode::TTL.from_now
        )
      end
      Result.ok(code: code, expires_at: record.expires_at)
    end
  end
end
```

`app/commands/citizens/issue_verification_code.rb`: mantém o guard `already_verified` e troca o corpo por `IssueCounterCode.call(citizen: citizen, purpose: "verification")`.

```ruby
# app/commands/citizens/issue_check_in_code.rb
# "Cheguei na unidade" (spec 2026-09-24-citizen-attendance-check-in §4).
module Citizens
  class IssueCheckInCode
    def self.call(citizen:, triage:)
      state = Attendances::CheckInEligibility.check(triage)
      return Result.fail(state) unless state == :ok

      IssueCounterCode.call(citizen: citizen, purpose: "check_in", triage: triage)
    end
  end
end
```

```ruby
# app/commands/attendances/check_in_eligibility.rb
# Quais triagens podem receber check-in (spec §2.3): web, concluídas há 3 dias
# ou menos e ainda sem atendimento.
module Attendances
  module CheckInEligibility
    WINDOW = 3.days

    module_function

    def check(triage)
      return :triage_not_eligible unless triage.status_completed? && triage.conversation.channel_web?
      return :already_checked_in if Attendance.exists?(triage_id: triage.id)
      return :triage_too_old if triage.completed_at.nil? || triage.completed_at < WINDOW.ago

      :ok
    end

    def eligible_for(citizens)
      Triage.joins(:conversation)
            .where(conversations: { channel: "web", citizen_id: citizens.select(:id) })
            .where(status: "completed").where("triages.completed_at >= ?", WINDOW.ago)
            .where.not(id: Attendance.select(:triage_id))
            .order(completed_at: :desc)
    end
  end
end
```

`app/commands/citizens/verification_code_match.rb`: acrescente o parâmetro `purpose: "verification"`; `candidates = CitizenVerificationCode.usable.for_purpose(purpose).where(citizen: citizens)...`; a checagem de "recente" também filtra `for_purpose(purpose)`; o `Result.ok` passa a incluir `triage: hit.triage`. O resto (código malformado, tentativas, esgotado) fica igual.

- [ ] **Step 4: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/commands/citizens spec/commands/attendances spec/requests/attendance_spec.rb`
Expected: PASS (as specs do subprojeto 2 continuam verdes).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/commands/citizens/issue_counter_code.rb app/commands/citizens/issue_check_in_code.rb app/commands/citizens/issue_verification_code.rb app/commands/citizens/verification_code_match.rb app/commands/attendances/check_in_eligibility.rb spec/commands/citizens/counter_code_purpose_spec.rb spec/commands/attendances/check_in_eligibility_spec.rb
/opt/homebrew/bin/git commit -m "feat: give counter codes a purpose and issue check-in codes" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Comandos de check-in, exceção e desfecho

**Repo:** `apps/api`.

**Files:**
- Modify: `app/commands/citizens/verify.rb` (extrai `Citizens::Verify.record!`)
- Create: `app/commands/attendances/lookup_for_check_in.rb`, `check_in.rb`, `eligible_triages.rb`, `check_in_by_exception.rb`, `close.rb`
- Modify: `config/initializers/domain_events.rb`
- Test: `spec/commands/attendances/check_in_spec.rb`, `spec/commands/attendances/check_in_by_exception_spec.rb`, `spec/commands/attendances/close_spec.rb`

**Interfaces:**
- Consumes: Tasks 1–2; `Citizens::Verify` (subprojeto 2).
- Produces:
  - `Citizens::Verify.record!(citizen:, by:) -> CitizenVerification` — cria a validação, atualiza o nível e publica `citizen.verified` (chamado dentro de uma transação existente); `Citizens::Verify.call` passa a usá-lo.
  - `Attendances::LookupForCheckIn.call(cpf:, code:) -> ok(citizen:, triage:) | fail(:invalid_cpf|:invalid_code|:code_expired|:code_exhausted|:triage_too_old|:already_checked_in|:triage_not_eligible)`; em `:already_checked_in`, `details: {unit_name:, checked_in_at:}`.
  - `Attendances::CheckIn.call(cpf:, code:, health_unit_id:, document_checked:, by:) -> ok(attendance:, verified: Boolean) | fail(… + :invalid_unit)`.
  - `Attendances::EligibleTriages.call(cpf:) -> ok(triages:) | fail(:invalid_cpf)`.
  - `Attendances::CheckInByException.call(cpf:, triage_id:, health_unit_id:, reason:, by:) -> ok(attendance:) | fail(:invalid_cpf|:reason_too_short|:triage_not_eligible|:invalid_unit)`.
  - `Attendances::Close.call(attendance:, outcome:, referral_unit_id:, referral_note:, by:) -> ok(attendance:) | fail(:already_closed|:invalid_outcome|:referral_required|:invalid_unit)`.
  - Eventos `attendance.checked_in` e `attendance.closed` (payload do spec §3).

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/commands/attendances/check_in_spec.rb
require "rails_helper"

RSpec.describe Attendances::CheckIn do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:staff) { User.create!(email_address: "atendente@cidade.gov.br", password: "senha-segura-123") }
  let(:unit) { create_unit }
  let(:triage) { completed_web_triage_for(citizen) }

  def code = Citizens::IssueCheckInCode.call(citizen: citizen, triage: triage).payload.fetch(:code)

  def check_in(c, unit_id: unit.id, checked: false)
    described_class.call(cpf: citizen.cpf, code: c, health_unit_id: unit_id, document_checked: checked, by: staff)
  end

  it "abre o atendimento, consome o código e publica o evento" do
    c = code
    result = check_in(c)
    attendance = result.payload[:attendance]
    expect(attendance).to have_attributes(triage_id: triage.id, health_unit_id: unit.id, check_in_method: "code",
                                          status: "open")
    expect(result.payload[:verified]).to be(false)
    expect(check_in(c).reason).to eq(:code_expired)
    expect(DomainEvent.where(name: "attendance.checked_in").sole.payload.keys)
      .to match_array(%w[attendance_id triage_id citizen_id health_unit_id checked_in_by_user_id check_in_method])
  end

  it "declarado com a caixa: valida e faz check-in juntos" do
    result = check_in(code, checked: true)
    expect(result.payload[:verified]).to be(true)
    expect(citizen.reload).to be_verification_level_verified
    expect(DomainEvent.where(name: "citizen.verified").count).to eq(1)
  end

  it "se o check-in falha, a validação também não fica" do
    Attendance.create!(triage: triage, citizen: citizen, health_unit: unit, checked_in_by_user: staff,
                       checked_in_at: Time.current, check_in_method: "code")
    CitizenVerificationCode.create!(citizen: citizen, purpose: "check_in", triage: triage,
                                    code_digest: CitizenVerificationCode.digest(citizen.id, "123456"),
                                    expires_at: 10.minutes.from_now)
    expect(check_in("123456", checked: true).reason).to eq(:already_checked_in)
    expect(citizen.reload).to be_verification_level_declared
    expect(CitizenVerification.count).to eq(0)
  end

  it "unidade inativa: invalid_unit, nada criado" do
    unit.update!(active: false)
    expect(check_in(code).reason).to eq(:invalid_unit)
    expect(Attendance.count).to eq(0)
  end

  it "triagem antiga: triage_too_old" do
    c = code
    triage.update_columns(completed_at: 4.days.ago)
    expect(check_in(c).reason).to eq(:triage_too_old)
  end
end
```

```ruby
# spec/commands/attendances/check_in_by_exception_spec.rb
require "rails_helper"

RSpec.describe Attendances::CheckInByException do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:stranger) { Citizen.create!(cpf: "11144477735", phone: "+5541911112222") }
  let(:staff) { User.create!(email_address: "atendente@cidade.gov.br", password: "senha-segura-123") }
  let(:unit) { create_unit }

  def call(triage_id:, reason: "cidadão sem celular")
    described_class.call(cpf: "529.982.247-25", triage_id: triage_id, health_unit_id: unit.id, reason: reason, by: staff)
  end

  it "lista só triagens elegíveis do CPF" do
    fresh = completed_web_triage_for(citizen)
    completed_web_triage_for(Citizen.create!(cpf: "52998224725", phone: "+5541933334444"), completed_at: 4.days.ago)
    expect(Attendances::EligibleTriages.call(cpf: citizen.cpf).payload[:triages]).to eq([fresh])
  end

  it "abre com motivo e não valida o cadastro" do
    t = completed_web_triage_for(citizen)
    a = call(triage_id: t.id).payload[:attendance]
    expect(a).to have_attributes(check_in_method: "cpf_exception", exception_reason: "cidadão sem celular")
    expect(citizen.reload).to be_verification_level_declared
  end

  it "motivo curto" do
    t = completed_web_triage_for(citizen)
    expect(call(triage_id: t.id, reason: "curto").reason).to eq(:reason_too_short)
  end

  it "triagem de outro CPF: triage_not_eligible, nada criado" do
    other = completed_web_triage_for(stranger)
    expect(call(triage_id: other.id).reason).to eq(:triage_not_eligible)
    expect(Attendance.count).to eq(0)
  end
end
```

```ruby
# spec/commands/attendances/close_spec.rb
require "rails_helper"

RSpec.describe Attendances::Close do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:staff) { User.create!(email_address: "atendente@cidade.gov.br", password: "senha-segura-123") }
  let(:unit) { create_unit }
  let(:upa) { create_unit("UPA Norte", kind: "upa") }
  let(:attendance) do
    t = completed_web_triage_for(citizen)
    Attendance.create!(triage: t, citizen: citizen, health_unit: unit, checked_in_by_user: staff,
                       checked_in_at: Time.current, check_in_method: "code")
  end

  def close(outcome, unit_id: nil, note: nil)
    described_class.call(attendance: attendance, outcome: outcome, referral_unit_id: unit_id, referral_note: note, by: staff)
  end

  it "liberado e saiu sem atendimento" do
    expect(close("discharged")).to be_ok
    expect(attendance.reload).to have_attributes(status: "closed", outcome: "discharged", closed_by_user_id: staff.id)
    expect(DomainEvent.where(name: "attendance.closed").sole.payload).to include("outcome" => "discharged")
  end

  it "encaminhado exige destino ou descrição, e o destino precisa estar ativo" do
    expect(close("referred").reason).to eq(:referral_required)
    upa.update!(active: false)
    expect(close("referred", unit_id: upa.id).reason).to eq(:invalid_unit)
    expect(close("referred", note: "cardiologia")).to be_ok
  end

  it "encerrar duas vezes, inclusive com registro velho, dá already_closed" do
    stale = Attendance.find(attendance.id)
    close("left")
    expect(described_class.call(attendance: stale, outcome: "discharged", referral_unit_id: nil, referral_note: nil,
                                by: staff).reason).to eq(:already_closed)
  end

  it "desfecho desconhecido" do
    expect(close("cured").reason).to eq(:invalid_outcome)
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/commands/attendances`
Expected: FAIL (`uninitialized constant Attendances::CheckIn`).

- [ ] **Step 3: Implementação**

`app/commands/citizens/verify.rb` — extraia a criação da validação:

```ruby
    # Cria a validação dentro de uma transação que o chamador já abriu (usado
    # também pelo check-in, spec 2026-09-24-citizen-attendance-check-in §2.6).
    def self.record!(citizen:, by:)
      verification = CitizenVerification.create!(citizen: citizen, verified_by_user: by, verified_at: Time.current)
      citizen.update!(verification_level: "verified")
      DomainEvents.publish("citizen.verified", citizen_id: citizen.id, verification_id: verification.id,
                                               verified_by_user_id: by.id)
      verification
    end
```

e, em `call`, troque as três linhas equivalentes por `verification = record!(citizen: citizen, by: by)`.

```ruby
# app/commands/attendances/lookup_for_check_in.rb
module Attendances
  class LookupForCheckIn
    def self.call(cpf:, code:)
      match = Citizens::VerificationCodeMatch.call(cpf: cpf, code: code, purpose: "check_in")
      return match if match.failure?

      triage = match.payload[:triage]
      state = CheckInEligibility.check(triage)
      return failure_for(state, triage) unless state == :ok

      Result.ok(citizen: match.payload[:citizen], triage: triage)
    end

    def self.failure_for(state, triage)
      return Result.fail(state) unless state == :already_checked_in

      a = triage.attendance
      Result.fail(:already_checked_in, details: { unit_name: a.health_unit.name, checked_in_at: a.checked_in_at })
    end
  end
end
```

```ruby
# app/commands/attendances/check_in.rb
# Check-in por código (spec §2.1, §2.6): confere e consome o código sob lock;
# se o par é declarado e o documento foi conferido, valida junto — tudo na
# mesma transação.
module Attendances
  class CheckIn
    def self.call(cpf:, code:, health_unit_id:, document_checked:, by:)
      unit = HealthUnit.active_units.find_by(id: health_unit_id)
      return Result.fail(:invalid_unit) unless unit

      result = nil
      ApplicationRecord.transaction do
        match = Citizens::VerificationCodeMatch.call(cpf: cpf, code: code, lock: true, purpose: "check_in")
        next result = match if match.failure?

        citizen = match.payload[:citizen]
        triage = match.payload[:triage]
        state = CheckInEligibility.check(triage)
        next result = LookupForCheckIn.failure_for(state, triage) unless state == :ok

        match.payload[:verification_code].update!(consumed_at: Time.current)
        verified = false
        if document_checked == true && citizen.active_verification.nil?
          Citizens::Verify.record!(citizen: citizen, by: by)
          verified = true
        end
        attendance = Attendance.create!(triage: triage, citizen: citizen, health_unit: unit, checked_in_by_user: by,
                                        checked_in_at: Time.current, check_in_method: "code")
        publish(attendance)
        result = Result.ok(attendance: attendance, verified: verified)
      end
      result
    rescue ActiveRecord::RecordNotUnique
      Result.fail(:already_checked_in)
    end

    def self.publish(attendance)
      DomainEvents.publish("attendance.checked_in",
                           attendance_id: attendance.id, triage_id: attendance.triage_id,
                           citizen_id: attendance.citizen_id, health_unit_id: attendance.health_unit_id,
                           checked_in_by_user_id: attendance.checked_in_by_user_id,
                           check_in_method: attendance.check_in_method)
    end
  end
end
```

Atenção ao teste "se o check-in falha, a validação também não fica": a checagem de elegibilidade vem **antes** de `Citizens::Verify.record!`, e o `next` fecha o bloco sem gravar a validação. Confira que o incremento de tentativas, se houver, continua sendo gravado (mesma semântica de `Citizens::Verify`).

```ruby
# app/commands/attendances/eligible_triages.rb
# Exceção sem código (spec §2.1): triagens elegíveis de um CPF.
module Attendances
  class EligibleTriages
    def self.call(cpf:)
      digits = CitizenIdentity::Cpf.normalize(cpf)
      return Result.fail(:invalid_cpf) unless digits

      Result.ok(triages: CheckInEligibility.eligible_for(Citizen.where(cpf: digits)).to_a)
    end
  end
end
```

```ruby
# app/commands/attendances/check_in_by_exception.rb
# Check-in sem código (spec §2.1): motivo obrigatório; não valida o cadastro.
module Attendances
  class CheckInByException
    MIN_REASON = 10

    def self.call(cpf:, triage_id:, health_unit_id:, reason:, by:)
      digits = CitizenIdentity::Cpf.normalize(cpf)
      return Result.fail(:invalid_cpf) unless digits
      return Result.fail(:reason_too_short) if reason.to_s.strip.length < MIN_REASON

      unit = HealthUnit.active_units.find_by(id: health_unit_id)
      return Result.fail(:invalid_unit) unless unit

      triage = CheckInEligibility.eligible_for(Citizen.where(cpf: digits)).find_by(id: triage_id)
      return Result.fail(:triage_not_eligible) unless triage

      attendance = nil
      ApplicationRecord.transaction do
        attendance = Attendance.create!(triage: triage, citizen: triage.conversation.citizen, health_unit: unit,
                                        checked_in_by_user: by, checked_in_at: Time.current,
                                        check_in_method: "cpf_exception", exception_reason: reason.to_s.strip)
        CheckIn.publish(attendance)
      end
      Result.ok(attendance: attendance)
    rescue ActiveRecord::RecordNotUnique
      Result.fail(:triage_not_eligible)
    end
  end
end
```

```ruby
# app/commands/attendances/close.rb
# Desfecho do atendimento (spec §2.4): encerra uma vez, sob lock.
module Attendances
  class Close
    def self.call(attendance:, outcome:, referral_unit_id:, referral_note:, by:)
      return Result.fail(:invalid_outcome) unless Attendance::OUTCOMES.include?(outcome.to_s)

      note = referral_note.to_s.strip.presence
      unit = nil
      if outcome.to_s == "referred"
        if referral_unit_id.present?
          unit = HealthUnit.active_units.find_by(id: referral_unit_id)
          return Result.fail(:invalid_unit) unless unit
        end
        return Result.fail(:referral_required) if unit.nil? && note.nil?
      end

      outcome_result = ApplicationRecord.transaction do
        attendance.lock!
        next :already_closed unless attendance.open?

        attendance.update!(status: "closed", outcome: outcome.to_s, closed_by_user: by, closed_at: Time.current,
                           referral_unit: (outcome.to_s == "referred" ? unit : nil),
                           referral_note: (outcome.to_s == "referred" ? note : nil))
        DomainEvents.publish("attendance.closed", attendance_id: attendance.id, outcome: attendance.outcome,
                                                  closed_by_user_id: by.id)
        :ok
      end
      outcome_result == :ok ? Result.ok(attendance: attendance) : Result.fail(outcome_result)
    end
  end
end
```

Em `config/initializers/domain_events.rb`: `DomainEvents.bind "attendance.checked_in", to: []` e `DomainEvents.bind "attendance.closed", to: []`, com um comentário citando o ADR 0018.

- [ ] **Step 4: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/commands/attendances spec/commands/citizens spec/events`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/commands/citizens/verify.rb app/commands/attendances config/initializers/domain_events.rb spec/commands/attendances
/opt/homebrew/bin/git commit -m "feat: check citizens into health units and record the outcome" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Rotas de servidor (`/attendance/*`)

**Repo:** `apps/api`.

**Files:**
- Create: `app/controllers/concerns/attendance_access.rb`, `app/controllers/health_units_controller.rb`, `app/controllers/check_ins_controller.rb`, `app/controllers/attendances_controller.rb`
- Modify: `app/controllers/attendance_controller.rb` (passa a usar o concern), `config/routes.rb`, `config/initializers/filter_parameter_logging.rb`
- Test: `spec/requests/health_units_spec.rb`, `spec/requests/check_ins_spec.rb`, `spec/requests/attendances_spec.rb`

**Interfaces:**
- Consumes: Tasks 1–3.
- Produces: `AttendanceAccess` (concern: `require_verifier`, `require_admin`, `forbid`, `RateLimitStore`, `render_failure(result, status_map)`); as rotas e formatos do spec §4, com os erros do spec §6. Formatos:
  - `GET /attendance/units` → `{units: [{id, name, kind}]}`; `GET /attendance/units/all` → `{units: [{id, name, kind, active}]}`; `POST /attendance/units` → 201 `{unit: {...}}`; `POST /attendance/units/:id` → 200; `POST /attendance/units/:id/deactivate|activate` → 200.
  - `POST /attendance/check_ins/lookup` → `{citizen: {id, cpf_masked, phone_masked, verification_level}, triage: {id, date, protocol_name, priority}}`.
  - `POST /attendance/check_ins` → 201 `{attendance: attendance_json, verified}`.
  - `POST /attendance/check_ins/search` → `{triages: [{id, date, protocol_name, priority}]}`.
  - `POST /attendance/check_ins/exception` → 201 `{attendance: attendance_json}`.
  - `GET /attendance/units/:id/open` → `{attendances: [{id, cpf_masked, checked_in_at, protocol_name, priority}]}` ordenado por `priority ASC NULLS LAST, checked_in_at ASC`.
  - `POST /attendance/attendances/:id/close` → 200 `{attendance: attendance_json}`.
  - `attendance_json = {id, triage_id, health_unit_id, unit_name, status, checked_in_at, check_in_method, outcome, referral_unit_name, referral_note, closed_at}`.
  - Erros: `invalid_cpf`, `invalid_code`, `code_expired`, `code_exhausted`, `triage_too_old`, `triage_not_eligible`, `invalid_unit`, `reason_too_short`, `referral_required`, `invalid_outcome`, `unit_name_taken`, `invalid_kind` → 422; `already_checked_in` (com `unit_name`, `checked_in_at`), `already_closed` → 409; sem papel → 403 `forbidden`; atendimento inexistente → 404.

- [ ] **Step 1: Write the failing tests** — `spec/requests/check_ins_spec.rb` (atendente: lookup com código de check-in → 200 sem celular em claro; lookup com código de validação → 422 `invalid_code`; check-in → 201 e depois 409 `already_checked_in` com `unit_name`; check-in com `document_checked: true` em declarado → `verified: true`; exceção: `search` só com elegíveis, `exception` com motivo → 201 e `check_in_method: "cpf_exception"`; viewer → 403; escrita sem JSON → 415), `spec/requests/attendances_spec.rb` (lista de abertos da unidade ordenada por prioridade e chegada; encerrar com cada desfecho; `referral_required`; `already_closed`; atendimento inexistente → 404; encerrado some da lista) e `spec/requests/health_units_spec.rb` (admin cria, edita, desativa e reativa; nome repetido em outra caixa → 422 `unit_name_taken`; tipo desconhecido → 422 `invalid_kind`; atendente lista só ativas em `GET /attendance/units` e recebe 403 nas escritas e em `/units/all`). Use `sign_in_as(user_with(email, role))` e `json_post`, no mesmo estilo de `spec/requests/attendance_spec.rb`; os dados vêm dos helpers `create_unit`, `completed_web_triage_for`, `Citizens::IssueCheckInCode`. Escreva um `it` por comportamento listado aqui.

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/requests/health_units_spec.rb spec/requests/check_ins_spec.rb spec/requests/attendances_spec.rb`
Expected: FAIL (404).

- [ ] **Step 3: Implementação**

```ruby
# app/controllers/concerns/attendance_access.rb
# Acesso às rotas do balcão (/attendance/*): papel da cidade, limite por
# servidor e tradução de Result para JSON. Usado pelo balcão de validação
# (subprojeto 2) e pelo check-in (subprojeto 3).
module AttendanceAccess
  extend ActiveSupport::Concern

  module RateLimitStore
    def self.increment(...) = Rails.cache.increment(...)
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

  def render_failure(result, status_map)
    payload = { error: result.reason.to_s }.merge(result.details.transform_values { |v| v.respond_to?(:iso8601) ? v.iso8601 : v })
    render json: payload, status: status_map.fetch(result.reason, :unprocessable_entity)
  end
end
```

`AttendanceController`: `include AttendanceAccess`; remova os métodos que o concern provê; troque `render_failure(result)` por `render_failure(result, ERROR_STATUS)` e o `store: RateLimitStore` por `store: AttendanceAccess::RateLimitStore`. As specs do subprojeto 2 (`spec/requests/attendance_spec.rb`) precisam seguir verdes sem mudança.

`CheckInsController` (`include Authentication, AttendanceAccess`; `before_action :require_verifier`; `rate_limit to: 30, within: 10.minutes, name: "attendance_check_in", by: -> { Current.user&.id || request.remote_ip }, store: AttendanceAccess::RateLimitStore`): ações `lookup`, `create` (`document_checked: params[:document_checked] == true`), `search`, `exception`, cada uma chamando o comando da Task 3 e serializando como nas Interfaces. `triage_json(t) = {id: t.id, date: (t.completed_at || t.created_at).iso8601, protocol_name: t.protocol_name, priority: t.priority}`.

`AttendancesController` (`before_action :require_verifier`): `open` (`health_unit = HealthUnit.find_by(id: params[:id])` → 404 se não existe; `Attendance.open_attendances.where(health_unit:).includes(:triage, :citizen).sort_by { |a| [a.triage.priority || 999, a.checked_in_at] }`) e `close` (`Attendance.find_by(id:)` → 404; `Attendances::Close.call(...)`).

`HealthUnitsController`: `index` (verificador ou admin: `require_verifier_or_admin` — escreva no controller `forbid unless policy.verify? || policy.manage?`) devolvendo `active_units`; `all`, `create`, `update`, `deactivate`, `activate` só admin. Erros de validação: nome repetido → `unit_name_taken`, tipo → `invalid_kind` (use `record.errors.details`; `ActiveRecord::RecordNotUnique` também vira `unit_name_taken`).

`config/routes.rb`, dentro do `scope "/attendance"` existente:

```ruby
    get  "units",                   to: "health_units#index"
    get  "units/all",               to: "health_units#all"
    post "units",                   to: "health_units#create"
    post "units/:id",               to: "health_units#update"
    post "units/:id/deactivate",    to: "health_units#deactivate"
    post "units/:id/activate",      to: "health_units#activate"
    get  "units/:id/open",          to: "attendances#open"
    post "check_ins/lookup",        to: "check_ins#lookup"
    post "check_ins",               to: "check_ins#create"
    post "check_ins/search",        to: "check_ins#search"
    post "check_ins/exception",     to: "check_ins#exception"
    post "attendances/:id/close",   to: "attendances#close"
```

(Declare `units/all` antes de `units/:id`.) `filter_parameter_logging.rb`: acrescente `:referral_note`.

- [ ] **Step 4: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/requests/health_units_spec.rb spec/requests/check_ins_spec.rb spec/requests/attendances_spec.rb spec/requests/attendance_spec.rb spec/architecture spec/requests/cookie_write_requires_json_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/concerns/attendance_access.rb app/controllers/health_units_controller.rb app/controllers/check_ins_controller.rb app/controllers/attendances_controller.rb app/controllers/attendance_controller.rb config/routes.rb config/initializers/filter_parameter_logging.rb spec/requests/health_units_spec.rb spec/requests/check_ins_spec.rb spec/requests/attendances_spec.rb
/opt/homebrew/bin/git commit -m "feat: expose unit check-in, open attendances and unit registry to staff" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Lado do cidadão na API

**Repo:** `apps/api`.

**Files:**
- Create: `app/controllers/citizen_api/check_in_codes_controller.rb`
- Modify: `app/controllers/citizen_api/triages_controller.rb`, `config/routes.rb`
- Test: `spec/requests/citizen_api/check_in_codes_spec.rb`, `spec/requests/citizen_api/attendance_in_history_spec.rb`

**Interfaces:**
- Consumes: `Citizens::IssueCheckInCode`, `Attendances::CheckInEligibility`, `Attendance` (Tasks 1–3).
- Produces: `POST /citizen/triages/:id/check_in_code` → 201 `{code, expires_at}` | 404 (triagem não visível ou não é de um par do celular) | 422 `triage_too_old`/`triage_not_eligible` | 409 `already_checked_in`; `GET /citizen/triages` e `/citizen/triages/:id`: cada item ganha `attendance` (`null` ou `{status, unit_name, checked_in_at, outcome, referral_unit_name, referral_note, closed_at}`) e `check_in_available` (`true` só para triagem do próprio par e elegível).

- [ ] **Step 1: Write the failing tests** — `check_in_codes_spec.rb`: gera o código para triagem própria elegível; triagem de outro celular → 404; triagem de 4 dias → 422 `triage_too_old`; triagem com atendimento → 409; limite 10 por hora pela sessão (mesmo `rate_limit` do `VerificationCodesController`). `attendance_in_history_spec.rb`: sem atendimento → `attendance: null` e `check_in_available: true`; depois do check-in → `attendance.status == "open"` e `unit_name`; depois de encerrar como encaminhado → `outcome: "referred"`, `referral_unit_name`/`referral_note`; triagem de outro par no histórico completo de um verificado → `check_in_available: false`; triagem antiga → `check_in_available: false`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/requests/citizen_api/check_in_codes_spec.rb spec/requests/citizen_api/attendance_in_history_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implementação**

```ruby
# app/controllers/citizen_api/check_in_codes_controller.rb
# POST /citizen/triages/:id/check_in_code — "Cheguei na unidade" (spec
# 2026-09-24-citizen-attendance-check-in §4). Só para triagem de um par do
# celular da sessão.
module CitizenApi
  class CheckInCodesController < BaseController
    ERROR_STATUS = { triage_too_old: :unprocessable_entity, triage_not_eligible: :unprocessable_entity,
                     already_checked_in: :conflict }.freeze

    rate_limit to: 10, within: 1.hour, only: :create, name: "citizen_check_in_code",
               by: -> { current_citizen_session&.id || request.remote_ip }, store: RateLimitStore,
               with: -> { render_error("too_many_requests", :too_many_requests) }

    def create
      triage = Triage.joins(:conversation)
                     .where(conversations: { channel: "web", citizen_id: current_citizen_session.citizens.select(:id) })
                     .find_by(id: params[:id])
      return render_error("not_found", :not_found) unless triage

      result = Citizens::IssueCheckInCode.call(citizen: triage.conversation.citizen, triage: triage)
      return render_error(result.reason, ERROR_STATUS.fetch(result.reason, :unprocessable_entity)) if result.failure?

      render json: { code: result.payload[:code], expires_at: result.payload[:expires_at].iso8601 }, status: :created
    end
  end
end
```

Rota, dentro do `scope "/citizen"`: `post "triages/:id/check_in_code", to: "check_in_codes#create"`.

Em `CitizenApi::TriagesController#summary`, acrescente:

```ruby
        attendance: attendance_json(triage.attendance),
        check_in_available: own && Attendances::CheckInEligibility.check(triage) == :ok
```

e o método privado:

```ruby
    def attendance_json(attendance)
      return nil unless attendance

      {
        status: attendance.status, unit_name: attendance.health_unit.name,
        checked_in_at: attendance.checked_in_at.iso8601, outcome: attendance.outcome,
        referral_unit_name: attendance.referral_unit&.name, referral_note: attendance.referral_note,
        closed_at: attendance.closed_at&.iso8601
      }
    end
```

Acrescente `includes(:attendance)` onde o escopo de triagens é montado (`history_scope`, `visible_triages`) para não gerar uma consulta por linha.

- [ ] **Step 4: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/requests/citizen_api`
Expected: PASS.

- [ ] **Step 5: Suíte completa** (worker parado antes, religado depois). Expected: 0 falhas, ~3 min.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/citizen_api/check_in_codes_controller.rb app/controllers/citizen_api/triages_controller.rb config/routes.rb spec/requests/citizen_api/check_in_codes_spec.rb spec/requests/citizen_api/attendance_in_history_spec.rb
/opt/homebrew/bin/git commit -m "feat: let citizens request a check-in code and follow their attendance" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Dashboard — unidade atual e cadastro de unidades

**Repo:** `apps/dashboard`. Antes: confira `main` limpo e crie `feat/citizen-attendance-check-in`. Comandos: `npm test`, `npm run typecheck`, `npm run build`.

**Files:**
- Modify: `src/lib/api.ts`, `src/lib/attendance.ts`, `src/modules/Attendance.tsx`
- Create: `src/modules/attendance/UnitPicker.tsx`, `src/modules/attendance/Units.tsx` (+ `UnitPicker.test.tsx`, `Units.test.tsx`)

**Interfaces:**
- Consumes: rotas de unidades da Task 4.
- Produces:
  - `src/lib/api.ts`: `HealthUnit {id, name, kind}`, `HealthUnitRow extends HealthUnit {active}`; `listActiveUnits()`, `listAllUnits()`, `createUnit(name, kind)`, `updateUnit(id, name, kind)`, `setUnitActive(id, active)`.
  - `src/lib/attendance.ts`: `UNIT_KINDS = [{value: "ubs", label: "UBS"}, {value: "upa", label: "UPA"}, {value: "hospital", label: "Hospital"}, {value: "other", label: "Outra"}]`; `currentUnitKey(userId) -> string` (chave do `localStorage`: `"attendance.unit." + userId`); `attendanceError` ganha `triage_too_old`, `triage_not_eligible`, `already_checked_in` (com unidade e hora), `invalid_unit`, `referral_required`, `already_closed`, `unit_name_taken`, `invalid_kind`, `invalid_outcome`.
  - `UnitPicker({ userId, onChange(unit: HealthUnit | null) })`: mostra "Unidade: *nome* [trocar]" ou a lista de ativas para escolher; lê e grava o `localStorage` (em `try/catch`); se a unidade guardada não está mais entre as ativas, limpa e pede escolha.
  - `Units()`: só para `municipal_admin` (lista, "Nova unidade", "Editar", "Desativar"/"Reativar").

- [ ] **Step 1: Write the failing tests** — `UnitPicker.test.tsx`: sem unidade guardada mostra a lista e grava a escolha; com unidade guardada ativa mostra "Unidade: UBS Centro"; com unidade guardada que não está mais ativa pede escolha de novo; "trocar" volta à lista. `Units.test.tsx`: cria (nome + tipo), edita, desativa, reativa; mostra a frase de `unit_name_taken`. Mesmo padrão de mocks de `src/modules/Attendance.test.tsx` (sem jest-dom).

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- src/modules/attendance`
Expected: FAIL.

- [ ] **Step 3: Implementação** conforme as Interfaces. Em `Attendance.tsx`, `Units` aparece como seção para `municipal_admin`; `UnitPicker` aparece no topo para `citizen_verifier` e guarda a unidade num estado que a Task 7 consome.

- [ ] **Step 4: Run tests**

Run: `npm test && npm run typecheck && npm run build`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/api.ts src/lib/attendance.ts src/modules/Attendance.tsx src/modules/attendance/UnitPicker.tsx src/modules/attendance/UnitPicker.test.tsx src/modules/attendance/Units.tsx src/modules/attendance/Units.test.tsx
/opt/homebrew/bin/git commit -m "feat: choose the current unit and manage health units" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Dashboard — check-in, exceção, abertos e desfecho

**Repo:** `apps/dashboard`.

**Files:**
- Modify: `src/lib/api.ts`, `src/modules/Attendance.tsx`
- Create: `src/modules/attendance/CheckIn.tsx`, `src/modules/attendance/OpenAttendances.tsx` (+ testes)

**Interfaces:**
- Consumes: rotas de check-in e atendimentos da Task 4; `UnitPicker` e `attendanceError` (Task 6).
- Produces: `api.ts`: `lookupCheckIn(cpf, code)`, `checkIn(cpf, code, unitId, documentChecked)`, `searchCheckInTriages(cpf)`, `checkInByException(cpf, triageId, unitId, reason)`, `listOpenAttendances(unitId)`, `closeAttendance(id, outcome, referralUnitId?, referralNote?)` e os tipos correspondentes; `CheckIn({ unit })`, `OpenAttendances({ unit, units })`.

- [ ] **Step 1: Write the failing tests** — `CheckIn.test.tsx`: busca exige CPF válido e 6 dígitos; cartão mostra CPF e celular mascarados, data, protocolo e prioridade; para `declared`, mostra a caixa "Conferi o documento com foto e o CPF confere" e manda `documentChecked` conforme a caixa; confirmação "Atendimento iniciado" (e "cadastro validado" quando `verified: true`) e "Próximo atendimento" limpa; `already_checked_in` mostra unidade e hora; `invalid_unit` chama o `onUnitInvalid` para voltar à escolha; exceção: "Cidadão sem o código" → busca por CPF → escolher triagem → motivo com ≥ 10 caracteres habilita "Iniciar atendimento". `OpenAttendances.test.tsx`: lista ordenada como veio da API; "Encerrar" com cada desfecho; "Encaminhado" sem destino e sem descrição fica desabilitado; `already_closed` recarrega a lista.

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- src/modules/attendance`
Expected: FAIL.

- [ ] **Step 3: Implementação** conforme as Interfaces e o spec §5. Em `Attendance.tsx`, com unidade escolhida, o atendente vê `CheckIn`, `OpenAttendances` e o balcão de validação que já existe; sem unidade, só a escolha. A lista de abertos recarrega depois de um check-in e de um encerramento (TanStack Query `invalidateQueries`).

- [ ] **Step 4: Run tests**

Run: `npm test && npm run typecheck && npm run build`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/api.ts src/modules/Attendance.tsx src/modules/attendance/CheckIn.tsx src/modules/attendance/CheckIn.test.tsx src/modules/attendance/OpenAttendances.tsx src/modules/attendance/OpenAttendances.test.tsx
/opt/homebrew/bin/git commit -m "feat: check citizens in at the unit and close attendances" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: wpda — "Cheguei na unidade" e situação do atendimento

**Repo:** `apps/wpda`. Antes: confira `main` limpo e crie `feat/citizen-attendance-check-in`. Comandos: `npm test && npm run typecheck && npm run build`.

**Files:**
- Create: `src/modules/citizen/CounterCodeStep.tsx` (+ test)
- Modify: `src/modules/citizen/VerificationCodeStep.tsx`, `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `src/lib/citizenApi.ts`, `src/modules/citizen/triage.test.tsx`

**Interfaces:**
- Consumes: Task 5.
- Produces: `citizenApi.issueCheckInCode(triageId) -> Promise<{code, expires_at}>`; `TriageSummary` ganha `attendance: AttendanceSummary | null` e `check_in_available: boolean`; `CounterCodeStep({ title, instruction, issue: () => Promise<{code, expires_at}>, onBack })` (a lógica que hoje está em `VerificationCodeStep`: contagem, "Gerar outro código", só a resposta mais recente, botão desabilitado em voo); `VerificationCodeStep` vira um invólucro de `CounterCodeStep`; `HistoryStep` ganha `onCheckIn(triageId)`; `Flow` ganha o estado `{ at: "check-in-code"; triageId; citizenId; consentVersion }`.

- [ ] **Step 1: Write the failing tests** — `CounterCodeStep.test.tsx`: os testes que hoje estão em `VerificationCodeStep.test.tsx` (código, instrução, contagem, gerar outro, código vencido, só a resposta mais recente), agora parametrizados por `issue` e `instruction`. Em `triage.test.tsx`: triagem com `check_in_available: true` mostra "Cheguei na unidade" e chama `onCheckIn(id)`; `attendance.status == "open"` mostra "Em atendimento na UBS Centro desde hh:mm"; encerrado mostra "Atendido e liberado" / "Encaminhado para UPA Norte — cardiologia" / "Saiu sem atendimento"; sem `check_in_available` nem atendimento, sem botão.

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- src/modules/citizen`
Expected: FAIL.

- [ ] **Step 3: Implementação** conforme as Interfaces e o spec §5. Instrução do check-in: "Mostre este código e um documento com foto na recepção da unidade." Título: "Cheguei na unidade". Todo texto interativo ≥ 18 px.

- [ ] **Step 4: Run tests**

Run: `npm test && npm run typecheck && npm run build`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/citizenApi.ts src/modules/citizen/CounterCodeStep.tsx src/modules/citizen/CounterCodeStep.test.tsx src/modules/citizen/VerificationCodeStep.tsx src/modules/citizen/VerificationCodeStep.test.tsx src/modules/citizen/HistoryStep.tsx src/modules/citizen/Flow.tsx src/modules/citizen/triage.test.tsx
/opt/homebrew/bin/git commit -m "feat: let citizens check in at a unit and follow the attendance" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Se `VerificationCodeStep.test.tsx` for removido por ter virado `CounterCodeStep.test.tsx`, use `git rm` nele.)

---

### Task 9: Verificação de ponta a ponta em desenvolvimento

Sem código novo, a menos que ache defeito (corrija na task dona, com teste).

- [ ] **Step 1:** `docker compose exec -T api bin/rails city:migrate:all`; `docker compose restart api worker wpda dashboard`; confira que `/attendance/units` (proxy do dashboard) e `/citizen/triages` (proxy do wpda) chegam à api.
- [ ] **Step 2:** Via `rails runner` em Curitiba: criar "UBS Centro" (ubs) e "UPA Norte" (upa).
- [ ] **Step 3:** No wpda, com um cidadão declarado e uma triagem de hoje: "Minhas triagens" → "Cheguei na unidade" → anotar o código.
- [ ] **Step 4:** Via `rails runner`, com um usuário `citizen_verifier` de dev: `Attendances::LookupForCheckIn` e `Attendances::CheckIn` com `document_checked: true` → conferir `verified: true`, atendimento aberto, eventos `citizen.verified` e `attendance.checked_in`.
- [ ] **Step 5:** No wpda: "Em atendimento na UBS Centro desde …" e o selo de verificado.
- [ ] **Step 6:** Via `rails runner`: `Attendances::Close` como `referred` para a UPA Norte com descrição → no wpda, "Encaminhado para UPA Norte — …".
- [ ] **Step 7:** Repetir o check-in pela exceção (`EligibleTriages` + `CheckInByException`) com outra triagem e conferir que o cadastro não é validado por esse caminho.
- [ ] **Step 8:** Relatório (sem códigos em claro). As telas do dashboard ficam para o usuário conferir no navegador (login de servidor exige senha).
