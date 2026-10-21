# Módulo 22 — Procedimentos e faturamento

- **Estado:** Planejado
- **Tipo:** Ciclo 2

## Escopo

Registro do procedimento realizado com críticas SIGTAP que marcam e não bloqueiam
(CBO, idade, sexo, CID, instrumento) e geração mensal do arquivo de interface do
BPA Magnético (BPA-I e BPA-C) por estabelecimento faturante (ADR 0039). Entram
procedimentos registrados no atendimento, na consulta ou avulsos, os laudos
liberados do laboratório municipal (21b) e o lançamento manual do faturista. Só
estabelecimentos não-APS (UPA, hospital, outro) e o laboratório municipal geram
BPA; a UBS nunca gera (sua produção vai por LEDI/PEC). Dados do paciente que o
cadastro não tem saem como "sem informação" onde o layout aceita; o resto vira
pendência no fechamento.

**Fora, por enquanto:** APAC e OCI; FPO; conciliação do retorno do SIA; RAAS;
RAC/CMD na RNDS; faturamento de prestador credenciado.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0039 | Procedimento realizado, críticas, BPA, fechamento |
| 0028 | Modo de prontuário e LEDI (o que não vai ao BPA) |
| 0037 | Laboratório municipal (produção que entra) |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-22.1 | Interruptor e papel de faturista | api, maintenance, dashboard | 0039, 0028 |
| F-22.2 | Registro de procedimento com críticas SIGTAP | api, dashboard | 0039 |
| F-22.3 | Produção do laboratório municipal no faturamento | api | 0039, 0037 |
| F-22.4 | Lançamento manual pelo faturista | api, dashboard | 0039 |
| F-22.5 | Fechamento da competência e pendências | api, dashboard | 0039 |
| F-22.6 | Arquivo BPA-I/BPA-C do BPA Magnético | api, contracts, dashboard | 0039 |
| F-22.7 | Procedimentos na linha do tempo (fecha api#71 para procedimentos) | api | 0039, 0035 |

## Critério de fechamento

- F-22.1 a F-22.7 verificadas.
- Arquivo golden byte a byte e prova no navegador.
- Gate de go-live: importação real no BPA Magnético (ou pela secretaria piloto).

## Integrações externas

SIGTAP mensal (já importado pela plataforma); arquivo de interface do BPA
Magnético (layout do DATASUS de 09/07/2026); RAC/CMD na RNDS no futuro. Detalhe e
fontes em [`pesquisa/2026-10-05-integracoes/frente-4-faturamento-regulacao.md`](../pesquisa/2026-10-05-integracoes/frente-4-faturamento-regulacao.md).

## Depende de

Módulos 09, 10, 16, 19 e 21b.

## Em aberto

- Formato de retorno do SIA (aprovado/glosado) para conciliação.
- Data da troca do BPA pelo RAC/CMD.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-21 — Desenho aprovado e `Planejado` (ADR 0039, spec
  `superpowers/specs/2026-10-21-module-22-procedures-billing-design.md`).
