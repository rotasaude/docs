# Aviso por e-mail quando o autenticador muda — plano

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** quando `POST /mfa/confirm` promove o autenticador pendente, o dono da conta recebe um e-mail dizendo se foi um cadastro ou uma troca, quando e de qual IP — de modo que um sequestro do segundo fator deixe de ser silencioso.

**Architecture:** um mailer novo, `SecurityMailer`, com view HTML e texto, no molde de `AlertMailer` (só valores simples, R42). O `MfaController#confirm` enfileira o e-mail **depois** de a promoção comitar, e só no resultado `:ok`. Falha ao enfileirar é registrada e engolida: o aviso nunca derruba a ação.

**Tech Stack:** Rails 8.1, ActionMailer (`delivery_method = :test` em dev e test), RSpec (apps/api). Nada de frontend.

**Spec:** `docs/superpowers/specs/2026-09-23-authenticator-change-notice-design.md` (commit `28d3c8f`). Este plano implementa §3 a §6.

## Global Constraints

- Commits: Conventional Commits **em inglês**, tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Git: `/opt/homebrew/bin/git`. Branch `feat/authenticator-change-notice` em `apps/api` (único repo de código). Nunca em `main`, nunca push, nunca merge.
- Staging explícito por arquivo; nunca `git add -A`.
- Suíte completa do api: na raiz do monorepo, `docker compose stop worker`, depois `docker compose exec -T api bundle exec rspec`, depois `docker compose start worker` (sempre religar). Acima de ~3 min é regressão.
- **R42 — só valores simples no mailer.** `deliver_later` roda no worker, fora da conexão da cidade, onde um `User` (GlobalID) não desserializa. Passe string, e nada de objeto de modelo. `occurred_at` viaja como string ISO 8601 e a view exibe em `America/Sao_Paulo`, como `AlertMailer` faz.
- **O e-mail nunca leva segredo:** nada de `otpauth`, `otp_secret`, código TOTP ou código de recuperação. Nem no assunto, nem no corpo, nem em log.
- **O aviso nunca faz a ação falhar:** enfileirar acontece depois do commit, e qualquer exceção ali é registrada em log (com o id do usuário, **nunca** o e-mail nem o IP) e engolida. A resposta segue `200`.
- Specs de request exigem `type: :request`; specs de mailer, `type: :mailer`. `sign_in_as(user)` vem de `spec/support/city_request_auth.rb`.
- Nunca imprima segredo nem código em relatório de execução.
- Nunca rode `start.sh`.

## File Structure

- Create: `app/mailers/security_mailer.rb`
- Create: `app/views/security_mailer/authenticator_changed.text.erb`
- Create: `app/views/security_mailer/authenticator_changed.html.erb`
- Create: `spec/mailers/security_mailer_spec.rb`
- Modify: `app/controllers/mfa_controller.rb`
- Create: `spec/requests/mfa_confirm_notice_spec.rb`
- Modify: `README.md`

---

### Task 1: `SecurityMailer` e as views

**Files:**
- Create: `apps/api/app/mailers/security_mailer.rb`, `apps/api/app/views/security_mailer/authenticator_changed.text.erb`, `apps/api/app/views/security_mailer/authenticator_changed.html.erb`
- Test: `apps/api/spec/mailers/security_mailer_spec.rb`

**Interfaces:**
- Consumes: `ApplicationMailer` (que já define `from` por `MAIL_FROM`, o layout `mailer` e `delivery_job = CityMailDeliveryJob`).
- Produces:
  ```ruby
  SecurityMailer.authenticator_changed(
    email_address:, kind:, city_name:, ip_address:, occurred_at:
  ) # kind: "enrolled" | "replaced"; occurred_at: string ISO8601
  ```
  Assunto: `[rota-saúde] Autenticador cadastrado` (enrolled) ou `[rota-saúde] Autenticador trocado` (replaced).

**Texto do corpo (spec §4), nesta ordem:**
1. o que aconteceu, com a cidade — "O autenticador da sua conta na <cidade> foi cadastrado." / "…foi trocado.";
2. quando e de onde — data e hora em `America/Sao_Paulo`, e o IP;
3. "Se não foi você, fale agora com o administrador municipal — quem tem o autenticador aprova protocolo em seu nome.";
4. só na troca: "Os códigos de recuperação anteriores deixaram de valer."

Sem link, sem anexo.

- [ ] **Step 1: Branch**

```bash
cd apps/api && /opt/homebrew/bin/git checkout -b feat/authenticator-change-notice
```

- [ ] **Step 2: Escrever a spec que falha**

`spec/mailers/security_mailer_spec.rb`. Molde: `spec/mailers/alert_mailer_spec.rb` — ele renderiza de verdade (sem stub), o que é o ponto: uma view ausente só aparece na entrega real.

```ruby
require "rails_helper"

# Aviso de mudança do segundo fator (spec 2026-09-23-authenticator-change-notice
# §4). Renderiza de verdade: view ausente ou layout quebrado tem de ficar
# vermelho aqui, não na primeira entrega em produção.
RSpec.describe SecurityMailer, type: :mailer do
  # 13:30 UTC == 10:30 em America/Sao_Paulo — prova que a view exibe no horário
  # de Brasília, qualquer que seja o offset da string recebida.
  let(:occurred_at) { "2026-09-23T13:30:00Z" }

  def build_mail(kind:)
    described_class.authenticator_changed(
      email_address: "ana@cidade.gov.br", kind: kind, city_name: "Curitiba",
      ip_address: "203.0.113.10", occurred_at: occurred_at
    )
  end

  def bodies(mail) = [ mail.text_part.body.decoded, mail.html_part.body.decoded ]

  it "cadastro: assunto e corpo com cidade, hora de Brasília, IP e o que fazer" do
    mail = build_mail(kind: "enrolled")

    expect(mail.to).to eq([ "ana@cidade.gov.br" ])
    expect(mail.subject).to eq("[rota-saúde] Autenticador cadastrado")
    bodies(mail).each do |body|
      expect(body).to include("Curitiba")
      expect(body).to include("cadastrado")
      expect(body).to include("23/09/2026 10:30")
      expect(body).to include("203.0.113.10")
      expect(body).to include("Se não foi você")
      expect(body).to include("administrador municipal")
    end
  end

  it "troca: assunto próprio e o aviso dos códigos antigos" do
    mail = build_mail(kind: "replaced")

    expect(mail.subject).to eq("[rota-saúde] Autenticador trocado")
    bodies(mail).each do |body|
      expect(body).to include("trocado")
      expect(body).to include("códigos de recuperação anteriores deixaram de valer")
    end
  end

  it "o cadastro NÃO fala de códigos antigos (não havia)" do
    bodies(build_mail(kind: "enrolled")).each do |body|
      expect(body).not_to include("códigos de recuperação anteriores")
    end
  end

  # Defensivo: o mailer recebe só valores simples, e nenhum deles é segredo.
  # Guarda contra alguém acrescentar segredo ou link no futuro.
  it "não leva segredo, código nem link" do
    bodies(build_mail(kind: "replaced")).each do |body|
      expect(body).not_to match(/otpauth|otp_secret/i)
      expect(body).not_to match(/https?:\/\//)
    end
  end

  it "recusa um kind desconhecido em vez de mandar e-mail ambíguo" do
    expect { described_class.authenticator_changed(
      email_address: "ana@cidade.gov.br", kind: "sei-la", city_name: "Curitiba",
      ip_address: "203.0.113.10", occurred_at: occurred_at
    ).subject }.to raise_error(ArgumentError)
  end
end
```

O layout `app/views/layouts/mailer.html.erb` pode conter URL (rodapé); se o exemplo "não leva link" falhar por causa do layout, restrinja a asserção ao trecho do mailer (por exemplo verificando o corpo do texto puro, ou o HTML sem o layout) e **diga no relatório** o que o layout traz. Não afrouxe a asserção para "qualquer link serve".

- [ ] **Step 3: Rodar e ver falhar**

Da raiz do monorepo:
```bash
docker compose exec -T api bundle exec rspec spec/mailers/security_mailer_spec.rb
```
Esperado: FAIL — `uninitialized constant SecurityMailer`.

- [ ] **Step 4: Implementar o mailer**

`app/mailers/security_mailer.rb`:

```ruby
# Aviso ao dono da conta quando o SEGUNDO FATOR dele muda (spec
# 2026-09-23-authenticator-change-notice-design). Duas brechas ficaram abertas
# por desenho no autenticador pendente: numa conta sem TOTP, quem tem a senha
# cadastra o seu; numa conta com TOTP, senha + um código de recuperação trocam
# o segundo fator inteiro. Nenhuma das duas é visível para o dono — este
# e-mail é o que rompe o silêncio.
#
# Só valores simples (R42): deliver_later roda no worker, fora da conexão da
# cidade, onde um User (GlobalID) não desserializa.
#
# Nada de segredo aqui: nem otpauth, nem código, nem link (um e-mail de
# segurança com link é o formato que o phishing imita).
class SecurityMailer < ApplicationMailer
  KINDS = { "enrolled" => "Autenticador cadastrado", "replaced" => "Autenticador trocado" }.freeze

  def authenticator_changed(email_address:, kind:, city_name:, ip_address:, occurred_at:)
    subject = KINDS.fetch(kind) { raise ArgumentError, "kind desconhecido: #{kind.inspect}" }

    @replaced = kind == "replaced"
    @city_name = city_name
    @ip_address = ip_address
    # Normaliza para America/Sao_Paulo na exibição, como AlertMailer.
    @occurred_at = Time.iso8601(occurred_at).in_time_zone("America/Sao_Paulo")

    mail(to: email_address, subject: "[rota-saúde] #{subject}")
  end
end
```

`app/views/security_mailer/authenticator_changed.text.erb`:

```erb
Olá,

O autenticador da sua conta na <%= @city_name %> foi <%= @replaced ? "trocado" : "cadastrado" %>.

Data/hora: <%= @occurred_at.strftime("%d/%m/%Y %H:%M") %> (horário de Brasília)
IP de origem: <%= @ip_address %>
<% if @replaced %>
Os códigos de recuperação anteriores deixaram de valer.
<% end %>
Se não foi você, fale agora com o administrador municipal — quem tem o autenticador aprova protocolo em seu nome.
```

`app/views/security_mailer/authenticator_changed.html.erb` (mesma marcação de `app/views/alert_mailer/urgent.html.erb`):

```erb
<p>Olá,</p>
<p>O autenticador da sua conta na <%= @city_name %> foi <%= @replaced ? "trocado" : "cadastrado" %>.</p>
<p>
  Data/hora: <%= @occurred_at.strftime("%d/%m/%Y %H:%M") %> (horário de Brasília)<br>
  IP de origem: <%= @ip_address %>
</p>
<% if @replaced %>
  <p>Os códigos de recuperação anteriores deixaram de valer.</p>
<% end %>
<p>Se não foi você, fale agora com o administrador municipal — quem tem o autenticador aprova protocolo em seu nome.</p>
```

- [ ] **Step 5: Rodar e ver passar**

```bash
docker compose exec -T api bundle exec rspec spec/mailers/security_mailer_spec.rb
```
Esperado: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add app/mailers/security_mailer.rb app/views/security_mailer spec/mailers/security_mailer_spec.rb
/opt/homebrew/bin/git commit -m "feat: add the authenticator change notice mailer" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: enfileirar na confirmação

**Files:**
- Modify: `apps/api/app/controllers/mfa_controller.rb`, `apps/api/README.md`
- Test: `apps/api/spec/requests/mfa_confirm_notice_spec.rb`

**Interfaces:**
- Consumes:
  - `SecurityMailer.authenticator_changed(...)` (Task 1);
  - `Mfa::PendingEnrollment.confirm(user, code:)` → `:ok | :no_pending_enrollment | :enrollment_expired | :invalid_code | :code_reused`;
  - `Current.user`, `Current.city` (nome da cidade), `request.remote_ip`.
- Produces: `POST /mfa/confirm` com resultado `:ok` enfileira **um** `SecurityMailer.authenticator_changed`, com `kind: "replaced"` quando a conta já tinha autenticador antes, e `"enrolled"` quando não tinha. Nenhuma recusa enfileira nada. A resposta não muda em nenhum caso.

**Como distinguir cadastro de troca:** o controller lê `Current.user.mfa_enrolled?` **antes** de chamar o command — depois da promoção a conta sempre tem autenticador, e essa é a única forma de saber, sem mudar o command.

- [ ] **Step 1: Escrever a spec que falha**

`spec/requests/mfa_confirm_notice_spec.rb`. Molde de arranjo: `spec/requests/mfa_pending_enrollment_spec.rb` (o `enroll!`, o `pending_code`, o `sign_in_as`) — copie de lá.

```ruby
require "rails_helper"

# Spec do aviso (2026-09-23-authenticator-change-notice §6): a confirmação que
# promove o pendente avisa o dono da conta. Recusa não avisa, e uma falha no
# envio não desfaz nem derruba a promoção.
RSpec.describe "MFA confirm notice", type: :request do
  def json = JSON.parse(response.body)
  def deliveries = ActionMailer::Base.deliveries

  let!(:user) { User.create!(email_address: "dan-#{SecureRandom.hex(3)}@example.org", password: "secret123") }

  before { deliveries.clear }

  def enroll!(stepped_up: false)
    session = sign_in_as(user)
    session.update!(mfa_verified_at: Time.current) if stepped_up
    post "/mfa/enroll", as: :json
    expect(response).to have_http_status(:ok)
  end

  def pending_code = ROTP::TOTP.new(user.reload.otp_pending_secret).now

  def confirm!(code: pending_code)
    perform_enqueued_jobs { post "/mfa/confirm", params: { code: code }, as: :json }
  end

  it "primeiro cadastro: um aviso, com o assunto de cadastro, para o dono da conta" do
    enroll!

    confirm!

    expect(response).to have_http_status(:ok)
    expect(deliveries.size).to eq(1)
    expect(deliveries.first.to).to eq([ user.email_address ])
    expect(deliveries.first.subject).to eq("[rota-saúde] Autenticador cadastrado")
  end

  it "troca: assunto de troca" do
    Mfa::Enroll.call(user)
    user.update!(otp_enabled: true)
    enroll!(stepped_up: true)

    confirm!

    expect(deliveries.size).to eq(1)
    expect(deliveries.first.subject).to eq("[rota-saúde] Autenticador trocado")
  end

  it "o corpo leva o IP da requisição e o nome da cidade" do
    enroll!

    confirm!

    # O nome vem do catálogo, não de Current: Current é por requisição e já
    # foi zerado quando o exemplo chega aqui.
    body = deliveries.first.text_part.body.decoded
    expect(body).to include(City.find_by!(slug: TEST_CITY_A.slug).name)
    expect(body).to include("127.0.0.1")
  end

  it "recusa não avisa ninguém" do
    enroll!

    confirm!(code: "000000")

    expect(response).to have_http_status(:unprocessable_entity)
    expect(deliveries).to be_empty
  end

  it "sem pendente: recusa e nenhum aviso" do
    sign_in_as(user)

    perform_enqueued_jobs { post "/mfa/confirm", params: { code: "123456" }, as: :json }

    expect(json).to eq("error" => "no_pending_enrollment")
    expect(deliveries).to be_empty
  end

  it "falha ao enfileirar não derruba a confirmação nem desfaz a promoção" do
    enroll!
    allow(SecurityMailer).to receive(:authenticator_changed).and_raise(StandardError, "fila fora do ar")

    confirm!

    expect(response).to have_http_status(:ok)
    expect(user.reload.mfa_enrolled?).to be(true)
    expect(user.otp_pending_secret).to be_nil
  end

  it "avisa uma vez por confirmação, não uma por tentativa" do
    enroll!
    confirm!(code: "000000")
    deliveries.clear

    confirm!

    expect(deliveries.size).to eq(1)
  end
end
```

`perform_enqueued_jobs` precisa de `ActiveJob::TestHelper` — veja como as specs que já asseguram e-mail fazem (`spec/requests/` de convite ou senha, e `spec/jobs/dispatch_municipality_alert_job_spec.rb`) e siga o mesmo caminho: se o projeto usa `deliver_later` com adapter de teste e verifica `enqueued_jobs`, faça igual em vez de forçar `perform_enqueued_jobs`. O que o teste precisa provar é: **um** aviso na promoção, **nenhum** na recusa, e resposta intacta quando o envio explode. Ajuste o mecanismo, não o que é provado.

O exemplo do IP assume que a requisição de teste chega de `127.0.0.1`; se o `remote_ip` do ambiente de teste for outro, afirme o valor real que o Rails reporta (leia-o no teste, não invente).

- [ ] **Step 2: Rodar e ver falhar**

```bash
docker compose exec -T api bundle exec rspec spec/requests/mfa_confirm_notice_spec.rb
```
Esperado: FAIL — nenhum e-mail é enfileirado hoje.

- [ ] **Step 3: Implementar**

Em `app/controllers/mfa_controller.rb`, troque o `confirm` por:

```ruby
  # A matrícula só vale depois daqui: é `confirm` que promove o pendente. O
  # autenticador anterior vale até esta linha passar.
  def confirm
    # ANTES do command: depois da promoção a conta sempre tem autenticador, e
    # este é o único ponto onde cadastro e troca se distinguem.
    replacing = Current.user.mfa_enrolled?

    outcome = Mfa::PendingEnrollment.confirm(Current.user, code: params[:code])
    if outcome == :ok
      notify_authenticator_change(replacing: replacing)
      return render(json: { ok: true })
    end

    # :no_pending_enrollment | :enrollment_expired | :invalid_code | :code_reused
    render json: { error: outcome.to_s }, status: :unprocessable_entity
  end
```

e, junto dos outros métodos privados:

```ruby
  # Aviso ao dono da conta (spec 2026-09-23-authenticator-change-notice §3).
  # Roda DEPOIS de a promoção comitar — dentro da transação, um rollback
  # mandaria aviso de algo que não aconteceu.
  #
  # Nunca derruba a ação: quem confirmou já tem o autenticador novo, e um
  # servidor de e-mail fora do ar não pode transformar isso em 500. O log leva
  # o id do usuário, nunca o e-mail nem o IP (CityMailDeliveryJob já desliga
  # log_arguments para a mesma razão).
  def notify_authenticator_change(replacing:)
    SecurityMailer.authenticator_changed(
      email_address: Current.user.email_address,
      kind: replacing ? "replaced" : "enrolled",
      city_name: Current.city&.name.to_s,
      ip_address: request.remote_ip,
      occurred_at: Time.current.iso8601
    ).deliver_later
  rescue StandardError => e
    Rails.logger.error("[mfa] aviso de autenticador não enfileirado para #{Current.user.id}: #{e.class}")
  end
```

O `rescue` registra **a classe** da exceção, não a mensagem: mensagem de erro de fila pode carregar payload.

- [ ] **Step 4: Rodar e ver passar**

```bash
docker compose exec -T api bundle exec rspec spec/requests/mfa_confirm_notice_spec.rb spec/requests/mfa_pending_enrollment_spec.rb spec/controllers/mfa_controller_spec.rb spec/mailers
```
Esperado: PASS.

- [ ] **Step 5: Suíte completa** (worker parado; ver Global Constraints). Esperado: 0 falhas.

- [ ] **Step 6: Conferir em dev**

Com o stack de pé, use a página Segurança do dashboard de uma cidade de dev (as contas estão no README; a semente `SignatureCrew` já deixa todas com autenticador) para **trocar** o autenticador de uma conta. Em dev o `delivery_method` é `:test`, então nada sai de verdade: confirme pelo log do worker que o job de entrega rodou, ou pelo `ActionMailer::Base.deliveries` num `rails runner`/console. Registre o que viu **sem** copiar segredo nem código.

- [ ] **Step 7: README**

Em `apps/api/README.md`, uma linha na parte de e-mail/MFA: ao cadastrar ou trocar o autenticador, a conta recebe um aviso com cidade, data/hora e IP; em dev a entrega é `:test` e nada é enviado.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/mfa_controller.rb spec/requests/mfa_confirm_notice_spec.rb README.md
/opt/homebrew/bin/git commit -m "feat: notify the account owner when the authenticator changes" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Fora deste plano

- Aviso de papel privilegiado concedido ou revogado, e de senha alterada (spec §8).
- Preferência para desligar o aviso: é aviso de segurança, não boletim.
- Mantenedor e operador, que têm cadastro próprio.
- Retenção de log do provedor de e-mail.
