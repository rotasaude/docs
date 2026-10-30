# Módulo 27 — Farmácia e almoxarifado

- **Estado:** 27a Entregue (2026-10-30; F-27.1–F-27.7 `Verified`); insumos e 27b depois
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

## Entrega do 27a (2026-10-30)

| Repo | Commit | Observação |
|---|---|---|
| `api` | `7e086d78` | suíte completa 8750/0; migração de cidade `20261029900001`; toca o 09 (`pharmacy_pending`) |
| `dashboard` | `fadfa87` | 24 commits; grupo "Farmácia" |
| `maintenance` | `1fdda62` | interruptor `pharmacy`; corrige maintenance#1 |
| `catalog` | `85b90d3` | F-27.6 "em validação" até a prova do SNCR real |
| `docs` | contrato §13 H1–H17 | runbook [`operacao/rollout-farmacia.md`](../operacao/rollout-farmacia.md) |

**Execução em paralelo:** frente do api e frente dos fronts com dublês, como no 26a.

**Prova no navegador (2026-10-30):** administração; almoxarife (posição com unidade de dispensação, transferência); auxiliar (recebimento com diferença como perda, controlado bloqueado, antimicrobiano só com retenção, devolução, saldo fechando a cada passo); farmacêutica (receita cancelada, controlado com retenção, item em texto livre mapeado, receita externa, ajuste com aprovação de outra pessoa); intervalo de 25 dias do uso contínuo no api real; interruptor no maintenance.

**Decisões tomadas na execução** (contrato §13): unidade de dispensação pela forma farmacêutica; antimicrobiano com retenção da 2ª via (RDC 471/2021) pela marca do catálogo; validade e intervalo por item; ajuste de contagem com aprovação e uma pendente por lote; devolução de controlado só pelo farmacêutico; unidade com transferência em trânsito não é desativada; busca de receita por paciente em qualquer farmácia da rede com papel, limite e trilha.

Página de status: https://claude.ai/artifact/CoH6yUQhQdPdkreoyPAFbE

**Gates de go-live:** prova do SNCR simulado (imagem de dev desatualizada na prova); perfil profissional para toda a equipe da farmácia; medir itens CATMAT sem forma farmacêutica.

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
- 2026-10-30 — 27a **implementado, publicado e Entregue** (ver "Entrega do 27a").
