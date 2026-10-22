# Módulo 22 — Procedimentos e faturamento

- **Estado:** Entregue (2026-10-22; F-22.1–F-22.5 e F-22.7 `Verified`; F-22.6 `Done`; `Fechado` depois do gate da importação real no BPA Magnético)
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

## Entrega (2026-10-22)

| Repo | Commit | Observação |
|---|---|---|
| `contracts` | `7019371` | tag `billing-v1.0.0` (perfil do arquivo BPA, oráculo Python, exemplos golden) |
| `api` | `c5b6a9d` | 37 commits; suíte completa 6525/0 |
| `dashboard` | `e45ca71` | 27 commits; 3473/3473 |
| `maintenance` | `f6d0a8d` | interruptor `procedure_billing` (requer IBGE) |
| `wpda` | `9748713` | rótulo "Procedimento" na linha do tempo do cidadão |
| `catalog` | `f3dfe7e` | F-22.6 "em validação" |
| `docs` | `277999e` | prova técnica do layout e runbook |

Migração de cidade `20261021900001` (procedimentos, competências, arquivos; raça/cor,
etnia, CEP e endereço do paciente, cifrados). Contrato com acréscimos §12 e §13
(E1–E24). Runbook [`operacao/rollout-faturamento-bpa.md`](../operacao/rollout-faturamento-bpa.md).

**Prova técnica:** layout do BPA Magnético 05.00 (09/07/2026) lido campo a campo
(130/48/351 posições, CR+LF, campo de controle); gerador conferido byte a byte com
oráculo Python independente.

**Prova no navegador (2026-10-22):** procedimento na UPA (crítica ok; CID faltando
avisa e grava; cancelamento com motivo); UBS sem procedimento faturável em lugar
nenhum; laudos do laboratório municipal entrando; lançamento manual; pendência de
dados do paciente completada pelo faturista e BPA-I com raça, CEP, número e bairro
nas posições do layout; fechamento com step-up (v1), arquivo conferido e baixado sem
cópia no navegador; reabertura com motivo (v2), admin sem download; procedimento
atrasado na competência seguinte; linha do tempo com "Realizado em"; telas em
celular; interruptor desligado esconde tudo. Três defeitos achados e corrigidos
antes do merge (busca de paciente sem resolver o par validado; nome nulo; resumo não
relido).

**Decisões tomadas na execução** (contrato §12–§13): UBS nunca gera BPA; paciente
ganha raça/cor e endereço (layout recusa "sem informação" e critica endereço);
críticas recalculadas no fechamento; competência atrasada nunca abre mês no passado;
resumo da competência aberta igual às pendências; lista de procedimentos com trilha;
arquivo adulterado nunca sai.

**Gate de go-live:** importação real no BPA Magnético (Windows) ou pela secretaria
piloto, conferindo `prd-org` (BPA × EXT), código de logradouro e INE.

F-22.1 verificada depois do merge na tela do maintenance publicado (liga e desliga
com confirmação). Página de status: https://claude.ai/artifact/EQbnquYhiTJY2rUvT1w9uJ

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
- 2026-10-22 — **Implementado, publicado e Entregue** (ver "Entrega"). `Fechado` depois do gate da importação real.
