# Banco por cidade — Plano 7: chave de cifra por cidade

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** cada cidade passa a cifrar os próprios dados com uma chave derivada só dela, de modo que um dump de uma cidade não seja legível com o material de outra.

**Architecture:** `cities.encryption_key` (hoje gerada e nunca lida) vira o material por cidade; a chave efetiva é **derivada** de `chave da plataforma + encryption_key da cidade`. Os atributos **não-determinísticos** passam a usar essa chave por troca de contexto dentro de `CityConnection.with`. Os **determinísticos** (`Conversation#phone`, `Author#token`) não podem: o `Scheme` do Rails resolve o provedor determinístico direto de `config.deterministic_key`, antes de olhar o contexto. Para eles, um provedor próprio declarado no `encrypts` resolve a chave da cidade a cada operação, lendo `Current.city`.

**Tech Stack:** Rails 8.1 (`ActiveRecord::Encryption`), PostgreSQL, Solid Queue, rake.

**Spec:** `docs/superpowers/specs/2026-09-12-banco-por-cidade-design.md` (§6 "Chaves por cidade"; Riscos abertos, item 4 "Custódia de chave por cidade")

## Global Constraints

- **Nunca imprimir material de chave**: nem `encryption_key`, nem chave derivada, nem ciphertext em log, relatório ou terminal. Contagens e booleanos, sim; valores, não.
- Commits em inglês, terminando com a linha exata `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Comandos Ruby/Rails/rspec rodam no container, a partir da raiz do monorepo `<raiz-do-monorepo>`: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Nunca no host.
- `git` do PATH está quebrado nesta máquina: use `/opt/homebrew/bin/git ...`. Nunca dar push.
- Suíte completa: `docker compose stop worker` antes, `docker compose start worker` depois, timeout ≥ 600000 ms, nunca duas suítes ao mesmo tempo, nunca `git stash`.
- Nada de operação destrutiva em `rota_saude_development`, `rota_saude_test`, `rota_saude_platform_*`, `rota_saude_no_city_selected`. As cidades de dev `curitiba` e `maringa` **são** migradas por este plano (Task 6), com backup antes.
- Nunca rodar `start.sh`. Só `apps/api` é tocado.

## Decisões (tomadas com o usuário, não re-perguntar)

1. **Escopo dividido.** Este plano é só chave por cidade e a migração dela. Formulário de provisionamento do console e gates operacionais (`config.hosts`, `max_connections`, PG 16, `verify-full`, backfill de `published_at`) ficam para o Plano 8.
2. **Todos os atributos vão para a chave da cidade**, inclusive os determinísticos. Custo aceito: provedor próprio nos dois modelos e janela com a cidade suspensa na migração.
3. **A chave da cidade é derivada** de `chave da plataforma + cities.encryption_key`. Não há cofre novo. **Consequência registrada:** quem tem a chave da plataforma continua abrindo tudo; o ganho é que ciphertext de uma cidade não é legível nem comparável com o de outra, e um dump roubado isolado não abre nada.
4. **`city:restore` fica fora** deste plano. Enquanto isso, um dump só é restaurável na cidade de origem com a `encryption_key` dela intacta — vai no runbook da Task 7.

## Fatos verificados no código e na gem (2026-09-16, não re-descobrir)

- `config/initializers/active_record_encryption.rb` define um único trio global `primary_key` / `deterministic_key` / `key_derivation_salt`.
- No banco **da cidade**: `User#otp_secret`, `Consent#evidence`, `InboundMessage#raw` (não-determinísticos) e `Conversation#phone` (`app/models/conversation.rb:7`, buscado em `:21,24`) e `Author#token` (`app/models/author.rb:3`, buscado em `app/controllers/protocols_controller.rb:44`), determinísticos.
- No banco **de plataforma**, seguem na chave global: `Operator#otp_secret`, `CityChannel#access_token`, `City#database_url`, `City#encryption_key`.
- `ActiveRecord::Encryption::Context::PROPERTIES` é `%i[key_provider key_generator cipher message_serializer encryptor frozen_encryption]` — **não existe `deterministic_key_provider`**; `with_encryption_context` faz `send("#{key}=", value)`, então passar uma propriedade inexistente levanta `NoMethodError`.
- `Scheme#key_provider` (gem, `scheme.rb:57`) é `@key_provider_param || key_provider_from_key || deterministic_key_provider || default_key_provider`. Para atributo determinístico, `deterministic_key_provider` monta `DeterministicKeyProvider.new(ActiveRecord::Encryption.config.deterministic_key)` — **global, fora do contexto**. Só `@key_provider_param` (o `key_provider:` passado no `encrypts`) vence isso.
- `DeterministicKeyProvider` aceita **uma senha só**; com mais de uma levanta `Errors::Configuration, "Deterministic encryption keys can't be rotated"`.
- Um provedor de chave precisa responder a `encryption_key` (a chave de escrita) e `decryption_keys(message)` (as candidatas de leitura); `Key#secret` é público.
- `CityConnection.with` **não** seta `Current.city` hoje — quem seta é `CityResolution` e `CityScopedJob`. A Task 3 passa a setar dentro de `with`, porque o provedor determinístico depende disso.
- `Result.ok(payload = {})` / `Result.fail(reason, message:)`, com `ok?` e `failure?` (`app/commands/result.rb`).
- Contagens em dev: curitiba 35 conversas e 40 `inbound_messages`; maringa 15 e 16; zero `otp_secret`, `consents.evidence` e `authors.token`.

## File Structure

- **Criar** `app/services/city_encryption.rb` — deriva o material da cidade e monta os dois provedores.
- **Criar** `app/services/city_deterministic_key_provider.rb` — provedor que resolve a chave determinística da cidade corrente a cada operação.
- **Modificar** `app/models/conversation.rb` e `app/models/author.rb` — `encrypts ... key_provider:` apontando para o provedor novo.
- **Modificar** `app/models/city_connection.rb` — `with` seta `Current.city` e envolve o bloco no contexto da cidade.
- **Criar** `app/commands/city_rekey.rb` — migração de uma cidade: lê no contexto antigo, grava no novo.
- **Modificar** `lib/tasks/city.rake` — `city:rekey[slug]` e `city:rotate_key[slug]`.
- **Modificar** `apps/api/README.md` — runbook de custódia, rotação e limite do restore.
- **Testes:** `spec/services/city_encryption_spec.rb`, `spec/services/city_deterministic_key_provider_spec.rb`, `spec/models/city_connection_encryption_spec.rb`, `spec/commands/city_rekey_spec.rb`, `spec/tasks/city_rekey_rake_spec.rb`.

---

### Task 1: `CityEncryption` — derivar o material da cidade

**Files:**
- Create: `apps/api/app/services/city_encryption.rb`
- Test: `apps/api/spec/services/city_encryption_spec.rb`

**Interfaces:**
- Produces: `CityEncryption.key_provider(city)`, `CityEncryption.deterministic_key_provider(city)`, `CityEncryption.context_properties(city) -> {key_provider:}`, `CityEncryption::MissingKey`.

- [ ] **Step 1: Escrever o spec**

```ruby
# apps/api/spec/services/city_encryption_spec.rb
require "rails_helper"

RSpec.describe CityEncryption do
  let(:city_a) { City.new(slug: "aaa", name: "A", status: "active", encryption_key: "a" * 64) }
  let(:city_b) { City.new(slug: "bbb", name: "B", status: "active", encryption_key: "b" * 64) }

  it "gives each city different key material" do
    expect(described_class.key_provider(city_a).encryption_key.secret)
      .not_to eq(described_class.key_provider(city_b).encryption_key.secret)
    expect(described_class.deterministic_key_provider(city_a).encryption_key.secret)
      .not_to eq(described_class.deterministic_key_provider(city_b).encryption_key.secret)
  end

  it "is stable for the same city" do
    expect(described_class.key_provider(city_a).encryption_key.secret)
      .to eq(described_class.key_provider(city_a).encryption_key.secret)
  end

  # Context::PROPERTIES não tem deterministic_key_provider — passar isso levanta
  # NoMethodError. O contexto só carrega o provedor não-determinístico.
  it "context_properties carries only key_provider" do
    expect(described_class.context_properties(city_a).keys).to eq([ :key_provider ])
  end

  it "fails closed without key material" do
    expect { described_class.key_provider(City.new(slug: "x", name: "X")) }.to raise_error(CityEncryption::MissingKey)
    expect { described_class.deterministic_key_provider(nil) }.to raise_error(CityEncryption::MissingKey)
  end
end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_encryption_spec.rb`
Expected: FAIL — `uninitialized constant CityEncryption`.

- [ ] **Step 3: Implementar**

```ruby
# apps/api/app/services/city_encryption.rb
# Material de cifra de uma cidade (Plano 7, spec §6).
#
# A chave efetiva é DERIVADA de (chave da plataforma + cities.encryption_key):
# não há cofre novo, o segredo raiz continua sendo o da plataforma. O que a
# derivação compra é separação entre cidades; o que ela não compra é isolamento
# contra quem tem a chave da plataforma.
#
# Dois provedores, porque o Rails trata os casos de formas diferentes:
#   - não-determinístico: entra por contexto (CityConnection.with);
#   - determinístico: o Scheme resolve `DeterministicKeyProvider` direto de
#     config.deterministic_key e NUNCA olha o contexto — só o `key_provider:`
#     passado no `encrypts` vence isso (ver CityDeterministicKeyProvider).
module CityEncryption
  class MissingKey < StandardError; end

  module_function

  def context_properties(city)
    { key_provider: key_provider(city) }
  end

  def key_provider(city)
    ActiveRecord::Encryption::DerivedSecretKeyProvider.new([ secret_for(city, platform_primary_key) ])
  end

  def deterministic_key_provider(city)
    ActiveRecord::Encryption::DeterministicKeyProvider.new(secret_for(city, platform_deterministic_key))
  end

  def secret_for(city, platform_secret)
    material = city.respond_to?(:encryption_key) ? city.encryption_key.to_s : ""
    raise MissingKey, "cidade sem encryption_key: não há chave a derivar" if material.blank?
    raise MissingKey, "chave de plataforma ausente" if platform_secret.to_s.blank?

    "#{platform_secret}:#{material}"
  end

  def platform_primary_key = Rails.application.config.active_record.encryption.primary_key
  def platform_deterministic_key = Rails.application.config.active_record.encryption.deterministic_key
end
```

- [ ] **Step 4: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_encryption_spec.rb`
Expected: PASS (4 examples).

- [ ] **Step 5: Commit**

```bash
cd apps/api && /opt/homebrew/bin/git add app/services/city_encryption.rb spec/services/city_encryption_spec.rb
/opt/homebrew/bin/git commit -m "Derive per-city encryption material from the platform secret

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: provedor determinístico por cidade

**Files:**
- Create: `apps/api/app/services/city_deterministic_key_provider.rb`
- Modify: `apps/api/app/models/conversation.rb:7`, `apps/api/app/models/author.rb:3`
- Test: `apps/api/spec/services/city_deterministic_key_provider_spec.rb`

**Interfaces:**
- Consumes: `CityEncryption.deterministic_key_provider(city)` (Task 1), `Current.city`.
- Produces: `CityDeterministicKeyProvider.new` respondendo a `encryption_key` e `decryption_keys(message)`; usado como `key_provider:` nos dois `encrypts`.

- [ ] **Step 1: Escrever o spec**

```ruby
# apps/api/spec/services/city_deterministic_key_provider_spec.rb
require "rails_helper"

RSpec.describe CityDeterministicKeyProvider do
  let(:city_a) { City.new(slug: "aaa", name: "A", status: "active", encryption_key: "a" * 64) }
  let(:city_b) { City.new(slug: "bbb", name: "B", status: "active", encryption_key: "b" * 64) }

  it "resolves the key of the city in Current at call time" do
    provider = described_class.new

    secret_a = Current.set(city: city_a) { provider.encryption_key.secret }
    secret_b = Current.set(city: city_b) { provider.encryption_key.secret }

    expect(secret_a).not_to eq(secret_b)
  end

  # Não chame decryption_keys com nil: KeyProvider#decryption_keys acessa
  # `message.headers` sem checar nil. A prova de leitura é o round-trip pelo
  # modelo, na Task 3.
  it "memoizes one provider per city material" do
    provider = described_class.new

    first  = Current.set(city: city_a) { provider.encryption_key.secret }
    second = Current.set(city: city_a) { provider.encryption_key.secret }
    rotated = Current.set(city: City.new(slug: "aaa", name: "A", status: "active", encryption_key: "c" * 64)) do
      provider.encryption_key.secret
    end

    expect(first).to eq(second)
    expect(rotated).not_to eq(first)
  end

  it "fails closed outside a city" do
    provider = described_class.new
    Current.set(city: nil) do
      expect { provider.encryption_key }.to raise_error(CityEncryption::MissingKey)
    end
  end
end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_deterministic_key_provider_spec.rb`
Expected: FAIL — `uninitialized constant CityDeterministicKeyProvider`.

- [ ] **Step 3: Implementar o provedor**

```ruby
# apps/api/app/services/city_deterministic_key_provider.rb
# Provedor de chave DETERMINÍSTICA por cidade (Plano 7).
#
# Por que existe: para atributo determinístico, o Scheme do Rails monta
# `DeterministicKeyProvider.new(config.deterministic_key)` e nunca consulta o
# contexto de cifra — trocar contexto em CityConnection.with não alcança
# `Conversation#phone` nem `Author#token`. O único ponto que vence essa
# resolução é o `key_provider:` do próprio `encrypts`, e ele é avaliado UMA vez,
# na carga da classe. Por isso o provedor é declarado uma vez e resolve a chave
# a cada chamada, a partir de Current.city.
#
# Consequência: toda leitura e escrita desses atributos precisa acontecer com
# Current.city setado. CityConnection.with passou a garantir isso (Plano 7).
class CityDeterministicKeyProvider
  def encryption_key
    provider.encryption_key
  end

  def decryption_keys(message = nil)
    provider.decryption_keys(message)
  end

  private

  # Um DeterministicKeyProvider por cidade, memoizado por id: derivar a cada
  # linha lida sairia caro numa consulta de muitas linhas.
  def provider
    city = Current.city
    raise CityEncryption::MissingKey, "atributo determinístico acessado fora de uma cidade" if city.nil?

    cache[cache_key(city)] ||= CityEncryption.deterministic_key_provider(city)
  end

  def cache_key(city) = [ city.id, city.encryption_key ].join(":")

  def cache
    Thread.current[:city_deterministic_key_providers] ||= {}
  end
end
```

A memoização é por thread e inclui o material na chave, então `city:rotate_key` (que troca `encryption_key`) não reaproveita provedor velho.

- [ ] **Step 4: Ligar nos dois modelos**

`apps/api/app/models/conversation.rb`, linha 7:

```ruby
  encrypts :phone, deterministic: true, key_provider: CityDeterministicKeyProvider.new
```

`apps/api/app/models/author.rb`, linha 3:

```ruby
  encrypts :token, deterministic: true, key_provider: CityDeterministicKeyProvider.new
```

- [ ] **Step 5: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_deterministic_key_provider_spec.rb`
Expected: PASS (3 examples).

- [ ] **Step 6: Commit**

```bash
cd apps/api && /opt/homebrew/bin/git add app/services/city_deterministic_key_provider.rb app/models/conversation.rb app/models/author.rb spec/services/city_deterministic_key_provider_spec.rb
/opt/homebrew/bin/git commit -m "Resolve the deterministic encryption key per city

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: contexto e `Current.city` dentro de `CityConnection.with`

**Files:**
- Modify: `apps/api/app/models/city_connection.rb` (método `with`)
- Test: `apps/api/spec/models/city_connection_encryption_spec.rb`

**Interfaces:**
- Consumes: `CityEncryption.context_properties(city)` (Task 1), `CityDeterministicKeyProvider` (Task 2).
- Produces: dentro de `CityConnection.with(city)`, `Current.city` está setado e todo atributo cifrado usa o material daquela cidade.

- [ ] **Step 1: Escrever o spec**

```ruby
# apps/api/spec/models/city_connection_encryption_spec.rb
require "rails_helper"

RSpec.describe "CityConnection encryption context" do
  let!(:city_a) { create(:city, database_url: city_database_url("rota_saude_test_city_a")) }
  let!(:city_b) { create(:city, database_url: city_database_url("rota_saude_test_city_b")) }

  it "sets Current.city for the block" do
    expect(CityConnection.with(city_a) { Current.city&.slug }).to eq(city_a.slug)
  end

  # InboundMessage exige from e kind (validação + NOT NULL) e `raw` é coluna de
  # texto: passe String, não Hash.
  it "round-trips a non-deterministic attribute inside the city" do
    id = CityConnection.with(city_a) do
      InboundMessage.create!(message_id: "wamid-#{SecureRandom.hex(4)}", from: "+5541999990000",
                             kind: "text", raw: '{"t":"oi"}').id
    end
    expect(CityConnection.with(city_a) { InboundMessage.find(id).raw }).to eq('{"t":"oi"}')
  end

  it "round-trips a deterministic attribute and keeps the lookup working" do
    CityConnection.with(city_a) { Conversation.create!(phone: "+5541999990002", state: :greeting) }

    expect(CityConnection.with(city_a) { Conversation.where(phone: "+5541999990002").count }).to eq(1)
  end

  it "does not decrypt a row of city A with city B's material" do
    id = CityConnection.with(city_a) do
      InboundMessage.create!(message_id: "wamid-#{SecureRandom.hex(4)}", from: "+5541999990001",
                             kind: "text", raw: '{"t":"segredo"}').id
    end
    raw = CityConnection.with(city_a) do
      InboundMessage.connection.select_value(InboundMessage.sanitize_sql(["SELECT raw FROM inbound_messages WHERE id = ?", id]))
    end

    expect {
      ActiveRecord::Encryption.with_encryption_context(**CityEncryption.context_properties(city_b)) do
        ActiveRecord::Encryption.encryptor.decrypt(raw)
      end
    }.to raise_error(ActiveRecord::Encryption::Errors::Decryption)
  end

  it "does not match a deterministic value written by the other city" do
    CityConnection.with(city_a) { Conversation.create!(phone: "+5541999990003", state: :greeting) }

    expect(CityConnection.with(city_b) { Conversation.where(phone: "+5541999990003").count }).to eq(0)
  end
end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_connection_encryption_spec.rb`
Expected: FAIL — sem `Current.city` dentro de `with`, o provedor da Task 2 levanta `MissingKey`; e sem o contexto, o material de B decifra dado de A.

- [ ] **Step 3: Implementar**

Em `apps/api/app/models/city_connection.rb`, o método `with`:

```ruby
    # Domínio, fila E CIFRA da cidade (Planos 5 e 7).
    #
    # Current.city é setado aqui porque o provedor determinístico
    # (CityDeterministicKeyProvider) resolve a chave a partir dele a cada
    # operação — sem isso, ler `phone` ou `token` levanta MissingKey. Os
    # chamadores que já setavam (CityResolution, CityScopedJob) continuam
    # funcionando: Current.set é reentrante.
    #
    # A ordem importa: o provedor não-determinístico é montado ANTES de entrar
    # no contexto, porque `city.encryption_key` vive no banco de plataforma e é
    # decifrado com a chave global.
    def with(city, &block)
      ensure_pool(city)
      properties = CityEncryption.context_properties(city)

      Current.set(city: city) do
        ActiveRecord::Encryption.with_encryption_context(**properties) do
          ActiveRecord::Base.connected_to_many([ CityRecord, SolidQueue::Record ], role: :writing, shard: city.shard, &block)
        end
      end
    end
```

- [ ] **Step 4: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/city_connection_encryption_spec.rb`
Expected: PASS (5 examples).

- [ ] **Step 5: Rodar a suíte inteira e enfrentar o que quebrar**

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
```

Esperado: **falhas**, e cada uma é informação. Três famílias prováveis:
- spec que escreve numa cidade e lê em outra sessão → prenda-o a uma cidade;
- spec que toca `phone`/`token` fora de `CityConnection.with` → passa a levantar `MissingKey`; envolva no `with` da cidade certa;
- dado criado antes desta mudança dentro do mesmo exemplo → recrie dentro do bloco.

**Não** relaxe asserção, não desligue o contexto e não dê rescue em `MissingKey` para fazer spec passar: o "falha fechado" é o comportamento desejado.

- [ ] **Step 6: Commit**

```bash
cd apps/api && /opt/homebrew/bin/git add app/models/city_connection.rb spec
/opt/homebrew/bin/git commit -m "Switch encryption context and Current.city with the city connection

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: `CityRekey` — ler com a chave antiga, gravar com a nova

**Files:**
- Create: `apps/api/app/commands/city_rekey.rb`
- Test: `apps/api/spec/commands/city_rekey_spec.rb`

**Interfaces:**
- Consumes: `CityEncryption`, `CityConnection.with`, `Result`.
- Produces: `CityRekey.call(city:, from_key: nil, to_key: nil) -> Result` com `payload[:counts]`. `from_key`/`to_key` são o **material** (string) de origem e destino; `nil` em `from_key` significa "chave global atual", e `nil` em `to_key` significa "material atual da cidade".

- [ ] **Step 1: Escrever o spec**

```ruby
# apps/api/spec/commands/city_rekey_spec.rb
require "rails_helper"

# Não dá para reusar o ReencryptionJob: ele chama record.encrypt, que lê e
# escreve no MESMO contexto; e o provedor determinístico não aceita duas chaves.
RSpec.describe CityRekey do
  let!(:city) { create(:city, database_url: city_database_url("rota_saude_test_city_a")) }

  def with_material(material, &block)
    other = City.new(slug: city.slug, name: city.name, status: city.status,
                     database_url: city.database_url, encryption_key: material)
    CityConnection.with(city) do
      Current.set(city: other) do
        ActiveRecord::Encryption.with_encryption_context(**CityEncryption.context_properties(other), &block)
      end
    end
  end

  it "rewrites rows so they read under the city's own material" do
    old_material = "0" * 64
    convo = with_material(old_material) { Conversation.create!(phone: "+5541988880001", state: :greeting) }

    result = CityRekey.call(city: city, from_key: old_material)

    expect(result).to be_ok
    expect(result.payload[:counts]["Conversation"]).to be >= 1
    expect(CityConnection.with(city) { Conversation.find(convo.id).phone }).to eq("+5541988880001")
  end

  it "makes the deterministic lookup work under the new material" do
    old_material = "0" * 64
    with_material(old_material) { Conversation.create!(phone: "+5541988880002", state: :greeting) }

    CityRekey.call(city: city, from_key: old_material)

    expect(CityConnection.with(city) { Conversation.where(phone: "+5541988880002").count }).to eq(1)
  end

  it "fails when a row cannot be read under the source material" do
    CityConnection.with(city) do
      InboundMessage.create!(message_id: "wamid-#{SecureRandom.hex(4)}", from: "+5541988880009",
                             kind: "text", raw: '{"t":"x"}')
    end

    result = CityRekey.call(city: city, from_key: "9" * 64)

    expect(result).to be_failure
    expect(result.reason).to eq(:unreadable)
  end
end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/commands/city_rekey_spec.rb`
Expected: FAIL — `uninitialized constant CityRekey`.

- [ ] **Step 3: Implementar**

```ruby
# apps/api/app/commands/city_rekey.rb
# Troca o material de cifra de UMA cidade (Plano 7).
#
# Lê cada registro no contexto de ORIGEM e grava no de DESTINO. Não dá para usar
# `record.encrypt` (ReencryptionJob) porque ele lê e escreve no mesmo contexto, e
# o provedor determinístico aceita uma chave só — não existe janela de duas.
#
# A cidade deve estar SUSPENSA: entre ler e gravar, uma busca determinística de
# outro processo usaria a chave errada e não acharia a linha.
class CityRekey
  BATCH_SIZE = 200

  # Só o que mora no banco DA CIDADE.
  TARGETS = [
    [ User,           :otp_secret ],
    [ Conversation,   :phone ],
    [ InboundMessage, :raw ],
    [ Consent,        :evidence ],
    [ Author,         :token ]
  ].freeze

  def self.call(city:, from_key: nil, to_key: nil) = new(city: city, from_key: from_key, to_key: to_key).call

  def initialize(city:, from_key: nil, to_key: nil)
    @city = city
    @from_city = shadow_city(from_key)
    @to_city = shadow_city(to_key)
  end

  def call
    counts = Hash.new(0)

    CityConnection.with(@city) do
      TARGETS.each { |model, attribute| counts[model.name] += rewrite(model, attribute) }
    end

    Result.ok(counts: counts)
  rescue ActiveRecord::Encryption::Errors::Decryption
    Result.fail(:unreadable, message: "cidade #{@city.slug}: linha ilegível com o material de origem")
  end

  private

  # `from_key`/`to_key` nil = material atual da cidade. A cópia em memória existe
  # só para montar o provedor: nada dela é salvo.
  def shadow_city(material)
    return @city if material.blank?

    City.new(slug: @city.slug, name: @city.name, status: @city.status,
             database_url: @city.database_url, encryption_key: material)
  end

  def rewrite(model, attribute)
    count = 0

    model.unscoped.in_batches(of: BATCH_SIZE) do |batch|
      batch.pluck(:id).each do |id|
        plaintext = in_city(@from_city) { model.unscoped.find(id).public_send(attribute) }
        next if plaintext.nil?

        in_city(@to_city) do
          record = model.unscoped.find(id)
          record.public_send("#{attribute}=", plaintext)
          record.save!(validate: false)
        end
        count += 1
      end
    end

    count
  end

  # Current.city governa o provedor determinístico; o contexto governa o resto.
  def in_city(city, &block)
    Current.set(city: city) do
      ActiveRecord::Encryption.with_encryption_context(**CityEncryption.context_properties(city), &block)
    end
  end
end
```

- [ ] **Step 4: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/commands/city_rekey_spec.rb`
Expected: PASS (3 examples).

- [ ] **Step 5: Commit**

```bash
cd apps/api && /opt/homebrew/bin/git add app/commands/city_rekey.rb spec/commands/city_rekey_spec.rb
/opt/homebrew/bin/git commit -m "Add CityRekey: read under the old material, write under the city's own

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 5: tarefas rake `city:rekey` e `city:rotate_key`

**Files:**
- Modify: `apps/api/lib/tasks/city.rake` (no mesmo `namespace :city`, depois de `:backup`)
- Test: `apps/api/spec/tasks/city_rekey_rake_spec.rb`

**Interfaces:**
- Consumes: `CityRekey.call`, o lambda `lifecycle_city` já existente no arquivo.
- Produces: `rails 'city:rekey[slug]'` e `rails 'city:rotate_key[slug]'`.

- [ ] **Step 1: Escrever o spec**

```ruby
# apps/api/spec/tasks/city_rekey_rake_spec.rb
require "rails_helper"
require "rake"

RSpec.describe "city:rekey and city:rotate_key rake tasks" do
  before(:all) { Rails.application.load_tasks unless Rake::Task.task_defined?("city:rekey") }
  before { %w[city:rekey city:rotate_key].each { |t| Rake::Task[t].reenable } }

  let!(:city) do
    create(:city, database_url: city_database_url("rota_saude_test_city_a"), status: "suspended")
  end

  def invoke_silently(name, *args)
    original_stderr, $stderr = $stderr, StringIO.new
    Rake::Task[name].invoke(*args)
  ensure
    $stderr = original_stderr
  end

  it "refuses an unknown slug" do
    expect { invoke_silently("city:rekey", "naoexiste") }.to raise_error(SystemExit)
  end

  it "refuses a city that is not suspended" do
    city.update!(status: "active")
    expect { invoke_silently("city:rekey", city.slug) }.to raise_error(SystemExit)
  end

  it "rekeys a suspended city without echoing key material" do
    expect { invoke_silently("city:rekey", city.slug) }
      .to output(/\[city:rekey\] #{city.slug}/).to_stdout
  end

  it "rotate_key replaces the stored material" do
    before_key = city.reload.encryption_key
    invoke_silently("city:rotate_key", city.slug)
    expect(city.reload.encryption_key).not_to eq(before_key)
  end
end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/tasks/city_rekey_rake_spec.rb`
Expected: FAIL — `Don't know how to build task 'city:rekey'`.

- [ ] **Step 3: Implementar**

Em `apps/api/lib/tasks/city.rake`, depois da task `:backup`:

```ruby
  # Plano 7: migração e rotação de chave de cifra de uma cidade.
  #
  # A cidade precisa estar SUSPENSA: entre ler uma linha com o material antigo e
  # gravá-la com o novo, uma busca determinística de outro processo (webhook,
  # job) usaria a chave errada e não acharia a linha.
  # Runbook: city:backup → city:suspend → city:rekey → city:resume
  desc "Migra uma cidade suspensa para a chave derivada dela. Uso: city:rekey[slug]"
  task :rekey, %i[slug] => :environment do |_t, args|
    city = lifecycle_city.call("city:rekey", args[:slug])
    unless city.status == "suspended"
      abort "[city:rekey] cidade #{city.slug} precisa estar suspensa (status=#{city.status}) — rode city:backup e city:suspend antes"
    end

    result = CityRekey.call(city: city)
    abort "[city:rekey] #{result.reason}: #{result.message}" if result.failure?
    puts "[city:rekey] #{city.slug} → #{result.payload[:counts].map { |m, n| "#{m}=#{n}" }.join(' ')}"
  end

  desc "Gera material NOVO para uma cidade suspensa e reescreve os dados. Uso: city:rotate_key[slug]"
  task :rotate_key, %i[slug] => :environment do |_t, args|
    city = lifecycle_city.call("city:rotate_key", args[:slug])
    abort "[city:rotate_key] cidade #{city.slug} precisa estar suspensa (status=#{city.status})" unless city.status == "suspended"

    previous = city.encryption_key
    city.update!(encryption_key: SecureRandom.hex(32))

    result = CityRekey.call(city: city.reload, from_key: previous)
    if result.failure?
      city.update!(encryption_key: previous)
      abort "[city:rotate_key] #{result.reason}: #{result.message} — material anterior restaurado no catálogo"
    end

    Platform.audit("city.key_rotated", city_id: city.id)
    puts "[city:rotate_key] #{city.slug} → #{result.payload[:counts].map { |m, n| "#{m}=#{n}" }.join(' ')}"
  end
```

O rollback do material em caso de falha é o que impede a cidade de ficar com metade das linhas ilegíveis. Se `city.key_rotated` precisar entrar na allowlist de eventos de plataforma (`spec/events/platform_event_payload_guard_spec.rb`), acrescente com payload só de `city_id`.

- [ ] **Step 4: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/tasks/city_rekey_rake_spec.rb`
Expected: PASS (4 examples).

- [ ] **Step 5: Suíte completa e commit**

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
cd apps/api && /opt/homebrew/bin/git add lib/tasks/city.rake spec
/opt/homebrew/bin/git commit -m "Add city:rekey and city:rotate_key

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 6: migrar as cidades de dev

**Files:** nenhum arquivo de produção; a entrega é o relatório com as saídas.

- [ ] **Step 1: Contar antes**

```bash
docker compose exec -T api bin/rails runner '
%w[curitiba maringa].each do |slug|
  city = City.find_by!(slug: slug)
  CityConnection.with(city) { puts "#{slug}: conversations=#{Conversation.count} inbound=#{InboundMessage.count} users=#{User.count}" }
end'
```
Esperado, pelo levantamento: curitiba 35 e 40; maringa 15 e 16. Registre.

- [ ] **Step 2: Backup das duas**

```bash
docker compose exec -T api bin/rails 'city:backup[curitiba]'
docker compose exec -T api bin/rails 'city:backup[maringa]'
```
Guarde os caminhos. Sem backup não se segue.

- [ ] **Step 3: Suspender, migrar, retomar — uma por vez**

```bash
docker compose exec -T api bin/rails 'city:suspend[curitiba]'
docker compose exec -T api bin/rails 'city:rekey[curitiba]'
docker compose exec -T api bin/rails 'city:resume[curitiba]'
```
Depois o mesmo para `maringa`. Registre as contagens impressas.

- [ ] **Step 4: Provar leitura e busca**

```bash
docker compose exec -T api bin/rails runner '
city = City.find_by!(slug: "curitiba")
CityConnection.with(city) do
  c = Conversation.first
  puts "phone_len=#{c.phone.to_s.length} lookup=#{Conversation.where(phone: c.phone).count}"
end'
```
Esperado: `phone_len` maior que zero e `lookup=1`. **Não imprima o telefone.**

- [ ] **Step 5: Provar o isolamento**

```bash
docker compose exec -T api bin/rails runner '
a = City.find_by!(slug: "curitiba"); b = City.find_by!(slug: "maringa")
raw = CityConnection.with(a) { InboundMessage.connection.select_value("SELECT raw FROM inbound_messages LIMIT 1") }
begin
  ActiveRecord::Encryption.with_encryption_context(**CityEncryption.context_properties(b)) do
    ActiveRecord::Encryption.encryptor.decrypt(raw)
  end
  puts "FALHOU: material de maringa decifrou dado de curitiba"
rescue ActiveRecord::Encryption::Errors::Decryption
  puts "OK: material de maringa NAO decifra dado de curitiba"
end'
```
Esperado: a linha `OK:`.

- [ ] **Step 6: Suíte completa**

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
```
Expected: 0 failures. Sem commit — a entrega é o relatório.

---

### Task 7: runbook de custódia, rotação e restauração

**Files:**
- Modify: `apps/api/README.md` (seção nova, antes de "## Worker por cidade (Plano 5)")

- [ ] **Step 1: Escrever a seção**

```markdown
## Chave de cifra por cidade (Plano 7)

Cada cidade cifra os próprios dados com uma chave **derivada** de `chave da plataforma + cities.encryption_key`.

- **Dá:** ciphertext de uma cidade não é legível nem comparável com o de outra; um dump roubado sozinho não abre nada;
  o mesmo telefone em duas cidades tem ciphertext diferente em cada uma.
- **Não dá:** isolamento contra quem tem a chave da plataforma — ela deriva todas. `ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY`
  e `..._DETERMINISTIC_KEY` são os segredos de maior valor do sistema: custódia separada, rotação própria, acesso restrito.

Atributos determinísticos (`Conversation#phone`, `Author#token`) usam `CityDeterministicKeyProvider`, que resolve a chave
pela cidade em `Current.city` a cada operação — o contexto de cifra do Rails não alcança esse caso. Ler ou escrever esses
atributos fora de `CityConnection.with` levanta `CityEncryption::MissingKey`, de propósito.

**Migrar uma cidade** (uma vez, por cidade) e **rotacionar** (vazamento suspeito ou política):

```bash
rails 'city:backup[slug]'
rails 'city:suspend[slug]'
rails 'city:rekey[slug]'        # ou city:rotate_key[slug], que gera material novo antes de reescrever
rails 'city:resume[slug]'
```

A cidade precisa estar suspensa porque, entre ler uma linha com o material antigo e gravá-la com o novo, uma busca
determinística de outro processo usaria a chave errada. `city:rotate_key` restaura o material anterior no catálogo se a
reescrita falhar no meio.

**Restaurar um dump.** Não existe `city:restore` ainda. Um dump só é restaurável **na cidade de origem, com a
`encryption_key` dela intacta** — o dump carrega ciphertext. Restaurar numa cidade cujo catálogo tem outro material
devolve dado ilegível **sem erro**. Antes de restaurar, confirme que a linha do catálogo é a mesma de quando o dump foi
tirado; guarde essa informação junto do arquivo.
```

- [ ] **Step 2: Conferir contra o código**

Leia `lib/tasks/city.rake` e confirme nomes de tarefas, exigência de cidade suspensa e a mensagem de rollback do `rotate_key`. Corrija o que divergir.

- [ ] **Step 3: Commit**

```bash
cd apps/api && /opt/homebrew/bin/git add README.md
/opt/homebrew/bin/git commit -m "Document per-city key custody, rotation and the restore limitation

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Definition of Done

- [ ] Material de uma cidade não decifra dado de outra (spec e prova em dev).
- [ ] Busca determinística por telefone não cruza cidades e continua funcionando dentro da cidade.
- [ ] Acesso a atributo determinístico fora de uma cidade levanta `MissingKey` em vez de usar a chave global.
- [ ] `curitiba` e `maringa` migradas, com backup antes e contagens registradas.
- [ ] `city:rekey` e `city:rotate_key` recusam cidade não suspensa; `rotate_key` faz rollback do material se a reescrita falhar.
- [ ] Nenhuma saída contém material de chave.
- [ ] Suíte completa verde.
- [ ] README descreve custódia, rotação, a janela de suspensão e o limite do restore.

## NÃO faz

- **`city:restore`** — decisão do usuário; plano posterior.
- **Cofre externo** (1Password/KMS) — a derivação a partir da chave da plataforma foi a escolha.
- **Rotação da chave da plataforma** — procedimento próprio, fora daqui.
- **Colunas do banco de plataforma** (`City#database_url`, `City#encryption_key`, `Operator#otp_secret`, `CityChannel#access_token`) — seguem na chave global.
- **Formulário de provisionamento do console** e **gates operacionais** — Plano 8.
- **`extend_queries`** do Rails — experimental na própria gem; a migração com cidade suspensa substitui a necessidade.

## Riscos

1. **A derivação não isola de quem tem a chave da plataforma.** Decisão tomada; o ganho é separação entre cidades e inutilidade de um dump isolado. Mudar o modelo de ameaça exige tirar a chave do catálogo — outro plano.
2. **`Current.city` vira dependência de leitura de dois atributos.** Qualquer caminho que leia `phone` ou `token` fora de `CityConnection.with` passa a levantar. É o comportamento desejado (falha fechada), mas a Task 3 vai revelar quantos caminhos existem — e o `ReencryptionJob` atual, que roda por `EachCityJob`, precisa continuar passando.
3. **Janela de suspensão por cidade.** Em dev são 50 conversas e 56 mensagens; em produção cresce com o volume, em lotes de 200.
4. **Provedor determinístico memoizado por thread.** A chave de cache inclui o material, então a rotação não reaproveita provedor velho; se um dia a memoização virar global por processo, isso quebra.
5. **Backup antigo não é restaurável depois da migração** sem o material da época. Enquanto não existir `city:restore`, isso é só disciplina de runbook — nada no código impede o engano.
6. **A suíte vai quebrar na Task 3** e cada quebra é informação, não obstáculo. A regra é corrigir o spec, nunca relaxar a asserção nem contornar o `MissingKey`.
