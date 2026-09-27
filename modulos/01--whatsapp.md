# Módulo 01 — WhatsApp

- **Estado:** Fechado
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
| `dashboard` | Saúde da ingestão da cidade: volume inbound, distribuição de ack (aproximada pelo status do outbound) e backlog de purga do `raw` |
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
| F-01.10 | Painel de saúde da ingestão (volume inbound, ack aproximado, backlog de purga) | dashboard | brief |

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

- **2026-09-27** — Módulo **Fechado**. F-01.1 a F-01.10 estão `Verified` no
  board. A suíte de invariante (dedup, HMAC, falha fechada, TTL do `raw`) e o
  runbook de provisionamento de canal estão publicados.

- **2026-09-27** — Lacunas da verificação fechadas (api c358656, admin 0445567,
  dashboard c162db4). Canal inativo passa a ser descartado e logado, e não mais
  registrado como número desconhecido (F-01.4). `GET /unknown_channels` e a tela
  "Números desconhecidos" do console (F-01.5). O backlog de purga do painel usa
  a retenção real do `raw`, de 90 dias, e conta só linhas ainda não purgadas
  (F-01.10). Suíte de invariante em
  `spec/architecture/whatsapp_edge_invariants_spec.rb`. Runbook em
  `operacao/provisionar-canal-whatsapp.md`.
</content>
</invoke>
