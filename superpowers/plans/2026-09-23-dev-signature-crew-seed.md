# Semente de dev para o ciclo assinado — plano

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** depois de `bin/rails db:seed`, cada cidade de dev tem autor, duas revisoras e publisher — com TOTP já cadastrado — e um rascunho pronto, para o ciclo assinado (enviar → assinar × 2 → publicar → assinar × 2 → ativar → reverter) ser percorrível no navegador sem nenhum preparo manual.

**Architecture:** a lógica vai para uma classe de lib testável, `SignatureCrew`, no molde de `lib/dashboard_demo.rb`: ela cria as contas com papel e segredo TOTP fixo, e cria um rascunho **pelo command de autoria** (`Protocols::SaveDraft`), com a conta de autor, para a linha de contribuição e o digest nascerem como o domínio espera. O `db/seeds.rb` só a chama, dentro da conexão de cada cidade, e continua guardado por `Rota.deployed?`.

**Tech Stack:** Rails 8.1, ROTP, RSpec (apps/api). Nada de frontend.

**Origem:** pendência registrada nas três fatias das telas de assinatura — o ciclo nunca rodou de ponta a ponta no navegador porque a Curitiba de dev só tem `admin@curitiba.demo`. Decisões tomadas em conversa (2026-09-23): segredo TOTP **fixo** por conta (como o operador já faz) e rascunho criado **pelo command**.

## Global Constraints

- Commits: Conventional Commits **em inglês**, tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Git: `/opt/homebrew/bin/git`. Branch `feat/dev-signature-crew` em `apps/api` (único repo de código). `docs` commita direto em `main`. Nunca push, nunca merge.
- Staging explícito por arquivo; nunca `git add -A`.
- Suíte completa do api: na raiz do monorepo, `docker compose stop worker`, depois `docker compose exec -T api bundle exec rspec`, depois `docker compose start worker` (sempre religar). Acima de ~3 min é regressão.
- **Só dev.** Nada disso pode rodar em ambiente publicado: o `db/seeds.rb` já é guardado por `Rota.deployed?`, e a classe nova não ganha guarda própria de ambiente — quem decide é o chamador. **Nunca** escreva comparação literal de ambiente (`Rails.env == "production"` e variantes): `spec/architecture/deployed_environment_guard_spec.rb` varre o repositório e falha.
- **Segredos fixos são de dev, e só aparecem em dev.** Os valores ficam em constantes com override por variável de ambiente, como `DEV_OPERATOR_OTP_SECRET` já faz. O seed pode imprimir o `otpauth://` de cada conta na saída (é dev, no terminal de quem rodou); nenhum relatório de execução deste plano deve copiar esses segredos.
- O segredo só é definido quando a conta **ainda não tem** MFA — nunca sobrescreva um segredo existente (é o que permite reset de banco sem reescanear).
- Idempotência: rodar `db:seed` N vezes não cria conta, papel, contribuição nem versão duplicada.
- BCrypt: se algum código de recuperação for gerado, use o custo de `Mfa::Enroll.recovery_code_cost`; custo fixo já levou a suíte a 10 minutos. (Estas contas **não** ganham código de recuperação — ver Task 1.)
- Specs rodam na conexão de `TEST_CITY_A` (o harness já abre); siga `spec/lib/dashboard_demo_spec.rb`.
- Nunca rode `start.sh`.

## File Structure

**apps/api**
- Create: `lib/signature_crew.rb` — as contas, os papéis, os segredos fixos e o rascunho.
- Create: `spec/lib/signature_crew_spec.rb`.
- Modify: `db/seeds.rb` — chama a classe dentro da conexão de cada cidade e imprime o resumo.
- Modify: `README.md` — documenta as contas novas e os segredos de dev.

**docs**
- Modify: `superpowers/specs/2026-09-22-dashboard-protocol-signatures-design.md` — a pendência "ciclo nunca rodou fim a fim" passa a apontar para esta semente.

---

### Task 1: `SignatureCrew` — contas, papéis e rascunho

**Files:**
- Create: `apps/api/lib/signature_crew.rb`
- Test: `apps/api/spec/lib/signature_crew_spec.rb`

**Interfaces:**
- Consumes:
  - `User` (`has_secure_password`, `encrypts :otp_secret`, `otp_enabled`), `Membership` (`ROLES`), ambos no banco da cidade corrente;
  - `Protocols::SaveDraft.call(definition:, by:, correlation_id: nil)` → `Result`, que cria a versão em `draft`, grava a `ProtocolContribution` do ator e publica `protocol.draft_saved`;
  - `ROTP::TOTP#provisioning_uri` para montar o `otpauth://`;
  - o protocolo de demo que o seed já cria (`triage-respiratoria`, versão 1, `active`) — a versão nova é a **2** do mesmo nome.
- Produces:
  ```ruby
  SignatureCrew::PASSWORD_ENV = "DEV_USER_PASSWORD"          # mesmo default do seeds
  SignatureCrew::MEMBERS       # array de { email_prefix:, role:, secret_env:, default_secret: }
  SignatureCrew.seed_current_city(slug:, password:) ->
    { accounts: [ { email:, role:, otpauth_uri: } ], draft: { name:, version:, status: } | nil }
  ```
  `seed_current_city` roda **dentro** de `CityConnection.with(city)`/`Current.set(city:)` — ela não abre conexão nem resolve cidade.

**Contas (por cidade, `<slug>` é o slug da cidade):**

| e-mail | papel | variável de override |
|---|---|---|
| `autor@<slug>.demo` | `protocol_author` | `DEV_AUTHOR_OTP_SECRET` |
| `revisora1@<slug>.demo` | `protocol_reviewer` | `DEV_REVIEWER1_OTP_SECRET` |
| `revisora2@<slug>.demo` | `protocol_reviewer` | `DEV_REVIEWER2_OTP_SECRET` |
| `publisher@<slug>.demo` | `protocol_publisher` | `DEV_PUBLISHER_OTP_SECRET` |

- A conta `admin@<slug>.demo` que o seed já cria **também passa a ter TOTP** (`DEV_MUNI_ADMIN_OTP_SECRET`), porque a tela Equipe exige step-up para conceder papel. Quem faz isso é a Task 2, no `db/seeds.rb`, usando o mesmo método público desta classe.
- **Sem código de recuperação.** Estas contas existem para exercitar TOTP; um código de recuperação em conta de dev com papel de aprovação é fator estático a mais sem ganho. `Mfa::Enroll` **não** é usado: a classe escreve `otp_secret` e `otp_enabled` direto, com o segredo fixo.
- As duas revisoras são **só** revisoras: nenhuma delas tem `protocol_author`, para que nenhuma seja contribuinte do rascunho e as duas contem como elegíveis.

- [ ] **Step 1: Branch**

```bash
cd apps/api && /opt/homebrew/bin/git checkout -b feat/dev-signature-crew
```

- [ ] **Step 2: Escrever a spec que falha**

`spec/lib/signature_crew_spec.rb`:

```ruby
require "rails_helper"
require Rails.root.join("lib/signature_crew").to_s

# A semente que torna o ciclo assinado percorrível em dev: autor, duas
# revisoras e publisher com TOTP fixo, mais um rascunho criado PELO command de
# autoria — é o command que grava a contribuição e o digest de que as
# assinaturas dependem. Roda na conexão da cidade corrente (TEST_CITY_A),
# como spec/lib/dashboard_demo_spec.rb.
RSpec.describe SignatureCrew do
  let(:slug) { "cidade-teste" }
  let(:password) { "dev-password" }

  def seed! = described_class.seed_current_city(slug: slug, password: password)

  def user(email_prefix) = User.find_by(email_address: "#{email_prefix}@#{slug}.demo")

  it "cria as quatro contas com o papel de cada uma" do
    out = seed!

    expect(out[:accounts].map { |a| a[:email] }).to eq([
      "autor@#{slug}.demo", "revisora1@#{slug}.demo", "revisora2@#{slug}.demo", "publisher@#{slug}.demo"
    ])
    expect(user("autor").memberships.active.pluck(:role)).to eq([ "protocol_author" ])
    expect(user("revisora1").memberships.active.pluck(:role)).to eq([ "protocol_reviewer" ])
    expect(user("revisora2").memberships.active.pluck(:role)).to eq([ "protocol_reviewer" ])
    expect(user("publisher").memberships.active.pluck(:role)).to eq([ "protocol_publisher" ])
  end

  it "deixa cada conta com TOTP pronto e com a senha de dev" do
    seed!

    described_class::MEMBERS.each do |member|
      account = user(member[:email_prefix])
      expect(account.mfa_enrolled?).to be(true)
      expect(account.authenticate(password)).to be_truthy
    end
  end

  it "devolve o otpauth de cada conta, com o segredo da conta" do
    out = seed!

    uri = out[:accounts].first[:otpauth_uri]
    expect(uri).to start_with("otpauth://totp/")
    secret = URI.decode_www_form(URI.parse(uri).query).to_h["secret"]
    expect(secret).to eq(user("autor").otp_secret)
  end

  it "não sobrescreve um TOTP já existente" do
    seed!
    account = user("autor")
    account.update!(otp_secret: ROTP::Base32.random)
    mine = account.reload.otp_secret

    seed!

    expect(user("autor").otp_secret).to eq(mine)
  end

  it "cria a versão 2 em rascunho pelo command, com a contribuição do autor" do
    out = seed!

    draft = ProtocolDefinition.find_by(name: described_class::PROTOCOL_NAME, version: 2)
    expect(out[:draft]).to include(name: described_class::PROTOCOL_NAME, version: 2, status: "draft")
    expect(draft.status).to eq("draft")
    expect(ProtocolContribution.where(protocol_definition: draft).pluck(:actor_kind, :actor_id))
      .to eq([ [ "user", user("autor").id ] ])
    expect(ProtocolContribution.find_by(protocol_definition: draft).content_digest).to eq(draft.content_digest)
  end

  it "as duas revisoras são elegíveis para assinar o rascunho, e o autor não" do
    seed!
    draft = ProtocolDefinition.find_by(name: described_class::PROTOCOL_NAME, version: 2)

    expect(Protocols::Signatures.eligible_reviewer_count(draft)).to eq(2)
    expect(Protocols::Signatures.valid_signer_ids(draft, purpose: "publication")).to eq([])
  end

  it "é idempotente: a segunda execução não duplica conta, papel nem contribuição" do
    first = seed!

    expect { seed! }
      .to not_change { User.count }
      .and not_change { Membership.count }
      .and not_change { ProtocolContribution.count }
      .and not_change { ProtocolDefinition.where(name: described_class::PROTOCOL_NAME).count }
    expect(seed!).to eq(first)
  end

  it "não mexe no protocolo ativo da cidade" do
    active = ProtocolDefinition.create!(name: described_class::PROTOCOL_NAME, version: 1, status: "active",
                                        definition: protocol_definition_hash(name: described_class::PROTOCOL_NAME))

    seed!

    expect(active.reload.status).to eq("active")
  end
end
```

`not_change` é o negativo composto do RSpec (`.to not_change { }`), que já é usado no projeto — se não estiver disponível, escreva com `expect { }.not_to change { }` em blocos separados, sem perder nenhuma das quatro verificações.

O `protocol_definition_hash` vem de `spec/support/protocol_signatures.rb`. A definição que a classe usa para a versão 2 deve passar no portão (`Protocols::Gate`): copie a forma do protocolo que `db/seeds.rb` já cria (`protocol_defn` lá) ou do helper de spec, ajustando `version` para 2 e mudando **algo visível** no conteúdo (por exemplo o texto de um passo, com o sufixo "(v2)"), para a versão nova não ser idêntica à v1.

- [ ] **Step 3: Rodar e ver falhar**

Da raiz do monorepo:
```bash
docker compose exec -T api bundle exec rspec spec/lib/signature_crew_spec.rb
```
Esperado: FAIL — `uninitialized constant SignatureCrew`.

- [ ] **Step 4: Implementar**

`lib/signature_crew.rb`:

```ruby
require "rotp"

# Semente de dev do ciclo assinado (plano 2026-09-23). Cria, DENTRO da conexão
# da cidade corrente, as quatro contas que o ciclo exige — autor, duas
# revisoras e publisher — com TOTP já pronto, e um rascunho de versão 2 criado
# PELO command de autoria.
#
# Por que pelo command: `Protocols::SaveDraft` grava a ProtocolContribution do
# autor e o content_digest. É disso que as assinaturas dependem: quem editou
# não assina, e editar de novo invalida assinatura pelo digest. Um
# `ProtocolDefinition.create!` na mão produziria uma versão que parece certa e
# um estado de assinatura que não existe no domínio.
#
# Por que segredo FIXO: sem ele, cada reset do banco de dev obrigaria a
# reescanear quatro autenticadores. Mesmo desenho do operador em db/seeds.rb —
# valor de dev, override por env, e nunca sobrescreve segredo já existente.
# Nada aqui é para ambiente publicado; quem garante isso é o chamador
# (db/seeds.rb, guardado por Rota.deployed?).
class SignatureCrew
  PROTOCOL_NAME = "triage-respiratoria".freeze

  MEMBERS = [
    { email_prefix: "autor",     role: "protocol_author",    secret_env: "DEV_AUTHOR_OTP_SECRET",
      default_secret: "KRSXG5CTMVRXEZLUKRSXG5CTMVRXEZLU" },
    { email_prefix: "revisora1", role: "protocol_reviewer",  secret_env: "DEV_REVIEWER1_OTP_SECRET",
      default_secret: "MFRGGZDFMZTWQ2LKMFRGGZDFMZTWQ2LK" },
    { email_prefix: "revisora2", role: "protocol_reviewer",  secret_env: "DEV_REVIEWER2_OTP_SECRET",
      default_secret: "NBSWY3DPFQQFO33SNBSWY3DPFQQFO33S" },
    { email_prefix: "publisher", role: "protocol_publisher", secret_env: "DEV_PUBLISHER_OTP_SECRET",
      default_secret: "OBQXG43XN5ZGILLQOBQXG43XN5ZGILLQ" }
  ].freeze

  class << self
    # Resumo: { accounts: [ { email:, role:, otpauth_uri: } ], draft: {...} | nil }
    def seed_current_city(slug:, password:)
      accounts = MEMBERS.map { |member| ensure_member(member, slug: slug, password: password) }
      { accounts: accounts, draft: ensure_draft(slug: slug) }
    end

    # Também usada pelo db/seeds.rb para dar TOTP ao admin municipal que ele
    # mesmo cria (a tela Equipe exige step-up para conceder papel).
    def ensure_totp(user, secret_env:, default_secret:)
      return user if user.mfa_enrolled?

      user.update!(otp_secret: ENV.fetch(secret_env, default_secret), otp_enabled: true)
      user
    end

    def otpauth_uri(user)
      ROTP::TOTP.new(user.otp_secret, issuer: "Rota Saúde (dev)").provisioning_uri(user.email_address)
    end

    private

    def ensure_member(member, slug:, password:)
      user = User.find_or_initialize_by(email_address: "#{member[:email_prefix]}@#{slug}.demo")
      user.password = password
      user.save!
      Membership.find_or_create_by!(user: user, role: member[:role]) { |m| m.granted_at = Time.current }
      ensure_totp(user, secret_env: member[:secret_env], default_secret: member[:default_secret])

      { email: user.email_address, role: member[:role], otpauth_uri: otpauth_uri(user) }
    end

    # O rascunho nasce do autor, pelo command. Já existindo, nada é reescrito:
    # reescrever mudaria o digest e derrubaria assinatura que você acabou de
    # coletar na mão.
    def ensure_draft(slug:)
      author = User.find_by!(email_address: "autor@#{slug}.demo")
      existing = ProtocolDefinition.find_by(name: PROTOCOL_NAME, version: 2)
      return summarize(existing) if existing

      result = Protocols::SaveDraft.call(definition: draft_definition, by: author)
      return nil unless result.ok?

      summarize(result.payload[:protocol_definition])
    end

    def summarize(record)
      { name: record.name, version: record.version, status: record.status }
    end

    # A definição vem do MESMO template que o provisionamento e o seed usam
    # (CityTemplates.protocol → config/city_templates/triage_respiratoria.json),
    # que já passa no portão. Só a versão muda, e o prompt do primeiro passo
    # ganha um sufixo para a diferença entre v1 e v2 ficar visível na tela.
    def draft_definition
      definition = CityTemplates.protocol.fetch(:definition).deep_dup
      definition["version"] = 2
      first_step = definition.fetch("steps").first
      first_step["prompt"] = "#{first_step['prompt']} (v2)"
      definition
    end
  end
end
```

Confira em `app/services/city_templates.rb` que `protocol` devolve `{ name:, definition: }` lendo o JSON do template, e que a chave dos passos é `steps` com `prompt` — ajuste só os nomes de chave, se o arquivo divergir; não invente esquema. Se `PROTOCOL_NAME` não for igual ao `name` do template, use o `name` do template como fonte da verdade e deixe `PROTOCOL_NAME` derivar dele.

**Atenção ao `SaveDraft`:** ele exige `Current.city` presente e checa `ProtocolPolicy#author?` no ator — por isso o rascunho é criado com a conta de autor, que acaba de receber o papel, e por isso `seed_current_city` precisa rodar dentro de `Current.set(city:)`. Se `result` não vier `ok?`, devolva `nil` e deixe o chamador avisar; não levante.

- [ ] **Step 5: Rodar e ver passar**

```bash
docker compose exec -T api bundle exec rspec spec/lib/signature_crew_spec.rb
```
Esperado: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add lib/signature_crew.rb spec/lib/signature_crew_spec.rb
/opt/homebrew/bin/git commit -m "feat: seed a signing crew and a draft for dev cities" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: ligar no seed, documentar e provar em dev

**Files:**
- Modify: `apps/api/db/seeds.rb`, `apps/api/README.md`
- Modify: `docs/superpowers/specs/2026-09-22-dashboard-protocol-signatures-design.md`

**Interfaces:**
- Consumes: `SignatureCrew.seed_current_city(slug:, password:)` e `SignatureCrew.ensure_totp(user, secret_env:, default_secret:)` (Task 1).
- Produces: `bin/rails db:seed` deixa cada cidade de dev com as quatro contas, o admin municipal com TOTP e a versão 2 em rascunho; e imprime, por cidade, uma linha por conta com o `otpauth://`.

- [ ] **Step 1: Ligar no `db/seeds.rb`**

Dentro do `CityConnection.with(city)` que já existe, **depois** do bloco que cria `muni_admin` e a membership dele, e **depois** do bloco que cria o protocolo `triage-respiratoria` v1 (o rascunho é a versão 2 do mesmo nome):

```ruby
        # ── Elenco do ciclo assinado (plano 2026-09-23) ───────────────────────
        # Autor, duas revisoras e publisher com TOTP fixo, mais a versão 2 em
        # rascunho criada pelo command de autoria — sem isso, exercitar
        # assinatura no navegador exige cadastrar quatro autenticadores à mão a
        # cada reset. O admin municipal também ganha TOTP: a tela Equipe pede
        # step-up para conceder papel.
        SignatureCrew.ensure_totp(muni_admin, secret_env: "DEV_MUNI_ADMIN_OTP_SECRET",
                                              default_secret: "JBSWY3DPEHPK3PXPJBSWY3DPEHPK3PXP")
        crew = SignatureCrew.seed_current_city(slug: slug, password: password)
        puts "[seeds] admin ...... #{muni_admin.email_address} / #{password} + MFA → #{SignatureCrew.otpauth_uri(muni_admin)}"
        crew[:accounts].each do |account|
          puts "[seeds] #{account[:role].ljust(18)} #{account[:email]} / #{password} + MFA → #{account[:otpauth_uri]}"
        end
        puts crew[:draft] ? "[seeds] rascunho ... #{crew[:draft][:name]} v#{crew[:draft][:version]}" \
                          : "[seeds] rascunho ... NÃO criado (veja o retorno do command)"
```

Acrescente `require Rails.root.join("lib/signature_crew").to_s` no topo do arquivo, como o seed já faz com o que precisa de lib (confira como `lib/dashboard_demo` é carregado; se houver autoload de `lib`, não duplique o require).

Atualize o comentário de topo do `db/seeds.rb`, que lista o que o seed cria por cidade.

- [ ] **Step 2: Rodar o seed em dev e conferir**

Da raiz do monorepo, com o stack de pé:

```bash
docker compose exec -T api bin/rails db:seed
docker compose exec -T api bin/rails db:seed   # segunda vez: idempotente
```
Esperado: as linhas das cinco contas e do rascunho nas duas execuções, sem erro e sem duplicata. **Não copie os `otpauth://` para o relatório** — diga apenas que saíram.

- [ ] **Step 3: Suíte completa** (worker parado; ver Global Constraints). Esperado: 0 falhas.

- [ ] **Step 4: Percorrer o ciclo no navegador**

Este é o objetivo do plano: prove que o ciclo anda. Abra `http://curitiba.localhost:5175/dashboard/` e, cada vez, entre com a conta do papel (senha de dev; o código vem do autenticador que você cadastrou com o `otpauth://` do seed):

1. **autor** → Protocolos → a versão 2 em rascunho oferece "Enviar para revisão". Envie.
2. **revisora1** → o KPI "Aguardando sua assinatura" conta 1 e filtra; assine a publicação.
3. **revisora2** → assine a publicação. A coluna Publicação vai a 2/2.
4. **publisher** → "Publicar" está habilitado (antes das duas assinaturas estava desabilitado, com o motivo). Publique.
5. **revisora1 e revisora2** → assinem a ativação.
6. **publisher** → "Ativar". Depois disso a v2 aparece **em uso** e a v1 sai de uso.
7. Na v2 em uso: **"Aposentar" não aparece** e **"Reverter" aparece**. Não é preciso reverter; se reverter, a v1 volta a valer.
8. **autor**, na versão em revisão: "Assinar publicação" aparece desabilitada com "você editou esta versão".

Registre o que viu em cada passo. Se um passo não puder ser feito, diga qual e por quê, sem inventar conta nem credencial.

- [ ] **Step 5: README**

Em `apps/api/README.md`, onde as contas de dev estão documentadas (a linha do `admin@curitiba.demo`), acrescente as contas novas, os papéis, a senha e o nome das variáveis de override dos segredos. Diga em uma frase que o TOTP é fixo em dev para sobreviver a reset, que o seed imprime o `otpauth://` de cada conta, e que nada disso roda em ambiente publicado.

- [ ] **Step 6: Registrar no spec (repo docs)**

Em `docs/superpowers/specs/2026-09-22-dashboard-protocol-signatures-design.md`, na subseção de §5.2 que registra as pendências, troque "o ciclo nunca rodou fim a fim no navegador" pelo estado novo: a semente de dev (`SignatureCrew`) cria o elenco e o rascunho, e o ciclo foi percorrido — ou, se algum passo ficou de fora, qual.

```bash
cd ../../docs
/opt/homebrew/bin/git add superpowers/specs/2026-09-22-dashboard-protocol-signatures-design.md
/opt/homebrew/bin/git commit -m "docs: record the dev signing crew seed" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

- [ ] **Step 7: Commit do api**

```bash
/opt/homebrew/bin/git add db/seeds.rb README.md
/opt/homebrew/bin/git commit -m "feat: seed the signing crew into every dev city" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Fora deste plano

- Conta de dev para o mantenedor da API de manutenção (já existe caminho próprio: `rake maintainer:invite`).
- Semente de assinaturas já coletadas: as assinaturas são o que se quer exercitar à mão.
- Qualquer mudança em `Protocols::*` ou nas telas.
- Aviso por e-mail ao cadastrar ou trocar autenticador (pendência de go-live).
