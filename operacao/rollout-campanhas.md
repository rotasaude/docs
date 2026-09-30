# Rollout — Campanhas (módulo 12, ADR 0024)

A migração de cidade `20260929100001_create_campaigns` é **só de expansão**:
tabelas `campaigns`, `campaign_recipients` e `citizen_contact_preferences`;
coluna `city_profile.campaigns_sms_enabled` (default `false`); triggers
`campaigns_frozen_after_send` e `campaign_recipients_append_only`; o CHECK
`ck_memberships_role` passa a aceitar `campaign_manager`.

O deploy **não envia SMS**: toda cidade nasce com a chave desligada e a
gateway de produção nasce não configurada (`config.x.sms_gateway = nil`).

## Ordem

1. **api** — publicar a imagem (a partir de `4425347`) e rodar
   `bin/rails city:migrate:all` **da imagem nova** antes de cortar o tráfego.
   Nunca migrar fora do rake (a cidade trava em 503). O **worker** precisa
   rodar a mesma imagem: com worker antigo, o envio falha com
   `ActiveJob::UnknownJobClassError` (`Campaigns::DispatchJob`) e a campanha
   fica em `sending` — o `Campaigns::DueJob` reenfileira sozinho a campanha
   presa entre 10 min e 24 h, depois que o worker novo sobe.
2. **dashboard** (`c69a932` ou posterior).
3. **wpda** (`32e6420` ou posterior). O servidor precisa servir `/wpda/avisos`
   e `/wpda/preferencias` pelo `index.html` (o `nginx.conf` do wpda já faz
   `try_files … /wpda/index.html`).

## Conferência depois do deploy

- `GET /campaigns/sms_setting` responde `{ enabled: false,
  gateway_configured: false }`.
- Em Equipe, "Gestor de campanhas" aparece entre os papéis e pede step-up.
- Uma campanha de teste (com público real de pelo menos 5 telefones) chega à
  caixa `/wpda/avisos`, e o painel dela só mostra números.
- A recorrência `campaigns_due` aparece no Solid Queue do worker.

## Ligar o SMS (gate de go-live)

Só depois de:

1. contratar o provedor e escrever o backend dele em `SmsGateway`;
2. resolver o risco de SMS duplicado: hoje o `SmsBatchJob` mantém a transação
   aberta durante a chamada ao provedor, e um `retry_on Deadlocked` ou uma
   queda depois do envio pode reenviar o lote (até 99 mensagens). Antes de
   ligar, o backend real precisa de chave de idempotência por destinatário ou
   o envio precisa sair da transação;
3. definir no backend real que `SmsGateway::Unavailable` significa só "sem
   provedor": falha transitória do provedor deve levantar outro erro (vai para
   a retentativa e depois `failed`); senão uma queda momentânea marca a
   campanha inteira como `unavailable`, sem reenvio;
4. testar em staging com a chave ligada numa cidade de teste.

SMS parado: o `Campaigns::DueJob` reenfileira sozinho os SMS `pending` há mais
de 10 min e os `deferred` dentro da janela, por até 48 h depois do envio da
campanha. Depois disso desiste; o que sobrar aparece no painel como pendente e
nas falhas do Solid Queue.

Então cada cidade liga a chave em Comunicação → Campanhas (`municipal_admin`,
com step-up). Desligar a chave não interrompe SMS já pendentes de uma
campanha enviada antes (a chave vale no congelamento).

## Rollback

- Código: voltar as imagens; as tabelas novas ficam sem uso.
- Papel: revogar o `campaign_manager` pela tela Equipe. O `down` da migração
  **falha** se já houver membership `campaign_manager` (memberships não se
  apagam), de propósito.

## Dev

A migração deixa os bancos de dev e de teste à frente de uma main antiga. Ao
voltar de um branch sem o módulo: DROP de `rota_saude_test_city_a`/`_b` e
`bin/rails city:test_databases`. A semente cria `campanhas@<cidade>.demo`
(TOTP fixo, segredo em `lib/campaign_crew.rb`) e cidadãos para cada critério,
incluindo um telefone com dois CPFs.
