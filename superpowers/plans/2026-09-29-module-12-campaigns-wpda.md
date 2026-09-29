# Módulo 12 — Campanhas (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** No canal web do cidadão, a pessoa vê no topo de toda tela logada, ao lado de "Sair", um link "Avisos" com o número de avisos não lidos, lê os avisos da Secretaria em `/avisos` (tocar abre o texto e marca lido), escolhe por pessoa do telefone se quer SMS e se quer silenciar o selo em `/preferencias`, e o link do SMS (`/wpda/avisos`) sem sessão passa pelo login e volta para a caixa de avisos.

**Architecture:** Tipos e chamadas novas em `src/lib/citizenApi.ts`, normalizados na borda (ausente vira `null`, `0`, `false` ou `[]`), como já é feito para triagem e agendamento. Três componentes novos na pasta do cidadão: `NoticesLink` (link + selo, na barra do topo das telas logadas, ao lado de "Sair"), `NoticesStep` (lista e texto do aviso) e `PreferencesStep` (interruptores por pessoa), mais a peça `Toggle` em `ui.tsx`. O `Flow` continua sendo a máquina de telas sem biblioteca de rotas; ganha os estados `notices` e `preferences` e um mecanismo mínimo de endereço: `src/lib/route.ts` lê `<base>avisos`/`<base>preferencias` da URL no carregamento, o `Flow` guarda esse destino até o login terminar e mantém a URL em dia com `history.replaceState`. O nginx e o Vite já devolvem o `index.html` para qualquer caminho sob `/wpda/`; não há mudança de infraestrutura.

**Tech Stack:** React 18 + TypeScript (strict, `noUnusedLocals`, `noUnusedParameters`), Vite 5 (base `/wpda/`), Vitest 2 + Testing Library com jest-dom e user-event, ambiente jsdom com `TZ=America/Sao_Paulo`. Sem TanStack Query nas telas do cidadão (estado local, como hoje). Nenhuma dependência nova.

**Spec:** `docs/.claude/mod12/superpowers/specs/2026-09-29-module-12-campaigns-design.md` (§6.2 "Cidadão", §8 "wpda", §9.4). ADR: `docs/.claude/mod12/adr/0024.md`. Contratos HTTP fixados (api ↔ dashboard ↔ wpda): `mod12-contracts.md` (seção "Cidadão"). Depende do plano do api do módulo 12 (rotas `/citizen/notices` e `/citizen/contact_preferences`) estar mergeado **antes** do merge deste (ordem de rollout da spec §11: api → dashboard → wpda).

## Global Constraints

- Rotas consumidas (contrato fixado; não mude nomes nem formatos):
  - `GET /citizen/notices` → `{ notices: [{ id, title, body, dispatched_at, read, cpf_masked }], unread_count }`; `id` é o do `campaign_recipient`; `cpf_masked` só quando o telefone tem mais de um cidadão, senão `null`; mais novo primeiro; `unread_count` já desconta quem silenciou (0 se todos silenciaram).
  - `POST /citizen/notices/:id/read` → `{ ok: true }`; id de outro telefone → 404.
  - `GET /citizen/contact_preferences` → `{ sms_available, people: [{ citizen_id, cpf_masked, sms_opt_in, notices_muted }] }`.
  - `PUT /citizen/contact_preferences/:citizen_id` com `{ sms_opt_in?, notices_muted? }` → `{ citizen_id, cpf_masked, sms_opt_in, notices_muted }`; cidadão de outro telefone → 404.
- **Cidadão não tem nome.** Onde a spec diz "nome da pessoa", mostre o `cpf_masked` do contrato (como a escolha "Para quem é esta triagem?" já faz: `CPF ***.982.247-**`).
- Textos fixos (copiar exatamente):
  - vazio: "Nenhum aviso da Secretaria por enquanto."
  - interruptor de SMS: "Receber avisos por SMS", com a explicação "A Secretaria de Saúde pode enviar um SMS avisando que há um aviso novo aqui. Você pode desligar quando quiser."
  - interruptor de silêncio: "Silenciar avisos", com a explicação "Os avisos novos deixam de aparecer no número ao lado de Avisos. Eles continuam na lista de avisos."
  - data do aviso: `dd/mm/aaaa`; aviso de outra pessoa do telefone: "Para o CPF …"; depois de salvar: "Preferência salva." (aprovados pelo usuário em 2026-09-29).
- O link "Avisos" com selo fica no topo, ao lado de "Sair", em toda tela logada (inclusive a do termo); sem sessão não aparece (decisão do usuário, 2026-09-29).
- Entrar em `/avisos` ou `/preferencias` depois do login vai direto à tela, sem o termo; o termo continua obrigatório para começar uma triagem (decisão do usuário, 2026-09-29).
- As preferências abrem pela caixa ("Preferências de avisos") e por `/wpda/preferencias`; não há atalho no topo.
- O interruptor de SMS só aparece quando `sms_available` é `true`.
- Silenciar tira o selo (o api devolve `unread_count` sem os silenciados); os avisos continuam na lista, com o marcador "novo".
- O texto do aviso é texto simples: quebras de linha preservadas, nada interpretado como HTML.
- Link do SMS: `<base do wpda>/avisos` (rota `/avisos` sob a base `/wpda/`). Sem sessão, passa pelo login de sempre (celular + código por SMS; ver "Divergências") e volta a `/avisos`.
- Rotas do wpda: `/avisos`, `/preferencias`. Todo `href` é montado com `import.meta.env.BASE_URL`, lido na hora do render (os testes trocam com `vi.stubEnv`), nunca com `/` fixo.
- wpda: texto com pelo menos 18 px e alvos de toque com pelo menos 48 px (usar `BigButton`, `Screen`, `ErrorText` de `ui.tsx`; o `body` global é 13 px, então todo `button`/`a`/`label` novo declara `fontSize: 18`). Linguagem simples.
- Testes que mostram datas fixam o relógio em `beforeEach` com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(...)` e soltam com `vi.useRealTimers()` no `afterEach` (só `Date`: o `CodeStep` usa `setTimeout` real).
- Testes que mexem na URL (`window.history.replaceState`) ou em `BASE_URL` (`vi.stubEnv`) desfazem no `afterEach` (`replaceState(null, "", "/")`, `vi.unstubAllEnvs()`).
- O WhatsApp está descontinuado; o wpda web é o único canal do cidadão.
- Nunca `git add -A` nem `git add .` (o worktree tem symlink de `node_modules`); adicione arquivos pelo nome.
- Commits em inglês, Conventional Commits com tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

Todos os comandos abaixo rodam a partir do worktree `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda/.claude/mod12` (a raiz do monorepo não é git; o repositório é `apps/wpda`).

- Criar o worktree (uma vez, antes da Task 1), a partir de `origin/main` (hoje `d1b552f`):

  ```bash
  cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda \
    && /opt/homebrew/bin/git fetch origin \
    && /opt/homebrew/bin/git worktree add -b feat/mod-12-campaigns .claude/mod12 origin/main
  ```

- Excluir `.superpowers/` do git (no `info/exclude` comum, que vale para o worktree) e ligar o `node_modules` do app principal por symlink (o `.gitignore` já ignora `node_modules`; mesmo assim, nunca `git add -A`):

  ```bash
  cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda/.claude/mod12 \
    && EX="$(/opt/homebrew/bin/git rev-parse --git-common-dir)/info/exclude" \
    && (grep -qxF ".superpowers/" "$EX" || echo ".superpowers/" >> "$EX") \
    && ([ -e node_modules ] || ln -s ../../node_modules node_modules)
  ```

- Confira: `/opt/homebrew/bin/git status -sb && /opt/homebrew/bin/git log --oneline -1`. Esperado: `## feat/mod-12-campaigns...origin/main` sem mudanças, topo `d1b552f` (ou o `origin/main` do dia).
- Testes (scripts reais do `package.json`: `"test": "vitest run"`, `"typecheck": "tsc --noEmit"`):

  ```bash
  npx vitest run <arquivos>   # um ou mais arquivos
  npm test                    # suíte inteira
  npm run typecheck           # antes de cada commit
  ```

- Anote a contagem da suíte antes da Task 1 (`npm test`, linha `Tests  N passed`); nenhum teste que já existe pode cair.

## Divergências entre a spec/contrato e o código (decididas neste plano)

- **Login é por celular + código, não "CPF + código"** (spec §8). O código real: `src/modules/citizen/PhoneStep.tsx:26-31` ("Seu celular" → "Receber código") e `src/modules/citizen/CodeStep.tsx:43-53` ("Código" → "Confirmar"); a sessão é por telefone (`citizen_sessions.phone`). O plano usa o login que existe e não cria outro.
- **Não há "voltar para" nem rotas**: `src/modules/citizen/Flow.tsx:1-2` ("Sem biblioteca de rotas… um F5 volta ao começo"); `src/App.tsx:6-9` só lê `?token`. A Task 5 cria o mecanismo mínimo (`src/lib/route.ts` + destino guardado no `Flow`).
- **Não há "tela inicial" própria**: com sessão, o `Flow` sempre abre o termo (`Flow.tsx:34-38`, estado `consent`), e a primeira tela depois dele é "Para quem é esta triagem?" (`PeopleStep.tsx:139`). Decisão do usuário (2026-09-29): o link "Avisos" com selo fica no topo, ao lado de "Sair" (`Flow.tsx:98-103`, hoje um botão `position: fixed`), em toda tela logada, inclusive a do termo (Task 3). A `PeopleStep` não muda.
- **O termo aparece a cada visita** (`Flow.tsx:34-38`). A caixa de avisos não depende dele (ADR 0024: o aviso não exige opt-in e o consentimento é por conversa de triagem). Decisão do usuário (2026-09-29): o link do SMS, e qualquer entrada em `/avisos` ou `/preferencias`, vai direto à tela depois do login, sem passar pelo termo (Task 5). O termo continua obrigatório para começar uma triagem.

## Review Focus

1. **Link do SMS aberto sem sessão, ou sessão que cai no meio da leitura:** `/wpda/avisos` sem cookie, ou um 401 enquanto a pessoa está na caixa. Ela esperaria entrar com celular + código e cair de novo na caixa de avisos, não no termo nem em "Para quem é esta triagem?"; e "Sair" esquece o destino. Teste: Task 5 (`Flow.route.test.tsx`: "sem sessão", "401 na caixa", "Sair esquece o destino").
2. **Marcar lido falha ou é repetido:** rede fora, 404 (a revogação de consentimento apagou a linha do destinatário) ou tocar no mesmo aviso de novo antes da resposta. Ela esperaria ler o texto sem erro na tela, com um POST só por aviso; se falhou, o aviso continua "novo" e tenta de novo na próxima abertura. Teste: Task 2 ("falha ao marcar lido", "abrir de novo antes da resposta").
3. **Texto do aviso com `<`, `>`, quebras de linha ou linha longa:** a secretaria escreve "<b>Atenção</b>" ou cola três parágrafos. A pessoa esperaria ver exatamente o que foi escrito, com as quebras, sem negrito inventado. Teste: Task 2 ("texto com HTML aparece como texto", "quebras preservadas").
4. **api antiga ou fora do ar ao montar a barra do topo:** `GET /citizen/notices` responde 404 ou a rede falha (deploy fora de ordem). Ela esperaria ver o link "Avisos" sem selo e seguir para a triagem, sem mensagem de erro. Teste: Task 3 (`NoticesLink.test.tsx` "falha ao buscar" e `Flow.notices.test.tsx` "api sem a rota").
5. **Silenciar com avisos não lidos:** depois de "Silenciar avisos", o selo some da tela inicial, mas a lista continua mostrando "novo" nos não lidos; e um toque duplo no interruptor manda um PUT só. Teste: Task 4 (`Flow.notices.test.tsx` "silenciar nas preferências tira o selo do topo" e `PreferencesStep.test.tsx` "toque duplo").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `src/lib/citizenApi.ts` | tipos `Notice`, `NoticesResult`, `ContactPreference`, `ContactPreferences`, `ContactPreferenceChange`; `notices()`, `readNotice()`, `contactPreferences()`, `updateContactPreference()` | 1 |
| `src/lib/format.ts` | `fmtDate` (só a data, fuso da cidade) | 1 |
| `src/modules/citizen/NoticesStep.tsx` | caixa de avisos: lista, texto do aviso, marcar lido | 2 |
| `src/modules/citizen/NoticesLink.tsx` | link "Avisos" com selo de não lidos (na barra do topo) | 3 |
| `src/modules/citizen/Flow.tsx` | barra "Conta" do topo (Avisos + Sair) e estado `notices` (3); estado `preferences` (4); destino pela URL e URL em dia (5) | 3, 4, 5 |
| `src/modules/citizen/ui.tsx` | `Toggle` (interruptor com alvo de 48 px) | 4 |
| `src/modules/citizen/PreferencesStep.tsx` | preferências por pessoa do telefone | 4 |
| `src/lib/route.ts` | `routeFromPath`, `pathFor` | 5 |

---

### Task 1: Tipos e cliente da API dos avisos e das preferências

**Files:**
- Modify: `src/lib/citizenApi.ts`, `src/lib/format.ts`
- Test: `src/lib/citizenApi.test.ts`, `src/lib/format.test.ts`

**Interfaces:**
- Produces (em `src/lib/citizenApi.ts`):
  - `interface Notice { id: string; title: string; body: string; dispatched_at: string; read: boolean; cpf_masked: string | null }`
  - `interface NoticesResult { notices: Notice[]; unread_count: number }`
  - `interface ContactPreference { citizen_id: string; cpf_masked: string; sms_opt_in: boolean; notices_muted: boolean }`
  - `interface ContactPreferences { sms_available: boolean; people: ContactPreference[] }`
  - `type ContactPreferenceChange = { sms_opt_in?: boolean; notices_muted?: boolean }`
  - `citizenApi.notices(): Promise<NoticesResult>` — `cpf_masked` ausente vira `null`, `read` vira booleano, `unread_count` ausente vira `0`, lista ausente vira `[]`.
  - `citizenApi.readNotice(id: string): Promise<{ ok: boolean }>`
  - `citizenApi.contactPreferences(): Promise<ContactPreferences>` — `sms_available` só é `true` quando a API manda `true`; lista ausente vira `[]`.
  - `citizenApi.updateContactPreference(citizenId: string, change: ContactPreferenceChange): Promise<ContactPreference>` — manda só as chaves de `change`.
- Produces (em `src/lib/format.ts`): `fmtDate(iso: string | null | undefined): string` — `dd/mm/aaaa` em `America/Sao_Paulo`; vazio ou inválido vira `—`.

- [ ] **Step 1: Escreva os testes do cliente**

Acrescente ao fim do `describe("citizenApi", ...)` em `src/lib/citizenApi.test.ts`:

```ts
  it("notices faz GET /citizen/notices e normaliza cpf_masked ausente para null", async () => {
    const fn = mockFetch(200, { notices: [
      { id: "r1", title: "Vacinação", body: "Texto", dispatched_at: "2026-09-28T13:00:00-03:00", read: false },
      { id: "r2", title: "Mutirão", body: "Texto", dispatched_at: "2026-09-27T13:00:00-03:00", read: true,
        cpf_masked: "***.982.247-**" }
    ], unread_count: 1 });
    const r = await citizenApi.notices();
    const [url, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(url).toBe("/citizen/notices");
    expect(init.method).toBe("GET");
    expect(r.unread_count).toBe(1);
    expect(r.notices.map(n => n.id)).toEqual([ "r1", "r2" ]);
    expect(r.notices[0].cpf_masked).toBeNull();
    expect(r.notices[1].cpf_masked).toBe("***.982.247-**");
    expect(r.notices[1].read).toBe(true);
  });

  it("notices sem lista nem unread_count vira lista vazia e zero", async () => {
    mockFetch(200, {});
    expect(await citizenApi.notices()).toEqual({ notices: [], unread_count: 0 });
  });

  it("readNotice faz POST no id do aviso, codificado", async () => {
    const fn = mockFetch(200, { ok: true });
    expect(await citizenApi.readNotice("r/1")).toEqual({ ok: true });
    const [url, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(url).toBe("/citizen/notices/r%2F1/read");
    expect(init.method).toBe("POST");
  });

  it("readNotice de outro telefone rejeita com 404", async () => {
    mockFetch(404, { error: "not_found" });
    await expect(citizenApi.readNotice("r9")).rejects.toEqual(new ApiError(404, "not_found"));
  });

  it("contactPreferences faz GET e só liga sms_available com true explícito", async () => {
    const people = [ { citizen_id: "p1", cpf_masked: "***.982.247-**", sms_opt_in: false, notices_muted: true } ];
    const fn = mockFetch(200, { sms_available: true, people });
    expect(await citizenApi.contactPreferences()).toEqual({ sms_available: true, people });
    const [url, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(url).toBe("/citizen/contact_preferences");
    expect(init.method).toBe("GET");

    mockFetch(200, { people });
    expect((await citizenApi.contactPreferences()).sms_available).toBe(false);
    mockFetch(200, {});
    expect(await citizenApi.contactPreferences()).toEqual({ sms_available: false, people: [] });
  });

  it("updateContactPreference manda PUT só com o campo mudado", async () => {
    const entry = { citizen_id: "p1", cpf_masked: "***.982.247-**", sms_opt_in: true, notices_muted: false };
    const fn = mockFetch(200, entry);
    expect(await citizenApi.updateContactPreference("p1", { sms_opt_in: true })).toEqual(entry);
    const [url, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(url).toBe("/citizen/contact_preferences/p1");
    expect(init.method).toBe("PUT");
    expect(JSON.parse(init.body as string)).toEqual({ sms_opt_in: true });
  });

  it("updateContactPreference de outro telefone rejeita com 404", async () => {
    mockFetch(404, { error: "not_found" });
    await expect(citizenApi.updateContactPreference("p9", { notices_muted: true }))
      .rejects.toEqual(new ApiError(404, "not_found"));
  });
```

- [ ] **Step 2: Escreva os testes de `fmtDate`**

Troque o conteúdo de `src/lib/format.test.ts` por:

```ts
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";
import { fmtDate, fmtDateTime } from "./format";

describe("fmtDateTime", () => {
  it("formata ISO em pt-BR (data)", () => {
    expect(fmtDateTime("2026-06-26T15:00:00Z")).toMatch(/26\/06\/2026/);
  });
  it("— para null", () => { expect(fmtDateTime(null)).toBe("—"); });
  it("— para data inválida", () => { expect(fmtDateTime("xxx")).toBe("—"); });
});

describe("fmtDate", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date("2026-09-29T10:00:00-03:00"));
  });
  afterEach(() => vi.useRealTimers());

  it("só a data, dd/mm/aaaa", () => {
    expect(fmtDate("2026-09-28T13:00:00-03:00")).toBe("28/09/2026");
  });
  it("perto da meia-noite usa o dia de São Paulo, não o de UTC", () => {
    expect(fmtDate("2026-09-29T02:30:00Z")).toBe("28/09/2026");
  });
  it.each([ null, undefined, "", "xxx" ])("%s vira —", (iso) => {
    expect(fmtDate(iso)).toBe("—");
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/lib/citizenApi.test.ts src/lib/format.test.ts`
Expected: FAIL (`citizenApi.notices is not a function`, `fmtDate` não exportado).

- [ ] **Step 4: Implemente o cliente**

Em `src/lib/citizenApi.ts`, depois da interface `ReferenceUnit`:

```ts
// Campanhas (spec 2026-09-29 §6.2; ADR 0024). Cidadão não tem nome: com mais de
// uma pessoa no telefone, o aviso diz de quem é pelo CPF mascarado.
export interface Notice {
  id: string; // id do campaign_recipient
  title: string;
  body: string; // texto simples, com quebras de linha
  dispatched_at: string;
  read: boolean;
  cpf_masked: string | null; // null quando o telefone tem uma pessoa só
}

export interface NoticesResult {
  notices: Notice[];
  // Já desconta as pessoas que silenciaram os avisos (0 se todas silenciaram).
  unread_count: number;
}

export interface ContactPreference {
  citizen_id: string;
  cpf_masked: string;
  sms_opt_in: boolean;
  notices_muted: boolean;
}

export interface ContactPreferences {
  sms_available: boolean; // chave de SMS da cidade
  people: ContactPreference[];
}

export type ContactPreferenceChange = { sms_opt_in?: boolean; notices_muted?: boolean };
```

No objeto `citizenApi`, depois de `issueAppointmentCheckInCode` (troque o `)` final da linha de `issueAppointmentCheckInCode` por `),` e acrescente):

```ts
  // Caixa de avisos (spec §6.2): avisos de todas as pessoas do telefone da
  // sessão, mais novo primeiro. Campos ausentes são normalizados aqui.
  notices: async (): Promise<NoticesResult> => {
    const data = await call<{ notices?: Notice[]; unread_count?: number }>("GET", "/notices");
    return {
      notices: (data.notices ?? []).map(n => ({ ...n, read: n.read === true, cpf_masked: n.cpf_masked ?? null })),
      unread_count: data.unread_count ?? 0
    };
  },
  readNotice: (id: string) => call<{ ok: boolean }>("POST", `/notices/${encodeURIComponent(id)}/read`),
  contactPreferences: async (): Promise<ContactPreferences> => {
    const data = await call<{ sms_available?: boolean; people?: ContactPreference[] }>("GET", "/contact_preferences");
    return { sms_available: data.sms_available === true, people: data.people ?? [] };
  },
  // Manda só o que mudou; a resposta é a entrada inteira da pessoa.
  updateContactPreference: (citizenId: string, change: ContactPreferenceChange) =>
    call<ContactPreference>("PUT", `/contact_preferences/${encodeURIComponent(citizenId)}`, change)
```

- [ ] **Step 5: Implemente `fmtDate`**

Acrescente ao fim de `src/lib/format.ts`:

```ts
const dateFmt = new Intl.DateTimeFormat("pt-BR", {
  timeZone: "America/Sao_Paulo",
  day: "2-digit", month: "2-digit", year: "numeric"
});

// Só a data (caixa de avisos), no fuso da cidade.
export function fmtDate(iso: string | null | undefined): string {
  if (!iso) return "—";
  const d = new Date(iso);
  if (Number.isNaN(d.getTime())) return "—";
  return dateFmt.format(d);
}
```

- [ ] **Step 6: Rode os testes e os tipos**

Run: `npx vitest run src/lib/citizenApi.test.ts src/lib/format.test.ts && npm run typecheck`
Expected: PASS, sem erro de tipo.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/lib/citizenApi.ts src/lib/citizenApi.test.ts src/lib/format.ts src/lib/format.test.ts
/opt/homebrew/bin/git commit -m "feat: add notice and contact preference calls to the citizen api client" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Caixa de avisos

**Files:**
- Create: `src/modules/citizen/NoticesStep.tsx`
- Test: `src/modules/citizen/NoticesStep.test.tsx` (novo)

**Interfaces:**
- Consumes (Task 1): `Notice`, `citizenApi.notices()`, `citizenApi.readNotice(id)`, `fmtDate`.
- Produces:
  - `NoticesStep({ onBack, onPreferences?, onRead? }: { onBack: () => void; onPreferences?: () => void; onRead?: () => void })` — sem `onPreferences`, o botão "Preferências de avisos" não aparece (a Task 4 passa a função); `onRead` é chamado depois de cada POST de leitura bem-sucedido (a Task 3 o usa para reler o selo do topo).
  - `export const EMPTY_NOTICES = "Nenhum aviso da Secretaria por enquanto."`

**Comportamento (spec §8):**
- Título da tela "Avisos". Lista na ordem da API (mais novo primeiro). Cada aviso é um botão com o título, a data (`fmtDate(dispatched_at)`), o marcador "novo" quando `read` é `false` e, quando `cpf_masked` vem, "Para o CPF ***.982.247-**".
- Vazio: "Nenhum aviso da Secretaria por enquanto.". Erro ao carregar: a mensagem de `messageFor` e "Tentar de novo".
- Tocar num aviso abre o texto completo (título como título da tela, data, CPF quando houver, corpo com `white-space: pre-wrap`) e, se não lido, chama `readNotice(id)` **uma vez**. Sucesso: o aviso fica lido na lista e `onRead` é chamado. Falha: nada na tela (o texto já está aberto), o aviso continua "novo" e a próxima abertura tenta de novo.
- "Voltar aos avisos" volta à lista, sem recarregar. Rodapé da lista: "Preferências de avisos" (se houver `onPreferences`) e "Voltar ao início".

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/citizen/NoticesStep.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, waitFor, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { NoticesStep, EMPTY_NOTICES } from "./NoticesStep";
import { citizenApi, ApiError, type Notice } from "../../lib/citizenApi";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-09-29T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const unread: Notice = {
  id: "r1", title: "Vacinação contra a gripe", body: "Leve a carteirinha.\nAté sexta.",
  dispatched_at: "2026-09-28T13:00:00-03:00", read: false, cpf_masked: null
};
const read: Notice = {
  id: "r2", title: "Mutirão de exames", body: "Sábado, das 8h às 12h.",
  dispatched_at: "2026-09-20T09:00:00-03:00", read: true, cpf_masked: null
};

function setup(notices: Notice[] = [ unread, read ]) {
  const list = vi.spyOn(citizenApi, "notices")
    .mockResolvedValue({ notices, unread_count: notices.filter(n => !n.read).length });
  const markRead = vi.spyOn(citizenApi, "readNotice").mockResolvedValue({ ok: true });
  const onBack = vi.fn();
  const onPreferences = vi.fn();
  render(<NoticesStep onBack={onBack} onPreferences={onPreferences} />);
  return { list, markRead, onBack, onPreferences };
}

const item = (name: RegExp) => screen.findByRole("button", { name });

describe("NoticesStep — lista", () => {
  it("mostra título, data e 'novo' só no não lido; com uma pessoa no telefone, sem CPF", async () => {
    setup();
    const first = await item(/Vacinação contra a gripe/);
    expect(within(first).getByText("novo")).toBeInTheDocument();
    expect(within(first).getByText("28/09/2026")).toBeInTheDocument();
    const second = screen.getByRole("button", { name: /Mutirão de exames/ });
    expect(within(second).queryByText("novo")).not.toBeInTheDocument();
    expect(within(second).getByText("20/09/2026")).toBeInTheDocument();
    expect(screen.queryByText(/CPF/)).not.toBeInTheDocument();
  });

  it("mantém a ordem da API (mais novo primeiro)", async () => {
    setup();
    await item(/Vacinação/);
    const items = screen.getAllByRole("listitem").map(li => li.textContent);
    expect(items[0]).toMatch(/Vacinação/);
    expect(items[1]).toMatch(/Mutirão/);
  });

  it("vazio: mensagem própria, sem lista", async () => {
    setup([]);
    expect(await screen.findByText(EMPTY_NOTICES)).toBeInTheDocument();
    expect(EMPTY_NOTICES).toBe("Nenhum aviso da Secretaria por enquanto.");
    expect(screen.queryByRole("list")).not.toBeInTheDocument();
  });

  it("várias pessoas no telefone: cada aviso diz o CPF mascarado de quem é", async () => {
    setup([
      { ...unread, cpf_masked: "***.982.247-**" },
      { ...read, cpf_masked: "***.111.222-**" }
    ]);
    expect(within(await item(/Vacinação/)).getByText("Para o CPF ***.982.247-**")).toBeInTheDocument();
    expect(within(screen.getByRole("button", { name: /Mutirão/ })).getByText("Para o CPF ***.111.222-**"))
      .toBeInTheDocument();
  });

  it("alvo de toque com pelo menos 48 px e texto com pelo menos 18 px", async () => {
    setup();
    const b = await item(/Vacinação/);
    expect(parseInt(b.style.minHeight, 10)).toBeGreaterThanOrEqual(48);
    expect(parseInt(b.style.fontSize, 10)).toBeGreaterThanOrEqual(18);
  });

  it("erro ao carregar: mensagem e 'Tentar de novo' recarrega", async () => {
    const list = vi.spyOn(citizenApi, "notices")
      .mockRejectedValueOnce(new TypeError("offline"))
      .mockResolvedValue({ notices: [ unread ], unread_count: 1 });
    render(<NoticesStep onBack={vi.fn()} />);
    expect(await screen.findByText("Sem conexão. Verifique a internet e tente de novo.")).toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Tentar de novo" }));
    expect(await item(/Vacinação/)).toBeInTheDocument();
    expect(list).toHaveBeenCalledTimes(2);
  });

  it("rodapé: 'Preferências de avisos' e 'Voltar ao início'", async () => {
    const { onBack, onPreferences } = setup();
    await item(/Vacinação/);
    await userEvent.click(screen.getByRole("button", { name: "Preferências de avisos" }));
    expect(onPreferences).toHaveBeenCalledTimes(1);
    await userEvent.click(screen.getByRole("button", { name: "Voltar ao início" }));
    expect(onBack).toHaveBeenCalledTimes(1);
  });

  it("sem onPreferences, não mostra o botão de preferências", async () => {
    vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [ unread ], unread_count: 1 });
    render(<NoticesStep onBack={vi.fn()} />);
    await item(/Vacinação/);
    expect(screen.queryByRole("button", { name: "Preferências de avisos" })).not.toBeInTheDocument();
  });
});

describe("NoticesStep — abrir um aviso", () => {
  it("tocar abre o texto completo, marca lido uma vez e, ao voltar, o aviso não é mais 'novo'", async () => {
    const { markRead, list } = setup();
    await userEvent.click(await item(/Vacinação/));

    expect(screen.getByRole("heading", { name: "Vacinação contra a gripe" })).toBeInTheDocument();
    expect(screen.getByText("28/09/2026")).toBeInTheDocument();
    expect(markRead).toHaveBeenCalledWith("r1");

    await userEvent.click(screen.getByRole("button", { name: "Voltar aos avisos" }));
    const first = await item(/Vacinação/);
    await waitFor(() => expect(within(first).queryByText("novo")).not.toBeInTheDocument());
    expect(markRead).toHaveBeenCalledTimes(1);
    expect(list).toHaveBeenCalledTimes(1); // voltar não recarrega
  });

  it("leitura bem-sucedida avisa o Flow (onRead) uma vez; aviso já lido não avisa", async () => {
    vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [ unread, read ], unread_count: 1 });
    vi.spyOn(citizenApi, "readNotice").mockResolvedValue({ ok: true });
    const onRead = vi.fn();
    render(<NoticesStep onBack={vi.fn()} onRead={onRead} />);
    await userEvent.click(await item(/Vacinação/));
    await waitFor(() => expect(onRead).toHaveBeenCalledTimes(1));
    await userEvent.click(screen.getByRole("button", { name: "Voltar aos avisos" }));
    await userEvent.click(await item(/Mutirão/));
    expect(onRead).toHaveBeenCalledTimes(1);
  });

  it("falha ao marcar lido não avisa o Flow", async () => {
    vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [ unread ], unread_count: 1 });
    const markRead = vi.spyOn(citizenApi, "readNotice").mockRejectedValue(new TypeError("offline"));
    const onRead = vi.fn();
    render(<NoticesStep onBack={vi.fn()} onRead={onRead} />);
    await userEvent.click(await item(/Vacinação/));
    await waitFor(() => expect(markRead).toHaveBeenCalled());
    expect(onRead).not.toHaveBeenCalled();
  });

  it("quebras de linha preservadas", async () => {
    setup();
    await userEvent.click(await item(/Vacinação/));
    const body = screen.getByText(/Leve a carteirinha/);
    expect(body.textContent).toBe("Leve a carteirinha.\nAté sexta.");
    expect(body).toHaveStyle({ whiteSpace: "pre-wrap" });
  });

  it("texto com HTML aparece como texto", async () => {
    setup([ { ...unread, body: "<b>Atenção</b>\n<script>alert(1)</script>" } ]);
    await userEvent.click(await item(/Vacinação/));
    expect(screen.getByText(/<b>Atenção<\/b>/)).toBeInTheDocument();
    expect(document.querySelector("main b")).toBeNull();
    expect(document.querySelector("main script")).toBeNull();
  });

  it("aviso já lido abre sem chamar o POST", async () => {
    const { markRead } = setup();
    await userEvent.click(await item(/Mutirão/));
    expect(screen.getByText("Sábado, das 8h às 12h.")).toBeInTheDocument();
    expect(markRead).not.toHaveBeenCalled();
  });

  it("abrir de novo antes da resposta não repete o POST", async () => {
    const { markRead } = setup();
    markRead.mockReturnValue(new Promise<{ ok: boolean }>(() => {}));
    await userEvent.click(await item(/Vacinação/));
    await userEvent.click(screen.getByRole("button", { name: "Voltar aos avisos" }));
    await userEvent.click(await item(/Vacinação/));
    expect(markRead).toHaveBeenCalledTimes(1);
  });

  it("falha ao marcar lido: o texto aparece sem erro, o aviso segue 'novo' e a próxima abertura tenta de novo", async () => {
    const { markRead } = setup();
    markRead.mockRejectedValue(new ApiError(404, "not_found"));
    await userEvent.click(await item(/Vacinação/));

    expect(screen.getByText(/Leve a carteirinha/)).toBeInTheDocument();
    await waitFor(() => expect(markRead).toHaveBeenCalledTimes(1));
    expect(screen.queryByRole("alert")).not.toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Voltar aos avisos" }));
    expect(within(await item(/Vacinação/)).getByText("novo")).toBeInTheDocument();

    await userEvent.click(await item(/Vacinação/));
    await waitFor(() => expect(markRead).toHaveBeenCalledTimes(2));
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/NoticesStep.test.tsx`
Expected: FAIL (`./NoticesStep` não existe).

- [ ] **Step 3: Implemente a caixa**

```tsx
// src/modules/citizen/NoticesStep.tsx
// Caixa de avisos da Secretaria (spec 2026-09-29 §8; ADR 0024). Avisos de todas
// as pessoas do telefone da sessão; com mais de uma, o CPF mascarado diz de quem
// é (cidadão não tem nome). Tocar abre o texto e marca lido. O texto é simples:
// o React escapa tudo e o pre-wrap preserva as quebras de linha.
import { useEffect, useRef, useState } from "react";
import { citizenApi, type Notice } from "../../lib/citizenApi";
import { fmtDate } from "../../lib/format";
import { BigButton, ErrorText, Screen, messageFor } from "./ui";

export const EMPTY_NOTICES = "Nenhum aviso da Secretaria por enquanto.";

const itemStyle = {
  display: "grid", gap: 4, width: "100%", minHeight: 56, padding: 12, textAlign: "left",
  borderRadius: 12, border: "1px solid var(--rule2, #ccc)", fontSize: 18, cursor: "pointer",
  color: "var(--ink, #222)"
} as const;

const newStyle = {
  alignSelf: "start", padding: "0 8px", borderRadius: 999, fontSize: 18, fontWeight: 600,
  background: "var(--accent, #2b4bd8)", color: "#fff"
} as const;

export function NoticesStep({ onBack, onPreferences, onRead }:
  { onBack: () => void; onPreferences?: () => void; onRead?: () => void }) {
  const [notices, setNotices] = useState<Notice[] | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [openId, setOpenId] = useState<string | null>(null);
  // Um POST por aviso: a abertura repetida antes da resposta não manda outro.
  const asked = useRef(new Set<string>());

  function load() {
    setError(null);
    citizenApi.notices().then(r => setNotices(r.notices)).catch(e => setError(messageFor(e)));
  }

  useEffect(() => { load(); }, []);

  function open(n: Notice) {
    setOpenId(n.id);
    if (n.read || asked.current.has(n.id)) return;
    asked.current.add(n.id);
    citizenApi.readNotice(n.id)
      .then(() => {
        setNotices(list => list && list.map(x => (x.id === n.id ? { ...x, read: true } : x)));
        onRead?.();
      })
      // Falhou (rede, ou 404 porque a linha foi apagada): o texto já está na
      // tela; o aviso segue "novo" e a próxima abertura tenta de novo.
      .catch(() => { asked.current.delete(n.id); });
  }

  const current = notices?.find(n => n.id === openId) ?? null;
  if (current) {
    return (
      <Screen title={current.title}
        footer={<BigButton variant="secondary" onClick={() => setOpenId(null)}>Voltar aos avisos</BigButton>}>
        <p style={{ margin: "0 0 4px", color: "var(--ink2, #555)" }}>{fmtDate(current.dispatched_at)}</p>
        {current.cpf_masked && <p style={{ margin: "0 0 4px" }}>Para o CPF {current.cpf_masked}</p>}
        <div style={{ whiteSpace: "pre-wrap", lineHeight: 1.5, overflowWrap: "anywhere", marginTop: 16 }}>
          {current.body}
        </div>
      </Screen>
    );
  }

  return (
    <Screen title="Avisos" footer={<>
      {onPreferences && <BigButton variant="secondary" onClick={onPreferences}>Preferências de avisos</BigButton>}
      <BigButton variant="secondary" onClick={onBack}>Voltar ao início</BigButton>
    </>}>
      {error && <>
        <ErrorText>{error}</ErrorText>
        <BigButton style={{ marginTop: 12 }} onClick={load}>Tentar de novo</BigButton>
      </>}
      {notices === null && !error && <p>Carregando…</p>}
      {notices !== null && notices.length === 0 && <p>{EMPTY_NOTICES}</p>}
      {notices !== null && notices.length > 0 && (
        <ul style={{ listStyle: "none", margin: 0, padding: 0, display: "grid", gap: 12 }}>
          {notices.map(n => (
            <li key={n.id}>
              <button type="button" onClick={() => open(n)}
                style={{ ...itemStyle, background: n.read ? "transparent" : "var(--accent-bg, #eef1ff)" }}>
                <span style={{ display: "flex", justifyContent: "space-between", gap: 8 }}>
                  <strong>{n.title}</strong>
                  {!n.read && <span style={newStyle}>novo</span>}
                </span>
                <span style={{ color: "var(--ink2, #555)" }}>{fmtDate(n.dispatched_at)}</span>
                {n.cpf_masked && <span>Para o CPF {n.cpf_masked}</span>}
              </button>
            </li>
          ))}
        </ul>
      )}
    </Screen>
  );
}
```

- [ ] **Step 4: Rode os testes e os tipos**

Run: `npx vitest run src/modules/citizen/NoticesStep.test.tsx && npm run typecheck`
Expected: PASS, sem erro de tipo.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/NoticesStep.tsx src/modules/citizen/NoticesStep.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the citizen notices inbox" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Link "Avisos" com selo no topo de toda tela logada

**Files:**
- Create: `src/modules/citizen/NoticesLink.tsx`
- Modify: `src/modules/citizen/Flow.tsx`
- Test: `src/modules/citizen/NoticesLink.test.tsx` (novo), `src/modules/citizen/Flow.notices.test.tsx` (novo)

**Interfaces:**
- Consumes (Task 1): `citizenApi.notices()`. (Task 2): `NoticesStep({ onBack, onPreferences?, onRead? })`.
- Produces:
  - `NoticesLink({ onOpen }: { onOpen: () => void })`: `<a href={`${BASE_URL}avisos`}>`; o toque chama `onOpen` e impede a navegação da página. Busca o selo ao montar.
  - `badgeText(count: number): string | null`: `null` para 0, negativo ou não número; `"99+"` acima de 99.
  - `Flow`:
    - estado `{ at: "notices"; consentVersion: string | null }`;
    - barra `<nav aria-label="Conta">` com o `NoticesLink` e o "Sair", mostrada em toda tela logada (todo estado menos `boot`, `phone` e `code`);
    - `badgeTick` (contador em `useState`) e `refreshBadge()`: o `NoticesLink` usa `key={`${state.at}-${badgeTick}`}`, então busca o selo de novo a cada troca de tela e a cada `refreshBadge()`.
  - A Task 4 acrescenta `preferences` e liga `onSaved` a `refreshBadge`. A Task 5 reescreve o começo do `Flow` e mantém `badgeTick`/`refreshBadge` e a barra.

**Comportamento (spec §8 "Início", decisão do usuário de 2026-09-29):**
- **Onde fica o link:** "Avisos" fica no topo, ao lado de "Sair", em **toda** tela logada. Isso inclui a do termo, "Para quem é esta triagem?", a triagem, o histórico, a caixa e as preferências. Sem sessão (carregando, celular, código), não há barra nem link.
- **Selo:**
  - alvo de 48 px ou mais, texto de 18 px;
  - com `unread_count > 0`, um selo com o número (acima de 99, "99+"); nome acessível "Avisos, 1 novo" / "Avisos, N novos";
  - com 0 (inclusive quando todas as pessoas silenciaram), sem selo, e o nome é só "Avisos".
- **Falha na busca:** se `GET /citizen/notices` falha (api antiga, rede), o link aparece sem selo e sem mensagem de erro. Um 401 continua levando ao login (o `call` já dispara `citizen:unauthenticated`).
- **Tocar no link:** abre a caixa (estado `notices`), guardando a versão do termo quando a tela atual tem uma. "Voltar ao início" leva a "Para quem é esta triagem?" com essa versão, ou ao termo quando não havia versão (link tocado na tela do termo). O termo continua obrigatório para começar uma triagem.
- **Selo em dia:** marcar um aviso como lido na caixa (`onRead`) chama `refreshBadge()`, e o selo do topo se atualiza sem trocar de tela.
- **Barra:** fica no fluxo da página, acima da tela, alinhada à direita na mesma largura de 520 px. O "Sair" deixa de ser `position: fixed`.

- [ ] **Step 1: Escreva os testes do link**

```tsx
// src/modules/citizen/NoticesLink.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { NoticesLink, badgeText } from "./NoticesLink";
import { citizenApi, ApiError } from "../../lib/citizenApi";

beforeEach(() => vi.stubEnv("BASE_URL", "/wpda/"));
afterEach(() => { vi.unstubAllEnvs(); vi.restoreAllMocks(); });

describe("badgeText", () => {
  it.each([ [ 0, null ], [ -1, null ], [ Number.NaN, null ], [ 1, "1" ], [ 99, "99" ], [ 100, "99+" ] ])(
    "%s → %s", (count, text) => expect(badgeText(count)).toBe(text));
});

describe("NoticesLink", () => {
  it("mostra o selo com os não lidos e aponta para <base>avisos", async () => {
    vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [], unread_count: 3 });
    render(<NoticesLink onOpen={vi.fn()} />);
    const link = await screen.findByRole("link", { name: "Avisos, 3 novos" });
    expect(link).toHaveAttribute("href", "/wpda/avisos");
    expect(within(link).getByText("3")).toBeInTheDocument();
    expect(parseInt(link.style.minHeight, 10)).toBeGreaterThanOrEqual(48);
    expect(parseInt(link.style.fontSize, 10)).toBeGreaterThanOrEqual(18);
  });

  it("um não lido: singular", async () => {
    vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [], unread_count: 1 });
    render(<NoticesLink onOpen={vi.fn()} />);
    expect(await screen.findByRole("link", { name: "Avisos, 1 novo" })).toBeInTheDocument();
  });

  it("mais de 99: selo '99+', nome com o número real", async () => {
    vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [], unread_count: 150 });
    render(<NoticesLink onOpen={vi.fn()} />);
    const link = await screen.findByRole("link", { name: "Avisos, 150 novos" });
    expect(within(link).getByText("99+")).toBeInTheDocument();
  });

  it("zero (inclusive quando todos silenciaram): sem selo", async () => {
    const notices = vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [], unread_count: 0 });
    render(<NoticesLink onOpen={vi.fn()} />);
    await waitFor(() => expect(notices).toHaveBeenCalled());
    expect(screen.getByRole("link", { name: "Avisos" }).textContent).toBe("Avisos");
  });

  it("falha ao buscar (api antiga ou rede): link sem selo e sem erro", async () => {
    const notices = vi.spyOn(citizenApi, "notices").mockRejectedValue(new ApiError(404, "not_found"));
    render(<NoticesLink onOpen={vi.fn()} />);
    await waitFor(() => expect(notices).toHaveBeenCalled());
    expect(screen.getByRole("link", { name: "Avisos" })).toBeInTheDocument();
    expect(screen.queryByRole("alert")).not.toBeInTheDocument();
  });

  it("tocar abre a caixa sem recarregar a página", async () => {
    vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [], unread_count: 0 });
    const onOpen = vi.fn();
    render(<NoticesLink onOpen={onOpen} />);
    const link = screen.getByRole("link", { name: "Avisos" });
    expect(fireEvent.click(link)).toBe(false); // preventDefault
    expect(onOpen).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 2: Escreva os testes do `Flow`**

Os mocks de `notices`/`readNotice` leem uma variável `unread`, em vez de uma fila de `mockResolvedValueOnce`. O link é remontado a cada troca de tela, então o número de chamadas depende da ordem dos efeitos, e o teste não deve depender dela.

```tsx
// src/modules/citizen/Flow.notices.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, waitFor, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Flow } from "./Flow";
import { citizenApi, ApiError, type Notice } from "../../lib/citizenApi";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-09-29T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const notice: Notice = {
  id: "r1", title: "Vacinação contra a gripe", body: "Leve a carteirinha.",
  dispatched_at: "2026-09-28T13:00:00-03:00", read: false, cpf_masked: null
};

function signedIn() {
  vi.spyOn(citizenApi, "currentSession").mockResolvedValue({ phone_masked: "(**) *****-5432" });
  vi.spyOn(citizenApi, "consentTerm").mockResolvedValue({ version: "1", body: "Termo" });
  vi.spyOn(citizenApi, "people").mockResolvedValue({
    people: [{ id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared" }]
  });
  vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue([]);
}

// Estado do api simulado: um aviso; ler (ou silenciar, na Task 4) zera o selo.
function inbox() {
  const box = { unread: 1 };
  const notices = vi.spyOn(citizenApi, "notices").mockImplementation(async () => ({
    notices: [ { ...notice, read: box.unread === 0 } ], unread_count: box.unread
  }));
  const readNotice = vi.spyOn(citizenApi, "readNotice").mockImplementation(async () => {
    box.unread = 0;
    return { ok: true };
  });
  return { box, notices, readNotice };
}

const bar = () => screen.getByRole("navigation", { name: "Conta" });

describe("Flow — link Avisos no topo", () => {
  it("aparece ao lado de Sair na tela do termo e na tela de pessoas", async () => {
    signedIn();
    inbox();
    render(<Flow />);

    expect(await screen.findByRole("button", { name: "Concordo" })).toBeInTheDocument();
    expect(await within(bar()).findByRole("link", { name: "Avisos, 1 novo" })).toBeInTheDocument();
    expect(within(bar()).getByRole("button", { name: "Sair" })).toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Concordo" }));
    expect(await screen.findByText("Para quem é esta triagem?")).toBeInTheDocument();
    expect(await within(bar()).findByRole("link", { name: "Avisos, 1 novo" })).toBeInTheDocument();
  });

  it("sem sessão (carregando, celular, código) não há link nem barra", async () => {
    vi.spyOn(citizenApi, "currentSession").mockRejectedValue(new ApiError(401, "unauthenticated"));
    vi.spyOn(citizenApi, "requestCode").mockResolvedValue({ status: "sent", resend_after: 60 });
    const notices = vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [], unread_count: 1 });
    render(<Flow />);

    expect(screen.queryByRole("navigation", { name: "Conta" })).not.toBeInTheDocument(); // boot
    await userEvent.type(await screen.findByLabelText("Seu celular"), "41998765432");
    expect(screen.queryByRole("link", { name: /Avisos/ })).not.toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Receber código" }));
    expect(await screen.findByLabelText("Código")).toBeInTheDocument();
    expect(screen.queryByRole("link", { name: /Avisos/ })).not.toBeInTheDocument();
    expect(notices).not.toHaveBeenCalled();
  });

  it("da tela de pessoas: abre a caixa, ler tira o selo do topo na hora, e voltar volta às pessoas", async () => {
    signedIn();
    const { readNotice } = inbox();
    render(<Flow />);
    await userEvent.click(await screen.findByRole("button", { name: "Concordo" }));
    await screen.findByText("Para quem é esta triagem?");
    await userEvent.click(await within(bar()).findByRole("link", { name: "Avisos, 1 novo" }));

    expect(await screen.findByRole("heading", { name: "Avisos" })).toBeInTheDocument();
    await userEvent.click(await screen.findByRole("button", { name: /Vacinação/ }));
    await waitFor(() => expect(readNotice).toHaveBeenCalledWith("r1"));
    // Ainda no texto do aviso, o selo do topo já foi relido.
    expect(await within(bar()).findByRole("link", { name: "Avisos" })).toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Voltar aos avisos" }));
    await userEvent.click(screen.getByRole("button", { name: "Voltar ao início" }));
    expect(await screen.findByText("Para quem é esta triagem?")).toBeInTheDocument();
    expect(within(bar()).getByRole("link", { name: "Avisos" }).textContent).toBe("Avisos");
  });

  it("da tela do termo: a caixa abre sem aceitar o termo, e voltar leva ao termo", async () => {
    signedIn();
    inbox();
    render(<Flow />);
    await screen.findByRole("button", { name: "Concordo" });
    await userEvent.click(await within(bar()).findByRole("link", { name: "Avisos, 1 novo" }));

    expect(await screen.findByRole("heading", { name: "Avisos" })).toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Voltar ao início" }));
    expect(await screen.findByRole("button", { name: "Concordo" })).toBeInTheDocument();
  });

  it("api sem a rota de avisos: o topo mostra o link sem selo, sem erro, e a triagem segue", async () => {
    signedIn();
    vi.spyOn(citizenApi, "notices").mockRejectedValue(new ApiError(404, "not_found"));
    render(<Flow />);
    await userEvent.click(await screen.findByRole("button", { name: "Concordo" }));

    expect(await screen.findByRole("button", { name: "CPF ***.982.247-**" })).toBeInTheDocument();
    expect(within(bar()).getByRole("link", { name: "Avisos" })).toBeInTheDocument();
    expect(screen.queryByRole("alert")).not.toBeInTheDocument();
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/NoticesLink.test.tsx src/modules/citizen/Flow.notices.test.tsx`
Expected: FAIL (`./NoticesLink` não existe; não há barra "Conta").

- [ ] **Step 4: Implemente o link**

```tsx
// src/modules/citizen/NoticesLink.tsx
// Link "Avisos" do topo das telas logadas (spec 2026-09-29 §8), com o selo de
// não lidos. O api já desconta quem silenciou (unread_count 0 = sem selo).
// Falha ao buscar (api antiga, rede) = link sem selo e sem erro.
import { useEffect, useState, type MouseEvent } from "react";
import { citizenApi } from "../../lib/citizenApi";

export function badgeText(count: number): string | null {
  if (!(count > 0)) return null;
  return count > 99 ? "99+" : String(count);
}

export function NoticesLink({ onOpen }: { onOpen: () => void }) {
  const [unread, setUnread] = useState(0);

  useEffect(() => {
    let alive = true;
    citizenApi.notices()
      .then(r => { if (alive) setUnread(r.unread_count); })
      .catch(() => {});
    return () => { alive = false; };
  }, []);

  const badge = badgeText(unread);
  const label = badge ? `Avisos, ${unread} ${unread === 1 ? "novo" : "novos"}` : "Avisos";

  function open(e: MouseEvent<HTMLAnchorElement>) {
    e.preventDefault();
    onOpen();
  }

  // base do Vite ("/wpda/"), lida no render (os testes trocam com vi.stubEnv).
  return (
    <a href={`${import.meta.env.BASE_URL}avisos`} aria-label={label} onClick={open}
      style={{ display: "inline-flex", alignItems: "center", gap: 8, minHeight: 48, padding: "0 12px",
        borderRadius: 12, fontSize: 18, fontWeight: 600, textDecoration: "none",
        color: "var(--ink, #222)", border: "1px solid var(--rule2, #ccc)" }}>
      <span>Avisos</span>
      {badge && (
        <span aria-hidden="true"
          style={{ display: "inline-flex", alignItems: "center", justifyContent: "center", minWidth: 32,
            height: 32, padding: "0 8px", borderRadius: 16, fontSize: 18,
            background: "var(--down, #c0392b)", color: "#fff" }}>
          {badge}
        </span>
      )}
    </a>
  );
}
```

- [ ] **Step 5: Barra do topo e caixa no `Flow`**

Em `src/modules/citizen/Flow.tsx`:
- imports, depois de `import { CounterCodeStep } from "./CounterCodeStep";`:

```tsx
import { NoticesLink } from "./NoticesLink";
import { NoticesStep } from "./NoticesStep";
```

- no tipo `State`, antes de `| { at: "declined" };`:

```tsx
  | { at: "notices"; consentVersion: string | null }
```

- logo depois de `const [error, setError] = useState<string | null>(null);`:

```tsx
  // Muda a key do link do topo: o selo é relido depois de ler ou silenciar.
  const [badgeTick, setBadgeTick] = useState(0);
  const refreshBadge = () => setBadgeTick(t => t + 1);
```

- troque a constante `exit` inteira (`const exit = state.at !== "boot" && ... );`) por:

```tsx
  // Topo de toda tela logada (decisão do usuário, 2026-09-29): "Avisos" com o
  // selo e "Sair". A key relê o selo a cada troca de tela e a cada refreshBadge.
  const signedIn = state.at !== "boot" && state.at !== "phone" && state.at !== "code";
  const consentVersion = "consentVersion" in state ? state.consentVersion : null;
  const top = signedIn && (
    <nav aria-label="Conta"
      style={{ display: "flex", justifyContent: "flex-end", alignItems: "center", gap: 8,
        maxWidth: 520, margin: "0 auto", padding: "8px 16px 0" }}>
      <NoticesLink key={`${state.at}-${badgeTick}`}
        onOpen={() => setState({ at: "notices", consentVersion })} />
      <button type="button" onClick={signOut}
        style={{ minHeight: 48, padding: "0 12px", background: "none", border: "none", fontSize: 18 }}>
        Sair
      </button>
    </nav>
  );
```

- antes do `case "declined":`:

```tsx
    case "notices":
      // A caixa não depende do termo (ADR 0024): sem versão guardada, voltar
      // leva ao termo, que segue obrigatório para começar a triagem.
      view = <NoticesStep onRead={refreshBadge}
        onBack={() => setState(state.consentVersion ? { at: "people", consentVersion: state.consentVersion } : { at: "consent" })} />;
      break;
```

- troque `return <>{exit}{view}</>;` por `return <>{top}{view}</>;`.

- [ ] **Step 6: Rode os testes, a suíte e os tipos**

Run: `npx vitest run src/modules/citizen/NoticesLink.test.tsx src/modules/citizen/Flow.notices.test.tsx && npm test && npm run typecheck`
Expected: PASS. Os testes antigos do `Flow` (`Flow.test.tsx`) continuam verdes: eles acham "Sair" pelo nome, e sem mock de `notices` a busca do selo falha em silêncio (caso 4 do Review Focus).

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/NoticesLink.tsx src/modules/citizen/NoticesLink.test.tsx \
  src/modules/citizen/Flow.notices.test.tsx src/modules/citizen/Flow.tsx
/opt/homebrew/bin/git commit -m "feat: show the unread notices badge next to sign out on every signed-in screen" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Preferências de avisos

**Files:**
- Create: `src/modules/citizen/PreferencesStep.tsx`
- Modify: `src/modules/citizen/ui.tsx`, `src/modules/citizen/Flow.tsx`
- Test: `src/modules/citizen/PreferencesStep.test.tsx` (novo), `src/modules/citizen/Flow.notices.test.tsx`

**Interfaces:**
- Consumes (Task 1): `ContactPreference`, `ContactPreferences`, `ContactPreferenceChange`, `citizenApi.contactPreferences()`, `citizenApi.updateContactPreference()`. (Task 2): `NoticesStep` com `onPreferences` e `onRead`. (Task 3): estado `notices`, `refreshBadge()` e a barra "Conta" no `Flow`.
- Produces:
  - `Toggle({ label, description?, checked, disabled?, onChange }: { label: string; description?: string; checked: boolean; disabled?: boolean; onChange: (checked: boolean) => void })` em `ui.tsx` — `<input type="checkbox" role="switch">` dentro de um `<label>` de 48 px.
  - `PreferencesStep({ onBack, onSaved? }: { onBack: () => void; onSaved?: () => void })` — `onSaved` é chamado depois de cada PUT bem-sucedido (o `Flow` liga em `refreshBadge`, porque silenciar muda o selo do topo).
  - `export const SMS_EXPLANATION`, `export const MUTE_EXPLANATION` (em `PreferencesStep.tsx`).
  - `Flow`: estado `{ at: "preferences"; consentVersion: string | null }`.

**Comportamento (spec §8 "`/preferencias`"):**
- Título "Preferências de avisos". Uma seção por pessoa do telefone, com o título `CPF ***.982.247-**` (é também o nome acessível da seção).
- Com `sms_available`: interruptor "Receber avisos por SMS" com a explicação fixa. Sem `sms_available`: o interruptor não aparece.
- Sempre: interruptor "Silenciar avisos" com a explicação "Os avisos novos deixam de aparecer no número ao lado de Avisos. Eles continuam na lista de avisos.".
- Mudar um interruptor manda `PUT` só com o campo mudado; enquanto não volta, **todos** os interruptores ficam desabilitados (um PUT por vez). Sucesso: o estado vem da resposta, aparece "Preferência salva." na seção da pessoa e `onSaved` é chamado. Falha: o interruptor volta ao estado anterior e a seção mostra a mensagem de `messageFor`.
- Rodapé: "Voltar aos avisos". A tela é aberta por "Preferências de avisos" na caixa.

- [ ] **Step 1: Escreva os testes da tela**

```tsx
// src/modules/citizen/PreferencesStep.test.tsx
import { describe, it, expect, vi, afterEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { PreferencesStep, SMS_EXPLANATION, MUTE_EXPLANATION } from "./PreferencesStep";
import { citizenApi, type ContactPreference, type ContactPreferences } from "../../lib/citizenApi";

afterEach(() => vi.restoreAllMocks());

const p1: ContactPreference = { citizen_id: "p1", cpf_masked: "***.982.247-**", sms_opt_in: false, notices_muted: false };
const p2: ContactPreference = { citizen_id: "p2", cpf_masked: "***.111.222-**", sms_opt_in: true, notices_muted: true };

function setup(prefs: ContactPreferences = { sms_available: true, people: [ p1, p2 ] }) {
  const get = vi.spyOn(citizenApi, "contactPreferences").mockResolvedValue(prefs);
  const put = vi.spyOn(citizenApi, "updateContactPreference");
  const onBack = vi.fn();
  const onSaved = vi.fn();
  render(<PreferencesStep onBack={onBack} onSaved={onSaved} />);
  return { get, put, onBack, onSaved };
}

const person = (cpf: string) => screen.findByRole("region", { name: `CPF ${cpf}` });
const smsSwitch = (el: HTMLElement) => within(el).getByRole("switch", { name: "Receber avisos por SMS" });
const muteSwitch = (el: HTMLElement) => within(el).getByRole("switch", { name: "Silenciar avisos" });

describe("PreferencesStep", () => {
  it("com SMS disponível: cada pessoa tem os dois interruptores, com o estado da API e as explicações", async () => {
    setup();
    const a = await person("***.982.247-**");
    expect(smsSwitch(a)).not.toBeChecked();
    expect(muteSwitch(a)).not.toBeChecked();
    expect(within(a).getByText(SMS_EXPLANATION)).toBeInTheDocument();
    expect(within(a).getByText(MUTE_EXPLANATION)).toBeInTheDocument();
    expect(SMS_EXPLANATION).toBe(
      "A Secretaria de Saúde pode enviar um SMS avisando que há um aviso novo aqui. Você pode desligar quando quiser.");

    const b = screen.getByRole("region", { name: "CPF ***.111.222-**" });
    expect(smsSwitch(b)).toBeChecked();
    expect(muteSwitch(b)).toBeChecked();
  });

  it("sem SMS na cidade: o interruptor de SMS não aparece; o de silêncio sim", async () => {
    setup({ sms_available: false, people: [ p1 ] });
    const a = await person("***.982.247-**");
    expect(within(a).queryByRole("switch", { name: "Receber avisos por SMS" })).not.toBeInTheDocument();
    expect(screen.queryByText(SMS_EXPLANATION)).not.toBeInTheDocument();
    expect(muteSwitch(a)).toBeInTheDocument();
  });

  it("ligar o SMS manda só sms_opt_in e mostra o estado salvo", async () => {
    const { put } = setup();
    put.mockResolvedValue({ ...p1, sms_opt_in: true });
    const a = await person("***.982.247-**");
    await userEvent.click(smsSwitch(a));

    expect(put).toHaveBeenCalledWith("p1", { sms_opt_in: true });
    expect(await within(a).findByText("Preferência salva.")).toBeInTheDocument();
    expect(smsSwitch(a)).toBeChecked();
    expect(muteSwitch(a)).not.toBeChecked();
  });

  it("silenciar manda só notices_muted e avisa o Flow (onSaved) para reler o selo", async () => {
    const { put, onSaved } = setup();
    put.mockResolvedValue({ ...p1, notices_muted: true });
    const a = await person("***.982.247-**");
    await userEvent.click(muteSwitch(a));

    expect(put).toHaveBeenCalledWith("p1", { notices_muted: true });
    expect(await within(a).findByText("Preferência salva.")).toBeInTheDocument();
    expect(muteSwitch(a)).toBeChecked();
    expect(onSaved).toHaveBeenCalledTimes(1);
  });

  it("desligar o SMS de quem tinha ligado manda false", async () => {
    const { put } = setup();
    put.mockResolvedValue({ ...p2, sms_opt_in: false });
    const b = await person("***.111.222-**");
    await userEvent.click(smsSwitch(b));
    expect(put).toHaveBeenCalledWith("p2", { sms_opt_in: false });
    expect(await within(b).findByText("Preferência salva.")).toBeInTheDocument();
    expect(smsSwitch(b)).not.toBeChecked();
  });

  it("falha ao salvar: o interruptor volta, a seção mostra a mensagem e o Flow não é avisado", async () => {
    const { put, onSaved } = setup();
    put.mockRejectedValue(new TypeError("offline"));
    const a = await person("***.982.247-**");
    await userEvent.click(smsSwitch(a));

    expect(await within(a).findByText("Sem conexão. Verifique a internet e tente de novo.")).toBeInTheDocument();
    expect(smsSwitch(a)).not.toBeChecked();
    expect(smsSwitch(a)).toBeEnabled();
    expect(onSaved).not.toHaveBeenCalled();
  });

  it("toque duplo: um PUT só, e os interruptores ficam desabilitados enquanto salva", async () => {
    const { put } = setup();
    put.mockReturnValue(new Promise<ContactPreference>(() => {}));
    const a = await person("***.982.247-**");
    await userEvent.click(smsSwitch(a));
    await userEvent.click(smsSwitch(a));
    await userEvent.click(muteSwitch(screen.getByRole("region", { name: "CPF ***.111.222-**" })));

    expect(put).toHaveBeenCalledTimes(1);
    expect(smsSwitch(a)).toBeDisabled();
    expect(muteSwitch(a)).toBeDisabled();
  });

  it("erro ao carregar: mensagem e 'Tentar de novo' recarrega", async () => {
    const get = vi.spyOn(citizenApi, "contactPreferences")
      .mockRejectedValueOnce(new TypeError("offline"))
      .mockResolvedValue({ sms_available: false, people: [ p1 ] });
    render(<PreferencesStep onBack={vi.fn()} />);
    expect(await screen.findByText("Sem conexão. Verifique a internet e tente de novo.")).toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Tentar de novo" }));
    expect(await person("***.982.247-**")).toBeInTheDocument();
    expect(get).toHaveBeenCalledTimes(2);
  });

  it("alvo de toque com pelo menos 48 px e texto com pelo menos 18 px; rodapé volta aos avisos", async () => {
    const { onBack } = setup();
    const label = muteSwitch(await person("***.982.247-**")).closest("label") as HTMLElement;
    expect(parseInt(label.style.minHeight, 10)).toBeGreaterThanOrEqual(48);
    expect(parseInt(label.style.fontSize, 10)).toBeGreaterThanOrEqual(18);
    await userEvent.click(screen.getByRole("button", { name: "Voltar aos avisos" }));
    expect(onBack).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 2: Escreva o teste de ponta a ponta no `Flow`**

Acrescente ao fim do `describe("Flow — link Avisos no topo", ...)` em `src/modules/citizen/Flow.notices.test.tsx` (usa `signedIn`, `inbox` e `bar` da Task 3):

```tsx
  it("silenciar nas preferências tira o selo do topo na hora; os avisos continuam na lista como 'novo'", async () => {
    signedIn();
    const { box } = inbox();
    vi.spyOn(citizenApi, "contactPreferences").mockResolvedValue({
      sms_available: false,
      people: [ { citizen_id: "p1", cpf_masked: "***.982.247-**", sms_opt_in: false, notices_muted: false } ]
    });
    // Silenciar zera o selo no api, mas o aviso continua não lido.
    const update = vi.spyOn(citizenApi, "updateContactPreference").mockImplementation(async () => {
      box.unread = 0;
      return { citizen_id: "p1", cpf_masked: "***.982.247-**", sms_opt_in: false, notices_muted: true };
    });
    vi.mocked(citizenApi.notices).mockImplementation(async () => ({
      notices: [ notice ], unread_count: box.unread
    }));

    render(<Flow />);
    await userEvent.click(await screen.findByRole("button", { name: "Concordo" }));
    await userEvent.click(await within(bar()).findByRole("link", { name: "Avisos, 1 novo" }));
    await userEvent.click(await screen.findByRole("button", { name: "Preferências de avisos" }));

    expect(await screen.findByRole("heading", { name: "Preferências de avisos" })).toBeInTheDocument();
    await userEvent.click(await screen.findByRole("switch", { name: "Silenciar avisos" }));
    expect(update).toHaveBeenCalledWith("p1", { notices_muted: true });
    expect(await screen.findByText("Preferência salva.")).toBeInTheDocument();
    // Ainda nas preferências, o selo do topo já foi relido.
    expect(await within(bar()).findByRole("link", { name: "Avisos" })).toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Voltar aos avisos" }));
    expect(within(await screen.findByRole("button", { name: /Vacinação/ })).getByText("novo")).toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Voltar ao início" }));
    expect(await screen.findByText("Para quem é esta triagem?")).toBeInTheDocument();
    expect(within(bar()).getByRole("link", { name: "Avisos" }).textContent).toBe("Avisos");
  });
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/PreferencesStep.test.tsx src/modules/citizen/Flow.notices.test.tsx`
Expected: FAIL (`./PreferencesStep` não existe; a caixa no `Flow` não tem "Preferências de avisos").

- [ ] **Step 4: Implemente o `Toggle`**

Acrescente em `src/modules/citizen/ui.tsx`, depois de `ErrorText`:

```tsx
// Interruptor liga/desliga: o rótulo inteiro é o alvo de toque (48 px ou mais).
export function Toggle({ label, description, checked, disabled, onChange }:
  { label: string; description?: string; checked: boolean; disabled?: boolean; onChange: (checked: boolean) => void }) {
  const id = useId();
  const descriptionId = `${id}-description`;
  return (
    <div style={{ display: "grid", gap: 4, marginBottom: 16 }}>
      <label htmlFor={id}
        style={{ display: "flex", alignItems: "center", justifyContent: "space-between", gap: 12, minHeight: 48,
          fontSize: 18, fontWeight: 600, cursor: disabled ? "default" : "pointer", opacity: disabled ? 0.6 : 1 }}>
        {label}
        <input id={id} type="checkbox" role="switch" checked={checked} disabled={disabled}
          aria-describedby={description ? descriptionId : undefined}
          onChange={e => onChange(e.target.checked)}
          style={{ width: 28, height: 28, flexShrink: 0, margin: 0 }} />
      </label>
      {description && <p id={descriptionId} style={{ margin: 0, fontSize: 18, color: "var(--ink2, #555)" }}>{description}</p>}
    </div>
  );
}
```

- [ ] **Step 5: Implemente a tela**

```tsx
// src/modules/citizen/PreferencesStep.tsx
// Preferências de aviso por pessoa do telefone (spec 2026-09-29 §8; ADR 0024):
// opt-in de SMS (só quando a cidade liga o SMS) e silêncio dos avisos (tira o
// selo; os avisos continuam na lista). Um PUT por vez, com o campo mudado.
import { useEffect, useRef, useState } from "react";
import {
  citizenApi, type ContactPreference, type ContactPreferenceChange, type ContactPreferences
} from "../../lib/citizenApi";
import { BigButton, ErrorText, Screen, Toggle, messageFor } from "./ui";

export const SMS_EXPLANATION =
  "A Secretaria de Saúde pode enviar um SMS avisando que há um aviso novo aqui. Você pode desligar quando quiser.";
export const MUTE_EXPLANATION =
  "Os avisos novos deixam de aparecer no número ao lado de Avisos. Eles continuam na lista de avisos.";

type Feedback = { citizenId: string; ok: boolean; text: string };

export function PreferencesStep({ onBack, onSaved }: { onBack: () => void; onSaved?: () => void }) {
  const [prefs, setPrefs] = useState<ContactPreferences | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [saving, setSaving] = useState(false);
  const [feedback, setFeedback] = useState<Feedback | null>(null);
  // Guarda síncrona: setSaving só vale no próximo render, e dois toques cabem antes dele.
  const inFlight = useRef(false);

  function load() {
    setError(null);
    citizenApi.contactPreferences().then(setPrefs).catch(e => setError(messageFor(e)));
  }

  useEffect(() => { load(); }, []);

  async function change(person: ContactPreference, patch: ContactPreferenceChange) {
    if (inFlight.current) return;
    inFlight.current = true;
    setSaving(true);
    setFeedback(null);
    try {
      const saved = await citizenApi.updateContactPreference(person.citizen_id, patch);
      setPrefs(p => p && {
        ...p,
        people: p.people.map(x => x.citizen_id === person.citizen_id
          ? { ...x, sms_opt_in: saved.sms_opt_in, notices_muted: saved.notices_muted }
          : x)
      });
      setFeedback({ citizenId: person.citizen_id, ok: true, text: "Preferência salva." });
      onSaved?.();
    } catch (e) {
      // O estado não mudou: o interruptor (controlado) volta sozinho.
      setFeedback({ citizenId: person.citizen_id, ok: false, text: messageFor(e) });
    } finally {
      inFlight.current = false;
      setSaving(false);
    }
  }

  return (
    <Screen title="Preferências de avisos"
      footer={<BigButton variant="secondary" onClick={onBack}>Voltar aos avisos</BigButton>}>
      {error && <>
        <ErrorText>{error}</ErrorText>
        <BigButton style={{ marginTop: 12 }} onClick={load}>Tentar de novo</BigButton>
      </>}
      {prefs === null && !error && <p>Carregando…</p>}
      {prefs?.people.map(p => {
        const title = `CPF ${p.cpf_masked}`;
        const mine = feedback?.citizenId === p.citizen_id ? feedback : null;
        return (
          <section key={p.citizen_id} aria-label={title}
            style={{ marginBottom: 24, padding: 12, borderRadius: 12, border: "1px solid var(--rule2, #ccc)" }}>
            <h2 style={{ fontSize: 20, margin: "0 0 8px" }}>{title}</h2>
            {prefs.sms_available &&
              <Toggle label="Receber avisos por SMS" description={SMS_EXPLANATION}
                checked={p.sms_opt_in} disabled={saving}
                onChange={v => void change(p, { sms_opt_in: v })} />}
            <Toggle label="Silenciar avisos" description={MUTE_EXPLANATION}
              checked={p.notices_muted} disabled={saving}
              onChange={v => void change(p, { notices_muted: v })} />
            {mine && (mine.ok ? <p role="status" style={{ margin: 0 }}>{mine.text}</p> : <ErrorText>{mine.text}</ErrorText>)}
          </section>
        );
      })}
    </Screen>
  );
}
```

- [ ] **Step 6: Ligue as preferências no `Flow`**

Em `src/modules/citizen/Flow.tsx`:
- import, depois de `import { NoticesStep } from "./NoticesStep";`: `import { PreferencesStep } from "./PreferencesStep";`
- no tipo `State`, depois de `| { at: "notices"; consentVersion: string | null }`:

```tsx
  | { at: "preferences"; consentVersion: string | null }
```

- troque o `case "notices":` por:

```tsx
    case "notices":
      // A caixa não depende do termo (ADR 0024): sem versão guardada, voltar
      // leva ao termo, que segue obrigatório para começar a triagem.
      view = <NoticesStep onRead={refreshBadge}
        onPreferences={() => setState({ at: "preferences", consentVersion: state.consentVersion })}
        onBack={() => setState(state.consentVersion ? { at: "people", consentVersion: state.consentVersion } : { at: "consent" })} />;
      break;
    case "preferences":
      view = <PreferencesStep onSaved={refreshBadge}
        onBack={() => setState({ at: "notices", consentVersion: state.consentVersion })} />;
      break;
```

- [ ] **Step 7: Rode os testes, a suíte e os tipos**

Run: `npx vitest run src/modules/citizen/PreferencesStep.test.tsx src/modules/citizen/Flow.notices.test.tsx && npm test && npm run typecheck`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/PreferencesStep.tsx src/modules/citizen/PreferencesStep.test.tsx \
  src/modules/citizen/ui.tsx src/modules/citizen/Flow.tsx src/modules/citizen/Flow.notices.test.tsx
/opt/homebrew/bin/git commit -m "feat: let the citizen choose sms and notice preferences" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Endereço `/avisos` e volta depois do login

**Files:**
- Create: `src/lib/route.ts`
- Modify: `src/modules/citizen/Flow.tsx`
- Test: `src/lib/route.test.ts` (novo), `src/modules/citizen/Flow.route.test.tsx` (novo)

**Interfaces:**
- Consumes (Tasks 3 e 4): estados `{ at: "notices"; consentVersion }` e `{ at: "preferences"; consentVersion }` do `Flow`; `badgeTick`/`refreshBadge` e a barra "Conta" (com "Avisos" e "Sair") da Task 3, que esta task não muda.
- Produces (em `src/lib/route.ts`):
  - `type AppRoute = "avisos" | "preferencias"`
  - `routeFromPath(pathname: string, base: string): AppRoute | null`
  - `pathFor(route: AppRoute | null, base: string): string`

**Comportamento (spec §8 "Link do SMS"):**
1. No carregamento, o `Flow` lê o destino da URL (`<base>avisos` ou `<base>preferencias`, com ou sem `/` no fim). Qualquer outro caminho = sem destino (fluxo de sempre).
2. **Com sessão:** vai direto ao destino, sem o termo (estado com `consentVersion: null`). Sem destino, abre o termo, como hoje.
3. **Sem sessão:** celular → código → destino (sem o termo). O destino vale uma vez: depois de usado, é esquecido.
4. **Sessão cai (401) na caixa ou nas preferências:** volta ao login e, depois dele, à mesma tela.
5. **"Sair"** esquece o destino e põe a URL na base.
6. **URL em dia:** fora das telas de login (`boot`, `phone`, `code`, onde a URL pedida é mantida para o F5), a URL mostra `<base>avisos` na caixa, `<base>preferencias` nas preferências e a base nas outras telas (`history.replaceState`, sem criar histórico).
7. O efeito de boot ignora a resposta de uma montagem já desfeita (o `StrictMode` do `main.tsx` monta duas vezes em dev; sem isso, a segunda resposta, sem destino, levaria ao termo).

- [ ] **Step 1: Escreva os testes das regras de endereço**

```ts
// src/lib/route.test.ts
import { describe, expect, it } from "vitest";
import { pathFor, routeFromPath } from "./route";

describe("routeFromPath", () => {
  it.each([
    [ "/wpda/avisos", "/wpda/", "avisos" ],
    [ "/wpda/avisos/", "/wpda/", "avisos" ],
    [ "/wpda/preferencias", "/wpda/", "preferencias" ],
    [ "/avisos", "/", "avisos" ],
    [ "/wpda/avisos", "/wpda", "avisos" ]
  ])("%s com base %s → %s", (path, base, route) => expect(routeFromPath(path, base)).toBe(route));

  it.each([
    [ "/wpda/", "/wpda/" ],
    [ "/wpda", "/wpda/" ],
    [ "/wpda/avisosx", "/wpda/" ],
    [ "/wpda/avisos/r1", "/wpda/" ],
    [ "/avisos", "/wpda/" ],
    [ "/wpda/qualquer", "/wpda/" ],
    [ "/", "/" ]
  ])("%s com base %s → sem destino", (path, base) => expect(routeFromPath(path, base)).toBeNull());
});

describe("pathFor", () => {
  it("monta o caminho sob a base", () => {
    expect(pathFor("avisos", "/wpda/")).toBe("/wpda/avisos");
    expect(pathFor("preferencias", "/wpda")).toBe("/wpda/preferencias");
    expect(pathFor(null, "/wpda/")).toBe("/wpda/");
    expect(pathFor(null, "/")).toBe("/");
  });
});
```

- [ ] **Step 2: Escreva os testes do `Flow`**

```tsx
// src/modules/citizen/Flow.route.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { act, render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Flow } from "./Flow";
import { citizenApi, ApiError, type Notice } from "../../lib/citizenApi";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-09-29T10:00:00-03:00"));
  vi.stubEnv("BASE_URL", "/wpda/");
});
afterEach(() => {
  vi.useRealTimers();
  vi.unstubAllEnvs();
  vi.restoreAllMocks();
  window.history.replaceState(null, "", "/");
});

const notice: Notice = {
  id: "r1", title: "Vacinação contra a gripe", body: "Leve a carteirinha.",
  dispatched_at: "2026-09-28T13:00:00-03:00", read: false, cpf_masked: null
};

function at(path: string) {
  window.history.replaceState(null, "", path);
}

function api({ session }: { session: boolean }) {
  const currentSession = session
    ? vi.spyOn(citizenApi, "currentSession").mockResolvedValue({ phone_masked: "(**) *****-5432" })
    : vi.spyOn(citizenApi, "currentSession").mockRejectedValue(new ApiError(401, "unauthenticated"));
  const term = vi.spyOn(citizenApi, "consentTerm").mockResolvedValue({ version: "1", body: "Termo" });
  vi.spyOn(citizenApi, "people").mockResolvedValue({
    people: [{ id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared" }]
  });
  vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue([]);
  vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [ notice ], unread_count: 1 });
  vi.spyOn(citizenApi, "contactPreferences").mockResolvedValue({ sms_available: false, people: [] });
  vi.spyOn(citizenApi, "requestCode").mockResolvedValue({ status: "sent", resend_after: 60 });
  vi.spyOn(citizenApi, "verifyCode").mockResolvedValue({ phone_masked: "(**) *****-5432" });
  vi.spyOn(citizenApi, "signOut").mockResolvedValue(undefined);
  return { currentSession, term };
}

async function signIn() {
  await userEvent.type(await screen.findByLabelText("Seu celular"), "41998765432");
  await userEvent.click(screen.getByRole("button", { name: "Receber código" }));
  await userEvent.type(await screen.findByLabelText("Código"), "123456");
  await userEvent.click(screen.getByRole("button", { name: "Confirmar" }));
}

describe("Flow — link do SMS (/avisos)", () => {
  it("sem sessão: passa pelo login e volta à caixa de avisos, sem o termo", async () => {
    at("/wpda/avisos");
    const { term } = api({ session: false });
    render(<Flow />);

    await signIn();

    expect(await screen.findByRole("heading", { name: "Avisos" })).toBeInTheDocument();
    expect(await screen.findByRole("button", { name: /Vacinação/ })).toBeInTheDocument();
    expect(term).not.toHaveBeenCalled();
    expect(window.location.pathname).toBe("/wpda/avisos");
  });

  it("durante o login, a URL pedida fica (F5 no código volta ao mesmo destino)", async () => {
    at("/wpda/avisos");
    api({ session: false });
    render(<Flow />);
    await userEvent.type(await screen.findByLabelText("Seu celular"), "41998765432");
    await userEvent.click(screen.getByRole("button", { name: "Receber código" }));
    await screen.findByLabelText("Código");
    expect(window.location.pathname).toBe("/wpda/avisos");
  });

  it("com sessão: abre a caixa direto; 'Voltar ao início' leva ao termo e a URL volta à base", async () => {
    at("/wpda/avisos/");
    const { term } = api({ session: true });
    render(<Flow />);

    expect(await screen.findByRole("heading", { name: "Avisos" })).toBeInTheDocument();
    expect(term).not.toHaveBeenCalled();
    await userEvent.click(screen.getByRole("button", { name: "Voltar ao início" }));

    expect(await screen.findByRole("button", { name: "Concordo" })).toBeInTheDocument();
    await waitFor(() => expect(window.location.pathname).toBe("/wpda/"));
  });

  it("/preferencias com sessão abre as preferências", async () => {
    at("/wpda/preferencias");
    api({ session: true });
    render(<Flow />);
    expect(await screen.findByRole("heading", { name: "Preferências de avisos" })).toBeInTheDocument();
    expect(window.location.pathname).toBe("/wpda/preferencias");
  });

  it("caminho desconhecido segue o fluxo de sempre (termo)", async () => {
    at("/wpda/qualquer");
    api({ session: true });
    render(<Flow />);
    expect(await screen.findByRole("button", { name: "Concordo" })).toBeInTheDocument();
    await waitFor(() => expect(window.location.pathname).toBe("/wpda/"));
  });

  it("sessão cai (401) na caixa: login de novo e volta à caixa", async () => {
    at("/wpda/avisos");
    const { term } = api({ session: true });
    render(<Flow />);
    expect(await screen.findByRole("heading", { name: "Avisos" })).toBeInTheDocument();

    act(() => window.dispatchEvent(new Event("citizen:unauthenticated")));
    await signIn();

    expect(await screen.findByRole("heading", { name: "Avisos" })).toBeInTheDocument();
    expect(term).not.toHaveBeenCalled();
  });

  it("sessão cai (401) nas preferências: volta às preferências", async () => {
    at("/wpda/preferencias");
    api({ session: true });
    render(<Flow />);
    expect(await screen.findByRole("heading", { name: "Preferências de avisos" })).toBeInTheDocument();

    act(() => window.dispatchEvent(new Event("citizen:unauthenticated")));
    await signIn();

    expect(await screen.findByRole("heading", { name: "Preferências de avisos" })).toBeInTheDocument();
  });

  it("'Sair' esquece o destino: o próximo login abre o termo e a URL fica na base", async () => {
    at("/wpda/avisos");
    api({ session: true });
    render(<Flow />);
    expect(await screen.findByRole("heading", { name: "Avisos" })).toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Sair" }));
    expect(await screen.findByLabelText("Seu celular")).toBeInTheDocument();
    expect(window.location.pathname).toBe("/wpda/");
    expect(screen.queryByRole("link", { name: /Avisos/ })).not.toBeInTheDocument();

    await signIn();
    expect(await screen.findByRole("button", { name: "Concordo" })).toBeInTheDocument();
  });

  it("abrir Avisos pela tela inicial põe /avisos na URL", async () => {
    at("/wpda/");
    api({ session: true });
    render(<Flow />);
    await userEvent.click(await screen.findByRole("button", { name: "Concordo" }));
    await screen.findByText("Para quem é esta triagem?");
    await userEvent.click(await screen.findByRole("link", { name: /^Avisos/ }));

    expect(await screen.findByRole("heading", { name: "Avisos" })).toBeInTheDocument();
    await waitFor(() => expect(window.location.pathname).toBe("/wpda/avisos"));
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/lib/route.test.ts src/modules/citizen/Flow.route.test.tsx`
Expected: FAIL (`./route` não existe; o `Flow` abre o termo em vez da caixa).

- [ ] **Step 4: Implemente as regras de endereço**

```ts
// src/lib/route.ts
// Endereços próprios do wpda (spec 2026-09-29 §8): o link do SMS é
// <base>/avisos. O resto do canal segue sem URL: é a máquina de telas do Flow.
export type AppRoute = "avisos" | "preferencias";

const ROUTES: readonly string[] = [ "avisos", "preferencias" ];

function withSlash(base: string): string {
  return base.endsWith("/") ? base : `${base}/`;
}

export function routeFromPath(pathname: string, base: string): AppRoute | null {
  const b = withSlash(base);
  if (!pathname.startsWith(b)) return null;
  const rest = pathname.slice(b.length).replace(/\/+$/, "");
  return ROUTES.includes(rest) ? (rest as AppRoute) : null;
}

export function pathFor(route: AppRoute | null, base: string): string {
  return route ? `${withSlash(base)}${route}` : withSlash(base);
}
```

- [ ] **Step 5: Guarde o destino e mantenha a URL no `Flow`**

Em `src/modules/citizen/Flow.tsx`:
- troque o comentário do topo (linhas 1-2) por:

```tsx
// Máquina de telas do canal web do cidadão (spec §4). Sem biblioteca de rotas:
// o estado mora aqui. Só a caixa de avisos e as preferências têm endereço
// (<base>avisos, <base>preferencias; spec 2026-09-29 §8): o link do SMS passa
// pelo login e volta para lá. Nas outras telas, um F5 volta ao começo.
```

- troque `import { useCallback, useEffect, useState } from "react";` por `import { useCallback, useEffect, useRef, useState } from "react";` e acrescente `import { pathFor, routeFromPath, type AppRoute } from "../../lib/route";` depois do import de `citizenApi`;
- troque o começo do componente, de `export function Flow() {` até o fim do segundo `useEffect` (o do `citizen:unauthenticated`), por (o `badgeTick`/`refreshBadge` da Task 3 continua igual):

```tsx
export function Flow() {
  const [state, setState] = useState<State>({ at: "boot" });
  const [error, setError] = useState<string | null>(null);
  // Muda a key do link do topo: o selo é relido depois de ler ou silenciar.
  const [badgeTick, setBadgeTick] = useState(0);
  const refreshBadge = () => setBadgeTick(t => t + 1);
  // Destino pedido pela URL no carregamento (link do SMS). Vale uma vez; a
  // sessão que cai numa dessas telas o guarda de novo; "Sair" o esquece.
  const target = useRef<AppRoute | null>(routeFromPath(window.location.pathname, import.meta.env.BASE_URL));
  const stateRef = useRef(state);
  stateRef.current = state;

  // Depois da sessão confirmada: o destino pedido, ou o termo, como sempre.
  // A caixa de avisos não depende do termo (ADR 0024).
  function enter() {
    const route = target.current;
    target.current = null;
    if (route === "avisos") setState({ at: "notices", consentVersion: null });
    else if (route === "preferencias") setState({ at: "preferences", consentVersion: null });
    else setState({ at: "consent" });
  }

  useEffect(() => {
    // StrictMode monta duas vezes em dev: a resposta da montagem desfeita não
    // pode gastar o destino nem sobrescrever a tela.
    let alive = true;
    citizenApi.currentSession()
      .then(() => { if (alive) enter(); })
      .catch(() => { if (alive) setState({ at: "phone" }); });
    return () => { alive = false; };
  }, []);

  useEffect(() => {
    function onUnauthenticated() {
      const s = stateRef.current;
      if (s.at === "notices") target.current = "avisos";
      else if (s.at === "preferences") target.current = "preferencias";
      setState({ at: "phone" });
    }
    window.addEventListener("citizen:unauthenticated", onUnauthenticated);
    return () => window.removeEventListener("citizen:unauthenticated", onUnauthenticated);
  }, []);

  // URL em dia fora do login (no login, a URL pedida fica para o F5).
  useEffect(() => {
    if (state.at === "boot" || state.at === "phone" || state.at === "code") return;
    const route: AppRoute | null = state.at === "notices" ? "avisos" : state.at === "preferences" ? "preferencias" : null;
    const path = pathFor(route, import.meta.env.BASE_URL);
    if (window.location.pathname !== path) window.history.replaceState(null, "", path);
  }, [state.at]);
```

- troque a função `signOut` por:

```tsx
  async function signOut() {
    target.current = null;
    window.history.replaceState(null, "", pathFor(null, import.meta.env.BASE_URL));
    try { await citizenApi.signOut(); } finally { setState({ at: "phone" }); }
  }
```

- troque o `case "code":` por:

```tsx
    case "code":
      view = <CodeStep phone={state.phone} onVerified={enter}
        onChangePhone={() => setState({ at: "phone" })} />;
      break;
```

- [ ] **Step 6: Rode os testes, a suíte e os tipos**

Run: `npx vitest run src/lib/route.test.ts src/modules/citizen/Flow.route.test.tsx && npm test && npm run typecheck`
Expected: PASS. Os testes antigos do `Flow` rodam em `/` com `BASE_URL` `/`: `routeFromPath("/", "/")` é `null` e `pathFor(null, "/")` é `/`, então nada muda para eles.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/lib/route.ts src/lib/route.test.ts src/modules/citizen/Flow.tsx \
  src/modules/citizen/Flow.route.test.tsx
/opt/homebrew/bin/git commit -m "feat: return to the notices inbox after signing in from the sms link" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Revisão e prova no navegador

**Files:** nenhum, a não ser que a prova ache um bug (nesse caso, teste + correção + commit `fix:` no wpda; se o bug for do api, avise a sessão dona do api em vez de mexer lá).

- [ ] **Step 1: Revisão do branch.** Um subagente revisor lê `/opt/homebrew/bin/git diff origin/main..HEAD` do wpda contra a spec §6.2 e §8 e o contrato fixado, com atenção a:
  - nenhum campo além dos do contrato (`notices[].{id,title,body,dispatched_at,read,cpf_masked}`, `unread_count`, `sms_available`, `people[].{citizen_id,cpf_masked,sms_opt_in,notices_muted}`); o PUT manda só o campo mudado;
  - nenhum `dangerouslySetInnerHTML` (`/opt/homebrew/bin/git diff origin/main..HEAD | grep -n dangerouslySetInnerHTML` vazio);
  - os três textos fixos exatamente como na spec;
  - todo `href` novo com `import.meta.env.BASE_URL`, lido no render;
  - todo `button`/`a`/`label` novo com `fontSize` ≥ 18 e alvo ≥ 48 px;
  - `vi.setSystemTime` em `beforeEach` em todo teste novo que mostra data; `replaceState`/`unstubAllEnvs` no `afterEach` de quem mexe na URL;
  - commits só com arquivos pelo nome (`/opt/homebrew/bin/git show --stat origin/main..HEAD` sem `node_modules` nem `.superpowers/`).
- [ ] **Step 2: Confira o contrato com o api.** Com o api do módulo 12 rodando, `GET /citizen/notices` e `GET /citizen/contact_preferences` devolvem os envelopes do contrato fixado (`{ notices, unread_count }` e `{ sms_available, people }`). Se divergir, pare e avise; não adapte o cliente a outro formato sem decisão.
- [ ] **Step 3: Prova no navegador (Curitiba).** Com o api temporário do branch do módulo 12 em `:3031`, que o plano do api sobe (migração de cidade aplicada, semente da spec §10: um telefone com dois CPFs, cidadãos com opt-in, uma campanha enviada pelo dashboard) e o Vite deste worktree:

  ```bash
  VITE_API_PROXY_TARGET=http://localhost:3031 npx vite --port 5187
  ```

  Abra `http://curitiba.localhost:5187/wpda/avisos` numa janela sem sessão. O login do cidadão (celular + código do log do api, `grep "[otp]"`) é feito pelo usuário; não digite o código. Depois, com screenshot de cada passo:
  1. depois do código, a tela é "Avisos" (sem o termo) e a URL continua `/wpda/avisos`;
  2. com o telefone de dois CPFs, cada aviso mostra "Para o CPF …"; tocar num aviso "novo" abre o texto com as quebras de linha; voltar e ver o aviso sem "novo";
  3. "Voltar ao início" → termo, já com "Avisos" e o selo no topo ao lado de "Sair" → "Concordo" → "Para quem é esta triagem?", com o mesmo link no topo e o selo coerente com o que sobrou não lido; na janela sem sessão (celular/código), não há link;
  4. "Avisos" → "Preferências de avisos": com a chave de SMS da cidade desligada, só "Silenciar avisos"; ligar a chave no dashboard (`municipal_admin`) e recarregar: aparece "Receber avisos por SMS" com a explicação; ligar e ver "Preferência salva.";
  5. silenciar todas as pessoas do telefone → voltar ao início → sem selo; a caixa continua mostrando os não lidos com "novo";
  6. "Sair" em `/wpda/avisos` → a URL vai para `/wpda/` e o próximo login abre o termo.
- [ ] **Step 4: Suíte final.** `npm test` e `npm run typecheck` no worktree; registre a contagem no relatório (antes → depois).
- [ ] **Step 5: Pare.** O merge do wpda vem depois do merge do api (spec §11: api → dashboard → wpda), e só com autorização explícita do usuário. Não faça push.

---

## Self-review

- **Cobertura da spec:** §8 "Início: selo com `unread_count` num link 'Avisos' (some quando 0)" → Task 3, no topo de toda tela logada por decisão do usuário (testes na tela do termo, na de pessoas e sem sessão); "`/avisos`: lista (título, data, 'novo'); tocar abre o texto e marca lido; CPF mascarado com mais de uma pessoa; vazio" → Task 2 (com a rota na Task 5); "`/preferencias`: por pessoa, SMS só com `sms_available`, com a explicação; 'Silenciar avisos' tira o selo e os avisos continuam na lista" → Task 4 (rota na Task 5); "Link do SMS: `/wpda/avisos` sem sessão passa pelo login e volta a `/avisos`" → Task 5; "texto ≥ 18px, alvos ≥ 48px, links com `BASE_URL`" → testes de estilo nas Tasks 2, 3 e 4 e `href` na Task 3; §6.2 (as quatro rotas, 404 cruzado) → Task 1; §9.4 (selo e silêncio, marcar lido, várias pessoas, SMS escondido sem `sms_available`, retorno do login, relógio fixo) → Tasks 2, 3, 4 e 5; §9.5 (prova no navegador do lado do cidadão) → Task 6.
- **Placeholders:** nenhum passo sem código; os ajustes em arquivos existentes dizem a linha âncora. A porta do api na prova é 3031 (api temporário do plano do api).
- **Tipos:** `Notice`, `NoticesResult`, `ContactPreference`, `ContactPreferences`, `ContactPreferenceChange` (Task 1) são os usados nas Tasks 2 e 4; `NoticesStep({ onBack, onPreferences?, onRead? })` (Task 2) e `PreferencesStep({ onBack, onSaved? })` (Task 4) são os que o `Flow` monta, com `onRead`/`onSaved` ligados a `refreshBadge` (Tasks 3 e 4); `badgeTick`/`refreshBadge` nascem na Task 3 e a Task 5 os mantém no bloco que reescreve; os estados `notices`/`preferences` com `consentVersion: string | null` (Tasks 3 e 4) são os que a Task 5 usa em `enter()` e no `onUnauthenticated`; `AppRoute` é `"avisos" | "preferencias"` em `route.ts` e no `Flow`.
- **Review Focus:** os cinco itens têm teste na task dona (Tasks 2, 3, 4 e 5).
