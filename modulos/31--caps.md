# Módulo 31 — Atenção psicossocial (CAPS)

- **Estado:** Planejado
- **Tipo:** Ciclo 2

## Escopo (intenção)

Atendimento nos CAPS e registro em RAAS (14 de 15 editais).

**Escopo (ADR 0049), decidido em 2026-11-04:** um módulo só, com:
- CAPS como tipo de unidade, com modalidade e equipe;
- encaminhamento da UBS, acolhimento e matrícula;
- PTS com versões;
- ações individuais, coletivas e no território;
- acolhimento noturno com leitos no CAPS III e no AD III;
- RAAS do mês com crítica, remessa no layout oficial e ficha espelho;
- matriciamento e BPA-C pelo módulo 22.

Sigilo: o prontuário é lido só pela equipe do CAPS. A UBS vê o vínculo e o texto compartilhado do matriciamento. Nada do CAPS vai ao wpda, a avisos, ao SMS ou ao LEDI.

Fora: residência terapêutica como serviço, UA/UAI, comunidade terapêutica, internação psiquiátrica e RAAS da atenção domiciliar.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0049 | Atenção psicossocial |
| 0031 | Leitura contextual do prontuário e trilha |
| 0034 | Receita controlada |
| 0035 | Linha do tempo (fonte restrita) |
| 0026 | Retenção na exclusão |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-31.1 | Interruptor, CAPS, modalidade e equipe | api, dashboard, maintenance | 0049 |
| F-31.2 | Encaminhamento, acolhimento e matrícula | api, dashboard | 0049 |
| F-31.3 | PTS com versões e revisão | api, dashboard | 0049 |
| F-31.4 | Ações individuais, coletivas e no território | api, dashboard | 0049, 0034 |
| F-31.5 | Acolhimento noturno: leitos e permanências | api, dashboard | 0049 |
| F-31.6 | RAAS do mês: crítica, fechamento, remessa e ficha espelho | api, dashboard | 0049 |
| F-31.7 | Matriciamento, vínculo na UBS e BPA-C | api, dashboard | 0049, 0031 |
| F-31.8 | Sigilo: nada no wpda nem em avisos, linha do tempo restrita, fora do LEDI, e painel | api, dashboard, wpda | 0049, 0035 |

## Critério de fechamento

- F-31.1 a F-31.8 verificadas, com prova no navegador.
- São gates de go-live:
  - remessa RAAS importada com sucesso no aplicativo RAAS ou no SIA;
  - códigos de acolhimento conferidos no SIGTAP oficial;
  - limite do CAPS III conferido na norma.

## Integrações externas

SIA — RAAS (Atenção Psicossocial) e BPA-C. Detalhe e fontes em
[`pesquisa/2026-11-04-atencao-psicossocial-31.md`](../pesquisa/2026-11-04-atencao-psicossocial-31.md) e
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 16, 19 (19d), 22.

## Documentos

- Spec: [`superpowers/specs/2026-11-04-module-31-psychosocial-care-design.md`](../superpowers/specs/2026-11-04-module-31-psychosocial-care-design.md)

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-11-04 — Desenho aprovado (ADR 0049); estado `Planejado`.
