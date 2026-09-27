# Módulo 10 — Profissionais (api, fatias 1 a 4) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Profissional de saúde como perfil 1:1 do usuário, vinculado a unidades com CBO, com turnos com data, e chamada/desfecho clínico restritos a quem tem vínculo ativo com a unidade do atendimento (F-10.1 a F-10.5, ADR 0021).

**Architecture:** Três tabelas novas no banco de cada cidade (uma migração), protegidas por triggers de só-acréscimo em `db/city_triggers.sql`, índice único parcial e exclusion constraint. Comandos em `app/commands/professionals/` devolvem `Result`. Três controllers sob o prefixo `/professionals`. A regra clínica mora em `Professionals::ClinicalAuthorization.check`, chamada dentro da transação de `Attendances::Call`/`Close` e antes do laço de `CallNext`.

**Tech Stack:** Rails 8.1 (API), PostgreSQL ≥ 15 (`btree_gist`), RSpec, Active Record Encryption (chave da cidade).

**Spec:** `docs/superpowers/specs/2026-09-27-module-10-professionals-design.md` (leia antes de começar; este plano argumenta a partir dela).

## Global Constraints

- Tudo no banco de cada cidade (`db/city_migrate`, `db/city_schema.rb` à mão, `db/city_triggers.sql`); nada no banco de plataforma.
- `db/city_triggers.sql` é re-executado por migrações antigas: todo `CREATE TRIGGER` novo fica dentro de `DO $do$ ... IF to_regclass('public.<tabela>') IS NOT NULL ...`.
- O dump `db/city_schema.rb` é feito à mão; o juiz é `spec/services/city_schema_spec.rb`. Texto exato de constraint vem de `pg_get_constraintdef` de um banco migrado.
- Eventos (`DomainEvents.publish`) carregam **só ids** (e `fields` com nomes de campo). Nunca CNS, registro, telefone, e-mail ou nome.
- Todo evento novo é declarado em `config/initializers/domain_events.rb` com `to: []`.
- Todo comando devolve `Result` (`app/commands/result.rb`).
- Recusas da regra clínica: `missing_role` e `missing_link`, sempre HTTP 403 com `{"error": "<motivo>"}`.
- `HealthUnit.lock_active!` continua onde está em `CheckIn`, `CheckInByException` e `Close`.
- Specs de request precisam de `type: :request` (a inferência por pasta está desligada).
- Spec com thread: `self.use_transactional_tests = false`, `pop(timeout: 5)` e limpeza no `after` (modelo: `spec/models/health_unit_lock_spec.rb`).
- Nenhum teste com data fixa contra o relógio real: use `travel_to` ou datas relativas a `Time.current`.
- Commits em inglês, Conventional Commits, com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

- Worktree do api a partir de `origin/main`:

  ```bash
  cd apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add -b feat/mod-10-professionals .claude/mod10 origin/main && cp config/master.key .claude/mod10/config/master.key
  ```

- Specs rodam dentro do container, no diretório do worktree:

  ```bash
  docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec <arquivos>
  ```

- Depois da migração (Task 1), recarregue os bancos de teste:

  ```bash
  docker compose exec -T -w /rails/.claude/mod10 api bin/rails city:test_databases
  ```

- Suíte completa só com o worker parado e sem outra sessão rodando suíte completa ao mesmo tempo:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec
  docker compose start worker
  ```

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API").

## Review Focus

1. **Vínculo encerrado com a tela do profissional aberta:** a chamada seguinte recebe 403 `missing_link`, nunca 500 nem sucesso. Teste: Task 11, "vínculo encerrado entre a leitura e a chamada".
2. **Plantão que atravessa a meia-noite e plantão de 24h exato:** 19h–07h e 07h–07h são aceitos; 24h01 não. Teste: Task 9, casos de borda de duração.
3. **Mesmo profissional em duas unidades com o mesmo horário:** recusado com `shift_overlap` nomeando o turno em conflito, e o turno cancelado não conta como conflito. Teste: Task 9.
4. **Corpo JSON com chave extra no `POST /professionals/me`** (ex.: `cns`, `user_id`, ou a chave `professional` que o ParamsWrapper do Rails injetaria): 422 `field_not_editable`, nada muda. Teste: Task 4.
5. **CNS repetido entre dois perfis (cifra determinística):** 409 `cns_taken`, não 500. Teste: Task 3.

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `db/city_migrate/20260928000001_create_professionals.rb` | três tabelas, extensão, triggers | 1 |
| `db/city_triggers.sql` | `professional_links_guard`, `professional_shifts_guard` | 1 |
| `db/city_schema.rb` | dump à mão | 1 |
| `app/models/professional.rb`, `professional_link.rb`, `professional_shift.rb` | modelos | 1, 3 |
| `app/models/user.rb` | `has_one :professional` | 1 |
| `app/services/professionals/cns.rb` | validação e gerador de CNS | 2 |
| `app/services/professionals/cbo.rb` + `config/professionals/cbo_saude.yml` | lista CBO | 6 |
| `app/services/professionals/status.rb` | `missing_profile`/`missing_link`/`ok` por usuário | 4 |
| `app/commands/professionals/*.rb` | `Create`, `UpdateProfile`, `OpenLink`, `EndLink`, `ScheduleShift`, `CancelShift` | 3, 7, 9 |
| `app/services/professionals/clinical_authorization.rb` | regra clínica | 11 |
| `app/controllers/professionals_controller.rb` | perfil, `me`, `pending`, `cbo` | 4, 6 |
| `app/controllers/professional_links_controller.rb` | abrir/encerrar vínculo | 8 |
| `app/controllers/professional_shifts_controller.rb` | listar/lançar/cancelar turno | 10 |
| `app/controllers/concerns/professional_rendering.rb` | JSON de perfil, vínculo e turno | 4, 8, 10 |
| `app/controllers/concerns/attendance_access.rb`, `attendances_controller.rb`, `setup_controller.rb` | 403 nomeado, `called_by_name`, `professional_status` | 12 |
| `app/commands/attendances/call.rb`, `call_next.rb`, `close.rb` | regra clínica | 11 |
| `config/routes.rb`, `config/initializers/domain_events.rb` | rotas e eventos | 4, 6, 8, 10 |
| `lib/professional_crew.rb`, `db/seeds.rb` | semente de dev | 13 |
| `spec/invariants/professional_invariants_spec.rb` | invariantes + mutação | 14 |

---

## Fatia 1 — F-10.1 Perfil

### Task 1: Migração, triggers e modelos mínimos

**Files:**
- Create: `db/city_migrate/20260928000001_create_professionals.rb`
- Modify: `db/city_triggers.sql` (acrescentar ao fim), `db/city_schema.rb`
- Create: `app/models/professional.rb`, `app/models/professional_link.rb`, `app/models/professional_shift.rb`
- Modify: `app/models/user.rb`
- Test: `spec/models/professional_tables_guard_spec.rb`

**Interfaces:**
- Produces: tabelas `professionals`, `professional_links`, `professional_shifts`; `User#professional`; `Professional#links`; `ProfessionalLink#shifts`, `ProfessionalLink.active`; `ProfessionalShift.valid_shifts`.

- [ ] **Step 1: Confira o número da migração**

Run: `ls db/city_migrate | tail -2`
Expected: a última é `20260927300001_guard_domain_events.rb`. Se houver outra mais nova, use um número maior que ela.

- [ ] **Step 2: Escreva a spec de guarda (falha: tabelas não existem)**

```ruby
# spec/models/professional_tables_guard_spec.rb
require "rails_helper"

# Módulo 10 (ADR 0021): vínculo e turno só aceitam acréscimo; o banco recusa
# por SQL direto, sem passar pelo modelo.
RSpec.describe "Guardas das tabelas de profissionais" do
  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:user) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:unit) { create_unit }
  let(:professional) do
    Professional.create!(user: user, professional_name: "Helena Duarte", council: "CRM", council_state: "PR",
                         registration_number: "12345", cns: "700000000000005")
  end
  let(:link) do
    ProfessionalLink.create!(professional: professional, health_unit: unit, cbo_code: "225125",
                             started_at: Time.current, started_by_user: admin)
  end

  def sql(statement) = ApplicationRecord.connection.execute(statement)

  it "vínculo: DELETE recusado" do
    expect { sql("DELETE FROM professional_links WHERE id = '#{link.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /append-only/)
  end

  it "vínculo: mudar unidade, CBO ou início é recusado" do
    other = create_unit("UBS Outra")
    [ "health_unit_id = '#{other.id}'", "cbo_code = '225124'", "started_at = now() - interval '1 day'" ].each do |set|
      expect { sql("UPDATE professional_links SET #{set} WHERE id = '#{link.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, /only the ending columns/), set
    end
  end

  it "vínculo: encerra uma vez; a segunda é recusada" do
    sql("UPDATE professional_links SET ended_at = now(), ended_by_user_id = '#{admin.id}' WHERE id = '#{link.id}'")
    expect { sql("UPDATE professional_links SET ended_at = now() WHERE id = '#{link.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /already ended/)
  end

  it "vínculo: dois ativos iguais é recusado pelo índice parcial; encerrado não conta" do
    link
    expect do
      ProfessionalLink.create!(professional: professional, health_unit: unit, cbo_code: "225125",
                               started_at: Time.current, started_by_user: admin)
    end.to raise_error(ActiveRecord::RecordNotUnique)
  end

  describe "turnos" do
    let(:shift) do
      ProfessionalShift.create!(professional_link: link, professional: professional, created_by_user: admin,
                                starts_at: 1.day.from_now.change(hour: 7), ends_at: 1.day.from_now.change(hour: 13))
    end

    it "DELETE recusado" do
      expect { sql("DELETE FROM professional_shifts WHERE id = '#{shift.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    end

    it "mudar horário é recusado; cancelar uma vez passa; a segunda é recusada" do
      expect { sql("UPDATE professional_shifts SET ends_at = ends_at + interval '1 hour' WHERE id = '#{shift.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, /only the cancellation columns/)
      sql("UPDATE professional_shifts SET cancelled_at = now(), cancelled_by_user_id = '#{admin.id}', " \
          "cancel_reason = 'troca' WHERE id = '#{shift.id}'")
      expect { sql("UPDATE professional_shifts SET cancel_reason = 'outra' WHERE id = '#{shift.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, /already cancelled/)
    end

    it "sobreposição do mesmo profissional é recusada; turno cancelado não conta" do
      shift
      overlapping = { professional_link: link, professional: professional, created_by_user: admin,
                      starts_at: shift.starts_at + 1.hour, ends_at: shift.ends_at + 1.hour }
      expect { ProfessionalShift.create!(overlapping) }.to raise_error(ActiveRecord::ExclusionViolation)
      shift.update!(cancelled_at: Time.current, cancelled_by_user: admin, cancel_reason: "troca")
      expect { ProfessionalShift.create!(overlapping) }.not_to raise_error
    end

    it "mais de 24h e fim antes do início são recusados pelo CHECK" do
      start = 1.day.from_now.change(hour: 7)
      [ [ start, start + 24.hours + 1.minute ], [ start, start - 1.hour ] ].each do |starts_at, ends_at|
        expect do
          ProfessionalShift.create!(professional_link: link, professional: professional, created_by_user: admin,
                                    starts_at: starts_at, ends_at: ends_at)
        end.to raise_error(ActiveRecord::StatementInvalid, /ck_professional_shifts_window/)
      end
    end

    it "turno com professional_id diferente do vínculo é recusado" do
      other_user = staff_with("outra@cidade.gov.br", "health_professional")
      other = Professional.create!(user: other_user, professional_name: "Outra", council: "CRM", council_state: "PR",
                                   registration_number: "54321", cns: "100000000000007")
      expect do
        ProfessionalShift.create!(professional_link: link, professional: other, created_by_user: admin,
                                  starts_at: 1.day.from_now, ends_at: 1.day.from_now + 2.hours)
      end.to raise_error(ActiveRecord::StatementInvalid, /must match the link/)
    end

    it "turno em vínculo encerrado é recusado pelo banco" do
      link.update!(ended_at: Time.current, ended_by_user: admin)
      expect { shift }.to raise_error(ActiveRecord::StatementInvalid, /link is ended/)
    end
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/models/professional_tables_guard_spec.rb`
Expected: FAIL com `uninitialized constant Professional` (ou relação inexistente).

- [ ] **Step 4: Escreva a migração**

```ruby
# db/city_migrate/20260928000001_create_professionals.rb
# Profissionais (ADR 0021; spec 2026-09-27-module-10-professionals §3): perfil
# 1:1 com o usuário, vínculo com unidade e CBO, turnos com instantes. Vínculo e
# turno só aceitam acréscimo (triggers em db/city_triggers.sql, a mesma fonte
# do load_city_schema). Timestamps são `timestamp` sem fuso em UTC, como o
# resto do banco — por isso a EXCLUDE usa tsrange, não tstzrange.
class CreateProfessionals < ActiveRecord::Migration[8.1]
  def up
    enable_extension "btree_gist"

    create_table :professionals, id: :uuid do |t|
      t.references :user, type: :uuid, null: false, foreign_key: true, index: { unique: true }
      t.string :professional_name, null: false
      t.string :council, null: false
      t.string :council_state, null: false
      t.string :registration_number, null: false
      t.string :cns, null: false
      t.string :phone
      t.string :contact_email
      t.timestamps
    end
    add_index :professionals, %i[council council_state registration_number], unique: true,
              name: "idx_professionals_registration"
    add_index :professionals, :cns, unique: true, name: "idx_professionals_cns"
    add_check_constraint :professionals, "length(btrim(professional_name::text)) > 0", name: "ck_professionals_name"
    add_check_constraint :professionals, "council_state::text ~ '^[A-Z]{2}$'::text", name: "ck_professionals_council_state"
    add_check_constraint :professionals, "registration_number::text ~ '^[0-9]{1,10}$'::text",
                         name: "ck_professionals_registration_number"

    create_table :professional_links, id: :uuid do |t|
      t.references :professional, type: :uuid, null: false, foreign_key: true, index: true
      t.references :health_unit, type: :uuid, null: false, foreign_key: true, index: true
      t.string :cbo_code, null: false
      t.datetime :started_at, null: false
      t.references :started_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: true
      t.datetime :ended_at
      t.references :ended_by_user, type: :uuid, foreign_key: { to_table: :users }, index: true
      t.datetime :created_at, null: false
    end
    add_index :professional_links, %i[professional_id health_unit_id cbo_code], unique: true,
              where: "(ended_at IS NULL)", name: "idx_professional_links_one_active"
    add_check_constraint :professional_links, "cbo_code::text ~ '^[0-9]{6}$'::text", name: "ck_professional_links_cbo_code"
    add_check_constraint :professional_links, "(ended_at IS NULL) = (ended_by_user_id IS NULL)",
                         name: "ck_professional_links_ending"
    add_check_constraint :professional_links, "ended_at IS NULL OR ended_at >= started_at",
                         name: "ck_professional_links_order"

    create_table :professional_shifts, id: :uuid do |t|
      t.references :professional_link, type: :uuid, null: false, foreign_key: true, index: true
      t.references :professional, type: :uuid, null: false, foreign_key: true, index: true
      t.datetime :starts_at, null: false
      t.datetime :ends_at, null: false
      t.references :created_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: true
      t.datetime :cancelled_at
      t.references :cancelled_by_user, type: :uuid, foreign_key: { to_table: :users }, index: true
      t.string :cancel_reason
      t.datetime :created_at, null: false
    end
    add_check_constraint :professional_shifts,
                         "ends_at > starts_at AND (ends_at - starts_at) <= '24:00:00'::interval",
                         name: "ck_professional_shifts_window"
    add_check_constraint :professional_shifts,
                         "(cancelled_at IS NULL AND cancelled_by_user_id IS NULL AND cancel_reason IS NULL) OR " \
                         "(cancelled_at IS NOT NULL AND cancelled_by_user_id IS NOT NULL AND cancel_reason IS NOT NULL " \
                         "AND length(btrim(cancel_reason::text)) > 0)",
                         name: "ck_professional_shifts_cancelling"
    add_exclusion_constraint :professional_shifts, "professional_id WITH =, tsrange(starts_at, ends_at) WITH &&",
                             using: :gist, where: "(cancelled_at IS NULL)", name: "excl_professional_shifts_overlap"
    add_index :professional_shifts, %i[professional_link_id starts_at], name: "idx_professional_shifts_link_start"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    drop_table :professional_shifts
    drop_table :professional_links
    drop_table :professionals
    execute "DROP FUNCTION IF EXISTS rota_professional_link_guard()"
    execute "DROP FUNCTION IF EXISTS rota_professional_shift_guard()"
  end
end
```

- [ ] **Step 5: Acrescente os triggers ao fim de `db/city_triggers.sql`**

```sql
-- professional_links (ADR 0021; spec 2026-09-27-module-10-professionals §3.2):
-- só acréscimo, exceto encerrar UMA vez. Quem, onde, com qual CBO e desde
-- quando nunca mudam; o vínculo nunca é apagado. Sem trigger de TRUNCATE,
-- como em memberships: a limpeza das suítes/restauração usa TRUNCATE/--clean.
CREATE OR REPLACE FUNCTION rota_professional_link_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'professional_links is append-only: DELETE refused';
  END IF;
  IF OLD.ended_at IS NOT NULL THEN
    RAISE EXCEPTION 'professional_links: already ended';
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id
     OR NEW.professional_id IS DISTINCT FROM OLD.professional_id
     OR NEW.health_unit_id IS DISTINCT FROM OLD.health_unit_id
     OR NEW.cbo_code IS DISTINCT FROM OLD.cbo_code
     OR NEW.started_at IS DISTINCT FROM OLD.started_at
     OR NEW.started_by_user_id IS DISTINCT FROM OLD.started_by_user_id
     OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'professional_links: only the ending columns may change';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- professional_shifts (§3.3): só acréscimo, exceto cancelar UMA vez. No
-- INSERT, o profissional tem de ser o do vínculo (a coluna existe só para a
-- EXCLUDE) e o vínculo tem de estar ativo — "não existe turno em vínculo
-- encerrado" fica garantido pelo banco, não só pelo comando.
CREATE OR REPLACE FUNCTION rota_professional_shift_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'INSERT' THEN
    IF NOT EXISTS (SELECT 1 FROM professional_links l
                   WHERE l.id = NEW.professional_link_id AND l.professional_id = NEW.professional_id) THEN
      RAISE EXCEPTION 'professional_shifts: professional_id must match the link';
    END IF;
    IF EXISTS (SELECT 1 FROM professional_links l
               WHERE l.id = NEW.professional_link_id AND l.ended_at IS NOT NULL) THEN
      RAISE EXCEPTION 'professional_shifts: link is ended';
    END IF;
    RETURN NEW;
  END IF;
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'professional_shifts is append-only: DELETE refused';
  END IF;
  IF OLD.cancelled_at IS NOT NULL THEN
    RAISE EXCEPTION 'professional_shifts: already cancelled';
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id
     OR NEW.professional_link_id IS DISTINCT FROM OLD.professional_link_id
     OR NEW.professional_id IS DISTINCT FROM OLD.professional_id
     OR NEW.starts_at IS DISTINCT FROM OLD.starts_at
     OR NEW.ends_at IS DISTINCT FROM OLD.ends_at
     OR NEW.created_by_user_id IS DISTINCT FROM OLD.created_by_user_id
     OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'professional_shifts: only the cancellation columns may change';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF to_regclass('public.professional_links') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS professional_links_guard ON professional_links';
    EXECUTE 'CREATE TRIGGER professional_links_guard
      BEFORE UPDATE OR DELETE ON professional_links
      FOR EACH ROW EXECUTE FUNCTION rota_professional_link_guard()';
  END IF;
  IF to_regclass('public.professional_shifts') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS professional_shifts_guard ON professional_shifts';
    EXECUTE 'CREATE TRIGGER professional_shifts_guard
      BEFORE INSERT OR UPDATE OR DELETE ON professional_shifts
      FOR EACH ROW EXECUTE FUNCTION rota_professional_shift_guard()';
  END IF;
END
$do$;
```

- [ ] **Step 6: Escreva os modelos mínimos e a associação no User**

```ruby
# app/models/professional.rb
# Perfil do profissional de saúde (ADR 0021): 1:1 com o usuário. O papel
# health_professional continua sendo o que autoriza; o perfil diz quem é e,
# pelos vínculos, onde atua. CNS cifrado determinístico (unicidade); contato
# cifrado com a chave da cidade; registro do conselho em claro (dado público).
class Professional < ApplicationRecord
  encrypts :cns, deterministic: true, key_provider: CityDeterministicKeyProvider.new
  encrypts :phone
  encrypts :contact_email

  belongs_to :user
  has_many :links, class_name: "ProfessionalLink", dependent: :restrict_with_error
end
```

```ruby
# app/models/professional_link.rb
# Vínculo do profissional com a unidade, com a ocupação (CBO) daquele lugar
# (ADR 0021). Só acréscimo: encerrar grava ended_at, nada se apaga.
class ProfessionalLink < ApplicationRecord
  belongs_to :professional
  belongs_to :health_unit
  belongs_to :started_by_user, class_name: "User"
  belongs_to :ended_by_user, class_name: "User", optional: true
  has_many :shifts, class_name: "ProfessionalShift", dependent: :restrict_with_error

  scope :active, -> { where(ended_at: nil) }

  def active? = ended_at.nil?
end
```

```ruby
# app/models/professional_shift.rb
# Turno com instantes de início e fim (ADR 0021, emenda de 2026-09-27). Só
# acréscimo: cancelar grava hora, quem e motivo. Nunca bloqueia ato clínico.
class ProfessionalShift < ApplicationRecord
  belongs_to :professional_link
  belongs_to :professional
  belongs_to :created_by_user, class_name: "User"
  belongs_to :cancelled_by_user, class_name: "User", optional: true

  scope :valid_shifts, -> { where(cancelled_at: nil) }
end
```

Em `app/models/user.rb`, logo depois de `has_many :memberships ...`:

```ruby
  has_one :professional, dependent: :restrict_with_error  # ADR 0021
```

- [ ] **Step 7: Migre um banco de dev e faça o dump à mão**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bin/rails city:migrate[curitiba]`

Em `db/city_schema.rb`:
- a versão do `define` passa para `2026_09_28_000001`;
- `enable_extension "btree_gist"` entra junto dos outros `enable_extension`, em ordem alfabética;
- as três tabelas entram em ordem alfabética, entre `outbound_messages`/`processed_events` e `protocol_*`, com colunas em ordem alfabética (é como o dump do projeto faz: confira uma tabela vizinha), índices e `t.check_constraint`;
- a exclusion constraint no bloco da tabela: `t.exclusion_constraint "professional_id WITH =, tsrange(starts_at, ends_at) WITH &&", where: "(cancelled_at IS NULL)", using: :gist, name: "excl_professional_shifts_overlap"`;
- os `add_foreign_key` novos na lista final, em ordem alfabética.

O texto exato de cada CHECK vem do banco:

```bash
docker compose exec -T db psql -U postgres -d <banco_de_curitiba> -c "select conname, pg_get_constraintdef(oid) from pg_constraint where conrelid::regclass::text like 'professional%'"
```

- [ ] **Step 8: Recarregue os bancos de teste e rode a paridade e a spec de guarda**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod10 api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/services/city_schema_spec.rb spec/models/professional_tables_guard_spec.rb
```
Expected: PASS. Se `enable_extension "btree_gist"` falhar por permissão, **pare e avise**: a extensão precisa ir para o provisionamento da cidade (risco da spec §10).

- [ ] **Step 9: Commit**

```bash
/opt/homebrew/bin/git add db/city_migrate/20260928000001_create_professionals.rb db/city_triggers.sql db/city_schema.rb app/models/professional.rb app/models/professional_link.rb app/models/professional_shift.rb app/models/user.rb spec/models/professional_tables_guard_spec.rb
/opt/homebrew/bin/git commit -m "feat: add professionals, links and shifts tables with append-only guards

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 2: Validação e gerador de CNS

**Files:**
- Create: `app/services/professionals/cns.rb`
- Test: `spec/services/professionals/cns_spec.rb`

**Interfaces:**
- Produces: `Professionals::Cns.valid?(value) → Boolean`; `Professionals::Cns.generate(seed) → String` (15 dígitos, começa com 7, válido, determinístico); `Professionals::Cns.mask(value) → "*** **** **** 1234"`.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/services/professionals/cns_spec.rb
require "rails_helper"

RSpec.describe Professionals::Cns do
  describe ".valid?" do
    %w[700000000000005 100000000000007 200123456789019 898000000000002 712345678901236 123456789012348].each do |cns|
      it("aceita #{cns}") { expect(described_class.valid?(cns)).to be(true) }
    end

    {
      "dígito verificador errado" => "712345678901237",
      "primeiro dígito fora de 1, 2, 7, 8, 9" => "312345678901236",
      "14 dígitos" => "70000000000000",
      "com letra" => "70000000000000a",
      "vazio" => "",
      "nil" => nil
    }.each do |label, cns|
      it("recusa #{label}") { expect(described_class.valid?(cns)).to be(false) }
    end
  end

  describe ".generate" do
    it "é válido, começa com 7 e é determinístico pela semente" do
      a = described_class.generate("curitiba:profissional")
      expect(a).to match(/\A7\d{14}\z/)
      expect(described_class.valid?(a)).to be(true)
      expect(described_class.generate("curitiba:profissional")).to eq(a)
      expect(described_class.generate("maringa:profissional")).not_to eq(a)
    end

    it "gera válidos para muitas sementes (cobre o caso de dígito 10)" do
      200.times { |i| expect(described_class.valid?(described_class.generate("s#{i}"))).to be(true) }
    end
  end

  it ".mask mostra só os 4 últimos" do
    expect(described_class.mask("712345678901236")).to eq("*** **** **** 1236")
    expect(described_class.mask(nil)).to be_nil
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/services/professionals/cns_spec.rb`
Expected: FAIL com `uninitialized constant Professionals::Cns`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/professionals/cns.rb
require "digest"

# Cartão Nacional de Saúde (spec 2026-09-27-module-10-professionals §3.1).
# Regra única para as duas famílias: 15 dígitos, primeiro em 1, 2, 7, 8 ou 9,
# soma ponderada (pesos 15 a 1) múltipla de 11 — o definitivo (1/2, derivado
# do PIS) é construído para fechar essa soma. Fonte única do modelo e da semente.
module Professionals
  module Cns
    FORMAT = /\A[12789]\d{14}\z/

    module_function

    def valid?(value)
      digits = value.to_s
      return false unless digits.match?(FORMAT)

      weighted_sum(digits).zero?
    end

    # Provisório (prefixo 7) determinístico pela semente, para a semente de dev
    # não duplicar a cada reseed. Quando o dígito verificador daria 10, troca
    # o contador e tenta de novo.
    def generate(seed)
      (0..).each do |attempt|
        body = "7" + Digest::SHA256.hexdigest("#{seed}:#{attempt}").scan(/\d/).join[0, 13].ljust(13, "0")
        check = (11 - (weighted_sum(body + "0"))) % 11
        return body + check.to_s if check < 10
      end
    end

    def mask(value)
      return nil if value.blank?

      "*** **** **** #{value.to_s[-4..]}"
    end

    def weighted_sum(digits)
      digits.chars.each_with_index.sum { |c, i| c.to_i * (15 - i) } % 11
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/services/professionals/cns_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/services/professionals/cns.rb spec/services/professionals/cns_spec.rb
/opt/homebrew/bin/git commit -m "feat: validate and generate CNS numbers for professionals

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 3: Validações do perfil e comandos Create e UpdateProfile

**Files:**
- Modify: `app/models/professional.rb`
- Create: `app/commands/professionals/create.rb`, `app/commands/professionals/update_profile.rb`
- Modify: `config/initializers/domain_events.rb`
- Test: `spec/models/professional_spec.rb`, `spec/commands/professionals/create_spec.rb`, `spec/commands/professionals/update_profile_spec.rb`

**Interfaces:**
- Consumes: `Professionals::Cns.valid?` (Task 2).
- Produces:
  - `Professional::FIELDS`, `Professional::SELF_EDITABLE`, `Professional::COUNCILS`, `Professional::UFS`, `Professional#cns_masked`;
  - `Professionals::Create.call(user_id:, attrs:, by:) → Result` (ok: `professional:`; fail: `:not_found`, `:missing_role`, `:already_exists`, `:invalid` com `details: { fields: [...] }`, `:cns_taken`, `:registration_taken`);
  - `Professionals::UpdateProfile.call(professional:, attrs:, by:, allowed: Professional::FIELDS) → Result` (fail: `:field_not_editable` com `details: { fields: [...] }`, `:invalid`, `:cns_taken`, `:registration_taken`).

- [ ] **Step 1: Escreva as specs do modelo**

```ruby
# spec/models/professional_spec.rb
require "rails_helper"

RSpec.describe Professional do
  let(:user) { staff_with("medica@cidade.gov.br", "health_professional") }

  def build_professional(**overrides)
    described_class.new({ user: user, professional_name: "  Helena   Duarte ", council: "CRM", council_state: "pr",
                          registration_number: "12.345", cns: "7000 0000 0000 005" }.merge(overrides))
  end

  it "normaliza nome, UF, registro e CNS" do
    p = build_professional
    expect(p).to be_valid
    expect(p).to have_attributes(professional_name: "Helena Duarte", council_state: "PR",
                                 registration_number: "12345", cns: "700000000000005")
  end

  it "normaliza telefone e e-mail de contato; vazio vira nil" do
    p = build_professional(phone: "(41) 99876-5432", contact_email: " Helena@Clinica.org ")
    expect(p).to have_attributes(phone: "41998765432", contact_email: "helena@clinica.org")
    expect(build_professional(phone: "", contact_email: "")).to have_attributes(phone: nil, contact_email: nil)
  end

  {
    professional_name: "   ", council: "XYZ", council_state: "ZZ", registration_number: "12345678901",
    cns: "712345678901237", phone: "4199", contact_email: "sem-arroba"
  }.each do |field, value|
    it "recusa #{field} = #{value.inspect}" do
      p = build_professional(field => value)
      expect(p).not_to be_valid
      expect(p.errors.attribute_names).to include(field)
    end
  end

  it "cns_masked mostra só os 4 últimos" do
    expect(build_professional.cns_masked).to eq("*** **** **** 0005")
  end
end
```

- [ ] **Step 2: Escreva as specs dos comandos**

```ruby
# spec/commands/professionals/create_spec.rb
require "rails_helper"

RSpec.describe Professionals::Create do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:user) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:attrs) do
    { professional_name: "Helena Duarte", council: "CRM", council_state: "PR",
      registration_number: "12345", cns: "700000000000005", phone: "41998765432" }
  end

  it "cria o perfil e publica professional.created só com ids" do
    result = described_class.call(user_id: user.id, attrs: attrs, by: admin)
    expect(result).to be_ok
    professional = result.payload[:professional]
    expect(professional).to be_persisted
    event = DomainEvent.where(name: "professional.created").sole
    expect(event.payload).to eq("professional_id" => professional.id, "user_id" => user.id, "by_user_id" => admin.id)
  end

  it "usuário sem o papel ativo: missing_role" do
    plain = staff_with("viewer@cidade.gov.br", "viewer")
    expect(described_class.call(user_id: plain.id, attrs: attrs, by: admin).reason).to eq(:missing_role)
    expect(Professional.count).to eq(0)
  end

  it "papel revogado: missing_role" do
    user.memberships.sole.revoke!
    expect(described_class.call(user_id: user.id, attrs: attrs, by: admin).reason).to eq(:missing_role)
  end

  it "usuário inexistente: not_found" do
    expect(described_class.call(user_id: SecureRandom.uuid, attrs: attrs, by: admin).reason).to eq(:not_found)
  end

  it "segundo perfil do mesmo usuário: already_exists" do
    described_class.call(user_id: user.id, attrs: attrs, by: admin)
    expect(described_class.call(user_id: user.id, attrs: attrs, by: admin).reason).to eq(:already_exists)
  end

  it "campo inválido: invalid com a lista dos campos" do
    result = described_class.call(user_id: user.id, attrs: attrs.merge(cns: "123", council: "X"), by: admin)
    expect(result.reason).to eq(:invalid)
    expect(result.details[:fields]).to contain_exactly("cns", "council")
  end

  it "CNS de outro perfil: cns_taken; registro de outro perfil: registration_taken" do
    described_class.call(user_id: user.id, attrs: attrs, by: admin)
    other = staff_with("outra@cidade.gov.br", "health_professional")
    same_cns = attrs.merge(registration_number: "999")
    expect(described_class.call(user_id: other.id, attrs: same_cns, by: admin).reason).to eq(:cns_taken)
    same_registration = attrs.merge(cns: "100000000000007")
    expect(described_class.call(user_id: other.id, attrs: same_registration, by: admin).reason).to eq(:registration_taken)
  end
end
```

```ruby
# spec/commands/professionals/update_profile_spec.rb
require "rails_helper"

RSpec.describe Professionals::UpdateProfile do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:user) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:professional) do
    Professionals::Create.call(user_id: user.id, by: admin, attrs: {
      professional_name: "Helena Duarte", council: "CRM", council_state: "PR",
      registration_number: "12345", cns: "700000000000005"
    }).payload[:professional]
  end

  it "admin muda o conselho e o evento leva só os nomes dos campos" do
    result = described_class.call(professional: professional, attrs: { "council_state" => "SC", "phone" => "41998765432" }, by: admin)
    expect(result).to be_ok
    expect(professional.reload.council_state).to eq("SC")
    event = DomainEvent.where(name: "professional.profile_updated").sole
    expect(event.payload).to eq("professional_id" => professional.id, "fields" => %w[council_state phone],
                                "by_user_id" => admin.id)
  end

  it "mesmos valores: ok, sem evento" do
    described_class.call(professional: professional, attrs: { "council_state" => "PR", "cns" => "700000000000005" }, by: admin)
    expect(DomainEvent.where(name: "professional.profile_updated").count).to eq(0)
  end

  it "com allowed restrito, chave fora da lista: field_not_editable e nada muda" do
    result = described_class.call(professional: professional, attrs: { "professional_name" => "Helena D.", "cns" => "100000000000007" },
                                  by: user, allowed: Professional::SELF_EDITABLE)
    expect(result.reason).to eq(:field_not_editable)
    expect(result.details[:fields]).to eq(%w[cns])
    expect(professional.reload.professional_name).to eq("Helena Duarte")
  end

  it "user_id nunca é editável, nem pelo admin" do
    other = staff_with("outra@cidade.gov.br", "health_professional")
    result = described_class.call(professional: professional, attrs: { "user_id" => other.id }, by: admin)
    expect(result.reason).to eq(:field_not_editable)
  end

  it "valor inválido: invalid e nada muda" do
    result = described_class.call(professional: professional, attrs: { "contact_email" => "sem-arroba" }, by: user,
                                  allowed: Professional::SELF_EDITABLE)
    expect(result.reason).to eq(:invalid)
    expect(professional.reload.contact_email).to be_nil
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/models/professional_spec.rb spec/commands/professionals`
Expected: FAIL (validações e comandos inexistentes).

- [ ] **Step 4: Complete o modelo**

```ruby
# app/models/professional.rb
# Perfil do profissional de saúde (ADR 0021): 1:1 com o usuário. O papel
# health_professional continua sendo o que autoriza; o perfil diz quem é e,
# pelos vínculos, onde atua. CNS cifrado determinístico (unicidade); contato
# cifrado com a chave da cidade; registro do conselho em claro (dado público).
class Professional < ApplicationRecord
  COUNCILS = %w[CRM COREN CRO CRF CRP CREFITO CRN CRFa CRESS CRBM CREF CRMV].freeze
  UFS = %w[AC AL AP AM BA CE DF ES GO MA MT MS MG PA PB PR PE PI RJ RN RS RO RR SC SP SE TO].freeze
  FIELDS = %w[professional_name council council_state registration_number cns phone contact_email].freeze
  # O profissional edita só nome e contato (emenda ao ADR 0021, 2026-09-27):
  # conselho, registro e CNS a prefeitura confere.
  SELF_EDITABLE = %w[professional_name phone contact_email].freeze

  encrypts :cns, deterministic: true, key_provider: CityDeterministicKeyProvider.new
  encrypts :phone
  encrypts :contact_email

  belongs_to :user
  has_many :links, class_name: "ProfessionalLink", dependent: :restrict_with_error

  normalizes :professional_name, with: ->(v) { v.to_s.squish }
  normalizes :council_state, with: ->(v) { v.to_s.strip.upcase }
  normalizes :registration_number, :cns, with: ->(v) { v.to_s.gsub(/\D/, "") }
  normalizes :phone, with: ->(v) { v.to_s.gsub(/\D/, "").presence }, apply_to_nil: true
  normalizes :contact_email, with: ->(v) { v.to_s.strip.downcase.presence }, apply_to_nil: true

  validates :professional_name, presence: true
  validates :council, inclusion: { in: COUNCILS }
  validates :council_state, inclusion: { in: UFS }
  validates :registration_number, format: { with: /\A\d{1,10}\z/ }
  validates :phone, format: { with: /\A\d{10,11}\z/ }, allow_nil: true
  validates :contact_email, format: { with: URI::MailTo::EMAIL_REGEXP }, allow_nil: true
  validate { errors.add(:cns, :invalid) unless Professionals::Cns.valid?(cns) }

  def cns_masked = Professionals::Cns.mask(cns)
end
```

- [ ] **Step 5: Implemente os comandos**

```ruby
# app/commands/professionals/create.rb
# Cria o perfil (ADR 0021): só para usuário ativo com o papel
# health_professional ativo, um por usuário. Quem chama já é municipal_admin.
module Professionals
  class Create
    UNIQUE_REASONS = {
      "index_professionals_on_user_id" => :already_exists,
      "idx_professionals_cns" => :cns_taken,
      "idx_professionals_registration" => :registration_taken
    }.freeze

    def self.call(user_id:, attrs:, by:)
      user = User.find_by(id: user_id)
      return Result.fail(:not_found) unless user
      return Result.fail(:missing_role) unless user.active? && user.has_role?("health_professional")
      return Result.fail(:already_exists) if Professional.exists?(user_id: user.id)

      professional = Professional.new(attrs.to_h.stringify_keys.slice(*Professional::FIELDS).merge("user" => user))
      unless professional.valid?
        return Result.fail(:invalid, details: { fields: professional.errors.attribute_names.map(&:to_s).sort })
      end

      ApplicationRecord.transaction do
        professional.save!
        DomainEvents.publish("professional.created", professional_id: professional.id, user_id: user.id, by_user_id: by.id)
      end
      Result.ok(professional: professional)
    rescue ActiveRecord::RecordNotUnique => e
      Result.fail(unique_reason(e))
    end

    def self.unique_reason(error)
      UNIQUE_REASONS.find { |index, _| error.message.include?(index) }&.last || :invalid
    end
  end
end
```

```ruby
# app/commands/professionals/update_profile.rb
# Edita o perfil. O admin passa a lista inteira (FIELDS); o próprio
# profissional passa SELF_EDITABLE (emenda ao ADR 0021). Chave fora da lista é
# recusada, não ignorada: a tela precisa saber que o campo não foi gravado.
module Professionals
  class UpdateProfile
    def self.call(professional:, attrs:, by:, allowed: Professional::FIELDS)
      attrs = attrs.to_h.stringify_keys
      extra = attrs.keys - allowed
      return Result.fail(:field_not_editable, details: { fields: extra.sort }) if extra.any?

      professional.assign_attributes(attrs)
      changed = professional.changed.sort
      return Result.ok(professional: professional) if changed.empty?

      unless professional.valid?
        fields = professional.errors.attribute_names.map(&:to_s).sort
        professional.restore_attributes
        return Result.fail(:invalid, details: { fields: fields })
      end

      ApplicationRecord.transaction do
        professional.save!
        DomainEvents.publish("professional.profile_updated", professional_id: professional.id, fields: changed,
                                                              by_user_id: by.id)
      end
      Result.ok(professional: professional)
    rescue ActiveRecord::RecordNotUnique => e
      professional.restore_attributes
      Result.fail(Create.unique_reason(e))
    end
  end
end
```

- [ ] **Step 6: Declare os eventos do módulo**

Ao fim do bloco de binds em `config/initializers/domain_events.rb`, antes do `end`:

```ruby
  # Profissionais (ADR 0021; spec 2026-09-27): trilha, só ids; sem consumidor,
  # de propósito.
  DomainEvents.bind "professional.created", to: []
  DomainEvents.bind "professional.profile_updated", to: []
  DomainEvents.bind "professional.linked", to: []
  DomainEvents.bind "professional.unlinked", to: []
  DomainEvents.bind "professional.shift_scheduled", to: []
  DomainEvents.bind "professional.shift_cancelled", to: []
```

- [ ] **Step 7: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/models/professional_spec.rb spec/commands/professionals spec/models/professional_tables_guard_spec.rb`
Expected: PASS. Se "mesmos valores: ok, sem evento" falhar porque o `cns` cifrado aparece em `changed`, compare pelo valor decifrado: `changed = professional.changes.reject { |_, (a, b)| a == b }.keys.sort`.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add app/models/professional.rb app/commands/professionals config/initializers/domain_events.rb spec/models/professional_spec.rb spec/commands/professionals
/opt/homebrew/bin/git commit -m "feat: create and edit professional profiles

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 4: Rotas de perfil, `me` e `pending`

**Files:**
- Create: `app/services/professionals/status.rb`, `app/controllers/professionals_controller.rb`, `app/controllers/concerns/professional_rendering.rb`
- Modify: `config/routes.rb`
- Test: `spec/services/professionals/status_spec.rb`, `spec/requests/professionals_spec.rb`

**Interfaces:**
- Consumes: `Professionals::Create`, `Professionals::UpdateProfile` (Task 3).
- Produces:
  - `Professionals::Status.for_users(user_ids) → Hash{user_id => "missing_profile" | "missing_link" | "ok"}`;
  - `ProfessionalRendering#profile_json(p, full:)`, `#link_json(l)`, `#shift_json(s)`, `#scalar_body(keys = nil)`;
  - rotas `GET/POST /professionals`, `GET /professionals/pending`, `GET/POST /professionals/me`, `GET/POST /professionals/:id`.

- [ ] **Step 1: Escreva a spec do Status**

```ruby
# spec/services/professionals/status_spec.rb
require "rails_helper"

RSpec.describe Professionals::Status do
  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }

  def profile_for(user, cns)
    Professional.create!(user: user, professional_name: "P", council: "CRM", council_state: "PR",
                         registration_number: cns[-5..], cns: cns)
  end

  it "sem perfil, perfil sem vínculo ativo, e com vínculo ativo" do
    none = staff_with("a@c.gov.br", "health_professional")
    unlinked = staff_with("b@c.gov.br", "health_professional")
    linked = staff_with("c@c.gov.br", "health_professional")
    profile_for(unlinked, "700000000000005")
    ended = ProfessionalLink.create!(professional: profile_for(linked, "100000000000007"), health_unit: create_unit,
                                     cbo_code: "225125", started_at: Time.current, started_by_user: admin)
    expect(described_class.for_users([ none.id, unlinked.id, linked.id ]))
      .to eq(none.id => "missing_profile", unlinked.id => "missing_link", linked.id => "ok")

    ended.update!(ended_at: Time.current, ended_by_user: admin)
    expect(described_class.for_users([ linked.id ])).to eq(linked.id => "missing_link")
  end
end
```

- [ ] **Step 2: Escreva a spec de request**

```ruby
# spec/requests/professionals_spec.rb
require "rails_helper"

RSpec.describe "Professionals", type: :request do
  def json = JSON.parse(response.body)

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:attrs) do
    { professional_name: "Helena Duarte", council: "CRM", council_state: "PR",
      registration_number: "12345", cns: "700000000000005" }
  end

  def create_profile!(user = doctor)
    Current.set(city: TEST_CITY_A) { Professionals::Create.call(user_id: user.id, attrs: attrs, by: admin) }
           .payload[:professional]
  end

  describe "POST /professionals" do
    it "admin cria; resposta com CNS completo" do
      sign_in_as(admin)
      json_post "/professionals", attrs.merge(user_id: doctor.id)
      expect(response).to have_http_status(:created)
      expect(json["professional"]).to include("user_id" => doctor.id, "cns" => "700000000000005",
                                              "cns_masked" => "*** **** **** 0005")
    end

    it "usuário sem o papel: 422 missing_role" do
      sign_in_as(admin)
      json_post "/professionals", attrs.merge(user_id: admin.id)
      expect(response).to have_http_status(:unprocessable_entity)
      expect(json["error"]).to eq("missing_role")
    end

    it "campo inválido: 422 invalid com fields" do
      sign_in_as(admin)
      json_post "/professionals", attrs.merge(user_id: doctor.id, cns: "1")
      expect(json).to include("error" => "invalid", "fields" => [ "cns" ])
    end

    it "valor não escalar: 422 invalid, nada criado" do
      sign_in_as(admin)
      json_post "/professionals", attrs.merge(user_id: doctor.id, professional_name: { "x" => 1 })
      expect(response).to have_http_status(:unprocessable_entity)
      expect(Professional.count).to eq(0)
    end
  end

  describe "GET /professionals e /professionals/:id" do
    it "a lista mascara o CNS e não traz contato; a ficha traz tudo" do
      p = create_profile!
      sign_in_as(admin)
      get "/professionals"
      row = json["professionals"].sole
      expect(row).to include("id" => p.id, "cns_masked" => "*** **** **** 0005", "links" => [])
      expect(row).not_to include("cns", "phone", "contact_email")

      get "/professionals/#{p.id}"
      expect(json["professional"]).to include("cns" => "700000000000005", "phone" => nil)
    end

    it "id inexistente: 404" do
      sign_in_as(admin)
      get "/professionals/#{SecureRandom.uuid}"
      expect(response).to have_http_status(:not_found)
    end
  end

  describe "POST /professionals/:id" do
    it "admin edita; user_id é field_not_editable" do
      p = create_profile!
      sign_in_as(admin)
      json_post "/professionals/#{p.id}", council_state: "SC"
      expect(response).to have_http_status(:ok)
      expect(p.reload.council_state).to eq("SC")

      json_post "/professionals/#{p.id}", user_id: admin.id
      expect(json).to include("error" => "field_not_editable", "fields" => [ "user_id" ])
    end
  end

  describe "GET /professionals/pending" do
    it "lista quem tem o papel sem perfil ou sem vínculo" do
      create_profile!
      novato = staff_with("novato@cidade.gov.br", "health_professional")
      sign_in_as(admin)
      get "/professionals/pending"
      expect(json["users"]).to contain_exactly(
        { "user_id" => doctor.id, "email_address" => doctor.email_address, "status" => "missing_link" },
        { "user_id" => novato.id, "email_address" => novato.email_address, "status" => "missing_profile" }
      )
    end
  end

  describe "me" do
    it "sem perfil: 404 no_profile" do
      sign_in_as(doctor)
      get "/professionals/me"
      expect(response).to have_http_status(:not_found)
      expect(json["error"]).to eq("no_profile")
    end

    it "lê o próprio perfil completo, com vínculos e turnos" do
      create_profile!
      sign_in_as(doctor)
      get "/professionals/me"
      expect(json["professional"]).to include("cns" => "700000000000005")
      expect(json).to include("links" => [], "shifts" => [])
    end

    it "edita nome e contato" do
      p = create_profile!
      sign_in_as(doctor)
      json_post "/professionals/me", professional_name: "Helena D. Moreira", contact_email: "helena@ubs.org"
      expect(response).to have_http_status(:ok)
      expect(p.reload).to have_attributes(professional_name: "Helena D. Moreira", contact_email: "helena@ubs.org")
    end

    it "chave fora de nome/contato: 422 field_not_editable, nada muda" do
      p = create_profile!
      sign_in_as(doctor)
      json_post "/professionals/me", professional_name: "Outro", cns: "100000000000007", user_id: admin.id
      expect(response).to have_http_status(:unprocessable_entity)
      expect(json).to include("error" => "field_not_editable", "fields" => %w[cns user_id])
      expect(p.reload.professional_name).to eq("Helena Duarte")
    end
  end

  describe "só o municipal_admin usa as rotas de admin" do
    (Membership::ROLES - %w[municipal_admin]).each do |role|
      it "#{role}: 403 em todas" do
        p = create_profile!
        sign_in_as(staff_with("#{role}@cidade.gov.br", role))
        get "/professionals"
        expect(response).to have_http_status(:forbidden)
        get "/professionals/pending"
        expect(response).to have_http_status(:forbidden)
        get "/professionals/#{p.id}"
        expect(response).to have_http_status(:forbidden)
        json_post "/professionals", attrs.merge(user_id: doctor.id)
        expect(response).to have_http_status(:forbidden)
        json_post "/professionals/#{p.id}", council_state: "SC"
        expect(response).to have_http_status(:forbidden)
        expect(p.reload.council_state).to eq("PR")
      end
    end
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/services/professionals/status_spec.rb spec/requests/professionals_spec.rb`
Expected: FAIL (rotas e classes inexistentes).

- [ ] **Step 4: Implemente o Status**

```ruby
# app/services/professionals/status.rb
# Situação do cadastro de quem tem o papel health_professional (spec §4.4):
# a tela Equipe e o painel de pendência dizem ao admin quem ainda não chama.
module Professionals
  module Status
    module_function

    def for_users(user_ids)
      ids = user_ids.map(&:to_s)
      with_profile = Professional.where(user_id: ids).pluck(:user_id).to_set
      linked = ProfessionalLink.active.joins(:professional).where(professionals: { user_id: ids })
                               .distinct.pluck("professionals.user_id").to_set
      ids.index_with do |id|
        next "missing_profile" unless with_profile.include?(id)

        linked.include?(id) ? "ok" : "missing_link"
      end
    end
  end
end
```

- [ ] **Step 5: Implemente a renderização e o controller**

```ruby
# app/controllers/concerns/professional_rendering.rb
# JSON do módulo de profissionais (spec §4.1). A lista mascara o CNS e deixa o
# contato de fora; só a ficha (admin) e o "me" trazem tudo.
module ProfessionalRendering
  extend ActiveSupport::Concern

  private

  def profile_json(p, full:)
    json = {
      id: p.id, user_id: p.user_id, email_address: p.user.email_address,
      professional_name: p.professional_name, council: p.council, council_state: p.council_state,
      registration_number: p.registration_number, cns_masked: p.cns_masked
    }
    json.merge!(cns: p.cns, phone: p.phone, contact_email: p.contact_email) if full
    json
  end

  def link_json(l)
    {
      id: l.id, health_unit_id: l.health_unit_id, unit_name: l.health_unit.name, cbo_code: l.cbo_code,
      cbo_title: Professionals::Cbo.find(l.cbo_code)&.title, started_at: l.started_at.iso8601,
      started_by: l.started_by_user.email_address, ended_at: l.ended_at&.iso8601,
      ended_by: l.ended_by_user&.email_address
    }
  end

  def shift_json(s)
    {
      id: s.id, professional_link_id: s.professional_link_id, unit_name: s.professional_link.health_unit.name,
      starts_at: s.starts_at.iso8601, ends_at: s.ends_at.iso8601, cancelled_at: s.cancelled_at&.iso8601,
      cancel_reason: s.cancel_reason
    }
  end

  # Corpo JSON como veio, sem o ParamsWrapper (desligado nos controllers do
  # módulo). Valor não escalar vira nil → recusado; `keys` restringe.
  def scalar_body(keys = nil)
    body = request.request_parameters.to_h
    body = body.slice(*keys) if keys
    body.transform_values { |v| v.is_a?(String) || v.is_a?(Numeric) ? v.to_s : (v.nil? ? nil : :non_scalar) }
  end
end
```

Nota: `link_json` usa `Professionals::Cbo`, que nasce na Task 6. Até lá nenhum vínculo é renderizado (a Task 4 só testa `links: []`); a Task 6 cria a classe antes de qualquer spec que renderize vínculo.

```ruby
# app/controllers/professionals_controller.rb
# Perfil do profissional (ADR 0021; spec 2026-09-27 §4.1). Admin cadastra e
# edita; o próprio profissional lê e edita nome e contato.
class ProfessionalsController < ApplicationController
  include Authentication
  include AttendanceAccess
  include ProfessionalRendering

  wrap_parameters false

  ERROR_STATUS = {
    not_found: :not_found, missing_role: :unprocessable_entity, already_exists: :conflict,
    invalid: :unprocessable_entity, cns_taken: :conflict, registration_taken: :conflict,
    field_not_editable: :unprocessable_entity
  }.freeze
  UPCOMING_DAYS = 14

  before_action :require_admin, except: %i[me update_me]
  before_action :set_professional, only: %i[show update]

  def index
    professionals = Professional.includes(:user, links: %i[health_unit started_by_user ended_by_user])
                                .order(:professional_name)
    render json: { professionals: professionals.map { |p| profile_json(p, full: false).merge(links: p.links.sort_by(&:started_at).map { |l| link_json(l) }) } }
  end

  def pending
    users = User.joins(:memberships).merge(Membership.active.where(role: "health_professional"))
                .where(deactivated_at: nil).distinct.order(:email_address).to_a
    status = Professionals::Status.for_users(users.map(&:id))
    rows = users.reject { |u| status[u.id] == "ok" }
                .map { |u| { user_id: u.id, email_address: u.email_address, status: status[u.id] } }
    render json: { users: rows }
  end

  def show
    render json: { professional: profile_json(@professional, full: true),
                   links: @professional.links.includes(:health_unit, :started_by_user, :ended_by_user)
                                       .order(:started_at).map { |l| link_json(l) } }
  end

  def create
    body = scalar_body([ "user_id", *Professional::FIELDS ])
    return render(json: { error: "invalid" }, status: :unprocessable_entity) if body.value?(:non_scalar)

    result = Professionals::Create.call(user_id: body["user_id"], attrs: body.except("user_id"), by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { professional: profile_json(result.payload[:professional], full: true) }, status: :created
  end

  def update
    update_with(@professional, allowed: Professional::FIELDS)
  end

  def me
    professional = Current.user.professional
    return render(json: { error: "no_profile" }, status: :not_found) unless professional

    window = Time.current..UPCOMING_DAYS.days.from_now
    shifts = ProfessionalShift.valid_shifts.where(professional: professional, starts_at: window)
                              .includes(professional_link: :health_unit).order(:starts_at)
    render json: {
      professional: profile_json(professional, full: true),
      links: professional.links.active.includes(:health_unit, :started_by_user, :ended_by_user)
                         .order(:started_at).map { |l| link_json(l) },
      shifts: shifts.map { |s| shift_json(s) }
    }
  end

  def update_me
    professional = Current.user.professional
    return render(json: { error: "no_profile" }, status: :not_found) unless professional

    update_with(professional, allowed: Professional::SELF_EDITABLE)
  end

  private

  def set_professional
    @professional = Professional.find_by(id: params[:id])
    render json: { error: "not_found" }, status: :not_found unless @professional
  end

  def update_with(professional, allowed:)
    body = scalar_body
    return render(json: { error: "invalid" }, status: :unprocessable_entity) if body.value?(:non_scalar)

    result = Professionals::UpdateProfile.call(professional: professional, attrs: body, by: Current.user, allowed: allowed)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { professional: profile_json(result.payload[:professional], full: true) }
  end
end
```

Nota: `render_failure` (de `AttendanceAccess`) já achata `details` no JSON, então `{ error: "invalid", fields: [...] }` sai sem código extra.

- [ ] **Step 6: Rotas**

Em `config/routes.rb`, logo depois do `scope "/attendance" do ... end`:

```ruby
  # Profissionais (ADR 0021; spec 2026-09-27-module-10-professionals §4.1).
  # Prefixo único: uma entrada só no proxy de dev do dashboard. As rotas
  # literais (me, pending, cbo, links, shifts) PRECISAM vir antes de `:id`.
  scope "/professionals" do
    get  "",        to: "professionals#index"
    post "",        to: "professionals#create"
    get  "pending", to: "professionals#pending"
    get  "me",      to: "professionals#me"
    post "me",      to: "professionals#update_me"
    get  ":id",     to: "professionals#show"
    post ":id",     to: "professionals#update"
  end
```

- [ ] **Step 7: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/services/professionals/status_spec.rb spec/requests/professionals_spec.rb`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add app/services/professionals/status.rb app/controllers/professionals_controller.rb app/controllers/concerns/professional_rendering.rb config/routes.rb spec/services/professionals/status_spec.rb spec/requests/professionals_spec.rb
/opt/homebrew/bin/git commit -m "feat: add professional profile routes with self-service name and contact

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 5: Revisão da fatia 1

- [ ] **Step 1:** Rode tudo do módulo e a regressão direta:

```bash
docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/models/professional_tables_guard_spec.rb spec/models/professional_spec.rb spec/services/professionals spec/commands/professionals spec/requests/professionals_spec.rb spec/services/city_schema_spec.rb spec/initializers/domain_events_bindings_spec.rb
```
Expected: PASS.

- [ ] **Step 2:** Revisor (subagente) lê o diff `origin/main..HEAD` contra a spec §3.1, §3.5 e §4.1 (perfil). Corrija o que ele achar antes da fatia 2.

---

## Fatia 2 — F-10.2 + F-10.3 Vínculo com CBO

### Task 6: Lista CBO

**Files:**
- Create: `config/professionals/cbo_saude.yml`, `app/services/professionals/cbo.rb`
- Modify: `app/controllers/professionals_controller.rb`, `config/routes.rb`
- Test: `spec/services/professionals/cbo_spec.rb`, `spec/requests/professionals_cbo_spec.rb`

**Interfaces:**
- Produces: `Professionals::Cbo.all`, `.active`, `.find(code) → Entry | nil`; `Entry` com `code`, `title`, `council` (String | nil), `deprecated` (Boolean); rota `GET /professionals/cbo`.

- [ ] **Step 1: Escreva as specs**

```ruby
# spec/services/professionals/cbo_spec.rb
require "rails_helper"

RSpec.describe Professionals::Cbo do
  it "toda entrada tem código de 6 dígitos único, título e conselho conhecido ou nulo" do
    codes = described_class.all.map(&:code)
    expect(codes).to all(match(/\A\d{6}\z/))
    expect(codes.uniq.size).to eq(codes.size)
    expect(described_class.all.map(&:title)).to all(be_present)
    expect(described_class.all.map(&:council).compact - Professional::COUNCILS).to be_empty
  end

  it "acha pelo código, com o conselho esperado" do
    expect(described_class.find("225125")).to have_attributes(title: "Médico clínico", council: "CRM")
    expect(described_class.find("515105").council).to be_nil
    expect(described_class.find("999999")).to be_nil
  end

  it "active deixa de fora os deprecated" do
    expect(described_class.active).to all(have_attributes(deprecated: false))
  end
end
```

```ruby
# spec/requests/professionals_cbo_spec.rb
require "rails_helper"

RSpec.describe "GET /professionals/cbo", type: :request do
  it "admin recebe a lista vigente" do
    sign_in_as(staff_with("admin@cidade.gov.br", "municipal_admin"))
    get "/professionals/cbo"
    codes = JSON.parse(response.body)["cbo"].map { |e| e["code"] }
    expect(codes).to include("225125", "223505", "322205", "515105")
  end

  it "outro papel: 403" do
    sign_in_as(staff_with("medica@cidade.gov.br", "health_professional"))
    get "/professionals/cbo"
    expect(response).to have_http_status(:forbidden)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/services/professionals/cbo_spec.rb spec/requests/professionals_cbo_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Crie a lista**

```yaml
# config/professionals/cbo_saude.yml
# Ocupações de saúde (CBO, Ministério do Trabalho) usadas no vínculo do
# profissional com a unidade (ADR 0021). Igual para todas as cidades; a
# prefeitura não edita. Código nunca sai daqui: em desuso, `deprecated: true`
# (continua válido nos vínculos existentes, some da lista de vínculo novo).
# `council`: conselho exigido do perfil para abrir vínculo com este CBO
# (null = qualquer perfil).
- { code: "225125", title: "Médico clínico", council: CRM }
- { code: "225124", title: "Médico pediatra", council: CRM }
- { code: "225142", title: "Médico da estratégia de saúde da família", council: CRM }
- { code: "225170", title: "Médico generalista", council: CRM }
- { code: "225250", title: "Médico ginecologista e obstetra", council: CRM }
- { code: "225133", title: "Médico psiquiatra", council: CRM }
- { code: "225120", title: "Médico cardiologista", council: CRM }
- { code: "225225", title: "Médico cirurgião geral", council: CRM }
- { code: "225150", title: "Médico em medicina intensiva", council: CRM }
- { code: "225112", title: "Médico neurologista", council: CRM }
- { code: "223505", title: "Enfermeiro", council: COREN }
- { code: "223565", title: "Enfermeiro da estratégia de saúde da família", council: COREN }
- { code: "322205", title: "Técnico de enfermagem", council: COREN }
- { code: "322245", title: "Técnico de enfermagem da estratégia de saúde da família", council: COREN }
- { code: "322230", title: "Auxiliar de enfermagem", council: COREN }
- { code: "223208", title: "Cirurgião-dentista - clínico geral", council: CRO }
- { code: "223293", title: "Cirurgião-dentista da estratégia de saúde da família", council: CRO }
- { code: "322405", title: "Técnico em saúde bucal", council: CRO }
- { code: "322415", title: "Auxiliar em saúde bucal", council: CRO }
- { code: "223405", title: "Farmacêutico", council: CRF }
- { code: "251510", title: "Psicólogo clínico", council: CRP }
- { code: "223605", title: "Fisioterapeuta geral", council: CREFITO }
- { code: "223905", title: "Terapeuta ocupacional", council: CREFITO }
- { code: "223710", title: "Nutricionista", council: CRN }
- { code: "223810", title: "Fonoaudiólogo geral", council: CRFa }
- { code: "251605", title: "Assistente social", council: CRESS }
- { code: "221205", title: "Biomédico", council: CRBM }
- { code: "224140", title: "Profissional de educação física na saúde", council: CREF }
- { code: "515105", title: "Agente comunitário de saúde", council: null }
- { code: "515140", title: "Agente de combate às endemias", council: null }
```

- [ ] **Step 4: Implemente a leitura**

```ruby
# app/services/professionals/cbo.rb
# Lista CBO da saúde versionada no api (ADR 0021): carregada uma vez por
# processo. Acrescentar ocupação é um commit, sem migração.
module Professionals
  module Cbo
    Entry = Data.define(:code, :title, :council, :deprecated)
    PATH = Rails.root.join("config/professionals/cbo_saude.yml")

    class << self
      def all
        @all ||= YAML.load_file(PATH).map do |row|
          Entry.new(code: row.fetch("code").to_s, title: row.fetch("title"), council: row["council"],
                    deprecated: row.fetch("deprecated", false))
        end.freeze
      end

      def active = all.reject(&:deprecated)

      def find(code) = all.find { |e| e.code == code.to_s }
    end
  end
end
```

Em `ProfessionalsController`, acrescente a ação (fica sob `require_admin`):

```ruby
  def cbo
    render json: { cbo: Professionals::Cbo.active.map { |e| { code: e.code, title: e.title, council: e.council } } }
  end
```

Em `config/routes.rb`, dentro do `scope "/professionals"`, **antes** de `get ":id"`:

```ruby
    get  "cbo",     to: "professionals#cbo"
```

- [ ] **Step 5: Rode e veja passar; commit**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/services/professionals/cbo_spec.rb spec/requests/professionals_cbo_spec.rb spec/requests/professionals_spec.rb`
Expected: PASS.

```bash
/opt/homebrew/bin/git add config/professionals/cbo_saude.yml app/services/professionals/cbo.rb app/controllers/professionals_controller.rb config/routes.rb spec/services/professionals/cbo_spec.rb spec/requests/professionals_cbo_spec.rb
/opt/homebrew/bin/git commit -m "feat: add versioned health CBO list

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 7: Comandos OpenLink e EndLink

**Files:**
- Create: `app/commands/professionals/open_link.rb`, `app/commands/professionals/end_link.rb`
- Test: `spec/commands/professionals/open_link_spec.rb`, `spec/commands/professionals/end_link_spec.rb`

**Interfaces:**
- Consumes: `Professionals::Cbo.find` (Task 6), `HealthUnit.lock_active!`.
- Produces:
  - `Professionals::OpenLink.call(professional:, health_unit_id:, cbo_code:, by:) → Result` (ok: `link:`; fail: `:invalid_cbo`, `:council_mismatch`, `:invalid_unit`, `:already_linked`);
  - `Professionals::EndLink.call(link:, by:) → Result` (ok: `link:`, `cancelled_shift_ids: []`; fail: `:already_ended`). A Task 9 passa a cancelar turnos aqui.

- [ ] **Step 1: Escreva as specs**

```ruby
# spec/commands/professionals/open_link_spec.rb
require "rails_helper"

RSpec.describe Professionals::OpenLink do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:unit) { create_unit }
  let(:doctor) do
    Professional.create!(user: staff_with("medica@cidade.gov.br", "health_professional"), professional_name: "Helena",
                         council: "CRM", council_state: "PR", registration_number: "12345", cns: "700000000000005")
  end

  def open(**overrides)
    described_class.call(**{ professional: doctor, health_unit_id: unit.id, cbo_code: "225125", by: admin }.merge(overrides))
  end

  it "abre com início = agora e publica professional.linked só com ids" do
    travel_to(Time.zone.parse("2026-10-01 09:00")) do
      link = open.payload[:link]
      expect(link).to have_attributes(started_at: Time.current, started_by_user: admin, ended_at: nil)
      expect(DomainEvent.where(name: "professional.linked").sole.payload).to eq(
        "professional_link_id" => link.id, "professional_id" => doctor.id, "health_unit_id" => unit.id,
        "by_user_id" => admin.id
      )
    end
  end

  it "CBO fora da lista ou deprecated: invalid_cbo" do
    expect(open(cbo_code: "999999").reason).to eq(:invalid_cbo)
    deprecated = Professionals::Cbo::Entry.new(code: "225125", title: "x", council: "CRM", deprecated: true)
    allow(Professionals::Cbo).to receive(:find).with("225125").and_return(deprecated)
    expect(open.reason).to eq(:invalid_cbo)
  end

  it "CBO que exige outro conselho: council_mismatch; CBO sem conselho aceita" do
    expect(open(cbo_code: "223505").reason).to eq(:council_mismatch)
    expect(open(cbo_code: "515105")).to be_ok
  end

  it "unidade inativa ou inexistente: invalid_unit" do
    expect(open(health_unit_id: create_unit("UBS Fechada", active: false).id).reason).to eq(:invalid_unit)
    expect(open(health_unit_id: SecureRandom.uuid).reason).to eq(:invalid_unit)
  end

  it "mesmo par ativo: already_linked; outro CBO na mesma unidade passa" do
    open
    expect(open.reason).to eq(:already_linked)
    expect(open(cbo_code: "225124")).to be_ok
  end
end
```

```ruby
# spec/commands/professionals/end_link_spec.rb
require "rails_helper"

RSpec.describe Professionals::EndLink do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:doctor) do
    Professional.create!(user: staff_with("medica@cidade.gov.br", "health_professional"), professional_name: "Helena",
                         council: "CRM", council_state: "PR", registration_number: "12345", cns: "700000000000005")
  end
  let(:link) do
    Professionals::OpenLink.call(professional: doctor, health_unit_id: create_unit.id, cbo_code: "225125", by: admin)
                           .payload[:link]
  end

  it "encerra com fim = agora e publica professional.unlinked" do
    result = described_class.call(link: link, by: admin)
    expect(result).to be_ok
    expect(link.reload).to have_attributes(ended_by_user: admin)
    expect(link.ended_at).to be_within(1.second).of(Time.current)
    expect(DomainEvent.where(name: "professional.unlinked").sole.payload).to include(
      "professional_link_id" => link.id, "cancelled_shift_ids" => []
    )
  end

  it "segunda vez: already_ended" do
    described_class.call(link: link, by: admin)
    expect(described_class.call(link: link, by: admin).reason).to eq(:already_ended)
  end

  it "depois de encerrar, o mesmo par pode ser aberto de novo" do
    described_class.call(link: link, by: admin)
    again = Professionals::OpenLink.call(professional: doctor, health_unit_id: link.health_unit_id, cbo_code: "225125", by: admin)
    expect(again).to be_ok
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/commands/professionals/open_link_spec.rb spec/commands/professionals/end_link_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implemente**

```ruby
# app/commands/professionals/open_link.rb
# Abre o vínculo do profissional com a unidade (ADR 0021): CBO vigente,
# coerente com o conselho do perfil, unidade ativa, um ativo por (profissional,
# unidade, CBO). Início = agora, sem data retroativa. Abrir concede autoridade
# clínica naquela unidade (F-10.5): quem chama já passou por step-up.
module Professionals
  class OpenLink
    def self.call(professional:, health_unit_id:, cbo_code:, by:)
      entry = Cbo.find(cbo_code)
      return Result.fail(:invalid_cbo) if entry.nil? || entry.deprecated
      return Result.fail(:council_mismatch) if entry.council && entry.council != professional.council

      ApplicationRecord.transaction do
        HealthUnit.lock_active!(health_unit_id)
        if professional.links.active.exists?(health_unit_id: health_unit_id, cbo_code: entry.code)
          next Result.fail(:already_linked)
        end

        link = ProfessionalLink.create!(professional: professional, health_unit_id: health_unit_id, cbo_code: entry.code,
                                        started_at: Time.current, started_by_user: by)
        DomainEvents.publish("professional.linked", professional_link_id: link.id, professional_id: professional.id,
                                                    health_unit_id: link.health_unit_id, by_user_id: by.id)
        Result.ok(link: link)
      end
    rescue HealthUnit::Inactive
      Result.fail(:invalid_unit)
    rescue ActiveRecord::RecordNotUnique
      Result.fail(:already_linked)
    end
  end
end
```

```ruby
# app/commands/professionals/end_link.rb
# Encerra o vínculo (ADR 0021): grava o fim, nunca apaga. FOR UPDATE na linha
# do vínculo: espera a chamada em curso (ClinicalAuthorization, FOR SHARE) e o
# lançamento de turno em curso (ScheduleShift, FOR SHARE).
module Professionals
  class EndLink
    def self.call(link:, by:)
      ApplicationRecord.transaction do
        link.lock!
        next Result.fail(:already_ended) unless link.active?

        link.update!(ended_at: Time.current, ended_by_user: by)
        cancelled_ids = cancel_future_shifts(link, by)
        DomainEvents.publish("professional.unlinked", professional_link_id: link.id, professional_id: link.professional_id,
                                                      health_unit_id: link.health_unit_id,
                                                      cancelled_shift_ids: cancelled_ids, by_user_id: by.id)
        Result.ok(link: link, cancelled_shift_ids: cancelled_ids)
      end
    end

    # Fatia 3 (Task 9) cancela aqui os turnos que ainda não começaram.
    def self.cancel_future_shifts(_link, _by) = []
  end
end
```

- [ ] **Step 4: Rode e veja passar; commit**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/commands/professionals`
Expected: PASS.

```bash
/opt/homebrew/bin/git add app/commands/professionals/open_link.rb app/commands/professionals/end_link.rb spec/commands/professionals/open_link_spec.rb spec/commands/professionals/end_link_spec.rb
/opt/homebrew/bin/git commit -m "feat: open and end professional unit links with CBO and council check

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 8: Rotas de vínculo com step-up

**Files:**
- Create: `app/controllers/professional_links_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/professional_links_spec.rb`

**Interfaces:**
- Consumes: `OpenLink`, `EndLink` (Task 7), `MfaStepUp#reauthenticated_recently?`, `#require_step_up!`, `ProfessionalRendering#link_json`.
- Produces: `POST /professionals/:id/links`, `POST /professionals/links/:id/end`.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/requests/professional_links_spec.rb
require "rails_helper"

RSpec.describe "Professional links", type: :request do
  def json = JSON.parse(response.body)

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:unit) { create_unit }
  let(:doctor) do
    Professional.create!(user: staff_with("medica@cidade.gov.br", "health_professional"), professional_name: "Helena",
                         council: "CRM", council_state: "PR", registration_number: "12345", cns: "700000000000005")
  end

  def sign_in_admin!(stepped_up: true)
    session = sign_in_as(admin)
    session.update!(mfa_verified_at: Time.current) if stepped_up
  end

  it "abre com step-up: 201 com o vínculo e o título do CBO" do
    sign_in_admin!
    json_post "/professionals/#{doctor.id}/links", health_unit_id: unit.id, cbo_code: "225125"
    expect(response).to have_http_status(:created)
    expect(json["link"]).to include("unit_name" => unit.name, "cbo_code" => "225125", "cbo_title" => "Médico clínico",
                                    "ended_at" => nil)
  end

  it "sem step-up: 401 mfa_required e nada abre" do
    sign_in_admin!(stepped_up: false)
    json_post "/professionals/#{doctor.id}/links", health_unit_id: unit.id, cbo_code: "225125"
    expect(response).to have_http_status(:unauthorized)
    expect(json).to eq("error" => "mfa_required")
    expect(ProfessionalLink.count).to eq(0)
  end

  it "recusas nomeadas: invalid_cbo, council_mismatch, invalid_unit (422) e already_linked (409)" do
    sign_in_admin!
    json_post "/professionals/#{doctor.id}/links", health_unit_id: unit.id, cbo_code: "999999"
    expect(json["error"]).to eq("invalid_cbo")
    json_post "/professionals/#{doctor.id}/links", health_unit_id: unit.id, cbo_code: "223505"
    expect(json["error"]).to eq("council_mismatch")
    json_post "/professionals/#{doctor.id}/links", health_unit_id: SecureRandom.uuid, cbo_code: "225125"
    expect(json["error"]).to eq("invalid_unit")
    json_post "/professionals/#{doctor.id}/links", health_unit_id: unit.id, cbo_code: "225125"
    json_post "/professionals/#{doctor.id}/links", health_unit_id: unit.id, cbo_code: "225125"
    expect(response).to have_http_status(:conflict)
    expect(json["error"]).to eq("already_linked")
  end

  it "encerra com step-up; sem step-up é 401; segunda vez é 409 already_ended" do
    link = Current.set(city: TEST_CITY_A) do
      Professionals::OpenLink.call(professional: doctor, health_unit_id: unit.id, cbo_code: "225125", by: admin).payload[:link]
    end
    sign_in_admin!(stepped_up: false)
    json_post "/professionals/links/#{link.id}/end"
    expect(response).to have_http_status(:unauthorized)
    expect(link.reload.ended_at).to be_nil

    Session.find_by(user: admin).update!(mfa_verified_at: Time.current)
    json_post "/professionals/links/#{link.id}/end"
    expect(response).to have_http_status(:ok)
    expect(json).to include("cancelled_shift_ids" => [])
    json_post "/professionals/links/#{link.id}/end"
    expect(response).to have_http_status(:conflict)
    expect(json["error"]).to eq("already_ended")
  end

  it "perfil ou vínculo inexistente: 404" do
    sign_in_admin!
    json_post "/professionals/#{SecureRandom.uuid}/links", health_unit_id: unit.id, cbo_code: "225125"
    expect(response).to have_http_status(:not_found)
    json_post "/professionals/links/#{SecureRandom.uuid}/end"
    expect(response).to have_http_status(:not_found)
  end

  (Membership::ROLES - %w[municipal_admin]).each do |role|
    it "#{role}: 403 nas duas rotas, mesmo com step-up" do
      user = staff_with("#{role}@cidade.gov.br", role)
      sign_in_as(user).update!(mfa_verified_at: Time.current)
      json_post "/professionals/#{doctor.id}/links", health_unit_id: unit.id, cbo_code: "225125"
      expect(response).to have_http_status(:forbidden)
      json_post "/professionals/links/#{SecureRandom.uuid}/end"
      expect(response).to have_http_status(:forbidden)
    end
  end
end
```

Nota: se `Session.find_by(user: admin)` não existir nesse formato, guarde a sessão devolvida por `sign_in_as` numa variável e atualize-a.

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/requests/professional_links_spec.rb`
Expected: FAIL (rota inexistente).

- [ ] **Step 3: Implemente**

```ruby
# app/controllers/professional_links_controller.rb
# Vínculo do profissional com a unidade (ADR 0021; spec §4.1). Abrir e
# encerrar exigem municipal_admin e step-up: o vínculo é a metade da
# autorização clínica que faltava ao papel (D5).
class ProfessionalLinksController < ApplicationController
  include Authentication
  include AttendanceAccess
  include MfaStepUp
  include ProfessionalRendering

  wrap_parameters false

  ERROR_STATUS = {
    invalid_cbo: :unprocessable_entity, council_mismatch: :unprocessable_entity, invalid_unit: :unprocessable_entity,
    already_linked: :conflict, already_ended: :conflict
  }.freeze

  before_action :require_admin

  def create
    professional = Professional.find_by(id: params[:id])
    return render(json: { error: "not_found" }, status: :not_found) unless professional
    return require_step_up! unless reauthenticated_recently?

    body = scalar_body(%w[health_unit_id cbo_code])
    result = Professionals::OpenLink.call(professional: professional, health_unit_id: body["health_unit_id"],
                                          cbo_code: body["cbo_code"], by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { link: link_json(result.payload[:link]) }, status: :created
  end

  def end_link
    link = ProfessionalLink.find_by(id: params[:id])
    return render(json: { error: "not_found" }, status: :not_found) unless link
    return require_step_up! unless reauthenticated_recently?

    result = Professionals::EndLink.call(link: link, by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { link: link_json(result.payload[:link]), cancelled_shift_ids: result.payload[:cancelled_shift_ids] }
  end
end
```

Nota: `scalar_body` devolve `:non_scalar` para valor não escalar; aqui esse valor chega ao comando e vira `invalid_unit`/`invalid_cbo`, o que é a resposta certa.

Em `config/routes.rb`, dentro do `scope "/professionals"`, **antes** de `get ":id"`:

```ruby
    post "links/:id/end", to: "professional_links#end_link"
```

e **depois** de `post ":id"`:

```ruby
    post ":id/links",     to: "professional_links#create"
```

- [ ] **Step 4: Rode e veja passar; commit**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/requests/professional_links_spec.rb spec/requests/professionals_spec.rb`
Expected: PASS.

```bash
/opt/homebrew/bin/git add app/controllers/professional_links_controller.rb config/routes.rb spec/requests/professional_links_spec.rb
/opt/homebrew/bin/git commit -m "feat: add step-up protected routes to open and end professional links

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 5:** Revisor (subagente) lê o diff da fatia 2 contra a spec §3.2, §3.4 e §4.1 (vínculo) e D5/D9.

---

## Fatia 3 — F-10.4 Turnos

### Task 9: Comandos ScheduleShift e CancelShift; EndLink cancela os futuros

**Files:**
- Create: `app/commands/professionals/schedule_shift.rb`, `app/commands/professionals/cancel_shift.rb`
- Modify: `app/commands/professionals/end_link.rb`, `app/models/professional_shift.rb`
- Test: `spec/commands/professionals/schedule_shift_spec.rb`, `spec/commands/professionals/cancel_shift_spec.rb`, `spec/commands/professionals/end_link_spec.rb`

**Interfaces:**
- Produces:
  - `ProfessionalShift::LINK_ENDED_REASON = "vínculo encerrado"`, `ProfessionalShift::MAX_DURATION = 24.hours`;
  - `Professionals::ScheduleShift.call(link:, starts_at:, ends_at:, by:) → Result` (ok: `shift:`; fail: `:invalid_shift`, `:link_ended`, `:shift_overlap` com `details: { conflict: { unit_name:, starts_at:, ends_at: } }`). `starts_at`/`ends_at` aceitam `Time` ou String ISO 8601;
  - `Professionals::CancelShift.call(shift:, reason:, by:) → Result` (fail: `:reason_required`, `:reason_too_long`, `:already_cancelled`).

- [ ] **Step 1: Escreva as specs**

```ruby
# spec/commands/professionals/schedule_shift_spec.rb
require "rails_helper"

RSpec.describe Professionals::ScheduleShift do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:ubs) { create_unit("UBS Jardim") }
  let(:upa) { create_unit("UPA Centro", kind: "upa") }
  let(:doctor) do
    Professional.create!(user: staff_with("medica@cidade.gov.br", "health_professional"), professional_name: "Helena",
                         council: "CRM", council_state: "PR", registration_number: "12345", cns: "700000000000005")
  end
  let(:ubs_link) { open_link(ubs, "225125") }
  let(:upa_link) { open_link(upa, "225124") }
  let(:tomorrow) { Time.zone.tomorrow }

  def open_link(unit, cbo)
    Professionals::OpenLink.call(professional: doctor, health_unit_id: unit.id, cbo_code: cbo, by: admin).payload[:link]
  end

  def at(day, hour) = day.in_time_zone.change(hour: hour)

  def schedule(link, starts_at, ends_at)
    described_class.call(link: link, starts_at: starts_at, ends_at: ends_at, by: admin)
  end

  it "lança e publica professional.shift_scheduled só com ids" do
    shift = schedule(ubs_link, at(tomorrow, 7), at(tomorrow, 13)).payload[:shift]
    expect(shift).to have_attributes(professional_id: doctor.id, created_by_user: admin)
    expect(DomainEvent.where(name: "professional.shift_scheduled").sole.payload).to eq(
      "shift_id" => shift.id, "professional_link_id" => ubs_link.id, "professional_id" => doctor.id,
      "by_user_id" => admin.id
    )
  end

  it "aceita ISO 8601 em string" do
    expect(schedule(ubs_link, at(tomorrow, 7).iso8601, at(tomorrow, 13).iso8601)).to be_ok
  end

  it "plantão 19h–07h e plantão de 24h exatas são aceitos; 24h01 não" do
    expect(schedule(upa_link, at(tomorrow, 19), at(tomorrow + 1, 7))).to be_ok
    expect(schedule(upa_link, at(tomorrow + 3, 7), at(tomorrow + 4, 7))).to be_ok
    expect(schedule(upa_link, at(tomorrow + 6, 7), at(tomorrow + 7, 7) + 1.minute).reason).to eq(:invalid_shift)
  end

  it "fim igual ou antes do início, data ilegível e antes do início do vínculo: invalid_shift" do
    expect(schedule(ubs_link, at(tomorrow, 13), at(tomorrow, 13)).reason).to eq(:invalid_shift)
    expect(schedule(ubs_link, at(tomorrow, 13), at(tomorrow, 7)).reason).to eq(:invalid_shift)
    expect(schedule(ubs_link, "ontem", at(tomorrow, 7)).reason).to eq(:invalid_shift)
    expect(schedule(ubs_link, ubs_link.started_at - 1.hour, ubs_link.started_at + 1.hour).reason).to eq(:invalid_shift)
  end

  it "sobrepor turno do mesmo profissional em OUTRA unidade: shift_overlap nomeando o conflito" do
    schedule(ubs_link, at(tomorrow, 7), at(tomorrow, 13))
    result = schedule(upa_link, at(tomorrow, 12), at(tomorrow, 18))
    expect(result.reason).to eq(:shift_overlap)
    expect(result.details[:conflict]).to eq(unit_name: "UBS Jardim", starts_at: at(tomorrow, 7).iso8601,
                                            ends_at: at(tomorrow, 13).iso8601)
  end

  it "encostar (fim de um = início do outro) não é sobreposição" do
    schedule(ubs_link, at(tomorrow, 7), at(tomorrow, 13))
    expect(schedule(upa_link, at(tomorrow, 13), at(tomorrow, 19))).to be_ok
  end

  it "turno cancelado não conta como conflito" do
    first = schedule(ubs_link, at(tomorrow, 7), at(tomorrow, 13)).payload[:shift]
    Professionals::CancelShift.call(shift: first, reason: "troca de escala", by: admin)
    expect(schedule(upa_link, at(tomorrow, 7), at(tomorrow, 13))).to be_ok
  end

  it "vínculo encerrado: link_ended" do
    Professionals::EndLink.call(link: ubs_link, by: admin)
    expect(schedule(ubs_link, at(tomorrow, 7), at(tomorrow, 13)).reason).to eq(:link_ended)
  end
end
```

```ruby
# spec/commands/professionals/cancel_shift_spec.rb
require "rails_helper"

RSpec.describe Professionals::CancelShift do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:doctor) do
    Professional.create!(user: staff_with("medica@cidade.gov.br", "health_professional"), professional_name: "Helena",
                         council: "CRM", council_state: "PR", registration_number: "12345", cns: "700000000000005")
  end
  let(:link) do
    Professionals::OpenLink.call(professional: doctor, health_unit_id: create_unit.id, cbo_code: "225125", by: admin)
                           .payload[:link]
  end
  let(:shift) do
    day = Time.zone.tomorrow.in_time_zone
    Professionals::ScheduleShift.call(link: link, starts_at: day.change(hour: 7), ends_at: day.change(hour: 13), by: admin)
                                .payload[:shift]
  end

  it "cancela com motivo e publica professional.shift_cancelled" do
    expect(described_class.call(shift: shift, reason: "  troca de escala ", by: admin)).to be_ok
    expect(shift.reload).to have_attributes(cancel_reason: "troca de escala", cancelled_by_user: admin)
    expect(DomainEvent.where(name: "professional.shift_cancelled").sole.payload)
      .to eq("shift_id" => shift.id, "professional_id" => doctor.id, "by_user_id" => admin.id)
  end

  it "sem motivo: reason_required; motivo com mais de 200: reason_too_long" do
    expect(described_class.call(shift: shift, reason: "  ", by: admin).reason).to eq(:reason_required)
    expect(described_class.call(shift: shift, reason: "x" * 201, by: admin).reason).to eq(:reason_too_long)
  end

  it "segunda vez: already_cancelled" do
    described_class.call(shift: shift, reason: "troca", by: admin)
    expect(described_class.call(shift: shift, reason: "outra", by: admin).reason).to eq(:already_cancelled)
  end
end
```

Acrescente ao fim de `spec/commands/professionals/end_link_spec.rb`, antes do último `end`:

```ruby
  describe "turnos do vínculo" do
    def schedule(starts_at, ends_at)
      Professionals::ScheduleShift.call(link: link, starts_at: starts_at, ends_at: ends_at, by: admin).payload[:shift]
    end

    it "cancela os futuros; mantém o que já passou e o que está em curso" do
      base = Time.zone.parse("2026-10-05 10:00")
      travel_to(base - 3.days) { link }
      past = travel_to(base - 2.days) { schedule(base - 1.day, base - 1.day + 4.hours) }
      current = travel_to(base - 2.days) { schedule(base - 2.hours, base + 2.hours) }
      future = travel_to(base - 2.days) { schedule(base + 1.day, base + 1.day + 6.hours) }

      result = travel_to(base) { described_class.call(link: link, by: admin) }
      expect(result.payload[:cancelled_shift_ids]).to eq([ future.id ])
      expect(future.reload).to have_attributes(cancel_reason: ProfessionalShift::LINK_ENDED_REASON, cancelled_by_user: admin)
      expect(past.reload.cancelled_at).to be_nil
      expect(current.reload.cancelled_at).to be_nil
    end
  end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/commands/professionals`
Expected: FAIL.

- [ ] **Step 3: Constantes no modelo**

```ruby
# app/models/professional_shift.rb
# Turno com instantes de início e fim (ADR 0021, emenda de 2026-09-27). Só
# acréscimo: cancelar grava hora, quem e motivo. Nunca bloqueia ato clínico.
class ProfessionalShift < ApplicationRecord
  LINK_ENDED_REASON = "vínculo encerrado".freeze
  MAX_DURATION = 24.hours
  MAX_REASON = 200

  belongs_to :professional_link
  belongs_to :professional
  belongs_to :created_by_user, class_name: "User"
  belongs_to :cancelled_by_user, class_name: "User", optional: true

  scope :valid_shifts, -> { where(cancelled_at: nil) }
end
```

- [ ] **Step 4: Implemente os comandos**

```ruby
# app/commands/professionals/schedule_shift.rb
# Lança um turno no vínculo (ADR 0021): até 24h, dentro do vínculo, sem
# sobrepor outro turno válido do mesmo profissional em qualquer unidade (a
# EXCLUDE do banco decide; aqui só se nomeia o conflito). FOR SHARE no
# vínculo: o encerramento (FOR UPDATE) espera, ou é visto.
module Professionals
  class ScheduleShift
    def self.call(link:, starts_at:, ends_at:, by:)
      starts = parse(starts_at)
      ends = parse(ends_at)
      return Result.fail(:invalid_shift) unless starts && ends && ends > starts && ends - starts <= ProfessionalShift::MAX_DURATION

      ApplicationRecord.transaction do
        locked = ProfessionalLink.lock("FOR SHARE").find(link.id)
        next Result.fail(:link_ended) unless locked.active?
        next Result.fail(:invalid_shift) if starts < locked.started_at

        shift = ProfessionalShift.create!(professional_link: locked, professional_id: locked.professional_id,
                                          starts_at: starts, ends_at: ends, created_by_user: by)
        DomainEvents.publish("professional.shift_scheduled", shift_id: shift.id, professional_link_id: locked.id,
                                                             professional_id: locked.professional_id, by_user_id: by.id)
        Result.ok(shift: shift)
      end
    rescue ActiveRecord::ExclusionViolation
      Result.fail(:shift_overlap, details: { conflict: conflict_for(link.professional_id, starts, ends) })
    end

    def self.parse(value)
      return value.in_time_zone if value.respond_to?(:in_time_zone) && !value.is_a?(String)

      Time.zone.iso8601(value.to_s)
    rescue ArgumentError
      nil
    end

    def self.conflict_for(professional_id, starts, ends)
      shift = ProfessionalShift.valid_shifts.includes(professional_link: :health_unit)
                               .where(professional_id: professional_id)
                               .where("starts_at < ? AND ends_at > ?", ends, starts).order(:starts_at).first
      return {} unless shift

      { unit_name: shift.professional_link.health_unit.name, starts_at: shift.starts_at.iso8601,
        ends_at: shift.ends_at.iso8601 }
    end
  end
end
```

```ruby
# app/commands/professionals/cancel_shift.rb
# Cancela um turno (ADR 0021): grava hora, quem e motivo; nunca apaga.
module Professionals
  class CancelShift
    def self.call(shift:, reason:, by:)
      reason = reason.to_s.strip
      return Result.fail(:reason_required) if reason.empty?
      return Result.fail(:reason_too_long) if reason.length > ProfessionalShift::MAX_REASON

      ApplicationRecord.transaction do
        shift.lock!
        next Result.fail(:already_cancelled) if shift.cancelled_at

        shift.update!(cancelled_at: Time.current, cancelled_by_user: by, cancel_reason: reason)
        DomainEvents.publish("professional.shift_cancelled", shift_id: shift.id, professional_id: shift.professional_id,
                                                             by_user_id: by.id)
        Result.ok(shift: shift)
      end
    end
  end
end
```

Em `app/commands/professionals/end_link.rb`, troque o método provisório:

```ruby
    # Turnos que ainda não começaram (D8): "não existe turno em vínculo
    # encerrado". O passado e o em curso ficam — valiam quando começaram.
    def self.cancel_future_shifts(link, by)
      now = link.ended_at
      link.shifts.valid_shifts.where("starts_at > ?", now).lock.order(:starts_at).map do |shift|
        shift.update!(cancelled_at: now, cancelled_by_user: by, cancel_reason: ProfessionalShift::LINK_ENDED_REASON)
        shift.id
      end
    end
```

- [ ] **Step 5: Rode e veja passar; commit**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/commands/professionals spec/models/professional_tables_guard_spec.rb`
Expected: PASS.

```bash
/opt/homebrew/bin/git add app/commands/professionals app/models/professional_shift.rb spec/commands/professionals
/opt/homebrew/bin/git commit -m "feat: schedule and cancel professional shifts; ending a link cancels future shifts

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 10: Rotas de turno

**Files:**
- Create: `app/controllers/professional_shifts_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/professional_shifts_spec.rb`

**Interfaces:**
- Consumes: `ScheduleShift`, `CancelShift` (Task 9), `ProfessionalRendering#shift_json`.
- Produces: `GET /professionals/:id/shifts?from=YYYY-MM-DD&to=YYYY-MM-DD`, `POST /professionals/links/:id/shifts`, `POST /professionals/shifts/:id/cancel`.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/requests/professional_shifts_spec.rb
require "rails_helper"

RSpec.describe "Professional shifts", type: :request do
  def json = JSON.parse(response.body)

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:doctor) do
    Professional.create!(user: staff_with("medica@cidade.gov.br", "health_professional"), professional_name: "Helena",
                         council: "CRM", council_state: "PR", registration_number: "12345", cns: "700000000000005")
  end
  let(:link) do
    Current.set(city: TEST_CITY_A) do
      Professionals::OpenLink.call(professional: doctor, health_unit_id: create_unit.id, cbo_code: "225125", by: admin)
                             .payload[:link]
    end
  end
  let(:day) { Time.zone.tomorrow.in_time_zone }

  before { sign_in_as(admin) }

  it "lança (sem step-up), lista no intervalo e cancela" do
    json_post "/professionals/links/#{link.id}/shifts", starts_at: day.change(hour: 19).iso8601,
                                                        ends_at: (day + 1.day).change(hour: 7).iso8601
    expect(response).to have_http_status(:created)
    shift_id = json.dig("shift", "id")

    get "/professionals/#{doctor.id}/shifts"
    expect(json["shifts"].map { |s| s["id"] }).to eq([ shift_id ])

    json_post "/professionals/shifts/#{shift_id}/cancel", reason: "troca de escala"
    expect(response).to have_http_status(:ok)
    get "/professionals/#{doctor.id}/shifts"
    expect(json["shifts"].sole).to include("cancel_reason" => "troca de escala")
  end

  it "sobreposição: 409 shift_overlap com o conflito" do
    json_post "/professionals/links/#{link.id}/shifts", starts_at: day.change(hour: 7).iso8601, ends_at: day.change(hour: 13).iso8601
    json_post "/professionals/links/#{link.id}/shifts", starts_at: day.change(hour: 12).iso8601, ends_at: day.change(hour: 14).iso8601
    expect(response).to have_http_status(:conflict)
    expect(json).to include("error" => "shift_overlap")
    expect(json["conflict"]).to include("starts_at" => day.change(hour: 7).iso8601)
  end

  it "intervalo inválido ou maior que 62 dias: 422 invalid_range" do
    get "/professionals/#{doctor.id}/shifts", params: { from: "2026-10-01", to: "2026-12-15" }
    expect(json["error"]).to eq("invalid_range")
    get "/professionals/#{doctor.id}/shifts", params: { from: "xx" }
    expect(json["error"]).to eq("invalid_range")
  end

  it "motivo vazio: 422 reason_required; cancelar duas vezes: 409" do
    json_post "/professionals/links/#{link.id}/shifts", starts_at: day.change(hour: 7).iso8601, ends_at: day.change(hour: 13).iso8601
    id = json.dig("shift", "id")
    json_post "/professionals/shifts/#{id}/cancel", reason: ""
    expect(json["error"]).to eq("reason_required")
    json_post "/professionals/shifts/#{id}/cancel", reason: "troca"
    json_post "/professionals/shifts/#{id}/cancel", reason: "troca"
    expect(response).to have_http_status(:conflict)
  end

  (Membership::ROLES - %w[municipal_admin]).each do |role|
    it "#{role}: 403 nas três rotas" do
      sign_in_as(staff_with("#{role}@cidade.gov.br", role))
      get "/professionals/#{doctor.id}/shifts"
      expect(response).to have_http_status(:forbidden)
      json_post "/professionals/links/#{link.id}/shifts", starts_at: day.iso8601, ends_at: (day + 1.hour).iso8601
      expect(response).to have_http_status(:forbidden)
      json_post "/professionals/shifts/#{SecureRandom.uuid}/cancel", reason: "x"
      expect(response).to have_http_status(:forbidden)
      expect(ProfessionalShift.count).to eq(0)
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/requests/professional_shifts_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implemente**

```ruby
# app/controllers/professional_shifts_controller.rb
# Turnos do profissional (ADR 0021; spec §4.1). Só municipal_admin; sem
# step-up — turno não muda quem pode fazer o quê (D5).
class ProfessionalShiftsController < ApplicationController
  include Authentication
  include AttendanceAccess
  include ProfessionalRendering

  wrap_parameters false

  ERROR_STATUS = {
    invalid_shift: :unprocessable_entity, link_ended: :conflict, shift_overlap: :conflict,
    reason_required: :unprocessable_entity, reason_too_long: :unprocessable_entity, already_cancelled: :conflict
  }.freeze
  DEFAULT_DAYS = 14
  MAX_DAYS = 62

  before_action :require_admin

  def index
    professional = Professional.find_by(id: params[:id])
    return render(json: { error: "not_found" }, status: :not_found) unless professional

    range = requested_range
    return render(json: { error: "invalid_range" }, status: :unprocessable_entity) unless range

    shifts = ProfessionalShift.where(professional: professional, starts_at: range)
                              .includes(professional_link: :health_unit).order(:starts_at)
    render json: { shifts: shifts.map { |s| shift_json(s) } }
  end

  def create
    link = ProfessionalLink.find_by(id: params[:id])
    return render(json: { error: "not_found" }, status: :not_found) unless link

    body = scalar_body(%w[starts_at ends_at])
    result = Professionals::ScheduleShift.call(link: link, starts_at: body["starts_at"], ends_at: body["ends_at"],
                                               by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { shift: shift_json(result.payload[:shift]) }, status: :created
  end

  def cancel
    shift = ProfessionalShift.find_by(id: params[:id])
    return render(json: { error: "not_found" }, status: :not_found) unless shift

    result = Professionals::CancelShift.call(shift: shift, reason: scalar_body(%w[reason])["reason"], by: Current.user)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: { shift: shift_json(result.payload[:shift]) }
  end

  private

  # Dias inteiros no fuso da aplicação; padrão hoje + 14; teto de 62 dias.
  def requested_range
    from = params[:from].present? ? Date.iso8601(params[:from].to_s) : Time.zone.today
    to = params[:to].present? ? Date.iso8601(params[:to].to_s) : from + DEFAULT_DAYS
    return nil if to < from || (to - from) > MAX_DAYS

    from.in_time_zone.beginning_of_day..to.in_time_zone.end_of_day
  rescue ArgumentError, Date::Error
    nil
  end
end
```

Nota: `render_failure` achata `details`, então `shift_overlap` sai como `{ "error": "shift_overlap", "conflict": {...} }`. Confira que `render_failure` não quebra com Hash aninhado em `details` (ele só chama `respond_to?(:iso8601)` em cada valor; Hash não responde).

Em `config/routes.rb`, dentro do `scope "/professionals"`, **antes** de `get ":id"`:

```ruby
    post "links/:id/shifts",  to: "professional_shifts#create"
    post "shifts/:id/cancel", to: "professional_shifts#cancel"
```

e **depois** de `post ":id/links"`:

```ruby
    get  ":id/shifts",        to: "professional_shifts#index"
```

- [ ] **Step 4: Rode e veja passar; commit**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/requests/professional_shifts_spec.rb spec/requests/professional_links_spec.rb spec/requests/professionals_spec.rb`
Expected: PASS.

```bash
/opt/homebrew/bin/git add app/controllers/professional_shifts_controller.rb config/routes.rb spec/requests/professional_shifts_spec.rb
/opt/homebrew/bin/git commit -m "feat: add routes to list, schedule and cancel professional shifts

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 5: Concorrência "encerrar vínculo × lançar turno"**

```ruby
# spec/commands/professionals/link_lock_spec.rb
require "rails_helper"

# ScheduleShift (FOR SHARE no vínculo) e EndLink (FOR UPDATE) se excluem:
# ou o turno entra antes e é cancelado pelo encerramento (se futuro), ou o
# lançamento espera e vê o vínculo encerrado. Threads reais, sem fixture
# transacional — mesmo padrão de spec/models/health_unit_lock_spec.rb.
RSpec.describe "Vínculo: encerrar × lançar turno" do
  self.use_transactional_tests = false

  let(:release) { Queue.new }
  let(:threads) { [] }
  let(:ids) { {} }

  before do
    CityConnection.with(TEST_CITY_A) do
      Current.set(city: TEST_CITY_A) do
        tag = SecureRandom.hex(4)
        admin = User.create!(email_address: "adm-#{tag}@c.gov.br", password: "senha-segura-123")
        doc_user = User.create!(email_address: "doc-#{tag}@c.gov.br", password: "senha-segura-123")
        Membership.create!(user: doc_user, role: "health_professional", granted_at: Time.current)
        unit = HealthUnit.create!(name: "UBS Trava #{tag}", kind: "ubs")
        pro = Professional.create!(user: doc_user, professional_name: "P", council: "CRM", council_state: "PR",
                                   registration_number: tag.to_i(16).to_s[0, 8], cns: Professionals::Cns.generate(tag))
        link = Professionals::OpenLink.call(professional: pro, health_unit_id: unit.id, cbo_code: "225125", by: admin)
                                      .payload[:link]
        ids.merge!(admin: admin.id, link: link.id, unit: unit.id, pro: pro.id, doc_user: doc_user.id)
      end
    end
  end

  after do
    2.times { release << true }
    threads.each { |t| t.join(5) || t.kill }
    purge_committed_rows(ids)
  end

  def in_city(&) = CityConnection.with(TEST_CITY_A) { Current.set(city: TEST_CITY_A) { ApplicationRecord.transaction(&) } }

  it "o lançamento que espera o encerramento recebe link_ended" do
    locked = Queue.new
    threads << ender = Thread.new do
      in_city do
        link = ProfessionalLink.find(ids[:link])
        link.lock!
        locked << true
        release.pop
        link.update!(ended_at: Time.current, ended_by_user_id: ids[:admin])
      end
    end
    locked.pop(timeout: 5) or raise "a thread não pegou o lock"

    outcome = Queue.new
    threads << scheduler = Thread.new do
      CityConnection.with(TEST_CITY_A) do
        Current.set(city: TEST_CITY_A) do
          start = 2.days.from_now
          outcome << Professionals::ScheduleShift.call(link: ProfessionalLink.find(ids[:link]), starts_at: start,
                                                       ends_at: start + 4.hours, by: User.find(ids[:admin])).reason
        end
      end
    end
    expect(scheduler.join(0.5)).to be_nil # esperando o FOR UPDATE

    release << true
    ender.join(5)
    expect(outcome.pop(timeout: 5)).to eq(:link_ended)
  end
end
```

A limpeza usa um helper novo, porque as linhas commitadas batem em guards de DELETE (`professional_links`, `professional_shifts`, `memberships`, `users`, `domain_events`). Mesmo recurso de `spec/commands/city_lifecycle/invite_admin_spec.rb`: a suíte conecta como superusuário e desliga os triggers **só na transação da limpeza**. Crie `spec/support/committed_rows_cleanup.rb`:

```ruby
# spec/support/committed_rows_cleanup.rb
# Specs sem fixture transacional (threads) commitam de verdade em TEST_CITY_A.
# As tabelas do módulo 10, memberships, users e domain_events recusam DELETE
# por trigger; a suíte conecta como superusuário e desliga os triggers SÓ na
# transação da limpeza (mesmo recurso de invite_admin_spec.rb).
module CommittedRowsCleanup
  def purge_committed_rows(ids)
    CityConnection.with(TEST_CITY_A) do
      ApplicationRecord.transaction do
        ApplicationRecord.connection.execute("SET LOCAL session_replication_role = replica")
        professional_ids = Professional.where(user_id: ids.values_at(:doctor, :doc_user).compact).pluck(:id)
        ProfessionalShift.where(professional_id: professional_ids).delete_all
        ProfessionalLink.where(professional_id: professional_ids).delete_all
        DomainEvent.where("payload->>'professional_id' IN (?)", professional_ids.presence || [ "" ]).delete_all
        Professional.where(id: professional_ids).delete_all
        user_ids = ids.values_at(:admin, :doctor, :doc_user).compact
        Session.where(user_id: user_ids).delete_all
        Membership.where(user_id: user_ids).delete_all
        User.where(id: user_ids).delete_all
        HealthUnit.where(id: ids[:unit]).delete_all
      end
    end
  end
end

RSpec.configure { |c| c.include CommittedRowsCleanup }
```

Inclua esse arquivo no commit desta Task.

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/commands/professionals/link_lock_spec.rb`
Expected: PASS em menos de 10 s.

```bash
/opt/homebrew/bin/git add spec/commands/professionals/link_lock_spec.rb spec/support/committed_rows_cleanup.rb
/opt/homebrew/bin/git commit -m "test: prove ending a link and scheduling a shift exclude each other

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 6:** Revisor (subagente) lê o diff da fatia 3 contra a spec §3.3, §4.1 (turnos) e D6/D8.

---

## Fatia 4 — F-10.5 Regra clínica, telas existentes e semente

### Task 11: ClinicalAuthorization em Call, CallNext e Close

**Files:**
- Create: `app/services/professionals/clinical_authorization.rb`
- Modify: `app/commands/attendances/call.rb`, `app/commands/attendances/call_next.rb`, `app/commands/attendances/close.rb`
- Modify: `spec/support/appointment_helpers.rb` (helper de vínculo) e as specs existentes que chamam ou fecham com profissional
- Test: `spec/services/professionals/clinical_authorization_spec.rb`, `spec/commands/attendances/call_spec.rb`, `spec/commands/attendances/close_spec.rb`, `spec/commands/professionals/clinical_lock_spec.rb`

**Interfaces:**
- Produces:
  - `Professionals::ClinicalAuthorization.check(user:, health_unit_id:) → :ok | :missing_role | :missing_link`;
  - helper de spec `link_professional!(user, unit, cbo: "225125")` em `AppointmentHelpers`, que cria (se preciso) o perfil e abre o vínculo e devolve o `ProfessionalLink`.

- [ ] **Step 1: Helper de spec**

Em `spec/support/appointment_helpers.rb`, dentro do módulo:

```ruby
  # F-10.5: chamar e registrar desfecho clínico exigem vínculo ativo com a
  # unidade. Cria o perfil (se faltar) e abre o vínculo direto no banco.
  def link_professional!(user, unit, cbo: "225125")
    professional = user.professional || Professional.create!(
      user: user, professional_name: user.email_address.split("@").first.capitalize, council: "CRM",
      council_state: "PR", registration_number: (Professional.count + 10_000).to_s,
      cns: Professionals::Cns.generate(user.id)
    )
    admin = User.joins(:memberships).merge(Membership.active.where(role: "municipal_admin")).first ||
            staff_with("admin-link-#{SecureRandom.hex(3)}@cidade.gov.br", "municipal_admin")
    ProfessionalLink.create!(professional: professional, health_unit: unit, cbo_code: cbo,
                             started_at: Time.current, started_by_user: admin)
  end
```

- [ ] **Step 2: Escreva a spec da regra**

```ruby
# spec/services/professionals/clinical_authorization_spec.rb
require "rails_helper"

RSpec.describe Professionals::ClinicalAuthorization do
  let(:unit) { create_unit }
  let(:other_unit) { create_unit("UPA Norte", kind: "upa") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }

  def check(user = doctor, unit_id = unit.id) = described_class.check(user: user, health_unit_id: unit_id)

  it "papel + vínculo ativo com a unidade: ok (qualquer CBO)" do
    link_professional!(doctor, unit, cbo: "225124")
    expect(check).to eq(:ok)
  end

  it "sem papel: missing_role, mesmo com vínculo" do
    link_professional!(doctor, unit)
    doctor.memberships.sole.revoke!
    expect(check).to eq(:missing_role)
  end

  it "papel sem perfil, sem vínculo, vínculo em outra unidade, vínculo encerrado: missing_link" do
    expect(check).to eq(:missing_link)
    link = link_professional!(doctor, other_unit)
    expect(check).to eq(:missing_link)
    link.update!(ended_at: Time.current, ended_by_user: link.started_by_user)
    expect(check(doctor, other_unit.id)).to eq(:missing_link)
  end

  it "nil: missing_role" do
    expect(described_class.check(user: nil, health_unit_id: unit.id)).to eq(:missing_role)
  end

  it "nunca consulta turnos" do
    link_professional!(doctor, unit)
    expect(ProfessionalShift).not_to receive(:where)
    expect(ProfessionalShift).not_to receive(:valid_shifts)
    expect(check).to eq(:ok)
  end
end
```

- [ ] **Step 3: Ajuste as specs de comando existentes e acrescente os casos novos**

Rode primeiro para ver quais specs existentes passam a falhar quando a regra entrar:

```bash
/opt/homebrew/bin/git grep -ln "Attendances::Call\|Attendances::CallNext\|Attendances::Close\|call_next\|/call\"\|/close\"" -- spec
```

Em cada uma, onde um `health_professional` chama ou registra desfecho clínico (`discharged`, `referred`, `return`), acrescente `link_professional!(<usuário>, <unidade>)` no setup (tipicamente num `before` ou logo depois do `let` do profissional). Não mexa nos casos de `left` pela recepção.

Acrescente a `spec/commands/attendances/call_spec.rb` (use os `let` já existentes na spec para cidadão, unidade, recepção e profissional; os nomes abaixo são os de `spec/invariants/health_unit_invariants_spec.rb`, ajuste ao arquivo):

```ruby
  describe "regra clínica (F-10.5)" do
    it "profissional sem vínculo com a unidade: missing_link e o atendimento segue aguardando" do
      attendance = waiting_attendance(citizen, unit: unit, by: reception)
      result = Attendances::Call.call(attendance: attendance, health_unit_id: unit.id, by: doctor)
      expect(result.reason).to eq(:missing_link)
      expect(attendance.reload.status).to eq("waiting")
    end

    it "usuário sem o papel: missing_role" do
      attendance = waiting_attendance(citizen, unit: unit, by: reception)
      result = Attendances::Call.call(attendance: attendance, health_unit_id: unit.id, by: reception)
      expect(result.reason).to eq(:missing_role)
    end

    it "vínculo ativo sem nenhum turno: chama" do
      link_professional!(doctor, unit)
      attendance = waiting_attendance(citizen, unit: unit, by: reception)
      expect(Attendances::Call.call(attendance: attendance, health_unit_id: unit.id, by: doctor)).to be_ok
    end

    it "CallNext sem vínculo: missing_link sem gastar tentativas" do
      waiting_attendance(citizen, unit: unit, by: reception)
      expect(Attendances::Call).not_to receive(:call)
      expect(Attendances::CallNext.call(health_unit_id: unit.id, by: doctor).reason).to eq(:missing_link)
    end
  end
```

Acrescente a `spec/commands/attendances/close_spec.rb`:

```ruby
  describe "regra clínica (F-10.5)" do
    it "desfecho clínico sem vínculo: missing_link e nada fecha" do
      attendance = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
      result = Attendances::Close.call(attendance: attendance, outcome: "discharged", referral_unit_id: nil,
                                       referral_note: nil, by: doctor)
      expect(result.reason).to eq(:missing_link)
      expect(attendance.reload).to be_open
    end

    it "left pela recepção, sem vínculo: fecha" do
      attendance = waiting_attendance(citizen, unit: unit, by: reception)
      result = Attendances::Close.call(attendance: attendance, outcome: "left", referral_unit_id: nil,
                                       referral_note: nil, by: reception)
      expect(result).to be_ok
    end

    it "vínculo encerrado entre a leitura e a chamada do desfecho: missing_link" do
      link = link_professional!(doctor, unit)
      attendance = in_care!(waiting_attendance(citizen, unit: unit, by: reception), by: doctor)
      link.update!(ended_at: Time.current, ended_by_user: link.started_by_user)
      result = Attendances::Close.call(attendance: attendance, outcome: "discharged", referral_unit_id: nil,
                                       referral_note: nil, by: doctor)
      expect(result.reason).to eq(:missing_link)
    end
  end
```

- [ ] **Step 4: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/services/professionals/clinical_authorization_spec.rb spec/commands/attendances`
Expected: FAIL nos casos novos (a regra ainda não existe).

- [ ] **Step 5: Implemente a regra e ligue nos comandos**

```ruby
# app/services/professionals/clinical_authorization.rb
# Regra da chamada e do desfecho clínico (ADR 0021; F-10.5): papel
# health_professional ativo E vínculo ativo com a unidade do atendimento, em
# qualquer CBO. Chame DENTRO da transação do comando: o FOR SHARE no vínculo
# faz o encerramento (EndLink, FOR UPDATE) esperar o ato em curso, ou ser
# visto por ele. Turno nunca entra aqui — falha de escala não bloqueia
# atendimento.
module Professionals
  module ClinicalAuthorization
    module_function

    def check(user:, health_unit_id:)
      return :missing_role unless user&.has_role?("health_professional")

      linked = ProfessionalLink.active.joins(:professional)
                               .where(professionals: { user_id: user.id }, health_unit_id: health_unit_id.to_s)
                               .lock("FOR SHARE OF professional_links").pick(:id)
      linked ? :ok : :missing_link
    end
  end
end
```

`app/commands/attendances/call.rb`:

```ruby
# Chamada (spec 2026-09-25 §2.1): waiting → in_care, sob lock. Exige papel e
# vínculo ativo com a unidade do atendimento (ADR 0021; F-10.5).
module Attendances
  class Call
    def self.call(attendance:, health_unit_id:, by:)
      return Result.fail(:wrong_unit) unless attendance.health_unit_id == health_unit_id.to_s

      state = ApplicationRecord.transaction do
        attendance.lock!
        authorization = Professionals::ClinicalAuthorization.check(user: by, health_unit_id: attendance.health_unit_id)
        next authorization unless authorization == :ok
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

`app/commands/attendances/call_next.rb`, no início de `self.call`:

```ruby
    def self.call(health_unit_id:, by:)
      # Atalho: sem papel ou vínculo, não gasta tentativas. Quem garante é o
      # Call, que rechecar sob lock.
      authorization = Professionals::ClinicalAuthorization.check(user: by, health_unit_id: health_unit_id)
      return Result.fail(authorization) unless authorization == :ok

      ATTEMPTS.times do
```

`app/commands/attendances/close.rb`, dentro da transação, logo depois de `HealthUnit.lock_active!(unit.id) if unit`:

```ruby
        # Desfecho clínico exige papel e vínculo (F-10.5); left é ato de balcão.
        unless outcome == "left"
          authorization = Professionals::ClinicalAuthorization.check(user: by, health_unit_id: attendance.health_unit_id)
          next Result.fail(authorization) unless authorization == :ok
        end
```

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/services/professionals/clinical_authorization_spec.rb spec/commands/attendances spec/invariants/appointment_invariants_spec.rb spec/invariants/health_unit_invariants_spec.rb`
Expected: PASS. Se as duas suítes de invariantes quebrarem por falta de vínculo no setup, acrescente `link_professional!` ao `let`/`before` delas (o comportamento que elas provam não muda).

- [ ] **Step 7: Concorrência "encerrar vínculo × chamar"**

```ruby
# spec/commands/professionals/clinical_lock_spec.rb
require "rails_helper"

# A chamada (ClinicalAuthorization, FOR SHARE no vínculo) e o encerramento
# (EndLink, FOR UPDATE) se excluem: o encerramento espera a chamada em curso;
# a chamada que chega depois de um encerramento em curso espera e recebe
# missing_link. Threads reais contra TEST_CITY_A, sem fixture transacional.
RSpec.describe "Vínculo: encerrar × chamar" do
  self.use_transactional_tests = false

  let(:release) { Queue.new }
  let(:threads) { [] }
  let(:ids) { {} }

  before do
    CityConnection.with(TEST_CITY_A) do
      Current.set(city: TEST_CITY_A) do
        tag = SecureRandom.hex(4)
        admin = User.create!(email_address: "adm-#{tag}@c.gov.br", password: "senha-segura-123")
        doctor = User.create!(email_address: "doc-#{tag}@c.gov.br", password: "senha-segura-123")
        Membership.create!(user: doctor, role: "health_professional", granted_at: Time.current)
        unit = HealthUnit.create!(name: "UBS Chamada #{tag}", kind: "ubs")
        pro = Professional.create!(user: doctor, professional_name: "P", council: "CRM", council_state: "PR",
                                   registration_number: tag.to_i(16).to_s[0, 8], cns: Professionals::Cns.generate(tag))
        link = ProfessionalLink.create!(professional: pro, health_unit: unit, cbo_code: "225125",
                                        started_at: Time.current, started_by_user: admin)
        ids.merge!(admin: admin.id, doctor: doctor.id, unit: unit.id, link: link.id)
      end
    end
  end

  after do
    2.times { release << true }
    threads.each { |t| t.join(5) || t.kill }
    purge_committed_rows(ids)
  end

  def in_city(&) = CityConnection.with(TEST_CITY_A) { Current.set(city: TEST_CITY_A) { ApplicationRecord.transaction(&) } }

  it "o encerramento espera a checagem em curso; a checagem seguinte recebe missing_link" do
    checked = Queue.new
    threads << caller_thread = Thread.new do
      in_city do
        checked << Professionals::ClinicalAuthorization.check(user: User.find(ids[:doctor]), health_unit_id: ids[:unit])
        release.pop
      end
    end
    expect(checked.pop(timeout: 5)).to eq(:ok)

    threads << ender = Thread.new do
      in_city do
        link = ProfessionalLink.find(ids[:link])
        link.lock!
        link.update!(ended_at: Time.current, ended_by_user_id: ids[:admin])
      end
    end
    expect(ender.join(0.5)).to be_nil # esperando o FOR SHARE da chamada

    release << true
    caller_thread.join(5)
    expect(ender.join(5)).to be(ender)
    after_end = in_city { Professionals::ClinicalAuthorization.check(user: User.find(ids[:doctor]), health_unit_id: ids[:unit]) }
    expect(after_end).to eq(:missing_link)
  end
end
```

A limpeza usa `purge_committed_rows` (criado na Task 10, Step 5).

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/commands/professionals/clinical_lock_spec.rb`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add app/services/professionals/clinical_authorization.rb app/commands/attendances spec/support/appointment_helpers.rb spec/services/professionals/clinical_authorization_spec.rb spec/commands spec/invariants spec/requests
/opt/homebrew/bin/git commit -m "feat: require an active unit link to call and record clinical outcomes

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 12: 403 nomeado, nome profissional na fila e `professional_status` na Equipe

**Files:**
- Modify: `app/controllers/concerns/attendance_access.rb`, `app/controllers/attendances_controller.rb`, `app/commands/attendances/unit_queue.rb`, `app/controllers/setup_controller.rb`
- Test: `spec/requests/attendances_spec.rb`, `spec/requests/setup_list_memberships_spec.rb`

**Interfaces:**
- Consumes: `ClinicalAuthorization` (Task 11), `Professionals::Status` (Task 4).
- Produces:
  - HTTP 403 `{"error":"missing_role"}` e `{"error":"missing_link"}` nas rotas de chamada e desfecho;
  - `called_by_name` e `closed_by_name` com o nome profissional (cai para o e-mail);
  - `professional_status` nas linhas de `health_professional` de `GET /setup/memberships`.

- [ ] **Step 1: Escreva as specs**

Acrescente a `spec/requests/attendances_spec.rb` (reuse os `let` do arquivo para cidadão, unidade, recepção e profissional):

```ruby
  describe "regra clínica (F-10.5) na API" do
    it "chamar sem vínculo: 403 missing_link" do
      attendance = waiting_attendance(citizen, unit: unit, by: reception)
      sign_in_as(doctor)
      json_post "/attendance/attendances/#{attendance.id}/call", health_unit_id: unit.id
      expect(response).to have_http_status(:forbidden)
      expect(JSON.parse(response.body)).to eq("error" => "missing_link")
    end

    it "chamar próximo sem o papel: 403 missing_role" do
      waiting_attendance(citizen, unit: unit, by: reception)
      sign_in_as(reception)
      json_post "/attendance/units/#{unit.id}/call_next"
      expect(response).to have_http_status(:forbidden)
      expect(JSON.parse(response.body)).to eq("error" => "missing_role")
    end

    it "desfecho clínico pela recepção: 403 missing_role; left pela recepção: 200" do
      attendance = waiting_attendance(citizen, unit: unit, by: reception)
      sign_in_as(reception)
      json_post "/attendance/attendances/#{attendance.id}/close", outcome: "discharged"
      expect(JSON.parse(response.body)).to eq("error" => "missing_role")
      json_post "/attendance/attendances/#{attendance.id}/close", outcome: "left"
      expect(response).to have_http_status(:ok)
    end

    it "a fila mostra o nome profissional de quem chamou" do
      link_professional!(doctor, unit)
      doctor.professional.update!(professional_name: "Helena Duarte")
      waiting_attendance(citizen, unit: unit, by: reception)
      sign_in_as(doctor)
      json_post "/attendance/units/#{unit.id}/call_next"
      get "/attendance/units/#{unit.id}/queue"
      expect(JSON.parse(response.body)["in_care"].sole["called_by_name"]).to eq("Helena Duarte")
    end
  end
```

Acrescente a `spec/requests/setup_list_memberships_spec.rb` (reuse o admin do arquivo):

```ruby
  it "linhas de health_professional trazem professional_status; outras não" do
    novato = staff_with("novato@cidade.gov.br", "health_professional")
    sign_in_as(admin)
    get "/setup/memberships"
    rows = JSON.parse(response.body)["data"]
    pro_row = rows.find { |r| r["user"]["id"] == novato.id }
    expect(pro_row["professional_status"]).to eq("missing_profile")
    expect(rows.find { |r| r["role"] == "municipal_admin" }).not_to have_key("professional_status")
  end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/requests/attendances_spec.rb spec/requests/setup_list_memberships_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implemente**

Em `app/controllers/concerns/attendance_access.rb`:

```ruby
  def require_professional
    # ADR 0021: a recusa diz o que falta. O vínculo é conferido no comando.
    forbid("missing_role") unless CitizenVerificationPolicy.new(Current.user, nil).care?
  end

  def forbid(reason = "forbidden")
    render json: { error: reason }, status: :forbidden
  end
```

Em `app/controllers/attendances_controller.rb`:
- acrescente `missing_role: :forbidden, missing_link: :forbidden` ao `ERROR_STATUS`;
- em `close`, troque `return forbid unless ...care?` por `return forbid("missing_role") unless ...care?`;
- em `queue_json`, troque `called_by_name: a.called_by_user&.email_address` por `called_by_name: staff_name(a.called_by_user)`;
- em `attendance_json`, acrescente `called_by_name: staff_name(a.called_by_user), closed_by_name: staff_name(a.closed_by_user)`;
- no `private`:

```ruby
  # F-10.5: quem chamou/fechou aparece pelo nome profissional; sem perfil,
  # pelo e-mail (recepção, ou profissional ainda sem cadastro).
  def staff_name(user)
    return nil unless user

    user.professional&.professional_name || user.email_address
  end
```

Em `app/commands/attendances/unit_queue.rb`, troque `:called_by_user` em `INCLUDES` por `{ called_by_user: :professional }`.

Em `app/controllers/setup_controller.rb#list_memberships`:

```ruby
    memberships = Membership.active.joins(:user).where(users: { deactivated_at: nil }).includes(:user).to_a
    pro_ids = memberships.select { |m| m.role == "health_professional" }.map(&:user_id)
    status = Professionals::Status.for_users(pro_ids)
    rows = memberships.map do |m|
      row = { id: m.id, user: { id: m.user.id, email_address: m.user.email_address }, role: m.role,
              granted_at: m.granted_at.iso8601 }
      row[:professional_status] = status[m.user_id] if m.role == "health_professional"
      row
    end
    render json: { data: rows }
```

- [ ] **Step 4: Rode a regressão do atendimento e da Equipe**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/requests/attendances_spec.rb spec/requests/attendance_spec.rb spec/requests/attendance_contract_spec.rb spec/requests/setup_list_memberships_spec.rb spec/requests/health_units_spec.rb`
Expected: PASS. Onde uma spec existente esperava `"forbidden"` para profissional/recepção nessas rotas, troque pela recusa nomeada (é a mudança pedida pelo ADR 0021) e diga no relatório quais trocou.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/concerns/attendance_access.rb app/controllers/attendances_controller.rb app/commands/attendances/unit_queue.rb app/controllers/setup_controller.rb spec/requests
/opt/homebrew/bin/git commit -m "feat: name clinical refusals, show professional names and flag incomplete professionals

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 13: Semente de dev realista

**Files:**
- Create: `lib/professional_crew.rb`
- Modify: `db/seeds.rb`
- Test: `spec/lib/professional_crew_spec.rb`

**Interfaces:**
- Consumes: `SignatureCrew.ensure_totp`, `SignatureCrew.otpauth_uri`, comandos das Tasks 3, 7, 9, `Professionals::Cns.generate`.
- Produces: `ProfessionalCrew.seed_current_city(slug:, password:, admin:) → { units: [...], accounts: [{ email:, role:, otpauth_uri: }], professionals: [{ email:, name:, links: n, shifts: n }] }`.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/lib/professional_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/professional_crew")

RSpec.describe ProfessionalCrew do
  before do
    Current.city = TEST_CITY_A
    (CityProfile.current || CityProfile.new).update!(name: "Cidade Teste", uf: "PR", ibge_code: "4106902")
    SignatureCrew.seed_current_city(slug: "teste", password: "dev-password")
  end
  after { Current.reset }

  let(:admin) { staff_with("admin@teste.demo", "municipal_admin") }

  def seed = described_class.seed_current_city(slug: "teste", password: "dev-password", admin: admin)

  it "cria unidades, perfis, vínculos e turnos com forma real" do
    travel_to(Time.zone.parse("2026-10-05 10:00")) { seed }

    expect(HealthUnit.pluck(:name)).to include("UBS Jardim das Flores", "UBS Vila Esperança", "UPA 24h Centro")
    expect(Professional.count).to eq(3)
    expect(Professional.all).to all(satisfy { |p| Professionals::Cns.valid?(p.cns) && p.council_state == "PR" })

    medica = User.find_by!(email_address: "profissional@teste.demo").professional
    expect(medica.council).to eq("CRM")
    expect(medica.links.active.map(&:cbo_code)).to contain_exactly("225125", "225124")

    enfermeira = User.find_by!(email_address: "enfermeira@teste.demo").professional
    expect(enfermeira.links.where.not(ended_at: nil).count).to eq(1)

    tecnico = User.find_by!(email_address: "tecnico@teste.demo").professional
    shift = ProfessionalShift.valid_shifts.find_by!(professional: tecnico)
    expect(shift.ends_at - shift.starts_at).to eq(24.hours)

    overnight = ProfessionalShift.valid_shifts.where(professional: medica)
                                 .find { |s| s.starts_at.hour == 19 }
    expect(overnight.ends_at.to_date).to eq(overnight.starts_at.to_date + 1)

    novato = User.find_by!(email_address: "novato@teste.demo")
    expect(novato.has_role?("health_professional")).to be(true)
    expect(novato.professional).to be_nil
  end

  it "é idempotente: rodar duas vezes não duplica nada" do
    travel_to(Time.zone.parse("2026-10-05 10:00")) { seed }
    counts = -> { [ HealthUnit.count, Professional.count, ProfessionalLink.count, ProfessionalShift.count, User.count ] }
    before = counts.call
    travel_to(Time.zone.parse("2026-10-05 11:00")) { seed }
    expect(counts.call).to eq(before)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/lib/professional_crew_spec.rb`
Expected: FAIL (`cannot load such file`).

- [ ] **Step 3: Implemente**

```ruby
# lib/professional_crew.rb
require "zlib"

# Semente de dev do módulo 10 (spec 2026-09-27-module-10-professionals §6).
# Dev é fictício, mas imita o real: CBO reais, CNS com dígito verificador
# válido, conselho coerente com a ocupação, UF da cidade, plantão noturno e de
# 24h, uma pessoa em duas unidades com ocupações diferentes, um vínculo
# encerrado e alguém com o papel sem cadastro. Tudo pelos comandos do domínio,
# para as validações rodarem. Idempotente. Roda depois do SignatureCrew, que
# cria profissional@ e recepcao@.
class ProfessionalCrew
  UNITS = [
    { name: "UBS Jardim das Flores", kind: "ubs" },
    { name: "UBS Vila Esperança", kind: "ubs" },
    { name: "UPA 24h Centro", kind: "upa" }
  ].freeze

  NEW_MEMBERS = [
    { email_prefix: "enfermeira", secret_env: "DEV_NURSE_OTP_SECRET", default_secret: "ONSWG4TFOQQGC3TEEBXXE2LUNFXW4ZLF" },
    { email_prefix: "tecnico", secret_env: "DEV_TECHNICIAN_OTP_SECRET", default_secret: "ORSWG3TJMNXSA43FNZRWK4TBMRXSA43F" },
    { email_prefix: "novato", secret_env: "DEV_NEWCOMER_OTP_SECRET", default_secret: "NZXXMYLUN5QWKZDFONUW4ZLTONQXIZLS" }
  ].freeze

  PROFILES = {
    "profissional" => { professional_name: "Helena Duarte Moreira", council: "CRM" },
    "enfermeira" => { professional_name: "Carla Nogueira Prado", council: "COREN" },
    "tecnico" => { professional_name: "Rafael Teixeira Lima", council: "COREN" }
  }.freeze

  class << self
    def seed_current_city(slug:, password:, admin:)
      units = UNITS.to_h { |u| [ u[:name], HealthUnit.find_or_create_by!(name: u[:name]) { |h| h.kind = u[:kind] } ] }
      accounts = NEW_MEMBERS.map { |m| ensure_member(m, slug: slug, password: password) }
      uf = CityProfile.current&.uf.presence || "PR"
      professionals = PROFILES.keys.to_h { |prefix| [ prefix, ensure_profile(prefix, slug: slug, uf: uf, admin: admin) ] }

      ubs1, ubs2, upa = units.values_at("UBS Jardim das Flores", "UBS Vila Esperança", "UPA 24h Centro")
      medica_ubs = ensure_link(professionals["profissional"], ubs1, "225125", admin)
      medica_upa = ensure_link(professionals["profissional"], upa, "225124", admin)
      enf_ubs = ensure_link(professionals["enfermeira"], ubs1, "223505", admin)
      ensure_ended_link(professionals["enfermeira"], ubs2, "223505", admin)
      tec_upa = ensure_link(professionals["tecnico"], upa, "322205", admin)

      business_days = (1..9).map { |n| Time.zone.today + n }.reject { |d| d.saturday? || d.sunday? }.first(5)
      ensure_shifts(medica_ubs, admin, business_days.map { |d| [ at(d, 7), at(d, 13) ] })
      ensure_shifts(medica_upa, admin, [ [ at(business_days.last + 1, 19), at(business_days.last + 2, 7) ] ])
      ensure_shifts(enf_ubs, admin, business_days.map { |d| [ at(d, 7), at(d, 19) ] })
      ensure_shifts(tec_upa, admin, [ [ at(Time.zone.today + 2, 7), at(Time.zone.today + 3, 7) ] ])

      {
        units: units.keys,
        accounts: accounts,
        professionals: professionals.map do |prefix, p|
          { email: "#{prefix}@#{slug}.demo", name: p.professional_name, links: p.links.active.count,
            shifts: ProfessionalShift.valid_shifts.where(professional: p).count }
        end
      }
    end

    private

    def at(day, hour) = day.in_time_zone.change(hour: hour)

    def ensure_member(member, slug:, password:)
      user = User.find_or_initialize_by(email_address: "#{member[:email_prefix]}@#{slug}.demo")
      user.password = password
      user.save!
      Membership.find_or_create_by!(user: user, role: "health_professional") { |m| m.granted_at = Time.current }
      SignatureCrew.ensure_totp(user, secret_env: member[:secret_env], default_secret: member[:default_secret])
      { email: user.email_address, role: "health_professional", otpauth_uri: SignatureCrew.otpauth_uri(user) }
    end

    def ensure_profile(prefix, slug:, uf:, admin:)
      user = User.find_by!(email_address: "#{prefix}@#{slug}.demo")
      return user.professional if user.professional

      seed = "#{slug}:#{prefix}"
      attrs = PROFILES.fetch(prefix).merge(council_state: uf, cns: Professionals::Cns.generate(seed),
                                           registration_number: (Zlib.crc32(seed) % 90_000 + 10_000).to_s)
      result = Professionals::Create.call(user_id: user.id, attrs: attrs, by: admin)
      raise "semente: perfil de #{prefix} recusado (#{result.reason} #{result.details})" if result.failure?

      result.payload[:professional]
    end

    def ensure_link(professional, unit, cbo, admin)
      existing = professional.links.active.find_by(health_unit: unit, cbo_code: cbo)
      return existing if existing

      result = Professionals::OpenLink.call(professional: professional, health_unit_id: unit.id, cbo_code: cbo, by: admin)
      raise "semente: vínculo #{cbo} recusado (#{result.reason})" if result.failure?

      result.payload[:link]
    end

    # Histórico: o vínculo encerrado só nasce se o par nunca existiu.
    def ensure_ended_link(professional, unit, cbo, admin)
      return if professional.links.exists?(health_unit: unit, cbo_code: cbo)

      Professionals::EndLink.call(link: ensure_link(professional, unit, cbo, admin), by: admin)
    end

    def ensure_shifts(link, admin, windows)
      return if link.shifts.valid_shifts.where("starts_at > ?", Time.current).exists?

      windows.each do |starts_at, ends_at|
        result = Professionals::ScheduleShift.call(link: link, starts_at: starts_at, ends_at: ends_at, by: admin)
        raise "semente: turno recusado (#{result.reason} #{result.details})" if result.failure?
      end
    end
  end
end
```

Nota sobre os plantões da médica: o 19h–07h na UPA cai no dia útil seguinte ao último da série da UBS (07–13), então não sobrepõe. O plantão de 24h do técnico é de outra pessoa.

Em `db/seeds.rb`, logo depois do laço que imprime `crew[:accounts]` e antes do `puts crew[:draft] ...`:

```ruby
        # ── Profissionais (módulo 10, spec 2026-09-27 §6) ─────────────────────
        pros = ProfessionalCrew.seed_current_city(slug: slug, password: password, admin: muni_admin)
        pros[:accounts].each do |account|
          puts "[seeds] #{account[:role].ljust(18)} #{account[:email]} / #{password} + MFA → #{account[:otpauth_uri]}"
        end
        pros[:professionals].each do |p|
          puts "[seeds] profissional . #{p[:email]} — #{p[:name]} (#{p[:links]} vínculos, #{p[:shifts]} turnos)"
        end
```

Garanta que `lib/professional_crew.rb` seja carregado como o `SignatureCrew` é (procure `signature_crew` em `db/seeds.rb`/`config/`; se for `require`, faça o mesmo).

- [ ] **Step 4: Rode a spec e a semente de dev**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/lib/professional_crew_spec.rb
docker compose exec -T -w /rails/.claude/mod10 api bin/rails db:seed
docker compose exec -T -w /rails/.claude/mod10 api bin/rails db:seed
```
Expected: PASS; as duas execuções da semente terminam sem erro e imprimem as mesmas contagens.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add lib/professional_crew.rb db/seeds.rb spec/lib/professional_crew_spec.rb
/opt/homebrew/bin/git commit -m "feat: seed realistic fictitious professionals, links and shifts in dev

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 14: Suíte de invariantes com teste de mutação

**Files:**
- Create: `spec/invariants/professional_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 1 a 13.

- [ ] **Step 1: Escreva a suíte**

```ruby
# spec/invariants/professional_invariants_spec.rb
require "rails_helper"

# Módulo 10, critério de fechamento (ADR 0021; spec 2026-09-27 §7.1). Cada
# bloco tem a mutação que precisa deixá-lo vermelho (registrada no relatório
# da fatia 4).
RSpec.describe "Invariantes dos profissionais (ADR 0021)", type: :request do
  before { Current.city = TEST_CITY_A; Rails.cache.clear }
  after { Current.reset }

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:reception) { staff_with("recepcao@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:unit) { create_unit }
  let(:attrs) do
    { professional_name: "Helena", council: "CRM", council_state: "PR", registration_number: "12345", cns: "700000000000005" }
  end

  def profile! = Professionals::Create.call(user_id: doctor.id, attrs: attrs, by: admin).payload[:professional]
  def sql(statement) = ApplicationRecord.connection.execute(statement)

  # Mutação: remover o índice único de professionals.user_id.
  it "perfil 1:1 com o usuário, garantido pelo banco" do
    profile!
    expect do
      sql("INSERT INTO professionals (id, user_id, professional_name, council, council_state, registration_number, cns, " \
          "created_at, updated_at) VALUES (gen_random_uuid(), '#{doctor.id}', 'X', 'CRM', 'PR', '999', 'x', now(), now())")
    end.to raise_error(ActiveRecord::RecordNotUnique)
  end

  # Mutação: tirar a checagem de papel de Professionals::Create.
  it "perfil só para quem tem o papel" do
    expect(Professionals::Create.call(user_id: reception.id, attrs: attrs, by: admin).reason).to eq(:missing_role)
  end

  # Mutação: desligar professional_links_guard.
  it "vínculo só por acréscimo" do
    link = link_professional!(doctor, unit)
    expect { sql("DELETE FROM professional_links WHERE id = '#{link.id}'") }.to raise_error(ActiveRecord::StatementInvalid)
    expect { sql("UPDATE professional_links SET cbo_code = '225124' WHERE id = '#{link.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid)
    Professionals::EndLink.call(link: link, by: admin)
    expect { sql("UPDATE professional_links SET ended_at = now() WHERE id = '#{link.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid)
  end

  # Mutação: remover idx_professional_links_one_active.
  it "um vínculo ativo por (profissional, unidade, CBO)" do
    link = link_professional!(doctor, unit)
    expect do
      ProfessionalLink.create!(professional: link.professional, health_unit: unit, cbo_code: "225125",
                               started_at: Time.current, started_by_user: admin)
    end.to raise_error(ActiveRecord::RecordNotUnique)
  end

  describe "turnos" do
    let(:link) { link_professional!(doctor, unit) }
    let(:day) { Time.zone.tomorrow.in_time_zone }
    let(:shift) do
      Professionals::ScheduleShift.call(link: link, starts_at: day.change(hour: 7), ends_at: day.change(hour: 13), by: admin)
                                  .payload[:shift]
    end

    # Mutação: remover professional_shifts_guard, a EXCLUDE ou o CHECK de janela.
    it "só por acréscimo, sem sobreposição, até 24h" do
      expect { sql("DELETE FROM professional_shifts WHERE id = '#{shift.id}'") }.to raise_error(ActiveRecord::StatementInvalid)
      expect { sql("UPDATE professional_shifts SET starts_at = starts_at - interval '1 hour' WHERE id = '#{shift.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid)
      expect do
        ProfessionalShift.create!(professional_link: link, professional: link.professional, created_by_user: admin,
                                  starts_at: shift.starts_at + 1.hour, ends_at: shift.ends_at + 1.hour)
      end.to raise_error(ActiveRecord::ExclusionViolation)
      expect do
        ProfessionalShift.create!(professional_link: link, professional: link.professional, created_by_user: admin,
                                  starts_at: day + 2.days, ends_at: day + 3.days + 1.minute)
      end.to raise_error(ActiveRecord::StatementInvalid)
    end

    # Mutação: tirar o cancelamento de EndLink (e o IF de vínculo encerrado do trigger).
    it "não existe turno válido futuro em vínculo encerrado" do
      shift
      Professionals::EndLink.call(link: link, by: admin)
      expect(shift.reload.cancelled_at).to be_present
      expect(link.shifts.valid_shifts.where("starts_at > ?", Time.current)).to be_empty
      expect(Professionals::ScheduleShift.call(link: link, starts_at: day + 2.days, ends_at: day + 2.days + 1.hour,
                                               by: admin).reason).to eq(:link_ended)
    end
  end

  # Mutação: trocar require_admin por require_attendance_staff num dos controllers.
  describe "só o municipal_admin cadastra" do
    (Membership::ROLES - %w[municipal_admin]).each do |role|
      it "#{role}: 403 em toda rota de escrita, com step-up aberto" do
        link = link_professional!(doctor, unit)
        sign_in_as(staff_with("#{role}@c.gov.br", role)).update!(mfa_verified_at: Time.current)
        [
          [ "/professionals", attrs.merge(user_id: doctor.id) ],
          [ "/professionals/#{link.professional_id}", { council_state: "SC" } ],
          [ "/professionals/#{link.professional_id}/links", { health_unit_id: unit.id, cbo_code: "225124" } ],
          [ "/professionals/links/#{link.id}/end", {} ],
          [ "/professionals/links/#{link.id}/shifts", { starts_at: 1.day.from_now.iso8601, ends_at: (1.day.from_now + 1.hour).iso8601 } ],
          [ "/professionals/shifts/#{SecureRandom.uuid}/cancel", { reason: "x" } ]
        ].each do |path, params|
          json_post path, params
          expect(response).to have_http_status(:forbidden), "#{role} POST #{path}"
        end
        expect(link.reload.ended_at).to be_nil
        expect(ProfessionalShift.count).to eq(0)
      end
    end
  end

  # Mutação: tirar o `check` de Attendances::Call e de Attendances::Close.
  describe "chamada e desfecho exigem papel + vínculo com a unidade do atendimento" do
    let(:other_unit) { create_unit("UPA Norte", kind: "upa") }

    # Um atendimento só: as recusas não mudam o estado, então o mesmo
    # atendimento serve a todas as tentativas.
    let(:attendance) { waiting_attendance(citizen, unit: unit, by: reception) }

    def call(by) = Attendances::Call.call(attendance: attendance.reload, health_unit_id: unit.id, by: by)

    it "sem papel: missing_role" do
      expect(call(reception).reason).to eq(:missing_role)
    end

    it "sem vínculo, vínculo em outra unidade, vínculo encerrado: missing_link" do
      expect(call(doctor).reason).to eq(:missing_link)
      other = link_professional!(doctor, other_unit)
      expect(call(doctor).reason).to eq(:missing_link)
      mine = link_professional!(doctor, unit)
      Professionals::EndLink.call(link: mine, by: admin)
      expect(call(doctor).reason).to eq(:missing_link)
      expect(other.reload).to be_active
    end

    it "desfecho clínico sem vínculo: missing_link; left pela recepção: ok" do
      in_care!(attendance, by: doctor)
      expect(Attendances::Close.call(attendance: attendance, outcome: "discharged", referral_unit_id: nil,
                                     referral_note: nil, by: doctor).reason).to eq(:missing_link)
      waiting = waiting_attendance(Citizen.create!(cpf: "11144477735", phone: "+5541998765433"), unit: unit, by: reception)
      expect(Attendances::Close.call(attendance: waiting, outcome: "left", referral_unit_id: nil,
                                     referral_note: nil, by: reception)).to be_ok
    end
  end

  # Mutação: fazer ClinicalAuthorization exigir turno válido no instante.
  it "turno nunca bloqueia ato clínico: vinculado sem turno, e com turno cancelado, chama e fecha" do
    link = link_professional!(doctor, unit)
    day = Time.zone.tomorrow.in_time_zone
    shift = Professionals::ScheduleShift.call(link: link, starts_at: day.change(hour: 7), ends_at: day.change(hour: 13),
                                              by: admin).payload[:shift]
    Professionals::CancelShift.call(shift: shift, reason: "troca", by: admin)

    attendance = waiting_attendance(citizen, unit: unit, by: reception)
    expect(Attendances::Call.call(attendance: attendance, health_unit_id: unit.id, by: doctor)).to be_ok
    expect(Attendances::Close.call(attendance: attendance.reload, outcome: "discharged", referral_unit_id: nil,
                                   referral_note: nil, by: doctor)).to be_ok
  end

  # Mutação: pôr `cns: professional.cns` no payload de professional.created.
  it "nenhum evento professional.* carrega dado sensível" do
    professional = profile!
    professional.update!(phone: "41998765432", contact_email: "helena@ubs.org")
    Professionals::UpdateProfile.call(professional: professional, attrs: { "professional_name" => "Helena D." }, by: admin)
    link = Professionals::OpenLink.call(professional: professional, health_unit_id: unit.id, cbo_code: "225125", by: admin)
                                  .payload[:link]
    day = Time.zone.tomorrow.in_time_zone
    shift = Professionals::ScheduleShift.call(link: link, starts_at: day.change(hour: 7), ends_at: day.change(hour: 13),
                                              by: admin).payload[:shift]
    Professionals::CancelShift.call(shift: shift, reason: "troca", by: admin)
    Professionals::EndLink.call(link: link, by: admin)

    payloads = DomainEvent.where("name LIKE 'professional.%'").map { |e| e.payload.to_json }
    expect(payloads.size).to eq(6)
    [ professional.cns, professional.registration_number, "41998765432", "helena@ubs.org", "Helena" ].each do |secret|
      expect(payloads).to all(satisfy { |p| !p.include?(secret) }), "vazou #{secret}"
    end
  end
end
```

- [ ] **Step 2: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/invariants/professional_invariants_spec.rb`
Expected: PASS.

- [ ] **Step 3: Teste de mutação**

Para cada comentário `# Mutação:` da suíte: aplique a mutação, rode só aquele bloco, confirme que fica **vermelho**, desfaça (`/opt/homebrew/bin/git checkout -- <arquivo>`; para trigger ou índice, recarregue com `bin/rails city:test_databases`). Registre no relatório da fatia 4 uma linha por invariante: mutação aplicada, spec que falhou, mensagem de falha. Se alguma mutação passar verde, a spec não prova a invariante: conserte a spec antes de seguir.

- [ ] **Step 4: Regressão e suíte completa**

```bash
docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec spec/invariants
docker compose stop worker
docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec
docker compose start worker
```
Expected: `spec/invariants` inteira verde (inclusive `appointment_invariants_spec.rb` e `health_unit_invariants_spec.rb`); suíte completa com 0 falhas, em até ~3 min (mais que isso é regressão: `rspec --profile`).

- [ ] **Step 5: Busca por data fixa contra o relógio real**

```bash
/opt/homebrew/bin/git diff origin/main --name-only -- spec | xargs grep -nE "Time\.zone\.parse\(\"20|Date\.new\(20|\"20[0-9]{2}-[0-9]{2}-[0-9]{2}" 
```
Expected: toda ocorrência está dentro de `travel_to(...)` ou é um parâmetro de intervalo comparado só com outro literal (ex.: `invalid_range`). Qualquer outra vira data relativa.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add spec/invariants/professional_invariants_spec.rb
/opt/homebrew/bin/git commit -m "test: add professional invariants suite with mutation evidence

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 15: Revisão final do api

- [ ] **Step 1:** Um subagente revisor lê `origin/main..HEAD` (só código de produção e migração) contra a spec inteira, com atenção a:
  - ordem das rotas em `config/routes.rb` (literais antes de `:id`);
  - todo `DomainEvents.publish` novo sem dado sensível e declarado no initializer;
  - `HealthUnit.lock_active!` intacto em `CheckIn`, `CheckInByException` e `Close`;
  - N+1 em `GET /professionals` e na fila;
  - testes com data fixa contra o relógio real.
- [ ] **Step 2:** Corrija os achados, rode de novo a suíte completa (worker parado) e escreva o relatório da entrega: commits, contagem da suíte, evidência de mutação, specs existentes alteradas e por quê.
- [ ] **Step 3:** **Pare.** Merge, push, board e docs só com autorização explícita do usuário, uma etapa de cada vez. Ordem: api antes do dashboard; em staging e produção, perfis e vínculos são cadastrados antes de publicar a imagem com a fatia 4.
