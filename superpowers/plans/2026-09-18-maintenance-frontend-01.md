# Frontend da manutenção — Plano 1

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** um frontend novo (`apps/maintenance`) em que o mantenedor entra com senha e TOTP, aceita convite, lê o catálogo e o detalhe de todas as cidades, administra mantenedores e tokens de serviço e consulta a auditoria — falando com a API de manutenção que já existe.

**Architecture:** Vite + React 18 + TypeScript + React Query, no molde do `dashboard`. Um cliente único (`src/lib/api.ts`) põe o header obrigatório em toda chamada e devolve erros tipados; os documentos GraphQL são tipados pelo `graphql-codegen` a partir de um `schema.graphql` extraído do api. Em dev, o proxy do Vite troca o `Host` para `maintenance-api.localhost`, e o navegador vê uma origem só. Navegação por estado no `App`; o convite é lido do fragmento da URL.

**Tech Stack:** Vite 5, React 18, TypeScript 5, @tanstack/react-query 5, graphql 16, @graphql-codegen/cli + client-preset, qrcode, Vitest + Testing Library + jsdom, Playwright, otplib (só no e2e). Rails 8.1 no api (Task 1).

**Spec:** `docs/superpowers/specs/2026-09-18-maintenance-frontend-design.md` (commit `3bc6ec7`). Decisões F1–F8, §4 transporte, §5 identidade, §6 telas, §7 testes.

**Baseline do api:** `main` em `58c8c27`, **1226 exemplos, 0 falhas**.

## Decisões minhas na escrita (cada uma com o custo se estiver errada)

1. **A API ganha a consulta `maintainers` e dois campos em `Maintainer` (Task 1).** A spec lista mantenedores com estado e matrícula, mas a API não tem a consulta e o tipo só expõe `id`, `emailAddress` e `createdAt`. O analisador `HumanOnly` já lista `maintainers` como só de sessão humana, então a regra de acesso existe — falta o campo. **Custo se errado:** um campo de raiz a mais no schema; nenhum dado novo além de dois booleanos.
2. **O link de convite segue o formato que a API já emite: `/invitations#<token>`** (fragmento cru), e não `#token=` como a spec supôs. O `rake maintainer:invite` é quem gera o link; mudar a API por causa da redação da spec seria o caminho errado. **Custo se errado:** nenhum.
3. **O schema vem de `Maintenance::Schema.to_definition` no api, não de introspecção.** O endpoint GraphQL exige credencial para qualquer operação, inclusive a introspecção; o dump do próprio schema é o mesmo SDL sem precisar de sessão. O script `npm run schema:pull` roda a partir da raiz do monorepo. **Custo se errado:** o script depende do layout do monorepo em dev — que é onde ele roda.
4. **O proxy repassa só os dois POSTs de convite** (`/invitations/enroll` e `/invitations/accept`), nunca `/invitations`. O link do convite abre `/invitations` no **frontend**; se o proxy casasse o prefixo, a página iria parar no api. **Custo se errado:** nenhum.
5. **O repositório nasce local (`git init` em `apps/maintenance`), sem remote.** Criar `rotasaude/maintenance` no GitHub é um ato fora desta máquina que cabe ao usuário. **Custo se errado:** um `git remote add` e um push depois.
6. **O compose da raiz está fora do git.** As mudanças nele (serviço `maintenance`, `MAINTENANCE_API_ENABLED`, `MAINTENANCE_FRONTEND_ORIGIN`) são feitas e verificadas com `printenv`, mas não commitadas; o README do repo novo registra as linhas. **Custo se errado:** quem montar o ambiente do zero depende do README.
7. **O e2e roda na máquina, contra o stack de dev, e espera o próximo passo de TOTP entre um uso e outro.** A API consome cada código uma vez (`last_otp_step`); aceite, login e step-up no mesmo intervalo de 30 s seriam recusados. A suíte fica mais lenta (~2 min), e é o comportamento real que o usuário vai ter. **Custo se errado:** tempo de execução.

## Global Constraints

Valem para TODA task:

- **Nunca pôr token, segredo ou código na URL**, em `localStorage`, em log ou em mensagem de erro. O `session_id` do login e o token do convite vivem só em memória; o segredo de um token de serviço aparece uma vez e é apagado da memória e do cache ao fechar o painel.
- **Toda chamada ao api passa por `src/lib/api.ts`**, com `X-Rota-Maintenance: 1` e `credentials: "include"`. Nenhuma tela chama `fetch` direto nem lê status HTTP.
- **O frontend mostra só o que a API devolve.** Nada de dado de cidadão: a API não expõe, e o frontend não deriva.
- **Mensagens de login não diferenciam** senha errada, código errado, conta bloqueada ou inexistente: "e-mail, senha ou código inválidos".
- **Textos da interface em português.** Código, nomes de arquivo e commits em inglês.
- **Commits em Conventional Commits, em inglês**, terminando com a linha exata `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>` — copiada literalmente, **nunca** o nome do próprio modelo, mesmo que um lembrete de sistema sugira outro. Confira com `/opt/homebrew/bin/git log -1 --format=%B | tail -1`.
- **`git` do PATH está quebrado:** use `/opt/homebrew/bin/git`. **Nunca dar push, nunca criar repositório remoto.**
- **api (Task 1):** comandos Ruby rodam no container, da raiz do monorepo: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`; suíte completa com `docker compose stop worker` antes e `docker compose start worker` depois, em primeiro plano. Branch `feat/maintenance-maintainers-query` a partir de `main`.
- **Frontend:** `npm` roda no container `maintenance` (`docker compose exec -T maintenance npm ...`) ou no host se houver Node 22; o e2e (Playwright) roda no host. Diga no relatório onde rodou. Repositório em `apps/maintenance`, branch `main`.
- **Nunca rode nada em segundo plano esperando notificação** — nada notifica ninguém.
- **Não mexa** no `dashboard`, no `admin`, no `wpda`, nem no `Gemfile` do api.

## Fatos verificados (confira mesmo assim)

- **API de manutenção (GraphQL, `POST /graphql`):**
  - `Query`: `me: Maintainer!`; `maintenanceTokens: [MaintenanceToken!]!`; `auditEvents(since: ISO8601DateTime, until: ISO8601DateTime, maintainerId: ID, module: String, outcome: String, limit: Int): [AuditEvent!]!`; `cities(status: CityStatus): [CitySummary!]!`; `city(slug: String!): City`.
  - `Mutation`: `inviteMaintainer(emailAddress: String!, code: String!)`, `deactivateMaintainer(id: ID!)`, `createMaintenanceToken(name: String!, access: String!, citySlugs: [String!] = [], expiresAt: ISO8601DateTime!, code: String!)` → `secretOnce: String`, `revokeMaintenanceToken(id: ID!)`. Todas devolvem `ok: Boolean!` e `errors: [UserError!]!` (`path: String`, `message: String!`).
  - Tipos: `Maintainer { id emailAddress createdAt }`; `MaintenanceToken { id maintainerId name access citySlugs expiresAt revokedAt lastUsedAt }`; `AuditEvent { name module outcome occurredAt maintainerId login correlationId }` — `login` já vem resolvido; `CitySummary { slug name uf status schemaVersion schemaBehind createdAt }` com `status: CityStatus` (enum `PROVISIONING ACTIVE SUSPENDED ARCHIVED`); `City` com os campos de plataforma, `channel`, `profile`, `consentTermVersion`, `protocols`, `alertRecipients`, `accounts`, `counts`, `operations`.
  - Erros de campo de cidade: `extensions.code` ∈ `CITY_UNREACHABLE`, `CITY_ARCHIVED`, `CITY_READ_FAILED`, `CITY_OUT_OF_SCOPE`; recusa de análise: `CITY_BUDGET_EXCEEDED`, `TOKEN_SCOPE_REFUSED`.
  - **Não existe** consulta `maintainers` (Task 1 cria).
- **Sessão (REST, host `maintenance-api.*`):**
  - `POST /session` `{ email_address, password }` → 200 `{ mfa_required: true, session_id }` | 401 `{ error: "invalid_credentials" }` | 429 `{ error: "too_many_requests" }`.
  - `POST /session/challenge` `{ session_id, code }` → 200 `{ id, email_address, mfa_verified_at, expires_at }` | 401 `{ error: "invalid_session" }`.
  - `GET /session` → 200 com o mesmo corpo | 401. `DELETE /session` → encerra.
- **Convite:** `POST /invitations/enroll` `{ token }` → 200 `{ email_address, otpauth_uri, secret }` | 404. `POST /invitations/accept` `{ token, password, code }` → sucesso | 422 `{ error: "weak_password" }` (senha < 12) | 422 `{ error: "invalid_code" }` | 404. Aceitar **não** abre sessão.
- **Link do convite:** `rake "maintainer:invite[email]"` imprime `[maintainer:invite] <MAINTENANCE_FRONTEND_ORIGIN>/invitations#<token>`.
- **CSRF da API:** `Origin` exatamente igual a `MAINTENANCE_FRONTEND_ORIGIN` e header `X-Rota-Maintenance: 1`; falha fechado. Cookie host-only, `SameSite=Strict`.
- **TOTP:** cada código é consumido uma vez (`last_otp_step`); o código do login não serve para o step-up no mesmo passo de 30 s.
- **Dev hoje:** `MAINTENANCE_API_ENABLED` **não** está definida no api (a API de manutenção está desligada); `MAINTENANCE_FRONTEND_ORIGIN` vale `https://maintenance.development.rotasaude.com.br` (linha 42 do `docker-compose.yml` da raiz). O Rails em development aceita hosts `*.localhost`. A rota casa com o rótulo `maintenance-api` (`CityCatalog.maintenance_api_host?`).
- **Molde do `dashboard`:** scripts `dev`/`build` (`tsc -b && vite build`)/`typecheck` (`tsc --noEmit`)/`test` (`vitest run`); `tsconfig.json` strict; `vitest.config.ts` com jsdom e `globals: false`; `src/theme/tokens.ts` + `src/theme/global.css`; CI `.github/workflows/ci.yml` com Node 22 (`npm ci`, typecheck, test, build); serviço no compose com `node:22-alpine`, `npm install && npm run dev -- --host 0.0.0.0`, volume de `node_modules`.

---

## File Structure

**apps/api** (Task 1)
- Modify: `app/graphql/maintenance/types/query_type.rb`, `app/graphql/maintenance/types/maintainer_type.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/maintainers_query_spec.rb`

**apps/maintenance** (novo)
- `package.json`, `tsconfig.json`, `vite.config.ts`, `vitest.config.ts`, `index.html`, `codegen.ts`, `schema.graphql`, `scripts/pull-schema.sh`, `playwright.config.ts`, `README.md`, `.gitignore`, `.github/workflows/ci.yml`
- `src/main.tsx`, `src/App.tsx`, `src/env.ts`
- `src/theme/tokens.ts`, `src/theme/global.css`
- `src/lib/api.ts`, `src/lib/errors.ts`, `src/lib/session.tsx`, `src/lib/invitation.ts`
- `src/gql/` (gerado pelo codegen)
- `src/components/EnvBanner.tsx`, `Panel.tsx`, `DataTable.tsx`, `ErrorState.tsx`, `EmptyState.tsx`, `Field.tsx`, `Button.tsx`, `Tag.tsx`, `ConfirmButton.tsx`
- `src/screens/Login.tsx`, `Invitation.tsx`, `Cities.tsx`, `CityDetail.tsx`, `Maintainers.tsx`, `Tokens.tsx`, `Audit.tsx`, `Shell.tsx`
- testes ao lado de cada arquivo (`*.test.ts(x)`), e `e2e/*.spec.ts`

**Raiz do monorepo (fora do git):** `docker-compose.yml`.

---

### Task 1: A consulta `maintainers` no api

**Files:**
- Modify: `apps/api/app/graphql/maintenance/types/query_type.rb`, `apps/api/app/graphql/maintenance/types/maintainer_type.rb`, `apps/api/spec/architecture/maintenance_schema_spec.rb`
- Test: `apps/api/spec/requests/maintenance/maintainers_query_spec.rb`

**Interfaces:**
- Produces: `Query.maintainers: [Maintainer!]!` (ordenado por e-mail, só sessão humana); `Maintainer.active: Boolean!`, `Maintainer.enrolled: Boolean!`.

- [ ] **Step 1: Escrever o spec que falha**

`spec/requests/maintenance/maintainers_query_spec.rb` — o arranjo de login segue `spec/requests/maintenance/maintainer_mutations_spec.rb` (sem constantes no topo; consultas em métodos):

```ruby
require "rails_helper"

# Spec do frontend §6: a tela de mantenedores lista e-mail, estado e
# matrícula. Só sessão humana — o analisador HumanOnly já lista `maintainers`.
RSpec.describe "Maintenance maintainers query", type: :request do
  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:password) { "s3nha-forte-1" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "zz-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end
  let!(:pending) { Maintainer.create!(email_address: "aa-#{SecureRandom.hex(3)}@rotasaude.app") }
  let!(:gone) do
    Maintainer.create!(email_address: "mm-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current, deactivated_at: Time.current)
  end

  def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
  def json = JSON.parse(response.body)
  def query = "{ maintainers { id emailAddress active enrolled createdAt } }"

  def login!
    post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now },
         headers: browser
    expect(response).to have_http_status(:ok)
  end

  before do
    host! "maintenance-api.rotasaude.app"
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
  end

  it "lists every maintainer by e-mail, with state and enrollment" do
    login!

    post "/graphql", params: { query: query }, headers: browser

    listed = json.dig("data", "maintainers")
    emails = [ pending, gone, maintainer ].map(&:email_address)
    expect(listed.map { |m| m["emailAddress"] } & emails).to eq(emails.sort)
    by_email = listed.index_by { |m| m["emailAddress"] }
    expect(by_email[pending.email_address]).to include("active" => true, "enrolled" => false)
    expect(by_email[gone.email_address]).to include("active" => false, "enrolled" => true)
    expect(by_email[maintainer.email_address]).to include("active" => true, "enrolled" => true)
  end

  it "is refused to a service token" do
    _token, secret = MaintenanceToken.issue!(maintainer: maintainer, name: "ci", access: "read",
                                             city_slugs: [], expires_at: 5.days.from_now)

    post "/graphql", params: { query: query }, headers: { "Authorization" => "Bearer #{secret}", "Cookie" => "" }

    expect(json["errors"].first["extensions"]["code"]).to eq("TOKEN_SCOPE_REFUSED")
    expect(json["data"]).to be_nil
  end

  it "never exposes a credential of a maintainer" do
    login!

    post "/graphql", params: { query: query }, headers: browser

    %w[password_digest otp_secret token_digest].each { |forbidden| expect(response.body).not_to include(forbidden) }
    expect(response.body).not_to include(maintainer.otp_secret)
  end
end
```

**Confira** que `Maintainer.create!` sem senha e sem TOTP é válido (é o estado de um convidado que não se matriculou); se não for, arrume o `pending` do jeito que `MaintainerInvitation`/`rake maintainer:invite` cria.

- [ ] **Step 2: Rodar e confirmar que falha**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/maintenance/maintainers_query_spec.rb`
Expected: FAIL — `maintainers` não existe no schema.

- [ ] **Step 3: Implementar**

Em `MaintainerType`:

```ruby
      field :active, Boolean, null: false, description: "Falso depois de desativado"
      field :enrolled, Boolean, null: false, description: "Senha e TOTP definidos pelo convite"

      def active = object.active?
      def enrolled = object.enrolled?
```

Em `QueryType`, junto de `maintenance_tokens`:

```ruby
      field :maintainers, [ Types::MaintainerType ], null: false,
            description: "Todos os mantenedores, por e-mail. Só sessão humana."

      def maintainers = Maintainer.order(:email_address)
```

`spec/architecture/maintenance_schema_spec.rb`: acrescente `maintainers` aos campos de `Query` e `active`/`enrolled` aos de `Maintainer` em `EXPECTED_TYPES`. **Confira** que `maintainers` já está em `HumanOnly::RESTRICTED` (está, pelo levantamento); se não estiver, acrescente.

- [ ] **Step 4: Rodar, suíte completa e commit (api)**

```bash
cd apps/api
/opt/homebrew/bin/git checkout -b feat/maintenance-maintainers-query
# ... (implementação e testes)
/opt/homebrew/bin/git add app/graphql/maintenance/types/query_type.rb app/graphql/maintenance/types/maintainer_type.rb \
  spec/architecture/maintenance_schema_spec.rb spec/requests/maintenance/maintainers_query_spec.rb
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: list maintainers with their state on the maintenance API

The maintenance frontend lists every maintainer with whether the account
is active and enrolled. Human sessions only, as the analyzer already
declared; no credential of a maintainer leaves.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

- [ ] **Step 5: Ligar a API de manutenção no dev (compose da raiz, fora do git)**

No `docker-compose.yml` da raiz, no mesmo bloco de ambiente da linha `MAINTENANCE_FRONTEND_ORIGIN`:

```yaml
  MAINTENANCE_API_ENABLED: ${MAINTENANCE_API_ENABLED:-true}
  MAINTENANCE_FRONTEND_ORIGIN: ${MAINTENANCE_FRONTEND_ORIGIN:-http://maintenance.localhost:5177}
```

Recrie o api (e o worker, se o bloco for compartilhado): `docker compose up -d api worker`. Confira com `docker compose exec -T api printenv MAINTENANCE_API_ENABLED MAINTENANCE_FRONTEND_ORIGIN` e com `curl -s -o /dev/null -w '%{http_code}' -H 'Host: maintenance-api.localhost' http://localhost:3030/session` (esperado `401`, não `404`). Registre as saídas no relatório. **Não** commite o compose (está fora do git).

---

### Task 2: O repositório e o esqueleto do app

**Files:**
- Create: `apps/maintenance/package.json`, `tsconfig.json`, `vite.config.ts`, `vitest.config.ts`, `index.html`, `.gitignore`, `README.md`, `.github/workflows/ci.yml`, `src/main.tsx`, `src/App.tsx`, `src/env.ts`, `src/theme/tokens.ts`, `src/theme/global.css`, `src/components/EnvBanner.tsx`
- Test: `src/components/EnvBanner.test.tsx`, `src/env.test.ts`
- Modify (fora do git): `docker-compose.yml` da raiz

**Interfaces:**
- Produces:
  - `env.ts`: `export type MaintenanceEnv = "development" | "staging"`; `export function maintenanceEnv(): MaintenanceEnv` (lê `import.meta.env.VITE_MAINTENANCE_ENV`, padrão `"development"`, valor desconhecido lança erro); `export function apiBase(): string` (`""` em development; `import.meta.env.VITE_MAINTENANCE_API_URL` em staging, obrigatória)
  - `EnvBanner` — `<EnvBanner env={...} />`, `role="status"`, texto `DEVELOPMENT` ou `STAGING`, `data-env`

- [ ] **Step 1: Criar o repositório**

```bash
mkdir -p apps/maintenance && cd apps/maintenance && /opt/homebrew/bin/git init -b main
```

- [ ] **Step 2: Arquivos de projeto**

`package.json`:

```json
{
  "name": "@rota-saude/maintenance",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "codegen": "graphql-codegen --config codegen.ts",
    "codegen:check": "graphql-codegen --config codegen.ts --check",
    "schema:pull": "sh scripts/pull-schema.sh",
    "e2e": "playwright test"
  },
  "dependencies": {
    "@fontsource-variable/geist": "^5.2.9",
    "@fontsource-variable/geist-mono": "^5.2.8",
    "@tanstack/react-query": "^5.59.0",
    "graphql": "^16.9.0",
    "qrcode": "^1.5.4",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@graphql-codegen/cli": "^5.0.3",
    "@graphql-codegen/client-preset": "^4.5.0",
    "@playwright/test": "^1.48.0",
    "@testing-library/react": "^16.0.0",
    "@testing-library/user-event": "^14.5.2",
    "@types/node": "^22.0.0",
    "@types/qrcode": "^1.5.5",
    "@types/react": "^18.3.10",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.1",
    "jsdom": "^25.0.0",
    "otplib": "^12.0.1",
    "typescript": "^5.6.2",
    "vite": "^5.4.8",
    "vitest": "^2.1.0"
  }
}
```

`tsconfig.json`: copie o do `apps/dashboard` (strict, `moduleResolution: "Bundler"`, `jsx: "react-jsx"`, `noEmit`), com `"include": ["src", "vite.config.ts", "vitest.config.ts", "codegen.ts", "playwright.config.ts", "e2e"]` e `"types": ["vite/client", "node"]`.

`vitest.config.ts`: igual ao do `dashboard` (jsdom, `globals: false`), com `test.exclude: ["e2e/**", "node_modules/**"]`.

`.gitignore`: `node_modules`, `dist`, `*.tsbuildinfo`, `test-results`, `playwright-report`, `.env*.local`.

`index.html`: um `<div id="root">` e `<script type="module" src="/src/main.tsx">`, `<title>Rota Saúde — Manutenção</title>`, `lang="pt-BR"`.

- [ ] **Step 3: O proxy do Vite**

`vite.config.ts`:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

// Frontend da API de manutenção (spec 2026-09-18-maintenance-frontend-design §4).
//
// Em dev, o navegador fala só com http://maintenance.localhost:5177, e o Vite
// repassa ao api TROCANDO O HOST para maintenance-api.localhost: a rota do api
// só casa com o rótulo `maintenance-api`, e este é o ÚNICO lugar onde o host
// muda. O Origin que o navegador mandou passa intacto — é ele que a API
// compara com MAINTENANCE_FRONTEND_ORIGIN.
//
// /invitations NÃO é repassado como prefixo: o link do convite abre
// /invitations#<token> NO FRONTEND. Só os dois POSTs vão ao api.
const TARGET = process.env.VITE_API_PROXY_TARGET || "http://localhost:3030";
const API_HOST = "maintenance-api.localhost";
const proxy = { target: TARGET, changeOrigin: false, headers: { host: API_HOST } };

export default defineConfig({
  plugins: [ react() ],
  server: {
    port: 5173,
    host: "0.0.0.0",
    allowedHosts: [ ".localhost" ],
    proxy: {
      "/graphql": proxy,
      "/session": proxy,
      "/invitations/enroll": proxy,
      "/invitations/accept": proxy
    }
  }
});
```

**Confira** no container que o `headers.host` do proxy de fato chega ao Rails como `Host: maintenance-api.localhost` (por exemplo, `curl -s -o /dev/null -w '%{http_code}' -H 'Origin: http://maintenance.localhost:5177' http://maintenance.localhost:5177/session` do host deve dar `401`, e o log do api deve mostrar o host). Se o `http-proxy` do Vite não sobrescrever o `Host` por `headers`, use o hook `configure` (`proxy.on("proxyReq", (req) => req.setHeader("host", API_HOST))`) e diga no relatório.

- [ ] **Step 4: Ambiente, tema e faixa**

`src/env.ts`:

```ts
// Onde o app está e para onde ele fala (spec §4). Uma função só decide o
// endereço da API: em dev, relativo (o proxy do Vite troca o host); em
// staging, absoluto, outro host no mesmo site, com o CORS que o api já tem.
export type MaintenanceEnv = "development" | "staging";

export function maintenanceEnv(value: string | undefined = import.meta.env.VITE_MAINTENANCE_ENV): MaintenanceEnv {
  if (value === undefined || value === "") return "development";
  if (value === "development" || value === "staging") return value;
  throw new Error(`VITE_MAINTENANCE_ENV desconhecido: ${value}`);
}

export function apiBase(
  env: MaintenanceEnv = maintenanceEnv(),
  url: string | undefined = import.meta.env.VITE_MAINTENANCE_API_URL
): string {
  if (env === "development") return "";
  if (!url) throw new Error("VITE_MAINTENANCE_API_URL é obrigatória em staging");
  return url.replace(/\/+$/, "");
}
```

`src/env.test.ts` — `maintenanceEnv` devolve `development` por padrão, aceita `staging`, recusa `production` e qualquer outro valor; `apiBase` é `""` em development e exige a URL em staging (remove a barra final).

`src/theme/tokens.ts` e `src/theme/global.css`: copie os do `apps/dashboard` e acrescente:

```ts
  envBanner: {
    development: { bg: "oklch(55% 0.13 245)", ink: "oklch(99% 0 0)" },
    staging:     { bg: "oklch(64% 0.15 55)",  ink: "oklch(20% 0.02 60)" }
  }
```

com as CSS vars correspondentes em `global.css` (`--env-dev-bg`, `--env-dev-ink`, `--env-stg-bg`, `--env-stg-ink`) e a classe `.env-banner` fixa no topo (`position: sticky; top: 0; z-index: 10`), fora de qualquer rolagem.

`src/components/EnvBanner.tsx`:

```tsx
import type { MaintenanceEnv } from "../env";

// Spec F6: com alcance total sobre as cidades, confundir staging com dev é o
// erro mais caro que a tela pode induzir. A faixa nunca sai do topo.
const LABEL: Record<MaintenanceEnv, string> = { development: "DEVELOPMENT", staging: "STAGING" };

export function EnvBanner({ env }: { env: MaintenanceEnv }) {
  return (
    <div role="status" className="env-banner" data-env={env}>
      {LABEL[env]}
    </div>
  );
}
```

`src/components/EnvBanner.test.tsx` — mostra `DEVELOPMENT` com `data-env="development"` e `STAGING` com `data-env="staging"`.

- [ ] **Step 5: Entrada mínima**

`src/App.tsx` por ora renderiza `<EnvBanner env={maintenanceEnv()} />` e um título "Manutenção". `src/main.tsx` monta `<App />` com `StrictMode`, importa as fontes Geist e `./theme/global.css`. (A Task 4 troca isto pelo fluxo de sessão.)

- [ ] **Step 6: CI e README**

`.github/workflows/ci.yml`: o do `dashboard` mais um passo `npm run codegen:check` antes do `test` (a Task 3 cria o codegen; até lá o passo pode ser acrescentado na Task 3 — escolha e diga no relatório).

`README.md`: o que é o app; como subir em dev (serviço `maintenance` no compose, `http://maintenance.localhost:5177`); **as duas linhas de ambiente do api** (`MAINTENANCE_API_ENABLED=true`, `MAINTENANCE_FRONTEND_ORIGIN=http://maintenance.localhost:5177`) e a lembrança de recriar o container e conferir com `printenv`; como convidar o primeiro mantenedor (`docker compose exec api bin/rails "maintainer:invite[email]"`, abrir o link impresso); `npm run schema:pull`/`codegen`; `npm run e2e` e o custo dele no banco de dev (§7 da spec); staging (`VITE_MAINTENANCE_ENV=staging`, `VITE_MAINTENANCE_API_URL`).

- [ ] **Step 7: Serviço no compose (fora do git)**

No `docker-compose.yml` da raiz, depois do `wpda`, no mesmo formato:

```yaml
  maintenance:
    container_name: maintenance-dev
    # Frontend da API de manutenção — Vite + React (apps/maintenance). Superusuário, dev/staging.
    image: node:22-alpine
    working_dir: /app
    command: sh -c "npm install --silent --no-audit --no-fund && npm run dev -- --host 0.0.0.0"
    environment:
      VITE_API_PROXY_TARGET: ${VITE_API_PROXY_TARGET:-http://api:3000}
      VITE_MAINTENANCE_ENV: development
    volumes:
      - ./apps/maintenance:/app
      - maintenance-node-modules:/app/node_modules
    ports:
      - "${MAINTENANCE_PORT:-5177}:5173"
    depends_on:
      api:
        condition: service_started
    restart: unless-stopped
```

e `maintenance-node-modules:` na seção `volumes:`. Suba com `docker compose up -d maintenance`, confira que `http://maintenance.localhost:5177` responde com a faixa `DEVELOPMENT`, e gere o `package-lock.json` (commitado).

- [ ] **Step 8: Rodar e commit**

Run: `npm run typecheck && npm test && npm run build`

```bash
cd apps/maintenance
/opt/homebrew/bin/git add -A
/opt/homebrew/bin/git commit -F - <<'EOF'
chore: scaffold the maintenance frontend

Vite, React and React Query like the other frontends, with the dev proxy
that swaps the host for maintenance-api so the browser sees one origin,
and an environment banner that never leaves the top of the screen.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 3: O cliente da API e os tipos gerados

**Files:**
- Create: `src/lib/errors.ts`, `src/lib/api.ts`, `codegen.ts`, `schema.graphql`, `scripts/pull-schema.sh`, `src/gql/` (gerado)
- Test: `src/lib/api.test.ts`

**Interfaces:**
- Produces:
  - `errors.ts`: `class AuthRequired extends Error`, `class RateLimited extends Error`, `class InvalidCredentials extends Error`, `class RequestRejected extends Error { code: string }` (erro REST com `{ error }` que não é dos acima, ex. `weak_password`, `invalid_code`, `not_found`), `class NetworkError extends Error`, `class GraphQLRefusal extends Error { code: string; path?: (string|number)[] }`; `export const CITY_FIELD_CODES = ["CITY_UNREACHABLE","CITY_ARCHIVED","CITY_READ_FAILED","CITY_OUT_OF_SCOPE"] as const`
  - `api.ts`:
    - `export async function rest<T>(method: "GET"|"POST"|"DELETE", path: string, body?: object): Promise<T>`
    - `export async function gql<TResult, TVars>(document: TypedDocumentNode<TResult, TVars>, variables?: TVars): Promise<{ data: TResult | null; fieldErrors: GraphQLRefusal[] }>` — lança `GraphQLRefusal` quando a resposta não tem `data` (recusa de análise) e devolve os erros de campo junto dos dados quando há `data` parcial
    - `export function onAuthRequired(listener: () => void): () => void` — quem registra é o shell (Task 4)
  - `graphql()` de `src/gql` para declarar documentos tipados

- [ ] **Step 1: Escrever o spec que falha**

`src/lib/api.test.ts` (simule `fetch` com `vi.stubGlobal`; sem constantes globais fora de `describe`):

```ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { gql, onAuthRequired, rest } from "./api";
import { AuthRequired, GraphQLRefusal, InvalidCredentials, NetworkError, RateLimited, RequestRejected } from "./errors";
import { graphql } from "../gql";

describe("api client", () => {
  const fetchMock = vi.fn();
  const meDoc = graphql(`query Me { me { id emailAddress } }`);

  function reply(status: number, body: unknown) {
    fetchMock.mockResolvedValueOnce(new Response(JSON.stringify(body), {
      status, headers: { "Content-Type": "application/json" }
    }));
  }

  beforeEach(() => { fetchMock.mockReset(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  it("sends the maintenance header and credentials on every call, REST and GraphQL", async () => {
    reply(200, { id: "1" });
    reply(200, { data: { me: { id: "1", emailAddress: "a@b" } } });

    await rest("GET", "/session");
    await gql(meDoc);

    for (const [ , init ] of fetchMock.mock.calls) {
      expect(new Headers(init.headers).get("X-Rota-Maintenance")).toBe("1");
      expect(init.credentials).toBe("include");
    }
  });

  it("never puts the body in the URL", async () => {
    reply(200, {});
    await rest("POST", "/invitations/enroll", { token: "tok-secreto" });

    const [ url, init ] = fetchMock.mock.calls[0];
    expect(String(url)).toBe("/invitations/enroll");
    expect(String(url)).not.toContain("tok-secreto");
    expect(JSON.parse(init.body)).toEqual({ token: "tok-secreto" });
  });

  it("turns 401 into AuthRequired and tells the listener", async () => {
    const listener = vi.fn();
    const off = onAuthRequired(listener);
    reply(401, { error: "unauthorized" });

    await expect(gql(meDoc)).rejects.toBeInstanceOf(AuthRequired);
    expect(listener).toHaveBeenCalledOnce();
    off();
  });

  it("turns invalid_credentials and invalid_session into InvalidCredentials, not AuthRequired", async () => {
    const listener = vi.fn();
    const off = onAuthRequired(listener);
    reply(401, { error: "invalid_credentials" });
    reply(401, { error: "invalid_session" });

    await expect(rest("POST", "/session", {})).rejects.toBeInstanceOf(InvalidCredentials);
    await expect(rest("POST", "/session/challenge", {})).rejects.toBeInstanceOf(InvalidCredentials);
    expect(listener).not.toHaveBeenCalled();
    off();
  });

  it("turns 429 into RateLimited and other REST errors into RequestRejected with the code", async () => {
    reply(429, { error: "too_many_requests" });
    reply(422, { error: "weak_password" });

    await expect(rest("POST", "/session", {})).rejects.toBeInstanceOf(RateLimited);
    await expect(rest("POST", "/invitations/accept", {})).rejects.toMatchObject({ code: "weak_password" });
  });

  it("turns a network failure into NetworkError", async () => {
    fetchMock.mockRejectedValueOnce(new TypeError("Failed to fetch"));
    await expect(rest("GET", "/session")).rejects.toBeInstanceOf(NetworkError);
  });

  it("throws a refusal when GraphQL answers no data, with its code", async () => {
    reply(200, { data: null, errors: [ { message: "teto", extensions: { code: "CITY_BUDGET_EXCEEDED" } } ] });
    await expect(gql(meDoc)).rejects.toMatchObject({ code: "CITY_BUDGET_EXCEEDED" });
  });

  it("returns partial data with field errors instead of throwing", async () => {
    reply(200, {
      data: { me: { id: "1", emailAddress: "a@b" } },
      errors: [ { message: "fora", path: [ "city", "profile" ], extensions: { code: "CITY_UNREACHABLE" } } ]
    });

    const result = await gql(meDoc);

    expect(result.data?.me.id).toBe("1");
    expect(result.fieldErrors).toHaveLength(1);
    expect(result.fieldErrors[0]).toBeInstanceOf(GraphQLRefusal);
    expect(result.fieldErrors[0]).toMatchObject({ code: "CITY_UNREACHABLE", path: [ "city", "profile" ] });
  });

  it("never exposes RequestRejected for a 401", async () => {
    reply(401, { error: "unauthorized" });
    await expect(rest("GET", "/session")).rejects.not.toBeInstanceOf(RequestRejected);
  });
});
```

- [ ] **Step 2: Extrair o schema e configurar o codegen**

`scripts/pull-schema.sh`:

```sh
#!/bin/sh
# Extrai o SDL da API de manutenção do próprio api (spec §4, Decisão 3 do
# plano): o endpoint GraphQL exige credencial até para introspecção, e o
# dump do schema é o mesmo SDL sem sessão. Roda a partir de apps/maintenance,
# com o stack de dev de pé. Quando o Plano 6 publicar o SDL em contracts,
# este script passa a copiar de lá.
set -eu
ROOT="$(cd "$(dirname "$0")/../../.." && pwd)"
cd "$ROOT"
docker compose exec -T api bin/rails runner 'puts Maintenance::Schema.to_definition' > apps/maintenance/schema.graphql
echo "schema.graphql atualizado ($(wc -l < apps/maintenance/schema.graphql) linhas)"
```

`codegen.ts`:

```ts
import type { CodegenConfig } from "@graphql-codegen/cli";

const config: CodegenConfig = {
  schema: "schema.graphql",
  documents: [ "src/**/*.{ts,tsx}", "!src/gql/**/*" ],
  generates: {
    "src/gql/": {
      preset: "client",
      config: { enumsAsTypes: true, scalars: { ISO8601DateTime: "string" } },
      presetConfig: { fragmentMasking: false }
    }
  }
};

export default config;
```

Rode `npm run schema:pull` (requer a Task 1 aplicada no api, para `maintainers` aparecer) e `npm run codegen`. Commite `schema.graphql` e `src/gql/`.

- [ ] **Step 3: Implementar o cliente**

`src/lib/errors.ts`:

```ts
// Erros que as telas tratam (spec §5). Nenhuma tela lê status HTTP: o
// cliente traduz aqui, num lugar só.
export class AuthRequired extends Error { constructor() { super("sessão expirada"); } }
export class InvalidCredentials extends Error { constructor() { super("e-mail, senha ou código inválidos"); } }
export class RateLimited extends Error { constructor() { super("muitas tentativas, aguarde"); } }
export class NetworkError extends Error { constructor() { super("sem conexão com a API"); } }

export class RequestRejected extends Error {
  constructor(public readonly code: string, public readonly status: number) { super(code); }
}

export class GraphQLRefusal extends Error {
  constructor(message: string, public readonly code: string, public readonly path?: (string | number)[]) {
    super(message);
  }
}

export const CITY_FIELD_CODES = [ "CITY_UNREACHABLE", "CITY_ARCHIVED", "CITY_READ_FAILED", "CITY_OUT_OF_SCOPE" ] as const;
```

`src/lib/api.ts`:

```ts
import { print } from "graphql";
import type { TypedDocumentNode } from "@graphql-typed-document-node/core";
import { apiBase } from "../env";
import { AuthRequired, GraphQLRefusal, InvalidCredentials, NetworkError, RateLimited, RequestRejected } from "./errors";

// O ÚNICO lugar que fala HTTP com a API de manutenção (spec §5).
//
// Toda chamada leva X-Rota-Maintenance: 1 — a API falha fechado sem ele — e
// credentials: "include", porque em staging a API é outro host do mesmo site.
// Nada vai na URL além do caminho: token, senha e código só no corpo.
const LOGIN_FAILURES = new Set([ "invalid_credentials", "invalid_session" ]);
const listeners = new Set<() => void>();

export function onAuthRequired(listener: () => void): () => void {
  listeners.add(listener);
  return () => listeners.delete(listener);
}

async function send(method: string, path: string, body?: object): Promise<Response> {
  try {
    return await fetch(`${apiBase()}${path}`, {
      method,
      credentials: "include",
      headers: { "Content-Type": "application/json", "Accept": "application/json", "X-Rota-Maintenance": "1" },
      body: body === undefined ? undefined : JSON.stringify(body)
    });
  } catch {
    throw new NetworkError();
  }
}

async function failure(response: Response): Promise<Error> {
  const payload = await response.json().catch(() => ({})) as { error?: string };
  const code = payload.error ?? `http_${response.status}`;

  if (response.status === 401 && LOGIN_FAILURES.has(code)) return new InvalidCredentials();
  if (response.status === 401) {
    listeners.forEach((listener) => listener());
    return new AuthRequired();
  }
  if (response.status === 429) return new RateLimited();
  return new RequestRejected(code, response.status);
}

export async function rest<T>(method: "GET" | "POST" | "DELETE", path: string, body?: object): Promise<T> {
  const response = await send(method, path, body);
  if (!response.ok) throw await failure(response);
  if (response.status === 204) return undefined as T;
  return await response.json().catch(() => undefined) as T;
}

type GraphQLError = { message: string; path?: (string | number)[]; extensions?: { code?: string } };

export async function gql<TResult, TVars>(
  document: TypedDocumentNode<TResult, TVars>,
  variables?: TVars
): Promise<{ data: TResult | null; fieldErrors: GraphQLRefusal[] }> {
  const response = await send("POST", "/graphql", { query: print(document), variables: variables ?? {} });
  if (!response.ok) throw await failure(response);

  const payload = await response.json() as { data?: TResult | null; errors?: GraphQLError[] };
  const errors = (payload.errors ?? []).map(
    (e) => new GraphQLRefusal(e.message, e.extensions?.code ?? "UNKNOWN", e.path)
  );

  if (payload.data == null) throw errors[0] ?? new GraphQLRefusal("resposta vazia", "EMPTY");
  return { data: payload.data, fieldErrors: errors };
}
```

**Confira** de onde o `client-preset` exporta `TypedDocumentNode` na versão instalada (`@graphql-typed-document-node/core` costuma vir como dependência dele); se não estiver resolvível, importe o tipo de `src/gql/graphql` ou acrescente a dependência e diga no relatório. **Confira** também o status que a API dá para GraphQL sem credencial (esperado 401 com `{ error }`) no `Maintenance::GraphqlController`, e ajuste o teste de 401 ao corpo real.

- [ ] **Step 4: CI com a checagem do codegen**

Acrescente `npm run codegen:check` ao `ci.yml` (se não entrou na Task 2).

- [ ] **Step 5: Rodar e commit**

Run: `npm run typecheck && npm test && npm run codegen:check && npm run build`

```bash
/opt/homebrew/bin/git add -A
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: add the maintenance API client and generated types

One client sends the maintenance header and credentials on every call,
keeps secrets out of URLs, and turns every failure into a typed error.
Field errors of a city come back with the partial data instead of
failing the whole screen. Types come from the api's own schema dump.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 4: Sessão e tela de entrar

**Files:**
- Create: `src/lib/session.tsx`, `src/screens/Login.tsx`, `src/components/Field.tsx`, `src/components/Button.tsx`, `src/components/ErrorState.tsx`
- Modify: `src/main.tsx`, `src/App.tsx`
- Test: `src/lib/session.test.tsx`, `src/screens/Login.test.tsx`

**Interfaces:**
- Consumes: `rest`, `onAuthRequired`, erros (Task 3).
- Produces:
  - `session.tsx`: `type Me = { id: string; emailAddress: string; expiresAt: string }`; `<SessionProvider>`; `useSession(): { state: "loading" | "signedOut" | "signedIn"; me: Me | null; notice: string | null; signIn(me: Me): void; signOut(): Promise<void> }`. Ao montar, `GET /session`; em `AuthRequired` (via `onAuthRequired`), limpa o cache do React Query e vai para `signedOut` com `notice = "sessão expirada"`.
  - `<Login onSignedIn={(me) => void} />` em dois passos.

- [ ] **Step 1: Escrever os specs que falham**

`src/lib/session.test.tsx` — com `fetch` simulado:
- `GET /session` 200 → `signedIn` com `me` preenchido (`email_address` → `emailAddress`, `expires_at` → `expiresAt`); 401 → `signedOut` **sem** `notice` (abrir o app deslogado não é "sessão expirada").
- depois de `signedIn`, um `AuthRequired` disparado por qualquer consulta leva a `signedOut` com `notice === "sessão expirada"` e **esvazia o cache do React Query** (semeie uma consulta antes e afirme que `queryClient.getQueryCache().getAll()` fica vazio).
- `signOut()` chama `DELETE /session`, esvazia o cache e vai para `signedOut` mesmo se o `DELETE` falhar com rede.

`src/screens/Login.test.tsx` — com `@testing-library/user-event`:
- passo 1: e-mail e senha → `POST /session` com `{ email_address, password }`; resposta com `session_id` mostra o campo de código; o `session_id` **não** aparece no DOM nem em `localStorage`.
- passo 2: código → `POST /session/challenge` com `{ session_id, code }`; sucesso chama `onSignedIn` com o `Me`.
- `InvalidCredentials` em qualquer passo mostra exatamente "e-mail, senha ou código inválidos" e, no passo 2, volta ao passo 1 (a sessão pendente não é reaproveitada).
- `RateLimited` mostra "muitas tentativas, aguarde".
- `localStorage` continua vazio ao fim de todo exemplo.

- [ ] **Step 2: Rodar e confirmar que falham**

- [ ] **Step 3: Implementar**

`src/lib/session.tsx`:

```tsx
import { createContext, useCallback, useContext, useEffect, useState, type ReactNode } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { onAuthRequired, rest } from "./api";
import { AuthRequired } from "./errors";

// Sessão do mantenedor (spec §5). O cookie é da API; aqui só se sabe SE há
// sessão. Qualquer 401 no meio do uso (30 min parado, 8 h no total) apaga
// o cache inteiro: nada do que estava na tela sobrevive à expiração.
export type Me = { id: string; emailAddress: string; expiresAt: string };
type SessionPayload = { id: string; email_address: string; expires_at: string };
type State = "loading" | "signedOut" | "signedIn";

type SessionContext = {
  state: State;
  me: Me | null;
  notice: string | null;
  signIn(me: Me): void;
  signOut(): Promise<void>;
};

const Context = createContext<SessionContext | null>(null);

export function toMe(payload: SessionPayload): Me {
  return { id: payload.id, emailAddress: payload.email_address, expiresAt: payload.expires_at };
}

export function SessionProvider({ children }: { children: ReactNode }) {
  const queryClient = useQueryClient();
  const [ state, setState ] = useState<State>("loading");
  const [ me, setMe ] = useState<Me | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);

  const drop = useCallback((message: string | null) => {
    queryClient.clear();
    setMe(null);
    setNotice(message);
    setState("signedOut");
  }, [ queryClient ]);

  useEffect(() => {
    let alive = true;
    rest<SessionPayload>("GET", "/session")
      .then((payload) => { if (alive) { setMe(toMe(payload)); setState("signedIn"); } })
      .catch(() => { if (alive) drop(null); });
    return () => { alive = false; };
  }, [ drop ]);

  useEffect(() => onAuthRequired(() => {
    setState((current) => { if (current === "signedIn") drop("sessão expirada"); return current; });
  }), [ drop ]);

  const signIn = useCallback((next: Me) => { setNotice(null); setMe(next); setState("signedIn"); }, []);
  const signOut = useCallback(async () => {
    try { await rest("DELETE", "/session"); } catch (error) { if (!(error instanceof AuthRequired)) { /* segue */ } }
    drop(null);
  }, [ drop ]);

  return <Context.Provider value={{ state, me, notice, signIn, signOut }}>{children}</Context.Provider>;
}

export function useSession(): SessionContext {
  const value = useContext(Context);
  if (!value) throw new Error("useSession fora do SessionProvider");
  return value;
}
```

**Atenção:** chamar `drop` dentro do atualizador de `setState` é efeito colateral dentro de setter — refatore para ler o estado atual por `ref` se o `StrictMode` expuser o problema, e diga no relatório. O comportamento exigido é o dos testes.

`src/screens/Login.tsx` — dois passos com `useState` local (`step: "credentials" | "code"`, `sessionId: string | null` em memória), `Field` e `Button`, mensagem de erro com `ErrorState`. Em `InvalidCredentials` no passo 2, zera `sessionId` e volta ao passo 1. Texto de ajuda no passo 2: "Código de 6 dígitos do seu app autenticador."

`src/components/Field.tsx` (rótulo + input controlado, `autoComplete` repassado), `Button.tsx` (`type`, `disabled`, `busy`), `ErrorState.tsx` (`role="alert"`, mensagem).

`src/main.tsx`: `QueryClientProvider` (`refetchOnWindowFocus: false`, `retry: (count, error) => !(error instanceof AuthRequired) && count < 1`) → `SessionProvider` → `App`. `src/App.tsx`: `EnvBanner` sempre; `state === "loading"` → "carregando…"; `signedOut` → `Login` (com `notice`, se houver); `signedIn` → por ora "Olá, <e-mail>" e "sair" (a Task 6 troca pelo shell).

- [ ] **Step 4: Rodar e commit**

Run: `npm run typecheck && npm test && npm run build`

```bash
/opt/homebrew/bin/git add -A
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: sign a maintainer in with password and TOTP

Two steps, the pending session id only in memory, one message for every
failure so the screen reveals nothing about the account. An expired
session empties the whole cache and returns to the sign-in screen.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 5: Convite e matrícula

**Files:**
- Create: `src/lib/invitation.ts`, `src/screens/Invitation.tsx`
- Modify: `src/App.tsx`
- Test: `src/lib/invitation.test.ts`, `src/screens/Invitation.test.tsx`

**Interfaces:**
- Produces:
  - `readInvitationToken(location: Location, history: History): string | null` — se `location.pathname` é `/invitations` e há fragmento, devolve o token (fragmento sem `#`, decodificado) e **limpa a URL** com `history.replaceState(null, "", "/")`; senão `null`
  - `<Invitation token={string} onDone={() => void} />`

- [ ] **Step 1: Escrever os specs que falham**

`src/lib/invitation.test.ts`:
- `/invitations#abc` → `"abc"` e `replaceState` chamado com `"/"` (a URL não guarda mais o token).
- `/invitations` sem fragmento → `null`, sem `replaceState`.
- `/` com fragmento → `null` (o fragmento só é convite nesse caminho).
- Token com caracteres codificados é decodificado.

`src/screens/Invitation.test.tsx` (fetch simulado; `qrcode` pode ser simulado com `vi.mock("qrcode")`):
- ao montar, `POST /invitations/enroll` com `{ token }` no **corpo**; mostra o e-mail, a chave em texto e uma imagem do QR (`alt="QR do autenticador"`), gerada a partir do `otpauth_uri` **sem** chamada de rede a outro host (afirme que o `fetch` só foi chamado para `/invitations/enroll` e `/invitations/accept`).
- senha, confirmação e código → `POST /invitations/accept` com `{ token, password, code }`; sucesso chama `onDone`.
- senha e confirmação diferentes: não chama a API e mostra "as senhas não coincidem".
- senha com menos de 12 caracteres: não chama a API e mostra "a senha precisa de pelo menos 12 caracteres".
- `RequestRejected("weak_password")` → a mesma mensagem de 12 caracteres; `RequestRejected("invalid_code")` → "código inválido — use o código atual do autenticador"; `enroll` com 404 (`RequestRejected("http_404")` ou código equivalente) → "convite inválido, usado ou expirado".
- o token nunca aparece no DOM.

- [ ] **Step 2: Rodar e confirmar que falham**

- [ ] **Step 3: Implementar**

`src/lib/invitation.ts`:

```ts
// O link do convite é /invitations#<token> (rake maintainer:invite). O
// fragmento nunca vai ao servidor; ainda assim ele fica no histórico do
// navegador — por isso é lido UMA vez e a URL é limpa na hora (spec §5).
export function readInvitationToken(location: Location, history: History): string | null {
  if (location.pathname !== "/invitations") return null;
  const raw = location.hash.replace(/^#/, "");
  if (raw === "") return null;

  history.replaceState(null, "", "/");
  return decodeURIComponent(raw);
}
```

`src/screens/Invitation.tsx` — ao montar, `rest("POST", "/invitations/enroll", { token })`; gera o QR com `QRCode.toDataURL(otpauth_uri)` (biblioteca local); mostra e-mail, QR, a chave (`secret`) em `<code>`, e o formulário de senha, confirmação e código. Validação local de senha antes de chamar a API. Sucesso: "Matrícula concluída. Entre com sua senha e o código do autenticador." e `onDone()`.

Em `App.tsx`: calcule `const [ invitationToken, setInvitationToken ] = useState(() => readInvitationToken(window.location, window.history))` **uma vez**; se houver token, renderize `Invitation` (independente do estado da sessão) até `onDone`, que zera o token e leva à tela de entrar.

- [ ] **Step 4: Rodar e commit**

```bash
/opt/homebrew/bin/git add -A
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: enroll a maintainer from an invitation link

The token is read once from the fragment and the URL is cleared at once,
the TOTP QR code is drawn in the browser so the secret never reaches
another service, and accepting does not open a session: the first
sign-in already asks for the authenticator.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 6: Shell, catálogo e detalhe de cidade

**Files:**
- Create: `src/screens/Shell.tsx`, `src/screens/Cities.tsx`, `src/screens/CityDetail.tsx`, `src/components/Panel.tsx`, `src/components/DataTable.tsx`, `src/components/EmptyState.tsx`, `src/components/Tag.tsx`
- Modify: `src/App.tsx`
- Test: `src/screens/Cities.test.tsx`, `src/screens/CityDetail.test.tsx`, `src/screens/Shell.test.tsx`

**Interfaces:**
- Consumes: `gql`, `graphql()`, `GraphQLRefusal`, `useSession`.
- Produces:
  - `type Screen = "cities" | "maintainers" | "tokens" | "audit"`
  - `<Shell />` — cabeçalho (e-mail, "sair", navegação), estado `screen` e `openCity: string | null`
  - `<Cities onOpen={(slug) => void} />`, `<CityDetail slug={string} onBack={() => void} />`
  - `DataTable<T>({ columns: { key, label, render? }[], rows: T[], onRowClick? })`

- [ ] **Step 1: Escrever os specs que falham**

`src/screens/Cities.test.tsx`:
- lista as cidades da consulta `cities` (slug, nome, UF, status, "schema atrasado" como `Tag`); filtro de status reconsulta com `status` na variável; clicar numa linha chama `onOpen(slug)`.
- lista vazia → `EmptyState` "nenhuma cidade".
- erro de rede → `ErrorState` com "sem conexão com a API".

`src/screens/CityDetail.test.tsx`:
- o topo (plataforma + canal) vem de uma consulta; **cada aba** (Perfil, Protocolos, Destinatários, Contas, Contagens, Operação) faz a **sua** consulta **só quando aberta** — afirme pelas chamadas de `fetch` que abrir o detalhe não consulta `profile`/`accounts`/etc., e que abrir "Contas" consulta só `accounts`.
- uma aba cuja resposta traz erro de campo `CITY_UNREACHABLE` mostra o código e a mensagem **naquela aba**; trocar para outra aba que responde normalmente mostra os dados. O topo continua visível.
- `CITY_ARCHIVED` e `CITY_READ_FAILED` aparecem da mesma forma.
- "atualizar" numa aba refaz só a consulta daquela aba.

`src/screens/Shell.test.tsx`:
- mostra o e-mail do `me`, a navegação com as quatro telas e "sair"; "sair" chama `signOut`.
- abrir uma cidade e voltar preserva a tela Cidades.

- [ ] **Step 2: Rodar e confirmar que falham**

- [ ] **Step 3: Implementar**

**Documentos** (declarados com `graphql()`; confira os campos contra `schema.graphql` — o codegen falha se um campo não existir):

```ts
const CitiesQuery = graphql(`
  query Cities($status: CityStatus) {
    cities(status: $status) { slug name uf status schemaVersion schemaBehind createdAt }
  }
`);

const CityHeaderQuery = graphql(`
  query CityHeader($slug: String!) {
    city(slug: $slug) {
      slug name uf status schemaVersion schemaBehind createdAt
      channel { phoneNumberId wabaId displayPhoneNumber active }
    }
  }
`);

const CityProfileQuery = graphql(`
  query CityProfile($slug: String!) { city(slug: $slug) { slug consentTermVersion profile { name uf ibgeCode } } }
`);
const CityProtocolsQuery = graphql(`
  query CityProtocols($slug: String!) { city(slug: $slug) { slug protocols { name version status } } }
`);
const CityRecipientsQuery = graphql(`
  query CityRecipients($slug: String!) {
    city(slug: $slug) { slug alertRecipients { channel destination escalationOrder } }
  }
`);
const CityAccountsQuery = graphql(`
  query CityAccounts($slug: String!) { city(slug: $slug) { slug accounts { login roles active mfaEnrolled } } }
`);
const CityCountsQuery = graphql(`
  query CityCounts($slug: String!) {
    city(slug: $slug) { slug counts { users conversations triages inboundMessages reportSnapshots consents } }
  }
`);
const CityOperationsQuery = graphql(`
  query CityOperations($slug: String!) {
    city(slug: $slug) {
      slug
      operations {
        domainEvents { name occurredAt publishedAt }
        reportSnapshots { id createdAt expiresAt }
        dashboardMetrics { dimension period label value computedAt }
        failedJobs { className failedAt errorClass }
      }
    }
  }
`);
```

**Cada aba** usa `useQuery({ queryKey: ["city", slug, tab], queryFn: () => gql(Doc, { slug }), enabled: tab === active })`. O erro de campo da aba vem em `fieldErrors` (não em `error`): a aba mostra `ErrorState` com `code — message` do primeiro `GraphQLRefusal` cujo `path` começa por `["city", <campo da aba>]`. O botão "atualizar" chama `refetch()` daquela consulta.

`Shell.tsx`: cabeçalho com o e-mail (`useSession().me`), navegação por botões (`aria-current` na tela ativa), "sair"; `screen` e `openCity` em `useState`; `Cities` → `CityDetail` quando `openCity` não é nulo. As telas Mantenedores, Tokens e Auditoria entram nas Tasks 7–9 — até lá, `EmptyState` "em breve".

`App.tsx`: `signedIn` → `<Shell />`.

- [ ] **Step 4: Rodar e commit**

```bash
/opt/homebrew/bin/git add -A
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: read the city catalogue and one city at a time

The catalogue opens no city connection. A city's detail loads its header
first and each tab only when it is opened, one connection per tab, and a
city that fails to answer shows its code in that tab while the rest of
the page keeps working.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 7: Mantenedores

**Files:**
- Create: `src/screens/Maintainers.tsx`, `src/components/ConfirmButton.tsx`
- Modify: `src/screens/Shell.tsx`
- Test: `src/screens/Maintainers.test.tsx`

**Interfaces:**
- Consumes: `maintainers` (Task 1), `inviteMaintainer`, `deactivateMaintainer`.
- Produces: `<Maintainers />`; `ConfirmButton({ label, confirmLabel, onConfirm })` — dois cliques, o segundo confirma.

- [ ] **Step 1: Escrever o spec que falha**

`src/screens/Maintainers.test.tsx`:
- lista e-mail, "ativo"/"desativado" e "matriculado"/"convite pendente".
- convidar: e-mail + código → `inviteMaintainer(emailAddress, code)`; `ok` mostra "Convite registrado. O link é entregue pelo `rake maintainer:invite` no servidor." e reconsulta a lista; `errors` aparecem ao lado do campo pelo `path` (`emailAddress` ou `code`).
- o campo de código mostra o aviso "Se acabou de entrar, espere o próximo código." e, em erro no `code`, "Tentativas erradas contam para o bloqueio da conta."
- desativar pede confirmação (`ConfirmButton`); `errors` da API (a si mesmo, o último ativo) aparecem na linha.
- o botão "desativar" não aparece na própria linha do mantenedor da sessão.

- [ ] **Step 2: Rodar e confirmar que falha**

- [ ] **Step 3: Implementar**

Documentos:

```ts
const MaintainersQuery = graphql(`query Maintainers { maintainers { id emailAddress active enrolled createdAt } }`);
const InviteMutation = graphql(`
  mutation InviteMaintainer($emailAddress: String!, $code: String!) {
    inviteMaintainer(emailAddress: $emailAddress, code: $code) { ok errors { path message } }
  }
`);
const DeactivateMutation = graphql(`
  mutation DeactivateMaintainer($id: ID!) { deactivateMaintainer(id: $id) { ok errors { path message } } }
`);
```

Mutations via `useMutation`; em `ok`, `queryClient.invalidateQueries({ queryKey: ["maintainers"] })` e limpa o campo de código.

- [ ] **Step 4: Rodar e commit**

```bash
/opt/homebrew/bin/git add -A
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: manage maintainers from the maintenance frontend

List every maintainer with state and enrollment, invite with the TOTP of
the moment, and deactivate after a confirmation. The screen says the
invitation link is delivered by the server task, since there is no mailer.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 8: Tokens de serviço

**Files:**
- Create: `src/screens/Tokens.tsx`
- Modify: `src/screens/Shell.tsx`
- Test: `src/screens/Tokens.test.tsx`

**Interfaces:**
- Consumes: `maintenanceTokens`, `createMaintenanceToken`, `revokeMaintenanceToken`, `cities` (para escolher o escopo).
- Produces: `<Tokens />`.

- [ ] **Step 1: Escrever o spec que falha**

`src/screens/Tokens.test.tsx`:
- lista nome, acesso, cidades ("todas" quando vazio), validade, último uso e "revogado".
- criar: nome, acesso (`read`/`read_write`), cidades (seleção múltipla a partir de `cities`; nenhuma marcada mostra o aviso "sem cidades marcadas, o token alcança todas"), validade em dias (1–90, padrão 30, convertida para `expiresAt` ISO), código → `createMaintenanceToken`.
- validade fora de 1–90 não chama a API.
- **o segredo:** a resposta com `secretOnce` abre um painel com o segredo, "copiar" e "Este segredo não será mostrado de novo."; ao clicar "fechei", o painel some, **o segredo não está mais no DOM** e **não está em nenhuma entrada do cache do React Query** (percorra `queryClient.getQueryCache().getAll()` e `getMutationCache().getAll()` e afirme que nenhum `state.data` serializado contém o segredo).
- revogar pede confirmação e reconsulta a lista.
- `errors` da API aparecem pelo `path`.

- [ ] **Step 2: Rodar e confirmar que falha**

- [ ] **Step 3: Implementar**

Documentos:

```ts
const TokensQuery = graphql(`
  query MaintenanceTokens {
    maintenanceTokens { id name access citySlugs expiresAt revokedAt lastUsedAt }
  }
`);
const CreateTokenMutation = graphql(`
  mutation CreateMaintenanceToken($name: String!, $access: String!, $citySlugs: [String!], $expiresAt: ISO8601DateTime!, $code: String!) {
    createMaintenanceToken(name: $name, access: $access, citySlugs: $citySlugs, expiresAt: $expiresAt, code: $code) {
      ok secretOnce errors { path message }
    }
  }
`);
const RevokeTokenMutation = graphql(`
  mutation RevokeMaintenanceToken($id: ID!) { revokeMaintenanceToken(id: $id) { ok errors { path message } } }
`);
```

**O segredo não passa pelo cache:** a criação usa `useMutation` com `gcTime: 0`; no `onSuccess`, copie `secretOnce` para um `useState` local do painel e **chame `mutation.reset()` na hora**, para que o resultado da mutation não guarde o segredo. Ao fechar o painel, zere o estado local. A lista é invalidada (`["maintenanceTokens"]`), e a consulta de lista nunca pede `secretOnce`. **Confira** no teste que, depois de `reset()`, o `MutationCache` não retém o dado; se retiver, remova a mutation do cache explicitamente e diga no relatório.

- [ ] **Step 4: Rodar e commit**

```bash
/opt/homebrew/bin/git add -A
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: manage service tokens from the maintenance frontend

Create a token scoped by access, cities and expiry with the TOTP of the
moment, show its secret exactly once, and drop it from memory and from
the query cache when the panel closes. Revoke after a confirmation.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 9: Auditoria

**Files:**
- Create: `src/screens/Audit.tsx`
- Modify: `src/screens/Shell.tsx`
- Test: `src/screens/Audit.test.tsx`

**Interfaces:**
- Consumes: `auditEvents(since, until, maintainerId, module, outcome, limit)`.
- Produces: `<Audit />`.

- [ ] **Step 1: Escrever o spec que falha**

`src/screens/Audit.test.tsx`:
- lista quando, evento, módulo, resultado, quem (`login`, ou "—" quando nulo) e o `correlationId`.
- filtros: período (desde/até), módulo, resultado (`attempted`, `ok`, `rejected`, `error`) e mantenedor (lista vinda de `maintainers`) viram variáveis da consulta; "limite" padrão 100.
- eventos com o mesmo `correlationId` ficam visualmente agrupados (tentativa e resultado lado a lado) — afirme pela ordem ou pelo atributo de grupo.
- somente leitura: nenhum botão de ação.

- [ ] **Step 2: Rodar e confirmar que falha**

- [ ] **Step 3: Implementar**

```ts
const AuditQuery = graphql(`
  query AuditEvents($since: ISO8601DateTime, $until: ISO8601DateTime, $maintainerId: ID, $module: String, $outcome: String, $limit: Int) {
    auditEvents(since: $since, until: $until, maintainerId: $maintainerId, module: $module, outcome: $outcome, limit: $limit) {
      name module outcome occurredAt maintainerId login correlationId
    }
  }
`);
```

- [ ] **Step 4: Rodar e commit**

```bash
/opt/homebrew/bin/git add -A
/opt/homebrew/bin/git commit -F - <<'EOF'
feat: read the maintenance audit trail

Filter by period, module, outcome and maintainer, and keep an attempt
next to its outcome by correlation id. Read only.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

### Task 10: e2e com Playwright contra o stack de dev

**Files:**
- Create: `playwright.config.ts`, `e2e/support.ts`, `e2e/maintenance.spec.ts`
- Modify: `README.md`

**Interfaces:**
- Consumes: todo o app; `rake maintainer:invite` no api; `otplib`.

- [ ] **Step 1: Configuração**

`playwright.config.ts`: `testDir: "e2e"`, `use.baseURL: process.env.E2E_BASE_URL ?? "http://maintenance.localhost:5177"`, um projeto `chromium`, `workers: 1`, `fullyParallel: false`, `timeout: 180_000`, sem `webServer` (o stack de dev já está de pé). **Confira** que o Chromium do Playwright resolve `maintenance.localhost` para `127.0.0.1` (o Chrome resolve `*.localhost` nativamente); se não resolver, use `launchOptions.args: ["--host-resolver-rules=MAP *.localhost 127.0.0.1"]` e diga no relatório.

`e2e/support.ts`:

```ts
import { execFileSync } from "node:child_process";
import { resolve } from "node:path";
import { authenticator } from "otplib";

// O teste cria o próprio mantenedor pelo caminho real (rake, como um
// operador faria) e calcula o TOTP da chave que a própria tela mostra.
// Cada código é consumido uma vez pela API: entre dois usos, espera-se o
// próximo passo de 30 s (Decisão 7 do plano).
const ROOT = process.env.ROTA_ROOT ?? resolve(__dirname, "../../..");

export function inviteMaintainer(email: string): string {
  const out = execFileSync("docker", [ "compose", "exec", "-T", "api", "bin/rails", `maintainer:invite[${email}]` ],
                           { cwd: ROOT, encoding: "utf8" });
  const link = out.split("\n").map((l) => l.match(/(\S+\/invitations#\S+)/)?.[1]).find(Boolean);
  if (!link) throw new Error("o rake não imprimiu o link do convite");
  return link;
}

let lastStep = -1;

export async function freshCode(secret: string): Promise<string> {
  let step = Math.floor(Date.now() / 30_000);
  while (step <= lastStep) {
    await new Promise((r) => setTimeout(r, 1_000));
    step = Math.floor(Date.now() / 30_000);
  }
  lastStep = step;
  return authenticator.generate(secret);
}
```

- [ ] **Step 2: Os fluxos**

`e2e/maintenance.spec.ts` — um `test.describe.serial`, com um e-mail único por execução (`e2e-${Date.now()}@rotasaude.app`):

1. **Convite e matrícula:** `inviteMaintainer` → abrir o link → a URL passa a ser `/` (sem o token) → ler a chave em texto da tela → senha de 16 caracteres, confirmação, `freshCode` → "Matrícula concluída".
2. **Entrar:** e-mail, senha → `freshCode` → aparece o e-mail no cabeçalho e a faixa `DEVELOPMENT`.
3. **Cidades:** a tabela tem `curitiba`; abrir `curitiba` mostra o topo; abrir "Contagens" mostra números.
4. **Token:** criar um token `read`, 1 dia, sem cidades, com `freshCode` → o painel mostra o segredo; fechar; **recarregar a página** e, depois de entrar de novo se preciso, afirmar que o segredo não aparece em lugar nenhum (`page.content()` não contém o valor) e que o token está na lista; revogar e ver "revogado".
5. **Sair:** "sair" → tela de entrar; `page.request.post("/graphql", …)` com o header obrigatório responde 401.

- [ ] **Step 3: Rodar**

Com o stack de dev de pé (`api`, `worker`, `maintenance`): `npm run e2e`. Registre a saída no relatório. O teste deixa um mantenedor e linhas de auditoria no banco de dev — é o custo aceito (spec §7); **não** tente apagá-los (a auditoria recusa `DELETE` por trigger).

- [ ] **Step 4: README e commit**

Documente no README: pré-requisitos (stack de dev de pé, `npx playwright install chromium` uma vez), o comando, a duração (~2 min por causa da espera do TOTP) e o custo no banco de dev.

```bash
/opt/homebrew/bin/git add -A
/opt/homebrew/bin/git commit -F - <<'EOF'
test: cover the maintenance frontend end to end against the dev stack

Invitation, enrollment, sign-in, a city, a service token whose secret is
shown once and never again, and sign-out — through the real proxy, the
real cookie and the real one-time TOTP codes that mocks would not catch.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
```

---

## Verificação final do plano

- [ ] api: suíte completa com o worker parado, 0 falhas; `maintainers` só para sessão humana.
- [ ] `docker compose exec -T api printenv MAINTENANCE_API_ENABLED MAINTENANCE_FRONTEND_ORIGIN` → `true` e `http://maintenance.localhost:5177`.
- [ ] `apps/maintenance`: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build` verdes; `npm run e2e` verde contra o stack de dev.
- [ ] `grep -rn "localStorage" apps/maintenance/src` só em testes que afirmam que ele está vazio.
- [ ] Nenhuma URL montada no código contém token, código ou senha (`grep -rn "?token\|?code\|?password" apps/maintenance/src` vazio).
- [ ] `dashboard`, `admin`, `wpda` intocados.

## Fora deste plano

- Criar `rotasaude/maintenance` no GitHub e o primeiro push (usuário).
- Telas de escrita em cidade (Plano 5 da API, reescrito).
- Hospedagem em staging.
- SDL de `contracts` (Plano 6).
- e2e na CI.
