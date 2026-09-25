# Chamada, retorno e agendamento (subprojeto 4) — plano

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** o profissional de saúde chama o cidadão da fila (`waiting → in_care`) e registra o desfecho. Retorno e encaminhamento com unidade viram um pedido de agendamento, que a recepção da unidade de destino marca à mão. O cidadão vê, confirma (até 24h antes) ou cancela no wpda, e no dia faz check-in do horário pelo mesmo balcão.

**Architecture:**
- **Dados:** uma migração de cidade cria `appointment_requests` e `appointments`, ajusta `attendances` (estados, chamada, origem triagem-ou-horário, desfecho `return`) e o código do balcão (`appointment_id`), e acrescenta o papel `health_professional`. Triggers em `db/city_triggers.sql` só aceitam acréscimo e as transições previstas.
- **Comandos:** em `Attendances::` (chamar, encerrar, check-in), `Appointments::` (marcar, confirmar, cancelar, expirar) e `AppointmentRequests::` (ciclo do pedido).
- **Jobs:** dois jobs por cidade a cada 15 minutos.
- **Telas:** o dashboard mostra painéis por papel, e o wpda ganha "Seus agendamentos".

**Tech Stack:** Rails 8.1, PostgreSQL e Solid Queue (apps/api); Vite, React 18, TanStack Query e Vitest (apps/dashboard sem jest-dom; apps/wpda com jest-dom e user-event).

**Spec:** `docs/superpowers/specs/2026-09-25-citizen-appointments-design.md` (commit `bcb0a2a`). ADR: `docs/adr/0019.md`.

## Global Constraints

- **Commits:** Conventional Commits **em inglês**, tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Git:**
  - use `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta);
  - branch `feat/citizen-appointments` em api, dashboard e wpda, a partir de `main`;
  - nunca trabalhe em `main` e nunca faça push;
  - faça o staging dos arquivos explicitamente.
- **Antes de criar cada branch:** confira que `git branch --show-current` dá `main` e que `git status --short` sai vazio. Outra sessão ("Página /maintenance em development") usa api e dashboard. Se o repositório não estiver em `main` limpo, pare e avise.
- **api:**
  - rode os comandos no container, a partir da raiz do monorepo: `docker compose exec -T api bundle exec rspec <arquivos>`;
  - a suíte completa roda em primeiro plano, com `docker compose stop worker` antes e `docker compose start worker` depois, sempre. Se passar de ~3 min, é regressão.
- **Specs da api:**
  - request specs levam `type: :request`;
  - `sign_in_as`, `sign_in_citizen`, `json_post`, `issue_code_for`, `create_unit` e `completed_web_triage_for` já existem;
  - não há factory `:user`: crie com `User.create!(email_address:, password: "senha-segura-123")` e depois `Membership.create!(user:, role:, granted_at: Time.current)`;
  - `travel_to` exige `include ActiveSupport::Testing::TimeHelpers`;
  - arquivo novo em `spec/support` precisa de `require_relative` em `spec/rails_helper.rb`.
- **Migração de cidade:**
  - arquivo `db/city_migrate/20260926000001_create_appointments.rb`. Confira `ls db/city_migrate | tail -1`: a última hoje é `20260925000001`. Se a outra sessão criou uma mais nova, use a versão seguinte a ela.
  - O dump é feito à mão em `db/city_schema.rb`, e o juiz é `spec/services/city_schema_spec.rb`.
  - Triggers novos vão em `db/city_triggers.sql`, **protegidos por `to_regclass(...) IS NOT NULL`**, porque migrações antigas re-executam o arquivo.
  - Depois da migração: `docker compose exec -T api bin/rails city:migrate:all` (dev) e `docker compose exec -T api bin/rails city:test_databases` (bancos de teste).
- **Eventos:** todo evento novo precisa de `DomainEvents.bind "<nome>", to: []` em `config/initializers/domain_events.rb`. O payload leva só ids e enums, nunca CPF, celular, motivo, justificativa ou nota.
- **Valores da spec:**

| Regra | Valor |
|---|---|
| Prazo de confirmação | **24h** antes do horário (`Appointment::CONFIRMATION_LEAD = 24.hours`) |
| Nasce confirmado | faltam **menos de 48h** (`Appointment::BORN_CONFIRMED_WITHIN = 48.hours`) |
| Horário mais distante | **180 dias** (`Appointment::MAX_AHEAD = 180.days`) |
| Motivo de cancelamento e justificativa | **≥ 10 caracteres** depois de `strip` |
| Jobs | a cada **15 minutos** |
| Fuso | `Time.zone` (`America/Sao_Paulo`) |

- **Estados e enums:**

| Coluna | Valores |
|---|---|
| `attendances.status` | `waiting` \| `in_care` \| `closed` |
| `attendances.outcome` | `discharged` \| `referred` \| `return` \| `left` |
| `appointment_requests.kind` | `return` \| `referral` |
| `appointment_requests.status` | `open` \| `scheduled` \| `closed` |
| `appointment_requests.reopened_reason` | `expired` \| `no_show` |
| `appointment_requests.closed_reason` | `fulfilled` \| `citizen_cancelled` \| `dismissed` |
| `appointments.status` | `scheduled` \| `confirmed` \| `checked_in` \| `cancelled_by_citizen` \| `expired` \| `no_show` |

- **Papéis:**
  - `health_professional` chama e registra `discharged`, `referred` e `return`;
  - `citizen_verifier` faz check-in, validação, pedidos e agenda;
  - os dois podem registrar `left` (só a partir de `waiting`) e ver a fila;
  - `health_professional` entra em `Membership::PRIVILEGED_ROLES`.
- **Proibições:** o CPF nunca vai na URL. Nunca imprima código de balcão, OTP ou TOTP em relatório ou commit.
- **dashboard:** testes **sem jest-dom** e sem dependência nova. Use asserções simples: `getBy*`, `(el as HTMLButtonElement).disabled` e `queryBy*(...)).toBeNull()`.
- **wpda:** texto interativo com pelo menos 18 px e alvos com pelo menos 48 px. Os campos novos da API são opcionais na entrada e normalizados num só lugar (`citizenApi.ts`).

## Review Focus

1. **`NOT IN` com NULL:** `CheckInEligibility.eligible_for` usa `where.not(id: Attendance.select(:triage_id))`. Assim que existir um atendimento vindo de horário (`triage_id` nulo), o `NOT IN` com NULL passa a excluir **todas** as triagens, sem erro. O esperado é que triagens sem atendimento continuem elegíveis. Teste na Task 1.
2. **Bordas do relógio:**
   - marcar com exatamente 48h de antecedência dá `scheduled` com prazo, e com 48h menos 1 segundo dá `confirmed`;
   - confirmar 1 segundo antes do prazo funciona, e no prazo exato dá `confirmation_closed`;
   - o job expira no prazo exato, e não 1 segundo antes.
   Testes nas Tasks 3, 5 e 6.
3. **Virada do dia no fuso da cidade:** um horário às 23h30 locais (02h30 UTC do dia seguinte) é "hoje" para o check-in e para o "Cheguei". Ele só vira `no_show` depois da meia-noite local, e não à meia-noite UTC. Testes nas Tasks 4 e 6.
4. **Corridas:**
   - o job de expiração contra a confirmação do cidadão no mesmo instante: quem trava primeiro vence, e o outro não faz nada nem quebra;
   - dois profissionais chamando o mesmo atendimento: um vence, e o outro recebe `already_called`.
   Testes nas Tasks 2 e 6.
5. **Unidade desativada:**
   - marcar horário num pedido cuja unidade de destino foi desativada dá `invalid_unit`;
   - desativar uma unidade com pedidos `open` ou `scheduled` dá 409 `unit_has_open_requests` (mesmo raciocínio do 409 de atendimentos abertos: senão o pedido fica sem ninguém para tratar).
   Teste na Task 3.

---

## File Structure

**apps/api**
- **Migração:** criar `db/city_migrate/20260926000001_create_appointments.rb`. Modificar `db/city_schema.rb` e `db/city_triggers.sql`.
- **Modelos:**
  - criar `app/models/appointment_request.rb` e `app/models/appointment.rb`;
  - modificar `app/models/attendance.rb`, `app/models/citizen_verification_code.rb` e `app/models/membership.rb`.
- **Comandos:**
  - criar `app/commands/attendances/unit_queue.rb`, `call.rb` e `call_next.rb`;
  - modificar `app/commands/attendances/close.rb`, `check_in_eligibility.rb`, `lookup_for_check_in.rb`, `check_in.rb`, `eligible_triages.rb` e `check_in_by_exception.rb`;
  - criar `app/commands/attendances/appointment_check_in_eligibility.rb`;
  - criar `app/commands/appointment_requests/lifecycle.rb` e `dismiss.rb`;
  - criar `app/commands/appointments/schedule.rb`, `confirm.rb`, `cancel_by_citizen.rb` e `lapse.rb`;
  - criar `app/commands/citizens/issue_appointment_check_in_code.rb`;
  - modificar `app/commands/citizens/issue_counter_code.rb` e `verification_code_match.rb`.
- **Jobs:** criar `app/jobs/expire_unconfirmed_appointments_job.rb` e `app/jobs/mark_no_show_appointments_job.rb`. Modificar `config/recurring.yml`.
- **Controllers:**
  - criar `app/controllers/appointment_requests_controller.rb` e `app/controllers/citizen_api/appointments_controller.rb`;
  - modificar `app/controllers/attendances_controller.rb`, `check_ins_controller.rb`, `health_units_controller.rb`, `concerns/attendance_access.rb` e `citizen_api/triages_controller.rb`;
  - modificar `app/policies/citizen_verification_policy.rb`.
- **Config:** modificar `config/routes.rb`, `config/initializers/domain_events.rb` e `config/initializers/filter_parameter_logging.rb`.
- **Outros:** modificar `lib/signature_crew.rb` (semente) e `spec/adr_pointers_spec.rb`.
- **Specs:**
  - em `spec/models/`, `spec/commands/attendances/`, `spec/commands/appointments/`, `spec/requests/`, `spec/requests/citizen_api/` e `spec/jobs/`;
  - helper novo `spec/support/appointment_helpers.rb`;
  - specs existentes a ajustar: `spec/requests/attendances_spec.rb`, `spec/commands/attendances/close_spec.rb`, `spec/models/attendance_spec.rb`, `spec/requests/attendance_contract_spec.rb`, `spec/requests/health_units_spec.rb` e `spec/requests/citizen_api/attendance_in_history_spec.rb`.

**apps/dashboard**
- **Modificar:** `src/lib/api.ts`, `src/lib/attendance.ts`, `src/lib/team.ts`, `src/modules/Team.tsx`, `src/shell/modules.ts`, `src/modules/Attendance.tsx` e `src/modules/attendance/CheckIn.tsx`.
- **Criar:** `src/modules/attendance/UnitQueue.tsx` (substitui `OpenAttendances.tsx`, que é removido), `src/modules/attendance/Requests.tsx` e `src/modules/attendance/Agenda.tsx`, cada um com testes.

**apps/wpda**
- **Criar:** `src/modules/citizen/AppointmentsSection.tsx`, com testes.
- **Modificar:** `src/lib/citizenApi.ts`, `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx` e `src/modules/citizen/ui.tsx`.

---

### Task 1: Tabelas, triggers, modelos e papel

**Repo:** `apps/api`. Antes, confira que `main` está limpo e crie `feat/citizen-appointments`.

**Files:**
- **Criar:** `db/city_migrate/20260926000001_create_appointments.rb`, `app/models/appointment_request.rb`, `app/models/appointment.rb` e `spec/support/appointment_helpers.rb` (com o `require_relative`).
- **Modificar:**
  - `db/city_schema.rb` e `db/city_triggers.sql`;
  - `app/models/attendance.rb`, `app/models/citizen_verification_code.rb` e `app/models/membership.rb`;
  - `app/commands/attendances/check_in_eligibility.rb` (bug do `NOT IN`);
  - `config/initializers/filter_parameter_logging.rb`.
- **Testes:** `spec/models/appointment_request_spec.rb`, `spec/models/appointment_spec.rb`, `spec/models/attendance_states_spec.rb` e `spec/models/membership_roles_spec.rb` (ampliar).

**Interfaces:**
- **Produces:**
  - `Attendance`:
    - `STATUSES = %w[waiting in_care closed]` e `OUTCOMES = %w[discharged referred return left]`;
    - `belongs_to :triage, optional: true`, `belongs_to :appointment, optional: true` e `belongs_to :called_by_user, optional: true`;
    - `has_one :appointment_request` (por `origin_attendance_id`);
    - scopes `open_attendances` (waiting ou in_care), `waiting` e `in_care`;
    - `#open?`, `#root_triage` e `#priority`.
  - `AppointmentRequest`:
    - `KINDS`, `STATUSES` e `CLOSED_REASONS`;
    - `belongs_to :origin_attendance, :citizen, :root_triage, :origin_unit, :target_unit`, e `:closed_by_user` opcional;
    - `has_many :appointments` (por `request_id`);
    - scope `live_requests` (open ou scheduled);
    - `#latest_appointment`.
  - `Appointment`:
    - `STATUSES`, `ENDED`, `CONFIRMATION_LEAD`, `BORN_CONFIRMED_WITHIN` e `MAX_AHEAD`;
    - `belongs_to :request` (`AppointmentRequest`), `:citizen`, `:health_unit` e `:scheduled_by_user`;
    - `has_one :attendance`;
    - scope `live`;
    - `#ended?` e `#today?`.
  - `CitizenVerificationCode`: `belongs_to :appointment, optional: true`.
  - `Membership::ROLES` e `PRIVILEGED_ROLES` passam a incluir `health_professional`.
  - Helpers de spec:
    - `staff_with(email, *roles)`;
    - `in_care!(attendance, by:)`;
    - `request_for(attendance, kind: "return", target: attendance.health_unit)`, que grava direto e serve para montar cenários.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/support/appointment_helpers.rb
module AppointmentHelpers
  def staff_with(email, *roles)
    User.create!(email_address: email, password: "senha-segura-123").tap do |u|
      roles.each { |r| Membership.create!(user: u, role: r, granted_at: Time.current) }
    end
  end

  # Atendimento aberto por check-in por código (caminho real), já na unidade.
  def waiting_attendance(citizen, unit:, by:)
    triage = completed_web_triage_for(citizen)
    code = Citizens::IssueCheckInCode.call(citizen: citizen, triage: triage).payload.fetch(:code)
    Attendances::CheckIn.call(cpf: citizen.cpf, code: code, health_unit_id: unit.id, document_checked: false, by: by)
                        .payload.fetch(:attendance)
  end

  def in_care!(attendance, by:)
    attendance.update!(status: "in_care", called_by_user: by, called_at: Time.current)
    attendance
  end

  # Grava o pedido direto (cenário de teste); o caminho real é Attendances::Close.
  def request_for(attendance, kind: "return", target: attendance.health_unit)
    AppointmentRequest.create!(origin_attendance: attendance, citizen: attendance.citizen,
                               root_triage: attendance.root_triage, origin_unit: attendance.health_unit,
                               target_unit: target, kind: kind)
  end
end

RSpec.configure { |c| c.include AppointmentHelpers }
```

Acrescente `require_relative "support/appointment_helpers"` em `spec/rails_helper.rb`, junto dos outros `require_relative "support/..."`.

```ruby
# spec/models/attendance_states_spec.rb
require "rails_helper"

RSpec.describe Attendance do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:reception) { staff_with("recepcao@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }

  it "nasce waiting" do
    expect(waiting_attendance(citizen, unit: unit, by: reception).status).to eq("waiting")
  end

  it "o banco aceita waiting → in_care → closed com desfecho clínico" do
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    expect { a.update!(status: "closed", outcome: "discharged", closed_by_user: doctor, closed_at: Time.current) }
      .not_to raise_error
  end

  it "o banco recusa desfecho clínico direto de waiting" do
    a = waiting_attendance(citizen, unit: unit, by: reception)
    expect { a.update!(status: "closed", outcome: "discharged", closed_by_user: doctor, closed_at: Time.current) }
      .to raise_error(ActiveRecord::StatementInvalid, /invalid transition/)
  end

  it "o banco recusa left a partir de in_care" do
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    expect { a.update!(status: "closed", outcome: "left", closed_by_user: doctor, closed_at: Time.current) }
      .to raise_error(ActiveRecord::StatementInvalid, /invalid transition/)
  end

  it "o banco recusa voltar de in_care para waiting e mudar a chamada" do
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    expect { a.update_columns(status: "waiting") }.to raise_error(ActiveRecord::StatementInvalid)
    expect { Attendance.where(id: a.id).update_all(called_at: 1.hour.ago) }
      .to raise_error(ActiveRecord::StatementInvalid, /call never changes/)
  end

  it "exige exatamente uma origem: triagem ou horário" do
    a = waiting_attendance(citizen, unit: unit, by: reception)
    expect { Attendance.where(id: a.id).update_all(triage_id: nil) }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "retorno aceita nota e recusa unidade de destino" do
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    expect do
      a.update!(status: "closed", outcome: "return", referral_note: "reavaliar em 15 dias",
                closed_by_user: doctor, closed_at: Time.current)
    end.not_to raise_error
  end

  it "triagem sem atendimento continua elegível mesmo havendo atendimento sem triagem (NOT IN com NULL)" do
    first = waiting_attendance(citizen, unit: unit, by: reception)
    in_care!(first, by: doctor)
    first.update!(status: "closed", outcome: "return", closed_by_user: doctor, closed_at: Time.current)
    req = request_for(first)
    appt = Appointment.create!(request: req, citizen: citizen, health_unit: unit, scheduled_at: 1.hour.from_now,
                               scheduled_by_user: reception, status: "confirmed", confirmed_at: Time.current)
    Attendance.create!(appointment: appt, citizen: citizen, health_unit: unit, checked_in_by_user: reception,
                       checked_in_at: Time.current, check_in_method: "code")

    fresh = completed_web_triage_for(citizen)
    expect(Attendances::CheckInEligibility.eligible_for(Citizen.where(id: citizen.id))).to include(fresh)
  end

  it "prioridade vem da triagem raiz quando nasce de horário" do
    first = waiting_attendance(citizen, unit: unit, by: reception)
    first.triage.update_columns(priority: 2)
    in_care!(first, by: doctor)
    first.update!(status: "closed", outcome: "return", closed_by_user: doctor, closed_at: Time.current)
    req = request_for(first)
    appt = Appointment.create!(request: req, citizen: citizen, health_unit: unit, scheduled_at: 1.hour.from_now,
                               scheduled_by_user: reception, status: "confirmed", confirmed_at: Time.current)
    second = Attendance.create!(appointment: appt, citizen: citizen, health_unit: unit, checked_in_by_user: reception,
                                checked_in_at: Time.current, check_in_method: "code")
    expect(second.priority).to eq(2)
    expect(second.root_triage).to eq(first.triage)
  end
end
```

```ruby
# spec/models/appointment_request_spec.rb
require "rails_helper"

RSpec.describe AppointmentRequest do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:other_unit) { create_unit("UPA Norte", kind: "upa") }
  let(:reception) { staff_with("recepcao@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:origin) do
    in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor).tap do |a|
      a.update!(status: "closed", outcome: "return", closed_by_user: doctor, closed_at: Time.current)
    end
  end

  it "retorno exige destino igual à origem" do
    expect { request_for(origin, kind: "return", target: other_unit) }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "encaminhamento aceita a própria unidade ou outra" do
    expect { request_for(origin, kind: "referral", target: unit) }.not_to raise_error
  end

  it "um pedido por atendimento de origem" do
    request_for(origin)
    expect { request_for(origin) }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "encerrar como dismissed exige justificativa de 10+" do
    req = request_for(origin)
    expect { req.update!(status: "closed", closed_reason: "dismissed", dismiss_reason: "curto", closed_at: Time.current) }
      .to raise_error(ActiveRecord::StatementInvalid)
  end

  it "pedido encerrado não muda e origem nunca muda" do
    req = request_for(origin)
    expect { AppointmentRequest.where(id: req.id).update_all(kind: "referral") }
      .to raise_error(ActiveRecord::StatementInvalid, /origin columns never change/)
    req.update!(status: "closed", closed_reason: "citizen_cancelled", closed_at: Time.current)
    expect { req.update!(status: "open") }.to raise_error(ActiveRecord::StatementInvalid, /already closed/)
    expect { req.destroy }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
  end
end
```

```ruby
# spec/models/appointment_spec.rb
require "rails_helper"

RSpec.describe Appointment do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:reception) { staff_with("recepcao@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:req) do
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    a.update!(status: "closed", outcome: "return", closed_by_user: doctor, closed_at: Time.current)
    request_for(a)
  end

  def scheduled(at: 3.days.from_now)
    Appointment.create!(request: req, citizen: citizen, health_unit: unit, scheduled_at: at, scheduled_by_user: reception,
                        status: "scheduled", confirmation_deadline_at: at - 24.hours)
  end

  it "scheduled exige prazo" do
    expect do
      Appointment.create!(request: req, citizen: citizen, health_unit: unit, scheduled_at: 3.days.from_now,
                          scheduled_by_user: reception, status: "scheduled")
    end.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "um só horário vivo por pedido" do
    scheduled
    expect { scheduled(at: 4.days.from_now) }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "transições permitidas e proibidas" do
    a = scheduled
    expect { a.update!(status: "checked_in", ended_at: Time.current) }
      .to raise_error(ActiveRecord::StatementInvalid, /invalid transition/)
    a.update!(status: "confirmed", confirmed_at: Time.current)
    a.update!(status: "no_show", ended_at: Time.current)
    expect { a.update!(status: "confirmed") }.to raise_error(ActiveRecord::StatementInvalid, /already ended/)
  end

  it "cancelado pelo cidadão exige motivo de 10+" do
    a = scheduled
    expect { a.update!(status: "cancelled_by_citizen", cancel_reason: "curto", ended_at: Time.current) }
      .to raise_error(ActiveRecord::StatementInvalid)
  end

  it "o que foi marcado nunca muda" do
    a = scheduled
    expect { Appointment.where(id: a.id).update_all(scheduled_at: 5.days.from_now) }
      .to raise_error(ActiveRecord::StatementInvalid, /never change/)
  end

  it "today? usa o fuso da cidade" do
    travel_to(Time.zone.parse("2026-10-02 10:00")) do
      late = scheduled(at: Time.zone.parse("2026-10-02 23:30"))
      expect(late.today?).to be(true)
    end
  end
end
```

Amplie `spec/models/membership_roles_spec.rb`:

```ruby
  it "conhece o papel health_professional e o trata como privilegiado" do
    expect(described_class::ROLES).to include("health_professional")
    expect(described_class::PRIVILEGED_ROLES).to include("health_professional")
    user = User.create!(email_address: "medica@cidade.gov.br", password: "senha-segura-123")
    expect { described_class.create!(user: user, role: "health_professional", granted_at: Time.current) }.not_to raise_error
  end
```

- [ ] **Step 2: Run to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/models/attendance_states_spec.rb spec/models/appointment_request_spec.rb spec/models/appointment_spec.rb spec/models/membership_roles_spec.rb`
Expected: FAIL (`uninitialized constant AppointmentRequest`, status `open`, papel desconhecido).

- [ ] **Step 3: Migration**

```ruby
# db/city_migrate/20260926000001_create_appointments.rb
# Chamada, retorno e agendamento (spec 2026-09-25-citizen-appointments §3; ADR 0019).
# Papel health_professional; attendances ganha estados waiting/in_care/closed,
# chamada, desfecho return e origem triagem-OU-horário; pedido e horário novos.
# Triggers vêm de db/city_triggers.sql (mesma fonte do load_city_schema).
class CreateAppointments < ActiveRecord::Migration[8.1]
  ROLES_BEFORE = %w[citizen_verifier municipal_admin protocol_author protocol_publisher protocol_reviewer viewer].freeze
  ROLES_AFTER = %w[citizen_verifier health_professional municipal_admin protocol_author protocol_publisher
                   protocol_reviewer viewer].freeze

  def up
    replace_roles_check(ROLES_AFTER)

    %w[ck_attendances_status ck_attendances_closing ck_attendances_outcome ck_attendances_referral].each do |name|
      remove_check_constraint :attendances, name: name
    end
    execute "UPDATE attendances SET status = 'waiting' WHERE status = 'open'"
    change_column_default :attendances, :status, from: "open", to: "waiting"
    change_column_null :attendances, :triage_id, true
    add_reference :attendances, :called_by_user, type: :uuid, foreign_key: { to_table: :users }, index: true
    add_column :attendances, :called_at, :datetime

    create_table :appointment_requests, id: :uuid do |t|
      t.references :origin_attendance, type: :uuid, null: false, foreign_key: { to_table: :attendances },
                                       index: { unique: true }
      t.references :citizen, type: :uuid, null: false, foreign_key: true, index: true
      t.references :root_triage, type: :uuid, null: false, foreign_key: { to_table: :triages }, index: true
      t.references :origin_unit, type: :uuid, null: false, foreign_key: { to_table: :health_units }, index: true
      t.references :target_unit, type: :uuid, null: false, foreign_key: { to_table: :health_units }, index: true
      t.string :kind, null: false
      t.text :note
      t.string :status, null: false, default: "open"
      t.string :reopened_reason
      t.string :closed_reason
      t.text :dismiss_reason
      t.references :closed_by_user, type: :uuid, foreign_key: { to_table: :users }, index: true
      t.datetime :closed_at
      t.timestamps
    end
    add_check_constraint :appointment_requests, "kind::text = ANY (ARRAY['return', 'referral']::text[])",
                         name: "ck_appointment_requests_kind"
    add_check_constraint :appointment_requests, "kind::text <> 'return'::text OR origin_unit_id = target_unit_id",
                         name: "ck_appointment_requests_return_same_unit"
    add_check_constraint :appointment_requests, "status::text = ANY (ARRAY['open', 'scheduled', 'closed']::text[])",
                         name: "ck_appointment_requests_status"
    add_check_constraint :appointment_requests,
                         "reopened_reason IS NULL OR reopened_reason::text = ANY (ARRAY['expired', 'no_show']::text[])",
                         name: "ck_appointment_requests_reopened_reason"
    add_check_constraint :appointment_requests,
                         "(status::text <> 'closed'::text AND closed_reason IS NULL AND closed_at IS NULL) OR " \
                         "(status::text = 'closed'::text AND closed_reason IS NOT NULL AND closed_at IS NOT NULL)",
                         name: "ck_appointment_requests_closing"
    add_check_constraint :appointment_requests,
                         "closed_reason IS NULL OR closed_reason::text = ANY (ARRAY['fulfilled', 'citizen_cancelled', 'dismissed']::text[])",
                         name: "ck_appointment_requests_closed_reason"
    add_check_constraint :appointment_requests,
                         "(closed_reason IS DISTINCT FROM 'dismissed' AND dismiss_reason IS NULL) OR " \
                         "(closed_reason = 'dismissed' AND dismiss_reason IS NOT NULL AND length(btrim(dismiss_reason)) >= 10)",
                         name: "ck_appointment_requests_dismiss_reason"

    create_table :appointments, id: :uuid do |t|
      t.references :request, type: :uuid, null: false, foreign_key: { to_table: :appointment_requests }, index: true
      t.references :citizen, type: :uuid, null: false, foreign_key: true, index: true
      t.references :health_unit, type: :uuid, null: false, foreign_key: true, index: true
      t.datetime :scheduled_at, null: false
      t.references :scheduled_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: true
      t.datetime :confirmation_deadline_at
      t.string :status, null: false
      t.datetime :confirmed_at
      t.text :cancel_reason
      t.datetime :ended_at
      t.timestamps
    end
    add_index :appointments, :request_id, unique: true, where: "status IN ('scheduled', 'confirmed')",
              name: "idx_appointments_one_live_per_request"
    add_index :appointments, %i[health_unit_id scheduled_at], name: "idx_appointments_unit_time"
    add_check_constraint :appointments,
                         "status::text = ANY (ARRAY['scheduled', 'confirmed', 'checked_in', 'cancelled_by_citizen', 'expired', 'no_show']::text[])",
                         name: "ck_appointments_status"
    add_check_constraint :appointments, "status::text <> 'scheduled'::text OR confirmation_deadline_at IS NOT NULL",
                         name: "ck_appointments_deadline"
    add_check_constraint :appointments,
                         "(status::text <> 'cancelled_by_citizen'::text AND cancel_reason IS NULL) OR " \
                         "(status::text = 'cancelled_by_citizen'::text AND cancel_reason IS NOT NULL AND length(btrim(cancel_reason)) >= 10)",
                         name: "ck_appointments_cancel_reason"
    add_check_constraint :appointments,
                         "(status::text = ANY (ARRAY['scheduled', 'confirmed']::text[])) = (ended_at IS NULL)",
                         name: "ck_appointments_ended"

    add_reference :attendances, :appointment, type: :uuid, foreign_key: true, index: { unique: true }
    add_check_constraint :attendances, "status::text = ANY (ARRAY['waiting', 'in_care', 'closed']::text[])",
                         name: "ck_attendances_status"
    add_check_constraint :attendances,
                         "outcome IS NULL OR outcome::text = ANY (ARRAY['discharged', 'referred', 'return', 'left']::text[])",
                         name: "ck_attendances_outcome"
    add_check_constraint :attendances, "(called_by_user_id IS NULL) = (called_at IS NULL)",
                         name: "ck_attendances_calling"
    add_check_constraint :attendances,
                         "(status::text = 'waiting'::text AND called_at IS NULL AND outcome IS NULL AND closed_by_user_id IS NULL " \
                         "AND closed_at IS NULL AND referral_unit_id IS NULL AND referral_note IS NULL) OR " \
                         "(status::text = 'in_care'::text AND called_at IS NOT NULL AND outcome IS NULL AND closed_by_user_id IS NULL " \
                         "AND closed_at IS NULL AND referral_unit_id IS NULL AND referral_note IS NULL) OR " \
                         "(status::text = 'closed'::text AND outcome IS NOT NULL AND closed_by_user_id IS NOT NULL AND closed_at IS NOT NULL)",
                         name: "ck_attendances_closing"
    add_check_constraint :attendances,
                         "((outcome IS NULL OR outcome::text = ANY (ARRAY['discharged', 'left']::text[])) " \
                         "AND referral_unit_id IS NULL AND referral_note IS NULL) OR " \
                         "(outcome = 'referred' AND (referral_unit_id IS NOT NULL OR " \
                         "(referral_note IS NOT NULL AND length(btrim(referral_note)) > 0))) OR " \
                         "(outcome = 'return' AND referral_unit_id IS NULL)",
                         name: "ck_attendances_referral"
    add_check_constraint :attendances, "(triage_id IS NULL) <> (appointment_id IS NULL)",
                         name: "ck_attendances_origin"

    add_reference :citizen_verification_codes, :appointment, type: :uuid, foreign_key: true, index: true
    remove_check_constraint :citizen_verification_codes, name: "ck_citizen_verification_codes_purpose_triage"
    add_check_constraint :citizen_verification_codes,
                         "(purpose::text = 'verification'::text AND triage_id IS NULL AND appointment_id IS NULL) OR " \
                         "(purpose::text = 'check_in'::text AND (triage_id IS NULL) <> (appointment_id IS NULL))",
                         name: "ck_citizen_verification_codes_purpose_target"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    raise ActiveRecord::IrreversibleMigration, "attendances e pedidos só aceitam acréscimo; não há volta segura"
  end

  private

  # Mesma forma de 20260924000001: ANY (ARRAY[...]::text[]) sobrevive ao round-trip.
  def replace_roles_check(roles)
    remove_check_constraint :memberships, name: "ck_memberships_role"
    add_check_constraint :memberships, "role::text = ANY (ARRAY[#{roles.map { |r| "'#{r}'" }.join(', ')}]::text[])",
                         name: "ck_memberships_role"
  end
end
```

Observação: a migração troca `open` por `waiting` **antes** de o arquivo de triggers ser re-executado. O guarda antigo permite a troca, porque a linha não está `closed` e as colunas de check-in não mudam.

- [ ] **Step 4: Triggers** (em `db/city_triggers.sql`)

Substitua a função `rota_attendance_guard()` inteira. O bloco `DO $do$` que cria as triggers de `attendances` continua como está.

```sql
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
     OR NEW.appointment_id IS DISTINCT FROM OLD.appointment_id
     OR NEW.citizen_id IS DISTINCT FROM OLD.citizen_id
     OR NEW.health_unit_id IS DISTINCT FROM OLD.health_unit_id
     OR NEW.checked_in_by_user_id IS DISTINCT FROM OLD.checked_in_by_user_id
     OR NEW.checked_in_at IS DISTINCT FROM OLD.checked_in_at
     OR NEW.check_in_method IS DISTINCT FROM OLD.check_in_method
     OR NEW.exception_reason IS DISTINCT FROM OLD.exception_reason
     OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'attendances: the check-in columns never change';
  END IF;
  IF OLD.called_at IS NOT NULL
     AND (NEW.called_at IS DISTINCT FROM OLD.called_at OR NEW.called_by_user_id IS DISTINCT FROM OLD.called_by_user_id) THEN
    RAISE EXCEPTION 'attendances: the call never changes';
  END IF;
  IF NEW.status IS DISTINCT FROM OLD.status
     AND NOT ((OLD.status = 'waiting' AND NEW.status = 'in_care')
          OR (OLD.status = 'waiting' AND NEW.status = 'closed' AND NEW.outcome = 'left')
          OR (OLD.status = 'in_care' AND NEW.status = 'closed' AND NEW.outcome IS DISTINCT FROM 'left')) THEN
    RAISE EXCEPTION 'attendances: invalid transition % -> %', OLD.status, NEW.status;
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;
```

Acrescente no fim do arquivo:

```sql
-- Pedido de agendamento (ADR 0019): só acréscimo; a origem nunca muda; encerrado não muda.
CREATE OR REPLACE FUNCTION rota_appointment_request_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'appointment_requests is append-only: DELETE refused';
  END IF;
  IF OLD.status = 'closed' THEN
    RAISE EXCEPTION 'appointment_requests: already closed';
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id
     OR NEW.origin_attendance_id IS DISTINCT FROM OLD.origin_attendance_id
     OR NEW.citizen_id IS DISTINCT FROM OLD.citizen_id
     OR NEW.root_triage_id IS DISTINCT FROM OLD.root_triage_id
     OR NEW.origin_unit_id IS DISTINCT FROM OLD.origin_unit_id
     OR NEW.target_unit_id IS DISTINCT FROM OLD.target_unit_id
     OR NEW.kind IS DISTINCT FROM OLD.kind
     OR NEW.note IS DISTINCT FROM OLD.note
     OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'appointment_requests: the origin columns never change';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- Horário (ADR 0019): só acréscimo e as transições previstas; o que foi marcado nunca muda.
CREATE OR REPLACE FUNCTION rota_appointment_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'appointments is append-only: DELETE refused';
  END IF;
  IF OLD.status IN ('checked_in', 'cancelled_by_citizen', 'expired', 'no_show') THEN
    RAISE EXCEPTION 'appointments: already ended';
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id
     OR NEW.request_id IS DISTINCT FROM OLD.request_id
     OR NEW.citizen_id IS DISTINCT FROM OLD.citizen_id
     OR NEW.health_unit_id IS DISTINCT FROM OLD.health_unit_id
     OR NEW.scheduled_at IS DISTINCT FROM OLD.scheduled_at
     OR NEW.scheduled_by_user_id IS DISTINCT FROM OLD.scheduled_by_user_id
     OR NEW.confirmation_deadline_at IS DISTINCT FROM OLD.confirmation_deadline_at
     OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'appointments: the scheduled columns never change';
  END IF;
  IF NEW.status IS DISTINCT FROM OLD.status
     AND NOT ((OLD.status = 'scheduled' AND NEW.status IN ('confirmed', 'cancelled_by_citizen', 'expired'))
          OR (OLD.status = 'confirmed' AND NEW.status IN ('checked_in', 'cancelled_by_citizen', 'no_show'))) THEN
    RAISE EXCEPTION 'appointments: invalid transition % -> %', OLD.status, NEW.status;
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF to_regclass('public.appointment_requests') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS appointment_requests_guard ON appointment_requests';
    EXECUTE 'CREATE TRIGGER appointment_requests_guard
      BEFORE UPDATE OR DELETE ON appointment_requests
      FOR EACH ROW EXECUTE FUNCTION rota_appointment_request_guard()';
    EXECUTE 'DROP TRIGGER IF EXISTS appointment_requests_append_only_truncate ON appointment_requests';
    EXECUTE 'CREATE TRIGGER appointment_requests_append_only_truncate
      BEFORE TRUNCATE ON appointment_requests
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
  END IF;
  IF to_regclass('public.appointments') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS appointments_guard ON appointments';
    EXECUTE 'CREATE TRIGGER appointments_guard
      BEFORE UPDATE OR DELETE ON appointments
      FOR EACH ROW EXECUTE FUNCTION rota_appointment_guard()';
    EXECUTE 'DROP TRIGGER IF EXISTS appointments_append_only_truncate ON appointments';
    EXECUTE 'CREATE TRIGGER appointments_append_only_truncate
      BEFORE TRUNCATE ON appointments
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
  END IF;
END
$do$;
```

Observação: plpgsql não valida colunas ao criar a função. Por isso a re-execução do arquivo por migrações antigas (antes de `appointment_id` existir) não quebra.

- [ ] **Step 5: Dump** — atualize `db/city_schema.rb` à mão:
- versão `2026_09_26_000001`;
- tabelas `appointment_requests` e `appointments` com índices e CHECKs;
- em `attendances`: `appointment_id`, `called_at`, `called_by_user_id`, `triage_id` sem `null: false`, `status` com padrão `"waiting"` e os CHECKs novos (`ck_attendances_calling`, `ck_attendances_origin` e os refeitos);
- em `citizen_verification_codes`: `appointment_id` e `ck_citizen_verification_codes_purpose_target`;
- `ck_memberships_role` com `health_professional`;
- as `add_foreign_key` novas.

Rode `docker compose exec -T api bin/rails city:migrate:all` (dev) e `docker compose exec -T api bin/rails city:test_databases`. Depois rode `docker compose exec -T api bundle exec rspec spec/services/city_schema_spec.rb` até passar. Quando houver divergência, copie a forma que o banco devolve.

- [ ] **Step 6: Models**

```ruby
# app/models/appointment_request.rb
# Pedido de agendamento (ADR 0019): nasce do desfecho return/referred com
# unidade; a recepção da unidade de destino marca o horário. Só acréscimo
# (trigger); a origem nunca muda.
class AppointmentRequest < ApplicationRecord
  KINDS = %w[return referral].freeze
  STATUSES = %w[open scheduled closed].freeze
  CLOSED_REASONS = %w[fulfilled citizen_cancelled dismissed].freeze

  belongs_to :origin_attendance, class_name: "Attendance", inverse_of: :appointment_request
  belongs_to :citizen
  belongs_to :root_triage, class_name: "Triage"
  belongs_to :origin_unit, class_name: "HealthUnit"
  belongs_to :target_unit, class_name: "HealthUnit"
  belongs_to :closed_by_user, class_name: "User", optional: true
  has_many :appointments, foreign_key: :request_id, inverse_of: :request, dependent: :restrict_with_error

  scope :live_requests, -> { where(status: %w[open scheduled]) }

  def latest_appointment
    appointments.max_by(&:created_at)
  end
end
```

```ruby
# app/models/appointment.rb
# Horário marcado dentro de um pedido (ADR 0019). Remarcar = linha nova;
# o que foi marcado nunca muda (trigger).
class Appointment < ApplicationRecord
  STATUSES = %w[scheduled confirmed checked_in cancelled_by_citizen expired no_show].freeze
  LIVE = %w[scheduled confirmed].freeze
  ENDED = %w[checked_in cancelled_by_citizen expired no_show].freeze
  CONFIRMATION_LEAD = 24.hours
  BORN_CONFIRMED_WITHIN = 48.hours
  MAX_AHEAD = 180.days

  belongs_to :request, class_name: "AppointmentRequest", inverse_of: :appointments
  belongs_to :citizen
  belongs_to :health_unit
  belongs_to :scheduled_by_user, class_name: "User"
  has_one :attendance

  scope :live, -> { where(status: LIVE) }

  def ended?
    ENDED.include?(status)
  end

  def today?
    scheduled_at.in_time_zone.to_date == Time.zone.today
  end
end
```

```ruby
# app/models/attendance.rb
# Atendimento numa unidade (ADR 0018, 0019): nasce no check-in a partir de
# uma triagem OU de um horário; waiting → in_care (chamada) → closed.
class Attendance < ApplicationRecord
  METHODS = %w[code cpf_exception].freeze
  STATUSES = %w[waiting in_care closed].freeze
  OUTCOMES = %w[discharged referred return left].freeze

  belongs_to :triage, optional: true
  belongs_to :appointment, optional: true
  belongs_to :citizen
  belongs_to :health_unit
  belongs_to :checked_in_by_user, class_name: "User"
  belongs_to :called_by_user, class_name: "User", optional: true
  belongs_to :referral_unit, class_name: "HealthUnit", optional: true
  belongs_to :closed_by_user, class_name: "User", optional: true
  has_one :appointment_request, foreign_key: :origin_attendance_id, inverse_of: :origin_attendance

  scope :open_attendances, -> { where(status: %w[waiting in_care]) }
  scope :waiting, -> { where(status: "waiting") }
  scope :in_care, -> { where(status: "in_care") }

  def open?
    status != "closed"
  end

  # A triagem que começou a cadeia: a própria, ou a do pedido do horário.
  def root_triage
    triage || appointment&.request&.root_triage
  end

  def priority
    root_triage&.priority
  end
end
```

Em `app/models/citizen_verification_code.rb`, acrescente `belongs_to :appointment, optional: true`.

Em `app/models/membership.rb`:
- `ROLES` passa a ser `%w[citizen_verifier health_professional municipal_admin protocol_author protocol_publisher protocol_reviewer viewer]`;
- `PRIVILEGED_ROLES` passa a ser `%w[municipal_admin protocol_reviewer citizen_verifier health_professional]`;
- acrescente ao comentário: "health_professional (spec 2026-09-25 §2.1): chama e registra desfecho clínico — step-up para conceder; o mantenedor não concede."

Em `app/commands/attendances/check_in_eligibility.rb`, corrija o `NOT IN` com NULL:

```ruby
            .where.not(id: Attendance.where.not(triage_id: nil).select(:triage_id))
```

Em `config/initializers/filter_parameter_logging.rb`, acrescente `:note` à lista. `:reason` já cobre `cancel_reason` e `dismiss_reason`.

- [ ] **Step 7: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/models spec/services/city_schema_spec.rb`
Expected: PASS.

Depois rode `docker compose exec -T api bundle exec rspec spec/requests/attendances_spec.rb spec/commands/attendances spec/requests/check_ins_spec.rb spec/requests/health_units_spec.rb spec/requests/attendance_contract_spec.rb spec/requests/citizen_api`. As falhas esperadas vêm do status `open` e do encerramento a partir de `waiting`, e são ajustadas nas Tasks 2 e 4. Neste passo, só atualize as expectativas literais de `"open"` para `"waiting"` onde o spec apenas lê o status.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add db/city_migrate/20260926000001_create_appointments.rb db/city_schema.rb db/city_triggers.sql \
  app/models/appointment_request.rb app/models/appointment.rb app/models/attendance.rb \
  app/models/citizen_verification_code.rb app/models/membership.rb app/commands/attendances/check_in_eligibility.rb \
  config/initializers/filter_parameter_logging.rb spec/support/appointment_helpers.rb spec/rails_helper.rb \
  spec/models/attendance_states_spec.rb spec/models/appointment_request_spec.rb spec/models/appointment_spec.rb \
  spec/models/membership_roles_spec.rb
/opt/homebrew/bin/git commit -m "feat: add appointment requests, appointments and attendance calling states" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Inclua no `add` qualquer spec existente que você tenha ajustado de `"open"` para `"waiting"`.)

---

### Task 2: Chamada, fila e desfecho que gera pedido

**Repo:** `apps/api` (mesmo branch).

**Files:**
- **Criar:** `app/commands/attendances/unit_queue.rb`, `app/commands/attendances/call.rb`, `app/commands/attendances/call_next.rb` e `app/commands/appointment_requests/lifecycle.rb`.
- **Modificar:**
  - `app/commands/attendances/close.rb`;
  - `app/controllers/attendances_controller.rb` e `app/controllers/concerns/attendance_access.rb`;
  - `app/policies/citizen_verification_policy.rb`;
  - `config/routes.rb` e `config/initializers/domain_events.rb`.
- **Testes:**
  - criar `spec/commands/attendances/call_spec.rb`;
  - reescrever `spec/commands/attendances/close_spec.rb` e `spec/requests/attendances_spec.rb`.

**Interfaces:**
- **Consumes (Task 1):** `Attendance` com estados, `#priority`, `AppointmentRequest` e `staff_with`, `waiting_attendance` e `in_care!`.
- **Produces:**
  - `Attendances::UnitQueue.waiting(unit_id) → Array<Attendance>` (prioridade, depois `checked_in_at`) e `.in_care(unit_id) → Array<Attendance>` (por `called_at`);
  - `Attendances::Call.call(attendance:, health_unit_id:, by:) → Result` (`:wrong_unit`, `:already_called`);
  - `Attendances::CallNext.call(health_unit_id:, by:) → Result` (`:queue_empty`);
  - `Attendances::Close.call(attendance:, outcome:, referral_unit_id:, referral_note:, by:) → Result(attendance:, appointment_request:)` (`:invalid_outcome`, `:invalid_unit`, `:referral_required`, `:already_closed` e `:invalid_transition`);
  - `AppointmentRequests::Lifecycle.open_for!(attendance, outcome:, unit:) → AppointmentRequest | nil`, `.close!(request, reason:, by: nil, dismiss_reason: nil)` e `.reopen!(request, reason:)`;
  - na policy: `CitizenVerificationPolicy#care?`;
  - no concern: `require_professional` e `require_attendance_staff`;
  - rotas `GET /attendance/units/:id/queue`, `POST /attendance/attendances/:id/call`, `POST /attendance/units/:id/call_next` e `POST /attendance/attendances/:id/close`.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/commands/attendances/call_spec.rb
require "rails_helper"

RSpec.describe Attendances::Call do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:other_unit) { create_unit("UPA Norte", kind: "upa") }
  let(:reception) { staff_with("recepcao@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:doctor2) { staff_with("medico@cidade.gov.br", "health_professional") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }

  it "chama: waiting → in_care, grava quem e quando, publica evento" do
    a = waiting_attendance(citizen, unit: unit, by: reception)
    expect { described_class.call(attendance: a, health_unit_id: unit.id, by: doctor) }
      .to change { DomainEvent.where(name: "attendance.called").count }.by(1)
    expect(a.reload).to have_attributes(status: "in_care", called_by_user_id: doctor.id)
    expect(a.called_at).to be_present
  end

  it "segundo profissional recebe already_called" do
    a = waiting_attendance(citizen, unit: unit, by: reception)
    described_class.call(attendance: a, health_unit_id: unit.id, by: doctor)
    result = described_class.call(attendance: Attendance.find(a.id), health_unit_id: unit.id, by: doctor2)
    expect(result.reason).to eq(:already_called)
    expect(a.reload.called_by_user_id).to eq(doctor.id)
  end

  it "atendimento de outra unidade: wrong_unit" do
    a = waiting_attendance(citizen, unit: unit, by: reception)
    expect(described_class.call(attendance: a, health_unit_id: other_unit.id, by: doctor).reason).to eq(:wrong_unit)
  end

  describe Attendances::CallNext do
    it "chama o primeiro por prioridade e depois por chegada; fila vazia dá queue_empty" do
      calm = waiting_attendance(Citizen.create!(cpf: "11144477735", phone: "+5541911112222"), unit: unit, by: reception)
      calm.triage.update_columns(priority: 9)
      urgent = waiting_attendance(citizen, unit: unit, by: reception)
      urgent.triage.update_columns(priority: 1)

      expect(Attendances::CallNext.call(health_unit_id: unit.id, by: doctor).payload[:attendance].id).to eq(urgent.id)
      expect(Attendances::CallNext.call(health_unit_id: unit.id, by: doctor).payload[:attendance].id).to eq(calm.id)
      expect(Attendances::CallNext.call(health_unit_id: unit.id, by: doctor).reason).to eq(:queue_empty)
    end
  end
end
```

`spec/commands/attendances/close_spec.rb`: reescreva o arquivo mantendo os casos atuais (três desfechos, `referral_required`, `invalid_unit` e `already_closed`), mas com o atendimento em `in_care` (`in_care!(..., by: doctor)`) e `by: doctor`. Acrescente:

```ruby
  it "left só a partir de waiting" do
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    r = described_class.call(attendance: a, outcome: "left", referral_unit_id: nil, referral_note: nil, by: doctor)
    expect(r.reason).to eq(:invalid_transition)
    fresh = waiting_attendance(Citizen.create!(cpf: "11144477735", phone: "+5541911112222"), unit: unit, by: reception)
    expect(described_class.call(attendance: fresh, outcome: "left", referral_unit_id: nil, referral_note: nil, by: reception))
      .to be_ok
  end

  it "desfecho clínico a partir de waiting: invalid_transition" do
    a = waiting_attendance(citizen, unit: unit, by: reception)
    r = described_class.call(attendance: a, outcome: "discharged", referral_unit_id: nil, referral_note: nil, by: doctor)
    expect(r.reason).to eq(:invalid_transition)
  end

  it "return cria pedido na própria unidade, com a nota, na mesma transação" do
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    r = described_class.call(attendance: a, outcome: "return", referral_unit_id: nil,
                             referral_note: "reavaliar em 15 dias", by: doctor)
    req = r.payload[:appointment_request]
    expect(req).to have_attributes(kind: "return", origin_unit_id: unit.id, target_unit_id: unit.id,
                                   status: "open", note: "reavaliar em 15 dias", root_triage_id: a.triage_id)
    expect(DomainEvent.where(name: "appointment_request.created").count).to eq(1)
  end

  it "referred com unidade cria pedido de encaminhamento; só com descrição não cria" do
    other = create_unit("UPA Norte", kind: "upa")
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    r = described_class.call(attendance: a, outcome: "referred", referral_unit_id: other.id, referral_note: nil, by: doctor)
    expect(r.payload[:appointment_request]).to have_attributes(kind: "referral", target_unit_id: other.id)

    b = in_care!(waiting_attendance(Citizen.create!(cpf: "11144477735", phone: "+5541911112222"), unit: unit,
                                    by: reception), by: doctor)
    r2 = described_class.call(attendance: b, outcome: "referred", referral_unit_id: nil,
                              referral_note: "hospital estadual", by: doctor)
    expect(r2.payload[:appointment_request]).to be_nil
  end

  it "falha ao criar o pedido desfaz o encerramento" do
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    allow(AppointmentRequest).to receive(:create!).and_raise(ActiveRecord::StatementInvalid, "boom")
    expect do
      described_class.call(attendance: a, outcome: "return", referral_unit_id: nil, referral_note: nil, by: doctor)
    end.to raise_error(ActiveRecord::StatementInvalid)
    expect(a.reload.status).to eq("in_care")
  end
```

`spec/requests/attendances_spec.rb`: troque `/open` por `/queue` e `body["attendances"]` por `body["waiting"]`, mantendo o teste de ordem. Troque o encerramento por verifier pelos casos de papel e acrescente:

```ruby
  let(:doctor) { user_with("medica@cidade.gov.br", "health_professional") }

  it "fila em duas partes e chamada pelo profissional" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    a = check_in!(citizen, completed_web_triage_for(citizen))
    sign_in_as(doctor)
    json_post "/attendance/attendances/#{a.id}/call", health_unit_id: unit.id
    expect(response).to have_http_status(:ok)
    get "/attendance/units/#{unit.id}/queue"
    expect(body["waiting"]).to eq([])
    expect(body["in_care"].first).to include("id" => a.id, "called_by_name" => "medica@cidade.gov.br")
  end

  it "recepção não chama nem registra desfecho clínico; pode marcar left" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    a = check_in!(citizen, completed_web_triage_for(citizen))
    sign_in_as(verifier)
    json_post "/attendance/attendances/#{a.id}/call", health_unit_id: unit.id
    expect(response).to have_http_status(:forbidden)
    json_post "/attendance/attendances/#{a.id}/close", outcome: "discharged"
    expect(response).to have_http_status(:forbidden)
    json_post "/attendance/attendances/#{a.id}/close", outcome: "left"
    expect(response).to have_http_status(:ok)
  end

  it "call_next com fila vazia: 404 queue_empty" do
    sign_in_as(doctor)
    json_post "/attendance/units/#{unit.id}/call_next"
    expect(response).to have_http_status(:not_found)
    expect(body["error"]).to eq("queue_empty")
  end

  it "encerrar com return devolve o pedido criado" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    a = check_in!(citizen, completed_web_triage_for(citizen))
    in_care!(a, by: doctor)
    sign_in_as(doctor)
    json_post "/attendance/attendances/#{a.id}/close", outcome: "return", referral_note: "reavaliar"
    expect(response).to have_http_status(:ok)
    expect(body["appointment_request"]).to include("kind" => "return", "target_unit_name" => unit.name)
  end
```

- [ ] **Step 2: Run to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/commands/attendances/call_spec.rb spec/commands/attendances/close_spec.rb spec/requests/attendances_spec.rb`
Expected: FAIL (`uninitialized constant Attendances::Call`, rota `queue` inexistente).

- [ ] **Step 3: Implement**

```ruby
# app/commands/attendances/unit_queue.rb
# Fila da unidade (spec 2026-09-25 §6): aguardando por prioridade (da
# triagem raiz) e chegada; em atendimento por hora da chamada.
module Attendances
  module UnitQueue
    INCLUDES = [ :citizen, :called_by_user, :triage, { appointment: { request: :root_triage } } ].freeze

    module_function

    def waiting(unit_id)
      Attendance.waiting.where(health_unit_id: unit_id).includes(*INCLUDES).to_a
                .sort_by { |a| [ a.priority || 999, a.checked_in_at ] }
    end

    def in_care(unit_id)
      Attendance.in_care.where(health_unit_id: unit_id).includes(*INCLUDES).order(:called_at).to_a
    end
  end
end
```

```ruby
# app/commands/attendances/call.rb
# Chamada (spec 2026-09-25 §2.1): waiting → in_care, sob lock.
module Attendances
  class Call
    def self.call(attendance:, health_unit_id:, by:)
      return Result.fail(:wrong_unit) unless attendance.health_unit_id == health_unit_id.to_s

      state = ApplicationRecord.transaction do
        attendance.lock!
        next :already_called unless attendance.status == "waiting"

        attendance.update!(status: "in_care", called_by_user: by, called_at: Time.current)
        DomainEvents.publish("attendance.called", attendance_id: attendance.id, called_by_user_id: by.id)
        :ok
      end
      state == :ok ? Result.ok(attendance: attendance) : Result.fail(state)
    end
  end
end
```

```ruby
# app/commands/attendances/call_next.rb
# "Chamar próximo": o primeiro da fila; se outro profissional levou esse
# no mesmo instante, tenta o seguinte.
module Attendances
  class CallNext
    ATTEMPTS = 3

    def self.call(health_unit_id:, by:)
      ATTEMPTS.times do
        candidate = UnitQueue.waiting(health_unit_id).first
        return Result.fail(:queue_empty) unless candidate

        result = Call.call(attendance: candidate, health_unit_id: health_unit_id, by: by)
        return result unless result.failure? && result.reason == :already_called
      end
      Result.fail(:already_called)
    end
  end
end
```

```ruby
# app/commands/appointment_requests/lifecycle.rb
# Ciclo do pedido (ADR 0019): abrir a partir do desfecho, encerrar e reabrir.
# Chamado DENTRO da transação de quem muda o estado.
module AppointmentRequests
  module Lifecycle
    module_function

    def open_for!(attendance, outcome:, unit:)
      target = outcome == "return" ? attendance.health_unit : unit
      return nil if target.nil? || !%w[return referred].include?(outcome)

      request = AppointmentRequest.create!(
        origin_attendance: attendance, citizen: attendance.citizen, root_triage: attendance.root_triage,
        origin_unit: attendance.health_unit, target_unit: target,
        kind: outcome == "return" ? "return" : "referral", note: attendance.referral_note
      )
      DomainEvents.publish("appointment_request.created", appointment_request_id: request.id,
                                                          origin_attendance_id: attendance.id,
                                                          target_unit_id: target.id, kind: request.kind)
      request
    end

    def close!(request, reason:, by: nil, dismiss_reason: nil)
      request.update!(status: "closed", closed_reason: reason, closed_by_user: by, closed_at: Time.current,
                      dismiss_reason: dismiss_reason)
      DomainEvents.publish("appointment_request.closed", appointment_request_id: request.id, closed_reason: reason)
    end

    def reopen!(request, reason:)
      request.update!(status: "open", reopened_reason: reason)
    end
  end
end
```

```ruby
# app/commands/attendances/close.rb
# Desfecho (spec 2026-09-25 §2.1, §2.3): left só de waiting; desfecho
# clínico só de in_care. return e referred com unidade abrem o pedido na
# mesma transação.
module Attendances
  class Close
    CLINICAL = %w[discharged referred return].freeze

    def self.call(attendance:, outcome:, referral_unit_id:, referral_note:, by:)
      outcome = outcome.to_s
      return Result.fail(:invalid_outcome) unless Attendance::OUTCOMES.include?(outcome)

      note = referral_note.to_s.strip.presence
      unit = nil
      if outcome == "referred"
        if referral_unit_id.present?
          unit = HealthUnit.active_units.find_by(id: referral_unit_id)
          return Result.fail(:invalid_unit) unless unit
        end
        return Result.fail(:referral_required) if unit.nil? && note.nil?
      end

      ApplicationRecord.transaction do
        attendance.lock!
        next Result.fail(:already_closed) unless attendance.open?
        next Result.fail(:invalid_transition) unless allowed?(attendance.status, outcome)

        attendance.update!(status: "closed", outcome: outcome, closed_by_user: by, closed_at: Time.current,
                           referral_unit: unit, referral_note: (%w[referred return].include?(outcome) ? note : nil))
        request = AppointmentRequests::Lifecycle.open_for!(attendance, outcome: outcome, unit: unit)
        DomainEvents.publish("attendance.closed", attendance_id: attendance.id, outcome: outcome,
                                                  closed_by_user_id: by.id)
        Result.ok(attendance: attendance, appointment_request: request)
      end
    end

    def self.allowed?(status, outcome)
      outcome == "left" ? status == "waiting" : status == "in_care"
    end
  end
end
```

Policy (`app/policies/citizen_verification_policy.rb`), acrescente:

```ruby
  def care?
    role?(:health_professional)
  end
```

Concern (`app/controllers/concerns/attendance_access.rb`), acrescente:

```ruby
  def require_professional
    forbid unless CitizenVerificationPolicy.new(Current.user, nil).care?
  end

  def require_attendance_staff
    policy = CitizenVerificationPolicy.new(Current.user, nil)
    forbid unless policy.verify? || policy.care?
  end
```

Controller (`app/controllers/attendances_controller.rb`), com a reescrita das ações e do JSON:

```ruby
class AttendancesController < ApplicationController
  include Authentication
  include AttendanceAccess

  ERROR_STATUS = {
    invalid_outcome: :unprocessable_entity, referral_required: :unprocessable_entity,
    invalid_unit: :unprocessable_entity, already_closed: :conflict, already_called: :conflict,
    wrong_unit: :unprocessable_entity, invalid_transition: :unprocessable_entity, queue_empty: :not_found
  }.freeze

  before_action :require_attendance_staff, only: %i[queue close]
  before_action :require_professional, only: %i[call call_next]

  def queue
    unit = HealthUnit.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless unit

    render json: { waiting: Attendances::UnitQueue.waiting(unit.id).map { |a| queue_json(a) },
                   in_care: Attendances::UnitQueue.in_care(unit.id).map { |a| queue_json(a) } }
  end

  def call
    attendance = Attendance.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless attendance

    result = Attendances::Call.call(attendance: attendance, health_unit_id: params[:health_unit_id], by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { attendance: attendance_json(result.payload[:attendance]) }
  end

  def call_next
    result = Attendances::CallNext.call(health_unit_id: params[:id], by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { attendance: attendance_json(result.payload[:attendance]) }
  end

  def close
    attendance = Attendance.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless attendance
    return forbid unless params[:outcome].to_s == "left" || CitizenVerificationPolicy.new(Current.user, nil).care?

    result = Attendances::Close.call(attendance: attendance, outcome: params[:outcome],
                                     referral_unit_id: params[:referral_unit_id],
                                     referral_note: params[:referral_note], by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { attendance: attendance_json(result.payload[:attendance]),
                   appointment_request: request_json(result.payload[:appointment_request]) }
  end

  private

  def queue_json(a)
    {
      id: a.id, cpf_masked: a.citizen.cpf_masked, checked_in_at: a.checked_in_at&.iso8601,
      protocol_name: a.root_triage&.protocol_name, priority: a.priority,
      source: a.appointment_id ? "appointment" : "triage", appointment_time: a.appointment&.scheduled_at&.iso8601,
      called_at: a.called_at&.iso8601, called_by_name: a.called_by_user&.email_address
    }
  end

  def attendance_json(a)
    {
      id: a.id, triage_id: a.triage_id, appointment_id: a.appointment_id, health_unit_id: a.health_unit_id,
      unit_name: a.health_unit.name, status: a.status, checked_in_at: a.checked_in_at&.iso8601,
      check_in_method: a.check_in_method, called_at: a.called_at&.iso8601, outcome: a.outcome,
      referral_unit_name: a.referral_unit&.name, referral_note: a.referral_note, closed_at: a.closed_at&.iso8601
    }
  end

  def request_json(r)
    return nil unless r

    { id: r.id, kind: r.kind, target_unit_name: r.target_unit.name, status: r.status }
  end
end
```

Rotas (`config/routes.rb`, dentro de `scope "/attendance"`). Troque `get "units/:id/open"` por:

```ruby
    get  "units/:id/queue",           to: "attendances#queue"
    post "units/:id/call_next",       to: "attendances#call_next"
    post "attendances/:id/call",      to: "attendances#call"
```

Eventos (`config/initializers/domain_events.rb`), junto dos `attendance.*`:

```ruby
  DomainEvents.bind "attendance.called", to: []
  DomainEvents.bind "appointment_request.created", to: []
  DomainEvents.bind "appointment_request.closed", to: []
```

- [ ] **Step 4: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/commands/attendances spec/requests/attendances_spec.rb spec/requests/attendance_contract_spec.rb`
Expected: PASS. Se o `attendance_contract_spec.rb` encerra a partir de `waiting` com desfecho clínico, ajuste-o para chamar antes (`in_care!`) e usar um `health_professional`.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/commands/attendances app/commands/appointment_requests/lifecycle.rb \
  app/controllers/attendances_controller.rb app/controllers/concerns/attendance_access.rb \
  app/policies/citizen_verification_policy.rb config/routes.rb config/initializers/domain_events.rb \
  spec/commands/attendances spec/requests/attendances_spec.rb spec/requests/attendance_contract_spec.rb
/opt/homebrew/bin/git commit -m "feat: let health professionals call patients and open follow-up requests on close" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Pedidos, marcação e agenda (recepção)

**Repo:** `apps/api` (mesmo branch).

**Files:**
- **Criar:**
  - `app/commands/appointments/schedule.rb` e `app/commands/appointment_requests/dismiss.rb`;
  - `app/controllers/appointment_requests_controller.rb`.
- **Modificar:** `app/controllers/health_units_controller.rb` (409 com pedidos vivos), `config/routes.rb` e `config/initializers/domain_events.rb`.
- **Testes:** `spec/commands/appointments/schedule_spec.rb`, `spec/commands/appointment_requests/dismiss_spec.rb`, `spec/requests/appointment_requests_spec.rb` e `spec/requests/health_units_spec.rb` (ampliar).

**Interfaces:**
- **Consumes:** `AppointmentRequests::Lifecycle`, `Appointment::*` e `staff_with`, `waiting_attendance`, `in_care!` e `request_for`.
- **Produces:**
  - `Appointments::Schedule.call(request:, scheduled_at:, health_unit_id:, by:, now: Time.current) → Result(appointment:)`, com os erros `:invalid_time`, `:request_not_open`, `:wrong_unit` e `:invalid_unit`;
  - `AppointmentRequests::Dismiss.call(request:, reason:, health_unit_id:, by:) → Result`, com os erros `:reason_too_short`, `:request_not_open` e `:wrong_unit`;
  - rotas:
    - `GET /attendance/units/:id/requests`;
    - `POST /attendance/requests/:id/appointments {scheduled_at, health_unit_id}`;
    - `POST /attendance/requests/:id/dismiss {reason, health_unit_id}`;
    - `GET /attendance/units/:id/agenda?date=AAAA-MM-DD`.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/commands/appointments/schedule_spec.rb
require "rails_helper"

RSpec.describe Appointments::Schedule do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:reception) { staff_with("recepcao@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:now) { Time.zone.parse("2026-10-01 10:00") }
  let(:req) do
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    Attendances::Close.call(attendance: a, outcome: "return", referral_unit_id: nil, referral_note: nil, by: doctor)
                      .payload.fetch(:appointment_request)
  end

  def schedule(at)
    described_class.call(request: req, scheduled_at: at.iso8601, health_unit_id: unit.id, by: reception, now: now)
  end

  it "com 48h exatas: scheduled com prazo 24h antes" do
    appt = schedule(now + 48.hours).payload[:appointment]
    expect(appt).to have_attributes(status: "scheduled", confirmation_deadline_at: now + 24.hours)
    expect(req.reload).to have_attributes(status: "scheduled", reopened_reason: nil)
  end

  it "com 48h menos 1s: nasce confirmado, sem prazo" do
    appt = schedule(now + 48.hours - 1.second).payload[:appointment]
    expect(appt).to have_attributes(status: "confirmed", confirmation_deadline_at: nil)
    expect(appt.confirmed_at).to be_present
  end

  it "passado, agora e mais de 180 dias: invalid_time" do
    expect(schedule(now - 1.minute).reason).to eq(:invalid_time)
    expect(schedule(now).reason).to eq(:invalid_time)
    expect(schedule(now + 180.days + 1.second).reason).to eq(:invalid_time)
    expect(described_class.call(request: req, scheduled_at: "amanhã", health_unit_id: unit.id, by: reception, now: now)
                          .reason).to eq(:invalid_time)
  end

  it "pedido já agendado: request_not_open" do
    schedule(now + 3.days)
    expect(schedule(now + 4.days).reason).to eq(:request_not_open)
  end

  it "outra unidade: wrong_unit; unidade de destino desativada: invalid_unit" do
    other = create_unit("UPA Norte", kind: "upa")
    expect(described_class.call(request: req, scheduled_at: (now + 3.days).iso8601, health_unit_id: other.id,
                                by: reception, now: now).reason).to eq(:wrong_unit)
    req.target_unit.update!(active: false)
    expect(schedule(now + 3.days).reason).to eq(:invalid_unit)
  end
end
```

```ruby
# spec/commands/appointment_requests/dismiss_spec.rb
require "rails_helper"

RSpec.describe AppointmentRequests::Dismiss do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:reception) { staff_with("recepcao@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:req) do
    a = in_care!(waiting_attendance(Citizen.create!(cpf: "52998224725", phone: "+5541998765432"), unit: unit,
                                    by: reception), by: doctor)
    a.update!(status: "closed", outcome: "return", closed_by_user: doctor, closed_at: Time.current)
    request_for(a)
  end

  it "encerra com justificativa e publica" do
    r = described_class.call(request: req, reason: "cidadão mudou de cidade", health_unit_id: unit.id, by: reception)
    expect(r).to be_ok
    expect(req.reload).to have_attributes(status: "closed", closed_reason: "dismissed", closed_by_user_id: reception.id)
    expect(DomainEvent.where(name: "appointment_request.closed").count).to eq(1)
  end

  it "justificativa curta: reason_too_short; pedido fechado: request_not_open" do
    expect(described_class.call(request: req, reason: "curto", health_unit_id: unit.id, by: reception).reason)
      .to eq(:reason_too_short)
    described_class.call(request: req, reason: "cidadão mudou de cidade", health_unit_id: unit.id, by: reception)
    expect(described_class.call(request: req.reload, reason: "de novo, por engano", health_unit_id: unit.id,
                                by: reception).reason).to eq(:request_not_open)
  end
end
```

```ruby
# spec/requests/appointment_requests_spec.rb
require "rails_helper"

RSpec.describe "Appointment requests", type: :request do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A; Rails.cache.clear }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:reception) { staff_with("recepcao@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  def body = JSON.parse(response.body)

  def returned_attendance(priority: 5)
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    a.triage.update_columns(priority: priority)
    Attendances::Close.call(attendance: a, outcome: "return", referral_unit_id: nil, referral_note: "reavaliar",
                            by: doctor).payload.fetch(:appointment_request)
  end

  it "lista pedidos abertos da unidade, marca horário e aparece na agenda do dia" do
    req = returned_attendance
    sign_in_as(reception)

    get "/attendance/units/#{unit.id}/requests"
    expect(body["requests"].first).to include("id" => req.id, "kind" => "return", "cpf_masked" => citizen.cpf_masked,
                                              "priority" => 5, "note" => "reavaliar", "reopened_reason" => nil)

    at = 3.days.from_now.change(hour: 14, min: 30)
    json_post "/attendance/requests/#{req.id}/appointments", scheduled_at: at.iso8601, health_unit_id: unit.id
    expect(response).to have_http_status(:created)
    expect(body["appointment"]).to include("status" => "scheduled")

    get "/attendance/units/#{unit.id}/requests"
    expect(body["requests"]).to eq([])

    get "/attendance/units/#{unit.id}/agenda", params: { date: at.to_date.iso8601 }
    expect(body["appointments"].first).to include("cpf_masked" => citizen.cpf_masked, "kind" => "return",
                                                  "status" => "scheduled")
  end

  it "profissional não marca horário (403)" do
    req = returned_attendance
    sign_in_as(doctor)
    json_post "/attendance/requests/#{req.id}/appointments", scheduled_at: 3.days.from_now.iso8601,
                                                             health_unit_id: unit.id
    expect(response).to have_http_status(:forbidden)
  end

  it "encerra pedido com justificativa" do
    req = returned_attendance
    sign_in_as(reception)
    json_post "/attendance/requests/#{req.id}/dismiss", reason: "cidadão mudou de cidade", health_unit_id: unit.id
    expect(response).to have_http_status(:ok)
    json_post "/attendance/requests/#{req.id}/dismiss", reason: "curto", health_unit_id: unit.id
    expect(response).to have_http_status(:conflict)
  end
end
```

Amplie `spec/requests/health_units_spec.rb` com:

```ruby
  it "desativar unidade com pedido vivo: 409 unit_has_open_requests" do
    reception = staff_with("recepcao2@cidade.gov.br", "citizen_verifier")
    doctor = staff_with("medica@cidade.gov.br", "health_professional")
    unit = create_unit("UBS Sul")
    a = in_care!(waiting_attendance(Citizen.create!(cpf: "52998224725", phone: "+5541998765432"), unit: unit,
                                    by: reception), by: doctor)
    Attendances::Close.call(attendance: a, outcome: "return", referral_unit_id: nil, referral_note: nil, by: doctor)
    sign_in_as(admin)
    json_post "/attendance/units/#{unit.id}/deactivate"
    expect(response).to have_http_status(:conflict)
    expect(JSON.parse(response.body)["error"]).to eq("unit_has_open_requests")
  end
```

(Use o `admin` que o arquivo já define. Se o nome for outro, siga o do arquivo.)

- [ ] **Step 2: Run to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/commands/appointments/schedule_spec.rb spec/commands/appointment_requests/dismiss_spec.rb spec/requests/appointment_requests_spec.rb spec/requests/health_units_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implement**

```ruby
# app/commands/appointments/schedule.rb
# Marcação pela recepção da unidade de destino (spec 2026-09-25 §2.4, §2.7):
# futuro e até 180 dias; com menos de 48h nasce confirmado, senão exige
# confirmação até 24h antes.
module Appointments
  class Schedule
    def self.call(request:, scheduled_at:, health_unit_id:, by:, now: Time.current)
      at = parse(scheduled_at)
      return Result.fail(:invalid_time) if at.nil? || at <= now || at > now + Appointment::MAX_AHEAD
      return Result.fail(:wrong_unit) unless request.target_unit_id == health_unit_id.to_s
      return Result.fail(:invalid_unit) unless request.target_unit.active?

      born_confirmed = at - now < Appointment::BORN_CONFIRMED_WITHIN
      ApplicationRecord.transaction do
        request.lock!
        next Result.fail(:request_not_open) unless request.status == "open"

        appointment = Appointment.create!(
          request: request, citizen: request.citizen, health_unit: request.target_unit, scheduled_at: at,
          scheduled_by_user: by, status: born_confirmed ? "confirmed" : "scheduled",
          confirmed_at: born_confirmed ? now : nil,
          confirmation_deadline_at: born_confirmed ? nil : at - Appointment::CONFIRMATION_LEAD
        )
        request.update!(status: "scheduled", reopened_reason: nil)
        DomainEvents.publish("appointment.scheduled", appointment_id: appointment.id,
                                                      appointment_request_id: request.id,
                                                      health_unit_id: appointment.health_unit_id,
                                                      born_confirmed: born_confirmed)
        Result.ok(appointment: appointment)
      end
    end

    def self.parse(value)
      Time.zone.iso8601(value.to_s)
    rescue ArgumentError
      nil
    end
  end
end
```

```ruby
# app/commands/appointment_requests/dismiss.rb
# A recepção decide não remarcar (spec 2026-09-25 §2.8): justificativa 10+.
module AppointmentRequests
  class Dismiss
    MIN_REASON = 10

    def self.call(request:, reason:, health_unit_id:, by:)
      return Result.fail(:reason_too_short) if reason.to_s.strip.length < MIN_REASON
      return Result.fail(:wrong_unit) unless request.target_unit_id == health_unit_id.to_s

      ApplicationRecord.transaction do
        request.lock!
        next Result.fail(:request_not_open) unless request.status == "open"

        Lifecycle.close!(request, reason: "dismissed", by: by, dismiss_reason: reason.to_s.strip)
        Result.ok(request: request)
      end
    end
  end
end
```

```ruby
# app/controllers/appointment_requests_controller.rb
# Pedidos de agendamento e agenda do dia da unidade (spec 2026-09-25 §4).
class AppointmentRequestsController < ApplicationController
  include Authentication
  include AttendanceAccess

  ERROR_STATUS = {
    invalid_time: :unprocessable_entity, request_not_open: :conflict, wrong_unit: :unprocessable_entity,
    invalid_unit: :unprocessable_entity, reason_too_short: :unprocessable_entity
  }.freeze

  before_action :require_verifier

  def index
    unit = HealthUnit.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless unit

    requests = AppointmentRequest.where(target_unit: unit, status: "open")
                                 .includes(:citizen, :origin_unit, :root_triage).to_a
                                 .sort_by { |r| [ r.root_triage.priority || 999, r.created_at ] }
    render json: { requests: requests.map { |r| request_json(r) } }
  end

  def schedule
    request = AppointmentRequest.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless request

    result = Appointments::Schedule.call(request: request, scheduled_at: params[:scheduled_at],
                                         health_unit_id: params[:health_unit_id], by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { appointment: appointment_json(result.payload[:appointment]) }, status: :created
  end

  def dismiss
    request = AppointmentRequest.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless request

    result = AppointmentRequests::Dismiss.call(request: request, reason: params[:reason],
                                               health_unit_id: params[:health_unit_id], by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { request: { id: request.id, status: request.status } }
  end

  def agenda
    unit = HealthUnit.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless unit

    day = (Date.iso8601(params[:date].to_s) rescue Time.zone.today)
    appointments = Appointment.where(health_unit: unit, scheduled_at: day.in_time_zone.all_day)
                              .includes(:citizen, :request).order(:scheduled_at)
    render json: { appointments: appointments.map { |a| agenda_json(a) } }
  end

  private

  def request_json(r)
    {
      id: r.id, kind: r.kind, origin_unit_name: r.origin_unit.name, created_at: r.created_at.iso8601,
      cpf_masked: r.citizen.cpf_masked, priority: r.root_triage.priority, note: r.note,
      reopened_reason: r.reopened_reason
    }
  end

  def appointment_json(a)
    { id: a.id, scheduled_at: a.scheduled_at.iso8601, status: a.status,
      confirmation_deadline_at: a.confirmation_deadline_at&.iso8601 }
  end

  def agenda_json(a)
    { id: a.id, scheduled_at: a.scheduled_at.iso8601, cpf_masked: a.citizen.cpf_masked, kind: a.request.kind,
      status: a.status }
  end
end
```

Rotas (dentro de `scope "/attendance"`):

```ruby
    get  "units/:id/requests",         to: "appointment_requests#index"
    get  "units/:id/agenda",           to: "appointment_requests#agenda"
    post "requests/:id/appointments",  to: "appointment_requests#schedule"
    post "requests/:id/dismiss",       to: "appointment_requests#dismiss"
```

Evento: `DomainEvents.bind "appointment.scheduled", to: []`.

`app/controllers/health_units_controller.rb`, na ação `deactivate`, **depois** da checagem de atendimentos abertos que já existe:

```ruby
    if AppointmentRequest.live_requests.where(target_unit: unit).exists?
      return render json: { error: "unit_has_open_requests" }, status: :conflict
    end
```

(Use o nome da variável de unidade que a ação já usa.)

- [ ] **Step 4: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/commands/appointments spec/commands/appointment_requests spec/requests/appointment_requests_spec.rb spec/requests/health_units_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/commands/appointments/schedule.rb app/commands/appointment_requests/dismiss.rb \
  app/controllers/appointment_requests_controller.rb app/controllers/health_units_controller.rb config/routes.rb \
  config/initializers/domain_events.rb spec/commands/appointments spec/commands/appointment_requests \
  spec/requests/appointment_requests_spec.rb spec/requests/health_units_spec.rb
/opt/homebrew/bin/git commit -m "feat: let reception schedule or dismiss follow-up requests and see the day agenda" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Check-in de um horário (código e exceção)

**Repo:** `apps/api` (mesmo branch).

**Files:**
- **Criar:** `app/commands/attendances/appointment_check_in_eligibility.rb` e `app/commands/citizens/issue_appointment_check_in_code.rb`.
- **Modificar:**
  - `app/commands/citizens/issue_counter_code.rb` e `app/commands/citizens/verification_code_match.rb`;
  - `app/commands/attendances/lookup_for_check_in.rb`, `check_in.rb`, `eligible_triages.rb` e `check_in_by_exception.rb`;
  - `app/controllers/check_ins_controller.rb` e `config/initializers/domain_events.rb`.
- **Testes:** `spec/commands/attendances/appointment_check_in_spec.rb` e `spec/requests/check_ins_spec.rb` (ampliar).

**Interfaces:**
- **Consumes:** `Appointment#today?`, `AppointmentRequests::Lifecycle.close!` e `Appointments::Schedule`.
- **Produces:**
  - `Attendances::AppointmentCheckInEligibility.check(appointment, health_unit_id: nil) → :ok | :appointment_not_eligible | :not_today | :wrong_unit`;
  - `Attendances::AppointmentCheckInEligibility.eligible_for(citizens, health_unit_id) → relation`;
  - `Citizens::IssueAppointmentCheckInCode.call(citizen:, appointment:) → Result(code:, expires_at:)`;
  - `Citizens::IssueCounterCode.call(citizen:, purpose:, triage: nil, appointment: nil)`;
  - o payload de `VerificationCodeMatch` ganha `appointment:`;
  - `LookupForCheckIn.call(cpf:, code:, health_unit_id: nil) → Result(citizen:, triage:, appointment:)`;
  - `EligibleTriages.call(cpf:, by:, health_unit_id: nil) → Result(triages:, appointments:)`;
  - `CheckInByException.call(cpf:, triage_id: nil, appointment_id: nil, health_unit_id:, reason:, by:)`.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/commands/attendances/appointment_check_in_spec.rb
require "rails_helper"

RSpec.describe "Check-in de horário" do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:other_unit) { create_unit("UPA Norte", kind: "upa") }
  let(:reception) { staff_with("recepcao@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }

  # Horário confirmado hoje às `hour` (nasce confirmado: < 48h).
  def confirmed_today(hour: 15)
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    a.triage.update_columns(priority: 3)
    req = Attendances::Close.call(attendance: a, outcome: "return", referral_unit_id: nil, referral_note: nil,
                                  by: doctor).payload.fetch(:appointment_request)
    Appointments::Schedule.call(request: req, scheduled_at: Time.zone.now.change(hour: hour).iso8601,
                                health_unit_id: unit.id, by: reception).payload.fetch(:appointment)
  end

  it "por código: abre atendimento do horário, encerra o pedido como fulfilled e publica" do
    travel_to(Time.zone.parse("2026-10-02 09:00")) do
      appt = confirmed_today
      code = Citizens::IssueAppointmentCheckInCode.call(citizen: citizen, appointment: appt).payload.fetch(:code)

      lookup = Attendances::LookupForCheckIn.call(cpf: citizen.cpf, code: code, health_unit_id: unit.id)
      expect(lookup.payload[:appointment]).to eq(appt)
      expect(lookup.payload[:triage]).to be_nil

      r = Attendances::CheckIn.call(cpf: citizen.cpf, code: code, health_unit_id: unit.id, document_checked: false,
                                    by: reception)
      attendance = r.payload[:attendance]
      expect(attendance).to have_attributes(appointment_id: appt.id, triage_id: nil, status: "waiting")
      expect(attendance.priority).to eq(3)
      expect(appt.reload.status).to eq("checked_in")
      expect(appt.request.reload).to have_attributes(status: "closed", closed_reason: "fulfilled")
      expect(DomainEvent.where(name: "appointment.checked_in").count).to eq(1)
    end
  end

  it "horário de outra unidade: wrong_unit com o nome da unidade" do
    travel_to(Time.zone.parse("2026-10-02 09:00")) do
      appt = confirmed_today
      code = Citizens::IssueAppointmentCheckInCode.call(citizen: citizen, appointment: appt).payload.fetch(:code)
      r = Attendances::LookupForCheckIn.call(cpf: citizen.cpf, code: code, health_unit_id: other_unit.id)
      expect(r.reason).to eq(:wrong_unit)
      expect(r.details[:unit_name]).to eq(unit.name)
    end
  end

  it "horário às 23h30 locais é hoje; o código não nasce em outro dia" do
    travel_to(Time.zone.parse("2026-10-02 09:00")) do
      appt = confirmed_today(hour: 23) # 23h00 locais = 02h00 UTC do dia 3
      expect(Citizens::IssueAppointmentCheckInCode.call(citizen: citizen, appointment: appt)).to be_ok
    end
    travel_to(Time.zone.parse("2026-10-03 00:30")) do
      appt = Appointment.last
      expect(Citizens::IssueAppointmentCheckInCode.call(citizen: citizen, appointment: appt).reason).to eq(:not_today)
    end
  end

  it "exceção por CPF lista os horários de hoje desta unidade e faz check-in com motivo" do
    travel_to(Time.zone.parse("2026-10-02 09:00")) do
      appt = confirmed_today
      search = Attendances::EligibleTriages.call(cpf: citizen.cpf, by: reception, health_unit_id: unit.id)
      expect(search.payload[:appointments]).to eq([ appt ])
      expect(Attendances::EligibleTriages.call(cpf: citizen.cpf, by: reception, health_unit_id: other_unit.id)
                                         .payload[:appointments]).to eq([])

      r = Attendances::CheckInByException.call(cpf: citizen.cpf, appointment_id: appt.id, health_unit_id: unit.id,
                                               reason: "chegou sem o celular", by: reception)
      expect(r.payload[:attendance]).to have_attributes(appointment_id: appt.id, check_in_method: "cpf_exception")
      expect(appt.reload.status).to eq("checked_in")
    end
  end

  it "horário ainda scheduled (não confirmado) não recebe check-in" do
    travel_to(Time.zone.parse("2026-10-02 09:00")) do
      a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
      req = Attendances::Close.call(attendance: a, outcome: "return", referral_unit_id: nil, referral_note: nil,
                                    by: doctor).payload.fetch(:appointment_request)
      appt = Appointments::Schedule.call(request: req, scheduled_at: 3.days.from_now.iso8601, health_unit_id: unit.id,
                                         by: reception).payload.fetch(:appointment)
      expect(Citizens::IssueAppointmentCheckInCode.call(citizen: citizen, appointment: appt).reason)
        .to eq(:appointment_not_eligible)
    end
  end
end
```

Em `spec/requests/check_ins_spec.rb`, acrescente:
- um caso de `POST /attendance/check_ins/lookup {cpf, code, health_unit_id}` com código de horário, que devolve `appointment` com `scheduled_at`, `kind` e `priority`, e `triage: null`;
- um caso de `check_ins/search` com `health_unit_id`, que devolve a chave `appointments`;
- um caso de `check_ins/exception` com `appointment_id`, que responde 201.

- [ ] **Step 2: Run to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/commands/attendances/appointment_check_in_spec.rb spec/requests/check_ins_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implement**

```ruby
# app/commands/attendances/appointment_check_in_eligibility.rb
# Check-in de um horário (spec 2026-09-25 §2.10): confirmado, hoje no fuso
# da cidade e desta unidade.
module Attendances
  module AppointmentCheckInEligibility
    module_function

    def check(appointment, health_unit_id: nil)
      return :appointment_not_eligible unless appointment.status == "confirmed"
      return :not_today unless appointment.today?
      return :wrong_unit if health_unit_id.present? && appointment.health_unit_id != health_unit_id.to_s

      :ok
    end

    def eligible_for(citizens, health_unit_id)
      Appointment.where(citizen_id: citizens.select(:id), status: "confirmed", health_unit_id: health_unit_id)
                 .where(scheduled_at: Time.zone.today.all_day).order(:scheduled_at)
    end

    def failure_for(state, appointment)
      return Result.fail(state) unless state == :wrong_unit

      Result.fail(:wrong_unit, details: { unit_name: appointment.health_unit.name })
    end
  end
end
```

```ruby
# app/commands/citizens/issue_appointment_check_in_code.rb
# "Cheguei na unidade" de um horário (spec 2026-09-25 §4): só no dia.
module Citizens
  class IssueAppointmentCheckInCode
    def self.call(citizen:, appointment:)
      state = Attendances::AppointmentCheckInEligibility.check(appointment)
      return Result.fail(state) unless state == :ok

      IssueCounterCode.call(citizen: citizen, purpose: "check_in", appointment: appointment)
    end
  end
end
```

Em `issue_counter_code.rb`, a assinatura passa a `def self.call(citizen:, purpose:, triage: nil, appointment: nil)`, e o `create!` ganha `appointment: appointment`.

Em `verification_code_match.rb`, o retorno passa a `Result.ok(citizen: hit.citizen, verification_code: hit, triage: hit.triage, appointment: hit.appointment)`.

Em `lookup_for_check_in.rb`:

```ruby
    def self.call(cpf:, code:, health_unit_id: nil)
      match = Citizens::VerificationCodeMatch.call(cpf: cpf, code: code, purpose: "check_in")
      return match if match.failure?

      appointment = match.payload[:appointment]
      if appointment
        state = AppointmentCheckInEligibility.check(appointment, health_unit_id: health_unit_id)
        return AppointmentCheckInEligibility.failure_for(state, appointment) unless state == :ok

        return Result.ok(citizen: match.payload[:citizen], triage: nil, appointment: appointment)
      end

      triage = match.payload[:triage]
      state = CheckInEligibility.check(triage)
      return failure_for(state, triage) unless state == :ok

      Result.ok(citizen: match.payload[:citizen], triage: triage, appointment: nil)
    end
```

Em `check_in.rb`, dentro da transação, depois do `match`:

```ruby
        citizen = match.payload[:citizen]
        appointment = match.payload[:appointment]
        triage = match.payload[:triage]
        if appointment
          state = AppointmentCheckInEligibility.check(appointment, health_unit_id: unit.id)
          next result = AppointmentCheckInEligibility.failure_for(state, appointment) unless state == :ok
        else
          state = CheckInEligibility.check(triage)
          next result = LookupForCheckIn.failure_for(state, triage) unless state == :ok
        end

        match.payload[:verification_code].update!(consumed_at: Time.current)
        verified = false
        if document_checked == true && citizen.active_verification.nil?
          Citizens::Verify.record!(citizen: citizen, by: by)
          verified = true
        end
        attendance = Attendance.create!(triage: triage, appointment: appointment, citizen: citizen, health_unit: unit,
                                        checked_in_by_user: by, checked_in_at: Time.current, check_in_method: "code")
        fulfil(appointment) if appointment
        publish(attendance)
        result = Result.ok(attendance: attendance, verified: verified)
```

Acrescente ao `CheckIn`:

```ruby
    # O horário virou atendimento: fecha o horário e o pedido (ADR 0019).
    def self.fulfil(appointment)
      appointment.lock!
      appointment.update!(status: "checked_in", ended_at: Time.current)
      AppointmentRequests::Lifecycle.close!(appointment.request, reason: "fulfilled")
      DomainEvents.publish("appointment.checked_in", appointment_id: appointment.id,
                                                     appointment_request_id: appointment.request_id)
    end
```

No `publish` do `CheckIn`, o payload de `attendance.checked_in` ganha `appointment_id: attendance.appointment_id`.

Em `eligible_triages.rb`:

```ruby
    def self.call(cpf:, by: nil, health_unit_id: nil)
      digits = CitizenIdentity::Cpf.normalize(cpf)
      return Result.fail(:invalid_cpf) unless digits

      citizens = Citizen.where(cpf: digits)
      triages = CheckInEligibility.eligible_for(citizens).to_a
      appointments = health_unit_id.present? ? AppointmentCheckInEligibility.eligible_for(citizens, health_unit_id).to_a : []
      DomainEvents.publish("attendance.exception_searched", by_user_id: by&.id,
                                                            result_count: triages.size + appointments.size)
      Result.ok(triages: triages, appointments: appointments)
    end
```

Em `check_in_by_exception.rb`:
- a assinatura passa a `def self.call(cpf:, health_unit_id:, reason:, by:, triage_id: nil, appointment_id: nil)`;
- quando vier `appointment_id`, busque em `AppointmentCheckInEligibility.eligible_for(Citizen.where(cpf: digits), unit.id).find_by(id: appointment_id)`. Sem resultado, devolva `triage_not_eligible`;
- crie o atendimento com `appointment:` e `triage: nil`, e chame `CheckIn.fulfil(appointment)` na mesma transação;
- sem `appointment_id`, o caminho da triagem continua como está.

No `check_ins_controller.rb`:
- `lookup` passa `health_unit_id: params[:health_unit_id]` e responde `triage: (t && triage_json(t))` e `appointment: (a && appointment_json(a))`;
- `search` passa `health_unit_id:` e responde `{ triages: ..., appointments: ... }`;
- `exception` passa `appointment_id: params[:appointment_id]`;
- `ERROR_STATUS` ganha `not_today: :unprocessable_entity`, `wrong_unit: :unprocessable_entity` e `appointment_not_eligible: :unprocessable_entity`;
- acrescente:

```ruby
  def appointment_json(a)
    root = a.request.root_triage
    { id: a.id, scheduled_at: a.scheduled_at.iso8601, kind: a.request.kind, unit_name: a.health_unit.name,
      protocol_name: root.protocol_name, priority: root.priority }
  end
```

`attendance_json` ganha `appointment_id`. Evento novo: `DomainEvents.bind "appointment.checked_in", to: []`.

- [ ] **Step 4: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/commands/attendances spec/commands/citizens spec/requests/check_ins_spec.rb spec/requests/attendance_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/commands/attendances app/commands/citizens app/controllers/check_ins_controller.rb \
  config/initializers/domain_events.rb spec/commands/attendances/appointment_check_in_spec.rb spec/requests/check_ins_spec.rb
/opt/homebrew/bin/git commit -m "feat: check citizens in for a confirmed appointment on its day" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Lado do cidadão (listar, confirmar, cancelar, código do dia)

**Repo:** `apps/api` (mesmo branch).

**Files:**
- **Criar:** `app/commands/appointments/confirm.rb`, `app/commands/appointments/cancel_by_citizen.rb` e `app/controllers/citizen_api/appointments_controller.rb`.
- **Modificar:** `app/controllers/citizen_api/triages_controller.rb`, `config/routes.rb` e `config/initializers/domain_events.rb`.
- **Testes:** `spec/commands/appointments/citizen_actions_spec.rb` e `spec/requests/citizen_api/appointments_spec.rb`.

**Interfaces:**
- **Consumes:** `Appointment`, `AppointmentRequests::Lifecycle.close!` e `Citizens::IssueAppointmentCheckInCode`.
- **Produces:**
  - `Appointments::Confirm.call(appointment:, now: Time.current) → Result`, com os erros `:confirmation_closed` e `:appointment_ended`. Confirmar um horário já `confirmed` é ok e não faz nada;
  - `Appointments::CancelByCitizen.call(appointment:, reason:) → Result`, com os erros `:reason_too_short` e `:appointment_ended`;
  - rotas:
    - `GET /citizen/appointments?citizen_id=` → `{appointments: [{request, appointment}]}`;
    - `POST /citizen/appointments/:id/confirm`;
    - `POST /citizen/appointments/:id/cancel {reason}`;
    - `POST /citizen/appointments/:id/check_in_code`;
  - o `attendance` de `GET /citizen/triages` ganha `called_at` e `request_kind`.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/commands/appointments/citizen_actions_spec.rb
require "rails_helper"

RSpec.describe "Ações do cidadão no horário" do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:reception) { staff_with("recepcao@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:now) { Time.zone.parse("2026-10-01 10:00") }

  def scheduled_in(delta)
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    req = Attendances::Close.call(attendance: a, outcome: "return", referral_unit_id: nil, referral_note: nil,
                                  by: doctor).payload.fetch(:appointment_request)
    Appointments::Schedule.call(request: req, scheduled_at: (now + delta).iso8601, health_unit_id: unit.id,
                                by: reception, now: now).payload.fetch(:appointment)
  end

  it "confirma 1s antes do prazo; no prazo exato dá confirmation_closed" do
    appt = scheduled_in(3.days) # prazo = now + 2 dias
    expect(Appointments::Confirm.call(appointment: appt, now: appt.confirmation_deadline_at).reason).to eq(:confirmation_closed)
    expect(Appointments::Confirm.call(appointment: appt, now: appt.confirmation_deadline_at - 1.second)).to be_ok
    expect(appt.reload.status).to eq("confirmed")
    expect(Appointments::Confirm.call(appointment: appt, now: appt.confirmation_deadline_at - 1.second)).to be_ok
  end

  it "cancelar exige motivo; cancela horário e encerra o pedido como citizen_cancelled" do
    appt = scheduled_in(3.days)
    expect(Appointments::CancelByCitizen.call(appointment: appt, reason: "curto").reason).to eq(:reason_too_short)
    r = Appointments::CancelByCitizen.call(appointment: appt, reason: "vou viajar nessa semana")
    expect(r).to be_ok
    expect(appt.reload).to have_attributes(status: "cancelled_by_citizen", cancel_reason: "vou viajar nessa semana")
    expect(appt.request.reload).to have_attributes(status: "closed", closed_reason: "citizen_cancelled")
    expect(Appointments::CancelByCitizen.call(appointment: appt, reason: "de novo, por engano").reason)
      .to eq(:appointment_ended)
  end
end
```

(O `Confirm` recebe `now:` para o teste; o controller não passa `now`.)

```ruby
# spec/requests/citizen_api/appointments_spec.rb
require "rails_helper"

RSpec.describe "Citizen appointments", type: :request do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A; Rails.cache.clear }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:reception) { staff_with("recepcao@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let!(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  def body = JSON.parse(response.body)

  def appointment_for(person, delta: 3.days)
    a = in_care!(waiting_attendance(person, unit: unit, by: reception), by: doctor)
    req = Attendances::Close.call(attendance: a, outcome: "return", referral_unit_id: nil, referral_note: nil,
                                  by: doctor).payload.fetch(:appointment_request)
    Appointments::Schedule.call(request: req, scheduled_at: delta.from_now.iso8601, health_unit_id: unit.id,
                                by: reception).payload.fetch(:appointment)
  end

  it "lista, confirma e cancela os próprios" do
    appt = appointment_for(citizen)
    sign_in_citizen("+5541998765432")

    get "/citizen/appointments", params: { citizen_id: citizen.id }
    item = body["appointments"].first
    expect(item["request"]).to include("kind" => "return", "target_unit_name" => unit.name, "status" => "scheduled")
    expect(item["appointment"]).to include("id" => appt.id, "status" => "scheduled", "check_in_available" => false)
    expect(item["appointment"]["confirmation_deadline_at"]).to be_present

    json_post "/citizen/appointments/#{appt.id}/confirm"
    expect(response).to have_http_status(:ok)
    json_post "/citizen/appointments/#{appt.id}/cancel", reason: "curto"
    expect(response).to have_http_status(:unprocessable_entity)
    json_post "/citizen/appointments/#{appt.id}/cancel", reason: "vou viajar nessa semana"
    expect(response).to have_http_status(:ok)
  end

  it "horário de outro celular com o mesmo CPF: 404 nos dois sentidos" do
    other = Citizen.create!(cpf: citizen.cpf, phone: "+5541911112222")
    mine = appointment_for(citizen)
    theirs = appointment_for(other)
    sign_in_citizen("+5541998765432")
    json_post "/citizen/appointments/#{theirs.id}/confirm"
    expect(response).to have_http_status(:not_found)
    get "/citizen/appointments", params: { citizen_id: other.id }
    expect(response).to have_http_status(:not_found)

    sign_in_citizen("+5541911112222")
    json_post "/citizen/appointments/#{mine.id}/cancel", reason: "vou viajar nessa semana"
    expect(response).to have_http_status(:not_found)
  end

  it "código de check-in só no dia do horário confirmado" do
    appt = appointment_for(citizen, delta: 2.hours) # nasce confirmado, hoje
    sign_in_citizen("+5541998765432")
    get "/citizen/appointments", params: { citizen_id: citizen.id }
    expect(body["appointments"].first["appointment"]["check_in_available"]).to be(true)
    json_post "/citizen/appointments/#{appt.id}/check_in_code"
    expect(response).to have_http_status(:created)
    expect(body["code"]).to match(/\A\d{6}\z/)
  end

  it "triagens mostram chamada e tipo do pedido" do
    a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
    Attendances::Close.call(attendance: a, outcome: "return", referral_unit_id: nil, referral_note: nil, by: doctor)
    sign_in_citizen("+5541998765432")
    get "/citizen/triages", params: { citizen_id: citizen.id }
    att = body["triages"].first["attendance"]
    expect(att).to include("status" => "closed", "outcome" => "return", "request_kind" => "return")
    expect(att["called_at"]).to be_present
  end
end
```

- [ ] **Step 2: Run to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/commands/appointments/citizen_actions_spec.rb spec/requests/citizen_api/appointments_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implement**

```ruby
# app/commands/appointments/confirm.rb
# O cidadão confirma até o prazo (spec 2026-09-25 §2.6). Confirmar de novo é ok.
module Appointments
  class Confirm
    def self.call(appointment:, now: Time.current)
      ApplicationRecord.transaction do
        appointment.lock!
        next Result.fail(:appointment_ended) if appointment.ended?
        next Result.ok(appointment: appointment) if appointment.status == "confirmed"
        next Result.fail(:confirmation_closed) if now >= appointment.confirmation_deadline_at

        appointment.update!(status: "confirmed", confirmed_at: now)
        DomainEvents.publish("appointment.confirmed", appointment_id: appointment.id)
        Result.ok(appointment: appointment)
      end
    end
  end
end
```

```ruby
# app/commands/appointments/cancel_by_citizen.rb
# Cancelamento pelo cidadão, com motivo (spec 2026-09-25 §2.5, §2.8): encerra o pedido.
module Appointments
  class CancelByCitizen
    MIN_REASON = 10

    def self.call(appointment:, reason:)
      return Result.fail(:reason_too_short) if reason.to_s.strip.length < MIN_REASON

      ApplicationRecord.transaction do
        appointment.lock!
        next Result.fail(:appointment_ended) if appointment.ended?

        appointment.update!(status: "cancelled_by_citizen", cancel_reason: reason.to_s.strip, ended_at: Time.current)
        AppointmentRequests::Lifecycle.close!(appointment.request, reason: "citizen_cancelled")
        DomainEvents.publish("appointment.cancelled", appointment_id: appointment.id, by: "citizen")
        Result.ok(appointment: appointment)
      end
    end
  end
end
```

```ruby
# app/controllers/citizen_api/appointments_controller.rb
# "Seus agendamentos" (spec 2026-09-25 §4, §6). Só pares do celular da sessão (senão 404).
module CitizenApi
  class AppointmentsController < BaseController
    ERROR_STATUS = {
      confirmation_closed: :conflict, appointment_ended: :conflict, reason_too_short: :unprocessable_entity,
      not_today: :unprocessable_entity, appointment_not_eligible: :unprocessable_entity
    }.freeze

    rate_limit to: 20, within: 1.hour, only: %i[confirm cancel], name: "citizen_appointment_write",
               by: -> { current_citizen_session&.id || request.remote_ip }, store: RateLimitStore,
               with: -> { render_error("too_many_requests", :too_many_requests) }
    rate_limit to: 10, within: 1.hour, only: :check_in_code, name: "citizen_appointment_check_in_code",
               by: -> { current_citizen_session&.id || request.remote_ip }, store: RateLimitStore,
               with: -> { render_error("too_many_requests", :too_many_requests) }

    def index
      citizen = current_citizen_session.citizens.find_by(id: params[:citizen_id])
      return render_error("not_found", :not_found) unless citizen

      requests = AppointmentRequest.where(citizen: citizen).includes(:target_unit, :appointments)
                                   .order(created_at: :desc)
      render json: { appointments: requests.map { |r| item_json(r) } }
    end

    def confirm
      with_appointment { |a| Appointments::Confirm.call(appointment: a) }
    end

    def cancel
      with_appointment { |a| Appointments::CancelByCitizen.call(appointment: a, reason: params[:reason]) }
    end

    def check_in_code
      appointment = own_appointment
      return render_error("not_found", :not_found) unless appointment

      result = Citizens::IssueAppointmentCheckInCode.call(citizen: appointment.citizen, appointment: appointment)
      return render_error(result.reason, ERROR_STATUS.fetch(result.reason, :unprocessable_entity)) if result.failure?

      render json: { code: result.payload[:code], expires_at: result.payload[:expires_at].iso8601 }, status: :created
    end

    private

    def own_appointment
      Appointment.where(citizen_id: current_citizen_session.citizens.select(:id)).find_by(id: params[:id])
    end

    def with_appointment
      appointment = own_appointment
      return render_error("not_found", :not_found) unless appointment

      result = yield(appointment)
      return render_error(result.reason, ERROR_STATUS.fetch(result.reason, :unprocessable_entity)) if result.failure?

      render json: { appointment: appointment_json(appointment.reload) }
    end

    def item_json(request)
      latest = request.latest_appointment
      {
        request: { id: request.id, kind: request.kind, target_unit_name: request.target_unit.name,
                   status: request.status, closed_reason: request.closed_reason,
                   reopened_reason: request.reopened_reason },
        appointment: latest && appointment_json(latest)
      }
    end

    def appointment_json(a)
      { id: a.id, scheduled_at: a.scheduled_at.iso8601, status: a.status,
        confirmation_deadline_at: a.confirmation_deadline_at&.iso8601,
        check_in_available: Attendances::AppointmentCheckInEligibility.check(a) == :ok }
    end
  end
end
```

Rotas (no escopo `/citizen` do `routes.rb`, junto de `triages/:id/check_in_code`):

```ruby
    get  "appointments",                   to: "appointments#index"
    post "appointments/:id/confirm",       to: "appointments#confirm"
    post "appointments/:id/cancel",        to: "appointments#cancel"
    post "appointments/:id/check_in_code", to: "appointments#check_in_code"
```

(Use o mesmo `module:` e `scope` que as rotas vizinhas do cidadão já usam.)

`citizen_api/triages_controller.rb`:
- o `includes(attendance: %i[health_unit referral_unit])` ganha `:appointment_request`;
- o `attendance_json` ganha `called_at: attendance.called_at&.iso8601` e `request_kind: attendance.appointment_request&.kind`.

Eventos: `DomainEvents.bind "appointment.confirmed", to: []` e `DomainEvents.bind "appointment.cancelled", to: []`.

- [ ] **Step 4: Run tests**

Run: `docker compose exec -T api bundle exec rspec spec/commands/appointments spec/requests/citizen_api`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/commands/appointments/confirm.rb app/commands/appointments/cancel_by_citizen.rb \
  app/controllers/citizen_api/appointments_controller.rb app/controllers/citizen_api/triages_controller.rb \
  config/routes.rb config/initializers/domain_events.rb spec/commands/appointments/citizen_actions_spec.rb \
  spec/requests/citizen_api/appointments_spec.rb
/opt/homebrew/bin/git commit -m "feat: let citizens see, confirm or cancel their appointments" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Jobs de expiração e falta, semente e guardas

**Repo:** `apps/api` (mesmo branch).

**Files:**
- **Criar:** `app/commands/appointments/lapse.rb`, `app/jobs/expire_unconfirmed_appointments_job.rb` e `app/jobs/mark_no_show_appointments_job.rb`.
- **Modificar:** `config/recurring.yml`, `config/initializers/domain_events.rb`, `lib/signature_crew.rb`, `spec/adr_pointers_spec.rb` e `spec/requests/attendance_contract_spec.rb`.
- **Testes:** `spec/jobs/appointment_lapse_jobs_spec.rb`.

**Interfaces:**
- **Consumes:** `Appointment`, `AppointmentRequests::Lifecycle.reopen!` e `Appointments::Confirm`.
- **Produces:**
  - `Appointments::Lapse.call(appointment:, to:, now: Time.current) → Result`, com `to` igual a `"expired"` ou `"no_show"`. Não faz nada se o estado já mudou ou se a condição deixou de valer;
  - os dois jobs com `prepend EachCityJob`;
  - as contas de semente `profissional@<cidade>.demo` (`health_professional`) e `recepcao@<cidade>.demo` (`citizen_verifier`), com TOTP fixo.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/jobs/appointment_lapse_jobs_spec.rb
require "rails_helper"

RSpec.describe "Jobs de expiração e falta" do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:reception) { staff_with("recepcao@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:t0) { Time.zone.parse("2026-10-01 10:00") }

  def appointment_at(at, now: t0)
    travel_to(now) do
      a = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
      req = Attendances::Close.call(attendance: a, outcome: "return", referral_unit_id: nil, referral_note: nil,
                                    by: doctor).payload.fetch(:appointment_request)
      Appointments::Schedule.call(request: req, scheduled_at: at.iso8601, health_unit_id: unit.id, by: reception)
                            .payload.fetch(:appointment)
    end
  end

  it "expira no prazo exato, não 1s antes; pedido volta à fila como expired" do
    appt = appointment_at(t0 + 3.days) # prazo = t0 + 2 dias
    travel_to(appt.confirmation_deadline_at - 1.second) { ExpireUnconfirmedAppointmentsJob.perform_now }
    expect(appt.reload.status).to eq("scheduled")
    travel_to(appt.confirmation_deadline_at) { ExpireUnconfirmedAppointmentsJob.perform_now }
    expect(appt.reload.status).to eq("expired")
    expect(appt.request.reload).to have_attributes(status: "open", reopened_reason: "expired")
    expect(DomainEvent.where(name: "appointment.expired").count).to eq(1)
  end

  it "falta só depois da meia-noite local; pedido volta como no_show" do
    appt = appointment_at(Time.zone.parse("2026-10-01 23:00"), now: Time.zone.parse("2026-10-01 20:00")) # nasce confirmado
    travel_to(Time.zone.parse("2026-10-01 23:59")) { MarkNoShowAppointmentsJob.perform_now }
    expect(appt.reload.status).to eq("confirmed")
    travel_to(Time.zone.parse("2026-10-02 00:01")) { MarkNoShowAppointmentsJob.perform_now }
    expect(appt.reload.status).to eq("no_show")
    expect(appt.request.reload).to have_attributes(status: "open", reopened_reason: "no_show")
  end

  it "rodar duas vezes não muda nada a mais" do
    appt = appointment_at(t0 + 3.days)
    travel_to(appt.confirmation_deadline_at + 1.minute) do
      2.times { ExpireUnconfirmedAppointmentsJob.perform_now }
    end
    expect(DomainEvent.where(name: "appointment.expired").count).to eq(1)
  end

  it "se o cidadão confirmou antes do lock do job, o job não faz nada" do
    appt = appointment_at(t0 + 3.days)
    stale = Appointment.find(appt.id) # visão antiga, ainda scheduled
    Appointments::Confirm.call(appointment: appt, now: appt.confirmation_deadline_at - 1.second)
    travel_to(appt.confirmation_deadline_at + 1.minute) do
      expect(Appointments::Lapse.call(appointment: stale, to: "expired")).to be_ok
    end
    expect(appt.reload.status).to eq("confirmed")
    expect(appt.request.reload.status).to eq("scheduled")
  end
end
```

Amplie `spec/requests/attendance_contract_spec.rb` com uma cadeia completa: check-in → chamar → encerrar como `return` → marcar com menos de 48h → check-in do horário → chamar → `discharged`. No fim, confira que `Triage`, `Consent`, `ReportSnapshot` e `DashboardMetric` têm as mesmas contagens e os mesmos `updated_at` de antes da cadeia. Siga o padrão de comparação que o arquivo já usa.

Em `spec/adr_pointers_spec.rb`, troque `VALID_RANGE = (1..18)` por `(1..19)`.

- [ ] **Step 2: Run to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/jobs/appointment_lapse_jobs_spec.rb spec/requests/attendance_contract_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implement**

```ruby
# app/commands/appointments/lapse.rb
# Horário que venceu sem ação (spec 2026-09-25 §5): expired (sem
# confirmação no prazo) ou no_show (confirmado, dia terminou). Sob lock e
# conferindo de novo: se o cidadão agiu antes, não faz nada.
module Appointments
  class Lapse
    FROM = { "expired" => "scheduled", "no_show" => "confirmed" }.freeze

    def self.call(appointment:, to:, now: Time.current)
      ApplicationRecord.transaction do
        appointment.lock!
        next Result.ok(appointment: appointment, skipped: true) unless due?(appointment, to, now)

        appointment.update!(status: to, ended_at: now)
        AppointmentRequests::Lifecycle.reopen!(appointment.request, reason: to)
        DomainEvents.publish("appointment.#{to}", appointment_id: appointment.id,
                                                  appointment_request_id: appointment.request_id)
        Result.ok(appointment: appointment)
      end
    end

    def self.due?(appointment, to, now)
      return false unless appointment.status == FROM.fetch(to)

      if to == "expired"
        appointment.confirmation_deadline_at <= now
      else
        appointment.scheduled_at < now.in_time_zone.beginning_of_day
      end
    end
  end
end
```

```ruby
# app/jobs/expire_unconfirmed_appointments_job.rb
# Horários sem confirmação no prazo viram expired; o pedido volta à fila (ADR 0019).
class ExpireUnconfirmedAppointmentsJob < ApplicationJob
  prepend EachCityJob
  queue_as :housekeeping

  def perform
    Appointment.where(status: "scheduled").where("confirmation_deadline_at <= ?", Time.current).find_each do |a|
      Appointments::Lapse.call(appointment: a, to: "expired")
    end
  end
end
```

```ruby
# app/jobs/mark_no_show_appointments_job.rb
# Horários confirmados cujo dia terminou (fuso da cidade) viram no_show; o pedido volta à fila (ADR 0019).
class MarkNoShowAppointmentsJob < ApplicationJob
  prepend EachCityJob
  queue_as :housekeeping

  def perform
    Appointment.where(status: "confirmed").where("scheduled_at < ?", Time.zone.today.beginning_of_day).find_each do |a|
      Appointments::Lapse.call(appointment: a, to: "no_show")
    end
  end
end
```

Em `config/recurring.yml`, no bloco `default`, junto das tarefas de cidade:

```yaml
  expire_unconfirmed_appointments:
    class: ExpireUnconfirmedAppointmentsJob
    queue: housekeeping
    schedule: "every 15 minutes"

  mark_no_show_appointments:
    class: MarkNoShowAppointmentsJob
    queue: housekeeping
    schedule: "every 15 minutes"
```

Eventos: `DomainEvents.bind "appointment.expired", to: []` e `DomainEvents.bind "appointment.no_show", to: []`.

Semente (`lib/signature_crew.rb`), acrescente ao fim de `MEMBERS` e ajuste o comentário da classe para mencionar "e a dupla do atendimento (spec 2026-09-25 §6)":

```ruby
    { email_prefix: "profissional", role: "health_professional", secret_env: "DEV_PROFESSIONAL_OTP_SECRET",
      default_secret: "GEZDGNBVGY3TQOJQGEZDGNBVGY3TQOJQ" },
    { email_prefix: "recepcao", role: "citizen_verifier", secret_env: "DEV_RECEPTION_OTP_SECRET",
      default_secret: "MZXW6YTBOIQHEZLDMVUXEZLTOQQGC3TE" }
```

- [ ] **Step 4: Run tests and the full suite**

Run: `docker compose exec -T api bundle exec rspec spec/jobs spec/requests/attendance_contract_spec.rb spec/adr_pointers_spec.rb`
Expected: PASS.

Depois, a suíte completa (em primeiro plano):

```bash
docker compose stop worker
docker compose exec -T api bundle exec rspec
docker compose start worker
```

Expected: 0 falhas, em cerca de 3 minutos. Rode a semente em dev: `docker compose exec -T api bin/rails db:seed`. Nas linhas `[seeds]` das duas contas novas, confira só o e-mail e o papel. **Não copie o link `otpauth` para o relatório.**

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/commands/appointments/lapse.rb app/jobs/expire_unconfirmed_appointments_job.rb \
  app/jobs/mark_no_show_appointments_job.rb config/recurring.yml config/initializers/domain_events.rb \
  lib/signature_crew.rb spec/jobs/appointment_lapse_jobs_spec.rb spec/requests/attendance_contract_spec.rb \
  spec/adr_pointers_spec.rb
/opt/homebrew/bin/git commit -m "feat: expire unconfirmed appointments and mark no-shows; seed care staff" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Dashboard — papel do profissional, fila e desfecho

**Repo:** `apps/dashboard`. Antes, confira que `main` está limpo e crie `feat/citizen-appointments`. Rode os testes com `npm test`, na pasta `apps/dashboard`.

**Files:**
- **Modificar:** `src/lib/api.ts`, `src/lib/attendance.ts`, `src/lib/team.ts`, `src/modules/Team.tsx`, `src/shell/modules.ts` e `src/modules/Attendance.tsx`.
- **Criar:** `src/modules/attendance/UnitQueue.tsx` e `src/modules/attendance/UnitQueue.test.tsx`.
- **Remover:** `src/modules/attendance/OpenAttendances.tsx` e `OpenAttendances.test.tsx`. Os casos de encerramento que ainda valem migram para `UnitQueue.test.tsx`.
- **Testes:** `Team.test.tsx`, `modules.test.ts` e `Attendance.test.tsx` (ampliar).

**Interfaces:**
- **Consumes (API, Tasks 2 e 3):**
  - `GET /attendance/units/:id/queue` → `{waiting: QueueRow[], in_care: QueueRow[]}`;
  - `POST /attendance/attendances/:id/call {health_unit_id}`;
  - `POST /attendance/units/:id/call_next`;
  - `POST /attendance/attendances/:id/close` → `{attendance, appointment_request?}`.
- **Produces (`src/lib/api.ts`):**
  - `interface QueueRow { id; cpf_masked; checked_in_at; protocol_name: string | null; priority: number | null; source: "triage" | "appointment"; appointment_time: string | null; called_at: string | null; called_by_name: string | null }`;
  - `listUnitQueue(unitId) → Promise<{ waiting: QueueRow[]; in_care: QueueRow[] }>`;
  - `callAttendance(id, unitId)` e `callNext(unitId)`;
  - `type AttendanceOutcome = "discharged" | "referred" | "return" | "left"`;
  - `closeAttendance(...)` passa a devolver `{ attendance, appointmentRequest: { id; kind; target_unit_name } | null }`.
- **Produces (`src/lib/team.ts`):** `PROFESSIONAL_ROLE = "health_professional"`, e `TeamMember` ganha `isProfessional` e `professionalMembershipId`.
- **Produces (`src/lib/attendance.ts`):** mensagens para `already_called`, `queue_empty`, `wrong_unit`, `invalid_transition`, `request_not_open`, `invalid_time`, `not_today` e `unit_has_open_requests`.

**Comportamento (spec §6), cada item com um teste em `UnitQueue.test.tsx`:**
1. **Duas listas:** "Aguardando", na ordem que a API manda, e "Em atendimento", com "chamado por *e-mail* às *hh:mm*". A prioridade aparece com o número, e a origem "Agendamento *hh:mm*" aparece quando `source === "appointment"`.
2. **Profissional** (`canCare`):
   - botão **"Chamar próximo"** no topo, e **"Chamar"** em cada linha de "Aguardando";
   - **"Encerrar"** em cada linha de "Em atendimento", que abre o painel de desfecho com cabeçalho nomeando o atendimento (como hoje) e as opções "Atendido e liberado", "Encaminhado" e "Retorno";
   - "Encaminhado" oferece a lista de unidades ativas, **incluindo a própria**, e/ou uma descrição;
   - "Retorno" oferece uma nota opcional.
   - Com unidade no encaminhamento, ou com Retorno, o painel mostra "Gera pedido de agendamento na *unidade*".
   - Depois de encerrar com pedido, aparece a confirmação "Pedido de agendamento criado na *unidade*".
3. **Recepção sem o papel de profissional:** não vê "Chamar", "Chamar próximo" nem "Encerrar" em "Em atendimento". Em "Aguardando", vê apenas **"Saiu sem atendimento"**, que os dois papéis veem.
4. **`queue_empty`** mostra "Ninguém aguardando". **`already_called`** recarrega a fila sem erro parado, no mesmo padrão do `already_closed` de hoje.
5. **Unidades novas:** a lista de destino usa `listActiveUnits` com a queryKey `["activeUnits"]`. Depois de criar ou reativar uma unidade em `Units.tsx`, chame `queryClient.invalidateQueries({ queryKey: ["activeUnits"] })`. Isso corrige o item 1 do card dashboard#4. Teste em `Units.test.tsx`.

**Papéis:**
- em `modules.ts`, `canAttend` passa a incluir `roles.includes("health_professional")`;
- em `Attendance.tsx`, o seletor de unidade e a fila aparecem para `canVerify || canCare`. `CheckIn` e `Counter` continuam só para `canVerify`;
- a tela Equipe ganha conceder e revogar `health_professional`, no mesmo padrão com step-up de `citizen_verifier`, com o rótulo "Profissional de saúde". Teste em `Team.test.tsx`.

- [ ] **Step 1:** Escreva os testes de `UnitQueue.test.tsx`, um por item de comportamento, com `vi.mock("../../lib/api")` como em `OpenAttendances.test.tsx`. Amplie também `Team.test.tsx`, `modules.test.ts` e `Units.test.tsx`.
- [ ] **Step 2:** Rode `npm test` e veja os testes novos falharem.
- [ ] **Step 3:** Implemente `api.ts`, `team.ts`, `attendance.ts`, `UnitQueue.tsx`, `Attendance.tsx`, `Team.tsx`, `modules.ts` e `Units.tsx`. Remova `OpenAttendances.*`.
- [ ] **Step 4:** Rode `npm test` (tudo verde) e `npx tsc --noEmit` (sem erros).
- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/api.ts src/lib/attendance.ts src/lib/team.ts src/modules/Team.tsx src/modules/Team.test.tsx \
  src/shell/modules.ts src/shell/modules.test.ts src/modules/Attendance.tsx src/modules/Attendance.test.tsx \
  src/modules/attendance/UnitQueue.tsx src/modules/attendance/UnitQueue.test.tsx src/modules/attendance/Units.tsx \
  src/modules/attendance/Units.test.tsx
/opt/homebrew/bin/git rm src/modules/attendance/OpenAttendances.tsx src/modules/attendance/OpenAttendances.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the care queue with calling and follow-up outcomes to the attendance screen" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Dashboard — pedidos, agenda do dia e check-in de horário

**Repo:** `apps/dashboard` (mesmo branch).

**Files:**
- **Criar:** `src/modules/attendance/Requests.tsx` e `Agenda.tsx`, cada um com testes.
- **Modificar:** `src/lib/api.ts`, `src/modules/attendance/CheckIn.tsx` (com testes) e `src/modules/Attendance.tsx`.

**Interfaces:**
- **Consumes (Tasks 3 e 4):**
  - `GET /attendance/units/:id/requests`;
  - `POST /attendance/requests/:id/appointments {scheduled_at, health_unit_id}`;
  - `POST /attendance/requests/:id/dismiss {reason, health_unit_id}`;
  - `GET /attendance/units/:id/agenda?date=`;
  - o `lookup` com `health_unit_id` (devolve `appointment`), o `search` com `health_unit_id` (devolve `appointments`) e o `exception` com `appointment_id`.
- **Produces (`api.ts`):**
  - `interface RequestRow { id; kind: "return" | "referral"; origin_unit_name; created_at; cpf_masked; priority: number | null; note: string | null; reopened_reason: "expired" | "no_show" | null }`;
  - `listUnitRequests(unitId)`, `scheduleRequest(id, scheduledAtIso, unitId)`, `dismissRequest(id, reason, unitId)` e `listUnitAgenda(unitId, dateIso)`;
  - `interface CheckInAppointment { id; scheduled_at; kind; unit_name; protocol_name; priority }`;
  - `lookupCheckIn(cpf, code, unitId)` passa a devolver `{ citizen, triage: CheckInTriage | null, appointment: CheckInAppointment | null }`;
  - `searchCheckIn(cpf, unitId)` passa a devolver `{ triages, appointments }`;
  - `checkInByException(cpf, target: { triageId } | { appointmentId }, unitId, reason)`.

**Comportamento (spec §6), cada item com um teste:**
1. **Requests.tsx:**
   - só para `canVerify`;
   - lista os pedidos com tipo ("Retorno" ou "Encaminhado de *origem*"), data, CPF mascarado, prioridade, nota e a marca "novo", "sem confirmação" (`expired`) ou "faltou" (`no_show`).
2. **"Marcar horário":** abre um `<input type="datetime-local">` e mostra um aviso calculado no navegador:
   - com menos de 48h até o horário: "O horário nasce confirmado";
   - senão: "O cidadão precisa confirmar até *dd/mm hh:mm*", que é o horário menos 24h.
   - Envie `new Date(valor).toISOString()`.
   - Depois de marcar, recarregue os pedidos e a agenda.
3. **"Encerrar pedido":** exige justificativa de pelo menos 10 caracteres (o botão fica desabilitado antes disso). `request_not_open` recarrega a lista.
4. **Agenda.tsx:**
   - só para `canVerify`;
   - mostra os horários de hoje: hora, CPF mascarado, tipo e estado ("aguardando confirmação", "confirmado" e "check-in feito" para `checked_in`, com os demais estados por extenso);
   - tem um seletor de data, com hoje como padrão.
5. **CheckIn.tsx:**
   - o `lookup` envia `unit.id`. Quando vier `appointment`, o cartão mostra "Agendamento *hh:mm* · Retorno" (ou "· Encaminhamento") com a prioridade;
   - a busca da exceção lista primeiro os horários de hoje ("Agendamento *hh:mm*") e depois as triagens, e a escolha envia `appointmentId` ou `triageId`;
   - `wrong_unit` mostra "Este agendamento é na *unit_name*", e `not_today` mostra "Este agendamento não é para hoje".

**Montagem em `Attendance.tsx`:** o módulo de recepção é Unidade → Check-in → Fila → Pedidos → Agenda → Balcão → (admin) Histórico e Unidades.

- [ ] **Step 1:** Escreva os testes de `Requests.test.tsx` e `Agenda.test.tsx` e amplie `CheckIn.test.tsx`, um caso por item acima. O aviso de "nasce confirmado" deve ser testado com `vi.setSystemTime`.
- [ ] **Step 2:** Rode `npm test` e veja os testes falharem.
- [ ] **Step 3:** Implemente.
- [ ] **Step 4:** Rode `npm test` e `npx tsc --noEmit`.
- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/api.ts src/modules/Attendance.tsx src/modules/attendance/Requests.tsx \
  src/modules/attendance/Requests.test.tsx src/modules/attendance/Agenda.tsx src/modules/attendance/Agenda.test.tsx \
  src/modules/attendance/CheckIn.tsx src/modules/attendance/CheckIn.test.tsx
/opt/homebrew/bin/git commit -m "feat: add follow-up requests, day agenda and appointment check-in to reception" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: wpda — "Seus agendamentos"

**Repo:** `apps/wpda`. Antes, confira que `main` está limpo e crie `feat/citizen-appointments`. Rode os testes com `npm test`, na pasta `apps/wpda`.

**Files:**
- **Criar:** `src/modules/citizen/AppointmentsSection.tsx` e `AppointmentsSection.test.tsx`.
- **Modificar:** `src/lib/citizenApi.ts`, `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx` e `src/modules/citizen/ui.tsx`.

**Interfaces:**
- **Consumes (Task 5):**
  - `GET /citizen/appointments?citizen_id=`;
  - `POST /citizen/appointments/:id/confirm`, `/cancel {reason}` e `/check_in_code`;
  - o `attendance` das triagens com `called_at?` e `request_kind?`.
- **Produces (`citizenApi.ts`):**
  - `interface AppointmentItem { request: { id; kind: "return" | "referral"; target_unit_name; status: "open" | "scheduled" | "closed"; closed_reason: ... | null; reopened_reason: "expired" | "no_show" | null }; appointment: { id; scheduled_at; status: ...; confirmation_deadline_at: string | null; check_in_available: boolean } | null }`;
  - `citizenApi.appointments(citizenId)`, `confirmAppointment(id)`, `cancelAppointment(id, reason)` e `issueAppointmentCheckInCode(id)`.
  - **Normalização na borda:** `normalizeAppointment` troca campos ausentes por `null` e `check_in_available` ausente por `false`. `normalizeTriage` passa a trocar o `status` legado `"open"` por `"waiting"`, e `called_at` e `request_kind` ausentes por `null`.
  - `AttendanceSummary.status` passa a ser `"waiting" | "in_care" | "closed"` e `outcome` ganha `"return"`.

**Comportamento (spec §6), cada item com um teste em `AppointmentsSection.test.tsx`:**
1. **Seção "Seus agendamentos":** fica acima de "Minhas triagens" em `HistoryStep` e só aparece quando a lista não está vazia.
2. **Pedido aberto:**
   - retorno: "Retorno pedido na *unidade* — a unidade vai marcar o horário";
   - encaminhamento: "Encaminhamento para *unidade* — a unidade vai marcar o horário";
   - se voltou por prazo ou falta, acrescenta "A unidade pode marcar outro horário".
3. **Horário `scheduled`:** "Agendado: *qui, 02/10, 14h30* — *unidade*. Confirme até *qua, 01/10, 14h30*", com os botões **Confirmar** e **Cancelar**.
4. **Horário `confirmed`:** "Confirmado: …", com **Cancelar**. Com `check_in_available`, aparece também **"Cheguei na unidade"**, que abre a mesma tela de código (`CounterCodeStep`, com a função de emitir memorizada por `useCallback`, como em `Flow`).
5. **Cancelar:** mostra um campo "Motivo do cancelamento" e o botão "Cancelar agendamento", desabilitado até 10 caracteres. Depois de cancelar, recarrega a lista.
6. **Confirmar:** recarrega a lista. `confirmation_closed` mostra "O prazo para confirmar terminou".
7. **Estados finais:** "Cancelado por você", "Cancelado: sem confirmação no prazo — a unidade pode marcar outro horário", "Você não compareceu — a unidade pode marcar outro horário" e "Atendido na *unidade*" (`checked_in`).
8. **Em "Minhas triagens"** (`HistoryStep`):
   - o atendimento mostra "Aguardando atendimento na *unidade*" (`waiting`) ou "Em atendimento na *unidade* desde *hh:mm*" (`in_care`, usando `called_at`);
   - o desfecho `return` mostra "Retorno — veja em Seus agendamentos".
9. **Datas:** use `Intl.DateTimeFormat("pt-BR", { timeZone: "America/Sao_Paulo", weekday: "short", day: "2-digit", month: "2-digit", hour: "2-digit", minute: "2-digit" })`.
10. **Acessibilidade:** botões com pelo menos 48px de altura e texto com pelo menos 18px, como nos componentes vizinhos.

- [ ] **Step 1:** Escreva os testes, um por item. Amplie os testes de `HistoryStep` e a normalização em `citizenApi`.
- [ ] **Step 2:** Rode `npm test` e veja os testes falharem.
- [ ] **Step 3:** Implemente.
- [ ] **Step 4:** Rode `npm test` e `npx tsc --noEmit`.
- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/citizenApi.ts src/modules/citizen/AppointmentsSection.tsx \
  src/modules/citizen/AppointmentsSection.test.tsx src/modules/citizen/HistoryStep.tsx src/modules/citizen/Flow.tsx \
  src/modules/citizen/ui.tsx src/modules/citizen/*.test.tsx
/opt/homebrew/bin/git commit -m "feat: show, confirm and cancel appointments in the citizen web channel" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Verificação de ponta a ponta em desenvolvimento, com prints

**Repos:** nenhum código, a não ser que a verificação ache um bug (nesse caso, corrija no repositório certo com teste e commit `fix:`).

- [ ] **Step 1:** Com a migração aplicada em dev e a semente rodada, confira pelo `rails runner`:
  - que `profissional@curitiba.demo` e `recepcao@curitiba.demo` existem com os papéis certos;
  - que `SELECT status, count(*) FROM attendances GROUP BY 1` não tem mais nenhum `open`.
- [ ] **Step 2:** Pelo `rails runner`, com os comandos reais, em Curitiba:
  1. triagem web de um cidadão novo;
  2. check-in pela recepção;
  3. `Attendances::Call` pelo profissional;
  4. `Close` como `return`;
  5. `Appointments::Schedule` com 3 dias de antecedência (sai `scheduled`);
  6. `Confirm`;
  7. um segundo pedido marcado para daqui a 2h (nasce `confirmed`), com o código do dia e o check-in desse horário, que vai para `fulfilled`;
  8. um terceiro pedido com prazo forçado por `travel_to`, rodando `ExpireUnconfirmedAppointmentsJob.perform_now` (o pedido volta como `expired`).

  Registre os ids e estados no relatório, **sem códigos**.
- [ ] **Step 3:** No wpda (`http://curitiba.localhost:5176/wpda/`), com o celular do cidadão do passo 2, confira "Seus agendamentos" com os três estados e o botão "Cheguei na unidade" no horário de hoje.
- [ ] **Step 4:** O dashboard exige login de servidor, feito pelo usuário. Deixe um roteiro no relatório:
  - **como `profissional@curitiba.demo`:** fila, "Chamar próximo" e encerrar como Retorno;
  - **como `recepcao@curitiba.demo`:** Pedidos, "Marcar horário" (os dois avisos), Agenda do dia e o check-in do horário.
  - Os prints vão para `evidencias/<data>-agendamento/`, com um `README.md` de índice, como em `evidencias/2026-09-25-telas-atendimento/`.
- [ ] **Step 5:** Suítes finais:
  - api completa, com o worker parado e reiniciado depois;
  - `npm test` no dashboard e no wpda.

  Registre as contagens no relatório.
