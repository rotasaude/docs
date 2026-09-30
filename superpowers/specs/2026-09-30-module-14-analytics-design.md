# Módulo 14 — Analytics (F-14.1 a F-14.9) — design

**Data:** 2026-09-30
**Status:** aprovado em conversa (2026-09-30), aguardando revisão do texto
**Afeta:**
- `apps/api`:
  - banco de cada cidade: `analytics_daily_facts` e `analytics_runs` novas;
  - banco de plataforma: `city_analytics_indicators` nova;
  - papel `analyst` (não privilegiado);
  - `Analytics::Consolidate` (um consolidador por frente), `ConsolidateAnalyticsJob` recorrente por cidade, rake `city:analytics:rebuild`;
  - `Analytics::Publish` (indicadores fixos da cidade para a plataforma);
  - rotas `GET /admin/api/analytics/*` (cidade), `GET /city_analytics` no host do console (`Operators::CityAnalyticsController`), campos novos no `CityType` do GraphQL de manutenção;
  - schema de protocolo: `analytic: true` em pergunta `boolean`/`enum` (cópia do `contracts`).
- `apps/dashboard`: área "Analytics" (Demanda, Qualidade, Calibração, Epidemiologia), caixa "Usar em Analytics" no `ProtocolEditor`, papel novo em Equipe.
- `apps/admin`: tela "Analytics das cidades".
- `apps/maintenance`: indicadores e estado do pipeline na ficha da cidade.
- `contracts`: `analytic` no schema de protocolo.
- `apps/wpda`: nada.

**ADR:** `docs/adr/0025.md` · **Módulo:** `docs/modulos/14--analytics.md`

**Fora desta entrega:** data warehouse externo, exportação CSV, painéis configuráveis pelo usuário, comparação entre cidades no dashboard da cidade, agregação de respostas `integer`/`text`, mapa/geometria (herda o "fora" do ADR 0023), alertas automáticos de surto, supressão complementar (subtração entre recortes).

## 1. Ponto de partida

- **Painéis do módulo 05 são ao vivo** (ADR 0022): leem o banco da cidade na hora, envelope `{ data:, as_of: }`, `Admin::Api::BaseController`. Respondem "como está agora"; nenhum guarda série longa.
- **`dashboard_metrics`** (ADR 0010) guarda contadores incrementais `(dimension, period, key)` para o painel Saúde. Não tem recorte por bairro, unidade ou versão de protocolo, e não serve de base para este módulo.
- **Supressão existente:** `Admin::SmallCount` (1–4 → `{ suppressed: true }`, depois de agregar, 0 continua 0) e `Admin::NeighborhoodFilter` (ADR 0023). Hoje só vale com filtro de bairro ligado.
- **Retenção do dado cru:** `domain_events` e `platform_events` com TTL de 12 meses (ADR 0014/módulo 07); a revogação aborta a triagem (`aborted_by_revocation`) e o `AnonymizeRevokedTriageJob` apaga o conteúdo clínico. Série sobre o dado cru morre com ele.
- **Dados que o módulo lê (banco da cidade):**
  - `triages`: `status`, `tier` (texto do protocolo, não enum), `priority`, `protocol_name`, `protocol_definition_id`, `neighborhood_id` (cópia imutável, ADR 0023), `answers` (jsonb por id de pergunta), `created_at`, `completed_at`.
  - `protocol_definitions`: `name`, `version`, `definition` (jsonb).
  - `attendances`: `health_unit_id`, `triage_id`, `checked_in_at`, `called_at`, `closed_at`, `status`, `outcome` (`discharged`, `referred`, `return`, `left`).
  - `appointments`: `health_unit_id`, `status` (`scheduled`, `confirmed`, `checked_in`, `cancelled_by_citizen`, `expired`, `no_show`), `ended_at`.
  - `appointment_requests`: `kind` (`return`, `referral`), `target_unit_id`, `status`, `closed_reason`, `created_at`, `closed_at`.
- **Schema de protocolo:** `answer_type` ∈ `boolean`, `enum`, `integer`, `text`; `options` obrigatório em `enum`. A fonte é o `contracts`; o api valida contra uma cópia (`config/protocols/schema.json`).
- **Editor de protocolo** mora no dashboard (`ProtocolEditor.tsx`, `lib/editor.ts`); publicar/ativar passa pelo ciclo assinado (ADR 0016).
- **Jobs recorrentes de cidade:** `config/recurring.yml`; o scheduler de cada cidade agenda na fila do banco dela e o job roda só naquela cidade (Plano 5). Fuso fixo `America/Sao_Paulo` (api#27).
- **Escrita na plataforma a partir da cidade** já existe (`Platform.audit` em jobs).
- **Acesso aos painéis:** hoje qualquer membership ativa lê `/admin/api`, e o operador com grant (Plano 3B) também (`allow_operator_grant_access`).
- **Maintenance:** GraphQL `CityType` já expõe dado operacional da cidade (`CityOperationsType`, `DashboardMetricType`), sem dado de cidadão.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| D1 | Quatro frentes no Ciclo 1: demanda por território, qualidade operacional, calibração de protocolo, epidemiológica. |
| D2 | Públicos: dashboard da cidade; console do operador (`admin`) e `maintenance` veem só **agregados por cidade**. |
| D3 | A plataforma recebe um **conjunto fixo** de indicadores semanais da cidade inteira, sem bairro, unidade, protocolo ou pergunta. |
| D4 | Pipeline **B**: consolidação diária em tabela anônima no banco da cidade; publicação suprimida no banco de plataforma. Dado até D-1. |
| D5 | Epidemiologia só sobre perguntas **marcadas** `analytic` pelo autor, e só `boolean`/`enum`; a marcação passa pelo ciclo assinado. |
| D6 | Papel novo `analyst` (só leitura analítica, sem step-up); `analyst` e `municipal_admin` leem o Analytics. |
| D7 | Supressão **1–4 sempre**, com ou sem recorte, na cidade e na plataforma. |
| D8 | Agregado consolidado **fica** após a revogação (dado anonimizado, LGPD art. 12). |
| D9 | Retenção dos agregados: **5 anos**. |
| D10 | A cada execução, reconsolida os **últimos 30 dias** (idempotente); desfecho que chegar depois disso não entra. |
| D11 | Espera em **faixas** (0–15, 15–30, 30–60, 60–120, 120+ min), não mediana. |
| D12 | (achado ao conferir o código) O operador com grant **não** lê `/admin/api/analytics`: o Analytics da cidade é só de papéis locais, e a plataforma só vê o publicado (D2). |
| D13 | (achado ao conferir o código) `tier` é texto de cada protocolo, sem escala comum entre cidades; por isso o conjunto da plataforma não tem "triagens de tier alto". |

## 3. Dados

### 3.1 `analytics_daily_facts` (banco da cidade)

Uma linha = uma contagem de um dia num recorte. Nenhuma coluna de pessoa.

| Coluna | Tipo | Regra |
|---|---|---|
| `id` | bigint | |
| `day` | date | NOT NULL; dia no fuso da cidade |
| `metric` | string | NOT NULL; check na lista da §3.4 |
| `health_unit_id` | uuid | NULL quando a métrica não é por unidade |
| `neighborhood_id` | uuid | NULL = sem bairro declarado ou métrica não territorial |
| `protocol_name` | string | |
| `protocol_version` | integer | |
| `tier` | string | texto do protocolo |
| `question_id` | string | só em `epi.answer` |
| `dim` | string | NOT NULL, default `''`; desfecho, estado, faixa, opção de resposta |
| `value` | integer | NOT NULL, ≥ 1 (linha com zero não é gravada) |
| `consolidated_at` | datetime | NOT NULL |

- Índice único `NULLS NOT DISTINCT` em (`day`, `metric`, `health_unit_id`, `neighborhood_id`, `protocol_name`, `protocol_version`, `tier`, `question_id`, `dim`).
- Índices de leitura: (`metric`, `day`), (`metric`, `neighborhood_id`, `day`), (`metric`, `health_unit_id`, `day`).
- Sem FK para `health_units`/`neighborhoods` (o fato sobrevive a desativação; unidade e bairro nunca são apagados, só desativados).
- A contagem é gravada **crua**; a supressão é feita na leitura, depois de somar período e recorte.

### 3.2 `analytics_runs` (banco da cidade)

| Coluna | Tipo | Regra |
|---|---|---|
| `id` | uuid | |
| `window_from`, `window_to` | date | NOT NULL |
| `kind` | string | `scheduled` ou `rebuild` |
| `status` | string | `running`, `succeeded`, `failed` (check) |
| `started_at`, `finished_at` | datetime | |
| `published_at` | datetime | quando a publicação na plataforma deu certo |
| `error` | string | classe + mensagem truncada em 500; nunca payload |

Purga de linhas com mais de 90 dias no próprio job.

### 3.3 `city_analytics_indicators` (banco de plataforma)

| Coluna | Tipo | Regra |
|---|---|---|
| `id` | uuid | |
| `city_id` | uuid | NOT NULL, FK `cities` |
| `week_start` | date | NOT NULL; segunda-feira, fuso da cidade |
| `indicator` | string | NOT NULL; check na lista da §5.2 |
| `value` | decimal(8,2) | NULL quando suprimido |
| `suppressed` | boolean | NOT NULL; check `suppressed = (value IS NULL)` |
| `published_at` | datetime | NOT NULL |

Único em (`city_id`, `week_start`, `indicator`). Upsert na publicação. Purga com mais de 5 anos.

### 3.4 Métricas

| `metric` | Dia | Recorte | `dim` |
|---|---|---|---|
| `triage.started` | `created_at` | bairro, protocolo, versão | — |
| `triage.completed` | `completed_at` | bairro, protocolo, versão, tier | — |
| `triage.aborted` | `created_at` | bairro, protocolo, versão | `timeout`, `cancellation`, `revocation` |
| `attendance.checked_in` | `checked_in_at` | unidade | método de check-in |
| `attendance.closed` | `closed_at` | unidade | `outcome` |
| `attendance.wait` | `called_at` | unidade | faixa (`0-15`, `15-30`, `30-60`, `60-120`, `120+`) |
| `appointment.ended` | `ended_at` | unidade | `checked_in`, `no_show`, `expired`, `cancelled_by_citizen` |
| `request.opened` | `created_at` | unidade-alvo | `kind` |
| `request.closed` | `closed_at` | unidade-alvo | `closed_reason` |
| `calibration.outcome` | `completed_at` da triagem | protocolo, versão, tier | `outcome` do atendimento ligado (`none` se não houve atendimento encerrado) |
| `epi.answer` | `completed_at` | bairro, protocolo, versão, `question_id` | opção (`true`/`false` ou valor do `enum`) |

Regras:
- Atendimento não tem bairro: o território da demanda é o bairro da triagem; atendimento, agendamento e pedido são recortados por unidade.
- `epi.answer` conta só triagens `completed` cuja versão de protocolo (`protocol_definition_id` → `definition`) marca a pergunta `analytic` com `answer_type` `boolean`/`enum`, e só respostas que batem com uma opção declarada (resposta fora da lista é ignorada).
- `calibration.outcome` usa o atendimento com `triage_id` = a triagem (`attendances.triage_id` é único: no máximo um). O atendimento de retorno, que nasce de agendamento, não entra na calibração.

## 4. Consolidação

### 4.1 `ConsolidateAnalyticsJob`

- Recorrente de cidade, `every day at 2am America/Sao_Paulo`, fila `housekeeping`.
- Janela: de `hoje − 30` até `ontem` (inclusive). O dia corrente nunca é consolidado.
- Advisory lock por cidade (`pg_try_advisory_lock`); se outro run estiver ativo, sai sem fazer nada e sem gravar `analytics_runs`.
- Cria `analytics_runs` `running`; numa **transação única**: apaga os fatos da janela e regrava a partir do cru, um consolidador por frente (`Analytics::Consolidate::Demand`, `::Quality`, `::Calibration`, `::Epidemiology`), cada um com `INSERT ... SELECT ... GROUP BY` no SQL.
- Depois do commit: `Analytics::Publish` (§5); falha na publicação marca o run `succeeded` com `published_at` nulo e o erro registrado; a próxima execução republica.
- Purga: fatos com `day` < hoje − 5 anos; `analytics_runs` com mais de 90 dias.
- Falha em qualquer consolidador: rollback da transação, run `failed` com o erro; os fatos anteriores ficam intactos.

### 4.2 Revogação

A revogação não toca `analytics_daily_facts`. **Triagem revogada** = `aborted_by_revocation` **ou** triagem cuja própria conversa tem consentimento revogado (o `RevokeConsent` só aborta triagem `in_progress`; a revogada depois de concluída continua `completed` no cru). Triagem revogada conta só em `triage.started` e em `triage.aborted`/`revocation`, sem bairro, e nunca em `triage.completed`, `calibration.outcome` ou `epi.answer`. Dentro da janela de 30 dias, a revogação passa a valer na próxima execução; fora dela, o número consolidado fica. Registrado no ADR 0025.

### 4.3 `city:analytics:rebuild[from,to]`

Rake por cidade (`CITY=slug`) ou `:all`. Reconsolida a janela em blocos de 30 dias com o mesmo código do job (`kind: rebuild`), limitado ao cru que ainda existe; republica as semanas afetadas. Usado no primeiro deploy e para corrigir bug de consolidação.

## 5. Publicação para a plataforma

### 5.1 `Analytics::Publish`

Calcula, a partir de `analytics_daily_facts`, os indicadores das semanas tocadas pela janela, aplica a supressão (§6.3) e faz upsert em `city_analytics_indicators` via `PlatformRecord`. Idempotente por (cidade, semana, indicador).

### 5.2 Conjunto fixo

| `indicator` | Cálculo (semana, cidade inteira) |
|---|---|
| `triages_started` | Σ `triage.started` |
| `triages_completed` | Σ `triage.completed` |
| `attendances_closed` | Σ `attendance.closed` |
| `wait_within_30_pct` | (faixas `0-15` + `15-30`) ÷ Σ `attendance.wait` × 100 |
| `no_show_pct` | `no_show` ÷ (`checked_in` + `no_show`) de `appointment.ended` × 100 |
| `left_pct` | `left` ÷ Σ `attendance.closed` × 100 |

Taxa: suprimida se numerador **ou** denominador estiver em 1–4. Com denominador 0, o indicador não é gravado naquela semana ("sem dado").

## 6. API

### 6.1 Cidade: `GET /admin/api/analytics/{demand,quality,calibration,epidemiology}`

- Controller próprio (`Admin::Api::AnalyticsController` ou um por frente) **sem** `allow_operator_grant_access` e com gate de papel: `analyst` ou `municipal_admin` ativos na cidade; os demais → 403.
- Parâmetros: `from`, `to` (datas), `granularity` (`week` | `month`); recortes opcionais `neighborhood_id`, `health_unit_id`, `protocol_name` (e `protocol_version` em calibração/epidemiologia). Limite: 104 semanas em `week`, 60 meses em `month`; fora disso 422. Recorte inválido → 422 (mesmo contrato de `Admin::NeighborhoodFilter::Invalid`).
- Envelope `{ data:, as_of: }` com `as_of` = `finished_at` do último run `succeeded` (nulo se nunca houve) e `stale: true` quando `as_of` tem mais de 36 h.
- Cada frente devolve séries por período e uma tabela do período inteiro; toda célula numérica passa por `Admin::SmallCount.wrap` **depois** de somar; taxas seguem a regra da §5.2.
- Epidemiologia devolve, por pergunta marcada, o `prompt` e as opções da versão mais recente que a marcou, e os valores por opção.

### 6.2 Console do operador: `GET /city_analytics?from&to` (host do console)

Sessão de operador; lê só `city_analytics_indicators` (cidades × semanas × indicadores) e o nome/slug da cidade. Nunca abre banco de cidade.

### 6.3 Supressão

`Admin::SmallCount` passa a valer sempre dentro de `analytics/*` (não só com filtro). Na plataforma, o valor suprimido é gravado como `value NULL, suppressed true` — o número 1–4 nunca sai do banco da cidade.

**Total do grupo (decisão de 2026-09-30):** um valor exibido que é a soma, ou
uma taxa, de partes exibidas na mesma resposta fica **oculto** sempre que
qualquer dessas partes estiver oculta. Concretamente:
- `total` de uma linha com série → oculto se qualquer célula da série for oculta;
- célula de período de um agregado (`triages.*[p]`) → oculta se qualquer parte
  daquele período (tier, protocolo) for oculta;
- `triages_total.*` → oculto se qualquer célula de `triages.*` ou qualquer
  `total` de `by_tier`/`by_protocol`/`by_neighborhood` for oculto;
- toda Rate → oculta se qualquer parte do numerador ou do denominador for
  oculta (faixas de espera, estados de agendamento, desfechos);
- calibração: se qualquer `outcomes[x]` de uma linha for oculto, `total` e
  todos os `shares` da linha ficam ocultos;
- epidemiologia: `total` da opção oculto se qualquer célula da série for oculta.
Cruzar tabelas diferentes continua possível (risco residual registrado no ADR 0025).

### 6.4 Maintenance (GraphQL)

- `City.analyticsIndicators(from:, to:)` → lista de `{ weekStart, indicator, value, suppressed }` de `city_analytics_indicators`.
- `City.analyticsStatus` → `{ lastSucceededAt, lastRunStatus, lastError, lastPublishedAt, stale }` lido de `analytics_runs` da cidade.

## 7. Protocolo

- Schema (`contracts` → cópia no api): propriedade opcional `analytic` (boolean) na pergunta; a validação recusa `analytic: true` quando `answer_type` é `integer` ou `text`.
- A marcação é parte da definição da versão: nova versão, ciclo assinado, ativação — nada muda em versão já publicada.
- `ProtocolEditor`: caixa "Usar em Analytics" visível só para `boolean`/`enum`, com a dica "respostas desta pergunta aparecerão agregadas por bairro, nunca por pessoa".

## 8. Dashboard

- Item de menu "Analytics" para `analyst`/`municipal_admin`; quatro abas.
- Cada aba: seletores de período e granularidade, recortes (bairro, unidade, protocolo conforme a frente), gráfico de série e tabela; célula suprimida = "oculto" (termo do módulo 11); carimbo "dados até DD/MM"; aviso "dados desatualizados" com `stale`; estado vazio "ainda sem dados consolidados".
- Calibração: tabela tier × desfecho por versão, com a proporção por linha.
- Epidemiologia: uma série por opção de cada pergunta marcada; sem perguntas marcadas, o texto explica como marcar no editor.
- Equipe: papel `analyst` na lista de papéis concedíveis (sem step-up).

## 9. Admin e maintenance

- `admin`: tela "Analytics das cidades" — tabela cidade × indicador da semana escolhida, com a tendência das últimas 12 semanas por célula; "oculto" e "sem dado" distintos.
- `maintenance`: bloco "Analytics" na ficha da cidade com `analyticsStatus` (atraso em destaque) e os indicadores das últimas 12 semanas.

## 10. Testes

### 10.1 api (RSpec, TDD)

- **Consolidadores:** um spec por métrica da §3.4 (entra, não entra, borda de dia no fuso da cidade); `epi.answer` ignora pergunta não marcada, `integer`/`text` e resposta fora das opções; `calibration.outcome` com 0, 1 e 2 atendimentos.
- **Job:** janela de 30 dias; desfecho atrasado dentro da janela entra, fora não; idempotência (duas execuções = mesmos fatos); falha num consolidador preserva os fatos anteriores; lock com threads (dois runs concorrentes → um só); purga de 5 anos e de 90 dias.
- **Revogação:** dentro da janela sai na próxima execução; fora, o fato fica.
- **Publish:** indicadores e taxas; supressão de numerador/denominador; idempotência; falha da plataforma não derruba o run.
- **Request (`type: :request`):** papéis (analyst, municipal_admin → 200; outros → 403; operador com grant → 403); limites de período; recortes inválidos; soma antes de suprimir (dois dias com 3 → 6 visível; um dia com 3 → oculto).
- **Console e GraphQL:** operador lê só plataforma; campos novos do `CityType`.
- **Schema:** `analytic` aceito em boolean/enum, recusado em integer/text.
- **Migrações:** spec de down/up das três tabelas.
- **Guardas:** `VALID_RANGE` 1..25 em `spec/adr_pointers_spec.rb`.

### 10.2 Suíte de invariante

`spec/invariants/analytics_invariants_spec.rb`, uma seção por invariante do ADR 0025.

### 10.3 dashboard, admin, maintenance (Vitest)

Célula "oculto", carimbo e `stale`, estado vazio, recortes, caixa do editor só em boolean/enum, papel em Equipe; no admin "oculto" × "sem dado"; no maintenance o atraso.

### 10.4 Prova no navegador (com o usuário)

Entrar como `analise@curitiba.demo`, ver as quatro abas com a semente, filtrar por bairro até aparecer "oculto"; marcar pergunta num rascunho no editor; ver os indicadores no console do operador e o estado no maintenance.

## 11. Semente de dev

`analise@curitiba.demo` e `analise@maringa.demo` com `analyst` (TOTP fixo como os demais); protocolo de dev com uma versão nova que marca 2 perguntas (`boolean` e `enum`), criada pelo `Protocols::SaveDraft` e ativada como na semente do ciclo assinado; ~6 meses de triagens, atendimentos, agendamentos e pedidos retroativos, coerentes com a semente realista (bairros e unidades reais, distribuição plausível por dia da semana); ao fim, `city:analytics:rebuild`.

## 12. Rollout

Runbook `operacao/analytics.md`:

1. Migração de cidade e de plataforma, só aditivas; `city:migrate:all` da imagem nova antes do tráfego.
2. Ordem: api → dashboard → admin → maintenance (os frontends dependem das rotas/campos novos).
3. Depois do deploy: `city:analytics:rebuild:all` desde o dado cru mais antigo de cada cidade.
4. Acompanhar `analyticsStatus` no maintenance nas primeiras execuções.

## 13. F-IDs

| F-ID | Funcionalidade | Apps |
|---|---|---|
| F-14.1 | Consolidação diária anônima por cidade (janela de 30 dias, runs, purga de 5 anos, rebuild) | api |
| F-14.2 | Papel `analyst` e área Analytics da cidade, com supressão 1–4 sempre | api, dashboard |
| F-14.3 | Demanda por território (triagens por bairro/protocolo/tier; atendimentos e pedidos por unidade) | api, dashboard |
| F-14.4 | Qualidade operacional (espera em faixas, falta, expiração, saiu sem atendimento, retorno) | api, dashboard |
| F-14.5 | Calibração de protocolo (versão × tier × desfecho) | api, dashboard |
| F-14.6 | Perguntas analíticas no protocolo (`analytic` no schema e no editor) | contracts, api, dashboard |
| F-14.7 | Epidemiologia por perguntas marcadas, por bairro | api, dashboard |
| F-14.8 | Indicadores fixos publicados na plataforma e tela no console do operador | api, admin |
| F-14.9 | Analytics no maintenance (indicadores e estado do pipeline) | api, maintenance |

## 14. Riscos

- **Subtração entre recortes:** alternar recortes e períodos ainda pode isolar uma contagem pequena por diferença (herdado do ADR 0023 e do módulo 12). Aceito no Ciclo 1: só papéis da própria cidade leem o detalhe; a plataforma vê só a cidade inteira.
- **Fuso fixo** `America/Sao_Paulo` (api#27): o "dia" herda o problema.
- **Desfecho depois de 30 dias** não entra na calibração; a janela pode crescer se o volume permitir.
- **`tier` é texto do protocolo:** renomear a faixa numa versão nova separa as séries; a calibração é por versão justamente por isso.
- **Rebuild limitado ao cru:** antes do primeiro deploy, o que já foi purgado (12 meses de eventos; conteúdo de revogados) não volta.
- **Volume:** `INSERT ... SELECT` sobre 30 dias por cidade é barato no Ciclo 1; se crescer, particionar `analytics_daily_facts` por ano.
