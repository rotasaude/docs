# Módulo 01 — WhatsApp

- **Estado:** Entregue
- **Tipo:** MVP

## Escopo

Borda de comunicação com o canal WhatsApp (Meta Cloud API). Recebe webhooks
do provedor, valida assinatura, persiste cada mensagem com dedup e roteia para
o tenant correto. Envia respostas e templates de volta. Não interpreta
conversa (módulo 02) nem aplica protocolo (módulo 03).

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0007 | Ack rápido, idempotência de borda, política HTTP; roteamento por `phone_number_id`, multi-tenant |
| 0013 | Cifragem e custódia do payload bruto; custódia do `access_token` por canal |
| 0005 | Resposta ao cidadão fora do `with_lock` |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | Webhook ingestion, `Whatsapp::Ingest`, `ProcessInboundMessageJob`, `SendWhatsappJob`, `Whatsapp::Outbound`, validação HMAC, dedup por `wamid` |
| `admin` | Configuração de canal por cidade (`municipality_channels`), custódia do token, registro do número, número desconhecido (`unknown_channels`) |
| `dashboard` | Saúde da ingestão (taxa de mensagens, falhas de HMAC, número desconhecido por cidade, fila de outbound) |
| `wpda` | — |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-01.1 | Webhook ingestão com HMAC | api | 0007 |
| F-01.2 | Dedup por `wamid` | api | 0007 |
| F-01.3 | Cifragem + TTL de `raw` | api | 0013 |
| F-01.4 | Roteamento por `phone_number_id` | api | 0007 |
| F-01.5 | Parking de número desconhecido | api, admin | 0007 |
| F-01.6 | Outbound (texto livre) | api | 0005 |
| F-01.7 | Outbound (template aprovado, janela >24h) | api | 0005 |
| F-01.8 | Cadastro de canal por cidade | admin | 0007, 0013 |
| F-01.9 | Custódia/rotação do `access_token` | admin | 0013 |
| F-01.10 | Painel de saúde da ingestão | dashboard | brief |

## Dependências

- Módulo 07 (LGPD/Auditoria) — purge do `raw` recorrente.
- Módulo 06 (Identidade/Acesso) — `platform_operator` autenticado para
  cadastrar canal; `municipal_admin` da cidade para ver saúde da ingestão.

## Riscos herdados

- **Clínica:** falha silenciosa no envio outbound. Hoje não há mecanismo de
  retry com confirmação no provedor — janela conhecida.
- **LGPD:** custódia da chave de AR Encryption (uma chave protege todos os
  tokens). 0013 define onde a chave vive; rotação não exercitada.
- **Operacional:** App Secret do Meta é compartilhado entre cidades; suspensão
  do App afeta todas. Blast radius assumido.
- **Em aberto:** SLA do caminho urgente (0006, dependente do módulo 03).

## Critério de fechamento do módulo

- Todas as F-01.* verificadas no board.
- Suíte de invariante: dedup por wamid, HMAC obrigatório, fail-closed sem
  tenant resolvido, TTL do `raw` executando.
- Documentação operacional do `admin` para provisionar canal novo.

## Histórico

_(funcionalidades concluídas aparecem aqui em ordem cronológica reversa)_
</content>
</invoke>
