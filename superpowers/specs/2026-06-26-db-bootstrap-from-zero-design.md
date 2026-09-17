# Design — Bootstrap-from-zero do Postgres de dev (`db:bootstrap` via `structure.sql`)

**Data:** 2026-06-26
**Escopo:** `apps/api` (Rails 8.1) + `start.sh` (orquestração dev)
**Objetivo:** tornar a reconstrução do banco de dev **do zero** confiável e
**com RLS** — hoje `start.sh --reset` é uma armadilha e qualquer rebuild perde a
isolação multi-tenant.

> **Histórico:** a v1 deste spec (Abordagem A: pré-conceder privilégios + `db:migrate`)
> foi **invalidada por teste empírico** — ver §1.2. Este documento descreve a
> abordagem validada (**A\*-escopado**: carregar um `structure.sql` privilegiado).

---

## 1. Problema

### 1.1 Privilégios (ADR-0019)

`database.yml` conecta as migrations como papéis sem privilégio de
provisionamento: `primary`→`rota_app` (least-priv, FORCE RLS),
`admin`→`rota_admin` (BYPASSRLS, **não-superuser**). Várias migrations precisam
de privilégios que esses papéis não têm: `CREATE EXTENSION` (enable_extensions),
`CREATE ROLE`/`GRANT` no DB (create_database_roles), `ALTER TABLE … OWNER TO
rota_admin` (enable_rls_on_data_plane). O `rota_saude` (dono, **superuser** neste
cluster de dev) provisionou o banco **uma vez**; os papéis são cluster-global e
sobrevivem a um `DROP DATABASE`.

### 1.2 Causa-raiz real (provado empiricamente, 2026-06-26)

Rodando `RAILS_ENV=test ./bin/rails db:migrate --trace` num banco **vazio**:

```
** Execute db:migrate
-- create_table("consents", {... /rails/db/schema.rb:55 ...}, force: :cascade)
```

**Zero banners `== Migrating`; os `create_table` vêm de `db/schema.rb` com
`force: :cascade`.** Ou seja: **o `db:migrate` do Rails 8, num banco novo, faz
`schema:load` do `schema.rb` — não replaya as migrations.** Como `schema.rb` é
Ruby, **não representa** RLS nem ownership `rota_admin` (que vivem como SQL cru na
migration `20260620000020`, via `PG.connect` própria). Resultado medido no
`rota_saude_test` recriado do zero: **35 tabelas, 30 migrations carimbadas, mas
`0` políticas `tenant_isolation` e data-plane com owner=`rota_app`.**

**Conclusão:** nem `db:prepare` nem `db:migrate` reproduzem RLS do zero. O banco
de dev atual só tem RLS por acidente histórico (migrations rodaram de verdade
antes do `schema.rb` existir). Qualquer rebuild com o tooling atual perde o RLS.

### 1.3 Solução validada: `structure.sql`

`pg_dump --schema-only` **captura RLS fielmente**. Provado contra o banco de dev
(que tem RLS): 9 `CREATE POLICY tenant_isolation`, 9 `OWNER TO rota_admin`, 9
`ENABLE`+`FORCE ROW LEVEL SECURITY`, 2 `CREATE EXTENSION`. Carregando esse dump
num banco vazio **como superuser `rota_saude`** (provado, 0 erros):

| Invariante | Resultado |
|---|---|
| políticas `tenant_isolation` | **9** |
| owner data-plane / app | `rota_admin` / `rota_app` |
| `rota_app` no schema `public` | `CREATE=false`, `USAGE=true` (least-priv **já vem no ACL do dump** — sem REVOKE separado) |
| extensions | `citext`, `pgcrypto` |

Papéis **não** saem no `pg_dump --schema-only` (são cluster-global); o bootstrap
os cria à parte.

---

## 2. Objetivo e não-objetivos

**Objetivo:** `start.sh --reset` e um novo `rails db:bootstrap` reconstroem o
banco de dev do zero **com RLS correto**, carregando um `db/structure.sql`
versionado como superuser, terminando no estado-alvo (data-plane=`rota_admin`,
`rota_app` least-priv, 9 políticas).

**Não-objetivos (YAGNI / decisões fixadas):**
- **Não** trocar `schema_format` para `:sql` global. `schema.rb` segue como schema
  do fluxo normal do Rails (dev/test **inalterados**, suite verde). O
  `structure.sql` é usado **só** pelo bootstrap from-zero.
- **Não** ligar RLS no banco de _test_. A política usa
  `current_setting('app.municipality_id')` **sem `missing_ok`** → sob `FORCE RLS`,
  todo acesso ao data-plane como `rota_app` sem o GUC setado dá erro; ligar RLS no
  test exigiria migrar a suite inteira. **Fica como follow-up separado**
  (o bug "RLS não-exercitado pelos testes" é real, mas é outro projeto).
- **Não** automatizar provisionamento de produção (a infra cuida; a mecânica é a
  mesma: criar roles + `psql -f structure.sql` como superuser).

---

## 3. Design

### 3.1 Artefato versionado: `apps/api/db/structure.sql`

Snapshot `pg_dump --schema-only` de um banco corretamente migrado (com RLS).
Contém tabelas, extensions, índices, políticas RLS, ownership e o ACL de schema
(least-priv do `rota_app`). **Não** contém papéis (cluster-global).

> **Risco aceito (trade-off do escopo dev):** convivem dois artefatos de schema —
> `schema.rb` (auto, Rails) e `structure.sql` (bootstrap). Mitigação: a task de
> dump (§3.4) regenera `structure.sql`; **deve ser rodada e commitada sempre que
> uma migration mexer em estrutura/RLS.** Documentado no README do `api`.

### 3.2 `rails db:bootstrap` (provisionamento privilegiado)

Nova rake task em `apps/api/lib/tasks/bootstrap.rake`. Roda dentro do container
(alcança o Postgres do host via `DATABASE_HOST`). Abre uma **`PG.connect` própria
como `rota_saude`** (superuser; senha de `POSTGRES_PASSWORD`) — mesmo padrão que
`enable_rls_on_data_plane` usa para `rota_admin`, porque a conexão AR padrão
(`rota_app`) não tem privilégio. Executa, idempotente:

1. **Papéis + memberships:**
   - `CREATE ROLE rota_admin … BYPASSRLS` e `rota_app …`, guardados por
     `IF NOT EXISTS` (senhas de `ROTA_ADMIN_PASSWORD`/`ROTA_APP_PASSWORD`).
   - `GRANT rota_saude TO rota_admin` (reproduz o estado observado).
2. **Load do schema:** `psql -v ON_ERROR_STOP=1 -f db/structure.sql` (ou
   `priv.exec(File.read(...))`) na DB-alvo, como `rota_saude`. Como superuser, os
   `ALTER … OWNER TO rota_admin`, `CREATE EXTENSION` e `CREATE POLICY` aplicam;
   o ACL do dump deixa `rota_app` com só `USAGE` (least-priv automático).

A DB-alvo é `ENV["BOOTSTRAP_DATABASE"]` (default: o banco do `RAILS_ENV` corrente,
lido de `ActiveRecord::Base.connection_db_config.database`) — o override permite a
validação (§4) apontar para um banco-rascunho. Pré-condição: a DB-alvo **já existe
e está vazia** (o `start.sh`/infra faz o `createdb -O rota_saude`). A task **falha
alto** se `POSTGRES_PASSWORD` faltar ou se `db/structure.sql` não existir.

### 3.3 Wiring no `start.sh`

- No caminho de bootstrap (first-create e `--reset`), após `createdb -O rota_saude`,
  invocar `docker compose run --rm --no-deps api ./bin/rails db:bootstrap`
  **em vez de** `db:prepare`. Remove o uso de `db:prepare`/`db:migrate` para
  banco novo (ambos fazem `schema:load` do `schema.rb` **sem RLS** — §1.2).
- **Manter** o fallback de carga do schema do `solid_cache` (dedup multi-db) como
  rede de segurança, após o bootstrap — `structure.sql` cobre o `public`, mas
  `solid_cache`/`solid_queue` têm seus próprios schemas.
- Runs normais (DB já existe, sem `--reset`): inalterados — o `docker-entrypoint`
  segue rodando `db:migrate` no boot (incremental sobre DB provisionado).

### 3.4 `rails db:bootstrap:dump` (regenerar o snapshot)

Task que roda `pg_dump --schema-only` do banco do `RAILS_ENV` corrente (como
`rota_saude`) para `db/structure.sql`. Workflow do dev: adicionar migration →
`db:migrate` no dev (incremental, **roda de verdade**, aplica RLS) →
`db:bootstrap:dump` → commitar `structure.sql`.

---

## 4. Validação (test-driven)

Script verificável (`apps/api/bin/verify-bootstrap`, bash) que **não toca** a DB
de dev — usa um banco-rascunho descartável (`rs_bootstrap_check`):

1. `createdb -O rota_saude rs_bootstrap_check` (vazio).
2. `BOOTSTRAP_DATABASE=rs_bootstrap_check rails db:bootstrap` (load privilegiado).
3. Assere as invariantes (mesmas medidas no probe §1.3):
   - **9** políticas `tenant_isolation`;
   - data-plane (`conversations`, `consents`, `domain_events`, … 9 tabelas) com
     owner=`rota_admin`; tabelas de app com owner=`rota_app`;
   - `has_schema_privilege('rota_app','public','CREATE')` = **false**,
     `USAGE` = **true**;
   - extensions `citext`, `pgcrypto` presentes;
   - contagem de tabelas == a do dump.
4. `dropdb rs_bootstrap_check`.

Esse é o teste que falha primeiro (TDD): roda antes de existir `db:bootstrap` →
falha; guia a implementação até passar.

---

## 5. Arquivos afetados

- **Novo:** `apps/api/db/structure.sql` — snapshot com RLS (gerado via §3.4).
- **Novo:** `apps/api/lib/tasks/bootstrap.rake` — tasks `db:bootstrap` e
  `db:bootstrap:dump`.
- **Novo:** `apps/api/bin/verify-bootstrap` — validação from-zero (§4).
- **Editado:** `start.sh` — troca `db:prepare` por `db:bootstrap` no caminho de
  bootstrap; mantém fallback do `solid_cache`.
- **Editado:** `apps/api/README.md` — documenta `db:bootstrap`,
  `db:bootstrap:dump` e a regra de regenerar `structure.sql`.
- **Intocado:** migrations, `database.yml`, `docker-entrypoint`, `schema.rb`,
  `schema_format` (segue `:ruby`), fluxo de test.

---

## 6. Decisões fixadas

1. **A\*-escopado** — carregar `structure.sql` privilegiado (vs. `db:migrate`, que
   carrega `schema.rb` sem RLS; vs. `schema_format=:sql` global, que ligaria RLS
   no test e quebraria a suite).
2. **`schema.rb` permanece** o schema do Rails; `structure.sql` é exclusivo do
   bootstrap. RLS-nos-testes = follow-up separado.
3. **Rake task `db:bootstrap`** com `PG.connect` privilegiada como `rota_saude`.
4. Least-priv do `rota_app` **vem do ACL do dump** — sem REVOKE explícito.
