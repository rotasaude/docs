# Módulo 31 — Atenção psicossocial (CAPS)

- **Estado:** Entregue
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

## Entrega (2026-11-05)

| Repo | Commit | Observação |
|---|---|---|
| `api` | `34ce852f` | 66 commits; suíte completa 10545/0; migrações `20261104900001` (plataforma, SIGTAP) e `20261104900002`–`900004` (cidade: CAPS, BPA do 22, documentos do 19) |
| `dashboard` | `3671781` | 40 commits; suíte 5967/0 |
| `wpda` | `54b44bf` | 2 commits; filtro de borda `psychosocial` e specs de ausência; suíte 855/0 |
| `maintenance` | `ec86846` | rótulo do interruptor `psychosocial_care`; suíte 614/0 |
| `catalog` | `0b449be` | PR catalog#36 mergeado; F-31.5 e F-31.6 "em validação" |
| `docs` | contrato §1–§14, §15 P1–P22 e acréscimos P23–P62 | |

**Prova no navegador (2026-11-05, dev de Curitiba, só a PsychosocialCrew semeada):**

O sigilo passou em todos os pontos:
- **UBS:** o cartão do vínculo é genérico ("Em acompanhamento no CAPS desde …"), o marcador da linha do tempo não tem detalhe e nenhuma ação do CAPS aparece como procedimento.
- **Encaminhamento:** o motivo fica fora da fila, e o log não traz CPF, nome nem motivo.
- **wpda:** para um cidadão do CAPS AD, nada do CAPS aparece na sessão, no histórico, no resumo de saúde, nos agendamentos ou nos avisos, e nenhum SMS foi enfileirado.

Fluxos conferidos: acolhimento, matrícula única, PTS, ações (recusas múltiplas, familiar e cancelamento), presença em grupo com clique duplo (uma ação só), leitos, limite de noites, fechamento e reabertura do mês com step-up, `raas_layout_unavailable` (gate) e painel com "menos de 5".

Achados corrigidos antes do merge (P61 e P62):
- procedimento RAAS oferecido no acolhimento sem matrícula;
- acolhimento noturno oferecido em CAPS sem leito;
- leito liberado que continuava ocupado no mapa;
- noite contada para permanência de mesmo dia;
- saída prevista vencida;
- defeitos de tela.

**Decisões do usuário na execução:**
- **P46:** vínculo genérico, sem nome nem modalidade do CAPS.
- **P48:** o telefone fica. Verificação pública e atestado saem como "Centro de Atenção Psicossocial".

O coordenador aplicou a mesma linha em mais quatro pontos:
- **P49:** atestado sem especialidade nem CBO;
- **P51 e P53:** nome genérico em todas as fontes e detalhes fora do CAPS;
- **P59:** CAPS fora do público de campanhas.

Página de status: https://claude.ai/artifact/ViX8XZ4dWRhhwXxzBekM2e

**Gates de go-live** (board #2, "A triar"):
- layout da remessa RAAS (api#126);
- códigos, instrumentos e regras de quantidade do SIGTAP do CAPS (api#127);
- limite do CAPS III e listas da RAAS (api#128);
- cadastros do go-live: CNES local, CAPS, telefone de contato e APAC (api#129).

Resíduos de ciclo:
- api#130;
- ajuste de interface dashboard#23 (nome em minúsculas no mapa de leitos), com comentário em dashboard#17.

**Catálogo:** `0b449be` (catalog#36). F-31.5 e F-31.6 ficam "em validação" até os gates.

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
- 2026-11-05 — Executado nos quatro apps, provado no navegador, mergeado e publicado; estado `Entregue`.
