# Módulo 27 — Farmácia e almoxarifado

- **Estado:** 27a Entregue (2026-10-30; F-27.1–F-27.7 `Verified`); 27b Entregue (2026-11-26; F-27.8–F-27.13 `Done`, Verified na verificação do módulo); insumos depois
- **Tipo:** Ciclo 2

## Escopo (intenção)

Dispensação, estoque por unidade, lotes e validade, almoxarifado central. Lacuna presente nos 15 editais analisados.

**Divisão (ADR 0045), decidida em 2026-10-29:**

| Entrega | O que entra | Quando |
|---|---|---|
| 27a — Estoque e dispensação | Almoxarifado central e farmácia da unidade, medicamentos CATMAT com lote e validade, entradas, transferências, inventário, dispensação pela receita, controlados pelo farmacêutico | Agora |
| Insumos | Catálogo de materiais (seringa, fita, lanceta, curativo) sobre o mesmo estoque | Depois do 27a |
| 27b — Envio federal | BNAFAR/Hórus, depois de pesquisa | Depois do 27a |
| Importação do Hórus | Histórico do Hórus importado para o Rota Saúde, para a cidade que deixar o Hórus (decisão do usuário, 2026-11-24) | Com o 27b |

Pesquisa dos insumos e do 27b (2026-11-23): [`pesquisa/2026-11-23-insumos-e-bnafar-27.md`](../pesquisa/2026-11-23-insumos-e-bnafar-27.md). Pela Portaria 5.713/2024, o SI-BNAFAR recebe, em D-1, a posição de estoque (REPE) e as saídas por transferência e perda (RESMPE). A dispensação vai à RNDS pelo REDFM. O Hórus está sendo substituído pelo eSUSAF. Insumos usam CATMAT 6515/6550.

**Importação do histórico do Hórus (decisão do usuário, 2026-11-24).** A Portaria GM/MS 11.585/2026 substitui o Hórus pelo e-SUS AF. Os lançamentos no Hórus terminam por volta de dez/2026, e depois disso a base histórica é entregue aos entes por API ou em arquivos tabulados. Quem deixar o Hórus precisa de um sistema que envie à BNAFAR. Com o 27b pronto, o Rota Saúde é essa porta de entrada, e importar o histórico do Hórus (estoque, lotes, dispensações) é o diferencial. A importação entra no escopo do 27b e precisa de pesquisa do formato que o Ministério entrega. Fonte: [`pesquisa/2026-11-23-insumos-e-bnafar-27.md`](../pesquisa/2026-11-23-insumos-e-bnafar-27.md), achado 10 do 27b.

Pesquisa da base do Hórus e da API do SI-BNAFAR v1.25 (2026-11-24): [`pesquisa/2026-11-24-horus-e-si-bnafar-27b.md`](../pesquisa/2026-11-24-horus-e-si-bnafar-27b.md). O formato da base histórica do Hórus ainda não foi publicado, e o e-SUS AF não migra nada. O SI-BNAFAR só aceita operações de hoje ou de ontem, então o histórico fica no Rota Saúde, para consulta. A autenticação é por JWT com HTTP Basic, e a retificação vale até 30 dias.

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

## Funcionalidades do 27b

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-27.8 | Interruptores e credencial BNAFAR | api, dashboard, maintenance | 0045 |
| F-27.9 | Dados BNAFAR do lote, do local e do produto (OBM, CATMAT, vacina) | api, dashboard, maintenance | 0045, 0047 |
| F-27.10 | Saída externa | api, dashboard | 0045 |
| F-27.11 | Envio diário ao SI-BNAFAR e simulado | api | 0045 |
| F-27.12 | Situação, pendências, reenvio, correção, exclusão e alerta | api, dashboard, maintenance | 0045 |
| F-27.13 | Saldo inicial por planilha | api, dashboard | 0045 |

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

**Gates de go-live:** prova do SNCR simulado na dispensação de controlado (api#103, complementa o api#63); perfil profissional para toda a equipe da farmácia (api#104); medir itens CATMAT sem forma farmacêutica antes do inventário de abertura (api#105).

## Entrega do 27b (2026-11-26)

Envio diário à BNAFAR (Portaria 5.713/2024: posição de estoque e saídas em D-1) e saldo inicial por planilha, para a cidade que deixa o Hórus. Spec [`superpowers/specs/2026-11-24-module-27b-bnafar-design.md`](../superpowers/specs/2026-11-24-module-27b-bnafar-design.md); contrato entre apps com as decisões Q1–Q48 em [`superpowers/plans/2026-11-24-module-27b-bnafar-contracts.md`](../superpowers/plans/2026-11-24-module-27b-bnafar-contracts.md); relato final em `evidencias/2026-11-26-mod27b-relato-final.md`.

| Repo | Intervalo | Commits | Suíte final |
|---|---|---|---|
| `api` | `ca2a92f0..e3e83842` | 46 | completa 13861/0 em `e3e83842` |
| `dashboard` | `51b1871..3b1f573` | 27 | 7959/0; tsc e build ok |
| `maintenance` | `14406bd..18e1c31` | 7 | 769/0; tsc, codegen e build ok |

**Migrações** (irreversíveis): cidade `20261125900001`–`20261125900004` (dados BNAFAR do lote, saída externa, remessas, importação do saldo inicial); plataforma `20261125600001` (apresentações OBM).

**Prova no navegador (2026-11-26),** contra o simulado `fake-bnafar` (serviço de dev na 8095):
- **Dashboard:** credencial com step-up e teste de conexão; CNES dos centrais; saldo inicial (modelo, `.xlsx` recusado, relatório, aplicar e desfazer); entrada com os campos de envio (AMPP, item sem VMP, fabricante internacional); complemento pela tela de pendências; saídas externas, inclusive da rede de frio com o envio desligado; remessa aceita e correção aceita.
- **Maintenance:** interruptores sem cascata e aviso do simulado; troca para homologação sem código e volta a produção com TOTP; carga OBM com as linhas recusadas (Q47) e lista de revisão; auditoria só com ids e contagens; 375 px.
- **Ressalvas:** a recusa de registro, o "Reenviar recusados" e o "ligar sem credencial" ficaram cobertos só por spec (api#187).

**Decisões de contrato na execução:** Q21 (CNPJ e VMP da vacina pelo YAML), Q22/Q23/Q40/Q41 (dado de origem só com o estoque; situação com o envio ligado; campos exigidos pelo interruptor), Q24 (desfazer do saldo inicial), Q25/Q42 (30 dias pelo envio; fim do reenvio no menor entre o fuso da cidade e o de Brasília), Q27 (resumo mensal de vacinas com saídas externas e aberturas desfeitas), Q28/Q29 (lote bloqueado ou vencido na saída externa), Q30/Q31/Q34 (purga em 30 dias; parte em processamento não se perde e mantém o alerta), Q32/Q33 (leitores e pendências paginadas), Q36–Q38/Q48 (códigos da planilha e fabricante internacional), Q39 (volta a produção com TOTP fora de produção), Q43–Q47 (campos para as telas e detalhe da recusa OBM).

**Pendências (board #2):** api#181 (homologação real; Alta), api#182 (YAML de imunobiológicos com CNPJ e VMP; Alta), api#183 (62 siglas do DM14 inferidas; Média), api#184 (central sem CNES próprio; Média), api#185 (teto de 7 MB antes do parse; Média), api#186 (importação parcialmente aplicada; Média), api#187 (provar "Reenviar recusados" pela tela; Baixa), api#188, api#189, api#190 e dashboard#32 (Baixa).

**Gates de go-live:** homologação real no SI-BNAFAR com os 6 gates do relato (api#181); YAML de vacinas antes de ligar o envio de imunobiológicos (api#182); central sem CNES próprio conferido com a cidade piloto (api#184); maintenance em produção (Q18, pendência da plataforma), e em produção a carga OBM é pela rake `medications:import_obm`; teto de corpo no proxy (api#185). Runbook de staging no relato final: o maintenance troca o ambiente antes de a cidade gravar a credencial; trocar de ambiente só com a fila vazia; o simulado nunca fora da rede interna.

Página de status: https://claude.ai/artifact/Fo8WF54XKaMUcW3sHDNyA9

**Importação do histórico do Hórus:** fora do 27b. O formato da base histórica ainda não foi publicado pelo Ministério (pesquisa de 2026-11-24); o saldo inicial por planilha cobre a entrada da cidade.

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
- 2026-11-24 — Desenho do 27b aprovado e `Planejado` (spec `superpowers/specs/2026-11-24-module-27b-bnafar-design.md`).
- 2026-11-26 — 27b **implementado, publicado e Entregue** (ver "Entrega do 27b").
