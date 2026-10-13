# Rodada especial do módulo 19 — fatia R4: conexões do Postgres — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fechar o api#74 com uma conta explícita de conexões do Postgres para **5 cidades**, a configuração de produção derivada dela, um teste que trava regressão e um runbook.

**Architecture:** São três peças:
- uma calculadora pura em Ruby (`Ops::ConnectionBudget`), que conta o teto de conexões por parte a partir dos arquivos reais (`config/queue.yml`, `config/queue_platform.yml` e `deploy/production/deploy.yml`);
- uma spec de configuração que aplica a calculadora aos arquivos reais e falha se a conta passar dos limites configurados;
- a documentação: runbook no docs e README do api.

O pool de cidade ganha `idle_timeout`, para o web devolver as conexões das cidades que não atende.

**Tech Stack:** Rails 8, RSpec, Solid Queue, Kamal 2 (`deploy.yml`).

**Spec:** `docs/superpowers/specs/2026-10-13-module-19-special-round-design.md` (§5). Card: rotasaude/api#74.

## Modelo da conta

É o teto de pico. Cada pool conta o menor entre o seu tamanho e as threads que o usam.

**Constantes do modelo**, em `Ops::ConnectionBudget`, cada uma com comentário do porquê:
- `SOLID_QUEUE_EXTRA = 2`: o Solid Queue usa threads mais 2 (polling e heartbeat);
- `CONTROL_PROCESSES = 3`: supervisor, dispatcher e scheduler por supervisor;
- `CONTROL_CITY = 2` e `CONTROL_PLATFORM = 1`: conexões de cada processo de controle;
- `RESERVED = 20`: console, migrações, backup, `kamal app exec` e as `superuser_reserved_connections`.

**Web.** São `hosts × WEB_CONCURRENCY` processos, cada um com `T = RAILS_MAX_THREADS`.
- Base (plataforma e cache): `processos × T × 2`.
- Por cidade (`CityRecord` e shard do Solid Queue, os dois no banco da cidade): `processos × T × 2`.

**Worker por cidade.** Para cada worker do `config/queue.yml` com `t` threads:
- banco da cidade: `(t + 2) + t + t` (Solid Queue padrão, shard do Solid Queue e `CityRecord`);
- plataforma: `t`.

Mais os 3 processos de controle: 2 no banco da cidade e 1 na plataforma, cada um.

**Plataforma.** Para cada worker do `config/queue_platform.yml`: `(t + 2) + t` na plataforma, mais 3 processos de controle × 2.

**Totais:**
- `total(C) = web_base + C × (web_por_cidade + worker_por_cidade) + plataforma + RESERVED`;
- o teto por role de cidade, `role_por_cidade = web_por_cidade + worker_cidade_no_banco_da_cidade`, é o que o `CONNECTION LIMIT` do `rota_city_<slug>` precisa comportar.

**Com os arquivos de hoje** (2 hosts web × 2 Puma × 5 threads; workers de produção 5, 10, 3 e 3 threads; plataforma com 2):

| Parte | Conexões |
|---|---|
| Web, base | 40 |
| Web, por cidade | 40 |
| Worker por cidade, no banco da cidade | 3 × 21 + 4 × 2 + 3 × 2 = **77** |
| Worker por cidade, na plataforma | 21 + 3 = **24** |
| Plataforma | (2 + 2) + 2 + 3 × 2 = **12** |
| Reserva | 20 |
| `role_por_cidade` | 40 + 77 = **117**. O padrão de hoje (`CITY_ROLE_CONNECTION_LIMIT` = 100) é **menor que o pico de uma cidade** e precisa subir |
| `total(1)` | 213 |
| `total(4)` | 636 |
| `total(5)` | **777** |
| `total(10)` | 1482 |

**Configuração resultante** (folga de 20%, isto é, o uso precisa caber em 80% do teto):
- `max_connections ≥ 777 / 0,8 = 972`, arredondado para **1000**;
- `CITY_ROLE_CONNECTION_LIMIT ≥ 117 / 0,8 = 147`, arredondado para **150**.

## Global Constraints

- **Cidades do go-live: 5.** A spec de orçamento usa esse número.
- **Folga:** a conta para 5 cidades cabe em 80% do `max_connections`, e o pico de uma cidade cabe em 80% do `CITY_ROLE_CONNECTION_LIMIT`.
- **Pool do worker:** `RAILS_MAX_THREADS` do papel worker no `deploy.yml` é ≥ o maior `threads` de produção do `config/queue.yml` + 2.
- **`idle_timeout` do pool de cidade:** 60 s.
- **Escopo:**
  - sem PgBouncer (fica no runbook como próximo passo);
  - sem mexer na decisão do worker por cidade;
  - a prova com carga em staging vira gate de go-live, registrada no runbook e nos rollouts.
- **Testes:**
  - todo comportamento novo ganha um teste que falha antes;
  - a suíte do api passa (`docker compose stop worker` antes da suíte completa e `start` depois);
  - os bancos de teste usam o sufixo `ROTA_TEST_DB_SUFFIX=_r19`, que já existe.
- **Git:**
  - `/opt/homebrew/bin/git`;
  - Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Worktree do api** (memória `worktree-do-api-no-container`): `apps/api/.claude/r4`, no branch `ops/connection-budget` a partir de `origin/main`. Copie `config/master.key` para lá e rode no container com `docker exec -w /rails/.claude/r4 api-dev …`.

## Review Focus

1. **Alguém acrescenta um 5º worker no `config/queue.yml`, ou sobe threads.**
   - Esperado: a spec de orçamento falha na CI, dizendo quanto passou.
   - Prova: Task 2.
2. **`deploy.yml` sem a linha `max_connections`** (alguém troca o `cmd` do acessório).
   - Esperado: a spec falha com mensagem clara, e não passa em silêncio.
   - Prova: Task 2.
3. **Web atendendo as 5 cidades.**
   - Esperado: depois de 60 s sem uso de uma cidade, o pool dela solta as conexões ociosas.
   - Prova: Task 2, com `idle_timeout` na configuração resolvida.
4. **Pico de uma cidade acima do `CONNECTION LIMIT` da role.**
   - Esperado: a spec acusa, porque hoje 117 é maior que 100.
   - Prova: Task 2.
5. **Rake com cidades = 0 ou valor inválido.**
   - Esperado: mensagem de uso, sem stack trace.
   - Prova: Task 1.

---

### Task 0: Preparação

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api fetch -q && $G -C $R/apps/api worktree add -q $R/apps/api/.claude/r4 -b ops/connection-budget origin/main
cp $R/apps/api/config/master.key $R/apps/api/.claude/r4/config/master.key
docker exec -w /rails/.claude/r4 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec spec/config 2>&1 | tail -3
```

Expected: verde. Anote a linha de base no ledger.

---

### Task 1: Calculadora e rake

**Files:**
- Create: `app/services/ops/connection_budget.rb`, `lib/tasks/ops.rake`
- Test: `spec/services/ops/connection_budget_spec.rb`, `spec/tasks/ops_connection_budget_spec.rb` (siga o padrão de spec de rake que já existir em `spec/tasks/`; se não houver, teste o método `Ops::ConnectionBudget.report(cities)` que a rake chama, e registre um Ruling)

**Interfaces:**
- Produces:
  - `Ops::ConnectionBudget.new(web_hosts:, web_concurrency:, web_threads:, city_workers:, platform_workers:)`, em que `city_workers` e `platform_workers` são arrays de inteiros (threads de cada worker);
  - `#web_base`, `#web_per_city`, `#worker_city_db_per_city`, `#worker_platform_per_city`, `#platform`, `#role_per_city` e `#total(cities)` (todos Integer);
  - `Ops::ConnectionBudget.from_files(root: Rails.root, env: "production")`, que lê os três arquivos (o `deploy.yml` com `YAML.safe_load(..., aliases: true)`);
  - `Ops::ConnectionBudget.report(cities)`: texto com a tabela por parte e o total;
  - `Ops::ConnectionBudget::RESERVED = 20`.

- [ ] **Step 1: testes que falham** (`spec/services/ops/connection_budget_spec.rb`, `type: :service` ou sem tipo: a spec não toca banco):

```ruby
require "rails_helper"

RSpec.describe Ops::ConnectionBudget do
  subject(:budget) do
    described_class.new(web_hosts: 2, web_concurrency: 2, web_threads: 5,
                        city_workers: [ 5, 10, 3, 3 ], platform_workers: [ 2 ])
  end

  it "counts the web base and per-city pools at the thread ceiling" do
    expect(budget.web_base).to eq(40)
    expect(budget.web_per_city).to eq(40)
  end

  it "counts each city supervisor on the city database and on the platform" do
    expect(budget.worker_city_db_per_city).to eq(77)
    expect(budget.worker_platform_per_city).to eq(24)
    expect(budget.platform).to eq(12)
  end

  it "sums the role ceiling of one city" do
    expect(budget.role_per_city).to eq(117)
  end

  it "totals for 1, 4 and 10 cities with the reserve" do
    expect(budget.total(1)).to eq(213)
    expect(budget.total(4)).to eq(636)
    expect(budget.total(10)).to eq(1482)
  end

  it "reads the real production files" do
    from_files = described_class.from_files
    expect(from_files.web_threads).to eq(5)
    expect(from_files.city_workers).to eq([ 5, 10, 3, 3 ])
  end

  it "refuses a non-positive number of cities" do
    expect { budget.total(0) }.to raise_error(ArgumentError, /cidades/)
  end
end
```

  A rake `ops:connection_budget[cidades]`:
  - imprime o `report`;
  - sem argumento ou com valor inválido, imprime `uso: bin/rails 'ops:connection_budget[5]'` e sai com 1.

  Teste a saída com `expect { … }.to output(/total.*777/m).to_stdout`, chamando `Rake::Task["ops:connection_budget"].invoke("5")` depois de `Rails.application.load_tasks`, se for o padrão do repo.

  Run: `docker exec -w /rails/.claude/r4 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec spec/services/ops spec/tasks/ops_connection_budget_spec.rb`
  Expected: FAIL (constante não definida).

- [ ] **Step 2: implementar** `app/services/ops/connection_budget.rb` com `Data.define` ou classe simples, as constantes do modelo comentadas e as fórmulas da seção "Modelo da conta":
  - `total(cities)` levanta `ArgumentError, "cidades precisa ser inteiro positivo"` se `cities < 1`;
  - `from_files` pega:
    - `web_hosts`: `servers.web.hosts.size` do `deploy.yml`;
    - `web_concurrency` e `web_threads`: `env.clear.WEB_CONCURRENCY` e `RAILS_MAX_THREADS` do `deploy.yml`, em inteiro;
    - `city_workers`: `threads` de cada worker de `config/queue.yml[env]`;
    - `platform_workers`: o mesmo de `config/queue_platform.yml[env]`.

  Run: o mesmo do Step 1.
  Expected: PASS.

- [ ] **Step 3: commit** (`feat: add a Postgres connection budget calculator and rake task`).

---

### Task 2: Configuração e spec de orçamento

**Files:**
- Modify: `app/models/city_connection.rb` (`database_config`: `&idle_timeout=60`, sem sobrescrever um `idle_timeout` já presente na URL, como já se faz com o `connect_timeout`)
- Modify: `deploy/production/deploy.yml`:
  - acessório: `cmd: postgres -c max_connections=1000`;
  - o comentário velho dá lugar a uma referência ao runbook e à rake;
  - mantenha a nota BLOQUEADO sobre a chave `cmd:` do Kamal;
  - `env.clear` ganha `CITY_ROLE_CONNECTION_LIMIT: "150"`.
- Test: `spec/config/connection_budget_spec.rb` (novo), `spec/models/city_connection_spec.rb` (ou o arquivo de spec que já cobre `database_config`; procure com `grep -rln "database_config" spec`)

**Interfaces:**
- Consumes: `Ops::ConnectionBudget.from_files` (Task 1).

- [ ] **Step 1: testes que falham.**

```ruby
# spec/config/connection_budget_spec.rb
require "rails_helper"

# Rodada especial do 19 (api#74): a conta de conexões para as cidades do go-live
# cabe no Postgres configurado. Acrescentar worker, thread ou host sem rever o
# max_connections reprova aqui.
RSpec.describe "Postgres connection budget for the go-live" do
  let(:go_live_cities) { 5 }   # decisão do usuário, 2026-10-13
  let(:headroom) { 0.8 }       # 20% de folga

  let(:deploy) { YAML.safe_load(Rails.root.join("deploy/production/deploy.yml").read, aliases: true) }
  let(:budget) { Ops::ConnectionBudget.from_files }

  def max_connections
    cmd = deploy.dig("accessories", "postgres", "cmd").to_s
    value = cmd[/max_connections=(\d+)/, 1]
    raise "deploy.yml: acessório postgres sem `-c max_connections=N` no cmd" unless value

    value.to_i
  end

  it "fits the go-live cities in 80% of max_connections" do
    total = budget.total(go_live_cities)
    expect(total).to be <= (max_connections * headroom).floor,
                     "#{total} conexões para #{go_live_cities} cidades; max_connections=#{max_connections}"
  end

  it "fits one city's peak in 80% of the per-role connection limit" do
    limit = deploy.dig("env", "clear", "CITY_ROLE_CONNECTION_LIMIT").to_i
    expect(budget.role_per_city).to be <= (limit * headroom).floor,
                     "pico de uma cidade #{budget.role_per_city}; CITY_ROLE_CONNECTION_LIMIT=#{limit}"
  end

  it "sizes the worker pool for the largest worker plus Solid Queue's two" do
    threads = deploy.dig("servers", "worker", "env", "clear", "RAILS_MAX_THREADS").to_i
    expect(threads).to be >= budget.city_workers.max + Ops::ConnectionBudget::SOLID_QUEUE_EXTRA
  end
end
```

  - **`database_config` (spec do `CityConnection`):** com uma cidade de `database_url` sem `idle_timeout`, `CityConnection.database_config(city).configuration_hash[:idle_timeout].to_i == 60`. Com `?idle_timeout=5` na URL, continua 5.
  - **Mensagem do `deploy.yml` sem `max_connections`:** um teste que chama o `max_connections` com um `deploy` sem a linha (por `let` sobrescrito num `context`) e espera `raise_error(/sem `-c max_connections=N`/)`.

  Run: `docker exec -w /rails/.claude/r4 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec spec/config/connection_budget_spec.rb <spec do CityConnection>`
  Expected: FAIL. `777 > 400` (500 × 0,8), `117 > 0` (a variável ausente vira 0) e `idle_timeout` nil.

- [ ] **Step 2: implementar.**
  - **`CityConnection.database_config`:**

    ```ruby
    url += "&idle_timeout=#{IDLE_TIMEOUT_SECONDS}" unless url.match?(/[?&]idle_timeout=/)
    ```

    `IDLE_TIMEOUT_SECONDS = 60`, ao lado do `CONNECT_TIMEOUT_SECONDS`, com comentário: o web guarda 2 pools por cidade atendida; ociosos por 60 s, devolvem as conexões.
  - **`deploy.yml`:** `max_connections=1000` e `CITY_ROLE_CONNECTION_LIMIT: "150"` em `env.clear`. Confira que a `staging_environment_parity_spec` e a `deploy_hosts_spec` continuam verdes. Se a paridade exigir o mesmo no `deploy/staging/deploy.yml`, aplique lá também (Ruling).

  Run: `docker exec -w /rails/.claude/r4 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec spec/config spec/architecture spec/models/city_connection_spec.rb`
  Expected: verde.

- [ ] **Step 3: commit** (`fix: size max_connections and the per-city role limit for five cities and free idle city pools`).

---

### Task 3: Runbook e documentação

**Files:**
- Create: `docs/operacao/conexoes-postgres.md` (repo docs)
- Modify: `docs/operacao/rollout-assinatura.md` (linhas 38–40 e 140), `docs/operacao/rollout-documentos-clinicos.md` (linha 179), `docs/operacao/README.md` (índice, se listar os runbooks)
- Modify (api): `README.md` (seção "Dimensionamento" e "Orçamento de conexões por cidade", linhas ~413–440)

- [ ] **Step 1: runbook** `docs/operacao/conexoes-postgres.md`, em português, no padrão dos rollouts:
  - para que serve;
  - a fórmula da seção "Modelo da conta", com a origem de cada número;
  - tabela de 1 a 20 cidades, gerada pela rake e colada (`bin/rails 'ops:connection_budget[N]'` para N = 1, 2, 3, 4, 5, 10, 15, 20);
  - configuração atual (`max_connections=1000`, `CITY_ROLE_CONNECTION_LIMIT=150`) e o custo em memória (cerca de 10 MB por conexão no pior caso, o que dá cerca de 10 GB com 1000; dimensione o host do Postgres junto);
  - quando revisar: mais cidades, mais workers ou threads, mais hosts web;
  - PgBouncer em modo transação como próximo passo, acima de cerca de 8 cidades ou quando a memória não fechar. Registre como item a avaliar, sem receita: *advisory locks* e *prepared statements* exigem configuração própria no modo transação.
  - **gate de go-live:** prova com carga em staging (5 cidades ativas, worker e web no tamanho de produção) medindo `select count(*) from pg_stat_activity` no pico, abaixo de 80% do `max_connections`.
- [ ] **Step 2: rollouts.** Os itens bloqueantes de `max_connections` passam a citar o runbook e dizer o que resta: a prova com carga em staging e a confirmação da chave `cmd:` do Kamal.
- [ ] **Step 3: README do api.** A seção "Orçamento de conexões" troca as contas velhas (~80 por cidade, ~280) por:
  - um resumo de 3 linhas;
  - o comando da rake;
  - o link para `docs/operacao/conexoes-postgres.md`.

  O "~6 processos por cidade" vira "7 processos (4 workers, dispatcher, scheduler e supervisor)".
- [ ] **Step 4: commits.** No api, `docs: point the connection budget section to the runbook`. No docs, `docs: add the Postgres connection budget runbook`.

---

### Task 4: Revisão, aprovação e publicação

- [ ] **Suíte completa do api.** `docker compose stop worker`; depois `docker exec -w /rails/.claude/r4 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec`; depois `docker compose start worker`. Expected: verde. Anote a contagem.
- [ ] **Revisão final** por revisor independente, no modelo mais capaz, com o Review Focus. Critical e Important são corrigidos com TDD e entram no ledger.
- [ ] **Aprovação do usuário:** a tabela da conta, a configuração nova e o runbook.
- [ ] **Publicação.**
  - avise as sessões do api (módulo 20, curadora);
  - confira `origin/main..main`;
  - `merge --ff-only` e `push` do api e do docs;
  - confira a CI do api;
  - remova o worktree;
  - mova o api#74 para "Resolvido" no board #2, com o comentário da conta e do gate de staging que fica aberto. O card não fecha, porque a prova com carga é gate de go-live. Mantenha-o aberto com o Status "Resolvido" e o comentário dizendo o que resta, ou acerte com o usuário na aprovação.
