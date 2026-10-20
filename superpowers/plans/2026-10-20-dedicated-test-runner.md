# Execução de testes dedicada — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rodar toda a suíte do api (e o vitest dos frontends) por um comando único, `bin/test`, num Postgres 16 descartável e só de teste, uma execução por vez, com saída resumida.

**Architecture:**
- Dois serviços novos no compose, no profile `test`: `postgres-test` (servidor em tmpfs) e `api-test` (one-shot).
- O container de execução tem nome fixo, `rota-test-run`, e esse nome é a trava: o Docker recusa um segundo container com o mesmo nome.
- O reset do banco e o bootstrap acontecem dentro desse container, então só quem tem a trava mexe no banco.
- Um guard em `spec/support` aborta a suíte se o servidor não for o de teste.

**Tech Stack:** Docker Compose, Postgres 16, Rails/RSpec, Bash, Ruby puro (resumo), Vitest.

**Spec:** `docs/superpowers/specs/2026-10-20-dedicated-test-runner-design.md`

## Global Constraints

- **Versão do Postgres de teste:** `postgres:16`, a de produção e da CI.
- **Nome do servidor de teste:** `cluster_name=rota_saude_test`. O guard só aceita esse valor.
- **Parâmetros do servidor:** `max_connections=300`, `fsync=off`, `synchronous_commit=off`, `full_page_writes=off`.
- **Profile:** `postgres-test` e `api-test` ficam no profile `test` e nunca sobem com o `docker compose up` comum.
- **Isolamento:** `api-test` não usa a âncora `*host-postgres` e define `DATABASE_HOST=postgres-test`.
- **Bootstrap:** os cinco passos, na ordem da CI:
  1. role `rota_app`;
  2. `platform:bootstrap`;
  3. `db:migrate`;
  4. `platform:triggers`;
  5. `city:test_databases`.
- **Trava:** o container de execução se chama `rota-test-run`, tanto no api quanto nos frontends. A espera na fila tem tempo limite de 30 min e nova tentativa a cada 5 s.
- **Log completo:** `apps/api/tmp/test-runs/<timestamp>.log` (api) ou `apps/<app>/tmp/test-runs/` (frontends).
- **Commits:** Conventional Commits em inglês, com tipo por extenso e o trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Repositórios:** `apps/api` é repo git; use `/opt/homebrew/bin/git`. A raiz do monorepo (`docker-compose.yml`, `start.sh`, `bin/test`, `CLAUDE.md`) **não** é versionada: nada a commitar ali.
- **Dono do app:** tocar em `apps/api` exige avisar a sessão "API" antes, via `SendMessage`.

## Review Focus

1. **Sessão na fila sobrescrevendo o banco de quem está rodando.** O reset não pode acontecer fora do container `rota-test-run`, e `postgres-test` nunca sobe com `--force-recreate` no `bin/test` (Task 4, verificação de duas execuções simultâneas).
2. **Execução interrompida (Ctrl-C, sessão encerrada) deixando trava presa.** Ao sair, o `bin/test` remove o container `rota-test-run` que ele criou (`trap`). Uma nova execução não pode ficar 30 min na fila por causa de um container órfão `Exited` (Task 4).
3. **`api-test` herdando `DATABASE_HOST` do `.env` ou do shell.** A variável é fixada no serviço, e o guard pega a sobra (Task 2, teste com host apontado para `host.docker.internal`).
4. **Worktree sem `master.key`.** O `bin/test -w` falha cedo, com mensagem que diz para copiar a chave, em vez de o Rails explodir em `MissingKeyError` no meio do bootstrap (Task 4).
5. **Falha antes do rspec (bootstrap quebrado, JSON ausente).** O resumo diz "suíte não rodou" e mostra o fim do log, nunca "0 falhas" (Task 3).

---

### Task 1: Bootstrap compartilhado e `postgres-test`

**Files:**
- Create: `apps/api/bin/test-db-bootstrap` (bash, executável)
- Modify: `apps/api/.github/workflows/ci.yml`:
  - os passos "Role rota_app…" e "Bancos e roles de plataforma e de cidade" viram um só, `run: bin/test-db-bootstrap`;
  - o serviço `postgres` ganha `cluster_name`.
- Modify: `docker-compose.yml` (raiz): serviços `postgres-test` e `api-test`.

**Interfaces:**
- Produces:
  - Serviço one-shot `api-test` (profile `test`), apontado para o `postgres-test`.
  - `bin/test-db-bootstrap [--reset]`. Lê `DATABASE_HOST`, `DATABASE_PORT`, `POSTGRES_PASSWORD`, `ROTA_APP_PASSWORD` e `BOOTSTRAP_SUPERUSER` (padrão `rota_saude`). Sai com código ≠ 0 no primeiro passo que falhar e diz qual foi.
  - Serviço `postgres-test`, alcançável em `postgres-test:5432` só pela rede do compose.

- [ ] **Step 1: Adicionar `postgres-test` ao compose.**
  - Imagem `postgres:16`, `profiles: [test]`, `container_name: postgres-test`.
  - `POSTGRES_USER=rota_saude`, `POSTGRES_PASSWORD=postgres`.
  - `tmpfs: /var/lib/postgresql/data`.
  - `command: postgres -c max_connections=300 -c fsync=off -c synchronous_commit=off -c full_page_writes=off -c cluster_name=rota_saude_test`.
  - Healthcheck `pg_isready -U rota_saude`, sem `ports`.

- [ ] **Step 2: Verificar o servidor.**

  Run: `docker compose --profile test up -d --wait postgres-test && docker compose exec postgres-test psql -U rota_saude -tAc "select version(); show cluster_name; show max_connections;"`

  Expected: `PostgreSQL 16.x…`, `rota_saude_test`, `300`.

- [ ] **Step 3: Escrever `bin/test-db-bootstrap`.**
  - Os cinco passos da CI, na mesma ordem.
  - O `rota_app` é criado com o heredoc + `\gexec` + `format(%L)`.
  - O comentário que explica o heredoc e o `\gexec` sai do `ci.yml` e vai para o script, resumido.
  - `--reset`, antes dos passos:
    1. `DROP DATABASE … WITH (FORCE)` para todo `datname` fora de `postgres` e `template0`/`template1`;
    2. `DROP OWNED BY` e depois `DROP ROLE` para todo role que não seja `rota_saude` e não comece com `pg_`.
  - Cada passo imprime `[test-db-bootstrap] <passo>` e nada mais. A saída dos comandos vai para stdout, que o `bin/test` manda para o log.

- [ ] **Step 4: Adicionar `api-test` ao compose.**
  - `profiles: [test]`, imagem `rota-saude/api:dev`.
  - Volumes iguais aos do `api`.
  - `environment`: `<<: *api-env` sobrescrito com `RAILS_ENV: test`, `DATABASE_HOST: postgres-test`, `DATABASE_PORT: "5432"` e `POSTGRES_PASSWORD: postgres`.
  - `depends_on: postgres-test: condition: service_healthy`, sem `<<: *host-postgres` e sem `ports`.

- [ ] **Step 4b: Verificar o bootstrap contra o `postgres-test`, rodando duas vezes para provar o `--reset`.**

  Run, duas vezes seguidas:

  `docker compose --profile test run --rm api-test bin/test-db-bootstrap --reset`

  Expected: nas duas vezes, os cinco `[test-db-bootstrap]` e saída 0. Depois, `psql -l` no `postgres-test` lista `rota_saude_platform_test`, `rota_saude_test_city_a`, `rota_saude_test_city_b` e `rota_saude_no_city_selected`.

- [ ] **Step 5: Trocar os dois passos da CI por `run: bin/test-db-bootstrap` e acrescentar `TEST_SERVER_GUARD: "off"` ao `env:` do job.** O service container do GitHub não aceita `command`, então a CI não tem como definir `cluster_name`. Ela já é descartável e isolada; o guard protege a máquina local. Registre o motivo num comentário ao lado da variável.

- [ ] **Step 6: Commit em `apps/api`.**

```bash
/opt/homebrew/bin/git -C apps/api add bin/test-db-bootstrap .github/workflows/ci.yml
/opt/homebrew/bin/git -C apps/api commit -m "ci: extract test database bootstrap into bin/test-db-bootstrap"
```

### Task 2: Guard de servidor de teste

**Files:**
- Create: `apps/api/spec/support/test_server_guard.rb`
- Create: `apps/api/spec/support/test_server_guard_spec.rb` (no padrão de `city_test_databases_spec.rb`)
- Modify: `apps/api/spec/rails_helper.rb`: `require_relative "support/test_server_guard"` **antes** de `city_test_databases` e um `config.before(:suite) { TestServerGuard.check! }`.

**Interfaces:**
- Produces:
  - `TestServerGuard::EXPECTED = "rota_saude_test"`.
  - `TestServerGuard.verify!(cluster_name, env = ENV)`: pura. Levanta `TestServerGuard::WrongServer` se o nome difere e `env["TEST_SERVER_GUARD"] != "off"`.
  - `TestServerGuard.check!`: lê `PlatformRecord.connection.select_value("SHOW cluster_name")` e chama `verify!`.

- [ ] **Step 1: Escrever os testes que falham.**

```ruby
RSpec.describe TestServerGuard do
  it "aceita o servidor de teste" do
    expect { described_class.verify!("rota_saude_test", {}) }.not_to raise_error
  end

  it "recusa outro servidor, nomeando o recebido" do
    expect { described_class.verify!("", {}) }
      .to raise_error(TestServerGuard::WrongServer, /rota_saude_test.*""/m)
  end

  it "pode ser desligado só explicitamente (CI)" do
    expect { described_class.verify!("", { "TEST_SERVER_GUARD" => "off" }) }.not_to raise_error
  end
end
```

- [ ] **Step 2: Rodar e ver falhar** (uninitialized constant).

  Run: `docker compose --profile test run --rm api-test sh -c 'bin/test-db-bootstrap --reset && bundle exec rspec spec/support/test_server_guard_spec.rb'`

- [ ] **Step 3: Implementar.** A mensagem de `WrongServer` diz:
  - o esperado;
  - o recebido;
  - "a suíte só roda no postgres-test (bin/test)".

- [ ] **Step 4: Rodar e ver passar.**

- [ ] **Step 5: Verificação de integração. O guard tem de barrar o servidor de dev.**

  Run: `docker compose run --rm --no-deps -e RAILS_ENV=test api bundle exec rspec spec/adr_pointers_spec.rb`

  Esse comando usa o Postgres do host, com os bancos de teste antigos.

  Expected: aborta com `TestServerGuard::WrongServer` e `0 examples`. Nenhum exemplo roda.

- [ ] **Step 6: Commit** — `test: refuse to run the suite outside the test Postgres server`.

### Task 3: Resumo da execução

**Files:**
- Create: `apps/api/script/test_summary.rb` (Ruby puro, sem Rails)
- Create: `apps/api/spec/script/test_summary_spec.rb`

**Interfaces:**
- Produces:
  - `TestSummary.render(json_path:, log_path:, exit_code:) -> String`.
  - CLI `ruby script/test_summary.rb <json> <log> <exit_code>`, que imprime o resultado de `render`.

- [ ] **Step 1: Escrever os testes que falham.** Use JSON de rspec mínimo como fixture inline.
  - **Verde:** a saída contém `1662 exemplos, 0 falhas, 2 pendentes` e `em 312.4s`, além de `log: <log_path>`.
  - **Com falhas:** cada falha vira uma linha `spec/x_spec.rb:12 — expected 1, got 2`, só com a primeira linha da mensagem. Com mais de 30 falhas, mostra 30 e `… +N (ver log)`.
  - **Suíte não rodou** (JSON ausente ou vazio): a saída contém `suíte não rodou (exit <code>)` e as últimas 20 linhas do log. Nunca contém `0 falhas`.
  - **Erros fora de exemplo** (`errors_outside_of_examples_count > 0`): a saída mostra `erros fora de exemplo: N` e as últimas 20 linhas do log.

- [ ] **Step 2: Rodar e ver falhar.**
- [ ] **Step 3: Implementar com `JSON.parse`.** Use os campos `summary` e `examples[].{file_path,line_number,exception.message}` do formatter JSON do RSpec.
- [ ] **Step 4: Rodar e ver passar.**
- [ ] **Step 5: Commit** — `feat: add compact test run summary script`.

### Task 4: `bin/test`

**Files:**
- Create: `bin/test` (raiz, bash, executável)

**Interfaces:**
- Consumes:
  - `bin/test-db-bootstrap --reset` (Task 1);
  - `TestServerGuard` (Task 2);
  - `script/test_summary.rb <json> <log> <exit>` (Task 3).
- Produces: `bin/test api [-w <wt>] [--full | <specs...>] [--keep-db]` e `bin/test <admin|dashboard|wpda|maintenance> [-w <wt>] [args do vitest]`. O código de saída é o do rspec ou do vitest.

- [ ] **Step 2: Escrever `bin/test`.**
  - **Args:**
    - `-w <wt>`, para o api, vira `-w /rails/.claude/<wt>` e exige `apps/api/.claude/<wt>/config/master.key`. Sem a chave, sai 2 com `copie apps/api/config/master.key para o worktree`.
    - `--full` roda sem args de spec.
    - Sem `--full` e sem specs, sai 2 com o uso.
  - **Fila:** enquanto `docker ps -aq -f name=^rota-test-run$` não estiver vazio:
    - se o container estiver `exited`, remova-o: é órfão;
    - senão, imprima `fila: aguardando execução em curso` uma vez e espere 5 s, até 30 min (depois disso, sai 3).
  - **Execução:** `trap 'docker rm -f rota-test-run >/dev/null 2>&1' EXIT INT TERM`, depois `docker compose --profile test up -d --wait postgres-test`, depois `docker compose --profile test run --rm --name rota-test-run [-w …] api-test sh -c '<reset/bootstrap> && bundle exec rspec … -f progress -f json -o tmp/test-runs/<ts>.json'`, com stdout e stderr indo para o log.
    - `--keep-db` troca o reset e o bootstrap por um teste de existência de `rota_saude_platform_test` (cai no bootstrap se não existir).
    - Se o `run` falhar com `already in use` (corrida entre a checagem e o run), volte à fila.
  - **Resumo:** `ruby script/test_summary.rb` dentro de um `docker compose run --rm --no-deps api-test`, ou com o Ruby do host se houver. Escolha do builder.
  - **Frontends:** `docker compose run --rm --no-deps --name rota-test-run [-v <wt>:/app] <app> npx vitest run <args>`, passando pela mesma fila e com o log em `apps/<app>/tmp/test-runs/`. O resumo do vitest são as linhas `Test Files` e `Tests` e as linhas `FAIL`, filtradas por `grep`.

- [ ] **Step 3: Verificar a execução completa com o worker ligado.**

  Run: `docker compose up -d worker && time bin/test api --full`

  Expected: o resumo de poucas linhas, sem `too many clients`, com o mesmo total de exemplos de uma suíte recente. Anote o tempo total e o do bootstrap (o timestamp de cada `[test-db-bootstrap]` está no log).

- [ ] **Step 4: Verificar a fila.**

  Run: dois terminais, `bin/test api --full` nos dois, com 2 s de diferença.

  Expected: o segundo imprime `fila: aguardando…`. O primeiro termina verde e intacto, e só então o segundo roda.

- [ ] **Step 5: Verificar a trava órfã.**

  Run: `bin/test api spec/adr_pointers_spec.rb`, Ctrl-C durante o bootstrap e, em seguida, `bin/test api spec/adr_pointers_spec.rb`.

  Expected: a segunda execução não espera e passa.

- [ ] **Step 6: Verificar a troca de branch.**

  Run: `bin/test api -w <wt-com-migração-de-cidade> --full`, depois `bin/test api --full` (main).

  Expected: os dois verdes, sem passo manual entre eles.

- [ ] **Step 7: Verificar um frontend e o worktree sem chave.**

  Run: `bin/test dashboard` e `bin/test api -w inexistente spec/x_spec.rb`.

  Expected: o resumo do vitest e saída 0. No segundo, a mensagem do `master.key` e saída 2.

- [ ] **Step 8: Sem commit.** A raiz não é versionada. Registre os tempos medidos no Step 3 para o relatório final.

### Task 5: Sair do Postgres do host, protocolo da sessão "Testes" e limpeza

**Files:**
- Modify: `start.sh` (raiz). Remova as três linhas `docker compose run … -e RAILS_ENV=test api …` (perto da linha 110) e o `say` que as anuncia. No lugar, ponha o comentário `# testes: bin/test (postgres-test, ver docs/superpowers/specs/2026-10-20-dedicated-test-runner-design.md)`.
- Modify: `CLAUDE.md` (raiz). Nova seção `## Testes` com:
  - `bin/test` e seus modos;
  - a política de escopo da spec §4.7, copiada literalmente;
  - o protocolo da sessão "Testes": pedido no formato `app · worktree/branch · escopo · motivo` e resposta só com o resumo;
  - a regra de que nenhuma sessão roda `rspec` fora do `bin/test`.
- Memória: atualize `suite-com-worker-parado.md`, `bancos-de-teste-entre-branches.md` e `apagar-bancos-com-sufixo-de-modulo.md`, marcando-os como superados pelo `bin/test`, e crie uma memória `bin-test-runner.md` com o índice no `MEMORY.md`.

- [ ] **Step 1: Editar o `start.sh` e conferir.**

  Run: `bash -n start.sh && grep -n "RAILS_ENV=test" start.sh`

  Expected: sintaxe ok e nenhuma linha encontrada.

- [ ] **Step 2: Escrever a seção do `CLAUDE.md` e as memórias.**

- [ ] **Step 3: Listar os bancos de teste no Postgres do host e pedir confirmação ao usuário. Não apagar sem um "sim" explícito.**

  Run: `psql -h 127.0.0.1 -U rota_saude -d postgres -tAc "select datname from pg_database where datname like 'rota_saude_test%' or datname = 'rota_saude_platform_test'"` e `select datname, count(*) from pg_stat_activity where datname like 'rota_saude_%test%' group by 1`.

  Mostre a lista e as conexões ativas ao usuário.

- [ ] **Step 4: Depois do "sim", fazer o `DROP DATABASE … WITH (FORCE)` de cada um e conferir.** A lista do Step 3 tem de voltar vazia, e `rota_saude_platform_development` e os bancos de cidade de dev precisam continuar presentes.

- [ ] **Step 5: Avisar as outras sessões do grupo** (admin, dashboard, wpda, api, maintenance) via `SendMessage` sobre o `bin/test` e a sessão "Testes". Peça ao usuário para criar a sessão "Testes", em Sonnet.

- [ ] **Step 6: Push do `apps/api`**, depois de checar `origin/main..main` conforme a memória de sessões paralelas, e confirmar que a CI fica verde com o `bin/test-db-bootstrap`.
