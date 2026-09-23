# Canal web do cidadão (subprojeto 1) — plano

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** o cidadão faz a triagem pelo wpda, identificado por CPF declarado + telefone confirmado por SMS, gerando exatamente os mesmos registros e eventos que a triagem pelo WhatsApp gerava.

**Architecture:** três tabelas novas no banco de cada cidade (`citizens`, `citizen_sessions`, `otp_challenges`) e três colunas em `conversations` (`channel`, `citizen_id`, `last_answer_key`). Rotas novas `/citizen/*` (`CitizenApi::*`), com sessão própria por cookie `citizen_session`, chamam comandos novos em `Citizens::` que reaproveitam `GiveConsent`, `CompleteTriage` e `RevokeConsent`, os comandos onde o contrato de dados mora. A web **não** passa pelo `ConversationAdvance` (spec §3.2). O wpda ganha um fluxo em etapas, sem biblioteca de rotas.

**Tech Stack:** Rails 8.1, PostgreSQL, Solid Queue (apps/api); Vite + React 18 + Vitest + Testing Library (apps/wpda).

**Spec:** `docs/superpowers/specs/2026-09-22-web-citizen-channel-design.md`. Este plano implementa §2 a §5 dele.

**ADR:** `docs/adr/0017.md` (canal web do cidadão: estende `api` e `wpda`, sem aplicação nova). Comentários de código que citarem a decisão apontam para o ADR 0017.

## Global Constraints

- Commits: Conventional Commits **em inglês**, tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Git: `/opt/homebrew/bin/git` (o `git` do PATH aborta). `apps/api`, `apps/wpda` e `docs` são repositórios separados; a raiz do monorepo não é git. Branch `feat/web-citizen-channel` em `apps/api` e em `apps/wpda`. Nunca em `main`, nunca push.
- Staging de commit sempre explícito: nunca `git add -A` nem `git add .`.
- Comandos do api rodam no container, a partir da raiz do monorepo: `docker compose exec -T api bundle exec rspec <arquivos>`.
- Suíte completa do api: `docker compose stop worker`, depois `docker compose exec -T api bundle exec rspec`, depois `docker compose start worker` (sempre religar, mesmo em falha). Acima de ~3 min é regressão.
- Specs de request exigem `type: :request` (a inferência por pasta está desligada). O host padrão das specs de request é o de `TEST_CITY_A` (`spec/support/city_request_auth.rb`).
- **Versão da migração de cidade:** `20260923000001`. A `20260922000003` já existe (`allow_null_inbound_message_raw`, api 4e92622). Antes de criar, confira `ls db/city_migrate | tail -1`; se aparecer versão maior, use a seguinte e ajuste o `define(version:)` do dump.
- **Migração de cidade:** arquivo em `db/city_migrate`, e `db/city_schema.rb` atualizado **à mão**. Quem prova que os dois batem é `spec/services/city_schema_spec.rb`.
- Todo `encrypts` num modelo de cidade entra em `CityEncryption::CITY_KEYED_TARGETS` (`app/services/city_encryption.rb`); `spec/architecture/city_encrypted_attributes_guard_spec.rb` falha se faltar.
- Nunca grave CPF, telefone, código OTP ou resposta de triagem em log, exceto o `OtpSender::Log` de desenvolvimento, que mostra o código e o telefone mascarado.
- Valores fixos do spec: sessão de **30 dias** deslizante; código de **6 dígitos**, válido por **10 min**, **5 tentativas**, reenvio após **60 s**, **5 SMS por telefone por dia**; **10 CPFs por telefone**; texto livre até **500** caracteres; só celular brasileiro (+55, DDD, 9, oito dígitos).
- Nunca rode `start.sh`. Nunca desligue trigger de imutabilidade.

## Review Focus

1. **Resposta fora do esperado** (texto "banana" num passo booleano, opção que não existe num enum): o motor encerra o fluxo e pontua quando a resposta não tem ramo. A web precisa recusar com 422 **sem** gravar nada. Teste na Task 5 (`SubmitAnswer`) e na Task 7 (request).
2. **Toque duplo / duas abas** mandando a mesma resposta: a segunda não pode avançar a triagem. Com a mesma `idempotency_key`, devolve o estado atual; com outra chave depois de concluída, 409. Teste na Task 5.
3. **Um telefone acessando dado de outro** por id na URL (`citizen_id`, conversa, triagem): sempre 404, nunca 403 (não confirmar que o id existe). Teste na Task 7.
4. **Termo novo publicado no meio de uma triagem:** ao retomar, a conversa pede o consentimento da versão nova antes de seguir; sem isso o `CompleteTriage` recusaria com `:no_consent`. Teste na Task 5.
5. **Código de um telefone usado em outro, ou código já usado:** recusado. E o mesmo telefone pode ter conversa ativa no WhatsApp e na web ao mesmo tempo sem colidir no índice único. Testes nas Tasks 2 e 3.

---

## File Structure

**apps/api**
- Create: `app/services/citizen_identity.rb` (CPF e celular: normalizar, validar, mascarar)
- Create: `db/city_migrate/20260923000001_create_citizens.rb`; Modify: `db/city_schema.rb` (à mão)
- Create: `app/models/citizen.rb`, `app/models/citizen_session.rb`, `app/models/otp_challenge.rb`
- Modify: `app/models/conversation.rb`, `app/services/city_encryption.rb`, `app/models/current.rb`
- Create: `app/services/otp_sender.rb`; Modify: `config/environments/development.rb`, `config/environments/test.rb`
- Create: `app/commands/start_triage.rb`, `app/commands/undo_last_answer.rb`
- Modify: `app/commands/conversation_advance.rb`, `app/commands/give_consent.rb`, `app/jobs/notify_citizen_job.rb`
- Create: `app/commands/citizens/register_person.rb`, `app/commands/citizens/start_conversation.rb`, `app/commands/citizens/submit_answer.rb`, `app/commands/citizens/answer_validator.rb`, `app/commands/citizens/step_payload.rb`
- Create: `config/locales/citizen.pt-BR.yml`
- Create: `app/controllers/concerns/citizen_authentication.rb`, `app/controllers/citizen_api/base_controller.rb`, `app/controllers/citizen_api/otps_controller.rb`, `app/controllers/citizen_api/sessions_controller.rb`, `app/controllers/citizen_api/consent_terms_controller.rb`, `app/controllers/citizen_api/people_controller.rb`, `app/controllers/citizen_api/conversations_controller.rb`, `app/controllers/citizen_api/triages_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/filter_parameter_logging.rb`
- Specs: `spec/services/citizen_identity_spec.rb`, `spec/models/citizen_spec.rb`, `spec/models/citizen_session_spec.rb`, `spec/models/otp_challenge_spec.rb`, `spec/services/otp_sender_spec.rb`, `spec/commands/start_triage_spec.rb`, `spec/commands/undo_last_answer_spec.rb`, `spec/jobs/notify_citizen_job_spec.rb`, `spec/commands/citizens/*_spec.rb`, `spec/requests/citizen_api/*_spec.rb`, `spec/integration/web_channel_data_contract_spec.rb`, `spec/support/citizen_request_helpers.rb`

**apps/wpda**
- Modify: `package.json` (Testing Library), `vitest.config.ts`, `vite.config.ts` (proxy `/citizen`)
- Create: `src/lib/masks.ts`, `src/lib/citizenApi.ts` (+ testes)
- Create: `src/modules/citizen/ui.tsx` (botão, campo, tela), `PhoneStep.tsx`, `CodeStep.tsx`, `ConsentStep.tsx`, `PeopleStep.tsx`, `QuestionStep.tsx`, `ResultStep.tsx`, `HistoryStep.tsx`, `Flow.tsx` (+ testes)
- Modify: `src/App.tsx`

---

### Task 1: CPF e celular (`CitizenIdentity`)

**Repo:** `apps/api`. Antes de tudo: `/opt/homebrew/bin/git checkout -b feat/web-citizen-channel`.

**Files:**
- Create: `app/services/citizen_identity.rb`
- Test: `spec/services/citizen_identity_spec.rb`

**Interfaces:**
- Produces: `CitizenIdentity::Cpf.normalize(String) -> String(11 dígitos) | nil`, `CitizenIdentity::Cpf.mask(String) -> "***.982.247-**"`, `CitizenIdentity::Phone.normalize(String) -> "+55DDD9XXXXXXXX" | nil`, `CitizenIdentity::Phone.mask(String) -> "(**) *****-1234"`.

- [ ] **Step 1: Write the failing test**

```ruby
# spec/services/citizen_identity_spec.rb
require "rails_helper"

RSpec.describe CitizenIdentity do
  describe CitizenIdentity::Cpf do
    it "normaliza um CPF válido com ou sem pontuação" do
      expect(described_class.normalize("529.982.247-25")).to eq("52998224725")
      expect(described_class.normalize("52998224725")).to eq("52998224725")
    end

    it "recusa dígito verificador errado" do
      expect(described_class.normalize("529.982.247-24")).to be_nil
    end

    it "recusa sequência repetida, que passa na conta do dígito" do
      expect(described_class.normalize("111.111.111-11")).to be_nil
    end

    it "recusa tamanho errado e entrada vazia" do
      expect(described_class.normalize("5299822472")).to be_nil
      expect(described_class.normalize(nil)).to be_nil
    end

    it "mascara mostrando só os dígitos do meio" do
      expect(described_class.mask("52998224725")).to eq("***.982.247-**")
    end
  end

  describe CitizenIdentity::Phone do
    it "normaliza celular com ou sem +55 para E.164" do
      expect(described_class.normalize("(41) 99876-5432")).to eq("+5541998765432")
      expect(described_class.normalize("+55 41 99876-5432")).to eq("+5541998765432")
    end

    it "recusa fixo (sem o 9), DDD inválido e tamanho errado" do
      expect(described_class.normalize("(41) 3333-4444")).to be_nil
      expect(described_class.normalize("(01) 99876-5432")).to be_nil
      expect(described_class.normalize("99876-5432")).to be_nil
    end

    it "mascara mostrando só os 4 últimos dígitos" do
      expect(described_class.mask("+5541998765432")).to eq("(**) *****-5432")
    end
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec -T api bundle exec rspec spec/services/citizen_identity_spec.rb`
Expected: FAIL com `uninitialized constant CitizenIdentity`.

- [ ] **Step 3: Write minimal implementation**

```ruby
# app/services/citizen_identity.rb
# Identidade DECLARADA do cidadão no canal web (spec 2026-09-22-web-citizen-
# channel §2): o CPF só passa pela conta do dígito verificador, e o celular só
# pelo formato. Quem prova posse do celular é o OTP; quem prova o CPF é a
# validação presencial (subprojeto 2).
module CitizenIdentity
  module Cpf
    module_function

    def normalize(input)
      digits = input.to_s.gsub(/\D/, "")
      return nil unless digits.length == 11
      return nil if digits.chars.uniq.size == 1
      return nil unless check_digits_ok?(digits)

      digits
    end

    def mask(digits)
      "***.#{digits[3, 3]}.#{digits[6, 3]}-**"
    end

    def check_digits_ok?(digits)
      nums = digits.chars.map(&:to_i)
      first = check_digit(nums[0, 9])
      second = check_digit(nums[0, 9] + [first])
      nums[9] == first && nums[10] == second
    end

    def check_digit(nums)
      start = nums.size + 1
      sum = nums.each_with_index.sum { |n, i| n * (start - i) }
      rest = (sum * 10) % 11
      rest == 10 ? 0 : rest
    end
  end

  module Phone
    module_function

    # Celular brasileiro: DDD (dois dígitos de 1 a 9) + 9 + oito dígitos.
    MOBILE = /\A[1-9][1-9]9\d{8}\z/

    def normalize(input)
      digits = input.to_s.gsub(/\D/, "")
      digits = digits[2..] if digits.length == 13 && digits.start_with?("55")
      return nil unless digits.match?(MOBILE)

      "+55#{digits}"
    end

    def mask(e164)
      "(**) *****-#{e164.to_s[-4..]}"
    end
  end
end
```

- [ ] **Step 4: Run test to verify it passes**

Run: `docker compose exec -T api bundle exec rspec spec/services/citizen_identity_spec.rb`
Expected: PASS (8 examples).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/services/citizen_identity.rb spec/services/citizen_identity_spec.rb
/opt/homebrew/bin/git commit -m "feat: validate and mask declared CPF and mobile numbers" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Tabelas do cidadão e canal na conversa

**Repo:** `apps/api`.

**Files:**
- Create: `db/city_migrate/20260923000001_create_citizens.rb`
- Modify: `db/city_schema.rb` (à mão)
- Create: `app/models/citizen.rb`, `app/models/citizen_session.rb`
- Modify: `app/models/conversation.rb`, `app/services/city_encryption.rb`, `app/models/current.rb`
- Test: `spec/models/citizen_spec.rb`, `spec/models/citizen_session_spec.rb`, `spec/models/conversation_channel_spec.rb`

**Interfaces:**
- Consumes: `CitizenIdentity::Cpf.mask` (Task 1).
- Produces: `Citizen` (`cpf`, `phone`, `verification_level_declared?`, `cpf_masked`, `MAX_PER_PHONE = 10`, `has_many :conversations`); `CitizenSession.start!(phone:) -> [CitizenSession, String token]`, `CitizenSession.resume(token) -> CitizenSession | nil` (desliza o prazo), `#revoke!`, `#citizens -> Citizen relation`, `TTL = 30.days`; `Conversation` com `enum :channel` (`channel_web?`, `channel_whatsapp?`), `belongs_to :citizen, optional: true`, `last_answer_key`; `Current.citizen_session`.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/models/citizen_spec.rb
require "rails_helper"

RSpec.describe Citizen do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  it "nasce declarado e acha pelo CPF e telefone cifrados" do
    citizen = described_class.create!(cpf: "52998224725", phone: "+5541998765432")
    expect(citizen).to be_verification_level_declared
    expect(described_class.find_by(cpf: "52998224725", phone: "+5541998765432")).to eq(citizen)
    raw = described_class.connection.select_value("SELECT cpf FROM citizens WHERE id = '#{citizen.id}'")
    expect(raw).not_to include("52998224725")
  end

  it "não repete o par (CPF, telefone), mas aceita o mesmo CPF em outro telefone" do
    described_class.create!(cpf: "52998224725", phone: "+5541998765432")
    expect { described_class.create!(cpf: "52998224725", phone: "+5541998765432") }
      .to raise_error(ActiveRecord::RecordNotUnique)
    expect { described_class.create!(cpf: "52998224725", phone: "+5541911112222") }.not_to raise_error
  end

  it "mascara o CPF" do
    expect(described_class.new(cpf: "52998224725").cpf_masked).to eq("***.982.247-**")
  end
end
```

```ruby
# spec/models/citizen_session_spec.rb
require "rails_helper"

RSpec.describe CitizenSession do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; travel_back }

  it "guarda só o hash do token e retoma a sessão pelo token" do
    session, token = described_class.start!(phone: "+5541998765432")
    expect(session.token_digest).not_to eq(token)
    expect(described_class.resume(token)).to eq(session)
    expect(described_class.resume("outro")).to be_nil
    expect(described_class.resume(nil)).to be_nil
  end

  it "expira em 30 dias sem uso e desliza o prazo quando usada" do
    session, token = described_class.start!(phone: "+5541998765432")
    travel 29.days
    expect(described_class.resume(token)).to eq(session)
    travel 29.days
    expect(described_class.resume(token)).to eq(session)
    travel 31.days
    expect(described_class.resume(token)).to be_nil
  end

  it "não retoma sessão revogada" do
    session, token = described_class.start!(phone: "+5541998765432")
    session.revoke!
    expect(described_class.resume(token)).to be_nil
  end

  it "lista as pessoas do telefone da sessão, e só elas" do
    session, = described_class.start!(phone: "+5541998765432")
    mine = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    Citizen.create!(cpf: "52998224725", phone: "+5541911112222")
    expect(session.citizens).to contain_exactly(mine)
  end
end
```

```ruby
# spec/models/conversation_channel_spec.rb
require "rails_helper"

RSpec.describe Conversation do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }

  it "nasce no canal whatsapp" do
    expect(described_class.create!(phone: "+5541998765432", state: :greeting)).to be_channel_whatsapp
  end

  it "o mesmo telefone tem uma conversa ativa no WhatsApp e outra na web sem colidir" do
    described_class.create!(phone: citizen.phone, state: :consented)
    expect {
      described_class.create!(phone: citizen.phone, state: :consented, channel: "web", citizen: citizen)
    }.not_to raise_error
  end

  it "Conversation.for (WhatsApp) ignora a conversa da web do mesmo telefone" do
    web = described_class.create!(phone: citizen.phone, state: :consented, channel: "web", citizen: citizen)
    whatsapp = described_class.for(citizen.phone)
    expect(whatsapp).not_to eq(web)
    expect(whatsapp).to be_channel_whatsapp
  end

  it "um cidadão tem no máximo uma conversa ativa na web" do
    described_class.create!(phone: citizen.phone, state: :consented, channel: "web", citizen: citizen)
    expect {
      described_class.create!(phone: citizen.phone, state: :awaiting_consent, channel: "web", citizen: citizen)
    }.to raise_error(ActiveRecord::RecordNotUnique)
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/models/citizen_spec.rb spec/models/citizen_session_spec.rb spec/models/conversation_channel_spec.rb`
Expected: FAIL com `uninitialized constant Citizen`.

- [ ] **Step 3: Migração**

```ruby
# db/city_migrate/20260923000001_create_citizens.rb
# Canal web do cidadão (spec 2026-09-22-web-citizen-channel §3.3).
#
# Aditiva. CPF e telefone vão cifrados de forma determinística com a chave da
# cidade (a busca por eles precisa do mesmo texto cifrado), como
# conversations.phone. Os checks usam a forma ANY (ARRAY[...]::text[]), a
# única que sobrevive ao round-trip migração → dump → load (ver
# 20260918000001_create_protocol_signatures.rb).
#
# O índice de conversa ativa se divide em dois: por telefone no WhatsApp (o
# que já existia) e por cidadão na web. Assim o mesmo celular tem uma conversa
# ativa em cada canal, e um celular da família tem uma por pessoa na web.
class CreateCitizens < ActiveRecord::Migration[8.1]
  ACTIVE_STATES = "state::text = ANY (ARRAY['greeting', 'awaiting_consent', 'consented']::text[])".freeze

  def up
    create_table :citizens, id: :uuid do |t|
      t.string :cpf, null: false
      t.string :phone, null: false
      t.string :verification_level, null: false, default: "declared"
      t.timestamps
    end
    add_index :citizens, [:cpf, :phone], unique: true
    add_index :citizens, :phone
    add_check_constraint :citizens, "verification_level::text = ANY (ARRAY['declared', 'verified']::text[])",
                         name: "ck_citizens_verification_level"

    create_table :citizen_sessions, id: :uuid do |t|
      t.string :token_digest, null: false
      t.string :phone, null: false
      t.datetime :expires_at, null: false
      t.datetime :last_seen_at
      t.datetime :revoked_at
      t.timestamps
    end
    add_index :citizen_sessions, :token_digest, unique: true
    add_index :citizen_sessions, :phone

    create_table :otp_challenges, id: :uuid do |t|
      t.string :phone, null: false
      t.string :code_digest, null: false
      t.integer :attempts, null: false, default: 0
      t.datetime :expires_at, null: false
      t.datetime :consumed_at
      t.timestamps
    end
    add_index :otp_challenges, [:phone, :created_at]

    add_column :conversations, :channel, :string, null: false, default: "whatsapp"
    add_reference :conversations, :citizen, type: :uuid, foreign_key: true, index: true
    add_column :conversations, :last_answer_key, :string
    add_check_constraint :conversations, "channel::text = ANY (ARRAY['whatsapp', 'web']::text[])",
                         name: "ck_conversations_channel"

    remove_index :conversations, name: "idx_conversations_active_phone"
    add_index :conversations, :phone, unique: true, name: "idx_conversations_active_phone",
              where: "channel::text = 'whatsapp'::text AND #{ACTIVE_STATES}"
    add_index :conversations, :citizen_id, unique: true, name: "idx_conversations_active_citizen",
              where: "channel::text = 'web'::text AND #{ACTIVE_STATES}"
  end

  def down
    remove_index :conversations, name: "idx_conversations_active_citizen"
    remove_index :conversations, name: "idx_conversations_active_phone"
    add_index :conversations, :phone, unique: true, name: "idx_conversations_active_phone",
              where: ACTIVE_STATES
    remove_check_constraint :conversations, name: "ck_conversations_channel"
    remove_column :conversations, :last_answer_key
    remove_reference :conversations, :citizen, foreign_key: true, index: true
    remove_column :conversations, :channel
    drop_table :otp_challenges
    drop_table :citizen_sessions
    drop_table :citizens
  end
end
```

- [ ] **Step 4: Atualizar o dump à mão**

Em `db/city_schema.rb`:

1. suba `define(version: ...)` para `2026_09_23_000001`;
2. antes de `create_table "consent_terms"` (ordem alfabética), acrescente:

```ruby
  create_table "citizen_sessions", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "created_at", null: false
    t.datetime "expires_at", null: false
    t.datetime "last_seen_at"
    t.string "phone", null: false
    t.datetime "revoked_at"
    t.string "token_digest", null: false
    t.datetime "updated_at", null: false
    t.index ["phone"], name: "index_citizen_sessions_on_phone"
    t.index ["token_digest"], name: "index_citizen_sessions_on_token_digest", unique: true
  end

  create_table "citizens", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.string "cpf", null: false
    t.datetime "created_at", null: false
    t.string "phone", null: false
    t.datetime "updated_at", null: false
    t.string "verification_level", default: "declared", null: false
    t.index ["cpf", "phone"], name: "index_citizens_on_cpf_and_phone", unique: true
    t.index ["phone"], name: "index_citizens_on_phone"
    t.check_constraint "(verification_level)::text = ANY (ARRAY['declared'::text, 'verified'::text])", name: "ck_citizens_verification_level"
  end
```

3. substitua o bloco `create_table "conversations"` inteiro por:

```ruby
  create_table "conversations", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.string "channel", default: "whatsapp", null: false
    t.uuid "citizen_id"
    t.datetime "created_at", null: false
    t.string "last_answer_key"
    t.string "phone", null: false
    t.string "state", default: "greeting", null: false
    t.datetime "updated_at", null: false
    t.index ["citizen_id"], name: "idx_conversations_active_citizen", unique: true, where: "(((channel)::text = 'web'::text) AND ((state)::text = ANY (ARRAY['greeting'::text, 'awaiting_consent'::text, 'consented'::text])))"
    t.index ["citizen_id"], name: "index_conversations_on_citizen_id"
    t.index ["phone"], name: "idx_conversations_active_phone", unique: true, where: "(((channel)::text = 'whatsapp'::text) AND ((state)::text = ANY (ARRAY['greeting'::text, 'awaiting_consent'::text, 'consented'::text])))"
    t.index ["state"], name: "index_conversations_on_state"
    t.check_constraint "(channel)::text = ANY (ARRAY['whatsapp'::text, 'web'::text])", name: "ck_conversations_channel"
  end
```

4. antes de `create_table "outbound_messages"` ("ot" < "ou" na ordem alfabética), acrescente:

```ruby
  create_table "otp_challenges", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.integer "attempts", default: 0, null: false
    t.string "code_digest", null: false
    t.datetime "consumed_at"
    t.datetime "created_at", null: false
    t.datetime "expires_at", null: false
    t.string "phone", null: false
    t.datetime "updated_at", null: false
    t.index ["phone", "created_at"], name: "index_otp_challenges_on_phone_and_created_at"
  end
```

5. na lista de `add_foreign_key` do fim do arquivo, em ordem alfabética: `add_foreign_key "conversations", "citizens"`.

O texto exato dos `where:` e `check_constraint` é o que o Postgres devolve, não o que a migração escreveu. **O juiz é a spec de paridade** (Step 7): se ela falhar, rode a migração num banco de cidade de dev (`docker compose exec -T api bin/rails city:migrate:all`) e copie o texto real com `SELECT indexdef FROM pg_indexes WHERE tablename = 'conversations'` e `SELECT conname, pg_get_constraintdef(oid) FROM pg_constraint WHERE conrelid IN ('citizens'::regclass, 'conversations'::regclass)`.

- [ ] **Step 5: Modelos**

```ruby
# app/models/citizen.rb
# Cidadão do canal web: o PAR (CPF, telefone), no banco da cidade. Ver spec
# 2026-09-22-web-citizen-channel §2. Um telefone serve a vários CPFs (a família
# que divide um celular) e um CPF aparece em vários telefones; quem junta os
# pares é a validação presencial (subprojeto 2).
class Citizen < ApplicationRecord
  MAX_PER_PHONE = 10

  encrypts :cpf,   deterministic: true, key_provider: CityDeterministicKeyProvider.new
  encrypts :phone, deterministic: true, key_provider: CityDeterministicKeyProvider.new

  has_many :conversations, dependent: :restrict_with_error

  enum :verification_level, { declared: "declared", verified: "verified" }, prefix: true

  validates :cpf, :phone, presence: true

  def cpf_masked
    CitizenIdentity::Cpf.mask(cpf)
  end
end
```

```ruby
# app/models/citizen_session.rb
# Sessão do cidadão no canal web. Pertence ao TELEFONE confirmado por OTP, não
# a um CPF: a escolha da pessoa vem depois (spec §2.6). Guarda só o hash do
# token; o token viaja no cookie assinado `citizen_session`.
class CitizenSession < ApplicationRecord
  TTL = 30.days
  SLIDE_EVERY = 1.hour

  encrypts :phone, deterministic: true, key_provider: CityDeterministicKeyProvider.new

  def self.digest(token)
    OpenSSL::Digest::SHA256.hexdigest(token)
  end

  def self.start!(phone:)
    token = SecureRandom.urlsafe_base64(32)
    session = create!(phone: phone, token_digest: digest(token),
                      expires_at: TTL.from_now, last_seen_at: Time.current)
    [session, token]
  end

  def self.resume(token)
    return nil if token.blank?

    session = find_by(token_digest: digest(token))
    return nil unless session&.usable?

    session.slide!
    session
  end

  def usable?
    revoked_at.nil? && expires_at.future?
  end

  # Prazo deslizante (spec §2.6), gravado no máximo uma vez por hora.
  def slide!
    return if last_seen_at && last_seen_at > SLIDE_EVERY.ago

    update!(last_seen_at: Time.current, expires_at: TTL.from_now)
  end

  def revoke!
    update!(revoked_at: Time.current)
  end

  def citizens
    Citizen.where(phone: phone)
  end
end
```

Em `app/models/conversation.rb`, depois de `has_many :consents`:

```ruby
  belongs_to :citizen, optional: true

  # Canal de entrada (spec 2026-09-22-web-citizen-channel §3.3). No WhatsApp a
  # conversa é do telefone; na web, do cidadão (par CPF + telefone).
  enum :channel, { whatsapp: "whatsapp", web: "web" }, prefix: true
```

e troque `self.for` para olhar só o WhatsApp:

```ruby
  ACTIVE_STATES = %w[greeting awaiting_consent consented].freeze

  def self.for(phone)
    channel_whatsapp.where(phone: phone, state: ACTIVE_STATES).first ||
      create!(phone: phone, state: :greeting, channel: "whatsapp")
  rescue ActiveRecord::RecordNotUnique
    channel_whatsapp.where(phone: phone, state: ACTIVE_STATES).first!
  end
```

Em `app/services/city_encryption.rb`, acrescente a `CITY_KEYED_TARGETS` (a linha de `OtpChallenge` entra na Task 3, junto com o modelo):

```ruby
    [ Citizen,        :cpf ],
    [ Citizen,        :phone ],
    [ CitizenSession, :phone ],
```

Em `app/models/current.rb`, junto dos outros atributos:

```ruby
  # Sessão do cidadão no canal web (CitizenAuthentication). Nunca coexiste com
  # `session` (servidor da cidade): são cookies e tabelas diferentes.
  attribute :citizen_session
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `docker compose exec -T api bundle exec rspec spec/models/citizen_spec.rb spec/models/citizen_session_spec.rb spec/models/conversation_channel_spec.rb`
Expected: PASS.

- [ ] **Step 7: Paridade do schema, guarda de cifra e regressão do WhatsApp**

Run: `docker compose exec -T api bundle exec rspec spec/services/city_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/commands/conversation_advance_spec.rb spec/jobs/process_inbound_message_job_spec.rb spec/requests/webhooks`
Expected: PASS. Se a paridade falhar, ajuste o dump conforme o Step 4.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add db/city_migrate/20260923000001_create_citizens.rb db/city_schema.rb app/models/citizen.rb app/models/citizen_session.rb app/models/conversation.rb app/models/current.rb app/services/city_encryption.rb spec/models/citizen_spec.rb spec/models/citizen_session_spec.rb spec/models/conversation_channel_spec.rb
/opt/homebrew/bin/git commit -m "feat: add citizens, citizen sessions and a channel per conversation" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Código por SMS (`OtpChallenge` + `OtpSender`)

**Repo:** `apps/api`.

**Files:**
- Create: `app/models/otp_challenge.rb`, `app/services/otp_sender.rb`
- Modify: `app/services/city_encryption.rb`, `config/environments/development.rb`, `config/environments/test.rb`
- Test: `spec/models/otp_challenge_spec.rb`, `spec/services/otp_sender_spec.rb`

**Interfaces:**
- Consumes: `CitizenIdentity::Phone.mask` (Task 1); tabela `otp_challenges` (Task 2).
- Produces: `OtpChallenge.issue!(phone:) -> [OtpChallenge, String code]` (levanta `OtpChallenge::TooSoon` ou `OtpChallenge::DailyLimit`); `OtpChallenge.verify(phone:, code:) -> :ok | :invalid | :expired | :exhausted | :missing`; `OtpChallenge::RESEND_AFTER`; `OtpSender.deliver(phone:, code:)` (levanta `OtpSender::Unavailable`); `OtpSender::Test.deliveries -> Array<{phone:, code:}>` e `OtpSender::Test.reset!`.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/models/otp_challenge_spec.rb
require "rails_helper"

RSpec.describe OtpChallenge do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; travel_back }

  let(:phone) { "+5541998765432" }

  it "emite um código de 6 dígitos e guarda só o hash" do
    challenge, code = described_class.issue!(phone: phone)
    expect(code).to match(/\A\d{6}\z/)
    expect(challenge.code_digest).not_to include(code)
  end

  it "aceita o código certo uma vez só" do
    _, code = described_class.issue!(phone: phone)
    expect(described_class.verify(phone: phone, code: code)).to eq(:ok)
    expect(described_class.verify(phone: phone, code: code)).to eq(:missing)
  end

  it "recusa o código de um telefone usado em outro" do
    _, code = described_class.issue!(phone: phone)
    expect(described_class.verify(phone: "+5541911112222", code: code)).to eq(:missing)
  end

  it "conta tentativas erradas e esgota na quinta" do
    _, code = described_class.issue!(phone: phone)
    wrong = code == "000000" ? "111111" : "000000"
    5.times { expect(described_class.verify(phone: phone, code: wrong)).to eq(:invalid) }
    expect(described_class.verify(phone: phone, code: code)).to eq(:exhausted)
  end

  it "vence em 10 minutos" do
    _, code = described_class.issue!(phone: phone)
    travel 11.minutes
    expect(described_class.verify(phone: phone, code: code)).to eq(:expired)
  end

  it "só reenvia depois de 60 s" do
    described_class.issue!(phone: phone)
    expect { described_class.issue!(phone: phone) }.to raise_error(OtpChallenge::TooSoon)
    travel 61.seconds
    expect { described_class.issue!(phone: phone) }.not_to raise_error
  end

  it "envia no máximo 5 códigos por telefone em 24 h" do
    5.times do
      described_class.issue!(phone: phone)
      travel 61.seconds
    end
    expect { described_class.issue!(phone: phone) }.to raise_error(OtpChallenge::DailyLimit)
    travel 24.hours
    expect { described_class.issue!(phone: phone) }.not_to raise_error
  end

  it "vale o código mais recente: um reenvio invalida o anterior" do
    _, first = described_class.issue!(phone: phone)
    travel 61.seconds
    _, second = described_class.issue!(phone: phone)
    skip "códigos coincidiram" if first == second
    expect(described_class.verify(phone: phone, code: first)).to eq(:invalid)
    expect(described_class.verify(phone: phone, code: second)).to eq(:ok)
  end
end
```

```ruby
# spec/services/otp_sender_spec.rb
require "rails_helper"

RSpec.describe OtpSender do
  after { described_class::Test.reset! }

  it "em teste, guarda o envio em memória" do
    described_class.deliver(phone: "+5541998765432", code: "123456")
    expect(described_class::Test.deliveries).to eq([{ phone: "+5541998765432", code: "123456" }])
  end

  it "sem provedor configurado, levanta Unavailable" do
    allow(Rails.configuration.x).to receive(:otp_sender).and_return(nil)
    expect { described_class.deliver(phone: "+5541998765432", code: "123456") }
      .to raise_error(OtpSender::Unavailable)
  end

  it "o de log mostra o telefone mascarado" do
    allow(Rails.configuration.x).to receive(:otp_sender).and_return(:log)
    expect(Rails.logger).to receive(:info).with("[otp] (**) *****-5432 code=123456")
    described_class.deliver(phone: "+5541998765432", code: "123456")
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/models/otp_challenge_spec.rb spec/services/otp_sender_spec.rb`
Expected: FAIL com `uninitialized constant OtpChallenge`.

- [ ] **Step 3: Implementação**

```ruby
# app/models/otp_challenge.rb
# Código de confirmação do celular (spec 2026-09-22-web-citizen-channel §4.2).
# O SMS custa dinheiro: os limites daqui existem para que ninguém infle envios.
# `created_at` faz o papel de "enviado em".
class OtpChallenge < ApplicationRecord
  TTL = 10.minutes
  MAX_ATTEMPTS = 5
  RESEND_AFTER = 60.seconds
  DAILY_LIMIT = 5

  class TooSoon < StandardError; end
  class DailyLimit < StandardError; end

  encrypts :phone, deterministic: true, key_provider: CityDeterministicKeyProvider.new

  def self.issue!(phone:)
    recent = where(phone: phone).where("created_at > ?", 24.hours.ago)
    raise DailyLimit if recent.count >= DAILY_LIMIT
    raise TooSoon if recent.where("created_at > ?", RESEND_AFTER.ago).exists?

    code = format("%06d", SecureRandom.random_number(1_000_000))
    challenge = create!(phone: phone, code_digest: digest(phone, code), expires_at: TTL.from_now)
    [challenge, code]
  end

  # Só o desafio mais recente e não usado do telefone vale: um reenvio
  # invalida o anterior.
  def self.verify(phone:, code:)
    challenge = where(phone: phone, consumed_at: nil).order(created_at: :desc).first
    return :missing unless challenge

    result = nil
    challenge.with_lock do
      result =
        if challenge.expires_at.past? then :expired
        elsif challenge.attempts >= MAX_ATTEMPTS then :exhausted
        elsif ActiveSupport::SecurityUtils.secure_compare(challenge.code_digest, digest(phone, code.to_s))
          challenge.update!(consumed_at: Time.current)
          :ok
        else
          challenge.increment!(:attempts)
          :invalid
        end
    end
    result
  end

  # HMAC com chave derivada do secret_key_base: um dump do banco sozinho não
  # basta para testar os 10^6 códigos possíveis.
  def self.digest(phone, code)
    key = Rails.application.key_generator.generate_key("citizen-otp", 32)
    OpenSSL::HMAC.hexdigest("SHA256", key, "#{phone}:#{code}")
  end
end
```

```ruby
# app/services/otp_sender.rb
# Entrega do código de confirmação (spec 2026-09-22-web-citizen-channel §2.5).
# Um provedor só, da plataforma, escolhido por `config.x.otp_sender`. O
# provedor real de SMS é pendência de go-live (spec §7): sem ele, o envio
# levanta Unavailable e a API responde 503.
module OtpSender
  class Unavailable < StandardError; end

  def self.deliver(phone:, code:)
    backend.deliver(phone: phone, code: code)
  end

  def self.backend
    case Rails.configuration.x.otp_sender
    when :log  then Log
    when :test then Test
    else Unconfigured
    end
  end

  # Desenvolvimento: o código aparece no log do api.
  module Log
    def self.deliver(phone:, code:)
      Rails.logger.info("[otp] #{CitizenIdentity::Phone.mask(phone)} code=#{code}")
    end
  end

  module Test
    def self.deliveries
      @deliveries ||= []
    end

    def self.deliver(phone:, code:)
      deliveries << { phone: phone, code: code }
    end

    def self.reset!
      deliveries.clear
    end
  end

  module Unconfigured
    def self.deliver(**)
      raise Unavailable, "no SMS provider configured"
    end
  end
end
```

Em `config/environments/development.rb`, dentro do bloco `configure`: `config.x.otp_sender = :log`.
Em `config/environments/test.rb`, dentro do bloco `configure`: `config.x.otp_sender = :test`.
Em `app/services/city_encryption.rb`, acrescente `[ OtpChallenge, :phone ],` a `CITY_KEYED_TARGETS`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `docker compose exec -T api bundle exec rspec spec/models/otp_challenge_spec.rb spec/services/otp_sender_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/models/otp_challenge.rb app/services/otp_sender.rb app/services/city_encryption.rb config/environments/development.rb config/environments/test.rb spec/models/otp_challenge_spec.rb spec/services/otp_sender_spec.rb
/opt/homebrew/bin/git commit -m "feat: issue and verify SMS confirmation codes behind an OtpSender" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Domínio neutro de canal (`StartTriage`, `UndoLastAnswer`, consentimento e aviso)

**Repo:** `apps/api`.

**Files:**
- Create: `app/commands/start_triage.rb`, `app/commands/undo_last_answer.rb`
- Modify: `app/commands/conversation_advance.rb`, `app/commands/give_consent.rb`, `app/jobs/notify_citizen_job.rb`
- Test: `spec/commands/start_triage_spec.rb`, `spec/commands/undo_last_answer_spec.rb`, `spec/jobs/notify_citizen_job_spec.rb`, `spec/commands/give_consent_channel_spec.rb`

**Interfaces:**
- Consumes: `Conversation#channel_web?` (Task 2).
- Produces: `StartTriage.call(conversation:) -> Result ok(triage:) | fail(:no_protocol)`, `StartTriage::DEFAULT_PROTOCOL_NAME`; `UndoLastAnswer.call(triage:) -> Result ok(triage:) | fail(:not_in_progress | :nothing_to_undo)`; `GiveConsent.call(conversation:, version:, evidence:, channel: "whatsapp")`.

Todas as specs desta task usam o mesmo protocolo de dois passos. Crie o helper antes:

```ruby
# spec/support/triage_protocol_helpers.rb
# Protocolo de dois passos booleanos, o mesmo de config/city_templates/
# triage_respiratoria.json: tosse (true → febre, false → fim) e febre (fim).
module TriageProtocolHelpers
  def create_default_protocol!
    ProtocolDefinition.create!(
      name: StartTriage::DEFAULT_PROTOCOL_NAME, version: 1, status: "active",
      definition: {
        "name" => StartTriage::DEFAULT_PROTOCOL_NAME, "version" => 1, "start_step_id" => "tosse",
        "steps" => [
          { "id" => "tosse", "prompt" => "Você está com tosse?", "answer_type" => "boolean",
            "branches" => { "true" => "febre", "false" => nil }, "weights" => { "true" => 3, "false" => 0 } },
          { "id" => "febre", "prompt" => "Está com febre alta?", "answer_type" => "boolean",
            "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 5, "false" => 0 } }
        ],
        "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "alta" => 5 },
                       "priority_map" => { "baixa" => 9, "alta" => 1 } }
      }
    )
  end
end

RSpec.configure { |c| c.include TriageProtocolHelpers }
```

Confira que `spec/rails_helper.rb` carrega `spec/support/**/*.rb` (`Dir[Rails.root.join("spec/support/**/*.rb")].each { |f| require f }`); se carregar arquivo por arquivo, acrescente este.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/commands/start_triage_spec.rb
require "rails_helper"

RSpec.describe StartTriage do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:conversation) { Conversation.create!(phone: "+5541998765432", state: :consented) }

  it "abre a triagem no primeiro passo do protocolo ativo" do
    create_default_protocol!
    result = described_class.call(conversation: conversation)
    expect(result).to be_ok
    triage = result.payload[:triage]
    expect(triage).to be_status_in_progress
    expect(triage.current_step).to eq("tosse")
    expect(triage.answers).to eq({})
  end

  it "falha com :no_protocol sem protocolo ativo" do
    expect(described_class.call(conversation: conversation).reason).to eq(:no_protocol)
  end
end
```

```ruby
# spec/commands/undo_last_answer_spec.rb
require "rails_helper"

RSpec.describe UndoLastAnswer do
  before do
    Current.city = TEST_CITY_A
    create_default_protocol!
    GiveConsent.call(conversation: conversation, version: Consents.current_version, evidence: {})
  end
  after { Current.reset; Rails.cache.clear }

  let(:conversation) { Conversation.create!(phone: "+5541998765432", state: :awaiting_consent) }
  let(:triage) { StartTriage.call(conversation: conversation).payload[:triage] }

  it "tira a última resposta e volta ao passo dela" do
    CompleteTriage.call(triage: triage, answer: "true")
    expect(triage.reload.current_step).to eq("febre")

    result = described_class.call(triage: triage)
    expect(result).to be_ok
    expect(triage.reload.answers).to eq({})
    expect(triage.current_step).to eq("tosse")
  end

  it "desfazer e responder de novo a mesma coisa leva ao mesmo estado" do
    CompleteTriage.call(triage: triage, answer: "true")
    before_undo = triage.reload.attributes.slice("answers", "current_step", "status")
    described_class.call(triage: triage)
    CompleteTriage.call(triage: triage.reload, answer: "true")
    expect(triage.reload.attributes.slice("answers", "current_step", "status")).to eq(before_undo)
  end

  it "recusa sem resposta a desfazer" do
    expect(described_class.call(triage: triage).reason).to eq(:nothing_to_undo)
  end

  it "recusa depois da triagem concluída" do
    CompleteTriage.call(triage: triage, answer: "true")
    CompleteTriage.call(triage: triage.reload, answer: "true")
    expect(triage.reload).to be_status_completed
    expect(described_class.call(triage: triage).reason).to eq(:not_in_progress)
    expect(triage.reload.answers).to eq("tosse" => "true", "febre" => "true")
  end
end
```

```ruby
# spec/commands/give_consent_channel_spec.rb
require "rails_helper"

RSpec.describe GiveConsent do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:conversation) { Conversation.create!(phone: "+5541998765432", state: :awaiting_consent) }

  it "grava whatsapp quando o canal não é informado" do
    described_class.call(conversation: conversation, version: Consents.current_version, evidence: {})
    expect(conversation.consents.last.channel).to eq("whatsapp")
  end

  it "grava o canal informado" do
    described_class.call(conversation: conversation, version: Consents.current_version, evidence: {}, channel: "web")
    expect(conversation.consents.last.channel).to eq("web")
  end
end
```

```ruby
# spec/jobs/notify_citizen_job_spec.rb
require "rails_helper"

RSpec.describe NotifyCitizenJob do
  include ActiveJob::TestHelper

  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  def completed_triage(conversation)
    pd = ProtocolDefinition.create!(
      name: "notify-spec", version: 1, status: "active",
      definition: { "name" => "notify-spec", "version" => 1, "start_step_id" => "s1",
                    "steps" => [{ "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
                                  "branches" => { "true" => nil, "false" => nil } }] }
    )
    triage = Triage.create!(conversation: conversation, protocol_definition: pd, protocol_name: "notify-spec",
                            status: "completed", tier: "alta", priority: 1, completed_at: Time.current,
                            outcome: { "trail" => [] })
    token = ReportSnapshot.mint_token
    ReportSnapshot.create!(triage: triage, protocol_definition: pd, outcome: { "tier" => "alta" },
                           payload: { "tier" => "alta" }, token: token,
                           signature: ReportSnapshot.sign(token), expires_at: 30.days.from_now)
    triage
  end

  it "no WhatsApp, manda o link do relatório" do
    triage = completed_triage(Conversation.create!(phone: "+5541998765432", state: :completed))
    expect { described_class.new.handle(triage_id: triage.id) }.to have_enqueued_job(SendWhatsappJob)
  end

  it "na web, não manda nada: o link aparece na tela" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    conversation = Conversation.create!(phone: citizen.phone, state: :completed, channel: "web", citizen: citizen)
    triage = completed_triage(conversation)
    expect { described_class.new.handle(triage_id: triage.id) }.not_to have_enqueued_job(SendWhatsappJob)
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/commands/start_triage_spec.rb spec/commands/undo_last_answer_spec.rb spec/commands/give_consent_channel_spec.rb spec/jobs/notify_citizen_job_spec.rb`
Expected: FAIL (`uninitialized constant StartTriage`, `unknown keyword: :channel`, e o job da web enfileirando envio).

- [ ] **Step 3: Implementação**

```ruby
# app/commands/start_triage.rb
# Abre a triagem do protocolo padrão numa conversa já consentida. Usado pelo
# WhatsApp (ConversationAdvance) e pela web (Citizens::StartConversation).
# Ver ADR-0009. Reasons: :no_protocol.
class StartTriage
  DEFAULT_PROTOCOL_NAME = "triage-respiratoria"

  def self.call(conversation:)
    record = ProtocolDefinition.where(name: DEFAULT_PROTOCOL_NAME, status: "active").first
    return Result.fail(:no_protocol) unless record

    engine = Protocols.current(name: DEFAULT_PROTOCOL_NAME)
    triage = conversation.triages.create!(
      protocol_definition: record,
      protocol_name: record.name,
      answers: {},
      current_step: engine.start_step_id.to_s,
      status: :in_progress
    )
    Result.ok(triage: triage)
  rescue Protocols::NotFound
    Result.fail(:no_protocol)
  end
end
```

```ruby
# app/commands/undo_last_answer.rb
# "Voltar" do canal web (spec 2026-09-22-web-citizen-channel §2.8): tira a
# última resposta e volta ao passo dela. O caminho é recalculado pelo motor
# (função pura) sobre as respostas gravadas, então o passo desfeito é sempre o
# último da trilha. Depois de concluída, a triagem é imutável.
# Reasons: :not_in_progress, :nothing_to_undo.
class UndoLastAnswer
  def self.call(triage:)
    result = nil
    triage.with_lock do
      result =
        if !triage.status_in_progress?
          Result.fail(:not_in_progress)
        else
          trail = triage.protocol.evaluate(triage.answers).trail
          if trail.empty?
            Result.fail(:nothing_to_undo)
          else
            last_step = trail.last[:step].to_s
            triage.update!(answers: triage.answers.except(last_step), current_step: last_step)
            Result.ok(triage: triage)
          end
        end
    end
    result
  end
end
```

Em `app/commands/conversation_advance.rb`:
- troque `DEFAULT_PROTOCOL_NAME = "triage-respiratoria"` por `DEFAULT_PROTOCOL_NAME = StartTriage::DEFAULT_PROTOCOL_NAME`;
- troque o corpo de `begin_triage_or_nil` inteiro (incluindo o `rescue`) por:

```ruby
  def begin_triage_or_nil
    StartTriage.call(conversation: @conversation).payload[:triage]
  end
```

Em `app/commands/give_consent.rb`: `self.call(conversation:, version:, evidence:, channel: "whatsapp")` repassa `channel:` para `new`, `initialize` guarda `@channel = channel`, e o `create!` usa `channel: @channel`.

Em `app/jobs/notify_citizen_job.rb`, logo depois de `triage = Triage.find(triage_id)`:

```ruby
    # Na web o link aparece na própria tela final (spec 2026-09-22-web-citizen-
    # channel §3.2); não há para onde mandar mensagem.
    return if triage.conversation.channel_web?
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `docker compose exec -T api bundle exec rspec spec/commands/start_triage_spec.rb spec/commands/undo_last_answer_spec.rb spec/commands/give_consent_channel_spec.rb spec/jobs/notify_citizen_job_spec.rb spec/commands/conversation_advance_spec.rb spec/commands/complete_triage_spec.rb spec/jobs/process_inbound_message_job_spec.rb`
Expected: PASS (as specs do WhatsApp inalteradas).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/commands/start_triage.rb app/commands/undo_last_answer.rb app/commands/conversation_advance.rb app/commands/give_consent.rb app/jobs/notify_citizen_job.rb spec/support/triage_protocol_helpers.rb spec/commands/start_triage_spec.rb spec/commands/undo_last_answer_spec.rb spec/commands/give_consent_channel_spec.rb spec/jobs/notify_citizen_job_spec.rb
/opt/homebrew/bin/git commit -m "refactor: extract StartTriage and make consent channel a parameter" -m "Also adds UndoLastAnswer and stops NotifyCitizenJob for web conversations." -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Comandos do cidadão (`Citizens::`)

**Repo:** `apps/api`.

**Files:**
- Create: `app/commands/citizens/register_person.rb`, `app/commands/citizens/start_conversation.rb`, `app/commands/citizens/submit_answer.rb`, `app/commands/citizens/answer_validator.rb`, `app/commands/citizens/step_payload.rb`, `config/locales/citizen.pt-BR.yml`
- Test: `spec/commands/citizens/register_person_spec.rb`, `spec/commands/citizens/start_conversation_spec.rb`, `spec/commands/citizens/submit_answer_spec.rb`, `spec/commands/citizens/step_payload_spec.rb`

**Interfaces:**
- Consumes: `Citizen`, `CitizenIdentity::Cpf` (Tasks 1–2); `StartTriage`, `GiveConsent(channel:)` (Task 4); `CompleteTriage`, `Consents.current_version`.
- Produces:
  - `Citizens::RegisterPerson.call(phone:, cpf:) -> ok(citizen:) | fail(:invalid_cpf | :too_many_people)`
  - `Citizens::StartConversation.call(citizen:, consent_version:, session_id:) -> ok(conversation:, triage:, resumed: Boolean) | fail(:consent_outdated | :no_protocol | :wrong_state | :version_mismatch)`
  - `Citizens::SubmitAnswer.call(conversation:, answer:, idempotency_key:) -> ok(triage:, replayed: Boolean) | fail(:invalid_answer | :not_in_progress | :no_consent)`
  - `Citizens::AnswerValidator.valid?(step, answer) -> Boolean`
  - `Citizens::StepPayload.for(triage) -> { triage_id:, step_id:, prompt:, answer_type:, options: [{id:, title:}], index:, total:, can_undo: }`

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/commands/citizens/register_person_spec.rb
require "rails_helper"

RSpec.describe Citizens::RegisterPerson do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:phone) { "+5541998765432" }

  it "cria o cidadão declarado com o CPF normalizado" do
    result = described_class.call(phone: phone, cpf: "529.982.247-25")
    expect(result.payload[:citizen].cpf).to eq("52998224725")
    expect(result.payload[:citizen]).to be_verification_level_declared
  end

  it "devolve o mesmo cidadão para o mesmo par" do
    first = described_class.call(phone: phone, cpf: "52998224725").payload[:citizen]
    expect(described_class.call(phone: phone, cpf: "529.982.247-25").payload[:citizen]).to eq(first)
  end

  it "recusa CPF inválido" do
    expect(described_class.call(phone: phone, cpf: "111.111.111-11").reason).to eq(:invalid_cpf)
  end

  # Monta um CPF válido a partir de 9 dígitos, com a mesma conta da produção.
  def valid_cpf(base)
    nums = base.chars.map(&:to_i)
    first = CitizenIdentity::Cpf.check_digit(nums)
    second = CitizenIdentity::Cpf.check_digit(nums + [first])
    "#{base}#{first}#{second}"
  end

  it "recusa o 11º CPF no mesmo telefone" do
    cpfs = (1..11).map { |i| valid_cpf(format("%09d", 100_000_000 + i * 7_919)) }
    cpfs.first(10).each { |cpf| expect(described_class.call(phone: phone, cpf: cpf)).to be_ok }
    expect(described_class.call(phone: phone, cpf: cpfs.last).reason).to eq(:too_many_people)
  end
end
```

```ruby
# spec/commands/citizens/start_conversation_spec.rb
require "rails_helper"

RSpec.describe Citizens::StartConversation do
  before do
    Current.city = TEST_CITY_A
    create_default_protocol!
  end
  after { Current.reset; Rails.cache.clear }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }

  def start(version: Consents.current_version)
    described_class.call(citizen: citizen, consent_version: version, session_id: "sess-1")
  end

  it "abre a conversa web, registra o consentimento web e começa a triagem" do
    result = start
    expect(result).to be_ok
    conversation = result.payload[:conversation]
    expect(conversation).to be_channel_web
    expect(conversation.citizen).to eq(citizen)
    expect(conversation.phone).to eq(citizen.phone)
    expect(conversation.consents.last.channel).to eq("web")
    expect(result.payload[:triage].current_step).to eq("tosse")
    expect(result.payload[:resumed]).to be(false)
  end

  it "retoma a conversa ativa em vez de abrir outra" do
    first = start.payload
    CompleteTriage.call(triage: first[:triage], answer: "true")
    again = start
    expect(again.payload[:conversation]).to eq(first[:conversation])
    expect(again.payload[:triage].reload.current_step).to eq("febre")
    expect(again.payload[:resumed]).to be(true)
  end

  it "recusa versão do termo que não é a vigente" do
    expect(start(version: "999").reason).to eq(:consent_outdated)
  end

  it "pede o consentimento de novo quando um termo novo sai no meio da triagem" do
    first = start.payload
    ConsentTerm.create!(version: (Consents.current_version.to_i + 1).to_s, body: "Termo novo", published_at: Time.current)
    again = start
    expect(again).to be_ok
    conversation = again.payload[:conversation]
    expect(conversation).to eq(first[:conversation])
    expect(conversation.reload).to be_consented
    expect(CompleteTriage.call(triage: again.payload[:triage], answer: "true")).to be_ok
  end

  it "falha com :no_protocol sem protocolo ativo, sem perder o consentimento" do
    ProtocolDefinition.update_all(status: "retired")
    result = start
    expect(result.reason).to eq(:no_protocol)
    expect(Conversation.channel_web.last).to be_state_consented
  end
end
```

`status: "retired"` precisa ser um status válido de `protocol_definitions`; confira em `db/city_schema.rb` (check `ck_protocol_definitions_status` ou similar) e use um que tire o protocolo de `status: "active"`.

```ruby
# spec/commands/citizens/submit_answer_spec.rb
require "rails_helper"

RSpec.describe Citizens::SubmitAnswer do
  before do
    Current.city = TEST_CITY_A
    create_default_protocol!
  end
  after { Current.reset; Rails.cache.clear }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:started) { Citizens::StartConversation.call(citizen: citizen, consent_version: Consents.current_version, session_id: "s").payload }
  let(:conversation) { started[:conversation] }
  let(:triage) { started[:triage] }

  def submit(answer, key = SecureRandom.uuid)
    described_class.call(conversation: conversation, answer: answer, idempotency_key: key)
  end

  it "avança para o próximo passo" do
    expect(submit("true")).to be_ok
    expect(triage.reload.current_step).to eq("febre")
  end

  it "conclui a triagem e a conversa no último passo, publicando os eventos" do
    submit("true")
    result = submit("true")
    expect(result.payload[:triage]).to be_status_completed
    expect(conversation.reload).to be_state_completed
    names = DomainEvent.where("payload->>'triage_id' = ?", triage.id).pluck(:name)
    expect(names).to include("triage.completed")
  end

  it "recusa resposta fora do esperado sem gravar nada" do
    expect(submit("banana").reason).to eq(:invalid_answer)
    expect(submit("sim").reason).to eq(:invalid_answer)
    expect(triage.reload.answers).to eq({})
    expect(triage).to be_status_in_progress
  end

  it "a mesma chave repetida não avança duas vezes" do
    submit("true", "k1")
    result = submit("true", "k1")
    expect(result.payload[:replayed]).to be(true)
    expect(triage.reload.answers).to eq("tosse" => "true")
    expect(triage.current_step).to eq("febre")
  end

  it "outra chave depois de concluída é recusada" do
    submit("true")
    submit("false")
    expect(submit("true").reason).to eq(:not_in_progress)
  end
end
```

```ruby
# spec/commands/citizens/step_payload_spec.rb
require "rails_helper"

RSpec.describe Citizens::StepPayload do
  before do
    Current.city = TEST_CITY_A
    create_default_protocol!
  end
  after { Current.reset; Rails.cache.clear }

  let(:conversation) do
    Conversation.create!(phone: "+5541998765432", state: :awaiting_consent).tap do |c|
      GiveConsent.call(conversation: c, version: Consents.current_version, evidence: {})
    end
  end
  let(:triage) { StartTriage.call(conversation: conversation).payload[:triage] }

  it "descreve o passo atual com opções, progresso e voltar" do
    expect(described_class.for(triage)).to eq(
      triage_id: triage.id, step_id: "tosse", prompt: "Você está com tosse?", answer_type: "boolean",
      options: [{ id: "true", title: "Sim" }, { id: "false", title: "Não" }],
      index: 1, total: 2, can_undo: false
    )
  end

  it "no segundo passo, pode voltar" do
    CompleteTriage.call(triage: triage, answer: "true")
    payload = described_class.for(triage.reload)
    expect(payload).to include(step_id: "febre", index: 2, total: 2, can_undo: true)
  end

  it "enum mostra todas as opções, sem truncar" do
    step = Protocols::Step.new(id: "s", prompt: "?", answer_type: :enum,
                               options: ["Uma opção com um título bem maior que vinte e quatro letras"] + (1..11).map(&:to_s))
    expect(described_class.options_for(step).size).to eq(12)
    expect(described_class.options_for(step).first[:title]).to eq("Uma opção com um título bem maior que vinte e quatro letras")
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/commands/citizens`
Expected: FAIL com `uninitialized constant Citizens`.

- [ ] **Step 3: Implementação**

```yaml
# config/locales/citizen.pt-BR.yml
pt-BR:
  citizen:
    "yes": "Sim"
    "no": "Não"
```

```ruby
# app/commands/citizens/answer_validator.rb
# O motor não recusa resposta fora do esperado: sem ramo, o fluxo acaba e
# pontua (Protocols::Protocol#evaluate). No WhatsApp isso é mitigado por botões;
# na web, a resposta é conferida contra o passo ANTES do CompleteTriage
# (spec 2026-09-22-web-citizen-channel §3.2).
module Citizens
  module AnswerValidator
    TEXT_MAX = 500

    module_function

    def valid?(step, answer)
      value = answer.to_s
      case step.answer_type
      when :boolean then %w[true false].include?(value)
      when :enum    then Array(step.options).map(&:to_s).include?(value)
      when :integer then value.match?(/\A\d{1,4}\z/)
      else value.strip.length.between?(1, TEXT_MAX)
      end
    end
  end
end
```

```ruby
# app/commands/citizens/step_payload.rb
# O passo atual da triagem, como a tela da web precisa: todas as opções (sem os
# limites do WhatsApp), progresso e se dá para voltar.
module Citizens
  module StepPayload
    module_function

    def for(triage)
      step = triage.protocol.steps[triage.current_step.to_sym]
      answered = triage.answers.size
      {
        triage_id: triage.id,
        step_id: step.id.to_s,
        prompt: step.prompt,
        answer_type: step.answer_type.to_s,
        options: options_for(step),
        index: answered + 1,
        total: [triage.protocol.steps.size, answered + 1].max,
        can_undo: answered.positive?
      }
    end

    def options_for(step)
      case step.answer_type
      when :boolean
        [{ id: "true", title: I18n.t("citizen.yes") }, { id: "false", title: I18n.t("citizen.no") }]
      when :enum
        Array(step.options).map { |o| { id: o.to_s, title: o.to_s } }
      else
        []
      end
    end
  end
end
```

```ruby
# app/commands/citizens/register_person.rb
# "Para quem é esta triagem?" com CPF novo: cria (ou acha) o par CPF + telefone.
# Reasons: :invalid_cpf, :too_many_people.
module Citizens
  class RegisterPerson
    def self.call(phone:, cpf:)
      digits = CitizenIdentity::Cpf.normalize(cpf)
      return Result.fail(:invalid_cpf) unless digits

      existing = Citizen.find_by(cpf: digits, phone: phone)
      return Result.ok(citizen: existing) if existing
      return Result.fail(:too_many_people) if Citizen.where(phone: phone).count >= Citizen::MAX_PER_PHONE

      Result.ok(citizen: Citizen.create!(cpf: digits, phone: phone))
    rescue ActiveRecord::RecordNotUnique
      Result.ok(citizen: Citizen.find_by!(cpf: digits, phone: phone))
    end
  end
end
```

```ruby
# app/commands/citizens/start_conversation.rb
# Abre ou retoma a conversa web de um cidadão, com o consentimento da versão
# vigente, e garante uma triagem em andamento. O consentimento é dado na tela
# ANTES do CPF (spec §4); aqui ele é registrado na conversa, pelo mesmo
# GiveConsent do WhatsApp, com channel "web".
# Reasons: :consent_outdated, :no_protocol (e as de GiveConsent).
module Citizens
  class StartConversation
    def self.call(citizen:, consent_version:, session_id:)
      new(citizen, consent_version.to_s, session_id).call
    end

    def initialize(citizen, consent_version, session_id)
      @citizen = citizen
      @consent_version = consent_version
      @session_id = session_id
    end

    def call
      return Result.fail(:consent_outdated) unless @consent_version == Consents.current_version

      conversation = find_or_create_conversation
      consent = ensure_consent(conversation)
      return consent if consent.failure?

      triage = conversation.triages.status_in_progress.order(created_at: :desc).first
      return Result.ok(conversation: conversation, triage: triage, resumed: true) if triage

      started = StartTriage.call(conversation: conversation)
      return started if started.failure?

      Result.ok(conversation: conversation, triage: started.payload[:triage], resumed: false)
    end

    private

    def active_scope
      Conversation.channel_web.where(citizen: @citizen, state: Conversation::ACTIVE_STATES)
    end

    def find_or_create_conversation
      active_scope.first ||
        Conversation.create!(channel: "web", citizen: @citizen, phone: @citizen.phone, state: :awaiting_consent)
    rescue ActiveRecord::RecordNotUnique
      active_scope.first!
    end

    # Uma conversa já consentida com termo antigo (termo novo publicado no meio
    # da triagem) volta a awaiting_consent e consente de novo; senão o
    # CompleteTriage recusaria com :no_consent.
    def ensure_consent(conversation)
      return Result.ok if conversation.consented?

      conversation.update!(state: :awaiting_consent) unless conversation.state_awaiting_consent?
      GiveConsent.call(
        conversation: conversation,
        version: @consent_version,
        channel: "web",
        evidence: { channel: "web", citizen_session_id: @session_id, version: @consent_version }
      )
    end
  end
end
```

```ruby
# app/commands/citizens/submit_answer.rb
# Uma resposta da web. Tudo sob o lock da conversa: duas abas ou um toque duplo
# não avançam a triagem duas vezes. A mesma idempotency_key devolve o estado
# atual sem gravar (spec §4.3).
# Reasons: :invalid_answer, :not_in_progress (e as de CompleteTriage).
module Citizens
  class SubmitAnswer
    def self.call(conversation:, answer:, idempotency_key:)
      new(conversation, answer.to_s.strip, idempotency_key.presence).call
    end

    def initialize(conversation, answer, idempotency_key)
      @conversation = conversation
      @answer = answer
      @idempotency_key = idempotency_key
    end

    def call
      result = nil
      @conversation.with_lock { result = locked_call }
      result
    end

    private

    def locked_call
      triage = @conversation.triages.order(created_at: :desc).first
      return Result.fail(:not_in_progress) unless triage
      return Result.ok(triage: triage, replayed: true) if replay?
      return Result.fail(:not_in_progress) unless triage.status_in_progress?

      step = triage.protocol.steps[triage.current_step.to_sym]
      return Result.fail(:invalid_answer) unless step && AnswerValidator.valid?(step, @answer)

      completed = CompleteTriage.call(triage: triage, answer: @answer)
      return completed if completed.failure?

      @conversation.update!(last_answer_key: @idempotency_key)
      @conversation.update!(state: :completed) if completed.payload[:outcome].terminal?
      Result.ok(triage: triage.reload, replayed: false)
    end

    def replay?
      @idempotency_key && @idempotency_key == @conversation.last_answer_key
    end
  end
end
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `docker compose exec -T api bundle exec rspec spec/commands/citizens`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/commands/citizens config/locales/citizen.pt-BR.yml spec/commands/citizens
/opt/homebrew/bin/git commit -m "feat: add citizen commands to start a web triage and submit answers" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Entrada do cidadão (`/citizen/otp` e `/citizen/session`)

**Repo:** `apps/api`.

**Files:**
- Create: `app/controllers/concerns/citizen_authentication.rb`, `app/controllers/citizen_api/base_controller.rb`, `app/controllers/citizen_api/otps_controller.rb`, `app/controllers/citizen_api/sessions_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/filter_parameter_logging.rb`
- Create: `spec/support/citizen_request_helpers.rb`
- Test: `spec/requests/citizen_api/otp_and_session_spec.rb`

**Interfaces:**
- Consumes: `OtpChallenge`, `OtpSender` (Task 3); `CitizenSession`, `Current.citizen_session` (Task 2); `CitizenIdentity::Phone` (Task 1).
- Produces:
  - `CitizenApi::BaseController` (inclui `CitizenAuthentication`; `render_error(code, status)`; `RateLimitStore`); os controllers da Task 7 herdam dele.
  - `CitizenAuthentication#current_citizen_session -> CitizenSession`, `allow_anonymous_citizen(**opts)`.
  - Cookie assinado `citizen_session` (httpOnly, SameSite=Lax, 30 dias).
  - Rotas: `POST /citizen/otp {phone}` → 202 `{status: "sent", resend_after: 60}`; `POST /citizen/session {phone, code}` → 201 `{phone_masked}`; `GET /citizen/session` → 200 | 401; `DELETE /citizen/session` → 204.
  - Helper de spec `sign_in_citizen(phone) -> CitizenSession`.

- [ ] **Step 1: Write the failing test**

```ruby
# spec/support/citizen_request_helpers.rb
# Planta o cookie assinado `citizen_session` de uma sessão real, como
# CitizenAuthentication lê. Não passa pelo OTP.
module CitizenRequestHelpers
  def sign_in_citizen(phone = "+5541998765432")
    session, token = CitizenSession.start!(phone: phone)
    jar = ActionDispatch::TestRequest.create.cookie_jar
    jar.signed[:citizen_session] = token
    cookies[:citizen_session] = jar[:citizen_session]
    session
  end

  def json_post(path, params = {})
    post path, params: params.to_json, headers: { "CONTENT_TYPE" => "application/json" }
  end
end

RSpec.configure { |c| c.include CitizenRequestHelpers, type: :request }
```

```ruby
# spec/requests/citizen_api/otp_and_session_spec.rb
require "rails_helper"

RSpec.describe "Citizen OTP and session", type: :request do
  before { OtpSender::Test.reset!; Rails.cache.clear }
  after { travel_back }

  let(:phone) { "(41) 99876-5432" }

  def last_code = OtpSender::Test.deliveries.last[:code]

  it "manda o código e abre a sessão com cookie httpOnly" do
    json_post "/citizen/otp", phone: phone
    expect(response).to have_http_status(:accepted)
    expect(JSON.parse(response.body)).to eq("status" => "sent", "resend_after" => 60)

    json_post "/citizen/session", phone: phone, code: last_code
    expect(response).to have_http_status(:created)
    expect(JSON.parse(response.body)).to eq("phone_masked" => "(**) *****-5432")
    expect(response.headers["Set-Cookie"]).to match(/citizen_session=.*httponly/i)

    get "/citizen/session"
    expect(response).to have_http_status(:ok)
  end

  it "responde igual para telefone novo e telefone já cadastrado" do
    Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    json_post "/citizen/otp", phone: phone
    known = [response.status, response.body]
    json_post "/citizen/otp", phone: "(41) 91111-2222"
    expect([response.status, response.body]).to eq(known)
  end

  it "recusa telefone que não é celular brasileiro" do
    json_post "/citizen/otp", phone: "(41) 3333-4444"
    expect(response).to have_http_status(:unprocessable_entity)
    expect(JSON.parse(response.body)["error"]).to eq("invalid_phone")
  end

  it "reenvio antes de 60 s responde 429 too_soon" do
    json_post "/citizen/otp", phone: phone
    json_post "/citizen/otp", phone: phone
    expect(response).to have_http_status(:too_many_requests)
    expect(JSON.parse(response.body)["error"]).to eq("too_soon")
  end

  it "sem provedor de SMS, 503 e o desafio não conta no limite" do
    allow(Rails.configuration.x).to receive(:otp_sender).and_return(nil)
    json_post "/citizen/otp", phone: phone
    expect(response).to have_http_status(:service_unavailable)
    expect(OtpChallenge.count).to eq(0)
  end

  it "código errado e código vencido" do
    json_post "/citizen/otp", phone: phone
    json_post "/citizen/session", phone: phone, code: (last_code == "000000" ? "111111" : "000000")
    expect(JSON.parse(response.body)["error"]).to eq("invalid_code")
    travel 11.minutes
    json_post "/citizen/session", phone: phone, code: last_code
    expect(JSON.parse(response.body)["error"]).to eq("code_expired")
  end

  it "sair revoga a sessão" do
    sign_in_citizen
    delete "/citizen/session", headers: { "CONTENT_TYPE" => "application/json" }
    expect(response).to have_http_status(:no_content)
    get "/citizen/session"
    expect(response).to have_http_status(:unauthorized)
  end

  it "sem cookie, 401" do
    get "/citizen/session"
    expect(response).to have_http_status(:unauthorized)
  end

  it "escrita autenticada sem JSON é recusada" do
    sign_in_citizen
    post "/citizen/otp", params: { phone: phone }
    expect(response).to have_http_status(:unsupported_media_type)
  end

  it "cidade com banco atrasado responde 503" do
    allow(CitySchema).to receive(:behind?).and_return(true)
    json_post "/citizen/otp", phone: phone
    expect(response).to have_http_status(:service_unavailable)
  end

  it "o cookie de servidor da cidade não abre sessão de cidadão" do
    user = create(:user)
    sign_in_as(user)
    get "/citizen/session"
    expect(response).to have_http_status(:unauthorized)
  end
end
```

Confira que existe a factory `:user` (`spec/factories`); se o nome for outro, use o que `spec/requests/city_session_isolation_spec.rb` usa.

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec -T api bundle exec rspec spec/requests/citizen_api/otp_and_session_spec.rb`
Expected: FAIL com `404` (rota inexistente).

- [ ] **Step 3: Implementação**

```ruby
# app/controllers/concerns/citizen_authentication.rb
# Sessão do cidadão no canal web (spec 2026-09-22-web-citizen-channel §2.6).
# Cookie e tabela próprios (`citizen_session`, citizen_sessions): nada aqui lê
# o `session_id` dos servidores da cidade, e vice-versa. Roda depois de
# CityResolution, então a sessão é procurada no banco da cidade do host.
module CitizenAuthentication
  extend ActiveSupport::Concern

  COOKIE = :citizen_session
  WRITE_METHODS = %w[POST PUT PATCH DELETE].freeze

  included do
    before_action :require_json_for_writes
    before_action :require_citizen_session
  end

  class_methods do
    def allow_anonymous_citizen(**options)
      skip_before_action :require_citizen_session, **options
    end
  end

  private

  def current_citizen_session
    Current.citizen_session ||= CitizenSession.resume(cookies.signed[COOKIE])
  end

  def require_citizen_session
    return render(json: { error: "unauthenticated" }, status: :unauthorized) unless current_citizen_session

    # Prazo deslizante também no navegador: o cookie acompanha a sessão.
    write_citizen_cookie(cookies.signed[COOKIE])
  end

  # Toda escrita é JSON: um formulário de outro site não consegue mandar
  # application/json sem preflight de CORS.
  def require_json_for_writes
    return unless WRITE_METHODS.include?(request.request_method)
    return if request.media_type == "application/json"

    render json: { error: "json_required" }, status: :unsupported_media_type
  end

  def write_citizen_cookie(token)
    cookies.signed[COOKIE] = {
      value: token,
      httponly: true,
      same_site: :lax,
      secure: Rota.deployed?,
      expires: CitizenSession::TTL.from_now
    }
  end

  def clear_citizen_cookie
    cookies.delete(COOKIE)
  end
end
```

```ruby
# app/controllers/citizen_api/base_controller.rb
# Base das rotas /citizen/* (canal web do cidadão). A cidade vem do host
# (CityResolution, em ApplicationController); a sessão, do cookie
# `citizen_session` (CitizenAuthentication).
module CitizenApi
  class BaseController < ApplicationController
    include CitizenAuthentication

    # Mesmo delegador de MfaController::RateLimitStore: resolve Rails.cache a
    # cada requisição, o que torna o teto exercitável em spec.
    module RateLimitStore
      def self.increment(...) = Rails.cache.increment(...)
    end

    private

    def render_error(code, status)
      render json: { error: code.to_s }, status: status
    end
  end
end
```

```ruby
# app/controllers/citizen_api/otps_controller.rb
# POST /citizen/otp { phone } — manda o código por SMS (spec §4, §4.2).
# A resposta é a mesma exista cadastro ou não: ela não revela quem usa o serviço.
module CitizenApi
  class OtpsController < BaseController
    allow_anonymous_citizen only: :create

    rate_limit to: 10, within: 1.hour, only: :create, name: "citizen_otp_ip", store: RateLimitStore,
               with: -> { render_error("too_many_requests", :too_many_requests) }

    def create
      phone = CitizenIdentity::Phone.normalize(params[:phone])
      return render_error("invalid_phone", :unprocessable_entity) unless phone

      challenge, code = OtpChallenge.issue!(phone: phone)
      begin
        OtpSender.deliver(phone: phone, code: code)
      rescue OtpSender::Unavailable
        challenge.destroy!
        return render_error("otp_unavailable", :service_unavailable)
      end

      render json: { status: "sent", resend_after: OtpChallenge::RESEND_AFTER.to_i }, status: :accepted
    rescue OtpChallenge::TooSoon
      render_error("too_soon", :too_many_requests)
    rescue OtpChallenge::DailyLimit
      render_error("daily_limit", :too_many_requests)
    end
  end
end
```

```ruby
# app/controllers/citizen_api/sessions_controller.rb
#   POST   /citizen/session { phone, code } → 201 + cookie
#   GET    /citizen/session                 → 200 | 401
#   DELETE /citizen/session                 → 204
module CitizenApi
  class SessionsController < BaseController
    allow_anonymous_citizen only: :create

    rate_limit to: 20, within: 1.hour, only: :create, name: "citizen_session_ip", store: RateLimitStore,
               with: -> { render_error("too_many_requests", :too_many_requests) }

    def create
      phone = CitizenIdentity::Phone.normalize(params[:phone])
      return render_error("invalid_code", :unprocessable_entity) unless phone

      case OtpChallenge.verify(phone: phone, code: params[:code])
      when :ok
        session, token = CitizenSession.start!(phone: phone)
        Current.citizen_session = session
        write_citizen_cookie(token)
        render json: session_json(session), status: :created
      when :expired, :missing then render_error("code_expired", :unprocessable_entity)
      when :exhausted         then render_error("code_exhausted", :unprocessable_entity)
      else                         render_error("invalid_code", :unprocessable_entity)
      end
    end

    def show
      render json: session_json(current_citizen_session)
    end

    def destroy
      current_citizen_session.revoke!
      clear_citizen_cookie
      head :no_content
    end

    private

    def session_json(session)
      { phone_masked: CitizenIdentity::Phone.mask(session.phone) }
    end
  end
end
```

Em `config/routes.rb`, logo depois do bloco do relatório público (`get "/r/:token"`):

```ruby
  # Canal web do cidadão (spec 2026-09-22-web-citizen-channel). Servido no host
  # da cidade, como o relatório; sessão própria por cookie `citizen_session`.
  scope "/citizen", module: "citizen_api", as: "citizen" do
    post   "otp",     to: "otps#create"
    post   "session", to: "sessions#create"
    get    "session", to: "sessions#show"
    delete "session", to: "sessions#destroy"
  end
```

Em `config/initializers/filter_parameter_logging.rb`, acrescente `:cpf, :answer` à lista (o `code` já é filtrado).

- [ ] **Step 4: Run test to verify it passes**

Run: `docker compose exec -T api bundle exec rspec spec/requests/citizen_api/otp_and_session_spec.rb spec/requests/cookie_write_requires_json_spec.rb spec/architecture`
Expected: PASS. Se algum spec de `spec/architecture` (rotas por host, CORS) falhar por causa de `/citizen`, leia a regra que ele protege e registre a rota nova do mesmo jeito que `/r/:token` está registrado.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/concerns/citizen_authentication.rb app/controllers/citizen_api/base_controller.rb app/controllers/citizen_api/otps_controller.rb app/controllers/citizen_api/sessions_controller.rb config/routes.rb config/initializers/filter_parameter_logging.rb spec/support/citizen_request_helpers.rb spec/requests/citizen_api/otp_and_session_spec.rb
/opt/homebrew/bin/git commit -m "feat: open citizen sessions with an SMS code" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Triagem do cidadão (`/citizen/*` restante)

**Repo:** `apps/api`.

**Files:**
- Create: `app/controllers/citizen_api/consent_terms_controller.rb`, `app/controllers/citizen_api/people_controller.rb`, `app/controllers/citizen_api/conversations_controller.rb`, `app/controllers/citizen_api/triages_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/citizen_api/triage_flow_spec.rb`, `spec/requests/citizen_api/isolation_spec.rb`

**Interfaces:**
- Consumes: `CitizenApi::BaseController`, `sign_in_citizen`, `json_post` (Task 6); `Citizens::*` (Task 5); `UndoLastAnswer` (Task 4); `RevokeConsent`, `ReportSnapshot#url`.
- Produces (JSON):
  - `GET /citizen/consent_term` → `{version, body}` | 503 `no_consent_term`
  - `GET /citizen/people` → `{people: [{id, cpf_masked, verification_level}]}`
  - `POST /citizen/conversations {citizen_id | cpf, consent_version}` → 201 (nova) / 200 (retomada) `{conversation_id, citizen_id, resumed, step}`; erros 422 `invalid_cpf`/`too_many_people`, 404, 409 `consent_outdated`, 503 `no_protocol`
  - `POST /citizen/conversations/:id/answers {answer, idempotency_key}` → 200 `{status: "in_progress", step}` ou `{status: "completed", triage_id}`; 422 `invalid_answer`; 409 `not_in_progress`
  - `POST /citizen/conversations/:id/undo` → 200 `{status: "in_progress", step}`; 409 `not_in_progress`/`nothing_to_undo`
  - `GET /citizen/triages?citizen_id=` → `{citizen: {id, cpf_masked, verification_level}, triages: [summary]}`
  - `GET /citizen/triages/:id` → `summary`
  - `POST /citizen/triages/:id/revoke_consent` → 200 `summary`; 409 `no_active_consent`
  - `summary = {id, status, tier, priority, created_at, completed_at, report_url, consent_active}`

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/requests/citizen_api/triage_flow_spec.rb
require "rails_helper"

RSpec.describe "Citizen triage flow", type: :request do
  before do
    Current.city = TEST_CITY_A
    create_default_protocol!
    ConsentTerm.create!(version: "1", body: "Termo de teste", published_at: Time.current)
    sign_in_citizen
  end
  after { Current.reset; Rails.cache.clear }

  def body = JSON.parse(response.body)

  def start_new(cpf = "529.982.247-25")
    json_post "/citizen/conversations", cpf: cpf, consent_version: "1"
    body
  end

  it "mostra o termo vigente" do
    get "/citizen/consent_term"
    expect(body).to eq("version" => "1", "body" => "Termo de teste")
  end

  it "faz a triagem inteira e lista o resultado" do
    started = start_new
    expect(response).to have_http_status(:created)
    expect(started["step"]).to include("step_id" => "tosse", "index" => 1, "can_undo" => false)

    json_post "/citizen/conversations/#{started['conversation_id']}/answers", answer: "true", idempotency_key: "a1"
    expect(body).to include("status" => "in_progress")
    expect(body["step"]).to include("step_id" => "febre", "can_undo" => true)

    json_post "/citizen/conversations/#{started['conversation_id']}/answers", answer: "true", idempotency_key: "a2"
    expect(body).to include("status" => "completed")
    triage_id = body["triage_id"]

    get "/citizen/people"
    person = body["people"].sole
    expect(person).to include("cpf_masked" => "***.982.247-**", "verification_level" => "declared")

    get "/citizen/triages", params: { citizen_id: person["id"] }
    expect(body["triages"].sole).to include("id" => triage_id, "status" => "completed", "tier" => "alta")

    get "/citizen/triages/#{triage_id}"
    expect(body).to include("id" => triage_id, "consent_active" => true)
  end

  it "voltar desfaz a última resposta" do
    started = start_new
    json_post "/citizen/conversations/#{started['conversation_id']}/answers", answer: "true", idempotency_key: "a1"
    json_post "/citizen/conversations/#{started['conversation_id']}/undo"
    expect(body["step"]).to include("step_id" => "tosse", "can_undo" => false)
  end

  it "retoma a conversa em andamento com 200" do
    started = start_new
    json_post "/citizen/conversations/#{started['conversation_id']}/answers", answer: "true", idempotency_key: "a1"
    json_post "/citizen/conversations", citizen_id: started["citizen_id"], consent_version: "1"
    expect(response).to have_http_status(:ok)
    expect(body).to include("resumed" => true)
    expect(body["step"]).to include("step_id" => "febre")
  end

  it "resposta fora do esperado: 422 e nada muda" do
    started = start_new
    json_post "/citizen/conversations/#{started['conversation_id']}/answers", answer: "banana", idempotency_key: "a1"
    expect(response).to have_http_status(:unprocessable_entity)
    expect(body["error"]).to eq("invalid_answer")
    expect(Triage.find(started["step"]["triage_id"]).answers).to eq({})
  end

  it "CPF inválido: 422" do
    start_new("111.111.111-11")
    expect(response).to have_http_status(:unprocessable_entity)
    expect(body["error"]).to eq("invalid_cpf")
  end

  it "termo desatualizado: 409" do
    json_post "/citizen/conversations", cpf: "529.982.247-25", consent_version: "0"
    expect(response).to have_http_status(:conflict)
    expect(body["error"]).to eq("consent_outdated")
  end

  it "revogar o consentimento anonimiza pelo fluxo de sempre" do
    started = start_new
    json_post "/citizen/conversations/#{started['conversation_id']}/answers", answer: "false", idempotency_key: "a1"
    triage_id = body["triage_id"]
    json_post "/citizen/triages/#{triage_id}/revoke_consent"
    expect(response).to have_http_status(:ok)
    expect(body["consent_active"]).to be(false)
    expect(DomainEvent.where(name: "consent.revoked")).to exist
  end
end
```

```ruby
# spec/requests/citizen_api/isolation_spec.rb
require "rails_helper"

# Uma sessão só enxerga os cidadãos do próprio telefone. Id de outro telefone é
# 404, nunca 403: a resposta não confirma que o id existe.
RSpec.describe "Citizen isolation", type: :request do
  before do
    Current.city = TEST_CITY_A
    create_default_protocol!
    ConsentTerm.create!(version: "1", body: "Termo", published_at: Time.current)
  end
  after { Current.reset; Rails.cache.clear }

  let!(:other) do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541911112222")
    started = Citizens::StartConversation.call(citizen: citizen, consent_version: "1", session_id: "x").payload
    Citizens::SubmitAnswer.call(conversation: started[:conversation], answer: "false", idempotency_key: "k")
    { citizen: citizen, conversation: started[:conversation], triage: started[:triage] }
  end

  before { sign_in_citizen("+5541998765432") }

  it "não lista pessoas de outro telefone, nem com o mesmo CPF" do
    get "/citizen/people"
    expect(JSON.parse(response.body)["people"]).to eq([])
  end

  it "404 para cidadão, conversa e triagem de outro telefone" do
    get "/citizen/triages", params: { citizen_id: other[:citizen].id }
    expect(response).to have_http_status(:not_found)

    json_post "/citizen/conversations", citizen_id: other[:citizen].id, consent_version: "1"
    expect(response).to have_http_status(:not_found)

    json_post "/citizen/conversations/#{other[:conversation].id}/answers", answer: "true", idempotency_key: "z"
    expect(response).to have_http_status(:not_found)

    get "/citizen/triages/#{other[:triage].id}"
    expect(response).to have_http_status(:not_found)

    json_post "/citizen/triages/#{other[:triage].id}/revoke_consent"
    expect(response).to have_http_status(:not_found)
  end

  it "o mesmo CPF digitado neste telefone vira outro cidadão, sem ver as triagens do primeiro" do
    json_post "/citizen/conversations", cpf: "529.982.247-25", consent_version: "1"
    mine = JSON.parse(response.body)["citizen_id"]
    expect(mine).not_to eq(other[:citizen].id)
    get "/citizen/triages", params: { citizen_id: mine }
    expect(JSON.parse(response.body)["triages"].map { |t| t["id"] }).not_to include(other[:triage].id)
  end
end
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `docker compose exec -T api bundle exec rspec spec/requests/citizen_api/triage_flow_spec.rb spec/requests/citizen_api/isolation_spec.rb`
Expected: FAIL com 404 nas rotas.

- [ ] **Step 3: Implementação**

```ruby
# app/controllers/citizen_api/consent_terms_controller.rb
# GET /citizen/consent_term — o termo vigente da cidade, mostrado antes do CPF.
module CitizenApi
  class ConsentTermsController < BaseController
    def show
      term = ConsentTerm.order(Arel.sql("version::bigint DESC")).first
      return render_error("no_consent_term", :service_unavailable) unless term

      render json: { version: term.version, body: term.body }
    end
  end
end
```

```ruby
# app/controllers/citizen_api/people_controller.rb
# GET /citizen/people — "para quem é esta triagem?": os CPFs ligados ao telefone
# da sessão, mascarados.
module CitizenApi
  class PeopleController < BaseController
    def index
      people = current_citizen_session.citizens.order(:created_at)
      render json: { people: people.map { |c| person_json(c) } }
    end

    private

    def person_json(citizen)
      { id: citizen.id, cpf_masked: citizen.cpf_masked, verification_level: citizen.verification_level }
    end
  end
end
```

```ruby
# app/controllers/citizen_api/conversations_controller.rb
#   POST /citizen/conversations              { citizen_id | cpf, consent_version }
#   POST /citizen/conversations/:id/answers  { answer, idempotency_key }
#   POST /citizen/conversations/:id/undo
module CitizenApi
  class ConversationsController < BaseController
    START_ERRORS = {
      consent_outdated: :conflict, wrong_state: :conflict, version_mismatch: :conflict,
      no_protocol: :service_unavailable
    }.freeze

    def create
      citizen = resolve_citizen
      return if performed?

      result = Citizens::StartConversation.call(
        citizen: citizen, consent_version: params[:consent_version], session_id: current_citizen_session.id
      )
      return render_error(result.reason, START_ERRORS.fetch(result.reason, :unprocessable_entity)) if result.failure?

      payload = result.payload
      render json: {
        conversation_id: payload[:conversation].id,
        citizen_id: citizen.id,
        resumed: payload[:resumed],
        step: Citizens::StepPayload.for(payload[:triage])
      }, status: payload[:resumed] ? :ok : :created
    end

    def answer
      conversation = find_conversation
      return if performed?

      result = Citizens::SubmitAnswer.call(
        conversation: conversation, answer: params[:answer], idempotency_key: params[:idempotency_key]
      )
      if result.failure?
        status = result.reason == :invalid_answer ? :unprocessable_entity : :conflict
        return render_error(result.reason, status)
      end

      render json: triage_state(result.payload[:triage])
    end

    def undo
      conversation = find_conversation
      return if performed?

      triage = conversation.triages.order(created_at: :desc).first
      return render_error("not_in_progress", :conflict) unless triage

      result = UndoLastAnswer.call(triage: triage)
      return render_error(result.reason, :conflict) if result.failure?

      render json: triage_state(triage.reload)
    end

    private

    def resolve_citizen
      if params[:citizen_id].present?
        citizen = current_citizen_session.citizens.find_by(id: params[:citizen_id])
        render_error("not_found", :not_found) unless citizen
        citizen
      else
        result = Citizens::RegisterPerson.call(phone: current_citizen_session.phone, cpf: params[:cpf])
        render_error(result.reason, :unprocessable_entity) if result.failure?
        result.payload[:citizen]
      end
    end

    def find_conversation
      conversation = Conversation.channel_web
                                 .where(citizen_id: current_citizen_session.citizens.select(:id))
                                 .find_by(id: params[:id])
      render_error("not_found", :not_found) unless conversation
      conversation
    end

    def triage_state(triage)
      if triage.status_in_progress?
        { status: "in_progress", step: Citizens::StepPayload.for(triage) }
      else
        { status: triage.status, triage_id: triage.id }
      end
    end
  end
end
```

```ruby
# app/controllers/citizen_api/triages_controller.rb
#   GET  /citizen/triages?citizen_id=
#   GET  /citizen/triages/:id
#   POST /citizen/triages/:id/revoke_consent
# O nível declarado vê só as triagens do próprio par CPF + telefone (spec §2.3).
module CitizenApi
  class TriagesController < BaseController
    def index
      citizen = current_citizen_session.citizens.find_by(id: params[:citizen_id])
      return render_error("not_found", :not_found) unless citizen

      triages = scoped_triages.where(conversations: { citizen_id: citizen.id }).order(created_at: :desc)
      render json: {
        citizen: { id: citizen.id, cpf_masked: citizen.cpf_masked, verification_level: citizen.verification_level },
        triages: triages.map { |t| summary(t) }
      }
    end

    def show
      triage = scoped_triages.find_by(id: params[:id])
      return render_error("not_found", :not_found) unless triage

      render json: summary(triage)
    end

    def revoke_consent
      triage = scoped_triages.find_by(id: params[:id])
      return render_error("not_found", :not_found) unless triage

      result = RevokeConsent.call(conversation: triage.conversation, reason: "citizen_web")
      return render_error(result.reason, :conflict) if result.failure?

      render json: summary(triage.reload)
    end

    private

    def scoped_triages
      Triage.joins(:conversation).where(
        conversations: { channel: "web", citizen_id: current_citizen_session.citizens.select(:id) }
      )
    end

    def summary(triage)
      {
        id: triage.id,
        status: triage.status,
        tier: triage.tier,
        priority: triage.priority,
        created_at: triage.created_at.iso8601,
        completed_at: triage.completed_at&.iso8601,
        report_url: triage.report_snapshot&.url,
        consent_active: triage.conversation.active_consent.present?
      }
    end
  end
end
```

Em `config/routes.rb`, dentro do `scope "/citizen"` da Task 6:

```ruby
    get  "consent_term",               to: "consent_terms#show"
    get  "people",                     to: "people#index"
    post "conversations",              to: "conversations#create"
    post "conversations/:id/answers",  to: "conversations#answer"
    post "conversations/:id/undo",     to: "conversations#undo"
    get  "triages",                    to: "triages#index"
    get  "triages/:id",                to: "triages#show"
    post "triages/:id/revoke_consent", to: "triages#revoke_consent"
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `docker compose exec -T api bundle exec rspec spec/requests/citizen_api`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/citizen_api/consent_terms_controller.rb app/controllers/citizen_api/people_controller.rb app/controllers/citizen_api/conversations_controller.rb app/controllers/citizen_api/triages_controller.rb config/routes.rb spec/requests/citizen_api/triage_flow_spec.rb spec/requests/citizen_api/isolation_spec.rb
/opt/homebrew/bin/git commit -m "feat: let citizens run and review triages over the web" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Contrato de dados web × WhatsApp e suíte completa

**Repo:** `apps/api`.

**Files:**
- Test: `spec/integration/web_channel_data_contract_spec.rb`

**Interfaces:**
- Consumes: `ConversationAdvance` + `InboundMessage` (caminho WhatsApp); `Citizens::StartConversation`, `Citizens::SubmitAnswer` (Task 5).

- [ ] **Step 1: Write the test**

```ruby
# spec/integration/web_channel_data_contract_spec.rb
require "rails_helper"

# O teste que prova o objetivo do spec 2026-09-22-web-citizen-channel: a mesma
# triagem feita pela web e pelo WhatsApp produz os MESMOS registros e eventos.
# Só consents.channel/evidence e conversations.channel/citizen_id podem diferir.
RSpec.describe "Web channel data contract" do
  before do
    Current.city = TEST_CITY_A
    create_default_protocol!
  end
  after { Current.reset; Rails.cache.clear }

  def whatsapp_triage(answers)
    phone = "+5541977776666"
    conversation = Conversation.for(phone)
    (["oi", "sim"] + answers).each do |text|
      inbound = InboundMessage.create!(
        message_id: "wamid.#{SecureRandom.hex(6)}", from: phone, kind: "text",
        raw: { "type" => "text", "text" => { "body" => text } }.to_json
      )
      ConversationAdvance.call(conversation: conversation.reload, inbound: inbound)
    end
    conversation.reload.triages.sole
  end

  def web_triage(answers)
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    started = Citizens::StartConversation.call(citizen: citizen, consent_version: Consents.current_version, session_id: "s").payload
    answers.each_with_index do |answer, i|
      Citizens::SubmitAnswer.call(conversation: started[:conversation], answer: answer, idempotency_key: "k#{i}")
    end
    started[:triage].reload
  end

  def triage_contract(triage)
    triage.attributes.slice("status", "tier", "priority", "answers", "outcome", "protocol_name", "protocol_definition_id")
  end

  def consent_contract(triage)
    triage.conversation.consents.sole.attributes.slice("version", "policy_text_sha", "revoked_at")
  end

  def events_contract(triage)
    DomainEvent.where("payload->>'triage_id' = ?", triage.id).order(:name)
               .map { |e| [e.name, e.payload.except("triage_id")] }
  end

  [%w[true true], %w[true false], %w[false]].each do |answers|
    it "respostas #{answers.inspect}: mesma triagem, mesmo consentimento, mesmos eventos" do
      whatsapp = whatsapp_triage(answers)
      web = web_triage(answers)

      expect(triage_contract(web)).to eq(triage_contract(whatsapp))
      expect(consent_contract(web)).to eq(consent_contract(whatsapp))
      expect(events_contract(web)).to eq(events_contract(whatsapp))
      expect(DomainEvent.where(name: "consent.given").count).to eq(2)

      expect(whatsapp.conversation).to be_state_completed
      expect(web.conversation).to be_state_completed
      expect(web.conversation.consents.sole.channel).to eq("web")
      expect(whatsapp.conversation.consents.sole.channel).to eq("whatsapp")
    end
  end
end
```

No WhatsApp, `complete_and_finish` põe a conversa em `completed`, como a web. Se o `outcome` tiver algo que dependa de horário (ex.: `completed_at` dentro dele), compare-o sem essa chave e registre o motivo num comentário.

- [ ] **Step 2: Run it**

Run: `docker compose exec -T api bundle exec rspec spec/integration/web_channel_data_contract_spec.rb`
Expected: PASS. Uma falha aqui é **bug no caminho da web**, não no teste: investigue antes de mexer na comparação.

- [ ] **Step 3: Suíte completa**

Na raiz do monorepo:

```bash
docker compose stop worker
docker compose exec -T api bundle exec rspec
docker compose start worker
```

Expected: 0 falhas, abaixo de ~3 min. Religue o worker mesmo em falha.

- [ ] **Step 4: Commit**

```bash
/opt/homebrew/bin/git add spec/integration/web_channel_data_contract_spec.rb
/opt/homebrew/bin/git commit -m "test: prove web and WhatsApp triages produce the same data" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: wpda — máscaras, cliente da API e ambiente de teste

**Repo:** `apps/wpda`. Antes de tudo: `/opt/homebrew/bin/git checkout -b feat/web-citizen-channel`. Comandos rodam em `apps/wpda` no host (`npm`), não no container.

**Files:**
- Modify: `package.json`, `vitest.config.ts`, `vite.config.ts`
- Create: `src/test/setup.ts`, `src/lib/masks.ts`, `src/lib/masks.test.ts`, `src/lib/citizenApi.ts`, `src/lib/citizenApi.test.ts`

**Interfaces:**
- Produces:
  - `maskPhone(s) -> "(41) 99876-5432"`, `isMobilePhone(s) -> boolean`, `maskCpf(s) -> "529.982.247-25"`, `isValidCpf(s) -> boolean`, `onlyDigits(s)`.
  - `ApiError { status: number; code: string }`.
  - Tipos `Step`, `AnswerState`, `Person`, `TriageSummary`, `StartResult`.
  - `citizenApi`: `requestCode(phone)`, `verifyCode(phone, code)`, `currentSession()`, `signOut()`, `consentTerm()`, `people()`, `start({citizenId?, cpf?, consentVersion})`, `answer(conversationId, answer, idempotencyKey)`, `undo(conversationId)`, `triages(citizenId)`, `triage(id)`, `revokeConsent(id)`.

- [ ] **Step 1: Dependências e ambiente**

```bash
npm install --save-dev @testing-library/react@^16 @testing-library/user-event@^14 @testing-library/jest-dom@^6
```

`vitest.config.ts`, dentro de `test`: `setupFiles: ["./src/test/setup.ts"]`.

```ts
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach } from "vitest";

afterEach(() => cleanup());
```

`vite.config.ts`: no `proxy`, acrescente `"/citizen": proxy(TARGET)` e, no comentário do topo, a linha `//   /citizen   → canal web do cidadão (CitizenApi).`

- [ ] **Step 2: Write the failing tests**

```ts
// src/lib/masks.test.ts
import { describe, it, expect } from "vitest";
import { maskPhone, isMobilePhone, maskCpf, isValidCpf } from "./masks";

describe("maskPhone", () => {
  it("formata enquanto digita", () => {
    expect(maskPhone("4")).toBe("(4");
    expect(maskPhone("41998")).toBe("(41) 998");
    expect(maskPhone("41998765432")).toBe("(41) 99876-5432");
    expect(maskPhone("4199876543299")).toBe("(41) 99876-5432");
  });
});

describe("isMobilePhone", () => {
  it("aceita celular e recusa fixo", () => {
    expect(isMobilePhone("(41) 99876-5432")).toBe(true);
    expect(isMobilePhone("(41) 3333-4444")).toBe(false);
  });
});

describe("maskCpf", () => {
  it("formata enquanto digita", () => {
    expect(maskCpf("529")).toBe("529");
    expect(maskCpf("5299822")).toBe("529.982.2");
    expect(maskCpf("52998224725")).toBe("529.982.247-25");
  });
});

describe("isValidCpf", () => {
  it("confere o dígito verificador", () => {
    expect(isValidCpf("529.982.247-25")).toBe(true);
    expect(isValidCpf("529.982.247-24")).toBe(false);
    expect(isValidCpf("111.111.111-11")).toBe(false);
  });
});
```

```ts
// src/lib/citizenApi.test.ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { citizenApi, ApiError } from "./citizenApi";

afterEach(() => vi.unstubAllGlobals());

function mockFetch(status: number, body?: unknown) {
  const fn = vi.fn(async () => new Response(body === undefined ? null : JSON.stringify(body), {
    status, headers: { "Content-Type": "application/json" }
  }));
  vi.stubGlobal("fetch", fn);
  return fn;
}

describe("citizenApi", () => {
  it("POST manda JSON, com cookie da mesma origem", async () => {
    const fn = mockFetch(202, { status: "sent", resend_after: 60 });
    await citizenApi.requestCode("(41) 99876-5432");
    const [url, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(url).toBe("/citizen/otp");
    expect(init.method).toBe("POST");
    expect(init.credentials).toBe("same-origin");
    expect((init.headers as Record<string, string>)["Content-Type"]).toBe("application/json");
    expect(JSON.parse(init.body as string)).toEqual({ phone: "(41) 99876-5432" });
  });

  it("erro vira ApiError com status e código", async () => {
    mockFetch(422, { error: "invalid_phone" });
    await expect(citizenApi.requestCode("x")).rejects.toEqual(new ApiError(422, "invalid_phone"));
  });

  it("204 devolve undefined", async () => {
    mockFetch(204);
    await expect(citizenApi.signOut()).resolves.toBeUndefined();
  });

  it("start manda citizen_id ou cpf e a versão do termo", async () => {
    const fn = mockFetch(201, { conversation_id: "c", citizen_id: "p", resumed: false, step: {} });
    await citizenApi.start({ cpf: "529.982.247-25", consentVersion: "1" });
    const init = (fn.mock.calls[0] as unknown as [string, RequestInit])[1];
    expect(JSON.parse(init.body as string)).toEqual({ cpf: "529.982.247-25", consent_version: "1" });
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `npm test`
Expected: FAIL (módulos inexistentes).

- [ ] **Step 4: Implementação**

```ts
// src/lib/masks.ts
// Máscaras e validações do canal web do cidadão. A API valida de novo
// (CitizenIdentity); aqui é só para o cidadão ver o erro na hora.
export const onlyDigits = (s: string) => s.replace(/\D/g, "");

export function maskPhone(input: string): string {
  const d = onlyDigits(input).slice(0, 11);
  if (d.length === 0) return "";
  if (d.length <= 2) return `(${d}`;
  if (d.length <= 7) return `(${d.slice(0, 2)}) ${d.slice(2)}`;
  return `(${d.slice(0, 2)}) ${d.slice(2, 7)}-${d.slice(7)}`;
}

export function isMobilePhone(input: string): boolean {
  return /^[1-9][1-9]9\d{8}$/.test(onlyDigits(input));
}

export function maskCpf(input: string): string {
  const d = onlyDigits(input).slice(0, 11);
  const head = [d.slice(0, 3), d.slice(3, 6), d.slice(6, 9)].filter(Boolean).join(".");
  return d.length > 9 ? `${head}-${d.slice(9)}` : head;
}

export function isValidCpf(input: string): boolean {
  const d = onlyDigits(input);
  if (d.length !== 11 || /^(\d)\1{10}$/.test(d)) return false;
  const n = d.split("").map(Number);
  const checkDigit = (len: number) => {
    const sum = n.slice(0, len).reduce((acc, x, i) => acc + x * (len + 1 - i), 0);
    const rest = (sum * 10) % 11;
    return rest === 10 ? 0 : rest;
  };
  return checkDigit(9) === n[9] && checkDigit(10) === n[10];
}
```

```ts
// src/lib/citizenApi.ts
// Cliente das rotas /citizen/* (apps/api, CitizenApi). Mesma origem: o cookie
// httpOnly `citizen_session` vai sozinho. Toda escrita é JSON (a API recusa
// o resto com 415).
export class ApiError extends Error {
  constructor(public status: number, public code: string) {
    super(code);
  }
}

export interface Option { id: string; title: string }

export interface Step {
  triage_id: string;
  step_id: string;
  prompt: string;
  answer_type: "boolean" | "enum" | "integer" | "text";
  options: Option[];
  index: number;
  total: number;
  can_undo: boolean;
}

export type AnswerState =
  | { status: "in_progress"; step: Step }
  | { status: string; triage_id: string };

export interface Person { id: string; cpf_masked: string; verification_level: "declared" | "verified" }

export interface TriageSummary {
  id: string;
  status: string;
  tier: string | null;
  priority: number | null;
  created_at: string;
  completed_at: string | null;
  report_url: string | null;
  consent_active: boolean;
}

export interface StartResult { conversation_id: string; citizen_id: string; resumed: boolean; step: Step }

async function call<T>(method: string, path: string, body?: unknown): Promise<T> {
  const write = method !== "GET";
  const res = await fetch(`/citizen${path}`, {
    method,
    credentials: "same-origin",
    headers: { Accept: "application/json", ...(write ? { "Content-Type": "application/json" } : {}) },
    body: write ? JSON.stringify(body ?? {}) : undefined
  });
  if (res.status === 204) return undefined as T;
  const data = await res.json().catch(() => ({}));
  if (!res.ok) throw new ApiError(res.status, (data as { error?: string }).error ?? `http_${res.status}`);
  return data as T;
}

export const citizenApi = {
  requestCode: (phone: string) =>
    call<{ status: string; resend_after: number }>("POST", "/otp", { phone }),
  verifyCode: (phone: string, code: string) =>
    call<{ phone_masked: string }>("POST", "/session", { phone, code }),
  currentSession: () => call<{ phone_masked: string }>("GET", "/session"),
  signOut: () => call<void>("DELETE", "/session"),
  consentTerm: () => call<{ version: string; body: string }>("GET", "/consent_term"),
  people: () => call<{ people: Person[] }>("GET", "/people"),
  start: (p: { citizenId?: string; cpf?: string; consentVersion: string }) =>
    call<StartResult>("POST", "/conversations", {
      ...(p.citizenId ? { citizen_id: p.citizenId } : { cpf: p.cpf }),
      consent_version: p.consentVersion
    }),
  answer: (conversationId: string, answer: string, idempotencyKey: string) =>
    call<AnswerState>("POST", `/conversations/${conversationId}/answers`, { answer, idempotency_key: idempotencyKey }),
  undo: (conversationId: string) => call<AnswerState>("POST", `/conversations/${conversationId}/undo`),
  triages: (citizenId: string) =>
    call<{ citizen: Person; triages: TriageSummary[] }>("GET", `/triages?citizen_id=${encodeURIComponent(citizenId)}`),
  triage: (id: string) => call<TriageSummary>("GET", `/triages/${id}`),
  revokeConsent: (id: string) => call<TriageSummary>("POST", `/triages/${id}/revoke_consent`)
};
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `npm test && npm run typecheck`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add package.json package-lock.json vitest.config.ts vite.config.ts src/test/setup.ts src/lib/masks.ts src/lib/masks.test.ts src/lib/citizenApi.ts src/lib/citizenApi.test.ts
/opt/homebrew/bin/git commit -m "feat: add citizen API client and input masks" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: wpda — telas de entrada (telefone, código, termo, pessoa)

**Repo:** `apps/wpda`.

**Files:**
- Create: `src/modules/citizen/ui.tsx`, `src/modules/citizen/PhoneStep.tsx`, `src/modules/citizen/CodeStep.tsx`, `src/modules/citizen/ConsentStep.tsx`, `src/modules/citizen/PeopleStep.tsx`
- Test: `src/modules/citizen/entry.test.tsx`

**Interfaces:**
- Consumes: `citizenApi`, `ApiError`, `maskPhone`, `isMobilePhone`, `maskCpf`, `isValidCpf`, `Person` (Task 9).
- Produces (props):
  - `PhoneStep({ onSent(phone: string) })`
  - `CodeStep({ phone, onVerified(), onChangePhone() })`
  - `ConsentStep({ onAccept(version: string), onDecline() })`
  - `PeopleStep({ onChoose(choice: { citizenId: string } | { cpf: string }), onHistory(citizenId: string) })`
  - `ui.tsx`: `Screen({ title, children, footer? })`, `BigButton(props: ButtonHTMLAttributes & { variant?: "primary" | "secondary" | "danger" })`, `Field({ label, error?, ...InputHTMLAttributes })`, `ErrorText({ children })`, `messageFor(error: unknown) -> string`.

Visual: alvos de toque ≥ 48 px, texto ≥ 18 px, ações principais embaixo. O acabamento visual vem do protótipo do Claude Design; aqui a meta é o fluxo funcional e acessível com os tokens atuais (`src/theme/global.css`).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/entry.test.tsx
import { describe, it, expect, vi, afterEach } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { PhoneStep } from "./PhoneStep";
import { CodeStep } from "./CodeStep";
import { ConsentStep } from "./ConsentStep";
import { PeopleStep } from "./PeopleStep";
import { citizenApi, ApiError } from "../../lib/citizenApi";

afterEach(() => vi.restoreAllMocks());

describe("PhoneStep", () => {
  it("só envia celular válido, com máscara", async () => {
    const spy = vi.spyOn(citizenApi, "requestCode").mockResolvedValue({ status: "sent", resend_after: 60 });
    const onSent = vi.fn();
    render(<PhoneStep onSent={onSent} />);
    const input = screen.getByLabelText("Seu celular");
    await userEvent.type(input, "4133334444");
    await userEvent.click(screen.getByRole("button", { name: "Receber código" }));
    expect(screen.getByText("Digite um celular com DDD, como (41) 99876-5432.")).toBeInTheDocument();
    expect(spy).not.toHaveBeenCalled();

    await userEvent.clear(input);
    await userEvent.type(input, "41998765432");
    expect(input).toHaveValue("(41) 99876-5432");
    await userEvent.click(screen.getByRole("button", { name: "Receber código" }));
    expect(onSent).toHaveBeenCalledWith("(41) 99876-5432");
  });
});

describe("CodeStep", () => {
  it("mostra o erro de código errado", async () => {
    vi.spyOn(citizenApi, "verifyCode").mockRejectedValue(new ApiError(422, "invalid_code"));
    render(<CodeStep phone="(41) 99876-5432" onVerified={vi.fn()} onChangePhone={vi.fn()} />);
    await userEvent.type(screen.getByLabelText("Código"), "123456");
    await userEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    expect(await screen.findByText("Código errado. Confira o SMS e tente de novo.")).toBeInTheDocument();
  });
});

describe("ConsentStep", () => {
  it("aceita com a versão do termo", async () => {
    vi.spyOn(citizenApi, "consentTerm").mockResolvedValue({ version: "3", body: "Texto do termo" });
    const onAccept = vi.fn();
    render(<ConsentStep onAccept={onAccept} onDecline={vi.fn()} />);
    expect(await screen.findByText("Texto do termo")).toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Concordo" }));
    expect(onAccept).toHaveBeenCalledWith("3");
  });
});

describe("PeopleStep", () => {
  it("escolhe uma pessoa da lista ou valida o CPF novo", async () => {
    vi.spyOn(citizenApi, "people").mockResolvedValue({
      people: [{ id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared" }]
    });
    const onChoose = vi.fn();
    render(<PeopleStep onChoose={onChoose} onHistory={vi.fn()} />);
    await userEvent.click(await screen.findByRole("button", { name: "CPF ***.982.247-**" }));
    expect(onChoose).toHaveBeenCalledWith({ citizenId: "p1" });

    await userEvent.type(screen.getByLabelText("CPF de outra pessoa"), "52998224724");
    await userEvent.click(screen.getByRole("button", { name: "Continuar com este CPF" }));
    expect(screen.getByText("CPF inválido. Confira os números.")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- src/modules/citizen/entry.test.tsx`
Expected: FAIL (módulos inexistentes).

- [ ] **Step 3: Implementação**

```tsx
// src/modules/citizen/ui.tsx
// Peças de tela do canal web do cidadão: alvos de toque de 48 px ou mais,
// texto de 18 px, ação principal embaixo (uso com uma mão).
import type { ButtonHTMLAttributes, InputHTMLAttributes, ReactNode } from "react";
import { useId } from "react";
import { ApiError } from "../../lib/citizenApi";

export function Screen({ title, children, footer }: { title: string; children: ReactNode; footer?: ReactNode }) {
  return (
    <main style={{ minHeight: "100vh", display: "flex", flexDirection: "column", padding: 16,
      fontFamily: "system-ui, sans-serif", fontSize: 18, maxWidth: 520, margin: "0 auto" }}>
      <header style={{ marginBottom: 16 }}>
        <strong style={{ fontSize: 14, color: "var(--ink3, #666)" }}>Rota Saúde</strong>
        <h1 style={{ fontSize: 24, margin: "4px 0 0" }}>{title}</h1>
      </header>
      <div style={{ flex: 1 }}>{children}</div>
      {footer && <footer style={{ display: "grid", gap: 12, paddingTop: 16 }}>{footer}</footer>}
      <p style={{ fontSize: 14, color: "var(--ink3, #666)", textAlign: "center" }}>Em emergência, ligue 192.</p>
    </main>
  );
}

const variants = {
  primary: { background: "var(--accent, #2b4bd8)", color: "#fff", border: "none" },
  secondary: { background: "transparent", color: "var(--ink, #222)", border: "1px solid var(--rule2, #ccc)" },
  danger: { background: "var(--down, #c0392b)", color: "#fff", border: "none" }
} as const;

export function BigButton({ variant = "primary", style, ...props }:
  ButtonHTMLAttributes<HTMLButtonElement> & { variant?: keyof typeof variants }) {
  return (
    <button type="button" {...props}
      style={{ minHeight: 56, width: "100%", borderRadius: 12, fontSize: 18, fontWeight: 600,
        cursor: "pointer", opacity: props.disabled ? 0.6 : 1, ...variants[variant], ...style }} />
  );
}

export function Field({ label, error, ...props }: InputHTMLAttributes<HTMLInputElement> & { label: string; error?: string }) {
  const id = useId();
  return (
    <div style={{ display: "grid", gap: 6, marginBottom: 12 }}>
      <label htmlFor={id} style={{ fontWeight: 600 }}>{label}</label>
      <input id={id} {...props} aria-invalid={Boolean(error)}
        style={{ minHeight: 56, fontSize: 20, padding: "0 12px", borderRadius: 12,
          border: `1px solid ${error ? "var(--down, #c0392b)" : "var(--rule2, #ccc)"}` }} />
      {error && <ErrorText>{error}</ErrorText>}
    </div>
  );
}

export function ErrorText({ children }: { children: ReactNode }) {
  return <p role="alert" style={{ color: "var(--down, #c0392b)", margin: 0 }}>{children}</p>;
}

const MESSAGES: Record<string, string> = {
  invalid_phone: "Digite um celular com DDD, como (41) 99876-5432.",
  too_soon: "Aguarde um minuto antes de pedir outro código.",
  daily_limit: "Você pediu muitos códigos hoje. Tente amanhã.",
  too_many_requests: "Muitas tentativas. Aguarde um pouco e tente de novo.",
  otp_unavailable: "O envio de SMS está fora do ar. Tente em instantes.",
  invalid_code: "Código errado. Confira o SMS e tente de novo.",
  code_expired: "O código venceu. Peça um novo.",
  code_exhausted: "Muitas tentativas com este código. Peça um novo.",
  invalid_cpf: "CPF inválido. Confira os números.",
  too_many_people: "Este celular já tem 10 pessoas cadastradas.",
  consent_outdated: "O termo foi atualizado. Leia de novo para continuar.",
  no_protocol: "A triagem não está disponível agora nesta cidade.",
  no_consent_term: "A triagem não está disponível agora nesta cidade.",
  invalid_answer: "Não entendemos a resposta. Tente de novo.",
  city_schema_behind: "Serviço em manutenção. Tente em instantes."
};

export function messageFor(error: unknown): string {
  if (error instanceof ApiError) return MESSAGES[error.code] ?? "Algo deu errado. Tente de novo.";
  return "Sem conexão. Verifique a internet e tente de novo.";
}
```

```tsx
// src/modules/citizen/PhoneStep.tsx
import { useState } from "react";
import { citizenApi } from "../../lib/citizenApi";
import { isMobilePhone, maskPhone } from "../../lib/masks";
import { BigButton, ErrorText, Field, Screen, messageFor } from "./ui";

export function PhoneStep({ onSent }: { onSent: (phone: string) => void }) {
  const [phone, setPhone] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [busy, setBusy] = useState(false);

  async function submit() {
    if (!isMobilePhone(phone)) return setError("Digite um celular com DDD, como (41) 99876-5432.");
    setBusy(true);
    setError(null);
    try {
      await citizenApi.requestCode(phone);
      onSent(phone);
    } catch (e) {
      setError(messageFor(e));
    } finally {
      setBusy(false);
    }
  }

  return (
    <Screen title="Triagem de saúde"
      footer={<BigButton onClick={submit} disabled={busy}>Receber código</BigButton>}>
      <p>Vamos mandar um código por SMS para confirmar seu celular.</p>
      <Field label="Seu celular" inputMode="numeric" autoComplete="tel-national" value={phone}
        onChange={e => setPhone(maskPhone(e.target.value))} />
      {error && <ErrorText>{error}</ErrorText>}
    </Screen>
  );
}
```

```tsx
// src/modules/citizen/CodeStep.tsx
import { useEffect, useState } from "react";
import { citizenApi } from "../../lib/citizenApi";
import { onlyDigits } from "../../lib/masks";
import { BigButton, ErrorText, Field, Screen, messageFor } from "./ui";

export function CodeStep({ phone, onVerified, onChangePhone }:
  { phone: string; onVerified: () => void; onChangePhone: () => void }) {
  const [code, setCode] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [busy, setBusy] = useState(false);
  const [wait, setWait] = useState(60);

  useEffect(() => {
    if (wait <= 0) return;
    const t = setTimeout(() => setWait(w => w - 1), 1000);
    return () => clearTimeout(t);
  }, [wait]);

  async function submit() {
    setBusy(true);
    setError(null);
    try {
      await citizenApi.verifyCode(phone, code);
      onVerified();
    } catch (e) {
      setError(messageFor(e));
    } finally {
      setBusy(false);
    }
  }

  async function resend() {
    setError(null);
    try {
      await citizenApi.requestCode(phone);
      setWait(60);
    } catch (e) {
      setError(messageFor(e));
    }
  }

  return (
    <Screen title="Digite o código"
      footer={<>
        <BigButton onClick={submit} disabled={busy || code.length !== 6}>Confirmar</BigButton>
        <BigButton variant="secondary" onClick={resend} disabled={wait > 0}>
          {wait > 0 ? `Reenviar código em 0:${String(wait).padStart(2, "0")}` : "Reenviar código"}
        </BigButton>
        <BigButton variant="secondary" onClick={onChangePhone}>Trocar número</BigButton>
      </>}>
      <p>Enviamos um código de 6 números para {phone}.</p>
      <Field label="Código" inputMode="numeric" autoComplete="one-time-code" maxLength={6} value={code}
        onChange={e => setCode(onlyDigits(e.target.value).slice(0, 6))} />
      {error && <ErrorText>{error}</ErrorText>}
    </Screen>
  );
}
```

```tsx
// src/modules/citizen/ConsentStep.tsx
import { useEffect, useState } from "react";
import { citizenApi } from "../../lib/citizenApi";
import { BigButton, ErrorText, Screen, messageFor } from "./ui";

export function ConsentStep({ onAccept, onDecline }: { onAccept: (version: string) => void; onDecline: () => void }) {
  const [term, setTerm] = useState<{ version: string; body: string } | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    citizenApi.consentTerm().then(setTerm).catch(e => setError(messageFor(e)));
  }, []);

  return (
    <Screen title="Antes de começar"
      footer={term && <>
        <BigButton onClick={() => onAccept(term.version)}>Concordo</BigButton>
        <BigButton variant="secondary" onClick={onDecline}>Não concordo</BigButton>
      </>}>
      {error && <ErrorText>{error}</ErrorText>}
      {!term && !error && <p>Carregando…</p>}
      {term && <div style={{ whiteSpace: "pre-wrap", lineHeight: 1.5 }}>{term.body}</div>}
    </Screen>
  );
}
```

```tsx
// src/modules/citizen/PeopleStep.tsx
import { useEffect, useState } from "react";
import { citizenApi, type Person } from "../../lib/citizenApi";
import { isValidCpf, maskCpf } from "../../lib/masks";
import { BigButton, ErrorText, Field, Screen, messageFor } from "./ui";

export type PersonChoice = { citizenId: string } | { cpf: string };

export function PeopleStep({ onChoose, onHistory }:
  { onChoose: (c: PersonChoice) => void; onHistory: (citizenId: string) => void }) {
  const [people, setPeople] = useState<Person[] | null>(null);
  const [cpf, setCpf] = useState("");
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    citizenApi.people().then(r => setPeople(r.people)).catch(e => setError(messageFor(e)));
  }, []);

  function submitNew() {
    if (!isValidCpf(cpf)) return setError("CPF inválido. Confira os números.");
    onChoose({ cpf });
  }

  return (
    <Screen title="Para quem é esta triagem?">
      {people === null && !error && <p>Carregando…</p>}
      <div style={{ display: "grid", gap: 12, marginBottom: 24 }}>
        {people?.map(p => (
          <div key={p.id} style={{ display: "grid", gap: 8 }}>
            <BigButton variant="secondary" onClick={() => onChoose({ citizenId: p.id })}>
              CPF {p.cpf_masked}
            </BigButton>
            <button type="button" onClick={() => onHistory(p.id)}
              style={{ minHeight: 48, background: "none", border: "none", textDecoration: "underline", fontSize: 16 }}>
              Ver triagens de {p.cpf_masked}
            </button>
          </div>
        ))}
      </div>
      <Field label="CPF de outra pessoa" inputMode="numeric" value={cpf}
        onChange={e => setCpf(maskCpf(e.target.value))} />
      {error && <ErrorText>{error}</ErrorText>}
      <BigButton onClick={submitNew}>Continuar com este CPF</BigButton>
    </Screen>
  );
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/ui.tsx src/modules/citizen/PhoneStep.tsx src/modules/citizen/CodeStep.tsx src/modules/citizen/ConsentStep.tsx src/modules/citizen/PeopleStep.tsx src/modules/citizen/entry.test.tsx
/opt/homebrew/bin/git commit -m "feat: add phone, code, consent and person screens" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: wpda — pergunta, resultado, histórico e o fluxo

**Repo:** `apps/wpda`.

**Files:**
- Create: `src/modules/citizen/QuestionStep.tsx`, `src/modules/citizen/ResultStep.tsx`, `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/citizen/triage.test.tsx`, `src/modules/citizen/Flow.test.tsx`

**Interfaces:**
- Consumes: Tasks 9–10; `Report` (`src/modules/Report.tsx`, prop `token`).
- Produces:
  - `QuestionStep({ conversationId, step, onStep(step: Step), onCompleted(triageId: string) })`
  - `ResultStep({ triageId, onAgain(), onHistory() })` — consulta `citizenApi.triage` a cada 1 s, até 15 tentativas, até ter `report_url`; então mostra `<Report token=…/>`.
  - `HistoryStep({ citizenId, onBack() })`
  - `Flow()` — máquina de telas: `boot → phone → code → consent → people → question → result → history`, `declined`.
  - `App`: com `?token=` mostra o relatório (como hoje); sem token mostra `Flow`.

- [ ] **Step 1: Write the failing tests**

```tsx
// src/modules/citizen/triage.test.tsx
import { describe, it, expect, vi, afterEach } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { QuestionStep } from "./QuestionStep";
import { HistoryStep } from "./HistoryStep";
import { citizenApi, ApiError, type Step } from "../../lib/citizenApi";

afterEach(() => vi.restoreAllMocks());

const boolStep: Step = {
  triage_id: "t1", step_id: "tosse", prompt: "Você está com tosse?", answer_type: "boolean",
  options: [{ id: "true", title: "Sim" }, { id: "false", title: "Não" }], index: 1, total: 2, can_undo: false
};

describe("QuestionStep", () => {
  it("mostra progresso e manda o id da opção com uma chave de idempotência", async () => {
    const spy = vi.spyOn(citizenApi, "answer").mockResolvedValue({ status: "in_progress", step: { ...boolStep, index: 2 } });
    const onStep = vi.fn();
    render(<QuestionStep conversationId="c1" step={boolStep} onStep={onStep} onCompleted={vi.fn()} />);
    expect(screen.getByText("Pergunta 1 de 2")).toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Voltar" })).not.toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Sim" }));
    expect(spy).toHaveBeenCalledWith("c1", "true", expect.any(String));
    expect(onStep).toHaveBeenCalledWith(expect.objectContaining({ index: 2 }));
  });

  it("enum com muitas opções mostra todas", () => {
    const options = Array.from({ length: 12 }, (_, i) => ({ id: `o${i}`, title: `Opção ${i}` }));
    render(<QuestionStep conversationId="c1" step={{ ...boolStep, answer_type: "enum", options }}
      onStep={vi.fn()} onCompleted={vi.fn()} />);
    expect(screen.getAllByRole("button", { name: /Opção/ })).toHaveLength(12);
  });

  it("número usa teclado numérico e envia o valor", async () => {
    const spy = vi.spyOn(citizenApi, "answer").mockResolvedValue({ status: "completed", triage_id: "t1" });
    const onCompleted = vi.fn();
    render(<QuestionStep conversationId="c1" step={{ ...boolStep, answer_type: "integer", options: [], prompt: "Há quantos dias?" }}
      onStep={vi.fn()} onCompleted={onCompleted} />);
    await userEvent.type(screen.getByLabelText("Há quantos dias?"), "3");
    await userEvent.click(screen.getByRole("button", { name: "Continuar" }));
    expect(spy).toHaveBeenCalledWith("c1", "3", expect.any(String));
    expect(onCompleted).toHaveBeenCalledWith("t1");
  });

  it("voltar chama undo", async () => {
    const spy = vi.spyOn(citizenApi, "undo").mockResolvedValue({ status: "in_progress", step: boolStep });
    render(<QuestionStep conversationId="c1" step={{ ...boolStep, can_undo: true }} onStep={vi.fn()} onCompleted={vi.fn()} />);
    await userEvent.click(screen.getByRole("button", { name: "Voltar" }));
    expect(spy).toHaveBeenCalledWith("c1");
  });

  it("resposta recusada mostra erro e mantém a pergunta", async () => {
    vi.spyOn(citizenApi, "answer").mockRejectedValue(new ApiError(422, "invalid_answer"));
    render(<QuestionStep conversationId="c1" step={boolStep} onStep={vi.fn()} onCompleted={vi.fn()} />);
    await userEvent.click(screen.getByRole("button", { name: "Não" }));
    expect(await screen.findByText("Não entendemos a resposta. Tente de novo.")).toBeInTheDocument();
    expect(screen.getByText("Você está com tosse?")).toBeInTheDocument();
  });
});

describe("HistoryStep", () => {
  it("lista triagens e avisa que o cadastro não é verificado", async () => {
    vi.spyOn(citizenApi, "triages").mockResolvedValue({
      citizen: { id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared" },
      triages: [{ id: "t1", status: "completed", tier: "alta", priority: 1, created_at: "2026-09-22T12:00:00Z",
                  completed_at: "2026-09-22T12:05:00Z", report_url: "http://x/wpda/?token=abc", consent_active: true }]
    });
    render(<HistoryStep citizenId="p1" onBack={vi.fn()} />);
    expect(await screen.findByText(/Cadastro não verificado/)).toBeInTheDocument();
    expect(screen.getByText(/Prioridade alta/)).toBeInTheDocument();
  });
});
```

```tsx
// src/modules/citizen/Flow.test.tsx
import { describe, it, expect, vi, afterEach } from "vitest";
import { render, screen } from "@testing-library/react";
import { Flow } from "./Flow";
import { citizenApi, ApiError } from "../../lib/citizenApi";

afterEach(() => vi.restoreAllMocks());

describe("Flow", () => {
  it("sem sessão, começa pelo telefone", async () => {
    vi.spyOn(citizenApi, "currentSession").mockRejectedValue(new ApiError(401, "unauthenticated"));
    render(<Flow />);
    expect(await screen.findByLabelText("Seu celular")).toBeInTheDocument();
  });

  it("com sessão, vai direto ao termo", async () => {
    vi.spyOn(citizenApi, "currentSession").mockResolvedValue({ phone_masked: "(**) *****-5432" });
    vi.spyOn(citizenApi, "consentTerm").mockResolvedValue({ version: "1", body: "Termo" });
    render(<Flow />);
    expect(await screen.findByRole("button", { name: "Concordo" })).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- src/modules/citizen/triage.test.tsx src/modules/citizen/Flow.test.tsx`
Expected: FAIL (módulos inexistentes).

- [ ] **Step 3: Implementação**

```tsx
// src/modules/citizen/QuestionStep.tsx
// Uma pergunta por tela (spec §2.7). A chave de idempotência é uma por
// pergunta mostrada: um toque duplo manda a mesma chave e não avança duas vezes.
import { useMemo, useState } from "react";
import { citizenApi, type AnswerState, type Step } from "../../lib/citizenApi";
import { onlyDigits } from "../../lib/masks";
import { BigButton, ErrorText, Field, Screen, messageFor } from "./ui";

export function QuestionStep({ conversationId, step, onStep, onCompleted }: {
  conversationId: string; step: Step; onStep: (s: Step) => void; onCompleted: (triageId: string) => void;
}) {
  const [value, setValue] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [busy, setBusy] = useState(false);
  const key = useMemo(() => crypto.randomUUID(), [step.triage_id, step.step_id, step.index]);

  function handle(state: AnswerState) {
    setValue("");
    if (state.status === "in_progress" && "step" in state) onStep(state.step);
    else if ("triage_id" in state) onCompleted(state.triage_id);
  }

  async function send(answer: string) {
    setBusy(true);
    setError(null);
    try {
      handle(await citizenApi.answer(conversationId, answer, key));
    } catch (e) {
      setError(messageFor(e));
    } finally {
      setBusy(false);
    }
  }

  async function back() {
    setBusy(true);
    setError(null);
    try {
      handle(await citizenApi.undo(conversationId));
    } catch (e) {
      setError(messageFor(e));
    } finally {
      setBusy(false);
    }
  }

  const choice = step.answer_type === "boolean" || step.answer_type === "enum";

  return (
    <Screen title={step.prompt}
      footer={<>
        {!choice && <BigButton onClick={() => send(value)} disabled={busy || value.trim() === ""}>Continuar</BigButton>}
        {step.can_undo && <BigButton variant="secondary" onClick={back} disabled={busy}>Voltar</BigButton>}
      </>}>
      <p aria-live="polite" style={{ margin: "0 0 8px", color: "var(--ink2, #444)" }}>
        Pergunta {step.index} de {step.total}
      </p>
      <progress value={step.index} max={step.total} style={{ width: "100%", height: 8, marginBottom: 16 }} />
      {choice && (
        <div style={{ display: "grid", gap: 12 }}>
          {step.options.map(o => (
            <BigButton key={o.id} variant="secondary" disabled={busy} onClick={() => send(o.id)}>{o.title}</BigButton>
          ))}
        </div>
      )}
      {step.answer_type === "integer" && (
        <Field label={step.prompt} inputMode="numeric" value={value}
          onChange={e => setValue(onlyDigits(e.target.value).slice(0, 4))} />
      )}
      {step.answer_type === "text" && (
        <Field label={step.prompt} maxLength={500} value={value} onChange={e => setValue(e.target.value)} />
      )}
      {error && <ErrorText>{error}</ErrorText>}
    </Screen>
  );
}
```

```tsx
// src/modules/citizen/ResultStep.tsx
// O relatório é gerado por um job depois de triage.completed: consulta a
// triagem até o link aparecer (spec §3.1) e então mostra o mesmo relatório que
// o link do WhatsApp mostrava.
import { useEffect, useState } from "react";
import { citizenApi } from "../../lib/citizenApi";
import { tokenFromUrl } from "../../lib/report";
import { Report } from "../Report";
import { BigButton, Screen } from "./ui";

const TRIES = 15;

export function ResultStep({ triageId, onAgain, onHistory }:
  { triageId: string; onAgain: () => void; onHistory: () => void }) {
  const [token, setToken] = useState<string | null>(null);
  const [gaveUp, setGaveUp] = useState(false);

  useEffect(() => {
    let alive = true;
    let tries = 0;
    async function poll() {
      tries += 1;
      try {
        const t = await citizenApi.triage(triageId);
        const found = t.report_url ? tokenFromUrl(new URL(t.report_url).search) : null;
        if (found) { if (alive) setToken(found); return; }
      } catch { /* tenta de novo */ }
      if (tries >= TRIES) { if (alive) setGaveUp(true); return; }
      if (alive) setTimeout(poll, 1000);
    }
    poll();
    return () => { alive = false; };
  }, [triageId]);

  const actions = (
    <div style={{ display: "grid", gap: 12, padding: 16, maxWidth: 520, margin: "0 auto" }}>
      <BigButton onClick={onAgain}>Fazer outra triagem</BigButton>
      <BigButton variant="secondary" onClick={onHistory}>Minhas triagens</BigButton>
    </div>
  );

  if (token) return <><Report token={token} />{actions}</>;
  return (
    <Screen title="Triagem concluída" footer={actions}>
      <p>{gaveUp ? "Seu resultado ainda está sendo preparado. Veja em \"Minhas triagens\" daqui a pouco." : "Preparando seu resultado…"}</p>
    </Screen>
  );
}
```

```tsx
// src/modules/citizen/HistoryStep.tsx
import { useEffect, useState } from "react";
import { citizenApi, type Person, type TriageSummary } from "../../lib/citizenApi";
import { fmtDateTime } from "../../lib/format";
import { BigButton, ErrorText, Screen, messageFor } from "./ui";

export function HistoryStep({ citizenId, onBack }: { citizenId: string; onBack: () => void }) {
  const [data, setData] = useState<{ citizen: Person; triages: TriageSummary[] } | null>(null);
  const [error, setError] = useState<string | null>(null);

  function load() {
    citizenApi.triages(citizenId).then(setData).catch(e => setError(messageFor(e)));
  }
  useEffect(load, [citizenId]);

  async function revoke(id: string) {
    if (!window.confirm("Revogar o consentimento apaga as respostas desta triagem. Continuar?")) return;
    try { await citizenApi.revokeConsent(id); load(); } catch (e) { setError(messageFor(e)); }
  }

  return (
    <Screen title="Minhas triagens" footer={<BigButton variant="secondary" onClick={onBack}>Voltar</BigButton>}>
      {error && <ErrorText>{error}</ErrorText>}
      {data && (
        <>
          <p>CPF {data.citizen.cpf_masked}</p>
          {data.citizen.verification_level === "declared" && (
            <p style={{ background: "var(--warnBg, #fff4e0)", padding: 12, borderRadius: 12 }}>
              Cadastro não verificado. Leve um documento com foto ao posto de saúde para ver seu histórico completo.
            </p>
          )}
          {data.triages.length === 0 && <p>Nenhuma triagem ainda.</p>}
          <ul style={{ listStyle: "none", padding: 0, display: "grid", gap: 12 }}>
            {data.triages.map(t => (
              <li key={t.id} style={{ border: "1px solid var(--rule2, #ccc)", borderRadius: 12, padding: 12 }}>
                <strong>{t.tier ? `Prioridade ${t.tier}` : "Em andamento"}</strong>
                <div style={{ fontSize: 16 }}>{fmtDateTime(t.completed_at ?? t.created_at)}</div>
                {t.report_url && <a href={t.report_url} style={{ display: "inline-block", minHeight: 48, lineHeight: "48px" }}>Ver relatório</a>}
                {t.consent_active && (
                  <button type="button" onClick={() => revoke(t.id)}
                    style={{ minHeight: 48, background: "none", border: "none", textDecoration: "underline", fontSize: 16 }}>
                    Revogar consentimento
                  </button>
                )}
              </li>
            ))}
          </ul>
        </>
      )}
    </Screen>
  );
}
```

Confira a assinatura de `fmtDateTime` em `src/lib/format.ts` (aceita `string | null`?) e ajuste a chamada se precisar.

```tsx
// src/modules/citizen/Flow.tsx
// Máquina de telas do canal web do cidadão (spec §4). Sem biblioteca de rotas:
// o estado mora aqui, e um F5 volta ao começo com a sessão (cookie) preservada.
import { useEffect, useState } from "react";
import { citizenApi, type Step } from "../../lib/citizenApi";
import { PhoneStep } from "./PhoneStep";
import { CodeStep } from "./CodeStep";
import { ConsentStep } from "./ConsentStep";
import { PeopleStep, type PersonChoice } from "./PeopleStep";
import { QuestionStep } from "./QuestionStep";
import { ResultStep } from "./ResultStep";
import { HistoryStep } from "./HistoryStep";
import { BigButton, ErrorText, Screen, messageFor } from "./ui";

type State =
  | { at: "boot" }
  | { at: "phone" }
  | { at: "code"; phone: string }
  | { at: "consent" }
  | { at: "people"; consentVersion: string }
  | { at: "question"; consentVersion: string; conversationId: string; citizenId: string; step: Step }
  | { at: "result"; consentVersion: string; triageId: string; citizenId: string }
  | { at: "history"; consentVersion: string | null; citizenId: string }
  | { at: "declined" };

export function Flow() {
  const [state, setState] = useState<State>({ at: "boot" });
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    citizenApi.currentSession()
      .then(() => setState({ at: "consent" }))
      .catch(() => setState({ at: "phone" }));
  }, []);

  async function choose(consentVersion: string, choice: PersonChoice) {
    setError(null);
    try {
      const r = await citizenApi.start({ ...choice, consentVersion });
      setState({ at: "question", consentVersion, conversationId: r.conversation_id, citizenId: r.citizen_id, step: r.step });
    } catch (e) {
      setError(messageFor(e));
    }
  }

  async function signOut() {
    try { await citizenApi.signOut(); } finally { setState({ at: "phone" }); }
  }

  const exit = state.at !== "boot" && state.at !== "phone" && state.at !== "code" && (
    <button type="button" onClick={signOut}
      style={{ position: "fixed", top: 8, right: 8, minHeight: 48, background: "none", border: "none", fontSize: 16 }}>
      Sair
    </button>
  );

  let view;
  switch (state.at) {
    case "boot": view = <Screen title="Triagem de saúde"><p>Carregando…</p></Screen>; break;
    case "phone": view = <PhoneStep onSent={phone => setState({ at: "code", phone })} />; break;
    case "code":
      view = <CodeStep phone={state.phone} onVerified={() => setState({ at: "consent" })}
        onChangePhone={() => setState({ at: "phone" })} />;
      break;
    case "consent":
      view = <ConsentStep onAccept={v => setState({ at: "people", consentVersion: v })}
        onDecline={() => setState({ at: "declined" })} />;
      break;
    case "people":
      view = <>
        {error && <ErrorText>{error}</ErrorText>}
        <PeopleStep onChoose={c => choose(state.consentVersion, c)}
          onHistory={id => setState({ at: "history", consentVersion: state.consentVersion, citizenId: id })} />
      </>;
      break;
    case "question":
      view = <QuestionStep conversationId={state.conversationId} step={state.step}
        onStep={step => setState({ ...state, step })}
        onCompleted={triageId => setState({ at: "result", consentVersion: state.consentVersion, triageId, citizenId: state.citizenId })} />;
      break;
    case "result":
      view = <ResultStep triageId={state.triageId}
        onAgain={() => setState({ at: "people", consentVersion: state.consentVersion })}
        onHistory={() => setState({ at: "history", consentVersion: state.consentVersion, citizenId: state.citizenId })} />;
      break;
    case "history":
      view = <HistoryStep citizenId={state.citizenId}
        onBack={() => setState(state.consentVersion ? { at: "people", consentVersion: state.consentVersion } : { at: "consent" })} />;
      break;
    case "declined":
      view = <Screen title="Tudo bem" footer={<BigButton onClick={() => setState({ at: "consent" })}>Ler o termo de novo</BigButton>}>
        <p>Sem o seu consentimento não fazemos a triagem, e nenhum dado seu foi guardado.</p>
      </Screen>;
      break;
  }

  return <>{exit}{view}</>;
}
```

Em `src/App.tsx`: sem token, em vez da tela "Link inválido", renderize `<Flow />` (importe de `./modules/citizen/Flow`). Com token, siga mostrando `<Report token={token} />`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test && npm run typecheck && npm run build`
Expected: PASS e build sem erro.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/QuestionStep.tsx src/modules/citizen/ResultStep.tsx src/modules/citizen/HistoryStep.tsx src/modules/citizen/Flow.tsx src/modules/citizen/triage.test.tsx src/modules/citizen/Flow.test.tsx src/App.tsx
/opt/homebrew/bin/git commit -m "feat: run the citizen triage step by step in the wpda" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Verificação de ponta a ponta em desenvolvimento

**Repos:** `apps/api`, `apps/wpda` (sem código novo, a menos que a verificação ache defeito).

- [ ] **Step 1: Migrar as cidades de dev e garantir termo e protocolo**

Na raiz do monorepo:

```bash
docker compose exec -T api bin/rails city:migrate:all
docker compose restart api
```

Confira se a cidade de dev (curitiba) tem termo de consentimento e protocolo ativo:

```bash
docker compose exec -T api bin/rails runner 'c = City.find_by!(slug: "curitiba"); CityConnection.with(c) { puts ConsentTerm.count; puts ProtocolDefinition.where(name: StartTriage::DEFAULT_PROTOCOL_NAME, status: "active").count }'
```

Se `ConsentTerm.count` for 0, crie um termo **de desenvolvimento** (o texto real depende do jurídico, spec §7):

```bash
docker compose exec -T api bin/rails runner 'c = City.find_by!(slug: "curitiba"); CityConnection.with(c) { ConsentTerm.create!(version: "1", body: "Termo de desenvolvimento — texto real pendente de validação jurídica.", published_at: Time.current) }'
```

Se não houver protocolo ativo, siga `db/seeds.rb` para ativar o `triage-respiratoria` da cidade.

- [ ] **Step 2: Subir o wpda e fazer a triagem no navegador**

`docker compose up -d wpda`, depois abra `http://curitiba.localhost:5176/wpda/` no navegador (preview). Faça o fluxo inteiro: celular → código (pegue com `docker compose logs api | grep "\[otp\]" | tail -1`) → termo → CPF novo `529.982.247-25` → responda → resultado com o relatório → "Minhas triagens". Teste também "Voltar" numa pergunta, um segundo CPF no mesmo celular e "Sair".

- [ ] **Step 3: Conferir no dashboard**

Entre no dashboard da mesma cidade e confira que a triagem aparece nas telas de triagens e nas métricas, como uma triagem do WhatsApp apareceria.

- [ ] **Step 4: Roteamento em produção**

Descubra como `/r/` chega à api no host da cidade em produção (procure `"/r"` ou `path` em `apps/*/deploy/production/deploy.yml` e na configuração do proxy de borda). `/citizen/` precisa seguir o mesmo caminho. Se a configuração estiver fora destes repositórios, registre como pendência de go-live no relatório final, sem inventar.

- [ ] **Step 5: Relatório**

Registre em texto: o que foi testado, prints das telas principais, e as pendências de go-live (provedor de SMS, texto jurídico do termo, roteamento de `/citizen` em produção, QR code nas UBS). Se o Step 2 ou 3 achar defeito, conserte na task dona com teste que o reproduza antes.

---

## Pendências de go-live (fora deste plano)

- Contratar o provedor de SMS e implementar o `OtpSender` dele (a interface já existe; em produção, sem provedor, `/citizen/otp` responde 503).
- Texto jurídico e base legal da nova versão do `ConsentTerm`.
- Roteamento de `/citizen/*` para a api no host da cidade em produção (Task 12, Step 4).
- Desligar o `CityChannel` do WhatsApp por cidade quando a web entrar no ar.
