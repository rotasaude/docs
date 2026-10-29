# Módulo 27 — Farmácia e almoxarifado

- **Estado:** Planejado (27a); insumos e 27b depois
- **Tipo:** Ciclo 2

## Escopo (intenção)

Dispensação, estoque por unidade, lotes e validade, almoxarifado central. Lacuna presente nos 15 editais analisados.

**Divisão (ADR 0045), decidida em 2026-10-29:**

| Entrega | O que entra | Quando |
|---|---|---|
| 27a — Estoque e dispensação | Almoxarifado central e farmácia da unidade, medicamentos CATMAT com lote e validade, entradas, transferências, inventário, dispensação pela receita, controlados pelo farmacêutico | Agora |
| Insumos | Catálogo de materiais (seringa, fita, lanceta, curativo) sobre o mesmo estoque | Depois do 27a |
| 27b — Envio federal | BNAFAR/Hórus, depois de pesquisa | Depois do 27a |

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0045 | Estoque e dispensação (27a) |
| 0033 | Receita (origem da dispensação) |
| 0034 | Controlados |

## Funcionalidades planejadas (27a)

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-27.1 | Interruptor, papéis e locais de estoque | api, dashboard, maintenance | 0045 |
| F-27.2 | Entradas, lotes e validade | api, dashboard | 0045 |
| F-27.3 | Transferência do almoxarifado para a unidade | api, dashboard | 0045 |
| F-27.4 | Inventário, ajustes e perdas | api, dashboard | 0045 |
| F-27.5 | Dispensação pela receita | api, dashboard | 0045, 0033 |
| F-27.6 | Receita externa e controlados | api, dashboard | 0045, 0034 |
| F-27.7 | Alertas, posição e extrato | api, dashboard | 0045 |

## Critério de fechamento (27a)

- F-27.1 a F-27.7 verificadas, com prova no navegador.

## Integrações externas

BNAFAR/Hórus; e-SUS AF (lançado em abr/2026) — a confirmar se aceita gravação de terceiros. Detalhe e fontes em
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 19 e 24a.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-29 — Desenho do 27a aprovado e `Planejado` (ADR 0045, spec
  `superpowers/specs/2026-10-29-module-27a-pharmacy-stock-design.md`).
