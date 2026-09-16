# Migração `apps/api` → `rotasaude/api` — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Popular o repo público `rotasaude/api` com o backend Rails do monorepo local, sem segredos, clonável e executável contra o host Postgres.

**Architecture:** Clonar o repo `rotasaude/api` (que já tem o commit `chore: bootstrap inicial`), importar o conteúdo de `apps/api/` aplicando exclusões, adicionar `docker-compose.yml` standalone (host Postgres) + `.env.example` + README expandido, endurecer `.gitignore`, rodar um gate de scan de segredos e dar push direto na `main`.

**Tech Stack:** Rails 8, Docker Compose, Postgres (host), `gh`/`git`, bash.

## Global Constraints

- Repo `rotasaude/api` é **PÚBLICO** — nenhum segredo pode entrar (`config/master.key`, `.env`, `deploy/*/secrets` reais, chaves). Scan pré-push é bloqueante.
- Origem: `<raiz-do-monorepo>/apps/api` (monorepo local, **não** é repo git).
- Commit limpo único sobre o `chore: bootstrap inicial` remoto (sem preservar histórico — não há histórico local).
- Push **direto na `main`** (admin bypass; `enforce_admins=false`).
- Postgres é do **host** (`host.docker.internal`); bootstrap-from-zero é out-of-scope.
- Diretório de trabalho: `/tmp/rotasaude-api-migrate`.
- Não alterar código de aplicação do `api`. Não tocar branch protection/visibilidade.

---

### Task 1: Clonar o repo e importar o conteúdo do `api` (com exclusões)

**Files:**
- Create (working tree): `/tmp/rotasaude-api-migrate/api/` (clone de `rotasaude/api`)
- Source: `<raiz-do-monorepo>/apps/api/`

**Interfaces:**
- Produces: working tree em `/tmp/rotasaude-api-migrate/api` com o app Rails importado, `.git` do bootstrap preservado, sem cruft nem `master.key`.

- [ ] **Step 1: Clonar o repo e limpar working dir anterior**

```bash
rm -rf /tmp/rotasaude-api-migrate
mkdir -p /tmp/rotasaude-api-migrate
git clone git@github.com:rotasaude/api.git /tmp/rotasaude-api-migrate/api
```

Expected: clone com 1 commit (`chore: bootstrap inicial`), arquivos `README.md` + `.gitignore`.

- [ ] **Step 2: Importar o conteúdo do `apps/api` excluindo cruft e segredos**

`rsync` preservando o `.git` do clone e excluindo o que não vai pro repo:

```bash
SRC=<raiz-do-monorepo>/apps/api
DST=/tmp/rotasaude-api-migrate/api
rsync -a \
  --exclude='.git/' \
  --exclude='RECONCILE_admin_console.md' \
  --exclude='log/' \
  --exclude='tmp/' \
  --exclude='.DS_Store' \
  --exclude='storage/*.sqlite3' \
  --exclude='deploy/*/secrets' \
  --exclude='config/master.key' \
  "$SRC"/ "$DST"/
```

Note: `config/credentials.yml.enc` (cifrado) É copiado de propósito. `config/master.key` é excluído.

- [ ] **Step 3: Verificar que master.key e cruft NÃO foram importados**

```bash
cd /tmp/rotasaude-api-migrate/api
test ! -f config/master.key && echo "OK: sem master.key"
test ! -f RECONCILE_admin_console.md && echo "OK: sem RECONCILE"
test -f config/credentials.yml.enc && echo "OK: credentials.yml.enc presente (cifrado)"
test -f Dockerfile && test -f Gemfile && echo "OK: app Rails presente"
```

Expected: as 4 linhas `OK:`.

---

### Task 2: Endurecer o `.gitignore` para repo público

**Files:**
- Modify: `/tmp/rotasaude-api-migrate/api/.gitignore`

**Interfaces:**
- Consumes: working tree da Task 1.
- Produces: `.gitignore` cobrindo `master.key`, `.env*`, `.DS_Store`, `deploy/*/secrets`.

- [ ] **Step 1: Inspecionar o `.gitignore` atual (veio do `apps/api`)**

```bash
cd /tmp/rotasaude-api-migrate/api
grep -nE "master.key|credentials|\.env|DS_Store|deploy" .gitignore || echo "(faltam entradas)"
```

Expected: já tem `/config/master.key`, `/config/credentials/*.key`, `/.env*`, `!/.env.example` (vindos do app). Provavelmente faltam `.DS_Store` e `/deploy/*/secrets`.

- [ ] **Step 2: Acrescentar as entradas que faltam**

```bash
cd /tmp/rotasaude-api-migrate/api
cat >> .gitignore <<'EOF'

# --- Endurecimento para repo público (migração 2026-06-26) ---
.DS_Store
# Arquivos de secrets do Kamal — versionar só os *.example
/deploy/*/secrets
EOF
```

- [ ] **Step 3: Verificar cobertura via git check-ignore**

```bash
cd /tmp/rotasaude-api-migrate/api
for p in config/master.key .env .DS_Store deploy/development/secrets deploy/production/secrets; do
  git check-ignore -q "$p" && echo "IGNORADO: $p" || echo "!! NÃO ignorado: $p"
done
```

Expected: as 5 linhas `IGNORADO:`. Qualquer `!! NÃO ignorado` → parar e corrigir o `.gitignore`.

---

### Task 3: Adicionar os templates `secrets.example`

**Files:**
- Create: `/tmp/rotasaude-api-migrate/api/deploy/development/secrets.example`
- Create: `/tmp/rotasaude-api-migrate/api/deploy/production/secrets.example`

**Interfaces:**
- Consumes: `.gitignore` da Task 2 (que ignora `deploy/*/secrets`).
- Produces: estrutura dos secrets visível no repo sem versionar o arquivo real.

- [ ] **Step 1: Gerar os `.example` a partir dos secrets atuais (já sem plaintext)**

```bash
SRC=<raiz-do-monorepo>/apps/api
DST=/tmp/rotasaude-api-migrate/api
cp "$SRC"/deploy/development/secrets "$DST"/deploy/development/secrets.example
cp "$SRC"/deploy/production/secrets  "$DST"/deploy/production/secrets.example
```

- [ ] **Step 2: Confirmar que os `.example` não têm valores reais**

```bash
cd /tmp/rotasaude-api-migrate/api
grep -nEi "=[A-Za-z0-9+/]{20,}|BEGIN .*PRIVATE KEY|password\s*=\s*[^$#]" deploy/*/secrets.example \
  && echo "!! POSSÍVEL SEGREDO nos .example — revisar" || echo "OK: .example sem valores reais"
```

Expected: `OK: .example sem valores reais`. Se aparecer match, inspecionar a linha; é template (`$VAR`/`op read`) ou vazamento? Vazamento → parar.

- [ ] **Step 3: Confirmar que o `secrets` real ficou de fora**

```bash
cd /tmp/rotasaude-api-migrate/api
test ! -e deploy/development/secrets && test ! -e deploy/production/secrets \
  && echo "OK: secrets reais ausentes (não copiados)" || echo "!! secrets real presente — remover"
```

Expected: `OK: secrets reais ausentes`.

---

### Task 4: `docker-compose.yml` standalone + `.env.example`

**Files:**
- Create: `/tmp/rotasaude-api-migrate/api/docker-compose.yml`
- Create: `/tmp/rotasaude-api-migrate/api/.env.example`

**Interfaces:**
- Consumes: `Dockerfile` (stage `development`), `bin/docker-entrypoint`, `bin/jobs`, `bin/rails` do app.
- Produces: stack `api`+`worker` contra host Postgres, parametrizada por `.env`.

- [ ] **Step 1: Escrever o `docker-compose.yml`**

```yaml
# Stack de desenvolvimento standalone do rota-saude/api (ADR-0001: web + worker
# da mesma imagem). Postgres é EXTERNO (host) — ver README. Em produção quem
# orquestra é Kamal 2 (deploy/).
name: rota-saude-api

x-api-env: &api-env
  RAILS_ENV: ${RAILS_ENV:-development}
  DATABASE_HOST: ${DATABASE_HOST:-host.docker.internal}
  DATABASE_PORT: ${DATABASE_PORT:-5432}
  ROTA_APP_PASSWORD: ${ROTA_APP_PASSWORD:-rota_app}
  ROTA_ADMIN_PASSWORD: ${ROTA_ADMIN_PASSWORD:-rota_admin}
  WHATSAPP_VERIFY_TOKEN: ${WHATSAPP_VERIFY_TOKEN:-devtoken}
  WHATSAPP_APP_SECRET:   ${WHATSAPP_APP_SECRET:-devsecret}
  WHATSAPP_ACCESS_TOKEN: ${WHATSAPP_ACCESS_TOKEN:-devaccess}
  WHATSAPP_PHONE_NUMBER_ID: ${WHATSAPP_PHONE_NUMBER_ID:-devphone}
  PUBLIC_HOST: ${PUBLIC_HOST:-localhost}
  PUBLIC_PORT: ${PUBLIC_PORT:-3030}
  PUBLIC_PROTOCOL: ${PUBLIC_PROTOCOL:-http}
  ALLOWED_ORIGINS: ${ALLOWED_ORIGINS:-http://localhost:5174,http://localhost:5175,http://localhost:5176}

# Em Linux host.docker.internal não resolve nativo; em Mac/Windows o Docker Desktop trata.
x-host-postgres: &host-postgres
  extra_hosts:
    - "host.docker.internal:host-gateway"

services:
  api:
    build:
      context: .
      target: development
    image: rota-saude/api:dev
    command: ./bin/rails server -b 0.0.0.0
    environment:
      <<: *api-env
    volumes:
      - .:/rails
      - rails-tmp:/rails/tmp
    ports:
      - "${API_PORT:-3030}:3000"
    <<: *host-postgres
    restart: unless-stopped

  worker:
    image: rota-saude/api:dev
    command: ./bin/jobs start
    environment:
      <<: *api-env
      RAILS_MAX_THREADS: ${WORKER_MAX_THREADS:-20}
    volumes:
      - .:/rails
      - rails-tmp:/rails/tmp
    depends_on:
      api:
        condition: service_started
    <<: *host-postgres
    restart: unless-stopped

volumes:
  rails-tmp:
```

Note: **sem** `container_name` (evita colidir com `api-dev`/`worker-dev` do stack da raiz).

- [ ] **Step 2: Escrever o `.env.example`**

```bash
# Copie para .env e ajuste. Em prod NÃO use este arquivo — secrets vêm do
# Kamal via 1Password (ver deploy/SECRETS.md e deploy/production/secrets).

RAILS_ENV=development

# Porta exposta no host (deve estar livre).
API_PORT=3030

# Postgres é EXTERNO (host do Docker). Pré-requisito no host (ver README):
#   createuser -d rota_saude && createdb -O rota_saude rota_saude_development
DATABASE_HOST=host.docker.internal
DATABASE_PORT=5432
ROTA_APP_PASSWORD=rota_app
ROTA_ADMIN_PASSWORD=rota_admin

# WhatsApp Cloud API (sandbox em dev).
WHATSAPP_VERIFY_TOKEN=devtoken
WHATSAPP_APP_SECRET=devsecret
WHATSAPP_ACCESS_TOKEN=devaccess
WHATSAPP_PHONE_NUMBER_ID=devphone

# URL pública (ReportSnapshot#url — ADR-0007).
PUBLIC_HOST=localhost
PUBLIC_PORT=3030
PUBLIC_PROTOCOL=http

# CORS — origins dos frontends Vite.
ALLOWED_ORIGINS=http://localhost:5174,http://localhost:5175,http://localhost:5176

# Worker — pool cobre as threads do queue.yml.
WORKER_MAX_THREADS=20
```

Escrever esse conteúdo em `/tmp/rotasaude-api-migrate/api/.env.example`.

- [ ] **Step 3: Validar que o compose parseia**

```bash
cd /tmp/rotasaude-api-migrate/api
docker compose config >/dev/null && echo "OK: compose válido"
```

Expected: `OK: compose válido` (sem erro de YAML/interpolação).

---

### Task 5: README expandido

**Files:**
- Modify: `/tmp/rotasaude-api-migrate/api/README.md`

**Interfaces:**
- Consumes: compose/.env.example das tasks anteriores.
- Produces: instruções de clone-e-roda, pré-requisito host Postgres, nota master.key e limitação from-zero.

- [ ] **Step 1: Sobrescrever o README (mantendo o cabeçalho do bootstrap)**

Escrever em `/tmp/rotasaude-api-migrate/api/README.md`:

```markdown
# api

Backend Rails 8 do Rota Saúde. Papéis web + worker da mesma imagem (ADR 0001).

Decisões arquiteturais em rotasaude/docs.

## Desenvolvimento

Postgres roda no **host** (não em container). Pré-requisitos:

- Docker + Docker Compose
- Postgres no host com owner e databases provisionados:

      createuser -d rota_saude
      createdb -O rota_saude rota_saude_development
      createdb -O rota_saude rota_saude_test

- `config/master.key` colocado manualmente em `config/` (não está no repo —
  obtenha com o time). Sem ele o Rails não decifra `credentials.yml.enc`.

Subir:

      cp .env.example .env
      docker compose up

Verificar:

      curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3030/up   # 200

Serviços: `api` (web, porta 3030→3000) e `worker` (Solid Queue). Logs:
`docker compose logs -f api`.

### Limitação conhecida (bootstrap-from-zero)

As migrations rodam **incrementalmente sobre um banco já provisionado**. Os
roles `rota_app`/`rota_admin` (ADR-0019) são criados por migration que conecta
como `rota_app`, então um Postgres **vazio** não sobe sozinho. Provisionar do
zero (ou Postgres containerizado efêmero) é follow-up — ver
`rotasaude/docs` e a issue de bootstrap.

## Produção

Deploy via Kamal 2 (`deploy/`). Secrets via 1Password (`deploy/SECRETS.md`).
```

- [ ] **Step 2: Verificar que o README não tem segredo nem path local**

```bash
cd /tmp/rotasaude-api-migrate/api
grep -nE "/Users/|master.key.*=|[A-Za-z0-9+/]{32,}" README.md \
  && echo "!! revisar README" || echo "OK: README limpo"
```

Expected: `OK: README limpo`.

---

### Task 6: Gate de scan de segredos (bloqueante)

**Files:**
- Working tree: `/tmp/rotasaude-api-migrate/api` (índice git)

**Interfaces:**
- Consumes: tudo das tasks anteriores.
- Produces: confirmação de que nada sensível está staged.

- [ ] **Step 1: Stagear tudo**

```bash
cd /tmp/rotasaude-api-migrate/api
git add -A
```

- [ ] **Step 2: Listar o que será commitado e checar arquivos proibidos**

```bash
cd /tmp/rotasaude-api-migrate/api
git diff --cached --name-only | sort | tee /tmp/rota-api-staged.txt | head -50
echo "--- proibidos? ---"
grep -E "(^|/)(master\.key|\.env$|\.DS_Store)$|deploy/(development|production)/secrets$|\.sqlite3$|RECONCILE_admin_console\.md$" /tmp/rota-api-staged.txt \
  && echo "!! ARQUIVO PROIBIDO STAGED — abortar" || echo "OK: nenhum arquivo proibido staged"
```

Expected: `OK: nenhum arquivo proibido staged`. Caso contrário, parar, desfazer (`git reset`), corrigir `.gitignore`/exclusões.

- [ ] **Step 3: Scan de conteúdo por padrões de segredo no índice**

```bash
cd /tmp/rotasaude-api-migrate/api
git grep -nEI --cached \
  "BEGIN [A-Z ]*PRIVATE KEY|AKIA[0-9A-Z]{16}|xox[baprs]-|ghp_[A-Za-z0-9]{30,}|-----BEGIN|(secret|password|token|api[_-]?key)[\"' ]*[:=][\"' ]*[A-Za-z0-9+/]{16,}" \
  -- . ':(exclude)*.enc' ':(exclude)Gemfile.lock' \
  && echo "!! POSSÍVEL SEGREDO — revisar cada match antes de pushar" \
  || echo "OK: scan de conteúdo limpo"
```

Expected: `OK: scan de conteúdo limpo`. Qualquer match → inspecionar manualmente; só prosseguir se for falso-positivo claro (ex.: nome de var em `.env.example` com valor dev `devtoken`).

---

### Task 7: Commit limpo + push direto na `main`

**Files:**
- Working tree: `/tmp/rotasaude-api-migrate/api`
- Remote: `rotasaude/api` (`main`)

**Interfaces:**
- Consumes: índice validado da Task 6.
- Produces: `rotasaude/api@main` com o backend importado.

- [ ] **Step 1: Commit limpo sobre o bootstrap**

```bash
cd /tmp/rotasaude-api-migrate/api
git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" \
  commit -q -m "feat: importa backend Rails (api)

Conteúdo de apps/api do monorepo, sem segredos. Compose standalone contra host
Postgres. Ref: docs/superpowers/specs/2026-06-26-api-migration-design.md"
git log --oneline -n 2
```

Expected: 2 commits — `feat: importa backend Rails (api)` em cima de `chore: bootstrap inicial`.

- [ ] **Step 2: Push direto na main (admin bypass)**

```bash
cd /tmp/rotasaude-api-migrate/api
git push origin main 2>&1 | tail -3
```

Expected: `... main -> main` sem rejeição (admin bypass; `enforce_admins=false`).

- [ ] **Step 3: Verificar no remoto**

```bash
gh api repos/rotasaude/api/contents -q '.[].name' | sort | head -40
gh api repos/rotasaude/api/contents/config/master.key 2>&1 | grep -q "Not Found" \
  && echo "OK: master.key NÃO está no remoto" || echo "!! master.key vazou — agir"
gh api repos/rotasaude/api/contents/docker-compose.yml -q .name
```

Expected: lista contém `Dockerfile`, `docker-compose.yml`, `app`, `config`, etc.; `OK: master.key NÃO está no remoto`; `docker-compose.yml`.

---

### Task 8: Smoke test "clone-e-roda" contra host Postgres

**Files:**
- Create (throwaway): `/tmp/rotasaude-api-smoke/api`

**Interfaces:**
- Consumes: `rotasaude/api@main` (Task 7), host Postgres já provisionado, `config/master.key` local.
- Produces: prova de que um clone limpo sobe e responde 200.

- [ ] **Step 1: Clonar limpo e injetar master.key out-of-band**

```bash
rm -rf /tmp/rotasaude-api-smoke
git clone git@github.com:rotasaude/api.git /tmp/rotasaude-api-smoke/api
cp <raiz-do-monorepo>/apps/api/config/master.key \
   /tmp/rotasaude-api-smoke/api/config/master.key
cp /tmp/rotasaude-api-smoke/api/.env.example /tmp/rotasaude-api-smoke/api/.env
```

- [ ] **Step 2: Subir em porta alternativa (evita colisão com o stack da raiz)**

```bash
cd /tmp/rotasaude-api-smoke/api
API_PORT=3031 docker compose up -d --build 2>&1 | tail -15
```

Expected: serviços `rota-saude-api-api-1` e `rota-saude-api-worker-1` criados (sem colidir com `api-dev`).

- [ ] **Step 3: Esperar e verificar `/up`=200**

```bash
for i in $(seq 1 30); do
  code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3031/up 2>/dev/null)
  [ "$code" = "200" ] && { echo "OK: /up=200 após ~${i}s"; break; }
  sleep 2
done
[ "$code" = "200" ] || { echo "!! /up != 200"; docker compose -p rota-saude-api logs api | tail -30; }
```

Expected: `OK: /up=200`. Se falhar, ler logs (causa provável: host Postgres sem o DB provisionado — é o pré-requisito documentado, não regressão da migração).

- [ ] **Step 4: Derrubar o smoke stack**

```bash
cd /tmp/rotasaude-api-smoke/api
docker compose -p rota-saude-api down
echo "smoke test concluído"
```

Expected: containers removidos. (O stack de dev da raiz, `api-dev`/`worker-dev`, segue intacto.)

---

## Notas de verificação final (critério de aceite do spec)

1. ✅ `rotasaude/api@main` com backend, sem segredo (Tasks 6–7).
2. ✅ Clone + master.key + `docker compose up` → `/up`=200 (Task 8).
3. ✅ Pré-requisito host Postgres + limitação from-zero no README (Task 5).
4. ✅ `RECONCILE_*`, `log/`, `tmp/`, `.DS_Store`, `secrets` reais ausentes (Tasks 1–2, 6).
