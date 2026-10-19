# Identidade da cidade e login do painel — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expor a identidade da cidade (nome, UF, fuso, órgão de saúde, brasão) numa rota pública do api, cadastrá-la pelo admin e mostrá-la no login novo do dashboard e no topo do wpda.

**Architecture:**
- **api:**
  - Active Storage no banco da plataforma, com arquivos no MinIO.
  - `cities` ganha `health_department_name` e o anexo `coat_of_arms`.
  - Rota pública `GET /city_identity`, que resolve a cidade só pela plataforma (sem `within_city`) e serve a imagem por proxy.
  - Rota do console `PATCH /cities/:id/identity`.
- **Fronts:** cada um tem um módulo próprio, `cityIdentity.ts`, sem pacote compartilhado (ADR 0025).

**Tech Stack:** Rails 8 (API, multibanco), Active Storage + `aws-sdk-s3` + MinIO, RSpec; React 18 + Vite 5 + TypeScript + Vitest + Testing Library; Playwright no wpda.

**Spec:** `docs/superpowers/specs/2026-10-19-city-identity-and-dashboard-login-design.md` (aprovada, docs `fe07675`). Leia a spec junto com este plano.

## Global Constraints

**Repositórios e fluxo**
- São quatro repositórios: `apps/api`, `apps/dashboard`, `apps/admin`, `apps/wpda`. Cada um tem branch própria: `feat/city-identity`.
- O `docker-compose.yml` da raiz do monorepo **não** está em git. É mudança local.
- Usar `/opt/homebrew/bin/git`, porque o git do PATH aborta.

**Worktrees**
- api: `apps/api/.claude/<wt>`, copiando `config/master.key`, e testes com `docker compose exec -w /rails/.claude/<wt> api …`.
- Fronts: em `rota-saude/.worktrees/<app>-city-identity`, **nunca** dentro de `apps/<app>`, porque isso duplica o React.

**Ordem e avisos**
- O api vai primeiro (Tasks 1–4). Dashboard (5–7), admin (8–9) e wpda (10) dependem dele.
- Antes de começar, avise a sessão "API" (api) e as sessões donas de admin, dashboard e wpda, via SendMessage.

**Testes e commits**
- Commits seguem Conventional Commits, em inglês, com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Specs de request levam `type: :request` explícito, porque a inferência está desligada.
- Antes da suíte completa do api: `docker compose stop worker`. Depois: `docker compose start worker`.

**Valores fixos da spec**
- Brasão: PNG ou WebP, até **256 KB** (262_144 bytes), lado menor ≥ **128 px**, lado maior ≤ **1024 px**, sem SVG.
- `health_department_name`: até **120** caracteres, nulo permitido. `""` grava nulo.
- JSON público, com exatamente estas chaves: `name`, `uf`, `time_zone`, `health_department_name`, `coat_of_arms_url`.
- `coat_of_arms_url = "/city_identity/coat_of_arms?v=<blob.checksum>"`, ou `null`.
- Cabeçalhos do JSON: `Cache-Control: public, max-age=300`.
- Cabeçalhos da imagem: `Cache-Control: public, max-age=31536000, immutable` e `X-Content-Type-Options: nosniff`.
- Cidade que não está `active` responde **404** `{ "error": "unknown_city" }`, o mesmo corpo do host desconhecido.
- Limite de taxa: JSON com **120 por minuto por IP** (`name: "city_identity"`, `store: RateLimitStore`). A imagem fica sem limite.
- Evento de auditoria: `city.identity_updated`, com payload `{ city_id:, fields: [...] }` e, quando há troca de brasão, `coat_of_arms_checksum`. Nunca leva os bytes.
- Saudação no fuso da cidade:
  - 05:00–11:59 → `"Bom dia."`
  - 12:00–17:59 → `"Boa tarde."`
  - 18:00–04:59 → `"Boa noite."`
- Rótulo da cidade: `"<nome> · <UF>"`. Sem UF, só `"<nome>"`.
- Textos do login:
  - Frase: `"Entre para começar o turno em <rótulo>."`. Sem identidade: `"Entre para começar o turno."`.
  - `h1`: `"Entrar no painel de <nome>"`. Sem identidade: `"Entrar no painel"`.
  - Rodapé: `"operado com Rota Saúde"`.
  - Brasão: `alt="Brasão de <nome>"`.
- Breakpoint do login: a moldura dividida vale a partir de **1024 px**. Abaixo, uma faixa no topo, igual ao `@media (max-width: 1023px)` do `responsive.css`.

## Review Focus

1. **Brasão trocado com a aba aberta:** a URL muda (`?v=<checksum>`), e o navegador não pode servir o brasão antigo. Teste na Task 4.
2. **Arquivo disfarçado:** um SVG ou HTML renomeado para `.png`, com `content_type: image/png`, tem de ser recusado pelos bytes. Teste na Task 2.
3. **Login lento ou sem api:** `fetch` pendurado ou rejeitado não atrasa nem bloqueia o formulário, e não aparece erro. Teste na Task 6.
4. **Remover o brasão** apaga o blob e o arquivo na hora (sem job), e a rota de imagem passa a dar 404. Teste na Task 3.
5. **Operador envia identidade com a cidade ainda em `provisioning`:** a gravação funciona, porque é só plataforma, e a rota pública continua 404 até a cidade ficar ativa. Teste nas Tasks 3 e 4.

---

### Task 1: Active Storage no banco da plataforma, com MinIO

**Files:**
- Modify: `apps/api/Gemfile` (adicionar `gem "aws-sdk-s3", require: false`)
- Create: `apps/api/db/platform_migrate/20261019100001_create_active_storage_tables.rb` (as três tabelas padrão do Active Storage, com uuid nas chaves como o resto do schema)
- Modify: `apps/api/db/platform_schema.rb` (gerado)
- Modify: `apps/api/config/application.rb` (`config.active_storage.draw_routes = false`)
- Create: `apps/api/config/initializers/active_storage_platform.rb`
- Create: `apps/api/config/initializers/03_object_storage.rb`
- Modify: `apps/api/config/storage.yml` (serviço `minio`)
- Modify: `apps/api/config/environments/{development,staging,production,test}.rb`
- Modify: `apps/api/.github/workflows/ci.yml` (env inerte do MinIO em `production-boot` e `staging-boot`)
- Modify (local, fora de git): `docker-compose.yml` (serviço `minio` e bucket)
- Test: `apps/api/spec/storage/active_storage_platform_spec.rb`, `apps/api/spec/config/object_storage_guard_spec.rb`

**Interfaces:**
- Produces:
  - `ActiveStorage::Record` conectado na plataforma.
  - Serviço `:minio` (dev, staging, produção) e `:test` (test).
  - `ObjectStorage.check!(env: ENV, rails_env: Rails.env)`, que levanta `ObjectStorage::MissingConfig` em staging e produção sem as quatro variáveis.

- [ ] **Step 1: Write the failing tests**

```ruby
# spec/storage/active_storage_platform_spec.rb
RSpec.describe "Active Storage na plataforma" do
  it "grava o blob no banco da plataforma sem cidade corrente" do
    blob = ActiveStorage::Blob.create_and_upload!(io: StringIO.new("x"), filename: "a.png", content_type: "image/png",
                                                  identify: false, metadata: { analyzed: true })
    expect(ActiveStorage::Blob.connection_db_config.name).to eq("platform")
    expect(ActiveStorage::Blob.find(blob.id)).to be_present
  end

  it "grava na plataforma mesmo dentro de CityConnection.with" do
    city = City.find_by!(slug: TEST_CITY_A.slug)
    id = CityConnection.with(city) { ActiveStorage::Blob.create_and_upload!(io: StringIO.new("y"), filename: "b.png", content_type: "image/png", identify: false, metadata: { analyzed: true }).id }
    expect(ActiveStorage::Blob.find(id)).to be_present
  end
end

RSpec.describe "Rotas do Active Storage", type: :request do
  it "não desenha /rails/active_storage" do
    expect(Rails.application.routes.recognize_path("/rails/active_storage/blobs/redirect/x/y") rescue :none).to eq(:none)
  end
end
```

```ruby
# spec/config/object_storage_guard_spec.rb
RSpec.describe ObjectStorage do
  FULL = { "MINIO_ENDPOINT" => "http://m:9000", "MINIO_BUCKET" => "b", "MINIO_ACCESS_KEY_ID" => "k", "MINIO_SECRET_ACCESS_KEY" => "s" }.freeze
  it("passa com as quatro em produção") { expect { described_class.check!(env: FULL, rails_env: "production") }.not_to raise_error }
  it "falha em produção e staging se faltar uma" do
    %w[production staging].each do |e|
      expect { described_class.check!(env: FULL.except("MINIO_BUCKET"), rails_env: e) }
        .to raise_error(ObjectStorage::MissingConfig, /MINIO_BUCKET/)
    end
  end
  it("não exige nada em development e test") { expect { described_class.check!(env: {}, rails_env: "development") }.not_to raise_error }
end
```

- [ ] **Step 2: Run to verify failure**

Run: `docker compose exec api bundle exec rspec spec/storage/active_storage_platform_spec.rb spec/config/object_storage_guard_spec.rb`
Expected: FAIL (tabela `active_storage_blobs` inexistente / `uninitialized constant ObjectStorage`)

- [ ] **Step 3: Implement**

- **`active_storage_platform.rb`:**
  - `config.active_storage.draw_routes = false` (vai em `config/application.rb`, porque precisa estar antes das rotas).
  - `ActiveSupport.on_load(:active_storage_record) { connects_to database: { writing: :platform, reading: :platform } }`.
- **`03_object_storage.rb`:**
  - Define `module ObjectStorage` com `REQUIRED = %w[MINIO_ENDPOINT MINIO_BUCKET MINIO_ACCESS_KEY_ID MINIO_SECRET_ACCESS_KEY]`, `class MissingConfig < StandardError` e `check!`. A mensagem do erro lista as variáveis ausentes.
  - Chama `ObjectStorage.check!` no boot.
- **`storage.yml`, serviço `minio`:** `service: S3`, com `endpoint`, `bucket`, `access_key_id` e `secret_access_key` lidos de `ENV.fetch(..., nil)`, `region: us-east-1` e `force_path_style: true`.
- **Ambientes:** development, staging e production usam `config.active_storage.service = :minio`; test usa `:test`.
- **CI:** nos jobs `production-boot` e `staging-boot`, acrescentar `MINIO_ENDPOINT: "http://127.0.0.1:1"`, `MINIO_BUCKET: inert`, `MINIO_ACCESS_KEY_ID: inert` e `MINIO_SECRET_ACCESS_KEY: inert`.
- **`docker-compose.yml`:**
  - Serviço `minio` (`minio/minio`, `server /data --console-address :9001`, volume nomeado `minio-data`, portas `9000` e `9001`).
  - Serviço `minio-init` (`minio/mc`), que cria o bucket `rota-saude-dev` e termina.
  - No `x-api-env`: `MINIO_ENDPOINT=http://minio:9000`, `MINIO_BUCKET=rota-saude-dev`, `MINIO_ACCESS_KEY_ID=rota-dev` e `MINIO_SECRET_ACCESS_KEY=rota-dev-secret`.
  - As variáveis novas só chegam a container recriado. Confira com `docker compose exec api printenv MINIO_BUCKET`.
- **Migração:** `bin/rails db:migrate` (plataforma). Depois, `platform:bootstrap` nos bancos de teste, como no README.

- [ ] **Step 4: Run tests to verify they pass**

Run: o mesmo comando do Step 2. Expected: PASS.
Rodar também `bin/rails runner 'puts ActiveStorage::Blob.service.name'` em dev. Expected: `minio`.

- [ ] **Step 5: Commit**

`build: store platform files with Active Storage on MinIO`

---

### Task 2: Verificação do arquivo do brasão pelos bytes

**Files:**
- Create: `apps/api/app/services/city_identity/coat_of_arms_check.rb`
- Test: `apps/api/spec/services/city_identity/coat_of_arms_check_spec.rb`

**Interfaces:**
- Produces: `CityIdentity::CoatOfArmsCheck.call(io) -> Result`.
  - `ok` traz `{ content_type: "image/png" | "image/webp", width: Integer, height: Integer, byte_size: Integer }`.
  - `fail` traz um destes motivos: `:too_large`, `:invalid_format` ou `:invalid_dimensions`.

- [ ] **Step 1: Write the failing test**

Gere os bytes no próprio spec, com helpers `png(w, h)` e `webp_vp8x(w, h)`, `webp_vp8l(w, h)`, `webp_vp8(w, h)`. Cada um monta só o cabeçalho e um preenchimento até o tamanho desejado.

```ruby
it("aceita PNG 512x512") { expect(check(png(512, 512)).payload).to include(content_type: "image/png", width: 512, height: 512) }
it("aceita os três WebP") { [webp_vp8x(300, 200), webp_vp8l(300, 200), webp_vp8(300, 200)].each { |b| expect(check(b).payload).to include(content_type: "image/webp", width: 300, height: 200) } }
it("recusa acima de 262_144 bytes") { expect(check(png(512, 512, pad_to: 262_145)).reason).to eq(:too_large) }
it("aceita exatamente 262_144 bytes") { expect(check(png(512, 512, pad_to: 262_144))).to be_success }
it("recusa lado menor < 128") { expect(check(png(127, 400)).reason).to eq(:invalid_dimensions) }
it("recusa lado maior > 1024") { expect(check(png(200, 1025)).reason).to eq(:invalid_dimensions) }
it "recusa SVG, HTML e GIF, mesmo que se digam PNG" do
  ['<svg xmlns="http://www.w3.org/2000/svg"/>', "<html><script>x</script>", "GIF89a" + "\0" * 20].each do |s|
    expect(check(s).reason).to eq(:invalid_format)
  end
end
it("recusa PNG truncado") { expect(check(png(512, 512)[0, 20]).reason).to eq(:invalid_format) }
```

- [ ] **Step 2: Run to verify failure**

Run: `docker compose exec api bundle exec rspec spec/services/city_identity/coat_of_arms_check_spec.rb`
Expected: FAIL (`uninitialized constant CityIdentity::CoatOfArmsCheck`)

- [ ] **Step 3: Implement `CityIdentity::CoatOfArmsCheck.call(io)`**

Leia no máximo `MAX_BYTES + 1` bytes. O tamanho é checado primeiro.

- **PNG:** assinatura `\x89PNG\r\n\x1a\n`, seguida do chunk `IHDR`. Largura e altura são `uint32` big-endian nos bytes 16–23.
- **WebP:** `RIFF....WEBP` e então um destes, todos little-endian:
  - `VP8X`: largura−1 e altura−1 em 24 bits, nos offsets 24 e 27;
  - `VP8L`: byte de assinatura `0x2f`, depois 14 bits de largura−1 e 14 bits de altura−1;
  - `VP8 `: `0x9d 0x01 0x2a` no offset 23, depois largura e altura em 14 bits, nos offsets 26 e 28.
- Qualquer outra coisa é `:invalid_format`.

- [ ] **Step 4: Run to verify pass**

Run: o mesmo comando. Expected: PASS.

- [ ] **Step 5: Commit**

`feat: validate coat of arms images by their bytes`

---

### Task 3: Identidade na cidade e `PATCH /cities/:id/identity`

**Files:**
- Create: `apps/api/db/platform_migrate/20261019100002_add_health_department_name_to_cities.rb` (`string :health_department_name, limit: 120`)
- Modify: `apps/api/app/models/city.rb` (`has_one_attached :coat_of_arms`, validação de tamanho de `health_department_name`)
- Create: `apps/api/app/commands/update_city_identity.rb`
- Modify: `apps/api/app/controllers/operators/cities_controller.rb` (`identity`, e `city_json` com as chaves novas)
- Modify: `apps/api/config/routes.rb` (`patch "/cities/:id/identity", to: "cities#identity"` no bloco `PlatformConsoleHost`, ao lado de `record_settings`)
- Modify: `apps/api/spec/events/platform_event_payload_guard_spec.rb` (incluir `city.identity_updated` em `R18_PLATFORM_EVENT_NAMES`)
- Test: `apps/api/spec/commands/update_city_identity_spec.rb`, `apps/api/spec/requests/operators/city_identity_spec.rb`

**Interfaces:**
- Consumes: `CityIdentity::CoatOfArmsCheck.call(io)` (Task 2).
- Produces:
  - `UpdateCityIdentity.call(city:, health_department_name: :keep, coat_of_arms: nil, remove_coat_of_arms: false) -> Result`, com `ok` trazendo `{ city:, changed: [String] }` e `fail` trazendo `:invalid_health_department_name`, `:too_large`, `:invalid_format` ou `:invalid_dimensions`.
  - `city_json` ganha `health_department_name` (String | null) e `coat_of_arms_url` (String | null, no mesmo formato da rota pública).
  - `PATCH /cities/:id/identity` (multipart) responde `200 { city: city_json }`, `422 { error: <reason> }` ou `404`.

- [ ] **Step 1: Write the failing tests**

Request spec, no mesmo molde de `city_record_settings_spec.rb` (operador com TOTP, `host! "admin.rotasaude.app"`), com `fixture_file_upload` de PNGs gerados como na Task 2:

```ruby
it "grava nome do órgão e brasão, audita sem bytes" do
  patch "/cities/#{city.id}/identity", params: { health_department_name: "Secretaria Municipal da Saúde", coat_of_arms: upload(png(512, 512)) }
  expect(response).to have_http_status(:ok)
  expect(json["city"]["health_department_name"]).to eq("Secretaria Municipal da Saúde")
  expect(json["city"]["coat_of_arms_url"]).to eq("/city_identity/coat_of_arms?v=#{city.reload.coat_of_arms.blob.checksum}")
  ev = PlatformEvent.where(name: "city.identity_updated").sole
  expect(ev.payload).to include("city_id" => city.id, "fields" => %w[coat_of_arms health_department_name])
  expect(ev.payload.to_s.bytesize).to be < 300
end
it "\"\" limpa o nome do órgão; nada mudou não audita"
it "121 caracteres dá 422 invalid_health_department_name e não grava nada (nem o brasão enviado junto)"
it "SVG renomeado .png dá 422 invalid_format"
it "remove_coat_of_arms apaga blob e anexo na hora, sem job enfileirado" do
  # depois de anexar:
  expect { patch "/cities/#{city.id}/identity", params: { remove_coat_of_arms: "true" } }
    .to change(ActiveStorage::Blob, :count).by(-1)
  expect(ActiveJob::Base.queue_adapter.enqueued_jobs).to be_empty
end
it "anexar não enfileira AnalyzeJob"
it "funciona com a cidade em provisioning"
it "exige operador autenticado (sem sessão: 401)"
it "GET /cities/:id traz health_department_name e coat_of_arms_url"
```

- [ ] **Step 2: Run to verify failure**

Run: `docker compose exec api bundle exec rspec spec/requests/operators/city_identity_spec.rb spec/commands/update_city_identity_spec.rb`
Expected: FAIL (rota inexistente)

- [ ] **Step 3: Implement**

`UpdateCityIdentity` valida tudo antes de escrever, como `UpdateCityRecordSettings`.

- **Anexar:** o upload passa por `CityIdentity::CoatOfArmsCheck` e vira `ActiveStorage::Blob.create_and_upload!(io:, filename: "coat_of_arms.<png|webp>", content_type: <do check>, identify: false, metadata: { analyzed: true, identified: true, width:, height: })`, com `city.coat_of_arms.attach(blob)`. Como o blob já nasce `analyzed`, não há `AnalyzeJob`.
- **Trocar ou remover:** o blob antigo sai com `purge` síncrono, nunca `purge_later`.
- **`changed`:** lista ordenada com `"health_department_name"` e/ou `"coat_of_arms"`.
- **Auditoria:** `Platform.audit("city.identity_updated", city_id:, fields: changed, coat_of_arms_checksum: <novo ou nil>)` só quando `changed` não está vazio.
- **Controller:** lê `params[:health_department_name]` só se a chave veio (senão `:keep`), `params[:coat_of_arms]` e `params[:remove_coat_of_arms] == "true"`.

- [ ] **Step 4: Run to verify pass**

Run: o mesmo comando, mais `spec/events/platform_event_payload_guard_spec.rb` e `spec/requests/operators/city_record_settings_spec.rb`. Expected: PASS.

- [ ] **Step 5: Commit**

`feat: let operators set the city's health department and coat of arms`

---

### Task 4: Rota pública `GET /city_identity`

**Files:**
- Create: `apps/api/app/controllers/city_identities_controller.rb`
- Modify: `apps/api/config/routes.rb` (`get "/city_identity", to: "city_identities#show"` e `get "/city_identity/coat_of_arms", to: "city_identities#coat_of_arms"`, junto das rotas de cidade sem sessão, perto de `/passwords`, e **depois** dos blocos de host da plataforma)
- Test: `apps/api/spec/requests/city_identity_spec.rb`

**Interfaces:**
- Consumes: `City#coat_of_arms` e `health_department_name` (Task 3).
- Produces: `GET /city_identity` → `200` com `{ name, uf, time_zone, health_department_name, coat_of_arms_url }`, ou `404 { error: "unknown_city" }`. `GET /city_identity/coat_of_arms` → bytes ou `404`.

- [ ] **Step 1: Write the failing tests** (`type: :request`, cidades via `create(:city, …)` como em `city_resolution_spec.rb`, e `CityCatalog.reset_cache!` no `after`)

```ruby
it "devolve exatamente as cinco chaves, com cache de 300 s" do
  get "/city_identity"
  expect(JSON.parse(response.body).keys).to match_array(%w[name uf time_zone health_department_name coat_of_arms_url])
  expect(response.headers["Cache-Control"]).to eq("public, max-age=300")
end
it "sem brasão e sem órgão: os dois null"
it "suspended, archived e provisioning dão 404 unknown_city (não 403)" do
  %w[suspended archived provisioning].each do |st|
    create(:city, slug: "c-#{st}", status: st, database_url: city_a_url); host! "c-#{st}.#{base}"
    get "/city_identity"
    expect([response.status, JSON.parse(response.body)]).to eq([404, { "error" => "unknown_city" }])
  end
end
it "host desconhecido e host reservado (admin.*) dão 404"
it "responde mesmo com o banco da cidade inalcançável (database_url inerte) e com schema atrasado"
it "a imagem vem com content_type do blob, cache imutável e nosniff"
it "a imagem dá 404 sem brasão e para cidade não ativa"
it "trocar o brasão muda o v= da URL" do
  # anexa A, lê url; anexa B via UpdateCityIdentity, lê url
  expect(url_after).not_to eq(url_before)
end
it "121ª chamada do JSON no minuto dá 429; a imagem não conta" # Rails.cache em memória no exemplo, como nos specs do mfa
```

- [ ] **Step 2: Run to verify failure**

Run: `docker compose exec api bundle exec rspec spec/requests/city_identity_spec.rb`
Expected: FAIL (`No route matches`)

- [ ] **Step 3: Implement `CityIdentitiesController < ApplicationController`**

- **Resolução de cidade:** `skip_city_resolution`, com um `before_action` próprio: `@city = CityCatalog.find_by_host(request.host)`, que renderiza 404 `unknown_city` a menos que `@city&.status == "active"`. Não chama `CitySchema` nem `CityConnection`.
- **Limite de taxa:** `rate_limit to: 120, within: 1.minute, only: :show, name: "city_identity", by: -> { request.remote_ip }, store: RateLimitStore`, com 429 `{ error: "too_many_requests" }`. O módulo `RateLimitStore` fica local, no molde de `MfaController::RateLimitStore`.
- **Imagem:** a ação `coat_of_arms` usa `send_data blob.download, type: blob.content_type, disposition: "inline"`, mais os dois cabeçalhos dos Global Constraints.
- **Escrita no catálogo:** a leitura do anexo é pela plataforma. O cache do `CityCatalog` guarda o `City`, então recarregue o anexo (`City.find(@city.id)`) para não servir checksum velho até `CACHE_TTL`.

- [ ] **Step 4: Run to verify pass, then full suite**

Run:

```bash
docker compose stop worker
docker compose exec api bundle exec rspec
docker compose start worker
```

Expected: 0 failures.

- [ ] **Step 5: Commit**

`feat: expose the city identity on a public route`

---

### Task 5: dashboard — `cityIdentity` (dado, rótulo e saudação)

**Files:**
- Create: `apps/dashboard/src/lib/cityIdentity.ts`
- Modify: `apps/dashboard/vite.config.ts` (`"/city_identity": proxy(TARGET)`)
- Test: `apps/dashboard/src/lib/cityIdentity.test.ts`

**Interfaces:**
- Produces:
  - `interface CityIdentity { name: string; uf: string | null; time_zone: string; health_department_name: string | null; coat_of_arms_url: string | null }`.
  - `fetchCityIdentity(signal?: AbortSignal): Promise<CityIdentity>`, que usa `fetch` com `credentials: "omit"` e lança erro em resposta não ok.
  - `useCityIdentity(): CityIdentity | null`, que é `null` enquanto carrega e quando falha, sem estado de erro exposto, e aborta ao desmontar.
  - `cityLabel(i: Pick<CityIdentity, "name" | "uf">): string`.
  - `shiftGreeting(now: Date, timeZone?: string): "Bom dia." | "Boa tarde." | "Boa noite."`.

- [ ] **Step 1: Write the failing tests**

```ts
it.each([
  ["2026-10-19T07:59:00Z", "Boa noite."], ["2026-10-19T08:00:00Z", "Bom dia."],   // 04:59 / 05:00 em São Paulo
  ["2026-10-19T14:59:00Z", "Bom dia."],   ["2026-10-19T15:00:00Z", "Boa tarde."],
  ["2026-10-19T20:59:00Z", "Boa tarde."], ["2026-10-19T21:00:00Z", "Boa noite."],
])("shiftGreeting(%s, America/Sao_Paulo) = %s", (iso, g) => expect(shiftGreeting(new Date(iso), "America/Sao_Paulo")).toBe(g));
it("respeita outro fuso", () => expect(shiftGreeting(new Date("2026-10-19T09:30:00Z"), "America/Manaus")).toBe("Bom dia.")); // 05:30
it("cityLabel com e sem UF", () => { expect(cityLabel({ name: "Curitiba", uf: "PR" })).toBe("Curitiba · PR"); expect(cityLabel({ name: "Curitiba", uf: null })).toBe("Curitiba"); });
it("useCityIdentity fica null com fetch rejeitado e não loga erro");
it("useCityIdentity aborta ao desmontar (o signal termina aborted)");
```

- [ ] **Step 2: Run to verify failure**

Run: `docker compose exec dashboard npx vitest run src/lib/cityIdentity.test.ts`
Expected: FAIL (módulo inexistente)

- [ ] **Step 3: Implement**

`shiftGreeting` lê a hora com `Intl.DateTimeFormat("pt-BR", { hour: "numeric", hourCycle: "h23", timeZone })`.

- [ ] **Step 4: Run to verify pass**

Run: o mesmo comando. Expected: PASS.

- [ ] **Step 5: Commit**

`feat: load the city identity before login`

---

### Task 6: dashboard — moldura de entrada e login novo

**Files:**
- Create: `apps/dashboard/src/modules/entry/EntryFrame.tsx`
- Create: `apps/dashboard/src/modules/entry/entry.css` (classes `entry-frame`, `entry-city`, `entry-form`, com `@media (max-width: 1023px)`; só tokens, sem cor literal)
- Modify: `apps/dashboard/src/modules/Login.tsx` (sem estilo inline; o comportamento atual fica igual)
- Modify: `apps/dashboard/src/theme/literals.test.ts` (incluir `modules/entry/EntryFrame.tsx` em `FILES`)
- Test: `apps/dashboard/src/modules/Login.test.tsx`

**Interfaces:**
- Consumes: `useCityIdentity`, `cityLabel` e `shiftGreeting` (Task 5); `Brand.tsx` (ícone).
- Produces: `EntryFrame({ identity, children }: { identity: CityIdentity | null; children: ReactNode })`. A metade da cidade fica à esquerda (faixa no topo abaixo de 1024 px) e `children` à direita. O rodapé `"operado com Rota Saúde"` está sempre presente.

- [ ] **Step 1: Write the failing tests** (`fetchCityIdentity` mockado via `vi.mock("../lib/cityIdentity", …)`, relógio com `vi.useFakeTimers().setSystemTime(...)`)

```ts
it("com identidade: brasão com alt, prefeitura, órgão, h1 e frase", async () => {
  // identity = Curitiba/PR/Secretaria Municipal da Saúde/url
  expect(await screen.findByRole("img", { name: "Brasão de Curitiba" })).toHaveAttribute("src", "/city_identity/coat_of_arms?v=abc");
  expect(screen.getByText("Curitiba · PR")).toBeInTheDocument();
  expect(screen.getByText("Secretaria Municipal da Saúde")).toBeInTheDocument();
  expect(screen.getByRole("heading", { level: 1, name: "Entrar no painel de Curitiba" })).toBeInTheDocument();
  expect(screen.getByText(/Entre para começar o turno em/)).toHaveTextContent("Entre para começar o turno em Curitiba · PR.");
});
it("sem brasão: ícone da marca no lugar, nenhum img de brasão");
it("sem órgão: a linha não aparece");
it("fetch pendurado: o formulário está utilizável no primeiro render", () => {
  // fetchCityIdentity -> new Promise(() => {})
  expect(screen.getByLabelText("E-mail")).toHaveFocus();
  expect(screen.getByRole("button", { name: "Entrar" })).toBeEnabled();
  expect(screen.getByRole("heading", { level: 1, name: "Entrar no painel" })).toBeInTheDocument();
  expect(screen.getByText("Entre para começar o turno.")).toBeInTheDocument();
});
it("fetch rejeitado: igual ao pendurado e sem role=alert");
it("saudação no fuso da cidade (08:00Z em São Paulo = Bom dia.)");
it("mantém erro de credencial, grant expirado, esqueci a senha e gov.br indisponível"); // um it por fluxo, com os textos atuais
```

- [ ] **Step 2: Run to verify failure**

Run: `docker compose exec dashboard npx vitest run src/modules/Login.test.tsx`
Expected: FAIL

- [ ] **Step 3: Implement**

- **`Login.tsx`:** chama `useCityIdentity()` e renderiza `EntryFrame` com a saudação, a frase e o formulário atual.
- **Modo "Esqueci minha senha":** usa a mesma moldura.
- **Saudação:** calculada no render, sem timer.
- **`h1`:** pode ter classe de leitor de tela. A saudação é um `p` de destaque.
- **Fundo da faixa:** `var(--accent)`, com texto em `var(--on-accent)`.

- [ ] **Step 4: Run to verify pass, then the dashboard suite**

Run: `docker compose exec dashboard npx vitest run`
Expected: PASS, inclusive `src/theme/*` (literais, contraste, a11y, usabilidade).

- [ ] **Step 5: Commit**

`feat: redesign the dashboard login around the city identity`

---

### Task 7: dashboard — telas irmãs na mesma moldura

**Files:**
- Modify: `apps/dashboard/src/modules/ResetPassword.tsx`, `apps/dashboard/src/modules/AcceptInvitation.tsx`
- Test: `apps/dashboard/src/modules/AcceptInvitation.test.tsx` (existente), `apps/dashboard/src/modules/ResetPassword.test.tsx` (novo)

**Interfaces:**
- Consumes: `EntryFrame` (Task 6) e `useCityIdentity` (Task 5).

- [ ] **Step 1: Write the failing tests**

Nas duas telas: `screen.getByText("operado com Rota Saúde")` está presente, e o brasão aparece com `alt="Brasão de Curitiba"` quando há identidade.

- [ ] **Step 2: Run to verify failure**

Run: `docker compose exec dashboard npx vitest run src/modules/ResetPassword.test.tsx src/modules/AcceptInvitation.test.tsx`
Expected: FAIL nos asserts novos. Os existentes passam.

- [ ] **Step 3: Implement**

Envolver o conteúdo atual em `EntryFrame`, sem mudar textos nem fluxo.

- [ ] **Step 4: Run the dashboard suite**

Run: `docker compose exec dashboard npx vitest run`
Expected: PASS.

- [ ] **Step 5: Commit**

`feat: use the city entry frame on password reset and invitation`

---

### Task 8: admin — cliente e seção "Identidade" na ficha da cidade

**Files:**
- Create: `apps/admin/src/lib/cityIdentity.ts`
- Modify: `apps/admin/src/lib/api.ts` (`CityDetail` ganha `health_department_name: string | null` e `coat_of_arms_url: string | null`)
- Create: `apps/admin/src/modules/cities/CityIdentityForm.tsx`
- Modify: `apps/admin/src/modules/cities/CityRecord.tsx` (seção "Identidade" acima de `RecordSettingsForm`)
- Test: `apps/admin/src/lib/cityIdentity.test.ts`, `apps/admin/src/modules/cities/CityIdentityForm.test.tsx`

**Interfaces:**
- Consumes: `PATCH /cities/:id/identity` (Task 3).
- Produces:
  - `updateCityIdentity(cityId: string, patch: { healthDepartmentName?: string; coatOfArms?: File; removeCoatOfArms?: boolean }): Promise<CityDetail>`. Monta um `FormData`, faz `fetch` próprio com `credentials: "include"`, `Accept: application/json` e **sem** `Content-Type`, e lança `ApiError` em resposta não ok.
  - `checkCoatOfArmsFile(file: File): Promise<string | null>`, que devolve a mensagem de erro em pt-BR ou `null`. Usa `file.type ∈ {image/png, image/webp}`, `file.size ≤ 262_144` e as dimensões via `createImageBitmap`.
  - `CityIdentityForm({ city, onSaved }: { city: CityDetail; onSaved: (c: CityDetail) => void })`.

- [ ] **Step 1: Write the failing tests**

```ts
it("updateCityIdentity envia FormData sem Content-Type", async () => {
  await updateCityIdentity("c1", { healthDepartmentName: "Secretaria", coatOfArms: file });
  const [url, init] = fetchMock.mock.calls[0];
  expect(url).toBe("/cities/c1/identity");
  expect(init.method).toBe("PATCH");
  expect(init.body).toBeInstanceOf(FormData);
  expect(new Headers(init.headers).has("Content-Type")).toBe(false);
});
it("checkCoatOfArmsFile recusa SVG, > 256 KB e 127 px, aceita PNG 512"); // createImageBitmap mockado
it("CityIdentityForm mostra a prévia como o login vai mostrar, troca, remove e salva o nome do órgão");
it("422 do api aparece como mensagem e não some o arquivo escolhido");
```

- [ ] **Step 2: Run to verify failure**

Run: `docker compose exec admin npx vitest run src/lib/cityIdentity.test.ts src/modules/cities/CityIdentityForm.test.tsx`
Expected: FAIL

- [ ] **Step 3: Implement**

Mensagens de erro do api mapeadas em pt-BR:
- `invalid_format`: "Use PNG ou WebP."
- `too_large`: "Até 256 KB."
- `invalid_dimensions`: "Entre 128 e 1024 px."
- `invalid_health_department_name`: "Até 120 caracteres."

- [ ] **Step 4: Run the admin suite**

Run: `docker compose exec admin npx vitest run`
Expected: PASS.

- [ ] **Step 5: Commit**

`feat: edit the city identity in the platform console`

---

### Task 9: admin — identidade no provisionamento

**Files:**
- Modify: `apps/admin/src/modules/setup/ProvisionCity.tsx`
- Test: `apps/admin/src/modules/setup/ProvisionCity.test.tsx` (criar, ou ampliar o existente)

**Interfaces:**
- Consumes: `createCity` (existente), `updateCityIdentity` e `checkCoatOfArmsFile` (Task 8).

- [ ] **Step 1: Write the failing tests**

```ts
it("com órgão e brasão: chama createCity e depois updateCityIdentity com o id devolvido");
it("sem os campos opcionais: não chama updateCityIdentity");
it("createCity ok e identidade falha: mostra 'Cidade criada, mas a identidade não foi salva.' com o botão 'Tentar de novo', que reenvia só a identidade");
it("arquivo inválido bloqueia o envio antes de criar a cidade");
```

- [ ] **Step 2: Run to verify failure**

Run: `docker compose exec admin npx vitest run src/modules/setup/ProvisionCity.test.tsx`
Expected: FAIL

- [ ] **Step 3: Implement**

- Campos opcionais: "Nome oficial do órgão de saúde" e "Brasão" (com prévia).
- O aviso é só estado local, sem persistir, conforme a spec §6.

- [ ] **Step 4: Run the admin suite**

Run: `docker compose exec admin npx vitest run`
Expected: PASS.

- [ ] **Step 5: Commit**

`feat: set the city identity while provisioning`

---

### Task 10: wpda — `CityHeader` alimentado pelo api

**Files:**
- Create: `apps/wpda/src/lib/cityIdentity.ts`
- Modify: `apps/wpda/src/modules/citizen/CityHeader.tsx`
- Modify: `apps/wpda/vite.config.ts` (`"/city_identity": proxy(TARGET)`)
- Test: `apps/wpda/src/lib/cityIdentity.test.ts`, `apps/wpda/src/modules/citizen/CityHeader.test.tsx`

**Interfaces:**
- Produces:
  - `loadCityIdentity(): Promise<CityIdentity | null>`, memoizada no módulo: uma requisição por carregamento de página, e `null` em falha. O tipo `CityIdentity` é o mesmo da Task 5, redeclarado aqui.
  - `useCityIdentity(): CityIdentity | null`.
  - `CityHeader()` deixa de receber props e chama o hook.

- [ ] **Step 1: Write the failing tests**

```ts
it("duas montagens de CityHeader fazem uma requisição só");
it("com identidade: img alt='Brasão de Curitiba' e 'Prefeitura de Curitiba'");
it("rota falhando: 'Rota Saúde' e ícone neutro, como hoje");
```

- [ ] **Step 2: Run to verify failure**

Run: `docker compose exec wpda npx vitest run src/lib/cityIdentity.test.ts src/modules/citizen/CityHeader.test.tsx`
Expected: FAIL

- [ ] **Step 3: Implement**

- O texto com identidade é `"Prefeitura de <nome>"`. A reserva continua `"Rota Saúde"`.
- O `alt` deixa de ser vazio.
- Os quatro usos atuais (`Report`, `PhoneOnly`, `A11yPage`, `ui.tsx`) não mudam a chamada.

- [ ] **Step 4: Run the wpda suite and e2e**

Run:

```bash
docker compose exec wpda npx vitest run
cd apps/wpda && npx playwright test
```

Expected: PASS nos perfis atuais.

- [ ] **Step 5: Commit**

`feat: show the city coat of arms in the citizen header`

---

## Encerramento (sem código)

- [ ] **Verificação manual em dev, com evidência (capturas):**
  1. Enviar o brasão de Curitiba e o nome do órgão pelo admin.
  2. Abrir `curitiba.localhost` no dashboard (login) e no wpda (topo).
  3. Verificar uma cidade sem brasão.
  4. Rodar `docker compose stop minio`: o login continua funcionando, só a imagem falha.
- [ ] **Card api#73:**
  - Atualizar o escopo: wpda e login do dashboard.
  - Listar as decisões da spec §3, com o link da spec e deste plano.
  - Registrar os gates: borda HTTP e MinIO de staging e produção.
  - Mover no board até Done ou Verified, conforme `board-91-features-roadmap`.
- [ ] **Gates de go-live:** acrescentar os dois gates da spec §10 à lista de gates.
- [ ] **Push:** só depois de checar `origin/main..main` em cada repo, publicando apenas os commits desta sessão.
