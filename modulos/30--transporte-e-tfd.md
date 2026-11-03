# Módulo 30 — Transporte sanitário e TFD

- **Estado:** Planejado
- **Tipo:** Ciclo 2

## Escopo (intenção)

Agendamento de viagens, veículos, passageiros e Tratamento Fora de Domicílio. Lacuna presente nos 15 editais.

**Escopo (ADR 0048), decidido em 2026-11-03:** um módulo só, com a frota (veículos e motoristas com CNH), destinos com distância, viagens avulsas e recorrentes (hemodiálise, radioterapia) sem sobreposição de veículo ou motorista, passageiros só com atendimento marcado (regulação, agenda, TFD, série) ou com motivo, lista de passageiros sem dado clínico, tela do motorista no celular; TFD com laudo do médico, comissão com segundo fator, recusa abaixo de 50 km ou na região metropolitana, trechos, ajuda de custo calculada (Portaria SAS/MS 55/1999), pagamento registrado e prestação de contas; produção do 08.03.01 no BPA-I pelo módulo 22. Fora: urgência e SAMU, GPS, aplicativo do motorista, contratos de frota, CNRAC, pagamento na tesouraria.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0048 | Transporte sanitário e TFD |
| 0044 | Agendamento externo da regulação, prestadores |
| 0031 | Leitura do laudo com trilha |
| 0035 | Fonte `transport` da linha do tempo |
| 0026 | Retenção do TFD na exclusão |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-30.1 | Interruptores, veículos, motoristas e destinos | api, dashboard, maintenance | 0048 |
| F-30.2 | Viagem, passageiros e lista sem dado clínico | api, dashboard | 0048, 0044 |
| F-30.3 | Séries recorrentes (hemodiálise, radioterapia) | api, dashboard | 0048 |
| F-30.4 | Tela do motorista, presença e fechamento | api, dashboard | 0048 |
| F-30.5 | Laudo e comissão de TFD | api, dashboard | 0048, 0031 |
| F-30.6 | Ajuda de custo, pagamento e prestação de contas | api, dashboard | 0048 |
| F-30.7 | Produção do TFD no BPA-I | api | 0048 |
| F-30.8 | Minhas viagens, avisos, linha do tempo e painel | api, wpda, dashboard | 0048, 0035 |

## Critério de fechamento

- F-30.1 a F-30.8 verificadas, com prova no navegador; códigos SIGTAP 08.03.01 conferidos no SIGTAP oficial e manual estadual de TFD da cidade piloto lido são gates de go-live.

## Integrações externas

SIGTAP/BPA-I para o TFD (grupo 08.03.01). Detalhe e fontes em
[`pesquisa/2026-11-03-transporte-e-tfd-30.md`](../pesquisa/2026-11-03-transporte-e-tfd-30.md) e
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 17, 19a, 22, 26a.

## Documentos

- Spec: [`superpowers/specs/2026-11-03-module-30-sanitary-transport-design.md`](../superpowers/specs/2026-11-03-module-30-sanitary-transport-design.md)

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-11-03 — Desenho aprovado (ADR 0048); estado `Planejado`.
