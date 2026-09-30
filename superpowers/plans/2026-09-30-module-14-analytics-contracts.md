# Módulo 14 — Analytics — contratos entre apps

Fonte única dos formatos que api, dashboard, admin e maintenance trocam. Os
planos por app citam este arquivo; mudança aqui muda os quatro planos.
Spec: `../specs/2026-09-30-module-14-analytics-design.md` · ADR `../../adr/0025.md`.

## 0. Tipos comuns

- **Cell** (contagem): `number` inteiro ≥ 0, ou `{ "suppressed": true }` quando o
  valor somado está em 1..4. Zero é `0`.
- **Rate** (percentual, 1 casa decimal): `number`, ou `{ "suppressed": true }`
  quando numerador **ou** denominador está em 1..4, ou `null` quando o
  denominador é 0 ("sem dado").
- **Period start**: data ISO `YYYY-MM-DD` — segunda-feira (`week`) ou dia 1
  (`month`), fuso `America/Sao_Paulo`.
- Séries (`series`) são arrays alinhados com `data.periods`.
- Supressão aplicada **depois** de somar período e recorte (`Admin::SmallCount`).
- **Total do grupo (decisão de 2026-09-30):** um valor exibido que é a soma, ou
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

## 1. Cidade — `GET /admin/api/analytics/:front`

`:front` ∈ `demand`, `quality`, `calibration`, `epidemiology`. Host da cidade,
sessão municipal. Só leitura.

**Acesso:** membership ativa com papel `analyst` ou `municipal_admin`. Operador
com grant, ou outro papel → `403 { "error": "forbidden_role" }`.

**Parâmetros:**

| Param | Frentes | Regra |
|---|---|---|
| `from`, `to` | todas | `YYYY-MM-DD`, obrigatórios, `from <= to`, `to <= ontem` (senão é truncado para ontem) |
| `granularity` | demand, quality, epidemiology | `week` (padrão) ou `month`; `week` até 104 períodos, `month` até 60 |
| `neighborhood_id` | demand, epidemiology | uuid de bairro da cidade (ativo ou não); `none` = sem bairro declarado |
| `health_unit_id` | demand (só séries por unidade), quality | uuid de unidade da cidade |
| `protocol_name` | demand, calibration, epidemiology | nome exato |
| `protocol_version` | calibration, epidemiology | inteiro; exige `protocol_name` |

Erros 422: `{ "error": "invalid_range" }` (datas ausentes/inválidas/invertidas
ou períodos demais), `{ "error": "invalid_neighborhood" }`,
`{ "error": "invalid_unit" }`, `{ "error": "invalid_protocol" }`.

**Unidades para o seletor:** `demand` e `quality` trazem sempre
`data.units: [ { "health_unit_id": "uuid", "name": "UBS ...", "active": true } ]`
— todas as unidades da cidade (ativas e inativas), ordenadas por nome,
independentes do recorte. É a fonte do seletor de unidade (o `analyst` não
lê `/attendance/units`).

**Envelope:**

```json
{
  "data": { "front": "demand", "granularity": "week", "from": "2026-06-01", "to": "2026-09-29",
            "filter": { "neighborhood_id": null, "health_unit_id": null, "protocol_name": null, "protocol_version": null },
            "periods": ["2026-06-01", "2026-06-08"], "...": "por frente" },
  "as_of": "2026-09-30T05:02:11Z",
  "stale": false
}
```

`as_of` = `finished_at` do último `analytics_runs` `succeeded` (ou `null` se nunca
houve; aí `data` vem com séries vazias e `stale: true`). `stale` = `as_of` nulo
ou com mais de 36 h.

### 1.1 `demand`

```json
{
  "triages": { "started": [Cell], "completed": [Cell], "aborted": [Cell] },
  "triages_total": { "started": Cell, "completed": Cell, "aborted": Cell },
  "by_tier":         [ { "tier": "alto", "series": [Cell], "total": Cell } ],
  "by_protocol":     [ { "protocol_name": "Respiratório", "series": [Cell], "total": Cell } ],
  "by_neighborhood": [ { "neighborhood_id": "uuid|null", "name": "Zona 7|Sem bairro", "total": Cell } ],
  "attendances_by_unit": [ { "health_unit_id": "uuid", "name": "UBS ...", "series": [Cell], "total": Cell } ],
  "requests_opened": [ { "kind": "return|referral", "series": [Cell], "total": Cell } ],
  "requests_closed": [ { "reason": "fulfilled|citizen_cancelled|dismissed", "series": [Cell], "total": Cell } ]
}
```

`by_tier`/`by_protocol`/`by_neighborhood` vêm de `triage.completed`
(`triage.started` para os totais de `triages.started`). `health_unit_id` recorta
só `attendances_by_unit`, `requests_*`. `neighborhood_id`/`protocol_name` recortam
só as métricas de triagem. Listas ordenadas por total desc (suprimido conta como 0
na ordenação), depois nome.

### 1.2 `quality`

```json
{
  "wait": { "buckets": [ { "bucket": "0-15", "series": [Cell], "total": Cell } ],
            "within_30_pct": [Rate], "within_30_pct_total": Rate },
  "appointments": [ { "status": "checked_in|no_show|expired|cancelled_by_citizen", "series": [Cell], "total": Cell } ],
  "no_show_pct": [Rate], "no_show_pct_total": Rate,
  "attendance_outcomes": [ { "outcome": "discharged|referred|return|left", "series": [Cell], "total": Cell } ],
  "left_pct": [Rate], "left_pct_total": Rate,
  "by_unit": [ { "health_unit_id": "uuid", "name": "UBS ...", "attendances": Cell,
                 "wait_within_30_pct": Rate, "no_show_pct": Rate, "left_pct": Rate } ]
}
```

Buckets sempre os 5, nesta ordem: `0-15`, `15-30`, `30-60`, `60-120`, `120+`.
`no_show_pct` = `no_show ÷ (checked_in + no_show)`. `left_pct` = `left ÷ Σ outcomes`.
`within_30_pct` = `(0-15 + 15-30) ÷ Σ buckets`.

### 1.3 `calibration` (período inteiro, sem série)

```json
{
  "versions": [ { "protocol_name": "Respiratório", "protocol_version": 3,
    "rows": [ { "tier": "alto", "total": Cell,
                "outcomes": { "discharged": Cell, "referred": Cell, "return": Cell, "left": Cell, "none": Cell },
                "shares":   { "discharged": Rate, "referred": Rate, "return": Rate, "left": Rate, "none": Rate } } ] } ]
}
```

`shares[x]` = `outcomes[x] ÷ total`. Versões ordenadas por nome, versão desc;
linhas por total desc.

### 1.4 `epidemiology`

```json
{
  "questions": [ { "protocol_name": "Arbovirose", "question_id": "febre", "prompt": "Teve febre?",
                   "answer_type": "boolean|enum",
                   "options": [ { "value": "true", "label": "Sim", "series": [Cell], "total": Cell } ] } ]
}
```

`prompt`/`options` vêm da versão mais recente (maior `version`), entre as
versões publicadas, ativas ou aposentadas (nunca rascunho), que marca a
pergunta `analytic`. Boolean: valores `"true"`/`"false"`, rótulos `Sim`/`Não`.
Enum: valor = rótulo = a string da opção. Sem pergunta marcada → `questions: []`.

## 2. Console do operador — `GET /city_analytics`

Host do console (`PlatformConsoleHost`), sessão de operador
(`Operators::BaseController`). Lê só `city_analytics_indicators` e `cities`.

Params: `from`, `to` (`YYYY-MM-DD`; padrão = últimas 12 semanas terminando na
semana anterior à atual; até 104 semanas; senão `422 { "error": "invalid_range" }`).

```json
{
  "data": {
    "weeks": ["2026-07-06", "2026-07-13"],
    "indicators": ["triages_started", "triages_completed", "attendances_closed",
                   "wait_within_30_pct", "no_show_pct", "left_pct"],
    "cities": [ { "id": "uuid", "slug": "curitiba", "name": "Curitiba", "uf": "PR",
                  "last_published_at": "2026-09-30T05:02:12Z|null",
                  "values": { "triages_started": [ 128, { "suppressed": true }, null ] } } ]
  }
}
```

Cada `values[indicator]` é alinhado com `weeks`: `number` (contagem inteira ou
percentual com 1 casa), `{ "suppressed": true }`, ou `null` (sem linha = sem dado).
Cidades ordenadas por nome; entram todas as de `status` ativo.

## 3. Maintenance — GraphQL (`CityType`)

```graphql
type AnalyticsIndicator { weekStart: ISO8601Date!  indicator: String!  value: Float  suppressed: Boolean! }
type AnalyticsStatus {
  lastRunStatus: String        # running | succeeded | failed | null (nunca rodou)
  lastSucceededAt: ISO8601DateTime
  lastPublishedAt: ISO8601DateTime
  lastError: String            # só da última execução se failed, ou erro de publicação
  stale: Boolean!              # lastSucceededAt nulo ou > 36 h
}
extend type City {
  analyticsIndicators(from: ISO8601Date!, to: ISO8601Date!): [AnalyticsIndicator!]!   # até 104 semanas
  analyticsStatus: AnalyticsStatus   # anulável, como counts/operations/profile: erro de cidade não anula `city`
}
```

`analyticsIndicators` lê o banco de plataforma; `analyticsStatus` lê
`analytics_runs` da cidade (como os demais campos operacionais do `CityType`).

## 4. Protocolo

Pergunta ganha `"analytic": true` opcional. Só válido com `answer_type`
`boolean` ou `enum`. O dashboard grava pelo fluxo de rascunho que já existe
(nenhum endpoint novo).

## 5. Papel

`analyst` em `Membership::ROLES`, fora de `PRIVILEGED_ROLES`. Rótulo na
interface: "Análise". Semente de dev: `analise@curitiba.demo`,
`analise@maringa.demo`.
