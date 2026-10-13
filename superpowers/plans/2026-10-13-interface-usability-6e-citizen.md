# Ciclo de interface — subprojeto 6, fatia 6e: cidadão (wpda) — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Corrigir os achados WP-01 a WP-18 do percurso "cidadão: triagem no wpda" e as pendências A-13 a A-20, só no front do wpda.

**Architecture:** Um worktree, `wpda`, e cinco frentes:
- a moldura de toda tela (`Screen`, `EmergencyCall`, `Report`);
- o resultado (`ResultStep`);
- os formulários (`Field`, `RadioGroup`, perfil, CPF, celular, código, bairro);
- as telas de código e de histórico;
- a barra e a página de acessibilidade, mais o portão de celular.

Não há peças da 6a no wpda (são do dashboard). O que for preciso aqui nasce pequeno em `modules/citizen/ui.tsx`: só `ConfirmInline`, usado em duas telas.

**Tech Stack:** React 18, Vite 5, TypeScript, Vitest 2, Testing Library (jsdom) e Playwright (5 perfis de dispositivo).

**Spec:** `docs/superpowers/specs/2026-10-12-interface-usability-design.md` (§4.4). Os achados estão em `rota-saude/evidencias/2026-10-12-usabilidade/wpda/achados.md`, e as pendências A em `evidencias/2026-10-12-usabilidade/pendencias-A.md`.

## Já resolvido e ajustes de escopo

- **WP-06 dividido:**
  - pular o termo já aceito exige saber se há consentimento vigente, e **vira card** (board #2);
  - no front, depois do termo, a pessoa volta direto ao catálogo de quem estava fazendo a triagem, onde "Em andamento — Continuar" já aparece. O `sessionStorage` guarda só o `citizenId` (UUID): nada de resposta, CPF ou dado clínico. Some em "Sair", na sessão vencida e quando a triagem termina.
  - Ruling: o identificador não é dado clínico nem pessoal legível; o ganho é um toque a menos e a pessoa ver que não perdeu nada.
- **WP-09 dividido:** apelido por pessoa **vira card**. No front, o botão mostra o que `/citizen/people` já traz: "Nascido(a) em dd/mm/aaaa · Bairro".
- **WP-04 "UBS":** é texto do protocolo, fica com a curadoria. Fora desta fatia.
- **WP-13 duplicidade de dengue:** é dado do catálogo, fica com a curadoria. Aqui só os nomes acessíveis.
- **WP-14 toque acidental grava o bairro:** confirmar o toque seria passo novo no fluxo. A lista deixa de aparecer inteira sem busca, o que já tira a maior parte dos toques ao rolar.
  - Ruling: sem confirmação extra; reavaliar na interface se aparecer de novo.
- **A-15:** o glifo do nível usa `content: "■" / ""`. Navegador sem suporte ao texto alternativo descarta a declaração inteira e o glifo some. A correção declara antes a forma simples `content: "■"` como reserva.
- **A-16:** celular dobrável aberto tem o lado menor ≥ 600 px e cai no "Abra no celular". O portão passa a aceitar também quem se declara celular (`navigator.userAgentData.mobile` ou "Android … Mobile" no UA), sempre com toque como entrada principal. iPad e tablets Android continuam bloqueados.

## Global Constraints

- **Escopo:**
  - só front-end, sem mudar regra;
  - nada de API, dados ou permissões;
  - nada clínico, nem CPF, guardado no navegador.
- **Testes:**
  - as suítes existentes passam;
  - uma asserção só muda quando o comportamento muda de propósito, sempre com uma linha `Ruling:` no ledger que cite o WP ou o A;
  - todo item corrigido ganha um teste que falha antes da correção.
- **e2e do wpda:** `npm run e2e` passa nos 5 perfis (iphone, pixel, ipad, desktop-chrome, desktop-firefox).
- **Telas:** alvos de toque ≥ 48 px e texto ≥ 16 px (18 px no conteúdo), como o resto do wpda.
- **Cores:** só por papel do tema. As guardas de `theme/guards.test.ts` continuam valendo.
- **Commits e git:**
  - Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - `/opt/homebrew/bin/git` e caminhos explícitos no `add`.
- **Worktree:** `rota-saude/.worktrees/wpda-ui6e` (nunca dentro de `apps/wpda`), no branch `ui/usability-6e` a partir de `origin/main`, com `npm ci`.

## Review Focus

1. **Celular 360x640 com o teclado aberto no campo de CPF.**
   - Esperado: a faixa do 192 não cobre o campo nem o botão Continuar; o fim da tela rola acima dela.
   - Prova: Task 1 (o `main` reserva o espaço da faixa) e as capturas.
2. **Leitor de tela respondendo duas perguntas seguidas com o mesmo texto.**
   - Esperado: o foco volta ao `h1` também na segunda, porque o índice mudou.
   - Prova: Task 1 (`focusKey`) e Task 5.
3. **Resultado que desiste e depois chega.**
   - Esperado: "Ver meu resultado de novo" reinicia a consulta; quando o relatório chega, aparecem as ações, sem duas consultas correndo juntas.
   - Prova: Task 3.
4. **Telefone "voltar" com a página de acessibilidade aberta, e depois "voltar" de novo.**
   - Esperado: o primeiro fecha a página e o cidadão continua no mesmo passo; o segundo segue o histórico normal do navegador.
   - Prova: Task 8.
5. **Código de chegada que vence com a confirmação de "Gerar outro código" aberta.**
   - Esperado: a confirmação some e "Gerar outro código" vira a ação principal, sem pedir confirmação.
   - Prova: Task 7.

---

### Task 0: Preparação

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda fetch -q && $G -C $R/apps/wpda worktree add -q $R/.worktrees/wpda-ui6e -b ui/usability-6e origin/main
(cd $R/.worktrees/wpda-ui6e && npm ci --silent && npx vitest run 2>&1 | tail -3)
```

  - Expected: suíte verde (linha de base anotada no ledger).
  - As capturas "antes" são as do percurso, em `evidencias/2026-10-12-usabilidade/wpda/`.

---

### Task 1: Moldura de toda tela (WP-01, A-17, WP-02, WP-17 do 192)

**Files:** `src/modules/citizen/ui.tsx`, `src/modules/citizen/EmergencyCall.tsx`, `src/test/setup.ts`, `src/modules/citizen/chrome.test.tsx`.

**Interfaces:**
- Produces: `Screen({ title, children, footer, focusKey? })`, em que `focusKey?: string | number` repete o foco no `h1` quando muda sem o título mudar. `EmergencyCall` passa a ser uma faixa fixa; quem a usa não muda.

- [ ] **Step 1: testes que falham** (em `chrome.test.tsx`, `describe("moldura — usabilidade (6e)")`):

```tsx
it("192 é uma faixa fixa no rodapé, com borda (WP-01, A-17, WP-17)", () => {
  render(<Screen title="T">x</Screen>);
  const link = screen.getByRole("link", { name: "Emergência? Ligar 192" });
  const bar = link.closest("[data-emergency-bar]") as HTMLElement;
  expect(bar.style.position).toBe("fixed");
  expect(bar.style.bottom).toBe("0px");
  expect(link.style.border).toBe("2px solid var(--down)");
});

it("o main reserva o espaço da faixa (WP-01)", () => {
  render(<Screen title="T">x</Screen>);
  expect(screen.getByRole("main").style.paddingBottom).toBe("calc(96px + env(safe-area-inset-bottom))");
});

it("o h1 recebe o foco ao montar e quando o título muda (WP-02)", () => {
  const { rerender } = render(<Screen title="Um">x</Screen>);
  expect(document.activeElement).toBe(screen.getByRole("heading", { level: 1, name: "Um" }));
  screen.getByRole("heading", { level: 1 }).blur();
  rerender(<Screen title="Dois">x</Screen>);
  expect(document.activeElement).toBe(screen.getByRole("heading", { level: 1, name: "Dois" }));
});

it("focusKey novo devolve o foco com o mesmo título (WP-02)", () => {
  const { rerender } = render(<Screen title="Igual" focusKey={1}>x</Screen>);
  screen.getByRole("heading", { level: 1 }).blur();
  rerender(<Screen title="Igual" focusKey={2}>x</Screen>);
  expect(document.activeElement).toBe(screen.getByRole("heading", { level: 1 }));
});

it("rola ao topo na troca de tela (WP-02)", () => {
  const scrollTo = vi.spyOn(window, "scrollTo");
  render(<Screen title="T">x</Screen>);
  expect(scrollTo).toHaveBeenCalledWith(0, 0);
});
```

  Run: `npx vitest run src/modules/citizen/chrome.test.tsx`
  Expected: FAIL nos cinco.

- [ ] **Step 2: implementar.**
  - `test/setup.ts`: `window.scrollTo = () => {};` dentro do `beforeEach` (o jsdom não implementa e polui a saída).
  - `EmergencyCall`:

```tsx
// Faixa fixa no rodapé (WP-01): sempre à vista, inclusive com teclado fechado
// em 360x640. Borda de 2 px: no alto contraste o fundo some e o 192 continua
// com forma de botão (WP-17).
export function EmergencyCall() {
  return (
    <div data-emergency-bar="" style={{ position: "fixed", left: 0, right: 0, bottom: 0, zIndex: 10,
      background: "var(--bg)", borderTop: "1px solid var(--rule)",
      padding: "8px 16px calc(8px + env(safe-area-inset-bottom))" }}>
      <a href="tel:192" style={{ display: "flex", alignItems: "center", justifyContent: "center", gap: 8, minHeight: 56,
        maxWidth: 520, margin: "0 auto", borderRadius: 12, background: "var(--down-bg)", color: "var(--down)",
        border: "2px solid var(--down)", fontSize: 18, fontWeight: 700, textDecoration: "underline" }}>
        {/* svg de hoje */}
        Emergência? Ligar 192
      </a>
    </div>
  );
}
```

  - Confira que `--bg` existe no tema do wpda (`theme/vendor`); se o papel do fundo da página tiver outro nome, use-o (Ruling se mudar).
  - `Screen`: `h1` com `ref` e `tabIndex={-1}`; `useEffect(() => { window.scrollTo(0, 0); h1.current?.focus({ preventScroll: true }); }, [title, focusKey])`; `paddingBottom: "calc(96px + env(safe-area-inset-bottom))"` no `main` (faixa de 56 + 2×8 + borda, com folga).
  - O teste existente "Screen traz o topo da cidade com reserva e o 192" continua passando sem mudança.

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit`
  Expected: verde.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git add src/modules/citizen/ui.tsx src/modules/citizen/EmergencyCall.tsx src/test/setup.ts src/modules/citizen/chrome.test.tsx
/opt/homebrew/bin/git commit -m "feat: fixed emergency bar and focus on the screen title on every step

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Relatório (WP-04, WP-17 do selo e do alinhamento, A-15)

**Files:** `src/modules/Report.tsx`, `src/modules/Report.test.tsx`, `src/modules/Report.tier.test.tsx`, `src/theme/global.css`, `src/theme/guards.test.ts`.

**Interfaces:**
- Produces: `Report({ token, children, signedIn? })`. Com `signedIn`, o relatório omite "Válido até" e o aviso do link e se alinha ao topo. `ResultStep` (Task 3) passa `signedIn`.

- [ ] **Step 1: testes que falham:**
  - **WP-04:**
    - com `tier: "alta"`, existe o texto "Seu resultado: prioridade alta" num elemento com `fontSize` ≥ 18 (`parseInt(style.fontSize) >= 18`), com `data-glyph="square"`;
    - com `priority: 3`, nenhum texto contém "prioridade: 3" (`queryByText(/prioridade: \d/)` é `null`).
  - **WP-04 (área logada):** com `signedIn`, `queryByText(/Válido até/)` e `queryByText(/não compartilhe este link/)` são `null`, e "Realizado em" continua. Sem `signedIn` (link público), os três continuam.
  - **WP-17:**
    - o selo tem `border: 2px solid var(--down)` para o tom `down` (e `var(--accent)` sem tom);
    - com `signedIn`, o `main` tem `justifyContent: "flex-start"`; sem, continua `center`.
  - **A-15** (em `guards.test.ts`, lendo `global.css`): para cada glifo, a forma simples vem antes da forma com texto alternativo:

```ts
it("glifo do nível tem reserva sem texto alternativo (A-15)", () => {
  for (const g of ["■", "▲", "●"]) {
    expect(css).toMatch(new RegExp(`content:\\s*"${g}";\\s*content:\\s*"${g}"\\s*/\\s*""`));
  }
});
```

  Run: `npx vitest run src/modules src/theme`
  Expected: FAIL nos novos.

- [ ] **Step 2: implementar.**
  - Selo vira frase: `<p data-glyph=… style={{ fontSize: 20, fontWeight: 700, padding: "8px 12px", borderRadius: 12, border: `2px solid var(--${tone ?? "accent"})`, background: …, color: … }}>Seu resultado: prioridade {r.tier}</p>`. O `h1` "Seu resultado" continua.
  - Remova o `<span>prioridade: N</span>`.
  - `signedIn`: esconde as duas linhas do rodapé e passa `align` ao `Centered` (`justifyContent: signedIn ? "flex-start" : "center"`).
  - `global.css`: `[data-glyph="square"]::before { content: "■"; content: "■" / ""; }`, e o mesmo para os outros dois.
  - Os testes existentes que procuram o selo de 12 px ou "prioridade: N" mudam (Ruling WP-04).

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit`
  Expected: verde.

- [ ] **Step 3: commit** (`feat: larger result level, no internal priority number and logged-in report without link notices`).

---

### Task 3: Tela de resultado (WP-03, WP-05)

**Files:** `src/modules/citizen/ResultStep.tsx`, `src/modules/citizen/ResultStep.test.tsx`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/Flow.test.tsx`.

**Interfaces:**
- Consumes: `Report({ signedIn })` (Task 2); `citizenApi.triage(id)` já devolve `check_in_available` normalizado.
- Produces: `ResultStep({ triageId, onAgain, onHistory, onStartSuggestion, onCheckIn })`, com `onCheckIn: (triageId: string) => void`. O `Flow` liga `onCheckIn` ao estado `check-in-code` que já existe.

- [ ] **Step 1: testes que falham** (com o harness de `ResultStep.test.tsx`, timers falsos):
  - **WP-03, espera:**
    - existe `role="status"` com "Preparando seu resultado…" e um `progressbar` (`<progress>` sem `value`);
    - "Fazer outra triagem", as sugestões ("Fazer agora") e as unidades **não** aparecem;
    - só "Ver minhas triagens e horários" aparece.
  - **WP-03, desistência:** depois de 15 falhas, aparece "Ver meu resultado de novo"; tocar nele chama `citizenApi.triage` de novo e, quando o `report_url` vem, mostra o relatório. Durante a nova espera, o botão some (uma consulta por vez).
  - **WP-03, pronto:** com o relatório, aparecem "Fazer outra triagem", as sugestões e as unidades.
  - **WP-05:**
    - com `check_in_available: true`, há a seção "E agora?" (`heading` nível 2), com "Cheguei na unidade" como botão principal; tocar chama `onCheckIn(triageId)`; "Fazer outra triagem" vira secundário;
    - com `false`, "E agora?" traz a frase sobre horários e "Fazer outra triagem" continua principal;
    - o secundário se chama "Ver minhas triagens e horários".
  - **Flow:** do resultado, "Cheguei na unidade" abre a tela "Cheguei na unidade" (o `issueCheckInCode` é chamado com o `triageId`).

  Run: `npx vitest run src/modules/citizen/ResultStep.test.tsx src/modules/citizen/Flow.test.tsx`
  Expected: FAIL nos novos.

- [ ] **Step 2: implementar.**
  - Estado `attempt` (número). O `useEffect` da consulta depende de `[triageId, attempt]`; "Ver meu resultado de novo" faz `setGaveUp(false); setAttempt(a => a + 1)`.
  - Guarde também `checkIn` (o `check_in_available` da última resposta).
  - Espera:

```tsx
<Screen title="Seu resultado" footer={<BigButton variant="secondary" onClick={onHistory}>Ver minhas triagens e horários</BigButton>}>
  <div role="status">
    <p>{gaveUp ? "Seu resultado ainda está sendo preparado." : "Preparando seu resultado…"}</p>
    {!gaveUp && <progress aria-label="Preparando seu resultado" style={{ width: "100%", height: 8 }} />}
  </div>
  {gaveUp && <BigButton onClick={() => { setGaveUp(false); setAttempt(a => a + 1); }}>Ver meu resultado de novo</BigButton>}
</Screen>
```

  - Pronto: `<Report token signedIn>` com `SchedulingRequestNotice`, `ReferenceUnits`, a seção "E agora?" e as ações. A seção:

```tsx
<section aria-labelledby="next-steps" style={{ margin: "16px 0" }}>
  <h2 id="next-steps" style={{ fontSize: 20, margin: "0 0 8px" }}>E agora?</h2>
  <p>{checkIn
    ? "Se for à unidade de saúde, ao chegar toque em \"Cheguei na unidade\" e mostre o código na recepção."
    : "Seus horários marcados e as triagens anteriores ficam em \"Ver minhas triagens e horários\"."}</p>
  {checkIn && <BigButton onClick={() => onCheckIn(triageId)}>Cheguei na unidade</BigButton>}
</section>
```

  - Ações depois: `recommended`, e então "Fazer outra triagem" (`variant={checkIn ? "secondary" : "primary"}`) e "Ver minhas triagens e horários" (secundário).
  - Os testes existentes que esperam "Fazer outra triagem" ou as sugestões durante a espera, ou o texto "Minhas triagens", mudam (Ruling WP-03/WP-05).

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit`
  Expected: verde.

- [ ] **Step 3: commit** (`feat: result wait without early actions, retry after giving up and a "what now" block with check-in`).

---

### Task 4: Campos e perfil (WP-07, A-18, A-19, WP-10, WP-11 do formulário)

**Files:** `src/modules/citizen/ui.tsx`, `src/lib/profile.ts`, `src/lib/profile.test.ts`, `src/modules/citizen/ProfileForm.tsx`, `src/modules/citizen/ProfileForm.test.tsx`, `src/modules/citizen/PhoneStep.tsx`, `src/modules/citizen/CodeStep.tsx`, e um teste novo `src/modules/citizen/fields.test.tsx`.

**Interfaces:**
- Produces:
  - `Field` passa a aceitar `ref` (`forwardRef<HTMLInputElement, …>`);
  - `RadioGroup` liga o erro ao grupo;
  - `BIRTH_DATE_NONEXISTENT` em `lib/profile`.

- [ ] **Step 1: testes que falham:**
  - **WP-07, celular:** "Receber código" com "4199" mostra o erro com `aria-invalid="true"` no campo "Seu celular", `aria-describedby` apontando para o erro, e o foco no campo. Um `ApiError("invalid_phone")` faz o mesmo. Um `ApiError("too_soon")` continua como `ErrorText` solto (não é do campo).
  - **WP-07, código:** `ApiError("invalid_code")` e `code_expired` marcam o campo "Código" (`aria-invalid="true"`) e focam nele.
  - **A-18:** `RadioGroup` com `error` → o `group` (fieldset) tem `aria-describedby` com o id do erro, e cada rádio tem `aria-invalid="true"`.
  - **A-19:** no perfil, depois de "Continuar" com tudo vazio, digitar a data some com o erro da data (o do sexo fica); escolher o sexo some com o do sexo.
  - **WP-10** (em `profile.test.ts`): `validateProfile({ birthDate: "31/02/1990", … })` dá `errors.birthDate === BIRTH_DATE_NONEXISTENT` ("Essa data não existe. Confira o dia e o mês."); `"02/04/19"` continua dando `BIRTH_DATE_INVALID`.
  - **WP-11, gênero recolhido:**
    - sem valor inicial, os 8 rádios de identidade de gênero não estão no DOM; há o botão "Informar identidade de gênero (opcional)" com `aria-expanded="false"`; tocar mostra os rádios;
    - com `initial.gender_identity` preenchido, o grupo já vem aberto.

  Run: `npx vitest run src/lib/profile.test.ts src/modules/citizen`
  Expected: FAIL nos novos.

- [ ] **Step 2: implementar.**
  - `Field` com `forwardRef`; o resto igual.
  - `RadioGroup`: `const errorId = \`${name}-error\``; `<fieldset aria-describedby={error ? errorId : undefined}>`; cada `input` com `aria-invalid={Boolean(error) || undefined}`; `<ErrorText id={errorId}>`.
  - `lib/profile.ts`:

```ts
export const BIRTH_DATE_NONEXISTENT = "Essa data não existe. Confira o dia e o mês.";
// …em validateProfile:
const complete = /^\d{2}\/\d{2}\/\d{4}$/.test(d.birthDate.trim());
if (!iso) errors.birthDate = complete ? BIRTH_DATE_NONEXISTENT : BIRTH_DATE_INVALID;
```

  - `ProfileForm`:
    - `onChange` da data faz `setErrors(e => ({ ...e, birthDate: undefined }))`, e o do sexo faz o mesmo com `sex`;
    - estado `genderOpen` (inicia em `initial?.gender_identity != null`); fechado, mostra um botão-link `aria-expanded` "Informar identidade de gênero (opcional)"; aberto, o `RadioGroup` de hoje.
  - `PhoneStep` e `CodeStep`: `fieldError` separado de `error`. Erro de validação local e os códigos do campo (`invalid_phone`; `invalid_code`, `code_expired`, `code_exhausted`) vão para `fieldError`, passado em `<Field error>`, e depois `inputRef.current?.focus()`. Os demais continuam em `ErrorText`.

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit`
  Expected: verde.

- [ ] **Step 3: commit** (`feat: field-bound errors, clearer birth date messages and collapsed gender identity`).

---

### Task 5: Para quem é, catálogo e perguntas (WP-08, WP-09 front, WP-11 CPF, A-20, WP-12, WP-13, WP-06 front)

**Files:** `src/modules/citizen/PeopleStep.tsx`, `src/modules/citizen/PeopleStep.test.tsx`, `src/lib/masks.ts`, `src/lib/masks.test.ts`, `src/modules/citizen/CatalogStep.tsx`, `src/modules/citizen/CatalogStep.test.tsx`, `src/modules/citizen/QuestionStep.tsx`, `src/modules/citizen/triage.test.tsx`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/Flow.test.tsx`.

**Interfaces:**
- Consumes: `Field` com `ref` e `error` (Task 4); `Screen({ focusKey })` (Task 1).
- Produces:
  - `hideCpf(cpf: string): string` em `lib/masks` (`"673.959.363-96"` → `"***.959.363-**"`, o mesmo formato de `cpf_masked`);
  - `QuestionStep({ …, onLeave? })`;
  - em `Flow`, `RESUME_KEY = "wpda:resume-citizen"`.

- [ ] **Step 1: testes que falham:**
  - **WP-08:**
    - com `people: []`, o campo se chama "Digite o seu CPF (ou o da pessoa que vai ser atendida)" e tem a dica "11 números, está no documento";
    - com pessoas, continua "CPF de outra pessoa".
  - **WP-07, CPF:** "Continuar com este CPF" com CPF inválido marca o campo (`aria-invalid="true"`) e foca nele.
  - **WP-09:**
    - pessoa com perfil `birth_date: "1958-05-10"` e bairro "Batel": o botão tem o nome acessível "CPF ***.959.363-**, nascido(a) em 10/05/1958, Batel";
    - os links visíveis são "Trocar bairro" e "Ver triagens", com `aria-label` "Trocar bairro de ***.959.363-**" e "Ver triagens de ***.959.363-**".
  - **WP-11:** depois de "Continuar com este CPF" com um CPF válido, o formulário mostra "CPF ***.959.363-**", e o CPF inteiro não aparece (`queryByText(/673\.959\.363-96/)` é `null`).
  - **A-20:** CPF novo → perfil preenchido → bairro → "Voltar" leva de volta ao formulário com a data e o sexo já preenchidos.
  - **WP-12:**
    - na pergunta 1 (`can_undo: false`), há o secundário "Escolher outra triagem", que chama `onLeave`;
    - com `can_undo: true`, ele não aparece;
    - no Flow, `onLeave` volta ao catálogo da mesma pessoa.
  - **WP-02:** o `Screen` da pergunta recebe `focusKey={step.index}` (no Flow, duas perguntas seguidas com o mesmo `prompt` devolvem o foco ao `h1`).
  - **WP-13:** no catálogo, os botões têm os nomes "Começar: Dengue" e "Continuar: Saúde mental" (`aria-label`); o texto visível continua "Começar"/"Continuar".
  - **WP-06 (front):**
    - ao chegar numa pergunta, `sessionStorage[RESUME_KEY]` guarda o `citizenId`;
    - num novo boot com sessão válida, aceitar o termo leva direto ao catálogo dessa pessoa;
    - "Sair", `citizen:unauthenticated` e o resultado apagam a chave;
    - sem a chave, o termo leva a "Para quem é", como hoje;
    - `sessionStorage` que lança erro não quebra o fluxo.

  Run: `npx vitest run src/lib/masks.test.ts src/modules/citizen`
  Expected: FAIL nos novos.

- [ ] **Step 2: implementar.**
  - `masks.ts`:

```ts
// Mesmo formato de `cpf_masked` do api: só os 6 do meio à vista.
export function hideCpf(cpf: string): string {
  const d = onlyDigits(cpf);
  return d.length === 11 ? `***.${d.slice(3, 6)}.${d.slice(6, 9)}-**` : "***";
}
```

  - `PeopleStep`:
    - `Mode` `profile-for-new` ganha `profile?: ProfileInput`; o `ProfileForm` recebe `initial={mode.profile}` e `who={\`CPF ${hideCpf(newCpf)}\`}`;
    - o `onBack` do `pick-for-start` volta a `profile-for-new` com `who.cpf`/`who.profile` quando `"cpf" in who`, e à lista nos outros casos;
    - o botão da pessoa: o texto visível é "CPF …" e, numa segunda linha, `personLine(p)` (`"Nascido(a) em 10/05/1958 · Batel"`, com as partes ausentes omitidas). O `aria-label` junta as partes com vírgulas;
    - `cpfError` vai no `Field`, com `ref`, e o foco vai para ele no erro;
    - o rótulo e a dica dependem de `people?.length === 0`.
  - `QuestionStep`: `onLeave` → `{!step.can_undo && onLeave && <BigButton variant="secondary" onClick={onLeave} disabled={busy}>Escolher outra triagem</BigButton>}`; `<Screen focusKey={step.index}>`.
  - `CatalogStep`: `startButton(name, label, title)` passa `aria-label={\`${label}: ${title}\`}`.
  - `Flow`:
    - `onLeave={() => setState({ at: "catalog", consentVersion, citizenId, notice: null })}`;
    - `remember`/`forget` com `try/catch` em volta do `sessionStorage`;
    - `remember(citizenId)` ao entrar em `question`; `forget()` em `result`, `signOut` e `onUnauthenticated`;
    - o `onAccept` do termo vai para `catalog` com o `citizenId` guardado, se houver.

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit`
  Expected: verde.

- [ ] **Step 3: commit** (`feat: clearer person choice, masked CPF, exit on the first question and resume after reload`).

---

### Task 6: Bairro (WP-14)

**Files:** `src/modules/citizen/NeighborhoodPicker.tsx` e um teste novo `src/modules/citizen/NeighborhoodPicker.test.tsx`.

- [ ] **Step 1: testes que falham** (com 76 bairros falsos, `n1`…`n76`, e um caso com 5):
  - a ordem no DOM é: busca, "Prefiro não informar" e só depois a lista (`compareDocumentPosition`);
  - com 76 e a busca vazia: nenhum botão de bairro; aparecem a dica "Digite o nome do bairro" e o botão "Ver todos os bairros (76)", que mostra os 76;
  - com 76 e a busca "Bat": só os que casam;
  - com 5: os 5 aparecem logo, sem o "Ver todos";
  - sem resultado: "Nenhum bairro encontrado com esse nome. Se preferir, toque em "Prefiro não informar"." (`role="status"`).

  Run: `npx vitest run src/modules/citizen/NeighborhoodPicker.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - `const SHOW_ALL_UP_TO = 8`, mais o estado `showAll`;
  - `listed = query.trim() || showAll || neighborhoods.length <= SHOW_ALL_UP_TO ? shown : []`;
  - "Prefiro não informar" sai do `footer` e vem logo depois do campo, como `BigButton variant="secondary"`; o `footer` fica só com "Voltar".

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit`
  Expected: verde (os testes de `PeopleStep` que tocam num bairro sem buscar usam cidades com até 8 bairros ou buscam antes; se algum mudar, Ruling WP-14).

- [ ] **Step 3: commit** (`feat: neighborhood search first with opt-out on top and no full list until asked`).

---

### Task 7: Código de chegada e Minhas triagens (WP-15, WP-16)

**Files:** `src/modules/citizen/ui.tsx`, `src/modules/citizen/CounterCodeStep.tsx`, `src/modules/citizen/HistoryStep.tsx`, e testes em `src/modules/citizen/review-fixes.test.tsx` (ou um novo `codes-history.test.tsx`).

**Interfaces:**
- Produces: `ConfirmInline({ label, question, confirmLabel, busy?, onConfirm, variant? })` em `ui.tsx`. No primeiro toque, `label` vira a `question` com "Confirmar"/"Cancelar"; o foco vai para a pergunta (`tabIndex=-1`); Cancelar devolve o foco ao botão.

- [ ] **Step 1: testes que falham:**
  - **WP-15:**
    - com o código válido, não há botão principal; "Gerar outro código" é secundário e pede confirmação ("Gerar outro código? O código de agora deixa de valer."); `issue` só é chamado de novo no "Confirmar";
    - com o código vencido (avance os timers até `expires_at`), "Gerar outro código" é principal e chama `issue` no primeiro toque;
    - com a confirmação aberta e o código vencendo, a confirmação some (Review Focus 5).
  - **WP-16:**
    - enquanto `triages` não responde, há `role="status"` com "Carregando…";
    - não há "Revogar consentimento" nem `window.confirm` (`vi.spyOn(window, "confirm")` nunca chamado). Há "Apagar minhas respostas desta triagem", que pede confirmação na tela ("As respostas desta triagem serão apagadas. Isso não pode ser desfeito.") e chama `revokeConsent` no "Confirmar";
    - "Validar no posto" é secundário (`style.background === "transparent"`); "Cheguei na unidade" continua principal;
    - "Cadastro não verificado" vira "Seu cadastro ainda não foi conferido na unidade de saúde.".

  Run: `npx vitest run src/modules/citizen`
  Expected: FAIL nos novos.

- [ ] **Step 2: implementar.**
  - `ConfirmInline` (≈30 linhas em `ui.tsx`): estado `asking`; com `asking`, renderiza um `<div role="group" aria-labelledby>` com `<p ref tabIndex={-1}>{question}</p>`, `BigButton variant={variant ?? "primary"}` "Confirmar" e `BigButton variant="secondary"` "Cancelar"; foco na pergunta ao abrir.
  - `CounterCodeStep`: `expired = data !== null && left <= 0`. No `footer`: com `expired` (ou erro sem `data`), "Gerar outro código" é principal e direto; senão, `ConfirmInline` secundário (`key={data?.code}`, para fechar quando o código muda ou vence). "Voltar" continua secundário.
  - `HistoryStep`: `Carregando…` com `role="status"` enquanto `data === null && !error`; o `revoke` sem `window.confirm` e atrás de `ConfirmInline` (`variant="danger"` no Confirmar); os textos acima.
  - Os testes existentes que usam `window.confirm` ou "Revogar consentimento" mudam (Ruling WP-16).

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit`
  Expected: verde.

- [ ] **Step 3: commit** (`feat: show the arrival code first, confirm regenerating it and plain-language history actions`).

---

### Task 8: Acessibilidade e portão de celular (WP-18, A-13, A-14, A-16)

**Files:** `src/modules/citizen/A11yBar.tsx`, `src/modules/citizen/a11yStore.ts`, `src/modules/citizen/A11yPage.tsx`, `src/modules/citizen/chrome.test.tsx`, `src/lib/device.ts`, `src/lib/device.test.ts`, `src/App.gate.test.tsx`.

- [ ] **Step 1: testes que falham:**
  - **WP-18:** "Acessibilidade" e "Alto contraste" têm `minHeight: 48px` e `fontSize` ≥ 16 px.
  - **A-13:** com a página de acessibilidade aberta, "Ir para o conteúdo" foca o título "Acessibilidade".
  - **A-14:**
    - abrir a página faz `history.pushState` com `{ a11y: true }`;
    - um `popstate` com o estado anterior fecha a página e mantém o passo (o `h1` do passo volta a ser visível);
    - "Voltar" da página faz `history.back()` quando o estado é o dela e fecha;
    - abrir duas vezes não empilha duas entradas.
  - **A-16** (em `device.test.ts`):
    - `screen` 673x841, toque e `navigator.userAgentData = { mobile: true }` → `isPhone` verdadeiro;
    - mesmo tamanho com UA "Android 14; SM-F946B … Mobile" → verdadeiro;
    - iPad Pro 11 (834x1194, UA sem "Mobile" de Android, `userAgentData` ausente) → falso;
    - desktop sem toque com `userAgentData.mobile` verdadeiro → falso.

  Run: `npx vitest run src/lib/device.test.ts src/modules/citizen/chrome.test.tsx`
  Expected: FAIL nos novos.

- [ ] **Step 2: implementar.**
  - `A11yBar`: `minHeight: 48` e `fontSize: 16` nos dois alvos e na barra. O `focusMain` passa a focar `#a11y-title` quando a página está aberta.
  - `a11yStore`:

```ts
// "Voltar" do celular fecha a página (A-14): abrir empilha uma entrada própria.
export function openA11y(): void {
  if (open) return;
  open = true;
  try { window.history.pushState({ a11y: true }, ""); } catch { /* sem histórico */ }
  emit();
}
export function closeA11y(): void {
  if (!open) return;
  if (window.history.state?.a11y) { window.history.back(); return; } // o popstate fecha
  open = false; emit();
}
window.addEventListener("popstate", () => {
  if (open && !window.history.state?.a11y) { open = false; emit(); }
});
```

  - `device.ts`:

```ts
// Dobrável aberto passa de 600 px no lado menor: vale também quem se declara
// celular (A-16). Toque como entrada principal continua obrigatório.
function declaresPhone(win: Window): boolean {
  const nav = win.navigator as Navigator & { userAgentData?: { mobile?: boolean } };
  return nav.userAgentData?.mobile === true || /Android.*Mobile/.test(nav.userAgent);
}
export function isPhone(win: Window = window): boolean {
  // …coarse como hoje
  return coarse && (shortest < 600 || declaresPhone(win));
}
```

  - A frase da decisão 9 no comentário do `device.ts` ganha a exceção do dobrável.

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit`
  Expected: verde.

- [ ] **Step 3: commit** (`feat: larger accessibility bar, back button closes the accessibility page and foldables pass the phone gate`).

---

### Task 9: e2e, capturas, revisão, cards e publicação

- [ ] **e2e:** `npm run e2e` no worktree, nos 5 perfis. O `device-gate.spec.ts` continua passando; ipad e desktop seguem bloqueados.
- [ ] **Capturas "depois"** em 360x800 e 360x640, com o tema claro e o alto contraste, em `evidencias/2026-10-12-usabilidade/6e/`:
  - use `scratchpad/capture/login.mjs` e `wpda-flow.mjs`, com o Vite do worktree numa porta própria e o contorno de Origin (`ctx.route` + `route.fetch`, Origin `http://curitiba.localhost:5176`);
  - um pedido de OTP por passada; respeite o limite (`too_soon`);
  - telas:
    - celular com erro;
    - Para quem é (vazio e com pessoas);
    - perfil (gênero recolhido);
    - bairro;
    - pergunta 1 com "Escolher outra triagem";
    - espera do resultado;
    - resultado com "E agora?";
    - código de chegada;
    - Minhas triagens com a confirmação;
    - página de acessibilidade;
  - monte `6e/comparacao.html` (antes × depois), no padrão da 6a–6d.
- [ ] **Revisão final** pelo revisor mais capaz, com o Review Focus acima:
  - corrija os achados Critical e Important com TDD;
  - registre-os no ledger.
- [ ] **Aprovação do usuário** com a comparação.
- [ ] **Cards no board #2, com permissão:**
  - WP-06: o api diz se a sessão tem consentimento vigente, para o wpda não mostrar o termo a cada abertura;
  - WP-09: apelido da pessoa no par celular+CPF.
- [ ] **Merge e push:**
  - avise as sessões que mexem no wpda;
  - confira `origin/main..main`;
  - faça `merge --ff-only` e `push` do wpda e do docs;
  - remova o worktree;
  - confira o wpda-dev (5176) depois do merge.
- [ ] **Fechamento:** o subprojeto 6 e o ciclo de interface fecham. Atualize a memória `interface-ciclo-tokens.md` e o `MEMORY.md`.
