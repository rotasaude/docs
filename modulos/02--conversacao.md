# Módulo 02 — Conversação

- **Estado:** Planejado
- **Tipo:** MVP

## Escopo

Máquina de estados da conversa com o cidadão e fluxo de consentimento LGPD.
Recebe inbound já roteado pelo módulo 01 (WhatsApp), serializa por conversa,
avança estado, chama o motor de protocolos (módulo 03) quando em
`in_progress`. Não interpreta protocolo (delega ao 03), não envia HTTP (delega
ao 01).

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0008 | Estados, transições, índice único parcial, consentimento e revogação |
| 0005 | Enqueue da resposta dentro da transação, envio fora do lock |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `Conversation`, `advance`, `with_lock`, `GiveConsent`, `RevokeConsent`, `Consents.interpret`, varredura de timeout (recurring task) |
| `admin` | — |
| `dashboard` | Distribuição por estado, taxa de consentimento (aceito/recusado/ambíguo), conversas em `awaiting_consent` há mais de X, taxa de revogação |
| `wpda` | — |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-02.1 | Estados `awaiting_consent` / `in_progress` | api | 0008 |
| F-02.2 | Estados terminais (`completed`, `declined`, `cancelled`, `abandoned`) | api | 0008 |
| F-02.3 | Índice único parcial em `phone` para conversas ativas do WhatsApp (por banco de cidade) | api | 0008, 0007, 0020 |
| F-02.4 | Captura de consentimento (botões interativos) | api | 0008 |
| F-02.5 | Revogação (`consent.revoked`) | api | 0008 |
| F-02.6 | Re-pergunta em ambiguidade (default-deny) | api | 0008 |
| F-02.7 | Varredura de timeout / `abandoned` | api | 0008 |
| F-02.8 | Enqueue de resposta dentro da transação do lock | api | 0005 |
| F-02.9 | Visão de conversas no dashboard da cidade | dashboard | brief |

## Dependências

- Módulo 01 (WhatsApp) — entrega o inbound roteado com `municipality_id`.
- Módulo 03 (Triagem) — `handle_answer` delega ao motor de protocolos.
- Módulo 07 (LGPD/Auditoria) — versionamento de termos por cidade
  (`consent_terms`).

## Riscos herdados

- **Clínica:** corrida de primeiro contato (ADR 0007) — dois inbounds do
  mesmo telefone criam conversas concorrentes; depende de índice + retry. Falta
  rescue explícito de `RecordNotUnique` em `Conversation.for`.
- **LGPD:** `consent.revoked` é publicado mas (a) não tem assinante hoje;
  (b) não desfaz triagem/snapshot já gerado. Base legal de retenção
  pós-revogação não declarada.
- **Clínica:** `Result` de `GiveConsent` ignorado dentro do
  `with_lock` — rescue interno do command engole o erro, transação externa
  segue.
- **Em aberto:** janela de 24h do WhatsApp vs. comunicação urgente ao
  cidadão (ADR 0007).

## Critério de fechamento do módulo

- Todas as F-02.* verificadas no board.
- Suíte de invariante: terminal é terminal; default-deny em ambiguidade;
  re-engajamento de telefone em terminal sempre cria conversa nova;
  consentimento versionado e congelado por conversa.
- Tratamento explícito de `RecordNotUnique` em `Conversation.for`.

## Histórico
