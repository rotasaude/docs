# Módulo 18 — Acolhimento (escuta inicial) (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Telas do módulo 18 no painel da cidade. Em Atendimento, o profissional com vínculo na unidade vê a fila do acolhimento, inicia ou retoma a escuta, escolhe a queixa em CIAP-2 por nome, registra os sinais vitais (alerta destacado, IMC na hora), vê a cor sugerida pelo protocolo assinado com o motivo, decide a cor final (justificando quando muda), escolhe o destino — no dia, agendar (pedido do módulo 17), orientação ou encaminhar —, abandona ou conclui; na fila do profissional, a cor aparece em todos os itens, o vermelho fica destacado, a escuta abre dentro do atendimento chamado e quem espera com destino "no dia" pode ser reavaliado; a recepção só vê a cor. Em Unidades, o `municipal_admin` escolhe o escopo do acolhimento. No editor de protocolo, o autor monta o protocolo `kind: "screening"` com as regras de cor no construtor de condições (variáveis `vitals.*`, `complaint.ciap2`) e confere num simulador com sinais vitais. Em Produção e-SUS, as recusas aparecem como códigos (`last_error_codes`) e um painel lista as "Fichas que não puderam ser geradas", com "gerar de novo" sob step-up.

**Architecture:** O cliente HTTP novo vai para o fim de `src/lib/api.ts`, com os tipos copiados do arquivo de contratos (e as quatro rotas/campos das Divergências D1–D4 marcados no comentário). As regras ficam fora do React, em funções puras: `src/lib/screening.ts` (rótulos, plausibilidade dos sinais espelhando o 422, IMC, alertas, cor, destino, espera, tradução de erro) e `src/lib/riskRules.ts` (leitura e escrita de `risk_rules`, no padrão de `schedulingRules.ts`). As telas novas ficam em `src/modules/attendance/` (`Ciap2Search`, `VitalSignsFields`, `ColorDecision`, `ScreeningForm`, `ScreeningQueue`, `ScreeningDetail`), em `src/modules/protocolEditor/` (`RiskRulesPanel`, `ScreeningSimulator`) e em `src/modules/production/GenerationFailures.tsx`; as existentes ganham o mínimo: `Attendance` monta o painel do acolhimento, `UnitQueue` ganha cor, destaque, espera, "Ver escuta" e "Reavaliar", `DataTable` ganha `rowStyle`, `UnitForm`/`Units` ganham o escopo, `condition.ts`/`ConditionBuilder` ganham o contexto `screening` e o campo de códigos, `ProtocolEditor` troca a coluna da direita quando a definição é `screening`, `Production` mostra códigos. A cor sugerida vem sempre do api (`suggest`, sem gravar); a tela nunca avalia regra.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("../lib/api")` e `vi.stubGlobal("fetch")`), Vite 5 (proxy de dev).

**Spec:** `docs/.claude/ciclo2/superpowers/specs/2026-10-07-module-18-screening-design.md` (§8 é deste plano; §3, §4, §5 e §6 dão as regras), `docs/.claude/ciclo2/adr/0030.md` e o arquivo de contratos `docs/.claude/ciclo2/superpowers/plans/2026-10-07-module-18-screening-contracts.md` (§1–§6; a unidade é editada em `POST /attendance/units/:id`). O plano do `api` do módulo 18 (`2026-10-07-module-18-screening-api.md`) precisa estar **mergeado antes** do merge deste (contratos §8); este plano segue os códigos de alerta e o limite da glicemia que ele fixou (Desvio 3 e Task 6 dele).

## Global Constraints

- Rotas usadas (sessão da cidade; nenhuma entrada nova de proxy: `/attendance`, `/authoring` e `/production` já estão no `vite.config.ts`):
  - `GET /attendance/units/:id/screening_queue` → `{ items }`; `POST /attendance/attendances/:id/screening` → 201 `<screening>`; `POST /attendance/screenings/:id/abandon`; `POST /attendance/screenings/suggest`; `POST /attendance/screenings/:id/complete`; `POST /attendance/screenings/:id/reassess`; `GET /attendance/screenings/:id` (com `revisions`; gera `screening.viewed`);
  - `GET /attendance/units/:id/queue` (existente): item com `screening: { color, destination, waited_minutes } | null`; `POST /attendance/attendances/:id/call` e `POST /attendance/units/:id/call_next` (existentes) devolvem também `screening`;
  - `POST /attendance/units/:id` (existente) aceita `screening_scope`; `GET /attendance/units` e `units/all` devolvem o campo;
  - `GET /production` (existente): `last_error_codes: [{ field, code }]` no lugar de `last_error`; `rejections: [{ field, code, count }]`; `GET /production/generation_failures?resolved=false` → `{ items }`; `POST /production/generation_failures/:id/retry` (step-up) → o item; 409 `already_resolved`;
  - propostas deste plano (Divergências): `POST /attendance/ciap2/search` `{ q }` → `{ items: [{ code, label }] }` (D1); `id` no bloco `screening` da fila do profissional (D2); `attendance_id` no `suggest` em vez de `citizen_id` (D3); `POST /authoring/protocols/simulate_screening` (D4).
- Valores (contratos §2): cor `red` | `yellow` | `green` | `blue` (gravidade nessa ordem; na tela "vermelho", "amarelo", "verde", "azul", com tons `down`, `warn`, `ok`, `info` do `Tag`); destino `same_day` | `schedule` | `oriented` | `referred`; escuta `in_progress` | `completed` | `abandoned`; `screening_scope` `walk_in` | `all`; `glucose_moment` `fasting` | `postprandial` | `random`.
- Regras espelhadas na tela (o api continua sendo quem garante): CIAP-2 obrigatório (`^[A-Z]\d{2}$` no construtor); sinais todos opcionais; sistólica 50–300, diastólica 20–200 e menor que a sistólica, as duas juntas; FC 20–250; FR 4–80; temperatura 30–45 com 1 casa; SpO2 50–100; glicemia 10–800 com momento obrigatório; peso 0,5–400 com 2 casas; altura 30–250; dor 0–10; vírgula decimal aceita; IMC = peso/altura² com 1 casa; justificativa de cor ≥ 10 quando a final difere da sugerida (sem sugestão, não há justificativa); orientação obrigatória e ≤ 500; queixa em texto ≤ 500; agendar exige tipo e prazo 1–365, padrão pela cor (amarelo 7, verde 15, azul 30, vermelho sem padrão); encaminhar exige unidade ou descrição; `risk_rules` 1–50.
- LGPD (spec §6; ADR 0030): a recepção nunca recebe nem mostra queixa ou sinais — na fila só a cor (e a espera); "Ver escuta" e "Reavaliar" só para quem tem `canCare`; texto livre (queixa, justificativa, orientação, descrição do encaminhamento, termo de busca da CIAP-2) só em corpo de POST, nunca em URL; todo texto livre com o `FrozenTextNotice`.
- Papéis: acolhimento e escuta para `health_professional` com vínculo ativo na unidade (o `canCare` de `Attendance.tsx`; CBO e escopo o api confere, e `cbo_not_allowed` vira frase); recepção (`citizen_verifier`) só lê a fila; escopo da unidade só `municipal_admin` (painel Unidades); protocolo no editor para quem já usa o editor; Produção para `municipal_admin` e `analyst`, "gerar de novo" só `municipal_admin`.
- Interface tem ciclo próprio: use os componentes e estilos existentes (`Panel`, `DataTable`, `Tag`, `EmptyState`, `FrozenTextNotice`, `SensitiveAction`, `formStyles`, `ConditionBuilder`), sem redesign. O único acréscimo a componente comum é `DataTable.rowStyle` (destaque de linha).
- Testes que dependem de "agora" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(new Date("2026-10-07T10:00:00-03:00"))` (constante `NOW18`) e `vi.useRealTimers()` no `afterEach`. Funções puras recebem `now` como argumento. A espera do `suggest` é prop (`suggestDelayMs`), zerada nos testes.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

- Crie o worktree a partir de `origin/main` e ligue o `node_modules` do checkout principal. `/.claude/` já está no `.gitignore` do dashboard. A partir da raiz do monorepo (`/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/dashboard fetch origin
  /opt/homebrew/bin/git -C apps/dashboard log --oneline -1 origin/main
  /opt/homebrew/bin/git -C apps/dashboard worktree add .claude/mod18 -b feat/mod-18-screening origin/main
  ln -s ../../node_modules apps/dashboard/.claude/mod18/node_modules
  ```

  Expected: `origin/main` em `ab00e4f fix: clear overlap state on slot reload, reload on type_not_served, accept null slot_minutes` (ou mais novo; se algum arquivo deste plano mudou, confira os trechos "antes" dos diffs antes de aplicar).

- Testes (script `test` = `vitest run`): `cd apps/dashboard/.claude/mod18 && npx vitest run <arquivos>`. Base de hoje: 114 arquivos, 1217 testes verdes.
- Tipos antes de cada commit (script `typecheck` = `tsc --noEmit`; o `tsconfig` inclui os testes): `cd apps/dashboard/.claude/mod18 && npx tsc --noEmit`.
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`.
- Os diffs abaixo têm o contexto do arquivo depois das tasks anteriores (para quase todos, igual a `origin/main` `ab00e4f`; `src/lib/api.ts` da Task 11 parte do estado da Task 1). Aplique à mão ou com `git apply` a partir de um arquivo no scratchpad (nunca dentro do worktree).
- O api do módulo 18 roda em dev na porta **3035** (Task 18 do plano do api); a prova no navegador aponta o proxy para ela (Task 12).
- Conferido em 2026-10-07: com as Tasks 1–11 aplicadas numa cópia de `origin/main`, `npx vitest run` dá 125 arquivos e 1329 testes verdes, `npx tsc --noEmit` limpo e `npm run build` ok.

## Review Focus

1. **Sugestão que chega fora de ordem.** A pessoa escolhe a queixa e digita a temperatura antes de a primeira sugestão voltar; a resposta velha (vermelho) não pode sobrescrever a nova (amarelo), senão a cor final e o motivo mostram outra regra. Teste: Task 5, "resposta velha da sugestão é descartada".
2. **O api pede justificativa que a tela não pedia.** A sugestão mudou no servidor (protocolo ativado agora, ou a última sugestão falhou) e a cor final ficou diferente da sugerida de verdade: 422 `color_change_reason_required` tem de abrir o campo de justificativa, avisar e pedir sugestão nova, sem perder o formulário. Testes: Task 4, "forceReason mostra a justificativa mesmo com a cor igual à sugerida"; Task 5, "422 color_change_reason_required mostra a justificativa e pede nova sugestão".
3. **Outro profissional (ou a chamada) mexeu primeiro.** Duas enfermeiras clicam "Iniciar escuta" na mesma pessoa; ou o médico chama a pessoa no meio da escuta (o api abandona a escuta): a tela recarrega a fila e diz a frase, sem abrir formulário nem ficar com erro parado. Testes: Task 6, "already_screening recarrega a fila e avisa, sem abrir formulário"; Task 5, "atendimento que saiu da espera fecha o formulário com a frase".
4. **A recepção nunca vê nem lê a escuta.** No mesmo painel da fila, a recepção vê a cor e a espera, mas nada de "Ver escuta"/"Reavaliar" e nenhuma chamada a `GET /attendance/screenings/:id`; o painel do acolhimento nem aparece para ela. Testes: Task 7, "recepção vê só a cor: sem Reavaliar nem Ver escuta, e nada é lido"; Task 6, "acolhimento (módulo 18): profissional com vínculo vê a fila da escuta; a recepção não".
5. **Números como a pessoa digita.** `37,8` com vírgula, `0,5` kg, a borda exata (300 mmHg, 800 mg/dL), `88,5` bpm, `37,85` °C, pressão só com a sistólica, diastólica igual à sistólica, glicemia sem momento: a tela diz o problema sob o campo e não manda o valor; nunca deixa a escuta seguir com um sinal que o api recusaria sem dizer qual. Testes: Task 2 (tabelas "nas bordas é aceito", "é recusado com a frase", "pressão: as duas juntas…", "glicemia pede o momento…"); Task 4, "pressão pela metade e glicemia sem momento avisam".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `src/lib/api.ts`, `src/lib/api.screening.test.ts`, `src/test/screeningFixtures.ts`, `src/lib/scheduling.ts`, `src/lib/scheduling.test.ts` | tipos do contrato; cliente da escuta, sugestão, CIAP-2, simulador, fichas não geradas; `screening` na fila e na chamada; escopo da unidade; pedido `screening` | 1 |
| `src/lib/screening.ts`, `src/lib/screening.test.ts`, `src/lib/attendance.ts` | rótulos, sinais vitais, IMC, alertas, cor, destino, espera; frases das recusas | 2 |
| `src/modules/attendance/Ciap2Search.tsx` (+ teste) | queixa em CIAP-2 por nome ou código | 3 |
| `src/modules/attendance/VitalSignsFields.tsx`, `src/modules/attendance/ColorDecision.tsx` (+ testes) | campos dos sinais com alerta e IMC; cor sugerida com motivo e cor final | 4 |
| `src/modules/attendance/ScreeningForm.tsx` (+ teste) | concluir (com destino) e reavaliar; sugestão sem resposta velha | 5 |
| `src/modules/attendance/ScreeningQueue.tsx` (+ teste), `src/modules/Attendance.tsx`, `src/modules/Attendance.test.tsx` | fila do acolhimento no Atendimento | 6 |
| `src/components/DataTable.tsx` (+ teste), `src/modules/attendance/ScreeningDetail.tsx`, `src/modules/attendance/UnitQueue.tsx`, `src/modules/attendance/UnitQueue.screening.test.tsx` | cor e destaque na fila do profissional; escuta no atendimento chamado; reavaliar | 7 |
| `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/attendance/Units.test.tsx` | escopo do acolhimento na unidade | 8 |
| `src/lib/condition.ts`, `src/lib/conditionPhrase.ts`, `src/modules/protocols/ConditionBuilder.tsx` (+ testes) | contexto `screening` e campo de códigos CIAP-2 | 9 |
| `src/lib/riskRules.ts`, `src/modules/protocolEditor/RiskRulesPanel.tsx`, `src/modules/protocolEditor/ScreeningSimulator.tsx`, `src/modules/ProtocolEditor.tsx` (+ testes) | protocolo de acolhimento no editor | 10 |
| `src/lib/api.ts`, `src/lib/production.ts`, `src/modules/Production.tsx`, `src/modules/production/GenerationFailures.tsx`, `src/test/recordModeFixtures.ts` (+ testes) | códigos de recusa e fichas não geradas | 11 |
| — | suíte, build, revisão e prova no navegador | 12 |

**Estratégia de teste:** regras puras com tabela de casos (Task 2, Task 9, Task 10), cliente HTTP com `fetch` falso conferindo URL, método e corpo (Task 1), e cada tela com `vi.mock("../lib/api")` cobrindo o caminho feliz, cada recusa que muda o comportamento da tela e as bordas do Review Focus. Componentes que compõem outros já testados trocam o filho por um dublê (`vi.mock("./ScreeningForm")`) para testar só o próprio fluxo. Relógio fixo em todo teste que depende de "agora". A prova final é no navegador, contra o api do módulo 18 (Task 12).

---
### Task 1: Cliente HTTP do módulo 18 e tipos do contrato

**Files:**
- Modify: `src/lib/api.ts` (`HealthUnit` e `updateUnit`; `QueueRow`; `callAttendance`/`callNext`; `RequestRow`; seção nova no fim do arquivo)
- Modify: `src/lib/scheduling.ts` (`requestKindLabel`), `src/lib/scheduling.test.ts`
- Create: `src/test/screeningFixtures.ts`
- Test: `src/lib/api.screening.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `postProfessional`, `ATTENDANCE_BASE`, `AUTHORING_BASE`, `PRODUCTION_BASE`, `SchedulingPriority`, `Sex`, `RequestRow` (todos já em `api.ts`).
- Produces:
  - tipos: `ScreeningColor`, `ScreeningDestination`, `ScreeningStatus`, `ScreeningScope`, `GlucoseMoment`, `VitalSigns`, `Ciap2Ref`, `MatchedRule`, `ScreeningRevision`, `Screening`, `ScreeningQueueItem`, `QueueScreening`, `ScreeningSuggestion`, `ScreeningSuggestInput`, `ScreeningRevisionInput`, `ScreeningScheduleInput`, `ScreeningReferralInput`, `ScreeningCompleteInput`, `SimulateScreeningInput`, `SimulateScreeningResult`, `GenerationFailure`, `CallResult`;
  - `HealthUnit.screening_scope?: ScreeningScope`; `QueueRow.screening?: QueueScreening | null`; `RequestRow.kind` com `"screening"` e `RequestRow.origin` com `"screening"`;
  - funções: `listScreeningQueue(unitId): Promise<ScreeningQueueItem[]>`, `startScreening(attendanceId): Promise<Screening>`, `abandonScreening(id): Promise<Screening>`, `suggestScreening(input: ScreeningSuggestInput): Promise<ScreeningSuggestion>`, `completeScreening(id, input: ScreeningCompleteInput): Promise<Screening>`, `reassessScreening(id, input: ScreeningRevisionInput): Promise<Screening>`, `getScreening(id): Promise<Screening>`, `searchCiap2(q): Promise<Ciap2Ref[]>`, `simulateScreening(input): Promise<SimulateScreeningResult>`, `listGenerationFailures(): Promise<GenerationFailure[]>`, `retryGenerationFailure(id): Promise<GenerationFailure>`; `updateUnit(id, name, kind, address, screeningScope?: ScreeningScope)`; `callAttendance`/`callNext` devolvem `CallResult`;
  - fixtures: `NOW18`, `revision()`, `screening()`, `queueItem()`, `suggestion()`;
  - `requestKindLabel({ kind: "screening" })` → `"Acolhimento"`.

- [ ] **Step 1: Write the failing test**

Fixtures (usadas nesta e nas próximas tasks):

```ts
// Dados comuns aos testes do módulo 18 (acolhimento). Relógio dos testes:
// quarta, 2026-10-07 10:00 em São Paulo (-03:00).
import type { Screening, ScreeningQueueItem, ScreeningRevision, ScreeningSuggestion } from "../lib/api";

export const NOW18 = "2026-10-07T10:00:00-03:00";

export function revision(over: Partial<ScreeningRevision> = {}): ScreeningRevision {
  return {
    id: "rv1", created_at: "2026-10-07T09:40:00-03:00", by: { id: "us1", name: "Enf. Lúcia Prado" },
    ciap2: { code: "K86", label: "Hipertensão sem complicações" }, complaint_note: "cefaleia desde ontem",
    vitals: { systolic: 185, diastolic: 110, heart_rate: 88, weight_kg: 80, height_cm: 170, bmi: 27.7 },
    alerts: [ "systolic_high", "diastolic_high" ],
    suggested_color: "red", final_color: "red", color_change_reason: null,
    matched_rules: [ { index: 0, text: "pressão sistólica a partir de 180 mmHg" } ],
    ...over
  };
}

export function screening(over: Partial<Screening> = {}): Screening {
  return {
    id: "sc1", attendance_id: "a1", status: "in_progress", started_at: "2026-10-07T09:35:00-03:00", completed_at: null,
    destination: null, orientation_note: null, appointment_request_id: null, current_revision: null, revisions_count: 0,
    ...over
  };
}

export function queueItem(over: Partial<ScreeningQueueItem> = {}): ScreeningQueueItem {
  return {
    attendance_id: "a1", citizen: { id: "c1", cpf_masked: "***.982.247-**" }, checked_in_at: "2026-10-07T09:20:00-03:00",
    triage_priority: 2, screening: null, ...over
  };
}

export function suggestion(over: Partial<ScreeningSuggestion> = {}): ScreeningSuggestion {
  return {
    suggested_color: "red", matched_rules: [ { index: 0, text: "pressão sistólica a partir de 180 mmHg" } ],
    alerts: [ "systolic_high", "diastolic_high" ], bmi: null, ...over
  };
}
```

Teste do cliente:

```ts
// src/lib/api.screening.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import {
  abandonScreening, callAttendance, completeScreening, getScreening, listGenerationFailures, listScreeningQueue,
  listUnitQueue, reassessScreening, retryGenerationFailure, searchCiap2, simulateScreening, startScreening,
  suggestScreening, updateUnit
} from "./api";
import { EMPTY_ADDRESS } from "./unitAddress";
import { queueItem, revision, screening, suggestion } from "../test/screeningFixtures";

afterEach(() => vi.unstubAllGlobals());

function stub(body: unknown, status = 200) {
  const fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
    new Response(body === undefined ? null : JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fn);
  return fn;
}
const call = (fn: ReturnType<typeof stub>, i = 0) => fn.mock.calls[i] as [ string, RequestInit ];
const sent = (fn: ReturnType<typeof stub>, i = 0) => JSON.parse(call(fn, i)[1].body as string);

describe("cliente do módulo 18 — escuta", () => {
  it("fila do acolhimento desembrulha { items }", async () => {
    const fn = stub({ items: [ queueItem() ] });
    expect((await listScreeningQueue("u/1"))[0].attendance_id).toBe("a1");
    expect(call(fn)[0]).toBe("/attendance/units/u%2F1/screening_queue");
    expect(call(fn)[1].credentials).toBe("include");
  });

  it("iniciar, abandonar e ler devolvem a escuta pura", async () => {
    let fn = stub(screening());
    expect((await startScreening("a1")).status).toBe("in_progress");
    expect(call(fn)[0]).toBe("/attendance/attendances/a1/screening");
    expect(call(fn)[1].method).toBe("POST");

    fn = stub(screening({ status: "abandoned" }));
    expect((await abandonScreening("sc1")).status).toBe("abandoned");
    expect(call(fn)[0]).toBe("/attendance/screenings/sc1/abandon");

    fn = stub(screening({ status: "completed", revisions: [ revision() ] }));
    expect((await getScreening("sc1")).revisions).toHaveLength(1);
    expect(call(fn)[0]).toBe("/attendance/screenings/sc1");
    expect(call(fn)[1].method).toBeUndefined();
  });

  it("sugestão manda queixa, sinais e o atendimento; nada vai na URL", async () => {
    const fn = stub(suggestion());
    const out = await suggestScreening({ ciap2_code: "K86", vitals: { systolic: 185, diastolic: 110 }, attendance_id: "a1" });
    expect(out.suggested_color).toBe("red");
    expect(call(fn)[0]).toBe("/attendance/screenings/suggest");
    expect(sent(fn)).toEqual({ ciap2_code: "K86", vitals: { systolic: 185, diastolic: 110 }, attendance_id: "a1" });
  });

  it("concluir e reavaliar mandam o corpo inteiro", async () => {
    let fn = stub(screening({ status: "completed", destination: "schedule" }));
    await completeScreening("sc1", {
      ciap2_code: "K86", vitals: { systolic: 150, diastolic: 95 }, final_color: "green", color_change_reason: "sem sintoma agudo agora",
      destination: "schedule", schedule: { appointment_type_key: "consulta_medica", priority: "routine", due_in_days: 15 }
    });
    expect(call(fn)[0]).toBe("/attendance/screenings/sc1/complete");
    expect(sent(fn).schedule).toEqual({ appointment_type_key: "consulta_medica", priority: "routine", due_in_days: 15 });
    expect(sent(fn).color_change_reason).toBe("sem sintoma agudo agora");

    fn = stub(screening({ status: "completed", destination: "same_day", revisions_count: 2 }));
    await reassessScreening("sc1", { ciap2_code: "K86", vitals: { systolic: 170 }, final_color: "yellow" });
    expect(call(fn)[0]).toBe("/attendance/screenings/sc1/reassess");
    expect(sent(fn)).toEqual({ ciap2_code: "K86", vitals: { systolic: 170 }, final_color: "yellow" });
  });

  it("CIAP-2: busca pelo corpo e desembrulha { items }", async () => {
    const fn = stub({ items: [ { code: "K86", label: "Hipertensão sem complicações" } ] });
    expect(await searchCiap2("pressão alta")).toEqual([ { code: "K86", label: "Hipertensão sem complicações" } ]);
    expect(call(fn)[0]).toBe("/attendance/ciap2/search");
    expect(sent(fn)).toEqual({ q: "pressão alta" });
  });

  it("simulador do editor manda a definição do rascunho", async () => {
    const fn = stub({ ...suggestion(), errors: [], warnings: [] });
    const out = await simulateScreening({
      definition: { name: "acolhimento", kind: "screening" }, ciap2_code: null, vitals: { spo2: 88 }, profile: { age: 70, sex: "female" }
    });
    expect(out.errors).toEqual([]);
    expect(call(fn)[0]).toBe("/authoring/protocols/simulate_screening");
    expect(sent(fn).profile).toEqual({ age: 70, sex: "female" });
  });
});

describe("cliente do módulo 18 — fila, chamada e unidade", () => {
  it("fila do profissional traz o bloco da escuta", async () => {
    stub({ waiting: [ { id: "a1", screening: { id: "sc1", color: "red", destination: "same_day", waited_minutes: 12 } } ], in_care: [] });
    const out = await listUnitQueue("u1");
    expect(out.waiting[0].screening?.color).toBe("red");
  });

  it("a chamada devolve a escuta junto do atendimento", async () => {
    stub({ attendance: { id: "a1" }, screening: screening({ status: "completed" }) });
    const out = await callAttendance("a1", "u1");
    expect(out.screening?.id).toBe("sc1");
  });

  it("updateUnit só manda screening_scope quando recebe", async () => {
    let fn = stub({ unit: { id: "u1" } });
    await updateUnit("u1", "UBS Centro", "ubs", EMPTY_ADDRESS, "all");
    expect(sent(fn).screening_scope).toBe("all");
    fn = stub({ unit: { id: "u1" } });
    await updateUnit("u1", "UBS Centro", "ubs", EMPTY_ADDRESS);
    expect("screening_scope" in sent(fn)).toBe(false);
  });
});

describe("cliente do módulo 18 — produção", () => {
  it("fichas não geradas: só as não resolvidas; gerar de novo é POST", async () => {
    let fn = stub({ items: [ { id: "g1", source_type: "Screening", source_id: "sc1", attendance_id: "a1",
      reason_codes: [ "unit_without_cnes" ], created_at: "2026-10-07T09:00:00Z", resolved_at: null } ] });
    expect((await listGenerationFailures())[0].reason_codes).toEqual([ "unit_without_cnes" ]);
    expect(call(fn)[0]).toBe("/production/generation_failures?resolved=false");

    fn = stub({ id: "g1", source_type: "Screening", source_id: "sc1", attendance_id: "a1", reason_codes: [],
      created_at: "2026-10-07T09:00:00Z", resolved_at: "2026-10-07T10:00:00Z" });
    expect((await retryGenerationFailure("g1")).resolved_at).not.toBeNull();
    expect(call(fn)[0]).toBe("/production/generation_failures/g1/retry");
    expect(call(fn)[1].method).toBe("POST");
  });
});
```

E o rótulo do pedido nascido no acolhimento, em `src/lib/scheduling.test.ts`:

```diff
--- a/src/lib/scheduling.test.ts
+++ b/src/lib/scheduling.test.ts
@@ -200,6 +200,10 @@ describe("rótulo do pedido", () => {
     expect(requestKindLabel({ kind: "referral", origin_unit_name: null })).toBe("Encaminhamento");
     expect(requestKindLabel({ kind: "triage", origin_unit_name: null })).toBe("Triagem");
   });
+
+  it("pedido nascido no acolhimento (módulo 18)", () => {
+    expect(requestKindLabel({ kind: "screening", origin_unit_name: null })).toBe("Acolhimento");
+  });
 });
 
 describe("estado do horário", () => {
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/lib/api.screening.test.ts src/lib/scheduling.test.ts`
Expected: FAIL — `listScreeningQueue is not a function` (e as outras funções novas), e `expected 'Encaminhamento' to be 'Acolhimento'`.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`:

```diff
--- a/src/lib/api.ts
+++ b/src/lib/api.ts
@@ -461,7 +461,8 @@ export interface UnitAddress {
   neighborhood_id: string | null;
 }
 
-export interface HealthUnit extends Partial<UnitAddress> { id: string; name: string; kind: string }
+// `screening_scope` (módulo 18; contratos §5): ausente em api anterior = `walk_in`.
+export interface HealthUnit extends Partial<UnitAddress> { id: string; name: string; kind: string; screening_scope?: ScreeningScope }
 export interface HealthUnitRow extends HealthUnit {
   active: boolean;
   // O que ainda prende a unidade (api#29): pedidos vivos e horários marcados.
@@ -496,9 +497,12 @@ export async function createUnit(name: string, kind: string, address: UnitAddres
   return payload.unit;
 }
 
-export async function updateUnit(id: string, name: string, kind: string, address: UnitAddress): Promise<HealthUnitRow> {
+export async function updateUnit(
+  id: string, name: string, kind: string, address: UnitAddress, screeningScope?: ScreeningScope
+): Promise<HealthUnitRow> {
+  const body = screeningScope ? { name, kind, ...address, screening_scope: screeningScope } : { name, kind, ...address };
   const payload = await jsonFetch<{ unit: HealthUnitRow }>(`${ATTENDANCE_BASE}/units/${encodeURIComponent(id)}`, {
-    method: "POST", body: JSON.stringify({ name, kind, ...address })
+    method: "POST", body: JSON.stringify(body)
   });
   return payload.unit;
 }
@@ -537,6 +541,8 @@ export interface QueueRow {
   // triagem deste atendimento (sem triagem, o bairro atual do cidadão).
   // Opcional: a API anterior ao módulo 11 não manda.
   reference_unit_ids?: string[];
+  // Módulo 18 (contratos §4): cor, destino e espera da escuta; null sem escuta.
+  screening?: QueueScreening | null;
 }
 
 export interface AppointmentRequestSummary { id: string; kind: string; target_unit_name: string; status: string }
@@ -581,13 +587,16 @@ export async function listUnitQueue(unitId: string): Promise<{ waiting: QueueRow
   return jsonFetch(`${ATTENDANCE_BASE}/units/${encodeURIComponent(unitId)}/queue`);
 }
 
-export async function callAttendance(id: string, healthUnitId: string): Promise<{ attendance: Attendance }> {
+// Módulo 18 (contratos §4): a chamada devolve também a escuta, para o profissional.
+export interface CallResult { attendance: Attendance; screening?: Screening | null }
+
+export async function callAttendance(id: string, healthUnitId: string): Promise<CallResult> {
   return jsonFetch(`${ATTENDANCE_BASE}/attendances/${encodeURIComponent(id)}/call`, {
     method: "POST", body: JSON.stringify({ health_unit_id: healthUnitId })
   });
 }
 
-export async function callNext(unitId: string): Promise<{ attendance: Attendance }> {
+export async function callNext(unitId: string): Promise<CallResult> {
   return jsonFetch(`${ATTENDANCE_BASE}/units/${encodeURIComponent(unitId)}/call_next`, { method: "POST", body: "{}" });
 }
 
@@ -609,7 +618,7 @@ export async function closeAttendance(
 // `priority` é a prioridade do pedido (rotina/prioritária); a da triagem é
 // `triage_priority`.
 export interface RequestRow {
-  id: string; kind: "return" | "referral" | "triage"; origin: "attendance" | "triage";
+  id: string; kind: "return" | "referral" | "triage" | "screening"; origin: "attendance" | "triage" | "screening";
   origin_unit_name: string | null; created_at: string; cpf_masked: string; note: string | null;
   reopened_reason: "expired" | "no_show" | null;
   appointment_type_key: string; appointment_type_name: string; priority: SchedulingPriority; due_on: string;
@@ -1413,3 +1422,125 @@ export async function bookAppointment(requestId: string, unitId: string, input:
 export function getUnitAgenda(unitId: string, date: string): Promise<UnitAgenda> {
   return jsonFetch(`${ATTENDANCE_BASE}/units/${professionalId(unitId)}/agenda?date=${professionalId(date)}`);
 }
+
+// ─── Acolhimento (módulo 18, ADR 0030; contratos 2026-10-07 §2–§6) ──────────
+// Texto livre (queixa, justificativa, orientação) só no corpo de POST, nunca
+// em URL. A recepção não chama nada daqui: ela só lê o bloco `screening` da fila.
+
+export type ScreeningColor = "red" | "yellow" | "green" | "blue";
+export type ScreeningDestination = "same_day" | "schedule" | "oriented" | "referred";
+export type ScreeningStatus = "in_progress" | "completed" | "abandoned";
+export type ScreeningScope = "walk_in" | "all";
+export type GlucoseMoment = "fasting" | "postprandial" | "random";
+
+export interface VitalSigns {
+  systolic?: number; diastolic?: number; heart_rate?: number; respiratory_rate?: number; temperature_c?: number;
+  spo2?: number; capillary_glucose?: number; glucose_moment?: GlucoseMoment; weight_kg?: number; height_cm?: number;
+  pain_score?: number;
+  // Só nas respostas (calculado pela API).
+  bmi?: number;
+}
+export interface Ciap2Ref { code: string; label: string }
+export interface MatchedRule { index: number; text: string }
+
+export interface ScreeningRevision {
+  id: string; created_at: string; by: { id: string; name: string };
+  ciap2: Ciap2Ref; complaint_note?: string | null; vitals: VitalSigns; alerts: string[];
+  suggested_color: ScreeningColor | null; final_color: ScreeningColor; color_change_reason?: string | null;
+  matched_rules: MatchedRule[];
+}
+export interface Screening {
+  id: string; attendance_id: string; status: ScreeningStatus; started_at: string; completed_at: string | null;
+  destination: ScreeningDestination | null; orientation_note?: string | null; appointment_request_id?: string | null;
+  current_revision: ScreeningRevision | null; revisions_count: number;
+  // Só em GET /attendance/screenings/:id.
+  revisions?: ScreeningRevision[];
+}
+export interface ScreeningQueueItem {
+  attendance_id: string; citizen: { id: string; cpf_masked: string }; checked_in_at: string; triage_priority: number | null;
+  screening: { id: string; status: ScreeningStatus; started_by_name: string } | null;
+}
+// Bloco da fila do profissional (contratos §4). `id` é a divergência D2 do
+// plano do dashboard: sem ele, "Ver escuta" e "Reavaliar" não aparecem.
+export interface QueueScreening { id?: string; color: ScreeningColor; destination: ScreeningDestination; waited_minutes: number }
+
+export interface ScreeningSuggestion { suggested_color: ScreeningColor | null; matched_rules: MatchedRule[]; alerts: string[]; bmi: number | null }
+// `attendance_id` no lugar de `citizen_id`: divergência D3 do plano do dashboard.
+export interface ScreeningSuggestInput { ciap2_code: string; vitals: VitalSigns; attendance_id: string }
+
+export interface ScreeningRevisionInput {
+  ciap2_code: string; complaint_note?: string; vitals: VitalSigns; final_color: ScreeningColor; color_change_reason?: string;
+}
+export interface ScreeningScheduleInput { appointment_type_key: string; priority: SchedulingPriority; due_in_days: number }
+export interface ScreeningReferralInput { referral_unit_id?: string; referral_note?: string }
+export interface ScreeningCompleteInput extends ScreeningRevisionInput {
+  destination: ScreeningDestination; orientation_note?: string; schedule?: ScreeningScheduleInput; referral?: ScreeningReferralInput;
+}
+
+export interface SimulateScreeningInput {
+  definition: unknown; ciap2_code: string | null; vitals: VitalSigns; profile: { age: number; sex: Sex };
+}
+export interface SimulateScreeningResult extends ScreeningSuggestion { errors: string[]; warnings: string[] }
+
+export interface GenerationFailure {
+  id: string; source_type: string; source_id: string; attendance_id: string | null; reason_codes: string[];
+  created_at: string; resolved_at: string | null;
+}
+
+const screeningPath = (id: string, action?: string) =>
+  `${ATTENDANCE_BASE}/screenings/${encodeURIComponent(id)}${action ? `/${action}` : ""}`;
+
+export async function listScreeningQueue(unitId: string): Promise<ScreeningQueueItem[]> {
+  const payload = await jsonFetch<{ items: ScreeningQueueItem[] }>(`${ATTENDANCE_BASE}/units/${encodeURIComponent(unitId)}/screening_queue`);
+  return payload.items;
+}
+
+export function startScreening(attendanceId: string): Promise<Screening> {
+  return jsonFetch(`${ATTENDANCE_BASE}/attendances/${encodeURIComponent(attendanceId)}/screening`, postProfessional({}));
+}
+
+export function abandonScreening(id: string): Promise<Screening> {
+  return jsonFetch(screeningPath(id, "abandon"), postProfessional({}));
+}
+
+// Não grava nada (contratos §3).
+export function suggestScreening(input: ScreeningSuggestInput): Promise<ScreeningSuggestion> {
+  return jsonFetch(`${ATTENDANCE_BASE}/screenings/suggest`, postProfessional(input));
+}
+
+export function completeScreening(id: string, input: ScreeningCompleteInput): Promise<Screening> {
+  return jsonFetch(screeningPath(id, "complete"), postProfessional(input));
+}
+
+export function reassessScreening(id: string, input: ScreeningRevisionInput): Promise<Screening> {
+  return jsonFetch(screeningPath(id, "reassess"), postProfessional(input));
+}
+
+// Abrir a escuta gera a trilha de leitura (`screening.viewed`) no api.
+export function getScreening(id: string): Promise<Screening> {
+  return jsonFetch(screeningPath(id));
+}
+
+// Divergência D1 do plano do dashboard: busca de CIAP-2 por nome ou código,
+// no corpo (o termo pode descrever a queixa) e na terminologia ativa.
+export async function searchCiap2(q: string): Promise<Ciap2Ref[]> {
+  const payload = await jsonFetch<{ items: Ciap2Ref[] }>(`${ATTENDANCE_BASE}/ciap2/search`, postProfessional({ q }));
+  return payload.items;
+}
+
+// Divergência D4 do plano do dashboard: simulador do editor com a definição
+// do rascunho. Como o simulate_offer, definição que falha no gate responde 200
+// com `errors`.
+export function simulateScreening(input: SimulateScreeningInput): Promise<SimulateScreeningResult> {
+  return jsonFetch(`${AUTHORING_BASE}/simulate_screening`, postProfessional(input));
+}
+
+export async function listGenerationFailures(): Promise<GenerationFailure[]> {
+  const payload = await jsonFetch<{ items: GenerationFailure[] }>(`${PRODUCTION_BASE}/generation_failures?resolved=false`);
+  return payload.items;
+}
+
+// Step-up: quem trata 401 mfa_required é o SensitiveAction.
+export function retryGenerationFailure(id: string): Promise<GenerationFailure> {
+  return jsonFetch(`${PRODUCTION_BASE}/generation_failures/${encodeURIComponent(id)}/retry`, postProfessional({}));
+}
```

Em `src/lib/scheduling.ts`:

```diff
--- a/src/lib/scheduling.ts
+++ b/src/lib/scheduling.ts
@@ -2,7 +2,7 @@
 // A validação daqui espelha os 422 do api para a tela avisar antes de enviar;
 // quem garante continua sendo o api.
 import type {
-  AppointmentType, AppointmentView, AvailabilitySlot, BlockKind, BookingKind, PreferredPeriod, RescheduleReasonCode,
+  AppointmentType, AppointmentView, AvailabilitySlot, BlockKind, BookingKind, PreferredPeriod, RequestRow, RescheduleReasonCode,
   ScheduleBlock, ScheduleTemplate, SchedulingPriority
 } from "./api";
 import { cityDateFormat, cityIsoDate, fmtHourMinute } from "./format";
@@ -208,9 +208,10 @@ export function slotsByDay(slots: AvailabilitySlot[]): Map<string, AvailabilityS
   return map;
 }
 
-export function requestKindLabel(row: { kind: "return" | "referral" | "triage"; origin_unit_name: string | null }): string {
+export function requestKindLabel(row: { kind: RequestRow["kind"]; origin_unit_name: string | null }): string {
   if (row.kind === "return") return "Retorno";
   if (row.kind === "triage") return "Triagem";
+  if (row.kind === "screening") return "Acolhimento";
   return row.origin_unit_name ? `Encaminhado de ${row.origin_unit_name}` : "Encaminhamento";
 }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/lib/api.screening.test.ts src/lib/scheduling.test.ts && npx tsc --noEmit`
Expected: PASS (45 testes nos dois arquivos); `tsc` limpo (`Requests`, `UnassignedRequests` e `RequestDetailPanel` aceitam o `kind` novo porque `requestKindLabel` passou a receber `RequestRow["kind"]`).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod18 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 add src/lib/api.ts src/lib/api.screening.test.ts src/test/screeningFixtures.ts src/lib/scheduling.ts src/lib/scheduling.test.ts
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 commit -m "feat: add the screening api client and contract types

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 2: Regras puras do acolhimento e frases das recusas

**Files:**
- Create: `src/lib/screening.ts`
- Modify: `src/lib/attendance.ts` (`MESSAGES`)
- Test: `src/lib/screening.test.ts`

**Interfaces:**
- Consumes: `ApiError` e os tipos da Task 1; `attendanceError` (`src/lib/attendance.ts`); `Tone` (`src/theme/tokens.ts`).
- Produces (`src/lib/screening.ts`): `COLORS`, `COLOR_LABEL`, `COLOR_HINT`, `COLOR_TONE`, `DESTINATIONS`, `DESTINATION_LABEL`, `SCOPE_LABEL`, `GLUCOSE_MOMENTS`, `GLUCOSE_MOMENT_LABEL`, `REASON_MIN` (10), `NOTE_MAX` (500), `DUE_MIN`/`DUE_MAX`; `type VitalKey`, `interface VitalSpec`, `VITALS: VitalSpec[]`; `type VitalsForm`, `type VitalsProblems`, `EMPTY_VITALS_FORM`, `vitalsFormFrom(v: VitalSigns): VitalsForm`, `parseVitals(form): { vitals: VitalSigns; problems: VitalsProblems }`, `bmiOf(weightKg?, heightCm?): number | null`, `alertField(code): VitalKey | null`, `alertLabel(code): string`, `defaultDueDays(color | null): number | null`, `needsColorReason(suggested, final): boolean`, `colorProblem(suggested, final, reason): string | null`, `interface DestinationDraft`, `emptyDestination(color | null): DestinationDraft`, `destinationProblem(d): string | null`, `destinationPayload(d)`, `waitedMinutes(sinceIso, now?): number`, `waitLabel(minutes): string`, `screeningError(err): string`.
- Códigos de alerta (do `Screenings::VitalSigns` do plano do api, Task 6): `systolic_high`, `diastolic_high`, `heart_rate_high`, `heart_rate_low`, `respiratory_rate_high`, `temperature_high`, `spo2_low`, `glucose_low`, `glucose_high`, `pain_severe`. `temperature`, `glucose` e `pain` são apelidos de `temperature_c`, `capillary_glucose` e `pain_score`; código que não casa aparece como veio (Divergência D5).

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/screening.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  EMPTY_VITALS_FORM, alertField, alertLabel, bmiOf, colorProblem, defaultDueDays, destinationPayload, destinationProblem,
  emptyDestination, needsColorReason, parseVitals, screeningError, vitalsFormFrom, waitLabel, waitedMinutes, type VitalsForm
} from "./screening";

const form = (over: Partial<VitalsForm>): VitalsForm => ({ ...EMPTY_VITALS_FORM, ...over });

describe("sinais vitais", () => {
  it("vazio é válido e não manda nada", () => {
    expect(parseVitals(EMPTY_VITALS_FORM)).toEqual({ vitals: {}, problems: {} });
  });

  it.each([
    [ "systolic", "300", 300 ], [ "systolic", "50", 50 ], [ "spo2", "100", 100 ], [ "temperature_c", "37,8", 37.8 ],
    [ "weight_kg", "0,5", 0.5 ], [ "weight_kg", "72,35", 72.35 ], [ "pain_score", "0", 0 ], [ "capillary_glucose", "800", 800 ]
  ] as const)("%s = %s nas bordas é aceito", (key, text, value) => {
    const extra = key === "systolic" ? { diastolic: "40" } : key === "capillary_glucose" ? { glucose_moment: "random" as const } : {};
    const out = parseVitals(form({ [key]: text, ...extra }));
    expect(out.problems).toEqual({});
    expect(out.vitals[key]).toBe(value);
  });

  it.each([
    [ "systolic", "301", "use de 50 a 300 mmHg" ], [ "heart_rate", "19", "use de 20 a 250 bpm" ],
    [ "spo2", "101", "use de 50 a 100 %" ], [ "temperature_c", "45,1", "use de 30 a 45 °C" ],
    [ "weight_kg", "0,4", "use de 0,5 a 400 kg" ], [ "pain_score", "11", "use de 0 a 10" ],
    [ "capillary_glucose", "801", "use de 10 a 800 mg/dL" ],
    [ "heart_rate", "88,5", "use um número inteiro" ], [ "temperature_c", "37,85", "use até 1 casa decimal" ],
    [ "weight_kg", "70,125", "use até 2 casas decimais" ], [ "height_cm", "1,70m", "use só números" ]
  ] as const)("%s = %s é recusado com a frase", (key, text, problem) => {
    const out = parseVitals(form({ [key]: text }));
    expect(out.problems[key]).toBe(problem);
    expect(key in out.vitals).toBe(false);
  });

  it("pressão: as duas juntas, e a diastólica menor", () => {
    expect(parseVitals(form({ systolic: "120" })).problems.bp).toBe("informe a sistólica e a diastólica juntas");
    expect(parseVitals(form({ systolic: "120" })).vitals).toEqual({});
    const out = parseVitals(form({ systolic: "120", diastolic: "120" }));
    expect(out.problems.diastolic).toBe("a diastólica precisa ser menor que a sistólica");
    expect(out.vitals).toEqual({ systolic: 120 });
    expect(parseVitals(form({ systolic: "185", diastolic: "110" })).vitals).toEqual({ systolic: 185, diastolic: 110 });
  });

  it("glicemia pede o momento; momento sozinho não vai", () => {
    expect(parseVitals(form({ capillary_glucose: "250" })).problems.glucose_moment).toBe("informe o momento da glicemia");
    expect(parseVitals(form({ capillary_glucose: "250", glucose_moment: "fasting" })).vitals)
      .toEqual({ capillary_glucose: 250, glucose_moment: "fasting" });
    expect(parseVitals(form({ glucose_moment: "fasting" })).vitals).toEqual({});
  });

  it("revisão volta ao formulário com vírgula decimal", () => {
    const f = vitalsFormFrom({ systolic: 185, diastolic: 110, temperature_c: 37.8, capillary_glucose: 90, glucose_moment: "random", bmi: 27.7 });
    expect(f.systolic).toBe("185");
    expect(f.temperature_c).toBe("37,8");
    expect(f.glucose_moment).toBe("random");
    expect(f.weight_kg).toBe("");
  });
});

describe("IMC e alertas", () => {
  it("IMC com uma casa; sem peso ou altura, nada", () => {
    expect(bmiOf(80, 170)).toBe(27.7);
    expect(bmiOf(80, undefined)).toBeNull();
    expect(bmiOf(undefined, 170)).toBeNull();
  });

  it("alerta aponta o campo e diz acima ou abaixo; código desconhecido aparece como veio", () => {
    expect(alertField("systolic_high")).toBe("systolic");
    expect(alertField("heart_rate_low")).toBe("heart_rate");
    expect(alertField("glucose_low")).toBe("capillary_glucose");
    expect(alertField("temperature_high")).toBe("temperature_c");
    expect(alertField("pain_severe")).toBe("pain_score");
    expect(alertField("pregnancy_flag")).toBeNull();
    expect(alertField("weird_high")).toBeNull();
    expect(alertLabel("spo2_low")).toBe("Saturação (SpO2): abaixo da faixa de alerta");
    expect(alertLabel("temperature_high")).toBe("Temperatura: acima da faixa de alerta");
    expect(alertLabel("pain_severe")).toBe("Dor (0 a 10): acima da faixa de alerta");
    expect(alertLabel("pregnancy_flag")).toBe("pregnancy_flag");
  });
});

describe("cor", () => {
  it("prazo padrão pela cor; vermelho sem padrão", () => {
    expect([ "red", "yellow", "green", "blue" ].map((c) => defaultDueDays(c as "red"))).toEqual([ null, 7, 15, 30 ]);
    expect(defaultDueDays(null)).toBeNull();
  });

  it("justificativa só quando muda a cor sugerida", () => {
    expect(needsColorReason("red", "red")).toBe(false);
    expect(needsColorReason("red", "yellow")).toBe(true);
    expect(needsColorReason(null, "green")).toBe(false);
    expect(colorProblem(null, null, "")).toBe("escolha a cor final");
    expect(colorProblem("red", "yellow", "curta")).toBe("explique por que a cor final é diferente da sugerida (pelo menos 10 caracteres)");
    expect(colorProblem("red", "yellow", "PA confirmada 150/95")).toBeNull();
    expect(colorProblem(null, "green", "")).toBeNull();
  });
});

describe("destino", () => {
  it("cada destino pede o seu campo", () => {
    const base = emptyDestination("green");
    expect(base.dueDays).toBe("15");
    expect(destinationProblem(base)).toBe("escolha o destino");
    expect(destinationProblem({ ...base, destination: "same_day" })).toBeNull();
    expect(destinationProblem({ ...base, destination: "oriented" })).toBe("escreva a orientação dada");
    expect(destinationProblem({ ...base, destination: "oriented", orientationNote: "x".repeat(501) })).toBe("a orientação passa de 500 caracteres");
    expect(destinationProblem({ ...base, destination: "schedule" })).toBe("escolha o tipo de atendimento");
    expect(destinationProblem({ ...base, destination: "schedule", typeKey: "consulta_medica", dueDays: "0" })).toBe("informe o prazo (1 a 365 dias)");
    expect(destinationProblem({ ...base, destination: "referred" })).toBe("informe a unidade de destino ou a descrição do encaminhamento");
    expect(destinationProblem({ ...base, destination: "referred", referralNote: "CAPS" })).toBeNull();
  });

  it("vermelho começa sem prazo", () => {
    expect(emptyDestination("red").dueDays).toBe("");
  });

  it("o corpo leva só o destino escolhido", () => {
    const base = { ...emptyDestination("yellow"), orientationNote: "rascunho", referralNote: "rascunho" };
    expect(destinationPayload({ ...base, destination: "same_day" })).toEqual({ destination: "same_day" });
    expect(destinationPayload({ ...base, destination: "schedule", typeKey: "consulta_medica", priority: "priority" })).toEqual({
      destination: "schedule", schedule: { appointment_type_key: "consulta_medica", priority: "priority", due_in_days: 7 }
    });
    expect(destinationPayload({ ...base, destination: "oriented", orientationNote: " hidratação " }))
      .toEqual({ destination: "oriented", orientation_note: "hidratação" });
    expect(destinationPayload({ ...base, destination: "referred", referralUnitId: "u2", referralNote: "" }))
      .toEqual({ destination: "referred", referral: { referral_unit_id: "u2" } });
  });
});

describe("espera e erros", () => {
  const now = new Date("2026-10-07T10:00:00-03:00");
  it("minutos desde a chegada, nunca negativo", () => {
    expect(waitedMinutes("2026-10-07T09:20:00-03:00", now)).toBe(40);
    expect(waitedMinutes("2026-10-07T10:05:00-03:00", now)).toBe(0);
    expect(waitLabel(40)).toBe("40 min");
    expect(waitLabel(65)).toBe("1 h 05 min");
  });

  it("implausible_vital diz qual campo; o resto passa por attendanceError", () => {
    expect(screeningError(new ApiError(422, { error: "implausible_vital", field: "spo2" }, "x")))
      .toBe("Saturação (SpO2): valor fora do plausível — confira");
    expect(screeningError(new ApiError(422, { error: "implausible_vital" }, "x")))
      .toBe("um sinal vital está fora do plausível — confira os valores");
    expect(screeningError(new ApiError(409, { error: "already_screening" }, "x")))
      .toBe("outra pessoa já começou a escuta deste atendimento — a fila foi atualizada");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/lib/screening.test.ts`
Expected: FAIL — `Failed to resolve import "./screening"`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/screening.ts
// Acolhimento (módulo 18; ADR 0030; spec §3–§4; contratos §2–§4). Regras puras
// da tela: rótulos, sinais vitais (plausibilidade espelhando o 422
// `implausible_vital`, que continua sendo a garantia), IMC, alertas, cor e
// destino. Nenhuma função aqui decide a cor: quem sugere é o protocolo
// assinado, no api.
import { ApiError, type GlucoseMoment, type ScreeningColor, type ScreeningCompleteInput, type ScreeningDestination,
  type ScreeningScope, type SchedulingPriority, type VitalSigns } from "./api";
import { attendanceError } from "./attendance";
import type { Tone } from "../theme/tokens";

export const COLORS: ScreeningColor[] = [ "red", "yellow", "green", "blue" ];
export const COLOR_LABEL: Record<ScreeningColor, string> = { red: "vermelho", yellow: "amarelo", green: "verde", blue: "azul" };
// Caderno de Atenção Básica nº 28.
export const COLOR_HINT: Record<ScreeningColor, string> = {
  red: "atendimento imediato", yellow: "atendimento prioritário", green: "atendimento no dia", blue: "consulta agendada"
};
export const COLOR_TONE: Record<ScreeningColor, Tone> = { red: "down", yellow: "warn", green: "ok", blue: "info" };
export const DESTINATIONS: ScreeningDestination[] = [ "same_day", "schedule", "oriented", "referred" ];
export const DESTINATION_LABEL: Record<ScreeningDestination, string> = {
  same_day: "consulta no dia", schedule: "agendar", oriented: "orientação", referred: "encaminhar"
};
export const SCOPE_LABEL: Record<ScreeningScope, string> = {
  walk_in: "só quem chega sem horário (padrão)", all: "todos os atendimentos"
};
export const GLUCOSE_MOMENTS: GlucoseMoment[] = [ "fasting", "postprandial", "random" ];
export const GLUCOSE_MOMENT_LABEL: Record<GlucoseMoment, string> = { fasting: "em jejum", postprandial: "pós-prandial", random: "casual" };

export const REASON_MIN = 10;
export const NOTE_MAX = 500;
export const DUE_MIN = 1;
export const DUE_MAX = 365;

export type VitalKey =
  | "systolic" | "diastolic" | "heart_rate" | "respiratory_rate" | "temperature_c" | "spo2" | "capillary_glucose"
  | "weight_kg" | "height_cm" | "pain_score";
export interface VitalSpec { key: VitalKey; label: string; unit: string; min: number; max: number; decimals: number }

// Limites de plausibilidade da spec §3.1, com a glicemia até 800 (o LEDI recusa
// acima; Desvio 3 do plano do api). O api é a fonte; aqui só se avisa antes.
export const VITALS: VitalSpec[] = [
  { key: "systolic", label: "Pressão sistólica", unit: "mmHg", min: 50, max: 300, decimals: 0 },
  { key: "diastolic", label: "Pressão diastólica", unit: "mmHg", min: 20, max: 200, decimals: 0 },
  { key: "heart_rate", label: "Frequência cardíaca", unit: "bpm", min: 20, max: 250, decimals: 0 },
  { key: "respiratory_rate", label: "Frequência respiratória", unit: "irpm", min: 4, max: 80, decimals: 0 },
  { key: "temperature_c", label: "Temperatura", unit: "°C", min: 30, max: 45, decimals: 1 },
  { key: "spo2", label: "Saturação (SpO2)", unit: "%", min: 50, max: 100, decimals: 0 },
  { key: "capillary_glucose", label: "Glicemia capilar", unit: "mg/dL", min: 10, max: 800, decimals: 0 },
  { key: "weight_kg", label: "Peso", unit: "kg", min: 0.5, max: 400, decimals: 2 },
  { key: "height_cm", label: "Altura", unit: "cm", min: 30, max: 250, decimals: 0 },
  { key: "pain_score", label: "Dor (0 a 10)", unit: "", min: 0, max: 10, decimals: 0 }
];
const SPEC = new Map(VITALS.map((v) => [ v.key, v ]));

export type VitalsForm = Record<VitalKey, string> & { glucose_moment: GlucoseMoment | "" };
export type VitalsProblems = Partial<Record<VitalKey | "glucose_moment" | "bp", string>>;

export const EMPTY_VITALS_FORM: VitalsForm = {
  systolic: "", diastolic: "", heart_rate: "", respiratory_rate: "", temperature_c: "", spo2: "", capillary_glucose: "",
  weight_kg: "", height_cm: "", pain_score: "", glucose_moment: ""
};

// Para a reavaliação: a revisão corrente preenche o formulário (vírgula decimal).
export function vitalsFormFrom(v: VitalSigns): VitalsForm {
  const form: VitalsForm = { ...EMPTY_VITALS_FORM, glucose_moment: v.glucose_moment ?? "" };
  for (const spec of VITALS) {
    const n = v[spec.key];
    if (typeof n === "number") form[spec.key] = String(n).replace(".", ",");
  }
  return form;
}

function decimalsOf(text: string): number {
  const dot = text.indexOf(".");
  return dot < 0 ? 0 : text.length - dot - 1;
}

// Valor válido entra em `vitals`; campo com problema fica de fora e ganha a frase.
export function parseVitals(form: VitalsForm): { vitals: VitalSigns; problems: VitalsProblems } {
  const vitals: VitalSigns = {};
  const problems: VitalsProblems = {};
  for (const spec of VITALS) {
    const text = form[spec.key].trim().replace(",", ".");
    if (text === "") continue;
    if (!/^\d+(\.\d+)?$/.test(text)) { problems[spec.key] = "use só números"; continue; }
    if (decimalsOf(text) > spec.decimals) {
      problems[spec.key] = spec.decimals === 0 ? "use um número inteiro"
        : spec.decimals === 1 ? "use até 1 casa decimal" : `use até ${spec.decimals} casas decimais`;
      continue;
    }
    const n = Number(text);
    if (n < spec.min || n > spec.max) {
      problems[spec.key] = `use de ${String(spec.min).replace(".", ",")} a ${spec.max}${spec.unit ? ` ${spec.unit}` : ""}`;
      continue;
    }
    vitals[spec.key] = n;
  }
  const hasSys = form.systolic.trim() !== "";
  const hasDia = form.diastolic.trim() !== "";
  if (hasSys !== hasDia) problems.bp = "informe a sistólica e a diastólica juntas";
  if (vitals.systolic !== undefined && vitals.diastolic !== undefined && vitals.diastolic >= vitals.systolic) {
    problems.diastolic = "a diastólica precisa ser menor que a sistólica";
    delete vitals.diastolic;
  }
  if (problems.bp) { delete vitals.systolic; delete vitals.diastolic; }
  if (vitals.capillary_glucose !== undefined) {
    if (form.glucose_moment === "") problems.glucose_moment = "informe o momento da glicemia";
    else vitals.glucose_moment = form.glucose_moment;
  }
  return { vitals, problems };
}

// IMC = peso / altura², uma casa decimal. Sem peso ou altura, nada.
export function bmiOf(weightKg: number | undefined, heightCm: number | undefined): number | null {
  if (!weightKg || !heightCm) return null;
  const m = heightCm / 100;
  return Math.round((weightKg / (m * m)) * 10) / 10;
}

// Código de alerta do api (contratos §3; `Screenings::VitalSigns` do plano do
// api): campo + `_high`/`_low`/`_severe`, com `temperature`, `glucose` e `pain`
// abreviados. Código que não casa aparece como veio.
const ALERT_ALIAS: Record<string, VitalKey> = { temperature: "temperature_c", glucose: "capillary_glucose", pain: "pain_score" };
const ALERT_CODE = /^(.+)_(high|low|severe)$/;

export function alertField(code: string): VitalKey | null {
  const m = ALERT_CODE.exec(code);
  if (!m) return null;
  const key = ALERT_ALIAS[m[1]] ?? m[1];
  return SPEC.has(key as VitalKey) ? (key as VitalKey) : null;
}

export function alertLabel(code: string): string {
  const field = alertField(code);
  if (!field) return code;
  return `${SPEC.get(field)!.label}: ${code.endsWith("_low") ? "abaixo" : "acima"} da faixa de alerta`;
}

// Prazo padrão do pedido pela cor (spec §4); vermelho não tem padrão.
export function defaultDueDays(color: ScreeningColor | null): number | null {
  if (color === "yellow") return 7;
  if (color === "green") return 15;
  if (color === "blue") return 30;
  return null;
}

// Sem sugestão (nenhum protocolo ativo ou nenhuma regra casou) não há o que mudar.
export function needsColorReason(suggested: ScreeningColor | null, final: ScreeningColor | null): boolean {
  return suggested !== null && final !== null && final !== suggested;
}

export function colorProblem(suggested: ScreeningColor | null, final: ScreeningColor | null, reason: string): string | null {
  if (final === null) return "escolha a cor final";
  if (needsColorReason(suggested, final) && reason.trim().length < REASON_MIN) {
    return `explique por que a cor final é diferente da sugerida (pelo menos ${REASON_MIN} caracteres)`;
  }
  return null;
}

export interface DestinationDraft {
  destination: ScreeningDestination | "";
  orientationNote: string;
  typeKey: string;
  priority: SchedulingPriority;
  dueDays: string;
  referralUnitId: string;
  referralNote: string;
}

export function emptyDestination(color: ScreeningColor | null): DestinationDraft {
  const due = defaultDueDays(color);
  return { destination: "", orientationNote: "", typeKey: "", priority: "routine", dueDays: due === null ? "" : String(due),
    referralUnitId: "", referralNote: "" };
}

function dueOf(text: string): number | null {
  const t = text.trim();
  if (!/^\d+$/.test(t)) return null;
  const n = Number(t);
  return n >= DUE_MIN && n <= DUE_MAX ? n : null;
}

export function destinationProblem(d: DestinationDraft): string | null {
  if (d.destination === "") return "escolha o destino";
  if (d.destination === "oriented") {
    if (d.orientationNote.trim() === "") return "escreva a orientação dada";
    if (d.orientationNote.length > NOTE_MAX) return `a orientação passa de ${NOTE_MAX} caracteres`;
  }
  if (d.destination === "schedule") {
    if (d.typeKey === "") return "escolha o tipo de atendimento";
    if (dueOf(d.dueDays) === null) return `informe o prazo (${DUE_MIN} a ${DUE_MAX} dias)`;
  }
  if (d.destination === "referred" && d.referralUnitId === "" && d.referralNote.trim() === "") {
    return "informe a unidade de destino ou a descrição do encaminhamento";
  }
  return null;
}

// Só os campos do destino escolhido vão no corpo.
export function destinationPayload(d: DestinationDraft): Pick<ScreeningCompleteInput, "destination" | "orientation_note" | "schedule" | "referral"> {
  const destination = d.destination as ScreeningDestination;
  if (destination === "oriented") return { destination, orientation_note: d.orientationNote.trim() };
  if (destination === "schedule") {
    return { destination, schedule: { appointment_type_key: d.typeKey, priority: d.priority, due_in_days: dueOf(d.dueDays) as number } };
  }
  if (destination === "referred") {
    return { destination, referral: {
      ...(d.referralUnitId ? { referral_unit_id: d.referralUnitId } : {}),
      ...(d.referralNote.trim() ? { referral_note: d.referralNote.trim() } : {})
    } };
  }
  return { destination };
}

export function waitedMinutes(sinceIso: string, now: Date = new Date()): number {
  const t = new Date(sinceIso).getTime();
  if (Number.isNaN(t)) return 0;
  return Math.max(0, Math.floor((now.getTime() - t) / 60_000));
}

export function waitLabel(minutes: number): string {
  if (minutes < 60) return `${minutes} min`;
  const h = Math.floor(minutes / 60);
  const m = minutes % 60;
  return `${h} h ${String(m).padStart(2, "0")} min`;
}

// `implausible_vital` vem com `field` (contratos §3): a frase diz qual.
export function screeningError(err: unknown): string {
  if (err instanceof ApiError) {
    const body = (err.body ?? {}) as { error?: string; field?: string };
    const spec = body.field ? SPEC.get(body.field as VitalKey) : undefined;
    if (body.error === "implausible_vital" && spec) return `${spec.label}: valor fora do plausível — confira`;
  }
  return attendanceError(err);
}
```

Em `src/lib/attendance.ts` (as frases das recusas do contrato §3 e §5; `screeningError` usa `attendanceError` como base):

```diff
--- a/src/lib/attendance.ts
+++ b/src/lib/attendance.ts
@@ -119,7 +119,25 @@ const MESSAGES: Record<string, string> = {
   outside_shift: "o encaixe precisa começar e terminar dentro do turno",
   already_assigned: "este pedido já foi atribuído a uma unidade",
   // Vagas da recepção e Minha agenda (appointment_requests_controller, professional_agenda_controller).
-  invalid_range: "período inválido — confira as datas"
+  invalid_range: "período inválido — confira as datas",
+  // Módulo 18 (contratos §3 e §5). `implausible_vital` com `field` é traduzido em screeningError.
+  already_screening: "outra pessoa já começou a escuta deste atendimento — a fila foi atualizada",
+  not_waiting: "este atendimento não está mais aguardando — a fila foi atualizada",
+  screening_not_required: "esta unidade não faz acolhimento para este atendimento — a fila foi atualizada",
+  cbo_not_allowed: "sua ocupação (CBO) não faz acolhimento",
+  not_in_progress: "esta escuta não está mais em andamento — a fila foi atualizada",
+  invalid_ciap2: "escolha a queixa (CIAP-2) da lista",
+  implausible_vital: "um sinal vital está fora do plausível — confira os valores",
+  bp_incomplete: "informe a sistólica e a diastólica juntas",
+  invalid_color: "escolha uma das quatro cores",
+  color_change_reason_required: "explique por que a cor final é diferente da sugerida",
+  invalid_destination: "escolha o destino",
+  orientation_required: "escreva a orientação dada",
+  invalid_schedule: "confira o tipo, a prioridade e o prazo do agendamento",
+  attendance_not_waiting: "o atendimento não está mais aguardando — a escuta não foi concluída",
+  not_reassessable: "esta escuta não pode mais ser reavaliada — a fila foi atualizada",
+  invalid_screening_scope: "escolha uma das opções de acolhimento",
+  terminology_unavailable: "a CIAP-2 não está disponível agora — tente de novo em instantes"
 };
 
 export function attendanceError(err: unknown): string {
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/lib/screening.test.ts src/lib/attendance.test.ts`
Expected: PASS (o `screening.test.ts` com 32 testes; `attendance.test.ts` continua verde).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod18 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 add src/lib/screening.ts src/lib/screening.test.ts src/lib/attendance.ts
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 commit -m "feat: add screening rules for vitals, colour and destination

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 3: Queixa em CIAP-2 por nome

**Files:**
- Create: `src/modules/attendance/Ciap2Search.tsx`
- Test: `src/modules/attendance/Ciap2Search.test.tsx`

**Interfaces:**
- Consumes: `searchCiap2`, `Ciap2Ref` (Task 1); `screeningError` (Task 2); `useDebouncedValue` (`src/lib/useDebouncedValue.ts`).
- Produces: `Ciap2Search({ value: Ciap2Ref | null; onChange(next: Ciap2Ref | null): void; delayMs?: number })` — rótulo "Queixa (CIAP-2)", botão `"<código> — <nome>"` por resultado, "trocar" volta à busca; `CIAP2_MIN_CHARS = 2`. Chave de cache `[ "ciap2Search", termo ]`. Rota da Divergência D1.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/attendance/Ciap2Search.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState, type ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, searchCiap2: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError, type Ciap2Ref } from "../../lib/api";
import { Ciap2Search } from "./Ciap2Search";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function Harness({ initial = null }: { initial?: Ciap2Ref | null }) {
  const [ value, setValue ] = useState<Ciap2Ref | null>(initial);
  return (
    <>
      <Ciap2Search value={value} onChange={setValue} delayMs={0} />
      <pre data-testid="value">{JSON.stringify(value)}</pre>
    </>
  );
}
function renderIt(initial?: Ciap2Ref | null) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<Harness initial={initial} />, { wrapper });
}
const value = () => JSON.parse(screen.getByTestId("value").textContent ?? "null");

describe("Ciap2Search", () => {
  beforeEach(() => {
    mocked(api.searchCiap2).mockReset();
    mocked(api.searchCiap2).mockResolvedValue([
      { code: "K86", label: "Hipertensão sem complicações" }, { code: "K87", label: "Hipertensão com complicações" }
    ]);
  });

  it("busca por nome e escolhe um código", async () => {
    renderIt();
    fireEvent.change(screen.getByLabelText("Queixa (CIAP-2)"), { target: { value: "hipertensão" } });
    fireEvent.click(await screen.findByRole("button", { name: "K86 — Hipertensão sem complicações" }));
    expect(api.searchCiap2).toHaveBeenCalledWith("hipertensão");
    expect(value()).toEqual({ code: "K86", label: "Hipertensão sem complicações" });
    expect(screen.getByText("Hipertensão sem complicações")).not.toBeNull();
    expect(screen.queryByLabelText("Queixa (CIAP-2)")).toBeNull();
  });

  it("um caractere não busca", async () => {
    renderIt();
    fireEvent.change(screen.getByLabelText("Queixa (CIAP-2)"), { target: { value: "k" } });
    expect(screen.getByText("digite pelo menos 2 caracteres")).not.toBeNull();
    await new Promise((r) => setTimeout(r, 20));
    expect(api.searchCiap2).not.toHaveBeenCalled();
  });

  it("nada encontrado e terminologia indisponível", async () => {
    mocked(api.searchCiap2).mockResolvedValueOnce([]);
    renderIt();
    fireEvent.change(screen.getByLabelText("Queixa (CIAP-2)"), { target: { value: "xyzw" } });
    expect(await screen.findByText("nenhum código encontrado")).not.toBeNull();
    mocked(api.searchCiap2).mockRejectedValueOnce(new ApiError(503, { error: "terminology_unavailable" }, "x"));
    fireEvent.change(screen.getByLabelText("Queixa (CIAP-2)"), { target: { value: "febre" } });
    expect((await screen.findByRole("alert")).textContent).toBe("a CIAP-2 não está disponível agora — tente de novo em instantes");
  });

  it("trocar volta para a busca", async () => {
    renderIt({ code: "K86", label: "Hipertensão sem complicações" });
    fireEvent.click(screen.getByRole("button", { name: "trocar" }));
    expect(value()).toBeNull();
    await waitFor(() => expect(screen.getByLabelText("Queixa (CIAP-2)")).not.toBeNull());
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/modules/attendance/Ciap2Search.test.tsx`
Expected: FAIL — `Failed to resolve import "./Ciap2Search"`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/attendance/Ciap2Search.tsx
// Queixa em CIAP-2 por nome ou código (módulo 18; spec §3, decisão 6). A busca
// vai no corpo de um POST (o termo pode descrever a queixa) e só começa com 2
// caracteres. Escolhido o código, o campo mostra código e nome e só volta à
// busca por "trocar".
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { searchCiap2, type Ciap2Ref } from "../../lib/api";
import { screeningError } from "../../lib/screening";
import { useDebouncedValue } from "../../lib/useDebouncedValue";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";

export const CIAP2_MIN_CHARS = 2;

interface Props { value: Ciap2Ref | null; onChange(next: Ciap2Ref | null): void; delayMs?: number }

export function Ciap2Search({ value, onChange, delayMs = 300 }: Props) {
  const [ text, setText ] = useState("");
  const term = useDebouncedValue(text.trim(), delayMs);
  const enabled = value === null && term.length >= CIAP2_MIN_CHARS;
  const query = useQuery({ queryKey: [ "ciap2Search", term ], queryFn: () => searchCiap2(term), enabled, staleTime: 5 * 60_000 });

  if (value) {
    return (
      <div style={line}>
        <span style={label}>Queixa (CIAP-2)</span>
        <strong className="mono" style={{ fontSize: 12.5 }}>{value.code}</strong>
        <span style={{ fontSize: 12.5 }}>{value.label}</span>
        <button type="button" style={secondaryButtonStyle} onClick={() => { setText(""); onChange(null); }}>trocar</button>
      </div>
    );
  }

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
      <label style={{ ...label, display: "flex", flexDirection: "column", gap: 4 }}>
        Queixa (CIAP-2)
        <input value={text} placeholder="nome ou código, ex.: cefaleia, K86" style={inputStyle}
          onChange={(e) => setText(e.target.value)} />
      </label>
      {text.trim().length > 0 && text.trim().length < CIAP2_MIN_CHARS && <small style={hint}>digite pelo menos {CIAP2_MIN_CHARS} caracteres</small>}
      {enabled && query.isPending && <small className="mono" style={hint}>buscando…</small>}
      {enabled && query.isError && <p role="alert" style={alert}>{screeningError(query.error)}</p>}
      {enabled && query.isSuccess && query.data.length === 0 && <small style={hint}>nenhum código encontrado</small>}
      {enabled && query.isSuccess && query.data.length > 0 && (
        <ul aria-label="códigos CIAP-2" style={{ listStyle: "none", margin: 0, padding: 0, display: "flex", flexDirection: "column", gap: 4 }}>
          {query.data.map((item) => (
            <li key={item.code}>
              <button type="button" style={{ ...secondaryButtonStyle, width: "100%", textAlign: "left" }} onClick={() => onChange(item)}>
                {`${item.code} — ${item.label}`}
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

const line: CSSProperties = { display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" };
const label: CSSProperties = { fontSize: 12, color: "var(--ink2)" };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/modules/attendance/Ciap2Search.test.tsx`
Expected: PASS (4 testes).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod18 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 add src/modules/attendance/Ciap2Search.tsx src/modules/attendance/Ciap2Search.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 commit -m "feat: search CIAP-2 codes by name for the screening complaint

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 4: Sinais vitais com alerta e IMC; cor sugerida e cor final

**Files:**
- Create: `src/modules/attendance/VitalSignsFields.tsx`, `src/modules/attendance/ColorDecision.tsx`
- Test: `src/modules/attendance/VitalSignsFields.test.tsx`, `src/modules/attendance/ColorDecision.test.tsx`

**Interfaces:**
- Consumes: `VITALS`, `GLUCOSE_MOMENTS`, `GLUCOSE_MOMENT_LABEL`, `alertField`, `alertLabel`, `COLORS`, `COLOR_LABEL`, `COLOR_HINT`, `COLOR_TONE`, `needsColorReason`, `VitalsForm`, `VitalsProblems` (Task 2); `ScreeningSuggestion`, `ScreeningColor`, `GlucoseMoment` (Task 1); `Tag`, `FrozenTextNotice`, `fmtNumber`.
- Produces:
  - `VitalSignsFields({ form: VitalsForm; problems: VitalsProblems; alerts: string[]; bmi: number | null; onChange(next: VitalsForm): void })` — fieldset "Sinais vitais"; rótulos `"<Rótulo> (<unidade>)"` (ex.: "Pressão sistólica (mmHg)", "Dor (0 a 10)") e "Momento da glicemia"; problema sob o campo com `role="alert"`; campo com alerta ganha borda `var(--down)`; linha "IMC: <valor>";
  - `type SuggestionState = { kind: "idle" } | { kind: "loading" } | { kind: "ready"; suggestion: ScreeningSuggestion } | { kind: "error"; message: string }`;
  - `ColorDecision({ state: SuggestionState; final: ScreeningColor | null; reason: string; forceReason?: boolean; onFinal(color): void; onReason(text): void })` — fieldset "Classificação de risco"; `role="status"` com a sugestão e a lista "motivo da sugestão"; `radiogroup` "Cor final" (um rádio por cor, o nome acessível contém o rótulo da cor); textarea "Justificativa da mudança de cor" com o `FrozenTextNotice` quando `forceReason` ou a final difere da sugerida.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/attendance/VitalSignsFields.test.tsx
import { afterEach, describe, expect, it } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { useState } from "react";
import { VitalSignsFields } from "./VitalSignsFields";
import { EMPTY_VITALS_FORM, bmiOf, parseVitals, type VitalsForm } from "../../lib/screening";

afterEach(cleanup);

function Harness({ alerts = [] }: { alerts?: string[] }) {
  const [ form, setForm ] = useState<VitalsForm>(EMPTY_VITALS_FORM);
  const { vitals, problems } = parseVitals(form);
  return (
    <>
      <VitalSignsFields form={form} problems={problems} alerts={alerts} bmi={bmiOf(vitals.weight_kg, vitals.height_cm)} onChange={setForm} />
      <pre data-testid="vitals">{JSON.stringify(vitals)}</pre>
    </>
  );
}
const vitals = () => JSON.parse(screen.getByTestId("vitals").textContent ?? "{}");
const type = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

describe("VitalSignsFields", () => {
  it("monta os sinais com vírgula decimal e calcula o IMC", () => {
    render(<Harness />);
    type("Pressão sistólica (mmHg)", "185");
    type("Pressão diastólica (mmHg)", "110");
    type("Temperatura (°C)", "37,8");
    type("Peso (kg)", "80");
    type("Altura (cm)", "170");
    expect(vitals()).toEqual({ systolic: 185, diastolic: 110, temperature_c: 37.8, weight_kg: 80, height_cm: 170 });
    expect(screen.getByText("27,7")).not.toBeNull();
  });

  it("fora do plausível mostra a frase sob o campo e não entra nos sinais", () => {
    render(<Harness />);
    type("Saturação (SpO2) (%)", "120");
    expect(screen.getByRole("alert").textContent).toBe("use de 50 a 100 %");
    expect(vitals()).toEqual({});
  });

  it("pressão pela metade e glicemia sem momento avisam", () => {
    render(<Harness />);
    type("Pressão sistólica (mmHg)", "140");
    type("Glicemia capilar (mg/dL)", "250");
    const alerts = screen.getAllByRole("alert").map((a) => a.textContent);
    expect(alerts).toContain("informe a sistólica e a diastólica juntas");
    expect(alerts).toContain("informe o momento da glicemia");
    fireEvent.change(screen.getByLabelText("Momento da glicemia"), { target: { value: "random" } });
    expect(vitals()).toEqual({ capillary_glucose: 250, glucose_moment: "random" });
  });

  it("alerta do api destaca o campo; código sem campo aparece como veio", () => {
    render(<Harness alerts={[ "systolic_high", "pregnancy_flag" ]} />);
    expect(screen.getByText("Pressão sistólica: acima da faixa de alerta")).not.toBeNull();
    expect(screen.getByText("alertas: pregnancy_flag")).not.toBeNull();
    const field = screen.getByLabelText("Pressão sistólica (mmHg)") as HTMLInputElement;
    expect(field.style.borderColor).toBe("var(--down)");
  });
});
```

```tsx
// src/modules/attendance/ColorDecision.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { ColorDecision, type SuggestionState } from "./ColorDecision";
import type { ScreeningColor } from "../../lib/api";
import { suggestion } from "../../test/screeningFixtures";
import { expectFrozenNotice } from "../../test/frozenNotice";

afterEach(cleanup);

function renderIt(state: SuggestionState, final: ScreeningColor | null, reason = "") {
  const onFinal = vi.fn();
  const onReason = vi.fn();
  render(<ColorDecision state={state} final={final} reason={reason} onFinal={onFinal} onReason={onReason} />);
  return { onFinal, onReason };
}

describe("ColorDecision", () => {
  it("mostra a cor sugerida com o motivo e a cor final marcada", () => {
    renderIt({ kind: "ready", suggestion: suggestion() }, "red");
    expect(screen.getByRole("status").textContent).toContain("Cor sugerida: vermelho · atendimento imediato");
    expect(screen.getByRole("list", { name: "motivo da sugestão" }).textContent).toBe("pressão sistólica a partir de 180 mmHg");
    expect((screen.getByRole("radio", { name: /vermelho/ }) as HTMLInputElement).checked).toBe(true);
    expect(screen.queryByLabelText("Justificativa da mudança de cor")).toBeNull();
  });

  it("mudar da sugerida pede justificativa com o aviso de texto congelado", () => {
    const { onFinal } = renderIt({ kind: "ready", suggestion: suggestion() }, "yellow");
    expectFrozenNotice(screen.getByLabelText("Justificativa da mudança de cor"));
    fireEvent.click(screen.getByRole("radio", { name: /verde/ }));
    expect(onFinal).toHaveBeenCalledWith("green");
  });

  it("forceReason mostra a justificativa mesmo com a cor igual à sugerida", () => {
    render(<ColorDecision state={{ kind: "ready", suggestion: suggestion() }} final="red" reason="" forceReason
      onFinal={vi.fn()} onReason={vi.fn()} />);
    expect(screen.getByLabelText("Justificativa da mudança de cor")).not.toBeNull();
  });

  it("sem sugestão: explica e não pede justificativa", () => {
    renderIt({ kind: "ready", suggestion: suggestion({ suggested_color: null, matched_rules: [] }) }, "green");
    expect(screen.getByRole("status").textContent)
      .toBe("Sem cor sugerida: nenhuma regra do protocolo de acolhimento casou, ou a cidade não tem protocolo ativo.");
    expect(screen.queryByLabelText("Justificativa da mudança de cor")).toBeNull();
  });

  it("antes da queixa, enquanto calcula e com erro", () => {
    renderIt({ kind: "idle" }, null);
    expect(screen.getByRole("status").textContent).toBe("A cor sugerida aparece quando a queixa estiver escolhida.");
    cleanup();
    renderIt({ kind: "loading" }, null);
    expect(screen.getByRole("status").textContent).toBe("calculando a sugestão…");
    cleanup();
    renderIt({ kind: "error", message: "não foi possível concluir — tente de novo" }, null);
    expect(screen.getByRole("status").textContent).toBe("não foi possível concluir — tente de novo");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/modules/attendance/VitalSignsFields.test.tsx src/modules/attendance/ColorDecision.test.tsx`
Expected: FAIL — `Failed to resolve import "./VitalSignsFields"` e `"./ColorDecision"`.

- [ ] **Step 3: Write minimal implementation**

(O problema e o alerta ficam **fora** do `<label>`: dentro dele, o texto entraria no nome acessível do campo.)

```tsx
// src/modules/attendance/VitalSignsFields.tsx
// Sinais vitais da escuta (módulo 18; spec §3.1): todos opcionais. Problema de
// plausibilidade aparece sob o campo; alerta do api (faixa de alerta) destaca o
// campo em vermelho; o IMC é calculado na hora.
import type { CSSProperties } from "react";
import type { GlucoseMoment } from "../../lib/api";
import { GLUCOSE_MOMENTS, GLUCOSE_MOMENT_LABEL, VITALS, alertField, alertLabel, type VitalKey, type VitalsForm,
  type VitalsProblems } from "../../lib/screening";
import { fmtNumber } from "../../lib/format";
import { inputStyle } from "../../components/formStyles";

interface Props {
  form: VitalsForm;
  problems: VitalsProblems;
  alerts: string[];
  bmi: number | null;
  onChange(next: VitalsForm): void;
}

export function VitalSignsFields({ form, problems, alerts, bmi, onChange }: Props) {
  const alertsByField = new Map<VitalKey, string>();
  const loose: string[] = [];
  for (const code of alerts) {
    const field = alertField(code);
    if (field) alertsByField.set(field, alertLabel(code));
    else loose.push(code);
  }
  const set = (key: keyof VitalsForm, value: string) => onChange({ ...form, [key]: value });

  return (
    <fieldset aria-label="Sinais vitais" style={box}>
      <legend style={legend}>Sinais vitais</legend>
      <div style={grid}>
        {VITALS.map((spec) => {
          const alert = alertsByField.get(spec.key);
          const problem = problems[spec.key];
          return (
            <div key={spec.key} style={cell}>
              <label style={label}>
                {spec.unit ? `${spec.label} (${spec.unit})` : spec.label}
                <input value={form[spec.key]} inputMode="decimal" aria-invalid={problem ? true : undefined}
                  style={alert ? { ...inputStyle, borderColor: "var(--down)", background: "var(--down-bg)" } : inputStyle}
                  onChange={(e) => set(spec.key, e.target.value)} />
              </label>
              {problem && <small role="alert" style={problemStyle}>{problem}</small>}
              {alert && <small style={alertStyle}>{alert}</small>}
            </div>
          );
        })}
        <div style={cell}>
          <label style={label}>
            Momento da glicemia
            <select value={form.glucose_moment} style={inputStyle} onChange={(e) => set("glucose_moment", e.target.value as GlucoseMoment | "")}>
              <option value="">—</option>
              {GLUCOSE_MOMENTS.map((m) => <option key={m} value={m}>{GLUCOSE_MOMENT_LABEL[m]}</option>)}
            </select>
          </label>
          {problems.glucose_moment && <small role="alert" style={problemStyle}>{problems.glucose_moment}</small>}
        </div>
      </div>
      {problems.bp && <small role="alert" style={problemStyle}>{problems.bp}</small>}
      {loose.length > 0 && <small style={alertStyle}>{`alertas: ${loose.join(", ")}`}</small>}
      <p style={{ margin: 0, fontSize: 12.5 }}>
        <span style={{ color: "var(--ink3)" }}>IMC: </span>
        <span className="mono">{bmi === null ? "—" : fmtNumber(bmi)}</span>
      </p>
    </fieldset>
  );
}

const box: CSSProperties = { border: "1px solid var(--rule)", borderRadius: 8, padding: "8px 12px", margin: 0, display: "flex", flexDirection: "column", gap: 8 };
const legend: CSSProperties = { fontSize: 13, fontWeight: 600 };
const grid: CSSProperties = { display: "grid", gridTemplateColumns: "repeat(auto-fill, minmax(170px, 1fr))", gap: 10 };
const cell: CSSProperties = { display: "flex", flexDirection: "column", gap: 4 };
const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const problemStyle: CSSProperties = { fontSize: 11.5, color: "var(--down)" };
const alertStyle: CSSProperties = { fontSize: 11.5, color: "var(--down)", fontWeight: 600 };
```

```tsx
// src/modules/attendance/ColorDecision.tsx
// Cor sugerida e cor final (módulo 18; spec §3.3; ADR 0030): a sugestão vem do
// protocolo assinado `acolhimento`, com o motivo (regras que casaram); a
// decisão é de quem escuta. Cor final diferente da sugerida pede justificativa.
import type { CSSProperties } from "react";
import type { ScreeningColor, ScreeningSuggestion } from "../../lib/api";
import { COLORS, COLOR_HINT, COLOR_LABEL, COLOR_TONE, needsColorReason } from "../../lib/screening";
import { Tag } from "../../components/Tag";
import { FrozenTextNotice } from "../../components/FrozenTextNotice";
import { inputStyle } from "../../components/formStyles";

export type SuggestionState =
  | { kind: "idle" } | { kind: "loading" } | { kind: "ready"; suggestion: ScreeningSuggestion } | { kind: "error"; message: string };

interface Props {
  state: SuggestionState;
  final: ScreeningColor | null;
  reason: string;
  // O api pediu justificativa (422 color_change_reason_required) com uma sugestão
  // que a tela ainda não tinha: o campo aparece mesmo sem diferença local.
  forceReason?: boolean;
  onFinal(color: ScreeningColor): void;
  onReason(text: string): void;
}

export function ColorDecision({ state, final, reason, forceReason = false, onFinal, onReason }: Props) {
  const suggested = state.kind === "ready" ? state.suggestion.suggested_color : null;
  const askReason = forceReason || needsColorReason(suggested, final);

  return (
    <fieldset aria-label="Classificação de risco" style={box}>
      <legend style={legend}>Classificação de risco</legend>
      <div role="status" style={{ display: "flex", flexDirection: "column", gap: 4, fontSize: 12.5 }}>
        {state.kind === "idle" && <span style={hint}>A cor sugerida aparece quando a queixa estiver escolhida.</span>}
        {state.kind === "loading" && <span className="mono" style={hint}>calculando a sugestão…</span>}
        {state.kind === "error" && <span style={{ color: "var(--down)" }}>{state.message}</span>}
        {state.kind === "ready" && suggested === null && (
          <span>Sem cor sugerida: nenhuma regra do protocolo de acolhimento casou, ou a cidade não tem protocolo ativo.</span>
        )}
        {state.kind === "ready" && suggested !== null && (
          <>
            <span>
              Cor sugerida: <Tag tone={COLOR_TONE[suggested]}>{COLOR_LABEL[suggested]}</Tag> · {COLOR_HINT[suggested]}
            </span>
            {state.suggestion.matched_rules.length > 0 && (
              <ul aria-label="motivo da sugestão" style={{ margin: 0, paddingLeft: 18, color: "var(--ink2)" }}>
                {state.suggestion.matched_rules.map((r) => <li key={r.index}>{r.text}</li>)}
              </ul>
            )}
          </>
        )}
      </div>
      <div role="radiogroup" aria-label="Cor final" style={{ display: "flex", gap: 12, flexWrap: "wrap" }}>
        {COLORS.map((c) => (
          <label key={c} style={radio}>
            <input type="radio" name="final-color" checked={final === c} onChange={() => onFinal(c)} />
            <Tag tone={COLOR_TONE[c]}>{COLOR_LABEL[c]}</Tag>
            <span style={hint}>{COLOR_HINT[c]}</span>
          </label>
        ))}
      </div>
      {askReason && (
        <>
          <label style={label}>
            Justificativa da mudança de cor
            <textarea value={reason} style={{ ...inputStyle, minHeight: 56 }} aria-describedby="color-reason-notice"
              onChange={(e) => onReason(e.target.value)} />
          </label>
          <FrozenTextNotice id="color-reason-notice" />
        </>
      )}
    </fieldset>
  );
}

const box: CSSProperties = { border: "1px solid var(--rule)", borderRadius: 8, padding: "8px 12px", margin: 0, display: "flex", flexDirection: "column", gap: 8 };
const legend: CSSProperties = { fontSize: 13, fontWeight: 600 };
const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const radio: CSSProperties = { display: "inline-flex", gap: 6, alignItems: "center", fontSize: 12 };
const hint: CSSProperties = { fontSize: 12, color: "var(--ink3)" };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/modules/attendance/VitalSignsFields.test.tsx src/modules/attendance/ColorDecision.test.tsx`
Expected: PASS (4 + 5 testes).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod18 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 add src/modules/attendance/VitalSignsFields.tsx src/modules/attendance/VitalSignsFields.test.tsx src/modules/attendance/ColorDecision.tsx src/modules/attendance/ColorDecision.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 commit -m "feat: show vital signs with alerts and the suggested risk colour

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 5: Formulário da escuta — concluir com destino e reavaliar

**Files:**
- Create: `src/modules/attendance/ScreeningForm.tsx`
- Test: `src/modules/attendance/ScreeningForm.test.tsx`

**Interfaces:**
- Consumes: `abandonScreening`, `completeScreening`, `reassessScreening`, `suggestScreening`, `errorCode`, tipos (Task 1); `parseVitals`, `vitalsFormFrom`, `bmiOf`, `colorProblem`, `needsColorReason`, `defaultDueDays`, `emptyDestination`, `destinationProblem`, `destinationPayload`, `screeningError`, `DESTINATIONS`, `DESTINATION_LABEL`, `REASON_MIN`, `NOTE_MAX` (Task 2); `Ciap2Search` (Task 3); `VitalSignsFields`, `ColorDecision`, `SuggestionState` (Task 4); `PRIORITY_LABEL` (`src/lib/scheduling.ts`); `useDebouncedValue`.
- Produces: `interface ScreeningFormProps { mode: "complete" | "reassess"; screening: Screening; citizenLabel: string; unit: HealthUnit; units: HealthUnit[]; types: AppointmentType[] | null; suggestDelayMs?: number; onDone(result: Screening): void; onClosed(message: string): void; onCancel(): void }` e `ScreeningForm(props)`. Seção com nome acessível "Escuta inicial" (concluir) ou "Reavaliação"; botões "Concluir escuta"/"Abandonar escuta" ou "Salvar reavaliação"/"Cancelar"; select "Destino" dentro do fieldset "Destino do acolhimento"; campos "Tipo de atendimento", "Prioridade", "Prazo (dias)", "Orientação dada", "Unidade de destino", "Descrição do encaminhamento", "Queixa em texto (opcional)". `onClosed` recebe a frase quando o api encerra o formulário (`not_in_progress`, `attendance_not_waiting`, `not_reassessable`) ou depois de abandonar.
- Comportamento: a cor sugerida vem do `suggest` a cada mudança de queixa ou sinais (espera `suggestDelayMs`, padrão 400 ms), com número de sequência para descartar resposta velha; enquanto a sugestão está sendo calculada, o botão de concluir fica travado; a cor final segue a sugerida até a pessoa escolher outra; o prazo padrão segue a cor final até a pessoa mexer nele; 422 `color_change_reason_required` liga `forceReason` e pede sugestão nova.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/attendance/ScreeningForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, searchCiap2: vi.fn(), suggestScreening: vi.fn(), completeScreening: vi.fn(), reassessScreening: vi.fn(),
    abandonScreening: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError, type ScreeningSuggestion } from "../../lib/api";
import { ScreeningForm, type ScreeningFormProps } from "./ScreeningForm";
import { revision, screening, suggestion } from "../../test/screeningFixtures";
import { TYPES } from "../../test/schedulingFixtures";
import { expectFrozenNotice } from "../../test/frozenNotice";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };
const other = { id: "u2", name: "UPA Norte", kind: "upa" };

function renderForm(over: Partial<ScreeningFormProps> = {}) {
  const props: ScreeningFormProps = {
    mode: "complete", screening: screening(), citizenLabel: "***.982.247-**", unit, units: [ unit, other ], types: TYPES,
    suggestDelayMs: 0, onDone: vi.fn(), onClosed: vi.fn(), onCancel: vi.fn(), ...over
  };
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<ScreeningForm {...props} />, { wrapper });
  return props;
}
const type = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });
const conclude = () => screen.getByRole("button", { name: "Concluir escuta" }) as HTMLButtonElement;

async function pickK86() {
  type("Queixa (CIAP-2)", "hipertensão");
  fireEvent.click(await screen.findByRole("button", { name: "K86 — Hipertensão sem complicações" }));
}
async function fillRedSameDay() {
  await pickK86();
  type("Pressão sistólica (mmHg)", "185");
  type("Pressão diastólica (mmHg)", "110");
  await screen.findByText("Pressão sistólica: acima da faixa de alerta");
  type("Destino", "same_day");
}

describe("ScreeningForm — concluir", () => {
  beforeEach(() => {
    for (const fn of [ api.searchCiap2, api.suggestScreening, api.completeScreening, api.reassessScreening, api.abandonScreening ]) {
      mocked(fn).mockReset();
    }
    mocked(api.searchCiap2).mockResolvedValue([ { code: "K86", label: "Hipertensão sem complicações" } ]);
    mocked(api.suggestScreening).mockResolvedValue(suggestion());
    mocked(api.completeScreening).mockResolvedValue(screening({ status: "completed", destination: "same_day" }));
  });

  it("queixa e pressão pedem a sugestão; a cor sugerida vira a final e conclui no dia", async () => {
    const props = renderForm();
    expect(conclude().disabled).toBe(true);
    await fillRedSameDay();
    expect(api.suggestScreening).toHaveBeenLastCalledWith({ ciap2_code: "K86", vitals: { systolic: 185, diastolic: 110 }, attendance_id: "a1" });
    expect((screen.getByRole("radio", { name: /vermelho/ }) as HTMLInputElement).checked).toBe(true);
    await waitFor(() => expect(conclude().disabled).toBe(false));
    fireEvent.click(conclude());
    await waitFor(() => expect(api.completeScreening).toHaveBeenCalledWith("sc1", {
      ciap2_code: "K86", vitals: { systolic: 185, diastolic: 110 }, final_color: "red", destination: "same_day"
    }));
    expect(props.onDone).toHaveBeenCalled();
  });

  it("mudar a cor pede justificativa, que vai no corpo", async () => {
    renderForm();
    await fillRedSameDay();
    fireEvent.click(screen.getByRole("radio", { name: /amarelo/ }));
    expect(conclude().disabled).toBe(true);
    const reason = screen.getByLabelText("Justificativa da mudança de cor");
    expectFrozenNotice(reason);
    fireEvent.change(reason, { target: { value: "PA confirmada em repouso 150/95" } });
    await waitFor(() => expect(conclude().disabled).toBe(false));
    fireEvent.click(conclude());
    await waitFor(() => expect(api.completeScreening).toHaveBeenCalledWith("sc1", expect.objectContaining({
      final_color: "yellow", color_change_reason: "PA confirmada em repouso 150/95"
    })));
  });

  it("agendar: prazo padrão pela cor final, e vermelho começa sem prazo", async () => {
    renderForm();
    await fillRedSameDay();
    type("Destino", "schedule");
    expect((screen.getByLabelText("Prazo (dias)") as HTMLInputElement).value).toBe("");
    fireEvent.click(screen.getByRole("radio", { name: /verde/ }));
    expect((screen.getByLabelText("Prazo (dias)") as HTMLInputElement).value).toBe("15");
    type("Justificativa da mudança de cor", "sem sinal de gravidade agora");
    type("Tipo de atendimento", "consulta_medica");
    await waitFor(() => expect(conclude().disabled).toBe(false));
    fireEvent.click(conclude());
    await waitFor(() => expect(api.completeScreening).toHaveBeenCalledWith("sc1", expect.objectContaining({
      destination: "schedule", schedule: { appointment_type_key: "consulta_medica", priority: "routine", due_in_days: 15 }
    })));
  });

  it("orientação exige o texto, com o aviso de texto congelado", async () => {
    renderForm();
    await fillRedSameDay();
    type("Destino", "oriented");
    expect(conclude().disabled).toBe(true);
    expect(screen.getByText("escreva a orientação dada")).not.toBeNull();
    expectFrozenNotice(screen.getByLabelText("Orientação dada"));
    type("Orientação dada", "hidratação e retorno se piorar");
    await waitFor(() => expect(conclude().disabled).toBe(false));
  });

  it("resposta velha da sugestão é descartada", async () => {
    let releaseOld: (s: ScreeningSuggestion) => void = () => {};
    mocked(api.suggestScreening)
      .mockImplementationOnce(() => new Promise((resolve) => { releaseOld = resolve; }))
      .mockResolvedValueOnce(suggestion({ suggested_color: "yellow", matched_rules: [ { index: 1, text: "temperatura a partir de 39 °C" } ], alerts: [] }));
    renderForm();
    await pickK86();
    await waitFor(() => expect(api.suggestScreening).toHaveBeenCalledTimes(1));
    type("Temperatura (°C)", "39,2");
    expect(await screen.findByText("temperatura a partir de 39 °C")).not.toBeNull();
    releaseOld(suggestion());
    await new Promise((r) => setTimeout(r, 10));
    expect(screen.getByRole("status").textContent).toContain("amarelo");
    expect(screen.queryByText("pressão sistólica a partir de 180 mmHg")).toBeNull();
  });

  it("422 color_change_reason_required mostra a justificativa e pede nova sugestão", async () => {
    mocked(api.completeScreening).mockRejectedValueOnce(new ApiError(422, { error: "color_change_reason_required" }, "x"));
    renderForm();
    await fillRedSameDay();
    const calls = mocked(api.suggestScreening).mock.calls.length;
    await waitFor(() => expect(conclude().disabled).toBe(false));
    fireEvent.click(conclude());
    expect(await screen.findByLabelText("Justificativa da mudança de cor")).not.toBeNull();
    expect(screen.getByRole("alert").textContent).toBe("explique por que a cor final é diferente da sugerida");
    await waitFor(() => expect(mocked(api.suggestScreening).mock.calls.length).toBeGreaterThan(calls));
  });

  it("atendimento que saiu da espera fecha o formulário com a frase", async () => {
    mocked(api.completeScreening).mockRejectedValueOnce(new ApiError(409, { error: "attendance_not_waiting" }, "x"));
    const props = renderForm();
    await fillRedSameDay();
    await waitFor(() => expect(conclude().disabled).toBe(false));
    fireEvent.click(conclude());
    await waitFor(() => expect(props.onClosed).toHaveBeenCalledWith("o atendimento não está mais aguardando — a escuta não foi concluída"));
  });

  it("abandonar devolve o atendimento à fila do acolhimento", async () => {
    mocked(api.abandonScreening).mockResolvedValue(screening({ status: "abandoned" }));
    const props = renderForm();
    fireEvent.click(screen.getByRole("button", { name: "Abandonar escuta" }));
    await waitFor(() => expect(api.abandonScreening).toHaveBeenCalledWith("sc1"));
    expect(props.onClosed).toHaveBeenCalledWith("Escuta abandonada: o atendimento voltou para a fila do acolhimento.");
  });
});

describe("ScreeningForm — reavaliar", () => {
  beforeEach(() => {
    for (const fn of [ api.searchCiap2, api.suggestScreening, api.completeScreening, api.reassessScreening, api.abandonScreening ]) {
      mocked(fn).mockReset();
    }
    mocked(api.suggestScreening).mockResolvedValue(suggestion({ suggested_color: "yellow", alerts: [] }));
    mocked(api.reassessScreening).mockResolvedValue(screening({ status: "completed", destination: "same_day", revisions_count: 2 }));
  });
  const completed = () => screening({ status: "completed", destination: "same_day", current_revision: revision(), revisions_count: 1 });

  it("começa da revisão corrente, sem destino, e salva só a revisão", async () => {
    const props = renderForm({ mode: "reassess", screening: completed() });
    expect(screen.getByText("Hipertensão sem complicações")).not.toBeNull();
    expect((screen.getByLabelText("Pressão sistólica (mmHg)") as HTMLInputElement).value).toBe("185");
    expect(screen.queryByLabelText("Destino")).toBeNull();
    type("Pressão sistólica (mmHg)", "160");
    type("Pressão diastólica (mmHg)", "100");
    const save = screen.getByRole("button", { name: "Salvar reavaliação" }) as HTMLButtonElement;
    await waitFor(() => expect(save.disabled).toBe(false));
    fireEvent.click(save);
    await waitFor(() => expect(api.reassessScreening).toHaveBeenCalledWith("sc1", {
      ciap2_code: "K86", complaint_note: "cefaleia desde ontem",
      vitals: { systolic: 160, diastolic: 100, heart_rate: 88, weight_kg: 80, height_cm: 170 }, final_color: "yellow"
    }));
    expect(props.onDone).toHaveBeenCalled();
  });

  it("not_reassessable fecha e avisa", async () => {
    mocked(api.reassessScreening).mockRejectedValueOnce(new ApiError(409, { error: "not_reassessable" }, "x"));
    const props = renderForm({ mode: "reassess", screening: completed() });
    const save = screen.getByRole("button", { name: "Salvar reavaliação" }) as HTMLButtonElement;
    await waitFor(() => expect(save.disabled).toBe(false));
    fireEvent.click(save);
    await waitFor(() => expect(props.onClosed).toHaveBeenCalledWith("esta escuta não pode mais ser reavaliada — a fila foi atualizada"));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/modules/attendance/ScreeningForm.test.tsx`
Expected: FAIL — `Failed to resolve import "./ScreeningForm"`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/attendance/ScreeningForm.tsx
// Escuta inicial (módulo 18; spec §3–§4; contratos §3). Um formulário para
// concluir (com destino) e para reavaliar (só queixa, sinais e cor). A cor
// sugerida é pedida ao api (`suggest`, não grava) a cada mudança de queixa ou
// sinais, com espera curta; resposta velha é descartada. Enquanto a sugestão
// está sendo calculada, não se conclui.
import { useEffect, useRef, useState, type CSSProperties } from "react";
import {
  abandonScreening, completeScreening, errorCode, reassessScreening, suggestScreening,
  type AppointmentType, type Ciap2Ref, type HealthUnit, type Screening, type ScreeningColor, type ScreeningDestination,
  type SchedulingPriority
} from "../../lib/api";
import {
  DESTINATIONS, DESTINATION_LABEL, EMPTY_VITALS_FORM, NOTE_MAX, REASON_MIN, bmiOf, colorProblem, defaultDueDays, destinationPayload,
  destinationProblem, emptyDestination, needsColorReason, parseVitals, screeningError, vitalsFormFrom,
  type DestinationDraft, type VitalsForm
} from "../../lib/screening";
import { PRIORITY_LABEL } from "../../lib/scheduling";
import { useDebouncedValue } from "../../lib/useDebouncedValue";
import { FrozenTextNotice } from "../../components/FrozenTextNotice";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { Ciap2Search } from "./Ciap2Search";
import { VitalSignsFields } from "./VitalSignsFields";
import { ColorDecision, type SuggestionState } from "./ColorDecision";

// Recusas que encerram o formulário: quem chamou recarrega a fila e mostra a frase.
const CLOSING = new Set([ "not_in_progress", "attendance_not_waiting", "not_reassessable" ]);

export interface ScreeningFormProps {
  mode: "complete" | "reassess";
  screening: Screening;
  citizenLabel: string;
  unit: HealthUnit;
  units: HealthUnit[];
  types: AppointmentType[] | null;
  suggestDelayMs?: number;
  onDone(result: Screening): void;
  onClosed(message: string): void;
  onCancel(): void;
}

export function ScreeningForm(props: ScreeningFormProps) {
  const { mode, screening, citizenLabel, unit, units, types, suggestDelayMs = 400, onDone, onClosed, onCancel } = props;
  const current = mode === "reassess" ? screening.current_revision : null;
  const [ ciap, setCiap ] = useState<Ciap2Ref | null>(current?.ciap2 ?? null);
  const [ note, setNote ] = useState(current?.complaint_note ?? "");
  const [ vitalsForm, setVitalsForm ] = useState<VitalsForm>(current ? vitalsFormFrom(current.vitals) : EMPTY_VITALS_FORM);
  const [ finalChoice, setFinalChoice ] = useState<ScreeningColor | null>(null);
  const [ reason, setReason ] = useState("");
  const [ forceReason, setForceReason ] = useState(false);
  const [ dest, setDest ] = useState<DestinationDraft>(() => emptyDestination(null));
  const [ dueTouched, setDueTouched ] = useState(false);
  const [ state, setState ] = useState<SuggestionState>({ kind: "idle" });
  const [ refresh, setRefresh ] = useState(0);
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const seq = useRef(0);

  const { vitals, problems } = parseVitals(vitalsForm);
  const inputKey = JSON.stringify({ code: ciap?.code ?? null, vitals, refresh });
  const settledKey = useDebouncedValue(inputKey, suggestDelayMs);

  useEffect(() => {
    const { code, vitals: v } = JSON.parse(settledKey) as { code: string | null; vitals: typeof vitals };
    if (!code) { seq.current += 1; setState({ kind: "idle" }); return; }
    const mine = ++seq.current;
    setState({ kind: "loading" });
    suggestScreening({ ciap2_code: code, vitals: v, attendance_id: screening.attendance_id })
      .then((suggestion) => { if (mine === seq.current) setState({ kind: "ready", suggestion }); })
      .catch((err) => { if (mine === seq.current) setState({ kind: "error", message: screeningError(err) }); });
  }, [ settledKey, screening.attendance_id ]);

  const suggested = state.kind === "ready" ? state.suggestion.suggested_color : null;
  const final = finalChoice ?? suggested;
  const stale = ciap !== null && (inputKey !== settledKey || state.kind === "loading");

  // Prazo padrão segue a cor final até a pessoa mexer nele.
  useEffect(() => {
    if (dueTouched) return;
    const due = defaultDueDays(final);
    setDest((d) => ({ ...d, dueDays: due === null ? "" : String(due) }));
  }, [ final, dueTouched ]);

  const sendReason = forceReason || needsColorReason(suggested, final);
  const problem =
    (ciap === null ? "escolha a queixa (CIAP-2)" : null) ??
    (Object.keys(problems).length > 0 ? "corrija os sinais vitais" : null) ??
    (note.length > NOTE_MAX ? `a queixa em texto passa de ${NOTE_MAX} caracteres` : null) ??
    (forceReason && reason.trim().length < REASON_MIN
      ? `explique por que a cor final é diferente da sugerida (pelo menos ${REASON_MIN} caracteres)` : null) ??
    colorProblem(suggested, final, reason) ??
    (mode === "complete" ? destinationProblem(dest) : null);
  const blocked = busy || stale || problem !== null;

  function revisionBody() {
    return {
      ciap2_code: (ciap as Ciap2Ref).code,
      ...(note.trim() ? { complaint_note: note.trim() } : {}),
      vitals,
      final_color: final as ScreeningColor,
      ...(sendReason ? { color_change_reason: reason.trim() } : {})
    };
  }

  async function submit() {
    if (blocked) return;
    setBusy(true); setError(null);
    try {
      const result = mode === "complete"
        ? await completeScreening(screening.id, { ...revisionBody(), ...destinationPayload(dest) })
        : await reassessScreening(screening.id, revisionBody());
      onDone(result);
    } catch (err) {
      const code = errorCode(err);
      if (code && CLOSING.has(code)) { onClosed(screeningError(err)); return; }
      if (code === "color_change_reason_required") { setForceReason(true); setRefresh((n) => n + 1); }
      setError(screeningError(err));
    } finally {
      setBusy(false);
    }
  }

  async function abandon() {
    if (busy) return;
    setBusy(true); setError(null);
    try {
      await abandonScreening(screening.id);
      onClosed("Escuta abandonada: o atendimento voltou para a fila do acolhimento.");
    } catch (err) {
      if (errorCode(err) === "not_in_progress") { onClosed(screeningError(err)); return; }
      setError(screeningError(err));
    } finally {
      setBusy(false);
    }
  }

  const setD = (patch: Partial<DestinationDraft>) => setDest((d) => ({ ...d, ...patch }));
  const activeTypes = (types ?? []).filter((t) => t.active);
  const otherUnits = units.filter((u) => u.id !== unit.id);

  return (
    <section aria-label={mode === "complete" ? "Escuta inicial" : "Reavaliação"} style={panel}>
      <strong>{mode === "complete" ? "Escuta inicial" : "Reavaliar a escuta"}</strong>
      <p className="mono" style={{ margin: 0, fontSize: 11, color: "var(--ink3)" }}>{citizenLabel}</p>
      {error && <p role="alert" style={alertStyle}>{error}</p>}

      <Ciap2Search value={ciap} onChange={setCiap} />
      <label style={label}>
        Queixa em texto (opcional)
        <textarea value={note} style={{ ...inputStyle, minHeight: 48 }} aria-describedby="complaint-note-notice"
          onChange={(e) => setNote(e.target.value)} />
      </label>
      <FrozenTextNotice id="complaint-note-notice" />

      <VitalSignsFields form={vitalsForm} problems={problems} alerts={state.kind === "ready" ? state.suggestion.alerts : []}
        bmi={bmiOf(vitals.weight_kg, vitals.height_cm)} onChange={setVitalsForm} />

      <ColorDecision state={stale && state.kind !== "error" ? { kind: "loading" } : state} final={final} reason={reason}
        forceReason={forceReason} onFinal={setFinalChoice} onReason={setReason} />

      {mode === "complete" && (
        <fieldset aria-label="Destino do acolhimento" style={box}>
          <legend style={legend}>Destino</legend>
          <label style={label}>
            Destino
            <select value={dest.destination} style={inputStyle}
              onChange={(e) => setD({ destination: e.target.value as ScreeningDestination | "" })}>
              <option value="">escolha…</option>
              {DESTINATIONS.map((d) => <option key={d} value={d}>{DESTINATION_LABEL[d]}</option>)}
            </select>
          </label>

          {dest.destination === "schedule" && (
            <>
              {types ? (
                <label style={label}>
                  Tipo de atendimento
                  <select value={dest.typeKey} style={inputStyle} onChange={(e) => setD({ typeKey: e.target.value })}>
                    <option value="">escolha…</option>
                    {activeTypes.map((t) => <option key={t.key} value={t.key}>{t.name}</option>)}
                  </select>
                </label>
              ) : (
                <label style={label}>
                  Tipo de atendimento (chave)
                  <input value={dest.typeKey} style={inputStyle} placeholder="consulta_medica"
                    onChange={(e) => setD({ typeKey: e.target.value.trim() })} />
                </label>
              )}
              <label style={label}>
                Prioridade
                <select value={dest.priority} style={inputStyle} onChange={(e) => setD({ priority: e.target.value as SchedulingPriority })}>
                  <option value="routine">{PRIORITY_LABEL.routine}</option>
                  <option value="priority">{PRIORITY_LABEL.priority}</option>
                </select>
              </label>
              <label style={label}>
                Prazo (dias)
                <input value={dest.dueDays} inputMode="numeric" style={inputStyle}
                  onChange={(e) => { setDueTouched(true); setD({ dueDays: e.target.value }); }} />
              </label>
              <small style={hint}>Gera um pedido de agendamento nesta unidade e encerra o atendimento.</small>
            </>
          )}

          {dest.destination === "oriented" && (
            <>
              <label style={label}>
                Orientação dada
                <textarea value={dest.orientationNote} style={{ ...inputStyle, minHeight: 56 }} aria-describedby="orientation-note-notice"
                  onChange={(e) => setD({ orientationNote: e.target.value })} />
              </label>
              <FrozenTextNotice id="orientation-note-notice" />
            </>
          )}

          {dest.destination === "referred" && (
            <>
              <label style={label}>
                Unidade de destino
                <select value={dest.referralUnitId} style={inputStyle} onChange={(e) => setD({ referralUnitId: e.target.value })}>
                  <option value="">—</option>
                  {otherUnits.map((u) => <option key={u.id} value={u.id}>{u.name}</option>)}
                </select>
              </label>
              <label style={label}>
                Descrição do encaminhamento
                <input value={dest.referralNote} style={inputStyle} aria-describedby="screening-referral-notice"
                  onChange={(e) => setD({ referralNote: e.target.value })} />
              </label>
              <FrozenTextNotice id="screening-referral-notice" />
            </>
          )}
        </fieldset>
      )}

      {problem && !busy && <small style={hint}>{problem}</small>}
      <div style={{ display: "flex", gap: 8, flexWrap: "wrap" }}>
        <button type="button" disabled={blocked} onClick={() => void submit()} style={blocked ? disabledButtonStyle : buttonStyle}>
          {mode === "complete" ? "Concluir escuta" : "Salvar reavaliação"}
        </button>
        {mode === "complete" ? (
          <button type="button" disabled={busy} onClick={() => void abandon()} style={secondaryButtonStyle}>Abandonar escuta</button>
        ) : (
          <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Cancelar</button>
        )}
      </div>
    </section>
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 12, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 };
const box: CSSProperties = { border: "1px solid var(--rule)", borderRadius: 8, padding: "8px 12px", margin: 0, display: "flex", flexDirection: "column", gap: 8 };
const legend: CSSProperties = { fontSize: 13, fontWeight: 600 };
const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alertStyle: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/modules/attendance/ScreeningForm.test.tsx`
Expected: PASS (10 testes). Se "resposta velha da sugestão é descartada" falhar com uma chamada só ao `suggest`, confira que o teste espera a primeira chamada (`toHaveBeenCalledTimes(1)`) antes de digitar a temperatura — sem isso, a espera junta as duas mudanças numa chamada só.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod18 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 add src/modules/attendance/ScreeningForm.tsx src/modules/attendance/ScreeningForm.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 commit -m "feat: complete and reassess the initial listening form

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 6: Fila do acolhimento no Atendimento

**Files:**
- Create: `src/modules/attendance/ScreeningQueue.tsx`
- Modify: `src/modules/Attendance.tsx` (import e o painel antes da `UnitQueue`)
- Test: `src/modules/attendance/ScreeningQueue.test.tsx`; Modify: `src/modules/Attendance.test.tsx`

**Interfaces:**
- Consumes: `listScreeningQueue`, `startScreening`, `getScreening`, `listAppointmentTypes`, `errorCode` (Task 1 e existentes); `COLOR_LABEL`, `screeningError`, `waitLabel`, `waitedMinutes` (Task 2); `ScreeningForm` (Task 5); `APPOINTMENT_TYPES_KEY` (`src/modules/professionals/AppointmentTypes.tsx`); `ATTENDANCE_REFETCH_MS`.
- Produces: `ScreeningQueue({ unit: HealthUnit; units: HealthUnit[]; now?(): Date })` — `Panel` "Acolhimento"; colunas CPF, Chegada, Espera, Triagem digital, Situação; botão "Iniciar escuta" ou "Retomar escuta" (escuta `in_progress`: lê pelo `GET` e abre o formulário); `SCREENING_QUEUE_KEY = "screeningQueue"` (chave `[ "screeningQueue", unit.id ]`). Concluir ou fechar invalida `[ "screeningQueue", unit.id ]` e `[ "unitQueue", unit.id ]`; destino `schedule` também `[ "unitRequests" ]`. 409 `already_screening`, `not_waiting`, `screening_not_required`, `not_in_progress` recarregam e avisam; 403 `cbo_not_allowed`/`missing_link`/`missing_role` na leitura viram só a frase, sem a fila.
- `Attendance` mostra o painel para `canCare && unit` (profissional com vínculo ativo na unidade), nunca para a recepção sem esse papel.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/attendance/ScreeningQueue.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, listScreeningQueue: vi.fn(), startScreening: vi.fn(), getScreening: vi.fn(), listAppointmentTypes: vi.fn() };
});
// O formulário tem testes próprios: aqui ele só devolve o desfecho.
vi.mock("./ScreeningForm", () => ({
  ScreeningForm: (p: { screening: { id: string }; onDone(s: unknown): void; onClosed(m: string): void }) => (
    <div>
      <span>{`formulário ${p.screening.id}`}</span>
      <button type="button" onClick={() => p.onDone({ id: p.screening.id, destination: "same_day",
        current_revision: { final_color: "red" } })}>concluir no dia</button>
      <button type="button" onClick={() => p.onDone({ id: p.screening.id, destination: "schedule", current_revision: null })}>
        concluir agendando
      </button>
      <button type="button" onClick={() => p.onClosed("Escuta abandonada: o atendimento voltou para a fila do acolhimento.")}>
        fechar
      </button>
    </div>
  )
}));

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { ScreeningQueue } from "./ScreeningQueue";
import { NOW18, queueItem, screening } from "../../test/screeningFixtures";
import { TYPES } from "../../test/schedulingFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };

function renderIt() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const spy = vi.spyOn(client, "invalidateQueries");
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<ScreeningQueue unit={unit} units={[ unit ]} />, { wrapper });
  return { spy };
}

describe("ScreeningQueue", () => {
  beforeEach(() => {
    for (const fn of [ api.listScreeningQueue, api.startScreening, api.getScreening, api.listAppointmentTypes ]) mocked(fn).mockReset();
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW18));
    mocked(api.listAppointmentTypes).mockResolvedValue(TYPES);
    mocked(api.listScreeningQueue).mockResolvedValue([
      queueItem(),
      queueItem({ attendance_id: "a2", citizen: { id: "c2", cpf_masked: "***.111.222-**" }, checked_in_at: "2026-10-07T08:55:00-03:00",
        triage_priority: null, screening: { id: "sc9", status: "in_progress", started_by_name: "Téc. Rui Alves" } })
    ]);
  });

  it("lista por chegada com espera, sinal da triagem digital e situação", async () => {
    renderIt();
    expect(await screen.findByText("***.982.247-**")).not.toBeNull();
    expect(screen.getByText("40 min")).not.toBeNull();
    expect(screen.getByText("1 h 05 min")).not.toBeNull();
    expect(screen.getByText("prioridade 2")).not.toBeNull();
    expect(screen.getByText("em escuta com Téc. Rui Alves")).not.toBeNull();
    expect(screen.getByRole("button", { name: "Retomar escuta" })).not.toBeNull();
  });

  it("iniciar abre o formulário; concluir no dia recarrega as duas filas e avisa", async () => {
    mocked(api.startScreening).mockResolvedValue(screening());
    const { spy } = renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar escuta" }));
    expect(await screen.findByText("formulário sc1")).not.toBeNull();
    expect(api.startScreening).toHaveBeenCalledWith("a1");
    fireEvent.click(screen.getByRole("button", { name: "concluir no dia" }));
    expect(await screen.findByText("Escuta concluída (vermelho): segue na fila do profissional.")).not.toBeNull();
    expect(spy).toHaveBeenCalledWith({ queryKey: [ "screeningQueue", "u1" ] });
    expect(spy).toHaveBeenCalledWith({ queryKey: [ "unitQueue", "u1" ] });
  });

  it("agendar também recarrega os pedidos da recepção", async () => {
    mocked(api.startScreening).mockResolvedValue(screening());
    const { spy } = renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar escuta" }));
    fireEvent.click(await screen.findByRole("button", { name: "concluir agendando" }));
    expect(await screen.findByText("Escuta concluída: pedido de agendamento criado e atendimento encerrado.")).not.toBeNull();
    expect(spy).toHaveBeenCalledWith({ queryKey: [ "unitRequests" ] });
  });

  it("retomar lê a escuta em andamento", async () => {
    mocked(api.getScreening).mockResolvedValue(screening({ id: "sc9", attendance_id: "a2" }));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Retomar escuta" }));
    expect(await screen.findByText("formulário sc9")).not.toBeNull();
    expect(api.getScreening).toHaveBeenCalledWith("sc9");
    expect(api.startScreening).not.toHaveBeenCalled();
  });

  it("already_screening recarrega a fila e avisa, sem abrir formulário", async () => {
    mocked(api.startScreening).mockRejectedValue(new ApiError(409, { error: "already_screening" }, "x"));
    const { spy } = renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar escuta" }));
    expect(await screen.findByText("outra pessoa já começou a escuta deste atendimento — a fila foi atualizada")).not.toBeNull();
    expect(spy).toHaveBeenCalledWith({ queryKey: [ "screeningQueue", "u1" ] });
    expect(screen.queryByText(/formulário/)).toBeNull();
  });

  it("fechar pelo formulário volta à fila com a frase", async () => {
    mocked(api.startScreening).mockResolvedValue(screening());
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar escuta" }));
    fireEvent.click(await screen.findByRole("button", { name: "fechar" }));
    expect(await screen.findByText("Escuta abandonada: o atendimento voltou para a fila do acolhimento.")).not.toBeNull();
    await waitFor(() => expect(screen.getByRole("button", { name: "Iniciar escuta" })).not.toBeNull());
  });

  it("CBO que não faz acolhimento: só a frase, sem a fila", async () => {
    mocked(api.listScreeningQueue).mockRejectedValue(new ApiError(403, { error: "cbo_not_allowed" }, "x"));
    renderIt();
    expect(await screen.findByText("sua ocupação (CBO) não faz acolhimento")).not.toBeNull();
    expect(screen.queryByRole("table")).toBeNull();
  });
});
```

E em `src/modules/Attendance.test.tsx` (mock, padrão e o teste novo):

```diff
--- a/src/modules/Attendance.test.tsx
+++ b/src/modules/Attendance.test.tsx
@@ -13,7 +13,8 @@ vi.mock("../lib/api", async (importOriginal) => {
     lookupCheckIn: vi.fn(), checkIn: vi.fn(), searchCheckIn: vi.fn(), checkInByException: vi.fn(),
     listUnitRequests: vi.fn(), bookAppointment: vi.fn(), getUnitAvailability: vi.fn(), dismissRequest: vi.fn(),
     getUnitAgenda: vi.fn(), listUnassignedRequests: vi.fn(),
-    getMyProfessional: vi.fn(), listPendingErasures: vi.fn(), cadsusLookup: vi.fn()
+    getMyProfessional: vi.fn(), listPendingErasures: vi.fn(), cadsusLookup: vi.fn(),
+    listScreeningQueue: vi.fn(), listAppointmentTypes: vi.fn()
   };
 });
 
@@ -59,7 +60,7 @@ describe("Attendance", () => {
       api.listUnitQueue, api.callAttendance, api.callNext, api.closeAttendance,
       api.lookupCheckIn, api.checkIn, api.searchCheckIn, api.checkInByException,
       api.listUnitRequests, api.bookAppointment, api.getUnitAvailability, api.dismissRequest, api.getUnitAgenda,
-      api.listUnassignedRequests, api.getMyProfessional
+      api.listUnassignedRequests, api.getMyProfessional, api.listScreeningQueue, api.listAppointmentTypes
     ]) {
       mocked(fn).mockReset();
     }
@@ -71,6 +72,8 @@ describe("Attendance", () => {
     mocked(api.getUnitAgenda).mockResolvedValue({ date: "2026-10-05", professionals: [], unassigned: [] });
     mocked(api.listUnassignedRequests).mockResolvedValue([]);
     mocked(api.getMyProfessional).mockResolvedValue(null);
+    mocked(api.listScreeningQueue).mockResolvedValue([]);
+    mocked(api.listAppointmentTypes).mockResolvedValue([]);
   });
 
   it("recepção vê 'Pedidos sem unidade' mesmo sem unidade escolhida", async () => {
@@ -256,6 +259,27 @@ describe("Attendance", () => {
     expect(screen.queryByText("Agenda do dia")).toBeNull();
   });
 
+  it("acolhimento (módulo 18): profissional com vínculo vê a fila da escuta; a recepção não", async () => {
+    const unit = { id: "un1", name: "UBS Centro", kind: "ubs" };
+    localStorage.setItem(currentUnitKey("u1"), unit.id);
+    mocked(api.listActiveUnits).mockResolvedValue([ unit ]);
+    renderAttendance();
+    expect(await screen.findByText("Pedidos de agendamento")).not.toBeNull();
+    expect(screen.queryByRole("region", { name: "Acolhimento" })).toBeNull();
+    expect(api.listScreeningQueue).not.toHaveBeenCalled();
+    cleanup();
+
+    mocked(api.fetchCurrentSession).mockResolvedValue(session("health_professional"));
+    mocked(api.getMyProfessional).mockResolvedValue({
+      professional: {} as api.Professional, shifts: [],
+      links: [ { id: "l1", health_unit_id: "un1", unit_name: "UBS Centro", cbo_code: "223505", cbo_title: null,
+        started_at: "x", started_by: "a", ended_at: null, ended_by: null } ]
+    });
+    renderAttendance();
+    expect(await screen.findByRole("region", { name: "Acolhimento" })).not.toBeNull();
+    await waitFor(() => expect(api.listScreeningQueue).toHaveBeenCalledWith("un1"));
+  });
+
   it("profissional sem vínculo com a unidade escolhida: sem ações clínicas", async () => {
     const unit = { id: "h1", name: "UBS Centro", kind: "ubs" };
     mocked(api.fetchCurrentSession).mockResolvedValue(session("health_professional"));
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/modules/attendance/ScreeningQueue.test.tsx src/modules/Attendance.test.tsx`
Expected: FAIL — `Failed to resolve import "./ScreeningQueue"`; no `Attendance.test.tsx`, "acolhimento (módulo 18)…" falha em `findByRole("region", { name: "Acolhimento" })`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/attendance/ScreeningQueue.tsx
// Fila do acolhimento (módulo 18; spec §4; contratos §3): atendimentos que
// aguardam e precisam de escuta pelo escopo da unidade, por chegada, com a
// prioridade da triagem digital só como sinal. Quem tem vínculo com a unidade
// e CBO permitido inicia ou retoma a escuta; o api é quem confere os dois.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import {
  errorCode, getScreening, listAppointmentTypes, listScreeningQueue, startScreening,
  type HealthUnit, type Screening, type ScreeningQueueItem
} from "../../lib/api";
import { ATTENDANCE_REFETCH_MS } from "../../lib/attendance";
import { COLOR_LABEL, screeningError, waitLabel, waitedMinutes } from "../../lib/screening";
import { fmtHourMinute } from "../../lib/format";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { EmptyState } from "../../components/EmptyState";
import { disabledButtonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { APPOINTMENT_TYPES_KEY } from "../professionals/AppointmentTypes";
import { ScreeningForm } from "./ScreeningForm";

export const SCREENING_QUEUE_KEY = "screeningQueue";
// Quem não faz escuta nesta unidade: a fila some e fica a frase.
const NOT_FOR_YOU = new Set([ "cbo_not_allowed", "missing_link", "missing_role" ]);
// Outro profissional mexeu primeiro: recarrega e avisa.
const STALE = new Set([ "already_screening", "not_waiting", "screening_not_required", "not_in_progress" ]);

interface Open { item: ScreeningQueueItem; screening: Screening }

export function ScreeningQueue({ unit, units, now = () => new Date() }: { unit: HealthUnit; units: HealthUnit[]; now?(): Date }) {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: [ SCREENING_QUEUE_KEY, unit.id ], queryFn: () => listScreeningQueue(unit.id),
    refetchInterval: ATTENDANCE_REFETCH_MS });
  const types = useQuery({ queryKey: APPOINTMENT_TYPES_KEY, queryFn: listAppointmentTypes });
  const [ open, setOpen ] = useState<Open | null>(null);
  const [ rowBusy, setRowBusy ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ actionError, setActionError ] = useState<string | null>(null);

  function refresh() {
    void queryClient.invalidateQueries({ queryKey: [ SCREENING_QUEUE_KEY, unit.id ] });
    void queryClient.invalidateQueries({ queryKey: [ "unitQueue", unit.id ] });
  }

  async function begin(item: ScreeningQueueItem) {
    if (rowBusy || open) return;
    setRowBusy(item.attendance_id); setNotice(null); setActionError(null);
    try {
      const resuming = item.screening?.status === "in_progress";
      const screening = resuming ? await getScreening(item.screening!.id) : await startScreening(item.attendance_id);
      setOpen({ item, screening });
    } catch (err) {
      const code = errorCode(err);
      if (code && STALE.has(code)) { refresh(); setNotice(screeningError(err)); return; }
      setActionError(screeningError(err));
    } finally {
      setRowBusy(null);
    }
  }

  const blockedCode = query.isError ? errorCode(query.error) : undefined;
  if (blockedCode && NOT_FOR_YOU.has(blockedCode)) {
    return (
      <Panel title="Acolhimento" sub="escuta inicial">
        <p role="status" style={{ margin: 0, fontSize: 12.5 }}>{screeningError(query.error)}</p>
      </Panel>
    );
  }

  const items = query.data ?? [];
  const at = now();

  return (
    <Panel title="Acolhimento" sub="escuta inicial · por ordem de chegada">
      <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
        {notice && <p role="status" style={{ margin: 0, fontSize: 12.5, fontWeight: 600 }}>{notice}</p>}
        {actionError && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{actionError}</p>}
        {query.isError && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{screeningError(query.error)}</p>}

        {open ? (
          <ScreeningForm
            key={open.screening.id}
            mode="complete"
            screening={open.screening}
            citizenLabel={`${open.item.citizen.cpf_masked} · chegou às ${fmtHourMinute(open.item.checked_in_at)}`}
            unit={unit}
            units={units}
            types={types.isSuccess ? types.data : null}
            onDone={(result) => {
              setOpen(null);
              refresh();
              if (result.destination === "schedule") void queryClient.invalidateQueries({ queryKey: [ "unitRequests" ] });
              setNotice(doneMessage(result));
            }}
            onClosed={(message) => { setOpen(null); refresh(); setNotice(message); }}
            onCancel={() => setOpen(null)}
          />
        ) : query.isPending ? (
          <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando…</p>
        ) : items.length === 0 ? (
          <EmptyState title="Ninguém aguardando escuta" />
        ) : (
          <DataTable<ScreeningQueueItem>
            cols={[
              { label: "CPF", w: "1.5fr", render: (r) => r.citizen.cpf_masked },
              { label: "Chegada", w: "0.8fr", render: (r) => fmtHourMinute(r.checked_in_at) },
              { label: "Espera", w: "0.8fr", render: (r) => waitLabel(waitedMinutes(r.checked_in_at, at)) },
              { label: "Triagem digital", w: "1fr", render: (r) => (r.triage_priority === null ? "—" : `prioridade ${r.triage_priority}`) },
              { label: "Situação", w: "1.6fr", render: (r) => situation(r) },
              {
                label: "", w: "auto", align: "right", render: (r) => (
                  <button type="button" disabled={rowBusy === r.attendance_id}
                    style={rowBusy === r.attendance_id ? disabledButtonStyle : secondaryButtonStyle}
                    onClick={() => void begin(r)}>
                    {r.screening?.status === "in_progress" ? "Retomar escuta" : "Iniciar escuta"}
                  </button>
                )
              }
            ]}
            rows={items}
            rowKey={(r) => r.attendance_id}
          />
        )}
      </div>
    </Panel>
  );
}

function situation(r: ScreeningQueueItem): string {
  if (!r.screening) return "aguardando escuta";
  if (r.screening.status === "in_progress") return `em escuta com ${r.screening.started_by_name}`;
  return "escuta abandonada — aguardando de novo";
}

function doneMessage(s: Screening): string {
  const color = s.current_revision ? COLOR_LABEL[s.current_revision.final_color] : null;
  switch (s.destination) {
    case "same_day": return `Escuta concluída${color ? ` (${color})` : ""}: segue na fila do profissional.`;
    case "schedule": return "Escuta concluída: pedido de agendamento criado e atendimento encerrado.";
    case "oriented": return "Escuta concluída: orientação registrada e atendimento encerrado.";
    case "referred": return "Escuta concluída: encaminhamento registrado e atendimento encerrado.";
    default: return "Escuta concluída.";
  }
}
```

Em `src/modules/Attendance.tsx`:

```diff
--- a/src/modules/Attendance.tsx
+++ b/src/modules/Attendance.tsx
@@ -20,6 +20,7 @@ import { ProfileCheck, initialProfileCheck, profileCheckProblem, type ProfileChe
 import { UnitPicker } from "./attendance/UnitPicker";
 import { CheckIn } from "./attendance/CheckIn";
 import { UnitQueue } from "./attendance/UnitQueue";
+import { ScreeningQueue } from "./attendance/ScreeningQueue";
 import { Requests } from "./attendance/Requests";
 import { Agenda } from "./attendance/Agenda";
 import { UnassignedRequests } from "./attendance/UnassignedRequests";
@@ -98,6 +99,7 @@ export function Attendance({ onNavigate }: { onNavigate(id: ModuleId): void }) {
       <PageHeader title="Atendimento" sub="balcão · verificação presencial" />
       {(canVerify || canCareRole) && <UnitPicker key={pickerKey} userId={user.id} onChange={setUnit} />}
       {canVerify && unit && <CheckIn unit={unit} onUnitInvalid={onUnitInvalid} />}
+      {canCare && unit && <ScreeningQueue unit={unit} units={unitsQuery.data ?? []} />}
       {(canVerify || canCareRole) && unit && (
         <UnitQueue
           unit={unit}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/modules/attendance/ScreeningQueue.test.tsx src/modules/Attendance.test.tsx`
Expected: PASS (7 testes; `Attendance.test.tsx` com 32).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod18 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 add src/modules/attendance/ScreeningQueue.tsx src/modules/attendance/ScreeningQueue.test.tsx src/modules/Attendance.tsx src/modules/Attendance.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 commit -m "feat: add the screening queue to the attendance page

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 7: Fila do profissional com cor, destaque do vermelho, escuta no atendimento chamado e reavaliação

**Files:**
- Modify: `src/components/DataTable.tsx` (prop `rowStyle`), `src/components/DataTable.test.tsx`
- Create: `src/modules/attendance/ScreeningDetail.tsx`
- Modify: `src/modules/attendance/UnitQueue.tsx`
- Test: `src/modules/attendance/UnitQueue.screening.test.tsx` (o `UnitQueue.test.tsx` existente não muda e continua verde)

**Interfaces:**
- Consumes: `getScreening`, `CallResult`, `QueueRow.screening` (Task 1); `COLOR_LABEL`, `COLOR_TONE`, `DESTINATION_LABEL`, `GLUCOSE_MOMENT_LABEL`, `VITALS`, `alertLabel`, `screeningError`, `waitLabel`, `waitedMinutes` (Task 2); `ScreeningForm` (Task 5).
- Produces:
  - `DataTable` aceita `rowStyle?: (row: T) => CSSProperties | undefined`, somado ao estilo da linha;
  - `ScreeningDetail({ screening: Screening; onClose(): void })` — seção "Escuta inicial do atendimento", com cor, destino, contagem de revisões, CIAP-2, queixa, sinais ("Pressão sistólica: 185 mmHg"), alertas, sugerida/justificativa, autor, "Revisões anteriores" e "Fechar escuta"; `ScreeningDetailLoader({ id; onClose })` lê pelo `GET` (chave `[ "screening", id ]`);
  - `UnitQueue` ganha `now?(): Date`; colunas "Cor" (em Aguardando e Em atendimento) e "Espera" (Aguardando: `waited_minutes` da escuta ou minutos desde a chegada); linha com cor vermelha com fundo `var(--down-bg)` e filete `var(--down)`; para `canCare`: "Reavaliar" (Aguardando, escuta com `id` e destino `same_day`), "Ver escuta" (Em atendimento, escuta com `id`), e a escuta que vem na resposta da chamada abre sozinha; a recepção (`canCare` falso) só vê a cor.
- Sem o `id` no bloco da fila (Divergência D2 recusada), os botões somem e a cor continua — o teste cobre a linha `noId`.

- [ ] **Step 1: Write the failing test**

```diff
--- a/src/components/DataTable.test.tsx
+++ b/src/components/DataTable.test.tsx
@@ -25,6 +25,14 @@ const plainCols: Column<Row>[] = [ { label: "Nome", render: (r) => r.label } ];
 let actionClicked: (r: Row) => void;
 
 describe("DataTable", () => {
+  it("rowStyle destaca só as linhas que pedem", () => {
+    render(<DataTable<Row> cols={plainCols} rows={rows} rowKey={(r) => r.id}
+      rowStyle={(r) => (r.id === "2" ? { background: "var(--down-bg)" } : undefined)} />);
+    const rowEls = screen.getAllByRole("row").slice(1);
+    expect(rowEls[0].style.background).toBe("transparent");
+    expect(rowEls[1].style.background).toBe("var(--down-bg)");
+  });
+
   it("sem onRowClick, a linha não é <button> e um botão de ação na célula recebe o clique", async () => {
     const onAction = vi.fn();
     actionClicked = onAction;
```

```tsx
// src/modules/attendance/UnitQueue.screening.test.tsx
// Módulo 18: cor, destaque do vermelho, espera, escuta no atendimento chamado e reavaliação.
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listUnitQueue: vi.fn(), callAttendance: vi.fn(), callNext: vi.fn(),
    closeAttendance: vi.fn(), getScreening: vi.fn() };
});
vi.mock("./ScreeningForm", () => ({
  ScreeningForm: (p: { mode: string; screening: { id: string }; onDone(s: unknown): void }) => (
    <div>
      <span>{`formulário ${p.mode} ${p.screening.id}`}</span>
      <button type="button" onClick={() => p.onDone(p.screening)}>salvar formulário</button>
    </div>
  )
}));

import * as api from "../../lib/api";
import type { QueueRow } from "../../lib/api";
import { AuthProvider } from "../../lib/auth";
import { UnitQueue } from "./UnitQueue";
import { NOW18, revision, screening } from "../../test/screeningFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };

function row(over: Partial<QueueRow>): QueueRow {
  return { id: "a1", cpf_masked: "***.982.247-**", checked_in_at: "2026-10-07T09:20:00-03:00", protocol_name: null, priority: null,
    source: "triage", appointment_time: null, called_at: null, called_by_name: null, screening: null, ...over };
}
const red = row({ id: "a1", screening: { id: "sc1", color: "red", destination: "same_day", waited_minutes: 25 } });
const plain = row({ id: "a2", cpf_masked: "***.111.222-**", checked_in_at: "2026-10-07T09:00:00-03:00" });
const noId = row({ id: "a4", cpf_masked: "***.555.666-**", screening: { color: "green", destination: "same_day", waited_minutes: 5 } });
const inCare = row({ id: "a3", cpf_masked: "***.333.444-**", called_at: "2026-10-07T09:50:00-03:00", called_by_name: "Dra. Helena",
  screening: { id: "sc3", color: "yellow", destination: "same_day", waited_minutes: 30 } });

function session(role: string) {
  return { id: "us1", email_address: "x@cidade.gov.br", operator: false, mfa_enrolled: true, mfa_verified_at: null,
    memberships: [ { city_slug: "m1", city_name: "Curitiba", city_uf: "PR", role } ] };
}
function renderQueue(canCare: boolean) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  render(<UnitQueue unit={unit} units={[ unit ]} canCare={canCare} />, { wrapper });
}
const waitingRows = async () => {
  const section = (await screen.findByText("Aguardando")).parentElement as HTMLElement;
  return within(section).getAllByRole("row").slice(1);
};

describe("UnitQueue — acolhimento (módulo 18)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listUnitQueue, api.callAttendance, api.callNext, api.closeAttendance, api.getScreening ]) {
      mocked(fn).mockReset();
    }
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW18));
    mocked(api.fetchCurrentSession).mockResolvedValue(session("health_professional"));
    mocked(api.listUnitQueue).mockResolvedValue({ waiting: [ red, plain, noId ], in_care: [ inCare ] });
  });

  it("cor, vermelho destacado e espera (da escuta ou desde a chegada)", async () => {
    renderQueue(true);
    const rows = await waitingRows();
    expect(within(rows[0]).getByText("vermelho")).not.toBeNull();
    expect(rows[0].style.background).toBe("var(--down-bg)");
    expect(within(rows[0]).getByText("25 min")).not.toBeNull();
    expect(rows[1].style.background).toBe("transparent");
    expect(within(rows[1]).getByText("1 h 00 min")).not.toBeNull();
  });

  it("recepção vê só a cor: sem Reavaliar nem Ver escuta, e nada é lido", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session("citizen_verifier"));
    renderQueue(false);
    const rows = await waitingRows();
    expect(within(rows[0]).getByText("vermelho")).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Reavaliar" })).toBeNull();
    expect(screen.queryByRole("button", { name: "Ver escuta" })).toBeNull();
    expect(api.getScreening).not.toHaveBeenCalled();
  });

  it("chamar mostra a escuta que veio na chamada", async () => {
    mocked(api.callAttendance).mockResolvedValue({ attendance: { id: "a1" },
      screening: screening({ status: "completed", destination: "same_day", current_revision: revision(), revisions_count: 1 }) });
    renderQueue(true);
    const rows = await waitingRows();
    fireEvent.click(within(rows[0]).getByRole("button", { name: "Chamar" }));
    const detail = await screen.findByRole("region", { name: "Escuta inicial do atendimento" });
    expect(within(detail).getByText("K86")).not.toBeNull();
    expect(within(detail).getByText("Pressão sistólica: 185 mmHg")).not.toBeNull();
    expect(within(detail).getByText("Pressão sistólica: acima da faixa de alerta · Pressão diastólica: acima da faixa de alerta")).not.toBeNull();
    expect(api.getScreening).not.toHaveBeenCalled();
    fireEvent.click(within(detail).getByRole("button", { name: "Fechar escuta" }));
    expect(screen.queryByRole("region", { name: "Escuta inicial do atendimento" })).toBeNull();
  });

  it("Ver escuta lê pelo id (trilha de leitura) e mostra as revisões anteriores", async () => {
    const older = revision({ id: "rv0", final_color: "red", created_at: "2026-10-07T09:10:00-03:00" });
    mocked(api.getScreening).mockResolvedValue(screening({ id: "sc3", status: "completed", destination: "same_day",
      current_revision: revision({ final_color: "yellow" }), revisions: [ older, revision({ final_color: "yellow" }) ], revisions_count: 2 }));
    renderQueue(true);
    fireEvent.click(await screen.findByRole("button", { name: "Ver escuta" }));
    const detail = await screen.findByRole("region", { name: "Escuta inicial do atendimento" });
    expect(api.getScreening).toHaveBeenCalledWith("sc3");
    expect(within(detail).getByText("2 revisões")).not.toBeNull();
    expect(within(detail).getByText("07/10/2026, 09:10 · Enf. Lúcia Prado · vermelho · K86")).not.toBeNull();
  });

  it("Reavaliar só em quem espera com escuta no dia; salvar recarrega e avisa", async () => {
    mocked(api.getScreening).mockResolvedValue(screening({ status: "completed", destination: "same_day", current_revision: revision() }));
    renderQueue(true);
    const rows = await waitingRows();
    expect(within(rows[1]).queryByRole("button", { name: "Reavaliar" })).toBeNull();
    expect(within(rows[2]).queryByRole("button", { name: "Reavaliar" })).toBeNull();
    fireEvent.click(within(rows[0]).getByRole("button", { name: "Reavaliar" }));
    expect(await screen.findByText("formulário reassess sc1")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "salvar formulário" }));
    expect(await screen.findByText("Reavaliação registrada.")).not.toBeNull();
    await waitFor(() => expect(api.listUnitQueue).toHaveBeenCalledTimes(2));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/components/DataTable.test.tsx src/modules/attendance/UnitQueue.screening.test.tsx`
Expected: FAIL — `rowStyle` ignorado (`expected 'transparent' to be 'var(--down-bg)'`) e, na fila, `Unable to find an element with the text: vermelho`.

- [ ] **Step 3: Write minimal implementation**

`src/components/DataTable.tsx`:

```diff
--- a/src/components/DataTable.tsx
+++ b/src/components/DataTable.tsx
@@ -1,7 +1,7 @@
 // DataTable — tabela em CSS grid. Header mono uppercase. Linhas via render(row).
 // `cols[*].w` aceita qualquer grid-template-columns value (fr, px, minmax).
 
-import type { ReactNode } from "react";
+import type { CSSProperties, ReactNode } from "react";
 
 export interface Column<T> {
   label: string;
@@ -16,9 +16,11 @@ interface Props<T> {
   rowKey: (row: T, i: number) => string;
   onRowClick?: (row: T) => void;
   empty?: ReactNode;
+  // Destaque de uma linha (ex.: cor vermelha do acolhimento, módulo 18).
+  rowStyle?: (row: T) => CSSProperties | undefined;
 }
 
-export function DataTable<T>({ cols, rows, rowKey, onRowClick, empty }: Props<T>) {
+export function DataTable<T>({ cols, rows, rowKey, onRowClick, empty, rowStyle: extraStyle }: Props<T>) {
   const gridCols = cols.map((c) => c.w || "1fr").join(" ");
   if (rows.length === 0) {
     return <div className="mono" style={{ fontSize: 10.5, color: "var(--ink3)" }}>{empty ?? "sem dados"}</div>;
@@ -61,7 +63,8 @@ export function DataTable<T>({ cols, rows, rowKey, onRowClick, empty }: Props<T>
           color: "var(--ink)",
           cursor: onRowClick ? "pointer" : "default",
           textAlign: "left" as const,
-          alignItems: "center"
+          alignItems: "center",
+          ...(extraStyle?.(r) ?? {})
         };
         const cells = cols.map((c, ci) => (
           <span key={ci} role="cell" style={{ textAlign: c.align || "left", minWidth: 0, overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>
```

`src/modules/attendance/ScreeningDetail.tsx`:

```tsx
// src/modules/attendance/ScreeningDetail.tsx
// A escuta dentro do atendimento chamado (módulo 18; spec §8; contratos §3–§4):
// só para o profissional. Abrir pelo GET gera a trilha de leitura no api
// (`screening.viewed`); a recepção nunca chega aqui.
import type { CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { getScreening, type Screening, type ScreeningRevision } from "../../lib/api";
import {
  COLOR_LABEL, COLOR_TONE, DESTINATION_LABEL, GLUCOSE_MOMENT_LABEL, VITALS, alertLabel, screeningError
} from "../../lib/screening";
import { fmtDateTime, fmtNumber } from "../../lib/format";
import { Tag } from "../../components/Tag";
import { secondaryButtonStyle } from "../../components/formStyles";

export function ScreeningDetailLoader({ id, onClose }: { id: string; onClose(): void }) {
  const query = useQuery({ queryKey: [ "screening", id ], queryFn: () => getScreening(id), staleTime: 0 });
  if (query.isPending) return <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando a escuta…</p>;
  if (query.isError) return <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{screeningError(query.error)}</p>;
  return <ScreeningDetail screening={query.data} onClose={onClose} />;
}

export function ScreeningDetail({ screening, onClose }: { screening: Screening; onClose(): void }) {
  const rev = screening.current_revision;
  const earlier = (screening.revisions ?? []).filter((r) => r.id !== rev?.id);
  return (
    <section aria-label="Escuta inicial do atendimento" style={panel}>
      <div style={{ display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" }}>
        <strong>Escuta inicial</strong>
        {rev && <Tag tone={COLOR_TONE[rev.final_color]}>{COLOR_LABEL[rev.final_color]}</Tag>}
        {screening.destination && <span style={muted}>{`destino: ${DESTINATION_LABEL[screening.destination]}`}</span>}
        <span style={muted}>{`${screening.revisions_count} ${screening.revisions_count === 1 ? "revisão" : "revisões"}`}</span>
      </div>
      {rev ? <RevisionBody rev={rev} /> : <p style={muted}>Escuta sem registro concluído.</p>}
      {screening.orientation_note && <p style={{ margin: 0, fontSize: 12.5 }}>{`Orientação: ${screening.orientation_note}`}</p>}
      {earlier.length > 0 && (
        <details>
          <summary style={{ fontSize: 12 }}>Revisões anteriores</summary>
          <ul style={{ margin: 0, paddingLeft: 18, fontSize: 12 }}>
            {earlier.map((r) => (
              <li key={r.id}>{`${fmtDateTime(r.created_at)} · ${r.by.name} · ${COLOR_LABEL[r.final_color]} · ${r.ciap2.code}`}</li>
            ))}
          </ul>
        </details>
      )}
      <div><button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar escuta</button></div>
    </section>
  );
}

function RevisionBody({ rev }: { rev: ScreeningRevision }) {
  const vitals = VITALS.filter((v) => typeof rev.vitals[v.key] === "number");
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 6, fontSize: 12.5 }}>
      <span><span className="mono">{rev.ciap2.code}</span>{` — ${rev.ciap2.label}`}</span>
      {rev.complaint_note && <span>{`Queixa: ${rev.complaint_note}`}</span>}
      {vitals.length > 0 && (
        <ul aria-label="sinais vitais registrados" style={{ margin: 0, paddingLeft: 18 }}>
          {vitals.map((v) => (
            <li key={v.key}>{`${v.label}: ${fmtNumber(rev.vitals[v.key] as number)}${v.unit ? ` ${v.unit}` : ""}`}</li>
          ))}
          {rev.vitals.glucose_moment && <li>{`Momento da glicemia: ${GLUCOSE_MOMENT_LABEL[rev.vitals.glucose_moment]}`}</li>}
          {typeof rev.vitals.bmi === "number" && <li>{`IMC: ${fmtNumber(rev.vitals.bmi)}`}</li>}
        </ul>
      )}
      {rev.alerts.length > 0 && <span style={{ color: "var(--down)", fontWeight: 600 }}>{rev.alerts.map(alertLabel).join(" · ")}</span>}
      <span style={muted}>
        {rev.suggested_color ? `sugerida: ${COLOR_LABEL[rev.suggested_color]}` : "sem cor sugerida"}
        {rev.color_change_reason ? ` · mudança: ${rev.color_change_reason}` : ""}
      </span>
      <span style={muted}>{`${rev.by.name} · ${fmtDateTime(rev.created_at)}`}</span>
    </div>
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
```

`src/modules/attendance/UnitQueue.tsx`:

```diff
--- a/src/modules/attendance/UnitQueue.tsx
+++ b/src/modules/attendance/UnitQueue.tsx
@@ -1,8 +1,8 @@
 import { useState } from "react";
 import { useQuery, useQueryClient } from "@tanstack/react-query";
 import {
-  callAttendance, callNext, closeAttendance, errorCode, listUnitQueue,
-  type AppointmentRequestSummary, type AttendanceOutcome, type HealthUnit, type QueueRow
+  callAttendance, callNext, closeAttendance, errorCode, getScreening, listUnitQueue,
+  type AppointmentRequestSummary, type AttendanceOutcome, type HealthUnit, type QueueRow, type Screening
 } from "../../lib/api";
 import { ATTENDANCE_REFETCH_MS, attendanceError, splitReferenceUnits } from "../../lib/attendance";
 import { fmtDateTime, fmtHourMinute } from "../../lib/format";
@@ -12,6 +12,10 @@ import { DataTable } from "../../components/DataTable";
 import { EmptyState } from "../../components/EmptyState";
 import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
 import { FrozenTextNotice } from "../../components/FrozenTextNotice";
+import { Tag } from "../../components/Tag";
+import { COLOR_LABEL, COLOR_TONE, screeningError, waitLabel, waitedMinutes } from "../../lib/screening";
+import { ScreeningDetail, ScreeningDetailLoader } from "./ScreeningDetail";
+import { ScreeningForm } from "./ScreeningForm";
 
 // UnitQueue (Task 7) — a fila da unidade atual (spec §6 "Fila"), em duas
 // partes: "Aguardando" (ordenada pela API — prioridade, depois chegada — esta
@@ -28,8 +32,20 @@ interface Props {
   canCare: boolean;
   careBlocked?: string | null;
   onClinicalRefused?(): void;
+  now?(): Date;
 }
 
+// Módulo 18: a escuta aberta no painel — a que veio na chamada, uma lida pelo
+// id ("Ver escuta") ou a reavaliação de quem espera com destino "no dia".
+type ScreeningPanel =
+  | { kind: "called"; screening: Screening }
+  | { kind: "view"; id: string }
+  | { kind: "reassess"; row: QueueRow; screening: Screening };
+
+// Vermelho no topo e destacado (spec §4); a ordem é do api.
+const redRow = (r: QueueRow) =>
+  r.screening?.color === "red" ? { background: "var(--down-bg)", boxShadow: "inset 3px 0 0 var(--down)" } : undefined;
+
 const OUTCOME_LABEL: Record<Exclude<AttendanceOutcome, "left">, string> = {
   discharged: "Atendido e liberado",
   referred: "Encaminhado",
@@ -46,7 +62,7 @@ function handleClinicalRefusal(code: string | undefined, reload: () => void, onC
   if (code === "missing_role") void reload();
 }
 
-export function UnitQueue({ unit, units, canCare, careBlocked, onClinicalRefused }: Props) {
+export function UnitQueue({ unit, units, canCare, careBlocked, onClinicalRefused, now = () => new Date() }: Props) {
   const queryClient = useQueryClient();
   const auth = useAuth();
   const query = useQuery({ queryKey: [ "unitQueue", unit.id ], queryFn: () => listUnitQueue(unit.id),
@@ -57,6 +73,7 @@ export function UnitQueue({ unit, units, canCare, careBlocked, onClinicalRefused
   const [ actionError, setActionError ] = useState<string | null>(null);
   const [ callingNext, setCallingNext ] = useState(false);
   const [ rowBusy, setRowBusy ] = useState<string | null>(null);
+  const [ panel, setPanel ] = useState<ScreeningPanel | null>(null);
 
   function invalidate() {
     void queryClient.invalidateQueries({ queryKey: [ "unitQueue", unit.id ] });
@@ -66,7 +83,8 @@ export function UnitQueue({ unit, units, canCare, careBlocked, onClinicalRefused
     if (callingNext) return;
     setCallingNext(true); setActionError(null);
     try {
-      await callNext(unit.id);
+      const result = await callNext(unit.id);
+      if (result.screening) setPanel({ kind: "called", screening: result.screening });
       invalidate();
     } catch (err) {
       const code = errorCode(err);
@@ -92,7 +110,8 @@ export function UnitQueue({ unit, units, canCare, careBlocked, onClinicalRefused
     if (rowBusy) return;
     setRowBusy(row.id); setActionError(null);
     try {
-      await callAttendance(row.id, unit.id);
+      const result = await callAttendance(row.id, unit.id);
+      if (result.screening) setPanel({ kind: "called", screening: result.screening });
       invalidate();
     } catch (err) {
       const code = errorCode(err);
@@ -120,6 +139,19 @@ export function UnitQueue({ unit, units, canCare, careBlocked, onClinicalRefused
     }
   }
 
+  async function onReassess(row: QueueRow) {
+    if (rowBusy || !row.screening?.id) return;
+    setRowBusy(row.id); setActionError(null);
+    try {
+      setPanel({ kind: "reassess", row, screening: await getScreening(row.screening.id) });
+    } catch (err) {
+      setActionError(screeningError(err));
+    } finally {
+      setRowBusy(null);
+    }
+  }
+
+  const at = now();
   const waiting = query.data?.waiting ?? [];
   const inCare = query.data?.in_care ?? [];
 
@@ -154,9 +186,13 @@ export function UnitQueue({ unit, units, canCare, careBlocked, onClinicalRefused
                 <EmptyState title="Ninguém aguardando" />
               ) : (
                 <DataTable<QueueRow>
+                  rowStyle={redRow}
                   cols={[
                     { label: "CPF", w: "1.5fr", render: (r) => r.cpf_masked },
+                    { label: "Cor", w: "0.8fr", render: (r) => colorTag(r) },
                     { label: "Chegada", w: "1fr", render: (r) => fmtDateTime(r.checked_in_at) },
+                    { label: "Espera", w: "0.8fr", render: (r) =>
+                      waitLabel(r.screening?.waited_minutes ?? waitedMinutes(r.checked_in_at, at)) },
                     { label: "Protocolo", w: "1.5fr", render: (r) => r.protocol_name ?? "—" },
                     { label: "Prioridade", w: "1fr", render: (r) => String(r.priority ?? "—") },
                     {
@@ -166,6 +202,12 @@ export function UnitQueue({ unit, units, canCare, careBlocked, onClinicalRefused
                     {
                       label: "", w: "auto", align: "right", render: (r) => (
                         <div style={{ display: "flex", gap: 8, justifyContent: "flex-end" }}>
+                          {canCare && r.screening?.id && r.screening.destination === "same_day" && (
+                            <button type="button" disabled={rowBusy === r.id} onClick={() => void onReassess(r)}
+                              style={rowBusy === r.id ? disabledButtonStyle : secondaryButtonStyle}>
+                              Reavaliar
+                            </button>
+                          )}
                           {canCare && (
                             <button
                               type="button"
@@ -204,12 +246,21 @@ export function UnitQueue({ unit, units, canCare, careBlocked, onClinicalRefused
                 <DataTable<QueueRow>
                   cols={[
                     { label: "CPF", w: "1.5fr", render: (r) => r.cpf_masked },
+                    { label: "Cor", w: "0.8fr", render: (r) => colorTag(r) },
                     { label: "Protocolo", w: "1.5fr", render: (r) => r.protocol_name ?? "—" },
                     { label: "Prioridade", w: "1fr", render: (r) => String(r.priority ?? "—") },
                     { label: "Chamada", w: "2fr", render: (r) => `chamado por ${r.called_by_name} às ${fmtHourMinute(r.called_at)}` },
                     ...(canCare ? [ {
                       label: "", w: "auto" as const, align: "right" as const, render: (r: QueueRow) => (
-                        <button type="button" style={secondaryButtonStyle} onClick={() => setClosing(r)}>Encerrar</button>
+                        <div style={{ display: "flex", gap: 8, justifyContent: "flex-end" }}>
+                          {r.screening?.id && (
+                            <button type="button" style={secondaryButtonStyle}
+                              onClick={() => setPanel({ kind: "view", id: r.screening!.id! })}>
+                              Ver escuta
+                            </button>
+                          )}
+                          <button type="button" style={secondaryButtonStyle} onClick={() => setClosing(r)}>Encerrar</button>
+                        </div>
                       )
                     } ] : [])
                   ]}
@@ -221,6 +272,23 @@ export function UnitQueue({ unit, units, canCare, careBlocked, onClinicalRefused
           </>
         )}
 
+        {canCare && panel?.kind === "called" && <ScreeningDetail screening={panel.screening} onClose={() => setPanel(null)} />}
+        {canCare && panel?.kind === "view" && <ScreeningDetailLoader key={panel.id} id={panel.id} onClose={() => setPanel(null)} />}
+        {canCare && panel?.kind === "reassess" && (
+          <ScreeningForm
+            key={panel.screening.id}
+            mode="reassess"
+            screening={panel.screening}
+            citizenLabel={`${panel.row.cpf_masked} · chegou às ${fmtHourMinute(panel.row.checked_in_at)}`}
+            unit={unit}
+            units={units}
+            types={null}
+            onDone={() => { setPanel(null); invalidate(); setDone("Reavaliação registrada."); }}
+            onClosed={(message) => { setPanel(null); invalidate(); setActionError(message); }}
+            onCancel={() => setPanel(null)}
+          />
+        )}
+
         {closing && (
           <ClosePanel
             key={closing.id}
@@ -355,3 +423,8 @@ function ClosePanel(
 }
 
 const labelStyle = { display: "flex", flexDirection: "column" as const, gap: 4, fontSize: 12, color: "var(--ink2)" };
+
+// A recepção vê só a cor (contratos §4): nada de queixa nem sinais nesta tela.
+function colorTag(r: QueueRow) {
+  return r.screening ? <Tag tone={COLOR_TONE[r.screening.color]}>{COLOR_LABEL[r.screening.color]}</Tag> : "—";
+}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/components/DataTable.test.tsx src/modules/attendance/UnitQueue.screening.test.tsx src/modules/attendance/UnitQueue.test.tsx`
Expected: PASS (`DataTable.test.tsx` com 5, `UnitQueue.screening.test.tsx` com 5; os 34 de `UnitQueue.test.tsx` continuam verdes).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod18 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 add src/components/DataTable.tsx src/components/DataTable.test.tsx src/modules/attendance/ScreeningDetail.tsx src/modules/attendance/UnitQueue.tsx src/modules/attendance/UnitQueue.screening.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 commit -m "feat: show the risk colour and the screening in the professional queue

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 8: Escopo do acolhimento na unidade

**Files:**
- Modify: `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`
- Test: `src/modules/attendance/Units.test.tsx` (o teste "edita uma unidade" passa a esperar o quinto argumento `"walk_in"`)

**Interfaces:**
- Consumes: `updateUnit(id, name, kind, address, screeningScope?)`, `HealthUnitRow.screening_scope`, `ScreeningScope` (Task 1); `SCOPE_LABEL` (Task 2).
- Produces: `UnitFormValue.screeningScope?: ScreeningScope` — presente só na edição (a unidade nova nasce `walk_in` no api); select "Acolhimento (escuta inicial)" com as opções de `SCOPE_LABEL`; coluna "Acolhimento" na lista ("sem horário" | "todos"; ausente = `walk_in`). 422 `invalid_screening_scope` vira "escolha uma das opções de acolhimento".

- [ ] **Step 1: Write the failing test**

```diff
--- a/src/modules/attendance/Units.test.tsx
+++ b/src/modules/attendance/Units.test.tsx
@@ -66,10 +66,38 @@ describe("Units", () => {
     fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
     await waitFor(() => expect(api.updateUnit).toHaveBeenCalledWith("u1", "UBS Centro Novo", "ubs", {
       address_street: "Rua A", address_number: "1", address_complement: null, address_zip: null, neighborhood_id: "n1"
-    }));
+    }, "walk_in"));
     expect(await screen.findByText("UBS Centro Novo")).not.toBeNull();
   });
 
+  it("acolhimento (módulo 18): a lista mostra o escopo e a edição troca para todos", async () => {
+    mocked(api.listAllUnits).mockResolvedValue([ { ...rows[0], screening_scope: "walk_in" }, { ...rows[1], screening_scope: "all" } ]);
+    mocked(api.updateUnit).mockResolvedValue({ id: "u1", name: "UBS Centro", kind: "ubs", active: true, screening_scope: "all" });
+    renderUnits();
+    expect(await screen.findByText("sem horário")).not.toBeNull();
+    expect(screen.getByText("todos")).not.toBeNull();
+    fireEvent.click((await screen.findAllByRole("button", { name: "Editar" }))[0]);
+    const scope = screen.getByLabelText("Acolhimento (escuta inicial)") as HTMLSelectElement;
+    expect(scope.value).toBe("walk_in");
+    fireEvent.change(scope, { target: { value: "all" } });
+    fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
+    await waitFor(() => expect(api.updateUnit).toHaveBeenCalledWith("u1", "UBS Centro", "ubs", expect.anything(), "all"));
+  });
+
+  it("unidade nova não mostra o escopo (nasce só demanda espontânea)", async () => {
+    renderUnits();
+    fireEvent.click(await screen.findByRole("button", { name: "Nova unidade" }));
+    expect(screen.queryByLabelText("Acolhimento (escuta inicial)")).toBeNull();
+  });
+
+  it("recusa invalid_screening_scope é traduzida", async () => {
+    mocked(api.updateUnit).mockRejectedValue(new ApiError(422, { error: "invalid_screening_scope" }, "x"));
+    renderUnits();
+    fireEvent.click((await screen.findAllByRole("button", { name: "Editar" }))[0]);
+    fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
+    expect((await screen.findByRole("alert")).textContent).toBe("escolha uma das opções de acolhimento");
+  });
+
   it("desativa uma unidade ativa", async () => {
     mocked(api.setUnitActive).mockResolvedValue({ id: "u1", name: "UBS Centro", kind: "ubs", active: false });
     renderUnits();
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/modules/attendance/Units.test.tsx`
Expected: FAIL em "edita uma unidade" (`updateUnit` chamado com 4 argumentos) e em "acolhimento (módulo 18)…" (`Unable to find an element with the text: sem horário`); "unidade nova não mostra o escopo" e "recusa invalid_screening_scope é traduzida" já passam (a frase veio na Task 2) e ficam como guarda.

- [ ] **Step 3: Write minimal implementation**

```diff
--- a/src/modules/attendance/UnitForm.tsx
+++ b/src/modules/attendance/UnitForm.tsx
@@ -1,5 +1,6 @@
 import { useRef, useState } from "react";
-import type { Neighborhood } from "../../lib/api";
+import type { Neighborhood, ScreeningScope } from "../../lib/api";
+import { SCOPE_LABEL } from "../../lib/screening";
 import { UNIT_KINDS, onlyDigits } from "../../lib/attendance";
 import { matchNeighborhood, sortByName } from "../../lib/territory";
 import { maskCep, zipError, type AddressFields } from "../../lib/unitAddress";
@@ -11,7 +12,8 @@ import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } fr
 // vem preenchido, e o bairro do CEP é só sugestão. Ele pré-seleciona um
 // bairro ATIVO de mesmo nome apenas se o admin ainda não escolheu nenhum.
 // Qualquer falha deixa os campos livres.
-export interface UnitFormValue extends AddressFields { name: string; kind: string }
+// `screeningScope` (módulo 18) só existe na edição: a unidade nova nasce `walk_in` no api.
+export interface UnitFormValue extends AddressFields { name: string; kind: string; screeningScope?: ScreeningScope }
 
 interface Props {
   initial: UnitFormValue;
@@ -115,6 +117,15 @@ export function UnitForm({ initial, neighborhoods, busy, cepTimeoutMs = VIACEP_T
           {options.map((n) => <option key={n.id} value={n.id}>{n.active ? n.name : `${n.name} (inativo)`}</option>)}
         </select>
       </label>
+      {value.screeningScope !== undefined && (
+        <label style={labelStyle}>
+          Acolhimento (escuta inicial)
+          <select value={value.screeningScope} style={inputStyle}
+            onChange={(e) => set({ screeningScope: e.target.value as ScreeningScope })}>
+            {(Object.keys(SCOPE_LABEL) as ScreeningScope[]).map((k) => <option key={k} value={k}>{SCOPE_LABEL[k]}</option>)}
+          </select>
+        </label>
+      )}
       <div style={{ display: "flex", gap: 8 }}>
         <button type="button" disabled={blocked} onClick={save} style={blocked ? disabledButtonStyle : buttonStyle}>Salvar</button>
         <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Cancelar</button>
```

```diff
--- a/src/modules/attendance/Units.tsx
+++ b/src/modules/attendance/Units.tsx
@@ -1,6 +1,8 @@
 import { useEffect, useState } from "react";
 import { useQuery, useQueryClient } from "@tanstack/react-query";
-import { createUnit, listAllUnits, listNeighborhoods, setUnitActive, updateUnit, type HealthUnitRow } from "../../lib/api";
+import {
+  createUnit, listAllUnits, listNeighborhoods, setUnitActive, updateUnit, type HealthUnitRow, type ScreeningScope
+} from "../../lib/api";
 import { UNIT_KINDS, attendanceError } from "../../lib/attendance";
 import { NEIGHBORHOODS_KEY } from "../../lib/territory";
 import { EMPTY_ADDRESS_FIELDS, addressFieldsFrom, addressPayload, formatAddress } from "../../lib/unitAddress";
@@ -18,6 +20,8 @@ import { DrainUnitPanel } from "./DrainUnitPanel";
 // destino de encaminhamento em UnitQueue): invalida a query `activeUnits`
 // para essas telas recarregarem (card dashboard#4, item 1 — Task 7).
 const KIND_LABEL: Record<string, string> = Object.fromEntries(UNIT_KINDS.map((k) => [ k.value, k.label ]));
+// Módulo 18 (contratos §5): quem passa pela escuta inicial nesta unidade.
+const SCOPE_SHORT: Record<ScreeningScope, string> = { walk_in: "sem horário", all: "todos" };
 
 interface Editing { id: string | null; initial: UnitFormValue }
 
@@ -61,7 +65,7 @@ export function Units() {
     try {
       const address = addressPayload(value);
       if (editing.id) {
-        await updateUnit(editing.id, value.name, value.kind, address);
+        await updateUnit(editing.id, value.name, value.kind, address, value.screeningScope);
       } else {
         await createUnit(value.name, value.kind, address);
         void queryClient.invalidateQueries({ queryKey: [ "activeUnits" ] });
@@ -127,12 +131,15 @@ export function Units() {
               { label: "Tipo", w: "1fr", render: (r) => KIND_LABEL[r.kind] ?? r.kind },
               { label: "Endereço", w: "3fr", render: (r) =>
                 formatAddress(r, r.neighborhood_id ? nameOf.get(r.neighborhood_id) : null) },
+              { label: "Acolhimento", w: "1.2fr", render: (r) => SCOPE_SHORT[r.screening_scope ?? "walk_in"] },
               { label: "Situação", w: "1fr", render: (r) => <Tag tone={r.active ? "ok" : undefined}>{r.active ? "ativa" : "inativa"}</Tag> },
               {
                 label: "", w: "auto", align: "right", render: (r) => (
                   <div style={{ display: "flex", gap: 8, justifyContent: "flex-end" }}>
                     <button type="button" style={secondaryButtonStyle}
-                      onClick={() => setEditing({ id: r.id, initial: { name: r.name, kind: r.kind, ...addressFieldsFrom(r) } })}>
+                      onClick={() => setEditing({ id: r.id, initial: {
+                        name: r.name, kind: r.kind, ...addressFieldsFrom(r), screeningScope: r.screening_scope ?? "walk_in"
+                      } })}>
                       Editar
                     </button>
                     {r.active && ((r.live_requests_count ?? 0) + (r.live_appointments_count ?? 0)) > 0 && (
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/modules/attendance/Units.test.tsx src/modules/attendance/UnitForm.test.tsx`
Expected: PASS (`Units.test.tsx` com 16; `UnitForm.test.tsx` sem mudança, verde).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod18 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 add src/modules/attendance/UnitForm.tsx src/modules/attendance/Units.tsx src/modules/attendance/Units.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 commit -m "feat: edit the unit screening scope

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 9: Sinais vitais e CIAP-2 no construtor de condições

**Files:**
- Modify: `src/lib/condition.ts` (`ConditionContext`, `ConditionField.codes`, `RESERVED_PREFIXES`, `fieldsFor`, `rowProblem`), `src/lib/conditionPhrase.ts` (`valueLabel`), `src/modules/protocols/ConditionBuilder.tsx` (`ValueEditor` e `CodesInput`)
- Test: `src/lib/condition.test.ts`, `src/modules/protocols/ConditionBuilder.test.tsx`

**Interfaces:**
- Consumes: `VITALS`, `GLUCOSE_MOMENTS`, `GLUCOSE_MOMENT_LABEL` (Task 2).
- Produces:
  - `ConditionContext` com `"screening"`; `fieldsFor("screening")` = os 10 sinais (`vitals.<campo>`, número, rótulo em minúscula, unidade e limites da escuta), `vitals.bmi` ("IMC"), `vitals.glucose_moment` (escolha), `complaint.ciap2` ("queixa (CIAP-2)", escolha por códigos) e, por último, `profile.age` e `profile.sex` — os sinais vêm primeiro porque o "+ condição" usa o primeiro campo;
  - `ConditionField.codes?: { pattern: RegExp; example: string }` — campo de escolha sem lista fechada; `rowProblem` diz "informe ao menos um código" e "código inválido: <X> (ex.: K86, A03)"; a frase mostra o código como veio;
  - `RESERVED_PREFIXES` com `"vitals."` e `"complaint."` (passo com esse prefixo não vira campo de resposta);
  - no `ConditionBuilder`, a linha de códigos é um campo de texto "códigos": separa por vírgula, ponto e vírgula ou espaço, põe em maiúsculas e tira repetidos.
- Sem `lt`/`gt` no construtor (como hoje): "SpO2 abaixo de 90" é `lte 89`; regra com `lt`/`gt` aparece como "regra avançada".

- [ ] **Step 1: Write the failing test**

```diff
--- a/src/lib/condition.test.ts
+++ b/src/lib/condition.test.ts
@@ -9,7 +9,8 @@ import { NEIGHBORHOODS, SUGGESTION_DEF } from "../test/conditionFixtures";
 const FIELDS: Record<ConditionContext, ConditionField[]> = {
   eligibility: fieldsFor("eligibility"),
   suggestion: fieldsFor("suggestion", { definition: SUGGESTION_DEF }),
-  restriction: fieldsFor("restriction", { neighborhoods: NEIGHBORHOODS })
+  restriction: fieldsFor("restriction", { neighborhoods: NEIGHBORHOODS }),
+  screening: fieldsFor("screening")
 };
 const field = (context: ConditionContext, id: string) => FIELDS[context].find((f) => f.id === id)!;
 const AGE = "profile.age";
@@ -85,6 +86,43 @@ const CANONICAL: Array<[ ConditionContext, string, unknown ]> = [
   [ "restriction", "idade e bairro", { all: [ { gte: [ AGE, 60 ] }, { in: [ "citizen.neighborhood_id", [ "n2" ] ] } ] } ]
 ];
 
+describe("acolhimento (módulo 18)", () => {
+  it("sinais vitais, queixa e perfil, com os limites da escuta", () => {
+    expect(FIELDS.screening.map((f) => f.id)).toEqual([
+      "vitals.systolic", "vitals.diastolic", "vitals.heart_rate", "vitals.respiratory_rate", "vitals.temperature_c", "vitals.spo2",
+      "vitals.capillary_glucose", "vitals.weight_kg", "vitals.height_cm", "vitals.pain_score", "vitals.bmi", "vitals.glucose_moment",
+      "complaint.ciap2", "profile.age", "profile.sex"
+    ]);
+    expect(field("screening", "vitals.systolic")).toMatchObject({ label: "pressão sistólica", unit: " mmHg", min: 50, max: 300 });
+    expect(field("screening", "vitals.spo2").unit).toBe("%");
+    expect(field("screening", "vitals.pain_score").unit).toBeUndefined();
+  });
+
+  it("decimal na temperatura vai e volta; fora do plausível é dito", () => {
+    const tree = { gte: [ "vitals.temperature_c", 37.8 ] };
+    const parsed = fromTree(tree, FIELDS.screening);
+    expect(parsed.ok && toTree(parsed.root)).toEqual(tree);
+    const row = { ...newRow(field("screening", "vitals.spo2"), "lte"), value: 120 } as ConditionRow;
+    expect(rowProblem(row, field("screening", "vitals.spo2"))).toBe("use um valor entre 50 e 100");
+  });
+
+  it("CIAP-2: lista de códigos com o padrão conferido", () => {
+    const ciap = field("screening", "complaint.ciap2");
+    const parsed = fromTree({ in: [ "complaint.ciap2", [ "K86", "K87" ] ] }, FIELDS.screening);
+    expect(parsed.ok && parsed.root.children[0]).toMatchObject({ field: "complaint.ciap2", op: "in", value: [ "K86", "K87" ] });
+    const empty = newRow(ciap);
+    expect(rowProblem(empty, ciap)).toBe("informe ao menos um código");
+    expect(rowProblem({ ...empty, value: [ "K86", "hipertensão" ] } as ConditionRow, ciap)).toBe("código inválido: hipertensão (ex.: K86, A03)");
+  });
+
+  it("passo com prefixo vitals. ou complaint. não vira campo de resposta", () => {
+    const def = { steps: [ { id: "vitals.pa", prompt: "PA?", answer_type: "integer" }, { id: "complaint.x", prompt: "Q?", answer_type: "boolean" } ] };
+    const ids = fieldsFor("suggestion", { definition: def }).map((f) => f.id);
+    expect(ids).not.toContain("vitals.pa");
+    expect(ids).not.toContain("complaint.x");
+  });
+});
+
 describe("ida e volta", () => {
   it.each(CANONICAL)("%s / %s: árvore → modelo → árvore é a mesma", (context, _name, tree) => {
     const parsed = fromTree(tree, FIELDS[context]);
```

```diff
--- a/src/modules/protocols/ConditionBuilder.test.tsx
+++ b/src/modules/protocols/ConditionBuilder.test.tsx
@@ -25,6 +25,35 @@ function Harness({ initial, fields = ELIG, onTree }: { initial: unknown; fields?
 const lastTree = (fn: ReturnType<typeof vi.fn>) => fn.mock.calls[fn.mock.calls.length - 1][0];
 const row = (n: number) => screen.getByRole("group", { name: `condição ${n}` });
 
+describe("ConditionBuilder — acolhimento (módulo 18)", () => {
+  const SCREENING = fieldsFor("screening");
+
+  it("sistólica a partir de 180 vira árvore e frase com a unidade", () => {
+    const onTree = vi.fn();
+    render(<Harness initial={undefined} fields={SCREENING} onTree={onTree} />);
+    fireEvent.click(screen.getByRole("button", { name: "+ condição" }));
+    fireEvent.change(within(row(1)).getByLabelText("valor"), { target: { value: "180" } });
+    expect(lastTree(onTree)).toEqual({ gte: [ "vitals.systolic", 180 ] });
+    expect(screen.getByText("pressão sistólica a partir de 180 mmHg")).not.toBeNull();
+  });
+
+  it("CIAP-2 por códigos digitados, em maiúsculas e sem repetir", () => {
+    const onTree = vi.fn();
+    render(<Harness initial={undefined} fields={SCREENING} onTree={onTree} />);
+    fireEvent.click(screen.getByRole("button", { name: "+ condição" }));
+    fireEvent.change(within(row(1)).getByLabelText("campo"), { target: { value: "complaint.ciap2" } });
+    fireEvent.change(within(row(1)).getByLabelText("códigos"), { target: { value: "k86, A03 k86" } });
+    expect(lastTree(onTree)).toEqual({ in: [ "complaint.ciap2", [ "K86", "A03" ] ] });
+    expect(screen.getByText("queixa (CIAP-2) é K86 ou A03")).not.toBeNull();
+  });
+
+  it("código fora do padrão aparece no motivo da linha", () => {
+    render(<Harness initial={{ in: [ "complaint.ciap2", [ "K86" ] ] }} fields={SCREENING} onTree={vi.fn()} />);
+    fireEvent.change(within(row(1)).getByLabelText("códigos"), { target: { value: "K86, febre" } });
+    expect(within(row(1)).getByText("código inválido: FEBRE (ex.: K86, A03)")).not.toBeNull();
+  });
+});
+
 describe("ConditionBuilder", () => {
   it("vazio diz 'para todos'; idade a partir de 60 vira a árvore e a frase", () => {
     const onTree = vi.fn();
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/lib/condition.test.ts src/modules/protocols/ConditionBuilder.test.tsx`
Expected: FAIL — `fieldsFor("screening")` devolve os campos de sugestão (lista diferente), `rowProblem` diz "marque ao menos uma opção", e no construtor `Unable to find a label with the text of: códigos`. (`tsc` também acusa o `Record<ConditionContext, …>` do teste até o tipo ganhar `"screening"`.)

- [ ] **Step 3: Write minimal implementation**

```diff
--- a/src/lib/condition.ts
+++ b/src/lib/condition.ts
@@ -15,8 +15,9 @@
 // Uma raiz E com uma linha só sai como a linha sozinha ({gte:["profile.age",60]}).
 import type { ConditionTree, PanelNeighborhood } from "./api";
 import { SEX_OPTIONS } from "./profile";
+import { GLUCOSE_MOMENTS, GLUCOSE_MOMENT_LABEL, VITALS } from "./screening";
 
-export type ConditionContext = "eligibility" | "suggestion" | "restriction";
+export type ConditionContext = "eligibility" | "suggestion" | "restriction" | "screening";
 export type FieldKind = "number" | "choice" | "boolean";
 export interface FieldOption { value: string; label: string; inactive?: boolean }
 export interface ConditionField {
@@ -31,6 +32,9 @@ export interface ConditionField {
   options?: FieldOption[];
   // Entre o rótulo e o valor na frase: " " para "sexo feminino", " é " para respostas.
   verb?: string;
+  // Campo de escolha sem lista fechada (CIAP-2, módulo 18): códigos digitados,
+  // conferidos pelo padrão; o gate do api confere se existem.
+  codes?: { pattern: RegExp; example: string };
 }
 
 export type ConditionRow =
@@ -49,7 +53,7 @@ export interface ConditionGroup {
 export type ConditionNode = ConditionRow | ConditionGroup;
 export type ParsedCondition = { ok: true; root: ConditionGroup } | { ok: false };
 
-export const RESERVED_PREFIXES = [ "profile.", "outcome.", "citizen." ];
+export const RESERVED_PREFIXES = [ "profile.", "outcome.", "citizen.", "vitals.", "complaint." ];
 export const BOOLEAN_OPTIONS: FieldOption[] = [ { value: "true", label: "sim" }, { value: "false", label: "não" } ];
 export const OP_LABELS: Record<RowOp, string> = {
   gte: "a partir de", lte: "até", between: "entre", eq: "igual a", in: "é um de", is: "é"
@@ -67,8 +71,26 @@ const SEX: ConditionField = {
 
 export interface FieldSources { definition?: unknown; neighborhoods?: PanelNeighborhood[] }
 
+// Acolhimento (módulo 18; contratos §1): sinais vitais, queixa e perfil. Os
+// limites de valor são os de plausibilidade da escuta.
+const lowerFirst = (s: string) => s.charAt(0).toLowerCase() + s.slice(1);
+const VITAL_FIELDS: ConditionField[] = [
+  ...VITALS.map((v): ConditionField => ({
+    id: `vitals.${v.key}`, label: lowerFirst(v.label), kind: "number", group: "Sinais vitais",
+    unit: v.unit === "" ? undefined : v.unit === "%" ? "%" : ` ${v.unit}`, min: v.min, max: v.max
+  })),
+  { id: "vitals.bmi", label: "IMC", kind: "number", group: "Sinais vitais" },
+  { id: "vitals.glucose_moment", label: "momento da glicemia", kind: "choice", group: "Sinais vitais",
+    options: GLUCOSE_MOMENTS.map((m) => ({ value: m, label: GLUCOSE_MOMENT_LABEL[m] })) }
+];
+const CIAP2: ConditionField = {
+  id: "complaint.ciap2", label: "queixa (CIAP-2)", kind: "choice", group: "Queixa", verb: " é ",
+  codes: { pattern: /^[A-Z]\d{2}$/, example: "K86, A03" }
+};
+
 export function fieldsFor(context: ConditionContext, sources: FieldSources = {}): ConditionField[] {
   if (context === "eligibility") return [ AGE, SEX ];
+  if (context === "screening") return [ ...VITAL_FIELDS, CIAP2, AGE, SEX ];
   if (context === "restriction") return [ AGE, SEX, neighborhoodField(sources.neighborhoods ?? []) ];
   return [ AGE, SEX, ...stepFields(sources.definition), ...outcomeFields(sources.definition) ];
 }
@@ -189,6 +211,11 @@ export function rowProblem(row: ConditionRow, field: ConditionField | undefined)
     ((field.min !== undefined && n < field.min) || (field.max !== undefined && n > field.max)));
   if (outOfRange) return `use um valor entre ${field.min ?? "…"} e ${field.max ?? "…"}`;
   if (row.op === "between" && (row.value[0] as number) >= (row.value[1] as number)) return "o início precisa ser menor que o fim";
+  if (row.op === "in" && field.codes) {
+    if (row.value.length === 0) return "informe ao menos um código";
+    const bad = row.value.find((v) => !field.codes!.pattern.test(v));
+    if (bad !== undefined) return `código inválido: ${bad} (ex.: ${field.codes.example})`;
+  }
   if (row.op === "in" && row.value.length === 0) return "marque ao menos uma opção";
   return null;
 }
```

```diff
--- a/src/lib/conditionPhrase.ts
+++ b/src/lib/conditionPhrase.ts
@@ -38,7 +38,7 @@ function withUnit(v: unknown, field?: ConditionField): string {
 
 function valueLabel(v: unknown, field?: ConditionField): string {
   const s = String(v);
-  if (!field?.options) return s;
+  if (!field?.options || field.codes) return s;
   return field.options.find((o) => o.value === s)?.label ?? `${s} (fora da lista)`;
 }
```

```diff
--- a/src/modules/protocols/ConditionBuilder.tsx
+++ b/src/modules/protocols/ConditionBuilder.tsx
@@ -176,6 +176,8 @@ function ValueEditor({ row, field, onChange }: { row: ConditionRow; field: Condi
     );
   }
 
+  if (row.op === "in" && field.codes) return <CodesInput row={row} example={field.codes.example} onChange={onChange} />;
+
   // Opção inativa só aparece se já estiver marcada (bairro desativado depois
   // da regra); valor que nem existe na lista aparece como "fora da lista".
   if (row.op !== "in") return null;
@@ -199,6 +201,22 @@ function ValueEditor({ row, field, onChange }: { row: ConditionRow; field: Condi
   );
 }
 
+// Códigos digitados (CIAP-2): o texto fica local; a lista vai em maiúsculas,
+// sem repetir. Código fora do padrão aparece no motivo da linha (rowProblem).
+function CodesInput({ row, example, onChange }: {
+  row: Extract<ConditionRow, { op: "in" }>; example: string; onChange(next: ConditionRow): void;
+}) {
+  const [ text, setText ] = useState(row.value.join(", "));
+  return (
+    <input aria-label="códigos" value={text} placeholder={example} style={{ ...inputStyle, width: 160, marginTop: 0, padding: 6, fontSize: 12.5 }}
+      onChange={(e) => {
+        setText(e.target.value);
+        const codes = e.target.value.split(/[\s,;]+/).map((c) => c.trim().toUpperCase()).filter((c) => c !== "");
+        onChange({ ...row, value: [ ...new Set(codes) ] });
+      }} />
+  );
+}
+
 const box: CSSProperties = { border: "1px solid var(--rule)", borderRadius: 8, padding: "8px 12px", margin: 0, display: "flex", flexDirection: "column", gap: 8 };
 const legend: CSSProperties = { fontSize: 13, fontWeight: 600 };
 const phraseStyle: CSSProperties = { margin: 0, fontSize: 12.5 };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/lib/condition.test.ts src/modules/protocols/ConditionBuilder.test.tsx src/lib/conditionPhrase.test.ts src/modules/protocolEditor`
Expected: PASS (`condition.test.ts` 58, `ConditionBuilder.test.tsx` 13, `conditionPhrase.test.ts` 8; os painéis do editor que já usam o construtor continuam verdes).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod18 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 add src/lib/condition.ts src/lib/condition.test.ts src/lib/conditionPhrase.ts src/modules/protocols/ConditionBuilder.tsx src/modules/protocols/ConditionBuilder.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 commit -m "feat: add vital signs and CIAP-2 fields to the condition builder

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 10: Protocolo de acolhimento no editor (regras de cor e simulador)

**Files:**
- Create: `src/lib/riskRules.ts`, `src/modules/protocolEditor/RiskRulesPanel.tsx`, `src/modules/protocolEditor/ScreeningSimulator.tsx`
- Modify: `src/modules/ProtocolEditor.tsx`
- Test: `src/lib/riskRules.test.ts`, `src/modules/protocolEditor/RiskRulesPanel.test.tsx`, `src/modules/protocolEditor/ScreeningSimulator.test.tsx`, `src/modules/ProtocolEditor.test.tsx`

**Interfaces:**
- Consumes: `simulateScreening`, `ScreeningColor`, `ConditionTree`, `Sex` (Task 1 e existentes); `COLORS`, `COLOR_LABEL`, `COLOR_HINT`, `COLOR_TONE`, `EMPTY_VITALS_FORM`, `parseVitals`, `bmiOf` (Task 2); `VitalSignsFields` (Task 4); `fieldsFor("screening")`, `ConditionBuilder` (Task 9); `MAX_AGE`, `SEX_OPTIONS` (`src/lib/profile.ts`).
- Produces:
  - `src/lib/riskRules.ts`: `interface RiskRuleDraft { when: ConditionTree | null; color: ScreeningColor }`, `type RiskRead`, `RISK_RULES_MAX = 50`, `SCREENING_NAME = "acolhimento"`, `isScreeningDefinition(def): boolean` (`kind === "screening"`), `readRiskRules(def): RiskRead`, `writeRiskRules(def, rules): unknown` (lista vazia continua no JSON; condição incompleta sai ausente), `riskRuleProblem(rule)`, `SCREENING_TEMPLATE` (as regras iniciais da spec §3.3: sistólica ≥ 180 ou SpO2 ≤ 89 → vermelho; temperatura ≥ 39 ou glicemia ≥ 300 → amarelo);
  - `RiskRulesPanel({ definition: unknown | null; onChange(next: unknown): void })` — seção "Regras de cor"; um grupo "regra de cor N" com select "Cor sugerida" e o construtor "Quando sugerir"; "+ regra de cor" (trava em 50) e "remover regra";
  - `ScreeningSimulator({ definition: unknown | null; valid: boolean })` — seção "Simulador do acolhimento" com Idade, Sexo, "Código CIAP-2" e os `VitalSignsFields`; "Simular" chama `simulateScreening` (Divergência D4) e mostra a cor, "regra N: <frase>", avisos e erros; resultado some quando a entrada muda;
  - `ProtocolEditor`: opção "Novo acolhimento (modelo)" (`__new_screening__`) no seletor; com `kind: "screening"`, a coluna da direita vira "Regras de cor (acolhimento)" + simulador, sem "Oferta e sugestões", "Agendamento", preview nem simulador de perfil.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/riskRules.test.ts
import { describe, expect, it } from "vitest";
import { SCREENING_TEMPLATE, isScreeningDefinition, readRiskRules, riskRuleProblem, writeRiskRules } from "./riskRules";

const DEF = { name: "acolhimento", version: 1, kind: "screening", risk_rules: [ { when: { gte: [ "vitals.systolic", 180 ] }, color: "red" } ] };

describe("regras de cor do acolhimento", () => {
  it("reconhece a variante só por kind: screening", () => {
    expect(isScreeningDefinition(DEF)).toBe(true);
    expect(isScreeningDefinition({ name: "x", steps: [] })).toBe(false);
    expect(isScreeningDefinition({ kind: "triage" })).toBe(false);
    expect(isScreeningDefinition(null)).toBe(false);
  });

  it("lê as regras; ausente é lista vazia; formato errado diz o motivo", () => {
    expect(readRiskRules(DEF)).toEqual({ ok: true, rules: [ { when: { gte: [ "vitals.systolic", 180 ] }, color: "red" } ] });
    expect(readRiskRules({ kind: "screening" })).toEqual({ ok: true, rules: [] });
    expect(readRiskRules({ kind: "screening", risk_rules: {} })).toEqual({ ok: false, reason: "“risk_rules” não é uma lista: corrija no JSON" });
    expect(readRiskRules({ kind: "screening", risk_rules: [ { when: {}, color: "orange" } ] }).ok).toBe(false);
  });

  it("escreve sem tocar no resto; lista vazia continua no JSON; condição incompleta sai ausente", () => {
    const next = writeRiskRules(DEF, [ { when: null, color: "yellow" } ]) as Record<string, unknown>;
    expect(next.name).toBe("acolhimento");
    expect(next.risk_rules).toEqual([ { color: "yellow" } ]);
    expect((writeRiskRules(DEF, []) as Record<string, unknown>).risk_rules).toEqual([]);
    expect(riskRuleProblem({ when: null, color: "red" })).toBe("defina quando sugerir esta cor");
  });

  it("o modelo é uma definição de acolhimento com as regras iniciais", () => {
    const t = JSON.parse(SCREENING_TEMPLATE);
    expect(isScreeningDefinition(t)).toBe(true);
    expect(t.name).toBe("acolhimento");
    expect(t.risk_rules.map((r: { color: string }) => r.color)).toEqual([ "red", "red", "yellow", "yellow" ]);
    expect("steps" in t).toBe(false);
  });
});
```

```tsx
// src/modules/protocolEditor/RiskRulesPanel.test.tsx
import { afterEach, describe, expect, it } from "vitest";
import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
import { useState } from "react";
import { RiskRulesPanel } from "./RiskRulesPanel";

afterEach(cleanup);

const DEF = { name: "acolhimento", version: 1, kind: "screening", risk_rules: [ { when: { gte: [ "vitals.systolic", 180 ] }, color: "red" } ] };

function Harness({ initial }: { initial: unknown }) {
  const [ def, setDef ] = useState<unknown>(initial);
  return (
    <>
      <RiskRulesPanel definition={def} onChange={setDef} />
      <pre data-testid="json">{JSON.stringify(def)}</pre>
    </>
  );
}
const json = () => JSON.parse(screen.getByTestId("json").textContent ?? "null");
const rule = (n: number) => screen.getByRole("group", { name: `regra de cor ${n}` });

describe("RiskRulesPanel", () => {
  it("mostra a regra do JSON em frase", () => {
    render(<Harness initial={DEF} />);
    expect(within(rule(1)).getByText("pressão sistólica a partir de 180 mmHg")).not.toBeNull();
    expect((within(rule(1)).getByLabelText("Cor sugerida") as HTMLSelectElement).value).toBe("red");
  });

  it("+ regra monta cor e condição de saturação no JSON", () => {
    render(<Harness initial={DEF} />);
    fireEvent.click(screen.getByRole("button", { name: "+ regra de cor" }));
    expect(within(rule(2)).getByText("defina quando sugerir esta cor")).not.toBeNull();
    fireEvent.change(within(rule(2)).getByLabelText("Cor sugerida"), { target: { value: "red" } });
    const when = within(rule(2)).getByRole("group", { name: "Quando sugerir" });
    fireEvent.click(within(when).getByRole("button", { name: "+ condição" }));
    fireEvent.change(within(when).getByLabelText("campo"), { target: { value: "vitals.spo2" } });
    fireEvent.change(within(when).getByLabelText("operador"), { target: { value: "lte" } });
    fireEvent.change(within(when).getByLabelText("valor"), { target: { value: "89" } });
    expect(json().risk_rules[1]).toEqual({ when: { lte: [ "vitals.spo2", 89 ] }, color: "red" });
  });

  it("remover a última deixa a lista vazia no JSON e pede uma regra", () => {
    render(<Harness initial={DEF} />);
    fireEvent.click(within(rule(1)).getByRole("button", { name: "remover regra" }));
    expect(json().risk_rules).toEqual([]);
    expect(screen.getByRole("alert").textContent).toBe("Inclua ao menos uma regra de cor.");
  });

  it("formato errado no JSON: diz o motivo e não reescreve", () => {
    render(<Harness initial={{ ...DEF, risk_rules: [ { when: {}, color: "laranja" } ] }} />);
    expect(screen.getByRole("alert").textContent).toBe("uma regra de cor está fora do formato { when, color }: corrija no JSON");
    expect(json().risk_rules[0].color).toBe("laranja");
  });

  it("50 regras travam o + regra", () => {
    const fifty = Array.from({ length: 50 }, (_, i) => ({ when: { gte: [ "vitals.heart_rate", 100 + i ] }, color: "yellow" }));
    render(<Harness initial={{ ...DEF, risk_rules: fifty }} />);
    expect((screen.getByRole("button", { name: "+ regra de cor" }) as HTMLButtonElement).disabled).toBe(true);
  });
});
```

```tsx
// src/modules/protocolEditor/ScreeningSimulator.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, simulateScreening: vi.fn() };
});

import * as api from "../../lib/api";
import { ScreeningSimulator } from "./ScreeningSimulator";
import { suggestion } from "../../test/screeningFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const DEF = { name: "acolhimento", version: 1, kind: "screening", risk_rules: [ { when: { gte: [ "vitals.systolic", 180 ] }, color: "red" } ] };

describe("ScreeningSimulator", () => {
  beforeEach(() => {
    mocked(api.simulateScreening).mockReset();
    mocked(api.simulateScreening).mockResolvedValue({ ...suggestion(), errors: [], warnings: [] });
  });

  it("manda a definição do editor, o perfil, a queixa e os sinais; mostra a cor e as regras", async () => {
    render(<ScreeningSimulator definition={DEF} valid />);
    fireEvent.change(screen.getByLabelText("Idade"), { target: { value: "70" } });
    fireEvent.change(screen.getByLabelText("Código CIAP-2"), { target: { value: "k86" } });
    fireEvent.change(screen.getByLabelText("Pressão sistólica (mmHg)"), { target: { value: "185" } });
    fireEvent.change(screen.getByLabelText("Pressão diastólica (mmHg)"), { target: { value: "110" } });
    fireEvent.click(screen.getByRole("button", { name: "Simular" }));
    await waitFor(() => expect(api.simulateScreening).toHaveBeenCalledWith({
      definition: DEF, ciap2_code: "K86", vitals: { systolic: 185, diastolic: 110 }, profile: { age: 70, sex: "female" }
    }));
    expect((await screen.findByRole("status")).textContent).toContain("Cor sugerida: vermelho · atendimento imediato");
    expect(screen.getByText("regra 1: pressão sistólica a partir de 180 mmHg")).not.toBeNull();
    expect(screen.getByText("Pressão sistólica: acima da faixa de alerta")).not.toBeNull();
  });

  it("nenhuma regra casou; erros do gate aparecem", async () => {
    mocked(api.simulateScreening).mockResolvedValue({ suggested_color: null, matched_rules: [], alerts: [], bmi: null,
      errors: [ "schema: /risk_rules minItems" ], warnings: [] });
    render(<ScreeningSimulator definition={DEF} valid />);
    fireEvent.click(screen.getByRole("button", { name: "Simular" }));
    expect(await screen.findByText("Nenhuma regra casou: sem cor sugerida")).not.toBeNull();
    expect(screen.getByText("schema: /risk_rules minItems")).not.toBeNull();
  });

  it("código CIAP-2 fora do padrão ou sinal implausível travam", () => {
    render(<ScreeningSimulator definition={DEF} valid />);
    fireEvent.change(screen.getByLabelText("Código CIAP-2"), { target: { value: "febre" } });
    expect((screen.getByRole("button", { name: "Simular" }) as HTMLButtonElement).disabled).toBe(true);
    fireEvent.change(screen.getByLabelText("Código CIAP-2"), { target: { value: "" } });
    fireEvent.change(screen.getByLabelText("Saturação (SpO2) (%)"), { target: { value: "130" } });
    expect((screen.getByRole("button", { name: "Simular" }) as HTMLButtonElement).disabled).toBe(true);
  });

  it("mudar a entrada esconde o resultado antigo", async () => {
    render(<ScreeningSimulator definition={DEF} valid />);
    fireEvent.click(screen.getByRole("button", { name: "Simular" }));
    expect(await screen.findByRole("status")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Idade"), { target: { value: "30" } });
    expect(screen.queryByRole("status")).toBeNull();
  });
});
```

Em `src/modules/ProtocolEditor.test.tsx`:

```diff
--- a/src/modules/ProtocolEditor.test.tsx
+++ b/src/modules/ProtocolEditor.test.tsx
@@ -28,6 +28,36 @@ const definitionBox = () => screen.getAllByRole("textbox")[0] as HTMLTextAreaEle
 const typeDefinition = (value: unknown) =>
   fireEvent.change(definitionBox(), { target: { value: JSON.stringify(value, null, 2) } });
 
+describe("ProtocolEditor — acolhimento (módulo 18)", () => {
+  beforeEach(() => {
+    vi.resetAllMocks();
+    mocked(api.listAuthorProtocols).mockResolvedValue([]);
+    mocked(api.gateProtocol).mockResolvedValue({ valid: true });
+    mocked(api.listAppointmentTypes).mockResolvedValue([]);
+  });
+
+  it("o modelo de acolhimento troca a coluna pelas regras de cor e o simulador", async () => {
+    render(<ProtocolEditor />);
+    expect(screen.getByText("Oferta e sugestões")).not.toBeNull();
+    fireEvent.change(screen.getAllByRole("combobox")[0], { target: { value: "__new_screening__" } });
+    expect(JSON.parse(definitionBox().value).kind).toBe("screening");
+    expect(screen.getByRole("region", { name: "Regras de cor" })).not.toBeNull();
+    expect(screen.getByRole("region", { name: "Simulador do acolhimento" })).not.toBeNull();
+    expect(screen.queryByText("Oferta e sugestões")).toBeNull();
+    expect(screen.queryByRole("region", { name: "Agendamento" })).toBeNull();
+    await waitFor(() => expect(api.gateProtocol).toHaveBeenCalled());
+  });
+
+  it("editar a cor no painel grava no JSON", () => {
+    render(<ProtocolEditor />);
+    typeDefinition({ name: "acolhimento", version: 2, kind: "screening",
+      risk_rules: [ { when: { gte: [ "vitals.systolic", 180 ] }, color: "red" } ] });
+    fireEvent.change(within(screen.getByRole("group", { name: "regra de cor 1" })).getByLabelText("Cor sugerida"),
+      { target: { value: "yellow" } });
+    expect(JSON.parse(definitionBox().value).risk_rules[0].color).toBe("yellow");
+  });
+});
+
 describe("ProtocolEditor — Usar em Analytics", () => {
   beforeEach(() => {
     vi.resetAllMocks();
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/lib/riskRules.test.ts src/modules/protocolEditor src/modules/ProtocolEditor.test.tsx`
Expected: FAIL — `Failed to resolve import "./riskRules"`, `"./RiskRulesPanel"`, `"./ScreeningSimulator"`; no editor, a opção `__new_screening__` não existe (o texto não muda).

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/riskRules.ts
// Protocolo de acolhimento (módulo 18; schema protocols-v1.6.0, contratos §1):
// `kind: "screening"` com `risk_rules: [{ when, color }]` (1 a 50). Mesmo padrão
// de schedulingRules.ts: o JSON do editor é a fonte; estas funções leem a
// definição e devolvem uma nova sem tocar no resto.
import type { ConditionTree, ScreeningColor } from "./api";
import { COLORS } from "./screening";

export interface RiskRuleDraft { when: ConditionTree | null; color: ScreeningColor }
export type RiskRead = { ok: true; rules: RiskRuleDraft[] } | { ok: false; reason: string };

export const RISK_RULES_MAX = 50;
export const SCREENING_NAME = "acolhimento";

type Obj = Record<string, unknown>;
const isObj = (v: unknown): v is Obj => !!v && typeof v === "object" && !Array.isArray(v);

export function isScreeningDefinition(definition: unknown): boolean {
  return isObj(definition) && definition.kind === "screening";
}

export function readRiskRules(definition: unknown): RiskRead {
  if (!isObj(definition)) return { ok: false, reason: "a definição precisa ser um objeto JSON" };
  const list = definition.risk_rules;
  if (list === undefined) return { ok: true, rules: [] };
  if (!Array.isArray(list)) return { ok: false, reason: "“risk_rules” não é uma lista: corrija no JSON" };
  const malformed = list.some((r) => !isObj(r) || (r.when !== undefined && !isObj(r.when)) ||
    !COLORS.includes(r.color as ScreeningColor));
  if (malformed) return { ok: false, reason: "uma regra de cor está fora do formato { when, color }: corrija no JSON" };
  return { ok: true, rules: (list as Obj[]).map((r) => ({ when: (r.when as ConditionTree | undefined) ?? null, color: r.color as ScreeningColor })) };
}

// `risk_rules` é obrigatório na variante: a lista vazia fica no JSON e o gate
// aponta o mínimo de 1. Condição incompleta sai ausente (o gate aponta).
export function writeRiskRules(definition: unknown, rules: RiskRuleDraft[]): unknown {
  return { ...(definition as Obj), risk_rules: rules.map((r) => (r.when === null ? { color: r.color } : { when: r.when, color: r.color })) };
}

export function riskRuleProblem(rule: RiskRuleDraft): string | null {
  return rule.when === null ? "defina quando sugerir esta cor" : null;
}

// Rascunho inicial (spec §3.3 e §10): ponto de partida que a cidade revisa e assina.
export const SCREENING_TEMPLATE = JSON.stringify(
  {
    name: SCREENING_NAME,
    version: 1,
    kind: "screening",
    risk_rules: [
      { when: { gte: [ "vitals.systolic", 180 ] }, color: "red" },
      { when: { lte: [ "vitals.spo2", 89 ] }, color: "red" },
      { when: { gte: [ "vitals.temperature_c", 39 ] }, color: "yellow" },
      { when: { gte: [ "vitals.capillary_glucose", 300 ] }, color: "yellow" }
    ]
  },
  null,
  2
);
```

```tsx
// src/modules/protocolEditor/RiskRulesPanel.tsx
// Painel "Regras de cor" do protocolo de acolhimento (módulo 18; spec §3.3).
// Lê `risk_rules` da definição a cada render e devolve uma definição nova a
// cada edição, como o painel "Agendamento". Conteúdo assinado (ADR 0016).
import type { CSSProperties, ReactNode } from "react";
import type { ScreeningColor } from "../../lib/api";
import { ConditionBuilder } from "../protocols/ConditionBuilder";
import { fieldsFor } from "../../lib/condition";
import { COLORS, COLOR_HINT, COLOR_LABEL } from "../../lib/screening";
import { RISK_RULES_MAX, readRiskRules, riskRuleProblem, writeRiskRules, type RiskRuleDraft } from "../../lib/riskRules";
import { disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";

interface Props { definition: unknown | null; onChange(next: unknown): void }

export function RiskRulesPanel({ definition, onChange }: Props) {
  if (definition === null) return <Section><p style={hint}>Corrija o JSON para editar as regras de cor.</p></Section>;
  const read = readRiskRules(definition);
  if (!read.ok) return <Section><p role="alert" style={alert}>{read.reason}</p></Section>;
  const rules = read.rules;
  const fields = fieldsFor("screening");
  const setRules = (list: RiskRuleDraft[]) => onChange(writeRiskRules(definition, list));
  const replace = (i: number, next: RiskRuleDraft) => setRules(rules.map((r, j) => (j === i ? next : r)));

  return (
    <Section>
      <p style={hint}>
        Vale a cor mais grave entre as regras que casarem (vermelho, amarelo, verde, azul). É só sugestão: a cor final é de quem escuta.
      </p>
      {rules.length === 0 && <p role="alert" style={alert}>Inclua ao menos uma regra de cor.</p>}
      {rules.map((r, i) => {
        const problem = riskRuleProblem(r);
        return (
          <div key={i} role="group" aria-label={`regra de cor ${i + 1}`} style={card}>
            <label style={label}>
              Cor sugerida
              <select value={r.color} style={inputStyle} onChange={(e) => replace(i, { ...r, color: e.target.value as ScreeningColor })}>
                {COLORS.map((c) => <option key={c} value={c}>{`${COLOR_LABEL[c]} — ${COLOR_HINT[c]}`}</option>)}
              </select>
            </label>
            <ConditionBuilder label="Quando sugerir" fields={fields} value={r.when} emptyText="—"
              onChange={(tree) => replace(i, { ...r, when: tree })} />
            {problem && <small style={hint}>{problem}</small>}
            <div>
              <button type="button" style={secondaryButtonStyle} onClick={() => setRules(rules.filter((_, j) => j !== i))}>remover regra</button>
            </div>
          </div>
        );
      })}
      <div>
        <button type="button" disabled={rules.length >= RISK_RULES_MAX}
          style={rules.length >= RISK_RULES_MAX ? disabledButtonStyle : secondaryButtonStyle}
          onClick={() => setRules([ ...rules, { when: null, color: "yellow" } ])}>
          + regra de cor
        </button>
      </div>
    </Section>
  );
}

function Section({ children }: { children: ReactNode }) {
  return <section aria-label="Regras de cor" style={{ display: "flex", flexDirection: "column", gap: 8 }}>{children}</section>;
}

const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12, color: "var(--down)" };
const card: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, padding: 10, border: "1px solid var(--rule)", borderRadius: 8 };
```

```tsx
// src/modules/protocolEditor/ScreeningSimulator.tsx
// Simulador do acolhimento (módulo 18; spec §8): confere a cor que ESTA
// definição sugeriria para um perfil, uma queixa e sinais vitais. Não grava
// nada e não usa o protocolo ativo da cidade.
import { useState, type CSSProperties } from "react";
import { simulateScreening, type Sex, type SimulateScreeningResult } from "../../lib/api";
import { COLOR_HINT, COLOR_LABEL, COLOR_TONE, EMPTY_VITALS_FORM, bmiOf, parseVitals, type VitalsForm } from "../../lib/screening";
import { MAX_AGE, SEX_OPTIONS } from "../../lib/profile";
import { Tag } from "../../components/Tag";
import { VitalSignsFields } from "../attendance/VitalSignsFields";
import { buttonStyle, disabledButtonStyle, inputStyle } from "../../components/formStyles";

const CIAP2 = /^[A-Z]\d{2}$/;

export function ScreeningSimulator({ definition, valid }: { definition: unknown | null; valid: boolean }) {
  const [ age, setAge ] = useState("45");
  const [ sex, setSex ] = useState<Sex>("female");
  const [ code, setCode ] = useState("");
  const [ vitalsForm, setVitalsForm ] = useState<VitalsForm>(EMPTY_VITALS_FORM);
  const [ shown, setShown ] = useState<{ key: string; result: SimulateScreeningResult } | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);

  const { vitals, problems } = parseVitals(vitalsForm);
  const ciap = code.trim().toUpperCase();
  const ageOk = /^\d+$/.test(age) && Number(age) <= MAX_AGE;
  const codeOk = ciap === "" || CIAP2.test(ciap);
  const can = valid && definition !== null && ageOk && codeOk && Object.keys(problems).length === 0 && !busy;
  const inputKey = JSON.stringify([ definition, age, sex, ciap, vitals ]);
  const result = shown && shown.key === inputKey ? shown.result : null;

  async function run() {
    if (!can) return;
    setBusy(true); setError(null);
    const key = inputKey;
    try {
      const res = await simulateScreening({ definition, ciap2_code: ciap === "" ? null : ciap, vitals, profile: { age: Number(age), sex } });
      setShown({ key, result: res });
    } catch {
      setError("não foi possível simular — tente de novo");
    } finally {
      setBusy(false);
    }
  }

  return (
    <section aria-label="Simulador do acolhimento" style={{ marginTop: 16, display: "flex", flexDirection: "column", gap: 8 }}>
      <h3 style={{ fontSize: 14, margin: 0 }}>Simulador do acolhimento</h3>
      <p style={hint}>Confere a cor que esta definição sugeriria. Não grava nada e não usa o protocolo ativo da cidade.</p>
      <div style={{ display: "flex", gap: 8, flexWrap: "wrap" }}>
        <label style={label}>Idade<input type="number" value={age} style={small} onChange={(e) => setAge(e.target.value)} /></label>
        <label style={label}>
          Sexo
          <select value={sex} style={small} onChange={(e) => setSex(e.target.value as Sex)}>
            {SEX_OPTIONS.map((o) => <option key={o.value} value={o.value}>{o.label}</option>)}
          </select>
        </label>
        <label style={label}>Código CIAP-2<input value={code} placeholder="K86" style={small} onChange={(e) => setCode(e.target.value)} /></label>
      </div>
      <VitalSignsFields form={vitalsForm} problems={problems} alerts={result?.alerts ?? []}
        bmi={bmiOf(vitals.weight_kg, vitals.height_cm)} onChange={setVitalsForm} />
      {(!ageOk || !codeOk) && <small role="alert" style={alert}>{`idade de 0 a ${MAX_AGE}; código CIAP-2 como K86`}</small>}
      <div>
        <button type="button" disabled={!can} style={can ? buttonStyle : disabledButtonStyle} onClick={() => void run()}>Simular</button>
      </div>
      {!valid && <small style={hint}>Corrija os erros para simular.</small>}
      {error && <p role="alert" style={alert}>{error}</p>}
      {result && (
        <div role="status" style={{ display: "flex", flexDirection: "column", gap: 4, fontSize: 13 }}>
          {result.suggested_color ? (
            <strong>Cor sugerida: <Tag tone={COLOR_TONE[result.suggested_color]}>{COLOR_LABEL[result.suggested_color]}</Tag> · {COLOR_HINT[result.suggested_color]}</strong>
          ) : (
            <strong>Nenhuma regra casou: sem cor sugerida</strong>
          )}
          {result.matched_rules.length > 0 && (
            <ul aria-label="regras que casaram" style={{ margin: 0, paddingLeft: 18 }}>
              {result.matched_rules.map((r) => <li key={r.index}>{`regra ${r.index + 1}: ${r.text}`}</li>)}
            </ul>
          )}
          {result.warnings.length > 0 && (
            <ul aria-label="avisos" style={{ ...hint, margin: 0, paddingLeft: 18 }}>{result.warnings.map((w, i) => <li key={i}>{w}</li>)}</ul>
          )}
          {result.errors.length > 0 && (
            <ul role="alert" style={{ ...alert, paddingLeft: 18 }}>{result.errors.map((e, i) => <li key={i}>{e}</li>)}</ul>
          )}
        </div>
      )}
    </section>
  );
}

const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const small: CSSProperties = { ...inputStyle, width: 110 };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12, color: "var(--down)" };
```

Em `src/modules/ProtocolEditor.tsx` (o bloco da coluna da direita de hoje fica inteiro no ramo `: (` — só ganha o `)}` no fim):

```diff
--- a/src/modules/ProtocolEditor.tsx
+++ b/src/modules/ProtocolEditor.tsx
@@ -8,6 +8,9 @@ import { OfferPanel } from "./protocolEditor/OfferPanel";
 import { OfferSimulator } from "./protocolEditor/OfferSimulator";
 import { QuestionPreview } from "./protocolEditor/QuestionPreview";
 import { SchedulingPanel } from "./protocolEditor/SchedulingPanel";
+import { RiskRulesPanel } from "./protocolEditor/RiskRulesPanel";
+import { ScreeningSimulator } from "./protocolEditor/ScreeningSimulator";
+import { SCREENING_TEMPLATE, isScreeningDefinition } from "../lib/riskRules";
 
 export function ProtocolEditor() {
   const [ text, setText ] = useState<string>(TEMPLATE);
@@ -42,6 +45,7 @@ export function ProtocolEditor() {
     setLoadErr(null);
     setOfferKey(k => k + 1); // remonta o painel: condições meio digitadas não sobrevivem à troca
     if (value === "__new__") { setText(TEMPLATE); return; }
+    if (value === "__new_screening__") { setText(SCREENING_TEMPLATE); return; }
     const [ name, version ] = value.split("@@");
     loadProtocolDefinition(name, version).then(def => {
       if (def) setText(JSON.stringify(def, null, 2));
@@ -82,6 +86,8 @@ export function ProtocolEditor() {
   }
 
   const current = parseDefinition(text);
+  // Módulo 18: `kind: "screening"` troca a coluna da direita pelas regras de cor e o simulador do acolhimento.
+  const screeningKind = current.ok && isScreeningDefinition(current.value);
   // Nomes de protocolo da cidade para a sugestão (um nome por protocolo, sem a versão).
   const protocolNames = [ ...new Set(opts.map((o) => o.name)) ].sort();
 
@@ -95,6 +101,7 @@ export function ProtocolEditor() {
           style={{ display: "block", marginBottom: 8, fontSize: 13 }}
         >
           <option value="__new__">Nova (template)</option>
+          <option value="__new_screening__">Novo acolhimento (modelo)</option>
           {opts.map(o => (
             <option key={`${o.name}@@${o.version}`} value={`${o.name}@@${o.version}`}>
               {o.name}@{o.version} ({o.status})
@@ -142,6 +149,17 @@ export function ProtocolEditor() {
         )}
       </section>
 
+      {screeningKind ? (
+        <section>
+          <h2 style={{ fontSize: 16, margin: "0 0 8px" }}>Regras de cor (acolhimento)</h2>
+          <RiskRulesPanel
+            key={offerKey}
+            definition={current.ok ? current.value : null}
+            onChange={(next) => setText(JSON.stringify(next, null, 2))}
+          />
+          <ScreeningSimulator definition={current.ok ? current.value : null} valid={valid} />
+        </section>
+      ) : (
       <section>
         <h2 style={{ fontSize: 16, margin: "0 0 8px" }}>Oferta e sugestões</h2>
         <OfferPanel
@@ -175,6 +193,7 @@ export function ProtocolEditor() {
         <QuestionPreview definition={current.ok ? current.value : null} />
         <OfferSimulator definition={current.ok ? current.value : null} valid={valid} answers={answers} />
       </section>
+      )}
     </div>
   );
 }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/lib/riskRules.test.ts src/modules/protocolEditor src/modules/ProtocolEditor.test.tsx`
Expected: PASS (`riskRules` 4, `RiskRulesPanel` 5, `ScreeningSimulator` 4, `ProtocolEditor` 20; os outros painéis do editor seguem verdes).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod18 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 add src/lib/riskRules.ts src/lib/riskRules.test.ts src/modules/protocolEditor/RiskRulesPanel.tsx src/modules/protocolEditor/RiskRulesPanel.test.tsx src/modules/protocolEditor/ScreeningSimulator.tsx src/modules/protocolEditor/ScreeningSimulator.test.tsx src/modules/ProtocolEditor.tsx src/modules/ProtocolEditor.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 commit -m "feat: edit screening protocols with risk rules and a simulator

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 11: Produção e-SUS — códigos de recusa e fichas que não puderam ser geradas

**Files:**
- Modify: `src/lib/api.ts` (`LediFicha.last_error_codes`, `LediErrorCode`, `LediRejection`, `Production.rejections`), `src/lib/production.ts`, `src/modules/Production.tsx`, `src/test/recordModeFixtures.ts`, `src/lib/api.recordMode.test.ts`
- Create: `src/modules/production/GenerationFailures.tsx`
- Test: `src/lib/production.test.ts`, `src/modules/Production.test.tsx`

**Interfaces:**
- Consumes: `listGenerationFailures`, `retryGenerationFailure`, `GenerationFailure` (Task 1); `SensitiveAction`, `featureDisabledKey`, `productionError`/`productionErrorCode` (existentes).
- Produces:
  - `api.ts`: `interface LediErrorCode { field: string; code: string }`, `interface LediRejection extends LediErrorCode { count: number }`; `LediFicha.last_error_codes: LediErrorCode[]` no lugar de `last_error`; `Production.rejections: LediRejection[]`;
  - `production.ts`: `ALREADY_RESOLVED`, `errorCodeLabel(c)` (`"<campo> · <código>"`; `other`/`unknown` = "erro não classificado"), `errorCodesLabel(codes)` ("—" vazio), `GENERATION_REASON`, `SOURCE_LABEL` (`Screening` = "escuta inicial"), `reasonsLabel(codes)`, `retryOutcome(failure)`, `canRetryGeneration(roles)` (só `municipal_admin`); `productionError` traduz `already_resolved`;
  - `GenerationFailures({ roles: string[]; onGoToSecurity?(): void })` — `Panel` "Fichas que não puderam ser geradas", colunas Origem, Motivo, Desde; botão "Gerar de novo <id>" (nome acessível) só para `municipal_admin`, via `SensitiveAction` com step-up (`confirmLabel` "Gerar de novo"); chave `[ "productionGenerationFailures" ]`; depois de gerar, diz se a ficha nasceu ou o que ainda falta, e relê; 409 `already_resolved` relê com a frase.
  - `Production`: coluna "Último erro" com os códigos; "Motivos de recusa" agrupados por campo e código; o painel novo no fim da tela.

- [ ] **Step 1: Write the failing test**

```diff
--- a/src/lib/production.test.ts
+++ b/src/lib/production.test.ts
@@ -2,7 +2,8 @@
 import { describe, expect, it } from "vitest";
 import { ApiError } from "./api";
 import {
-  FICHA_STATUS, alertBanner, canReadProduction, canResend, deadlinePhrase, hasNextPage, productionError, productionErrorCode
+  FICHA_STATUS, alertBanner, canReadProduction, canResend, canRetryGeneration, deadlinePhrase, errorCodesLabel, hasNextPage,
+  productionError, productionErrorCode, reasonsLabel, retryOutcome
 } from "./production";
 import { ficha } from "../test/recordModeFixtures";
 
@@ -49,6 +50,32 @@ describe("Produção — papéis e paginação", () => {
   });
 });
 
+describe("Produção — códigos e fichas não geradas (módulo 18)", () => {
+  it("códigos de recusa como vêm; other/unknown vira 'erro não classificado'; vazio é traço", () => {
+    expect(errorCodesLabel([ { field: "profissional.cns", code: "invalid" }, { field: "other", code: "unknown" } ]))
+      .toBe("profissional.cns · invalid; erro não classificado");
+    expect(errorCodesLabel([])).toBe("—");
+    expect(errorCodesLabel(undefined)).toBe("—");
+  });
+
+  it("motivos em português; desconhecido aparece como veio", () => {
+    expect(reasonsLabel([ "unit_without_cnes", "citizen_without_sex", "new_reason" ]))
+      .toBe("unidade sem CNES, cidadão sem sexo no cadastro, new_reason");
+  });
+
+  it("gerar de novo: resolvida ou o que ainda falta", () => {
+    expect(retryOutcome({ resolved_at: "2026-10-07T10:00:00Z", reason_codes: [] })).toBe("Ficha gerada: ela entra na fila de envio.");
+    expect(retryOutcome({ resolved_at: null, reason_codes: [ "professional_without_team" ] }))
+      .toBe("Ainda não foi possível gerar: profissional sem equipe (INE). Corrija na origem e tente de novo.");
+  });
+
+  it("só o municipal_admin gera de novo; already_resolved tem frase", () => {
+    expect(canRetryGeneration([ "municipal_admin" ])).toBe(true);
+    expect(canRetryGeneration([ "analyst" ])).toBe(false);
+    expect(productionError(new ApiError(409, { error: "already_resolved" }, "409"))).toBe("esta ficha já foi gerada — a lista foi atualizada");
+  });
+});
+
 describe("Produção — erros", () => {
   it("not_rejected, invalid_competence e feature_disabled têm frase própria; o resto segue o padrão", () => {
     const stale = new ApiError(409, { error: "not_rejected" }, "409");
```

```diff
--- a/src/modules/Production.test.tsx
+++ b/src/modules/Production.test.tsx
@@ -4,7 +4,8 @@ import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
 
 vi.mock("../lib/api", async (importOriginal) => {
   const real = await importOriginal<typeof import("../lib/api")>();
-  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), getProduction: vi.fn(), resendFicha: vi.fn() };
+  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), getProduction: vi.fn(), resendFicha: vi.fn(),
+    listGenerationFailures: vi.fn(), retryGenerationFailure: vi.fn() };
 });
 
 import * as api from "../lib/api";
@@ -17,9 +18,15 @@ afterEach(cleanup);
 const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
 const DISABLED = "o envio da produção ao e-SUS está desligado nesta cidade";
 const withLedi = (roles: string[]) => sessionWith(roles, { features: [ "ledi_export" ] });
+const failure = (over: Partial<api.GenerationFailure> = {}): api.GenerationFailure => ({
+  id: "g1", source_type: "Screening", source_id: "sc1", attendance_id: "a1",
+  reason_codes: [ "unit_without_cnes", "professional_without_team" ], created_at: "2026-10-07T12:00:00Z", resolved_at: null, ...over
+});
 
 function resetAll() {
-  for (const fn of [ api.fetchCurrentSession, api.stepUpMfa, api.getProduction, api.resendFicha ]) m(fn).mockReset();
+  for (const fn of [ api.fetchCurrentSession, api.stepUpMfa, api.getProduction, api.resendFicha, api.listGenerationFailures,
+    api.retryGenerationFailure ]) m(fn).mockReset();
+  m(api.listGenerationFailures).mockResolvedValue([]);
 }
 
 describe("Produção e-SUS (módulo 16)", () => {
@@ -34,7 +41,7 @@ describe("Produção e-SUS (módulo 16)", () => {
     renderWithProviders(<Production />);
     expect(await screen.findByText("prazo em 16/11/2026 · faltam 7 dias úteis")).not.toBeNull();
     expect(screen.getByText("Há fichas pendentes, recusadas ou com falha, e o prazo está perto. Confira a situação abaixo.")).not.toBeNull();
-    expect(screen.getAllByText("CNS do profissional inválido").length).toBe(2);
+    expect(screen.getAllByText("profissional.cns · invalid").length).toBe(2);
     expect(screen.getByText("recusada")).not.toBeNull();
     expect(screen.queryByRole("button", { name: /Reenviar ficha/ })).toBeNull();
     expect(api.getProduction).toHaveBeenCalledWith(null, 1);
@@ -130,6 +137,41 @@ describe("Produção e-SUS (módulo 16)", () => {
     expect(api.getProduction).toHaveBeenCalledTimes(1);
   });
 
+  it("módulo 18: fichas não geradas com motivo; o analista não gera de novo", async () => {
+    m(api.fetchCurrentSession).mockResolvedValue(withLedi([ "analyst" ]));
+    m(api.listGenerationFailures).mockResolvedValue([ failure() ]);
+    renderWithProviders(<Production />);
+    expect(await screen.findByText("unidade sem CNES, profissional sem equipe (INE)")).not.toBeNull();
+    expect(screen.getByText("escuta inicial")).not.toBeNull();
+    expect(screen.queryByRole("button", { name: "Gerar de novo g1" })).toBeNull();
+  });
+
+  it("módulo 18: municipal_admin gera de novo com step-up; ainda faltando, diz o quê", async () => {
+    m(api.listGenerationFailures).mockResolvedValue([ failure() ]);
+    m(api.retryGenerationFailure).mockResolvedValue(failure({ reason_codes: [ "professional_without_team" ] }));
+    renderWithProviders(<Production />);
+    fireEvent.click(await screen.findByRole("button", { name: "Gerar de novo g1" }));
+    fireEvent.click(await screen.findByRole("button", { name: "Gerar de novo" }));
+    await waitFor(() => expect(api.retryGenerationFailure).toHaveBeenCalledWith("g1"));
+    expect(await screen.findByText("Ainda não foi possível gerar: profissional sem equipe (INE). Corrija na origem e tente de novo."))
+      .not.toBeNull();
+    await waitFor(() => expect(api.listGenerationFailures).toHaveBeenCalledTimes(2));
+  });
+
+  it("módulo 18: resolvida diz que a ficha nasceu; already_resolved relê", async () => {
+    m(api.listGenerationFailures).mockResolvedValue([ failure() ]);
+    m(api.retryGenerationFailure).mockResolvedValueOnce(failure({ reason_codes: [], resolved_at: "2026-10-07T13:00:00Z" }));
+    renderWithProviders(<Production />);
+    fireEvent.click(await screen.findByRole("button", { name: "Gerar de novo g1" }));
+    fireEvent.click(await screen.findByRole("button", { name: "Gerar de novo" }));
+    expect(await screen.findByText("Ficha gerada: ela entra na fila de envio.")).not.toBeNull();
+
+    m(api.retryGenerationFailure).mockRejectedValueOnce(new ApiError(409, { error: "already_resolved" }, "409"));
+    fireEvent.click(await screen.findByRole("button", { name: "Gerar de novo g1" }));
+    fireEvent.click(await screen.findByRole("button", { name: "Gerar de novo" }));
+    expect(await screen.findByText("esta ficha já foi gerada — a lista foi atualizada")).not.toBeNull();
+  });
+
   it("papel sem leitura: não chama a API", async () => {
     m(api.fetchCurrentSession).mockResolvedValue(withLedi([ "citizen_verifier" ]));
     renderWithProviders(<Production />);
```

Fixtures e o teste do cliente passam para a forma nova:

```diff
--- a/src/test/recordModeFixtures.ts
+++ b/src/test/recordModeFixtures.ts
@@ -51,7 +51,7 @@ export function cnesFixture(overrides: Partial<CnesOverview> = {}): CnesOverview
 
 export function ficha(overrides: Partial<LediFicha> = {}): LediFicha {
   return {
-    id: "f1", ficha_type: "synthetic", status: "accepted", attempts: 1, last_error: null,
+    id: "f1", ficha_type: "synthetic", status: "accepted", attempts: 1, last_error_codes: [],
     created_at: "2026-10-02T13:00:00Z", accepted_at: "2026-10-02T13:01:00Z", ...overrides
   };
 }
@@ -62,10 +62,10 @@ export function productionFixture(overrides: Partial<Production> = {}): Producti
   return {
     competence: "202610", deadline_on: "2026-11-16", business_days_left: 7, alert: "attention",
     counts: { accepted: 120, rejected: 3, pending: 10, sending: 0, failed: 0 },
-    rejections: [ { message: "CNS do profissional inválido", count: 3 } ],
+    rejections: [ { field: "profissional.cns", code: "invalid", count: 3 } ],
     fichas: [
       ficha(),
-      ficha({ id: "f2", status: "rejected", accepted_at: null, last_error: "CNS do profissional inválido" })
+      ficha({ id: "f2", status: "rejected", accepted_at: null, last_error_codes: [ { field: "profissional.cns", code: "invalid" } ] })
     ],
     fichas_total: 2,
     ...overrides
```

```diff
--- a/src/lib/api.recordMode.test.ts
+++ b/src/lib/api.recordMode.test.ts
@@ -91,7 +91,7 @@ describe("cliente da Produção (contratos §5.3)", () => {
   });
 
   it("reenvia a ficha com POST no id escapado", async () => {
-    const fn = stub({ id: "a/b", ficha_type: "synthetic", status: "pending", attempts: 2, last_error: null,
+    const fn = stub({ id: "a/b", ficha_type: "synthetic", status: "pending", attempts: 2, last_error_codes: [],
       created_at: "2026-10-02T13:00:00Z", accepted_at: null });
     expect((await resendFicha("a/b")).status).toBe("pending");
     expect(call(fn)[0]).toBe("/production/fichas/a%2Fb/resend");
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/lib/production.test.ts src/modules/Production.test.tsx`
Expected: FAIL — `errorCodesLabel is not a function` (e as outras funções novas), `Unable to find an element with the text: profissional.cns · invalid`, e os testes "módulo 18" sem o painel. (`tsc` acusa `last_error_codes` nas fixtures até o tipo mudar.)

- [ ] **Step 3: Write minimal implementation**

```diff
--- a/src/lib/api.ts
+++ b/src/lib/api.ts
@@ -1245,12 +1245,15 @@ export async function applyCnesProposals(proposalIds: string[]): Promise<CnesApp
 
 export type CompetenceAlert = "none" | "attention" | "critical";
 export type FichaStatus = "pending" | "sending" | "accepted" | "rejected" | "failed";
+export interface LediErrorCode { field: string; code: string }
+export interface LediRejection extends LediErrorCode { count: number }
 export interface LediFicha {
   id: string;
   ficha_type: string;
   status: FichaStatus;
   attempts: number;
-  last_error: string | null;
+  // Módulo 18 (ADR 0030; contratos §6): só códigos, nunca o texto do PEC.
+  last_error_codes: LediErrorCode[];
   created_at: string;
   accepted_at: string | null;
 }
@@ -1261,7 +1264,7 @@ export interface Production {
   alert: CompetenceAlert;
   // `sending` pode faltar na resposta e `pending` não o soma (contratos §5.3).
   counts: { accepted: number; rejected: number; pending: number; failed: number; sending?: number };
-  rejections: { message: string; count: number }[];
+  rejections: LediRejection[];
   fichas: LediFicha[];
   // Total de fichas da competência, para a paginação (contratos §5.3).
   fichas_total: number;
```

```diff
--- a/src/lib/production.ts
+++ b/src/lib/production.ts
@@ -4,9 +4,9 @@
 // prazos do SIAPS do ano (dias úteis com feriados nacionais só como fallback
 // para ano sem tabela); aqui só se diz em português. `deadline_on` é data sem
 // hora: formata por texto (fmtDay), nunca por new Date.
-// Mensagens de recusa (`rejections[].message`, `last_error`) já chegam
-// mascaradas da API e são mostradas como vieram.
-import { ApiError, FICHAS_PER_PAGE, type LediFicha } from "./api";
+// Desde o módulo 18 (ADR 0030; api#43) a recusa chega só como códigos
+// (`{ field, code }`), nunca o texto do PEC: a tela mostra os códigos como vêm.
+import { ApiError, FICHAS_PER_PAGE, type LediErrorCode, type LediFicha } from "./api";
 import { describeActionError } from "./actionErrors";
 import { FEATURE_DISABLED_MESSAGE, featureDisabledKey } from "./features";
 import { fmtDay } from "./audiencePhrase";
@@ -24,6 +24,42 @@ export const FICHA_STATUS: Record<string, { label: string; tone: "neutral" | "in
   failed: { label: "falhou — sem novas tentativas", tone: "down" }
 };
 
+export const ALREADY_RESOLVED = "esta ficha já foi gerada — a lista foi atualizada";
+
+export function errorCodeLabel(c: LediErrorCode): string {
+  return c.field === "other" && c.code === "unknown" ? "erro não classificado" : `${c.field} · ${c.code}`;
+}
+
+export function errorCodesLabel(codes: LediErrorCode[] | undefined): string {
+  return codes && codes.length > 0 ? codes.map(errorCodeLabel).join("; ") : "—";
+}
+
+// Fichas que não puderam ser geradas (módulo 18; spec §5): o que falta na origem.
+export const GENERATION_REASON: Record<string, string> = {
+  unit_without_cnes: "unidade sem CNES",
+  professional_without_team: "profissional sem equipe (INE)",
+  professional_without_cns: "profissional sem CNS",
+  citizen_without_birth_date: "cidadão sem data de nascimento",
+  citizen_without_sex: "cidadão sem sexo no cadastro",
+  unknown_ciap2: "CIAP-2 fora da terminologia ativa"
+};
+export const SOURCE_LABEL: Record<string, string> = { Screening: "escuta inicial" };
+
+export function reasonsLabel(codes: string[]): string {
+  return codes.map((c) => GENERATION_REASON[c] ?? c).join(", ");
+}
+
+// Gerar de novo: resolvida = ficha nasceu; senão, diz o que ainda falta.
+export function retryOutcome(f: { resolved_at: string | null; reason_codes: string[] }): string {
+  return f.resolved_at
+    ? "Ficha gerada: ela entra na fila de envio."
+    : `Ainda não foi possível gerar: ${reasonsLabel(f.reason_codes)}. Corrija na origem e tente de novo.`;
+}
+
+export function canRetryGeneration(roles: string[]): boolean {
+  return roles.includes("municipal_admin");
+}
+
 export function canReadProduction(roles: string[]): boolean {
   return roles.includes("municipal_admin") || roles.includes("analyst");
 }
@@ -67,6 +103,7 @@ export function productionErrorCode(err: unknown): string | null {
 export function productionError(err: unknown): string {
   const code = productionErrorCode(err);
   if (code === "not_rejected") return RESEND_STALE;
+  if (code === "already_resolved") return ALREADY_RESOLVED;
   if (code === "invalid_competence") return INVALID_COMPETENCE;
   if (featureDisabledKey(err) !== null) return FEATURE_DISABLED_MESSAGE;
   const d = describeActionError(err);
```

```tsx
// src/modules/production/GenerationFailures.tsx
// "Fichas que não puderam ser geradas" (módulo 18; spec §5; contratos §6): a
// escuta concluída sem identificação completa não vira ficha; aqui aparece o
// motivo, e o municipal_admin pede "gerar de novo" (step-up) depois de
// corrigir a origem (CNES da unidade, equipe do profissional, cadastro).
import { useRef, useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { listGenerationFailures, retryGenerationFailure, type GenerationFailure } from "../../lib/api";
import {
  ALREADY_RESOLVED, SOURCE_LABEL, canRetryGeneration, productionError, productionErrorCode, reasonsLabel, retryOutcome
} from "../../lib/production";
import { featureDisabledKey } from "../../lib/features";
import { fmtDateTime } from "../../lib/format";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { SensitiveAction } from "../../components/SensitiveAction";
import { secondaryButtonStyle } from "../../components/formStyles";

export const GENERATION_FAILURES_KEY = "productionGenerationFailures";

export function GenerationFailures({ roles, onGoToSecurity }: { roles: string[]; onGoToSecurity?(): void }) {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: [ GENERATION_FAILURES_KEY ], queryFn: listGenerationFailures });
  const [ retrying, setRetrying ] = useState<GenerationFailure | null>(null);
  const [ done, setDone ] = useState<string | null>(null);
  const outcome = useRef<GenerationFailure | null>(null);
  const refresh = () => void queryClient.invalidateQueries({ queryKey: [ GENERATION_FAILURES_KEY ] });
  const canRetry = canRetryGeneration(roles);

  return (
    <Panel title="Fichas que não puderam ser geradas" sub="falta identificação na origem">
      <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
        {done && <p role="status" style={{ margin: 0, fontSize: 13, fontWeight: 600 }}>{done}</p>}
        {query.isError && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{productionError(query.error)}</p>}
        {retrying && (
          <SensitiveAction
            key={retrying.id}
            title="Gerar a ficha de novo"
            description="Confira antes se o motivo foi corrigido na origem. Se ainda faltar algo, a ficha continua nesta lista."
            requiresStepUp
            confirmLabel="Gerar de novo"
            run={async () => { outcome.current = await retryGenerationFailure(retrying.id); }}
            onDone={() => {
              setRetrying(null);
              setDone(outcome.current ? retryOutcome(outcome.current) : null);
              refresh();
            }}
            onCancel={() => setRetrying(null)}
            onGoToSecurity={onGoToSecurity}
            translateError={(err) => {
              const code = productionErrorCode(err);
              if (featureDisabledKey(err) !== null) { refresh(); return productionError(err); }
              if (code !== "already_resolved") return null;
              refresh();
              return ALREADY_RESOLVED;
            }}
          />
        )}
        {query.isPending ? (
          <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando…</p>
        ) : (
          <DataTable<GenerationFailure>
            cols={[
              { label: "Origem", w: "1fr", render: (f) => SOURCE_LABEL[f.source_type] ?? f.source_type },
              { label: "Motivo", w: "3fr", render: (f) => reasonsLabel(f.reason_codes) },
              { label: "Desde", w: "1fr", render: (f) => fmtDateTime(f.created_at) },
              { label: "", w: "auto", align: "right", render: (f) => canRetry && (
                <button type="button" aria-label={`Gerar de novo ${f.id}`} style={secondaryButtonStyle}
                  onClick={() => { setDone(null); setRetrying(f); }}>
                  Gerar de novo
                </button>
              ) }
            ]}
            rows={query.data ?? []}
            rowKey={(f) => f.id}
            empty="nenhuma ficha pendente de identificação"
          />
        )}
      </div>
    </Panel>
  );
}
```

```diff
--- a/src/modules/Production.tsx
+++ b/src/modules/Production.tsx
@@ -14,9 +14,11 @@ import { competenceLabel, competenceOptions } from "../lib/competence";
 import { todayInCity } from "../lib/campaigns";
 import { fmtDateTime, fmtNumber } from "../lib/format";
 import {
-  FICHA_STATUS, PRODUCTION_KEY, RESEND_STALE, alertBanner, canReadProduction, canResend, deadlinePhrase, hasNextPage,
-  productionError, productionErrorCode
+  FICHA_STATUS, PRODUCTION_KEY, RESEND_STALE, alertBanner, canReadProduction, canResend, deadlinePhrase, errorCodeLabel,
+  errorCodesLabel, hasNextPage, productionError, productionErrorCode
 } from "../lib/production";
+import type { LediRejection } from "../lib/api";
+import { GenerationFailures } from "./production/GenerationFailures";
 import { PageHeader } from "../components/PageHeader";
 import { Panel } from "../components/Panel";
 import { DataTable, type Column } from "../components/DataTable";
@@ -76,7 +78,7 @@ export function Production({ onGoToSecurity }: { onGoToSecurity?(): void }) {
     { label: "Situação", w: "1.2fr", render: (f) =>
       <Tag tone={FICHA_STATUS[f.status]?.tone}>{FICHA_STATUS[f.status]?.label ?? f.status}</Tag> },
     { label: "Tentativas", w: "0.7fr", align: "right", render: (f) => <span className="mono">{fmtNumber(f.attempts)}</span> },
-    { label: "Último erro", w: "2fr", render: (f) => f.last_error ?? "—" },
+    { label: "Último erro", w: "2fr", render: (f) => <span className="mono">{errorCodesLabel(f.last_error_codes)}</span> },
     { label: "Criada em", w: "1fr", render: (f) => fmtDateTime(f.created_at) },
     { label: "Aceita em", w: "1fr", render: (f) => fmtDateTime(f.accepted_at) },
     { label: "", w: "auto", align: "right", render: (f) => canResend(roles, f) && (
@@ -144,14 +146,14 @@ export function Production({ onGoToSecurity }: { onGoToSecurity?(): void }) {
             <StatTile label="Falharam" value={data.counts.failed} tone={data.counts.failed > 0 ? "down" : undefined} />
           </KpiGrid>
 
-          <Panel title="Motivos de recusa" sub="agrupados pela mensagem do PEC">
-            <DataTable<{ message: string; count: number }>
+          <Panel title="Motivos de recusa" sub="agrupados por campo e código">
+            <DataTable<LediRejection>
               cols={[
-                { label: "Motivo", w: "3fr", render: (r) => r.message },
+                { label: "Campo e código", w: "3fr", render: (r) => <span className="mono">{errorCodeLabel(r)}</span> },
                 { label: "Fichas", w: "0.7fr", align: "right", render: (r) => <span className="mono">{fmtNumber(r.count)}</span> }
               ]}
               rows={data.rejections}
-              rowKey={(r) => r.message}
+              rowKey={(r) => `${r.field}:${r.code}`}
               empty="nenhuma recusa nesta competência"
             />
           </Panel>
@@ -174,6 +176,8 @@ export function Production({ onGoToSecurity }: { onGoToSecurity?(): void }) {
           </Panel>
         </>
       )}
+
+      <GenerationFailures roles={roles} onGoToSecurity={onGoToSecurity} />
     </Frame>
   );
 }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run src/lib/production.test.ts src/modules/Production.test.tsx src/lib/api.recordMode.test.ts && npx tsc --noEmit`
Expected: PASS (`production.test.ts` 10, `Production.test.tsx` 15); `grep -rn "last_error\b" src` não acha nada além de `last_error_codes`.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod18 && npx tsc --noEmit
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 add src/lib/api.ts src/lib/production.ts src/lib/production.test.ts src/modules/Production.tsx src/modules/Production.test.tsx src/modules/production/GenerationFailures.tsx src/test/recordModeFixtures.ts src/lib/api.recordMode.test.ts
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 commit -m "feat: show error codes and generation failures in e-SUS production

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 12: Suíte, build, revisão e prova no navegador

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd apps/dashboard/.claude/mod18 && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde (125 arquivos e 1329 testes, se `origin/main` não andou). A CI roda os mesmos três; `dist/` está no `.gitignore`.

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod18 log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e a contagem `0`. Se o symlink aparecer, **não** o adicione.

- [ ] **Step 2: Nada sobrou do formato antigo e nada vaza**

```bash
cd apps/dashboard/.claude/mod18
grep -rn "last_error\b" src
grep -rn "complaint_note\|orientation_note\|color_change_reason[^_]" src --include='*.tsx' | grep -v "\.test\." | grep -v "attendance/ScreeningForm\|attendance/ScreeningDetail"
grep -n "complaint\|vitals" src/modules/attendance/UnitQueue.tsx
```

Expected: os três sem saída — nenhum `last_error` antigo; texto livre da escuta só no formulário e no detalhe do profissional; a fila (`UnitQueue`) não toca queixa nem sinais (a escuta inteira só no `ScreeningDetail`, atrás de `canCare`).

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§3, §4, §6, §8), o ADR 0030 e o arquivo de contratos (§1–§6). Pontos de atenção:
- a recepção nunca recebe ação nem leitura da escuta (`canCare` em "Ver escuta", "Reavaliar" e no painel do acolhimento); nenhum texto livre em URL (`searchCiap2` é POST);
- a cor sugerida nunca é calculada na tela; resposta velha do `suggest` é descartada; concluir fica travado enquanto a sugestão é recalculada;
- a justificativa aparece quando a final difere da sugerida **ou** quando o api pediu (422); todo texto livre com o `FrozenTextNotice`;
- 409 de corrida (`already_screening`, `not_waiting`, `not_in_progress`, `attendance_not_waiting`, `not_reassessable`, `already_resolved`) recarregam e avisam, sem erro parado;
- o editor continua com o JSON como fonte: abrir "Regras de cor" não reescreve a definição; `risk_rules` vazio continua no JSON; o construtor só emite condição estruturada;
- "gerar de novo" só com step-up e só para `municipal_admin`;
- nenhum componente novo além dos listados (interface tem ciclo próprio); o único acréscimo comum é `DataTable.rowStyle`.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Rode o api da branch do módulo 18 na porta **3035**, com a semente do módulo (spec §10 e Task 17 do plano do api: protocolo `acolhimento` ativo em Curitiba, enfermeira e técnica de enfermagem com vínculo na UBS da semente, unidade com `screening_scope = walk_in`, dois cidadãos de demanda espontânea com check-in). Depois o Vite do worktree apontando para ele:

```bash
cd apps/dashboard/.claude/mod18 && VITE_API_PROXY_TARGET=http://localhost:3035 npx vite --port 5185 --host 0.0.0.0
```

Abra `http://curitiba.localhost:5185/dashboard/`. O usuário faz o login; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Confira com screenshot:
- como a enfermeira da semente, em Atendimento, escolhendo a UBS: o painel "Acolhimento" lista os dois cidadãos por chegada, com a espera;
- "Iniciar escuta" no primeiro: queixa "hipertensão" → K86; PA 185/110 → os dois campos destacados e a cor sugerida **vermelho** com o motivo; destino "consulta no dia"; concluir → a pessoa some do acolhimento e aparece **no topo** da fila do profissional, linha destacada em vermelho;
- no segundo: queixa e sinais sem alerta, mudar a cor para verde (justificativa), destino "agendar" com prazo 15 → o pedido aparece em "Pedidos de agendamento" da recepção com o rótulo "Acolhimento";
- "Reavaliar" no primeiro (ainda esperando): nova revisão; como médico da semente, "Chamar" → a escuta abre no atendimento chamado com as duas revisões;
- como a recepção (`admin@curitiba.demo`, que tem `citizen_verifier` em dev): a fila mostra só a cor e a espera, sem "Ver escuta"/"Reavaliar", e o painel "Acolhimento" não aparece;
- como `admin@curitiba.demo` em Unidades: trocar o escopo da UBS para "todos" e voltar;
- como autor da `SignatureCrew`, no Editor de protocolo: "Novo acolhimento (modelo)" troca a coluna; montar "SpO2 até 89 → vermelho" no construtor; o simulador com SpO2 88 sugere vermelho com "regra N: …";
- em Produção e-SUS (com `ledi_export` ligado): a ficha da escuta gerada ou listada em "Fichas que não puderam ser geradas" com o motivo; "Gerar de novo" pede o código do autenticador.

- [ ] **Step 5:** **Pare.** O merge do dashboard só vem depois do merge do api, e só com autorização explícita do usuário. A ordem de deploy é contracts → api → dashboard (contratos §8; spec §11).

---

## Divergências propostas ao contrato

Formatos que este plano precisou e que o contrato não traz (ou traz de outro jeito). O plano já está escrito com a proposta; se uma for recusada, o ajuste fica dito.

**Situação depois da execução (2026-10-08; dashboard `4bf895b`, api `eb732e8`):**

- D1 a D4 foram aceitas pelo contrato §9: a busca de CIAP-2, o `id` no bloco da fila, o `suggest` com `attendance_id` e o `simulate_screening`. O `simulate_screening` saiu **sem `alerts` nem `bmi`** (`{ suggested_color, matched_rules, errors, warnings }`). O simulador não lista alertas do servidor; os campos de sinais vitais destacam pela regra local.
- D5 a D7 estão cobertas pelos "Valores fixados" do plano do api, que o contrato §9 incorpora: alertas, `last_error_codes`, `rejections` como `{ field, code, count }`, pedido com `kind: "screening"` e `origin: "attendance"` (o `origin` não ganhou `"screening"`), e glicemia de 10 a 800 com momento obrigatório.
- D8 está resolvida: `schema_version` é inteiro (contrato §9).
- Acréscimos que este plano não trazia (contrato §10):
  - **`awaiting_screening`** em cada item da fila do profissional. A tela mostra um marcador neutro "aguardando acolhimento", nunca azul, para não ler como risco baixo. A recepção também vê o marcador, porque ele não é dado clínico. A faixa só existe quando a cidade tem protocolo `acolhimento` ativo.
  - **Escuta aninhada na chamada:** `call` e `call_next` devolvem `{ attendance: { ..., screening } }`. A tela lê `result.attendance.screening`. Na chamada aparece só a revisão corrente; as anteriores aparecem em "Ver escuta".
  - **Rótulos dos desfechos novos:** "agendado pelo acolhimento" (`scheduled_from_screening`) e "orientado no acolhimento" (`oriented`) entram nas campanhas e na frase do público. Ficam fora do formulário de encerramento. O Analytics segue com os quatro desfechos antigos (rotasaude/api#49).
  - **Ficha regerada:** `replaces_outbox_id` na ficha. "Reenviar" some na linha que já foi substituída.
- Não feito, opcional: critério de campanha pelo pedido de acolhimento (`kind: "screening"`), chamado R15 no relatório do dashboard.

1. **D1 — busca de CIAP-2 por nome.** O módulo 16 trouxe as tabelas e a importação da terminologia (`ciap2_codes` na plataforma), mas nenhuma rota de busca; a spec §8 pede "CIAP-2 por nome". Proposta: `POST /attendance/ciap2/search` `{ "q": "<texto>" }` → `{ "items": [ { "code", "label" } ] }` (até 20, só do release ativo, por código exato ou por descrição sem acento, a partir de 2 caracteres); 503 `terminology_unavailable` sem release ativo; mesmo acesso da escuta. POST porque o termo pode descrever a queixa (texto livre fora da URL). O mapa de arquivos do plano do api prevê um `ciap2_codes_controller` (Task 12 dele): se lá a rota sair diferente, muda só `searchCiap2` e seu teste (Task 1).
2. **D2 — `id` no bloco `screening` da fila do profissional.** O contrato §4 dá `{ color, destination, waited_minutes }`. Sem o id, "Ver escuta" (Em atendimento, depois de recarregar a página) e "Reavaliar" (quem espera com destino "no dia") não têm como chamar `GET /attendance/screenings/:id` nem `reassess`. Proposta: `{ "id", "color", "destination", "waited_minutes" }`. O id não é dado clínico, e a recepção continua recebendo 403 no `GET`. Recusada, a tela esconde os dois botões (o tipo já é `id?`) e a escuta só aparece na resposta da chamada.
3. **D3 — `suggest` com `attendance_id` no lugar de `citizen_id`.** A reavaliação parte da fila do profissional e da escuta, que não trazem o id do cidadão; e ligar a sugestão ao atendimento deixa o api conferir que a pessoa está em atendimento numa unidade do vínculo (sem isso, `suggest` com `citizen_id` qualquer revelaria idade e sexo de qualquer cidadão pela cor). Proposta: `{ "ciap2_code", "vitals", "attendance_id" }`. Recusada, o contrato precisa trazer `citizen.id` no `<screening>` e no item da fila, e `suggestScreening` volta a mandar `citizen_id`.
4. **D4 — simulador do editor.** O `suggest` do contrato usa o protocolo **ativo**; o simulador da spec §8 confere o **rascunho** no editor. Proposta: `POST /authoring/protocols/simulate_screening` `{ "definition", "ciap2_code": "K86" | null, "vitals", "profile": { "age", "sex" } }` → 200 `{ "suggested_color", "matched_rules", "alerts", "bmi", "errors", "warnings" }`, sem gravar, como o `simulate_offer` (definição que falha no gate devolve 200 com `errors`).
5. **D5 — listas fechadas que a tela traduz.** O contrato §3 mostra só `"systolic_high", ...` nos alertas. Este plano segue a lista do `Screenings::VitalSigns` do plano do api (`systolic_high`, `diastolic_high`, `heart_rate_high`, `heart_rate_low`, `respiratory_rate_high`, `temperature_high`, `spo2_low`, `glucose_low`, `glucose_high`, `pain_severe`) e mostra cru o que não casar. Proposta: o contrato listar os alertas e também os campos e códigos de `last_error_codes` (o plano do api, Desvio 13, fixa códigos `required`, `invalid`, `not_allowed`, `out_of_range`, `duplicate`, `unknown` e o transporte `http_error`, `unreachable`, `invalid_url`, `login_failed`, `internal_error`) e a forma exata de `rejections` (`{ field, code, count }`); a tela hoje mostra `<campo> · <código>` sem tradução.
6. **D6 — pedido nascido no acolhimento na fila da recepção.** O contrato não diz como o pedido `kind = screening` aparece em `GET /attendance/units/:id/requests`. Este plano aceita `kind: "screening"` (rótulo "Acolhimento") e `origin` `"screening"` além dos de hoje, sem depender de `origin_unit_name`. Proposta: o contrato fixar `kind: "screening"` e o `origin` que o api mandar (o plano do api grava `origin_attendance_id` e `origin_screening_id`, Desvio 9).
7. **D7 — glicemia.** Spec §3.1 dá 10–1000 mg/dL; o plano do api fixou 10–800 (o LEDI recusa acima, Desvio 3) e exige o momento junto. A tela segue 800 e o momento obrigatório. Proposta: o contrato registrar os dois (e o código de erro de glicemia sem momento, que hoje a tela evita antes do envio).
8. **D8 — `schema_version` do exemplo do contrato §1** é texto (`"1.6.0"`), mas o schema o tem como inteiro; o modelo do editor (`SCREENING_TEMPLATE`) não usa a chave. Mesma correção pedida no plano do `contracts` (C1).

## Self-review

- **Cobertura (spec §8 e contrato §3–§6):**
  - Atendimento → Acolhimento: fila, chamar (iniciar/retomar) — Task 6; formulário com CIAP-2 por nome — Tasks 3 e 5; sinais com alerta e IMC — Tasks 2 e 4; cor sugerida com motivo, cor final e justificativa — Tasks 4 e 5; destino e campos por destino, com o pedido do módulo 17 — Tasks 2 e 5; abandonar — Task 5; reavaliar — Tasks 5 e 7;
  - fila do profissional com cor e destaque do vermelho, escuta no atendimento chamado, recepção só cor — Task 7;
  - Unidades com `screening_scope` — Task 8;
  - editor com `kind: screening`, construtor com `vitals.*`/`complaint.ciap2`, simulador com sinais vitais, rascunho inicial — Tasks 9 e 10;
  - Produção: "Fichas que não puderam ser geradas" com "gerar de novo" sob step-up, `last_error_codes` no lugar de `last_error`, `rejections` por campo e código — Task 11;
  - spec §6 (LGPD na tela: recepção, texto livre, trilha de leitura pelo `GET`) — Tasks 1, 5, 7 e a revisão da Task 12; spec §9 (front com relógio fixo; prova no navegador) — Tasks 6, 7 e 12.
- **Placeholders:** nenhum. Arquivos novos vêm inteiros; os existentes, por diff (contexto de `origin/main` `ab00e4f` ou do estado da task anterior). Todo o código e os testes foram rodados numa cópia de `origin/main` em 2026-10-07 (Tasks 1–11 em sequência: 125 arquivos, 1329 testes, `tsc` limpo, build ok).
- **Consistência de nomes:** `SCREENING_QUEUE_KEY` e `[ "unitQueue", unit.id ]` (Task 6) são as chaves que a Task 7 invalida; `ScreeningForm` (Task 5) é usado com as mesmas props nas Tasks 6 e 7; `SuggestionState` e `forceReason` (Task 4) usados na Task 5; `VitalSignsFields` (Task 4) reaproveitado no simulador (Task 10); `fieldsFor("screening")` (Task 9) usado no `RiskRulesPanel` (Task 10); `NOW18`, `revision`, `screening`, `queueItem`, `suggestion` (Task 1) usados nas Tasks 4–7 e 10; `COLOR_*`, `VITALS`, `alertLabel`, `screeningError` (Task 2) usados nas Tasks 3–7, 9 e 10.
- **Review Focus:** 1 — Task 5 ("resposta velha…"); 2 — Task 4 ("forceReason…") e Task 5 ("422 color_change_reason_required…"); 3 — Task 6 ("already_screening…") e Task 5 ("atendimento que saiu da espera…"); 4 — Task 7 ("recepção vê só a cor…") e Task 6 (Attendance, "acolhimento (módulo 18)…"); 5 — Task 2 (tabelas) e Task 4 ("pressão pela metade…").
