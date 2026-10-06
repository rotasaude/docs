# Módulo 17 — Agenda dos profissionais (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Telas do módulo 17 no painel da cidade. O `municipal_admin` mantém os tipos de atendimento e os modelos de agenda (editor de faixas com validação local e pré-visualização), escolhe o modelo de cada turno e o tipo padrão de cada vínculo. A recepção vê a fila com prazo, prioridade e marcas, abre o detalhe do pedido, marca numa vaga calculada, faz encaixe com justificativa e contador, usa a marcação livre só em dia sem turno, vê a agenda da unidade por profissional e atribui unidade aos pedidos "sem unidade". O profissional lê a própria agenda (dia e semana). O autor de protocolo escreve as regras de `scheduling` num painel "Agendamento" com o construtor de condições do módulo 15.

**Architecture:** O cliente HTTP novo vai para o fim de `src/lib/api.ts`, com os tipos copiados do arquivo de contratos. As regras ficam fora do React, em funções puras: `src/lib/scheduling.ts` (rótulos, validação de tipo e de faixas espelhando os 422, aviso de confirmação, dias e semanas no fuso da cidade, marcas da fila) e `src/lib/schedulingRules.ts` (leitura e escrita do bloco `scheduling` da definição, no mesmo padrão de `src/lib/offer.ts`). As telas reaproveitam o que existe: `Professionals` ganha abas (`SegmentedControl`, como `Protocols`), `ProfessionalDetail` ganha modelo no turno e tipo padrão no vínculo, `Attendance` continua montando os painéis da recepção (`Requests`, `Agenda`, e o novo `UnassignedRequests`), `ProtocolEditor` ganha o painel ao lado de "Oferta e sugestões", e "Minha agenda" é um módulo novo no grupo "Atendimento". As traduções de erro continuam em `attendanceError` e `professionalError`.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("../lib/api")` e `vi.stubGlobal("fetch")`), Vite 5 (proxy de dev).

**Spec:** `docs/superpowers/specs/2026-10-05-module-17-scheduling-design.md` (§7 é deste plano; §3, §4, §5.3 e §8 dão as regras), `docs/adr/0029.md` e o arquivo de contratos `docs/superpowers/plans/2026-10-05-module-17-scheduling-contracts.md` (§1, §2, §3, §4, o último item de §8 e §9 são deste plano; §9 vale sobre as seções anteriores). Os tipos da Task 1 copiam o contrato. O plano do `api` do módulo 17 precisa estar **mergeado antes** do merge deste (contratos §7).

## Global Constraints

- Rotas usadas (sessão da cidade; nenhuma rota nova de proxy: `/professionals`, `/attendance` e `/authoring` já estão no `vite.config.ts`):
  - `GET /professionals/appointment_types` → `{ types }`; `POST /professionals/appointment_types` (criar, `origin=city`); `POST /professionals/appointment_types/:key` (`name`?, `duration_minutes`?, `cbo_prefixes`? só em `city`, `active`?) → o tipo, puro;
  - `GET /professionals/schedule_templates` → `{ templates }`; `POST /professionals/schedule_templates` e `POST /professionals/schedule_templates/:id` → o modelo, puro;
  - `POST /professionals/schedule_templates/preview` `{ blocks, fit_in_limit, sample: { starts_at, ends_at, cbo_code } }` → `{ slots, blocks }`, nada gravado;
  - `POST /professionals/links/:id/shifts` (existente, devolve `{ shift }`) aceita `schedule_template_id`; `POST /professionals/shifts/:id/template` `{ schedule_template_id | null }` → o turno; `GET /professionals/:id/shifts` devolve `schedule_template_id`;
  - `POST /professionals/links/:id/default_type` `{ appointment_type_key | null }` → o vínculo; `GET /professionals/:id` traz `default_appointment_type_key` em cada vínculo;
  - `GET /professionals/appointment_types` também é lido por `protocol_author` e `protocol_reviewer` (§9);
  - `GET /professionals/me/agenda?from=&to=` (até 7 dias, `from`/`to` inclusivos) → `{ days: [ { date, shifts: [ { shift_id, unit: { id, name }, starts_at, ends_at, cancelled_at, blocks, appointments } ] } ] }`; sem cadastro profissional, 404 `no_profile`;
  - `GET /attendance/units/:id/requests` (existente, `{ requests }`) com os campos novos e a ordenação do api; o número de prioridade da triagem passa a ser `triage_priority`, e `priority` é só `routine|priority`; `origin_unit_name` é nulo em `kind = triage`; `GET /attendance/requests/:id` = o item da fila + `reschedule_note` (único lugar com a nota); `GET /attendance/requests/unassigned` → `{ requests }`; `POST /attendance/requests/:id/assign_unit` `{ unit_id }` → o pedido, puro; 409 `already_assigned`;
  - `GET /attendance/units/:id/availability?type=&from=&to=` (até 14 dias, inclusivos) → `{ slots, legacy_days }`;
  - `POST /attendance/requests/:id/appointments` (existente, 201 `{ appointment }`) com `kind` `slot` | `fit_in` | `legacy`, sempre com `health_unit_id`; a marcação livre mantém o 409 `slot_taken` com `taken` e o `allow_overlap` (api#26);
  - `GET /attendance/units/:id/agenda?date=` (existente) → `{ date, professionals, unassigned }` (forma nova, contratos §4.5); turno com `starts_at`, `ends_at`, `cancelled_at`; faixas `bookable` com `appointment_type_name`; em `legacy`, `ends_at`, tipo, `professional` e `shift_id` vêm `null`;
  - `POST /authoring/gate`: aviso de tipo inexistente = 200 `{ warnings: [string] }`.
- Recusas e o que a tela faz:
  - marcar: 409 `slot_taken` e `slot_unavailable` → recarrega as vagas e mantém o painel aberto, sem marcar outra vaga sozinha; 409 `use_slots` → recarrega as vagas (o dia ganhou turno); 409 `fit_in_limit` → recarrega a agenda (contador) e trava o botão; 409 `citizen_busy`; 422 `invalid_reason`, `type_not_served`, `outside_shift`; 409 `request_not_open` → fecha o painel e recarrega a fila (comportamento de hoje);
  - tipos: 422 `invalid_key`, `key_taken`, `invalid_duration`, `invalid_cbo_prefixes`, `platform_type_locked`;
  - tipos e modelos: 422 `invalid_name` (§9);
  - modelos: 422 `invalid_blocks` com `detail` (`overlap`, `missing_type`, `unknown_type`, `bad_time`; e, pela §9, `empty`, `bad_slot_minutes`, `inactive_type`, `crosses_midnight`) e `invalid_fit_in_limit`.
- Valores (contratos §2): `booking_kind` `slot` | `fit_in` | `legacy`; faixa `{ starts: "HH:MM", ends: "HH:MM", kind: walk_in|bookable|blocked, appointment_type_key?, slot_minutes? }`; `priority` `routine` | `priority`; `preferred_period` `morning` | `afternoon` | `any`; `reschedule_reason_code` `work` | `health` | `transport` | `other`. Datas `YYYY-MM-DD` no fuso da cidade; instantes ISO 8601 com fuso.
- Regras espelhadas na tela (a API continua sendo quem garante): chave `^[a-z][a-z0-9_]{1,40}$`; nome não vazio; duração 5–240; `cbo_prefixes` de 1 a 20 itens, cada um com 1 a 6 dígitos; `fit_in_limit` 0–20; modelo com pelo menos uma faixa; faixa `bookable` exige tipo **ativo** (inexistente = `unknown_type`, inativo = `inactive_type`); `slot_minutes` 5–240; faixa não cruza a meia-noite (fim depois do início, sem `24:00` = `crosses_midnight`); formato fora de `HH:MM` = `bad_time`; faixas sem sobreposição (encostar não é sobrepor); justificativa do encaixe ≥ 10 caracteres; vagas até 14 dias por consulta; Minha agenda até 7 dias.
- Schema `protocols-v1.5.0` (contratos §1): `scheduling` com até 10 regras `{ when, appointment_type, priority, due_in_days }`, todas obrigatórias; `appointment_type` casa `^[a-z][a-z0-9_]{1,40}$`; `due_in_days` inteiro 1–365; `when` usa `outcome.*`, `profile.*` e ids de passo (os campos de `fieldsFor("suggestion", { definition })`). Lista vazia sai **ausente** do JSON. Tipo inexistente na cidade é aviso do gate, sem bloquear. `when` aceita só a condição estruturada (contratos §8: o mapa legado `{passo: valor}` é recusado pelo schema); o construtor só emite a forma estruturada, e regra fora do subconjunto aparece como "regra avançada", como no módulo 15. Vale a primeira regra que casar; resultado urgente nunca gera pedido.
- LGPD (spec §8): `reschedule_note` só aparece no detalhe do pedido, nunca na lista; `fit_in_reason` só aparece para quem marca e para o `municipal_admin` (a API já filtra; a tela só mostra o que veio); justificativa do encaixe com o `FrozenTextNotice`; nenhum texto livre em URL (só corpo de POST).
- O cidadão nunca escolhe a vaga (ADR 0029): nada desta entrega toca o wpda.
- Papéis: Profissionais (tipos, modelos, modelo no turno, tipo padrão) só `municipal_admin` (grupo "Equipe"); fila, marcação, encaixe, agenda e "sem unidade" para quem hoje marca (`citizen_verifier`, o `canVerify` de `Attendance.tsx`); Minha agenda só `health_professional`.
- Interface tem ciclo próprio: use os componentes e estilos existentes (`Panel`, `DataTable`, `Tag`, `SegmentedControl`, `EmptyState`, `FrozenTextNotice`, `formStyles`), sem redesign.
- Testes que dependem de "hoje" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(new Date("2026-10-05T10:00:00-03:00"))` (segunda-feira) e `vi.useRealTimers()` no `afterEach`. Funções puras recebem `now`/`today` como argumento.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

- Crie o worktree a partir de `origin/main` e ligue o `node_modules` do checkout principal. `/.claude/` já está no `.gitignore` do dashboard.

  ```bash
  cd apps/dashboard
  /opt/homebrew/bin/git fetch origin
  /opt/homebrew/bin/git worktree add .claude/mod17 -b feat/mod-17-scheduling origin/main
  ln -s ../../node_modules .claude/mod17/node_modules
  ```

- Testes (script `test` = `vitest run`): `cd apps/dashboard/.claude/mod17 && npx vitest run <arquivos>`.
- Tipos antes de cada commit (script `typecheck` = `tsc --noEmit`; o `tsconfig` inclui os testes): `cd apps/dashboard/.claude/mod17 && npx tsc --noEmit`.
- Base do ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBeTruthy()`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`.
- O api do módulo 17 roda em dev na porta **3034**; a prova no navegador aponta o proxy para ela (Task 13).

### Conflito com o módulo 16 (em execução em outra sessão)

O módulo 16 (`feat/mod-16-record-mode`) não é dependência deste plano. Os dois tocam os mesmos arquivos do dashboard:

| Arquivo | Módulo 16 | Este plano | Como resolver no rebase |
|---|---|---|---|
| `src/lib/api.ts` | `features` em `SessionUser`; seção nova no fim; `verifyCitizen` | seção nova no fim; `GateResult`/`gateProtocol`; `ProfessionalShift`, `ProfessionalLink`, `scheduleShift`, `RequestRow`; some `scheduleRequest`/`listUnitAgenda` | manter as duas seções do fim, uma depois da outra; os trechos do meio não se cruzam |
| `src/lib/attendance.ts` | mensagens do CADSUS | mensagens do agendamento | unir as duas listas em `MESSAGES` |
| `src/modules/Attendance.tsx` | painel `CadsusCheck` no balcão | painel `UnassignedRequests` | manter os dois painéis |
| `src/shell/modules.ts`, `modules.test.ts`, `src/App.tsx` | grupo "e-SUS" e rotas | item `my-agenda` e rota | manter os dois; refazer as contagens de `NAV_GROUPS.length` no teste se o 16 mudou o número de grupos (este plano não cria grupo) |

Quem chegar depois a `origin/main` faz `git rebase origin/main`, resolve como na tabela, roda `npx vitest run && npx tsc --noEmit` e só então segue. A validação presencial (`ProfileCheck`, `verifyCitizen`) não é tocada aqui.

## Review Focus

1. **Vaga que deixa de existir entre a leitura e o clique** (outra recepção marcou, o relógio passou do início, o dia ganhou turno). A tela não pode marcar outra vaga sozinha nem fechar o painel: avisa, recarrega as vagas e deixa a pessoa escolher de novo. Testes:
   - Task 8, "slot_taken recarrega as vagas, avisa e não marca outra";
   - Task 8, "use_slots num dia de marcação livre recarrega e some a marcação livre".
2. **Dia da cidade × dia do navegador.** Vaga às 23h30 de São Paulo é 02h30Z do dia seguinte; ela tem que aparecer no dia certo da cidade, e a semana da Minha agenda começa na segunda da cidade. Testes:
   - Task 2, "vaga das 23h30 fica no dia da cidade" e "semana começa na segunda";
   - Task 12, "semana pede de segunda a domingo".
3. **Tipo desativado em uso.** Faixa de modelo ou regra de protocolo com tipo que a cidade desativou continua aparecendo com o nome e "(inativo)", não some do select nem é trocada em silêncio. No modelo, a tela avisa antes do 422 `inactive_type` e trava o salvar até a pessoa trocar o tipo; na regra de protocolo é só aviso. Testes:
   - Task 2, "tipo inativo é inactive_type, não unknown_type";
   - Task 4, "faixa com tipo inativo continua no select marcada e pede troca";
   - Task 6, "regra com tipo inativo ou desconhecido continua no select".
4. **Contador de encaixe velho.** A recepção abre o encaixe com 1 de 2, outra recepção faz o segundo; o 409 `fit_in_limit` recarrega a agenda, o contador passa a 2 de 2 e o botão trava. Testes:
   - Task 9, "fit_in_limit recarrega o contador e trava o botão".
5. **Lista de tipos indisponível no editor.** Autor e revisor leem os tipos (§9), mas outro papel com acesso ao editor, ou uma falha de rede, recebe erro. O painel "Agendamento" não pode quebrar nem bloquear: vira campo de texto com o padrão da chave, e o gate avisa tipo inexistente. Testes:
   - Task 6, "sem lista de tipos, o tipo é texto livre com o padrão da chave".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `src/lib/api.ts`, `src/lib/api.scheduling.test.ts` | tipos do contrato; cliente de tipos, modelos, pré-visualização, turno, vínculo, Minha agenda, vagas, marcação, agenda nova, "sem unidade"; avisos do gate | 1 |
| `src/lib/scheduling.ts`, `src/lib/scheduling.test.ts`, `src/test/schedulingFixtures.ts`, `src/lib/attendance.ts`, `src/lib/professionals.ts` | rótulos, validação de tipo e faixas, aviso de confirmação, dias/semana, marcas da fila; mensagens das recusas | 2 |
| `src/modules/professionals/AppointmentTypes.tsx`, `src/modules/Professionals.tsx` | aba "Tipos de atendimento" | 3 |
| `src/modules/professionals/ScheduleTemplates.tsx`, `src/modules/professionals/TemplateEditor.tsx`, `src/modules/Professionals.tsx` | aba "Modelos de agenda": lista, editor de faixas, pré-visualização | 4 |
| `src/modules/professionals/ProfessionalDetail.tsx` | modelo no turno (ao lançar e depois) e tipo padrão no vínculo | 5 |
| `src/lib/schedulingRules.ts`, `src/modules/protocolEditor/SchedulingPanel.tsx`, `src/modules/ProtocolEditor.tsx` | painel "Agendamento" do protocolo; avisos do gate | 6 |
| `src/lib/api.ts` (`RequestRow`, `RequestDetail`, `getRequest`), `src/modules/attendance/Requests.tsx`, `src/modules/attendance/RequestDetailPanel.tsx` | fila com marcas e ordem do api; detalhe do pedido | 7 |
| `src/modules/attendance/BookPanel.tsx`, `src/modules/attendance/Requests.tsx`, `src/lib/api.ts` (some `scheduleRequest`) | marcar em vaga e marcação livre em `legacy_days` | 8 |
| `src/modules/attendance/FitInPanel.tsx`, `src/modules/attendance/BookPanel.tsx` | encaixe com justificativa e contador | 9 |
| `src/modules/attendance/Agenda.tsx`, `src/lib/api.ts` (some `listUnitAgenda`) | agenda da unidade por dia e profissional | 10 |
| `src/modules/attendance/UnassignedRequests.tsx`, `src/modules/Attendance.tsx` | fila "sem unidade" com atribuição | 11 |
| `src/modules/MyAgenda.tsx`, `src/shell/modules.ts`, `src/App.tsx` | Minha agenda (dia e semana) | 12 |
| — | suíte, build, revisão e prova no navegador | 13 |

**Estratégia de teste:** regras puras com tabela de casos (Task 2 e Task 6), cliente HTTP com `fetch` falso conferindo URL, método e corpo (Task 1), e cada tela com `vi.mock("../lib/api")` cobrindo o caminho feliz, cada recusa que muda o comportamento da tela e as bordas do Review Focus. Relógio fixo em todo teste que depende de "hoje". A prova final é no navegador, contra o api do módulo 17 (Task 13).

---

### Task 1: Cliente HTTP do módulo 17 e tipos do contrato

**Files:**
- Modify: `src/lib/api.ts` (`GateResult` e `gateProtocol`, linhas 192–212; `ProfessionalLink` e `ProfessionalShift`, linhas 634–654; `scheduleShift`, linhas 718–721; seção nova no fim do arquivo)
- Test: `src/lib/api.scheduling.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `PROFESSIONALS_BASE`, `ATTENDANCE_BASE`, `AUTHORING_BASE`, `RequestRow`, `ScheduledAppointment` (todos já em `api.ts`).
- Produces:
  - tipos: `AppointmentTypeOrigin`, `AppointmentType`, `AppointmentTypeFields`, `NewAppointmentType`, `BlockKind`, `ScheduleBlock`, `ScheduleTemplate`, `ScheduleTemplateFields`, `NewScheduleTemplate`, `TemplatePreviewInput`, `TemplatePreview`, `SchedulingPriority`, `PreferredPeriod`, `RescheduleReasonCode`, `BookingKind`, `AppointmentView`, `AgendaShift`, `AgendaProfessional`, `UnitAgenda`, `AvailabilitySlot`, `Availability`, `BookingInput`, `MyAgendaShift`, `MyAgendaDay`, `MyAgenda`;
  - `GateResult.warnings?: string[]`; `ProfessionalLink.default_appointment_type_key?: string | null`; `ProfessionalShift.schedule_template_id?: string | null`;
  - funções: `listAppointmentTypes(): Promise<AppointmentType[]>`, `createAppointmentType(input: NewAppointmentType): Promise<AppointmentType>`, `updateAppointmentType(key: string, fields: AppointmentTypeFields): Promise<AppointmentType>`, `listScheduleTemplates(): Promise<ScheduleTemplate[]>`, `createScheduleTemplate(input: NewScheduleTemplate): Promise<ScheduleTemplate>`, `updateScheduleTemplate(id: string, fields: ScheduleTemplateFields): Promise<ScheduleTemplate>`, `previewScheduleTemplate(input: TemplatePreviewInput): Promise<TemplatePreview>`, `scheduleShift(linkId, startsAt, endsAt, templateId?: string | null): Promise<ProfessionalShift>`, `setShiftTemplate(shiftId: string, templateId: string | null): Promise<void>`, `setLinkDefaultType(linkId: string, key: string | null): Promise<void>`, `getMyAgenda(from: string, to: string): Promise<MyAgenda>`, `listUnassignedRequests(): Promise<RequestRow[]>`, `assignRequestUnit(id: string, unitId: string): Promise<RequestRow>`, `getUnitAvailability(unitId: string, type: string, from: string, to: string): Promise<Availability>`, `bookAppointment(requestId: string, unitId: string, input: BookingInput): Promise<ScheduledAppointment>`, `getUnitAgenda(unitId: string, date: string): Promise<UnitAgenda>`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.scheduling.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import {
  ApiError, assignRequestUnit, bookAppointment, createAppointmentType, createScheduleTemplate, gateProtocol, getMyAgenda,
  getUnitAgenda, getUnitAvailability, listAppointmentTypes, listScheduleTemplates, listUnassignedRequests,
  previewScheduleTemplate, scheduleShift, setLinkDefaultType, setShiftTemplate, updateAppointmentType, updateScheduleTemplate,
  type AppointmentType, type ScheduleTemplate
} from "./api";

afterEach(() => vi.unstubAllGlobals());

function stub(body: unknown, status = 200) {
  const fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
    new Response(body === undefined ? null : JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fn);
  return fn;
}
const call = (fn: ReturnType<typeof stub>, i = 0) => fn.mock.calls[i] as [ string, RequestInit ];
const sent = (fn: ReturnType<typeof stub>, i = 0) => JSON.parse(call(fn, i)[1].body as string);

const TYPE: AppointmentType = { key: "consulta_medica", name: "Consulta médica", duration_minutes: 20,
  cbo_prefixes: [ "2251", "2252", "2253" ], active: true, origin: "platform" };
const TEMPLATE: ScheduleTemplate = { id: "t1", name: "Manhã", fit_in_limit: 2, active: true,
  blocks: [ { starts: "09:00", ends: "11:00", kind: "bookable", appointment_type_key: "consulta_medica" } ] };

describe("cliente do módulo 17 — Profissionais", () => {
  it("tipos: lista desembrulha { types }; criar e alterar devolvem o tipo puro", async () => {
    let fn = stub({ types: [ TYPE ] });
    expect((await listAppointmentTypes())[0].key).toBe("consulta_medica");
    expect(call(fn)[0]).toBe("/professionals/appointment_types");
    expect(call(fn)[1].credentials).toBe("include");

    fn = stub({ ...TYPE, key: "puericultura", origin: "city" });
    const created = await createAppointmentType({ key: "puericultura", name: "Puericultura", duration_minutes: 30, cbo_prefixes: [ "2235" ] });
    expect(created.origin).toBe("city");
    expect(call(fn)[1].method).toBe("POST");
    expect(sent(fn)).toEqual({ key: "puericultura", name: "Puericultura", duration_minutes: 30, cbo_prefixes: [ "2235" ] });

    fn = stub(TYPE);
    await updateAppointmentType("consulta/medica", { active: false });
    expect(call(fn)[0]).toBe("/professionals/appointment_types/consulta%2Fmedica");
    expect(sent(fn)).toEqual({ active: false });
  });

  it("modelos: lista, criar, alterar e pré-visualizar", async () => {
    let fn = stub({ templates: [ TEMPLATE ] });
    expect((await listScheduleTemplates())[0].name).toBe("Manhã");
    expect(call(fn)[0]).toBe("/professionals/schedule_templates");

    fn = stub(TEMPLATE);
    await createScheduleTemplate({ name: "Manhã", fit_in_limit: 2, blocks: TEMPLATE.blocks });
    expect(call(fn)[0]).toBe("/professionals/schedule_templates");
    expect(sent(fn)).toEqual({ name: "Manhã", fit_in_limit: 2, blocks: TEMPLATE.blocks });

    fn = stub(TEMPLATE);
    await updateScheduleTemplate("t1", { active: false });
    expect(call(fn)[0]).toBe("/professionals/schedule_templates/t1");

    fn = stub({ slots: [ { starts_at: "2026-10-06T09:00:00-03:00", ends_at: "2026-10-06T09:20:00-03:00", appointment_type_key: "consulta_medica" } ], blocks: TEMPLATE.blocks });
    const preview = await previewScheduleTemplate({ blocks: TEMPLATE.blocks, fit_in_limit: 2,
      sample: { starts_at: "2026-10-06T07:00:00-03:00", ends_at: "2026-10-06T13:00:00-03:00", cbo_code: "225125" } });
    expect(preview.slots).toHaveLength(1);
    expect(call(fn)[0]).toBe("/professionals/schedule_templates/preview");
    expect(sent(fn).sample.cbo_code).toBe("225125");
  });

  it("turno: lançar manda o modelo só quando informado; trocar o modelo aceita null", async () => {
    let fn = stub({ shift: { id: "s1" } });
    await scheduleShift("l1", "a", "b");
    expect(sent(fn)).toEqual({ starts_at: "a", ends_at: "b" });

    fn = stub({ shift: { id: "s1" } });
    await scheduleShift("l1", "a", "b", "t1");
    expect(sent(fn)).toEqual({ starts_at: "a", ends_at: "b", schedule_template_id: "t1" });

    fn = stub({ id: "s1" });
    await setShiftTemplate("s1", null);
    expect(call(fn)[0]).toBe("/professionals/shifts/s1/template");
    expect(sent(fn)).toEqual({ schedule_template_id: null });
  });

  it("vínculo: tipo padrão; Minha agenda com from e to na query", async () => {
    let fn = stub({ id: "l1" });
    await setLinkDefaultType("l1", "consulta_medica");
    expect(call(fn)[0]).toBe("/professionals/links/l1/default_type");
    expect(sent(fn)).toEqual({ appointment_type_key: "consulta_medica" });

    fn = stub({ days: [] });
    await getMyAgenda("2026-10-05", "2026-10-11");
    expect(call(fn)[0]).toBe("/professionals/me/agenda?from=2026-10-05&to=2026-10-11");
  });
});

describe("cliente do módulo 17 — Atendimento", () => {
  it("vagas: tipo e período na query; legacy_days vem junto", async () => {
    const fn = stub({ slots: [], legacy_days: [ "2026-10-07" ] });
    const result = await getUnitAvailability("u1", "consulta_medica", "2026-10-05", "2026-10-18");
    expect(result.legacy_days).toEqual([ "2026-10-07" ]);
    expect(call(fn)[0]).toBe("/attendance/units/u1/availability?type=consulta_medica&from=2026-10-05&to=2026-10-18");
  });

  it("marcar: as três formas levam a unidade e desembrulham { appointment }", async () => {
    const appointment = { id: "a1", scheduled_at: "x", status: "scheduled", confirmation_deadline_at: null };
    let fn = stub({ appointment }, 201);
    expect((await bookAppointment("r1", "u1", { kind: "slot", professional_id: "p1",
      starts_at: "2026-10-06T09:00:00-03:00", appointment_type_key: "consulta_medica" })).id).toBe("a1");
    expect(call(fn)[0]).toBe("/attendance/requests/r1/appointments");
    expect(sent(fn)).toEqual({ kind: "slot", professional_id: "p1", starts_at: "2026-10-06T09:00:00-03:00",
      appointment_type_key: "consulta_medica", health_unit_id: "u1" });

    fn = stub({ appointment }, 201);
    await bookAppointment("r1", "u1", { kind: "fit_in", professional_id: "p1", shift_id: "s1",
      starts_at: "2026-10-06T10:10:00-03:00", appointment_type_key: "consulta_medica", reason: "gestante com dor" });
    expect(sent(fn).reason).toBe("gestante com dor");

    fn = stub({ appointment }, 201);
    await bookAppointment("r1", "u1", { kind: "legacy", scheduled_at: "2026-10-07T12:00:00.000Z", allow_overlap: true });
    expect(sent(fn)).toEqual({ kind: "legacy", scheduled_at: "2026-10-07T12:00:00.000Z", allow_overlap: true, health_unit_id: "u1" });
  });

  it("marcar: 409 slot_taken vira ApiError com o código", async () => {
    stub({ error: "slot_taken" }, 409);
    const err = await bookAppointment("r1", "u1", { kind: "slot", professional_id: "p1", starts_at: "x", appointment_type_key: "k" })
      .catch((e: unknown) => e);
    expect(err).toBeInstanceOf(ApiError);
    expect((err as ApiError).body).toEqual({ error: "slot_taken" });
  });

  it("agenda nova é o objeto puro; sem unidade e atribuição", async () => {
    let fn = stub({ date: "2026-10-06", professionals: [], unassigned: [] });
    expect((await getUnitAgenda("u1", "2026-10-06")).date).toBe("2026-10-06");
    expect(call(fn)[0]).toBe("/attendance/units/u1/agenda?date=2026-10-06");

    fn = stub({ requests: [] });
    expect(await listUnassignedRequests()).toEqual([]);
    expect(call(fn)[0]).toBe("/attendance/requests/unassigned");

    fn = stub({ id: "r1" });
    await assignRequestUnit("r1", "u2");
    expect(call(fn)[0]).toBe("/attendance/requests/r1/assign_unit");
    expect(sent(fn)).toEqual({ unit_id: "u2" });
  });
});

describe("gate com avisos", () => {
  it("200 sem corpo continua { valid: true }; 200 com warnings devolve os avisos", async () => {
    stub(undefined, 200);
    expect(await gateProtocol({})).toEqual({ valid: true });
    stub({ warnings: [ "scheduling[0].appointment_type: tipo inexistente na cidade (puericultura)" ] });
    expect(await gateProtocol({})).toEqual({
      valid: true, warnings: [ "scheduling[0].appointment_type: tipo inexistente na cidade (puericultura)" ]
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/lib/api.scheduling.test.ts`
Expected: FAIL — `listAppointmentTypes` (e as outras) não são exportadas por `./api`.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`, troque `GateResult` e `gateProtocol` (linhas 192–212):

```ts
// Módulo 17 (contratos §1): o gate pode aceitar com avisos (tipo de
// atendimento inexistente na cidade). Sem avisos, continua { valid: true }.
export interface GateResult { valid: boolean; errors?: string[]; warnings?: string[]; }
```

```ts
export async function gateProtocol(definition: unknown): Promise<GateResult> {
  try {
    const body = await jsonFetch<{ warnings?: unknown } | null>(`${AUTHORING_BASE}/gate`, {
      method: "POST", body: JSON.stringify({ definition })
    }).catch((err: unknown) => {
      // 200 sem corpo JSON: res.json() falha, e isso não é recusa.
      if (err instanceof SyntaxError) return null;
      throw err;
    });
    const warnings = Array.isArray(body?.warnings)
      ? body.warnings.filter((w): w is string => typeof w === "string") : [];
    return warnings.length > 0 ? { valid: true, warnings } : { valid: true };
  } catch (err) {
    if (err instanceof ApiError && err.status === 422) {
      const body = (err.body ?? {}) as GateResult;
      return { valid: false, errors: body.errors ?? [] };
    }
    throw err;
  }
}
```

Em `ProfessionalLink` (depois de `ended_by`) e `ProfessionalShift` (depois de `cancel_reason`):

```ts
  // Módulo 17: tipo padrão do vínculo para turno sem modelo (nulo = o da base pelo CBO).
  default_appointment_type_key?: string | null;
```

```ts
  // Módulo 17: modelo de agenda do turno (nulo = turno inteiro como vagas do tipo padrão).
  schedule_template_id?: string | null;
```

Troque `scheduleShift` (linhas 718–721):

```ts
export async function scheduleShift(
  linkId: string, startsAt: string, endsAt: string, templateId?: string | null
): Promise<ProfessionalShift> {
  const body: Record<string, unknown> = { starts_at: startsAt, ends_at: endsAt };
  if (templateId) body.schedule_template_id = templateId;
  return (await jsonFetch<{ shift: ProfessionalShift }>(`${PROFESSIONALS_BASE}/links/${professionalId(linkId)}/shifts`,
    postProfessional(body))).shift;
}
```

No fim do arquivo, a seção nova:

```ts
// ─── Agenda (módulo 17, ADR 0029; contratos 2026-10-05 §2–§4) ───────────────
// Rotas NOVAS devolvem o objeto puro na escrita (contratos, cabeçalho); as que
// já existiam mantêm o envelope de hoje ({ shift }, { appointment }, { requests }).

export type AppointmentTypeOrigin = "platform" | "city";
export interface AppointmentType {
  key: string; name: string; duration_minutes: number; cbo_prefixes: string[]; active: boolean; origin: AppointmentTypeOrigin;
}
export interface AppointmentTypeFields { name?: string; duration_minutes?: number; cbo_prefixes?: string[]; active?: boolean }
export interface NewAppointmentType { key: string; name: string; duration_minutes: number; cbo_prefixes: string[] }

export type BlockKind = "walk_in" | "bookable" | "blocked";
// `appointment_type_name` só vem nas respostas (§9: faixas `bookable` da agenda
// e da Minha agenda); nunca é enviado de volta.
export interface ScheduleBlock {
  starts: string; ends: string; kind: BlockKind; appointment_type_key?: string; slot_minutes?: number;
  appointment_type_name?: string;
}
export interface ScheduleTemplate { id: string; name: string; fit_in_limit: number; blocks: ScheduleBlock[]; active: boolean }
export interface ScheduleTemplateFields { name?: string; fit_in_limit?: number; blocks?: ScheduleBlock[]; active?: boolean }
export interface NewScheduleTemplate { name: string; fit_in_limit: number; blocks: ScheduleBlock[] }
export interface TemplatePreviewInput {
  blocks: ScheduleBlock[]; fit_in_limit: number; sample: { starts_at: string; ends_at: string; cbo_code: string };
}
export interface TemplatePreview {
  slots: { starts_at: string; ends_at: string; appointment_type_key: string }[];
  blocks: ScheduleBlock[];
}

export type SchedulingPriority = "routine" | "priority";
export type PreferredPeriod = "morning" | "afternoon" | "any";
export type RescheduleReasonCode = "work" | "health" | "transport" | "other";
export type BookingKind = "slot" | "fit_in" | "legacy";

// Horário (contratos §4.4), a mesma forma na agenda, na fila e na Minha agenda.
// Em `legacy`, fim, tipo, profissional e turno vêm nulos.
export interface AppointmentView {
  id: string; status: string; booking_kind: BookingKind; scheduled_at: string; ends_at: string | null;
  appointment_type_key: string | null; appointment_type_name: string | null;
  professional: { id: string; name: string } | null; shift_id: string | null; fit_in: boolean;
  fit_in_reason?: string; outside_template: boolean; shift_cancelled: boolean;
  citizen: { id: string; cpf_masked: string; name?: string };
}

export interface AgendaShift {
  shift_id: string; starts_at: string; ends_at: string; blocks: ScheduleBlock[];
  fit_in_count: number; fit_in_limit: number;
  // Turno cancelado continua na agenda (spec §4.3; contratos §9).
  cancelled_at: string | null;
}
export interface AgendaProfessional { id: string; name: string; shifts: AgendaShift[]; appointments: AppointmentView[] }
export interface UnitAgenda { date: string; professionals: AgendaProfessional[]; unassigned: AppointmentView[] }

export interface AvailabilitySlot {
  professional_id: string; professional_name: string; shift_id: string; starts_at: string; ends_at: string;
}
export interface Availability { slots: AvailabilitySlot[]; legacy_days: string[] }

export type BookingInput =
  | { kind: "slot"; professional_id: string; starts_at: string; appointment_type_key: string }
  | { kind: "fit_in"; professional_id: string; shift_id: string; starts_at: string; appointment_type_key: string; reason: string }
  | { kind: "legacy"; scheduled_at: string; allow_overlap?: boolean };

export interface MyAgendaShift {
  shift_id: string; unit: { id: string; name: string }; starts_at: string; ends_at: string; cancelled_at: string | null;
  blocks: ScheduleBlock[]; appointments: AppointmentView[];
}
export interface MyAgendaDay { date: string; shifts: MyAgendaShift[] }
export interface MyAgenda { days: MyAgendaDay[] }

const postJson = (body: unknown): RequestInit => ({ method: "POST", body: JSON.stringify(body) });
const seg = encodeURIComponent;

export async function listAppointmentTypes(): Promise<AppointmentType[]> {
  return (await jsonFetch<{ types: AppointmentType[] }>(`${PROFESSIONALS_BASE}/appointment_types`)).types;
}

export function createAppointmentType(input: NewAppointmentType): Promise<AppointmentType> {
  return jsonFetch(`${PROFESSIONALS_BASE}/appointment_types`, postJson(input));
}

export function updateAppointmentType(key: string, fields: AppointmentTypeFields): Promise<AppointmentType> {
  return jsonFetch(`${PROFESSIONALS_BASE}/appointment_types/${seg(key)}`, postJson(fields));
}

export async function listScheduleTemplates(): Promise<ScheduleTemplate[]> {
  return (await jsonFetch<{ templates: ScheduleTemplate[] }>(`${PROFESSIONALS_BASE}/schedule_templates`)).templates;
}

export function createScheduleTemplate(input: NewScheduleTemplate): Promise<ScheduleTemplate> {
  return jsonFetch(`${PROFESSIONALS_BASE}/schedule_templates`, postJson(input));
}

export function updateScheduleTemplate(id: string, fields: ScheduleTemplateFields): Promise<ScheduleTemplate> {
  return jsonFetch(`${PROFESSIONALS_BASE}/schedule_templates/${seg(id)}`, postJson(fields));
}

export function previewScheduleTemplate(input: TemplatePreviewInput): Promise<TemplatePreview> {
  return jsonFetch(`${PROFESSIONALS_BASE}/schedule_templates/preview`, postJson(input));
}

export async function setShiftTemplate(shiftId: string, templateId: string | null): Promise<void> {
  await jsonFetch<unknown>(`${PROFESSIONALS_BASE}/shifts/${seg(shiftId)}/template`, postJson({ schedule_template_id: templateId }));
}

export async function setLinkDefaultType(linkId: string, key: string | null): Promise<void> {
  await jsonFetch<unknown>(`${PROFESSIONALS_BASE}/links/${seg(linkId)}/default_type`, postJson({ appointment_type_key: key }));
}

export function getMyAgenda(from: string, to: string): Promise<MyAgenda> {
  return jsonFetch(`${PROFESSIONALS_BASE}/me/agenda?${new URLSearchParams({ from, to }).toString()}`);
}

export async function listUnassignedRequests(): Promise<RequestRow[]> {
  return (await jsonFetch<{ requests: RequestRow[] }>(`${ATTENDANCE_BASE}/requests/unassigned`)).requests;
}

export function assignRequestUnit(id: string, unitId: string): Promise<RequestRow> {
  return jsonFetch(`${ATTENDANCE_BASE}/requests/${seg(id)}/assign_unit`, postJson({ unit_id: unitId }));
}

export function getUnitAvailability(unitId: string, type: string, from: string, to: string): Promise<Availability> {
  const qs = new URLSearchParams({ type, from, to }).toString();
  return jsonFetch(`${ATTENDANCE_BASE}/units/${seg(unitId)}/availability?${qs}`);
}

// A unidade vai em toda forma: o api confere `wrong_unit` como hoje.
export async function bookAppointment(requestId: string, unitId: string, input: BookingInput): Promise<ScheduledAppointment> {
  const payload = await jsonFetch<{ appointment: ScheduledAppointment }>(
    `${ATTENDANCE_BASE}/requests/${seg(requestId)}/appointments`, postJson({ ...input, health_unit_id: unitId })
  );
  return payload.appointment;
}

export function getUnitAgenda(unitId: string, date: string): Promise<UnitAgenda> {
  return jsonFetch(`${ATTENDANCE_BASE}/units/${seg(unitId)}/agenda?date=${seg(date)}`);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/lib/api.scheduling.test.ts src/lib/api.authoring.test.ts src/modules/ProtocolEditor.test.tsx && npx tsc --noEmit`
Expected: PASS; `tsc` sem erro. Os testes antigos do gate continuam verdes (`{ valid: true }` sem avisos).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod17
/opt/homebrew/bin/git add src/lib/api.ts src/lib/api.scheduling.test.ts
/opt/homebrew/bin/git commit -m "feat: add module 17 scheduling client and contract types

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras puras de agenda e mensagens das recusas

**Files:**
- Create: `src/lib/scheduling.ts`, `src/test/schedulingFixtures.ts`
- Modify: `src/lib/attendance.ts` (mapa `MESSAGES`), `src/lib/professionals.ts` (`MESSAGES`, `ProfessionalErrorBody`, `translateProfessionalErrorBody`)
- Test: `src/lib/scheduling.test.ts`

**Interfaces:**
- Consumes: tipos da Task 1; `cityIsoDate`, `cityDateFormat`, `fmtHourMinute` de `./format`.
- Produces (`src/lib/scheduling.ts`):
  - constantes `TYPE_KEY_PATTERN`, `DURATION_MIN = 5`, `DURATION_MAX = 240`, `FIT_IN_LIMIT_MAX = 20`, `FIT_IN_REASON_MIN = 10`, `AVAILABILITY_DAYS = 14`, `MY_AGENDA_DAYS = 7`;
  - rótulos `BLOCK_KIND_LABEL`, `PRIORITY_LABEL`, `PERIOD_LABEL`, `REASON_CODE_LABEL`, `BOOKING_KIND_LABEL`, `BLOCK_DETAIL_MESSAGE`;
  - `type BlockProblem = "overlap" | "missing_type" | "unknown_type" | "inactive_type" | "bad_time" | "crosses_midnight" | "bad_slot_minutes" | "empty"` (os `detail` de `invalid_blocks`, contratos §3 e §9);
  - `blockProblem(block: ScheduleBlock, types: AppointmentType[]): BlockProblem | null`;
  - `overlappingBlocks(blocks: ScheduleBlock[]): Set<number>`;
  - `parseFitInLimit(text: string): number | null`;
  - `interface TemplateDraft { name: string; fitInLimit: string; blocks: ScheduleBlock[] }`, `templateDraftFrom(t: ScheduleTemplate | null): TemplateDraft`, `templateProblem(draft: TemplateDraft, types: AppointmentType[]): string | null`;
  - `interface TypeDraft { key: string; name: string; duration: string; cbo: string }`, `typeDraftFrom(t: AppointmentType | null): TypeDraft`, `parseCboPrefixes(text: string): string[] | null`, `typeDraftProblem(draft: TypeDraft, mode: "create" | "edit"): string | null`, `typeLabel(key: string | null | undefined, types: AppointmentType[] | null): string` (`null` = lista de tipos indisponível: mostra a chave);
  - `blockLine(block: ScheduleBlock, types: AppointmentType[] | null): string`;
  - `confirmationWarning(start: Date, now: Date): string`;
  - `addDaysIso(iso: string, n: number): string`, `weekStart(iso: string): string`, `dayLabel(iso: string): string`, `daysBetween(from: string, to: string): string[]`;
  - `slotsByDay(slots: AvailabilitySlot[]): Map<string, AvailabilitySlot[]>`;
  - `fmtDueOn(iso: string): string`;
  - `requestMarks(row: QueueMarksInput): { label: string; tone: "down" | "warn" | "info" | "neutral" }[]` com `QueueMarksInput` = `{ overdue: boolean; reschedule_requested: boolean; needs_reschedule: boolean; reopened_reason: "expired" | "no_show" | null }`;
  - `appointmentFlags(a: AppointmentView): string[]`.
- Produces (fixtures): `TYPES`, `MORNING`, `slot(over)`, `appointmentView(over)`.

- [ ] **Step 1: Write the fixtures and the failing test**

```ts
// src/test/schedulingFixtures.ts
// Dados comuns aos testes do módulo 17. Relógio dos testes: segunda,
// 2026-10-05 10:00 em São Paulo (-03:00).
import type { AppointmentType, AppointmentView, AvailabilitySlot, ScheduleTemplate } from "../lib/api";

export const NOW = "2026-10-05T10:00:00-03:00";

export const TYPES: AppointmentType[] = [
  { key: "consulta_medica", name: "Consulta médica", duration_minutes: 20, cbo_prefixes: [ "2251", "2252", "2253" ], active: true, origin: "platform" },
  { key: "consulta_enfermagem", name: "Consulta de enfermagem", duration_minutes: 15, cbo_prefixes: [ "2235" ], active: true, origin: "platform" },
  { key: "retorno", name: "Retorno", duration_minutes: 15, cbo_prefixes: [ "2251", "2252", "2253", "2235", "2232" ], active: true, origin: "platform" },
  { key: "puericultura", name: "Puericultura", duration_minutes: 30, cbo_prefixes: [ "2235" ], active: false, origin: "city" }
];

export const MORNING: ScheduleTemplate = {
  id: "t1", name: "Manhã", fit_in_limit: 2, active: true,
  blocks: [
    { starts: "07:00", ends: "09:00", kind: "walk_in" },
    { starts: "09:00", ends: "11:00", kind: "bookable", appointment_type_key: "consulta_medica" },
    { starts: "11:00", ends: "12:00", kind: "blocked" }
  ]
};

export function slot(over: Partial<AvailabilitySlot> = {}): AvailabilitySlot {
  return { professional_id: "p1", professional_name: "Helena Duarte", shift_id: "s1",
    starts_at: "2026-10-06T09:00:00-03:00", ends_at: "2026-10-06T09:20:00-03:00", ...over };
}

export function appointmentView(over: Partial<AppointmentView> = {}): AppointmentView {
  return {
    id: "a1", status: "confirmed", booking_kind: "slot", scheduled_at: "2026-10-06T09:00:00-03:00",
    ends_at: "2026-10-06T09:20:00-03:00", appointment_type_key: "consulta_medica", appointment_type_name: "Consulta médica",
    professional: { id: "p1", name: "Helena Duarte" }, shift_id: "s1", fit_in: false, outside_template: false,
    shift_cancelled: false, citizen: { id: "c1", cpf_masked: "***.982.247-**" }, ...over
  };
}
```

```ts
// src/lib/scheduling.test.ts
import { describe, expect, it } from "vitest";
import {
  addDaysIso, appointmentFlags, blockLine, blockProblem, confirmationWarning, dayLabel, daysBetween, fmtDueOn,
  overlappingBlocks, parseCboPrefixes, parseFitInLimit, requestMarks, slotsByDay, templateDraftFrom, templateProblem,
  typeDraftFrom, typeDraftProblem, typeLabel, weekStart
} from "./scheduling";
import { appointmentView, MORNING, slot, TYPES } from "../test/schedulingFixtures";

describe("faixas do modelo", () => {
  it.each([
    [ { starts: "09:00", ends: "11:00", kind: "bookable", appointment_type_key: "consulta_medica" }, null ],
    [ { starts: "07:00", ends: "09:00", kind: "walk_in" }, null ],
    [ { starts: "11:00", ends: "11:00", kind: "blocked" }, "crosses_midnight" ],
    [ { starts: "22:00", ends: "02:00", kind: "blocked" }, "crosses_midnight" ],
    [ { starts: "23:00", ends: "24:00", kind: "walk_in" }, "crosses_midnight" ],
    [ { starts: "7:00", ends: "09:00", kind: "walk_in" }, "bad_time" ],
    [ { starts: "09:00", ends: "25:00", kind: "walk_in" }, "bad_time" ],
    [ { starts: "09:00", ends: "11:00", kind: "bookable" }, "missing_type" ],
    [ { starts: "09:00", ends: "11:00", kind: "bookable", appointment_type_key: "nao_existe" }, "unknown_type" ],
    [ { starts: "09:00", ends: "11:00", kind: "bookable", appointment_type_key: "consulta_medica", slot_minutes: 4 }, "bad_slot_minutes" ],
    [ { starts: "09:00", ends: "11:00", kind: "bookable", appointment_type_key: "consulta_medica", slot_minutes: 241 }, "bad_slot_minutes" ]
  ] as const)("%j → %s", (block, expected) => {
    expect(blockProblem(block, TYPES)).toBe(expected);
  });

  it("tipo inativo é inactive_type, não unknown_type", () => {
    expect(blockProblem({ starts: "09:00", ends: "11:00", kind: "bookable", appointment_type_key: "puericultura" }, TYPES))
      .toBe("inactive_type");
  });

  it("sobreposição marca as duas faixas; encostar não é sobrepor", () => {
    expect([ ...overlappingBlocks(MORNING.blocks) ]).toEqual([]);
    const blocks = [ ...MORNING.blocks, { starts: "10:30", ends: "11:30", kind: "blocked" as const } ];
    expect([ ...overlappingBlocks(blocks) ].sort()).toEqual([ 1, 2, 3 ]);
  });

  it("limite de encaixes: inteiro de 0 a 20", () => {
    expect(parseFitInLimit("0")).toBe(0);
    expect(parseFitInLimit("20")).toBe(20);
    expect(parseFitInLimit("21")).toBeNull();
    expect(parseFitInLimit("1,5")).toBeNull();
    expect(parseFitInLimit("")).toBeNull();
  });

  it("modelo: primeiro problema em frase", () => {
    const ok = templateDraftFrom(MORNING);
    expect(templateProblem(ok, TYPES)).toBeNull();
    expect(templateProblem({ ...ok, name: "  " }, TYPES)).toBe("dê um nome ao modelo");
    expect(templateProblem({ ...ok, fitInLimit: "30" }, TYPES)).toBe("limite de encaixes entre 0 e 20");
    expect(templateProblem({ ...ok, blocks: [] }, TYPES)).toBe("inclua pelo menos uma faixa");
    expect(templateProblem({ ...ok, blocks: [ { starts: "19:00", ends: "07:00", kind: "walk_in" } ] }, TYPES))
      .toBe("a faixa não cruza a meia-noite — use fim depois do início (sem 24:00)");
    expect(templateProblem({ ...ok, blocks: [ ...ok.blocks, { starts: "08:00", ends: "09:30", kind: "blocked" } ] }, TYPES))
      .toBe("faixas sobrepostas — ajuste os horários");
    expect(templateProblem({ ...ok, blocks: [ { starts: "09:00", ends: "11:00", kind: "bookable" } ] }, TYPES))
      .toBe("faixa agendável precisa de um tipo de atendimento");
  });

  it("modelo novo começa com uma faixa agendável sem tipo e limite 2", () => {
    expect(templateDraftFrom(null)).toEqual({ name: "", fitInLimit: "2", blocks: [ { starts: "08:00", ends: "12:00", kind: "bookable" } ] });
  });

  it("linha da faixa em frase, com o nome do tipo", () => {
    expect(blockLine(MORNING.blocks[1], TYPES)).toBe("09:00–11:00 · agendável · Consulta médica");
    expect(blockLine(MORNING.blocks[0], TYPES)).toBe("07:00–09:00 · demanda do dia");
    expect(blockLine({ starts: "09:00", ends: "10:00", kind: "bookable", appointment_type_key: "puericultura", slot_minutes: 30 }, TYPES))
      .toBe("09:00–10:00 · agendável · Puericultura (inativo) · vagas de 30 min");
    expect(blockLine(MORNING.blocks[1], null)).toBe("09:00–11:00 · agendável · consulta_medica");
    // Respostas de agenda trazem o nome na faixa (contratos §9): sem lista de tipos, vale o nome.
    expect(blockLine({ ...MORNING.blocks[1], appointment_type_name: "Consulta médica" }, null))
      .toBe("09:00–11:00 · agendável · Consulta médica");
  });
});

describe("tipos de atendimento", () => {
  it("CBOs: lista separada por vírgula ou espaço, só dígitos (1 a 6)", () => {
    expect(parseCboPrefixes("2251, 2252 2253")).toEqual([ "2251", "2252", "2253" ]);
    expect(parseCboPrefixes("")).toBeNull();
    expect(parseCboPrefixes("22a1")).toBeNull();
    expect(parseCboPrefixes("1234567")).toBeNull();
    expect(parseCboPrefixes(Array.from({ length: 20 }, (_, i) => String(2200 + i)).join(","))).toHaveLength(20);
    expect(parseCboPrefixes(Array.from({ length: 21 }, (_, i) => String(2200 + i)).join(","))).toBeNull();
  });

  it("rascunho: chave só na criação, duração 5–240, nome e CBO obrigatórios", () => {
    const draft = { key: "puericultura", name: "Puericultura", duration: "30", cbo: "2235" };
    expect(typeDraftProblem(draft, "create")).toBeNull();
    expect(typeDraftProblem({ ...draft, key: "Puericultura" }, "create")).toBe("chave: minúsculas, números e _, começando por letra (2 a 41)");
    expect(typeDraftProblem({ ...draft, key: "x" }, "edit")).toBeNull();
    expect(typeDraftProblem({ ...draft, name: "" }, "create")).toBe("dê um nome ao tipo");
    expect(typeDraftProblem({ ...draft, duration: "4" }, "create")).toBe("duração entre 5 e 240 minutos");
    expect(typeDraftProblem({ ...draft, cbo: "x" }, "create")).toBe("informe de 1 a 20 grupos de CBO (só números, separados por vírgula)");
    expect(typeDraftFrom(TYPES[0])).toEqual({ key: "consulta_medica", name: "Consulta médica", duration: "20", cbo: "2251, 2252, 2253" });
  });

  it("rótulo do tipo: nome, (inativo), ou a chave quando não existe", () => {
    expect(typeLabel("consulta_medica", TYPES)).toBe("Consulta médica");
    expect(typeLabel("puericultura", TYPES)).toBe("Puericultura (inativo)");
    expect(typeLabel("sumiu", TYPES)).toBe("sumiu (não existe na cidade)");
    expect(typeLabel(null, TYPES)).toBe("—");
    expect(typeLabel("consulta_medica", null)).toBe("consulta_medica");
  });
});

describe("datas no fuso da cidade", () => {
  it("vaga das 23h30 fica no dia da cidade, não no dia UTC", () => {
    const late = slot({ starts_at: "2026-10-07T02:30:00Z", ends_at: "2026-10-07T02:50:00Z" });
    const map = slotsByDay([ slot(), late ]);
    expect([ ...map.keys() ]).toEqual([ "2026-10-06" ]);
    expect(map.get("2026-10-06")).toHaveLength(2);
  });

  it("semana começa na segunda; dias e rótulos", () => {
    expect(weekStart("2026-10-05")).toBe("2026-10-05");
    expect(weekStart("2026-10-11")).toBe("2026-10-05");
    expect(weekStart("2026-10-12")).toBe("2026-10-12");
    expect(addDaysIso("2026-10-31", 1)).toBe("2026-11-01");
    expect(daysBetween("2026-10-05", "2026-10-07")).toEqual([ "2026-10-05", "2026-10-06", "2026-10-07" ]);
    expect(dayLabel("2026-10-06")).toBe("ter 06/10");
    expect(fmtDueOn("2026-10-20")).toBe("até 20/10");
  });

  it("aviso de confirmação: menos de 48h nasce confirmado; senão prazo = horário − 24h", () => {
    const now = new Date("2026-10-05T10:00:00-03:00");
    expect(confirmationWarning(new Date("2026-10-06T09:00:00-03:00"), now)).toBe("O horário nasce confirmado");
    expect(confirmationWarning(new Date("2026-10-08T14:30:00-03:00"), now)).toBe("O cidadão precisa confirmar até 07/10 14:30");
  });
});

describe("fila e agenda", () => {
  it("marcas do pedido, na ordem: atrasado, pediu outro horário, precisa remarcar, reaberto", () => {
    const base = { overdue: false, reschedule_requested: false, needs_reschedule: false, reopened_reason: null };
    expect(requestMarks(base)).toEqual([]);
    expect(requestMarks({ overdue: true, reschedule_requested: true, needs_reschedule: true, reopened_reason: "no_show" }))
      .toEqual([
        { label: "atrasado", tone: "down" },
        { label: "pediu outro horário", tone: "warn" },
        { label: "precisa remarcar", tone: "warn" },
        { label: "faltou", tone: "neutral" }
      ]);
    expect(requestMarks({ ...base, reopened_reason: "expired" })).toEqual([ { label: "sem confirmação", tone: "neutral" } ]);
  });

  it("marcas do horário: encaixe, fora do modelo, turno cancelado", () => {
    expect(appointmentFlags(appointmentView())).toEqual([]);
    expect(appointmentFlags(appointmentView({ fit_in: true, outside_template: true, shift_cancelled: true })))
      .toEqual([ "encaixe", "fora do modelo", "turno cancelado" ]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/lib/scheduling.test.ts`
Expected: FAIL — `Failed to resolve import "./scheduling"`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/scheduling.ts
// Regras do módulo 17 sem React (spec 2026-10-05 §3–§5; contratos §2–§4).
// A validação daqui espelha os 422 do api para a tela avisar antes de enviar;
// quem garante continua sendo o api.
import type {
  AppointmentType, AppointmentView, AvailabilitySlot, BlockKind, BookingKind, PreferredPeriod, RescheduleReasonCode,
  ScheduleBlock, ScheduleTemplate, SchedulingPriority
} from "./api";
import { cityDateFormat, cityIsoDate, fmtHourMinute } from "./format";

export const TYPE_KEY_PATTERN = /^[a-z][a-z0-9_]{1,40}$/;
export const DURATION_MIN = 5;
export const DURATION_MAX = 240;
export const FIT_IN_LIMIT_MAX = 20;
export const CBO_PREFIXES_MAX = 20;
export const FIT_IN_REASON_MIN = 10;
export const AVAILABILITY_DAYS = 14;
export const MY_AGENDA_DAYS = 7;

export const BLOCK_KIND_LABEL: Record<BlockKind, string> = {
  walk_in: "demanda do dia", bookable: "agendável", blocked: "bloqueada"
};
export const PRIORITY_LABEL: Record<SchedulingPriority, string> = { routine: "rotina", priority: "prioritária" };
export const PERIOD_LABEL: Record<PreferredPeriod, string> = { morning: "manhã", afternoon: "tarde", any: "qualquer período" };
export const REASON_CODE_LABEL: Record<RescheduleReasonCode, string> = {
  work: "trabalho", health: "saúde", transport: "transporte", other: "outro motivo"
};
export const BOOKING_KIND_LABEL: Record<BookingKind, string> = { slot: "vaga", fit_in: "encaixe", legacy: "marcação livre" };

// Todos são `detail` do 422 invalid_blocks (contratos §3 e §9).
export type BlockProblem =
  | "overlap" | "missing_type" | "unknown_type" | "inactive_type" | "bad_time" | "crosses_midnight" | "bad_slot_minutes" | "empty";
export const BLOCK_DETAIL_MESSAGE: Record<BlockProblem, string> = {
  overlap: "faixas sobrepostas — ajuste os horários",
  missing_type: "faixa agendável precisa de um tipo de atendimento",
  unknown_type: "faixa com tipo de atendimento que não existe na cidade",
  inactive_type: "faixa com tipo de atendimento desativado — escolha um tipo ativo",
  bad_time: "horário inválido — use HH:MM",
  crosses_midnight: "a faixa não cruza a meia-noite — use fim depois do início (sem 24:00)",
  bad_slot_minutes: `duração da vaga entre ${DURATION_MIN} e ${DURATION_MAX} minutos`,
  empty: "inclua pelo menos uma faixa"
};

const HHMM = /^([01]\d|2[0-3]):[0-5]\d$/;
const toMinutes = (t: string) => Number(t.slice(0, 2)) * 60 + Number(t.slice(3, 5));
const validSpan = (b: ScheduleBlock) => HHMM.test(b.starts) && HHMM.test(b.ends) && toMinutes(b.ends) > toMinutes(b.starts);

// Desativar um tipo nunca quebra o que já existe (ADR 0029): o modelo salvo
// continua valendo. Mas salvar de novo um modelo com tipo inativo é recusado
// (inactive_type, §9); a tela avisa antes e mostra o tipo marcado "(inativo)".
export function blockProblem(block: ScheduleBlock, types: AppointmentType[]): BlockProblem | null {
  if (block.ends === "24:00" && HHMM.test(block.starts)) return "crosses_midnight";
  if (!HHMM.test(block.starts) || !HHMM.test(block.ends)) return "bad_time";
  if (!validSpan(block)) return "crosses_midnight";
  if (block.kind === "bookable") {
    if (!block.appointment_type_key) return "missing_type";
    const type = types.find((t) => t.key === block.appointment_type_key);
    if (!type) return "unknown_type";
    if (!type.active) return "inactive_type";
  }
  if (block.slot_minutes !== undefined &&
      (!Number.isInteger(block.slot_minutes) || block.slot_minutes < DURATION_MIN || block.slot_minutes > DURATION_MAX)) {
    return "bad_slot_minutes";
  }
  return null;
}

export function overlappingBlocks(blocks: ScheduleBlock[]): Set<number> {
  const out = new Set<number>();
  blocks.forEach((a, i) => {
    blocks.forEach((b, j) => {
      if (j <= i || !validSpan(a) || !validSpan(b)) return;
      if (toMinutes(a.starts) < toMinutes(b.ends) && toMinutes(b.starts) < toMinutes(a.ends)) { out.add(i); out.add(j); }
    });
  });
  return out;
}

export function parseFitInLimit(text: string): number | null {
  const t = text.trim();
  if (!/^\d+$/.test(t)) return null;
  const n = Number(t);
  return n <= FIT_IN_LIMIT_MAX ? n : null;
}

export interface TemplateDraft { name: string; fitInLimit: string; blocks: ScheduleBlock[] }

export function templateDraftFrom(t: ScheduleTemplate | null): TemplateDraft {
  if (!t) return { name: "", fitInLimit: "2", blocks: [ { starts: "08:00", ends: "12:00", kind: "bookable" } ] };
  return { name: t.name, fitInLimit: String(t.fit_in_limit), blocks: t.blocks.map((b) => ({ ...b })) };
}

export function templateProblem(draft: TemplateDraft, types: AppointmentType[]): string | null {
  if (draft.name.trim() === "") return "dê um nome ao modelo";
  if (parseFitInLimit(draft.fitInLimit) === null) return `limite de encaixes entre 0 e ${FIT_IN_LIMIT_MAX}`;
  if (draft.blocks.length === 0) return BLOCK_DETAIL_MESSAGE.empty;
  for (const b of draft.blocks) {
    const p = blockProblem(b, types);
    if (p) return BLOCK_DETAIL_MESSAGE[p];
  }
  if (overlappingBlocks(draft.blocks).size > 0) return BLOCK_DETAIL_MESSAGE.overlap;
  return null;
}

export interface TypeDraft { key: string; name: string; duration: string; cbo: string }

export function typeDraftFrom(t: AppointmentType | null): TypeDraft {
  if (!t) return { key: "", name: "", duration: "", cbo: "" };
  return { key: t.key, name: t.name, duration: String(t.duration_minutes), cbo: t.cbo_prefixes.join(", ") };
}

export function parseCboPrefixes(text: string): string[] | null {
  const parts = text.split(/[\s,;]+/).filter(Boolean);
  if (parts.length === 0 || parts.length > CBO_PREFIXES_MAX || parts.some((p) => !/^\d{1,6}$/.test(p))) return null;
  return parts;
}

export function typeDraftProblem(draft: TypeDraft, mode: "create" | "edit"): string | null {
  if (mode === "create" && !TYPE_KEY_PATTERN.test(draft.key)) return "chave: minúsculas, números e _, começando por letra (2 a 41)";
  if (draft.name.trim() === "") return "dê um nome ao tipo";
  const d = Number(draft.duration);
  if (!/^\d+$/.test(draft.duration.trim()) || d < DURATION_MIN || d > DURATION_MAX) {
    return `duração entre ${DURATION_MIN} e ${DURATION_MAX} minutos`;
  }
  if (parseCboPrefixes(draft.cbo) === null) return "informe de 1 a 20 grupos de CBO (só números, separados por vírgula)";
  return null;
}

// `types` nulo: quem chama não lê os tipos (a recepção e o profissional não
// têm GET /professionals/appointment_types) — mostra a chave.
export function typeLabel(key: string | null | undefined, types: AppointmentType[] | null): string {
  if (!key) return "—";
  if (!types) return key;
  const t = types.find((x) => x.key === key);
  if (!t) return `${key} (não existe na cidade)`;
  return t.active ? t.name : `${t.name} (inativo)`;
}

export function blockLine(block: ScheduleBlock, types: AppointmentType[] | null): string {
  const parts = [ `${block.starts}–${block.ends}`, BLOCK_KIND_LABEL[block.kind] ];
  if (block.kind === "bookable") {
    parts.push(!types && block.appointment_type_name ? block.appointment_type_name : typeLabel(block.appointment_type_key, types));
  }
  if (block.slot_minutes !== undefined) parts.push(`vagas de ${block.slot_minutes} min`);
  return parts.join(" · ");
}

// O mesmo aviso que a marcação de hoje mostra (spec 2026-09-25 §6).
export function confirmationWarning(start: Date, now: Date): string {
  const hoursUntil = (start.getTime() - now.getTime()) / 3_600_000;
  if (hoursUntil < 48) return "O horário nasce confirmado";
  const deadline = new Date(start.getTime() - 24 * 3_600_000);
  const day = cityDateFormat({ day: "2-digit", month: "2-digit" }).format(deadline);
  return `O cidadão precisa confirmar até ${day} ${fmtHourMinute(deadline.toISOString())}`;
}

// Aritmética de dia em UTC ao meio-dia: nenhum fuso desloca a data.
const noon = (iso: string) => new Date(`${iso}T12:00:00Z`);

export function addDaysIso(iso: string, n: number): string {
  const d = noon(iso);
  d.setUTCDate(d.getUTCDate() + n);
  return d.toISOString().slice(0, 10);
}

export function weekStart(iso: string): string {
  return addDaysIso(iso, -((noon(iso).getUTCDay() + 6) % 7));
}

export function daysBetween(from: string, to: string): string[] {
  const out: string[] = [];
  for (let d = from; d <= to; d = addDaysIso(d, 1)) out.push(d);
  return out;
}

const WEEKDAYS = [ "dom", "seg", "ter", "qua", "qui", "sex", "sáb" ];
export function dayLabel(iso: string): string {
  return `${WEEKDAYS[noon(iso).getUTCDay()]} ${iso.slice(8, 10)}/${iso.slice(5, 7)}`;
}

export function fmtDueOn(iso: string): string {
  return `até ${iso.slice(8, 10)}/${iso.slice(5, 7)}`;
}

export function slotsByDay(slots: AvailabilitySlot[]): Map<string, AvailabilitySlot[]> {
  const map = new Map<string, AvailabilitySlot[]>();
  for (const s of slots) {
    const day = cityIsoDate(new Date(s.starts_at));
    map.set(day, [ ...(map.get(day) ?? []), s ]);
  }
  return map;
}

export interface QueueMarksInput {
  overdue: boolean; reschedule_requested: boolean; needs_reschedule: boolean; reopened_reason: "expired" | "no_show" | null;
}
type Mark = { label: string; tone: "down" | "warn" | "info" | "neutral" };

export function requestMarks(row: QueueMarksInput): Mark[] {
  const out: Mark[] = [];
  if (row.overdue) out.push({ label: "atrasado", tone: "down" });
  if (row.reschedule_requested) out.push({ label: "pediu outro horário", tone: "warn" });
  if (row.needs_reschedule) out.push({ label: "precisa remarcar", tone: "warn" });
  if (row.reopened_reason === "expired") out.push({ label: "sem confirmação", tone: "neutral" });
  if (row.reopened_reason === "no_show") out.push({ label: "faltou", tone: "neutral" });
  return out;
}

export function appointmentFlags(a: AppointmentView): string[] {
  const out: string[] = [];
  if (a.fit_in) out.push("encaixe");
  if (a.outside_template) out.push("fora do modelo");
  if (a.shift_cancelled) out.push("turno cancelado");
  return out;
}
```

Em `src/lib/attendance.ts`, troque a última entrada de `MESSAGES`:

```ts
  invalid_gender_identity: "identidade de gênero inválida — escolha da lista"
};
```

por:

```ts
  invalid_gender_identity: "identidade de gênero inválida — escolha da lista",
  // Módulo 17 (contratos §4.3 e §4.1). `slot_taken` fica fora: cada painel
  // trata o 409 antes de chamar attendanceError.
  slot_unavailable: "essa vaga não está mais disponível — as vagas foram recarregadas",
  citizen_busy: "o cidadão já tem outro horário nesse período",
  fit_in_limit: "o turno já chegou ao limite de encaixes",
  use_slots: "a unidade tem turno neste dia — marque numa vaga ou faça um encaixe",
  invalid_reason: "a justificativa do encaixe precisa de pelo menos 10 caracteres",
  type_not_served: "este profissional não atende este tipo de atendimento",
  outside_shift: "o encaixe precisa começar e terminar dentro do turno",
  already_assigned: "este pedido já foi atribuído a uma unidade"
};
```

Em `src/lib/professionals.ts`:

```ts
// em MESSAGES, depois de `forbidden`:
  invalid_key: "chave inválida: minúsculas, números e _, começando por letra",
  key_taken: "já existe um tipo com esta chave",
  invalid_duration: "duração entre 5 e 240 minutos",
  invalid_cbo_prefixes: "grupos de CBO inválidos — de 1 a 20, só números (até 6 dígitos cada)",
  platform_type_locked: "tipo da plataforma: a chave e os grupos de CBO não mudam",
  invalid_fit_in_limit: "limite de encaixes entre 0 e 20",
  invalid_name: "dê um nome — o campo não pode ficar vazio"
```

```ts
type ProfessionalErrorBody = {
  error?: string; fields?: string[]; detail?: string; conflict?: { unit_name?: string; starts_at: string; ends_at: string };
};
```

E, em `translateProfessionalErrorBody`, antes do `return` final:

```ts
  if (body.error === "invalid_blocks") {
    const detail = body.detail as keyof typeof BLOCK_DETAIL_MESSAGE | undefined;
    return (detail && BLOCK_DETAIL_MESSAGE[detail]) || "faixas inválidas — confira o modelo";
  }
```

com `import { BLOCK_DETAIL_MESSAGE } from "./scheduling";` no topo.

- [ ] **Step 4: Run test to verify it passes**

Acrescente ao fim de `src/lib/professionals.test.ts` (e, no topo, `ApiError` e `professionalError` nos imports, se ainda não estiverem):

```ts
describe("recusas do módulo 17", () => {
  it("invalid_blocks traduz o detail; sem detail, frase genérica", () => {
    expect(professionalError(new ApiError(422, { error: "invalid_blocks", detail: "overlap" }, "x")))
      .toBe("faixas sobrepostas — ajuste os horários");
    expect(professionalError(new ApiError(422, { error: "invalid_blocks" }, "x"))).toBe("faixas inválidas — confira o modelo");
    expect(professionalError(new ApiError(422, { error: "invalid_blocks", detail: "inactive_type" }, "x")))
      .toBe("faixa com tipo de atendimento desativado — escolha um tipo ativo");
    expect(professionalError(new ApiError(422, { error: "platform_type_locked" }, "x")))
      .toBe("tipo da plataforma: a chave e os grupos de CBO não mudam");
  });
});
```

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/lib/scheduling.test.ts src/lib/professionals.test.ts src/lib/attendance.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod17
/opt/homebrew/bin/git add src/lib/scheduling.ts src/lib/scheduling.test.ts src/test/schedulingFixtures.ts \
  src/lib/attendance.ts src/lib/professionals.ts src/lib/professionals.test.ts
/opt/homebrew/bin/git commit -m "feat: add scheduling rules and refusal messages for module 17

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 3: Profissionais → aba "Tipos de atendimento"

**Files:**
- Create: `src/modules/professionals/AppointmentTypes.tsx`
- Modify: `src/modules/Professionals.tsx` (abas; o corpo atual vira `People`)
- Test: `src/modules/professionals/AppointmentTypes.test.tsx`

**Interfaces:**
- Consumes: `listAppointmentTypes`, `createAppointmentType`, `updateAppointmentType`, `AppointmentType` (Task 1); `typeDraftFrom`, `typeDraftProblem`, `parseCboPrefixes`, `TypeDraft` (Task 2); `professionalError`.
- Produces: `APPOINTMENT_TYPES_KEY = [ "appointmentTypes" ] as const` e `AppointmentTypes()` (exportados de `AppointmentTypes.tsx`); `type ProfessionalsTab = "people" | "types" | "templates"` em `Professionals.tsx` (a aba `templates` entra na Task 4).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/professionals/AppointmentTypes.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, listAppointmentTypes: vi.fn(), createAppointmentType: vi.fn(), updateAppointmentType: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { AppointmentTypes } from "./AppointmentTypes";
import { TYPES } from "../../test/schedulingFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function renderIt() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<AppointmentTypes />, { wrapper });
}

describe("AppointmentTypes", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    mocked(api.listAppointmentTypes).mockResolvedValue(TYPES);
  });

  it("lista nome, chave, duração, CBOs, origem e situação", async () => {
    renderIt();
    expect(await screen.findByText("Consulta médica")).not.toBeNull();
    expect(screen.getByText("consulta_medica")).not.toBeNull();
    expect(screen.getByText("20 min")).not.toBeNull();
    expect(screen.getByText("2251, 2252, 2253")).not.toBeNull();
    expect(screen.getAllByText("plataforma")).toHaveLength(3);
    expect(screen.getByText("cidade")).not.toBeNull();
    expect(screen.getByText("inativo")).not.toBeNull();
  });

  it("novo tipo: chave inválida trava o botão; válido manda o payload e relê a lista", async () => {
    mocked(api.createAppointmentType).mockResolvedValue({ ...TYPES[3], key: "pre_natal", active: true });
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Novo tipo" }));
    fireEvent.change(screen.getByLabelText("Chave"), { target: { value: "Pre-natal" } });
    fireEvent.change(screen.getByLabelText("Nome"), { target: { value: "Pré-natal" } });
    fireEvent.change(screen.getByLabelText("Duração (min)"), { target: { value: "30" } });
    fireEvent.change(screen.getByLabelText("Grupos de CBO"), { target: { value: "2235, 2251" } });
    expect((screen.getByRole("button", { name: "Salvar tipo" }) as HTMLButtonElement).disabled).toBe(true);
    expect(screen.getByText("chave: minúsculas, números e _, começando por letra (2 a 41)")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Chave"), { target: { value: "pre_natal" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar tipo" }));
    await waitFor(() => expect(api.createAppointmentType).toHaveBeenCalledWith({
      key: "pre_natal", name: "Pré-natal", duration_minutes: 30, cbo_prefixes: [ "2235", "2251" ]
    }));
    await waitFor(() => expect(api.listAppointmentTypes).toHaveBeenCalledTimes(2));
  });

  it("key_taken aparece traduzido e o formulário continua aberto", async () => {
    mocked(api.createAppointmentType).mockRejectedValue(new ApiError(422, { error: "key_taken" }, "x"));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Novo tipo" }));
    fireEvent.change(screen.getByLabelText("Chave"), { target: { value: "retorno" } });
    fireEvent.change(screen.getByLabelText("Nome"), { target: { value: "Retorno" } });
    fireEvent.change(screen.getByLabelText("Duração (min)"), { target: { value: "15" } });
    fireEvent.change(screen.getByLabelText("Grupos de CBO"), { target: { value: "2251" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar tipo" }));
    expect(await screen.findByText("já existe um tipo com esta chave")).not.toBeNull();
    expect(screen.getByRole("button", { name: "Salvar tipo" })).not.toBeNull();
  });

  it("tipo da plataforma: CBO travado, e salvar manda só nome e duração", async () => {
    mocked(api.updateAppointmentType).mockResolvedValue({ ...TYPES[0], duration_minutes: 25 });
    renderIt();
    await screen.findByText("Consulta médica");
    const row = screen.getByText("consulta_medica").closest("[role=row]") as HTMLElement;
    fireEvent.click(within(row).getByRole("button", { name: "Editar" }));
    expect((screen.getByLabelText("Grupos de CBO") as HTMLInputElement).disabled).toBe(true);
    expect(screen.queryByLabelText("Chave")).toBeNull();
    fireEvent.change(screen.getByLabelText("Duração (min)"), { target: { value: "25" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar tipo" }));
    await waitFor(() => expect(api.updateAppointmentType).toHaveBeenCalledWith("consulta_medica",
      { name: "Consulta médica", duration_minutes: 25 }));
  });

  it("tipo da cidade manda os CBOs; desativar e reativar mandam só active", async () => {
    mocked(api.updateAppointmentType).mockResolvedValue(TYPES[3]);
    renderIt();
    await screen.findByText("Puericultura");
    const row = screen.getByText("puericultura").closest("[role=row]") as HTMLElement;
    fireEvent.click(within(row).getByRole("button", { name: "Reativar" }));
    await waitFor(() => expect(api.updateAppointmentType).toHaveBeenCalledWith("puericultura", { active: true }));
    const medica = screen.getByText("consulta_medica").closest("[role=row]") as HTMLElement;
    fireEvent.click(within(medica).getByRole("button", { name: "Desativar" }));
    await waitFor(() => expect(api.updateAppointmentType).toHaveBeenCalledWith("consulta_medica", { active: false }));
    fireEvent.click(within(row).getByRole("button", { name: "Editar" }));
    fireEvent.click(screen.getByRole("button", { name: "Salvar tipo" }));
    await waitFor(() => expect(api.updateAppointmentType).toHaveBeenLastCalledWith("puericultura",
      { name: "Puericultura", duration_minutes: 30, cbo_prefixes: [ "2235" ] }));
  });
});
```

Confira antes que `DataTable` marca cada linha com `role="row"` (`src/components/DataTable.tsx`); é o que `closest("[role=row]")` usa.

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/modules/professionals/AppointmentTypes.test.tsx`
Expected: FAIL — `Failed to resolve import "./AppointmentTypes"`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/professionals/AppointmentTypes.tsx
// Tipos de atendimento (módulo 17; spec §3.1, contratos §3): base da
// plataforma copiada para a cidade + tipos próprios. Só municipal_admin.
// Tipo da plataforma não muda chave nem grupos de CBO; desativar nunca
// quebra pedido ou horário existente (ADR 0029).
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { createAppointmentType, listAppointmentTypes, updateAppointmentType, type AppointmentType } from "../../lib/api";
import { professionalError } from "../../lib/professionals";
import { parseCboPrefixes, typeDraftFrom, typeDraftProblem, type TypeDraft } from "../../lib/scheduling";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { EmptyState } from "../../components/EmptyState";
import { Tag } from "../../components/Tag";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";

export const APPOINTMENT_TYPES_KEY = [ "appointmentTypes" ] as const;

export function AppointmentTypes() {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: APPOINTMENT_TYPES_KEY, queryFn: listAppointmentTypes });
  const [ editing, setEditing ] = useState<AppointmentType | "new" | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const refresh = () => void queryClient.invalidateQueries({ queryKey: APPOINTMENT_TYPES_KEY });

  async function toggle(t: AppointmentType) {
    setError(null);
    try {
      await updateAppointmentType(t.key, { active: !t.active });
      refresh();
    } catch (err) {
      setError(professionalError(err));
    }
  }

  return (
    <Panel title="Tipos de atendimento" sub="base da plataforma · tipos da cidade"
      right={<button type="button" style={buttonStyle} onClick={() => setEditing("new")}>Novo tipo</button>}>
      <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
        {error && <p role="alert" style={alert}>{error}</p>}
        {query.isError ? <p role="alert" style={alert}>{professionalError(query.error)}</p>
          : query.isPending ? <p className="mono" style={hint}>carregando…</p>
          : (query.data ?? []).length === 0 ? <EmptyState title="nenhum tipo de atendimento" />
          : (
            <DataTable<AppointmentType>
              cols={[
                { label: "Nome", w: "2fr", render: (t) => t.name },
                { label: "Chave", w: "2fr", render: (t) => <span className="mono">{t.key}</span> },
                { label: "Duração", w: "1fr", render: (t) => `${t.duration_minutes} min` },
                { label: "Grupos de CBO", w: "2fr", render: (t) => t.cbo_prefixes.join(", ") },
                { label: "Origem", w: "1fr", render: (t) => <Tag>{t.origin === "platform" ? "plataforma" : "cidade"}</Tag> },
                { label: "Situação", w: "1fr", render: (t) => t.active ? "ativo" : <Tag tone="warn">inativo</Tag> },
                { label: "", w: "auto", align: "right", render: (t) => (
                  <span style={{ display: "flex", gap: 6 }}>
                    <button type="button" style={secondaryButtonStyle} onClick={() => setEditing(t)}>Editar</button>
                    <button type="button" style={secondaryButtonStyle} onClick={() => void toggle(t)}>
                      {t.active ? "Desativar" : "Reativar"}
                    </button>
                  </span>
                ) }
              ]}
              rows={query.data ?? []}
              rowKey={(t) => t.key}
            />
          )}
        {editing && (
          <TypeForm
            key={editing === "new" ? "new" : editing.key}
            type={editing === "new" ? null : editing}
            onDone={() => { setEditing(null); refresh(); }}
            onCancel={() => setEditing(null)}
          />
        )}
      </div>
    </Panel>
  );
}

function TypeForm({ type, onDone, onCancel }: { type: AppointmentType | null; onDone(): void; onCancel(): void }) {
  const mode = type ? "edit" : "create";
  const locked = type?.origin === "platform";
  const [ draft, setDraft ] = useState<TypeDraft>(() => typeDraftFrom(type));
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const problem = typeDraftProblem(draft, mode);
  const set = (patch: Partial<TypeDraft>) => setDraft((d) => ({ ...d, ...patch }));

  async function save() {
    if (busy || problem) return;
    const name = draft.name.trim();
    const duration = Number(draft.duration);
    const cbo = parseCboPrefixes(draft.cbo) ?? [];
    setBusy(true); setError(null);
    try {
      if (!type) {
        await createAppointmentType({ key: draft.key, name, duration_minutes: duration, cbo_prefixes: cbo });
      } else {
        await updateAppointmentType(type.key, locked
          ? { name, duration_minutes: duration }
          : { name, duration_minutes: duration, cbo_prefixes: cbo });
      }
      onDone();
    } catch (err) {
      setError(professionalError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <section style={box}>
      <strong>{type ? `Editar ${type.name}` : "Novo tipo de atendimento"}</strong>
      {error && <p role="alert" style={alert}>{error}</p>}
      {type ? <span className="mono" style={hint}>{type.key}</span> : (
        <label style={label}>Chave
          <input value={draft.key} onChange={(e) => set({ key: e.target.value })} style={inputStyle} placeholder="pre_natal" />
        </label>
      )}
      <label style={label}>Nome<input value={draft.name} onChange={(e) => set({ name: e.target.value })} style={inputStyle} /></label>
      <label style={label}>Duração (min)
        <input inputMode="numeric" value={draft.duration} onChange={(e) => set({ duration: e.target.value })} style={inputStyle} />
      </label>
      <label style={label}>Grupos de CBO
        <input value={draft.cbo} disabled={locked} onChange={(e) => set({ cbo: e.target.value })} style={inputStyle}
          placeholder="2251, 2252" />
      </label>
      {locked && <small style={hint}>tipo da plataforma: chave e grupos de CBO fixos</small>}
      {problem && <small style={hint}>{problem}</small>}
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={!!problem || busy} onClick={() => void save()}
          style={problem || busy ? disabledButtonStyle : buttonStyle}>Salvar tipo</button>
        <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Cancelar</button>
      </div>
    </section>
  );
}

const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)", maxWidth: 360 };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const box: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 12, border: "1px solid var(--rule)", borderRadius: 8 };
```

Em `src/modules/Professionals.tsx`: renomeie a função exportada `Professionals` para `function People()` (sem `export`, corpo igual) e acrescente, acima dela:

```tsx
import { SegmentedControl } from "../shell/SegmentedControl";
import { AppointmentTypes } from "./professionals/AppointmentTypes";

// Módulo 17: tipos de atendimento e modelos de agenda moram aqui, com o
// mesmo papel (municipal_admin) da lista de profissionais.
type ProfessionalsTab = "people" | "types";
const TABS: { key: ProfessionalsTab; label: string }[] = [
  { key: "people", label: "Profissionais" },
  { key: "types", label: "Tipos de atendimento" }
];

export function Professionals() {
  const [ tab, setTab ] = useState<ProfessionalsTab>("people");
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <div><SegmentedControl options={TABS} value={tab} onChange={setTab} /></div>
      {tab === "people" && <People />}
      {tab === "types" && <AppointmentTypes />}
    </div>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/modules/professionals/AppointmentTypes.test.tsx src/modules/Professionals.test.tsx && npx tsc --noEmit`
Expected: PASS. O `Professionals.test.tsx` existente continua verde (a aba padrão é a lista de sempre).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod17
/opt/homebrew/bin/git add src/modules/professionals/AppointmentTypes.tsx src/modules/professionals/AppointmentTypes.test.tsx \
  src/modules/Professionals.tsx
/opt/homebrew/bin/git commit -m "feat: manage appointment types in the professionals module

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Profissionais → aba "Modelos de agenda" (editor de faixas e pré-visualização)

**Files:**
- Create: `src/modules/professionals/ScheduleTemplates.tsx`, `src/modules/professionals/TemplateEditor.tsx`
- Modify: `src/modules/Professionals.tsx` (aba `templates`)
- Test: `src/modules/professionals/TemplateEditor.test.tsx`, `src/modules/professionals/ScheduleTemplates.test.tsx`

**Interfaces:**
- Consumes: `listScheduleTemplates`, `createScheduleTemplate`, `updateScheduleTemplate`, `previewScheduleTemplate`, `listAppointmentTypes`, `listCbo`, `ScheduleTemplate`, `ScheduleBlock`, `AppointmentType`, `TemplatePreview` (Task 1); `templateDraftFrom`, `templateProblem`, `blockProblem`, `overlappingBlocks`, `parseFitInLimit`, `blockLine`, `typeLabel`, `BLOCK_KIND_LABEL`, `BLOCK_DETAIL_MESSAGE`, `TemplateDraft` (Task 2); `APPOINTMENT_TYPES_KEY` (Task 3); `shiftWindow`, `professionalError` (`src/lib/professionals.ts`); `fmtHourMinute`, `cityIsoDate`.
- Produces: `SCHEDULE_TEMPLATES_KEY = [ "scheduleTemplates" ] as const` e `ScheduleTemplates()` (de `ScheduleTemplates.tsx`); `TemplateEditor({ template, types, onSaved, onCancel })` (de `TemplateEditor.tsx`).

- [ ] **Step 1: Write the failing tests**

```tsx
// src/modules/professionals/TemplateEditor.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, createScheduleTemplate: vi.fn(), updateScheduleTemplate: vi.fn(), previewScheduleTemplate: vi.fn(), listCbo: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { TemplateEditor } from "./TemplateEditor";
import { MORNING, NOW, TYPES } from "../../test/schedulingFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function renderIt(template = MORNING as typeof MORNING | null) {
  const onSaved = vi.fn();
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<TemplateEditor template={template} types={TYPES} onSaved={onSaved} onCancel={vi.fn()} />, { wrapper });
  return { onSaved };
}
const block = (n: number) => screen.getByRole("group", { name: `faixa ${n}` });
const save = () => screen.getByRole("button", { name: "Salvar modelo" }) as HTMLButtonElement;

describe("TemplateEditor", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW));
    mocked(api.listCbo).mockResolvedValue([ { code: "225125", title: "Médico clínico", council: "CRM" } ]);
  });

  it("abre o modelo com as três faixas e salva sem mudar nada", async () => {
    mocked(api.updateScheduleTemplate).mockResolvedValue(MORNING);
    const { onSaved } = renderIt();
    expect((within(block(2)).getByLabelText("Tipo de atendimento") as HTMLSelectElement).value).toBe("consulta_medica");
    expect(save().disabled).toBe(false);
    fireEvent.click(save());
    await waitFor(() => expect(api.updateScheduleTemplate).toHaveBeenCalledWith("t1",
      { name: "Manhã", fit_in_limit: 2, blocks: MORNING.blocks }));
    await waitFor(() => expect(onSaved).toHaveBeenCalled());
  });

  it("faixas sobrepostas: as duas mostram o motivo e o botão trava; corrigir destrava", () => {
    renderIt();
    fireEvent.change(within(block(1)).getByLabelText("Fim"), { target: { value: "09:30" } });
    expect(within(block(1)).getByText("faixas sobrepostas — ajuste os horários")).not.toBeNull();
    expect(within(block(2)).getByText("faixas sobrepostas — ajuste os horários")).not.toBeNull();
    expect(save().disabled).toBe(true);
    fireEvent.change(within(block(1)).getByLabelText("Fim"), { target: { value: "09:00" } });
    expect(save().disabled).toBe(false);
  });

  it("faixa agendável sem tipo trava; trocar para demanda do dia tira o tipo e a vaga", () => {
    renderIt();
    fireEvent.change(within(block(2)).getByLabelText("Tipo de atendimento"), { target: { value: "" } });
    expect(within(block(2)).getByText("faixa agendável precisa de um tipo de atendimento")).not.toBeNull();
    expect(save().disabled).toBe(true);
    fireEvent.change(within(block(2)).getByLabelText("Tipo de faixa"), { target: { value: "walk_in" } });
    expect(within(block(2)).queryByLabelText("Tipo de atendimento")).toBeNull();
    expect(save().disabled).toBe(false);
  });

  it("faixa com tipo inativo continua no select marcada e pede troca", () => {
    renderIt({ ...MORNING, blocks: [ { starts: "09:00", ends: "11:00", kind: "bookable", appointment_type_key: "puericultura" } ] });
    const select = within(block(1)).getByLabelText("Tipo de atendimento") as HTMLSelectElement;
    expect(select.value).toBe("puericultura");
    expect(within(select).getByRole("option", { name: "Puericultura (inativo)" })).not.toBeNull();
    expect(within(block(1)).getByText("faixa com tipo de atendimento desativado — escolha um tipo ativo")).not.toBeNull();
    expect(save().disabled).toBe(true);
    fireEvent.change(select, { target: { value: "consulta_enfermagem" } });
    expect(save().disabled).toBe(false);
  });

  it("faixa que cruza a meia-noite trava", () => {
    renderIt();
    fireEvent.change(within(block(3)).getByLabelText("Fim"), { target: { value: "10:00" } });
    expect(within(block(3)).getByText("a faixa não cruza a meia-noite — use fim depois do início (sem 24:00)")).not.toBeNull();
    expect(save().disabled).toBe(true);
  });

  it("nome da faixa que veio da API não volta no salvar", async () => {
    mocked(api.updateScheduleTemplate).mockResolvedValue(MORNING);
    renderIt({ ...MORNING, blocks: [ { ...MORNING.blocks[1], appointment_type_name: "Consulta médica" } ] });
    fireEvent.click(save());
    await waitFor(() => expect(api.updateScheduleTemplate).toHaveBeenCalledWith("t1",
      { name: "Manhã", fit_in_limit: 2, blocks: [ MORNING.blocks[1] ] }));
    const sent = mocked(api.updateScheduleTemplate).mock.calls[0][1] as { blocks: object[] };
    expect("appointment_type_name" in sent.blocks[0]).toBe(false);
  });

  it("+ faixa começa onde a última termina; remover tira a faixa", () => {
    renderIt();
    fireEvent.click(screen.getByRole("button", { name: "+ faixa" }));
    expect((within(block(4)).getByLabelText("Início") as HTMLInputElement).value).toBe("12:00");
    expect((within(block(4)).getByLabelText("Fim") as HTMLInputElement).value).toBe("13:00");
    fireEvent.click(within(block(4)).getByRole("button", { name: "remover faixa" }));
    expect(screen.queryByRole("group", { name: "faixa 4" })).toBeNull();
  });

  it("modelo novo: cria com nome, limite e faixas; 422 invalid_blocks mostra o detalhe", async () => {
    mocked(api.createScheduleTemplate).mockRejectedValue(new ApiError(422, { error: "invalid_blocks", detail: "unknown_type" }, "x"));
    renderIt(null);
    expect(save().disabled).toBe(true); // sem nome e com faixa agendável sem tipo
    fireEvent.change(screen.getByLabelText("Nome do modelo"), { target: { value: "Tarde" } });
    fireEvent.change(within(block(1)).getByLabelText("Tipo de atendimento"), { target: { value: "consulta_enfermagem" } });
    fireEvent.change(screen.getByLabelText("Encaixes por turno"), { target: { value: "3" } });
    fireEvent.click(save());
    await waitFor(() => expect(api.createScheduleTemplate).toHaveBeenCalledWith({ name: "Tarde", fit_in_limit: 3,
      blocks: [ { starts: "08:00", ends: "12:00", kind: "bookable", appointment_type_key: "consulta_enfermagem" } ] }));
    expect(await screen.findByText("faixa com tipo de atendimento que não existe na cidade")).not.toBeNull();
  });

  it("limite de encaixes fora de 0–20 trava", () => {
    renderIt();
    fireEvent.change(screen.getByLabelText("Encaixes por turno"), { target: { value: "21" } });
    expect(screen.getByText("limite de encaixes entre 0 e 20")).not.toBeNull();
    expect(save().disabled).toBe(true);
  });

  it("pré-visualização: manda o turno de exemplo no fuso da cidade e lista as vagas; mudar o modelo esconde o resultado", async () => {
    mocked(api.previewScheduleTemplate).mockResolvedValue({
      slots: [
        { starts_at: "2026-10-06T09:00:00-03:00", ends_at: "2026-10-06T09:20:00-03:00", appointment_type_key: "consulta_medica" },
        { starts_at: "2026-10-06T09:20:00-03:00", ends_at: "2026-10-06T09:40:00-03:00", appointment_type_key: "consulta_medica" }
      ],
      blocks: MORNING.blocks
    });
    renderIt();
    const box = screen.getByRole("group", { name: "Pré-visualização" });
    await within(box).findByRole("option", { name: "225125 · Médico clínico" });
    fireEvent.change(within(box).getByLabelText("Data do exemplo"), { target: { value: "2026-10-06" } });
    fireEvent.change(within(box).getByLabelText("Início do turno"), { target: { value: "07:00" } });
    fireEvent.change(within(box).getByLabelText("Fim do turno"), { target: { value: "12:00" } });
    fireEvent.change(within(box).getByLabelText("Ocupação do exemplo (CBO)"), { target: { value: "225125" } });
    fireEvent.click(within(box).getByRole("button", { name: "Pré-visualizar" }));
    await waitFor(() => expect(api.previewScheduleTemplate).toHaveBeenCalledWith({
      blocks: MORNING.blocks, fit_in_limit: 2,
      sample: { starts_at: "2026-10-06T07:00:00-03:00", ends_at: "2026-10-06T12:00:00-03:00", cbo_code: "225125" }
    }));
    expect(await within(box).findByText("09:00–09:20 · Consulta médica")).not.toBeNull();
    expect(within(box).getByText("2 vagas")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Encaixes por turno"), { target: { value: "3" } });
    expect(within(box).queryByText("09:00–09:20 · Consulta médica")).toBeNull();
  });

  it("pré-visualização sem vaga diz por quê olhar", async () => {
    mocked(api.previewScheduleTemplate).mockResolvedValue({ slots: [], blocks: MORNING.blocks });
    renderIt();
    const box = screen.getByRole("group", { name: "Pré-visualização" });
    await within(box).findByRole("option", { name: "225125 · Médico clínico" });
    fireEvent.change(within(box).getByLabelText("Data do exemplo"), { target: { value: "2026-10-06" } });
    fireEvent.change(within(box).getByLabelText("Ocupação do exemplo (CBO)"), { target: { value: "225125" } });
    fireEvent.click(within(box).getByRole("button", { name: "Pré-visualizar" }));
    expect(await within(box).findByText("nenhuma vaga — confira o tipo das faixas e se o CBO do exemplo é atendido por ele"))
      .not.toBeNull();
  });
});
```

```tsx
// src/modules/professionals/ScheduleTemplates.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, listScheduleTemplates: vi.fn(), listAppointmentTypes: vi.fn(), updateScheduleTemplate: vi.fn(), listCbo: vi.fn() };
});

import * as api from "../../lib/api";
import { ScheduleTemplates } from "./ScheduleTemplates";
import { MORNING, TYPES } from "../../test/schedulingFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function renderIt() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<ScheduleTemplates />, { wrapper });
}

describe("ScheduleTemplates", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    mocked(api.listScheduleTemplates).mockResolvedValue([ MORNING ]);
    mocked(api.listAppointmentTypes).mockResolvedValue(TYPES);
    mocked(api.listCbo).mockResolvedValue([]);
  });

  it("lista as faixas em frase, o limite e a situação", async () => {
    renderIt();
    expect(await screen.findByText("09:00–11:00 · agendável · Consulta médica")).not.toBeNull();
    expect(screen.getByText("07:00–09:00 · demanda do dia")).not.toBeNull();
    expect(screen.getByText("11:00–12:00 · bloqueada")).not.toBeNull();
    expect(screen.getByText("2 por turno")).not.toBeNull();
  });

  it("desativar manda só active e relê", async () => {
    mocked(api.updateScheduleTemplate).mockResolvedValue({ ...MORNING, active: false });
    renderIt();
    const row = (await screen.findByText("Manhã")).closest("[role=row]") as HTMLElement;
    fireEvent.click(within(row).getByRole("button", { name: "Desativar" }));
    await waitFor(() => expect(api.updateScheduleTemplate).toHaveBeenCalledWith("t1", { active: false }));
    await waitFor(() => expect(api.listScheduleTemplates).toHaveBeenCalledTimes(2));
  });

  it("Editar abre o editor com o modelo", async () => {
    renderIt();
    const row = (await screen.findByText("Manhã")).closest("[role=row]") as HTMLElement;
    fireEvent.click(within(row).getByRole("button", { name: "Editar" }));
    expect((screen.getByLabelText("Nome do modelo") as HTMLInputElement).value).toBe("Manhã");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/modules/professionals/TemplateEditor.test.tsx src/modules/professionals/ScheduleTemplates.test.tsx`
Expected: FAIL — `Failed to resolve import "./TemplateEditor"` e `"./ScheduleTemplates"`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/professionals/TemplateEditor.tsx
// Editor de faixas do modelo de agenda (módulo 17; spec §3.2, contratos §2–§3).
// A validação local espelha o 422 invalid_blocks (todos os `detail` das
// §3 e §9 do contrato), invalid_name e invalid_fit_in_limit; o api continua
// sendo quem garante. A pré-visualização calcula as vagas de um turno de exemplo sem
// gravar nada.
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import {
  createScheduleTemplate, listCbo, previewScheduleTemplate, updateScheduleTemplate,
  type AppointmentType, type BlockKind, type ScheduleBlock, type ScheduleTemplate, type TemplatePreview
} from "../../lib/api";
import { professionalError, shiftWindow } from "../../lib/professionals";
import {
  BLOCK_DETAIL_MESSAGE, BLOCK_KIND_LABEL, blockLine, blockProblem, overlappingBlocks, parseFitInLimit, templateDraftFrom,
  templateProblem, typeLabel, type TemplateDraft
} from "../../lib/scheduling";
import { cityIsoDate, fmtHourMinute } from "../../lib/format";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";

interface Props { template: ScheduleTemplate | null; types: AppointmentType[]; onSaved(): void; onCancel(): void }

const KINDS: BlockKind[] = [ "walk_in", "bookable", "blocked" ];

// Faixa que deixa de ser agendável perde tipo e duração da vaga: o schema
// do api só aceita os dois em `bookable`.
function normalize(b: ScheduleBlock): ScheduleBlock {
  if (b.kind === "bookable") return b;
  return { starts: b.starts, ends: b.ends, kind: b.kind };
}

function nextBlock(blocks: ScheduleBlock[]): ScheduleBlock {
  const last = blocks[blocks.length - 1];
  const start = last?.ends && /^\d{2}:\d{2}$/.test(last.ends) ? last.ends : "08:00";
  const hour = Math.min(Number(start.slice(0, 2)) + 1, 23);
  const end = hour === Number(start.slice(0, 2)) ? "23:59" : `${String(hour).padStart(2, "0")}:${start.slice(3, 5)}`;
  return { starts: start, ends: end, kind: "bookable" };
}

export function TemplateEditor({ template, types, onSaved, onCancel }: Props) {
  const [ draft, setDraft ] = useState<TemplateDraft>(() => templateDraftFrom(template));
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const overlaps = overlappingBlocks(draft.blocks);
  const problem = templateProblem(draft, types);

  const setBlocks = (blocks: ScheduleBlock[]) => setDraft((d) => ({ ...d, blocks }));
  const setBlock = (i: number, patch: Partial<ScheduleBlock>) =>
    setBlocks(draft.blocks.map((b, j) => (j === i ? normalize({ ...b, ...patch }) : b)));

  async function save() {
    if (busy || problem) return;
    // `appointment_type_name` é só de leitura (§9): nunca volta para o api.
    const blocks = draft.blocks.map(({ appointment_type_name: _name, ...rest }) => rest);
    const fields = { name: draft.name.trim(), fit_in_limit: parseFitInLimit(draft.fitInLimit) ?? 0, blocks };
    setBusy(true); setError(null);
    try {
      if (template) await updateScheduleTemplate(template.id, fields);
      else await createScheduleTemplate(fields);
      onSaved();
    } catch (err) {
      setError(professionalError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <section style={box}>
      <strong>{template ? `Editar ${template.name}` : "Novo modelo de agenda"}</strong>
      {error && <p role="alert" style={alert}>{error}</p>}
      <label style={label}>Nome do modelo
        <input value={draft.name} onChange={(e) => setDraft((d) => ({ ...d, name: e.target.value }))} style={inputStyle} />
      </label>
      <label style={label}>Encaixes por turno
        <input inputMode="numeric" value={draft.fitInLimit} style={inputStyle}
          onChange={(e) => setDraft((d) => ({ ...d, fitInLimit: e.target.value }))} />
      </label>

      {draft.blocks.map((b, i) => {
        const p = blockProblem(b, types) ?? (overlaps.has(i) ? "overlap" : null);
        const activeTypes = types.filter((t) => t.active);
        const current = b.appointment_type_key;
        return (
          <div key={i} role="group" aria-label={`faixa ${i + 1}`} style={card}>
            <div style={{ display: "flex", gap: 8, flexWrap: "wrap" }}>
              <label style={label}>Início
                <input type="time" value={b.starts} onChange={(e) => setBlock(i, { starts: e.target.value })} style={inputStyle} />
              </label>
              <label style={label}>Fim
                <input type="time" value={b.ends} onChange={(e) => setBlock(i, { ends: e.target.value })} style={inputStyle} />
              </label>
              <label style={label}>Tipo de faixa
                <select value={b.kind} onChange={(e) => setBlock(i, { kind: e.target.value as BlockKind })} style={inputStyle}>
                  {KINDS.map((k) => <option key={k} value={k}>{BLOCK_KIND_LABEL[k]}</option>)}
                </select>
              </label>
              {b.kind === "bookable" && (
                <label style={label}>Tipo de atendimento
                  <select value={current ?? ""} style={inputStyle}
                    onChange={(e) => setBlock(i, { appointment_type_key: e.target.value || undefined })}>
                    <option value="">escolha…</option>
                    {activeTypes.map((t) => <option key={t.key} value={t.key}>{t.name}</option>)}
                    {current && !activeTypes.some((t) => t.key === current) && (
                      <option value={current}>{typeLabel(current, types)}</option>
                    )}
                  </select>
                </label>
              )}
              {b.kind === "bookable" && (
                <label style={label}>Vaga (min)
                  <input type="number" min={5} max={240} value={b.slot_minutes ?? ""} style={inputStyle}
                    placeholder={String(types.find((t) => t.key === current)?.duration_minutes ?? "")}
                    onChange={(e) => setBlock(i, { slot_minutes: e.target.value === "" ? undefined : Number(e.target.value) })} />
                </label>
              )}
            </div>
            {p && <small style={hint}>{BLOCK_DETAIL_MESSAGE[p]}</small>}
            <div>
              <button type="button" style={secondaryButtonStyle}
                onClick={() => setBlocks(draft.blocks.filter((_, j) => j !== i))}>remover faixa</button>
            </div>
          </div>
        );
      })}
      <div>
        <button type="button" style={secondaryButtonStyle} onClick={() => setBlocks([ ...draft.blocks, nextBlock(draft.blocks) ])}>
          + faixa
        </button>
      </div>
      <small style={hint}>Vaga sem duração usa a duração do tipo. A sobra no fim da faixa não vira vaga.</small>
      {problem && <small style={hint}>{problem}</small>}

      <Preview draft={draft} types={types} blocked={problem !== null} />

      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={!!problem || busy} onClick={() => void save()}
          style={problem || busy ? disabledButtonStyle : buttonStyle}>Salvar modelo</button>
        <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Cancelar</button>
      </div>
    </section>
  );
}

function Preview({ draft, types, blocked }: { draft: TemplateDraft; types: AppointmentType[]; blocked: boolean }) {
  const cbo = useQuery({ queryKey: [ "cbo" ], queryFn: listCbo });
  const [ date, setDate ] = useState(() => cityIsoDate());
  const [ start, setStart ] = useState("07:00");
  const [ end, setEnd ] = useState("12:00");
  const [ cboCode, setCboCode ] = useState("");
  const [ result, setResult ] = useState<{ key: string; preview: TemplatePreview } | null>(null);
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const span = shiftWindow(date, start, end);
  // O resultado vale só para o que foi pré-visualizado: mudar modelo ou
  // exemplo esconde a lista velha.
  const inputKey = JSON.stringify([ draft, date, start, end, cboCode ]);
  const ready = !blocked && !!span && cboCode !== "" && !busy;

  async function run() {
    if (!ready || !span) return;
    setBusy(true); setError(null);
    try {
      const preview = await previewScheduleTemplate({
        blocks: draft.blocks, fit_in_limit: parseFitInLimit(draft.fitInLimit) ?? 0,
        sample: { starts_at: span.startsAt, ends_at: span.endsAt, cbo_code: cboCode }
      });
      setResult({ key: inputKey, preview });
    } catch (err) {
      setError(professionalError(err));
    } finally {
      setBusy(false);
    }
  }

  const shown = result && result.key === inputKey ? result.preview : null;

  return (
    <div role="group" aria-label="Pré-visualização" style={card}>
      <strong style={{ fontSize: 13 }}>Pré-visualização</strong>
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={{ display: "flex", gap: 8, flexWrap: "wrap" }}>
        <label style={label}>Data do exemplo<input type="date" value={date} onChange={(e) => setDate(e.target.value)} style={inputStyle} /></label>
        <label style={label}>Início do turno<input type="time" value={start} onChange={(e) => setStart(e.target.value)} style={inputStyle} /></label>
        <label style={label}>Fim do turno<input type="time" value={end} onChange={(e) => setEnd(e.target.value)} style={inputStyle} /></label>
        <label style={label}>Ocupação do exemplo (CBO)
          <select value={cboCode} onChange={(e) => setCboCode(e.target.value)} style={inputStyle}>
            <option value="">—</option>
            {(cbo.data ?? []).map((c) => <option key={c.code} value={c.code}>{`${c.code} · ${c.title}`}</option>)}
          </select>
        </label>
      </div>
      <div>
        <button type="button" disabled={!ready} onClick={() => void run()} style={ready ? secondaryButtonStyle : disabledButtonStyle}>
          Pré-visualizar
        </button>
      </div>
      {blocked && <small style={hint}>corrija o modelo para pré-visualizar</small>}
      {shown && (shown.slots.length === 0
        ? <p style={hint}>nenhuma vaga — confira o tipo das faixas e se o CBO do exemplo é atendido por ele</p>
        : (
          <div style={{ display: "flex", flexDirection: "column", gap: 4 }}>
            <span style={{ fontSize: 12.5, fontWeight: 600 }}>{shown.slots.length === 1 ? "1 vaga" : `${shown.slots.length} vagas`}</span>
            {shown.slots.map((s) => (
              <span key={s.starts_at} style={{ fontSize: 12.5 }}>
                {`${fmtHourMinute(s.starts_at)}–${fmtHourMinute(s.ends_at)} · ${typeLabel(s.appointment_type_key, types)}`}
              </span>
            ))}
          </div>
        ))}
      {shown && (
        <div style={{ display: "flex", flexDirection: "column", gap: 2 }}>
          <span style={{ fontSize: 12, color: "var(--ink3)" }}>Faixas no turno de exemplo</span>
          {shown.blocks.map((b, i) => <span key={i} style={{ fontSize: 12 }}>{blockLine(b, types)}</span>)}
        </div>
      )}
    </div>
  );
}

const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const box: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 12, border: "1px solid var(--rule)", borderRadius: 8 };
const card: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, padding: 10, border: "1px solid var(--rule)", borderRadius: 8 };
```

```tsx
// src/modules/professionals/ScheduleTemplates.tsx
// Modelos de agenda (módulo 17; spec §3.2): lista e editor. Só
// municipal_admin. Mudar um modelo nunca apaga nem move horário marcado
// (ADR 0029): o que sair do modelo aparece como "fora do modelo" na agenda.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { listAppointmentTypes, listScheduleTemplates, updateScheduleTemplate, type ScheduleTemplate } from "../../lib/api";
import { professionalError } from "../../lib/professionals";
import { blockLine } from "../../lib/scheduling";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { EmptyState } from "../../components/EmptyState";
import { Tag } from "../../components/Tag";
import { buttonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { APPOINTMENT_TYPES_KEY } from "./AppointmentTypes";
import { TemplateEditor } from "./TemplateEditor";

export const SCHEDULE_TEMPLATES_KEY = [ "scheduleTemplates" ] as const;

export function ScheduleTemplates() {
  const queryClient = useQueryClient();
  const templates = useQuery({ queryKey: SCHEDULE_TEMPLATES_KEY, queryFn: listScheduleTemplates });
  const types = useQuery({ queryKey: APPOINTMENT_TYPES_KEY, queryFn: listAppointmentTypes });
  const [ editing, setEditing ] = useState<ScheduleTemplate | "new" | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const refresh = () => void queryClient.invalidateQueries({ queryKey: SCHEDULE_TEMPLATES_KEY });

  async function toggle(t: ScheduleTemplate) {
    setError(null);
    try {
      await updateScheduleTemplate(t.id, { active: !t.active });
      refresh();
    } catch (err) {
      setError(professionalError(err));
    }
  }

  const loadError = templates.error ?? types.error;

  return (
    <Panel title="Modelos de agenda" sub="faixas do turno · limite de encaixes"
      right={<button type="button" style={buttonStyle} onClick={() => setEditing("new")}>Novo modelo</button>}>
      <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
        {error && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{error}</p>}
        {loadError ? <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{professionalError(loadError)}</p>
          : templates.isPending || types.isPending ? <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando…</p>
          : (templates.data ?? []).length === 0 ? <EmptyState title="nenhum modelo — turno sem modelo vira vagas do tipo padrão" />
          : (
            <DataTable<ScheduleTemplate>
              cols={[
                { label: "Nome", w: "1.5fr", render: (t) => t.name },
                { label: "Faixas", w: "4fr", render: (t) => (
                  <span style={{ display: "flex", flexDirection: "column", gap: 2 }}>
                    {t.blocks.map((b, i) => <span key={i}>{blockLine(b, types.data ?? [])}</span>)}
                  </span>
                ) },
                { label: "Encaixes", w: "1fr", render: (t) => `${t.fit_in_limit} por turno` },
                { label: "Situação", w: "1fr", render: (t) => t.active ? "ativo" : <Tag tone="warn">inativo</Tag> },
                { label: "", w: "auto", align: "right", render: (t) => (
                  <span style={{ display: "flex", gap: 6 }}>
                    <button type="button" style={secondaryButtonStyle} onClick={() => setEditing(t)}>Editar</button>
                    <button type="button" style={secondaryButtonStyle} onClick={() => void toggle(t)}>
                      {t.active ? "Desativar" : "Reativar"}
                    </button>
                  </span>
                ) }
              ]}
              rows={templates.data ?? []}
              rowKey={(t) => t.id}
            />
          )}
        {editing && types.data && (
          <TemplateEditor
            key={editing === "new" ? "new" : editing.id}
            template={editing === "new" ? null : editing}
            types={types.data}
            onSaved={() => { setEditing(null); refresh(); }}
            onCancel={() => setEditing(null)}
          />
        )}
      </div>
    </Panel>
  );
}
```

Em `src/modules/Professionals.tsx`, a terceira aba:

```tsx
import { ScheduleTemplates } from "./professionals/ScheduleTemplates";

type ProfessionalsTab = "people" | "types" | "templates";
const TABS: { key: ProfessionalsTab; label: string }[] = [
  { key: "people", label: "Profissionais" },
  { key: "types", label: "Tipos de atendimento" },
  { key: "templates", label: "Modelos de agenda" }
];
```

e, no `return` de `Professionals`, depois da linha de `types`:

```tsx
      {tab === "templates" && <ScheduleTemplates />}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/modules/professionals src/modules/Professionals.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod17
/opt/homebrew/bin/git add src/modules/professionals/TemplateEditor.tsx src/modules/professionals/TemplateEditor.test.tsx \
  src/modules/professionals/ScheduleTemplates.tsx src/modules/professionals/ScheduleTemplates.test.tsx src/modules/Professionals.tsx
/opt/homebrew/bin/git commit -m "feat: edit schedule templates with local validation and preview

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Modelo no turno e tipo padrão no vínculo

**Files:**
- Modify: `src/lib/scheduling.ts` (acrescenta `typeServes`), `src/lib/scheduling.test.ts`
- Modify: `src/modules/professionals/ProfessionalDetail.tsx`
- Test: `src/modules/professionals/ProfessionalDetail.test.tsx`

**Interfaces:**
- Consumes: `listScheduleTemplates`, `listAppointmentTypes`, `setShiftTemplate`, `setLinkDefaultType`, `scheduleShift(linkId, startsAt, endsAt, templateId?)`, `ProfessionalShift.schedule_template_id`, `ProfessionalLink.default_appointment_type_key` (Task 1); `typeLabel` (Task 2); `APPOINTMENT_TYPES_KEY` (Task 3); `SCHEDULE_TEMPLATES_KEY` (Task 4).
- Produces: `typeServes(type: AppointmentType, cboCode: string): boolean` em `src/lib/scheduling.ts`.

- [ ] **Step 1: Write the failing tests**

Em `src/lib/scheduling.test.ts` (importe `typeServes`):

```ts
describe("tipo serve o CBO", () => {
  it("por prefixo do grupo", () => {
    expect(typeServes(TYPES[0], "225125")).toBe(true);
    expect(typeServes(TYPES[0], "223505")).toBe(false);
    expect(typeServes(TYPES[1], "223505")).toBe(true);
  });
});
```

Em `src/modules/professionals/ProfessionalDetail.test.tsx`:
- no `vi.mock`, acrescente `listScheduleTemplates: vi.fn(), listAppointmentTypes: vi.fn(), setShiftTemplate: vi.fn(), setLinkDefaultType: vi.fn()`;
- no `beforeEach`, acrescente:

```ts
    mocked(api.listScheduleTemplates).mockResolvedValue([ MORNING, { ...MORNING, id: "t2", name: "Antigo", active: false } ]);
    mocked(api.listAppointmentTypes).mockResolvedValue(TYPES);
```

  com `import { MORNING, TYPES } from "../../test/schedulingFixtures";` no topo;
- na linha 150, o lançamento sem modelo passa a mandar `null`:

```ts
    await waitFor(() => expect(api.scheduleShift).toHaveBeenCalledWith("l1", "2026-10-06T19:00:00-03:00", "2026-10-07T07:00:00-03:00", null));
```

- e os testes novos, no fim do `describe`:

```tsx
  it("lançar turno com modelo manda o id; só modelos ativos aparecem", async () => {
    mocked(api.scheduleShift).mockResolvedValue({} as api.ProfessionalShift);
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Lançar turno" }));
    const select = screen.getByLabelText("Modelo de agenda") as HTMLSelectElement;
    await screen.findByRole("option", { name: "Manhã" });
    expect(screen.queryByRole("option", { name: "Antigo" })).toBeNull();
    fireEvent.change(screen.getByLabelText("Data"), { target: { value: "2026-10-06" } });
    fireEvent.change(screen.getByLabelText("Início"), { target: { value: "07:00" } });
    fireEvent.change(screen.getByLabelText("Fim"), { target: { value: "12:00" } });
    fireEvent.change(select, { target: { value: "t1" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar turno" }));
    await waitFor(() => expect(api.scheduleShift).toHaveBeenCalledWith("l1", "2026-10-06T07:00:00-03:00", "2026-10-06T12:00:00-03:00", "t1"));
  });

  it("turno mostra o modelo; trocar o modelo grava e avisa que horários marcados ficam", async () => {
    mocked(api.listProfessionalShifts).mockResolvedValue([
      { id: "s1", professional_link_id: "l1", unit_name: "UBS Jardim", starts_at: "2026-10-06T10:00:00Z",
        ends_at: "2026-10-06T15:00:00Z", cancelled_at: null, cancel_reason: null, schedule_template_id: "t2" }
    ]);
    mocked(api.setShiftTemplate).mockResolvedValue(undefined);
    renderIt();
    expect(await screen.findByText("Antigo (inativo)")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Modelo" }));
    expect(screen.getByText("Horários já marcados ficam como estão; os que saírem do modelo aparecem como “fora do modelo”.")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Modelo do turno"), { target: { value: "" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar modelo do turno" }));
    await waitFor(() => expect(api.setShiftTemplate).toHaveBeenCalledWith("s1", null));
    await waitFor(() => expect(api.listProfessionalShifts).toHaveBeenCalledTimes(2));
  });

  it("vínculo mostra o tipo padrão e só oferece tipos que servem o CBO", async () => {
    mocked(api.getProfessional).mockResolvedValue({
      professional: { id: "p1", user_id: "u1", email_address: "medica@c.gov.br", professional_name: "Helena Duarte",
        council: "CRM", council_state: "PR", registration_number: "12345", cns_masked: "*** **** **** 0005",
        cns: "700000000000005", phone: null, contact_email: null },
      links: [ { ...link, default_appointment_type_key: null } ]
    });
    mocked(api.setLinkDefaultType).mockResolvedValue(undefined);
    renderIt();
    expect(await screen.findByText("pelo CBO")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Tipo padrão" }));
    const select = screen.getByLabelText("Tipo padrão do vínculo") as HTMLSelectElement;
    await within(select).findByRole("option", { name: "Consulta médica" });
    expect(within(select).queryByRole("option", { name: "Consulta de enfermagem" })).toBeNull();
    fireEvent.change(select, { target: { value: "retorno" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar tipo padrão" }));
    await waitFor(() => expect(api.setLinkDefaultType).toHaveBeenCalledWith("l1", "retorno"));
  });
```

(`within` entra no import de `@testing-library/react`.)

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/lib/scheduling.test.ts src/modules/professionals/ProfessionalDetail.test.tsx`
Expected: FAIL — `typeServes` não existe; "Modelo de agenda", "Modelo" e "Tipo padrão" não estão na tela; `scheduleShift` chamado com três argumentos.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/scheduling.ts`:

```ts
// Grupos de CBO são prefixos (spec §3.1).
export function typeServes(type: AppointmentType, cboCode: string): boolean {
  return type.cbo_prefixes.some((p) => cboCode.startsWith(p));
}
```

Em `src/modules/professionals/ProfessionalDetail.tsx`:

1. Imports:

```tsx
import {
  cancelShift, endProfessionalLink, getProfessional, listActiveUnits, listAppointmentTypes, listCbo, listProfessionalShifts,
  listScheduleTemplates, openProfessionalLink, scheduleShift, setLinkDefaultType, setShiftTemplate, updateProfessional,
  type AppointmentType, type CboEntry, type HealthUnit, type ProfessionalLink, type ProfessionalShift, type ScheduleTemplate
} from "../../lib/api";
import { typeLabel, typeServes } from "../../lib/scheduling";
import { APPOINTMENT_TYPES_KEY } from "./AppointmentTypes";
import { SCHEDULE_TEMPLATES_KEY } from "./ScheduleTemplates";
```

2. Em `ProfessionalDetail`, junto das outras consultas e estados:

```tsx
  const templates = useQuery({ queryKey: SCHEDULE_TEMPLATES_KEY, queryFn: listScheduleTemplates });
  const types = useQuery({ queryKey: APPOINTMENT_TYPES_KEY, queryFn: listAppointmentTypes });
  const [ templateFor, setTemplateFor ] = useState<ProfessionalShift | null>(null);
  const [ defaultFor, setDefaultFor ] = useState<ProfessionalLink | null>(null);
  const templateName = (id: string | null | undefined) => {
    if (!id) return "sem modelo";
    const t = (templates.data ?? []).find((x) => x.id === id);
    return t ? (t.active ? t.name : `${t.name} (inativo)`) : "modelo removido";
  };
```

3. Tabela de vínculos: coluna nova depois de "Ocupação" e botão novo na coluna de ações:

```tsx
            { label: "Tipo padrão", w: "1.5fr", render: (l) =>
              l.default_appointment_type_key ? typeLabel(l.default_appointment_type_key, types.data ?? null) : "pelo CBO" },
```

```tsx
                <button type="button" style={secondaryButtonStyle} onClick={() => setDefaultFor(l)}>Tipo padrão</button>
```

(o botão entra antes de "Lançar turno", dentro do mesmo `<span>`), e, depois do bloco `{ending && ( … )}`:

```tsx
        {defaultFor && (
          <DefaultTypePanel
            key={defaultFor.id}
            link={defaultFor}
            types={types.data ?? []}
            onDone={() => { setDefaultFor(null); refresh(); }}
            onCancel={() => setDefaultFor(null)}
          />
        )}
```

4. `ScheduleShift` recebe os modelos: `<ScheduleShift … templates={templates.data ?? []} … />`.

5. Tabela de turnos: coluna nova depois de "Fim" e botão "Modelo" nos turnos válidos:

```tsx
              { label: "Modelo", w: "1.5fr", render: (s) => templateName(s.schedule_template_id) },
```

```tsx
              { label: "", w: "auto", align: "right", render: (s) => !s.cancelled_at && (
                <span style={{ display: "flex", gap: 6 }}>
                  <button type="button" style={secondaryButtonStyle} onClick={() => setTemplateFor(s)}>Modelo</button>
                  <button type="button" style={secondaryButtonStyle} onClick={() => setCancelling(s)}>Cancelar</button>
                </span>
              ) }
```

e, depois do `{cancelling && …}`:

```tsx
        {templateFor && (
          <ShiftTemplatePanel
            key={templateFor.id}
            shift={templateFor}
            templates={templates.data ?? []}
            onDone={() => { setTemplateFor(null); refresh(); }}
            onCancel={() => setTemplateFor(null)}
          />
        )}
```

6. `ScheduleShift` ganha o select (props `templates: ScheduleTemplate[]`), estado `const [ templateId, setTemplateId ] = useState("");`, o campo depois das horas:

```tsx
      <label style={labelStyle}>Modelo de agenda
        <select value={templateId} onChange={(e) => setTemplateId(e.target.value)} style={inputStyle}>
          <option value="">sem modelo (vagas do tipo padrão)</option>
          {templates.filter((t) => t.active).map((t) => <option key={t.id} value={t.id}>{t.name}</option>)}
        </select>
      </label>
```

e a chamada: `await scheduleShift(link.id, span.startsAt, span.endsAt, templateId === "" ? null : templateId);`.

7. Os dois painéis novos, no fim do arquivo (antes das constantes de estilo):

```tsx
function ShiftTemplatePanel({ shift, templates, onDone, onCancel }: {
  shift: ProfessionalShift; templates: ScheduleTemplate[]; onDone(): void; onCancel(): void;
}) {
  const [ value, setValue ] = useState(shift.schedule_template_id ?? "");
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const current = templates.find((t) => t.id === shift.schedule_template_id);

  async function save() {
    if (busy) return;
    setBusy(true); setError(null);
    try {
      await setShiftTemplate(shift.id, value === "" ? null : value);
      onDone();
    } catch (err) {
      setError(professionalError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <section style={{ ...panelStyle, gap: 10 }}>
      <strong>{`Modelo do turno de ${fmtDateTime(shift.starts_at)}`}</strong>
      {error && <p role="alert" style={alertStyle}>{error}</p>}
      <label style={labelStyle}>Modelo do turno
        <select value={value} onChange={(e) => setValue(e.target.value)} style={inputStyle}>
          <option value="">sem modelo (vagas do tipo padrão)</option>
          {templates.filter((t) => t.active).map((t) => <option key={t.id} value={t.id}>{t.name}</option>)}
          {current && !current.active && <option value={current.id}>{`${current.name} (inativo)`}</option>}
        </select>
      </label>
      <p style={{ margin: 0, fontSize: 12.5 }}>
        Horários já marcados ficam como estão; os que saírem do modelo aparecem como “fora do modelo”.
      </p>
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={busy} onClick={() => void save()} style={busy ? disabledButtonStyle : buttonStyle}>
          Salvar modelo do turno
        </button>
        <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Voltar</button>
      </div>
    </section>
  );
}

function DefaultTypePanel({ link, types, onDone, onCancel }: {
  link: ProfessionalLink; types: AppointmentType[]; onDone(): void; onCancel(): void;
}) {
  const [ value, setValue ] = useState(link.default_appointment_type_key ?? "");
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const serving = types.filter((t) => t.active && typeServes(t, link.cbo_code));
  const current = link.default_appointment_type_key;

  async function save() {
    if (busy) return;
    setBusy(true); setError(null);
    try {
      await setLinkDefaultType(link.id, value === "" ? null : value);
      onDone();
    } catch (err) {
      setError(professionalError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <section style={{ ...panelStyle, gap: 10 }}>
      <strong>{`Tipo padrão em ${link.unit_name}`}</strong>
      {error && <p role="alert" style={alertStyle}>{error}</p>}
      <label style={labelStyle}>Tipo padrão do vínculo
        <select value={value} onChange={(e) => setValue(e.target.value)} style={inputStyle}>
          <option value="">pelo CBO (tipo da base)</option>
          {serving.map((t) => <option key={t.key} value={t.key}>{t.name}</option>)}
          {current && !serving.some((t) => t.key === current) && (
            <option value={current}>{typeLabel(current, types)}</option>
          )}
        </select>
      </label>
      <p style={{ margin: 0, fontSize: 12.5 }}>Vale para turno sem modelo: o turno inteiro vira vagas deste tipo.</p>
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={busy} onClick={() => void save()} style={busy ? disabledButtonStyle : buttonStyle}>
          Salvar tipo padrão
        </button>
        <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Voltar</button>
      </div>
    </section>
  );
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/lib/scheduling.test.ts src/modules/professionals && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod17
/opt/homebrew/bin/git add src/lib/scheduling.ts src/lib/scheduling.test.ts \
  src/modules/professionals/ProfessionalDetail.tsx src/modules/professionals/ProfessionalDetail.test.tsx
/opt/homebrew/bin/git commit -m "feat: pick a schedule template per shift and a default type per link

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 6: Painel "Agendamento" no editor de protocolo

**Files:**
- Create: `src/lib/schedulingRules.ts`, `src/modules/protocolEditor/SchedulingPanel.tsx`
- Modify: `src/modules/ProtocolEditor.tsx`, `src/modules/ProtocolEditor.test.tsx` (mock e testes novos)
- Test: `src/lib/schedulingRules.test.ts`, `src/modules/protocolEditor/SchedulingPanel.test.tsx`

**Interfaces:**
- Consumes: `ConditionTree`, `AppointmentType`, `SchedulingPriority`, `listAppointmentTypes`, `GateResult.warnings` (Task 1); `TYPE_KEY_PATTERN`, `PRIORITY_LABEL`, `typeLabel` (Task 2); `ConditionBuilder` (`src/modules/protocols/ConditionBuilder.tsx`, módulo 15) e `fieldsFor("suggestion", { definition })` (`src/lib/condition.ts`): o `when` de `scheduling` usa as mesmas variáveis das sugestões (`profile.*`, `outcome.*`, ids de passo).
- Produces (`src/lib/schedulingRules.ts`): `SchedulingRuleDraft { when: ConditionTree | null; appointmentType: string; priority: SchedulingPriority; dueInDays: number | null }`, `SCHEDULING_MAX = 10`, `DUE_MIN = 1`, `DUE_MAX = 365`, `readScheduling(definition: unknown): { ok: true; rules: SchedulingRuleDraft[] } | { ok: false; reason: string }`, `writeScheduling(definition: unknown, rules: SchedulingRuleDraft[]): unknown`, `parseDueDays(text: string): { days: number | null; problem: string | null }`, `schedulingRuleProblem(rule: SchedulingRuleDraft, types: AppointmentType[] | null): string | null`, `moveRule(list: SchedulingRuleDraft[], i: number, delta: -1 | 1): SchedulingRuleDraft[]`. `SchedulingPanel({ definition, types, onChange })`.

- [ ] **Step 1: Write the failing tests**

```ts
// src/lib/schedulingRules.test.ts
import { describe, expect, it } from "vitest";
import { moveRule, parseDueDays, readScheduling, schedulingRuleProblem, writeScheduling, type SchedulingRuleDraft } from "./schedulingRules";
import { TYPES } from "../test/schedulingFixtures";

const WHEN = { gte: [ "profile.age", 60 ] };
const RULE: SchedulingRuleDraft = { when: WHEN, appointmentType: "consulta_medica", priority: "routine", dueInDays: 30 };

describe("bloco scheduling na definição", () => {
  it("sem scheduling: lista vazia; com regras: rascunho", () => {
    expect(readScheduling({ name: "x" })).toEqual({ ok: true, rules: [] });
    expect(readScheduling({ scheduling: [ { when: WHEN, appointment_type: "consulta_medica", priority: "routine", due_in_days: 30 } ] }))
      .toEqual({ ok: true, rules: [ RULE ] });
  });

  it("campo ausente vira vazio no rascunho; prioridade ausente vira rotina", () => {
    expect(readScheduling({ scheduling: [ {} ] })).toEqual({ ok: true,
      rules: [ { when: null, appointmentType: "", priority: "routine", dueInDays: null } ] });
  });

  it("formato errado: motivo, sem tocar no JSON", () => {
    expect(readScheduling({ scheduling: {} })).toEqual({ ok: false, reason: "“scheduling” não é uma lista: corrija no JSON" });
    expect(readScheduling({ scheduling: [ { due_in_days: "30" } ] }).ok).toBe(false);
    expect(readScheduling({ scheduling: [ { priority: "urgent" } ] }).ok).toBe(false);
  });

  it("escrita: ordem das chaves do contrato, vazio sai ausente, sem when e sem prazo saem ausentes", () => {
    const def = { name: "x", scheduling: [] as unknown[] };
    expect(writeScheduling(def, [ RULE ])).toEqual({ name: "x",
      scheduling: [ { when: WHEN, appointment_type: "consulta_medica", priority: "routine", due_in_days: 30 } ] });
    expect("scheduling" in (writeScheduling(def, []) as object)).toBe(false);
    expect(writeScheduling({}, [ { ...RULE, when: null, dueInDays: null, appointmentType: "" } ]))
      .toEqual({ scheduling: [ { priority: "routine" } ] });
  });

  it("prazo: inteiro de 1 a 365", () => {
    expect(parseDueDays("30")).toEqual({ days: 30, problem: null });
    expect(parseDueDays("")).toEqual({ days: null, problem: null });
    expect(parseDueDays("0").problem).toBe("use de 1 a 365 dias");
    expect(parseDueDays("366").problem).toBe("use de 1 a 365 dias");
    expect(parseDueDays("1,5").problem).toBe("use um número inteiro de dias");
  });

  it("problema da regra, na ordem: tipo, padrão, condição, prazo; tipo inexistente e inativo avisam", () => {
    expect(schedulingRuleProblem(RULE, TYPES)).toBeNull();
    expect(schedulingRuleProblem({ ...RULE, appointmentType: "" }, TYPES)).toBe("escolha o tipo de atendimento");
    expect(schedulingRuleProblem({ ...RULE, appointmentType: "Consulta" }, null)).toBe("tipo inválido: minúsculas, números e _");
    expect(schedulingRuleProblem({ ...RULE, when: null }, TYPES)).toBe("defina quando gerar o pedido");
    expect(schedulingRuleProblem({ ...RULE, dueInDays: null }, TYPES)).toBe("informe o prazo (1 a 365 dias)");
    expect(schedulingRuleProblem({ ...RULE, appointmentType: "sumiu" }, TYPES))
      .toBe("tipo não existe nesta cidade: o gate avisa, sem bloquear");
    expect(schedulingRuleProblem({ ...RULE, appointmentType: "puericultura" }, TYPES)).toBe("tipo inativo nesta cidade");
    expect(schedulingRuleProblem({ ...RULE, appointmentType: "sumiu" }, null)).toBeNull();
  });

  it("mover regra: a ordem decide (vale a primeira que casar)", () => {
    const b = { ...RULE, appointmentType: "retorno" };
    expect(moveRule([ RULE, b ], 1, -1)).toEqual([ b, RULE ]);
    expect(moveRule([ RULE, b ], 0, -1)).toEqual([ RULE, b ]);
  });
});
```

```tsx
// src/modules/protocolEditor/SchedulingPanel.test.tsx
import { afterEach, describe, expect, it } from "vitest";
import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
import { useState } from "react";
import { SchedulingPanel } from "./SchedulingPanel";
import { TYPES } from "../../test/schedulingFixtures";
import { SUGGESTION_DEF } from "../../test/conditionFixtures";
import type { AppointmentType } from "../../lib/api";

afterEach(cleanup);

function Harness({ initial, types }: { initial: unknown; types: AppointmentType[] | null }) {
  const [ def, setDef ] = useState<unknown>(initial);
  return (
    <>
      <SchedulingPanel definition={def} types={types} onChange={setDef} />
      <pre data-testid="json">{JSON.stringify(def)}</pre>
    </>
  );
}
const json = () => JSON.parse(screen.getByTestId("json").textContent ?? "null");
const rule = (n: number) => screen.getByRole("group", { name: `regra ${n}` });

describe("SchedulingPanel", () => {
  it("+ regra monta tipo, prioridade, prazo e condição no JSON", () => {
    render(<Harness initial={SUGGESTION_DEF} types={TYPES} />);
    fireEvent.click(screen.getByRole("button", { name: "+ regra de agendamento" }));
    fireEvent.change(within(rule(1)).getByLabelText("Tipo de atendimento"), { target: { value: "consulta_medica" } });
    fireEvent.change(within(rule(1)).getByLabelText("Prioridade"), { target: { value: "priority" } });
    fireEvent.change(within(rule(1)).getByLabelText("Prazo (dias)"), { target: { value: "30" } });
    const when = within(rule(1)).getByRole("group", { name: "Quando gerar o pedido" });
    fireEvent.click(within(when).getByRole("button", { name: "+ condição" }));
    fireEvent.change(within(when).getByLabelText("valor"), { target: { value: "60" } });
    expect(json().scheduling).toEqual([
      { when: { gte: [ "profile.age", 60 ] }, appointment_type: "consulta_medica", priority: "priority", due_in_days: 30 }
    ]);
  });

  it("prazo inválido mostra o motivo e não chega ao JSON", () => {
    render(<Harness initial={{ ...SUGGESTION_DEF, scheduling: [ { when: { gte: [ "profile.age", 60 ] },
      appointment_type: "consulta_medica", priority: "routine", due_in_days: 30 } ] }} types={TYPES} />);
    fireEvent.change(within(rule(1)).getByLabelText("Prazo (dias)"), { target: { value: "400" } });
    expect(within(rule(1)).getByText("use de 1 a 365 dias")).not.toBeNull();
    expect(json().scheduling[0].due_in_days).toBe(30);
  });

  it("subir troca a ordem; remover a última tira o bloco do JSON", () => {
    const a = { when: { gte: [ "profile.age", 60 ] }, appointment_type: "consulta_medica", priority: "routine", due_in_days: 30 };
    const b = { ...a, appointment_type: "retorno" };
    render(<Harness initial={{ ...SUGGESTION_DEF, scheduling: [ a, b ] }} types={TYPES} />);
    fireEvent.click(within(rule(2)).getByRole("button", { name: "subir" }));
    expect(json().scheduling.map((r: { appointment_type: string }) => r.appointment_type)).toEqual([ "retorno", "consulta_medica" ]);
    fireEvent.click(within(rule(2)).getByRole("button", { name: "remover regra" }));
    fireEvent.click(within(rule(1)).getByRole("button", { name: "remover regra" }));
    expect("scheduling" in json()).toBe(false);
  });

  it("regra com tipo inativo ou desconhecido continua no select", () => {
    const base = { when: { gte: [ "profile.age", 60 ] }, priority: "routine", due_in_days: 30 };
    render(<Harness initial={{ ...SUGGESTION_DEF, scheduling: [
      { ...base, appointment_type: "puericultura" }, { ...base, appointment_type: "sumiu" } ] }} types={TYPES} />);
    const first = within(rule(1)).getByLabelText("Tipo de atendimento") as HTMLSelectElement;
    expect(first.value).toBe("puericultura");
    expect(within(first).getByRole("option", { name: "Puericultura (inativo)" })).not.toBeNull();
    const second = within(rule(2)).getByLabelText("Tipo de atendimento") as HTMLSelectElement;
    expect(second.value).toBe("sumiu");
    expect(within(second).getByRole("option", { name: "sumiu (não existe na cidade)" })).not.toBeNull();
    expect(within(rule(2)).getByText("tipo não existe nesta cidade: o gate avisa, sem bloquear")).not.toBeNull();
    expect(json().scheduling[1].appointment_type).toBe("sumiu");
  });

  it("sem lista de tipos, o tipo é texto livre com o padrão da chave", () => {
    render(<Harness initial={SUGGESTION_DEF} types={null} />);
    fireEvent.click(screen.getByRole("button", { name: "+ regra de agendamento" }));
    const field = within(rule(1)).getByLabelText("Tipo de atendimento (chave)") as HTMLInputElement;
    expect(field.tagName).toBe("INPUT");
    fireEvent.change(field, { target: { value: "Consulta" } });
    expect(within(rule(1)).getByText("tipo inválido: minúsculas, números e _")).not.toBeNull();
    fireEvent.change(field, { target: { value: "consulta_medica" } });
    expect(json().scheduling[0].appointment_type).toBe("consulta_medica");
    expect(screen.getByText("Sem acesso à lista de tipos da cidade: digite a chave (o gate avisa se ela não existir).")).not.toBeNull();
  });

  it("no limite de 10 regras o botão trava", () => {
    const r = { when: { gte: [ "profile.age", 60 ] }, appointment_type: "retorno", priority: "routine", due_in_days: 30 };
    render(<Harness initial={{ ...SUGGESTION_DEF, scheduling: Array.from({ length: 10 }, () => r) }} types={TYPES} />);
    expect((screen.getByRole("button", { name: "+ regra de agendamento" }) as HTMLButtonElement).disabled).toBe(true);
  });

  it("scheduling fora do formato: motivo, sem controles", () => {
    render(<Harness initial={{ ...SUGGESTION_DEF, scheduling: { a: 1 } }} types={TYPES} />);
    expect(screen.getByRole("alert").textContent).toBe("“scheduling” não é uma lista: corrija no JSON");
    expect(screen.queryByRole("button", { name: "+ regra de agendamento" })).toBeNull();
    expect(json().scheduling).toEqual({ a: 1 });
  });
});
```

Em `src/modules/ProtocolEditor.test.tsx`: acrescente `listAppointmentTypes: vi.fn()` ao `vi.mock`, e no fim do arquivo:

```tsx
describe("ProtocolEditor — Agendamento", () => {
  beforeEach(() => {
    vi.resetAllMocks();
    mocked(api.listAuthorProtocols).mockResolvedValue([]);
    mocked(api.gateProtocol).mockResolvedValue({ valid: true });
  });

  it("o painel aparece ao lado do JSON e usa a lista de tipos quando o papel lê", async () => {
    mocked(api.listAppointmentTypes).mockResolvedValue([
      { key: "consulta_medica", name: "Consulta médica", duration_minutes: 20, cbo_prefixes: [ "2251" ], active: true, origin: "platform" }
    ]);
    render(<ProtocolEditor />);
    typeDefinition(DEF);
    fireEvent.click(screen.getByRole("button", { name: "+ regra de agendamento" }));
    expect(await screen.findByRole("option", { name: "Consulta médica" })).not.toBeNull();
  });

  it("403 nos tipos: o painel continua, com campo de texto", async () => {
    mocked(api.listAppointmentTypes).mockRejectedValue(new api.ApiError(403, { error: "forbidden" }, "x"));
    render(<ProtocolEditor />);
    typeDefinition(DEF);
    fireEvent.click(screen.getByRole("button", { name: "+ regra de agendamento" }));
    expect(await screen.findByLabelText("Tipo de atendimento (chave)")).not.toBeNull();
  });

  it("avisos do gate aparecem sem tirar o 'válido'", async () => {
    mocked(api.listAppointmentTypes).mockResolvedValue([]);
    mocked(api.gateProtocol).mockResolvedValue({ valid: true,
      warnings: [ "scheduling[0].appointment_type: tipo inexistente na cidade (puericultura)" ] });
    render(<ProtocolEditor />);
    typeDefinition(DEF);
    expect(await screen.findByText("aviso: scheduling[0].appointment_type: tipo inexistente na cidade (puericultura)", {}, { timeout: 2000 }))
      .not.toBeNull();
    expect(screen.getByText("válido ✓")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/lib/schedulingRules.test.ts src/modules/protocolEditor/SchedulingPanel.test.tsx src/modules/ProtocolEditor.test.tsx`
Expected: FAIL — `./schedulingRules` e `./SchedulingPanel` não existem; o editor não tem "+ regra de agendamento".

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/schedulingRules.ts
// Bloco `scheduling` do protocolo (módulo 17; schema protocols-v1.5.0,
// contratos §1). Mesmo padrão de offer.ts: o JSON do editor é a fonte; estas
// funções leem a definição e devolvem uma definição nova sem tocar no resto.
import type { AppointmentType, ConditionTree, SchedulingPriority } from "./api";
import { TYPE_KEY_PATTERN } from "./scheduling";

export interface SchedulingRuleDraft {
  when: ConditionTree | null; appointmentType: string; priority: SchedulingPriority; dueInDays: number | null;
}
export type SchedulingRead = { ok: true; rules: SchedulingRuleDraft[] } | { ok: false; reason: string };

export const SCHEDULING_MAX = 10;
export const DUE_MIN = 1;
export const DUE_MAX = 365;

type Obj = Record<string, unknown>;
const isObj = (v: unknown): v is Obj => !!v && typeof v === "object" && !Array.isArray(v);
const fail = (reason: string): SchedulingRead => ({ ok: false, reason });

export function readScheduling(definition: unknown): SchedulingRead {
  if (!isObj(definition)) return fail("a definição precisa ser um objeto JSON");
  const list = definition.scheduling;
  if (list === undefined) return { ok: true, rules: [] };
  if (!Array.isArray(list)) return fail("“scheduling” não é uma lista: corrija no JSON");
  const malformed = list.some((r) => !isObj(r) ||
    (r.when !== undefined && !isObj(r.when)) ||
    (r.appointment_type !== undefined && typeof r.appointment_type !== "string") ||
    (r.priority !== undefined && r.priority !== "routine" && r.priority !== "priority") ||
    (r.due_in_days !== undefined && !Number.isInteger(r.due_in_days)));
  if (malformed) {
    return fail("uma regra de agendamento está fora do formato { when, appointment_type, priority, due_in_days }: corrija no JSON");
  }
  return {
    ok: true,
    rules: (list as Obj[]).map((r) => ({
      when: (r.when as ConditionTree | undefined) ?? null,
      appointmentType: (r.appointment_type as string | undefined) ?? "",
      priority: (r.priority as SchedulingPriority | undefined) ?? "routine",
      dueInDays: (r.due_in_days as number | undefined) ?? null
    }))
  };
}

// Campo incompleto sai ausente: o gate do api aponta o erro, e o painel
// mostra o motivo ao lado (como as sugestões do módulo 15).
export function writeScheduling(definition: unknown, rules: SchedulingRuleDraft[]): unknown {
  const next: Obj = { ...(definition as Obj) };
  if (rules.length === 0) { delete next.scheduling; return next; }
  next.scheduling = rules.map((r) => {
    const out: Obj = {};
    if (r.when !== null) out.when = r.when;
    if (r.appointmentType !== "") out.appointment_type = r.appointmentType;
    out.priority = r.priority;
    if (r.dueInDays !== null) out.due_in_days = r.dueInDays;
    return out;
  });
  return next;
}

export function parseDueDays(text: string): { days: number | null; problem: string | null } {
  const t = text.trim();
  if (t === "") return { days: null, problem: null };
  if (!/^\d+$/.test(t)) return { days: null, problem: "use um número inteiro de dias" };
  const n = Number(t);
  if (n < DUE_MIN || n > DUE_MAX) return { days: null, problem: `use de ${DUE_MIN} a ${DUE_MAX} dias` };
  return { days: n, problem: null };
}

export function schedulingRuleProblem(rule: SchedulingRuleDraft, types: AppointmentType[] | null): string | null {
  if (rule.appointmentType === "") return "escolha o tipo de atendimento";
  if (!TYPE_KEY_PATTERN.test(rule.appointmentType)) return "tipo inválido: minúsculas, números e _";
  if (rule.when === null) return "defina quando gerar o pedido";
  if (rule.dueInDays === null) return `informe o prazo (${DUE_MIN} a ${DUE_MAX} dias)`;
  if (types) {
    const t = types.find((x) => x.key === rule.appointmentType);
    if (!t) return "tipo não existe nesta cidade: o gate avisa, sem bloquear";
    if (!t.active) return "tipo inativo nesta cidade";
  }
  return null;
}

export function moveRule(list: SchedulingRuleDraft[], i: number, delta: -1 | 1): SchedulingRuleDraft[] {
  const j = i + delta;
  if (j < 0 || j >= list.length) return list;
  const next = [ ...list ];
  [ next[i], next[j] ] = [ next[j], next[i] ];
  return next;
}
```

```tsx
// src/modules/protocolEditor/SchedulingPanel.tsx
// Painel "Agendamento" (módulo 17; spec §5.1 e §7). Lê `scheduling` da
// definição a cada render e devolve uma definição nova a cada edição, como o
// painel "Oferta e sugestões". O conteúdo é parte da versão assinada (ADR 0016).
// `types` nulo = o papel não lê os tipos (403): o tipo vira texto com o padrão
// da chave, e o gate avisa tipo inexistente.
import { useEffect, useState, type CSSProperties, type ReactNode } from "react";
import type { AppointmentType, SchedulingPriority } from "../../lib/api";
import { ConditionBuilder } from "../protocols/ConditionBuilder";
import { fieldsFor } from "../../lib/condition";
import { PRIORITY_LABEL, typeLabel } from "../../lib/scheduling";
import {
  SCHEDULING_MAX, moveRule, parseDueDays, readScheduling, schedulingRuleProblem, writeScheduling, type SchedulingRuleDraft
} from "../../lib/schedulingRules";
import { disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";

interface Props { definition: unknown | null; types: AppointmentType[] | null; onChange(next: unknown): void }

export function SchedulingPanel({ definition, types, onChange }: Props) {
  if (definition === null) return <Section><p style={hint}>Corrija o JSON para editar o agendamento.</p></Section>;
  const read = readScheduling(definition);
  if (!read.ok) return <Section><p role="alert" style={alert}>{read.reason}</p></Section>;
  return <SchedulingForm definition={definition} rules={read.rules} types={types} onChange={onChange} />;
}

function SchedulingForm({ definition, rules, types, onChange }: {
  definition: unknown; rules: SchedulingRuleDraft[]; types: AppointmentType[] | null; onChange(next: unknown): void;
}) {
  const fields = fieldsFor("suggestion", { definition });
  const setRules = (list: SchedulingRuleDraft[]) => onChange(writeScheduling(definition, list));
  const replace = (i: number, next: SchedulingRuleDraft) => setRules(rules.map((r, j) => (j === i ? next : r)));
  const activeTypes = (types ?? []).filter((t) => t.active);

  return (
    <Section>
      <p style={hint}>Vale a primeira regra que casar. Resultado urgente nunca gera pedido. Sem regra, a triagem só orienta.</p>
      {types === null && (
        <p style={hint}>Sem acesso à lista de tipos da cidade: digite a chave (o gate avisa se ela não existir).</p>
      )}
      {rules.map((r, i) => {
        const problem = schedulingRuleProblem(r, types);
        return (
          <div key={i} role="group" aria-label={`regra ${i + 1}`} style={card}>
            {types ? (
              <label style={label}>Tipo de atendimento
                <select value={r.appointmentType} style={inputStyle} onChange={(e) => replace(i, { ...r, appointmentType: e.target.value })}>
                  <option value="">escolha…</option>
                  {activeTypes.map((t) => <option key={t.key} value={t.key}>{t.name}</option>)}
                  {r.appointmentType !== "" && !activeTypes.some((t) => t.key === r.appointmentType) && (
                    <option value={r.appointmentType}>{typeLabel(r.appointmentType, types)}</option>
                  )}
                </select>
              </label>
            ) : (
              <label style={label}>Tipo de atendimento (chave)
                <input value={r.appointmentType} style={inputStyle} placeholder="consulta_medica"
                  onChange={(e) => replace(i, { ...r, appointmentType: e.target.value.trim() })} />
              </label>
            )}
            <label style={label}>Prioridade
              <select value={r.priority} style={inputStyle}
                onChange={(e) => replace(i, { ...r, priority: e.target.value as SchedulingPriority })}>
                <option value="routine">{PRIORITY_LABEL.routine}</option>
                <option value="priority">{PRIORITY_LABEL.priority}</option>
              </select>
            </label>
            <DueField days={r.dueInDays} onChange={(days) => replace(i, { ...r, dueInDays: days })} />
            <ConditionBuilder label="Quando gerar o pedido" fields={fields} value={r.when} emptyText="—"
              onChange={(tree) => replace(i, { ...r, when: tree })} />
            {problem && <small style={hint}>{problem}</small>}
            <div style={{ display: "flex", gap: 8 }}>
              <button type="button" style={secondaryButtonStyle} disabled={i === 0} onClick={() => setRules(moveRule(rules, i, -1))}>subir</button>
              <button type="button" style={secondaryButtonStyle} disabled={i === rules.length - 1}
                onClick={() => setRules(moveRule(rules, i, 1))}>descer</button>
              <button type="button" style={secondaryButtonStyle}
                onClick={() => setRules(rules.filter((_, j) => j !== i))}>remover regra</button>
            </div>
          </div>
        );
      })}
      <div>
        <button type="button" disabled={rules.length >= SCHEDULING_MAX}
          style={rules.length >= SCHEDULING_MAX ? disabledButtonStyle : secondaryButtonStyle}
          onClick={() => setRules([ ...rules, { when: null, appointmentType: "", priority: "routine", dueInDays: null } ])}>
          + regra de agendamento
        </button>
      </div>
    </Section>
  );
}

// Texto digitado fica local: só valor válido (ou vazio) chega ao JSON.
function DueField({ days, onChange }: { days: number | null; onChange(days: number | null): void }) {
  const [ text, setText ] = useState(days === null ? "" : String(days));
  useEffect(() => {
    if (parseDueDays(text).days !== days) setText(days === null ? "" : String(days));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [ days ]);
  const { problem } = parseDueDays(text);
  return (
    <>
      <label style={label}>Prazo (dias)
        <input inputMode="numeric" value={text} style={inputStyle} onChange={(e) => {
          setText(e.target.value);
          const parsed = parseDueDays(e.target.value);
          if (!parsed.problem) onChange(parsed.days);
        }} />
      </label>
      {problem && <small role="alert" style={alert}>{problem}</small>}
    </>
  );
}

function Section({ children }: { children: ReactNode }) {
  return <section aria-label="Agendamento" style={{ display: "flex", flexDirection: "column", gap: 8 }}>{children}</section>;
}

const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12, color: "var(--down)" };
const card: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, padding: 10, border: "1px solid var(--rule)", borderRadius: 8 };
```

Em `src/modules/ProtocolEditor.tsx`:

1. Imports: `listAppointmentTypes, type AppointmentType` de `../lib/api` e `import { SchedulingPanel } from "./protocolEditor/SchedulingPanel";`.
2. Estado e carga (junto do `useEffect` de `listAuthorProtocols`):

```tsx
  // null = sem lista (o autor não é municipal_admin e recebe 403): o painel
  // usa texto livre. Promise.resolve protege contra mock sem implementação.
  const [ types, setTypes ] = useState<AppointmentType[] | null>(null);
  useEffect(() => {
    Promise.resolve().then(() => listAppointmentTypes())
      .then((list) => setTypes(Array.isArray(list) ? list : null))
      .catch(() => setTypes(null));
  }, []);
```

3. Avisos do gate, logo depois da lista de erros do gate:

```tsx
          {!parseError && (gate?.warnings ?? []).length > 0 && (
            <ul style={{ color: "var(--warn, #a60)", margin: 0, paddingLeft: 18 }}>
              {(gate?.warnings ?? []).map((w, i) => <li key={i}>{`aviso: ${w}`}</li>)}
            </ul>
          )}
```

4. Painel, logo depois do `<OfferPanel … />`:

```tsx
        <h2 style={{ fontSize: 16, margin: "16px 0 8px" }}>Agendamento</h2>
        <SchedulingPanel
          key={offerKey}
          definition={current.ok ? current.value : null}
          types={types}
          onChange={(next) => setText(JSON.stringify(next, null, 2))}
        />
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/lib/schedulingRules.test.ts src/modules/protocolEditor src/modules/ProtocolEditor.test.tsx && npx tsc --noEmit`
Expected: PASS. Os testes antigos do editor continuam verdes (o painel novo não escreve nada ao abrir).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod17
/opt/homebrew/bin/git add src/lib/schedulingRules.ts src/lib/schedulingRules.test.ts \
  src/modules/protocolEditor/SchedulingPanel.tsx src/modules/protocolEditor/SchedulingPanel.test.tsx \
  src/modules/ProtocolEditor.tsx src/modules/ProtocolEditor.test.tsx
/opt/homebrew/bin/git commit -m "feat: edit protocol scheduling rules with the condition builder

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 7: Fila da recepção com prazo, prioridade e marcas; detalhe do pedido

**Files:**
- Modify: `src/lib/api.ts` (`RequestRow`, linhas 567–571; `RequestDetail` e `getRequest` novos, logo depois de `listUnitRequests`)
- Modify: `src/lib/scheduling.ts` (acrescenta `requestKindLabel`), `src/lib/scheduling.test.ts`
- Modify: `src/test/schedulingFixtures.ts` (acrescenta `requestRow`)
- Create: `src/modules/attendance/RequestDetailPanel.tsx`
- Modify: `src/modules/attendance/Requests.tsx`, `src/modules/attendance/Requests.test.tsx`

**Interfaces:**
- Consumes: `SchedulingPriority`, `PreferredPeriod`, `RescheduleReasonCode` (Task 1); `requestMarks`, `fmtDueOn`, `PRIORITY_LABEL`, `PERIOD_LABEL`, `REASON_CODE_LABEL` (Task 2).
- Produces:
  - `RequestRow` na forma do contrato §4.1 e §9: `{ id; kind: "return" | "referral" | "triage"; origin: "attendance" | "triage"; origin_unit_name: string | null; created_at; cpf_masked; note: string | null; reopened_reason: "expired" | "no_show" | null; appointment_type_key: string; appointment_type_name: string; priority: SchedulingPriority; triage_priority: number | null; due_on: string; overdue: boolean; reschedule_requested: boolean; reschedule_reason_code: RescheduleReasonCode | null; preferred_period: PreferredPeriod | null; reschedule_count: number; needs_reschedule: boolean }`;
  - `RequestDetail extends RequestRow { reschedule_note: string | null }`, `getRequest(id: string): Promise<RequestDetail>`;
  - `requestKindLabel(row: Pick<RequestRow, "kind" | "origin_unit_name">): string`;
  - `requestRow(over?: Partial<RequestRow>): RequestRow` (fixture);
  - `RequestDetailPanel({ requestId, onClose })`.

- [ ] **Step 1: Write the failing tests**

Em `src/test/schedulingFixtures.ts` (acrescente `RequestRow` ao import de tipos):

```ts
export function requestRow(over: Partial<RequestRow> = {}): RequestRow {
  return {
    id: "r1", kind: "return", origin: "attendance", origin_unit_name: "UBS Centro", created_at: "2026-09-24T10:00:00Z",
    cpf_masked: "***.982.247-**", note: "controle de pressão", reopened_reason: null,
    appointment_type_key: "consulta_medica", appointment_type_name: "Consulta médica", priority: "routine",
    triage_priority: 2, due_on: "2026-10-20", overdue: false, reschedule_requested: false, reschedule_reason_code: null,
    preferred_period: null, reschedule_count: 0, needs_reschedule: false, ...over
  };
}
```

Em `src/lib/scheduling.test.ts`:

```ts
describe("rótulo do pedido", () => {
  it("retorno, encaminhamento com e sem unidade de origem, triagem", () => {
    expect(requestKindLabel({ kind: "return", origin_unit_name: "UBS Centro" })).toBe("Retorno");
    expect(requestKindLabel({ kind: "referral", origin_unit_name: "UPA Norte" })).toBe("Encaminhado de UPA Norte");
    expect(requestKindLabel({ kind: "referral", origin_unit_name: null })).toBe("Encaminhamento");
    expect(requestKindLabel({ kind: "triage", origin_unit_name: null })).toBe("Triagem");
  });
});
```

Em `src/modules/attendance/Requests.test.tsx`:
- `vi.mock` ganha `getRequest: vi.fn()`, e o `beforeEach` passa a resetar `api.getRequest` também;
- `import { requestRow } from "../../test/schedulingFixtures";`;
- troque a constante `rows` por:

```ts
const rows: api.RequestRow[] = [
  requestRow({ id: "r1" }),
  requestRow({ id: "r2", kind: "referral", origin_unit_name: "UPA Norte", cpf_masked: "***.111.222-**", note: null,
    reopened_reason: "expired", priority: "priority", due_on: "2026-10-01", overdue: true }),
  requestRow({ id: "r3", kind: "triage", origin: "triage", origin_unit_name: null, cpf_masked: "***.333.444-**", note: null,
    reopened_reason: "no_show", reschedule_requested: true, reschedule_reason_code: "work", preferred_period: "morning",
    reschedule_count: 1, needs_reschedule: true })
];
```

- troque o primeiro teste ("lista os pedidos com tipo, data, CPF, prioridade, nota e a marca") por:

```tsx
  it("lista na ordem do api, com pedido, atendimento, prazo, prioridade, nota e marcas", async () => {
    mocked(api.listUnitRequests).mockResolvedValue(rows);
    renderRequests();
    const cpfs = (await screen.findAllByText(/\*\*\*\.\d{3}\.\d{3}-\*\*/)).map((n) => n.textContent);
    expect(cpfs).toEqual([ "***.982.247-**", "***.111.222-**", "***.333.444-**" ]);
    expect(screen.getByText("Retorno")).not.toBeNull();
    expect(screen.getByText("Encaminhado de UPA Norte")).not.toBeNull();
    expect(screen.getByText("Triagem")).not.toBeNull();
    expect(screen.getAllByText("Consulta médica")).toHaveLength(3);
    expect(screen.getByText("até 01/10")).not.toBeNull();
    expect(screen.getByText("prioritária")).not.toBeNull();
    expect(screen.getByText("controle de pressão")).not.toBeNull();
    expect(screen.getByText("atrasado")).not.toBeNull();
    expect(screen.getByText("sem confirmação")).not.toBeNull();
    expect(screen.getByText("pediu outro horário")).not.toBeNull();
    expect(screen.getByText("precisa remarcar")).not.toBeNull();
    expect(screen.getByText("faltou")).not.toBeNull();
  });

  it("nota do cidadão só no detalhe, com motivo, período e quantas vezes pediu", async () => {
    mocked(api.listUnitRequests).mockResolvedValue([ rows[2] ]);
    mocked(api.getRequest).mockResolvedValue({ ...rows[2], reschedule_note: "não consigo sair do trabalho de manhã" });
    renderRequests();
    await screen.findByText("***.333.444-**");
    expect(screen.queryByText("não consigo sair do trabalho de manhã")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Detalhes" }));
    await waitFor(() => expect(api.getRequest).toHaveBeenCalledWith("r3"));
    expect(await screen.findByText("não consigo sair do trabalho de manhã")).not.toBeNull();
    expect(screen.getByText("trabalho")).not.toBeNull();
    expect(screen.getByText("manhã")).not.toBeNull();
    expect(screen.getByText("1 vez")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Fechar detalhes" }));
    expect(screen.queryByText("não consigo sair do trabalho de manhã")).toBeNull();
  });
```

- nos testes restantes, troque `await screen.findByText("Retorno");` por `await screen.findByText("***.982.247-**");` (o tipo "Retorno" deixou de ser único na linha).

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/lib/scheduling.test.ts src/modules/attendance/Requests.test.tsx`
Expected: FAIL — `requestKindLabel` não existe; `tsc`/vitest acusam os campos novos de `RequestRow`; "Triagem", "até 01/10", "Detalhes" não aparecem.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`, troque `RequestRow` e acrescente o detalhe:

```ts
// Pedido de agendamento (contratos §4.1 e §9, módulo 17). `priority` agora é a
// prioridade do pedido (rotina/prioritária); a da triagem é `triage_priority`.
export interface RequestRow {
  id: string; kind: "return" | "referral" | "triage"; origin: "attendance" | "triage";
  origin_unit_name: string | null; created_at: string; cpf_masked: string; note: string | null;
  reopened_reason: "expired" | "no_show" | null;
  appointment_type_key: string; appointment_type_name: string; priority: SchedulingPriority; due_on: string;
  // O número de prioridade da triagem que a fila já mandava como `priority` (§9).
  triage_priority: number | null;
  overdue: boolean; reschedule_requested: boolean; reschedule_reason_code: RescheduleReasonCode | null;
  preferred_period: PreferredPeriod | null; reschedule_count: number; needs_reschedule: boolean;
}

// Só o detalhe traz a nota livre do cidadão (spec §8).
export interface RequestDetail extends RequestRow { reschedule_note: string | null }

export function getRequest(id: string): Promise<RequestDetail> {
  return jsonFetch(`${ATTENDANCE_BASE}/requests/${encodeURIComponent(id)}`);
}
```

Em `src/lib/scheduling.ts`:

```ts
export function requestKindLabel(row: { kind: "return" | "referral" | "triage"; origin_unit_name: string | null }): string {
  if (row.kind === "return") return "Retorno";
  if (row.kind === "triage") return "Triagem";
  return row.origin_unit_name ? `Encaminhado de ${row.origin_unit_name}` : "Encaminhamento";
}
```

```tsx
// src/modules/attendance/RequestDetailPanel.tsx
// Detalhe do pedido (contratos §4.1): o único lugar com a nota livre do
// cidadão. Lido só quando a recepção abre (spec §8).
import { useQuery } from "@tanstack/react-query";
import { getRequest } from "../../lib/api";
import { attendanceError } from "../../lib/attendance";
import { PERIOD_LABEL, PRIORITY_LABEL, REASON_CODE_LABEL, fmtDueOn, requestKindLabel } from "../../lib/scheduling";
import { KeyValue } from "../../components/KeyValue";
import { secondaryButtonStyle } from "../../components/formStyles";

export function RequestDetailPanel({ requestId, onClose }: { requestId: string; onClose(): void }) {
  const query = useQuery({ queryKey: [ "requestDetail", requestId ], queryFn: () => getRequest(requestId) });
  const d = query.data;
  return (
    <section aria-label="Detalhe do pedido" style={{ display: "flex", flexDirection: "column", gap: 10, padding: 16,
      border: "1px solid var(--rule)", borderRadius: 8 }}>
      <strong>Detalhe do pedido</strong>
      {query.isError && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{attendanceError(query.error)}</p>}
      {query.isPending && <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando…</p>}
      {d && (
        <div style={{ display: "flex", gap: 24, flexWrap: "wrap" }}>
          <KeyValue k="Pedido" v={requestKindLabel(d)} />
          <KeyValue k="Atendimento" v={d.appointment_type_name} />
          <KeyValue k="Prazo" v={fmtDueOn(d.due_on)} />
          <KeyValue k="Prioridade" v={PRIORITY_LABEL[d.priority]} />
          <KeyValue k="Pediu outro horário" v={d.reschedule_count === 1 ? "1 vez" : `${d.reschedule_count} vezes`} />
          {d.reschedule_reason_code && <KeyValue k="Motivo" v={REASON_CODE_LABEL[d.reschedule_reason_code]} />}
          {d.preferred_period && <KeyValue k="Período preferido" v={PERIOD_LABEL[d.preferred_period]} />}
          <KeyValue k="Nota do cidadão" v={d.reschedule_note ?? "—"} />
        </div>
      )}
      <div><button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar detalhes</button></div>
    </section>
  );
}
```

Em `src/modules/attendance/Requests.tsx`:
- remova `kindLabel` e `reopenedLabel`; importe `requestKindLabel`, `requestMarks`, `fmtDueOn`, `PRIORITY_LABEL` de `../../lib/scheduling` e `RequestDetailPanel` de `./RequestDetailPanel`;
- estado novo `const [ detail, setDetail ] = useState<RequestRow | null>(null);`;
- a fila mostra as linhas **na ordem do api** (atrasados, prazo, prioridade); a tela não reordena e não mostra `triage_priority` (a ordem já o considera);
- troque `cols` por:

```tsx
              cols={[
                { label: "Pedido", w: "1.5fr", render: (r) => requestKindLabel(r) },
                { label: "Atendimento", w: "1.5fr", render: (r) => r.appointment_type_name },
                { label: "CPF", w: "1.5fr", render: (r) => r.cpf_masked },
                { label: "Prazo", w: "1fr", render: (r) => fmtDueOn(r.due_on) },
                { label: "Prioridade", w: "1fr", render: (r) =>
                  <Tag tone={r.priority === "priority" ? "warn" : "neutral"}>{PRIORITY_LABEL[r.priority]}</Tag> },
                { label: "Nota", w: "2fr", render: (r) => r.note ?? "—" },
                { label: "Marcas", w: "2fr", render: (r) => {
                  const marks = requestMarks(r);
                  if (marks.length === 0) return "—";
                  return (
                    <span style={{ display: "flex", gap: 4, flexWrap: "wrap" }}>
                      {marks.map((m) => <Tag key={m.label} tone={m.tone}>{m.label}</Tag>)}
                    </span>
                  );
                } },
                {
                  label: "", w: "auto", align: "right", render: (r) => (
                    <div style={{ display: "flex", gap: 8, justifyContent: "flex-end" }}>
                      <button type="button" style={secondaryButtonStyle} onClick={() => setDetail(r)}>Detalhes</button>
                      <button type="button" style={secondaryButtonStyle} onClick={() => setScheduling(r)}>
                        Marcar horário
                      </button>
                      <button type="button" style={secondaryButtonStyle} onClick={() => setDismissing(r)}>
                        Encerrar pedido
                      </button>
                    </div>
                  )
                }
              ]}
```

- e, antes de `{scheduling && (`:

```tsx
        {detail && <RequestDetailPanel key={detail.id} requestId={detail.id} onClose={() => setDetail(null)} />}
```

Confira se o `Attendance.test.tsx` monta algum `RequestRow` literal; se montar, troque por `requestRow(...)`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/lib/scheduling.test.ts src/modules/attendance src/modules/Attendance.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod17
/opt/homebrew/bin/git add src/lib/api.ts src/lib/scheduling.ts src/lib/scheduling.test.ts src/test/schedulingFixtures.ts \
  src/modules/attendance/RequestDetailPanel.tsx src/modules/attendance/Requests.tsx src/modules/attendance/Requests.test.tsx
/opt/homebrew/bin/git commit -m "feat: show due date, priority and marks in the request queue

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Marcar a partir do pedido (vagas por dia e profissional, marcação livre só em `legacy_days`)

**Files:**
- Create: `src/modules/attendance/BookPanel.tsx`
- Modify: `src/modules/attendance/Requests.tsx` (troca `SchedulePanel` por `BookPanel`; `invalidateAll` recarrega também as vagas)
- Modify: `src/lib/api.ts` (remove `scheduleRequest`; `ScheduledAppointment` fica)
- Modify: `src/modules/attendance/Requests.test.tsx`, `src/modules/Attendance.test.tsx` (mocks)
- Test: `src/modules/attendance/BookPanel.test.tsx`

**Interfaces:**
- Consumes: `getUnitAvailability`, `bookAppointment`, `AvailabilitySlot`, `BookingInput` (Task 1); `RequestRow` (Task 7); `AVAILABILITY_DAYS`, `addDaysIso`, `daysBetween`, `dayLabel`, `fmtDueOn`, `slotsByDay`, `confirmationWarning` (Task 2); `attendanceError` com as mensagens da Task 2; `parseCityLocal`, `cityIsoDate`, `fmtHourMinute`.
- Produces: `availabilityKey(unitId: string): readonly ["unitAvailability", string]`; `BookPanel({ row, unit, onCancel, onDone })`. A Task 9 acrescenta o botão "Encaixe" dentro dele.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/attendance/BookPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, getUnitAvailability: vi.fn(), bookAppointment: vi.fn(), getUnitAgenda: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { BookPanel } from "./BookPanel";
import { NOW, requestRow, slot } from "../../test/schedulingFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };
const APPOINTMENT = { id: "a1", scheduled_at: "x", status: "scheduled", confirmation_deadline_at: null };

const AVAILABILITY: api.Availability = {
  slots: [
    slot(),
    slot({ starts_at: "2026-10-06T09:20:00-03:00", ends_at: "2026-10-06T09:40:00-03:00" }),
    slot({ professional_id: "p2", professional_name: "Marta Lima", shift_id: "s2" })
  ],
  legacy_days: [ "2026-10-07" ]
};

function renderIt(row = requestRow()) {
  const onDone = vi.fn();
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<BookPanel row={row} unit={unit} onCancel={vi.fn()} onDone={onDone} />, { wrapper });
  return { onDone };
}
const pickDay = async (value: string) => {
  const select = await screen.findByLabelText("Dia");
  fireEvent.change(select, { target: { value } });
};
const confirm = () => screen.getByRole("button", { name: "Confirmar horário" }) as HTMLButtonElement;

describe("BookPanel", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW));
    mocked(api.getUnitAvailability).mockResolvedValue(AVAILABILITY);
  });

  it("pede 14 dias do tipo do pedido, a partir de hoje na cidade", async () => {
    renderIt();
    await waitFor(() => expect(api.getUnitAvailability).toHaveBeenCalledWith("u1", "consulta_medica", "2026-10-05", "2026-10-18"));
    expect(screen.getByText("Marcar horário — Consulta médica · prazo até 20/10")).not.toBeNull();
    const day = await screen.findByLabelText("Dia");
    expect(within(day).getByRole("option", { name: "ter 06/10 · 3 vagas" })).not.toBeNull();
    expect(within(day).getByRole("option", { name: "qua 07/10 · sem turno (marcação livre)" })).not.toBeNull();
    expect(within(day).getByRole("option", { name: "qui 08/10 · sem vaga" })).not.toBeNull();
  });

  it("vaga: escolher dia e profissional, avisar a confirmação e marcar", async () => {
    const { onDone } = renderIt();
    mocked(api.bookAppointment).mockResolvedValue(APPOINTMENT);
    await pickDay("2026-10-06");
    fireEvent.change(screen.getByLabelText("Profissional"), { target: { value: "p1" } });
    expect(screen.queryByRole("radio", { name: "09:00 · Marta Lima" })).toBeNull();
    expect(confirm().disabled).toBe(true);
    fireEvent.click(screen.getByRole("radio", { name: "09:20 · Helena Duarte" }));
    expect(screen.getByText("O horário nasce confirmado")).not.toBeNull();
    fireEvent.click(confirm());
    await waitFor(() => expect(api.bookAppointment).toHaveBeenCalledWith("r1", "u1", {
      kind: "slot", professional_id: "p1", starts_at: "2026-10-06T09:20:00-03:00", appointment_type_key: "consulta_medica"
    }));
    await waitFor(() => expect(onDone).toHaveBeenCalled());
  });

  it("slot_taken recarrega as vagas, avisa e não marca outra", async () => {
    const { onDone } = renderIt();
    mocked(api.bookAppointment).mockRejectedValueOnce(new ApiError(409, { error: "slot_taken" }, "x"));
    await pickDay("2026-10-06");
    fireEvent.click(screen.getByRole("radio", { name: "09:00 · Helena Duarte" }));
    fireEvent.click(confirm());
    expect(await screen.findByText("Essa vaga acabou de ser ocupada. As vagas foram recarregadas — escolha outra.")).not.toBeNull();
    await waitFor(() => expect(api.getUnitAvailability).toHaveBeenCalledTimes(2));
    expect(api.bookAppointment).toHaveBeenCalledTimes(1);
    expect(onDone).not.toHaveBeenCalled();
    expect(screen.getAllByRole("radio").some((r) => (r as HTMLInputElement).checked)).toBe(false);
    expect(confirm().disabled).toBe(true);
  });

  it("slot_unavailable também recarrega, com a frase da recusa", async () => {
    renderIt();
    mocked(api.bookAppointment).mockRejectedValueOnce(new ApiError(409, { error: "slot_unavailable" }, "x"));
    await pickDay("2026-10-06");
    fireEvent.click(screen.getByRole("radio", { name: "09:00 · Helena Duarte" }));
    fireEvent.click(confirm());
    expect(await screen.findByText("essa vaga não está mais disponível — as vagas foram recarregadas")).not.toBeNull();
    await waitFor(() => expect(api.getUnitAvailability).toHaveBeenCalledTimes(2));
  });

  it("citizen_busy fica como erro, sem recarregar", async () => {
    renderIt();
    mocked(api.bookAppointment).mockRejectedValueOnce(new ApiError(409, { error: "citizen_busy" }, "x"));
    await pickDay("2026-10-06");
    fireEvent.click(screen.getByRole("radio", { name: "09:00 · Helena Duarte" }));
    fireEvent.click(confirm());
    expect(await screen.findByText("o cidadão já tem outro horário nesse período")).not.toBeNull();
    expect(api.getUnitAvailability).toHaveBeenCalledTimes(1);
  });

  it("dia sem turno: marcação livre com hora da cidade", async () => {
    renderIt();
    mocked(api.bookAppointment).mockResolvedValue(APPOINTMENT);
    await pickDay("2026-10-07");
    expect(screen.queryByRole("radio")).toBeNull();
    fireEvent.change(screen.getByLabelText("Horário (marcação livre)"), { target: { value: "14:30" } });
    expect(screen.getByText("O cidadão precisa confirmar até 06/10 14:30")).not.toBeNull(); // 52h30 à frente
    fireEvent.click(confirm());
    await waitFor(() => expect(api.bookAppointment).toHaveBeenCalledWith("r1", "u1",
      { kind: "legacy", scheduled_at: "2026-10-07T17:30:00.000Z" }));
  });

  it("marcação livre ocupada: avisa quantos e 'Marcar mesmo assim' manda allow_overlap", async () => {
    renderIt();
    mocked(api.bookAppointment)
      .mockRejectedValueOnce(new ApiError(409, { error: "slot_taken", taken: 2 }, "x"))
      .mockResolvedValueOnce(APPOINTMENT);
    await pickDay("2026-10-07");
    fireEvent.change(screen.getByLabelText("Horário (marcação livre)"), { target: { value: "14:30" } });
    fireEvent.click(confirm());
    expect(await screen.findByText("Já há 2 horários marcados na UBS Centro nesse horário. Marcar mesmo assim deixa os dois no mesmo horário."))
      .not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Marcar mesmo assim" }));
    await waitFor(() => expect(api.bookAppointment).toHaveBeenLastCalledWith("r1", "u1",
      { kind: "legacy", scheduled_at: "2026-10-07T17:30:00.000Z", allow_overlap: true }));
  });

  it("use_slots num dia de marcação livre recarrega e some a marcação livre", async () => {
    renderIt();
    mocked(api.getUnitAvailability)
      .mockResolvedValueOnce(AVAILABILITY)
      .mockResolvedValueOnce({ slots: [ ...AVAILABILITY.slots, slot({ starts_at: "2026-10-07T08:00:00-03:00", ends_at: "2026-10-07T08:20:00-03:00" }) ],
        legacy_days: [] });
    mocked(api.bookAppointment).mockRejectedValueOnce(new ApiError(409, { error: "use_slots" }, "x"));
    await pickDay("2026-10-07");
    fireEvent.change(screen.getByLabelText("Horário (marcação livre)"), { target: { value: "14:30" } });
    fireEvent.click(confirm());
    expect(await screen.findByText("a unidade tem turno neste dia — marque numa vaga ou faça um encaixe")).not.toBeNull();
    expect(await screen.findByRole("radio", { name: "08:00 · Helena Duarte" })).not.toBeNull();
    expect(screen.queryByLabelText("Horário (marcação livre)")).toBeNull();
  });

  it("14 dias → pede o período seguinte; ← volta e nunca passa de hoje", async () => {
    renderIt();
    await screen.findByLabelText("Dia");
    expect((screen.getByRole("button", { name: "← 14 dias" }) as HTMLButtonElement).disabled).toBe(true);
    fireEvent.click(screen.getByRole("button", { name: "14 dias →" }));
    await waitFor(() => expect(api.getUnitAvailability).toHaveBeenLastCalledWith("u1", "consulta_medica", "2026-10-19", "2026-11-01"));
    fireEvent.click(screen.getByRole("button", { name: "← 14 dias" }));
    await waitFor(() => expect(api.getUnitAvailability).toHaveBeenLastCalledWith("u1", "consulta_medica", "2026-10-05", "2026-10-18"));
  });

  it("request_not_open fecha o painel (outra recepção já marcou)", async () => {
    const { onDone } = renderIt();
    mocked(api.bookAppointment).mockRejectedValueOnce(new ApiError(409, { error: "request_not_open" }, "x"));
    await pickDay("2026-10-06");
    fireEvent.click(screen.getByRole("radio", { name: "09:00 · Helena Duarte" }));
    fireEvent.click(confirm());
    await waitFor(() => expect(onDone).toHaveBeenCalled());
  });
});
```

Em `src/modules/attendance/Requests.test.tsx`:
- no `vi.mock` e no `beforeEach`, troque `scheduleRequest` por `bookAppointment` e acrescente `getUnitAvailability`;
- apague os quatro testes de "Marcar horário" de hoje (48h, prazo de confirmação, ocupado + "Marcar mesmo assim", trocar o horário): o comportamento passou para `BookPanel.test.tsx`;
- acrescente:

```tsx
  it("Marcar horário abre as vagas do tipo do pedido; marcar recarrega a fila", async () => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date("2026-10-05T10:00:00-03:00"));
    mocked(api.listUnitRequests).mockResolvedValue([ rows[0] ]);
    mocked(api.getUnitAvailability).mockResolvedValue({ slots: [ slot() ], legacy_days: [] });
    mocked(api.bookAppointment).mockResolvedValue({ id: "a1", scheduled_at: "x", status: "scheduled", confirmation_deadline_at: null });
    renderRequests();
    await screen.findByText("***.982.247-**");
    fireEvent.click(screen.getByRole("button", { name: "Marcar horário" }));
    fireEvent.change(await screen.findByLabelText("Dia"), { target: { value: "2026-10-06" } });
    fireEvent.click(screen.getByRole("radio", { name: "09:00 · Helena Duarte" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar horário" }));
    await waitFor(() => expect(api.listUnitRequests).toHaveBeenCalledTimes(2));
  });
```

  com `slot` no import de `../../test/schedulingFixtures`.

Em `src/modules/Attendance.test.tsx`: no `vi.mock` e na lista do `beforeEach`, troque `scheduleRequest` por `bookAppointment` e acrescente `getUnitAvailability`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/modules/attendance/BookPanel.test.tsx src/modules/attendance/Requests.test.tsx`
Expected: FAIL — `./BookPanel` não existe; "Dia" não aparece no painel de hoje.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/attendance/BookPanel.tsx
// Marcar a partir do pedido (módulo 17; spec §4, contratos §4.2–§4.3). A
// recepção escolhe dia, profissional e vaga calculada pelo api. Marcação livre
// (sem profissional) só em dia sem nenhum turno na unidade (`legacy_days`).
// Vaga que some entre a leitura e o clique (slot_taken, slot_unavailable,
// use_slots) recarrega as vagas e deixa a pessoa escolher de novo: a tela
// nunca marca outra vaga sozinha.
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { ApiError, bookAppointment, getUnitAvailability, type AvailabilitySlot, type BookingInput, type HealthUnit, type RequestRow } from "../../lib/api";
import { attendanceError } from "../../lib/attendance";
import { cityIsoDate, fmtHourMinute, parseCityLocal } from "../../lib/format";
import { AVAILABILITY_DAYS, addDaysIso, confirmationWarning, dayLabel, daysBetween, fmtDueOn, slotsByDay } from "../../lib/scheduling";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";

interface Props { row: RequestRow; unit: HealthUnit; onCancel(): void; onDone(): void }

export const availabilityKey = (unitId: string) => [ "unitAvailability", unitId ] as const;

function errorCode(err: unknown): string | undefined {
  return err instanceof ApiError ? (err.body as { error?: string } | undefined)?.error : undefined;
}

function takenCount(err: unknown): number {
  const body = (err instanceof ApiError ? err.body : null) as { taken?: number } | null;
  return typeof body?.taken === "number" && body.taken > 0 ? body.taken : 1;
}

const sameSlot = (a: AvailabilitySlot | null, b: AvailabilitySlot) =>
  !!a && a.professional_id === b.professional_id && a.starts_at === b.starts_at;

export function BookPanel({ row, unit, onCancel, onDone }: Props) {
  const queryClient = useQueryClient();
  const today = cityIsoDate();
  const [ from, setFrom ] = useState(today);
  const to = addDaysIso(from, AVAILABILITY_DAYS - 1);
  const query = useQuery({
    queryKey: [ ...availabilityKey(unit.id), row.appointment_type_key, from ],
    queryFn: () => getUnitAvailability(unit.id, row.appointment_type_key, from, to)
  });
  const [ day, setDay ] = useState<string | null>(null);
  const [ professionalId, setProfessionalId ] = useState("");
  const [ picked, setPicked ] = useState<AvailabilitySlot | null>(null);
  const [ legacyTime, setLegacyTime ] = useState("");
  const [ taken, setTaken ] = useState<number | null>(null);
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);

  const byDay = slotsByDay(query.data?.slots ?? []);
  const legacyDays = new Set(query.data?.legacy_days ?? []);
  const daySlots = day ? (byDay.get(day) ?? []) : [];
  const professionals = [ ...new Map(daySlots.map((s) => [ s.professional_id, s.professional_name ])).entries() ]
    .sort((a, b) => a[1].localeCompare(b[1], "pt-BR"));
  const visible = professionalId ? daySlots.filter((s) => s.professional_id === professionalId) : daySlots;
  const isLegacy = day !== null && legacyDays.has(day);
  const legacyAt = isLegacy && /^\d{2}:\d{2}$/.test(legacyTime) ? parseCityLocal(`${day}T${legacyTime}`) : null;
  const start = picked ? new Date(picked.starts_at) : legacyAt;
  const warning = start ? confirmationWarning(start, new Date()) : null;

  function dayOption(d: string): string {
    if (legacyDays.has(d)) return `${dayLabel(d)} · sem turno (marcação livre)`;
    const n = byDay.get(d)?.length ?? 0;
    return `${dayLabel(d)} · ${n === 0 ? "sem vaga" : n === 1 ? "1 vaga" : `${n} vagas`}`;
  }

  function changePeriod(next: string) {
    setFrom(next < today ? today : next);
    setDay(null); setPicked(null); setProfessionalId(""); setLegacyTime(""); setTaken(null); setNotice(null);
  }

  function reloadSlots(message: string) {
    setPicked(null); setNotice(message);
    void queryClient.invalidateQueries({ queryKey: availabilityKey(unit.id) });
  }

  async function submit(allowOverlap = false) {
    if (busy) return;
    let input: BookingInput | null = null;
    if (picked) {
      input = { kind: "slot", professional_id: picked.professional_id, starts_at: picked.starts_at, appointment_type_key: row.appointment_type_key };
    } else if (legacyAt) {
      input = allowOverlap
        ? { kind: "legacy", scheduled_at: legacyAt.toISOString(), allow_overlap: true }
        : { kind: "legacy", scheduled_at: legacyAt.toISOString() };
    }
    if (!input) return;
    setBusy(true); setError(null); setNotice(null);
    try {
      await bookAppointment(row.id, unit.id, input);
      onDone();
    } catch (err) {
      const code = errorCode(err);
      if (code === "request_not_open") { onDone(); return; }
      if (code === "slot_taken" && input.kind === "slot") {
        reloadSlots("Essa vaga acabou de ser ocupada. As vagas foram recarregadas — escolha outra.");
        return;
      }
      if (code === "slot_taken") { setTaken(takenCount(err)); return; }
      if (code === "slot_unavailable") { reloadSlots(attendanceError(err)); return; }
      if (code === "use_slots") { setLegacyTime(""); reloadSlots(attendanceError(err)); return; }
      setError(attendanceError(err));
    } finally {
      setBusy(false);
    }
  }

  const ready = !!picked || !!legacyAt;

  return (
    <section aria-label="Marcar horário" style={panel}>
      <strong>{`Marcar horário — ${row.appointment_type_name} · prazo ${fmtDueOn(row.due_on)}`}</strong>
      {error && <p role="alert" style={alert}>{error}</p>}
      {notice && <p role="status" style={{ ...alert, color: "var(--warn)", fontWeight: 600 }}>{notice}</p>}
      <div style={{ display: "flex", gap: 8, alignItems: "center" }}>
        <button type="button" disabled={from <= today} style={from <= today ? disabledButtonStyle : secondaryButtonStyle}
          onClick={() => changePeriod(addDaysIso(from, -AVAILABILITY_DAYS))}>← 14 dias</button>
        <span style={{ fontSize: 12.5 }}>{`${dayLabel(from)} a ${dayLabel(to)}`}</span>
        <button type="button" style={secondaryButtonStyle} onClick={() => changePeriod(addDaysIso(from, AVAILABILITY_DAYS))}>
          14 dias →
        </button>
      </div>
      {query.isError ? <p role="alert" style={alert}>{attendanceError(query.error)}</p>
        : query.isPending ? <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando vagas…</p>
        : (
          <label style={{ ...label, maxWidth: 320 }}>Dia
            <select value={day ?? ""} style={inputStyle} onChange={(e) => {
              setDay(e.target.value || null); setPicked(null); setProfessionalId(""); setLegacyTime(""); setTaken(null);
            }}>
              <option value="">escolha…</option>
              {daysBetween(from, to).map((d) => <option key={d} value={d}>{dayOption(d)}</option>)}
            </select>
          </label>
        )}

      {day && !isLegacy && (
        <>
          {professionals.length > 1 && (
            <label style={{ ...label, maxWidth: 320 }}>Profissional
              <select value={professionalId} style={inputStyle}
                onChange={(e) => { setProfessionalId(e.target.value); setPicked(null); }}>
                <option value="">todos</option>
                {professionals.map(([ id, name ]) => <option key={id} value={id}>{name}</option>)}
              </select>
            </label>
          )}
          {visible.length === 0 ? <p style={hint}>nenhuma vaga neste dia</p> : (
            <div role="radiogroup" aria-label="Vagas" style={{ display: "flex", flexDirection: "column", gap: 4 }}>
              {visible.map((s) => (
                <label key={`${s.professional_id}-${s.starts_at}`} style={{ display: "flex", gap: 6, fontSize: 13 }}>
                  <input type="radio" name="slot" checked={sameSlot(picked, s)}
                    onChange={() => { setPicked(s); setNotice(null); setError(null); }} />
                  {`${fmtHourMinute(s.starts_at)} · ${s.professional_name}`}
                </label>
              ))}
            </div>
          )}
        </>
      )}

      {day && isLegacy && (
        <>
          <p style={hint}>Sem turno neste dia: marcação livre, sem profissional.</p>
          <label style={{ ...label, maxWidth: 240 }}>Horário (marcação livre)
            <input type="time" value={legacyTime} style={inputStyle}
              onChange={(e) => { setLegacyTime(e.target.value); setTaken(null); }} />
          </label>
          {taken !== null && (
            <p role="alert" style={{ ...alert, fontWeight: 600, color: "var(--warn)" }}>
              {taken === 1
                ? `Já há 1 horário marcado na ${unit.name} nesse horário.`
                : `Já há ${taken} horários marcados na ${unit.name} nesse horário.`} Marcar mesmo assim deixa os dois no mesmo horário.
            </p>
          )}
        </>
      )}

      {warning && <p role="status" style={{ margin: 0, fontSize: 12.5, fontWeight: 600 }}>{warning}</p>}
      <div style={{ display: "flex", gap: 8 }}>
        {taken === null ? (
          <button type="button" disabled={!ready || busy} onClick={() => void submit()}
            style={!ready || busy ? disabledButtonStyle : buttonStyle}>Confirmar horário</button>
        ) : (
          <button type="button" disabled={busy} onClick={() => void submit(true)}
            style={busy ? disabledButtonStyle : buttonStyle}>Marcar mesmo assim</button>
        )}
        <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Cancelar</button>
      </div>
    </section>
  );
}

const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const hint: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 };
```

Em `src/modules/attendance/Requests.tsx`:
- remova `SchedulePanel`, `slotTaken`, `fmtShort` e os imports que ficarem sem uso (`scheduleRequest`, `cityDateFormat`, `fmtHourMinute`, `parseCityLocal`); importe `{ BookPanel, availabilityKey } from "./BookPanel"`;
- `invalidateAll` passa a recarregar também as vagas:

```tsx
  function invalidateAll() {
    invalidateRequests();
    void queryClient.invalidateQueries({ queryKey: [ "unitAgenda", unit.id ] });
    void queryClient.invalidateQueries({ queryKey: availabilityKey(unit.id) });
  }
```

- troque o bloco `{scheduling && ( <SchedulePanel … /> )}` por:

```tsx
        {scheduling && (
          <BookPanel
            key={`s-${scheduling.id}`}
            row={scheduling}
            unit={unit}
            onCancel={() => setScheduling(null)}
            onDone={() => { setScheduling(null); invalidateAll(); }}
          />
        )}
```

Em `src/lib/api.ts`, apague `scheduleRequest` (o comentário `allowOverlap` e a função); `ScheduledAppointment` continua, usado por `bookAppointment`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/modules/attendance src/modules/Attendance.test.tsx && npx tsc --noEmit && grep -rn "scheduleRequest" src`
Expected: PASS; o `grep` não encontra nada.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod17
/opt/homebrew/bin/git add src/modules/attendance/BookPanel.tsx src/modules/attendance/BookPanel.test.tsx \
  src/modules/attendance/Requests.tsx src/modules/attendance/Requests.test.tsx src/modules/Attendance.test.tsx src/lib/api.ts
/opt/homebrew/bin/git commit -m "feat: book requests into computed slots with legacy booking on days without shifts

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 9: Encaixe com justificativa e contador do limite

**Files:**
- Create: `src/modules/attendance/FitInPanel.tsx`
- Modify: `src/modules/attendance/BookPanel.tsx` (botão "Encaixe" no dia com turno), `src/modules/attendance/BookPanel.test.tsx`
- Modify: `src/test/schedulingFixtures.ts` (acrescenta `agendaShift` e `unitAgenda`)
- Test: `src/modules/attendance/FitInPanel.test.tsx`

**Interfaces:**
- Consumes: `getUnitAgenda`, `bookAppointment`, `AgendaShift`, `UnitAgenda` (Task 1); `RequestRow` (Task 7); `FIT_IN_REASON_MIN`, `confirmationWarning` (Task 2); `cityLocalIso`, `fmtHourMinute`; `FrozenTextNotice`.
- Produces: `FitInPanel({ row, unit, date, onBack, onDone })`; fixtures `agendaShift(over?: Partial<AgendaShift>): AgendaShift`, `unitAgenda(over?: Partial<UnitAgenda>): UnitAgenda`. A chave da consulta é `[ "unitAgenda", unit.id, date ]`, a mesma da agenda da unidade (Task 10): recarregar uma recarrega a outra.

- [ ] **Step 1: Write the failing tests**

Em `src/test/schedulingFixtures.ts` (acrescente `AgendaShift` e `UnitAgenda` ao import de tipos):

```ts
export function agendaShift(over: Partial<AgendaShift> = {}): AgendaShift {
  return { shift_id: "s1", starts_at: "2026-10-06T07:00:00-03:00", ends_at: "2026-10-06T12:00:00-03:00",
    blocks: MORNING.blocks, fit_in_count: 1, fit_in_limit: 2, cancelled_at: null, ...over };
}

export function unitAgenda(over: Partial<UnitAgenda> = {}): UnitAgenda {
  return {
    date: "2026-10-06",
    professionals: [
      { id: "p1", name: "Helena Duarte", shifts: [ agendaShift() ], appointments: [ appointmentView() ] },
      { id: "p2", name: "Marta Lima", shifts: [ agendaShift({ shift_id: "s2", cancelled_at: "2026-10-05T08:00:00-03:00" }) ], appointments: [] }
    ],
    unassigned: [],
    ...over
  };
}
```

```tsx
// src/modules/attendance/FitInPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, getUnitAgenda: vi.fn(), bookAppointment: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { FitInPanel } from "./FitInPanel";
import { agendaShift, NOW, requestRow, unitAgenda } from "../../test/schedulingFixtures";
import { expectFrozenNotice } from "../../test/frozenNotice";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };

function renderIt() {
  const onDone = vi.fn();
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<FitInPanel row={requestRow()} unit={unit} date="2026-10-06" onBack={vi.fn()} onDone={onDone} />, { wrapper });
  return { onDone };
}
const submit = () => screen.getByRole("button", { name: "Confirmar encaixe" }) as HTMLButtonElement;
async function fill(reason = "gestante com dor abdominal") {
  const select = await screen.findByLabelText("Turno");
  await within(select).findByRole("option", { name: "Helena Duarte · 07:00–12:00 · encaixes 1 de 2" });
  fireEvent.change(select, { target: { value: "s1" } });
  fireEvent.change(screen.getByLabelText("Início do encaixe"), { target: { value: "10:10" } });
  fireEvent.change(screen.getByLabelText("Justificativa do encaixe"), { target: { value: reason } });
}

describe("FitInPanel", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW));
    mocked(api.getUnitAgenda).mockResolvedValue(unitAgenda());
  });

  it("lista os turnos do dia com o contador; turno cancelado não entra", async () => {
    renderIt();
    const select = await screen.findByLabelText("Turno");
    await within(select).findByRole("option", { name: "Helena Duarte · 07:00–12:00 · encaixes 1 de 2" });
    expect(within(select).queryByRole("option", { name: /Marta Lima/ })).toBeNull();
    expect(api.getUnitAgenda).toHaveBeenCalledWith("u1", "2026-10-06");
  });

  it("justificativa curta trava; completa manda o encaixe no fuso da cidade", async () => {
    const { onDone } = renderIt();
    mocked(api.bookAppointment).mockResolvedValue({ id: "a9", scheduled_at: "x", status: "confirmed", confirmation_deadline_at: null });
    await fill("dor");
    expect(screen.getByText("Encaixes neste turno: 1 de 2")).not.toBeNull();
    expect(submit().disabled).toBe(true);
    expectFrozenNotice(screen.getByLabelText("Justificativa do encaixe"));
    fireEvent.change(screen.getByLabelText("Justificativa do encaixe"), { target: { value: "gestante com dor abdominal" } });
    expect(submit().disabled).toBe(false);
    fireEvent.click(submit());
    await waitFor(() => expect(api.bookAppointment).toHaveBeenCalledWith("r1", "u1", {
      kind: "fit_in", professional_id: "p1", shift_id: "s1", starts_at: "2026-10-06T10:10:00-03:00",
      appointment_type_key: "consulta_medica", reason: "gestante com dor abdominal"
    }));
    await waitFor(() => expect(onDone).toHaveBeenCalled());
  });

  it("início fora do turno trava antes da API", async () => {
    renderIt();
    await fill();
    fireEvent.change(screen.getByLabelText("Início do encaixe"), { target: { value: "13:00" } });
    expect(screen.getByText("o encaixe precisa começar dentro do turno (07:00–12:00)")).not.toBeNull();
    expect(submit().disabled).toBe(true);
  });

  it("turno no limite: aviso e botão travado", async () => {
    mocked(api.getUnitAgenda).mockResolvedValue(unitAgenda({ professionals: [
      { id: "p1", name: "Helena Duarte", shifts: [ agendaShift({ fit_in_count: 2 }) ], appointments: [] } ] }));
    renderIt();
    const select = await screen.findByLabelText("Turno");
    await within(select).findByRole("option", { name: "Helena Duarte · 07:00–12:00 · encaixes 2 de 2" });
    fireEvent.change(select, { target: { value: "s1" } });
    expect(screen.getByText("limite de encaixes atingido neste turno")).not.toBeNull();
    expect(submit().disabled).toBe(true);
  });

  it("fit_in_limit recarrega o contador e trava o botão", async () => {
    mocked(api.getUnitAgenda)
      .mockResolvedValueOnce(unitAgenda())
      .mockResolvedValue(unitAgenda({ professionals: [
        { id: "p1", name: "Helena Duarte", shifts: [ agendaShift({ fit_in_count: 2 }) ], appointments: [] } ] }));
    mocked(api.bookAppointment).mockRejectedValueOnce(new ApiError(409, { error: "fit_in_limit" }, "x"));
    const { onDone } = renderIt();
    await fill();
    fireEvent.click(submit());
    expect(await screen.findByText("o turno já chegou ao limite de encaixes")).not.toBeNull();
    expect(await screen.findByText("Encaixes neste turno: 2 de 2")).not.toBeNull();
    expect(submit().disabled).toBe(true);
    expect(onDone).not.toHaveBeenCalled();
  });

  it("type_not_served e outside_shift aparecem traduzidos", async () => {
    mocked(api.bookAppointment).mockRejectedValueOnce(new ApiError(422, { error: "type_not_served" }, "x"));
    renderIt();
    await fill();
    fireEvent.click(submit());
    expect(await screen.findByText("este profissional não atende este tipo de atendimento")).not.toBeNull();
  });
});
```

Em `src/modules/attendance/BookPanel.test.tsx`, acrescente (com `unitAgenda` no import das fixtures):

```tsx
  it("Encaixe aparece em dia com turno, não em dia de marcação livre, e abre o painel do dia", async () => {
    mocked(api.getUnitAgenda).mockResolvedValue(unitAgenda());
    renderIt();
    await pickDay("2026-10-07");
    expect(screen.queryByRole("button", { name: "Encaixe" })).toBeNull();
    await pickDay("2026-10-06");
    fireEvent.click(screen.getByRole("button", { name: "Encaixe" }));
    expect(await screen.findByLabelText("Justificativa do encaixe")).not.toBeNull();
    expect(api.getUnitAgenda).toHaveBeenCalledWith("u1", "2026-10-06");
    fireEvent.click(screen.getByRole("button", { name: "Voltar às vagas" }));
    expect(screen.getByLabelText("Dia")).not.toBeNull();
  });
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/modules/attendance/FitInPanel.test.tsx src/modules/attendance/BookPanel.test.tsx`
Expected: FAIL — `./FitInPanel` não existe; não há botão "Encaixe".

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/attendance/FitInPanel.tsx
// Encaixe (módulo 17; spec §4.3, ADR 0029): horário extra dentro do turno, com
// justificativa (≥ 10, texto congelado) e contado contra o limite do turno. O
// contador vem da agenda do dia; o api confere sob lock, e o 409 fit_in_limit
// recarrega a agenda para a tela mostrar o número de verdade.
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { ApiError, bookAppointment, getUnitAgenda, type HealthUnit, type RequestRow } from "../../lib/api";
import { attendanceError } from "../../lib/attendance";
import { cityLocalIso, fmtHourMinute } from "../../lib/format";
import { FIT_IN_REASON_MIN, confirmationWarning } from "../../lib/scheduling";
import { FrozenTextNotice } from "../../components/FrozenTextNotice";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";

interface Props { row: RequestRow; unit: HealthUnit; date: string; onBack(): void; onDone(): void }

function errorCode(err: unknown): string | undefined {
  return err instanceof ApiError ? (err.body as { error?: string } | undefined)?.error : undefined;
}

export function FitInPanel({ row, unit, date, onBack, onDone }: Props) {
  const queryClient = useQueryClient();
  const agenda = useQuery({ queryKey: [ "unitAgenda", unit.id, date ], queryFn: () => getUnitAgenda(unit.id, date) });
  const [ shiftId, setShiftId ] = useState("");
  const [ time, setTime ] = useState("");
  const [ reason, setReason ] = useState("");
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);

  const options = (agenda.data?.professionals ?? []).flatMap((p) =>
    p.shifts.filter((s) => !s.cancelled_at).map((s) => ({ professional: p, shift: s })));
  const chosen = options.find((o) => o.shift.shift_id === shiftId) ?? null;
  const startsAt = /^\d{2}:\d{2}$/.test(time) ? cityLocalIso(`${date}T${time}`) : null;
  const span = chosen ? `${fmtHourMinute(chosen.shift.starts_at)}–${fmtHourMinute(chosen.shift.ends_at)}` : "";
  const inside = !chosen || !startsAt ||
    (Date.parse(startsAt) >= Date.parse(chosen.shift.starts_at) && Date.parse(startsAt) < Date.parse(chosen.shift.ends_at));
  const atLimit = !!chosen && chosen.shift.fit_in_count >= chosen.shift.fit_in_limit;
  const reasonOk = reason.trim().length >= FIT_IN_REASON_MIN;
  const ready = !!chosen && !!startsAt && inside && !atLimit && reasonOk && !busy;
  const warning = startsAt && inside ? confirmationWarning(new Date(startsAt), new Date()) : null;

  async function submit() {
    if (!ready || !chosen || !startsAt) return;
    setBusy(true); setError(null);
    try {
      await bookAppointment(row.id, unit.id, {
        kind: "fit_in", professional_id: chosen.professional.id, shift_id: chosen.shift.shift_id, starts_at: startsAt,
        appointment_type_key: row.appointment_type_key, reason: reason.trim()
      });
      onDone();
    } catch (err) {
      const code = errorCode(err);
      if (code === "request_not_open") { onDone(); return; }
      if (code === "fit_in_limit") void queryClient.invalidateQueries({ queryKey: [ "unitAgenda", unit.id ] });
      setError(attendanceError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <section aria-label="Encaixe" style={panel}>
      <strong>{`Encaixe — ${row.appointment_type_name}`}</strong>
      {error && <p role="alert" style={alert}>{error}</p>}
      {agenda.isError && <p role="alert" style={alert}>{attendanceError(agenda.error)}</p>}
      <label style={{ ...label, maxWidth: 420 }}>Turno
        <select value={shiftId} style={inputStyle} onChange={(e) => setShiftId(e.target.value)}>
          <option value="">escolha…</option>
          {options.map(({ professional, shift }) => (
            <option key={shift.shift_id} value={shift.shift_id}>
              {`${professional.name} · ${fmtHourMinute(shift.starts_at)}–${fmtHourMinute(shift.ends_at)} · encaixes ${shift.fit_in_count} de ${shift.fit_in_limit}`}
            </option>
          ))}
        </select>
      </label>
      {chosen && <p style={hint}>{`Encaixes neste turno: ${chosen.shift.fit_in_count} de ${chosen.shift.fit_in_limit}`}</p>}
      {atLimit && <p role="alert" style={{ ...alert, fontWeight: 600 }}>limite de encaixes atingido neste turno</p>}
      <label style={{ ...label, maxWidth: 200 }}>Início do encaixe
        <input type="time" value={time} style={inputStyle} onChange={(e) => setTime(e.target.value)} />
      </label>
      {!inside && <small style={hint}>{`o encaixe precisa começar dentro do turno (${span})`}</small>}
      <label style={label}>Justificativa do encaixe
        <textarea value={reason} onChange={(e) => setReason(e.target.value)} style={{ ...inputStyle, minHeight: 60 }}
          aria-describedby="fit-in-reason-notice" />
      </label>
      <FrozenTextNotice id="fit-in-reason-notice" />
      {reason !== "" && !reasonOk && <small style={hint}>{`pelo menos ${FIT_IN_REASON_MIN} caracteres`}</small>}
      {warning && <p role="status" style={{ margin: 0, fontSize: 12.5, fontWeight: 600 }}>{warning}</p>}
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={!ready} onClick={() => void submit()} style={ready ? buttonStyle : disabledButtonStyle}>
          Confirmar encaixe
        </button>
        <button type="button" disabled={busy} onClick={onBack} style={secondaryButtonStyle}>Voltar às vagas</button>
      </div>
    </section>
  );
}

const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const hint: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 };
```

Em `src/modules/attendance/BookPanel.tsx`:
- `import { FitInPanel } from "./FitInPanel";` e estado `const [ fitIn, setFitIn ] = useState(false);`;
- logo depois do cálculo de `ready`, antes do `return` principal:

```tsx
  if (fitIn && day) {
    return <FitInPanel row={row} unit={unit} date={day} onBack={() => setFitIn(false)} onDone={onDone} />;
  }
```

- dentro do bloco `{day && !isLegacy && ( <> … </> )}`, depois da lista de vagas:

```tsx
          <div>
            <button type="button" style={secondaryButtonStyle} onClick={() => setFitIn(true)}>Encaixe</button>
          </div>
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/modules/attendance && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod17
/opt/homebrew/bin/git add src/modules/attendance/FitInPanel.tsx src/modules/attendance/FitInPanel.test.tsx \
  src/modules/attendance/BookPanel.tsx src/modules/attendance/BookPanel.test.tsx src/test/schedulingFixtures.ts
/opt/homebrew/bin/git commit -m "feat: fit in appointments with a reason and the shift limit counter

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Agenda da unidade por dia e profissional

**Files:**
- Modify: `src/modules/attendance/Agenda.tsx` (reescrito), `src/modules/attendance/Agenda.test.tsx` (reescrito)
- Modify: `src/lib/scheduling.ts` (acrescenta `APPOINTMENT_STATUS_LABEL` e `statusLabel`), `src/lib/scheduling.test.ts`
- Modify: `src/lib/api.ts` (remove `AgendaAppointment` e `listUnitAgenda`), `src/modules/Attendance.test.tsx` (mock)

**Interfaces:**
- Consumes: `getUnitAgenda`, `UnitAgenda`, `AppointmentView` (Task 1); `blockLine`, `appointmentFlags`, `BOOKING_KIND_LABEL` (Task 2); `unitAgenda`, `appointmentView` (fixtures, Tasks 2 e 9).
- Produces: `APPOINTMENT_STATUS_LABEL: Record<string, string>` e `statusLabel(status: string): string` em `src/lib/scheduling.ts` (usados também na Task 12); `AppointmentsTable({ rows })` exportado de `Agenda.tsx` (usado na Task 12).

- [ ] **Step 1: Write the failing tests**

Em `src/lib/scheduling.test.ts`:

```ts
describe("estado do horário", () => {
  it("rótulos de hoje; código novo aparece como veio", () => {
    expect(statusLabel("scheduled")).toBe("aguardando confirmação");
    expect(statusLabel("moved")).toBe("movido para outra unidade");
    expect(statusLabel("algo_novo")).toBe("algo_novo");
  });
});
```

```tsx
// src/modules/attendance/Agenda.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, getUnitAgenda: vi.fn() };
});

import * as api from "../../lib/api";
import { Agenda } from "./Agenda";
import { agendaShift, appointmentView, unitAgenda } from "../../test/schedulingFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };

function renderAgenda() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<Agenda unit={unit} />, { wrapper });
}

describe("Agenda", () => {
  beforeEach(() => { mocked(api.getUnitAgenda).mockReset(); });

  it("por profissional: turno com contador, faixas, horários com marcas e o motivo do encaixe", async () => {
    mocked(api.getUnitAgenda).mockResolvedValue(unitAgenda({ professionals: [
      { id: "p1", name: "Helena Duarte", shifts: [ agendaShift() ], appointments: [
        appointmentView(),
        appointmentView({ id: "a2", scheduled_at: "2026-10-06T10:10:00-03:00", ends_at: "2026-10-06T10:30:00-03:00",
          booking_kind: "fit_in", fit_in: true, fit_in_reason: "gestante com dor", citizen: { id: "c2", cpf_masked: "***.111.222-**" } }),
        appointmentView({ id: "a3", scheduled_at: "2026-10-06T11:20:00-03:00", ends_at: "2026-10-06T11:40:00-03:00",
          outside_template: true, status: "scheduled", citizen: { id: "c3", cpf_masked: "***.333.444-**" } })
      ] },
      { id: "p2", name: "Marta Lima", shifts: [ agendaShift({ shift_id: "s2", cancelled_at: "2026-10-05T08:00:00-03:00" }) ],
        appointments: [ appointmentView({ id: "a4", shift_cancelled: true, professional: { id: "p2", name: "Marta Lima" },
          citizen: { id: "c4", cpf_masked: "***.555.666-**" } }) ] }
    ] }));
    renderAgenda();
    const helena = await screen.findByRole("region", { name: "Helena Duarte" });
    expect(within(helena).getByText("Turno 07:00–12:00 · encaixes 1 de 2")).not.toBeNull();
    expect(within(helena).getByText("09:00–11:00 · agendável · consulta_medica")).not.toBeNull();
    expect(within(helena).getByText("09:00–09:20")).not.toBeNull();
    expect(within(helena).getByText("encaixe")).not.toBeNull();
    expect(within(helena).getByText("gestante com dor")).not.toBeNull();
    expect(within(helena).getByText("fora do modelo")).not.toBeNull();
    expect(within(helena).getByText("aguardando confirmação")).not.toBeNull();
    const marta = screen.getByRole("region", { name: "Marta Lima" });
    expect(within(marta).getByText("Turno 07:00–12:00 · encaixes 1 de 2 · cancelado")).not.toBeNull();
    expect(within(marta).getByText("turno cancelado")).not.toBeNull();
  });

  it("marcação livre aparece em 'Sem profissional'", async () => {
    mocked(api.getUnitAgenda).mockResolvedValue(unitAgenda({ professionals: [], unassigned: [
      appointmentView({ id: "l1", booking_kind: "legacy", professional: null, ends_at: null, shift_id: null,
        appointment_type_key: "retorno", appointment_type_name: "Retorno", scheduled_at: "2026-10-06T14:00:00-03:00" }) ] }));
    renderAgenda();
    const legacy = await screen.findByRole("region", { name: "Sem profissional (marcação livre)" });
    expect(within(legacy).getByText("14:00")).not.toBeNull();
    expect(within(legacy).getByText("Retorno")).not.toBeNull();
  });

  it("dia sem turno nem horário: estado vazio", async () => {
    mocked(api.getUnitAgenda).mockResolvedValue(unitAgenda({ professionals: [], unassigned: [] }));
    renderAgenda();
    expect(await screen.findByText("nenhum turno nem horário neste dia")).not.toBeNull();
  });

  it("seletor de data começa em hoje na cidade e trocar recarrega", async () => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date("2026-10-05T10:00:00-03:00"));
    mocked(api.getUnitAgenda).mockResolvedValue(unitAgenda({ professionals: [], unassigned: [] }));
    renderAgenda();
    await waitFor(() => expect(api.getUnitAgenda).toHaveBeenCalledWith("u1", "2026-10-05"));
    fireEvent.change(screen.getByLabelText("Data"), { target: { value: "2026-10-06" } });
    await waitFor(() => expect(api.getUnitAgenda).toHaveBeenCalledWith("u1", "2026-10-06"));
  });
});
```

Em `src/modules/Attendance.test.tsx`: troque `listUnitAgenda` por `getUnitAgenda` no `vi.mock` e no `beforeEach`, e `mocked(api.listUnitAgenda).mockResolvedValue([])` por `mocked(api.getUnitAgenda).mockResolvedValue({ date: "2026-10-05", professionals: [], unassigned: [] })`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/lib/scheduling.test.ts src/modules/attendance/Agenda.test.tsx`
Expected: FAIL — `statusLabel` não existe; a agenda de hoje chama `listUnitAgenda` e não tem regiões por profissional.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/scheduling.ts`:

```ts
export const APPOINTMENT_STATUS_LABEL: Record<string, string> = {
  scheduled: "aguardando confirmação",
  confirmed: "confirmado",
  checked_in: "check-in feito",
  cancelled_by_citizen: "cancelado pelo cidadão",
  expired: "sem confirmação no prazo",
  no_show: "faltou",
  moved: "movido para outra unidade"
};

export function statusLabel(status: string): string {
  return APPOINTMENT_STATUS_LABEL[status] ?? status;
}
```

```tsx
// src/modules/attendance/Agenda.tsx
// Agenda da unidade por dia (módulo 17; contratos §4.5): um bloco por
// profissional com turnos (faixas, contador de encaixes, cancelado) e
// horários (encaixe, fora do modelo, turno cancelado), e os horários de
// marcação livre sem profissional. O motivo do encaixe só vem para quem
// marca e para o municipal_admin (o api filtra).
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { getUnitAgenda, type AgendaProfessional, type AppointmentView, type HealthUnit } from "../../lib/api";
import { ATTENDANCE_REFETCH_MS, attendanceError } from "../../lib/attendance";
import { cityIsoDate, fmtHourMinute } from "../../lib/format";
import { appointmentFlags, blockLine, statusLabel } from "../../lib/scheduling";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { EmptyState } from "../../components/EmptyState";
import { Tag } from "../../components/Tag";
import { inputStyle } from "../../components/formStyles";

interface Props { unit: HealthUnit }

const span = (a: AppointmentView) => a.ends_at ? `${fmtHourMinute(a.scheduled_at)}–${fmtHourMinute(a.ends_at)}` : fmtHourMinute(a.scheduled_at);

export function AppointmentsTable({ rows }: { rows: AppointmentView[] }) {
  return (
    <DataTable<AppointmentView>
      cols={[
        { label: "Hora", w: "1fr", render: span },
        { label: "CPF", w: "1.5fr", render: (a) => a.citizen.cpf_masked },
        { label: "Atendimento", w: "1.5fr", render: (a) => a.appointment_type_name ?? "—" },
        { label: "Estado", w: "1.5fr", render: (a) => statusLabel(a.status) },
        { label: "Marcas", w: "2fr", render: (a) => {
          const flags = appointmentFlags(a);
          if (flags.length === 0 && !a.fit_in_reason) return "—";
          return (
            <span style={{ display: "flex", gap: 4, flexWrap: "wrap", alignItems: "center" }}>
              {flags.map((f) => <Tag key={f} tone={f === "turno cancelado" ? "down" : "warn"}>{f}</Tag>)}
              {a.fit_in_reason && <small style={{ color: "var(--ink3)" }}>{a.fit_in_reason}</small>}
            </span>
          );
        } }
      ]}
      rows={rows}
      rowKey={(a) => a.id}
      empty="nenhum horário"
    />
  );
}

function ProfessionalBlock({ p }: { p: AgendaProfessional }) {
  return (
    <section aria-label={p.name} style={{ display: "flex", flexDirection: "column", gap: 6 }}>
      <strong style={{ fontSize: 13 }}>{p.name}</strong>
      {p.shifts.map((s) => (
        <div key={s.shift_id} style={{ display: "flex", flexDirection: "column", gap: 2, fontSize: 12.5 }}>
          <span>{`Turno ${fmtHourMinute(s.starts_at)}–${fmtHourMinute(s.ends_at)} · encaixes ${s.fit_in_count} de ${s.fit_in_limit}${s.cancelled_at ? " · cancelado" : ""}`}</span>
          {s.blocks.map((b, i) => <span key={i} style={{ color: "var(--ink3)" }}>{blockLine(b, null)}</span>)}
        </div>
      ))}
      <AppointmentsTable rows={p.appointments} />
    </section>
  );
}

export function Agenda({ unit }: Props) {
  const [ date, setDate ] = useState(() => cityIsoDate());
  const query = useQuery({ queryKey: [ "unitAgenda", unit.id, date ], queryFn: () => getUnitAgenda(unit.id, date),
    refetchInterval: ATTENDANCE_REFETCH_MS
  });
  const data = query.data;
  const empty = !!data && data.professionals.length === 0 && data.unassigned.length === 0;

  return (
    <Panel
      title="Agenda do dia"
      right={
        <label style={{ display: "flex", alignItems: "center", gap: 6, fontSize: 12, color: "var(--ink2)" }}>
          Data
          <input type="date" value={date} onChange={(e) => setDate(e.target.value)} style={{ ...inputStyle, width: "auto", marginTop: 0 }} />
        </label>
      }
    >
      {query.isError ? (
        <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{attendanceError(query.error)}</p>
      ) : query.isPending || !data ? (
        <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando…</p>
      ) : empty ? (
        <EmptyState title="nenhum turno nem horário neste dia" />
      ) : (
        <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
          {data.professionals.map((p) => <ProfessionalBlock key={p.id} p={p} />)}
          {data.unassigned.length > 0 && (
            <section aria-label="Sem profissional (marcação livre)" style={{ display: "flex", flexDirection: "column", gap: 6 }}>
              <strong style={{ fontSize: 13 }}>Sem profissional (marcação livre)</strong>
              <AppointmentsTable rows={data.unassigned} />
            </section>
          )}
        </div>
      )}
    </Panel>
  );
}
```

Em `src/lib/api.ts`, apague `AgendaAppointment` e `listUnitAgenda` (a agenda nova é `getUnitAgenda`, Task 1).

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/lib/scheduling.test.ts src/modules/attendance src/modules/Attendance.test.tsx && npx tsc --noEmit && grep -rn "listUnitAgenda\|AgendaAppointment" src`
Expected: PASS; o `grep` não encontra nada.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod17
/opt/homebrew/bin/git add src/modules/attendance/Agenda.tsx src/modules/attendance/Agenda.test.tsx src/lib/scheduling.ts \
  src/lib/scheduling.test.ts src/lib/api.ts src/modules/Attendance.test.tsx
/opt/homebrew/bin/git commit -m "feat: show the unit agenda per professional with shifts, fit-ins and flags

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 11: Fila "sem unidade" com atribuição

**Files:**
- Create: `src/modules/attendance/UnassignedRequests.tsx`
- Modify: `src/modules/Attendance.tsx` (monta o painel para `canVerify`), `src/modules/Attendance.test.tsx` (mock)
- Test: `src/modules/attendance/UnassignedRequests.test.tsx`

**Interfaces:**
- Consumes: `listUnassignedRequests`, `assignRequestUnit`, `listActiveUnits`, `RequestRow` (Tasks 1 e 7); `requestKindLabel`, `requestMarks`, `fmtDueOn`, `PRIORITY_LABEL` (Tasks 2 e 7); `requestRow` (fixture).
- Produces: `UnassignedRequests()`; chave `[ "unassignedRequests" ]`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/attendance/UnassignedRequests.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, listUnassignedRequests: vi.fn(), assignRequestUnit: vi.fn(), listActiveUnits: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { UnassignedRequests } from "./UnassignedRequests";
import { requestRow } from "../../test/schedulingFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ROW = requestRow({ id: "r9", kind: "triage", origin: "triage", origin_unit_name: null, priority: "priority", due_on: "2026-10-08" });

function renderIt() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<UnassignedRequests />, { wrapper });
}

describe("UnassignedRequests", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    mocked(api.listUnassignedRequests).mockResolvedValue([ ROW ]);
    mocked(api.listActiveUnits).mockResolvedValue([ { id: "u1", name: "UBS Centro", kind: "ubs" }, { id: "u2", name: "UBS Xaxim", kind: "ubs" } ]);
  });

  it("lista os pedidos sem unidade com pedido, atendimento, prazo e prioridade", async () => {
    renderIt();
    expect(await screen.findByText("Triagem")).not.toBeNull();
    expect(screen.getByText("Consulta médica")).not.toBeNull();
    expect(screen.getByText("até 08/10")).not.toBeNull();
    expect(screen.getByText("prioritária")).not.toBeNull();
  });

  it("atribuir manda a unidade e relê a fila", async () => {
    mocked(api.assignRequestUnit).mockResolvedValue({ ...ROW });
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Atribuir unidade" }));
    const confirm = screen.getByRole("button", { name: "Confirmar unidade" }) as HTMLButtonElement;
    expect(confirm.disabled).toBe(true);
    await screen.findByRole("option", { name: "UBS Xaxim" });
    fireEvent.change(screen.getByLabelText("Unidade que vai marcar"), { target: { value: "u2" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar unidade" }));
    await waitFor(() => expect(api.assignRequestUnit).toHaveBeenCalledWith("r9", "u2"));
    await waitFor(() => expect(api.listUnassignedRequests).toHaveBeenCalledTimes(2));
    expect(screen.queryByRole("button", { name: "Confirmar unidade" })).toBeNull();
  });

  it("409 already_assigned avisa, fecha e relê", async () => {
    mocked(api.assignRequestUnit).mockRejectedValue(new ApiError(409, { error: "already_assigned" }, "x"));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Atribuir unidade" }));
    await screen.findByRole("option", { name: "UBS Centro" });
    fireEvent.change(screen.getByLabelText("Unidade que vai marcar"), { target: { value: "u1" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar unidade" }));
    expect(await screen.findByText("este pedido já foi atribuído a uma unidade")).not.toBeNull();
    await waitFor(() => expect(api.listUnassignedRequests).toHaveBeenCalledTimes(2));
  });

  it("fila vazia: estado vazio", async () => {
    mocked(api.listUnassignedRequests).mockResolvedValue([]);
    renderIt();
    expect(await screen.findByText("nenhum pedido sem unidade")).not.toBeNull();
  });
});
```

Em `src/modules/Attendance.test.tsx`: acrescente `listUnassignedRequests: vi.fn()` ao `vi.mock` e à lista do `beforeEach`, com `mocked(api.listUnassignedRequests).mockResolvedValue([]);`, e um teste:

```tsx
  it("recepção vê 'Pedidos sem unidade' mesmo sem unidade escolhida", async () => {
    renderAttendance();
    expect(await screen.findByRole("region", { name: "Pedidos sem unidade" })).not.toBeNull();
  });
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/modules/attendance/UnassignedRequests.test.tsx src/modules/Attendance.test.tsx`
Expected: FAIL — `./UnassignedRequests` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/attendance/UnassignedRequests.tsx
// Fila "sem unidade" (módulo 17; spec §5.2–§5.3, contratos §4.1): pedido de
// triagem cujo bairro não tem unidade de referência. Qualquer recepção da
// cidade atribui a unidade que vai marcar; 409 already_assigned = outra
// recepção chegou antes.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { ApiError, assignRequestUnit, listActiveUnits, listUnassignedRequests, type RequestRow } from "../../lib/api";
import { ATTENDANCE_REFETCH_MS, attendanceError } from "../../lib/attendance";
import { PRIORITY_LABEL, fmtDueOn, requestKindLabel, requestMarks } from "../../lib/scheduling";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { EmptyState } from "../../components/EmptyState";
import { Tag } from "../../components/Tag";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";

const KEY = [ "unassignedRequests" ] as const;

export function UnassignedRequests() {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: KEY, queryFn: listUnassignedRequests, refetchInterval: ATTENDANCE_REFETCH_MS });
  const [ assigning, setAssigning ] = useState<RequestRow | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const reload = () => void queryClient.invalidateQueries({ queryKey: KEY });

  return (
    <Panel title="Pedidos sem unidade" sub="bairro sem unidade de referência">
      <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
        {notice && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{notice}</p>}
        {query.isError ? <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{attendanceError(query.error)}</p>
          : query.isPending ? <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando…</p>
          : (query.data ?? []).length === 0 ? <EmptyState title="nenhum pedido sem unidade" />
          : (
            <DataTable<RequestRow>
              cols={[
                { label: "Pedido", w: "1fr", render: (r) => requestKindLabel(r) },
                { label: "Atendimento", w: "1.5fr", render: (r) => r.appointment_type_name },
                { label: "CPF", w: "1.5fr", render: (r) => r.cpf_masked },
                { label: "Prazo", w: "1fr", render: (r) => fmtDueOn(r.due_on) },
                { label: "Prioridade", w: "1fr", render: (r) =>
                  <Tag tone={r.priority === "priority" ? "warn" : "neutral"}>{PRIORITY_LABEL[r.priority]}</Tag> },
                { label: "Marcas", w: "1.5fr", render: (r) => requestMarks(r).map((m) => m.label).join(", ") || "—" },
                { label: "", w: "auto", align: "right", render: (r) => (
                  <button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setAssigning(r); }}>
                    Atribuir unidade
                  </button>
                ) }
              ]}
              rows={query.data ?? []}
              rowKey={(r) => r.id}
            />
          )}
        {assigning && (
          <AssignPanel
            key={assigning.id}
            row={assigning}
            onCancel={() => setAssigning(null)}
            onDone={(unitId) => {
              setAssigning(null); reload();
              void queryClient.invalidateQueries({ queryKey: [ "unitRequests", unitId ] });
            }}
            onConflict={(message) => { setAssigning(null); setNotice(message); reload(); }}
          />
        )}
      </div>
    </Panel>
  );
}

function AssignPanel({ row, onCancel, onDone, onConflict }: {
  row: RequestRow; onCancel(): void; onDone(unitId: string): void; onConflict(message: string): void;
}) {
  const units = useQuery({ queryKey: [ "activeUnits" ], queryFn: listActiveUnits });
  const [ unitId, setUnitId ] = useState("");
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);

  async function confirm() {
    if (busy || !unitId) return;
    setBusy(true); setError(null);
    try {
      await assignRequestUnit(row.id, unitId);
      onDone(unitId);
    } catch (err) {
      const code = err instanceof ApiError ? (err.body as { error?: string } | undefined)?.error : undefined;
      if (code === "already_assigned") { onConflict(attendanceError(err)); return; }
      setError(attendanceError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <section style={{ display: "flex", flexDirection: "column", gap: 10, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 }}>
      <strong>{`Atribuir unidade — ${row.cpf_masked}`}</strong>
      {error && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{error}</p>}
      <label style={{ display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)", maxWidth: 320 }}>
        Unidade que vai marcar
        <select value={unitId} onChange={(e) => setUnitId(e.target.value)} style={inputStyle}>
          <option value="">escolha…</option>
          {(units.data ?? []).map((u) => <option key={u.id} value={u.id}>{u.name}</option>)}
        </select>
      </label>
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={!unitId || busy} onClick={() => void confirm()}
          style={!unitId || busy ? disabledButtonStyle : buttonStyle}>Confirmar unidade</button>
        <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Cancelar</button>
      </div>
    </section>
  );
}
```

Em `src/modules/Attendance.tsx`: `import { UnassignedRequests } from "./attendance/UnassignedRequests";` e, logo depois de `{canVerify && unit && <Agenda unit={unit} />}`:

```tsx
      {canVerify && <UnassignedRequests />}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/modules/attendance src/modules/Attendance.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod17
/opt/homebrew/bin/git add src/modules/attendance/UnassignedRequests.tsx src/modules/attendance/UnassignedRequests.test.tsx \
  src/modules/Attendance.tsx src/modules/Attendance.test.tsx
/opt/homebrew/bin/git commit -m "feat: assign a unit to scheduling requests without one

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Minha agenda (dia e semana, só leitura)

**Files:**
- Create: `src/modules/MyAgenda.tsx`
- Modify: `src/shell/modules.ts` (`ModuleId`, item no grupo "Atendimento", filtro por papel), `src/shell/modules.test.ts`, `src/App.tsx`
- Test: `src/modules/MyAgenda.test.tsx`

**Interfaces:**
- Consumes: `getMyAgenda`, `MyAgenda`, `MyAgendaShift` (com `starts_at`, `ends_at`, `cancelled_at`), `ApiError` (Task 1); `weekStart`, `addDaysIso`, `dayLabel`, `blockLine`, `MY_AGENDA_DAYS` (Task 2); `AppointmentsTable` (Task 10); `SegmentedControl`; `useAuth`; `ATTENDANCE_REFETCH_MS`.
- Produces: `MyAgenda()`; `ModuleId` ganha `"my-agenda"`.

- [ ] **Step 1: Write the failing tests**

```tsx
// src/modules/MyAgenda.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getMyAgenda: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { AuthProvider } from "../lib/auth";
import { MyAgenda } from "./MyAgenda";
import { MORNING, NOW, appointmentView } from "../test/schedulingFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function renderIt() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  render(<MyAgenda />, { wrapper });
}

const DAY: api.MyAgenda = { days: [ { date: "2026-10-05", shifts: [
  { shift_id: "s1", unit: { id: "u1", name: "UBS Centro" }, starts_at: "2026-10-05T07:00:00-03:00",
    ends_at: "2026-10-05T12:00:00-03:00", cancelled_at: null, blocks: MORNING.blocks,
    appointments: [ appointmentView({ scheduled_at: "2026-10-05T09:00:00-03:00", ends_at: "2026-10-05T09:20:00-03:00" }) ] } ] } ] };

describe("MyAgenda", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW));
    mocked(api.fetchCurrentSession).mockResolvedValue({ id: "u-prof", email_address: "medica@c.gov.br", operator: false,
      mfa_enrolled: true, mfa_verified_at: null,
      memberships: [ { city_slug: "m1", city_name: "Curitiba", city_uf: "PR", role: "health_professional" } ] });
    mocked(api.getMyAgenda).mockResolvedValue(DAY);
  });

  it("dia: pede hoje a hoje e mostra unidade, faixas e horários, sem botão de ação", async () => {
    renderIt();
    await waitFor(() => expect(api.getMyAgenda).toHaveBeenCalledWith("2026-10-05", "2026-10-05"));
    const day = await screen.findByRole("region", { name: "seg 05/10" });
    expect(within(day).getByText("UBS Centro")).not.toBeNull();
    expect(within(day).getByText("Turno 07:00–12:00")).not.toBeNull();
    expect(within(day).getByText("09:00–11:00 · agendável · consulta_medica")).not.toBeNull();
    expect(within(day).getByText("09:00–09:20")).not.toBeNull();
    expect(within(day).getByText("***.982.247-**")).not.toBeNull();
    expect(within(day).queryAllByRole("button")).toHaveLength(0);
  });

  it("semana pede de segunda a domingo; próximo avança 7 dias", async () => {
    vi.setSystemTime(new Date("2026-10-08T10:00:00-03:00")); // quinta
    renderIt();
    await waitFor(() => expect(api.getMyAgenda).toHaveBeenCalledWith("2026-10-08", "2026-10-08"));
    fireEvent.click(await screen.findByRole("tab", { name: "Semana" }));
    await waitFor(() => expect(api.getMyAgenda).toHaveBeenLastCalledWith("2026-10-05", "2026-10-11"));
    fireEvent.click(screen.getByRole("button", { name: "próximo →" }));
    await waitFor(() => expect(api.getMyAgenda).toHaveBeenLastCalledWith("2026-10-12", "2026-10-18"));
  });

  it("dia sem turno: estado vazio", async () => {
    mocked(api.getMyAgenda).mockResolvedValue({ days: [] });
    renderIt();
    expect(await screen.findByText("nenhum turno no período")).not.toBeNull();
  });

  it("sem cadastro profissional (404): a frase de 'Meu perfil'", async () => {
    mocked(api.getMyAgenda).mockRejectedValue(new ApiError(404, { error: "no_profile" }, "x"));
    renderIt();
    expect(await screen.findByText("Seu cadastro profissional ainda não foi feito. Fale com a administração da cidade.")).not.toBeNull();
  });
});
```

Em `src/shell/modules.test.ts`, dentro de `describe("navGroupsFor", …)`:

```ts
    it("Minha agenda só para health_professional, no grupo Atendimento", () => {
      const prof = { operator: false, memberships: [ { role: "health_professional" } ] };
      const admin = { operator: false, memberships: [ { role: "municipal_admin" } ] };
      const items = (u: typeof prof) => navGroupsFor(u).find((g) => g.label === "Atendimento")?.items.map((i) => i.id) ?? [];
      expect(items(prof)).toContain("my-agenda");
      expect(items(admin)).not.toContain("my-agenda");
      expect(labelFor("my-agenda")).toBe("Minha agenda");
    });
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/modules/MyAgenda.test.tsx src/shell/modules.test.ts`
Expected: FAIL — `./MyAgenda` não existe; `"my-agenda"` não é `ModuleId`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/MyAgenda.tsx
// Minha agenda (módulo 17; spec §7, contratos §3): o profissional lê os
// próprios turnos, faixas e horários, por dia ou semana (segunda a domingo,
// no fuso da cidade). Só leitura: quem marca é a recepção.
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { ApiError, getMyAgenda, type MyAgendaShift } from "../lib/api";
import { ATTENDANCE_REFETCH_MS, attendanceError } from "../lib/attendance";
import { useAuth } from "../lib/auth";
import { cityIsoDate, fmtHourMinute } from "../lib/format";
import { MY_AGENDA_DAYS, addDaysIso, blockLine, dayLabel, weekStart } from "../lib/scheduling";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { EmptyState } from "../components/EmptyState";
import { inputStyle, secondaryButtonStyle } from "../components/formStyles";
import { SegmentedControl } from "../shell/SegmentedControl";
import { AppointmentsTable } from "./attendance/Agenda";

type View = "day" | "week";
const VIEWS: { key: View; label: string }[] = [ { key: "day", label: "Dia" }, { key: "week", label: "Semana" } ];

export function MyAgenda() {
  const { user } = useAuth();
  const [ view, setView ] = useState<View>("day");
  const [ date, setDate ] = useState(() => cityIsoDate());
  const from = view === "day" ? date : weekStart(date);
  const to = view === "day" ? date : addDaysIso(from, MY_AGENDA_DAYS - 1);
  const step = view === "day" ? 1 : MY_AGENDA_DAYS;
  // Chave por usuário (F-10.5): o QueryClient sobrevive à troca de sessão.
  const query = useQuery({ queryKey: [ "myAgenda", user?.id ?? null, from, to ], queryFn: () => getMyAgenda(from, to),
    refetchInterval: ATTENDANCE_REFETCH_MS });
  const noProfile = query.error instanceof ApiError && query.error.status === 404;
  const days = (query.data?.days ?? []).filter((d) => d.shifts.length > 0);

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Minha agenda" sub="só leitura · quem marca é a recepção" />
      <div style={{ display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" }}>
        <SegmentedControl options={VIEWS} value={view} onChange={setView} />
        <button type="button" style={secondaryButtonStyle} onClick={() => setDate(addDaysIso(date, -step))}>← anterior</button>
        <label style={{ display: "flex", alignItems: "center", gap: 6, fontSize: 12, color: "var(--ink2)" }}>Data
          <input type="date" value={date} onChange={(e) => e.target.value && setDate(e.target.value)}
            style={{ ...inputStyle, width: "auto", marginTop: 0 }} />
        </label>
        <button type="button" style={secondaryButtonStyle} onClick={() => setDate(addDaysIso(date, step))}>próximo →</button>
      </div>
      {noProfile ? <EmptyState title="Seu cadastro profissional ainda não foi feito. Fale com a administração da cidade." />
        : query.isError ? <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{attendanceError(query.error)}</p>
        : query.isPending ? <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando…</p>
        : days.length === 0 ? <EmptyState title="nenhum turno no período" />
        : days.map((d) => (
          <Panel key={d.date} title={dayLabel(d.date)}>
            <div style={{ display: "flex", flexDirection: "column", gap: 14 }}>
              {d.shifts.map((s) => <ShiftBlock key={s.shift_id} shift={s} />)}
            </div>
          </Panel>
        ))}
    </div>
  );
}

function ShiftBlock({ shift }: { shift: MyAgendaShift }) {
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
      <strong style={{ fontSize: 13 }}>{shift.unit.name}</strong>
      <span style={{ fontSize: 12.5 }}>
        {`Turno ${fmtHourMinute(shift.starts_at)}–${fmtHourMinute(shift.ends_at)}${shift.cancelled_at ? " · cancelado" : ""}`}
      </span>
      {shift.blocks.map((b, i) => <span key={i} style={{ fontSize: 12, color: "var(--ink3)" }}>{blockLine(b, null)}</span>)}
      <AppointmentsTable rows={shift.appointments} />
    </div>
  );
}
```

Em `src/shell/modules.ts`:
- `ModuleId` ganha `| "my-agenda"`;
- o grupo "Atendimento" passa a ser:

```ts
  { label: "Atendimento", items: [
    { id: "attendance", label: "Atendimento", icon: "☑" },
    { id: "my-agenda", label: "Minha agenda", icon: "◷" }
  ]},
```

- o filtro de itens em `navGroupsFor`:

```ts
    // Módulo 10/17: "Meu perfil" e "Minha agenda" são do profissional; sem
    // sessão ainda, somem (a API responderia 404 no_profile).
    items: group.items.filter((item) =>
      (item.id !== "my-profile" && item.id !== "my-agenda") || isProfessional)
```

Em `src/App.tsx`: `import { MyAgenda } from "./modules/MyAgenda";` e, no `switch`, depois de `my-profile`:

```tsx
    case "my-agenda":      return <MyAgenda />;
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run src/modules/MyAgenda.test.tsx src/shell && npx tsc --noEmit`
Expected: PASS. As contagens de `NAV_GROUPS.length` em `modules.test.ts` não mudam (nenhum grupo novo).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod17
/opt/homebrew/bin/git add src/modules/MyAgenda.tsx src/modules/MyAgenda.test.tsx src/shell/modules.ts src/shell/modules.test.ts \
  src/App.tsx
/opt/homebrew/bin/git commit -m "feat: add read-only my agenda for health professionals

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Suíte, build, revisão e prova no navegador

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd apps/dashboard/.claude/mod17 && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde. A CI roda os mesmos três; `dist/` está no `.gitignore`.

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod17 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod17 log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e a contagem `0`. Se o symlink aparecer, **não** o adicione.

- [ ] **Step 2: Nada sobrou do fluxo antigo**

```bash
cd apps/dashboard/.claude/mod17
grep -rn "scheduleRequest\|listUnitAgenda\|AgendaAppointment\|allowOverlap" src
grep -rn "reschedule_note" src --include=*.tsx | grep -v RequestDetailPanel | grep -v test
```

Expected: o primeiro só acha `allow_overlap` dentro de `BookingInput`/`BookPanel` (marcação livre); o segundo, nada (a nota do cidadão só aparece no detalhe).

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§3, §4, §5.3, §7, §8), o ADR 0029 e o arquivo de contratos (§1–§4 e §8). Pontos de atenção:
- a recepção nunca marca outra vaga sozinha: `slot_taken`, `slot_unavailable` e `use_slots` recarregam e limpam a escolha;
- marcação livre só aparece em `legacy_days`; encaixe só em dia com turno e nunca em turno cancelado;
- dias e semanas sempre no fuso da cidade (`cityIsoDate`, `weekStart`), nunca `new Date().toISOString().slice(0, 10)`;
- `reschedule_note` só no detalhe; `fit_in_reason` só mostrado quando o api manda; a justificativa do encaixe tem o `FrozenTextNotice`;
- o JSON do editor continua a fonte: abrir o painel "Agendamento" não reescreve a definição; lista vazia sai ausente; o autor sem acesso aos tipos continua editando;
- tipos, modelos, modelo do turno e tipo padrão só no módulo Profissionais (`municipal_admin`); Minha agenda só para `health_professional` e sem botão de ação;
- nenhum componente novo fora dos que já existem (interface tem ciclo próprio).

- [ ] **Step 4: Prova no navegador (com o usuário)**

Rode o api da branch do módulo 17 na porta **3034**, com a semente do módulo (spec §10: tipos da base, modelo "Manhã" em Curitiba, turnos da semana da médica e da enfermeira, "Saúde do idoso" com regra de agendamento). Depois o Vite do worktree apontando para ele:

```bash
cd apps/dashboard/.claude/mod17 && VITE_API_PROXY_TARGET=http://localhost:3034 npx vite --port 5184 --host 0.0.0.0
```

Abra `http://curitiba.localhost:5184/dashboard/`. O usuário faz o login; não digite senha nem TOTP. Confira com screenshot:
- como `admin@curitiba.demo`, em Profissionais:
  - "Tipos de atendimento": a base com origem "plataforma"; criar um tipo da cidade; desativar e reativar;
  - "Modelos de agenda": abrir "Manhã", criar sobreposição e ver o botão travar; pré-visualizar um turno de exemplo 07–12 com CBO de médico e ver as vagas de 09:00 a 10:40;
  - na ficha da médica: trocar o modelo de um turno e o tipo padrão do vínculo;
- como autor da `SignatureCrew`, no Editor de protocolo, carregar "Saúde do idoso": o painel "Agendamento" mostra a regra (rotina, 30 dias) com o tipo no select (o autor lê os tipos, §9); trocar no JSON para um tipo inexistente mostra o aviso do gate;
- como `admin@curitiba.demo` (que tem `citizen_verifier` em dev), em Atendimento, com um pedido gerado por uma triagem no wpda:
  - a fila mostra "Triagem", prazo e prioridade; "Detalhes" abre o pedido;
  - "Marcar horário": vagas por dia e profissional; marcar numa vaga; abrir outro pedido e tentar a mesma vaga em outra aba para ver o `slot_taken` recarregar;
  - "Encaixe": justificativa, contador; encaixar até o limite e ver a recusa acima dele;
  - "Agenda do dia": faixas, contador, encaixe marcado;
  - "Pedidos sem unidade": atribuir uma unidade;
- como a médica da semente, "Minha agenda": dia e semana, só leitura.

- [ ] **Step 5:** **Pare.** O merge do dashboard só vem depois do merge do api, e só com autorização explícita do usuário. A ordem de deploy é contracts → api → dashboard e wpda (spec §11).

---

## Nota: incorporado ao contrato

As doze divergências levantadas na escrita deste plano foram aceitas e estão na §9 do arquivo de contratos (2026-10-06); o plano segue o texto decidido:

1. **§4.1:** a prioridade numérica da triagem passa a ser `triage_priority`; `priority` é só `routine|priority` (Task 7).
2. **§4.1:** `origin_unit_name` é nulo em `kind = triage` (Task 7, `requestKindLabel`).
3. **§4.1:** `GET /attendance/requests/:id` = item da fila + `reschedule_note` (Task 7, `RequestDetail`).
4. **§4.3:** `health_unit_id` nas três formas; a marcação livre mantém 409 `slot_taken` com `taken` e `allow_overlap`; resposta 201 `{ appointment }` (Tasks 1 e 8).
5. **§4.4:** em `legacy`, `ends_at`, tipo, `professional` e `shift_id` vêm `null` (Task 1, `AppointmentView`; Task 10).
6. **§4.5:** turno da agenda com `starts_at`, `ends_at`, `cancelled_at` (Tasks 1, 9 e 10).
7. **§4.5 e §3:** faixas `bookable` das respostas trazem `appointment_type_name`; a tela usa o nome quando não tem a lista de tipos e nunca o devolve ao salvar um modelo (Tasks 1, 2, 4, 10 e 12).
8. **§3:** Minha agenda com `unit: { id, name }` e turno com horários e `cancelled_at`; 404 `no_profile` sem cadastro; `from`/`to` inclusivos, também em `availability` (Tasks 1, 8 e 12).
9. **§3:** `protocol_author` e `protocol_reviewer` leem `GET /professionals/appointment_types`; o texto livre do painel "Agendamento" fica só como recaída (Task 6, Review Focus 5).
10. **§1:** aviso do gate = 200 `{ warnings: [string] }` (Tasks 1 e 6).
11. **§3:** `shifts/:id/template` devolve o turno, `links/:id/default_type` devolve o vínculo, e `GET /professionals/:id` traz `default_appointment_type_key` por vínculo (Tasks 1 e 5).
12. **§3:** 422 `invalid_name` (tipo e modelo); `invalid_blocks` com `detail` `empty`, `bad_slot_minutes` (5–240), `inactive_type` e `crosses_midnight` (fim depois do início, sem `24:00`); `cbo_prefixes` de 1 a 20 itens com 1 a 6 dígitos (Task 2, espelhado nas Tasks 3 e 4).

## Self-review

- **Cobertura da spec (§7 e o que ela puxa):**
  - Profissionais: tipos (ajustar, desativar, criar) — Task 3; modelos com editor de faixas e pré-visualização — Task 4; modelo no turno e tipo padrão no vínculo — Task 5;
  - Atendimento: fila com marcas e ordem do api, detalhe — Task 7; marcar a partir do pedido com vagas por dia e profissional, marcação livre em `legacy_days`, `slot_taken` recarregando — Task 8; encaixe com justificativa e contador — Task 9; agenda da unidade por dia (faixas, encaixes, fora do modelo, turno cancelado) — Task 10; fila "sem unidade" com atribuição — Task 11;
  - Minha agenda (dia e semana, só leitura) — Task 12;
  - editor de protocolo, painel "Agendamento" com o construtor do módulo 15 — Task 6;
  - §8 LGPD (nota só no detalhe, motivo do encaixe só quando vem, aviso de texto congelado) — Tasks 7, 9, 10 e a revisão da Task 13;
  - §9 testes de front com relógio fixo — Tasks 2, 4, 5, 8, 9, 10, 12; prova no navegador — Task 13.
- **Contrato §9:** todas as formas decididas estão nos tipos da Task 1, nas regras da Task 2 e nas telas; nenhuma divergência em aberto.
- **Placeholders:** nenhum. Arquivos existentes são alterados por trechos exatos; `Agenda.tsx` e `Agenda.test.tsx` são reescritos inteiros.
- **Consistência de nomes:** `APPOINTMENT_TYPES_KEY` (Task 3) e `SCHEDULE_TEMPLATES_KEY` (Task 4) usados na Task 5; `availabilityKey` (Task 8) usado em `Requests`; `[ "unitAgenda", unitId, date ]` é a mesma chave em `FitInPanel` (Task 9) e `Agenda` (Task 10); `typeLabel`/`blockLine` aceitam `null` desde a Task 2 e são chamados assim nas Tasks 10 e 12; `requestKindLabel` (Task 7) usado nas Tasks 7, 11; `statusLabel` e `AppointmentsTable` (Task 10) usados na Task 12; `requestRow`, `agendaShift`, `unitAgenda` são acrescentados às fixtures antes do primeiro uso (Tasks 7 e 9).
- **Review Focus:** 1 — Task 8 ("slot_taken…", "use_slots…"); 2 — Task 2 ("vaga das 23h30…", "semana começa na segunda") e Task 12 ("semana pede de segunda a domingo"); 3 — Tasks 2, 4 e 6 (tipo inativo: `inactive_type` no modelo, aviso na regra); 4 — Task 9 ("fit_in_limit recarrega o contador"); 5 — Task 6 ("sem lista de tipos…").
