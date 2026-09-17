# F-04.5 — Renderização do relatório no wpda — Design

**Data:** 2026-06-26
**Feature:** F-04.5 (mod-04, board #1). Toca também a `api` (ajuste do link do cidadão).
**Repos:** `rotasaude/wpda` (local `apps/wpda`) + `rotasaude/api` (local `apps/api`).

## Contexto

O `wpda` é o app **do cidadão**: público, acessado por **token assinado** (sem login). Hoje é um
stub (App.tsx só pinga `/up`; o texto "autoria de protocolo" é rótulo legado errado). O backend do
relatório já está pronto (F-04.1–04.4, F-06.16):
- `GET /r/:token` (`ReportsController#show`, público, cross-tenant, BYPASSRLS) verifica o HMAC e
  devolve JSON congelado `{ tier, priority, summary, completed_at, expires_at }`; **404** se o token
  for inválido/expirado. Nunca recalcula (lê do `report_snapshots`).
- Token assinado: `ReportSnapshot.mint_token` + HMAC; `find_by_signed_token` com `secure_compare` +
  expiry.

O que falta (F-04.5): a **página** que o cidadão vê. Hoje `snapshot.url` (link enviado por WhatsApp
em F-04.7) aponta pro endpoint **JSON** da api — o cidadão veria JSON cru. Esta fatia entrega a tela
no wpda + aponta o link pra ela.

## Decisões fechadas

1. **Token via query param:** link `/<wpda-base>/?token=<token>`; o wpda lê `?token`, busca
   `GET /r/:token`, renderiza. Sem react-router (YAGNI).
2. **Backend muda (nesta fatia):** `ReportSnapshot#url` passa a gerar a URL do **wpda**, via novo env
   `WPDA_PUBLIC_BASE`. O endpoint `/r/:token` JSON fica intacto.
3. **vitest introduzido no wpda** (não tem testes), espelhando o setup do dashboard.
4. **i18n:** PT-BR inline + Intl `America/Sao_Paulo`. Sem framework. Mobile-first, acessível.

## Mudança no backend (`apps/api`)

`app/models/report_snapshot.rb#url` — trocar:
```ruby
Rails.application.routes.url_helpers.report_url(token: token)
```
por uma URL apontando pro wpda:
```ruby
base = ENV.fetch("WPDA_PUBLIC_BASE", "http://localhost:5176/wpda")
"#{base.chomp('/')}/?token=#{token}"
```
- Novo env **`WPDA_PUBLIC_BASE`** (dev default `http://localhost:5176/wpda`; documentar em
  `.env.example` da api). Em prod = URL pública do wpda.
- Spec: `spec/models/report_snapshot_spec.rb` — `url` inclui `WPDA_PUBLIC_BASE` e o `token` como
  query param. (Não há spec de ReportSnapshot hoje; criar com este caso.)

## Estrutura no wpda (`apps/wpda/src`) — substitui o stub

- `vite.config.ts`: adicionar proxy **`/r`** → `${VITE_API_PROXY_TARGET}` (api). (Já proxa `/up`.)
- `lib/report.ts`:
  - tipo `Report = { tier: string|null; priority: string|null; summary: string|null; completed_at: string|null; expires_at: string|null }`.
  - `tokenFromUrl(search?: string): string | null` — lê `?token` (default `window.location.search`).
  - `fetchReport(token): Promise<Report | null>` — `GET /r/<token>`; 200 → `Report`; **404 → null**;
    outros status → lança `Error` (estado de erro genérico).
- `lib/format.ts`: `fmtDateTime(iso): string` (Intl pt-BR, `America/Sao_Paulo`; `"—"` se vazio).
- `modules/Report.tsx`: dado um `token`, usa `fetchReport` e renderiza um dos estados:
  - **loading** (skeleton/spinner simples),
  - **relatório** (conteúdo, ver §Renderização),
  - **inválido/expirado** (404 → mensagem amigável),
  - **erro** (falha de rede/5xx → "Não foi possível carregar. Tente novamente.").
- `App.tsx`: `tokenFromUrl()` → se null, tela "Link inválido — use o link enviado por WhatsApp";
  se presente, `<Report token={token} />`.
- `main.tsx`: render do `App` (sem QueryClient/auth — fetch direto, página única).
- `theme/`: reusa `tokens.ts`/`global.css` do scaffold; layout centralizado, mobile-first.

## Renderização (citizen-friendly)

Campos do JSON, sem jargão clínico (o backend já filtra):
- `summary` — corpo principal (o texto pro cidadão).
- `tier` — badge/rótulo (ex.: cor por severidade; mapear valor → rótulo PT-BR amigável).
- `priority` — rótulo secundário.
- `completed_at` — "Realizado em \<data formatada\>".
- `expires_at` — nota discreta "Válido até \<data\>".
Cabeçalho "Rota Saúde" + nota de privacidade curta. Sem links de navegação (página única pública).

## Data flow

1. Cidadão abre `/<wpda>/?token=<token>` (link do WhatsApp, agora apontando pro wpda).
2. `App` lê `?token`; `Report` chama `fetchReport(token)` → proxy `/r/<token>` → api.
3. 200 → renderiza; 404 → "inválido/expirado"; erro de rede → estado de erro.

## Tratamento de erro

- Sem `?token`: mensagem amigável, não quebra.
- 404 (token inválido/expirado): tela dedicada "Este link é inválido ou expirou." (não vaza se é
  inválido vs expirado — o backend devolve 404 pros dois).
- 5xx/rede: estado de erro genérico com opção de recarregar.

## Testes

**Backend (`apps/api`):** RSpec `spec/models/report_snapshot_spec.rb` — `url` usa `WPDA_PUBLIC_BASE`
e inclui `?token=<token>`.

**wpda (vitest, novo):** setup `vitest` + `jsdom` no `package.json` + `vitest.config.ts`.
- `lib/report.test.ts` — `tokenFromUrl` extrai `?token` (e null quando ausente); `fetchReport`
  devolve `Report` em 200, `null` em 404, lança em 500 (fetch mockado).
- `lib/format.test.ts` — `fmtDateTime` pt-BR + `"—"` para null.

## Critério de aceite

1. `apps/wpda` rodando (container host 5176 ou `npm run dev`), abrir `/wpda/?token=<token-válido>` →
   relatório renderizado (summary, tier, priority, data), com a api de pé.
2. Token inválido/expirado → tela "inválido ou expirou" (não JSON, não erro cru).
3. Sem `?token` → mensagem amigável.
4. `snapshot.url` (backend) gera `WPDA_PUBLIC_BASE/?token=<token>`; spec passa.
5. `npm run test` (wpda) e o spec do backend verdes; typecheck/build do wpda verdes.
6. Sem login, sem react-router, sem jargão clínico exposto.

## Out-of-scope

- F-03.17 (resultado da triagem renderizado) — conteúdo/feature diferente, fatia própria.
- Autoria de protocolo, qualquer outra tela do wpda.
- Mudar o endpoint `/r/:token` (continua JSON).
- Internacionalização multi-locale, PWA/offline, analytics.
