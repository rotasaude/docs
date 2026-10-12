# Ciclo de interface — subprojeto 2: shell dos painéis — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Trocar o cabeçalho lotado do dashboard e do admin por um shell com barra eMAG, barra de topo, faixa de contexto, trilho, gaveta, Início em cards, trilha de navegação, busca de tela, roteamento por hash e tema por pessoa. As telas e o comportamento delas não mudam.

**Architecture:**
- **Peças idênticas nos dois apps** (copiadas, ADR 0002), todas puras e dirigidas por props: `router.ts` (hash ↔ rota, `useHashRoute`), componentes de apresentação em `src/shell/` e `theme/theme.ts`.
- **O que cada app monta à sua maneira:** `App.tsx` calcula os grupos visíveis com o filtro de papel de hoje, passa a rota ao `Shell` e escolhe o conteúdo (Início, grupo, acessibilidade ou a tela, pelo `renderModule` de hoje). Os atalhos entre telas passam a chamar `go`.
- **Contracts:** ganha a Figtree e o papel `--on-tone` em `tokens-v1.1.0`.

**Tech Stack:** React 18, Vite 5, Vitest 2, jsdom, Testing Library, `@tabler/icons-react` 3 e CSS com `@media` (`src/shell/shell.css`).

**Spec:** `docs/superpowers/specs/2026-10-12-interface-panel-shell-design.md`. A spec do ciclo também vale: `docs/superpowers/specs/2026-10-12-interface-cycle-foundation-design.md`.

## Global Constraints

- **Só interface.** As telas, os dados, as permissões e as chamadas ao api não mudam. As únicas mudanças de uso permitidas são:
  1. o Início como entrada;
  2. o Voltar do navegador subindo um nível;
  3. o endereço refletindo a tela (`#/…`).
- **Testes anteriores ao ciclo** passam **sem nenhuma asserção editada**. A guarda `a11y-guards.test.ts` foi criada neste ciclo e só troca o arquivo que vigia (Task 9).
- **O que se mantém:**
  - os rótulos de `NAV_GROUPS`, os ids e os filtros (`navGroupsFor` no dashboard e `visible` no admin);
  - o `SignatureSessionBadge` sem nenhuma mudança;
  - o `NotificationCenter`, que só ganha a cor do selo (Task 10);
  - o `ScopeContext` (período);
  - o `?bairro=`;
  - o `initialModule`;
  - o `return_to` `"/"+id`.
- **Rotas:** vazio ou `#/` é o Início, `#/grupo/<slug>` é o grupo, `#/<id>` é a tela e `#/acessibilidade` é a página de acessibilidade. Um hash inválido, ou uma rota que o papel não vê, vira o Início com `replaceState`. Um atalho interno (`go`) não é filtrado.
- **Larguras:**
  - de 768 a 1023px: gaveta, 2 cards por linha;
  - de 1024 a 1279px: trilho, 3 cards por linha;
  - a partir de 1280px: trilho, 4 cards por linha, e o relógio aparece.
- **Alvos de toque** do shell com 44px ou mais.
- **Tema:**
  - chave `rotasaude:theme`, valores `govbr`, `rota` e `alto-contraste`;
  - o padrão é `govbr`, que é também o valor quando o armazenamento falha;
  - `<html lang="pt-BR" data-theme="govbr">` continua no `index.html`.
- **Acessibilidade:**
  - sem `outline: "none"`;
  - sem `color: "var(--ink4)"` em texto legível;
  - `accesskey` 1, 2 e 3 na barra eMAG.
- **Ícones:** só `@tabler/icons-react`, nunca símbolos unicode no shell novo.
- **Commits:** Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Git:**
  - use `/opt/homebrew/bin/git`;
  - adicione caminhos explícitos, nunca `git add -A`;
  - trabalhe em worktrees `ui/panel-shell` com `npm ci` rodado dentro do worktree.

## Review Focus

1. **Link do card aberto em nova aba (Ctrl/Cmd+clique ou clique do meio).** A pessoa espera a tela numa aba nova, com a aba atual intacta. O `RouteLink` só intercepta o clique primário sem modificador. A Task 5 testa isso.
2. **Voltar logo depois do login ou do retorno de assinatura.** O Voltar não pode voltar para o mesmo `#/` nem para o callback. A rota inicial entra com `replaceState` e não empilha histórico. A Task 3 testa isso.
3. **Link colado de uma tela que o papel não vê** (um profissional abrindo `#/team`). O esperado é o Início, sem erro e sem a tela aparecer de relance. A Task 3 testa o `replace`, e a Task 8 testa com o papel real.
4. **Botão "Ir para o conteúdo" da barra eMAG** com o roteador por hash. A pessoa espera o foco no conteúdo, não ser mandada ao Início. A barra impede a navegação do link (`preventDefault`) e só move o foco. A Task 6 testa isso.
5. **Armazenamento bloqueado** (navegador em modo privado, ou `localStorage` lançando erro). Trocar o tema precisa funcionar durante a sessão, sem quebrar a tela. A Task 2 testa isso.

---

## File Structure

**contracts** (worktree `contracts/.claude/worktrees/ui-shell`):
- `design-tokens/src/roles.json`: papel `on-tone`, pares e faces da Figtree.
- `design-tokens/src/themes/*.json`: `on-tone` por tema.
- `design-tokens/src/base.json`: sem mudança de cores.
- `design-tokens/dist/fonts/figtree-{400,600,700}.woff2` e `figtree-OFL.txt`.
- `design-tokens/dist/tokens.css` e `dist/tokens.ts`, gerados.
- `design-tokens/test/build.test.mjs`: teste de faces por família.
- `design-tokens/README.md` e `design-tokens/CHANGELOG.md`.

**Iguais byte a byte no dashboard e no admin** (cada um no seu worktree `.claude/worktrees/ui-shell`):

| Arquivo | Responsabilidade |
|---|---|
| `src/theme/theme.ts` + `theme.store.test.ts` | Tema por pessoa (`getTheme`, `setTheme`, `toggleHighContrast`, `useTheme`) |
| `src/shell/router.ts` + `router.test.ts` | `Route`, `slugOf`, `parseHash`, `toHash`, `useHashRoute` |
| `src/shell/types.ts` | `ShellItem`, `ShellGroup`, `Meta`, `Catalog` |
| `src/shell/RouteLink.tsx` | `<a href>` que chama `go` no clique primário |
| `src/shell/Cards.tsx` + `Cards.test.tsx` | `HomeCards` e `GroupCards` |
| `src/shell/Breadcrumb.tsx` + `Breadcrumb.test.tsx` | Trilha "Início › Grupo › Tela" com troca dentro do grupo |
| `src/shell/Rail.tsx` + `Rail.test.tsx` | Trilho de ícones |
| `src/shell/Drawer.tsx` + `Drawer.test.tsx` | Gaveta abaixo de 1024px |
| `src/shell/ScreenSearch.tsx` + `ScreenSearch.test.tsx` | Combobox de busca de tela |
| `src/shell/EmagBar.tsx` + `EmagBar.test.tsx` | Barra de acessibilidade eMAG |
| `src/shell/ContaMenu.tsx` + `ContaMenu.test.tsx` | Menu da conta (tema e Sair) |
| `src/shell/Brand.tsx` | Marca (extraída do `AppHeader`, com subtítulo por prop) |
| `src/shell/ContextStrip.tsx` | Faixa de contexto (`left` e `right`) |
| `src/shell/TopBar.tsx` | Barra de topo |
| `src/shell/A11yPage.tsx` | Página de acessibilidade (o texto de contato vem por prop) |
| `src/shell/Shell.tsx` + `Shell.test.tsx` | Composição, `<title>` e foco no `h1` |
| `src/shell/shell.css` | Larguras e grade dos cards |

**Específicos de cada app:**

| Arquivo | Responsabilidade |
|---|---|
| `src/shell/catalog.ts` + `catalog.test.ts` | Ícones e descrições (seção 4.3 da spec) |
| `src/shell/LiveClock.tsx` | Relógio extraído do `AppHeader`. O dashboard usa `cityOffsetLabel()`, o admin usa "BRT" |
| `src/shell/TenantChip.tsx` | Só no dashboard, extraído do `AppHeader` |
| `src/App.tsx` + `App.navigation.test.tsx` | Rota, grupos visíveis, conteúdo e atalhos entre telas |
| `index.html` + `src/theme/theme.inline.test.ts` | Script inline do tema |
| `src/shell/NotificationCenter.tsx` + `NotificationCenter.badge.test.ts` | Selo do sino com `--on-tone` |

**Saem:** `src/shell/AppHeader.tsx` e `src/shell/NavDropdown.tsx`, nos dois apps.

---

### Task 0: Preparação

**Files:** nenhum de produto.

- [ ] **Step 1: Avisar a sessão do módulo 19.** Mande à sessão `MVP Module 19` (por `SendMessage`) este texto:

  > Ciclo de interface, subprojeto 2 (shell dos painéis) começando em worktree `ui/panel-shell` no dashboard e no admin. Vão mudar `App.tsx` e `src/shell/` inteiro: saem `AppHeader` e `NavDropdown` e entra o shell novo, com roteamento por `#/`. As telas não mudam, e as props `onNavigate`/`onGoToSecurity` continuam iguais. Se a rodada especial (dashboard#12–14) for mexer no `App.tsx`, me avise para combinarmos a ordem do merge. Spec: `docs/superpowers/specs/2026-10-12-interface-panel-shell-design.md`.

- [ ] **Step 2: Criar os worktrees.** Antes, confira em cada repo que `origin/main..main` está vazio.

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude && G=/opt/homebrew/bin/git
$G -C contracts worktree add -q .claude/worktrees/ui-shell -b ui/panel-shell origin/main
$G -C apps/dashboard worktree add -q .claude/worktrees/ui-shell -b ui/panel-shell origin/main && (cd apps/dashboard/.claude/worktrees/ui-shell && npm ci --silent)
$G -C apps/admin worktree add -q .claude/worktrees/ui-shell -b ui/panel-shell origin/main && (cd apps/admin/.claude/worktrees/ui-shell && npm ci --silent)
```

- [ ] **Step 3: Medir a linha de base.** Em cada worktree de app, rode `npx vitest run`. O resultado esperado é o dashboard com 199 arquivos e 2194 testes, e o admin com 27 e 209, tudo verde. Anote os números. Se algo estiver vermelho, pare.

- [ ] **Step 4: Capturas "antes".**
  1. Copie o script `capture.mjs` do scratchpad do subprojeto 1 para o scratchpad atual.
  2. Acrescente a leitura de `WIDTHS=768,1024,1280` (variável de ambiente; o padrão é 1280) e faça um laço por largura, com o arquivo `<app>-<w>-<tela>.png`.
  3. Rode contra os containers de dev (dashboard em `curitiba.localhost:5175`, admin em `admin.localhost:5174`) nas três larguras, para as telas Visão geral, Atendimento, Minha agenda (grupo Atendimento, item "Minha agenda") e Equipe no dashboard, e Cidades no admin.
  4. Salve em `rota-saude/evidencias/2026-10-12-panel-shell/antes/`.

---

### Task 1: Figtree e `--on-tone` no contracts (`tokens-v1.1.0`)

**Files** (worktree do contracts):
- Modify: `design-tokens/src/roles.json`, `design-tokens/src/themes/{govbr,rota,alto-contraste,govbr-escuro}.json`, `design-tokens/test/build.test.mjs`, `design-tokens/README.md`, `design-tokens/CHANGELOG.md`
- Create: `design-tokens/dist/fonts/figtree-{400,600,700}.woff2`, `design-tokens/dist/fonts/figtree-OFL.txt`
- Regenerate: `design-tokens/dist/tokens.css`, `design-tokens/dist/tokens.ts`

**Interfaces:**
- Produces: as variáveis `--on-tone` (texto sobre um preenchimento de tom) e `@font-face "Figtree"` 400, 600 e 700, na tag `tokens-v1.1.0`.

- [ ] **Step 1: Escrever os testes que vão falhar.** Em `design-tokens/test/build.test.mjs`, acrescente:

```js
test("faces declaradas para Rawline e Figtree, cada uma com 400, 600 e 700", () => {
  const { fontFaces } = loadRoles();
  for (const family of ["Rawline", "Figtree"]) {
    const weights = fontFaces.filter((f) => f.family === family).map((f) => f.weight).sort();
    assert.deepEqual(weights, [400, 600, 700], family);
  }
  assert.ok(existsSync(join(root, "dist/fonts/figtree-OFL.txt")), "licença da Figtree junto da fonte");
});

test("on-tone é papel de cor e tem par de contraste sobre cada tom", () => {
  const { colors, pairs } = loadRoles();
  assert.ok(colors.includes("on-tone"));
  for (const tone of ["down", "warn", "info", "accent"]) {
    assert.ok(pairs.text.some(([fg, bg]) => fg === "on-tone" && bg === tone), tone);
  }
});
```

  Run: `node --test design-tokens/test/build.test.mjs`
  Expected: FAIL nos dois testes novos.

- [ ] **Step 2: Pedir permissão para o download.**
  - Fonte proposta: o pacote npm `@fontsource/figtree` (OFL 1.1, o `LICENSE` vem no pacote), baixado com `npm pack @fontsource/figtree` num diretório vazio do scratchpad.
  - Informe ao usuário o nome do pacote, a origem (registro npm) e o tamanho do `.tgz` (`npm view @fontsource/figtree dist.unpackedSize`).
  - **Espere o sim.**

- [ ] **Step 3: Colocar os arquivos.**
  1. Extraia o `.tgz` no scratchpad.
  2. Copie `package/files/figtree-latin-{400,600,700}-normal.woff2` para `design-tokens/dist/fonts/figtree-{400,600,700}.woff2`, e `package/LICENSE` para `design-tokens/dist/fonts/figtree-OFL.txt`.
  3. Confira cada fonte com `file`. A saída esperada contém `Web Open Font Format (Version 2)`.

- [ ] **Step 4: Declarar papel, pares e faces.**

  Em `roles.json`:
  - no array `colors`, acrescente `"on-tone"` depois de `"on-accent"`;
  - em `pairs.text`, acrescente `["on-tone", "down"], ["on-tone", "warn"], ["on-tone", "info"], ["on-tone", "accent"]`;
  - em `fontFaces`, acrescente:

```json
    { "family": "Figtree", "weight": 400, "file": "figtree-400.woff2" },
    { "family": "Figtree", "weight": 600, "file": "figtree-600.woff2" },
    { "family": "Figtree", "weight": 700, "file": "figtree-700.woff2" }
```

  Em cada tema, acrescente a chave `"on-tone"` depois de `"on-accent"`:

  | Tema | Valor de `on-tone` |
  |---|---|
  | `govbr.json` | `"white"` |
  | `rota.json` | `"white"` |
  | `alto-contraste.json` | `"black"` |
  | `govbr-escuro.json` | `"black"` |

- [ ] **Step 5: Gerar e testar.**

  Run: `node design-tokens/build.mjs && node --test design-tokens/test/*.test.mjs`
  Expected: PASS em todos, incluindo o contraste de `on-tone` sobre os quatro tons nos quatro temas. Se algum par falhar, troque o valor no tema e anote a decisão no ledger. Nunca afrouxe o teste.

- [ ] **Step 6: Documentar.**
  - **`README.md`:** na seção "Fonte", troque a frase da Figtree "ainda não é embarcada…" por `A Figtree (SIL OFL 1.1, de @fontsource/figtree) é a fonte do tema rota, em dist/fonts/ com figtree-OFL.txt.`. Na lista de papéis, acrescente `--on-tone` (texto sobre preenchimento de tom: selos de contagem).
  - **`CHANGELOG.md`:** acrescente no topo:

```markdown
## tokens-v1.1.0 — 2026-10-12

- Papel `--on-tone`: texto sobre preenchimento de `--down`/`--warn`/`--info`/`--accent` (selos), com contraste testado nos quatro temas.
- Fonte Figtree 400/600/700 (OFL 1.1) para o tema `rota`.
```

- [ ] **Step 7: Commit e tag local.**

```bash
/opt/homebrew/bin/git add design-tokens/src design-tokens/dist design-tokens/test/build.test.mjs design-tokens/README.md design-tokens/CHANGELOG.md
/opt/homebrew/bin/git commit -m "feat: add Figtree font and on-tone role (tokens-v1.1.0)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
/opt/homebrew/bin/git tag tokens-v1.1.0
```

---

### Task 2: Tokens v1.1.0, ícones e tema por pessoa (dashboard e admin)

Faça nos dois worktrees, com o mesmo código.

**Files:**
- Regenerate: `src/theme/vendor/**`, com `node scripts/tokens-sync.mjs tokens-v1.1.0 --contracts …`
- Modify: `package.json` e o lock, com `npm install @tabler/icons-react@^3`
- Create: `src/theme/theme.ts`, `src/theme/theme.store.test.ts`
- Modify: `index.html` (script inline)
- Create: `src/theme/theme.inline.test.ts`

**Interfaces:**
- Produces:
  - `THEME_KEY`
  - `PANEL_THEMES: readonly ["govbr","rota","alto-contraste"]`
  - `type PanelTheme`
  - `getTheme(): PanelTheme`
  - `setTheme(t: PanelTheme): void`
  - `toggleHighContrast(): void`
  - `useTheme(): PanelTheme`

- [ ] **Step 1: Sincronizar os tokens e instalar os ícones.**

```bash
GIT=/opt/homebrew/bin/git node scripts/tokens-sync.mjs tokens-v1.1.0 --contracts /Users/eduardovrocha/Development/ioit.solutions/rota-saude/contracts
npm install @tabler/icons-react@^3 --silent
npx vitest run src/theme/vendor.lock.test.ts
```

  Expected: o sync informa 10 arquivos, e o lock passa nos 3 testes.

- [ ] **Step 2: Escrever o teste do tema, que vai falhar.** `src/theme/theme.store.test.ts`:

```ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, renderHook } from "@testing-library/react";
import { THEME_KEY, getTheme, setTheme, toggleHighContrast, useTheme } from "./theme";

// setTheme("govbr") zera também a escolha em memória entre um teste e outro.
beforeEach(() => { setTheme("govbr"); localStorage.clear(); });
afterEach(() => { vi.restoreAllMocks(); });

describe("tema por pessoa", () => {
  it("sem nada salvo vale govbr", () => {
    expect(getTheme()).toBe("govbr");
  });

  it("valor inválido salvo vale govbr", () => {
    localStorage.setItem(THEME_KEY, "neon");
    expect(getTheme()).toBe("govbr");
  });

  it("setTheme aplica no <html> e salva", () => {
    setTheme("rota");
    expect(document.documentElement.dataset.theme).toBe("rota");
    expect(localStorage.getItem(THEME_KEY)).toBe("rota");
    expect(getTheme()).toBe("rota");
  });

  it("armazenamento quebrado: o tema troca na sessão e nada lança", () => {
    vi.spyOn(Storage.prototype, "getItem").mockImplementation(() => { throw new Error("blocked"); });
    vi.spyOn(Storage.prototype, "setItem").mockImplementation(() => { throw new Error("blocked"); });
    expect(getTheme()).toBe("govbr");
    expect(() => setTheme("alto-contraste")).not.toThrow();
    expect(document.documentElement.dataset.theme).toBe("alto-contraste");
    expect(getTheme()).toBe("alto-contraste");
  });

  it("alto contraste liga e volta ao tema anterior", () => {
    setTheme("rota");
    toggleHighContrast();
    expect(getTheme()).toBe("alto-contraste");
    toggleHighContrast();
    expect(getTheme()).toBe("rota");
  });

  it("useTheme acompanha a troca", () => {
    const { result } = renderHook(() => useTheme());
    expect(result.current).toBe("govbr");
    act(() => setTheme("rota"));
    expect(result.current).toBe("rota");
  });
});
```

  Run: `npx vitest run src/theme/theme.store.test.ts`
  Expected: FAIL, porque o módulo `./theme` não existe.

- [ ] **Step 3: Implementar `src/theme/theme.ts`.**

```ts
// Tema por pessoa (spec do subprojeto 2, §5.1). Salvo no navegador; se o
// armazenamento falhar (modo privado, bloqueio), vale a escolha em memória
// durante a sessão e, sem escolha, o govbr.
import { useSyncExternalStore } from "react";

export const THEME_KEY = "rotasaude:theme";
const PREV_KEY = "rotasaude:theme:prev";
export const PANEL_THEMES = [ "govbr", "rota", "alto-contraste" ] as const;
export type PanelTheme = (typeof PANEL_THEMES)[number];

const listeners = new Set<() => void>();
let memory: PanelTheme | null = null;

const isTheme = (v: unknown): v is PanelTheme =>
  typeof v === "string" && (PANEL_THEMES as readonly string[]).includes(v);

function read(key: string): string | null {
  try { return window.localStorage.getItem(key); } catch { return null; }
}

function write(key: string, value: string): void {
  try { window.localStorage.setItem(key, value); } catch { /* sessão só em memória */ }
}

export function getTheme(): PanelTheme {
  const saved = read(THEME_KEY);
  if (isTheme(saved)) return saved;
  return memory ?? "govbr";
}

export function setTheme(theme: PanelTheme): void {
  memory = theme;
  document.documentElement.dataset.theme = theme;
  write(THEME_KEY, theme);
  listeners.forEach((l) => l());
}

export function toggleHighContrast(): void {
  const current = getTheme();
  if (current === "alto-contraste") {
    const prev = read(PREV_KEY);
    setTheme(isTheme(prev) && prev !== "alto-contraste" ? prev : "govbr");
  } else {
    write(PREV_KEY, current);
    setTheme("alto-contraste");
  }
}

function subscribe(onChange: () => void): () => void {
  listeners.add(onChange);
  return () => { listeners.delete(onChange); };
}

export function useTheme(): PanelTheme {
  return useSyncExternalStore(subscribe, getTheme);
}
```

  Um detalhe do teste de armazenamento quebrado: o `toggleHighContrast` lê o anterior com `read`, que devolve `null`. Por isso, sem armazenamento, sair do alto contraste volta ao `govbr`. Isso é aceitável e coberto pela regra "sem escolha, govbr".

  Run: `npx vitest run src/theme/theme.store.test.ts`
  Expected: PASS nos 6 testes.

- [ ] **Step 4: Escrever o teste do script inline, que vai falhar.** `src/theme/theme.inline.test.ts`:

```ts
// O index.html aplica o tema salvo antes do React montar (sem flash).
import { beforeEach, describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const html = readFileSync(join(dirname(fileURLToPath(import.meta.url)), "../../index.html"), "utf8");
const inline = /<script data-theme-boot>([\s\S]*?)<\/script>/.exec(html)?.[1];

beforeEach(() => { localStorage.clear(); document.documentElement.dataset.theme = "govbr"; });

describe("script inline do tema", () => {
  it("existe no <head>, antes do módulo da aplicação", () => {
    expect(inline).toBeTruthy();
    expect(html.indexOf("data-theme-boot")).toBeLessThan(html.indexOf('type="module"'));
  });

  it("aplica o tema salvo", () => {
    localStorage.setItem("rotasaude:theme", "alto-contraste");
    new Function(inline!)();
    expect(document.documentElement.dataset.theme).toBe("alto-contraste");
  });

  it("ignora valor inválido", () => {
    localStorage.setItem("rotasaude:theme", "<script>");
    new Function(inline!)();
    expect(document.documentElement.dataset.theme).toBe("govbr");
  });
});
```

  Run: `npx vitest run src/theme/theme.inline.test.ts`
  Expected: FAIL, com `expected undefined to be truthy`.

- [ ] **Step 5: Pôr o script no `index.html`.** Coloque-o dentro do `<head>`, logo depois de `<meta name="viewport" …>`:

```html
    <script data-theme-boot>try{var t=localStorage.getItem("rotasaude:theme");if(t==="govbr"||t==="rota"||t==="alto-contraste")document.documentElement.setAttribute("data-theme",t)}catch(e){}</script>
```

  O `nginx.conf` dos dois apps não declara Content-Security-Policy (conferido no plano), então o script inline roda em produção.

  Run: `npx vitest run src/theme/`
  Expected: PASS em `theme.store`, `theme.inline`, `theme.test` (o `data-theme="govbr"` continua), `vendor.lock`, `literals` (só no dashboard) e `a11y-guards`.

- [ ] **Step 6: Commit** (em cada app).

```bash
/opt/homebrew/bin/git add src/theme package.json package-lock.json index.html
/opt/homebrew/bin/git commit -m "feat: add per-person theme store and tokens-v1.1.0 with Tabler icons

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Roteador por hash (dashboard e admin)

Mesmo código nos dois apps.

**Files:**
- Create: `src/shell/router.ts`, `src/shell/router.test.ts`

**Interfaces:**
- Produces:
  - `type Route<Id extends string = string> = { kind: "home" } | { kind: "group"; slug: string } | { kind: "screen"; id: Id } | { kind: "a11y" }`
  - `slugOf(label: string): string`
  - `parseHash<Id extends string>(hash: string, ids: ReadonlySet<string>, slugs: ReadonlySet<string>): Route<Id> | null`
  - `toHash(route: Route): string`
  - `useHashRoute<Id extends string>(opts: { ids: ReadonlySet<string>; slugs: ReadonlySet<string>; initial?: Route<Id>; isAllowed: (r: Route<Id>) => boolean }): { route: Route<Id>; go: (r: Route<Id>) => void }`

- [ ] **Step 1: Escrever o teste, que vai falhar.** `src/shell/router.test.ts`:

```ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, renderHook } from "@testing-library/react";
import { parseHash, slugOf, toHash, useHashRoute, type Route } from "./router";

const IDS = new Set([ "overview", "team", "my-agenda" ]);
const SLUGS = new Set([ "visao-geral", "atendimento", "e-sus" ]);

function setHash(hash: string) {
  window.history.replaceState(null, "", `/dashboard/${hash}`);
}

beforeEach(() => setHash(""));
afterEach(() => vi.restoreAllMocks());

describe("slugOf", () => {
  it("tira acento, põe minúsculas e troca espaço por hífen", () => {
    expect(slugOf("Visão geral")).toBe("visao-geral");
    expect(slugOf("e-SUS")).toBe("e-sus");
    expect(slugOf("Comunicação")).toBe("comunicacao");
  });
});

describe("parseHash e toHash", () => {
  const cases: [string, Route][] = [
    [ "#/", { kind: "home" } ],
    [ "#/acessibilidade", { kind: "a11y" } ],
    [ "#/grupo/atendimento", { kind: "group", slug: "atendimento" } ],
    [ "#/my-agenda", { kind: "screen", id: "my-agenda" } ]
  ];
  it.each(cases)("%s ida e volta", (hash, route) => {
    expect(parseHash(hash, IDS, SLUGS)).toEqual(route);
    expect(toHash(route)).toBe(hash);
  });

  it("vazio é o Início", () => {
    expect(parseHash("", IDS, SLUGS)).toEqual({ kind: "home" });
  });

  it("slug ou id desconhecido, ou lixo, dá null", () => {
    expect(parseHash("#/grupo/nada", IDS, SLUGS)).toBeNull();
    expect(parseHash("#/nada", IDS, SLUGS)).toBeNull();
    expect(parseHash("#conteudo", IDS, SLUGS)).toBeNull();
  });
});

describe("useHashRoute", () => {
  const allowAll = () => true;

  it("go empilha no histórico e muda o hash", () => {
    const push = vi.spyOn(window.history, "pushState");
    const { result } = renderHook(() => useHashRoute({ ids: IDS, slugs: SLUGS, isAllowed: allowAll }));
    act(() => result.current.go({ kind: "screen", id: "team" }));
    expect(window.location.hash).toBe("#/team");
    expect(result.current.route).toEqual({ kind: "screen", id: "team" });
    expect(push).toHaveBeenCalledTimes(1);
  });

  it("Voltar (popstate) relê o hash", () => {
    const { result } = renderHook(() => useHashRoute({ ids: IDS, slugs: SLUGS, isAllowed: allowAll }));
    act(() => { setHash("#/grupo/atendimento"); window.dispatchEvent(new PopStateEvent("popstate")); });
    expect(result.current.route).toEqual({ kind: "group", slug: "atendimento" });
  });

  it("rota não permitida vira Início com replaceState, sem histórico novo", () => {
    setHash("#/team");
    const push = vi.spyOn(window.history, "pushState");
    const replace = vi.spyOn(window.history, "replaceState");
    const isAllowed = (r: Route) => !(r.kind === "screen" && r.id === "team");
    const { result } = renderHook(() => useHashRoute({ ids: IDS, slugs: SLUGS, isAllowed }));
    expect(result.current.route).toEqual({ kind: "home" });
    expect(window.location.hash).toBe("#/");
    expect(replace).toHaveBeenCalled();
    expect(push).not.toHaveBeenCalled();
  });

  it("hash inválido vira Início", () => {
    setHash("#/xyz");
    const { result } = renderHook(() => useHashRoute({ ids: IDS, slugs: SLUGS, isAllowed: allowAll }));
    expect(result.current.route).toEqual({ kind: "home" });
  });

  it("rota inicial (retorno de assinatura) entra sem empilhar", () => {
    const push = vi.spyOn(window.history, "pushState");
    const { result } = renderHook(() =>
      useHashRoute({ ids: IDS, slugs: SLUGS, isAllowed: allowAll, initial: { kind: "screen", id: "my-agenda" } }));
    expect(result.current.route).toEqual({ kind: "screen", id: "my-agenda" });
    expect(window.location.hash).toBe("#/my-agenda");
    expect(push).not.toHaveBeenCalled();
  });

  it("atalho interno não é filtrado", () => {
    const isAllowed = (r: Route) => r.kind !== "screen";
    const { result } = renderHook(() => useHashRoute({ ids: IDS, slugs: SLUGS, isAllowed }));
    act(() => result.current.go({ kind: "screen", id: "team" }));
    expect(result.current.route).toEqual({ kind: "screen", id: "team" });
  });

  it("preserva a query (?bairro=) ao navegar", () => {
    window.history.replaceState(null, "", "/dashboard/?bairro=none#/");
    const { result } = renderHook(() => useHashRoute({ ids: IDS, slugs: SLUGS, isAllowed: allowAll }));
    act(() => result.current.go({ kind: "screen", id: "overview" }));
    expect(window.location.search).toBe("?bairro=none");
  });
});
```

  Run: `npx vitest run src/shell/router.test.ts`
  Expected: FAIL, porque o módulo `./router` não existe.

- [ ] **Step 2: Implementar `src/shell/router.ts`.**

```ts
// Roteador por hash do shell (spec do subprojeto 2, §3). #/ Início,
// #/grupo/<slug>, #/<id da tela>, #/acessibilidade. O Voltar sobe um nível
// porque cada `go` empilha um hash; rota inválida ou não permitida vira o
// Início com replaceState (não entra no histórico). Atalho interno (`go`)
// não passa pelo filtro: abre o que abre hoje.
import { useCallback, useEffect, useRef, useState } from "react";

export type Route<Id extends string = string> =
  | { kind: "home" }
  | { kind: "group"; slug: string }
  | { kind: "screen"; id: Id }
  | { kind: "a11y" };

export function slugOf(label: string): string {
  return label.normalize("NFD").replace(/[̀-ͯ]/g, "").toLowerCase().trim().replace(/\s+/g, "-");
}

export function parseHash<Id extends string>(
  hash: string, ids: ReadonlySet<string>, slugs: ReadonlySet<string>
): Route<Id> | null {
  const path = hash.replace(/^#/, "");
  if (path === "" || path === "/") return { kind: "home" };
  if (path === "/acessibilidade") return { kind: "a11y" };
  const group = /^\/grupo\/([a-z0-9-]+)$/.exec(path);
  if (group) return slugs.has(group[1]) ? { kind: "group", slug: group[1] } : null;
  const screen = /^\/([a-z0-9_-]+)$/.exec(path);
  if (screen && ids.has(screen[1])) return { kind: "screen", id: screen[1] as Id };
  return null;
}

export function toHash(route: Route): string {
  switch (route.kind) {
    case "home":   return "#/";
    case "a11y":   return "#/acessibilidade";
    case "group":  return `#/grupo/${route.slug}`;
    case "screen": return `#/${route.id}`;
  }
}

function writeHash(route: Route, mode: "push" | "replace"): void {
  const url = new URL(window.location.href);
  url.hash = toHash(route);
  if (mode === "push") window.history.pushState(window.history.state, "", url.toString());
  else window.history.replaceState(window.history.state, "", url.toString());
}

export function useHashRoute<Id extends string>(opts: {
  ids: ReadonlySet<string>;
  slugs: ReadonlySet<string>;
  initial?: Route<Id>;
  isAllowed: (r: Route<Id>) => boolean;
}): { route: Route<Id>; go: (r: Route<Id>) => void } {
  const optsRef = useRef(opts);
  optsRef.current = opts;

  const resolve = useCallback((): Route<Id> => {
    const { ids, slugs, isAllowed } = optsRef.current;
    const parsed = parseHash<Id>(window.location.hash, ids, slugs);
    if (parsed && isAllowed(parsed)) return parsed;
    const home: Route<Id> = { kind: "home" };
    writeHash(home, "replace");
    return home;
  }, []);

  const [ route, setRoute ] = useState<Route<Id>>(() => {
    const empty = window.location.hash === "" || window.location.hash === "#";
    if (empty && opts.initial) {
      writeHash(opts.initial, "replace");
      return opts.initial;
    }
    return resolve();
  });

  useEffect(() => {
    const onNav = () => setRoute(resolve());
    window.addEventListener("popstate", onNav);
    window.addEventListener("hashchange", onNav);
    return () => {
      window.removeEventListener("popstate", onNav);
      window.removeEventListener("hashchange", onNav);
    };
  }, [ resolve ]);

  const go = useCallback((r: Route<Id>) => {
    writeHash(r, "push");
    setRoute(r);
  }, []);

  return { route, go };
}
```

  O teste "rota não permitida" exige que, ao montar com `#/team` não permitido, o `resolve` faça `replaceState` para `#/`. O `writeHash(home, "replace")` cuida disso.

  Run: `npx vitest run src/shell/router.test.ts`
  Expected: PASS em todos.

- [ ] **Step 3: Commit** (em cada app).

```bash
/opt/homebrew/bin/git add src/shell/router.ts src/shell/router.test.ts
/opt/homebrew/bin/git commit -m "feat: add hash router for the panel shell

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Tipos do shell e catálogo de ícones e descrições

**Files:**
- Create (iguais nos dois apps): `src/shell/types.ts`
- Create (por app): `src/shell/catalog.ts`, `src/shell/catalog.test.ts`

**Interfaces:**
- Produces (`types.ts`):

```ts
// Tipos do shell, iguais no dashboard e no admin.
import type { Icon } from "@tabler/icons-react";

export interface ShellItem { id: string; label: string; count?: number }
export interface ShellGroup { label: string; slug: string; items: ShellItem[] }
export interface Meta { icon: Icon; description: string }
export interface Catalog { groups: Record<string, Meta>; screens: Record<string, Meta> }
```

- Produces (`catalog.ts`): `export const CATALOG: Catalog`, com as chaves `groups` pelo **rótulo** do grupo e `screens` pelo **id**.

- [ ] **Step 1: Criar o `types.ts`** com o conteúdo acima, nos dois apps.

- [ ] **Step 2: Escrever o teste do catálogo, que vai falhar.** `src/shell/catalog.test.ts`, igual nos dois apps:

```ts
// Todo grupo e toda tela do menu têm ícone e descrição no catálogo.
import { describe, expect, it } from "vitest";
import { NAV_GROUPS } from "./modules";
import { CATALOG } from "./catalog";

describe("catálogo do shell", () => {
  it.each(NAV_GROUPS.map((g) => g.label))("grupo %s tem ícone e descrição", (label) => {
    expect(CATALOG.groups[label]?.description).toBeTruthy();
    expect(CATALOG.groups[label]?.icon).toBeTruthy();
  });

  it.each(NAV_GROUPS.flatMap((g) => g.items.map((i) => i.id)))("tela %s tem ícone e descrição", (id) => {
    expect(CATALOG.screens[id]?.description).toBeTruthy();
    expect(CATALOG.screens[id]?.icon).toBeTruthy();
  });
});
```

  Run: `npx vitest run src/shell/catalog.test.ts`
  Expected: FAIL, porque o módulo `./catalog` não existe.

- [ ] **Step 3: `src/shell/catalog.ts` do dashboard.**

```ts
// Ícones e descrições dos cards (spec do subprojeto 2, §4.3). Rótulos vêm
// de NAV_GROUPS; aqui só o que o card mostra além do rótulo.
import {
  IconArrowsExchange, IconBuildingCommunity, IconBuildingHospital, IconCalendar, IconCertificate,
  IconChartLine, IconChartPie, IconClipboardList, IconDownload, IconFileCertificate, IconFileDescription,
  IconFilePencil, IconFileText, IconHeartbeat, IconHistory, IconIdBadge2, IconInbox, IconLayoutDashboard,
  IconListCheck, IconLock, IconMapPin, IconMessages, IconNotes, IconPencil, IconPill, IconPlugConnected,
  IconServerCog, IconShieldCheck, IconSignature, IconSpeakerphone, IconStack2, IconStethoscope, IconUpload,
  IconUser, IconUserCircle, IconUsers, IconUsersGroup
} from "@tabler/icons-react";
import type { Catalog } from "./types";

export const CATALOG: Catalog = {
  groups: {
    "Visão geral": { icon: IconLayoutDashboard, description: "Indicadores do dia na cidade" },
    "Aquisição":   { icon: IconInbox, description: "Conversas e consentimentos" },
    "Triagem":     { icon: IconListCheck, description: "Triagens, classificação e relatórios" },
    "Análise":     { icon: IconChartLine, description: "Analytics da cidade" },
    "Governança":  { icon: IconFileCertificate, description: "Protocolos e auditoria" },
    "Operação":    { icon: IconServerCog, description: "Filas e saúde do sistema" },
    "Atendimento": { icon: IconStethoscope, description: "Agenda, prontuário e consultas" },
    "Comunicação": { icon: IconSpeakerphone, description: "Campanhas por SMS" },
    "Equipe":      { icon: IconUsers, description: "Pessoas, papéis e assinaturas" },
    "Cidade":      { icon: IconBuildingCommunity, description: "Território e documentos" },
    "e-SUS":       { icon: IconArrowsExchange, description: "Integrações, CNES e produção" },
    "Conta":       { icon: IconUserCircle, description: "Segurança, perfil e assinatura" }
  },
  screens: {
    "overview": { icon: IconLayoutDashboard, description: "Triagens, filas e alertas de hoje" },
    "ingestion": { icon: IconDownload, description: "Volume de mensagens recebidas" },
    "conversations": { icon: IconMessages, description: "Conversas do canal do cidadão" },
    "consent": { icon: IconShieldCheck, description: "Aceites e revogações do termo" },
    "triages": { icon: IconListCheck, description: "Triagens concluídas e em andamento" },
    "classification": { icon: IconChartPie, description: "Distribuição por risco e prioridade" },
    "reports": { icon: IconFileText, description: "Relatórios entregues ao cidadão" },
    "analytics": { icon: IconChartLine, description: "Tendências por semana e bairro" },
    "protocols": { icon: IconFileCertificate, description: "Versões, assinaturas e ativação" },
    "protocol-editor": { icon: IconPencil, description: "Escrever e revisar rascunhos" },
    "events": { icon: IconHistory, description: "Quem fez o quê e quando" },
    "queues": { icon: IconStack2, description: "Filas de processamento e falhas" },
    "health": { icon: IconHeartbeat, description: "Frescor dos dados e alertas do sistema" },
    "attendance": { icon: IconStethoscope, description: "Fila de chamada e desfecho" },
    "my-agenda": { icon: IconCalendar, description: "Seus horários de hoje e da semana" },
    "signature-pending": { icon: IconSignature, description: "Documentos esperando sua assinatura" },
    "clinical-record": { icon: IconNotes, description: "Histórico clínico do cidadão" },
    "my-consultations": { icon: IconClipboardList, description: "Consultas que você registrou" },
    "professional-consultations": { icon: IconUsersGroup, description: "Consultas da equipe por pessoa" },
    "open-drafts": { icon: IconFilePencil, description: "Consultas iniciadas e não concluídas" },
    "campaigns": { icon: IconSpeakerphone, description: "Criar e enviar campanhas por SMS" },
    "team": { icon: IconUsers, description: "Usuários, convites e papéis" },
    "professionals": { icon: IconIdBadge2, description: "Vínculos, CBO e turnos" },
    "signature-overview": { icon: IconCertificate, description: "Certificados digitais da equipe" },
    "sncr-overview": { icon: IconPill, description: "Numeração de receita controlada da equipe" },
    "territory": { icon: IconMapPin, description: "Bairros e unidades de referência" },
    "clinical-documents-admin": { icon: IconFileDescription, description: "Modelos de atestado e receita" },
    "integrations": { icon: IconPlugConnected, description: "Conexão com o PEC e-SUS" },
    "cnes": { icon: IconBuildingHospital, description: "Estabelecimentos e profissionais do CNES" },
    "production": { icon: IconUpload, description: "Envio da produção ao e-SUS" },
    "security": { icon: IconLock, description: "Senha e verificação em duas etapas" },
    "my-profile": { icon: IconUser, description: "Seus dados profissionais" },
    "signature": { icon: IconCertificate, description: "Seu certificado para assinar" },
    "sncr": { icon: IconPill, description: "Sua numeração de receita controlada" }
  }
};
```

- [ ] **Step 4: `src/shell/catalog.ts` do admin.**

```ts
// Ícones e descrições dos cards do console (spec do subprojeto 2, §4.3).
import {
  IconArrowsExchange, IconBuildingCommunity, IconChartDots, IconPhoneOff, IconPhonePlus, IconPlus,
  IconSettings, IconUpload
} from "@tabler/icons-react";
import type { Catalog } from "./types";

export const CATALOG: Catalog = {
  groups: {
    "Setup":     { icon: IconSettings, description: "Cidades, canais e provisionamento" },
    "Analytics": { icon: IconChartDots, description: "Indicadores de todas as cidades" },
    "e-SUS":     { icon: IconArrowsExchange, description: "Produção enviada pelas cidades" }
  },
  screens: {
    "cities": { icon: IconBuildingCommunity, description: "Catálogo das cidades da plataforma" },
    "provision_city": { icon: IconPlus, description: "Criar uma cidade nova" },
    "register_channel": { icon: IconPhonePlus, description: "Ligar um número de canal a uma cidade" },
    "unknown_channels": { icon: IconPhoneOff, description: "Mensagens de números sem cidade" },
    "city_analytics": { icon: IconChartDots, description: "Indicadores de todas as cidades" },
    "city_production": { icon: IconUpload, description: "Produção enviada pelas cidades" }
  }
};
```

- [ ] **Step 5: Rodar o teste e conferir os tipos.**

  Run: `npx vitest run src/shell/catalog.test.ts && npx tsc --noEmit -p .`
  Expected: PASS, e o `tsc` sem erro. Se algum nome de ícone não existir na versão instalada, o `tsc` acusa. Troque pelo nome mais próximo listado em `node_modules/@tabler/icons-react/dist/esm/icons/` e anote a decisão no ledger.

- [ ] **Step 6: Commit** (em cada app).

```bash
/opt/homebrew/bin/git add src/shell/types.ts src/shell/catalog.ts src/shell/catalog.test.ts
/opt/homebrew/bin/git commit -m "feat: add shell types and card catalog with Tabler icons

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Peças de navegação (`RouteLink`, `Cards`, `Breadcrumb`, `Rail`, `Drawer`, `shell.css`)

Mesmo código nos dois apps.

**Files:**
- Create: `src/shell/RouteLink.tsx`, `src/shell/Cards.tsx`, `src/shell/Breadcrumb.tsx`, `src/shell/Rail.tsx`, `src/shell/Drawer.tsx`, `src/shell/shell.css`
- Create (testes): `src/shell/Cards.test.tsx`, `src/shell/Breadcrumb.test.tsx`, `src/shell/Rail.test.tsx`, `src/shell/Drawer.test.tsx`

**Interfaces:**
- Consumes: `Route` e `toHash` (Task 3); `ShellGroup` e `Catalog` (Task 4).
- Produces:
  - `RouteLink({ route, go, className?, style?, children, ...aria })`
  - `routeForGroup(g: ShellGroup): Route`, que dá a tela direto num grupo de 1 item e `{kind:"group"}` nos demais
  - `HomeCards({ groups, catalog, go, now? })`
  - `GroupCards({ group, catalog, go })`
  - `Breadcrumb({ groups, route, catalog, go, titleFor })`
  - `Rail({ groups, catalog, route, go })`
  - `Drawer({ open, onClose, groups, route, go, returnFocusTo })`

- [ ] **Step 1: Escrever os testes, que vão falhar.**

`src/shell/Cards.test.tsx`:
```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { IconUsers } from "@tabler/icons-react";
import { GroupCards, HomeCards } from "./Cards";
import type { Catalog, ShellGroup } from "./types";

afterEach(cleanup);

const groups: ShellGroup[] = [
  { label: "Visão geral", slug: "visao-geral", items: [ { id: "overview", label: "Visão geral" } ] },
  { label: "Atendimento", slug: "atendimento", items: [
    { id: "attendance", label: "Atendimento" },
    { id: "signature-pending", label: "Pendentes de assinatura", count: 3 }
  ] }
];
const meta = { icon: IconUsers, description: "descrição" };
const catalog: Catalog = {
  groups: { "Visão geral": meta, "Atendimento": meta },
  screens: { overview: meta, attendance: meta, "signature-pending": meta }
};

describe("HomeCards", () => {
  it("um card por grupo; grupo de uma tela vai direto à tela", () => {
    render(<HomeCards groups={groups} catalog={catalog} go={vi.fn()} now={new Date(2026, 9, 12, 9)} />);
    expect(screen.getByRole("heading", { level: 1 }).textContent).toBe("Bom dia");
    expect(screen.getByRole("link", { name: /Visão geral/ }).getAttribute("href")).toBe("#/overview");
    expect(screen.getByRole("link", { name: /Atendimento/ }).getAttribute("href")).toBe("#/grupo/atendimento");
  });

  it("clique primário chama go; Ctrl+clique deixa o navegador abrir em nova aba", () => {
    const go = vi.fn();
    render(<HomeCards groups={groups} catalog={catalog} go={go} />);
    const link = screen.getByRole("link", { name: /Atendimento/ });
    fireEvent.click(link, { ctrlKey: true });
    expect(go).not.toHaveBeenCalled();
    fireEvent.click(link);
    expect(go).toHaveBeenCalledWith({ kind: "group", slug: "atendimento" });
  });
});

describe("GroupCards", () => {
  it("um sub-card por tela, com selo de contagem", () => {
    render(<GroupCards group={groups[1]} catalog={catalog} go={vi.fn()} />);
    expect(screen.getByRole("heading", { level: 1 }).textContent).toBe("Atendimento");
    const pending = screen.getByRole("link", { name: /Pendentes de assinatura/ });
    expect(pending.getAttribute("href")).toBe("#/signature-pending");
    expect(pending.textContent).toContain("3");
  });
});
```

`src/shell/Breadcrumb.test.tsx`:
```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { Breadcrumb } from "./Breadcrumb";
import type { ShellGroup } from "./types";

afterEach(cleanup);

const groups: ShellGroup[] = [
  { label: "Visão geral", slug: "visao-geral", items: [ { id: "overview", label: "Visão geral" } ] },
  { label: "Atendimento", slug: "atendimento", items: [
    { id: "attendance", label: "Atendimento" }, { id: "my-agenda", label: "Minha agenda" } ] }
];
const titleFor = (id: string) => groups.flatMap((g) => g.items).find((i) => i.id === id)?.label ?? id;

describe("Breadcrumb", () => {
  it("Início › Grupo › Tela, e troca de tela no grupo", () => {
    const go = vi.fn();
    render(<Breadcrumb groups={groups} route={{ kind: "screen", id: "my-agenda" }} go={go} titleFor={titleFor} />);
    const nav = screen.getByRole("navigation", { name: "Trilha" });
    expect(nav.textContent).toContain("Início");
    expect(screen.getByRole("link", { name: "Atendimento" }).getAttribute("href")).toBe("#/grupo/atendimento");
    fireEvent.click(screen.getByRole("button", { name: /Minha agenda/ }));
    fireEvent.click(screen.getByRole("menuitem", { name: "Atendimento" }));
    expect(go).toHaveBeenCalledWith({ kind: "screen", id: "attendance" });
  });

  it("grupo de uma tela: Início › Tela", () => {
    render(<Breadcrumb groups={groups} route={{ kind: "screen", id: "overview" }} go={vi.fn()} titleFor={titleFor} />);
    expect(screen.queryByRole("link", { name: "Visão geral" })).toBeNull();
    expect(screen.getByRole("navigation", { name: "Trilha" }).textContent).toContain("Visão geral");
  });

  it("tela fora do menu: Início › Tela", () => {
    render(<Breadcrumb groups={groups} route={{ kind: "screen", id: "security" }} go={vi.fn()} titleFor={() => "Segurança"} />);
    expect(screen.getByRole("navigation", { name: "Trilha" }).textContent).toBe("Início›Segurança");
  });
});
```

`src/shell/Rail.test.tsx`:
```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen } from "@testing-library/react";
import { IconUsers } from "@tabler/icons-react";
import { Rail } from "./Rail";
import type { Catalog, ShellGroup } from "./types";

afterEach(cleanup);

const groups: ShellGroup[] = [
  { label: "Visão geral", slug: "visao-geral", items: [ { id: "overview", label: "Visão geral" } ] },
  { label: "Atendimento", slug: "atendimento", items: [
    { id: "attendance", label: "Atendimento" }, { id: "my-agenda", label: "Minha agenda" } ] }
];
const meta = { icon: IconUsers, description: "d" };
const catalog: Catalog = { groups: { "Visão geral": meta, "Atendimento": meta }, screens: {} };

describe("Rail", () => {
  it("Início e um link por grupo, marcando o grupo da tela atual", () => {
    render(<Rail groups={groups} catalog={catalog} route={{ kind: "screen", id: "my-agenda" }} go={vi.fn()} />);
    const nav = screen.getByRole("navigation", { name: "Grupos" });
    expect(nav.className).toContain("shell-rail");
    expect(screen.getByRole("link", { name: "Início" }).getAttribute("href")).toBe("#/");
    expect(screen.getByRole("link", { name: "Atendimento" }).getAttribute("aria-current")).toBe("page");
    expect(screen.getByRole("link", { name: "Visão geral" }).getAttribute("href")).toBe("#/overview");
  });
});
```

`src/shell/Drawer.test.tsx`:
```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { Drawer } from "./Drawer";
import type { ShellGroup } from "./types";

afterEach(cleanup);

const groups: ShellGroup[] = [
  { label: "Atendimento", slug: "atendimento", items: [
    { id: "attendance", label: "Atendimento" }, { id: "my-agenda", label: "Minha agenda" } ] }
];

function setup(onClose = vi.fn(), go = vi.fn()) {
  const opener = document.createElement("button");
  document.body.appendChild(opener);
  render(<Drawer open onClose={onClose} groups={groups} route={{ kind: "home" }} go={go} returnFocusTo={opener} />);
  return { onClose, go, opener };
}

describe("Drawer", () => {
  it("é um diálogo modal com o menu completo e foco dentro", () => {
    setup();
    const dialog = screen.getByRole("dialog", { name: "Menu" });
    expect(dialog.getAttribute("aria-modal")).toBe("true");
    expect(dialog.contains(document.activeElement)).toBe(true);
    expect(screen.getByRole("link", { name: "Minha agenda" }).getAttribute("href")).toBe("#/my-agenda");
  });

  it("Esc fecha", () => {
    const { onClose } = setup();
    fireEvent.keyDown(document, { key: "Escape" });
    expect(onClose).toHaveBeenCalled();
  });

  it("clique no fundo fecha", () => {
    const { onClose } = setup();
    fireEvent.mouseDown(screen.getByTestId("drawer-backdrop"));
    expect(onClose).toHaveBeenCalled();
  });

  it("escolher uma tela navega e fecha", () => {
    const { onClose, go } = setup();
    fireEvent.click(screen.getByRole("link", { name: "Minha agenda" }));
    expect(go).toHaveBeenCalledWith({ kind: "screen", id: "my-agenda" });
    expect(onClose).toHaveBeenCalled();
  });

  it("Tab no último volta ao primeiro (foco preso)", () => {
    setup();
    const focusables = screen.getByRole("dialog").querySelectorAll<HTMLElement>("a,button");
    focusables[focusables.length - 1].focus();
    fireEvent.keyDown(document, { key: "Tab" });
    expect(document.activeElement).toBe(focusables[0]);
  });
});
```

  Run: `npx vitest run src/shell/Cards.test.tsx src/shell/Breadcrumb.test.tsx src/shell/Rail.test.tsx src/shell/Drawer.test.tsx`
  Expected: FAIL, porque os módulos não existem.

- [ ] **Step 2: `src/shell/RouteLink.tsx`.**

```tsx
// Link de rota: <a href="#/…"> de verdade (nova aba, copiar link), mas o
// clique primário sem modificador navega pelo `go` (empilha histórico).
import type { AnchorHTMLAttributes, MouseEvent, ReactNode } from "react";
import { toHash, type Route } from "./router";
import type { ShellGroup } from "./types";

type Props = Omit<AnchorHTMLAttributes<HTMLAnchorElement>, "href"> & {
  route: Route; go: (r: Route) => void; children: ReactNode; afterGo?: () => void;
};

export function RouteLink({ route, go, children, onClick, afterGo, ...rest }: Props) {
  function handle(e: MouseEvent<HTMLAnchorElement>) {
    onClick?.(e);
    if (e.defaultPrevented || e.button !== 0 || e.metaKey || e.ctrlKey || e.shiftKey || e.altKey) return;
    e.preventDefault();
    go(route);
    afterGo?.();
  }
  return <a href={toHash(route)} onClick={handle} {...rest}>{children}</a>;
}

export function routeForGroup(g: ShellGroup): Route {
  return g.items.length === 1 ? { kind: "screen", id: g.items[0].id } : { kind: "group", slug: g.slug };
}
```

- [ ] **Step 3: `src/shell/Cards.tsx`.**

```tsx
// Início (cards dos grupos) e tela do grupo (sub-cards): ícone + título +
// descrição, mesmo visual nos dois níveis (spec do subprojeto 2, §4.2).
import type { Icon } from "@tabler/icons-react";
import { RouteLink, routeForGroup } from "./RouteLink";
import type { Route } from "./router";
import type { Catalog, ShellGroup } from "./types";

function greeting(now: Date): string {
  const h = now.getHours();
  return h < 12 ? "Bom dia" : h < 18 ? "Boa tarde" : "Boa noite";
}

function Card({ route, go, icon: IconC, title, description, count }: {
  route: Route; go: (r: Route) => void; icon?: Icon; title: string; description?: string; count?: number;
}) {
  return (
    <RouteLink route={route} go={go} className="shell-card">
      <span className="shell-card-icon" aria-hidden="true">{IconC && <IconC size={24} stroke={1.75} />}</span>
      <span className="shell-card-title">
        {title}
        {count ? <span className="shell-card-count"> {count > 200 ? "200+" : count}</span> : null}
      </span>
      {description && <span className="shell-card-desc">{description}</span>}
    </RouteLink>
  );
}

export function HomeCards({ groups, catalog, go, now = new Date() }: {
  groups: ShellGroup[]; catalog: Catalog; go: (r: Route) => void; now?: Date;
}) {
  return (
    <section>
      <h1 className="shell-page-title">{greeting(now)}</h1>
      <p className="shell-page-sub">Escolha uma área para começar.</p>
      <div className="shell-cards">
        {groups.map((g) => (
          <Card key={g.slug} route={routeForGroup(g)} go={go} icon={catalog.groups[g.label]?.icon}
            title={g.label} description={catalog.groups[g.label]?.description} />
        ))}
      </div>
    </section>
  );
}

export function GroupCards({ group, catalog, go }: { group: ShellGroup; catalog: Catalog; go: (r: Route) => void }) {
  const meta = catalog.groups[group.label];
  const IconG = meta?.icon;
  return (
    <section>
      <div className="shell-page-head">
        <span className="shell-card-icon" aria-hidden="true">{IconG && <IconG size={24} stroke={1.75} />}</span>
        <div>
          <h1 className="shell-page-title">{group.label}</h1>
          {meta?.description && <p className="shell-page-sub">{meta.description}</p>}
        </div>
      </div>
      <div className="shell-cards">
        {group.items.map((i) => (
          <Card key={i.id} route={{ kind: "screen", id: i.id }} go={go} icon={catalog.screens[i.id]?.icon}
            title={i.label} description={catalog.screens[i.id]?.description} count={i.count} />
        ))}
      </div>
    </section>
  );
}
```

- [ ] **Step 4: `src/shell/Breadcrumb.tsx`.**

```tsx
// Trilha das telas internas: Início › Grupo › Tela. O último item abre as
// outras telas do mesmo grupo. Grupo de uma tela (ou tela fora do menu):
// Início › Tela.
import { useEffect, useRef, useState } from "react";
import { IconChevronDown } from "@tabler/icons-react";
import { RouteLink } from "./RouteLink";
import type { Route } from "./router";
import type { ShellGroup } from "./types";

export function Breadcrumb({ groups, route, go, titleFor }: {
  groups: ShellGroup[]; route: Route; go: (r: Route) => void; titleFor: (id: string) => string;
}) {
  const [ open, setOpen ] = useState(false);
  const ref = useRef<HTMLDivElement | null>(null);
  useEffect(() => {
    if (!open) return;
    const onDoc = (e: MouseEvent) => { if (!ref.current?.contains(e.target as Node)) setOpen(false); };
    const onKey = (e: KeyboardEvent) => { if (e.key === "Escape") setOpen(false); };
    document.addEventListener("mousedown", onDoc);
    document.addEventListener("keydown", onKey);
    return () => { document.removeEventListener("mousedown", onDoc); document.removeEventListener("keydown", onKey); };
  }, [ open ]);

  if (route.kind !== "screen") return null;
  const group = groups.find((g) => g.items.some((i) => i.id === route.id));
  const title = titleFor(route.id);
  const multi = group && group.items.length > 1;

  return (
    <nav aria-label="Trilha" className="shell-crumbs">
      <RouteLink route={{ kind: "home" }} go={go}>Início</RouteLink>
      <span aria-hidden="true">›</span>
      {multi && (
        <>
          <RouteLink route={{ kind: "group", slug: group.slug }} go={go}>{group.label}</RouteLink>
          <span aria-hidden="true">›</span>
          <div ref={ref} className="shell-crumb-switch">
            <button type="button" aria-haspopup="menu" aria-expanded={open} onClick={() => setOpen((v) => !v)}>
              {title} <IconChevronDown size={14} aria-hidden="true" />
            </button>
            {open && (
              <div role="menu" className="shell-menu">
                {group.items.map((i) => (
                  <button key={i.id} type="button" role="menuitem" aria-current={i.id === route.id ? "page" : undefined}
                    onClick={() => { setOpen(false); go({ kind: "screen", id: i.id }); }}>
                    {i.label}
                  </button>
                ))}
              </div>
            )}
          </div>
        </>
      )}
      {!multi && <span aria-current="page">{title}</span>}
    </nav>
  );
}
```

- [ ] **Step 5: `src/shell/Rail.tsx`.**

```tsx
// Trilho de ícones (≥1024px): Início + um link por grupo visível; o grupo da
// rota atual fica com aria-current. Rótulo acessível e dica = nome do grupo.
import { IconHome } from "@tabler/icons-react";
import { RouteLink, routeForGroup } from "./RouteLink";
import type { Route } from "./router";
import type { Catalog, ShellGroup } from "./types";

export function groupOfRoute(groups: ShellGroup[], route: Route): ShellGroup | undefined {
  if (route.kind === "group") return groups.find((g) => g.slug === route.slug);
  if (route.kind === "screen") return groups.find((g) => g.items.some((i) => i.id === route.id));
  return undefined;
}

export function Rail({ groups, catalog, route, go }: {
  groups: ShellGroup[]; catalog: Catalog; route: Route; go: (r: Route) => void;
}) {
  const current = groupOfRoute(groups, route);
  return (
    <nav aria-label="Grupos" id="menu-grupos" className="shell-rail">
      <RouteLink route={{ kind: "home" }} go={go} aria-label="Início" title="Início"
        aria-current={route.kind === "home" ? "page" : undefined} className="shell-rail-link">
        <IconHome size={22} stroke={1.75} aria-hidden="true" />
      </RouteLink>
      {groups.map((g) => {
        const IconG = catalog.groups[g.label]?.icon;
        const dot = g.items.some((i) => (i.count ?? 0) > 0);
        return (
          <RouteLink key={g.slug} route={routeForGroup(g)} go={go} aria-label={g.label} title={g.label}
            aria-current={current?.slug === g.slug ? "page" : undefined} className="shell-rail-link">
            {IconG && <IconG size={22} stroke={1.75} aria-hidden="true" />}
            {dot && <span className="shell-rail-dot" aria-hidden="true" />}
          </RouteLink>
        );
      })}
    </nav>
  );
}
```

- [ ] **Step 6: `src/shell/Drawer.tsx`.**

```tsx
// Gaveta (<1024px): diálogo modal com todos os grupos e telas. Esc, clique
// no fundo ou escolher uma tela fecham; foco preso dentro e devolvido ao
// botão Menu ao fechar.
import { useEffect, useRef } from "react";
import { IconX } from "@tabler/icons-react";
import { RouteLink } from "./RouteLink";
import type { Route } from "./router";
import type { ShellGroup } from "./types";

export function Drawer({ open, onClose, groups, route, go, returnFocusTo }: {
  open: boolean; onClose: () => void; groups: ShellGroup[]; route: Route; go: (r: Route) => void;
  returnFocusTo?: HTMLElement | null;
}) {
  const ref = useRef<HTMLDivElement | null>(null);

  useEffect(() => {
    if (!open) return;
    const panel = ref.current;
    panel?.querySelector<HTMLElement>("a,button")?.focus();
    function onKey(e: KeyboardEvent) {
      if (e.key === "Escape") { onClose(); return; }
      if (e.key !== "Tab" || !panel) return;
      const items = Array.from(panel.querySelectorAll<HTMLElement>("a,button"));
      if (items.length === 0) return;
      const first = items[0];
      const last = items[items.length - 1];
      if (!e.shiftKey && document.activeElement === last) { e.preventDefault(); first.focus(); }
      else if (e.shiftKey && document.activeElement === first) { e.preventDefault(); last.focus(); }
    }
    document.addEventListener("keydown", onKey);
    return () => { document.removeEventListener("keydown", onKey); returnFocusTo?.focus(); };
  }, [ open, onClose, returnFocusTo ]);

  if (!open) return null;
  const currentId = route.kind === "screen" ? route.id : null;
  return (
    <div className="shell-drawer-root">
      <div className="shell-drawer-backdrop" data-testid="drawer-backdrop" onMouseDown={onClose} />
      <div ref={ref} role="dialog" aria-modal="true" aria-label="Menu" className="shell-drawer">
        <div className="shell-drawer-head">
          <strong>Menu</strong>
          <button type="button" aria-label="Fechar menu" onClick={onClose}><IconX size={20} aria-hidden="true" /></button>
        </div>
        <nav aria-label="Menu">
          <RouteLink route={{ kind: "home" }} go={go} afterGo={onClose} className="shell-drawer-link">Início</RouteLink>
          {groups.map((g) => (
            <div key={g.slug}>
              <div className="shell-drawer-group">{g.label}</div>
              {g.items.map((i) => (
                <RouteLink key={i.id} route={{ kind: "screen", id: i.id }} go={go} afterGo={onClose}
                  aria-current={i.id === currentId ? "page" : undefined} className="shell-drawer-link">
                  {i.label}
                </RouteLink>
              ))}
            </div>
          ))}
        </nav>
      </div>
    </div>
  );
}
```

- [ ] **Step 7: `src/shell/shell.css`.**

```css
/* Shell dos painéis (spec do subprojeto 2, §4). Larguras: <1024 gaveta,
   ≥1024 trilho, ≥1280 relógio e 4 colunas de cards. */

.shell-root { min-height: 100vh; display: flex; flex-direction: column; }
.shell-header { position: sticky; top: 0; z-index: 10; background: var(--panel); }
.shell-body { display: flex; flex: 1; min-height: 0; }
.shell-main { flex: 1; min-width: 0; padding: 20px 24px 48px; }
.shell-main:focus { outline: none; }
.shell-main:focus-visible { outline: 3px solid var(--focus); outline-offset: -3px; }

.shell-emag { display: flex; justify-content: space-between; flex-wrap: wrap; gap: 6px 16px;
  padding: 4px 16px; font-size: 12px; background: var(--bg); border-bottom: 1px solid var(--rule); }
.shell-emag a, .shell-emag button { color: var(--accent); text-decoration: underline; font-size: 12px; padding: 2px 0; }
.shell-emag-group { display: flex; flex-wrap: wrap; gap: 4px 16px; }

.shell-topbar { display: flex; align-items: center; gap: 16px; padding: 8px 16px; border-bottom: 1px solid var(--rule); min-height: 56px; }
.shell-topbar-end { margin-left: auto; display: flex; align-items: center; gap: 10px; }
.shell-menu-btn { display: inline-flex; align-items: center; gap: 6px; min-height: 44px; padding: 0 12px;
  border: 1px solid var(--rule2) !important; border-radius: 8px; }

.shell-context { display: flex; align-items: center; flex-wrap: wrap; gap: 8px 12px; padding: 6px 16px;
  background: var(--sunken); border-bottom: 1px solid var(--rule); }
.shell-context-end { margin-left: auto; display: flex; align-items: center; flex-wrap: wrap; gap: 8px; }
.shell-clock { display: none; }

.shell-rail { display: none; flex-direction: column; align-items: center; gap: 4px; width: 60px; flex-shrink: 0;
  padding: 10px 0; background: var(--panel); border-right: 1px solid var(--rule); }
.shell-rail-link { position: relative; display: flex; align-items: center; justify-content: center;
  width: 44px; height: 44px; border-radius: 8px; color: var(--ink2); }
.shell-rail-link:hover { background: var(--sunken); }
.shell-rail-link[aria-current="page"] { background: var(--accent-bg); color: var(--accent); }
.shell-rail-dot { position: absolute; top: 8px; right: 8px; width: 8px; height: 8px; border-radius: 999px; background: var(--down); }

.shell-drawer-root { position: fixed; inset: 0; z-index: 40; }
.shell-drawer-backdrop { position: absolute; inset: 0; background: rgba(0, 0, 0, .4); }
.shell-drawer { position: absolute; top: 0; left: 0; bottom: 0; width: min(320px, 90vw); overflow-y: auto;
  background: var(--panel); box-shadow: var(--shadow-notif); padding: 8px 12px 24px; }
.shell-drawer-head { display: flex; align-items: center; justify-content: space-between; min-height: 48px; }
.shell-drawer-head button { width: 44px; height: 44px; display: inline-flex; align-items: center; justify-content: center; }
.shell-drawer-group { font-size: 12px; font-weight: 700; color: var(--ink2); padding: 12px 8px 4px; }
.shell-drawer-link { display: flex; align-items: center; min-height: 44px; padding: 0 12px; border-radius: 6px; color: var(--ink); }
.shell-drawer-link[aria-current="page"] { background: var(--accent-bg); color: var(--accent); font-weight: 600; }

.shell-page-title { margin: 0; font-size: 22px; font-weight: 700; color: var(--ink); }
.shell-page-sub { margin: 4px 0 16px; color: var(--ink2); }
.shell-page-head { display: flex; align-items: center; gap: 12px; margin-bottom: 16px; }
.shell-page-head .shell-page-sub { margin: 2px 0 0; }

.shell-cards { display: grid; gap: 14px; grid-template-columns: repeat(2, minmax(0, 1fr)); }
.shell-card { display: grid; gap: 6px; align-content: start; min-height: 132px; padding: 16px;
  background: var(--panel); border: 1px solid var(--rule2); border-radius: var(--radius-panel); color: var(--ink); }
.shell-card:hover { border-color: var(--accent); box-shadow: inset 0 0 0 1px var(--accent); }
.shell-card-icon { width: 44px; height: 44px; border-radius: 8px; display: inline-flex; align-items: center;
  justify-content: center; background: var(--accent-bg); color: var(--accent); flex-shrink: 0; }
.shell-card-title { font-size: 16px; font-weight: 700; }
.shell-card-count { display: inline-block; margin-left: 6px; padding: 0 7px; border-radius: 999px;
  background: var(--down); color: var(--on-tone); font-size: 12px; }
.shell-card-desc { font-size: 13px; color: var(--ink2); line-height: 1.45; }

.shell-crumbs { display: flex; align-items: center; flex-wrap: wrap; gap: 6px; font-size: 13px; color: var(--ink2); margin-bottom: 10px; }
.shell-crumbs a { color: var(--accent); text-decoration: underline; }
.shell-crumb-switch { position: relative; }
.shell-crumb-switch > button { display: inline-flex; align-items: center; gap: 4px; min-height: 32px; padding: 0 10px;
  border: 1px solid var(--rule2) !important; border-radius: 999px; font-weight: 600; color: var(--ink); }
.shell-menu { position: absolute; top: calc(100% + 6px); left: 0; z-index: 30; min-width: 240px; padding: 6px;
  background: var(--panel); border: 1px solid var(--rule2); border-radius: var(--radius-menu); box-shadow: var(--shadow-dropdown); }
.shell-menu [role="menuitem"], .shell-menu [role="menuitemradio"] { display: flex; width: 100%; min-height: 40px; align-items: center;
  gap: 8px; padding: 0 10px; border-radius: 6px; text-align: left; color: var(--ink); }
.shell-menu [role="menuitem"]:hover, .shell-menu [role="menuitemradio"]:hover { background: var(--sunken); }
.shell-menu [aria-current="page"], .shell-menu [aria-checked="true"] { font-weight: 700; color: var(--accent); }
.shell-menu-label { padding: 8px 10px 4px; font-size: 12px; color: var(--ink2); }
.shell-menu hr { border: 0; border-top: 1px solid var(--rule); margin: 4px 0; }

.shell-search { position: relative; }
.shell-search input { width: 100%; min-height: 40px; padding: 0 12px 0 34px; border: 1px solid var(--rule2);
  border-radius: 8px; background: var(--panel); color: var(--ink); font: inherit; }
.shell-search-icon { position: absolute; left: 10px; top: 50%; transform: translateY(-50%); color: var(--ink2); pointer-events: none; }
.shell-search [role="listbox"] { position: absolute; top: calc(100% + 6px); left: 0; right: 0; z-index: 30; padding: 6px;
  background: var(--panel); border: 1px solid var(--rule2); border-radius: var(--radius-menu); box-shadow: var(--shadow-dropdown); }
.shell-search [role="option"] { display: grid; padding: 8px 10px; border-radius: 6px; cursor: pointer; }
.shell-search [role="option"][aria-selected="true"] { background: var(--accent-bg); }
.shell-search-meta { font-size: 12px; color: var(--ink2); }

@media (min-width: 1024px) {
  .shell-menu-btn { display: none; }
  .shell-rail { display: flex; }
  .shell-cards { grid-template-columns: repeat(3, minmax(0, 1fr)); }
}

@media (min-width: 1280px) {
  .shell-clock { display: inline-flex; }
  .shell-cards { grid-template-columns: repeat(4, minmax(0, 1fr)); }
}
```

  O `.shell-main:focus { outline: none; }` não é estilo inline e só vale para o foco programático do `main` (Task 7). O `:focus-visible` logo abaixo devolve o contorno para o teclado. A guarda `a11y-guards` olha só `outline: "none"` em TS/TSX, então este CSS é permitido.

- [ ] **Step 8: Rodar os testes.**

  Run: `npx vitest run src/shell/Cards.test.tsx src/shell/Breadcrumb.test.tsx src/shell/Rail.test.tsx src/shell/Drawer.test.tsx`
  Expected: PASS em todos. Para o `Breadcrumb` "tela fora do menu", o `textContent` esperado é `"Início›Segurança"`, porque o separador é um `span`.

- [ ] **Step 9: Commit** (em cada app).

```bash
/opt/homebrew/bin/git add src/shell/RouteLink.tsx src/shell/Cards.tsx src/shell/Breadcrumb.tsx src/shell/Rail.tsx src/shell/Drawer.tsx src/shell/shell.css src/shell/Cards.test.tsx src/shell/Breadcrumb.test.tsx src/shell/Rail.test.tsx src/shell/Drawer.test.tsx
/opt/homebrew/bin/git commit -m "feat: add shell navigation pieces (cards, breadcrumb, rail, drawer)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Busca de tela, barra eMAG, menu da conta e página de acessibilidade

Mesmo código nos dois apps.

**Files:**
- Create: `src/shell/ScreenSearch.tsx`, `src/shell/EmagBar.tsx`, `src/shell/ContaMenu.tsx`, `src/shell/A11yPage.tsx`
- Create (testes): `src/shell/ScreenSearch.test.tsx`, `src/shell/EmagBar.test.tsx`, `src/shell/ContaMenu.test.tsx`

**Interfaces:**
- Consumes: `Route` (Task 3); `ShellGroup` e `Catalog` (Task 4); `useTheme`, `setTheme`, `toggleHighContrast` e `PanelTheme` (Task 2).
- Produces:
  - `ScreenSearch({ groups, catalog, go })`. O input tem `id="busca-tela"`.
  - `normalize(s: string): string`
  - `EmagBar()`, com âncoras para `#conteudo`, `#menu-grupos` / `#menu-botao` e `#busca-tela`
  - `ContaMenu({ email, onLogout })`
  - `A11yPage({ contact })`

- [ ] **Step 1: Escrever os testes, que vão falhar.**

`src/shell/ScreenSearch.test.tsx`:
```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { IconUsers } from "@tabler/icons-react";
import { ScreenSearch } from "./ScreenSearch";
import type { Catalog, ShellGroup } from "./types";

afterEach(cleanup);

const groups: ShellGroup[] = [
  { label: "Atendimento", slug: "atendimento", items: [ { id: "my-agenda", label: "Minha agenda" } ] },
  { label: "e-SUS", slug: "e-sus", items: [ { id: "production", label: "Produção e-SUS" } ] }
];
const meta = (d: string) => ({ icon: IconUsers, description: d });
const catalog: Catalog = {
  groups: {},
  screens: { "my-agenda": meta("Seus horários"), production: meta("Envio da produção") }
};

describe("ScreenSearch", () => {
  it("é um combobox ligado à lista", () => {
    render(<ScreenSearch groups={groups} catalog={catalog} go={vi.fn()} />);
    const box = screen.getByRole("combobox", { name: "Buscar tela" });
    expect(box.id).toBe("busca-tela");
    expect(box.getAttribute("aria-expanded")).toBe("false");
  });

  it("acha sem acento e abre com Enter", () => {
    const go = vi.fn();
    render(<ScreenSearch groups={groups} catalog={catalog} go={go} />);
    const box = screen.getByRole("combobox");
    fireEvent.change(box, { target: { value: "producao" } });
    expect(screen.getAllByRole("option").map((o) => o.textContent)).toEqual([ expect.stringContaining("Produção e-SUS") ]);
    fireEvent.keyDown(box, { key: "Enter" });
    expect(go).toHaveBeenCalledWith({ kind: "screen", id: "production" });
  });

  it("busca também na descrição e no grupo; setas movem a seleção", () => {
    render(<ScreenSearch groups={groups} catalog={catalog} go={vi.fn()} />);
    const box = screen.getByRole("combobox");
    fireEvent.change(box, { target: { value: "e" } });
    fireEvent.keyDown(box, { key: "ArrowDown" });
    const selected = screen.getAllByRole("option").find((o) => o.getAttribute("aria-selected") === "true");
    expect(box.getAttribute("aria-activedescendant")).toBe(selected?.id);
  });

  it("tela fora dos grupos visíveis não aparece; Esc limpa", () => {
    render(<ScreenSearch groups={groups} catalog={catalog} go={vi.fn()} />);
    const box = screen.getByRole("combobox") as HTMLInputElement;
    fireEvent.change(box, { target: { value: "equipe" } });
    expect(screen.queryAllByRole("option")).toHaveLength(0);
    fireEvent.keyDown(box, { key: "Escape" });
    expect(box.value).toBe("");
  });
});
```

`src/shell/EmagBar.test.tsx`:
```tsx
import { afterEach, describe, expect, it } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { EmagBar } from "./EmagBar";
import { getTheme } from "../theme/theme";

afterEach(() => { cleanup(); document.body.innerHTML = ""; localStorage.clear(); });

describe("EmagBar", () => {
  it("atalhos com accesskey 1, 2 e 3", () => {
    render(<EmagBar />);
    expect(screen.getByRole("link", { name: /Ir para o conteúdo/ }).getAttribute("accesskey")).toBe("1");
    expect(screen.getByRole("link", { name: /Ir para o menu/ }).getAttribute("accesskey")).toBe("2");
    expect(screen.getByRole("link", { name: /Ir para a busca/ }).getAttribute("accesskey")).toBe("3");
    expect(screen.getByRole("link", { name: "Acessibilidade" }).getAttribute("href")).toBe("#/acessibilidade");
  });

  it("Ir para o conteúdo move o foco e não troca o hash", () => {
    window.history.replaceState(null, "", "/dashboard/#/team");
    const main = document.createElement("main");
    main.id = "conteudo";
    document.body.appendChild(main);
    render(<EmagBar />);
    fireEvent.click(screen.getByRole("link", { name: /Ir para o conteúdo/ }));
    expect(document.activeElement).toBe(main);
    expect(window.location.hash).toBe("#/team");
  });

  it("Ir para o menu usa o trilho, ou o botão Menu quando o trilho está oculto", () => {
    const menuBtn = document.createElement("button");
    menuBtn.id = "menu-botao";
    document.body.appendChild(menuBtn);
    render(<EmagBar />);
    fireEvent.click(screen.getByRole("link", { name: /Ir para o menu/ }));
    expect(document.activeElement).toBe(menuBtn);
  });

  it("Alto contraste liga o tema", () => {
    render(<EmagBar />);
    fireEvent.click(screen.getByRole("button", { name: "Alto contraste" }));
    expect(getTheme()).toBe("alto-contraste");
    expect(screen.getByRole("button", { name: "Alto contraste" }).getAttribute("aria-pressed")).toBe("true");
  });
});
```

`src/shell/ContaMenu.test.tsx`:
```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { ContaMenu } from "./ContaMenu";
import { getTheme } from "../theme/theme";

afterEach(() => { cleanup(); localStorage.clear(); });

describe("ContaMenu", () => {
  it("mantém o botão Conta e o Sair", () => {
    const onLogout = vi.fn();
    render(<ContaMenu email="admin@curitiba.demo" onLogout={onLogout} />);
    fireEvent.click(screen.getByRole("button", { name: "Conta" }));
    expect(screen.getByText("admin@curitiba.demo")).toBeTruthy();
    fireEvent.click(screen.getByRole("menuitem", { name: "Sair" }));
    expect(onLogout).toHaveBeenCalled();
  });

  it("escolhe o tema entre gov.br, Rota Saúde e Alto contraste", () => {
    render(<ContaMenu email="a@b.c" onLogout={vi.fn()} />);
    fireEvent.click(screen.getByRole("button", { name: "Conta" }));
    expect(screen.getByRole("menuitemradio", { name: "gov.br" }).getAttribute("aria-checked")).toBe("true");
    fireEvent.click(screen.getByRole("menuitemradio", { name: "Rota Saúde" }));
    expect(getTheme()).toBe("rota");
  });
});
```

  Run: `npx vitest run src/shell/ScreenSearch.test.tsx src/shell/EmagBar.test.tsx src/shell/ContaMenu.test.tsx`
  Expected: FAIL, porque os módulos não existem.

- [ ] **Step 2: `src/shell/ScreenSearch.tsx`.**

```tsx
// Busca de tela (combobox ARIA 1.2): rótulo, grupo e descrição das telas
// visíveis, sem diferenciar acento nem maiúscula; até 8 resultados.
import { useId, useMemo, useState } from "react";
import { IconSearch } from "@tabler/icons-react";
import type { Route } from "./router";
import type { Catalog, ShellGroup } from "./types";

export function normalize(s: string): string {
  return s.normalize("NFD").replace(/[̀-ͯ]/g, "").toLowerCase();
}

export function ScreenSearch({ groups, catalog, go }: { groups: ShellGroup[]; catalog: Catalog; go: (r: Route) => void }) {
  const listId = useId();
  const [ query, setQuery ] = useState("");
  const [ active, setActive ] = useState(0);

  const entries = useMemo(() => groups.flatMap((g) => g.items.map((i) => ({
    id: i.id, label: i.label, group: g.label, haystack: normalize(`${i.label} ${g.label} ${catalog.screens[i.id]?.description ?? ""}`)
  }))), [ groups, catalog ]);

  const q = normalize(query.trim());
  const results = q ? entries.filter((e) => e.haystack.includes(q)).slice(0, 8) : [];
  const open = results.length > 0;
  const optionId = (n: number) => `${listId}-opt-${n}`;

  function choose(n: number) {
    const r = results[n];
    if (!r) return;
    setQuery("");
    setActive(0);
    go({ kind: "screen", id: r.id });
  }

  return (
    <div className="shell-search">
      <IconSearch size={16} className="shell-search-icon" aria-hidden="true" />
      <input
        id="busca-tela"
        type="text"
        role="combobox"
        aria-label="Buscar tela"
        aria-autocomplete="list"
        aria-expanded={open}
        aria-controls={listId}
        aria-activedescendant={open ? optionId(active) : undefined}
        placeholder="Buscar tela"
        value={query}
        onChange={(e) => { setQuery(e.target.value); setActive(0); }}
        onKeyDown={(e) => {
          if (e.key === "ArrowDown" && open) { e.preventDefault(); setActive((a) => (a + 1) % results.length); }
          else if (e.key === "ArrowUp" && open) { e.preventDefault(); setActive((a) => (a - 1 + results.length) % results.length); }
          else if (e.key === "Enter" && open) { e.preventDefault(); choose(active); }
          else if (e.key === "Escape") { setQuery(""); setActive(0); }
        }}
      />
      {open && (
        <div role="listbox" id={listId} aria-label="Telas encontradas">
          {results.map((r, n) => (
            <div key={r.id} id={optionId(n)} role="option" aria-selected={n === active}
              onMouseDown={(e) => { e.preventDefault(); choose(n); }}>
              <span>{r.label}</span>
              <span className="shell-search-meta">{r.group}</span>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 3: `src/shell/EmagBar.tsx`.**

```tsx
// Barra de acessibilidade (eMAG 3.1): atalhos 1 conteúdo, 2 menu, 3 busca;
// página de acessibilidade e alto contraste. Os atalhos movem o foco sem
// trocar o hash (o roteador usa o hash para as telas).
import type { MouseEvent } from "react";
import { toggleHighContrast, useTheme } from "../theme/theme";

// Foca o primeiro alvo visível; se nenhum estiver visível, o último da
// lista (o trilho some abaixo de 1024px e o botão Menu assume).
function focusFirst(...ids: string[]) {
  for (const id of ids) {
    const el = document.getElementById(id);
    if (el && (el.offsetParent !== null || id === ids[ids.length - 1])) {
      if (!el.hasAttribute("tabindex") && !/^(A|BUTTON|INPUT|SELECT|TEXTAREA)$/.test(el.tagName)) el.setAttribute("tabindex", "-1");
      el.focus();
      return;
    }
  }
}

export function EmagBar() {
  const theme = useTheme();
  const jump = (...ids: string[]) => (e: MouseEvent<HTMLAnchorElement>) => { e.preventDefault(); focusFirst(...ids); };
  return (
    <div className="shell-emag">
      <div className="shell-emag-group">
        <a href="#conteudo" accessKey="1" onClick={jump("conteudo")}>Ir para o conteúdo [1]</a>
        <a href="#menu-grupos" accessKey="2" onClick={jump("menu-grupos", "menu-botao")}>Ir para o menu [2]</a>
        <a href="#busca-tela" accessKey="3" onClick={jump("busca-tela")}>Ir para a busca [3]</a>
      </div>
      <div className="shell-emag-group">
        <a href="#/acessibilidade">Acessibilidade</a>
        <button type="button" aria-pressed={theme === "alto-contraste"} onClick={toggleHighContrast}>Alto contraste</button>
      </div>
    </div>
  );
}
```

  No jsdom, `offsetParent` é sempre `null`. Por isso, no teste "Ir para o menu", só o último alvo (`#menu-botao`) recebe o foco. No navegador, com o trilho visível, o `#menu-grupos` ganha.

- [ ] **Step 4: `src/shell/ContaMenu.tsx`.**

```tsx
// Menu da conta: e-mail, tema (gov.br, Rota Saúde, Alto contraste) e Sair.
// Mantém o botão aria-label="Conta" e o item "Sair" do UserMenu antigo.
import { useEffect, useRef, useState } from "react";
import { PANEL_THEMES, setTheme, useTheme, type PanelTheme } from "../theme/theme";

const THEME_LABEL: Record<PanelTheme, string> = { govbr: "gov.br", rota: "Rota Saúde", "alto-contraste": "Alto contraste" };

export function ContaMenu({ email, onLogout }: { email: string; onLogout: () => void }) {
  const [ open, setOpen ] = useState(false);
  const ref = useRef<HTMLDivElement | null>(null);
  const theme = useTheme();

  useEffect(() => {
    if (!open) return;
    const onDoc = (e: MouseEvent) => { if (!ref.current?.contains(e.target as Node)) setOpen(false); };
    const onKey = (e: KeyboardEvent) => { if (e.key === "Escape") setOpen(false); };
    document.addEventListener("mousedown", onDoc);
    document.addEventListener("keydown", onKey);
    return () => { document.removeEventListener("mousedown", onDoc); document.removeEventListener("keydown", onKey); };
  }, [ open ]);

  return (
    <div ref={ref} style={{ position: "relative" }}>
      <button type="button" aria-label="Conta" aria-haspopup="menu" aria-expanded={open} onClick={() => setOpen((v) => !v)}
        style={{ width: 40, height: 40, borderRadius: 999, background: "var(--ink)", color: "var(--panel)",
          fontFamily: "var(--font-mono)", fontSize: 12, fontWeight: 600 }}>
        {email.slice(0, 2).toUpperCase()}
      </button>
      {open && (
        <div role="menu" className="shell-menu" style={{ left: "auto", right: 0, width: 260 }}>
          <div className="shell-menu-label mono">{email}</div>
          <hr />
          <div className="shell-menu-label" id="tema-label">Tema</div>
          <div role="group" aria-labelledby="tema-label">
            {PANEL_THEMES.map((t) => (
              <button key={t} type="button" role="menuitemradio" aria-checked={theme === t} onClick={() => setTheme(t)}>
                {THEME_LABEL[t]}
              </button>
            ))}
          </div>
          <hr />
          <button type="button" role="menuitem" onClick={() => { setOpen(false); onLogout(); }}>Sair</button>
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 5: `src/shell/A11yPage.tsx`.**

```tsx
// Página de acessibilidade (eMAG 3.1): o que o painel oferece e como relatar
// problema. Estática.
export function A11yPage({ contact }: { contact: string }) {
  return (
    <article style={{ maxWidth: 760 }}>
      <h1 className="shell-page-title">Acessibilidade</h1>
      <p className="shell-page-sub">Este painel segue a WCAG 2.2 nível AA e o eMAG 3.1.</p>
      <h2>Atalhos de teclado</h2>
      <table>
        <thead><tr><th>Tecla</th><th>Vai para</th></tr></thead>
        <tbody>
          <tr><td>1</td><td>Conteúdo da página</td></tr>
          <tr><td>2</td><td>Menu de grupos</td></tr>
          <tr><td>3</td><td>Busca de tela</td></tr>
        </tbody>
      </table>
      <p>Use a tecla junto do modificador do seu navegador: Alt no Chrome e no Edge, Alt+Shift no Firefox.</p>
      <h2>Recursos</h2>
      <ul>
        <li>Alto contraste, pelo botão na barra do topo ou pelo menu da conta.</li>
        <li>Temas gov.br e Rota Saúde, no menu da conta. A escolha fica salva neste navegador.</li>
        <li>Foco visível em todo elemento que recebe o teclado.</li>
        <li>Nível de risco mostrado por cor, forma (■ alto, ▲ moderado, ● baixo) e texto.</li>
        <li>Títulos de página e marcos de região para leitores de tela.</li>
      </ul>
      <h2>Encontrou um problema?</h2>
      <p>{contact}</p>
    </article>
  );
}
```

- [ ] **Step 6: Rodar os testes.**

  Run: `npx vitest run src/shell/ScreenSearch.test.tsx src/shell/EmagBar.test.tsx src/shell/ContaMenu.test.tsx`
  Expected: PASS em todos.

- [ ] **Step 7: Commit** (em cada app).

```bash
/opt/homebrew/bin/git add src/shell/ScreenSearch.tsx src/shell/EmagBar.tsx src/shell/ContaMenu.tsx src/shell/A11yPage.tsx src/shell/ScreenSearch.test.tsx src/shell/EmagBar.test.tsx src/shell/ContaMenu.test.tsx
/opt/homebrew/bin/git commit -m "feat: add screen search, eMAG bar, account menu and accessibility page

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: `Brand`, `LiveClock`, `TenantChip`, `ContextStrip`, `TopBar` e `Shell`

**Files:**
- Create (iguais nos dois apps): `src/shell/Brand.tsx`, `src/shell/ContextStrip.tsx`, `src/shell/TopBar.tsx`, `src/shell/Shell.tsx`, `src/shell/Shell.test.tsx`
- Create (por app): `src/shell/LiveClock.tsx`. Só no dashboard: `src/shell/TenantChip.tsx`.

**Interfaces:**
- Consumes: todas as peças das Tasks 3 a 6.
- Produces:
  - `Brand({ product })`
  - `LiveClock()`
  - `TenantChip({ user })`, só no dashboard
  - `ContextStrip({ left, right })`
  - `TopBar({ product, groups, catalog, go, notifications, account, onMenu, menuOpen, menuButtonRef })`
  - `Shell({ product, groups, catalog, route, go, titleFor, context, notifications, account, children })`

- [ ] **Step 1: Escrever o teste do `Shell`, que vai falhar.** `src/shell/Shell.test.tsx`:

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { IconUsers } from "@tabler/icons-react";
import { Shell } from "./Shell";
import type { Route } from "./router";
import type { Catalog, ShellGroup } from "./types";

afterEach(cleanup);

const groups: ShellGroup[] = [
  { label: "Atendimento", slug: "atendimento", items: [
    { id: "attendance", label: "Atendimento" }, { id: "my-agenda", label: "Minha agenda" } ] }
];
const meta = { icon: IconUsers, description: "d" };
const catalog: Catalog = { groups: { Atendimento: meta }, screens: { attendance: meta, "my-agenda": meta } };
const titleFor = (id: string) => (id === "my-agenda" ? "Minha agenda" : "Atendimento");

function renderShell(route: Route) {
  return render(
    <Shell product="Dashboard" groups={groups} catalog={catalog} route={route} go={vi.fn()} titleFor={titleFor}
      context={<span>contexto</span>} notifications={<span>sino</span>} account={<span>conta</span>}>
      {route.kind === "screen" ? <h1>{titleFor(route.id)}</h1> : <h1>Início</h1>}
    </Shell>
  );
}

describe("Shell", () => {
  it("tem os marcos: header, nav Grupos, main#conteudo e a trilha nas telas", () => {
    renderShell({ kind: "screen", id: "my-agenda" });
    expect(screen.getByRole("banner")).toBeTruthy();
    expect(screen.getByRole("navigation", { name: "Grupos" })).toBeTruthy();
    expect(screen.getByRole("main").id).toBe("conteudo");
    expect(screen.getByRole("navigation", { name: "Trilha" })).toBeTruthy();
    expect(screen.getByText("contexto")).toBeTruthy();
  });

  it("título da aba por rota e foco no h1 ao trocar", () => {
    const { rerender } = renderShell({ kind: "home" });
    expect(document.title).toBe("Início · Rota Saúde");
    rerender(
      <Shell product="Dashboard" groups={groups} catalog={catalog} route={{ kind: "screen", id: "my-agenda" }} go={vi.fn()}
        titleFor={titleFor} context={null} notifications={null} account={null}>
        <h1>Minha agenda</h1>
      </Shell>
    );
    expect(document.title).toBe("Minha agenda · Rota Saúde");
    expect(document.activeElement?.textContent).toBe("Minha agenda");
  });

  it("botão Menu abre a gaveta", () => {
    renderShell({ kind: "home" });
    fireEvent.click(screen.getByRole("button", { name: "Menu" }));
    expect(screen.getByRole("dialog", { name: "Menu" })).toBeTruthy();
  });
});
```

  O `Shell` foca o `h1` num `useEffect` (síncrono depois do render), não com `requestAnimationFrame`. Por isso o teste não precisa de timers.

  Run: `npx vitest run src/shell/Shell.test.tsx`
  Expected: FAIL, porque o módulo `./Shell` não existe.

- [ ] **Step 2: Extrair `Brand`, `LiveClock` e `TenantChip`.**

  `src/shell/Brand.tsx` (igual nos dois apps). Os estilos são os do `Brand` atual em `AppHeader.tsx`, com o subtítulo vindo por prop e a marca virando link para o Início:

```tsx
// Marca (NAVBAR.md §2), agora link para o Início.
export function Brand({ product }: { product: string }) {
  return (
    <a href="#/" aria-label={`Rota Saúde ${product} — Início`} style={{ display: "flex", alignItems: "center", gap: 10, flexShrink: 0, color: "var(--ink)" }}>
      <span aria-hidden="true" style={{ width: 30, height: 30, borderRadius: 7, background: "var(--ink)", color: "var(--panel)",
        display: "flex", alignItems: "center", justifyContent: "center", flexShrink: 0 }}>
        <svg width="19" height="19" viewBox="0 0 48 48" fill="none" stroke="currentColor">
          <circle cx="24" cy="24" r="18" strokeWidth="3" />
          <path d="M8 24 H16 L19 18 L24 30 L29 18 L32 24 H40" strokeWidth="3.4" strokeLinecap="round" strokeLinejoin="round" />
        </svg>
      </span>
      <span style={{ lineHeight: 1.12 }}>
        <span style={{ display: "block", fontSize: 14, fontWeight: 700 }}>Rota Saúde</span>
        <span className="mono" style={{ display: "block", marginTop: 1, fontSize: 10.5, color: "var(--ink3)",
          textTransform: "uppercase", letterSpacing: 0.8 }}>{product}</span>
      </span>
    </a>
  );
}
```

  A marca é um `<a href="#/">` comum. O clique muda o hash, e o roteador escuta `hashchange` e atualiza a rota (Task 3).

  - **`src/shell/LiveClock.tsx`:** mova `LiveClock` e `formatNow` do `AppHeader.tsx` de cada app sem mudar nada além do `export` e do `className="shell-clock"` no `<span>` externo.
    - O dashboard importa `cityDateFormat` e `cityOffsetLabel` de `../lib/format`.
    - O admin mantém o texto `BRT` e importa `formatNow` do que o `AppHeader` do admin usa hoje: copie as linhas do `AppHeader` do admin tal como estão.
    - No `<span>` externo, troque `display: "flex"` por nada. Quem controla a exibição é a classe `shell-clock` (`display:none` abaixo de 1280px e `inline-flex` a partir daí), e o resto dos estilos fica igual.
  - **`src/shell/TenantChip.tsx` (dashboard):** mova `TenantChip` do `AppHeader.tsx` sem mudanças, só com `export`.

- [ ] **Step 3: `src/shell/ContextStrip.tsx` e `src/shell/TopBar.tsx`** (iguais nos dois apps).

```tsx
// ContextStrip.tsx — faixa de contexto sempre visível: o que muda os números
// (período), onde se está (cidade) e avisos legais (sessão de assinatura).
import type { ReactNode } from "react";

export function ContextStrip({ left, right }: { left: ReactNode; right?: ReactNode }) {
  return (
    <div className="shell-context" aria-label="Contexto">
      {left}
      {right && <div className="shell-context-end">{right}</div>}
    </div>
  );
}
```

```tsx
// TopBar.tsx — barra de topo só de navegação: Menu (<1024px), marca, busca,
// sino e conta.
import type { ReactNode, RefObject } from "react";
import { IconMenu2 } from "@tabler/icons-react";
import { Brand } from "./Brand";
import { ScreenSearch } from "./ScreenSearch";
import type { Route } from "./router";
import type { Catalog, ShellGroup } from "./types";

export function TopBar({ product, groups, catalog, go, notifications, account, onMenu, menuOpen, menuButtonRef }: {
  product: string; groups: ShellGroup[]; catalog: Catalog; go: (r: Route) => void;
  notifications: ReactNode; account: ReactNode; onMenu: () => void; menuOpen: boolean;
  menuButtonRef: RefObject<HTMLButtonElement>;
}) {
  return (
    <div className="shell-topbar">
      <button ref={menuButtonRef} id="menu-botao" type="button" className="shell-menu-btn"
        aria-haspopup="dialog" aria-expanded={menuOpen} onClick={onMenu}>
        <IconMenu2 size={20} aria-hidden="true" /> Menu
      </button>
      <Brand product={product} />
      <div style={{ flex: "0 1 320px", minWidth: 160 }}>
        <ScreenSearch groups={groups} catalog={catalog} go={go} />
      </div>
      <div className="shell-topbar-end">
        {notifications}
        {account}
      </div>
    </div>
  );
}
```

- [ ] **Step 4: `src/shell/Shell.tsx`** (igual nos dois apps).

```tsx
// Shell dos painéis (spec do subprojeto 2, §4): barra eMAG, barra de topo,
// faixa de contexto, trilho (≥1024) ou gaveta (<1024), trilha e conteúdo.
// Troca de rota: <title> e foco no h1 do conteúdo (ou no main).
import { useEffect, useRef, useState, type ReactNode } from "react";
import "./shell.css";
import { EmagBar } from "./EmagBar";
import { TopBar } from "./TopBar";
import { Rail } from "./Rail";
import { Drawer } from "./Drawer";
import { Breadcrumb } from "./Breadcrumb";
import { toHash, type Route } from "./router";
import type { Catalog, ShellGroup } from "./types";

export function Shell({ product, groups, catalog, route, go, titleFor, context, notifications, account, children }: {
  product: string; groups: ShellGroup[]; catalog: Catalog; route: Route; go: (r: Route) => void;
  titleFor: (id: string) => string; context: ReactNode; notifications: ReactNode; account: ReactNode; children: ReactNode;
}) {
  const [ menuOpen, setMenuOpen ] = useState(false);
  const menuButtonRef = useRef<HTMLButtonElement>(null);
  const mainRef = useRef<HTMLElement>(null);
  const first = useRef(true);
  const key = toHash(route);

  const pageTitle =
    route.kind === "home" ? "Início" :
    route.kind === "a11y" ? "Acessibilidade" :
    route.kind === "group" ? (groups.find((g) => g.slug === route.slug)?.label ?? "Início") :
    titleFor(route.id);

  useEffect(() => {
    document.title = `${pageTitle} · Rota Saúde`;
    if (first.current) { first.current = false; return; }
    const main = mainRef.current;
    if (!main) return;
    const target = main.querySelector<HTMLElement>("h1") ?? main;
    if (!target.hasAttribute("tabindex")) target.setAttribute("tabindex", "-1");
    target.focus();
  }, [ key, pageTitle ]);

  return (
    <div className="shell-root">
      <header className="shell-header">
        <EmagBar />
        <TopBar product={product} groups={groups} catalog={catalog} go={go} notifications={notifications}
          account={account} onMenu={() => setMenuOpen(true)} menuOpen={menuOpen} menuButtonRef={menuButtonRef} />
        {context}
      </header>
      <Drawer open={menuOpen} onClose={() => setMenuOpen(false)} groups={groups} route={route} go={go}
        returnFocusTo={menuButtonRef.current} />
      <div className="shell-body">
        <Rail groups={groups} catalog={catalog} route={route} go={go} />
        <main id="conteudo" ref={mainRef} className="shell-main">
          <Breadcrumb groups={groups} route={route} go={go} titleFor={titleFor} />
          {children}
        </main>
      </div>
    </div>
  );
}
```

- [ ] **Step 5: Rodar o teste e conferir os tipos.**

  Run: `npx vitest run src/shell/Shell.test.tsx && npx tsc --noEmit -p .`
  Expected: PASS e `tsc` limpo. O `AppHeader.tsx` continua intacto até a Task 8: nesta task, `LiveClock` e `TenantChip` são **copiados** para os arquivos novos, e as cópias antigas saem junto com o `AppHeader`.

- [ ] **Step 6: Commit** (em cada app).

```bash
/opt/homebrew/bin/git add src/shell/Brand.tsx src/shell/LiveClock.tsx src/shell/ContextStrip.tsx src/shell/TopBar.tsx src/shell/Shell.tsx src/shell/Shell.test.tsx
/opt/homebrew/bin/git commit -m "feat: compose the panel shell layout

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

  No dashboard, inclua também `src/shell/TenantChip.tsx`.

---

### Task 8: Ligar o shell no `App.tsx` do dashboard

**Files:**
- Modify: `src/App.tsx`
- Create: `src/App.navigation.test.tsx`
- Delete: `src/shell/AppHeader.tsx`, `src/shell/NavDropdown.tsx`
- Modify: `src/theme/a11y-guards.test.ts` (troca o arquivo vigiado)

**Interfaces:**
- Consumes: `useHashRoute`, `slugOf`, `Route` (Task 3); `Shell`, `ContextStrip`, `LiveClock`, `TenantChip`, `ContaMenu`, `HomeCards`, `GroupCards`, `A11yPage` e `CATALOG` (Tasks 4 a 7).
- Produces:
  - `export function renderModule(active: ModuleId, navigate: (id: ModuleId) => void): JSX.Element`, exportada para o teste
  - `export function shellGroups(groups: NavGroupDef[], pending: number): ShellGroup[]`

- [ ] **Step 1: Escrever o teste de navegação entre telas, que vai falhar.** `src/App.navigation.test.tsx`:

```tsx
// Os atalhos entre telas continuam abrindo o mesmo destino, agora via rota.
import { describe, expect, it, vi } from "vitest";
import type { ReactElement } from "react";
import { renderModule, shellGroups } from "./App";
import { NAV_GROUPS, navGroupsFor } from "./shell/modules";

type El = ReactElement<Record<string, (id?: string) => void>>;

describe("atalhos entre telas", () => {
  it.each([
    [ "overview", "onNavigate", "queues" ],
    [ "overview", "onNavigate", "health" ],
    [ "overview", "onNavigate", "events" ],
    [ "protocols", "onNavigate", "security" ],
    [ "team", "onNavigate", "professionals" ],
    [ "attendance", "onNavigate", "security" ],
    [ "campaigns", "onNavigate", "security" ],
    [ "signature-pending", "onNavigate", "signature" ]
  ] as const)("%s.%s(%s) navega", (screenId, prop, target) => {
    const navigate = vi.fn();
    const el = renderModule(screenId, navigate) as El;
    el.props[prop](target);
    expect(navigate).toHaveBeenCalledWith(target);
  });

  it.each([ "integrations", "cnes", "production", "my-consultations", "professional-consultations",
    "open-drafts", "clinical-record", "signature", "clinical-documents-admin" ] as const)
  ("%s.onGoToSecurity abre Segurança", (screenId) => {
    const navigate = vi.fn();
    (renderModule(screenId, navigate) as El).props.onGoToSecurity();
    expect(navigate).toHaveBeenCalledWith("security");
  });
});

describe("grupos do shell", () => {
  it("slugs e itens vêm de NAV_GROUPS, com contagem só em Pendentes", () => {
    const gs = shellGroups(NAV_GROUPS, 3);
    expect(gs.map((g) => g.slug)).toContain("visao-geral");
    expect(gs.map((g) => g.slug)).toContain("e-sus");
    const pending = gs.flatMap((g) => g.items).find((i) => i.id === "signature-pending");
    expect(pending).toEqual({ id: "signature-pending", label: "Pendentes de assinatura", count: 3 });
  });

  it("profissional de saúde não vê Equipe nem Cidade", () => {
    const user = { operator: false, memberships: [ { role: "health_professional" } ] };
    const slugs = shellGroups(navGroupsFor(user), 0).map((g) => g.slug);
    expect(slugs).not.toContain("equipe");
    expect(slugs).not.toContain("cidade");
    expect(slugs).toContain("atendimento");
  });
});
```

  Run: `npx vitest run src/App.navigation.test.tsx`
  Expected: FAIL, porque `renderModule` e `shellGroups` não são exportadas.

- [ ] **Step 2: Reescrever `src/App.tsx`.** Os imports das telas ficam iguais aos de hoje. Remova o import de `AppHeader` e acrescente:

```tsx
import { useCallback, useMemo, useState } from "react";
import { ScopeContext, PERIOD_OPTIONS, useScope, type PeriodKey } from "./lib/scope";
import { useAuth } from "./lib/auth";
import { NAV_GROUPS, labelFor, navGroupsFor, type ModuleId, type NavGroupDef } from "./shell/modules";
import { useAlerts } from "./hooks/useAlerts";
import { usePendingSignatures } from "./hooks/usePendingSignatures";
import { canSign } from "./lib/signature";
import { Shell } from "./shell/Shell";
import { ContextStrip } from "./shell/ContextStrip";
import { TenantChip } from "./shell/TenantChip";
import { LiveClock } from "./shell/LiveClock";
import { ContaMenu } from "./shell/ContaMenu";
import { HomeCards, GroupCards } from "./shell/Cards";
import { A11yPage } from "./shell/A11yPage";
import { SegmentedControl } from "./shell/SegmentedControl";
import { NotificationCenter } from "./shell/NotificationCenter";
import { SignatureSessionBadge } from "./shell/SignatureSessionBadge";
import { CATALOG } from "./shell/catalog";
import { slugOf, useHashRoute, type Route } from "./shell/router";
import type { ShellGroup } from "./shell/types";
```

  Corpo, substituindo do `export function App` até o fim do `ShellInner`:

```tsx
const ALL_IDS: ReadonlySet<string> = new Set(NAV_GROUPS.flatMap((g) => g.items.map((i) => i.id)));
const ALL_SLUGS: ReadonlySet<string> = new Set(NAV_GROUPS.map((g) => slugOf(g.label)));

// Grupos do shell a partir do filtro por papel de hoje. A contagem de
// pendentes de assinatura vira selo (o rótulo fica sem o "(n)").
export function shellGroups(groups: NavGroupDef[], pending: number): ShellGroup[] {
  return groups.map((g) => ({
    label: g.label,
    slug: slugOf(g.label),
    items: g.items.map((i) => (i.id === "signature-pending" && pending > 0
      ? { id: i.id, label: i.label, count: pending }
      : { id: i.id, label: i.label }))
  }));
}

// `initialModule`: a tela de onde a pessoa saiu para o prestador de
// assinatura (módulo 19b) ou para o SNCR, devolvida pelo retorno do OAuth.
export function App({ initialModule }: { initialModule?: ModuleId } = {}) {
  const [ period, setPeriod ] = useState<PeriodKey>("7d");
  const { citySlug } = useAuth();
  return (
    <ScopeContext.Provider value={{ period, citySlug, setPeriod }}>
      <ShellInner initialModule={initialModule} />
    </ScopeContext.Provider>
  );
}

// Dentro do Provider: useAlerts (via useHealth → useScope) consome o contexto.
function ShellInner({ initialModule }: { initialModule?: ModuleId }) {
  const alerts = useAlerts();
  const auth = useAuth();
  const scope = useScope();
  const pending = usePendingSignatures(canSign(auth.user));
  const groups = useMemo(
    () => shellGroups(navGroupsFor(auth.user), pending.data?.length ?? 0),
    [ auth.user, pending.data?.length ]
  );

  const isAllowed = useCallback((r: Route<ModuleId>) => {
    if (r.kind === "home" || r.kind === "a11y") return true;
    if (r.kind === "group") return groups.some((g) => g.slug === r.slug);
    return groups.some((g) => g.items.some((i) => i.id === r.id));
  }, [ groups ]);

  const { route, go } = useHashRoute<ModuleId>({
    ids: ALL_IDS, slugs: ALL_SLUGS, isAllowed,
    initial: initialModule ? { kind: "screen", id: initialModule } : undefined
  });
  const navigate = useCallback((id: ModuleId) => go({ kind: "screen", id }), [ go ]);
  const active: ModuleId = route.kind === "screen" ? route.id : "overview";

  let content;
  if (route.kind === "home") content = <HomeCards groups={groups} catalog={CATALOG} go={go} />;
  else if (route.kind === "a11y") content = <A11yPage contact="Fale com o administrador da sua prefeitura." />;
  else if (route.kind === "group") {
    const group = groups.find((g) => g.slug === route.slug);
    content = group ? <GroupCards group={group} catalog={CATALOG} go={go} /> : null;
  } else content = renderModule(route.id, navigate);

  return (
    <Shell
      product="Dashboard"
      groups={groups}
      catalog={CATALOG}
      route={route}
      go={go}
      titleFor={(id) => labelFor(id as ModuleId)}
      context={
        <ContextStrip
          left={<>
            <TenantChip user={auth.user} />
            <SegmentedControl options={PERIOD_OPTIONS} value={scope.period} onChange={scope.setPeriod} />
          </>}
          right={<>
            <SignatureSessionBadge active={active} onSelect={navigate} />
            <LiveClock />
          </>}
        />
      }
      notifications={<NotificationCenter active={alerts.active} limitations={alerts.limitations} onNavigate={navigate} />}
      account={auth.user ? <ContaMenu email={auth.user.email_address} onLogout={() => { void auth.logout(); }} /> : null}
    >
      {content}
    </Shell>
  );
}
```

  Na função `renderModule`:
  - acrescente `export` antes de `function renderModule`;
  - troque a assinatura para `(active: ModuleId, setActive: (id: ModuleId) => void)` (o nome do parâmetro continua `setActive`);
  - o corpo do `switch` **não muda**.

  O teste chama `renderModule(id, navigate)`, e cada `onNavigate={setActive}` e `onGoToSecurity={() => setActive("security")}` passa a chamar o `navigate`.

  Ruling registrada no plano: fora de uma tela (no Início ou no grupo), o `SignatureSessionBadge` recebe `active="overview"`. O `return_to` passa a ser `/overview`, e quem abrir a sessão pelo Início volta à Visão geral, não ao Início. O custo é pequeno, porque o `moduleFromPath` só aceita ids de tela.

- [ ] **Step 3: Remover o cabeçalho antigo e redirecionar a guarda.**

```bash
/opt/homebrew/bin/git rm -q src/shell/AppHeader.tsx src/shell/NavDropdown.tsx
grep -rn "AppHeader\|NavDropdown" src
```

  O `grep` deve achar só `a11y-guards.test.ts`. Nele, troque `it.each(["shell/AppHeader.tsx", "modules/Overview.tsx"])` por:

```ts
  it.each(["shell/TopBar.tsx", "shell/ContextStrip.tsx", "shell/LiveClock.tsx", "modules/Overview.tsx"])("%s", (file) => {
```

- [ ] **Step 4: Rodar tudo.**

  Run: `npx vitest run src/App.navigation.test.tsx` e em seguida `npx vitest run 2>&1 | tail -5`, `npx tsc --noEmit -p .` e `npm run build`.
  Expected: o teste novo passa; a suíte fica toda verde, com os testes anteriores ao ciclo sem edição; o `tsc` e o build ficam limpos.

- [ ] **Step 5: Commit.**

```bash
/opt/homebrew/bin/git add src/App.tsx src/App.navigation.test.tsx src/theme/a11y-guards.test.ts
/opt/homebrew/bin/git commit -m "feat: wire the dashboard to the new shell with hash routing

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

  O `git rm` do Step 3 já deixou as remoções preparadas para este commit.

---

### Task 9: Ligar o shell no `App.tsx` do admin

**Files:**
- Modify: `src/App.tsx`
- Create: `src/App.navigation.test.tsx`
- Delete: `src/shell/AppHeader.tsx`, `src/shell/NavDropdown.tsx`
- Modify: `src/theme/a11y-guards.test.ts`

**Interfaces:**
- Produces:
  - `export function renderModule(id: ModuleId, onNavigate: (id: ModuleId) => void)`
  - `export function shellGroupsFor(user: { operator: boolean; memberships: { role: string }[] } | null): ShellGroup[]`

- [ ] **Step 1: Escrever o teste, que vai falhar.** `src/App.navigation.test.tsx`:

```tsx
import { describe, expect, it, vi } from "vitest";
import type { ReactElement } from "react";
import { renderModule, shellGroupsFor } from "./App";

type El = ReactElement<Record<string, (id?: string) => void>>;

describe("atalhos entre telas do console", () => {
  it.each([ [ "overview", "queues" ], [ "overview", "health" ], [ "overview", "events" ], [ "provision_city", "cities" ] ] as const)
  ("%s navega para %s", (screenId, target) => {
    const navigate = vi.fn();
    (renderModule(screenId, navigate) as El).props.onNavigate(target);
    expect(navigate).toHaveBeenCalledWith(target);
  });
});

describe("grupos do console", () => {
  it("operador vê Setup, Analytics e e-SUS", () => {
    const gs = shellGroupsFor({ operator: true, memberships: [] });
    expect(gs.map((g) => g.slug)).toEqual([ "setup", "analytics", "e-sus" ]);
    expect(gs[0].items.map((i) => i.id)).toEqual([ "cities", "provision_city", "register_channel", "unknown_channels" ]);
  });

  it("sem operador, nenhum grupo", () => {
    expect(shellGroupsFor({ operator: false, memberships: [] })).toEqual([]);
    expect(shellGroupsFor(null)).toEqual([]);
  });
});
```

  Run: `npx vitest run src/App.navigation.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: Reescrever `src/App.tsx` do admin.** Os imports das telas ficam iguais aos de hoje. Remova o import de `AppHeader`, acrescente os mesmos imports do shell da Task 8 (sem `TenantChip`, `SignatureSessionBadge`, `usePendingSignatures`, `canSign` e `useAlerts`) e importe `NAV_GROUPS` de `./shell/modules`.

```tsx
const OFF_MENU: ReadonlySet<string> = new Set([ "setup_mfa" ]);
const ALL_IDS: ReadonlySet<string> = new Set([ ...NAV_GROUPS.flatMap((g) => g.items.map((i) => i.id)), ...OFF_MENU ]);
const ALL_SLUGS: ReadonlySet<string> = new Set(NAV_GROUPS.map((g) => slugOf(g.label)));

// Mesmo filtro do AppHeader antigo: item com `visible` exige usuário.
export function shellGroupsFor(user: { operator: boolean; memberships: { role: string }[] } | null): ShellGroup[] {
  return NAV_GROUPS.map((g) => ({
    label: g.label,
    slug: slugOf(g.label),
    items: g.items
      .filter((it) => !it.visible || (user ? it.visible({ operator: user.operator, memberships: user.memberships }) : false))
      .map((it) => ({ id: it.id, label: it.label }))
  })).filter((g) => g.items.length > 0);
}

export function App() {
  const [ period, setPeriod ] = useState<PeriodKey>("7d");
  // O console não tem noção de cidade ativa; o switch real de cidade é o
  // grant de 60s, não um estado de sessão.
  return (
    <ScopeContext.Provider value={{ period, setPeriod }}>
      <ShellInner />
    </ScopeContext.Provider>
  );
}

function ShellInner() {
  const auth = useAuth();
  const scope = useScope();
  const groups = useMemo(() => shellGroupsFor(auth.user), [ auth.user ]);
  const isAllowed = useCallback((r: Route<ModuleId>) => {
    if (r.kind === "home" || r.kind === "a11y") return true;
    if (r.kind === "group") return groups.some((g) => g.slug === r.slug);
    return OFF_MENU.has(r.id) || groups.some((g) => g.items.some((i) => i.id === r.id));
  }, [ groups ]);
  const { route, go } = useHashRoute<ModuleId>({ ids: ALL_IDS, slugs: ALL_SLUGS, isAllowed });
  const navigate = useCallback((id: ModuleId) => go({ kind: "screen", id }), [ go ]);
  const titleFor = (id: string) => NAV_GROUPS.flatMap((g) => g.items).find((i) => i.id === id)?.label ?? id;

  let content;
  if (route.kind === "home") content = <HomeCards groups={groups} catalog={CATALOG} go={go} />;
  else if (route.kind === "a11y") content = <A11yPage contact="Fale com a equipe da plataforma." />;
  else if (route.kind === "group") {
    const group = groups.find((g) => g.slug === route.slug);
    content = group ? <GroupCards group={group} catalog={CATALOG} go={go} /> : null;
  } else content = renderModule(route.id, navigate);

  return (
    <Shell product="Admin Console" groups={groups} catalog={CATALOG} route={route} go={go} titleFor={titleFor}
      context={<ContextStrip
        left={<SegmentedControl options={PERIOD_OPTIONS} value={scope.period} onChange={scope.setPeriod} />}
        right={<LiveClock />} />}
      notifications={<NotificationCenter active={[]} limitations={[]} onNavigate={navigate} />}
      account={auth.user ? <ContaMenu email={auth.user.email_address} onLogout={() => { void auth.logout(); }} /> : null}>
      {content}
    </Shell>
  );
}
```

  Acrescente `export` em `function renderModule` sem mudar o corpo. Confira que `useAuth` e `useScope` existem no admin (`src/lib/auth`, `src/lib/scope`), com os mesmos nomes do `AppHeader` antigo do admin, e que `PERIOD_OPTIONS` vem de `./lib/scope`. Se o nome for diferente, use o que o `AppHeader` antigo importava.

  O `NotificationCenter` recebe `[]` e `[]`, como fazia o `AppHeader` antigo (`alerts?.active ?? []`).

- [ ] **Step 3: Remover o cabeçalho antigo e redirecionar a guarda.** Rode `git rm` em `src/shell/AppHeader.tsx` e `src/shell/NavDropdown.tsx`. No `a11y-guards.test.ts`, faça a mesma troca da Task 8, Step 3.

- [ ] **Step 4: Rodar tudo.**

  Run: `npx vitest run 2>&1 | tail -5`, depois `npx tsc --noEmit -p .` e `npm run build`.
  Expected: tudo verde e limpo.

- [ ] **Step 5: Commit.**

```bash
/opt/homebrew/bin/git add src/App.tsx src/App.navigation.test.tsx src/theme/a11y-guards.test.ts
/opt/homebrew/bin/git commit -m "feat: wire the admin console to the new shell with hash routing

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Selo do sino com `--on-tone` (dashboard e admin)

**Files:**
- Modify: `src/shell/NotificationCenter.tsx` (o `color: "white"` do selo)
- Create: `src/shell/NotificationCenter.badge.test.ts`

- [ ] **Step 1: Escrever a guarda, que vai falhar.** `src/shell/NotificationCenter.badge.test.ts`:

```ts
// O selo do sino usa --on-tone: branco fixo fica ilegível sobre amarelo no
// alto contraste e no escuro.
import { describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const code = readFileSync(join(dirname(fileURLToPath(import.meta.url)), "NotificationCenter.tsx"), "utf8");

describe("selo do sino", () => {
  it("não fixa branco", () => {
    expect(code).not.toMatch(/color:\s*"white"/);
    expect(code).toMatch(/color:\s*"var\(--on-tone\)"/);
  });
});
```

  Run: `npx vitest run src/shell/NotificationCenter.badge.test.ts`
  Expected: FAIL.

- [ ] **Step 2: Trocar a cor.** No `NotificationCenter.tsx`, troque `color: "white",` por `color: "var(--on-tone)",`.

  Run: `npx vitest run src/shell/NotificationCenter.badge.test.ts && npx vitest run 2>&1 | tail -3`
  Expected: PASS e a suíte verde.

- [ ] **Step 3: Commit** (em cada app).

```bash
/opt/homebrew/bin/git add src/shell/NotificationCenter.tsx src/shell/NotificationCenter.badge.test.ts
/opt/homebrew/bin/git commit -m "style: use on-tone for the notification badge text

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Capturas "depois", aprovação, merge e publicação

- [ ] **Step 1: Subir os worktrees.**

```bash
VITE_API_PROXY_TARGET=http://localhost:3030 npx vite --port 5185 --strictPort --host 127.0.0.1
VITE_API_PROXY_TARGET=http://localhost:3030 npx vite --port 5186 --strictPort --host 127.0.0.1
```

  O primeiro sobe o dashboard, o segundo o admin, cada um em segundo plano no seu worktree.

- [ ] **Step 2: Adaptar o script de captura à navegação nova.**
  - Navegue por hash, e não por clique: `page.goto(base + "/dashboard/#/" + id)` (no admin, `/admin/#/…`).
  - Telas do dashboard:

    | Arquivo | Hash |
    |---|---|
    | `inicio` | `#/` |
    | `grupo-atendimento` | `#/grupo/atendimento` |
    | `minha-agenda` | `#/my-agenda` |
    | `equipe` | `#/team` |
    | `acessibilidade` | `#/acessibilidade` |

  - Para cada largura (768, 1024 e 1280) e, só em 1280, para cada tema, use `page.addInitScript(t => localStorage.setItem("rotasaude:theme", t), tema)` antes do `goto`.
  - Também em 768, capture a gaveta aberta (clique em "Menu").
  - No admin, capture `#/` e `#/cities`.
  - Salve em `rota-saude/evidencias/2026-10-12-panel-shell/depois/`.

- [ ] **Step 3: Conferir cada captura.**
  - nada sobreposto no topo em nenhuma largura;
  - cidade, período e sessão de assinatura visíveis na faixa;
  - trilho só a partir de 1024px;
  - a gaveta em 768px;
  - 2, 3 e 4 cards por linha;
  - o relógio só em 1280px;
  - o alto contraste legível, inclusive o selo do sino;
  - a Figtree no tema Rota Saúde.

- [ ] **Step 4: Conferir o fluxo no navegador.** Navegue pelo Início, depois por um grupo, depois por uma tela, e use o Voltar duas vezes; o esperado é chegar ao grupo e depois ao Início. Também confira:
  - recarregar a página mantém a tela;
  - colar `#/team` com o usuário profissional (`profissional@curitiba.demo`) leva ao Início;
  - Alt+1, Alt+2 e Alt+3 movem o foco.

  Registre tudo no ledger.

- [ ] **Step 5: Revisão final.** Faça a revisão do branch inteiro com o revisor mais capaz (veja a skill executing-plans), corrija os achados Critical e Important com TDD e grave as decisões no ledger.

- [ ] **Step 6: Pedir a aprovação do usuário.** Monte a comparação lado a lado (`comparacao.html`) e **espere a aprovação**.

- [ ] **Step 7: Merge e push, um repo por vez.** A ordem é contracts (com `git push origin tokens-v1.1.0`), depois dashboard, depois admin, e por fim docs (este plano). Em cada repo:
  1. confira `origin/main..main`;
  2. faça o merge `--ff-only` de `ui/panel-shell`;
  3. rode `push`;
  4. remova o worktree com `--force` (o `node_modules` dele não é rastreado);
  5. apague o branch.

- [ ] **Step 8: Conferir os containers e avisar.** Confira que os containers de dev (5175 e 5174) servem o shell novo. Avise a sessão do módulo 19 e a "Próximos módulos" com os hashes publicados.
