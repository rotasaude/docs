# Fase 0 — Organizar o funcionamento dos apps em desenvolvimento

- **Data:** 2026-06-24
- **Status:** Aprovado (design); pendente de plano de implementação
- **Escopo:** ambiente de **desenvolvimento** local (Docker Compose). Produção (Kamal) fora de escopo.

## Objetivo

Fazer as quatro superfícies do Rota Saúde — `api`, `admin`, `dashboard`, `wpda` —
subirem juntas em desenvolvimento, cada uma como repositório próprio (ADR-0002),
com orquestração e nomenclatura corretas. **Não** implementa telas/painéis de
domínio; entrega a *fundação* sobre a qual os módulos serão construídos em specs
seguintes.

## Contexto e decisões de governança

- **Multi-repo (ADR-0002).** Cada app é repositório próprio. `apps/api`,
  `apps/admin` e `apps/wpda` já têm `.git`. **Não** se adota workspace
  npm/pnpm — isso superseria a ADR-0002 e é decisão à parte. Os `packages/ui`,
  `packages/types`, `packages/protocols` permanecem stubs de contrato (só README,
  exceto `protocols/schema.json`) e **não** são integrados nesta fase.
- **Reconciliação de naming.** O `apps/admin` carrega herança de quando se
  chamava "dashboard" (package `@rota-saude/dashboard`, Vite `base: "/dashboard/"`).
  Agora que `dashboard` é app próprio, isso colide e é corrigido aqui.
- **`web` vira `wpda`.** O serviço `web` do compose (stub de "autoria de
  protocolo", ADR-0016) é a superfície que o `wpda` materializa. O stub `web` é
  **removido**; `wpda` ocupa o lugar.

## Stack (espelha `apps/admin`)

Vite 5 · React 18 · TypeScript 5.6 (strict) · `@tanstack/react-query`.
`tsconfig.json` idêntico ao do admin (target ES2022, `moduleResolution: Bundler`,
strict + `noUnusedLocals`/`noUnusedParameters`). Gerenciador de pacotes: **npm**
(o compose roda `npm install`).

## Padrão de execução em dev (todos os frontends)

Cada serviço frontend no `docker-compose.yml`:

- imagem `node:22-alpine`, `working_dir: /app`;
- `command: sh -c "npm install --silent --no-audit --no-fund && npm run dev -- --host 0.0.0.0"`;
- bind mount do código (`./apps/<app>:/app`) + **named volume** para `node_modules`
  (evita lentidão de bind no Mac e conflito host×container);
- `depends_on: api` (`condition: service_started`);
- `environment: VITE_API_PROXY_TARGET=http://api:3000` (alvo in-container);
- `restart: unless-stopped`;
- Vite `server.host: "0.0.0.0"`, `port: 5173` interno, mapeado pra porta de host
  própria.

## Unidades de trabalho

### U1 — Reconciliar `admin` (repo `apps/admin`)

Apenas identidade; nenhuma lógica muda. Porta dev permanece **5174**.

| Arquivo | Mudança |
|---|---|
| `apps/admin/package.json` | `name`: `@rota-saude/dashboard` → `@rota-saude/admin` |
| `apps/admin/vite.config.ts` | `base: "/dashboard/"` → `base: "/admin/"` |
| `apps/admin/.env.example` | linha de acesso `…/dashboard/` → `…/admin/` |

Verificação: `grep -rn '/dashboard/' apps/admin` retorna vazio depois (fora de
`node_modules`); `npm run build` no admin continua passando.

### U2 — Scaffold `dashboard` (repo novo `apps/dashboard`)

Hoje é casco (sem `.git`, sem `package.json`; só `node_modules/` vazio e um
`package-lock.json` de 82 bytes). Esvaziar o casco e criar app mínimo funcional:

Arquivos:
- `package.json` (`@rota-saude/dashboard`, scripts `dev`/`build`/`preview`/`typecheck`,
  deps: react, react-dom, @tanstack/react-query; devDeps: vite, @vitejs/plugin-react,
  typescript, @types/react, @types/react-dom).
- `tsconfig.json` (cópia do admin).
- `vite.config.ts`: `base: "/dashboard/"`, `server.port: 5173`, `host: "0.0.0.0"`,
  proxy de `/up`, `/admin/api` e `/session` para `VITE_API_PROXY_TARGET`
  (default `http://localhost:3030`). `/up` é incluído justamente para o
  health-ping sem auth.
- `index.html` (`<title>Rota Saúde · Dashboard</title>`).
- `.env.example` (`VITE_API_PROXY_TARGET=http://localhost:3030`, base da API).
- `.gitignore` (node_modules, dist, .env).
- `src/main.tsx`, `src/App.tsx`, `src/vite-env.d.ts`.
- `src/theme/global.css` + `src/theme/tokens.ts` — **copiados do admin** (baseline
  visual compartilhado; barato e evita divergência nos módulos futuros).

Comportamento da `App.tsx`: tela inicial que faz um *health-ping*
(`GET /up`, proxiado para o Rails, sem auth) e renderiza o status
(carregando / ok 200 / erro). É a prova viva de que o proxy dev→Rails funciona.
Nenhum painel de domínio.

Git: `git init` em `apps/dashboard` (alinha com ADR-0002 — cada app, um repo).
Porta dev **5175**.

### U3 — Scaffold `wpda` (repo existente `apps/wpda`)

`apps/wpda` já tem `.git` e `deploy/{development,production}/.env`. **Não** se
recria o repo nem se mexe em `deploy/`. Preenche-se o conteúdo (hoje
`package.json` vazio, `src/` vazio) com a mesma estrutura de U2, exceto:
- `package.json` `@rota-saude/wpda`;
- `vite.config.ts` `base: "/wpda/"`;
- `index.html` `<title>Rota Saúde · WPDA</title>`;
- porta dev **5176**.

### U4 — `docker-compose.yml`

- Serviço `dashboard` **atual** (que monta o casco `./apps/dashboard`) → renomeado
  para **`admin`**, montando `./apps/admin`, volume `admin-node-modules`, porta
  `${ADMIN_PORT:-5174}`, env Vite do admin.
- **Novo** serviço `dashboard` → `./apps/dashboard`, volume `dashboard-node-modules`,
  porta `${DASHBOARD_PORT:-5175}`.
- **Novo** serviço `wpda` → `./apps/wpda`, volume `wpda-node-modules`, porta
  `${WPDA_PORT:-5176}`.
- Serviço `web` (stub) **removido**.
- Bloco `volumes:` atualizado: `rails-tmp`, `admin-node-modules`,
  `dashboard-node-modules`, `wpda-node-modules`.
- Comentários do topo atualizados (lista de serviços e exemplos de `up`).

### U5 — `.env` e `.env.example` (root)

- Adicionar `ADMIN_PORT=5174`, `DASHBOARD_PORT=5175`, `WPDA_PORT=5176`.
- Remover/observar `WEB_PORT` (não há mais serviço `web`).
- `ALLOWED_ORIGINS` passa a incluir as três origens:
  `http://localhost:5174,http://localhost:5175,http://localhost:5176`.

### U6 — `start.sh`

- Bloco final de URLs lista os três frontends (admin/dashboard/wpda) com portas.
- Após `compose up -d`, exibir `docker compose ps` dos frontends para **não
  mascarar crash** (hoje só a `api` tem healthcheck).
- Healthcheck bloqueante continua **só na `api`** — `npm install` dos frontends é
  lento e não deve travar o boot do backend. Justificativa registrada em
  comentário no script. (Healthcheck real de cada Vite fica como melhoria
  opcional futura.)

## Critérios de aceite

1. `./start.sh` sobe `api` + `worker`, e `docker compose up -d` sobe também
   `admin`, `dashboard`, `wpda` sem crash-loop.
2. `http://localhost:3030/up` → 200 (inalterado).
3. Cada frontend responde no browser na sua porta/base:
   `:5174/admin/`, `:5175/dashboard/`, `:5176/wpda/`, e a tela inicial mostra o
   status do health-ping da API (proxy funcionando).
4. `grep -rn '/dashboard/' apps/admin` (fora de `node_modules`) retorna vazio;
   `apps/admin/package.json` é `@rota-saude/admin`.
5. Não existe mais serviço `web` no compose; não há serviço apontando para
   diretório vazio.
6. `apps/dashboard` tem `.git`; `apps/wpda` manteve seu `.git` e `deploy/`.

## Fora de escopo (specs seguintes)

- 9 painéis do `dashboard` (recharts, contratos de dados, navegação por módulo).
- Autoria de protocolo no `wpda`.
- Integração real com `packages/ui` / `packages/types`.
- MFA, rate-limiting e reset de senha do `admin`.
- Healthcheck Docker dedicado por frontend.
- Qualquer mudança em produção/Kamal.

## Riscos e notas

- **Bind-mount duplicado stale** (RECONCILE §6): há uma segunda cópia do repo
  fora deste path que fica desatualizada. Fora do escopo desta fase, mas vale
  consolidar.
- **Bootstrap de roles Postgres** (`rota_app`/`rota_admin`): criados por migração;
  inalterado aqui, mas é ponto frágil de first-run a observar.
- **Spec não versionado:** `docs/` não está sob repositório git, então este
  documento não é commitado (registro em arquivo apenas).
