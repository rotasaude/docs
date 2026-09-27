# Módulo 10 — Profissionais (dashboard, fatia 5) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Telas do módulo 10 no dashboard. O admin cadastra perfil, vínculos e turnos. O profissional vê o próprio cadastro e edita nome e contato. A Equipe mostra quem ainda não pode chamar. O Atendimento só oferece chamar e desfecho clínico a quem tem vínculo com a unidade.

**Architecture:** Cliente HTTP novo em `src/lib/api.ts` para as rotas `/professionals/*` (plano do api). Regras puras (CNS, janela do turno e mensagens de erro) em `src/lib/professionals.ts`, testadas sem React. Dois módulos novos (`Professionals`, `MyProfile`) e ajustes em `Team`, `Attendance` e `UnitQueue`. Step-up pelo `SensitiveAction` que já existe.

**Tech Stack:** React + TypeScript, TanStack Query, Vitest + Testing Library, Vite (proxy de dev).

**Spec:** `docs/superpowers/specs/2026-09-27-module-10-professionals-design.md` (§5). Depende do plano `docs/superpowers/plans/2026-09-27-module-10-professionals-api.md` estar mergeado **antes** do merge desta fatia.

## Global Constraints

- Rotas da API: todas sob `/professionals` (uma entrada nova no proxy do Vite).
- Recusas da regra clínica: `missing_role` e `missing_link` (HTTP 403).
- CNS aparece mascarado na tela (`*** **** **** 1234`), inclusive no "Meu perfil".
- Horários no fuso `America/Sao_Paulo` (o mesmo de `src/lib/format.ts`). Sem horário de verão desde 2019, então o deslocamento é fixo `-03:00`.
- Turno: fim ≤ início quer dizer dia seguinte; duração máxima de 24h.
- Abrir e encerrar vínculo passam por `SensitiveAction` com `requiresStepUp`. Perfil e turnos não.
- Testes que dependem de "hoje" fixam o relógio com `vi.setSystemTime`.
- Nunca `git add -A` com o symlink de `node_modules` no worktree; adicione arquivos pelo nome.
- Commits em inglês, Conventional Commits, com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`.

## Ambiente de execução

- Crie o worktree do dashboard a partir de `origin/main`:

  ```bash
  cd apps/dashboard && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add -b feat/mod-10-professionals .claude/mod10 origin/main && ln -s ../../node_modules .claude/mod10/node_modules
  ```

- Testes:

  ```bash
  cd apps/dashboard/.claude/mod10 && npx vitest run <arquivos>
  ```

- Checagem de tipos antes de cada commit:

  ```bash
  cd apps/dashboard/.claude/mod10 && npx tsc --noEmit
  ```

## Review Focus

1. **Plantão 19h–07h lançado num dia e mostrado no seguinte:** a prévia diz "termina em DD/MM às 07:00", e o que vai para a API é o instante do dia seguinte. Teste: Task 1, casos de `shiftWindow`.
2. **Vínculo encerrado com a fila aberta:** o próximo "Chamar" recebe 403 `missing_link`, a tela mostra "Você não tem vínculo com esta unidade" e os botões somem, sem erro genérico. Teste: Task 6.
3. **Profissional com vínculo na UBS mas não na UPA:** ao trocar para a UPA no seletor, os botões clínicos somem; "Saiu sem atendimento" continua para quem tem `citizen_verifier`. Teste: Task 6.
4. **Janela de step-up que fecha entre abrir e confirmar o vínculo:** o `SensitiveAction` pede código de novo e repete a ação uma vez. Teste: Task 4, "sem janela: pede código".
5. **CNS digitado com espaços ou pontos:** aceito se os 15 dígitos forem válidos; recusado na tela, antes de ir para a API, se o dígito verificador errar. Teste: Task 1.

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `vite.config.ts` | proxy de `/professionals` | 1 |
| `src/lib/api.ts` | tipos e chamadas `/professionals/*`; `MembershipRow.professional_status` | 1 |
| `src/lib/professionals.ts` | CNS, `shiftWindow`, mensagens de erro, `COUNCILS`, `UFS` | 1 |
| `src/shell/modules.ts`, `src/App.tsx` | navegação | 2 |
| `src/modules/Professionals.tsx` | lista, pendências e cadastro de perfil | 3 |
| `src/modules/professionals/ProfessionalDetail.tsx` | ficha: perfil, vínculos, turnos | 4 |
| `src/modules/MyProfile.tsx` | "Meu perfil" | 5 |
| `src/modules/Team.tsx`, `src/modules/Attendance.tsx`, `src/modules/attendance/UnitQueue.tsx`, `src/lib/attendance.ts` | etiqueta, `canCare` por vínculo, 403 nomeado | 6 |

---

### Task 1: Cliente da API, proxy e regras puras

**Files:**
- Modify: `vite.config.ts`, `src/lib/api.ts`
- Create: `src/lib/professionals.ts`
- Test: `src/lib/professionals.test.ts`

**Interfaces:**
- Produces (em `src/lib/api.ts`): tipos `Professional`, `ProfessionalLink`, `ProfessionalShift`, `CboEntry`, `PendingProfessional`, `MyProfessional`; funções `listProfessionals`, `listPendingProfessionals`, `getProfessional`, `createProfessional`, `updateProfessional`, `getMyProfessional`, `updateMyProfessional`, `listCbo`, `openProfessionalLink`, `endProfessionalLink`, `listProfessionalShifts`, `scheduleShift`, `cancelShift`; `MembershipRow.professional_status?`.
- Produces (em `src/lib/professionals.ts`): `isValidCns(input) → boolean`, `maskCns(cns) → string`, `shiftWindow(date, start, end) → { startsAt, endsAt, nextDay, tooLong } | null`, `professionalError(err) → string`, `COUNCILS`, `UFS`, `STATUS_LABEL`.

- [ ] **Step 1: Escreva os testes das regras**

```ts
// src/lib/professionals.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import { isValidCns, maskCns, professionalError, shiftWindow } from "./professionals";

describe("isValidCns", () => {
  it.each([ "700000000000005", "100000000000007", "200123456789019", "898000000000002", "712345678901236", "123456789012348" ])(
    "aceita %s", (cns) => expect(isValidCns(cns)).toBe(true)
  );
  it("aceita com espaços e pontos", () => expect(isValidCns("7123 4567 8901 236")).toBe(true));
  it.each([ "712345678901237", "312345678901236", "70000000000000", "" ])("recusa %s", (cns) =>
    expect(isValidCns(cns)).toBe(false));
});

describe("maskCns", () => {
  it("mostra só os 4 últimos", () => expect(maskCns("712345678901236")).toBe("*** **** **** 1236"));
});

describe("shiftWindow", () => {
  it("turno no mesmo dia", () => {
    expect(shiftWindow("2026-10-05", "07:00", "13:00")).toEqual({
      startsAt: "2026-10-05T07:00:00-03:00", endsAt: "2026-10-05T13:00:00-03:00", nextDay: false, tooLong: false
    });
  });
  it("fim antes do início: termina no dia seguinte", () => {
    expect(shiftWindow("2026-10-05", "19:00", "07:00")).toMatchObject({
      endsAt: "2026-10-06T07:00:00-03:00", nextDay: true, tooLong: false
    });
  });
  it("fim igual ao início: 24h, no dia seguinte", () => {
    expect(shiftWindow("2026-10-31", "07:00", "07:00")).toMatchObject({
      endsAt: "2026-11-01T07:00:00-03:00", nextDay: true, tooLong: false
    });
  });
  it("vira o ano", () => {
    expect(shiftWindow("2026-12-31", "19:00", "07:00")?.endsAt).toBe("2027-01-01T07:00:00-03:00");
  });
  it("entrada ilegível: null", () => {
    expect(shiftWindow("", "07:00", "13:00")).toBeNull();
    expect(shiftWindow("2026-10-05", "7h", "13:00")).toBeNull();
  });
});

describe("professionalError", () => {
  const err = (status: number, body: unknown) => new ApiError(status, body, "x");
  it("traduz as recusas nomeadas", () => {
    expect(professionalError(err(422, { error: "council_mismatch" }))).toMatch(/conselho/);
    expect(professionalError(err(409, { error: "shift_overlap", conflict: { unit_name: "UBS Jardim", starts_at: "2026-10-05T10:00:00Z", ends_at: "2026-10-05T16:00:00Z" } })))
      .toBe("conflita com o turno em UBS Jardim, 05/10 07:00–13:00");
    expect(professionalError(err(422, { error: "invalid", fields: [ "cns", "phone" ] }))).toBe("confira: CNS, telefone");
    expect(professionalError(err(403, { error: "missing_link" }))).toBe("Você não tem vínculo com esta unidade");
  });
  it("desconhecido: genérico", () => {
    expect(professionalError(new Error("rede"))).toBe("não foi possível concluir — tente de novo");
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/professionals.test.ts`
Expected: FAIL (módulo inexistente).

- [ ] **Step 3: Proxy e cliente da API**

Em `vite.config.ts`, na lista de `proxy`, depois de `"/attendance"`:

```ts
      "/professionals": proxy(TARGET),
```

Em `src/lib/api.ts`, acrescente `professional_status?: "missing_profile" | "missing_link" | "ok";` à interface `MembershipRow`. No fim do arquivo:

```ts
// Profissionais (módulo 10, ADR 0021; spec 2026-09-27 §4.1). Tudo sob
// /professionals — uma entrada só no proxy de dev.
const PROFESSIONALS_BASE = import.meta.env.VITE_PROFESSIONALS_BASE || "/professionals";

export interface Professional {
  id: string;
  user_id: string;
  email_address: string;
  professional_name: string;
  council: string;
  council_state: string;
  registration_number: string;
  cns_masked: string;
  cns?: string;
  phone?: string | null;
  contact_email?: string | null;
}

export interface ProfessionalLink {
  id: string;
  health_unit_id: string;
  unit_name: string;
  cbo_code: string;
  cbo_title: string | null;
  started_at: string;
  started_by: string;
  ended_at: string | null;
  ended_by: string | null;
}

export interface ProfessionalShift {
  id: string;
  professional_link_id: string;
  unit_name: string;
  starts_at: string;
  ends_at: string;
  cancelled_at: string | null;
  cancel_reason: string | null;
}

export interface CboEntry { code: string; title: string; council: string | null }
export interface PendingProfessional { user_id: string; email_address: string; status: "missing_profile" | "missing_link" }
export interface MyProfessional { professional: Professional; links: ProfessionalLink[]; shifts: ProfessionalShift[] }

export type ProfessionalFields = Partial<Pick<Professional,
  "professional_name" | "council" | "council_state" | "registration_number" | "cns" | "phone" | "contact_email">>;

const post = (body: unknown): RequestInit => ({ method: "POST", body: JSON.stringify(body) });
const id = encodeURIComponent;

export async function listProfessionals(): Promise<(Professional & { links: ProfessionalLink[] })[]> {
  return (await jsonFetch<{ professionals: (Professional & { links: ProfessionalLink[] })[] }>(PROFESSIONALS_BASE)).professionals;
}

export async function listPendingProfessionals(): Promise<PendingProfessional[]> {
  return (await jsonFetch<{ users: PendingProfessional[] }>(`${PROFESSIONALS_BASE}/pending`)).users;
}

export async function getProfessional(professionalId: string): Promise<{ professional: Professional; links: ProfessionalLink[] }> {
  return jsonFetch(`${PROFESSIONALS_BASE}/${id(professionalId)}`);
}

export async function createProfessional(userId: string, fields: ProfessionalFields): Promise<Professional> {
  return (await jsonFetch<{ professional: Professional }>(PROFESSIONALS_BASE, post({ user_id: userId, ...fields }))).professional;
}

export async function updateProfessional(professionalId: string, fields: ProfessionalFields): Promise<Professional> {
  return (await jsonFetch<{ professional: Professional }>(`${PROFESSIONALS_BASE}/${id(professionalId)}`, post(fields))).professional;
}

// 404 no_profile vira null: "seu cadastro ainda não foi feito" é estado, não erro.
export async function getMyProfessional(): Promise<MyProfessional | null> {
  try {
    return await jsonFetch<MyProfessional>(`${PROFESSIONALS_BASE}/me`);
  } catch (err) {
    if (err instanceof ApiError && err.status === 404) return null;
    throw err;
  }
}

export async function updateMyProfessional(fields: Pick<ProfessionalFields, "professional_name" | "phone" | "contact_email">): Promise<Professional> {
  return (await jsonFetch<{ professional: Professional }>(`${PROFESSIONALS_BASE}/me`, post(fields))).professional;
}

export async function listCbo(): Promise<CboEntry[]> {
  return (await jsonFetch<{ cbo: CboEntry[] }>(`${PROFESSIONALS_BASE}/cbo`)).cbo;
}

export async function openProfessionalLink(professionalId: string, healthUnitId: string, cboCode: string): Promise<ProfessionalLink> {
  return (await jsonFetch<{ link: ProfessionalLink }>(`${PROFESSIONALS_BASE}/${id(professionalId)}/links`,
    post({ health_unit_id: healthUnitId, cbo_code: cboCode }))).link;
}

export async function endProfessionalLink(linkId: string): Promise<{ link: ProfessionalLink; cancelled_shift_ids: string[] }> {
  return jsonFetch(`${PROFESSIONALS_BASE}/links/${id(linkId)}/end`, post({}));
}

export async function listProfessionalShifts(professionalId: string, from: string, to: string): Promise<ProfessionalShift[]> {
  const qs = new URLSearchParams({ from, to }).toString();
  return (await jsonFetch<{ shifts: ProfessionalShift[] }>(`${PROFESSIONALS_BASE}/${id(professionalId)}/shifts?${qs}`)).shifts;
}

export async function scheduleShift(linkId: string, startsAt: string, endsAt: string): Promise<ProfessionalShift> {
  return (await jsonFetch<{ shift: ProfessionalShift }>(`${PROFESSIONALS_BASE}/links/${id(linkId)}/shifts`,
    post({ starts_at: startsAt, ends_at: endsAt }))).shift;
}

export async function cancelShift(shiftId: string, reason: string): Promise<ProfessionalShift> {
  return (await jsonFetch<{ shift: ProfessionalShift }>(`${PROFESSIONALS_BASE}/shifts/${id(shiftId)}/cancel`, post({ reason }))).shift;
}
```

Nota: se `id` ou `post` já existirem como nomes no arquivo, renomeie para `pid`/`postJson`.

- [ ] **Step 4: Regras puras**

```ts
// src/lib/professionals.ts
// Regras do módulo 10 sem React (spec 2026-09-27 §5): CNS igual ao api
// (Professionals::Cns), janela do turno e mensagens das recusas.
import { ApiError } from "./api";
import { fmtHourMinute } from "./format";

export const COUNCILS = [ "CRM", "COREN", "CRO", "CRF", "CRP", "CREFITO", "CRN", "CRFa", "CRESS", "CRBM", "CREF", "CRMV" ];
export const UFS = [ "AC", "AL", "AP", "AM", "BA", "CE", "DF", "ES", "GO", "MA", "MT", "MS", "MG", "PA", "PB", "PR", "PE",
  "PI", "RJ", "RN", "RS", "RO", "RR", "SC", "SP", "SE", "TO" ];
export const STATUS_LABEL = { missing_profile: "sem perfil", missing_link: "sem vínculo" } as const;

const digits = (s: string) => s.replace(/\D/g, "");

// 15 dígitos, primeiro em 1, 2, 7, 8 ou 9, soma ponderada (15..1) múltipla de 11.
export function isValidCns(input: string): boolean {
  const d = digits(input);
  if (!/^[12789]\d{14}$/.test(d)) return false;
  return d.split("").reduce((acc, c, i) => acc + Number(c) * (15 - i), 0) % 11 === 0;
}

export function maskCns(cns: string): string {
  return `*** **** **** ${digits(cns).slice(-4)}`;
}

// Sem horário de verão em America/Sao_Paulo desde 2019: deslocamento fixo.
const OFFSET = "-03:00";

function nextDate(date: string): string {
  const d = new Date(`${date}T12:00:00Z`);
  d.setUTCDate(d.getUTCDate() + 1);
  return d.toISOString().slice(0, 10);
}

// Data + horas da tela → instantes da API. Fim ≤ início = dia seguinte (D6);
// 24h exatas (07:00–07:00) são o máximo, então `tooLong` nunca vem daqui —
// fica para a API recusar o que a tela não consegue expressar.
export function shiftWindow(date: string, start: string, end: string):
  { startsAt: string; endsAt: string; nextDay: boolean; tooLong: boolean } | null {
  if (!/^\d{4}-\d{2}-\d{2}$/.test(date) || !/^\d{2}:\d{2}$/.test(start) || !/^\d{2}:\d{2}$/.test(end)) return null;
  const nextDay = end <= start;
  const endDate = nextDay ? nextDate(date) : date;
  return { startsAt: `${date}T${start}:00${OFFSET}`, endsAt: `${endDate}T${end}:00${OFFSET}`, nextDay, tooLong: false };
}

const FIELD_LABEL: Record<string, string> = {
  professional_name: "nome profissional", council: "conselho", council_state: "UF do conselho",
  registration_number: "número do registro", cns: "CNS", phone: "telefone", contact_email: "e-mail de contato"
};

const MESSAGES: Record<string, string> = {
  user_missing_role: "este usuário não tem o papel de profissional de saúde",
  missing_role: "seu papel não permite esta ação",
  missing_link: "Você não tem vínculo com esta unidade",
  already_exists: "este usuário já tem perfil profissional",
  cns_taken: "este CNS já está em outro perfil",
  registration_taken: "este registro de conselho já está em outro perfil",
  invalid_unit: "unidade inválida ou desativada — escolha outra",
  invalid_cbo: "ocupação fora da lista vigente",
  council_mismatch: "a ocupação exige outro conselho profissional que o do perfil",
  already_linked: "já existe vínculo ativo com esta unidade e ocupação",
  already_ended: "este vínculo já foi encerrado",
  link_ended: "o vínculo foi encerrado — recarregue a ficha",
  invalid_shift: "turno inválido: confira data e horas (máximo de 24h, a partir do início do vínculo)",
  reason_required: "informe o motivo do cancelamento",
  reason_too_long: "motivo com mais de 200 caracteres",
  already_cancelled: "este turno já foi cancelado",
  invalid_range: "intervalo de datas inválido",
  forbidden: "seu papel não permite esta ação"
};
const GENERIC = "não foi possível concluir — tente de novo";

function fmtDayMonth(iso: string): string {
  return new Intl.DateTimeFormat("pt-BR", { timeZone: "America/Sao_Paulo", day: "2-digit", month: "2-digit" }).format(new Date(iso));
}

export function professionalError(err: unknown): string {
  if (!(err instanceof ApiError)) return GENERIC;
  const body = (err.body ?? {}) as { error?: string; fields?: string[]; conflict?: { unit_name: string; starts_at: string; ends_at: string } };
  if (body.error === "invalid" && body.fields?.length) {
    return `confira: ${body.fields.map((f) => FIELD_LABEL[f] ?? f).join(", ")}`;
  }
  if (body.error === "field_not_editable") return "há campos que só a administração da cidade pode mudar";
  if (body.error === "shift_overlap" && body.conflict?.unit_name) {
    const c = body.conflict;
    return `conflita com o turno em ${c.unit_name}, ${fmtDayMonth(c.starts_at)} ${fmtHourMinute(c.starts_at)}–${fmtHourMinute(c.ends_at)}`;
  }
  return (body.error && MESSAGES[body.error]) || GENERIC;
}
```

Confira que `fmtHourMinute` existe em `src/lib/format.ts` e devolve `HH:MM` no fuso de São Paulo (a `UnitQueue` já o usa). Se o nome for outro, use o existente.

- [ ] **Step 5: Rode, cheque tipos e faça commit**

Run: `npx vitest run src/lib/professionals.test.ts && npx tsc --noEmit`
Expected: PASS.

```bash
/opt/homebrew/bin/git add vite.config.ts src/lib/api.ts src/lib/professionals.ts src/lib/professionals.test.ts
/opt/homebrew/bin/git commit -m "feat: add professionals API client and CNS and shift window rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 2: Navegação

**Files:**
- Modify: `src/shell/modules.ts`, `src/App.tsx`
- Test: `src/shell/modules.test.ts`

**Interfaces:**
- Produces: `ModuleId` ganha `"professionals"` e `"my-profile"`; `navGroupsFor` mostra "Profissionais" só para `municipal_admin` e "Meu perfil" só para quem tem `health_professional`, e só para usuário que não é operador.

- [ ] **Step 1: Escreva os testes**

Acrescente a `src/shell/modules.test.ts`:

```ts
describe("módulo 10 na navegação", () => {
  const user = (roles: string[], operator = false) => ({ operator, memberships: roles.map((role) => ({ role })) });
  const ids = (u: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(u).flatMap((g) => g.items.map((i) => i.id));

  it("Profissionais só para municipal_admin", () => {
    expect(ids(user([ "municipal_admin" ]))).toContain("professionals");
    expect(ids(user([ "health_professional" ]))).not.toContain("professionals");
  });

  it("Meu perfil só para health_professional que não é operador", () => {
    expect(ids(user([ "health_professional" ]))).toContain("my-profile");
    expect(ids(user([ "municipal_admin" ]))).not.toContain("my-profile");
    expect(ids(user([], true))).not.toContain("my-profile");
  });
});
```

(Se `navGroupsFor` não estiver importado no arquivo de teste, importe-o de `./modules`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/shell/modules.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente**

Em `src/shell/modules.ts`:
- acrescente `| "professionals" | "my-profile"` ao `ModuleId`;
- no grupo `"Equipe"`, acrescente `{ id: "professionals", label: "Profissionais", icon: "✚" }` depois de `team`;
- no grupo `"Conta"`, acrescente `{ id: "my-profile", label: "Meu perfil", icon: "☺" }` depois de `security`;
- em `navGroupsFor`, depois do `filter` por grupo, filtre os itens:

```ts
  const isProfessional = roles.includes("health_professional");
  return NAV_GROUPS.filter((group) => {
    if (group.label === "Conta") return !user?.operator;
    if (group.label === "Equipe") return isAdmin;
    if (group.label === "Atendimento") return canAttend;
    return true;
  }).map((group) => ({
    ...group,
    // Módulo 10: "Meu perfil" é do profissional; sem sessão ainda, some
    // (a API responderia 404 no_profile para quem não é profissional).
    items: group.items.filter((item) => item.id !== "my-profile" || isProfessional)
  }));
```

Em `src/App.tsx`, importe os dois módulos (criados nas Tasks 3 e 5; até lá, crie arquivos com um componente mínimo que devolve `<Placeholder title="Profissionais" />` e `<Placeholder title="Meu perfil" />`, substituídos depois) e acrescente:

```tsx
    case "professionals":  return <Professionals />;
    case "my-profile":     return <MyProfile />;
```

- [ ] **Step 4: Rode, cheque tipos, faça commit**

Run: `npx vitest run src/shell && npx tsc --noEmit`
Expected: PASS.

```bash
/opt/homebrew/bin/git add src/shell/modules.ts src/shell/modules.test.ts src/App.tsx src/modules/Professionals.tsx src/modules/MyProfile.tsx
/opt/homebrew/bin/git commit -m "feat: add professionals and my profile to the dashboard navigation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 3: Módulo Profissionais — lista, pendências e cadastro de perfil

**Files:**
- Modify: `src/modules/Professionals.tsx`
- Create: `src/modules/professionals/ProfileForm.tsx`
- Test: `src/modules/Professionals.test.tsx`

**Interfaces:**
- Consumes: `listProfessionals`, `listPendingProfessionals`, `createProfessional` (Task 1); `professionalError`, `isValidCns`, `COUNCILS`, `UFS`, `STATUS_LABEL`.
- Produces: `ProfileForm({ initial, onSubmit, submitLabel, selfService? })`, reutilizado nas Tasks 4 e 5; `Professionals` abre a ficha (`ProfessionalDetail`, Task 4) ao clicar numa linha.

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/Professionals.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listProfessionals: vi.fn(), listPendingProfessionals: vi.fn(),
    createProfessional: vi.fn(), getProfessional: vi.fn(), listProfessionalShifts: vi.fn(), listCbo: vi.fn(),
    listActiveUnits: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { AuthProvider } from "../lib/auth";
import { Professionals } from "./Professionals";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function renderIt() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  render(<Professionals />, { wrapper });
}

describe("Professionals", () => {
  beforeEach(() => {
    mocked(api.fetchCurrentSession).mockResolvedValue({
      id: "u-admin", email_address: "admin@cidade.gov.br", operator: false, mfa_enrolled: true,
      mfa_verified_at: new Date().toISOString(),
      memberships: [ { municipality_id: "m1", municipality_name: "Curitiba", municipality_uf: "PR", role: "municipal_admin" } ]
    });
    mocked(api.listProfessionals).mockResolvedValue([ {
      id: "p1", user_id: "u1", email_address: "medica@c.gov.br", professional_name: "Helena Duarte", council: "CRM",
      council_state: "PR", registration_number: "12345", cns_masked: "*** **** **** 0005",
      links: [ { id: "l1", health_unit_id: "h1", unit_name: "UBS Jardim", cbo_code: "225125", cbo_title: "Médico clínico",
        started_at: "2026-09-01T12:00:00Z", started_by: "admin@c.gov.br", ended_at: null, ended_by: null } ]
    } ]);
    mocked(api.listPendingProfessionals).mockResolvedValue([
      { user_id: "u2", email_address: "novato@c.gov.br", status: "missing_profile" }
    ]);
  });

  it("lista perfis com vínculos ativos e o painel de pendência", async () => {
    renderIt();
    expect(await screen.findByText("Helena Duarte")).toBeTruthy();
    expect(screen.getByText("CRM-PR 12345")).toBeTruthy();
    expect(screen.getByText("UBS Jardim · Médico clínico")).toBeTruthy();
    expect(screen.getByText("novato@c.gov.br")).toBeTruthy();
    expect(screen.getByText("sem perfil")).toBeTruthy();
  });

  it("cadastra o perfil a partir da pendência", async () => {
    mocked(api.createProfessional).mockResolvedValue({ id: "p2" } as api.Professional);
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar perfil" }));
    fireEvent.change(screen.getByLabelText("Nome profissional"), { target: { value: "Rafael Lima" } });
    fireEvent.change(screen.getByLabelText("Conselho"), { target: { value: "COREN" } });
    fireEvent.change(screen.getByLabelText("UF do conselho"), { target: { value: "PR" } });
    fireEvent.change(screen.getByLabelText("Número do registro"), { target: { value: "54321" } });
    fireEvent.change(screen.getByLabelText("CNS"), { target: { value: "7000 0000 0000 005" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar perfil" }));
    await waitFor(() => expect(api.createProfessional).toHaveBeenCalledWith("u2", expect.objectContaining({
      professional_name: "Rafael Lima", council: "COREN", council_state: "PR", registration_number: "54321",
      cns: "700000000000005"
    })));
  });

  it("CNS inválido é recusado na tela, sem chamar a API", async () => {
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar perfil" }));
    fireEvent.change(screen.getByLabelText("CNS"), { target: { value: "712345678901237" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar perfil" }));
    expect(await screen.findByRole("alert")).toHaveProperty("textContent", expect.stringContaining("CNS"));
    expect(api.createProfessional).not.toHaveBeenCalled();
  });

  it("recusa da API aparece traduzida", async () => {
    mocked(api.createProfessional).mockRejectedValue(new ApiError(409, { error: "cns_taken" }, "x"));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar perfil" }));
    fireEvent.change(screen.getByLabelText("Nome profissional"), { target: { value: "Rafael" } });
    fireEvent.change(screen.getByLabelText("Número do registro"), { target: { value: "1" } });
    fireEvent.change(screen.getByLabelText("CNS"), { target: { value: "700000000000005" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar perfil" }));
    expect(await screen.findByText("este CNS já está em outro perfil")).toBeTruthy();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/Professionals.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Formulário de perfil**

```tsx
// src/modules/professionals/ProfileForm.tsx
import { useState, type FormEvent } from "react";
import type { Professional, ProfessionalFields } from "../../lib/api";
import { COUNCILS, UFS, isValidCns, professionalError } from "../../lib/professionals";
import { buttonStyle, disabledButtonStyle, inputStyle } from "../../components/formStyles";

// Formulário do perfil (spec §5). `selfService` mostra só nome e contato
// (emenda ao ADR 0021): conselho, registro e CNS a prefeitura confere.
interface Props {
  initial?: Partial<Professional>;
  submitLabel: string;
  selfService?: boolean;
  onSubmit(fields: ProfessionalFields): Promise<void>;
}

const onlyDigits = (s: string) => s.replace(/\D/g, "");

export function ProfileForm({ initial = {}, submitLabel, selfService = false, onSubmit }: Props) {
  const [ name, setName ] = useState(initial.professional_name ?? "");
  const [ council, setCouncil ] = useState(initial.council ?? "CRM");
  const [ uf, setUf ] = useState(initial.council_state ?? "PR");
  const [ registration, setRegistration ] = useState(initial.registration_number ?? "");
  const [ cns, setCns ] = useState("");
  const [ phone, setPhone ] = useState(initial.phone ?? "");
  const [ email, setEmail ] = useState(initial.contact_email ?? "");
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const [ saved, setSaved ] = useState(false);

  async function submit(event: FormEvent) {
    event.preventDefault();
    if (busy) return;
    setError(null); setSaved(false);
    const fields: ProfessionalFields = { professional_name: name, phone: phone || null, contact_email: email || null };
    if (!selfService) {
      // CNS em branco na edição = manter o atual (o formulário não o recebe em claro).
      if (cns || !initial.id) {
        if (!isValidCns(cns)) { setError("CNS inválido: confira os 15 dígitos"); return; }
        fields.cns = onlyDigits(cns);
      }
      Object.assign(fields, { council, council_state: uf, registration_number: onlyDigits(registration) });
    }
    setBusy(true);
    try {
      await onSubmit(fields);
      setSaved(true);
    } catch (err) {
      setError(professionalError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <form onSubmit={submit} style={{ display: "flex", flexDirection: "column", gap: 10, maxWidth: 420 }}>
      {error && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{error}</p>}
      {saved && <p role="status" style={{ margin: 0, fontSize: 12.5 }}>Perfil salvo</p>}
      <label style={labelStyle}>Nome profissional
        <input value={name} onChange={(e) => setName(e.target.value)} style={inputStyle} />
      </label>
      {!selfService && (
        <>
          <div style={{ display: "flex", gap: 8 }}>
            <label style={{ ...labelStyle, flex: 1 }}>Conselho
              <select value={council} onChange={(e) => setCouncil(e.target.value)} style={inputStyle}>
                {COUNCILS.map((c) => <option key={c} value={c}>{c}</option>)}
              </select>
            </label>
            <label style={{ ...labelStyle, width: 90 }}>UF do conselho
              <select value={uf} onChange={(e) => setUf(e.target.value)} style={inputStyle}>
                {UFS.map((u) => <option key={u} value={u}>{u}</option>)}
              </select>
            </label>
          </div>
          <label style={labelStyle}>Número do registro
            <input value={registration} onChange={(e) => setRegistration(e.target.value)} style={inputStyle} inputMode="numeric" />
          </label>
          <label style={labelStyle}>CNS
            <input value={cns} onChange={(e) => setCns(e.target.value)} style={inputStyle} inputMode="numeric"
              placeholder={initial.cns_masked ? `${initial.cns_masked} (em branco mantém)` : "15 dígitos"} />
          </label>
        </>
      )}
      <label style={labelStyle}>Telefone profissional
        <input value={phone} onChange={(e) => setPhone(e.target.value)} style={inputStyle} inputMode="tel" />
      </label>
      <label style={labelStyle}>E-mail de contato
        <input value={email} onChange={(e) => setEmail(e.target.value)} style={inputStyle} type="email" />
      </label>
      <div>
        <button type="submit" disabled={busy} style={busy ? disabledButtonStyle : buttonStyle}>{submitLabel}</button>
      </div>
    </form>
  );
}

const labelStyle = { display: "flex", flexDirection: "column" as const, gap: 4, fontSize: 12, color: "var(--ink2)" };
```

- [ ] **Step 4: Módulo**

```tsx
// src/modules/Professionals.tsx
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { createProfessional, listPendingProfessionals, listProfessionals, type PendingProfessional } from "../lib/api";
import { STATUS_LABEL, professionalError } from "../lib/professionals";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { Tag } from "../components/Tag";
import { buttonStyle, secondaryButtonStyle } from "../components/formStyles";
import { ProfileForm } from "./professionals/ProfileForm";
import { ProfessionalDetail } from "./professionals/ProfessionalDetail";

// Profissionais (módulo 10; spec 2026-09-27 §5): só municipal_admin. Lista,
// painel de quem tem o papel sem cadastro completo e ficha por profissional.
export function Professionals() {
  const queryClient = useQueryClient();
  const list = useQuery({ queryKey: [ "professionals" ], queryFn: listProfessionals });
  const pending = useQuery({ queryKey: [ "professionalsPending" ], queryFn: listPendingProfessionals });
  const [ creatingFor, setCreatingFor ] = useState<PendingProfessional | null>(null);
  const [ openId, setOpenId ] = useState<string | null>(null);

  function refresh() {
    void queryClient.invalidateQueries({ queryKey: [ "professionals" ] });
    void queryClient.invalidateQueries({ queryKey: [ "professionalsPending" ] });
  }

  if (openId) {
    return <ProfessionalDetail professionalId={openId} onBack={() => { setOpenId(null); refresh(); }} />;
  }

  const byUser = new Map((list.data ?? []).map((p) => [ p.user_id, p.id ]));

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Profissionais" sub="perfil · vínculos · turnos" />

      {(pending.data?.length ?? 0) > 0 && (
        <Panel title="Com papel, sem cadastro completo" sub="não chamam pacientes até ter perfil e vínculo">
          <DataTable<PendingProfessional>
            cols={[
              { label: "Usuário", w: "2fr", render: (u) => <span className="mono">{u.email_address}</span> },
              { label: "Situação", w: "1fr", render: (u) => <Tag tone="warn">{STATUS_LABEL[u.status]}</Tag> },
              { label: "", w: "auto", align: "right", render: (u) => u.status === "missing_profile"
                ? <button type="button" style={buttonStyle} onClick={() => setCreatingFor(u)}>Cadastrar perfil</button>
                : <button type="button" style={secondaryButtonStyle} onClick={() => setOpenId(byUser.get(u.user_id) ?? null)}>Vincular</button> }
            ]}
            rows={pending.data ?? []}
            rowKey={(u) => u.user_id}
          />
        </Panel>
      )}

      {creatingFor && (
        <Panel title={`Perfil de ${creatingFor.email_address}`} right={
          <button type="button" style={secondaryButtonStyle} onClick={() => setCreatingFor(null)}>Fechar</button>
        }>
          <ProfileForm submitLabel="Salvar perfil" onSubmit={async (fields) => {
            const created = await createProfessional(creatingFor.user_id, fields);
            setCreatingFor(null);
            refresh();
            setOpenId(created.id);
          }} />
        </Panel>
      )}

      <Panel title="Cadastrados">
        {list.error ? <p role="alert">{professionalError(list.error)}</p> : (list.data ?? []).length === 0
          ? <EmptyState title="nenhum profissional cadastrado" />
          : (
            <DataTable
              cols={[
                { label: "Nome", w: "2fr", render: (p) => (
                  <button type="button" onClick={() => setOpenId(p.id)} style={{ ...secondaryButtonStyle, border: "none" }}>
                    {p.professional_name}
                  </button>
                ) },
                { label: "Conselho", w: "1fr", render: (p) => `${p.council}-${p.council_state} ${p.registration_number}` },
                { label: "Vínculos ativos", w: "3fr", render: (p) => (
                  <span style={{ display: "flex", gap: 4, flexWrap: "wrap" }}>
                    {p.links.filter((l) => !l.ended_at).map((l) => <Tag key={l.id} mono={false}>{`${l.unit_name} · ${l.cbo_title ?? l.cbo_code}`}</Tag>)}
                  </span>
                ) }
              ]}
              rows={list.data ?? []}
              rowKey={(p) => p.id}
            />
          )}
      </Panel>
    </div>
  );
}
```

Até a Task 4, crie `src/modules/professionals/ProfessionalDetail.tsx` com `export function ProfessionalDetail(_: { professionalId: string; onBack(): void }) { return null; }`, para compilar.

Confira os nomes aceitos por `Tag tone` (`src/theme/tokens.ts`); se `"warn"` não existir, use o tom de alerta que existir.

- [ ] **Step 5: Rode, cheque tipos, faça commit**

Run: `npx vitest run src/modules/Professionals.test.tsx && npx tsc --noEmit`
Expected: PASS.

```bash
/opt/homebrew/bin/git add src/modules/Professionals.tsx src/modules/Professionals.test.tsx src/modules/professionals/ProfileForm.tsx src/modules/professionals/ProfessionalDetail.tsx
/opt/homebrew/bin/git commit -m "feat: list professionals, show pending registrations and create profiles

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 4: Ficha do profissional — perfil, vínculos e turnos

**Files:**
- Modify: `src/modules/professionals/ProfessionalDetail.tsx`
- Test: `src/modules/professionals/ProfessionalDetail.test.tsx`

**Interfaces:**
- Consumes: `getProfessional`, `updateProfessional`, `listCbo`, `listActiveUnits`, `openProfessionalLink`, `endProfessionalLink`, `listProfessionalShifts`, `scheduleShift`, `cancelShift`; `SensitiveAction`; `ProfileForm`; `shiftWindow`, `professionalError`.

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/professionals/ProfessionalDetail.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), getProfessional: vi.fn(), updateProfessional: vi.fn(),
    listCbo: vi.fn(), listActiveUnits: vi.fn(), openProfessionalLink: vi.fn(), endProfessionalLink: vi.fn(),
    listProfessionalShifts: vi.fn(), scheduleShift: vi.fn(), cancelShift: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { AuthProvider } from "../../lib/auth";
import { ProfessionalDetail } from "./ProfessionalDetail";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

const link = { id: "l1", health_unit_id: "h1", unit_name: "UBS Jardim", cbo_code: "225125", cbo_title: "Médico clínico",
  started_at: "2026-09-01T12:00:00Z", started_by: "admin@c.gov.br", ended_at: null, ended_by: null };

function session(stepped: boolean) {
  return { id: "u-admin", email_address: "admin@c.gov.br", operator: false, mfa_enrolled: true,
    mfa_verified_at: stepped ? new Date().toISOString() : null,
    memberships: [ { municipality_id: "m1", municipality_name: "Curitiba", municipality_uf: "PR", role: "municipal_admin" } ] };
}

function renderIt() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  render(<ProfessionalDetail professionalId="p1" onBack={vi.fn()} />, { wrapper });
}

describe("ProfessionalDetail", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date("2026-10-05T12:00:00-03:00"));
    mocked(api.fetchCurrentSession).mockResolvedValue(session(true));
    mocked(api.getProfessional).mockResolvedValue({
      professional: { id: "p1", user_id: "u1", email_address: "medica@c.gov.br", professional_name: "Helena Duarte",
        council: "CRM", council_state: "PR", registration_number: "12345", cns_masked: "*** **** **** 0005",
        cns: "700000000000005", phone: null, contact_email: null },
      links: [ link ]
    });
    mocked(api.listCbo).mockResolvedValue([ { code: "225124", title: "Médico pediatra", council: "CRM" } ]);
    mocked(api.listActiveUnits).mockResolvedValue([ { id: "h2", name: "UPA Centro", kind: "upa" } ]);
    mocked(api.listProfessionalShifts).mockResolvedValue([]);
  });

  it("mostra o CNS mascarado, nunca em claro", async () => {
    renderIt();
    expect(await screen.findByText("*** **** **** 0005")).toBeTruthy();
    expect(screen.queryByText("700000000000005")).toBeNull();
  });

  it("abre vínculo pelo SensitiveAction com a janela de step-up aberta", async () => {
    mocked(api.openProfessionalLink).mockResolvedValue({ ...link, id: "l2", health_unit_id: "h2", unit_name: "UPA Centro" });
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Abrir vínculo" }));
    fireEvent.change(await screen.findByLabelText("Unidade"), { target: { value: "h2" } });
    fireEvent.change(screen.getByLabelText("Ocupação (CBO)"), { target: { value: "225124" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar vínculo" }));
    await waitFor(() => expect(api.openProfessionalLink).toHaveBeenCalledWith("p1", "h2", "225124"));
  });

  it("sem janela: pede código antes de abrir", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session(false));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Abrir vínculo" }));
    expect(await screen.findByLabelText(/código/i)).toBeTruthy();
  });

  it("encerrar avisa quantos turnos futuros serão cancelados", async () => {
    mocked(api.listProfessionalShifts).mockResolvedValue([
      { id: "s1", professional_link_id: "l1", unit_name: "UBS Jardim", starts_at: "2026-10-06T10:00:00Z",
        ends_at: "2026-10-06T16:00:00Z", cancelled_at: null, cancel_reason: null }
    ]);
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Encerrar" }));
    expect(await screen.findByText(/1 turno futuro será cancelado/)).toBeTruthy();
  });

  it("lança plantão noturno mostrando o dia seguinte e envia os dois instantes", async () => {
    mocked(api.scheduleShift).mockResolvedValue({} as api.ProfessionalShift);
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Lançar turno" }));
    fireEvent.change(screen.getByLabelText("Data"), { target: { value: "2026-10-06" } });
    fireEvent.change(screen.getByLabelText("Início"), { target: { value: "19:00" } });
    fireEvent.change(screen.getByLabelText("Fim"), { target: { value: "07:00" } });
    expect(screen.getByText("termina em 07/10 às 07:00")).toBeTruthy();
    fireEvent.click(screen.getByRole("button", { name: "Salvar turno" }));
    await waitFor(() => expect(api.scheduleShift).toHaveBeenCalledWith("l1", "2026-10-06T19:00:00-03:00", "2026-10-07T07:00:00-03:00"));
  });

  it("sobreposição aparece nomeando o turno em conflito", async () => {
    mocked(api.scheduleShift).mockRejectedValue(new ApiError(409, { error: "shift_overlap",
      conflict: { unit_name: "UPA Centro", starts_at: "2026-10-06T22:00:00Z", ends_at: "2026-10-07T10:00:00Z" } }, "x"));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Lançar turno" }));
    fireEvent.change(screen.getByLabelText("Data"), { target: { value: "2026-10-06" } });
    fireEvent.change(screen.getByLabelText("Início"), { target: { value: "19:00" } });
    fireEvent.change(screen.getByLabelText("Fim"), { target: { value: "07:00" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar turno" }));
    expect(await screen.findByText("conflita com o turno em UPA Centro, 06/10 19:00–07:00")).toBeTruthy();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/professionals/ProfessionalDetail.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente a ficha**

```tsx
// src/modules/professionals/ProfessionalDetail.tsx
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import {
  cancelShift, endProfessionalLink, getProfessional, listActiveUnits, listCbo, listProfessionalShifts,
  openProfessionalLink, scheduleShift, updateProfessional, type ProfessionalLink, type ProfessionalShift
} from "../../lib/api";
import { professionalError, shiftWindow } from "../../lib/professionals";
import { fmtDateTime } from "../../lib/format";
import { PageHeader } from "../../components/PageHeader";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { KeyValue } from "../../components/KeyValue";
import { SensitiveAction } from "../../components/SensitiveAction";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { ProfileForm } from "./ProfileForm";

// Ficha do profissional (spec §5): perfil, vínculos (step-up para abrir e
// encerrar, D5) e turnos por vínculo numa janela de 14 dias (D6, D8).
interface Props { professionalId: string; onBack(): void }

const WINDOW_DAYS = 14;
const dayIso = (d: Date) => new Intl.DateTimeFormat("en-CA", { timeZone: "America/Sao_Paulo" }).format(d);
const addDays = (iso: string, n: number) => { const d = new Date(`${iso}T12:00:00Z`); d.setUTCDate(d.getUTCDate() + n); return d.toISOString().slice(0, 10); };
const ddmm = (iso: string) => `${iso.slice(8, 10)}/${iso.slice(5, 7)}`;

export function ProfessionalDetail({ professionalId, onBack }: Props) {
  const queryClient = useQueryClient();
  const [ from, setFrom ] = useState(() => dayIso(new Date()));
  const to = addDays(from, WINDOW_DAYS);
  const detail = useQuery({ queryKey: [ "professional", professionalId ], queryFn: () => getProfessional(professionalId) });
  const shifts = useQuery({ queryKey: [ "professionalShifts", professionalId, from ],
    queryFn: () => listProfessionalShifts(professionalId, from, to) });
  const [ opening, setOpening ] = useState(false);
  const [ ending, setEnding ] = useState<ProfessionalLink | null>(null);
  const [ scheduling, setScheduling ] = useState<ProfessionalLink | null>(null);
  const [ cancelling, setCancelling ] = useState<ProfessionalShift | null>(null);

  function refresh() {
    void queryClient.invalidateQueries({ queryKey: [ "professional", professionalId ] });
    void queryClient.invalidateQueries({ queryKey: [ "professionalShifts", professionalId ] });
  }

  if (!detail.data) return detail.error ? <p role="alert">{professionalError(detail.error)}</p> : null;
  const { professional, links } = detail.data;
  const active = links.filter((l) => !l.ended_at);
  const now = Date.now();
  const futureCount = (l: ProfessionalLink) =>
    (shifts.data ?? []).filter((s) => s.professional_link_id === l.id && !s.cancelled_at && Date.parse(s.starts_at) > now).length;

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title={professional.professional_name} sub={professional.email_address} />
      <div><button type="button" style={secondaryButtonStyle} onClick={onBack}>← Profissionais</button></div>

      <Panel title="Perfil">
        <div style={{ display: "flex", gap: 24, flexWrap: "wrap", marginBottom: 12 }}>
          <KeyValue k="Conselho" v={`${professional.council}-${professional.council_state} ${professional.registration_number}`} />
          <KeyValue k="CNS" v={professional.cns_masked} />
        </div>
        <ProfileForm initial={professional} submitLabel="Salvar perfil"
          onSubmit={async (fields) => { await updateProfessional(professional.id, fields); refresh(); }} />
      </Panel>

      <Panel title="Vínculos" right={<button type="button" style={buttonStyle} onClick={() => setOpening(true)}>Abrir vínculo</button>}>
        <DataTable<ProfessionalLink>
          cols={[
            { label: "Unidade", w: "2fr", render: (l) => l.unit_name },
            { label: "Ocupação", w: "2fr", render: (l) => `${l.cbo_code} · ${l.cbo_title ?? ""}` },
            { label: "Início", w: "1.5fr", render: (l) => `${fmtDateTime(l.started_at)} · ${l.started_by}` },
            { label: "Fim", w: "1.5fr", render: (l) => l.ended_at ? `${fmtDateTime(l.ended_at)} · ${l.ended_by}` : "ativo" },
            { label: "", w: "auto", align: "right", render: (l) => !l.ended_at && (
              <span style={{ display: "flex", gap: 6 }}>
                <button type="button" style={secondaryButtonStyle} onClick={() => setScheduling(l)}>Lançar turno</button>
                <button type="button" style={secondaryButtonStyle} onClick={() => setEnding(l)}>Encerrar</button>
              </span>
            ) }
          ]}
          rows={links}
          rowKey={(l) => l.id}
          empty="nenhum vínculo"
        />
        {opening && <OpenLink professionalId={professional.id} onDone={() => { setOpening(false); refresh(); }} onCancel={() => setOpening(false)} />}
        {ending && (
          <SensitiveAction
            title={`Encerrar vínculo em ${ending.unit_name}`}
            description={futureCount(ending) > 0
              ? `${futureCount(ending)} ${futureCount(ending) === 1 ? "turno futuro será cancelado" : "turnos futuros serão cancelados"}.`
              : "Nenhum turno futuro neste vínculo."}
            requiresStepUp
            confirmLabel="Confirmar encerramento"
            run={async () => { await endProfessionalLink(ending.id); }}
            translateError={(err) => professionalError(err)}
            onDone={() => { setEnding(null); refresh(); }}
            onCancel={() => setEnding(null)}
          />
        )}
      </Panel>

      <Panel title="Turnos" sub={`${ddmm(from)} a ${ddmm(to)}`} right={
        <span style={{ display: "flex", gap: 6 }}>
          <button type="button" style={secondaryButtonStyle} onClick={() => setFrom(addDays(from, -7))}>← semana</button>
          <button type="button" style={secondaryButtonStyle} onClick={() => setFrom(addDays(from, 7))}>semana →</button>
        </span>
      }>
        {scheduling && <ScheduleShift link={scheduling} onDone={() => { setScheduling(null); refresh(); }} onCancel={() => setScheduling(null)} />}
        <DataTable<ProfessionalShift>
          cols={[
            { label: "Unidade", w: "2fr", render: (s) => s.unit_name },
            { label: "Início", w: "1.5fr", render: (s) => fmtDateTime(s.starts_at) },
            { label: "Fim", w: "1.5fr", render: (s) => fmtDateTime(s.ends_at) },
            { label: "Situação", w: "2fr", render: (s) => s.cancelled_at ? `cancelado — ${s.cancel_reason}` : "válido" },
            { label: "", w: "auto", align: "right", render: (s) => !s.cancelled_at && (
              <button type="button" style={secondaryButtonStyle} onClick={() => setCancelling(s)}>Cancelar</button>
            ) }
          ]}
          rows={shifts.data ?? []}
          rowKey={(s) => s.id}
          empty="nenhum turno no período"
        />
        {cancelling && <CancelShiftPanel shift={cancelling} onDone={() => { setCancelling(null); refresh(); }} onCancel={() => setCancelling(null)} />}
      </Panel>
      {active.length === 0 && <p style={{ fontSize: 12.5 }}>Sem vínculo ativo: este profissional não chama pacientes.</p>}
    </div>
  );
}

function OpenLink({ professionalId, onDone, onCancel }: { professionalId: string; onDone(): void; onCancel(): void }) {
  const units = useQuery({ queryKey: [ "activeUnits" ], queryFn: listActiveUnits });
  const cbo = useQuery({ queryKey: [ "cbo" ], queryFn: listCbo });
  const [ unitId, setUnitId ] = useState("");
  const [ code, setCode ] = useState("");
  const [ search, setSearch ] = useState("");
  const options = (cbo.data ?? []).filter((e) => `${e.code} ${e.title}`.toLowerCase().includes(search.toLowerCase()));

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 10, marginTop: 12 }}>
      <label style={labelStyle}>Unidade
        <select value={unitId} onChange={(e) => setUnitId(e.target.value)} style={inputStyle}>
          <option value="">—</option>
          {(units.data ?? []).map((u) => <option key={u.id} value={u.id}>{u.name}</option>)}
        </select>
      </label>
      <label style={labelStyle}>Buscar ocupação
        <input value={search} onChange={(e) => setSearch(e.target.value)} style={inputStyle} placeholder="código ou título" />
      </label>
      <label style={labelStyle}>Ocupação (CBO)
        <select value={code} onChange={(e) => setCode(e.target.value)} style={inputStyle}>
          <option value="">—</option>
          {options.map((e) => <option key={e.code} value={e.code}>{`${e.code} · ${e.title}`}</option>)}
        </select>
      </label>
      {unitId && code && (
        <SensitiveAction
          title="Abrir vínculo"
          description="Com o vínculo, o profissional passa a chamar e registrar desfecho nesta unidade."
          requiresStepUp
          confirmLabel="Confirmar vínculo"
          run={async () => { await openProfessionalLink(professionalId, unitId, code); }}
          translateError={(err) => professionalError(err)}
          onDone={onDone}
          onCancel={onCancel}
        />
      )}
      {!(unitId && code) && <div><button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button></div>}
    </div>
  );
}

function ScheduleShift({ link, onDone, onCancel }: { link: ProfessionalLink; onDone(): void; onCancel(): void }) {
  const [ date, setDate ] = useState("");
  const [ start, setStart ] = useState("");
  const [ end, setEnd ] = useState("");
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const span = shiftWindow(date, start, end);

  async function save() {
    if (busy || !span) return;
    setBusy(true); setError(null);
    try {
      await scheduleShift(link.id, span.startsAt, span.endsAt);
      onDone();
    } catch (err) {
      setError(professionalError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <section style={{ display: "flex", flexDirection: "column", gap: 10, padding: 12, border: "1px solid var(--rule)", borderRadius: 8, marginBottom: 12 }}>
      <strong>{`Turno em ${link.unit_name}`}</strong>
      {error && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{error}</p>}
      <div style={{ display: "flex", gap: 8 }}>
        <label style={labelStyle}>Data<input type="date" value={date} onChange={(e) => setDate(e.target.value)} style={inputStyle} /></label>
        <label style={labelStyle}>Início<input type="time" value={start} onChange={(e) => setStart(e.target.value)} style={inputStyle} /></label>
        <label style={labelStyle}>Fim<input type="time" value={end} onChange={(e) => setEnd(e.target.value)} style={inputStyle} /></label>
      </div>
      {span?.nextDay && <p style={{ margin: 0, fontSize: 12.5 }}>{`termina em ${ddmm(span.endsAt.slice(0, 10))} às ${end}`}</p>}
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={!span || busy} onClick={() => void save()} style={!span || busy ? disabledButtonStyle : buttonStyle}>Salvar turno</button>
        <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Cancelar</button>
      </div>
    </section>
  );
}

function CancelShiftPanel({ shift, onDone, onCancel }: { shift: ProfessionalShift; onDone(): void; onCancel(): void }) {
  const [ reason, setReason ] = useState("");
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);

  async function confirm() {
    if (busy || !reason.trim()) return;
    setBusy(true); setError(null);
    try {
      await cancelShift(shift.id, reason);
      onDone();
    } catch (err) {
      setError(professionalError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <section style={{ display: "flex", flexDirection: "column", gap: 10, padding: 12, border: "1px solid var(--rule)", borderRadius: 8, marginTop: 12 }}>
      <strong>Cancelar turno</strong>
      {error && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{error}</p>}
      <label style={labelStyle}>Motivo<input value={reason} maxLength={200} onChange={(e) => setReason(e.target.value)} style={inputStyle} /></label>
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={!reason.trim() || busy} onClick={() => void confirm()} style={!reason.trim() || busy ? disabledButtonStyle : buttonStyle}>Confirmar cancelamento</button>
        <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Voltar</button>
      </div>
    </section>
  );
}

const labelStyle = { display: "flex", flexDirection: "column" as const, gap: 4, fontSize: 12, color: "var(--ink2)" };
```

Notas:
- O teste "sobreposição" espera `06/10 19:00–07:00` porque `22:00Z`/`10:00Z` são 19:00 e 07:00 em São Paulo.
- Confira a assinatura real de `SensitiveAction` (`description` aceita `ReactNode`; o rótulo do campo de código pode ser outro — ajuste a regex `/código/i` do teste ao rótulo real).
- Se `fmtDateTime` mostrar segundos, use-o mesmo assim; não crie formatador novo.

- [ ] **Step 4: Rode, cheque tipos, faça commit**

Run: `npx vitest run src/modules/professionals && npx tsc --noEmit`
Expected: PASS.

```bash
/opt/homebrew/bin/git add src/modules/professionals/ProfessionalDetail.tsx src/modules/professionals/ProfessionalDetail.test.tsx
/opt/homebrew/bin/git commit -m "feat: manage a professional's profile, unit links and shifts

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 5: Meu perfil

**Files:**
- Modify: `src/modules/MyProfile.tsx`
- Test: `src/modules/MyProfile.test.tsx`

**Interfaces:**
- Consumes: `getMyProfessional`, `updateMyProfessional`; `ProfileForm` com `selfService`.
- Produces: query key `["myProfessional"]`, reutilizada pela Task 6.

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/MyProfile.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, getMyProfessional: vi.fn(), updateMyProfessional: vi.fn() };
});

import * as api from "../lib/api";
import { MyProfile } from "./MyProfile";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function renderIt() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  render(<MyProfile />, { wrapper: ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider> });
}

describe("MyProfile", () => {
  beforeEach(() => {
    mocked(api.getMyProfessional).mockResolvedValue({
      professional: { id: "p1", user_id: "u1", email_address: "medica@c.gov.br", professional_name: "Helena Duarte",
        council: "CRM", council_state: "PR", registration_number: "12345", cns_masked: "*** **** **** 0005",
        cns: "700000000000005", phone: null, contact_email: null },
      links: [ { id: "l1", health_unit_id: "h1", unit_name: "UBS Jardim", cbo_code: "225125", cbo_title: "Médico clínico",
        started_at: "2026-09-01T12:00:00Z", started_by: "a", ended_at: null, ended_by: null } ],
      shifts: []
    });
  });

  it("mostra dados conferidos só para leitura, com CNS mascarado", async () => {
    renderIt();
    expect(await screen.findByText("CRM-PR 12345")).toBeTruthy();
    expect(screen.getByText("*** **** **** 0005")).toBeTruthy();
    expect(screen.queryByText("700000000000005")).toBeNull();
    expect(screen.getByText("UBS Jardim · Médico clínico")).toBeTruthy();
    expect(screen.queryByLabelText("CNS")).toBeNull();
    expect(screen.queryByLabelText("Conselho")).toBeNull();
  });

  it("edita só nome e contato", async () => {
    mocked(api.updateMyProfessional).mockResolvedValue({} as api.Professional);
    renderIt();
    fireEvent.change(await screen.findByLabelText("Telefone profissional"), { target: { value: "41998765432" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
    await waitFor(() => expect(api.updateMyProfessional).toHaveBeenCalledWith({
      professional_name: "Helena Duarte", phone: "41998765432", contact_email: null
    }));
  });

  it("sem perfil: orienta a procurar a administração", async () => {
    mocked(api.getMyProfessional).mockResolvedValue(null);
    renderIt();
    expect(await screen.findByText("Seu cadastro profissional ainda não foi feito. Fale com a administração da cidade.")).toBeTruthy();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/MyProfile.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

```tsx
// src/modules/MyProfile.tsx
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { getMyProfessional, updateMyProfessional, type ProfessionalLink, type ProfessionalShift } from "../lib/api";
import { professionalError } from "../lib/professionals";
import { fmtDateTime } from "../lib/format";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { KeyValue } from "../components/KeyValue";
import { EmptyState } from "../components/EmptyState";
import { ProfileForm } from "./professionals/ProfileForm";

// Meu perfil (módulo 10; emenda ao ADR 0021): o profissional lê conselho,
// registro, CNS, vínculos e turnos e edita só nome e contato.
export function MyProfile() {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: [ "myProfessional" ], queryFn: getMyProfessional });

  if (query.error) return <p role="alert">{professionalError(query.error)}</p>;
  if (query.isLoading) return null;
  if (!query.data) {
    return (
      <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
        <PageHeader title="Meu perfil" sub="profissional de saúde" />
        <EmptyState title="Seu cadastro profissional ainda não foi feito. Fale com a administração da cidade." />
      </div>
    );
  }

  const { professional, links, shifts } = query.data;
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Meu perfil" sub={professional.email_address} />
      <Panel title="Dados conferidos pela prefeitura">
        <div style={{ display: "flex", gap: 24, flexWrap: "wrap" }}>
          <KeyValue k="Conselho" v={`${professional.council}-${professional.council_state} ${professional.registration_number}`} />
          <KeyValue k="CNS" v={professional.cns_masked} />
        </div>
      </Panel>
      <Panel title="Nome e contato">
        <ProfileForm initial={professional} selfService submitLabel="Salvar" onSubmit={async (fields) => {
          await updateMyProfessional({ professional_name: fields.professional_name, phone: fields.phone ?? null,
            contact_email: fields.contact_email ?? null });
          void queryClient.invalidateQueries({ queryKey: [ "myProfessional" ] });
        }} />
      </Panel>
      <Panel title="Vínculos ativos">
        <DataTable<ProfessionalLink>
          cols={[ { label: "Vínculo", w: "1fr", render: (l) => `${l.unit_name} · ${l.cbo_title ?? l.cbo_code}` } ]}
          rows={links} rowKey={(l) => l.id} empty="nenhum vínculo ativo — você não chama pacientes"
        />
      </Panel>
      <Panel title="Próximos turnos" sub="14 dias">
        <DataTable<ProfessionalShift>
          cols={[
            { label: "Unidade", w: "2fr", render: (s) => s.unit_name },
            { label: "Início", w: "1fr", render: (s) => fmtDateTime(s.starts_at) },
            { label: "Fim", w: "1fr", render: (s) => fmtDateTime(s.ends_at) }
          ]}
          rows={shifts} rowKey={(s) => s.id} empty="nenhum turno lançado"
        />
      </Panel>
    </div>
  );
}
```

- [ ] **Step 4: Rode, cheque tipos, faça commit**

Run: `npx vitest run src/modules/MyProfile.test.tsx && npx tsc --noEmit`
Expected: PASS.

```bash
/opt/homebrew/bin/git add src/modules/MyProfile.tsx src/modules/MyProfile.test.tsx
/opt/homebrew/bin/git commit -m "feat: let professionals read their record and edit name and contact

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 6: Equipe e Atendimento

**Files:**
- Modify: `src/lib/team.ts`, `src/modules/Team.tsx`, `src/modules/Attendance.tsx`, `src/modules/attendance/UnitQueue.tsx`, `src/lib/attendance.ts`
- Test: `src/lib/team.test.ts`, `src/modules/Team.test.tsx`, `src/modules/Attendance.test.tsx`, `src/modules/attendance/UnitQueue.test.tsx`

**Interfaces:**
- Consumes: `MembershipRow.professional_status`, `getMyProfessional` (query key `["myProfessional"]`).
- Produces: `TeamMember.professionalStatus: "missing_profile" | "missing_link" | "ok" | null`; `UnitQueue` ganha a prop `careBlocked?: string | null` (mensagem quando o profissional não tem vínculo com a unidade) e `onClinicalRefused?(): void`.

- [ ] **Step 1: Escreva os testes**

Acrescente a `src/lib/team.test.ts`:

```ts
it("professionalStatus vem da linha de health_professional", () => {
  const rows = [
    { id: "1", user: { id: "u1", email_address: "a@c" }, role: "health_professional", granted_at: "x", professional_status: "missing_link" as const },
    { id: "2", user: { id: "u2", email_address: "b@c" }, role: "viewer", granted_at: "x" }
  ];
  const [ a, b ] = teamMembers(rows);
  expect(a.professionalStatus).toBe("missing_link");
  expect(b.professionalStatus).toBeNull();
});
```

Acrescente a `src/modules/Team.test.tsx` (use o `membership()` e o `renderTeam()` do arquivo):

```tsx
it("marca 'sem vínculo' em quem tem o papel e não pode chamar", async () => {
  mocked(api.listMemberships).mockResolvedValue([
    { ...membership("medica@c.gov.br", "health_professional"), professional_status: "missing_link" }
  ]);
  renderTeam();
  expect(await screen.findByText("sem vínculo")).toBeTruthy();
});
```

Acrescente a `src/modules/attendance/UnitQueue.test.tsx` (reuse os helpers do arquivo para renderizar com uma fila que tem um aguardando):

```tsx
it("careBlocked: esconde chamar e desfecho, mostra a mensagem, mantém 'Saiu sem atendimento'", async () => {
  renderQueue({ canCare: false, careBlocked: "Você não tem vínculo com esta unidade" });
  expect(await screen.findByText("Você não tem vínculo com esta unidade")).toBeTruthy();
  expect(screen.queryByRole("button", { name: "Chamar próximo" })).toBeNull();
});

it("403 missing_link ao chamar: mensagem nomeada e avisa a tela para reler o vínculo", async () => {
  const onClinicalRefused = vi.fn();
  mocked(api.callNext).mockRejectedValue(new ApiError(403, { error: "missing_link" }, "x"));
  renderQueue({ canCare: true, onClinicalRefused });
  fireEvent.click(await screen.findByRole("button", { name: "Chamar próximo" }));
  expect(await screen.findByText("Você não tem vínculo com esta unidade")).toBeTruthy();
  expect(onClinicalRefused).toHaveBeenCalled();
});
```

(Se o arquivo não tiver um `renderQueue(props)`, crie um no topo que renderize `<UnitQueue unit={...} units={[]} {...props} />` com o `QueryClientProvider`, no mesmo formato dos testes já existentes.)

Acrescente a `src/modules/Attendance.test.tsx` (reuse a montagem de sessão do arquivo; mocke `getMyProfessional`):

```tsx
it("profissional sem vínculo com a unidade escolhida: sem ações clínicas", async () => {
  mocked(api.getMyProfessional).mockResolvedValue({ professional: {} as api.Professional, shifts: [],
    links: [ { id: "l1", health_unit_id: "outra", unit_name: "UPA", cbo_code: "225125", cbo_title: null,
      started_at: "x", started_by: "a", ended_at: null, ended_by: null } ] });
  // sessão com health_professional e unidade escolhida "h1" (use o helper do arquivo)
  renderAttendanceWithUnit("h1", [ "health_professional" ]);
  expect(await screen.findByText("Você não tem vínculo com esta unidade")).toBeTruthy();
});
```

Ajuste o nome do helper ao que o arquivo já usa; se não houver um que escolha a unidade, mocke `listActiveUnits` com uma unidade e grave `localStorage` com `currentUnitKey(userId)` antes de renderizar (é o que o `UnitPicker` lê).

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/team.test.ts src/modules/Team.test.tsx src/modules/attendance/UnitQueue.test.tsx src/modules/Attendance.test.tsx`
Expected: FAIL nos casos novos.

- [ ] **Step 3: Implemente**

`src/lib/team.ts`:
- em `TeamMember`, acrescente `professionalStatus: "missing_profile" | "missing_link" | "ok" | null;`;
- no objeto inicial de `teamMembers`, `professionalStatus: null`;
- no ramo `row.role === PROFESSIONAL_ROLE`, acrescente `current.professionalStatus = row.professional_status ?? null;`.

`src/modules/Team.tsx`, na coluna "Papéis", depois do `map` das tags:

```tsx
                      {m.professionalStatus && m.professionalStatus !== "ok" &&
                        <Tag tone="warn">{STATUS_LABEL[m.professionalStatus]}</Tag>}
```

(importe `STATUS_LABEL` de `../lib/professionals`; mesmo tom usado na Task 3.) Se a tela já tiver `onNavigate`, torne a etiqueta um botão que chama `onNavigate("professionals")`.

`src/lib/attendance.ts`, em `MESSAGES`:

```ts
  missing_link: "Você não tem vínculo com esta unidade",
  missing_role: "seu papel não permite esta ação",
```

`src/modules/attendance/UnitQueue.tsx`:
- em `Props`, acrescente `careBlocked?: string | null;` e `onClinicalRefused?(): void;`;
- desestruture as duas props;
- logo depois dos avisos `done`/`actionError`, mostre `{careBlocked && <p role="status" style={{ margin: 0, fontSize: 12.5 }}>{careBlocked}</p>}`;
- em `onCallNext`, `onCall` e no fechamento clínico, no `catch`, antes do `setActionError`:

```tsx
      const code = errorCode(err);
      if (code === "missing_link" || code === "missing_role") onClinicalRefused?.();
```

`src/modules/Attendance.tsx`:

```tsx
  const myProfessional = useQuery({ queryKey: [ "myProfessional" ], queryFn: getMyProfessional, enabled: canCareRole });
```

com `const canCareRole = roles.includes("health_professional");` no lugar do atual `canCare`, e:

```tsx
  // F-10.5: chamar e registrar desfecho exigem vínculo ativo com a unidade
  // escolhida. A API é quem garante; aqui só não se oferece o que ela recusaria.
  const linkedUnitIds = new Set((myProfessional.data?.links ?? []).filter((l) => !l.ended_at).map((l) => l.health_unit_id));
  const canCare = canCareRole && !!unit && linkedUnitIds.has(unit.id);
  const careBlocked = canCareRole && unit && myProfessional.isSuccess && !canCare
    ? "Você não tem vínculo com esta unidade" : null;
```

Troque `(canVerify || canCare)` por `(canVerify || canCareRole)` nas condições de `UnitPicker` e `UnitQueue`, e passe para a `UnitQueue`:

```tsx
careBlocked={careBlocked}
onClinicalRefused={() => void queryClient.invalidateQueries({ queryKey: [ "myProfessional" ] })}
```

(com `const queryClient = useQueryClient();` no topo e o import de `getMyProfessional`). A checagem `if (!canVerify && !canCare && !isAdmin)` passa a usar `canCareRole`.

- [ ] **Step 4: Rode a suíte inteira do dashboard e cheque tipos**

Run: `npx vitest run && npx tsc --noEmit`
Expected: PASS, sem cair nenhum teste que já existia. Se um teste antigo de `Attendance` montava profissional sem mockar `getMyProfessional`, acrescente o mock com um vínculo na unidade do teste (o comportamento que ele provava não muda).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/team.ts src/lib/team.test.ts src/modules/Team.tsx src/modules/Team.test.tsx src/lib/attendance.ts src/modules/Attendance.tsx src/modules/Attendance.test.tsx src/modules/attendance/UnitQueue.tsx src/modules/attendance/UnitQueue.test.tsx
/opt/homebrew/bin/git commit -m "feat: flag incomplete professionals and offer clinical actions only on linked units

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 7: Revisão e prova no navegador

- [ ] **Step 1:** Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec §5, com atenção a:
  - CNS nunca renderizado em claro;
  - `vi.setSystemTime` em todo teste que depende de "hoje";
  - nenhum `git add -A` com o symlink de `node_modules`.
- [ ] **Step 2:** Com o api da fatia 4 rodando em dev (semente aplicada), abra o dashboard e confira, com screenshot:
  - como `admin@curitiba.demo`: a lista, a pendência do `novato@`, abrir e encerrar vínculo com step-up, lançar um 19h–07h;
  - como `profissional@curitiba.demo`: "Meu perfil", e na fila da UBS Jardim das Flores "Chamar próximo" aparece; na UBS Vila Esperança não aparece.
  O login como usuário da cidade é feito pelo usuário; não digite senha nem TOTP.
- [ ] **Step 3:** **Pare.** O merge do dashboard só vem depois do merge do api, e só com autorização explícita do usuário.
