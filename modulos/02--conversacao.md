# Módulo 02 — Conversação

- **Estado:** Fechado
- **Tipo:** MVP

## Escopo

Máquina de estados da conversa com o cidadão e fluxo de consentimento LGPD.
Abre ou retoma a conversa, serializa cada passo por conversa (`with_lock`),
registra o consentimento na versão vigente do termo e chama o motor de
protocolos (módulo 03) enquanto a conversa está consentida. Não interpreta
protocolo (delega ao 03).

**Canal do cidadão:** desde 2026-09-27 o WhatsApp está **descontinuado** como
canal de produto, e o **wpda (canal web, ADR 0017)** assume toda a comunicação
e a triagem. O código do WhatsApp continua no api e o que o módulo 01 entregou
não foi desfeito. Quando os canais divergem, vale o comportamento da web.

Estados no código (`Conversation#state`): ativos `greeting`, `awaiting_consent`
e `consented` (o `in_progress` do título da F-02.1 é o `consented`; a web nasce
em `awaiting_consent`); terminais `completed`, `abandoned`, `declined`,
`cancelled` e `revoked`.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0008 | Estados, transições, índice único parcial, consentimento e revogação |
| 0005 | Enqueue da resposta dentro da transação, envio fora do lock |
| 0017 | Canal web do cidadão (wpda): conversa por cidadão, consentimento na tela, `/citizen/*` |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `Conversation` (guarda "terminal é terminal"), `Citizens::StartConversation`, `Citizens::SubmitAnswer`, `GiveConsent`, `RevokeConsent`, trigger `consents_guard`, varredura de abandono; no WhatsApp, `ConversationAdvance` e `Consents.interpret` |
| `admin` | — |
| `dashboard` | Painel Conversas: funil dos estados ativos, desfechos terminais, taxa de abandono, vivas agora e tempo médio até concluir; painel Consentimento (dados, revogados, por versão) |
| `wpda` | Consentimento na versão vigente, triagem em etapas, desfazer e revogar (`/citizen/*`) |

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
| F-02.9 | Visão de conversas no dashboard: distribuição por estado (funil e desfechos), taxa de abandono, vivas agora e tempo até concluir | api, dashboard | brief |
| F-02.10 | Conversa pelo canal web (`channel` `web`; uma ativa por cidadão; consentimento, resposta, desfazer e revogação) | api, wpda | 0017, 0008 |

## Dependências

- Módulo 01 (WhatsApp) — canal descontinuado; entregava o inbound já no banco
  da cidade dona do `phone_number_id` (ADR 0020).
- Módulo 03 (Triagem) — `CompleteTriage` delega ao motor de protocolos.
- Módulo 06 (Identidade/Acesso) — sessão do cidadão e identidade declarada
  (ADR 0017), que abrem a conversa web.
- Módulo 07 (LGPD/Auditoria) — versionamento de termos por cidade
  (`consent_terms`).

## Riscos que continuam

- **LGPD:** a revogação aborta e anonimiza a triagem em andamento
  (`AnonymizeRevokedTriageJob`), mas **não** desfaz a triagem já concluída nem
  o `ReportSnapshot` dela; o link do relatório segue válido. A base legal de
  retenção pós-revogação não está declarada.
- **Produto:** a web não tem comando de cancelar; uma conversa largada só
  termina pela varredura de abandono (24 h).
- **WhatsApp (descontinuado):** o rescue de `RecordNotUnique` em
  `Conversation.for` não funciona dentro da transação do job
  (`PG::InFailedSqlTransaction`, reproduzido em teste), e o WhatsApp responde
  "revogado" mesmo quando `RevokeConsent` recusa. Os consertos ficaram de fora
  por decisão (canal descontinuado).
- **Rollout:** o trigger `consents_guard` chega às cidades existentes pela
  migração de cidade `20260927000001`; rode `city:migrate:all` com a imagem
  nova antes de cortar o tráfego.

## Critério de fechamento do módulo

Cumprido em 2026-09-27.

- Todas as F-02.* verificadas no board (F-02.1 a F-02.10 `Verified`).
- Suíte de invariante (`api` `spec/architecture/conversation_invariants_spec.rb`):
  terminal é terminal (a única saída de um estado terminal é `revoked`, porque
  o cidadão pode revogar a qualquer momento); default-deny sem consentimento
  na versão vigente; re-engajamento depois de estado terminal sempre abre
  conversa nova; consentimento versionado e congelado (trigger no banco da
  cidade).
- Tratamento explícito de `RecordNotUnique` na abertura da conversa web
  (`Citizens::StartConversation`), com teste da corrida. O critério original
  apontava para `Conversation.for`, do WhatsApp, descontinuado.

## Histórico

_(funcionalidades concluídas aparecem aqui em ordem cronológica reversa)_

- **2026-09-27** — Módulo **Fechado**. F-02.1 a F-02.10 `Verified` no board;
  F-02.9 e F-02.10 reescritas para o que existe (CSV, esta tabela e cards).
  WhatsApp registrado como descontinuado; a web passa a ser o canal do módulo.

- **2026-09-27** — Lacunas da verificação fechadas (api c6583ce, dashboard
  b0cc2e6). Guarda "terminal é terminal" em `Conversation`; trigger
  `consents_guard` torna o consentimento imutável (só `revoked_at` uma vez e a
  recifragem de `evidence`); suíte de invariante do módulo. Painel Conversas
  passa a mostrar os cinco desfechos terminais e a taxa de abandono, com
  request spec, spec da query e vitest.

- **2026-09-26** — Dossiê de verificação. Testes de recusa do `GiveConsent`,
  do `RevokeConsent`, dos casos negativos de `Consents.interpret` e da
  reabertura e corrida da conversa web (api 037e4b0). Corrigido o laço de
  reconsentimento do WhatsApp quando sai termo novo no meio da triagem (api
  c82d928).
