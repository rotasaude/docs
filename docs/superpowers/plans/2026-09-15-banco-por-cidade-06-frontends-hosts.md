# Banco por cidade — Plano 6: frontends e hosts por cidade

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** os três frontends passam a viver no host da cidade (`<slug>.<domínio>`), o console fica em `admin.*` sem visão cross-tenant, e todo link que a API gera para um cidadão ou usuário sai do host da cidade certa.

**Architecture:** a cidade já é resolvida pelo `Host` da requisição (`CityCatalog` + `CityResolution`). Hoje o proxy do Vite reescreve esse Host (`changeOrigin: true`), então nenhum subdomínio chega ao Rails. Este plano desliga essa reescrita, ensina o CORS a aceitar qualquer host de cidade do catálogo, deriva as URLs públicas (dashboard, wpda, reset de senha) do slug em vez de env vars únicas, e ajusta as telas: console lista cidades e entra numa delas por grant; dashboard consome `?grant=` e `?invite=` e inicia o gov.br da própria cidade.

**Tech Stack:** Rails 8.1 (API), Rack::Cors, Vite 5.4.21 + React 18 + TypeScript (admin, dashboard, wpda), Vitest + @testing-library/react, Docker Compose (dev), Kamal 2 (deploy).

**Spec:** `docs/superpowers/specs/2026-09-12-banco-por-cidade-design.md` (§2 resolução por host, §5 identidade e frontends, §6 chaves por cidade — a chave fica para o Plano 7)

## Global Constraints

- Resolução: `CityCatalog::RESERVED = %w[admin api auth www]`; qualquer outro primeiro label é slug de cidade. 404 se não existir, 403 se `suspended`, 503 se o schema estiver atrasado (`app/controllers/concerns/city_resolution.rb:22-34`).
- Cookie de sessão **nunca** recebe `domain:` (host-only, `config/application.rb`): não existe sessão compartilhada entre `admin.*` e a cidade, nem entre duas cidades. Entrar numa cidade é sempre por grant (`POST /session/grant`).
- `/admin/api/*` é read-only (critério §10): nenhuma rota de escrita entra nesse namespace.
- Mailer recebe string montada dentro da cidade, nunca objeto ActiveRecord (R42).
- Commits em inglês, terminando com a linha `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Comandos Ruby/Rails/rspec rodam no container, a partir da raiz do monorepo: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Nunca no host.
- Comandos de frontend rodam no container do app: `docker compose exec -T dashboard npm run test`. Se o serviço estiver parado, suba só ele: `docker compose up -d dashboard`.
- Suíte completa do api: `docker compose stop worker` antes, `docker compose start worker` depois, timeout ≥ 600000 ms, nunca duas suítes ao mesmo tempo, nunca `git stash`.
- Nada de operação destrutiva em `rota_saude_development`, `rota_saude_test`, `rota_saude_platform_*`, `rota_saude_no_city_selected`, `rota_saude_city_curitiba`, `rota_saude_city_maringa`. Nunca rodar `start.sh`.
- `docker-compose.yml`, `.env.example` e `start.sh` ficam na raiz do monorepo, **fora do git** — editar no lugar e registrar o diff no relatório.
- Nunca imprimir segredo, token de convite, grant ou telefone de cidadão.

## Decisões

1. **Host em dev sem proxy reverso novo** (decisão do usuário): `changeOrigin: false` nos três `vite.config.ts` e `server.allowedHosts` liberando `.localhost`. O Vite repassa `Host: curitiba.localhost:5175` para o Rails, que resolve a cidade. Vite instalado é 5.4.21 nos três apps (lockfiles), acima do 5.4.12 que introduziu `allowedHosts`.
2. **Uma env var de host por cidade.** `CITY_PUBLIC_BASE_TEMPLATE` (default `http://%{slug}.localhost:5175`) substitui `CITY_DASHBOARD_URL_TEMPLATE`, `PUBLIC_DASHBOARD_URL` e `WPDA_PUBLIC_BASE`. Dashboard, wpda e reset de senha derivam dela.
3. **CORS pelo catálogo**, não por lista: origem cujo host resolve cidade servível, ou é host reservado, é aceita. `ALLOWED_ORIGINS` continua valendo como lista extra (ferramentas internas).
4. **O console perde a visão cross-tenant de métricas** (spec §5). A tela "Cidades" passa a listar o catálogo (slug, status, schema) vindo de `GET /cities` no host do console, e o detalhe rico com KPIs por cidade sai. Quem quer número de uma cidade entra nela.
5. **gov.br é da cidade, não do console.** O botão sai do `Login` do admin e entra no `Login` do dashboard, chamando `POST /auth/govbr/start` (que devolve `authorize_url` já com `state` assinado). O console não tem login gov.br.
6. **`X-Municipality-Id` morre.** O escopo é o host; o header não é lido pelo backend (`Admin::Api::BaseController:14-15`).
7. **Admin ganha Vitest**, hoje sem nenhum teste, espelhando a configuração do dashboard.
8. **Wildcard de TLS não entra aqui.** O Kamal publica `admin.*`, `auth.*` e o host de cada cidade; certificado curinga e DNS são gate de go-live do Plano 7.

## File Structure

**API (`apps/api`)**
- Criar `app/services/city_public_url.rb` — base pública da cidade e os caminhos derivados (`dashboard`, `wpda`).
- Modificar `app/services/city_dashboard_url.rb` — passa a derivar de `CityPublicUrl`.
- Modificar `app/models/report_snapshot.rb:28-31` — link do cidadão pelo slug.
- Modificar `app/controllers/passwords_controller.rb:38-42` — link de reset pelo slug.
- Modificar `app/controllers/reports_controller.rb:1-8` — comentário desatualizado.
- Modificar `config/initializers/cors.rb` — origens pelo catálogo.
- Modificar `app/controllers/operators/cities_controller.rb` — `index` do catálogo.
- Modificar `config/routes.rb:18` — `resources :cities, only: %i[index create show]`.
- Criar `spec/services/city_public_url_spec.rb`, `spec/requests/cors_spec.rb`.
- Modificar `spec/requests/operators/cities_spec.rb`, `spec/requests/passwords_spec.rb`, `spec/requests/reports_spec.rb`.

**Admin (`apps/admin`)** — console de plataforma
- Modificar `vite.config.ts`, `package.json`, `.env.example`, `README.md`.
- Criar `vitest.config.ts`, `src/lib/cities.ts`, `src/lib/cities.test.ts`.
- Modificar `src/lib/api.ts` (sai `X-Municipality-Id`, entram `listCities` e `enterCity`), `src/lib/auth.tsx` (sai o header), `src/modules/Cities.tsx` (catálogo + "Entrar"), `src/modules/Login.tsx` (sai gov.br), `src/App.tsx` (sai o escopo de município).
- Apagar `src/hooks/useCityDetail.ts`.

**Dashboard (`apps/dashboard`)** — app da cidade
- Modificar `vite.config.ts` (proxy + `/authoring`), `.env.example`, `README.md`.
- Modificar `src/lib/api.ts` (`redeemGrant`, `acceptInvitation`, `startGovBr`), `src/lib/scope.ts` (sai `municipality_id`), `src/main.tsx` (`?grant=`, `?invite=`), `src/modules/Login.tsx` (botão gov.br).
- Criar `src/modules/AcceptInvitation.tsx`, `src/lib/entry.ts`, `src/lib/entry.test.ts`.

**WPDA (`apps/wpda`)**
- Modificar `vite.config.ts`, `README.md`.

**Infra (fora do git)**
- `docker-compose.yml` — `CITY_PUBLIC_BASE_TEMPLATE` no api/worker.
- `.env.example` — a mesma variável e as URLs de dev por cidade.
- `start.sh` — bloco final de URLs por cidade.

**Deploy (`apps/api/deploy`)**
- `development/deploy.yml` e `production/deploy.yml` — `CITY_PUBLIC_BASE_TEMPLATE` no lugar de `CITY_DASHBOARD_URL_TEMPLATE`, e `proxy.hosts` com console, auth e cidades.

---

### Task 1: URLs públicas derivadas do slug

**Files:**
- Create: `apps/api/app/services/city_public_url.rb`
- Create: `apps/api/spec/services/city_public_url_spec.rb`
- Modify: `apps/api/app/services/city_dashboard_url.rb`
- Modify: `apps/api/app/models/report_snapshot.rb:28-31`
- Modify: `apps/api/app/controllers/passwords_controller.rb:38-42`
- Modify: `apps/api/app/controllers/reports_controller.rb:1-8`
- Modify: `apps/api/deploy/development/deploy.yml:54`, `apps/api/deploy/production/deploy.yml:59`
- Modify: `apps/api/README.md:24`
- Test: `apps/api/spec/services/city_public_url_spec.rb`, `apps/api/spec/requests/passwords_spec.rb`

**Interfaces:**
- Produces: `CityPublicUrl.base(city) -> String` (sem barra final), `CityPublicUrl.dashboard(city) -> String`, `CityPublicUrl.wpda(city) -> String`, `CityPublicUrl::CityMissing`. Template: `ENV["CITY_PUBLIC_BASE_TEMPLATE"]`, default `http://%{slug}.localhost:5175`.
- Consumes: `City#slug`, `Current.city`.

- [ ] **Step 1: Escrever o spec do serviço**

```ruby
# apps/api/spec/services/city_public_url_spec.rb
require "rails_helper"

RSpec.describe CityPublicUrl do
  let(:city) { City.new(slug: "curitiba", name: "Curitiba", status: "active") }

  it "derives base, dashboard and wpda from the slug" do
    expect(described_class.base(city)).to eq("http://curitiba.localhost:5175")
    expect(described_class.dashboard(city)).to eq("http://curitiba.localhost:5175/dashboard/")
    expect(described_class.wpda(city)).to eq("http://curitiba.localhost:5175/wpda/")
  end

  it "honours CITY_PUBLIC_BASE_TEMPLATE and never doubles the slash" do
    # Mesmo padrão de spec/services/city_database_spec.rb:43-46 (stub de ENV com
    # and_call_original): não há gem de env var na suíte.
    allow(ENV).to receive(:fetch).and_call_original
    allow(ENV).to receive(:fetch)
      .with("CITY_PUBLIC_BASE_TEMPLATE", described_class::DEFAULT_TEMPLATE)
      .and_return("https://%{slug}.rota-saude.example/")

    expect(described_class.base(city)).to eq("https://curitiba.rota-saude.example")
    expect(described_class.dashboard(city)).to eq("https://curitiba.rota-saude.example/dashboard/")
  end

  it "raises instead of building a link without a city" do
    expect { described_class.base(nil) }.to raise_error(CityPublicUrl::CityMissing)
  end
end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_public_url_spec.rb`
Expected: FAIL com `NameError: uninitialized constant CityPublicUrl`.

- [ ] **Step 3: Implementar o serviço**

```ruby
# apps/api/app/services/city_public_url.rb
# Host público de uma cidade e os caminhos que a API precisa apontar para ele
# (Plano 6). Uma env var só, com o slug interpolado: dashboard, wpda e o link de
# reset de senha saem daqui, em vez de três variáveis de host único
# (CITY_DASHBOARD_URL_TEMPLATE, PUBLIC_DASHBOARD_URL, WPDA_PUBLIC_BASE).
#
# Sem cidade no contexto o certo é levantar: um link montado com o host errado
# vai por WhatsApp ou e-mail e quebra sem erro nenhum no servidor.
module CityPublicUrl
  DEFAULT_TEMPLATE = "http://%{slug}.localhost:5175"

  class CityMissing < StandardError; end

  module_function

  def base(city)
    raise CityMissing, "CityPublicUrl sem cidade no contexto" if city.nil?

    format(ENV.fetch("CITY_PUBLIC_BASE_TEMPLATE", DEFAULT_TEMPLATE), slug: city.slug).chomp("/")
  end

  def dashboard(city)
    "#{base(city)}/dashboard/"
  end

  def wpda(city)
    "#{base(city)}/wpda/"
  end
end
```

- [ ] **Step 4: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_public_url_spec.rb`
Expected: PASS (3 examples).

- [ ] **Step 5: Ligar os três consumidores**

`apps/api/app/services/city_dashboard_url.rb` inteiro passa a ser:

```ruby
# URLs do dashboard de uma cidade (Planos 3B e 4): a entrada com grant, para onde o
# console e o callback do gov.br mandam o navegador, e o link do convite do
# primeiro municipal_admin. O host por cidade vem de CityPublicUrl (Plano 6).
module CityDashboardUrl
  module_function

  def for(city, grant:)
    "#{base(city)}?#{{ grant: grant }.to_query}"
  end

  def invitation(city, token:)
    "#{base(city)}?#{{ invite: token }.to_query}"
  end

  def base(city)
    CityPublicUrl.dashboard(city)
  end
end
```

`apps/api/app/models/report_snapshot.rb:28-31` passa a ser:

```ruby
  # Link do cidadão: host da cidade do snapshot (Plano 6). O único chamador é
  # NotifyCitizenJob, que roda dentro de CityScopedJob#with_city — Current.city
  # está setado. Sem cidade, CityPublicUrl levanta em vez de mandar um link
  # para o host errado.
  def url
    "#{CityPublicUrl.wpda(Current.city)}?token=#{token}"
  end
```

`apps/api/app/controllers/passwords_controller.rb:38-42` passa a ser:

```ruby
  # Link para o dashboard DA CIDADE da requisição (Plano 6): o reset é sempre
  # dentro de uma cidade (CityResolution roda antes), então Current.city existe.
  def password_reset_link(token)
    "#{CityPublicUrl.dashboard(Current.city)}?#{{ reset: token }.to_query}"
  end
```

Em `apps/api/app/controllers/reports_controller.rb`, troque as linhas 6-8 do comentário por:

```ruby
# banco da cidade do host (CityResolution), então um token só vale no host da
# própria cidade — o de outra cidade não existe ali. O link enviado ao cidadão
# sai de CityPublicUrl.wpda (host da cidade, Plano 6).
```

- [ ] **Step 6: Especificar o link por cidade no fluxo real**

Adicione ao fim do `describe` de `apps/api/spec/requests/passwords_spec.rb` (leia o arquivo antes para reaproveitar os `let`/helpers existentes):

```ruby
  it "sends a reset link on the host of the city that asked for it" do
    user = create(:user, email_address: "muni@#{TEST_CITY_A.slug}.demo")

    perform_enqueued_jobs do
      post "/passwords", params: { email_address: user.email_address }
    end

    expect(response).to have_http_status(:no_content)
    mail = ActionMailer::Base.deliveries.last
    expect(mail.body.to_s).to include("http://#{TEST_CITY_A.slug}.localhost:5175/dashboard/?reset=")
  end
```

Se o spec de passwords já tiver um exemplo cobrindo o corpo do e-mail, altere-o em vez de duplicar. Se `create(:user, ...)` não for o padrão do arquivo, siga o padrão que estiver lá.

- [ ] **Step 7: Rodar os specs afetados**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_public_url_spec.rb spec/requests/passwords_spec.rb spec/requests/reports_spec.rb spec/jobs/send_whatsapp_job_spec.rb spec/jobs/provision_city_job_spec.rb spec/commands/city_lifecycle/invite_admin_spec.rb spec/tasks/city_rake_spec.rb spec/requests/session_grant_spec.rb`

(Esses são os specs que hoje tocam os três consumidores: `send_whatsapp_job_spec` exercita `NotifyCitizenJob`, e `CityDashboardUrl` aparece em `provision_city_job_spec`, `invite_admin_spec` e `city_rake_spec`. Não existe spec dedicado a `NotifyCitizenJob` nem a `CityDashboardUrl`.)
Expected: PASS. Qualquer spec que ainda espere `localhost:5176/wpda` ou `PUBLIC_DASHBOARD_URL` deve ser atualizado para o host da cidade — é a mudança de contrato desta task.

- [ ] **Step 8: Renomear a env var no deploy e no README**

Em `apps/api/deploy/production/deploy.yml`, troque a linha 59 por:

```yaml
    CITY_PUBLIC_BASE_TEMPLATE: "https://%{slug}.rota-saude.example"
```

Em `apps/api/deploy/development/deploy.yml`, troque a linha 54 por:

```yaml
    CITY_PUBLIC_BASE_TEMPLATE: "https://%{slug}.dev.rota-saude.example"
```

Em `apps/api/README.md`, troque a linha 24 da tabela por:

```markdown
| `CITY_PUBLIC_BASE_TEMPLATE` | `http://%{slug}.localhost:5175` | host público de cada cidade: dashboard, wpda e link de reset de senha |
```

e, na linha 58, troque a frase sobre `CITY_DASHBOARD_URL_TEMPLATE` por:

```markdown
O destino de volta usa `CITY_PUBLIC_BASE_TEMPLATE` (default `http://%{slug}.localhost:5175`), com `/dashboard/`.
```

- [ ] **Step 9: Rodar a suíte completa**

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
```
Expected: 0 failures.

- [ ] **Step 10: Commit**

```bash
cd apps/api && git add app/services/city_public_url.rb app/services/city_dashboard_url.rb app/models/report_snapshot.rb app/controllers/passwords_controller.rb app/controllers/reports_controller.rb deploy README.md spec
git commit -m "Derive every public city link from the city slug

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: CORS pelo catálogo de cidades

**Files:**
- Modify: `apps/api/config/initializers/cors.rb`
- Create: `apps/api/spec/requests/cors_spec.rb`

**Interfaces:**
- Consumes: `CityCatalog.find_by_host(host)`, `CityCatalog.reserved_host?(host)`, `City#servable?`.
- Produces: nenhuma API nova; o comportamento do middleware é o contrato.

- [ ] **Step 1: Escrever o spec**

```ruby
# apps/api/spec/requests/cors_spec.rb
require "rails_helper"

# Plano 6: a lista estática ALLOWED_ORIGINS não enumera N cidades. A origem é
# aceita quando o host dela resolve uma cidade servível do catálogo, ou é um
# host reservado da plataforma (admin/auth/api/www).
RSpec.describe "CORS", type: :request do
  def cors_header_for(origin)
    get "/session", headers: { "Origin" => origin, "Host" => "#{TEST_CITY_A.slug}.rotasaude.app" }
    response.headers["Access-Control-Allow-Origin"]
  end

  before { CityCatalog.reset_cache! }

  it "allows the origin of a city in the catalog" do
    expect(cors_header_for("http://#{TEST_CITY_A.slug}.localhost:5175")).to eq("http://#{TEST_CITY_A.slug}.localhost:5175")
  end

  it "allows the platform console origin" do
    expect(cors_header_for("http://admin.localhost:5174")).to eq("http://admin.localhost:5174")
  end

  it "refuses a host that is not a city" do
    expect(cors_header_for("http://naoexiste.localhost:5175")).to be_nil
  end

  it "refuses a malformed origin" do
    expect(cors_header_for("not a url")).to be_nil
  end
end
```

`TEST_CITY_A` já existe como City no catálogo de teste? Confirme com `grep -rn "TEST_CITY_A.slug" apps/api/spec/requests | head -3`; os specs de request que usam `host!` criam a linha de catálogo via `spec/support/city_request_auth.rb`. Se não criar, adicione no `before` do spec:
`City.find_or_create_by!(slug: TEST_CITY_A.slug) { |c| c.name = "Test City A"; c.status = "active"; c.database_url = TEST_CITY_A.database_url; c.encryption_key = SecureRandom.hex(32); c.schema_version = CitySchema.expected_version.to_s }`.

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/cors_spec.rb`
Expected: FAIL nos dois primeiros exemplos (header nulo: a origem da cidade não está na lista estática).

- [ ] **Step 3: Implementar**

`apps/api/config/initializers/cors.rb`, substituindo a linha 11:

```ruby
    origins do |source, _env|
      next true if ENV.fetch("ALLOWED_ORIGINS", "").split(",").map(&:strip).include?(source)

      host = begin
        URI.parse(source).host
      rescue URI::InvalidURIError
        nil
      end
      next false if host.blank?
      next true if CityCatalog.reserved_host?(host)

      CityCatalog.find_by_host(host)&.servable? || false
    end
```

E troque o comentário do topo (linhas 1-8) por:

```ruby
# API-only. CORS aberto para os hosts conhecidos: qualquer cidade servível do
# catálogo (Plano 6 — a lista estática não enumera N cidades), os hosts
# reservados da plataforma (admin/api/auth/www) e o que estiver em
# ALLOWED_ORIGINS (ferramentas internas, testes).
#
# Webhook do WhatsApp NÃO precisa de CORS (request vem do servidor da Meta).
# credentials: true é obrigatório para o browser enviar/aceitar o cookie de
# sessão (ADR-0011). A resolução usa o cache do CityCatalog (TTL 30 s, teto de
# 500 entradas), então uma origem inventada não vira query por requisição.
```

- [ ] **Step 4: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/cors_spec.rb`
Expected: PASS (4 examples).

- [ ] **Step 5: Suíte completa e commit**

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
cd apps/api && git add config/initializers/cors.rb spec/requests/cors_spec.rb
git commit -m "Accept CORS origins from the city catalog instead of a static list

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: Host da cidade chegando na API em dev

**Files:**
- Modify: `apps/dashboard/vite.config.ts`, `apps/admin/vite.config.ts`, `apps/wpda/vite.config.ts`
- Modify: `apps/dashboard/README.md`, `apps/admin/README.md`, `apps/wpda/README.md`, `apps/api/README.md:36-40`
- Modify (fora do git): `docker-compose.yml`, `.env.example`, `start.sh`

**Interfaces:**
- Produces: em dev, `http://<slug>.localhost:5175/dashboard/` chega ao Rails com `Host: <slug>.localhost:5175`; idem `admin.localhost:5174` e `<slug>.localhost:5176/wpda/`.
- Consumes: `CityPublicUrl` (Task 1) para os links que o backend devolve.

- [ ] **Step 1: Provar o problema antes de mexer**

```bash
docker compose up -d dashboard
docker compose exec -T api sh -c 'true'   # garante api de pé
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: curitiba.localhost:5175" http://localhost:3030/admin/api/overview
curl -s -o /dev/null -w "%{http_code}\n" http://curitiba.localhost:5175/admin/api/overview
```
Expected: o primeiro (Host direto no Rails) responde `401` (cidade resolvida, sessão ausente); o segundo, via Vite, responde `404` — o proxy trocou o Host e o Rails não achou cidade nenhuma. Registre as duas saídas no relatório.

- [ ] **Step 2: Corrigir os três `vite.config.ts`**

Em `apps/dashboard/vite.config.ts`, troque as linhas 4-10 e o bloco `server` por:

```ts
// Frontend do Dashboard operacional da cidade (tenant-scoped, municipal_admin).
// Em dev, Vite proxa as rotas de backend para o Rails (apps/api, :3030).
//   /up         → healthcheck do Rails (sem auth), usado pelo health-ping.
//   /admin/api  → Admin::Api::* (read-only).
//   /session    → SessionsController (inclui /session/grant).
//   /passwords  → PasswordsController.
//   /auth       → POST /auth/govbr/start.
//   /setup      → SetupController (aceite de convite).
//
// Plano 6: changeOrigin FICA FALSE. O Rails resolve a cidade pelo Host da
// requisição (CityCatalog); com changeOrigin: true o proxy reescrevia o Host
// para o alvo (api:3000) e nenhuma cidade chegava. allowedHosts libera
// <slug>.localhost, que o Vite 5.4.12+ bloqueia por padrão.
const proxy = (target: string) => ({ target, changeOrigin: false });
const TARGET = process.env.VITE_API_PROXY_TARGET || "http://localhost:3030";

export default defineConfig({
  plugins: [react()],
  base: "/dashboard/",
  server: {
    port: 5173,
    host: "0.0.0.0",
    allowedHosts: [ ".localhost" ],
    proxy: {
      "/up":        proxy(TARGET),
      "/admin/api": proxy(TARGET),
      "/authoring": proxy(TARGET),
      "/session":   proxy(TARGET),
      "/passwords": proxy(TARGET),
      "/auth":      proxy(TARGET),
      "/setup":     proxy(TARGET)
    }
  }
});
```

Em `apps/admin/vite.config.ts`, troque a linha 14 e o bloco `server`:

```ts
const proxy = (target: string) => ({ target, changeOrigin: false });
const TARGET = process.env.VITE_API_PROXY_TARGET || "http://localhost:3030";

export default defineConfig({
  plugins: [react()],
  base: "/admin/",
  server: {
    port: 5173,
    host: "0.0.0.0",
    allowedHosts: [ ".localhost" ],
    proxy: {
      "/admin/api": proxy(TARGET),
      "/session":   proxy(TARGET),
      "/mfa":       proxy(TARGET),
      "/setup":     proxy(TARGET),
      "/cities":    proxy(TARGET),
      "/city_grants": proxy(TARGET)
    }
  }
});
```

(`/auth` sai do admin: o gov.br é da cidade, decisão 5. `/cities` e `/city_grants` entram para a Task 4.)

Em `apps/wpda/vite.config.ts`, troque a linha 8 e o bloco `server`:

```ts
const proxy = (target: string) => ({ target, changeOrigin: false });
const TARGET = process.env.VITE_API_PROXY_TARGET || "http://localhost:3030";

export default defineConfig({
  plugins: [react()],
  base: "/wpda/",
  server: {
    port: 5173,
    host: "0.0.0.0",
    allowedHosts: [ ".localhost" ],
    proxy: {
      "/up": proxy(TARGET),
      "/r":  proxy(TARGET)
    }
  }
});
```

(`/admin/api` e `/session` saem do wpda: é app público por token, não fala com nenhum dos dois.)

- [ ] **Step 3: Provar que o Host agora chega**

```bash
docker compose restart dashboard admin wpda
sleep 5
curl -s -o /dev/null -w "%{http_code}\n" http://curitiba.localhost:5175/admin/api/overview
curl -s -o /dev/null -w "%{http_code}\n" http://naoexiste.localhost:5175/admin/api/overview
```
Expected: `401` para curitiba (cidade resolvida, falta sessão) e `404` para o slug inexistente. Registre.

- [ ] **Step 4: Ajustar compose, `.env.example` e `start.sh` (fora do git)**

Em `docker-compose.yml`, dentro de `x-api-env`, acrescente depois da linha `ALLOWED_ORIGINS`:

```yaml
  CITY_PUBLIC_BASE_TEMPLATE: ${CITY_PUBLIC_BASE_TEMPLATE:-http://%{slug}.localhost:5175}
```

Em `.env.example`, troque o bloco de CORS por:

```bash
# CORS — além do catálogo de cidades (que é automático), origens extras.
ALLOWED_ORIGINS=http://localhost:5174,http://localhost:5175,http://localhost:5176

# Host público de cada cidade (dashboard, wpda e link de reset de senha).
CITY_PUBLIC_BASE_TEMPLATE=http://%{slug}.localhost:5175
```

Em `start.sh`, no heredoc final, troque as três linhas de admin/dashboard/wpda por:

```bash
  admin ................. http://admin.localhost:${ADMIN_PORT:-5174}/admin/
  dashboard (curitiba) .. http://curitiba.localhost:${DASHBOARD_PORT:-5175}/dashboard/
  dashboard (maringa) ... http://maringa.localhost:${DASHBOARD_PORT:-5175}/dashboard/
  wpda (curitiba) ....... http://curitiba.localhost:${WPDA_PORT:-5176}/wpda/
```

- [ ] **Step 5: Atualizar os READMEs**

Em `apps/api/README.md`, troque as linhas 38-40 por:

```markdown
Hosts de dev: `curitiba.localhost:5175`, `maringa.localhost:5175` (dashboard), `admin.localhost:5174` (console),
`curitiba.localhost:5176` (wpda). O proxy do Vite repassa o Host (Plano 6), então o Rails resolve a cidade pelo
subdomínio como em produção. `*.localhost` resolve para 127.0.0.1 no Chrome e no Firefox sem `/etc/hosts`; no Safari,
acrescente uma linha por cidade.
```

Em `apps/dashboard/README.md`, troque a linha 18-19 por:

```markdown
Acesso: http://curitiba.localhost:5175/dashboard/ (ou o slug de outra cidade de dev). O Vite proxa as chamadas de API
para `VITE_API_PROXY_TARGET` (default `http://localhost:3030`) **sem reescrever o Host** — é o subdomínio que diz ao
Rails qual cidade servir.
```

Em `apps/admin/README.md`, troque a linha 31 por:

```markdown
pnpm dev                   # abre em http://admin.localhost:5174/admin/
```

e a linha 34 por:

```markdown
O Vite proxa `/admin/api/*`, `/session`, `/mfa`, `/setup`, `/cities` e `/city_grants` para `http://localhost:3030`
(container do Rails), sem reescrever o Host: o console vive no host reservado `admin.*`.
```

Em `apps/wpda/README.md`, ajuste a linha de acesso para `http://curitiba.localhost:5176/wpda/?token=<token>`.

- [ ] **Step 6: Commit**

```bash
cd apps/dashboard && git add vite.config.ts README.md
cd ../admin && git add vite.config.ts README.md
cd ../wpda && git add vite.config.ts README.md
cd ../api && git add README.md
```

Cada app é um repositório próprio? Confirme com `git -C apps/dashboard rev-parse --show-toplevel`. Se os quatro estiverem no mesmo repo `apps/api`, só `apps/api/README.md` é versionado — os outros três são pastas fora do git, e nesse caso registre o diff deles no relatório em vez de commitar. Commit (no que for versionado):

```bash
git commit -m "Let the city subdomain reach the API through the Vite dev proxy

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: Console de plataforma — catálogo e entrada na cidade

**Files:**
- Modify: `apps/api/config/routes.rb:18`
- Modify: `apps/api/app/controllers/operators/cities_controller.rb`
- Modify: `apps/api/spec/requests/operators/cities_spec.rb`
- Create: `apps/admin/vitest.config.ts`, `apps/admin/src/lib/cities.ts`, `apps/admin/src/lib/cities.test.ts`
- Modify: `apps/admin/package.json`, `apps/admin/src/lib/api.ts`, `apps/admin/src/lib/auth.tsx`, `apps/admin/src/App.tsx`, `apps/admin/src/modules/Cities.tsx`, `apps/admin/src/modules/Login.tsx`, `apps/admin/src/lib/types.ts`, `apps/admin/src/hooks/useCities.ts`
- Delete: `apps/admin/src/hooks/useCityDetail.ts`

**Interfaces:**
- Consumes: `POST /city_grants {city_slug}` → `201 {redirect_url, expires_in}` (já existe).
- Produces: `GET /cities` no host do console → `200 {data: [{id, slug, name, uf, status, schema_version, created_at}]}`; no frontend, `listCities()` e `enterCity(slug)`.

- [ ] **Step 1: Spec do endpoint de catálogo**

Acrescente ao fim de `apps/api/spec/requests/operators/cities_spec.rb`:

```ruby
  it "lists the catalog for the console, newest first, without secrets" do
    verified_login!
    on_platform_queue { post "/cities", params: params }

    get "/cities"

    expect(response).to have_http_status(:ok)
    rows = json["data"]
    expect(rows.first["slug"]).to eq("novacidade")
    expect(rows.first.keys).to match_array(%w[id slug name uf status schema_version created_at])
    expect(response.body).not_to include("postgres://")
  end

  it "does not list the catalog without a verified operator session" do
    get "/cities"
    expect(response).to have_http_status(:unauthorized)
  end
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/operators/cities_spec.rb`
Expected: FAIL com `ActionController::RoutingError` (não há `GET /cities`).

- [ ] **Step 3: Implementar rota e action**

`apps/api/config/routes.rb`, linha 18:

```ruby
      resources :cities, only: %i[index create show]
```

Em `apps/api/app/controllers/operators/cities_controller.rb`, acrescente antes de `def create`:

```ruby
    # GET /cities — catálogo para o console (Plano 6). Só o que vive na
    # plataforma: nada aqui abre conexão de cidade, então a lista continua
    # barata com N cidades. Métrica por cidade é dentro da cidade (spec §5: o
    # console perde a visão cross-tenant).
    def index
      rows = City.order(created_at: :desc).map do |city|
        {
          id: city.id, slug: city.slug, name: city.name, uf: city.uf,
          status: city.status, schema_version: city.schema_version,
          created_at: city.created_at.iso8601
        }
      end
      render json: { data: rows }
    end
```

e atualize o comentário do topo do arquivo, acrescentando depois da linha do `GET /cities/:id`:

```ruby
#   GET  /cities                                                        → 200 { data: [...] }
```

- [ ] **Step 4: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/operators/cities_spec.rb`
Expected: PASS.

- [ ] **Step 5: Vitest no admin**

`apps/admin/package.json`: acrescente `"test": "vitest run"` em `scripts` e estas três entradas em `devDependencies`:

```json
    "@testing-library/react": "^16.0.0",
    "jsdom": "^25.0.0",
    "vitest": "^2.1.0"
```

Crie `apps/admin/vitest.config.ts`:

```ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: false
  }
});
```

Instale: `docker compose up -d admin && docker compose exec -T admin npm install --no-audit --no-fund`.

- [ ] **Step 6: Teste da camada de cidades do console (RED)**

```ts
// apps/admin/src/lib/cities.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";

vi.mock("./api", () => ({ listCities: vi.fn(), createCityGrant: vi.fn() }));

import { enterCity, sortedForDisplay } from "./cities";
import * as api from "./api";

beforeEach(() => vi.clearAllMocks());

describe("enterCity", () => {
  it("redirects the browser to the grant URL the console received", async () => {
    (api.createCityGrant as ReturnType<typeof vi.fn>).mockResolvedValue({
      redirect_url: "http://curitiba.localhost:5175/dashboard/?grant=tok", expires_in: 60
    });
    const go = vi.fn();

    await enterCity("curitiba", go);

    expect(api.createCityGrant).toHaveBeenCalledWith("curitiba");
    expect(go).toHaveBeenCalledWith("http://curitiba.localhost:5175/dashboard/?grant=tok");
  });
});

describe("sortedForDisplay", () => {
  it("puts cities that need attention first, then the rest by name", () => {
    const rows = [
      { id: "1", slug: "b", name: "Bela", uf: "PR", status: "active", schema_version: "1", created_at: "" },
      { id: "2", slug: "a", name: "Aurora", uf: "PR", status: "provisioning", schema_version: null, created_at: "" },
      { id: "3", slug: "c", name: "Cascavel", uf: "PR", status: "suspended", schema_version: "1", created_at: "" }
    ];
    expect(sortedForDisplay(rows).map((c) => c.slug)).toEqual([ "a", "c", "b" ]);
  });
});
```

Run: `docker compose exec -T admin npm run test`
Expected: FAIL (`Failed to resolve import "./cities"`).

- [ ] **Step 7: Implementar `cities.ts` e o cliente**

```ts
// apps/admin/src/lib/cities.ts
// Catálogo do console e entrada numa cidade (Plano 6). O console não tem sessão
// na cidade: ele pede um grant de 60 s (POST /city_grants) e manda o navegador
// para o host da cidade, que consome o grant em POST /session/grant.
import { createCityGrant } from "./api";
import type { CityRow } from "./types";

const ATTENTION = [ "provisioning", "suspended", "archived" ];

export function sortedForDisplay(rows: CityRow[]): CityRow[] {
  return [ ...rows ].sort((a, b) => {
    const aFirst = ATTENTION.includes(a.status) ? 0 : 1;
    const bFirst = ATTENTION.includes(b.status) ? 0 : 1;
    if (aFirst !== bFirst) return aFirst - bFirst;
    return a.name.localeCompare(b.name, "pt-BR");
  });
}

export async function enterCity(slug: string, go: (url: string) => void): Promise<void> {
  const grant = await createCityGrant(slug);
  go(grant.redirect_url);
}
```

Em `apps/admin/src/lib/types.ts`, troque todo o bloco "Cidades" (linhas 180-229) por:

```ts
// ─── Cidades (catálogo do console, Plano 6) ──────────────────────────────────
export interface CityRow {
  id: string;
  slug: string;
  name: string;
  uf: string | null;
  status: string;
  schema_version: string | null;
  created_at: string;
}
```

Em `apps/admin/src/lib/api.ts`: remova as linhas 14-19 (`municipalityHeader` e `setMunicipalityHeader`), remova a linha 77 (o header em `jsonFetch`), remova as linhas 6-8 do comentário do topo, e acrescente ao fim do arquivo:

```ts
// ─── Console de plataforma (Plano 6) ─────────────────────────────────────────

export interface CityGrant { redirect_url: string; expires_in: number }

export async function listCities(): Promise<CityRow[]> {
  const res = await jsonFetch<{ data: CityRow[] }>("/cities");
  return res.data;
}

export async function createCityGrant(city_slug: string): Promise<CityGrant> {
  return jsonFetch<CityGrant>("/city_grants", {
    method: "POST",
    body: JSON.stringify({ city_slug })
  });
}
```

acrescentando `import type { CityRow } from "./types";` no topo do arquivo.

Em `apps/admin/src/lib/auth.tsx`: remova `setMunicipalityHeader` do import (linha 20) e todas as suas chamadas (linhas 52, 55, 66, 102, 108); `pickDefaultMunicipality` vira:

```tsx
  const pickDefaultMunicipality = useCallback((user: SessionUser) => {
    setActiveMunicipalityIdState(user.memberships[0]?.municipality_id ?? null);
  }, []);
```

- [ ] **Step 8: Reescrever a tela Cidades**

`apps/admin/src/hooks/useCities.ts` inteiro:

```ts
import { useQuery } from "@tanstack/react-query";
import { listCities } from "../lib/api";

// Catálogo da plataforma: não depende de período nem de cidade ativa.
export function useCities() {
  return useQuery({ queryKey: [ "cities" ], queryFn: listCities, staleTime: 30_000 });
}
```

Apague `apps/admin/src/hooks/useCityDetail.ts`.

`apps/admin/src/modules/Cities.tsx` inteiro:

```tsx
// Cidades do catálogo (operador). Sem métricas cross-tenant: o console mostra
// estado de provisionamento e leva o operador para dentro da cidade (Plano 6,
// spec §5).
import { useState } from "react";
import { useCities } from "../hooks/useCities";
import { enterCity, sortedForDisplay } from "../lib/cities";
import { PageHeader } from "../components/PageHeader";
import { DataTable, type Column } from "../components/DataTable";
import { StatusDot } from "../components/StatusDot";
import { EmptyState } from "../components/EmptyState";
import { ErrorState } from "../components/ErrorState";
import { Skeleton } from "../components/Skeleton";
import { fmtTime } from "../lib/format";
import type { CityRow } from "../lib/types";

export function Cities() {
  const { data, isLoading, isError, error, refetch } = useCities();
  const [ busySlug, setBusySlug ] = useState<string | null>(null);
  const [ failure, setFailure ] = useState<string | null>(null);

  async function onEnter(city: CityRow) {
    setBusySlug(city.slug);
    setFailure(null);
    try {
      await enterCity(city.slug, (url) => { window.location.href = url; });
    } catch {
      setFailure(`Não foi possível entrar em ${city.name}. O grant vale 60 s — tente de novo.`);
    } finally {
      setBusySlug(null);
    }
  }

  const cols: Column<CityRow>[] = [
    { label: "Cidade", w: "1.4fr", render: (c) => <span>{c.name}{c.uf ? ` · ${c.uf}` : ""}</span> },
    { label: "Slug", w: "1fr", render: (c) => <span className="mono">{c.slug}</span> },
    { label: "Status", w: "0.9fr", render: (c) => (
        <span style={{ display: "inline-flex", alignItems: "center", gap: 6 }}>
          <StatusDot level={c.status === "active" ? "ok" : "warn"} /> {c.status}
        </span>
      ) },
    { label: "Schema", w: "0.7fr", render: (c) => <span className="mono">{c.schema_version ?? "—"}</span> },
    { label: "Criada", w: "1fr", render: (c) => c.created_at ? fmtTime(c.created_at) : "—" },
    { label: "", w: "0.7fr", align: "right", render: (c) => (
        <button
          type="button"
          disabled={c.status !== "active" || busySlug === c.slug}
          onClick={() => { void onEnter(c); }}
          style={enterBtn}
        >
          {busySlug === c.slug ? "Entrando…" : "Entrar"}
        </button>
      ) }
  ];

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Cidades" sub="catálogo da plataforma" />
      {isLoading && <Skeleton rows={6} />}
      {isError && <ErrorState message={(error as Error)?.message || "Erro"} onRetry={() => refetch()} />}
      {failure && <p role="alert" style={{ color: "var(--down)", fontSize: 12, margin: 0 }}>{failure}</p>}
      {data && data.length === 0 && (
        <EmptyState title="Nenhuma cidade provisionada" sub="use Setup → Provisionar cidade para criar a primeira" />
      )}
      {data && data.length > 0 && (
        <DataTable<CityRow> cols={cols} rows={sortedForDisplay(data)} rowKey={(c) => c.id} />
      )}
    </div>
  );
}

const enterBtn: React.CSSProperties = {
  padding: "5px 10px", borderRadius: 8, border: "1px solid var(--rule2)",
  background: "var(--panel)", color: "var(--ink)", fontSize: 12, cursor: "pointer"
};
```

Em `apps/admin/src/App.tsx`, a linha 76 vira `case "cities": return <Cities />;` (a tela não navega mais para módulos).

- [ ] **Step 9: Tirar o gov.br do console**

Em `apps/admin/src/modules/Login.tsx`: remova a linha 121 (`<GovBrButton />`) e as funções `GovBrButton` (127-175) e `Separator` (177-185). O gov.br passa a existir só no dashboard (Task 5), porque o `state` do gov.br carrega a cidade e o callback volta para o host dela.

- [ ] **Step 10: Rodar testes e typecheck do admin**

```bash
docker compose exec -T admin npm run test
docker compose exec -T admin npm run typecheck
```
Expected: testes PASS; `typecheck` sem erro (se sobrar referência a `CitySummary`/`CityDetailData`/`setMunicipalityHeader`, remova-a).

- [ ] **Step 11: Suíte do api e commit**

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
```

```bash
cd apps/api && git add config/routes.rb app/controllers/operators/cities_controller.rb spec/requests/operators/cities_spec.rb
git commit -m "List the city catalog on the platform console

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

(Os arquivos de `apps/admin` entram no commit se o app for versionado; senão, registre o diff no relatório — ver Task 3, Step 6.)

---

### Task 5: Dashboard — entrar por grant, aceitar convite, gov.br

**Files:**
- Create: `apps/dashboard/src/lib/entry.ts`, `apps/dashboard/src/lib/entry.test.ts`, `apps/dashboard/src/modules/AcceptInvitation.tsx`
- Modify: `apps/dashboard/src/lib/api.ts`, `apps/dashboard/src/lib/scope.ts`, `apps/dashboard/src/main.tsx`, `apps/dashboard/src/modules/Login.tsx`

**Interfaces:**
- Consumes: `POST /session/grant {token}` → 201 `SessionUser` | 401 `{error:"invalid_grant"}`; `POST /setup/accept_invitation {token, password}` → 201 `{id, email_address}`; `POST /auth/govbr/start` → 200 `{authorize_url}`.
- Produces: `readEntryFromUrl(search) -> {kind:"grant"|"invite"|"reset"|null, token}`, `clearEntryFromUrl()`.

- [ ] **Step 1: Teste da leitura da URL (RED)**

```ts
// apps/dashboard/src/lib/entry.test.ts
import { describe, it, expect } from "vitest";
import { readEntryFromUrl } from "./entry";

describe("readEntryFromUrl", () => {
  it("reads a grant", () => {
    expect(readEntryFromUrl("?grant=abc")).toEqual({ kind: "grant", token: "abc" });
  });

  it("reads an invitation", () => {
    expect(readEntryFromUrl("?invite=xyz")).toEqual({ kind: "invite", token: "xyz" });
  });

  it("reads a password reset", () => {
    expect(readEntryFromUrl("?reset=r1")).toEqual({ kind: "reset", token: "r1" });
  });

  it("prefers the grant when more than one is present", () => {
    expect(readEntryFromUrl("?invite=xyz&grant=abc")).toEqual({ kind: "grant", token: "abc" });
  });

  it("returns null when there is nothing to consume", () => {
    expect(readEntryFromUrl("?period=7d")).toBeNull();
    expect(readEntryFromUrl("?grant=")).toBeNull();
  });
});
```

Run: `docker compose up -d dashboard && docker compose exec -T dashboard npm run test`
Expected: FAIL (`Failed to resolve import "./entry"`).

- [ ] **Step 2: Implementar `entry.ts`**

```ts
// apps/dashboard/src/lib/entry.ts
// Entradas que chegam pela URL no host da cidade (Plano 6):
//   ?grant=  → operador vindo do console, ou usuário voltando do gov.br
//   ?invite= → primeiro municipal_admin aceitando o convite
//   ?reset=  → link do e-mail de redefinição de senha
// O grant vem primeiro: ele vale 60 s e uso único, então não pode esperar.
export type EntryKind = "grant" | "invite" | "reset";
export interface Entry { kind: EntryKind; token: string }

const ORDER: EntryKind[] = [ "grant", "invite", "reset" ];

export function readEntryFromUrl(search: string = window.location.search): Entry | null {
  const params = new URLSearchParams(search);
  for (const kind of ORDER) {
    const token = params.get(kind);
    if (token && token.trim() !== "") return { kind, token };
  }
  return null;
}

export function clearEntryFromUrl(): void {
  const url = new URL(window.location.href);
  ORDER.forEach((kind) => url.searchParams.delete(kind));
  window.history.replaceState({}, "", url.toString());
}
```

Run: `docker compose exec -T dashboard npm run test`
Expected: PASS.

- [ ] **Step 3: Cliente HTTP do dashboard**

Acrescente ao fim de `apps/dashboard/src/lib/api.ts`:

```ts
// ─── Entradas na cidade (Plano 6) ────────────────────────────────────────────

export async function redeemGrant(token: string): Promise<SessionUser> {
  return jsonFetch<SessionUser>(`${SESSION_BASE}/grant`, {
    method: "POST",
    body: JSON.stringify({ token })
  });
}

export async function acceptInvitation(token: string, password: string): Promise<void> {
  await jsonFetch<{ id: string; email_address: string }>("/setup/accept_invitation", {
    method: "POST",
    body: JSON.stringify({ token, password })
  });
}

export async function startGovBr(): Promise<string> {
  const res = await jsonFetch<{ authorize_url: string }>("/auth/govbr/start", {
    method: "POST",
    body: JSON.stringify({})
  });
  return res.authorize_url;
}
```

- [ ] **Step 4: Tela de aceite de convite**

```tsx
// apps/dashboard/src/modules/AcceptInvitation.tsx
// Primeiro municipal_admin da cidade define a senha e entra (Plano 6). O token
// é a credencial: chega por e-mail, em ?invite= no host da cidade.
import { useState, type FormEvent } from "react";
import { acceptInvitation } from "../lib/api";
import { ApiError } from "../lib/api";
import { useAuth } from "../lib/auth";

export function AcceptInvitation({ token, onDone }: { token: string; onDone: () => void }) {
  const auth = useAuth();
  const [ password, setPassword ] = useState("");
  const [ confirmation, setConfirmation ] = useState("");
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);

  async function onSubmit(e: FormEvent) {
    e.preventDefault();
    setError(null);
    if (password !== confirmation) {
      setError("As senhas não conferem.");
      return;
    }
    setBusy(true);
    try {
      await acceptInvitation(token, password);
      await auth.reload();
      onDone();
    } catch (err) {
      if (err instanceof ApiError && err.status === 422) {
        setError("Convite inválido ou expirado. Peça um novo convite ao operador.");
      } else if (err instanceof ApiError && err.status === 429) {
        setError("Muitas tentativas. Tente novamente em alguns minutos.");
      } else {
        setError("Não foi possível aceitar o convite. Tente novamente.");
      }
    } finally {
      setBusy(false);
    }
  }

  return (
    <div style={{ minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center" }}>
      <form onSubmit={onSubmit}
        style={{ width: 320, display: "flex", flexDirection: "column", gap: 12, padding: 24,
          border: "1px solid var(--line, #e6e6e6)", borderRadius: 10 }}>
        <strong style={{ fontFamily: "var(--font-mono, monospace)", fontSize: 14 }}>
          Definir senha de acesso
        </strong>
        <label style={{ fontSize: 12, color: "var(--ink2, #444)" }}>
          Senha
          <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} required autoFocus
            style={{ width: "100%", padding: 8, marginTop: 4, borderRadius: 6, border: "1px solid var(--line, #ccc)" }} />
        </label>
        <label style={{ fontSize: 12, color: "var(--ink2, #444)" }}>
          Confirmar senha
          <input type="password" value={confirmation} onChange={(e) => setConfirmation(e.target.value)} required
            style={{ width: "100%", padding: 8, marginTop: 4, borderRadius: 6, border: "1px solid var(--line, #ccc)" }} />
        </label>
        {error && <p role="alert" style={{ color: "var(--danger, #c0341d)", fontSize: 12, margin: 0 }}>{error}</p>}
        <button type="submit" disabled={busy}
          style={{ padding: "8px 12px", borderRadius: 6, border: "none", cursor: busy ? "default" : "pointer",
            background: "var(--accent, #2b59ff)", color: "#fff", fontSize: 13 }}>
          {busy ? "Salvando…" : "Entrar"}
        </button>
      </form>
    </div>
  );
}
```

- [ ] **Step 5: Montar as entradas no `main.tsx`**

Em `apps/dashboard/src/main.tsx`, troque o corpo de `AppRoot` (linhas 13-43) por:

```tsx
function AppRoot() {
  const auth = useAuth();
  const [ entry, setEntry ] = useState<Entry | null>(() => readEntryFromUrl());
  const [ grantError, setGrantError ] = useState(false);
  const [ queryClient ] = useState(() => new QueryClient({
    queryCache: new QueryCache({
      onError(err) {
        if (err instanceof ApiError && err.status === 401) void auth.reload();
      }
    }),
    defaultOptions: {
      queries: {
        refetchOnWindowFocus: false,
        retry: (count, err) => {
          if (err instanceof ApiError && (err.status === 401 || err.status === 404)) return false;
          return count < 1;
        },
        staleTime: 30_000
      }
    }
  }));

  // Grant vale 60 s e uso único: consome na montagem, antes de qualquer tela.
  useEffect(() => {
    if (entry?.kind !== "grant") return;
    let cancelled = false;
    void (async () => {
      try {
        await redeemGrant(entry.token);
        await auth.reload();
      } catch {
        if (!cancelled) setGrantError(true);
      } finally {
        if (!cancelled) { clearEntryFromUrl(); setEntry(null); }
      }
    })();
    return () => { cancelled = true; };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [ entry?.kind, entry?.token ]);

  if (entry?.kind === "reset") return <ResetPassword token={entry.token} />;
  if (entry?.kind === "grant") return <Splash />;
  if (entry?.kind === "invite" && auth.state.kind !== "authenticated") {
    return <AcceptInvitation token={entry.token} onDone={() => { clearEntryFromUrl(); setEntry(null); }} />;
  }

  if (auth.state.kind === "loading") return <Splash />;
  if (auth.state.kind === "anonymous") return <Login expiredGrant={grantError} />;
  return (
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  );
}
```

e ajuste os imports do topo (linhas 1-11) para:

```tsx
import { StrictMode, useEffect, useState } from "react";
import { createRoot } from "react-dom/client";
import { QueryCache, QueryClient, QueryClientProvider } from "@tanstack/react-query";
import "@fontsource-variable/geist/index.css";
import "@fontsource-variable/geist-mono/index.css";
import { App } from "./App";
import { Login } from "./modules/Login";
import { ResetPassword } from "./modules/ResetPassword";
import { AcceptInvitation } from "./modules/AcceptInvitation";
import { AuthProvider, useAuth } from "./lib/auth";
import { ApiError, redeemGrant } from "./lib/api";
import { clearEntryFromUrl, readEntryFromUrl, type Entry } from "./lib/entry";
import "./theme/global.css";
```

- [ ] **Step 6: gov.br e aviso de grant vencido no Login**

Em `apps/dashboard/src/modules/Login.tsx`: a assinatura vira `export function Login({ expiredGrant = false }: { expiredGrant?: boolean }) {`, o import da linha 3 vira `import { ApiError, requestPasswordReset, startGovBr } from "../lib/api";`, e no formulário de login, logo antes do `{error && ...}` (linha 101), acrescente:

```tsx
        {expiredGrant && (
          <p role="alert" style={{ color: "var(--danger, #c0341d)", fontSize: 12, margin: 0 }}>
            O link de entrada expirou (vale 60 segundos). Entre com e-mail e senha, ou peça um novo acesso.
          </p>
        )}
```

e, depois do botão "Esqueci minha senha" (linha 110), acrescente:

```tsx
        <button type="button" onClick={() => { void goToGovBr(setError); }}
          style={{ padding: "8px 12px", borderRadius: 6, border: "1px solid var(--line, #ccc)",
            background: "transparent", color: "var(--ink, #222)", cursor: "pointer", fontSize: 13 }}>
          Entrar com gov.br
        </button>
```

e, ao fim do arquivo:

```tsx
// O gov.br é da CIDADE: o backend monta authorize_url com o state assinado
// (cidade + nonce) e o callback único em auth.* devolve o navegador para cá
// com ?grant= (Planos 3B e 6).
async function goToGovBr(setError: (m: string | null) => void) {
  try {
    window.location.href = await startGovBr();
  } catch {
    setError("gov.br indisponível no momento. Entre com e-mail e senha.");
  }
}
```

- [ ] **Step 7: Tirar `municipality_id` do escopo**

Em `apps/dashboard/src/lib/scope.ts`, `scopeParams` vira:

```ts
// O escopo é o banco da cidade do host (Plano 6): não há município a mandar.
export function scopeParams(scope: Scope): Record<string, string> {
  return { period: scope.period };
}
```

- [ ] **Step 8: Testes e typecheck**

```bash
docker compose exec -T dashboard npm run test
docker compose exec -T dashboard npm run typecheck
```
Expected: PASS. `auth.test.tsx` continua verde (o `municipalityId` do contexto não muda nesta task).

- [ ] **Step 9: Commit**

```bash
cd apps/dashboard && git add src/lib/entry.ts src/lib/entry.test.ts src/lib/api.ts src/lib/scope.ts src/main.tsx src/modules/AcceptInvitation.tsx src/modules/Login.tsx
git commit -m "Consume city grants and invitations on the city host

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

(Se `apps/dashboard` não for repositório versionado, registre o diff no relatório.)

---

### Task 6: Publicar os hosts no proxy do Kamal

**Files:**
- Modify: `apps/api/deploy/production/deploy.yml:34-40`, `apps/api/deploy/development/deploy.yml:31-37`
- Modify: `apps/api/README.md` (seção de deploy)
- Create: `apps/api/spec/config/deploy_hosts_spec.rb`

**Interfaces:**
- Consumes: `CityCatalog::RESERVED`.
- Produces: nenhum código de runtime; o contrato é o arquivo de deploy.

- [ ] **Step 1: Spec do arquivo de deploy (RED)**

```ruby
# apps/api/spec/config/deploy_hosts_spec.rb
require "rails_helper"

# Plano 6: o proxy do Kamal precisa aceitar o console, o callback do gov.br e
# QUALQUER host de cidade — senão a cidade nova provisionada não atende, mesmo
# com a aplicação pronta para servi-la.
RSpec.describe "Kamal proxy hosts" do
  %w[development production].each do |env|
    it "publishes console, auth and a city wildcard in #{env}" do
      config = YAML.load_file(Rails.root.join("deploy/#{env}/deploy.yml"))
      hosts = config.fetch("proxy").fetch("hosts")

      expect(hosts.any? { |h| h.start_with?("admin.") }).to be(true), "sem host do console em #{env}"
      expect(hosts.any? { |h| h.start_with?("auth.") }).to be(true), "sem host do callback gov.br em #{env}"
      expect(hosts.any? { |h| h.start_with?("*.") }).to be(true), "sem curinga de cidade em #{env}"
      expect(config.fetch("proxy")).not_to have_key("host")
    end
  end
end
```

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/config/deploy_hosts_spec.rb`
Expected: FAIL (`key not found: "hosts"`).

- [ ] **Step 2: Publicar os hosts**

`apps/api/deploy/production/deploy.yml`, linhas 34-40:

```yaml
proxy:
  ssl: true
  # Plano 6: um host por papel do domínio. O curinga atende qualquer cidade
  # provisionada sem novo deploy — a aplicação resolve o slug pelo Host
  # (CityCatalog). `admin.` é o console e `auth.` é o callback único do gov.br.
  # O certificado curinga de *.rota-saude.example e o DNS são gate de go-live.
  hosts:
    - api.rota-saude.example
    - admin.rota-saude.example
    - auth.rota-saude.example
    - "*.rota-saude.example"
  app_port: 3000
  healthcheck:
    path: /up
    interval: 5
```

`apps/api/deploy/development/deploy.yml`, linhas 31-37:

```yaml
proxy:
  ssl: false
  hosts:
    - dev.rota-saude.example
    - admin.dev.rota-saude.example
    - auth.dev.rota-saude.example
    - "*.dev.rota-saude.example"
  app_port: 3000
  healthcheck:
    path: /up
    interval: 5
```

- [ ] **Step 3: Rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/config/deploy_hosts_spec.rb`
Expected: PASS (2 examples).

- [ ] **Step 4: Documentar o gate de DNS/TLS**

No `apps/api/README.md`, logo antes da seção "## Ciclo de vida da cidade (Plano 4)", acrescente:

```markdown
## Hosts publicados (Plano 6)

O proxy do Kamal publica quatro hosts: o da API (`api.*`), o console (`admin.*`), o callback do gov.br (`auth.*`) e o
curinga das cidades (`*.<domínio>`). Uma cidade provisionada passa a atender sem deploy novo — quem decide é o
`CityCatalog`, pelo Host da requisição.

**Gate de go-live:** o curinga exige (a) registro DNS `*.<domínio>` apontando para os hosts web e (b) certificado
curinga. O Let's Encrypt do kamal-proxy emite por host via HTTP-01, o que NÃO cobre curinga: para `*.<domínio>` é
preciso DNS-01 com certificado provisionado fora do Kamal, montado no proxy. Sem isso, cada cidade nova precisa de um
host explícito na lista e de um `kamal proxy reboot`.
```

- [ ] **Step 5: Commit**

```bash
cd apps/api && git add deploy/development/deploy.yml deploy/production/deploy.yml README.md spec/config/deploy_hosts_spec.rb
git commit -m "Publish console, auth and city hosts on the Kamal proxy

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 7: Prova em desenvolvimento

**Files:** nenhum arquivo de produção; a entrega é o relatório com as saídas.

**Interfaces:** consome tudo das Tasks 1 a 6.

- [ ] **Step 1: Subir a stack**

```bash
docker compose up -d api worker admin dashboard wpda
docker compose ps --format 'table {{.Name}}\t{{.Status}}\t{{.Ports}}'
```
Expected: cinco serviços `running`. (Não rode `start.sh`.)

- [ ] **Step 2: Cidade resolvida pelo subdomínio**

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://curitiba.localhost:5175/admin/api/overview
curl -s -o /dev/null -w "%{http_code}\n" http://maringa.localhost:5175/admin/api/overview
curl -s -o /dev/null -w "%{http_code}\n" http://naoexiste.localhost:5175/admin/api/overview
```
Expected: `401`, `401`, `404`.

- [ ] **Step 3: Login na cidade e escopo correto**

```bash
curl -s -c /tmp/p6-curitiba.txt -H "Content-Type: application/json" \
  -d '{"email_address":"admin@curitiba.demo","password":"dev-password"}' \
  http://curitiba.localhost:5175/session | head -c 200; echo
curl -s -b /tmp/p6-curitiba.txt http://curitiba.localhost:5175/admin/api/overview | head -c 200; echo
curl -s -o /dev/null -w "%{http_code}\n" -b /tmp/p6-curitiba.txt http://maringa.localhost:5175/admin/api/overview
```
Expected: o login devolve o `SessionUser` com `memberships[0].municipality_id == "curitiba"`; o overview de curitiba responde com envelope; o mesmo cookie em maringa responde `401` — cookie host-only não atravessa cidade.

- [ ] **Step 4: Console lista e entra na cidade**

```bash
curl -s -c /tmp/p6-op.txt -H "Content-Type: application/json" \
  -d '{"email_address":"dev@local","password":"dev-password"}' \
  http://admin.localhost:5174/session | head -c 120; echo
```
Complete o TOTP com o segredo do operador de dev:
```bash
docker compose exec -T api bin/rails runner 'op = Operator.find_by(email_address: "dev@local"); puts ROTP::TOTP.new(op.otp_secret).now'
curl -s -b /tmp/p6-op.txt -c /tmp/p6-op.txt -H "Content-Type: application/json" \
  -d '{"session_id":"<session_id do passo anterior>","code":"<código>"}' \
  http://admin.localhost:5174/session/challenge | head -c 120; echo
curl -s -b /tmp/p6-op.txt http://admin.localhost:5174/cities | head -c 300; echo
curl -s -b /tmp/p6-op.txt -H "Content-Type: application/json" -d '{"city_slug":"curitiba"}' \
  http://admin.localhost:5174/city_grants | sed 's/grant=[^"&]*/grant=REDACTED/' ; echo
```
Expected: `/cities` lista curitiba e maringa com `status` e `schema_version`; `/city_grants` devolve `redirect_url` apontando para `http://curitiba.localhost:5175/dashboard/?grant=…` e `expires_in: 60`. **Não** cole o token no relatório — a linha acima já o mascara.

- [ ] **Step 5: Grant consumido pela cidade**

Use o token do passo anterior (sem imprimi-lo) direto no endpoint da cidade:
```bash
GRANT=$(curl -s -b /tmp/p6-op.txt -H "Content-Type: application/json" -d '{"city_slug":"curitiba"}' \
  http://admin.localhost:5174/city_grants | sed -n 's/.*grant=\([^"&]*\).*/\1/p')
curl -s -o /dev/null -w "%{http_code}\n" -c /tmp/p6-grant.txt -H "Content-Type: application/json" \
  -d "{\"token\":\"$GRANT\"}" http://curitiba.localhost:5175/session/grant
curl -s -o /dev/null -w "%{http_code}\n" -H "Content-Type: application/json" \
  -d "{\"token\":\"$GRANT\"}" http://curitiba.localhost:5175/session/grant
unset GRANT
```
Expected: `201` na primeira chamada e `401` na segunda (uso único).

- [ ] **Step 6: Link do relatório e do reset saem do host da cidade**

```bash
docker compose exec -T api bin/rails runner '
city = City.find_by!(slug: "curitiba")
Current.set(city: city) do
  CityConnection.with(city) do
    s = ReportSnapshot.first
    puts s ? s.url.sub(/token=.*/, "token=REDACTED") : "sem snapshot em curitiba"
  end
end'
curl -s -o /dev/null -w "%{http_code}\n" -H "Content-Type: application/json" \
  -d '{"email_address":"admin@curitiba.demo"}' http://curitiba.localhost:5175/passwords
docker compose exec -T api sh -c 'grep -a -m1 "dashboard/?reset=" log/development.log | sed "s/reset=[^ \"]*/reset=REDACTED/"'
```
Expected: a URL do snapshot começa com `http://curitiba.localhost:5175/wpda/`; o POST responde `204`; o log mostra o link de reset em `http://curitiba.localhost:5175/dashboard/?reset=REDACTED`.

- [ ] **Step 7: CORS pelo catálogo**

```bash
curl -s -D - -o /dev/null -H "Origin: http://curitiba.localhost:5175" \
  http://curitiba.localhost:5175/session | grep -i "access-control-allow-origin"
curl -s -D - -o /dev/null -H "Origin: http://naoexiste.localhost:5175" \
  http://curitiba.localhost:5175/session | grep -ci "access-control-allow-origin"
```
Expected: a primeira devolve o header com a origem; a segunda devolve `0`.

- [ ] **Step 8: Telas no navegador**

Abra e confira, anotando o que viu:
- `http://admin.localhost:5174/admin/` → login do console **sem** botão gov.br; depois do TOTP, "Cidades" lista o catálogo com botão "Entrar" (desabilitado para cidade não ativa).
- Clicar "Entrar" em curitiba → o navegador vai para `http://curitiba.localhost:5175/dashboard/` já autenticado como operador (só leitura).
- `http://curitiba.localhost:5175/dashboard/` em aba anônima → login da cidade com "Entrar com gov.br".
- `http://curitiba.localhost:5175/dashboard/?grant=jatousado` → cai no login com o aviso de link expirado.

- [ ] **Step 9: Suíte completa final**

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
docker compose exec -T dashboard npm run test
docker compose exec -T admin npm run test
docker compose exec -T wpda npm run test
```
Expected: api 0 failures; três suítes de frontend verdes.

- [ ] **Step 10: Relatório**

Registre no relatório da task: as saídas dos passos 2 a 7 (com tokens mascarados), o que apareceu nas telas do passo 8, os totais do passo 9 e o estado final dos containers.

---

## Definition of Done

- [ ] `http://<slug>.localhost:5175/dashboard/` resolve a cidade pelo Host, em dev, sem proxy reverso novo.
- [ ] Console em `admin.localhost:5174/admin/` lista o catálogo e entra numa cidade por grant de 60 s.
- [ ] Dashboard consome `?grant=`, `?invite=` e `?reset=`, e inicia o gov.br da própria cidade.
- [ ] Link do relatório do cidadão, link de reset de senha e URL do convite saem do host da cidade certa.
- [ ] CORS aceita qualquer cidade servível do catálogo e recusa host que não é cidade.
- [ ] `X-Municipality-Id` não existe mais em nenhum frontend, e `scopeParams` não manda `municipality_id`.
- [ ] Kamal publica `api.*`, `admin.*`, `auth.*` e o curinga das cidades, nos dois ambientes.
- [ ] Suíte do api verde; `npm run test` verde nos três frontends; `npm run typecheck` verde no admin e no dashboard.
- [ ] READMEs, `.env.example`, `docker-compose.yml` e `start.sh` descrevem os hosts por cidade.

## NÃO faz

- **Chave de cifra por cidade** (`cities.encryption_key`), custódia, rotação e `city:restore`: Plano 7.
- **Certificado curinga e DNS** de `*.<domínio>`: gate de go-live documentado, não configurado aqui.
- **`contracts/events` com `municipality_id`** (ADR-0015, expand/contract): vive no repositório de contratos, fora deste monorepo.
- **Deploy dos frontends** (Dockerfile, hospedagem estática, CDN): continua inexistente; este plano só corrige dev e os hosts do backend.
- **Renomear o contrato JSON** `municipality_*` → `city_*` nas respostas da API: o envelope segue como está; os frontends apenas param de mandar o parâmetro.
- **Painéis cross-tenant com métricas** no console: saem por decisão de spec (§5).
- **MFA do usuário da cidade** e enrolamento de operador: fora do escopo.

## Riscos

1. **`allowedHosts` e a versão do Vite.** Os lockfiles trazem 5.4.21 nos três apps, acima do 5.4.12 que introduziu a opção. Se um `npm install` resolver para algo anterior, o Vite recusa o Host com "Blocked request". Mitigação: o passo de prova da Task 3 falha alto, e a correção é fixar `vite@^5.4.21`.
2. **Safari e `*.localhost`.** Chrome e Firefox resolvem sozinhos; Safari historicamente não. Mitigação: uma linha por cidade em `/etc/hosts`, documentada no README do api.
3. **Cookie host-only entre cidades.** É o desenho (spec §5), mas na prática significa que o operador que entra em duas cidades tem duas sessões independentes, e sair de uma não sai da outra. A tela do console não promete o contrário.
4. **Grant de 60 s.** Um clique em "Entrar" seguido de demora (ex.: autenticação do SSO no meio) expira o grant. Mitigação: a Task 5 mostra aviso explícito no login em vez de um 401 mudo.
5. **Curinga no proxy sem TLS curinga.** Publicar `*.<domínio>` sem certificado correspondente derruba HTTPS para cidades novas. Mitigação: o gate está documentado na Task 6 e o `ssl: false` do ambiente `development` não sofre disso.
6. **Origem inventada no CORS.** O bloco consulta o catálogo por requisição pré-flight; o cache do `CityCatalog` tem TTL de 30 s e teto de 500 entradas, então uma enxurrada de hosts falsos não vira enxurrada de queries — mas também não é gratuita. Aceito: é o mesmo caminho que `CityResolution` já percorre.
