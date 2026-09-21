# Plano 5 — Escrita em cidade: protocolos (fatia 5a)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** o mantenedor passa a escrever dentro de uma cidade pela API de manutenção, começando pelo ciclo de vida de protocolo — salvar rascunho, enviar para revisão, publicar, ativar, aposentar e reverter em emergência —, com a auditoria em três passos da spec §9, **sem nunca assinar** e sem contornar nenhuma regra de domínio.

**Architecture:** uma base `Maintenance::Mutations::CityMutation` é o caminho único da escrita: escopo do token → tentativa auditada na plataforma → `Maintenance::CityWriter` abre a cidade e roda o command com `Maintenance::MaintainerActor` → resultado auditado com o mesmo `correlation_id`, que o command grava no evento de domínio da cidade. As regras (duas assinaturas de revisores da cidade para publicar e para ativar, R1, R4, portão, reversão estreita) já estão nos commands; a API só as alcança.

**Tech Stack:** Rails 8.1.3, `graphql-ruby` 2.6, Postgres, RSpec.

**Spec:** `docs/superpowers/specs/2026-09-17-maintenance-graphql-api-design.md` (§2 D6, §7, §8, §9, §10, §11 fatia 5) e `docs/superpowers/specs/2026-09-18-protocol-signatures-design.md` (§4, §7, §8). ADR 0016.

**Substitui** o rascunho `2026-09-18-maintenance-api-05-escrita-e-protocolos.md`, escrito antes das assinaturas e nunca commitado.

**Baseline de entrada:** `main` do api em `9bad950`, **1319 exemplos, 0 falhas**, suíte em ~2,5 min.

## Decisões minhas na escrita (cada uma com o custo se estiver errada)

1. **Nenhuma mutation de assinatura.** O mantenedor nunca assina (spec de assinaturas §7; `Protocols::Sign` recusa `actor_kind != "user"`). Uma guarda da Task 5 falha se alguma mutation chamar `Protocols::Sign` ou `GrantRole`/`InviteMember` com papel privilegiado. **Custo se errado:** nenhum — é regra do usuário.
2. **Step-up de TOTP em publicar, ativar, aposentar e reverter; não em salvar rascunho nem em enviar para revisão.** Espelha a API da cidade (fatia 2): o step-up guarda os atos que aprovam ou põem em uso. **Custo se errado:** um mantenedor com sessão sequestrada consegue editar rascunhos — que ninguém assina por ele.
3. **Todas as mutations de protocolo são só de sessão humana** (`HumanOnly::RESTRICTED`). Escrita por token de serviço é decisão própria (spec da API §7). **Custo se errado:** uma automação que queira salvar rascunho fica sem caminho.
4. **O motivo da reversão fica na cidade, não na auditoria de plataforma.** O command grava o motivo na linha `emergency_revert` e no evento de domínio da cidade; a auditoria de manutenção leva só `reason_given: true`. O motivo é texto livre, e `platform_events` é governado pela Ruling R18 (nada de conteúdo livre que possa carregar dado pessoal). **Custo se errado:** quem investiga pela plataforma precisa do `correlation_id` para achar o motivo na cidade.
5. **`definition` entra como `JSON!` por variável** (`GraphQL::Types::JSON`), sujeita ao teto de 64 KB do corpo. **Custo se errado:** um protocolo acima de 64 KB não sai pela API — o editor do dashboard continua sendo o caminho de autoria.
6. **Fecha o resíduo do Plano 4 aqui (Task 1):** `CityReader` distingue falha de conexão (mensagem redigida, `CITY_UNREACHABLE`) de qualquer outra falha (só a classe, `CITY_READ_FAILED`), com uma lista de classes de conexão que o `CityWriter` também usa. **Custo se errado:** diagnóstico de bug de leitura passa a exigir o log.
7. **Uma mutation de cidade conta no teto de 5 cidades por operação** (`CityBudget`). **Custo se errado:** ler 5 cidades e escrever em uma exige duas chamadas.
8. **Escrita só em cidade `active`** (`City#servable?`); suspensa, arquivada e em provisionamento recusam como erro de usuário, sem conectar. **Custo se errado:** manutenção em cidade suspensa continua exigindo o console.

## Global Constraints

Valem para TODA task:

- **O mantenedor nunca assina, nunca concede nem convida `protocol_reviewer`/`municipal_admin`.** Nenhuma mutation deste plano chama `Protocols::Sign`, `GrantRole` ou `InviteMember`.
- **Toda escrita passa por um command** (`Protocols::SaveDraft`, `SubmitForReview`, `Publish`, `Activate`, `Retire`, `RevertActivation`), com `by: Maintenance::MaintainerActor.new(maintainer)` e `correlation_id:`. Nenhum `update!` em resolver.
- **Auditoria em três passos (spec §9):** tentativa gravada ANTES; se a gravação falhar, o command não roda; resultado `ok`/`rejected`/`error` com o mesmo `correlation_id`; `changed_fields` nunca carrega valores.
- **Ruling R18** para todo evento novo: nome em `MaintenanceAudit::NAMES`, no `case` de `MaintenanceAudit.record` e em `R18_PLATFORM_EVENT_NAMES`; nenhuma chave com `email`, `name`, `phone`, `cpf`, `body`, `wa_id`, `provider_uid`, `from` — por isso o nome do protocolo entra como `protocol_key`.
- **Nunca expor segredo nem conteúdo de cidadão; mensagem de exceção sai só pela classe**, salvo falha de conexão, cuja mensagem passa por `CitySchema.redact`.
- **Leitura não é auditada.**
- **Commits em inglês, Conventional Commits**, terminando com a linha exata `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>` — copiada literalmente, **nunca** o nome do próprio modelo.
- **Comandos Ruby/rspec no container**, da raiz do monorepo: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Suíte completa uma vez por task, em primeiro plano, `docker compose stop worker` antes e `start` depois. **A suíte leva ~2,5 min; acima de ~3 min é regressão — rode `--profile` e ache o custo fixo antes de seguir.**
- **Specs não definem constante no topo**, afirmam valores contra linhas reais, sem `skip` como caminho esperado. Cidade inalcançável se simula com stub, nunca com URL real. Specs de mutation fazem o login com `travel_to(1.minute.ago) { login! }` (o código do login é consumido).
- **Não mexa** no `Gemfile`, em `app/services/city_inventory.rb`, nos frontends, nem nos commands de protocolo (só os chame).
- **`git` do PATH está quebrado:** use `/opt/homebrew/bin/git`, a partir de `apps/api`. **Nunca dar push.** Branch `feat/maintenance-protocol-writes`, criada de `main`.

## Fatos verificados (confira mesmo assim)

- **`Maintenance::MaintainerActor.new(maintainer)`** (`app/services/maintenance/maintainer_actor.rb`): `id`, `actor_kind = "maintainer"`, `has_role?(_) = true`.
- **Commands** (todos exigem `Current.city`, devolvem `Result`): `Protocols::SaveDraft.call(definition:, by:, correlation_id: nil)` (`:forbidden`, `:version_not_editable`, `:invalid_definition`); `SubmitForReview.call(name:, version:, by:, correlation_id: nil)` (`:not_found`, `:forbidden`, `:invalid_state`, `:invalid`); `Publish.call(version:, by:, name: nil, correlation_id: nil)` (`:signatures_missing` com mensagem "falta N assinatura(s) de publicação; revisores elegíveis na cidade: M", `:invalid_state`, `:invalid`, `:ambiguous`); `Activate.call(version:, by:, name: nil, correlation_id: nil)` (`:signatures_missing`, `:not_published`); `Retire.call(version:, by:, name: nil, correlation_id: nil)` (`:active_in_city`); `RevertActivation.call(name:, by:, reason:, correlation_id: nil)` (`:reason_required`, `:not_revertible`, `:no_previous_activation`). Todos travam a linha do protocolo e reconferem sob a trava.
- **`BaseMutation`** (`app/graphql/maintenance/mutations/base_mutation.rb`): `audited(event:, module_name:, **fields)` grava tentativa e resultado com o mesmo `correlation_id`, mas faz `result = yield` **sem** entregar o `correlation_id` ao bloco; `Rejected.new(message, path:)` vira `{ ok: false, errors: [...] }`; `step_up!(code)` consome o TOTP, conta falha para o bloqueio, recusa conta bloqueada.
- **`CityReader`** (`app/queries/maintenance/city_reader.rb`): `Archived`, `Unreachable`; o `rescue StandardError` transforma **qualquer** exceção em `Unreachable` com a mensagem redigida (resíduo do Plano 4). `CityType#inside` converte `Archived`/`Unreachable` em erros de campo `CITY_ARCHIVED`/`CITY_UNREACHABLE`.
- **Analisadores:** `WriteScope` recusa mutation de token `read`; `HumanOnly::RESTRICTED = %w[maintainers maintenanceTokens auditEvents inviteMaintainer deactivateMaintainer createMaintenanceToken revokeMaintenanceToken]`, `TOKEN_ALLOWED = %w[me cities city]`, e a guarda de cobertura exige todo campo de raiz em uma das listas; `CityBudget::MAX_CITIES = 5` conta campos `city` de `Query`; `Refusal::CODES` inclui `CITY_OUT_OF_SCOPE`.
- **Guardas a atualizar, não afrouxar:** em `spec/graphql/maintenance/analyzers_spec.rb`, `"routes every city-scoped root field through the credential's city scope"`; em `spec/architecture/maintenance_schema_spec.rb`, a guarda do caminho único (pares arquivo → chamada permitida de `CityConnection.with`/`CityReader.call` em `app/{graphql,queries,services}/maintenance/**`, proíbe `CityInventory`), `EXPECTED_TYPES` e as guardas de conteúdo; `spec/architecture/protocol_signatures_guard_spec.rb` (nenhuma escrita de status fora dos três commands).
- **`MaintenanceAudit::NAMES`** tem hoje `maintenance.session.*`, `maintenance.maintainer.*`, `maintenance.token.*`.
- **Helpers de protocolo em spec:** `protocol_definition_hash(name:, version:)`, `make_reviewer!`, `sign!(protocol, purpose:, by:)`.
- **O frontend** (`apps/maintenance`) tem `schema.graphql` commitado; mutations novas no api não o quebram (o `codegen:check` compara documentos com o schema commitado). Telas de escrita no frontend ficam fora deste plano.

---

## File Structure

**apps/api**
- Create: `app/queries/maintenance/city_connection_errors.rb`, `app/queries/maintenance/city_writer.rb`, `app/graphql/maintenance/mutations/city_mutation.rb`, `app/graphql/maintenance/mutations/save_protocol_draft.rb`, `submit_protocol_for_review.rb`, `publish_protocol.rb`, `activate_protocol.rb`, `retire_protocol.rb`, `revert_protocol_activation.rb`
- Create (specs): `spec/queries/maintenance/city_writer_spec.rb`, `spec/requests/maintenance/protocol_mutations_spec.rb`
- Modify: `app/queries/maintenance/city_reader.rb`, `app/graphql/maintenance/types/city_type.rb`, `app/graphql/maintenance/mutations/base_mutation.rb`, `app/graphql/maintenance/types/mutation_type.rb`, `app/graphql/maintenance/analyzers/city_budget.rb`, `app/graphql/maintenance/analyzers/human_only.rb`, `app/events/maintenance_audit.rb`
- Modify (specs): `spec/queries/maintenance/city_reader_spec.rb`, `spec/requests/maintenance/city_spec.rb`, `spec/graphql/maintenance/analyzers_spec.rb`, `spec/architecture/maintenance_schema_spec.rb`, `spec/events/platform_event_payload_guard_spec.rb`

**docs**
- Modify: `superpowers/specs/2026-09-17-maintenance-graphql-api-design.md` (§8, §11)

---

### Task 1: Falha de conexão ≠ qualquer outra falha (resíduo do Plano 4)

**Files:**
- Create: `app/queries/maintenance/city_connection_errors.rb`
- Modify: `app/queries/maintenance/city_reader.rb`, `app/graphql/maintenance/types/city_type.rb`
- Test: `spec/queries/maintenance/city_reader_spec.rb`, `spec/requests/maintenance/city_spec.rb`

**Interfaces:**
- Produces: `Maintenance::CityConnectionErrors::CLASSES`; `Maintenance::CityReader::Failed` (mensagem = só o nome da classe); `CityType#inside` converte `Failed` em `extensions.code = "CITY_READ_FAILED"`.

- [ ] **Step 1: Escrever os specs que falham**

Em `spec/queries/maintenance/city_reader_spec.rb` (reaproveite o arranjo do arquivo):

```ruby
  it "reports a non-connection failure by its class only, never its message" do
    expect {
      described_class.call(city) { raise ArgumentError, "telefone +55 41 99999-0000 inválido" }
    }.to raise_error(described_class::Failed) { |e|
      expect(e.message).to eq("ArgumentError")
      expect(e.message).not_to include("99999")
    }
  end

  it "logs a non-connection failure by its class, without the raw message" do
    logged = []
    allow(Rails.logger).to receive(:warn) { |msg| logged << msg }

    expect { described_class.call(city) { raise ArgumentError, "telefone +55 41 99999-0000" } }
      .to raise_error(described_class::Failed)

    expect(logged.join).to include("ArgumentError")
    expect(logged.join).not_to include("99999")
  end

  it "still treats every class in CityConnectionErrors as unreachable, with a redacted message" do
    Maintenance::CityConnectionErrors::CLASSES.each do |klass|
      allow(CityConnection).to receive(:with)
        .and_raise(klass.new("connection to postgres://rota_city_x:s3nha@db:5432/x failed"))

      expect { described_class.call(city) { :never } }
        .to raise_error(described_class::Unreachable) { |e| expect(e.message).not_to include("s3nha") }
    end
  end
```

(Se alguma classe da lista não aceitar só a mensagem no construtor, construa-a como ela exige e diga no relatório.)

Em `spec/requests/maintenance/city_spec.rb`: com `Maintenance::CityReader` stubado para levantar `Failed, "ArgumentError"` numa cidade, o campo `profile` dela sai nulo com `CITY_READ_FAILED` e mensagem `"ArgumentError"`, e a outra cidade responde.

- [ ] **Step 2: Rodar e confirmar que falham**

- [ ] **Step 3: Implementar**

`app/queries/maintenance/city_connection_errors.rb`:

```ruby
# O que conta como "a cidade não respondeu" na API de manutenção — leitura e
# escrita. Só estas classes publicam a mensagem (redigida por
# CitySchema.redact): é ela que diz QUAL host e banco falharam, e ela traz a
# URL com a senha do role. Qualquer outra exceção é texto sem controle (um bug
# pode interpolar dado de cidadão) e sai só pelo nome da classe.
module Maintenance
  module CityConnectionErrors
    CLASSES = [
      ActiveRecord::ConnectionNotEstablished,
      ActiveRecord::ConnectionTimeoutError,
      PG::ConnectionBad,
      CityConnection::InvalidCityDatabase
    ].freeze
  end
end
```

Confira a hierarquia no Rails 8.1 e mantenha a lista **mínima e explícita** — nada de `ActiveRecord::StatementInvalid` (cobre qualquer erro de SQL) nem `ActiveRecord::QueryCanceled` (timeout de consulta não é conexão).

Em `CityReader.call`, troque o `rescue StandardError` por:

```ruby
    rescue *CityConnectionErrors::CLASSES => e
      redacted = CitySchema.redact(e.message)
      Rails.logger.warn("Maintenance::CityReader: #{e.class}: #{redacted}")
      raise Unreachable, "#{e.class}: #{redacted}"
    rescue StandardError => e
      Rails.logger.warn("Maintenance::CityReader: #{e.class} (mensagem omitida)")
      raise Failed, e.class.name
```

com `class Failed < StandardError; end` e o comentário do topo descrevendo as três saídas. Em `CityType#inside`:

```ruby
      rescue CityReader::Failed => e
        raise GraphQL::ExecutionError.new(e.message, extensions: { "code" => "CITY_READ_FAILED" })
```

- [ ] **Step 4: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/queries/maintenance/city_connection_errors.rb app/queries/maintenance/city_reader.rb \
  app/graphql/maintenance/types/city_type.rb spec/queries/maintenance/city_reader_spec.rb spec/requests/maintenance/city_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
fix: publish only the class of a non-connection city read failure

Only a connection failure's message is worth showing — and redacting. Any
other exception message is uncontrolled text that can carry citizen data,
so it now leaves as CITY_READ_FAILED with the class name alone, in the
response and in the log.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 2: O caminho da escrita e `saveProtocolDraft`

**Files:**
- Create: `app/queries/maintenance/city_writer.rb`, `app/graphql/maintenance/mutations/city_mutation.rb`, `app/graphql/maintenance/mutations/save_protocol_draft.rb`
- Modify: `app/graphql/maintenance/mutations/base_mutation.rb`, `app/graphql/maintenance/types/mutation_type.rb`, `app/graphql/maintenance/analyzers/city_budget.rb`, `app/graphql/maintenance/analyzers/human_only.rb`, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/graphql/maintenance/analyzers_spec.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/queries/maintenance/city_writer_spec.rb`, `spec/requests/maintenance/protocol_mutations_spec.rb`

**Interfaces:**
- Consumes: `CityConnectionErrors::CLASSES` (Task 1); `MaintainerActor`; `Protocols::SaveDraft`.
- Produces:
  - `Maintenance::CityWriter.call(city) { ... }` — recusa cidade não `active` com `NotWritable` (sem conectar); falha de conexão → `Unreachable` (mensagem redigida); qualquer outra exceção sobe intacta
  - `BaseMutation#audited` entrega o `correlation_id` ao bloco (`yield(correlation_id)`); as mutations existentes seguem iguais
  - `Maintenance::Mutations::CityMutation < BaseMutation` com `argument :city_slug`, `field :ok`, `field :errors`, e `in_city(city_slug:, event:, module_name:, **fields) { |actor, correlation_id| Result }`
  - `saveProtocolDraft(citySlug: String!, definition: JSON!)` → `{ ok, errors }`; evento `maintenance.protocol.draft_saved` com `city_slug`, `protocol_key`, `version`, `changed_fields: ["definition"]`
  - `CityBudget` conta campos de raiz de `Mutation` com argumento `citySlug`

- [ ] **Step 1: Escrever os specs que falham**

`spec/queries/maintenance/city_writer_spec.rb`:

```ruby
require "rails_helper"

# A escrita entra no banco da cidade por UM lugar. Diferente da leitura, uma
# exceção de dentro do command NÃO vira "cidade inalcançável": ela sobe, e a
# mutation a registra como `error`. Só falha de conexão vira Unreachable.
RSpec.describe Maintenance::CityWriter do
  # Registre a City de TEST_CITY_A como spec/queries/maintenance/city_reader_spec.rb faz.
  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }

  it "runs the block inside the city's connection and returns its value" do
    expect(CityConnection).to receive(:with).with(city).and_call_original

    expect(described_class.call(city) { Current.city.slug }).to eq(city.slug)
  end

  it "refuses a city that is not active, without connecting" do
    %w[suspended archived provisioning].each do |status|
      other = City.new(slug: "w-#{status}", name: "W", status: status,
                       database_url: "postgres://u:p@invalid.invalid:5432/x", encryption_key: SecureRandom.hex(32))
      expect(CityConnection).not_to receive(:with)

      expect { described_class.call(other) { :never } }.to raise_error(described_class::NotWritable, /#{status}/)
    end
  end

  it "turns a connection failure into Unreachable with a redacted message" do
    allow(CityConnection).to receive(:with)
      .and_raise(PG::ConnectionBad.new("connection to postgres://rota_city_x:s3nha@db:5432/x failed"))

    expect { described_class.call(city) { :never } }
      .to raise_error(described_class::Unreachable) { |e| expect(e.message).not_to include("s3nha") }
  end

  it "lets any other exception raised by the block go up untouched" do
    expect { described_class.call(city) { raise ArgumentError, "bug" } }.to raise_error(ArgumentError, "bug")
  end
end
```

`spec/requests/maintenance/protocol_mutations_spec.rb` — arranjo de login como `spec/requests/maintenance/maintainer_mutations_spec.rb` (`travel_to(1.minute.ago) { login! }`), cidades como `spec/requests/maintenance/city_spec.rb` (inclui uma arquivada com URL não roteável). Consultas em métodos, nunca em constante. Nesta task, os exemplos de `saveProtocolDraft`:

1. **Salva e audita:** a mutation com `definition: protocol_definition_hash` (por variável) → `{ ok: true, errors: [] }`; a versão existe em `draft` no banco da cidade; há uma `ProtocolContribution` com `actor_kind: "maintainer"` e `actor_id` do mantenedor; a auditoria tem `attempted` e `ok` com o mesmo `correlation_id`, e `city_slug`, `protocol_key`, `version`, `changed_fields: ["definition"]`, `maintainer_id`; o `DomainEvent` `protocol.draft_saved` carrega `actor_kind: "maintainer"` e esse `correlation_id`.
2. **Recusa do domínio vira erro de usuário e `rejected`:** salvar sobre uma versão `published` → `ok: false`, erro no caminho `definition`, auditoria `rejected`, nada muda.
3. **Cidade inexistente e cidade não ativa:** erro no caminho `citySlug`, auditoria `rejected`, sem conectar.
4. **Tentativa que não pode ser gravada não roda o command:** stub de `MaintenanceAudit.record` levantando no `attempted` → `Protocols::SaveDraft` não é chamado, nada muda.
5. **Cidade inalcançável:** `CityWriter` stubado levantando `Unreachable` → erro GraphQL `CITY_UNREACHABLE`, auditoria `error`.
6. **Qualquer outra falha:** `Protocols::SaveDraft` stubado levantando `ArgumentError, "telefone +55 41 99999-0000"` → `CITY_WRITE_FAILED`, o marcador não aparece em `response.body`, auditoria `error`.
7. **Token de serviço** (`read_write`, sem cidades) → recusado antes de executar (`TOKEN_SCOPE_REFUSED`), nada muda.
8. **Nenhum valor na auditoria:** nenhum evento `maintenance.protocol.*` carrega a `definition` nem pedaço dela.

- [ ] **Step 2: Rodar e confirmar que falham**

- [ ] **Step 3: `CityWriter`**

`app/queries/maintenance/city_writer.rb`:

```ruby
# Entrada única da ESCRITA no banco de uma cidade, para a API de manutenção.
#
# Irmão de CityReader, com uma diferença de propósito: na leitura, qualquer
# falha vira erro de campo; na escrita, uma exceção de dentro do command é um
# RESULTADO — `error` na auditoria — e não pode ser confundida com "a cidade não
# respondeu". Só falha de conexão vira Unreachable. Uma conexão que cai NO MEIO
# do command também vira Unreachable: o command pode ou não ter comitado — é o
# "resultado desconhecido" da spec §9, e o correlation_id no evento de domínio
# da cidade é o que permite conferir.
#
# Só cidade `active` recebe escrita (CityLifecycle::SuspensionGuard).
module Maintenance
  class CityWriter
    class NotWritable < StandardError; end
    class Unreachable < StandardError; end

    def self.call(city)
      raise NotWritable, "cidade #{city.status}: só cidade ativa recebe escrita" unless city.servable?

      CityConnection.with(city) { yield }
    rescue *CityConnectionErrors::CLASSES => e
      redacted = CitySchema.redact(e.message)
      Rails.logger.warn("Maintenance::CityWriter: #{e.class}: #{redacted}")
      raise Unreachable, "#{e.class}: #{redacted}"
    end
  end
end
```

- [ ] **Step 4: `audited` entrega o `correlation_id`**

Em `BaseMutation#audited`, troque `result = yield` por `result = yield(correlation_id)`. Rode `spec/requests/maintenance` para confirmar que as mutations existentes seguem iguais.

- [ ] **Step 5: A base `CityMutation`**

`app/graphql/maintenance/mutations/city_mutation.rb`:

```ruby
# Base de toda mutation que escreve dentro de uma cidade (spec da API §8, §9).
#
# O caminho é um só, nesta ordem:
#   1. escopo do token (Credential#allows_city?) — como em city(slug:);
#   2. tentativa gravada na plataforma (BaseMutation#audited) — se falhar, nada roda;
#   3. CityWriter abre a cidade e o bloco roda o command com o ator mantenedor e
#      o correlation_id, que o command grava no evento de domínio;
#   4. resultado: `ok`, `rejected` (regra de domínio, cidade inexistente ou não
#      ativa, step-up) ou `error` (qualquer exceção, inclusive conexão).
#
# A mensagem de uma exceção nunca sai: a resposta leva a classe (ou, para falha
# de conexão, a mensagem já redigida por CityWriter).
module Maintenance
  module Mutations
    class CityMutation < BaseMutation
      argument :city_slug, String, required: true

      field :ok, Boolean, null: false
      field :errors, [ Types::UserErrorType ], null: false

      private

      # O bloco recebe (actor, correlation_id) e devolve o Result do command.
      # Result de falha vira Rejected no caminho `rejection_path`, com a
      # mensagem do próprio command — texto nosso, não do usuário.
      def in_city(city_slug:, event:, module_name:, rejection_path: "version", **fields)
        refuse_out_of_scope!(city_slug)

        audited(event: event, module_name: module_name, city_slug: city_slug, **fields) do |correlation_id|
          city = City.find_by(slug: city_slug)
          raise Rejected.new("cidade inexistente", path: "citySlug") if city.nil?

          begin
            result = CityWriter.call(city) { yield(MaintainerActor.new(credential.maintainer), correlation_id) }
          rescue CityWriter::NotWritable => e
            raise Rejected.new(e.message, path: "citySlug")
          end
          raise Rejected.new(result.message.presence || result.reason.to_s, path: rejection_path) if result.failure?

          result
        end
      rescue CityWriter::Unreachable => e
        raise GraphQL::ExecutionError.new(e.message, extensions: { "code" => "CITY_UNREACHABLE" })
      rescue GraphQL::ExecutionError
        raise
      rescue StandardError => e
        Rails.logger.warn("Maintenance::CityMutation: #{e.class} (mensagem omitida)")
        raise GraphQL::ExecutionError.new("falha ao escrever na cidade (#{e.class.name})",
                                          extensions: { "code" => "CITY_WRITE_FAILED" })
      end

      # Mesma recusa, com a mesma etiqueta, de QueryType#city — e pelo mesmo
      # motivo o controller a audita uma vez (Refusal::CODES).
      def refuse_out_of_scope!(city_slug)
        return if credential.allows_city?(city_slug)

        raise GraphQL::ExecutionError.new(
          "cidade fora do escopo do token",
          extensions: { "code" => Analyzers::Refusal::CITY_OUT_OF_SCOPE,
                        Analyzers::Refusal::FIELDS => [ field.graphql_name ] }
        )
      end
    end
  end
end
```

**Confira no `graphql-ruby` 2.6:** que `field`/`argument` declarados numa superclasse de `GraphQL::Schema::Mutation` são herdados; qual método dá o nome camelCase do campo de raiz dentro do resolver; e a ordem dos `rescue` (`audited` grava `error` e re-levanta; `in_city` converte em `GraphQL::ExecutionError`). `Rejected` é tratado dentro de `audited` e vira `{ ok: false, errors }` + `rejected`. Diga no relatório o que confirmou.

- [ ] **Step 6: `saveProtocolDraft`**

`app/graphql/maintenance/mutations/save_protocol_draft.rb`:

```ruby
module Maintenance
  module Mutations
    class SaveProtocolDraft < CityMutation
      description "Salva uma versão de protocolo em rascunho numa cidade. O mantenedor passa a " \
                  "constar como quem editou a versão, e quem edita nunca a assina."

      argument :definition, GraphQL::Types::JSON, required: true

      def resolve(city_slug:, definition:)
        definition = definition.to_h.deep_stringify_keys
        in_city(city_slug: city_slug, event: "maintenance.protocol.draft_saved", module_name: "protocol",
                rejection_path: "definition", protocol_key: definition["name"].to_s,
                version: definition["version"], changed_fields: [ "definition" ]) do |actor, correlation_id|
          Protocols::SaveDraft.call(definition: definition, by: actor, correlation_id: correlation_id)
        end
      end
    end
  end
end
```

**Confira** que `protocol_key`/`version` vindos da entrada não carregam nada além de nome e número (se `definition["name"]` não for String ou `version` não for Integer, recuse com erro de usuário no caminho `definition` ANTES de auditar — a auditoria não pode gravar lixo arbitrário do cliente).

- [ ] **Step 7: Registrar, auditar, analisar, guardar**

1. `MutationType`: `field :save_protocol_draft, mutation: Mutations::SaveProtocolDraft`.
2. `MaintenanceAudit` (`NAMES` e `case`) e `R18_PLATFORM_EVENT_NAMES`: `maintenance.protocol.draft_saved`.
3. `HumanOnly::RESTRICTED`: `saveProtocolDraft` (Decisão 3).
4. `CityBudget`: conte também campo de raiz de `Mutation` cuja definição tenha argumento `citySlug`; exemplo no spec do analisador: 5 `city` + 1 `saveProtocolDraft` numa operação → `CITY_BUDGET_EXCEEDED`.
5. `analyzers_spec.rb`, exemplo do escopo por cidade: `Query` como hoje; em `Mutation`, todo campo com `citySlug` tem classe que herda de `Maintenance::Mutations::CityMutation`, e `city_mutation.rb` chama `allows_city?`.
6. `maintenance_schema_spec.rb`, caminho único: `CityConnection.with` permitido em `city_reader.rb` **e** `city_writer.rb`; `CityWriter.call` só em `mutations/city_mutation.rb`; `CityReader.call` só em `city_type.rb`. Prove que falha com uma chamada de `CityWriter.call` fora da base (temporária, revertida). `EXPECTED_TYPES`: o campo novo de `Mutation` e o payload (nome real gerado pelo gem).

- [ ] **Step 8: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/queries/maintenance/city_writer.rb app/graphql/maintenance app/events/maintenance_audit.rb \
  spec/queries/maintenance/city_writer_spec.rb spec/requests/maintenance/protocol_mutations_spec.rb \
  spec/graphql/maintenance/analyzers_spec.rb spec/architecture/maintenance_schema_spec.rb \
  spec/events/platform_event_payload_guard_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: save a city protocol draft through the maintenance API

City writes get one path: the token's city scope, the attempt recorded
on the platform before anything runs, the city opened by CityWriter and
the command run as the maintainer with the audit's correlation id, then
the outcome. The maintainer is recorded as an editor of the version, so
it can never sign it.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 3: `submitProtocolForReview` e `publishProtocol`

**Files:**
- Create: `app/graphql/maintenance/mutations/submit_protocol_for_review.rb`, `app/graphql/maintenance/mutations/publish_protocol.rb`
- Modify: `app/graphql/maintenance/types/mutation_type.rb`, `app/graphql/maintenance/analyzers/human_only.rb`, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/protocol_mutations_spec.rb`

**Interfaces:**
- Produces:
  - `submitProtocolForReview(citySlug: String!, name: String!, version: Int!)` — sem step-up; evento `maintenance.protocol.submitted`, `changed_fields: ["status"]`
  - `publishProtocol(citySlug: String!, name: String!, version: Int!, code: String!)` — step-up; evento `maintenance.protocol.published`, `changed_fields: ["status"]`

- [ ] **Step 1: Escrever os specs que falham**

No mesmo arquivo de request:

1. **Fluxo com assinaturas da cidade:** o mantenedor salva e envia para revisão (`in_review`); dois revisores da cidade assinam a publicação (via `sign!`, no banco da cidade); o mantenedor publica com step-up → `published`; o `DomainEvent` `protocol.published` traz `actor_kind: "maintainer"`, o `correlation_id` da auditoria e os signatários.
2. **Sem assinaturas:** publicar → `ok: false`, erro no caminho `version` com a mensagem "falta 2 assinaturas…"/"faltam 2 assinaturas de publicação; revisores elegíveis na cidade: N" (a do command), auditoria `rejected`, versão continua `in_review`.
3. **O mantenedor não conta como revisor:** mesmo com um revisor da cidade assinando, a assinatura que falta não pode vir do mantenedor — não existe mutation de assinatura (a guarda da Task 5 prova), e `publishProtocol` segue recusando com 1 faltando.
4. **Step-up:** publicar com código errado → erro no caminho `code`, auditoria `rejected`, nada muda; enviar para revisão não pede código.
5. **Enviar uma versão que não é rascunho** → `invalid_state` no caminho `version`.

- [ ] **Step 2: Rodar e confirmar que falham**

- [ ] **Step 3: Implementar**

```ruby
module Maintenance
  module Mutations
    class PublishProtocol < CityMutation
      description "Publica uma versão em revisão. Exige duas assinaturas de revisores da cidade que " \
                  "não a editaram, e o TOTP do momento."

      argument :name, String, required: true
      argument :version, Integer, required: true
      argument :code, String, required: true, description: "TOTP do momento"

      def resolve(city_slug:, name:, version:, code:)
        in_city(city_slug: city_slug, event: "maintenance.protocol.published", module_name: "protocol",
                protocol_key: name, version: version, changed_fields: [ "status" ]) do |actor, correlation_id|
          step_up!(code)
          Protocols::Publish.call(version: version, name: name, by: actor, correlation_id: correlation_id)
        end
      end
    end
  end
end
```

`SubmitProtocolForReview` no mesmo formato, sem `code`, chamando `Protocols::SubmitForReview.call(name:, version:, by:, correlation_id:)`, evento `maintenance.protocol.submitted`, description "Envia um rascunho para revisão: a partir daqui revisores da cidade podem assinar."

**Confira** que `step_up!` escreve no `Maintainer` (banco de plataforma) e funciona dentro de `CityConnection.with`; se não, mova-o para antes do `CityWriter.call`, ainda dentro de `audited`, e diga no relatório.

Registre as duas em `MutationType`, `HumanOnly::RESTRICTED`, `MaintenanceAudit` (+ R18) e `EXPECTED_TYPES`.

- [ ] **Step 4: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/graphql/maintenance app/events/maintenance_audit.rb \
  spec/requests/maintenance/protocol_mutations_spec.rb spec/architecture/maintenance_schema_spec.rb \
  spec/events/platform_event_payload_guard_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: submit and publish a city protocol through the maintenance API

The maintainer can send a draft to review and carry out the publication
once two city reviewers who did not edit it have signed, with the TOTP
of the moment. It never signs: publication stays refused, with the
command's own count of missing signatures, until the city does.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 4: `activateProtocol`, `retireProtocol` e `revertProtocolActivation`

**Files:**
- Create: `app/graphql/maintenance/mutations/activate_protocol.rb`, `retire_protocol.rb`, `revert_protocol_activation.rb`
- Modify: `app/graphql/maintenance/types/mutation_type.rb`, `app/graphql/maintenance/analyzers/human_only.rb`, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/protocol_mutations_spec.rb`

**Interfaces:**
- Produces (todas com step-up):
  - `activateProtocol(citySlug, name, version, code)` → evento `maintenance.protocol.activated`, `changed_fields: ["status", "activated_at"]`
  - `retireProtocol(citySlug, name, version, code)` → `maintenance.protocol.retired`, `changed_fields: ["status", "retired_at"]`
  - `revertProtocolActivation(citySlug, name, reason, code)` → `maintenance.protocol.reverted`, `changed_fields: ["status"]`, `reason_given: true` (o texto do motivo NÃO entra na auditoria — Decisão 4)

- [ ] **Step 1: Escrever os specs que falham**

1. **Ativar com assinaturas de ativação da cidade** → `active`, a anterior volta a `published`, auditoria `attempted`/`ok` com o mesmo `correlation_id`, que aparece no `DomainEvent` `protocol.activated`.
2. **Ativar sem assinaturas de ativação** (mesmo com as de publicação) → `signatures_missing`, `rejected`.
3. **R1:** ativar um rascunho → `not_published`. **R4:** aposentar a ativa → `active_in_city`.
4. **Reversão:** versão em uso com linha-base + ativação assinada de outra → `revertProtocolActivation` com motivo volta à anterior; a linha `emergency_revert` tem o motivo e o ator mantenedor; **nenhum evento `maintenance.protocol.*` contém o texto do motivo**; sem motivo → `reason_required`; reverter uma reversão → `not_revertible`.
5. **Step-up** exigido nas três (código errado → caminho `code`, nada muda).

- [ ] **Step 2: Rodar e confirmar que falham**

- [ ] **Step 3: Implementar** as três no formato de `PublishProtocol` (`step_up!` primeiro, depois o command com `by: actor, correlation_id:`). `RevertProtocolActivation` recebe `reason: String!` e passa ao command; na auditoria, só `reason_given: reason.to_s.strip.present?`. Registre em `MutationType`, `HumanOnly::RESTRICTED`, `MaintenanceAudit` (+ R18) e `EXPECTED_TYPES`.

- [ ] **Step 4: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/graphql/maintenance app/events/maintenance_audit.rb \
  spec/requests/maintenance/protocol_mutations_spec.rb spec/architecture/maintenance_schema_spec.rb \
  spec/events/platform_event_payload_guard_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: activate, retire and revert a city protocol through the maintenance API

All three go through the same audited city path, require the TOTP of the
moment and stay bound by the domain: activation needs the city's two
activation signatures, the active version is never retired, and an
emergency revert goes back only one signed step. The revert reason stays
in the city; the platform audit only records that one was given.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 5: Guardas da escrita

**Files:**
- Modify: `spec/architecture/maintenance_schema_spec.rb`, `spec/graphql/maintenance/analyzers_spec.rb`, `spec/requests/maintenance/protocol_mutations_spec.rb`

Esta task não acrescenta comportamento: ela fecha as guardas que impedem a escrita de crescer errado. **Prove cada guarda nas duas direções** (verde no código atual; vermelha com uma violação temporária, revertida).

- [ ] **Step 1: Toda mutation de cidade é auditada e passa por command.** Para todo campo de `Mutation` cuja classe herda de `CityMutation`, lendo o arquivo da classe (linhas não comentadas): chama `in_city(`; o evento passado está em `MaintenanceAudit::NAMES`; nenhuma persistência direta (`update!`, `update(`, `save!`, `save(`, `create!`, `destroy`, `delete`, `update_all`, `update_column`, `insert`).
- [ ] **Step 2: O mantenedor nunca assina nem cria quem aprova.** Nenhum arquivo em `app/graphql/maintenance/` chama `Protocols::Sign`, `GrantRole` ou `InviteMember`, e nenhum campo de `Mutation` tem `sign` ou `signature` no nome.
- [ ] **Step 3: Step-up onde se aprova ou põe em uso.** Os campos `publishProtocol`, `activateProtocol`, `retireProtocol`, `revertProtocolActivation` declaram `argument :code` e chamam `step_up!`; uma lista explícita no spec diz quais mutations de cidade dispensam step-up (`saveProtocolDraft`, `submitProtocolForReview`) — mutation de cidade nova que não esteja em nenhuma das duas listas faz a guarda falhar.
- [ ] **Step 4: Nenhuma mensagem de exceção sai.** Para cada mutation de cidade (derivada do schema, com um mapa explícito mutation → command que falha se faltar alguma), stub do command levantando `RuntimeError` com um marcador → o marcador não aparece em `response.body` e o código é `CITY_WRITE_FAILED`.
- [ ] **Step 5: Rodar, suíte completa e commit**

```bash
cd apps/api
/opt/homebrew/bin/git add spec/architecture/maintenance_schema_spec.rb spec/graphql/maintenance/analyzers_spec.rb \
  spec/requests/maintenance/protocol_mutations_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
test: guard the maintenance city writes

Every city mutation goes through in_city and a declared audit event and
never persists on its own; no mutation signs or creates an approver;
every act that approves or puts a version in use requires the step-up;
and no exception message ever leaves one.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 6: Spec da API de manutenção (repo `docs`)

**Files:**
- Modify: `superpowers/specs/2026-09-17-maintenance-graphql-api-design.md`

- [ ] **Step 1: Atualizar**, descrevendo o que foi construído, na regra atual (sem notas de emenda):
- **§8 Mutations:** a lista das seis mutations de protocolo, quais pedem step-up, que o mantenedor nunca assina (e que não há mutation de assinatura, papel ou convite de papel privilegiado), escrita só em cidade ativa, e os códigos `CITY_UNREACHABLE`/`CITY_WRITE_FAILED` (escrita) e `CITY_READ_FAILED` (leitura).
- **§9 Auditoria:** o motivo da reversão fica na cidade; a plataforma registra só `reason_given`.
- **§11 Fatias:** a fatia 5 começa por protocolos (5a); as próximas seguem por módulo.

- [ ] **Step 2: Commit no repo `docs`** (só esse arquivo):

```bash
cd docs
/opt/homebrew/bin/git add superpowers/specs/2026-09-17-maintenance-graphql-api-design.md
/opt/homebrew/bin/git commit -F - <<'EOF'
docs: record the maintenance API protocol writes

The spec now lists the six protocol mutations, which require the step-up,
that the maintainer never signs nor creates approvers, that writes only
reach active cities, and that the revert reason stays in the city.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

## Verificação final do plano

- [ ] Suíte completa com o worker parado: 0 falhas, em ~2,5 min.
- [ ] Pela API de manutenção: salvar → enviar → (cidade assina) → publicar → (cidade assina) → ativar → reverter, com step-up onde pede, auditoria de três passos e o `correlation_id` no evento de domínio.
- [ ] Publicar e ativar sem as assinaturas da cidade são recusados com a contagem que falta.
- [ ] Nenhuma mutation assina, concede ou convida papel privilegiado.
- [ ] Token de serviço não alcança nenhuma mutation de protocolo.
- [ ] Nenhuma mensagem de exceção sai, na leitura ou na escrita, fora a de conexão, redigida.
- [ ] `git diff main -- app/commands app/services/city_inventory.rb` vazio (os commands só são chamados).

## Fora deste plano

- **Telas de escrita no frontend da manutenção** (`apps/maintenance`): plano próprio, depois deste.
- **Próximas fatias de escrita (5b em diante), uma por módulo:** canal de WhatsApp, membros não privilegiados (nunca `InviteAdmin`/`city:invite_admin` pelo mantenedor), perfil da cidade e destinatários de alerta (sem command hoje — criar antes), operação (republicar evento, reconstruir métricas, jobs falhos).
- **Escrita por token de serviço:** decisão própria.
- **SDL publicado em `contracts`:** fatia 6.
