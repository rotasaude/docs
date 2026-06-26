# Módulo 14 — Analytics

- **Estado:** Stub
- **Tipo:** Estratégico (pós-MVP)

## Escopo (preliminar)

Análise epidemiológica, demanda por território, calibração de protocolo,
métricas de qualidade clínica. Distinto do dashboard operacional do MVP
(módulo 05): este responde "como está a operação agora"; analytics
responde "que padrões emergem ao longo do tempo".

## ADRs governantes

A definir. Provavelmente:

- ADR de pipeline de analytics (real-time vs. batch, agregação, retenção
  diferenciada).
- ADR de privacidade em agregação (k-anonymity, supressão de pequenos
  números).

## Superfícies

| Superfície | Papel previsto |
|---|---|
| `api` | Projeções analíticas, ETL para data warehouse (se houver) |
| `admin` | Visão cross-tenant (com agregação que respeite LGPD) |
| `dashboard` | Visão analítica da cidade |
| `wpda` | — |

## Pré-requisitos

- Módulo 11 (Território) — cortes geográficos.
- Acúmulo de dado suficiente — pré-condição operacional.

## Funcionalidades planejadas

_(a detalhar)_

## Histórico
