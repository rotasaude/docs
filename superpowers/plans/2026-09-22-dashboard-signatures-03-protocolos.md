# Dashboard — assinaturas, fatia 3: Protocolos

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** a tela Protocolos do dashboard mostra o estado das assinaturas de cada versão e oferece o ciclo de vida — enviar para revisão, **assinar**, publicar, ativar, aposentar e reverter —, com um indicador de "aguardando sua assinatura" para o revisor.

**Architecture:** a leitura já existe: `/admin/api/protocols` e `/admin/api/protocols/:id` devolvem, por versão, `signatures`, `editors`, `eligibleReviewers` e `revertible`. Esta fatia só consome. As escritas vão para os endpoints da cidade (`ProtocolLifecycleController` e `PublicationsController`), pelo `SensitiveAction` da fatia 1. Uma função pura decide quais ações cabem para o usuário atual e por que uma estaria desabilitada.

**Tech Stack:** Vite + React 18 + React Query 5 + Vitest + @testing-library/react (apps/dashboard). Nenhuma mudança na API.

**Spec:** `docs/superpowers/specs/2026-09-22-dashboard-protocol-signatures-design.md` (commit `26090e9`). Esta fatia implementa §2 (decisão 4), §3, §5.2 "Protocolos" e §9 fatia 3. As regras de domínio vêm de `docs/superpowers/specs/2026-09-18-protocol-signatures-design.md` e do ADR 0016.

## Global Constraints

- Commits: Conventional Commits **em inglês**, tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Git: `/opt/homebrew/bin/git`. Branch `feat/dashboard-protocol-signatures` em `apps/dashboard` (é o único repo de código desta fatia). Nunca em `main`, nunca push, nunca merge.
- Staging explícito: nunca `git add -A` nem `git add .` (há um `.superpowers/` fora do git).
- **Nada muda na API.** Se algo parecer faltar no servidor, pare e relate em vez de mexer em `apps/api`.
- Nenhuma tela lê status HTTP: erro vem de `describeActionError` (`src/lib/actionErrors.ts`). Mensagens ao usuário em português.
- O `SensitiveAction` (`src/components/SensitiveAction.tsx`) é quem cuida do step-up, do código TOTP (limpo a cada tentativa) e da tradução de recusa. A tela não repete nada disso.
- **Quem decide é a API.** O que o front calcula é previsão: a ação desabilitada evita gastar um código à toa, e a recusa da API aparece inline.
- Testes: `npm run typecheck && npm test && npm run build` verdes antes de cada commit.
- Nunca imprima código TOTP, chave ou código de recuperação — nem em relatório.
- Nunca rode `start.sh`.

## Contrato já existente (leitura)

`GET /admin/api/protocols` → `{ data: { list: [row] }, as_of }`, e `GET /admin/api/protocols/:id` → `{ data: { id, name, versions: [version], events: [...] }, as_of }`. Cada `row`/`version` carrega, além do que já vinha:

```json
{
  "signatures": {
    "publication": { "signers": [ { "id": "uuid", "email": "ana@cidade.gov.br" } ], "missing": 1 },
    "activation":  { "signers": [], "missing": 2 }
  },
  "eligibleReviewers": 2,
  "editors": [ { "kind": "user", "id": "uuid", "email": "autor@cidade.gov.br" },
               { "kind": "maintainer", "id": "uuid", "email": null } ],
  "revertible": false
}
```

`row` tem `name` e `version` (string); `version` (do detalhe) tem `version` (string) e `status`. `signers`/`editors` trazem e-mail de staff, nunca dado de cidadão.

## Contrato já existente (escrita, API da cidade)

| ação | requisição | step-up |
|---|---|---|
| enviar para revisão | `POST /protocols/:version/submit` `{ name }` | não |
| assinar | `POST /protocols/:version/signatures` `{ name, purpose }` (`publication` ou `activation`) | sim |
| publicar | `POST /protocols/:version/publish` `{ name }` | sim |
| ativar | `POST /protocols/:version/activate` `{ name }` | sim |
| aposentar | `POST /protocols/:version/retire` `{ name }` | sim |
| reverter | `POST /protocols/revert` `{ name, reason }` | sim |

Sucesso: `200 { ok: true, protocol: { name, version, status } }` (assinar traz também `signature`). Recusa de domínio: `422 { error, message }`; papel insuficiente: `403`; step-up vencido: `401 { error: "mfa_required" }`. O proxy de dev já repassa `/protocols`.

## File Structure

- Modify: `src/lib/types.ts` — `ProtocolRow` e as versões do detalhe ganham o estado de assinatura.
- Modify: `src/lib/api.ts` — as seis escritas de ciclo de vida.
- Modify: `src/lib/api.test.ts`.
- Create: `src/lib/protocolLifecycle.ts` + `src/lib/protocolLifecycle.test.ts` — a regra pura.
- Modify: `src/modules/Protocols.tsx` — colunas, KPI, filtro e ações.
- Create: `src/modules/Protocols.test.tsx`.

---

### Task 1: tipos e cliente das escritas

**Files:**
- Modify: `apps/dashboard/src/lib/types.ts`, `apps/dashboard/src/lib/api.ts`
- Test: `apps/dashboard/src/lib/api.test.ts`

**Interfaces:**
- Consumes: `jsonFetch` (`src/lib/api.ts`), que já manda `credentials: "include"` e põe `Content-Type: application/json` quando há corpo.
- Produces:
  ```ts
  // types.ts
  export interface SignatureBlock { signers: Array<{ id: string; email: string | null }>; missing: number }
  export interface ProtocolEditor { kind: string; id: string; email: string | null }
  export interface SignatureState {
    signatures: { publication: SignatureBlock; activation: SignatureBlock };
    eligibleReviewers: number;
    editors: ProtocolEditor[];
    revertible: boolean;
  }
  // ProtocolRow e ProtocolDetailData["versions"][number] passam a estender SignatureState
  // api.ts
  export type SignaturePurpose = "publication" | "activation";
  export function submitProtocol(name: string, version: string): Promise<void>;
  export function signProtocol(name: string, version: string, purpose: SignaturePurpose): Promise<void>;
  export function publishProtocolVersion(name: string, version: string): Promise<void>;
  export function activateProtocol(name: string, version: string): Promise<void>;
  export function retireProtocol(name: string, version: string): Promise<void>;
  export function revertProtocol(name: string, reason: string): Promise<void>;
  ```
  `publishProtocolVersion` tem esse nome porque `src/lib/api.ts` já exporta `previewProtocol`/`gateProtocol`/`saveProtocolDraft` do editor; confira se algum nome colide antes de criar e, se colidir, relate em vez de renomear o que já existe.

- [ ] **Step 1: Branch**

```bash
cd apps/dashboard && /opt/homebrew/bin/git checkout -b feat/dashboard-protocol-signatures
```

- [ ] **Step 2: Escrever os testes que falham**

Acrescente a `src/lib/api.test.ts` (o arquivo já tem `mockFetch`; importe as funções novas):

```ts
describe("ciclo de vida de protocolo", () => {
  function lastCall() {
    const fetchMock = globalThis.fetch as unknown as ReturnType<typeof vi.fn>;
    const [ url, init ] = fetchMock.mock.calls.at(-1) as [ string, RequestInit ];
    return { url, init, body: init.body ? JSON.parse(init.body as string) : undefined };
  }

  it("submitProtocol manda o nome na rota da versão", async () => {
    mockFetch(200, { ok: true });
    await submitProtocol("dengue", "2");
    const { url, init, body } = lastCall();
    expect(url).toBe("/protocols/2/submit");
    expect(init.method).toBe("POST");
    expect(body).toEqual({ name: "dengue" });
  });

  it("signProtocol manda a finalidade", async () => {
    mockFetch(200, { ok: true });
    await signProtocol("dengue", "2", "activation");
    expect(lastCall().url).toBe("/protocols/2/signatures");
    expect(lastCall().body).toEqual({ name: "dengue", purpose: "activation" });
  });

  it("publicar, ativar e aposentar usam a rota da versão", async () => {
    for (const [ fn, path ] of [
      [ publishProtocolVersion, "publish" ], [ activateProtocol, "activate" ], [ retireProtocol, "retire" ]
    ] as const) {
      mockFetch(200, { ok: true });
      await fn("dengue", "3");
      expect(lastCall().url).toBe(`/protocols/3/${path}`);
      expect(lastCall().body).toEqual({ name: "dengue" });
    }
  });

  it("revertProtocol manda nome e motivo, sem versão na rota", async () => {
    mockFetch(200, { ok: true });
    await revertProtocol("dengue", "regra errada em produção");
    expect(lastCall().url).toBe("/protocols/revert");
    expect(lastCall().body).toEqual({ name: "dengue", reason: "regra errada em produção" });
  });

  it("o nome e a versão vão codificados na URL", async () => {
    mockFetch(200, { ok: true });
    await submitProtocol("a/b", "1 2");
    expect(lastCall().url).toBe("/protocols/1%202/submit");
  });

  it("recusa da API vira ApiError", async () => {
    mockFetch(422, { error: "invalid_state", message: "só in_review pode ser publicado" });
    await expect(publishProtocolVersion("dengue", "1")).rejects.toBeInstanceOf(ApiError);
  });
});
```

- [ ] **Step 3: Rodar e ver falhar**

Run: `npx vitest run src/lib/api.test.ts`
Esperado: FAIL — as funções não existem.

- [ ] **Step 4: Implementar os tipos**

Em `src/lib/types.ts`, acrescente os tipos novos e faça `ProtocolRow` e as versões do detalhe carregarem o estado de assinatura:

```ts
// Estado de assinatura de uma versão (spec de assinaturas §5/§6). Vem das três
// tabelas append-only, nunca de domain_events — o mesmo bloco aparece na lista
// e em cada versão do detalhe.
export interface SignatureBlock {
  signers: Array<{ id: string; email: string | null }>;
  missing: number;
}
export interface ProtocolEditor { kind: string; id: string; email: string | null }
export interface SignatureState {
  signatures: { publication: SignatureBlock; activation: SignatureBlock };
  eligibleReviewers: number;
  editors: ProtocolEditor[];
  revertible: boolean;
}
```

`ProtocolRow extends SignatureState` e o item de `ProtocolDetailData["versions"]` também. `createdBy`, `publishedBy` e `fourEyes` **continuam nos tipos** (a API ainda os manda), mas a tela deixa de usá-los na Task 3.

- [ ] **Step 5: Implementar o cliente**

Em `src/lib/api.ts`, depois das funções de membership:

```ts
const PROTOCOLS_BASE = import.meta.env.VITE_PROTOCOLS_BASE || "/protocols";

export type SignaturePurpose = "publication" | "activation";

// Escritas do ciclo de vida na API da cidade (ProtocolLifecycleController e
// PublicationsController). Tudo menos `submit` exige step-up — quem cuida
// disso é o SensitiveAction, não estas funções.
function versionPath(version: string, action: string): string {
  return `${PROTOCOLS_BASE}/${encodeURIComponent(version)}/${action}`;
}

export async function submitProtocol(name: string, version: string): Promise<void> {
  await jsonFetch<unknown>(versionPath(version, "submit"), { method: "POST", body: JSON.stringify({ name }) });
}

export async function signProtocol(name: string, version: string, purpose: SignaturePurpose): Promise<void> {
  await jsonFetch<unknown>(versionPath(version, "signatures"), {
    method: "POST", body: JSON.stringify({ name, purpose })
  });
}

export async function publishProtocolVersion(name: string, version: string): Promise<void> {
  await jsonFetch<unknown>(versionPath(version, "publish"), { method: "POST", body: JSON.stringify({ name }) });
}

export async function activateProtocol(name: string, version: string): Promise<void> {
  await jsonFetch<unknown>(versionPath(version, "activate"), { method: "POST", body: JSON.stringify({ name }) });
}

export async function retireProtocol(name: string, version: string): Promise<void> {
  await jsonFetch<unknown>(versionPath(version, "retire"), { method: "POST", body: JSON.stringify({ name }) });
}

// Reversão de emergência: a rota não leva versão — a API acha a ativa pelo nome.
export async function revertProtocol(name: string, reason: string): Promise<void> {
  await jsonFetch<unknown>(`${PROTOCOLS_BASE}/revert`, {
    method: "POST", body: JSON.stringify({ name, reason })
  });
}
```

- [ ] **Step 6: Rodar e ver passar**

```bash
npx vitest run src/lib/api.test.ts && npm run typecheck && npm test
```
Esperado: verde. Se o `typecheck` reclamar de algum lugar que monta `ProtocolRow` (mocks de teste antigos, por exemplo), acrescente o estado de assinatura só nesses fixtures.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/lib/types.ts src/lib/api.ts src/lib/api.test.ts
/opt/homebrew/bin/git add -u src   # fixtures ajustadas, se houver
/opt/homebrew/bin/git commit -m "feat: add the protocol lifecycle client and signature types" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: a regra pura do ciclo de vida

**Files:**
- Create: `apps/dashboard/src/lib/protocolLifecycle.ts`
- Test: `apps/dashboard/src/lib/protocolLifecycle.test.ts`

**Interfaces:**
- Consumes: `SignatureState` (Task 1); `SignaturePurpose` (Task 1).
- Produces:
  ```ts
  export const REQUIRED_SIGNATURES = 2;
  export type LifecycleKind = "submit" | "sign" | "publish" | "activate" | "retire" | "revert";
  export interface Viewer { id: string; roles: string[] }
  export interface LifecycleTarget extends SignatureState { version: string; status: string }
  export interface LifecycleAction {
    kind: LifecycleKind;
    label: string;
    purpose?: SignaturePurpose;   // só em "sign"
    stepUp: boolean;
    needsReason: boolean;         // só em "revert"
    disabledReason: string | null;
  }
  export function shortfallMessage(missing: number): string;
  export function purposeForStatus(status: string): SignaturePurpose | null;
  export function actionsFor(target: LifecycleTarget, viewer: Viewer): LifecycleAction[];
  export function awaitingMySignature(target: LifecycleTarget, viewer: Viewer): boolean;
  ```

**Regras (spec do dashboard §3, conferidas nos commands da API):**

| status | ação | papel |
|---|---|---|
| `draft` | Enviar para revisão | `protocol_author` |
| `in_review` | Assinar publicação | `protocol_reviewer` |
| `in_review` | Publicar | `protocol_publisher` |
| `published` | Assinar ativação | `protocol_reviewer` |
| `published` | Ativar | `protocol_publisher` ou `municipal_admin` |
| `draft`, `in_review`, `published` | Aposentar | `protocol_publisher` |
| `active` | Reverter (só com `revertible`) | `protocol_publisher` ou `municipal_admin` |

- A ação só entra na lista se o papel do `viewer` permite. Step-up em tudo menos `submit`; motivo só em `revert`.
- **Assinar** fica desabilitado com "você editou esta versão" quando o `viewer.id` está em `editors` com `kind === "user"`, e com "você já assinou" quando está nos `signers` da finalidade.
- **Publicar/Ativar** desabilitam com "a cidade tem N revisor(es) elegível(is); são necessários 2" quando `eligibleReviewers < 2` (tem precedência), senão com `shortfallMessage(missing)` da finalidade — publicação para publicar, ativação para ativar.
- `awaitingMySignature` é verdadeiro quando existe uma ação `sign` habilitada para esse viewer.

- [ ] **Step 1: Escrever o teste que falha**

`src/lib/protocolLifecycle.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { actionsFor, awaitingMySignature, purposeForStatus, shortfallMessage, type LifecycleTarget, type Viewer } from "./protocolLifecycle";

const ME = "u-me";

function target(overrides: Partial<LifecycleTarget> = {}): LifecycleTarget {
  return {
    version: "1", status: "in_review",
    signatures: { publication: { signers: [], missing: 2 }, activation: { signers: [], missing: 2 } },
    eligibleReviewers: 3, editors: [], revertible: false, ...overrides
  };
}
const viewer = (...roles: string[]): Viewer => ({ id: ME, roles });
const kinds = (t: LifecycleTarget, v: Viewer) => actionsFor(t, v).map((a) => a.kind);
const action = (t: LifecycleTarget, v: Viewer, kind: string) => actionsFor(t, v).find((a) => a.kind === kind)!;

describe("actionsFor", () => {
  it("oferece por status e por papel", () => {
    expect(kinds(target({ status: "draft" }), viewer("protocol_author"))).toEqual([ "submit" ]);
    expect(kinds(target({ status: "draft" }), viewer("protocol_publisher"))).toEqual([ "retire" ]);
    expect(kinds(target({ status: "in_review" }), viewer("protocol_reviewer"))).toEqual([ "sign" ]);
    expect(kinds(target({ status: "in_review" }), viewer("protocol_publisher"))).toEqual([ "publish", "retire" ]);
    expect(kinds(target({ status: "published" }), viewer("protocol_reviewer"))).toEqual([ "sign" ]);
    expect(kinds(target({ status: "published" }), viewer("municipal_admin"))).toEqual([ "activate" ]);
    expect(kinds(target({ status: "published" }), viewer("protocol_publisher"))).toEqual([ "activate", "retire" ]);
    expect(kinds(target({ status: "retired" }), viewer("protocol_publisher"))).toEqual([]);
  });

  it("quem não tem papel nenhum não vê ação", () => {
    expect(kinds(target({ status: "in_review" }), viewer("viewer"))).toEqual([]);
  });

  it("reverter só na versão ativa e reversível, para publisher ou admin", () => {
    expect(kinds(target({ status: "active", revertible: true }), viewer("protocol_publisher"))).toEqual([ "revert" ]);
    expect(kinds(target({ status: "active", revertible: true }), viewer("municipal_admin"))).toEqual([ "revert" ]);
    expect(kinds(target({ status: "active", revertible: false }), viewer("protocol_publisher"))).toEqual([]);
    expect(kinds(target({ status: "active", revertible: true }), viewer("protocol_reviewer"))).toEqual([]);
  });

  it("step-up em tudo menos enviar para revisão; motivo só na reversão", () => {
    expect(action(target({ status: "draft" }), viewer("protocol_author"), "submit")).toMatchObject({ stepUp: false, needsReason: false });
    expect(action(target({ status: "in_review" }), viewer("protocol_reviewer"), "sign")).toMatchObject({ stepUp: true, needsReason: false });
    expect(action(target({ status: "active", revertible: true }), viewer("protocol_publisher"), "revert"))
      .toMatchObject({ stepUp: true, needsReason: true });
  });

  it("assinar leva a finalidade do status", () => {
    expect(action(target({ status: "in_review" }), viewer("protocol_reviewer"), "sign").purpose).toBe("publication");
    expect(action(target({ status: "published" }), viewer("protocol_reviewer"), "sign").purpose).toBe("activation");
    expect(purposeForStatus("draft")).toBeNull();
  });

  it("quem editou a versão não assina", () => {
    const t = target({ status: "in_review", editors: [ { kind: "user", id: ME, email: "eu@cidade.gov.br" } ] });

    expect(action(t, viewer("protocol_reviewer"), "sign").disabledReason).toBe("você editou esta versão");
  });

  it("um mantenedor entre os editores não bloqueia o revisor", () => {
    const t = target({ status: "in_review", editors: [ { kind: "maintainer", id: ME, email: null } ] });

    expect(action(t, viewer("protocol_reviewer"), "sign").disabledReason).toBeNull();
  });

  it("quem já assinou a finalidade não assina de novo", () => {
    const t = target({
      status: "in_review",
      signatures: { publication: { signers: [ { id: ME, email: "eu@cidade.gov.br" } ], missing: 1 }, activation: { signers: [], missing: 2 } }
    });

    expect(action(t, viewer("protocol_reviewer"), "sign").disabledReason).toBe("você já assinou");
  });

  it("publicar e ativar dependem da finalidade certa", () => {
    const t = target({
      status: "in_review",
      signatures: { publication: { signers: [], missing: 1 }, activation: { signers: [], missing: 0 } }
    });
    expect(action(t, viewer("protocol_publisher"), "publish").disabledReason).toBe("falta 1 assinatura");

    const published = target({
      status: "published",
      signatures: { publication: { signers: [], missing: 0 }, activation: { signers: [], missing: 2 } }
    });
    expect(action(published, viewer("protocol_publisher"), "activate").disabledReason).toBe("faltam 2 assinaturas");
  });

  it("revisores insuficientes vencem o faltante", () => {
    const t = target({ status: "in_review", eligibleReviewers: 1 });

    expect(action(t, viewer("protocol_publisher"), "publish").disabledReason)
      .toBe("a cidade tem 1 revisor(es) elegível(is); são necessários 2");
  });

  it("enviar e aposentar nunca dependem de assinatura", () => {
    const t = target({ status: "draft", eligibleReviewers: 0 });

    expect(action(t, viewer("protocol_author"), "submit").disabledReason).toBeNull();
    expect(action(t, viewer("protocol_publisher"), "retire").disabledReason).toBeNull();
  });

  it("publicar com tudo pronto fica habilitado", () => {
    const t = target({
      status: "in_review",
      signatures: { publication: { signers: [], missing: 0 }, activation: { signers: [], missing: 2 } }
    });

    expect(action(t, viewer("protocol_publisher"), "publish").disabledReason).toBeNull();
  });
});

describe("shortfallMessage", () => {
  it("singular e plural, como Protocols::Signatures.shortfall_message", () => {
    expect(shortfallMessage(1)).toBe("falta 1 assinatura");
    expect(shortfallMessage(2)).toBe("faltam 2 assinaturas");
  });
});

describe("awaitingMySignature", () => {
  it("verdadeiro para o revisor que pode assinar agora", () => {
    expect(awaitingMySignature(target({ status: "in_review" }), viewer("protocol_reviewer"))).toBe(true);
    expect(awaitingMySignature(target({ status: "published" }), viewer("protocol_reviewer"))).toBe(true);
  });

  it("falso quando editou, quando já assinou, quando não é revisor e quando o status não pede", () => {
    const edited = target({ status: "in_review", editors: [ { kind: "user", id: ME, email: null } ] });
    const signed = target({
      status: "in_review",
      signatures: { publication: { signers: [ { id: ME, email: null } ], missing: 1 }, activation: { signers: [], missing: 2 } }
    });

    expect(awaitingMySignature(edited, viewer("protocol_reviewer"))).toBe(false);
    expect(awaitingMySignature(signed, viewer("protocol_reviewer"))).toBe(false);
    expect(awaitingMySignature(target({ status: "in_review" }), viewer("protocol_publisher"))).toBe(false);
    expect(awaitingMySignature(target({ status: "draft" }), viewer("protocol_reviewer"))).toBe(false);
  });

  it("o revisor que assinou a publicação ainda pode assinar a ativação", () => {
    const t = target({
      status: "published",
      signatures: { publication: { signers: [ { id: ME, email: null } ], missing: 1 }, activation: { signers: [], missing: 2 } }
    });

    expect(awaitingMySignature(t, viewer("protocol_reviewer"))).toBe(true);
  });
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `npx vitest run src/lib/protocolLifecycle.test.ts`
Esperado: FAIL — o módulo não existe.

- [ ] **Step 3: Implementar**

`src/lib/protocolLifecycle.ts`:

```ts
// Que ações de ciclo de vida uma versão oferece a QUEM está olhando, e por que
// uma delas estaria desabilitada (spec do dashboard §3; ADR 0016).
//
// É previsão, não decisão: os motivos abaixo são os que o estado lido permite
// antecipar, para o revisor não gastar um código de autenticador numa ação que
// a API recusaria. Quem decide é o command, que trava a linha e reconfere.
import type { SignatureBlock, SignatureState } from "./types";
import type { SignaturePurpose } from "./api";

// Protocols::Signatures::REQUIRED na API.
export const REQUIRED_SIGNATURES = 2;

export type LifecycleKind = "submit" | "sign" | "publish" | "activate" | "retire" | "revert";

export interface Viewer { id: string; roles: string[] }
export interface LifecycleTarget extends SignatureState { version: string; status: string }

export interface LifecycleAction {
  kind: LifecycleKind;
  label: string;
  purpose?: SignaturePurpose;
  stepUp: boolean;
  needsReason: boolean;
  disabledReason: string | null;
}

// Mesma frase de Protocols::Signatures.shortfall_message (a da API é mais
// longa: traz a finalidade e a contagem de revisores; aqui fica o núcleo).
export function shortfallMessage(missing: number): string {
  return missing === 1 ? "falta 1 assinatura" : `faltam ${missing} assinaturas`;
}

// A finalidade que o status pede: in_review assina publicação, published
// assina ativação. Qualquer outro status não pede assinatura.
export function purposeForStatus(status: string): SignaturePurpose | null {
  if (status === "in_review") return "publication";
  if (status === "published") return "activation";
  return null;
}

function has(viewer: Viewer, ...roles: string[]): boolean {
  return roles.some((role) => viewer.roles.includes(role));
}

function blockFor(target: LifecycleTarget, purpose: SignaturePurpose): SignatureBlock {
  return target.signatures[purpose];
}

function signatureBlocked(target: LifecycleTarget, purpose: SignaturePurpose): string | null {
  if (target.eligibleReviewers < REQUIRED_SIGNATURES) {
    return `a cidade tem ${target.eligibleReviewers} revisor(es) elegível(is); são necessários ${REQUIRED_SIGNATURES}`;
  }
  const missing = blockFor(target, purpose).missing;
  return missing > 0 ? shortfallMessage(missing) : null;
}

// Só editor `user` bloqueia o revisor: um mantenedor que editou a versão é
// outra pessoa, e o id dele vive em outro banco (ADR 0016).
function iEdited(target: LifecycleTarget, viewer: Viewer): boolean {
  return target.editors.some((e) => e.kind === "user" && e.id === viewer.id);
}

function iSigned(target: LifecycleTarget, viewer: Viewer, purpose: SignaturePurpose): boolean {
  return blockFor(target, purpose).signers.some((s) => s.id === viewer.id);
}

function signAction(target: LifecycleTarget, viewer: Viewer, purpose: SignaturePurpose): LifecycleAction {
  const label = purpose === "publication" ? "Assinar publicação" : "Assinar ativação";
  const disabledReason = iEdited(target, viewer)
    ? "você editou esta versão"
    : iSigned(target, viewer, purpose) ? "você já assinou" : null;

  return { kind: "sign", label, purpose, stepUp: true, needsReason: false, disabledReason };
}

function retireAction(): LifecycleAction {
  return { kind: "retire", label: "Aposentar", stepUp: true, needsReason: false, disabledReason: null };
}

export function actionsFor(target: LifecycleTarget, viewer: Viewer): LifecycleAction[] {
  const actions: LifecycleAction[] = [];
  const purpose = purposeForStatus(target.status);

  if (target.status === "draft" && has(viewer, "protocol_author")) {
    actions.push({ kind: "submit", label: "Enviar para revisão", stepUp: false, needsReason: false, disabledReason: null });
  }

  if (purpose && has(viewer, "protocol_reviewer")) {
    actions.push(signAction(target, viewer, purpose));
  }

  if (target.status === "in_review" && has(viewer, "protocol_publisher")) {
    actions.push({
      kind: "publish", label: "Publicar", stepUp: true, needsReason: false,
      disabledReason: signatureBlocked(target, "publication")
    });
  }

  if (target.status === "published" && has(viewer, "protocol_publisher", "municipal_admin")) {
    actions.push({
      kind: "activate", label: "Ativar", stepUp: true, needsReason: false,
      disabledReason: signatureBlocked(target, "activation")
    });
  }

  // R4: a versão ativa nunca é aposentada — só revertida.
  if ([ "draft", "in_review", "published" ].includes(target.status) && has(viewer, "protocol_publisher")) {
    actions.push(retireAction());
  }

  if (target.status === "active" && target.revertible && has(viewer, "protocol_publisher", "municipal_admin")) {
    actions.push({ kind: "revert", label: "Reverter", stepUp: true, needsReason: true, disabledReason: null });
  }

  return actions;
}

export function awaitingMySignature(target: LifecycleTarget, viewer: Viewer): boolean {
  return actionsFor(target, viewer).some((a) => a.kind === "sign" && a.disabledReason === null);
}
```

A ordem em que as ações saem é a que a tela mostra, e os testes do Step 1 fixam essa ordem — não reordene os blocos sem atualizar o teste.

- [ ] **Step 4: Rodar e ver passar**

```bash
npx vitest run src/lib/protocolLifecycle.test.ts && npm run typecheck
```
Esperado: verde.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/protocolLifecycle.ts src/lib/protocolLifecycle.test.ts
/opt/homebrew/bin/git commit -m "feat: derive protocol lifecycle actions for the viewer" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: a tela Protocolos

**Files:**
- Modify: `apps/dashboard/src/modules/Protocols.tsx`
- Test: `apps/dashboard/src/modules/Protocols.test.tsx`

**Interfaces:**
- Consumes:
  - `useProtocols()` e `useProtocolDetail(id)` (`src/hooks/useProtocols.ts`), que já devolvem o envelope `{ data, as_of }`;
  - `actionsFor`, `awaitingMySignature`, `LifecycleAction`, `LifecycleTarget`, `Viewer` (Task 2);
  - as seis escritas (Task 1);
  - `SensitiveAction` (`src/components/SensitiveAction.tsx`);
  - `useAuth()` para montar o `Viewer`: `{ id: user.id, roles: user.memberships.map((m) => m.role) }`;
  - `useQueryClient()` para invalidar `[ "protocols", ... ]` e `[ "protocol-detail", id ]` — confira as chaves exatas em `src/hooks/useProtocols.ts` e use as mesmas.
- Produces: `Protocols` continua sendo `export function Protocols()` sem props obrigatórias novas. Se precisar navegar para a Segurança (conta sem autenticador), aceite `onNavigate?: (id: ModuleId) => void` e passe do `App.tsx` como as outras telas fazem.

**Comportamento (spec §5.2 "Protocolos"):**
- **KPIs:** "Protocolos & versões" (total) e "Publicados" ficam. O KPI "4-olhos colapsado" **sai**; entra **"Aguardando sua assinatura"**, com a contagem de linhas em que `awaitingMySignature` é verdadeiro. Clicar nele alterna um filtro que mostra só essas linhas.
- **Colunas da lista:** ID, Versão, Status, **Publicação** (`signers.length`/2), **Ativação** (`signers.length`/2), **Revisores** (`eligibleReviewers`), Detalhe. A coluna "4-olhos" sai; Schema e Linter ficam.
- **Painel de detalhe (o drawer que já existe), por versão:**
  - as duas finalidades, com os e-mails de quem assinou e o que falta;
  - os editores: e-mail quando `kind === "user"`, e a palavra "mantenedor" quando `kind === "maintainer"`;
  - `eligibleReviewers`;
  - as ações de `actionsFor`, cada uma como um botão; desabilitada mostra o motivo num `<small>`;
  - a lista de eventos continua como está.
- **Ação clicada:** abre um `SensitiveAction` dentro do drawer, com `title` = o rótulo mais `<nome> v<versão>`, `requiresStepUp` = `action.stepUp`, e `fields` com "Motivo" quando `needsReason`. O `run` chama a função correspondente da Task 1 (`sign` passa `action.purpose!`).
- **Sucesso:** fecha o painel, mostra `role="status"` "<rótulo> concluído: <nome> v<versão>", e invalida a lista e o detalhe.
- **Erro:** o `SensitiveAction` mostra; a tela não traduz nada por conta própria.
- Nada de `createdBy`, `publishedBy` ou `fourEyes` na tela — aquele era o modelo antigo, anterior às assinaturas.

- [ ] **Step 1: Escrever os testes que falham**

`src/modules/Protocols.test.tsx`. Molde: `src/modules/Team.test.tsx` (mock de `../lib/api`, `AuthProvider`, `QueryClientProvider`, `fireEvent`). Como a tela lê pelo `adminFetch`, mock esse também.

```tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return {
    ...real,
    fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), adminFetch: vi.fn(),
    submitProtocol: vi.fn(), signProtocol: vi.fn(), publishProtocolVersion: vi.fn(),
    activateProtocol: vi.fn(), retireProtocol: vi.fn(), revertProtocol: vi.fn()
  };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { AuthProvider } from "../lib/auth";
import { ScopeContext } from "../lib/scope";
import { Protocols } from "./Protocols";

afterEach(cleanup);

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ME = "u-me";

function session(role: string): api.SessionUser {
  return {
    id: ME, email_address: "eu@cidade.gov.br", operator: false,
    memberships: [ { municipality_id: "m1", municipality_name: "Curitiba", municipality_uf: "PR", role } ],
    mfa_enrolled: true, mfa_verified_at: new Date().toISOString()
  };
}

const EMPTY_SIGNATURES = {
  publication: { signers: [], missing: 2 },
  activation: { signers: [], missing: 2 }
};

function row(overrides: Record<string, unknown> = {}) {
  return {
    id: "dengue", name: "dengue", version: "1", status: "in_review",
    createdBy: null, publishedBy: null, fourEyes: null, publishedAt: null, retiredAt: null,
    schema: "ok", linter: "ok", gates: "ok",
    signatures: EMPTY_SIGNATURES, eligibleReviewers: 3, editors: [], revertible: false,
    ...overrides
  };
}

function versionRow(overrides: Record<string, unknown> = {}) {
  return {
    version: "1", status: "in_review", createdBy: null, publishedBy: null, fourEyes: null,
    at: "2026-09-01T00:00:00Z", schema: "ok", linter: "ok", gates: "ok",
    signatures: EMPTY_SIGNATURES, eligibleReviewers: 3, editors: [], revertible: false,
    ...overrides
  };
}

// adminFetch é chamado com "/protocols" (lista) e "/protocols/:id" (detalhe).
function stubReads(rows: unknown[], versions: unknown[] = [ versionRow() ]) {
  mocked(api.adminFetch).mockImplementation((path: string) => {
    if (path === "/protocols") return Promise.resolve({ data: { list: rows }, as_of: "2026-09-22T00:00:00Z" });
    return Promise.resolve({
      data: { id: "dengue", name: "dengue", versions, events: [] }, as_of: "2026-09-22T00:00:00Z"
    });
  });
}

function renderProtocols(role = "protocol_reviewer") {
  mocked(api.fetchCurrentSession).mockResolvedValue(session(role));
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function wrapper({ children }: { children: ReactNode }) {
    return (
      <QueryClientProvider client={client}>
        <AuthProvider>
          <ScopeContext.Provider value={{ period: "7d", municipalityId: "m1", setPeriod: vi.fn() }}>
            {children}
          </ScopeContext.Provider>
        </AuthProvider>
      </QueryClientProvider>
    );
  }
  return render(<Protocols />, { wrapper });
}

describe("Protocols", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.adminFetch, api.stepUpMfa, api.submitProtocol,
                       api.signProtocol, api.publishProtocolVersion, api.activateProtocol,
                       api.retireProtocol, api.revertProtocol ]) {
      mocked(fn).mockReset();
    }
  });

  it("mostra assinaturas por finalidade e os revisores elegíveis, sem 4-olhos", async () => {
    stubReads([ row({ signatures: { publication: { signers: [ { id: "u-a", email: "a@c.gov" } ], missing: 1 }, activation: { signers: [], missing: 2 } } }) ]);
    renderProtocols();

    const table = within(await screen.findByRole("table"));
    expect(table.getByText("1/2")).not.toBeNull();
    expect(table.getByText("0/2")).not.toBeNull();
    expect(screen.queryByText(/4-olhos/)).toBeNull();
  });

  it("conta e filtra 'aguardando sua assinatura'", async () => {
    stubReads([
      row({ id: "dengue", name: "dengue", status: "in_review" }),
      row({ id: "zika", name: "zika", status: "draft" })
    ]);
    renderProtocols("protocol_reviewer");

    const kpi = (await screen.findByText("Aguardando sua assinatura")).closest("div")!;
    // dengue está em revisão e eu sou revisor sem assinatura: 1.
    // A contagem é lida DENTRO do cartão: "1" também aparece na coluna Versão.
    expect(within(kpi).getByText("1")).not.toBeNull();

    fireEvent.click(screen.getByText("Aguardando sua assinatura"));

    await waitFor(() => expect(screen.queryByText("zika")).toBeNull());
    expect(screen.getByText("dengue")).not.toBeNull();
  });

  it("o detalhe mostra quem assinou, quem editou e o mantenedor por extenso", async () => {
    stubReads([ row() ], [ versionRow({
      signatures: { publication: { signers: [ { id: "u-a", email: "ana@c.gov" } ], missing: 1 }, activation: { signers: [], missing: 2 } },
      editors: [ { kind: "user", id: "u-b", email: "autor@c.gov" }, { kind: "maintainer", id: "m-1", email: null } ]
    }) ]);
    renderProtocols();

    fireEvent.click(await screen.findByText("dengue"));

    expect(await screen.findByText("ana@c.gov")).not.toBeNull();
    expect(screen.getByText("autor@c.gov")).not.toBeNull();
    expect(screen.getByText("mantenedor")).not.toBeNull();
  });

  it("revisor assina a publicação com step-up", async () => {
    stubReads([ row() ]);
    mocked(api.signProtocol).mockResolvedValue(undefined);
    renderProtocols("protocol_reviewer");

    fireEvent.click(await screen.findByText("dengue"));
    fireEvent.click(await screen.findByRole("button", { name: "Assinar publicação" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    await waitFor(() => expect(api.signProtocol).toHaveBeenCalledWith("dengue", "1", "publication"));
    expect((await screen.findByRole("status")).textContent).toBe("Assinar publicação concluído: dengue v1");
  });

  it("quem editou a versão vê a assinatura desabilitada com o motivo", async () => {
    stubReads([ row() ], [ versionRow({ editors: [ { kind: "user", id: ME, email: "eu@cidade.gov.br" } ] }) ]);
    renderProtocols("protocol_reviewer");

    fireEvent.click(await screen.findByText("dengue"));

    const button = await screen.findByRole("button", { name: "Assinar publicação" });
    expect((button as HTMLButtonElement).disabled).toBe(true);
    expect(screen.getByText("você editou esta versão")).not.toBeNull();
  });

  it("publisher vê Publicar bloqueado pelo que falta", async () => {
    stubReads([ row() ], [ versionRow({ signatures: { publication: { signers: [], missing: 1 }, activation: { signers: [], missing: 2 } } }) ]);
    renderProtocols("protocol_publisher");

    fireEvent.click(await screen.findByText("dengue"));

    expect((await screen.findByRole("button", { name: "Publicar" }) as HTMLButtonElement).disabled).toBe(true);
    expect(screen.getByText("falta 1 assinatura")).not.toBeNull();
  });

  it("reverter pede motivo e manda nome e motivo", async () => {
    stubReads([ row({ status: "active", revertible: true }) ], [ versionRow({ status: "active", revertible: true }) ]);
    mocked(api.revertProtocol).mockResolvedValue(undefined);
    renderProtocols("protocol_publisher");

    fireEvent.click(await screen.findByText("dengue"));
    fireEvent.click(await screen.findByRole("button", { name: "Reverter" }));
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "regra errada" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    await waitFor(() => expect(api.revertProtocol).toHaveBeenCalledWith("dengue", "regra errada"));
  });

  it("a recusa da API aparece e o painel continua aberto", async () => {
    stubReads([ row() ]);
    mocked(api.signProtocol).mockRejectedValue(new ApiError(422, { error: "invalid_state", message: "só in_review aceita assinatura" }, "422"));
    renderProtocols("protocol_reviewer");

    fireEvent.click(await screen.findByText("dengue"));
    fireEvent.click(await screen.findByRole("button", { name: "Assinar publicação" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    expect(await screen.findByText("só in_review aceita assinatura")).not.toBeNull();
    expect(screen.getByRole("button", { name: "Confirmar" })).not.toBeNull();
  });

  it("sucesso recarrega lista e detalhe", async () => {
    stubReads([ row() ]);
    mocked(api.signProtocol).mockResolvedValue(undefined);
    renderProtocols("protocol_reviewer");

    fireEvent.click(await screen.findByText("dengue"));
    const before = mocked(api.adminFetch).mock.calls.length;
    fireEvent.click(await screen.findByRole("button", { name: "Assinar publicação" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    await waitFor(() => expect(mocked(api.adminFetch).mock.calls.length).toBeGreaterThan(before + 1));
  });

  it("quem só tem viewer não vê ação nenhuma", async () => {
    stubReads([ row() ]);
    renderProtocols("viewer");

    fireEvent.click(await screen.findByText("dengue"));

    await screen.findByText("Versões");
    expect(screen.queryByRole("button", { name: "Assinar publicação" })).toBeNull();
    expect(screen.queryByRole("button", { name: "Publicar" })).toBeNull();
  });
});
```

Confira os nomes exatos que a tela usa hoje (`Versões`, `Eventos`, o texto do link "ver →") antes de afirmar textos; ajuste o teste ao que existe, sem mudar o que ele prova. Se `ScopeContext` exigir outro formato de valor, copie do `src/lib/scope.ts`.

- [ ] **Step 2: Rodar e ver falhar**

Run: `npx vitest run src/modules/Protocols.test.tsx`
Esperado: FAIL — a tela ainda é a antiga (4-olhos, sem ações).

- [ ] **Step 3: Implementar a lista**

Em `src/modules/Protocols.tsx`:

- monte o viewer e o filtro:

```tsx
  const auth = useAuth();
  const viewer: Viewer = useMemo(
    () => ({ id: auth.user?.id ?? "", roles: (auth.user?.memberships ?? []).map((m) => m.role) }),
    [ auth.user ]
  );
  const [ onlyMine, setOnlyMine ] = useState(false);
```

- troque o KPI de 4-olhos:

```tsx
  const awaiting = list.filter((r) => awaitingMySignature(r, viewer)).length;
  const shown = onlyMine ? list.filter((r) => awaitingMySignature(r, viewer)) : list;
```

```tsx
        <StatTile label="Protocolos & versões" value={list.length} source="live" />
        <StatTile label="Publicados" value={published} tone="ok" source="live" />
        <button type="button" onClick={() => setOnlyMine((v) => !v)} style={{ all: "unset", cursor: "pointer" }}>
          <StatTile label="Aguardando sua assinatura" value={awaiting}
                    tone={awaiting > 0 ? "warn" : "ok"} source="live" />
        </button>
```

  `all: "unset"` mantém o cartão com a aparência de sempre; o `StatTile` não conhece clique. Se o `StatTile` já aceitar `onClick`, use a prop dele em vez do botão.

- troque as colunas "4-olhos" por assinaturas e revisores, e use `shown` como `rows`:

```tsx
            { label: "Publicação", w: "1fr", render: (r) => <span className="mono">{r.signatures.publication.signers.length}/2</span> },
            { label: "Ativação", w: "1fr", render: (r) => <span className="mono">{r.signatures.activation.signers.length}/2</span> },
            { label: "Revisores", w: "1fr", render: (r) => <span className="mono">{r.eligibleReviewers}</span> },
```

- [ ] **Step 4: Implementar o detalhe e as ações**

Ainda em `Protocols.tsx`, no `DetailDrawer`:

- receba `viewer` e o nome do protocolo por props;
- guarde `pending: { version: string; action: LifecycleAction } | null` e `done: string | null`;
- para cada versão, acrescente um bloco com as assinaturas, os editores e as ações:

```tsx
function VersionActions({
  name, version, viewer, onPick
}: { name: string; version: LifecycleTarget; viewer: Viewer; onPick(action: LifecycleAction): void }) {
  const actions = actionsFor(version, viewer);
  if (actions.length === 0) return null;

  return (
    <div style={{ display: "flex", gap: 8, flexWrap: "wrap" }}>
      {actions.map((action) => (
        <div key={`${action.kind}-${action.purpose ?? ""}`} style={{ display: "flex", flexDirection: "column", gap: 2 }}>
          <button type="button" style={buttonStyle} disabled={action.disabledReason !== null} onClick={() => onPick(action)}>
            {action.label}
          </button>
          {action.disabledReason && (
            <small style={{ fontSize: 11, color: "var(--ink3)" }}>{action.disabledReason}</small>
          )}
        </div>
      ))}
    </div>
  );
}
```

- e a execução da ação escolhida:

```tsx
async function runAction(name: string, version: string, action: LifecycleAction, values: Record<string, string>) {
  switch (action.kind) {
    case "submit":   return submitProtocol(name, version);
    case "sign":     return signProtocol(name, version, action.purpose!);
    case "publish":  return publishProtocolVersion(name, version);
    case "activate": return activateProtocol(name, version);
    case "retire":   return retireProtocol(name, version);
    case "revert":   return revertProtocol(name, values.reason ?? "");
  }
}
```

- o painel, dentro do drawer:

```tsx
      {pending && (
        <SensitiveAction
          title={`${pending.action.label} ${name} v${pending.version}`}
          requiresStepUp={pending.action.stepUp}
          fields={pending.action.needsReason ? [ { name: "reason", label: "Motivo", required: true } ] : []}
          run={(values) => runAction(name, pending.version, pending.action, values)}
          onDone={() => {
            setDone(`${pending.action.label} concluído: ${name} v${pending.version}`);
            setPending(null);
            void queryClient.invalidateQueries({ queryKey: [ "protocols" ] });
            void queryClient.invalidateQueries({ queryKey: [ "protocol-detail", id ] });
          }}
          onCancel={() => setPending(null)}
        />
      )}
```

O `buttonStyle` vem de `src/components/formStyles.ts` (criado na fatia 1) — acrescente o import; não recopie literais de estilo.

Confira as chaves de invalidação em `src/hooks/useProtocols.ts` (a lista usa `[ "protocols", scope.municipalityId ]`) e use a mesma forma — `invalidateQueries` com prefixo casa as duas.

- [ ] **Step 5: Rodar e ver passar**

```bash
npx vitest run src/modules/Protocols.test.tsx && npm run typecheck && npm test && npm run build
```
Esperado: verde. Se algum teste antigo afirmava "4-olhos", ele muda de sentido: atualize, sem afrouxar.

- [ ] **Step 6: Verificação no navegador** (stack de dev de pé)

Este é o primeiro momento em que o ciclo inteiro roda contra a API real. Na raiz do monorepo: `docker compose up -d api worker dashboard`.

1. Prepare a cidade: na tela Equipe (fatia 2), garanta **duas** pessoas revisoras que **não** sejam a autora da versão. As contas de dev estão nos READMEs; não invente. Cada uma precisa de autenticador cadastrado (página Segurança).
2. Com a conta de autor, envie uma versão `draft` para revisão.
3. Com cada revisora, assine a publicação. Confira que o KPI "Aguardando sua assinatura" conta e filtra, e que a segunda assinatura zera o que falta.
4. Com a conta de publisher, publique. Confira que antes das duas assinaturas o botão estava desabilitado com o motivo.
5. Assine a ativação com as duas revisoras e ative.
6. Confira que "Aposentar" não aparece na versão ativa e que "Reverter" aparece quando há versão anterior.

Registre o que foi visto, sem nenhum código de autenticador. Se faltar conta de dev para algum papel, relate em vez de criar credencial.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/modules/Protocols.tsx src/modules/Protocols.test.tsx
/opt/homebrew/bin/git commit -m "feat: run the protocol lifecycle from the protocols screen" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Fora desta fatia

- O editor de protocolo (`ProtocolEditor`), que segue como está.
- Assinatura com certificado digital (ICP-Brasil) — o spec de assinaturas já a deixa fora.
- `createdBy`/`publishedBy`/`fourEyes` na API: continuam sendo devolvidos e deixam de ser usados; removê-los é outro trabalho.
- E2E com Playwright no dashboard.
- Aviso por e-mail quando alguém assina ou publica (pendência de go-live).
