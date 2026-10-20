# Ciclo de interface — subprojeto 4: wpda — design

**Data:** 2026-10-12
**Status:** em revisão
**Afeta:** só `apps/wpda`. Os temas `govbr`, `govbr-escuro` e `alto-contraste` já existem em `contracts` (`tokens-v1.1.0`), então o contracts não muda. Também não mudam api, dashboard, admin e maintenance.
**Spec do ciclo:** `docs/superpowers/specs/2026-10-12-interface-cycle-foundation-design.md`, decisões 4 a 9:
- AA com eMAG;
- linguagem gov.br nos nossos componentes;
- topo por cidade com reserva "Rota Saúde";
- modo escuro seguindo o sistema;
- alto contraste;
- bloqueio fora do celular.
**Card relacionado:** rotasaude/api#73, dado da cidade. Continua aberto; o topo nasce com a reserva.

**Regra do ciclo:** só interface. A única exceção autorizada é a decisão 9: o fluxo do cidadão é bloqueado fora do celular, e o relatório (`?token=`) abre em qualquer tela.

## 1. Ponto de partida

- **Tema:** o wpda usa a cópia antiga do tema console (`src/theme/global.css`, com corpo de 13px). Nenhuma fonte é carregada, e `Screen` e `Report` fixam `system-ui`. O `src/theme/tokens.ts` não é importado por ninguém. Não há `data-theme` nem script de tema.
- **Componentes base:** ficam em `modules/citizen/ui.tsx` (`Screen`, `BigButton`, `Field`, `ErrorText`, `Toggle`, `RadioGroup`) e usam só estilo inline. Os tamanhos ficam como estão (decisão 4).
- **O que falha no AA:**
  - borda `#ccc`, com contraste de 1,6:1;
  - foco visível só em `button` e `a`;
  - o "Em emergência, ligue 192" é texto puro (`ui.tsx:17`), sem `tel:`, e não aparece no relatório;
  - o estado desabilitado é mostrado só por opacidade;
  - o erro do `Field` não entra no `aria-describedby`.
- **Cores fixas:** são 25 arquivos com `var(--x, #hex)` ou hex fixo. Alguns nomes não existem em lugar nenhum: `--warnBg` em vez de `--warn-bg`, e `--accent-soft`. Por isso esses pontos sempre exibem o valor de reserva.
- **Relatório:** o `Report` tem layout próprio e não usa o `Screen`. O resultado logado (`ResultStep`) também renderiza o `Report`. O nível (`tier`) é texto livre definido pelo autor do protocolo e aparece numa etiqueta de mesma cor para qualquer nível.
- **Navegação:** a barra "Avisos/Sair" (`Flow.tsx:208-221`) fica fora do `Screen`.
- **Testes:** são 35 arquivos.
  - Cerca de 7 conferem tamanho inline (`minHeight`, `fontSize`).
  - O `Report.test` confere o texto exato do nível ("alta") e "Seu resultado".
  - Há muitas consultas de títulos por nome.
  - O jsdom não tem `matchMedia`.
  - Não existe Playwright no wpda.
- **Dev:** o serviço é `wpda-dev`, na porta 5176, com `base: "/wpda/"`. A cidade é resolvida pelo host, por exemplo `curitiba.localhost:5176/wpda/`.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | **Topo no celular:** uma barra compacta de acessibilidade, sempre visível, com "Acessibilidade" e "Alto contraste". O "Ir para o conteúdo" fica escondido até receber foco. Abaixo dela vem o espaço da cidade. |
| 2 | **Nível no resultado:** ganha cor e forma por nível, como nos painéis (■ alto, ▲ moderado, ● baixo). O texto do nível não muda. Nível desconhecido fica neutro. |
| 3 | **Abordagem:** trocar só as cores, mantendo o estilo inline. Os tamanhos e espaçamentos inline ficam como estão. Mudar para classes CSS foi recusado. |

## 3. Tema, tokens e componentes do topo

### 3.1 Tokens e CSS global

- **Cópia dos tokens:**
  - `scripts/tokens-sync.mjs`, igual ao dos painéis;
  - `src/theme/vendor/` com `tokens-v1.1.0`;
  - `src/theme/vendor.lock.test.ts`.
- **`src/theme/global.css` reescrito:**
  - `@import "./vendor/tokens.css";`;
  - reset;
  - `body` com `--bg`, `--ink`, `--font-sans` e 18px;
  - `:focus-visible { outline: 3px solid var(--focus); outline-offset: 2px; }` em todo elemento;
  - `a { color: var(--accent) }`;
  - as regras de forma do nível (seção 4.2).
- **Apagado:** `src/theme/tokens.ts`, que ninguém importa.
- **`Screen` e `Report`:** deixam de fixar `fontFamily`, e a fonte vem do `body`.

### 3.2 Tema (`src/theme/theme.ts`)

- **Escolha:** `WpdaThemeChoice = "auto" | "alto-contraste"`, salva com a chave `THEME_KEY = "rotasaude:wpda-theme"` no `localStorage`, dentro de `try/catch`. Se o armazenamento falhar, a escolha fica em memória. O padrão é `auto`.
- **Tema efetivo** (`effectiveTheme()`):
  - `alto-contraste` quando essa é a escolha;
  - senão, `govbr-escuro` se `matchMedia("(prefers-color-scheme: dark)")` casar;
  - senão, `govbr`.
- **Funções:**
  - `applyTheme()` grava o tema efetivo no `data-theme` do `<html>`;
  - `toggleHighContrast()` alterna entre `auto` e `alto-contraste`;
  - `useThemeChoice()` expõe a escolha.
- **Troca de modo com a tela aberta:** um ouvinte de `change` no `matchMedia` reaplica o tema quando a pessoa muda o modo do celular.
- **Antes do React montar:** um script inline no `index.html` aplica o tema, com a mesma regra, para não piscar. O `<html>` ganha `data-theme="govbr"` como padrão.

### 3.3 Componentes novos

Ficam em `src/modules/citizen/`.

- **`A11yBar`:** uma faixa no topo, com fonte de 13px e fundo `--sunken`.
  - Link "Ir para o conteúdo": visualmente escondido até receber foco. Move o foco para o `main` e não navega.
  - Link "Acessibilidade": abre o passo `a11y` do fluxo. No relatório, que não tem fluxo, abre a mesma página como camada sobre o relatório, com botão Voltar.
  - Botão "Alto contraste", com `aria-pressed`.
- **`A11yPage`:** página estática com:
  - o nível declarado: WCAG 2.2 AA e eMAG 3.1;
  - o alto contraste e o modo escuro automático;
  - os alvos de toque grandes;
  - o nível de risco indicado por cor, forma e texto;
  - a compatibilidade com leitor de tela;
  - a orientação para relatar problemas: "procure a unidade de saúde ou a prefeitura".
  - Tem um `BigButton` "Voltar".
- **`CityHeader`:** substitui o `<strong>Rota Saúde</strong>` do `Screen` e do `Report`.
  - Hoje mostra um ícone neutro (desenhado em SVG inline, sem biblioteca de ícones no wpda) e "Rota Saúde".
  - A interface já aceita `{ name?: string; coatOfArmsUrl?: string }` para o api#73. Por enquanto, nada passa esses dados.
- **`EmergencyCall`:** um `<a href="tel:192">` com o texto "Emergência? Ligar 192".
  - Altura mínima de 48px, fundo `--down-bg`, texto `--down` e raio de 12px.
  - Substitui o `<p>` do `Screen` e entra também no `Report`.

### 3.4 Composição

- **`Screen`:** `A11yBar` → `<header>` com `CityHeader` e `h1` → conteúdo → rodapé → `EmergencyCall`.
- **`Report`:** `A11yBar` → `CityHeader` → o artigo de hoje → `EmergencyCall`. O título "Seu resultado" e o texto do nível ficam como estão.
- **`Flow`:** a barra "Avisos/Sair" fica no mesmo lugar, só com as cores novas. O passo `a11y` entra no `switch` de passos sem roteador novo: o "Voltar" retorna ao passo anterior.

## 4. Correções AA, cores, nível e bloqueio

### 4.1 Componentes base e cores

- **`BigButton`:**
  - `secondary` com borda de 2px em `--rule2`;
  - `primary` com `--accent` e `--on-accent`;
  - `danger` com `--down` e `--on-tone`;
  - desabilitado com fundo `--sunken`, texto `--ink2`, borda `--rule` e `cursor: default`, no lugar da opacidade. O atributo `disabled` continua.
- **`Field`:** borda `--rule2`, ou `--down` quando há erro. O erro recebe um `id`, e o `aria-describedby` passa a juntar a dica e o erro. A dica usa `--ink2`.
- **`Toggle` e `RadioGroup`:** borda `--rule2`. O desabilitado deixa de usar opacidade e passa a ter texto `--ink2` e borda `--rule`.
- **Os 25 arquivos com cor fixa:**
  - todo `var(--x, #hex)` vira `var(--x)` com o papel certo;
  - `--warnBg` vira `--warn-bg`, e `--accent-soft` vira `--accent-bg`;
  - `#fff` sobre `--accent` vira `--on-accent`, e `#fff` sobre `--down` (selo de avisos) vira `--on-tone`;
  - `--line` agora existe nos tokens (é um alias de `--rule2`).

### 4.2 Nível no resultado

- **Arquivo:** `src/lib/tier.ts` exporta `tierTone(tier: string | null): "down" | "warn" | "ok" | null`.
- **Normalização:** antes de comparar, o texto perde acento e maiúsculas.

  | Termos | Tom |
  |---|---|
  | alta, alto, urgente, emergencia, emergente, grave, vermelho | `down` |
  | media, medio, moderada, moderado, intermediaria, amarelo | `warn` |
  | baixa, baixo, leve, verde | `ok` |
  | qualquer outro | `null` |

- **No `Report`:** a etiqueta usa `--<tom>-bg` no fundo e `--<tom>` no texto, ou `--accent-bg` e `--accent` quando o tom é `null`. Ganha `data-glyph="square|triangle|circle"`, e o `global.css` desenha a forma em `::before` com texto alternativo vazio. O texto do nível não muda.

### 4.3 Bloqueio fora do celular

- **`src/lib/device.ts`** exporta `isPhone(win = window): boolean`, que é verdadeiro quando `Math.min(win.screen.width, win.screen.height) < 600` **e** `win.matchMedia("(pointer: coarse)").matches`.
- **`App.tsx`:** `token` mantém o `Report`. Sem token, o app mostra o `Flow` se `isPhone()`, e `PhoneOnly` nos outros casos.
- **`PhoneOnly`** (`modules/citizen/PhoneOnly.tsx`):
  - `A11yBar`;
  - `CityHeader`;
  - `h1` "Abra no celular";
  - o texto "Este serviço foi feito para ser usado no celular. Abra este endereço no seu celular.";
  - o endereço atual (`location.origin + location.pathname`) num `<code>` selecionável;
  - `EmergencyCall`.
- **Testes:** o `src/test/setup.ts` ganha um stub padrão de celular, com `matchMedia`, `screen.width` 390 e `screen.height` 844. Os testes de bloqueio sobrescrevem o stub. Isso é infraestrutura de teste, não edição de asserção.
- **Revisão 2026-10-20 — exceção em dev e staging (wpda 87a2de2, card rotasaude/wpda#2):** o fluxo do cidadão fica liberado no desktop em development (Vite com `MODE` "development") e em staging (container nginx com `WPDA_ALLOW_DESKTOP=1`; o entrypoint `docker/40-wpda-runtime-config.sh` grava `/wpda/runtime-config.js`, servido sem cache, na mesma imagem de produção). Em produção, sem a variável ou com qualquer valor diferente de "1", o bloqueio "Abra no celular" continua: o padrão é bloquear. Com a liberação ativa, a faixa "Modo desktop — ambiente de teste" aparece no topo. O bloqueio é de usabilidade, não de segurança (roda no navegador). O deploy de staging do wpda precisa incluir `WPDA_ALLOW_DESKTOP=1`; o README do wpda (seção "Desktop em dev e staging") descreve a variável.

### 4.4 Fora do escopo

- **`wpdaLook.ts` do dashboard:** é a cópia do visual do wpda na prévia do editor de protocolo. Fica desatualizado e vira um card no board #2.
- **O dado da cidade (api#73).**

## 5. Testes e critério de pronto

**Garantia de funcionalidade:** os 35 arquivos de teste atuais passam sem nenhuma asserção editada.

**Testes novos (Vitest):**
- **Tokens:**
  - o `vendor.lock`;
  - o script do `index.html`: aplica `alto-contraste` salvo, segue o modo escuro simulado e ignora valor inválido;
  - o `index.html` com `data-theme="govbr"`.
- **Tema:**
  - `auto` vira `govbr` ou `govbr-escuro` conforme o `matchMedia` simulado, e reage à troca de modo;
  - o alto contraste liga, desliga e vence o modo escuro;
  - com armazenamento quebrado, nada lança erro e a escolha fica em memória.
- **`A11yBar`:**
  - "Ir para o conteúdo" foca o `main` sem mudar o histórico;
  - "Alto contraste" tem `aria-pressed`;
  - "Acessibilidade" abre a página, e o "Voltar" volta ao passo anterior.
- **`EmergencyCall`:** `getByRole("link", { name: "Emergência? Ligar 192" })` com `href="tel:192"`, presente no `Screen`, no `Report` e no `PhoneOnly`.
- **`CityHeader`:** mostra "Rota Saúde" quando não recebe dados.
- **`tierTone`:** cada termo da tabela, com e sem acento e com maiúsculas; desconhecido e `null` dão `null`. No `Report`, `data-glyph` e o texto "alta" inalterado.
- **`isPhone` e `App`:**
  - celular em pé e deitado passam;
  - tablet (768 × 1024, toque), notebook com toque (1366 × 768, toque) e desktop (1920 × 1080, mouse) são bloqueados e mostram `PhoneOnly`;
  - com `?token=`, sempre abre o `Report`.
- **Guardas de fonte:** em `src`, fora de `vendor/` e dos testes, não pode haver:
  - `var(--x, #hex)`;
  - `"#fff"` nem `"white"`;
  - `--warnBg` nem `--accent-soft`;
  - `outline: "none"`;
  - `opacity: props.disabled` nem `opacity: disabled`.

**Playwright no wpda:**
- **Novos:** `playwright.config.ts` e `e2e/device-gate.spec.ts`.
- **Perfis:** iPhone 13 (WebKit), Pixel 7 (Chromium), iPad Pro 11 (WebKit), Desktop Chrome e Desktop Firefox.
- **O que confere,** contra `http://curitiba.localhost:5176/wpda/`:
  - celulares mostram a primeira tela do fluxo (o campo de celular);
  - iPad e desktops mostram o `h1` "Abra no celular";
  - `/wpda/?token=invalido` nunca mostra "Abra no celular".
- **Download:** os navegadores WebKit e Firefox do Playwright precisam ser baixados, com permissão do usuário na execução.

**Conferência visual:** capturas em 360 × 800 nas telas abaixo, nos temas claro, escuro (`prefers-color-scheme` emulado) e alto contraste. O "antes" fica só no claro, porque hoje não há outros temas.
- telas do fluxo: celular, código, pessoas, pergunta e avisos;
- relatório;
- `PhoneOnly`: em 1280 × 800 com mouse.

**Pronto quando:**
1. as suítes estiverem verdes;
2. o Playwright passar nos 5 perfis;
3. o usuário aprovar as capturas;
4. o merge e o push estiverem feitos no wpda e no docs;
5. o card do `wpdaLook.ts` estiver aberto no board #2.

Antes de começar, avisar a sessão dona do wpda, se existir.

## 6. Riscos

- **Mudar o `data-theme` na carga e ao trocar de modo** pode afetar testes que renderizam o `App`. O stub do `setup.ts` define o modo claro, e o efeito só escreve no `<html>`.
- **`screen` e `pointer` em navegadores incomuns.** Um celular que informa `pointer: fine` (com caneta ou mouse Bluetooth) seria bloqueado. A regra é a decidida (decisão 9), e o Playwright cobre os perfis comuns.
- **Nível com termo inesperado:** fica neutro, como hoje. A tabela pode crescer depois sem mudar a interface.
- **WebKit no Playwright:** exige o download do navegador, e no macOS o perfil "iPhone" usa o WebKit do Playwright, não o Safari real.
