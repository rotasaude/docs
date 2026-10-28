# Módulo 26 — Regulação de média e alta complexidade

- **Estado:** Planejado (26a); 26b e 26c depois
- **Tipo:** Ciclo 2

## Escopo (intenção)

Encaminhamento ao especialista, fila regulada, classificação de risco, cotas (próprias, PPI, consórcio), APAC/OCI e teto MAC.

**Divisão (ADR 0044), decidida em 2026-10-28:**

| Entrega | O que entra | Quando |
|---|---|---|
| 26a — Regulação municipal | Pedido estruturado na consulta, fila regulada por cor e espera, prestadores, ofertas e cotas em quantidade, agendamento e desfecho, convivência com o estado, situação no wpda | Agora |
| 26b — APAC e OCI | Laudo e faixa de números da APAC, OCI (PMAE), financiamento MAC/FAEC, teto em dinheiro | Depois do 26a |
| 26c — MIRA | Envio diário à RNDS no padrão de regulação | Com credencial |

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0044 | Regulação municipal (26a) |
| 0029 | Agenda (rede própria) |
| 0031 | Consulta (origem do pedido) |

## Funcionalidades planejadas (26a)

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-26.1 | Interruptor, papel de regulador e prazos por cor | api, dashboard, maintenance | 0044 |
| F-26.2 | Pedido de regulação na consulta e guia | api, dashboard | 0044, 0031 |
| F-26.3 | Fila regulada e decisões do regulador | api, dashboard | 0044 |
| F-26.4 | Prestadores, ofertas e cotas | api, dashboard | 0044 |
| F-26.5 | Agendamento e desfecho (realizado, falta) | api, dashboard | 0044, 0029 |
| F-26.6 | Convivência com o sistema do estado | api, dashboard | 0044 |
| F-26.7 | Situação do pedido no wpda | api, wpda | 0044, 0017 |

## Critério de fechamento (26a)

- F-26.1 a F-26.7 verificadas, com prova no navegador.

## Integrações externas

MIRA diário à RNDS se a cidade regular no Rota Saúde (Portaria 6.656/2025); convivência com SISREG (sem novas adesões), e-SUS Regulação e sistemas estaduais (CARE-PR, CROSS), todos sem API de escrita. Detalhe e fontes em
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 19, 17.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-28 — Desenho do 26a aprovado e `Planejado` (ADR 0044, spec
  `superpowers/specs/2026-10-28-module-26a-care-regulation-design.md`).
