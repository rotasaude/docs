# Módulo 17 — Agenda dos profissionais (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** No canal web do cidadão, o resultado da triagem diz quando um pedido de agendamento foi aberto ("A UBS X vai entrar em contato… Prazo previsto: até DD/MM", com texto próprio quando não há unidade); "Seus agendamentos" mostra tipo, profissional, unidade com endereço e fim do horário, o pedido nascido da triagem com o prazo, e ganha "Não posso nesse horário" (motivo de lista fixa, período preferido, nota opcional de até 200 caracteres), distinto do "Cancelar"; a caixa de avisos passa a mostrar, além das campanhas, o lembrete da véspera do horário confirmado (F-17.5, F-17.7, F-17.8; ADR 0029).

**Architecture:** Tipos e chamadas novas em `src/lib/citizenApi.ts`, com os campos novos normalizados na borda (ausente vira `null`/`false`), como já é feito para `attendance`, `reference_units` e `suggestions`. Datas novas em `src/lib/format.ts` (prazo `AAAA-MM-DD` por texto; hora de fim no fuso da cidade). Um componente novo por bloco: `SchedulingRequestNotice` (resultado) e `RescheduleForm` ("Não posso nesse horário"); `AppointmentsSection` e `NoticesStep` mudam no lugar. Quem decide o que pode ser feito é o api (`can_request_reschedule`); o wpda só esconde o botão quando o horário já começou.

**Tech Stack:** React 18 + TypeScript (strict, `noUnusedLocals`, `noUnusedParameters`), Vite 5, Vitest 2 + Testing Library com jest-dom e user-event. Sem TanStack Query nas telas do cidadão (estado local, como hoje). Nenhuma dependência nova.

**Spec:** `docs/.claude/ciclo2/superpowers/specs/2026-10-05-module-17-scheduling-design.md` (§6 "Cidadão", §7 "wpda", §8 LGPD, §9 testes de front e prova no navegador, §11 rollout). ADR: `docs/.claude/ciclo2/adr/0029.md`. Contrato entre apps (fonte única dos formatos): `docs/.claude/ciclo2/superpowers/plans/2026-10-05-module-17-scheduling-contracts.md` (§2 valores comuns, §5 cidadão). Depende do plano do api do módulo 17 estar mergeado **antes** do merge deste (contrato §7).

## Global Constraints

- Rotas consumidas, nos formatos do contrato §5 com os acréscimos do §8 (erro sempre `{ "error": "<reason>" }`). Campos ausentes só são tolerados por compatibilidade com api anterior ao módulo 17 (rollout), nunca como forma alternativa:
  - `GET /citizen/appointments?citizen_id=` (existente): cada horário ganha `appointment_type_name`, `professional_name`, `unit: { name, address }`, `ends_at`, `can_request_reschedule`; cada pedido ganha `kind` (`return`|`referral`|`triage`), `target_unit_name: string|null`, `appointment_type_name`, `due_on` (§8). Endereço (`unit.address`, `unit_address`) = forma de `reference_units`: `{ street, number, complement, zip }` (§8);
  - `POST /citizen/appointments/:id/reschedule_request` com corpo `{ reason_code, preferred_period, note? }` → 200 `{ "appointment": … }`, como `confirm` e `cancel` (§8); 409 `not_reschedulable`; 422 `invalid_reason_code`, `invalid_period`, `note_too_long`;
  - `GET /citizen/triages/:id` (existente): ganha `scheduling_request: { unit_name: string|null, due_on, appointment_type_name } | null`;
  - `GET /citizen/notices` (existente): cada aviso ganha `kind` (`campaign` | `appointment_reminder`); lembrete = `{ kind, id, appointment_id, appointment_type_name, unit_name, unit_address, scheduled_at, professional_name, read: boolean, cpf_masked }` (§8: `read`, não `read_at`; `cpf_masked` com mais de uma pessoa no celular); `unread_count` conta os lembretes com a regra de `notices_muted`; o id do aviso é opaco e `POST /citizen/notices/:id/read` vale para os dois.
- Valores fixos (contrato §2): `reason_code` ∈ `work | health | transport | other` (rótulos Trabalho, Saúde, Transporte, Outro motivo); `preferred_period` ∈ `morning | afternoon | any` (Manhã, Tarde, Qualquer período). Nota opcional, no máximo 200 caracteres, sem espaços nas pontas; em branco, a chave `note` não vai.
- **"Não posso nesse horário" não é "Cancelar"** (spec §6, ADR 0029): o primeiro devolve o pedido à fila com o mesmo prazo; o segundo (existente, motivo ≥ 10 caracteres) encerra o pedido. Os dois formulários nunca aparecem juntos.
- **O cidadão nunca escolhe a vaga** (ADR 0029, invariante): nenhuma tela do wpda lista vagas ou propõe horário.
- **LGPD** (spec §8): motivo, período e nota só no corpo JSON do POST, montado campo a campo; nunca em URL, query string, `console.*` ou `history.*State`. Nada de `<form>` nativo (um submit GET poria a nota na URL). O campo da nota mostra o aviso de texto congelado (`FROZEN_TEXT_NOTICE`, api#32).
- Datas de calendário (`due_on`) são formatadas por texto (`fmtDayMonth`), **nunca** por `new Date(...)`: `new Date("2026-10-31")` é meia-noite UTC e vira 30/10 no fuso de Brasília. Instantes (`scheduled_at`, `ends_at`) passam por `cityDateFormat`, no fuso da cidade.
- Interface tem ciclo próprio (decisão do usuário): nada de redesign nem renomeação. A seção existente se chama **"Seus agendamentos"** (é o "Meus horários" do pedido); continua com esse título. Use `Screen`, `BigButton`, `Field`, `ErrorText`, `RadioGroup`, `FROZEN_TEXT_NOTICE` de `src/modules/citizen/ui.tsx`, o estilo do bloco `ReferenceUnits` e `formatAddress` de `src/lib/territory.ts`. Texto ≥ 18 px, alvo de toque ≥ 48 px.
- Testes que dependem de data fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(...)` em `beforeEach` (só `Date`: o polling da `ResultStep` e o `findBy` usam `setTimeout` real). Mocks do cliente com `vi.spyOn(citizenApi, ...)`; do `fetch` com `vi.stubGlobal`, como nos testes existentes. A suíte roda com `TZ=America/Sao_Paulo` (`vitest.config.ts`).
- O WhatsApp está descontinuado; o wpda web é o único canal do cidadão.
- Nunca `git add -A` (o worktree tem symlink de `node_modules`); adicione arquivos pelo nome.
- Commits em inglês, Conventional Commits com tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).
- Módulos 15 e 16: o 15 já está em `origin/main` do wpda (`6a5d55c`); o 16 não tem plano para o wpda (mexe em api, dashboard, admin, maintenance e contracts). Os arquivos que o 16 e o 17 disputam (validação presencial, `city_schema.rb`, `adr_pointers_spec.rb`) são do api e do dashboard — nenhum conflito esperado aqui. Se a `main` do wpda andar antes do merge, rebase da branch e suíte inteira de novo; conflito só pode aparecer nos arquivos do "Mapa de arquivos".

## Ambiente de execução

- Crie o worktree a partir de `origin/main` do wpda (topo em 2026-10-05: `6a5d55c fix: harden catalog navigation, suggestion source and consent term retry`):

  ```bash
  cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda
  /opt/homebrew/bin/git fetch origin
  /opt/homebrew/bin/git worktree add .claude/mod17 -b feat/mod-17-scheduling origin/main
  cd .claude/mod17 && [ -e node_modules ] || ln -s ../../node_modules node_modules
  /opt/homebrew/bin/git status -sb && /opt/homebrew/bin/git log --oneline -1
  ```

  Esperado: `## feat/mod-17-scheduling...origin/main` sem mudanças. O `.gitignore` ignora `node_modules` e `/.claude/`; mesmo assim, nunca `git add -A`.

- Todos os comandos abaixo rodam em `apps/wpda/.claude/mod17` (rodar o vitest em `apps/wpda` descobre também os testes dos worktrees em `.claude/` e duplica a contagem):

  ```bash
  npx vitest run <arquivos>    # um ou mais arquivos
  npm test                     # suíte inteira
  npx tsc --noEmit             # tipos, antes de cada commit
  ```

- Anote a contagem da suíte antes da Task 1 (`npm test`, linha `Tests  N passed`; em 2026-10-05, `origin/main` tinha 389 testes em 30 arquivos). Nenhum teste que já existe pode cair sem que a task diga qual e por quê.

## Review Focus

1. **Api anterior ao módulo 17 (rollout, ou api do branch sem algum campo):** nenhuma tela mostra `null`/`undefined`; sem `can_request_reschedule`, "Não posso nesse horário" não aparece; aviso sem `kind` é campanha; resultado sem `scheduling_request` não tem bloco. Testes: normalização na Task 1 (horário, triagem) e na Task 5 (aviso), botão ausente na Task 4, bloco ausente na Task 2.
2. **Pedido da triagem sem unidade de referência (fila "sem unidade"):** o texto diz que a Secretaria vai indicar a unidade, nunca "A null…" nem "na null". Testes: Task 2 (resultado) e Task 3 ("Seus agendamentos").
3. **Prazo no último dia do mês (`due_on: "2026-10-31"`):** aparece "até 31/10", não "30/10" (deslocamento de `new Date` em UTC). Testes: `fmtDayMonth` na Task 1 e o bloco do resultado na Task 2.
4. **Tela aberta depois do início do horário:** o botão "Não posso nesse horário" some pelo relógio, mesmo com `can_request_reschedule: true` vindo de uma leitura antiga; e quando o api recusa (409 `not_reschedulable`), a mensagem própria aparece no formulário, que continua aberto, sem `onDone`. Testes: Task 4 (`AppointmentsSection` com o relógio depois do início; `RescheduleForm` com o 409).
5. **Toque duplo em "Pedir outro horário":** um POST só (o segundo seria recusado com 409 e mostraria erro para quem já conseguiu). Teste: Task 4 (`RescheduleForm`).

## Mapa de arquivos

| Arquivo (`apps/wpda`) | Responsabilidade | Task |
|---|---|---|
| `src/lib/format.ts` (+ `.test.ts`) | `fmtDayMonth`, `fmtWeekdayDateTime`, `fmtHourMinute` | 1 |
| `src/lib/citizenApi.ts` (+ `.test.ts`) | `SchedulingRequest`, campos novos do horário, `requestReschedule` (Task 1); campos novos do pedido (Task 3); avisos com `kind` (Task 5) | 1, 3, 5 |
| `src/modules/citizen/ui.tsx` (+ `ui.test.tsx`) | mensagens dos erros novos | 1 |
| `src/modules/citizen/SchedulingRequestNotice.tsx` (novo) | bloco "Pedido de agendamento" do resultado | 2 |
| `src/modules/citizen/ResultStep.tsx` (+ `.test.tsx`) | lê `scheduling_request` no polling e mostra o bloco | 2 |
| `src/modules/citizen/AppointmentsSection.tsx` (+ `.test.tsx`) | campos novos, pedido da triagem, prazo (Task 3); botão e estado "pediu outro horário" (Task 4) | 3, 4 |
| `src/modules/citizen/RescheduleForm.tsx` (novo, + `.test.tsx`) | formulário "Não posso nesse horário" | 4 |
| `src/modules/citizen/NoticesStep.tsx` (+ `.test.tsx`) | lembrete na lista e no detalhe | 5 |

---

### Task 1: Datas, tipos do horário e do resultado, `requestReschedule` e mensagens

**Files:**
- Modify: `src/lib/format.ts`, `src/lib/format.test.ts`
- Modify: `src/lib/citizenApi.ts`, `src/lib/citizenApi.test.ts`
- Modify: `src/modules/citizen/ui.tsx`, `src/modules/citizen/ui.test.tsx`

**Interfaces:**
- Produces (format): `fmtDayMonth(iso: string | null | undefined): string` (`"2026-10-31"` → `"31/10"`, inválido → `"—"`); `fmtWeekdayDateTime(iso): string` (`"qui., 08/10, 09:00"`); `fmtHourMinute(iso): string` (`"09:20"`), os dois no fuso da cidade e `"—"` para vazio/inválido.
- Produces (citizenApi): `interface SchedulingRequest { unit_name: string | null; due_on: string | null; appointment_type_name: string | null }`; `TriageSummary.scheduling_request?: SchedulingRequest | null` (normalizado para `null`); `interface AppointmentUnit { name: string; address: UnitAddress | null }`; em `Appointment`, opcionais `ends_at`, `appointment_type_name`, `professional_name` (`string | null`), `unit` (`AppointmentUnit | null`), `can_request_reschedule` (`boolean`), normalizados para `null`/`false`; `type RescheduleReasonCode = "work" | "health" | "transport" | "other"`; `type PreferredPeriod = "morning" | "afternoon" | "any"`; `const RESCHEDULE_NOTE_MAX = 200`; `citizenApi.requestReschedule(id: string, r: { reasonCode: RescheduleReasonCode; preferredPeriod: PreferredPeriod; note: string }): Promise<{ appointment: Appointment }>`.
- Produces (ui): `messageFor` com `not_reschedulable`, `invalid_reason_code`, `invalid_period`, `note_too_long`.

- [ ] **Step 1: Escreva os testes de data**

No fim de `src/lib/format.test.ts`, acrescente (e inclua `fmtDayMonth, fmtHourMinute, fmtWeekdayDateTime` no `import` de `./format` da primeira linha):

```ts
// Módulo 17: prazo previsto do pedido (data de calendário) e horário marcado
// com início e fim no fuso da cidade.
describe("datas do módulo 17", () => {
  afterEach(() => setCityTimeZone(null));

  it("fmtDayMonth: AAAA-MM-DD vira DD/MM, sem deslocar o dia pelo fuso", () => {
    expect(fmtDayMonth("2026-10-31")).toBe("31/10");
    expect(fmtDayMonth("2027-01-01")).toBe("01/01");
  });
  it("fmtDayMonth não depende do fuso da cidade", () => {
    setCityTimeZone("America/Manaus");
    expect(fmtDayMonth("2026-10-31")).toBe("31/10");
  });
  it.each([ null, undefined, "", "2026-10-31T00:00:00Z", "31/10/2026" ])("fmtDayMonth(%s) vira —", (v) => {
    expect(fmtDayMonth(v)).toBe("—");
  });

  it("fmtWeekdayDateTime: dia da semana, data e hora no fuso da cidade", () => {
    expect(fmtWeekdayDateTime("2026-10-08T09:00:00-03:00")).toBe("qui., 08/10, 09:00");
  });
  it("fmtHourMinute: só a hora, no fuso da cidade", () => {
    expect(fmtHourMinute("2026-10-08T12:20:00Z")).toBe("09:20");
    setCityTimeZone("America/Manaus");
    expect(fmtHourMinute("2026-10-08T12:20:00Z")).toBe("08:20");
  });
  it.each([ null, undefined, "xxx" ])("%s vira — em fmtHourMinute e fmtWeekdayDateTime", (v) => {
    expect(fmtHourMinute(v)).toBe("—");
    expect(fmtWeekdayDateTime(v)).toBe("—");
  });
});
```

Confira que `afterEach` já está no `import` de `vitest` do arquivo (está: `import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";`).

- [ ] **Step 2: Escreva os testes do cliente**

No fim de `src/lib/citizenApi.test.ts`, acrescente:

```ts
// Módulo 17 (contrato §5): pedido gerado pela triagem, horário com tipo,
// profissional, unidade e fim, e "Não posso nesse horário".
describe("citizenApi — módulo 17 (agenda)", () => {
  const baseTriage = {
    id: "t1", status: "completed", tier: "baixa", priority: 9, created_at: "2026-10-06T12:00:00Z",
    completed_at: "2026-10-06T12:05:00Z", report_url: null, consent_active: true, origin_phone_masked: null
  };

  it("triage() sem scheduling_request (api anterior) normaliza para null", async () => {
    mockFetch(200, baseTriage);
    expect((await citizenApi.triage("t1")).scheduling_request).toBeNull();
  });

  it("triage() com o pedido mantém unidade, prazo e tipo; unidade ausente vira null", async () => {
    const request = { unit_name: "UBS Batel", due_on: "2026-11-05", appointment_type_name: "Consulta médica" };
    mockFetch(200, { ...baseTriage, scheduling_request: request });
    expect((await citizenApi.triage("t1")).scheduling_request).toEqual(request);
    mockFetch(200, { ...baseTriage, scheduling_request: { due_on: "2026-11-05", appointment_type_name: "Consulta médica" } });
    expect((await citizenApi.triage("t1")).scheduling_request).toEqual({
      unit_name: null, due_on: "2026-11-05", appointment_type_name: "Consulta médica"
    });
  });

  it("appointments: sem os campos novos (api anterior), viram null e can_request_reschedule false", async () => {
    mockFetch(200, { appointments: [{
      request: { id: "r1", kind: "return", target_unit_name: "UBS Centro", status: "scheduled" },
      appointment: { id: "a1", scheduled_at: "2026-10-08T12:00:00Z", status: "scheduled" }
    }] });
    const a = (await citizenApi.appointments("p1")).appointments[0].appointment!;
    expect(a.ends_at).toBeNull();
    expect(a.appointment_type_name).toBeNull();
    expect(a.professional_name).toBeNull();
    expect(a.unit).toBeNull();
    expect(a.can_request_reschedule).toBe(false);
  });

  it("appointments: com os campos novos, passam; unidade sem endereço fica com address null", async () => {
    mockFetch(200, { appointments: [{
      request: { id: "r1", kind: "return", target_unit_name: "UBS Batel", status: "scheduled" },
      appointment: {
        id: "a1", scheduled_at: "2026-10-08T12:00:00Z", ends_at: "2026-10-08T12:20:00Z", status: "confirmed",
        appointment_type_name: "Consulta médica", professional_name: "Ana Souza", unit: { name: "UBS Batel" },
        can_request_reschedule: true
      }
    }] });
    const a = (await citizenApi.appointments("p1")).appointments[0].appointment!;
    expect(a.ends_at).toBe("2026-10-08T12:20:00Z");
    expect(a.appointment_type_name).toBe("Consulta médica");
    expect(a.professional_name).toBe("Ana Souza");
    expect(a.unit).toEqual({ name: "UBS Batel", address: null });
    expect(a.can_request_reschedule).toBe(true);
  });

  it("requestReschedule manda motivo, período e a nota sem espaços nas pontas, só no corpo", async () => {
    const fn = mockFetch(200, { appointment: { id: "a1", scheduled_at: "2026-10-08T12:00:00Z", status: "cancelled_by_citizen" } });
    const consoleSpies = [ "log", "info", "warn", "error", "debug" ].map(m => vi.spyOn(console, m as "log"));
    await citizenApi.requestReschedule("a1", { reasonCode: "work", preferredPeriod: "afternoon", note: "  Plantão no trabalho  " });
    const [url, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(url).toBe("/citizen/appointments/a1/reschedule_request");
    expect(url).not.toMatch(/Plant|work|afternoon/);
    expect(init.method).toBe("POST");
    expect(JSON.parse(init.body as string)).toEqual({
      reason_code: "work", preferred_period: "afternoon", note: "Plantão no trabalho"
    });
    for (const spy of consoleSpies) expect(spy).not.toHaveBeenCalled();
    consoleSpies.forEach(s => s.mockRestore());
  });

  it("requestReschedule com nota em branco não manda a chave note", async () => {
    const fn = mockFetch(200, { appointment: { id: "a1", scheduled_at: "2026-10-08T12:00:00Z", status: "cancelled_by_citizen" } });
    await citizenApi.requestReschedule("a1", { reasonCode: "health", preferredPeriod: "any", note: "   " });
    const [, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(JSON.parse(init.body as string)).toEqual({ reason_code: "health", preferred_period: "any" });
  });

  it("requestReschedule recusado vira ApiError com o código", async () => {
    mockFetch(409, { error: "not_reschedulable" });
    await expect(citizenApi.requestReschedule("a1", { reasonCode: "other", preferredPeriod: "morning", note: "" }))
      .rejects.toEqual(new ApiError(409, "not_reschedulable"));
  });
});
```

E no fim de `src/modules/citizen/ui.test.tsx`:

```ts
// Módulo 17 (contrato §5): recusas de "Não posso nesse horário".
describe("messageFor — módulo 17", () => {
  it.each([
    [ "not_reschedulable", "Este horário não pode mais ser trocado por aqui. Fale com a unidade." ],
    [ "invalid_reason_code", "Escolha o motivo." ],
    [ "invalid_period", "Escolha o melhor período." ],
    [ "note_too_long", "Escreva no máximo 200 caracteres." ]
  ])("%s", (code, message) => {
    expect(messageFor(new ApiError(422, code))).toBe(message);
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/lib/format.test.ts src/lib/citizenApi.test.ts src/modules/citizen/ui.test.tsx`
Expected: FAIL — `fmtDayMonth is not a function` (e os outros dois formatadores), `scheduling_request` `undefined` em vez de `null`, `a.ends_at` `undefined`, `citizenApi.requestReschedule is not a function`, e as quatro mensagens caindo no genérico "Algo deu errado. Tente de novo.".

- [ ] **Step 4: Implemente as datas**

No fim de `src/lib/format.ts`:

```ts
// Dia e mês ("31/10") de uma data de calendário AAAA-MM-DD (prazo previsto do
// pedido de agendamento, módulo 17): por texto, nunca por new Date.
export function fmtDayMonth(iso: string | null | undefined): string {
  const m = /^(\d{4})-(\d{2})-(\d{2})$/.exec(iso ?? "");
  return m ? `${m[3]}/${m[2]}` : "—";
}

// Horário marcado ("qui., 08/10, 09:00"), no fuso da cidade.
export function fmtWeekdayDateTime(iso: string | null | undefined): string {
  return format(iso, { weekday: "short", day: "2-digit", month: "2-digit", hour: "2-digit", minute: "2-digit" });
}

// Só a hora ("09:20"), no fuso da cidade: o fim do horário marcado.
export function fmtHourMinute(iso: string | null | undefined): string {
  return format(iso, { hour: "2-digit", minute: "2-digit" });
}
```

- [ ] **Step 5: Implemente os tipos e o cliente**

Em `src/lib/citizenApi.ts`:

(a) Logo depois de `export interface TriageSuggestion { … }` (linha 72), acrescente:

```ts
// Pedido de agendamento gerado pela triagem (módulo 17; contrato §5). null =
// só orientação, resultado urgente ou api anterior. unit_name null = fila
// "sem unidade" da cidade (o bairro não tem unidade de referência).
export interface SchedulingRequest {
  unit_name: string | null;
  due_on: string | null; // AAAA-MM-DD, prazo previsto
  appointment_type_name: string | null;
}
```

(b) Em `TriageSummary`, depois de `suggestions?: TriageSuggestion[];`, acrescente:

```ts
  // Pedido gerado por esta triagem (módulo 17). Ausente numa api anterior:
  // normalizado para null (ver normalizeTriage).
  scheduling_request?: SchedulingRequest | null;
```

(c) Troque a interface `Appointment` inteira por:

```ts
// Unidade do horário (módulo 17): o endereço tem o formato de reference_units.
export interface AppointmentUnit { name: string; address: UnitAddress | null }

export interface Appointment {
  id: string;
  scheduled_at: string;
  status: "scheduled" | "confirmed" | "checked_in" | "cancelled_by_citizen" | "expired" | "no_show" | "moved";
  confirmation_deadline_at?: string | null;
  check_in_available?: boolean;
  // Módulo 17 (contrato §5). Ausentes numa api anterior: normalizados em
  // normalizeAppointment (null / false).
  ends_at?: string | null;
  appointment_type_name?: string | null;
  professional_name?: string | null;
  unit?: AppointmentUnit | null;
  can_request_reschedule?: boolean;
}

// "Não posso nesse horário" (módulo 17; contrato §2).
export type RescheduleReasonCode = "work" | "health" | "transport" | "other";
export type PreferredPeriod = "morning" | "afternoon" | "any";
export const RESCHEDULE_NOTE_MAX = 200;
```

(d) Em `normalizeTriage`, troque o fim da função

```ts
    suggestions: (t.suggestions ?? []).map(s => ({ ...s, summary: s.summary ?? null }))
  };
}
```

por

```ts
    suggestions: (t.suggestions ?? []).map(s => ({ ...s, summary: s.summary ?? null })),
    scheduling_request: t.scheduling_request
      ? {
          unit_name: t.scheduling_request.unit_name ?? null,
          due_on: t.scheduling_request.due_on ?? null,
          appointment_type_name: t.scheduling_request.appointment_type_name ?? null
        }
      : null
  };
}
```

(e) Em `normalizeAppointment`, troque o ramo do horário

```ts
      ? {
          ...item.appointment,
          confirmation_deadline_at: item.appointment.confirmation_deadline_at ?? null,
          check_in_available: item.appointment.check_in_available ?? false
        }
```

por

```ts
      ? {
          ...item.appointment,
          confirmation_deadline_at: item.appointment.confirmation_deadline_at ?? null,
          check_in_available: item.appointment.check_in_available ?? false,
          ends_at: item.appointment.ends_at ?? null,
          appointment_type_name: item.appointment.appointment_type_name ?? null,
          professional_name: item.appointment.professional_name ?? null,
          unit: item.appointment.unit
            ? { name: item.appointment.unit.name, address: item.appointment.unit.address ?? null }
            : null,
          can_request_reschedule: item.appointment.can_request_reschedule === true
        }
```

(f) Em `citizenApi`, logo depois de `issueAppointmentCheckInCode: …,`, acrescente:

```ts
  // "Não posso nesse horário" (contrato §5): cancela o horário e devolve o
  // pedido à fila. Motivo, período e nota só no corpo (nunca em URL ou
  // console); nota em branco não vai. Responde { appointment } (contrato §8),
  // como confirm/cancel; a tela não usa o corpo: relê a lista.
  requestReschedule: (id: string, r: { reasonCode: RescheduleReasonCode; preferredPeriod: PreferredPeriod; note: string }) => {
    const note = r.note.trim();
    return call<{ appointment: Appointment }>("POST", `/appointments/${encodeURIComponent(id)}/reschedule_request`, {
      reason_code: r.reasonCode,
      preferred_period: r.preferredPeriod,
      ...(note ? { note } : {})
    });
  },
```

- [ ] **Step 6: Implemente as mensagens**

Em `src/modules/citizen/ui.tsx`, no objeto `MESSAGES`, troque a última entrada

```ts
  protocol_name_required: "Escolha uma triagem para começar."
};
```

por

```ts
  protocol_name_required: "Escolha uma triagem para começar.",
  // Módulo 17 (contrato §5).
  not_reschedulable: "Este horário não pode mais ser trocado por aqui. Fale com a unidade.",
  invalid_reason_code: "Escolha o motivo.",
  invalid_period: "Escolha o melhor período.",
  note_too_long: "Escreva no máximo 200 caracteres."
};
```

- [ ] **Step 7: Rode e veja passar**

Run: `npx vitest run src/lib/format.test.ts src/lib/citizenApi.test.ts src/modules/citizen/ui.test.tsx && npx tsc --noEmit`
Expected: PASS em todos; `tsc` sem erros.

- [ ] **Step 8: Suíte inteira e commit**

Run: `npm test` — a contagem anotada antes + os testes novos, nenhum caindo.

```bash
/opt/homebrew/bin/git add src/lib/format.ts src/lib/format.test.ts src/lib/citizenApi.ts src/lib/citizenApi.test.ts src/modules/citizen/ui.tsx src/modules/citizen/ui.test.tsx
/opt/homebrew/bin/git status --short
/opt/homebrew/bin/git commit -m "feat: add citizen api fields and reschedule request for professional schedules

Normalizes the triage scheduling_request and the new appointment fields
(type, professional, unit, end time, can_request_reschedule) at the edge,
adds the reschedule request call with a body-only note, calendar day/month
and end time formatters, and messages for the new refusals (ADR 0029).

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```
Expected: `status --short` vazio depois do commit (só os seis arquivos entraram).

---

### Task 2: Pedido de agendamento no resultado da triagem

**Files:**
- Create: `src/modules/citizen/SchedulingRequestNotice.tsx`
- Modify: `src/modules/citizen/ResultStep.tsx`
- Test: `src/modules/citizen/ResultStep.test.tsx`

**Interfaces:**
- Consumes: `SchedulingRequest`, `TriageSummary.scheduling_request` (Task 1), `fmtDayMonth` (Task 1).
- Produces: `schedulingRequestText(r: SchedulingRequest): string`; `SchedulingRequestNotice({ request }: { request: SchedulingRequest | null | undefined })` — `null` = nada renderizado.

- [ ] **Step 1: Escreva os testes**

No fim de `src/modules/citizen/ResultStep.test.tsx`:

```ts
// Módulo 17 (spec §6; ADR 0029): a triagem pode abrir um pedido de
// agendamento; quem marca é a unidade, o cidadão só fica sabendo do prazo.
describe("ResultStep — pedido de agendamento (módulo 17)", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date("2026-10-06T10:00:00-03:00"));
  });
  afterEach(() => vi.useRealTimers());

  const request = { unit_name: "UBS Batel", due_on: "2026-11-05", appointment_type_name: "Consulta médica" };

  it("com unidade: diz o tipo, quem vai entrar em contato e o prazo previsto", async () => {
    vi.spyOn(citizenApi, "triage").mockResolvedValue({ ...summary(null), scheduling_request: request });
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} onStartSuggestion={vi.fn()} />);

    const block = (await screen.findByRole("heading", { name: "Pedido de agendamento" })).closest("section")!;
    expect(within(block).getByText("Consulta médica")).toBeInTheDocument();
    expect(within(block).getByText(
      "A UBS Batel vai entrar em contato para marcar sua consulta. Prazo previsto: até 05/11."
    )).toBeInTheDocument();
  });

  it("sem unidade de referência (fila 'sem unidade'): texto próprio, sem 'null'", async () => {
    vi.spyOn(citizenApi, "triage").mockResolvedValue({ ...summary(null), scheduling_request: { ...request, unit_name: null } });
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} onStartSuggestion={vi.fn()} />);

    const block = (await screen.findByRole("heading", { name: "Pedido de agendamento" })).closest("section")!;
    expect(within(block).getByText(
      "A Secretaria de Saúde vai indicar a unidade, que vai entrar em contato para marcar sua consulta. Prazo previsto: até 05/11."
    )).toBeInTheDocument();
    expect(within(block).queryByText(/null|undefined/)).not.toBeInTheDocument();
  });

  it("prazo no último dia do mês não desloca o dia", async () => {
    vi.spyOn(citizenApi, "triage").mockResolvedValue({ ...summary(null), scheduling_request: { ...request, due_on: "2026-10-31" } });
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} onStartSuggestion={vi.fn()} />);
    expect(await screen.findByText(/Prazo previsto: até 31\/10\./)).toBeInTheDocument();
  });

  it.each([
    [ "só orientação ou resultado urgente", { scheduling_request: null } ],
    [ "api anterior, sem o campo", {} ]
  ])("sem pedido (%s): bloco ausente", async (_label, extra) => {
    const triage = vi.spyOn(citizenApi, "triage").mockResolvedValue({ ...summary(null), ...extra });
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} onStartSuggestion={vi.fn()} />);
    await waitFor(() => expect(triage).toHaveBeenCalled());
    await act(async () => { await Promise.resolve(); });
    expect(screen.queryByRole("heading", { name: "Pedido de agendamento" })).not.toBeInTheDocument();
    expect(screen.queryByText(/Prazo previsto/)).not.toBeInTheDocument();
  });

  it("com o relatório pronto, o pedido fica dentro do cartão, uma vez só", async () => {
    vi.spyOn(citizenApi, "triage").mockResolvedValue({
      ...summary("http://curitiba.localhost/wpda/?token=abc"), scheduling_request: request
    });
    stubReport();
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} onStartSuggestion={vi.fn()} />);

    const card = (await screen.findByText("Procure atendimento hoje")).closest("article")!;
    expect(within(card).getByRole("heading", { name: "Pedido de agendamento" })).toBeInTheDocument();
    expect(screen.getAllByRole("heading", { name: "Pedido de agendamento" })).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/ResultStep.test.tsx`
Expected: FAIL nos testes de "com unidade", "sem unidade", "último dia do mês" e "relatório pronto" (`Unable to find role="heading" and name "Pedido de agendamento"`); os de "bloco ausente" já passam.

- [ ] **Step 3: Crie o bloco**

`src/modules/citizen/SchedulingRequestNotice.tsx`:

```tsx
// src/modules/citizen/SchedulingRequestNotice.tsx
// "Pedido de agendamento" no resultado da triagem (spec 2026-10-05 do módulo
// 17 §6; ADR 0029): quem marca é sempre a unidade; o cidadão nunca escolhe a
// vaga, só fica sabendo de quem vai ligar e do prazo previsto. null (só
// orientação, resultado urgente, api anterior) = bloco ausente.
import type { SchedulingRequest } from "../../lib/citizenApi";
import { fmtDayMonth } from "../../lib/format";

export function schedulingRequestText(r: SchedulingRequest): string {
  const due = r.due_on ? ` Prazo previsto: até ${fmtDayMonth(r.due_on)}.` : "";
  return r.unit_name
    ? `A ${r.unit_name} vai entrar em contato para marcar sua consulta.${due}`
    : `A Secretaria de Saúde vai indicar a unidade, que vai entrar em contato para marcar sua consulta.${due}`;
}

export function SchedulingRequestNotice({ request }: { request: SchedulingRequest | null | undefined }) {
  if (!request) return null;
  return (
    <section aria-labelledby="scheduling-request-title"
      style={{ margin: "0 0 24px", padding: 12, borderRadius: 12, border: "1px solid var(--line, #eee)" }}>
      <h2 id="scheduling-request-title" style={{ fontSize: 20, margin: "0 0 8px" }}>Pedido de agendamento</h2>
      {request.appointment_type_name && (
        <p style={{ fontSize: 18, fontWeight: 600, margin: "0 0 4px" }}>{request.appointment_type_name}</p>
      )}
      <p style={{ fontSize: 18, margin: 0 }}>{schedulingRequestText(request)}</p>
      <p style={{ fontSize: 18, color: "var(--ink2, #555)", margin: "8px 0 0" }}>
        Quando a unidade marcar, o horário aparece em "Seus agendamentos", em Minhas triagens.
      </p>
    </section>
  );
}
```

- [ ] **Step 4: Ligue o bloco na `ResultStep`**

Em `src/modules/citizen/ResultStep.tsx`:

(a) imports — troque

```tsx
import { citizenApi, type ReferenceUnit, type TriageSuggestion } from "../../lib/citizenApi";
import { AlsoRecommended } from "./AlsoRecommended";
```

por

```tsx
import { citizenApi, type ReferenceUnit, type SchedulingRequest, type TriageSuggestion } from "../../lib/citizenApi";
import { AlsoRecommended } from "./AlsoRecommended";
import { SchedulingRequestNotice } from "./SchedulingRequestNotice";
```

(b) estado — depois de `const [suggestions, setSuggestions] = useState<TriageSuggestion[]>([]);` acrescente

```tsx
  const [scheduling, setScheduling] = useState<SchedulingRequest | null>(null);
```

(c) polling — troque

```tsx
          setSuggestions(t.suggestions ?? []);
```

por

```tsx
          setSuggestions(t.suggestions ?? []);
          setScheduling(t.scheduling_request ?? null);
```

(d) render — troque

```tsx
    return <Report token={token}><ReferenceUnits units={units} />{recommended}{actions}</Report>;
```

por

```tsx
    return (
      <Report token={token}>
        <SchedulingRequestNotice request={scheduling} /><ReferenceUnits units={units} />{recommended}{actions}
      </Report>
    );
```

e, no ramo do `Screen`, troque

```tsx
      <ReferenceUnits units={units} />
      {recommended}
```

por

```tsx
      <SchedulingRequestNotice request={scheduling} />
      <ReferenceUnits units={units} />
      {recommended}
```

- [ ] **Step 5: Rode e veja passar**

Run: `npx vitest run src/modules/citizen/ResultStep.test.tsx && npx tsc --noEmit`
Expected: PASS (todos os testes do arquivo, os antigos inclusive); `tsc` sem erros.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/SchedulingRequestNotice.tsx src/modules/citizen/ResultStep.tsx src/modules/citizen/ResultStep.test.tsx
/opt/homebrew/bin/git commit -m "feat: show the appointment request opened by the triage on the citizen result

Says which unit will call and the expected deadline, with its own text
when the neighborhood has no reference unit. No block for guidance-only or
urgent results (ADR 0029).

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: "Seus agendamentos" com tipo, profissional, endereço, fim e o pedido da triagem

**Files:**
- Modify: `src/lib/citizenApi.ts`, `src/lib/citizenApi.test.ts`
- Modify: `src/modules/citizen/AppointmentsSection.tsx`
- Test: `src/modules/citizen/AppointmentsSection.test.tsx`

**Interfaces:**
- Consumes: campos novos de `Appointment`, `fmtDayMonth`, `fmtHourMinute` (Task 1); `formatAddress` de `src/lib/territory.ts`.
- Produces: `AppointmentRequest.kind` aceita `"triage"`; `AppointmentRequest.target_unit_name: string | null`; opcionais `appointment_type_name` e `due_on` (`string | null`), normalizados para `null`. Em `AppointmentsSection.tsx`, as funções internas `when(appointment)`, `unitName(appointment, request)` e o componente interno `AppointmentDetails`, que a Task 4 reaproveita.

- [ ] **Step 1: Escreva os testes**

No `describe("citizenApi — módulo 17 (agenda)")` de `src/lib/citizenApi.test.ts`, acrescente:

```ts
  it("appointments: pedido da triagem sem unidade; tipo e prazo do pedido ausentes viram null", async () => {
    mockFetch(200, { appointments: [
      { request: { id: "r1", kind: "triage", target_unit_name: null, status: "open",
                   appointment_type_name: "Consulta médica", due_on: "2026-11-05" }, appointment: null },
      { request: { id: "r2", kind: "triage", status: "open" }, appointment: null },
      { request: { id: "r3", kind: "return", target_unit_name: "UBS Centro", status: "open" }, appointment: null }
    ] });
    const [withType, bare, ret] = (await citizenApi.appointments("p1")).appointments.map(i => i.request);
    expect(withType.target_unit_name).toBeNull();
    expect(withType.appointment_type_name).toBe("Consulta médica");
    expect(withType.due_on).toBe("2026-11-05");
    expect(bare.target_unit_name).toBeNull();
    expect(ret.appointment_type_name).toBeNull();
    expect(ret.due_on).toBeNull();
  });
```

No fim de `src/modules/citizen/AppointmentsSection.test.tsx`:

```ts
// Módulo 17 (contrato §5): o horário diz tipo, profissional, unidade com
// endereço e fim; o pedido aberto pela triagem diz quem marca e o prazo.
describe("AppointmentsSection — módulo 17", () => {
  beforeEach(() => vi.setSystemTime(new Date("2026-10-06T10:00:00-03:00")));

  const START = "2026-10-08T09:00:00-03:00";
  const DEADLINE = "2026-10-07T09:00:00-03:00";

  it("horário com tipo, profissional, unidade com endereço e fim", async () => {
    mockAppointments([{
      request: { id: "r1", kind: "triage", target_unit_name: "UBS Batel", status: "scheduled", closed_reason: null,
                 reopened_reason: null, appointment_type_name: "Consulta médica", due_on: "2026-11-05" },
      appointment: {
        id: "a1", scheduled_at: START, ends_at: "2026-10-08T09:20:00-03:00", status: "scheduled",
        confirmation_deadline_at: DEADLINE, check_in_available: false,
        appointment_type_name: "Consulta médica", professional_name: "Ana Souza",
        unit: { name: "UBS Batel", address: { street: "Rua Padre Anchieta", number: "1500", complement: null, zip: "80730000" } },
        can_request_reschedule: false
      }
    }]);
    render(<AppointmentsSection citizenId="p1" onCheckIn={vi.fn()} />);
    expect(await screen.findByText(`Agendado: ${fmt(START)} às 09:20 — UBS Batel. Confirme até ${fmt(DEADLINE)}`))
      .toBeInTheDocument();
    expect(screen.getByText("Consulta médica")).toBeInTheDocument();
    expect(screen.getByText("Com Ana Souza")).toBeInTheDocument();
    expect(screen.getByText("Rua Padre Anchieta, 1500 · CEP 80730-000")).toBeInTheDocument();
    // Pedido já com horário: o prazo previsto não aparece mais.
    expect(screen.queryByText(/Prazo previsto/)).not.toBeInTheDocument();
  });

  it("confirmado sem profissional nem endereço: só o que veio, sem 'null'", async () => {
    mockAppointments([{
      request: { id: "r2", kind: "return", target_unit_name: "UBS Centro", status: "scheduled", closed_reason: null, reopened_reason: null },
      appointment: {
        id: "a2", scheduled_at: START, ends_at: "2026-10-08T09:15:00-03:00", status: "confirmed",
        confirmation_deadline_at: null, check_in_available: false, appointment_type_name: "Retorno",
        professional_name: null, unit: { name: "UBS Centro", address: null }, can_request_reschedule: false
      }
    }]);
    render(<AppointmentsSection citizenId="p1" onCheckIn={vi.fn()} />);
    expect(await screen.findByText(`Confirmado: ${fmt(START)} às 09:15 — UBS Centro`)).toBeInTheDocument();
    expect(screen.getByText("Retorno")).toBeInTheDocument();
    expect(screen.queryByText(/^Com /)).not.toBeInTheDocument();
    expect(screen.queryByText(/null|undefined/)).not.toBeInTheDocument();
  });

  it("pedido da triagem com unidade: tipo, quem marca e prazo previsto", async () => {
    mockAppointments([{
      request: { id: "r3", kind: "triage", target_unit_name: "UBS Batel", status: "open", closed_reason: null,
                 reopened_reason: null, appointment_type_name: "Consulta médica", due_on: "2026-11-05" },
      appointment: null
    }]);
    render(<AppointmentsSection citizenId="p1" onCheckIn={vi.fn()} />);
    expect(await screen.findByText(
      "Pedido da triagem: Consulta médica — a UBS Batel vai entrar em contato para marcar o horário"
    )).toBeInTheDocument();
    expect(screen.getByText("Prazo previsto: até 05/11")).toBeInTheDocument();
  });

  it("pedido da triagem sem unidade (fila 'sem unidade'): a Secretaria indica, sem 'null'", async () => {
    mockAppointments([{
      request: { id: "r4", kind: "triage", target_unit_name: null, status: "open", closed_reason: null,
                 reopened_reason: null, appointment_type_name: "Consulta médica", due_on: "2026-10-31" },
      appointment: null
    }]);
    render(<AppointmentsSection citizenId="p1" onCheckIn={vi.fn()} />);
    expect(await screen.findByText(
      "Pedido da triagem: Consulta médica — a Secretaria de Saúde vai indicar a unidade e marcar o horário"
    )).toBeInTheDocument();
    expect(screen.getByText("Prazo previsto: até 31/10")).toBeInTheDocument();
    expect(screen.queryByText(/null|undefined/)).not.toBeInTheDocument();
  });

  it("api anterior: pedido de retorno aberto sem prazo não mostra 'Prazo previsto'", async () => {
    mockAppointments([openReturn]);
    render(<AppointmentsSection citizenId="p1" onCheckIn={vi.fn()} />);
    expect(await screen.findByText("Retorno pedido na UBS Centro — a unidade vai marcar o horário")).toBeInTheDocument();
    expect(screen.queryByText(/Prazo previsto/)).not.toBeInTheDocument();
  });
});
```

(O `beforeEach` do arquivo liga o relógio falso e põe 30/09; este `beforeEach` interno roda depois e só move a data.)

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/citizenApi.test.ts src/modules/citizen/AppointmentsSection.test.tsx`
Expected: FAIL — `target_unit_name` `undefined` em vez de `null` no cliente; na tela, "Agendado: … às 09:20" não existe (o texto antigo não tem o fim), "Com Ana Souza" e o endereço ausentes, e o pedido da triagem cai no texto de encaminhamento ("Encaminhamento para UBS Batel…"). O tipo `kind: "triage"` também falha no `tsc` (rode `npx tsc --noEmit`: erro em `AppointmentsSection.test.tsx`).

- [ ] **Step 3: Tipos do pedido**

Em `src/lib/citizenApi.ts`, troque a interface `AppointmentRequest` inteira por:

```ts
export interface AppointmentRequest {
  id: string;
  // "triage" = pedido aberto pela triagem (módulo 17, ADR 0029).
  kind: "return" | "referral" | "triage";
  // null só no pedido da triagem que está na fila "sem unidade" da cidade.
  target_unit_name: string | null;
  status: "open" | "scheduled" | "closed";
  // Ausentes numa api anterior a este deploy: normalizados na borda (ver
  // normalizeAppointment).
  closed_reason?: "fulfilled" | "citizen_cancelled" | "dismissed" | null;
  reopened_reason?: "expired" | "no_show" | null;
  // Unidade de onde o pedido foi movido (api#29); ausente se nunca mudou.
  moved_from_unit_name?: string | null;
  // Módulo 17 (contrato §8): tipo e prazo previsto (AAAA-MM-DD).
  // Ausentes numa api anterior: normalizados para null.
  appointment_type_name?: string | null;
  due_on?: string | null;
}
```

E em `normalizeAppointment`, troque o ramo do pedido

```ts
    request: {
      ...item.request,
      closed_reason: item.request.closed_reason ?? null,
      reopened_reason: item.request.reopened_reason ?? null
    },
```

por

```ts
    request: {
      ...item.request,
      target_unit_name: item.request.target_unit_name ?? null,
      closed_reason: item.request.closed_reason ?? null,
      reopened_reason: item.request.reopened_reason ?? null,
      appointment_type_name: item.request.appointment_type_name ?? null,
      due_on: item.request.due_on ?? null
    },
```

- [ ] **Step 4: A tela**

Substitua `src/modules/citizen/AppointmentsSection.tsx` inteiro por:

```tsx
// "Seus agendamentos" (spec 2026-09-25-citizen-appointments §6; módulo 17 §6):
// pedidos de retorno, encaminhamento e triagem, e os horários marcados pela
// unidade, acima de "Minhas triagens" em HistoryStep. Só aparece quando há
// algo a mostrar. Os campos do módulo 17 podem faltar (api anterior): a tela
// lê com ?? e nunca escreve null.
import { useEffect, useState } from "react";
import { citizenApi, type Appointment, type AppointmentItem, type AppointmentRequest } from "../../lib/citizenApi";
import { cityDateFormat, fmtDayMonth, fmtHourMinute } from "../../lib/format";
import { formatAddress } from "../../lib/territory";
import { BigButton, ErrorText, FROZEN_TEXT_NOTICE, Field, messageFor } from "./ui";

function fmt(iso: string): string {
  return cityDateFormat({ weekday: "short", day: "2-digit", month: "2-digit", hour: "2-digit", minute: "2-digit" })
    .format(new Date(iso));
}

// Início e, quando o api manda (módulo 17), o fim: "qui., 08/10, 09:00 às 09:20".
function when(appointment: Appointment): string {
  return appointment.ends_at
    ? `${fmt(appointment.scheduled_at)} às ${fmtHourMinute(appointment.ends_at)}`
    : fmt(appointment.scheduled_at);
}

// O horário diz a própria unidade (módulo 17); numa api anterior, vale a do pedido.
function unitName(appointment: Appointment, request: AppointmentRequest): string {
  return appointment.unit?.name ?? request.target_unit_name ?? "unidade de saúde";
}

// Só pedido sem horário nenhum chega aqui: um pedido reaberto sempre traz o
// último horário (expired/no_show), que já diz "pode marcar outro horário".
function openRequestText(request: AppointmentRequest): string {
  if (request.kind === "triage") {
    const what = request.appointment_type_name ?? "Consulta";
    return request.target_unit_name
      ? `Pedido da triagem: ${what} — a ${request.target_unit_name} vai entrar em contato para marcar o horário`
      : `Pedido da triagem: ${what} — a Secretaria de Saúde vai indicar a unidade e marcar o horário`;
  }
  return request.kind === "return"
    ? `Retorno pedido na ${request.target_unit_name} — a unidade vai marcar o horário`
    : `Encaminhamento para ${request.target_unit_name} — a unidade vai marcar o horário`;
}

// O job de expiração roda a cada 15 min; até lá o horário ainda vem
// "scheduled", mas confirmar já seria recusado (confirmation_closed).
function confirmationClosed(appointment: Appointment): boolean {
  return !!appointment.confirmation_deadline_at && Date.now() >= new Date(appointment.confirmation_deadline_at).getTime();
}

function finalStatusText(appointment: Appointment, unit: string): string | null {
  switch (appointment.status) {
    case "cancelled_by_citizen": return "Cancelado por você";
    case "expired": return "Cancelado: sem confirmação no prazo — a unidade pode marcar outro horário";
    case "no_show": return "Você não compareceu — a unidade pode marcar outro horário";
    case "checked_in": return `Atendido na ${unit}`;
    default: return null;
  }
}

// Tipo, profissional e endereço (módulo 17): uma linha por dado presente.
function AppointmentDetails({ appointment }: { appointment: Appointment }) {
  const lines = [
    appointment.appointment_type_name ?? null,
    appointment.professional_name ? `Com ${appointment.professional_name}` : null,
    formatAddress(appointment.unit?.address)
  ].filter((line): line is string => !!line);
  if (lines.length === 0) return null;
  return (
    <>
      {lines.map(line => (
        <p key={line} style={{ fontSize: 18, margin: "0 0 4px", color: "var(--ink2, #555)" }}>{line}</p>
      ))}
    </>
  );
}

function AppointmentRow({ item, onReload, onCheckIn }:
  { item: AppointmentItem; onReload: () => void; onCheckIn: (appointmentId: string) => void }) {
  const { request, appointment } = item;
  const [cancelling, setCancelling] = useState(false);
  const [reason, setReason] = useState("");
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function confirm() {
    if (!appointment) return;
    setBusy(true);
    setError(null);
    try {
      await citizenApi.confirmAppointment(appointment.id);
      onReload();
    } catch (e) {
      setError(messageFor(e));
    } finally {
      setBusy(false);
    }
  }

  async function cancel() {
    if (!appointment) return;
    setBusy(true);
    setError(null);
    try {
      await citizenApi.cancelAppointment(appointment.id, reason);
      setCancelling(false);
      setReason("");
      onReload();
    } catch (e) {
      setError(messageFor(e));
    } finally {
      setBusy(false);
    }
  }

  const final = appointment ? finalStatusText(appointment, unitName(appointment, request)) : null;

  // A unidade só dispensa pedido aberto — e um pedido reaberto sempre traz o
  // último horário (expired/no_show). Dispensado vale mais que esse horário:
  // "pode marcar outro horário" deixou de ser verdade.
  if (request.status === "closed" && request.closed_reason === "dismissed") {
    return (
      <li style={{ border: "1px solid var(--rule2, #ccc)", borderRadius: 12, padding: 12 }}>
        <p style={{ fontSize: 18 }}>Pedido encerrado pela unidade</p>
      </li>
    );
  }

  return (
    <li style={{ border: "1px solid var(--rule2, #ccc)", borderRadius: 12, padding: 12 }}>
      {error && <ErrorText>{error}</ErrorText>}

      {request.moved_from_unit_name && (
        <p style={{ fontSize: 16, fontWeight: 600, margin: "0 0 8px" }}>
          {`Local alterado: este atendimento passou da ${request.moved_from_unit_name} para a ${request.target_unit_name}.`}
        </p>
      )}

      {!appointment && request.status === "open" && (
        <p style={{ fontSize: 18 }}>{openRequestText(request)}</p>
      )}

      {appointment && appointment.status === "scheduled" && (
        <>
          <p style={{ fontSize: 18 }}>
            Agendado: {when(appointment)} — {unitName(appointment, request)}.
            {appointment.confirmation_deadline_at && ` Confirme até ${fmt(appointment.confirmation_deadline_at)}`}
          </p>
          <AppointmentDetails appointment={appointment} />
          {confirmationClosed(appointment) && (
            <p style={{ fontSize: 18 }}>O prazo para confirmar terminou. A unidade pode marcar outro horário.</p>
          )}
          {!cancelling && (
            <div style={{ display: "grid", gap: 8 }}>
              {!confirmationClosed(appointment) && <BigButton onClick={confirm} disabled={busy}>Confirmar</BigButton>}
              <BigButton variant="secondary" onClick={() => setCancelling(true)} disabled={busy}>Cancelar</BigButton>
            </div>
          )}
        </>
      )}

      {appointment && appointment.status === "confirmed" && (
        <>
          <p style={{ fontSize: 18 }}>
            Confirmado: {when(appointment)} — {unitName(appointment, request)}
          </p>
          <AppointmentDetails appointment={appointment} />
          {!cancelling && (
            <div style={{ display: "grid", gap: 8 }}>
              {appointment.check_in_available && (
                <BigButton onClick={() => onCheckIn(appointment.id)}>Cheguei na unidade</BigButton>
              )}
              <BigButton variant="secondary" onClick={() => setCancelling(true)} disabled={busy}>Cancelar</BigButton>
            </div>
          )}
        </>
      )}

      {cancelling && appointment && (appointment.status === "scheduled" || appointment.status === "confirmed") && (
        <div style={{ display: "grid", gap: 8, marginTop: 8 }}>
          <Field label="Motivo do cancelamento" value={reason} onChange={e => setReason(e.target.value)}
            hint={FROZEN_TEXT_NOTICE} />
          <BigButton variant="danger" onClick={cancel} disabled={busy || reason.trim().length < 10}>
            Cancelar agendamento
          </BigButton>
        </div>
      )}

      {final && <p style={{ fontSize: 18 }}>{final}</p>}

      {request.status === "open" && request.due_on && (
        <p style={{ fontSize: 18 }}>{`Prazo previsto: até ${fmtDayMonth(request.due_on)}`}</p>
      )}
    </li>
  );
}

export function AppointmentsSection({ citizenId, onCheckIn }:
  { citizenId: string; onCheckIn: (appointmentId: string) => void }) {
  const [items, setItems] = useState<AppointmentItem[] | null>(null);
  const [error, setError] = useState<string | null>(null);

  function load() {
    citizenApi.appointments(citizenId).then(d => setItems(d.appointments)).catch(e => setError(messageFor(e)));
  }
  useEffect(load, [citizenId]);

  if (error) return <ErrorText>{error}</ErrorText>;
  if (!items || items.length === 0) return null;

  return (
    <section style={{ marginBottom: 24 }}>
      <h2 style={{ fontSize: 20 }}>Seus agendamentos</h2>
      <ul style={{ listStyle: "none", padding: 0, display: "grid", gap: 12 }}>
        {items.map(item => (
          <AppointmentRow key={item.request.id} item={item} onReload={load} onCheckIn={onCheckIn} />
        ))}
      </ul>
    </section>
  );
}
```

- [ ] **Step 5: Rode e veja passar**

Run: `npx vitest run src/lib/citizenApi.test.ts src/modules/citizen/AppointmentsSection.test.tsx src/modules/citizen/PendingConfirmations.test.tsx src/modules/citizen/triage.test.tsx && npx tsc --noEmit`
Expected: PASS em todos, os testes antigos de `AppointmentsSection` inclusive (sem `ends_at`, o texto "Agendado: … — UBS Centro. Confirme até …" é o mesmo de antes); `tsc` sem erros.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/lib/citizenApi.ts src/lib/citizenApi.test.ts src/modules/citizen/AppointmentsSection.tsx src/modules/citizen/AppointmentsSection.test.tsx
/opt/homebrew/bin/git commit -m "feat: show appointment type, professional, unit address and end time in citizen appointments

Triage requests show who will call and the expected deadline, with their
own text while the request has no unit yet. Older api responses keep the
previous texts (ADR 0029).

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: "Não posso nesse horário"

**Files:**
- Create: `src/modules/citizen/RescheduleForm.tsx`, `src/modules/citizen/RescheduleForm.test.tsx`
- Modify: `src/modules/citizen/AppointmentsSection.tsx`
- Test: `src/modules/citizen/AppointmentsSection.test.tsx`

**Interfaces:**
- Consumes: `citizenApi.requestReschedule`, `RescheduleReasonCode`, `PreferredPeriod`, `RESCHEDULE_NOTE_MAX` (Task 1); `RadioGroup`, `Field`, `BigButton`, `ErrorText`, `FROZEN_TEXT_NOTICE`, `messageFor` (`ui.tsx`); `AppointmentsSection.tsx` da Task 3.
- Produces: `RescheduleForm({ appointmentId, onDone, onClose }: { appointmentId: string; onDone: () => void; onClose: () => void })`; `RESCHEDULE_INTRO: string`; `noteLength(note: string): number` (pontos de código depois do `trim`); em `AppointmentsSection.tsx`, `export const RESCHEDULE_REQUESTED = "Você pediu outro horário. A unidade vai marcar um novo."`.

- [ ] **Step 1: Testes do formulário**

`src/modules/citizen/RescheduleForm.test.tsx`:

```tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { RescheduleForm, RESCHEDULE_INTRO, noteLength } from "./RescheduleForm";
import { FROZEN_TEXT_NOTICE } from "./ui";
import { citizenApi, ApiError } from "../../lib/citizenApi";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-06T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const rescheduled = { appointment: { id: "a1", scheduled_at: "2026-10-08T12:00:00Z", status: "cancelled_by_citizen" as const } };

function setup() {
  const onDone = vi.fn();
  const onClose = vi.fn();
  render(<RescheduleForm appointmentId="a1" onDone={onDone} onClose={onClose} />);
  return { onDone, onClose, submit: () => screen.getByRole("button", { name: "Pedir outro horário" }) };
}

async function choose(reason: string, period: string) {
  await userEvent.click(screen.getByRole("radio", { name: reason }));
  await userEvent.click(screen.getByRole("radio", { name: period }));
}

describe("RescheduleForm", () => {
  it("explica o que acontece e só libera o envio com motivo e período", async () => {
    const { submit } = setup();
    expect(screen.getByText(RESCHEDULE_INTRO)).toBeInTheDocument();
    for (const label of [ "Trabalho", "Saúde", "Transporte", "Outro motivo", "Manhã", "Tarde", "Qualquer período" ]) {
      expect(screen.getByRole("radio", { name: label })).toBeInTheDocument();
    }
    expect(submit()).toBeDisabled();
    await userEvent.click(screen.getByRole("radio", { name: "Trabalho" }));
    expect(submit()).toBeDisabled();
    await userEvent.click(screen.getByRole("radio", { name: "Tarde" }));
    expect(submit()).toBeEnabled();
  });

  it("envia motivo, período e nota e chama onDone", async () => {
    const spy = vi.spyOn(citizenApi, "requestReschedule").mockResolvedValue(rescheduled);
    const { onDone, submit } = setup();
    await choose("Trabalho", "Tarde");
    await userEvent.type(screen.getByLabelText("Quer explicar? (opcional)"), "Plantão");
    await userEvent.click(submit());
    expect(spy).toHaveBeenCalledWith("a1", { reasonCode: "work", preferredPeriod: "afternoon", note: "Plantão" });
    await waitFor(() => expect(onDone).toHaveBeenCalledTimes(1));
  });

  it("nota opcional: sem nota, envia com a nota vazia (o cliente não manda a chave)", async () => {
    const spy = vi.spyOn(citizenApi, "requestReschedule").mockResolvedValue(rescheduled);
    const { submit } = setup();
    await choose("Transporte", "Qualquer período");
    await userEvent.click(submit());
    expect(spy).toHaveBeenCalledWith("a1", { reasonCode: "transport", preferredPeriod: "any", note: "" });
  });

  it("a nota mostra o aviso de texto congelado e o contador até 200", async () => {
    setup();
    const field = screen.getByLabelText("Quer explicar? (opcional)");
    expect(field).toHaveAccessibleDescription(FROZEN_TEXT_NOTICE);
    expect(field).toHaveAttribute("maxLength", "200");
    expect(screen.getByText("0/200")).toBeInTheDocument();
    await userEvent.type(field, "Plantão");
    expect(screen.getByText("7/200")).toBeInTheDocument();
  });

  it("409 not_reschedulable: mensagem própria, formulário segue aberto, sem onDone", async () => {
    vi.spyOn(citizenApi, "requestReschedule").mockRejectedValue(new ApiError(409, "not_reschedulable"));
    const { onDone, submit } = setup();
    await choose("Saúde", "Manhã");
    await userEvent.click(submit());
    expect(await screen.findByText("Este horário não pode mais ser trocado por aqui. Fale com a unidade.")).toBeInTheDocument();
    expect(submit()).toBeEnabled();
    expect(onDone).not.toHaveBeenCalled();
  });

  it("toque duplo em 'Pedir outro horário' manda um POST só", async () => {
    const spy = vi.spyOn(citizenApi, "requestReschedule").mockReturnValue(new Promise(() => {}));
    const { submit } = setup();
    await choose("Outro motivo", "Tarde");
    const button = submit();
    await userEvent.click(button);
    await userEvent.click(button);
    expect(spy).toHaveBeenCalledTimes(1);
    expect(button).toBeDisabled();
  });

  it("Voltar fecha sem enviar", async () => {
    const spy = vi.spyOn(citizenApi, "requestReschedule");
    const { onClose } = setup();
    await userEvent.click(screen.getByRole("button", { name: "Voltar" }));
    expect(onClose).toHaveBeenCalledTimes(1);
    expect(spy).not.toHaveBeenCalled();
  });

  it("botões com alvo de toque >= 48 px e texto >= 18 px", () => {
    const { submit } = setup();
    for (const b of [ submit(), screen.getByRole("button", { name: "Voltar" }) ]) {
      expect(parseInt(b.style.minHeight, 10)).toBeGreaterThanOrEqual(48);
      expect(parseInt(b.style.fontSize, 10)).toBeGreaterThanOrEqual(18);
    }
  });
});

describe("noteLength", () => {
  it("conta depois do trim e por ponto de código (como o api)", () => {
    expect(noteLength("  Plantão  ")).toBe(7);
    expect(noteLength("ok 👍")).toBe(4);
    expect(noteLength("   ")).toBe(0);
  });
});
```

- [ ] **Step 2: Testes da lista**

No fim de `src/modules/citizen/AppointmentsSection.test.tsx` (e acrescente `RESCHEDULE_REQUESTED` ao `import { AppointmentsSection } from "./AppointmentsSection";`, que passa a `import { AppointmentsSection, RESCHEDULE_REQUESTED } from "./AppointmentsSection";`):

```ts
// Módulo 17 (spec §6; ADR 0029): "Não posso nesse horário" devolve o pedido à
// fila com o mesmo prazo — não é o "Cancelar", que encerra o pedido.
describe("AppointmentsSection — 'Não posso nesse horário'", () => {
  beforeEach(() => vi.setSystemTime(new Date("2026-10-06T10:00:00-03:00")));

  const START = "2026-10-08T09:00:00-03:00";

  function item(over: Partial<NonNullable<AppointmentItem["appointment"]>> = {}): AppointmentItem {
    return {
      request: { id: "r30", kind: "triage", target_unit_name: "UBS Batel", status: "scheduled", closed_reason: null,
                 reopened_reason: null, appointment_type_name: "Consulta médica", due_on: "2026-11-05" },
      appointment: {
        id: "a30", scheduled_at: START, ends_at: "2026-10-08T09:20:00-03:00", status: "confirmed",
        confirmation_deadline_at: null, check_in_available: false, appointment_type_name: "Consulta médica",
        professional_name: "Ana Souza", unit: { name: "UBS Batel", address: null }, can_request_reschedule: true,
        ...over
      }
    };
  }

  it("aparece em 'confirmed' e em 'scheduled', ao lado do Cancelar", async () => {
    mockAppointments([ item(), { ...item({ id: "a31", status: "scheduled", confirmation_deadline_at: "2026-10-07T09:00:00-03:00" }),
      request: { ...item().request, id: "r31" } } ]);
    render(<AppointmentsSection citizenId="p1" onCheckIn={vi.fn()} />);
    await screen.findByText(/Confirmado:/);
    expect(screen.getAllByRole("button", { name: "Não posso nesse horário" })).toHaveLength(2);
    expect(screen.getAllByRole("button", { name: "Cancelar" })).toHaveLength(2);
  });

  it.each([
    [ "api anterior, sem o campo", { can_request_reschedule: undefined } ],
    [ "api diz que não pode", { can_request_reschedule: false } ]
  ])("ausente quando %s", async (_label, over) => {
    mockAppointments([ item(over) ]);
    render(<AppointmentsSection citizenId="p1" onCheckIn={vi.fn()} />);
    await screen.findByText(/Confirmado:/);
    expect(screen.queryByRole("button", { name: "Não posso nesse horário" })).not.toBeInTheDocument();
    expect(screen.getByRole("button", { name: "Cancelar" })).toBeInTheDocument();
  });

  it("ausente quando o horário já começou, mesmo com can_request_reschedule de uma leitura antiga", async () => {
    vi.setSystemTime(new Date("2026-10-08T09:05:00-03:00"));
    mockAppointments([ item() ]);
    render(<AppointmentsSection citizenId="p1" onCheckIn={vi.fn()} />);
    await screen.findByText(/Confirmado:/);
    expect(screen.queryByRole("button", { name: "Não posso nesse horário" })).not.toBeInTheDocument();
  });

  it("os dois formulários nunca aparecem juntos", async () => {
    mockAppointments([ item() ]);
    render(<AppointmentsSection citizenId="p1" onCheckIn={vi.fn()} />);
    await userEvent.click(await screen.findByRole("button", { name: "Não posso nesse horário" }));
    expect(screen.getByRole("button", { name: "Pedir outro horário" })).toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Cancelar" })).not.toBeInTheDocument();
    expect(screen.queryByLabelText("Motivo do cancelamento")).not.toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Voltar" }));
    await userEvent.click(screen.getByRole("button", { name: "Cancelar" }));
    expect(screen.getByLabelText("Motivo do cancelamento")).toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Não posso nesse horário" })).not.toBeInTheDocument();
  });

  it("pedido enviado: relê a lista e mostra que pediu outro horário, com o prazo mantido", async () => {
    const rescheduled = { appointment: { id: "a30", scheduled_at: START, status: "cancelled_by_citizen" as const } };
    const after: AppointmentItem = {
      request: { ...item().request, status: "open" },
      appointment: { ...item().appointment!, status: "cancelled_by_citizen", can_request_reschedule: false }
    };
    const list = vi.spyOn(citizenApi, "appointments")
      .mockResolvedValueOnce({ appointments: [ item() ] })
      .mockResolvedValueOnce({ appointments: [ after ] });
    const spy = vi.spyOn(citizenApi, "requestReschedule").mockResolvedValue(rescheduled);
    render(<AppointmentsSection citizenId="p1" onCheckIn={vi.fn()} />);

    await userEvent.click(await screen.findByRole("button", { name: "Não posso nesse horário" }));
    await userEvent.click(screen.getByRole("radio", { name: "Trabalho" }));
    await userEvent.click(screen.getByRole("radio", { name: "Manhã" }));
    await userEvent.click(screen.getByRole("button", { name: "Pedir outro horário" }));

    expect(spy).toHaveBeenCalledWith("a30", { reasonCode: "work", preferredPeriod: "morning", note: "" });
    expect(await screen.findByText(RESCHEDULE_REQUESTED)).toBeInTheDocument();
    expect(RESCHEDULE_REQUESTED).toBe("Você pediu outro horário. A unidade vai marcar um novo.");
    expect(screen.getByText("Prazo previsto: até 05/11")).toBeInTheDocument();
    expect(screen.queryByText("Cancelado por você")).not.toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Pedir outro horário" })).not.toBeInTheDocument();
    expect(list).toHaveBeenCalledTimes(2);
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/RescheduleForm.test.tsx src/modules/citizen/AppointmentsSection.test.tsx`
Expected: FAIL — `Failed to resolve import "./RescheduleForm"` no primeiro arquivo; no segundo, `RESCHEDULE_REQUESTED` `undefined` e `Unable to find role="button" and name "Não posso nesse horário"`.

- [ ] **Step 4: Crie o formulário**

`src/modules/citizen/RescheduleForm.tsx`:

```tsx
// src/modules/citizen/RescheduleForm.tsx
// "Não posso nesse horário" (spec 2026-10-05 do módulo 17 §6; ADR 0029):
// cancela este horário e devolve o pedido à unidade, com motivo (lista fixa),
// período preferido e uma nota opcional. Diferente do "Cancelar", que encerra
// o pedido. O prazo previsto não muda. Motivo, período e nota só vão no corpo
// do POST (citizenApi.requestReschedule): nada de <form>, URL ou console.
import { useRef, useState } from "react";
import { citizenApi, RESCHEDULE_NOTE_MAX, type PreferredPeriod, type RescheduleReasonCode } from "../../lib/citizenApi";
import { BigButton, ErrorText, FROZEN_TEXT_NOTICE, Field, RadioGroup, messageFor } from "./ui";

export const RESCHEDULE_INTRO =
  "Este horário será cancelado e o pedido volta para a unidade marcar outro. O prazo previsto continua o mesmo.";

const REASONS: readonly { value: RescheduleReasonCode; label: string }[] = [
  { value: "work", label: "Trabalho" },
  { value: "health", label: "Saúde" },
  { value: "transport", label: "Transporte" },
  { value: "other", label: "Outro motivo" }
];

const PERIODS: readonly { value: PreferredPeriod; label: string }[] = [
  { value: "morning", label: "Manhã" },
  { value: "afternoon", label: "Tarde" },
  { value: "any", label: "Qualquer período" }
];

// Como o api conta: depois do trim, por ponto de código (um emoji = 1).
export function noteLength(note: string): number {
  return [ ...note.trim() ].length;
}

export function RescheduleForm({ appointmentId, onDone, onClose }:
  { appointmentId: string; onDone: () => void; onClose: () => void }) {
  const [reason, setReason] = useState<RescheduleReasonCode | undefined>(undefined);
  const [period, setPeriod] = useState<PreferredPeriod | undefined>(undefined);
  const [note, setNote] = useState("");
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState<string | null>(null);
  // Guarda síncrona: setBusy só vale no próximo render. Depois do sucesso fica
  // ligada: a lista é relida e este formulário some.
  const inFlight = useRef(false);

  const length = noteLength(note);
  const ready = reason !== undefined && period !== undefined && length <= RESCHEDULE_NOTE_MAX;

  async function submit() {
    if (!reason || !period || length > RESCHEDULE_NOTE_MAX || inFlight.current) return;
    inFlight.current = true;
    setBusy(true);
    setError(null);
    try {
      await citizenApi.requestReschedule(appointmentId, { reasonCode: reason, preferredPeriod: period, note });
    } catch (e) {
      setError(messageFor(e));
      inFlight.current = false;
      setBusy(false);
      return;
    }
    onDone();
  }

  return (
    <div style={{ display: "grid", gap: 8, marginTop: 8 }}>
      <p style={{ fontSize: 18, margin: 0 }}>{RESCHEDULE_INTRO}</p>
      <RadioGroup legend="Por que não pode ir?" options={REASONS} value={reason} onChange={setReason} disabled={busy} />
      <RadioGroup legend="Qual período é melhor para você?" options={PERIODS} value={period} onChange={setPeriod}
        disabled={busy} />
      <Field label="Quer explicar? (opcional)" value={note} maxLength={RESCHEDULE_NOTE_MAX} autoComplete="off"
        onChange={e => setNote(e.target.value)} hint={FROZEN_TEXT_NOTICE} disabled={busy} />
      <p aria-live="polite"
        style={{ fontSize: 18, margin: 0,
          color: length > RESCHEDULE_NOTE_MAX ? "var(--down, #c0392b)" : "var(--ink2, #555)" }}>
        {`${length}/${RESCHEDULE_NOTE_MAX}`}
      </p>
      {error && <ErrorText>{error}</ErrorText>}
      <BigButton onClick={() => void submit()} disabled={busy || !ready}>Pedir outro horário</BigButton>
      <BigButton variant="secondary" onClick={onClose} disabled={busy}>Voltar</BigButton>
    </div>
  );
}
```

- [ ] **Step 5: Ligue o formulário na lista**

Em `src/modules/citizen/AppointmentsSection.tsx` (versão da Task 3):

(a) imports — troque

```tsx
import { BigButton, ErrorText, FROZEN_TEXT_NOTICE, Field, messageFor } from "./ui";
```

por

```tsx
import { RescheduleForm } from "./RescheduleForm";
import { BigButton, ErrorText, FROZEN_TEXT_NOTICE, Field, messageFor } from "./ui";

// "Não posso nesse horário" enviado: o horário vira cancelled_by_citizen e o
// pedido volta a "open" (o "Cancelar" o fecha como citizen_cancelled).
export const RESCHEDULE_REQUESTED = "Você pediu outro horário. A unidade vai marcar um novo.";
```

(b) depois de `confirmationClosed`, acrescente:

```tsx
// Quem decide é o api (can_request_reschedule, falso numa api anterior); o
// relógio só esconde o botão de uma leitura feita antes do início.
function canReschedule(appointment: Appointment): boolean {
  return appointment.can_request_reschedule === true && Date.now() < new Date(appointment.scheduled_at).getTime();
}
```

(c) troque `finalStatusText` inteira por:

```tsx
function finalStatusText(appointment: Appointment, request: AppointmentRequest, unit: string): string | null {
  switch (appointment.status) {
    case "cancelled_by_citizen": return request.status === "open" ? RESCHEDULE_REQUESTED : "Cancelado por você";
    case "expired": return "Cancelado: sem confirmação no prazo — a unidade pode marcar outro horário";
    case "no_show": return "Você não compareceu — a unidade pode marcar outro horário";
    case "checked_in": return `Atendido na ${unit}`;
    default: return null;
  }
}
```

(d) estado — depois de `const [cancelling, setCancelling] = useState(false);` acrescente

```tsx
  const [rescheduling, setRescheduling] = useState(false);
```

(e) troque

```tsx
  const final = appointment ? finalStatusText(appointment, unitName(appointment, request)) : null;
```

por

```tsx
  const final = appointment ? finalStatusText(appointment, request, unitName(appointment, request)) : null;
  const rescheduleButton = appointment && canReschedule(appointment) && (
    <BigButton variant="secondary" onClick={() => setRescheduling(true)} disabled={busy}>Não posso nesse horário</BigButton>
  );
```

(f) no bloco `scheduled`, troque

```tsx
          {!cancelling && (
            <div style={{ display: "grid", gap: 8 }}>
              {!confirmationClosed(appointment) && <BigButton onClick={confirm} disabled={busy}>Confirmar</BigButton>}
              <BigButton variant="secondary" onClick={() => setCancelling(true)} disabled={busy}>Cancelar</BigButton>
            </div>
          )}
```

por

```tsx
          {!cancelling && !rescheduling && (
            <div style={{ display: "grid", gap: 8 }}>
              {!confirmationClosed(appointment) && <BigButton onClick={confirm} disabled={busy}>Confirmar</BigButton>}
              {rescheduleButton}
              <BigButton variant="secondary" onClick={() => setCancelling(true)} disabled={busy}>Cancelar</BigButton>
            </div>
          )}
```

(g) no bloco `confirmed`, troque

```tsx
          {!cancelling && (
            <div style={{ display: "grid", gap: 8 }}>
              {appointment.check_in_available && (
                <BigButton onClick={() => onCheckIn(appointment.id)}>Cheguei na unidade</BigButton>
              )}
              <BigButton variant="secondary" onClick={() => setCancelling(true)} disabled={busy}>Cancelar</BigButton>
            </div>
          )}
```

por

```tsx
          {!cancelling && !rescheduling && (
            <div style={{ display: "grid", gap: 8 }}>
              {appointment.check_in_available && (
                <BigButton onClick={() => onCheckIn(appointment.id)}>Cheguei na unidade</BigButton>
              )}
              {rescheduleButton}
              <BigButton variant="secondary" onClick={() => setCancelling(true)} disabled={busy}>Cancelar</BigButton>
            </div>
          )}
```

(h) logo depois do bloco do formulário de cancelamento (o que termina em `Cancelar agendamento … </div> )}`), acrescente:

```tsx
      {rescheduling && appointment && (appointment.status === "scheduled" || appointment.status === "confirmed") && (
        <RescheduleForm appointmentId={appointment.id} onClose={() => setRescheduling(false)}
          onDone={() => { setRescheduling(false); onReload(); }} />
      )}
```

- [ ] **Step 6: Rode e veja passar**

Run: `npx vitest run src/modules/citizen/RescheduleForm.test.tsx src/modules/citizen/AppointmentsSection.test.tsx && npx tsc --noEmit`
Expected: PASS em todos (os testes antigos de "Cancelar" e "7. estados finais" inclusive: o cancelado com pedido `closed` continua "Cancelado por você"); `tsc` sem erros.

- [ ] **Step 7: Suíte inteira e commit**

Run: `npm test` — nenhum teste antigo caindo.

```bash
/opt/homebrew/bin/git add src/modules/citizen/RescheduleForm.tsx src/modules/citizen/RescheduleForm.test.tsx src/modules/citizen/AppointmentsSection.tsx src/modules/citizen/AppointmentsSection.test.tsx
/opt/homebrew/bin/git commit -m "feat: let citizens ask for another appointment time

\"Não posso nesse horário\" cancels the booked time and sends the request
back to the unit with a fixed reason, a preferred period and an optional
note of up to 200 characters, keeping the deadline. It is separate from the
existing cancel, which closes the request (ADR 0029).

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Lembrete de horário na caixa de avisos

**Files:**
- Modify: `src/lib/citizenApi.ts`, `src/lib/citizenApi.test.ts`
- Modify: `src/modules/citizen/NoticesStep.tsx`
- Test: `src/modules/citizen/NoticesStep.test.tsx`

**Interfaces:**
- Consumes: `UnitAddress`, `fmtWeekdayDateTime` (Task 1), `formatAddress`.
- Produces: `interface CampaignNotice { kind?: "campaign"; id; title; body; dispatched_at; read: boolean; cpf_masked: string | null }`; `interface ReminderNotice { kind: "appointment_reminder"; id; appointment_id; appointment_type_name: string | null; unit_name: string | null; unit_address: UnitAddress | null; scheduled_at: string; professional_name: string | null; read: boolean; cpf_masked: string | null }`; `type Notice = CampaignNotice | ReminderNotice` (os testes existentes que tipam campanhas como `Notice` sem `kind` continuam compilando); `citizenApi.notices()` devolve campanhas com `kind: "campaign"` (ausente = api anterior), lembretes com `read` e `cpf_masked` do contrato §8, e descarta `kind` desconhecido. Em `NoticesStep.tsx`: `REMINDER_TITLE = "Lembrete de horário"`, `REMINDER_HINT`.

- [ ] **Step 1: Testes do cliente**

No `describe("citizenApi — módulo 17 (agenda)")` de `src/lib/citizenApi.test.ts`, acrescente:

```ts
  it("notices: aviso sem kind (api anterior) é campanha", async () => {
    mockFetch(200, { notices: [
      { id: "c1", title: "Vacinação", body: "Texto", dispatched_at: "2026-09-28T13:00:00-03:00", read: false }
    ], unread_count: 1 });
    const [n] = (await citizenApi.notices()).notices;
    expect(n.kind).toBe("campaign");
  });

  it("notices: lembrete com read e cpf_masked do contrato; campos ausentes viram null/false", async () => {
    mockFetch(200, { notices: [
      { kind: "appointment_reminder", id: "rem1", appointment_id: "a1", appointment_type_name: "Consulta médica",
        unit_name: "UBS Batel", unit_address: { street: "Rua Padre Anchieta", number: "1500", complement: null, zip: "80730000" },
        scheduled_at: "2026-10-08T09:00:00-03:00", professional_name: "Ana Souza", read: true, cpf_masked: "***.982.247-**" },
      { kind: "appointment_reminder", id: "rem2", appointment_id: "a2", scheduled_at: "2026-10-08T10:00:00-03:00" }
    ], unread_count: 1 });
    const [read, bare] = (await citizenApi.notices()).notices;
    expect(read).toMatchObject({ kind: "appointment_reminder", id: "rem1", appointment_id: "a1", read: true, cpf_masked: "***.982.247-**" });
    expect(bare).toEqual({
      kind: "appointment_reminder", id: "rem2", appointment_id: "a2", appointment_type_name: null, unit_name: null,
      unit_address: null, scheduled_at: "2026-10-08T10:00:00-03:00", professional_name: null, read: false, cpf_masked: null
    });
  });

  it("notices: tipo de aviso desconhecido (api mais nova) fica fora da lista", async () => {
    mockFetch(200, { notices: [
      { kind: "exam_result", id: "x1", read: false },
      { kind: "campaign", id: "c1", title: "Vacinação", body: "Texto", dispatched_at: "2026-09-28T13:00:00-03:00", read: true }
    ], unread_count: 0 });
    expect((await citizenApi.notices()).notices.map(n => n.id)).toEqual([ "c1" ]);
  });
```

- [ ] **Step 2: Testes da tela**

No fim de `src/modules/citizen/NoticesStep.test.tsx` (e troque o `import` do cliente por `import { citizenApi, ApiError, type Notice, type ReminderNotice } from "../../lib/citizenApi";` e acrescente `REMINDER_TITLE, REMINDER_HINT` ao `import` de `./NoticesStep`):

```ts
// Módulo 17 (F-17.8): o lembrete da véspera do horário confirmado divide a
// caixa com as campanhas.
describe("NoticesStep — lembrete de horário", () => {
  beforeEach(() => vi.setSystemTime(new Date("2026-10-07T18:00:00-03:00")));

  const reminder: ReminderNotice = {
    kind: "appointment_reminder", id: "rem1", appointment_id: "a1", appointment_type_name: "Consulta médica",
    unit_name: "UBS Batel", unit_address: { street: "Rua Padre Anchieta", number: "1500", complement: null, zip: "80730000" },
    scheduled_at: "2026-10-08T09:00:00-03:00", professional_name: "Ana Souza", read: false, cpf_masked: null
  };

  it("lista lembrete e campanha na ordem da API; o lembrete tem título próprio, dia e hora e 'novo'", async () => {
    setup([ reminder, read ]);
    const first = await item(/Lembrete de horário/);
    expect(REMINDER_TITLE).toBe("Lembrete de horário");
    expect(within(first).getByText("qui., 08/10, 09:00")).toBeInTheDocument();
    expect(within(first).getByText("Consulta médica")).toBeInTheDocument();
    expect(within(first).getByText("novo")).toBeInTheDocument();
    const items = screen.getAllByRole("listitem").map(li => li.textContent);
    expect(items[0]).toMatch(/Lembrete de horário/);
    expect(items[1]).toMatch(/Mutirão/);
  });

  it("abrir mostra quando, o quê, com quem, onde e o endereço, e marca lido pelo id do lembrete", async () => {
    const { markRead } = setup([ reminder ]);
    await userEvent.click(await item(/Lembrete de horário/));
    expect(screen.getByRole("heading", { name: "Lembrete de horário" })).toBeInTheDocument();
    expect(screen.getByText("qui., 08/10, 09:00")).toBeInTheDocument();
    expect(screen.getByText("Consulta médica")).toBeInTheDocument();
    expect(screen.getByText("Com Ana Souza")).toBeInTheDocument();
    expect(screen.getByText("UBS Batel")).toBeInTheDocument();
    expect(screen.getByText("Rua Padre Anchieta, 1500 · CEP 80730-000")).toBeInTheDocument();
    expect(screen.getByText(REMINDER_HINT)).toBeInTheDocument();
    expect(markRead).toHaveBeenCalledWith("rem1");

    await userEvent.click(screen.getByRole("button", { name: "Voltar aos avisos" }));
    await waitFor(() => expect(within(screen.getByRole("button", { name: /Lembrete de horário/ })).queryByText("novo"))
      .not.toBeInTheDocument());
  });

  it("lembrete sem profissional, unidade nem endereço: só o que veio, sem 'null'", async () => {
    setup([ { ...reminder, professional_name: null, unit_name: null, unit_address: null } ]);
    await userEvent.click(await item(/Lembrete de horário/));
    expect(screen.queryByText(/^Com /)).not.toBeInTheDocument();
    expect(screen.queryByText(/null|undefined/)).not.toBeInTheDocument();
  });

  it("várias pessoas no telefone: o lembrete diz o CPF mascarado de quem é", async () => {
    setup([ { ...reminder, cpf_masked: "***.982.247-**" } ]);
    expect(within(await item(/Lembrete de horário/)).getByText("Para o CPF ***.982.247-**")).toBeInTheDocument();
  });
});
```

(`setup`, `item` e `read` são os do topo do arquivo; o `beforeEach` do arquivo liga o relógio falso, este só move a data.)

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/lib/citizenApi.test.ts src/modules/citizen/NoticesStep.test.tsx`
Expected: FAIL — `n.kind` `undefined`; lembrete sem `read` normalizado; tipo desconhecido na lista; na tela, `REMINDER_TITLE` `undefined` e `Unable to find role="button" and name /Lembrete de horário/`. O `tsc` também acusa `ReminderNotice` inexistente.

- [ ] **Step 4: Tipos e normalização dos avisos**

Em `src/lib/citizenApi.ts`, troque a interface `Notice` inteira (do comentário `// Campanhas (spec 2026-09-29 §6.2; ADR 0024)…` até o `}` antes de `NoticesResult`) por:

```ts
// Caixa de avisos (spec 2026-09-29 §6.2; ADR 0024; módulo 17, contrato §5):
// campanhas e lembretes de horário. Cidadão não tem nome: com mais de uma
// pessoa no telefone, o aviso diz de quem é pelo CPF mascarado.
export interface CampaignNotice {
  kind?: "campaign"; // ausente numa api anterior ao módulo 17
  id: string; // id do campaign_recipient
  title: string;
  body: string; // texto simples, com quebras de linha
  dispatched_at: string;
  read: boolean;
  cpf_masked: string | null; // null quando o telefone tem uma pessoa só
}

// Lembrete da véspera de um horário confirmado (F-17.8; contrato §8): read e
// cpf_masked com a mesma regra das campanhas.
export interface ReminderNotice {
  kind: "appointment_reminder";
  id: string;
  appointment_id: string;
  appointment_type_name: string | null;
  unit_name: string | null;
  unit_address: UnitAddress | null;
  scheduled_at: string;
  professional_name: string | null;
  read: boolean;
  cpf_masked: string | null;
}

export type Notice = CampaignNotice | ReminderNotice;

// Forma crua de GET /citizen/notices: tudo pode faltar, menos o id.
interface RawNotice {
  kind?: string;
  id: string;
  read?: boolean;
  cpf_masked?: string | null;
  title?: string;
  body?: string;
  dispatched_at?: string;
  appointment_id?: string;
  appointment_type_name?: string | null;
  unit_name?: string | null;
  unit_address?: UnitAddress | null;
  scheduled_at?: string;
  professional_name?: string | null;
}

// Tipo desconhecido (api mais nova) ou lembrete sem horário: fora da lista,
// em vez de um item quebrado.
function normalizeNotice(n: RawNotice): Notice | null {
  const read = n.read === true;
  const cpf_masked = n.cpf_masked ?? null;
  if (n.kind === "appointment_reminder") {
    if (!n.appointment_id || !n.scheduled_at) return null;
    return {
      kind: "appointment_reminder", id: n.id, appointment_id: n.appointment_id,
      appointment_type_name: n.appointment_type_name ?? null, unit_name: n.unit_name ?? null,
      unit_address: n.unit_address ?? null, scheduled_at: n.scheduled_at,
      professional_name: n.professional_name ?? null, read, cpf_masked
    };
  }
  if (n.kind !== undefined && n.kind !== "campaign") return null;
  return {
    kind: "campaign", id: n.id, title: n.title ?? "", body: n.body ?? "",
    dispatched_at: n.dispatched_at ?? "", read, cpf_masked
  };
}
```

E troque o corpo de `notices`:

```ts
  notices: async (): Promise<NoticesResult> => {
    const data = await call<{ notices?: Notice[]; unread_count?: number }>("GET", "/notices");
    return {
      notices: (data.notices ?? []).map(n => ({ ...n, read: n.read === true, cpf_masked: n.cpf_masked ?? null })),
      unread_count: data.unread_count ?? 0
    };
  },
```

por

```ts
  notices: async (): Promise<NoticesResult> => {
    const data = await call<{ notices?: RawNotice[]; unread_count?: number }>("GET", "/notices");
    return {
      notices: (data.notices ?? []).map(normalizeNotice).filter((n): n is Notice => n !== null),
      unread_count: data.unread_count ?? 0
    };
  },
```

- [ ] **Step 5: A tela**

Substitua `src/modules/citizen/NoticesStep.tsx` inteiro por:

```tsx
// src/modules/citizen/NoticesStep.tsx
// Caixa de avisos da Secretaria (spec 2026-09-29 §8; ADR 0024; módulo 17
// F-17.8). Avisos de todas as pessoas do telefone da sessão: campanhas e o
// lembrete da véspera de um horário confirmado. Com mais de uma pessoa, o CPF
// mascarado diz de quem é (cidadão não tem nome). Tocar abre o texto e marca
// lido. O texto é simples: o React escapa tudo e o pre-wrap preserva as
// quebras de linha.
import { useEffect, useRef, useState } from "react";
import { citizenApi, type Notice, type ReminderNotice } from "../../lib/citizenApi";
import { fmtDate, fmtWeekdayDateTime } from "../../lib/format";
import { formatAddress } from "../../lib/territory";
import { BigButton, ErrorText, Screen, messageFor } from "./ui";

export const EMPTY_NOTICES = "Nenhum aviso da Secretaria por enquanto.";
export const REMINDER_TITLE = "Lembrete de horário";
export const REMINDER_HINT =
  "Se não puder ir, abra \"Seus agendamentos\", em Minhas triagens, e toque em \"Não posso nesse horário\".";

const itemStyle = {
  display: "grid", gap: 4, width: "100%", minHeight: 56, padding: 12, textAlign: "left",
  borderRadius: 12, border: "1px solid var(--rule2, #ccc)", fontSize: 18, cursor: "pointer",
  color: "var(--ink, #222)"
} as const;

const newStyle = {
  alignSelf: "start", padding: "0 8px", borderRadius: 999, fontSize: 18, fontWeight: 600,
  background: "var(--accent, #2b4bd8)", color: "#fff"
} as const;

function ReminderDetail({ notice, onBack }: { notice: ReminderNotice; onBack: () => void }) {
  const address = formatAddress(notice.unit_address);
  return (
    <Screen title={REMINDER_TITLE}
      footer={<BigButton variant="secondary" onClick={onBack}>Voltar aos avisos</BigButton>}>
      <p style={{ margin: "0 0 8px", fontWeight: 600 }}>{fmtWeekdayDateTime(notice.scheduled_at)}</p>
      {notice.cpf_masked && <p style={{ margin: "0 0 4px" }}>Para o CPF {notice.cpf_masked}</p>}
      {notice.appointment_type_name && <p style={{ margin: "0 0 4px" }}>{notice.appointment_type_name}</p>}
      {notice.professional_name && <p style={{ margin: "0 0 4px" }}>Com {notice.professional_name}</p>}
      {notice.unit_name && <p style={{ margin: "0 0 4px" }}>{notice.unit_name}</p>}
      {address && <p style={{ margin: "0 0 4px", color: "var(--ink2, #555)" }}>{address}</p>}
      <p style={{ marginTop: 16 }}>{REMINDER_HINT}</p>
    </Screen>
  );
}

function ItemContent({ notice }: { notice: Notice }) {
  const badge = !notice.read && <span style={newStyle}>novo</span>;
  const cpf = notice.cpf_masked && <span>Para o CPF {notice.cpf_masked}</span>;
  if (notice.kind === "appointment_reminder") {
    return (
      <>
        <span style={{ display: "flex", justifyContent: "space-between", gap: 8 }}>
          <strong>{REMINDER_TITLE}</strong>{badge}
        </span>
        <span style={{ color: "var(--ink2, #555)" }}>{fmtWeekdayDateTime(notice.scheduled_at)}</span>
        {notice.appointment_type_name && <span>{notice.appointment_type_name}</span>}
        {cpf}
      </>
    );
  }
  return (
    <>
      <span style={{ display: "flex", justifyContent: "space-between", gap: 8 }}>
        <strong>{notice.title}</strong>{badge}
      </span>
      <span style={{ color: "var(--ink2, #555)" }}>{fmtDate(notice.dispatched_at)}</span>
      {cpf}
    </>
  );
}

export function NoticesStep({ onBack, onPreferences, onRead }:
  { onBack: () => void; onPreferences?: () => void; onRead?: () => void }) {
  const [notices, setNotices] = useState<Notice[] | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [openId, setOpenId] = useState<string | null>(null);
  // Um POST por aviso: a abertura repetida antes da resposta não manda outro.
  const asked = useRef(new Set<string>());

  function load() {
    setError(null);
    citizenApi.notices().then(r => setNotices(r.notices)).catch(e => setError(messageFor(e)));
  }

  useEffect(() => { load(); }, []);

  function open(n: Notice) {
    setOpenId(n.id);
    if (n.read || asked.current.has(n.id)) return;
    asked.current.add(n.id);
    citizenApi.readNotice(n.id)
      .then(() => {
        setNotices(list => list && list.map(x => (x.id === n.id ? { ...x, read: true } : x)));
        onRead?.();
      })
      // Falhou (rede, ou 404 porque a linha foi apagada): o texto já está na
      // tela; o aviso segue "novo" e a próxima abertura tenta de novo.
      .catch(() => { asked.current.delete(n.id); });
  }

  const current = notices?.find(n => n.id === openId) ?? null;
  if (current && current.kind === "appointment_reminder") {
    return <ReminderDetail notice={current} onBack={() => setOpenId(null)} />;
  }
  if (current) {
    return (
      <Screen title={current.title}
        footer={<BigButton variant="secondary" onClick={() => setOpenId(null)}>Voltar aos avisos</BigButton>}>
        <p style={{ margin: "0 0 4px", color: "var(--ink2, #555)" }}>{fmtDate(current.dispatched_at)}</p>
        {current.cpf_masked && <p style={{ margin: "0 0 4px" }}>Para o CPF {current.cpf_masked}</p>}
        <div style={{ whiteSpace: "pre-wrap", lineHeight: 1.5, overflowWrap: "anywhere", marginTop: 16 }}>
          {current.body}
        </div>
      </Screen>
    );
  }

  return (
    <Screen title="Avisos" footer={<>
      {onPreferences && <BigButton variant="secondary" onClick={onPreferences}>Preferências de avisos</BigButton>}
      <BigButton variant="secondary" onClick={onBack}>Voltar ao início</BigButton>
    </>}>
      {error && <>
        <ErrorText>{error}</ErrorText>
        <BigButton style={{ marginTop: 12 }} onClick={load}>Tentar de novo</BigButton>
      </>}
      {notices === null && !error && <p>Carregando…</p>}
      {notices !== null && notices.length === 0 && <p>{EMPTY_NOTICES}</p>}
      {notices !== null && notices.length > 0 && (
        <ul style={{ listStyle: "none", margin: 0, padding: 0, display: "grid", gap: 12 }}>
          {notices.map(n => (
            <li key={`${n.kind ?? "campaign"}:${n.id}`}>
              <button type="button" onClick={() => open(n)}
                style={{ ...itemStyle, background: n.read ? "transparent" : "var(--accent-bg, #eef1ff)" }}>
                <ItemContent notice={n} />
              </button>
            </li>
          ))}
        </ul>
      )}
    </Screen>
  );
}
```

- [ ] **Step 6: Rode e veja passar**

Run: `npx vitest run src/lib/citizenApi.test.ts src/modules/citizen/NoticesStep.test.tsx src/modules/citizen/NoticesLink.test.tsx src/modules/citizen/Flow.notices.test.tsx src/modules/citizen/Flow.route.test.tsx && npx tsc --noEmit`
Expected: PASS em todos (as fixtures antigas tipadas como `Notice` sem `kind` continuam valendo como campanha); `tsc` sem erros.

- [ ] **Step 7: Suíte inteira e commit**

Run: `npm test` — nenhum teste antigo caindo.

```bash
/opt/homebrew/bin/git add src/lib/citizenApi.ts src/lib/citizenApi.test.ts src/modules/citizen/NoticesStep.tsx src/modules/citizen/NoticesStep.test.tsx
/opt/homebrew/bin/git commit -m "feat: show appointment reminders in the citizen notice box

The notice box now reads a kind per notice: campaigns as before, and the
reminder of a confirmed appointment with when, what, who and where. Unknown
kinds from a newer api are left out (ADR 0029).

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Revisão, conferência com o api do módulo 17 e prova no navegador

**Files:** nenhum, a não ser que a prova ache um bug (nesse caso, teste + correção + commit `fix:` no wpda; se o bug for do api, avise a sessão dona do api em vez de mexer lá).

- [ ] **Step 1: Revisão do branch.** Um subagente revisor lê `/opt/homebrew/bin/git diff origin/main..HEAD` do wpda contra a spec §6–§8, o contrato §2 e §5 e este plano, com atenção a:
  - motivo, período e nota só no corpo de `requestReschedule`: `/opt/homebrew/bin/git diff origin/main..HEAD -- src | grep -n "console\.\|location\|history\.\(push\|replace\)State\|<form"` só pode mostrar linhas de teste;
  - nenhuma data de calendário passando por `new Date(`: `grep -n "new Date(" src/modules/citizen/SchedulingRequestNotice.tsx src/modules/citizen/RescheduleForm.tsx` vazio, e `due_on` só formatado por `fmtDayMonth`;
  - nenhuma tela lista vagas ou propõe horário (o cidadão nunca escolhe a vaga);
  - "Não posso nesse horário" e "Cancelar" nunca juntos; o "Cancelar" continua exigindo 10 caracteres;
  - todo `button`/`label` novo com `fontSize` ≥ 18 e alvo ≥ 48 px;
  - `vi.setSystemTime` em `beforeEach` em todo teste novo que depende de data;
  - commits só com arquivos pelo nome (`/opt/homebrew/bin/git show --stat origin/main..HEAD` sem `node_modules`).
- [ ] **Step 2: Confira o contrato com o api.** Com o api do branch do módulo 17 em `:3034` (subido pelo plano do api), logado como cidadão no navegador da prova (Step 3), leia as respostas pela aba de rede de `GET /citizen/appointments`, `GET /citizen/triages/:id`, `GET /citizen/notices` e `POST /citizen/appointments/:id/reschedule_request`. Elas batem com o contrato §5 e §8 — todos os campos presentes (o api do módulo 17 não tem desculpa de rollout): pedidos com `kind`/`target_unit_name`/`appointment_type_name`/`due_on`, endereço na forma de `reference_units`, lembrete com `read` e `cpf_masked`, `unread_count` contando o lembrete, `reschedule_request` respondendo `{ appointment }`. Se um campo faltar ou vier com **outro** formato (ex.: `unit.address` como texto, `read_at` no lugar de `read`), pare e avise; não adapte o cliente a outro formato sem decisão.
- [ ] **Step 3: Prova no navegador (Curitiba, spec §9).** Com o api do módulo 17 em `:3034` (migração de cidade aplicada e a semente da spec §10: modelo "Manhã" em Curitiba, turnos da semana da médica e da enfermeira, "Saúde do idoso" com regra de agendamento rotina/30 dias) e o Vite deste worktree:

  ```bash
  cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda/.claude/mod17
  VITE_API_PROXY_TARGET=http://localhost:3034 npx vite --port 5189 --host 0.0.0.0
  ```

  Abra `http://curitiba.localhost:5189/wpda/`. O login do cidadão (celular + código do log do api) é feito pelo usuário; não digite o código. Depois, com screenshot de cada passo:
  1. a avó (62, com bairro coberto por unidade de referência) faz "Saúde do idoso" → o resultado mostra "Pedido de agendamento", o tipo e "A <unidade> vai entrar em contato… Prazo previsto: até <hoje + 30, DD/MM>.";
  2. "Minhas triagens" → "Seus agendamentos" mostra "Pedido da triagem: <tipo> — a <unidade> vai entrar em contato…" e "Prazo previsto: até DD/MM";
  3. a recepção marca uma vaga para esse pedido (dashboard do módulo 17 ou, sem ele mergeado, pedido à sessão dona do api) → recarregar: "Agendado: <dia>, <início> às <fim> — <unidade>.", tipo, "Com <profissional>" e o endereço; os botões Confirmar, "Não posso nesse horário" e Cancelar;
  4. "Não posso nesse horário" → Trabalho, Tarde, nota curta → "Pedir outro horário" → a linha passa a "Você pediu outro horário. A unidade vai marcar um novo." com o mesmo prazo; a URL continua `/wpda/` e o console do navegador sem a nota;
  5. com um bairro sem unidade de referência, a mesma triagem mostra o texto "A Secretaria de Saúde vai indicar a unidade…" no resultado e em "Seus agendamentos";
  6. um horário confirmado para amanhã + o `Appointments::RemindJob` de Curitiba rodado pela sessão dona do api → "Avisos" com o selo; o lembrete aparece como "Lembrete de horário" com dia e hora, e abre com tipo, profissional, unidade e endereço; ao voltar, o selo diminui.
- [ ] **Step 4: Suíte final.** `npm test` e `npx tsc --noEmit` no worktree; registre a contagem no relatório (antes → depois).
- [ ] **Step 5: Pare.** O merge do wpda vem depois do merge do api (contrato §7), e só com autorização explícita do usuário. Não faça push.

---

## Incorporado ao contrato

As cinco propostas deste plano foram aceitas e estão no contrato §8 (2026-10-05); o código acima já usa as formas decididas, sem alternativa:

1. **Pedido em `GET /citizen/appointments`:** `request` ganha `kind` (`return`|`referral`|`triage`), `target_unit_name: string|null` (null na fila "sem unidade"), `appointment_type_name` e `due_on` (`AAAA-MM-DD`). O api não pode quebrar com pedido sem unidade (`item_json` hoje faz `request.target_unit.name`). Ausência só por api anterior (rollout): o wpda mostra "Pedido da triagem: Consulta — …" sem prazo.
2. **Endereço:** `unit.address` e `unit_address` têm a forma de `reference_units` (`{ street, number, complement, zip }`, cada campo podendo ser `null`), formatados por `formatAddress`.
3. **Lembrete na caixa de avisos:** `read: boolean` (não `read_at`), `cpf_masked` quando o celular tem mais de uma pessoa, e `unread_count` conta os lembretes com a mesma regra de `notices_muted`.
4. **Resposta de `reschedule_request`:** `{ "appointment": … }`, como `confirm` e `cancel`. O wpda tipa a resposta assim e não lê o corpo (relê a lista).
5. **Id do aviso:** opaco; `POST /citizen/notices/:id/read` procura nas duas fontes (404 para id de outro telefone, como hoje).

## Self-review

- **Cobertura da spec:** §6 resultado da triagem ("A Unidade X vai entrar em contato… Prazo previsto: até DD/MM", sem unidade = texto próprio) → Task 2; §7 "pedido … em 'Meus horários'" → Task 3 (pedido da triagem com tipo e prazo; horário com tipo, profissional, unidade com endereço e fim, contrato §5); §6 "Não posso nesse horário" (motivo de lista fixa, nota ≤ 200, período, prazo mantido, distinto do Cancelar) → Tasks 1 e 4; §6 caixa de avisos com campanhas e lembretes → Task 5; §8 LGPD (nota fora de URL/console/log) → Tasks 1, 4 e 6; §9 front (botão, motivo/período, avisos, relógio fixo) → Tasks 2–5; §9 prova no navegador → Task 6; §11 rollout (wpda depois do api; compatível com api anterior) → normalização na borda em todas as tasks e Step 5 da Task 6.
- **Placeholders:** nenhum passo sem código; edições em arquivos existentes dizem o trecho âncora exato ou substituem o arquivo inteiro (`AppointmentsSection.tsx` na Task 3, `NoticesStep.tsx` na Task 5). A porta do api na prova é 3034.
- **Tipos:** `SchedulingRequest` (Task 1) é o que `SchedulingRequestNotice` e `ResultStep` usam (Task 2); `Appointment.ends_at/unit/appointment_type_name/professional_name/can_request_reschedule` (Task 1) são os lidos em `when`, `unitName`, `AppointmentDetails` (Task 3) e `canReschedule` (Task 4); `AppointmentRequest.kind "triage"`, `target_unit_name: string | null`, `due_on` (Task 3) são os lidos em `openRequestText` e na linha do prazo; `RescheduleReasonCode`, `PreferredPeriod`, `RESCHEDULE_NOTE_MAX` e `requestReschedule(id, { reasonCode, preferredPeriod, note })` (Task 1) são os usados pelo `RescheduleForm` (Task 4); `Notice = CampaignNotice | ReminderNotice` (Task 5) mantém compiláveis as fixtures existentes (campanha com `kind` opcional). Os formatadores `fmtDayMonth`, `fmtWeekdayDateTime`, `fmtHourMinute` (Task 1) não colidem com o `fmtTime` local de `HistoryStep.tsx`.
- **Ordem e `tsc`:** cada task deixa `tsc` verde — por isso o tipo do pedido muda só na Task 3 (junto da tela que o lê) e a união `Notice` só na Task 5 (junto da `NoticesStep`, que lê `title`).
- **Review Focus:** os cinco itens têm teste na task dona (Tasks 1, 2, 3, 4 e 5).
