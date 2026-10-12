# Ciclo de interface — subprojeto 3: tabelas em cartões e layouts em 768px — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Abaixo de 1024px, cada linha do `DataTable` vira um cartão (título, pares rótulo/valor e ações), e os layouts largos (Visões gerais, Editor de protocolo, formulários do admin) empilham. Tudo só com CSS e atributos, sem mudar a funcionalidade nem o DOM que os testes consultam.

**Architecture:**
- **`DataTable`:** ganha classes (`dt`, `dt-row`, `dt-head`, `dt-cell`) e atributos (`data-label`, `data-align`, `data-card`). Os estilos de célula que eram inline (`nowrap`, reticências e alinhamento) passam para o `datatable.css`, para que o modo cartão possa sobrescrevê-los sem `!important`.
- **Modo cartão:** um único bloco `@media (max-width: 1023px)`. A linha vira `flex` com quebra; o título, as ações e o `card:"hide"` são resolvidos por seletor, com `:has()` para detectar ações.
- **Layouts largos:** recebem `className` e regras em `src/theme/responsive.css`.

**Tech Stack:** React 18, Vite 5, Vitest 2 e CSS (`:has()`, `attr()` em `content`).

**Spec:** `docs/superpowers/specs/2026-10-12-interface-tables-as-cards-design.md`

## Global Constraints

- **Só interface.** Testes anteriores ao ciclo passam **sem asserção editada**.
- **O DOM consultado pelos testes não muda:**
  - os papéis `table`, `row`, `columnheader` e `cell`;
  - o cabeçalho como primeira linha;
  - as células como filhas diretas da linha;
  - o `textContent` exato de linhas e células;
  - o `style.background` inline da linha (`transparent` ou `rowStyle`);
  - a linha `BUTTON` só com `onRowClick`;
  - sem `role="table"` quando a lista está vazia.
- **Rótulo do cartão:** só por `::before { content: attr(data-label) }`, nunca como texto no DOM.
- **`!important`:** permitido só em `display` (a linha do `DataTable` em modo cartão) e em `grid-template-columns` (as grades inline das telas). Sempre dentro de `@media (max-width: 1023px)`. Há um teste que garante isso.
- **Fora do `@media`:** só as regras de `.dt-cell`, que substituem o que antes era estilo inline da célula.
- **Commits:** Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Git:** use `/opt/homebrew/bin/git` e caminhos explícitos no `add`.
- **Worktrees:** `ui/tables-as-cards` em `apps/dashboard/.claude/worktrees/ui-cards` e `apps/admin/.claude/worktrees/ui-cards`, com `npm ci`.

## Review Focus

1. **Número alinhado à direita num par do cartão.** Ex.: a contagem "13" na Classificação. A pessoa espera o rótulo e o valor alinhados juntos à esquerda. O `data-align` só vale fora do `@media`. A Task 1 testa os atributos, e as capturas da Task 4 conferem o visual.
2. **Linha clicável (`onRowClick`) em modo cartão.** Ex.: Protocolos e Classificação. A pessoa espera que o cartão inteiro seja clicável e mostre o foco. O `<button role="row">` vira o cartão. A Task 4 confere no navegador o clique e o foco por teclado.
3. **Tabela cuja primeira coluna é um controle.** Ex.: a caixa de seleção do CNES. A pessoa espera um título legível, não a caixa de seleção como título. A Task 2 testa o `card:"title"` e o `card:"actions"`.
4. **Botões de várias colunas.** Ex.: Equipe, com 6 colunas de botões. A pessoa espera os botões lado a lado no rodapé do cartão, quebrando linha, e não um por linha ocupando a largura toda. As ações são `flex: 0 0 auto` dentro do cartão em `flex-wrap`. A Task 4 confere nas capturas.
5. **Célula com "—" ou vazia.** A pessoa espera ver o rótulo com "—", em vez de o par sumir. Nenhuma regra esconde par vazio, e as capturas da Task 4 conferem isso.

---

## File Structure

| Arquivo | Onde | Responsabilidade |
|---|---|---|
| `src/components/DataTable.tsx` | dashboard e admin | Classes e atributos; a célula perde o estilo inline de overflow e alinhamento |
| `src/components/datatable.css` | idêntico nos dois apps | Base `.dt-cell` e o modo cartão |
| `src/components/DataTable.cards.test.tsx` | idêntico nos dois apps | Atributos, DOM e `textContent` |
| `src/theme/css-guards.test.ts` | idêntico nos dois apps | Regras de `!important` e `@media` nos dois CSS |
| `src/theme/responsive.css` | por app | Classes dos layouts largos |
| `src/theme/global.css` | por app | Acrescenta `@import "./responsive.css";` |
| `src/theme/layout-classes.test.ts` | por app | As classes estão no código das telas |
| `src/components/KpiGrid.tsx` | só dashboard | `className="kpi-grid"` quando há `columns` |
| `src/modules/Overview.tsx` | dashboard e admin | `className` nas grades |
| `src/modules/ProtocolEditor.tsx` | dashboard | `className` |
| `src/modules/Cnes.tsx` | dashboard | `card` nas colunas |
| `src/modules/setup/ProvisionCity.tsx`, `src/modules/setup/MfaEnroll.tsx` | admin | `className` |

---

### Task 0: Preparação

- [ ] **Step 1: Criar os worktrees.** Confira antes que `origin/main..main` está vazio nos dois repos.

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude && G=/opt/homebrew/bin/git
$G -C apps/dashboard worktree add -q .claude/worktrees/ui-cards -b ui/tables-as-cards origin/main && (cd apps/dashboard/.claude/worktrees/ui-cards && npm ci --silent)
$G -C apps/admin worktree add -q .claude/worktrees/ui-cards -b ui/tables-as-cards origin/main && (cd apps/admin/.claude/worktrees/ui-cards && npm ci --silent)
```

- [ ] **Step 2: Medir a linha de base.** Rode `npx vitest run` em cada worktree. O esperado é o dashboard com 2314 testes e o admin com 279, todos verdes.

- [ ] **Step 3: Conferir se algum teste lê o estilo inline da célula.**

```bash
grep -rnE "role=\"cell\"|getAllByRole\(\"cell\"\)|\.cells?\b" src --include=*.test.tsx | grep -E "style\.(textAlign|whiteSpace|overflow)" || echo "nenhum"
```

  O esperado é `nenhum`. Se algum teste ler esses estilos, pare: mover o estilo para CSS quebraria o teste, e a decisão é do usuário.

- [ ] **Step 4: Escrever o script de varredura e rodar o "antes".**
  - Arquivo: `scratchpad/capture/sweep.mjs`.
  - O script entra como admin municipal no dashboard (`POST /session`) ou como operador no admin (TOTP).
  - Para cada largura (768, 1024 e 1280px) e cada id de tela do menu, ele abre `#/<id>`, espera 1,8 s e registra `{ id, w, cw: clientWidth, sw: scrollWidth, h1 }`.
  - Os ids do dashboard são os 34 de `NAV_GROUPS`; os do admin são os 6 do menu. O script lê os ids de `src/shell/modules.ts` com uma expressão regular sobre `id: "…"`.
  - Rode contra os containers de dev (5175 e 5174), que ainda servem o `main`, e salve o resultado em `rota-saude/evidencias/2026-10-12-tables-as-cards/sweep-antes.json`.
  - No mesmo script, com `CAPTURE=1`, tire as capturas em 768px das telas a seguir:

    | App | Telas (hash) |
    |---|---|
    | dashboard | `team`, `attendance` (a tela de Atendimento contém Pedidos e Fila da unidade), `protocols`, `queues`, `cnes`, `overview`, `protocol-editor` |
    | admin | `cities`, `city_production` |

    Salve em `…/antes/`.

---

### Task 1: `DataTable` em modo cartão (dashboard e admin)

**Files:**
- Modify: `src/components/DataTable.tsx`
- Create: `src/components/datatable.css`, `src/components/DataTable.cards.test.tsx`, `src/theme/css-guards.test.ts`

**Interfaces:**
- Produces: `Column<T>` ganha `card?: "title" | "actions" | "hide"`. As classes são `dt`, `dt-row`, `dt-head` e `dt-cell`, e os atributos `data-label`, `data-align` e `data-card`.

- [ ] **Step 1: Escrever os testes, que vão falhar.**

`src/components/DataTable.cards.test.tsx` (idêntico nos dois apps):
```tsx
// Modo cartão (spec do subprojeto 3, §3): só classes e atributos. O DOM
// consultado pelos testes (roles, filhos diretos, textContent) não muda.
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen } from "@testing-library/react";
import { DataTable, type Column } from "./DataTable";

afterEach(cleanup);

type Row = { id: string; nome: string; n: number };
const rows: Row[] = [ { id: "a", nome: "Ana", n: 3 }, { id: "b", nome: "Bia", n: 7 } ];
const cols: Column<Row>[] = [
  { label: "Nome", render: (r) => r.nome },
  { label: "Total", align: "right", render: (r) => r.n },
  { label: "Ações", card: "actions", render: () => <button type="button">Abrir</button> },
  { label: "Interno", card: "hide", render: () => "x" }
];

describe("DataTable em modo cartão", () => {
  it("classes na tabela, no cabeçalho e nas linhas", () => {
    render(<DataTable cols={cols} rows={rows} rowKey={(r) => r.id} />);
    const [ head, ...data ] = screen.getAllByRole("row");
    expect(screen.getByRole("table").className).toBe("dt");
    expect(head.classList.contains("dt-row") && head.classList.contains("dt-head")).toBe(true);
    data.forEach((r) => expect(r.className).toBe("dt-row"));
  });

  it("cada célula é dt-cell, filha direta da linha, com data-label", () => {
    render(<DataTable cols={cols} rows={rows} rowKey={(r) => r.id} />);
    const row = screen.getAllByRole("row")[1];
    const cells = screen.getAllByRole("cell").slice(0, 4);
    cells.forEach((c) => { expect(c.parentElement).toBe(row); expect(c.className).toBe("dt-cell"); });
    expect(cells.map((c) => c.getAttribute("data-label"))).toEqual([ "Nome", "Total", "Ações", "Interno" ]);
  });

  it("data-align só fora do padrão e data-card só quando declarado", () => {
    render(<DataTable cols={cols} rows={rows} rowKey={(r) => r.id} />);
    const cells = screen.getAllByRole("cell").slice(0, 4);
    expect(cells.map((c) => c.getAttribute("data-align"))).toEqual([ null, "right", null, null ]);
    expect(cells.map((c) => c.getAttribute("data-card"))).toEqual([ null, null, "actions", "hide" ]);
  });

  it("o texto de linhas e células não muda (o rótulo é só CSS)", () => {
    render(<DataTable cols={cols} rows={rows} rowKey={(r) => r.id} />);
    expect(screen.getAllByRole("row")[1].textContent).toBe("Ana3Abrirx");
    expect(screen.getAllByRole("cell")[0].textContent).toBe("Ana");
  });

  it("cabeçalho sem data-label; linha só é BUTTON com onRowClick", () => {
    const { rerender } = render(<DataTable cols={cols} rows={rows} rowKey={(r) => r.id} />);
    screen.getAllByRole("columnheader").forEach((h) => expect(h.hasAttribute("data-label")).toBe(false));
    expect(screen.getAllByRole("row")[1].tagName).toBe("DIV");
    rerender(<DataTable cols={cols} rows={rows} rowKey={(r) => r.id} onRowClick={vi.fn()} />);
    expect(screen.getAllByRole("row")[1].tagName).toBe("BUTTON");
    expect(screen.getAllByRole("row")[1].className).toBe("dt-row");
  });

  it("vazio continua sem role=table", () => {
    render(<DataTable cols={cols} rows={[]} rowKey={(r) => r.id} empty="nada" />);
    expect(screen.queryByRole("table")).toBeNull();
  });
});
```

`src/theme/css-guards.test.ts` (idêntico nos dois apps; o `responsive.css` nasce na Task 3, então o teste dele usa `it.skipIf` até lá):
```ts
// Guardas dos CSS do ciclo (spec do subprojeto 3): !important só em display
// e grid-template-columns, e só dentro de @media (max-width: 1023px); fora
// do @media, só as regras de .dt-cell (antes estilo inline da célula).
import { describe, expect, it } from "vitest";
import { existsSync, readFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const src = join(dirname(fileURLToPath(import.meta.url)), "..");
const strip = (css: string) => css.replace(/\/\*[\s\S]*?\*\//g, "");

function splitMedia(css: string): { inside: string; outside: string } {
  const at = css.indexOf("@media (max-width: 1023px)");
  if (at < 0) return { inside: "", outside: css };
  let depth = 0;
  let end = -1;
  for (let i = css.indexOf("{", at); i < css.length; i++) {
    if (css[i] === "{") depth++;
    if (css[i] === "}") { depth--; if (depth === 0) { end = i; break; } }
  }
  return { inside: css.slice(at, end + 1), outside: css.slice(0, at) + css.slice(end + 1) };
}

function check(file: string, outsideSelectors: RegExp | null) {
  const css = strip(readFileSync(join(src, file), "utf8"));
  const { inside, outside } = splitMedia(css);
  expect((css.match(/@media/g) ?? []).length, "um único @media").toBe(1);
  const importants = css.match(/[\w-]+\s*:[^;{}]*!important/g) ?? [];
  importants.forEach((d) => expect(d, d).toMatch(/^(display|grid-template-columns)\s*:/));
  expect(outside, "nada de !important fora do @media").not.toMatch(/!important/);
  const outsideRules = outside.match(/[^{}]+(?=\{)/g)?.map((s) => s.trim()).filter(Boolean) ?? [];
  if (outsideSelectors) outsideRules.forEach((sel) => expect(sel, sel).toMatch(outsideSelectors));
  else expect(outsideRules).toEqual([]);
  expect(inside).not.toMatch(/\.dt-head[^{]*\{[^}]*display\s*:\s*none/);
}

describe("CSS do ciclo de interface", () => {
  it("datatable.css", () => check("components/datatable.css", /^\.dt-cell(\[data-align="(right|center)"\])?$/));
  it.skipIf(!existsSync(join(src, "theme/responsive.css")))("responsive.css", () => check("theme/responsive.css", null));
});
```

  Run: `npx vitest run src/components/DataTable.cards.test.tsx src/theme/css-guards.test.ts`
  Expected: FAIL. As classes são `""`/`null` e o `datatable.css` não existe (ENOENT).

- [ ] **Step 2: Criar `src/components/datatable.css`** (idêntico nos dois apps).

```css
/* DataTable (spec do subprojeto 3, §3). Base: o que antes era estilo inline
   da célula, agora em classe para o modo cartão poder trocar sem !important. */
.dt-cell { min-width: 0; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.dt-cell[data-align="right"] { text-align: right; }
.dt-cell[data-align="center"] { text-align: center; }

/* Abaixo de 1024px cada linha vira um cartão: título, pares rótulo/valor e
   ações. Mesmo DOM (roles, filhas diretas, textContent); o rótulo é CSS. */
@media (max-width: 1023px) {
  .dt { display: grid; gap: 12px; }

  /* Cabeçalho fora da tela, mas lido pelo leitor de tela. */
  .dt > .dt-head {
    position: absolute; width: 1px; height: 1px; margin: -1px; padding: 0; overflow: hidden;
    clip: rect(0 0 0 0); clip-path: inset(50%); white-space: nowrap; border: 0;
  }

  .dt > .dt-row:not(.dt-head) {
    display: flex !important;
    flex-wrap: wrap;
    align-items: flex-start;
    gap: 10px 16px;
    padding: 12px 14px;
    border: 1px solid var(--rule);
    border-radius: var(--radius-panel);
    text-align: left;
  }

  /* Par rótulo/valor: duas por linha. */
  .dt-row .dt-cell {
    flex: 1 1 calc(50% - 8px);
    white-space: normal; overflow: visible; text-overflow: clip; text-align: left;
    overflow-wrap: anywhere;
  }
  .dt-row .dt-cell::before {
    content: attr(data-label);
    display: block; margin-bottom: 2px;
    font-family: var(--font-mono); font-size: 11px; letter-spacing: .04em;
    text-transform: uppercase; color: var(--ink2);
  }
  .dt-row .dt-cell[data-label=""]::before { content: none; }

  /* Título: a primeira célula, ou a coluna marcada com card="title". */
  .dt-row:not(:has(> [data-card="title"])) > .dt-cell:first-child:not([data-card]),
  .dt-row > .dt-cell[data-card="title"] {
    flex-basis: 100%; font-size: 15px; font-weight: 700;
  }
  .dt-row:not(:has(> [data-card="title"])) > .dt-cell:first-child:not([data-card])::before,
  .dt-row > .dt-cell[data-card="title"]::before { content: none; }

  /* Ações: célula com controle (que não seja o título), ou card="actions".
     Ficam no fim, lado a lado, quebrando linha. */
  .dt-row > .dt-cell:not(:first-child):not([data-card]):has(button, input, a),
  .dt-row:has(> [data-card="title"]) > .dt-cell:first-child:not([data-card]):has(button, input, a),
  .dt-row > .dt-cell[data-card="actions"] {
    flex: 0 0 auto; order: 99; display: flex; flex-wrap: wrap; align-items: center; gap: 8px;
  }
  .dt-row > .dt-cell:not(:first-child):not([data-card]):has(button, input, a)::before,
  .dt-row:has(> [data-card="title"]) > .dt-cell:first-child:not([data-card]):has(button, input, a)::before,
  .dt-row > .dt-cell[data-card="actions"]::before { content: none; }
  .dt-row > .dt-cell[data-card="actions"] button,
  .dt-row > .dt-cell:has(button) button { min-height: 44px; }

  .dt-row > .dt-cell[data-card="hide"] { display: none; }
}
```

- [ ] **Step 3: Mudar o `DataTable.tsx`** (os dois apps; os blocos abaixo valem para ambos).
  - **Import:** acrescente `import "./datatable.css";` depois do import de tipos.
  - **Tipo:** no `Column<T>`, acrescente depois de `render` o campo `card?: "title" | "actions" | "hide";`, com o comentário `// Modo cartão (<1024px): papel da coluna quando a regra automática não serve.`.
  - **Tabela:** `<div role="table" style={{ width: "100%" }}>` vira `<div role="table" className="dt" style={{ width: "100%" }}>`.
  - **Cabeçalho:** a linha `role="row"` com `className="mono"` passa a ter `className="mono dt-row dt-head"`. O teste confere as classes por `classList`, então o `mono` continua.

  - **Célula:** troque a célula de dado

```tsx
<span key={ci} role="cell" style={{ textAlign: c.align || "left", minWidth: 0, overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>
```

  por:

```tsx
<span key={ci} role="cell" className="dt-cell" data-label={c.label}
  data-align={c.align && c.align !== "left" ? c.align : undefined} data-card={c.card}>
```

  - **Linha:** em cada linha de dado (`<button role="row">`/`<div role="row">` no dashboard, `<Row role="row">` no admin), acrescente `className="dt-row"`.

- [ ] **Step 4: Rodar os testes novos e a suíte.**

  Run: `npx vitest run src/components/DataTable.cards.test.tsx src/theme/css-guards.test.ts && npx vitest run 2>&1 | tail -4 && npx tsc --noEmit -p .`
  Expected: os testes novos passam (o `responsive.css` aparece como `skipped`), e a suíte inteira fica verde, sem teste antigo editado. O `DataTable.test` original do dashboard continua passando.

- [ ] **Step 5: Commit** (em cada app).

```bash
/opt/homebrew/bin/git add src/components/DataTable.tsx src/components/datatable.css src/components/DataTable.cards.test.tsx src/theme/css-guards.test.ts
/opt/homebrew/bin/git commit -m "feat: render DataTable rows as cards below 1024px

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Ajuste do cartão no CNES (dashboard)

**Files:**
- Modify: `src/modules/Cnes.tsx` (a tabela de propostas, em torno da linha 115)
- Create: `src/modules/Cnes.cards.test.tsx`

- [ ] **Step 1: Escrever o teste, que vai falhar.** Use o mesmo `render` do teste existente do CNES. Antes, rode `ls src/modules/Cnes*.test.tsx` e copie dele o `beforeEach`, os mocks de API e o fixture com pelo menos uma proposta.

```tsx
it("propostas: caixa de seleção vai para ações e o tipo é o título do cartão", async () => {
  // render como no teste existente, com uma proposta
  const box = await screen.findByRole("checkbox", { name: /^Selecionar: / });
  const cell = box.closest('[role="cell"]')!;
  expect(cell.getAttribute("data-card")).toBe("actions");
  const row = cell.parentElement!;
  const title = row.querySelector('[data-card="title"]');
  expect(title?.getAttribute("data-label")).toBe("Tipo");
});
```

  Run: `npx vitest run src/modules/Cnes.cards.test.tsx`
  Expected: FAIL, porque o `data-card` é `null`.

- [ ] **Step 2: Marcar as colunas.** Na tabela de propostas:
  - a primeira coluna `{ label: "", w: "32px", render: … }` ganha `card: "actions",`;
  - a coluna `{ label: "Tipo", … }` ganha `card: "title",`.

  Run: `npx vitest run src/modules/Cnes.cards.test.tsx && npx vitest run 2>&1 | tail -3`
  Expected: PASS e a suíte verde.

- [ ] **Step 3: Commit.**

```bash
/opt/homebrew/bin/git add src/modules/Cnes.tsx src/modules/Cnes.cards.test.tsx
/opt/homebrew/bin/git commit -m "style: set card roles for the CNES proposals table

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Layouts largos empilham abaixo de 1024px

**Files:**
- Create: `src/theme/responsive.css` e `src/theme/layout-classes.test.ts` (por app)
- Modify: `src/theme/global.css` (por app)
- Modify (dashboard): `src/components/KpiGrid.tsx`, `src/modules/Overview.tsx`, `src/modules/ProtocolEditor.tsx`
- Modify (admin): `src/modules/Overview.tsx`, `src/modules/setup/ProvisionCity.tsx`, `src/modules/setup/MfaEnroll.tsx`

- [ ] **Step 1: Escrever o teste de classes, que vai falhar.** É uma guarda de fonte, igual às outras guardas do ciclo.

  Ruling: o teste lê o código das telas em vez de renderizá-las. As Visões gerais precisam de muitos mocks para renderizar, e a garantia é a mesma: a classe está no elemento cuja grade inline ela sobrescreve. O visual é conferido na Task 4.

`src/theme/layout-classes.test.ts` do **dashboard**:
```ts
import { describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const src = join(dirname(fileURLToPath(import.meta.url)), "..");
const read = (f: string) => readFileSync(join(src, f), "utf8");

describe("layouts largos têm classe para 768px", () => {
  it.each([
    [ "components/KpiGrid.tsx", /className=\{columns \? "kpi-grid" : undefined\}/ ],
    [ "modules/Overview.tsx", /className="ov-split"[^>]*gridTemplateColumns: "1\.5fr 1fr"/ ],
    [ "modules/Overview.tsx", /className="ov-split"[^>]*gridTemplateColumns: "1fr 1\.4fr"/ ],
    [ "modules/Overview.tsx", /className="ov-kv3"/ ],
    [ "modules/Overview.tsx", /className="ov-event"/ ],
    [ "modules/ProtocolEditor.tsx", /className="pe-split"/ ]
  ])("%s %s", (file, re) => expect(read(file)).toMatch(re));

  it("global.css importa responsive.css logo depois dos tokens", () => {
    expect(read("theme/global.css")).toMatch(/@import "\.\/vendor\/tokens\.css";\s*@import "\.\/responsive\.css";/);
  });
});
```

`src/theme/layout-classes.test.ts` do **admin** (mesmo cabeçalho de imports e `read`):
```ts
describe("layouts largos têm classe para 768px", () => {
  it.each([
    [ "modules/Overview.tsx", /className="kpi-grid"/ ],
    [ "modules/Overview.tsx", /className="ov-split"[^>]*gridTemplateColumns: "1\.5fr 1fr"/ ],
    [ "modules/Overview.tsx", /className="ov-split"[^>]*gridTemplateColumns: "1fr 1\.4fr"/ ],
    [ "modules/Overview.tsx", /className="ov-kv3"/ ],
    [ "modules/Overview.tsx", /className="ov-event"/ ],
    [ "modules/setup/ProvisionCity.tsx", /className="form-2col"/ ],
    [ "modules/setup/MfaEnroll.tsx", /className="form-2col"/ ]
  ])("%s %s", (file, re) => expect(read(file)).toMatch(re));

  it("global.css importa responsive.css logo depois dos tokens", () => {
    expect(read("theme/global.css")).toMatch(/@import "\.\/vendor\/tokens\.css";\s*@import "\.\/responsive\.css";/);
  });
});
```

  Run: `npx vitest run src/theme/layout-classes.test.ts`
  Expected: FAIL.

- [ ] **Step 2: Criar `src/theme/responsive.css`** (o mesmo conteúdo nos dois apps; uma classe que não existe num app é inofensiva).

```css
/* Layouts largos abaixo de 1024px (spec do subprojeto 3, §4). As grades são
   inline nas telas; por isso grid-template-columns leva !important aqui. */
@media (max-width: 1023px) {
  .kpi-grid { grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)) !important; }
  .ov-split { grid-template-columns: minmax(0, 1fr) !important; }
  .ov-kv3 { grid-template-columns: repeat(2, minmax(0, 1fr)) !important; }
  .ov-event { grid-template-columns: 60px minmax(0, 1fr) !important; }
  .ov-event > :nth-child(3) { grid-column: 1 / -1; }
  .pe-split { grid-template-columns: minmax(0, 1fr) !important; }
  .form-2col { grid-template-columns: minmax(0, 1fr) !important; }
}
```

  No `global.css`, logo depois de `@import "./vendor/tokens.css";`, acrescente `@import "./responsive.css";`.

- [ ] **Step 3: Pôr as classes.**

  **Dashboard:**
  - **`KpiGrid.tsx`:** no `<div>` interno com `gridTemplateColumns`, acrescente `className={columns ? "kpi-grid" : undefined}`.
  - **`Overview.tsx`:**
    - linha 42: `<div style={{ display: "grid", gridTemplateColumns: "1.5fr 1fr", gap: 14 }}>` vira `<div className="ov-split" style={{ display: "grid", gridTemplateColumns: "1.5fr 1fr", gap: 14 }}>`;
    - linha 46: o mesmo, com `"1fr 1.4fr"`;
    - linha 213: no elemento cujo style tem `gridTemplateColumns: "repeat(3, 1fr)"`, acrescente `className="ov-kv3"`;
    - linha 347: no elemento com `gridTemplateColumns: "60px 1.1fr 1.4fr"`, acrescente `className="ov-event"`.

      Se esse elemento já tiver `className`, junte os valores com espaço, e o teste precisa aceitar isso: troque o padrão por `/className="[^"]*ov-event/`.
  - **`ProtocolEditor.tsx`**, linha 95: acrescente `className="pe-split"`.

  **Admin:**
  - **`Overview.tsx`:**
    - linha 83 (o `KpiGrid5`): `className="kpi-grid"`;
    - linhas 38 e 42: `className="ov-split"`;
    - linha 207: `ov-kv3`;
    - linha 340: `ov-event`.
  - **`setup/ProvisionCity.tsx`**, linha 100: `className="form-2col"`.
  - **`setup/MfaEnroll.tsx`**, linha 96: `className="form-2col"`.

  Para os testes de `ov-split`, que exigem `className` antes de `gridTemplateColumns` no mesmo elemento, ponha o `className` logo antes do `style`.

- [ ] **Step 4: Rodar.**

  Run: `npx vitest run src/theme/ && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit -p . && npm run build`
  Expected: o `layout-classes` passa, o `css-guards` agora roda as duas guardas (sem `skipped`) e passa, a suíte fica verde e o build fica limpo.

- [ ] **Step 5: Commit** (em cada app). No dashboard, o `git add` inclui `src/theme/responsive.css src/theme/global.css src/theme/layout-classes.test.ts src/components/KpiGrid.tsx src/modules/Overview.tsx src/modules/ProtocolEditor.tsx`. No admin, inclui `src/theme/responsive.css src/theme/global.css src/theme/layout-classes.test.ts src/modules/Overview.tsx src/modules/setup/ProvisionCity.tsx src/modules/setup/MfaEnroll.tsx`.

```bash
/opt/homebrew/bin/git commit -m "style: stack wide panel layouts below 1024px

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Varredura, capturas, ajustes por tela, revisão e publicação

- [ ] **Step 1: Subir os worktrees.** Rode `VITE_API_PROXY_TARGET=http://localhost:3030 npx vite --port 5185 --strictPort --host 127.0.0.1` no worktree do dashboard e `--port 5186` no do admin.

- [ ] **Step 2: Varredura e capturas "depois".**
  - Rode o `sweep.mjs` contra 5185 e 5186, com `CAPTURE=1`.
  - Salve em `…/sweep-depois.json` e `…/depois/`.
  - **Critério:** `sw === cw` em toda tela verificada, nas três larguras.
  - Liste à parte as telas cujo `h1` não carregou ou que mostram estado vazio ou erro: essas ficam marcadas como "não verificadas".

- [ ] **Step 3: Conferir as capturas de 768px** contra o Review Focus:
  - título certo;
  - pares legíveis, alinhados à esquerda;
  - botões lado a lado no rodapé;
  - o "—" aparecendo;
  - cartão clicável em Protocolos e Classificação, com o foco visível ao navegar por Tab.

  Para cada tabela em que o título ou as ações ficarem errados, acrescente `card` na coluna, com teste da mesma forma da Task 2, e registre a decisão no ledger.

- [ ] **Step 4: Revisão final.** Faça a revisão do branch inteiro com o revisor mais capaz, corrija os achados Critical e Important com TDD e registre-os no ledger.

- [ ] **Step 5: Aprovação.** Monte o `comparacao.html` (antes e depois em 768px, mais a tabela da varredura) e **espere a aprovação** do usuário.

- [ ] **Step 6: Merge e push.**
  - Repos, nesta ordem: dashboard, admin e docs (este plano).
  - Em cada repo: confira `origin/main..main`, faça o merge `--ff-only`, o `push`, remova o worktree com `--force` e apague o branch.
  - Em seguida, confira os containers de dev. Não há dependência nova, então não precisa reiniciar.
  - Por fim, avise as sessões "Próximos módulos" e "MVP Module 19" do que muda: o `Column.card` e as classes dos layouts.
