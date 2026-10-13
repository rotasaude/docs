# Ciclo de interface — subprojeto 6, fatia 6b: fila e chamada — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Corrigir na tela Atendimento do dashboard (`#/attendance`) os achados FC-01 a FC-19 do percurso "fila e chamada", usando as peças da fatia 6a. FC-09 (nome do protocolo) e FC-18 (escuta em curso) viram cards.

**Architecture:** Primeiro, funções puras novas em `src/lib`: rótulos de chegada, espera, prazo, prioridade e dica de motivo. Depois, os componentes da tela são ajustados um por um:
- `UnitQueue`
- `CheckIn` e `Counter`
- `Requests` e `BookPanel`
- `UnitPicker`, `Attendance` e `Breadcrumb`

Só o dashboard muda; o admin não tem essa tela.

**Tech Stack:** React 18, Vite 5, TypeScript, Vitest 2, Testing Library e Playwright (capturas).

**Spec:** `docs/superpowers/specs/2026-10-12-interface-usability-design.md` (§4.1; padrões da §3). Os achados com captura estão em `rota-saude/evidencias/2026-10-12-usabilidade/fila-chamada/achados.md`.

## Global Constraints

- **Escopo:** só front-end, sem mudar regra. Nada de API, dados, permissões ou regra de negócio.
- **Testes existentes:**
  - a suíte existente passa;
  - uma asserção só muda quando o comportamento muda de propósito (por exemplo, "Saiu sem atendimento" passar a pedir confirmação), sempre com uma linha `Ruling:` no ledger que cite o FC;
  - todo FC corrigido ganha um teste que falha antes da correção.
- **Peças da 6a**, de `src/components` e `src/lib`:
  - `ConfirmButton`, `ActionButton`, `ActionMessage` + `useActionMessage`;
  - `useReveal`, `formStyles` (`dangerButtonStyle`).
- **Bloqueio por regra:** motivo com menos de 10 caracteres ou documento não conferido mantém `disabled` e ganha a dica visível (`ReasonHint`). Ação em andamento usa `aria-disabled`, que preserva o foco.
- **Âncoras da página:** nada de `href="#…"`, porque o hash é do roteador. Use botão com `scrollIntoView` + foco.
- **Cores:** só por papel do tema. Nenhum `role="status"`/`alert` novo além dos de mensagem.
- **Commits e git:** Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Use `/opt/homebrew/bin/git` e caminhos explícitos no `add`.
- **Worktree:** `rota-saude/.worktrees/dashboard-ui6b`, no branch `ui/usability-6b` a partir de `origin/main`, com `npm ci`.
- **Comandos de verificação:** `npx vitest run`, `npx tsc --noEmit` e `npm run build`.

## Review Focus

1. **Duas pessoas em atendimento, a consulta aberta é de uma delas, e a outra é encerrada.**
   - Esperado: o painel continua aberto e com o nome certo.
   - Onde se prova: a Task 2 testa o fechamento só quando a pessoa do painel sai.
2. **A recepção confirma "Saiu sem atendimento" e outra pessoa já tinha chamado a mesma linha (`already_closed`).**
   - Esperado: a fila recarrega, e a mensagem não diz que saiu.
   - Onde se prova: Task 2.
3. **Pessoa chegou ontem às 23h50 e hoje são 00h10.**
   - Esperado: "ontem 23:50" e o selo "chegou ontem", no fuso da cidade.
   - Onde se prova: Task 1.
4. **CPF digitado no modo código e troca para "Cidadão sem o código".**
   - Esperado: o CPF continua lá, e o erro antigo some ao editar.
   - Onde se prova: Task 4.
5. **Lista com 24 pedidos, a recepção abre "Marcar horário" do 18º.**
   - Esperado: o painel diz de quem é e a tela rola até ele, mesmo com a lista recolhida e expandida depois.
   - Onde se prova: Task 5.

---

## File Structure (`apps/dashboard/src/`)

| Arquivo | Responsabilidade |
|---|---|
| `lib/queueLabels.ts` (+ teste) | `arrivalLabel`, `staleArrival`, `priorityLabel`, `personLabel` |
| `lib/screening.ts` | `waitLabel` acima de 24 h em dias |
| `lib/scheduling.ts` | `fmtDueOn` com ano fora do ano corrente |
| `components/ReasonHint.tsx` (+ teste) | "mínimo de 10 caracteres — faltam N", ligado por `aria-describedby` |
| `components/datatable.css` | Célula vazia some no modo cartão (FC-15) |
| `modules/attendance/UnitQueue.tsx` (+ testes) | FC-01 a FC-05, FC-07, FC-08, FC-13, FC-15, FC-19 |
| `modules/attendance/CheckIn.tsx` (+ teste) | FC-05, FC-06, FC-12, FC-17 |
| `modules/Attendance.tsx` (+ teste) | `Counter` (FC-06, FC-12, FC-17), `RevokePanel` (FC-12), índice e ordem (FC-10) |
| `modules/attendance/Requests.tsx`, `BookPanel.tsx`, `ErasureRequest.tsx` (+ testes) | FC-10, FC-11, FC-12 |
| `modules/attendance/UnitPicker.tsx`, `shell/Breadcrumb.tsx` (+ testes) | FC-16 |

---

### Task 0: Preparação

- [ ] **Step 1: Worktree e linha de base.**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q && $G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-ui6b -b ui/usability-6b origin/main
cd $R/.worktrees/dashboard-ui6b && npm ci --silent && npx vitest run 2>&1 | tail -3
```

  Expected: verde. Anote a contagem no ledger.

- [ ] **Step 2: Capturas "antes".** Reaproveite os scripts do percurso em `scratchpad/u6-fila/`:
  - `lib.mjs` e `auth.mjs` (sessões `recepcao` e `profissional`);
  - `a-recepcao.mjs` e `b-prof.mjs`.

  Copie-os para `scratchpad/u6b/`, troque a base por argumento e a pasta de saída para `evidencias/2026-10-12-usabilidade/6b/antes/`. Rode contra `http://curitiba.localhost:5175`. As capturas cobrem:
  - a escolha da unidade;
  - a recepção com fila, pedidos e balcão, em 1280 e 768;
  - o profissional com a fila e o painel "Encerrar" aberto.

---

### Task 1: Rótulos da fila (FC-07, FC-08, FC-09 rótulo, FC-10 prazo)

**Files:** `lib/queueLabels.ts`, `lib/queueLabels.test.ts`, `lib/screening.ts`, `lib/screening.test.ts`, `lib/scheduling.ts`, `lib/scheduling.test.ts`.

**Interfaces — Produces:**
- `arrivalLabel(iso: string, now: Date): string`:
  - hoje: "HH:MM";
  - ontem: "ontem HH:MM";
  - antes disso: "dd/mm HH:MM".
- `staleArrival(iso: string, now: Date): string | null`:
  - hoje: `null`;
  - ontem: "chegou ontem";
  - antes disso: "há N dias".
- `priorityLabel(p: number | null | undefined): string`:
  - número: "prioridade N";
  - nulo: "—".
- `personLabel(r: { display_name?: string | null; cpf_masked: string }): { title: string; sub: string | null }`:
  - com nome: `{ title: display_name, sub: cpf_masked }`;
  - sem nome: `{ title: cpf_masked, sub: null }`.
- `waitLabel(minutes)`:
  - abaixo de 1440 min: igual a hoje;
  - a partir de 1440 min: "N d M h".
- `fmtDueOn(iso, today = cityIsoDate(new Date()))`:
  - mesmo ano: "até dd/mm";
  - outro ano: "até dd/mm/aaaa".

O dia de referência é o do fuso da cidade. Use os helpers que já existem: `cityIsoDate` em `lib/scheduling.ts` e `cityDateFormat` em `lib/format.ts`. Leia esses dois antes de escrever.

- [ ] **Step 1: Escrever os testes, que vão falhar.**

`lib/queueLabels.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { arrivalLabel, personLabel, priorityLabel, staleArrival } from "./queueLabels";

// Fuso da cidade em dev e teste: America/Sao_Paulo (UTC-3).
const now = new Date("2026-10-12T03:10:00Z"); // 00:10 de 12/10 em São Paulo

describe("chegada na fila", () => {
  it("hoje mostra só a hora; ontem diz ontem; antes, dia e hora", () => {
    expect(arrivalLabel("2026-10-12T03:05:00Z", now)).toBe("00:05");
    expect(arrivalLabel("2026-10-12T02:50:00Z", now)).toBe("ontem 23:50");
    expect(arrivalLabel("2026-10-09T14:00:00Z", now)).toBe("09/10 11:00");
  });

  it("chegada de outro dia ganha selo; hoje não", () => {
    expect(staleArrival("2026-10-12T03:05:00Z", now)).toBeNull();
    expect(staleArrival("2026-10-12T02:50:00Z", now)).toBe("chegou ontem");
    expect(staleArrival("2026-10-09T14:00:00Z", now)).toBe("há 3 dias");
  });
});

describe("prioridade e pessoa", () => {
  it("prioridade com rótulo, como no Acolhimento", () => {
    expect(priorityLabel(1)).toBe("prioridade 1");
    expect(priorityLabel(null)).toBe("—");
  });
  it("nome primeiro, CPF mascarado embaixo; sem nome, o CPF", () => {
    expect(personLabel({ display_name: "Luíza Alves", cpf_masked: "***.343.399-**" })).toEqual({ title: "Luíza Alves", sub: "***.343.399-**" });
    expect(personLabel({ display_name: null, cpf_masked: "***.343.399-**" })).toEqual({ title: "***.343.399-**", sub: null });
  });
});
```

  No fim do `describe` de `waitLabel`, em `lib/screening.test.ts`:

```ts
    expect(waitLabel(1439)).toBe("23 h 59 min");
    expect(waitLabel(4832)).toBe("3 d 8 h");
```

  Em `lib/scheduling.test.ts`:

```ts
it("prazo de outro ano mostra o ano (FC-10)", () => {
  expect(fmtDueOn("2026-11-11", "2026-10-12")).toBe("até 11/11");
  expect(fmtDueOn("2025-05-22", "2026-10-12")).toBe("até 22/05/2025");
});
```

  Run: `npx vitest run src/lib/queueLabels.test.ts src/lib/screening.test.ts src/lib/scheduling.test.ts`
  Expected: FAIL.

- [ ] **Step 2: Implementar.**

`lib/queueLabels.ts`:

```ts
// Rótulos da fila do posto (spec 6b, FC-07/08/09): chegada curta quando é de
// hoje, selo para quem está na fila desde outro dia, prioridade com nome e a
// pessoa pelo nome quando houver. Dia no fuso da cidade.
import { fmtHourMinute } from "./format";
import { cityIsoDate, ddmm } from "./scheduling";

const DAY_MS = 86_400_000;

function daysBetween(iso: string, now: Date): number {
  const a = Date.parse(`${cityIsoDate(new Date(iso))}T12:00:00Z`);
  const b = Date.parse(`${cityIsoDate(now)}T12:00:00Z`);
  return Math.round((b - a) / DAY_MS);
}

export function arrivalLabel(iso: string, now: Date): string {
  const days = daysBetween(iso, now);
  const hour = fmtHourMinute(iso);
  if (days <= 0) return hour;
  if (days === 1) return `ontem ${hour}`;
  return `${ddmm(cityIsoDate(new Date(iso)))} ${hour}`;
}

export function staleArrival(iso: string, now: Date): string | null {
  const days = daysBetween(iso, now);
  if (days <= 0) return null;
  return days === 1 ? "chegou ontem" : `há ${days} dias`;
}

export function priorityLabel(p: number | null | undefined): string {
  return p === null || p === undefined ? "—" : `prioridade ${p}`;
}

export function personLabel(r: { display_name?: string | null; cpf_masked: string }): { title: string; sub: string | null } {
  return r.display_name ? { title: r.display_name, sub: r.cpf_masked } : { title: r.cpf_masked, sub: null };
}
```

  - **`waitLabel`, em `lib/screening.ts`.** Antes do cálculo de horas, inclua `if (minutes >= 1440) { const d = Math.floor(minutes / 1440); return \`${d} d ${Math.floor((minutes % 1440) / 60)} h\`; }`.
  - **`fmtDueOn`, em `lib/scheduling.ts`.**

```ts
export function fmtDueOn(iso: string, today: string = cityIsoDate(new Date())): string {
  return iso.slice(0, 4) === today.slice(0, 4) ? `até ${ddmm(iso)}` : `até ${ddmm(iso)}/${iso.slice(0, 4)}`;
}
```

  Se `cityIsoDate` não for exportado de `lib/scheduling.ts`, exporte-o. Se ele estiver noutro módulo, importe de lá e registre no ledger.

  Run: `npx vitest run 2>&1 | tail -3`
  Expected: verde.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/lib/queueLabels.ts src/lib/queueLabels.test.ts src/lib/screening.ts src/lib/screening.test.ts src/lib/scheduling.ts src/lib/scheduling.test.ts
/opt/homebrew/bin/git commit -m "feat: readable arrival, wait, priority and due-date labels for the unit queue

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Fila — ações, mensagens e painéis (FC-01 a FC-05, FC-13, FC-19)

**Files:** `modules/attendance/UnitQueue.tsx`, `modules/attendance/UnitQueue.test.tsx` e `modules/attendance/UnitQueue.usability.test.tsx` (novo; copie o harness `renderQueue`/fixtures do `UnitQueue.test.tsx`, lendo-o antes).

**Interfaces — Consumes:** `ConfirmButton`, `ActionButton`, `ActionMessage`, `useActionMessage`, `useReveal`, `personLabel`.

- [ ] **Step 1: Escrever os testes, que vão falhar** (`UnitQueue.usability.test.tsx`).

  Use as fixtures do arquivo existente: duas linhas em `waiting` (`a1` e `a2`), mais `in_care` quando o teste precisar.

```tsx
it("FC-01: 'Saiu sem atendimento' pede confirmação nomeando a pessoa e só então encerra", async () => {
  renderQueue({ canCare: false });
  fireEvent.click((await screen.findAllByRole("button", { name: "Saiu sem atendimento" }))[0]);
  expect(api.closeAttendance).not.toHaveBeenCalled();
  expect(screen.getByRole("group", { name: /Confirmar que .* saiu sem atendimento\?/ })).toBeTruthy();
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  await waitFor(() => expect(api.closeAttendance).toHaveBeenCalledWith("a1", "left", undefined, undefined));
  expect((await screen.findByRole("status")).textContent).toMatch(/marcad[oa] como saiu sem atendimento/);
});

it("FC-02: a mensagem da ação anterior some quando outra ação começa", async () => {
  // 1ª ação: Saiu sem atendimento (gera status); 2ª: Chamar (sem mensagem própria)
  renderQueue({ canCare: true });
  fireEvent.click((await screen.findAllByRole("button", { name: "Saiu sem atendimento" }))[0]);
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  expect(await screen.findByRole("status")).toBeTruthy();
  fireEvent.click(screen.getAllByRole("button", { name: "Chamar" })[0]);
  await waitFor(() => expect(screen.queryByText(/saiu sem atendimento$/)).toBeNull());
});

it("FC-04/FC-13: Encerrar rola até o painel, foca o título com a pessoa; Escape fecha e devolve o foco", async () => {
  const scroll = vi.fn(); Element.prototype.scrollIntoView = scroll;
  renderQueue({ canCare: true, inCare: true });
  const opener = (await screen.findAllByRole("button", { name: "Encerrar" }))[0];
  opener.focus(); fireEvent.click(opener);
  const title = screen.getByRole("heading", { name: /Encerrar atendimento de/ });
  expect(scroll).toHaveBeenCalled();
  expect(document.activeElement).toBe(title);
  fireEvent.keyDown(title, { key: "Escape" });
  expect(screen.queryByRole("heading", { name: /Encerrar atendimento de/ })).toBeNull();
  expect(document.activeElement).toBe(opener);
});

it("FC-03: o painel da consulta diz de quem é e fecha quando ESSA pessoa sai de Em atendimento", async () => {
  // clinicalRecord ligado; duas pessoas em atendimento (c1, c2); abre a consulta de c1
  // e depois a fila recarrega sem c2 → continua aberto; recarrega sem c1 → fecha.
  // (monte com o mock de listUnitQueue trocando o retorno entre os passos e invalidando a query)
});

it("FC-05: Chamar mantém o foco no botão durante a chamada", async () => {
  let resolve!: (v: unknown) => void;
  mocked(api.callAttendance).mockReturnValue(new Promise((r) => { resolve = r; }));
  renderQueue({ canCare: true });
  const call = (await screen.findAllByRole("button", { name: "Chamar" }))[0];
  call.focus(); fireEvent.click(call);
  expect(call.getAttribute("aria-busy")).toBe("true");
  expect(document.activeElement).toBe(call);
  await act(async () => { resolve({ attendance: { id: "a1", screening: null } }); });
});

it("FC-19: a mesma pessoa não aparece duplicada em Aguardando nem também em Em atendimento", async () => {
  // listUnitQueue devolve a1 duas vezes em waiting e também em in_care
  // → Aguardando tem só a2; Em atendimento tem a1 uma vez; sem aviso de key no console
});
```

  O teste FC-03 tem só o roteiro, porque depende do harness. Ao escrevê-lo, siga o padrão de `UnitQueue.consultation.test.tsx`: leia o arquivo e copie o mock de `ConsultationWorkspace`, se houver. O teste FC-19 também: asserção de contagem de linhas mais `vi.spyOn(console, "error")` sem chamadas que contenham "same key".

  Run: `npx vitest run src/modules/attendance/UnitQueue.usability.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: Implementar em `UnitQueue.tsx`.**

  1. **Mensagens (FC-02).**
     - Troque `done`/`actionError` por `const msg = useActionMessage();`.
     - Toda ação começa com `msg.start()`.
     - Os textos de sucesso viram `msg.ok(...)`: "Reavaliação registrada.", "Consulta finalizada e atendimento encerrado." e "Pedido de agendamento criado na …".
     - Os erros viram `msg.fail(attendanceError(err))`.
     - Renderize `<ActionMessage message={msg.message} />` no lugar dos dois `<p>`.
     - Mantenha a exceção de hoje: o erro que é igual a `careBlocked` não aparece duas vezes. Use `msg.message?.text !== careBlocked` como guarda.
  2. **"Saiu sem atendimento" (FC-01).**
     - Troque o botão por `<ConfirmButton label="Saiu sem atendimento" variant="danger" question={\`Confirmar que ${personLabel(r).title} saiu sem atendimento?\`} onConfirm={() => onLeft(r)} />`.
     - Em `onLeft`, no sucesso: `msg.ok(\`${personLabel(row).title} marcado como saiu sem atendimento\`)`.
     - Com `already_closed`: só `invalidate()`, sem mensagem.
     - Ponha o `ConfirmButton` por último no grupo, separado dos outros botões por `marginLeft: 8`.
  3. **Chamar, Chamar próximo e Reavaliar (FC-05).** Viram `ActionButton variant="secondary"` (Chamar próximo fica `primary`), com `busy={rowBusy === r.id}` ou `busy={callingNext}`. Remova `disabled` e os estilos manuais.
  4. **`ClosePanel` (FC-04, FC-13).**
     - `const ref = useReveal<HTMLElement>(true);` no `<section ref={ref}>`.
     - O título passa a ser `<h3 style={{ margin: 0, fontSize: 14 }}>Encerrar atendimento de {personLabel(row).title}</h3>`.
     - `onKeyDown` na `section`: `if (e.key === "Escape" && !busy) onCancel()`.
     - "Confirmar encerramento" vira `ActionButton` com `disabledReason`. Quando `outcomeProblem(...)` não é nulo, o motivo é a mensagem que `outcomeProblem` já devolve. Leia `lib/outcome.ts` para confirmar se ele devolve string.
     - No `UnitQueue`, guarde o botão que abriu: `const closeOpener = useRef<HTMLElement | null>(null)`. No clique de "Encerrar", faça `closeOpener.current = e.currentTarget`. Em `onCancel`, `setClosing(null); closeOpener.current?.focus()`. Como o `focus` acontece depois do desmonte, use `queueMicrotask`.
  5. **Painel da consulta (FC-03).**
     - Envolva o `ConsultationWorkspace` em `<section ref={revealRef} aria-labelledby="consulta-titulo">`, com `<h3 id="consulta-titulo">Consulta de {personLabel(rowDaConsulta).title} · chamada às {fmtHourMinute(row.called_at)}</h3>`. A linha vem de `inCare.find(...)`.
     - Feche quando a pessoa sai:

```ts
useEffect(() => {
  if (panel?.kind === "consultation" && query.isSuccess && !inCare.some((r) => r.id === panel.attendanceId)) setPanel(null);
}, [ panel, query.isSuccess, inCare ]);
```

     - `revealRef = useReveal<HTMLElement>(panel?.kind === "consultation")`.
  6. **Duplicatas (FC-19).** Antes de renderizar:

```ts
const inCareIds = new Set(inCare.map((r) => r.id));
const uniq = (rows: QueueRow[]) => rows.filter((r, i) => rows.findIndex((x) => x.id === r.id) === i);
const waiting = uniq((query.data?.waiting ?? []).filter((r) => !inCareIds.has(r.id)));
const inCare = uniq(query.data?.in_care ?? []);
```

     `inCare` precisa ser declarado antes de `waiting`.

  Run: `npx vitest run src/modules/attendance 2>&1 | tail -5`
  Expected:
  - os testes novos passam;
  - em `UnitQueue.test.tsx`, os casos "'Saiu sem atendimento' … encerra com 'left'" (l. 337) e, se houver, os que procuram `role="status"` de "Pedido de agendamento criado" mudam de propósito. O primeiro passa a clicar "Confirmar" depois de "Saiu sem atendimento". Registre um `Ruling` para cada um.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/modules/attendance/UnitQueue.tsx src/modules/attendance/UnitQueue.test.tsx src/modules/attendance/UnitQueue.usability.test.tsx
/opt/homebrew/bin/git commit -m "feat: confirm 'left without care', keep messages and focus next to queue actions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Fila — colunas e cartões (FC-07, FC-08, FC-09 rótulo, FC-15)

**Files:** `modules/attendance/UnitQueue.tsx`, `modules/attendance/UnitQueue.usability.test.tsx` e `components/datatable.css`.

- [ ] **Step 1: Escrever os testes, que vão falhar.**

```tsx
it("FC-07/FC-15: coluna Pessoa com nome e CPF; Origem vazia não vira '—'", async () => {
  renderQueue({ canCare: false, clinicalRecord: true }); // fixture a1 com display_name "Luíza Alves"
  const row = (await screen.findAllByRole("row")).find((r) => r.textContent?.includes("Luíza Alves"))!;
  expect(within(row).getByText("***.343.399-**")).toBeTruthy();
  expect(row.querySelector('[data-label="Origem"]')?.textContent).toBe("");
});

it("FC-08: quem chegou em outro dia tem selo e a espera em dias", async () => {
  // fixture com checked_in_at 3 dias antes de `now` e waited_minutes 4832
  renderQueue({ canCare: false, now: () => new Date("2026-10-12T15:00:00Z") });
  expect(await screen.findByText("há 3 dias")).toBeTruthy();
  expect(screen.getByText("3 d 8 h")).toBeTruthy();
  expect(screen.getByText(/1 pessoa na fila desde dias anteriores/)).toBeTruthy();
});

it("FC-09: prioridade com rótulo", async () => {
  renderQueue({ canCare: false });
  expect((await screen.findAllByText("prioridade 1")).length).toBeGreaterThan(0);
});
```

  Em `usability-guards.test.ts`, acrescente ao `describe` da tabela:

```ts
it("célula vazia some no modo cartão (FC-15)", () => {
  expect(css).toMatch(/\.dt-row > \.dt-cell:empty\s*\{\s*display:\s*none/);
});
```

  Run: `npx vitest run src/modules/attendance/UnitQueue.usability.test.tsx src/theme/usability-guards.test.ts`
  Expected: FAIL.

- [ ] **Step 2: Implementar.**
  - **Pessoa.** Nas duas tabelas, troque as colunas `CPF` e `nameCol` por uma só:

```tsx
{ label: "Pessoa", w: "1.6fr", render: (r) => {
  const p = personLabel(r);
  return <span style={{ display: "flex", flexDirection: "column" }}><strong style={{ fontWeight: 600 }}>{p.title}</strong>
    {p.sub && <span className="mono" style={{ fontSize: 11, color: "var(--ink2)" }}>{p.sub}</span>}</span>;
} }
```

  - **Chegada.** `arrivalLabel(r.checked_in_at, at)`.
  - **Espera.**

```tsx
(r) => { const stale = staleArrival(r.checked_in_at, at); const w = waitLabel(r.screening?.waited_minutes ?? waitedMinutes(r.checked_in_at, at));
  return stale ? <span style={{ display: "flex", flexDirection: "column", gap: 2 }}>{w}<Tag tone="warn">{stale}</Tag></span> : w; }
```

  - **Prioridade.** `priorityLabel(r.priority)` nas duas tabelas.
  - **Origem.** Devolve `null` quando não é agendamento.
  - **Aviso de dias anteriores.** Acima da tabela de Aguardando, quando `n = waiting.filter((r) => staleArrival(r.checked_in_at, at)).length` for maior que zero:

```tsx
<p style={{ margin: 0, fontSize: 12.5, color: "var(--warn)" }}>{n === 1 ? "1 pessoa" : `${n} pessoas`} na fila desde dias anteriores</p>
```

  - **`datatable.css`.** Dentro do `@media`, inclua `.dt-row > .dt-cell:empty { display: none; }`.
  - **Testes existentes.** Os que procuram a coluna `CPF` por texto continuam passando, porque o CPF segue visível. Se algum procurar o cabeçalho "CPF", registre um `Ruling` e troque por "Pessoa".

  Run: `npx vitest run 2>&1 | tail -3`
  Expected: verde.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/modules/attendance/UnitQueue.tsx src/modules/attendance/UnitQueue.usability.test.tsx src/modules/attendance/UnitQueue.test.tsx src/components/datatable.css src/theme/usability-guards.test.ts
/opt/homebrew/bin/git commit -m "feat: person column, stale-arrival flag and readable wait in the unit queue

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Check-in e Balcão (FC-05, FC-06, FC-12, FC-17)

**Files:** `components/ReasonHint.tsx`, `components/ReasonHint.test.tsx`, `modules/attendance/CheckIn.tsx`, `modules/attendance/CheckIn.test.tsx`, `modules/Attendance.tsx` e o teste do `Counter` (ache com `grep -ln "Buscar validação" src/modules/*.test.tsx`).

**Interfaces — Produces:**
- `reasonHint(text: string, min = 10): string | null`:
  - `null` quando `text.trim().length >= min`;
  - "mínimo de 10 caracteres — faltam N" no resto.
- `ReasonHint({ id, text, min }: { id: string; text: string; min?: number })` renderiza `<p id>` com a dica ou não renderiza nada.

- [ ] **Step 1: Escrever os testes, que vão falhar.**

`components/ReasonHint.test.tsx`:

```tsx
import { afterEach, describe, expect, it } from "vitest";
import { cleanup, render, screen } from "@testing-library/react";
import { ReasonHint, reasonHint } from "./ReasonHint";

afterEach(cleanup);

describe("ReasonHint (FC-12)", () => {
  it("diz quanto falta e some quando basta", () => {
    expect(reasonHint("faltou")).toBe("mínimo de 10 caracteres — faltam 4");
    expect(reasonHint("documento perdido")).toBeNull();
    render(<ReasonHint id="h" text="curto" />);
    expect(screen.getByText("mínimo de 10 caracteres — faltam 5").id).toBe("h");
  });
});
```

  Em `CheckIn.test.tsx`, um `describe` novo, usando o `renderCheckIn` e os mocks do arquivo:

```tsx
describe("usabilidade do check-in (6b)", () => {
  it("FC-06: Enter no código envia a busca", async () => {
    mocked(api.lookupCheckIn).mockRejectedValue(new ApiError(422, { error: "invalid_code" }));
    renderCheckIn();
    fireEvent.change(screen.getByLabelText("CPF do cidadão (check-in)"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Código de check-in"), { target: { value: "123456" } });
    fireEvent.submit(screen.getByLabelText("Código de check-in").closest("form")!);
    await waitFor(() => expect(api.lookupCheckIn).toHaveBeenCalled());
  });

  it("FC-06: o erro some ao editar o CPF, e o CPF segue ao trocar de modo", async () => {
    renderCheckIn();
    fireEvent.change(screen.getByLabelText("CPF do cidadão (check-in)"), { target: { value: "111" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar check-in" }));
    expect(screen.getByRole("alert").textContent).toBe("CPF inválido");
    fireEvent.change(screen.getByLabelText("CPF do cidadão (check-in)"), { target: { value: "529982247" } });
    expect(screen.queryByRole("alert")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Cidadão sem o código" }));
    expect((screen.getByLabelText("CPF do cidadão (exceção)") as HTMLInputElement).value).toBe("529.982.247");
  });

  it("FC-17: código que não confere sugere o outro painel", async () => {
    mocked(api.lookupCheckIn).mockRejectedValue(new ApiError(422, { error: "invalid_code" }));
    renderCheckIn();
    fireEvent.change(screen.getByLabelText("CPF do cidadão (check-in)"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Código de check-in"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar check-in" }));
    expect((await screen.findByRole("alert")).textContent).toMatch(/pode ser de 'Validar no posto'/);
  });

  it("FC-12: Motivo da exceção mostra quanto falta, ligado ao botão", async () => {
    // mesmo caminho do teste existente (l. ~205) até selecionar a linha
    // → Motivo "curto": o botão "Iniciar atendimento" tem aria-describedby apontando para "mínimo de 10 caracteres — faltam 5"
  });
});
```

  - Antes de escrever a mensagem do FC-17, confira em `lib/attendance.ts` (`attendanceError`) o texto e o código de "código não confere". A sugestão é acrescentada só no check-in e no balcão.
  - O teste FC-12 tem só o roteiro, porque segue o teste existente. Escreva-o seguindo esse teste.
  - No teste do `Counter`, faça os mesmos três casos com "CPF do cidadão (validação)", "Código de validação" e "Buscar validação". A sugestão do balcão é "pode ser de 'Cheguei na unidade'".

  Run: `npx vitest run src/components/ReasonHint.test.tsx src/modules/attendance/CheckIn.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: Implementar.**
  - **`components/ReasonHint.tsx`.**

```tsx
// Dica do motivo mínimo (spec 6b, FC-12): o botão bloqueado diz o que falta.
export function reasonHint(text: string, min = 10): string | null {
  const missing = min - text.trim().length;
  return missing > 0 ? `mínimo de ${min} caracteres — faltam ${missing}` : null;
}

export function ReasonHint({ id, text, min = 10 }: { id: string; text: string; min?: number }) {
  const hint = reasonHint(text, min);
  return hint ? <p id={id} style={{ margin: 0, fontSize: 12, color: "var(--ink2)" }}>{hint}</p> : null;
}
```

  - **`CheckIn.tsx`:**
    - **CPF compartilhado.** `const [ cpf, setCpf ] = useState("")` sobe para `CheckIn` e desce como `cpf`/`onCpf` para `CodeFlow` e `ExceptionFlow`. Os dois `reset` deixam de limpar o CPF; só "Próximo atendimento" limpa. Para isso, passe `onCpf("")` no reset do estado "done".
    - **Formulário.** Envolva os campos e o botão de busca de cada fluxo num `<form onSubmit={(e) => { e.preventDefault(); void search(); }}>`. O botão vira `type="submit"`, com `aria-disabled={busy || undefined}` no lugar de `disabled`. `search` já ignora o clique quando `busy`.
    - **Erro.** No `onChange` do CPF e do código, chame `setError(null)`.
    - **Sugestão (FC-17).** No `catch` de `search`, o texto passa por `suggestOther(msg, "Validar no posto")`:

```ts
const suggestOther = (msg: string, other: string) =>
  /código não confere/.test(msg) ? `${msg}. Este código pode ser de '${other}' — use o outro painel.` : msg;
```

      Ajuste a regex ao texto real de `attendanceError`.
    - **Títulos (FC-17).** O `Panel` passa a "Check-in de quem chegou", com `sub` "código de 'Cheguei na unidade', no celular do cidadão".
    - **Motivo da exceção (FC-12).** Abaixo do textarea, `<ReasonHint id="exception-reason-hint" text={reason} />`. O botão "Iniciar atendimento" ganha `aria-describedby="exception-reason-hint"`.
  - **`Attendance.tsx`:**
    - **`Counter`.** Recebe o mesmo tratamento: formulário, limpeza do erro, sugestão com "Cheguei na unidade" e o título "Validar cadastro com documento", com `sub` "código de 'Validar no posto', no celular do cidadão". Ele não troca de modo, então não há CPF a compartilhar.
    - **Caixa "Conferi o documento" (FC-12).** Quando não está marcada, mostre ao lado do botão "Validar cadastro" o motivo "marque 'Conferi o documento' para validar". Use um `<p id="counter-block">` ligado por `aria-describedby`, só quando `!checked`.
    - **`RevokePanel`.** Acrescente `<ReasonHint id="revoke-reason-hint" text={reason} />` e o `aria-describedby` no botão.
  - **Testes que procuram os títulos antigos.** Se algum procurar "Balcão" ou "Check-in" pelo título do painel, ele muda de propósito: registre um `Ruling`.

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit`
  Expected: verde.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/components/ReasonHint.tsx src/components/ReasonHint.test.tsx src/modules/attendance/CheckIn.tsx src/modules/attendance/CheckIn.test.tsx src/modules/Attendance.tsx
/opt/homebrew/bin/git commit -m "feat: submit check-in on Enter, keep the CPF, explain blocked reasons and tell check-in from validation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

  Inclua no `add` o teste do `Counter`, se ele estiver em outro arquivo.

---

### Task 5: Pedidos, marcar horário e ordem da página (FC-10, FC-11, FC-12, FC-16)

**Files:**
- `modules/attendance/Requests.tsx` e `Requests.test.tsx`;
- `modules/attendance/BookPanel.tsx` e `BookPanel.test.tsx`;
- `modules/attendance/ErasureRequest.tsx` e `ErasureRequest.test.tsx`;
- `modules/Attendance.tsx`;
- `modules/attendance/UnitPicker.tsx` e o teste dele;
- `shell/Breadcrumb.tsx` e `shell/Breadcrumb.test.tsx`.

- [ ] **Step 1: Escrever os testes, que vão falhar.**

```tsx
// Requests.test.tsx
it("FC-10: com mais de 5 pedidos, mostra 5 e 'ver todos (N)'", async () => {
  // listUnitRequests devolve 7 linhas
  expect((await screen.findAllByRole("button", { name: "Marcar horário" })).length).toBe(5);
  fireEvent.click(screen.getByRole("button", { name: "ver todos (7)" }));
  expect(screen.getAllByRole("button", { name: "Marcar horário" }).length).toBe(7);
});

it("FC-10: coluna Atendimento vazia quando repete o Pedido", async () => {
  // linha com requestKindLabel(r) === r.appointment_type_name ("Retorno")
  // → a célula data-label="Atendimento" tem textContent ""
});

it("FC-12: Justificativa do encerramento mostra quanto falta", async () => {
  // abrir "Encerrar pedido", digitar "curto" → texto "mínimo de 10 caracteres — faltam 5" visível
});

// BookPanel.test.tsx
it("FC-11: o painel diz para quem é e rola até ele", () => {
  const scroll = vi.fn(); Element.prototype.scrollIntoView = scroll;
  // render com row.cpf_masked "***.343.399-**"
  expect(screen.getByRole("heading", { name: /Marcar horário para \*\*\*\.343\.399-\*\*/ })).toBeTruthy();
  expect(scroll).toHaveBeenCalled();
});

// UnitPicker teste
it("FC-16: o painel se chama Unidade e explica o que libera", async () => {
  // sem unidade guardada: texto "Escolha a unidade para ver o check-in e a fila"
  // com unidade escolhida: Panel "Unidade" (não "Atendente")
});

// Breadcrumb.test.tsx
it("FC-16: grupo com o mesmo nome da tela não se repete", () => {
  // groups: [{ label: "Atendimento", items: [attendance("Atendimento"), my-agenda] }], rota attendance
  // → o texto da trilha não contém "Atendimento›Atendimento" (o link do grupo some quando label === título)
});
```

  - **Attendance.** No teste da página (ache com `grep -ln "Attendance" src/modules/Attendance*.test.tsx`), escreva o FC-10 assim: para quem tem `citizen_verifier`, a ordem dos títulos de painel é Unidade, Check-in de quem chegou, Validar cadastro com documento, Fila, Pedidos de agendamento, Agenda. Existe um índice "Nesta página" com botões, e clicar em "Pedidos" chama `scrollIntoView` no painel de pedidos.
  - **Roteiros.** Os testes acima que têm só o roteiro seguem os harnesses de cada arquivo. Leia cada um antes de escrever e mantenha as asserções exatamente como descritas.

  Run: `npx vitest run src/modules/attendance src/modules/Attendance* src/shell/Breadcrumb.test.tsx`
  Expected: FAIL nos novos.

- [ ] **Step 2: Implementar.**
  - **`Requests.tsx`:**
    - **Lista recolhida.** `const [ all, setAll ] = useState(false); const shown = all ? rows : rows.slice(0, 5);`. Abaixo da tabela, quando `rows.length > 5 && !all`, um `<button style={secondaryButtonStyle} onClick={() => setAll(true)}>ver todos ({rows.length})</button>`.
    - **Coluna Atendimento.** Devolve `null` quando `r.appointment_type_name === requestKindLabel(r)`.
    - **Justificativa.** No `DismissPanel`, `<ReasonHint id="dismiss-reason-hint" text={reason} />` e o `aria-describedby` no botão.
    - **Abrir pedido escondido.** Ao abrir "Marcar horário" de uma linha escondida, nada muda, porque o botão só existe nas linhas mostradas.
  - **`BookPanel.tsx`.**
    - O `<strong>` do título vira `<h3 style={{ margin: 0, fontSize: 14 }}>{\`Marcar horário para ${row.cpf_masked} — ${row.appointment_type_name} · prazo ${fmtDueOn(row.due_on)}\`}</h3>`.
    - A `section` usa `ref={useReveal<HTMLElement>(true)}`.
    - Confira em `lib/api.ts` se `RequestRow` tem `cpf_masked`. A tabela usa esse campo, então deve ter.
    - Escape fecha o painel (FC-13): `onKeyDown` na `section` com `if (e.key === "Escape" && !busy) onCancel()`.
    - Acrescente ao teste do FC-11 um `fireEvent.keyDown(heading, { key: "Escape" })` que espera `onCancel` chamado.
    - O mesmo vale para o `DeclarationPanel`, com teste equivalente em `UnitQueue.declaration.test.tsx`.
    - Na 6b, a cobertura do FC-14 (alvos de 44px) vem da 6a, por CSS global. A prova é a captura em 768.
  - **`ErasureRequest.tsx`.**
    - Leia o arquivo. Se o bloqueio for o motivo, acrescente `ReasonHint`.
    - Se for a caixa "Conferi", mostre a frase de bloqueio igual à do `Counter`.
    - O botão continua `disabled`.
  - **`UnitPicker.tsx`.**
    - Os dois `Panel title="Atendente"` passam a `title="Unidade"`.
    - O painel de escolha ganha `sub="Escolha a unidade para ver o check-in e a fila"`. Confira a prop `sub` do `Panel`.
  - **`Breadcrumb.tsx` (só no dashboard).** Quando `multi && group.label === title`, não renderize o link do grupo nem o separador seguinte. Deixe o botão do menu. Isso vale para todas as telas assim, não só para Atendimento.
  - **`Attendance.tsx`.**
    - Nova ordem: `UnitPicker`, `CheckIn`, `Counter`, `ScreeningQueue`, `UnitQueue`, `Requests`, `Agenda`, `UnassignedRequests`, `ErasureRequest`, e por último os painéis de admin.
    - Antes do `UnitPicker`, para quem tem `canVerify && unit`, entra o índice:

```tsx
<nav aria-label="Nesta página" style={{ display: "flex", gap: 8, flexWrap: "wrap" }}>
  {[ [ "Check-in", "sec-checkin" ], [ "Validar cadastro", "sec-balcao" ], [ "Fila", "sec-fila" ],
     [ "Pedidos", "sec-pedidos" ], [ "Agenda", "sec-agenda" ] ].map(([ label, id ]) => (
    <button key={id} type="button" style={secondaryButtonStyle} onClick={() => {
      const el = document.getElementById(id); el?.scrollIntoView({ block: "start" });
      const h = el?.querySelector<HTMLElement>("h2,h3,[data-panel-title]"); if (h) { h.tabIndex = -1; h.focus(); }
    }}>{label}</button>
  ))}
</nav>
```

    - Cada um desses painéis fica num `<div id="sec-…">`. Confira em `components/Panel.tsx` como o título é renderizado e ajuste o seletor do foco.

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && npm run build`
  Expected: verde.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/modules/attendance/Requests.tsx src/modules/attendance/Requests.test.tsx src/modules/attendance/BookPanel.tsx src/modules/attendance/BookPanel.test.tsx src/modules/attendance/ErasureRequest.tsx src/modules/attendance/ErasureRequest.test.tsx src/modules/attendance/UnitPicker.tsx src/modules/Attendance.tsx src/shell/Breadcrumb.tsx src/shell/Breadcrumb.test.tsx
/opt/homebrew/bin/git commit -m "feat: page index, shorter request list, named booking panel and clearer unit picker

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

  Inclua no `add` os testes de `UnitPicker` e de `Attendance` que forem tocados.

---

### Task 6: Capturas, revisão, cards e publicação

- [ ] **Step 1: Suíte.** `npx vitest run`, `npx tsc --noEmit` e `npm run build`. Expected: verde.
- [ ] **Step 2: Capturas "depois".**
  - Suba o Vite do worktree (`npx vite --port 5196 --strictPort`) e rode os scripts da Task 0 contra ele, com o contorno de Origin de `scratchpad/capture/sweep6.mjs` (`ctx.route` com `route.fetch`).
  - Refaça os passos do percurso: chamar, encerrar com Retorno e marcar "Saiu sem atendimento". O teste usa dados do próprio fluxo, como no percurso.
  - Rode também `sweep6.mjs`. Expected: zero cortes.
- [ ] **Step 3: Revisão final** do branch pelo revisor mais capaz, com o Review Focus e a lista FC-01 a FC-19 como checagem. Os achados Critical e Important são corrigidos com TDD e registrados no ledger.
- [ ] **Step 4: Aprovação do usuário**, com `evidencias/2026-10-12-usabilidade/6b/comparacao.html`.
- [ ] **Step 5: Cards (com permissão do usuário).** Crie no board #2 dois cards, seguindo o template `pendencia-de-ciclo` e a memória `board-pendencias-de-ciclo`:
  - **FC-09:** nome legível do protocolo na fila e no check-in. A API devolve o slug em `protocol_name`.
  - **FC-18:** estado da escuta em curso na linha da fila, com confirmação antes de chamar.
- [ ] **Step 6: Merge e push.**
  1. Avise as sessões que mexem no dashboard.
  2. Confira `origin/main..main`.
  3. Faça `merge --ff-only` e `push` do dashboard e do docs.
  4. Remova o worktree.
  5. Confira o 5175.
