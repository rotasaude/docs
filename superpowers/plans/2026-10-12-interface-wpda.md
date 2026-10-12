# Ciclo de interface — subprojeto 4: wpda — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Levar o wpda à linguagem gov.br com os tokens do contracts e a fonte Rawline. O wpda passa a ter:
- tema claro ou escuro seguindo o celular, e alto contraste;
- barra compacta de acessibilidade;
- topo por cidade, com reserva "Rota Saúde";
- botão "Ligar 192";
- correções AA;
- nível do resultado com cor e forma;
- bloqueio do fluxo fora do celular, com o relatório sempre aberto.

Fora do bloqueio, nenhuma funcionalidade muda.

**Architecture:**
- **Tokens:** copiados com trava, como nos painéis.
- **Cores:** só os valores de cor trocam; tamanhos e espaçamentos inline ficam como estão.
- **`App`:** passa a renderizar a `A11yBar` no topo de tudo e sobrepõe a página de acessibilidade com `hidden` no conteúdo, para o fluxo não desmontar nem perder o passo. Escolhe entre `Report` (com token), `Flow` (no celular) e `PhoneOnly`.
- **`Screen`, `Report` e `PhoneOnly`:** ganham o `CityHeader` e o `EmergencyCall`.

**Tech Stack:** React 18, Vite 5, Vitest 2, Testing Library, jest-dom e `@playwright/test` (novo, só dev).

**Spec:** `docs/superpowers/specs/2026-10-12-interface-wpda-design.md`

## Global Constraints

- **Testes:** os 35 arquivos atuais passam **sem asserção editada**. A única mudança na infraestrutura de teste é o stub de celular no `src/test/setup.ts` e o `exclude` de `e2e/**` no Vitest.
- **Tamanhos:** os inline ficam como estão (`minHeight`, `fontSize`, `padding`). Os testes leem esses valores.
- **Textos que não mudam:** "Seu resultado", "Carregando…", o nível ("alta"), os nomes dos títulos e a navegação "Conta" com "Avisos" e "Sair".
- **Cores:** sempre `var(--papel)` sem reserva. Os nomes corrigidos são `--warn-bg` e `--accent-bg`. Texto sobre `--accent` usa `--on-accent`, e sobre `--down`, `--on-tone`.
- **Tema:**
  - chave `rotasaude:wpda-theme`, com valores `auto` ou `alto-contraste`;
  - o efetivo é `govbr`, `govbr-escuro` ou `alto-contraste`;
  - `<html data-theme="govbr">` é o padrão.
- **Celular:** `isPhone` é verdadeiro quando `Math.min(screen.width, screen.height) < 600` e `matchMedia("(pointer: coarse)").matches`.
- **Commits e git:** Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Use `/opt/homebrew/bin/git` e caminhos explícitos no `add`.
- **Worktree:** `apps/wpda/.claude/worktrees/ui-wpda`, no branch `ui/wpda`, com `npm ci`.

## Review Focus

1. **Cidadão a meio da triagem toca em "Acessibilidade" e depois em "Voltar".** Ele espera voltar à mesma pergunta, sem perder o progresso. A página é sobreposta e o conteúdo fica `hidden`, sem desmontar. A Task 4 testa isso.
2. **Celular troca para o modo escuro com a tela aberta.** A pessoa espera o tema mudar na hora, sem recarregar. Há um ouvinte de `change` no `matchMedia`. A Task 2 testa isso.
3. **Link do relatório aberto no computador da unidade.** Quem abre espera ver o relatório, nunca a tela "Abra no celular". O ramo `token` vem antes do `isPhone`. A Task 5 e o Playwright testam isso.
4. **Botão desabilitado** (por exemplo "Continuar" com o campo vazio). A pessoa espera que ele pareça claramente inativo sem depender de opacidade, e que continue sem agir. O atributo `disabled` fica, e só muda a cor. A Task 3 testa isso.
5. **Nível escrito pelo autor com acento ou maiúscula** ("Média", "URGENTE"). A pessoa espera a cor e a forma certas, com o texto do nível intacto. A Task 6 testa isso.

---

## File Structure (todos os caminhos são relativos a `apps/wpda/`)

| Arquivo | Responsabilidade |
|---|---|
| `scripts/tokens-sync.mjs`, `src/theme/vendor/**`, `src/theme/vendor.lock.test.ts` | Cópia travada dos tokens (igual aos painéis) |
| `src/theme/global.css` | Reescrito: tokens, `body`, foco, links e forma do nível |
| `src/theme/tokens.ts` | **Apagado** (morto) |
| `src/theme/theme.ts`, `theme.test.ts`, `theme.inline.test.ts` | Tema automático e alto contraste |
| `index.html` | `data-theme="govbr"` e script de tema |
| `src/test/setup.ts`, `vitest.config.ts` | Stub de celular e `exclude` de `e2e` |
| `src/modules/citizen/a11yStore.ts` | Estado "página de acessibilidade aberta" |
| `src/modules/citizen/A11yBar.tsx`, `A11yPage.tsx`, `CityHeader.tsx`, `EmergencyCall.tsx`, `PhoneOnly.tsx` e testes | Componentes novos |
| `src/modules/citizen/ui.tsx` | Cores, desabilitado sem opacidade, `aria-describedby` do erro, `CityHeader` e `EmergencyCall` |
| `src/modules/Report.tsx` | `CityHeader`, `EmergencyCall`, forma e cor do nível |
| `src/lib/tier.ts`, `src/lib/device.ts` e testes | `tierTone` e `isPhone` |
| `src/App.tsx`, `src/App.gate.test.tsx` | Barra, sobreposição e bloqueio |
| 18 arquivos com cor de reserva | Troca de cor |
| `src/theme/guards.test.ts` | Guardas de fonte |
| `playwright.config.ts`, `e2e/device-gate.spec.ts`, `package.json` | Teste por dispositivo |

---

### Task 0: Preparação

- [ ] **Step 1: Avisar e criar o worktree.**
  1. Rode `ListAgents`. Se houver uma sessão dona do wpda, mande a ela: "ciclo de interface, subprojeto 4: worktree `ui/wpda` no wpda; troca de cores, barra de acessibilidade, topo por cidade, 192 como link, bloqueio fora do celular; as telas e o comportamento não mudam. Spec: `docs/superpowers/specs/2026-10-12-interface-wpda-design.md`."
  2. Crie o worktree:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude && /opt/homebrew/bin/git -C apps/wpda worktree add -q .claude/worktrees/ui-wpda -b ui/wpda origin/main && (cd apps/wpda/.claude/worktrees/ui-wpda && npm ci --silent)
```

- [ ] **Step 2: Medir a linha de base.** Rode `npx vitest run` e anote as contagens. Tudo tem que estar verde.

- [ ] **Step 3: Descobrir como entrar como cidadão em dev, para as capturas.**
  - Procure o código do SMS de dev, por exemplo com `grep -rn "OTP\|otp" apps/api/app/services apps/api/config/environments/development.rb`, ou no log do `api-dev`/`worker-dev` depois de pedir um código.
  - Anote o método no ledger. Se não houver um jeito sem SMS real, as capturas logadas ficam só nas telas públicas (celular, código e relatório) e registre o motivo.

- [ ] **Step 4: Capturas "antes".**
  - Script Playwright no scratchpad, com viewport 360 × 800 e `isMobile: true`, contra `http://curitiba.localhost:5176/wpda/`.
  - Telas: celular, código, e com login (se o Step 3 permitir) pessoas, pergunta e avisos; mais o relatório (`?token=` de uma triagem real da semente, se houver; senão um inválido) e o desktop em 1280 × 800.
  - Salve em `rota-saude/evidencias/2026-10-12-wpda/antes/`.

---

### Task 1: Tokens, CSS global e infraestrutura de teste

**Files:**
- Create: `scripts/tokens-sync.mjs` (copie byte a byte de `apps/admin/scripts/tokens-sync.mjs`), `src/theme/vendor/**` (via sync), `src/theme/vendor.lock.test.ts` (copie byte a byte de `apps/admin/src/theme/vendor.lock.test.ts`)
- Rewrite: `src/theme/global.css`
- Delete: `src/theme/tokens.ts`
- Modify: `src/test/setup.ts`, `vitest.config.ts`, `src/main.tsx` (sem mudança de import; o `global.css` continua sendo importado)
- Modify: `package.json` (acrescente `@types/node` em `devDependencies` se ainda não tiver, para o teste de lock)

- [ ] **Step 1: Copiar o teste do lock e conferir que falha.**

  Run: `npx vitest run src/theme/vendor.lock.test.ts`
  Expected: FAIL com ENOENT no `tokens.lock`.

- [ ] **Step 2: Copiar o script e sincronizar.**

```bash
cp ../../../../admin/scripts/tokens-sync.mjs scripts/tokens-sync.mjs
GIT=/opt/homebrew/bin/git node scripts/tokens-sync.mjs tokens-v1.1.0 --contracts /Users/eduardovrocha/Development/ioit.solutions/rota-saude/contracts
npx vitest run src/theme/vendor.lock.test.ts
```

  Expected: 10 arquivos copiados e o lock passando nos 3 testes. Se o `tsc` reclamar de `node:fs`, rode `npm install --save-dev @types/node@^22`.

- [ ] **Step 3: Reescrever `src/theme/global.css`.**

```css
@import "./vendor/tokens.css";

/* Globals do wpda (canal web do cidadão). Papéis e temas vêm de
   vendor/tokens.css (contracts/design-tokens); o tema ativo é o data-theme
   do <html> (theme.ts). Tamanhos ficam inline nos componentes. */

* { box-sizing: border-box; }

html, body, #root { margin: 0; padding: 0; min-height: 100%; }

body {
  background: var(--bg);
  color: var(--ink);
  font-family: var(--font-sans);
  font-size: 18px;
  line-height: 1.45;
  -webkit-font-smoothing: antialiased;
}

button, input, select, textarea { font-family: inherit; }

/* Foco visível em todo elemento focável (WCAG 2.4.7 / 2.4.11). */
:focus-visible { outline: 3px solid var(--focus); outline-offset: 2px; }

a { color: var(--accent); }

/* Forma do nível de risco (tierTone), fora do textContent e com texto
   alternativo vazio: o leitor de tela lê só o rótulo. */
[data-glyph]::before { margin-right: 6px; }
[data-glyph="square"]::before   { content: "■" / ""; }
[data-glyph="triangle"]::before { content: "▲" / ""; }
[data-glyph="circle"]::before   { content: "●" / ""; }

/* Visualmente escondido até receber foco ("Ir para o conteúdo"). */
.skip-link { position: absolute; left: -9999px; }
.skip-link:focus { position: static; left: auto; }
```

  Apague `src/theme/tokens.ts` com `git rm`. Antes, rode `grep -rn "theme/tokens" src`; o esperado é nenhum resultado.

- [ ] **Step 4: Stub de celular e `exclude` de `e2e`.**

  Acrescente ao fim do `src/test/setup.ts`:

```ts
// Ambiente padrão dos testes: um celular em modo claro (o wpda bloqueia o
// fluxo fora do celular e o tema segue prefers-color-scheme). Testes de
// bloqueio e de tema sobrescrevem estes valores.
import { beforeEach } from "vitest";

export function stubDevice({ width = 390, height = 844, coarse = true, dark = false } = {}) {
  Object.defineProperty(window, "screen", { configurable: true, value: { width, height } });
  window.matchMedia = ((query: string) => ({
    matches: (query.includes("pointer: coarse") && coarse) || (query.includes("prefers-color-scheme: dark") && dark),
    media: query, onchange: null,
    addEventListener: () => {}, removeEventListener: () => {},
    addListener: () => {}, removeListener: () => {}, dispatchEvent: () => false
  })) as unknown as typeof window.matchMedia;
}

beforeEach(() => stubDevice());
```

  No `vitest.config.ts`, importe `configDefaults` de `vitest/config` e, dentro de `test`, acrescente `exclude: [ ...configDefaults.exclude, "e2e/**" ],`.

- [ ] **Step 5: Rodar a suíte inteira.**

  Run: `npx vitest run 2>&1 | tail -4 && npx tsc --noEmit -p .`
  Expected: tudo verde, só com os 3 testes do lock a mais.

- [ ] **Step 6: Commit.**

```bash
/opt/homebrew/bin/git add scripts/tokens-sync.mjs src/theme package.json package-lock.json src/test/setup.ts vitest.config.ts
/opt/homebrew/bin/git commit -m "feat: vendor design tokens and gov.br base styles in the wpda

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Tema automático e alto contraste

**Files:**
- Create: `src/theme/theme.ts`, `src/theme/theme.test.ts`, `src/theme/theme.inline.test.ts`
- Modify: `index.html`, `src/main.tsx`

**Interfaces:**
- Produces:
  - `THEME_KEY`
  - `type WpdaThemeChoice = "auto" | "alto-contraste"`
  - `getChoice(): WpdaThemeChoice`
  - `effectiveTheme(): "govbr" | "govbr-escuro" | "alto-contraste"`
  - `applyTheme(): void`
  - `toggleHighContrast(): void`
  - `watchSystemTheme(): () => void`
  - `useThemeChoice(): WpdaThemeChoice`

- [ ] **Step 1: Escrever os testes, que vão falhar.**

`src/theme/theme.test.ts`:
```ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, renderHook } from "@testing-library/react";
import { stubDevice } from "../test/setup";
import { THEME_KEY, applyTheme, effectiveTheme, getChoice, toggleHighContrast, useThemeChoice, watchSystemTheme } from "./theme";

beforeEach(() => { localStorage.clear(); if (getChoice() !== "auto") toggleHighContrast(); localStorage.clear(); });
afterEach(() => vi.restoreAllMocks());

describe("tema do wpda", () => {
  it("auto segue o celular: claro → govbr, escuro → govbr-escuro", () => {
    stubDevice({ dark: false });
    expect(effectiveTheme()).toBe("govbr");
    stubDevice({ dark: true });
    expect(effectiveTheme()).toBe("govbr-escuro");
    applyTheme();
    expect(document.documentElement.dataset.theme).toBe("govbr-escuro");
  });

  it("alto contraste vence o modo escuro e volta ao automático", () => {
    stubDevice({ dark: true });
    toggleHighContrast();
    expect(getChoice()).toBe("alto-contraste");
    expect(document.documentElement.dataset.theme).toBe("alto-contraste");
    expect(localStorage.getItem(THEME_KEY)).toBe("alto-contraste");
    toggleHighContrast();
    expect(getChoice()).toBe("auto");
    expect(document.documentElement.dataset.theme).toBe("govbr-escuro");
  });

  it("troca do modo do celular com a tela aberta reaplica o tema", () => {
    let listener: (() => void) | undefined;
    stubDevice({ dark: false });
    const mm = window.matchMedia;
    window.matchMedia = ((q: string) => ({ ...mm(q), addEventListener: (_: string, l: () => void) => { listener = l; } })) as typeof window.matchMedia;
    const stop = watchSystemTheme();
    stubDevice({ dark: true });
    listener!();
    expect(document.documentElement.dataset.theme).toBe("govbr-escuro");
    stop();
  });

  it("armazenamento quebrado: alterna em memória sem lançar", () => {
    vi.spyOn(Storage.prototype, "getItem").mockImplementation(() => { throw new Error("x"); });
    vi.spyOn(Storage.prototype, "setItem").mockImplementation(() => { throw new Error("x"); });
    expect(() => toggleHighContrast()).not.toThrow();
    expect(getChoice()).toBe("alto-contraste");
    toggleHighContrast();
    expect(getChoice()).toBe("auto");
  });

  it("useThemeChoice acompanha", () => {
    const { result } = renderHook(() => useThemeChoice());
    expect(result.current).toBe("auto");
    act(() => toggleHighContrast());
    expect(result.current).toBe("alto-contraste");
  });
});
```

`src/theme/theme.inline.test.ts`:
```ts
import { beforeEach, describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";
import { stubDevice } from "../test/setup";

const html = readFileSync(join(dirname(fileURLToPath(import.meta.url)), "../../index.html"), "utf8");
const inline = /<script data-theme-boot>([\s\S]*?)<\/script>/.exec(html)?.[1];

beforeEach(() => { localStorage.clear(); document.documentElement.dataset.theme = "govbr"; });

describe("script inline do tema do wpda", () => {
  it("<html> abre em govbr e o script vem antes do módulo", () => {
    expect(html).toMatch(/<html lang="pt-BR" data-theme="govbr">/);
    expect(inline).toBeTruthy();
    expect(html.indexOf("data-theme-boot")).toBeLessThan(html.indexOf('type="module"'));
  });

  it("aplica alto contraste salvo", () => {
    localStorage.setItem("rotasaude:wpda-theme", "alto-contraste");
    new Function(inline!)();
    expect(document.documentElement.dataset.theme).toBe("alto-contraste");
  });

  it("segue o modo escuro do celular quando automático", () => {
    stubDevice({ dark: true });
    new Function(inline!)();
    expect(document.documentElement.dataset.theme).toBe("govbr-escuro");
  });

  it("ignora valor inválido", () => {
    localStorage.setItem("rotasaude:wpda-theme", "<x>");
    new Function(inline!)();
    expect(document.documentElement.dataset.theme).toBe("govbr");
  });
});
```

  Run: `npx vitest run src/theme/theme.test.ts src/theme/theme.inline.test.ts`
  Expected: FAIL.

- [ ] **Step 2: Implementar `src/theme/theme.ts`.**

```ts
// Tema do wpda (spec do subprojeto 4, §3.2): "auto" segue o claro/escuro do
// celular (govbr / govbr-escuro); "alto-contraste" vence o modo do celular.
// Salvo no navegador; se o armazenamento falhar, vale a escolha em memória.
import { useSyncExternalStore } from "react";

export const THEME_KEY = "rotasaude:wpda-theme";
export type WpdaThemeChoice = "auto" | "alto-contraste";
export type WpdaTheme = "govbr" | "govbr-escuro" | "alto-contraste";

const listeners = new Set<() => void>();
let memory: WpdaThemeChoice | null = null;

function read(): string | null {
  try { return window.localStorage.getItem(THEME_KEY); } catch { return null; }
}

function write(v: WpdaThemeChoice): void {
  try { window.localStorage.setItem(THEME_KEY, v); } catch { /* só em memória */ }
}

export function getChoice(): WpdaThemeChoice {
  if (memory) return memory;
  return read() === "alto-contraste" ? "alto-contraste" : "auto";
}

function prefersDark(): boolean {
  try { return window.matchMedia("(prefers-color-scheme: dark)").matches; } catch { return false; }
}

export function effectiveTheme(): WpdaTheme {
  if (getChoice() === "alto-contraste") return "alto-contraste";
  return prefersDark() ? "govbr-escuro" : "govbr";
}

export function applyTheme(): void {
  document.documentElement.dataset.theme = effectiveTheme();
}

export function toggleHighContrast(): void {
  const next: WpdaThemeChoice = getChoice() === "alto-contraste" ? "auto" : "alto-contraste";
  memory = next;
  write(next);
  applyTheme();
  listeners.forEach((l) => l());
}

// Reaplica quando a pessoa troca o modo do celular com a tela aberta.
export function watchSystemTheme(): () => void {
  let mql: MediaQueryList | null = null;
  try { mql = window.matchMedia("(prefers-color-scheme: dark)"); } catch { return () => {}; }
  const onChange = () => applyTheme();
  mql.addEventListener("change", onChange);
  return () => mql?.removeEventListener("change", onChange);
}

function subscribe(cb: () => void): () => void {
  listeners.add(cb);
  return () => { listeners.delete(cb); };
}

export function useThemeChoice(): WpdaThemeChoice {
  return useSyncExternalStore(subscribe, getChoice);
}
```

  O teste "armazenamento quebrado" alterna em memória porque `memory` tem prioridade sobre o armazenamento. O `beforeEach` dos testes de tema volta a `auto`, e é por isso que o teste começa conferindo `getChoice()`.

- [ ] **Step 3: Ajustar o `index.html` e o `main.tsx`.**
  - No `index.html`, troque `<html lang="pt-BR">` por `<html lang="pt-BR" data-theme="govbr">`. Logo depois da meta `viewport`, acrescente:

```html
    <script data-theme-boot>try{var c=localStorage.getItem("rotasaude:wpda-theme");var t=c==="alto-contraste"?"alto-contraste":(window.matchMedia&&window.matchMedia("(prefers-color-scheme: dark)").matches?"govbr-escuro":"govbr");document.documentElement.setAttribute("data-theme",t)}catch(e){}</script>
```

  - No `src/main.tsx`, acrescente `import { applyTheme, watchSystemTheme } from "./theme/theme";` e, antes do `createRoot`, chame `applyTheme(); watchSystemTheme();`.

  Run: `npx vitest run src/theme/ && npx vitest run 2>&1 | tail -3`
  Expected: PASS e a suíte verde.

- [ ] **Step 4: Commit.**

```bash
/opt/homebrew/bin/git add src/theme/theme.ts src/theme/theme.test.ts src/theme/theme.inline.test.ts index.html src/main.tsx
/opt/homebrew/bin/git commit -m "feat: follow the phone's light or dark mode and add high contrast in the wpda

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Cores nos componentes base e nos 18 arquivos, mais as guardas

**Files:**
- Modify: `src/modules/citizen/ui.tsx` e os demais arquivos com cor de reserva
- Create: `src/theme/guards.test.ts`, `src/modules/citizen/ui.a11y.test.tsx`

- [ ] **Step 1: Escrever os testes, que vão falhar.**

`src/theme/guards.test.ts`:
```ts
// Guardas do wpda (spec do subprojeto 4, §5): cor só por papel do tema,
// sem reserva fixa nem nome inexistente; foco nunca zerado; desabilitado
// sem depender de opacidade.
import { describe, expect, it } from "vitest";
import { readFileSync, readdirSync, statSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join, relative } from "node:path";

const src = join(dirname(fileURLToPath(import.meta.url)), "..");
function files(dir: string): string[] {
  return readdirSync(dir).flatMap((n) => {
    const f = join(dir, n);
    if (statSync(f).isDirectory()) return n === "vendor" || n === "test" ? [] : files(f);
    return /\.(tsx?)$/.test(n) && !/\.test\./.test(n) ? [f] : [];
  });
}
const all = files(src).map((f) => ({ f: relative(src, f), code: readFileSync(f, "utf8") }));
const offenders = (re: RegExp) => all.filter(({ code }) => re.test(code)).map(({ f }) => f);

describe("guardas de cor e acessibilidade do wpda", () => {
  it("nenhum var(--x, reserva)", () => expect(offenders(/var\(--[\w-]+\s*,/)).toEqual([]));
  it("nenhum branco fixo", () => expect(offenders(/"#fff"|"white"/)).toEqual([]));
  it("nenhum nome de variável inexistente", () => expect(offenders(/--warnBg|--accent-soft/)).toEqual([]));
  it("nenhum outline zerado", () => expect(offenders(/outline:\s*"none"/)).toEqual([]));
  it("desabilitado não depende de opacidade", () => expect(offenders(/opacity:\s*(props\.)?disabled/)).toEqual([]));
});
```

`src/modules/citizen/ui.a11y.test.tsx`:
```tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { BigButton, Field } from "./ui";

describe("componentes base: AA", () => {
  it("botão desabilitado segue desabilitado e sem opacidade", () => {
    render(<BigButton disabled>Continuar</BigButton>);
    const b = screen.getByRole("button", { name: "Continuar" });
    expect(b).toBeDisabled();
    expect(b.style.opacity).toBe("");
    expect(b.style.minHeight).toBe("56px");
  });

  it("erro do campo entra no aria-describedby junto da dica", () => {
    render(<Field label="CPF" hint="só números" error="CPF inválido." value="" onChange={() => {}} />);
    const input = screen.getByLabelText("CPF");
    const ids = (input.getAttribute("aria-describedby") ?? "").split(" ");
    expect(ids).toHaveLength(2);
    expect(ids.map((id) => document.getElementById(id)?.textContent)).toEqual([ "só números", "CPF inválido." ]);
  });
});
```

  Run: `npx vitest run src/theme/guards.test.ts src/modules/citizen/ui.a11y.test.tsx`
  Expected: FAIL nas guardas de reserva, branco, nomes e opacidade, e nos dois testes de AA.

- [ ] **Step 2: Componentes base (`ui.tsx`).**
  - **`variants`:**

```ts
const variants = {
  primary: { background: "var(--accent)", color: "var(--on-accent)", border: "none" },
  secondary: { background: "transparent", color: "var(--ink)", border: "2px solid var(--rule2)" },
  danger: { background: "var(--down)", color: "var(--on-tone)", border: "none" }
} as const;

// Desabilitado: cor própria em vez de opacidade (contraste legível).
const disabledLook = { background: "var(--sunken)", color: "var(--ink2)", border: "2px solid var(--rule)", cursor: "default" } as const;
```

  - **`BigButton`:** o `style` vira `{ minHeight: 56, width: "100%", borderRadius: 12, fontSize: 18, fontWeight: 600, cursor: "pointer", ...variants[variant], ...(props.disabled ? disabledLook : {}), ...style }`, sem `opacity`.
  - **`Field`:**
    - crie `const errorId = \`${id}-error\`;` e `const describedBy = [ hint ? hintId : null, error ? errorId : null ].filter(Boolean).join(" ") || undefined;`;
    - o input usa `aria-describedby={describedBy}` e `border: \`2px solid ${error ? "var(--down)" : "var(--rule2)"}\``;
    - a dica ganha `color: "var(--ink2)"`;
    - o erro vira `<ErrorText id={errorId}>{error}</ErrorText>`;
    - o `ErrorText` passa a aceitar `id?: string`, repassado ao `<p>`, e usa `color: "var(--down)"`.
  - **`Toggle`:** o `label` fica com `cursor: disabled ? "default" : "pointer", color: disabled ? "var(--ink2)" : undefined`, sem `opacity`. A descrição usa `var(--ink2)`.
  - **`RadioGroup`:** o `label` da opção fica com `border: \`2px solid ${disabled ? "var(--rule)" : "var(--rule2)"}\`, color: disabled ? "var(--ink2)" : undefined`, sem `opacity`.
  - **`Screen`:**
    - no `<main>`, troque `fontFamily: "system-ui, sans-serif"` por nada (a fonte vem do `body`);
    - o `CityHeader` e o `EmergencyCall` entram na Task 4, então nesta task só troque `var(--ink3, #666)` por `var(--ink2)` no `<strong>` e no `<p>` do 192.

- [ ] **Step 3: Os demais arquivos (troca mecânica).** Num script Python, em todo `src/**/*.tsx|ts` que não seja teste nem `vendor`:
  - **Renomear:** `--warnBg` vira `--warn-bg`, e `--accent-soft` vira `--accent-bg`.
  - **Tirar a reserva:** `var\(--([\w-]+),\s*[^)]+\)` vira `var(--\1)`.
  - **Branco fixo:** em `NoticesStep.tsx:27`, `color: "#fff"` vira `color: "var(--on-accent)"`. Em `NoticesLink.tsx:43`, `color: "#fff"` vira `color: "var(--on-tone)"`.
  - **Fonte do `Report`:** em `Report.tsx`, troque `fontFamily: "system-ui, sans-serif"` por nada. O `var(--font-mono, monospace)` vira `var(--font-mono)`.

  Depois, rode `grep -rnE "var\(--[a-zA-Z-]+," src | grep -v test` (o esperado é nenhum resultado) e mostre o `git diff --stat`.

- [ ] **Step 4: Rodar.**

  Run: `npx vitest run src/theme/guards.test.ts src/modules/citizen/ui.a11y.test.tsx && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit -p .`
  Expected: PASS, e a suíte inteira verde sem teste antigo editado. Os testes de `minHeight` e `fontSize` continuam verdes porque os tamanhos não mudaram.

- [ ] **Step 5: Commit.**

```bash
/opt/homebrew/bin/git add src
/opt/homebrew/bin/git commit -m "style: use theme roles for every wpda color and fix AA in base components

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

  O `git add src` aqui é aceitável porque é o próprio worktree e só tem as mudanças desta task. Antes, confira com `git status --short` que não há arquivo estranho.

---

### Task 4: Barra de acessibilidade, topo por cidade e 192

**Files:**
- Create: `src/modules/citizen/a11yStore.ts`, `A11yBar.tsx`, `A11yPage.tsx`, `CityHeader.tsx`, `EmergencyCall.tsx`, `src/modules/citizen/chrome.test.tsx`
- Modify: `src/modules/citizen/ui.tsx` (`Screen`), `src/modules/Report.tsx`, `src/App.tsx`

**Interfaces:**
- Produces:
  - `openA11y()`, `closeA11y()`, `useA11yOpen(): boolean`
  - `<A11yBar/>`
  - `<A11yPage/>`
  - `<CityHeader name?: string; coatOfArmsUrl?: string/>`
  - `<EmergencyCall/>`

- [ ] **Step 1: Escrever os testes, que vão falhar.** `src/modules/citizen/chrome.test.tsx`:

```tsx
import { afterEach, describe, expect, it } from "vitest";
import { fireEvent, render, screen } from "@testing-library/react";
import { useState } from "react";
import { A11yBar } from "./A11yBar";
import { A11yPage } from "./A11yPage";
import { CityHeader } from "./CityHeader";
import { EmergencyCall } from "./EmergencyCall";
import { Screen } from "./ui";
import { closeA11y, useA11yOpen } from "./a11yStore";
import { getChoice, toggleHighContrast } from "../../theme/theme";

afterEach(() => { closeA11y(); if (getChoice() !== "auto") toggleHighContrast(); });

function Harness() {
  const open = useA11yOpen();
  const [ n, setN ] = useState(0);
  return (
    <>
      <A11yBar />
      {open && <A11yPage />}
      <div hidden={open}>
        <Screen title="Pergunta">
          <button type="button" onClick={() => setN(n + 1)}>passo {n}</button>
        </Screen>
      </div>
    </>
  );
}

describe("topo, acessibilidade e 192", () => {
  it("192 é um link tel:", () => {
    render(<EmergencyCall />);
    expect(screen.getByRole("link", { name: "Emergência? Ligar 192" }).getAttribute("href")).toBe("tel:192");
  });

  it("Screen traz o topo da cidade com reserva e o 192", () => {
    render(<Screen title="Seu celular">x</Screen>);
    expect(screen.getByText("Rota Saúde")).toBeInTheDocument();
    expect(screen.getByRole("link", { name: "Emergência? Ligar 192" })).toBeInTheDocument();
    expect(screen.getByRole("heading", { level: 1, name: "Seu celular" })).toBeInTheDocument();
  });

  it("CityHeader mostra o nome quando houver", () => {
    render(<CityHeader name="Curitiba" />);
    expect(screen.getByText("Curitiba")).toBeInTheDocument();
  });

  it("Alto contraste tem aria-pressed", () => {
    render(<A11yBar />);
    const b = screen.getByRole("button", { name: "Alto contraste" });
    expect(b.getAttribute("aria-pressed")).toBe("false");
    fireEvent.click(b);
    expect(b.getAttribute("aria-pressed")).toBe("true");
  });

  it("Ir para o conteúdo foca o main sem mudar o histórico", () => {
    render(<Harness />);
    const before = window.history.length;
    fireEvent.click(screen.getByRole("link", { name: "Ir para o conteúdo" }));
    expect(document.activeElement?.tagName).toBe("MAIN");
    expect(window.history.length).toBe(before);
  });

  it("Acessibilidade sobrepõe e o Voltar devolve ao mesmo passo, sem perder estado", () => {
    render(<Harness />);
    fireEvent.click(screen.getByRole("button", { name: "passo 0" }));
    fireEvent.click(screen.getByRole("link", { name: "Acessibilidade" }));
    expect(screen.getByRole("heading", { name: "Acessibilidade" })).toBeInTheDocument();
    fireEvent.click(screen.getByRole("button", { name: "Voltar" }));
    expect(screen.getByRole("button", { name: "passo 1" })).toBeVisible();
  });
});
```

  Run: `npx vitest run src/modules/citizen/chrome.test.tsx`
  Expected: FAIL, porque os módulos não existem.

- [ ] **Step 2: Implementar os componentes.**

`src/modules/citizen/a11yStore.ts`:
```ts
// Página de acessibilidade aberta? Estado global mínimo: a página é
// sobreposta ao conteúdo (que fica hidden, sem desmontar o fluxo).
import { useSyncExternalStore } from "react";

let open = false;
const listeners = new Set<() => void>();
const emit = () => listeners.forEach((l) => l());

export function openA11y(): void { open = true; emit(); }
export function closeA11y(): void { open = false; emit(); }
export function useA11yOpen(): boolean {
  return useSyncExternalStore((cb) => { listeners.add(cb); return () => { listeners.delete(cb); }; }, () => open);
}
```

`src/modules/citizen/A11yBar.tsx`:
```tsx
// Barra compacta de acessibilidade (eMAG), sempre no topo: pular para o
// conteúdo (aparece só com foco), página de acessibilidade e alto contraste.
import type { MouseEvent } from "react";
import { toggleHighContrast, useThemeChoice } from "../../theme/theme";
import { openA11y } from "./a11yStore";

function focusMain(e: MouseEvent<HTMLAnchorElement>) {
  e.preventDefault();
  const main = Array.from(document.querySelectorAll("main")).find((m) => !m.closest("[hidden]"));
  if (!main) return;
  if (!main.hasAttribute("tabindex")) main.setAttribute("tabindex", "-1");
  main.focus();
}

export function A11yBar() {
  const choice = useThemeChoice();
  return (
    <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", gap: 12, maxWidth: 520,
      margin: "0 auto", padding: "4px 16px", fontSize: 14, background: "var(--sunken)", borderBottom: "1px solid var(--rule)" }}>
      <a href="#conteudo" className="skip-link" onClick={focusMain}>Ir para o conteúdo</a>
      <a href="#acessibilidade" onClick={(e) => { e.preventDefault(); openA11y(); }} style={{ minHeight: 44, display: "inline-flex", alignItems: "center" }}>
        Acessibilidade
      </a>
      <button type="button" aria-pressed={choice === "alto-contraste"} onClick={toggleHighContrast}
        style={{ minHeight: 44, background: "none", border: "none", color: "var(--accent)", textDecoration: "underline", fontSize: 14, cursor: "pointer" }}>
        Alto contraste
      </button>
    </div>
  );
}
```

`src/modules/citizen/CityHeader.tsx`:
```tsx
// Topo por cidade (decisão 6 do ciclo): brasão e nome da prefeitura quando o
// api expuser (rotasaude/api#73); até lá, "Rota Saúde" com um ícone neutro.
export function CityHeader({ name, coatOfArmsUrl }: { name?: string; coatOfArmsUrl?: string }) {
  return (
    <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
      {coatOfArmsUrl
        ? <img src={coatOfArmsUrl} alt="" width={32} height={32} style={{ objectFit: "contain" }} />
        : (
          <span aria-hidden="true" style={{ width: 32, height: 32, borderRadius: 6, display: "inline-flex", alignItems: "center",
            justifyContent: "center", background: "var(--accent-bg)", color: "var(--accent)" }}>
            <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.8" strokeLinecap="round" strokeLinejoin="round">
              <path d="M3 21h18M5 21V9l7-5 7 5v12M9 21v-6h6v6" />
            </svg>
          </span>
        )}
      <strong style={{ fontSize: 14, color: "var(--ink2)" }}>{name ?? "Rota Saúde"}</strong>
    </div>
  );
}
```

`src/modules/citizen/EmergencyCall.tsx`:
```tsx
// "Em emergência, ligue 192" como link que liga (antes era só texto).
export function EmergencyCall() {
  return (
    <a href="tel:192" style={{ display: "flex", alignItems: "center", justifyContent: "center", gap: 8, minHeight: 48,
      marginTop: 12, borderRadius: 12, background: "var(--down-bg)", color: "var(--down)", fontSize: 16, fontWeight: 600,
      textDecoration: "none" }}>
      <svg aria-hidden="true" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
        <path d="M5 4h4l2 5-2.5 1.5a11 11 0 0 0 5 5L15 13l5 2v4a2 2 0 0 1-2 2A16 16 0 0 1 3 6a2 2 0 0 1 2-2" />
      </svg>
      Emergência? Ligar 192
    </a>
  );
}
```

`src/modules/citizen/A11yPage.tsx`:
```tsx
// Página de acessibilidade do wpda (eMAG 3.1), sobreposta ao conteúdo.
import { closeA11y } from "./a11yStore";
import { BigButton } from "./ui";
import { CityHeader } from "./CityHeader";

export function A11yPage() {
  return (
    <section aria-labelledby="a11y-title" style={{ maxWidth: 520, margin: "0 auto", padding: "0 16px 16px" }}>
      <header style={{ margin: "16px 0" }}>
        <CityHeader />
        <h1 id="a11y-title" style={{ fontSize: 24, margin: "4px 0 0" }}>Acessibilidade</h1>
      </header>
      <p>Este serviço segue a WCAG 2.2 nível AA e o eMAG 3.1.</p>
      <ul style={{ paddingLeft: 20, lineHeight: 1.6 }}>
        <li>Alto contraste no botão do topo. Sem ele, as cores seguem o modo claro ou escuro do seu celular.</li>
        <li>Botões grandes, fáceis de tocar.</li>
        <li>O nível de risco aparece por cor, forma (■ alto, ▲ moderado, ● baixo) e texto.</li>
        <li>Títulos e rótulos para leitores de tela.</li>
      </ul>
      <p>Encontrou um problema? Procure a unidade de saúde ou a prefeitura.</p>
      <BigButton variant="secondary" onClick={closeA11y}>Voltar</BigButton>
    </section>
  );
}
```

- [ ] **Step 3: Compor no `Screen`, no `Report` e no `App`.**
  - **`Screen` (`ui.tsx`):**
    - o `<main>` ganha `id="conteudo"`;
    - no `<header>`, o `<strong>Rota Saúde</strong>` vira `<CityHeader />`;
    - o `<p>Em emergência, ligue 192.</p>` vira `<EmergencyCall />`;
    - importe `CityHeader` e `EmergencyCall`. Pode haver dependência circular com o `A11yPage`, que importa `BigButton` de `ui.tsx`; ela é aceitável porque nenhum dos dois roda código no topo do módulo, mas, se o Vitest reclamar, mova o `BigButton` usado pelo `A11yPage` para um `<button>` com o mesmo estilo e registre a decisão no ledger.
  - **`Report.tsx`:**
    - no `Centered`, o `<main>` ganha `id="conteudo"` e passa a `flexDirection: "column"`;
    - dentro do `article`, o `<strong …>Rota Saúde</strong>` vira `<CityHeader />`;
    - depois do `footer` e antes dos `children`, acrescente `<EmergencyCall />`.
  - **`App.tsx`:**

```tsx
import { tokenFromUrl } from "./lib/report";
import { Report } from "./modules/Report";
import { BackHomeLink } from "./modules/BackHomeLink";
import { Flow } from "./modules/citizen/Flow";
import { A11yBar } from "./modules/citizen/A11yBar";
import { A11yPage } from "./modules/citizen/A11yPage";
import { useA11yOpen } from "./modules/citizen/a11yStore";

export function App() {
  const token = tokenFromUrl();
  const a11y = useA11yOpen();
  const content = token ? <Report token={token}><BackHomeLink /></Report> : <Flow />;
  return (
    <>
      <A11yBar />
      {a11y && <A11yPage />}
      <div hidden={a11y}>{content}</div>
    </>
  );
}
```

  O bloqueio entra na Task 5.

- [ ] **Step 4: Rodar.**

  Run: `npx vitest run src/modules/citizen/chrome.test.tsx && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit -p .`
  Expected: PASS, e a suíte verde sem teste antigo editado. O `App.test` (link do relatório dentro de `article`) e os testes de `main` continuam valendo: só há um `main` visível, porque o `A11yPage` é `section`.

- [ ] **Step 5: Commit.**

```bash
/opt/homebrew/bin/git add src/modules/citizen/a11yStore.ts src/modules/citizen/A11yBar.tsx src/modules/citizen/A11yPage.tsx src/modules/citizen/CityHeader.tsx src/modules/citizen/EmergencyCall.tsx src/modules/citizen/chrome.test.tsx src/modules/citizen/ui.tsx src/modules/Report.tsx src/App.tsx
/opt/homebrew/bin/git commit -m "feat: add accessibility bar, city header and tap-to-call 192 in the wpda

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Bloqueio fora do celular

**Files:**
- Create: `src/lib/device.ts`, `src/lib/device.test.ts`, `src/modules/citizen/PhoneOnly.tsx`, `src/App.gate.test.tsx`
- Modify: `src/App.tsx`

- [ ] **Step 1: Escrever os testes, que vão falhar.**

`src/lib/device.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import { stubDevice } from "../test/setup";
import { isPhone } from "./device";

describe("isPhone", () => {
  it.each([
    [ "celular em pé", { width: 390, height: 844, coarse: true }, true ],
    [ "celular deitado", { width: 844, height: 390, coarse: true }, true ],
    [ "tablet", { width: 768, height: 1024, coarse: true }, false ],
    [ "notebook com toque", { width: 1366, height: 768, coarse: true }, false ],
    [ "desktop", { width: 1920, height: 1080, coarse: false }, false ],
    [ "celular com mouse", { width: 390, height: 844, coarse: false }, false ]
  ])("%s", (_name, dev, expected) => {
    stubDevice(dev);
    expect(isPhone()).toBe(expected);
  });
});
```

`src/App.gate.test.tsx`:
```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { render, screen } from "@testing-library/react";
import { stubDevice } from "./test/setup";

vi.mock("./modules/citizen/Flow", () => ({ Flow: () => <main>FLUXO</main> }));
vi.mock("./modules/Report", () => ({ Report: () => <main>RELATORIO</main> }));
import { App } from "./App";

afterEach(() => window.history.replaceState(null, "", "/wpda/"));

describe("bloqueio fora do celular", () => {
  it("celular vê o fluxo", () => {
    render(<App />);
    expect(screen.getByText("FLUXO")).toBeInTheDocument();
  });

  it("desktop vê 'Abra no celular' com endereço e 192, e não o fluxo", () => {
    stubDevice({ width: 1920, height: 1080, coarse: false });
    render(<App />);
    expect(screen.getByRole("heading", { level: 1, name: "Abra no celular" })).toBeInTheDocument();
    expect(screen.getByText(/Este serviço foi feito para ser usado no celular/)).toBeInTheDocument();
    expect(screen.getByText(window.location.origin + window.location.pathname)).toBeInTheDocument();
    expect(screen.getByRole("link", { name: "Emergência? Ligar 192" })).toBeInTheDocument();
    expect(screen.queryByText("FLUXO")).toBeNull();
  });

  it("relatório (?token=) abre em qualquer tela", () => {
    stubDevice({ width: 1920, height: 1080, coarse: false });
    window.history.replaceState(null, "", "/wpda/?token=abc");
    render(<App />);
    expect(screen.getByText("RELATORIO")).toBeInTheDocument();
    expect(screen.queryByRole("heading", { name: "Abra no celular" })).toBeNull();
  });
});
```

  Run: `npx vitest run src/lib/device.test.ts src/App.gate.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: Implementar.**

`src/lib/device.ts`:
```ts
// Celular = lado menor da tela física < 600px E toque como entrada principal
// (decisão 9 do ciclo de interface). Girar o celular ou dar zoom não muda
// `screen`; diminuir a janela no desktop não libera.
export function isPhone(win: Window = window): boolean {
  const shortest = Math.min(win.screen.width, win.screen.height);
  let coarse = false;
  try { coarse = win.matchMedia("(pointer: coarse)").matches; } catch { coarse = false; }
  return shortest < 600 && coarse;
}
```

`src/modules/citizen/PhoneOnly.tsx`:
```tsx
// Fora do celular, o fluxo do cidadão não abre (decisão 9 do ciclo).
import { CityHeader } from "./CityHeader";
import { EmergencyCall } from "./EmergencyCall";

export function PhoneOnly() {
  const address = window.location.origin + window.location.pathname;
  return (
    <main id="conteudo" style={{ maxWidth: 520, margin: "0 auto", padding: "16px" }}>
      <header style={{ marginBottom: 16 }}>
        <CityHeader />
        <h1 style={{ fontSize: 24, margin: "4px 0 0" }}>Abra no celular</h1>
      </header>
      <p>Este serviço foi feito para ser usado no celular. Abra este endereço no seu celular.</p>
      <p><code style={{ fontSize: 18, userSelect: "all", overflowWrap: "anywhere" }}>{address}</code></p>
      <EmergencyCall />
    </main>
  );
}
```

  No `App.tsx`, importe `isPhone` e `PhoneOnly`. Troque a linha do `content` por:

```tsx
  const content = token
    ? <Report token={token}><BackHomeLink /></Report>
    : isPhone() ? <Flow /> : <PhoneOnly />;
```

  Run: `npx vitest run src/lib/device.test.ts src/App.gate.test.tsx && npx vitest run 2>&1 | tail -3`
  Expected: PASS, e a suíte verde. Os testes do `Flow` e do `App` continuam vendo o fluxo por causa do stub de celular no `setup.ts`.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/lib/device.ts src/lib/device.test.ts src/modules/citizen/PhoneOnly.tsx src/App.gate.test.tsx src/App.tsx
/opt/homebrew/bin/git commit -m "feat: keep the citizen flow phone-only while reports open anywhere

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Nível do resultado com cor e forma

**Files:**
- Create: `src/lib/tier.ts`, `src/lib/tier.test.ts`, `src/modules/Report.tier.test.tsx`
- Modify: `src/modules/Report.tsx`

- [ ] **Step 1: Escrever os testes, que vão falhar.**

`src/lib/tier.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import { tierTone } from "./tier";

describe("tierTone", () => {
  it.each([
    [ "alta", "down" ], [ "Alto", "down" ], [ "URGENTE", "down" ], [ "emergência", "down" ], [ "grave", "down" ],
    [ "média", "warn" ], [ "Moderada", "warn" ], [ "intermediária", "warn" ],
    [ "baixa", "ok" ], [ "Leve", "ok" ], [ "verde", "ok" ],
    [ "azul-claro", null ], [ "", null ], [ null, null ]
  ] as const)("%s → %s", (tier, tone) => expect(tierTone(tier)).toBe(tone));
});
```

`src/modules/Report.tier.test.tsx`. Copie do `Report.test.tsx` existente o `vi.mock`/`mockResolvedValue` e o fixture do relatório; o arquivo é independente e usa o mesmo `fetchReport` simulado:
```tsx
it("nível 'alta' ganha forma quadrada e o texto fica igual", async () => {
  // render <Report token="t" /> com fetchReport → { ...fixture, tier: "alta" }
  const tag = await screen.findByText("alta");
  expect(tag.getAttribute("data-glyph")).toBe("square");
  expect(tag.textContent).toBe("alta");
  expect(tag.style.color).toBe("var(--down)");
});

it("nível desconhecido fica neutro", async () => {
  // fetchReport → { ...fixture, tier: "azul" }
  const tag = await screen.findByText("azul");
  expect(tag.hasAttribute("data-glyph")).toBe(false);
  expect(tag.style.color).toBe("var(--accent)");
});
```

  Run: `npx vitest run src/lib/tier.test.ts src/modules/Report.tier.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: Implementar.**

`src/lib/tier.ts`:
```ts
// Nível do resultado → tom (spec do subprojeto 4, §4.2). O nível é texto
// livre do autor do protocolo; termos desconhecidos ficam neutros.
export type TierTone = "down" | "warn" | "ok";

const TONES: Record<string, TierTone> = {
  alta: "down", alto: "down", urgente: "down", emergencia: "down", emergente: "down", grave: "down", vermelho: "down",
  media: "warn", medio: "warn", moderada: "warn", moderado: "warn", intermediaria: "warn", amarelo: "warn",
  baixa: "ok", baixo: "ok", leve: "ok", verde: "ok"
};

export function tierTone(tier: string | null | undefined): TierTone | null {
  if (!tier) return null;
  const key = tier.normalize("NFD").replace(/[̀-ͯ]/g, "").toLowerCase().trim();
  return TONES[key] ?? null;
}

export const TONE_GLYPH: Record<TierTone, "square" | "triangle" | "circle"> = { down: "square", warn: "triangle", ok: "circle" };
```

  No `Report.tsx`, troque a etiqueta do nível por:

```tsx
{r.tier && (() => {
  const tone = tierTone(r.tier);
  return (
    <span data-glyph={tone ? TONE_GLYPH[tone] : undefined}
      style={{ display: "inline-block", padding: "2px 10px", borderRadius: 999, marginRight: 8, fontSize: 12, fontWeight: 600,
        background: tone ? `var(--${tone}-bg)` : "var(--accent-bg)", color: tone ? `var(--${tone})` : "var(--accent)" }}>
      {r.tier}
    </span>
  );
})()}
```

  Importe `tierTone` e `TONE_GLYPH` de `../lib/tier`. O tamanho continua 12px, como manda a regra dos tamanhos.

  Run: `npx vitest run src/lib/tier.test.ts src/modules/Report.tier.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: Implementar.**

`src/lib/tier.ts`:
```ts
// Nível do resultado → tom (spec do subprojeto 4, §4.2). O nível é texto
// livre do autor do protocolo; termos desconhecidos ficam neutros.
export type TierTone = "down" | "warn" | "ok";

const TONES: Record<string, TierTone> = {
  alta: "down", alto: "down", urgente: "down", emergencia: "down", emergente: "down", grave: "down", vermelho: "down",
  media: "warn", medio: "warn", moderada: "warn", moderado: "warn", intermediaria: "warn", amarelo: "warn",
  baixa: "ok", baixo: "ok", leve: "ok", verde: "ok"
};

export function tierTone(tier: string | null | undefined): TierTone | null {
  if (!tier) return null;
  const key = tier.normalize("NFD").replace(/[̀-ͯ]/g, "").toLowerCase().trim();
  return TONES[key] ?? null;
}

export const TONE_GLYPH: Record<TierTone, "square" | "triangle" | "circle"> = { down: "square", warn: "triangle", ok: "circle" };
```

  No `Report.tsx`, troque a etiqueta do nível por:

```tsx
{r.tier && (() => {
  const tone = tierTone(r.tier);
  return (
    <span data-glyph={tone ? TONE_GLYPH[tone] : undefined}
      style={{ display: "inline-block", padding: "2px 10px", borderRadius: 999, marginRight: 8, fontSize: 12, fontWeight: 600,
        background: tone ? `var(--${tone}-bg)` : "var(--accent-bg)", color: tone ? `var(--${tone})` : "var(--accent)" }}>
      {r.tier}
    </span>
  );
})()}
```

  Importe `tierTone` e `TONE_GLYPH` de `../lib/tier`. O tamanho continua 12px, como manda a regra dos tamanhos.

  Run: `npx vitest run src/lib/tier.test.ts src/modules/Report.tier.test.tsx src/modules/Report.test.tsx && npx vitest run 2>&1 | tail -3`
  Expected: PASS. O `Report.test` original continua verde, com "alta" e "prioridade: 1" intactos.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/lib/tier.ts src/lib/tier.test.ts src/modules/Report.tier.test.tsx src/modules/Report.tsx
/opt/homebrew/bin/git commit -m "feat: show the report risk level by colour and shape as well as text

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Playwright por dispositivo

**Files:**
- Create: `playwright.config.ts`, `e2e/device-gate.spec.ts`
- Modify: `package.json` (acrescente `@playwright/test` em `devDependencies` e o script `"e2e": "playwright test"`)

- [ ] **Step 1: Pedir permissão e instalar.**
  - Diga ao usuário que serão instalados `npm install --save-dev @playwright/test@<a mesma versão do maintenance>` (rode `grep '"@playwright/test"\|"playwright"' ../../../../maintenance/package.json`) e, para baixar os navegadores, `npx playwright install webkit firefox`.
  - O Chromium já está no cache, que hoje tem `chromium-1223` e `chromium-1243`. Confira se a versão casa; se não, inclua `chromium` no install.
  - Informe os tamanhos aproximados (cerca de 70 MB o WebKit e 90 MB o Firefox) e **espere o sim**.

- [ ] **Step 2: Escrever a configuração e o teste.**

`playwright.config.ts`:
```ts
// Bloqueio por dispositivo (decisão 9 do ciclo): roda contra o wpda de dev.
import { defineConfig, devices } from "@playwright/test";

const baseURL = process.env.WPDA_URL ?? "http://curitiba.localhost:5176";

export default defineConfig({
  testDir: "./e2e",
  timeout: 30_000,
  use: { baseURL, trace: "off", screenshot: "off", video: "off" },
  projects: [
    { name: "iphone", use: { ...devices["iPhone 13"] } },
    { name: "pixel", use: { ...devices["Pixel 7"] } },
    { name: "ipad", use: { ...devices["iPad Pro 11"] } },
    { name: "desktop-chrome", use: { ...devices["Desktop Chrome"] } },
    { name: "desktop-firefox", use: { ...devices["Desktop Firefox"] } }
  ]
});
```

`e2e/device-gate.spec.ts`:
```ts
import { expect, test } from "@playwright/test";

const PHONES = new Set([ "iphone", "pixel" ]);

test("o fluxo abre só no celular", async ({ page }, info) => {
  await page.goto("/wpda/");
  const blocked = page.getByRole("heading", { level: 1, name: "Abra no celular" });
  if (PHONES.has(info.project.name)) {
    await expect(page.getByRole("link", { name: "Emergência? Ligar 192" }).first()).toBeVisible();
    await expect(blocked).toHaveCount(0);
  } else {
    await expect(blocked).toBeVisible();
  }
});

test("o relatório nunca é bloqueado", async ({ page }) => {
  await page.goto("/wpda/?token=invalido");
  await expect(page.getByRole("heading", { level: 1, name: "Abra no celular" })).toHaveCount(0);
  await expect(page.getByText(/Link inválido|inválido ou expirou|Carregando/).first()).toBeVisible();
});
```

- [ ] **Step 3: Rodar contra o worktree.**
  1. Suba o wpda do worktree: `VITE_API_PROXY_TARGET=http://localhost:3030 npx vite --port 5186 --strictPort --host 127.0.0.1`, em segundo plano. Use uma porta livre; se a 5186 estiver ocupada, use 5187.
  2. Rode `WPDA_URL=http://curitiba.localhost:5186 npx playwright test`.

  Expected: 10 testes passando (2 testes × 5 perfis).

- [ ] **Step 4: Commit.**

```bash
/opt/homebrew/bin/git add playwright.config.ts e2e/device-gate.spec.ts package.json package-lock.json
/opt/homebrew/bin/git commit -m "test: verify the phone-only gate on real device profiles

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Capturas, card do `wpdaLook`, revisão e publicação

- [ ] **Step 1: Capturas "depois".**
  - Telas: as mesmas da Task 0, em 360 × 800 (`isMobile`, `hasTouch`).
  - Temas: claro, escuro (contexto com `colorScheme: "dark"`) e alto contraste (`addInitScript` grava `rotasaude:wpda-theme` = `alto-contraste`). Inclua também a página de acessibilidade aberta e o `PhoneOnly` em 1280 × 800 sem toque.
  - Salve em `…/depois/` e monte o `comparacao.html`.
  - Confira: contraste legível nos três temas, o 192 em todas as telas, o topo, a barra compacta e o nível com forma.

- [ ] **Step 2: Abrir o card do `wpdaLook.ts`** no board #2. Peça permissão ao usuário antes, porque é publicação externa.
  - **Repo e título:** `rotasaude/dashboard`, "[CICLO-02] Prévia do protocolo (wpdaLook.ts) com o visual antigo do wpda".
  - **Labels:** `ciclo:02` e `app:dashboard`.
  - **Corpo:** o `wpdaLook.ts` copia à mão os tamanhos e as cores do wpda antigo. Depois do subprojeto 4 a prévia fica diferente do wpda real. A correção é ler `vendor/tokens.css` e usar `data-theme="govbr"` na prévia.
  - **Campos do board:** A triar, Ciclo 2, Implementação, Encerramento do ciclo, Baixa, Ajuste de interface e mod-05.

- [ ] **Step 3: Revisão final.** Faça a revisão do branch inteiro com o revisor mais capaz, corrija os achados Critical e Important com TDD e registre-os no ledger.

- [ ] **Step 4: Aprovação do usuário.**

- [ ] **Step 5: Merge e push do wpda e do docs.**
  1. Faça o merge `--ff-only` e o push.
  2. Remova o worktree e apague o branch.
  3. Confira o `wpda-dev` (5176) no celular emulado e no desktop.
  4. O `@playwright/test` é só dependência de dev e o container não roda e2e, então não precisa reiniciar o container. Se o Vite do container reclamar, reinicie com `docker compose restart wpda`.
  5. Avise a sessão "Próximos módulos".
