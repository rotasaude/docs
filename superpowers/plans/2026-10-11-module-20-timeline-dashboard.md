# Módulo 20 — Linha do tempo do paciente (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 20** rodando na branch dele (`feat/mod-20-timeline`, porta **3040**) com `GET /attendance/timeline` e o interruptor `patient_timeline` na sessão. Ele é **mergeado antes** deste (contrato §8: api → dashboard e wpda → maintenance). As Tasks 0–5 não dependem dele (os testes usam dublês); a prova da Task 6 depende.

**Goal:** Lado dashboard de F-20.3: a aba **"Linha do tempo"** no painel do paciente — lista cronológica da pessoa (triagens, atendimentos, escutas, agendamentos, consultas, documentos, Notificações e campanhas), filtros por tipo e período, "Carregar mais" pelo cursor, o detalhe de cada entrada aberto nas telas que já existem (consulta, documento, escuta, pedido de agendamento), o aviso de histórico parcial no modo integrado e a leitura fora do atendimento pela abertura justificada do 19a.

**Architecture:** O cliente HTTP ganha uma seção nova no fim de `src/lib/api.ts` (tipos do contrato §2–§3 e `getTimeline`, que põe na URL só ids, tipos, datas e o cursor). As regras de tela ficam em `src/lib/timeline.ts` (puras: rótulos dos tipos, filtros, junção das páginas sem repetir, qual entrada abre qual tela, frases das recusas). Componentes novos em `src/modules/timeline/`: `PatientTimeline` (lista com `useInfiniteQuery`, `gcTime: 0`), `TimelineDetail` (escolhe a tela existente: `ConsultationLoader`, `ScreeningDetailLoader`, `RequestDetailPanel` ou o `DocumentDetail` novo, que lê `getClinicalDocument` e imprime pelo `printDocument` do 19c) e `JustifiedTimeline` (a abertura justificada sem prontuário, para o modo integrado). As telas existentes ganham o mínimo: `PatientPanel` (abas "Resumo" e "Linha do tempo" quando recebe `timeline`), `ConsultationWorkspace` e `JustifiedRecord` (passam a pessoa), `UnitQueue`/`Attendance` (botão "Linha do tempo" no atendimento chamado quando a cidade não está no modo `record`), `ClinicalRecord`/`OpeningForm`/`modules.ts` (abertura justificada também com só `patient_timeline`). O dashboard nunca decide o que o api garante: quem lê, o que entra na pessoa, a ordem e o que é parcial vêm do api.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5 (`useInfiniteQuery`), Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("../lib/api")`), Vite 5 (base `/dashboard/`, proxy de dev).

**Spec:** `docs/superpowers/specs/2026-10-11-module-20-patient-timeline-design.md` (§3 "a pessoa", §4 tabela de tipos, §5 leitura e trilha, §8 "Dashboard", §9, §12 F-20.3), `docs/adr/0035.md` e o contrato `docs/superpowers/plans/2026-10-11-module-20-timeline-contracts.md` (§1–§3; o que ele não fixa está em "Divergências propostas ao contrato", no fim).

## Global Constraints

- Rota (contrato §3; sessão da cidade, cookie): `GET /attendance/timeline?patient_id=|attendance_id=|opening_id=&before=&limit=&kinds[]=&from=&to=` → `{ person: { display_name, partial, patient_id|null }, entries: [<entry>], next_before|null }`. `limit` 50 (máximo 100); `before` é cursor opaco (`occurred_at|kind|id`), devolvido tal como veio. Recusas: 403 `out_of_context`, `opening_required`, `missing_role`, `feature_disabled` (`feature: "patient_timeline"`); 422 `invalid_cursor`, `invalid_period`, `invalid_kinds`; 409 `citizen_not_verified` (atendimento de par só declarado; plano do api); 404 `not_found`.
- `<entry>` (contrato §2): `{ id: "<kind>:<source_id>", kind, occurred_at, unit: { id, name }|null, professional: { name, cbo_code }|null, title, summary|null, status|null, detail_ref: { type, id } }`. `kind` ∈ `triage` | `attendance` | `screening` | `appointment` | `consultation` | `clinical_document` | `controlled_notification` | `campaign`; desconhecido aparece cru, nunca some nem vira `undefined`. `title` e `summary` chegam prontos em português; `status` é o código da origem (`completed`, `in_care`, `confirmed`, `finalized`, `issued`, `read`…), que a tela traduz por `statusLabel` — código desconhecido aparece cru (Divergência D3).
- Abre detalhe (Divergência D2): `detail_ref.type` `consultation` → `ConsultationLoader`; `clinical_document` (também da Notificação de papel) → `DocumentDetail`; `screening` → `ScreeningDetailLoader`; `appointment_request` → `RequestDetailPanel`. Qualquer outro tipo (`triage`, `attendance`, `campaign`, desconhecido) não tem botão "Abrir".
- Na URL vão só ids, tipos, datas (`AAAA-MM-DD`) e o cursor — nunca CPF, nome ou texto. A tela recebe a pessoa por `attendance_id` (em contexto), `opening_id` (abertura justificada) ou `patient_id`; a abertura continua pedindo o CPF **no corpo do POST** (19a).
- Interruptor (spec §3): `patient_timeline` na sessão liga a aba, o botão da fila e a abertura justificada sem prontuário; desligado, tudo é exatamente o do 19d. 403 `feature_disabled` → "a linha do tempo está desligada nesta cidade".
- Histórico parcial (spec §4 decisão 4; contrato §3): `person.partial: true` → nota `role="note"` "histórico parcial" com `PARTIAL_NOTICE`.
- Abertura justificada (19a, ADR 0031): a linha do tempo pela abertura encerra junto com ela — 403 `opening_required` (ou `out_of_context`, só na leitura pela abertura) tira os dados da tela e do cache (`TIMELINE_KEY`, `CONSULTATION_KEY`). No modo integrado a abertura pode vir com `patient_id: null` (Divergência D1).
- Leituras com dado de saúde com `gcTime: 0` e `staleTime: 0`; sem `console`, `localStorage` ou `sessionStorage`.
- Interface tem ciclo próprio: use os componentes e estilos existentes (`Panel`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `SegmentedControl`, `formStyles`), sem redesign. Nenhum componente comum muda.
- Testes que dependem de "agora" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(new Date(NOW19))` e `vi.useRealTimers()` no `afterEach`; a sessão de teste é criada **depois** do `setSystemTime`.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.

## Ambiente de execução

- Worktree `apps/dashboard/.claude/mod20`, branch `feat/mod-20-timeline` a partir de `origin/main` (Task 0). Caminhos de arquivo relativos a ele; comandos a partir da raiz do monorepo (`/Users/eduardovrocha/Development/ioit.solutions/rota-saude`).
- Testes: `cd apps/dashboard/.claude/mod20 && npx vitest run <arquivos>`. Tipos antes de cada commit: `npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`, `(el as HTMLButtonElement).disabled`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. `fmtDateTime("2026-09-10T14:30:00-03:00")` → `"10/09/2026, 14:30"`.
- O api do módulo 20 roda em dev na porta **3040** (plano do api). O Vite do worktree sobe **num container da rede do compose**, na **5190**, com o proxy para `http://api:3040` (Task 6). `/attendance` e `/clinical_record` já estão no proxy do `vite.config.ts`: nada muda nele.

## Review Focus

1. **"Carregar mais" repete ou perde entrada.** Duas entradas no mesmo instante, ou uma que muda entre duas páginas, não podem aparecer duas vezes; sem `next_before` o botão some; o cursor vai exatamente como veio. Testes: Task 2, "mergePages: junta sem repetir, na ordem"; Task 3, "carregar mais: pede com o cursor e não repete entrada".
2. **A abertura justificada vence no meio da leitura.** O "Carregar mais" (ou abrir um documento) depois dos 30 minutos volta 403 `opening_required`: a lista some da tela e do cache e a pessoa pode abrir de novo — nunca uma lista velha com erro parado. Testes: Task 3, "403 opening_required ao carregar mais encerra a abertura"; Task 5, "a abertura acaba: a linha do tempo sai da tela e do cache".
3. **Filtro inválido ou trocado com páginas carregadas.** Período invertido ou nenhum tipo marcado não consulta e diz por quê; filtro novo recomeça da primeira página (sem cursor velho); 422 `invalid_cursor`/`invalid_period` vira frase com "Atualizar". Testes: Task 3, "filtros: tipos e período vão para a consulta e voltam à primeira página", "período invertido ou nenhum tipo: diz o motivo e não consulta" e "422 invalid_cursor: diz para atualizar; Atualizar relê".
4. **Api mais novo traz tipo ou detalhe que a tela não conhece.** Entrada de `kind` desconhecido (ex.: `vaccination`) aparece com o tipo cru, sem "Abrir" e sem quebrar a lista; campos nulos viram "—". Teste: Task 3, "entrada de tipo desconhecido: aparece crua e sem Abrir".
5. **Modo integrado: sem prontuário, mas com linha do tempo.** O profissional chama a pessoa na fila e precisa da linha do tempo (botão "Linha do tempo", não "Consulta"), com o aviso de histórico parcial; fora do atendimento, a abertura justificada abre só a linha do tempo, mesmo com `patient_id: null`. Testes: Task 4, "modo integrado: Linha do tempo no atendimento chamado, com o aviso de parcial"; Task 5, "modo integrado: a abertura justificada abre só a linha do tempo".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | conferência do main e do api do módulo 20, worktree, base | 0 |
| `src/lib/api.ts`, `src/lib/api.timeline.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `src/test/timelineFixtures.ts` | tipos do contrato e `getTimeline`; `Opening.patient_id` anulável; interruptor `patient_timeline`; dados de teste | 1 |
| `src/lib/timeline.ts`, `src/lib/timeline.test.ts` | regras de tela da linha do tempo | 2 |
| `src/modules/timeline/PatientTimeline.tsx` (+ teste), `src/modules/timeline/TimelineDetail.tsx`, `src/modules/timeline/DocumentDetail.tsx`, `src/modules/timeline/TimelineDetail.test.tsx` | lista, filtros, "Carregar mais", parcial, recusas; detalhe nas telas existentes | 3 |
| `src/modules/consultation/PatientPanel.tsx`, `src/modules/consultation/PatientPanel.timeline.test.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/attendance/UnitQueue.tsx`, `src/modules/attendance/UnitQueue.timeline.test.tsx`, `src/modules/Attendance.tsx` | aba no painel do paciente (em contexto, modo `record`); botão na fila (modo integrado) | 4 |
| `src/modules/clinicalRecord/JustifiedTimeline.tsx`, `src/modules/clinicalRecord/JustifiedRecord.tsx`, `src/modules/clinicalRecord/OpeningForm.tsx`, `src/modules/ClinicalRecord.tsx`, `src/lib/clinicalRecord.ts`, `src/modules/ClinicalRecord.timeline.test.tsx`, `src/shell/modules.ts`, `src/shell/modules.test.ts`, `README.md` | abertura justificada com a linha do tempo, nos dois modos | 5 |
| — | suíte, build, revisão e prova no navegador | 6 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo a URL (só ids, tipos, datas e cursor; `kinds[]` repetido) (Task 1); regras puras com tabela de casos (Task 2); a lista com `vi.mock("../../lib/api")` e o detalhe por dublê, cobrindo ordem, cursor, filtros, parcial, vazio, tipo desconhecido e cada recusa que muda o comportamento da tela (Task 3); o detalhe com dublês das telas existentes e do `printDocument` (Task 3); as integrações pelos pontos de entrada reais — `PatientPanel`, `ConsultationWorkspace`, `UnitQueue`, `ClinicalRecord` — com o relógio fixo onde há contagem (Tasks 4 e 5). A prova final é no navegador, contra o api do módulo 20, nos dois modos (Task 6).

---

### Task 0: Conferência do main e do api do módulo 20, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (19d e os consertos do 19, `daafeee` ou mais novo).
- Produces: worktree `apps/dashboard/.claude/mod20` na branch `feat/mod-20-timeline`; a base anotada.

- [ ] **Step 1: Confira as peças que este plano altera**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/dashboard fetch origin
/opt/homebrew/bin/git -C apps/dashboard log --oneline -3 origin/main
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/lib/api.ts | grep -n \
  "export interface Opening {\|export function getJustifiedRecord\|export function getClinicalDocument\|^const ATTENDANCE_BASE\|// ─── Rascunhos de consulta abertos"
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/lib/features.ts | grep -n "export type FeatureKey\|controlled_prescriptions: \"Receita"
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/modules/consultation/PatientPanel.tsx | grep -n \
  "export function PatientPanel\|const rev = record.today_screening\|<KeyValue k=\"CPF\"\|{besideProblems}"
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/modules/consultation/ConsultationWorkspace.tsx | grep -n \
  "const documentsOn = canUseDocuments(user);\|<PatientPanel record={record.data}"
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/modules/clinicalRecord/JustifiedRecord.tsx | grep -n \
  "queryFn: () => getJustifiedRecord(opening.patient_id)\|<PatientPanel record={record.data}\|queryClient.removeQueries({ queryKey: \[ CONSULTATION_KEY \] });"
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/modules/attendance/UnitQueue.tsx | grep -n \
  "| { kind: \"consultation\"; attendanceId: string };\|clinicalDocuments = false, professional = false\|Ver escuta\|{clinicalDocuments && declaring && ("
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/modules/Attendance.tsx | grep -n 'clinicalRecord={hasFeature(user, "clinical_record")}'
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/shell/modules.ts | grep -n "const canClinicalRecord ="
```

Expected: cada `grep` acha as linhas pedidas (as âncoras dos diffs). Se alguma mudou, anote: as tasks aplicam a mesma intenção sobre o texto real.

- [ ] **Step 2: Confira o api do módulo 20 no ar**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/api branch --list feat/mod-20-timeline
docker compose exec -T -w /rails/.claude/mod20 api bin/rails routes 2>/dev/null | grep -E "timeline|openings" | head
```

Expected: a branch e as rotas `GET /attendance/timeline` e `POST /clinical_record/openings`. Sem o api do módulo 20, as Tasks 1–5 seguem; a Task 6 espera.

- [ ] **Step 3: Crie o worktree e ligue o `node_modules`** (`/.claude/` já está no `.gitignore` do dashboard)

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/dashboard worktree add .claude/mod20 -b feat/mod-20-timeline origin/main
ln -s ../../node_modules apps/dashboard/.claude/mod20/node_modules
cd apps/dashboard/.claude/mod20 && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: tudo verde e `tsc-ok`. **Anote** a base (arquivos e testes). A Task 6 espera a base mais **7 arquivos de teste novos** (`api.timeline`, `timeline`, `PatientTimeline`, `TimelineDetail`, `PatientPanel.timeline`, `UnitQueue.timeline`, `ClinicalRecord.timeline`).

---

### Task 1: Tipos, cliente HTTP, interruptor e dados de teste

**Files:**
- Modify: `src/lib/api.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`
- Create: `src/test/timelineFixtures.ts`
- Test: `src/lib/api.timeline.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `ATTENDANCE_BASE` (existentes em `api.ts`).
- Produces (em `src/lib/api.ts`):
  ```ts
  export type TimelineKind = "triage" | "attendance" | "screening" | "appointment" | "consultation"
    | "clinical_document" | "controlled_notification" | "campaign";
  export interface TimelineDetailRef { type: string; id: string }
  export interface TimelineEntry {
    id: string; kind: TimelineKind | string; occurred_at: string;
    unit: { id: string; name: string } | null; professional: { name: string; cbo_code: string | null } | null;
    title: string; summary: string | null; status: string | null; detail_ref: TimelineDetailRef | null;
  }
  export interface TimelinePerson { display_name: string; partial: boolean; patient_id: string | null }
  export interface TimelinePage { person: TimelinePerson; entries: TimelineEntry[]; next_before: string | null }
  export type TimelineSubject = { attendanceId: string } | { patientId: string } | { openingId: string };
  export interface TimelineFilters { kinds: TimelineKind[]; from: string; to: string }
  export const TIMELINE_PAGE_SIZE = 50;
  export function getTimeline(subject: TimelineSubject, filters: TimelineFilters, before: string | null): Promise<TimelinePage>;
  // Opening passa a ter patient_id: string | null (Divergência D1).
  ```
  `FeatureKey` ganha `"patient_timeline"` (rótulo "Linha do tempo do paciente"). Em `src/test/timelineFixtures.ts`: `entry(over)`, `page(over)` e as entradas `APPOINTMENT`, `DOCUMENT`, `CONSULTATION`, `SCREENING`, `TRIAGE` (ordem decrescente de `occurred_at`).

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.timeline.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { getTimeline, type TimelineFilters } from "./api";

const ALL: TimelineFilters = { kinds: [], from: "", to: "" };
const PAGE = { person: { display_name: "Joana Lima", partial: false, patient_id: "pa1" }, entries: [], next_before: null };

describe("getTimeline (contrato §3)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => {
    fetchMock = vi.fn(() => Promise.resolve(new Response(JSON.stringify(PAGE), { status: 200, headers: { "Content-Type": "application/json" } })));
    vi.stubGlobal("fetch", fetchMock);
  });
  afterEach(() => vi.unstubAllGlobals());

  function urlOf(i = 0): URL {
    return new URL(String(fetchMock.mock.calls[i][0]), "http://curitiba.localhost");
  }

  it("em contexto: attendance_id, limit 50 e nada mais", async () => {
    expect(await getTimeline({ attendanceId: "a1" }, ALL, null)).toEqual(PAGE);
    const url = urlOf();
    expect(url.pathname).toBe("/attendance/timeline");
    expect([ ...url.searchParams.entries() ]).toEqual([ [ "attendance_id", "a1" ], [ "limit", "50" ] ]);
    const init = fetchMock.mock.calls[0][1] as RequestInit;
    expect(init.method).toBeUndefined();
    expect(init.credentials).toBe("include");
  });

  it("abertura e paciente; kinds[] repetido, período e o cursor como veio", async () => {
    await getTimeline({ openingId: "op1" }, { kinds: [ "consultation", "clinical_document" ], from: "2026-09-01", to: "2026-09-30" },
      "2026-09-10T14:40:00-03:00|clinical_document|dc1");
    const url = urlOf();
    expect(url.searchParams.get("opening_id")).toBe("op1");
    expect(url.searchParams.getAll("kinds[]")).toEqual([ "consultation", "clinical_document" ]);
    expect(url.searchParams.get("from")).toBe("2026-09-01");
    expect(url.searchParams.get("to")).toBe("2026-09-30");
    expect(url.searchParams.get("before")).toBe("2026-09-10T14:40:00-03:00|clinical_document|dc1");

    await getTimeline({ patientId: "pa1" }, ALL, null);
    expect(urlOf(1).searchParams.get("patient_id")).toBe("pa1");
    expect(urlOf(1).searchParams.has("before")).toBe(false);
  });

  it("nunca põe CPF, nome ou texto na URL", async () => {
    await getTimeline({ attendanceId: "a1" }, ALL, null);
    expect(String(fetchMock.mock.calls[0][0])).not.toMatch(/cpf|name|nome|\d{11}/i);
  });

  it("recusa vira ApiError com o código do corpo", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(new Response(JSON.stringify({ error: "opening_required" }), { status: 403 })));
    await expect(getTimeline({ openingId: "op1" }, ALL, null)).rejects.toMatchObject({ status: 403, body: { error: "opening_required" } });
  });
});
```

Em `src/lib/features.test.ts`, no fim do `describe` "features da sessão (contratos §1)":

```ts
  it("patient_timeline (módulo 20) tem rótulo e é lida da sessão", () => {
    expect(featureLabel("patient_timeline")).toBe("Linha do tempo do paciente");
    expect(hasFeature({ features: [ "patient_timeline" ] }, "patient_timeline")).toBe(true);
    expect(hasFeature({ features: [ "clinical_record" ] }, "patient_timeline")).toBe(false);
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod20 && npx vitest run src/lib/api.timeline.test.ts src/lib/features.test.ts`
Expected: FAIL — `getTimeline` não existe; `featureLabel("patient_timeline")` devolve a chave crua (e o `tsc` recusaria `"patient_timeline"` como `FeatureKey`).

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`, troque a linha do `Opening` (na seção do 19a):

```ts
export interface Opening { opening_id: string; patient_id: string; expires_at: string }
```

por

```ts
// Módulo 20 (Divergência D1): no modo integrated a abertura vale para a pessoa
// (pares validados do CPF) e pode não ter paciente do 19a.
export interface Opening { opening_id: string; patient_id: string | null; expires_at: string }
```

e acrescente, logo antes do comentário `// ─── Rascunhos de consulta abertos (fix51, api#51)`:

```ts
// ─── Linha do tempo do paciente (módulo 20, ADR 0035; contrato 2026-10-11 §1–§3) ───
// Só ids, tipos, datas e o cursor na URL — nunca CPF, nome ou texto. Ler gera
// a trilha `clinical_record.viewed` com access "timeline" no api; quem lê e o
// que entra na pessoa é do api (ClinicalRecord::Access generalizado).

export type TimelineKind =
  | "triage" | "attendance" | "screening" | "appointment" | "consultation"
  | "clinical_document" | "controlled_notification" | "campaign";

export interface TimelineDetailRef { type: string; id: string }
export interface TimelineEntry {
  id: string; kind: TimelineKind | string; occurred_at: string;
  unit: { id: string; name: string } | null;
  professional: { name: string; cbo_code: string | null } | null;
  // title e summary em português, prontos; status é o código da origem
  // (finalized, issued, confirmed…), traduzido na tela (Divergência D3).
  title: string; summary: string | null; status: string | null;
  // Divergência D2: nulo (ou tipo sem tela) = a entrada não abre detalhe.
  detail_ref: TimelineDetailRef | null;
}
export interface TimelinePerson { display_name: string; partial: boolean; patient_id: string | null }
export interface TimelinePage { person: TimelinePerson; entries: TimelineEntry[]; next_before: string | null }
// Em contexto (atendimento), pela abertura justificada ou pelo paciente.
export type TimelineSubject = { attendanceId: string } | { patientId: string } | { openingId: string };
// `kinds` vazio = todos; datas AAAA-MM-DD ou "".
export interface TimelineFilters { kinds: TimelineKind[]; from: string; to: string }
export const TIMELINE_PAGE_SIZE = 50;

export function getTimeline(subject: TimelineSubject, filters: TimelineFilters, before: string | null): Promise<TimelinePage> {
  const params = new URLSearchParams();
  if ("attendanceId" in subject) params.set("attendance_id", subject.attendanceId);
  else if ("patientId" in subject) params.set("patient_id", subject.patientId);
  else params.set("opening_id", subject.openingId);
  for (const kind of filters.kinds) params.append("kinds[]", kind);
  if (filters.from) params.set("from", filters.from);
  if (filters.to) params.set("to", filters.to);
  if (before) params.set("before", before);
  params.set("limit", String(TIMELINE_PAGE_SIZE));
  return jsonFetch(`${ATTENDANCE_BASE}/timeline?${params.toString()}`);
}

```

Em `src/modules/clinicalRecord/JustifiedRecord.tsx` (o `tsc` passa a recusar `string | null`), troque

```tsx
    queryKey: [ JUSTIFIED_KEY, opening.opening_id ], queryFn: () => getJustifiedRecord(opening.patient_id),
    enabled: !over, gcTime: 0, staleTime: 0, retry: false
```

por

```tsx
    // Só monta com paciente (ClinicalRecord escolhe JustifiedTimeline sem ele).
    queryKey: [ JUSTIFIED_KEY, opening.opening_id ], queryFn: () => getJustifiedRecord(opening.patient_id ?? ""),
    enabled: !over && opening.patient_id !== null, gcTime: 0, staleTime: 0, retry: false
```

Em `src/lib/features.ts`, a união passa a

```ts
export type FeatureKey =
  | "ledi_export" | "cadsus_lookup" | "clinical_record" | "digital_signature" | "clinical_documents" | "controlled_prescriptions"
  | "patient_timeline";
```

e no `FEATURE_LABEL`, depois da linha de `controlled_prescriptions`:

```ts
  controlled_prescriptions: "Receita de controle especial e de antimicrobiano (SNCR)",
  patient_timeline: "Linha do tempo do paciente"
```

(a linha de `controlled_prescriptions` ganha a vírgula).

```ts
// src/test/timelineFixtures.ts
// Dados comuns aos testes do módulo 20 (linha do tempo). A ordem das
// constantes é a do api (occurred_at decrescente). title e summary são
// ilustrativos (vêm prontos do api); status é o código da origem (D3).
import type { TimelineEntry, TimelinePage } from "../lib/api";

export function entry(over: Partial<TimelineEntry> = {}): TimelineEntry {
  return {
    id: "consultation:cs0", kind: "consultation", occurred_at: "2026-09-10T14:30:00-03:00",
    unit: { id: "u1", name: "UBS Centro" }, professional: { name: "Dra. Helena Prado", cbo_code: "225142" },
    title: "Consulta agendada", summary: "T90 Diabetes não insulino-dependente — avaliado", status: "finalized",
    detail_ref: { type: "consultation", id: "cs0" }, ...over
  };
}

export const APPOINTMENT = entry({
  id: "appointment:ap1", kind: "appointment", occurred_at: "2026-10-01T09:00:00-03:00",
  professional: { name: "Enf. Lúcia Prado", cbo_code: "223565" }, title: "Retorno — Consulta de enfermagem",
  summary: null, status: "confirmed", detail_ref: { type: "appointment_request", id: "rq1" }
});
export const DOCUMENT = entry({
  id: "clinical_document:doc2", kind: "clinical_document", occurred_at: "2026-09-10T14:40:00-03:00",
  title: "Receita", summary: "comum · digital", status: "issued", detail_ref: { type: "clinical_document", id: "doc2" }
});
export const CONSULTATION = entry();
export const SCREENING = entry({
  id: "screening:sc1", kind: "screening", occurred_at: "2026-09-10T13:50:00-03:00",
  professional: { name: "Enf. Lúcia Prado", cbo_code: "223565" }, title: "Escuta inicial",
  summary: "amarelo · R74 · atendimento no dia", status: "completed", detail_ref: { type: "screening", id: "sc1" }
});
export const TRIAGE = entry({
  id: "triage:tr1", kind: "triage", occurred_at: "2026-09-09T20:15:00-03:00", unit: null, professional: null,
  title: "Triagem respiratória", summary: "prioridade 2", status: "completed", detail_ref: { type: "triage", id: "tr1" }
});

export function page(over: Partial<TimelinePage> = {}): TimelinePage {
  return {
    person: { display_name: "Joana Lima", partial: false, patient_id: "pa1" },
    entries: [ APPOINTMENT, DOCUMENT, CONSULTATION, SCREENING, TRIAGE ],
    next_before: null,
    ...over
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod20 && npx vitest run src/lib/api.timeline.test.ts src/lib/features.test.ts src/modules/clinicalRecord && npx tsc --noEmit`
Expected: PASS (os testes do `JustifiedRecord` seguem verdes: a fixture `opening()` traz `patient_id: "pa1"`) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod20
/opt/homebrew/bin/git add src/lib/api.ts src/lib/api.timeline.test.ts src/lib/features.ts src/lib/features.test.ts \
  src/test/timelineFixtures.ts src/modules/clinicalRecord/JustifiedRecord.tsx
/opt/homebrew/bin/git commit -m "feat: add the patient timeline client and switch to the dashboard

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela da linha do tempo

**Files:**
- Create: `src/lib/timeline.ts`
- Test: `src/lib/timeline.test.ts`

**Interfaces:**
- Consumes: `ApiError`, `errorCode`, `TimelineEntry`, `TimelineFilters`, `TimelineKind`, `TimelinePage`, `TimelineSubject` (Task 1); `featureDisabledKey` (`features.ts`); `fmtDateTime` (`format.ts`).
- Produces:
  ```ts
  export const TIMELINE_KEY = "patientTimeline";
  export const TIMELINE_KINDS: { key: TimelineKind; label: string }[];
  export const ALL_FILTERS: TimelineFilters;                // { kinds: [], from: "", to: "" }
  export const PARTIAL_NOTICE: string;
  export const TIMELINE_EMPTY = "nenhum registro neste período";
  export const TIMELINE_DISABLED = "a linha do tempo está desligada nesta cidade";
  export function timelineKindLabel(kind: string): string;
  export function subjectKey(subject: TimelineSubject): string;
  export interface FilterDraft { kinds: TimelineKind[]; from: string; to: string }
  export const EMPTY_DRAFT: FilterDraft;                    // todos os tipos marcados, sem período
  export function toggleKind(d: FilterDraft, kind: TimelineKind): FilterDraft;
  export function filterProblem(d: FilterDraft): string | null;
  export function filtersFrom(d: FilterDraft): TimelineFilters;
  export function mergePages(pages: TimelinePage[]): TimelineEntry[];
  export type DetailType = "consultation" | "clinical_document" | "screening" | "appointment_request";
  export interface DetailTarget { type: DetailType; id: string }
  export function detailTarget(entry: TimelineEntry): DetailTarget | null;
  export function entryWhen(entry: TimelineEntry): string;
  export function entryWhere(entry: TimelineEntry): string;
  export function openLabel(entry: TimelineEntry): string;  // "Abrir consulta de 10/09/2026, 14:30"
  export function statusLabel(status: string | null): string;    // código → português; nulo → "—"
  export function endsOpening(err: unknown, endOnOutOfContext: boolean): boolean;
  export function timelineError(err: unknown): string;
  ```

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/timeline.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  ALL_FILTERS, EMPTY_DRAFT, TIMELINE_DISABLED, TIMELINE_KINDS, detailTarget, endsOpening, entryWhen, entryWhere,
  filterProblem, filtersFrom, mergePages, openLabel, statusLabel, subjectKey, timelineError, timelineKindLabel, toggleKind
} from "./timeline";
import { APPOINTMENT, CONSULTATION, DOCUMENT, SCREENING, TRIAGE, entry, page } from "../test/timelineFixtures";

const refusal = (status: number, error: string, extra: object = {}) => new ApiError(status, { error, ...extra }, String(status));

describe("tipos", () => {
  it("oito tipos do contrato, em português; desconhecido sai cru", () => {
    expect(TIMELINE_KINDS.map((k) => k.key)).toEqual([
      "triage", "attendance", "screening", "appointment", "consultation", "clinical_document", "controlled_notification", "campaign"
    ]);
    expect(timelineKindLabel("screening")).toBe("Escuta");
    expect(timelineKindLabel("controlled_notification")).toBe("Notificação de receita");
    expect(timelineKindLabel("vaccination")).toBe("vaccination");
  });

  it("subjectKey separa as três formas de chegar à pessoa", () => {
    expect(subjectKey({ attendanceId: "a1" })).toBe("attendance:a1");
    expect(subjectKey({ openingId: "op1" })).toBe("opening:op1");
    expect(subjectKey({ patientId: "pa1" })).toBe("patient:pa1");
  });
});

describe("filtros", () => {
  it("todos marcados = sem kinds[]; parte marcada vai na ordem do contrato", () => {
    expect(filtersFrom(EMPTY_DRAFT)).toEqual(ALL_FILTERS);
    const d = toggleKind(toggleKind(EMPTY_DRAFT, "campaign"), "triage");
    expect(filtersFrom({ ...d, from: "2026-09-01", to: "2026-09-30" })).toEqual({
      kinds: [ "attendance", "screening", "appointment", "consultation", "clinical_document", "controlled_notification" ],
      from: "2026-09-01", to: "2026-09-30"
    });
    expect(toggleKind(d, "triage").kinds).toContain("triage");
  });

  it("período invertido ou nenhum tipo: o motivo", () => {
    expect(filterProblem({ ...EMPTY_DRAFT, from: "2026-10-01", to: "2026-09-01" })).toBe("o início do período é depois do fim");
    expect(filterProblem({ ...EMPTY_DRAFT, kinds: [] })).toBe("marque ao menos um tipo de registro");
    expect(filterProblem({ ...EMPTY_DRAFT, from: "2026-09-01", to: "2026-09-01" })).toBeNull();
    expect(filterProblem({ ...EMPTY_DRAFT, from: "2026-09-01" })).toBeNull();
  });
});

describe("mergePages: junta sem repetir, na ordem", () => {
  it("a entrada repetida na página seguinte aparece uma vez, no primeiro lugar", () => {
    const merged = mergePages([
      page({ entries: [ APPOINTMENT, DOCUMENT ] }),
      page({ entries: [ DOCUMENT, CONSULTATION, SCREENING ] }),
      page({ entries: [ TRIAGE ] })
    ]);
    expect(merged.map((e) => e.id)).toEqual([ APPOINTMENT.id, DOCUMENT.id, CONSULTATION.id, SCREENING.id, TRIAGE.id ]);
    expect(mergePages([])).toEqual([]);
  });
});

describe("detalhe", () => {
  it("abre só as quatro telas que existem", () => {
    expect(detailTarget(CONSULTATION)).toEqual({ type: "consultation", id: "cs0" });
    expect(detailTarget(DOCUMENT)).toEqual({ type: "clinical_document", id: "doc2" });
    expect(detailTarget(SCREENING)).toEqual({ type: "screening", id: "sc1" });
    expect(detailTarget(APPOINTMENT)).toEqual({ type: "appointment_request", id: "rq1" });
    expect(detailTarget(TRIAGE)).toBeNull();
    expect(detailTarget(entry({ detail_ref: null }))).toBeNull();
    expect(detailTarget(entry({ kind: "vaccination", detail_ref: { type: "vaccination", id: "v1" } }))).toBeNull();
  });

  it("quando, onde/quem e o nome do botão", () => {
    expect(entryWhen(CONSULTATION)).toBe("10/09/2026, 14:30");
    expect(entryWhere(CONSULTATION)).toBe("UBS Centro · Dra. Helena Prado");
    expect(entryWhere(entry({ professional: null }))).toBe("UBS Centro");
    expect(entryWhere(TRIAGE)).toBe("—");
    expect(openLabel(CONSULTATION)).toBe("Abrir consulta de 10/09/2026, 14:30");
    expect(openLabel(APPOINTMENT)).toBe("Abrir agendamento de 01/10/2026, 09:00");
  });
});

describe("situação", () => {
  it("códigos das origens em português; desconhecido cru; nulo vira —", () => {
    expect(statusLabel("finalized")).toBe("finalizada");
    expect(statusLabel("issued")).toBe("emitido");
    expect(statusLabel("in_care")).toBe("em atendimento");
    expect(statusLabel("cancelled_by_citizen")).toBe("cancelado pelo cidadão");
    expect(statusLabel("aborted_by_revocation")).toBe("consentimento revogado");
    expect(statusLabel("unread")).toBe("aviso não lido");
    expect(statusLabel("something_new")).toBe("something_new");
    expect(statusLabel(null)).toBe("—");
  });
});

describe("recusas", () => {
  it("a abertura acaba com opening_required; out_of_context só na leitura pela abertura", () => {
    expect(endsOpening(refusal(403, "opening_required"), false)).toBe(true);
    expect(endsOpening(refusal(403, "out_of_context"), true)).toBe(true);
    expect(endsOpening(refusal(403, "out_of_context"), false)).toBe(false);
    expect(endsOpening(refusal(422, "invalid_cursor"), true)).toBe(false);
    expect(endsOpening(new Error("rede"), true)).toBe(false);
  });

  it("frases", () => {
    expect(timelineError(refusal(403, "feature_disabled", { feature: "patient_timeline" }))).toBe(TIMELINE_DISABLED);
    expect(timelineError(refusal(403, "out_of_context")))
      .toBe("esta pessoa não está em atendimento com você — para ler, use a abertura justificada em Prontuário");
    expect(timelineError(refusal(403, "opening_required"))).toBe("a abertura justificada terminou — abra de novo com o motivo");
    expect(timelineError(refusal(403, "missing_role"))).toBe("seu papel não permite ler a linha do tempo");
    expect(timelineError(refusal(422, "invalid_cursor"))).toBe("a lista mudou desde a última página — use Atualizar");
    expect(timelineError(refusal(422, "invalid_period"))).toBe("período inválido — confira as datas");
    expect(timelineError(refusal(404, "not_found"))).toBe("pessoa não encontrada — atualize a fila");
    expect(timelineError(refusal(422, "invalid_kinds"))).toBe("tipo de registro inválido — use Limpar filtros");
    expect(timelineError(refusal(409, "citizen_not_verified")))
      .toBe("esta pessoa ainda não validou o cadastro na unidade — a linha do tempo só existe para cadastro validado");
    expect(timelineError(refusal(500, "boom"))).toBe("não foi possível ler a linha do tempo (500) — tente de novo");
    expect(timelineError(new TypeError("Failed to fetch"))).toBe("sem conexão com o servidor — tente de novo");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod20 && npx vitest run src/lib/timeline.test.ts`
Expected: FAIL — `./timeline` não existe.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/timeline.ts
// Linha do tempo do paciente no painel da cidade (módulo 20, F-20.3; ADR 0035;
// contrato §2–§3). Quem lê, o que entra na pessoa, a ordem e o que é parcial
// são do api: aqui só se monta o pedido, se juntam as páginas e se escolhe a
// tela existente que mostra cada entrada.
import {
  ApiError, errorCode, type TimelineEntry, type TimelineFilters, type TimelineKind, type TimelinePage, type TimelineSubject
} from "./api";
import { featureDisabledKey } from "./features";
import { fmtDateTime } from "./format";

export const TIMELINE_KEY = "patientTimeline";

export const TIMELINE_KINDS: { key: TimelineKind; label: string }[] = [
  { key: "triage", label: "Triagem" },
  { key: "attendance", label: "Atendimento" },
  { key: "screening", label: "Escuta" },
  { key: "appointment", label: "Agendamento" },
  { key: "consultation", label: "Consulta" },
  { key: "clinical_document", label: "Documento" },
  { key: "controlled_notification", label: "Notificação de receita" },
  { key: "campaign", label: "Campanha" }
];

export const ALL_FILTERS: TimelineFilters = { kinds: [], from: "", to: "" };
export const PARTIAL_NOTICE =
  "Histórico parcial: aqui está só o que passou pelo Rota Saúde. As consultas desta cidade são registradas no PEC.";
export const TIMELINE_EMPTY = "nenhum registro neste período";
export const TIMELINE_DISABLED = "a linha do tempo está desligada nesta cidade";

export function timelineKindLabel(kind: string): string {
  return TIMELINE_KINDS.find((k) => k.key === kind)?.label ?? kind;
}

export function subjectKey(subject: TimelineSubject): string {
  if ("attendanceId" in subject) return `attendance:${subject.attendanceId}`;
  if ("patientId" in subject) return `patient:${subject.patientId}`;
  return `opening:${subject.openingId}`;
}

export interface FilterDraft { kinds: TimelineKind[]; from: string; to: string }
export const EMPTY_DRAFT: FilterDraft = { kinds: TIMELINE_KINDS.map((k) => k.key), from: "", to: "" };

export function toggleKind(d: FilterDraft, kind: TimelineKind): FilterDraft {
  return { ...d, kinds: d.kinds.includes(kind) ? d.kinds.filter((k) => k !== kind) : [ ...d.kinds, kind ] };
}

export function filterProblem(d: FilterDraft): string | null {
  if (d.kinds.length === 0) return "marque ao menos um tipo de registro";
  if (d.from && d.to && d.from > d.to) return "o início do período é depois do fim";
  return null;
}

// Todos marcados = sem kinds[] (o api devolve tudo, inclusive tipo novo).
export function filtersFrom(d: FilterDraft): TimelineFilters {
  const all = TIMELINE_KINDS.every((k) => d.kinds.includes(k.key));
  return {
    kinds: all ? [] : TIMELINE_KINDS.map((k) => k.key).filter((k) => d.kinds.includes(k)),
    from: d.from, to: d.to
  };
}

// O cursor composto não repete, mas uma entrada que muda entre duas leituras
// (ex.: documento cancelado) não pode aparecer duas vezes: vale a primeira.
export function mergePages(pages: TimelinePage[]): TimelineEntry[] {
  const seen = new Set<string>();
  const out: TimelineEntry[] = [];
  for (const p of pages) {
    for (const e of p.entries) {
      if (seen.has(e.id)) continue;
      seen.add(e.id);
      out.push(e);
    }
  }
  return out;
}

export type DetailType = "consultation" | "clinical_document" | "screening" | "appointment_request";
export interface DetailTarget { type: DetailType; id: string }
const OPENABLE: ReadonlySet<string> = new Set<DetailType>([ "consultation", "clinical_document", "screening", "appointment_request" ]);

export function detailTarget(entry: TimelineEntry): DetailTarget | null {
  const ref = entry.detail_ref;
  if (!ref || !OPENABLE.has(ref.type) || !ref.id) return null;
  return { type: ref.type as DetailType, id: ref.id };
}

export function entryWhen(entry: TimelineEntry): string {
  return fmtDateTime(entry.occurred_at);
}

export function entryWhere(entry: TimelineEntry): string {
  const parts = [ entry.unit?.name, entry.professional?.name ].filter((p): p is string => !!p);
  return parts.length > 0 ? parts.join(" · ") : "—";
}

export function openLabel(entry: TimelineEntry): string {
  return `Abrir ${timelineKindLabel(entry.kind).toLowerCase()} de ${entryWhen(entry)}`;
}

// Códigos de situação das origens (triagem, atendimento, escuta, agendamento,
// consulta, documento, campanha). Código novo do api aparece cru.
const STATUS_LABEL: Record<string, string> = {
  in_progress: "em andamento",
  completed: "concluído",
  abandoned: "abandonado",
  aborted_by_revocation: "consentimento revogado",
  aborted_by_timeout: "interrompido por tempo",
  aborted_by_cancellation: "cancelado",
  waiting: "aguardando",
  in_care: "em atendimento",
  closed: "encerrado",
  scheduled: "marcado",
  confirmed: "confirmado",
  checked_in: "compareceu",
  cancelled_by_citizen: "cancelado pelo cidadão",
  expired: "não confirmado no prazo",
  no_show: "faltou",
  draft: "rascunho",
  finalized: "finalizada",
  issued: "emitido",
  cancelled: "cancelado",
  read: "aviso lido",
  unread: "aviso não lido"
};

export function statusLabel(status: string | null): string {
  if (!status) return "—";
  return STATUS_LABEL[status] ?? status;
}

export function endsOpening(err: unknown, endOnOutOfContext: boolean): boolean {
  const code = errorCode(err);
  return code === "opening_required" || (endOnOutOfContext && code === "out_of_context");
}

const ERRORS: Record<string, string> = {
  out_of_context: "esta pessoa não está em atendimento com você — para ler, use a abertura justificada em Prontuário",
  opening_required: "a abertura justificada terminou — abra de novo com o motivo",
  missing_role: "seu papel não permite ler a linha do tempo",
  invalid_cursor: "a lista mudou desde a última página — use Atualizar",
  invalid_period: "período inválido — confira as datas",
  not_found: "pessoa não encontrada — atualize a fila",
  invalid_kinds: "tipo de registro inválido — use Limpar filtros",
  citizen_not_verified: "esta pessoa ainda não validou o cadastro na unidade — a linha do tempo só existe para cadastro validado"
};

export function timelineError(err: unknown): string {
  if (featureDisabledKey(err) !== null) return TIMELINE_DISABLED;
  const code = errorCode(err);
  if (code && ERRORS[code]) return ERRORS[code];
  if (err instanceof ApiError) return `não foi possível ler a linha do tempo (${err.status}) — tente de novo`;
  return "sem conexão com o servidor — tente de novo";
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod20 && npx vitest run src/lib/timeline.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod20
/opt/homebrew/bin/git add src/lib/timeline.ts src/lib/timeline.test.ts
/opt/homebrew/bin/git commit -m "feat: add patient timeline screen rules to the dashboard

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Lista da linha do tempo e detalhe nas telas existentes

**Files:**
- Create: `src/modules/timeline/PatientTimeline.tsx`, `src/modules/timeline/TimelineDetail.tsx`, `src/modules/timeline/DocumentDetail.tsx`
- Test: `src/modules/timeline/PatientTimeline.test.tsx`, `src/modules/timeline/TimelineDetail.test.tsx`

**Interfaces:**
- Consumes: `getTimeline`, `getConsultationOptions`, `getClinicalDocument`, `errorCode`, `TimelineEntry`, `TimelineFilters`, `TimelineSubject` (`api.ts`); tudo de `timeline.ts` (Task 2); `ConsultationLoader` (`../consultation/ConsultationView`), `ScreeningDetailLoader` (`../attendance/ScreeningDetail`), `RequestDetailPanel` (`../attendance/RequestDetailPanel`), `printDocument` (`../documents/printDocument`), `kindLabel`, `issueModeLabel`, `documentSituation`, `documentError` (`clinicalDocuments.ts`), `OPTIONS_KEY` (`consultation.ts`), `useAuth`, `DataTable`, `Tag`, `KeyValue`, `formStyles`.
- Produces:
  - `export function PatientTimeline(props: { subject: TimelineSubject; endOnOutOfContext?: boolean; onOpeningRequired?(): void; searchDelayMs?: number }): JSX.Element` — `<section aria-label="Linha do tempo">` com o título "Linha do tempo de <nome>", a nota `role="note"` "histórico parcial", o `<fieldset aria-label="filtros da linha do tempo">` (caixas por tipo com o rótulo de `TIMELINE_KINDS`, datas "De"/"Até", botões "Filtrar", "Limpar filtros"), o botão "Atualizar", a tabela, "Carregar mais" e o detalhe aberto. Chave de cache `[ TIMELINE_KEY, subjectKey(subject), filters ]`.
  - `export function TimelineDetail(props: { target: DetailTarget; onClose(): void; endOnOutOfContext?: boolean; onOpeningRequired?(): void; searchDelayMs?: number }): JSX.Element`.
  - `export function DocumentDetail(props: { id: string; onClose(): void; onOpeningRequired?(): void; endOnOutOfContext?: boolean }): JSX.Element` — `<section aria-label="Documento">`, botões "Imprimir" (não na Notificação de papel) e "Fechar documento".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/timeline/PatientTimeline.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getTimeline: vi.fn() };
});
vi.mock("./TimelineDetail", () => ({
  TimelineDetail: (p: { target: { type: string; id: string }; onClose(): void }) => (
    <div>
      <span>{`detalhe ${p.target.type} ${p.target.id}`}</span>
      <button type="button" onClick={p.onClose}>fechar detalhe (dublê)</button>
    </div>
  )
}));

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { PatientTimeline } from "./PatientTimeline";
import { PARTIAL_NOTICE, TIMELINE_DISABLED, TIMELINE_EMPTY } from "../../lib/timeline";
import { renderWithProviders, sessionWith } from "../../test/campaignFixtures";
import { APPOINTMENT, CONSULTATION, DOCUMENT, SCREENING, entry, page } from "../../test/timelineFixtures";

const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = { kinds: [], from: "", to: "" };
afterEach(cleanup);

describe("PatientTimeline", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.getTimeline ]) m(fn).mockReset();
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "patient_timeline" ] }));
    m(api.getTimeline).mockResolvedValue(page());
  });

  it("lista na ordem do api, com tipo, onde e quem; abre o detalhe só do que tem tela", async () => {
    renderWithProviders(<PatientTimeline subject={{ attendanceId: "a1" }} />);
    expect(await screen.findByText("Linha do tempo de Joana Lima")).not.toBeNull();
    expect(api.getTimeline).toHaveBeenCalledWith({ attendanceId: "a1" }, ALL, null);

    const text = document.body.textContent ?? "";
    const order = [ "Retorno — Consulta de enfermagem", "Receita", "Consulta agendada", "Escuta inicial", "Triagem respiratória" ]
      .map((t) => text.indexOf(t));
    expect(order.every((p) => p >= 0)).toBe(true);
    expect([ ...order ].sort((a, b) => a - b)).toEqual(order);
    expect(screen.getAllByText("UBS Centro · Dra. Helena Prado").length).toBeGreaterThan(0);
    expect(screen.getByText("comum · digital")).not.toBeNull();
    expect(screen.getByText("finalizada")).not.toBeNull();
    expect(screen.getByText("confirmado")).not.toBeNull();
    expect(screen.queryByRole("note", { name: "histórico parcial" })).toBeNull();

    fireEvent.click(screen.getByRole("button", { name: "Abrir consulta de 10/09/2026, 14:30" }));
    expect(screen.getByText("detalhe consultation cs0")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Abrir agendamento de 01/10/2026, 09:00" }));
    expect(screen.getByText("detalhe appointment_request rq1")).not.toBeNull();
    expect(screen.queryByText("detalhe consultation cs0")).toBeNull();
    expect(screen.queryByRole("button", { name: /^Abrir triagem/ })).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "fechar detalhe (dublê)" }));
    expect(screen.queryByText(/^detalhe /)).toBeNull();
  });

  it("carregar mais: pede com o cursor e não repete entrada", async () => {
    m(api.getTimeline)
      .mockResolvedValueOnce(page({ entries: [ APPOINTMENT, DOCUMENT ], next_before: "2026-09-10T14:40:00-03:00|clinical_document|doc2" }))
      .mockResolvedValueOnce(page({ entries: [ DOCUMENT, CONSULTATION ], next_before: null }));
    renderWithProviders(<PatientTimeline subject={{ patientId: "pa1" }} />);
    fireEvent.click(await screen.findByRole("button", { name: "Carregar mais" }));
    expect(await screen.findByText("Consulta agendada")).not.toBeNull();
    expect(api.getTimeline).toHaveBeenLastCalledWith({ patientId: "pa1" }, ALL, "2026-09-10T14:40:00-03:00|clinical_document|doc2");
    expect(screen.getAllByText("Receita")).toHaveLength(1);
    expect(screen.queryByRole("button", { name: "Carregar mais" })).toBeNull();
  });

  it("filtros: tipos e período vão para a consulta e voltam à primeira página", async () => {
    m(api.getTimeline).mockResolvedValue(page({ next_before: "c1" }));
    renderWithProviders(<PatientTimeline subject={{ attendanceId: "a1" }} />);
    await screen.findByText("Linha do tempo de Joana Lima");
    fireEvent.click(screen.getByLabelText("Campanha"));
    fireEvent.click(screen.getByLabelText("Triagem"));
    fireEvent.change(screen.getByLabelText("De"), { target: { value: "2026-09-01" } });
    fireEvent.change(screen.getByLabelText("Até"), { target: { value: "2026-09-30" } });
    fireEvent.click(screen.getByRole("button", { name: "Filtrar" }));
    await waitFor(() => expect(api.getTimeline).toHaveBeenLastCalledWith({ attendanceId: "a1" }, {
      kinds: [ "attendance", "screening", "appointment", "consultation", "clinical_document", "controlled_notification" ],
      from: "2026-09-01", to: "2026-09-30"
    }, null));

    fireEvent.click(screen.getByRole("button", { name: "Limpar filtros" }));
    await waitFor(() => expect(api.getTimeline).toHaveBeenLastCalledWith({ attendanceId: "a1" }, ALL, null));
    expect((screen.getByLabelText("Campanha") as HTMLInputElement).checked).toBe(true);
  });

  it("período invertido ou nenhum tipo: diz o motivo e não consulta", async () => {
    renderWithProviders(<PatientTimeline subject={{ attendanceId: "a1" }} />);
    await screen.findByText("Linha do tempo de Joana Lima");
    fireEvent.change(screen.getByLabelText("De"), { target: { value: "2026-10-01" } });
    fireEvent.change(screen.getByLabelText("Até"), { target: { value: "2026-09-01" } });
    fireEvent.click(screen.getByRole("button", { name: "Filtrar" }));
    expect(screen.getByRole("alert").textContent).toBe("o início do período é depois do fim");
    expect(api.getTimeline).toHaveBeenCalledTimes(1);
  });

  it("histórico parcial (modo integrado): o aviso", async () => {
    m(api.getTimeline).mockResolvedValue(page({ person: { display_name: "Joana Lima", partial: true, patient_id: null } }));
    renderWithProviders(<PatientTimeline subject={{ attendanceId: "a1" }} />);
    expect((await screen.findByRole("note", { name: "histórico parcial" })).textContent).toBe(PARTIAL_NOTICE);
  });

  it("vazio: diz que não há registro, sem undefined/null", async () => {
    m(api.getTimeline).mockResolvedValue(page({ entries: [] }));
    renderWithProviders(<PatientTimeline subject={{ attendanceId: "a1" }} />);
    expect(await screen.findByText(TIMELINE_EMPTY)).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("entrada de tipo desconhecido: aparece crua e sem Abrir", async () => {
    m(api.getTimeline).mockResolvedValue(page({ entries: [
      entry({ id: "vaccination:v1", kind: "vaccination", title: "Vacina contra a gripe", summary: null, status: null, unit: null,
        professional: null, detail_ref: { type: "vaccination", id: "v1" } }),
      CONSULTATION
    ] }));
    renderWithProviders(<PatientTimeline subject={{ attendanceId: "a1" }} />);
    expect(await screen.findByText("Vacina contra a gripe")).not.toBeNull();
    expect(screen.getByText("vaccination")).not.toBeNull();
    expect(screen.queryByRole("button", { name: /^Abrir vaccination/ })).toBeNull();
    expect(screen.getByRole("button", { name: "Abrir consulta de 10/09/2026, 14:30" })).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("403 opening_required ao carregar mais encerra a abertura", async () => {
    const onOpeningRequired = vi.fn();
    m(api.getTimeline)
      .mockResolvedValueOnce(page({ entries: [ APPOINTMENT ], next_before: "c1" }))
      .mockRejectedValueOnce(new ApiError(403, { error: "opening_required" }, "403"));
    renderWithProviders(<PatientTimeline subject={{ openingId: "op1" }} endOnOutOfContext onOpeningRequired={onOpeningRequired} />);
    fireEvent.click(await screen.findByRole("button", { name: "Carregar mais" }));
    await waitFor(() => expect(onOpeningRequired).toHaveBeenCalled());
  });

  it("403 out_of_context em contexto (sem abertura): diz o caminho e não encerra nada", async () => {
    const onOpeningRequired = vi.fn();
    m(api.getTimeline).mockRejectedValue(new ApiError(403, { error: "out_of_context" }, "403"));
    renderWithProviders(<PatientTimeline subject={{ attendanceId: "a1" }} onOpeningRequired={onOpeningRequired} />);
    expect((await screen.findByRole("alert")).textContent)
      .toBe("esta pessoa não está em atendimento com você — para ler, use a abertura justificada em Prontuário");
    expect(onOpeningRequired).not.toHaveBeenCalled();
  });

  it("422 invalid_cursor: diz para atualizar; Atualizar relê", async () => {
    m(api.getTimeline)
      .mockResolvedValueOnce(page({ entries: [ APPOINTMENT ], next_before: "c1" }))
      .mockRejectedValueOnce(new ApiError(422, { error: "invalid_cursor" }, "422"))
      .mockResolvedValue(page({ entries: [ APPOINTMENT, SCREENING ] }));
    renderWithProviders(<PatientTimeline subject={{ attendanceId: "a1" }} />);
    fireEvent.click(await screen.findByRole("button", { name: "Carregar mais" }));
    expect((await screen.findByRole("alert")).textContent).toBe("a lista mudou desde a última página — use Atualizar");
    expect(screen.getByText("Retorno — Consulta de enfermagem")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Atualizar" }));
    expect(await screen.findByText("Escuta inicial")).not.toBeNull();
    expect(m(api.getTimeline).mock.calls[2]).toEqual([ { attendanceId: "a1" }, ALL, null ]);
  });

  it("interruptor desligado no api: a frase", async () => {
    m(api.getTimeline).mockRejectedValue(new ApiError(403, { error: "feature_disabled", feature: "patient_timeline" }, "403"));
    renderWithProviders(<PatientTimeline subject={{ attendanceId: "a1" }} />);
    expect((await screen.findByRole("alert")).textContent).toBe(TIMELINE_DISABLED);
  });
});
```

```tsx
// src/modules/timeline/TimelineDetail.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getConsultationOptions: vi.fn(), getClinicalDocument: vi.fn() };
});
vi.mock("../documents/printDocument", () => ({ printDocument: vi.fn(() => Promise.resolve(null)) }));
vi.mock("../consultation/ConsultationView", () => ({
  ConsultationLoader: (p: { id: string; endOnOutOfContext?: boolean; patientProblems: unknown[] }) =>
    <span>{`consulta ${p.id} · fim em out_of_context ${p.endOnOutOfContext ? "sim" : "não"} · problemas ${p.patientProblems.length}`}</span>
}));
vi.mock("../attendance/ScreeningDetail", () => ({
  ScreeningDetailLoader: (p: { id: string }) => <span>{`escuta ${p.id}`}</span>
}));
vi.mock("../attendance/RequestDetailPanel", () => ({
  RequestDetailPanel: (p: { requestId: string }) => <span>{`pedido ${p.requestId}`}</span>
}));

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { printDocument } from "../documents/printDocument";
import { TimelineDetail } from "./TimelineDetail";
import { renderWithProviders, sessionWith } from "../../test/campaignFixtures";
import { options } from "../../test/consultationFixtures";
import { prescriptionDoc } from "../../test/documentFixtures";

const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);

describe("TimelineDetail", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.getConsultationOptions, api.getClinicalDocument, printDocument ]) m(fn).mockReset();
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "patient_timeline" ] }));
    m(api.getConsultationOptions).mockResolvedValue(options());
    m(api.getClinicalDocument).mockResolvedValue(prescriptionDoc());
    m(printDocument).mockResolvedValue(null);
  });

  it("consulta, escuta e pedido abrem as telas existentes", async () => {
    const { rerender } = renderWithProviders(
      <TimelineDetail target={{ type: "consultation", id: "cs0" }} endOnOutOfContext onClose={vi.fn()} />);
    expect(screen.getByText("consulta cs0 · fim em out_of_context sim · problemas 0")).not.toBeNull();
    rerender(<TimelineDetail target={{ type: "screening", id: "sc1" }} onClose={vi.fn()} />);
    expect(screen.getByText("escuta sc1")).not.toBeNull();
    rerender(<TimelineDetail target={{ type: "appointment_request", id: "rq1" }} onClose={vi.fn()} />);
    expect(screen.getByText("pedido rq1")).not.toBeNull();
  });

  it("documento: tipo, situação, modo, código e Imprimir pela rotina do 19c", async () => {
    const onClose = vi.fn();
    renderWithProviders(<TimelineDetail target={{ type: "clinical_document", id: "doc2" }} onClose={onClose} />);
    expect(await screen.findByText("P3H8T-2LQ6R")).not.toBeNull();
    expect(api.getClinicalDocument).toHaveBeenCalledWith("doc2");
    const region = screen.getByRole("region", { name: "Documento" });
    expect(region.textContent).toContain("Receita");
    expect(region.textContent).toContain("emitido");
    expect(region.textContent).toContain("papel");
    expect(region.textContent).toContain("Dra. Helena Prado");
    fireEvent.click(screen.getByRole("button", { name: "Imprimir" }));
    await waitFor(() => expect(printDocument).toHaveBeenCalledWith("doc2", expect.any(Function)));
    fireEvent.click(screen.getByRole("button", { name: "Fechar documento" }));
    expect(onClose).toHaveBeenCalled();
  });

  it("Notificação de papel: sem Imprimir e sem código", async () => {
    m(api.getClinicalDocument).mockResolvedValue(prescriptionDoc({
      id: "doc9", kind: "controlled_notification_record", short_code: null, verification_url: null
    }));
    renderWithProviders(<TimelineDetail target={{ type: "clinical_document", id: "doc9" }} onClose={vi.fn()} />);
    expect(await screen.findByText("Notificação de receita (papel)")).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Imprimir" })).toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("documento lido depois do fim da abertura: encerra a leitura", async () => {
    const onOpeningRequired = vi.fn();
    m(api.getClinicalDocument).mockRejectedValue(new ApiError(403, { error: "opening_required" }, "403"));
    renderWithProviders(<TimelineDetail target={{ type: "clinical_document", id: "doc2" }} onClose={vi.fn()}
      onOpeningRequired={onOpeningRequired} endOnOutOfContext />);
    await waitFor(() => expect(onOpeningRequired).toHaveBeenCalled());
  });

  it("falha do Imprimir aparece como alerta", async () => {
    m(printDocument).mockResolvedValue("o navegador bloqueou a janela nova — permita janelas deste site para imprimir");
    renderWithProviders(<TimelineDetail target={{ type: "clinical_document", id: "doc2" }} onClose={vi.fn()} />);
    fireEvent.click(await screen.findByRole("button", { name: "Imprimir" }));
    expect((await screen.findByRole("alert")).textContent).toMatch(/bloqueou a janela nova/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod20 && npx vitest run src/modules/timeline`
Expected: FAIL — `./PatientTimeline` e `./TimelineDetail` não existem.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/timeline/PatientTimeline.tsx
// Linha do tempo da pessoa no painel da cidade (módulo 20, F-20.3; spec §4,
// §5 e §8; contrato §2–§3). Lista cronológica na ordem do api, filtros por
// tipo e período, "Carregar mais" pelo cursor e o detalhe nas telas que já
// existem. Toda leitura gera trilha no api; aqui nada fica guardado
// (gcTime: 0). Pela abertura justificada, opening_required (e out_of_context)
// encerra a leitura em quem chamou.
import { useEffect, useState, type CSSProperties } from "react";
import { useInfiniteQuery } from "@tanstack/react-query";
import { getTimeline, type TimelineEntry, type TimelineFilters, type TimelineSubject } from "../../lib/api";
import {
  ALL_FILTERS, EMPTY_DRAFT, PARTIAL_NOTICE, TIMELINE_EMPTY, TIMELINE_KEY, TIMELINE_KINDS, detailTarget, endsOpening, entryWhen,
  entryWhere, filterProblem, filtersFrom, mergePages, openLabel, statusLabel, subjectKey, timelineError, timelineKindLabel,
  toggleKind, type DetailTarget, type FilterDraft
} from "../../lib/timeline";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { TimelineDetail } from "./TimelineDetail";

interface Props {
  subject: TimelineSubject;
  endOnOutOfContext?: boolean;
  onOpeningRequired?(): void;
  searchDelayMs?: number;
}

export function PatientTimeline({ subject, endOnOutOfContext = false, onOpeningRequired, searchDelayMs }: Props) {
  const [ draft, setDraft ] = useState<FilterDraft>(EMPTY_DRAFT);
  const [ filters, setFilters ] = useState<TimelineFilters>(ALL_FILTERS);
  const [ problem, setProblem ] = useState<string | null>(null);
  const [ open, setOpen ] = useState<DetailTarget | null>(null);

  const query = useInfiniteQuery({
    queryKey: [ TIMELINE_KEY, subjectKey(subject), filters ],
    queryFn: ({ pageParam }) => getTimeline(subject, filters, pageParam),
    initialPageParam: null as string | null,
    getNextPageParam: (last) => last.next_before ?? undefined,
    gcTime: 0, staleTime: 0, retry: false
  });

  const ended = query.isError && endsOpening(query.error, endOnOutOfContext);
  useEffect(() => { if (ended) onOpeningRequired?.(); }, [ ended, onOpeningRequired ]);

  const pages = query.data?.pages ?? [];
  const person = pages[0]?.person ?? null;
  const entries = mergePages(pages);

  function apply() {
    const p = filterProblem(draft);
    setProblem(p);
    if (p) return;
    setOpen(null);
    setFilters(filtersFrom(draft));
  }

  function clear() {
    setDraft(EMPTY_DRAFT); setProblem(null); setOpen(null); setFilters(ALL_FILTERS);
  }

  return (
    <section aria-label="Linha do tempo" style={panel}>
      <div style={{ display: "flex", gap: 8, alignItems: "center" }}>
        <strong style={{ flex: 1 }}>{person ? `Linha do tempo de ${person.display_name}` : "Linha do tempo"}</strong>
        <button type="button" style={secondaryButtonStyle} onClick={() => void query.refetch()}>Atualizar</button>
      </div>
      {person?.partial && <p role="note" aria-label="histórico parcial" style={note}>{PARTIAL_NOTICE}</p>}

      <fieldset aria-label="filtros da linha do tempo" style={fieldset}>
        <div style={{ display: "flex", flexWrap: "wrap", gap: 12 }}>
          {TIMELINE_KINDS.map((k) => (
            <label key={k.key} style={check}>
              <input type="checkbox" checked={draft.kinds.includes(k.key)} onChange={() => setDraft(toggleKind(draft, k.key))} />
              {k.label}
            </label>
          ))}
        </div>
        <div style={{ display: "flex", flexWrap: "wrap", gap: 10, alignItems: "flex-end" }}>
          <label style={labelStyle}>
            De
            <input type="date" value={draft.from} style={inputStyle} onChange={(e) => setDraft({ ...draft, from: e.target.value })} />
          </label>
          <label style={labelStyle}>
            Até
            <input type="date" value={draft.to} style={inputStyle} onChange={(e) => setDraft({ ...draft, to: e.target.value })} />
          </label>
          <button type="button" style={secondaryButtonStyle} onClick={apply}>Filtrar</button>
          <button type="button" style={secondaryButtonStyle} onClick={clear}>Limpar filtros</button>
        </div>
        {problem && <p role="alert" style={alert}>{problem}</p>}
      </fieldset>

      {query.isPending && <p className="mono" style={loading}>carregando a linha do tempo…</p>}
      {query.isError && !ended && <p role="alert" style={alert}>{timelineError(query.error)}</p>}
      {query.data && (
        <DataTable<TimelineEntry>
          cols={[
            { label: "Quando", w: "1fr", render: (e) => entryWhen(e) },
            { label: "Tipo", w: "0.9fr", render: (e) => <Tag>{timelineKindLabel(e.kind)}</Tag> },
            {
              label: "Registro", w: "2.2fr", render: (e) => (
                <div style={{ display: "flex", flexDirection: "column", gap: 2 }}>
                  <span>{e.title}</span>
                  {e.summary && <span style={muted}>{e.summary}</span>}
                </div>
              )
            },
            { label: "Situação", w: "0.9fr", render: (e) => statusLabel(e.status) },
            { label: "Onde e quem", w: "1.6fr", render: (e) => entryWhere(e) },
            {
              label: "", w: "auto", align: "right", render: (e) => {
                const target = detailTarget(e);
                return target ? (
                  <button type="button" aria-label={openLabel(e)} style={secondaryButtonStyle} onClick={() => setOpen(target)}>
                    Abrir
                  </button>
                ) : null;
              }
            }
          ]}
          rows={entries}
          rowKey={(e) => e.id}
          empty={TIMELINE_EMPTY}
        />
      )}
      {query.hasNextPage && (
        <div>
          <button type="button" disabled={query.isFetchingNextPage}
            style={query.isFetchingNextPage ? disabledButtonStyle : secondaryButtonStyle}
            onClick={() => void query.fetchNextPage()}>
            Carregar mais
          </button>
        </div>
      )}
      {open && (
        <TimelineDetail key={`${open.type}:${open.id}`} target={open} onClose={() => setOpen(null)}
          endOnOutOfContext={endOnOutOfContext} onOpeningRequired={onOpeningRequired} searchDelayMs={searchDelayMs} />
      )}
    </section>
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 10 };
const fieldset: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, margin: 0, padding: 10, border: "1px solid var(--rule)", borderRadius: 8 };
const check: CSSProperties = { display: "flex", gap: 4, alignItems: "center", fontSize: 12.5 };
const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const note: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600, color: "var(--warn)" };
const muted: CSSProperties = { fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const loading: CSSProperties = { margin: 0, fontSize: 10.5, color: "var(--ink3)" };
```

```tsx
// src/modules/timeline/TimelineDetail.tsx
// O detalhe de uma entrada da linha do tempo, nas telas que já existem
// (módulo 20; spec §8): consulta (19a), documento (19c/19d), escuta (18) e
// pedido de agendamento (08/17). Cada uma lê pelo próprio caminho do api, que
// confere de novo quem lê e grava a trilha.
import { useQuery } from "@tanstack/react-query";
import { getConsultationOptions } from "../../lib/api";
import { OPTIONS_KEY } from "../../lib/consultation";
import type { DetailTarget } from "../../lib/timeline";
import { useAuth } from "../../lib/auth";
import { ConsultationLoader } from "../consultation/ConsultationView";
import { ScreeningDetailLoader } from "../attendance/ScreeningDetail";
import { RequestDetailPanel } from "../attendance/RequestDetailPanel";
import { DocumentDetail } from "./DocumentDetail";

interface Props {
  target: DetailTarget;
  onClose(): void;
  endOnOutOfContext?: boolean;
  onOpeningRequired?(): void;
  searchDelayMs?: number;
}

export function TimelineDetail({ target, onClose, endOnOutOfContext, onOpeningRequired, searchDelayMs }: Props) {
  const { user } = useAuth();
  const options = useQuery({
    queryKey: [ OPTIONS_KEY, user?.id ?? null ], queryFn: getConsultationOptions, staleTime: 5 * 60_000,
    enabled: target.type === "consultation"
  });

  switch (target.type) {
    case "consultation":
      return (
        <ConsultationLoader id={target.id} options={options.data ?? null} patientProblems={[]} searchDelayMs={searchDelayMs}
          endOnOutOfContext={endOnOutOfContext} onOpeningRequired={onOpeningRequired} onClose={onClose} />
      );
    case "clinical_document":
      return <DocumentDetail id={target.id} onClose={onClose} endOnOutOfContext={endOnOutOfContext} onOpeningRequired={onOpeningRequired} />;
    case "screening":
      return <ScreeningDetailLoader id={target.id} onClose={onClose} />;
    case "appointment_request":
      return <RequestDetailPanel requestId={target.id} onClose={onClose} />;
  }
}
```

```tsx
// src/modules/timeline/DocumentDetail.tsx
// Um documento clínico aberto pela linha do tempo (módulo 20): o resumo e o
// Imprimir do 19c (`printDocument`). A Notificação de papel (19d) não tem PDF.
// Sem texto do documento aqui: o conteúdo está no PDF.
import { useEffect, useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { getClinicalDocument } from "../../lib/api";
import { documentError, documentSituation, issueModeLabel, kindLabel } from "../../lib/clinicalDocuments";
import { endsOpening } from "../../lib/timeline";
import { fmtDateTime } from "../../lib/format";
import { KeyValue } from "../../components/KeyValue";
import { secondaryButtonStyle, disabledButtonStyle } from "../../components/formStyles";
import { printDocument } from "../documents/printDocument";

export const TIMELINE_DOCUMENT_KEY = "timelineDocument";

interface Props { id: string; onClose(): void; endOnOutOfContext?: boolean; onOpeningRequired?(): void }

export function DocumentDetail({ id, onClose, endOnOutOfContext = false, onOpeningRequired }: Props) {
  const query = useQuery({
    queryKey: [ TIMELINE_DOCUMENT_KEY, id ], queryFn: () => getClinicalDocument(id), gcTime: 0, staleTime: 0, retry: false
  });
  const [ printing, setPrinting ] = useState(false);
  const [ printError, setPrintError ] = useState<string | null>(null);
  const ended = query.isError && endsOpening(query.error, endOnOutOfContext);
  useEffect(() => { if (ended) onOpeningRequired?.(); }, [ ended, onOpeningRequired ]);

  async function print() {
    if (printing) return;
    setPrinting(true); setPrintError(null);
    setPrintError(await printDocument(id, (err) => documentError(err)));
    setPrinting(false);
  }

  const d = query.data;
  return (
    <section aria-label="Documento" style={panel}>
      <strong>Documento</strong>
      {query.isPending && <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando o documento…</p>}
      {query.isError && !ended && <p role="alert" style={alert}>{documentError(query.error)}</p>}
      {printError && <p role="alert" style={alert}>{printError}</p>}
      {d && (
        <div style={{ display: "flex", gap: 24, flexWrap: "wrap" }}>
          <KeyValue k="Documento" v={kindLabel(d.kind)} mono={false} />
          <KeyValue k="Situação" v={documentSituation(d)} mono={false} />
          <KeyValue k="Modo" v={issueModeLabel(d.issue_mode)} />
          <KeyValue k="Emitido em" v={fmtDateTime(d.issued_at)} />
          <KeyValue k="Profissional" v={d.author.name} mono={false} />
          <KeyValue k="Código" v={d.short_code ?? "—"} />
        </div>
      )}
      <div style={{ display: "flex", gap: 8 }}>
        {d && d.kind !== "controlled_notification_record" && (
          <button type="button" disabled={printing} style={printing ? disabledButtonStyle : secondaryButtonStyle} onClick={() => void print()}>
            Imprimir
          </button>
        )}
        <button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar documento</button>
      </div>
    </section>
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod20 && npx vitest run src/modules/timeline && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod20
/opt/homebrew/bin/git add src/modules/timeline/PatientTimeline.tsx src/modules/timeline/PatientTimeline.test.tsx \
  src/modules/timeline/TimelineDetail.tsx src/modules/timeline/TimelineDetail.test.tsx src/modules/timeline/DocumentDetail.tsx
/opt/homebrew/bin/git commit -m "feat: list the patient timeline with filters and open entries in existing screens

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: A aba no painel do paciente e o botão na fila (modo integrado)

**Files:**
- Modify: `src/modules/consultation/PatientPanel.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/attendance/UnitQueue.tsx`, `src/modules/Attendance.tsx`
- Test: `src/modules/consultation/PatientPanel.timeline.test.tsx`, `src/modules/attendance/UnitQueue.timeline.test.tsx`

**Interfaces:**
- Consumes: `PatientTimeline` (Task 3); `TimelineSubject` (Task 1); `hasFeature` (`features.ts`); `SegmentedControl` (`src/shell/SegmentedControl.tsx`).
- Produces:
  - `PatientPanel` ganha a prop opcional `timeline?: PatientTimelineProps` (`{ subject; endOnOutOfContext?; onOpeningRequired?; searchDelayMs? }`); com ela, um `SegmentedControl` com "Resumo" e "Linha do tempo"; a linha do tempo só monta na primeira abertura da aba e depois fica montada (escondida), como a aba Documentos do 19c. Sem a prop, nada muda.
  - `ConsultationWorkspace` passa `timeline={{ subject: { attendanceId } }}` com `patient_timeline` na sessão.
  - `UnitQueue` ganha a prop `patientTimeline?: boolean`; com ela, `canCare` e **sem** `clinicalRecord`, cada linha de "Em atendimento" tem o botão "Linha do tempo", que abre a `PatientTimeline` do atendimento com "Fechar linha do tempo". `Attendance` passa `patientTimeline={hasFeature(user, "patient_timeline")}`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/consultation/PatientPanel.timeline.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getTimeline: vi.fn(), getAttendanceRecord: vi.fn(), getConsultationOptions: vi.fn() };
});

import * as api from "../../lib/api";
import { PatientPanel } from "./PatientPanel";
import { ConsultationWorkspace } from "./ConsultationWorkspace";
import { renderWithProviders, sessionWith } from "../../test/campaignFixtures";
import { options, record } from "../../test/consultationFixtures";
import { page } from "../../test/timelineFixtures";

const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };

describe("PatientPanel — aba Linha do tempo (módulo 20)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.getTimeline, api.getAttendanceRecord, api.getConsultationOptions ]) m(fn).mockReset();
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], {
      id: "us1", features: [ "clinical_record", "patient_timeline" ]
    }));
    m(api.getTimeline).mockResolvedValue(page());
    m(api.getAttendanceRecord).mockResolvedValue(record());
    m(api.getConsultationOptions).mockResolvedValue(options());
  });

  it("sem a prop: nenhuma aba, nada muda", () => {
    renderWithProviders(<PatientPanel record={record()} onOpenConsultation={vi.fn()} />);
    expect(screen.queryByRole("tab", { name: "Linha do tempo" })).toBeNull();
    expect(screen.getByText("Problemas ativos")).not.toBeNull();
  });

  it("com a prop: lê só ao abrir a aba e mantém montada na volta ao Resumo", async () => {
    renderWithProviders(<PatientPanel record={record()} onOpenConsultation={vi.fn()} timeline={{ subject: { attendanceId: "a1" } }} />);
    expect(screen.getByRole("tab", { name: "Resumo" }).getAttribute("aria-selected")).toBe("true");
    expect(api.getTimeline).not.toHaveBeenCalled();

    fireEvent.click(screen.getByRole("tab", { name: "Linha do tempo" }));
    expect(await screen.findByText("Linha do tempo de Joana Lima")).not.toBeNull();
    expect(api.getTimeline).toHaveBeenCalledWith({ attendanceId: "a1" }, { kinds: [], from: "", to: "" }, null);
    // Escondido (hidden), não desmontado: *ByRole ignora o que está escondido.
    expect(screen.queryByRole("list", { name: "problemas ativos" })).toBeNull();

    fireEvent.click(screen.getByRole("tab", { name: "Resumo" }));
    expect(screen.getByRole("list", { name: "problemas ativos" })).not.toBeNull();
    expect(screen.queryByRole("region", { name: "Linha do tempo" })).toBeNull();
    fireEvent.click(screen.getByRole("tab", { name: "Linha do tempo" }));
    expect(api.getTimeline).toHaveBeenCalledTimes(1);
  });

  it("no atendimento chamado (modo record): a aba lê pelo atendimento", async () => {
    renderWithProviders(<ConsultationWorkspace attendanceId="a1" unit={unit} units={[ unit ]} onClose={vi.fn()} onFinalized={vi.fn()} />);
    fireEvent.click(await screen.findByRole("tab", { name: "Linha do tempo" }));
    expect(await screen.findByText("Linha do tempo de Joana Lima")).not.toBeNull();
    expect(api.getTimeline).toHaveBeenCalledWith({ attendanceId: "a1" }, { kinds: [], from: "", to: "" }, null);
  });

  it("no atendimento chamado sem patient_timeline: sem aba", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "clinical_record" ] }));
    renderWithProviders(<ConsultationWorkspace attendanceId="a1" unit={unit} units={[ unit ]} onClose={vi.fn()} onFinalized={vi.fn()} />);
    expect(await screen.findByText("Joana Lima")).not.toBeNull();
    expect(screen.queryByRole("tab", { name: "Linha do tempo" })).toBeNull();
  });
});
```

(`ConsultationWorkspace` com a sessão acima não tem `clinical_documents`: o `MedicationsBlock` e as abas Consulta/Documentos não montam. O único `role="tab"` vem do `PatientPanel`. A sessão chega pelo `AuthProvider` depois do primeiro render; os `findByRole` esperam por ela.)

```tsx
// src/modules/attendance/UnitQueue.timeline.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listUnitQueue: vi.fn(), getTimeline: vi.fn() };
});

import * as api from "../../lib/api";
import { AuthProvider } from "../../lib/auth";
import { UnitQueue } from "./UnitQueue";
import { PARTIAL_NOTICE } from "../../lib/timeline";
import { page } from "../../test/timelineFixtures";

afterEach(cleanup);
const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };
const inCare = [ {
  id: "a3", cpf_masked: "***.333.444-**", checked_in_at: "2026-09-25T08:00:00Z", protocol_name: "triage-dor", priority: 3,
  source: "triage" as const, appointment_time: null, called_at: "2026-09-25T12:00:00Z", called_by_name: "dr@cidade.gov.br"
} ];

function renderQueue(props: { canCare: boolean; clinicalRecord?: boolean; patientTimeline?: boolean }) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  render(<UnitQueue unit={unit} units={[ unit ]} {...props} />, { wrapper });
}

describe("UnitQueue — linha do tempo no modo integrado (módulo 20)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listUnitQueue, api.getTimeline ]) m(fn).mockReset();
    m(api.fetchCurrentSession).mockResolvedValue({ id: "u1", email_address: "dr@cidade.gov.br", operator: false, mfa_enrolled: true,
      mfa_verified_at: null, features: [ "patient_timeline" ],
      memberships: [ { city_slug: "m1", city_name: "Curitiba", city_uf: "PR", role: "health_professional" } ] });
    m(api.listUnitQueue).mockResolvedValue({ waiting: [], in_care: inCare });
    m(api.getTimeline).mockResolvedValue(page({ person: { display_name: "Joana Lima", partial: true, patient_id: null } }));
  });

  it("modo integrado: Linha do tempo no atendimento chamado, com o aviso de parcial", async () => {
    renderQueue({ canCare: true, patientTimeline: true });
    fireEvent.click(await screen.findByRole("button", { name: "Linha do tempo" }));
    expect(await screen.findByText("Linha do tempo de Joana Lima")).not.toBeNull();
    expect(api.getTimeline).toHaveBeenCalledWith({ attendanceId: "a3" }, { kinds: [], from: "", to: "" }, null);
    expect(screen.getByRole("note", { name: "histórico parcial" }).textContent).toBe(PARTIAL_NOTICE);
    fireEvent.click(screen.getByRole("button", { name: "Fechar linha do tempo" }));
    expect(screen.queryByText("Linha do tempo de Joana Lima")).toBeNull();
  });

  it("modo record (clinicalRecord): sem o botão — a linha do tempo está no painel do paciente", async () => {
    renderQueue({ canCare: true, patientTimeline: true, clinicalRecord: true });
    expect(await screen.findByRole("button", { name: "Consulta" })).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Linha do tempo" })).toBeNull();
  });

  it("sem o interruptor ou sem papel clínico: sem o botão", async () => {
    renderQueue({ canCare: true });
    expect(await screen.findByText("Em atendimento")).not.toBeNull();
    await screen.findByText("***.333.444-**");
    expect(screen.queryByRole("button", { name: "Linha do tempo" })).toBeNull();
    cleanup();
    renderQueue({ canCare: false, patientTimeline: true });
    await screen.findByText("***.333.444-**");
    expect(screen.queryByRole("button", { name: "Linha do tempo" })).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod20 && npx vitest run src/modules/consultation/PatientPanel.timeline.test.tsx src/modules/attendance/UnitQueue.timeline.test.tsx`
Expected: FAIL — `PatientPanel` não tem a prop `timeline` (sem abas); `UnitQueue` não tem o botão.

- [ ] **Step 3: Write minimal implementation**

Em `src/modules/consultation/PatientPanel.tsx`:

(a) troque o import de React e acrescente os novos, logo depois de `import { secondaryButtonStyle } from "../../components/formStyles";`:

```tsx
import { useState, type CSSProperties, type ReactNode } from "react";
```

(no lugar de `import type { CSSProperties, ReactNode } from "react";`) e

```tsx
import type { TimelineSubject } from "../../lib/api";
import { SegmentedControl } from "../../shell/SegmentedControl";
import { PatientTimeline } from "../timeline/PatientTimeline";

// Módulo 20: a aba "Linha do tempo" (quem passa a prop decide a pessoa).
export interface PatientTimelineProps {
  subject: TimelineSubject; endOnOutOfContext?: boolean; onOpeningRequired?(): void; searchDelayMs?: number;
}
```

(b) a assinatura passa a

```tsx
// Módulo 19c: `besideProblems` = os medicamentos em uso, logo depois dos problemas.
// Módulo 20: `timeline` liga as abas "Resumo" e "Linha do tempo".
export function PatientPanel({ record, onOpenConsultation, besideProblems, timeline }: {
  record: ClinicalRecord; onOpenConsultation(id: string): void; besideProblems?: ReactNode; timeline?: PatientTimelineProps;
}) {
  const { patient, problems, consultations } = record;
  const active = problems.filter((p) => p.status === "active");
  const rev = record.today_screening?.current_revision ?? null;
  // A linha do tempo só lê na primeira abertura da aba e depois fica montada
  // (escondida), para a volta ao Resumo não perder filtros nem páginas.
  const [ tab, setTab ] = useState<"summary" | "timeline">("summary");
  const [ timelineOpened, setTimelineOpened ] = useState(false);
  const onSummary = !timeline || tab === "summary";
```

(c) logo depois do `</div>` que fecha a linha de `KeyValue` (Nome, Idade, Sexo, CPF), abra as abas e envolva os blocos existentes — de `<div style={block}>` "Problemas ativos" até o `</div>` de "Últimas consultas", inclusive `{besideProblems}` — num contêiner do Resumo:

```tsx
      {timeline && (
        <SegmentedControl
          options={[ { key: "summary", label: "Resumo" }, { key: "timeline", label: "Linha do tempo" } ]}
          value={tab} onChange={(key) => { setTab(key); if (key === "timeline") setTimelineOpened(true); }} />
      )}

      {/* display: contents não mexe no gap do painel; só vale com o Resumo aberto. */}
      <div hidden={!onSummary} style={onSummary ? { display: "contents" } : undefined}>
        {/* … os blocos existentes: Problemas ativos, {besideProblems}, Escuta de hoje, Últimas consultas … */}
      </div>

      {timeline && timelineOpened && (
        <div hidden={onSummary} style={onSummary ? undefined : { display: "contents" }}>
          <PatientTimeline {...timeline} />
        </div>
      )}
```

(O comentário `{/* … os blocos existentes … */}` marca onde os quatro blocos **que já estão no arquivo** ficam, sem mudar uma linha deles: o diff é só o `<div hidden…>` em volta.)

Em `src/modules/consultation/ConsultationWorkspace.tsx`:

(a) nos imports, depois de `import { canUseDocuments } from "../../lib/clinicalDocuments";`:

```tsx
import { hasFeature } from "../../lib/features";
```

(b) depois de `const documentsOn = canUseDocuments(user);`:

```tsx
  // Módulo 20: a aba "Linha do tempo" no painel do paciente, pelo atendimento (em contexto).
  const timelineOn = hasFeature(user, "patient_timeline");
```

(c) na linha `<PatientPanel record={record.data} onOpenConsultation={setViewing} besideProblems={documentsOn ? (`, acrescente a prop antes de `besideProblems`:

```tsx
          <PatientPanel record={record.data} onOpenConsultation={setViewing}
            timeline={timelineOn ? { subject: { attendanceId: props.attendanceId }, searchDelayMs: props.searchDelayMs } : undefined}
            besideProblems={documentsOn ? (
```

Em `src/modules/attendance/UnitQueue.tsx`:

(a) import, depois de `import { DeclarationPanel } from "./DeclarationPanel";`:

```tsx
import { PatientTimeline } from "../timeline/PatientTimeline";
```

(b) em `Props`, depois de `professional?: boolean;`:

```tsx
  // Módulo 20: `patient_timeline` ligado na sessão. Sem o prontuário (modo
  // integrated), a linha do tempo abre pelo atendimento chamado.
  patientTimeline?: boolean;
```

(c) no tipo `ScreeningPanel`, troque a última linha `| { kind: "consultation"; attendanceId: string };` por

```tsx
  | { kind: "consultation"; attendanceId: string }
  // Módulo 20: a linha do tempo do atendimento chamado, quando não há prontuário.
  | { kind: "timeline"; attendanceId: string };
```

(d) na desestruturação dos props, `clinicalDocuments = false, professional = false` passa a `clinicalDocuments = false, professional = false, patientTimeline = false`, e logo depois de `const [ declaring, setDeclaring ] = useState<QueueRow | null>(null);`:

```tsx
  const timelineButton = canCare && patientTimeline && !clinicalRecord;
```

(e) a condição da coluna de ações de "Em atendimento", `...(canCare || clinicalDocuments ? [ {`, continua; dentro dela, logo antes do botão "Ver escuta" (`{canCare && r.screening?.id && (`):

```tsx
                          {timelineButton && (
                            <button type="button" style={secondaryButtonStyle}
                              onClick={() => setPanel({ kind: "timeline", attendanceId: r.id })}>
                              Linha do tempo
                            </button>
                          )}
```

(f) logo antes de `{clinicalDocuments && declaring && (`:

```tsx
        {timelineButton && panel?.kind === "timeline" && (
          <div style={{ display: "flex", flexDirection: "column", gap: 8, padding: 16, border: "1px solid var(--rule2)", borderRadius: 8 }}>
            <PatientTimeline key={panel.attendanceId} subject={{ attendanceId: panel.attendanceId }} />
            <div><button type="button" style={secondaryButtonStyle} onClick={() => setPanel(null)}>Fechar linha do tempo</button></div>
          </div>
        )}

```

Em `src/modules/Attendance.tsx`, depois da linha `clinicalRecord={hasFeature(user, "clinical_record")}`:

```tsx
          patientTimeline={hasFeature(user, "patient_timeline")}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod20 && npx vitest run src/modules/consultation src/modules/attendance src/modules/Attendance.test.tsx src/modules/clinicalRecord && npx tsc --noEmit`
Expected: PASS (os testes antigos do `PatientPanel`, do `ConsultationWorkspace`, do `JustifiedRecord` e da `UnitQueue` seguem verdes sem mudança: sem a prop nova não há abas nem botão) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod20
/opt/homebrew/bin/git add src/modules/consultation/PatientPanel.tsx src/modules/consultation/PatientPanel.timeline.test.tsx \
  src/modules/consultation/ConsultationWorkspace.tsx src/modules/attendance/UnitQueue.tsx \
  src/modules/attendance/UnitQueue.timeline.test.tsx src/modules/Attendance.tsx
/opt/homebrew/bin/git commit -m "feat: add the timeline tab to the patient panel and the called attendance

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Abertura justificada com a linha do tempo, nos dois modos

**Files:**
- Create: `src/modules/clinicalRecord/JustifiedTimeline.tsx`
- Modify: `src/modules/clinicalRecord/JustifiedRecord.tsx`, `src/modules/clinicalRecord/OpeningForm.tsx`, `src/modules/ClinicalRecord.tsx`, `src/lib/clinicalRecord.ts`, `src/shell/modules.ts`, `src/shell/modules.test.ts`, `README.md`
- Test: `src/modules/ClinicalRecord.timeline.test.tsx`, `src/shell/modules.test.ts`

**Interfaces:**
- Consumes: `PatientTimeline` (Task 3); `TIMELINE_KEY` (Task 2); `deadlineMs`, `countdownLabel`, `OPENING_ENDED`, `clinicalRecordError` (`clinicalRecord.ts`); `CONSULTATION_KEY` (`consultation.ts`).
- Produces:
  - `export function JustifiedTimeline({ opening, onEnd, searchDelayMs }: { opening: Opening; onEnd(): void; searchDelayMs?: number }): JSX.Element` — `Panel` "Linha do tempo (abertura justificada)" com a contagem, a `PatientTimeline` pela abertura (`endOnOutOfContext`) e "Encerrar leitura"; ao fim, `OPENING_ENDED`, "Nova abertura" e o cache de `TIMELINE_KEY`/`CONSULTATION_KEY` limpo.
  - `JustifiedRecord` passa `timeline={{ subject: { openingId }, endOnOutOfContext: true, onOpeningRequired }}` ao `PatientPanel` com `patient_timeline` e limpa `TIMELINE_KEY` no fim.
  - `OpeningForm` ganha `timelineOnly?: boolean` (textos "linha do tempo", botão "Abrir linha do tempo" e a frase própria para `patient_not_found`, que no modo integrado quer dizer "sem par validado"; plano do api, Task 4).
  - `ClinicalRecord`: com `clinical_record` **ou** `patient_timeline` (este só para `health_professional`); abertura com `patient_id` e prontuário → `JustifiedRecord`; senão → `JustifiedTimeline`. O relatório do admin continua só com `clinical_record`.
  - `clinicalRecordError(err, { timelineOnly: true })` com `patient_not_found` → "nenhum cadastro validado com este CPF — a linha do tempo só existe para quem validou o cadastro na unidade"; sem a opção, a frase do 19a.
  - `navGroupsFor`: "Prontuário" também para `health_professional` com só `patient_timeline`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/ClinicalRecord.timeline.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), openClinicalRecord: vi.fn(), getJustifiedRecord: vi.fn(),
    getConsultationOptions: vi.fn(), listOpenings: vi.fn(), getTimeline: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { ClinicalRecord } from "./ClinicalRecord";
import { OPENING_ENDED } from "../lib/clinicalRecord";
import { TIMELINE_KEY } from "../lib/timeline";
import { renderWithProviders, sessionWith } from "../test/campaignFixtures";
import { NOW19, opening, options, record } from "../test/consultationFixtures";
import { APPOINTMENT, page } from "../test/timelineFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const expiresAt = () => new Date(Date.parse(NOW19) + 30 * 60_000).toISOString();

async function openFor(cpf = "52998224725") {
  fireEvent.change(await screen.findByLabelText("CPF do paciente"), { target: { value: cpf } });
  fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "continuity_of_care" } });
  fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
}

describe("Prontuário — linha do tempo pela abertura justificada (módulo 20)", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date", "setInterval", "clearInterval" ] });
    vi.setSystemTime(new Date(NOW19));
    for (const fn of [ api.fetchCurrentSession, api.stepUpMfa, api.openClinicalRecord, api.getJustifiedRecord,
      api.getConsultationOptions, api.listOpenings, api.getTimeline ]) m(fn).mockReset();
    m(api.getConsultationOptions).mockResolvedValue(options());
    m(api.listOpenings).mockResolvedValue([]);
    m(api.getJustifiedRecord).mockResolvedValue(record({ access: "justified" }));
    m(api.getTimeline).mockResolvedValue(page({ person: { display_name: "Joana Lima", partial: true, patient_id: null } }));
  });

  it("modo integrado: a abertura justificada abre só a linha do tempo", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "patient_timeline" ] }));
    m(api.openClinicalRecord).mockResolvedValue(opening({ patient_id: null, expires_at: expiresAt() }));
    renderWithProviders(<ClinicalRecord />);
    await openFor();
    fireEvent.click(screen.getByRole("button", { name: "Abrir linha do tempo" }));
    expect(await screen.findByText("Linha do tempo de Joana Lima")).not.toBeNull();
    expect(api.getTimeline).toHaveBeenCalledWith({ openingId: "op1" }, { kinds: [], from: "", to: "" }, null);
    expect(api.getJustifiedRecord).not.toHaveBeenCalled();
    expect(screen.getByText("Abertura justificada · expira em 30:00")).not.toBeNull();
    expect(screen.getByRole("note", { name: "histórico parcial" })).not.toBeNull();
  });

  it("a abertura acaba: a linha do tempo sai da tela e do cache", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "patient_timeline" ] }));
    m(api.openClinicalRecord).mockResolvedValue(opening({ patient_id: null, expires_at: expiresAt() }));
    const { client } = renderWithProviders(<ClinicalRecord />);
    await openFor();
    fireEvent.click(screen.getByRole("button", { name: "Abrir linha do tempo" }));
    expect(await screen.findByText(APPOINTMENT.title)).not.toBeNull();
    act(() => { vi.advanceTimersByTime(30 * 60_000 + 1000); });
    expect(screen.getByText(OPENING_ENDED)).not.toBeNull();
    expect(screen.queryByText(APPOINTMENT.title)).toBeNull();
    expect(client.getQueryCache().findAll({ queryKey: [ TIMELINE_KEY ] })).toHaveLength(0);
    fireEvent.click(screen.getByRole("button", { name: "Nova abertura" }));
    expect(screen.getByLabelText("CPF do paciente")).not.toBeNull();
  });

  it("403 opening_required na linha do tempo encerra a abertura", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "patient_timeline" ] }));
    m(api.openClinicalRecord).mockResolvedValue(opening({ patient_id: null, expires_at: expiresAt() }));
    m(api.getTimeline).mockRejectedValue(new ApiError(403, { error: "opening_required" }, "403"));
    renderWithProviders(<ClinicalRecord />);
    await openFor();
    fireEvent.click(screen.getByRole("button", { name: "Abrir linha do tempo" }));
    expect(await screen.findByText(OPENING_ENDED)).not.toBeNull();
  });

  it("CPF sem par validado (patient_not_found no modo integrado): a frase da linha do tempo, sem abrir", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "patient_timeline" ] }));
    m(api.openClinicalRecord).mockRejectedValue(new ApiError(404, { error: "patient_not_found" }, "404"));
    renderWithProviders(<ClinicalRecord />);
    await openFor();
    fireEvent.click(screen.getByRole("button", { name: "Abrir linha do tempo" }));
    expect(await screen.findByText(
      "nenhum cadastro validado com este CPF — a linha do tempo só existe para quem validou o cadastro na unidade")).not.toBeNull();
    expect(api.getTimeline).not.toHaveBeenCalled();
  });

  it("modo record: o prontuário justificado ganha a aba, lida pela abertura", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], {
      id: "us1", features: [ "clinical_record", "patient_timeline" ]
    }));
    m(api.openClinicalRecord).mockResolvedValue(opening({ expires_at: expiresAt() }));
    renderWithProviders(<ClinicalRecord />);
    await openFor();
    fireEvent.click(screen.getByRole("button", { name: "Abrir prontuário" }));
    fireEvent.click(await screen.findByRole("tab", { name: "Linha do tempo" }));
    await waitFor(() => expect(api.getTimeline).toHaveBeenCalledWith({ openingId: "op1" }, { kinds: [], from: "", to: "" }, null));
    expect(api.getJustifiedRecord).toHaveBeenCalledWith("pa1");
  });

  it("sem nenhum dos dois interruptores, ou admin só com a linha do tempo: nada abre", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ]));
    renderWithProviders(<ClinicalRecord />);
    expect(await screen.findByText("o prontuário está desligado nesta cidade")).not.toBeNull();
    cleanup();
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "municipal_admin" ], { features: [ "patient_timeline" ] }));
    renderWithProviders(<ClinicalRecord />);
    expect(await screen.findByText("seu papel não permite abrir o prontuário")).not.toBeNull();
    expect(api.listOpenings).not.toHaveBeenCalled();
  });
});
```

Em `src/shell/modules.test.ts`, logo depois do teste "Prontuário no menu: só profissional ou admin, só com clinical_record, nunca operador":

```ts
    it("Prontuário no menu também com só patient_timeline, para o profissional (módulo 20)", () => {
      expect(ids(user([ "health_professional" ], [ "patient_timeline" ]))).toContain("clinical-record");
      expect(ids(user([ "municipal_admin" ], [ "patient_timeline" ]))).not.toContain("clinical-record");
      expect(ids(user([ "citizen_verifier" ], [ "patient_timeline" ]))).not.toContain("clinical-record");
      expect(ids(user([ "health_professional" ], [ "patient_timeline" ], true))).not.toContain("clinical-record");
      expect(ids(user([ "health_professional" ], [ "patient_timeline" ]))).not.toContain("my-consultations");
    });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod20 && npx vitest run src/modules/ClinicalRecord.timeline.test.tsx src/shell/modules.test.ts`
Expected: FAIL — com só `patient_timeline` o módulo diz "desligado"; não há "Abrir linha do tempo"; o menu não tem "Prontuário".

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/clinicalRecord/JustifiedTimeline.tsx
// Linha do tempo pela abertura justificada, sem o prontuário (módulo 20;
// spec §5 e §8): no modo integrated a cidade não tem prontuário no Rota
// Saúde, mas a pessoa tem linha do tempo. Mesmas regras da abertura do 19a:
// marcada, 30 minutos, e quando acaba (relógio ou 403 do api) os dados saem
// da tela e do cache.
import { useCallback, useEffect, useState, type CSSProperties } from "react";
import { useQueryClient } from "@tanstack/react-query";
import type { Opening } from "../../lib/api";
import { OPENING_ENDED, countdownLabel, deadlineMs } from "../../lib/clinicalRecord";
import { CONSULTATION_KEY } from "../../lib/consultation";
import { TIMELINE_KEY } from "../../lib/timeline";
import { Panel } from "../../components/Panel";
import { Tag } from "../../components/Tag";
import { buttonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { PatientTimeline } from "../timeline/PatientTimeline";

interface Props { opening: Opening; onEnd(): void; searchDelayMs?: number }

export function JustifiedTimeline({ opening, onEnd, searchDelayMs }: Props) {
  const queryClient = useQueryClient();
  const [ receivedAt ] = useState(() => Date.now());
  const [ nowMs, setNowMs ] = useState(receivedAt);
  const [ ended, setEnded ] = useState(false);

  const left = Math.max(0, deadlineMs(opening.expires_at, receivedAt) - nowMs);
  const over = ended || left <= 0;

  useEffect(() => {
    if (over) return;
    const id = setInterval(() => setNowMs(Date.now()), 1000);
    return () => clearInterval(id);
  }, [ over ]);

  const end = useCallback(() => setEnded(true), []);

  useEffect(() => {
    if (!over) return;
    queryClient.removeQueries({ queryKey: [ TIMELINE_KEY ] });
    queryClient.removeQueries({ queryKey: [ CONSULTATION_KEY ] });
  }, [ over, queryClient ]);

  if (over) {
    return (
      <Panel title="Linha do tempo (abertura justificada)">
        <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
          <p role="status" style={{ margin: 0, fontSize: 13, fontWeight: 600 }}>{OPENING_ENDED}</p>
          <div><button type="button" style={buttonStyle} onClick={onEnd}>Nova abertura</button></div>
        </div>
      </Panel>
    );
  }

  return (
    <Panel title="Linha do tempo (abertura justificada)" sub="leitura fora do atendimento · fica registrada"
      right={<Tag tone="warn">abertura justificada</Tag>}>
      <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
        <p style={countdown}>{`Abertura justificada · expira em ${countdownLabel(left)}`}</p>
        <PatientTimeline subject={{ openingId: opening.opening_id }} endOnOutOfContext onOpeningRequired={end} searchDelayMs={searchDelayMs} />
        <div><button type="button" style={secondaryButtonStyle} onClick={onEnd}>Encerrar leitura</button></div>
      </div>
    </Panel>
  );
}

const countdown: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600, color: "var(--warn)" };
```

Em `src/modules/clinicalRecord/JustifiedRecord.tsx`:

(a) imports, depois de `import { useAuth } from "../../lib/auth";`:

```tsx
import { hasFeature } from "../../lib/features";
import { TIMELINE_KEY } from "../../lib/timeline";
```

(b) depois de `queryClient.removeQueries({ queryKey: [ CONSULTATION_KEY ] });`:

```tsx
    queryClient.removeQueries({ queryKey: [ TIMELINE_KEY ] });
```

(c) a linha `<PatientPanel record={record.data} onOpenConsultation={setViewing} />` passa a

```tsx
            <PatientPanel record={record.data} onOpenConsultation={setViewing}
              timeline={hasFeature(user, "patient_timeline") ? {
                subject: { openingId: opening.opening_id }, endOnOutOfContext: true, onOpeningRequired: handleOpeningRequired, searchDelayMs
              } : undefined} />
```

Em `src/lib/clinicalRecord.ts`, `clinicalRecordError` ganha a opção e a frase do modo integrado (a assinatura antiga continua valendo):

```ts
export const NO_VERIFIED_PAIR =
  "nenhum cadastro validado com este CPF — a linha do tempo só existe para quem validou o cadastro na unidade";

export function clinicalRecordError(err: unknown, opts: { timelineOnly?: boolean } = {}): string {
  const code = err instanceof ApiError ? (err.body as { error?: string } | null)?.error : undefined;
  // Módulo 20: sem prontuário (modo integrated), patient_not_found quer dizer "sem par validado".
  if (code === "patient_not_found" && opts.timelineOnly) return NO_VERIFIED_PAIR;
  if (code === "patient_not_found") return "nenhum prontuário para este CPF — o prontuário nasce na primeira consulta de um cadastro validado";
  if (code === "invalid_reason") return `escolha o motivo; em 'Outro motivo', descreva com pelo menos ${OPENING_NOTE_MIN} caracteres`;
  return consultationError(err);
}
```

Em `src/modules/clinicalRecord/OpeningForm.tsx`:

(a) a tradução das recusas do `SensitiveAction`, `translateError={(err) => (OWN_REFUSALS.has(errorCode(err) ?? "") ? clinicalRecordError(err) : null)}`, passa a

```tsx
            translateError={(err) => (OWN_REFUSALS.has(errorCode(err) ?? "") ? clinicalRecordError(err, { timelineOnly }) : null)}
```

(b) a assinatura e os três textos que dizem "prontuário":

```tsx
// Módulo 20: `timelineOnly` (modo integrated, só patient_timeline) — a abertura lê a linha do tempo.
export function OpeningForm({ onOpened, onGoToSecurity, timelineOnly = false }: {
  onOpened(o: Opening): void; onGoToSecurity?(): void; timelineOnly?: boolean;
}) {
  const what = timelineOnly ? "a linha do tempo" : "o prontuário";
```

e, no JSX: o título do `Panel` vira ``title={`Abrir ${timelineOnly ? "linha do tempo" : "prontuário"} fora do atendimento`}``; o parágrafo começa com ``{`Use só quando precisar ler ${what} de alguém que não está em atendimento com você. A abertura fica registrada, com o motivo, e aparece no relatório da administração.`}``; e o `SensitiveAction` usa `confirmLabel={timelineOnly ? "Abrir linha do tempo" : "Abrir prontuário"}`.

Em `src/modules/ClinicalRecord.tsx`, o corpo do componente passa a:

```tsx
export function ClinicalRecord({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  const { user } = useAuth();
  const [ opening, setOpening ] = useState<Opening | null>(null);
  if (!user) return null;
  const roles = user.memberships.map((m) => m.role);
  const isProfessional = roles.includes("health_professional");
  const isAdmin = roles.includes("municipal_admin");
  // Módulo 20: no modo integrated não há prontuário, mas há linha do tempo
  // (só do profissional; o relatório das aberturas segue do clinical_record).
  const recordOn = hasFeature(user, "clinical_record");
  const timelineOn = hasFeature(user, "patient_timeline");

  if (!recordOn && !timelineOn) return <Frame><EmptyState title="o prontuário está desligado nesta cidade" /></Frame>;
  if (user.operator || (!isProfessional && !(isAdmin && recordOn))) {
    return <Frame><EmptyState title="seu papel não permite abrir o prontuário" /></Frame>;
  }

  return (
    <Frame>
      {isProfessional && (opening
        ? (recordOn && opening.patient_id !== null
          ? <JustifiedRecord key={opening.opening_id} opening={opening} onEnd={() => setOpening(null)} />
          : <JustifiedTimeline key={opening.opening_id} opening={opening} onEnd={() => setOpening(null)} />)
        : <OpeningForm onOpened={setOpening} onGoToSecurity={onGoToSecurity} timelineOnly={!recordOn} />)}
      {isAdmin && recordOn && <OpeningsReport />}
    </Frame>
  );
}
```

com o import `import { JustifiedTimeline } from "./clinicalRecord/JustifiedTimeline";` depois do de `JustifiedRecord`.

Em `src/shell/modules.ts`, troque

```ts
  const canClinicalRecord = !user?.operator && (isProfessional || isAdmin) && hasFeature(user, "clinical_record");
```

por

```ts
  // Módulo 20: com só `patient_timeline` (modo integrated), a abertura
  // justificada da linha do tempo, só do profissional.
  const canClinicalRecord = !user?.operator && (
    ((isProfessional || isAdmin) && hasFeature(user, "clinical_record")) ||
    (isProfessional && hasFeature(user, "patient_timeline"))
  );
```

No `README.md`, depois do parágrafo do módulo 19d (ou, se não houver, no fim da seção de módulos):

```markdown

**Linha do tempo do paciente (módulo 20).** Com `patient_timeline` ligado, o
painel do paciente no atendimento chamado (modo `record`) e na abertura
justificada ganha as abas **Resumo** e **Linha do tempo**: a lista cronológica
da pessoa (triagens, atendimentos, escutas, agendamentos, consultas,
documentos, Notificações e campanhas), com filtros por tipo e período,
"Carregar mais" e o detalhe nas telas existentes. No modo `integrated` não há
prontuário: o atendimento chamado tem o botão **Linha do tempo**, com o aviso de
histórico parcial, e **Prontuário** abre só a linha do tempo pela abertura
justificada. Toda leitura fica na trilha do api; nada vai para a URL além de
ids, tipos, datas e o cursor.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod20 && npx vitest run src/modules/ClinicalRecord.timeline.test.tsx src/modules/ClinicalRecord.test.tsx src/modules/clinicalRecord src/shell/modules.test.ts src/lib/clinicalRecord.test.ts && npx tsc --noEmit`
Expected: PASS (os testes antigos do `ClinicalRecord` seguem: com `clinical_record` o texto e o botão "Abrir prontuário" não mudam) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod20
/opt/homebrew/bin/git add src/modules/clinicalRecord/JustifiedTimeline.tsx src/modules/clinicalRecord/JustifiedRecord.tsx \
  src/modules/clinicalRecord/OpeningForm.tsx src/modules/ClinicalRecord.tsx src/modules/ClinicalRecord.timeline.test.tsx \
  src/lib/clinicalRecord.ts src/shell/modules.ts src/shell/modules.test.ts README.md
/opt/homebrew/bin/git commit -m "feat: open the patient timeline through the justified opening in both record modes

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–5 na branch `feat/mod-20-timeline`; o api do módulo 20 na **3040** com a semente do módulo 20.
- Produces: branch pronta para merge (depois do merge do api do módulo 20).

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd apps/dashboard/.claude/mod20 && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base anotada na Task 0 mais **7 arquivos de teste novos**. Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod20 status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod20 log --stat origin/main..HEAD | grep -c node_modules
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod20 diff origin/main --stat -- src/components src/shell/SegmentedControl.tsx
```

Expected: `status` vazio, contagem `0` e nenhum componente comum mudado.

- [ ] **Step 2: Nada vaza**

```bash
cd apps/dashboard/.claude/mod20
grep -rn "console\.\|localStorage\|sessionStorage" src/lib/timeline.ts src/modules/timeline src/modules/clinicalRecord/JustifiedTimeline.tsx
sed -n '/─── Linha do tempo do paciente (módulo 20/,/─── Rascunhos de consulta abertos/p' src/lib/api.ts | grep -n "cpf\|name\|display"
```

Expected: o primeiro sem saída; o segundo só com `display_name` e `name` **nos tipos de resposta** (nenhum `params.set` com CPF ou nome).

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§3–§5, §8, §9, §12 F-20.3), o ADR 0035 e o contrato (§1–§3). Pontos de atenção:
- sem `patient_timeline`, o painel do paciente, a fila, o Prontuário e o menu são exatamente os do 19d (testes antigos verdes sem mudança);
- a URL leva só ids, tipos, datas e o cursor; a abertura continua com o CPF no corpo;
- "Carregar mais" usa o cursor do api tal como veio, não repete entrada e some sem `next_before`; filtro novo recomeça da primeira página;
- pela abertura, `opening_required`/`out_of_context` tira a lista da tela e do cache; em contexto, `out_of_context` só explica o caminho;
- cada entrada abre na tela existente (consulta, documento, escuta, pedido), e só essas; tipo desconhecido aparece cru e sem "Abrir";
- histórico parcial com o aviso; leituras com `gcTime: 0`;
- interface sem redesign: só componentes existentes; nenhum componente comum mudou.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do módulo 20 na porta **3040** (plano do api do módulo 20: servidor do worktree no container, semente com Curitiba em `record_mode = record` e uma cidade em `integrated` — a que o plano do api fixar na semente —, `patient_timeline` ligado nas duas, a pessoa da semente com triagem, escuta, agendamento, consulta, documentos e campanha). O Vite do worktree sobe num container da rede do compose:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose run -d --rm --no-deps --name dashboard-mod20 -p 5190:5190 \
  -w /app/.claude/mod20 -e VITE_API_PROXY_TARGET=http://api:3040 \
  dashboard npx vite --port 5190 --host 0.0.0.0
docker logs -f dashboard-mod20   # espere o "ready"; Ctrl+C sai do log, o container segue
```

(Se o plano do api subir o servidor do módulo 20 num container próprio, use o nome dele no lugar de `api`.)

Abra `http://curitiba.localhost:5190/dashboard/`. O usuário faz o login; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Confira com screenshot:
- **modo record (Curitiba)**, como a médica: Atendimento → chamar a pessoa da semente → painel do paciente com **Resumo** e **Linha do tempo**; a lista em ordem decrescente com tipo, unidade e profissional, sem o aviso de parcial; desmarcar "Campanha" e "Triagem" e filtrar por setembro → só o resto, do mês; "Carregar mais" (se a semente tiver mais de 50 entradas, ou com `limit` menor pelo api) sem repetir; **Abrir** a consulta (a leitura do 19a), um documento (resumo e **Imprimir** → o PDF), a escuta e o agendamento (o detalhe do pedido); a triagem sem "Abrir";
- **Prontuário** → abertura justificada da mesma pessoa → o prontuário com a aba **Linha do tempo** e a contagem; **Encerrar leitura**;
- **modo integrated**, como o profissional da outra cidade (`http://<slug>.localhost:5190/dashboard/`): na fila, **Em atendimento** → **Linha do tempo** (sem "Consulta"), com "Histórico parcial: …"; **Prontuário** → "Abrir linha do tempo fora do atendimento" → CPF, motivo, step-up → só a linha do tempo, com a contagem; um CPF só declarado → "nenhum cadastro validado com este CPF…";
- o relatório das aberturas (como `admin@curitiba.demo`) lista as aberturas da prova;
- no maintenance, desligar `patient_timeline` em Curitiba e recarregar: sem abas, sem botão, e o Prontuário como no 19d.

Depois: `docker rm -f dashboard-mod20`. Religue o que a prova desligou.

- [ ] **Step 5:** **Pare.** O merge do dashboard só vem depois do merge do api do módulo 20, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: api → dashboard e wpda → maintenance (contrato §8). Antes do push, confira `origin/main..main` no dashboard e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-20-timeline
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove .claude/mod20
/opt/homebrew/bin/git branch -d feat/mod-20-timeline
```

---

## Divergências propostas ao contrato

1. **D1 — Abertura justificada no modo integrated.** O contrato §3 aceita `opening_id`, mas não diz que a abertura do 19a (`POST /clinical_record/openings`, que exige `clinical_record` utilizável e um paciente) muda. Proposta — a mesma do plano do api (Desvio 6 / Divergência D5 dele): com `patient_timeline` ligado, a abertura vale sem prontuário, pelo par validado do CPF (`citizen_id`), e responde `{ opening_id, patient_id: null, expires_at }`; CPF sem par validado segue 404 `patient_not_found`. O dashboard trata `patient_id: null` (Task 1), escolhe a tela pelo par interruptor/paciente e dá a `patient_not_found` a frase "sem cadastro validado" quando só há linha do tempo (Task 5).
2. **D2 — `detail_ref.type` do agendamento.** O contrato não lista os valores; o plano do api usa `triage`, `attendance`, `screening`, `appointment` (id do **horário**), `consultation`, `clinical_document` (inclusive a Notificação de papel) e `campaign`. Não existe tela no dashboard que leia um horário pelo id; a que existe, `RequestDetailPanel`, lê o **pedido**. Proposta: o agendamento leva `{ type: "appointment_request", id: <id do pedido> }` (o horário sabe o seu pedido). Enquanto isso não muda, o dashboard trata `appointment` como sem detalhe (o código já faz isso: só abre `consultation`, `clinical_document`, `screening` e `appointment_request`). Além disso, cada leitura de detalhe pela abertura justificada (`getClinicalDocument`, `getScreening`, `getRequest`) precisa aceitar a abertura da pessoa como a linha do tempo aceita (hoje só a consulta aceita); sem isso, abrir um documento pela abertura mostra a recusa do api.
3. **D3 — Quem traduz.** O contrato não diz. Fica como no plano do api: `title` e `summary` em português, prontos; `status` é o **código** da origem (`completed`, `in_care`, `confirmed`, `finalized`, `issued`, `read`/`unread`…). O dashboard traduz os códigos conhecidos (`statusLabel`, Task 2) e mostra cru o desconhecido. Proposta: o contrato lista esses códigos por `kind`, para o dashboard, o wpda e a página do PEC não divergirem.
4. **D4 — Parâmetro `cpf`.** A spec §6 tem `cpf=` na rota do profissional; o contrato §3 trocou por `opening_id`. Este plano segue o contrato: CPF só no corpo do POST da abertura, nunca na URL da linha do tempo.

## Self-review

- **Cobertura (spec §8 "Dashboard", §3–§5, §9; contrato §1–§3; F-20.3):** aba "Linha do tempo" no painel do paciente — Task 4 (`PatientPanel`, `ConsultationWorkspace`) e Task 5 (`JustifiedRecord`); lista cronológica, filtros por tipo e período, "Carregar mais" — Tasks 2 e 3; detalhe nas telas existentes — Task 3 (`TimelineDetail`, `DocumentDetail`); abertura justificada fora de contexto reaproveitando o 19a — Task 5 (nos dois modos); aviso de histórico parcial — Tasks 3 e 4; modo integrated sem prontuário — Tasks 4 e 5; Vite 5190 com proxy para `http://api:3040` — Task 6.
- **Placeholders:** nenhum código faltando. No `PatientPanel` (Task 4), o comentário `{/* … os blocos existentes … */}` marca onde ficam os quatro blocos que já estão no arquivo, sem mudança — o diff é só o contêiner. As variáveis de ambiente (cidade integrada da semente, nome do container do api) são do plano do api.
- **Tipos:** `TimelineSubject`, `TimelineFilters`, `TimelinePage`, `TimelineEntry` (Task 1) são os usados nas Tasks 2–5; `getTimeline(subject, filters, before)` tem a mesma assinatura nos testes e no componente; `DetailTarget`/`DetailType` (Task 2) é o que `TimelineDetail` recebe; `PatientTimelineProps` (Task 4) espelha as props de `PatientTimeline` (Task 3); `Opening.patient_id: string | null` (Task 1) é o que `ClinicalRecord` testa (Task 5); `timelineKindLabel` não colide com o `kindLabel` dos documentos (usado no `DocumentDetail`).
- **Review Focus:** 1 — Task 2 ("mergePages…") e Task 3 ("carregar mais…"); 2 — Task 3 ("403 opening_required ao carregar mais…") e Task 5 ("a abertura acaba…"); 3 — Task 3 ("filtros…", "período invertido…", "422 invalid_cursor…"); 4 — Task 3 ("entrada de tipo desconhecido…"); 5 — Task 4 ("modo integrado: Linha do tempo no atendimento chamado…") e Task 5 ("modo integrado: a abertura justificada abre só a linha do tempo").
