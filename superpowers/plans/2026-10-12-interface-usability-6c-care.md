# Ciclo de interface — subprojeto 6, fatia 6c: atendimento — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Corrigir, no dashboard, os achados AT-03 a AT-20 do percurso "atendimento". O percurso cobre:
- escuta;
- consulta;
- documentos;
- assinatura;
- linha do tempo;
- unidade.

**Architecture:** Seis frentes, cada uma com seus testes, sobre as peças da 6a e da 6b:
1. salvamento automático e aviso ao sair;
2. escuta;
3. consulta;
4. documentos;
5. assinatura e linha do tempo;
6. unidade, página e fila.

Só o dashboard muda.

**Tech Stack:** React 18, Vite 5, TypeScript, Vitest 2, Testing Library e Playwright (capturas).

**Spec:** `docs/superpowers/specs/2026-10-12-interface-usability-design.md` (§4.2; padrões da §3). Os achados estão em `rota-saude/evidencias/2026-10-12-usabilidade/atendimento/achados.md`.

## Já resolvidos (só conferir nas capturas)

| Achado | Resolvido em | O que foi feito |
|---|---|---|
| AT-01 | 6a | Tag quebra linha e as colunas têm piso |
| AT-02 | 6a | Colunas compartilhadas via subgrid |
| AT-08 | 6b | Consulta à vista, com "Consulta de X" |
| AT-17 | 6a | 44px abaixo de 1024, rádio e caixa de 24px |
| AT-18 (parte) | 6b | Painel "Unidade" e "prioridade N" |

## Viram card (board #2)

| Achados | Assunto |
|---|---|
| AT-03 e AT-04 | Rascunho da escuta e da consulta no servidor |
| AT-05 e AT-18 | Tabela CIAP-2 completa no dev e acentos |
| AT-13 e AT-14 | Tipo do documento clínico na lista de pendentes e lista do que foi assinado no retorno |

Os cards só são criados com a permissão do usuário.

## Global Constraints

- **Escopo:**
  - só front-end, sem mudar regra;
  - nada de API, dados ou permissões;
  - **nenhum dado clínico guardado no navegador** (localStorage, sessionStorage ou IndexedDB).
- **Testes:**
  - a suíte existente passa;
  - uma asserção só muda quando o comportamento muda de propósito, sempre com uma linha `Ruling:` no ledger que cite o AT;
  - todo AT corrigido ganha um teste que falha antes da correção.
- **Peças da 6a e da 6b:**
  - `useReveal`, `ActionButton`, `ConfirmButton`, `ActionMessage`;
  - `ReasonHint` (`reasonHint`), `formStyles`.
- **Cores:** só por papel do tema.
- **Papéis ARIA:** nenhum `role="status"`/`alert` novo além dos de mensagem.
- **Commits e git:**
  - Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - `/opt/homebrew/bin/git` e caminhos explícitos no `add`.
- **Worktree:** `rota-saude/.worktrees/dashboard-ui6c`, no branch `ui/usability-6c` a partir de `origin/main`, com `npm ci`.
- **Comandos de verificação:** `npx vitest run`, `npx tsc --noEmit` e `npm run build`.

## Review Focus

1. **Fechar a aba com o SOAP digitado há menos de 1,5 s.**
   - Esperado: o navegador avisa.
   - Esperado: ao trocar de aba, o texto é salvo na hora.
   - Prova: Task 1.
2. **O médico abre a aba Documentos com a consulta em rascunho e digita no editor (escondido) antes.**
   - Esperado: o aviso de saída considera o editor.
   - Prova: Task 1, com o hook no editor.
3. **Atestado preenchido e clique em "Receita".**
   - Esperado: pergunta antes de descartar; Cancelar mantém o atestado.
   - Prova: Task 4.
4. **Retorno da assinatura, e a pessoa clica no menu.**
   - Esperado: a tela de retorno sai e vale a tela escolhida.
   - Prova: Task 5.
5. **Profissional com vínculo em 1 de 7 unidades.**
   - Esperado: a unidade dele vem primeiro, marcada "seu vínculo".
   - Esperado: ao trocar, a atual aparece marcada e Escape cancela.
   - Prova: Task 6.

---

### Task 0: Preparação

- [ ] **Step 1:** worktree e linha de base.

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q && $G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-ui6c -b ui/usability-6c origin/main
cd $R/.worktrees/dashboard-ui6c && npm ci --silent && npx vitest run 2>&1 | tail -3
```

  - As capturas "antes" são as do percurso, em `evidencias/2026-10-12-usabilidade/atendimento/`.

---

### Task 1: Não perder o que foi digitado (AT-03, AT-04)

**Files:**
- `lib/useAutosave.ts` e `lib/useAutosave.test.ts` (procure o teste com `ls src/lib | grep -i autosave`)
- `lib/useLeaveGuard.ts` e `lib/useLeaveGuard.test.tsx`
- `modules/consultation/ConsultationEditor.tsx`
- `modules/attendance/ScreeningForm.tsx`

**Interfaces — Produces:**
- `useLeaveGuard(active: boolean): void`. Enquanto `active`, registra um `beforeunload` que chama `e.preventDefault()` e põe `e.returnValue = ""`. Remove ao desativar ou desmontar.
- `useAutosave` passa a devolver `pending: boolean`, verdadeiro quando o valor atual difere do último salvo.
- `useAutosave` salva na hora (`pump`) quando a aba fica oculta (`visibilitychange` com `document.visibilityState === "hidden"`).
- `saveStatusLabel(status, blocked, pending?)` mostra "alterações não salvas" quando `pending` é verdadeiro e o status é `idle` ou `saved`.

- [ ] **Step 1: testes que falham.**

```tsx
// lib/useLeaveGuard.test.tsx
import { afterEach, describe, expect, it } from "vitest";
import { cleanup, renderHook } from "@testing-library/react";
import { useLeaveGuard } from "./useLeaveGuard";
afterEach(cleanup);
describe("useLeaveGuard (AT-03/AT-04)", () => {
  it("avisa ao sair só enquanto ativo", () => {
    const { rerender, unmount } = renderHook(({ on }) => useLeaveGuard(on), { initialProps: { on: true } });
    const e = new Event("beforeunload", { cancelable: true });
    window.dispatchEvent(e);
    expect(e.defaultPrevented).toBe(true);
    rerender({ on: false });
    const e2 = new Event("beforeunload", { cancelable: true });
    window.dispatchEvent(e2);
    expect(e2.defaultPrevented).toBe(false);
    unmount();
  });
});
```

  No teste do `useAutosave`, use o harness existente (`renderHook` com `save` mockado e `delayMs` grande):
  - **`pending`:** com o valor alterado, `pending` é verdadeiro; depois do salvamento, é falso.
  - **Aba oculta:** com `delayMs: 10_000`, alterar o valor e disparar `visibilitychange` com `visibilityState = "hidden"` (via `Object.defineProperty(document, "visibilityState", { value: "hidden", configurable: true })`) chama `save` sem esperar o tempo.
  - **Rótulo:** `saveStatusLabel({ kind: "idle" }, null, true)` devolve "alterações não salvas".

  Run: `npx vitest run src/lib/useLeaveGuard.test.tsx src/lib/useAutosave.test.ts`
  Expected: FAIL.

- [ ] **Step 2: implementar.**

```ts
// lib/useLeaveGuard.ts — aviso do navegador ao sair com algo não salvo
// (AT-03/AT-04). Nada é guardado: só se avisa.
import { useEffect } from "react";
export function useLeaveGuard(active: boolean): void {
  useEffect(() => {
    if (!active) return;
    const onLeave = (e: BeforeUnloadEvent) => { e.preventDefault(); e.returnValue = ""; };
    window.addEventListener("beforeunload", onLeave);
    return () => window.removeEventListener("beforeunload", onLeave);
  }, [ active ]);
}
```

  - **`useAutosave`:**
    - devolva `pending: key !== savedKey.current` (re-render a cada tecla já acontece);
    - acrescente um efeito com `document.addEventListener("visibilitychange", …)` que chama `pump()` quando `document.visibilityState === "hidden"`;
    - `saveStatusLabel` ganha o terceiro parâmetro.
  - **`ConsultationEditor`:**
    - `useLeaveGuard(autosave.pending && !locked && !sealing)`;
    - o rótulo passa a `saveStatusLabel(autosave.status, blocked, autosave.pending)`;
    - `onBlur` no `<fieldset>`, que borbulha do `focusout`: `() => void autosave.flush()`.
  - **`ScreeningForm`:** `useLeaveGuard(!busy && (ciap !== null || note !== "" || JSON.stringify(vitalsForm) !== JSON.stringify(EMPTY_VITALS_FORM) || reason !== ""))`.

  Run: `npx vitest run 2>&1 | tail -3`
  Expected: verde.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git add src/lib/useLeaveGuard.ts src/lib/useLeaveGuard.test.tsx src/lib/useAutosave.ts src/lib/useAutosave.test.ts src/modules/consultation/ConsultationEditor.tsx src/modules/attendance/ScreeningForm.tsx
/opt/homebrew/bin/git commit -m "feat: warn before leaving unsaved care forms and save the draft when the tab is hidden

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Escuta (AT-05, AT-07 escuta, AT-11, AT-12)

**Files:**
- `modules/attendance/Ciap2Search.tsx`
- `modules/attendance/ColorDecision.tsx`
- `modules/attendance/ScreeningForm.tsx`
- os testes: `ScreeningForm.test.tsx` e, se houver, `Ciap2Search.test.tsx` e `ColorDecision.test.tsx`

- [ ] **Step 1: testes que falham** (no `ScreeningForm.test.tsx`, com o harness do arquivo).
  - **AT-05:**
    - o campo de busca tem `placeholder` "código ou nome, ex.: K86";
    - sem resultados, aparece "nenhum código encontrado — tente o código (ex.: K86) ou outro termo".
  - **AT-11:** com a sugestão `ready` trazendo `alerts: [ "<um código de alerta das fixtures> " ]`, aparece "Nesta escuta: <rótulo do alerta>", com o mesmo `alertLabel` que `VitalSignsFields` usa.
  - **AT-12:**
    - com a cor final diferente da sugerida e justificativa "curta", o textarea da justificativa tem `aria-describedby` que inclui um id cujo texto é "mínimo de 10 caracteres — faltam 5";
    - o botão "Concluir escuta" tem `aria-disabled="true"` e `aria-describedby` apontando para o texto do `problem`.
  - **AT-07:** ao montar o formulário, `scrollIntoView` é chamado e o foco vai para o título "Escuta inicial".

  Run: `npx vitest run src/modules/attendance/ScreeningForm.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - **`Ciap2Search`:** troque o `placeholder` pelo texto acima e o "nenhum código encontrado" pela frase com a orientação.
  - **`ColorDecision`:**
    - abaixo de `matched_rules`, quando `state.suggestion.alerts.length > 0`, mostre `<span style={{ color: "var(--ink2)" }}>Nesta escuta: {alerts.map(alertLabel).join(", ")}</span>`;
    - importe `alertLabel` de onde `VitalSignsFields` importa;
    - abaixo do textarea da justificativa: `<ReasonHint id="color-reason-hint" text={reason} />`;
    - o textarea passa a `aria-describedby="color-reason-notice color-reason-hint"`.
  - **`ScreeningForm`:**
    - o `<strong>` do título vira `<h3 style={{ margin: 0, fontSize: 14 }}>`;
    - `const ref = useReveal<HTMLElement>(true)` na `section`;
    - o botão de concluir vira `<ActionButton busy={busy} disabledReason={stale ? "calculando a cor sugerida…" : problem} onClick={submit}>`;
    - remova o `<small>` do `problem`, porque o `ActionButton` já mostra o motivo.
  - **Testes existentes:** os que esperavam `disabled` no "Concluir escuta"/"Salvar reavaliação" mudam para `aria-disabled` (Ruling AT-12/AT-07). Os que procuram "nenhum código encontrado" exato e o `placeholder` antigo também mudam (Ruling AT-05).

  Run: `npx vitest run 2>&1 | tail -3`
  Expected: verde.

- [ ] **Step 3: commit**, com `git add` dos arquivos tocados.

```bash
/opt/homebrew/bin/git commit -m "feat: clearer CIAP-2 search, matched vitals and blocked reasons in the screening form

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Consulta (AT-07 consulta, AT-09, AT-10)

**Files:** `modules/consultation/ConsultationWorkspace.tsx`, `ConsultationEditor.tsx`, `ConsultationView.tsx` e os testes deles.

- [ ] **Step 1: testes que falham.**
  - **AT-10:**
    - sem consulta aberta, abaixo de "Iniciar consulta" aparece "Se você já começou esta consulta, o seu rascunho abre";
    - quando `startConsultation` recusa com `already_exists` e o rascunho é lido, aparece o `status` "Rascunho retomado.".
  - **AT-07:** ao abrir o editor, o foco vai para o título "Consulta — rascunho". Ao finalizar, o foco vai para o título da consulta finalizada no `ConsultationView` (leia o arquivo para o texto exato).
  - **AT-09:** o `ConsultationView` não tem botão "Fechar consulta"; tem "Fechar painel".

  Run: `npx vitest run src/modules/consultation`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - **`ConsultationWorkspace`:**
    - a dica fica num `<p style={muted}>` abaixo do botão;
    - no `catch`, com `existing`, depois de `setConsultation(...)`, chame `setNotice("Rascunho retomado.")`.
  - **`ConsultationEditor`:** o `<strong>Consulta — rascunho</strong>` vira `<h3 style={{ margin: 0, fontSize: 14 }}>` e a `section` recebe `ref={useReveal<HTMLElement>(true)}`.
  - **`ConsultationView`:**
    - aplique o mesmo `useReveal(true)` no título;
    - troque o rótulo "Fechar consulta" por "Fechar painel".
  - **Testes existentes:** os que clicam "Fechar consulta" mudam para "Fechar painel" (Ruling AT-09).

  Run: `npx vitest run 2>&1 | tail -3`
  Expected: verde.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git commit -m "feat: reveal the consultation editor and view, say when a draft is resumed and rename the close button

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Documentos (AT-06, AT-07 documentos, AT-20)

**Files:**
- `modules/documents/ConsultationDocuments.tsx`
- `modules/documents/PrescriptionForm.tsx`
- `modules/documents/MedicationSearch.tsx`
- os testes: `ConsultationDocuments.test.tsx`, `PrescriptionForm.test.tsx` e `MedicationSearch.test.tsx`

- [ ] **Step 1: testes que falham.**
  - **AT-06:**
    - com o formulário do atestado aberto, clicar em outro tipo (por exemplo "Receita") mostra o grupo "Descartar o formulário em edição?";
    - Cancelar mantém o atestado;
    - Confirmar abre a receita.
  - **AT-07:**
    - depois de emitir, o foco vai para o aviso de sucesso (`role="status"`);
    - quando a emissão recusa, o foco vai para o alerta.
  - **AT-20:**
    - o campo da quantidade do item 1 tem nome acessível "Quantidade do item 1" e o texto visível "Quantidade" (`getByText("Quantidade", { selector: "span" })` dentro do `<li>`);
    - o mesmo vale para Unidade, Via, Duração em dias, Posologia e Uso contínuo;
    - na busca, com 3 resultados controlados e bloqueados, o texto de bloqueio aparece uma vez no topo da lista, e cada linha mostra só a tag "controlado".

  Run: `npx vitest run src/modules/documents`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - **`ConsultationDocuments`:**
    - `const [ switching, setSwitching ] = useState<DocumentKind | null>(null)`;
    - em `start(kind)`, quando `form` existe e `form.kind !== kind`, chame `setSwitching(kind)` e retorne;
    - renderize, perto dos botões de tipo, quando `switching`, um grupo com a pergunta "Descartar o formulário em edição?", Confirmar (que chama `setSwitching(null)` e depois `open(...)` do tipo) e Cancelar;
    - para isso, tire a lógica de `open` por tipo de dentro de `start`, numa função `openKind(kind)`;
    - os `<p>` de `notice` e `error` ganham `tabIndex={-1}` e `ref`, com um efeito que foca o que aparecer.
  - **`PrescriptionForm`:**
    - cada rótulo visível perde o índice (`<span>Quantidade</span>`);
    - o input ganha `aria-label={\`Quantidade do item ${n}\`}`, e o mesmo nos outros campos.
  - **`MedicationSearch`:**
    - quando `blockControlled` e algum item é controlado, mostre `CONTROLLED_BLOCKED` uma vez acima da `<ul>`;
    - na linha, a tag passa a "controlado".
  - **Testes existentes** que procuram "Quantidade 1", "Via 1", "Uso contínuo 1" e semelhantes mudam para "Quantidade do item 1" (Ruling AT-20). O texto `CONTROLLED_BLOCKED` por linha também muda.

  Run: `npx vitest run 2>&1 | tail -3`
  Expected: verde.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git commit -m "feat: confirm before switching documents, focus the result and simplify prescription item labels

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Assinatura e linha do tempo (AT-13, AT-14, AT-15)

**Files:**
- `lib/signature.ts`
- `modules/SignaturePending.tsx`
- `modules/SignatureCallback.tsx`
- `main.tsx`
- `lib/timeline.ts`
- `modules/timeline/PatientTimeline.tsx`
- os testes deles

- [ ] **Step 1: testes que falham.**
  - **AT-13:**
    - `reasonLabel("session_expired")` vira "sem sessão de assinatura aberta — abra a sessão para assinar";
    - a coluna da lista de pendentes passa a se chamar "Emitido ou finalizado em".
  - **AT-14:** com o `SignatureCallback` no estado "done", disparar `hashchange` chama a prop nova `onLeave`.
  - **AT-15:**
    - `entryWhere` devolve `{ unit, person }`;
    - um profissional cujo nome é um e-mail vira "pessoa sem nome cadastrado";
    - a coluna "Onde e quem" renderiza a unidade e a pessoa em duas linhas.

  Run: `npx vitest run src/lib/signature.test.ts src/modules/SignaturePending.test.tsx src/modules/SignatureCallback.test.tsx src/lib/timeline.test.ts src/modules/timeline`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - **`lib/signature.ts`:** troque o texto de `session_expired`.
  - **`SignaturePending`:** troque o rótulo da coluna.
  - **`SignatureCallback`:**
    - prop opcional `onLeave?(): void`;
    - `useEffect(() => { if (!onLeave) return; window.addEventListener("hashchange", onLeave); return () => window.removeEventListener("hashchange", onLeave); }, [ onLeave ])`.
  - **`main.tsx`:** passe `onLeave={() => setSignatureReturn(null)}`.
  - **`lib/timeline.ts`:**

```ts
export function entryWhere(entry: TimelineEntry): { unit: string | null; person: string | null } {
  const name = entry.professional?.name ?? null;
  return { unit: entry.unit?.name ?? null, person: name && name.includes("@") ? "pessoa sem nome cadastrado" : name };
}
```

  - **`PatientTimeline`:** a célula renderiza `<span style={{ display: "flex", flexDirection: "column" }}>{w.unit ?? "—"}{w.person && <span style={{ color: "var(--ink2)" }}>{w.person}</span>}</span>`.
  - **Outros usos de `entryWhere`:** confira com `grep -rn entryWhere src` e ajuste-os.
  - **Testes existentes** de `session_expired` e de `entryWhere` mudam (Ruling AT-13/AT-15).

  Run: `npx vitest run 2>&1 | tail -3`
  Expected: verde.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git commit -m "feat: actionable signature reason, leave the signature return on navigation and two-line timeline authorship

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Unidade, página e fila (AT-16, AT-18, AT-19)

**Files:**
- `modules/attendance/UnitPicker.tsx` e `UnitPicker.test.tsx`
- `modules/Attendance.tsx` e `Attendance.test.tsx`
- `modules/attendance/UnitQueue.tsx` e `UnitQueue.usability.test.tsx`

- [ ] **Step 1: testes que falham.**
  - **AT-16, no `UnitPicker`:**
    - com `linkedUnitIds={new Set([ "u2" ])}`, a primeira unidade da lista é a `u2`, com o texto "seu vínculo";
    - ao clicar "trocar", a unidade atual tem `aria-current="true"`, e há um botão "manter UBS Centro";
    - Escape volta sem trocar;
    - `onChoosing(true)`/`onChoosing(false)` é chamado ao abrir e ao fechar.
  - **AT-16, na página:** com o seletor aberto, `Attendance` não mostra a Fila nem o Check-in.
  - **AT-18:** para quem só tem `health_professional`, o subtítulo da página é "fila clínica · acolhimento e consulta". Para quem tem `citizen_verifier`, continua "balcão · verificação presencial". Com os dois papéis, "balcão e fila clínica".
  - **AT-19:** com `waiting` vazio, "Chamar próximo" tem `aria-disabled="true"` e o motivo "ninguém aguardando".

  Run: `npx vitest run src/modules/attendance/UnitPicker.test.tsx src/modules/Attendance.test.tsx src/modules/attendance/UnitQueue.usability.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - **`UnitPicker`:**
    - props opcionais `linkedUnitIds?: ReadonlySet<string>` e `onChoosing?(open: boolean): void`;
    - ordene as unidades com vínculo primeiro, mantendo a ordem original dentro de cada grupo;
    - na linha, mostre `<Tag tone="ok" mono={false}>seu vínculo</Tag>`;
    - a atual recebe `aria-current="true"` e a marca "atual";
    - quando `choosing && selected`, há o botão `manter ${selected.name}` (`setChoosing(false)`) e Escape no painel faz o mesmo;
    - chame `onChoosing` num efeito que observa `choosing`.
  - **`Attendance`:**
    - passe `linkedUnitIds={linkedUnitIds}`;
    - `onChoosing={setChoosingUnit}`, com `const [ choosingUnit, setChoosingUnit ] = useState(false)`;
    - esconda as seções da unidade (check-in, validação, acolhimento, fila, pedidos e agenda) quando `choosingUnit`;
    - subtítulo por papel: `canVerify && canCareRole ? "balcão e fila clínica" : canCareRole ? "fila clínica · acolhimento e consulta" : "balcão · verificação presencial"`.
  - **`UnitQueue`:** `<ActionButton busy={callingNext} disabledReason={query.isSuccess && waiting.length === 0 ? "ninguém aguardando" : null} onClick={onCallNext}>`.

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && npm run build`
  Expected: verde.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git commit -m "feat: linked units first, keep-current and Escape in the unit picker, role-based subtitle and idle call-next

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Capturas, revisão, cards e publicação

- [ ] **Capturas "depois":**
  - suba o Vite do worktree na porta 5197;
  - use os scripts de `scratchpad/u6b` (Origin e unidade) e da pasta do percurso `scratchpad/u6-atend/`;
  - cubra: escuta (motivo da cor, justificativa), consulta (dica de retomar, status "alterações não salvas"), documentos (pergunta ao trocar), receita, pendentes de assinatura, linha do tempo e seletor de unidade;
  - onde o dev não tiver dados, simule a resposta só no navegador e marque a captura como "sim-".
- [ ] **Revisão final** pelo revisor mais capaz:
  - use o Review Focus e a lista AT;
  - corrija os achados Critical e Important com TDD;
  - registre-os no ledger.
- [ ] **Aprovação do usuário** com `evidencias/2026-10-12-usabilidade/6c/comparacao.html`.
- [ ] **Cards, com permissão:** 3 cards no board #2, seguindo o template e os campos da memória `board-pendencias-de-ciclo`:
  - AT-03/04;
  - AT-05/18;
  - AT-13/14.
- [ ] **Merge e push:**
  - avise a sessão do módulo 20 (consulta, linha do tempo e documentos são dela);
  - confira `origin/main..main`;
  - faça `merge --ff-only` e `push` do dashboard e do docs;
  - remova o worktree.
