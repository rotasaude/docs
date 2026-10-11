# Módulo 20 — Linha do tempo do paciente (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 20** rodando na branch dele (`feat/mod-20-timeline`, porta **3040**) com as rotas do cidadão (`/citizen/history?citizen_id=`, `/citizen/health_summary?citizen_id=`, `/citizen/documents/:id/pdf`; a pessoa vai em `citizen_id`, como no plano do api, Desvio 8 / Divergência D2 dele) e `features` na sessão do cidadão (Divergência W2 deste plano). Ele é **mergeado antes** deste (contrato §8: api → dashboard e wpda → maintenance). As Tasks 0–4 não dependem dele (os testes usam dublês); a prova da Task 5 depende.

**Goal:** Lado wpda de F-20.4 e F-20.5: a área **"Meu histórico"** de cada pessoa do celular. Par **validado**: resumo de saúde (problemas ativos, medicamentos em uso, exames pedidos nos últimos 12 meses, última pressão, peso e glicemia), documentos com **download do PDF** e **código de conferência**, e a linha do tempo. Par **só declarado**: as próprias triagens e agendamentos, com o convite para validar o cadastro na unidade. Nos dois, o aviso de **histórico parcial** quando a cidade não está no modo `record`.

**Architecture:** O cliente (`src/lib/citizenApi.ts`) ganha três chamadas por pessoa (`history`, `healthSummary`, `documentPdf`), com a normalização na borda como o resto do arquivo, e a sessão passa a guardar os interruptores ligados (`src/lib/sessionFeatures.ts`, instalado junto com o fuso, como `setCityTimeZone`). As regras de tela ficam em `src/lib/history.ts` (puras: rótulos dos tipos para o cidadão, junção das páginas sem repetir, linhas dos sinais vitais, nome seguro do arquivo, frases). A tela nova `src/modules/citizen/MyHistoryStep.tsx` segue as peças do canal (`Screen`, `BigButton`, `ErrorText`, alvos de 48 px, texto de 18 px). O `Flow` ganha o estado `record`; a tela **Minhas triagens** (`HistoryStep`) ganha o botão **"Meu histórico"** só com `patient_timeline` ligado. Quem vê o quê é do api (par validado, a pessoa, o que é parcial); a tela mostra o que ele devolveu e nunca pede ao api o que ele recusaria.

**Tech Stack:** React 18 + TypeScript, Vitest 2 + Testing Library 16 + user-event 14 + jest-dom (jsdom, `setupFiles` com `cleanup`), `vi.spyOn(citizenApi, …)`, Vite 5 (base `/wpda/`, proxy de dev).

**Spec:** `docs/superpowers/specs/2026-10-11-module-20-patient-timeline-design.md` (§2 decisões 2 e 3, §3 "a pessoa", §4 coluna "Cidadão vê" e "Resumo clínico do cidadão", §6, §8 "wpda", §9, §12 F-20.4 e F-20.5), `docs/adr/0035.md` (invariantes do cidadão) e o contrato `docs/superpowers/plans/2026-10-11-module-20-timeline-contracts.md` (§2 `<citizen_entry>`, §4; as diferenças estão em "Divergências propostas ao contrato"). ADR 0017 (par CPF + celular, validação presencial).

## Global Constraints

- Rotas (contrato §4, na forma das Divergências W1–W3; sessão do cidadão, cookie `citizen_session`, mesma origem):
  - `GET /citizen/history?citizen_id=&before=&limit=20&kinds[]=` → `{ verified, partial, entries: [<citizen_entry>], next_before|null }`; par só declarado: `verified: false` e só `triage` e `appointment` do próprio par; `citizen_id` que não é do celular da sessão → 404 `not_found`;
  - `GET /citizen/health_summary?citizen_id=` → `{ problems, medications, exam_requests, vitals }`; 403 `not_verified`; 404 `not_available` (sem paciente ou sem `clinical_record`);
  - `GET /citizen/documents/:id/pdf` → `application/pdf`; 404 `not_found` (de outra pessoa, cancelado ou inexistente, sem distinguir); 403 `not_verified`;
  - `GET /citizen/session` → `{ phone_masked, time_zone?, features?: string[] }` (W2).
  - 403 `feature_disabled` (`feature: "patient_timeline"`) em qualquer uma; 401 dispara `citizen:unauthenticated`, como hoje.
- `<citizen_entry>` (contrato §2): `{ id, kind, occurred_at, unit: { name }|null, professional_name|null, title, summary|null, status|null, document?: { id, short_code, verification_url, downloadable } }`. `title` e `summary` vêm prontos do api, em português e já sem nada clínico livre; `status` é o código da origem (`confirmed`, `issued`, `read`…, como no plano do api), traduzido por `citizenStatusLabel`. O cidadão não vê código de máquina: tipo desconhecido aparece como "Registro" e situação desconhecida não aparece.
- O cidadão nunca vê texto clínico livre, CID sem nome, número SNCR nem dado de outra pessoa (ADR 0035): a tela só mostra os campos acima e os do resumo; nada de CPF na tela nova além do que já existe em "Minhas triagens".
- Na URL vão só o id da pessoa, o id do documento, os tipos, o cursor e o `limit` — nunca CPF, celular ou texto. O PDF é lido com `fetch` e salvo por um endereço `blob:` com nome `documento-<código>.pdf` (sem dado pessoal); nada em `localStorage`, `sessionStorage` ou `console`.
- Interruptor (spec §3): o botão "Meu histórico" só aparece com `patient_timeline` em `session.features`; sem a chave (api antigo ou desligado), "Minhas triagens" é exatamente a de hoje. Se o interruptor cair com a tela aberta, o 403 `feature_disabled` vira "O histórico ainda não está disponível nesta cidade.".
- Interface do canal (ui.tsx): alvos de toque de 48 px ou mais, texto de 18 px, ação principal embaixo; componentes existentes (`Screen`, `BigButton`, `ErrorText`), sem redesign. Nenhum componente comum muda além das frases novas em `MESSAGES`.
- Testes que dependem de "agora" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` e `vi.useRealTimers()` no `afterEach`; a suíte roda em `TZ=America/Sao_Paulo`.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.

## Ambiente de execução

- Worktree `apps/wpda/.claude/mod20`, branch `feat/mod-20-timeline` a partir de `origin/main` (Task 0; `/.claude/` está no `.gitignore` do wpda). Caminhos de arquivo relativos a ele; comandos a partir da raiz do monorepo (`/Users/eduardovrocha/Development/ioit.solutions/rota-saude`).
- Testes: `cd apps/wpda/.claude/mod20 && npx vitest run <arquivos>`. Tipos: `npx tsc --noEmit`.
- Ambiente de teste: `vitest.config.ts` com `environment: "jsdom"`, `globals: false`, `setupFiles: ["./src/test/setup.ts"]` (jest-dom e `cleanup` automático) e `TZ=America/Sao_Paulo`. `fmtDateTime("2026-09-10T14:30:00-03:00")` → `"10/09/2026, 14:30"`; `fmtCalendarDate("2019-03-01")` → `"01/03/2019"`.
- O api do módulo 20 roda em dev na porta **3040**. O Vite do worktree sobe **num container da rede do compose**, na **5191**, com o proxy para `http://api:3040` (Task 5). `/citizen` já está no proxy do `vite.config.ts`: nada muda nele.

## Review Focus

1. **Par só declarado chega ao resumo ou aos documentos.** A tela não pode nem pedir o resumo nem a lista de documentos quando `verified: false` (o api recusaria, mas o pedido já é um sinal errado), e mostra o convite para validar com o caminho até o código do posto. Testes: Task 3, "par declarado: triagens e agendamentos, convite para validar e nada de resumo ou documentos".
2. **Baixar o PDF dá errado.** Documento cancelado entre a lista e o toque (404), sessão vencida (401), interruptor desligado: a tela diz o que houve em português, sem abrir janela vazia, e o nome do arquivo nunca leva CPF ou nome. Testes: Task 1, "documentPdf: blob, recusa com código e 401"; Task 2, "documentFilename: só código ou id"; Task 3, "baixar o PDF: arquivo com o código" e "documento que sumiu: diz e não baixa".
3. **"Carregar mais" repete ou some com entrada.** Página seguinte que repete uma entrada (mesmo instante, documento cancelado no meio) não pode duplicar; sem `next_before` o botão some; o cursor vai como veio. Testes: Task 2, "mergeEntries: junta sem repetir, na ordem"; Task 3, "carregar mais: cursor como veio e sem repetir".
4. **Cidade no modo integrado ou sem prontuário.** O resumo não existe (404 `not_available`): a tela não mostra erro, só não mostra o resumo; o aviso de histórico parcial aparece. Testes: Task 3, "resumo indisponível (404): sem a seção e sem erro" e "histórico parcial: o aviso".
5. **Api antigo, ou interruptor desligado, ou tipo novo.** Sem `features` na sessão o botão não aparece e "Minhas triagens" fica igual; 403 `feature_disabled` vira frase; `kind` desconhecido aparece como "Registro" e campos ausentes não viram `undefined`. Testes: Task 1, "sessão: interruptores instalados; ausentes = nenhum"; Task 3, "interruptor desligado no meio: a frase" e "tipo desconhecido vira Registro"; Task 4, "sem patient_timeline na sessão: Minhas triagens como hoje".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | conferência do main e do api do módulo 20, worktree, base | 0 |
| `src/lib/citizenApi.ts`, `src/lib/sessionFeatures.ts`, `src/lib/citizenApi.history.test.ts` | tipos do contrato; `history`, `healthSummary`, `documentPdf`; interruptores da sessão | 1 |
| `src/lib/history.ts`, `src/lib/history.test.ts`, `src/modules/citizen/ui.tsx` | regras de tela de "Meu histórico" (rótulos, situação, junção, sinais, nome do PDF); frases novas das recusas | 2 |
| `src/modules/citizen/MyHistoryStep.tsx`, `src/modules/citizen/MyHistoryStep.test.tsx` | a tela "Meu histórico" | 3 |
| `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/Flow.history.test.tsx`, `README.md` | o botão em "Minhas triagens" e o estado `record` no `Flow` | 4 |
| — | suíte, build, revisão e prova no navegador | 5 |

**Estratégia de teste:** cliente com `fetch` falso conferindo URL (por pessoa, `kinds[]`, cursor, `limit`, nada pessoal), normalização, PDF binário e o 401 (Task 1); regras puras com tabela de casos (Task 2); a tela com `vi.spyOn(citizenApi, …)` cobrindo par validado, par declarado, resumo indisponível, parcial, "Carregar mais", download (com `URL.createObjectURL` falso) e cada recusa que muda a tela (Task 3); a navegação real pelo `Flow` com a sessão com e sem o interruptor (Task 4). A prova final é no navegador, contra o api do módulo 20 (Task 5).

---

### Task 0: Conferência do main e do api do módulo 20, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do wpda (`20e5e85` ou mais novo).
- Produces: worktree `apps/wpda/.claude/mod20` na branch `feat/mod-20-timeline`; a base anotada.

- [ ] **Step 1: Confira as peças que este plano altera**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/wpda fetch origin
/opt/homebrew/bin/git -C apps/wpda log --oneline -3 origin/main
/opt/homebrew/bin/git -C apps/wpda show origin/main:src/lib/citizenApi.ts | grep -n \
  "export interface CitizenSession\|function withCityTimeZone\|signOut: () => call<void>(\"DELETE\", \"/session\"),\|updateContactPreference: (citizenId"
/opt/homebrew/bin/git -C apps/wpda show origin/main:src/modules/citizen/HistoryStep.tsx | grep -n \
  "export function HistoryStep\|onAppointmentCheckIn: (appointmentId: string) => void }) {\|<AppointmentsSection citizenId"
/opt/homebrew/bin/git -C apps/wpda show origin/main:src/modules/citizen/Flow.tsx | grep -n \
  '| { at: "history"; consentVersion: string | null; citizenId: string }\|case "verify-code":\|view = <HistoryStep citizenId={state.citizenId}'
/opt/homebrew/bin/git -C apps/wpda show origin/main:src/modules/citizen/ui.tsx | grep -n 'note_too_long: "Escreva no máximo 200 caracteres."'
```

Expected: cada `grep` acha as linhas pedidas (as âncoras dos diffs). Se alguma mudou, anote: as tasks aplicam a mesma intenção sobre o texto real.

- [ ] **Step 2: Confira o api do módulo 20 no ar**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/api branch --list feat/mod-20-timeline
docker compose exec -T -w /rails/.claude/mod20 api bin/rails routes 2>/dev/null | grep -E "citizen.*(history|health_summary|documents)" | head
```

Expected: a branch e as três rotas do cidadão (`GET /citizen/history`, `GET /citizen/health_summary`, `GET /citizen/documents/:id/pdf`). Confira no controller do api (`app/controllers/citizen_api/history_controller.rb`) que a pessoa vem em `citizen_id`, que `kinds[]` é aceito e que a sessão traz `features`; se alguma das três não estiver lá, as Tasks 1–4 seguem (os testes usam dublês), mas **reporte** antes da Task 5: as Divergências W1–W3 precisam ser decididas para a prova.

- [ ] **Step 3: Crie o worktree e ligue o `node_modules`**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/wpda worktree add .claude/mod20 -b feat/mod-20-timeline origin/main
ln -s ../../node_modules apps/wpda/.claude/mod20/node_modules
cd apps/wpda/.claude/mod20 && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: tudo verde e `tsc-ok`. **Anote** a base (arquivos e testes). A Task 5 espera a base mais **4 arquivos de teste novos** (`citizenApi.history`, `history`, `MyHistoryStep`, `Flow.history`).

---

### Task 1: Cliente do histórico, PDF e interruptores da sessão

**Files:**
- Modify: `src/lib/citizenApi.ts`
- Create: `src/lib/sessionFeatures.ts`
- Test: `src/lib/citizenApi.history.test.ts`

**Interfaces:**
- Consumes: `call`, `ApiError`, `withCityTimeZone` (existentes em `citizenApi.ts`).
- Produces:
  ```ts
  // src/lib/sessionFeatures.ts
  export function setSessionFeatures(raw: unknown): void;
  export function hasSessionFeature(key: string): boolean;
  // src/lib/citizenApi.ts
  export interface CitizenDocumentRef { id: string; short_code: string | null; verification_url: string | null; downloadable: boolean }
  export interface CitizenEntry {
    id: string; kind: string; occurred_at: string; unit: { name: string } | null; professional_name: string | null;
    title: string; summary: string | null; status: string | null; document: CitizenDocumentRef | null;
  }
  export interface CitizenHistory { verified: boolean; partial: boolean; entries: CitizenEntry[]; next_before: string | null }
  export interface SummaryProblem { terminology: string; code: string; label: string; since: string | null }
  export interface SummaryMedication { label: string; dosage_summary: string | null; since: string | null }
  export interface SummaryExamRequest { sigtap_code: string; label: string; requested_at: string }
  export interface HealthVitals {
    blood_pressure?: { systolic: number; diastolic: number; measured_at: string };
    weight_kg?: { value: number; measured_at: string };
    capillary_glucose?: { value: number; moment: string | null; measured_at: string };
  }
  export interface HealthSummary { problems: SummaryProblem[]; medications: SummaryMedication[]; exam_requests: SummaryExamRequest[]; vitals: HealthVitals }
  export const HISTORY_PAGE_SIZE = 20;
  export const DOCUMENT_KINDS = [ "clinical_document" ];
  // CitizenSession ganha features?: string[]
  citizenApi.history(citizenId: string, opts?: { before?: string | null; kinds?: string[] }): Promise<CitizenHistory>;
  citizenApi.healthSummary(citizenId: string): Promise<HealthSummary>;
  citizenApi.documentPdf(documentId: string): Promise<Blob>;
  ```

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/citizenApi.history.test.ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { citizenApi, ApiError, HISTORY_PAGE_SIZE } from "./citizenApi";
import { hasSessionFeature, setSessionFeatures } from "./sessionFeatures";
import { setCityTimeZone } from "./format";

afterEach(() => { vi.unstubAllGlobals(); vi.restoreAllMocks(); setCityTimeZone(null); setSessionFeatures(null); });

function mockJson(status: number, body: unknown) {
  const fn = vi.fn(async () => new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fn);
  return fn;
}
const urlOf = (fn: ReturnType<typeof vi.fn>, i = 0) => new URL(String(fn.mock.calls[i][0]), "http://curitiba.localhost");

describe("citizenApi — Meu histórico (módulo 20)", () => {
  it("history: citizen_id, limit 20, kinds[] e o cursor como veio; nada pessoal na URL", async () => {
    const fn = mockJson(200, { verified: true, partial: false, entries: [], next_before: null });
    await citizenApi.history("p1");
    expect(urlOf(fn).pathname).toBe("/citizen/history");
    expect([ ...urlOf(fn).searchParams.entries() ]).toEqual([ [ "citizen_id", "p1" ], [ "limit", String(HISTORY_PAGE_SIZE) ] ]);

    await citizenApi.history("p 1", { before: "2026-09-10T14:40:00-03:00|clinical_document|doc2", kinds: [ "clinical_document" ] });
    const url = urlOf(fn, 1);
    expect(url.searchParams.get("citizen_id")).toBe("p 1");
    expect(url.searchParams.getAll("kinds[]")).toEqual([ "clinical_document" ]);
    expect(url.searchParams.get("before")).toBe("2026-09-10T14:40:00-03:00|clinical_document|doc2");
    expect(String(fn.mock.calls[1][0])).not.toMatch(/cpf|phone|\d{11}/i);
    expect((fn.mock.calls[0][1] as RequestInit).method).toBe("GET");
  });

  it("history: normaliza o que faltar (api mais novo ou incompleto)", async () => {
    mockJson(200, { entries: [
      { id: "triage:t1", kind: "triage", occurred_at: "2026-09-09T20:15:00-03:00", title: "Triagem respiratória" },
      { id: "clinical_document:doc2", kind: "clinical_document", occurred_at: "2026-09-10T14:40:00-03:00", title: "Receita",
        unit: { name: "UBS Centro" }, document: { id: "doc2", short_code: "P3H8T-2LQ6R" } }
    ] });
    const h = await citizenApi.history("p1");
    expect(h.verified).toBe(false);
    expect(h.partial).toBe(false);
    expect(h.next_before).toBeNull();
    expect(h.entries[0]).toEqual({
      id: "triage:t1", kind: "triage", occurred_at: "2026-09-09T20:15:00-03:00", unit: null, professional_name: null,
      title: "Triagem respiratória", summary: null, status: null, document: null
    });
    expect(h.entries[1].document).toEqual({ id: "doc2", short_code: "P3H8T-2LQ6R", verification_url: null, downloadable: false });
  });

  it("healthSummary: citizen_id, listas vazias e sinais ausentes viram vazio", async () => {
    const fn = mockJson(200, { problems: [ { terminology: "ciap2", code: "T90", label: "Diabetes" } ] });
    const s = await citizenApi.healthSummary("p1");
    expect(urlOf(fn).pathname).toBe("/citizen/health_summary");
    expect(urlOf(fn).searchParams.get("citizen_id")).toBe("p1");
    expect(s).toEqual({
      problems: [ { terminology: "ciap2", code: "T90", label: "Diabetes", since: null } ],
      medications: [], exam_requests: [], vitals: {}
    });
  });

  it("healthSummary: 404 not_available vira ApiError com o código", async () => {
    mockJson(404, { error: "not_available" });
    await expect(citizenApi.healthSummary("p1")).rejects.toEqual(new ApiError(404, "not_available"));
  });

  it("documentPdf: blob, recusa com código e 401", async () => {
    const pdf = new Blob([ "%PDF-1.7" ], { type: "application/pdf" });
    const fn = vi.fn(async () => new Response(pdf, { status: 200, headers: { "Content-Type": "application/pdf" } }));
    vi.stubGlobal("fetch", fn);
    const blob = await citizenApi.documentPdf("doc2");
    expect(blob.type).toBe("application/pdf");
    const [ url, init ] = fn.mock.calls[0] as unknown as [ string, RequestInit ];
    expect(url).toBe("/citizen/documents/doc2/pdf");
    expect(init.credentials).toBe("same-origin");
    expect((init.headers as Record<string, string>).Accept).toBe("application/pdf");

    mockJson(404, { error: "not_found" });
    await expect(citizenApi.documentPdf("doc2")).rejects.toEqual(new ApiError(404, "not_found"));

    mockJson(401, { error: "unauthenticated" });
    const spy = vi.spyOn(window, "dispatchEvent");
    await expect(citizenApi.documentPdf("doc2")).rejects.toEqual(new ApiError(401, "unauthenticated"));
    expect(spy).toHaveBeenCalledWith(expect.objectContaining({ type: "citizen:unauthenticated" }));
  });

  it("sessão: interruptores instalados; ausentes = nenhum; sair limpa", async () => {
    mockJson(200, { phone_masked: "(**) *****-5432", time_zone: "America/Sao_Paulo", features: [ "patient_timeline", 3 ] });
    await citizenApi.currentSession();
    expect(hasSessionFeature("patient_timeline")).toBe(true);

    mockJson(200, { phone_masked: "(**) *****-5432" });
    await citizenApi.currentSession();
    expect(hasSessionFeature("patient_timeline")).toBe(false);

    mockJson(200, { phone_masked: "(**) *****-5432", features: [ "patient_timeline" ] });
    await citizenApi.verifyCode("41998765432", "123456");
    expect(hasSessionFeature("patient_timeline")).toBe(true);
    vi.stubGlobal("fetch", vi.fn(async () => new Response(null, { status: 204 })));
    await citizenApi.signOut();
    expect(hasSessionFeature("patient_timeline")).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/wpda/.claude/mod20 && npx vitest run src/lib/citizenApi.history.test.ts`
Expected: FAIL — `./sessionFeatures` não existe; `citizenApi.history` não existe.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/sessionFeatures.ts
// Interruptores LIGADOS da cidade, vindos da sessão do cidadão (módulo 20,
// Divergência W2). Instalados junto com o fuso a cada leitura da sessão,
// como o setCityTimeZone; "Sair" limpa. Ausente (api antigo) = nenhum.
let current: string[] = [];

export function setSessionFeatures(raw: unknown): void {
  current = Array.isArray(raw) ? raw.filter((key): key is string => typeof key === "string") : [];
}

export function hasSessionFeature(key: string): boolean {
  return current.includes(key);
}
```

Em `src/lib/citizenApi.ts`:

(a) no topo, depois de `import { setCityTimeZone } from "./format";`:

```ts
import { setSessionFeatures } from "./sessionFeatures";
```

(b) `CitizenSession` e `withCityTimeZone` passam a:

```ts
export interface CitizenSession {
  phone_masked: string;
  // Fuso IANA da cidade; ausente em api antigo.
  time_zone?: string;
  // Módulo 20 (Divergência W2): interruptores ligados da cidade; ausente em api antigo.
  features?: string[];
}

function withCityTimeZone(session: CitizenSession): CitizenSession {
  setCityTimeZone(session?.time_zone);
  setSessionFeatures(session?.features);
  return session;
}
```

(c) logo antes de `export interface CitizenSession`, a seção nova:

```ts
// ─── Meu histórico (módulo 20, ADR 0035; contrato §2 e §4; Divergências W1–W3) ───
// Por pessoa do celular (o par, em citizen_id, como triagens e agendamentos):
// só o id da pessoa, os tipos e o cursor na URL. Os textos vêm prontos do api, sem nada clínico livre; quem decide o
// que o par vê (validado ou só declarado) é o api.
export interface CitizenDocumentRef { id: string; short_code: string | null; verification_url: string | null; downloadable: boolean }
export interface CitizenEntry {
  id: string; kind: string; occurred_at: string;
  unit: { name: string } | null; professional_name: string | null;
  title: string; summary: string | null; status: string | null;
  document: CitizenDocumentRef | null;
}
export interface CitizenHistory { verified: boolean; partial: boolean; entries: CitizenEntry[]; next_before: string | null }

export interface SummaryProblem { terminology: string; code: string; label: string; since: string | null }
export interface SummaryMedication { label: string; dosage_summary: string | null; since: string | null }
export interface SummaryExamRequest { sigtap_code: string; label: string; requested_at: string }
export interface HealthVitals {
  blood_pressure?: { systolic: number; diastolic: number; measured_at: string };
  weight_kg?: { value: number; measured_at: string };
  capillary_glucose?: { value: number; moment: string | null; measured_at: string };
}
export interface HealthSummary {
  problems: SummaryProblem[]; medications: SummaryMedication[]; exam_requests: SummaryExamRequest[]; vitals: HealthVitals;
}

export const HISTORY_PAGE_SIZE = 20;
// A seção "Documentos" lê a mesma rota filtrada (Divergência W3).
export const DOCUMENT_KINDS = [ "clinical_document" ];

type RawEntry = Partial<Omit<CitizenEntry, "document" | "unit">> & {
  id: string; unit?: { name?: string } | null; document?: Partial<CitizenDocumentRef> & { id: string } | null;
};

function normalizeEntry(e: RawEntry): CitizenEntry {
  return {
    id: e.id, kind: e.kind ?? "", occurred_at: e.occurred_at ?? "",
    unit: e.unit?.name ? { name: e.unit.name } : null,
    professional_name: e.professional_name ?? null,
    title: e.title ?? "", summary: e.summary ?? null, status: e.status ?? null,
    document: e.document
      ? {
          id: e.document.id, short_code: e.document.short_code ?? null,
          verification_url: e.document.verification_url ?? null, downloadable: e.document.downloadable === true
        }
      : null
  };
}

function normalizeSummary(s: Partial<HealthSummary>): HealthSummary {
  return {
    problems: (s.problems ?? []).map((p) => ({ ...p, since: p.since ?? null })),
    medications: (s.medications ?? []).map((m) => ({ ...m, dosage_summary: m.dosage_summary ?? null, since: m.since ?? null })),
    exam_requests: s.exam_requests ?? [],
    vitals: s.vitals ?? {}
  };
}

// PDF lido com fetch (não por navegação): a tela mostra a recusa em vez de
// uma página de erro, e o arquivo sai por um endereço blob:.
async function pdf(path: string): Promise<Blob> {
  const res = await fetch(`/citizen${path}`, { method: "GET", credentials: "same-origin", headers: { Accept: "application/pdf" } });
  if (!res.ok) {
    const data = await res.json().catch(() => ({}));
    if (res.status === 401) window.dispatchEvent(new Event("citizen:unauthenticated"));
    throw new ApiError(res.status, (data as { error?: string }).error ?? `http_${res.status}`);
  }
  return res.blob();
}

```

(d) dentro de `citizenApi`, `signOut` passa a limpar os interruptores:

```ts
  signOut: () => { setSessionFeatures(null); return call<void>("DELETE", "/session"); },
```

e, depois de `updateContactPreference`, as três chamadas novas (a vírgula vai no fim de `updateContactPreference`):

```ts
  // Meu histórico (módulo 20): linha do tempo da pessoa, mais nova primeiro.
  history: async (citizenId: string, opts: { before?: string | null; kinds?: string[] } = {}): Promise<CitizenHistory> => {
    const params = new URLSearchParams({ citizen_id: citizenId });
    for (const kind of opts.kinds ?? []) params.append("kinds[]", kind);
    if (opts.before) params.set("before", opts.before);
    params.set("limit", String(HISTORY_PAGE_SIZE));
    const d = await call<{ verified?: boolean; partial?: boolean; entries?: RawEntry[]; next_before?: string | null }>(
      "GET", `/history?${params.toString()}`);
    return {
      verified: d.verified === true, partial: d.partial === true,
      entries: (d.entries ?? []).map(normalizeEntry), next_before: d.next_before ?? null
    };
  },
  // Só par validado com prontuário (403 not_verified; 404 not_available).
  healthSummary: async (citizenId: string): Promise<HealthSummary> =>
    normalizeSummary(await call<Partial<HealthSummary>>("GET", `/health_summary?citizen_id=${encodeURIComponent(citizenId)}`)),
  documentPdf: (documentId: string): Promise<Blob> => pdf(`/documents/${encodeURIComponent(documentId)}/pdf`)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/wpda/.claude/mod20 && npx vitest run src/lib && npx tsc --noEmit`
Expected: PASS (o `citizenApi.test.ts` antigo segue verde) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda/.claude/mod20
/opt/homebrew/bin/git add src/lib/citizenApi.ts src/lib/sessionFeatures.ts src/lib/citizenApi.history.test.ts
/opt/homebrew/bin/git commit -m "feat: add the citizen history, health summary and document PDF client

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela de "Meu histórico" e frases das recusas

**Files:**
- Create: `src/lib/history.ts`
- Modify: `src/modules/citizen/ui.tsx`
- Test: `src/lib/history.test.ts`

**Interfaces:**
- Consumes: `CitizenEntry`, `CitizenDocumentRef`, `HealthVitals` (Task 1); `fmtDate`, `fmtDateTime`, `fmtCalendarDate` (`format.ts`).
- Produces:
  ```ts
  export const PARTIAL_NOTICE: string;
  export const NOT_VERIFIED_INVITE: string;
  export const SUMMARY_EMPTY: { problems: string; medications: string; exam_requests: string; vitals: string };
  export function citizenKindLabel(kind: string): string;          // desconhecido → "Registro"
  export function citizenStatusLabel(status: string | null): string | null;  // desconhecido ou nulo → null
  export function mergeEntries(current: CitizenEntry[], next: CitizenEntry[]): CitizenEntry[];
  export function entryPlace(entry: CitizenEntry): string | null;   // "UBS Centro · Dra. Helena Prado"
  export function sinceText(since: string | null): string | null;   // "desde 01/03/2019"
  export function vitalsLines(v: HealthVitals): string[];
  export function documentFilename(doc: CitizenDocumentRef): string;
  export const DOCUMENT_GONE: string;
  ```
  Em `MESSAGES` (`ui.tsx`): `not_verified`, `not_available`, `feature_disabled`, `invalid_cursor`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/history.test.ts
import { describe, it, expect } from "vitest";
import {
  NOT_VERIFIED_INVITE, PARTIAL_NOTICE, citizenKindLabel, citizenStatusLabel, documentFilename, entryPlace, mergeEntries, sinceText,
  vitalsLines
} from "./history";
import type { CitizenEntry } from "./citizenApi";
import { ApiError } from "./citizenApi";
import { messageFor } from "../modules/citizen/ui";

const e = (id: string, over: Partial<CitizenEntry> = {}): CitizenEntry => ({
  id, kind: "triage", occurred_at: "2026-09-09T20:15:00-03:00", unit: null, professional_name: null,
  title: id, summary: null, status: null, document: null, ...over
});

describe("tipos para o cidadão", () => {
  it("rótulos do contrato; desconhecido vira Registro", () => {
    expect(citizenKindLabel("screening")).toBe("Acolhimento");
    expect(citizenKindLabel("controlled_notification")).toBe("Receita de controle especial");
    expect(citizenKindLabel("clinical_document")).toBe("Documento");
    expect(citizenKindLabel("vaccination")).toBe("Registro");
    expect(citizenKindLabel("")).toBe("Registro");
  });

  it("situação em português; código desconhecido não aparece", () => {
    expect(citizenStatusLabel("confirmed")).toBe("confirmado");
    expect(citizenStatusLabel("cancelled_by_citizen")).toBe("cancelado por você");
    expect(citizenStatusLabel("issued")).toBe("emitido");
    expect(citizenStatusLabel("cancelled")).toBe("cancelado");
    expect(citizenStatusLabel("read")).toBe("aviso lido");
    expect(citizenStatusLabel("something_new")).toBeNull();
    expect(citizenStatusLabel(null)).toBeNull();
  });
});

describe("mergeEntries: junta sem repetir, na ordem", () => {
  it("a repetida da página seguinte some; a ordem do api fica", () => {
    expect(mergeEntries([ e("a"), e("b") ], [ e("b"), e("c") ]).map((x) => x.id)).toEqual([ "a", "b", "c" ]);
    expect(mergeEntries([], [ e("a") ]).map((x) => x.id)).toEqual([ "a" ]);
  });
});

describe("textos", () => {
  it("onde e com quem", () => {
    expect(entryPlace(e("x", { unit: { name: "UBS Centro" }, professional_name: "Dra. Helena Prado" }))).toBe("UBS Centro · Dra. Helena Prado");
    expect(entryPlace(e("x", { unit: { name: "UBS Centro" } }))).toBe("UBS Centro");
    expect(entryPlace(e("x"))).toBeNull();
  });

  it("desde quando", () => {
    expect(sinceText("2019-03-01")).toBe("desde 01/03/2019");
    expect(sinceText(null)).toBeNull();
    expect(sinceText("2019-03")).toBeNull();
  });

  it("sinais: só o que existe, com a data", () => {
    expect(vitalsLines({
      blood_pressure: { systolic: 130, diastolic: 85, measured_at: "2026-09-10T14:30:00-03:00" },
      weight_kg: { value: 72.5, measured_at: "2026-09-10T14:30:00-03:00" },
      capillary_glucose: { value: 110, moment: "fasting", measured_at: "2026-08-02T08:00:00-03:00" }
    })).toEqual([
      "Pressão: 130/85 mmHg — 10/09/2026",
      "Peso: 72,5 kg — 10/09/2026",
      "Glicemia capilar: 110 mg/dL (em jejum) — 02/08/2026"
    ]);
    expect(vitalsLines({ capillary_glucose: { value: 180, moment: null, measured_at: "2026-08-02T08:00:00-03:00" } }))
      .toEqual([ "Glicemia capilar: 180 mg/dL — 02/08/2026" ]);
    expect(vitalsLines({})).toEqual([]);
  });

  it("nome do arquivo: só o código (ou o id), nada pessoal", () => {
    expect(documentFilename({ id: "doc2", short_code: "P3H8T-2LQ6R", verification_url: null, downloadable: true }))
      .toBe("documento-P3H8T-2LQ6R.pdf");
    expect(documentFilename({ id: "doc2", short_code: null, verification_url: null, downloadable: true })).toBe("documento-doc2.pdf");
    expect(documentFilename({ id: "../x y", short_code: "a/b", verification_url: null, downloadable: true })).toBe("documento-ab.pdf");
  });

  it("os avisos dizem o essencial", () => {
    expect(PARTIAL_NOTICE).toMatch(/só o que passou pelo Rota Saúde/);
    expect(NOT_VERIFIED_INVITE).toMatch(/valide seu cadastro/);
  });

  it("frases novas das recusas", () => {
    expect(messageFor(new ApiError(403, "feature_disabled"))).toBe("O histórico ainda não está disponível nesta cidade.");
    expect(messageFor(new ApiError(403, "not_verified"))).toBe("Para ver isto, valide seu cadastro na unidade de saúde.");
    expect(messageFor(new ApiError(422, "invalid_cursor"))).toBe("A lista mudou. Volte e abra o histórico de novo.");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/wpda/.claude/mod20 && npx vitest run src/lib/history.test.ts`
Expected: FAIL — `./history` não existe.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/history.ts
// Regras de tela de "Meu histórico" (módulo 20, F-20.4 e F-20.5; spec §4 e
// §8; contrato §2 e §4). Os textos das entradas vêm prontos do api; aqui só o
// rótulo do tipo, a junção das páginas, as linhas do resumo e o nome do PDF.
import type { CitizenDocumentRef, CitizenEntry, HealthVitals } from "./citizenApi";
import { fmtCalendarDate, fmtDate } from "./format";

export const PARTIAL_NOTICE =
  "Este histórico mostra só o que passou pelo Rota Saúde. Consultas e exames registrados em outros sistemas da prefeitura não aparecem aqui.";
export const NOT_VERIFIED_INVITE =
  "Aqui aparecem só as suas triagens e os seus agendamentos. Para ver também o resumo de saúde, as consultas e os documentos, valide seu cadastro na unidade de saúde com um documento com foto.";
export const SUMMARY_EMPTY = {
  problems: "Nenhum problema de saúde ativo registrado.",
  medications: "Nenhum medicamento em uso registrado.",
  exam_requests: "Nenhum exame pedido nos últimos 12 meses.",
  vitals: "Nenhuma medida registrada."
};
export const DOCUMENT_GONE = "Este documento não está mais disponível. Ele pode ter sido cancelado.";

const KIND_LABEL: Record<string, string> = {
  triage: "Triagem",
  attendance: "Atendimento",
  screening: "Acolhimento",
  appointment: "Agendamento",
  consultation: "Consulta",
  clinical_document: "Documento",
  controlled_notification: "Receita de controle especial",
  campaign: "Campanha"
};

// O cidadão não vê código de máquina: tipo novo do api vira "Registro".
export function citizenKindLabel(kind: string): string {
  return KIND_LABEL[kind] ?? "Registro";
}

// Situação das origens, em palavras do cidadão. Código novo do api: não mostra.
const STATUS_LABEL: Record<string, string> = {
  completed: "concluído",
  aborted_by_revocation: "consentimento revogado",
  aborted_by_timeout: "interrompido",
  aborted_by_cancellation: "cancelado",
  waiting: "aguardando atendimento",
  in_care: "em atendimento",
  closed: "atendimento encerrado",
  scheduled: "marcado",
  confirmed: "confirmado",
  checked_in: "você compareceu",
  cancelled_by_citizen: "cancelado por você",
  expired: "não confirmado no prazo",
  no_show: "falta",
  finalized: "realizada",
  issued: "emitido",
  cancelled: "cancelado",
  read: "aviso lido",
  unread: "aviso não lido"
};

export function citizenStatusLabel(status: string | null): string | null {
  return status ? STATUS_LABEL[status] ?? null : null;
}

export function mergeEntries(current: CitizenEntry[], next: CitizenEntry[]): CitizenEntry[] {
  const seen = new Set(current.map((e) => e.id));
  return [ ...current, ...next.filter((e) => !seen.has(e.id)) ];
}

export function entryPlace(entry: CitizenEntry): string | null {
  const parts = [ entry.unit?.name, entry.professional_name ].filter((p): p is string => !!p);
  return parts.length > 0 ? parts.join(" · ") : null;
}

export function sinceText(since: string | null): string | null {
  const day = fmtCalendarDate(since);
  return day === "—" ? null : `desde ${day}`;
}

const GLUCOSE_MOMENT: Record<string, string> = { fasting: "em jejum", postprandial: "depois de comer", random: "a qualquer hora" };
const number = (n: number) => n.toLocaleString("pt-BR");

export function vitalsLines(v: HealthVitals): string[] {
  const out: string[] = [];
  if (v.blood_pressure) {
    out.push(`Pressão: ${v.blood_pressure.systolic}/${v.blood_pressure.diastolic} mmHg — ${fmtDate(v.blood_pressure.measured_at)}`);
  }
  if (v.weight_kg) out.push(`Peso: ${number(v.weight_kg.value)} kg — ${fmtDate(v.weight_kg.measured_at)}`);
  if (v.capillary_glucose) {
    const g = v.capillary_glucose;
    const moment = g.moment ? GLUCOSE_MOMENT[g.moment] : null;
    out.push(`Glicemia capilar: ${number(g.value)} mg/dL${moment ? ` (${moment})` : ""} — ${fmtDate(g.measured_at)}`);
  }
  return out;
}

// Só letras, números e hífen: nunca CPF, nome ou separador de caminho.
export function documentFilename(doc: CitizenDocumentRef): string {
  const raw = (doc.short_code ?? doc.id).replace(/[^A-Za-z0-9-]/g, "");
  return `documento-${raw || "pdf"}.pdf`;
}
```

Em `src/modules/citizen/ui.tsx`, no `MESSAGES`, depois de `note_too_long: "Escreva no máximo 200 caracteres."` (a vírgula vai nessa linha):

```ts
  note_too_long: "Escreva no máximo 200 caracteres.",
  // Módulo 20 (contrato §4).
  feature_disabled: "O histórico ainda não está disponível nesta cidade.",
  not_verified: "Para ver isto, valide seu cadastro na unidade de saúde.",
  not_available: "O resumo de saúde ainda não está disponível.",
  invalid_cursor: "A lista mudou. Volte e abra o histórico de novo."
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/wpda/.claude/mod20 && npx vitest run src/lib/history.test.ts src/modules/citizen/ui.test.tsx && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda/.claude/mod20
/opt/homebrew/bin/git add src/lib/history.ts src/lib/history.test.ts src/modules/citizen/ui.tsx
/opt/homebrew/bin/git commit -m "feat: add citizen history screen rules and refusal messages

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: A tela "Meu histórico"

**Files:**
- Create: `src/modules/citizen/MyHistoryStep.tsx`
- Test: `src/modules/citizen/MyHistoryStep.test.tsx`

**Interfaces:**
- Consumes: `citizenApi.history`, `citizenApi.healthSummary`, `citizenApi.documentPdf`, `ApiError`, `DOCUMENT_KINDS`, `CitizenEntry`, `CitizenHistory`, `HealthSummary` (Task 1); tudo de `history.ts` (Task 2); `Screen`, `BigButton`, `ErrorText`, `messageFor` (`ui.tsx`); `fmtDate`, `fmtDateTime` (`format.ts`).
- Produces: `export function MyHistoryStep({ citizenId, onBack, onValidate }: { citizenId: string; onBack(): void; onValidate(citizenId: string): void }): JSX.Element` — `Screen` "Meu histórico" com, nesta ordem: a nota `role="note"` "histórico parcial"; para par declarado, a nota `role="note"` "cadastro não validado" e o botão "Validar no posto"; para par validado, as seções `aria-label="Resumo de saúde"` (só se o resumo existir) e `aria-label="Documentos"` (com "Baixar PDF" e o link "Conferir o documento <código>"); e a seção `aria-label="Linha do tempo"`; cada lista com "Carregar mais" enquanto houver `next_before`; "Voltar" no rodapé.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/MyHistoryStep.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { MyHistoryStep } from "./MyHistoryStep";
import { citizenApi, ApiError, type CitizenEntry, type CitizenHistory, type HealthSummary } from "../../lib/citizenApi";
import { DOCUMENT_GONE, NOT_VERIFIED_INVITE, PARTIAL_NOTICE } from "../../lib/history";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-11T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); vi.unstubAllGlobals(); });

const entry = (over: Partial<CitizenEntry>): CitizenEntry => ({
  id: "x", kind: "triage", occurred_at: "2026-09-09T20:15:00-03:00", unit: null, professional_name: null,
  title: "", summary: null, status: null, document: null, ...over
});
const APPOINTMENT = entry({ id: "appointment:ap1", kind: "appointment", occurred_at: "2026-10-01T09:00:00-03:00",
  unit: { name: "UBS Centro" }, professional_name: "Enf. Lúcia Prado", title: "Retorno — Consulta de enfermagem", status: "confirmed" });
const DOCUMENT = entry({ id: "clinical_document:doc2", kind: "clinical_document", occurred_at: "2026-09-10T14:40:00-03:00",
  unit: { name: "UBS Centro" }, professional_name: "Dra. Helena Prado", title: "Receita", status: "issued",
  document: { id: "doc2", short_code: "P3H8T-2LQ6R", verification_url: "https://curitiba.rotasaude.app/v/tok2", downloadable: true } });
const CONSULTATION = entry({ id: "consultation:cs0", kind: "consultation", occurred_at: "2026-09-10T14:30:00-03:00",
  unit: { name: "UBS Centro" }, title: "Consulta com Dra. Helena Prado" });
const TRIAGE = entry({ id: "triage:tr1", title: "Triagem respiratória", summary: "prioridade 2" });

const history = (over: Partial<CitizenHistory> = {}): CitizenHistory =>
  ({ verified: true, partial: false, entries: [ APPOINTMENT, DOCUMENT, CONSULTATION, TRIAGE ], next_before: null, ...over });
const SUMMARY: HealthSummary = {
  problems: [ { terminology: "ciap2", code: "T90", label: "Diabetes", since: "2019-03-01" } ],
  medications: [ { label: "Metformina 850 mg", dosage_summary: "1 comprimido 2 vezes ao dia", since: null } ],
  exam_requests: [ { sigtap_code: "0202010503", label: "Hemoglobina glicada", requested_at: "2026-09-10T14:30:00-03:00" } ],
  vitals: { blood_pressure: { systolic: 130, diastolic: 85, measured_at: "2026-09-10T14:30:00-03:00" } }
};

function api(opts: { timeline?: CitizenHistory; documents?: CitizenHistory; summary?: HealthSummary | ApiError } = {}) {
  const historySpy = vi.spyOn(citizenApi, "history").mockImplementation(async (_id, o = {}) =>
    (o.kinds?.length ? (opts.documents ?? history({ entries: [ DOCUMENT ] })) : (opts.timeline ?? history())));
  const summary = opts.summary ?? SUMMARY;
  const summarySpy = vi.spyOn(citizenApi, "healthSummary").mockImplementation(async () => {
    if (summary instanceof ApiError) throw summary;
    return summary;
  });
  return { historySpy, summarySpy };
}

const renderStep = (onValidate = vi.fn(), onBack = vi.fn()) =>
  render(<MyHistoryStep citizenId="p1" onBack={onBack} onValidate={onValidate} />);

describe("MyHistoryStep", () => {
  it("par validado: resumo, documentos e linha do tempo", async () => {
    const { historySpy, summarySpy } = api();
    renderStep();

    const summary = await screen.findByRole("region", { name: "Resumo de saúde" });
    expect(within(summary).getByText("Diabetes")).toBeInTheDocument();
    expect(within(summary).getByText("desde 01/03/2019")).toBeInTheDocument();
    expect(within(summary).getByText("Metformina 850 mg — 1 comprimido 2 vezes ao dia")).toBeInTheDocument();
    expect(within(summary).getByText("Hemoglobina glicada — pedido em 10/09/2026")).toBeInTheDocument();
    expect(within(summary).getByText("Pressão: 130/85 mmHg — 10/09/2026")).toBeInTheDocument();

    const docs = await screen.findByRole("region", { name: "Documentos" });
    expect(within(docs).getByText("Receita")).toBeInTheDocument();
    expect(within(docs).getByText("Código de conferência: P3H8T-2LQ6R")).toBeInTheDocument();
    expect(within(docs).getByRole("link", { name: "Conferir o documento P3H8T-2LQ6R" }))
      .toHaveAttribute("href", "https://curitiba.rotasaude.app/v/tok2");
    expect(within(docs).getByRole("button", { name: "Baixar PDF de Receita" })).toBeInTheDocument();

    const timeline = screen.getByRole("region", { name: "Linha do tempo" });
    const text = timeline.textContent ?? "";
    const order = [ "Retorno — Consulta de enfermagem", "Receita", "Consulta com Dra. Helena Prado", "Triagem respiratória" ]
      .map((t) => text.indexOf(t));
    expect([ ...order ].sort((a, b) => a - b)).toEqual(order);
    expect(within(timeline).getByText("Agendamento")).toBeInTheDocument();
    expect(within(timeline).getByText("UBS Centro · Enf. Lúcia Prado")).toBeInTheDocument();
    expect(within(timeline).getByText("01/10/2026, 09:00")).toBeInTheDocument();
    expect(within(timeline).getByText("confirmado")).toBeInTheDocument();
    expect(within(timeline).queryByText("confirmed")).not.toBeInTheDocument();

    expect(historySpy).toHaveBeenCalledWith("p1");
    expect(historySpy).toHaveBeenCalledWith("p1", { kinds: [ "clinical_document" ] });
    expect(summarySpy).toHaveBeenCalledWith("p1");
    expect(screen.queryByRole("note", { name: "histórico parcial" })).not.toBeInTheDocument();
    expect(screen.queryByRole("note", { name: "cadastro não validado" })).not.toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("par declarado: triagens e agendamentos, convite para validar e nada de resumo ou documentos", async () => {
    const onValidate = vi.fn();
    const { historySpy, summarySpy } = api({ timeline: history({ verified: false, entries: [ APPOINTMENT, TRIAGE ] }) });
    renderStep(onValidate);
    expect((await screen.findByRole("note", { name: "cadastro não validado" })).textContent).toBe(NOT_VERIFIED_INVITE);
    expect(screen.getByText("Triagem respiratória")).toBeInTheDocument();
    expect(screen.queryByRole("region", { name: "Resumo de saúde" })).not.toBeInTheDocument();
    expect(screen.queryByRole("region", { name: "Documentos" })).not.toBeInTheDocument();
    expect(summarySpy).not.toHaveBeenCalled();
    expect(historySpy).toHaveBeenCalledTimes(1);
    await userEvent.click(screen.getByRole("button", { name: "Validar no posto" }));
    expect(onValidate).toHaveBeenCalledWith("p1");
  });

  it("resumo indisponível (404): sem a seção e sem erro", async () => {
    api({ summary: new ApiError(404, "not_available") });
    renderStep();
    expect(await screen.findByRole("region", { name: "Documentos" })).toBeInTheDocument();
    expect(screen.queryByRole("region", { name: "Resumo de saúde" })).not.toBeInTheDocument();
    expect(screen.queryByRole("alert")).not.toBeInTheDocument();
  });

  it("resumo com listas vazias: as frases de vazio", async () => {
    api({ summary: { problems: [], medications: [], exam_requests: [], vitals: {} } });
    renderStep();
    const summary = await screen.findByRole("region", { name: "Resumo de saúde" });
    expect(within(summary).getByText("Nenhum problema de saúde ativo registrado.")).toBeInTheDocument();
    expect(within(summary).getByText("Nenhuma medida registrada.")).toBeInTheDocument();
  });

  it("histórico parcial: o aviso", async () => {
    api({ timeline: history({ partial: true }) });
    renderStep();
    expect((await screen.findByRole("note", { name: "histórico parcial" })).textContent).toBe(PARTIAL_NOTICE);
  });

  it("carregar mais: cursor como veio e sem repetir", async () => {
    const historySpy = vi.spyOn(citizenApi, "history").mockImplementation(async (_id, o = {}) => {
      if (o.kinds?.length) return history({ entries: [] });
      return o.before
        ? history({ entries: [ DOCUMENT, CONSULTATION ], next_before: null })
        : history({ entries: [ APPOINTMENT, DOCUMENT ], next_before: "2026-09-10T14:40:00-03:00|clinical_document|doc2" });
    });
    vi.spyOn(citizenApi, "healthSummary").mockResolvedValue(SUMMARY);
    renderStep();
    const timeline = await screen.findByRole("region", { name: "Linha do tempo" });
    await userEvent.click(await within(timeline).findByRole("button", { name: "Carregar mais" }));
    expect(await within(timeline).findByText("Consulta com Dra. Helena Prado")).toBeInTheDocument();
    expect(historySpy).toHaveBeenLastCalledWith("p1", { before: "2026-09-10T14:40:00-03:00|clinical_document|doc2" });
    expect(within(timeline).getAllByText("Receita")).toHaveLength(1);
    expect(within(timeline).queryByRole("button", { name: "Carregar mais" })).not.toBeInTheDocument();
  });

  it("baixar o PDF: arquivo com o código", async () => {
    api();
    const blob = new Blob([ "%PDF-1.7" ], { type: "application/pdf" });
    const pdf = vi.spyOn(citizenApi, "documentPdf").mockResolvedValue(blob);
    const createObjectURL = vi.fn(() => "blob:http://curitiba.localhost/abc");
    // O jsdom não tem createObjectURL/revokeObjectURL: instala os dublês no URL global.
    Object.assign(URL, { createObjectURL, revokeObjectURL: vi.fn() });
    const click = vi.spyOn(HTMLAnchorElement.prototype, "click").mockImplementation(function (this: HTMLAnchorElement) {
      expect(this.download).toBe("documento-P3H8T-2LQ6R.pdf");
      expect(this.href).toBe("blob:http://curitiba.localhost/abc");
    });
    renderStep();
    const docs = await screen.findByRole("region", { name: "Documentos" });
    await userEvent.click(within(docs).getByRole("button", { name: "Baixar PDF de Receita" }));
    expect(pdf).toHaveBeenCalledWith("doc2");
    expect(createObjectURL).toHaveBeenCalledWith(blob);
    expect(click).toHaveBeenCalledTimes(1);
  });

  it("documento que sumiu: diz e não baixa", async () => {
    api();
    vi.spyOn(citizenApi, "documentPdf").mockRejectedValue(new ApiError(404, "not_found"));
    const click = vi.spyOn(HTMLAnchorElement.prototype, "click");
    renderStep();
    const docs = await screen.findByRole("region", { name: "Documentos" });
    await userEvent.click(within(docs).getByRole("button", { name: "Baixar PDF de Receita" }));
    expect(await screen.findByText(DOCUMENT_GONE)).toBeInTheDocument();
    expect(click).not.toHaveBeenCalled();
  });

  it("documento sem PDF (downloadable false): só o código, sem botão", async () => {
    api({ documents: history({ entries: [ { ...DOCUMENT, document: { ...DOCUMENT.document!, downloadable: false } } ] }) });
    renderStep();
    const docs = await screen.findByRole("region", { name: "Documentos" });
    expect(within(docs).getByText("Código de conferência: P3H8T-2LQ6R")).toBeInTheDocument();
    expect(within(docs).queryByRole("button", { name: /Baixar PDF/ })).not.toBeInTheDocument();
  });

  it("tipo desconhecido vira Registro", async () => {
    api({ timeline: history({ entries: [ entry({ id: "vaccination:v1", kind: "vaccination", title: "Vacina contra a gripe" }) ] }) });
    renderStep();
    const timeline = await screen.findByRole("region", { name: "Linha do tempo" });
    expect(within(timeline).getByText("Registro")).toBeInTheDocument();
    expect(within(timeline).queryByText("vaccination")).not.toBeInTheDocument();
  });

  it("interruptor desligado no meio: a frase", async () => {
    vi.spyOn(citizenApi, "history").mockRejectedValue(new ApiError(403, "feature_disabled"));
    renderStep();
    expect(await screen.findByRole("alert")).toHaveTextContent("O histórico ainda não está disponível nesta cidade.");
  });

  it("Voltar", async () => {
    api();
    const onBack = vi.fn();
    renderStep(vi.fn(), onBack);
    await userEvent.click(await screen.findByRole("button", { name: "Voltar" }));
    expect(onBack).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/wpda/.claude/mod20 && npx vitest run src/modules/citizen/MyHistoryStep.test.tsx`
Expected: FAIL — `./MyHistoryStep` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/citizen/MyHistoryStep.tsx
// "Meu histórico" de uma pessoa do celular (módulo 20, F-20.4 e F-20.5; spec
// §2, §4 e §8; contrato §2 e §4). Par validado: resumo de saúde (quando
// houver), documentos com o PDF e o código de conferência, e a linha do
// tempo. Par só declarado: as próprias triagens e agendamentos e o convite
// para validar. O que o par vê é do api; a tela nem pede resumo ou
// documentos para par declarado.
import { useEffect, useState, type CSSProperties } from "react";
import {
  citizenApi, ApiError, DOCUMENT_KINDS, type CitizenDocumentRef, type CitizenEntry, type CitizenHistory, type HealthSummary
} from "../../lib/citizenApi";
import {
  DOCUMENT_GONE, NOT_VERIFIED_INVITE, PARTIAL_NOTICE, SUMMARY_EMPTY, citizenKindLabel, citizenStatusLabel, documentFilename,
  entryPlace, mergeEntries, sinceText, vitalsLines
} from "../../lib/history";
import { fmtDate, fmtDateTime } from "../../lib/format";
import { BigButton, ErrorText, Screen, messageFor } from "./ui";

const PDF_URL_TTL_MS = 60_000;

type ListKey = "timeline" | "documents";

export function MyHistoryStep({ citizenId, onBack, onValidate }:
  { citizenId: string; onBack: () => void; onValidate: (citizenId: string) => void }) {
  const [timeline, setTimeline] = useState<CitizenHistory | null>(null);
  const [documents, setDocuments] = useState<CitizenHistory | null>(null);
  const [summary, setSummary] = useState<HealthSummary | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [more, setMore] = useState<ListKey | null>(null);
  const [downloading, setDownloading] = useState<string | null>(null);
  const [downloadError, setDownloadError] = useState<string | null>(null);

  useEffect(() => {
    // StrictMode monta duas vezes em dev: a resposta da montagem desfeita não escreve.
    let alive = true;
    citizenApi.history(citizenId)
      .then((h) => {
        if (!alive) return;
        setTimeline(h);
        if (!h.verified) return;
        citizenApi.healthSummary(citizenId)
          .then((s) => { if (alive) setSummary(s); })
          // Sem prontuário (modo integrado) ou sem paciente: só não há resumo.
          .catch((e) => { if (alive && !(e instanceof ApiError && e.code === "not_available")) setError(messageFor(e)); });
        citizenApi.history(citizenId, { kinds: DOCUMENT_KINDS })
          .then((d) => { if (alive) setDocuments(d); })
          .catch((e) => { if (alive) setError(messageFor(e)); });
      })
      .catch((e) => { if (alive) setError(messageFor(e)); });
    return () => { alive = false; };
  }, [citizenId]);

  async function loadMore(list: ListKey) {
    const current = list === "timeline" ? timeline : documents;
    if (!current?.next_before || more) return;
    setMore(list);
    try {
      const next = await citizenApi.history(citizenId,
        list === "timeline" ? { before: current.next_before } : { before: current.next_before, kinds: DOCUMENT_KINDS });
      const merged = { ...current, entries: mergeEntries(current.entries, next.entries), next_before: next.next_before };
      if (list === "timeline") setTimeline(merged); else setDocuments(merged);
    } catch (e) {
      setError(messageFor(e));
    } finally {
      setMore(null);
    }
  }

  async function download(doc: CitizenDocumentRef) {
    if (downloading) return;
    setDownloading(doc.id); setDownloadError(null);
    try {
      const blob = await citizenApi.documentPdf(doc.id);
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = documentFilename(doc);
      a.click();
      setTimeout(() => URL.revokeObjectURL(url), PDF_URL_TTL_MS);
    } catch (e) {
      setDownloadError(e instanceof ApiError && e.code === "not_found" ? DOCUMENT_GONE : messageFor(e));
    } finally {
      setDownloading(null);
    }
  }

  return (
    <Screen title="Meu histórico" footer={<BigButton variant="secondary" onClick={onBack}>Voltar</BigButton>}>
      {error && <ErrorText>{error}</ErrorText>}
      {!timeline && !error && <p>Carregando…</p>}
      {timeline && (
        <div style={{ display: "grid", gap: 24 }}>
          {timeline.partial && <p role="note" aria-label="histórico parcial" style={notice}>{PARTIAL_NOTICE}</p>}
          {!timeline.verified && (
            <div style={{ display: "grid", gap: 12 }}>
              <p role="note" aria-label="cadastro não validado" style={notice}>{NOT_VERIFIED_INVITE}</p>
              <BigButton onClick={() => onValidate(citizenId)}>Validar no posto</BigButton>
            </div>
          )}
          {timeline.verified && summary && <SummarySection summary={summary} />}
          {timeline.verified && documents && (
            <section aria-label="Documentos">
              <h2 style={h2}>Documentos</h2>
              {downloadError && <ErrorText>{downloadError}</ErrorText>}
              {documents.entries.length === 0 && <p>Nenhum documento ainda.</p>}
              <EntryList entries={documents.entries} downloading={downloading} onDownload={download} />
              {documents.next_before && (
                <BigButton variant="secondary" disabled={more !== null} onClick={() => void loadMore("documents")}>Carregar mais</BigButton>
              )}
            </section>
          )}
          <section aria-label="Linha do tempo">
            <h2 style={h2}>Linha do tempo</h2>
            {timeline.entries.length === 0 && <p>Nenhum registro ainda.</p>}
            <EntryList entries={timeline.entries} downloading={downloading} onDownload={download} />
            {timeline.next_before && (
              <BigButton variant="secondary" disabled={more !== null} onClick={() => void loadMore("timeline")}>Carregar mais</BigButton>
            )}
          </section>
        </div>
      )}
    </Screen>
  );
}

function SummarySection({ summary }: { summary: HealthSummary }) {
  const vitals = vitalsLines(summary.vitals);
  return (
    <section aria-label="Resumo de saúde">
      <h2 style={h2}>Resumo de saúde</h2>
      <h3 style={h3}>Problemas de saúde</h3>
      {summary.problems.length === 0 ? <p>{SUMMARY_EMPTY.problems}</p> : (
        <ul style={list}>
          {summary.problems.map((p) => (
            <li key={`${p.terminology}:${p.code}`}>
              <div>{p.label}</div>
              {sinceText(p.since) && <div style={muted}>{sinceText(p.since)}</div>}
            </li>
          ))}
        </ul>
      )}
      <h3 style={h3}>Medicamentos em uso</h3>
      {summary.medications.length === 0 ? <p>{SUMMARY_EMPTY.medications}</p> : (
        <ul style={list}>
          {summary.medications.map((m, i) => (
            <li key={`${m.label}-${i}`}>{m.dosage_summary ? `${m.label} — ${m.dosage_summary}` : m.label}</li>
          ))}
        </ul>
      )}
      <h3 style={h3}>Exames pedidos nos últimos 12 meses</h3>
      {summary.exam_requests.length === 0 ? <p>{SUMMARY_EMPTY.exam_requests}</p> : (
        <ul style={list}>
          {summary.exam_requests.map((x, i) => (
            <li key={`${x.sigtap_code}-${i}`}>{`${x.label} — pedido em ${fmtDate(x.requested_at)}`}</li>
          ))}
        </ul>
      )}
      <h3 style={h3}>Últimas medidas</h3>
      {vitals.length === 0 ? <p>{SUMMARY_EMPTY.vitals}</p> : (
        <ul style={list}>{vitals.map((line) => <li key={line}>{line}</li>)}</ul>
      )}
    </section>
  );
}

function EntryList({ entries, downloading, onDownload }:
  { entries: CitizenEntry[]; downloading: string | null; onDownload: (doc: CitizenDocumentRef) => void }) {
  return (
    <ul style={{ listStyle: "none", padding: 0, margin: "0 0 12px", display: "grid", gap: 12 }}>
      {entries.map((e) => {
        const place = entryPlace(e);
        const status = citizenStatusLabel(e.status);
        return (
          <li key={e.id} style={card}>
            <strong>{citizenKindLabel(e.kind)}</strong>
            <div>{fmtDateTime(e.occurred_at)}</div>
            <div>{e.title}</div>
            {e.summary && <div style={muted}>{e.summary}</div>}
            {place && <div style={muted}>{place}</div>}
            {status && <div>{status}</div>}
            {e.document && (
              <div style={{ display: "grid", gap: 8, marginTop: 8 }}>
                {e.document.short_code && <div>{`Código de conferência: ${e.document.short_code}`}</div>}
                {e.document.verification_url && e.document.short_code && (
                  <a href={e.document.verification_url} style={link}>{`Conferir o documento ${e.document.short_code}`}</a>
                )}
                {e.document.downloadable && (
                  <BigButton variant="secondary" disabled={downloading !== null} aria-label={`Baixar PDF de ${e.title}`}
                    onClick={() => onDownload(e.document as CitizenDocumentRef)}>
                    {downloading === e.document.id ? "Baixando…" : "Baixar PDF"}
                  </BigButton>
                )}
              </div>
            )}
          </li>
        );
      })}
    </ul>
  );
}

const notice: CSSProperties = { background: "var(--warnBg, #fff4e0)", padding: 12, borderRadius: 12, margin: 0 };
const h2: CSSProperties = { fontSize: 20, margin: "0 0 12px" };
const h3: CSSProperties = { fontSize: 18, margin: "16px 0 8px" };
const list: CSSProperties = { margin: 0, paddingLeft: 20, display: "grid", gap: 8 };
const card: CSSProperties = { border: "1px solid var(--rule2, #ccc)", borderRadius: 12, padding: 12, display: "grid", gap: 4 };
const muted: CSSProperties = { color: "var(--ink2, #555)" };
const link: CSSProperties = { display: "inline-block", minHeight: 48, lineHeight: "48px", fontSize: 18 };
```

(O título da entrada vai no nome acessível do botão — "Baixar PDF de Receita" — para quem usa leitor de tela distinguir os documentos; o texto visível continua "Baixar PDF".)

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/wpda/.claude/mod20 && npx vitest run src/modules/citizen/MyHistoryStep.test.tsx && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda/.claude/mod20
/opt/homebrew/bin/git add src/modules/citizen/MyHistoryStep.tsx src/modules/citizen/MyHistoryStep.test.tsx
/opt/homebrew/bin/git commit -m "feat: show the citizen history with health summary, documents and timeline

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: O botão em "Minhas triagens" e o estado no `Flow`

**Files:**
- Modify: `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `README.md`
- Test: `src/modules/citizen/Flow.history.test.tsx`

**Interfaces:**
- Consumes: `MyHistoryStep` (Task 3); `hasSessionFeature` (Task 1).
- Produces: `HistoryStep` ganha a prop opcional `onRecord?: () => void` (botão "Meu histórico", logo depois de "Seus agendamentos"); `Flow` ganha o estado `{ at: "record"; citizenId: string; consentVersion: string | null }`, passa `onRecord` só com `hasSessionFeature("patient_timeline")`, e de "Meu histórico" volta a "Minhas triagens"; "Validar no posto" vai ao código do posto (o mesmo estado `verify-code` de hoje).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/Flow.history.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Flow } from "./Flow";
import { citizenApi } from "../../lib/citizenApi";
import { setSessionFeatures } from "../../lib/sessionFeatures";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-11T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); setSessionFeatures(null); window.history.replaceState(null, "", "/"); });

function signedIn(features?: string[]) {
  // O spy não passa pelo withCityTimeZone do cliente: instala os interruptores como ele faria.
  vi.spyOn(citizenApi, "currentSession").mockImplementation(async () => {
    setSessionFeatures(features);
    return { phone_masked: "(**) *****-5432", ...(features ? { features } : {}) };
  });
  vi.spyOn(citizenApi, "consentTerm").mockResolvedValue({ version: "1", body: "Termo" });
  vi.spyOn(citizenApi, "people").mockResolvedValue({
    people: [ { id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared", neighborhood: null, profile: null } ]
  });
  vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue([]);
  vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [], unread_count: 0 });
  vi.spyOn(citizenApi, "appointments").mockResolvedValue({ appointments: [] });
  vi.spyOn(citizenApi, "triages").mockResolvedValue({
    citizen: { id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared" }, triages: []
  });
  vi.spyOn(citizenApi, "history").mockResolvedValue({ verified: false, partial: true, entries: [], next_before: null });
  vi.spyOn(citizenApi, "issueVerificationCode").mockResolvedValue({ code: "482913", expires_at: "2026-10-11T10:10:00-03:00" });
}

async function toMyTriages() {
  render(<Flow />);
  await userEvent.click(await screen.findByRole("button", { name: "Concordo" }));
  await userEvent.click(await screen.findByRole("button", { name: "Ver triagens de ***.982.247-**" }));
  expect(await screen.findByRole("heading", { name: "Minhas triagens" })).toBeInTheDocument();
}

describe("Flow — Meu histórico (módulo 20)", () => {
  it("com patient_timeline: Minhas triagens → Meu histórico → Validar no posto e volta", async () => {
    signedIn([ "patient_timeline" ]);
    await toMyTriages();
    await userEvent.click(screen.getByRole("button", { name: "Meu histórico" }));
    expect(await screen.findByRole("heading", { name: "Meu histórico" })).toBeInTheDocument();
    expect(citizenApi.history).toHaveBeenCalledWith("p1");
    expect(await screen.findByRole("note", { name: "cadastro não validado" })).toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Voltar" }));
    expect(await screen.findByRole("heading", { name: "Minhas triagens" })).toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Meu histórico" }));
    await userEvent.click(await screen.findByRole("button", { name: "Validar no posto" }));
    expect(await screen.findByText("482913")).toBeInTheDocument();
    expect(citizenApi.issueVerificationCode).toHaveBeenCalledWith("p1");
  });

  it("sem patient_timeline na sessão: Minhas triagens como hoje", async () => {
    signedIn();
    await toMyTriages();
    expect(screen.queryByRole("button", { name: "Meu histórico" })).not.toBeInTheDocument();
    expect(citizenApi.history).not.toHaveBeenCalled();
  });
});
```

(Se o texto do código no `CounterCodeStep` vier formatado — por exemplo `482 913` —, troque o `findByText` pelo formato real que o componente usa; confira em `src/modules/citizen/CounterCodeStep.test.tsx`.)

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/wpda/.claude/mod20 && npx vitest run src/modules/citizen/Flow.history.test.tsx`
Expected: FAIL — não há botão "Meu histórico".

- [ ] **Step 3: Write minimal implementation**

Em `src/modules/citizen/HistoryStep.tsx`, a assinatura passa a

```tsx
export function HistoryStep({ citizenId, onBack, onValidate, onCheckIn, onAppointmentCheckIn, onRecord }:
  { citizenId: string; onBack: () => void; onValidate: (citizenId: string) => void; onCheckIn: (triageId: string) => void;
    onAppointmentCheckIn: (appointmentId: string) => void;
    // Módulo 20: "Meu histórico", só com patient_timeline ligado (quem decide é o Flow).
    onRecord?: () => void }) {
```

e, logo depois de `<AppointmentsSection citizenId={citizenId} onCheckIn={onAppointmentCheckIn} />`:

```tsx
          {onRecord && <BigButton variant="secondary" onClick={onRecord} style={{ marginBottom: 16 }}>Meu histórico</BigButton>}
```

Em `src/modules/citizen/Flow.tsx`:

(a) imports, depois de `import { HistoryStep } from "./HistoryStep";`:

```tsx
import { MyHistoryStep } from "./MyHistoryStep";
import { hasSessionFeature } from "../../lib/sessionFeatures";
```

(b) no tipo `State`, depois da linha de `history`:

```tsx
  | { at: "record"; consentVersion: string | null; citizenId: string }
```

(c) no `case "history":`, acrescente a prop ao `HistoryStep` (depois de `onAppointmentCheckIn={…}`):

```tsx
        onRecord={hasSessionFeature("patient_timeline")
          ? () => setState({ at: "record", consentVersion: state.consentVersion, citizenId: state.citizenId })
          : undefined}
```

(d) logo antes de `case "verify-code":`:

```tsx
    case "record":
      view = <MyHistoryStep key={state.citizenId} citizenId={state.citizenId}
        onBack={() => setState({ at: "history", citizenId: state.citizenId, consentVersion: state.consentVersion })}
        onValidate={id => setState({ at: "verify-code", citizenId: id, consentVersion: state.consentVersion })} />;
      break;
```

No `README.md` do wpda, no fim da seção do canal web do cidadão:

```markdown

**Meu histórico (módulo 20).** Com `patient_timeline` ligado na cidade (vem em
`features` da sessão), "Minhas triagens" ganha o botão **Meu histórico**. Par
validado: resumo de saúde (problemas ativos, medicamentos em uso, exames
pedidos nos últimos 12 meses, últimas medidas), documentos com o código de
conferência e o **Baixar PDF**, e a linha do tempo. Par só declarado: as
próprias triagens e agendamentos, com o convite para validar no posto. Fora do
modo `record`, o aviso de histórico parcial. Nunca texto clínico livre, CID
sem nome ou número SNCR; o PDF é baixado com o nome `documento-<código>.pdf`.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/wpda/.claude/mod20 && npx vitest run src/modules/citizen && npx tsc --noEmit`
Expected: PASS (os testes antigos do `Flow` e do `HistoryStep` seguem: sem `onRecord` nada muda) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda/.claude/mod20
/opt/homebrew/bin/git add src/modules/citizen/HistoryStep.tsx src/modules/citizen/Flow.tsx \
  src/modules/citizen/Flow.history.test.tsx README.md
/opt/homebrew/bin/git commit -m "feat: open the citizen history from my triages when the timeline is on

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–4 na branch `feat/mod-20-timeline`; o api do módulo 20 na **3040** com a semente do módulo 20.
- Produces: branch pronta para merge (depois do merge do api do módulo 20).

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd apps/wpda/.claude/mod20 && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base anotada na Task 0 mais **4 arquivos de teste novos**. Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
/opt/homebrew/bin/git -C apps/wpda/.claude/mod20 status --short
/opt/homebrew/bin/git -C apps/wpda/.claude/mod20 log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e contagem `0`.

- [ ] **Step 2: Nada vaza**

```bash
cd apps/wpda/.claude/mod20
grep -n "console\.\|localStorage\|sessionStorage" src/lib/history.ts src/lib/sessionFeatures.ts src/modules/citizen/MyHistoryStep.tsx
sed -n '/─── Meu histórico (módulo 20/,/^export interface CitizenSession/p' src/lib/citizenApi.ts | grep -n "cpf\|phone"
```

Expected: os dois sem saída.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do wpda contra a spec (§2–§4, §8, §9, §12 F-20.4 e F-20.5), o ADR 0035 e o contrato (§2, §4). Pontos de atenção:
- par declarado: nenhum pedido de resumo nem de documentos; o convite leva ao código do posto;
- a tela só mostra os campos do `<citizen_entry>` e do resumo: nada de texto clínico livre, CID sem nome, número SNCR, CPF na tela nova;
- o PDF é lido com `fetch`, salvo por `blob:` com nome sem dado pessoal; 404 diz que o documento sumiu; 401 volta ao login;
- "Carregar mais" sem repetir, com o cursor como veio; resumo 404 sem erro; parcial com o aviso;
- sem `features` na sessão, "Minhas triagens" é a de hoje; toda URL só com ids, tipos, cursor e `limit`;
- interface do canal: alvos de 48 px, texto de 18 px, componentes existentes.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do módulo 20 na porta **3040** (plano do api: semente com Curitiba em `record`, `patient_timeline`, `clinical_record` e `clinical_documents` ligados; um celular com duas pessoas — uma **validada** com consulta, documentos (um digital assinado e um de papel), escuta, agendamento e campanha, e uma **só declarada** com triagem e agendamento; e uma cidade em `integrated`). O Vite do worktree sobe num container da rede do compose:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose run -d --rm --no-deps --name wpda-mod20 -p 5191:5191 \
  -w /app/.claude/mod20 -e VITE_API_PROXY_TARGET=http://api:3040 \
  wpda npx vite --port 5191 --host 0.0.0.0
docker logs -f wpda-mod20   # espere o "ready"; Ctrl+C sai do log, o container segue
```

(Se o plano do api subir o servidor do módulo 20 num container próprio, use o nome dele no lugar de `api`.)

Abra `http://curitiba.localhost:5191/wpda/` (no celular de verdade ou no modo de dispositivo do navegador, 375 px). O usuário entra com o celular da semente e o código do SMS de dev, como nas provas do canal web. Confira com screenshot:
- pessoa **validada** → "Ver triagens de …" → **Meu histórico**: o resumo (problemas, medicamentos, exames dos últimos 12 meses, pressão/peso/glicemia com data), **Documentos** com o código de conferência, "Conferir o documento …" (abre a página pública de conferência do 19c) e **Baixar PDF** → o arquivo `documento-<código>.pdf` com o PDF assinado; o documento de papel com o PDF do papel; a **Linha do tempo** em ordem decrescente, com a consulta como "Consulta com <profissional>" (sem texto da consulta), o acolhimento como "acolhimento realizado", a receita de controle especial sem número SNCR;
- "Carregar mais" (se a semente tiver mais de 20 entradas) sem repetir;
- cancelar um documento no dashboard e tocar em "Baixar PDF" dele aqui sem recarregar → "Este documento não está mais disponível…";
- pessoa **só declarada** → Meu histórico: só triagens e agendamentos, o convite e "Validar no posto" → o código do posto;
- na cidade **integrada** (`http://<slug>.localhost:5191/wpda/`): o aviso de histórico parcial e nenhuma seção de resumo;
- no maintenance, desligar `patient_timeline` em Curitiba e sair/entrar de novo: "Minhas triagens" sem o botão.

Depois: `docker rm -f wpda-mod20`. Religue o que a prova desligou e desfaça o cancelamento de teste só se o plano do api tiver como (senão, registre o documento cancelado na semente de dev).

- [ ] **Step 5:** **Pare.** O merge do wpda só vem depois do merge do api do módulo 20, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: api → dashboard e wpda → maintenance (contrato §8). Antes do push, confira `origin/main..main` no wpda e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda
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

1. **W1 — A pessoa nas rotas do cidadão.** O contrato §4 tem `GET /citizen/history` e `GET /citizen/health_summary` sem dizer de quem: a sessão do wpda é do **celular**, que pode ter até 10 pessoas (pares), cada uma validada ou não (ADR 0017). Proposta — a mesma do plano do api (Desvio 8 / Divergência D2 dele), no molde de `GET /citizen/triages?citizen_id=`: `GET /citizen/history?citizen_id=` e `GET /citizen/health_summary?citizen_id=` (404 `not_found` se a pessoa não é do celular da sessão). O PDF fica `GET /citizen/documents/:id/pdf`, e o api confere que o documento é da pessoa de algum par **validado** do celular da sessão.
2. **W2 — Interruptores na sessão do cidadão.** O contrato não diz como o wpda sabe que `patient_timeline` está ligado; sem isso o botão apareceria em toda cidade e levaria a um 403. Proposta: `GET /citizen/session` (e o `POST /citizen/session`) devolvem `features: string[]` com os interruptores **ligados e utilizáveis** que o cidadão usa (hoje só `patient_timeline`), como a sessão do dashboard (session-v1.1.0). Ausente = nenhum.
3. **W3 — Filtro por tipo e tamanho da página no cidadão.** A seção "Documentos" precisa listar só os documentos sem varrer a linha do tempo inteira. Proposta: `kinds[]` também na rota do cidadão (mesmos valores do §1; o api ignora tipo que o par não pode ver), e `limit` aceito de 1 a 100 (o wpda usa 20, por ser celular).
4. **W4 — Campos ausentes no `<citizen_entry>`.** O contrato marca `document` como opcional (`document?`). Proposta: o api sempre manda a chave (`null` quando não é documento) e `downloadable: false` quando não houver PDF (documento digital ainda sem assinatura e que não voltou ao papel); `verification_url` nulo na Notificação de papel. A tela já normaliza a ausência (Task 1).
5. **W5 — Resumo para par validado sem paciente.** No modo `integrated` não há paciente do 19a: o contrato já dá 404 `not_available`. Fica registrado que a tela trata isso como "sem resumo", sem erro (Task 3).

## Self-review

- **Cobertura (spec §2 decisões 2 e 3, §3, §4, §8 "wpda", §9; contrato §2, §4; F-20.4 e F-20.5):** área "Meu histórico" — Tasks 3 e 4; par validado com resumo clínico, linha do tempo e documentos com download do PDF e código de conferência — Tasks 1–3; par declarado com triagens e agendamentos e o convite para validar — Task 3 (e Task 4, até o código do posto); histórico parcial — Task 3; interruptor — Tasks 1 e 4; Vite 5191 — Task 5.
- **Placeholders:** nenhum código faltando. A única condicional é o formato do código no `CounterCodeStep` (Task 4, Step 1), que já existe no repositório e o executor confere no teste dele; as variáveis de ambiente (cidade integrada e pessoas da semente, nome do container) são do plano do api.
- **Tipos:** `CitizenEntry`, `CitizenHistory`, `CitizenDocumentRef`, `HealthSummary`, `HealthVitals` (Task 1) são os usados nas Tasks 2–4; `citizenApi.history(id, { before?, kinds? })` tem a mesma forma no cliente, na tela e nos testes; `DOCUMENT_KINDS` é o filtro da seção Documentos nos dois lados; `hasSessionFeature`/`setSessionFeatures` (Task 1) são os do `Flow` e do teste (Task 4); `onRecord` é opcional no `HistoryStep` para os testes antigos seguirem iguais.
- **Review Focus:** 1 — Task 3 ("par declarado: …"); 2 — Task 1 ("documentPdf: …"), Task 2 ("nome do arquivo: …") e Task 3 ("baixar o PDF…", "documento que sumiu…"); 3 — Task 2 ("mergeEntries…") e Task 3 ("carregar mais…"); 4 — Task 3 ("resumo indisponível (404)…", "histórico parcial…"); 5 — Task 1 ("sessão: interruptores…"), Task 3 ("interruptor desligado no meio…", "tipo desconhecido vira Registro") e Task 4 ("sem patient_timeline na sessão…").
