# Módulo 04 — Relatórios

- **Estado:** Fechado
- **Tipo:** MVP

## Escopo

Geração de relatório por triagem (snapshot congelado), link assinado de
acesso pelo paciente, renderização. Não classifica (é o 03); não envia
notificação (é o 01 via consumer). Trabalha exclusivamente do lado de
leitura (CQRS).

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0010 | `report_snapshots` congelado por `protocol_version`, leitura nunca recalcula |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `GenerateReportJob`, `report_snapshots`, token assinado, rota `/r/<token>` |
| `admin` | — |
| `dashboard` | Lista de relatórios da cidade, status, regerar (se for o caso), inspeção |
| `wpda` | Visualização do relatório pelo paciente via link assinado |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-04.1 | `GenerateReportJob` consumindo `triage.completed` | api | 0004, 0005, 0010 |
| F-04.2 | Snapshot `report_snapshots` com `payload` + versão do protocolo (`protocol_definition_id`) | api | 0010 |
| F-04.3 | Token assinado para acesso público (sem login) | api | — (toca módulo 06) |
| F-04.4 | Rota `/r/<token>` lê do snapshot, nunca recalcula | api | 0010 |
| F-04.5 | Renderização do relatório no wpda | wpda | 0010 |
| F-04.6 | Lista de relatórios no dashboard da cidade | dashboard | brief |
| F-04.7 | Notificação ao cidadão: link via WhatsApp (no canal web o link aparece na tela final) | api | 0004 (via `NotifyCitizenJob`) |

## Dependências

- Módulo 03 (Triagem) — emite `triage.completed` que dispara o snapshot.
- Módulo 01 (WhatsApp) — `NotifyCitizenJob` usa o outbound.
- Módulo 06 (Identidade/Acesso) — desenho do token assinado, expiração,
  revogação.

## Riscos herdados

- **LGPD:** o token assinado é credencial de acesso a dado clínico sem
  autenticação forte; expiração curta e impossibilidade de adivinhar são
  invariantes.
- **(ADR 0005) (idempotência):** `NotifyCitizenJob` herda o bug — grava
  marcador antes do envio externo. Notificação pode sumir silenciosamente.
- **Em aberto:** revogação de relatório quando o cidadão exerce Art. 18 da
  LGPD (apaga relatório? mascara? mantém para auditoria?).

## Critério de fechamento do módulo

- F-04.1 a F-04.7 verificadas.
- Suíte de invariante: snapshot imutável; mudança de protocolo não muda
  relatório antigo; `/r/<token>` jamais recalcula.
- Caso clínico de regressão: relatório gerado em `municipal-1.2.0`
  permanece idêntico após publicação de `municipal-1.3.0`.

## Histórico

- 2026-09-26 — verificação do módulo iniciada. F-04.2 renomeada: a versão fica
  congelada por `protocol_definition_id`, não por coluna `protocol_version`.
  F-04.7 passa a dizer que o canal web mostra o link na tela. ADR 0010 revisado:
  o HMAC do token é conferido depois do lookup pelo índice único. Achado bloqueante
  em F-04.7 (o aviso se perde quando roda antes do snapshot) em correção no api.
- 2026-09-26 — módulo **Fechado**. F-04.1 a F-04.7 verificadas pelo usuário.
  Critério cumprido no api (merge abe6dfe): trigger de imutabilidade em
  `report_snapshots` (migração de cidade 20260926100001), suítes
  `spec/invariants/report_snapshot_immutability_spec.rb` e
  `spec/invariants/report_snapshot_frozen_spec.rb` (caso clínico: `municipal`
  v120 → v130, com a resposta de `/r/:token` idêntica byte a byte, porque as
  versões são inteiras), e o bug de F-04.7 corrigido (o aviso tenta de novo até
  o snapshot existir). Testes de tela no wpda (cccba3d) e no dashboard (9abe471).
  Decisões: a tela do wpda segue sem mostrar o `summary`; o roteamento de
  `/wpda/` em produção é bloqueio de go-live da plataforma (README do api, ADR
  0020 "Em aberto"), não pendência da F-04.5.
