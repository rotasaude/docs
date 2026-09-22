# Canal web do cidadão (substituto do WhatsApp) — design

**Data:** 2026-09-22
**Status:** aprovado em conversa (2026-09-22), aguardando revisão do texto
**Afeta:** `apps/api` (banco de cada cidade, `ConversationAdvance`, consentimento, rotas novas `/citizen/*`) e `apps/wpda` (triagem em etapas e "minhas triagens").

**Origem:** a Meta passou a cobrar por mensagem. O cidadão entrava pelo WhatsApp (ADR-0007, borda única); queremos uma entrada própria, de interface simples, que faça o mesmo papel **sem mudar o consumo de dados da plataforma**: o que o dashboard, o relatório e os alertas leem continua igual.

Este é o **subprojeto 1** de quatro:

| # | Subprojeto | Depende de |
|---|---|---|
| **1** | **Canal web + identidade declarada** (este spec) | — |
| 2 | Validação presencial na UBS (dashboard): marcar cidadão como verificado, corrigir telefone, histórico completo, exclusão do cadastro | 1 |
| 3 | Confirmação de atendimento / encaminhamento pelo cidadão verificado | 2 + modelo de encaminhamento (não existe hoje; ver módulo 13) |
| 4 | Agendamento | 2 + módulos 08, 09, 10 |

## 1. O ponto de partida

O motor de triagem já é neutro em relação ao canal; o WhatsApp só encosta nas bordas:

| Camada | Acoplada ao WhatsApp? | Onde |
|---|---|---|
| Entrada (webhook, HMAC, cidade por `phone_number_id`) | sim | `Webhooks::WhatsappController`, `Whatsapp::Ingest` |
| O motor relê o JSON bruto da Meta para extrair a resposta | sim | `ConversationAdvance#extract_body` |
| Limites de 3 botões / 10 linhas de lista | sim | `Whatsapp::QuestionElement` |
| Consentimento grava `channel: "whatsapp"` fixo | sim | `GiveConsent`, `ConversationAdvance#handle_awaiting_consent` |
| Cidadão identificado só pelo telefone, uma conversa ativa por telefone | sim | `Conversation.phone`, `idx_conversations_active_phone` |
| Envio (`SendWhatsappJob`, janela de 24h, templates) | sim | `Whatsapp::Outbound`, `Whatsapp::SessionWindow` |
| Protocolos, `Triage`, `Consent`, `DomainEvents`, relatório, métricas, alerta à prefeitura | **não** | `app/protocols/*`, `CompleteTriage` |
| Resposta do sistema (`Messaging::Reply`: texto, botões, lista) | **não** | `app/messaging/reply.rb` |

## 2. Decisões

1. **A web substitui o WhatsApp; o código do WhatsApp fica, desligado por cidade** (`CityChannel` inativo). Não há cidade em produção, então não há conversa antiga a migrar. Se a política de preço da Meta mudar, religa-se o canal.
2. **Identidade em dois níveis.** `declared`: CPF declarado (só dígito verificador) + telefone confirmado por OTP. `verified`: documento conferido presencialmente na UBS (subprojeto 2). Este spec só cria o nível `declared`, mas o campo nasce com os dois valores.
3. **O que cada nível pode fazer.** `declared`: fazer triagem e ver as triagens que ele mesmo fez com aquele telefone. `verified`: histórico completo, confirmações e agendamento (subprojetos 2–4).
4. **O cidadão é o par (CPF, telefone).** Um telefone serve a vários CPFs (a família que divide um celular) e um CPF pode aparecer em vários telefones. Cada par vê só as suas triagens; quem junta os pares do mesmo CPF é a validação presencial.
5. **OTP por SMS, com um provedor só, da plataforma**, atrás da interface `OtpSender`. Em desenvolvimento e teste, um `OtpSender` falso grava o código no log. A escolha do provedor (Zenvia, Twilio, SNS…) é pendência de go-live: o plano entrega o `OtpSender` de log (dev), o de teste e um `Unconfigured` que responde 503 onde nenhum provedor foi configurado.
6. **Sessão longa e deslizante:** cookie `httpOnly` de 30 dias, renovado a cada uso, com botão "sair" visível. A sessão pertence ao **telefone**; a escolha do CPF vem depois dela, então trocar de pessoa no mesmo aparelho não pede outro SMS.
7. **Formulário em etapas**, uma pergunta por tela, com botões grandes, barra de progresso e "voltar". Não é chat.
8. **"Voltar" desfaz a última resposta no backend** (`UndoLastAnswer`), só enquanto a triagem não terminou. Depois de `triage.completed`, o resultado é imutável.
9. **A web não grava em `inbound_messages`/`outbound_messages`** nesta fatia. Essas tabelas servem ao dedup e à auditoria da Meta; a tela de saúde da ingestão continua mostrando só o WhatsApp.

## 3. Arquitetura

### 3.1 Fluxo

```
wpda (formulário em etapas)
  → POST /citizen/otp                          CPF ainda não; só telefone → SMS
  → POST /citizen/session                      código → cookie httpOnly (30 dias)
  → GET  /citizen/consent_term                 termo vigente
  → GET  /citizen/people                       CPFs ligados a este telefone (mascarados)
  → POST /citizen/conversations                { citizen_id | cpf, consent_version } → abre ou retoma a conversa
  → POST /citizen/conversations/:id/answers    { answer, idempotency_key } → Citizens::SubmitAnswer → CompleteTriage → próximo passo
  → POST /citizen/conversations/:id/undo       → UndoLastAnswer → passo anterior
  → GET  /citizen/triages?citizen_id=…         triagens daquele par (CPF, telefone)
  → GET  /citizen/triages/:id                  status da triagem + link do relatório quando pronto
  → POST /citizen/triages/:id/revoke_consent   → RevokeConsent
  → DELETE /citizen/session                    sair
```

A cidade é resolvida pelo subdomínio, como no dashboard (`CityConnection` a partir do host). As respostas são síncronas: o pedido que grava a resposta devolve o próximo passo. A única espera é a do relatório, que um job gera depois de `triage.completed`: a tela final consulta `GET /citizen/triages/:id` até o link aparecer (até ~15 s).

### 3.2 Pontos de extensão no backend

- **A web não passa pelo `ConversationAdvance`** (revisto no plano, 2026-09-22). Ele é o intérprete de texto livre do chat: casa "sair", "cancelar" e "revogar" por expressão regular, e trata o consentimento como mensagem. Na web, isso sequestraria uma resposta de texto livre que fosse "cancelar", e o consentimento é uma tela antes do CPF. O contrato de dados não mora no `ConversationAdvance`, e sim em `GiveConsent` e `CompleteTriage`, e são esses que a web chama, por comandos novos em `Citizens::` (`StartConversation`, `SubmitAnswer`). A criação da triagem sai do `ConversationAdvance` para um comando `StartTriage`, usado pelos dois caminhos. O `ConversationAdvance` e o caminho do WhatsApp ficam como estão.
- **Validação da resposta:** antes do `CompleteTriage`, a web confere a resposta contra o passo (booleano `true`/`false`, uma das opções do enum, inteiro, texto até 500 caracteres). O motor não recusa resposta fora do esperado: uma resposta sem ramo encerra o fluxo e pontua.
- **Resposta:** para a web, o passo vai como JSON (`Citizens::StepPayload`) com **todas** as opções, sem truncar títulos, mais `index`, `total` (estimado pelo número de passos do protocolo) e `can_undo`, para a barra de progresso e o botão voltar. `Whatsapp::QuestionElement` fica só no caminho do WhatsApp.
- **Consentimento:** `channel` vira parâmetro de `GiveConsent` (`"web"` ou `"whatsapp"`), e a evidência da web registra o id da sessão e a versão do termo aceita.
- **`NotifyCitizenJob`:** não envia nada quando `conversation.channel == "web"`. O link do relatório aparece na tela final.
- **`UndoLastAnswer`:** comando novo. Retira a última resposta de `triages.answers` e recalcula `current_step` rodando o protocolo (função pura) sobre as respostas que restaram. Falha com erro de domínio se a triagem não estiver em andamento.

### 3.3 Dados (banco de cada cidade)

- **`citizens`** (nova): `cpf` e `phone` cifrados de forma determinística com a chave da cidade (`CityDeterministicKeyProvider`, como `Conversation.phone`), `verification_level` (`declared` | `verified`, padrão `declared`), `created_at`/`updated_at`. Índice único em `(cpf, phone)`.
- **`citizen_sessions`** (nova): `token_digest`, `phone` (cifrado determinístico), `expires_at`, `last_seen_at`, `revoked_at`.
- **`otp_challenges`** (nova): `phone` (cifrado determinístico), `code_digest`, `attempts`, `expires_at`, `consumed_at`, `sent_at`.
- **`conversations`**: ganha `channel` (`whatsapp` | `web`, padrão `whatsapp`), `citizen_id` (nulo no WhatsApp) e `last_answer_key` (a chave de idempotência da última resposta da web). O índice de conversa ativa se divide em dois índices parciais: por `citizen_id` quando `channel = 'web'`, e por `phone` quando `channel = 'whatsapp'`. Na web, `phone` recebe o telefone do cidadão, para manter a coluna `null: false` e quem já a lê.

**O que não muda:** `triages`, `consents` (exceto o valor de `channel`), `domain_events` e seus consumidores, `report_snapshots`, `dashboard_metrics` e todas as consultas do dashboard. Se for preciso um nome novo em `Platform.audit`, ele entra em `R18_PLATFORM_EVENT_NAMES`.

## 4. Fluxo do cidadão

1. **Telefone.** Só celular brasileiro (+55, 11 dígitos). Envia o SMS.
2. **Código.** 6 dígitos. Aceito o código, abre a sessão de 30 dias.
3. **Termo de consentimento.** O termo vigente aparece antes de pedir o CPF. Se o cidadão recusar, o fluxo termina e **nenhum CPF é gravado**.
4. **"Para quem é esta triagem?"** Lista os CPFs já ligados a este telefone, mascarados (`***.456.789-**`), e oferece "novo CPF". Um CPF novo passa pela checagem do dígito verificador e é criado como `declared`. O consentimento é registrado nesta conversa (`GiveConsent`, `channel: "web"`).
5. **Perguntas.** Uma por tela, com "voltar". Uma conversa ativa daquele cidadão é retomada de onde parou.
6. **Resultado.** Mostra a classificação e o link do relatório (`/r/:token`). A triagem entra em "minhas triagens" daquele CPF.

### 4.1 Consentimento

- **Nova versão do `ConsentTerm`**, que cite a coleta de CPF e telefone e as finalidades: identificação, validação presencial e, no futuro, agendamento. A base legal precisa ser validada com o jurídico antes do go-live.
- A revogação usa o `RevokeConsent` que já existe (anonimiza a triagem) e fica em "minhas triagens".
- A exclusão do cadastro (CPF) é do subprojeto 2, porque exige atendimento.

### 4.2 Proteção contra abuso

O SMS custa dinheiro e existe fraude de inflar envios.

- **Código:** vale 10 minutos, aceita no máximo 5 tentativas, reenvio só depois de 60 s. Guardado como hash.
- **SMS:** no máximo 5 por telefone por dia e um teto por IP por hora (`rate_limit` do Rails 8).
- **Enumeração:** `POST /citizen/otp` responde sempre igual, exista cadastro ou não.
- **CPFs por telefone:** no máximo 10.
- **Captcha:** fora desta fatia. Se aparecer abuso, a saída prevista é o Turnstile.

### 4.3 Erros

| Situação | Resposta |
|---|---|
| Provedor de SMS fora do ar | 503, "tente em instantes" |
| Código vencido ou tentativas esgotadas | 422; o cidadão pede um novo código |
| Sessão expirada ou revogada | 401; o wpda volta para a etapa do telefone |
| Banco da cidade atrás da versão, ou cidade fora de serviço | 503, reaproveitando `CitySchema.behind?` |
| Resposta repetida ou duas abas | o lock da conversa e a `idempotency_key` devolvem o passo atual sem gravar em dobro |
| Resposta inválida | 200 com a mesma pergunta e a mensagem de erro |
| CPF com dígito verificador inválido | 422 no campo |
| "Voltar" com a triagem concluída | 409 |

## 5. Testes

### 5.1 API (RSpec, TDD)

- **Unitários:** validador de CPF (dígito verificador, sequências repetidas como `111.111.111-11`, formatação); `OtpSender` falso; `otp_challenges` (validade, tentativas, hash); `UndoLastAnswer` (desfazer e responder de novo a mesma coisa leva ao mesmo estado; recusa depois de `completed`); busca em `Citizen` com campos cifrados.
- **Request specs**, com `type: :request` explícito: o fluxo inteiro de `/citizen/*`; limites de uso; resposta igual de `/citizen/otp`; isolamento (um CPF nunca vê a triagem de outro, nem no mesmo telefone); sessão expirada; 503 com o banco da cidade atrasado.
- **Contrato de dados — o teste que prova o objetivo.** Uma triagem completa pela web gera os **mesmos** registros e eventos que a mesma triagem pelo WhatsApp: `triages`, `consents`, `triage.completed`/`triage.urgent`, `ReportSnapshot`, `dashboard_metrics`. Só `consents.channel` e `conversations.channel` diferem.
- **Regressão do WhatsApp:** a suíte atual do webhook e do `ConversationAdvance` segue verde depois da troca de entrada do motor.
- **Suíte completa** com o worker parado; acima de ~3 min é regressão.

### 5.2 wpda (Vitest)

- Cada etapa renderiza um `Messaging::Reply`: texto, botões e lista com qualquer número de opções.
- Máscara e validação de CPF e telefone.
- "Voltar", retomada de conversa e sessão expirada.
- Acessibilidade básica: rótulos, foco visível, alvos de toque grandes.

### 5.3 Verificação manual em desenvolvimento

Uma triagem de ponta a ponta no navegador, lendo o código do SMS no log, e conferir que ela aparece nas telas do dashboard.

## 6. Fora do escopo

- Validação presencial, histórico completo, exclusão do cadastro, correção de telefone (subprojeto 2).
- Confirmação de atendimento e encaminhamento (subprojeto 3); agendamento (subprojeto 4).
- Notificação push, PWA instalável com cache offline, captcha.
- Escolha de protocolo: continua o `triage-respiratoria` fixo.
- Remoção do código do WhatsApp.
- Mostrar a web na tela de saúde da ingestão.
- Escolha do provedor de SMS (pendência de go-live, §7).

## 7. Pendências de go-live

- Validar com o jurídico a base legal e o texto da nova versão do `ConsentTerm`.
- Contratar o provedor de SMS e definir o repasse de custo às cidades.
- Definir como o cidadão chega ao link (QR code nas UBS, divulgação da prefeitura).
