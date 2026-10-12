# Ciclo de interface — subprojeto 5: maintenance — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Pôr o maintenance no tema `rota`, com Figtree, topo petróleo com selo "Manutenção", trilho rotulado de 5 telas, faixa de development em roxo e tabelas com rolagem lateral, sem mudar nenhuma funcionalidade.

**Architecture:**
- **Tokens:** copiados com trava, como nos outros apps, e `data-theme="rota"` fixo.
- **`theme/tokens.ts`:** fica reduzido a `Tone` e `toneColor` via `var()`. O arquivo é mantido porque 7 arquivos importam `Tone` dele.
- **Shell:** o `<header>` continua tendo o e-mail e o "sair", porque o e2e exige. Ganha a marca e o selo, e o trilho entra à esquerda do `<main>`.
- **`DataTable` e Auditoria:** a tabela vai para dentro de um `.table-scroll`. A ProtocolsTab (e as demais telas) usam `DataTable`; só `Audit.tsx:185` tem `<table>` à mão.
- **Desvio da spec (§3.1):** a spec move `toneColor` para `tone.ts` e remove `tokens.ts`; o plano mantém o nome `tokens.ts` com o conteúdo novo, para não tocar os 7 imports. O efeito visível é o mesmo.

**Tech Stack:** React 18, Vite 5, Vitest 2, Testing Library e Playwright 1.48 (e2e existente).

**Spec:** `docs/superpowers/specs/2026-10-12-interface-maintenance-design.md`

## Global Constraints

- **Testes:** os 29 arquivos do Vitest passam sem asserção editada, e o e2e `e2e/maintenance.spec.ts` passa inteiro.
- **DOM que não muda:**
  - os botões de navegação: "Cidades", "Medicamentos", "Mantenedores", "Tokens" e "Auditoria", com nome exato e `aria-current="page"`;
  - o botão "sair";
  - o e-mail num único nó de texto, dentro de `<header>`;
  - a `EnvBanner`, com `role="status"`, texto "DEVELOPMENT"/"STAGING" e `data-env`;
  - as tabelas: `<table>/<tr>/<td>`;
  - nenhum `role="status"` novo.
- **Cores:**
  - só por papel do tema;
  - nada de oklch, rgba, hex em string, `"white"` ou `var(--x, reserva)` em `src` (fora `vendor/` e testes);
  - a exceção são as variáveis da faixa no `global.css`.
- **Faixa:** development em `#6b2fa3` com texto `#ffffff`; staging em `#e0952a` com texto `#2b1a00`.
- **Commits e git:** Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Use `/opt/homebrew/bin/git` e caminhos explícitos no `add`.
- **Worktree:** `rota-saude/.worktrees/maintenance-ui`, no branch `ui/maintenance`, com `npm ci`. Nunca dentro de `apps/maintenance`.

## Review Focus

1. **Mantenedor no detalhe da cidade clica em "Cidades" no trilho.**
   - Esperado: volta ao catálogo, como hoje.
   - Garantia: o `goTo` não muda.
   - Onde se prova: a Task 3 e o e2e.
2. **Tabela de Protocolos larga em 1024px com botões de ação.**
   - Esperado: os botões continuam clicáveis, sem a dica de rolagem cobrir nada.
   - Garantia: a dica fica fora da tabela, embaixo, e só aparece se houver transbordo.
   - Onde se prova: as capturas da Task 5.
3. **Botão "sair" ou de ação enquanto ocupado.**
   - Esperado: continua desabilitado com "…", visivelmente inativo, sem depender de opacidade.
   - Onde se prova: a Task 2.
4. **Faixa de staging sobre o topo petróleo.**
   - Esperado: as duas não se confundem.
   - Onde se prova: o teste de cores da Task 1 e a captura de staging da Task 5.
5. **Leitor de tela no trilho.**
   - Esperado: ouvir "Cidades, botão, página atual", sem ler o ícone.
   - Garantia: o SVG é `aria-hidden`.
   - Onde se prova: a Task 3.

---

## File Structure (todos os caminhos são relativos a `apps/maintenance/`)

| Arquivo | Responsabilidade |
|---|---|
| `scripts/tokens-sync.mjs`, `src/theme/vendor/**`, `src/theme/vendor.lock.test.ts` | Cópia travada dos tokens |
| `src/theme/global.css` | Reescrito: tokens, `body`, botões, foco, faixa e `.table-scroll` |
| `src/theme/tokens.ts` | Só `Tone` e `toneColor` via `var()` |
| `src/theme/theme.test.ts` | `data-theme`, tons e cores da faixa |
| `src/theme/guards.test.ts` | Guardas de fonte |
| `index.html`, `src/main.tsx`, `package.json` | `data-theme="rota"`; saem os imports e as dependências da Geist |
| `src/components/Button.tsx` (+ teste) | `--on-accent`; desabilitado tracejado |
| `src/components/TableScroll.tsx` (+ teste) | Contêiner com rolagem e dica |
| `src/components/DataTable.tsx`, `src/screens/Audit.tsx` | Usam `TableScroll` |
| `src/screens/Shell.tsx` (+ `Shell.rail.test.tsx`) | Topo com marca e selo, e trilho |

---

### Task 0: Preparação

- [ ] **Step 1: Worktree e linha de base.**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude && /opt/homebrew/bin/git -C apps/maintenance worktree add -q /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/maintenance-ui -b ui/maintenance origin/main && (cd .worktrees/maintenance-ui && npm ci --silent && npx vitest run 2>&1 | tail -3)
```

  Expected: tudo verde. Anote a contagem.

- [ ] **Step 2: e2e de linha de base** contra o `maintenance-dev` (5177), com `npx playwright test` no worktree. A config usa o stack de dev. Expected: verde. Se falhar antes de qualquer mudança, pare e reporte.

- [ ] **Step 3: Capturas "antes".**
  - Script Playwright no scratchpad, com login `dev@local` / `dev-password` e TOTP do segredo de dev. O segredo está em `DEV_MAINTAINER_OTP_SECRET` ou no `DEFAULT_OTP_SECRET` de `apps/api/lib/dev_maintainer.rb`.
  - Larguras: 768, 1024 e 1280px.
  - Telas:
    - Login;
    - Cidades;
    - detalhe da cidade `curitiba` › Protocolos;
    - Tokens;
    - Auditoria.
  - Salve em `rota-saude/evidencias/2026-10-12-maintenance/antes/`.

---

### Task 1: Tokens, tema `rota`, faixa e CSS global

**Files:** `scripts/tokens-sync.mjs` (cópia byte a byte de `apps/admin/scripts/tokens-sync.mjs`), `src/theme/vendor/**`, `src/theme/vendor.lock.test.ts` (cópia de `apps/admin/src/theme/vendor.lock.test.ts`), `src/theme/global.css`, `src/theme/tokens.ts`, `src/theme/theme.test.ts`, `index.html`, `src/main.tsx`, `package.json`.

- [ ] **Step 1: Escrever os testes, que vão falhar.**

`src/theme/theme.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";
import { toneColor } from "./tokens";

const here = dirname(fileURLToPath(import.meta.url));
const read = (p: string) => readFileSync(join(here, p), "utf8");

function lum(hex: string) {
  const c = [1, 3, 5].map((i) => parseInt(hex.slice(i, i + 2), 16) / 255)
    .map((v) => (v <= 0.03928 ? v / 12.92 : ((v + 0.055) / 1.055) ** 2.4));
  return 0.2126 * c[0] + 0.7152 * c[1] + 0.0722 * c[2];
}
const ratio = (a: string, b: string) => { const [x, y] = [lum(a), lum(b)]; return (Math.max(x, y) + 0.05) / (Math.min(x, y) + 0.05); };

describe("tema do maintenance", () => {
  it("index.html abre no tema rota", () => {
    expect(read("../../index.html")).toMatch(/<html lang="pt-BR" data-theme="rota">/);
  });

  it("toneColor lê papéis do tema", () => {
    expect(toneColor("ok")).toEqual({ fg: "var(--ok)", bg: "var(--ok-bg)" });
    expect(toneColor("down")).toEqual({ fg: "var(--down)", bg: "var(--down-bg)" });
    expect(toneColor(undefined)).toEqual({ fg: "var(--ink2)", bg: "var(--sunken)" });
  });

  it("faixa de development é roxa (7:1) e nenhuma cor da faixa coincide com o tema rota", () => {
    const css = read("global.css");
    const v = (name: string) => new RegExp(`--${name}:\\s*(#[0-9a-f]{6})`, "i").exec(css)?.[1]?.toLowerCase();
    expect(v("env-dev-bg")).toBe("#6b2fa3");
    expect(ratio(v("env-dev-bg")!, v("env-dev-ink")!)).toBeGreaterThanOrEqual(7);
    expect(ratio(v("env-stg-bg")!, v("env-stg-ink")!)).toBeGreaterThanOrEqual(4.5);
    const tokens = read("vendor/tokens.css");
    const rota = /:root\[data-theme="rota"\]\s*\{([^}]*)\}/.exec(tokens)![1].toLowerCase();
    for (const name of [ "env-dev-bg", "env-stg-bg" ]) expect(rota).not.toContain(v(name)!);
  });
});
```

  Copie também o `vendor.lock.test.ts`.

  Run: `npx vitest run src/theme/`
  Expected: FAIL.

- [ ] **Step 2: Sincronizar os tokens.**

```bash
cp /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/admin/scripts/tokens-sync.mjs scripts/tokens-sync.mjs
GIT=/opt/homebrew/bin/git node scripts/tokens-sync.mjs tokens-v1.1.0 --contracts /Users/eduardovrocha/Development/ioit.solutions/rota-saude/contracts
```

  Crie a pasta `scripts/` se ela não existir.

- [ ] **Step 3: Reescrever `src/theme/tokens.ts`.**

```ts
// Tons semânticos do maintenance. Os valores vivem no tema `rota`
// (src/theme/vendor/tokens.css, do contracts/design-tokens); aqui só o nome.
export type Tone = "ok" | "warn" | "down" | "info" | "accent" | "neutral";

export const toneColor = (t: Tone | string | undefined): { fg: string; bg: string } => {
  switch (t) {
    case "ok":     return { fg: "var(--ok)", bg: "var(--ok-bg)" };
    case "warn":   return { fg: "var(--warn)", bg: "var(--warn-bg)" };
    case "down":   return { fg: "var(--down)", bg: "var(--down-bg)" };
    case "info":   return { fg: "var(--info)", bg: "var(--info-bg)" };
    case "accent": return { fg: "var(--accent)", bg: "var(--accent-bg)" };
    default:       return { fg: "var(--ink2)", bg: "var(--sunken)" };
  }
};
```

  Antes de salvar, confira com `grep -rn "tokens\.\(color\|font\|size\|radius\|shadow\|space\|envBanner\)" src` que ninguém usa outro export do arquivo antigo. Se alguém usar, mantenha o export com o valor `var(--…)` equivalente e registre a decisão no ledger. Confira também, com `grep -n "export type Tone" src/theme/tokens.ts` no arquivo antigo, que a união `Tone` antiga é a mesma. Se tiver mais membros, preserve todos.

- [ ] **Step 4: Reescrever `src/theme/global.css`.**

```css
@import "./vendor/tokens.css";

/* Globals do maintenance. Papéis e tema (`rota`, fixo no <html>) vêm de
   vendor/tokens.css (contracts/design-tokens). */

/* Faixa de ambiente (spec F6): cores próprias, fora da paleta do tema, para
   development/staging nunca se confundirem com a interface. */
:root {
  --env-dev-bg:  #6b2fa3;
  --env-dev-ink: #ffffff;
  --env-stg-bg:  #e0952a;
  --env-stg-ink: #2b1a00;
}

* { box-sizing: border-box; }

html, body, #root { margin: 0; padding: 0; height: 100%; }

body {
  background: var(--bg);
  color: var(--ink);
  font-family: var(--font-sans);
  font-size: 13px;
  line-height: 1.4;
  -webkit-font-smoothing: antialiased;
  text-rendering: optimizeLegibility;
}

button { font-family: inherit; color: inherit; background: transparent; border: 0; cursor: pointer; padding: 0; }

/* Foco visível em todo elemento focável (WCAG 2.4.7 / 2.4.11). */
:focus-visible { outline: 3px solid var(--focus); outline-offset: 2px; }

a { color: var(--accent); text-decoration: none; }

.mono { font-family: var(--font-mono); }

/* Faixa de ambiente: fixa no topo, fora de qualquer rolagem (spec F6). */
.env-banner {
  position: sticky; top: 0; z-index: 10;
  display: flex; align-items: center; justify-content: center;
  padding: 6px 12px; font-size: 11px; font-weight: 700; letter-spacing: 0.08em; text-align: center;
}
.env-banner[data-env="development"] { background: var(--env-dev-bg); color: var(--env-dev-ink); }
.env-banner[data-env="staging"] { background: var(--env-stg-bg); color: var(--env-stg-ink); }

/* Tabelas largas: rolam dentro do contêiner; a dica só com transbordo. */
.table-scroll { overflow-x: auto; }
.table-scroll.is-overflowing::after {
  content: "⇆ role para o lado";
  display: block; padding: 4px 10px; font-size: 11px; color: var(--ink2);
}
```

  Os keyframes `pulse-dot` e `skeleton-shimmer` saem: nenhum arquivo de `src` os usa (conferido com grep na escrita do plano).

- [ ] **Step 5: `index.html`, `main.tsx` e dependências.**
  - No `index.html`, `<html lang="pt-BR">` vira `<html lang="pt-BR" data-theme="rota">`.
  - No `src/main.tsx`, apague os imports `@fontsource-variable/geist` e `geist-mono`.
  - Rode `npm uninstall @fontsource-variable/geist @fontsource-variable/geist-mono`.

- [ ] **Step 6: Rodar.**

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit -p . && npm run build && npm run codegen:check`
  Expected: verde e limpo.

- [ ] **Step 7: Commit.**

```bash
/opt/homebrew/bin/git add scripts/tokens-sync.mjs src/theme index.html src/main.tsx package.json package-lock.json
/opt/homebrew/bin/git commit -m "feat: vendor design tokens and switch the maintenance console to the rota theme

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: `Button` e guardas de cor

**Files:** `src/components/Button.tsx`, `src/components/Button.theme.test.tsx`, `src/theme/guards.test.ts`.

- [ ] **Step 1: Escrever os testes, que vão falhar.**

`src/components/Button.theme.test.tsx`:
```tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { Button } from "./Button";

describe("Button no tema rota", () => {
  it("texto sobre o accent usa --on-accent", () => {
    render(<Button>Salvar</Button>);
    expect(screen.getByRole("button", { name: "Salvar" }).style.color).toBe("var(--on-accent)");
  });

  it("desabilitado e ocupado: sem opacidade, borda tracejada, segue disabled", () => {
    const { rerender } = render(<Button disabled>Salvar</Button>);
    let b = screen.getByRole("button", { name: "Salvar" });
    expect(b).toHaveProperty("disabled", true);
    expect(b.style.opacity).toBe("");
    expect(b.style.border).toContain("dashed");
    rerender(<Button busy>Salvar</Button>);
    b = screen.getByRole("button", { name: "…" });
    expect(b).toHaveProperty("disabled", true);
    expect(b.style.border).toContain("dashed");
  });
});
```

`src/theme/guards.test.ts`:
```ts
// Guardas do maintenance: cor só por papel do tema (exceto a faixa no
// global.css), foco nunca zerado, desabilitado sem opacidade.
import { describe, expect, it } from "vitest";
import { readFileSync, readdirSync, statSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join, relative } from "node:path";

const src = join(dirname(fileURLToPath(import.meta.url)), "..");
function files(dir: string): string[] {
  return readdirSync(dir).flatMap((n) => {
    const f = join(dir, n);
    if (statSync(f).isDirectory()) return [ "vendor", "gql", "test" ].includes(n) ? [] : files(f);
    return /\.tsx?$/.test(n) && !/\.test\./.test(n) ? [f] : [];
  });
}
const all = files(src).map((f) => ({ f: relative(src, f), code: readFileSync(f, "utf8") }));
const offenders = (re: RegExp) => all.filter(({ code }) => re.test(code)).map(({ f }) => f);

describe("guardas do maintenance", () => {
  it("nenhuma cor fixa (oklch, rgba, hex em string, white)", () =>
    expect(offenders(/oklch\(|rgba?\(|"#[0-9a-fA-F]{3,6}"|"white"/)).toEqual([]));
  it("nenhum var(--x, reserva)", () => expect(offenders(/var\(--[\w-]+\s*,/)).toEqual([]));
  it("nenhum outline zerado", () => expect(offenders(/outline:\s*"none"/)).toEqual([]));
  it("desabilitado não depende de opacidade", () => expect(offenders(/opacity:\s*[^,}]*(disabled|busy)/)).toEqual([]));
});
```

  Run: `npx vitest run src/components/Button.theme.test.tsx src/theme/guards.test.ts`
  Expected: FAIL no `Button` e nas guardas de branco e opacidade. Se a guarda de cor fixa acusar outros arquivos além do `Button`, troque o literal pelo papel equivalente e registre a decisão no ledger. Exemplos: um `rgba` de sombra vira `var(--shadow-…)`, que já existe nos tokens; um `#fff` vira `var(--panel)` ou `var(--on-accent)`.

- [ ] **Step 2: Mudar o `Button`.** O `style` passa a ser:

```tsx
      style={{
        padding: "9px 16px",
        borderRadius: 6,
        fontSize: 13,
        fontWeight: 600,
        ...(disabled || busy
          ? { border: "2px dashed var(--rule2)", background: "var(--sunken)", color: "var(--ink2)", cursor: "default" }
          : { border: "1px solid var(--accent)", background: "var(--accent)", color: "var(--on-accent)" })
      }}
```

  Run: `npx vitest run src/components/Button.theme.test.tsx src/theme/guards.test.ts && npx vitest run 2>&1 | tail -3`
  Expected: PASS e a suíte verde.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/components/Button.tsx src/components/Button.theme.test.tsx src/theme/guards.test.ts
/opt/homebrew/bin/git commit -m "style: theme-aware maintenance button and colour guards

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

  Se a guarda de cor fixa tiver exigido trocar literais em outros arquivos, acrescente esses arquivos ao `git add`.

---

### Task 3: Shell com marca, selo e trilho

**Files:** `src/screens/Shell.tsx`, `src/screens/Shell.rail.test.tsx`.

- [ ] **Step 1: Escrever o teste, que vai falhar.** Copie do `Shell.test.tsx` existente o mock de sessão e o `render`; leia o arquivo antes de escrever.

```tsx
it("trilho 'Telas' com os 5 botões de nome exato, ícone escondido, Cidades ativo", async () => {
  // render do Shell logado, como no Shell.test.tsx
  const rail = await screen.findByRole("navigation", { name: "Telas" });
  for (const label of [ "Cidades", "Medicamentos", "Mantenedores", "Tokens", "Auditoria" ]) {
    const b = within(rail).getByRole("button", { name: label });
    expect(b.querySelector("svg")?.getAttribute("aria-hidden")).toBe("true");
  }
  expect(within(rail).getByRole("button", { name: "Cidades" }).getAttribute("aria-current")).toBe("page");
});

it("topo: marca, selo Manutenção (sem role status), e-mail e sair dentro do header", async () => {
  // render do Shell logado
  const header = (await screen.findByText("mantenedor@rotasaude.app")).closest("header")!;
  expect(header).not.toBeNull();
  expect(within(header).getByText("Manutenção").closest('[role="status"]')).toBeNull();
  expect(within(header).getByText("Rota Saúde")).toBeTruthy();
  expect(within(header).getByRole("button", { name: "sair" })).toBeTruthy();
});
```

  Use o e-mail do mock existente, que no `Shell.test.tsx` é `mantenedor@rotasaude.app`.

  Run: `npx vitest run src/screens/Shell.rail.test.tsx`
  Expected: FAIL, porque não há `navigation` "Telas".

- [ ] **Step 2: Reescrever o retorno do `Shell`.** Mantenha imports, estado, `NAV` e `goTo`. Acrescente ao `NAV` um campo `icon: ReactNode` com um SVG por item (24×24, `stroke="currentColor"`, `fill="none"`, `strokeWidth="1.8"`, `aria-hidden="true"`):

```tsx
const svg = (d: string) => (
  <svg aria-hidden="true" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.8" strokeLinecap="round" strokeLinejoin="round"><path d={d} /></svg>
);
const ICON: Record<Screen, ReactNode> = {
  cities: svg("M3 21h18M5 21V9l7-5 7 5v12M9 21v-6h6v6"),
  medications: svg("M10.5 20.5l10-10a4.95 4.95 0 1 0-7-7l-10 10a4.95 4.95 0 1 0 7 7zM8.5 8.5l7 7"),
  maintainers: svg("M9 7a4 4 0 1 0 0 .01M3 21v-2a4 4 0 0 1 4-4h4a4 4 0 0 1 4 4v2M16 3.13a4 4 0 0 1 0 7.75M21 21v-2a4 4 0 0 0-3-3.85"),
  tokens: svg("M15 7a4 4 0 1 1-3.87 5H8v3H5v-3H3v-3h8.13A4 4 0 0 1 15 7z"),
  audit: svg("M12 8v4l3 3M3.05 11a9 9 0 1 1 .5 4M3 4v5h5")
};
```

  O retorno do `Shell` fica assim:

```tsx
  return (
    <div style={{ minHeight: "100%", display: "flex", flexDirection: "column" }}>
      <header style={{ display: "flex", alignItems: "center", gap: 12, padding: "10px 16px",
        background: "var(--ink)", color: "var(--panel)" }}>
        <span aria-hidden="true" style={{ width: 28, height: 28, borderRadius: 6, display: "inline-flex", alignItems: "center",
          justifyContent: "center", background: "var(--accent)", color: "var(--on-accent)" }}>
          {svg("M4 12h4l3-6 4 12 3-6h2")}
        </span>
        <strong style={{ fontSize: 15 }}>Rota Saúde</strong>
        <span style={{ padding: "2px 8px", borderRadius: 6, fontSize: 12, fontWeight: 700,
          background: "var(--accent-bg)", color: "var(--accent)" }}>Manutenção</span>
        <div style={{ marginLeft: "auto", display: "flex", alignItems: "center", gap: 12 }}>
          <span style={{ fontSize: 12.5 }}>{me?.emailAddress}</span>
          <Button onClick={() => void signOut()}>sair</Button>
        </div>
      </header>

      <div style={{ display: "flex", flex: 1, minHeight: 0 }}>
        <nav aria-label="Telas" style={{ width: 92, flexShrink: 0, display: "flex", flexDirection: "column", gap: 4,
          padding: "10px 6px", background: "var(--panel)", borderRight: "1px solid var(--rule)" }}>
          {NAV.map((item) => {
            const active = screen === item.key;
            return (
              <button
                key={item.key}
                onClick={() => goTo(item.key)}
                aria-current={active ? "page" : undefined}
                style={{ display: "flex", flexDirection: "column", alignItems: "center", gap: 3, minHeight: 56,
                  padding: "8px 2px", borderRadius: 8, fontSize: 12, fontWeight: active ? 700 : 500,
                  color: active ? "var(--accent)" : "var(--ink2)", background: active ? "var(--accent-bg)" : "transparent" }}
              >
                {ICON[item.key]}
                {item.label}
              </button>
            );
          })}
        </nav>

        <main style={{ padding: 24, flex: 1, minWidth: 0 }}>
          {/* as mesmas renderizações condicionais de hoje, sem mudança */}
        </main>
      </div>
    </div>
  );
```

  Pontos de atenção:
  - Copie para dentro do `<main>` o bloco atual de telas, sem mudar nada.
  - O nome acessível do botão é só o rótulo, porque o SVG é `aria-hidden`. O teste `Shell.test.tsx:49-52` confere isso.
  - Importe `type ReactNode` de `react`.
  - O `Button` "sair" fica sobre fundo petróleo. Isso é aceitável, porque o `accent` (petróleo 70) tem borda própria. Se o contraste da borda contra o `--ink` ficar ruim na captura, registre a decisão no ledger e use `--accent-bg`.

- [ ] **Step 3: Rodar.**

  Run: `npx vitest run src/screens/ && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit -p .`
  Expected: PASS. O `Shell.test` original continua verde.

- [ ] **Step 4: Commit.**

```bash
/opt/homebrew/bin/git add src/screens/Shell.tsx src/screens/Shell.rail.test.tsx
/opt/homebrew/bin/git commit -m "feat: add brand, maintenance seal and labelled rail to the maintenance shell

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Tabelas com rolagem lateral

**Files:** `src/components/TableScroll.tsx`, `src/components/TableScroll.test.tsx`, `src/components/DataTable.tsx`, `src/screens/Audit.tsx`.

- [ ] **Step 1: Escrever o teste, que vai falhar.**

```tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { TableScroll } from "./TableScroll";
import { DataTable } from "./DataTable";

describe("TableScroll", () => {
  it("envolve a tabela num contêiner com rolagem, sem texto extra no DOM", () => {
    const { container } = render(<TableScroll><table><tbody><tr><td>a</td></tr></tbody></table></TableScroll>);
    const wrap = container.firstElementChild as HTMLElement;
    expect(wrap.className).toContain("table-scroll");
    expect(wrap.textContent).toBe("a");
  });

  it("DataTable fica dentro do contêiner e continua sendo uma única table", () => {
    render(<DataTable columns={[ { key: "n", label: "Nome" } ]} rows={[ { n: "x" } ]} />);
    const table = screen.getByRole("table");
    expect(table.parentElement?.className).toContain("table-scroll");
  });
});
```

  Ruling: o teste da Auditoria (`Audit.test.tsx`) é existente e não é editado. A cobertura do contêiner da Auditoria vem da guarda de fonte do Step 3.

  Run: `npx vitest run src/components/TableScroll.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: Implementar.**

`src/components/TableScroll.tsx`:
```tsx
// Tabelas largas rolam para o lado dentro deste contêiner (spec §4.1). A dica
// "⇆ role para o lado" é CSS (::after) e só aparece com transbordo real.
import { useEffect, useRef, type ReactNode } from "react";

export function TableScroll({ children }: { children: ReactNode }) {
  const ref = useRef<HTMLDivElement>(null);
  useEffect(() => {
    const el = ref.current;
    if (!el || typeof ResizeObserver === "undefined") return;
    const check = () => el.classList.toggle("is-overflowing", el.scrollWidth > el.clientWidth + 1);
    const ro = new ResizeObserver(check);
    ro.observe(el);
    check();
    return () => ro.disconnect();
  }, []);
  return <div ref={ref} className="table-scroll">{children}</div>;
}
```

  No `DataTable`, envolva o `<table>` com `<TableScroll>…</TableScroll>`. No `Audit.tsx` (linha 185), faça o mesmo com a tabela.

- [ ] **Step 3: Guarda da Auditoria.** Acrescente ao `src/theme/guards.test.ts`:

```ts
it("a tabela da Auditoria fica dentro de TableScroll", () => {
  const code = readFileSync(join(src, "screens/Audit.tsx"), "utf8");
  expect(code).toMatch(/<TableScroll>\s*<table/);
});
```

  Run: `npx vitest run src/components/ src/theme/ && npx vitest run 2>&1 | tail -3`
  Expected: PASS. Os testes de tabela existentes (`getByRole("table")` único, `.closest("tr")`, linhas) continuam verdes.

- [ ] **Step 4: Commit.**

```bash
/opt/homebrew/bin/git add src/components/TableScroll.tsx src/components/TableScroll.test.tsx src/components/DataTable.tsx src/screens/Audit.tsx src/theme/guards.test.ts
/opt/homebrew/bin/git commit -m "style: scroll wide maintenance tables horizontally with an overflow hint

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: e2e, capturas, revisão e publicação

- [ ] **Step 1: Rodar o e2e contra o worktree.**
  - Suba o maintenance do worktree com `VITE_MAINTENANCE_ENV=development VITE_API_PROXY_TARGET=http://localhost:3030 npx vite --port 5188 --strictPort --host 127.0.0.1`.
  - O proxy injeta `Origin http://maintenance.localhost:5177`, e a porta muda. Confira em `vite.config.ts` se o Origin injetado é fixo. Se a API recusar o Origin de 5188, rode o e2e contra o `maintenance-dev` (5177) depois do merge, como prova pós-merge, e registre a decisão no ledger.
  - Rode `npx playwright test`, mudando a URL base pela variável que o `e2e/support.ts` usa. Leia o arquivo antes.
  - Expected: verde.

- [ ] **Step 2: Capturas "depois".**
  - Mesmas telas e larguras da Task 0.
  - Mais uma captura em 1280px com um segundo Vite do worktree com `VITE_MAINTENANCE_ENV=staging` (porta 5189).
  - Confira:
    - nenhum texto cortado com a Figtree;
    - a dica de rolagem nas tabelas largas em 768px;
    - o trilho e o topo legíveis;
    - a faixa roxa (development) e a âmbar (staging) distintas do topo petróleo.

- [ ] **Step 3: Revisão final.** Faça a revisão do branch inteiro com o revisor mais capaz. Corrija os achados Critical e Important com TDD e registre-os no ledger.

- [ ] **Step 4: Aprovação do usuário** (`comparacao.html`).

- [ ] **Step 5: Merge e push do maintenance e do docs.**
  1. Remova o worktree e apague o branch.
  2. Reinicie o `maintenance-dev` (`docker compose restart maintenance`), porque a Geist saiu das dependências.
  3. Confira o 5177.
  4. Rode o e2e contra o 5177.
  5. Avise a sessão "Próximos módulos".
