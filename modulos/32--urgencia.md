# Módulo 32 — Urgência (UPA)

- **Estado:** Entregue (32a em 2026-11-10; 32b em 2026-11-11); `Fechado` depois dos gates de go-live
- **Tipo:** Ciclo 2

## Escopo (intenção)

Classificação de risco e atendimento de urgência (10 de 15 editais).

**Escopo (ADR 0050), decidido em 2026-11-05:** o módulo sai em duas entregas.

**32a**
- chegada espontânea na UPA, inclusive de paciente não identificado;
- classificação de risco em 5 cores, pelo protocolo assinado da cidade;
- fila por cor, com tempo máximo;
- atendimento médico como consulta do 19a;
- desfecho obrigatório, com contrarreferência à UBS;
- painel de chamada por senha para TV (UPA e UBS);
- BPA, vigilância, histórico no wpda e painel do gestor.

**32b**
- observação com leitos e alerta de 24 h;
- prescrição interna com checagem da enfermagem;
- baixa no estoque da UPA, ligada ou desligada por interruptor.

**Fora:** SAMU, pronto-socorro hospitalar, central estadual de leitos e fluxogramas licenciados do Manchester.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0050 | Urgência |
| 0030 | Protocolo assinado (motor do 18) |
| 0031 | Consulta e leitura do prontuário |
| 0035 | Fonte `urgency` da linha do tempo |
| 0026 | Retenção na exclusão |

## Funcionalidades planejadas

| ID | Funcionalidade | Entrega | Superfície | ADRs |
|---|---|---|---|---|
| F-32.1 | Interruptores, protocolo de 5 cores e configuração da UPA | 32a | api, dashboard, maintenance, contracts | 0050, 0030 |
| F-32.2 | Chegada, senha e paciente não identificado | 32a | api, dashboard | 0050 |
| F-32.3 | Classificação de risco e fila por cor | 32a | api, dashboard | 0050, 0030 |
| F-32.4 | Atendimento médico e desfecho com contrarreferência | 32a | api, dashboard | 0050, 0031 |
| F-32.5 | Chamada com local e painel da TV (UPA e UBS) | 32a | api, dashboard | 0050 |
| F-32.6 | BPA, vigilância, histórico no wpda e painel do gestor | 32a | api, dashboard, wpda | 0050, 0035 |
| F-32.7 | Observação com leitos e alerta de 24 h | 32b | api, dashboard | 0050 |
| F-32.8 | Prescrição interna, checagem e baixa de estoque | 32b | api, dashboard, maintenance | 0050 |

## Critério de fechamento

- F-32.1 a F-32.8 verificadas, com prova no navegador.
- São gates de go-live:
  - códigos SIGTAP da urgência;
  - revisão clínica do protocolo da cidade.

## Integrações externas

SIA (BPA) para a produção da UPA. Detalhe e fontes em
[`pesquisa/2026-11-05-urgencia-upa-32.md`](../pesquisa/2026-11-05-urgencia-upa-32.md) e
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 13, 18, 19, 21, 22, 25, 27a.

## Documentos

- Spec: [`superpowers/specs/2026-11-05-module-32-urgency-design.md`](../superpowers/specs/2026-11-05-module-32-urgency-design.md)

## Entrega

### 32a — 2026-11-10

- **Commits na main:**
  - api 69d283c4;
  - dashboard 979e881;
  - wpda 339e4ac;
  - maintenance bbff803;
  - contracts b0d1f87 (tag `protocols-v1.7.0`).
- **Cards:** F-32.1 a F-32.6 em Verified, com prova no navegador nos quatro apps. F-32.7 e F-32.8 (32b) seguem em Not Started.
- **Testes:**

  | App | Resultado |
  |---|---|
  | api | suíte completa 10823/0 |
  | dashboard | 6241/0 |
  | wpda | 861/0 |
  | maintenance | 622/0 |

- **Decisões de execução:** Q16–Q31 no contrato [`superpowers/plans/2026-11-05-module-32a-urgency-contracts.md`](../superpowers/plans/2026-11-05-module-32a-urgency-contracts.md). A Q31 é a revisão do ADR 0026.
- **Página de status:** https://claude.ai/artifact/JR7dArccMh7akYssERmtJZ
- **Catálogo:** #m-32a em rotasaude/catalog#37 (desde o catalog#38, #m-32).
- **Gates de go-live (board #2):**
  - SIGTAP da UPA (api#136);
  - revisão clínica do protocolo (api#137);
  - licença do Manchester por cidade (api#138);
  - rotação do token da TV (api#139);
  - trusted proxies (api#140).
- **Pendências (board #2):** api#133–#135 e dashboard#24.

### 32b — 2026-11-11

- **Commits na main:**
  - api 3e96796d;
  - dashboard 84e8824 (443faca corrige a CI, vermelha desde o 32a por um fetch real escapando em dois testes de faturamento);
  - maintenance f2086a1.
- **Cards:** F-32.7 e F-32.8 em Verified.
- **Testes:**

  | App | Resultado |
  |---|---|
  | api | suíte completa 11097/0 |
  | dashboard | 6418/0 |
  | maintenance | 631/0 |

- **Decisões do usuário:** P1 = A (a prescrição interna aceita todas as listas da Portaria 344; a escrituração fica no gate api#63) e P2 = A (só a observação é faturada; o 0301100012 fica no api#136).
- **Decisões de reconciliação e execução:** Q1–Q17 no contrato [`superpowers/plans/2026-11-10-module-32b-urgency-observation-contracts.md`](../superpowers/plans/2026-11-10-module-32b-urgency-observation-contracts.md).
- **Página de status:** https://claude.ai/artifact/TMnNY1Xzq3Qjqh3z4Yuvrj
- **Catálogo:** o módulo passa a `#m-32` (F-32.1 a F-32.8) em rotasaude/catalog#38.
- **Gates de go-live:** comentados no api#63 (controlados de qualquer lista) e no api#136 (BPA só da observação).
- **Pendências (board #2):**
  - api#143 (deadlock entre adendo e identificação);
  - api#144 (trilha nas respostas da prescrição);
  - api#145 (dose fracionada entre lotes);
  - api#146 (correção de checagem errada);
  - api#147 (termo da busca de medicamento no log DEBUG);
  - dashboard#25–#27 (botão de iniciar atendimento, painel após troca de leito, 7 ajustes visuais).

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-11-05 — Desenho aprovado (ADR 0050); estado `Planejado` (32a e 32b).
- 2026-11-10 — 32a entregue (F-32.1 a F-32.6 Verified); 32b segue planejado.
- 2026-11-11 — 32b entregue (F-32.7 e F-32.8 Verified); módulo 32 Entregue.
