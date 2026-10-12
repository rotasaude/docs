# Ciclo de interface — subprojeto 5: maintenance — design

**Data:** 2026-10-12
**Status:** em revisão
**Afeta:** só `apps/maintenance`. O tema `rota` e a fonte Figtree já estão em `contracts` (`tokens-v1.1.0`).

**Spec do ciclo:** `docs/superpowers/specs/2026-10-12-interface-cycle-foundation-design.md`. Valem duas decisões de lá:

- **Decisão 13:** identidade própria `rota`, com selo "Manutenção", acessibilidade AA sem eMAG, e a faixa de ambiente nunca pode se confundir com o tema.
- **Decisão 16:** navegação só por trilho, sem cards, mantendo as abas no detalhe da cidade.

**Regra do ciclo:** só interface. Telas, dados, permissões e chamadas ao api não mudam.

## 1. Ponto de partida

- **Cinco telas** (Cidades, Medicamentos, Mantenedores, Tokens, Auditoria). A tela ativa fica em `useState` em `screens/Shell.tsx`, sem roteador nem hash.
- **Detalhe da cidade:** substitui a lista de cidades e tem 8 abas, também em `useState`.
- **Cabeçalho atual:** uma linha só, com os botões de navegação, o e-mail e "sair". Não há marca nem selo.
- **Tema antigo:** `src/theme/tokens.ts` e `global.css` são cópias do console, com oklch e a fonte Geist importada via fontsource. O corpo tem 13px.
- **Cores fixas:**
  - `toneColor()` devolve oklch fixo e alimenta o `Tag`;
  - `Button` fixa `color: "white"`;
  - o estado desabilitado é mostrado só por opacidade.
  - Fora isso, componentes e telas usam `var(--x)` sem reserva.
- **Faixa de ambiente:**
  - `EnvBanner` é renderizada pelo `App` acima de tudo, com `role="status"` e o texto exato "DEVELOPMENT" ou "STAGING";
  - as cores ficam em `--env-dev-*` e `--env-stg-*`, no `global.css`;
  - o azul de development fica perto do `--info` do tema `rota`.
- **Tabelas:**
  - são `<table>` de verdade: o `DataTable` e as tabelas feitas à mão em Auditoria e ProtocolsTab;
  - cinco delas têm de 6 a 7 colunas, e nenhuma tem rolagem.
- **Testes:**
  - Vitest com 29 arquivos.
  - Nenhum teste confere estilo.
  - O que os testes exigem do DOM:
    - nav com `button` de nome exato e `aria-current="page"`;
    - um único `role="status"` nas telas;
    - `getByRole("table")` com uma única ocorrência, linhas `<tr>` e `.closest("tr")`.
  - O e2e Playwright (`e2e/maintenance.spec.ts`) exige ainda:
    - o e-mail dentro de um `<header>`;
    - linhas `tr`;
    - `dt`/`dd`;
    - a `section` do segredo com o primeiro `<p>` sendo o segredo;
    - um único `<code>` no convite.
- **Dev:** em `http://maintenance.localhost:5177` (porta 5173 no container, base `/`). O login é `dev@local` / `dev-password` com TOTP fixo da semente.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | **Tabelas em telas estreitas:** rolagem lateral dentro de um contêiner, com dica visual. Não viram cartões. |
| 2 | **Faixa de development em roxo `#6b2fa3`** com texto branco (7,6:1). Staging continua âmbar (`#e0952a`) com texto `#2b1a00`. |

## 3. Tokens, tema, shell e faixa

### 3.1 Tokens e tema

- **Cópia dos tokens:**
  - `scripts/tokens-sync.mjs` (igual ao dos outros apps);
  - `src/theme/vendor/` com `tokens-v1.1.0`;
  - `src/theme/vendor.lock.test.ts`.
- **Tema fixo:** `<html lang="pt-BR" data-theme="rota">` no `index.html`. Não há seletor de tema nem alto contraste.
- **`src/theme/global.css` reescrito:**
  - `@import "./vendor/tokens.css"`;
  - reset;
  - `body` com `--bg`, `--ink`, `--font-sans` (Figtree) e 13px;
  - reset de `button`;
  - `:focus-visible` com contorno de 3px em `--focus`;
  - `a` em `--accent`;
  - `.mono`;
  - as variáveis da faixa (`--env-dev-bg: #6b2fa3; --env-dev-ink: #ffffff; --env-stg-bg: #e0952a; --env-stg-ink: #2b1a00`) e a classe `.env-banner`, que segue como hoje;
  - `.table-scroll` (seção 4.1).
- **Removidos:** `src/theme/tokens.ts` e as dependências `@fontsource-variable/geist*`. O `toneColor()` é reescrito para devolver `var(--ok)`, `var(--ok-bg)` e os demais pares, e vai para `src/theme/tone.ts`. O `Tag` passa a importar dali.

### 3.2 Componentes base

- **`Button`:**
  - texto `var(--on-accent)`;
  - desabilitado ou ocupado com fundo `--sunken`, texto `--ink2` e borda `2px dashed var(--rule2)`, sem opacidade;
  - o atributo `disabled` e o rótulo "…" enquanto está ocupado continuam iguais.

### 3.3 Faixa de ambiente

- **`EnvBanner`:** não muda. Continua com `role="status"`, o texto exato e `data-env`.
- **O que muda:** só as cores. As de development e staging não podem coincidir com nenhum papel de cor do tema `rota`.

### 3.4 Shell

- **`<header>`:**
  - fundo `var(--ink)`, que é o petróleo escuro do tema `rota`, com texto `var(--panel)`;
  - marca: um ícone SVG e "Rota Saúde";
  - selo "Manutenção": um `<span>` com fundo `--accent-bg` e texto `--accent`, sem `role`;
  - à direita, o e-mail num único nó de texto e o `Button` "sair".
- **Trilho:**
  - `<nav aria-label="Telas">` com 92px de largura;
  - um `<button>` por tela, com ícone SVG inline (`aria-hidden`) e o rótulo visível abaixo;
  - o nome acessível continua exatamente "Cidades", "Medicamentos", "Mantenedores", "Tokens" ou "Auditoria";
  - `aria-current="page"` no ativo;
  - visível em todas as larguras, sem gaveta, porque são só 5 itens.
  - Ícones: cidade, comprimido, pessoas, chave e histórico. Sem biblioteca nova.
- **`<main>`:** o mesmo padding, ao lado do trilho.
- **O que não muda:**
  - o estado e o `goTo`;
  - o detalhe da cidade e as abas, que só recebem as cores do tema;
  - o Login e o Convite, que só recebem as cores e a fonte.

## 4. Tabelas, testes e critério de pronto

### 4.1 Tabelas

- **Contêiner `<div className="table-scroll">`** em volta da `<table>`, em três lugares:
  - no `DataTable`;
  - na tabela feita à mão de `Audit.tsx`;
  - na tabela de `ProtocolsTab.tsx`.
- **CSS:** `.table-scroll { overflow-x: auto; }`. A dica "⇆ role para o lado" aparece por `::after` só quando o contêiner tem a classe `is-overflowing`.
  - Quem põe a classe é um efeito com `ResizeObserver`, que compara `scrollWidth` com `clientWidth`. No jsdom não há `ResizeObserver`, então o efeito não faz nada nos testes.
  - A dica não entra no texto do DOM.
- **O que não muda:** a estrutura `<table>/<thead>/<tr>/<td>`.

### 4.2 Testes

**Garantia:**
- os 29 arquivos do Vitest passam sem asserção editada;
- o e2e Playwright (`e2e/maintenance.spec.ts`) passa inteiro contra o stack de dev.

**Testes novos (Vitest):**
- **Tokens:** o lock da cópia e o `index.html` com `data-theme="rota"`.
- **Shell:**
  - `getByRole("navigation", { name: "Telas" })` contém os 5 botões, com nome exato e ícone `aria-hidden`;
  - o texto "Manutenção" está dentro do `<header>` e não é `role="status"`;
  - o e-mail e "sair" estão dentro do `<header>`.
- **Faixa:**
  - as cores de development e staging, lidas do `global.css`, não coincidem com nenhum valor do bloco `rota` em `vendor/tokens.css`;
  - development tem contraste de 7:1 ou mais com o texto.
- **`tone.ts`:** devolve `{ fg: "var(--ok)", bg: "var(--ok-bg)" }` e os demais pares.
- **`Button`:** desabilitado sem opacidade, com borda tracejada.
- **Tabelas:** o `DataTable`, a Auditoria e a ProtocolsTab renderizam a `<table>` dentro de `.table-scroll`.
- **Guardas de fonte:** em `src`, fora de `vendor/` e dos testes, não pode haver:
  - `oklch(`, `rgba(`, hex em string, `"white"` nem `var(--x, reserva)`;
  - `outline: "none"`;
  - `opacity` atrelada a `disabled`.

**Conferência visual:**
- capturas antes e depois em 768, 1024 e 1280px de Login, Cidades, Detalhe › Protocolos, Tokens e Auditoria, com a faixa de development;
- uma captura de staging, num Vite do worktree com `VITE_MAINTENANCE_ENV=staging`.

**Pronto quando:**
1. o Vitest e o e2e estiverem verdes;
2. o usuário aprovar as capturas;
3. o merge e o push estiverem feitos no maintenance e no docs.

O worktree fica em `rota-saude/.worktrees/maintenance-ui`, fora da pasta montada pelo container.

## 5. Riscos

- **O e2e cria um mantenedor novo a cada execução** via `docker compose exec api bin/rails maintainer:invite`, e isso deixa dados no dev. Esse é o comportamento atual do e2e, que já é aceito.
- **Figtree com 13px** pode ficar mais estreita ou mais larga que a Geist. As capturas conferem quebra de linha e corte, e só espaçamento pode ser ajustado.
- **`ResizeObserver` em navegador antigo:** sem ele, a dica de rolagem não aparece, mas a rolagem continua funcionando.
