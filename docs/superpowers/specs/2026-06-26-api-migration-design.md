# Migração `apps/api` → `rotasaude/api` — Design

**Data:** 2026-06-26
**Escopo:** mover o backend Rails do monorepo local (sem git) para o repo GitHub `rotasaude/api`, deixando-o clonável e executável de forma independente.

## Contexto

O código do Rota Saúde existe localmente como monorepo sem git em
`<raiz-do-monorepo>`. Os 7 repos GitHub da org `rotasaude`
foram criados vazios (commit `chore: bootstrap inicial` com README + `.gitignore`) e estão **públicos**
(decisão para habilitar branch protection no plano Free). Este é o primeiro repo a ser populado —
piloto que valida o fluxo de migração antes de replicar para os outros.

Decisões de migração já fechadas com o usuário:
1. Orquestração **descentralizada** — não há repo "dev/infra"; cada repo carrega seu próprio dev/infra.
2. `contracts` é a fonte única dos compartilhados (não afeta o `api`, que é self-contained).
3. Sem preservar histórico — cada repo nasce com **commit limpo** sobre o `chore: bootstrap inicial`.

Verificações já feitas:
- O `api` **não referencia** `../contracts` nem `../packages` — é auto-suficiente.
- Backend saudável: boot limpo, `/up`=200, endpoints protegidos=401, 30 migrations aplicadas
  (blocker antigo `PendingMigration` resolvido e reverificado).

## 1. Escopo do repo

O conteúdo de `apps/api/` vira a **raiz** de `rotasaude/api`.

**Inclui:** `app/`, `bin/`, `config/` (com `credentials.yml.enc` cifrado), `db/`, `lib/`, `spec/`,
`test/`, `public/`, `storage/` (estrutura, sem dados), `Dockerfile`, `Gemfile`, `Gemfile.lock`,
`Rakefile`, `config.ru`, `deploy/` (configs Kamal).

**Exclui (não commitar):** `RECONCILE_admin_console.md` (nota de trabalho), `log/*`, `tmp/*`,
`.DS_Store`, qualquer `storage/*.sqlite3`, e os arquivos de segredo (ver §3).

## 2. Dev/infra standalone (decisão revisada: host Postgres default)

**Descoberta que revisou o design:** as migrations **não** sobem um banco do zero — a
`20260620000001_create_database_roles` cria `rota_app`/`rota_admin`, mas o `db:migrate` conecta
**como `rota_app`** (que ainda não existe num banco zerado). O próprio `bin/docker-entrypoint`
documenta: "migrations incrementais sobre um banco já provisionado... `rota_app` sozinho NÃO
consegue subir um banco vazio". Provisionar do zero (Postgres containerizado efêmero) é um
problema **não resolvido** neste codebase — fica como follow-up, fora deste piloto (ver Out-of-scope).

Novo `docker-compose.yml` **dentro do repo**, derivado do compose da raiz, **apontando para o
Postgres do host** (consistente com o setup de dev atual):

**Serviços (ADR-0001, mesma imagem `rota-saude/api:dev`):**
- `api` — Rails server (`./bin/rails server -b 0.0.0.0`), porta host `${API_PORT:-3030}`→3000,
  `build: context: . / target: development`, volume `.:/rails`.
- `worker` — Solid Queue worker (`./bin/jobs start`), mesma imagem, `depends_on: api`.
- **Sem** `container_name` fixo (evita colisão com o stack da raiz que usa `api-dev`/`worker-dev`).
- Postgres do host via `extra_hosts: host.docker.internal:host-gateway`,
  `DATABASE_HOST=${DATABASE_HOST:-host.docker.internal}`.

**Entrypoint:** mantém `bin/docker-entrypoint` já corrigido (`db:migrate` idempotente). Sem alteração.

**`.env.example`** (subset do `api`): portas, `DATABASE_HOST/PORT`, senhas dos roles, WhatsApp dev,
`PUBLIC_*`, `ALLOWED_ORIGINS`, `WORKER_MAX_THREADS`.

**README** expandido documenta: pré-requisito de **host Postgres provisionado** (owner `rota_saude`
+ databases, conforme `start.sh`), que `config/master.key` deve ser colocado **out-of-band** (não
está no repo), `docker compose up`, verificação `curl :3030/up`, e a **limitação conhecida** de
bootstrap-from-zero (follow-up).

## 3. Segurança (repo público — inegociável)

**`.gitignore` reforçado antes do primeiro commit.** Confirmar/garantir entradas:
- `/config/master.key` (já presente) — fica **local**, nunca commitado.
- `/config/credentials/*.key` (já presente).
- `/.env*` com exceção `!/.env.example` (já presente).
- **Adicionar:** `.DS_Store`, `/deploy/*/secrets` (templates Kamal — não commitar o arquivo real).

**Templates visíveis:** commitar `deploy/development/secrets.example` e
`deploy/production/secrets.example` (cópias dos atuais, que já não têm plaintext) para preservar a
estrutura sem versionar o arquivo `secrets` em si.

**Pode commitar:** `config/credentials.yml.enc` (cifrado — inútil sem `master.key`),
`deploy/SECRETS.md` (doc de inventário ADR-0024, sem valores).

**Gate de verificação pré-push (bloqueante):** antes do `git push`, rodar um scan no índice
(`git grep -nE` por padrões de chave/token/senha em base64/hex longas, `BEGIN ... PRIVATE KEY`,
`master.key` material) sobre **tudo que está staged**. Se achar qualquer segredo → abortar push,
não improvisar. Só pushar com o scan limpo.

## 4. Commit & push

1. `mkdir` em diretório de trabalho temporário; copiar `apps/api/` aplicando as exclusões da §1.
2. `git init -b main`; aplicar `.gitignore` reforçado (§3).
3. `git remote add origin git@github.com:rotasaude/api.git`; `git fetch` para herdar o
   `chore: bootstrap inicial` remoto.
4. Commit limpo único sobre o bootstrap (decisão 3), ex.: `feat: importa backend Rails (api)`.
5. Rodar o **gate de verificação pré-push** (§3).
6. **Push direto na `main`** (decisão 4) — admin bypass, já que `enforce_admins=false`.
7. Verificar: `gh repo view rotasaude/api`, CI inexistente (esperado), `:3000/up` ainda 200 num
   clone limpo do repo (smoke test do "clone-e-roda").

## Critério de aceite

1. `rotasaude/api` em `main` contém o backend Rails sem nenhum segredo (scan pré-push limpo).
2. `git clone` do repo + colocar `config/master.key` out-of-band + `docker compose up` sobe
   `api`+`worker` contra o **host Postgres provisionado** e `curl :3030/up`=200, sem depender de
   nenhum arquivo do monorepo raiz.
3. Pré-requisito de host Postgres + limitação de bootstrap-from-zero documentados no README.
4. `RECONCILE_admin_console.md`, `log/`, `tmp/`, `.DS_Store`, arquivos `secrets` reais ausentes do repo.

## Out-of-scope

- Migração dos outros 6 repos (replicar o padrão depois, repo a repo).
- Resolver `contracts` como fonte única (não afeta o `api`).
- **Bootstrap-from-zero / Postgres containerizado efêmero** — requer resolver o chicken-and-egg de
  criação de roles (rota_app criado por migration que conecta como rota_app). Follow-up próprio
  (issue separada), provavelmente um bootstrap privilegiado ou seed por `pg_dump`.
- Configurar CI/Actions (setup separado por repo).
- Mudar branch protection ou visibilidade dos repos.
- Qualquer refactor de código de aplicação do `api`.
