# Execução de testes dedicada — design

**Data:** 2026-10-20
**Status:** aprovado (2026-10-20)
**Afeta:**
- `docker-compose.yml`: serviços `postgres-test` e `api-test` (profile `test`).
- `apps/api`:
  - script de bootstrap do banco de teste;
  - `bin/test` (executor único);
  - trava de servidor em `spec/support`;
  - `.github/workflows/ci.yml` passa a chamar o bootstrap compartilhado.
- `start.sh`: deixa de criar bancos de teste no Postgres do host.
- `CLAUDE.md` da raiz: protocolo da sessão "Testes" e política de escopo.

## 1. Intenção

Hoje a suíte do api roda no **mesmo servidor Postgres do host** que guarda os bancos de desenvolvimento. Daí vêm falhas que não são culpa do código sob teste e que custam tempo e tokens para diagnosticar:

- bancos de teste à frente da main depois de trocar de branch;
- "too many clients" com o worker ligado;
- bancos `_modNN` que precisam de `DROP` manual;
- medições de tempo infladas pela carga do dev.

Além disso, local roda Postgres 15, enquanto CI e produção rodam 16.

O objetivo é que rodar teste seja **um comando**, num **servidor descartável e só de teste**, **um de cada vez**, com saída **curta** para quem pediu. A suíte completa fica concentrada numa sessão dedicada.

## 2. Objetivos, não objetivos e antiobjetivos

**Objetivos**
- A suíte nunca alcança o servidor de desenvolvimento, nem por engano de configuração.
- Banco de teste recriado a partir do schema da branch em teste, sem passo manual.
- Uma execução de teste por vez em toda a máquina (api e frontends).
- Postgres 16 local, igual ao da CI e ao de produção.
- Saída resumida: contagem, duração e falhas (arquivo:linha + mensagem). O log completo fica em arquivo.
- Uma sessão "Testes" é a dona da suíte completa.

**Não objetivos**
- Testcontainers.
- Paralelismo de suíte (`parallel_tests`, um servidor por worktree).
- Mudar a CI além de reaproveitar o script de bootstrap.
- Remover o código de `ROTA_TEST_DB_SUFFIX`: fica inerte.

**Antiobjetivos** (é falha mesmo que funcione)
- Qualquer caminho em que `RAILS_ENV=test` conecte no Postgres do host.
- Duas suítes rodando ao mesmo tempo contra o `postgres-test`.
- A sessão que pediu o teste recebendo a saída bruta do rspec.
- A CI e o local divergindo no bootstrap: duas receitas em vez de uma.

## 3. Restrições

- O Postgres de dev continua sendo o do host (`host.docker.internal`). Este design não o move.
- O bootstrap tem de reproduzir exatamente o da CI:
  1. role `rota_app` (o `platform:bootstrap` não o cria);
  2. `platform:bootstrap`;
  3. `db:migrate`;
  4. `platform:triggers`, porque o schema dump não carrega trigger;
  5. `city:test_databases`.
- `config/database.yml` avalia o ERB inteiro. `ROTA_PLATFORM_PASSWORD`, `WHATSAPP_VERIFY_TOKEN` e `WHATSAPP_APP_SECRET` precisam estar no ambiente do `api-test`.
- Worktrees do api ficam em `apps/api/.claude/<wt>`, com `master.key` copiada, e rodam com diretório de trabalho `/rails/.claude/<wt>`.
- Pela regra do grupo de sessões, mudanças em `apps/api` são combinadas antes com a sessão "API".

## 4. Desenho

### 4.1 `postgres-test`
Serviço no `docker-compose.yml`, **profile `test`**, para não subir com o `docker compose up` comum.

- Imagem `postgres:16`.
- `POSTGRES_USER=rota_saude` e `POSTGRES_PASSWORD=postgres`: o superusuário de bootstrap, como na CI.
- Dados em `tmpfs` (`/var/lib/postgresql/data`): recriar o container zera o servidor.
- Parâmetros:
  - `max_connections=300`;
  - `fsync=off`;
  - `synchronous_commit=off`;
  - `full_page_writes=off`;
  - `cluster_name=rota_saude_test`.
- Sem porta publicada no host: só a rede do compose o alcança.
- Healthcheck `pg_isready -U rota_saude`.

### 4.2 `api-test`
Serviço one-shot, profile `test`.

- Imagem `rota-saude/api:dev`.
- Volumes iguais aos do `api`.
- `RAILS_ENV=test`, `DATABASE_HOST=postgres-test`, `DATABASE_PORT=5432`, mais as variáveis exigidas pela §3.
- Não herda a âncora `*host-postgres` nem `DATABASE_HOST` do dev.
- `depends_on: postgres-test (service_healthy)`.

### 4.3 Bootstrap compartilhado
Script `apps/api/bin/test-db-bootstrap`, com os cinco passos da §3 na ordem da CI. A criação do `rota_app` usa `psql` com heredoc, `\gexec` e `format(%L)`, preservando o motivo documentado no `ci.yml`.

O `ci.yml` troca os dois passos atuais ("Role rota_app" e "Bancos e roles…") por uma chamada a este script. A explicação longa sobre o `psql` sai do `ci.yml` e vai para o script.

### 4.4 Trava de servidor
Em `spec/support`, um guard carregado antes de qualquer exemplo:
- consulta `SHOW cluster_name` pela conexão de plataforma;
- se o valor não for `rota_saude_test`, aborta a suíte com mensagem clara e sem rodar nenhum exemplo.

A CI precisa do mesmo nome: o serviço `postgres:16` do `ci.yml` ganha `cluster_name=rota_saude_test`. Se o GitHub não aceitar o parâmetro por `options`, fica a cargo do builder ajustar via `ALTER SYSTEM` + reload no bootstrap.

### 4.5 `bin/test` (executor único)
Script na raiz do monorepo, executado no host. Interface:

```
bin/test api [-w <worktree>] [--full | <specs...>] [--keep-db]
bin/test <admin|dashboard|wpda|maintenance> [-w <worktree>] [args do vitest]
```

Fluxo do `api`:
1. **Fila.** Executa via `docker compose run --rm --name rota-test-run`. Se o nome estiver em uso, espera e tenta de novo a cada poucos segundos, com tempo limite configurável (padrão de 30 min) e mensagem "fila: aguardando execução em curso". O `--rm` remove o container ao final, então não sobra trava presa. A mesma trava serve para os frontends: o executor do vitest também roda com `--name rota-test-run`.
2. **Banco.** Recria o `postgres-test` (`up -d --force-recreate`, espera ficar saudável) e roda `bin/test-db-bootstrap`. Com `--keep-db`, pula esta etapa se o servidor já estiver de pé (atalho para rodadas de arquivo isolado).
3. **Execução.** `bundle exec rspec` com `-f failures`, saída de progresso para o log e `--format json` para um arquivo.
4. **Resumo.** Imprime só: exemplos, falhas, pendentes, duração, lista `arquivo:linha — mensagem` (primeira linha) e o caminho do log completo em `apps/api/tmp/test-runs/<timestamp>.log`.
5. **Saída.** O código de saída é o do rspec.

### 4.6 Sessão "Testes"
- Única sessão que roda `bin/test api --full`.
- Recebe pedidos por `SendMessage` no formato `app · worktree/branch · escopo · motivo` e responde **só** o resumo do §4.5.
- Investiga falha só se for pedido: o diagnóstico é da sessão dona do app.
- Modelo recomendado: Sonnet.

### 4.7 Política de escopo
Vai para o `CLAUDE.md` da raiz.

**Escopo fechado** (`bin/test api <arquivos>`, na própria sessão do app):
- ciclo de TDD;
- feature pronta antes do commit, com os specs dela e os vizinhos diretos;
- bugfix;
- correções de review;
- vitest do app tocado.

**Suíte completa** (pedida à sessão Testes):
- antes de merge na main;
- antes de push, se a branch não passou pelo merge;
- mudanças transversais: migração de cidade ou plataforma, `spec/support`, initializers, `Current`, `CityConnection`, `Platform.audit`/R18, jobs base, Gemfile;
- antes de mover um card para Verified;
- no fechamento de módulo;
- depois de sync com a main, se trouxe migração ou conflito.

**Regra de decisão:** se a mudança pode afetar um spec que você não abriu, é suíte completa. Várias features na mesma branch levam **uma** suíte completa no fim.

### 4.8 Limpeza do host
- `start.sh` deixa de rodar `platform:bootstrap`/`db:migrate:platform`/`city:test_databases` com `RAILS_ENV=test`.
- A remoção dos bancos de teste existentes no Postgres do host (`rota_saude_platform_test`, `rota_saude_test_*`, sufixos `_modNN`, `rota_saude_test_scratch_*`) é um passo único, **com confirmação explícita do usuário**, e conferindo conexões ativas antes do `DROP`.

## 5. Estratégia de teste

| Verificação | Resultado esperado |
|---|---|
| `api-test` com `DATABASE_HOST=host.docker.internal` | Guard aborta antes do 1º exemplo |
| Dois `bin/test api` simultâneos | O segundo entra na fila e roda depois |
| Branch com migração de cidade, depois main | Parity spec verde nas duas, sem passo manual |
| Suíte completa com `worker-dev` ligado | Sem "too many clients" |
| `SELECT version()` no `postgres-test` | 16.x |
| CI com `bin/test-db-bootstrap` | Verde |
| Resumo de rodada com falha forçada | Só contagem + `arquivo:linha — mensagem` + caminho do log |
| Tempo da suíte completa, antes × depois, e tempo do bootstrap | Registrado no PR/commit |

## 6. Decisões do builder (calls do Claude)

- Nomes: `postgres-test`, `api-test`, `rota-test-run`, `bin/test`, `bin/test-db-bootstrap`.
- Tempo limite da fila: 30 min. Intervalo de nova tentativa: 5 s.
- Recriar o banco a cada rodada é o padrão. Se o bootstrap passar de ~60 s, a otimização seguinte é recriar só quando o hash de `apps/api/db/` mudar. Fica fora deste escopo até medir.
- O formato exato do resumo e a ferramenta de parse do JSON do rspec (Ruby no container ou `jq` no host) ficam com o builder.
