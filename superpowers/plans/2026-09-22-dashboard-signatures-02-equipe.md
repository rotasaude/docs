# Dashboard — assinaturas, fatia 2: Equipe

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** o `municipal_admin` concede e revoga o papel `protocol_reviewer` pelo dashboard, com step-up, e vê quando a cidade tem menos de 2 revisores — sem isso, nenhuma cidade real consegue publicar protocolo.

**Architecture:** a API passa a exigir step-up em conceder e revogar papel privilegiado (`municipal_admin`, `protocol_reviewer`). O dashboard ganha o módulo Equipe, que lista as pessoas com papel ativo (`GET /setup/memberships`, agrupado por usuário) e usa o `SensitiveAction` da fatia 1 para as duas ações. Uma função pura faz o agrupamento e a contagem de revisores.

**Tech Stack:** Rails 8.1 (apps/api); Vite + React 18 + Vitest + @testing-library/react (apps/dashboard).

**Spec:** `docs/superpowers/specs/2026-09-22-dashboard-protocol-signatures-design.md` (commit `26090e9`). Esta fatia implementa §2 (decisões 5 e 6), §4.1, §5.2 "Equipe" e §9 fatia 2. Ela também **altera** o §8 de `2026-09-18-protocol-signatures-design.md`, que dizia que conceder papel não pedia step-up.

## Global Constraints

- Commits: Conventional Commits **em inglês**, tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Git: `/opt/homebrew/bin/git`. `apps/api`, `apps/dashboard` e `docs` são repositórios separados; a raiz do monorepo não é git. Branch `feat/dashboard-team` em api e dashboard; o docs commita direto em `main` (é só texto). Nunca push, nunca merge.
- Staging explícito: nunca `git add -A` nem `git add .` (apps/dashboard tem um `.superpowers/` fora do git).
- Suíte completa do api: na raiz do monorepo, `docker compose stop worker`, depois `docker compose exec -T api bundle exec rspec`, depois `docker compose start worker` (sempre religar). Acima de ~3 min é regressão.
- Specs de request exigem `type: :request`; `sign_in_as(user)` vem de `spec/support/city_request_auth.rb`, e carimbar `mfa_verified_at` na sessão é como as specs simulam step-up (ver `spec/requests/protocol_lifecycle_spec.rb`).
- **Papéis privilegiados:** `Membership::PRIVILEGED_ROLES = %w[municipal_admin protocol_reviewer]`. A regra nova vale para esses dois, em conceder **e** revogar.
- Janela de step-up: 5 minutos (`MfaStepUp::STEP_UP_WINDOW`). Resposta fora dela: `401 { error: "mfa_required" }`.
- Frontend: nenhuma tela lê status HTTP (tudo por `describeActionError`); mensagens em português; o código TOTP é limpo a cada tentativa (o `SensitiveAction` já cuida disso).
- Nunca imprima segredo, chave TOTP ou código de recuperação.
- Nunca rode `start.sh`.

## File Structure

**apps/api**
- Modify: `app/controllers/setup_controller.rb`, com a guarda de step-up em `grant_role` e `revoke_membership`.
- Create: `spec/requests/setup_privileged_role_step_up_spec.rb`.

**docs**
- Modify: `superpowers/specs/2026-09-18-protocol-signatures-design.md` (§8).

**apps/dashboard**
- Modify: `src/lib/api.ts`, com `listMemberships`, `grantRole` e `revokeMembership`.
- Modify: `src/lib/api.test.ts`.
- Create: `src/lib/team.ts` + `src/lib/team.test.ts`, com o agrupamento por usuário e a contagem de revisores.
- Create: `src/modules/Team.tsx` + `src/modules/Team.test.tsx`.
- Modify: `src/shell/modules.ts` e `src/shell/modules.test.ts`, com o módulo `team` e a visibilidade por papel.
- Modify: `src/App.tsx`, com o `case "team"`.

---

### Task 1: API — step-up para conceder e revogar papel privilegiado

**Files:**
- Modify: `apps/api/app/controllers/setup_controller.rb`
- Test: `apps/api/spec/requests/setup_privileged_role_step_up_spec.rb`
- Modify: `docs/superpowers/specs/2026-09-18-protocol-signatures-design.md`

**Interfaces:**
- Consumes:
  - `MfaStepUp#reauthenticated_recently?` e `#require_step_up!` (`app/controllers/concerns/mfa_step_up.rb`), que responde `401 { error: "mfa_required" }`;
  - `Membership::PRIVILEGED_ROLES`;
  - `can_manage_members?`, o método privado que já existe no controller (`current_user.has_role?("municipal_admin")`).
- Produces:
  - `POST /setup/memberships` com `role` privilegiado, sem janela de step-up → `401 { "error": "mfa_required" }`, **sem criar membership**;
  - `POST /setup/memberships/:id/revoke` de um membership privilegiado, sem janela → `401`, **sem revogar**;
  - papel não privilegiado (ex.: `viewer`) segue sem step-up;
  - com a janela aberta, o comportamento de hoje (201 / 200).

**Ordem das guardas:** `can_manage_members?` primeiro (403 para quem não é admin, independente de step-up), depois o step-up. Um não-admin nunca descobre nada sobre o estado de MFA da própria sessão por esse caminho, e o 403 continua sendo a primeira resposta como hoje.

- [ ] **Step 1: Branch**

```bash
cd apps/api && /opt/homebrew/bin/git checkout -b feat/dashboard-team
```

- [ ] **Step 2: Escrever a spec que falha**

`spec/requests/setup_privileged_role_step_up_spec.rb`. Use `spec/requests/setup_grant_role_spec.rb` como molde do arranjo (usuários, papéis, `sign_in_as`); copie de lá, não invente.

```ruby
require "rails_helper"

# Spec do dashboard §4.1: conceder e revogar papel PRIVILEGIADO
# (Membership::PRIVILEGED_ROLES) exige step-up de MFA. Decidir quem revisa
# protocolo é decidir quem aprova protocolo clínico: uma sessão roubada, só
# com a senha, montaria dois revisores e publicaria qualquer coisa.
#
# Muda o §8 de 2026-09-18-protocol-signatures-design.md, que dizia
# "não em conceder papel".
RSpec.describe "Setup privileged role step-up", type: :request do
  def json = JSON.parse(response.body)

  def enrolled_user(email:, role: nil)
    u = User.create!(email_address: email, password: "secret123")
    Mfa::Enroll.call(u)
    u.update!(otp_enabled: true)
    Membership.create!(user: u, role: role, granted_at: Time.current) if role
    u
  end

  let!(:admin)  { enrolled_user(email: "adm-#{SecureRandom.hex(3)}@example.org", role: "municipal_admin") }
  let!(:target) { enrolled_user(email: "alvo-#{SecureRandom.hex(3)}@example.org", role: "protocol_publisher") }

  def sign_in_admin!(stepped_up:)
    session = sign_in_as(admin)
    session.update!(mfa_verified_at: Time.current) if stepped_up
    session
  end

  def grant!(role: "protocol_reviewer")
    post "/setup/memberships", params: { user_id: target.id, role: role }, as: :json
  end

  describe "conceder" do
    it "sem janela de step-up: 401 mfa_required e nenhuma membership criada" do
      sign_in_admin!(stepped_up: false)

      expect { grant! }.not_to change { Membership.where(user: target, role: "protocol_reviewer").count }
      expect(response).to have_http_status(:unauthorized)
      expect(json).to eq("error" => "mfa_required")
    end

    it "com a janela aberta: concede" do
      sign_in_admin!(stepped_up: true)

      grant!

      expect(response).to have_http_status(:created)
      expect(target.reload.has_role?("protocol_reviewer")).to be(true)
    end

    it "com a janela vencida (6 min): recusa" do
      sign_in_as(admin).update!(mfa_verified_at: 6.minutes.ago)

      grant!

      expect(response).to have_http_status(:unauthorized)
    end

    it "papel NÃO privilegiado continua sem step-up" do
      sign_in_admin!(stepped_up: false)

      grant!(role: "viewer")

      expect(response).to have_http_status(:created)
      expect(target.reload.has_role?("viewer")).to be(true)
    end

    it "quem não é municipal_admin leva 403 antes de qualquer checagem de MFA" do
      other = enrolled_user(email: "zz-#{SecureRandom.hex(3)}@example.org", role: "protocol_publisher")
      sign_in_as(other)

      grant!

      expect(response).to have_http_status(:forbidden)
    end
  end

  describe "revogar" do
    let!(:membership) { Membership.create!(user: target, role: "protocol_reviewer", granted_at: Time.current) }

    it "sem janela de step-up: 401 e a membership continua ativa" do
      sign_in_admin!(stepped_up: false)

      post "/setup/memberships/#{membership.id}/revoke", as: :json

      expect(response).to have_http_status(:unauthorized)
      expect(json).to eq("error" => "mfa_required")
      expect(membership.reload.revoked_at).to be_nil
    end

    it "com a janela aberta: revoga" do
      sign_in_admin!(stepped_up: true)

      post "/setup/memberships/#{membership.id}/revoke", as: :json

      expect(response).to have_http_status(:ok)
      expect(membership.reload.revoked_at).to be_present
    end

    it "papel não privilegiado é revogado sem step-up" do
      plain = Membership.create!(user: target, role: "viewer", granted_at: Time.current)
      sign_in_admin!(stepped_up: false)

      post "/setup/memberships/#{plain.id}/revoke", as: :json

      expect(response).to have_http_status(:ok)
      expect(plain.reload.revoked_at).to be_present
    end
  end
end
```

- [ ] **Step 3: Rodar e ver falhar**

Da raiz do monorepo:
```bash
docker compose exec -T api bundle exec rspec spec/requests/setup_privileged_role_step_up_spec.rb
```
Esperado: FAIL nos exemplos "sem janela" e "janela vencida" (hoje respondem 201/200).

- [ ] **Step 4: Implementar**

Em `app/controllers/setup_controller.rb`, acrescente `include MfaStepUp` ao lado dos outros `include`, e a guarda nas duas ações, **depois** do `can_manage_members?`:

```ruby
  def grant_role
    return head(:forbidden) unless can_manage_members?
    return require_step_up! if privileged_role?(params[:role])
```

```ruby
  def revoke_membership
    membership = Membership.find_by(id: params[:id])
    return head(:not_found) unless membership
    return head(:forbidden) unless can_manage_members?
    return require_step_up! if privileged_role?(membership.role)
```

e, junto dos outros métodos privados:

```ruby
  # Spec do dashboard §4.1: conceder ou revogar um papel de
  # Membership::PRIVILEGED_ROLES exige verificação recente de TOTP — quem
  # decide quem revisa protocolo decide quem aprova protocolo clínico, e o
  # login da cidade é só senha. Papel comum (viewer, author, publisher) segue
  # sem step-up: o §8 do spec de assinaturas mudou só para os privilegiados.
  def privileged_role?(role)
    Membership::PRIVILEGED_ROLES.include?(role.to_s)
  end
```

`require_step_up!` já renderiza; o `return` impede o resto da ação.

- [ ] **Step 5: Rodar e ver passar**

```bash
docker compose exec -T api bundle exec rspec spec/requests/setup_privileged_role_step_up_spec.rb spec/requests/setup_grant_role_spec.rb
```
Esperado: PASS. Se `setup_grant_role_spec.rb` falhar porque concede papel privilegiado sem step-up, **adapte aquele spec** (carimbe `mfa_verified_at` no arranjo), sem afrouxar o que ele prova.

- [ ] **Step 6: Suíte completa** (worker parado; ver Global Constraints). Esperado: 0 falhas.

- [ ] **Step 7: Corrigir o §8 do spec de assinaturas**

No repositório `docs`, em `superpowers/specs/2026-09-18-protocol-signatures-design.md`, a última frase do primeiro item de §8 diz hoje:

> O step-up da cidade usa o `MfaStepUp` existente, exigido em assinar, publicar, ativar, aposentar e reverter; não em enviar para revisão nem em conceder papel.

Troque por:

> O step-up da cidade usa o `MfaStepUp` existente, exigido em assinar, publicar, ativar, aposentar, reverter e também em conceder ou revogar papel privilegiado (`Membership::PRIVILEGED_ROLES`, decidido na fatia 2 das telas do dashboard — quem escolhe os revisores decide quem aprova protocolo clínico); não em enviar para revisão.

Ajuste também a linha da tabela de §8 sobre conceder papel, para dizer "com step-up".

```bash
cd ../../docs   # ou o caminho do repositório docs
/opt/homebrew/bin/git add superpowers/specs/2026-09-18-protocol-signatures-design.md
/opt/homebrew/bin/git commit -m "docs: require step-up to grant or revoke a privileged role" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

- [ ] **Step 8: Commit do api**

```bash
/opt/homebrew/bin/git add app/controllers/setup_controller.rb spec/requests/setup_privileged_role_step_up_spec.rb
/opt/homebrew/bin/git add -u spec/requests/setup_grant_role_spec.rb   # se foi adaptado
/opt/homebrew/bin/git commit -m "feat: require step-up to grant or revoke a privileged role" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: dashboard — cliente e a regra pura da equipe

**Files:**
- Modify: `apps/dashboard/src/lib/api.ts`, `apps/dashboard/src/lib/api.test.ts`
- Create: `apps/dashboard/src/lib/team.ts`, `apps/dashboard/src/lib/team.test.ts`

**Interfaces:**
- Consumes: `jsonFetch` e `ApiError` (`src/lib/api.ts`). Formato de `GET /setup/memberships` (ver `SetupController#list_memberships`):
  ```json
  { "data": [ { "id": "uuid", "user": { "id": "uuid", "email_address": "a@b" }, "role": "protocol_reviewer", "granted_at": "2026-09-22T12:00:00Z" } ] }
  ```
  Só memberships **ativas** vêm; a lista é por membership, então a mesma pessoa aparece uma vez por papel.
- Produces:
  ```ts
  // api.ts
  export interface MembershipRow { id: string; user: { id: string; email_address: string }; role: string; granted_at: string; }
  export function listMemberships(): Promise<MembershipRow[]>;               // GET /setup/memberships → data
  export function grantRole(userId: string, role: string): Promise<void>;    // POST /setup/memberships
  export function revokeMembership(id: string): Promise<void>;               // POST /setup/memberships/:id/revoke
  // team.ts
  export const REVIEWER_ROLE = "protocol_reviewer";
  export const REQUIRED_REVIEWERS = 2;
  export interface TeamMember {
    userId: string; email: string; roles: string[];
    isReviewer: boolean; reviewerMembershipId: string | null;
  }
  export function teamMembers(rows: MembershipRow[]): TeamMember[];   // por usuário, e-mail asc, papéis asc
  export function reviewerCount(members: TeamMember[]): number;
  ```

- [ ] **Step 1: Branch**

```bash
cd apps/dashboard && /opt/homebrew/bin/git checkout -b feat/dashboard-team
```

- [ ] **Step 2: Escrever os testes que falham — regra pura**

`src/lib/team.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { REQUIRED_REVIEWERS, reviewerCount, teamMembers } from "./team";
import type { MembershipRow } from "./api";

function row(email: string, role: string, id = `${email}-${role}`): MembershipRow {
  return { id, user: { id: `u-${email}`, email_address: email }, role, granted_at: "2026-09-01T00:00:00Z" };
}

describe("teamMembers", () => {
  it("agrupa as memberships por usuário, com os papéis em ordem", () => {
    const members = teamMembers([
      row("bia@cidade.gov.br", "protocol_reviewer"),
      row("ana@cidade.gov.br", "protocol_publisher"),
      row("ana@cidade.gov.br", "municipal_admin")
    ]);

    expect(members.map((m) => m.email)).toEqual([ "ana@cidade.gov.br", "bia@cidade.gov.br" ]);
    expect(members[0].roles).toEqual([ "municipal_admin", "protocol_publisher" ]);
    expect(members[0].isReviewer).toBe(false);
    expect(members[0].reviewerMembershipId).toBeNull();
  });

  it("marca quem é revisor e guarda o id da membership de revisor (para revogar)", () => {
    const members = teamMembers([ row("bia@cidade.gov.br", "protocol_reviewer", "m-9") ]);

    expect(members[0].isReviewer).toBe(true);
    expect(members[0].reviewerMembershipId).toBe("m-9");
  });

  it("lista vazia vira equipe vazia", () => {
    expect(teamMembers([])).toEqual([]);
  });
});

describe("reviewerCount", () => {
  it("conta pessoas, não memberships", () => {
    const members = teamMembers([
      row("ana@cidade.gov.br", "protocol_reviewer"),
      row("ana@cidade.gov.br", "protocol_publisher"),
      row("bia@cidade.gov.br", "protocol_reviewer")
    ]);

    expect(reviewerCount(members)).toBe(2);
    expect(REQUIRED_REVIEWERS).toBe(2);
  });

  it("zero quando ninguém é revisor", () => {
    expect(reviewerCount(teamMembers([ row("ana@cidade.gov.br", "viewer") ]))).toBe(0);
  });
});
```

- [ ] **Step 3: Escrever os testes que falham — cliente**

Acrescente a `src/lib/api.test.ts` (usa o `mockFetch` que já existe no arquivo; importe `listMemberships, grantRole, revokeMembership`):

```ts
describe("memberships", () => {
  function lastCall() {
    const fetchMock = globalThis.fetch as unknown as ReturnType<typeof vi.fn>;
    const [ url, init ] = fetchMock.mock.calls.at(-1) as [ string, RequestInit ];
    return { url, init };
  }

  it("listMemberships devolve data", async () => {
    mockFetch(200, { data: [ { id: "m1", user: { id: "u1", email_address: "a@b" }, role: "viewer", granted_at: "2026-09-01T00:00:00Z" } ] });

    const rows = await listMemberships();

    expect(lastCall().url).toBe("/setup/memberships");
    expect(rows).toHaveLength(1);
    expect(rows[0].user.email_address).toBe("a@b");
  });

  it("grantRole manda user_id e role em JSON", async () => {
    mockFetch(201, { id: "m2" });

    await grantRole("u1", "protocol_reviewer");

    const { url, init } = lastCall();
    expect(url).toBe("/setup/memberships");
    expect(init.method).toBe("POST");
    expect(JSON.parse(init.body as string)).toEqual({ user_id: "u1", role: "protocol_reviewer" });
  });

  it("revokeMembership chama a rota de revogação", async () => {
    mockFetch(200, { id: "m2", revoked_at: "2026-09-22T00:00:00Z" });

    await revokeMembership("m2");

    const { url, init } = lastCall();
    expect(url).toBe("/setup/memberships/m2/revoke");
    expect(init.method).toBe("POST");
  });
});
```

- [ ] **Step 4: Rodar e ver falhar**

Run: `npx vitest run src/lib/team.test.ts src/lib/api.test.ts`
Esperado: FAIL — módulo e exports inexistentes.

- [ ] **Step 5: Implementar**

Em `src/lib/api.ts`, depois das funções de MFA:

```ts
const SETUP_BASE = import.meta.env.VITE_SETUP_BASE || "/setup";

export interface MembershipRow {
  id: string;
  user: { id: string; email_address: string };
  role: string;
  granted_at: string;
}

// Só memberships ATIVAS (SetupController#list_memberships): uma linha por
// papel, então a mesma pessoa aparece mais de uma vez — quem agrupa é
// src/lib/team.ts.
export async function listMemberships(): Promise<MembershipRow[]> {
  const payload = await jsonFetch<{ data: MembershipRow[] }>(`${SETUP_BASE}/memberships`);
  return payload.data;
}

export async function grantRole(userId: string, role: string): Promise<void> {
  await jsonFetch<unknown>(`${SETUP_BASE}/memberships`, {
    method: "POST", body: JSON.stringify({ user_id: userId, role })
  });
}

export async function revokeMembership(id: string): Promise<void> {
  await jsonFetch<unknown>(`${SETUP_BASE}/memberships/${encodeURIComponent(id)}/revoke`, {
    method: "POST", body: "{}"
  });
}
```

`body: "{}"` na revogação de propósito: `jsonFetch` só põe `Content-Type` quando há corpo, e a API recusa escrita por cookie sem `application/json`.

`src/lib/team.ts`:

```ts
// A equipe da cidade, vista por PESSOA (spec do dashboard §5.2). A API
// devolve uma linha por membership ativa; aqui elas viram uma linha por
// usuário, com os papéis juntos.
//
// Limite conhecido: quem não tem nenhum papel ativo não aparece — a API não
// lista usuários, lista memberships. Na prática todo usuário da cidade nasce
// de um convite com papel.
import type { MembershipRow } from "./api";

export const REVIEWER_ROLE = "protocol_reviewer";
export const REQUIRED_REVIEWERS = 2;

export interface TeamMember {
  userId: string;
  email: string;
  roles: string[];
  isReviewer: boolean;
  reviewerMembershipId: string | null;
}

export function teamMembers(rows: MembershipRow[]): TeamMember[] {
  const byUser = new Map<string, TeamMember>();

  for (const row of rows) {
    const current = byUser.get(row.user.id) ?? {
      userId: row.user.id, email: row.user.email_address, roles: [],
      isReviewer: false, reviewerMembershipId: null
    };
    current.roles = [ ...current.roles, row.role ].sort();
    if (row.role === REVIEWER_ROLE) {
      current.isReviewer = true;
      current.reviewerMembershipId = row.id;
    }
    byUser.set(row.user.id, current);
  }

  return [ ...byUser.values() ].sort((a, b) => a.email.localeCompare(b.email));
}

export function reviewerCount(members: TeamMember[]): number {
  return members.filter((m) => m.isReviewer).length;
}
```

- [ ] **Step 6: Rodar e ver passar**

```bash
npx vitest run src/lib/team.test.ts src/lib/api.test.ts && npm run typecheck && npm test
```
Esperado: verde.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/lib/api.ts src/lib/api.test.ts src/lib/team.ts src/lib/team.test.ts
/opt/homebrew/bin/git commit -m "feat: add the membership client and the team grouping rule" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: dashboard — o módulo Equipe

**Files:**
- Create: `apps/dashboard/src/modules/Team.tsx`
- Test: `apps/dashboard/src/modules/Team.test.tsx`
- Modify: `apps/dashboard/src/shell/modules.ts`, `apps/dashboard/src/shell/modules.test.ts`, `apps/dashboard/src/App.tsx`

**Interfaces:**
- Consumes:
  - `listMemberships`, `grantRole`, `revokeMembership` (Task 2);
  - `teamMembers`, `reviewerCount`, `REQUIRED_REVIEWERS`, `REVIEWER_ROLE`, `TeamMember` (Task 2);
  - `SensitiveAction` (`src/components/SensitiveAction.tsx`), com as props `{ title, description?, requiresStepUp, fields?, confirmLabel?, run(values), onDone(), onCancel(), onGoToSecurity? }`;
  - `useAuth()` para saber se a pessoa é `municipal_admin` (`user.memberships[].role`);
  - `PageHeader`, `Panel`, `DataTable`, `EmptyState`, `ErrorState`, `Skeleton`, `Tag` (`src/components/`);
  - `describeActionError` (`src/lib/actionErrors.ts`).
- Produces:
  - `export function Team({ onNavigate }: { onNavigate(id: ModuleId): void })`;
  - `ModuleId` ganha `"team"`; `NAV_GROUPS` ganha o grupo `{ label: "Equipe", items: [ { id: "team", label: "Equipe", icon: "☷" } ] }`;
  - `navGroupsFor(user)` passa a esconder esse grupo de quem **não** é `municipal_admin` (além da regra de operador que já existe).

**Comportamento (spec §5.2 "Equipe"):**
- **Carregamento:** React Query, `queryKey: [ "memberships" ]`, `queryFn: listMemberships`. `Skeleton` enquanto carrega; `ErrorState` com a mensagem de `describeActionError` no erro (403 → "seu papel não permite esta ação").
- **Tabela**, uma linha por pessoa: e-mail; papéis (`Tag` por papel, ordem alfabética); e a ação.
- **Ação por linha:**
  - quem não é revisor → botão "Tornar revisor", que abre um `SensitiveAction` com `title` "Tornar revisor", `description` "<e-mail> poderá assinar publicação e ativação de protocolo.", `requiresStepUp` e `run` = `grantRole(userId, REVIEWER_ROLE)`;
  - quem é revisor → botão "Remover revisor", `title` "Remover revisor", `description` "<e-mail> deixa de assinar protocolos. As assinaturas que já deu continuam valendo.", `requiresStepUp` e `run` = `revokeMembership(reviewerMembershipId)`.
  - Um painel por vez; o `onGoToSecurity` aponta para o módulo `security`, via `onNavigate("security")`.
- **Depois de concluir:** fecha o painel, mostra `role="status"` "<e-mail> agora é revisor" ou "<e-mail> não é mais revisor", e invalida `[ "memberships" ]`.
- **Faixa de aviso**, acima da tabela, quando `reviewerCount(members) < REQUIRED_REVIEWERS`: "sem 2 revisores, nenhum protocolo é publicado ou ativado nesta cidade".
- **Sem ninguém na lista:** `EmptyState` "nenhuma pessoa com papel ativo".

- [ ] **Step 1: Escrever os testes que falham**

`src/modules/Team.test.tsx`. Use o molde de `src/modules/Security.test.tsx` (mock de `../lib/api`, `AuthProvider`, `fireEvent`), acrescentando o `QueryClientProvider` que as telas com React Query usam (veja `src/modules/Protocols.tsx` e os testes existentes que montam `QueryClient`).

```tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return {
    ...real,
    fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(),
    listMemberships: vi.fn(), grantRole: vi.fn(), revokeMembership: vi.fn()
  };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { AuthProvider } from "../lib/auth";
import { Team } from "./Team";

afterEach(cleanup);

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function session(overrides: Partial<api.SessionUser> = {}): api.SessionUser {
  return {
    id: "u-admin", email_address: "admin@cidade.gov.br", operator: false,
    memberships: [ { municipality_id: "m1", municipality_name: "Curitiba", municipality_uf: "PR", role: "municipal_admin" } ],
    mfa_enrolled: true, mfa_verified_at: new Date().toISOString(), ...overrides
  };
}

function membership(email: string, role: string, id = `${email}-${role}`) {
  return { id, user: { id: `u-${email}`, email_address: email }, role, granted_at: "2026-09-01T00:00:00Z" };
}

function renderTeam(onNavigate = vi.fn()) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  }
  render(<Team onNavigate={onNavigate} />, { wrapper });
  return { onNavigate };
}

describe("Team", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.stepUpMfa, api.listMemberships, api.grantRole, api.revokeMembership ]) {
      mocked(fn).mockReset();
    }
    mocked(api.fetchCurrentSession).mockResolvedValue(session());
  });

  it("lista uma linha por pessoa, com os papéis juntos", async () => {
    mocked(api.listMemberships).mockResolvedValue([
      membership("ana@cidade.gov.br", "municipal_admin"),
      membership("ana@cidade.gov.br", "protocol_publisher"),
      membership("bia@cidade.gov.br", "protocol_reviewer")
    ]);
    renderTeam();

    expect(await screen.findByText("ana@cidade.gov.br")).not.toBeNull();
    expect(screen.getAllByRole("row")).toHaveLength(3); // cabeçalho + 2 pessoas
    expect(screen.getByText("municipal_admin")).not.toBeNull();
    expect(screen.getByText("protocol_publisher")).not.toBeNull();
  });

  it("avisa quando a cidade tem menos de 2 revisores", async () => {
    mocked(api.listMemberships).mockResolvedValue([ membership("bia@cidade.gov.br", "protocol_reviewer") ]);
    renderTeam();

    expect(await screen.findByText("sem 2 revisores, nenhum protocolo é publicado ou ativado nesta cidade")).not.toBeNull();
  });

  it("com 2 revisores, some o aviso", async () => {
    mocked(api.listMemberships).mockResolvedValue([
      membership("ana@cidade.gov.br", "protocol_reviewer"),
      membership("bia@cidade.gov.br", "protocol_reviewer")
    ]);
    renderTeam();

    await screen.findByText("ana@cidade.gov.br");
    expect(screen.queryByText(/sem 2 revisores/)).toBeNull();
  });

  it("tornar revisor: confirma com a janela aberta e invalida a lista", async () => {
    mocked(api.listMemberships).mockResolvedValue([ membership("ana@cidade.gov.br", "protocol_publisher") ]);
    mocked(api.grantRole).mockResolvedValue(undefined);
    renderTeam();

    fireEvent.click(await screen.findByRole("button", { name: "Tornar revisor" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    await waitFor(() => expect(api.grantRole).toHaveBeenCalledWith("u-ana@cidade.gov.br", "protocol_reviewer"));
    expect((await screen.findByRole("status")).textContent).toBe("ana@cidade.gov.br agora é revisor");
    await waitFor(() => expect(mocked(api.listMemberships).mock.calls.length).toBe(2));
  });

  it("remover revisor: manda o id da membership de revisor", async () => {
    mocked(api.listMemberships).mockResolvedValue([ membership("bia@cidade.gov.br", "protocol_reviewer", "m-9") ]);
    mocked(api.revokeMembership).mockResolvedValue(undefined);
    renderTeam();

    fireEvent.click(await screen.findByRole("button", { name: "Remover revisor" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    await waitFor(() => expect(api.revokeMembership).toHaveBeenCalledWith("m-9"));
    expect((await screen.findByRole("status")).textContent).toBe("bia@cidade.gov.br não é mais revisor");
  });

  it("a recusa da API aparece no painel, que continua aberto", async () => {
    mocked(api.listMemberships).mockResolvedValue([ membership("ana@cidade.gov.br", "protocol_publisher") ]);
    mocked(api.grantRole).mockRejectedValue(new ApiError(422, { error: "already_granted", message: "papel já concedido" }, "422"));
    renderTeam();

    fireEvent.click(await screen.findByRole("button", { name: "Tornar revisor" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    expect(await screen.findByText("papel já concedido")).not.toBeNull();
    expect(screen.getByRole("button", { name: "Confirmar" })).not.toBeNull();
  });

  it("sem TOTP cadastrado, aponta para a Segurança", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session({ mfa_enrolled: false, mfa_verified_at: null }));
    mocked(api.listMemberships).mockResolvedValue([ membership("ana@cidade.gov.br", "protocol_publisher") ]);
    const { onNavigate } = renderTeam();

    fireEvent.click(await screen.findByRole("button", { name: "Tornar revisor" }));
    fireEvent.click(await screen.findByRole("button", { name: "cadastre seu autenticador" }));

    expect(onNavigate).toHaveBeenCalledWith("security");
  });

  it("403 na listagem mostra a mensagem de papel", async () => {
    mocked(api.listMemberships).mockRejectedValue(new ApiError(403, "", "403"));
    renderTeam();

    expect(await screen.findByText("seu papel não permite esta ação")).not.toBeNull();
  });

  it("lista vazia mostra estado vazio", async () => {
    mocked(api.listMemberships).mockResolvedValue([]);
    renderTeam();

    expect(await screen.findByText("nenhuma pessoa com papel ativo")).not.toBeNull();
  });
});
```

Em `src/shell/modules.test.ts`, acrescente:

```ts
  it("Equipe só aparece para municipal_admin", () => {
    const admin = { operator: false, memberships: [ { role: "municipal_admin" } ] };
    const publisher = { operator: false, memberships: [ { role: "protocol_publisher" } ] };

    expect(navGroupsFor(admin).some((g) => g.label === "Equipe")).toBe(true);
    expect(navGroupsFor(publisher).some((g) => g.label === "Equipe")).toBe(false);
    expect(navGroupsFor(null).some((g) => g.label === "Equipe")).toBe(false);
  });
```

**Atenção, dois testes existentes mudam de número:**

- `"operador não vê o grupo Conta"` afirma hoje `expect(groups.length).toBe(NAV_GROUPS.length - 1)`. Com Equipe escondida também (o operador não é `municipal_admin`), o operador passa a ver `NAV_GROUPS.length - 2`. Atualize o número e acrescente `expect(groups.some((g) => g.label === "Equipe")).toBe(false)` — o exemplo passa a provar as duas exclusões.
- `"sem sessão, mostra tudo"` afirma `navGroupsFor(null).length === NAV_GROUPS.length`. Sem sessão, Equipe fica escondida, então o número é `NAV_GROUPS.length - 1`. Renomeie para `"sem sessão, esconde só a Equipe"` e afirme que Conta continua visível e Equipe não.

Sem sessão, o grupo Conta aparece (comportamento atual, mantido), mas Equipe não: ela depende de um papel que só a sessão informa, e mostrá-la antes seria oferecer uma tela que a API recusaria com 403.

- [ ] **Step 2: Rodar e ver falhar**

Run: `npx vitest run src/modules/Team.test.tsx src/shell/modules.test.ts`
Esperado: FAIL — o módulo não existe e `navGroupsFor` não conhece papéis.

- [ ] **Step 3: Implementar `modules.ts`**

- `ModuleId` ganha `| "team"`.
- `NAV_GROUPS` ganha, antes do grupo "Conta":

```ts
  { label: "Equipe", items: [
    { id: "team", label: "Equipe", icon: "☷" }
  ]},
```

- `navGroupsFor` passa a receber também os papéis:

```ts
// D6: "Conta" (autenticador, senha) é de usuário de cidade — um operador
// entra por grant e não tem essas telas no servidor. Sem sessão ainda,
// mostra tudo: filtrar cedo demais esconderia o grupo por um instante para
// quem tem direito a ele.
//
// Fatia 2: "Equipe" é o contrário — só municipal_admin, e escondido enquanto
// não se sabe quem é. A API recusaria (403) para qualquer outro papel, então
// oferecer o item antes da sessão seria oferecer uma porta trancada.
export function navGroupsFor(
  user: { operator: boolean; memberships?: { role: string }[] } | null
): NavGroupDef[] {
  const isAdmin = user?.memberships?.some((m) => m.role === "municipal_admin") ?? false;
  return NAV_GROUPS.filter((group) => {
    if (group.label === "Conta") return !user?.operator;
    if (group.label === "Equipe") return isAdmin;
    return true;
  });
}
```

`AppHeader.tsx` já chama `navGroupsFor(auth.user)`, e `SessionUser` já tem `memberships`, então nada muda lá.

- [ ] **Step 4: Implementar `Team.tsx`**

Estilo e composição no molde de `src/modules/Security.tsx`; confira as props de `PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `ErrorState` e `Skeleton` em `src/components/` e ajuste só as chamadas, se diferirem.

```tsx
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { grantRole, listMemberships, revokeMembership } from "../lib/api";
import { describeActionError } from "../lib/actionErrors";
import { REQUIRED_REVIEWERS, REVIEWER_ROLE, reviewerCount, teamMembers, type TeamMember } from "../lib/team";
import { SensitiveAction } from "../components/SensitiveAction";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { EmptyState } from "../components/EmptyState";
import { ErrorState } from "../components/ErrorState";
import { Skeleton } from "../components/Skeleton";
import { buttonStyle } from "../components/formStyles";
import type { ModuleId } from "../shell/modules";

// Equipe (spec do dashboard §5.2). Só municipal_admin chega aqui — a API
// recusa o resto com 403, e o item de menu já não aparece (navGroupsFor).
//
// Escopo: conceder e revogar APENAS protocol_reviewer. Convidar membro,
// outros papéis e desativar usuário são de outro spec (§10).
//
// Quem traduz recusa da API e cuida do código TOTP é o SensitiveAction; esta
// tela não tenta interpretar erro de ação por conta própria.
const NO_REVIEWERS_WARNING = "sem 2 revisores, nenhum protocolo é publicado ou ativado nesta cidade";

type Pending = { member: TeamMember; kind: "grant" | "revoke" };

export function Team({ onNavigate }: { onNavigate(id: ModuleId): void }) {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: [ "memberships" ], queryFn: listMemberships });
  const [ pending, setPending ] = useState<Pending | null>(null);
  const [ done, setDone ] = useState<string | null>(null);

  const members = teamMembers(query.data ?? []);
  const reviewers = reviewerCount(members);

  function open(member: TeamMember, kind: Pending["kind"]) {
    setPending({ member, kind });
    setDone(null);
  }

  function finish(message: string) {
    setPending(null);
    setDone(message);
    void queryClient.invalidateQueries({ queryKey: [ "memberships" ] });
  }

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Equipe" sub="papéis · revisores de protocolo" />

      {query.isLoading && <Panel title="Pessoas"><Skeleton rows={4} /></Panel>}
      {query.isError && <ErrorState message={describeActionError(query.error).message ?? "erro inesperado"} />}

      {query.isSuccess && (
        <Panel title="Pessoas" sub="papéis ativos">
          <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
            {done && <p role="status" style={{ margin: 0, fontSize: 12.5 }}>{done}</p>}
            {reviewers < REQUIRED_REVIEWERS && (
              <p style={{ margin: 0, fontSize: 12.5, fontWeight: 600 }}>{NO_REVIEWERS_WARNING}</p>
            )}

            {members.length === 0 ? <EmptyState title="nenhuma pessoa com papel ativo" /> : (
              <DataTable<TeamMember>
                cols={[
                  { label: "E-mail", w: "2fr", render: (m) => <span className="mono">{m.email}</span> },
                  { label: "Papéis", w: "2fr", render: (m) => (
                    <span style={{ display: "inline-flex", gap: 4, flexWrap: "wrap" }}>
                      {m.roles.map((role) => <Tag key={role}>{role}</Tag>)}
                    </span>
                  ) },
                  { label: "Revisor", w: "auto", align: "right", render: (m) => (
                    m.isReviewer
                      ? <button type="button" style={buttonStyle} onClick={() => open(m, "revoke")}>Remover revisor</button>
                      : <button type="button" style={buttonStyle} onClick={() => open(m, "grant")}>Tornar revisor</button>
                  ) }
                ]}
                rows={members}
                rowKey={(m) => m.userId}
                empty="nenhuma pessoa com papel ativo"
              />
            )}
          </div>
        </Panel>
      )}

      {pending && (
        pending.kind === "grant" ? (
          <SensitiveAction
            title="Tornar revisor"
            description={`${pending.member.email} poderá assinar publicação e ativação de protocolo.`}
            requiresStepUp
            run={async () => { await grantRole(pending.member.userId, REVIEWER_ROLE); }}
            onDone={() => finish(`${pending.member.email} agora é revisor`)}
            onCancel={() => setPending(null)}
            onGoToSecurity={() => onNavigate("security")}
          />
        ) : (
          <SensitiveAction
            title="Remover revisor"
            description={`${pending.member.email} deixa de assinar protocolos. As assinaturas que já deu continuam valendo.`}
            requiresStepUp
            run={async () => { await revokeMembership(pending.member.reviewerMembershipId!); }}
            onDone={() => finish(`${pending.member.email} não é mais revisor`)}
            onCancel={() => setPending(null)}
            onGoToSecurity={() => onNavigate("security")}
          />
        )
      )}
    </div>
  );
}
```

O `!` em `reviewerMembershipId!` é seguro porque o botão "Remover revisor" só existe quando `isReviewer` é true, e `teamMembers` preenche os dois juntos. Se o `tsconfig` do projeto recusar a não-nulidade assertiva, troque por uma checagem explícita que não renderiza o painel sem o id.

- [ ] **Step 5: Ligar no App**

Em `src/App.tsx`: `import { Team } from "./modules/Team";` e `case "team": return <Team onNavigate={setActive} />;`. O `renderModule` já recebe `setActive` (o Overview usa), então passe o mesmo.

- [ ] **Step 6: Rodar e ver passar**

```bash
npx vitest run src/modules/Team.test.tsx src/shell/modules.test.ts && npm run typecheck && npm test && npm run build
```
Esperado: verde.

- [ ] **Step 7: Verificação no navegador** (stack de dev de pé)

1. Na raiz do monorepo: `docker compose up -d api worker dashboard`.
2. Abra `http://curitiba.localhost:5175/dashboard/` com a conta de `municipal_admin` de dev (está nos READMEs; não invente) e cadastre o autenticador se a conta ainda não tiver.
3. Em Equipe: conceda "revisor" a alguém, informe o código quando pedido, e confira que o papel aparece na linha e que o aviso de menos de 2 revisores some quando o segundo revisor entra.
4. Remova o papel de um deles e confira que o aviso volta.
5. Entre com uma conta que não seja `municipal_admin` e confira que o item Equipe não aparece no menu.

Registre no relatório o que foi visto, sem códigos.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add src/modules/Team.tsx src/modules/Team.test.tsx src/shell/modules.ts src/shell/modules.test.ts src/App.tsx
/opt/homebrew/bin/git commit -m "feat: manage protocol reviewers from the team screen" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Fora desta fatia

- Convidar membro, conceder outros papéis e desativar usuário (spec §10).
- A tela de Protocolos com assinaturas e ciclo — fatia 3.
- Listar pessoas **sem** papel ativo: a API não tem esse endpoint hoje.
- Aviso por e-mail quando alguém ganha ou perde papel privilegiado (pendência de go-live).
