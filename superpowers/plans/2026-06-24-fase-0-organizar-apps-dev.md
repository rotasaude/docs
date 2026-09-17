# Fase 0 — Organizar o funcionamento dos apps em dev · Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fazer `api`, `admin`, `dashboard` e `wpda` subirem juntos em desenvolvimento, cada um repo próprio, com orquestração e naming corretos — sem telas de domínio.

**Architecture:** Multi-repo (ADR-0002): `apps/admin`, `apps/dashboard`, `apps/wpda` são repos git independentes; o `docker-compose.yml` na raiz os orquestra. Cada frontend é Vite+React+TS servido por container `node:22-alpine`, proxiando rotas de backend para o Rails (`apps/api`, exposto em `:3030`). Postgres é externo (host). Sem workspace, sem integração de `packages/`.

**Tech Stack:** Vite 5 · React 18 · TypeScript 5.6 (strict) · `@tanstack/react-query` · npm · Docker Compose.

## Global Constraints

- **Multi-repo (ADR-0002):** nenhum workspace npm/pnpm; nenhum `package.json` na raiz; `packages/*` permanecem stubs (não integrar).
- **Stack pinada (espelha `apps/admin`):** `react@^18.3.1`, `react-dom@^18.3.1`, `@tanstack/react-query@^5.59.0`, `vite@^5.4.8`, `@vitejs/plugin-react@^4.3.1`, `typescript@^5.6.2`, `@types/react@^18.3.10`, `@types/react-dom@^18.3.0`.
- **Sem telas de domínio:** cada novo app entrega só tela inicial + health-ping `GET /up`.
- **Gate por tarefa:** typecheck + build + boot. NÃO há harness de teste unitário (decisão YAGNI da fundação).
- **Portas dev:** admin `5174`, dashboard `5175`, wpda `5176`. Vite interno sempre `5173`.
- **Proxy dev:** alvo `VITE_API_PROXY_TARGET`; default local `http://localhost:3030`, no compose `http://api:3000`.
- **Commits:** só `apps/admin`, `apps/dashboard`, `apps/wpda` são repos. Arquivos de raiz (`docker-compose.yml`, `.env*`, `start.sh`) **não** são commitáveis (raiz não é repo). Toda mensagem de commit termina com o trailer `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`.

---

### Task 1: Reconciliar `admin` (repo `apps/admin`)

Só identidade; nenhuma lógica muda. Porta permanece 5174.

**Files:**
- Modify: `apps/admin/package.json` (campo `name`)
- Modify: `apps/admin/vite.config.ts` (campo `base`)
- Modify: `apps/admin/.env.example` (linha de acesso)

**Interfaces:**
- Produces: app `admin` servido em base `/admin/`, package `@rota-saude/admin`. O serviço de compose (Task 4) e o `ALLOWED_ORIGINS` (Task 4) dependem da porta 5174 e da base `/admin/`.

- [ ] **Step 1: Renomear o package**

Em `apps/admin/package.json`, trocar:
```json
  "name": "@rota-saude/dashboard",
```
por:
```json
  "name": "@rota-saude/admin",
```

- [ ] **Step 2: Corrigir a base do Vite**

Em `apps/admin/vite.config.ts`, trocar:
```ts
  base: "/dashboard/",
```
por:
```ts
  base: "/admin/",
```

- [ ] **Step 3: Corrigir a linha de acesso no `.env.example`**

Em `apps/admin/.env.example`, trocar a ocorrência `http://localhost:5174/dashboard/` por `http://localhost:5174/admin/`.

- [ ] **Step 4: Verificar que não sobrou `/dashboard/` no admin**

Run: `grep -rn '/dashboard/' apps/admin --exclude-dir=node_modules --exclude-dir=dist`
Expected: nenhuma linha (exit 1).

- [ ] **Step 5: Verificar que o build do admin continua passando**

Run: `cd apps/admin && npm install --silent && npm run build`
Expected: build conclui sem erro; gera `dist/`.

- [ ] **Step 6: Commit (no repo `apps/admin`)**

```bash
cd apps/admin && git add package.json vite.config.ts .env.example && git commit -m "refactor: reconciliar naming admin (dashboard -> admin)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 2: Scaffold `dashboard` (repo novo `apps/dashboard`)

`apps/dashboard` hoje é casco (sem `.git`, sem `package.json`; só `node_modules/` vazio e `package-lock.json` de 82 bytes). Esvaziar e criar app mínimo funcional.

**Files:**
- Delete: `apps/dashboard/package-lock.json` (stub de 82 bytes), `apps/dashboard/node_modules` (vazio), `apps/dashboard/.DS_Store`
- Create: `apps/dashboard/package.json`, `tsconfig.json`, `vite.config.ts`, `index.html`, `.env.example`, `.gitignore`
- Create: `apps/dashboard/src/main.tsx`, `src/App.tsx`, `src/vite-env.d.ts`
- Create: `apps/dashboard/src/theme/global.css`, `src/theme/tokens.ts` (copiados do admin)

**Interfaces:**
- Consumes: convenções de stack do admin (versões pinadas em Global Constraints).
- Produces: app `dashboard` em base `/dashboard/`, package `@rota-saude/dashboard`; o serviço de compose (Task 4) monta `./apps/dashboard` e expõe na porta 5175.

- [ ] **Step 1: Limpar o casco**

```bash
rm -f apps/dashboard/package-lock.json apps/dashboard/.DS_Store
rmdir apps/dashboard/node_modules 2>/dev/null || true
mkdir -p apps/dashboard/src/theme
```

- [ ] **Step 2: Criar `apps/dashboard/package.json`**

```json
{
  "name": "@rota-saude/dashboard",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@tanstack/react-query": "^5.59.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@types/react": "^18.3.10",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.1",
    "typescript": "^5.6.2",
    "vite": "^5.4.8"
  }
}
```

- [ ] **Step 3: Copiar `tsconfig.json` e o tema do admin**

```bash
cp apps/admin/tsconfig.json apps/dashboard/tsconfig.json
cp apps/admin/.gitignore apps/dashboard/.gitignore
cp apps/admin/src/theme/global.css apps/dashboard/src/theme/global.css
cp apps/admin/src/theme/tokens.ts apps/dashboard/src/theme/tokens.ts
```

- [ ] **Step 4: Criar `apps/dashboard/vite.config.ts`**

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

// Frontend do Dashboard operacional da cidade (tenant-scoped, municipal_admin).
// Em dev, Vite proxa as rotas de backend para o Rails (apps/api, :3030).
//   /up         → healthcheck do Rails (sem auth), usado pelo health-ping.
//   /admin/api  → Admin::Api::* (read-only).
//   /session    → SessionsController.
const proxy = (target: string) => ({ target, changeOrigin: true });
const TARGET = process.env.VITE_API_PROXY_TARGET || "http://localhost:3030";

export default defineConfig({
  plugins: [react()],
  base: "/dashboard/",
  server: {
    port: 5173,
    host: "0.0.0.0",
    proxy: {
      "/up":        proxy(TARGET),
      "/admin/api": proxy(TARGET),
      "/session":   proxy(TARGET)
    }
  }
});
```

- [ ] **Step 5: Criar `apps/dashboard/index.html`**

```html
<!doctype html>
<html lang="pt-BR">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Rota Saúde · Dashboard</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

- [ ] **Step 6: Criar `apps/dashboard/.env.example`**

```
# Em dev, alvo do proxy (Rails container exposto em 3030).
VITE_API_PROXY_TARGET=http://localhost:3030

# Acesso: http://localhost:5175/dashboard/
```

- [ ] **Step 7: Criar `apps/dashboard/src/vite-env.d.ts`**

```ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_PROXY_TARGET?: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

- [ ] **Step 8: Criar `apps/dashboard/src/main.tsx`**

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { App } from "./App";
import "./theme/global.css";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

- [ ] **Step 9: Criar `apps/dashboard/src/App.tsx`**

```tsx
import { useEffect, useState } from "react";

type Health =
  | { kind: "loading" }
  | { kind: "ok"; status: number }
  | { kind: "error"; detail: string };

export function App() {
  const [health, setHealth] = useState<Health>({ kind: "loading" });

  useEffect(() => {
    fetch("/up")
      .then((res) => setHealth({ kind: "ok", status: res.status }))
      .catch((err) => setHealth({ kind: "error", detail: String(err) }));
  }, []);

  return (
    <main style={{ fontFamily: "var(--font-mono, monospace)", padding: 24 }}>
      <h1>Rota Saúde · Dashboard</h1>
      <p>Painel operacional da cidade (tenant-scoped). Fundação — sem painéis ainda.</p>
      <HealthLine health={health} />
    </main>
  );
}

function HealthLine({ health }: { health: Health }) {
  if (health.kind === "loading") return <p>API: verificando…</p>;
  if (health.kind === "ok") return <p>API /up: {health.status} ✓ (proxy dev→Rails ok)</p>;
  return <p>API /up: falhou — {health.detail}</p>;
}
```

- [ ] **Step 10: Instalar deps e verificar typecheck + build**

Run: `cd apps/dashboard && npm install && npm run build`
Expected: instala, `tsc -b` sem erros, gera `dist/`.

- [ ] **Step 11: Verificar boot do dev server (smoke)**

Run: `cd apps/dashboard && (npm run dev -- --host 0.0.0.0 & sleep 4; curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:5173/dashboard/; kill %1)`
Expected: imprime `200`.

- [ ] **Step 12: Init repo e commit**

```bash
cd apps/dashboard && git init -q && git add -A && git commit -q -m "feat: scaffold dashboard (Vite+React+TS, health-ping)

Fundação do app operacional da cidade. Sem painéis ainda (ADR-0002, multi-repo).

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 3: Scaffold `wpda` (repo existente `apps/wpda`)

`apps/wpda` já tem `.git` e `deploy/{development,production}/.env`. **Não** recriar repo nem tocar em `deploy/`. Preencher conteúdo (hoje `package.json` vazio, `src/` com stubs).

**Files:**
- Delete: stubs vazios em `apps/wpda/src/*` (se existirem)
- Create/overwrite: `apps/wpda/package.json`, `tsconfig.json`, `vite.config.ts`, `index.html`, `.env.example`
- Create: `apps/wpda/src/main.tsx`, `src/App.tsx`, `src/vite-env.d.ts`, `src/theme/global.css`, `src/theme/tokens.ts`

**Interfaces:**
- Produces: app `wpda` em base `/wpda/`, package `@rota-saude/wpda`; o serviço de compose (Task 4) monta `./apps/wpda` e expõe na porta 5176.

- [ ] **Step 1: Preparar diretórios e limpar stubs**

```bash
ls -la apps/wpda/src
rm -rf apps/wpda/src/*
mkdir -p apps/wpda/src/theme
```

- [ ] **Step 2: Criar `apps/wpda/package.json`**

```json
{
  "name": "@rota-saude/wpda",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@tanstack/react-query": "^5.59.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@types/react": "^18.3.10",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.1",
    "typescript": "^5.6.2",
    "vite": "^5.4.8"
  }
}
```

- [ ] **Step 3: Copiar `tsconfig.json` e o tema do admin**

```bash
cp apps/admin/tsconfig.json apps/wpda/tsconfig.json
cp apps/admin/src/theme/global.css apps/wpda/src/theme/global.css
cp apps/admin/src/theme/tokens.ts apps/wpda/src/theme/tokens.ts
```

(`apps/wpda/.gitignore` já existe e cobre node_modules/dist — não sobrescrever.)

- [ ] **Step 4: Criar `apps/wpda/vite.config.ts`**

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

// Frontend WPDA — autoria de protocolo da cidade.
// Em dev, Vite proxa as rotas de backend para o Rails (apps/api, :3030).
//   /up         → healthcheck do Rails (sem auth), usado pelo health-ping.
//   /admin/api  → Admin::Api::* (read-only).
//   /session    → SessionsController.
const proxy = (target: string) => ({ target, changeOrigin: true });
const TARGET = process.env.VITE_API_PROXY_TARGET || "http://localhost:3030";

export default defineConfig({
  plugins: [react()],
  base: "/wpda/",
  server: {
    port: 5173,
    host: "0.0.0.0",
    proxy: {
      "/up":        proxy(TARGET),
      "/admin/api": proxy(TARGET),
      "/session":   proxy(TARGET)
    }
  }
});
```

- [ ] **Step 5: Criar `apps/wpda/index.html`**

```html
<!doctype html>
<html lang="pt-BR">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Rota Saúde · WPDA</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

- [ ] **Step 6: Criar `apps/wpda/.env.example`**

```
# Em dev, alvo do proxy (Rails container exposto em 3030).
VITE_API_PROXY_TARGET=http://localhost:3030

# Acesso: http://localhost:5176/wpda/
```

- [ ] **Step 7: Criar `apps/wpda/src/vite-env.d.ts`**

```ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_PROXY_TARGET?: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

- [ ] **Step 8: Criar `apps/wpda/src/main.tsx`**

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { App } from "./App";
import "./theme/global.css";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

- [ ] **Step 9: Criar `apps/wpda/src/App.tsx`**

```tsx
import { useEffect, useState } from "react";

type Health =
  | { kind: "loading" }
  | { kind: "ok"; status: number }
  | { kind: "error"; detail: string };

export function App() {
  const [health, setHealth] = useState<Health>({ kind: "loading" });

  useEffect(() => {
    fetch("/up")
      .then((res) => setHealth({ kind: "ok", status: res.status }))
      .catch((err) => setHealth({ kind: "error", detail: String(err) }));
  }, []);

  return (
    <main style={{ fontFamily: "var(--font-mono, monospace)", padding: 24 }}>
      <h1>Rota Saúde · WPDA</h1>
      <p>Autoria de protocolo da cidade. Fundação — sem editor ainda.</p>
      <HealthLine health={health} />
    </main>
  );
}

function HealthLine({ health }: { health: Health }) {
  if (health.kind === "loading") return <p>API: verificando…</p>;
  if (health.kind === "ok") return <p>API /up: {health.status} ✓ (proxy dev→Rails ok)</p>;
  return <p>API /up: falhou — {health.detail}</p>;
}
```

- [ ] **Step 10: Instalar deps e verificar typecheck + build**

Run: `cd apps/wpda && npm install && npm run build`
Expected: instala, `tsc -b` sem erros, gera `dist/`.

- [ ] **Step 11: Verificar boot do dev server (smoke)**

Run: `cd apps/wpda && (npm run dev -- --host 0.0.0.0 & sleep 4; curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:5173/wpda/; kill %1)`
Expected: imprime `200`.

- [ ] **Step 12: Commit (no repo `apps/wpda` existente)**

```bash
cd apps/wpda && git add -A && git commit -q -m "feat: scaffold wpda (Vite+React+TS, health-ping)

Fundação do app de autoria de protocolo. Sem editor ainda.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 4: `docker-compose.yml` + `.env` (raiz — não commitável)

Renomear o serviço `dashboard` (que monta o casco) para `admin`; adicionar serviços `dashboard` e `wpda`; remover o stub `web`; atualizar volumes e env.

**Files:**
- Modify: `docker-compose.yml`
- Modify: `.env`, `.env.example`

**Interfaces:**
- Consumes: apps em `./apps/{admin,dashboard,wpda}` (Tasks 1–3), portas 5174/5175/5176.

- [ ] **Step 1: Substituir o serviço `web` e o `dashboard` antigo pelos três frontends**

Em `docker-compose.yml`, remover o bloco do serviço `web` **e** o bloco do serviço `dashboard` atual, e inserir no lugar (mesmo nível de indentação dos demais `services:`):

```yaml
  admin:
    # Admin Console — Vite + React (apps/admin). Cross-tenant, platform_operator.
    image: node:22-alpine
    working_dir: /app
    command: sh -c "npm install --silent --no-audit --no-fund && npm run dev -- --host 0.0.0.0"
    environment:
      VITE_API_PROXY_TARGET: ${VITE_API_PROXY_TARGET:-http://api:3000}
    volumes:
      - ./apps/admin:/app
      - admin-node-modules:/app/node_modules
    ports:
      - "${ADMIN_PORT:-5174}:5173"
    depends_on:
      api:
        condition: service_started
    restart: unless-stopped

  dashboard:
    # Dashboard operacional da cidade — Vite + React (apps/dashboard). Tenant-scoped.
    image: node:22-alpine
    working_dir: /app
    command: sh -c "npm install --silent --no-audit --no-fund && npm run dev -- --host 0.0.0.0"
    environment:
      VITE_API_PROXY_TARGET: ${VITE_API_PROXY_TARGET:-http://api:3000}
    volumes:
      - ./apps/dashboard:/app
      - dashboard-node-modules:/app/node_modules
    ports:
      - "${DASHBOARD_PORT:-5175}:5173"
    depends_on:
      api:
        condition: service_started
    restart: unless-stopped

  wpda:
    # WPDA — autoria de protocolo da cidade. Vite + React (apps/wpda).
    image: node:22-alpine
    working_dir: /app
    command: sh -c "npm install --silent --no-audit --no-fund && npm run dev -- --host 0.0.0.0"
    environment:
      VITE_API_PROXY_TARGET: ${VITE_API_PROXY_TARGET:-http://api:3000}
    volumes:
      - ./apps/wpda:/app
      - wpda-node-modules:/app/node_modules
    ports:
      - "${WPDA_PORT:-5176}:5173"
    depends_on:
      api:
        condition: service_started
    restart: unless-stopped
```

- [ ] **Step 2: Atualizar o bloco `volumes:` no fim do arquivo**

```yaml
volumes:
  rails-tmp:
  admin-node-modules:
  dashboard-node-modules:
  wpda-node-modules:
```

- [ ] **Step 3: Atualizar os comentários do topo do arquivo**

No cabeçalho do `docker-compose.yml`, trocar a seção de exemplos de uso por:
```yaml
# Uso:
#   docker compose up                       # api -> worker + admin/dashboard/wpda
#   docker compose up api worker            # só backend
#   docker compose up admin dashboard wpda  # frontends (precisam do api de pé)
#   docker compose down                     # encerra
```

- [ ] **Step 4: Atualizar `.env` e `.env.example` (raiz)**

Em ambos, remover a linha `WEB_PORT=5173` e adicionar, na seção de portas:
```
ADMIN_PORT=5174
DASHBOARD_PORT=5175
WPDA_PORT=5176
```
E trocar a linha de CORS por:
```
ALLOWED_ORIGINS=http://localhost:5174,http://localhost:5175,http://localhost:5176
```

- [ ] **Step 5: Validar o compose**

Run: `docker compose config >/dev/null && echo OK`
Expected: imprime `OK` (sem erro de sintaxe/refs); nenhum serviço `web`.

- [ ] **Step 6: Confirmar serviços e ausência do `web`**

Run: `docker compose config --services | sort`
Expected: exatamente `admin`, `api`, `dashboard`, `worker`, `wpda` (sem `web`).

(Sem commit: a raiz não é repositório git.)

---

### Task 5: `start.sh` (raiz — não commitável)

Listar os três frontends no bloco final e mostrar o status deles pós-boot.

**Files:**
- Modify: `start.sh`

- [ ] **Step 1: Estender o bloco de URLs finais**

No heredoc final do `start.sh` (o bloco `cat <<EOF ... EOF`), adicionar após a linha do `webhook`:
```
  admin ................. http://localhost:${ADMIN_PORT:-5174}/admin/
  dashboard ............. http://localhost:${DASHBOARD_PORT:-5175}/dashboard/
  wpda .................. http://localhost:${WPDA_PORT:-5176}/wpda/
```

- [ ] **Step 2: Mostrar status dos frontends pós-boot (não mascarar crash)**

Logo após o bloco de healthcheck da `api` (depois do `for i in $(seq 1 30)`), adicionar:
```bash
# Frontends sobem com npm install (lento); não bloqueiam o boot do backend.
# Mostramos o status pra não mascarar crash-loop silencioso.
say "status dos frontends (npm install pode levar ~1min na 1ª vez)"
docker compose ps admin dashboard wpda --format 'table {{.Name}}\t{{.Status}}\t{{.Ports}}'
```

- [ ] **Step 3: Validar sintaxe do script**

Run: `bash -n start.sh && echo OK`
Expected: imprime `OK`.

(Sem commit: a raiz não é repositório git.)

---

### Task 6: Aceitação end-to-end (boot real da stack)

Prova de que tudo sobe e o proxy dev→Rails funciona.

**Files:** nenhum (verificação).

- [ ] **Step 1: Subir a stack**

Run: `./start.sh`
Expected: api responde `/up` 200; o bloco de status mostra `admin`, `dashboard`, `wpda`.

- [ ] **Step 2: Aguardar o npm install dos frontends e checar que estão de pé**

Run: `sleep 60; docker compose ps admin dashboard wpda --format 'table {{.Name}}\t{{.Status}}'`
Expected: os três `Up` (não `Restarting`).

- [ ] **Step 3: Cada base de frontend responde 200**

Run:
```bash
for p in 5174:admin 5175:dashboard 5176:wpda; do
  port=${p%%:*}; name=${p##*:}
  code=$(curl -sS -o /dev/null -w "%{http_code}" "http://localhost:$port/$name/")
  echo "$name ($port): $code"
done
```
Expected: `admin (5174): 200`, `dashboard (5175): 200`, `wpda (5176): 200`.

- [ ] **Step 4: Proxy dev→Rails funciona (server-side)**

Run: `curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:5175/up`
Expected: `200` (o Vite do dashboard proxiou `/up` para o Rails).

- [ ] **Step 5: Encerrar**

Run: `docker compose down`
Expected: containers encerrados.

---

## Self-Review

**Spec coverage:**
- U1 (reconciliar admin) → Task 1 ✓
- U2 (scaffold dashboard) → Task 2 ✓
- U3 (scaffold wpda) → Task 3 ✓
- U4 (docker-compose) → Task 4 (steps 1–3, 5–6) ✓
- U5 (.env root) → Task 4 step 4 ✓
- U6 (start.sh) → Task 5 ✓
- Critérios de aceite 1–6 do spec → Task 6 + verificações inline ✓
- Remoção do `web` → Task 4 steps 1, 6 ✓
- `git init` dashboard / preservar `.git` wpda → Task 2 step 12 / Task 3 (sem init) ✓

**Placeholder scan:** sem TBD/TODO; todo passo tem comando ou conteúdo exato.

**Type consistency:** o tipo `Health` e o componente `HealthLine` são idênticos entre dashboard e wpda; `App` exportado nomeado em ambos, importado igual em `main.tsx`. Proxies (`/up`,`/admin/api`,`/session`) idênticos. Portas coerentes (5174/5175/5176) entre apps, compose e env.
