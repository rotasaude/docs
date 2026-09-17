# Plano 1 — Ambiente staging (API de manutenção, fatia 1)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `RAILS_ENV=staging` passa a existir como ensaio de produção. Staging tem as mesmas proteções de produção (cookie `Secure`, TLS no banco da cidade, env var obrigatória, HostAuthorization), credentials próprias e um boot real verificado na CI.

**Architecture:** um predicado único, `Rota.deployed?`, em `lib/rota.rb`, substitui todo `Rails.env.production?` usado no sentido de "ambiente publicado", e um spec de arquitetura impede que ele volte. `config/environments/staging.rb` herda de `production.rb`, e cada YAML por ambiente declara `staging` como alias de `production`, com um spec de paridade. Um initializer recusa staging lendo o arquivo de credentials compartilhado, e um job de CI sobe `RAILS_ENV=staging` de verdade, sem banco.

**Tech Stack:** Rails 8.1.3 (API), RSpec, Solid Queue, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-09-17-maintenance-graphql-api-design.md` (§4 Ambiente staging e isolamento, §10 Paridade de staging, §11 fatia 1).

**Planos seguintes:** 2 (fundação: mantenedores, sessão, trava de ativação, auditoria), 3 (tokens), 4 (leitura), 5 (mutations por módulo), 6 (contrato). Cada um é escrito depois que o anterior for mergeado.

## Decisões do usuário (2026-09-17, não re-perguntar)

1. **`RAILS_ENV=staging` explícito**, não `production` com uma variável extra.
2. **Isolamento total entre ambientes:** bancos, roles, chaves, credentials, segredos externos e contas. Staging nunca recebe dump de produção.
3. **Staging é ensaio de produção:** mesma imagem e mesma configuração. Só muda o que precisa mudar.

## Decisões minhas na escrita (cada uma com o custo se estiver errada)

4. **YAML por alias (`staging: *production`)**, não cópia. Uma cópia divergiria na primeira mudança de produção (ex.: threads do worker `urgent`), e o spec de paridade só avisaria depois. Custo se errado: se staging um dia precisar de valor próprio, o alias vira stanza explícita e o spec de paridade ganha exceção.
5. **Trava de credentials só para staging.** Produção hoje lê `config/credentials.yml.enc`, o mesmo arquivo de development e test, embora `deploy/production/secrets` fale em `production.yml.enc`. Produção está fora do escopo da spec, então não é corrigida aqui, só registrada (seção "Fora deste plano"). Custo se errado: a lacuna de produção continua aberta até alguém pegá-la.
6. **Script de verificação de boot versionado** (`script/staging_boot_check.rb`), usado pela CI e à mão. O job `production-boot` tem a verificação inline, mas a de staging checa sete coisas e precisa rodar localmente antes do push. Custo se errado: um arquivo a mais em `script/`.
7. **Cookie `Secure` sem request spec próprio.** O spec de guarda garante que nenhum ponto pergunta `Rails.env.production?`, `Rota.deployed?` tem teste unitário, e o boot de staging confere o `CookieStore`. Os cookies `session_id` e `operator_session_id` exigiriam login real num ambiente não-test para serem observados. Custo se errado: uma regressão que troque `Rota.deployed?` por um literal `false` não seria pega. É improvável, e o guard pega a forma comum.

## Global Constraints

Valem para TODA task:

- **Nunca imprimir segredo:** conteúdo de `config/credentials/*.key`, `secret_key_base`, chaves de AR Encryption, `report_signing_key`, senhas. Nomes de chave, contagens, booleanos e nomes de arquivo, sim.
- **Nunca commitar `config/credentials/staging.key`.** Antes de qualquer `git add` em `config/credentials/`, confirme com `/opt/homebrew/bin/git check-ignore config/credentials/staging.key` (tem que imprimir o caminho).
- **Commits em inglês, Conventional Commits** com o tipo por extenso (`feat`, `fix`, `refactor`, `test`, `ci`, `docs`, `chore`), terminando com a linha exata `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- **Comandos Ruby/Rails/rspec rodam no container**, a partir da raiz do monorepo `<raiz-do-monorepo>`: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Nunca no host: o Ruby do host é 3.4 e o Gemfile exige 3.3.6.
- **`git` do PATH está quebrado** nesta máquina (`/usr/bin/git` aborta com erro do Xcode): use `/opt/homebrew/bin/git`, a partir de `apps/api`. **Nunca dar push.**
- **Suíte completa:** `docker compose stop worker` antes e `docker compose start worker` depois, timeout ≥ 600000 ms, nunca duas suítes ao mesmo tempo, nunca `git stash`. **Registre a baseline** (exemplos e falhas) antes da Task 1. A última conhecida é 841 exemplos, 0 falhas.
- **Nada de operação destrutiva** em bancos de development ou test. Este plano não cria nem apaga banco.
- **Nunca rodar `start.sh`.**
- **Não confunda os comentários com o código:** o spec de guarda ignora linhas que começam com `#`, e comentários que citam "produção" em prosa continuam válidos.
- Passo marcado **BLOQUEADO (dono: usuário)** não é para o implementador executar: é para documentar, deixar pronto e parar, relatando o que falta.

## Fatos verificados (não re-descobrir)

- **`lib/` não é autoloaded.** `config/application.rb` não chama `autoload_lib`, e `lib/platform_hosts.rb` é `require`ado explicitamente por `config/environments/production.rb`, porque os arquivos de ambiente rodam antes do Zeitwerk.
- **Usos de `production` com sentido de "ambiente publicado"**, fora de `config/environments/`:
  - `app/controllers/concerns/authentication.rb:98`: `secure: Rails.env.production?`
  - `app/controllers/concerns/operator_authentication.rb:71`: `secure: Rails.env.production?`
  - `config/application.rb:32`: `secure: Rails.env.production?` (`CookieStore`)
  - `app/services/city_database.rb:53` (sslmode), `:151` (`CITY_DATABASE_HOST` obrigatório), `:157` (porta), `:161` (`PROVISIONER_DATABASE_URL` obrigatório)
  - `lib/platform_hosts.rb:37`: `return [] unless env.to_s == "production"`
  - `db/seeds.rb:21`: `if Rails.env.production?` e `db/seeds.rb:119`: `!Rails.env.production?`
- **`config/environments/production.rb`** termina com `config.hosts += PlatformHosts.for("production")`, com o literal. Em staging, que herda o arquivo, o literal esconderia o ambiente real.
- **YAMLs com stanza `production`:** `database.yml`, `cache.yml`, `queue.yml`, `queue_platform.yml`, `recurring.yml`, `recurring_platform.yml`. Em todos, `production:` é **a última stanza** do arquivo. `storage.yml` não tem stanza por ambiente.
- **Specs que hoje stubam `Rails.env.production?`** e deixam de exercitar o código depois da troca: `spec/services/city_database_spec.rb:258-265` ("requires PROVISIONER_DATABASE_URL in production") e o describe `CityDatabase, ".url_for in production"` (`:270-292`).
- **Specs que iteram ambientes:** `spec/config/solid_queue_configuration_spec.rb` (`%w[development production]` em `:15` e `:64`), `spec/architecture/database_config_parity_spec.rb`, `spec/architecture/host_authorization_spec.rb`.
- **Credentials:** só existe `config/credentials.yml.enc`, com as chaves `secret_key_base`, `active_record_encryption.{primary_key,deterministic_key,key_derivation_salt}`, `report_signing_key`, `policy.version`, `policy.v1.text`. `Rails.application.config.credentials.content_path` aponta para `config/credentials/<env>.yml.enc` se existir, senão para `config/credentials.yml.enc`. O `.gitignore` já ignora `/config/credentials/*.key`.
- **Container `api`:** `apps/api` montado em `/rails`, `RAILS_ENV=development`, sem `DATABASE_URL` e sem `RAILS_MASTER_KEY` no ambiente.
- **CI** (`.github/workflows/ci.yml`): jobs `rspec` e `production-boot`. O segundo sobe `RAILS_ENV=production` com URLs inertes (porta 1) e `bin/rails runner`, sem tocar banco.

---

## File Structure

**apps/api**
- Create: `lib/rota.rb` (predicados de ambiente e trava de credentials), `config/environments/staging.rb`, `config/initializers/00_credentials_isolation.rb`, `config/credentials/staging.yml.enc`, `script/staging_boot_check.rb`
- Create (specs): `spec/lib/rota_spec.rb`, `spec/architecture/deployed_environment_guard_spec.rb`, `spec/architecture/staging_environment_parity_spec.rb`, `spec/config/ci_workflow_spec.rb`
- Modify: `config/application.rb`, `config/environments/production.rb`, `app/controllers/concerns/authentication.rb`, `app/controllers/concerns/operator_authentication.rb`, `app/services/city_database.rb`, `lib/platform_hosts.rb`, `db/seeds.rb`, `config/{database,cache,queue,queue_platform,recurring,recurring_platform}.yml`, `.github/workflows/ci.yml`, `deploy/SECRETS.md`
- Modify (specs): `spec/services/city_database_spec.rb`, `spec/architecture/host_authorization_spec.rb`, `spec/architecture/database_config_parity_spec.rb`, `spec/config/solid_queue_configuration_spec.rb`

Branch: `feat/staging-environment` em `apps/api`, criada a partir de `main`.

---

### Task 1: `Rota.deployed?` e a trava de credentials

**Files:**
- Create: `lib/rota.rb`
- Modify: `config/application.rb:5` (logo depois de `Bundler.require(*Rails.groups)`)
- Test: `spec/lib/rota_spec.rb`

**Interfaces:**
- Consumes: nada.
- Produces:
  - `Rota::DEPLOYED_ENVS` → `%w[production staging]`
  - `Rota.deployed?(env = Rails.env) → Boolean`: aceita `String`, `Symbol` ou `ActiveSupport::EnvironmentInquirer`.
  - `Rota::ISOLATED_CREDENTIALS_ENVS` → `%w[staging]`
  - `Rota::SharedCredentials < StandardError`
  - `Rota.check_credentials!(env: Rails.env, content_path: Rails.application.config.credentials.content_path) → nil`: levanta `Rota::SharedCredentials` quando `env` está em `ISOLATED_CREDENTIALS_ENVS` e o basename de `content_path` não é `"#{env}.yml.enc"`.
  - `Rota` está carregado em todo processo Rails (inclusive em `config/environments/*.rb` e `db/seeds.rb`), via `require_relative` em `config/application.rb`.

- [ ] **Step 1: Escrever o spec que falha**

`spec/lib/rota_spec.rb`:

```ruby
require "rails_helper"

# Spec da API de manutenção §4: staging é ensaio de produção. "Ambiente
# publicado" é uma pergunta só — Rota.deployed? — e staging precisa de
# credentials próprias, sem cair em silêncio no arquivo compartilhado.
RSpec.describe Rota do
  describe ".deployed?" do
    it "is true only for the environments that run on shared infrastructure" do
      expect(described_class.deployed?("production")).to be(true)
      expect(described_class.deployed?("staging")).to be(true)
      expect(described_class.deployed?("development")).to be(false)
      expect(described_class.deployed?("test")).to be(false)
    end

    it "accepts the Rails environment inquirer and defaults to the current environment" do
      expect(described_class.deployed?(ActiveSupport::EnvironmentInquirer.new("staging"))).to be(true)
      expect(described_class.deployed?(:production)).to be(true)
      expect(described_class.deployed?).to be(false)
    end
  end

  describe ".check_credentials!" do
    it "refuses staging reading the shared credentials file" do
      expect do
        described_class.check_credentials!(env: "staging", content_path: Pathname("/rails/config/credentials.yml.enc"))
      end.to raise_error(Rota::SharedCredentials, %r{config/credentials/staging\.yml\.enc})
    end

    it "accepts staging reading its own credentials file" do
      expect do
        described_class.check_credentials!(env: "staging", content_path: Pathname("/rails/config/credentials/staging.yml.enc"))
      end.not_to raise_error
    end

    it "refuses staging reading another environment's own file" do
      expect do
        described_class.check_credentials!(env: "staging", content_path: "/rails/config/credentials/production.yml.enc")
      end.to raise_error(Rota::SharedCredentials)
    end

    it "does not judge the environments outside the isolated list" do
      %w[development test production].each do |env|
        expect do
          described_class.check_credentials!(env: env, content_path: Pathname("/rails/config/credentials.yml.enc"))
        end.not_to raise_error
      end
    end
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/lib/rota_spec.rb`
Expected: FAIL com `NameError: uninitialized constant Rota`.

- [ ] **Step 3: Implementar `lib/rota.rb`**

```ruby
# Predicados de ambiente que valem para a aplicação inteira (spec da API de
# manutenção §4).
#
# Fica em lib/ e é `require`ado por config/application.rb, não autoloaded:
# config/application.rb e config/environments/*.rb rodam antes do Zeitwerk, e
# ambos perguntam Rota.deployed? (mesmo motivo de lib/platform_hosts.rb).
module Rota
  # Ambientes publicados em infraestrutura compartilhada. staging é ensaio de
  # produção: tudo que endurece produção — cookie Secure, TLS no banco da cidade,
  # env var obrigatória, HostAuthorization — vale igual lá. Perguntar
  # `Rails.env.production?` com esse sentido deixaria staging sem essas proteções
  # sem aviso nenhum; spec/architecture/deployed_environment_guard_spec.rb proíbe.
  DEPLOYED_ENVS = %w[production staging].freeze

  # Ambientes que exigem config/credentials/<env>.yml.enc. Sem esse arquivo o Rails
  # cai, em silêncio, em config/credentials.yml.enc — as chaves de outro ambiente.
  # production entra aqui quando ganhar o arquivo próprio.
  ISOLATED_CREDENTIALS_ENVS = %w[staging].freeze

  class SharedCredentials < StandardError; end

  module_function

  def deployed?(env = Rails.env)
    DEPLOYED_ENVS.include?(env.to_s)
  end

  def check_credentials!(env: Rails.env, content_path: Rails.application.config.credentials.content_path)
    return unless ISOLATED_CREDENTIALS_ENVS.include?(env.to_s)

    expected = "#{env}.yml.enc"
    return if Pathname(content_path.to_s).basename.to_s == expected

    raise SharedCredentials, "#{env} está lendo #{Pathname(content_path.to_s).basename} — precisa de " \
                             "config/credentials/#{expected} com chaves próprias (isolamento entre ambientes)"
  end
end
```

- [ ] **Step 4: Carregar `Rota` no boot**

Em `config/application.rb`, logo depois da linha `Bundler.require(*Rails.groups)`:

```ruby

# Rota.deployed? (lib/rota.rb) é perguntado aqui mesmo (CookieStore, abaixo) e em
# config/environments/*.rb, que rodam antes do Zeitwerk.
require_relative "../lib/rota"
```

- [ ] **Step 5: Rodar e confirmar que passa**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/lib/rota_spec.rb`
Expected: PASS, 6 exemplos, 0 falhas.

- [ ] **Step 6: Commit**

```bash
cd apps/api
/opt/homebrew/bin/git add lib/rota.rb config/application.rb spec/lib/rota_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: add Rota.deployed? and staging credentials check

One predicate for "runs on shared infrastructure" (production or
staging) and a check that staging reads its own credentials file instead
of silently falling back to the shared one.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 2: Nenhuma proteção presa a `production`

**Files:**
- Create: `spec/architecture/deployed_environment_guard_spec.rb`
- Modify: `app/controllers/concerns/authentication.rb:98`, `app/controllers/concerns/operator_authentication.rb:71`, `config/application.rb` (`CookieStore`, `secure:`), `app/services/city_database.rb:47-53,151,157,161`, `lib/platform_hosts.rb:1-16,37`, `config/environments/production.rb` (última linha de `config.hosts`), `db/seeds.rb:21,119`
- Modify (specs): `spec/services/city_database_spec.rb:14-18,258-292`, `spec/architecture/host_authorization_spec.rb`

**Interfaces:**
- Consumes: `Rota.deployed?(env = Rails.env)` (Task 1).
- Produces: `PlatformHosts.for(env)` devolve os hosts para qualquer `env` em que `Rota.deployed?(env)` é verdadeiro, e `[]` nos demais. `production.rb` passa `Rails.env`, então staging herda hosts do seu próprio template.

- [ ] **Step 1: Escrever o spec de guarda**

`spec/architecture/deployed_environment_guard_spec.rb`:

```ruby
require "rails_helper"

# Spec da API de manutenção §4: staging roda com RAILS_ENV=staging e precisa de
# tudo que endurece produção. `Rails.env.production?` com o sentido de "ambiente
# publicado" deixa staging sem cookie Secure, sem TLS no banco da cidade e sem
# HostAuthorization — e a suíte, que roda em test, não percebe. A pergunta certa é
# Rota.deployed? (lib/rota.rb). config/environments/ fica de fora: ali cada
# arquivo É o ambiente.
RSpec.describe "Deployed environment guard" do
  def pattern
    /Rails\.env\.production\?|==\s*["']production["']|["']production["']\s*==/
  end

  def scanned_files
    Dir.chdir(Rails.root) do
      files = %w[app config lib db].flat_map { |root| Dir.glob("#{root}/**/*.{rb,rake}") }
      files += Dir.glob("bin/*").select { |path| File.file?(path) }
      files.reject { |path| path.start_with?("config/environments/") }.sort
    end
  end

  def offending_lines
    Dir.chdir(Rails.root) do
      scanned_files.flat_map do |path|
        File.readlines(path, encoding: "UTF-8").each_with_index.filter_map do |line, i|
          next if line.lstrip.start_with?("#")

          "#{path}:#{i + 1}: #{line.strip}" if line.match?(pattern)
        end
      end
    end
  end

  it "asks Rota.deployed? instead of tying a protection to the production environment" do
    expect(offending_lines).to eq([])
  end

  it "the pattern catches the forms it exists for" do
    [
      "secure: Rails.env.production?",
      'return [] unless env.to_s == "production"',
      "raise ConfigMissing if 'production' == env"
    ].each { |sample| expect(sample).to match(pattern) }

    expect("DEPLOYED_ENVS = %w[production staging].freeze").not_to match(pattern)
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha, listando os 10 pontos**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/architecture/deployed_environment_guard_spec.rb`
Expected: FAIL no primeiro exemplo. O diff lista exatamente estas 10 linhas (e nenhuma outra):
`app/controllers/concerns/authentication.rb:98`, `app/controllers/concerns/operator_authentication.rb:71`, `app/services/city_database.rb:53`, `:151`, `:157`, `:161`, `config/application.rb` (linha do `secure:` do `CookieStore`), `db/seeds.rb:21`, `db/seeds.rb:119`, `lib/platform_hosts.rb:37`.
Se aparecer uma 11ª, pare e reporte: é um ponto que este plano não conhece.

- [ ] **Step 3: Mudar os specs de `CityDatabase` para ambiente real, em production e staging**

Os stubs de `Rails.env.production?` deixariam de exercitar o código. Troque pelo ambiente inteiro.

Em `spec/services/city_database_spec.rb`, substitua o comentário das linhas 14–18 por:

```ruby
  # The provisioner example below stubs Rails.env for its own assertion; RSpec
  # mocks teardown runs after every after-hook, so the stub is still active here
  # for that example. Nothing was ever provisioned under it (that example never
  # calls ensure!), so ProvisionerMissing there means "nothing to clean up," not a
  # real cleanup failure.
```

Substitua o exemplo `it "requires PROVISIONER_DATABASE_URL in production" do ... end` por:

```ruby
  %w[production staging].each do |env|
    it "requires PROVISIONER_DATABASE_URL in #{env}" do
      allow(Rails).to receive(:env).and_return(ActiveSupport::EnvironmentInquirer.new(env))
      allow(ENV).to receive(:[]).and_call_original
      allow(ENV).to receive(:[]).with("PROVISIONER_DATABASE_URL").and_return(nil)

      expect { described_class.provisioner_url }.to raise_error(CityDatabase::ProvisionerMissing)
    end
  end
```

Substitua o comentário e o describe inteiro `RSpec.describe CityDatabase, ".url_for in production" do ... end` por:

```ruby
# The web process builds the city URL in every deployed environment without the
# worker-only provisioner secret. Its own describe, with no drop hook: the
# Rails.env stub is still active in `after` hooks.
%w[production staging].each do |env|
  RSpec.describe CityDatabase, ".url_for in #{env}" do
    before do
      allow(Rails).to receive(:env).and_return(ActiveSupport::EnvironmentInquirer.new(env))
      allow(ENV).to receive(:[]).and_call_original
      allow(ENV).to receive(:[]).with("PROVISIONER_DATABASE_URL").and_return(nil)
      allow(ENV).to receive(:[]).with("CITY_DATABASE_PORT").and_return(nil)
      allow(ENV).to receive(:[]).with("CITY_DATABASE_SSLMODE").and_return(nil)
    end

    it "uses CITY_DATABASE_HOST, port 5432 and sslmode=require, never the provisioner URL" do
      allow(ENV).to receive(:[]).with("CITY_DATABASE_HOST").and_return("db.example")

      expect(described_class.url_for(slug: "curitiba", password: "abc123"))
        .to eq("postgres://rota_city_curitiba:abc123@db.example:5432/rota_saude_city_curitiba?sslmode=require")
    end

    it "requires CITY_DATABASE_HOST" do
      allow(ENV).to receive(:[]).with("CITY_DATABASE_HOST").and_return("")

      expect { described_class.url_for(slug: "curitiba", password: "abc123") }
        .to raise_error(CityDatabase::ConfigMissing, "CITY_DATABASE_HOST ausente")
    end
  end
end
```

- [ ] **Step 4: Cobrir staging em `PlatformHosts`**

Em `spec/architecture/host_authorization_spec.rb`, substitua os dois primeiros exemplos por:

```ruby
  %w[production staging].each do |env|
    it "declares the platform domain and the city wildcard in #{env}" do
      with_production_template do
        expect(described_class.for(env)).to eq([ "rota-saude.example", ".rota-saude.example" ])
      end
    end
  end

  it "stays empty outside the deployed environments, where the harness uses synthetic hosts" do
    with_production_template do
      expect(described_class.for("test")).to eq([])
      expect(described_class.for("development")).to eq([])
    end
  end
```

- [ ] **Step 5: Rodar os specs alterados e confirmar que falham pelo motivo certo**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_database_spec.rb spec/architecture/host_authorization_spec.rb`
Expected: FAIL nos exemplos de **staging** (`url_for` sem `sslmode=require` e com a porta de dev, `ConfigMissing` e `ProvisionerMissing` não levantados, `PlatformHosts.for("staging")` devolvendo `[]`). Os exemplos de production passam.

- [ ] **Step 6: Trocar os 10 pontos por `Rota.deployed?`**

`app/controllers/concerns/authentication.rb:98` e `app/controllers/concerns/operator_authentication.rb:71`:

```ruby
      secure: Rota.deployed?
```

`config/application.rb`, no `CookieStore`:

```ruby
                          secure: Rota.deployed?
```

`app/services/city_database.rb`, comentário de `url_for` (linhas 47–51):

```ruby
    # URL que vai para cities.database_url: role e banco da cidade. Servidor e TLS
    # vêm de CITY_DATABASE_HOST (obrigatória em ambiente publicado — production e
    # staging), CITY_DATABASE_PORT e CITY_DATABASE_SSLMODE (default require em
    # ambiente publicado) — nunca da credencial do provisioner, que o processo web
    # não recebe. Fora deles, cai em DATABASE_HOST/DATABASE_PORT e sem sslmode.
```

Linha 53:

```ruby
      sslmode = ENV["CITY_DATABASE_SSLMODE"].presence || (Rota.deployed? ? "require" : nil)
```

Linha 151:

```ruby
      raise ConfigMissing, "CITY_DATABASE_HOST ausente" if Rota.deployed?
```

Linha 157:

```ruby
      ENV["CITY_DATABASE_PORT"].presence || (Rota.deployed? ? "5432" : ENV.fetch("DATABASE_PORT", "5432"))
```

Linha 161:

```ruby
      raise ProvisionerMissing, "PROVISIONER_DATABASE_URL ausente" if Rota.deployed?
```

`lib/platform_hosts.rb`, linha 37:

```ruby
    return [] unless Rota.deployed?(env)
```

E, no comentário do topo, troque a frase `Lista VAZIA desliga o middleware: é por isso que produção precisa declarar` por `Lista VAZIA desliga o middleware: é por isso que todo ambiente publicado (Rota.deployed?) precisa declarar`.

`config/environments/production.rb`, última linha de hosts:

```ruby
  # Rails.env, não o literal: config/environments/staging.rb herda este arquivo.
  config.hosts += PlatformHosts.for(Rails.env)
```

`db/seeds.rb:21`:

```ruby
if Rota.deployed?
  warn "[seeds] pulando: seeds de dev não rodam em ambiente publicado (#{Rails.env})"
```

`db/seeds.rb:119`:

```ruby
if ENV["SEED_DASHBOARD_DEMO"] == "1" && !Rota.deployed?
```

- [ ] **Step 7: Rodar os specs da task e confirmar que passam**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/architecture/deployed_environment_guard_spec.rb spec/services/city_database_spec.rb spec/architecture/host_authorization_spec.rb spec/architecture/cookie_domain_spec.rb spec/lib/rota_spec.rb`
Expected: PASS, 0 falhas.

- [ ] **Step 8: Suíte completa**

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
```
Expected: baseline + os exemplos novos (6 de `rota_spec`, 2 do guard, +3 de `city_database_spec`, +1 de `host_authorization_spec`), 0 falhas.

- [ ] **Step 9: Commit**

```bash
cd apps/api
/opt/homebrew/bin/git add spec/architecture/deployed_environment_guard_spec.rb \
  app/controllers/concerns/authentication.rb app/controllers/concerns/operator_authentication.rb \
  config/application.rb app/services/city_database.rb lib/platform_hosts.rb \
  config/environments/production.rb db/seeds.rb \
  spec/services/city_database_spec.rb spec/architecture/host_authorization_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
refactor: tie production hardening to Rota.deployed?

Secure cookies, sslmode=require for city databases, mandatory
CITY_DATABASE_HOST and PROVISIONER_DATABASE_URL, HostAuthorization hosts
and the dev-seed skip asked Rails.env.production?, so a staging
environment would silently lose all of them. They now ask
Rota.deployed?, and an architecture spec keeps production-only checks
out of app, config, lib, db and bin.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 3: `RAILS_ENV=staging` como cópia fiel de production

**Files:**
- Create: `config/environments/staging.rb`, `spec/architecture/staging_environment_parity_spec.rb`
- Modify: `config/database.yml`, `config/cache.yml`, `config/queue.yml`, `config/queue_platform.yml`, `config/recurring.yml`, `config/recurring_platform.yml`
- Modify (specs): `spec/architecture/database_config_parity_spec.rb`, `spec/config/solid_queue_configuration_spec.rb:15,64`

**Interfaces:**
- Consumes: `Rota.deployed?` e `PlatformHosts.for(Rails.env)` (Tasks 1–2).
- Produces: `config/environments/staging.rb` e a stanza `staging` em cada YAML por ambiente, idêntica à de `production`.

- [ ] **Step 1: Escrever o spec de paridade**

`spec/architecture/staging_environment_parity_spec.rb`:

```ruby
require "rails_helper"

# Spec da API de manutenção §4: staging é ensaio de produção. Cada arquivo de
# config com stanza por ambiente declara staging IGUAL a production — uma stanza
# esquecida só apareceria no boot de staging (AdapterNotSpecified, cidade sem
# worker, recorrência que não roda), longe da suíte, que roda em test.
RSpec.describe "Staging environment parity" do
  def parsed(file)
    ActiveSupport::ConfigurationFile.parse(Rails.root.join("config/#{file}.yml"))
  end

  %w[database cache queue queue_platform recurring recurring_platform].each do |file|
    it "declares config/#{file}.yml staging exactly as production" do
      config = parsed(file)

      expect(config).to have_key("staging"), "config/#{file}.yml: sem stanza staging"
      expect(config["staging"]).to eq(config["production"])
    end
  end

  it "builds config/environments/staging.rb on top of production.rb" do
    code = Rails.root.join("config/environments/staging.rb").read

    expect(code).to match(/^require_relative "production"$/)
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/architecture/staging_environment_parity_spec.rb`
Expected: FAIL nos 7 exemplos (6 com "sem stanza staging" e 1 com `Errno::ENOENT` para `staging.rb`).

- [ ] **Step 3: Criar `config/environments/staging.rb`**

```ruby
# staging: ensaio de produção (spec da API de manutenção §4). Mesma configuração
# de production — inclusive config.hosts (PlatformHosts.for(Rails.env)) e SMTP —,
# com banco, chaves, credentials e segredos próprios vindos do ambiente e de
# config/credentials/staging.yml.enc. Só entra aqui o que PRECISA ser diferente;
# hoje, nada.
require_relative "production"
```

- [ ] **Step 4: Declarar `staging` como alias de `production` nos seis YAMLs**

Em cada um de `config/database.yml`, `config/cache.yml`, `config/queue.yml`, `config/queue_platform.yml`, `config/recurring.yml` e `config/recurring_platform.yml`:

1. Troque a linha `production:` por `production: &production`.
2. No fim do arquivo (`production` é a última stanza em todos), acrescente:

```yaml

# staging é ensaio de produção: mesma configuração, por construção (spec da API
# de manutenção §4; spec/architecture/staging_environment_parity_spec.rb).
staging: *production
```

- [ ] **Step 5: Incluir staging nos specs que iteram ambientes**

Em `spec/architecture/database_config_parity_spec.rb`, acrescente o exemplo depois do primeiro:

```ruby
  it "declares the same database keys under staging as under production" do
    expect(raw["staging"].keys).to match_array(raw["production"].keys)
  end
```

E troque `%w[development test production].each do |env|` por `%w[development test production staging].each do |env|`.

Em `spec/config/solid_queue_configuration_spec.rb`, nas linhas 15 e 64, troque `%w[development production]` por `%w[development production staging]`.

- [ ] **Step 6: Rodar e confirmar que passa**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/architecture/staging_environment_parity_spec.rb spec/architecture/database_config_parity_spec.rb spec/config/solid_queue_configuration_spec.rb spec/boot/eager_load_spec.rb`
Expected: PASS, 0 falhas.

- [ ] **Step 7: Commit**

```bash
cd apps/api
/opt/homebrew/bin/git add config/environments/staging.rb config/database.yml config/cache.yml config/queue.yml \
  config/queue_platform.yml config/recurring.yml config/recurring_platform.yml \
  spec/architecture/staging_environment_parity_spec.rb spec/architecture/database_config_parity_spec.rb \
  spec/config/solid_queue_configuration_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: add staging environment mirroring production

config/environments/staging.rb builds on production.rb, and every
per-environment YAML declares staging as an alias of production, so the
rehearsal environment cannot drift from what ships.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 4: Credentials próprias e boot real de staging na CI

**Files:**
- Create: `config/initializers/00_credentials_isolation.rb`, `script/staging_boot_check.rb`, `config/credentials/staging.yml.enc` (gerado), `spec/config/ci_workflow_spec.rb`
- Modify: `.github/workflows/ci.yml` (novo job `staging-boot`), `deploy/SECRETS.md`
- Nunca commitar: `config/credentials/staging.key`

**Interfaces:**
- Consumes: `Rota.check_credentials!`, `Rota.deployed?` (Task 1); `config/environments/staging.rb` e stanzas `staging` (Task 3); `CityDatabase.url_for(slug:, password:)` (existente).
- Produces: `script/staging_boot_check.rb`, que imprime `STAGING_BOOT_OK hosts=[...]` e sai com 0, ou imprime `STAGING_BOOT_FAILED` com a lista de falhas e sai com código diferente de 0. Job de CI `staging-boot`.

- [ ] **Step 1: Escrever o spec do workflow**

`spec/config/ci_workflow_spec.rb`:

```ruby
require "rails_helper"

# Spec da API de manutenção §10: nada na suíte (RAILS_ENV=test) carrega staging.
# O job staging-boot é o único lugar em que o ambiente sobe de verdade antes do
# deploy — esta guarda impede que ele suma do workflow sem ninguém notar.
RSpec.describe "CI workflow" do
  let(:jobs) { YAML.load_file(Rails.root.join(".github/workflows/ci.yml")).fetch("jobs") }

  it "boots RAILS_ENV=staging for real through script/staging_boot_check.rb" do
    job = jobs.fetch("staging-boot")

    expect(job.dig("env", "RAILS_ENV")).to eq("staging")
    expect(job.fetch("steps").map { |step| step["run"].to_s }.join("\n"))
      .to include("bin/rails runner script/staging_boot_check.rb")
  end

  it "keeps booting RAILS_ENV=production" do
    expect(jobs.fetch("production-boot").dig("env", "RAILS_ENV")).to eq("production")
  end
end
```

- [ ] **Step 2: Rodar e confirmar que falha**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/config/ci_workflow_spec.rb`
Expected: FAIL no primeiro exemplo com `KeyError: key not found: "staging-boot"`.

- [ ] **Step 3: Criar o initializer**

`config/initializers/00_credentials_isolation.rb`:

```ruby
# Isolamento entre ambientes (spec da API de manutenção §4). Sem
# config/credentials/staging.yml.enc, o Rails lê config/credentials.yml.enc em
# silêncio — e staging subiria com as chaves de outro ambiente. Prefixo 00_: roda
# antes de active_record_encryption.rb, para falhar antes de qualquer chave ser
# lida. Ver lib/rota.rb.
Rota.check_credentials!
```

- [ ] **Step 4: Criar o script de verificação de boot**

`script/staging_boot_check.rb`:

```ruby
# Boot real de RAILS_ENV=staging, sem banco (spec da API de manutenção §4 e §10).
# Roda no job `staging-boot` da CI e à mão:
#   RAILS_ENV=staging bin/rails runner script/staging_boot_check.rb
# Nunca imprime segredo: só nomes, booleanos e a lista de hosts.
failures = []
check = ->(ok, message) { failures << message unless ok }

check.(Rails.env.staging?, "Rails.env é #{Rails.env}, não staging")
check.(Rota.deployed?, "Rota.deployed? é falso")

credentials_file = Pathname(Rails.application.config.credentials.content_path.to_s).basename.to_s
check.(credentials_file == "staging.yml.enc", "credentials vêm de #{credentials_file}, não de staging.yml.enc")

check.(Rails.application.config.hosts.any?, "config.hosts vazio: HostAuthorization inerte")
check.(Rails.application.middleware.map(&:klass).include?(ActionDispatch::HostAuthorization),
       "ActionDispatch::HostAuthorization não está instalado")

cookie_store = Rails.application.middleware.find { |middleware| middleware.klass == ActionDispatch::Session::CookieStore }
cookie_options = cookie_store&.args&.last
check.(cookie_options.is_a?(Hash) && cookie_options[:secure] == true, "CookieStore sem secure: true")

city_url = URI.parse(CityDatabase.url_for(slug: "curitiba", password: "inert"))
check.(city_url.query == "sslmode=require", "URL do banco da cidade sem sslmode=require")
check.(city_url.port == 5432, "URL do banco da cidade fora da porta 5432")

abort("STAGING_BOOT_FAILED\n- #{failures.join("\n- ")}") if failures.any?

puts "STAGING_BOOT_OK hosts=#{Rails.application.config.hosts.inspect}"
```

- [ ] **Step 5: Rodar o boot de staging e confirmar que o initializer recusa**

Da raiz do monorepo:

```bash
docker compose exec -T \
  -e RAILS_ENV=staging -e RAILS_LOG_LEVEL=warn \
  -e ROTA_APP_PASSWORD=inert -e ROTA_PLATFORM_PASSWORD=inert \
  -e CITY_PUBLIC_BASE_TEMPLATE='https://%{slug}.ci.rota-saude.example' \
  -e CITY_UNSET_DATABASE_URL=postgres://inert:inert@127.0.0.1:1/inert \
  -e PLATFORM_DATABASE_URL=postgres://inert:inert@127.0.0.1:1/inert \
  -e CITY_DATABASE_HOST=db.ci.rota-saude.example \
  api bin/rails runner script/staging_boot_check.rb
```

Expected: saída com código diferente de 0 e `Rota::SharedCredentials` citando `credentials.yml.enc` e `config/credentials/staging.yml.enc`. Esse é o RED da trava: sem o arquivo próprio, staging não sobe.

- [ ] **Step 6: Gerar `config/credentials/staging.yml.enc` com chaves novas**

Da raiz do monorepo. O script roda em development para copiar `policy`, que é o texto público do termo, não segredo. Todo o resto é gerado do zero. Não commite o script.

```bash
docker compose exec -T api bin/rails runner '
  require "securerandom"

  dir = Rails.root.join("config/credentials")
  content_path = dir.join("staging.yml.enc")
  key_path = dir.join("staging.key")
  abort "staging credentials already exist — refusing to overwrite" if content_path.exist? || key_path.exist?

  FileUtils.mkdir_p(dir)
  File.write(key_path, ActiveSupport::EncryptedFile.generate_key)
  File.chmod(0o600, key_path)

  data = {
    "secret_key_base" => SecureRandom.hex(64),
    "active_record_encryption" => {
      "primary_key" => SecureRandom.alphanumeric(32),
      "deterministic_key" => SecureRandom.alphanumeric(32),
      "key_derivation_salt" => SecureRandom.alphanumeric(32)
    },
    "report_signing_key" => SecureRandom.hex(64),
    "policy" => Rails.application.credentials.policy.to_h.deep_stringify_keys
  }

  ActiveSupport::EncryptedConfiguration.new(
    config_path: content_path, key_path: key_path, env_key: "RAILS_MASTER_KEY", raise_if_missing_key: true
  ).write(data.to_yaml)

  puts "wrote #{content_path.basename}; top-level keys: #{data.keys.join(", ")}"
'
```

Expected: `wrote staging.yml.enc; top-level keys: secret_key_base, active_record_encryption, report_signing_key, policy`.

Confirme que a chave está ignorada, a partir de `apps/api`:

```bash
/opt/homebrew/bin/git check-ignore config/credentials/staging.key
```

Expected: imprime `config/credentials/staging.key`. Se não imprimir, **pare**: a chave entraria no próximo commit.

- [ ] **Step 7: Confirmar que staging lê as próprias credentials, com as mesmas chaves de nível superior**

```bash
docker compose exec -T -e RAILS_ENV=staging \
  -e ROTA_APP_PASSWORD=inert -e ROTA_PLATFORM_PASSWORD=inert \
  -e CITY_PUBLIC_BASE_TEMPLATE='https://%{slug}.ci.rota-saude.example' \
  -e CITY_UNSET_DATABASE_URL=postgres://inert:inert@127.0.0.1:1/inert \
  -e PLATFORM_DATABASE_URL=postgres://inert:inert@127.0.0.1:1/inert \
  -e CITY_DATABASE_HOST=db.ci.rota-saude.example \
  api bin/rails runner '
    def names(hash, prefix = []) = hash.flat_map { |k, v| v.is_a?(Hash) ? names(v, prefix + [k]) : [(prefix + [k]).join(".")] }
    puts Pathname(Rails.application.config.credentials.content_path.to_s).basename
    puts names(Rails.application.credentials.config).sort
  '
```

Expected: `staging.yml.enc`, seguido de `active_record_encryption.deterministic_key`, `active_record_encryption.key_derivation_salt`, `active_record_encryption.primary_key`, `policy.v1.text`, `policy.version`, `report_signing_key`, `secret_key_base`, a mesma lista de nomes das credentials de development.

- [ ] **Step 8: Rodar o boot de staging e confirmar que passa**

Mesmo comando do Step 5.
Expected: `STAGING_BOOT_OK hosts=["ci.rota-saude.example", ".ci.rota-saude.example"]`, código de saída 0.

Se aparecer `CookieStore sem secure: true` mesmo com `secure: Rota.deployed?` em `config/application.rb`, o formato de `middleware.args` não é o esperado. Nesse caso, inspecione `cookie_store.args.inspect` e ajuste **só o script**, nunca a configuração, e reporte.

- [ ] **Step 9: Adicionar o job `staging-boot` na CI**

Em `.github/workflows/ci.yml`, acrescente no fim do arquivo, no mesmo nível de `production-boot`:

```yaml

  staging-boot:
    runs-on: ubuntu-latest
    # Spec da API de manutenção §4/§10: staging é ensaio de produção, com
    # RAILS_ENV=staging. Nada na suíte (RAILS_ENV=test) carrega esse ambiente —
    # uma stanza esquecida, credentials caindo no arquivo compartilhado ou uma
    # proteção presa a Rails.env.production? só apareceriam no deploy. Mesmo
    # desenho do production-boot: boot real, URLs inertes, nenhum banco tocado.
    # O que é verificado mora em script/staging_boot_check.rb (roda à mão também).
    env:
      RAILS_ENV: staging
      RAILS_MASTER_KEY: ${{ secrets.RAILS_STAGING_MASTER_KEY }}
      ROTA_APP_PASSWORD: inert
      ROTA_PLATFORM_PASSWORD: inert
      CITY_PUBLIC_BASE_TEMPLATE: "https://%{slug}.ci.rota-saude.example"
      CITY_UNSET_DATABASE_URL: "postgres://inert:inert@127.0.0.1:1/inert"
      PLATFORM_DATABASE_URL: "postgres://inert:inert@127.0.0.1:1/inert"
      CITY_DATABASE_HOST: db.ci.rota-saude.example
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - name: Boot RAILS_ENV=staging for real (no database connection)
        run: bin/rails runner script/staging_boot_check.rb
```

- [ ] **Step 10: Rodar o spec do workflow e confirmar que passa**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/config/ci_workflow_spec.rb`
Expected: PASS, 2 exemplos.

- [ ] **Step 11: Documentar a custódia de staging**

Em `deploy/SECRETS.md`, logo depois do parágrafo que começa com `Em produção os valores vêm do 1Password`, acrescente:

```markdown
### Staging

`RAILS_ENV=staging` é ensaio de produção e **não compartilha nenhum segredo** com os outros ambientes (spec da API de
manutenção §4). As credentials ficam em `config/credentials/staging.yml.enc`, versionado, com chaves próprias
(`secret_key_base`, `active_record_encryption.*`, `report_signing_key`). A chave que o decifra,
`config/credentials/staging.key`, **nunca** entra no git:

- no servidor de staging, vai como `RAILS_MASTER_KEY`;
- na CI, é o secret `RAILS_STAGING_MASTER_KEY` do repositório (job `staging-boot`);
- no cofre, fica no item `rails-master-key` do cofre `rota-saude-staging`.

Staging sem `staging.yml.enc` **não sobe** (`config/initializers/00_credentials_isolation.rb`): cair em
`config/credentials.yml.enc` seria usar as chaves de outro ambiente. Staging nunca recebe dump de produção.
```

- [ ] **Step 12: Suíte completa**

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
```
Expected: total da Task 2 + 7 (paridade) + 1 (`database_config_parity`) + os de `solid_queue_configuration` para staging + 2 (workflow), 0 falhas.

- [ ] **Step 13: Commit**

A partir de `apps/api`. Confira de novo que a chave está fora:

```bash
/opt/homebrew/bin/git check-ignore config/credentials/staging.key
/opt/homebrew/bin/git add config/initializers/00_credentials_isolation.rb script/staging_boot_check.rb \
  config/credentials/staging.yml.enc spec/config/ci_workflow_spec.rb .github/workflows/ci.yml deploy/SECRETS.md
/opt/homebrew/bin/git status --short config/credentials
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: give staging its own credentials and a real boot in CI

Staging refuses to boot on the shared credentials file, carries freshly
generated keys in config/credentials/staging.yml.enc, and a staging-boot
CI job verifies hosts, HostAuthorization, secure session cookies and
sslmode=require without touching a database.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

O `git status --short config/credentials` antes do commit deve listar **só** `A  config/credentials/staging.yml.enc`.

- [ ] **Step 14: BLOQUEADO (dono: usuário): guardar a chave de staging**

O implementador **não** executa este passo. Ele só relata que ele existe. O usuário:

1. Copia o conteúdo de `apps/api/config/credentials/staging.key` para o 1Password, cofre `rota-saude-staging`, item `rails-master-key`.
2. Cria o secret de Actions `RAILS_STAGING_MASTER_KEY` no repositório `rotasaude/api` com o mesmo valor.
3. Mantém o arquivo local só enquanto precisar rodar staging na máquina. Ele é ignorado pelo git.

Enquanto o secret não existir, o job `staging-boot` roda com `RAILS_MASTER_KEY` vazio. O Rails trata chave ausente como credentials vazias (`require_master_key` não está ligado), então o boot deve passar, mas isso **não foi verificado na CI**. Se o job falhar ao ler credentials antes do secret existir, a causa é essa, e o conserto é este passo, não o código.

---

## Verificação final do plano

- [ ] `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec`, com o worker parado: 0 falhas.
- [ ] O boot de staging (comando da Task 4, Step 5) imprime `STAGING_BOOT_OK`.
- [ ] `grep -rnE 'Rails\.env\.production\?' app config lib db bin | grep -v '^config/environments/'` não devolve nada.
- [ ] `/opt/homebrew/bin/git log --oneline main..` mostra 4 commits, e nenhum deles contém `staging.key`: `/opt/homebrew/bin/git log --stat main.. | grep -c staging.key` imprime `0`.

## Fora deste plano

- **Produção lê `config/credentials.yml.enc`, o mesmo arquivo de development e test.** `deploy/production/secrets` fala em `config/credentials/production.yml.enc`, que não existe. A correção é gerar `production.yml.enc` e incluir `production` em `Rota::ISOLATED_CREDENTIALS_ENVS`. Produção está fora do escopo da spec, então fica registrado aqui e não é feito.
- **Infraestrutura de staging:** `deploy/staging/deploy.yml` (Kamal), servidor, DNS de `*.staging.rotasaude.com.br`, Postgres e cofre. Não existe servidor de staging ainda. Quando existir, o spec `spec/config/deploy_hosts_spec.rb` ganha `staging` na lista de ambientes.
- **Tudo o que é GraphQL, mantenedor e auditoria:** Planos 2 a 6.
