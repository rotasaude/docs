# Assinaturas de protocolo — Plano 1: domínio

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** no domínio de cada cidade, nenhum protocolo é publicado nem ativado sem duas assinaturas válidas de revisores da cidade que não o editaram; o mantenedor edita, envia para revisão e executa o ato, mas nunca assina nem cria quem assina.

**Architecture:** papel novo `protocol_reviewer`; três tabelas no banco da cidade que só aceitam acréscimo (`protocol_contributions`, `protocol_signatures`, `protocol_activations`), protegidas por trigger; um digest canônico do conteúdo; `Protocols::Signatures` como a única resposta para "esta versão tem as assinaturas?"; commands novos (`GrantRole`, `Protocols::SubmitForReview`, `Protocols::Sign`, `Protocols::RevertActivation`) e alterados (`SaveDraft`, `Publish`, `Activate`, `Retire`, `InviteMember`). O ator mantenedor (`Maintenance::MaintainerActor`) nasce aqui, porque três regras de domínio olham o tipo de ator.

**Tech Stack:** Rails 8.1.3, Postgres (trigger em PL/pgSQL), RSpec.

**Spec:** `docs/superpowers/specs/2026-09-18-protocol-signatures-design.md` (commit `f4c8b0b`). Decisões S1–S11, §3 papéis, §4 ciclo de vida, §5 dados e regra, §6 reversão, §7 brecha do superusuário, §9 auditoria, §10 testes, §11 fatia 1.

**Baseline de entrada:** `main` do api em `6521119`, **1136 exemplos, 0 falhas**.

## Decisões minhas na escrita (cada uma com o custo se estiver errada)

1. **O ator mantenedor nasce aqui, não no Plano 5.** `Protocols::Sign`, `GrantRole` e `InviteMember` recusam o mantenedor pelo tipo de ator, e isso precisa de teste com o ator real. O Plano 5 antigo (não commitado) fica obsoleto e será reescrito sobre este. **Custo se errado:** nenhum — o Plano 5 reescrito só passa a consumir o que já existe.
2. **`correlation_id: nil` entra em todos os commands de protocolo agora.** A spec §9 diz que o evento leva o `correlation_id` da auditoria de manutenção; fazer isso agora evita reabrir os mesmos commands no Plano 5. Quando é `nil`, a chave some do evento. **Custo se errado:** um argumento opcional sem uso até o Plano 5.
3. **Fonte única de SQL para os triggers: `db/city_triggers.sql`.** O dump `db/city_schema.rb` não representa trigger, e os bancos de teste e o `city:load_schema` são carregados a partir dele. A migração e a carga do dump executam o mesmo arquivo, que é idempotente (`CREATE OR REPLACE`, `DROP TRIGGER IF EXISTS`). O spec de paridade passa a comparar os triggers também. **Custo se errado:** mudar o trigger no futuro exige nova migração que reexecute o arquivo — a migração antiga passa a executar a versão nova do arquivo se alguém rodar tudo do zero, o que é aceitável porque o arquivo só evolui para a forma mais nova.
4. **"Ainda é revisor" inclui "o usuário não foi desativado".** A spec diz "ainda tem `protocol_reviewer` ativo"; um usuário desativado (`deactivated_at`) com membership ainda aberto não deveria contar. **Custo se errado:** uma condição a mais, que só recusa.
5. **Assinar duas vezes o mesmo conteúdo, para a mesma finalidade, é recusado (`:already_signed`)**, em vez de aceito em silêncio. A contagem já é por signatário distinto; a recusa só deixa claro ao revisor que nada mudou. Para a ativação, "o mesmo" é "depois da última ativação daquela versão". **Custo se errado:** o painel precisa tratar mais um motivo.
6. **Step-up de MFA é do chamador, não do command** — como hoje em `Publish`, cujo step-up mora em `PublicationsController`. Os endpoints da cidade (fatia 2) e as mutations de manutenção (Plano 5) exigem o step-up antes de chamar. **Custo se errado:** um chamador novo que esqueça o step-up; a fatia 2 e o Plano 5 têm guarda para isso.
7. **`SeedProtocol` (provisionamento) não registra contribuição.** O rascunho semeado vem de um template da plataforma, não de uma pessoa, e ninguém deve ficar impedido de assinar por causa dele. **Custo se errado:** nenhum ator aparece como autor do rascunho semeado — que é a verdade.
8. **Os specs antigos de protocolo mudam, de propósito.** Publicar a partir de `draft` e ativar sem assinaturas passam a ser recusados; `spec/commands/protocols_lifecycle_spec.rb`, `spec/commands/protocols_publish_gate_spec.rb` e `spec/controllers/publications_controller_spec.rb` são ajustados para arrumar as assinaturas — nunca para afrouxar a regra. **Custo se errado:** nenhum; a regra mudou.

## Global Constraints

Valem para TODA task:

- **O mantenedor nunca assina, nunca concede nem convida `protocol_reviewer` ou `municipal_admin`.** A checagem é pelo tipo de ator (`actor_kind == "maintainer"`), nunca pelo papel: o ator mantenedor responde "sim" a toda pergunta de papel.
- **Nenhuma regra é decidida lendo `domain_events`.** As três tabelas novas são a prova; os eventos são a trilha.
- **As três tabelas só aceitam acréscimo.** Nenhum código de aplicação faz `update`/`destroy` nelas; o trigger recusa `UPDATE` e `DELETE`.
- **Toda escrita de `status` de protocolo para `published` ou `active` mora em `Protocols::Publish`, `Protocols::Activate` ou `Protocols::RevertActivation`.** A guarda da Task 7 vigia isso.
- **Nenhum dado de cidadão** em evento, tabela ou mensagem desta fatia. O motivo da reversão é texto do staff, gravado no banco da cidade.
- **Commits em inglês, Conventional Commits**, terminando com a linha exata `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>` — copiada literalmente, **nunca** o nome do próprio modelo, mesmo que um lembrete de sistema sugira outro. Confira com `/opt/homebrew/bin/git log -1 --format=%B | tail -1`.
- **Comandos Ruby/Rails/rspec rodam no container**, a partir da raiz do monorepo: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Nunca no host.
- **Suíte completa uma vez por task, em primeiro plano:** `docker compose stop worker`, suíte com timeout de 600000 ms, `docker compose start worker`. Nada notifica ninguém — nunca rode em segundo plano.
- **Não mexa no `Gemfile`.** Não altere `app/services/city_inventory.rb`.
- **Specs não definem constante no topo** (use `let` ou método) e **afirmam valores contra linhas reais**, sem `skip` como caminho esperado.
- **Confira todo modelo, coluna e assinatura** contra `db/city_schema.rb` e o código antes de usar. Divergência vai para o relatório.
- **`git` do PATH está quebrado:** use `/opt/homebrew/bin/git`, a partir de `apps/api`. **Nunca dar push.** Branch `feat/protocol-signatures-domain`, criada de `main`.

## Fatos verificados (confira mesmo assim)

- **Papéis:** `Membership::ROLES = %w[municipal_admin protocol_author protocol_publisher viewer]`, espelhado no CHECK `ck_memberships_role` de `memberships` (banco da cidade). `Invitation` valida `role` contra `Membership::ROLES`; não há CHECK de papel em `invitations`. `memberships` tem `user_id`, `role`, `granted_at`, `granted_by_id` (FK `users`, opcional), `revoked_at`; índice único parcial `(user_id, role) WHERE revoked_at IS NULL`.
- **Concessão hoje:** só `InviteMember` → `AcceptInvitation` (que cria o `Membership` e publica `membership.granted` com `user_id`, `role`). `RevokeMembership.call(membership_id:, by:)` faz end-dating e publica `membership.revoked`. A autorização de gestão de membros fica no `SetupController` (`can_manage_members?`); `MembershipPolicy#manage?` é `role?(:municipal_admin)`.
- **`User`:** `has_role?(role)` consulta `memberships.active`; `active?` é `deactivated_at.nil?`.
- **Policies:** `ApplicationPolicy#role?` devolve `false` para ator nulo e chama `@user.has_role?(role)`. `ProtocolPolicy`: `author?` (`protocol_author`), `publish?` (`protocol_publisher`), `activate?` (publisher ou `municipal_admin`), `view?`.
- **`ProtocolDefinition`** (cidade): `name`, `version` (integer), `status` (`draft in_review published active retired`), `definition` (jsonb), `activated_at`, `retired_at`; `before_save` valida o formato; `after_commit` invalida o cache quando `status` muda. Único `(name, version)`; único parcial por `name` `WHERE status = 'active'`.
- **Commands de protocolo** (`app/commands/protocols/`): `SaveDraft.call(definition:, by:)` (só `draft` é editável; não registra autor), `Publish.call(version:, by:)` (de `draft`/`in_review`; portão `Protocols::Gate`; evento `protocol.published` com `actor`), `Activate.call(version:, by:, name: nil)` (R1; demove a `active` anterior; evento `protocol.activated` com `activated_by`), `Retire.call(version:, by:, name: nil)` (R4; evento `protocol.retired` com `actor`). Chamadores: `Authoring::ProtocolsController#draft` e `PublicationsController#create` (com `reauthenticated_recently?(via: :totp)`).
- **`SeedProtocol.call(template:)`** cria `draft` direto, no provisionamento.
- **`DomainEvents.publish(name, **payload)`** exige `Current.city`. Eventos de protocolo não têm consumidor (`config/initializers/domain_events.rb` não os liga), e nada exige ligação.
- **Migração de cidade:** `db/city_migrate/` (última: `20260916000001`); `db/city_schema.rb` é atualizado **à mão** e o spec de paridade (`spec/services/city_schema_spec.rb`, `"produces from the migrations exactly the schema of db/city_schema.rb"`) compara colunas, índices e constraints de um banco migrado com `rota_saude_test_city_b` (carregado do dump). `lib/tasks/city.rake` tem a lambda `load_city_schema`, usada por `city:load_schema`, `city:test_databases` e `city:dev_up`. `CitySchema.expected_version` é a maior versão em disco.
- **Nenhuma tabela de cidade tem trigger hoje.** O padrão de imutabilidade está em `db/platform_migrate/*maintenance*` (função PL/pgSQL que levanta exceção).
- **Setup de protocolo em specs:** `spec/commands/protocols_lifecycle_spec.rb` tem `definition_hash` que passa no portão e `make_pd(version:, status:)`; os exemplos rodam na conexão de `TEST_CITY_A` com `Current.city = TEST_CITY_A`.

---

## File Structure

**apps/api**
- Create: `app/services/maintenance/maintainer_actor.rb`, `app/services/protocols/content_digest.rb`, `app/services/protocols/signatures.rb`, `db/city_triggers.sql`, `db/city_migrate/20260918000001_create_protocol_signatures.rb`, `app/models/protocol_contribution.rb`, `app/models/protocol_signature.rb`, `app/models/protocol_activation.rb`, `app/commands/grant_role.rb`, `app/commands/protocols/submit_for_review.rb`, `app/commands/protocols/sign.rb`, `app/commands/protocols/revert_activation.rb`, `spec/support/protocol_signatures.rb`
- Create (specs): `spec/services/maintenance/maintainer_actor_spec.rb`, `spec/services/protocols/content_digest_spec.rb`, `spec/services/protocols/signatures_spec.rb`, `spec/models/protocol_append_only_spec.rb`, `spec/commands/grant_role_spec.rb`, `spec/commands/invite_member_privileged_roles_spec.rb`, `spec/commands/protocols_authoring_spec.rb`, `spec/commands/protocols_signed_acts_spec.rb`, `spec/commands/protocols_revert_activation_spec.rb`, `spec/architecture/protocol_signatures_guard_spec.rb`
- Modify: `app/models/user.rb`, `app/models/membership.rb`, `app/models/protocol_definition.rb`, `app/policies/protocol_policy.rb`, `app/commands/invite_member.rb`, `app/commands/protocols/save_draft.rb`, `app/commands/protocols/publish.rb`, `app/commands/protocols/activate.rb`, `app/commands/protocols/retire.rb`, `db/city_schema.rb`, `lib/tasks/city.rake`, `spec/services/city_schema_spec.rb`, `spec/commands/protocols_lifecycle_spec.rb`, `spec/commands/protocols_publish_gate_spec.rb`, `spec/controllers/publications_controller_spec.rb`, `README.md`

**docs**
- Create: `adr/0016.md`
- Modify: `adr/0012.md`, `adr/0009.md`, `adr/README.md`

---

### Task 1: O ator mantenedor e o digest do conteúdo

**Files:**
- Create: `app/services/maintenance/maintainer_actor.rb`, `app/services/protocols/content_digest.rb`
- Modify: `app/models/user.rb`
- Test: `spec/services/maintenance/maintainer_actor_spec.rb`, `spec/services/protocols/content_digest_spec.rb`

**Interfaces:**
- Produces:
  - `Maintenance::MaintainerActor.new(maintainer)` → `#maintainer`, `#id` (id do mantenedor), `#actor_kind` → `"maintainer"`, `#has_role?(_role)` → `true`
  - `User#actor_kind` → `"user"`
  - `Protocols::ContentDigest.call(definition)` → String hex SHA-256 do JSON canônico (chaves ordenadas em todos os níveis, ordem de array preservada)

- [ ] **Step 1: Escrever os specs que falham**

`spec/services/maintenance/maintainer_actor_spec.rb`:

```ruby
require "rails_helper"

# D6 da API de manutenção: o mantenedor pula a AUTORIZAÇÃO (papéis) e só ela.
# É o ator que diz "sim" à policy; as regras de domínio que olham o TIPO de
# ator (assinar, conceder papel privilegiado) continuam recusando.
RSpec.describe Maintenance::MaintainerActor do
  let(:maintainer) do
    Maintainer.create!(email_address: "ma-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end
  let(:actor) { described_class.new(maintainer) }

  it "answers yes to every role a policy asks" do
    %i[protocol_author protocol_publisher protocol_reviewer municipal_admin viewer].each do |role|
      expect(actor.has_role?(role)).to be(true)
    end
  end

  it "identifies itself as a maintainer, by the maintainer's id" do
    expect(actor.id).to eq(maintainer.id)
    expect(actor.actor_kind).to eq("maintainer")
  end

  it "passes ProtocolPolicy without any membership in the city" do
    policy = ProtocolPolicy.new(actor, ProtocolDefinition.new)

    expect(policy.author?).to be(true)
    expect(policy.publish?).to be(true)
    expect(policy.activate?).to be(true)
  end

  it "has a user counterpart that says it is a user" do
    expect(User.new.actor_kind).to eq("user")
  end
end
```

`spec/services/protocols/content_digest_spec.rb`:

```ruby
require "rails_helper"

# A assinatura vale para o conteúdo EXATO (spec S5). O digest não pode depender
# da ordem em que as chaves foram escritas — o jsonb do Postgres reordena —, e
# tem de mudar com qualquer mudança de conteúdo, inclusive a ordem de uma lista.
RSpec.describe Protocols::ContentDigest do
  let(:definition) do
    { "name" => "dengue", "version" => 1,
      "steps" => [ { "id" => "s1", "prompt" => "?" }, { "id" => "s2", "prompt" => "!" } ] }
  end

  it "is a hex SHA-256" do
    expect(described_class.call(definition)).to match(/\A\h{64}\z/)
  end

  it "does not depend on key order, at any depth" do
    reordered = { "steps" => [ { "prompt" => "?", "id" => "s1" }, { "prompt" => "!", "id" => "s2" } ],
                  "version" => 1, "name" => "dengue" }

    expect(described_class.call(reordered)).to eq(described_class.call(definition))
  end

  it "treats symbol and string keys alike" do
    expect(described_class.call(definition.deep_symbolize_keys)).to eq(described_class.call(definition))
  end

  it "changes when any value changes" do
    changed = definition.deep_dup.tap { |d| d["steps"][1]["prompt"] = "?!" }

    expect(described_class.call(changed)).not_to eq(described_class.call(definition))
  end

  it "changes when the order of a list changes" do
    swapped = definition.merge("steps" => definition["steps"].reverse)

    expect(described_class.call(swapped)).not_to eq(described_class.call(definition))
  end

  it "answers the same digest for what the database stores and gives back" do
    Current.city = TEST_CITY_A
    stored = ProtocolDefinition.create!(name: "dengue", version: 1, status: "draft",
                                        definition: protocol_definition_hash)

    expect(described_class.call(stored.reload.definition)).to eq(described_class.call(protocol_definition_hash))
  ensure
    Current.reset
  end
end
```

O último exemplo usa `protocol_definition_hash`, o helper da Task 2 (`spec/support/protocol_signatures.rb`). **Nesta task**, defina `let(:protocol_definition_hash)` no próprio spec copiando o `definition_hash` de `spec/commands/protocols_lifecycle_spec.rb`; a Task 2 troca pelo helper.

- [ ] **Step 2: Rodar e confirmar que falham**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/maintenance/maintainer_actor_spec.rb spec/services/protocols/content_digest_spec.rb`
Expected: FAIL — `Maintenance::MaintainerActor` e `Protocols::ContentDigest` não existem.

- [ ] **Step 3: Implementar**

`app/services/maintenance/maintainer_actor.rb`:

```ruby
# O mantenedor da API de manutenção como ator de um command de cidade.
#
# D6: o superusuário pula a AUTORIZAÇÃO — papéis e memberships — e nada mais.
# Por isso a única coisa que este objeto responde de diferente de um User é
# `has_role?`: toda policy (ApplicationPolicy#role?) pergunta ao ator, e o
# mantenedor diz sim.
#
# As regras de domínio que existem para PRENDER o mantenedor olham
# `actor_kind`, nunca o papel: ele não assina protocolo, não concede nem
# convida quem assina ou quem concede (spec de assinaturas §7). Perguntar o
# papel a este objeto é sempre "sim" — por isso a pergunta certa é o tipo.
#
# O mantenedor NÃO é um User da cidade, e não vira um: um User de sistema por
# cidade faria mantenedores diferentes parecerem a mesma pessoa na trilha.
module Maintenance
  class MaintainerActor
    attr_reader :maintainer

    def initialize(maintainer)
      @maintainer = maintainer
    end

    def id = maintainer.id
    def actor_kind = "maintainer"
    def has_role?(_role) = true
  end
end
```

Em `app/models/user.rb`, ao lado de `has_role?`:

```ruby
  # Par de Maintenance::MaintainerActor#actor_kind: o evento de domínio e as
  # tabelas de protocolo dizem de qual tabela vem o id do ator.
  def actor_kind = "user"
```

`app/services/protocols/content_digest.rb`:

```ruby
require "digest"

# Digest do conteúdo de um protocolo (spec de assinaturas §5).
#
# A assinatura vale para o conteúdo EXATO: editar depois de assinar faz o
# digest mudar, e a assinatura antiga deixa de contar sem que nenhuma linha
# seja alterada. Por isso o digest precisa ser estável para o mesmo conteúdo —
# chaves ordenadas em todos os níveis, porque o jsonb do Postgres não guarda a
# ordem em que foram escritas — e sensível a qualquer mudança, inclusive a
# ordem de uma lista (a ordem dos passos é conteúdo).
module Protocols
  module ContentDigest
    module_function

    def call(definition)
      Digest::SHA256.hexdigest(JSON.generate(canonical(definition)))
    end

    def canonical(value)
      case value
      when Hash then value.to_h { |key, inner| [ key.to_s, canonical(inner) ] }.sort.to_h
      when Array then value.map { |inner| canonical(inner) }
      else value
      end
    end
  end
end
```

- [ ] **Step 4: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/services/maintenance/maintainer_actor.rb app/services/protocols/content_digest.rb \
  app/models/user.rb spec/services/maintenance/maintainer_actor_spec.rb spec/services/protocols/content_digest_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: add the maintainer actor and the protocol content digest

The maintainer acts in city commands as an actor that answers yes to
every role a policy asks, and says it is a maintainer so the rules that
exist to hold it back can ask its kind instead of its role. The content
digest pins a signature to the exact content, whatever the key order.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 2: O papel de revisor e as três tabelas que só aceitam acréscimo

**Files:**
- Create: `db/city_triggers.sql`, `db/city_migrate/20260918000001_create_protocol_signatures.rb`, `app/models/protocol_contribution.rb`, `app/models/protocol_signature.rb`, `app/models/protocol_activation.rb`, `spec/support/protocol_signatures.rb`
- Modify: `app/models/membership.rb`, `app/models/protocol_definition.rb`, `db/city_schema.rb`, `lib/tasks/city.rake`, `spec/services/city_schema_spec.rb`, `spec/services/protocols/content_digest_spec.rb`, `README.md`
- Test: `spec/models/protocol_append_only_spec.rb`, `spec/services/city_schema_spec.rb`

**Interfaces:**
- Produces:
  - `Membership::ROLES` com `protocol_reviewer`; `Membership::PRIVILEGED_ROLES = %w[municipal_admin protocol_reviewer]`
  - Tabelas e modelos:
    - `ProtocolContribution` (`protocol_definition_id`, `actor_id`, `actor_kind` ∈ `user`/`maintainer`, `content_digest`, `created_at`)
    - `ProtocolSignature` (`protocol_definition_id`, `purpose` ∈ `publication`/`activation`, `signer_user_id` → `users`, `content_digest`, `created_at`), `belongs_to :signer, class_name: "User"`
    - `ProtocolActivation` (`protocol_definition_id`, `kind` ∈ `signed`/`emergency_revert`, `actor_id`, `actor_kind`, `reason`, `created_at`)
  - `ProtocolDefinition has_many :contributions, :signatures, :activations` e `#content_digest`
  - Helpers de spec (em `spec/support/protocol_signatures.rb`, incluídos em todos os specs): `protocol_definition_hash(name: "dengue", version: 1)`, `make_reviewer!(email: nil)` → `User` com `protocol_reviewer`, `sign!(protocol, purpose:, by:)` → cria `ProtocolSignature` com o digest atual

- [ ] **Step 1: Escrever os specs que falham**

`spec/models/protocol_append_only_spec.rb`:

```ruby
require "rails_helper"

# As três tabelas são a PROVA de quem editou, assinou e ativou (spec §5): um
# registro que se pode alterar não prova nada. O trigger fecha o caminho que o
# modelo não fecha — um bug, um update_all, um psql aberto com o papel da app.
RSpec.describe "Protocol append-only tables" do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:protocol) do
    ProtocolDefinition.create!(name: "dengue", version: 1, status: "draft", definition: protocol_definition_hash)
  end
  let(:reviewer) { make_reviewer! }

  def rows
    [
      ProtocolContribution.create!(protocol_definition: protocol, actor_id: reviewer.id, actor_kind: "user",
                                   content_digest: protocol.content_digest),
      ProtocolSignature.create!(protocol_definition: protocol, purpose: "publication", signer: reviewer,
                                content_digest: protocol.content_digest),
      ProtocolActivation.create!(protocol_definition: protocol, kind: "signed", actor_id: reviewer.id,
                                 actor_kind: "user")
    ]
  end

  it "refuses UPDATE on every table, even through update_all" do
    rows.each do |row|
      expect { row.class.where(id: row.id).update_all(created_at: 1.day.ago) }
        .to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    end
  end

  it "refuses DELETE on every table, even through delete_all" do
    rows.each do |row|
      expect { row.class.where(id: row.id).delete_all }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    end
  end

  it "treats a persisted row as read-only in the model too" do
    rows.each { |row| expect(row).to be_readonly }
  end

  it "accepts protocol_reviewer as a membership role, in the model and in the database CHECK" do
    user = User.create!(email_address: "rv-#{SecureRandom.hex(3)}@example.org", password: "secret123")

    expect { Membership.create!(user: user, role: "protocol_reviewer", granted_at: Time.current) }.not_to raise_error
  end

  it "refuses an unknown purpose, kind or actor kind in the database" do
    expect {
      ProtocolSignature.new(protocol_definition: protocol, purpose: "x", signer: reviewer,
                            content_digest: "d").save!(validate: false)
    }.to raise_error(ActiveRecord::StatementInvalid)
  end
end
```

Cada exemplo que levanta dentro da transação de fixture a deixa abortada. Se isso quebrar o exemplo seguinte **dentro do mesmo exemplo** (o `each` faz várias tentativas), envolva cada tentativa em `ActiveRecord::Base.transaction(requires_new: true) { ... }` — um savepoint — e diga no relatório.

Em `spec/services/city_schema_spec.rb`, acrescente `triggers:` ao `schema_fingerprint`:

```ruby
        triggers: conn.exec(<<~SQL).values
          SELECT c.relname, t.tgname, pg_get_triggerdef(t.oid)
          FROM pg_trigger t
          JOIN pg_class c ON c.oid = t.tgrelid
          JOIN pg_namespace n ON n.oid = c.relnamespace
          WHERE n.nspname = 'public' AND NOT t.tgisinternal AND c.relname NOT IN #{ignored}
          ORDER BY 1, 2
        SQL
```

e um exemplo novo que afirma que `rota_saude_test_city_b` (carregado do dump) tem os três triggers `*_append_only`.

- [ ] **Step 2: Rodar e confirmar que falham**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/protocol_append_only_spec.rb spec/services/city_schema_spec.rb`
Expected: FAIL — tabelas e modelos não existem.

- [ ] **Step 3: Escrever o SQL dos triggers**

`db/city_triggers.sql`:

```sql
-- Triggers do banco de CIDADE. Fonte ÚNICA: executado pela migração que os
-- criou E por lib/tasks/city.rake (load_city_schema), depois de carregar
-- db/city_schema.rb — o dump em Ruby não representa trigger, e sem isto os
-- bancos carregados do dump (testes, dev) ficariam sem a proteção.
-- Idempotente: pode rodar quantas vezes for preciso.

CREATE OR REPLACE FUNCTION rota_append_only() RETURNS trigger AS $fn$
BEGIN
  RAISE EXCEPTION '% is append-only: % refused', TG_TABLE_NAME, TG_OP;
END;
$fn$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS protocol_contributions_append_only ON protocol_contributions;
CREATE TRIGGER protocol_contributions_append_only
  BEFORE UPDATE OR DELETE ON protocol_contributions
  FOR EACH ROW EXECUTE FUNCTION rota_append_only();

DROP TRIGGER IF EXISTS protocol_signatures_append_only ON protocol_signatures;
CREATE TRIGGER protocol_signatures_append_only
  BEFORE UPDATE OR DELETE ON protocol_signatures
  FOR EACH ROW EXECUTE FUNCTION rota_append_only();

DROP TRIGGER IF EXISTS protocol_activations_append_only ON protocol_activations;
CREATE TRIGGER protocol_activations_append_only
  BEFORE UPDATE OR DELETE ON protocol_activations
  FOR EACH ROW EXECUTE FUNCTION rota_append_only();
```

- [ ] **Step 4: Escrever a migração**

`db/city_migrate/20260918000001_create_protocol_signatures.rb`:

```ruby
# Assinaturas de protocolo (spec 2026-09-18-protocol-signatures-design §3, §5).
#
# Aditiva: um papel novo no CHECK de memberships e três tabelas que só aceitam
# acréscimo. Os triggers vêm de db/city_triggers.sql, a mesma fonte que
# load_city_schema executa depois de carregar o dump.
class CreateProtocolSignatures < ActiveRecord::Migration[8.1]
  ROLES_BEFORE = %w[municipal_admin protocol_author protocol_publisher viewer].freeze
  ROLES_AFTER = %w[municipal_admin protocol_author protocol_publisher protocol_reviewer viewer].freeze

  def up
    replace_roles_check(ROLES_AFTER)

    create_table :protocol_contributions, id: :uuid do |t|
      t.references :protocol_definition, type: :uuid, null: false, foreign_key: true, index: true
      t.uuid :actor_id, null: false
      t.string :actor_kind, null: false
      t.string :content_digest, null: false
      t.datetime :created_at, null: false
    end
    add_check_constraint :protocol_contributions, "actor_kind IN ('user', 'maintainer')",
                         name: "ck_protocol_contributions_actor_kind"

    create_table :protocol_signatures, id: :uuid do |t|
      t.references :protocol_definition, type: :uuid, null: false, foreign_key: true, index: true
      t.string :purpose, null: false
      t.references :signer_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: true
      t.string :content_digest, null: false
      t.datetime :created_at, null: false
    end
    add_check_constraint :protocol_signatures, "purpose IN ('publication', 'activation')",
                         name: "ck_protocol_signatures_purpose"

    create_table :protocol_activations, id: :uuid do |t|
      t.references :protocol_definition, type: :uuid, null: false, foreign_key: true, index: true
      t.string :kind, null: false
      t.uuid :actor_id, null: false
      t.string :actor_kind, null: false
      t.text :reason
      t.datetime :created_at, null: false
    end
    add_check_constraint :protocol_activations, "kind IN ('signed', 'emergency_revert')",
                         name: "ck_protocol_activations_kind"
    add_check_constraint :protocol_activations, "actor_kind IN ('user', 'maintainer')",
                         name: "ck_protocol_activations_actor_kind"
    add_check_constraint :protocol_activations, "kind = 'signed' OR length(btrim(reason)) > 0",
                         name: "ck_protocol_activations_revert_reason"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    execute "DROP FUNCTION IF EXISTS rota_append_only() CASCADE"
    drop_table :protocol_activations
    drop_table :protocol_signatures
    drop_table :protocol_contributions
    replace_roles_check(ROLES_BEFORE)
  end

  private

  def replace_roles_check(roles)
    remove_check_constraint :memberships, name: "ck_memberships_role"
    add_check_constraint :memberships, "role::text = ANY (ARRAY[#{roles.map { |r| "'#{r}'" }.join(', ')}]::text[])",
                         name: "ck_memberships_role"
  end
end
```

**Confira** a expressão do CHECK que o Postgres devolve (`pg_get_constraintdef`) contra a que está hoje em `db/city_schema.rb`; o spec de paridade compara as duas, então a forma escrita no dump tem de ser a que o banco migrado produz.

- [ ] **Step 5: Atualizar o dump e a carga do dump**

1. `db/city_schema.rb`: versão `2026_09_18_000001`, as três tabelas com índices, FKs e CHECKs, e o CHECK novo de `memberships`, **na forma exata do dump do Rails**. Gere-a migrando um banco scratch (como o spec de paridade faz com `ScratchDatabases` e `CitySchema.migrate!`) e rodando o dumper nele; não escreva à mão o que o dumper escreve.
2. `lib/tasks/city.rake`, na lambda `load_city_schema`, logo depois de `ActiveRecord::Tasks::DatabaseTasks.load_schema(...)`, ainda dentro de `with_temporary_connection`:

```ruby
      # O dump em Ruby não representa trigger (db/city_triggers.sql, cabeçalho).
      ActiveRecord::Base.connection.execute(File.read(Rails.root.join("db/city_triggers.sql")))
```

   **Confira** que, dentro de `with_temporary_connection(db_config)`, `ActiveRecord::Base.connection` é a conexão do banco alvo, e não outra. Se não for, use a conexão que o bloco entrega.
3. `README.md`, na seção que diz para atualizar `db/city_schema.rb` à mão: acrescente que trigger de cidade mora em `db/city_triggers.sql`, executado pela migração e por `load_city_schema`, e que o spec de paridade compara triggers também.

- [ ] **Step 6: Implementar modelos e o papel**

`app/models/membership.rb`:

```ruby
  ROLES = %w[municipal_admin protocol_author protocol_publisher protocol_reviewer viewer].freeze

  # Papéis que o mantenedor da API de manutenção nunca concede nem convida
  # (spec de assinaturas §7): quem aprova protocolo e quem concede aprovação.
  # Com eles, o superusuário criaria as duas contas de revisor e assinaria.
  PRIVILEGED_ROLES = %w[municipal_admin protocol_reviewer].freeze
```

(atualize também o comentário do topo: `ROLES` espelha `ck_memberships_role`).

`app/models/protocol_contribution.rb`:

```ruby
# Um salvamento de rascunho: quem editou a versão e qual conteúdo salvou
# (spec de assinaturas §5). Quem tem uma linha aqui nunca assina a versão.
# Só aceita acréscimo — o trigger rota_append_only recusa UPDATE e DELETE.
class ProtocolContribution < ApplicationRecord
  ACTOR_KINDS = %w[user maintainer].freeze

  belongs_to :protocol_definition

  validates :actor_id, :content_digest, presence: true
  validates :actor_kind, inclusion: { in: ACTOR_KINDS }

  def readonly? = persisted?
end
```

`app/models/protocol_signature.rb`:

```ruby
# Uma assinatura de revisor sobre um conteúdo exato, para uma finalidade
# (spec de assinaturas §5). Se conta ou não para um ato é pergunta de
# Protocols::Signatures, feita no momento do ato. Só aceita acréscimo.
class ProtocolSignature < ApplicationRecord
  PURPOSES = %w[publication activation].freeze

  belongs_to :protocol_definition
  belongs_to :signer, class_name: "User", foreign_key: :signer_user_id, inverse_of: false

  validates :content_digest, presence: true
  validates :purpose, inclusion: { in: PURPOSES }

  def readonly? = persisted?
end
```

`app/models/protocol_activation.rb`:

```ruby
# Um ato de ativação (spec de assinaturas §5, §6). A última linha de uma versão
# marca até quando as assinaturas de ativação dela já foram consumidas, e a
# sequência por protocolo é o que a reversão de emergência lê. Só aceita
# acréscimo.
class ProtocolActivation < ApplicationRecord
  KINDS = %w[signed emergency_revert].freeze

  belongs_to :protocol_definition

  validates :actor_id, presence: true
  validates :kind, inclusion: { in: KINDS }
  validates :actor_kind, inclusion: { in: ProtocolContribution::ACTOR_KINDS }
  validates :reason, presence: true, if: -> { kind == "emergency_revert" }

  def readonly? = persisted?
end
```

Em `ProtocolDefinition`:

```ruby
  has_many :contributions, class_name: "ProtocolContribution", dependent: :restrict_with_error
  has_many :signatures, class_name: "ProtocolSignature", dependent: :restrict_with_error
  has_many :activations, class_name: "ProtocolActivation", dependent: :restrict_with_error

  def content_digest = Protocols::ContentDigest.call(definition)
```

- [ ] **Step 7: Helpers de spec**

`spec/support/protocol_signatures.rb`:

```ruby
# Arranjo de protocolo e assinaturas para specs de domínio. Os exemplos rodam
# na conexão de TEST_CITY_A (city_test_databases.rb), então o que se cria aqui
# é visível pelos commands.
module ProtocolSignaturesSpecHelpers
  # O mesmo protocolo de spec/commands/protocols_lifecycle_spec.rb, que passa
  # no portão (Protocols::Gate).
  def protocol_definition_hash(name: "dengue", version: 1)
    {
      "name" => name, "version" => version, "start_step_id" => "s1",
      "steps" => [
        { "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
          "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 1, "false" => 0 } }
      ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 9 } }
    }
  end

  def make_reviewer!(email: "rv-#{SecureRandom.hex(4)}@example.org")
    user = User.create!(email_address: email, password: "secret123")
    Membership.create!(user: user, role: "protocol_reviewer", granted_at: Time.current)
    user
  end

  def sign!(protocol, purpose:, by:)
    ProtocolSignature.create!(protocol_definition: protocol, purpose: purpose, signer: by,
                              content_digest: protocol.reload.content_digest)
  end
end

RSpec.configure { |config| config.include ProtocolSignaturesSpecHelpers }
```

**Confira** que `spec/support/**/*.rb` é carregado pelo `rails_helper` (os outros arquivos de `spec/support` já são). Troque o `let(:protocol_definition_hash)` provisório da Task 1 pelo helper.

- [ ] **Step 8: Recarregar os bancos de teste e migrar as cidades de dev**

```bash
docker compose exec -T -e POSTGRES_PASSWORD=postgres -e RAILS_ENV=test api bin/rails city:test_databases
docker compose exec -T api bin/rails city:migrate:all
```

A migração é aditiva. As cidades de dev ativas (`curitiba`, `maringa`) precisam dela: cidade com schema atrasado responde `503 city_schema_behind`. Confira depois com `docker compose exec -T api bin/rails runner 'puts City.where(status: "active").map { |c| [c.slug, c.schema_version] }.inspect'` e registre a saída no relatório.

- [ ] **Step 9: Rodar, suíte completa e commit**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/protocol_append_only_spec.rb spec/services/city_schema_spec.rb spec/services/protocols`

Depois a suíte completa (worker parado antes, religado depois).

```bash
cd apps/api
/opt/homebrew/bin/git add db/city_triggers.sql db/city_migrate/20260918000001_create_protocol_signatures.rb db/city_schema.rb \
  lib/tasks/city.rake app/models/membership.rb app/models/protocol_definition.rb app/models/protocol_contribution.rb \
  app/models/protocol_signature.rb app/models/protocol_activation.rb spec/support/protocol_signatures.rb \
  spec/models/protocol_append_only_spec.rb spec/services/city_schema_spec.rb spec/services/protocols/content_digest_spec.rb \
  README.md
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: add the protocol reviewer role and the append-only signature tables

Contributions, signatures and activations are the proof of who edited,
signed and activated a protocol version, so the database refuses to
update or delete them. The dump cannot carry a trigger, so the trigger
SQL has one source that both the migration and the dump loader run, and
the schema parity spec now compares triggers too.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 3: Conceder o papel e fechar a brecha do convite

**Files:**
- Create: `app/commands/grant_role.rb`
- Modify: `app/commands/invite_member.rb`, `app/policies/protocol_policy.rb`
- Test: `spec/commands/grant_role_spec.rb`, `spec/commands/invite_member_privileged_roles_spec.rb`

**Interfaces:**
- Consumes: `Maintenance::MaintainerActor`, `User#actor_kind` (Task 1); `Membership::PRIVILEGED_ROLES` (Task 2).
- Produces:
  - `GrantRole.call(user_id:, role:, by:)` → `Result.ok(membership:)` | `Result.fail(:forbidden_for_maintainer | :forbidden | :invalid_role | :user_not_found | :user_inactive | :already_granted)`; evento `membership.granted` (`user_id`, `role`, `by`, `actor_kind`)
  - `InviteMember.call(...)` recusa `:forbidden_for_maintainer` quando `invited_by` é mantenedor e o papel é privilegiado
  - `ProtocolPolicy#review?` → `role?(:protocol_reviewer)`

- [ ] **Step 1: Escrever os specs que falham**

`spec/commands/grant_role_spec.rb`:

```ruby
require "rails_helper"

# Spec de assinaturas §3 e §7: só o municipal_admin concede papel, e o
# mantenedor nunca concede quem aprova protocolo nem quem concede aprovação.
RSpec.describe GrantRole do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:admin) do
    User.create!(email_address: "ad-#{SecureRandom.hex(3)}@example.org", password: "secret123").tap do |u|
      Membership.create!(user: u, role: "municipal_admin", granted_at: Time.current)
    end
  end
  let(:publisher) do
    User.create!(email_address: "pb-#{SecureRandom.hex(3)}@example.org", password: "secret123").tap do |u|
      Membership.create!(user: u, role: "protocol_publisher", granted_at: Time.current)
    end
  end
  let(:maintainer_actor) do
    Maintenance::MaintainerActor.new(
      Maintainer.create!(email_address: "gm-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                         otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
    )
  end

  it "lets a municipal admin make a publisher a reviewer too, and records who granted it" do
    result = described_class.call(user_id: publisher.id, role: "protocol_reviewer", by: admin)

    expect(result.ok?).to be(true)
    expect(publisher.reload.has_role?(:protocol_reviewer)).to be(true)
    expect(publisher.has_role?(:protocol_publisher)).to be(true)
    expect(result.payload[:membership].granted_by_id).to eq(admin.id)
    event = DomainEvent.where(name: "membership.granted").order(:occurred_at).last
    expect(event.payload).to include("user_id" => publisher.id, "role" => "protocol_reviewer",
                                     "by" => admin.id, "actor_kind" => "user")
  end

  it "refuses anyone who is not a municipal admin" do
    other = publisher

    expect(described_class.call(user_id: other.id, role: "protocol_reviewer", by: other).reason).to eq(:forbidden)
  end

  it "refuses the maintainer for every privileged role, even though it passes every role question" do
    Membership::PRIVILEGED_ROLES.each do |role|
      result = described_class.call(user_id: publisher.id, role: role, by: maintainer_actor)

      expect(result.reason).to eq(:forbidden_for_maintainer)
    end
    expect(publisher.reload.has_role?(:protocol_reviewer)).to be(false)
  end

  it "lets the maintainer grant a role that is not privileged, without a granted_by user" do
    result = described_class.call(user_id: publisher.id, role: "viewer", by: maintainer_actor)

    expect(result.ok?).to be(true)
    expect(result.payload[:membership].granted_by_id).to be_nil
    expect(DomainEvent.where(name: "membership.granted").order(:occurred_at).last.payload)
      .to include("by" => maintainer_actor.id, "actor_kind" => "maintainer")
  end

  it "refuses an unknown role, a missing user, a deactivated user and a role already held" do
    expect(described_class.call(user_id: publisher.id, role: "god", by: admin).reason).to eq(:invalid_role)
    expect(described_class.call(user_id: SecureRandom.uuid, role: "viewer", by: admin).reason).to eq(:user_not_found)
    expect(described_class.call(user_id: publisher.id, role: "protocol_publisher", by: admin).reason)
      .to eq(:already_granted)

    publisher.update!(deactivated_at: Time.current)
    expect(described_class.call(user_id: publisher.id, role: "viewer", by: admin).reason).to eq(:user_inactive)
  end
end
```

`spec/commands/invite_member_privileged_roles_spec.rb`:

```ruby
require "rails_helper"

# Spec de assinaturas §7: sem esta recusa, o mantenedor convidaria duas contas
# de revisor e assinaria por elas — e a regra inteira cairia.
RSpec.describe "InviteMember and privileged roles" do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:maintainer_actor) do
    Maintenance::MaintainerActor.new(
      Maintainer.create!(email_address: "im-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                         otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
    )
  end

  it "refuses a maintainer inviting a reviewer or a municipal admin, and creates no invitation" do
    Membership::PRIVILEGED_ROLES.each do |role|
      result = InviteMember.call(email: "x-#{role}@example.org", role: role, invited_by: maintainer_actor)

      expect(result.reason).to eq(:forbidden_for_maintainer)
    end
    expect(Invitation.where(email: Membership::PRIVILEGED_ROLES.map { |r| "x-#{r}@example.org" })).to be_empty
  end

  it "still lets a municipal user invite a reviewer" do
    admin = User.create!(email_address: "ad-#{SecureRandom.hex(3)}@example.org", password: "secret123")
    Membership.create!(user: admin, role: "municipal_admin", granted_at: Time.current)

    expect(InviteMember.call(email: "novo@example.org", role: "protocol_reviewer", invited_by: admin).ok?).to be(true)
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falham**

- [ ] **Step 3: Implementar**

`app/policies/protocol_policy.rb`, acrescente:

```ruby
  # Quem assina uma versão (spec de assinaturas S1). Papel próprio, separado de
  # quem publica. NOTA: o mantenedor responde "sim" a esta pergunta (D6) — é
  # Protocols::Sign que o recusa, pelo tipo de ator.
  def review?
    role?(:protocol_reviewer)
  end
```

`app/commands/grant_role.rb`:

```ruby
# Concede um papel a um usuário que já existe na cidade da conexão corrente
# (spec de assinaturas §3). Até aqui um papel só nascia pelo convite; o
# revisor de protocolo é, em geral, alguém que já é publisher.
#
# Duas recusas, nesta ordem:
#   1. o mantenedor nunca concede um papel de Membership::PRIVILEGED_ROLES —
#      pelo TIPO de ator, porque ele responde "sim" a toda pergunta de papel;
#   2. quem concede precisa ser municipal_admin (MembershipPolicy#manage?).
#
# Append-only como o resto de memberships: conceder cria uma linha; revogar é
# RevokeMembership (fim de vigência).
class GrantRole
  def self.call(user_id:, role:, by:)
    return Result.fail(:city_missing) if Current.city.nil?
    return Result.fail(:invalid_role) unless Membership::ROLES.include?(role.to_s)

    if by.actor_kind == "maintainer" && Membership::PRIVILEGED_ROLES.include?(role.to_s)
      return Result.fail(:forbidden_for_maintainer,
                         message: "o mantenedor não concede #{role}: quem aprova protocolo é escolhido pela cidade")
    end
    return Result.fail(:forbidden) unless MembershipPolicy.new(by, nil).manage?

    user = User.find_by(id: user_id)
    return Result.fail(:user_not_found) if user.nil?
    return Result.fail(:user_inactive) unless user.active?
    return Result.fail(:already_granted) if user.has_role?(role)

    membership = nil
    ApplicationRecord.transaction do
      membership = Membership.create!(user: user, role: role.to_s, granted_at: Time.current,
                                      granted_by_id: (by.id if by.actor_kind == "user"))
      DomainEvents.publish("membership.granted", user_id: user.id, role: role.to_s,
                                                 by: by.id, actor_kind: by.actor_kind)
    end

    Result.ok(membership: membership)
  rescue ActiveRecord::RecordInvalid => e
    Result.fail(:invalid, message: e.record.errors.full_messages.join(", "))
  end
end
```

Em `app/commands/invite_member.rb`, no começo de `#call`:

```ruby
    # Spec de assinaturas §7: o mantenedor não convida quem aprova protocolo
    # nem quem concede aprovação. Pelo TIPO de ator — ele passa em toda
    # pergunta de papel.
    if @invited_by.respond_to?(:actor_kind) && @invited_by.actor_kind == "maintainer" &&
       Membership::PRIVILEGED_ROLES.include?(@role.to_s)
      return Result.fail(:forbidden_for_maintainer,
                         message: "o mantenedor não convida #{@role}: quem aprova protocolo é escolhido pela cidade")
    end
```

**Não** trate aqui o `invited_by` de um convite não privilegiado feito por mantenedor (a FK aponta para `users`): isso é da fatia de membros do Plano 5.

- [ ] **Step 4: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/commands/grant_role.rb app/commands/invite_member.rb app/policies/protocol_policy.rb \
  spec/commands/grant_role_spec.rb spec/commands/invite_member_privileged_roles_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: let a municipal admin grant a role to an existing user

A reviewer is usually someone who is already a publisher, and a role
could only be born from an invitation. The maintainer never grants nor
invites a reviewer or a municipal admin: with either, it could create
two reviewer accounts and sign for them, so both commands ask the kind
of actor, not its role.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 4: Autoria, envio para revisão e assinatura

**Files:**
- Create: `app/services/protocols/signatures.rb`, `app/commands/protocols/submit_for_review.rb`, `app/commands/protocols/sign.rb`
- Modify: `app/commands/protocols/save_draft.rb`
- Test: `spec/services/protocols/signatures_spec.rb`, `spec/commands/protocols_authoring_spec.rb`

**Interfaces:**
- Consumes: modelos e helpers da Task 2; `ProtocolPolicy#review?` (Task 3).
- Produces:
  - `Protocols::Signatures::REQUIRED = 2`
  - `Protocols::Signatures.valid_signer_ids(protocol, purpose:)` → `[uuid]` distintos que satisfazem as cinco condições da spec §5 (mais o usuário ativo, Decisão 4)
  - `Protocols::Signatures.missing(protocol, purpose:)` → Integer ≥ 0
  - `Protocols::Signatures.eligible_reviewer_count(protocol)` → Integer: revisores ativos da cidade que não contribuíram para a versão
  - `Protocols::Signatures.shortfall_message(protocol, purpose:)` → String para o `Result.fail`
  - `Protocols::SaveDraft.call(definition:, by:, correlation_id: nil)` — edita `draft` e `in_review` (em `in_review` volta a `draft`); registra `ProtocolContribution`; evento `protocol.draft_saved`
  - `Protocols::SubmitForReview.call(name:, version:, by:, correlation_id: nil)` → `draft → in_review`, portão; evento `protocol.submitted_for_review`
  - `Protocols::Sign.call(name:, version:, purpose:, by:)` → `Result.ok(signature:)` | `Result.fail(:maintainer_cannot_sign | :forbidden | :invalid_purpose | :not_found | :invalid_state | :contributor_cannot_sign | :already_signed)`; evento `protocol.signed`

- [ ] **Step 1: Escrever os specs que falham**

`spec/services/protocols/signatures_spec.rb`:

```ruby
require "rails_helper"

# A ÚNICA resposta para "esta versão tem as assinaturas?" (spec §5). Cada
# condição tem o seu exemplo: uma regra de aprovação que só é testada no
# caminho feliz aprova o que não devia.
RSpec.describe Protocols::Signatures do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:protocol) do
    ProtocolDefinition.create!(name: "dengue", version: 1, status: "in_review", definition: protocol_definition_hash)
  end
  let(:ana) { make_reviewer! }
  let(:bia) { make_reviewer! }

  it "counts two distinct reviewers on the current content" do
    sign!(protocol, purpose: "publication", by: ana)
    sign!(protocol, purpose: "publication", by: bia)

    expect(described_class.valid_signer_ids(protocol, purpose: "publication")).to contain_exactly(ana.id, bia.id)
    expect(described_class.missing(protocol, purpose: "publication")).to eq(0)
  end

  it "counts a reviewer once however many times they signed" do
    2.times { sign!(protocol, purpose: "publication", by: ana) }

    expect(described_class.missing(protocol, purpose: "publication")).to eq(1)
  end

  it "does not count a signature on older content" do
    sign!(protocol, purpose: "publication", by: ana)
    protocol.update!(definition: protocol_definition_hash.merge("start_step_id" => "s1", "note" => "x"))

    expect(described_class.valid_signer_ids(protocol, purpose: "publication")).to be_empty
  end

  it "does not count a signature for the other purpose" do
    sign!(protocol, purpose: "activation", by: ana)

    expect(described_class.valid_signer_ids(protocol, purpose: "publication")).to be_empty
  end

  it "does not count a reviewer whose role was revoked, nor a deactivated user" do
    sign!(protocol, purpose: "publication", by: ana)
    sign!(protocol, purpose: "publication", by: bia)
    ana.memberships.active.find_by!(role: "protocol_reviewer").update!(revoked_at: Time.current)
    bia.update!(deactivated_at: Time.current)

    expect(described_class.valid_signer_ids(protocol, purpose: "publication")).to be_empty
  end

  it "does not count a reviewer who contributed to the version, even after signing" do
    sign!(protocol, purpose: "publication", by: ana)
    ProtocolContribution.create!(protocol_definition: protocol, actor_id: ana.id, actor_kind: "user",
                                 content_digest: protocol.content_digest)

    expect(described_class.valid_signer_ids(protocol, purpose: "publication")).to be_empty
  end

  it "counts, for activation, only signatures after the version's last activation" do
    protocol.update!(status: "published")
    sign!(protocol, purpose: "activation", by: ana)
    ProtocolActivation.create!(protocol_definition: protocol, kind: "signed", actor_id: bia.id, actor_kind: "user")
    sign!(protocol, purpose: "activation", by: bia)

    expect(described_class.valid_signer_ids(protocol, purpose: "activation")).to contain_exactly(bia.id)
  end

  it "counts eligible reviewers as active reviewers who did not contribute" do
    ana
    bia
    ProtocolContribution.create!(protocol_definition: protocol, actor_id: ana.id, actor_kind: "user",
                                 content_digest: protocol.content_digest)

    expect(described_class.eligible_reviewer_count(protocol)).to eq(1)
    expect(described_class.shortfall_message(protocol, purpose: "publication"))
      .to eq("faltam 2 assinaturas de publicação; revisores elegíveis na cidade: 1")
  end
end
```

**Atenção ao "older content":** `update!` de `definition` precisa passar no `before_save` de formato. Use uma mudança que o validador aceite (acrescentar uma chave inócua pode não passar — confira `Protocols::Validator`; se não passar, mude um `prompt`). O que importa é o digest mudar.

**Atenção à ordem temporal no exemplo de ativação:** a assinatura de `bia` precisa ter `created_at` estritamente depois da ativação. Se as duas caírem no mesmo microssegundo, use `travel(1.second)` entre elas (`include ActiveSupport::Testing::TimeHelpers`).

`spec/commands/protocols_authoring_spec.rb`:

```ruby
require "rails_helper"

# Spec §4: quem edita fica registrado e não assina; enviar para revisão congela
# o conteúdo que se assina; editar em revisão volta a rascunho.
RSpec.describe "Protocol authoring and signing" do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:author) do
    User.create!(email_address: "au-#{SecureRandom.hex(3)}@example.org", password: "secret123").tap do |u|
      Membership.create!(user: u, role: "protocol_author", granted_at: Time.current)
    end
  end
  let(:reviewer) { make_reviewer! }
  let(:maintainer_actor) do
    Maintenance::MaintainerActor.new(
      Maintainer.create!(email_address: "am-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                         otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
    )
  end
  let(:correlation_id) { SecureRandom.uuid }

  def save!(by:, definition: protocol_definition_hash, **kw)
    Protocols::SaveDraft.call(definition: definition, by: by, **kw)
  end

  def version = ProtocolDefinition.find_by!(name: "dengue", version: 1)

  describe "SaveDraft" do
    it "records every editor, user or maintainer, with the digest they saved" do
      save!(by: author)
      save!(by: maintainer_actor, correlation_id: correlation_id)

      rows = version.contributions.order(:created_at)
      expect(rows.map(&:actor_kind)).to eq(%w[user maintainer])
      expect(rows.map(&:actor_id)).to eq([ author.id, maintainer_actor.id ])
      expect(rows.map(&:content_digest).uniq).to eq([ version.content_digest ])
      event = DomainEvent.where(name: "protocol.draft_saved").order(:occurred_at).last
      expect(event.payload).to include("actor" => maintainer_actor.id, "actor_kind" => "maintainer",
                                       "correlation_id" => correlation_id)
    end

    it "sends a version under review back to draft when it is edited" do
      save!(by: author)
      version.update!(status: "in_review")

      save!(by: author)

      expect(version.status).to eq("draft")
    end

    it "still refuses to edit a published version" do
      save!(by: author)
      version.update!(status: "published")

      expect(save!(by: author).reason).to eq(:version_not_editable)
    end
  end

  describe "SubmitForReview" do
    it "moves a draft to in_review and records the digest under review" do
      save!(by: author)

      result = Protocols::SubmitForReview.call(name: "dengue", version: 1, by: author)

      expect(result.ok?).to be(true)
      expect(version.status).to eq("in_review")
      expect(DomainEvent.find_by!(name: "protocol.submitted_for_review").payload)
        .to include("content_digest" => version.content_digest, "actor_kind" => "user")
    end

    it "is allowed to the maintainer" do
      save!(by: author)

      expect(Protocols::SubmitForReview.call(name: "dengue", version: 1, by: maintainer_actor).ok?).to be(true)
    end

    it "refuses a version that is not a draft" do
      save!(by: author)
      version.update!(status: "published")

      expect(Protocols::SubmitForReview.call(name: "dengue", version: 1, by: author).reason).to eq(:invalid_state)
    end
  end

  describe "Sign" do
    before do
      save!(by: author)
      Protocols::SubmitForReview.call(name: "dengue", version: 1, by: author)
    end

    def sign(by:, purpose: "publication")
      Protocols::Sign.call(name: "dengue", version: 1, purpose: purpose, by: by)
    end

    it "records a reviewer's signature on the current content" do
      result = sign(by: reviewer)

      expect(result.ok?).to be(true)
      expect(result.payload[:signature]).to have_attributes(signer_user_id: reviewer.id, purpose: "publication",
                                                            content_digest: version.content_digest)
      expect(DomainEvent.find_by!(name: "protocol.signed").payload)
        .to include("purpose" => "publication", "actor" => reviewer.id, "content_digest" => version.content_digest)
    end

    it "refuses the maintainer, even though it passes every role question" do
      expect(sign(by: maintainer_actor).reason).to eq(:maintainer_cannot_sign)
      expect(version.signatures).to be_empty
    end

    it "refuses someone without the reviewer role" do
      expect(sign(by: author).reason).to eq(:forbidden)
    end

    it "refuses a reviewer who edited the version" do
      Membership.create!(user: author, role: "protocol_reviewer", granted_at: Time.current)

      expect(sign(by: author).reason).to eq(:contributor_cannot_sign)
    end

    it "refuses a second signature on the same content for the same purpose" do
      sign(by: reviewer)

      expect(sign(by: reviewer).reason).to eq(:already_signed)
    end

    it "signs publication only under review, and activation only when published" do
      expect(sign(by: reviewer, purpose: "activation").reason).to eq(:invalid_state)

      version.update!(status: "published")
      expect(sign(by: reviewer, purpose: "publication").reason).to eq(:invalid_state)
      expect(sign(by: reviewer, purpose: "activation").ok?).to be(true)
    end

    it "refuses an unknown purpose and an unknown version" do
      expect(sign(by: reviewer, purpose: "x").reason).to eq(:invalid_purpose)
      expect(Protocols::Sign.call(name: "dengue", version: 9, purpose: "publication", by: reviewer).reason)
        .to eq(:not_found)
    end
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falham**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/protocols/signatures_spec.rb spec/commands/protocols_authoring_spec.rb`

- [ ] **Step 3: Implementar a regra**

`app/services/protocols/signatures.rb`:

```ruby
# A ÚNICA resposta para "esta versão tem as assinaturas?" (spec de assinaturas
# §5). Publish e Activate perguntam aqui, no momento do ato — nunca antes —,
# porque as condições mudam depois da assinatura: o conteúdo pode ser editado,
# o revisor pode perder o papel, quem assinou pode passar a editar.
#
# Uma assinatura conta quando, AGORA:
#   1. a finalidade é a do ato;
#   2. o digest é o do conteúdo atual;
#   3. o signatário ainda tem protocol_reviewer ativo e não foi desativado;
#   4. o signatário nunca contribuiu para a versão;
#   5. (ativação) é posterior à última ativação da versão — o ato consome as
#      assinaturas sem alterar nenhuma linha.
module Protocols
  module Signatures
    REQUIRED = 2
    PURPOSE_LABELS = { "publication" => "publicação", "activation" => "ativação" }.freeze

    module_function

    def valid_signer_ids(protocol, purpose:)
      scope = ProtocolSignature.where(protocol_definition_id: protocol.id, purpose: purpose,
                                      content_digest: protocol.content_digest)
                               .where(signer_user_id: active_reviewer_ids)
                               .where.not(signer_user_id: contributor_ids(protocol))

      if purpose == "activation"
        last_activation = ProtocolActivation.where(protocol_definition_id: protocol.id).maximum(:created_at)
        scope = scope.where("protocol_signatures.created_at > ?", last_activation) if last_activation
      end

      scope.distinct.pluck(:signer_user_id)
    end

    def missing(protocol, purpose:)
      [ REQUIRED - valid_signer_ids(protocol, purpose: purpose).size, 0 ].max
    end

    def eligible_reviewer_count(protocol)
      User.where(id: active_reviewer_ids).where.not(id: contributor_ids(protocol)).count
    end

    def shortfall_message(protocol, purpose:)
      count = missing(protocol, purpose: purpose)
      falta = count == 1 ? "falta 1 assinatura" : "faltam #{count} assinaturas"
      "#{falta} de #{PURPOSE_LABELS.fetch(purpose)}; revisores elegíveis na cidade: #{eligible_reviewer_count(protocol)}"
    end

    def active_reviewer_ids
      Membership.active.where(role: "protocol_reviewer")
                .joins(:user).where(users: { deactivated_at: nil })
                .select(:user_id)
    end

    def contributor_ids(protocol)
      ProtocolContribution.where(protocol_definition_id: protocol.id, actor_kind: "user").select(:actor_id)
    end
  end
end
```

- [ ] **Step 4: Implementar os commands**

`app/commands/protocols/save_draft.rb` — mantenha a estrutura atual e mude:

```ruby
module Protocols
  module SaveDraft
    # in_review é editável (spec §4): editar em revisão devolve a versão a
    # draft, e as assinaturas do conteúdo antigo deixam de contar pelo digest.
    EDITABLE = %w[draft in_review].freeze

    def self.call(definition:, by:, correlation_id: nil)
      return Result.fail(:city_missing) if Current.city.nil?

      record = ProtocolDefinition.find_or_initialize_by(name: definition["name"], version: definition["version"])
      return Result.fail(:forbidden) unless ProtocolPolicy.new(by, record).author?

      if record.persisted? && !EDITABLE.include?(record.status)
        return Result.fail(:version_not_editable, message: "versão #{record.version} está #{record.status}")
      end

      ApplicationRecord.transaction do
        record.status = "draft"
        record.definition = definition
        record.save!

        # Quem editou nunca assina esta versão (spec S4) — inclusive o mantenedor,
        # que de todo modo não assina.
        ProtocolContribution.create!(protocol_definition: record, actor_id: by.id, actor_kind: by.actor_kind,
                                     content_digest: record.content_digest)

        DomainEvents.publish("protocol.draft_saved", **{
          protocol_definition_id: record.id, protocol_key: record.name, version: record.version,
          content_digest: record.content_digest, actor: by.id, actor_kind: by.actor_kind,
          correlation_id: correlation_id
        }.compact)
      end

      Result.ok(protocol_definition: record)
    rescue ActiveRecord::RecordInvalid => e
      Result.fail(:invalid_definition, message: e.record.errors.full_messages.join(", "))
    end
  end
end
```

Atualize o comentário de cabeçalho do arquivo (autor ou mantenedor; registra a contribuição; `in_review` volta a `draft`).

`app/commands/protocols/submit_for_review.rb`:

```ruby
# Envia um rascunho para revisão: draft → in_review (spec de assinaturas §4).
# É só em in_review que se assina a publicação. O portão roda aqui para que
# revisores não gastem assinatura num protocolo que a publicação recusaria.
#
# Result.ok(protocol_definition:) | Result.fail(:not_found|:forbidden|:invalid_state|:invalid)
module Protocols
  module SubmitForReview
    def self.call(name:, version:, by:, correlation_id: nil)
      return Result.fail(:city_missing) if Current.city.nil?

      protocol = ProtocolDefinition.find_by(name: name, version: version)
      return Result.fail(:not_found) if protocol.nil?
      return Result.fail(:forbidden) unless ProtocolPolicy.new(by, protocol).author?
      unless protocol.status == "draft"
        return Result.fail(:invalid_state, message: "só rascunho vai para revisão (está #{protocol.status})")
      end

      gate = Protocols::Gate.call(protocol.definition)
      return Result.fail(:invalid, message: gate.errors.join("; ")) unless gate.valid?

      ApplicationRecord.transaction do
        protocol.update!(status: "in_review")
        DomainEvents.publish("protocol.submitted_for_review", **{
          protocol_definition_id: protocol.id, protocol_key: protocol.name, version: protocol.version,
          content_digest: protocol.content_digest, actor: by.id, actor_kind: by.actor_kind,
          correlation_id: correlation_id
        }.compact)
      end

      Result.ok(protocol_definition: protocol)
    rescue ActiveRecord::RecordInvalid => e
      Result.fail(:invalid, message: e.record.errors.full_messages.join(", "))
    end
  end
end
```

`app/commands/protocols/sign.rb`:

```ruby
# Um revisor assina uma versão para uma finalidade (spec de assinaturas §5).
#
# O mantenedor é recusado PELO TIPO DE ATOR, antes de qualquer outra coisa: ele
# responde "sim" a toda pergunta de papel (D6), então ProtocolPolicy#review?
# o deixaria passar. Assinar é aprovar, e aprovar é só de gente da cidade.
#
# Step-up de MFA é do chamador (endpoint da cidade), como em Publish.
#
# Result.ok(signature:) | Result.fail(:maintainer_cannot_sign|:forbidden|:invalid_purpose|
#   :not_found|:invalid_state|:contributor_cannot_sign|:already_signed)
module Protocols
  module Sign
    SIGNABLE_IN = { "publication" => "in_review", "activation" => "published" }.freeze

    def self.call(name:, version:, purpose:, by:)
      return Result.fail(:city_missing) if Current.city.nil?
      return Result.fail(:maintainer_cannot_sign, message: "o mantenedor não assina protocolo") unless by.actor_kind == "user"
      return Result.fail(:invalid_purpose) unless SIGNABLE_IN.key?(purpose.to_s)

      protocol = ProtocolDefinition.find_by(name: name, version: version)
      return Result.fail(:not_found) if protocol.nil?
      return Result.fail(:forbidden) unless ProtocolPolicy.new(by, protocol).review?

      expected = SIGNABLE_IN.fetch(purpose.to_s)
      unless protocol.status == expected
        return Result.fail(:invalid_state, message: "assinatura de #{purpose} só com a versão em #{expected} " \
                                                    "(está #{protocol.status})")
      end
      if protocol.contributions.exists?(actor_id: by.id, actor_kind: "user")
        return Result.fail(:contributor_cannot_sign, message: "quem editou a versão não a assina")
      end
      if Signatures.valid_signer_ids(protocol, purpose: purpose.to_s).include?(by.id)
        return Result.fail(:already_signed, message: "você já assinou este conteúdo para #{purpose}")
      end

      signature = nil
      ApplicationRecord.transaction do
        signature = ProtocolSignature.create!(protocol_definition: protocol, purpose: purpose.to_s, signer: by,
                                              content_digest: protocol.content_digest)
        DomainEvents.publish("protocol.signed",
                             protocol_definition_id: protocol.id, protocol_key: protocol.name,
                             version: protocol.version, purpose: purpose.to_s,
                             content_digest: signature.content_digest, actor: by.id, actor_kind: by.actor_kind)
      end

      Result.ok(signature: signature)
    rescue ActiveRecord::RecordInvalid => e
      Result.fail(:invalid, message: e.record.errors.full_messages.join(", "))
    end
  end
end
```

- [ ] **Step 5: Rodar, suíte completa e commit**

`Authoring::ProtocolsController#draft` continua chamando `SaveDraft` com `by: Current.user`; confira que o spec dele (se existir) segue verde e registre.

```bash
cd apps/api
/opt/homebrew/bin/git add app/services/protocols/signatures.rb app/commands/protocols/save_draft.rb \
  app/commands/protocols/submit_for_review.rb app/commands/protocols/sign.rb \
  spec/services/protocols/signatures_spec.rb spec/commands/protocols_authoring_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: record protocol editors, submit for review and sign

Every draft save records who edited the version, so they can never sign
it. Signing happens only under review for publication and only when
published for activation, by a reviewer of the city, never by the
maintainer. Whether a version has its signatures has one answer, asked
at the moment of the act, because every condition can change after a
signature is given.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 5: Publicar e ativar só com as assinaturas

**Files:**
- Modify: `app/commands/protocols/publish.rb`, `app/commands/protocols/activate.rb`, `app/commands/protocols/retire.rb`, `spec/commands/protocols_lifecycle_spec.rb`, `spec/commands/protocols_publish_gate_spec.rb`, `spec/controllers/publications_controller_spec.rb`
- Test: `spec/commands/protocols_signed_acts_spec.rb`

**Interfaces:**
- Consumes: `Protocols::Signatures` (Task 4); `ProtocolActivation` (Task 2).
- Produces:
  - `Protocols::Publish.call(version:, by:, name: nil, correlation_id: nil)` — só de `in_review`; exige `Signatures.missing(..., purpose: "publication") == 0`, senão `Result.fail(:signatures_missing, message: shortfall_message)`
  - `Protocols::Activate.call(version:, by:, name: nil, correlation_id: nil)` — exige as assinaturas de ativação; grava `ProtocolActivation(kind: "signed")` na mesma transação
  - `Protocols::Retire.call(version:, by:, name: nil, correlation_id: nil)` — regra inalterada
  - Os três eventos levam `actor_kind` e, quando informado, `correlation_id`

- [ ] **Step 1: Escrever o spec que falha**

`spec/commands/protocols_signed_acts_spec.rb`:

```ruby
require "rails_helper"

# Spec §4/§5: publicar e ativar exigem duas assinaturas válidas cada; o
# mantenedor executa o ato quando elas existem e é recusado quando não.
RSpec.describe "Protocol acts that require signatures" do
  include ActiveSupport::Testing::TimeHelpers

  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:publisher) do
    User.create!(email_address: "pb-#{SecureRandom.hex(3)}@example.org", password: "secret123").tap do |u|
      Membership.create!(user: u, role: "protocol_publisher", granted_at: Time.current)
    end
  end
  let(:ana) { make_reviewer! }
  let(:bia) { make_reviewer! }
  let(:maintainer_actor) do
    Maintenance::MaintainerActor.new(
      Maintainer.create!(email_address: "sa-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                         otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
    )
  end

  def version(v = 1) = ProtocolDefinition.find_by!(name: "dengue", version: v)

  def in_review!(v = 1)
    ProtocolDefinition.create!(name: "dengue", version: v, status: "in_review",
                               definition: protocol_definition_hash(version: v))
  end

  describe "Publish" do
    it "publishes with two valid publication signatures" do
      in_review!
      sign!(version, purpose: "publication", by: ana)
      sign!(version, purpose: "publication", by: bia)

      expect(Protocols::Publish.call(version: 1, name: "dengue", by: publisher).ok?).to be(true)
      expect(version.status).to eq("published")
    end

    it "refuses with one signature, saying how many are missing and how many reviewers are eligible" do
      in_review!
      sign!(version, purpose: "publication", by: ana)
      bia

      result = Protocols::Publish.call(version: 1, name: "dengue", by: publisher)

      expect(result.reason).to eq(:signatures_missing)
      expect(result.message).to eq("falta 1 assinatura de publicação; revisores elegíveis na cidade: 2")
      expect(version.status).to eq("in_review")
    end

    it "no longer publishes straight from a draft" do
      ProtocolDefinition.create!(name: "dengue", version: 1, status: "draft", definition: protocol_definition_hash)

      expect(Protocols::Publish.call(version: 1, name: "dengue", by: publisher).reason).to eq(:invalid_state)
    end

    it "lets the maintainer publish when the city's signatures exist, and refuses it when they do not" do
      in_review!
      expect(Protocols::Publish.call(version: 1, name: "dengue", by: maintainer_actor).reason).to eq(:signatures_missing)

      sign!(version, purpose: "publication", by: ana)
      sign!(version, purpose: "publication", by: bia)
      correlation_id = SecureRandom.uuid

      expect(Protocols::Publish.call(version: 1, name: "dengue", by: maintainer_actor,
                                     correlation_id: correlation_id).ok?).to be(true)
      expect(DomainEvent.find_by!(name: "protocol.published").payload)
        .to include("actor" => maintainer_actor.id, "actor_kind" => "maintainer", "correlation_id" => correlation_id)
    end

    it "does not accept activation signatures for publication" do
      in_review!
      sign!(version, purpose: "activation", by: ana)
      sign!(version, purpose: "activation", by: bia)

      expect(Protocols::Publish.call(version: 1, name: "dengue", by: publisher).reason).to eq(:signatures_missing)
    end
  end

  describe "Activate" do
    def published!(v = 1)
      ProtocolDefinition.create!(name: "dengue", version: v, status: "published",
                                 definition: protocol_definition_hash(version: v))
    end

    it "activates with two activation signatures and records the act" do
      published!
      sign!(version, purpose: "activation", by: ana)
      sign!(version, purpose: "activation", by: bia)

      expect(Protocols::Activate.call(version: 1, name: "dengue", by: publisher).ok?).to be(true)
      expect(version.status).to eq("active")
      expect(version.activations.pluck(:kind, :actor_id)).to eq([ [ "signed", publisher.id ] ])
    end

    it "refuses without activation signatures, even with publication signatures" do
      published!
      sign!(version, purpose: "publication", by: ana)
      sign!(version, purpose: "publication", by: bia)

      expect(Protocols::Activate.call(version: 1, name: "dengue", by: publisher).reason).to eq(:signatures_missing)
    end

    it "accepts the same reviewers who signed the publication (S8)" do
      published!
      %w[publication activation].each do |purpose|
        sign!(version, purpose: purpose, by: ana)
        sign!(version, purpose: purpose, by: bia)
      end

      expect(Protocols::Activate.call(version: 1, name: "dengue", by: publisher).ok?).to be(true)
    end

    it "consumes the signatures: activating the same version again requires signing again" do
      published!
      published!(2)
      [ 1, 2 ].each do |v|
        sign!(version(v), purpose: "activation", by: ana)
        sign!(version(v), purpose: "activation", by: bia)
      end
      Protocols::Activate.call(version: 1, name: "dengue", by: publisher)
      travel(1.second)
      Protocols::Activate.call(version: 2, name: "dengue", by: publisher)
      travel(1.second)

      expect(Protocols::Activate.call(version: 1, name: "dengue", by: publisher).reason).to eq(:signatures_missing)

      sign!(version(1), purpose: "activation", by: ana)
      sign!(version(1), purpose: "activation", by: bia)
      expect(Protocols::Activate.call(version: 1, name: "dengue", by: publisher).ok?).to be(true)
    end

    it "lets the maintainer activate when the signatures exist, and records it as the actor" do
      published!
      sign!(version, purpose: "activation", by: ana)
      sign!(version, purpose: "activation", by: bia)

      expect(Protocols::Activate.call(version: 1, name: "dengue", by: maintainer_actor).ok?).to be(true)
      expect(version.activations.sole).to have_attributes(actor_id: maintainer_actor.id, actor_kind: "maintainer")
    end

    it "keeps R1: a draft never becomes active" do
      ProtocolDefinition.create!(name: "dengue", version: 1, status: "draft", definition: protocol_definition_hash)

      expect(Protocols::Activate.call(version: 1, name: "dengue", by: publisher).reason).to eq(:not_published)
    end
  end

  it "keeps Retire as it was, now saying who acted and how" do
    ProtocolDefinition.create!(name: "dengue", version: 1, status: "published", definition: protocol_definition_hash)

    expect(Protocols::Retire.call(version: 1, name: "dengue", by: publisher).ok?).to be(true)
    expect(DomainEvent.find_by!(name: "protocol.retired").payload).to include("actor_kind" => "user")
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falham**

- [ ] **Step 3: Implementar**

`Protocols::Publish`:

```ruby
    # Spec de assinaturas §4: publicar só a partir de in_review (draft não é
    # mais publicável) e só com duas assinaturas de publicação válidas.
    PUBLISHABLE_FROM = %w[in_review].freeze

    def self.call(version:, by:, name: nil, correlation_id: nil)
      return Result.fail(:city_missing) if Current.city.nil?

      candidates = ProtocolDefinition.where(version: version)
      candidates = candidates.where(name: name) if name
      # ... not_found, ambiguous, policy (publish?) e o estado, como hoje ...

      gate = Protocols::Gate.call(protocol.definition)
      return Result.fail(:invalid, message: gate.errors.join("; ")) unless gate.valid?

      if Signatures.missing(protocol, purpose: "publication").positive?
        return Result.fail(:signatures_missing, message: Signatures.shortfall_message(protocol, purpose: "publication"))
      end

      ApplicationRecord.transaction do
        protocol.update!(status: "published")
        DomainEvents.publish("protocol.published", **{
          protocol_definition_id: protocol.id, protocol_key: protocol.name, version: protocol.version,
          content_digest: protocol.content_digest,
          signers: Signatures.valid_signer_ids(protocol, purpose: "publication"),
          actor: by.id, actor_kind: by.actor_kind, correlation_id: correlation_id
        }.compact)
      end
```

A mensagem de `:invalid_state` passa a dizer "só in_review pode ser publicado".

`Protocols::Activate`, depois da checagem de R1:

```ruby
      if Signatures.missing(protocol, purpose: "activation").positive?
        return Result.fail(:signatures_missing, message: Signatures.shortfall_message(protocol, purpose: "activation"))
      end
      signers = Signatures.valid_signer_ids(protocol, purpose: "activation")

      ApplicationRecord.transaction do
        # (demoção da active anterior e update! como hoje)
        ProtocolActivation.create!(protocol_definition: protocol, kind: "signed",
                                   actor_id: by.id, actor_kind: by.actor_kind)
        DomainEvents.publish("protocol.activated", **{
          protocol_key: protocol.name, protocol_definition_id: protocol.id, version: protocol.version,
          signers: signers, activated_by: by.id, actor_kind: by.actor_kind, correlation_id: correlation_id
        }.compact)
      end
```

Os signatários são lidos **antes** do `ProtocolActivation.create!` — depois dele, as assinaturas já estão consumidas e a lista viria vazia.

`Protocols::Retire`: `correlation_id: nil` na assinatura e `actor_kind`/`correlation_id` no evento (com `.compact`). Nenhuma outra mudança.

Atualize os comentários de cabeçalho dos três arquivos (`by:` é `User` ou `Maintenance::MaintainerActor`; Publish e Activate exigem assinaturas; step-up é do chamador).

- [ ] **Step 4: Ajustar os specs antigos**

`spec/commands/protocols_lifecycle_spec.rb`, `spec/commands/protocols_publish_gate_spec.rb` e `spec/controllers/publications_controller_spec.rb` quebram de propósito (Decisão 8). Para cada exemplo que publica ou ativa com sucesso, arrume o estado certo (`in_review` para publicar) e as duas assinaturas com `sign!`/`make_reviewer!`. Para cada exemplo que publicava a partir de `draft`, troque a expectativa para a recusa `:invalid_state` **ou** mova a versão para `in_review`, conforme o que o exemplo pretendia provar — e diga no relatório, exemplo por exemplo, qual das duas escolheu. **Nenhum exemplo pode ser apagado sem substituto.**

- [ ] **Step 5: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/commands/protocols/publish.rb app/commands/protocols/activate.rb app/commands/protocols/retire.rb \
  spec/commands/protocols_signed_acts_spec.rb spec/commands/protocols_lifecycle_spec.rb \
  spec/commands/protocols_publish_gate_spec.rb spec/controllers/publications_controller_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: require two reviewer signatures to publish and to activate

Publication now starts only from review and activation only from a
published version, each with two valid signatures of its own purpose.
An activation consumes its signatures, so putting the same version back
in use later asks for new ones. The maintainer may carry out either act
once the city has signed, never before.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 6: Reversão de emergência

**Files:**
- Create: `app/commands/protocols/revert_activation.rb`
- Test: `spec/commands/protocols_revert_activation_spec.rb`

**Interfaces:**
- Consumes: `ProtocolActivation`, `ProtocolPolicy#activate?`.
- Produces: `Protocols::RevertActivation.call(name:, by:, reason:, correlation_id: nil)` → `Result.ok(protocol_definition:)` (a versão que voltou a `active`) | `Result.fail(:reason_required | :not_found | :forbidden | :not_revertible | :no_previous_activation)`; evento `protocol.activation_reverted` (`protocol_key`, `from_version`, `to_version`, `reason`, `actor`, `actor_kind`, `correlation_id`)

- [ ] **Step 1: Escrever o spec que falha**

`spec/commands/protocols_revert_activation_spec.rb`:

```ruby
require "rails_helper"

# Spec §6: voltar para a versão que estava em uso imediatamente antes, sem
# assinaturas, com motivo — e nunca reverter uma reversão, que reativaria a
# versão com erro sem assinatura nenhuma.
RSpec.describe Protocols::RevertActivation do
  include ActiveSupport::Testing::TimeHelpers

  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:publisher) do
    User.create!(email_address: "pb-#{SecureRandom.hex(3)}@example.org", password: "secret123").tap do |u|
      Membership.create!(user: u, role: "protocol_publisher", granted_at: Time.current)
    end
  end
  let(:ana) { make_reviewer! }
  let(:bia) { make_reviewer! }

  def version(v) = ProtocolDefinition.find_by!(name: "dengue", version: v)

  def activate_signed!(v)
    ProtocolDefinition.find_by(name: "dengue", version: v) ||
      ProtocolDefinition.create!(name: "dengue", version: v, status: "published",
                                 definition: protocol_definition_hash(version: v))
    sign!(version(v), purpose: "activation", by: ana)
    sign!(version(v), purpose: "activation", by: bia)
    expect(Protocols::Activate.call(version: v, name: "dengue", by: publisher).ok?).to be(true)
    travel(1.second)
  end

  def revert(reason: "v2 prioriza febre errado", by: publisher)
    described_class.call(name: "dengue", by: by, reason: reason)
  end

  it "puts the previous version back in use without signatures, recording the reason" do
    activate_signed!(1)
    activate_signed!(2)

    result = revert

    expect(result.ok?).to be(true)
    expect([ version(1).status, version(2).status ]).to eq(%w[active published])
    expect(version(1).activations.order(:created_at).last)
      .to have_attributes(kind: "emergency_revert", reason: "v2 prioriza febre errado", actor_id: publisher.id)
    expect(DomainEvent.find_by!(name: "protocol.activation_reverted").payload)
      .to include("from_version" => 2, "to_version" => 1, "reason" => "v2 prioriza febre errado")
  end

  it "refuses to revert a revert" do
    activate_signed!(1)
    activate_signed!(2)
    revert
    travel(1.second)

    expect(revert.reason).to eq(:not_revertible)
    expect(version(1).status).to eq("active")
  end

  it "requires a reason" do
    activate_signed!(1)
    activate_signed!(2)

    expect(revert(reason: "  ").reason).to eq(:reason_required)
  end

  it "refuses when there is no previous activation" do
    activate_signed!(1)

    expect(revert.reason).to eq(:no_previous_activation)
  end

  it "refuses when the previous version was retired in the meantime" do
    activate_signed!(1)
    activate_signed!(2)
    version(1).update!(status: "retired", retired_at: Time.current)

    expect(revert.reason).to eq(:not_revertible)
  end

  it "refuses someone who cannot activate" do
    activate_signed!(1)
    activate_signed!(2)

    expect(revert(by: ana).reason).to eq(:forbidden)
  end

  it "refuses a version activated before signatures existed (no activation record)" do
    ProtocolDefinition.create!(name: "dengue", version: 1, status: "active", definition: protocol_definition_hash)

    expect(revert.reason).to eq(:not_revertible)
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

- [ ] **Step 3: Implementar**

`app/commands/protocols/revert_activation.rb`:

```ruby
# Reversão de emergência (spec de assinaturas §6, S10).
#
# Um protocolo com erro em uso prioriza triagens errado enquanto se esperam
# duas assinaturas. Voltar para a versão que estava em uso IMEDIATAMENTE antes
# não aprova nada de novo: aquele conteúdo já foi publicado e ativado com
# assinaturas nesta cidade. Por isso dispensa assinatura — e por isso é
# estreita:
#   - só a ativação ANTERIOR à atual, por protocolo;
#   - só se a ativação atual foi assinada: reverter uma reversão reativaria a
#     versão com erro sem assinatura nenhuma;
#   - só se a versão anterior ainda está publicada (não aposentada);
#   - motivo obrigatório, gravado na linha e no evento.
#
# Step-up de MFA é do chamador.
module Protocols
  module RevertActivation
    def self.call(name:, by:, reason:, correlation_id: nil)
      return Result.fail(:city_missing) if Current.city.nil?
      return Result.fail(:reason_required, message: "a reversão de emergência exige um motivo") if reason.to_s.strip.empty?

      current = ProtocolDefinition.find_by(name: name, status: "active")
      return Result.fail(:not_found) if current.nil?
      return Result.fail(:forbidden) unless ProtocolPolicy.new(by, current).activate?

      latest, previous = ProtocolActivation.joins(:protocol_definition)
                                           .where(protocol_definitions: { name: name })
                                           .order(created_at: :desc).limit(2).to_a

      unless latest&.protocol_definition_id == current.id && latest.kind == "signed"
        return Result.fail(:not_revertible,
                           message: "só se reverte uma ativação assinada; reversão de reversão é recusada")
      end
      return Result.fail(:no_previous_activation) if previous.nil?

      target = previous.protocol_definition
      unless target.status == "published"
        return Result.fail(:not_revertible, message: "a versão anterior não está publicada (está #{target.status})")
      end

      ApplicationRecord.transaction do
        current.update!(status: "published")
        target.update!(status: "active", activated_at: Time.current)
        ProtocolActivation.create!(protocol_definition: target, kind: "emergency_revert",
                                   actor_id: by.id, actor_kind: by.actor_kind, reason: reason.to_s.strip)
        DomainEvents.publish("protocol.activation_reverted", **{
          protocol_key: name, from_version: current.version, to_version: target.version,
          reason: reason.to_s.strip, actor: by.id, actor_kind: by.actor_kind, correlation_id: correlation_id
        }.compact)
      end

      Result.ok(protocol_definition: target)
    rescue ActiveRecord::RecordInvalid => e
      Result.fail(:invalid, message: e.record.errors.full_messages.join(", "))
    end
  end
end
```

**Confira** a ordem das duas escritas contra o índice único parcial `WHERE status = 'active'`: a atual é demovida **antes** de a anterior virar `active` (mesma ordem de `Activate`).

- [ ] **Step 4: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/commands/protocols/revert_activation.rb spec/commands/protocols_revert_activation_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: allow an emergency revert to the previously active protocol

A protocol in use with an error misprioritizes triages while two
signatures are gathered. Going back to the version in use immediately
before approves nothing new, so it needs no signatures, only a reason.
It only moves back, only from a signed activation, and never reverts a
revert, which would bring the faulty version back unsigned.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 7: Guardas e ADRs

**Files:**
- Create: `spec/architecture/protocol_signatures_guard_spec.rb`; no repo `docs`: `adr/0016.md`
- Modify: no repo `docs`: `adr/0012.md`, `adr/0009.md`, `adr/README.md`
- Test: `spec/architecture/protocol_signatures_guard_spec.rb`

- [ ] **Step 1: Guarda "só três commands põem protocolo em uso ou publicado"**

`spec/architecture/protocol_signatures_guard_spec.rb`:

```ruby
require "rails_helper"

# Spec de assinaturas §10: um caminho novo que publique ou ative protocolo sem
# passar pela verificação de assinaturas derrubaria a regra inteira sem nenhum
# teste vermelho. Esta guarda lê o código.
RSpec.describe "Protocol signatures guard" do
  # Escrita de status de protocolo para published/active: update!/update/
  # update_all/update_column(s) com status: "published" | "active", ou atribuição.
  # Método, não constante: constante dentro de RSpec.describe vaza para Object.
  def status_write
    /(update!?|update_all|update_columns?)\s*\(?\s*status:\s*"(published|active)"|\.status\s*=\s*"(published|active)"/
  end

  def app_files_touching_protocols
    Dir[Rails.root.join("app/**/*.rb")].select { |path| File.read(path).match?(/ProtocolDefinition|protocol\b/) }
  end

  it "writes a protocol status of published or active only in the three signed-act commands" do
    allowed = %w[publish.rb activate.rb revert_activation.rb].map { |f| Rails.root.join("app/commands/protocols", f).to_s }

    offenders = app_files_touching_protocols.reject { |path| allowed.include?(path) }
                                            .select { |path| File.read(path).match?(status_write) }

    expect(offenders).to be_empty
  end

  it "makes publish and activate ask Protocols::Signatures before the act" do
    %w[publish.rb activate.rb].each do |file|
      source = File.read(Rails.root.join("app/commands/protocols", file))
      expect(source).to include("Signatures.missing("), "#{file} publica/ativa sem perguntar as assinaturas"
    end
  end

  it "refuses the maintainer by kind of actor wherever approval is created" do
    {
      "app/commands/protocols/sign.rb" => /actor_kind\s*==\s*"user"/,
      "app/commands/grant_role.rb" => /actor_kind\s*==\s*"maintainer"/,
      "app/commands/invite_member.rb" => /actor_kind\s*==\s*"maintainer"/
    }.each do |file, pattern|
      expect(File.read(Rails.root.join(file))).to match(pattern), "#{file} não checa o tipo de ator"
    end
  end
end
```

**Confira** a guarda contra o código real: `ConversationAdvance` escreve `status: "active"` de **conversa** — o filtro `app_files_touching_protocols` precisa deixá-lo de fora sem esconder um arquivo de protocolo. Se o filtro por texto for frouxo ou apertado demais, ajuste-o (por exemplo, exigindo `ProtocolDefinition` no arquivo) e prove as duas direções: a guarda passa no código atual e falha com uma escrita temporária de `status: "published"` num arquivo de protocolo fora da lista (revertida).

- [ ] **Step 2: Rodar e commit (api)**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/architecture/protocol_signatures_guard_spec.rb`

Depois a suíte completa (worker parado antes, religado depois).

```bash
cd apps/api
/opt/homebrew/bin/git add spec/architecture/protocol_signatures_guard_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
test: guard that only signed acts publish or activate a protocol

Publication and activation are written in three commands only, two of
them ask for the signatures before the act, and every place that could
create an approver refuses the maintainer by its kind of actor.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

- [ ] **Step 3: ADRs (repo `docs`)**

Leia `docs/adr/0015.md` e `docs/adr/README.md` para seguir o formato e a indexação.

1. **`adr/0016.md` — "Protocol signatures: two city reviewers per publication and per activation".** Status aceita, data 2026-09-18. Decisão: o conteúdo das decisões S1–S11 da spec `superpowers/specs/2026-09-18-protocol-signatures-design.md`, em forma de ADR (contexto: o colapso do quatro-olhos permitido pelo ADR 0012; decisão; consequências: cidade pequena bloqueada até designar revisores, reversão de emergência como única exceção, mantenedor nunca assina). Referências: ADRs 0009, 0011, 0012 e a spec.
2. **`adr/0012.md`:** no parágrafo **Quatro-olhos**, marque o texto como substituído pelo ADR 0016 (colapso não é mais permitido; o operador de plataforma não é mais reserva do publisher), sem apagar o histórico; acrescente `protocol_reviewer` à tabela de papéis ("assina publicação e ativação de protocolo; concedido só pelo `municipal_admin`"); atualize o status do ADR para "Aceita; quatro-olhos substituído pelo ADR 0016".
3. **`adr/0009.md`:** o `in_review` passa a ser o estado em que se assina (`draft → in_review` por envio explícito; editar volta a `draft`); publicar sai só de `in_review` com duas assinaturas; ativar exige duas assinaturas de ativação; a reversão de emergência. Aponte para o ADR 0016.
4. **`adr/README.md`:** o ADR 0016 no índice.

```bash
cd docs
/opt/homebrew/bin/git add adr/0016.md adr/0012.md adr/0009.md adr/README.md
/opt/homebrew/bin/git commit -F - <<'EOF'
docs: add ADR 0016 on protocol signatures

Publication and activation of a protocol each require two signatures
from city reviewers who did not edit the version. The four-eyes collapse
that ADR 0012 allowed, and the platform operator as a backup publisher,
are superseded; ADR 0009 now uses in_review as the state where
signatures happen.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

## Verificação final do plano

- [ ] Suíte completa com o worker parado: 0 falhas.
- [ ] Os bancos de teste e as cidades de dev ativas estão na versão `20260918000001`, com os três triggers (`SELECT tgname FROM pg_trigger WHERE tgname LIKE '%append_only'`).
- [ ] Publicar e ativar exigem duas assinaturas válidas cada; quem editou não assina; editar derruba as assinaturas; revisor revogado ou desativado não conta; a ativação consome as assinaturas.
- [ ] O mantenedor edita, envia para revisão, publica e ativa com as assinaturas da cidade — e nunca assina, concede nem convida revisor ou `municipal_admin`.
- [ ] A reversão de emergência volta só para a ativação anterior, com motivo, e recusa reverter uma reversão.
- [ ] Nenhuma regra lê `domain_events`: `grep -rn "DomainEvent\.\(where\|find\)" app/commands app/services/protocols` sem ocorrência nova.
- [ ] `CityInventory` intocado: `git diff main -- app/services/city_inventory.rb` vazio.

## Fora deste plano

- **Endpoints da cidade** (enviar para revisão, assinar, ativar, reverter, conceder papel), com step-up: fatia 2 — plano próprio.
- **Plano 5 da API de manutenção, reescrito** sobre este: mutations de salvar rascunho, enviar para revisão, publicar, ativar, reverter e aposentar; nenhuma de assinatura. O Plano 5 antigo (`2026-09-18-maintenance-api-05-escrita-e-protocolos.md`, não commitado) fica obsoleto.
- **Leitura** das assinaturas nas superfícies (quantas faltam, quem assinou, revisores elegíveis) e o `Admin::ProtocolsQuery#four_eyes`, que passa a poder ler as tabelas novas: fatia 2.
- **Telas do dashboard:** spec própria.
- **Recusa formal e retirada de assinatura:** fora de escopo (spec §12).
