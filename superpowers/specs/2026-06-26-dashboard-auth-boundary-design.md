# Dashboard — Fronteira de auth (módulo 06, fatia) — Design

**Data:** 2026-06-26
**Módulo:** 06 — Identidade/Acesso (`docs/modulos/06--identidade-acesso.md`). Fatia da superfície `dashboard`: "Login (cidade)" + tenant da membership.
**Repo alvo:** `rotasaude/dashboard` (local: `apps/dashboard`).

## Contexto

A Fatia 1 do dashboard ([[dashboard-fatia1-overview]]) está construída mas o happy-path está bloqueado:
os endpoints `/admin/api/*` exigem sessão (`Current.user`), e o dashboard não tem login (tenant vinha
de `VITE_MUNICIPALITY_ID`). Logado, o Overview e os próximos painéis renderizam dados reais.

O backend de sessão **já existe** (usado pelo `admin`): `POST /session` (login), `GET /session`
(sessão atual), `DELETE /session` (logout); cookie HttpOnly (ADR-0022). O `Admin::Api::BaseController`
deriva o tenant da **membership ativa** do usuário para não-operadores (`first_member_municipality`,
`base_controller.rb:70-78`) — o param `municipality_id` só vale para operador. Logo, **logado como
usuário municipal, o dashboard recebe dados tenant-scoped automaticamente**.

Esta fatia é só **frontend**: portar/adaptar a auth do `admin` (`lib/auth.tsx`, funções de sessão de
`lib/api.ts`) e trocar a fonte do tenant (env → sessão). Escopo confirmado = fronteira de auth do
dashboard; o resto do módulo 06 (gestão de usuários da cidade, convites, step-up MFA de publicação,
provisionamento) = specs próprios.

## Decisões fechadas

1. **Reuso (decisão C):** adaptar `lib/auth.tsx` + as funções de sessão do `api.ts` do `admin`.
2. **Sem MFA no login:** usuários municipais (operator=false) não fazem MFA no login (MFA é do
   operador; step-up de publicação é F-06.5, fora desta fatia read-only). Estado `mfa_required`
   removido do AuthProvider do dashboard.
3. **Login enxuto novo:** escrever `modules/Login.tsx` focado (email+senha, ~60 linhas), não portar o
   `Login.tsx` de 254 linhas do admin (tem MFA/gov.br/operador).
4. **Aposentar `VITE_MUNICIPALITY_ID`:** remover `lib/tenant.ts`; o tenant vem da sessão.

## Arquitetura & estrutura de arquivos (`apps/dashboard/src/`)

**`lib/api.ts` (modificar):** adicionar, abaixo do que já existe:
- Tipos `Membership` (`{ municipality_id, municipality_name, municipality_uf, role }`) e `SessionUser`
  (`{ id, email_address, operator, memberships: Membership[] }` — subconjunto usado; sem campos de MFA).
- `login(email_address, password): Promise<SessionUser>` → `POST` em `VITE_SESSION_BASE` (default
  `/session`), `credentials: include`. (Sem o ramo `MfaRequired` — fora do escopo.)
- `fetchCurrentSession(): Promise<SessionUser | null>` → `GET /session`; `null` em 401.
- `logout(): Promise<void>` → `DELETE /session`.
- Um helper `sessionFetch` interno espelhando o `jsonFetch` (POST/DELETE com body). O `adminFetch`
  existente fica intacto.

**`lib/auth.tsx` (novo):** `AuthProvider` + `useAuth()`. Estados: `loading` (boot, GET /session),
`anonymous`, `authenticated` (`{ user: SessionUser }`). Métodos: `login(email, password)`,
`logout()`, `reload()`. Deriva `municipalityId: string | null` de `user.memberships[0]?.municipality_id`
(primeira membership municipal). **Sem** `mfa_required`, **sem** `setActiveMunicipality`/switching.

**`lib/scope.ts` (modificar):** `municipalityId` deixa de vir de `lib/tenant`. O `App` passa o
`municipalityId` da sessão para o `ScopeContext`. `Scope.municipalityId` muda para `string | null`.
`scopeParams` passa a **omitir** `municipality_id` quando null (só inclui `period` nesse caso) —
mantém o tipo `Record<string,string>` válido. O `municipality_id`, quando presente, segue inofensivo
(backend ignora p/ municipal e usa a membership). As React Query keys continuam incluindo
`municipalityId` (tenant-aware). Hooks e Overview inalterados (a assinatura de `scopeParams` não muda).

**`lib/tenant.ts` (remover)** + seu teste. (Substituído pela sessão.)

**`modules/Login.tsx` (novo):** form email+senha, botão "Entrar", estado de erro (credencial inválida
via `ApiError` 401/422). Chama `useAuth().login`. PT-BR. Layout enxuto centralizado.

**`main.tsx` (modificar):** envolver com `<AuthProvider>`. `AppRoot`: `loading`→`Splash`,
`anonymous`→`<Login/>`, `authenticated`→`<QueryClientProvider><App/></QueryClientProvider>`. No
`QueryClient`, `queryCache.onError`: se `ApiError` status 401 → `auth.reload()` (sessão expirou).

**`App.tsx` (modificar):** ler `municipalityId` de `useAuth()` (não mais de `lib/tenant`) e passar ao
`ScopeContext`. Resto inalterado.

**`shell/AppHeader.tsx` (modificar):** adicionar à direita o e-mail do usuário (de `useAuth`) + botão
discreto "Sair" (`auth.logout`).

## Data flow

1. Boot: `AuthProvider` faz `GET /session`. Sem cookie → `anonymous` → `<Login>`.
2. Login: `POST /session` → cookie setado → `SessionUser` → `authenticated`. `municipalityId` =
   membership[0].
3. App monta com `ScopeContext` (período + `municipalityId` da sessão). Hooks chamam `/admin/api/*`
   com cookie → backend deriva tenant da membership → **dados reais**.
4. 401 em qualquer chamada (sessão expirou) → `onError` → `auth.reload()` → `anonymous` → `<Login>`.

## Tratamento de erro

- Login com credencial errada: `login` lança `ApiError` (401/422) → `Login` mostra "E-mail ou senha
  inválidos".
- `fetchCurrentSession` engole 401 (→ `null` → anonymous), não propaga.
- Sessão expira em uso → volta pro login (sem dado clínico exposto).

## Testes (vitest)

- `lib/auth.test.tsx` — renderiza `AuthProvider`, mockando as funções de `api`:
  - `login` ok → estado `authenticated`, `municipalityId` = membership[0].municipality_id.
  - `logout` → `anonymous`, `municipalityId` null.
  - boot com `fetchCurrentSession` → null → `anonymous`.
  (Usa `@testing-library/react` — `renderHook`/`act` — já instalado na Fatia 1.)
- `lib/api.test.ts` (acrescentar) — `login` faz POST e devolve `SessionUser`; `fetchCurrentSession`
  devolve `null` em 401 (fetch mockado).

## Critério de aceite

1. Sem sessão, o dashboard mostra `<Login>` (não o app).
2. Login com `admin@curitiba.demo` (usuário municipal de dev) → app monta, **Overview renderiza KPIs
   reais** da cidade do usuário (Curitiba Demo), com `AsOfStamp`.
3. `municipality_id` não precisa ser informado pelo usuário — vem da sessão; nenhum dado de outra
   cidade aparece.
4. Logout → volta pro `<Login>`. Sessão expirada (401) em uso → volta pro `<Login>`.
5. `VITE_MUNICIPALITY_ID` e `lib/tenant.ts` removidos; `npm run test`, `typecheck`, `build` verdes.
6. Sem MFA, sem ScopePicker, sem gestão de usuários/convites (fora do escopo).

## Out-of-scope (specs próprios)

- Gestão de usuários da cidade, convites, desativação (F-06.9/10 superfície dashboard).
- Step-up MFA na publicação (F-06.5).
- MFA de login / recuperação de senha / gov.br.
- Provisionamento de município, features cross-tenant do admin.
- Os outros 8 painéis do dashboard (specs próprios; reusam esta auth).
