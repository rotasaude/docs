# Provisionar canal WhatsApp de uma cidade

Liga o número de WhatsApp (Meta Cloud API) de uma cidade já provisionada. O
canal (`CityChannel`) mora no banco de **plataforma**. As mensagens moram no
banco da cidade. O `phone_number_id` do webhook é o que liga uma à outra.

## Quando aplicar

- A cidade está `active` e a Meta liberou o número dela.
- Troca de número: registre o canal novo. Hoje não há edição de canal. Veja
  **Rollback**.
- Não se aplica às cidades que atendem só pelo canal web (ADR 0017). Nelas, o
  cidadão entra pelo wpda e não existe canal WhatsApp.

## Pré-requisitos

**Na Meta (uma vez por plataforma).** O App Secret é compartilhado entre todas
as cidades, então a suspensão do App derruba o WhatsApp de todas. Esse blast
radius está assumido no módulo 01.

1. O App da Meta tem o produto WhatsApp.
2. O webhook do App aponta para `https://<host do api>/webhooks/whatsapp`, com o
   campo `messages` assinado.
3. O verify token configurado na Meta é igual a `WHATSAPP_VERIFY_TOKEN`.
4. O App Secret é igual a `WHATSAPP_APP_SECRET`. É com ele que o api confere o
   `X-Hub-Signature-256` de cada POST. Assinatura errada ou ausente devolve 401,
   e nada é gravado.
5. Em produção, os dois vêm do 1Password
   (`op://<vault>/whatsapp/verify_token` e `.../app_secret`), pelo
   `deploy/production/secrets`. Veja `apps/api/deploy/SECRETS.md`.

**Na Meta (por cidade).**

1. O número está registrado numa WABA.
2. Anote o `phone_number_id`, o `waba_id` e o número exibido (`display_phone_number`).
3. Gere um access token de System User com permissão de envio nessa WABA. O
   token é **da cidade**: fica cifrado no `CityChannel` com a chave de
   plataforma. O `WHATSAPP_ACCESS_TOKEN` do ambiente não é usado no envio.

**No Rota Saúde.** Você precisa de um operador de plataforma com TOTP no
console (`admin.*`), e a cidade precisa estar `active` (`GET /cities/:id`).

## Passos

1. Entre no console (`admin.*`), em **Setup → Registrar canal**.
2. Escolha a cidade e preencha `phone_number_id`, `waba_id`, número exibido e
   access token.
3. Envie. A tela chama `POST /cities/:id/channel`, que responde `201` com o
   canal ativo. O token nunca volta na resposta nem vai para o log. A auditoria
   (`Platform.audit`) registra o registro sem o token.

Alternativa sem console, dentro do container do api:

```bash
CITY_SLUG=<slug> PHONE_NUMBER_ID=<id> WABA_ID=<waba> DISPLAY_PHONE_NUMBER=<+55...> ACCESS_TOKEN=<token> bin/rails channels:register
```

Recusas esperadas (422):

- cidade que não está `active`;
- token vazio;
- `phone_number_id` já registrado, inclusive em outra cidade.

## Validação

1. **Handshake.** Na Meta, "verificar e salvar" o webhook devolve o `challenge`.
   Um 403 aqui é verify token divergente.
2. **Entrada.** Mande uma mensagem de teste para o número. O painel **Ingestão**
   do dashboard da cidade deve contar a mensagem em "Mensagens recebidas
   (WhatsApp)".
3. **Número não reconhecido.** Em **Setup → Números desconhecidos** do console,
   o `phone_number_id` **não** pode aparecer. Se ele aparecer ali, a mensagem
   chegou sem canal correspondente: o registro falhou, foi feito com o id
   errado, ou a Meta está mandando de outra WABA.
4. **Saída.** O cidadão recebe a resposta. Um 401/403 da Meta no
   `SendWhatsappJob` indica token sem permissão na WABA.

## Rotação do access token

Zero-downtime: o envio lê o token a cada chamada. Hoje a rotação é só por
rake, e o ator precisa ser um `municipal_admin` da cidade (F-01.9):

```bash
CITY_SLUG=<slug> ROTATE_TOKEN=<novo token> ACTOR_EMAIL=<admin da cidade> bin/rails channels:rotate_token
```

Depois de rotacionar, revogue o token antigo na Meta e repita a validação 4.

## Rollback

- **Não existe edição nem desativação de um canal isolado.** Canal só é
  desativado no offboard da cidade (`CityLifecycle::Offboard` marca os canais
  da cidade como inativos).
- Mensagem para um canal inativo é descartada e logada. Ela não é gravada na
  cidade e não aparece como número desconhecido.
- Canal registrado com dado errado: corrigir hoje exige intervenção no banco de
  plataforma (`city_channels`), feita por quem opera o banco e registrada no
  incidente. Se isso ficar recorrente, vira funcionalidade: editar e desativar
  canal, desligando o WhatsApp por cidade como o ADR 0017 prevê.

## ADRs relacionados

- 0007: borda do WhatsApp, ack, idempotência, roteamento por `phone_number_id`.
- 0013: custódia do payload e do access token.
- 0017: WhatsApp desligável por cidade (canal web do cidadão).
- 0020: banco por cidade. O canal fica na plataforma, a mensagem fica na cidade.
