# Módulo 32 — Urgência (UPA)

- **Estado:** Planejado
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

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-11-05 — Desenho aprovado (ADR 0050); estado `Planejado` (32a e 32b).
