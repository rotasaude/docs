# Ciclo de interface — subprojeto 3: tabelas em cartões e layouts em 768px — design

**Data:** 2026-10-12
**Status:** em revisão
**Afeta:**
- `apps/dashboard` e `apps/admin`:
  - `src/components/DataTable.tsx` passa a ter um modo cartão por CSS e ganha um `datatable.css`;
  - `src/components/KpiGrid.tsx`;
  - `className` e regras de 768px nas Visões gerais;
  - no dashboard, o Editor de protocolo; no admin, os formulários Provisionar cidade e MFA;
  - um ajuste por tela no CNES.
- `contracts`, `apps/wpda`, `apps/maintenance` e `apps/api`: nada.

**Specs anteriores:**
- `docs/superpowers/specs/2026-10-12-interface-cycle-foundation-design.md`. A decisão 14 diz que 768, 1024 e 1280px são de primeira classe e que, em 768px, as tabelas viram cartões.
- `docs/superpowers/specs/2026-10-12-interface-panel-shell-design.md`, que define o shell.

**Regra do ciclo:** só interface. Nenhuma tela, dado, permissão ou chamada ao api muda.

## 1. Ponto de partida

- **Um único componente de tabela.** O dashboard tem 60 chamadas ao `DataTable` em 47 arquivos e o admin tem 17 em 11. O `DataTable` **não usa `<table>`**: é uma grade CSS de `div`s com `role="table"`, `role="row"` (o cabeçalho é a primeira linha), `role="columnheader"` e `role="cell"`.
  - **Colunas:** `Column<T> = { label, w?, align?, render }`. As larguras viram um `gridTemplateColumns` inline.
  - **Linha clicável:** com `onRowClick`, a linha é um `<button role="row">`.
  - **Fundo da linha:** o dashboard tem `rowStyle`, que é inline.
  - **Lista vazia:** sem linhas, o componente não renderiza `role="table"`.
  - **Células:** usam `nowrap` com reticências.
- **Conteúdo das células:** só texto e botões. O CNES é a exceção, com uma caixa de seleção na primeira coluna (32px).
- **Testes que dependem do DOM da tabela:** 23 arquivos e mais o `DataTable.test`. Eles consultam `getAllByRole("row")`, `getAllByRole("cell")`, `getAllByRole("columnheader")`, `queryByRole("table")`, os filhos diretos de uma linha e o `textContent` exato de linhas e células. Também comparam `style.background` da linha (`transparent` ou `var(--down-bg)`) e conferem que a linha não é `BUTTON` quando não há `onRowClick`. O jsdom não aplica `@media`.
- **Outros layouts que apertam em 768px.** Abaixo de 1024px o conteúdo tem cerca de 720px de largura, porque o trilho some.
  - Visão geral, nos dois apps:
    - `KpiGrid columns={5}` no dashboard e `KpiGrid5` (`repeat(5, 1fr)`) no admin;
    - painéis lado a lado (`1.5fr 1fr`, `1fr 1.4fr`);
    - `repeat(3, 1fr)` nos indicadores-chave;
    - fluxo de eventos (`60px 1.1fr 1.4fr`).
  - Editor de protocolo (`1fr 1fr`).
  - Formulários do admin Provisionar cidade e MFA (`1fr 1fr`).
- **O que não precisa de nada:**
  - a única `<table>` com dados, a tabela por período do Analytics (`PeriodTable`), já rola dentro do painel recolhível;
  - os gráficos (recharts) já são responsivos;
  - o painel do sino tem 384px e cabe.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | **Anatomia do cartão:** a primeira coluna vira o título, as colunas de dados viram pares rótulo/valor e as células com controles descem para uma faixa de ações no rodapé. |
| 2 | **Como decidir o papel de cada coluna:** é automático por CSS. Por padrão a primeira coluna é o título, e a célula com `button`, `input` ou `a` é ação (seletor `:has()`). Quando o automático não serve, a coluna recebe a propriedade opcional `card: "title" \| "actions" \| "hide"`. O ajuste por tela começa só no CNES; outros saem das capturas. |

## 3. O `DataTable` em modo cartão

### 3.1 Mudanças no componente

São só atributos e classes, iguais nos dois apps:

- `role="table"` ganha `className="dt"`.
- A linha de cabeçalho ganha `className="dt-row dt-head"`, e cada linha de dado ganha `className="dt-row"`.
- Cada célula de dado ganha `data-label="<label da coluna>"`, além de `data-card="<valor>"` quando a coluna tiver `card`.
- O tipo `Column<T>` ganha `card?: "title" | "actions" | "hide"`.
- Fica tudo como está: as props de hoje, a estrutura (as células continuam filhas diretas da linha), o texto, o `gridTemplateColumns` inline, o `rowStyle`, o `<button role="row">` com `onRowClick` e o estado vazio sem `role="table"`.

### 3.2 Regras de CSS

O arquivo é `src/components/datatable.css`, importado pelo `DataTable`. Todas as regras ficam dentro de `@media (max-width: 1023px)`.

| Elemento | Regra |
|---|---|
| `.dt` | Sem borda externa; lista de cartões com espaço de 12px entre eles |
| `.dt-head` | Escondido só visualmente (`position:absolute; width:1px; height:1px; overflow:hidden; clip:rect(0 0 0 0); white-space:nowrap`), sem `display:none`, para o leitor de tela continuar lendo os cabeçalhos |
| `.dt-row` | `grid-template-columns: repeat(2, minmax(0, 1fr)) !important`, que é o único `!important` do componente e existe porque o grid inline vence o CSS. Borda `--rule2`, raio `--radius-panel`, espaçamento de 12px a 14px e espaço de 8px a 16px entre os itens. A cor de fundo inline (`rowStyle`) continua valendo |
| `.dt-row > [role="cell"]` | Sem `nowrap` e sem reticências: `white-space: normal; overflow: visible; text-overflow: clip`, para o texto quebrar em vez de cortar. Alinhamento à esquerda |
| Par rótulo/valor | `.dt-row > [role="cell"]::before { content: attr(data-label); display:block; font-size:11px; text-transform:uppercase; color: var(--ink2); }`. O rótulo **não** entra no `textContent` |
| Título | `.dt-row > [role="cell"]:first-child:not([data-card]), [data-card="title"]`: ocupa a linha inteira (`grid-column: 1 / -1`), com 15px em negrito e sem rótulo (`::before { content: none }`). Quando uma coluna declara `data-card="title"`, a primeira célula sem `data-card` deixa de ser título |
| Ações | `[role="cell"]:has(button, input, a):not([data-card]), [data-card="actions"]`: ocupa a linha inteira, sem rótulo, `display:flex; flex-wrap:wrap; gap:8px`, com borda superior `--rule` e alvos de 44px ou mais. Ficam no fim do cartão (`order: 99`) |
| `data-card="hide"` | `display: none` |
| Linha clicável | O `<button role="row">` vira o cartão inteiro, com `text-align:left` e `width:100%` |

Regras que o CSS precisa respeitar:
- **Prioridade dos seletores.** Para a primeira coluna virar título e, ao mesmo tempo, nunca ser tratada como ação por engano, a regra de ação exclui `:first-child` quando a tabela não declara título. A ordem de prioridade é: `data-card` explícito, depois título automático, depois ação automática, depois par.
- **Cabeçalho escondido.** As regras do cabeçalho e as da linha não podem se sobrepor: o `.dt-head` é escondido inteiro e não recebe o layout de cartão.

### 3.3 Ajuste por tela

- **CNES (`modules/Cnes.tsx`, tabela com caixa de seleção):** a primeira coluna (caixa de seleção) recebe `card: "actions"`, e a coluna do nome do estabelecimento recebe `card: "title"`.
- **Outras tabelas:** só recebem ajuste se a captura de 768px mostrar título ou ação errados. Cada ajuste é registrado no ledger.

## 4. Outros layouts em 768px

Todos usam o mesmo padrão: um `className` no elemento e a regra num CSS do componente ou da tela, dentro de `@media (max-width: 1023px)`. Quando o valor original é inline, a regra usa `!important` **só** em `grid-template-columns`.

| Onde | Classe | Abaixo de 1024px |
|---|---|---|
| `KpiGrid` (dashboard e admin), quando há `columns` | `kpi-grid` | `repeat(auto-fit, minmax(200px, 1fr))` |
| Admin `Overview.tsx` `KpiGrid5` | `kpi-grid` | Igual à linha anterior |
| Visão geral, linhas de dois painéis (dashboard `Overview.tsx:42,:46`; admin `:38,:42`) | `ov-split` | Uma coluna |
| Visão geral, `repeat(3, 1fr)` (dashboard `:213`; admin `:207`) | `ov-kv3` | 2 colunas |
| Visão geral, fluxo de eventos (dashboard `:347`; admin `:340`) | `ov-event` | `60px 1fr` (o detalhe desce para baixo da hora e do evento) |
| `ProtocolEditor.tsx:95` | `pe-split` | Uma coluna (editor em cima, prévia embaixo) |
| Admin `setup/ProvisionCity.tsx:100`, `setup/MfaEnroll.tsx:96` | `form-2col` | Uma coluna |

As regras ficam em `src/theme/responsive.css`, importado pelo `global.css` em cada app, num arquivo só por app.

## 5. Testes e critério de pronto

**Garantia de funcionalidade:** as suítes atuais passam sem nenhuma asserção editada, incluindo os 23 arquivos que consultam linhas e células e o `DataTable.test`.

**Testes novos (Vitest), nos dois apps:**
- **`DataTable`:**
  - `className` `dt`, `dt-row` e `dt-head`;
  - `data-label` igual ao `label` em cada célula de dado, e o cabeçalho sem `data-label`;
  - `data-card` só nas colunas que o declaram;
  - o `textContent` de cada linha e célula continua igual ao de antes;
  - as células continuam filhas diretas da linha;
  - `role="row"` em `button` só com `onRowClick`.
- **CSS:** um teste que lê o `datatable.css` e garante que todas as regras estão dentro de `@media (max-width: 1023px)`, que não há `display: none` no `.dt-head` e que o `!important` aparece só em `grid-template-columns`. O mesmo vale para o `responsive.css`.
- **CNES:** `data-card="actions"` na célula da caixa de seleção e `data-card="title"` no nome.
- **`className` dos layouts:** um teste por tela, que renderiza com os fixtures existentes e confere que o elemento tem a classe.

**Verificação no navegador (Playwright, contra os worktrees):**
- **Varredura de rolagem lateral:** para cada tela do menu (34 no dashboard com o admin municipal da semente, 6 no admin com o operador) e para cada largura (768, 1024 e 1280px), `document.documentElement.scrollWidth === clientWidth`.
  - Telas que o perfil não alcança, ou que estão vazias na semente, entram nas evidências como **não verificadas**, nunca como aprovadas.
- **Capturas em 768px (antes e depois):**
  - dashboard: Equipe, Pedidos, Fila da unidade, Protocolos, Filas, CNES, Visão geral e Editor de protocolo;
  - admin: Cidades, Produção das cidades e Visão geral (esta só se alcançável; senão, fica registrada como não verificada).

**Pronto quando:**
1. as suítes estiverem verdes;
2. a varredura não mostrar rolagem lateral em nenhuma tela verificada;
3. o usuário aprovar as capturas;
4. o merge e o push estiverem feitos no dashboard, no admin e no docs.

## 6. Riscos

- **`:has()` não suportado:** fica sem efeito em navegador antigo (anterior ao Firefox 121, de 2023). Nesse caso a célula de botão vira um par com rótulo e não vai para o rodapé. O conteúdo continua acessível.
- **Primeira coluna que não serve de título:** o automático erra nas tabelas cuja primeira coluna é um botão, uma data curta ou uma caixa de seleção. Só o CNES está mapeado; as capturas e a varredura apontam os outros casos, que recebem `card` na coluna.
- **`!important` no grid inline:** fica restrito a `grid-template-columns`, nas classes listadas, dentro do `@media`. O teste de CSS impede que escape.
- **Telas não alcançáveis na varredura:** ficam listadas como não verificadas e não são presumidas corretas.
