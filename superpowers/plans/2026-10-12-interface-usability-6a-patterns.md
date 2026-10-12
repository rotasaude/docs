# Ciclo de interface — subprojeto 6, fatia 6a: padrões comuns — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Entregar no dashboard as peças de usabilidade que as fatias 6b, 6c e 6d vão usar: revelar o painel, diálogo, botão de ação, confirmação em dois passos, mensagem junto da ação e descrição obrigatória na ação sensível. Também acabar com as reticências, garantir alvos de 44px abaixo de 1024px e fechar as pendências A-01 a A-12 no dashboard e no admin.

**Architecture:** As peças novas vivem no dashboard, em `src/lib` (hooks) e `src/components` (componentes), cada uma com o próprio teste. Elas não são adotadas pelas telas nesta fatia, com uma exceção: a gaveta (`shell/Drawer.tsx`) passa a usar `useDialog`. A adoção nas telas é trabalho das fatias 6b, 6c e 6d. As mudanças globais (reticências, alvos de toque, pendências A) valem nos dois apps, porque o shell e o tema do admin são cópias do dashboard.

**Tech Stack:** React 18, Vite 5, TypeScript, Vitest 2 com jsdom e Testing Library, e Playwright para a varredura.

**Spec:** `docs/superpowers/specs/2026-10-12-interface-usability-design.md` (§3 e §2, linha 6a).

## Global Constraints

- **Escopo:** só front-end, sem mudar regra. Nada de API, dados, permissões ou regra de negócio.
- **Testes:**
  - a suíte existente passa;
  - uma asserção só muda quando o comportamento muda de propósito, e então com uma linha `Ruling:` no ledger que nomeie a pendência;
  - toda peça nova ganha um teste que falha antes da implementação.
- **Cores:** só por papel do tema (`var(--x)`), sem cor fixa e sem reserva nova (`var(--x, …)`). O foco nunca é zerado.
- **Papéis ARIA:** nenhum `role="status"` ou `alert` além dos que já existem, exceto o do `ActionMessage`, que só aparece quando há mensagem.
- **Commits:** Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Use `/opt/homebrew/bin/git` e caminhos explícitos no `add`.
- **Worktrees:** `rota-saude/.worktrees/dashboard-ui6a` e `rota-saude/.worktrees/admin-ui6a`, no branch `ui/usability-6a` a partir de `origin/main`, com `npm ci`. Nunca dentro de `apps/<app>`.
- **Comandos de verificação** (em cada app): `npx vitest run`, `npx tsc --noEmit` e `npm run build`.

## Review Focus

1. **Gaveta aberta em 900px quando a janela passa para 1200px.**
   - Esperado: fecha sozinha, sem roubar o foco do conteúdo.
   - Onde se prova: teste na Task 1.
2. **Duplo clique rápido num `ActionButton` ocupado.**
   - Esperado: a ação roda uma vez só e o foco fica no botão.
   - Onde se prova: teste na Task 6.
3. **`ConfirmButton` com Escape, ou com Cancelar, no segundo passo.**
   - Esperado: volta ao botão original com o foco nele, sem executar.
   - Onde se prova: teste na Task 7.
4. **Texto sem espaço (UUID, e-mail longo) numa célula estreita em 1024px.**
   - Esperado: quebra dentro da célula, sem vazar nem cortar.
   - Onde se prova: varredura da Task 10.
5. **Tema escolhido quando o `localStorage` lê mas falha ao gravar.**
   - Esperado: a escolha vale até recarregar.
   - Onde se prova: teste na Task 1.

---

## File Structure (dashboard; o admin espelha o que está marcado "e admin")

| Arquivo | Responsabilidade |
|---|---|
| `src/theme/theme.ts` (e admin) | A-06: a escolha em memória vale antes da lida do armazenamento |
| `src/shell/router.ts` (e admin) | A-07: `go` para o lugar atual não empilha |
| `src/shell/Shell.tsx` (e admin) | A-10: fecha a gaveta ao passar de 1024px |
| `src/shell/Breadcrumb.tsx` (e admin) | A-08/A-09: itens do menu são links e o menu navega por setas |
| `src/lib/useDialog.ts` (+ teste) | Foco preso, Escape e devolução do foco; usado pela gaveta |
| `src/lib/useReveal.ts` (+ teste) | Rola até o painel e foca o título |
| `src/components/ActionButton.tsx` (+ teste) | Ocupado e bloqueado com `aria-disabled` e motivo visível |
| `src/components/ConfirmButton.tsx` (+ teste) | Confirmação em dois passos |
| `src/components/ActionMessage.tsx` (+ `src/lib/useActionMessage.ts` e testes) | Mensagem junto da ação, que some na próxima |
| `src/components/SensitiveAction.tsx`, `src/modules/Protocols.tsx` | `description` obrigatória; texto de consequência por ação de protocolo |
| `src/components/formStyles.ts` | Desabilitado sem opacidade |
| `src/components/datatable.css`, `KeyValue.tsx`, `modules/Overview.tsx` (e admin) | Fim das reticências |
| `src/theme/responsive.css` (e admin) | Alvos de 44px abaixo de 1024px |
| `src/theme/usability-guards.test.ts` (e admin) | Guardas: reticências sem `title`, alvos de toque, cores, `--ink4` |
| `src/components/StatusDot.tsx` (e admin) | A-04: halo do triângulo |
| `apps/admin/src/modules/setup/{MfaEnroll,ProvisionCity,RegisterChannel}.tsx` | A-03: `--up` passa a `--ok` |

---

### Task 0: Preparação

- [ ] **Step 1: Worktrees e linha de base.**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
for app in dashboard admin; do
  $G -C $R/apps/$app fetch -q && $G -C $R/apps/$app worktree add -q $R/.worktrees/$app-ui6a -b ui/usability-6a origin/main
  (cd $R/.worktrees/$app-ui6a && npm ci --silent && npx vitest run 2>&1 | tail -3)
done
```

  Expected: as duas suítes verdes. Anote as contagens no ledger.

- [ ] **Step 2: Varredura e capturas "antes".**
  - Crie `scratchpad/capture/sweep6.mjs`. Ele entra no dashboard de dev em `http://curitiba.localhost:5175/dashboard/` como `recepcao@curitiba.demo` e como `admin@curitiba.demo`, com POST `/session` e `password: "dev-password"`.
  - O login tem limite de 10 a cada 3 minutos. Guarde o `storageState` e reaproveite a sessão.
  - Para cada tela, abra o hash e conte os elementos cortados com reticências e sem `title`:

```js
[...document.querySelectorAll("body *")].filter((el) => {
  const cs = getComputedStyle(el);
  return cs.textOverflow === "ellipsis" && el.scrollWidth > el.clientWidth + 1 && !el.closest("[title]");
}).length
```

  - Telas: `#/attendance`, `#/triages`, `#/team`, `#/protocols` e `#/`.
  - Larguras: 1024×800 e 1280×800.
  - Saída: um JSON e capturas em `rota-saude/evidencias/2026-10-12-usabilidade/6a/antes/`.
  - Faça o mesmo no admin. Antes, descubra a URL de dev do admin em `docker-compose.yml`; o login do admin é por TOTP do operador, como na captura do subprojeto 2, em `scratchpad/capture/capture.mjs`.
  - O script recebe a base como argumento, para rodar depois contra o Vite do worktree.
  - Expected: contagens maiores que zero, por exemplo, Recepção em 1024 com cerca de 90.

---

### Task 1: Tema, rota e gaveta (A-06, A-07, A-10; dashboard e admin)

**Files:** `src/theme/theme.ts`, `src/theme/theme.store.test.ts`, `src/shell/router.ts`, `src/shell/router.test.ts`, `src/shell/Shell.tsx`, `src/shell/Shell.test.tsx`, nos dois apps.

- [ ] **Step 1: Escrever os testes, que vão falhar** (dashboard; repita idêntico no admin).

  No fim de `theme.store.test.ts`:

```ts
describe("armazenamento que lê mas não grava (A-06)", () => {
  it("a escolha da sessão vale sobre o valor antigo salvo", () => {
    localStorage.setItem(THEME_KEY, "rota");
    vi.spyOn(Storage.prototype, "setItem").mockImplementation(() => { throw new Error("quota"); });
    setTheme("alto-contraste");
    expect(getTheme()).toBe("alto-contraste");
  });
});
```

  Em `router.test.ts`, use os helpers do arquivo para montar `useHashRoute`. Leia o arquivo antes.

```ts
it("go para o lugar atual não empilha histórico (A-07)", () => {
  window.location.hash = "#/team";
  const { result } = renderHook(() => useHashRoute({ ids: new Set([ "team" ]), slugs: new Set(), isAllowed: () => true }));
  const push = vi.spyOn(window.history, "pushState");
  act(() => result.current.go({ kind: "screen", id: "team" }));
  expect(push).not.toHaveBeenCalled();
  act(() => result.current.go({ kind: "home" }));
  expect(push).toHaveBeenCalledTimes(1);
});
```

  Em `Shell.test.tsx`, use o render existente do arquivo. Antes, estube `window.matchMedia` com um objeto controlável:

```ts
it("a gaveta fecha quando a janela passa de 1024px (A-10)", () => {
  let listener: ((e: { matches: boolean }) => void) | null = null;
  vi.stubGlobal("matchMedia", (q: string) => ({ matches: false, media: q,
    addEventListener: (_: string, l: (e: { matches: boolean }) => void) => { listener = l; },
    removeEventListener: () => { listener = null; } }));
  renderShell(); // helper do arquivo
  fireEvent.click(screen.getByRole("button", { name: /menu/i }));
  expect(screen.getByRole("dialog", { name: "Menu" })).toBeTruthy();
  act(() => listener?.({ matches: true }));
  expect(screen.queryByRole("dialog", { name: "Menu" })).toBeNull();
});
```

  Run: `npx vitest run src/theme/theme.store.test.ts src/shell/router.test.ts src/shell/Shell.test.tsx`
  Expected: FAIL nos três casos novos.

- [ ] **Step 2: Implementar.**
  - `theme.ts`, `getTheme`: `if (memory) return memory;` antes de ler o armazenamento. A memória só existe depois de um `setTheme` nesta sessão.
  - `router.ts`, `go`: `if (toHash(r) === window.location.hash) { setRoute(r); return; }` antes do `writeHash`.
  - `Shell.tsx`:

```tsx
useEffect(() => {
  if (!menuOpen) return;
  const mq = window.matchMedia("(min-width: 1024px)");
  const onChange = (e: { matches: boolean }) => { if (e.matches) setMenuOpen(false); };
  mq.addEventListener("change", onChange);
  return () => mq.removeEventListener("change", onChange);
}, [ menuOpen ]);
```

  Run: `npx vitest run 2>&1 | tail -3` nos dois apps.
  Expected: verde.

- [ ] **Step 3: Commit** em cada app.

```bash
/opt/homebrew/bin/git add src/theme/theme.ts src/theme/theme.store.test.ts src/shell/router.ts src/shell/router.test.ts src/shell/Shell.tsx src/shell/Shell.test.tsx
/opt/homebrew/bin/git commit -m "fix: keep session theme, skip duplicate history entries and close the drawer on wide screens

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Menu da trilha (A-08, A-09; dashboard e admin)

**Files:** `src/shell/Breadcrumb.tsx` e `src/shell/Breadcrumb.test.tsx`, nos dois apps.

- [ ] **Step 1: Escrever o teste, que vai falhar.** Monte a trilha como os testes existentes do arquivo, com um grupo de 3 telas.

```tsx
it("itens do menu são links com href e navegam por setas (A-08, A-09)", () => {
  renderMultiGroup(); // helper do arquivo: rota numa tela de grupo com 3 telas
  fireEvent.click(screen.getByRole("button", { expanded: false }));
  const items = screen.getAllByRole("menuitem");
  expect(items[0].tagName).toBe("A");
  expect(items[0].getAttribute("href")).toMatch(/^#\//);
  expect(document.activeElement).toBe(items[0]);
  fireEvent.keyDown(items[0], { key: "ArrowDown" });
  expect(document.activeElement).toBe(items[1]);
  fireEvent.keyDown(items[1], { key: "End" });
  expect(document.activeElement).toBe(items[2]);
  fireEvent.keyDown(items[2], { key: "ArrowDown" });
  expect(document.activeElement).toBe(items[0]);
  fireEvent.keyDown(items[0], { key: "Escape" });
  expect(screen.queryByRole("menu")).toBeNull();
  expect(document.activeElement?.getAttribute("aria-haspopup")).toBe("menu");
});
```

  Run: `npx vitest run src/shell/Breadcrumb.test.tsx`
  Expected: FAIL, porque o item é `BUTTON`.

- [ ] **Step 2: Implementar.**
  - Troque os `<button role="menuitem">` por `<RouteLink role="menuitem" route={{ kind: "screen", id: i.id }} go={go} afterGo={() => setOpen(false)} aria-current=…>`.
  - Ao abrir, um `useEffect` foca o primeiro item.
  - No `div role="menu"`, um `onKeyDown` move o foco entre os itens:
    - ArrowDown e ArrowUp, com volta circular;
    - Home e End;
    - Escape fecha e devolve o foco ao botão (guarde-o num `ref`).
  - O `onKey` de Escape do documento continua para cliques fora.

  Run: `npx vitest run 2>&1 | tail -3`
  Expected: verde nos dois apps.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/shell/Breadcrumb.tsx src/shell/Breadcrumb.test.tsx
/opt/homebrew/bin/git commit -m "fix: make breadcrumb menu items real links with arrow-key navigation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Cores e guardas pendentes (A-01 a A-05, A-11)

**Files:**
- dashboard e admin: `src/components/StatusDot.tsx`, `src/components/StatusDot.test.tsx`, `src/theme/a11y-guards.test.ts`, `src/theme/contrast.test.ts` (novo);
- só admin: `src/modules/setup/{MfaEnroll,ProvisionCity,RegisterChannel}.tsx`.

- [ ] **Step 1: Conferir A-01 e A-02.**
  - O tom `urgent` já mapeia para `--down` com forma `square` (`src/theme/tokens.ts:67,81`), e o selo de notificação já usa `var(--on-tone)` (`src/shell/NotificationCenter.tsx`).
  - Registre no ledger: `Ruling: A-01 e A-02 já resolvidos no subprojeto 2 — conferidos no código — custo: nenhum`.
  - Repita a conferência no admin. Se lá ainda não estiverem resolvidos, aplique as mesmas linhas.

- [ ] **Step 2: Escrever os testes, que vão falhar.**

  `src/theme/contrast.test.ts` (A-05):

```ts
// Pares de contraste que a tela usa e o contracts não testava: link (accent)
// e tom sobre o painel, em todos os temas dos painéis.
import { describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const css = readFileSync(join(dirname(fileURLToPath(import.meta.url)), "vendor/tokens.css"), "utf8");
function block(theme: string): Record<string, string> {
  const m = new RegExp(`:root\\[data-theme="${theme}"\\]\\s*\\{([^}]*)\\}`).exec(css);
  if (!m) throw new Error(`tema ${theme} não está no tokens.css`);
  return Object.fromEntries([ ...m[1].matchAll(/--([\w-]+):\s*(#[0-9a-fA-F]{6})/g) ].map((x) => [ x[1], x[2] ]));
}
function lum(hex: string) {
  const c = [ 1, 3, 5 ].map((i) => parseInt(hex.slice(i, i + 2), 16) / 255)
    .map((v) => (v <= 0.03928 ? v / 12.92 : ((v + 0.055) / 1.055) ** 2.4));
  return 0.2126 * c[0] + 0.7152 * c[1] + 0.0722 * c[2];
}
const ratio = (a: string, b: string) => { const [ x, y ] = [ lum(a), lum(b) ]; return (Math.max(x, y) + 0.05) / (Math.min(x, y) + 0.05); };

describe.each([ "govbr", "rota", "alto-contraste" ])("tema %s", (theme) => {
  const t = block(theme);
  it.each([ "accent", "ok", "warn", "down", "info" ])("--%s sobre --panel ≥ 4,5:1", (role) => {
    expect(ratio(t[role], t.panel)).toBeGreaterThanOrEqual(4.5);
  });
  it.each([ "ok", "warn", "down", "info" ])("--%s sobre --%s-bg ≥ 4,5:1", (role) => {
    expect(ratio(t[role], t[`${role}-bg`])).toBeGreaterThanOrEqual(4.5);
  });
});
```

  Se o tema `govbr` for o `:root` sem atributo no `tokens.css`, leia o arquivo e ajuste o seletor do `block`. Registre o ajuste no ledger.

  Em `a11y-guards.test.ts`, a A-11 acrescenta `"shell/ContaMenu.tsx"` à lista do `it.each` de `--ink4`, e também `"shell/TenantChip.tsx"`, que só existe no dashboard.

  Em `StatusDot.test.tsx`, a A-04:

```tsx
it("triângulo pulsante tem halo fora do recorte (A-04)", () => {
  const { container } = render(<StatusDot level="warn" pulse />);
  const outer = container.firstElementChild as HTMLElement;
  expect(outer.style.animation).toContain("pulse-dot");
  expect(outer.style.clipPath).toBe("");
  expect((outer.firstElementChild as HTMLElement).style.clipPath).toContain("polygon");
});
```

  Run: `npx vitest run src/theme src/components/StatusDot.test.tsx`
  Expected:
  - FAIL no StatusDot;
  - se algum par do `contrast.test` falhar, isso é achado real: pare e reporte o par, sem mexer no `tokens.css`, que é do `contracts`;
  - a lista do `--ink4` passa ou acusa algum arquivo.

- [ ] **Step 3: Implementar.**
  - **StatusDot.** Quando `shape === "triangle" && pulse`, renderize um `span` externo com a animação e o tamanho, e um `span` interno com o `clipPath` e o fundo. Nos demais casos, mantenha o elemento único de hoje.
  - **A-03 (admin).** Nos três arquivos de `setup`, troque `var(--up-bg)` por `var(--ok-bg)` e `var(--up)` por `var(--ok)`. Acrescente uma guarda em `a11y-guards.test.ts` do admin:

```ts
it("nenhum papel inexistente --up (A-03)", () => {
  const offenders = sources(src).filter((f) => /var\(--up(-bg)?\)/.test(readFileSync(f, "utf8")));
  expect(offenders.map((f) => relative(src, f))).toEqual([]);
});
```

  - **ContaMenu.** Se a guarda acusar `--ink4` em `ContaMenu`, troque por `--ink2`.

  Run: `npx vitest run 2>&1 | tail -3` nos dois apps.
  Expected: verde.

- [ ] **Step 4: Commit** em cada app, com os arquivos tocados.

```bash
/opt/homebrew/bin/git commit -m "fix: pending colour items from the interface cycle (status halo, ok tone, contrast pairs)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: `useDialog` (só dashboard)

**Files:** `src/lib/useDialog.ts`, `src/lib/useDialog.test.tsx` e `src/shell/Drawer.tsx`.

**Interfaces — Produces:** `useDialog<T extends HTMLElement>(open: boolean, onClose: () => void, returnFocusTo?: HTMLElement | null): React.RefObject<T>`. Quando `open` é verdadeiro:
- foca o primeiro elemento focável dentro do `ref`;
- prende o Tab;
- Escape chama `onClose`;
- ao fechar, devolve o foco a `returnFocusTo`.

- [ ] **Step 1: Escrever o teste, que vai falhar.**

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { useState } from "react";
import { useDialog } from "./useDialog";

afterEach(cleanup);

function Harness({ onClose }: { onClose: () => void }) {
  const [ open, setOpen ] = useState(false);
  const [ opener, setOpener ] = useState<HTMLButtonElement | null>(null);
  const ref = useDialog<HTMLDivElement>(open, () => { onClose(); setOpen(false); }, opener);
  return (
    <>
      <button ref={setOpener} onClick={() => setOpen(true)}>abrir</button>
      {open && (
        <div ref={ref} role="dialog" aria-modal="true" aria-label="Painel">
          <input aria-label="campo" />
          <button>último</button>
        </div>
      )}
    </>
  );
}

describe("useDialog", () => {
  it("foca dentro, prende o Tab, Escape fecha e devolve o foco", () => {
    const onClose = vi.fn();
    render(<Harness onClose={onClose} />);
    const opener = screen.getByRole("button", { name: "abrir" });
    opener.focus();
    fireEvent.click(opener);
    expect(document.activeElement).toBe(screen.getByLabelText("campo"));
    screen.getByRole("button", { name: "último" }).focus();
    fireEvent.keyDown(document, { key: "Tab" });
    expect(document.activeElement).toBe(screen.getByLabelText("campo"));
    fireEvent.keyDown(document, { key: "Escape" });
    expect(onClose).toHaveBeenCalledTimes(1);
    expect(screen.queryByRole("dialog")).toBeNull();
    expect(document.activeElement).toBe(opener);
  });
});
```

  Run: `npx vitest run src/lib/useDialog.test.tsx`
  Expected: FAIL, porque o módulo não existe.

- [ ] **Step 2: Implementar `src/lib/useDialog.ts`** com a lógica de hoje do `Drawer` e um seletor de focáveis mais amplo.

```ts
// Diálogo modal (spec 6a §3.2): foco no primeiro focável, Tab preso, Escape
// fecha, foco devolvido a quem abriu. Callbacks por ref: um onClose novo a
// cada render não reabre o efeito.
import { useEffect, useRef } from "react";

const FOCUSABLE = 'a[href],button:not([disabled]),input:not([disabled]),select:not([disabled]),textarea:not([disabled]),[tabindex]:not([tabindex="-1"])';

export function useDialog<T extends HTMLElement>(open: boolean, onClose: () => void, returnFocusTo?: HTMLElement | null) {
  const ref = useRef<T | null>(null);
  const onCloseRef = useRef(onClose);
  onCloseRef.current = onClose;
  const returnRef = useRef(returnFocusTo);
  returnRef.current = returnFocusTo;

  useEffect(() => {
    if (!open) return;
    const panel = ref.current;
    panel?.querySelector<HTMLElement>(FOCUSABLE)?.focus();
    function onKey(e: KeyboardEvent) {
      if (e.key === "Escape") { onCloseRef.current(); return; }
      if (e.key !== "Tab" || !panel) return;
      const items = Array.from(panel.querySelectorAll<HTMLElement>(FOCUSABLE));
      if (items.length === 0) return;
      const first = items[0];
      const last = items[items.length - 1];
      if (!e.shiftKey && document.activeElement === last) { e.preventDefault(); first.focus(); }
      else if (e.shiftKey && document.activeElement === first) { e.preventDefault(); last.focus(); }
    }
    document.addEventListener("keydown", onKey);
    return () => { document.removeEventListener("keydown", onKey); returnRef.current?.focus(); };
  }, [ open ]);

  return ref;
}
```

  No `Drawer.tsx`, apague o `useEffect` e os dois `ref` de callback. Use `const ref = useDialog<HTMLDivElement>(open, onClose, returnFocusTo);`.

  Run: `npx vitest run src/lib/useDialog.test.tsx src/shell/Drawer.test.tsx && npx vitest run 2>&1 | tail -3`
  Expected: verde; o `Drawer.test` continua igual.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/lib/useDialog.ts src/lib/useDialog.test.tsx src/shell/Drawer.tsx
/opt/homebrew/bin/git commit -m "refactor: extract useDialog from the drawer for reuse by side panels

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: `useReveal` (só dashboard)

**Files:** `src/lib/useReveal.ts` e `src/lib/useReveal.test.tsx`.

**Interfaces — Produces:** `useReveal<T extends HTMLElement>(active: boolean): React.RefObject<T>`. Quando `active` passa de falso para verdadeiro:
- chama `ref.current.scrollIntoView({ block: "nearest" })`;
- foca o primeiro `h2`, `h3` ou `[data-reveal-title]` dentro do elemento, ou o próprio elemento, pondo `tabIndex = -1` se faltar.

- [ ] **Step 1: Escrever o teste, que vai falhar.**

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { useState } from "react";
import { useReveal } from "./useReveal";

afterEach(cleanup);

function Harness() {
  const [ open, setOpen ] = useState(false);
  const ref = useReveal<HTMLElement>(open);
  return (
    <>
      <button onClick={() => setOpen(true)}>abrir</button>
      {open && <section ref={ref}><h3>Encerrar atendimento de Luíza</h3><button>ok</button></section>}
    </>
  );
}

describe("useReveal", () => {
  it("rola até o painel aberto e foca o título", () => {
    const scroll = vi.fn();
    Element.prototype.scrollIntoView = scroll;
    render(<Harness />);
    fireEvent.click(screen.getByRole("button", { name: "abrir" }));
    expect(scroll).toHaveBeenCalledWith({ block: "nearest" });
    const title = screen.getByRole("heading", { name: /Encerrar atendimento/ });
    expect(document.activeElement).toBe(title);
    expect(title.getAttribute("tabindex")).toBe("-1");
  });
});
```

  Run: `npx vitest run src/lib/useReveal.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: Implementar.**

```ts
// Painel que aparece por causa de um clique (spec 6a §3.1): rola até ele e
// põe o foco no título, para quem usa teclado ou leitor saber que abriu.
import { useEffect, useRef } from "react";

export function useReveal<T extends HTMLElement>(active: boolean) {
  const ref = useRef<T | null>(null);
  useEffect(() => {
    if (!active) return;
    const el = ref.current;
    if (!el) return;
    el.scrollIntoView?.({ block: "nearest" });
    const target = el.querySelector<HTMLElement>("[data-reveal-title],h2,h3") ?? el;
    if (!target.hasAttribute("tabindex")) target.setAttribute("tabindex", "-1");
    target.focus();
  }, [ active ]);
  return ref;
}
```

  Run: `npx vitest run src/lib/useReveal.test.tsx`
  Expected: PASS.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/lib/useReveal.ts src/lib/useReveal.test.tsx
/opt/homebrew/bin/git commit -m "feat: add useReveal to scroll opened panels into view and focus their title

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: `ActionButton` e desabilitado sem opacidade (só dashboard)

**Files:** `src/components/ActionButton.tsx`, `src/components/ActionButton.test.tsx` e `src/components/formStyles.ts`.

**Interfaces — Produces:**

```ts
ActionButton(props: {
  children: ReactNode;
  onClick(): void | Promise<void>;
  busy?: boolean;
  disabledReason?: string | null;
  variant?: "primary" | "secondary" | "danger";
  type?: "button" | "submit";
})
```

- Quando `busy` é verdadeiro ou há `disabledReason`:
  - o botão recebe `aria-disabled="true"` e ignora o clique;
  - **não** recebe o atributo `disabled`, para manter o foco.
- O motivo aparece num `<p id>` logo abaixo, ligado por `aria-describedby`.
- Enquanto a promessa de `onClick` não termina, o botão ignora novos cliques. O rótulo ganha o sufixo " …" e `aria-busy="true"`.

- [ ] **Step 1: Escrever o teste, que vai falhar.**

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, render, screen } from "@testing-library/react";
import { ActionButton } from "./ActionButton";

afterEach(cleanup);

describe("ActionButton", () => {
  it("bloqueado mostra o motivo, ignora o clique e continua focável", () => {
    const onClick = vi.fn();
    render(<ActionButton onClick={onClick} disabledReason="o motivo precisa de pelo menos 10 caracteres">Encerrar</ActionButton>);
    const b = screen.getByRole("button", { name: "Encerrar" });
    expect(b.getAttribute("aria-disabled")).toBe("true");
    expect(b.hasAttribute("disabled")).toBe(false);
    const reason = document.getElementById(b.getAttribute("aria-describedby")!);
    expect(reason?.textContent).toBe("o motivo precisa de pelo menos 10 caracteres");
    fireEvent.click(b);
    expect(onClick).not.toHaveBeenCalled();
  });

  it("duplo clique durante a ação roda uma vez e o foco fica no botão", async () => {
    let resolve!: () => void;
    const onClick = vi.fn(() => new Promise<void>((r) => { resolve = r; }));
    render(<ActionButton onClick={onClick}>Chamar</ActionButton>);
    const b = screen.getByRole("button", { name: "Chamar" });
    b.focus();
    fireEvent.click(b);
    fireEvent.click(b);
    expect(onClick).toHaveBeenCalledTimes(1);
    expect(b.getAttribute("aria-busy")).toBe("true");
    expect(document.activeElement).toBe(b);
    await act(async () => { resolve(); });
    expect(b.getAttribute("aria-busy")).toBeNull();
  });
});
```

  Run: `npx vitest run src/components/ActionButton.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: Implementar.**
  - Em `formStyles.ts`, o `disabledButtonStyle` passa a ser:

```ts
export const disabledButtonStyle: CSSProperties = {
  ...buttonStyle, background: "var(--sunken)", color: "var(--ink2)", border: "2px dashed var(--rule2)", cursor: "not-allowed"
};
export const dangerButtonStyle: CSSProperties = {
  ...buttonStyle, background: "var(--panel)", color: "var(--down)", border: "1px solid var(--down)"
};
```

  - `src/components/ActionButton.tsx`:

```tsx
// Botão de ação (spec 6a §3.3/§3.6): ocupado e bloqueado usam aria-disabled
// (o foco não se perde), o motivo do bloqueio fica visível e ligado ao botão,
// e um clique repetido durante a ação não dispara de novo.
import { useId, useRef, useState, type ReactNode } from "react";
import { buttonStyle, dangerButtonStyle, disabledButtonStyle, secondaryButtonStyle } from "./formStyles";

export function ActionButton({ children, onClick, busy, disabledReason, variant = "primary", type = "button" }: {
  children: ReactNode; onClick(): void | Promise<void>; busy?: boolean; disabledReason?: string | null;
  variant?: "primary" | "secondary" | "danger"; type?: "button" | "submit";
}) {
  const reasonId = useId();
  const running = useRef(false);
  const [ pending, setPending ] = useState(false);
  const isBusy = busy || pending;
  const blocked = isBusy || !!disabledReason;
  const base = variant === "danger" ? dangerButtonStyle : variant === "secondary" ? secondaryButtonStyle : buttonStyle;

  async function handle() {
    if (blocked || running.current) return;
    running.current = true;
    setPending(true);
    try { await onClick(); } finally { running.current = false; setPending(false); }
  }

  return (
    <span style={{ display: "inline-flex", flexDirection: "column", gap: 4 }}>
      <button type={type} onClick={() => void handle()} aria-disabled={blocked || undefined}
        aria-busy={isBusy || undefined} aria-describedby={disabledReason ? reasonId : undefined}
        style={blocked ? disabledButtonStyle : base}>
        {children}{isBusy ? " …" : null}
      </button>
      {disabledReason && <p id={reasonId} style={{ margin: 0, fontSize: 12, color: "var(--ink2)" }}>{disabledReason}</p>}
    </span>
  );
}
```

  O nome acessível com " …" passa a ser "Chamar …". O teste busca o botão antes do clique, então não quebra. Registre isso no ledger.

  Run: `npx vitest run src/components/ActionButton.test.tsx && npx vitest run 2>&1 | tail -3`
  Expected: PASS. Se algum teste existente checar a opacidade do `disabledButtonStyle`, ele muda de propósito: registre no ledger, porque o desabilitado não deve depender de opacidade.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/components/ActionButton.tsx src/components/ActionButton.test.tsx src/components/formStyles.ts
/opt/homebrew/bin/git commit -m "feat: add ActionButton with focus-keeping busy state and visible disabled reason

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: `ConfirmButton` (só dashboard)

**Files:** `src/components/ConfirmButton.tsx` e `src/components/ConfirmButton.test.tsx`.

**Interfaces — Produces:**

```ts
ConfirmButton(props: {
  label: string;
  question: string;
  onConfirm(): void | Promise<void>;
  variant?: "primary" | "secondary" | "danger";
  disabledReason?: string | null;
})
```

- O primeiro clique troca o botão por um grupo `role="group"` com rótulo igual a `question`, contendo `question`, um `ActionButton` "Confirmar" e um botão "Cancelar".
- O foco vai para "Confirmar".
- Cancelar ou Escape voltam ao botão original, com o foco nele.
- Confirmar executa `onConfirm` e, ao terminar, volta ao estado inicial.

- [ ] **Step 1: Escrever o teste, que vai falhar.**

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, render, screen } from "@testing-library/react";
import { ConfirmButton } from "./ConfirmButton";

afterEach(cleanup);

describe("ConfirmButton", () => {
  it("só executa no segundo passo", async () => {
    const onConfirm = vi.fn();
    render(<ConfirmButton label="Saiu sem atendimento" question="Marcar que a pessoa saiu sem atendimento?" onConfirm={onConfirm} />);
    fireEvent.click(screen.getByRole("button", { name: "Saiu sem atendimento" }));
    expect(onConfirm).not.toHaveBeenCalled();
    const group = screen.getByRole("group", { name: "Marcar que a pessoa saiu sem atendimento?" });
    expect(document.activeElement).toBe(screen.getByRole("button", { name: "Confirmar" }));
    await act(async () => { fireEvent.click(screen.getByRole("button", { name: "Confirmar" })); });
    expect(onConfirm).toHaveBeenCalledTimes(1);
    expect(group.isConnected).toBe(false);
  });

  it.each([ "Cancelar", "Escape" ])("%s volta ao botão com foco, sem executar", (how) => {
    const onConfirm = vi.fn();
    render(<ConfirmButton label="Gerar outro código" question="Gerar outro código? O atual deixa de valer." onConfirm={onConfirm} />);
    fireEvent.click(screen.getByRole("button", { name: "Gerar outro código" }));
    if (how === "Cancelar") fireEvent.click(screen.getByRole("button", { name: "Cancelar" }));
    else fireEvent.keyDown(screen.getByRole("button", { name: "Confirmar" }), { key: "Escape" });
    expect(onConfirm).not.toHaveBeenCalled();
    expect(document.activeElement).toBe(screen.getByRole("button", { name: "Gerar outro código" }));
  });
});
```

  Run: `npx vitest run src/components/ConfirmButton.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: Implementar.**

```tsx
// Confirmação em dois passos no próprio lugar (spec 6a §3.5), para ação
// irreversível que não pede código. Ação com código continua no SensitiveAction.
import { useEffect, useRef, useState } from "react";
import { ActionButton } from "./ActionButton";
import { secondaryButtonStyle } from "./formStyles";

export function ConfirmButton({ label, question, onConfirm, variant = "secondary", disabledReason }: {
  label: string; question: string; onConfirm(): void | Promise<void>;
  variant?: "primary" | "secondary" | "danger"; disabledReason?: string | null;
}) {
  const [ asking, setAsking ] = useState(false);
  const groupRef = useRef<HTMLDivElement | null>(null);
  const triggerRef = useRef<HTMLSpanElement | null>(null);
  const wasAsking = useRef(false);

  useEffect(() => {
    if (asking) groupRef.current?.querySelector<HTMLButtonElement>("button")?.focus();
    else if (wasAsking.current) triggerRef.current?.querySelector<HTMLButtonElement>("button")?.focus();
    wasAsking.current = asking;
  }, [ asking ]);

  if (!asking) {
    return (
      <span ref={triggerRef}>
        <ActionButton variant={variant} disabledReason={disabledReason} onClick={() => setAsking(true)}>{label}</ActionButton>
      </span>
    );
  }
  return (
    <div ref={groupRef} role="group" aria-label={question}
      onKeyDown={(e) => { if (e.key === "Escape") { e.stopPropagation(); setAsking(false); } }}
      style={{ display: "inline-flex", flexWrap: "wrap", alignItems: "center", gap: 8 }}>
      <span>{question}</span>
      <ActionButton variant={variant === "danger" ? "danger" : "primary"}
        onClick={async () => { await onConfirm(); setAsking(false); }}>Confirmar</ActionButton>
      <button type="button" style={secondaryButtonStyle} onClick={() => setAsking(false)}>Cancelar</button>
    </div>
  );
}
```

  Run: `npx vitest run src/components/ConfirmButton.test.tsx`
  Expected: PASS.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/components/ConfirmButton.tsx src/components/ConfirmButton.test.tsx
/opt/homebrew/bin/git commit -m "feat: add two-step ConfirmButton for irreversible actions without step-up

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: `ActionMessage` (só dashboard)

**Files:** `src/lib/useActionMessage.ts`, `src/components/ActionMessage.tsx` e `src/components/ActionMessage.test.tsx`.

**Interfaces — Produces:**
- `useActionMessage(): { message: { kind: "ok" | "error"; text: string } | null; start(): void; ok(text: string): void; fail(text: string): void }`
  - `start()` limpa a mensagem;
  - `ok` e `fail` definem a mensagem.
- `ActionMessage({ message }: { message: … | null })`
  - renderiza nada quando a mensagem é nula;
  - `ok` vira `<p role="status" tabIndex={-1}>`; `error` vira `<p role="alert" tabIndex={-1}>`;
  - a mensagem recebe o foco quando aparece.

- [ ] **Step 1: Escrever o teste, que vai falhar.**

```tsx
import { afterEach, describe, expect, it } from "vitest";
import { act, cleanup, fireEvent, render, screen } from "@testing-library/react";
import { ActionMessage } from "./ActionMessage";
import { useActionMessage } from "../lib/useActionMessage";

afterEach(cleanup);

function Harness() {
  const m = useActionMessage();
  return (
    <div>
      <button onClick={() => { m.start(); m.ok("Pedido de agendamento criado"); }}>a</button>
      <button onClick={() => m.start()}>b</button>
      <button onClick={() => { m.start(); m.fail("Não deu para encerrar"); }}>c</button>
      <ActionMessage message={m.message} />
    </div>
  );
}

describe("ActionMessage", () => {
  it("mostra junto da ação, recebe o foco e some quando outra ação começa", () => {
    render(<Harness />);
    fireEvent.click(screen.getByRole("button", { name: "a" }));
    const ok = screen.getByRole("status");
    expect(ok.textContent).toBe("Pedido de agendamento criado");
    expect(document.activeElement).toBe(ok);
    fireEvent.click(screen.getByRole("button", { name: "b" }));
    expect(screen.queryByRole("status")).toBeNull();
    act(() => { fireEvent.click(screen.getByRole("button", { name: "c" })); });
    expect(screen.getByRole("alert").textContent).toBe("Não deu para encerrar");
  });
});
```

  Run: `npx vitest run src/components/ActionMessage.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: Implementar.**

```ts
// src/lib/useActionMessage.ts — estado da mensagem de uma ação (spec 6a §3.4).
import { useCallback, useState } from "react";

export type ActionMessageValue = { kind: "ok" | "error"; text: string } | null;

export function useActionMessage() {
  const [ message, setMessage ] = useState<ActionMessageValue>(null);
  const start = useCallback(() => setMessage(null), []);
  const ok = useCallback((text: string) => setMessage({ kind: "ok", text }), []);
  const fail = useCallback((text: string) => setMessage({ kind: "error", text }), []);
  return { message, start, ok, fail };
}
```

```tsx
// src/components/ActionMessage.tsx — a mensagem fica logo abaixo das ações
// que a geraram e recebe o foco, para o resultado não se perder.
import { useEffect, useRef } from "react";
import type { ActionMessageValue } from "../lib/useActionMessage";

export function ActionMessage({ message }: { message: ActionMessageValue }) {
  const ref = useRef<HTMLParagraphElement | null>(null);
  useEffect(() => { if (message) ref.current?.focus(); }, [ message ]);
  if (!message) return null;
  return (
    <p ref={ref} tabIndex={-1} role={message.kind === "ok" ? "status" : "alert"}
      style={{ margin: "8px 0 0", fontSize: 13, color: message.kind === "ok" ? "var(--ok)" : "var(--down)" }}>
      {message.text}
    </p>
  );
}
```

  Run: `npx vitest run src/components/ActionMessage.test.tsx`
  Expected: PASS.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/lib/useActionMessage.ts src/components/ActionMessage.tsx src/components/ActionMessage.test.tsx
/opt/homebrew/bin/git commit -m "feat: add ActionMessage that sits next to its action and clears on the next one

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Ação sensível sempre diz o que acontece (só dashboard)

**Files:** `src/components/SensitiveAction.tsx`, `src/modules/Protocols.tsx` e `src/modules/Protocols.test.tsx` (ou o teste existente que cobre a confirmação de protocolo; ache-o com `grep -rln "Ativar" src/modules/*.test.tsx`).

- [ ] **Step 1: Escrever o teste, que vai falhar.** No teste de Protocolos, com o helper existente que abre a confirmação de uma ação:

```tsx
it.each([
  [ "Ativar", /passa a valer para toda a cidade/ ],
  [ "Publicar", /fica pronta para ativação/ ],
  [ "Aposentar", /não pode mais ser ativada/ ]
])("a confirmação de %s diz o que acontece", async (label, text) => {
  // abrir a confirmação de `label` como os testes existentes do arquivo
  expect(await screen.findByText(text)).toBeTruthy();
});
```

  Run: `npx vitest run src/modules/Protocols.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: Implementar.**
  - Em `SensitiveAction.tsx`, `description?: ReactNode` passa a `description: ReactNode`. Rode `npx tsc --noEmit`: só `Protocols.tsx` deve acusar, porque as demais chamadas já passam `description`.
  - Em `Protocols.tsx`, acima de `revertDescription`:

```ts
function consequence(kind: LifecycleKind, revertTarget: string | null): string {
  switch (kind) {
    case "submit":   return "A versão sai do rascunho e não pode mais ser editada. Os revisores passam a poder assinar a publicação.";
    case "sign":     return "Sua assinatura conta para o próximo passo desta versão. Ela fica registrada com seu nome.";
    case "publish":  return "A versão fica pronta para ativação, mas ainda não vale para a cidade.";
    case "activate": return "A versão passa a valer para toda a cidade. A versão em uso deste protocolo sai de uso.";
    case "retire":   return "A versão não pode mais ser ativada. Isso não pode ser desfeito.";
    case "revert":   return revertDescription(revertTarget);
  }
}
```

  - Use `description={consequence(pending.action.kind, pending.revertTargetVersion)}`.
  - Importe `type LifecycleKind` de `../lib/protocolLifecycle`. Confira os nomes de `kind` no arquivo; são `submit`, `sign`, `publish`, `activate`, `retire` e `revert`.

  Run: `npx tsc --noEmit && npx vitest run 2>&1 | tail -3`
  Expected: verde.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/components/SensitiveAction.tsx src/modules/Protocols.tsx src/modules/Protocols.test.tsx
/opt/homebrew/bin/git commit -m "feat: require a consequence text on every sensitive action

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Fim das reticências (A-12; dashboard e admin)

**Files:** `src/components/datatable.css`, `src/components/KeyValue.tsx`, `src/modules/Overview.tsx` e `src/theme/usability-guards.test.ts` (novo), nos dois apps.

- [ ] **Step 1: Escrever a guarda, que vai falhar.**

```ts
// Guardas de usabilidade (spec 6a §3.7/§3.8).
import { describe, expect, it } from "vitest";
import { readFileSync, readdirSync, statSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join, relative } from "node:path";

const src = join(dirname(fileURLToPath(import.meta.url)), "..");
function sources(base: string): string[] {
  return readdirSync(base).flatMap((name) => {
    const full = join(base, name);
    if (statSync(full).isDirectory()) return name === "vendor" ? [] : sources(full);
    return /\.(tsx?|css)$/.test(name) && !/\.test\./.test(name) ? [full] : [];
  });
}

describe("reticências só com o texto inteiro em title", () => {
  it("nenhum textOverflow ellipsis em elemento sem title", () => {
    const offenders: string[] = [];
    for (const f of sources(src)) {
      const code = readFileSync(f, "utf8");
      for (const m of code.matchAll(/textOverflow:\s*"ellipsis"|text-overflow:\s*ellipsis/g)) {
        const tagStart = code.lastIndexOf("<", m.index);
        const tag = code.slice(tagStart, code.indexOf(">", m.index));
        if (f.endsWith(".css") || !/\btitle=/.test(tag)) offenders.push(`${relative(src, f)}@${m.index}`);
      }
    }
    expect(offenders).toEqual([]);
  });
});
```

  Run: `npx vitest run src/theme/usability-guards.test.ts`
  Expected: FAIL, acusando `datatable.css`, `KeyValue.tsx` e `Overview.tsx`.

- [ ] **Step 2: Implementar.**
  - **`datatable.css`.** A primeira linha passa a:

```css
.dt-cell { min-width: 0; white-space: normal; overflow-wrap: anywhere; }
```

  Remova da regra `@media` os `overflow: visible; text-overflow: clip;`, que ficaram redundantes, e mantenha o resto.
  - **`KeyValue.tsx`.** Nos dois `span`, troque `whiteSpace: "nowrap", overflow: "hidden", textOverflow: "ellipsis"` por `overflowWrap: "anywhere"`.
  - **`Overview.tsx`.** Nos três trechos, mantenha o corte, porque são listas de resumo na Início, e acrescente `title={…}` com o mesmo texto renderizado no `span`. Por exemplo, `title={p.name}`; leia cada trecho para achar a expressão.
  - Se um teste existente esperar `nowrap` ou `ellipsis`, registre o `Ruling` e atualize a asserção.

  Run: `npx vitest run 2>&1 | tail -3`
  Expected: verde nos dois apps.

- [ ] **Step 3: Commit** em cada app.

```bash
/opt/homebrew/bin/git add src/components/datatable.css src/components/KeyValue.tsx src/modules/Overview.tsx src/theme/usability-guards.test.ts
/opt/homebrew/bin/git commit -m "style: wrap table and key-value text instead of truncating with ellipsis

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Alvos de toque abaixo de 1024px (dashboard e admin)

**Files:** `src/theme/responsive.css` e `src/theme/usability-guards.test.ts`, nos dois apps.

- [ ] **Step 1: Escrever a guarda, que vai falhar.** Acrescente ao `usability-guards.test.ts`:

```ts
describe("alvos de toque abaixo de 1024px", () => {
  it("responsive.css dá 44px a botões, abas, selects e campos, e 24px a rádio e caixa", () => {
    const css = readFileSync(join(src, "theme/responsive.css"), "utf8");
    const block = /\/\* alvos de toque \*\/([\s\S]*?)\/\* fim dos alvos \*\//.exec(css)?.[1] ?? "";
    expect(block).toMatch(/button,[\s\S]*\[role="tab"\][\s\S]*select[\s\S]*min-height:\s*44px/);
    expect(block).toMatch(/input\[type="checkbox"\],\s*input\[type="radio"\]\s*\{[^}]*width:\s*24px[^}]*height:\s*24px/);
    expect(block).toMatch(/label:has\(> input\[type="checkbox"\]\),\s*label:has\(> input\[type="radio"\]\)\s*\{[^}]*min-height:\s*44px/);
  });
});
```

  Run: `npx vitest run src/theme/usability-guards.test.ts`
  Expected: FAIL.

- [ ] **Step 2: Implementar.** Dentro do `@media (max-width: 1023px)` existente em `responsive.css`:

```css
  /* alvos de toque */
  button, [role="tab"], select, input:not([type="checkbox"]):not([type="radio"]) { min-height: 44px; }
  input[type="checkbox"], input[type="radio"] { width: 24px; height: 24px; }
  label:has(> input[type="checkbox"]), label:has(> input[type="radio"]) {
    display: inline-flex; align-items: center; gap: 8px; min-height: 44px;
  }
  /* fim dos alvos */
```

  A guarda `css-guards.test.ts` limita `!important`; este bloco não usa nenhum.

  Run: `npx vitest run 2>&1 | tail -3`
  Expected: verde nos dois apps.

- [ ] **Step 3: Commit** em cada app.

```bash
/opt/homebrew/bin/git add src/theme/responsive.css src/theme/usability-guards.test.ts
/opt/homebrew/bin/git commit -m "style: enforce 44px touch targets below 1024px

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Varredura, capturas, revisão e publicação

- [ ] **Step 1: Suítes.** Nos dois worktrees, rode `npx vitest run`, `npx tsc --noEmit` e `npm run build`. Expected: tudo verde.

- [ ] **Step 2: Varredura e capturas "depois".**
  - Suba o Vite de cada worktree numa porta livre, conferindo em `vite.config.ts` o proxy e a origem.
  - Se a API recusar a origem, use o contorno do subprojeto 5: `ctx.route` com `route.fetch` para trocar o `Origin` pelo do container. Veja `scratchpad/capture/maint.mjs`.
  - Rode `sweep6.mjs` contra eles. Expected: zero elementos cortados sem `title` em todas as telas, em 1024 e 1280.
  - Também em 768, confira nas capturas:
    - os alvos de 44px;
    - a quebra de UUID e e-mail nas células (Review Focus 4);
    - nenhum layout quebrado pelos `min-height`.

- [ ] **Step 3: Revisão final** do branch dos dois apps pelo revisor mais capaz, com o Review Focus deste plano e a seção 3 da spec. Corrija os achados Critical e Important com TDD e registre-os no ledger.

- [ ] **Step 4: Aprovação do usuário**, com `evidencias/2026-10-12-usabilidade/6a/comparacao.html` mostrando antes e depois e a tabela da varredura.

- [ ] **Step 5: Merge e push.**
  1. Confira `origin/main..main` em cada repo e avise a sessão dona do dashboard (as sessões dos módulos 19/20) antes do merge.
  2. Faça `merge --ff-only` e `push` no dashboard, no admin e no docs (spec e plano).
  3. Remova os worktrees.
  4. Rode `docker compose restart dashboard admin`, só se `package.json` tiver mudado.
  5. Confira as portas 5175 e a do admin.
