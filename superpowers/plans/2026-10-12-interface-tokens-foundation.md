# Ciclo de interface — subprojeto 1: fundação de tokens — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fazer o `contracts/design-tokens` virar a fonte única dos tokens, com quatro temas validados por contraste, e passar o dashboard e o admin para o tema `govbr`, com Rawline, AA e risco indicado por forma, sem mudar nenhuma funcionalidade.

**Architecture:**
- **Contracts.** Três camadas JSON (`base`, `roles`, `themes`) alimentam um gerador Node sem dependências. Ele emite `dist/tokens.css` (um bloco `:root[data-theme]` por tema) e `dist/tokens.ts`. Um teste de contraste trava os mínimos.
- **Apps.** Cada app copia o `dist/` de uma tag do contracts para `src/theme/vendor/` e grava um lock com os hashes. O `global.css` importa a cópia. O `tokens.ts` do app passa a apontar para as variáveis CSS (`var(--…)`), mantendo os mesmos nomes exportados.

**Tech Stack:**
- Contracts: Node 22 (`node --test`, sem dependências) e JSON.
- Apps: React 18, Vite 5, Vitest 2, jsdom e Testing Library.

**Spec:** `docs/superpowers/specs/2026-10-12-interface-cycle-foundation-design.md`

## Global Constraints

- **Regra do ciclo:** só interface. Nenhuma funcionalidade muda.
- **Testes existentes:** os do dashboard e do admin precisam passar **sem nenhuma asserção editada**. Se um teste quebrar, a mudança de interface está errada, não o teste.
- **Nomes de variável:** os de hoje são mantidos (`--bg`, `--panel`, `--sunken`, `--sunken2`, `--rule`, `--rule2`, `--ink`…`--ink4`, `--accent`, `--accent-bg`, `--ok`/`--warn`/`--down`/`--info` e os seus `-bg`). Os nomes novos são só `--focus`, `--on-accent`, `--font-sans`, `--font-mono`, `--space-1`…`--space-8` e os raios. Também entram os apelidos `--line`, `--danger` e `--surface`, que o código já usa sem que estejam definidos.
- **Contraste:**

  | Par | Mínimo |
  |---|---|
  | Texto (`--ink`, `--ink2`, `--ink3`; cor semântica sobre o seu `-bg`; `--on-accent` sobre `--accent`) | 4,5:1 |
  | `--rule2` e `--focus` contra o fundo | 3:1 |
  | Qualquer par no tema `alto-contraste` | 7:1 |

  `--ink4` fica fora, porque só é usado em texto desabilitado ou decorativo.
- **Temas:** `govbr` (padrão), `rota`, `alto-contraste` e `govbr-escuro`. Neste subprojeto, só o `govbr` é consumido pelos apps.
- **Fonte:** Rawline em arquivo local, sem CDN, com reserva Verdana. Corpo de 14px (antes era 13px).
- **Componentes divergentes:** cada app mantém a sua cópia. A lógica não é tocada (ADR 0002).
- **Fora do escopo:** wpda, maintenance, api, seletor de tema, cards, trilho, layout de 768px e barra eMAG.
- **Commits:** Conventional Commits em inglês, tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Git:** use `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta) e **nunca** `git add -A`; adicione os caminhos explícitos.
- **Isolamento:** trabalhe em worktrees, no branch `ui/tokens-foundation` de cada repo, e rode `npm ci` no worktree. Nunca crie symlink de `node_modules`.

## Review Focus

1. **Cor dentro de SVG.** `toneColor()` passa a devolver `var(--ok)` e afins, e o recharts põe esse valor em atributos SVG (`stroke` e `fill` de Sparkline, StackedBar e Funnel). A cor precisa aparecer no navegador, e não um traço preto ou sumido. A Task 5 tem uma conferência visual dedicada.
2. **Glifo de forma no `textContent`.** Muitos testes conferem o texto de linhas e células. A forma (■ ▲ ●) entra só por CSS `::before` com texto alternativo vazio, e nunca como nó de texto. A Task 6 testa que o `textContent` do `Tag` é exatamente o `children`.
3. **Cópia editada à mão.** Qualquer arquivo em `src/theme/vendor/` diferente do lock, ou um arquivo a mais que o lock não conhece, precisa reprovar a suíte. A Task 4 testa os dois casos.
4. **Botão com texto branco fixo sobre `--accent`.** Login, ResetPassword, AcceptInvitation e formStyles usam `color: "#fff"`, o que quebra no alto contraste (fundo amarelo). Essas cores passam para `var(--on-accent)` na Task 7.
5. **Corpo de 13px para 14px em tela densa.** Tabelas e cabeçalhos podem quebrar linha ou cortar. Isso é conferido nas capturas da Task 8. Ajustar espaçamento é permitido; mudar estrutura, não.

---

## File Structure

**contracts** (`rotasaude/contracts`, worktree `contracts/.claude/worktrees/ui-tokens`):

| Arquivo | Responsabilidade |
|---|---|
| `design-tokens/src/base.json` | Paleta crua (nome → hex) e pilhas de fonte |
| `design-tokens/src/roles.json` | Papéis de cor, apelidos, constantes, faces de fonte e pares de contraste |
| `design-tokens/src/themes/{govbr,rota,alto-contraste,govbr-escuro}.json` | Papel → nome da paleta, por tema |
| `design-tokens/build.mjs` | `build()` (puro) e `resolveThemes()`. Quando executado como CLI, grava o `dist/` |
| `design-tokens/dist/tokens.css`, `dist/tokens.ts`, `dist/fonts/*` | Arquivos gerados e fontes, versionados no git |
| `design-tokens/test/build.test.mjs` | Validação do gerador e determinismo do `dist/` |
| `design-tokens/test/contrast.test.mjs` | Mínimos de contraste por tema |
| `design-tokens/README.md`, `design-tokens/CHANGELOG.md` | Documentação e histórico da tag |

**dashboard e admin** (cada um no seu repo e worktree):

| Arquivo | Responsabilidade |
|---|---|
| `scripts/tokens-sync.mjs` | Copia o `design-tokens/dist` de uma ref do contracts para `src/theme/vendor/` e grava o lock |
| `src/theme/vendor/**` | Cópia gerada e `tokens.lock` |
| `src/theme/vendor.lock.test.ts` | Confere a cópia contra o lock |
| `src/theme/global.css` | Importa a cópia. Contém reset, foco e glifos de forma |
| `src/theme/tokens.ts` | Mesmos nomes exportados, com valores `var(--…)` e `toneShape()` |
| `src/theme/theme.test.ts` | Testa `data-theme` no `index.html`, `toneColor` e `toneShape` |
| `src/components/Tag.tsx`, `StatusDot.tsx` e os seus testes | Forma além da cor |
| `index.html` | `data-theme="govbr"` |
| `src/main.tsx` | Remove os imports da Geist |
| `package.json` | Remove `@fontsource-variable/geist*`. No admin, acrescenta `@types/node` |

No dashboard, a Task 7 também mexe em `src/modules/Login.tsx`, `ResetPassword.tsx`, `AcceptInvitation.tsx` e `src/components/formStyles.ts`.

**Componentes que divergiram:** DataTable, KpiGrid, Sparkline, StatTile, PageHeader, BarMini, Panel e AppHeader. Na exploração deste plano, todas as diferenças entre as cópias do dashboard e do admin se mostraram de **comportamento**. O estilo já é igual, e as duas cópias já leem as mesmas variáveis. Exemplos:

- o `aria-label` do Panel;
- `null` como lacuna no BarMini e no Sparkline;
- o `right` do PageHeader;
- a supressão de contagens pequenas e o `hint` do StatTile;
- o `asOf` e o `columns` do KpiGrid;
- o fuso no AppHeader.

Por isso, **nenhuma dessas cópias é editada**. A unificação visual vem de graça com a troca de tokens. Isso fica registrado no README do contracts (Task 3).

---

### Task 0: Preparação (avisos, worktrees, linha de base e capturas "antes")

**Files:** nenhum arquivo de produto.

- [ ] **Step 1: Avisar as sessões donas.** Liste as sessões com `ListAgents`. Mande à sessão do dashboard e à do admin (por `SendMessage`) este texto:

  > Ciclo de interface, subprojeto 1: vou trabalhar em worktree `ui/tokens-foundation` no seu app, mexendo só em `src/theme/`, `Tag`, `StatusDot`, `index.html`, `main.tsx` (imports de fonte), `package.json` (fontes) e, no dashboard, nas cores fixas de `Login`, `ResetPassword`, `AcceptInvitation` e `formStyles`. Nenhuma funcionalidade muda. Spec: `docs/superpowers/specs/2026-10-12-interface-cycle-foundation-design.md`. Me avise se tiver algo em andamento nesses arquivos.

  Espere a resposta ou um "ok" do usuário antes da Task 4.

- [ ] **Step 2: Criar os três worktrees.**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/contracts && /opt/homebrew/bin/git worktree add .claude/worktrees/ui-tokens -b ui/tokens-foundation origin/main
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard && /opt/homebrew/bin/git worktree add .claude/worktrees/ui-tokens -b ui/tokens-foundation origin/main && cd .claude/worktrees/ui-tokens && npm ci
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/admin && /opt/homebrew/bin/git worktree add .claude/worktrees/ui-tokens -b ui/tokens-foundation origin/main && cd .claude/worktrees/ui-tokens && npm ci
```

  Antes, confira `origin/main..main` em cada repo. Se o main local estiver à frente do origin, crie o worktree a partir de `main` e anote isso no relatório.

- [ ] **Step 3: Medir a linha de base das suítes.** Em cada worktree de app:

```bash
npx vitest run 2>&1 | tail -5
```

  Anote as contagens de arquivos e testes passando. Todos têm que estar verdes. Se algum já estiver vermelho na linha de base, pare e reporte ao usuário: não é culpa deste plano.

- [ ] **Step 4: Capturas "antes".** Com a API de dev rodando em `:3030`:
  1. Suba o dashboard do worktree com `npx vite --port 5185 --strictPort`.
  2. No navegador embutido, abra `http://curitiba.localhost:5185/`, com a janela em 1280×800 (`resize_window`).
  3. Entre com a conta de dev da semente (`admin@curitiba.demo`, senha e TOTP da semente de dev).
  4. Capture estas telas: Visão geral, Triagens, Atendimento, Protocolos e Equipe.
  5. Salve as capturas em `docs/evidencias/2026-10-12-tokens-foundation/antes/<tela>.png`.
  6. Repita no admin (`npx vite --port 5186 --strictPort`, login de operador de dev) para Visão geral e Cidades.

  Não faça commit ainda.

---

### Task 1: Fonte dos tokens e gerador (contracts)

**Files:**
- Create: `design-tokens/src/base.json`, `design-tokens/src/roles.json`, `design-tokens/src/themes/govbr.json`, `rota.json`, `alto-contraste.json`, `govbr-escuro.json`
- Create: `design-tokens/build.mjs`
- Create: `design-tokens/test/build.test.mjs`
- Create (gerado): `design-tokens/dist/tokens.css`, `design-tokens/dist/tokens.ts`

Todos os caminhos são relativos ao worktree do contracts.

**Interfaces:**
- Produces:
  - `build(): { css: string, ts: string }`, puro: lê `src/` e não grava nada.
  - `resolveThemes(): Record<ThemeName, Record<ColorRole, string /* #rrggbb */>>`.
  - `loadRoles(): { colors: string[], aliases: Record<string,string>, constants: Record<string,string>, fontFaces: {family,weight,file}[], pairs: { text: [string,string][], nonText: [string,string][] } }`.
  - `loadThemes(): Array<{ name, minContrast?, colors: Record<role, baseKey>, fonts: Record<'font-sans'|'font-mono', baseFontKey> }>`.
  - CLI: `node design-tokens/build.mjs` grava `dist/tokens.css` e `dist/tokens.ts`.
  - CSS: o bloco `:root:not([data-theme]), :root[data-theme="govbr"]` vem primeiro, seguido de `:root[data-theme="<tema>"]` em ordem alfabética. As constantes e os apelidos ficam num `:root {}` à parte.

- [ ] **Step 1: Escrever `design-tokens/src/base.json`.**

```json
{
  "colors": {
    "white": "#ffffff", "black": "#000000",
    "gray-2": "#f8f8f8", "gray-5": "#f0f0f0", "gray-10": "#e6e6e6", "gray-15": "#dcdcdc",
    "gray-50": "#888888", "gray-60": "#555555", "gray-80": "#333333", "gray-90": "#1b1b1b",
    "gov-blue-70": "#1351b4", "gov-blue-10": "#dbe8fb",
    "gov-green-70": "#0f6b1d", "gov-green-5": "#e3f5e1",
    "gov-yellow-80": "#7a4b00", "gov-yellow-5": "#fff5c2",
    "gov-red-70": "#b31800", "gov-red-5": "#fde0db",
    "gov-info-80": "#0c4ca3", "gov-info-10": "#d4e5ff",
    "focus-orange": "#b35c00",
    "sand-2": "#f6f4ef", "sand-5": "#eeeae1", "sand-10": "#e4ded2", "sand-15": "#ddd7cc", "sand-50": "#857b6c",
    "petrol-ink-90": "#1d2a2a", "petrol-ink-80": "#2f3d3d", "petrol-ink-60": "#4c5a59",
    "petrol-70": "#0d6363", "petrol-10": "#d7ecea",
    "rota-green-70": "#1f6b35", "rota-green-5": "#e1f2e5",
    "rota-yellow-5": "#fbefd2",
    "rota-red-70": "#a8321c", "rota-red-5": "#fbe1d8",
    "rota-blue-70": "#1d5486", "rota-blue-10": "#dce9f5",
    "hc-gray-95": "#111111", "hc-gray-20": "#cccccc",
    "hc-yellow": "#ffff00", "hc-green": "#66ff99", "hc-amber": "#ffd400", "hc-red": "#ff8080", "hc-blue": "#80bfff",
    "dk-0": "#121212", "dk-1": "#1b1b1b", "dk-2": "#242424", "dk-3": "#2e2e2e", "dk-4": "#3a3a3a",
    "dk-50": "#8a8a8a", "dk-70": "#b8b8b8", "dk-85": "#d6d6d6", "dk-95": "#f2f2f2",
    "dk-blue": "#8ab4ff", "dk-blue-bg": "#0c326f",
    "dk-green": "#6fdc8c", "dk-green-bg": "#0f2e18",
    "dk-yellow": "#ffcd07", "dk-yellow-bg": "#3a2d00",
    "dk-red": "#ff8a75", "dk-red-bg": "#3d0d06",
    "dk-info-bg": "#0c2650", "dk-orange": "#ffb54d"
  },
  "fonts": {
    "rawline": "\"Rawline\", Verdana, system-ui, sans-serif",
    "figtree": "\"Figtree\", system-ui, sans-serif",
    "mono": "ui-monospace, \"SF Mono\", Menlo, Consolas, monospace"
  }
}
```

- [ ] **Step 2: Escrever `design-tokens/src/roles.json`.**

```json
{
  "colors": [
    "bg", "panel", "sunken", "sunken2", "rule", "rule2",
    "ink", "ink2", "ink3", "ink4",
    "accent", "accent-bg", "on-accent",
    "ok", "ok-bg", "warn", "warn-bg", "down", "down-bg", "info", "info-bg",
    "focus"
  ],
  "aliases": { "line": "rule2", "danger": "down", "surface": "sunken" },
  "constants": {
    "radius-chip": "6px", "radius-panel": "10px", "radius-menu": "12px",
    "space-1": "4px", "space-2": "8px", "space-3": "12px", "space-4": "16px",
    "space-5": "20px", "space-6": "24px", "space-7": "32px", "space-8": "48px",
    "shadow-dropdown": "0 8px 28px -8px rgba(20,20,40,.22), 0 2px 6px rgba(20,20,40,.06)",
    "shadow-notif": "0 14px 40px -10px rgba(20,20,40,.26), 0 3px 8px rgba(20,20,40,.07)"
  },
  "fontFaces": [],
  "pairs": {
    "text": [
      ["ink", "bg"], ["ink", "panel"], ["ink", "sunken"],
      ["ink2", "bg"], ["ink2", "panel"], ["ink2", "sunken"],
      ["ink3", "bg"], ["ink3", "panel"], ["ink3", "sunken"],
      ["ok", "ok-bg"], ["warn", "warn-bg"], ["down", "down-bg"], ["info", "info-bg"],
      ["accent", "accent-bg"], ["on-accent", "accent"]
    ],
    "nonText": [
      ["rule2", "bg"], ["rule2", "panel"], ["rule2", "sunken"],
      ["focus", "bg"], ["focus", "panel"], ["focus", "sunken"]
    ]
  }
}
```

  `fontFaces` começa vazio e é preenchido na Task 3, quando os arquivos da fonte existirem.

- [ ] **Step 3: Escrever os quatro temas.**

`design-tokens/src/themes/govbr.json`:
```json
{
  "name": "govbr",
  "colors": {
    "bg": "gray-2", "panel": "white", "sunken": "gray-5", "sunken2": "gray-10", "rule": "gray-15", "rule2": "gray-50",
    "ink": "gray-90", "ink2": "gray-80", "ink3": "gray-60", "ink4": "gray-50",
    "accent": "gov-blue-70", "accent-bg": "gov-blue-10", "on-accent": "white",
    "ok": "gov-green-70", "ok-bg": "gov-green-5", "warn": "gov-yellow-80", "warn-bg": "gov-yellow-5",
    "down": "gov-red-70", "down-bg": "gov-red-5", "info": "gov-info-80", "info-bg": "gov-info-10",
    "focus": "focus-orange"
  },
  "fonts": { "font-sans": "rawline", "font-mono": "mono" }
}
```

`design-tokens/src/themes/rota.json`:
```json
{
  "name": "rota",
  "colors": {
    "bg": "sand-2", "panel": "white", "sunken": "sand-5", "sunken2": "sand-10", "rule": "sand-15", "rule2": "sand-50",
    "ink": "petrol-ink-90", "ink2": "petrol-ink-80", "ink3": "petrol-ink-60", "ink4": "sand-50",
    "accent": "petrol-70", "accent-bg": "petrol-10", "on-accent": "white",
    "ok": "rota-green-70", "ok-bg": "rota-green-5", "warn": "gov-yellow-80", "warn-bg": "rota-yellow-5",
    "down": "rota-red-70", "down-bg": "rota-red-5", "info": "rota-blue-70", "info-bg": "rota-blue-10",
    "focus": "focus-orange"
  },
  "fonts": { "font-sans": "figtree", "font-mono": "mono" }
}
```

`design-tokens/src/themes/alto-contraste.json`:
```json
{
  "name": "alto-contraste",
  "minContrast": 7,
  "colors": {
    "bg": "black", "panel": "black", "sunken": "black", "sunken2": "hc-gray-95", "rule": "white", "rule2": "white",
    "ink": "white", "ink2": "white", "ink3": "white", "ink4": "hc-gray-20",
    "accent": "hc-yellow", "accent-bg": "black", "on-accent": "black",
    "ok": "hc-green", "ok-bg": "black", "warn": "hc-amber", "warn-bg": "black",
    "down": "hc-red", "down-bg": "black", "info": "hc-blue", "info-bg": "black",
    "focus": "hc-yellow"
  },
  "fonts": { "font-sans": "rawline", "font-mono": "mono" }
}
```

`design-tokens/src/themes/govbr-escuro.json`:
```json
{
  "name": "govbr-escuro",
  "colors": {
    "bg": "dk-0", "panel": "dk-1", "sunken": "dk-2", "sunken2": "dk-3", "rule": "dk-4", "rule2": "dk-50",
    "ink": "dk-95", "ink2": "dk-85", "ink3": "dk-70", "ink4": "dk-50",
    "accent": "dk-blue", "accent-bg": "dk-blue-bg", "on-accent": "black",
    "ok": "dk-green", "ok-bg": "dk-green-bg", "warn": "dk-yellow", "warn-bg": "dk-yellow-bg",
    "down": "dk-red", "down-bg": "dk-red-bg", "info": "dk-blue", "info-bg": "dk-info-bg",
    "focus": "dk-orange"
  },
  "fonts": { "font-sans": "rawline", "font-mono": "mono" }
}
```

- [ ] **Step 4: Escrever o teste do gerador, que vai falhar.** `design-tokens/test/build.test.mjs`:

```js
// Gerador: todo tema define todos os papéis com nomes que existem na base, e o
// dist/ versionado é exatamente a saída do gerador (ninguém edita o dist à mão).
import { test } from "node:test";
import assert from "node:assert/strict";
import { readFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";
import { build, loadRoles, loadThemes, resolveThemes } from "../build.mjs";

const root = join(dirname(fileURLToPath(import.meta.url)), "..");

test("todo tema define exatamente os papéis de roles.json", () => {
  const { colors } = loadRoles();
  for (const theme of loadThemes()) {
    assert.deepEqual(Object.keys(theme.colors).sort(), [...colors].sort(), theme.name);
  }
});

test("todo valor de tema existe na paleta e vira #rrggbb", () => {
  for (const [name, roles] of Object.entries(resolveThemes())) {
    for (const [role, hex] of Object.entries(roles)) {
      assert.match(hex, /^#[0-9a-f]{6}$/, `${name}.${role}`);
    }
  }
});

test("os quatro temas existem e o govbr é o padrão do CSS", () => {
  assert.deepEqual(loadThemes().map(t => t.name).sort(), ["alto-contraste", "govbr", "govbr-escuro", "rota"]);
  const { css } = build();
  assert.match(css, /:root:not\(\[data-theme\]\),\n:root\[data-theme="govbr"\] \{/);
});

test("apelidos apontam para papéis e saem como var()", () => {
  const { css } = build();
  assert.match(css, /--line: var\(--rule2\);/);
  assert.match(css, /--danger: var\(--down\);/);
  assert.match(css, /--surface: var\(--sunken\);/);
});

test("dist/ versionado é igual à saída do gerador", () => {
  const { css, ts } = build();
  assert.equal(readFileSync(join(root, "dist/tokens.css"), "utf8"), css);
  assert.equal(readFileSync(join(root, "dist/tokens.ts"), "utf8"), ts);
});
```

- [ ] **Step 5: Rodar e confirmar que falha.**

  Run: `node --test design-tokens/test/build.test.mjs`
  Expected: FAIL, com `Cannot find module '.../design-tokens/build.mjs'`.

- [ ] **Step 6: Escrever `design-tokens/build.mjs`.**

```js
#!/usr/bin/env node
// Gerador dos tokens: lê src/ (base, roles, themes) e emite dist/tokens.css e
// dist/tokens.ts. Sem dependências. build() é puro; só o CLI grava arquivos.
import { readFileSync, readdirSync, writeFileSync, mkdirSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const root = dirname(fileURLToPath(import.meta.url));
const read = (p) => JSON.parse(readFileSync(join(root, p), "utf8"));
const HEADER = "Gerado por design-tokens/build.mjs a partir de src/. Não edite.";
export const DEFAULT_THEME = "govbr";

export const loadBase = () => read("src/base.json");
export const loadRoles = () => read("src/roles.json");
export function loadThemes() {
  return readdirSync(join(root, "src/themes"))
    .filter((f) => f.endsWith(".json"))
    .sort()
    .map((f) => read(`src/themes/${f}`));
}

export function resolveThemes() {
  const base = loadBase();
  const { colors } = loadRoles();
  const out = {};
  for (const theme of loadThemes()) {
    out[theme.name] = {};
    for (const role of colors) {
      const key = theme.colors[role];
      if (key === undefined) throw new Error(`${theme.name}: papel ${role} sem valor`);
      const hex = base.colors[key];
      if (hex === undefined) throw new Error(`${theme.name}.${role}: ${key} não existe na base`);
      out[theme.name][role] = hex.toLowerCase();
    }
  }
  return out;
}

function themeBlock(theme, resolved, fonts) {
  const lines = Object.entries(resolved).map(([role, hex]) => `  --${role}: ${hex};`);
  for (const [prop, key] of Object.entries(theme.fonts)) {
    if (fonts[key] === undefined) throw new Error(`${theme.name}.${prop}: fonte ${key} não existe na base`);
    lines.push(`  --${prop}: ${fonts[key]};`);
  }
  return lines.join("\n");
}

export function build() {
  const base = loadBase();
  const roles = loadRoles();
  const themes = loadThemes();
  const resolved = resolveThemes();

  const faces = roles.fontFaces.map((f) =>
    `@font-face {\n  font-family: "${f.family}";\n  src: url("./fonts/${f.file}") format("woff2");\n` +
    `  font-weight: ${f.weight};\n  font-style: normal;\n  font-display: swap;\n}`
  );

  const shared = [
    ...Object.entries(roles.constants).map(([k, v]) => `  --${k}: ${v};`),
    ...Object.entries(roles.aliases).map(([k, v]) => `  --${k}: var(--${v});`)
  ].join("\n");

  const ordered = [
    ...themes.filter((t) => t.name === DEFAULT_THEME),
    ...themes.filter((t) => t.name !== DEFAULT_THEME)
  ];
  const blocks = ordered.map((t) => {
    const selector = t.name === DEFAULT_THEME
      ? `:root:not([data-theme]),\n:root[data-theme="${t.name}"]`
      : `:root[data-theme="${t.name}"]`;
    return `${selector} {\n${themeBlock(t, resolved[t.name], base.fonts)}\n}`;
  });

  const css = [`/* ${HEADER} */`, ...faces, `:root {\n${shared}\n}`, ...blocks].join("\n\n") + "\n";

  const names = themes.map((t) => t.name).sort();
  const ts = [
    `// ${HEADER}`,
    `export const THEMES = ${JSON.stringify(names)} as const;`,
    `export type ThemeName = (typeof THEMES)[number];`,
    `export const DEFAULT_THEME: ThemeName = "${DEFAULT_THEME}";`,
    `export const COLOR_ROLES = ${JSON.stringify(roles.colors)} as const;`,
    `export type ColorRole = (typeof COLOR_ROLES)[number];`,
    ""
  ].join("\n");

  return { css, ts };
}

if (process.argv[1] && fileURLToPath(import.meta.url) === process.argv[1]) {
  const { css, ts } = build();
  mkdirSync(join(root, "dist"), { recursive: true });
  writeFileSync(join(root, "dist/tokens.css"), css);
  writeFileSync(join(root, "dist/tokens.ts"), ts);
  console.log("dist/tokens.css e dist/tokens.ts gerados");
}
```

- [ ] **Step 7: Gerar o `dist/` e rodar o teste.**

  Run: `node design-tokens/build.mjs && node --test design-tokens/test/build.test.mjs`
  Expected: imprime `dist/tokens.css e dist/tokens.ts gerados`, e os 5 testes passam.

  Abra `design-tokens/dist/tokens.css` e confira duas coisas:
  - o primeiro bloco de tema é o `govbr`, com `--bg: #f8f8f8;`;
  - o `:root {}` compartilhado tem `--space-4: 16px;` e `--line: var(--rule2);`.

- [ ] **Step 8: Commit.**

```bash
/opt/homebrew/bin/git add design-tokens/src design-tokens/build.mjs design-tokens/test/build.test.mjs design-tokens/dist/tokens.css design-tokens/dist/tokens.ts
/opt/homebrew/bin/git commit -m "feat: add design token sources, themes and generator

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Teste de contraste por tema (contracts)

**Files:**
- Create: `design-tokens/test/contrast.test.mjs`

**Interfaces:**
- Consumes: `resolveThemes()`, `loadRoles()` e `loadThemes()` da Task 1.
- Produces: `contrastRatio(hexA, hexB): number`, exportado do próprio teste para uso em diagnóstico.

- [ ] **Step 1: Escrever o teste.** `design-tokens/test/contrast.test.mjs`:

```js
// Mínimos de contraste WCAG 2.2 por tema. Texto 4,5:1 e borda/foco 3:1; no
// alto-contraste (minContrast 7) todo par sobe para 7:1. --ink4 fica fora:
// só aparece em texto desabilitado ou decorativo.
import { test } from "node:test";
import assert from "node:assert/strict";
import { loadRoles, loadThemes, resolveThemes } from "../build.mjs";

function luminance(hex) {
  const c = [1, 3, 5].map((i) => parseInt(hex.slice(i, i + 2), 16) / 255)
    .map((v) => (v <= 0.03928 ? v / 12.92 : ((v + 0.055) / 1.055) ** 2.4));
  return 0.2126 * c[0] + 0.7152 * c[1] + 0.0722 * c[2];
}

export function contrastRatio(a, b) {
  const [x, y] = [luminance(a), luminance(b)];
  return (Math.max(x, y) + 0.05) / (Math.min(x, y) + 0.05);
}

test("contrastRatio confere com valores conhecidos", () => {
  assert.equal(Math.round(contrastRatio("#000000", "#ffffff") * 100) / 100, 21);
  assert.equal(Math.round(contrastRatio("#767676", "#ffffff") * 100) / 100, 4.54);
});

const { pairs } = loadRoles();
const resolved = resolveThemes();

for (const theme of loadThemes()) {
  const textMin = theme.minContrast ?? 4.5;
  const nonTextMin = theme.minContrast ?? 3;
  const roles = resolved[theme.name];

  test(`${theme.name}: texto atinge ${textMin}:1`, () => {
    for (const [fg, bg] of pairs.text) {
      const r = contrastRatio(roles[fg], roles[bg]);
      assert.ok(r >= textMin, `${fg} sobre ${bg} = ${r.toFixed(2)}:1`);
    }
  });

  test(`${theme.name}: borda e foco atingem ${nonTextMin}:1`, () => {
    for (const [fg, bg] of pairs.nonText) {
      const r = contrastRatio(roles[fg], roles[bg]);
      assert.ok(r >= nonTextMin, `${fg} contra ${bg} = ${r.toFixed(2)}:1`);
    }
  });
}
```

- [ ] **Step 2: Provar que o teste pega uma regressão.** Em `src/themes/govbr.json`, troque `"ink3": "gray-60"` por `"ink3": "gray-50"` e rode:

  Run: `node --test design-tokens/test/contrast.test.mjs`
  Expected: FAIL em `govbr: texto atinge 4.5:1`, com `ink3 sobre bg = 3.3`, ou valor próximo.

  Desfaça a troca (`"ink3": "gray-60"`).

- [ ] **Step 3: Rodar com os valores certos.**

  Run: `node --test design-tokens/test/*.test.mjs`
  Expected: PASS em todos (5 do gerador, 1 de valores conhecidos e 8 de temas).

- [ ] **Step 4: Commit.**

```bash
/opt/homebrew/bin/git add design-tokens/test/contrast.test.mjs
/opt/homebrew/bin/git commit -m "test: enforce WCAG contrast minimums for every token theme

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Fonte Rawline, documentação e tag `tokens-v1.0.0` (contracts)

**Files:**
- Create: `design-tokens/dist/fonts/rawline-400.woff2`, `rawline-600.woff2`, `rawline-700.woff2`, `design-tokens/dist/fonts/OFL.txt`
- Modify: `design-tokens/src/roles.json` (o campo `fontFaces`)
- Modify (gerado): `design-tokens/dist/tokens.css`
- Modify: `design-tokens/test/build.test.mjs` (teste das faces)
- Rewrite: `design-tokens/README.md`
- Create: `design-tokens/CHANGELOG.md`
- Modify: `README.md` (a linha `design-tokens` da tabela de domínios)

**Interfaces:**
- Produces: a tag `tokens-v1.0.0`, criada localmente e ainda não publicada. O `dist/` dela é o que as Tasks 4 a 7 copiam.

- [ ] **Step 1: Pedir permissão para o download.**
  - A Rawline está licenciada sob a SIL OFL 1.1. É um derivado da Raleway, com algarismos alinhados, e a OFL permite redistribuir e embarcar a fonte em software.
  - Fonte proposta: o CDN do Design System gov.br (`https://cdngovbr-ds.estaleiro.serpro.gov.br/design-system/fonts/rawline/css/rawline.css`), do qual se extraem as URLs `woff2` dos pesos 400, 600 e 700.
  - Diga ao usuário os nomes dos arquivos, a origem e o tamanho (cada um tem cerca de 30–60 KB) e **espere um sim** antes de baixar.
  - Se o usuário recusar, ou se o pacote não trouxer o texto da OFL, use a **reserva**:
    1. Rode `npm pack @fontsource/raleway` no scratchpad.
    2. Copie `files/raleway-latin-{400,600,700}-normal.woff2` como `raleway-{400,600,700}.woff2`, junto com o `LICENSE` (OFL).
    3. Use `"family": "Raleway"` nos passos seguintes e troque a pilha `rawline` da base para `"\"Raleway\", Verdana, system-ui, sans-serif"`.
    4. Registre a troca no CHANGELOG.

- [ ] **Step 2: Colocar os arquivos.** Baixe cada arquivo num diretório vazio do scratchpad. Confira que é woff2 de verdade:

```bash
file <arquivo>
```

  A saída esperada contém `Web Open Font Format (Version 2)`. Copie os arquivos para `design-tokens/dist/fonts/` e ponha o texto da OFL 1.1 em `design-tokens/dist/fonts/OFL.txt`.

- [ ] **Step 3: Escrever o teste das faces, que vai falhar.** Acrescente ao `design-tokens/test/build.test.mjs`:

```js
import { existsSync } from "node:fs";

test("cada @font-face aponta para um arquivo que existe em dist/fonts", () => {
  const { fontFaces } = loadRoles();
  assert.ok(fontFaces.length >= 3, "faces da fonte do govbr declaradas");
  for (const f of fontFaces) {
    assert.ok(existsSync(join(root, "dist/fonts", f.file)), f.file);
  }
  assert.ok(existsSync(join(root, "dist/fonts/OFL.txt")), "licença junto da fonte");
});
```

  Run: `node --test design-tokens/test/build.test.mjs`
  Expected: FAIL, com `faces da fonte do govbr declaradas`.

- [ ] **Step 4: Declarar as faces e regerar.** Em `design-tokens/src/roles.json`, troque `"fontFaces": []` por:

```json
"fontFaces": [
  { "family": "Rawline", "weight": 400, "file": "rawline-400.woff2" },
  { "family": "Rawline", "weight": 600, "file": "rawline-600.woff2" },
  { "family": "Rawline", "weight": 700, "file": "rawline-700.woff2" }
]
```

  Run: `node design-tokens/build.mjs && node --test design-tokens/test/*.test.mjs`
  Expected: PASS em todos. O `dist/tokens.css` agora começa com três `@font-face`.

- [ ] **Step 5: Reescrever `design-tokens/README.md`.**

```markdown
# design-tokens — cores, tipografia e espaçamento como dado

O design system do Rota Saúde é compartilhado **como tokens**, não como componentes (ADR 0002, ADR 0025): cada frontend tem os próprios componentes e consome estes tokens.

## Camadas (`src/`)

- `base.json` — paleta crua (nome → hex) e pilhas de fonte.
- `roles.json` — papéis que os componentes leem (`--bg`, `--ink`, `--accent`, `--ok`…), apelidos (`--line`, `--danger`, `--surface`), constantes (espaço, raio, sombra), faces de fonte e os pares de contraste.
- `themes/*.json` — cada tema liga cada papel a um nome da paleta.

Temas: `govbr` (padrão: painéis e wpda), `rota` (identidade própria; maintenance e opcional nos painéis), `alto-contraste` (eMAG) e `govbr-escuro` (só wpda).

## Gerar

`node design-tokens/build.mjs` grava `dist/tokens.css` (um bloco `:root[data-theme="…"]` por tema; sem atributo vale o `govbr`) e `dist/tokens.ts` (nomes de tema e de papel). O `dist/` é versionado; nunca edite à mão.

## Validar

`node --test design-tokens/test/*.test.mjs`. O teste de contraste exige texto ≥ 4,5:1, borda e foco ≥ 3:1 e, no `alto-contraste`, tudo ≥ 7:1. Tag nenhuma sai com ele vermelho.

## Consumir

Cada app roda `node scripts/tokens-sync.mjs <tag> --contracts <caminho do clone>`, que copia `design-tokens/dist` da tag para `src/theme/vendor/` e grava `tokens.lock` (tag + SHA-256 por arquivo). Um teste do app falha se a cópia divergir do lock. O app importa `vendor/tokens.css` no `global.css` e escolhe o tema com `data-theme` no `<html>`.

## Fonte

Rawline (SIL OFL 1.1, derivada da Raleway), em `dist/fonts/` com a licença ao lado. Reserva da pilha: Verdana (recomendação do manual da marca gov.br), depois a fonte do sistema.

## Componentes que divergiam entre dashboard e admin

DataTable, KpiGrid, Sparkline, StatTile, PageHeader, BarMini, Panel e AppHeader têm cópias diferentes por **comportamento** (lacunas `null`, supressão de contagem pequena, `asOf`, `right`, fuso), não por estilo: as duas cópias já leem os mesmos papéis. A unificação visual vem destes tokens; a lógica de cada cópia fica no seu app.

## Versão

Tags `tokens-vX.Y.Z` (ADR 0015). Histórico em `CHANGELOG.md`.
```

- [ ] **Step 6: Criar o `design-tokens/CHANGELOG.md` e atualizar o README raiz.**

```markdown
# Changelog — design-tokens

## tokens-v1.0.0 — 2026-10-12

- Primeira versão: camadas `base`/`roles`/`themes`, gerador e teste de contraste.
- Temas `govbr` (padrão), `rota`, `alto-contraste` e `govbr-escuro`.
- Papéis novos: `--focus`, `--on-accent`, `--space-1`…`--space-8`; apelidos `--line`, `--danger`, `--surface`.
- Fonte Rawline (OFL 1.1) em `dist/fonts/`.
```

  No `README.md` raiz, troque a linha da tabela de domínios:

  `| [`design-tokens/`](design-tokens/README.md) | Cores, espaçamento e tipografia como dado | — | Scaffold: precisa de input de design |`

  por:

  `| [`design-tokens/`](design-tokens/README.md) | Cores, espaçamento e tipografia como dado (4 temas, contraste validado) | `tokens-v1.0.0` | Materializado |`

- [ ] **Step 7: Commit e tag local.**

```bash
/opt/homebrew/bin/git add design-tokens/dist/fonts design-tokens/src/roles.json design-tokens/dist/tokens.css design-tokens/test/build.test.mjs design-tokens/README.md design-tokens/CHANGELOG.md README.md
/opt/homebrew/bin/git commit -m "feat: ship Rawline font and document design tokens (tokens-v1.0.0)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
/opt/homebrew/bin/git tag tokens-v1.0.0
```

  Não faça push. A publicação acontece na Task 8, depois da aprovação das capturas.

---

### Task 4: Cópia com trava nos dois apps

Repita esta task **duas vezes**, uma no worktree do dashboard e outra no do admin. O código é idêntico.

**Files** (por app):
- Create: `scripts/tokens-sync.mjs`
- Create (gerado): `src/theme/vendor/tokens.css`, `src/theme/vendor/tokens.ts`, `src/theme/vendor/fonts/*`, `src/theme/vendor/tokens.lock`
- Create: `src/theme/vendor.lock.test.ts`
- Modify: `package.json`. No admin, acrescente `"@types/node": "^22.0.0"` em `devDependencies` e rode `npm install` no worktree. O dashboard já tem a dependência.

**Interfaces:**
- Consumes: a tag `tokens-v1.0.0` do clone local do contracts (Task 3).
- Produces:
  - `src/theme/vendor/tokens.css`, com as variáveis da seção Global Constraints;
  - `src/theme/vendor/tokens.ts`, que exporta `THEMES`, `ThemeName`, `DEFAULT_THEME`, `COLOR_ROLES` e `ColorRole`;
  - o comando `node scripts/tokens-sync.mjs <ref> --contracts <path>`.

- [ ] **Step 1: Escrever o teste do lock, que vai falhar.** `src/theme/vendor.lock.test.ts`:

```ts
// A cópia dos tokens (src/theme/vendor/) é gerada pelo scripts/tokens-sync.mjs
// a partir de uma tag do contracts. Editar a cópia à mão, ou deixar um arquivo
// que o lock não conhece, reprova a suíte.
import { describe, expect, it } from "vitest";
import { readFileSync, readdirSync, statSync } from "node:fs";
import { createHash } from "node:crypto";
import { fileURLToPath } from "node:url";
import { dirname, join, relative } from "node:path";

const dir = join(dirname(fileURLToPath(import.meta.url)), "vendor");

function listFiles(base: string): string[] {
  return readdirSync(base).flatMap((name) => {
    const full = join(base, name);
    return statSync(full).isDirectory() ? listFiles(full) : [relative(dir, full)];
  });
}

describe("cópia dos tokens", () => {
  const lock = JSON.parse(readFileSync(join(dir, "tokens.lock"), "utf8")) as {
    ref: string; files: Record<string, string>;
  };

  it("vem de uma tag tokens-vX.Y.Z", () => {
    expect(lock.ref).toMatch(/^tokens-v\d+\.\d+\.\d+$/);
  });

  it("cada arquivo bate com o hash do lock", () => {
    for (const [file, sha] of Object.entries(lock.files)) {
      const actual = createHash("sha256").update(readFileSync(join(dir, file))).digest("hex");
      expect(actual, file).toBe(sha);
    }
  });

  it("não há arquivo na cópia fora do lock", () => {
    const present = listFiles(dir).filter((f) => f !== "tokens.lock").sort();
    expect(present).toEqual(Object.keys(lock.files).sort());
  });
});
```

  Run: `npx vitest run src/theme/vendor.lock.test.ts`
  Expected: FAIL com `ENOENT` em `vendor/tokens.lock`.

- [ ] **Step 2: Escrever `scripts/tokens-sync.mjs`.**

```js
#!/usr/bin/env node
// Copia design-tokens/dist do contracts, numa ref (tag), para src/theme/vendor/
// e grava src/theme/vendor/tokens.lock (ref + SHA-256 por arquivo).
// Uso: node scripts/tokens-sync.mjs tokens-v1.0.0 --contracts ../../contracts
// GIT=/opt/homebrew/bin/git quando o git do PATH não funcionar.
import { execFileSync } from "node:child_process";
import { cpSync, mkdtempSync, readdirSync, readFileSync, rmSync, statSync, writeFileSync } from "node:fs";
import { createHash } from "node:crypto";
import { tmpdir } from "node:os";
import { dirname, join, relative, resolve } from "node:path";
import { fileURLToPath } from "node:url";

const args = process.argv.slice(2);
const ref = args[0];
const at = args.indexOf("--contracts");
if (!ref || at < 0 || !args[at + 1]) {
  console.error("uso: node scripts/tokens-sync.mjs <ref> --contracts <caminho do clone do contracts>");
  process.exit(1);
}
const contracts = resolve(args[at + 1]);
const git = process.env.GIT || "git";
const appRoot = join(dirname(fileURLToPath(import.meta.url)), "..");
const vendor = join(appRoot, "src/theme/vendor");

const tmp = mkdtempSync(join(tmpdir(), "tokens-"));
const tar = execFileSync(git, ["-C", contracts, "archive", "--format=tar", ref, "design-tokens/dist"]);
execFileSync("tar", ["-x", "-C", tmp], { input: tar });

rmSync(vendor, { recursive: true, force: true });
cpSync(join(tmp, "design-tokens/dist"), vendor, { recursive: true });
rmSync(tmp, { recursive: true, force: true });

const list = (base) => readdirSync(base).flatMap((n) => {
  const full = join(base, n);
  return statSync(full).isDirectory() ? list(full) : [relative(vendor, full)];
});
const files = {};
for (const f of list(vendor).sort()) {
  files[f] = createHash("sha256").update(readFileSync(join(vendor, f))).digest("hex");
}
writeFileSync(join(vendor, "tokens.lock"), JSON.stringify({ ref, files }, null, 2) + "\n");
console.log(`tokens ${ref} copiados para src/theme/vendor (${Object.keys(files).length} arquivos)`);
```

- [ ] **Step 3: Sincronizar a partir da tag.**

```bash
GIT=/opt/homebrew/bin/git node scripts/tokens-sync.mjs tokens-v1.0.0 --contracts /Users/eduardovrocha/Development/ioit.solutions/rota-saude/contracts
```

  Expected: `tokens tokens-v1.0.0 copiados para src/theme/vendor (6 arquivos)`, ou 7 com a reserva Raleway (que traz `LICENSE`).

  O worktree do contracts compartilha as tags com o clone principal, então a tag local da Task 3 já está visível.

- [ ] **Step 4: Rodar o teste do lock.**

  Run: `npx vitest run src/theme/vendor.lock.test.ts`
  Expected: PASS nos 3 testes.

- [ ] **Step 5: Provar que a trava pega a cópia editada.** Acrescente um espaço ao fim de `src/theme/vendor/tokens.css` e rode o mesmo comando.

  Expected: FAIL em `cada arquivo bate com o hash do lock`, com o nome `tokens.css`.

  Rode o sync de novo (Step 3) para restaurar a cópia.

- [ ] **Step 6: Commit** (no repo do app).

```bash
/opt/homebrew/bin/git add scripts/tokens-sync.mjs src/theme/vendor src/theme/vendor.lock.test.ts package.json package-lock.json
/opt/homebrew/bin/git commit -m "build: vendor design tokens tokens-v1.0.0 with a hash lock

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

  No dashboard, o `package.json` não muda nesta task. Tire-o, junto com o lock, do `git add` se o `git status` mostrar os dois sem alteração.

---

### Task 5: Tema `govbr` no lugar do console (dashboard e admin)

Repita nos dois worktrees, com o mesmo código.

**Files** (por app):
- Modify: `src/theme/global.css`
- Modify: `src/theme/tokens.ts`
- Modify: `index.html`
- Modify: `src/main.tsx` (remove os dois imports `@fontsource-variable/geist*`)
- Modify: `package.json` e o lock (remove `@fontsource-variable/geist` e `@fontsource-variable/geist-mono`)
- Create: `src/theme/theme.test.ts`

**Interfaces:**
- Consumes: `src/theme/vendor/tokens.css` (Task 4).
- Produces:
  - `tokens.color.*` passa a valer `"var(--<papel>)"`;
  - `toneColor(t)` devolve `{ fg: "var(--x)", bg: "var(--x-bg)" }`;
  - `toneShape(t): "square" | "triangle" | "circle" | null`, nova e usada na Task 6;
  - os demais exports (`tokens.size`, `tokens.radius`, `tokens.shadow`, `tokens.space`, `Tone`) mantêm nome e valor.

- [ ] **Step 1: Escrever o teste, que vai falhar.** `src/theme/theme.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";
import { tokens, toneColor, toneShape } from "./tokens";

const appRoot = join(dirname(fileURLToPath(import.meta.url)), "../..");

describe("tema", () => {
  it("o index.html abre no tema govbr", () => {
    const html = readFileSync(join(appRoot, "index.html"), "utf8");
    expect(html).toMatch(/<html lang="pt-BR" data-theme="govbr">/);
  });

  it("o global.css importa a cópia dos tokens e não redefine papéis", () => {
    const css = readFileSync(join(appRoot, "src/theme/global.css"), "utf8");
    expect(css.trimStart().startsWith('@import "./vendor/tokens.css";')).toBe(true);
    expect(css).not.toMatch(/--ink:\s/);
  });

  it("toneColor lê papéis do tema ativo", () => {
    expect(toneColor("ok")).toEqual({ fg: "var(--ok)", bg: "var(--ok-bg)" });
    expect(toneColor("urgent")).toEqual({ fg: "var(--down)", bg: "var(--down-bg)" });
    expect(toneColor("accent")).toEqual({ fg: "var(--accent)", bg: "var(--accent-bg)" });
    expect(toneColor(undefined)).toEqual({ fg: "var(--ink3)", bg: "var(--sunken)" });
    expect(tokens.color.ink).toBe("var(--ink)");
  });

  it("toneShape dá forma ao nível, além da cor", () => {
    expect(toneShape("down")).toBe("square");
    expect(toneShape("urgent")).toBe("square");
    expect(toneShape("warn")).toBe("triangle");
    expect(toneShape("ok")).toBe("circle");
    expect(toneShape("info")).toBeNull();
    expect(toneShape("accent")).toBeNull();
    expect(toneShape(undefined)).toBeNull();
  });
});
```

  Run: `npx vitest run src/theme/theme.test.ts`
  Expected: FAIL. `toneShape` não é exportado, e o `index.html` ainda não tem `data-theme`.

- [ ] **Step 2: Reescrever `src/theme/tokens.ts`.**

```ts
// Tokens do painel. Os valores vivem no tema (src/theme/vendor/tokens.css,
// copiado do contracts/design-tokens); aqui ficam os nomes tipados, que leem
// as variáveis do tema ativo (<html data-theme>). Escalas de tamanho, raio,
// sombra e espaço seguem as do console até o shell novo (subprojeto 2).

const v = (role: string) => `var(--${role})`;

export const tokens = {
  color: {
    bg: v("bg"), panel: v("panel"), sunken: v("sunken"), sunken2: v("sunken2"),
    rule: v("rule"), rule2: v("rule2"),
    ink: v("ink"), ink2: v("ink2"), ink3: v("ink3"), ink4: v("ink4"),

    accent: v("accent"), accentBg: v("accent-bg"),
    ok: v("ok"), okBg: v("ok-bg"),
    warn: v("warn"), warnBg: v("warn-bg"),
    down: v("down"), downBg: v("down-bg"),
    info: v("info"), infoBg: v("info-bg")
  },

  font: {
    sans: v("font-sans"),
    mono: v("font-mono")
  },

  size: {
    kpiValue: 27,
    h1: 20,
    kvValue: 14,
    panelTitle: 13.5,
    sectionTitle: 13,
    nav: 12.5,
    cell: 12,
    kpiLabel: 11.5,
    aux: 11,
    badge: 10.5
  },

  radius: {
    panel: 10,
    menu: 12,
    chip: 6,
    pill: 999
  },

  shadow: {
    dropdown: v("shadow-dropdown"),
    notif: v("shadow-notif")
  },

  space: {
    panelGap: 16,
    panelPad: 16,
    panelHeadPad: "13px 16px",
    contentPad: "22px 32px 48px"
  }
} as const;

// Mapeamento de tone → cor semântica + bg.
export type Tone = "ok" | "warn" | "down" | "info" | "accent" | "neutral" | "urgent";

export const toneColor = (t: Tone | string | undefined): { fg: string; bg: string } => {
  switch (t) {
    case "ok":      return { fg: tokens.color.ok,     bg: tokens.color.okBg };
    case "warn":    return { fg: tokens.color.warn,   bg: tokens.color.warnBg };
    case "down":
    case "urgent":  return { fg: tokens.color.down,   bg: tokens.color.downBg };
    case "info":    return { fg: tokens.color.info,   bg: tokens.color.infoBg };
    case "accent":  return { fg: tokens.color.accent, bg: tokens.color.accentBg };
    default:        return { fg: tokens.color.ink3,   bg: tokens.color.sunken };
  }
};

// Nível também por forma (WCAG 1.4.1: cor nunca é o único sinal).
// ■ alto/urgente · ▲ moderado · ● baixo/ok; demais tons não carregam nível.
export type Shape = "square" | "triangle" | "circle";

export const toneShape = (t: Tone | string | undefined): Shape | null => {
  switch (t) {
    case "down":
    case "urgent": return "square";
    case "warn":   return "triangle";
    case "ok":     return "circle";
    default:       return null;
  }
};
```

  Antes de salvar, confira com `grep -rn "tokens\.shadow\." src` que nenhum consumidor espera o valor literal da sombra. Na exploração, só o `toneColor` lia `tokens.color`, e ninguém lia `tokens.shadow` nem `tokens.font`.

- [ ] **Step 3: Reescrever `src/theme/global.css`.**

```css
@import "./vendor/tokens.css";

/* Globals do painel. Papéis e temas vêm de vendor/tokens.css
   (contracts/design-tokens); o tema ativo é o data-theme do <html>. */

* { box-sizing: border-box; }

html, body, #root {
  margin: 0;
  padding: 0;
  height: 100%;
}

body {
  background: var(--bg);
  color: var(--ink);
  font-family: var(--font-sans);
  font-size: 14px;
  line-height: 1.45;
  -webkit-font-smoothing: antialiased;
  text-rendering: optimizeLegibility;
}

button {
  font-family: inherit;
  color: inherit;
  background: transparent;
  border: 0;
  cursor: pointer;
  padding: 0;
}

/* Foco visível em todo elemento focável (WCAG 2.4.7 / 2.4.11). */
:focus-visible {
  outline: 3px solid var(--focus);
  outline-offset: 2px;
}

a {
  color: var(--accent);
  text-decoration: none;
}

.mono { font-family: var(--font-mono); }

/* Forma do nível no Tag (toneShape), fora do textContent e com texto
   alternativo vazio: o leitor de tela lê só o rótulo. */
[data-glyph]::before { margin-right: 4px; }
[data-glyph="square"]::before   { content: "■" / ""; }
[data-glyph="triangle"]::before { content: "▲" / ""; }
[data-glyph="circle"]::before   { content: "●" / ""; }

@keyframes pulse-dot {
  0%, 100% { box-shadow: 0 0 0 0 currentColor; opacity: 1; }
  50%      { box-shadow: 0 0 0 6px transparent; opacity: .6; }
}

@keyframes skeleton-shimmer {
  0%   { background-position: -200px 0; }
  100% { background-position: calc(200px + 100%) 0; }
}
```

- [ ] **Step 4: Ajustar o `index.html`, o `main.tsx` e as dependências.**
  - No `index.html`, troque `<html lang="pt-BR">` por `<html lang="pt-BR" data-theme="govbr">`.
  - No `src/main.tsx`, apague as linhas `import "@fontsource-variable/geist/index.css";` e `import "@fontsource-variable/geist-mono/index.css";`. No admin, apague também o comentário `// Fontes self-hosted via @fontsource …` que vem logo antes e troque-o por `// Fonte do tema (Rawline) vem de src/theme/vendor/fonts, sem fetch externo.`
  - Rode:

```bash
npm uninstall @fontsource-variable/geist @fontsource-variable/geist-mono
```

- [ ] **Step 5: Rodar o teste novo e a suíte inteira.**

  Run: `npx vitest run src/theme/theme.test.ts`
  Expected: PASS nos 4 testes.

  Run: `npx vitest run 2>&1 | tail -5`
  Expected: todos verdes, com as mesmas contagens da linha de base da Task 0 mais os testes novos. **Nenhum teste existente foi editado.** Se algum quebrar, investigue a mudança, não o teste.

  Run: `npm run build`
  Expected: `tsc -b` e o `vite build` sem erro. No `dist/assets`, a saída tem os `rawline-*.woff2` com hash.

- [ ] **Step 6: Conferência visual da cor em SVG (Review Focus 1).**
  1. Suba o app do worktree (`npx vite --port 5185 --strictPort` no dashboard, `5186` no admin).
  2. Abra no navegador embutido uma tela com Sparkline: a Visão geral, nos KPIs com tendência.
  3. Rode `getComputedStyle(document.querySelector('.recharts-area-curve')).stroke` com o `javascript_tool`.
  4. O valor esperado é uma cor resolvida (por exemplo `rgb(19, 81, 180)`), não `none` nem vazio.
  5. Tire uma captura para o relatório.
  6. Se o atributo SVG não resolver `var()`, pare e reporte. A correção possível, um `style={{ stroke }}` no Sparkline, é decisão do usuário, porque toca componente divergente.

- [ ] **Step 7: Commit.**

```bash
/opt/homebrew/bin/git add src/theme/global.css src/theme/tokens.ts src/theme/theme.test.ts index.html src/main.tsx package.json package-lock.json
/opt/homebrew/bin/git commit -m "feat: switch panel theme to govbr design tokens with Rawline

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

  O admin tem `pnpm-lock.yaml` além do `package-lock.json`. Inclua os lockfiles que o `git status` mostrar alterados.

---

### Task 6: Nível por forma no `Tag` e no `StatusDot` (dashboard e admin)

Repita nos dois worktrees, com o mesmo código.

**Files** (por app):
- Modify: `src/components/Tag.tsx`
- Modify: `src/components/StatusDot.tsx`
- Create: `src/components/Tag.test.tsx`
- Create: `src/components/StatusDot.test.tsx`

**Interfaces:**
- Consumes: `toneShape()` e `toneColor()` de `src/theme/tokens.ts` (Task 5), e as regras `[data-glyph]` do `global.css` (Task 5).
- Produces:
  - `<Tag>` ganha o atributo `data-glyph="square|triangle|circle"` quando o tom tem nível;
  - `<StatusDot>` ganha `data-shape="square|triangle|circle"`, sempre presente, com círculo como padrão;
  - as props não mudam.

- [ ] **Step 1: Escrever os testes, que vão falhar.** `src/components/Tag.test.tsx`:

```tsx
import { afterEach, describe, expect, it } from "vitest";
import { cleanup, render } from "@testing-library/react";
import { Tag } from "./Tag";

afterEach(cleanup);

describe("Tag", () => {
  it.each([
    ["down", "square"], ["urgent", "square"], ["warn", "triangle"], ["ok", "circle"]
  ])("tom %s marca a forma %s além da cor", (tone, shape) => {
    const { container } = render(<Tag tone={tone}>alto</Tag>);
    const el = container.firstElementChild as HTMLElement;
    expect(el.getAttribute("data-glyph")).toBe(shape);
  });

  it.each(["info", "accent", "neutral", undefined])("tom %s não carrega nível", (tone) => {
    const { container } = render(<Tag tone={tone}>publicado</Tag>);
    expect((container.firstElementChild as HTMLElement).hasAttribute("data-glyph")).toBe(false);
  });

  it("a forma não entra no texto: textContent é só o rótulo", () => {
    const { container } = render(<Tag tone="down">alto</Tag>);
    expect(container.textContent).toBe("alto");
  });
});
```

`src/components/StatusDot.test.tsx`:
```tsx
import { afterEach, describe, expect, it } from "vitest";
import { cleanup, render } from "@testing-library/react";
import { StatusDot } from "./StatusDot";

afterEach(cleanup);

const dot = (level?: string) =>
  render(<StatusDot level={level} />).container.firstElementChild as HTMLElement;

describe("StatusDot", () => {
  it("alto/urgente é quadrado", () => {
    expect(dot("down").getAttribute("data-shape")).toBe("square");
    expect(dot("urgent").getAttribute("data-shape")).toBe("square");
  });

  it("moderado é triângulo", () => {
    // Só o atributo: o jsdom descarta clip-path do style inline, então o
    // desenho do triângulo é conferido nas capturas da Task 8.
    expect(dot("warn").getAttribute("data-shape")).toBe("triangle");
  });

  it("ok e demais tons são círculo", () => {
    expect(dot("ok").getAttribute("data-shape")).toBe("circle");
    expect(dot("info").getAttribute("data-shape")).toBe("circle");
    expect(dot(undefined).getAttribute("data-shape")).toBe("circle");
  });

  it("continua decorativo para o leitor de tela", () => {
    expect(dot("down").getAttribute("aria-hidden")).toBe("true");
  });
});
```

  Run: `npx vitest run src/components/Tag.test.tsx src/components/StatusDot.test.tsx`
  Expected: FAIL. `data-glyph` e `data-shape` são `null`.

- [ ] **Step 2: Implementar no `Tag.tsx`.**

```tsx
// Tag/Badge — pílula de status. tone semântico; tom com nível também
// mostra a forma (■ ▲ ●) via CSS, fora do texto (global.css [data-glyph]).
import type { ReactNode } from "react";
import { toneColor, toneShape, type Tone } from "../theme/tokens";

interface Props {
  tone?: Tone | string;
  mono?: boolean;
  children: ReactNode;
}

export function Tag({ tone, mono = true, children }: Props) {
  const { fg, bg } = toneColor(tone);
  const shape = toneShape(tone);
  return (
    <span
      className={mono ? "mono" : ""}
      data-glyph={shape ?? undefined}
      style={{
        display: "inline-flex",
        alignItems: "center",
        padding: "2px 8px",
        borderRadius: 999,
        fontSize: 10.5,
        color: fg,
        background: bg,
        border: `1px solid color-mix(in oklch, ${fg}, transparent 75%)`,
        textTransform: "lowercase",
        letterSpacing: 0.2,
        whiteSpace: "nowrap"
      }}
    >
      {children}
    </span>
  );
}
```

- [ ] **Step 3: Implementar no `StatusDot.tsx`.**

```tsx
// StatusDot — marcador de status. pulse = halo "ao vivo". A forma acompanha
// o nível (■ alto · ▲ moderado · ● demais), para a cor não ser o único sinal.
import { toneColor, toneShape, type Tone } from "../theme/tokens";

interface Props {
  level?: Tone | string;
  size?: number;
  pulse?: boolean;
}

const TRIANGLE = "polygon(50% 0, 100% 100%, 0 100%)";

export function StatusDot({ level, size = 8, pulse }: Props) {
  const { fg } = toneColor(level);
  const shape = toneShape(level) ?? "circle";
  return (
    <span
      aria-hidden
      data-shape={shape}
      style={{
        display: "inline-block",
        width: size,
        height: size,
        borderRadius: shape === "circle" ? 999 : shape === "square" ? 1 : 0,
        clipPath: shape === "triangle" ? TRIANGLE : undefined,
        background: fg,
        color: fg,
        animation: pulse ? "pulse-dot 1.6s ease-in-out infinite" : undefined,
        flexShrink: 0
      }}
    />
  );
}
```

- [ ] **Step 4: Rodar os testes novos e a suíte.**

  Run: `npx vitest run src/components/Tag.test.tsx src/components/StatusDot.test.tsx`
  Expected: PASS.

  Run: `npx vitest run 2>&1 | tail -5`
  Expected: todos verdes, sem nenhum teste existente editado. Se algum teste que confere `textContent` quebrar, a forma vazou para o texto, e o problema é a implementação.

- [ ] **Step 5: Commit.**

```bash
/opt/homebrew/bin/git add src/components/Tag.tsx src/components/StatusDot.tsx src/components/Tag.test.tsx src/components/StatusDot.test.tsx
/opt/homebrew/bin/git commit -m "feat: show risk level by shape as well as color in Tag and StatusDot

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Cores fixas para papéis (dashboard; o admin é só conferência)

**Files** (dashboard):
- Modify: `src/modules/Login.tsx` (linhas com `color: "#fff"` em botão sobre `--accent`; na exploração, cerca das linhas 49 e 109)
- Modify: `src/modules/ResetPassword.tsx` (cerca da linha 69)
- Modify: `src/modules/AcceptInvitation.tsx` (cerca da linha 80)
- Modify: `src/components/formStyles.ts` (cerca da linha 11)
- Create: `src/theme/literals.test.ts`

**Interfaces:**
- Consumes: o papel `--on-accent` (Task 1).
- Produces: nada novo para outras tasks.

- [ ] **Step 1: Escrever a guarda, que vai falhar.** `src/theme/literals.test.ts`:

```ts
// Texto sobre --accent usa --on-accent (no alto-contraste o accent é amarelo e
// o texto precisa ser preto). Guarda contra `color: "#fff"` fixo nesses botões.
import { describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const src = join(dirname(fileURLToPath(import.meta.url)), "..");
const FILES = [
  "modules/Login.tsx", "modules/ResetPassword.tsx",
  "modules/AcceptInvitation.tsx", "components/formStyles.ts"
];

describe("texto sobre accent", () => {
  it.each(FILES)("%s não fixa branco sobre o accent", (file) => {
    const code = readFileSync(join(src, file), "utf8");
    expect(code).not.toMatch(/color:\s*"#fff"/);
  });
});
```

  Run: `npx vitest run src/theme/literals.test.ts`
  Expected: FAIL nos 4 arquivos.

- [ ] **Step 2: Trocar os literais.** Em cada um dos quatro arquivos, troque **só** as ocorrências de `color: "#fff"` por `color: "var(--on-accent, #fff)"`. Exemplo, no `formStyles.ts`:

```ts
  background: "var(--accent, #2b59ff)", color: "var(--on-accent, #fff)", fontSize: 13
```

  Não toque nos `var(--x, #fallback)` que já existem. Os nomes `--line`, `--danger` e `--surface` passam a existir pelos apelidos dos tokens. O `rgba(20,20,40,0.32)` dos véus de modal (Protocols e Classification) e as sombras do `SegmentedControl` ficam como estão, porque são escurecimento e não cor de tema.

- [ ] **Step 3: Rodar a guarda e a suíte.**

  Run: `npx vitest run src/theme/literals.test.ts && npx vitest run 2>&1 | tail -5`
  Expected: PASS na guarda, e a suíte toda verde, sem teste existente editado.

- [ ] **Step 4: Conferir o admin.** Rode no worktree do admin:

```bash
grep -rnE 'color:\s*"#fff"' src
```

  Expected: nenhuma ocorrência sobre `--accent`. O único literal conhecido é o `background: "#fff"` do `QrCode`, que é obrigatório para a leitura do QR e fica como está.

- [ ] **Step 5: Commit** (dashboard).

```bash
/opt/homebrew/bin/git add src/modules/Login.tsx src/modules/ResetPassword.tsx src/modules/AcceptInvitation.tsx src/components/formStyles.ts src/theme/literals.test.ts
/opt/homebrew/bin/git commit -m "style: use on-accent token for text on accent buttons

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Capturas "depois", aprovação, merge e publicação

**Files:**
- Create: `docs/evidencias/2026-10-12-tokens-foundation/{antes,depois}/*.png`
- Create: `docs/evidencias/2026-10-12-tokens-foundation/README.md`

- [ ] **Step 1: Capturas "depois".** Repita o Step 4 da Task 0 nas mesmas telas, com a mesma conta e a janela em 1280×800. Salve em `docs/evidencias/2026-10-12-tokens-foundation/depois/<tela>.png`.

  Em cada captura, confira três coisas:
  - nenhum texto cortado ou sobreposto por causa do corpo de 14px (Review Focus 5);
  - o risco aparece com forma;
  - o foco âmbar aparece ao navegar com Tab.

  Se o corpo maior apertar alguma tela, ajuste só espaçamento, com commit `style:` e uma justificativa. Nunca mude a estrutura.

- [ ] **Step 2: Escrever o `README.md` da evidência.**

```markdown
# Evidência — ciclo de interface, subprojeto 1 (tokens-v1.0.0)

Spec: `docs/superpowers/specs/2026-10-12-interface-cycle-foundation-design.md` · Plano: `docs/superpowers/plans/2026-10-12-interface-tokens-foundation.md`

| Tela | Antes | Depois |
|---|---|---|
| Dashboard · Visão geral | antes/dashboard-visao-geral.png | depois/dashboard-visao-geral.png |
| Dashboard · Triagens | antes/dashboard-triagens.png | depois/dashboard-triagens.png |
| Dashboard · Atendimento | antes/dashboard-atendimento.png | depois/dashboard-atendimento.png |
| Dashboard · Protocolos | antes/dashboard-protocolos.png | depois/dashboard-protocolos.png |
| Dashboard · Equipe | antes/dashboard-equipe.png | depois/dashboard-equipe.png |
| Admin · Visão geral | antes/admin-visao-geral.png | depois/admin-visao-geral.png |
| Admin · Cidades | antes/admin-cidades.png | depois/admin-cidades.png |

Suítes: dashboard <N>/0 (linha de base <M>), admin <N>/0 (linha de base <M>), contracts design-tokens <N>/0. Nenhuma asserção existente editada.
```

  Troque `<N>` e `<M>` pelos números medidos.

- [ ] **Step 3: Aprovação do usuário.** Mostre as capturas lado a lado (antes e depois) e a cor resolvida no SVG da Task 5. **Espere a aprovação explícita** antes de seguir.

- [ ] **Step 4: Suítes finais.**

```bash
node --test design-tokens/test/*.test.mjs
```

  Rode isso no worktree do contracts. No dashboard e no admin, rode `npx vitest run` e `npm run build`. Todas têm que estar verdes.

- [ ] **Step 5: Merge e push, um repo por vez, contracts primeiro.** Em cada repo:
  1. Confira `origin/main..main`. Se houver commits de outra sessão, publique só os seus a partir do worktree e avise a dona.
  2. Faça o merge `--ff-only` de `ui/tokens-foundation` em `main`.
  3. Rode `git push origin main`.
  4. Só no contracts, rode também `git push origin tokens-v1.0.0`.
  5. Remova o worktree com `git worktree remove .claude/worktrees/ui-tokens`.

  Por fim, commite e publique a evidência e a atualização do plano no `docs`:

```bash
/opt/homebrew/bin/git add evidencias/2026-10-12-tokens-foundation
/opt/homebrew/bin/git commit -m "docs: add before/after evidence for the token foundation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 6: Avisar as sessões donas** do dashboard e do admin, com os hashes publicados. Informe também o comando de sync para versões futuras:

```bash
node scripts/tokens-sync.mjs tokens-vX.Y.Z --contracts <clone do contracts>
```
