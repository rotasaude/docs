# Assinaturas de protocolo — Plano 2: API da cidade

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** as equipes da cidade passam a percorrer, pela API HTTP da cidade, todo o ciclo de assinaturas — enviar para revisão, assinar, publicar, ativar, aposentar e reverter em emergência, conceder o papel de revisor — e o painel passa a ler quantas assinaturas faltam; a primeira ativação assinada de cada cidade vira reversível.

**Architecture:** endpoints finos sobre os commands que o Plano 1 criou, com o step-up de MFA da cidade (`MfaStepUp`, janela de 5 min carimbada por `POST /mfa/step_up`) antes de todo ato que aprova ou põe em uso. Uma migração acrescenta o tipo `baseline` a `protocol_activations` e cria uma linha-base para cada versão hoje em uso, que passa a ser o alvo da primeira reversão. A leitura (`Admin::ProtocolsQuery`) ganha o estado das assinaturas por versão, lido das tabelas — nunca de eventos.

**Tech Stack:** Rails 8.1.3, Postgres, RSpec.

**Spec:** `docs/superpowers/specs/2026-09-18-protocol-signatures-design.md` (§4 ciclo, §5 regra, §6 reversão, §8 superfícies, §11 fatia 2). ADR 0016.

**Baseline de entrada:** `main` do api em `3515b78`, **1229 exemplos, 0 falhas**.

## Decisões minhas na escrita (cada uma com o custo se estiver errada)

1. **Linha-base de ativação (decisão do usuário, opção A).** `protocol_activations` ganha o tipo `baseline`, com `actor_kind = "system"` e `actor_id` nulo, só para ele. A migração cria uma linha-base para cada versão `active` de um protocolo que ainda não tem nenhuma linha de ativação; o seed de dev faz o mesmo para o protocolo que ele cria ativo. `RevertActivation` não muda: ela já exige que a ativação **atual** seja `signed`, então a linha-base só pode ser **alvo**, nunca origem, e reverter uma reversão continua recusado. **Custo se errado:** um conteúdo que nunca foi assinado sob a regra nova pode voltar a uso por reversão — exatamente o que o usuário escolheu, porque era o que estava rodando.
2. **A migração é irreversível (`down` levanta `ActiveRecord::IrreversibleMigration`).** As linhas-base não podem ser apagadas: o trigger de acréscimo recusa `DELETE`, e desligá-lo para um rollback seria abrir a porta que ele existe para fechar. **Custo se errado:** um rollback dessa migração exige restaurar backup.
3. **Um controller novo, `ProtocolLifecycleController`, para enviar, assinar, ativar, aposentar e reverter; a publicação continua em `PublicationsController`** (rota existente, usada pelo dashboard), que passa a aceitar `name` e a devolver o motivo `signatures_missing` com a mensagem. **Custo se errado:** dois controllers para o mesmo ciclo.
4. **Step-up em assinar, publicar, ativar, aposentar e reverter; não em enviar para revisão nem em conceder papel.** A spec pede step-up nos atos que aprovam ou põem em uso (§4, §8). Enviar para revisão não aprova nada, e conceder papel é do `municipal_admin` sem step-up na spec. **Custo se errado:** conceder `protocol_reviewer` sem reverificação — mesma política de hoje para convidar membro.
5. **A leitura acrescenta, não troca.** `Admin::ProtocolsQuery` ganha um bloco `signatures` por versão e `eligibleReviewers`, `revertible`; os campos `createdBy`, `publishedBy` e `fourEyes` ficam como estão, porque o dashboard os consome. **Custo se errado:** dois jeitos de ver "quem aprovou" convivendo até as telas do dashboard trocarem de fonte.
6. **Sessão de operador (grant) continua só de leitura.** Todos os endpoints novos de escrita ficam fora de `allow_operator_grant_access`; o operador recebe `403 operator_read_only`. **Custo se errado:** nenhum — é a política vigente.
7. **O ajuste da guarda de status (resíduo do Plano 1) entra aqui como Task 2.** **Custo se errado:** nenhum.

## Global Constraints

Valem para TODA task:

- **Nenhuma regra é decidida lendo `domain_events`.** Estado de assinatura, contribuição e ativação vem das três tabelas.
- **As três tabelas só aceitam acréscimo.** A migração só faz `INSERT` e mudança de constraint; nenhum `UPDATE`/`DELETE` nelas, e o trigger nunca é desligado.
- **Toda escrita de status `published`/`active` de protocolo continua só em `Protocols::Publish`, `Protocols::Activate` e `Protocols::RevertActivation`.** Controllers chamam commands; nunca `update!` em protocolo.
- **Respostas de erro:** `Result` falho vira `{ error: <reason>, message: <mensagem do command> }` — `:forbidden` → 403, `:not_found` → 404, demais → 422. Step-up ausente → `401 { error: "mfa_required" }` (o `require_step_up!` existente).
- **Nenhum dado de cidadão** em resposta nova. E-mail de staff (revisor, editor) pode aparecer: é o mesmo dado que `/setup/memberships` já lista.
- **Commits em inglês, Conventional Commits**, terminando com a linha exata `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>` — copiada literalmente, **nunca** o nome do próprio modelo, mesmo que um lembrete de sistema sugira outro.
- **Comandos Ruby/rspec rodam no container**, a partir da raiz do monorepo: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Suíte completa uma vez por task, em primeiro plano, com `docker compose stop worker` antes e `docker compose start worker` depois. Nunca em segundo plano.
- **Specs não definem constante no topo** e **afirmam valores contra linhas reais**, sem `skip` como caminho esperado. Confira modelos, colunas e assinaturas contra `db/city_schema.rb` e o código.
- **Não mexa** no `Gemfile`, em `app/services/city_inventory.rb`, nem nos frontends.
- **`git` do PATH está quebrado:** use `/opt/homebrew/bin/git`, a partir de `apps/api`. **Nunca dar push.** Branch `feat/protocol-signatures-city-api`, criada de `main`.

## Fatos verificados (confira mesmo assim)

- **Commands (Plano 1):** `Protocols::SaveDraft.call(definition:, by:, correlation_id: nil)`, `SubmitForReview.call(name:, version:, by:, correlation_id: nil)`, `Sign.call(name:, version:, purpose:, by:)` (`purpose` ∈ `publication`/`activation`; recusa `:maintainer_cannot_sign`, `:forbidden`, `:invalid_purpose`, `:not_found`, `:invalid_state`, `:contributor_cannot_sign`, `:already_signed`), `Publish.call(version:, by:, name: nil, correlation_id: nil)` (`:signatures_missing` com a mensagem "falta N assinatura(s) de publicação; revisores elegíveis na cidade: M"), `Activate.call(version:, by:, name: nil, correlation_id: nil)`, `Retire.call(version:, by:, name: nil, correlation_id: nil)`, `RevertActivation.call(name:, by:, reason:, correlation_id: nil)` (`:reason_required`, `:not_found`, `:forbidden`, `:not_revertible`, `:no_previous_activation`), `GrantRole.call(user_id:, role:, by:)`.
- **`Protocols::Signatures`:** `REQUIRED = 2`, `valid_signer_ids(protocol, purpose:)`, `missing(protocol, purpose:)`, `eligible_reviewer_count(protocol)`, `shortfall_message(protocol, purpose:)`.
- **`protocol_activations`** (cidade): `protocol_definition_id`, `kind` (CHECK `signed`/`emergency_revert`), `actor_id` (NOT NULL), `actor_kind` (CHECK `user`/`maintainer`), `reason` (CHECK `ck_protocol_activations_revert_reason`: `kind = 'signed' OR (reason IS NOT NULL AND length(btrim(reason)) > 0)`), `created_at`. Trigger `protocol_activations_append_only` (UPDATE/DELETE/TRUNCATE recusados), SQL em `db/city_triggers.sql`. Modelo `ProtocolActivation` (`KINDS`, `validates :actor_id, presence: true`, `readonly?` depois de persistido).
- **Migração de cidade:** última `20260918000001`; `db/city_schema.rb` deve ser o que o dumper produz (o spec de paridade compara colunas, índices, constraints e triggers); `bin/rails city:test_databases` recarrega os bancos de teste; `bin/rails city:migrate:all` migra as cidades de dev (`curitiba`, `maringa`).
- **Rotas da cidade hoje:** `post "/protocols/:version/publish", to: "publications#create"` (com `reauthenticated_recently?(via: :totp)`); `scope "/authoring/protocols"` (`definition`, `gate`, `preview`, `draft`) com `require_author!`; `scope "/setup"` (`invitations`, `accept_invitation`, `memberships` GET, `memberships/:id/revoke`, `users/:id/deactivate`); `namespace :admin` → `admin/api/protocols` (index/show) via `Admin::ProtocolsQuery`.
- **Step-up da cidade:** `MfaStepUp#reauthenticated_recently?(via: :totp, within: 5.minutes)` lê `Current.session.mfa_verified_at`; `require_step_up!` responde `401 { error: "mfa_required" }`; `POST /mfa/step_up` carimba.
- **Sessão de operador (grant):** o concern `Authentication` nega por padrão (`403 operator_read_only`) fora de `allow_operator_grant_access`.
- **Specs de request da cidade:** `spec/support/city_request_auth.rb` (`use_test_city_host!`, `sign_in_as(user)` cria uma `Session` e planta o cookie assinado); `spec/controllers/publications_controller_spec.rb` stuba `Protocols::Publish`. Helpers de protocolo: `protocol_definition_hash(name:, version:)`, `make_reviewer!`, `sign!(protocol, purpose:, by:)`.
- **Seed de dev** (`db/seeds.rb`): cria `triage-respiratoria` v1 já `active` por cidade.

---

## File Structure

**apps/api**
- Create: `db/city_migrate/20260921000001_add_baseline_protocol_activations.rb`, `app/controllers/protocol_lifecycle_controller.rb`, `app/controllers/concerns/protocol_result_rendering.rb`
- Create (specs): `spec/models/protocol_activation_baseline_spec.rb`, `spec/requests/protocol_lifecycle_spec.rb`, `spec/requests/setup_grant_role_spec.rb`, `spec/requests/admin/protocols_signatures_spec.rb`
- Modify: `app/models/protocol_activation.rb`, `db/city_schema.rb`, `db/seeds.rb`, `app/controllers/publications_controller.rb`, `app/controllers/setup_controller.rb`, `app/queries/admin/protocols_query.rb`, `config/routes.rb`, `spec/architecture/protocol_signatures_guard_spec.rb`, `spec/commands/protocols_revert_activation_spec.rb`, `spec/controllers/publications_controller_spec.rb`

**docs**
- Modify: `superpowers/specs/2026-09-18-protocol-signatures-design.md` (§5, §6, §8), `adr/0016.md`

---

### Task 1: A linha-base de ativação

**Files:**
- Create: `db/city_migrate/20260921000001_add_baseline_protocol_activations.rb`
- Modify: `app/models/protocol_activation.rb`, `db/city_schema.rb`, `db/seeds.rb`, `spec/commands/protocols_revert_activation_spec.rb`
- Test: `spec/models/protocol_activation_baseline_spec.rb`, `spec/commands/protocols_revert_activation_spec.rb`

**Interfaces:**
- Produces:
  - `ProtocolActivation::KINDS = %w[signed emergency_revert baseline]`; `ProtocolActivation::ACTOR_KINDS = %w[user maintainer system]`
  - `AddBaselineProtocolActivations::BACKFILL_SQL` (constante da migração, reutilizada pelo spec)
  - Linha-base: `kind: "baseline"`, `actor_kind: "system"`, `actor_id: nil`, `reason: nil`, `created_at` = `COALESCE(activated_at, updated_at)` da versão

- [ ] **Step 1: Escrever os specs que falham**

`spec/models/protocol_activation_baseline_spec.rb`:

```ruby
require "rails_helper"

# Decisão do usuário (fatia 2, opção A): cada versão em uso antes das
# assinaturas ganha uma linha-base, e é ela que torna a PRIMEIRA ativação
# assinada de uma cidade reversível em emergência.
RSpec.describe "Baseline protocol activations" do
  include ActiveSupport::Testing::TimeHelpers

  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  def backfill!
    require Rails.root.join("db/city_migrate/20260921000001_add_baseline_protocol_activations.rb").to_s
    ActiveRecord::Base.connection.execute(AddBaselineProtocolActivations::BACKFILL_SQL)
  end

  let(:legacy) do
    ProtocolDefinition.create!(name: "dengue", version: 1, status: "active", activated_at: 3.days.ago,
                               definition: protocol_definition_hash)
  end

  it "creates one baseline for a version in use with no activation record, dated when it was activated" do
    legacy
    backfill!

    row = legacy.activations.sole
    expect(row).to have_attributes(kind: "baseline", actor_kind: "system", actor_id: nil, reason: nil)
    expect(row.created_at).to be_within(1.second).of(legacy.activated_at)
  end

  it "is idempotent and skips a protocol that already has any activation record" do
    legacy
    backfill!
    backfill!
    ProtocolDefinition.create!(name: "zika", version: 1, status: "active", activated_at: 1.day.ago,
                               definition: protocol_definition_hash(name: "zika"))
                      .activations.create!(kind: "signed", actor_id: SecureRandom.uuid, actor_kind: "user")
    backfill!

    expect(ProtocolActivation.where(kind: "baseline").count).to eq(1)
  end

  it "never creates a baseline for a version that is not in use" do
    ProtocolDefinition.create!(name: "dengue", version: 1, status: "published", definition: protocol_definition_hash)
    backfill!

    expect(ProtocolActivation.count).to eq(0)
  end

  it "refuses a baseline with an actor, a non-baseline without an actor, and a system actor on a signed act" do
    expect { legacy.activations.new(kind: "baseline", actor_kind: "system", actor_id: SecureRandom.uuid).save!(validate: false) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_protocol_activations_/)
  end
end
```

**Atenção ao último exemplo:** cada tentativa que viola uma CHECK aborta a transação de fixture; faça as três violações (baseline com ator; `signed` sem ator; `signed` com `actor_kind: "system"`) cada uma num savepoint, como o `spec/models/protocol_append_only_spec.rb` já faz, e afirme o nome da constraint de cada uma.

Em `spec/commands/protocols_revert_activation_spec.rb`, acrescente:

```ruby
  it "reverts the first signed activation of a city to the baseline version" do
    legacy = ProtocolDefinition.create!(name: "dengue", version: 1, status: "active", activated_at: 3.days.ago,
                                        definition: protocol_definition_hash)
    legacy.activations.create!(kind: "baseline", actor_kind: "system", actor_id: nil, created_at: 3.days.ago)
    activate_signed!(2)

    result = revert

    expect(result.ok?).to be(true)
    expect([ version(1).status, version(2).status ]).to eq(%w[active published])
  end

  it "refuses to revert while the only activation is the baseline" do
    legacy = ProtocolDefinition.create!(name: "dengue", version: 1, status: "active", definition: protocol_definition_hash)
    legacy.activations.create!(kind: "baseline", actor_kind: "system", actor_id: nil)

    expect(revert.reason).to eq(:not_revertible)
  end
```

(`activate_signed!(2)` cria a v2 publicada, assina e ativa — confira que o helper existente aceita uma v1 já `active` criada fora dele; ajuste o helper se preciso, sem mudar o que ele prova nos exemplos antigos.)

- [ ] **Step 2: Rodar e confirmar que falham**

- [ ] **Step 3: Escrever a migração**

`db/city_migrate/20260921000001_add_baseline_protocol_activations.rb`:

```ruby
# Linha-base de ativação (spec de assinaturas §6, decisão do usuário na fatia 2).
#
# Versões em uso antes das assinaturas não têm linha em protocol_activations,
# e sem uma linha ANTERIOR a primeira ativação assinada de cada cidade não
# seria reversível em emergência. Cada uma ganha uma linha `baseline`, de ator
# `system`, datada de quando entrou em uso. Ela só pode ser ALVO de reversão:
# Protocols::RevertActivation exige que a ativação atual seja `signed`.
#
# Irreversível: o trigger de acréscimo recusa DELETE, e desligá-lo para um
# rollback abriria a porta que ele existe para fechar.
class AddBaselineProtocolActivations < ActiveRecord::Migration[8.1]
  BACKFILL_SQL = <<~SQL.freeze
    INSERT INTO protocol_activations (id, protocol_definition_id, kind, actor_id, actor_kind, reason, created_at)
    SELECT gen_random_uuid(), pd.id, 'baseline', NULL, 'system', NULL, COALESCE(pd.activated_at, pd.updated_at)
    FROM protocol_definitions pd
    WHERE pd.status = 'active'
      AND NOT EXISTS (
        SELECT 1 FROM protocol_activations pa
        JOIN protocol_definitions other ON other.id = pa.protocol_definition_id
        WHERE other.name = pd.name
      )
  SQL

  def up
    replace_check :ck_protocol_activations_kind,
                  "kind::text = ANY (ARRAY['signed'::text, 'emergency_revert'::text, 'baseline'::text])"
    replace_check :ck_protocol_activations_actor_kind,
                  "actor_kind::text = ANY (ARRAY['user'::text, 'maintainer'::text, 'system'::text])"
    replace_check :ck_protocol_activations_revert_reason,
                  "kind::text <> 'emergency_revert'::text OR reason IS NOT NULL AND length(btrim(reason)) > 0"

    change_column_null :protocol_activations, :actor_id, true
    add_check_constraint :protocol_activations,
                         "(kind::text = 'baseline'::text) = (actor_kind::text = 'system'::text)",
                         name: "ck_protocol_activations_system_is_baseline"
    add_check_constraint :protocol_activations,
                         "(kind::text = 'baseline'::text) = (actor_id IS NULL)",
                         name: "ck_protocol_activations_baseline_has_no_actor"

    execute BACKFILL_SQL
  end

  def down
    raise ActiveRecord::IrreversibleMigration,
          "linhas-base não se apagam: protocol_activations só aceita acréscimo (trigger rota_append_only)"
  end

  private

  def replace_check(name, expression)
    remove_check_constraint :protocol_activations, name: name.to_s
    add_check_constraint :protocol_activations, expression, name: name.to_s
  end
end
```

**Confira** que as expressões saem do dump exatamente como escritas (o spec de paridade compara); ajuste a forma, não o significado. **Confira** que a CHECK nova de `revert_reason` preserva a regra antiga: `emergency_revert` exige motivo; `signed` e `baseline` não.

- [ ] **Step 4: Modelo, dump, seed**

`ProtocolActivation`:

```ruby
  KINDS = %w[signed emergency_revert baseline].freeze
  # `system` só existe para a linha-base, criada pela migração e pelo seed de
  # dev — nenhum command cria uma.
  ACTOR_KINDS = %w[user maintainer system].freeze

  validates :kind, inclusion: { in: KINDS }
  validates :actor_kind, inclusion: { in: ACTOR_KINDS }
  validates :actor_id, presence: true, unless: -> { kind == "baseline" }
  validates :actor_id, absence: true, if: -> { kind == "baseline" }
  validates :reason, presence: true, if: -> { kind == "emergency_revert" }
```

(atualize o comentário do topo: três tipos; a linha-base só como alvo de reversão).

`db/city_schema.rb`: gere pelo dumper a partir de um banco scratch migrado, versão `2026_09_21_000001`.

`db/seeds.rb`: logo depois de criar `triage-respiratoria` v1 `active`, crie a linha-base se a versão não tiver nenhuma ativação:

```ruby
        # Linha-base (fatia 2 das assinaturas): o protocolo nasce ativo no seed,
        # como uma versão que já estava em uso antes das assinaturas.
        unless protocol.activations.exists?
          protocol.activations.create!(kind: "baseline", actor_kind: "system", actor_id: nil,
                                       created_at: protocol.activated_at || protocol.created_at)
        end
```

- [ ] **Step 5: Recarregar os bancos de teste e migrar as cidades de dev**

```bash
docker compose exec -T -e POSTGRES_PASSWORD=postgres -e RAILS_ENV=test api bin/rails city:test_databases
docker compose exec -T api bin/rails city:migrate:all
```

Registre no relatório a versão de schema de cada cidade de dev ativa e quantas linhas-base cada uma ganhou (`SELECT count(*) FROM protocol_activations WHERE kind = 'baseline'`, leitura).

- [ ] **Step 6: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add db/city_migrate/20260921000001_add_baseline_protocol_activations.rb db/city_schema.rb db/seeds.rb \
  app/models/protocol_activation.rb spec/models/protocol_activation_baseline_spec.rb spec/commands/protocols_revert_activation_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: give every protocol version in use a baseline activation

Versions in use before signatures had no activation record, so the first
signed activation in a city could not be reverted in an emergency. Each
one now gets a baseline, dated when it went into use, that can only be
the target of a revert, never its origin.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 2: A guarda de status sem janela de exclusão

**Files:**
- Modify: `spec/architecture/protocol_signatures_guard_spec.rb`

- [ ] **Step 1: Os casos que a guarda precisa pegar**

A revisão final do Plano 1 mostrou que a exclusão de contextos só-leitura (`where(`, `find_by(`, `exists?(`, `==`, `!=`, `include?`, `scope`) vale para a janela inteira de ±3 linhas, e deixa passar a escrita comum de "ler, conferir, gravar". A guarda precisa pegar, **num arquivo que contém `ProtocolDefinition`, fora da lista permitida**, cada um destes (prove RED com cada um, temporário, revertido):

1. `version = ProtocolDefinition.find_by(name: n)` e, duas linhas abaixo, `version.update!(status: "published")`;
2. `if protocol.status == "in_review"` e, na linha seguinte, `protocol.update!(status: "published")`;
3. `return unless ALLOWED.include?(x)` e, abaixo, `record.update!(status: "active")`;
4. `scope = ProtocolDefinition.where(name: n)` e, abaixo, `scope.first.update!(status: "active")`;
5. `record.update!(activated_at: Time.current, status: "active")` (status fora da primeira posição);
6. `record.update!(status: :published)` (símbolo);
7. `record.update_attribute(:status, "active")`, `record.write_attribute(:status, "active")`, `record[:status] = "active"`;
8. `ProtocolDefinition.where(name: n).update_all(status: "active")` (escrita encadeada a um `where`).

E precisa **continuar verde** no código atual, inclusive em `app/jobs/provision_city_job.rb` (`city.update!(status: "active")` — status de CIDADE) e em qualquer `where(status: "active")` de leitura.

- [ ] **Step 2: Implementar**

Troque a exclusão por janela por uma decisão **por instrução**: para cada linha não comentada com o padrão de escrita, a exclusão só vale se o **próprio trecho da escrita** é leitura — o casamento de `status` está dentro dos parênteses de um `where(`/`find_by(`/`exists?(` que abre na mesma instrução (ande para trás até o `(` não fechado), ou a linha é uma comparação sem escrita. Escrita encadeada (`where(...).update_all(status: ...)`) é escrita. Para distinguir status de cidade de status de protocolo, mantenha o filtro por arquivo (`ProtocolDefinition` no arquivo) e acrescente uma lista **explícita e comentada** de receptores que não são protocolo, se precisar (`city.`, `@city.`, `City.`), em vez da janela de proximidade com "protocol".

- [ ] **Step 3: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add spec/architecture/protocol_signatures_guard_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
test: judge each protocol status write by its own statement

The guard excluded a write whenever a read-only check sat within three
lines, so the ordinary read, check and write pattern slipped through. It
now decides per statement: a status inside a where or find_by is a read,
a chained update_all is a write, and city status writes stay out by
receiver.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 3: Os endpoints do ciclo de vida

**Files:**
- Create: `app/controllers/protocol_lifecycle_controller.rb`, `app/controllers/concerns/protocol_result_rendering.rb`
- Modify: `app/controllers/publications_controller.rb`, `config/routes.rb`, `spec/controllers/publications_controller_spec.rb`
- Test: `spec/requests/protocol_lifecycle_spec.rb`

**Interfaces:**
- Produces (todas no host da cidade, sessão da cidade, corpo JSON):
  - `POST /protocols/:version/submit` `{ name }` → `SubmitForReview` (sem step-up; `ProtocolPolicy#author?` no command)
  - `POST /protocols/:version/signatures` `{ name, purpose }` → `Sign` (step-up)
  - `POST /protocols/:version/publish` `{ name }` → `Publish` (step-up; rota existente, agora com `name`)
  - `POST /protocols/:version/activate` `{ name }` → `Activate` (step-up)
  - `POST /protocols/:version/retire` `{ name }` → `Retire` (step-up)
  - `POST /protocols/revert` `{ name, reason }` → `RevertActivation` (step-up)
  - Sucesso: `200 { ok: true, protocol: { name, version, status } }`; `Sign` devolve também `signature: { purpose, created_at }`
  - `ProtocolResultRendering#render_protocol_result(result)` — o mapeamento único de `Result` para HTTP

- [ ] **Step 1: Escrever o spec que falha**

`spec/requests/protocol_lifecycle_spec.rb` — `use_test_city_host!`, usuários criados no banco de `TEST_CITY_A`, `sign_in_as(user)`; para o step-up, carimbe `mfa_verified_at` na sessão criada (`session.update!(mfa_verified_at: Time.current)`) e dê ao usuário um TOTP matriculado (confira em `User#mfa_enrolled?` o que é preciso). Cubra, com valores:

1. **Fluxo completo com as regras reais:** autor salva (via command) e envia (`POST /protocols/1/submit`) → `in_review`; dois revisores assinam publicação (`POST /protocols/1/signatures`, `purpose: "publication"`) → um publisher publica → `published`; os mesmos revisores assinam ativação → publisher ativa → `active`. Afirme o status no banco a cada passo e o corpo `{ ok: true, protocol: {...} }`.
2. **Step-up:** assinar, publicar, ativar, aposentar e reverter **sem** `mfa_verified_at` recente → `401 { error: "mfa_required" }`, e nada muda no banco; enviar para revisão sem step-up funciona.
3. **Regras aparecem como 422 com a mensagem do command:** publicar com uma assinatura → `422 { error: "signatures_missing", message: "falta 1 assinatura de publicação; revisores elegíveis na cidade: …" }`; o autor tentando assinar a própria versão → `422 contributor_cannot_sign`; ativar um rascunho → `422 not_published`; aposentar a ativa → `422 active_in_city`; reverter sem motivo → `422 reason_required`.
4. **Autorização:** assinar sem `protocol_reviewer` → `403 { error: "forbidden" }`; versão inexistente → 404.
5. **Reversão:** com linha-base (Task 1) + uma ativação assinada, `POST /protocols/revert` com motivo volta à versão anterior; o motivo aparece na linha `emergency_revert`.
6. **Sessão de operador (grant):** todo endpoint novo responde `403 { error: "operator_read_only" }` (use o helper/fixture que os specs de `operator_city_session_spec.rb` já usam para abrir uma sessão de grant).
7. **Nenhum dado de cidadão** no corpo de nenhuma resposta (`phone`, `wa_id`, conteúdo de mensagem).

- [ ] **Step 2: Rodar e confirmar que falha**

- [ ] **Step 3: O mapeamento único e o controller**

`app/controllers/concerns/protocol_result_rendering.rb`:

```ruby
# Um só lugar traduz o Result de um command de protocolo para HTTP (fatia 2
# das assinaturas). A mensagem é a do próprio command — texto nosso, com a
# contagem de assinaturas que faltam —, nunca a de uma exceção.
module ProtocolResultRendering
  STATUS_FOR = { forbidden: :forbidden, not_found: :not_found }.freeze

  private

  def render_protocol_result(result)
    if result.ok?
      protocol = result.payload[:protocol_definition]
      render json: { ok: true, protocol: { name: protocol.name, version: protocol.version, status: protocol.status } }
    else
      render json: { error: result.reason.to_s, message: result.message }.compact,
             status: STATUS_FOR.fetch(result.reason, :unprocessable_entity)
    end
  end
end
```

`app/controllers/protocol_lifecycle_controller.rb`:

```ruby
# Ciclo de vida do protocolo na cidade, com assinaturas (ADR 0016, spec de
# assinaturas §4 e §8). Endpoints finos: a regra mora nos commands.
#
# Step-up de MFA (ADR 0011) em todo ato que aprova ou põe em uso — assinar,
# ativar, aposentar, reverter; a publicação mora em PublicationsController.
# Enviar para revisão não aprova nada e não pede step-up.
#
# Sessão de operador (grant) é só leitura: nenhuma ação aqui entra em
# allow_operator_grant_access.
class ProtocolLifecycleController < ApplicationController
  include Authentication
  include MfaStepUp
  include ProtocolResultRendering

  before_action :require_step_up!, except: :submit

  def submit
    render_protocol_result(Protocols::SubmitForReview.call(name: protocol_name, version: version, by: Current.user))
  end

  def sign
    result = Protocols::Sign.call(name: protocol_name, version: version, purpose: params.require(:purpose),
                                  by: Current.user)
    return render_protocol_result(result) unless result.ok?

    signature = result.payload[:signature]
    render json: { ok: true,
                   protocol: { name: signature.protocol_definition.name, version: signature.protocol_definition.version,
                               status: signature.protocol_definition.status },
                   signature: { purpose: signature.purpose, created_at: signature.created_at.iso8601 } }
  end

  def activate
    render_protocol_result(Protocols::Activate.call(version: version, name: protocol_name, by: Current.user))
  end

  def retire
    render_protocol_result(Protocols::Retire.call(version: version, name: protocol_name, by: Current.user))
  end

  def revert
    render_protocol_result(Protocols::RevertActivation.call(name: protocol_name, reason: params[:reason].to_s,
                                                            by: Current.user))
  end

  private

  def protocol_name = params.require(:name)
  def version = Integer(params.require(:version))
end
```

**Confira:** que `require_step_up!` responde **antes** de qualquer leitura do protocolo (a recusa não revela se a versão existe); que `Integer(...)` com lixo vira 404 ou 422 limpo (rota com `constraints: { version: /\d+/ }` resolve isso na borda); o que `Authentication` exige para negar a sessão de grant (ele já nega por padrão — confirme com o spec).

`PublicationsController#create`: passe `name: params[:name]` ao `Publish` e troque o `case` local pelo `render_protocol_result` (mantendo o comportamento de hoje para o dashboard: 404, 403 e 422 com `error`/`message`). Atualize `spec/controllers/publications_controller_spec.rb` para o novo formato de sucesso **sem** afrouxar nenhuma expectativa; se o formato de sucesso mudar para o dashboard, diga no relatório.

`config/routes.rb`, junto da rota de publicação:

```ruby
  # Ciclo de vida com assinaturas (ADR 0016). `name` vai no corpo.
  constraints(version: /\d+/) do
    post "/protocols/:version/submit",     to: "protocol_lifecycle#submit"
    post "/protocols/:version/signatures", to: "protocol_lifecycle#sign"
    post "/protocols/:version/activate",   to: "protocol_lifecycle#activate"
    post "/protocols/:version/retire",     to: "protocol_lifecycle#retire"
  end
  post "/protocols/revert", to: "protocol_lifecycle#revert"
```

**Confira** que nenhuma rota nova é engolida por `scope "/protocols"` (`get ":name"` só casa GET) nem pelas rotas de manutenção/console (que ficam antes, restritas por host).

- [ ] **Step 4: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/controllers/protocol_lifecycle_controller.rb app/controllers/concerns/protocol_result_rendering.rb \
  app/controllers/publications_controller.rb config/routes.rb spec/requests/protocol_lifecycle_spec.rb \
  spec/controllers/publications_controller_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: expose the signed protocol lifecycle on the city API

Submit for review, sign, publish, activate, retire and emergency revert,
each a thin endpoint over its command, with the city's MFA step-up on
every act that approves or puts a version in use. Domain refusals come
back as 422 with the command's own message, so the panel can say how
many signatures are missing.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 4: Conceder papel pela API

**Files:**
- Modify: `app/controllers/setup_controller.rb`, `config/routes.rb`
- Test: `spec/requests/setup_grant_role_spec.rb`

**Interfaces:**
- Produces: `POST /setup/memberships` `{ user_id, role }` → `GrantRole` → `201 { id, user_id, role, granted_at }` | `403` (não é `municipal_admin`) | `422 { error, message }`.

- [ ] **Step 1: Escrever o spec que falha**

`spec/requests/setup_grant_role_spec.rb`:
- um `municipal_admin` torna um publisher também revisor → 201, e `User#has_role?(:protocol_reviewer)` passa a ser verdadeiro; o evento `membership.granted` existe na cidade;
- um publisher (não admin) → 403, nada criado;
- papel desconhecido → 422 `invalid_role`; papel já concedido → 422 `already_granted`; usuário inexistente → 422 `user_not_found`;
- revogar o papel concedido pela rota existente (`POST /setup/memberships/:id/revoke`) funciona e o revisor deixa de contar em `Protocols::Signatures.eligible_reviewer_count`;
- sessão de grant de operador → 403 `operator_read_only`.

- [ ] **Step 2: Rodar e confirmar que falha**

- [ ] **Step 3: Implementar**

Em `SetupController`:

```ruby
  # POST /setup/memberships
  # body: { user_id, role } — só municipal_admin (spec de assinaturas §3). O
  # command recusa o resto: papel desconhecido, já concedido, usuário inativo.
  def grant_role
    return head(:forbidden) unless can_manage_members?

    result = GrantRole.call(user_id: params.require(:user_id), role: params.require(:role), by: current_user)
    if result.ok?
      m = result.payload[:membership]
      render json: { id: m.id, user_id: m.user_id, role: m.role, granted_at: m.granted_at.iso8601 }, status: :created
    else
      render json: { error: result.reason.to_s, message: result.message }.compact,
             status: result.reason == :forbidden ? :forbidden : :unprocessable_entity
    end
  end
```

Rota: `post "/memberships", to: "setup#grant_role"` dentro de `scope "/setup"`.

- [ ] **Step 4: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/controllers/setup_controller.rb config/routes.rb spec/requests/setup_grant_role_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: let a municipal admin grant a role through the city API

A reviewer is usually someone who is already a publisher: the admin
grants the role to the existing user instead of inviting them again, and
the existing revoke route takes it back.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 5: O painel lê o estado das assinaturas

**Files:**
- Modify: `app/queries/admin/protocols_query.rb`
- Test: `spec/requests/admin/protocols_signatures_spec.rb`

**Interfaces:**
- Produces: em cada versão de `GET /admin/api/protocols` e `/admin/api/protocols/:id`, além dos campos atuais:
  - `signatures: { publication: { signers: [{ id, email }], missing: Integer }, activation: { signers: [...], missing: Integer } }` — só as assinaturas **válidas agora** (`Protocols::Signatures.valid_signer_ids`)
  - `eligibleReviewers: Integer` (`Protocols::Signatures.eligible_reviewer_count`)
  - `editors: [{ kind: "user" | "maintainer", id, email }]` — `email` só para `user` (o mantenedor mora na plataforma; `null` para ele)
  - `revertible: Boolean` — verdadeiro só na versão `active` cuja última ativação do protocolo é `signed` e há uma ativação anterior cuja versão está `published`

- [ ] **Step 1: Escrever o spec que falha**

`spec/requests/admin/protocols_signatures_spec.rb` (sessão de `municipal_admin` na cidade, como os outros specs de `spec/requests/admin/`):
- uma versão em `in_review` com uma assinatura de publicação válida e uma de conteúdo antigo → `signers` lista só a válida, `missing: 1`; `eligibleReviewers` bate com os revisores criados que não editaram;
- `editors` lista o autor (com e-mail) e um mantenedor (`kind: "maintainer"`, `email: null`);
- `revertible` é falso só com a linha-base e verdadeiro depois de uma ativação assinada sobre ela;
- os campos antigos (`createdBy`, `publishedBy`, `fourEyes`) continuam presentes e com o mesmo significado.
- nenhuma consulta a `domain_events` é usada para os campos novos (afirme por `expect(DomainEvent).not_to receive(:where)` só no bloco que monta os campos novos, ou isole a montagem num método e teste-o).

- [ ] **Step 2: Rodar e confirmar que falha**

- [ ] **Step 3: Implementar**

Em `Admin::ProtocolsQuery`, um método `signature_state(d)` montado a partir de `Protocols::Signatures`, `ProtocolContribution`, `ProtocolActivation` e `User` (e-mails em lote, sem N+1 por assinatura), incorporado a `serialize_row` e `serialize_version`. Deixe `fetch_audit`/`four_eyes` como estão (Decisão 5), com um comentário dizendo que os campos novos são a fonte de verdade das assinaturas.

**Atenção à contagem de conexões:** tudo isto roda na cidade corrente, no mesmo request; não abra conexão com outra cidade.

- [ ] **Step 4: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/queries/admin/protocols_query.rb spec/requests/admin/protocols_signatures_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: show each protocol version's signature state in the city panel

Per version, the panel reads who signed validly for publication and for
activation, how many signatures are missing, how many reviewers are
eligible, who edited it and whether an emergency revert is possible —
all from the signature tables, never from events.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 6: Spec e ADR (repo `docs`)

**Files:**
- Modify: `superpowers/specs/2026-09-18-protocol-signatures-design.md`, `adr/0016.md`

- [ ] **Step 1: Atualizar**

- **Spec §5** (tabela `protocol_activations`): `kind` passa a `signed | emergency_revert | baseline`; `actor_kind` ganha `system` (só na linha-base); `actor_id` nulo só na linha-base.
- **Spec §6:** novo parágrafo "Linha-base": cada versão em uso antes das assinaturas ganha uma linha `baseline` datada de quando entrou em uso; ela só pode ser **alvo** de reversão; a primeira ativação assinada de cada cidade passa a ser reversível; a migração é irreversível pelo mesmo motivo que as tabelas só aceitam acréscimo.
- **Spec §8:** a tabela de superfícies ganha as rotas da fatia 2 (`/protocols/:version/submit|signatures|publish|activate|retire`, `/protocols/revert`, `POST /setup/memberships`) e a nota "step-up em assinar, publicar, ativar, aposentar e reverter".
- **ADR 0016:** a reversão de emergência passa a valer desde a primeira ativação assinada, por causa da linha-base; o texto segue a convenção do corpus (regra atual, sem nota de emenda).

- [ ] **Step 2: Commit no repo `docs`**

```bash
cd docs
/opt/homebrew/bin/git add superpowers/specs/2026-09-18-protocol-signatures-design.md adr/0016.md
/opt/homebrew/bin/git commit -F - <<'EOF'
docs: record the baseline activation and the city API of protocol signatures

Versions in use before signatures get a baseline activation, so the
first signed activation of a city can be reverted in an emergency, and
the spec lists the city endpoints and which of them require the step-up.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

## Verificação final do plano

- [ ] Suíte completa com o worker parado: 0 falhas.
- [ ] Bancos de teste e cidades de dev na versão `20260921000001`; cada cidade de dev com uma linha-base para `triage-respiratoria` v1.
- [ ] Pela API da cidade: enviar → duas assinaturas de publicação → publicar → duas de ativação → ativar → reverter para a linha-base, com step-up onde a spec pede e 401 `mfa_required` sem ele.
- [ ] Sessão de operador recusada em toda escrita nova.
- [ ] O painel lê assinaturas válidas, quantas faltam, revisores elegíveis, editores e se é reversível — das tabelas.
- [ ] A guarda de status pega os oito casos da Task 2 e continua verde no código atual.

## Fora deste plano

- **Telas do dashboard** para enviar, assinar e acompanhar: spec própria no repo do dashboard.
- **Plano 5 da API de manutenção**, reescrito sobre as assinaturas.
- **Recusa formal e retirada de assinatura** (spec §12).
