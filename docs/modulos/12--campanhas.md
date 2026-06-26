# Módulo 12 — Campanhas

- **Estado:** Stub
- **Tipo:** Estratégico (pós-MVP)

## Escopo (preliminar)

Comunicação ativa em massa a partir da cidade para grupos de cidadãos
(vacinação, prevenção sazonal, retorno de exame). Distinto da
conversação reativa do MVP (módulo 02). Depende de templates aprovados
pelo Meta.

## ADRs governantes

A definir. Provavelmente:

- ADR de modelagem de campanha (segmentação, agendamento, opt-out).
- ADR de janela de comunicação ativa (templates, taxa, escalonamento).

## Superfícies

| Superfície | Papel previsto |
|---|---|
| `api` | Modelo de campanha, jobs de envio, opt-out |
| `admin` | — |
| `dashboard` | Criar/disparar campanha, acompanhar entrega |
| `wpda` | Receber comunicação, gerenciar preferências |

## Pré-requisitos

- Módulo 11 (Território) — segmentação geográfica.
- Templates aprovados pelo Meta — dependência externa não resolvida.

## Funcionalidades planejadas

_(a detalhar)_

## Histórico
