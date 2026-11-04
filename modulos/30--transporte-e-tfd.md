# Módulo 30 — Transporte sanitário e TFD

- **Estado:** Entregue
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

## Entrega (2026-11-04)

| Repo | Commit | Observação |
|---|---|---|
| `api` | `dc1c87df` | 55 commits; suíte completa 9973/0; migração de cidade `20261103900001` (irreversível); rota `GET /transport/health_units` (T46) |
| `dashboard` | `f6fb6e4` | 47 commits; suíte 5519/0; campo CNES no cadastro de unidade (T47); "Pedir TFD" em Minhas consultas (T48) |
| `wpda` | `f923cdc` | 14 commits; suíte 847/0 + e2e em celulares |
| `maintenance` | `90237b1` | interruptores `sanitary_transport` e `tfd`; suíte 607/0 |
| `docs` | contrato §14 T1–T21 e acréscimos T22–T49 | |

**Prova no navegador (2026-11-04, dev de Curitiba; Maringá para os interruptores):**
- **Frota:** alerta de CNH a vencer; CNH B recusada em van com mais de 8 lugares; mesmo veículo ou motorista em horários sobrepostos recusado (422).
- **Viagem:** passageiro sem atendimento entra só com motivo; acompanhante fora da regra recusado; lotação exibida; lista em PDF sem CPF, documento do acompanhante ou dado clínico.
- **Motorista no celular (375 px):** só Transporte e Conta; toque duplo grava uma vez; não fecha com pendente; fechamento com odômetro e abastecimento.
- **Séries:** hemodiálise seg/qua/sex com job idempotente.
- **TFD:** recusa por distância e região metropolitana; autorização exige atendimento marcado e segundo fator. O ciclo do dinheiro foi completo: trecho de frota realizado, ajuda de custo, 3 linhas BPA `elective` com o CID do laudo, pagamento com step-up e comprovante (12 MB e .docx recusados antes de ler).
- **Painel e linha do tempo:** painel com "menos de 5"; linha do tempo "TFD para Londrina", sem detalhe.
- **wpda:** "Minhas viagens" sem placa, motorista ou dado clínico; aviso só com data, hora e embarque.
- **maintenance:** `tfd` "ligada, falta pré-requisito".

Achados corrigidos antes do merge: frase velha de atendimento; "destino a escolher"; T48; aviso de pagamento acima do devido (T49); agenda no dia da viagem criada (T49). A ajuda de custo saiu R$ 0,00 porque o SIGTAP de dev não tem os valores do 08.03.01 (gate).

**Decisões tomadas na execução** (contrato, T22–T49): data da viagem no fuso da cidade; passageiro anonimizado congelado; só a comissão cancela TFD; trecho anulado e dia travado; recibo único; pendência `refused` da série; exclusão recusada com `patient_on_trip` (409) e retenção tudo ou nada; linha do tempo do PEC sem transporte; janela do cidadão; rota de unidades para a central; CNES no cadastro de unidade; Pedir TFD depois da consulta.

**Gates de go-live** (board #2): SIGTAP 08.03.01 (códigos, valores, CBOs, trecho de frota); manual de TFD da UF piloto; `primary_care_groups.yml`; CNES da unidade de gestão no BPA; veículos no CNES (Portaria SAES 4.904/2026); km reais dos destinos; runbook para refazer o evento do BPA depois das 12 tentativas. Limite conhecido: acompanhante em texto livre não é localizado na exclusão LGPD.

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
- 2026-11-04 — Executado nos quatro apps, provado no navegador, mergeado e publicado; estado `Entregue`.
