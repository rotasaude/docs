# Aviso por e-mail quando o autenticador muda — design

**Data:** 2026-09-23
**Status:** aprovado em conversa (2026-09-23)
**Afeta:** `apps/api` (um mailer novo e um ponto de chamada). Nada de frontend.

**Origem:** pendência de go-live registrada no spec do autenticador pendente (`2026-09-22-pending-authenticator-design.md` §4, "o limite que continua"). Duas brechas seguem abertas por desenho, e as duas são invisíveis para o dono da conta:

1. numa conta que **nunca** cadastrou TOTP, quem tiver a senha cadastra o próprio autenticador — não há fator anterior a pedir;
2. numa conta **já cadastrada**, senha mais **um** código de recuperação bastam: `step_up` aceita código de recuperação, e daí `enroll` + `confirm` trocam o segundo fator e substituem os códigos restantes.

Nos dois casos o sequestro é silencioso. Este spec fecha o silêncio: a conta recebe um e-mail dizendo o que mudou, quando e de onde.

## 1. Escopo

**Dentro:** dois eventos do segundo fator do usuário de cidade, ambos no momento em que `POST /mfa/confirm` promove o pendente:

| evento | quando |
|---|---|
| autenticador **cadastrado** | a conta não tinha autenticador ativo antes da confirmação |
| autenticador **trocado** | a conta tinha, e a confirmação substituiu |

**Fora:**
- papel privilegiado concedido ou revogado (destinatário e texto diferentes; trabalho próprio);
- senha alterada (`PasswordsController` já manda o e-mail de redefinição, mas não um "sua senha mudou");
- mantenedor e operador, que têm cadastro próprio (`Mfa::Enroll`, convite);
- qualquer mudança de comportamento no cadastro em si. O aviso **observa**, não decide.

## 2. Decisões

1. **Só o segundo fator** (§1). É o que fecha as duas brechas nomeadas na origem.
2. **O e-mail diz o que aconteceu, quando, de onde e em qual cidade**, e o que fazer se não foi a pessoa: falar com o administrador municipal. Traz a **hora** e o **IP** de onde a confirmação partiu.
   - Sem link. Um e-mail de segurança com link é o formato que o phishing imita, e treinar a pessoa a clicar nele é o contrário do objetivo.
   - Sem segredo, sem código, sem `otpauth://` — nada que sirva para cadastrar nada.
3. **O aviso nunca faz a ação falhar.** É enfileirado depois de a promoção comitar, e uma falha ao enfileirar é registrada em log e engolida: um servidor de e-mail fora do ar não pode impedir alguém de cadastrar o próprio autenticador (nem deixar o usuário sem saber se a troca valeu).
4. **Um mailer novo, `SecurityMailer`**, em vez de acrescentar ação ao `PasswordMailer`: o assunto é outro (conta, não senha) e o destinatário é sempre o dono da conta.
5. **Só valores simples** para o mailer (a convenção R42 que `PasswordMailer`, `InvitationMailer` e `AlertMailer` já seguem): `deliver_later` roda no worker, fora da conexão da cidade, onde um `User` (GlobalID) não desserializa.

## 3. Como funciona

**Mailer:** `SecurityMailer#authenticator_changed(email_address:, kind:, city_name:, ip_address:, occurred_at:)`, com versão HTML e texto, no layout que já existe.

- `kind` é `"enrolled"` ou `"replaced"` — decide o assunto e a frase de abertura;
- `occurred_at` chega como string ISO 8601 (mesmo padrão de `AlertMailer`);
- `ip_address` é o IP da requisição que confirmou.

**Ponto de chamada:** `MfaController#confirm`. Antes de chamar `Mfa::PendingEnrollment.confirm`, o controller guarda se a conta já tinha autenticador ativo (`Current.user.mfa_enrolled?`) — depois da promoção ela sempre tem, e essa é a única forma de distinguir cadastro de troca sem mudar o command. Com o resultado `:ok`, e só com ele, o e-mail é enfileirado.

**Ordem e contexto:** o `confirm` do command roda a promoção na própria transação e devolve; o enfileiramento acontece depois, já comitado, dentro da requisição da cidade — que é onde `CityMailDeliveryJob` exige estar (`PlatformQueue.check!`). Nada de enfileirar dentro da transação: um rollback mandaria um aviso de algo que não aconteceu.

**Falha ao enfileirar:** capturada, registrada em log com o id do usuário (nunca o e-mail nem o IP) e engolida. A resposta do `confirm` continua `200`.

## 4. O que o e-mail diz

Assunto: `[rota-saúde] Autenticador cadastrado` ou `[rota-saúde] Autenticador trocado`.

Corpo, nesta ordem:

1. o que aconteceu, com a cidade: "O autenticador da sua conta na <cidade> foi cadastrado" / "…foi trocado";
2. quando, no fuso da aplicação, e de qual IP;
3. o que fazer se não foi você: "Se não foi você, fale agora com o administrador municipal — quem tem o autenticador aprova protocolo em seu nome.";
4. na troca, uma linha a mais: os códigos de recuperação anteriores deixaram de valer.

Sem saudação personalizada além do endereço, sem link, sem anexo.

## 5. Privacidade

- Destinatário é o próprio dono da conta, e o conteúdo é sobre ele. E-mail de staff e IP de staff, sob base de operação do serviço (ADR 0013); nenhum dado de cidadão.
- O IP sai no corpo porque o dono precisa julgar "não fui eu" — foi decisão explícita deste spec.
- Log da falha de enfileiramento não leva e-mail nem IP.
- `CityMailDeliveryJob` já desliga `log_arguments`, então os argumentos do mailer (incluindo o e-mail e o IP) não vão para o log no nível `info` — o padrão de produção. Isso não é o quadro completo: ver Pendências (§9) para o nível `debug` e para a persistência em `solid_queue_jobs`.

## 6. Estratégia de teste

- **Mailer (spec de mailer):** para `kind: "enrolled"` e `"replaced"`, o assunto certo, o endereço certo, e o corpo (HTML e texto) contendo cidade, hora, IP e a frase de "se não foi você"; a versão de troca mencionando os códigos antigos; e **nenhuma** das duas versões contendo `otpauth`, segredo ou código.
- **Requisição (`POST /mfa/confirm`):**
  - conta sem autenticador → um e-mail, com o assunto de cadastro;
  - conta com autenticador (troca) → um e-mail, com o assunto de troca;
  - recusa (`invalid_code`, `enrollment_expired`, `no_pending_enrollment`, `code_reused`) → **nenhum** e-mail;
  - exatamente **um** e-mail por confirmação;
  - uma falha no enfileiramento não muda a resposta `200` nem desfaz a promoção.
- O ambiente de teste usa `delivery_method = :test`, então as asserções são sobre `ActionMailer::Base.deliveries` (ou o matcher que o projeto já usa nas specs de `PasswordMailer`/`AlertMailer` — seguir o que existe).

## 7. Entrega

Um plano, duas tarefas:

1. `SecurityMailer` e as duas views, com o spec de mailer;
2. o ponto de chamada em `MfaController#confirm`, com o spec de requisição, e uma linha no README sobre o aviso.

## 8. Fora de escopo

- Aviso de papel privilegiado concedido ou revogado.
- Aviso de senha alterada.
- Segunda via ou reenvio do aviso.
- Preferência do usuário para desligar o aviso: é aviso de segurança, não boletim.
- Retenção e expurgo de log do provedor de e-mail.

## 9. Pendências

1. **Gate de go-live:** provar `request.remote_ip` atrás do proxy de produção/staging (um `curl` pelo proxy conferindo o IP que o Rails reporta). A verificação em dev mostrou o IP do container do proxy Vite, não o do cliente — se o `X-Forwarded-For` não chegar, a linha do IP fica inútil.
2. **Privacidade, limite conhecido:** os argumentos do job (e-mail e IP) ficam persistidos em `solid_queue_jobs.arguments` no banco da cidade, e sobrevivem em `solid_queue_failed_executions` se a entrega falhar; e aparecem no log SQL em nível `debug`. A frase "não vão para log" do §5 vale para o nível `info` (o padrão de produção), não para todos os níveis nem para o banco.
3. **Follow-up:** senha mais um código de recuperação dá sessão privilegiada por `POST /mfa/step_up` sem tocar no autenticador, e isso NÃO gera aviso. Reusar este mailer com um `kind` novo, avisando quando `Mfa::Verify.consume_recovery_code` tem sucesso no step-up, é o próximo passo natural.
