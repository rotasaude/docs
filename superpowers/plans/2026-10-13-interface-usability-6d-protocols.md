# Ciclo de interface — subprojeto 6, fatia 6d: protocolos — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Corrigir os achados PR-01 a PR-17 do percurso "protocolos" no dashboard (lista, detalhe e editor) e no maintenance (aba Protocolos). Fechar também as pendências A-21 a A-28 do maintenance.

**Architecture:** São dois worktrees e quatro frentes:
- no dashboard, o painel de detalhe (diálogo, ações, assinaturas e eventos), a lista, a confirmação com código e o editor;
- no maintenance, a aba Protocolos e as pendências do subprojeto 5.

As peças da 6a já existem no dashboard: `useDialog`, `useReveal`, `ActionMessage` e `dangerButtonStyle`. No maintenance, o que for preciso é criado localmente e fica pequeno.

**Tech Stack:** React 18, Vite 5, TypeScript, Vitest 2, Testing Library e Playwright (e2e do maintenance).

**Spec:** `docs/superpowers/specs/2026-10-12-interface-usability-design.md` (§4.3). Os achados estão em `rota-saude/evidencias/2026-10-12-usabilidade/protocolos/achados.md`, e as pendências A em `evidencias/2026-10-12-usabilidade/pendencias-A.md`.

## Já resolvido e ajustes de escopo

- **PR-03 (dashboard):** o texto de consequência por ação já existe desde a 6a. Esta fatia só passa a nomear a versão que sai de uso ao Ativar.
- **PR-11 vira card** (board #2): o api não manda quem ativou, o motivo da reversão, a versão-alvo nem a finalidade da assinatura. No front, esta fatia traduz o nome do evento e troca o UUID pelo e-mail que o painel já conhece.
- **PR-13:** mesma ordem nos dois apps (por nome; dentro do nome, versão decrescente; aposentadas no fim), destaque de "em uso", filtro "esconder aposentadas" e busca por nome. Não há agrupamento com linha mestra.
  - Ruling: agrupar reestrutura a lista e o modo cartão; a ordem e o filtro resolvem achar "o que está em uso".
- **PR-10:** o link leva à lista de Protocolos já filtrada pelo nome salvo, via a busca nova. Não abre o painel de detalhe direto: o estado do painel é interno da tela.

## Global Constraints

- **Escopo:**
  - só front-end, sem mudar regra;
  - nada de API, GraphQL, dados ou permissões.
- **Testes:**
  - as suítes existentes passam;
  - uma asserção só muda quando o comportamento muda de propósito, sempre com uma linha `Ruling:` no ledger que cite o PR ou o A;
  - todo item corrigido ganha um teste que falha antes da correção.
- **e2e do maintenance:** passa (`npx playwright test`) contra o worktree, com o contorno de Origin da 5ª fatia.
- **Cores:** só por papel do tema. No maintenance, as guardas de cor do `guards.test.ts` continuam valendo.
- **Commits e git:**
  - Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - `/opt/homebrew/bin/git` e caminhos explícitos no `add`.
- **Worktrees:** `rota-saude/.worktrees/dashboard-ui6d` e `rota-saude/.worktrees/maintenance-ui6d`, no branch `ui/usability-6d` a partir de `origin/main`, com `npm ci`.

## Review Focus

1. **Revisora com a confirmação aberta no painel.**
   - Esperado: a versão e o histórico continuam visíveis e roláveis.
   - Esperado: Escape fecha a confirmação e depois o painel.
   - Prova: Task 1.
2. **Ativar v2 com v1 em uso.**
   - Esperado: o texto diz "v1 (em uso) sai de uso".
   - Esperado: sem nenhuma em uso, o texto não inventa versão.
   - Prova: Task 1 (dashboard) e Task 5 (maintenance).
3. **Mantenedor rolado até a 20ª linha clica em Ativar.**
   - Esperado: a confirmação aparece à vista, com o foco no código.
   - Esperado: ao concluir, a mensagem recebe o foco.
   - Prova: Task 5.
4. **JSON com vírgula faltando na linha 5.**
   - Esperado: "linha 5, coluna 3: …" em português, e Salvar bloqueado com o motivo.
   - Prova: Task 4.
5. **Carregar uma versão em uso no editor.**
   - Esperado: aviso de que ela não se edita, e o botão "Criar nova versão a partir desta" com a versão seguinte.
   - Prova: Task 4.

---

### Task 0: Preparação

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
for app in dashboard maintenance; do
  $G -C $R/apps/$app fetch -q && $G -C $R/apps/$app worktree add -q $R/.worktrees/$app-ui6d -b ui/usability-6d origin/main
  (cd $R/.worktrees/$app-ui6d && npm ci --silent && npx vitest run 2>&1 | tail -3)
done
```

  - As capturas "antes" são as do percurso, em `evidencias/2026-10-12-usabilidade/protocolos/`.

---

### Task 1: Painel de detalhe do protocolo (dashboard: PR-01, PR-02, PR-03/05, PR-06, PR-12, PR-16)

**Files:** `src/modules/Protocols.tsx` e `src/modules/Protocols.test.tsx`. Use o harness `renderProtocols`/`stubReads`/`row`/`versionRow` do arquivo.

- [ ] **Step 1: testes que falham.** Num `describe("detalhe — usabilidade (6d)")`:
  - **PR-06:**
    - o painel tem `role="dialog"` com nome "Detalhe do protocolo" (`aria-labelledby`);
    - ao abrir, o foco fica dentro dele;
    - Escape fecha e o foco volta para a linha que abriu.
  - **PR-02:** com a confirmação aberta, ela é renderizada logo depois da versão escolhida, dentro do painel "Versões", e o painel "Eventos" continua no DOM.
  - **PR-03/05:** com v1 `active` e v2 `published` com as ativações completas, a confirmação de "Ativar" diz "A versão 1 (em uso) sai de uso".
  - **PR-01:**
    - versão `active`: em vez de "Ativação: ninguém assinou ainda · falta 2", aparece "Ativação: concluída — versão em uso";
    - na lista, a coluna Ativação da linha em uso mostra "em uso", e a de rascunho ou em revisão mostra "—".
  - **PR-12:** "Aposentar" e "Reverter" usam o estilo de perigo (`style.color === "var(--down)"`) e ficam por último na linha de ações.
  - **PR-16:** versão `active` sem `revertible` mostra "sem versão anterior para reverter; aposentar não vale para a versão em uso".

- [ ] **Step 2: implementar.**
  - **Diálogo (PR-06):**
    - o painel interno usa `useDialog(true, onClose, opener)`, onde `opener` é o `document.activeElement` capturado ao abrir (estado em `ProtocolVersions`);
    - o `h2` ganha `id="protocol-detail-title"` e o diálogo `aria-labelledby` para ele;
    - o fundo deixa de usar `rgba(...)`: passa a `color-mix(in srgb, var(--ink) 32%, transparent)`.
  - **Layout (PR-02):** o contêiner rolável vira `display: block`, e o conteúdo fica num filho `display: flex; flexDirection: column; gap: 16`, para que os painéis não encolham. O `SensitiveAction` passa a ser renderizado dentro do mapa de versões, logo após a `VersionDetail` cuja versão é `pending.version`.
  - **Consequência (PR-03/05):** `consequence(kind, revertTarget, inUse)`. `inUse` é a versão `active` do mesmo nome, se houver e for diferente da que se ativa. O texto de `activate` passa a "A versão passa a valer para toda a cidade. A versão ${inUse} (em uso) sai de uso." quando há `inUse`, e mantém o texto de hoje quando não há.
  - **Assinaturas (PR-01):** `SignaturesSummary` recebe o `status`.
    - Para `Ativação` em `active` ou `retired`, mostra "concluída — versão em uso" ou "—".
    - Para `Publicação` em `draft`, mostra "—".
    - Na lista, a coluna Ativação: `active` → "em uso"; `draft`/`in_review` → "—"; os demais, `n/2`.
  - **Ações (PR-12):** em `VersionActions`, ordene com `retire`/`revert` por último e use `dangerButtonStyle` neles (`disabledButtonStyle` quando bloqueados).
  - **Sem reversão (PR-16):** quando `status === "active" && !revertible`, mostre o `<small>` com a frase acima.
  - **Mensagem de status:** o `done` vira `<p ref tabIndex={-1} role="status">`, com o foco quando muda, como no `ConsultationDocuments`.
  - **Testes existentes** que dependem do texto "falta 2" em versão ativa ou do "0/2" na lista mudam (Ruling PR-01).

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit`
  Expected: verde.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git commit -m "feat: accessible protocol detail dialog with inline confirmation, clear signature state and danger styling

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Lista e eventos (dashboard: PR-10 busca, PR-11 front, PR-13, PR-14)

**Files:** `src/modules/Protocols.tsx`, `src/modules/Protocols.test.tsx` e `src/lib/protocolEvents.ts` (novo, com teste).

**Interfaces — Produces:**
- `eventLabel(name: string): string`. Mapeia:

| Evento | Frase |
|---|---|
| `protocol.draft_saved` | rascunho salvo |
| `protocol.submitted` | enviada para revisão |
| `protocol.signed` | assinatura registrada |
| `protocol.published` | publicada |
| `protocol.activated` | ativada |
| `protocol.retired` | aposentada |
| `protocol.activation_reverted` | ativação revertida |

  Nome desconhecido sai cru.
- `actorLabel(actor: string | null, people: { id: string; email: string | null; kind?: string }[]): string`:
  - `null` → "sistema";
  - id conhecido → e-mail ("mantenedor" quando `kind === "maintainer"`);
  - desconhecido → "pessoa da equipe".

- [ ] **Step 1: testes que falham.**
  - **`protocolEvents.test.ts`:** a tabela acima, mais `actorLabel` nos três casos.
  - **`Protocols.test.tsx`:**
    - **PR-13:**
      - a lista vem ordenada por nome e, dentro do nome, por versão decrescente, com aposentadas no fim;
      - a caixa "esconder aposentadas" tira as aposentadas;
      - a busca "Buscar por nome" filtra.
    - **PR-14:**
      - o KPI se chama "Publicados ou em uso";
      - as mensagens de sucesso viram "Publicação assinada: dengue v1", "Publicada: …", "Ativada: …", "Aposentada: …" e "Enviada para revisão: …", e a de reversão continua como hoje.
    - **PR-11:** a tabela de eventos mostra "assinatura registrada" e o e-mail do signatário, não o UUID.
    - **PR-10:** com o endereço `?protocolo=dengue#/protocols`, o filtro de busca já vem "dengue".
      - O parâmetro fica na query, antes do `#`, porque o roteador preserva `location.search`.
      - Leia com `new URLSearchParams(window.location.search).get("protocolo")` na montagem.

- [ ] **Step 2: implementar.**
  - **Ordem e filtros:**
    - ordene `list` com `localeCompare` por id e depois `Number(b.version) - Number(a.version)`, com as aposentadas no fim;
    - `const [ hideRetired, setHideRetired ] = useState(false)` e `const [ query, setQuery ] = useState(initialFromUrl)`;
    - os controles (`<label><input type="checkbox"/> esconder aposentadas</label>` e `<label>Buscar por nome<input/></label>`) ficam acima da tabela.
  - **Eventos:**
    - Evento: `eventLabel(e.name)`;
    - Actor: `actorLabel(e.actor, pessoas)`, com `pessoas` vindo dos `signers` e `editors` de todas as versões do detalhe.
  - **KPI e mensagens:** o KPI é renomeado; `doneMessage` usa as frases no particípio.
  - **Testes existentes** que esperam "Assinar publicação concluído: dengue v1" mudam (Ruling PR-14).

  Run: `npx vitest run 2>&1 | tail -3`
  Expected: verde.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git commit -m "feat: ordered, filterable protocol list with readable events and outcome messages

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Confirmação com código e motivo (dashboard: PR-15, PR-17)

**Files:** `src/components/SensitiveAction.tsx` e `src/components/SensitiveAction.test.tsx`.

- [ ] **Step 1: testes que falham.**
  - **PR-15:**
    - o campo "Código do autenticador" tem `inputMode="numeric"` e `maxLength=6` e recebe o foco ao abrir;
    - abaixo dele aparecem "Se acabou de entrar, espere o próximo código." e "Tentativas erradas contam para o bloqueio da conta.".
  - **PR-17:** um campo com `multiline: true` e `placeholder` vira `<textarea>` com o `placeholder`.

- [ ] **Step 2: implementar.**
  - `SensitiveField` ganha `multiline?: boolean` e `placeholder?: string`.
  - No input do código, adicione os atributos e o `autoFocus`, com os dois textos de ajuda.
  - Em `Protocols.tsx`, o campo do motivo da reversão passa a `{ name: "reason", label: "Motivo", required: true, multiline: true, placeholder: "ex.: v2 classificou febre alta como baixa prioridade" }`.

  Run: `npx vitest run 2>&1 | tail -3`
  Expected: verde.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git commit -m "feat: numeric autofocused step-up code with help, multiline reason field

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Editor de protocolo (dashboard: PR-07, PR-08, PR-09, PR-10)

**Files:**
- `src/modules/ProtocolEditor.tsx` e o teste (procure com `ls src/modules | grep -i editor`)
- `src/lib/editorMessages.ts` (novo, com teste)

**Interfaces — Produces:**
- `parseErrorMessage(raw: string): string`. Quando há `(line L column C)`, devolve "linha L, coluna C: " mais a causa:
  - "Expected ',' or '}'" vira "falta vírgula ou fechar chave";
  - "Unexpected token" vira "caractere inesperado";
  - outro caso vira "formato inválido".

  Sem linha, devolve "JSON inválido".
- `gateErrorMessage(raw: string): string`:
  - `schema: /steps/N/campo motivo` vira "Pergunta N+1: <campo> <motivo em PT>":
    - campo: `answer_type` → "tipo de resposta"; `prompt` → "texto"; `id` → "identificador"; `branches` → "caminhos"; `weights` → "pesos"; outro, cru;
    - motivo: `enum` → "inválido (use boolean, enum, integer ou text)" quando o campo é `answer_type`, senão "fora das opções"; `required` → "obrigatório"; `type` → "com tipo errado".
  - `Definition start_step_id refers to unknown step` vira "start_step_id aponta para uma pergunta que não existe".
  - Desconhecido sai cru.

- [ ] **Step 1: testes que falham.**
  - **`editorMessages.test.ts`:** os casos acima, com as três mensagens reais do percurso.
  - **Editor:**
    - **PR-07:**
      - "Salvar rascunho" e "Pré-visualizar" têm o estilo de botão (`style.background` não vazio);
      - o seletor tem o rótulo "Carregar versão";
      - a área tem o rótulo "Definição JSON";
      - há um `h1` "Editor de protocolo".
    - **PR-08:**
      - JSON inválido mostra a frase traduzida;
      - "Salvar rascunho" fica `disabled`, com o motivo "corrija os erros antes de salvar" ligado por `aria-describedby`.
    - **PR-09:**
      - as opções do seletor mostram o status em português ("em uso", "em revisão");
      - ao carregar uma versão que não é rascunho, aparece "Esta versão não se edita" e o botão "Criar nova versão a partir desta";
      - clicar no botão põe `version` igual à maior versão daquele nome mais 1 no JSON.
    - **PR-10:** depois de salvar, aparece o link "Ver no ciclo de vida →" com `href` `?protocolo=<nome>#/protocols`. Monte o href preservando o `pathname`.

- [ ] **Step 2: implementar.**
  - **Cabeçalho:** use `PageHeader` com o título "Editor de protocolo", como as outras telas.
  - **Botões e rótulos:**
    - aplique `buttonStyle`/`secondaryButtonStyle`;
    - envolva o select e o textarea num `<label>`;
    - os `color: var(--danger, #c00)` passam a `var(--down)` (sem reserva).
  - **Mensagens e Salvar:** `parseErrorMessage`/`gateErrorMessage` nas listas; Salvar fica `disabled={!valid}`, com o `<small id>` do motivo.
  - **Versão não editável:** guarde o status da versão carregada; em `!== "draft"`, mostre o aviso e o botão, que ajusta o JSON.
  - **Status nas opções:** use o mesmo mapa de rótulos de `Protocols.tsx`. Exporte `statusLabel` de um módulo comum (`src/lib/protocolStatus.ts`) e use-o nos dois arquivos.
  - **Link:** depois de `saved.status`, um `<a href={...}>Ver no ciclo de vida →</a>`.
  - **Testes existentes** que procuram "JSON inválido: …" cru ou a mensagem inglesa do gate mudam (Ruling PR-08).

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && npm run build`
  Expected: verde.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git commit -m "feat: protocol editor with labelled controls, translated errors, new-version helper and link to the lifecycle

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Aba Protocolos do maintenance (PR-01, PR-04, PR-05, PR-12, PR-13, PR-14, PR-16, PR-17)

**Files:**
- `src/screens/ProtocolsTab.tsx`
- `src/components/Button.tsx`, `src/components/Field.tsx` e `src/components/Tag.tsx`
- os testes de cada um (o da aba: `grep -ln "ProtocolsTab" src/screens/*.test.tsx`)

- [ ] **Step 1: testes que falham.**
  - **PR-04:**
    - ao abrir uma ação, `scrollIntoView` é chamado no painel de confirmação e o foco vai para o primeiro campo (código ou motivo) ou para o título quando não há campo;
    - ao concluir, a mensagem `role="status"` recebe o foco.
  - **PR-05:** "Ativar" de v2 com v1 `active` mostra "A cidade passa a usar a versão 2; a versão 1 (em uso) sai de uso.".
  - **PR-12:** o botão "Aposentar" e o "Reverter" usam `variant="danger"`, com borda `var(--down)` e texto `var(--down)`.
  - **PR-13:**
    - a ordem é por nome e depois por versão decrescente, com aposentadas no fim;
    - a caixa "esconder aposentadas" filtra;
    - o tag "em uso" tem tom `accent`.
  - **PR-14:**
    - o status `active` se chama "em uso";
    - as mensagens de sucesso no particípio são as mesmas frases do dashboard;
    - a reversão segue as frases de hoje.
  - **PR-16:** uma linha `active` sem `revertible` mostra o texto curto "sem versão anterior para reverter".
  - **PR-17:** o motivo da reversão é um `<textarea>` com o `placeholder` de exemplo.
  - **PR-01:** a coluna Ativação mostra "em uso" na linha ativa e "—" em rascunho ou em revisão.

- [ ] **Step 2: implementar.**
  - **`Button`:** ganha `variant?: "primary" | "secondary" | "danger"`. O `secondary` usa fundo `var(--panel)`, texto `var(--ink)` e borda `var(--rule2)`; o `danger` usa fundo `var(--panel)`, texto `var(--down)` e borda `var(--down)`. O desabilitado continua tracejado.
  - **`Field`:** ganha `multiline?: boolean` e `placeholder?: string`.
  - **`ProtocolsTab`:**
    - a confirmação (o `Panel`) fica num `<section ref>` com um efeito que rola até ela e foca o primeiro input, textarea ou título;
    - o `done` ganha `ref`, `tabIndex={-1}` e foco quando muda;
    - o texto de `activate` é derivado das linhas;
    - a ordem, o filtro e o tom, como no dashboard;
    - `STATUS_LABELS.active = "em uso"`;
    - `doneMessage` usa o particípio;
    - o texto de "sem reversão" e o motivo com `multiline`.
  - **Testes e e2e existentes:**
    - testes que esperam "ativa" ou "Enviar para revisão concluído: …" mudam (Ruling PR-14);
    - o e2e `e2e/maintenance.spec.ts` espera "Enviar para revisão concluído: …" e "Aposentar concluído: …": ajuste para as frases novas (Ruling PR-14) e rode o e2e.

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && npm run build && npm run codegen:check`
  Expected: verde.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git commit -m "feat: maintenance protocol tab reveals confirmations, names the version leaving use and styles destructive actions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Pendências do maintenance (A-21 a A-28)

**Files:**
- `src/components/TableScroll.tsx` e o teste
- `src/theme/global.css`
- `src/screens/Shell.tsx` e `Shell.rail.test.tsx`
- `src/theme/guards.test.ts`

- [ ] **Step 1: testes que falham.**
  - **A-21:** com transbordo, o `TableScroll` tem `tabIndex=0`, `role="region"` e `aria-label="tabela com rolagem lateral"`. Para simular, defina `scrollWidth` e `clientWidth` com `Object.defineProperty` e chame o callback do `ResizeObserver` (stub no teste).
  - **A-22:** o `global.css` tem `.table-scroll.is-overflowing::before` com a dica.
  - **A-28:** o `content` da dica tem texto alternativo vazio (`/ ""`).
  - **A-23:** o `global.css` tem `#root { display: flex; flex-direction: column; }` e o Shell usa `flex: 1` em vez de `minHeight: "100%"`.
  - **A-24:** no detalhe da cidade, clicar em "Cidades" no trilho volta ao catálogo. Use o mock do `Shell.test`.
  - **A-25:** a guarda passa a varrer o `global.css`, fora do bloco `:root` da faixa, contra hex, `oklch` e `outline: none`. A regex de hex passa a aceitar 3 a 8 dígitos e aspas simples ou crase, e a de outline pega `outline(Width)?:\s*["']?(none|0)`.
  - **A-26:** o `ResizeObserver` observa também o filho (`observe` chamado 2 vezes).
  - **A-27:** o "sair" usa `variant="secondary"`.

- [ ] **Step 2: implementar** o que os testes pedem. No `TableScroll`, `tabIndex`, `role` e `aria-label` só valem enquanto há transbordo; guarde isso em estado.

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && npm run build`
  Expected: verde.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git commit -m "fix: maintenance pending items (keyboard scroll region, top hint, page height, stricter guards, sign-out contrast)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: e2e, capturas, revisão, card e publicação

- [ ] **e2e do maintenance** contra o worktree (Vite 5198, contorno de Origin com `extraHTTPHeaders` como na 5ª fatia).
- [ ] **Capturas "depois":**
  - **dashboard:** lista, painel com confirmação de Ativar, eventos e editor com erro;
  - **maintenance:** aba Protocolos com a confirmação à vista;
  - use o protocolo de teste `u6-teste-193133` de Curitiba, sem tocar nos protocolos reais.
- [ ] **Revisão final** pelo revisor mais capaz:
  - corrija os achados Critical e Important com TDD;
  - registre-os no ledger.
- [ ] **Aprovação do usuário** com `evidencias/2026-10-12-usabilidade/6d/comparacao.html`.
- [ ] **Card PR-11, com permissão:** quem ativou, o motivo e a versão-alvo da reversão, e a finalidade da assinatura no payload dos eventos de protocolo.
- [ ] **Merge e push:**
  - avise as sessões que mexem no dashboard e no maintenance;
  - confira `origin/main..main`;
  - faça `merge --ff-only` e `push` do dashboard, do maintenance e do docs;
  - remova os worktrees;
  - rode o e2e contra o 5177 depois do merge.
