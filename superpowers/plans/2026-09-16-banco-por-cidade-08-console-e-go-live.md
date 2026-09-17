# Plano 8 — Console de provisionamento e go-live

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** fechar a migração banco-por-cidade: o console volta a provisionar cidade pelo endpoint que existe, a cidade provisionada passa a ter canal de WhatsApp, e os gates operacionais acumulados (hosts, conexões, TLS, chave de relatório, restore, servir os SPAs, CI) saem de "documentado" para "feito".

**Architecture:** o console (`admin.*`) só fala com rotas de `PlatformConsoleHost`; tudo que age sobre dado de cidade acontece dentro da cidade, via grant. A API ganha um endpoint de canal e uma rake de restore; a chave que assina token de relatório passa a derivar por cidade, no mesmo padrão do Plano 7. A infraestrutura ganha certificado curinga, TLS verificado, orçamento de conexões e um pipeline que serve os três SPAs.

**Tech Stack:** Rails 8.1 (API), React 19 + Vite 5 + Vitest (3 SPAs), Postgres 16 em produção / 15 em dev, Kamal 2 + kamal-proxy, Solid Queue.

**Spec:** `docs/superpowers/specs/2026-09-12-banco-por-cidade-design.md` (§4 ciclo de vida, §5 identidade e frontends, §6 chaves por cidade, Riscos abertos 2 e 4, Verification).

**Planos anteriores:** 1–7 mergeados e pusheados. Este é o último da série.

## Decisões do usuário (2026-09-16, não re-perguntar)

1. **Escopo = código + infraestrutura agora.** Os itens operacionais não viram só checklist: viram tarefas, com os passos que dependem de acesso externo marcados como **BLOQUEADO (dono: usuário)**.
2. **As telas city-scoped saem do console.** Membros, convites, revogação e desativação não voltam para `admin.*`: o operador entra na cidade pelo grant e opera lá. O console fica com catálogo, provisionamento, canal e entrada por grant.
3. **Entram os quatro opcionais:** `config.hosts` em produção, `report_signing_key` por cidade, `CONNECTION LIMIT` por role, e `city:restore`.

## Decisões minhas na escrita (cada uma com o custo se estiver errada)

4. **Endpoint de canal entra no plano.** `MunicipalityChannels::Register` existe desde o Plano 4 e **não tem rota nenhuma**. Uma cidade provisionada hoje fica `active` sem canal, e `Whatsapp::Ingest` não consegue rotear mensagem para ela — o provisionamento pelo console seguiria incompleto mesmo depois de repontado. Custo se errado: um endpoint e uma tela a mais no console.
5. **Token de relatório: verificação dupla durante a transição, não reescrita imediata.** Assinar por cidade invalida as assinaturas já gravadas. Como o snapshot expira em 30 dias (`GenerateReportJob::EXPIRATION`), a verificação aceita a assinatura nova e, como fallback, a legada; uma rake reescreve as existentes. O fallback tem data de remoção documentada. Custo se errado: 30 dias com dois caminhos de verificação.
6. **CI entra, mínimo.** Nenhum dos quatro repositórios tem `.github/workflows`. Sem CI, os gates que a suíte cobre não protegem nada no merge, e a validação em PG 16 (que a suíte nunca exercita) não acontece em lugar nenhum. Custo se errado: quatro arquivos de workflow que ninguém olha.
7. **Servir os SPAs: nginx numa imagem por app, publicada pelo mesmo proxy.** Alternativa descartada: servir os assets pelo Rails (`public/`), que acoplaria o deploy dos três frontends ao da API e quebraria o cache por app. Custo se errado: três Dockerfiles a mais.

## Global Constraints

Valem para TODA task:

- **Nunca imprimir segredo**: `encryption_key`, chave derivada, ciphertext, `access_token` de canal, senha de role, token de convite ou de relatório — nem em log, relatório, terminal ou diff de falha de spec. Contagens, booleanos, digests e classes de exceção, sim.
- Commits em inglês, terminando com a linha exata `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Comandos Ruby/Rails/rspec rodam no container, a partir da raiz do monorepo `<raiz-do-monorepo>`: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec ...`. Nunca no host.
- `git` do PATH está quebrado nesta máquina (`/usr/bin/git` aborta com erro do Xcode): use `/opt/homebrew/bin/git`. **Nunca dar push.**
- Suíte completa: `docker compose stop worker` antes, `docker compose start worker` depois, timeout ≥ 600000 ms, nunca duas suítes ao mesmo tempo, nunca `git stash`. Baseline de entrada: **812 exemplos, 0 falhas**.
- Nada de operação destrutiva em `rota_saude_development`, `rota_saude_test`, `rota_saude_platform_*`, `rota_saude_no_city_selected`. As cidades de dev `curitiba` e `maringa` **já estão migradas para chave própria** (Plano 7) e não devem ser recriadas; `cascavel` e `londrina` estão `archived` e sem banco.
- Nunca rodar `start.sh`.
- **Os frontends são repositórios git separados.** `apps/dashboard` e `apps/wpda` têm remote no GitHub; **`apps/admin` NÃO tem remote** — os commits dele só existem nesta máquina. Commitar sempre com caminhos explícitos (o `admin` carrega WIP alheio não commitado, preservado de propósito).
- Passo marcado **BLOQUEADO (dono: usuário)** não é para o implementador executar: é para documentar, deixar pronto e parar, relatando o que falta.

## Fatos verificados (não re-descobrir)

- `POST /cities` existe em `PlatformConsoleHost` → `Operators::CitiesController#create`, payload `{slug, name, uf, ibge_code, admin_email, alert_email}`, resposta `202 {id}`. Erros: `503 {error:"misconfigured"}` sem `CITY_DATABASE_HOST`, `409` com `city_exists`, `422` com `{error, message}`.
- `ProvisionCity` valida: slug por `CityDatabase.valid_slug?` (2–40 chars, rótulo DNS, não reservado), `uf` `/\A[A-Z]{2}\z/`, `ibge_code` `/\A\d{7}\z/`, os dois e-mails por `URI::MailTo::EMAIL_REGEXP`.
- `ProvisionCityJob` **não semeia `consent_terms`** (o texto vem das credentials) e semeia `city_profile`, um `AlertRecipient` de e-mail, o protocolo template e o convite do primeiro `municipal_admin`. O canal **não** é semeado.
- `/setup/*` são rotas de cidade (sem constraint de host) e `SetupController` herda de `ApplicationController`, que inclui `CityResolution`. Em `admin.*` (rótulo reservado) elas respondem `404 unknown_city`.
- `NAV_GROUPS` (`apps/admin/src/shell/modules.ts`) já expõe só "Cidades"; os módulos city-scoped continuam no `ModuleId` e no `switch` de `App.tsx`, apenas fora da navegação.
- `ReportSnapshot.sign` é chamado em `GenerateReportJob:19` e `lib/dashboard_demo.rb:200`; a verificação é `find_by_signed_token`, usada só por `ReportsController#show` (`GET /r/:token`, rota de cidade).
- `CityDatabase.ensure!` cria/altera role com `WITH LOGIN PASSWORD`, sem `CONNECTION LIMIT`.
- `config.hosts` só é setado em `development.rb:32-33`. Vazio em produção = `ActionDispatch::HostAuthorization` inerte.
- Kamal: `proxy.ssl: true` com `"*.rota-saude.example"` em `proxy.hosts`; acessório `postgres:16` sem `max_connections`; `CITY_DATABASE_SSLMODE: "require"`; o Dockerfile já instala `ca-certificates`.
- Os três SPAs: `base` `/admin/`, `/dashboard/`, `/wpda/`; porta 5173 interna; `build` = `tsc -b && vite build`; `test` = `vitest run`. Nenhum repositório tem `.github/workflows`.

---

## File Structure

**apps/api**
- Modify: `config/routes.rb` (rota do canal), `app/controllers/operators/` (novo controller), `app/services/city_encryption.rb` (chave de relatório), `app/models/report_snapshot.rb`, `app/jobs/generate_report_job.rb`, `app/services/city_database.rb` (CONNECTION LIMIT), `config/environments/production.rb` (`config.hosts`), `lib/tasks/city.rake` (restore, resign), `deploy/production/deploy.yml`, `deploy/development/deploy.yml`, `Dockerfile`, `README.md`, `deploy/SECRETS.md`
- Create: `app/controllers/operators/city_channels_controller.rb`, `app/commands/city_lifecycle/restore.rb`, `app/commands/city_reports/resign.rb`, `.github/workflows/ci.yml`
- Test: `spec/requests/operators/city_channels_spec.rb`, `spec/architecture/host_authorization_spec.rb`, `spec/models/report_snapshot_spec.rb`, `spec/commands/city_lifecycle/restore_spec.rb`, `spec/commands/city_reports/resign_spec.rb`, `spec/services/city_database_spec.rb`

**apps/admin**
- Modify: `src/lib/api.ts`, `src/App.tsx`, `src/shell/modules.ts`, `src/modules/Cities.tsx`
- Create: `src/modules/setup/ProvisionCity.tsx`, `src/modules/setup/RegisterChannel.tsx`, `src/lib/provisioning.ts`, `src/lib/provisioning.test.ts`, `Dockerfile`, `nginx.conf`, `.github/workflows/ci.yml`
- Delete: `src/modules/setup/ProvisionMunicipality.tsx`, `src/modules/setup/Members.tsx`

**apps/dashboard**, **apps/wpda**
- Create: `Dockerfile`, `nginx.conf`, `.github/workflows/ci.yml`

---

## Task 1: o console para de chamar rota que não existe

**Files:**
- Modify: `apps/admin/src/lib/api.ts`, `apps/admin/src/App.tsx`, `apps/admin/src/shell/modules.ts`
- Delete: `apps/admin/src/modules/setup/ProvisionMunicipality.tsx`, `apps/admin/src/modules/setup/Members.tsx`

**Interfaces:**
- Consome: nada de tasks anteriores.
- Produz: `ModuleId` sem `setup_provision` e `setup_members`; `api.ts` sem as funções `/setup/*` de município e membership.

Por que remover em vez de consertar: essas telas chamam `/setup/*`, que são rotas de CIDADE. Em `admin.*` o `CityResolution` responde `404 unknown_city` antes da autenticação, porque `admin` é rótulo reservado. Elas não estão "quebradas por um bug": estão no host errado por desenho. O caminho é o grant (decisão 2 do usuário).

- [ ] **Step 1: apagar as duas telas mortas**

```bash
cd apps/admin && /opt/homebrew/bin/git rm src/modules/setup/ProvisionMunicipality.tsx src/modules/setup/Members.tsx
```

- [ ] **Step 2: tirar do `ModuleId` e da navegação**

Em `src/shell/modules.ts`, remova `| "setup_provision"` e `| "setup_members"` do union `ModuleId`. **Não acrescente nada aqui:** cada tela nova entra junto com o seu próprio membro do union, na Task 2 e na Task 3. Declarar o membro antes da tela existir deixaria o `switch` de `renderModule` não-exaustivo e quebraria o `typecheck` no fim desta task. `NAV_GROUPS` já lista só "Cidades".

Atualize o comentário do bloco (linhas 34-40) para dizer o que passou a ser verdade:

```ts
// Plano 6/8: o console (admin.*) só responde às rotas do PlatformConsoleHost.
// Os módulos city-scoped (Visão geral, Ingestão, Conversas, Consentimento,
// Triagens, Classificação, Protocolos, Eventos, Filas, Saúde) 404 em admin.* e
// ficam fora da navegação; os arquivos seguem no repo. As telas de membership e
// de provisionamento antigo foram REMOVIDAS no Plano 8: membership é operação
// dentro da cidade (entre pela cidade, via grant) e o provisionamento passou a
// falar com POST /cities.
```

- [ ] **Step 3: tirar do `switch` de `App.tsx`**

Remova os imports de `ProvisionMunicipality` e `Members` e os dois `case` correspondentes em `renderModule`. Mantenha `setup_mfa` e `cities`.

- [ ] **Step 4: tirar do cliente HTTP**

Em `src/lib/api.ts`, remova: `ProvisionPayload`, `ProvisionResult`, `setupProvisionMunicipality`, `InvitePayload`, `InviteResult`, `setupInviteMember`, `MembershipRow`, `setupListMemberships`, `setupRevokeMembership`. **Mantenha** `setupAcceptInvitation` (é a tela de aceite do convite, que roda no host da CIDADE, não no console) e `SETUP_BASE`.

- [ ] **Step 5: verificar que nada ficou pendurado**

```bash
cd apps/admin && grep -rn "setup_provision\|setup_members\|ProvisionMunicipality\|setupInviteMember\|setupListMemberships\|setupRevokeMembership" src/ || echo "(limpo)"
npm run typecheck && npm test
```
Expected: `(limpo)`, typecheck sem erro, vitest verde.

- [ ] **Step 6: commit**

```bash
cd apps/admin && /opt/homebrew/bin/git add src/lib/api.ts src/App.tsx src/shell/modules.ts
/opt/homebrew/bin/git commit -m "Drop console screens that call city-scoped routes

Membership and the old provisioning form posted to /setup/*, which resolve the
city by host. On admin.* the reserved label makes CityResolution answer 404
before auth, so those screens could never work there. Membership belongs inside
the city, reached through the signed grant.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 2: provisionar cidade pelo endpoint real

**Files:**
- Create: `apps/admin/src/lib/provisioning.ts`, `apps/admin/src/lib/provisioning.test.ts`, `apps/admin/src/modules/setup/ProvisionCity.tsx`
- Modify: `apps/admin/src/lib/api.ts`, `apps/admin/src/shell/modules.ts`, `apps/admin/src/App.tsx`, `apps/admin/src/modules/Cities.tsx`

**Interfaces:**
- Consome: `ModuleId` da Task 1.
- Produz: `createCity(payload)` em `api.ts`; `validateProvisionForm` em `provisioning.ts`; módulo `provision_city`.

O endpoint é assíncrono: responde `202 {id}` e o worker faz o resto. A tela não pode prometer cidade pronta nem mostrar convite — o convite vai por e-mail, e o token nunca volta para o console (contrato do Plano 4).

- [ ] **Step 1: escrever o teste da validação (RED)**

```ts
// apps/admin/src/lib/provisioning.test.ts
import { describe, it, expect } from "vitest";
import { validateProvisionForm, type ProvisionCityPayload } from "./provisioning";

const valid: ProvisionCityPayload = {
  slug: "curitiba", name: "Curitiba", uf: "PR", ibge_code: "4106902",
  admin_email: "admin@curitiba.pr.gov.br", alert_email: "urgencia@curitiba.pr.gov.br"
};

describe("validateProvisionForm", () => {
  it("accepts a payload the API would accept", () => {
    expect(validateProvisionForm(valid)).toEqual([]);
  });

  // Espelha ProvisionCity#validation_errors e CityDatabase.valid_slug? — o
  // objetivo é o operador ver o erro ANTES do 422, não duplicar a autoridade:
  // a API continua validando.
  it("rejects a slug that is not a DNS label", () => {
    expect(validateProvisionForm({ ...valid, slug: "Curitiba_PR" })).toContain("slug");
  });

  it("rejects a reserved slug", () => {
    expect(validateProvisionForm({ ...valid, slug: "admin" })).toContain("slug");
  });

  it("rejects UF that is not two uppercase letters", () => {
    expect(validateProvisionForm({ ...valid, uf: "pr" })).toContain("uf");
  });

  it("rejects an IBGE code that is not 7 digits", () => {
    expect(validateProvisionForm({ ...valid, ibge_code: "12345" })).toContain("ibge_code");
  });

  it("rejects a malformed e-mail in either field", () => {
    expect(validateProvisionForm({ ...valid, admin_email: "sem-arroba" })).toContain("admin_email");
    expect(validateProvisionForm({ ...valid, alert_email: "sem-arroba" })).toContain("alert_email");
  });
});
```

- [ ] **Step 2: rodar e ver falhar**

Run: `cd apps/admin && npm test -- provisioning`
Expected: FAIL — `Failed to resolve import "./provisioning"`.

- [ ] **Step 3: escrever a validação**

```ts
// apps/admin/src/lib/provisioning.ts
// Validação do formulário de provisionamento. Espelha ProvisionCity
// (apps/api/app/commands/provision_city.rb) e CityDatabase.valid_slug? — a API
// continua sendo a autoridade; isto só evita um 422 previsível.

export interface ProvisionCityPayload {
  slug: string;
  name: string;
  uf: string;
  ibge_code: string;
  admin_email: string;
  alert_email: string;
}

const SLUG = /^[a-z0-9]([a-z0-9-]*[a-z0-9])?$/;
const RESERVED = [ "admin", "api", "auth", "www" ];
const UF = /^[A-Z]{2}$/;
const IBGE = /^\d{7}$/;
const EMAIL = /^[^@\s]+@[^@\s]+\.[^@\s]+$/;

export function validateProvisionForm(p: ProvisionCityPayload): string[] {
  const bad: string[] = [];
  const slugOk = p.slug.length >= 2 && p.slug.length <= 40 && SLUG.test(p.slug) && !RESERVED.includes(p.slug);
  if (!slugOk) bad.push("slug");
  if (!p.name.trim()) bad.push("name");
  if (!UF.test(p.uf)) bad.push("uf");
  if (!IBGE.test(p.ibge_code)) bad.push("ibge_code");
  if (!EMAIL.test(p.admin_email)) bad.push("admin_email");
  if (!EMAIL.test(p.alert_email)) bad.push("alert_email");
  return bad;
}
```

- [ ] **Step 4: rodar e ver passar**

Run: `cd apps/admin && npm test -- provisioning`
Expected: PASS (6 examples).

- [ ] **Step 5: acrescentar `createCity` ao cliente**

Em `src/lib/api.ts`, na seção de cidades (junto de `listCities`/`createCityGrant`):

```ts
export interface CreateCityResult { id: string }

// POST /cities — PlatformConsoleHost, operador com MFA. Assíncrono: 202 {id} e
// o worker provisiona. O convite do primeiro admin vai por e-mail; o token
// nunca volta para o console (contrato do Plano 4).
export async function createCity(payload: ProvisionCityPayload): Promise<CreateCityResult> {
  return jsonFetch<CreateCityResult>("/cities", {
    method: "POST",
    body: JSON.stringify(payload)
  });
}
```

Importe o tipo de `./provisioning`.

- [ ] **Step 6: escrever a tela**

`src/modules/setup/ProvisionCity.tsx`: formulário com os seis campos, `validateProvisionForm` no submit, `createCity` no envio, e resultado que diz a verdade sobre o que aconteceu:

- sucesso → "Cidade registrada. O provisionamento roda em segundo plano; acompanhe o status em Cidades." com botão que leva ao módulo `cities`;
- `409` → "já existe uma cidade com esse slug";
- `503` → "o servidor de bancos de cidade não está configurado (CITY_DATABASE_HOST)";
- `422` → mostra `message` do corpo.

Reaproveite `ApiError` e o estilo de `Cities.tsx` (`PageHeader`, botões). Invalide a query `["cities"]` do react-query no sucesso, para a lista refletir a cidade nova:

```ts
const qc = useQueryClient();
// ... no sucesso:
await qc.invalidateQueries({ queryKey: [ "cities" ] });
```

- [ ] **Step 7: ligar na navegação**

Em `modules.ts`, acrescente ao grupo "Setup": `{ id: "provision_city", label: "Provisionar cidade", icon: "＋", visible: (u) => u.operator }`. Em `App.tsx`, importe e acrescente `case "provision_city": return <ProvisionCity />;`.

Em `Cities.tsx`, corrija o `EmptyState`, que hoje aponta para a tela removida:

```tsx
<EmptyState title="Nenhuma cidade provisionada" sub="use Setup → Provisionar cidade para criar a primeira" />
```
(o texto já está certo; confirme que o rótulo bate com o `label` do módulo novo.)

- [ ] **Step 8: verificar e commitar**

Run: `cd apps/admin && npm run typecheck && npm test`
Expected: verde, com os 6 exemplos novos.

```bash
cd apps/admin && /opt/homebrew/bin/git add src/lib/provisioning.ts src/lib/provisioning.test.ts src/modules/setup/ProvisionCity.tsx src/lib/api.ts src/shell/modules.ts src/App.tsx src/modules/Cities.tsx
/opt/homebrew/bin/git commit -m "Point the console's provisioning at POST /cities

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 3: canal de WhatsApp da cidade (endpoint + tela)

**Files:**
- Create: `apps/api/app/controllers/operators/city_channels_controller.rb`, `apps/api/spec/requests/operators/city_channels_spec.rb`, `apps/admin/src/modules/setup/RegisterChannel.tsx`
- Modify: `apps/api/config/routes.rb`, `apps/admin/src/lib/api.ts`, `apps/admin/src/shell/modules.ts`, `apps/admin/src/App.tsx`

**Interfaces:**
- Consome: `MunicipalityChannels::Register.call(city:, phone_number_id:, waba_id:, display_phone_number:, access_token:)` → `Result`.
- Produz: `POST /cities/:id/channel` (PlatformConsoleHost); `registerCityChannel` no console.

Por que existe: o canal saiu do provisionamento no Plano 4 ("feito quando a Meta libera o número") e nunca ganhou rota. Sem ele a cidade fica ativa e muda — `Whatsapp::Ingest` casa `phone_number_id` com `CityChannel` e, sem linha, a mensagem vira `UnknownChannel`.

- [ ] **Step 1: escrever o spec (RED)**

```ruby
# apps/api/spec/requests/operators/city_channels_spec.rb
require "rails_helper"

RSpec.describe "Operators::CityChannels", type: :request do
  # Idioma do spec vizinho (spec/requests/operators/cities_spec.rb): NÃO existe
  # factory de operator — o operador é criado na mão —, a sessão do console é
  # login + desafio TOTP, e o host se troca com `host!`, não com header.
  let(:password) { "s3nha-forte-1" }
  let!(:operator) do
    Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end
  let!(:city) { create(:city, status: "active") }

  def json = JSON.parse(response.body)

  def verified_login!
    post "/session", params: { email_address: operator.email_address, password: password }
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(operator.otp_secret).now }
    expect(response).to have_http_status(:ok)
  end

  def register(params)
    post "/cities/#{city.id}/channel", params: params
  end

  before do
    host! "admin.rotasaude.app"
    verified_login!
  end

  it "registers the channel of an active city" do
    register(phone_number_id: "PNID-1", waba_id: "WABA-1",
             display_phone_number: "+551133334444", access_token: "EAAtoken")

    expect(response).to have_http_status(:created)
    expect(JSON.parse(response.body)).to include("phone_number_id" => "PNID-1")
    expect(CityChannel.find_by(city_id: city.id)).to be_present
  end

  # O token do canal é segredo: a resposta não pode devolvê-lo, nem o log
  # carregá-lo (ADR-0012/0013 e a Global Constraint deste plano).
  it "never echoes the access token back" do
    register(phone_number_id: "PNID-2", waba_id: "WABA-2",
             display_phone_number: "+551133334445", access_token: "EAAsegredo")

    expect(response.body.include?("EAAsegredo")).to be(false)
  end

  it "refuses a city that is not servable" do
    city.update!(status: "suspended")
    register(phone_number_id: "PNID-3", waba_id: "WABA-3",
             display_phone_number: "+551133334446", access_token: "EAAtoken")

    expect(response).to have_http_status(:unprocessable_content)
    expect(JSON.parse(response.body)["error"]).to eq("city_not_servable")
  end

  it "refuses an empty access token" do
    register(phone_number_id: "PNID-4", waba_id: "WABA-4",
             display_phone_number: "+551133334447", access_token: "")

    expect(response).to have_http_status(:unprocessable_content)
  end

  it "404s for an unknown city" do
    post "/cities/00000000-0000-0000-0000-000000000000/channel",
         params: { phone_number_id: "x", waba_id: "y", display_phone_number: "+55", access_token: "z" }

    expect(response).to have_http_status(:not_found)
  end
end
```

`unprocessable_content` é o nome novo do 422 (o `unprocessable_entity` está deprecado no Rack desta versão e a suíte já avisa sobre ele).

- [ ] **Step 2: rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/operators/city_channels_spec.rb`
Expected: FAIL — rota inexistente.

- [ ] **Step 3: rota**

Em `config/routes.rb`, dentro de `constraints(PlatformConsoleHost) do ... scope module: :operators`, logo depois de `resources :cities, only: %i[index create show]`:

```ruby
      # Canal do WhatsApp da cidade (Plano 8). O canal mora na PLATAFORMA e é
      # passo à parte do provisionamento (Plano 4): entra quando a Meta libera o
      # número. Sem ele, Whatsapp::Ingest não acha a cidade pelo phone_number_id.
      resources :cities, only: [] do
        resource :channel, only: :create, controller: "city_channels"
      end
```

- [ ] **Step 4: controller**

```ruby
# apps/api/app/controllers/operators/city_channels_controller.rb
# POST /cities/:city_id/channel (Plano 8) — registra o canal WhatsApp de uma
# cidade ativa. O access_token é segredo: entra cifrado (CityChannel#encrypts,
# chave de PLATAFORMA) e NUNCA volta na resposta nem vai para log.
module Operators
  class CityChannelsController < BaseController
    def create
      city = City.find_by(id: params[:city_id].to_s)
      return head(:not_found) unless city

      result = MunicipalityChannels::Register.call(
        city: city,
        phone_number_id: params[:phone_number_id],
        waba_id: params[:waba_id],
        display_phone_number: params[:display_phone_number],
        access_token: params[:access_token]
      )

      if result.ok?
        channel = result.payload[:channel]
        render json: { id: channel.id, phone_number_id: channel.phone_number_id,
                       display_phone_number: channel.display_phone_number, active: channel.active },
               status: :created
      else
        render json: { error: result.reason.to_s, message: result.message }, status: :unprocessable_content
      end
    end
  end
end
```

- [ ] **Step 5: rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/requests/operators/city_channels_spec.rb`
Expected: PASS (5 examples).

- [ ] **Step 6: tela no console**

Em `api.ts`:

```ts
export interface RegisterChannelPayload {
  phone_number_id: string;
  waba_id: string;
  display_phone_number: string;
  access_token: string;
}

export async function registerCityChannel(cityId: string, payload: RegisterChannelPayload) {
  return jsonFetch<{ id: string; phone_number_id: string; display_phone_number: string; active: boolean }>(
    `/cities/${cityId}/channel`, { method: "POST", body: JSON.stringify(payload) }
  );
}
```

`RegisterChannel.tsx`: seleciona a cidade (lista de `useCities()`, só `active`), quatro campos, `access_token` com `type="password"`, e aviso de que o token não é exibido depois. Ligue como módulo `register_channel` em `modules.ts` e `App.tsx`, no grupo "Setup".

- [ ] **Step 7: suíte e commit**

```bash
docker compose stop worker
docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec
docker compose start worker
cd apps/api && /opt/homebrew/bin/git add config/routes.rb app/controllers/operators/city_channels_controller.rb spec/requests/operators/city_channels_spec.rb
/opt/homebrew/bin/git commit -m "Add the console endpoint that registers a city's WhatsApp channel

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
cd ../admin && /opt/homebrew/bin/git add src/lib/api.ts src/modules/setup/RegisterChannel.tsx src/shell/modules.ts src/App.tsx
/opt/homebrew/bin/git commit -m "Add the channel registration screen

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 4: `config.hosts` em produção

**Files:**
- Modify: `apps/api/config/environments/production.rb`
- Test: `apps/api/spec/architecture/host_authorization_spec.rb`

**Interfaces:**
- Consome: `CITY_PUBLIC_BASE_TEMPLATE` (já existe, `https://%{slug}.rota-saude.example` em produção).
- Produz: `config.hosts` populado fora de development.

Hoje `config.hosts` é vazio em produção, e com lista vazia o Rails **pula** o `HostAuthorization`. O impacto é menor desde o Plano 6 (nenhum link deriva de `request.host`), mas é a guarda que impede um Host forjado de chegar ao `CityCatalog`.

- [ ] **Step 1: escrever a guarda (RED)**

A lista fica num módulo próprio (`PlatformHosts`) porque carregar `config/environments/production.rb` dentro do spec para inspecionar `config.hosts` é frágil. O spec testa o módulo direto, e o `production.rb` só consome.

```ruby
# apps/api/spec/architecture/host_authorization_spec.rb
require "rails_helper"

# Spec §5 e Plano 8: fora de development o Host precisa casar com o domínio da
# plataforma. Lista VAZIA faz o Rails PULAR o middleware inteiro — por isso a
# guarda é sobre a lista, e em test ela continua vazia de propósito (o harness
# usa hosts sintéticos como testcitya.rotasaude.app).
RSpec.describe PlatformHosts do
  # Mesmo idioma de stub do spec/requests/cors_spec.rb: o template público é a
  # única fonte do domínio, então é dele que os hosts derivam.
  def with_production_template
    allow(ENV).to receive(:fetch).and_call_original
    allow(ENV).to receive(:fetch).with("CITY_PUBLIC_BASE_TEMPLATE", anything)
      .and_return("https://%{slug}.rota-saude.example")
    yield
  end

  it "declares the platform domain and the city wildcard in production" do
    with_production_template do
      expect(described_class.for("production")).to eq([ "rota-saude.example", ".rota-saude.example" ])
    end
  end

  it "stays empty outside production, where the harness uses synthetic hosts" do
    with_production_template do
      expect(described_class.for("test")).to eq([])
      expect(described_class.for("development")).to eq([])
    end
  end
end
```

- [ ] **Step 2: rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/architecture/host_authorization_spec.rb`
Expected: FAIL.

- [ ] **Step 3: implementar**

```ruby
# apps/api/app/services/platform_hosts.rb
# Hosts aceitos pelo ActionDispatch::HostAuthorization (Plano 8). Derivam do
# mesmo template público das cidades (CITY_PUBLIC_BASE_TEMPLATE), para não
# existir uma segunda fonte de verdade do domínio.
#
# Lista VAZIA desliga o middleware: é por isso que produção precisa declarar.
module PlatformHosts
  module_function

  def for(env)
    return [] unless env.to_s == "production"

    domain = URI.parse(CityPublicUrl.base_for_slug("x")).host.to_s.delete_prefix("x.")
    return [] if domain.blank?

    [ domain, ".#{domain}" ]
  end
end
```

Em `production.rb`, antes do `end`:

```ruby
  # Plano 8: sem isto a lista fica vazia e o Rails PULA o HostAuthorization.
  # `.dominio` cobre api./admin./auth. e o curinga das cidades.
  config.hosts += PlatformHosts.for("production")
  config.host_authorization = { exclude: ->(request) { request.path == "/up" } }
```

O `exclude` do healthcheck é necessário: o kamal-proxy bate em `/up` pelo IP do container, sem Host do domínio.

- [ ] **Step 4: rodar e ver passar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/architecture/host_authorization_spec.rb`
Expected: PASS.

- [ ] **Step 5: suíte e commit**

```bash
docker compose stop worker && docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec; docker compose start worker
cd apps/api && /opt/homebrew/bin/git add app/services/platform_hosts.rb config/environments/production.rb spec/architecture/host_authorization_spec.rb
/opt/homebrew/bin/git commit -m "Declare config.hosts in production

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 5: `CONNECTION LIMIT` por role de cidade

**Files:**
- Modify: `apps/api/app/services/city_database.rb`, `apps/api/README.md`
- Test: `apps/api/spec/services/city_database_spec.rb`

O orçamento de conexões está calculado no README (~280 contra `max_connections` 100) e não é aplicado em lugar nenhum. Um role sem limite deixa uma cidade consumir o pool inteiro e derrubar as vizinhas.

- [ ] **Step 1: escrever o spec (RED)**

Acrescente a `spec/services/city_database_spec.rb`, que já roda com `use_transactional_tests = false`, já apaga `slug_a`/`slug_b` no `after` e já tem o helper `superuser_value(sql, *params)` (linha 26, sobre `ScratchDatabases.superuser`) — use ESSE helper, não crie outro:

```ruby
  it "caps how many connections a city's role can open" do
    described_class.ensure!(slug: slug_a, password: pwd_a)

    limit = superuser_value("SELECT rolconnlimit FROM pg_roles WHERE rolname = $1", described_class.role_name(slug_a))

    expect(limit.to_i).to eq(described_class::ROLE_CONNECTION_LIMIT)
  end

  # ensure! é idempotente e realinha a senha; o teto tem de seguir a mesma
  # regra, senão um role criado antes deste plano ficaria sem limite para
  # sempre.
  it "re-applies the cap when the role already exists without one" do
    described_class.ensure!(slug: slug_a, password: pwd_a)
    ScratchDatabases.superuser do |conn|
      conn.exec("ALTER ROLE #{PG::Connection.quote_ident(described_class.role_name(slug_a))} CONNECTION LIMIT -1")
    end

    described_class.ensure!(slug: slug_a, password: pwd_a)

    limit = superuser_value("SELECT rolconnlimit FROM pg_roles WHERE rolname = $1", described_class.role_name(slug_a))
    expect(limit.to_i).to eq(described_class::ROLE_CONNECTION_LIMIT)
  end
```

- [ ] **Step 2: rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/services/city_database_spec.rb`
Expected: FAIL — `ROLE_CONNECTION_LIMIT` não existe e `rolconnlimit` é `-1`.

- [ ] **Step 3: implementar**

Em `CityDatabase`, junto das outras constantes:

```ruby
  # Teto de conexões por role de cidade (Plano 8). O orçamento do README dá ~80
  # conexões de worker + a fatia de web por cidade; 100 deixa folga para uma
  # rake de manutenção sem permitir que UMA cidade esgote o servidor e derrube
  # as vizinhas. Configurável para quem rodar com max_connections maior.
  ROLE_CONNECTION_LIMIT = ENV.fetch("CITY_ROLE_CONNECTION_LIMIT", "100").to_i
```

E em `ensure!`, na mesma transação de comandos, trocando a linha do `CREATE/ALTER ROLE`:

```ruby
        conn.exec("#{verb} ROLE #{quote(role)} WITH LOGIN PASSWORD #{secret} CONNECTION LIMIT #{ROLE_CONNECTION_LIMIT}")
```

`CONNECTION LIMIT` é válido tanto em `CREATE ROLE` quanto em `ALTER ROLE`, então o idempotente continua realinhando o teto de quem já existe.

- [ ] **Step 4: rodar e ver passar**

Run: mesmo comando do Step 2. Expected: PASS.

- [ ] **Step 5: README**

Na seção do orçamento de conexões, troque o "Gate de go-live" por o que passou a ser verdade: o teto por role agora é aplicado no `ensure!` (`CITY_ROLE_CONNECTION_LIMIT`, default 100), e o que **resta** é dimensionar o `max_connections` do acessório — que é a Task 10.

- [ ] **Step 6: suíte e commit**

```bash
docker compose stop worker && docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec; docker compose start worker
cd apps/api && /opt/homebrew/bin/git add app/services/city_database.rb spec/services/city_database_spec.rb README.md
/opt/homebrew/bin/git commit -m "Cap connections per city role

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 6: chave de assinatura de relatório por cidade

**Files:**
- Modify: `apps/api/app/services/city_encryption.rb`, `apps/api/app/models/report_snapshot.rb`, `apps/api/lib/tasks/city.rake`
- Create: `apps/api/app/commands/city_reports/resign.rb`, `apps/api/spec/commands/city_reports/resign_spec.rb`
- Test: `apps/api/spec/models/report_snapshot_spec.rb`

Promessa do §6 do spec que o Plano 7 não entregou. Não há risco prático de cruzamento (a busca é `find_by(token:)` dentro do banco da cidade), mas a custódia fica coerente: uma chave por cidade, derivada, como a de cifra.

- [ ] **Step 1: escrever os specs (RED)**

```ruby
# apps/api/spec/models/report_snapshot_spec.rb (acrescentar)
  # Duas cidades porque isolamento não se prova com uma (spec, Verification).
  let!(:city_a) { create(:city, slug: TEST_CITY_A.slug, database_url: city_database_url("rota_saude_test_city_a")) }
  let!(:city_b) { create(:city, slug: TEST_CITY_B.slug, database_url: city_database_url("rota_saude_test_city_b")) }

  # Snapshot mínimo dentro da cidade corrente. `create_snapshot` do
  # spec/requests/reports_spec.rb NÃO está disponível aqui: é local daquele
  # arquivo. Este helper é o equivalente enxuto.
  def create_snapshot
    pd = ProtocolDefinition.create!(
      name: "triagem-sign", version: 1, status: "active",
      definition: { "name" => "triagem-sign", "version" => 1, "start_step_id" => "s1",
                    "steps" => [ { "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
                                   "branches" => { "true" => nil, "false" => nil } } ] }
    )
    convo = Conversation.create!(phone: "+5541977776666", state: "greeting")
    triage = Triage.create!(conversation: convo, protocol_definition: pd, protocol_name: "triagem-sign",
                            status: "completed", tier: "alta", priority: 1,
                            completed_at: Time.current, outcome: { "trail" => [] })
    token = ReportSnapshot.mint_token
    ReportSnapshot.create!(triage: triage, protocol_definition: pd, outcome: { "tier" => "alta" },
                           payload: { "tier" => "alta" }, token: token,
                           signature: ReportSnapshot.sign(token), expires_at: 30.days.from_now)
  end

  describe "signing" do
    # Digests: comparar assinatura crua imprimiria material no diff de falha.
    def digest(value) = Digest::SHA256.hexdigest(value)

    it "signs with a key derived from the city" do
      token = ReportSnapshot.mint_token
      a = CityConnection.with(city_a) { ReportSnapshot.sign(token) }
      b = CityConnection.with(city_b) { ReportSnapshot.sign(token) }

      expect(digest(a)).not_to eq(digest(b))
    end

    it "verifies a signature minted in the same city" do
      snap = CityConnection.with(city_a) { create_snapshot }
      found = CityConnection.with(city_a) { ReportSnapshot.find_by_signed_token(snap.token) }

      expect(found&.id).to eq(snap.id)
    end

    # Transição (decisão 5 do plano): assinatura gravada com a chave global
    # antes do Plano 8 continua verificando até a rake reescrever.
    it "still verifies a legacy signature during the transition window" do
      snap = CityConnection.with(city_a) { create_snapshot }
      legacy = OpenSSL::HMAC.hexdigest("sha256", Rails.application.credentials.fetch(:report_signing_key), snap.token)
      CityConnection.with(city_a) { snap.update_column(:signature, legacy) }

      found = CityConnection.with(city_a) { ReportSnapshot.find_by_signed_token(snap.token) }
      expect(found&.id).to eq(snap.id)
    end

    it "refuses a signature that is neither the city's nor the legacy one" do
      snap = CityConnection.with(city_a) { create_snapshot }
      CityConnection.with(city_a) { snap.update_column(:signature, "deadbeef") }

      expect(CityConnection.with(city_a) { ReportSnapshot.find_by_signed_token(snap.token) }).to be_nil
    end
  end
```

- [ ] **Step 2: rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/models/report_snapshot_spec.rb`
Expected: FAIL — hoje a assinatura não depende da cidade.

- [ ] **Step 3: derivar a chave**

Em `CityEncryption`, método público novo (os de material cru continuam privados):

```ruby
  # Chave HMAC do token de relatório, por cidade (Plano 8, spec §6). Derivada da
  # chave global de assinatura + material da cidade: a custódia é a mesma da
  # chave de cifra, e um dump de A não permite forjar token de B.
  def report_signing_key(city)
    material = city.respond_to?(:encryption_key) ? city.encryption_key.to_s : ""
    raise MissingKey, "cidade sem encryption_key: não há chave a derivar" if material.blank?

    OpenSSL::HMAC.digest("sha256", legacy_report_signing_key, "report-signing:#{material}")
  end

  def legacy_report_signing_key = Rails.application.credentials.fetch(:report_signing_key)
```

`legacy_report_signing_key` fica **pública** de propósito: `ReportSnapshot` precisa dela para o fallback da transição. Não devolve material derivado de cidade.

Em `ReportSnapshot`:

```ruby
  def self.find_by_signed_token(token)
    record = find_by(token: token)
    return nil unless record
    return nil unless signature_matches?(record, token)
    return nil if record.expires_at && record.expires_at < Time.current
    record
  end

  def self.sign(token)
    OpenSSL::HMAC.hexdigest("sha256", CityEncryption.report_signing_key(Current.city), token)
  end

  # Transição do Plano 8: assinaturas gravadas antes da chave por cidade usam a
  # chave global. A rake city:resign_reports[slug] reescreve as existentes; este
  # fallback pode sair depois de 30 dias (GenerateReportJob::EXPIRATION), quando
  # todo snapshot vivo já tiver nascido com a chave da cidade.
  def self.signature_matches?(record, token)
    return true if ActiveSupport::SecurityUtils.secure_compare(record.signature, sign(token))

    legacy = OpenSSL::HMAC.hexdigest("sha256", CityEncryption.legacy_report_signing_key, token)
    ActiveSupport::SecurityUtils.secure_compare(record.signature, legacy)
  end
  private_class_method :signature_matches?
```

`sign` passa a exigir `Current.city` — como `url` já exige. `lib/dashboard_demo.rb:200` roda dentro da cidade; confirme e ajuste se não rodar.

- [ ] **Step 4: rodar e ver passar**

Run: mesmo comando do Step 2. Expected: PASS.

- [ ] **Step 5: rake que reescreve as assinaturas existentes**

```ruby
# apps/api/app/commands/city_reports/resign.rb
# Reescreve as assinaturas de relatório de UMA cidade com a chave derivada dela
# (Plano 8). Roda dentro de CityConnection.with: Current.city governa a chave.
#
# ApplicationRecord.transaction, NÃO ActiveRecord::Base.transaction — dentro de
# connected_to_many esta última abre na conexão `primary`, que é o banco vazio
# rota_saude_no_city_selected (mesmo tropeço do Plano 7, Task 4).
#
# record_timestamps = false: reassinar não é mudança de domínio, e
# sweep_abandoned_conversations_job e a query de overview selecionam por
# updated_at.
module CityReports
  module Resign
    def self.call
      count = 0

      ApplicationRecord.transaction do
        ReportSnapshot.live.find_each do |snapshot|
          signature = ReportSnapshot.sign(snapshot.token)
          next if ActiveSupport::SecurityUtils.secure_compare(snapshot.signature, signature)

          snapshot.record_timestamps = false
          snapshot.update_columns(signature: signature)
          count += 1
        end
      end

      Result.ok(count: count)
    rescue CityEncryption::MissingKey => e
      Result.fail(:missing_key, message: e.message)
    end
  end
end
```

Rake em `city.rake`, junto das de ciclo de vida:

```ruby
  desc "Reescreve as assinaturas de relatório de uma cidade com a chave dela. Uso: city:resign_reports[slug]"
  task :resign_reports, %i[slug] => :environment do |_t, args|
    city = lifecycle_city.call("city:resign_reports", args[:slug])
    result = CityConnection.with(city) { CityReports::Resign.call }
    abort "[city:resign_reports] #{result.reason}: #{result.message}" if result.failure?
    puts "[city:resign_reports] #{city.slug} → #{result.payload[:count]} assinatura(s)"
  end
```

```ruby
# apps/api/spec/commands/city_reports/resign_spec.rb
require "rails_helper"

RSpec.describe CityReports::Resign do
  let!(:city) { create(:city, database_url: city_database_url("rota_saude_test_city_a")) }

  def legacy_signature(token)
    OpenSSL::HMAC.hexdigest("sha256", CityEncryption.legacy_report_signing_key, token)
  end

  # Snapshot com assinatura LEGADA, como as linhas gravadas antes deste plano.
  def snapshot_with_legacy_signature
    CityConnection.with(city) do
      pd = ProtocolDefinition.create!(
        name: "triagem-resign", version: 1, status: "active",
        definition: { "name" => "triagem-resign", "version" => 1, "start_step_id" => "s1",
                      "steps" => [ { "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
                                     "branches" => { "true" => nil, "false" => nil } } ] }
      )
      convo = Conversation.create!(phone: "+5541988887777", state: "greeting")
      triage = Triage.create!(conversation: convo, protocol_definition: pd, protocol_name: "triagem-resign",
                              status: "completed", tier: "alta", priority: 1,
                              completed_at: Time.current, outcome: { "trail" => [] })
      token = ReportSnapshot.mint_token
      ReportSnapshot.create!(triage: triage, protocol_definition: pd, outcome: { "tier" => "alta" },
                             payload: { "tier" => "alta" }, token: token,
                             signature: legacy_signature(token), expires_at: 30.days.from_now)
    end
  end

  it "rewrites a legacy signature with the city's own key" do
    snapshot = snapshot_with_legacy_signature

    result = CityConnection.with(city) { described_class.call }

    expect(result.ok?).to be(true)
    expect(result.payload[:count]).to eq(1)
    reloaded = CityConnection.with(city) { ReportSnapshot.find(snapshot.id) }
    expect(Digest::SHA256.hexdigest(reloaded.signature))
      .not_to eq(Digest::SHA256.hexdigest(legacy_signature(snapshot.token)))
  end

  it "leaves the token verifiable afterwards" do
    snapshot = snapshot_with_legacy_signature
    CityConnection.with(city) { described_class.call }

    found = CityConnection.with(city) { ReportSnapshot.find_by_signed_token(snapshot.token) }
    expect(found&.id).to eq(snapshot.id)
  end

  it "does not bump updated_at" do
    snapshot = snapshot_with_legacy_signature
    before = snapshot.updated_at

    travel(1.hour) { CityConnection.with(city) { described_class.call } }

    reloaded = CityConnection.with(city) { ReportSnapshot.find(snapshot.id) }
    expect(reloaded.updated_at).to eq(before)
  end

  it "counts only what it changed, so a second run is a no-op" do
    snapshot_with_legacy_signature
    CityConnection.with(city) { described_class.call }

    second = CityConnection.with(city) { described_class.call }
    expect(second.payload[:count]).to eq(0)
  end
end
```

- [ ] **Step 6: rodar nas cidades de dev**

```bash
docker compose exec -T api bin/rails 'city:resign_reports[curitiba]'
docker compose exec -T api bin/rails 'city:resign_reports[maringa]'
```
Registre as contagens. Não imprima token nem assinatura.

- [ ] **Step 7: suíte e commit**

```bash
docker compose stop worker && docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec; docker compose start worker
cd apps/api && /opt/homebrew/bin/git add app/services/city_encryption.rb app/models/report_snapshot.rb app/commands/city_reports/resign.rb lib/tasks/city.rake spec/
/opt/homebrew/bin/git commit -m "Derive the report signing key per city

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 7: `city:restore`

**Files:**
- Create: `apps/api/app/commands/city_lifecycle/restore.rb`, `apps/api/spec/commands/city_lifecycle/restore_spec.rb`
- Modify: `apps/api/lib/tasks/city.rake`, `apps/api/README.md`

O inverso de `CityLifecycle::Backup`. A armadilha que o README já documenta: restaurar um dump numa cidade cujo `encryption_key` mudou devolve dado **ilegível sem erro**. O comando tem de recusar isso, não avisar depois.

- [ ] **Step 1: gravar o digest da chave junto do dump**

Em `CityLifecycle::Backup`, logo depois do `File.chmod(0o600, path)`:

```ruby
      # Plano 8: o digest do material da cidade no momento do dump. É o que
      # permite ao city:restore RECUSAR um dump de outra época — sem ele, a
      # restauração devolve dado ilegível sem erro nenhum.
      digest_path = "#{path}.key-digest"
      File.write(digest_path, Digest::SHA256.hexdigest(city.encryption_key), perm: 0o600)
```

E devolva `Result.ok(path: path, key_digest_path: digest_path)`.

- [ ] **Step 2: escrever o spec (RED)**

```ruby
# apps/api/spec/commands/city_lifecycle/restore_spec.rb
require "rails_helper"
require "tmpdir"

# Restore é o inverso do Backup (spec §4). A armadilha que ele existe para
# fechar: dump restaurado numa cidade cujo encryption_key mudou devolve dado
# ILEGÍVEL SEM ERRO — por isso a guarda de digest vem ANTES de tocar o banco.
RSpec.describe CityLifecycle::Restore do
  self.use_transactional_tests = false

  let!(:city) { provision_city!(status: "active") }
  let(:dir) { Dir.mktmpdir("city-restore") }

  after do
    cleanup_provisioned_city!(city)
  ensure
    FileUtils.rm_rf(dir)
  end

  # ATENÇÃO ao que a guarda realmente lê: SuspensionGuard.suspended_recently?
  # consulta o `occurred_at` do PlatformEvent "city.suspended" desta cidade —
  # NÃO o updated_at da City. Envelhecer o campo errado faria o exemplo de
  # recusa nunca ficar vermelho e o de sucesso passar pelo motivo errado.
  def suspend_at!(moment)
    PlatformEvent.where(name: "city.suspended").where("payload->>'city_id' = ?", city.id).delete_all
    PlatformEvent.create!(name: "city.suspended", occurred_at: moment, payload: { "city_id" => city.id })
    city.update!(status: "suspended")
  end

  # Dump de uma cidade com uma linha conhecida, já suspensa e FORA da
  # quarentena (o Restore exige as duas coisas).
  def dump_with_one_user!
    CityConnection.with(city) { User.create!(email_address: "servidora@cidade.gov.br", password: "secret123") }
    path = CityLifecycle::Backup.call(city: city, dir: dir).payload[:path]
    suspend_at!((CityLifecycle::SuspensionGuard::QUIET_PERIOD + 60.seconds).ago)
    path
  end

  it "restores a dump into its own city" do
    path = dump_with_one_user!
    CityConnection.with(city) { User.delete_all }

    result = described_class.call(city: city, path: path)

    expect(result.ok?).to be(true)
    emails = CityConnection.with(city) { User.pluck(:email_address) }
    expect(emails).to eq([ "servidora@cidade.gov.br" ])
  end

  it "refuses a city that is not suspended" do
    path = dump_with_one_user!
    city.update!(status: "active")

    result = described_class.call(city: city, path: path)

    expect(result.failure?).to be(true)
    expect(result.reason).to eq(:invalid_status)
  end

  it "refuses while the suspension is still within the quiet period" do
    path = dump_with_one_user!
    suspend_at!(Time.current)

    expect(described_class.call(city: city, path: path).reason).to eq(:suspension_too_recent)
  end

  it "refuses a dump taken under different key material, before touching the database" do
    path = dump_with_one_user!
    File.write("#{path}.key-digest", Digest::SHA256.hexdigest("outro material"))

    result = described_class.call(city: city, path: path)

    expect(result.reason).to eq(:key_mismatch)
    # Nada foi apagado: o banco continua com a linha do dump original.
    expect(CityConnection.with(city) { User.count }).to eq(1)
  end

  it "refuses a file that does not exist" do
    dump_with_one_user!
    expect(described_class.call(city: city, path: File.join(dir, "nao-existe.dump")).reason).to eq(:missing_file)
  end
end
```

- [ ] **Step 3: rodar e ver falhar**

Run: `docker compose exec -T -e POSTGRES_PASSWORD=postgres api bundle exec rspec spec/commands/city_lifecycle/restore_spec.rb`
Expected: FAIL — `uninitialized constant CityLifecycle::Restore`.

- [ ] **Step 4: implementar**

```ruby
# apps/api/app/commands/city_lifecycle/restore.rb
require "open3"

# Restaura um dump de cidade (Plano 8), o inverso de Backup.
#
# Ordem das guardas, e por quê: status e quarentena primeiro (outro processo
# pode seguir servindo a cidade por até 2× o TTL do CityCatalog, mesmo depois
# do suspend), arquivo depois, e o DIGEST DA CHAVE por último — mas ainda antes
# de qualquer DDL. Restaurar dump de outra época devolve dado ilegível SEM
# erro: essa é a única guarda que transforma o engano em falha visível.
module CityLifecycle
  module Restore
    def self.call(city:, path:)
      unless city.status == "suspended"
        return Result.fail(:invalid_status, message: "cidade #{city.slug} precisa estar suspensa (status=#{city.status})")
      end
      if SuspensionGuard.suspended_recently?(city)
        return Result.fail(:suspension_too_recent,
                           message: "aguarde #{SuspensionGuard::QUIET_PERIOD.to_i} s depois da suspensão")
      end
      return Result.fail(:missing_file, message: "dump não encontrado: #{File.basename(path.to_s)}") unless File.file?(path.to_s)

      mismatch = key_digest_mismatch(city, path)
      return mismatch if mismatch

      url = URI.parse(city.database_url)
      env = { "PGPASSWORD" => URI::DEFAULT_PARSER.unescape(url.password.to_s) }
      sslmode = URI.decode_www_form(url.query.to_s).to_h["sslmode"]
      env["PGSSLMODE"] = sslmode if sslmode.present?

      CityConnection.forget(city.shard)

      out, status = Open3.capture2e(
        env,
        "pg_restore", "--clean", "--if-exists", "--no-owner", "--no-acl",
        "--host", url.host.to_s, "--port", (url.port || 5432).to_s,
        "--username", URI::DEFAULT_PARSER.unescape(url.user.to_s),
        "--dbname", url.path.delete_prefix("/"), path.to_s
      )
      unless status.success?
        return Result.fail(:restore_failed, message: CitySchema.redact(out.lines.last(3).join).strip)
      end

      Platform.audit("city.restored", city_id: city.id, file: File.basename(path.to_s))
      Result.ok(path: path.to_s)
    end

    # Dump anterior ao Plano 8 não tem o arquivo irmão: não dá para provar nada,
    # e recusar impediria restaurar backup legítimo. Passa, e o runbook manda
    # conferir na mão.
    def self.key_digest_mismatch(city, path)
      digest_path = "#{path}.key-digest"
      return nil unless File.file?(digest_path)

      recorded = File.read(digest_path).strip
      return nil if ActiveSupport::SecurityUtils.secure_compare(recorded, Digest::SHA256.hexdigest(city.encryption_key))

      Result.fail(:key_mismatch,
                  message: "o dump foi tirado com outro material de cifra desta cidade — restaurar devolveria " \
                           "dado ilegível sem erro. Confirme a chave da época antes de seguir.")
    end
    private_class_method :key_digest_mismatch
  end
end
```

- [ ] **Step 5: rodar e ver passar**

Run: mesmo comando do Step 3. Expected: PASS (5 examples).

- [ ] **Step 6: rake**

```ruby
  desc "Restaura um dump numa cidade suspensa. Uso: city:restore[slug,caminho]"
  task :restore, %i[slug path] => :environment do |_t, args|
    city = lifecycle_city.call("city:restore", args[:slug])
    abort "uso: rails 'city:restore[slug,/caminho/do.dump]'" if args[:path].blank?

    result = CityLifecycle::Restore.call(city: city, path: args[:path])
    abort "[city:restore] #{result.reason}: #{result.message}" if result.failure?
    puts "[city:restore] #{city.slug} ← #{File.basename(args[:path])}"
  end
```

- [ ] **Step 7: README**

Substitua o parágrafo "Não existe `city:restore` ainda" pelo procedimento real, mantendo o aviso: a guarda de digest recusa dump de outra época, mas **só se o arquivo irmão existir** — dump tirado antes do Plano 8 não tem, e nesse caso a conferência é humana.

- [ ] **Step 8: suíte e commit**

---

## Task 8: CI nos quatro repositórios

**Files:**
- Create: `apps/api/.github/workflows/ci.yml`, `apps/admin/.github/workflows/ci.yml`, `apps/dashboard/.github/workflows/ci.yml`, `apps/wpda/.github/workflows/ci.yml`

Nenhum repositório tem CI. Sem ele, as guardas de arquitetura e os specs de isolamento não protegem merge nenhum, e a validação em **PG 16** (que a suíte local, em PG 15, nunca exercita) não acontece em lugar nenhum.

- [ ] **Step 1: workflow da API**

```yaml
# apps/api/.github/workflows/ci.yml
name: CI
on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  rspec:
    runs-on: ubuntu-latest
    # postgres:16 de propósito: é a versão de PRODUÇÃO (deploy/production/deploy.yml).
    # A suíte local roda no Postgres do host (15), então o caminho sensível a
    # versão — CREATEROLE sem herança implícita, resolvido pelo GRANT explícito
    # em CityDatabase#ensure! — nunca é exercitado fora daqui.
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: rota_saude
          POSTGRES_PASSWORD: postgres
        ports: [ "5432:5432" ]
        options: >-
          --health-cmd "pg_isready -U rota_saude"
          --health-interval 5s --health-timeout 5s --health-retries 10
    env:
      RAILS_ENV: test
      DATABASE_HOST: 127.0.0.1
      DATABASE_PORT: "5432"
      POSTGRES_PASSWORD: postgres
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - name: Bancos de plataforma e de cidade
        run: |
          bin/rails platform:bootstrap
          bin/rails db:migrate:platform
          bin/rails city:test_databases
      - run: bundle exec rspec
```

**Antes de commitar, verifique e ajuste:** em dev os roles (`rota_provisioner`, `rota_app`, `rota_platform`) nascem no `start.sh`, que a CI não roda. Confira quais deles o `platform:bootstrap` cria por conta própria e acrescente ao passo o `psql` que faltar — rode o workflow e leia o erro real em vez de adivinhar.

- [ ] **Step 2: workflow dos três SPAs (o mesmo arquivo nos três)**

```yaml
# apps/<app>/.github/workflows/ci.yml
name: CI
on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  vitest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm
      - run: npm ci
      - run: npm run typecheck
      - run: npm test
      - run: npm run build
```

- [ ] **Step 3: BLOQUEADO (dono: usuário) — `apps/admin` não tem remote**

O workflow do admin não roda em lugar nenhum enquanto o repositório não existir no GitHub. Deixe o arquivo commitado localmente e **relate**: criar `rotasaude/admin` e empurrar é decisão e acesso do usuário.

- [ ] **Step 4: commits (um por repositório, caminhos explícitos)**

---

## Task 9: servir os três SPAs em produção

**Files:**
- Create: `apps/admin/Dockerfile`, `apps/admin/nginx.conf`, e os equivalentes em `apps/dashboard` e `apps/wpda`
- Modify: `apps/api/deploy/production/deploy.yml`

Hoje o proxy publica `api.`, `admin.`, `auth.` e `*.` apontando para o Rails, que só tem API: `admin.<domínio>/admin/` e `<slug>.<domínio>/dashboard/` não têm nada atrás.

- [ ] **Step 1: imagem por SPA**

O mesmo par de arquivos nos três, mudando só o `base` (`/admin/`, `/dashboard/`, `/wpda/`). Exemplo do `admin`:

```dockerfile
# apps/admin/Dockerfile
# SPA servida por nginx. O base do Vite (/admin/) é o caminho público, então o
# conteúdo vai para o mesmo caminho dentro do container.
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html/admin
```

```nginx
# apps/admin/nginx.conf
server {
  listen 80;
  root /usr/share/nginx/html;

  gzip on;
  gzip_types text/css application/javascript application/json image/svg+xml;

  # Assets com hash no nome podem ser cacheados para sempre; o index.html não,
  # senão um deploy novo continua servindo o bundle velho.
  location /admin/assets/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
  }

  location = /admin/index.html {
    add_header Cache-Control "no-cache";
  }

  # Rota de SPA: qualquer caminho sob /admin/ cai no index.
  location /admin/ {
    try_files $uri $uri/ /admin/index.html;
  }

  location = /up {
    access_log off;
    return 200 "ok";
  }
}
```

Para `dashboard` e `wpda`, troque `/admin/` por `/dashboard/` e `/wpda/` nos quatro lugares (o `COPY --from=build`, os dois `location` de cache e o `try_files`).

- [ ] **Step 2: publicar pelo proxy**

Em `deploy/production/deploy.yml`, acrescente os três como `accessories` com o host correspondente, e mova para eles os caminhos `/admin/`, `/dashboard/` e `/wpda/`. **Atenção:** o kamal-proxy roteia por host, não por caminho; se o roteamento por caminho não for viável na versão em uso, a alternativa é um nginx de borda por host. Verifique antes de escrever e relate o que encontrou.

- [ ] **Step 3: BLOQUEADO (dono: usuário) — registro de imagens**

Publicar as três imagens exige credencial no `ghcr.io` (o `registry` do Kamal). Deixe pronto e relate.

- [ ] **Step 4: README**

Substitua "servir os frontends em produção continua em aberto" pelo que passou a existir.

---

## Task 10: TLS, certificado curinga e orçamento do Postgres

**Files:**
- Modify: `apps/api/deploy/production/deploy.yml`, `apps/api/deploy/development/deploy.yml`, `apps/api/Dockerfile`, `apps/api/README.md`, `apps/api/deploy/SECRETS.md`

Três gates operacionais que hoje **quebram ou degradam o primeiro deploy**.

- [ ] **Step 1: certificado curinga — BLOQUEADO (dono: usuário)**

Com `ssl: true` e `"*.rota-saude.example"` em `proxy.hosts`, o kamal-proxy tenta HTTP-01 para o curinga a cada deploy e o Let's Encrypt **não emite curinga por HTTP-01**: o deploy falha com erro de ACME. As duas saídas já estão no README. Documente qual foi escolhida, deixe o `deploy.yml` pronto para ela, e **pare**: emitir por DNS-01 exige acesso ao provedor de DNS.

- [ ] **Step 2: `max_connections` do acessório**

Em `deploy/production/deploy.yml`, no acessório `postgres`, entre `image:` e `host:`:

```yaml
    # Plano 8. Orçamento do README: ~120 conexões de web (2 hosts × 2 Puma × 5
    # threads × pools) + ~80 por cidade ativa de worker. O default do Postgres é
    # 100, que não fecha nem com duas cidades. 500 cobre ~4 cidades com folga;
    # recalcule pelo README antes de passar disso, e lembre que cada role de
    # cidade tem CITY_ROLE_CONNECTION_LIMIT (default 100) como teto próprio.
    cmd: postgres -c max_connections=500
```

**Não tente verificar a chave aqui: não dá.** O Kamal não está instalado nesta máquina — não aparece no `Gemfile` nem no `Gemfile.lock`, não há diretório `.kamal/`, e o binário não está no PATH. O deploy é dirigido de outro lugar. O único precedente do próprio repositório é `cmd:`, usado no **papel** `worker` (`deploy/*/deploy.yml:20,23`) — mas papel e acessório têm esquemas diferentes no Kamal, então isso é indício, não prova.

Escreva a linha como `cmd:` com um comentário dizendo que a chave precisa ser confirmada contra a versão de Kamal que roda o deploy, e trate tanto a confirmação quanto a aplicação como **BLOQUEADO (dono: usuário)** — aplicar exige reboot do acessório de qualquer forma.

- [ ] **Step 3: `sslmode=verify-full`**

O Dockerfile já instala `ca-certificates`, então falta só o lado do servidor. Deixe os dois `deploy/*/deploy.yml` prontos, com a troca comentada e o procedimento ao lado:

```yaml
    # Plano 8. `require` cifra mas NÃO verifica a identidade do servidor.
    # Para `verify-full`, nesta ordem:
    #   1. emitir certificado para o host do Postgres (db.rota-saude.example) por
    #      uma CA que a imagem conheça (as do ca-certificates servem);
    #   2. configurar ssl_cert_file/ssl_key_file no acessório postgres;
    #   3. só então trocar a linha abaixo e fazer deploy.
    # Trocar ANTES do passo 2 derruba TODA conexão de cidade: a aplicação para.
    CITY_DATABASE_SSLMODE: "require"   # → "verify-full" depois dos passos acima
```

**BLOQUEADO (dono: usuário)** nos passos 1 e 2: exigem acesso ao servidor e à CA.

- [ ] **Step 4: SECRETS.md**

Acrescente o item de custódia que falta: `report_signing_key` agora deriva por cidade (Task 6) — quem restaura um dump precisa da chave global **e** do `cities.encryption_key` da época.

---

## Task 11: fechar a documentação e corrigir a spec

**Files:**
- Modify: `apps/api/README.md`, `docs/superpowers/specs/2026-09-12-banco-por-cidade-design.md`

- [ ] **Step 1: corrigir a spec**

§4 diz "Backup é `pg_dump` por cidade, **restaurável sozinho**". Desde o Plano 7 isso é **falso**: o dump carrega ciphertext e exige a chave da cidade da época. Corrija a frase e aponte para o runbook.

- [ ] **Step 2: registrar o que este plano NÃO fez**

No README, uma lista curta e honesta: rotação da chave de plataforma segue sem procedimento válido; o rekey não tem progresso nem retomada; a guarda de quarentena vive na camada das rakes e não é herdada por um futuro chamador não-rake de `CityRekey.call`.

- [ ] **Step 3: commit**

---

## Definition of Done

- [ ] O console não tem nenhuma chamada a rota inexistente ou city-scoped; `grep` por `/setup/municipalities` e por membership no `apps/admin/src` volta vazio.
- [ ] Provisionar uma cidade pelo console leva a cidade a `active` em dev, com convite enviado por e-mail e sem token na resposta.
- [ ] Registrar canal pelo console faz `Whatsapp::Ingest` rotear mensagem para a cidade.
- [ ] `config.hosts` populado em produção, com `/up` excluído.
- [ ] Role de cidade nasce e é realinhado com `CONNECTION LIMIT`.
- [ ] Token de relatório assina e verifica por cidade; assinatura legada ainda verifica; cidades de dev reescritas.
- [ ] `city:restore` existe, recusa cidade não suspensa, dentro da quarentena, e dump de chave divergente.
- [ ] CI roda a suíte da API em **PG 16** e os três SPAs.
- [ ] As três SPAs têm imagem e são publicadas pelo proxy (ou o bloqueio está documentado com dono).
- [ ] Nenhuma saída contém segredo.
- [ ] Suíte completa verde.

## NÃO faz

- **Reconstruir membership no console** — decisão 2 do usuário: entra na cidade pelo grant.
- **Procedimento de rotação da chave de plataforma** — precisa de desenho próprio (hoje o `SECRETS.md` só avisa que o antigo trancaria todas as cidades).
- **Progresso e retomada do `CityRekey`** — a transação única já garante tudo-ou-nada; chunking é outro desenho.
- **`contracts/events`** — remover `municipality_id` do `EVENTS.md` é MAJOR pelo ADR-0015, com CHANGELOG e migração coordenada: repositório e plano próprios.
- **Renomear as chaves `municipality_*`** do contrato de sessão consumidas por `dashboard` e `admin` — mesma razão.

## Riscos

1. **`apps/admin` sem remote.** Tudo que este plano faz nele existe só nesta máquina. É o risco mais concreto do plano e não se resolve dentro dele.
2. **O roteamento por caminho no kamal-proxy** (Task 9, Step 2) pode não existir na versão em uso; a alternativa (nginx de borda) muda o tamanho da tarefa.
3. **`verify-full` derruba tudo se ligado antes do certificado.** Por isso o passo é documentar e parar.
4. **A guarda de digest do restore** só protege dumps tirados depois da Task 7. Os anteriores dependem de conferência humana.
5. **A validação no console duplica regra da API** (Task 2). Divergir é possível; por isso a API segue sendo a autoridade e o teste diz isso explicitamente.
