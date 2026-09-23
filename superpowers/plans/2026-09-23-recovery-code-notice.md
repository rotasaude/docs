# Aviso por e-mail quando um código de recuperação é usado — plano

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** quando um step-up passa por **código de recuperação** (e não por TOTP), o dono da conta recebe um e-mail dizendo quando, de onde e quantos códigos sobraram — fechando o último caminho pelo qual senha mais um código davam acesso privilegiado em silêncio.

**Architecture:** um método novo no `SecurityMailer` que já existe (`recovery_code_used`), com view HTML e texto, e um ponto de chamada em `MfaController#step_up`. O controller passa a distinguir qual fator aprovou o step-up — hoje TOTP e código de recuperação levam ao mesmo `true` —, e o aviso sai só no segundo caso, depois de a sessão ser carimbada.

**Tech Stack:** Rails 8.1, ActionMailer, RSpec (apps/api). Nada de frontend.

**Spec:** `docs/superpowers/specs/2026-09-23-recovery-code-notice-design.md` (commit `cee7240`). Este plano implementa §3 a §6. O aviso irmão (`2026-09-23-authenticator-change-notice-design.md`) é a referência das regras que valem aqui sem repetição.

## Global Constraints

- Commits: Conventional Commits **em inglês**, um tipo por assunto, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Git: `/opt/homebrew/bin/git`. Branch `feat/recovery-code-notice` em `apps/api` (único repo de código). Nunca em `main`, nunca push, nunca merge.
- Staging explícito por arquivo; nunca `git add -A`.
- Suíte completa do api: na raiz do monorepo, `docker compose stop worker`, depois `docker compose exec -T api bundle exec rspec`, depois `docker compose start worker` (sempre religar). Acima de ~3,5 min é regressão.
- **R42 — só valores simples no mailer.** `deliver_later` roda no worker, fora da conexão da cidade. `occurred_at` viaja como string ISO 8601 e a view exibe em `America/Sao_Paulo`; `remaining` é inteiro.
- **Nada de código no e-mail:** nem o código usado, nem parte dele, nem os restantes. Só a contagem.
- **O aviso nunca faz a ação falhar:** a sessão é carimbada primeiro, o enfileiramento depois; exceção ali é registrada em log (id do usuário, **nunca** e-mail nem IP) e engolida; a resposta segue `200`. O IP usa o `safe_remote_ip` que já existe no controller.
- **Não mudar o comportamento do step-up.** O TOTP continua sendo consumido **uma vez só** (`reused_totp?` já consome; nada pode consumir de novo), e um código de recuperação continua sendo consumido uma vez. Se a refatoração mexer nisso, é bug.
- Specs de request exigem `type: :request`; de mailer, `type: :mailer`.
- Nunca imprima código, segredo nem `otpauth` em relatório.
- Nunca rode `start.sh`.

## File Structure

- Modify: `app/mailers/security_mailer.rb`
- Create: `app/views/security_mailer/recovery_code_used.text.erb`
- Create: `app/views/security_mailer/recovery_code_used.html.erb`
- Modify: `spec/mailers/security_mailer_spec.rb`
- Modify: `app/controllers/mfa_controller.rb`
- Create: `spec/requests/mfa_recovery_code_notice_spec.rb`

---

### Task 1: `SecurityMailer#recovery_code_used` e as views

**Files:**
- Modify: `apps/api/app/mailers/security_mailer.rb`
- Create: `apps/api/app/views/security_mailer/recovery_code_used.text.erb`, `apps/api/app/views/security_mailer/recovery_code_used.html.erb`
- Test: `apps/api/spec/mailers/security_mailer_spec.rb` (acréscimo)

**Interfaces:**
- Consumes: `ApplicationMailer` (from por `MAIL_FROM`, layout `mailer`, `delivery_job = CityMailDeliveryJob`).
- Produces:
  ```ruby
  SecurityMailer.recovery_code_used(
    email_address:, city_name:, ip_address:, occurred_at:, remaining:
  ) # occurred_at: string ISO8601; remaining: Integer
  ```
  Assunto: `[rota-saúde] Código de recuperação usado`.

**Corpo (spec §4), nesta ordem:**
1. "Um código de recuperação da sua conta na cidade de <cidade> foi usado para aprovar uma ação sensível.";
2. data e hora em `America/Sao_Paulo`, e o IP;
3. "Se não foi você, fale agora com o administrador municipal — com este acesso é possível assinar e publicar protocolo em seu nome.";
4. "Restam N códigos de recuperação." e, quando `remaining` é zero, "Não resta nenhum código: a partir de agora só o autenticador aprova ações sensíveis."

Com linha em branco antes de (3) e antes de (4), como o `authenticator_changed` já faz. Sem link, sem anexo.

- [ ] **Step 1: Branch**

```bash
cd apps/api && /opt/homebrew/bin/git checkout -b feat/recovery-code-notice
```

- [ ] **Step 2: Escrever a spec que falha**

Acrescente a `spec/mailers/security_mailer_spec.rb` (o arquivo já tem o `describe` do mailer, o `occurred_at` de 13:30 UTC e o helper `bodies`; reuse-os):

```ruby
  describe "recovery_code_used" do
    def build_recovery_mail(remaining:)
      described_class.recovery_code_used(
        email_address: "ana@cidade.gov.br", city_name: "Curitiba",
        ip_address: "203.0.113.10", occurred_at: occurred_at, remaining: remaining
      )
    end

    it "assunto e corpo com cidade, hora de Brasília, IP, frase de ação e contagem" do
      mail = build_recovery_mail(remaining: 9)

      expect(mail.to).to eq([ "ana@cidade.gov.br" ])
      expect(mail.subject).to eq("[rota-saúde] Código de recuperação usado")
      bodies(mail).each do |body|
        expect(body).to include("cidade de Curitiba")
        expect(body).to include("23/09/2026 10:30")
        expect(body).to include("203.0.113.10")
        expect(body).to include("Se não foi você")
        expect(body).to include("Restam 9 códigos de recuperação")
      end
    end

    it "sem nenhum código restante, avisa que só o autenticador aprova" do
      bodies(build_recovery_mail(remaining: 0)).each do |body|
        expect(body).to include("Não resta nenhum código")
        expect(body).to include("só o autenticador")
      end
    end

    it "com códigos restantes, NÃO fala do 'só o autenticador'" do
      bodies(build_recovery_mail(remaining: 9)).each do |body|
        expect(body).not_to include("só o autenticador")
      end
    end

    it "a frase de ação vem antes da contagem" do
      bodies(build_recovery_mail(remaining: 9)).each do |body|
        expect(body.index("Se não foi você")).to be < body.index("Restam 9")
      end
    end

    it "não leva código, segredo nem link" do
      bodies(build_recovery_mail(remaining: 1)).each do |body|
        expect(body).not_to match(/otpauth|otp_secret/i)
        expect(body).not_to match(/https?:\/\//)
        # Nenhum código de recuperação tem esta forma no corpo: 10 caracteres
        # alfanuméricos minúsculos isolados (Mfa::Enroll::RECOVERY_LEN).
        expect(body).not_to match(/\b[a-z0-9]{10}\b/)
      end
    end
  end
```

O exemplo do formato de código é defensivo: se algum dia alguém passar o código consumido para a view, ele fica vermelho. Se alguma palavra legítima do corpo casar com esse padrão, ajuste **o texto do corpo** (ou torne a expressão mais estreita) e diga no relatório — não apague o exemplo.

- [ ] **Step 3: Rodar e ver falhar**

Da raiz do monorepo:
```bash
docker compose exec -T api bundle exec rspec spec/mailers/security_mailer_spec.rb
```
Esperado: FAIL — método inexistente / template ausente.

- [ ] **Step 4: Implementar**

Em `app/mailers/security_mailer.rb`, depois de `authenticator_changed`:

```ruby
  # Aviso de uso de código de recuperação (spec 2026-09-23-recovery-code-notice).
  # É o caminho irmão do authenticator_changed: senha + um código dão step-up
  # válido por 5 minutos, e com ele se assina e publica protocolo sem tocar no
  # autenticador. `remaining` é a contagem DEPOIS do consumo — o número é o que
  # deixa o aviso acionável ("restam 9" é uso normal; "restam 2" é lista sendo
  # consumida; zero é conta sem rede de segurança).
  #
  # Nenhum código, nem parte dele, entra aqui. Só a contagem.
  def recovery_code_used(email_address:, city_name:, ip_address:, occurred_at:, remaining:)
    @city_name = city_name
    @ip_address = ip_address
    @remaining = Integer(remaining)
    @occurred_at = Time.iso8601(occurred_at).in_time_zone("America/Sao_Paulo")

    mail(to: email_address, subject: "[rota-saúde] Código de recuperação usado")
  end
```

`app/views/security_mailer/recovery_code_used.text.erb`:

```erb
Olá,

Um código de recuperação da sua conta na cidade de <%= @city_name %> foi usado para aprovar uma ação sensível.

Data/hora: <%= @occurred_at.strftime("%d/%m/%Y %H:%M") %> (horário de Brasília)
IP de origem: <%= @ip_address %>

Se não foi você, fale agora com o administrador municipal — com este acesso é possível assinar e publicar protocolo em seu nome.

<% if @remaining.zero? -%>
Não resta nenhum código: a partir de agora só o autenticador aprova ações sensíveis.
<% else -%>
Restam <%= @remaining %> códigos de recuperação.
<% end -%>
```

`app/views/security_mailer/recovery_code_used.html.erb`:

```erb
<p>Olá,</p>
<p>Um código de recuperação da sua conta na cidade de <%= @city_name %> foi usado para aprovar uma ação sensível.</p>
<p>
  Data/hora: <%= @occurred_at.strftime("%d/%m/%Y %H:%M") %> (horário de Brasília)<br>
  IP de origem: <%= @ip_address %>
</p>
<p>Se não foi você, fale agora com o administrador municipal — com este acesso é possível assinar e publicar protocolo em seu nome.</p>
<% if @remaining.zero? %>
  <p>Não resta nenhum código: a partir de agora só o autenticador aprova ações sensíveis.</p>
<% else %>
  <p>Restam <%= @remaining %> códigos de recuperação.</p>
<% end %>
```

Confira `app/views/security_mailer/authenticator_changed.text.erb` antes de escrever e mantenha a mesma convenção de `-%>` que ele usa, para não deixar linha solta.

- [ ] **Step 5: Rodar e ver passar**

```bash
docker compose exec -T api bundle exec rspec spec/mailers/security_mailer_spec.rb
```
Esperado: PASS, incluindo os exemplos do `authenticator_changed` que já existiam.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add app/mailers/security_mailer.rb app/views/security_mailer/recovery_code_used.text.erb app/views/security_mailer/recovery_code_used.html.erb spec/mailers/security_mailer_spec.rb
/opt/homebrew/bin/git commit -m "feat: add the recovery code usage notice mailer" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: avisar no step-up

**Files:**
- Modify: `apps/api/app/controllers/mfa_controller.rb`
- Test: `apps/api/spec/requests/mfa_recovery_code_notice_spec.rb`

**Interfaces:**
- Consumes:
  - `SecurityMailer.recovery_code_used(...)` (Task 1);
  - `Mfa::Verify.totp_step_for(user, code)` (leitura pura) e `Mfa::Verify.consume_recovery_code(user, code)` (consome e regrava a lista);
  - `safe_remote_ip` e o padrão de `notify_authenticator_change`, ambos já privados no controller.
- Produces: `POST /mfa/step_up` aprovado por código de recuperação enfileira **um** `SecurityMailer.recovery_code_used`, com `remaining` igual ao número de códigos que sobraram. Aprovado por TOTP, ou recusado, não enfileira nada. Nenhuma resposta muda.

**A refatoração:** hoje `stepped_up?` devolve `true` tanto para TOTP quanto para código de recuperação. **Apague `stepped_up?`** (nada mais o chama depois desta mudança — confirme com `grep -rn "stepped_up?" app spec`) e ponha no lugar um método que diz **qual** fator aprovou, preservando a ordem e o consumo único:

```ruby
  # Qual fator aprovou o step-up: :totp, :recovery, ou nil quando nenhum.
  #
  # A ordem importa e o consumo também: `reused_totp?` (chamado antes, na ação)
  # já consumiu o passo do TOTP quando o código é de TOTP válido, então aqui a
  # checagem de TOTP é leitura pura (`totp_step_for`) e nunca consome de novo.
  # `consume_recovery_code` é o único consumo deste método, e só é tentado
  # quando o código não é um TOTP válido.
  def step_up_factor
    return :totp if Mfa::Verify.totp_step_for(Current.user, params[:code]).present?
    return :recovery if Mfa::Verify.consume_recovery_code(Current.user, params[:code])

    nil
  end
```

e a ação:

```ruby
  def step_up
    return render(json: { error: "code_reused" }, status: :unprocessable_entity) if reused_totp?

    factor = step_up_factor
    return render(json: { error: "invalid_code" }, status: :unprocessable_entity) if factor.nil?

    # O carimbo vem primeiro: um aviso não pode sair se a sessão não valeu.
    Current.session.update!(mfa_verified_at: Time.current)
    notify_recovery_code_used if factor == :recovery
    render json: { ok: true }
  end
```

e o aviso, junto dos outros métodos privados:

```ruby
  # Aviso de uso de código de recuperação (spec 2026-09-23-recovery-code-notice
  # §3). Mesmas regras do aviso de autenticador: depois do carimbo, nunca
  # derruba a ação, log só com o id do usuário.
  #
  # `otp_recovery_codes` já está atualizado em memória: consume_recovery_code
  # regrava a lista no mesmo registro (`user.update!`).
  def notify_recovery_code_used
    SecurityMailer.recovery_code_used(
      email_address: Current.user.email_address,
      city_name: Current.city&.name.to_s,
      ip_address: safe_remote_ip,
      occurred_at: Time.current.iso8601,
      remaining: Current.user.otp_recovery_codes.size
    ).deliver_later
  rescue StandardError => e
    Rails.logger.error("[mfa] aviso de recovery code não enfileirado para #{Current.user.id}: #{e.class}")
  end
```

Confirme lendo `Mfa::Verify.consume_recovery_code` que a lista é regravada no **mesmo** objeto (`user.update!`), de modo que `Current.user.otp_recovery_codes.size` já reflete o consumo sem `reload`. Se não for o caso, use `Current.user.reload.otp_recovery_codes.size` e diga no relatório.

- [ ] **Step 1: Escrever a spec que falha**

`spec/requests/mfa_recovery_code_notice_spec.rb`. Molde de arranjo: `spec/requests/mfa_confirm_notice_spec.rb` (o `sign_in_as`, o `deliveries`, o `ActiveJob::TestHelper`) — copie de lá, inclusive o `travel_to` onde fizer diferença para a janela do TOTP.

```ruby
require "rails_helper"

# Spec do aviso de código de recuperação (2026-09-23-recovery-code-notice §6).
# Só o step-up aprovado por CÓDIGO DE RECUPERAÇÃO avisa; TOTP e recusa não.
RSpec.describe "MFA recovery code notice", type: :request do
  include ActiveJob::TestHelper

  def json = JSON.parse(response.body)
  def deliveries = ActionMailer::Base.deliveries

  let!(:user) { User.create!(email_address: "eve-#{SecureRandom.hex(3)}@example.org", password: "secret123") }
  let(:codes) { @codes }

  before do
    @codes = Mfa::Enroll.call(user)[:recovery_codes]
    user.update!(otp_enabled: true)
    deliveries.clear
  end

  def step_up!(code)
    perform_enqueued_jobs { post "/mfa/step_up", params: { code: code }, as: :json }
  end

  it "código de recuperação: um aviso, com a contagem que sobrou" do
    session = sign_in_as(user)

    step_up!(codes.first)

    expect(response).to have_http_status(:ok)
    expect(session.reload.mfa_verified_at).to be_within(5.seconds).of(Time.current)
    expect(deliveries.size).to eq(1)
    expect(deliveries.first.to).to eq([ user.email_address ])
    expect(deliveries.first.subject).to eq("[rota-saúde] Código de recuperação usado")
    expect(deliveries.first.text_part.body.decoded).to include("Restam #{user.reload.otp_recovery_codes.size} códigos")
  end

  it "TOTP não avisa" do
    sign_in_as(user)

    step_up!(ROTP::TOTP.new(user.reload.otp_secret).now)

    expect(response).to have_http_status(:ok)
    expect(deliveries).to be_empty
  end

  it "código inválido não avisa" do
    sign_in_as(user)

    step_up!("000000")

    expect(response).to have_http_status(:unprocessable_entity)
    expect(json).to eq("error" => "invalid_code")
    expect(deliveries).to be_empty
  end

  it "código de recuperação já usado não avisa de novo" do
    sign_in_as(user)
    step_up!(codes.first)
    deliveries.clear

    step_up!(codes.first)

    expect(response).to have_http_status(:unprocessable_entity)
    expect(deliveries).to be_empty
  end

  it "último código: aviso dizendo que só o autenticador aprova" do
    user.update!(otp_recovery_codes: user.otp_recovery_codes.first(1))
    sign_in_as(user)

    step_up!(codes.first)

    expect(deliveries.size).to eq(1)
    expect(deliveries.first.text_part.body.decoded).to include("Não resta nenhum código")
    expect(user.reload.otp_recovery_codes).to eq([])
  end

  it "falha ao enfileirar não descarimba a sessão nem muda a resposta" do
    session = sign_in_as(user)
    allow(SecurityMailer).to receive(:recovery_code_used).and_raise(StandardError, "fila fora do ar")

    step_up!(codes.first)

    expect(response).to have_http_status(:ok)
    expect(session.reload.mfa_verified_at).to be_present
  end

  it "o TOTP continua sendo consumido uma vez só" do
    sign_in_as(user)
    code = ROTP::TOTP.new(user.reload.otp_secret).now
    step_up!(code)
    expect(response).to have_http_status(:ok)

    step_up!(code)

    expect(response).to have_http_status(:unprocessable_entity)
    expect(json).to eq("error" => "code_reused")
    expect(deliveries).to be_empty
  end
end
```

O exemplo "último código" corta a lista para um item e usa `codes.first`: a ordem é garantida porque `Mfa::Enroll` monta os hashes com `codes.map`, preservando a posição (confira o arquivo antes de confiar nisso).

- [ ] **Step 2: Rodar e ver falhar**

```bash
docker compose exec -T api bundle exec rspec spec/requests/mfa_recovery_code_notice_spec.rb
```
Esperado: FAIL — nenhum aviso é enfileirado hoje.

- [ ] **Step 3: Implementar** (o código da seção "A refatoração", acima)

- [ ] **Step 4: Rodar e ver passar**

```bash
docker compose exec -T api bundle exec rspec spec/requests/mfa_recovery_code_notice_spec.rb spec/requests/mfa_confirm_notice_spec.rb spec/controllers/mfa_controller_spec.rb spec/requests/mfa_pending_enrollment_spec.rb spec/mailers
```
Esperado: PASS. Esses quatro cobrem o resto do controller — se algum quebrar, a refatoração mudou comportamento, o que é bug.

- [ ] **Step 5: Suíte completa** (worker parado; ver Global Constraints). Esperado: 0 falhas.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/mfa_controller.rb spec/requests/mfa_recovery_code_notice_spec.rb
/opt/homebrew/bin/git commit -m "feat: notify the owner when a recovery code approves a step-up" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Fora deste plano

- Aviso ao operador de plataforma, que consome código de recuperação no login do console.
- Reemissão de códigos de recuperação e qualquer tela para isso.
- Limitar ou bloquear step-up por código de recuperação.
- Os gates já registrados no aviso irmão (provar `remote_ip` atrás do proxy; persistência dos argumentos na fila).
