# Aviso por e-mail quando um código de recuperação é usado — design

**Data:** 2026-09-23
**Status:** aprovado em conversa (2026-09-23)
**Afeta:** `apps/api` (um método novo no `SecurityMailer` e um ponto de chamada). Nada de frontend.

**Origem:** pendência registrada em `2026-09-23-authenticator-change-notice-design.md` §9.3. O aviso de mudança de autenticador fechou o silêncio de `POST /mfa/confirm`, mas sobrou o caminho irmão: **senha mais um código de recuperação** dá uma sessão com step-up válido por 5 minutos. Com ela é possível assinar, publicar, ativar, aposentar, reverter e gerir papéis — sem tocar no autenticador e, hoje, sem nenhum aviso ao dono da conta.

## 1. Escopo

**Dentro:** um evento, no único lugar onde o usuário de cidade consome código de recuperação — `POST /mfa/step_up`, quando `Mfa::Verify.consume_recovery_code` tem sucesso (`MfaController`).

**Fora:**
- o operador de plataforma, que consome código de recuperação no login do console (`Operators::SessionsController`): conta de outra natureza, com outro destinatário e outro texto;
- o mantenedor, que não tem código de recuperação por desenho;
- step-up com TOTP, que é o caminho normal e não merece e-mail;
- qualquer mudança no comportamento do step-up. O aviso **observa**, não decide.

## 2. Decisões

1. **Um método novo no `SecurityMailer`, `recovery_code_used`**, e não um `kind` a mais em `authenticator_changed`: o texto é outro (nada foi cadastrado nem trocado) e tem um dado próprio, a contagem restante.
2. **O e-mail diz quantos códigos sobraram**, contados depois do consumo, e avisa quando chega a zero. É o que transforma o aviso em ação: "restam 9" é uso normal, "restam 2" é lista sendo consumida, "restam 0" é conta sem rede de segurança. A contagem não é segredo; os códigos em si estão com hash no banco.
3. **Mesmas regras do aviso irmão** (§2 e §3 daquele spec, valendo aqui sem repetição de argumento): só valores simples (R42), hora e IP, sem link, sem segredo, IP degradando para "desconhecido" se a checagem de IP forjado levantar, enfileiramento depois do commit, e **o aviso nunca faz a ação falhar**.
4. **Um aviso por código consumido.** Um step-up com TOTP não avisa nada; um step-up recusado não avisa nada.

## 3. Como funciona

**Mailer:** `SecurityMailer#recovery_code_used(email_address:, city_name:, ip_address:, occurred_at:, remaining:)`, com view HTML e texto.

- `remaining` é um inteiro, contado **depois** do consumo (`Mfa::Verify.consume_recovery_code` já regrava a lista);
- `occurred_at` viaja como string ISO 8601 e a view exibe em `America/Sao_Paulo`, como os outros mailers.

**Ponto de chamada:** `MfaController#step_up`. Hoje o método tenta TOTP e, se não for TOTP válido, tenta código de recuperação. O aviso sai **só** no segundo caminho, e só quando ele tem sucesso — o que exige distinguir "passou por TOTP" de "passou por código de recuperação" onde hoje as duas coisas levam ao mesmo `true`.

**Ordem:** o carimbo da sessão (`mfa_verified_at`) acontece primeiro, e o enfileiramento depois — um aviso não pode ser enviado se a sessão não foi carimbada.

**Falha ao enfileirar:** capturada, registrada em log com o id do usuário (nunca o e-mail nem o IP), engolida; a resposta segue `200`. Reusa o mesmo caminho defensivo do aviso irmão, inclusive o IP degradado.

## 4. O que o e-mail diz

Assunto: `[rota-saúde] Código de recuperação usado`.

Corpo, nesta ordem:

1. o que aconteceu, com a cidade: "Um código de recuperação da sua conta na cidade de <cidade> foi usado para aprovar uma ação sensível.";
2. quando e de onde: data e hora em `America/Sao_Paulo`, e o IP;
3. o que fazer se não foi você: "Se não foi você, fale agora com o administrador municipal — com este acesso é possível assinar e publicar protocolo em seu nome.";
4. quantos sobraram: "Restam N códigos de recuperação." — e, com `remaining` igual a zero, "Não resta nenhum código: a partir de agora só o autenticador aprova ações sensíveis."

Sem link, sem anexo, sem nenhum código.

## 5. Privacidade

- Destinatário é o dono da conta; conteúdo sobre ele. E-mail e IP de staff, sob base de operação do serviço (ADR 0013); nenhum dado de cidadão.
- Nenhum código de recuperação, nem parte dele, aparece no e-mail — só a contagem.
- Valem os dois limites já registrados no aviso irmão (§9.2 dele): os argumentos do envio ficam persistidos na fila do banco da cidade e aparecem em log SQL no nível `debug`.

## 6. Estratégia de teste

- **Mailer:** assunto e corpo (HTML e texto) com cidade, hora de Brasília, IP, a frase de ação e a contagem; `remaining: 0` acrescentando a frase do "só o autenticador"; `remaining: 9` **não** trazendo essa frase; nenhuma das versões contendo código, segredo, `otpauth` ou link.
- **Requisição (`POST /mfa/step_up`):**
  - step-up com código de recuperação → **um** aviso, com a contagem restante correta (o número de códigos que sobraram no banco);
  - step-up com TOTP → **nenhum** aviso;
  - código inválido (nem TOTP nem recuperação) → nenhum aviso, e a recusa de sempre;
  - código de recuperação já usado → nenhum aviso (o consumo falha);
  - último código consumido → aviso com `remaining: 0`;
  - uma falha no enfileiramento não muda a resposta `200` nem descarimba a sessão.
- Um exemplo garantindo que o TOTP continua sendo consumido uma vez só e que a distinção "TOTP vs recuperação" não mudou o comportamento de nenhum dos dois caminhos.

## 7. Entrega

Um plano, duas tarefas:

1. `SecurityMailer#recovery_code_used` e as duas views, com o spec de mailer;
2. o ponto de chamada em `MfaController#step_up` — incluindo separar "passou por TOTP" de "passou por código de recuperação" —, com o spec de requisição.

## 8. Fora de escopo

- Aviso ao operador de plataforma.
- Reemissão de códigos de recuperação, e qualquer tela para isso.
- Bloquear step-up por código de recuperação, ou limitar quantos podem ser usados por período.
- Aviso de papel privilegiado concedido ou revogado, e de senha alterada (seguem fora, como no spec irmão).
