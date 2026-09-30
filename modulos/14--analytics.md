# Módulo 14 — Analytics

- **Estado:** Fechado
- **Tipo:** MVP

## Escopo

Padrões ao longo do tempo, não o "agora" dos painéis do módulo 05 (ADR 0022):
demanda por território, qualidade operacional, calibração de protocolo e
epidemiologia. Tudo sai de uma **consolidação diária anônima** no banco de
cada cidade (ADR 0025): contagens por dia e recorte, sem coluna de pessoa,
refeitas para os últimos 30 dias a cada execução, com dado até D-1, guardadas
por 5 anos.

**Entregue e verificado (F-14.1 a F-14.9):**
- consolidação diária por cidade, execuções registradas, purga de 5 anos e
  reconstrução por rake;
- papel `analyst` (só leitura, sem step-up) e área Analytics no dashboard para
  `analyst` e `municipal_admin`; supressão de 1 a 4 sempre, depois de somar
  período e recorte;
- demanda: triagens por bairro, protocolo e tier; atendimentos e pedidos de
  agendamento por unidade;
- qualidade: espera do check-in à chamada em faixas, falta, expiração, "saiu
  sem atendimento", retorno;
- calibração: versão do protocolo × tier × desfecho do atendimento;
- perguntas `analytic` (só `boolean`/`enum`) marcadas no protocolo, dentro do
  ciclo assinado;
- epidemiologia: respostas das perguntas marcadas, por bairro;
- seis indicadores semanais da cidade inteira, já suprimidos, publicados no
  banco de plataforma e lidos pelo console do operador;
- no maintenance, esses indicadores e o estado do pipeline de cada cidade.

**Fora, por enquanto:** data warehouse externo, exportação CSV, painéis
configuráveis, comparação entre cidades no dashboard da cidade, agregação de
respostas `integer`/`text`, mapa e geometria, alertas automáticos de surto,
supressão complementar contra subtração entre recortes.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0025 | Consolidação diária anônima, supressão de 1 a 4 sempre, perguntas marcadas, papel `analyst`, conjunto fixo para a plataforma, retenção e revogação |
| 0023 | Bairro copiado na triagem e regra de supressão de origem |
| 0016 | Ciclo assinado, por onde passa a marcação `analytic` |
| 0020 | Banco por cidade; a plataforma só lê o publicado |

## Superfícies

| Superfície | Papel previsto |
|---|---|
| `api` | `analytics_daily_facts`, `analytics_runs` (cidade); `city_analytics_indicators` (plataforma); `ConsolidateAnalyticsJob`, `city:analytics:rebuild`; `GET /admin/api/analytics/*`; `GET /city_analytics` (console); `analyticsIndicators` e `analyticsStatus` no GraphQL de manutenção; `analytic` no schema de protocolo |
| `admin` | Tela "Analytics das cidades" (indicadores fixos por semana) |
| `dashboard` | Área Analytics (Demanda, Qualidade, Calibração, Epidemiologia); caixa "Usar em Analytics" no editor de protocolo; papel em Equipe |
| `maintenance` | Bloco Analytics na ficha da cidade: estado do pipeline e indicadores |
| `wpda` | — |

## Pré-requisitos

- Módulos 03 (triagem), 08 (agendamento), 09 (unidades), 11 (bairros) e 13
  (atendimento) como fonte.
- Módulo 05 (Dashboard) — `Admin::SmallCount` e envelope dos painéis.

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-14.1 | Consolidação diária anônima por cidade (janela de 30 dias, execuções registradas, purga de 5 anos, reconstrução) | api | 0025 |
| F-14.2 | Papel `analyst` e área Analytics da cidade, com supressão de 1 a 4 sempre | api, dashboard | 0025, 0023 |
| F-14.3 | Demanda por território: triagens por bairro/protocolo/tier; atendimentos e pedidos por unidade | api, dashboard | 0025, 0023 |
| F-14.4 | Qualidade operacional: espera em faixas, falta, expiração, saiu sem atendimento, retorno | api, dashboard | 0025, 0019 |
| F-14.5 | Calibração de protocolo: versão × tier × desfecho do atendimento | api, dashboard | 0025, 0016 |
| F-14.6 | Perguntas analíticas no protocolo (`analytic` em `boolean`/`enum` no schema e no editor) | contracts, api, dashboard | 0025, 0016 |
| F-14.7 | Epidemiologia por perguntas marcadas, por bairro | api, dashboard | 0025, 0023 |
| F-14.8 | Indicadores semanais fixos publicados na plataforma e tela no console do operador | api, admin | 0025, 0020 |
| F-14.9 | Analytics no maintenance: indicadores e estado do pipeline | api, maintenance | 0025 |

## Riscos herdados

- **Subtração entre recortes** ainda pode isolar contagem pequena (herdado do
  ADR 0023 e do módulo 12); aceito no Ciclo 1, detalhe só para papéis da
  própria cidade.
- **Fuso fixo** `America/Sao_Paulo` (api#27): o "dia" herda o problema.
- **Desfecho depois de 30 dias** não entra na calibração.
- **Reconstrução limitada ao cru:** o que já foi purgado antes do primeiro
  deploy não volta.

## Critério de fechamento do módulo

- F-14.1 a F-14.9 verificadas.
- Suíte de invariante (`spec/invariants/analytics_invariants_spec.rb`): sem
  coluna de pessoa; nenhum valor de 1 a 4 fora do banco da cidade; epidemiologia
  só de pergunta marcada `boolean`/`enum`; consolidação idempotente; revogação
  não altera fato fora da janela; purga só acima de 5 anos; só `analyst` e
  `municipal_admin` leem, operador nunca; plataforma nunca abre banco de cidade.
- Runbook `operacao/analytics.md`.

## Histórico

- 2026-09-30 — Escopo decidido com o usuário; ADR 0025 e spec
  `superpowers/specs/2026-09-30-module-14-analytics-design.md`. F-14.1 a
  F-14.9 criados. Módulo passa de `Stub` a `Planejado`.
- 2026-09-30 — F-14.1 a F-14.9 entregues e publicados: contracts `3747c80`
  (tag `protocols-v1.2.0`), api `ecadd09` (suíte 2960/0; invariantes do ADR 0025
  com teste de mutação), dashboard `c4fd035` (765 testes), admin `6436f23`
  (87), maintenance `baf3219` (166). Decidido na execução: total ou taxa
  oculto quando qualquer parte da mesma resposta é oculta (ADR 0025); triagem
  concluída e depois revogada tratada como revogada na consolidação. Prova no
  navegador feita com a semente de dev (analista de Maringá, console do
  operador, aba Analytics do maintenance); o total de iniciadas herdava as
  partes das concluídas e foi corrigido. Runbook
  [`operacao/analytics.md`](../operacao/analytics.md). Módulo passa a
  `Entregue`; falta a verificação (dossiê por F-ID).
- 2026-09-30 — Verificação do módulo (dossiê por F-ID,
  [`relatorios/2026-09-30-verificacao-modulo-14.md`](../relatorios/2026-09-30-verificacao-modulo-14.md)):
  nenhum GAP-IMPL; lacunas fechadas antes de subir — só semanas fechadas na
  plataforma, calibração vazia sem consolidação, vazamento de `by_unit` da
  qualidade corrigido, enum exige `options` (contracts `protocols-v1.3.0`), 28
  testes novos (api `22535ac`, suíte 3005/0; dashboard `f10ca49`; admin
  `60ed688`). F-14.1 a F-14.9 `Verified` por aprovação do usuário. Critério de
  fechamento cumprido; módulo `Fechado`. Com ele, os 14 módulos do MVP (Ciclo 1)
  estão fechados.
