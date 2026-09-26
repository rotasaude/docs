# Módulo 05 — Dashboard

- **Estado:** Entregue
- **Tipo:** MVP

## Escopo

O `dashboard` em si — painel operacional da cidade, tenant-scoped,
dono `municipal_admin` / `viewer`. Read-only na fase 1. Nove painéis
operacionais (overview + 8 módulos), seguindo o brief do dashboard.

Este módulo é peculiar: é o **app inteiro**, não um corte de funcionalidade
backend. Implementa-se em paralelo com os demais módulos MVP — cada um
contribui com seu painel observacional aqui.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0010 | `dashboard_metrics` projeção, leitura nunca recalcula |
| 0020 | Cada cidade é um banco: o painel lê só a cidade do Host, sem filtro por `municipality_id` |
| brief do dashboard | Especificação completa dos 9 painéis e contrato de API |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | Namespace `Admin::` (read-only), `app/queries/`, projeção `dashboard_metrics` mantida por consumer |
| `admin` | — (admin tem dashboard próprio cross-tenant; não é este) |
| `dashboard` | **É o app.** React, recharts, 9 painéis, navegação por módulo |
| `wpda` | — |

## Painéis (espelham os módulos operacionais)

| Painel | Módulo backend correspondente |
|---|---|
| Overview | (cross) |
| Ingestão WhatsApp | 01 |
| Conversas | 02 |
| Consentimento | 02 + 07 |
| Triagens | 03 |
| Classificação/Scoring | 03 |
| Protocolos | 03 |
| Filas | (transversal — Solid Queue) |
| Eventos de domínio | 07 |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-05.1 | Esqueleto do app + roteamento + i18n PT-BR | dashboard | brief |
| F-05.2 | Auth boundary (consome módulo 06; não implementa aqui) | dashboard, api | 0011, 0012 |
| F-05.3 | Carimbo "dados de \<timestamp\>" universal | dashboard | brief |
| F-05.4 | Painel Overview | dashboard, api | brief |
| F-05.5 | Painel Ingestão | dashboard, api | brief |
| F-05.6 | Painel Conversas | dashboard, api | brief |
| F-05.7 | Painel Consentimento | dashboard, api | brief |
| F-05.8 | Painel Triagens | dashboard, api | brief |
| F-05.9 | Painel Classificação | dashboard, api | brief |
| F-05.10 | Painel Protocolos | dashboard, api | brief |
| F-05.11 | Painel Filas (Solid Queue) | dashboard, api | brief |
| F-05.12 | Painel Eventos de domínio | dashboard, api | brief |
| F-05.13 | Atualização de projeções (`UpdateDashboardJob`) | api | 0004, 0005, 0010 |
| F-05.14 | Job de reconciliação (rebuild de projeção) | api | 0010 |

## Dependências

- Todos os outros módulos MVP — cada um contribui com fonte de dado.
- Módulo 06 (Identidade/Acesso) — auth boundary é pré-requisito de produção.

## Riscos herdados

- **(ADR 0005):** painel de Filas não cobre a pior classe de falha (no-op
  silencioso do idempotency bug). Documentar como limitação.
- **(ADR 0006):** monitoramento de Solid Queue não tem ADR próprio; o
  painel é a primeira tentativa, mas alerta proativo (Slack/email quando
  fila cresce) fica fora do escopo do dashboard.
- **Operacional:** dashboard sem auth não vai para produção; bloqueio no
  módulo 06.

## Critério de fechamento do módulo

- F-05.1 a F-05.14 verificadas.
- Suíte de invariante: nenhum endpoint retorna dado fora do tenant; todo
  agregado mostra timestamp de origem; nenhum painel dispara recompute
  pesado.
- Decisão registrada: quais KPIs vêm de `dashboard_metrics` (projeção) vs.
  agregação ao vivo.

## Histórico
