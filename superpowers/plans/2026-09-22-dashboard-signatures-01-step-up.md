# Dashboard — assinaturas, fatia 1: step-up e Segurança da conta

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** o usuário da cidade cadastra o próprio autenticador (TOTP) no dashboard, e o dashboard ganha o componente `SensitiveAction`, que confirma qualquer ação sensível com step-up consciente da janela de 5 minutos. As fatias 2 (Equipe) e 3 (Protocolos) usam esse componente.

**Architecture:**
- **API:** ganha uma regra: trocar o autenticador de uma conta já cadastrada exige step-up.
- **Dashboard:**
  - uma função pura calcula a janela de step-up a partir do `/session`;
  - um hook junta essa janela ao `POST /mfa/step_up`;
  - um tradutor de erros, também puro, converte as respostas da API;
  - `SensitiveAction` concentra código, repetição e mensagens;
  - a página "Segurança da conta" faz o cadastro e a troca.
- A navegação continua por estado (`ModuleId`), como hoje.

**Tech Stack:** Rails 8.1 (apps/api); Vite + React 18 + React Query 5 + Vitest + @testing-library/react + `qrcode` (apps/dashboard).

**Spec:** `docs/superpowers/specs/2026-09-22-dashboard-protocol-signatures-design.md`. Esta fatia implementa §2 (decisões 2, 3, 7 e 8), §4.2, §5.1 (as unidades de step-up), §5.2 "Segurança da conta", §6 e §9 fatia 1.

## Global Constraints

- Commits: Conventional Commits **em inglês**, tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Git: use `/opt/homebrew/bin/git`. `apps/api` e `apps/dashboard` são repositórios separados; a raiz do monorepo não é git. Trabalho na branch `feat/dashboard-step-up` em cada um. Nunca em `main`, nunca push.
- Suíte completa do api: na raiz do monorepo, `docker compose stop worker`, depois `docker compose exec -T api bundle exec rspec`, depois `docker compose start worker` (sempre religar, mesmo em falha). Acima de ~3 min é regressão.
- Specs de request exigem `type: :request`, e o host de cidade vem de `spec/support/city_request_auth.rb` (`sign_in_as(user)`).
- Janela de step-up: **5 minutos** (`MfaStepUp::STEP_UP_WINDOW`). A regra da API é `mfa_verified_at > 5.minutes.ago`, então no limite exato a janela está fechada.
- Toda escrita do dashboard vai com `Content-Type: application/json`. A API recusa com 415 uma escrita por cookie sem JSON. Em `jsonFetch`, o header só é posto quando há `body`, então um POST sem dados manda `body: "{}"`.
- O código TOTP é limpo a cada tentativa e nunca entra no cache do React Query. A chave, o QR e os códigos de recuperação existem só no estado da página de Segurança, e o QR é gerado localmente (`qrcode`), nunca por serviço externo.
- `POST /mfa/enroll` só roda num clique, nunca num efeito: o StrictMode chamaria duas vezes e trocaria o segredo.
- Mensagens ao usuário em português. Sem dado de cidadão.
- Nunca rode `start.sh`. Nunca imprima segredos (chave TOTP, códigos de recuperação) em logs ou relatórios.

## File Structure

**apps/api**
- Modify: `app/controllers/mfa_controller.rb`, onde o `enroll` exige step-up quando a conta já tem TOTP.
- Create: `spec/requests/mfa_enroll_step_up_spec.rb`.

**apps/dashboard**
- Modify: `vite.config.ts`, com o proxy de `/protocols` e `/mfa`.
- Modify: `src/lib/api.ts`, com `SessionUser.mfa_enrolled`/`mfa_verified_at`, `enrollMfa`, `confirmMfa`, `stepUpMfa` e `errorCode`.
- Modify: `src/lib/api.test.ts`.
- Create: `src/lib/stepUp.ts` + `src/lib/stepUp.test.ts`, com a janela, pura.
- Create: `src/lib/actionErrors.ts` + `src/lib/actionErrors.test.ts`, com o tradutor de erros, puro.
- Create: `src/lib/useStepUp.ts`, o hook.
- Create: `src/components/formStyles.ts`, com os estilos de input e botão, extraídos do padrão de `Login.tsx`.
- Create: `src/components/SensitiveAction.tsx` + `src/components/SensitiveAction.test.tsx`.
- Create: `src/modules/Security.tsx` + `src/modules/Security.test.tsx`.
- Modify: `src/shell/modules.ts` e `src/App.tsx`, com o módulo `security`.
- Modify: `package.json` / `package-lock.json`, com `qrcode` e `@types/qrcode`.

---

### Task 1: API — trocar o autenticador exige step-up

**Files:**
- Modify: `apps/api/app/controllers/mfa_controller.rb`
- Test: `apps/api/spec/requests/mfa_enroll_step_up_spec.rb`

**Interfaces:**
- Consumes:
  - `MfaStepUp#reauthenticated_recently?` e `#require_step_up!`, este último respondendo `401 { error: "mfa_required" }`;
  - `User#mfa_enrolled?`, que é `otp_enabled? && otp_secret.present?`;
  - `sign_in_as(user)` (`spec/support/city_request_auth.rb`), que devolve a `Session`.
- Produces: `POST /mfa/enroll` numa conta com `mfa_enrolled?` e sem janela aberta → `401 { "error": "mfa_required" }`, com o segredo e `otp_enabled` intactos. Sem TOTP cadastrado, ou com a janela aberta, o comportamento de hoje: `200 { otpauth_uri, recovery_codes }`.

- [ ] **Step 1: Branch**

```bash
cd apps/api && /opt/homebrew/bin/git checkout -b feat/dashboard-step-up
```

- [ ] **Step 2: Escrever a spec que falha**

`spec/requests/mfa_enroll_step_up_spec.rb`:

```ruby
require "rails_helper"

# Spec do dashboard §4.2: trocar o autenticador de uma conta que JÁ tem TOTP
# exige step-up. Sem isso, quem tivesse só a senha (ou uma sessão roubada)
# cadastraria o próprio autenticador e passaria a fazer step-up — o segundo
# fator viraria enfeite. O primeiro cadastro continua só com a sessão: não há
# fator anterior a pedir.
RSpec.describe "MFA enroll with step-up", type: :request do
  def json = JSON.parse(response.body)

  let!(:user) { User.create!(email_address: "eve-#{SecureRandom.hex(3)}@example.org", password: "secret123") }

  it "primeiro cadastro não pede step-up" do
    sign_in_as(user)

    post "/mfa/enroll", as: :json

    expect(response).to have_http_status(:ok)
    expect(json["otpauth_uri"]).to start_with("otpauth://totp/")
  end

  it "cadastro começado e nunca confirmado recomeça sem step-up" do
    Mfa::Enroll.call(user) # otp_enabled continua false
    sign_in_as(user)

    post "/mfa/enroll", as: :json

    expect(response).to have_http_status(:ok)
  end

  context "conta com autenticador ativo" do
    before do
      Mfa::Enroll.call(user)
      user.update!(otp_enabled: true)
    end

    it "sem janela de step-up recusa com mfa_required e não mexe no segredo" do
      secret = user.reload.otp_secret
      sign_in_as(user)

      post "/mfa/enroll", as: :json

      expect(response).to have_http_status(:unauthorized)
      expect(json).to eq("error" => "mfa_required")
      expect(user.reload.otp_secret).to eq(secret)
      expect(user.otp_enabled).to be(true)
    end

    it "com a janela vencida (6 min) também recusa" do
      sign_in_as(user).update!(mfa_verified_at: 6.minutes.ago)

      post "/mfa/enroll", as: :json

      expect(response).to have_http_status(:unauthorized)
      expect(json).to eq("error" => "mfa_required")
    end

    it "com a janela aberta troca o autenticador" do
      secret = user.reload.otp_secret
      sign_in_as(user).update!(mfa_verified_at: 1.minute.ago)

      post "/mfa/enroll", as: :json

      expect(response).to have_http_status(:ok)
      expect(user.reload.otp_secret).not_to eq(secret)
    end
  end
end
```

- [ ] **Step 3: Rodar e ver falhar**

Da raiz do monorepo:
```bash
docker compose exec -T api bundle exec rspec spec/requests/mfa_enroll_step_up_spec.rb
```
Esperado: os dois exemplos "recusa" FALHAM (recebem 200). Os três outros passam.

- [ ] **Step 4: Implementar**

Em `app/controllers/mfa_controller.rb`, acrescente `include MfaStepUp` logo abaixo de `include Authentication` e a guarda no topo de `enroll`:

```ruby
  def enroll
    # Spec do dashboard §4.2: trocar o autenticador de uma conta que JÁ tem
    # TOTP exige step-up — senão a senha sozinha (ou uma sessão roubada)
    # substituiria o segundo fator. O primeiro cadastro não tem fator anterior
    # a pedir e segue só com a sessão.
    return require_step_up! if Current.user.mfa_enrolled? && !reauthenticated_recently?

    payload = Mfa::Enroll.call(Current.user)
```

O resto do método fica como está.

- [ ] **Step 5: Rodar e ver passar**

```bash
docker compose exec -T api bundle exec rspec spec/requests/mfa_enroll_step_up_spec.rb spec/controllers/mfa_controller_spec.rb spec/requests/operator_city_session_spec.rb
```
Esperado: PASS.

- [ ] **Step 6: Suíte completa** (worker parado; ver Global Constraints). Esperado: 0 falhas.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/mfa_controller.rb spec/requests/mfa_enroll_step_up_spec.rb
/opt/homebrew/bin/git commit -m "fix: require step-up to replace an enrolled authenticator" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: dashboard — proxy, cliente de MFA, janela e erros (puros)

**Files:**
- Modify: `apps/dashboard/vite.config.ts`, `apps/dashboard/src/lib/api.ts`, `apps/dashboard/src/lib/api.test.ts`
- Create: `apps/dashboard/src/lib/stepUp.ts`, `apps/dashboard/src/lib/stepUp.test.ts`
- Create: `apps/dashboard/src/lib/actionErrors.ts`, `apps/dashboard/src/lib/actionErrors.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `ApiError` (`status`, `body`) de `src/lib/api.ts`.
- Produces:
  ```ts
  // api.ts
  export interface SessionUser { /* + */ mfa_enrolled: boolean; mfa_verified_at: string | null; }
  export interface MfaEnrollment { otpauth_uri: string; recovery_codes: string[]; }
  export function enrollMfa(): Promise<MfaEnrollment>;          // POST /mfa/enroll
  export function confirmMfa(code: string): Promise<void>;      // POST /mfa/confirm
  export function stepUpMfa(code: string): Promise<void>;       // POST /mfa/step_up
  export function errorCode(err: unknown): string | null;       // body.error de um ApiError
  // stepUp.ts
  export const STEP_UP_WINDOW_MS = 300_000;
  export interface StepUpWindow { enrolled: boolean; open: boolean; remainingMs: number; }
  export function stepUpWindow(
    session: { mfa_enrolled: boolean; mfa_verified_at: string | null } | null, now: number
  ): StepUpWindow;
  // actionErrors.ts
  export type ActionError =
    | { kind: "mfa_required" }
    | { kind: "invalid_code"; message: string }
    | { kind: "forbidden"; message: string }
    | { kind: "rejected"; message: string }
    | { kind: "session_expired"; message: string }
    | { kind: "failed"; message: string };
  export function describeActionError(err: unknown): ActionError;
  ```

- [ ] **Step 1: Branch**

```bash
cd apps/dashboard && /opt/homebrew/bin/git checkout -b feat/dashboard-step-up
```

- [ ] **Step 2: Proxy**

Em `vite.config.ts`, acrescente as duas linhas ao `proxy` e ao comentário de topo, que lista as rotas:

```ts
      "/protocols": proxy(TARGET),
      "/mfa":       proxy(TARGET)
```
Comentário:
```
//   /protocols  → ciclo de vida com assinaturas (ProtocolLifecycleController, PublicationsController).
//   /mfa        → cadastro de TOTP e step-up (MfaController).
```

- [ ] **Step 3: Testes que falham — janela**

`src/lib/stepUp.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { STEP_UP_WINDOW_MS, stepUpWindow } from "./stepUp";

const NOW = Date.parse("2026-09-22T12:00:00Z");
const at = (msAgo: number) => new Date(NOW - msAgo).toISOString();

describe("stepUpWindow", () => {
  it("sem sessão ou sem TOTP: não cadastrado, fechada", () => {
    expect(stepUpWindow(null, NOW)).toEqual({ enrolled: false, open: false, remainingMs: 0 });
    expect(stepUpWindow({ mfa_enrolled: false, mfa_verified_at: at(1000) }, NOW))
      .toEqual({ enrolled: false, open: false, remainingMs: 0 });
  });

  it("cadastrado sem verificação: fechada", () => {
    expect(stepUpWindow({ mfa_enrolled: true, mfa_verified_at: null }, NOW))
      .toEqual({ enrolled: true, open: false, remainingMs: 0 });
  });

  it("verificado há 1 min: aberta, com 4 min restantes", () => {
    expect(stepUpWindow({ mfa_enrolled: true, mfa_verified_at: at(60_000) }, NOW))
      .toEqual({ enrolled: true, open: true, remainingMs: STEP_UP_WINDOW_MS - 60_000 });
  });

  it("exatamente no limite de 5 min: fechada (a API exige mfa_verified_at > 5.minutes.ago)", () => {
    expect(stepUpWindow({ mfa_enrolled: true, mfa_verified_at: at(STEP_UP_WINDOW_MS) }, NOW).open).toBe(false);
  });

  it("data ilegível: fechada", () => {
    expect(stepUpWindow({ mfa_enrolled: true, mfa_verified_at: "ontem" }, NOW).open).toBe(false);
  });
});
```

- [ ] **Step 4: Testes que falham — erros**

`src/lib/actionErrors.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import { describeActionError } from "./actionErrors";

const apiError = (status: number, body: unknown) => new ApiError(status, body, `${status}`);

describe("describeActionError", () => {
  it("401 mfa_required", () => {
    expect(describeActionError(apiError(401, { error: "mfa_required" }))).toEqual({ kind: "mfa_required" });
  });

  it("422 invalid_code", () => {
    expect(describeActionError(apiError(422, { error: "invalid_code" })))
      .toEqual({ kind: "invalid_code", message: "código inválido" });
  });

  it("403 sem corpo", () => {
    expect(describeActionError(apiError(403, "")))
      .toEqual({ kind: "forbidden", message: "seu papel não permite esta ação" });
  });

  it("422/409 com mensagem do domínio: a mensagem da API, verbatim", () => {
    expect(describeActionError(apiError(422, { error: "insufficient_signatures", message: "falta 1 assinatura" })))
      .toEqual({ kind: "rejected", message: "falta 1 assinatura" });
    expect(describeActionError(apiError(409, { error: "conflict", message: "versão mudou" })))
      .toEqual({ kind: "rejected", message: "versão mudou" });
  });

  it("422 sem mensagem: frase genérica, nunca o código cru", () => {
    expect(describeActionError(apiError(422, { error: "invalid_state" })))
      .toEqual({ kind: "rejected", message: "a API recusou a ação" });
  });

  it("401 que não é mfa_required: sessão expirada", () => {
    expect(describeActionError(apiError(401, "")))
      .toEqual({ kind: "session_expired", message: "sessão expirada — entre de novo" });
  });

  it("5xx, rede e qualquer outra coisa: genérico", () => {
    const generic = { kind: "failed", message: "não foi possível concluir — tente de novo" };
    expect(describeActionError(apiError(500, ""))).toEqual(generic);
    expect(describeActionError(new TypeError("Failed to fetch"))).toEqual(generic);
    expect(describeActionError("x")).toEqual(generic);
  });
});
```

- [ ] **Step 5: Testes que falham — cliente de MFA**

Acrescente a `src/lib/api.test.ts` (usa o `mockFetch` que o arquivo já tem; importe `enrollMfa, confirmMfa, stepUpMfa, errorCode, ApiError`):

```ts
describe("mfa", () => {
  function lastCall() {
    const fetchMock = globalThis.fetch as unknown as ReturnType<typeof vi.fn>;
    const [ url, init ] = fetchMock.mock.calls.at(-1) as [ string, RequestInit ];
    return { url, init, headers: init.headers as Record<string, string> };
  }

  it("enrollMfa: POST /mfa/enroll com JSON (a API recusa escrita por cookie sem JSON)", async () => {
    mockFetch(200, { otpauth_uri: "otpauth://totp/x?secret=ABC", recovery_codes: [ "a1" ] });
    const out = await enrollMfa();
    const { url, init, headers } = lastCall();
    expect(url).toBe("/mfa/enroll");
    expect(init.method).toBe("POST");
    expect(headers["Content-Type"]).toBe("application/json");
    expect(out.recovery_codes).toEqual([ "a1" ]);
  });

  it("confirmMfa e stepUpMfa mandam { code }", async () => {
    mockFetch(200, { ok: true });
    await confirmMfa("123456");
    expect(lastCall().url).toBe("/mfa/confirm");
    expect(JSON.parse(lastCall().init.body as string)).toEqual({ code: "123456" });

    mockFetch(200, { ok: true });
    await stepUpMfa("654321");
    expect(lastCall().url).toBe("/mfa/step_up");
    expect(JSON.parse(lastCall().init.body as string)).toEqual({ code: "654321" });
  });

  it("errorCode lê body.error de um ApiError", () => {
    expect(errorCode(new ApiError(401, { error: "mfa_required" }, "x"))).toBe("mfa_required");
    expect(errorCode(new ApiError(403, "", "x"))).toBeNull();
    expect(errorCode(new Error("x"))).toBeNull();
  });
});
```

- [ ] **Step 6: Rodar e ver falhar**

Run: `npx vitest run src/lib/stepUp.test.ts src/lib/actionErrors.test.ts src/lib/api.test.ts`
Esperado: FAIL, módulos e exports inexistentes.

- [ ] **Step 7: Implementar**

`src/lib/stepUp.ts`:

```ts
// Janela de step-up (spec do dashboard §2.3, ADR 0011): a API aceita uma ação
// sensível sem pedir código se `mfa_verified_at` for mais recente que 5 min
// (MfaStepUp::STEP_UP_WINDOW, regra `ts > 5.minutes.ago`). Isto é só uma
// PREVISÃO a partir do /session — relógios diferentes ou uma janela que fecha
// entre a leitura e o clique viram `mfa_required`, que o SensitiveAction trata.
export const STEP_UP_WINDOW_MS = 5 * 60 * 1000;

export interface StepUpWindow { enrolled: boolean; open: boolean; remainingMs: number; }

const CLOSED = { open: false, remainingMs: 0 } as const;

export function stepUpWindow(
  session: { mfa_enrolled: boolean; mfa_verified_at: string | null } | null,
  now: number
): StepUpWindow {
  if (!session?.mfa_enrolled) return { enrolled: false, ...CLOSED };
  const verified = session.mfa_verified_at ? Date.parse(session.mfa_verified_at) : Number.NaN;
  if (Number.isNaN(verified)) return { enrolled: true, ...CLOSED };
  const remainingMs = verified + STEP_UP_WINDOW_MS - now;
  return remainingMs > 0 ? { enrolled: true, open: true, remainingMs } : { enrolled: true, ...CLOSED };
}
```

`src/lib/actionErrors.ts`:

```ts
// Tradução, num lugar só, das respostas da API da cidade a uma ação sensível
// (spec do dashboard §6). Nenhuma tela lê status HTTP: todas perguntam isto.
// A mensagem de domínio (`message` dos commands) sai verbatim; um código cru
// (`invalid_state`, `http_500`) nunca chega ao usuário.
import { ApiError } from "./api";

export type ActionError =
  | { kind: "mfa_required" }
  | { kind: "invalid_code"; message: string }
  | { kind: "forbidden"; message: string }
  | { kind: "rejected"; message: string }
  | { kind: "session_expired"; message: string }
  | { kind: "failed"; message: string };

const GENERIC = "não foi possível concluir — tente de novo";

function bodyField(body: unknown, key: string): string | null {
  if (body && typeof body === "object" && typeof (body as Record<string, unknown>)[key] === "string") {
    return (body as Record<string, string>)[key];
  }
  return null;
}

export function describeActionError(err: unknown): ActionError {
  if (!(err instanceof ApiError)) return { kind: "failed", message: GENERIC };
  const code = bodyField(err.body, "error");

  if (err.status === 401 && code === "mfa_required") return { kind: "mfa_required" };
  if (err.status === 401) return { kind: "session_expired", message: "sessão expirada — entre de novo" };
  if (err.status === 422 && code === "invalid_code") return { kind: "invalid_code", message: "código inválido" };
  if (err.status === 403) return { kind: "forbidden", message: "seu papel não permite esta ação" };
  if (err.status === 422 || err.status === 409) {
    return { kind: "rejected", message: bodyField(err.body, "message") ?? "a API recusou a ação" };
  }
  return { kind: "failed", message: GENERIC };
}
```

Em `src/lib/api.ts`:
- em `SessionUser`, acrescente `mfa_enrolled: boolean;` e `mfa_verified_at: string | null;`, que o `/session` da cidade já devolve (`SessionsController#serialize`);
- depois de `logout`, acrescente:

```ts
const MFA_BASE = import.meta.env.VITE_MFA_BASE || "/mfa";

export interface MfaEnrollment { otpauth_uri: string; recovery_codes: string[]; }

// `body: "{}"` de propósito: jsonFetch só põe Content-Type quando há body, e a
// API recusa (415) escrita por cookie sem application/json.
export async function enrollMfa(): Promise<MfaEnrollment> {
  return jsonFetch<MfaEnrollment>(`${MFA_BASE}/enroll`, { method: "POST", body: "{}" });
}

export async function confirmMfa(code: string): Promise<void> {
  await jsonFetch<unknown>(`${MFA_BASE}/confirm`, { method: "POST", body: JSON.stringify({ code }) });
}

export async function stepUpMfa(code: string): Promise<void> {
  await jsonFetch<unknown>(`${MFA_BASE}/step_up`, { method: "POST", body: JSON.stringify({ code }) });
}

// O `error` do corpo de uma recusa (`mfa_required`, `invalid_code`...), ou null.
export function errorCode(err: unknown): string | null {
  if (!(err instanceof ApiError)) return null;
  const body = err.body;
  if (body && typeof body === "object" && typeof (body as Record<string, unknown>).error === "string") {
    return (body as Record<string, string>).error;
  }
  return null;
}
```

Se o `typecheck` acusar fixtures de `SessionUser` sem os dois campos novos (ex.: `USER` em `src/lib/auth.test.tsx`), acrescente `mfa_enrolled: false, mfa_verified_at: null` nelas, e só nelas.

- [ ] **Step 8: Rodar e ver passar**

```bash
npx vitest run src/lib/stepUp.test.ts src/lib/actionErrors.test.ts src/lib/api.test.ts && npm run typecheck && npm test
```
Esperado: tudo verde.

- [ ] **Step 9: Commit**

```bash
/opt/homebrew/bin/git add vite.config.ts src/lib/api.ts src/lib/api.test.ts src/lib/stepUp.ts src/lib/stepUp.test.ts src/lib/actionErrors.ts src/lib/actionErrors.test.ts
/opt/homebrew/bin/git add -u src   # fixtures ajustadas, se houver
/opt/homebrew/bin/git commit -m "feat: add the MFA client, step-up window and action error mapping" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: dashboard — `useStepUp` e `SensitiveAction`

**Files:**
- Create: `apps/dashboard/src/lib/useStepUp.ts`
- Create: `apps/dashboard/src/components/formStyles.ts`
- Create: `apps/dashboard/src/components/SensitiveAction.tsx`
- Test: `apps/dashboard/src/components/SensitiveAction.test.tsx`

**Interfaces:**
- Consumes:
  - `stepUpWindow` e `STEP_UP_WINDOW_MS` (Task 2);
  - `describeActionError` e `ActionError` (Task 2);
  - `stepUpMfa` (Task 2);
  - `useAuth()` (`src/lib/auth.tsx`), que dá `user: SessionUser | null` e `reload()`.
- Produces:
  ```ts
  // useStepUp.ts
  export function useStepUp(): { window: StepUpWindow; stepUp(code: string): Promise<void> };
  // SensitiveAction.tsx
  export interface SensitiveField { name: string; label: string; required?: boolean; }
  export interface SensitiveActionProps {
    title: string;
    description?: ReactNode;
    requiresStepUp: boolean;
    fields?: SensitiveField[];
    confirmLabel?: string;                                // default "Confirmar"
    run(values: Record<string, string>): Promise<void>;   // a ação; lança ApiError na recusa
    onDone(): void;                                        // sucesso
    onCancel(): void;
    onGoToSecurity?(): void;                               // link "cadastre seu autenticador"
  }
  export function SensitiveAction(props: SensitiveActionProps): JSX.Element;
  ```

**Comportamento (spec §5.1 e §6):**
- **O que o painel mostra:**
  - `requiresStepUp && !window.enrolled` → o texto "Esta ação exige um autenticador cadastrado.", mais o botão "cadastre seu autenticador" (só se houver `onGoToSecurity`) e "Cancelar". Não há confirmar.
  - `requiresStepUp && !window.open` (ou depois de um `mfa_required`) → campo "Código do autenticador" (`autoComplete="one-time-code"`, `inputMode="numeric"`).
  - `requiresStepUp && window.open` → "verificação válida por mais N min", com N = `Math.ceil(remainingMs / 60000)`, e nenhum campo de código.
- **Validação local, antes de qualquer rede:** um campo `required` vazio (após `trim`) mostra "preencha: <label>". Um código vazio, quando o campo aparece, mostra "informe o código do autenticador".
- **Envio:**
  - se o campo de código está visível, `await stepUp(code)`;
  - depois `await run(values)`;
  - o código é limpo **antes** das chamadas (`setCode("")` com o valor guardado numa variável local), então nunca sobra;
  - com o envio em andamento, Confirmar e Cancelar ficam desabilitados.
- **Erros** (via `describeActionError`, tanto de `stepUp` quanto de `run`):
  - `invalid_code` → a mensagem debaixo do campo de código.
  - `mfa_required` vindo de `run` na **primeira** vez → mostra o campo de código com o aviso "sua verificação expirou — informe um novo código". O próximo envio faz step-up e repete `run`. Na **segunda** vez → `run` não é repetido de novo, e o erro geral é "a verificação não foi aceita — recarregue a página e tente de novo".
  - `session_expired` → a mensagem, mais `auth.reload()`, que devolve ao login como hoje.
  - `forbidden`, `rejected` e `failed` → a mensagem no topo do painel. O painel continua aberto.
- **Sucesso:** `onDone()`.

- [ ] **Step 1: Escrever os testes que falham**

`src/components/SensitiveAction.test.tsx`:

```tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import type { ReactNode } from "react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { AuthProvider } from "../lib/auth";
import { SensitiveAction } from "./SensitiveAction";

afterEach(cleanup);

const fetchSession = api.fetchCurrentSession as unknown as ReturnType<typeof vi.fn>;
const stepUpMfa = api.stepUpMfa as unknown as ReturnType<typeof vi.fn>;

function session(overrides: Partial<api.SessionUser> = {}): api.SessionUser {
  return {
    id: "u1", email_address: "ana@cidade.gov.br", operator: false, memberships: [],
    mfa_enrolled: true, mfa_verified_at: null, ...overrides
  };
}
const minutesAgo = (m: number) => new Date(Date.now() - m * 60_000).toISOString();
const apiError = (status: number, body: unknown) => new ApiError(status, body, String(status));

function wrapper({ children }: { children: ReactNode }) {
  return <AuthProvider>{children}</AuthProvider>;
}

function renderAction(props: Partial<Parameters<typeof SensitiveAction>[0]> = {}) {
  const run = props.run ?? vi.fn().mockResolvedValue(undefined);
  const onDone = props.onDone ?? vi.fn();
  render(
    <SensitiveAction title="Assinar publicação" requiresStepUp run={run} onDone={onDone} onCancel={vi.fn()} {...props} />,
    { wrapper }
  );
  return { run, onDone };
}

const codeField = () => screen.queryByLabelText("Código do autenticador") as HTMLInputElement | null;
const confirm = () => screen.getByRole("button", { name: "Confirmar" });

describe("SensitiveAction", () => {
  beforeEach(() => { fetchSession.mockReset(); stepUpMfa.mockReset(); });

  it("janela fechada: pede o código, faz step-up e depois roda a ação", async () => {
    fetchSession.mockResolvedValue(session());
    const { run, onDone } = renderAction();
    await waitFor(() => expect(codeField()).not.toBeNull());

    stepUpMfa.mockResolvedValue(undefined);
    fireEvent.change(codeField()!, { target: { value: "123456" } });
    fireEvent.click(confirm());

    await waitFor(() => expect(onDone).toHaveBeenCalled());
    expect(stepUpMfa).toHaveBeenCalledWith("123456");
    expect(run).toHaveBeenCalledTimes(1);
    expect(stepUpMfa.mock.invocationCallOrder[0]).toBeLessThan(run.mock.invocationCallOrder[0]);
  });

  it("janela aberta: não pede código, mostra o tempo e roda direto", async () => {
    fetchSession.mockResolvedValue(session({ mfa_verified_at: minutesAgo(1) }));
    const { run, onDone } = renderAction();
    expect(await screen.findByText("verificação válida por mais 4 min")).not.toBeNull();
    expect(codeField()).toBeNull();

    fireEvent.click(confirm());

    await waitFor(() => expect(onDone).toHaveBeenCalled());
    expect(stepUpMfa).not.toHaveBeenCalled();
    expect(run).toHaveBeenCalledTimes(1);
  });

  it("sem autenticador: não oferece confirmar e aponta para a Segurança", async () => {
    fetchSession.mockResolvedValue(session({ mfa_enrolled: false }));
    const onGoToSecurity = vi.fn();
    renderAction({ onGoToSecurity });

    expect(await screen.findByText("Esta ação exige um autenticador cadastrado.")).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Confirmar" })).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "cadastre seu autenticador" }));
    expect(onGoToSecurity).toHaveBeenCalled();
  });

  it("ação sem step-up: nunca pede código", async () => {
    fetchSession.mockResolvedValue(session({ mfa_enrolled: false }));
    const { run } = renderAction({ requiresStepUp: false });
    await waitFor(() => expect(fetchSession).toHaveBeenCalled());

    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));

    await waitFor(() => expect(run).toHaveBeenCalledTimes(1));
    expect(codeField()).toBeNull();
  });

  it("código inválido: mensagem no campo, campo limpo, ação não roda", async () => {
    fetchSession.mockResolvedValue(session());
    const { run } = renderAction();
    await waitFor(() => expect(codeField()).not.toBeNull());

    stepUpMfa.mockRejectedValue(apiError(422, { error: "invalid_code" }));
    fireEvent.change(codeField()!, { target: { value: "000000" } });
    fireEvent.click(confirm());

    expect(await screen.findByText("código inválido")).not.toBeNull();
    expect(codeField()!.value).toBe("");
    expect(run).not.toHaveBeenCalled();
  });

  it("mfa_required da ação: pede novo código e repete UMA vez; na segunda, para", async () => {
    fetchSession.mockResolvedValue(session({ mfa_verified_at: minutesAgo(1) }));
    const run = vi.fn().mockRejectedValue(apiError(401, { error: "mfa_required" }));
    renderAction({ run });
    await screen.findByText("verificação válida por mais 4 min");

    fireEvent.click(confirm());
    expect(await screen.findByText("sua verificação expirou — informe um novo código")).not.toBeNull();
    expect(run).toHaveBeenCalledTimes(1);

    stepUpMfa.mockResolvedValue(undefined);
    fireEvent.change(codeField()!, { target: { value: "123456" } });
    fireEvent.click(confirm());

    expect(await screen.findByText("a verificação não foi aceita — recarregue a página e tente de novo")).not.toBeNull();
    expect(run).toHaveBeenCalledTimes(2);
  });

  it("recusa do domínio: mensagem da API no painel, que continua aberto", async () => {
    fetchSession.mockResolvedValue(session({ mfa_verified_at: minutesAgo(1) }));
    const run = vi.fn().mockRejectedValue(apiError(422, { error: "insufficient_signatures", message: "falta 1 assinatura" }));
    const { onDone } = renderAction({ run });
    await screen.findByText("verificação válida por mais 4 min");

    fireEvent.click(confirm());

    expect(await screen.findByText("falta 1 assinatura")).not.toBeNull();
    expect(onDone).not.toHaveBeenCalled();
    expect(confirm()).not.toBeNull();
  });

  it("403: seu papel não permite", async () => {
    fetchSession.mockResolvedValue(session({ mfa_verified_at: minutesAgo(1) }));
    renderAction({ run: vi.fn().mockRejectedValue(apiError(403, "")) });
    await screen.findByText("verificação válida por mais 4 min");

    fireEvent.click(confirm());

    expect(await screen.findByText("seu papel não permite esta ação")).not.toBeNull();
  });

  it("campo obrigatório vazio e código vazio não chegam à rede", async () => {
    fetchSession.mockResolvedValue(session());
    const { run } = renderAction({ fields: [ { name: "reason", label: "Motivo", required: true } ] });
    await waitFor(() => expect(codeField()).not.toBeNull());

    fireEvent.click(confirm());

    expect(await screen.findByText("preencha: Motivo")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "regra errada" } });
    fireEvent.click(confirm());
    expect(await screen.findByText("informe o código do autenticador")).not.toBeNull();
    expect(stepUpMfa).not.toHaveBeenCalled();
    expect(run).not.toHaveBeenCalled();
  });

  it("passa os campos para a ação", async () => {
    fetchSession.mockResolvedValue(session({ mfa_verified_at: minutesAgo(1) }));
    const { run } = renderAction({ fields: [ { name: "reason", label: "Motivo", required: true } ] });
    await screen.findByText("verificação válida por mais 4 min");

    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "  regra errada " } });
    fireEvent.click(confirm());

    await waitFor(() => expect(run).toHaveBeenCalledWith({ reason: "  regra errada " }));
  });
});
```

`run` recebe os valores **como digitados**; a validação é que usa `trim`. Quem aparar ou não é a API, e é o que o command de reversão já faz.

- [ ] **Step 2: Rodar e ver falhar**

Run: `npx vitest run src/components/SensitiveAction.test.tsx`
Esperado: FAIL, o módulo não existe.

- [ ] **Step 3: Implementar `useStepUp`**

`src/lib/useStepUp.ts`:

```ts
import { useCallback, useEffect, useState } from "react";
import { useAuth } from "./auth";
import { stepUpMfa } from "./api";
import { stepUpWindow, type StepUpWindow } from "./stepUp";

// Janela de step-up viva: recalcula a cada 15 s (o "mais N min" anda
// sozinho) e, depois de um step-up aceito, recarrega a sessão — é o
// /session que traz o `mfa_verified_at` novo.
export function useStepUp(): { window: StepUpWindow; stepUp(code: string): Promise<void> } {
  const auth = useAuth();
  const [ now, setNow ] = useState(() => Date.now());

  useEffect(() => {
    const id = setInterval(() => setNow(Date.now()), 15_000);
    return () => clearInterval(id);
  }, []);

  const stepUp = useCallback(async (code: string) => {
    await stepUpMfa(code);
    await auth.reload();
    setNow(Date.now());
  }, [ auth ]);

  return { window: stepUpWindow(auth.user, now), stepUp };
}
```

- [ ] **Step 4: Implementar `formStyles` e `SensitiveAction`**

`src/components/formStyles.ts` (os valores são os mesmos de `inputStyle`/`btnStyle` em `src/modules/Login.tsx`; copie de lá, sem inventar):

```ts
import type { CSSProperties } from "react";

// Estilos de formulário do dashboard, os mesmos de Login.tsx — extraídos
// para as telas de ação sensível não copiarem os literais.
export const inputStyle: CSSProperties = { /* copiar de Login.tsx */ };
export const buttonStyle: CSSProperties = { /* copiar btnStyle de Login.tsx */ };
export const secondaryButtonStyle: CSSProperties = {
  ...buttonStyle, background: "var(--panel)", color: "var(--ink)", border: "1px solid var(--rule2)"
};
```

Não mexa em `Login.tsx` nesta task: só copie os valores.

`src/components/SensitiveAction.tsx`:

```tsx
import { useState, type FormEvent, type ReactNode } from "react";
import { useAuth } from "../lib/auth";
import { useStepUp } from "../lib/useStepUp";
import { describeActionError } from "../lib/actionErrors";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "./formStyles";

// Confirmação de ação sensível (spec do dashboard §5.1/§6, abordagem 1): o
// único lugar que conhece step-up, repetição e tradução de erro. As telas só
// dizem QUAL ação rodar (`run`) e com quais campos.
//
// O código TOTP vive só em estado local e é limpo antes de cada chamada.
// `mfa_required` vindo da ação (a janela fechou entre a leitura e o clique)
// pede um código novo e repete a ação uma vez; a segunda vez para.
export interface SensitiveField { name: string; label: string; required?: boolean; }

export interface SensitiveActionProps {
  title: string;
  description?: ReactNode;
  requiresStepUp: boolean;
  fields?: SensitiveField[];
  confirmLabel?: string;
  run(values: Record<string, string>): Promise<void>;
  onDone(): void;
  onCancel(): void;
  onGoToSecurity?(): void;
}

const EXPIRED = "sua verificação expirou — informe um novo código";
const GAVE_UP = "a verificação não foi aceita — recarregue a página e tente de novo";

export function SensitiveAction({
  title, description, requiresStepUp, fields = [], confirmLabel = "Confirmar",
  run, onDone, onCancel, onGoToSecurity
}: SensitiveActionProps) {
  const auth = useAuth();
  const { window, stepUp } = useStepUp();
  const [ values, setValues ] = useState<Record<string, string>>(() =>
    Object.fromEntries(fields.map((f) => [ f.name, "" ])));
  const [ code, setCode ] = useState("");
  const [ forceCode, setForceCode ] = useState(false);
  const [ retried, setRetried ] = useState(false);
  const [ busy, setBusy ] = useState(false);
  const [ codeError, setCodeError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const [ fieldError, setFieldError ] = useState<string | null>(null);

  const showCode = requiresStepUp && (forceCode || !window.open);

  async function submit(event: FormEvent) {
    event.preventDefault();
    if (busy) return;
    setError(null); setCodeError(null); setFieldError(null);

    const missing = fields.find((f) => f.required && !values[f.name]?.trim());
    if (missing) { setFieldError(`preencha: ${missing.label}`); return; }
    const typed = code;
    if (showCode && !typed.trim()) { setCodeError("informe o código do autenticador"); return; }

    setCode("");
    setBusy(true);
    try {
      if (showCode) await stepUp(typed);
      await run(values);
      onDone();
    } catch (err) {
      const described = describeActionError(err);
      if (described.kind === "mfa_required") {
        if (retried) {
          setError(GAVE_UP);
        } else {
          setRetried(true);
          setForceCode(true);
          setNotice(EXPIRED);
        }
      } else if (described.kind === "invalid_code") {
        setCodeError(described.message);
      } else {
        setError(described.message);
        if (described.kind === "session_expired") void auth.reload();
      }
    } finally {
      setBusy(false);
    }
  }

  if (requiresStepUp && !window.enrolled) {
    return (
      <section aria-label={title} style={panel}>
        <strong>{title}</strong>
        <p style={text}>Esta ação exige um autenticador cadastrado.</p>
        <div style={row}>
          {onGoToSecurity && <button type="button" onClick={onGoToSecurity} style={buttonStyle}>cadastre seu autenticador</button>}
          <button type="button" onClick={onCancel} style={secondaryButtonStyle}>Cancelar</button>
        </div>
      </section>
    );
  }

  return (
    <section aria-label={title} style={panel}>
      <strong>{title}</strong>
      {description && <div style={text}>{description}</div>}
      <form onSubmit={submit} style={{ display: "flex", flexDirection: "column", gap: 10 }}>
        {error && <p role="alert" style={alert}>{error}</p>}
        {fields.map((f) => (
          <label key={f.name} style={label}>
            {f.label}
            <input
              value={values[f.name] ?? ""}
              onChange={(e) => setValues((prev) => ({ ...prev, [f.name]: e.target.value }))}
              style={inputStyle}
            />
          </label>
        ))}
        {fieldError && <p role="alert" style={alert}>{fieldError}</p>}
        {requiresStepUp && !showCode && (
          <p style={text}>verificação válida por mais {Math.ceil(window.remainingMs / 60_000)} min</p>
        )}
        {showCode && (
          <>
            {notice && <p style={text}>{notice}</p>}
            <label style={label}>
              Código do autenticador
              <input
                value={code}
                onChange={(e) => setCode(e.target.value)}
                autoComplete="one-time-code"
                inputMode="numeric"
                style={inputStyle}
              />
            </label>
            {codeError && <p role="alert" style={alert}>{codeError}</p>}
          </>
        )}
        <div style={row}>
          <button type="submit" disabled={busy} style={buttonStyle}>{confirmLabel}</button>
          <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Cancelar</button>
        </div>
      </form>
    </section>
  );
}

const panel = { display: "flex", flexDirection: "column" as const, gap: 10, padding: 16,
  border: "1px solid var(--rule)", borderRadius: 8, background: "var(--panel)" };
const row = { display: "flex", gap: 8 };
const text = { margin: 0, fontSize: 12.5, color: "var(--ink2)" };
const label = { display: "flex", flexDirection: "column" as const, gap: 4, fontSize: 12, color: "var(--ink2)" };
const alert = { margin: 0, fontSize: 12, color: "var(--down)" };
```

**Atenção**, ponto que o teste do "mfa_required repete UMA vez" exige: quando o primeiro `mfa_required` chega com a janela prevista **aberta**, o segundo envio tem de fazer step-up (`showCode` é true por `forceCode`) e repetir `run`. Se a ação voltar `mfa_required` de novo, a mensagem é `GAVE_UP`.

- [ ] **Step 5: Rodar e ver passar**

```bash
npx vitest run src/components/SensitiveAction.test.tsx && npm run typecheck && npm test
```
Esperado: verde. Se o `findByText("verificação válida por mais 4 min")` oscilar porque o minuto virou durante o teste (1 min atrás → `ceil(4 min − ε)` = 4), mantenha: `ceil` dá 4 para qualquer restante entre 3 min e 4 min.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/lib/useStepUp.ts src/components/formStyles.ts src/components/SensitiveAction.tsx src/components/SensitiveAction.test.tsx
/opt/homebrew/bin/git commit -m "feat: add a shared sensitive action panel with window-aware step-up" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: dashboard — página "Segurança da conta"

**Files:**
- Modify: `apps/dashboard/package.json`, `apps/dashboard/package-lock.json`, com `qrcode` e `@types/qrcode`
- Create: `apps/dashboard/src/modules/Security.tsx`
- Test: `apps/dashboard/src/modules/Security.test.tsx`
- Modify: `apps/dashboard/src/shell/modules.ts`, `apps/dashboard/src/App.tsx`

**Interfaces:**
- Consumes:
  - `enrollMfa`, `confirmMfa`, `MfaEnrollment` e `errorCode` (Task 2);
  - `useStepUp` e `SensitiveAction` (Task 3);
  - `useAuth().reload`.
- Produces:
  - `export function Security(): JSX.Element`;
  - `ModuleId` ganha `"security"`, e `NAV_GROUPS` ganha o grupo `{ label: "Conta", items: [ { id: "security", label: "Segurança", icon: "⚿" } ] }`.

**Comportamento (spec §5.2 "Segurança da conta"):**
- **Sem TOTP** (`!window.enrolled`) e sem cadastro em andamento: o texto "Cadastre um autenticador (Google Authenticator, 1Password, etc.) para assinar e aprovar ações sensíveis." e o botão **"Cadastrar autenticador"**.
  - O clique chama `enrollMfa()`, com o botão desabilitado enquanto espera, **só no clique**.
- **Cadastro em andamento** (`enrollment` no estado):
  - o QR (`toDataURL(otpauth_uri)` da biblioteca `qrcode`, num `<img alt="QR do autenticador">`, gerado num efeito a partir do `otpauth_uri`, sem rede);
  - a chave em texto (o parâmetro `secret` do `otpauth_uri`, via `new URL(uri).searchParams.get("secret")`);
  - os códigos de recuperação em lista, com o aviso: **"Cada código vale como o autenticador, uma única vez. Guarde-os fora do computador."**;
  - uma caixa "guardei os códigos", que libera o campo "Código do autenticador" e o botão "Confirmar cadastro" → `confirmMfa(code)`.
  - Na confirmação:
    - sucesso: descarta `enrollment`, `auth.reload()` e mostra `role="status"` "autenticador ativo";
    - `invalid_code`: "código inválido — confira se o relógio do celular está certo", com o campo limpo;
    - outro erro: a mensagem de `describeActionError`.
- **Com TOTP:** "Autenticador ativo." A janela, se aberta: "verificação válida por mais N min". O botão **"Trocar autenticador"** abre um `SensitiveAction`:
  - `title` "Trocar autenticador";
  - `requiresStepUp`;
  - `description` "O autenticador atual deixa de valer assim que você confirmar o novo.";
  - `run` = `enrollMfa()`, cujo resultado vai para o estado `enrollment`. A partir daí, é o fluxo "cadastro em andamento" acima.
- Chave, QR e códigos existem só no estado do componente. Nada vai para `localStorage`, React Query ou console.

- [ ] **Step 1: Dependência**

```bash
cd apps/dashboard && npm install qrcode@^1.5.4 && npm install -D @types/qrcode@^1.5.5
```
São as mesmas versões de `apps/maintenance/package.json`. O container `dashboard` roda `npm install` ao subir, então, para o dev server enxergar a dependência, `docker compose restart dashboard` na raiz do monorepo.

- [ ] **Step 2: Escrever os testes que falham**

`src/modules/Security.test.tsx`:

```tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { StrictMode, type ReactNode } from "react";

vi.mock("qrcode", () => ({ toDataURL: vi.fn().mockResolvedValue("data:image/png;base64,QR") }));
vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), enrollMfa: vi.fn(), confirmMfa: vi.fn(), stepUpMfa: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { AuthProvider } from "../lib/auth";
import { Security } from "./Security";

afterEach(cleanup);

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ENROLLMENT = { otpauth_uri: "otpauth://totp/Rota%20Sa%C3%BAde:ana?secret=JBSWY3DPEHPK3PXP&issuer=Rota%20Sa%C3%BAde",
                     recovery_codes: [ "aaaa111111", "bbbb222222" ] };

function session(overrides: Partial<api.SessionUser> = {}): api.SessionUser {
  return { id: "u1", email_address: "ana@cidade.gov.br", operator: false, memberships: [],
           mfa_enrolled: false, mfa_verified_at: null, ...overrides };
}

function renderSecurity() {
  function wrapper({ children }: { children: ReactNode }) {
    return <StrictMode><AuthProvider>{children}</AuthProvider></StrictMode>;
  }
  return render(<Security />, { wrapper });
}

describe("Security", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.enrollMfa, api.confirmMfa, api.stepUpMfa ]) mocked(fn).mockReset();
  });

  it("sem TOTP: nada é cadastrado sozinho, nem sob StrictMode", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session());
    renderSecurity();

    expect(await screen.findByRole("button", { name: "Cadastrar autenticador" })).not.toBeNull();
    expect(api.enrollMfa).not.toHaveBeenCalled();
  });

  it("cadastrar: uma chamada só, QR local, chave e códigos com aviso", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session());
    mocked(api.enrollMfa).mockResolvedValue(ENROLLMENT);
    renderSecurity();

    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar autenticador" }));

    expect(await screen.findByAltText("QR do autenticador")).toHaveProperty("src", "data:image/png;base64,QR");
    expect(api.enrollMfa).toHaveBeenCalledTimes(1);
    expect(screen.getByText("JBSWY3DPEHPK3PXP")).not.toBeNull();
    expect(screen.getByText("aaaa111111")).not.toBeNull();
    expect(screen.getByText("Cada código vale como o autenticador, uma única vez. Guarde-os fora do computador.")).not.toBeNull();
  });

  it("confirmação só depois de 'guardei os códigos'; sucesso recarrega a sessão", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session());
    mocked(api.enrollMfa).mockResolvedValue(ENROLLMENT);
    mocked(api.confirmMfa).mockResolvedValue(undefined);
    renderSecurity();

    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar autenticador" }));
    await screen.findByAltText("QR do autenticador");
    expect(screen.queryByLabelText("Código do autenticador")).toBeNull();

    fireEvent.click(screen.getByLabelText("guardei os códigos"));
    mocked(api.fetchCurrentSession).mockResolvedValue(session({ mfa_enrolled: true }));
    fireEvent.change(screen.getByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar cadastro" }));

    expect(await screen.findByRole("status")).toHaveProperty("textContent", "autenticador ativo");
    expect(api.confirmMfa).toHaveBeenCalledWith("123456");
    expect(screen.queryByText("aaaa111111")).toBeNull();
  });

  it("código errado na confirmação: mensagem, campo limpo, cadastro continua na tela", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session());
    mocked(api.enrollMfa).mockResolvedValue(ENROLLMENT);
    mocked(api.confirmMfa).mockRejectedValue(new ApiError(422, { error: "invalid_code" }, "422"));
    renderSecurity();

    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar autenticador" }));
    await screen.findByAltText("QR do autenticador");
    fireEvent.click(screen.getByLabelText("guardei os códigos"));
    fireEvent.change(screen.getByLabelText("Código do autenticador"), { target: { value: "000000" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar cadastro" }));

    expect(await screen.findByText("código inválido — confira se o relógio do celular está certo")).not.toBeNull();
    expect((screen.getByLabelText("Código do autenticador") as HTMLInputElement).value).toBe("");
    expect(screen.getByAltText("QR do autenticador")).not.toBeNull();
  });

  it("com TOTP: trocar passa por step-up antes de cadastrar de novo", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session({ mfa_enrolled: true }));
    mocked(api.stepUpMfa).mockResolvedValue(undefined);
    mocked(api.enrollMfa).mockResolvedValue(ENROLLMENT);
    renderSecurity();

    expect(await screen.findByText("Autenticador ativo.")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Trocar autenticador" }));
    fireEvent.change(await screen.findByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    expect(await screen.findByAltText("QR do autenticador")).not.toBeNull();
    expect(mocked(api.stepUpMfa).mock.invocationCallOrder[0])
      .toBeLessThan(mocked(api.enrollMfa).mock.invocationCallOrder[0]);
  });
});
```

Acrescente também, num `src/shell/modules.test.ts` novo:

```ts
import { describe, expect, it } from "vitest";
import { NAV_GROUPS, labelFor } from "./modules";

describe("modules", () => {
  it("Segurança fica no grupo Conta", () => {
    const conta = NAV_GROUPS.find((g) => g.label === "Conta");
    expect(conta?.items.map((i) => i.id)).toEqual([ "security" ]);
    expect(labelFor("security")).toBe("Segurança");
  });
});
```

- [ ] **Step 3: Rodar e ver falhar**

Run: `npx vitest run src/modules/Security.test.tsx src/shell/modules.test.ts`
Esperado: FAIL, o módulo não existe e o grupo "Conta" também não.

- [ ] **Step 4: Implementar `Security.tsx`**

```tsx
import { useEffect, useState } from "react";
import { toDataURL } from "qrcode";
import { confirmMfa, enrollMfa, type MfaEnrollment } from "../lib/api";
import { useAuth } from "../lib/auth";
import { useStepUp } from "../lib/useStepUp";
import { describeActionError } from "../lib/actionErrors";
import { SensitiveAction } from "../components/SensitiveAction";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { buttonStyle, inputStyle } from "../components/formStyles";

// Segurança da conta (spec do dashboard §5.2). Cadastro e troca do
// autenticador do usuário da cidade — pré-requisito do step-up que assinar e
// aprovar exigem.
//
// Segredos: a chave, o QR e os códigos de recuperação vivem só no estado
// deste componente (somem ao sair da tela). O QR é gerado aqui, localmente —
// ele carrega o segredo e nunca vai para um serviço externo. `enrollMfa` só
// roda num clique: num efeito, o StrictMode chamaria duas vezes e o segundo
// cadastro trocaria o segredo do primeiro.
const RECOVERY_WARNING = "Cada código vale como o autenticador, uma única vez. Guarde-os fora do computador.";

export function Security() {
  const auth = useAuth();
  const { window } = useStepUp();
  const [ enrollment, setEnrollment ] = useState<MfaEnrollment | null>(null);
  const [ qr, setQr ] = useState<string | null>(null);
  const [ saved, setSaved ] = useState(false);
  const [ code, setCode ] = useState("");
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const [ done, setDone ] = useState(false);
  const [ replacing, setReplacing ] = useState(false);

  useEffect(() => {
    if (!enrollment) { setQr(null); return; }
    let live = true;
    void toDataURL(enrollment.otpauth_uri).then((url) => { if (live) setQr(url); });
    return () => { live = false; };
  }, [ enrollment ]);

  function start(next: MfaEnrollment) {
    setEnrollment(next); setSaved(false); setCode(""); setError(null); setDone(false); setReplacing(false);
  }

  async function begin() {
    if (busy) return;
    setBusy(true); setError(null);
    try { start(await enrollMfa()); }
    catch (err) {
      const described = describeActionError(err);
      setError("message" in described ? described.message : "não foi possível concluir — tente de novo");
    }
    finally { setBusy(false); }
  }

  async function confirm(event: React.FormEvent) {
    event.preventDefault();
    if (busy) return;
    const typed = code;
    setCode("");
    if (!typed.trim()) { setError("informe o código do autenticador"); return; }
    setBusy(true); setError(null);
    try {
      await confirmMfa(typed);
      setEnrollment(null);
      await auth.reload();
      setDone(true);
    } catch (err) {
      const described = describeActionError(err);
      setError(described.kind === "invalid_code"
        ? "código inválido — confira se o relógio do celular está certo"
        : "message" in described ? described.message : "não foi possível concluir — tente de novo");
    } finally {
      setBusy(false);
    }
  }

  const secret = enrollment ? new URL(enrollment.otpauth_uri).searchParams.get("secret") : null;

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Segurança" sub="conta · autenticador" />
      <Panel title="Autenticador">
        <div style={{ display: "flex", flexDirection: "column", gap: 12, fontSize: 12.5 }}>
          {done && <p role="status" style={{ margin: 0 }}>autenticador ativo</p>}
          {error && <p role="alert" style={{ margin: 0, color: "var(--down)" }}>{error}</p>}

          {enrollment ? (
            <>
              {qr && <img src={qr} alt="QR do autenticador" width={180} height={180} />}
              {secret && <p style={{ margin: 0 }}>Chave: <span className="mono">{secret}</span></p>}
              <p style={{ margin: 0, fontWeight: 600 }}>{RECOVERY_WARNING}</p>
              <ul className="mono" style={{ margin: 0 }}>
                {enrollment.recovery_codes.map((c) => <li key={c}>{c}</li>)}
              </ul>
              <label style={{ display: "flex", gap: 6, alignItems: "center" }}>
                <input type="checkbox" checked={saved} onChange={(e) => setSaved(e.target.checked)} />
                guardei os códigos
              </label>
              {saved && (
                <form onSubmit={confirm} style={{ display: "flex", flexDirection: "column", gap: 8, maxWidth: 280 }}>
                  <label style={{ display: "flex", flexDirection: "column", gap: 4 }}>
                    Código do autenticador
                    <input value={code} onChange={(e) => setCode(e.target.value)}
                           autoComplete="one-time-code" inputMode="numeric" style={inputStyle} />
                  </label>
                  <button type="submit" disabled={busy} style={buttonStyle}>Confirmar cadastro</button>
                </form>
              )}
            </>
          ) : window.enrolled ? (
            <>
              <p style={{ margin: 0 }}>Autenticador ativo.</p>
              {window.open && <p style={{ margin: 0 }}>verificação válida por mais {Math.ceil(window.remainingMs / 60_000)} min</p>}
              {replacing ? (
                <SensitiveAction
                  title="Trocar autenticador"
                  description="O autenticador atual deixa de valer assim que você confirmar o novo."
                  requiresStepUp
                  run={async () => { start(await enrollMfa()); }}
                  onDone={() => setReplacing(false)}
                  onCancel={() => setReplacing(false)}
                />
              ) : (
                <div><button type="button" onClick={() => setReplacing(true)} style={buttonStyle}>Trocar autenticador</button></div>
              )}
            </>
          ) : (
            <>
              <p style={{ margin: 0 }}>
                Cadastre um autenticador (Google Authenticator, 1Password, etc.) para assinar e aprovar ações sensíveis.
              </p>
              <div><button type="button" onClick={() => void begin()} disabled={busy} style={buttonStyle}>Cadastrar autenticador</button></div>
            </>
          )}
        </div>
      </Panel>
    </div>
  );
}
```

Confira as props de `PageHeader` e `Panel` em `src/components/` e ajuste só a chamada, se diferirem.

**Nota sobre a troca:** o `SensitiveAction` chama `start(...)` dentro de `run`, e `start` já faz `setReplacing(false)`. O `onDone` só repete isso, sem efeito.

- [ ] **Step 5: Ligar o módulo**

`src/shell/modules.ts`: acrescente `| "security"` ao `ModuleId` e, ao fim de `NAV_GROUPS`:

```ts
  { label: "Conta", items: [
    { id: "security", label: "Segurança", icon: "⚿" }
  ]}
```

`src/App.tsx`: `import { Security } from "./modules/Security";` e `case "security": return <Security />;` no `renderModule`.

- [ ] **Step 6: Rodar e ver passar**

```bash
npx vitest run src/modules/Security.test.tsx src/shell/modules.test.ts && npm run typecheck && npm test && npm run build
```
Esperado: verde.

- [ ] **Step 7: Verificação no navegador** (stack de dev de pé)

1. Na raiz do monorepo, `docker compose restart dashboard`, para instalar o `qrcode` no container.
2. Abra o dashboard da cidade, `http://curitiba.localhost:5175/dashboard/`, com uma conta de dev da cidade. As contas estão no README da raiz ou do dashboard; não invente.
3. Vá em Conta › Segurança e cadastre. Confira:
   - o QR aparece;
   - a aba de rede mostra **um** `POST /mfa/enroll`;
   - a confirmação com um código de um app autenticador (ou gerado a partir da chave) leva a "autenticador ativo".
4. Clique em "Trocar autenticador", informe um código e confira que o cadastro recomeça.

Registre no relatório o que foi visto, sem a chave e sem os códigos.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add package.json package-lock.json src/modules/Security.tsx src/modules/Security.test.tsx src/shell/modules.ts src/shell/modules.test.ts src/App.tsx
/opt/homebrew/bin/git commit -m "feat: add the account security page for authenticator enrollment" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Fora desta fatia

- **Equipe e o step-up na concessão de papel:** fatia 2.
- **Protocolos com assinaturas e ciclo:** fatia 3, que usa `SensitiveAction` com `onGoToSecurity` apontando para o módulo `security`.
- **Login com TOTP e cadastro obrigatório:** spec §10.
