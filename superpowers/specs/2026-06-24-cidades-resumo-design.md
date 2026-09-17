# Cidades provisionadas + resumo de atividades — Design

**Data:** 2026-06-24
**Status:** aprovado (design); pendente plano de implementação
**Área:** `apps/api` (Rails 8, namespace read-only `Admin::Api`) + `apps/admin` (React/Vite)

## 1. Contexto e objetivo

O Admin Console já permite **provisionar** uma cidade (Setup → "Provisionar cidade",
operador apenas — ADR-0024). Falta o lado de **leitura**: visualizar as cidades
provisionadas e um **resumo das atividades** de cada uma, com **detalhe rico** por cidade
(recursos provisionados + KPIs + timeline de eventos + mini-gráficos).

A feature encaixa no padrão **read-only `Admin::Api`** (controller → query com
`Admin::Scoped` → hook → módulo), que já suporta visão cross-tenant (`:all`) via conexão
`rota_admin` (BYPASSRLS, ADR-0019).

## 2. Decisões (travadas no brainstorming)

| Tema | Decisão |
|---|---|
| Audiência | **Operador** (`platform_operator`), visão cross-tenant de plataforma. Gated como o provisionamento. |
| Profundidade | **Lista + detalhe rico** (timeline de `domain_events` + sparklines por cidade). |
| Métricas | **Todas**: conversas & triagens; mensagens & consentimentos; saúde do canal & última atividade; protocolos & eventos. |
| Período | **Volumes** respeitam o seletor `hoje/7d/30d` do shell; **estado** (canal ativo?, última atividade, conversas ativas, protocolos ativos) é **point-in-time**. |
| Abordagem | **A** — endpoints dedicados + agregação live (sem projeções por ora). |

## 3. Arquitetura

### 3.1 Backend

**Rotas** (`config/routes.rb`, dentro de `namespace :admin { namespace :api }`):

```ruby
get "cities",      to: "cities#index"
get "cities/:id",  to: "cities#show"
```

**Controller** `Admin::Api::CitiesController < Admin::Api::BaseController`
- `before_action :require_operator!` → `head :forbidden unless cross_tenant?` (operador).
- Herda do `BaseController`: `with_admin_connection` (rota_admin/BYPASSRLS → leitura
  cross-tenant), `resolve_scope` (parse do `period`; `current_municipality` é ignorado aqui).
- `index` → `Admin::CitiesQuery.call(period:)` → envelope próprio `{ data: { cities: [...] }, as_of }`.
- `show` → busca `Municipality` (admin connection); `head :not_found` se não existir →
  `Admin::CityDetailQuery.call(municipality:, period:)` + `Admin::CityTimelineQuery.call(municipality:)`.

**Queries** (`app/queries/admin/`), agregação **live sem N+1** — uma query agrupada por
métrica (`GROUP BY municipality_id`), costuradas em Ruby por `municipality_id`:

- `Admin::CitiesQuery` — `Municipality.all` + agregados (ver §4 para colunas exatas).
- `Admin::CityDetailQuery` — mesmas métricas para 1 cidade + **recursos provisionados** +
  KPIs com `spark` via `Admin::Api::Period#series`.
- `Admin::CityTimelineQuery` — `DomainEvent.where(municipality_id:).order(occurred_at: :desc).limit(50)`.

### 3.2 Frontend (`apps/admin/src`)

- `shell/modules.ts`: `ModuleId += "cities"`; novo item no grupo **Setup** (primeiro),
  `{ id: "cities", label: "Cidades", icon: "▤", visible: (u) => u.operator }`.
- `App.tsx`: `import { Cities }`; `case "cities": return <Cities/>`.
- `modules/Cities.tsx`: alterna **lista ↔ detalhe** por estado local `selectedCityId`
  (mesmo padrão de Triages↔trail), lendo `period` do shell.
  - **Lista**: `PageHeader` + `DataTable` (linha clicável) — colunas Cidade (name · uf),
    Status (`StatusDot`), Canal (ativo? + número), Última atividade (relativo), e
    métricas-chave (conversas ativas, triagens concluídas no período, mensagens, eventos).
    `EmptyState` (sem cidades, atalho p/ Provisionar) · `Skeleton` (loading) · `ErrorState`.
  - **Detalhe**: botão "← Cidades" + (1) Header; (2) `KpiGrid` com sparklines (volumes do
    período); (3) Painel "Recursos provisionados" (canal, termo LGPD vigente, alert
    recipients, protocolos ativos, 1º admin); (4) Painel "Atividade recente" (timeline).
- `lib/api.ts`: `getCities(period)`, `getCityDetail(id, period)`.
- `lib/types.ts`: `CitySummary`, `CityDetail`, `CityResources`, `TimelineEntry`.
- `hooks/useCities.ts`, `hooks/useCityDetail.ts` — espelham `useOverview` (refetch em
  mudança de period/id; estados loading/error).
- **Reuso total** de componentes (DataTable, KpiGrid, Sparkline, Panel, StatusDot,
  EmptyState, ErrorState, Skeleton, AsOfStamp) — sem novos primitivos de UI.

## 4. Métricas — definição exata (colunas/estados)

Período = `@period.from..@period.to`. Estado = ponto no tempo (sem filtro de período).

| Métrica | Tipo | Fonte |
|---|---|---|
| `conversations_active` | estado | `conversations` `state IN ('greeting','awaiting_consent','consented')`, `GROUP BY municipality_id` |
| `triages_done` | período | `triages` `status='completed'` e `completed_at` no período, `GROUP BY municipality_id` (coluna direta) |
| `triages_in_progress` | estado | `triages` `status='in_progress'`, `GROUP BY municipality_id` |
| `inbound` | período | `inbound_messages` `created_at` no período, `GROUP BY municipality_id` |
| `outbound` | período | `outbound_messages` `created_at` no período, `GROUP BY municipality_id` |
| `consents` | período | `consents` `given_at` no período, `GROUP BY municipality_id` (coluna direta) |
| `events` | período | `domain_events` `occurred_at` no período, `GROUP BY municipality_id` |
| `protocols_active` | estado | `protocol_definitions` `status='active'` e `municipality_id = <id>` (muni-específicos; protocolos de plataforma com `municipality_id IS NULL` ficam fora da contagem por-cidade), `GROUP BY municipality_id` |
| `channel` | estado | `municipality_channels` `active` + `display_phone_number` por muni |
| `last_activity_at` | estado | `MAX` entre `inbound_messages.created_at`, `outbound_messages.created_at`, `domain_events.occurred_at` por muni |

**Sparklines (detalhe)**: `Admin::Api::Period#series(relation, time_column)` (mesmo helper das
sparklines do Overview), por métrica de volume.

**Recursos provisionados (detalhe)**:
- `channel`: `MunicipalityChannel` (display_phone_number, phone_number_id, active).
- `consent_term`: `ConsentTerm` vigente = maior `published_at` (version, published_at).
- `alert_recipients`: lista `active` (channel, destination, escalation_order).
- `protocols_active`: lista `protocol_definitions` `status='active'` da cidade (name, version).
- `first_admin`: primeiro `Membership` `municipal_admin` ativo da cidade (email do user) ou,
  se ainda não aceito, o `Invitation` pendente (email, expires_at).

**Timeline**: `domain_events` (name → `type`; `payload`/`name` → `summary`; `occurred_at` → `at`).

## 5. Contratos JSON

`GET /admin/api/cities?period=7d`
```json
{ "data": { "cities": [ {
  "id": "uuid", "name": "Curitiba", "uf": "PR", "slug": "curitiba", "status": "active",
  "channel": { "active": true, "display_phone_number": "+5541..." },
  "last_activity_at": "2026-06-24T12:00:00Z",
  "metrics": {
    "conversations_active": 3, "protocols_active": 2,
    "triages_done": 12, "triages_in_progress": 4,
    "inbound": 80, "outbound": 75, "consents": 9, "events": 140
  } } ] }, "as_of": "2026-06-24T12:00:00Z" }
```

`GET /admin/api/cities/:id?period=7d`
```json
{ "data": {
  "city": { "id": "uuid", "name": "Curitiba", "uf": "PR", "slug": "curitiba",
            "ibge_code": "4106902", "status": "active",
            "channel": { "active": true, "display_phone_number": "+5541..." },
            "last_activity_at": "2026-06-24T12:00:00Z" },
  "resources": {
    "channel": { "display_phone_number": "+5541...", "phone_number_id": "PNID", "active": true },
    "consent_term": { "version": "v1", "published_at": "..." },
    "alert_recipients": [ { "channel": "email", "destination": "ops@...", "escalation_order": 0 } ],
    "protocols_active": [ { "name": "dengue", "version": 3 } ],
    "first_admin": { "email": "admin@...", "status": "active" }
  },
  "kpis": [ { "id": "triages_done", "label": "Triagens concluídas", "value": 12, "spark": [/* buckets */] } ],
  "timeline": [ { "at": "...", "type": "triage.completed", "summary": "..." } ]
}, "as_of": "..." }
```

## 6. Tratamento de erro

- Não-operador: item de nav oculto (`visible: u.operator`) **e** endpoints `403`
  (`require_operator!`) — defesa em profundidade.
- `show` com id inexistente → `404` → `ErrorState`.
- Período inválido → `422 invalid_scope` (já tratado no `BaseController`).
- Sem cidades → `EmptyState` (atalho p/ Provisionar).
- Rede/5xx → `ErrorState` + retry; loading → `Skeleton`. `as_of` em todo envelope.

## 7. Testes (test-first no backend)

- **Request specs** `spec/requests/admin/api/cities_spec.rb`:
  - `index`: operador vê **todas** as cidades (cross-tenant); não-operador → `403`;
    recorte de período reflete nos volumes; lista vazia.
  - `show`: devolve `resources`+`kpis`+`timeline`; id desconhecido → `404`; não-operador → `403`.
- **Query specs** `spec/queries/admin/`:
  - `cities_query_spec`: agregação por muni correta; fronteira de período (dentro/fora);
    point-in-time vs período; isolamento entre 2+ cidades.
  - `city_detail_query_spec`: recursos presentes; buckets do sparkline.
  - `city_timeline_query_spec`: ordem desc + limit.
  - Setup como os specs admin existentes (fixtures via rota_admin/bypass, cf.
    `spec/rls/tenant_isolation_spec.rb`).
- **Frontend**: `apps/admin` não tem harness de teste → **verificação manual** (subir admin
  como operador, abrir Cidades, conferir lista/detalhe/timeline). Vitest fica fora de escopo.

## 8. Fora de escopo (YAGNI)

- Projeções `dashboard_metrics` para a lista (Abordagem B) — adicionar só se a escala exigir.
- Escrita/ações na visão de cidades (suspender, editar) — read-only por critério §10.
- Acesso de `municipal_admin` à própria cidade — pode virar incremento depois.
- Testes de frontend automatizados.

## 9. Arquivos

**Criar (backend):** `app/controllers/admin/api/cities_controller.rb`,
`app/queries/admin/cities_query.rb`, `app/queries/admin/city_detail_query.rb`,
`app/queries/admin/city_timeline_query.rb`; specs em `spec/requests/admin/api/` e `spec/queries/admin/`.

**Alterar (backend):** `config/routes.rb` (2 rotas).

**Criar (frontend):** `src/modules/Cities.tsx` (+ subview de detalhe),
`src/hooks/useCities.ts`, `src/hooks/useCityDetail.ts`.

**Alterar (frontend):** `src/shell/modules.ts` (ModuleId + item nav), `src/App.tsx` (case),
`src/lib/api.ts` (2 funções), `src/lib/types.ts` (tipos).
