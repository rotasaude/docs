# Dashboard — Fatia 1: Fundação + painel Overview — Design

**Data:** 2026-06-26
**Módulo:** 05 — Dashboard (`docs/modulos/05--dashboard.md`), funcionalidades F-05.1, F-05.3, F-05.4.
**Repo alvo:** `rotasaude/dashboard` (local: `apps/dashboard`).

## Contexto

O `dashboard` é o painel operacional da **cidade**, tenant-scoped, dono `municipal_admin`/`viewer`,
read-only na fase 1 (ADR-0003 escopa por `municipality_id` via RLS; ADR-0010 leitura nunca recalcula).
Os 9 painéis (Overview + Ingestão, Conversas, Consentimento, Triagens, Classificação, Protocolos,
Filas, Eventos) **espelham as views já construídas no `admin`** e consomem a **mesma API read-only**
`/admin/api/*` (namespace `Admin::`), que já existe. A diferença para o `admin` (cross-tenant,
`platform_operator`): o dashboard é **single-tenant** (sem seletor de cidade).

`apps/dashboard/src` hoje só tem scaffold (App/main/theme). Esta fatia constrói a **fundação** do app
+ o **painel Overview** como vertical slice, validando o padrão ponta-a-ponta contra a API real antes
de replicar os outros 8 painéis (cada um vira spec/plano próprio depois).

## Decisões fechadas

1. **Reuso (decisão C — duplicar-e-adaptar):** portar `lib/`, `components/`, `shell/`, `hooks/` e
   `modules/Overview.tsx` do `admin` para o repo `dashboard`, adaptando para single-tenant. Extração
   de pacote compartilhado (`packages/ui`) entre `admin` e `dashboard` = follow-up, quando ambos
   estabilizarem. (Repos são separados; wiring de pacote cross-repo foi deferido na migração.)
2. **Tenant via `VITE_MUNICIPALITY_ID`:** município fixo vindo de env. Sem `ScopePicker`, sem
   `municipalityId: "all"`. Seam limpo: quando o módulo 06 (auth) entrar, troca a fonte do tenant
   (env → token) sem mexer no resto.
3. **i18n PT-BR sem framework:** Intl `pt-BR` + `America/Sao_Paulo` (via `lib/format`) e texto PT-BR
   inline — igual ao `admin`. Locale único; react-i18next seria YAGNI.
4. **Testes com vitest:** introduzir vitest (o `admin` não tem testes) cobrindo **lógica pura**.
   Componentes visuais = verificação rodando contra a API real.

## Arquitetura & estrutura de arquivos

Tudo em `apps/dashboard/src/`. Portado/adaptado do `apps/admin/src/`.

**`lib/`**
- `api.ts` — cliente HTTP. Adaptado: mantém envelope `{ data, as_of }`, `ApiError`,
  `credentials: "include"`, base `VITE_ADMIN_API_BASE` (default `/admin/api`). **Remove** o header
  dinâmico de operador (`setMunicipalityHeader`/`X-Municipality-Id`) — dashboard não troca cidade.
- `tenant.ts` — **novo.** Resolve `VITE_MUNICIPALITY_ID` (string). Lança erro claro se ausente em
  runtime (config inválida). Único ponto que conhece a origem do tenant.
- `scope.ts` — adaptado. Mantém `period` (`today`/`7d`/`30d`) + `PERIOD_OPTIONS`. `municipalityId`
  passa a ser **constante** (de `tenant.ts`), sem `setMunicipality`. `scopeParams(scope)` devolve
  `{ period, municipality_id }`.
- `format.ts` — portado as-is (Intl pt-BR, timezone, números/datas).
- `types.ts` — subconjunto de tipos que o Overview e seus hooks usam.

**`components/`** (subconjunto que o Overview precisa): `AsOfStamp`, `StatTile`, `Panel`,
`PageHeader`, `Sparkline`, `Skeleton`, `EmptyState`, `ErrorState`, `KpiGrid`, `SourceBadge`. Portados
as-is (são apresentacionais, sem acoplamento a tenant).

**`shell/`**
- `AppHeader.tsx` — adaptado: nav por módulo + seletor de **período**. **Sem** `ScopePicker`.
  Mostra o nome do município (do `as_of`/scope block) como rótulo estático.

**`hooks/`** (um por endpoint, portados, usando `scopeParams` com tenant fixo): `useOverview`,
`useIngestion`, `useConversations`, `useQueues`, `useHealth`, `useEvents`.

**`modules/`**
- `Overview.tsx` — portado: 5 KPIs + 4 painéis-resumo (Ingestão&triagem, Filas, Saúde, Eventos),
  6 hooks paralelos, estados loading/erro/vazio, `AsOfStamp`.
- `Placeholder.tsx` — **novo.** Placeholder honesto reutilizável para os 8 módulos ainda não ligados.

**`App.tsx`** — nav listando **os 9 módulos**; Overview ligado, os outros 8 renderizam `Placeholder`.
Módulo ativo via switch (espelha o `admin`, sem react-router nesta fatia).
**`main.tsx`** — `QueryClient` + `ScopeContext.Provider` (período em estado local, tenant do env) + render.
**`theme/`** — alinhar `tokens.ts`/`global.css` com os do `admin` (consistência visual).

## Data flow

1. `main.tsx` monta `ScopeContext` com `period` (estado, default `7d`) e `municipalityId` (de `tenant.ts`).
2. Cada hook usa React Query com key `[<recurso>, period, municipalityId]` e chama
   `apiFetch<T>("/<recurso>", scopeParams(scope))`.
3. Envelope `{ data, as_of }`: `data` alimenta o painel; `as_of` alimenta o `AsOfStamp`.
4. `municipality_id` (do env) vai em todo request como query param. RLS no backend escopa; sem auth
   (módulo 06 deferido), em dev confia-se no param.

## Tratamento de erro

- `ApiError(status, body, message)` no cliente. Cada painel: `Skeleton` (loading) → `ErrorState`
  (erro, com status) → `EmptyState` (vazio) → dados. Nunca renderizar `0` como dado real.
- `tenant.ts` lança se `VITE_MUNICIPALITY_ID` ausente — falha cedo e clara, não silenciosa.

## Testes (vitest)

Setup: `vitest` + `@testing-library/react` + `jsdom` no `package.json`; `vitest.config.ts`; script
`"test": "vitest run"`. Cobertura de **lógica pura** (não UI visual):
- `lib/scope.test.ts` — `scopeParams` devolve `period` + `municipality_id` do tenant fixo; muda com `period`.
- `lib/tenant.test.ts` — resolve `VITE_MUNICIPALITY_ID`; lança quando ausente.
- `lib/api.test.ts` — parse de envelope `{data, as_of}`; `ApiError` em status != 2xx (fetch mockado).
- `lib/format.test.ts` — formatação pt-BR de número/data/timestamp (casos de borda: zero, null).

## Critério de aceite

1. `npm run dev` no repo `dashboard` sobe o app; nav mostra os 9 módulos.
2. Com `VITE_MUNICIPALITY_ID` setado e o `api` de pé, o **Overview** renderiza KPIs + 4 painéis-resumo
   reais (zerados em dev limpo), cada bloco com `AsOfStamp`, estados loading/erro/vazio funcionando.
3. Os outros 8 módulos mostram `Placeholder` honesto.
4. Toda chamada manda `municipality_id` (do env) — nenhum dado fora do tenant (invariante do módulo).
5. `npm run test` passa (vitest, lógica pura).
6. Sem `ScopePicker`/seletor de cidade em lugar nenhum.

## Out-of-scope (fatias/trabalhos próprios)

- Os outros 8 painéis (Ingestão, Conversas, Consentimento, Triagens, Classificação, Protocolos,
  Filas, Eventos) — cada um spec/plano próprio reusando este padrão.
- Auth/login (F-05.2 — módulo 06); o tenant vem de env nesta fatia.
- Jobs de projeção backend (F-05.13 `UpdateDashboardJob`, F-05.14 reconciliação).
- Extração de pacote UI compartilhado entre `admin` e `dashboard`.
- react-router / i18n framework.
- Testes visuais de componente / e2e.
