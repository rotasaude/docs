# Ciclo de interface — decisões do ciclo e subprojeto 1 (fundação de tokens) — design

**Data:** 2026-10-12
**Status:** em revisão
**Afeta (subprojeto 1):**
- `contracts`: `design-tokens/`, que deixa de ser esqueleto e ganha fonte JSON, gerador, arquivos gerados, teste de contraste e tag `tokens-v1.0.0`.
- `apps/dashboard` e `apps/admin`: `src/theme/` passa a consumir a cópia dos tokens, com o tema `govbr`, a fonte Rawline e correções de AA nos componentes base.
- `apps/wpda`, `apps/maintenance`, `apps/api`, `apps/catalog` e `apps/signer`: nada neste subprojeto.

**ADRs:** `docs/adr/0002.md` (design system compartilhado como tokens, não como componentes), `docs/adr/0015.md` (versionamento do contracts), `docs/adr/0025.md` (dissolução do `packages/ui`).
**Card relacionado:** rotasaude/api#73 (nome e brasão da cidade para o wpda).

**Regra do ciclo:** só interface. Nenhuma funcionalidade muda. A única exceção autorizada é o bloqueio do fluxo do wpda fora do celular (decisão 9). Ela é implementada no subprojeto 4.

## 1. Ponto de partida

- **Stack:** os quatro frontends (wpda, dashboard, admin e maintenance) usam React 18, Vite 5 e TypeScript. Nenhum usa Tailwind, biblioteca de componentes, biblioteca de ícones ou router. Quase todo o estilo é `style={{}}` inline, lendo variáveis CSS.
- **Tema:** um tema "console" único, copiado à mão em `src/theme/{tokens.ts,global.css}`. As cópias do wpda, do dashboard e do admin são idênticas byte a byte. O maintenance acrescenta as variáveis `--env-*` e a classe `.env-banner`. A paleta é oklch neutra com acento azul-violeta, a fonte é Geist e não há modo escuro.
- **Componentes copiados entre apps:**
  - Entre admin e dashboard, cerca de 15 são idênticos.
  - Outros 8 divergiram: DataTable, KpiGrid, Sparkline, StatTile, PageHeader, BarMini, Panel e AppHeader.
  - O maintenance reescreveu os seus.
- **Responsividade:** zero `@media` nos quatro apps. O menu do dashboard já se sobrepõe por volta de 1024px.
- **wpda:** a UI do cidadão (`modules/citizen/ui.tsx`) já usa texto de 18px, alvos de 56px e rótulos visíveis. O que falha no AA:
  - a borda das opções `#ccc` tem 1,6:1, abaixo dos 3:1 exigidos;
  - não há foco visível;
  - o "ligue 192" é texto sem link;
  - o estado desabilitado é indicado só por opacidade.
- **Testes:** Vitest com Testing Library nos quatro apps (wpda 31, dashboard 186, admin 22, maintenance 27 arquivos). Playwright existe só no maintenance. Não há teste visual nem de snapshot.
- **`contracts/design-tokens/`:** só um README que avisa: "SCAFFOLD — precisa de input de design".
- **Distribuição atual do contracts:** por cópia. O api valida protocolos contra `apps/api/config/protocols/schema.json`, e nada garante que essa cópia seja igual à tag.

## 2. Decisões do ciclo

Estas decisões valem para todos os subprojetos.

| # | Decisão |
|---|---|
| 1 | Prioridade do ciclo: consistência, depois uso em tablet, depois identidade, depois usabilidade dos fluxos. |
| 2 | Três famílias visuais: **painel** (dashboard e admin, mesma linguagem, tablet e desktop), **manutenção** (maintenance, tablet e desktop) e **cidadão** (wpda, só celular). |
| 3 | Não havia marca. A identidade é definida neste ciclo, do zero. |
| 4 | **wpda: WCAG 2.2 AA + eMAG.** Os tamanhos atuais ficam, e as falhas da seção 1 são corrigidas. AAA foi considerado e recusado. |
| 5 | **wpda: linguagem do Padrão Digital de Governo (Design System gov.br)**, implementada nos nossos próprios componentes. O pacote `@govbr-ds/core` não é usado. |
| 6 | **wpda: topo por cidade**, com brasão e nome da prefeitura e reserva "Rota Saúde" enquanto o dado não existe. O dado vem do api (rotasaude/api#73). A marca gov.br só aparece se a cidade tiver direito a ela. |
| 7 | **wpda: modo escuro seguindo o sistema** (`prefers-color-scheme`). Contraste e cores de risco são validados nos dois fundos. |
| 8 | **wpda: tema de alto contraste**, exigido pelo eMAG. Ao todo, o wpda tem três temas: claro, escuro e alto contraste. |
| 9 | **wpda fora do celular:** o fluxo do cidadão é bloqueado. A tela de bloqueio mostra o topo da cidade, a mensagem, o endereço por extenso e o botão "Ligar 192". O relatório público (`?token=`) abre em qualquer tela. A regra de "é celular" é: lado menor de `screen` < 600px **e** `pointer: coarse`. Os testes são unitários da regra e Playwright no wpda com perfis reais (iPhone/WebKit, Pixel/Chrome, iPad/WebKit e desktop com Chrome e Firefox). |
| 10 | **Painéis (dashboard e admin): WCAG 2.2 AA + eMAG**, com barra de atalhos numerados, página de acessibilidade e tema de alto contraste. O modo escuro fica adiado. |
| 11 | **Painéis: dois temas de identidade escolhíveis**, ambos no AA. O padrão é `govbr`, na família do wpda. O outro é `rota`, a identidade própria: Figtree, petróleo `#0D6363` com coral, fundo areia e cantos de 14px. Este último é um esboço, e a marca definitiva (logo e regras) é um trabalho separado. O tema de alto contraste é único e comum aos dois. |
| 12 | **Quem escolhe o tema:** cada pessoa, num seletor no menu da conta. A escolha fica salva no navegador, sem api. Escolha por cidade ou por perfil fica para depois. |
| 13 | **maintenance:** usa a identidade própria (`rota`) com o selo "Manutenção", em AA sem eMAG. A faixa de ambiente (development/staging) nunca pode se confundir com o tema (spec F6 do maintenance). |
| 14 | **Painéis responsivos, alvo B:** 768px (tablet em pé), 1024px (tablet deitado) e 1280px ou mais (desktop) são todos de primeira classe. Em 768px, as tabelas viram cartões, o que exige um layout alternativo para cada tela com tabela. |
| 15 | **Navegação dos painéis:** <ul><li>o Início mostra cards dos grupos (ícone, título e descrição), filtrados pelo papel do usuário;</li><li>a tela do grupo mostra sub-cards das telas, no mesmo visual;</li><li>a tela final tem uma trilha de navegação cujo último item troca de tela dentro do grupo, e uma busca de tela no topo;</li><li>nas telas internas, a barra lateral fica recolhida (trilho de ícones) e vira gaveta em 768px;</li><li>um grupo com uma tela só abre essa tela direto.</li></ul> |
| 16 | **Navegação do maintenance:** só o trilho, sem cards, porque são 5 telas. O detalhe da cidade continua com abas. |
| 17 | **Abordagem de implementação:** primeiro os tokens, depois a migração no lugar, app por app, mantendo a API dos componentes. Foram recusadas a camada de CSS por cima (exigiria `!important` em massa) e a biblioteca de componentes compartilhada (contraria o ADR 0002/0025). |

**Decisão 9 — revisão 2026-10-20 — exceção em dev e staging (wpda 87a2de2, card rotasaude/wpda#2):** o fluxo do cidadão fica liberado no desktop em development (Vite com `MODE` "development") e em staging (container nginx com `WPDA_ALLOW_DESKTOP=1`; o entrypoint `docker/40-wpda-runtime-config.sh` grava `/wpda/runtime-config.js`, servido sem cache, na mesma imagem de produção). Em produção, sem a variável ou com qualquer valor diferente de "1", o bloqueio "Abra no celular" continua: o padrão é bloquear. Com a liberação ativa, a faixa "Modo desktop — ambiente de teste" aparece no topo. O bloqueio é de usabilidade, não de segurança (roda no navegador). O deploy de staging do wpda precisa incluir `WPDA_ALLOW_DESKTOP=1`; o README do wpda (seção "Desktop em dev e staging") descreve a variável.

### Subprojetos, na ordem

Cada subprojeto tem a sua própria spec, o seu plano e a sua execução.

1. **Fundação de tokens:** contracts, mais dashboard e admin no tema `govbr` e no AA. **É esta spec.**
2. **Shell dos painéis:** cards e sub-cards, trilho, gaveta, os três tamanhos, seletor de tema, barra eMAG e alto contraste.
3. **Telas dos painéis em 768px:** tabelas viram cartões, tela a tela.
4. **wpda:** linguagem gov.br, topo por cidade com reserva, modo escuro, alto contraste, correções de AA e bloqueio fora do celular.
5. **maintenance:** tema `rota`, selo, trilho e faixa de ambiente revisada.
6. **Usabilidade dos fluxos:** levantamento por tela.

## 3. Subprojeto 1: forma e distribuição dos tokens

### 3.1 Fonte em três camadas

Os arquivos ficam em `contracts/design-tokens/src/`:

- **`base.json`:** paletas cruas. Inclui o azul gov.br, o petróleo, o coral, os neutros, as cores semânticas (ok, warn, down, info) e as famílias tipográficas (Rawline, Figtree e a pilha de sistema como reserva).
- **`roles.json`:** a lista dos papéis que os componentes consomem. **Os nomes de variável de hoje são mantidos**, para que nenhum componente precise ser renomeado: `--bg`, `--panel`, `--sunken`, `--sunken2`, `--rule`, `--rule2`, `--ink`…`--ink4`, `--accent`, `--accent-bg`, `--ok`, `--ok-bg`, `--warn`, `--warn-bg`, `--down`, `--down-bg`, `--info` e `--info-bg`.
  - Nomes novos entram só para o que hoje não existe: `--focus`, `--font-sans`, `--font-mono`, a escala `--space-1`…`--space-8` e os raios `--radius-chip`, `--radius-panel` e `--radius-menu`.
- **`themes/*.json`:** cada tema associa cada papel a um valor da base. Os temas são:
  - `govbr`: padrão dos painéis e do wpda;
  - `rota`: identidade própria;
  - `alto-contraste`: fundo preto, texto branco e destaque amarelo, conforme o eMAG;
  - `govbr-escuro`: usado só pelo wpda.
  - Neste subprojeto, só o `govbr` precisa estar completo e validado. Os outros três entram com os valores das provas visuais e são validados pelo mesmo teste, mas nenhum app os consome até os subprojetos 2, 4 e 5.

### 3.2 Gerador

`contracts/design-tokens/build.mjs` é um script Node sem dependências. Ele lê as três camadas e grava dois arquivos em `contracts/design-tokens/dist/`:

- **`tokens.css`:** um bloco `:root[data-theme="<tema>"] { … }` por tema, e `:root:not([data-theme])` igual ao `govbr`.
- **`tokens.ts`:** os tipos dos nomes de papel e de tema, e os valores do tema padrão.

Os arquivos de `dist/` são versionados no git. Um teste confere que rodar o gerador não altera `dist/`, ou seja, que o que está versionado é o resultado do gerador.

### 3.3 Validação de contraste

O teste `contracts/design-tokens/test/contrast.test.mjs` usa o runner nativo `node --test`. Ele calcula a razão de contraste WCAG para cada par declarado em `roles.json`, em todos os temas:

- texto sobre fundo: `--ink`, `--ink2` e `--ink3` sobre `--bg`, `--panel` e `--sunken`;
- cor semântica sobre o seu `-bg`;
- texto sobre `--accent`.

Os mínimos são:

| Par | Mínimo |
|---|---|
| Texto | 4,5:1 |
| Borda de controle (`--rule2`) e foco (`--focus`) contra o fundo | 3:1 |
| Qualquer par no tema `alto-contraste` | 7:1 |

`--ink4` só é usado em texto desabilitado ou decorativo, e por isso fica fora da regra de texto. O teste falha se algum par não atingir o mínimo, e nenhuma tag é criada com o teste vermelho.

### 3.4 Distribuição por cópia com trava

Cada app consumidor tem três peças:

- **`scripts/tokens-sync.mjs <tag>`:** busca `dist/` do `rotasaude/contracts` na tag pedida (pelo clone local `contracts/` ou por `git archive` do remoto) e copia os arquivos para `src/theme/vendor/`.
- **`src/theme/vendor/tokens.lock`:** registra a tag e o SHA-256 de cada arquivo copiado.
- **Um teste Vitest** que recalcula os hashes de `src/theme/vendor/` e falha se divergirem do lock. Isso impede que alguém edite a cópia à mão.

Trade-off registrado:

- **Escolhida — cópia com tag:** é simples, cada app sobe de versão no seu ritmo e o build não depende de rede.
- **Recusada — dependência npm via git:** o npm não instala uma subpasta de repositório de forma limpa, e o build passaria a depender de rede.

O README de `contracts/design-tokens/` é reescrito para descrever as camadas, o gerador, a validação, a distribuição e a política de tag (ADR 0015).

## 4. Subprojeto 1: dashboard e admin no tema `govbr`

Os dois apps recebem as mesmas mudanças:

1. **Fonte do tema:**
   - `src/theme/global.css` passa a importar `vendor/tokens.css` e perde o bloco `:root` próprio. O reset continua.
   - `index.html` ganha `<html data-theme="govbr">`.
   - `src/theme/tokens.ts` mantém todos os nomes que exporta hoje (`size.*`, `space.*`, `radius.*`, `toneColor()` e os demais), agora com valores derivados de `vendor/tokens.ts`. Quem importa esses nomes não muda.
2. **Tipografia:**
   - A Geist é trocada pela **Rawline**, servida de arquivo local (`public/fonts/`), sem CDN externa. Os motivos são o CSP e o funcionamento em posto com rede ruim. Saem as dependências `@fontsource-variable/geist*`.
   - O corpo passa de 13px para **14px**.
   - Bloqueio antes de embarcar o arquivo da fonte: confirmar que a licença da Rawline permite redistribuição. Se não permitir, a reserva é a pilha de sistema, com a Raleway (OFL) como alternativa visual.
3. **AA nos componentes base:**
   - `:focus-visible` global com contorno de 3px em `--focus` e afastamento de 2px.
   - `Tag` e `StatusDot` passam a indicar risco ou estado por **cor + forma + texto**: ■ alto, ▲ moderado, ● baixo. Hoje a indicação é só por cor.
   - O texto secundário usa `--ink2` ou `--ink3` com pelo menos 4,5:1, e as bordas de controle usam `--rule2` com pelo menos 3:1.
4. **Divergência visual entre as cópias** (DataTable, KpiGrid, Sparkline, StatTile, PageHeader, BarMini, Panel e AppHeader):
   - Cada app continua com a sua cópia, como o ADR 0002 manda.
   - Os valores de estilo das duas cópias passam a usar os mesmos tokens.
   - **A lógica de cada cópia não é tocada.** Se a diferença entre as cópias for de comportamento (uma coluna, um prop, um texto a mais), ela continua como está e é listada no plano.

**Fora deste subprojeto:**
- seletor de tema, cards e sub-cards, trilho, gaveta, layout de 768px, barra eMAG e alto contraste em uso (subprojeto 2);
- wpda e maintenance, que continuam com o `src/theme/` atual até os subprojetos 4 e 5;
- qualquer mudança no api.

## 5. Testes e critério de pronto

**Garantia de funcionalidade:** as suítes atuais do dashboard e do admin precisam passar **sem nenhuma asserção editada**. Se um teste quebrar, a mudança de interface está errada, não o teste.

**Testes novos:**
- **contracts:**
  - contraste de todos os pares em todos os temas;
  - o gerador é determinístico e o `dist/` versionado é igual à saída do gerador.
- **dashboard e admin:**
  - a cópia em `vendor/` bate com o `tokens.lock`;
  - o `index.html` declara `data-theme="govbr"`;
  - `Tag` e `StatusDot` exibem forma e texto além da cor, para cada nível de risco ou estado.

**Conferência visual (não automatizada):**
- O Playwright captura, antes e depois, as telas principais em 1280px: Visão geral, Triagens, Atendimento, Protocolos e Equipe no dashboard, e Visão geral e Cidades no admin.
- As capturas são para aprovação do usuário. Elas não viram asserção, porque a aparência muda de propósito.

**Pronto quando:**
1. a suíte do contracts está verde e a tag `tokens-v1.0.0` foi criada e publicada;
2. as suítes do dashboard e do admin estão verdes sem asserção editada;
3. o usuário aprovou as capturas antes e depois;
4. os commits do contracts, do dashboard e do admin foram publicados, cada app com o `tokens.lock` apontando para `tokens-v1.0.0`.

## 6. Riscos

- **Licença da Rawline:** pode impedir que a fonte seja embarcada. A reserva está definida na seção 4.
- **Estilo inline com valor fixo:** se algum componente usa uma cor fixa em vez de variável, ele não muda de tema. O plano faz um levantamento com `grep` de hex, `rgb(` e `oklch(` fora de `src/theme/` e converte o que encontrar para papéis.
- **Corpo de 13px para 14px:** pode empurrar layouts densos e gerar quebra de linha ou corte. Isso é conferido nas capturas. Ajustes de espaçamento são permitidos, mudança de estrutura não.
- **Sessões paralelas:** o dashboard e o admin têm sessões donas. Avisar essas sessões antes de mexer, conforme a regra do grupo de sessões por app.
