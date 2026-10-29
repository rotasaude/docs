# Módulo 26 — Regulação de média e alta complexidade

- **Estado:** 26a Entregue (2026-10-29; F-26.1–F-26.7 `Verified`); 26b e 26c depois
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

## Entrega do 26a (2026-10-29)

| Repo | Commit | Observação |
|---|---|---|
| `api` | `51c33aee` | 44 commits; suíte completa 8072/0; migração de cidade `20261028900001` (inclui `attendances.referral_regulated`) |
| `dashboard` | `75a2d06` | 25 commits; 4607/0 |
| `wpda` | `e699cd0` | 9 commits; "Meus encaminhamentos" |
| `maintenance` | `eb18725` | interruptor `care_regulation` |
| `catalog` | `4807d7e` | F-26.6 "em validação" até a secretaria piloto digitar pedidos reais no sistema do estado |
| `docs` | contrato §13 G1–G16 | runbook [`operacao/rollout-regulacao.md`](../operacao/rollout-regulacao.md) |

**Execução em paralelo** (pedido do usuário): três frentes (api; dashboard; wpda e maintenance) com até três implementadores, fronts contra dublês até a task do api correspondente ficar pronta.

**Prova no navegador (2026-10-28/29):** pedido na consulta até a fila; ordem com vencidos no topo; motivo para furar a fila; saldo e cota esgotada; devolvido e complemento; externo agendado e realizado; CSV e guia em PDF; prazos e lista do enfermeiro; cidadão no celular (agendado, autorizado, realizado, devolvido, só declarado, interruptor desligado), sem cor, posição, CID ou prazo; interruptor no maintenance.

**Decisões tomadas na execução** (contrato §13): SIGTAP regulável com BPA mesmo havendo APAC; patologia, radiologia, medicina nuclear, patologia bucal e endoscopia como procedimento; dentista e enfermeiro com CIAP-2 quando não podem CID-10; rascunho inválido nunca bloqueia a finalização nem o adendo; `attendances.referral_regulated`; cor confirmada × efetiva; o cidadão não recebe o prazo (revelaria a cor); desfecho só a partir do dia do agendamento; "Autorizado — aguardando o agendamento".

Página de status: https://claude.ai/artifact/TEtXBsJ5skPLaU4SoTTLvj

**Rodada de fechamento (2026-10-29):** api `ded8a0eb`, dashboard `fb18217`, wpda `886ab9a` — rastro da troca de número externo, CNES do prestador próprio lido da unidade, sementes de dev resistentes a óbito, paginação dos devolvidos (contrato G17), e2e do wpda sem api real, teste do formulário de domicílio estável; suíte completa 8087/0. Card api#102 (documento assinado com os pedidos de regulação).

**Evolução registrada:** fila de cirurgia eletiva (AIH) no 26b; documento assinado listando os pedidos de regulação.

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
- 2026-10-29 — 26a **implementado, publicado e Entregue** (ver "Entrega do 26a").
