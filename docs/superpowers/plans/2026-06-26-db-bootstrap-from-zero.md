# DB Bootstrap-from-zero Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `start.sh --reset` (and a new `rails db:bootstrap`) rebuild the dev database from zero **with RLS intact**, by loading a versioned `db/structure.sql` as a privileged superuser.

**Architecture:** Rails 8's `db:migrate` loads `schema.rb` on a fresh DB, and Ruby `schema.rb` cannot represent the raw-SQL RLS policies / `rota_admin` ownership (proven empirically). Fix: a committed `db/structure.sql` (`pg_dump --schema-only`, which faithfully captures RLS) loaded by a new `db:bootstrap` rake task via its own privileged `PG.connect`/`psql` as `rota_saude` (superuser). `schema.rb` and the test flow stay unchanged (scoped to dev).

**Tech Stack:** Ruby on Rails 8.1, PostgreSQL 15, Docker Compose, bash.

## Global Constraints

- Privileged step connects as **`rota_saude`** (the dev-cluster superuser); password from `POSTGRES_PASSWORD`. App roles `rota_app`/`rota_admin` from `ROTA_APP_PASSWORD`/`ROTA_ADMIN_PASSWORD` (defaults `rota_app`/`rota_admin`).
- All commands run **inside the `api` container** (reaches host Postgres via `DATABASE_HOST`). Run via `docker compose run --rm --no-deps api <cmd>` or `docker exec api-dev <cmd>`.
- **Do not** change `schema_format` (stays `:ruby`), `database.yml`, `db/migrate/*`, `bin/docker-entrypoint`, or the test flow.
- `db/structure.sql` is a generated artifact: regenerate with `rails db:bootstrap:dump` whenever a migration changes structure/RLS, and commit it.
- Commit messages in **English** (project convention).
- Target DB selection: `ENV["BOOTSTRAP_DATABASE"]` or, if unset, the current `RAILS_ENV` primary database.

---

### Task 1: From-zero verification harness (the failing test)

**Files:**
- Create: `apps/api/bin/verify-bootstrap`

**Interfaces:**
- Consumes: `rails db:bootstrap` (Task 2) honoring `BOOTSTRAP_DATABASE`; `db/structure.sql` (Task 2).
- Produces: `bin/verify-bootstrap [scratch_db]` — exits 0 iff a from-zero bootstrap yields the RLS invariants. Used by Task 2 as its pass gate.

- [ ] **Step 1: Write the verification script**

Create `apps/api/bin/verify-bootstrap`:

```bash
#!/usr/bin/env bash
# Valida o bootstrap-from-zero (db:bootstrap) num banco-rascunho descartável,
# SEM tocar o banco de dev. Rode dentro do container api.
# Ver docs/superpowers/specs/2026-06-26-db-bootstrap-from-zero-design.md
set -euo pipefail

SCRATCH="${1:-rs_bootstrap_check}"
H="${DATABASE_HOST:-127.0.0.1}"
export PGPASSWORD="${POSTGRES_PASSWORD:-postgres}"
su() { psql -h "$H" -U rota_saude "$@"; }

cleanup() { su -d postgres -qc "DROP DATABASE IF EXISTS $SCRATCH;" >/dev/null 2>&1 || true; }
trap cleanup EXIT

echo "[verify] (re)criando $SCRATCH vazio"
su -d postgres -qc "DROP DATABASE IF EXISTS $SCRATCH;" >/dev/null
su -d postgres -qc "CREATE DATABASE $SCRATCH OWNER rota_saude;" >/dev/null

echo "[verify] db:bootstrap -> $SCRATCH"
BOOTSTRAP_DATABASE="$SCRATCH" ./bin/rails db:bootstrap

echo "[verify] asserindo invariantes"
fail=0
q() { su -d "$SCRATCH" -tAc "$1" | tr -d '[:space:]'; }
check() { if [ "$2" = "$3" ]; then echo "  ok: $1 ($3)"; else echo "  FAIL: $1 — esperado '$2', obtido '$3'"; fail=1; fi; }

check "policies tenant_isolation" 9          "$(q "SELECT count(*) FROM pg_policy WHERE polname='tenant_isolation';")"
check "owner conversations"       rota_admin "$(q "SELECT tableowner FROM pg_tables WHERE tablename='conversations';")"
check "owner consents"            rota_admin "$(q "SELECT tableowner FROM pg_tables WHERE tablename='consents';")"
check "owner municipalities"      rota_app   "$(q "SELECT tableowner FROM pg_tables WHERE tablename='municipalities';")"
check "rota_app CREATE on public" f          "$(q "SELECT has_schema_privilege('rota_app','public','CREATE');")"
check "rota_app USAGE on public"  t          "$(q "SELECT has_schema_privilege('rota_app','public','USAGE');")"
check "ext citext"                1          "$(q "SELECT count(*) FROM pg_extension WHERE extname='citext';")"
check "ext pgcrypto"              1          "$(q "SELECT count(*) FROM pg_extension WHERE extname='pgcrypto';")"

if [ "$fail" = 0 ]; then echo "[verify] OK — bootstrap-from-zero íntegro"; else echo "[verify] FALHOU"; exit 1; fi
```

- [ ] **Step 2: Make it executable**

Run: `chmod +x apps/api/bin/verify-bootstrap`

- [ ] **Step 3: Run it to verify it FAILS (no db:bootstrap yet)**

Run: `docker exec api-dev ./bin/verify-bootstrap`
Expected: FAIL — `rails aborted! Don't know how to build task 'db:bootstrap'` (scratch DB is created then dropped by the trap).

- [ ] **Step 4: Commit**

```bash
cd apps/api
git add bin/verify-bootstrap
git commit -m "test: add from-zero bootstrap verification harness"
```

---

### Task 2: `db:bootstrap` + `db:bootstrap:dump` rake tasks and structure.sql

**Files:**
- Create: `apps/api/lib/tasks/bootstrap.rake`
- Create: `apps/api/db/structure.sql` (generated by `db:bootstrap:dump`)

**Interfaces:**
- Consumes: `POSTGRES_PASSWORD`, `ROTA_APP_PASSWORD`, `ROTA_ADMIN_PASSWORD`, `DATABASE_HOST`, `DATABASE_PORT`, optional `BOOTSTRAP_DATABASE`/`BOOTSTRAP_SUPERUSER`; the dev database (with RLS) for the dump.
- Produces: `db:bootstrap` (creates roles+memberships, loads `db/structure.sql` as superuser into the target DB) and `db:bootstrap:dump` (regenerates `db/structure.sql`). Consumed by Task 1's harness and Task 3's `start.sh`.

- [ ] **Step 1: Write `lib/tasks/bootstrap.rake`**

Create `apps/api/lib/tasks/bootstrap.rake`:

```ruby
# Bootstrap-from-zero do banco sob ADR-0019. db:migrate num banco novo carrega o
# schema.rb (Ruby), que NÃO representa RLS/ownership (SQL cru). Aqui carregamos um
# db/structure.sql (pg_dump --schema-only) como superuser rota_saude, reproduzindo
# RLS + ownership + least-priv fielmente.
# Ver docs/superpowers/specs/2026-06-26-db-bootstrap-from-zero-design.md
require "open3"

namespace :db do
  STRUCTURE_SQL = File.expand_path("../../db/structure.sql", __dir__)

  def bootstrap_conn_params
    cfg = ActiveRecord::Base.connection_db_config.configuration_hash
    {
      db:      ENV.fetch("BOOTSTRAP_DATABASE", cfg[:database]),
      host:    ENV.fetch("DATABASE_HOST", cfg[:host] || "127.0.0.1").to_s,
      port:    ENV.fetch("DATABASE_PORT", cfg[:port] || 5432).to_s,
      su_user: ENV.fetch("BOOTSTRAP_SUPERUSER", "rota_saude"),
      su_pwd:  ENV.fetch("POSTGRES_PASSWORD") { abort "[db:bootstrap] POSTGRES_PASSWORD ausente — o passo privilegiado exige o superuser." }
    }
  end

  desc "Provisiona roles e carrega db/structure.sql (RLS) como superuser. Idempotente."
  task bootstrap: :environment do
    p = bootstrap_conn_params
    abort "[db:bootstrap] #{STRUCTURE_SQL} não existe — rode `rails db:bootstrap:dump`." unless File.exist?(STRUCTURE_SQL)

    app_pwd   = ENV.fetch("ROTA_APP_PASSWORD", "rota_app")
    admin_pwd = ENV.fetch("ROTA_ADMIN_PASSWORD", "rota_admin")
    env       = { "PGPASSWORD" => p[:su_pwd] }
    base      = ["psql", "-h", p[:host], "-p", p[:port], "-U", p[:su_user], "-v", "ON_ERROR_STOP=1"]

    puts "[db:bootstrap] (1/2) papéis + memberships em #{p[:db]} como #{p[:su_user]}"
    roles_sql = <<~SQL
      DO $$ BEGIN
        IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='rota_app')   THEN CREATE ROLE rota_app   LOGIN PASSWORD '#{app_pwd}'; END IF;
        IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='rota_admin') THEN CREATE ROLE rota_admin LOGIN PASSWORD '#{admin_pwd}' BYPASSRLS; END IF;
      END $$;
      GRANT rota_saude TO rota_admin;
    SQL
    out, st = Open3.capture2e(env, *base, "-d", p[:db], "-c", roles_sql)
    abort "[db:bootstrap] falha nos papéis:\n#{out}" unless st.success?

    puts "[db:bootstrap] (2/2) carregando structure.sql em #{p[:db]}"
    out, st = Open3.capture2e(env, *base, "-d", p[:db], "-f", STRUCTURE_SQL)
    abort "[db:bootstrap] falha no load do structure.sql:\n#{out}" unless st.success?

    puts "[db:bootstrap] OK — #{p[:db]} provisionado do zero (com RLS)."
  end

  namespace :bootstrap do
    desc "Regenera db/structure.sql via pg_dump --schema-only do banco corrente."
    task dump: :environment do
      p   = bootstrap_conn_params
      env = { "PGPASSWORD" => p[:su_pwd] }
      out, st = Open3.capture2(env, "pg_dump", "-h", p[:host], "-p", p[:port],
                               "-U", p[:su_user], "--schema-only", p[:db])
      abort "[db:bootstrap:dump] pg_dump falhou:\n#{out}" unless st.success?
      File.write(STRUCTURE_SQL, out)
      puts "[db:bootstrap:dump] OK — #{STRUCTURE_SQL} (#{out.lines.size} linhas)."
    end
  end
end
```

- [ ] **Step 2: Generate `db/structure.sql` from the dev DB**

The dev DB (`rota_saude_development`) already has RLS, so the dump captures it.
Run: `docker exec api-dev ./bin/rails db:bootstrap:dump`
Expected: `[db:bootstrap:dump] OK — .../db/structure.sql (~2700 linhas).`

- [ ] **Step 3: Sanity-check the generated dump captured RLS**

Run:
```bash
docker exec api-dev bash -lc "grep -c 'CREATE POLICY tenant_isolation' db/structure.sql; grep -c 'OWNER TO rota_admin' db/structure.sql"
```
Expected: `9` and `9`.

- [ ] **Step 4: Run the verification harness — expect PASS**

Run: `docker exec api-dev ./bin/verify-bootstrap`
Expected: all `ok:` lines, ending `[verify] OK — bootstrap-from-zero íntegro` (exit 0). In particular `policies tenant_isolation (9)`, `owner conversations (rota_admin)`, `rota_app CREATE on public (f)`.

- [ ] **Step 5: Commit**

```bash
cd apps/api
git add lib/tasks/bootstrap.rake db/structure.sql
git commit -m "feat(api): db:bootstrap loads structure.sql with RLS from zero"
```

---

### Task 3: Wire `start.sh` and document

**Files:**
- Modify: `start.sh` (the "migrations + cache schema" section)
- Modify: `apps/api/README.md` (add a bootstrap section)

**Interfaces:**
- Consumes: `rails db:bootstrap` (Task 2).
- Produces: `start.sh --reset` rebuilds the dev DB with RLS; README documents the workflow.

- [ ] **Step 1: Replace `db:prepare` with `db:bootstrap` in `start.sh`**

In `start.sh`, find the block:

```bash
# --- migrations + cache schema ------------------------------------------------
say "rodando db:prepare contra o host"
docker compose run --rm --no-deps api ./bin/rails db:prepare
```

Replace with:

```bash
# --- bootstrap do schema (structure.sql, com RLS) -----------------------------
# db:prepare/db:migrate num banco novo carregam schema.rb (sem RLS). db:bootstrap
# cria roles e carrega db/structure.sql como superuser, reproduzindo RLS+ownership.
# Ver docs/superpowers/specs/2026-06-26-db-bootstrap-from-zero-design.md
say "provisionando schema com RLS (db:bootstrap)"
docker compose run --rm --no-deps api ./bin/rails db:bootstrap
```

(The `solid_cache` fallback block immediately after stays unchanged — `structure.sql` covers `public`, not the solid_cache/queue schemas.)

- [ ] **Step 2: Smoke-test a real from-zero dev reset**

Run: `./start.sh --reset`
Expected: completes; `[db:bootstrap] OK — rota_saude_development provisionado do zero (com RLS).`; `/up` returns 200.

- [ ] **Step 3: Verify the rebuilt dev DB has RLS**

Run:
```bash
docker exec api-dev bash -c 'PGPASSWORD=${ROTA_ADMIN_PASSWORD:-rota_admin} psql -h $DATABASE_HOST -U rota_admin -d rota_saude_development -tAc "SELECT count(*) FROM pg_policy WHERE polname='"'"'tenant_isolation'"'"';"'
```
Expected: `9`.

- [ ] **Step 4: Document in `apps/api/README.md`**

Add a section:

```markdown
## Bootstrap do banco (do zero)

`schema.rb` (Ruby) não representa RLS/ownership do ADR-0019 — eles vivem como SQL
cru. Por isso o rebuild from-zero usa `db/structure.sql` (gerado por `pg_dump`):

- `rails db:bootstrap` — cria roles e carrega `db/structure.sql` como superuser
  (`rota_saude`/`POSTGRES_PASSWORD`) na DB-alvo (`BOOTSTRAP_DATABASE` ou a do
  `RAILS_ENV`). É o que `start.sh --reset` usa.
- `rails db:bootstrap:dump` — regenera `db/structure.sql`. **Rode e commite sempre
  que uma migration mexer em estrutura/RLS** (depois de aplicar a migration no dev).
- `bin/verify-bootstrap` — valida o bootstrap from-zero num banco-rascunho.

Migrations incrementais no dev seguem via `db:migrate` (entrypoint), normalmente.
```

- [ ] **Step 5: Commit**

```bash
cd <raiz-do-monorepo>
git -C apps/api add README.md 2>/dev/null || true
# start.sh vive na raiz do monorepo (não é repo git aqui); apps/api é repo git.
cd apps/api && git add README.md && git commit -m "docs(api): document db:bootstrap from-zero workflow"
```

> **Note:** `start.sh` lives at the monorepo root, which is **not** a git repo locally (only `apps/*` are). Commit the `apps/api` changes; the `start.sh` edit is saved to disk and syncs via whatever process manages the root tree.

---

## Self-Review

- **Spec coverage:** structure.sql artifact (§3.1 → Task 2), `db:bootstrap` privileged load (§3.2 → Task 2), `start.sh` wiring (§3.3 → Task 3), `db:bootstrap:dump` (§3.4 → Task 2), validation harness (§4 → Task 1), README (§5 → Task 3). All covered.
- **Placeholder scan:** none — every step has concrete code/commands and expected output.
- **Type consistency:** `BOOTSTRAP_DATABASE`/`BOOTSTRAP_SUPERUSER`/`POSTGRES_PASSWORD` names and the `STRUCTURE_SQL` path are consistent across the rake task, harness, and start.sh. Invariant checks (9 policies, owners, rota_app CREATE=f/USAGE=t, extensions) match the spec's validated table.
