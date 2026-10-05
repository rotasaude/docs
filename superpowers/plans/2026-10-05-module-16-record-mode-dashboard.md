# Módulo 16 — Modo de prontuário e exportação (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Telas do módulo 16 no painel da cidade. A sessão passa a trazer `features` (interruptores ligados) e o menu esconde o que está desligado. O `municipal_admin` ganha **Integrações** (modo, PEC e IBGE só como estado; credenciais `ledi` e `cadsus` cadastradas e trocadas com step-up; teste de conexão; interruptores com o que falta em linguagem simples) e **CNES** (retrato, propostas confirmadas em lote com step-up, divergências, dados mascarados). `municipal_admin` e `analyst` ganham **Produção e-SUS** (competência, prazo, alertas, contagens, motivos de recusa, fichas paginadas; reenviar ficha recusada só o admin, com step-up). O balcão de validação presencial ganha **Consultar CADSUS** quando `cadsus_lookup` está ligado.

**Architecture:** O cliente HTTP novo vai para `src/lib/api.ts`, com os tipos copiados do arquivo de contratos. As regras ficam fora do React, em funções puras: `src/lib/features.ts` (sessão e `403 feature_disabled`), `src/lib/integrations.ts`, `src/lib/cnes.ts`, `src/lib/competence.ts` e `src/lib/production.ts`. Cada tela é um módulo novo (`Integrations.tsx`, `Cnes.tsx`, `Production.tsx`) num grupo novo do menu, "e-SUS". O step-up é sempre o `SensitiveAction`, que já existe; ele só ganha o campo do tipo senha. O CADSUS do balcão é um componente (`attendance/CadsusCheck.tsx`) encaixado no `Counter` de `Attendance.tsx`.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("../lib/api")` e `vi.stubGlobal("fetch")`), Vite 5 (proxy de dev).

**Spec:** `docs/superpowers/specs/2026-10-05-module-16-record-mode-and-export-design.md` (§3.1, §3.3, §5, §6.5, §7 e §9 "Front" são deste plano), `docs/adr/0028.md` e o arquivo de contratos `docs/superpowers/plans/2026-10-05-module-16-record-mode-contracts.md` (§1 e §5 são deste plano). Os tipos da Task 2 copiam o contrato. O merge deste plano vem **depois** do merge do `api-foundation` (Integrações, CNES, CADSUS) e do `api-exporter` (Produção), contratos §7.

## Global Constraints

- Sessão (contratos §1, `session-v1.1.0`): `session_user.features` é opcional, array de string com as chaves **ligadas** (não necessariamente utilizáveis). Ausente, nulo ou fora de formato = `[]`. Chave desconhecida é ignorada.
- Recusa de funcionalidade desligada: `403 { "error": "feature_disabled", "feature": "<key>" }`. Nunca é traduzida como "seu papel não permite".
- Chaves e pré-requisitos (contratos §2): `ledi_export` (`record_mode_off`, `pec_url_missing`, `ibge_code_missing`, `credential_missing:ledi`, `credential_unauthorized:ledi`); `cadsus_lookup` (`credential_missing:cadsus`, `credential_unauthorized:cadsus`). Código de `missing` desconhecido aparece como "pendência não reconhecida: <código>", nunca some.
- Rotas usadas (sessão municipal, banco da cidade):
  - `GET /integrations`, `PUT /integrations/credentials/:kind` (step-up, `{ username, password }`), `POST /integrations/credentials/:kind/check` — só `municipal_admin`;
  - `GET /cnes`, `POST /cnes/apply` (step-up, `{ proposal_ids }`) — só `municipal_admin`;
  - `GET /production?competence=AAAAMM&page=N`, `POST /production/fichas/:id/resend` (step-up) — leitura `municipal_admin` e `analyst`, reenvio só `municipal_admin`; exige `ledi_export` ligado;
  - `POST /attendance/cadsus_lookup` `{ cpf, code }` e `POST /attendance/verifications` com `cadsus_confirmed` — `citizen_verifier`.
- Três entradas novas no proxy do Vite: `/integrations`, `/cnes`, `/production`. `/attendance` já existe.
- Step-up: 401 `{ "error": "mfa_required" }`, que o `SensitiveAction` já trata. Toda escrita deste plano (credencial, CNES, reenvio) passa por ele com `requiresStepUp`; teste de conexão e consulta ao CADSUS não pedem step-up (contrato não pede).
- Segredo: a senha da credencial só vai no corpo do `PUT`, **como digitada** (sem `trim`), em `<input type="password">`; nunca volta da API, nunca aparece na tela, nunca vai para URL, log ou cache do TanStack Query.
- CPF e CNS chegam **mascarados** da API (`cpf_masked`, `cns_masked`) e aparecem como vieram. O dashboard nunca recebe nem desmascara CPF/CNS do CNES ou do CADSUS.
- Nenhum casamento do CNES é aplicado sem a confirmação explícita (seleção + step-up). Proposta pulada por `stale` é dita ao usuário.
- Datas sem hora (`deadline_on`) são formatadas por texto (`fmtDay`), nunca por `new Date(...)`. Competência `AAAAMM` é calculada por texto a partir de `todayInCity()` (fuso da cidade).
- Testes que dependem de "hoje" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(...)` em `beforeEach`, e `vi.useRealTimers()` em `afterEach`. Funções puras recebem `today` como argumento.
- Interface tem ciclo próprio: use os componentes e estilos existentes (`PageHeader`, `Panel`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `KpiGrid`, `StatTile`, `SensitiveAction`, `formStyles`), sem redesign.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

- Crie o worktree a partir de `origin/main` e ligue o `node_modules` do checkout principal. `/.claude/` já está no `.gitignore` do dashboard.

  ```bash
  cd apps/dashboard
  /opt/homebrew/bin/git fetch origin
  /opt/homebrew/bin/git worktree add .claude/mod16 -b feat/mod-16-record-mode origin/main
  ln -s ../../node_modules .claude/mod16/node_modules
  ```

- Testes (script `test` = `vitest run`):

  ```bash
  cd apps/dashboard/.claude/mod16 && npx vitest run <arquivos>
  ```

- Checagem de tipos antes de cada commit (script `typecheck` = `tsc --noEmit`; o `tsconfig` tem `noUnusedLocals` e `noUnusedParameters`):

  ```bash
  cd apps/dashboard/.claude/mod16 && npx tsc --noEmit
  ```

- Base do ambiente de teste:
  - o `vitest.config.ts` já fixa `TZ=America/Sao_Paulo`;
  - não há `setupFiles` nem jest-dom (use `toBeTruthy`, `not.toBeNull()` e `toBeNull()`);
  - `globals: false`, então todo teste de componente chama `afterEach(cleanup)`;
  - `sessionWith(roles, overrides)` e `renderWithProviders(ui)` já existem em `src/test/campaignFixtures.tsx` (sessão com janela de step-up aberta; `{ mfa_verified_at: null }` força o código).
- O `SensitiveAction` só mostra o botão de confirmar depois que o `AuthProvider` carregou a sessão. Nos testes, procure o botão de confirmar com `findByRole`, nunca com `getByRole`.
- O api do módulo 16 roda em dev na porta **3033**; a prova no navegador aponta o proxy para ela (Task 11).

### Nota de integração com o módulo 15

O módulo 15 (`feat/mod-15-triage-catalog`) mexe nos mesmos trechos: `verifyCitizen` e o fim de `src/lib/api.ts`, o `MESSAGES` de `src/lib/attendance.ts` (última linha, `invalid_neighborhood`), o `Counter` de `src/modules/Attendance.tsx`, `src/modules/Attendance.test.tsx`, `vite.config.ts` e a lista do proxy no `README.md`. Este plano parte de `origin/main` e não depende dele. Se o 15 entrar em `main` antes, rebase e resolva assim:
- assinatura final: `verifyCitizen(cpf, code, profile: VerifiedProfile, extra: VerifyExtra = {})`, corpo `{ cpf, code, document_checked: true, ...profile, ...extra }`;
- no `Counter`, a chamada passa sempre o perfil e só passa `extra` com `cadsusOn`; os testes do CADSUS (Task 9) ganham o perfil como terceiro argumento esperado e preenchem data de nascimento e sexo antes de validar, como os testes do 15;
- `MESSAGES` fica com as entradas dos dois módulos; o proxy fica com `/triage_catalog` e as três deste plano.

## Review Focus

1. **Funcionalidade desligada no meio do uso.** A sessão ainda diz `ledi_export` ligado, mas o mantenedor desligou e a API responde `403 feature_disabled`. A tela mostra "desligado nesta cidade", relê a sessão **uma vez** (o menu esconde a tela) e nunca diz "seu papel não permite", nem entra em laço de releitura. Testes:
   - Task 1, "403 feature_disabled diz que a funcionalidade foi desligada, não o papel";
   - Task 8, "403 feature_disabled com a sessão ainda dizendo ligada: mostra desligada e relê a sessão uma vez só".
2. **Senha da credencial.** Digitada num campo de senha, enviada como digitada (espaços no começo e no fim preservados), e ausente da tela depois de salvar. Testes:
   - Task 2, "a senha vai como digitada, sem trim, e só no corpo";
   - Task 4, "cadastrar: campo de senha mascarado, senha como digitada, formulário fecha e a senha não fica na tela".
3. **Seleção do CNES obsoleta.** Proposta que mudou entre a leitura e a confirmação é pulada pela API (`skipped: stale`) e isso é dito; id que sumiu da lista relida sai da seleção e nunca vai num lote seguinte. Testes:
   - Task 5, "pruneSelection tira o que sumiu" e "applySummary diz as puladas";
   - Task 6, "proposta que mudou é pulada e dita; a seleção volta a zero".
4. **Prazo e competência nas bordas.** `deadline_on` sem hora não volta um dia no fuso de São Paulo; prazo de hoje (`0`) e vencido (`< 0`); às 22h30 de 31/10 na cidade (já 01/11 em UTC) a competência corrente ainda é 10/2026. Testes:
   - Task 5, "competência corrente pelo dia da cidade" (relógio fixo);
   - Task 7, "prazo: dias úteis, hoje e vencido, sem deslocar a data";
   - Task 8, "competência no fuso da cidade: às 22h30 de 31/10 a primeira opção é 10/2026".
5. **CADSUS indisponível ou divergente no balcão.** Com `503 cadsus_unavailable` o atendente segue pelo documento e a validação manda `cadsus_confirmed: false`; com divergência de nascimento ou sexo, aparece o aviso e a confirmação **não** vem marcada; nova busca limpa a confirmação anterior. Testes:
   - Task 9, "divergência: aviso, e a confirmação não vem marcada";
   - Task 9, "CADSUS fora do ar: o balcão segue pelo documento com cadsus_confirmed: false" e "nova busca limpa a confirmação anterior".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `src/lib/api.ts` (`SessionUser`), `src/lib/features.ts`, `src/lib/actionErrors.ts` | `features` da sessão, `hasFeature`, `featureDisabledKey`, rótulos; `feature_disabled` nas ações | 1 |
| `src/lib/api.ts` (fim e `verifyCitizen`), `vite.config.ts`, `README.md` | tipos do contrato §5; cliente de Integrações, CNES, Produção e CADSUS; proxy | 2 |
| `src/lib/integrations.ts`, `src/test/recordModeFixtures.ts` | rótulos de modo/credencial/teste, "o que falta" em frase, estado da funcionalidade, erros | 3 |
| `src/components/SensitiveAction.tsx`, `src/modules/Integrations.tsx` | campo de senha no step-up; tela Integrações | 4 |
| `src/lib/competence.ts`, `src/lib/cnes.ts`, `src/test/recordModeFixtures.ts` | competência AAAAMM; rótulos do CNES, lados mascarados, seleção, resumo da aplicação | 5 |
| `src/modules/Cnes.tsx` | tela CNES com confirmação em lote | 6 |
| `src/lib/production.ts`, `src/test/recordModeFixtures.ts` | situação da ficha, prazo, alerta, paginação, papéis, erros | 7 |
| `src/modules/Production.tsx` | tela Produção e-SUS, com reenvio | 8 |
| `src/lib/attendance.ts`, `src/modules/attendance/CadsusCheck.tsx`, `src/modules/Attendance.tsx` | Consultar CADSUS no balcão e `cadsus_confirmed` | 9 |
| `src/shell/modules.ts`, `src/App.tsx` | grupo "e-SUS" no menu, por papel e por interruptor; rotas | 10 |
| — | suíte, build, revisão e prova no navegador | 11 |

---

### Task 1: `features` na sessão e recusa de funcionalidade desligada

**Files:**
- Modify: `src/lib/api.ts` (interface `SessionUser`), `src/lib/actionErrors.ts`
- Create: `src/lib/features.ts`
- Test: `src/lib/features.test.ts`, `src/lib/actionErrors.test.ts`

**Interfaces:**
- Consumes: `ApiError` (`src/lib/api.ts`).
- Produces (em `src/lib/features.ts`):
  - `type FeatureKey = "ledi_export" | "cadsus_lookup"`;
  - `sessionFeatures(user: { features?: unknown } | null | undefined): string[]`;
  - `hasFeature(user: { features?: unknown } | null | undefined, key: FeatureKey): boolean`;
  - `featureDisabledKey(err: unknown): string | null` — a chave de um `403 feature_disabled` (`""` se a API não mandou `feature`), `null` para qualquer outro erro;
  - `featureLabel(key: string): string`;
  - `FEATURE_DISABLED_MESSAGE = "esta funcionalidade está desligada para a cidade"`.
- Produces (em `src/lib/api.ts`): `SessionUser.features?: string[]`.
- `describeActionError` devolve `{ kind: "rejected", code: "feature_disabled", message: FEATURE_DISABLED_MESSAGE }` para `403 feature_disabled`.

- [ ] **Step 1: Escreva os testes**

```ts
// src/lib/features.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import { featureDisabledKey, featureLabel, hasFeature, sessionFeatures } from "./features";

describe("features da sessão (contratos §1)", () => {
  it("ausente, nulo ou fora de formato vira []", () => {
    expect(sessionFeatures(null)).toEqual([]);
    expect(sessionFeatures(undefined)).toEqual([]);
    expect(sessionFeatures({})).toEqual([]);
    expect(sessionFeatures({ features: null })).toEqual([]);
    expect(sessionFeatures({ features: "ledi_export" })).toEqual([]);
  });

  it("mantém só strings e não quebra com chave desconhecida", () => {
    expect(sessionFeatures({ features: [ "ledi_export", 3, "rnds_sync" ] })).toEqual([ "ledi_export", "rnds_sync" ]);
    expect(hasFeature({ features: [ "rnds_sync" ] }, "ledi_export")).toBe(false);
    expect(hasFeature({ features: [ "cadsus_lookup" ] }, "cadsus_lookup")).toBe(true);
    expect(hasFeature(null, "cadsus_lookup")).toBe(false);
  });

  it("featureDisabledKey só reconhece 403 feature_disabled", () => {
    expect(featureDisabledKey(new ApiError(403, { error: "feature_disabled", feature: "ledi_export" }, "x"))).toBe("ledi_export");
    expect(featureDisabledKey(new ApiError(403, { error: "feature_disabled" }, "x"))).toBe("");
    expect(featureDisabledKey(new ApiError(403, { error: "missing_role" }, "x"))).toBeNull();
    expect(featureDisabledKey(new ApiError(403, "", "x"))).toBeNull();
    expect(featureDisabledKey(new ApiError(404, { error: "feature_disabled" }, "x"))).toBeNull();
    expect(featureDisabledKey(new Error("x"))).toBeNull();
  });

  it("rótulo conhecido em português; desconhecido sai como a chave", () => {
    expect(featureLabel("ledi_export")).toBe("Envio da produção ao e-SUS (LEDI)");
    expect(featureLabel("cadsus_lookup")).toBe("Consulta ao CADSUS na validação presencial");
    expect(featureLabel("rnds_sync")).toBe("rnds_sync");
  });
});
```

Em `src/lib/actionErrors.test.ts`, dentro do `describe("describeActionError", ...)`, logo depois do teste `it("403 sem corpo", ...)`, acrescente:

```ts
  it("403 feature_disabled diz que a funcionalidade foi desligada, não o papel", () => {
    expect(describeActionError(apiError(403, { error: "feature_disabled", feature: "ledi_export" })))
      .toEqual({ kind: "rejected", code: "feature_disabled", message: "esta funcionalidade está desligada para a cidade" });
  });
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/features.test.ts src/lib/actionErrors.test.ts`
Expected: FAIL — `./features` não existe; o 403 cai em "seu papel não permite esta ação".

- [ ] **Step 3: `SessionUser` (`src/lib/api.ts`)**

Troque

```ts
  // Fuso IANA da cidade do host (api#27); ausente em api antigo.
  time_zone?: string;
}
```

por

```ts
  // Fuso IANA da cidade do host (api#27); ausente em api antigo.
  time_zone?: string;
  // Módulo 16 (session-v1.1.0): interruptores LIGADOS da cidade do host.
  // Ausente em api antigo e na sessão do console; leia por sessionFeatures.
  features?: string[];
}
```

- [ ] **Step 4: Regras (`src/lib/features.ts`)**

```ts
// src/lib/features.ts
// Interruptores por cidade (módulo 16; ADR 0028; contratos §1 e §2). A sessão
// traz as chaves LIGADAS, não necessariamente utilizáveis: o que falta para
// funcionar vem de GET /integrations. Ausente = [] (api antigo, sessão do
// console de plataforma); chave desconhecida é ignorada por quem não a usa.
// Só o maintenance liga e desliga; o dashboard nunca escreve interruptor.
import { ApiError } from "./api";

export type FeatureKey = "ledi_export" | "cadsus_lookup";

export const FEATURE_DISABLED_MESSAGE = "esta funcionalidade está desligada para a cidade";

const FEATURE_LABEL: Record<string, string> = {
  ledi_export: "Envio da produção ao e-SUS (LEDI)",
  cadsus_lookup: "Consulta ao CADSUS na validação presencial"
};

export function sessionFeatures(user: { features?: unknown } | null | undefined): string[] {
  const raw = user?.features;
  return Array.isArray(raw) ? raw.filter((key): key is string => typeof key === "string") : [];
}

export function hasFeature(user: { features?: unknown } | null | undefined, key: FeatureKey): boolean {
  return sessionFeatures(user).includes(key);
}

// 403 { error: "feature_disabled", feature } → a chave ("" sem `feature`);
// qualquer outro erro → null. Um 403 de papel (missing_role) não é isto.
export function featureDisabledKey(err: unknown): string | null {
  if (!(err instanceof ApiError) || err.status !== 403) return null;
  const body = err.body as { error?: unknown; feature?: unknown } | null;
  if (!body || typeof body !== "object" || body.error !== "feature_disabled") return null;
  return typeof body.feature === "string" ? body.feature : "";
}

export function featureLabel(key: string): string {
  return FEATURE_LABEL[key] ?? key;
}
```

- [ ] **Step 5: `src/lib/actionErrors.ts`**

Acrescente o import, depois de `import { ApiError } from "./api";`:

```ts
import { FEATURE_DISABLED_MESSAGE } from "./features";
```

E troque

```ts
  if (err.status === 403) return { kind: "forbidden", message: "seu papel não permite esta ação" };
```

por

```ts
  // Módulo 16 (contratos §1): o mantenedor desligou a funcionalidade entre a
  // leitura da tela e o clique. Não é questão de papel.
  if (err.status === 403 && code === "feature_disabled") {
    return { kind: "rejected", code, message: FEATURE_DISABLED_MESSAGE };
  }
  if (err.status === 403) return { kind: "forbidden", message: "seu papel não permite esta ação" };
```

- [ ] **Step 6: Rode e veja passar**

Run: `npx vitest run src/lib/features.test.ts src/lib/actionErrors.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/lib/api.ts src/lib/features.ts src/lib/features.test.ts src/lib/actionErrors.ts src/lib/actionErrors.test.ts
/opt/homebrew/bin/git commit -m "feat: read city feature switches from the session

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Cliente HTTP do módulo 16, tipos do contrato e proxy

**Files:**
- Modify: `src/lib/api.ts` (`verifyCitizen` e fim do arquivo), `vite.config.ts`, `README.md`
- Test: `src/lib/api.recordMode.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `ApiError`, `ATTENDANCE_BASE` (já em `src/lib/api.ts`).
- Produces (em `src/lib/api.ts`):
  - tipos `RecordMode`, `CredentialKind`, `CredentialCheckStatus`, `IntegrationCredential`, `CityFeatureState`, `Integrations` (contratos §5.1);
  - `getIntegrations(): Promise<Integrations>`, `setIntegrationCredential(kind, username, password): Promise<IntegrationCredential>`, `checkIntegrationCredential(kind): Promise<IntegrationCredential>`;
  - tipos `CnesProposalKind`, `CnesProposalAction`, `CnesSide`, `CnesProposal`, `CnesDivergence`, `CnesOverview`, `CnesApplyResult` (contratos §5.2);
  - `getCnes(): Promise<CnesOverview>`, `applyCnesProposals(ids: string[]): Promise<CnesApplyResult>`;
  - tipos `CompetenceAlert`, `FichaStatus`, `LediFicha`, `Production` (contratos §5.3, com `fichas_total`), constante `FICHAS_PER_PAGE = 50`;
  - `getProduction(competence: string | null, page: number): Promise<Production>`, `resendFicha(id: string): Promise<LediFicha>`;
  - `CadsusLookupResult`, `cadsusLookup(cpf, code): Promise<CadsusLookupResult>` (contratos §5.4);
  - `VerifyExtra { cadsus_confirmed?: boolean }` e `verifyCitizen(cpf, code, extra: VerifyExtra = {})` — sem `extra`, o corpo é exatamente o de antes.

- [ ] **Step 1: Escreva o teste do cliente**

```ts
// src/lib/api.recordMode.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import {
  ApiError, applyCnesProposals, cadsusLookup, checkIntegrationCredential, getCnes, getIntegrations, getProduction,
  resendFicha, setIntegrationCredential, verifyCitizen
} from "./api";

afterEach(() => vi.unstubAllGlobals());

function stub(body: unknown, status = 200) {
  const fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
    new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fn);
  return fn;
}
const call = (fn: ReturnType<typeof stub>, i = 0) => fn.mock.calls[i] as [ string, RequestInit ];
const ITEM = {
  kind: "ledi", set: true, set_at: "2026-10-01T13:00:00Z", set_by: "admin@curitiba.demo",
  last_check_at: null, last_check_status: null, last_check_message: null
};

describe("cliente de Integrações (contratos §5.1)", () => {
  it("lê /integrations com o cookie", async () => {
    const fn = stub({ record_mode: "off", pec_url_set: false, ibge_code_set: true, credentials: [], features: [] });
    expect((await getIntegrations()).ibge_code_set).toBe(true);
    expect(call(fn)[0]).toBe("/integrations");
    expect(call(fn)[1].credentials).toBe("include");
  });

  it("a senha vai como digitada, sem trim, e só no corpo", async () => {
    const fn = stub(ITEM);
    await setIntegrationCredential("ledi", "integ.pec", "  s3nh@ com espaço  ");
    expect(call(fn)[0]).toBe("/integrations/credentials/ledi");
    expect(call(fn)[1].method).toBe("PUT");
    expect(JSON.parse(call(fn)[1].body as string)).toEqual({ username: "integ.pec", password: "  s3nh@ com espaço  " });
    expect(call(fn)[0]).not.toContain("s3nh");
  });

  it("testa a conexão com POST e devolve o item atualizado", async () => {
    const fn = stub({ ...ITEM, kind: "cadsus", last_check_status: "ok" });
    expect((await checkIntegrationCredential("cadsus")).last_check_status).toBe("ok");
    expect(call(fn)[0]).toBe("/integrations/credentials/cadsus/check");
    expect(call(fn)[1].method).toBe("POST");
  });

  it("409 credential_missing é exceção com o código", async () => {
    stub({ error: "credential_missing" }, 409);
    const err = await checkIntegrationCredential("ledi").catch((e: unknown) => e);
    expect(err).toBeInstanceOf(ApiError);
    expect((err as ApiError).body).toEqual({ error: "credential_missing" });
  });
});

describe("cliente do CNES (contratos §5.2)", () => {
  it("lê /cnes e aplica o lote em /cnes/apply", async () => {
    const fn = stub({ snapshot: null, proposals: [], divergences: [] });
    expect((await getCnes()).snapshot).toBeNull();
    expect(call(fn)[0]).toBe("/cnes");
    const fn2 = stub({ applied: 1, skipped: [ { id: "p2", reason: "stale" } ] });
    expect((await applyCnesProposals([ "p1", "p2" ])).skipped[0].reason).toBe("stale");
    expect(call(fn2)[0]).toBe("/cnes/apply");
    expect(call(fn2)[1].method).toBe("POST");
    expect(JSON.parse(call(fn2)[1].body as string)).toEqual({ proposal_ids: [ "p1", "p2" ] });
  });
});

describe("cliente da Produção (contratos §5.3)", () => {
  const PROD = {
    competence: "202610", deadline_on: "2026-11-16", business_days_left: 7, alert: "none",
    counts: { accepted: 0, rejected: 0, pending: 0, sending: 0, failed: 0 }, rejections: [], fichas: [], fichas_total: 0
  };

  it("sem competência nem página: /production puro (a API usa a corrente)", async () => {
    const fn = stub(PROD);
    await getProduction(null, 1);
    expect(call(fn)[0]).toBe("/production");
  });

  it("competência e página vão na query", async () => {
    const fn = stub(PROD);
    await getProduction("202609", 3);
    expect(call(fn)[0]).toBe("/production?competence=202609&page=3");
  });

  it("reenvia a ficha com POST no id escapado", async () => {
    const fn = stub({ id: "a/b", ficha_type: "synthetic", status: "pending", attempts: 2, last_error: null,
      created_at: "2026-10-02T13:00:00Z", accepted_at: null });
    expect((await resendFicha("a/b")).status).toBe("pending");
    expect(call(fn)[0]).toBe("/production/fichas/a%2Fb/resend");
    expect(call(fn)[1].method).toBe("POST");
  });
});

describe("CADSUS no balcão (contratos §5.4)", () => {
  it("consulta com CPF e código no corpo, nunca na URL", async () => {
    const fn = stub({ found: true, cns_masked: "7** **** **** 1234", birth_date_matches: true, sex_matches: null });
    expect((await cadsusLookup("529.982.247-25", "123456")).cns_masked).toBe("7** **** **** 1234");
    expect(call(fn)[0]).toBe("/attendance/cadsus_lookup");
    expect(JSON.parse(call(fn)[1].body as string)).toEqual({ cpf: "529.982.247-25", code: "123456" });
  });

  it("validação sem extra manda o corpo de antes; com extra, cadsus_confirmed", async () => {
    const fn = stub({ verification: { id: "v1" } }, 201);
    await verifyCitizen("529.982.247-25", "123456");
    expect(JSON.parse(call(fn)[1].body as string)).toEqual({ cpf: "529.982.247-25", code: "123456", document_checked: true });
    await verifyCitizen("529.982.247-25", "123456", { cadsus_confirmed: true });
    expect(JSON.parse(call(fn, 1)[1].body as string))
      .toEqual({ cpf: "529.982.247-25", code: "123456", document_checked: true, cadsus_confirmed: true });
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/api.recordMode.test.ts`
Expected: FAIL — `getIntegrations`, `getCnes`, `getProduction`, `cadsusLookup` e os demais não existem.

- [ ] **Step 3: `verifyCitizen` e `cadsusLookup` (`src/lib/api.ts`)**

Troque

```ts
export async function verifyCitizen(cpf: string, code: string): Promise<void> {
  await jsonFetch<unknown>(`${ATTENDANCE_BASE}/verifications`, {
    method: "POST", body: JSON.stringify({ cpf, code, document_checked: true })
  });
}
```

por

```ts
// Módulo 16 (contratos §5.4): `cadsus_confirmed` só vai quando a cidade tem a
// consulta ao CADSUS ligada (o atendente decidiu, sim ou não). Sem `extra`, o
// corpo é exatamente o de antes.
export interface VerifyExtra { cadsus_confirmed?: boolean }

export async function verifyCitizen(cpf: string, code: string, extra: VerifyExtra = {}): Promise<void> {
  await jsonFetch<unknown>(`${ATTENDANCE_BASE}/verifications`, {
    method: "POST", body: JSON.stringify({ cpf, code, document_checked: true, ...extra })
  });
}

// Consulta ao CADSUS do par do código (contratos §5.4: o CPF vai junto,
// porque o código só identifica o par com ele). Nunca traz nome,
// mãe ou endereço: só o CNS mascarado e se nascimento e sexo conferem.
export interface CadsusLookupResult {
  found: boolean;
  cns_masked: string | null;
  birth_date_matches: boolean | null;
  sex_matches: boolean | null;
}

export async function cadsusLookup(cpf: string, code: string): Promise<CadsusLookupResult> {
  return jsonFetch<CadsusLookupResult>(`${ATTENDANCE_BASE}/cadsus_lookup`, {
    method: "POST", body: JSON.stringify({ cpf, code })
  });
}
```

- [ ] **Step 4: Integrações, CNES e Produção (fim de `src/lib/api.ts`)**

```ts
// ─── Módulo 16: modo de prontuário e exportação (ADR 0028; contratos §5) ─────
// Rotas da cidade, sessão municipal. Nenhuma devolve segredo: a senha da
// credencial só vai (PUT) e nunca volta. CPF e CNS chegam mascarados da API e
// são mostrados como vieram.
const INTEGRATIONS_BASE = import.meta.env.VITE_INTEGRATIONS_BASE || "/integrations";
const CNES_BASE = import.meta.env.VITE_CNES_BASE || "/cnes";
const PRODUCTION_BASE = import.meta.env.VITE_PRODUCTION_BASE || "/production";

export type RecordMode = "off" | "integrated" | "record";
export type CredentialKind = "ledi" | "cadsus";
export type CredentialCheckStatus = "ok" | "unauthorized" | "unreachable" | "error";

export interface IntegrationCredential {
  kind: CredentialKind;
  set: boolean;
  set_at: string | null;
  set_by: string | null;
  last_check_at: string | null;
  last_check_status: CredentialCheckStatus | null;
  last_check_message: string | null;
}
export interface CityFeatureState { key: string; enabled: boolean; usable: boolean; missing: string[] }
export interface Integrations {
  record_mode: RecordMode;
  pec_url_set: boolean;
  ibge_code_set: boolean;
  credentials: IntegrationCredential[];
  features: CityFeatureState[];
}

export async function getIntegrations(): Promise<Integrations> {
  return jsonFetch<Integrations>(INTEGRATIONS_BASE);
}

// Escrita só. A senha vai como digitada (sem trim): uma senha do PEC com
// espaço nas pontas é outra senha.
export async function setIntegrationCredential(
  kind: CredentialKind, username: string, password: string
): Promise<IntegrationCredential> {
  return jsonFetch<IntegrationCredential>(`${INTEGRATIONS_BASE}/credentials/${encodeURIComponent(kind)}`, {
    method: "PUT", body: JSON.stringify({ username, password })
  });
}

export async function checkIntegrationCredential(kind: CredentialKind): Promise<IntegrationCredential> {
  return jsonFetch<IntegrationCredential>(`${INTEGRATIONS_BASE}/credentials/${encodeURIComponent(kind)}/check`, {
    method: "POST", body: "{}"
  });
}

export type CnesProposalKind = "unit" | "team" | "member";
export type CnesProposalAction = "link" | "create" | "end";
// Um lado da proposta (contratos §5.2): cada chave
// só vem quando se aplica ao tipo; CPF e CNS sempre mascarados.
export interface CnesSide {
  name?: string | null;
  cnes?: string | null;
  ine?: string | null;
  cbo?: string | null;
  cpf_masked?: string | null;
  cns_masked?: string | null;
}
export interface CnesProposal {
  id: string;
  kind: CnesProposalKind;
  action: CnesProposalAction;
  local: CnesSide | null;
  cnes: CnesSide;
  confidence: "exact" | "probable";
}
export interface CnesDivergence {
  kind: string;
  subject: { type: string; id: string; label: string };
  detail: string | null;
}
export interface CnesOverview {
  snapshot: { competence: string; imported_at: string } | null;
  proposals: CnesProposal[];
  divergences: CnesDivergence[];
}
export interface CnesApplyResult { applied: number; skipped: { id: string; reason: string }[] }

export async function getCnes(): Promise<CnesOverview> {
  return jsonFetch<CnesOverview>(CNES_BASE);
}

export async function applyCnesProposals(proposalIds: string[]): Promise<CnesApplyResult> {
  return jsonFetch<CnesApplyResult>(`${CNES_BASE}/apply`, {
    method: "POST", body: JSON.stringify({ proposal_ids: proposalIds })
  });
}

export type CompetenceAlert = "none" | "attention" | "critical";
export type FichaStatus = "pending" | "sending" | "accepted" | "rejected" | "failed";
export interface LediFicha {
  id: string;
  ficha_type: string;
  status: FichaStatus;
  attempts: number;
  last_error: string | null;
  created_at: string;
  accepted_at: string | null;
}
export interface Production {
  competence: string;
  deadline_on: string;
  business_days_left: number;
  alert: CompetenceAlert;
  counts: Record<FichaStatus, number>;
  rejections: { message: string; count: number }[];
  fichas: LediFicha[];
  // Total de fichas da competência, para a paginação (contratos §5.3).
  fichas_total: number;
}
// Página fixa da API (contratos §5.3).
export const FICHAS_PER_PAGE = 50;

// `competence` null = a corrente, decidida pela API no fuso da cidade.
export async function getProduction(competence: string | null, page: number): Promise<Production> {
  const params = new URLSearchParams();
  if (competence) params.set("competence", competence);
  if (page > 1) params.set("page", String(page));
  const qs = params.toString();
  return jsonFetch<Production>(qs ? `${PRODUCTION_BASE}?${qs}` : PRODUCTION_BASE);
}

export async function resendFicha(id: string): Promise<LediFicha> {
  return jsonFetch<LediFicha>(`${PRODUCTION_BASE}/fichas/${encodeURIComponent(id)}/resend`, {
    method: "POST", body: "{}"
  });
}
```

- [ ] **Step 5: Proxy (`vite.config.ts`) e `README.md`**

Em `vite.config.ts`, troque

```ts
//   /campaigns  → campanhas (módulo 12; campaign_manager, e a chave de SMS também para municipal_admin).
```

por

```ts
//   /campaigns  → campanhas (módulo 12; campaign_manager, e a chave de SMS também para municipal_admin).
//   /integrations, /cnes, /production → módulo 16 (credenciais, CNES e produção e-SUS).
```

e troque

```ts
      "/campaigns": proxy(TARGET)
```

por

```ts
      "/campaigns": proxy(TARGET),
      "/integrations": proxy(TARGET),
      "/cnes": proxy(TARGET),
      "/production": proxy(TARGET)
```

No `README.md`, troque

```
`/auth`, `/setup`, `/protocols`, `/mfa`, `/attendance`, `/professionals`, `/territory` e `/campaigns` para
```

por

```
`/auth`, `/setup`, `/protocols`, `/mfa`, `/attendance`, `/professionals`, `/territory`, `/campaigns`,
`/integrations`, `/cnes` e `/production` para
```

- [ ] **Step 6: Rode e veja passar**

Run: `npx vitest run src/lib/api.recordMode.test.ts src/modules/Attendance.test.tsx && npx tsc --noEmit`
Expected: PASS. O teste existente do balcão continua esperando `verifyCitizen("529.982.247-25", "123456")` com dois argumentos e passa, porque o `Counter` ainda não mudou.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/lib/api.ts src/lib/api.recordMode.test.ts vite.config.ts README.md
/opt/homebrew/bin/git commit -m "feat: add the record mode client for integrations, CNES, production and CADSUS

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Regras das Integrações

**Files:**
- Create: `src/lib/integrations.ts`, `src/test/recordModeFixtures.ts`
- Test: `src/lib/integrations.test.ts`

**Interfaces:**
- Consumes: `IntegrationCredential`, `CityFeatureState`, `CredentialCheckStatus`, `ApiError` (Task 2); `describeActionError` (`src/lib/actionErrors.ts`); `fmtDateTime` (`src/lib/format.ts`).
- Produces (em `src/lib/integrations.ts`):
  - `INTEGRATIONS_KEY = [ "integrations" ] as const`;
  - `recordModeLabel(mode: string): string`, `credentialLabel(kind: string): string`, `checkLabel(status: CredentialCheckStatus | null): string`;
  - `setSummary(c: IntegrationCredential): string`;
  - `checkSummary(c: IntegrationCredential): { text: string; tone: "ok" | "warn" | "down" | "neutral" }`;
  - `missingPhrase(code: string): string`;
  - `featureState(f: CityFeatureState): { state: "on" | "blocked" | "off"; label: string; tone: "ok" | "warn" | "neutral" }`;
  - `integrationsError(err: unknown): string`.
- Produces (em `src/test/recordModeFixtures.ts`): `credential(overrides?)`, `integrationsFixture(overrides?)`.

- [ ] **Step 1: Fixtures**

```ts
// src/test/recordModeFixtures.ts
// Dados comuns aos testes do módulo 16. Só tipos do api: cada arquivo de teste
// faz o próprio vi.mock("…/lib/api").
import type { IntegrationCredential, Integrations } from "../lib/api";

export function credential(overrides: Partial<IntegrationCredential> = {}): IntegrationCredential {
  return {
    kind: "ledi", set: true, set_at: "2026-10-01T13:00:00Z", set_by: "admin@curitiba.demo",
    last_check_at: "2026-10-02T12:30:00Z", last_check_status: "ok", last_check_message: null,
    ...overrides
  };
}

export function integrationsFixture(overrides: Partial<Integrations> = {}): Integrations {
  return {
    record_mode: "integrated", pec_url_set: true, ibge_code_set: true,
    credentials: [
      credential(),
      credential({ kind: "cadsus", set: false, set_at: null, set_by: null, last_check_at: null, last_check_status: null })
    ],
    features: [
      { key: "ledi_export", enabled: true, usable: true, missing: [] },
      { key: "cadsus_lookup", enabled: false, usable: false, missing: [ "credential_missing:cadsus" ] }
    ],
    ...overrides
  };
}
```

- [ ] **Step 2: Escreva o teste**

```ts
// src/lib/integrations.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  checkLabel, checkSummary, credentialLabel, featureState, integrationsError, missingPhrase, recordModeLabel, setSummary
} from "./integrations";
import { credential } from "../test/recordModeFixtures";

const apiError = (status: number, body: unknown) => new ApiError(status, body, String(status));

describe("integrações — rótulos", () => {
  it("modo de prontuário e credencial em português; desconhecido sai cru", () => {
    expect(recordModeLabel("off")).toBe("desligado — o Rota Saúde não envia produção");
    expect(recordModeLabel("integrated")).toBe("integrado — o PEC é o prontuário; o Rota Saúde envia o que registra");
    expect(recordModeLabel("record")).toBe("prontuário — o Rota Saúde é o prontuário da cidade");
    expect(recordModeLabel("hybrid")).toBe("hybrid");
    expect(credentialLabel("ledi")).toBe("e-SUS PEC (envio LEDI)");
    expect(credentialLabel("cadsus")).toBe("CADSUS");
    expect(credentialLabel("rnds")).toBe("rnds");
  });

  it("cadastro: quando e por quem; não cadastrada", () => {
    expect(setSummary(credential())).toMatch(/^cadastrada em 01\/10\/2026.*10:00 por admin@curitiba\.demo$/);
    expect(setSummary(credential({ set_by: null }))).not.toContain(" por ");
    expect(setSummary(credential({ set: false, set_at: null, set_by: null }))).toBe("não cadastrada");
  });

  it("último teste: frase e tom por situação", () => {
    expect(checkSummary(credential({ set: false }))).toEqual({ text: "não cadastrada", tone: "neutral" });
    expect(checkSummary(credential({ last_check_status: null, last_check_at: null }))).toEqual({ text: "nunca testada", tone: "neutral" });
    expect(checkSummary(credential()).text).toMatch(/^conexão ok · 02\/10\/2026/);
    expect(checkSummary(credential()).tone).toBe("ok");
    expect(checkSummary(credential({ last_check_status: "unauthorized" })).tone).toBe("down");
    expect(checkSummary(credential({ last_check_status: "unreachable" })).tone).toBe("warn");
    expect(checkSummary(credential({ last_check_status: "error" })).tone).toBe("warn");
    expect(checkLabel(null)).toBe("nunca testada");
    expect(checkLabel("unauthorized")).toBe("usuário ou senha recusados");
  });
});

describe("integrações — o que falta (contratos §2)", () => {
  it("cada código em linguagem simples, dizendo quem resolve", () => {
    expect(missingPhrase("record_mode_off"))
      .toBe("o modo de prontuário da cidade ainda está desligado (quem define é o operador da plataforma)");
    expect(missingPhrase("pec_url_missing")).toBe("falta o endereço do PEC da cidade (quem cadastra é o operador da plataforma)");
    expect(missingPhrase("ibge_code_missing")).toBe("falta o código IBGE da cidade (quem cadastra é o operador da plataforma)");
    expect(missingPhrase("credential_missing:cadsus")).toBe("cadastre a credencial do CADSUS, no quadro Credenciais");
    expect(missingPhrase("credential_unauthorized:ledi"))
      .toBe("a credencial do e-SUS PEC (envio LEDI) foi recusada — troque o usuário e a senha e teste de novo");
  });

  it("código desconhecido nunca some", () => {
    expect(missingPhrase("rnds_certificate_missing")).toBe("pendência não reconhecida: rnds_certificate_missing");
    expect(missingPhrase("credential_missing:rnds")).toBe("cadastre a credencial do rnds, no quadro Credenciais");
  });

  it("estado da funcionalidade: ligada e funcionando, ligada e parada, desligada", () => {
    expect(featureState({ key: "ledi_export", enabled: true, usable: true, missing: [] }))
      .toEqual({ state: "on", label: "ligada e funcionando", tone: "ok" });
    expect(featureState({ key: "ledi_export", enabled: true, usable: false, missing: [ "pec_url_missing" ] }))
      .toEqual({ state: "blocked", label: "ligada, mas parada até resolver o que falta", tone: "warn" });
    expect(featureState({ key: "cadsus_lookup", enabled: false, usable: false, missing: [] }))
      .toEqual({ state: "off", label: "desligada", tone: "neutral" });
  });
});

describe("integrações — erros", () => {
  it("códigos da tela viram frase; o resto segue o padrão das ações", () => {
    expect(integrationsError(apiError(422, { error: "invalid_credential" }))).toBe("preencha usuário e senha");
    expect(integrationsError(apiError(422, { error: "unknown_kind" }))).toBe("tipo de credencial desconhecido — recarregue a página");
    expect(integrationsError(apiError(409, { error: "credential_missing" }))).toBe("cadastre a credencial antes de testar");
    expect(integrationsError(apiError(403, { error: "missing_role" }))).toBe("seu papel não permite esta ação");
    expect(integrationsError(apiError(401, ""))).toBe("sessão expirada — entre de novo");
    expect(integrationsError(new Error("x"))).toBe("não foi possível concluir — tente de novo");
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/lib/integrations.test.ts`
Expected: FAIL — `./integrations` não existe.

- [ ] **Step 4: Implemente (`src/lib/integrations.ts`)**

```ts
// src/lib/integrations.ts
// Integrações da cidade (módulo 16; ADR 0028; spec §3; contratos §2 e §5.1).
// Modo, PEC e IBGE são do operador da plataforma; credenciais são da cidade;
// interruptores são do mantenedor. Aqui só se traduz o estado em português.
import { ApiError, type CityFeatureState, type CredentialCheckStatus, type IntegrationCredential } from "./api";
import { describeActionError } from "./actionErrors";
import { fmtDateTime } from "./format";

export const INTEGRATIONS_KEY = [ "integrations" ] as const;

const RECORD_MODE_LABEL: Record<string, string> = {
  off: "desligado — o Rota Saúde não envia produção",
  integrated: "integrado — o PEC é o prontuário; o Rota Saúde envia o que registra",
  record: "prontuário — o Rota Saúde é o prontuário da cidade"
};

const CREDENTIAL_LABEL: Record<string, string> = { ledi: "e-SUS PEC (envio LEDI)", cadsus: "CADSUS" };

const CHECK_LABEL: Record<CredentialCheckStatus, string> = {
  ok: "conexão ok",
  unauthorized: "usuário ou senha recusados",
  unreachable: "serviço fora do ar ou endereço errado",
  error: "erro no teste"
};

const MESSAGES: Record<string, string> = {
  invalid_credential: "preencha usuário e senha",
  unknown_kind: "tipo de credencial desconhecido — recarregue a página",
  credential_missing: "cadastre a credencial antes de testar"
};

export function recordModeLabel(mode: string): string {
  return RECORD_MODE_LABEL[mode] ?? mode;
}

export function credentialLabel(kind: string): string {
  return CREDENTIAL_LABEL[kind] ?? kind;
}

export function checkLabel(status: CredentialCheckStatus | null): string {
  return status ? CHECK_LABEL[status] ?? status : "nunca testada";
}

export function setSummary(c: IntegrationCredential): string {
  if (!c.set) return "não cadastrada";
  return `cadastrada em ${fmtDateTime(c.set_at)}${c.set_by ? ` por ${c.set_by}` : ""}`;
}

export function checkSummary(c: IntegrationCredential): { text: string; tone: "ok" | "warn" | "down" | "neutral" } {
  if (!c.set) return { text: "não cadastrada", tone: "neutral" };
  if (!c.last_check_status) return { text: "nunca testada", tone: "neutral" };
  const tone = c.last_check_status === "ok" ? "ok" : c.last_check_status === "unauthorized" ? "down" : "warn";
  return { text: `${checkLabel(c.last_check_status)} · ${fmtDateTime(c.last_check_at)}`, tone };
}

// Pré-requisito que falta (contratos §2), dizendo quem resolve. Código novo
// do api aparece cru, nunca some: esconder um motivo deixaria a cidade sem
// saber por que a funcionalidade não anda.
export function missingPhrase(code: string): string {
  const [ head, kind = "" ] = code.split(":");
  switch (head) {
    case "record_mode_off":
      return "o modo de prontuário da cidade ainda está desligado (quem define é o operador da plataforma)";
    case "pec_url_missing":
      return "falta o endereço do PEC da cidade (quem cadastra é o operador da plataforma)";
    case "ibge_code_missing":
      return "falta o código IBGE da cidade (quem cadastra é o operador da plataforma)";
    case "credential_missing":
      return `cadastre a credencial do ${credentialLabel(kind)}, no quadro Credenciais`;
    case "credential_unauthorized":
      return `a credencial do ${credentialLabel(kind)} foi recusada — troque o usuário e a senha e teste de novo`;
    default:
      return `pendência não reconhecida: ${code}`;
  }
}

export function featureState(f: CityFeatureState): { state: "on" | "blocked" | "off"; label: string; tone: "ok" | "warn" | "neutral" } {
  if (f.enabled && f.usable) return { state: "on", label: "ligada e funcionando", tone: "ok" };
  if (f.enabled) return { state: "blocked", label: "ligada, mas parada até resolver o que falta", tone: "warn" };
  return { state: "off", label: "desligada", tone: "neutral" };
}

export function integrationsError(err: unknown): string {
  const code = err instanceof ApiError ? (err.body as { error?: unknown } | null)?.error : null;
  if (typeof code === "string" && MESSAGES[code]) return MESSAGES[code];
  return describeActionError(err).message;
}
```

- [ ] **Step 5: Rode e veja passar**

Run: `npx vitest run src/lib/integrations.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/lib/integrations.ts src/lib/integrations.test.ts src/test/recordModeFixtures.ts
/opt/homebrew/bin/git commit -m "feat: describe integration credentials and missing prerequisites in plain language

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Tela Integrações (estado, teste de conexão e credencial com step-up)

**Files:**
- Modify: `src/components/SensitiveAction.tsx`
- Create: `src/modules/Integrations.tsx`
- Test: `src/components/SensitiveAction.test.tsx`, `src/modules/Integrations.test.tsx`

**Interfaces:**
- Consumes: `getIntegrations`, `setIntegrationCredential`, `checkIntegrationCredential`, `IntegrationCredential`, `CityFeatureState` (Task 2); tudo de `src/lib/integrations.ts` (Task 3); `featureLabel` (Task 1); `SensitiveAction`.
- Produces:
  - `SensitiveField.type?: "text" | "password"` — `password` vira `<input type="password" autoComplete="new-password">`;
  - `Integrations({ onGoToSecurity }: { onGoToSecurity?(): void })`, ligada ao menu na Task 10.

- [ ] **Step 1: Teste do campo de senha no `SensitiveAction`**

Em `src/components/SensitiveAction.test.tsx`, no fim do `describe("SensitiveAction", ...)`, acrescente:

```tsx
  it("campo do tipo password não mostra o valor e não é autocompletado", async () => {
    const run = vi.fn().mockResolvedValue(undefined);
    fetchSession.mockResolvedValue(session({ mfa_verified_at: minutesAgo(1) }));
    renderAction({ run, fields: [ { name: "username", label: "Usuário", required: true },
      { name: "password", label: "Senha", type: "password", required: true } ] });
    const password = (await screen.findByLabelText("Senha")) as HTMLInputElement;
    expect(password.type).toBe("password");
    expect(password.getAttribute("autocomplete")).toBe("new-password");
    expect((screen.getByLabelText("Usuário") as HTMLInputElement).type).toBe("text");
    fireEvent.change(screen.getByLabelText("Usuário"), { target: { value: "integ" } });
    fireEvent.change(password, { target: { value: " a b " } });
    fireEvent.click(confirm());
    await waitFor(() => expect(run).toHaveBeenCalledWith({ username: "integ", password: " a b " }));
  });
```

- [ ] **Step 2: Teste da tela**

```tsx
// src/modules/Integrations.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return {
    ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(),
    getIntegrations: vi.fn(), setIntegrationCredential: vi.fn(), checkIntegrationCredential: vi.fn()
  };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { renderWithProviders, sessionWith } from "../test/campaignFixtures";
import { credential, integrationsFixture } from "../test/recordModeFixtures";
import { Integrations } from "./Integrations";

afterEach(cleanup);
const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("Integrações (módulo 16)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.stepUpMfa, api.getIntegrations, api.setIntegrationCredential, api.checkIntegrationCredential ]) {
      m(fn).mockReset();
    }
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "municipal_admin" ]));
    m(api.getIntegrations).mockResolvedValue(integrationsFixture());
  });

  it("mostra modo, PEC e IBGE só como estado, sem campo de edição", async () => {
    renderWithProviders(<Integrations />);
    expect(await screen.findByText("integrado — o PEC é o prontuário; o Rota Saúde envia o que registra")).not.toBeNull();
    expect(screen.getAllByText("cadastrado")).toHaveLength(2);
    expect(screen.queryAllByRole("textbox")).toHaveLength(0);
  });

  it("credenciais: cadastro, último teste e Testar travado sem credencial", async () => {
    renderWithProviders(<Integrations />);
    expect(await screen.findByText(/^cadastrada em 01\/10\/2026.* por admin@curitiba\.demo$/)).not.toBeNull();
    expect(screen.getByText(/^conexão ok · 02\/10\/2026/)).not.toBeNull();
    expect(screen.getAllByText("não cadastrada").length).toBeGreaterThan(0);
    expect((screen.getByRole("button", { name: "Testar conexão — CADSUS" }) as HTMLButtonElement).disabled).toBe(true);
    expect((screen.getByRole("button", { name: "Testar conexão — e-SUS PEC (envio LEDI)" }) as HTMLButtonElement).disabled).toBe(false);
  });

  it("funcionalidades: estado e o que falta em linguagem simples", async () => {
    renderWithProviders(<Integrations />);
    const ledi = within(await screen.findByRole("region", { name: "Envio da produção ao e-SUS (LEDI)" }));
    expect(ledi.getByText("ligada e funcionando")).not.toBeNull();
    const cadsus = within(screen.getByRole("region", { name: "Consulta ao CADSUS na validação presencial" }));
    expect(cadsus.getByText("desligada")).not.toBeNull();
    expect(cadsus.getByText("Antes de ligar, falta:")).not.toBeNull();
    expect(cadsus.getByText("cadastre a credencial do CADSUS, no quadro Credenciais")).not.toBeNull();
  });

  it("ligada mas parada: diz o que fazer para voltar a funcionar", async () => {
    m(api.getIntegrations).mockResolvedValue(integrationsFixture({ features: [
      { key: "ledi_export", enabled: true, usable: false, missing: [ "credential_unauthorized:ledi", "pec_url_missing" ] }
    ] }));
    renderWithProviders(<Integrations />);
    const ledi = within(await screen.findByRole("region", { name: "Envio da produção ao e-SUS (LEDI)" }));
    expect(ledi.getByText("ligada, mas parada até resolver o que falta")).not.toBeNull();
    expect(ledi.getByText("Para voltar a funcionar:")).not.toBeNull();
    expect(ledi.getByText("a credencial do e-SUS PEC (envio LEDI) foi recusada — troque o usuário e a senha e teste de novo")).not.toBeNull();
    expect(ledi.getByText("falta o endereço do PEC da cidade (quem cadastra é o operador da plataforma)")).not.toBeNull();
  });

  it("testar conexão chama a API, diz o resultado e relê", async () => {
    m(api.checkIntegrationCredential).mockResolvedValue(credential({ last_check_status: "unauthorized" }));
    renderWithProviders(<Integrations />);
    fireEvent.click(await screen.findByRole("button", { name: "Testar conexão — e-SUS PEC (envio LEDI)" }));
    expect(await screen.findByText("Teste de conexão — e-SUS PEC (envio LEDI): usuário ou senha recusados")).not.toBeNull();
    expect(api.checkIntegrationCredential).toHaveBeenCalledWith("ledi");
    await waitFor(() => expect(api.getIntegrations).toHaveBeenCalledTimes(2));
  });

  it("409 credential_missing no teste vira frase", async () => {
    m(api.checkIntegrationCredential).mockRejectedValue(new ApiError(409, { error: "credential_missing" }, "409"));
    renderWithProviders(<Integrations />);
    fireEvent.click(await screen.findByRole("button", { name: "Testar conexão — e-SUS PEC (envio LEDI)" }));
    expect(await screen.findByText("cadastre a credencial antes de testar")).not.toBeNull();
  });

  it("cadastrada mostra Trocar; não cadastrada mostra Cadastrar", async () => {
    renderWithProviders(<Integrations />);
    expect(await screen.findByRole("button", { name: "Trocar — e-SUS PEC (envio LEDI)" })).not.toBeNull();
    expect(screen.getByRole("button", { name: "Cadastrar — CADSUS" })).not.toBeNull();
  });

  it("cadastrar: campo de senha mascarado, senha como digitada, formulário fecha e a senha não fica na tela", async () => {
    m(api.setIntegrationCredential).mockResolvedValue(credential({ kind: "cadsus" }));
    renderWithProviders(<Integrations />);
    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar — CADSUS" }));
    const password = (await screen.findByLabelText("Senha")) as HTMLInputElement;
    expect(password.type).toBe("password");
    fireEvent.change(screen.getByLabelText("Usuário"), { target: { value: "  integ.cadsus " } });
    fireEvent.change(password, { target: { value: "  s3nh@ com espaço  " } });
    fireEvent.click(await screen.findByRole("button", { name: "Salvar credencial" }));
    await waitFor(() => expect(api.setIntegrationCredential).toHaveBeenCalledWith("cadsus", "integ.cadsus", "  s3nh@ com espaço  "));
    expect(await screen.findByText("Credencial do CADSUS salva. Teste a conexão para conferir.")).not.toBeNull();
    expect(screen.queryByLabelText("Senha")).toBeNull();
    expect(document.body.textContent).not.toContain("s3nh@");
    await waitFor(() => expect(api.getIntegrations).toHaveBeenCalledTimes(2));
  });

  it("janela de step-up fechada: pede o código antes de gravar", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "municipal_admin" ], { mfa_verified_at: null }));
    m(api.stepUpMfa).mockResolvedValue(undefined);
    m(api.setIntegrationCredential).mockResolvedValue(credential());
    renderWithProviders(<Integrations />);
    fireEvent.click(await screen.findByRole("button", { name: "Trocar — e-SUS PEC (envio LEDI)" }));
    fireEvent.change(await screen.findByLabelText("Usuário"), { target: { value: "integ.pec" } });
    fireEvent.change(screen.getByLabelText("Senha"), { target: { value: "nova" } });
    fireEvent.change(await screen.findByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar credencial" }));
    await waitFor(() => expect(api.stepUpMfa).toHaveBeenCalledWith("123456"));
    await waitFor(() => expect(api.setIntegrationCredential).toHaveBeenCalledWith("ledi", "integ.pec", "nova"));
  });

  it("422 invalid_credential vira a frase da tela", async () => {
    m(api.setIntegrationCredential).mockRejectedValue(new ApiError(422, { error: "invalid_credential" }, "422"));
    renderWithProviders(<Integrations />);
    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar — CADSUS" }));
    fireEvent.change(await screen.findByLabelText("Usuário"), { target: { value: "x" } });
    fireEvent.change(screen.getByLabelText("Senha"), { target: { value: "y" } });
    fireEvent.click(await screen.findByRole("button", { name: "Salvar credencial" }));
    expect(await screen.findByText("preencha usuário e senha")).not.toBeNull();
  });

  it("sem municipal_admin: não chama a API", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "analyst" ]));
    renderWithProviders(<Integrations />);
    expect(await screen.findByText("seu papel não permite ver as integrações")).not.toBeNull();
    expect(api.getIntegrations).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/components/SensitiveAction.test.tsx src/modules/Integrations.test.tsx`
Expected: FAIL — `./Integrations` não existe e `SensitiveField` não aceita `type`.

- [ ] **Step 4: Campo de senha no `SensitiveAction`**

Em `src/components/SensitiveAction.tsx`, troque

```ts
export interface SensitiveField { name: string; label: string; required?: boolean; }
```

por

```ts
// `type: "password"` (módulo 16, credencial de integração): o valor não
// aparece na tela e o navegador não o autocompleta com a senha do usuário.
export interface SensitiveField { name: string; label: string; required?: boolean; type?: "text" | "password"; }
```

e, no `fields.map`, troque

```tsx
            <input
              value={values[f.name] ?? ""}
              onChange={(e) => setValues((prev) => ({ ...prev, [f.name]: e.target.value }))}
              style={inputStyle}
            />
```

por

```tsx
            <input
              type={f.type ?? "text"}
              autoComplete={f.type === "password" ? "new-password" : undefined}
              value={values[f.name] ?? ""}
              onChange={(e) => setValues((prev) => ({ ...prev, [f.name]: e.target.value }))}
              style={inputStyle}
            />
```

- [ ] **Step 5: A tela (`src/modules/Integrations.tsx`)**

```tsx
// src/modules/Integrations.tsx
// Integrações da cidade (módulo 16; ADR 0028; spec §3; contratos §5.1): só
// municipal_admin. Modo, endereço do PEC e IBGE são do operador da plataforma
// e aparecem só como estado. Credenciais são da cidade: cadastrar e trocar
// passam pelo step-up (SensitiveAction); testar conexão não. Interruptores
// são do mantenedor e aparecem com o que falta, em linguagem simples.
// Nenhum segredo chega aqui; a senha digitada só vive no SensitiveAction.
import { useRef, useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import {
  checkIntegrationCredential, getIntegrations, setIntegrationCredential,
  type CityFeatureState, type IntegrationCredential
} from "../lib/api";
import { useAuth } from "../lib/auth";
import { featureLabel } from "../lib/features";
import {
  INTEGRATIONS_KEY, checkLabel, checkSummary, credentialLabel, featureState, integrationsError, missingPhrase,
  recordModeLabel, setSummary
} from "../lib/integrations";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { KeyValue } from "../components/KeyValue";
import { EmptyState } from "../components/EmptyState";
import { SensitiveAction } from "../components/SensitiveAction";
import { buttonStyle, disabledButtonStyle, secondaryButtonStyle } from "../components/formStyles";

const SCREEN_ERRORS = [ "invalid_credential", "unknown_kind" ];

export function Integrations({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  const { user } = useAuth();
  const queryClient = useQueryClient();
  const isAdmin = !!user && !user.operator && user.memberships.some((m) => m.role === "municipal_admin");
  const query = useQuery({ queryKey: INTEGRATIONS_KEY, queryFn: getIntegrations, enabled: isAdmin });
  const [ checking, setChecking ] = useState<string | null>(null);
  const [ editing, setEditing ] = useState<IntegrationCredential | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const saved = useRef("");

  if (!user) return null;
  if (!isAdmin) {
    return (
      <div style={column}>
        <PageHeader title="Integrações" sub="e-SUS PEC · CADSUS" />
        <EmptyState title="seu papel não permite ver as integrações" />
      </div>
    );
  }

  const reload = () => void queryClient.invalidateQueries({ queryKey: INTEGRATIONS_KEY });

  async function check(c: IntegrationCredential) {
    if (checking) return;
    setChecking(c.kind); setNotice(null); setError(null);
    try {
      const updated = await checkIntegrationCredential(c.kind);
      setNotice(`Teste de conexão — ${credentialLabel(c.kind)}: ${checkLabel(updated.last_check_status)}`);
      reload();
    } catch (err) {
      setError(integrationsError(err));
    } finally {
      setChecking(null);
    }
  }

  const data = query.data;

  return (
    <div style={column}>
      <PageHeader title="Integrações" sub="e-SUS PEC · CADSUS" />
      {query.isError && <p role="alert" style={alertStyle}>{integrationsError(query.error)}</p>}
      {error && <p role="alert" style={alertStyle}>{error}</p>}
      {notice && <p role="status" style={statusStyle}>{notice}</p>}
      {query.isPending && <p className="mono" style={loadingStyle}>carregando…</p>}

      {editing && (
        <SensitiveAction
          key={editing.kind}
          title={`${editing.set ? "Trocar" : "Cadastrar"} credencial — ${credentialLabel(editing.kind)}`}
          description="A senha é gravada cifrada e não aparece de novo, nem para você. Trocar substitui a anterior."
          requiresStepUp
          fields={[
            { name: "username", label: "Usuário", required: true },
            { name: "password", label: "Senha", type: "password", required: true }
          ]}
          confirmLabel="Salvar credencial"
          run={async (values) => {
            // Usuário sem espaços nas pontas; a senha vai como digitada.
            await setIntegrationCredential(editing.kind, values.username.trim(), values.password);
            saved.current = `Credencial do ${credentialLabel(editing.kind)} salva. Teste a conexão para conferir.`;
          }}
          onDone={() => { setEditing(null); setError(null); setNotice(saved.current); reload(); }}
          onCancel={() => setEditing(null)}
          onGoToSecurity={onGoToSecurity}
          translateError={(err) => {
            const code = (err as { body?: { error?: string } })?.body?.error;
            return code && SCREEN_ERRORS.includes(code) ? integrationsError(err) : null;
          }}
        />
      )}

      {data && (
        <>
          <Panel title="Prontuário da cidade" sub="definido pelo operador da plataforma">
            <div style={{ display: "flex", gap: 24, flexWrap: "wrap" }}>
              <KeyValue k="Modo" v={recordModeLabel(data.record_mode)} mono={false} />
              <KeyValue k="Endereço do PEC" v={data.pec_url_set ? "cadastrado" : "não cadastrado"} mono={false} />
              <KeyValue k="Código IBGE" v={data.ibge_code_set ? "cadastrado" : "não cadastrado"} mono={false} />
            </div>
          </Panel>

          <Panel title="Credenciais" sub="da cidade · a senha nunca aparece de novo">
            <DataTable<IntegrationCredential>
              cols={[
                { label: "Integração", w: "1.4fr", render: (c) => credentialLabel(c.kind) },
                { label: "Cadastro", w: "2fr", render: (c) => setSummary(c) },
                { label: "Último teste", w: "2fr", render: (c) => <CheckCell credential={c} /> },
                { label: "", w: "auto", align: "right", render: (c) => (
                  <span style={{ display: "flex", gap: 8, justifyContent: "flex-end" }}>
                    <button type="button" aria-label={`${c.set ? "Trocar" : "Cadastrar"} — ${credentialLabel(c.kind)}`}
                      style={buttonStyle}
                      onClick={() => { setNotice(null); setError(null); setEditing(c); }}>
                      {c.set ? "Trocar" : "Cadastrar"}
                    </button>
                    <button type="button" aria-label={`Testar conexão — ${credentialLabel(c.kind)}`}
                      disabled={!c.set || checking !== null}
                      style={!c.set || checking !== null ? disabledButtonStyle : secondaryButtonStyle}
                      onClick={() => void check(c)}>
                      {checking === c.kind ? "testando…" : "Testar conexão"}
                    </button>
                  </span>
                ) }
              ]}
              rows={data.credentials}
              rowKey={(c) => c.kind}
              empty="nenhuma credencial prevista"
            />
          </Panel>

          <Panel title="Funcionalidades" sub="ligadas e desligadas pela equipe do Rota Saúde">
            {data.features.length === 0 ? <EmptyState title="nenhuma funcionalidade com interruptor" /> : (
              <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
                {data.features.map((f) => <FeatureRow key={f.key} feature={f} />)}
              </div>
            )}
          </Panel>
        </>
      )}
    </div>
  );
}

function CheckCell({ credential }: { credential: IntegrationCredential }) {
  const summary = checkSummary(credential);
  return (
    <span style={{ display: "flex", flexDirection: "column", gap: 2 }}>
      <span><Tag tone={summary.tone}>{summary.text}</Tag></span>
      {credential.last_check_message && <small style={hint}>{credential.last_check_message}</small>}
    </span>
  );
}

function FeatureRow({ feature }: { feature: CityFeatureState }) {
  const state = featureState(feature);
  return (
    <section aria-label={featureLabel(feature.key)} style={featureBox}>
      <div style={{ display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" }}>
        <strong style={{ fontSize: 13 }}>{featureLabel(feature.key)}</strong>
        <Tag tone={state.tone}>{state.label}</Tag>
      </div>
      {feature.missing.length > 0 && (
        <>
          <p style={hint}>{feature.enabled ? "Para voltar a funcionar:" : "Antes de ligar, falta:"}</p>
          <ul style={{ margin: 0, paddingLeft: 18, fontSize: 12.5, color: "var(--ink2)" }}>
            {feature.missing.map((code) => <li key={code}>{missingPhrase(code)}</li>)}
          </ul>
        </>
      )}
    </section>
  );
}

const column = { display: "flex", flexDirection: "column" as const, gap: 16 };
const featureBox = { display: "flex", flexDirection: "column" as const, gap: 6, padding: "10px 12px",
  border: "1px solid var(--rule)", borderRadius: 8 };
const hint = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alertStyle = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const statusStyle = { margin: 0, fontSize: 13, fontWeight: 600 };
const loadingStyle = { margin: 0, fontSize: 10.5, color: "var(--ink3)" };
```

- [ ] **Step 6: Rode e veja passar**

Run: `npx vitest run src/components/SensitiveAction.test.tsx src/modules/Integrations.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/components/SensitiveAction.tsx src/components/SensitiveAction.test.tsx src/modules/Integrations.tsx src/modules/Integrations.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the integrations screen with step-up credential changes

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Competência e regras do CNES

**Files:**
- Create: `src/lib/competence.ts`, `src/lib/cnes.ts`
- Modify: `src/test/recordModeFixtures.ts` (acrescenta CNES)
- Test: `src/lib/competence.test.ts`, `src/lib/cnes.test.ts`

**Interfaces:**
- Consumes: `CnesSide`, `CnesProposal`, `CnesApplyResult`, `CnesOverview` (Task 2); `todayInCity` (`src/lib/campaigns.ts`, existente).
- Produces (em `src/lib/competence.ts`):
  - `isCompetence(value: string): boolean`, `competenceLabel(c: string): string` ("202610" → "10/2026");
  - `competenceOf(today: string): string` ("2026-10-31" → "202610");
  - `recentCompetences(today: string, count = 13): string[]` (corrente primeiro);
  - `competenceOptions(today: string, extra: string | null | undefined): string[]` (as recentes + `extra` se faltar, ordem decrescente).
- Produces (em `src/lib/cnes.ts`):
  - `CNES_KEY = [ "cnes" ] as const`;
  - `PROPOSAL_KIND_LABEL`, `PROPOSAL_ACTION_LABEL`, `CONFIDENCE` (`Record<string, { label; tone }>`);
  - `describeSide(side: CnesSide | null): string`, `divergenceLabel(kind: string): string`, `subjectLabel(subject: { type; label }): string`;
  - `toggleOne(selected: ReadonlySet<string>, id: string): Set<string>`, `toggleAll(selected, proposals): Set<string>`, `pruneSelection(selected, proposals): Set<string>`;
  - `applySummary(r: CnesApplyResult): string`.
- Produces (em `src/test/recordModeFixtures.ts`): `proposal(overrides?)`, `cnesFixture(overrides?)`.

- [ ] **Step 1: Fixtures (fim de `src/test/recordModeFixtures.ts`)**

Troque o import do topo

```ts
import type { IntegrationCredential, Integrations } from "../lib/api";
```

por

```ts
import type { CnesOverview, CnesProposal, IntegrationCredential, Integrations } from "../lib/api";
```

e acrescente no fim:

```ts
export function proposal(overrides: Partial<CnesProposal> = {}): CnesProposal {
  return {
    id: "p1", kind: "unit", action: "link", local: { name: "UBS Centro" },
    cnes: { name: "UBS CENTRO", cnes: "2384299" }, confidence: "exact", ...overrides
  };
}

export function cnesFixture(overrides: Partial<CnesOverview> = {}): CnesOverview {
  return {
    snapshot: { competence: "202609", imported_at: "2026-10-03T12:00:00Z" },
    proposals: [
      proposal(),
      proposal({
        id: "p2", kind: "member", action: "create", local: null, confidence: "probable",
        cnes: { name: "ANA SOUZA", ine: "0001234567", cbo: "225142", cpf_masked: "***.982.247-**", cns_masked: "7** **** **** 1234" }
      })
    ],
    divergences: [
      { kind: "cbo_mismatch", subject: { type: "professional", id: "pr1", label: "Bruno Lima" }, detail: "CBO local 225125, CNES 225142" }
    ],
    ...overrides
  };
}
```

- [ ] **Step 2: Escreva os testes**

```ts
// src/lib/competence.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { competenceLabel, competenceOf, competenceOptions, isCompetence, recentCompetences } from "./competence";
import { todayInCity } from "./campaigns";

describe("competência AAAAMM", () => {
  it("valida e rotula por texto", () => {
    expect(isCompetence("202610")).toBe(true);
    expect(isCompetence("202613")).toBe(false);
    expect(isCompetence("2026-10")).toBe(false);
    expect(competenceLabel("202610")).toBe("10/2026");
    expect(competenceLabel("lixo")).toBe("lixo");
  });

  it("recentes atravessam a virada do ano, corrente primeiro", () => {
    expect(recentCompetences("2026-02-10", 4)).toEqual([ "202602", "202601", "202512", "202511" ]);
    expect(recentCompetences("2026-10-05")).toHaveLength(13);
  });

  it("opções incluem a competência da API se ela não estiver entre as recentes", () => {
    expect(competenceOptions("2026-10-05", "202610").slice(0, 2)).toEqual([ "202610", "202609" ]);
    expect(competenceOptions("2026-10-05", "202501")).toContain("202501");
    expect(competenceOptions("2026-10-05", null)).toHaveLength(13);
  });
});

describe("competência corrente pelo dia da cidade", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    // 22h30 de 31/10 em São Paulo; em UTC já é 01/11.
    vi.setSystemTime(new Date("2026-11-01T01:30:00Z"));
  });
  afterEach(() => vi.useRealTimers());

  it("às 22h30 de 31/10 ainda é 10/2026", () => {
    expect(competenceOf(todayInCity())).toBe("202610");
  });
});
```

```ts
// src/lib/cnes.test.ts
import { describe, expect, it } from "vitest";
import {
  CONFIDENCE, PROPOSAL_ACTION_LABEL, PROPOSAL_KIND_LABEL, applySummary, describeSide, divergenceLabel, pruneSelection,
  subjectLabel, toggleAll, toggleOne
} from "./cnes";
import { proposal } from "../test/recordModeFixtures";

describe("CNES — rótulos", () => {
  it("tipo, ação e casamento", () => {
    expect(PROPOSAL_KIND_LABEL.unit).toBe("Unidade");
    expect(PROPOSAL_KIND_LABEL.member).toBe("Profissional na equipe");
    expect(PROPOSAL_ACTION_LABEL.end).toBe("encerrar (saiu do CNES)");
    expect(CONFIDENCE.probable).toEqual({ label: "provável — confira", tone: "warn" });
  });

  it("lado da proposta: só o que veio, CPF e CNS como vieram (mascarados)", () => {
    expect(describeSide({ name: "ANA SOUZA", ine: "0001234567", cbo: "225142", cpf_masked: "***.982.247-**", cns_masked: "7** **** **** 1234" }))
      .toBe("ANA SOUZA · INE 0001234567 · CBO 225142 · CPF ***.982.247-** · CNS 7** **** **** 1234");
    expect(describeSide({ name: "UBS CENTRO", cnes: "2384299", ine: null })).toBe("UBS CENTRO · CNES 2384299");
    expect(describeSide(null)).toBe("— (não existe no cadastro)");
    expect(describeSide({})).toBe("—");
  });

  it("divergência e assunto; desconhecido sai cru", () => {
    expect(divergenceLabel("no_bond_in_cnes")).toBe("profissional sem vínculo no CNES");
    expect(divergenceLabel("cbo_mismatch")).toBe("CBO diferente do CNES");
    expect(divergenceLabel("team_inactive_in_cnes")).toBe("equipe desativada no CNES");
    expect(divergenceLabel("unit_without_cnes")).toBe("unidade sem CNES");
    expect(divergenceLabel("other")).toBe("other");
    expect(subjectLabel({ type: "professional", label: "Bruno Lima" })).toBe("profissional · Bruno Lima");
    expect(subjectLabel({ type: "x", label: "Y" })).toBe("x · Y");
  });
});

describe("CNES — seleção", () => {
  const list = [ proposal({ id: "p1" }), proposal({ id: "p2" }) ];

  it("marca e desmarca uma; todas e nenhuma", () => {
    expect([ ...toggleOne(new Set(), "p1") ]).toEqual([ "p1" ]);
    expect([ ...toggleOne(new Set([ "p1" ]), "p1") ]).toEqual([]);
    expect([ ...toggleAll(new Set([ "p1" ]), list) ].sort()).toEqual([ "p1", "p2" ]);
    expect([ ...toggleAll(new Set([ "p1", "p2" ]), list) ]).toEqual([]);
    expect([ ...toggleAll(new Set(), []) ]).toEqual([]);
  });

  it("pruneSelection tira o que sumiu", () => {
    expect([ ...pruneSelection(new Set([ "p1", "gone" ]), list) ]).toEqual([ "p1" ]);
    expect([ ...pruneSelection(new Set([ "gone" ]), []) ]).toEqual([]);
  });

  it("applySummary diz as puladas", () => {
    expect(applySummary({ applied: 2, skipped: [] })).toBe("2 propostas aplicadas.");
    expect(applySummary({ applied: 1, skipped: [] })).toBe("1 proposta aplicada.");
    expect(applySummary({ applied: 0, skipped: [ { id: "p1", reason: "stale" } ] }))
      .toBe("Nenhuma proposta aplicada; 1 pulada porque mudou desde a leitura — confira a lista de novo.");
    expect(applySummary({ applied: 3, skipped: [ { id: "a", reason: "stale" }, { id: "b", reason: "stale" } ] }))
      .toBe("3 propostas aplicadas; 2 puladas porque mudaram desde a leitura — confira a lista de novo.");
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/lib/competence.test.ts src/lib/cnes.test.ts`
Expected: FAIL — `./competence` e `./cnes` não existem.

- [ ] **Step 4: `src/lib/competence.ts`**

```ts
// src/lib/competence.ts
// Competência AAAAMM (mês de produção do SISAB; módulo 16). Aritmética por
// texto, a partir do dia da CIDADE (todayInCity): às 22h30 de 31/10 em São
// Paulo já é novembro em UTC, e a competência ainda é outubro.
export function isCompetence(value: string): boolean {
  return /^\d{4}(0[1-9]|1[0-2])$/.test(value);
}

export function competenceLabel(c: string): string {
  return isCompetence(c) ? `${c.slice(4)}/${c.slice(0, 4)}` : c;
}

// "YYYY-MM-DD" → "YYYYMM".
export function competenceOf(today: string): string {
  return `${today.slice(0, 4)}${today.slice(5, 7)}`;
}

export function recentCompetences(today: string, count = 13): string[] {
  let year = Number(today.slice(0, 4));
  let month = Number(today.slice(5, 7));
  const out: string[] = [];
  for (let i = 0; i < count; i++) {
    out.push(`${year}${String(month).padStart(2, "0")}`);
    month -= 1;
    if (month === 0) { month = 12; year -= 1; }
  }
  return out;
}

// As recentes, mais a que a API respondeu se ela não estiver entre elas.
export function competenceOptions(today: string, extra: string | null | undefined): string[] {
  const list = recentCompetences(today);
  if (extra && isCompetence(extra) && !list.includes(extra)) list.push(extra);
  return list.sort((a, b) => b.localeCompare(a));
}
```

- [ ] **Step 5: `src/lib/cnes.ts`**

```ts
// src/lib/cnes.ts
// CNES da cidade (módulo 16; ADR 0028; spec §5; contratos §5.2). A tela só
// mostra e confirma: nenhuma proposta é aplicada sem a seleção explícita e o
// step-up. CPF e CNS chegam mascarados e saem como vieram.
import type { CnesApplyResult, CnesProposal, CnesSide } from "./api";

export const CNES_KEY = [ "cnes" ] as const;

export const PROPOSAL_KIND_LABEL: Record<string, string> = {
  unit: "Unidade", team: "Equipe", member: "Profissional na equipe"
};

export const PROPOSAL_ACTION_LABEL: Record<string, string> = {
  link: "vincular ao CNES", create: "criar a partir do CNES", end: "encerrar (saiu do CNES)"
};

export const CONFIDENCE: Record<string, { label: string; tone: "ok" | "warn" }> = {
  exact: { label: "exato", tone: "ok" },
  probable: { label: "provável — confira", tone: "warn" }
};

const DIVERGENCE_LABEL: Record<string, string> = {
  no_bond_in_cnes: "profissional sem vínculo no CNES",
  cbo_mismatch: "CBO diferente do CNES",
  team_inactive_in_cnes: "equipe desativada no CNES",
  unit_without_cnes: "unidade sem CNES"
};

const SUBJECT_LABEL: Record<string, string> = { unit: "unidade", team: "equipe", professional: "profissional" };

export function describeSide(side: CnesSide | null): string {
  if (!side) return "— (não existe no cadastro)";
  const parts = [
    side.name,
    side.cnes && `CNES ${side.cnes}`,
    side.ine && `INE ${side.ine}`,
    side.cbo && `CBO ${side.cbo}`,
    side.cpf_masked && `CPF ${side.cpf_masked}`,
    side.cns_masked && `CNS ${side.cns_masked}`
  ].filter((part): part is string => !!part);
  return parts.length > 0 ? parts.join(" · ") : "—";
}

export function divergenceLabel(kind: string): string {
  return DIVERGENCE_LABEL[kind] ?? kind;
}

export function subjectLabel(subject: { type: string; label: string }): string {
  return `${SUBJECT_LABEL[subject.type] ?? subject.type} · ${subject.label}`;
}

export function toggleOne(selected: ReadonlySet<string>, id: string): Set<string> {
  const next = new Set(selected);
  if (next.has(id)) next.delete(id); else next.add(id);
  return next;
}

export function toggleAll(selected: ReadonlySet<string>, proposals: CnesProposal[]): Set<string> {
  const all = proposals.length > 0 && proposals.every((p) => selected.has(p.id));
  return all ? new Set() : new Set(proposals.map((p) => p.id));
}

// Depois de reler a lista, o que sumiu sai da seleção: nunca vai num lote.
export function pruneSelection(selected: ReadonlySet<string>, proposals: CnesProposal[]): Set<string> {
  const ids = new Set(proposals.map((p) => p.id));
  return new Set([ ...selected ].filter((id) => ids.has(id)));
}

export function applySummary(r: CnesApplyResult): string {
  const applied = r.applied === 0 ? "Nenhuma proposta aplicada"
    : r.applied === 1 ? "1 proposta aplicada" : `${r.applied} propostas aplicadas`;
  const skipped = r.skipped.length;
  if (skipped === 0) return `${applied}.`;
  const why = skipped === 1 ? "1 pulada porque mudou" : `${skipped} puladas porque mudaram`;
  return `${applied}; ${why} desde a leitura — confira a lista de novo.`;
}
```

- [ ] **Step 6: Rode e veja passar**

Run: `npx vitest run src/lib/competence.test.ts src/lib/cnes.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/lib/competence.ts src/lib/competence.test.ts src/lib/cnes.ts src/lib/cnes.test.ts src/test/recordModeFixtures.ts
/opt/homebrew/bin/git commit -m "feat: add competence helpers and CNES proposal rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Tela CNES

**Files:**
- Create: `src/modules/Cnes.tsx`
- Test: `src/modules/Cnes.test.tsx`

**Interfaces:**
- Consumes: `getCnes`, `applyCnesProposals`, `CnesProposal`, `CnesDivergence` (Task 2); tudo de `src/lib/cnes.ts` e `competenceLabel` (Task 5); `describeActionError`; `SensitiveAction`.
- Produces: `Cnes({ onGoToSecurity }: { onGoToSecurity?(): void })`, ligada ao menu na Task 10.

- [ ] **Step 1: Escreva o teste**

```tsx
// src/modules/Cnes.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), getCnes: vi.fn(), applyCnesProposals: vi.fn() };
});

import * as api from "../lib/api";
import { renderWithProviders, sessionWith } from "../test/campaignFixtures";
import { cnesFixture, proposal } from "../test/recordModeFixtures";
import { Cnes } from "./Cnes";

afterEach(cleanup);
const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const confirmButton = (n: number) => screen.getByRole("button", { name: `Confirmar selecionadas (${n})` }) as HTMLButtonElement;

describe("CNES (módulo 16)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.stepUpMfa, api.getCnes, api.applyCnesProposals ]) m(fn).mockReset();
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "municipal_admin" ]));
    m(api.getCnes).mockResolvedValue(cnesFixture());
  });

  it("sem retrato importado: explica quem importa e não mostra listas", async () => {
    m(api.getCnes).mockResolvedValue(cnesFixture({ snapshot: null, proposals: [], divergences: [] }));
    renderWithProviders(<Cnes />);
    expect(await screen.findByText("nenhum retrato do CNES importado")).not.toBeNull();
    expect(screen.queryByRole("button", { name: /Confirmar selecionadas/ })).toBeNull();
  });

  it("mostra retrato, os dois lados mascarados e as divergências", async () => {
    renderWithProviders(<Cnes />);
    expect(await screen.findByText("09/2026")).not.toBeNull();
    expect(screen.getByText("UBS CENTRO · CNES 2384299")).not.toBeNull();
    expect(screen.getByText("ANA SOUZA · INE 0001234567 · CBO 225142 · CPF ***.982.247-** · CNS 7** **** **** 1234")).not.toBeNull();
    expect(screen.getByText("— (não existe no cadastro)")).not.toBeNull();
    expect(screen.getByText("provável — confira")).not.toBeNull();
    expect(screen.getByText("CBO diferente do CNES")).not.toBeNull();
    expect(screen.getByText("profissional · Bruno Lima")).not.toBeNull();
    expect(screen.getByText("CBO local 225125, CNES 225142")).not.toBeNull();
  });

  it("nada é aplicado sem confirmar: botão travado, seleção, step-up e releitura", async () => {
    m(api.applyCnesProposals).mockResolvedValue({ applied: 2, skipped: [] });
    renderWithProviders(<Cnes />);
    await screen.findByText("UBS CENTRO · CNES 2384299");
    expect(confirmButton(0).disabled).toBe(true);
    fireEvent.click(screen.getByLabelText("Selecionar todas"));
    fireEvent.click(confirmButton(2));
    expect(screen.getByText(/2 propostas serão aplicadas ao cadastro da cidade/)).not.toBeNull();
    expect(api.applyCnesProposals).not.toHaveBeenCalled();
    fireEvent.click(await screen.findByRole("button", { name: "Aplicar" }));
    await waitFor(() => expect(api.applyCnesProposals).toHaveBeenCalledWith([ "p1", "p2" ]));
    expect(await screen.findByText("2 propostas aplicadas.")).not.toBeNull();
    await waitFor(() => expect(api.getCnes).toHaveBeenCalledTimes(2));
  });

  it("proposta que mudou é pulada e dita; a seleção volta a zero", async () => {
    m(api.applyCnesProposals).mockResolvedValue({ applied: 0, skipped: [ { id: "p1", reason: "stale" } ] });
    renderWithProviders(<Cnes />);
    fireEvent.click(await screen.findByLabelText("Selecionar: UBS CENTRO · CNES 2384299"));
    m(api.getCnes).mockResolvedValue(cnesFixture({ proposals: [ proposal({ id: "p3", cnes: { name: "UBS CENTRO", cnes: "2384299" } }) ] }));
    fireEvent.click(confirmButton(1));
    fireEvent.click(await screen.findByRole("button", { name: "Aplicar" }));
    await waitFor(() => expect(api.applyCnesProposals).toHaveBeenCalledWith([ "p1" ]));
    expect(await screen.findByText("Nenhuma proposta aplicada; 1 pulada porque mudou desde a leitura — confira a lista de novo.")).not.toBeNull();
    await waitFor(() => expect(confirmButton(0).disabled).toBe(true));
  });

  it("cancelar não aplica nada e mantém a seleção", async () => {
    renderWithProviders(<Cnes />);
    fireEvent.click(await screen.findByLabelText("Selecionar: UBS CENTRO · CNES 2384299"));
    fireEvent.click(confirmButton(1));
    fireEvent.click(await screen.findByRole("button", { name: "Cancelar" }));
    expect(api.applyCnesProposals).not.toHaveBeenCalled();
    expect(confirmButton(1).disabled).toBe(false);
  });

  it("sem municipal_admin: não chama a API", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "analyst" ]));
    renderWithProviders(<Cnes />);
    expect(await screen.findByText("seu papel não permite ver o CNES")).not.toBeNull();
    expect(api.getCnes).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/Cnes.test.tsx`
Expected: FAIL — `./Cnes` não existe.

- [ ] **Step 3: A tela (`src/modules/Cnes.tsx`)**

```tsx
// src/modules/Cnes.tsx
// CNES da cidade (módulo 16; ADR 0028; spec §5; contratos §5.2): só
// municipal_admin. Mostra o retrato importado pelo operador, as propostas de
// casamento (unidade por CNES, equipe por INE, profissional por CPF/CNS) e as
// divergências. Nada se aplica sozinho: a pessoa seleciona e confirma com
// step-up; proposta que mudou desde a leitura é pulada pela API e dita aqui.
import { useEffect, useRef, useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { applyCnesProposals, getCnes, type CnesDivergence, type CnesProposal } from "../lib/api";
import { useAuth } from "../lib/auth";
import { describeActionError } from "../lib/actionErrors";
import { competenceLabel } from "../lib/competence";
import { fmtDateTime } from "../lib/format";
import {
  CNES_KEY, CONFIDENCE, PROPOSAL_ACTION_LABEL, PROPOSAL_KIND_LABEL, applySummary, describeSide, divergenceLabel,
  pruneSelection, subjectLabel, toggleAll, toggleOne
} from "../lib/cnes";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { KeyValue } from "../components/KeyValue";
import { EmptyState } from "../components/EmptyState";
import { SensitiveAction } from "../components/SensitiveAction";
import { buttonStyle, disabledButtonStyle } from "../components/formStyles";

export function Cnes({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  const { user } = useAuth();
  const queryClient = useQueryClient();
  const isAdmin = !!user && !user.operator && user.memberships.some((m) => m.role === "municipal_admin");
  const query = useQuery({ queryKey: CNES_KEY, queryFn: getCnes, enabled: isAdmin });
  const [ selected, setSelected ] = useState<Set<string>>(() => new Set());
  const [ confirming, setConfirming ] = useState<string[] | null>(null);
  const [ done, setDone ] = useState<string | null>(null);
  const outcome = useRef("");
  const proposals = query.data?.proposals;

  useEffect(() => {
    if (proposals) setSelected((current) => pruneSelection(current, proposals));
  }, [ proposals ]);

  if (!user) return null;
  if (!isAdmin) {
    return (
      <div style={column}>
        <PageHeader title="CNES" sub="unidades · equipes · profissionais" />
        <EmptyState title="seu papel não permite ver o CNES" />
      </div>
    );
  }

  const data = query.data;
  const count = selected.size;

  return (
    <div style={column}>
      <PageHeader title="CNES" sub="unidades · equipes · profissionais" />
      {query.isError && <p role="alert" style={alertStyle}>{describeActionError(query.error).message}</p>}
      {done && <p role="status" style={statusStyle}>{done}</p>}
      {query.isPending && <p className="mono" style={loadingStyle}>carregando…</p>}

      {data && data.snapshot === null && (
        <Panel title="Retrato do CNES">
          <EmptyState title="nenhum retrato do CNES importado"
            sub="o operador da plataforma importa a base mensal do CNES; depois disso, as propostas aparecem aqui" />
        </Panel>
      )}

      {data && data.snapshot && (
        <>
          <Panel title="Retrato do CNES" sub="importado pelo operador da plataforma">
            <div style={{ display: "flex", gap: 24, flexWrap: "wrap" }}>
              <KeyValue k="Competência" v={competenceLabel(data.snapshot.competence)} />
              <KeyValue k="Importado em" v={fmtDateTime(data.snapshot.imported_at)} />
            </div>
          </Panel>

          {confirming && (
            <SensitiveAction
              title="Confirmar propostas do CNES"
              description={`${confirming.length === 1 ? "1 proposta será aplicada" : `${confirming.length} propostas serão aplicadas`} ao cadastro da cidade. Proposta que mudou desde a leitura é pulada.`}
              requiresStepUp
              confirmLabel="Aplicar"
              run={async () => { outcome.current = applySummary(await applyCnesProposals(confirming)); }}
              onDone={() => {
                setConfirming(null);
                setSelected(new Set());
                setDone(outcome.current);
                void queryClient.invalidateQueries({ queryKey: CNES_KEY });
              }}
              onCancel={() => setConfirming(null)}
              onGoToSecurity={onGoToSecurity}
            />
          )}

          <Panel title="Propostas" sub="nada é aplicado sem a sua confirmação" right={
            <button type="button" disabled={count === 0 || confirming !== null}
              style={count === 0 || confirming !== null ? disabledButtonStyle : buttonStyle}
              onClick={() => { setDone(null); setConfirming([ ...selected ]); }}>
              {`Confirmar selecionadas (${count})`}
            </button>
          }>
            <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
              {data.proposals.length > 0 && (
                <label style={checkLabel}>
                  <input type="checkbox"
                    checked={data.proposals.every((p) => selected.has(p.id))}
                    onChange={() => setSelected((current) => toggleAll(current, data.proposals))} />
                  Selecionar todas
                </label>
              )}
              <DataTable<CnesProposal>
                cols={[
                  { label: "", w: "32px", render: (p) => (
                    <input type="checkbox" aria-label={`Selecionar: ${describeSide(p.cnes)}`} checked={selected.has(p.id)}
                      onChange={() => setSelected((current) => toggleOne(current, p.id))} />
                  ) },
                  { label: "Tipo", w: "1fr", render: (p) => PROPOSAL_KIND_LABEL[p.kind] ?? p.kind },
                  { label: "Ação", w: "1.2fr", render: (p) => PROPOSAL_ACTION_LABEL[p.action] ?? p.action },
                  { label: "No cadastro da cidade", w: "2fr", render: (p) => describeSide(p.local) },
                  { label: "No CNES", w: "2fr", render: (p) => describeSide(p.cnes) },
                  { label: "Casamento", w: "1fr", render: (p) =>
                    <Tag tone={CONFIDENCE[p.confidence]?.tone}>{CONFIDENCE[p.confidence]?.label ?? p.confidence}</Tag> }
                ]}
                rows={data.proposals}
                rowKey={(p) => p.id}
                empty="nenhuma proposta — o cadastro confere com o CNES"
              />
            </div>
          </Panel>

          <Panel title="Divergências" sub="corrija no cadastro ou na base do CNES">
            <DataTable<CnesDivergence>
              cols={[
                { label: "Divergência", w: "1.5fr", render: (d) => divergenceLabel(d.kind) },
                { label: "Onde", w: "1.5fr", render: (d) => subjectLabel(d.subject) },
                { label: "Detalhe", w: "2fr", render: (d) => d.detail ?? "—" }
              ]}
              rows={data.divergences}
              rowKey={(d) => `${d.kind}:${d.subject.type}:${d.subject.id}`}
              empty="nenhuma divergência"
            />
          </Panel>
        </>
      )}
    </div>
  );
}

const column = { display: "flex", flexDirection: "column" as const, gap: 16 };
const checkLabel = { display: "flex", alignItems: "center", gap: 8, fontSize: 12, color: "var(--ink2)" };
const alertStyle = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const statusStyle = { margin: 0, fontSize: 13, fontWeight: 600 };
const loadingStyle = { margin: 0, fontSize: 10.5, color: "var(--ink3)" };
```

- [ ] **Step 4: Rode e veja passar**

Run: `npx vitest run src/modules/Cnes.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/Cnes.tsx src/modules/Cnes.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the CNES screen with batch confirmation under step-up

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Regras da Produção e-SUS

**Files:**
- Create: `src/lib/production.ts`
- Modify: `src/test/recordModeFixtures.ts` (acrescenta Produção)
- Test: `src/lib/production.test.ts`

**Interfaces:**
- Consumes: `ApiError`, `FichaStatus`, `CompetenceAlert`, `FICHAS_PER_PAGE`, `LediFicha`, `Production` (Task 2); `describeActionError`; `featureDisabledKey`, `FEATURE_DISABLED_MESSAGE` (Task 1); `fmtDay` (`src/lib/audiencePhrase.ts`, existente).
- Produces (em `src/lib/production.ts`):
  - `PRODUCTION_KEY = "production"`;
  - `FICHA_STATUS: Record<string, { label: string; tone: "neutral" | "info" | "ok" | "down" }>`;
  - `canReadProduction(roles: string[]): boolean`, `canResend(roles: string[], ficha: LediFicha): boolean`;
  - `deadlinePhrase(deadlineOn: string, businessDaysLeft: number): string`;
  - `alertBanner(alert: string): { tone: "warn" | "down"; text: string } | null`;
  - `hasNextPage(page: number, total: number): boolean` — pelo `fichas_total` da API;
  - `RESEND_STALE`, `productionErrorCode(err: unknown): string | null`, `productionError(err: unknown): string`.
- Produces (em `src/test/recordModeFixtures.ts`): `ficha(overrides?)`, `productionFixture(overrides?)`.

- [ ] **Step 1: Fixtures (fim de `src/test/recordModeFixtures.ts`)**

Troque o import do topo

```ts
import type { CnesOverview, CnesProposal, IntegrationCredential, Integrations } from "../lib/api";
```

por

```ts
import type { CnesOverview, CnesProposal, IntegrationCredential, Integrations, LediFicha, Production } from "../lib/api";
```

e acrescente no fim:

```ts
export function ficha(overrides: Partial<LediFicha> = {}): LediFicha {
  return {
    id: "f1", ficha_type: "synthetic", status: "accepted", attempts: 1, last_error: null,
    created_at: "2026-10-02T13:00:00Z", accepted_at: "2026-10-02T13:01:00Z", ...overrides
  };
}

// 16/11/2026 é o 10º dia útil de novembro (02/11 Finados; 20/11 é depois).
export function productionFixture(overrides: Partial<Production> = {}): Production {
  return {
    competence: "202610", deadline_on: "2026-11-16", business_days_left: 7, alert: "attention",
    counts: { accepted: 120, rejected: 3, pending: 10, sending: 0, failed: 0 },
    rejections: [ { message: "CNS do profissional inválido", count: 3 } ],
    fichas: [
      ficha(),
      ficha({ id: "f2", status: "rejected", accepted_at: null, last_error: "CNS do profissional inválido" })
    ],
    fichas_total: 2,
    ...overrides
  };
}
```

- [ ] **Step 2: Escreva o teste**

```ts
// src/lib/production.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  FICHA_STATUS, alertBanner, canReadProduction, canResend, deadlinePhrase, hasNextPage, productionError, productionErrorCode
} from "./production";
import { ficha } from "../test/recordModeFixtures";

describe("Produção — prazo e alertas (spec §6.5)", () => {
  it("prazo: dias úteis, hoje e vencido, sem deslocar a data", () => {
    expect(deadlinePhrase("2026-11-16", 7)).toBe("prazo em 16/11/2026 · faltam 7 dias úteis");
    expect(deadlinePhrase("2026-11-16", 1)).toBe("prazo em 16/11/2026 · falta 1 dia útil");
    expect(deadlinePhrase("2026-11-16", 0)).toBe("o prazo vence hoje (16/11/2026)");
    expect(deadlinePhrase("2026-11-16", -2)).toBe("prazo vencido em 16/11/2026");
  });

  it("alerta: atenção, crítico e nenhum", () => {
    expect(alertBanner("attention")).toEqual({ tone: "warn",
      text: "Há fichas pendentes ou recusadas, e o prazo está perto. Confira as recusas abaixo." });
    expect(alertBanner("critical")).toEqual({ tone: "down",
      text: "Nenhuma ficha aceita nesta competência, e o prazo está perto. Sem envio, o repasse da cidade fica em risco." });
    expect(alertBanner("none")).toBeNull();
    expect(alertBanner("novo")).toBeNull();
  });

  it("situação da ficha em português", () => {
    expect(FICHA_STATUS.accepted).toEqual({ label: "aceita", tone: "ok" });
    expect(FICHA_STATUS.rejected).toEqual({ label: "recusada", tone: "down" });
    expect(FICHA_STATUS.failed.label).toBe("falhou — sem novas tentativas");
  });
});

describe("Produção — papéis e paginação", () => {
  it("lê municipal_admin e analyst; reenvia só o admin e só ficha recusada", () => {
    expect(canReadProduction([ "analyst" ])).toBe(true);
    expect(canReadProduction([ "municipal_admin" ])).toBe(true);
    expect(canReadProduction([ "citizen_verifier" ])).toBe(false);
    expect(canResend([ "municipal_admin" ], ficha({ status: "rejected" }))).toBe(true);
    expect(canResend([ "municipal_admin" ], ficha({ status: "failed" }))).toBe(false);
    expect(canResend([ "analyst" ], ficha({ status: "rejected" }))).toBe(false);
  });

  it("há próxima página só quando o total passa da página atual", () => {
    expect(hasNextPage(1, 51)).toBe(true);
    expect(hasNextPage(1, 50)).toBe(false);
    expect(hasNextPage(2, 100)).toBe(false);
    expect(hasNextPage(2, 101)).toBe(true);
    expect(hasNextPage(1, 0)).toBe(false);
  });
});

describe("Produção — erros", () => {
  it("not_rejected e feature_disabled têm frase própria; o resto segue o padrão", () => {
    const stale = new ApiError(409, { error: "not_rejected" }, "409");
    expect(productionErrorCode(stale)).toBe("not_rejected");
    expect(productionError(stale)).toBe("esta ficha não está mais recusada — a lista foi atualizada");
    expect(productionError(new ApiError(403, { error: "feature_disabled", feature: "ledi_export" }, "403")))
      .toBe("esta funcionalidade está desligada para a cidade");
    expect(productionError(new ApiError(403, { error: "missing_role" }, "403"))).toBe("seu papel não permite esta ação");
    expect(productionErrorCode(new Error("x"))).toBeNull();
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/lib/production.test.ts`
Expected: FAIL — `./production` não existe.

- [ ] **Step 4: Implemente (`src/lib/production.ts`)**

```ts
// src/lib/production.ts
// Produção e-SUS da cidade (módulo 16; ADR 0028; spec §6.5; contratos §5.3).
// Prazo e dias úteis vêm calculados da API (10º dia útil do mês seguinte, no
// fuso da cidade); aqui só se diz em português. `deadline_on` é data sem hora:
// formata por texto (fmtDay), nunca por new Date.
import { ApiError, FICHAS_PER_PAGE, type LediFicha } from "./api";
import { describeActionError } from "./actionErrors";
import { FEATURE_DISABLED_MESSAGE, featureDisabledKey } from "./features";
import { fmtDay } from "./audiencePhrase";

export const PRODUCTION_KEY = "production";

export const RESEND_STALE = "esta ficha não está mais recusada — a lista foi atualizada";

export const FICHA_STATUS: Record<string, { label: string; tone: "neutral" | "info" | "ok" | "down" }> = {
  pending: { label: "pendente", tone: "neutral" },
  sending: { label: "enviando", tone: "info" },
  accepted: { label: "aceita", tone: "ok" },
  rejected: { label: "recusada", tone: "down" },
  failed: { label: "falhou — sem novas tentativas", tone: "down" }
};

export function canReadProduction(roles: string[]): boolean {
  return roles.includes("municipal_admin") || roles.includes("analyst");
}

// Só ficha recusada (400 do PEC) se reenvia; `failed` já esgotou as
// tentativas automáticas e não é recusa de validação.
export function canResend(roles: string[], ficha: LediFicha): boolean {
  return roles.includes("municipal_admin") && ficha.status === "rejected";
}

export function deadlinePhrase(deadlineOn: string, businessDaysLeft: number): string {
  const day = fmtDay(deadlineOn);
  if (businessDaysLeft < 0) return `prazo vencido em ${day}`;
  if (businessDaysLeft === 0) return `o prazo vence hoje (${day})`;
  const left = businessDaysLeft === 1 ? "falta 1 dia útil" : `faltam ${businessDaysLeft} dias úteis`;
  return `prazo em ${day} · ${left}`;
}

export function alertBanner(alert: string): { tone: "warn" | "down"; text: string } | null {
  if (alert === "critical") {
    return { tone: "down",
      text: "Nenhuma ficha aceita nesta competência, e o prazo está perto. Sem envio, o repasse da cidade fica em risco." };
  }
  if (alert === "attention") {
    return { tone: "warn", text: "Há fichas pendentes ou recusadas, e o prazo está perto. Confira as recusas abaixo." };
  }
  return null;
}

// Pelo total da API (contratos §5.3): total múltiplo de 50 não abre página vazia.
export function hasNextPage(page: number, total: number): boolean {
  return page * FICHAS_PER_PAGE < total;
}

export function productionErrorCode(err: unknown): string | null {
  if (!(err instanceof ApiError)) return null;
  const code = (err.body as { error?: unknown } | null)?.error;
  return typeof code === "string" ? code : null;
}

export function productionError(err: unknown): string {
  if (productionErrorCode(err) === "not_rejected") return RESEND_STALE;
  if (featureDisabledKey(err) !== null) return FEATURE_DISABLED_MESSAGE;
  return describeActionError(err).message;
}
```

- [ ] **Step 5: Rode e veja passar**

Run: `npx vitest run src/lib/production.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/lib/production.ts src/lib/production.test.ts src/test/recordModeFixtures.ts
/opt/homebrew/bin/git commit -m "feat: add e-SUS production rules for deadline, alerts and resend

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Tela Produção e-SUS (leitura e reenvio)

**Files:**
- Create: `src/modules/Production.tsx`
- Test: `src/modules/Production.test.tsx`

**Interfaces:**
- Consumes: `getProduction`, `resendFicha`, `LediFicha` (Task 2); `hasFeature`, `featureDisabledKey` (Task 1); `competenceLabel`, `competenceOptions` (Task 5); tudo de `src/lib/production.ts` (Task 7); `todayInCity`; `useAuth().reload`; `SensitiveAction`, `KpiGrid`, `StatTile`.
- Produces: `Production({ onGoToSecurity }: { onGoToSecurity?(): void })`, ligada ao menu na Task 10.

- [ ] **Step 1: Escreva o teste**

```tsx
// src/modules/Production.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), getProduction: vi.fn(), resendFicha: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { renderWithProviders, sessionWith } from "../test/campaignFixtures";
import { ficha, productionFixture } from "../test/recordModeFixtures";
import { Production } from "./Production";

afterEach(cleanup);
const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const DISABLED = "o envio da produção ao e-SUS está desligado nesta cidade";
const withLedi = (roles: string[]) => sessionWith(roles, { features: [ "ledi_export" ] });

function resetAll() {
  for (const fn of [ api.fetchCurrentSession, api.stepUpMfa, api.getProduction, api.resendFicha ]) m(fn).mockReset();
}

describe("Produção e-SUS (módulo 16)", () => {
  beforeEach(() => {
    resetAll();
    m(api.fetchCurrentSession).mockResolvedValue(withLedi([ "municipal_admin" ]));
    m(api.getProduction).mockResolvedValue(productionFixture());
  });

  it("analista lê prazo, alerta, motivos e fichas, sem Reenviar", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(withLedi([ "analyst" ]));
    renderWithProviders(<Production />);
    expect(await screen.findByText("prazo em 16/11/2026 · faltam 7 dias úteis")).not.toBeNull();
    expect(screen.getByText("Há fichas pendentes ou recusadas, e o prazo está perto. Confira as recusas abaixo.")).not.toBeNull();
    expect(screen.getAllByText("CNS do profissional inválido").length).toBe(2);
    expect(screen.getByText("recusada")).not.toBeNull();
    expect(screen.queryByRole("button", { name: /Reenviar ficha/ })).toBeNull();
    expect(api.getProduction).toHaveBeenCalledWith(null, 1);
  });

  it("prazo vencido e alerta crítico", async () => {
    m(api.getProduction).mockResolvedValue(productionFixture({ business_days_left: -2, alert: "critical" }));
    renderWithProviders(<Production />);
    expect(await screen.findByText("prazo vencido em 16/11/2026")).not.toBeNull();
    expect(screen.getByRole("alert").textContent)
      .toBe("Nenhuma ficha aceita nesta competência, e o prazo está perto. Sem envio, o repasse da cidade fica em risco.");
  });

  it("sem ledi_export na sessão: tela desligada e nenhuma chamada", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "municipal_admin" ]));
    renderWithProviders(<Production />);
    expect(await screen.findByText(DISABLED)).not.toBeNull();
    expect(api.getProduction).not.toHaveBeenCalled();
  });

  it("403 feature_disabled com a sessão ainda dizendo ligada: mostra desligada e relê a sessão uma vez só", async () => {
    m(api.getProduction).mockRejectedValue(new ApiError(403, { error: "feature_disabled", feature: "ledi_export" }, "403"));
    renderWithProviders(<Production />);
    expect(await screen.findByText(DISABLED)).not.toBeNull();
    await waitFor(() => expect(api.fetchCurrentSession).toHaveBeenCalledTimes(2));
    await new Promise((resolve) => setTimeout(resolve, 50));
    expect(api.fetchCurrentSession).toHaveBeenCalledTimes(2);
    expect(screen.queryByText("seu papel não permite esta ação")).toBeNull();
  });

  it("paginação: próxima só quando fichas_total passa da página, fixando a competência", async () => {
    const fifty = Array.from({ length: 50 }, (_, i) => ficha({ id: `f${i}` }));
    m(api.getProduction)
      .mockResolvedValueOnce(productionFixture({ fichas: fifty, fichas_total: 51 }))
      .mockResolvedValueOnce(productionFixture({ fichas: [ ficha({ id: "f50" }) ], fichas_total: 51 }));
    renderWithProviders(<Production />);
    const next = (await screen.findByRole("button", { name: "Próxima página" })) as HTMLButtonElement;
    expect(next.disabled).toBe(false);
    expect((screen.getByRole("button", { name: "Página anterior" }) as HTMLButtonElement).disabled).toBe(true);
    fireEvent.click(next);
    await waitFor(() => expect(api.getProduction).toHaveBeenLastCalledWith("202610", 2));
    await waitFor(() => expect((screen.getByRole("button", { name: "Próxima página" }) as HTMLButtonElement).disabled).toBe(true));
    expect((screen.getByRole("button", { name: "Página anterior" }) as HTMLButtonElement).disabled).toBe(false);
  });

  it("municipal_admin reenvia a ficha recusada pelo step-up e relê", async () => {
    m(api.resendFicha).mockResolvedValue(ficha({ id: "f2", status: "pending", attempts: 2 }));
    renderWithProviders(<Production />);
    fireEvent.click(await screen.findByRole("button", { name: "Reenviar ficha f2" }));
    expect(screen.queryByRole("button", { name: "Reenviar ficha f1" })).toBeNull();
    fireEvent.click(await screen.findByRole("button", { name: "Reenviar" }));
    await waitFor(() => expect(api.resendFicha).toHaveBeenCalledWith("f2"));
    expect(await screen.findByText("Ficha reenviada para a fila. A situação muda quando o PEC responder.")).not.toBeNull();
    await waitFor(() => expect(api.getProduction).toHaveBeenCalledTimes(2));
  });

  it("409 not_rejected: frase da tela e a lista é relida", async () => {
    m(api.resendFicha).mockRejectedValue(new ApiError(409, { error: "not_rejected" }, "409"));
    renderWithProviders(<Production />);
    fireEvent.click(await screen.findByRole("button", { name: "Reenviar ficha f2" }));
    fireEvent.click(await screen.findByRole("button", { name: "Reenviar" }));
    expect(await screen.findByText("esta ficha não está mais recusada — a lista foi atualizada")).not.toBeNull();
    await waitFor(() => expect(api.getProduction).toHaveBeenCalledTimes(2));
  });

  it("papel sem leitura: não chama a API", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(withLedi([ "citizen_verifier" ]));
    renderWithProviders(<Production />);
    expect(await screen.findByText("seu papel não permite ver a produção")).not.toBeNull();
    expect(api.getProduction).not.toHaveBeenCalled();
  });
});

describe("Produção e-SUS — competência no fuso da cidade", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    // 22h30 de 31/10 em São Paulo; em UTC já é 01/11.
    vi.setSystemTime(new Date("2026-11-01T01:30:00Z"));
    resetAll();
    m(api.fetchCurrentSession).mockResolvedValue(withLedi([ "analyst" ]));
    m(api.getProduction).mockResolvedValue(productionFixture());
  });
  afterEach(() => vi.useRealTimers());

  it("às 22h30 de 31/10 a primeira opção é 10/2026, e trocar pede a outra na página 1", async () => {
    renderWithProviders(<Production />);
    await screen.findByText("prazo em 16/11/2026 · faltam 7 dias úteis");
    const select = screen.getByLabelText("Competência") as HTMLSelectElement;
    expect(select.options[0].value).toBe("202610");
    expect(select.value).toBe("202610");
    fireEvent.change(select, { target: { value: "202609" } });
    await waitFor(() => expect(api.getProduction).toHaveBeenLastCalledWith("202609", 1));
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/Production.test.tsx`
Expected: FAIL — `./Production` não existe.

- [ ] **Step 3: A tela (`src/modules/Production.tsx`)**

```tsx
// src/modules/Production.tsx
// Produção e-SUS (módulo 16; ADR 0028; spec §6.5; contratos §5.3): leitura
// para municipal_admin e analyst, só com `ledi_export` ligado na sessão.
// Reenviar ficha recusada é do municipal_admin, com step-up. Se a API disser
// `feature_disabled` com a sessão ainda dizendo ligada (o mantenedor desligou
// agora), a tela mostra "desligado" e relê a sessão UMA vez, para o menu
// esconder a tela; nunca entra em laço.
import { useEffect, useRef, useState, type ReactNode } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { getProduction, resendFicha, type LediFicha } from "../lib/api";
import { useAuth } from "../lib/auth";
import { featureDisabledKey, hasFeature } from "../lib/features";
import { competenceLabel, competenceOptions } from "../lib/competence";
import { todayInCity } from "../lib/campaigns";
import { fmtDateTime, fmtNumber } from "../lib/format";
import {
  FICHA_STATUS, PRODUCTION_KEY, RESEND_STALE, alertBanner, canReadProduction, canResend, deadlinePhrase, hasNextPage,
  productionError, productionErrorCode
} from "../lib/production";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { DataTable, type Column } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { EmptyState } from "../components/EmptyState";
import { KpiGrid } from "../components/KpiGrid";
import { StatTile } from "../components/StatTile";
import { SensitiveAction } from "../components/SensitiveAction";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../components/formStyles";

const DISABLED_TITLE = "o envio da produção ao e-SUS está desligado nesta cidade";
const DISABLED_SUB = "quem liga é a equipe do Rota Saúde, depois que o operador da plataforma e a administração da cidade completam a configuração (veja Integrações)";

export function Production({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  const { user, reload } = useAuth();
  const queryClient = useQueryClient();
  const roles = user?.memberships.map((m) => m.role) ?? [];
  const canRead = !!user && !user.operator && canReadProduction(roles);
  const switchedOn = hasFeature(user, "ledi_export");
  const [ competence, setCompetence ] = useState<string | null>(null);
  const [ page, setPage ] = useState(1);
  const [ resending, setResending ] = useState<LediFicha | null>(null);
  const [ done, setDone ] = useState<string | null>(null);
  const query = useQuery({
    queryKey: [ PRODUCTION_KEY, competence, page ],
    queryFn: () => getProduction(competence, page),
    enabled: canRead && switchedOn
  });
  const disabledByServer = query.isError && featureDisabledKey(query.error) !== null;
  const reloaded = useRef(false);

  useEffect(() => {
    if (disabledByServer && !reloaded.current) {
      reloaded.current = true;
      void reload();
    }
  }, [ disabledByServer, reload ]);

  if (!user) return null;
  if (!canRead) return <Frame><EmptyState title="seu papel não permite ver a produção" /></Frame>;
  if (!switchedOn || disabledByServer) return <Frame><EmptyState title={DISABLED_TITLE} sub={DISABLED_SUB} /></Frame>;

  const data = query.data;
  const refresh = () => void queryClient.invalidateQueries({ queryKey: [ PRODUCTION_KEY ] });
  const banner = data ? alertBanner(data.alert) : null;
  const options = competenceOptions(todayInCity(), data?.competence);

  // Página seguinte fixa a competência que está na tela: na virada do mês, a
  // "corrente" da API mudaria entre a página 1 e a 2.
  function goTo(next: number) {
    setCompetence((current) => current ?? data?.competence ?? null);
    setPage(next);
  }

  const cols: Column<LediFicha>[] = [
    { label: "Tipo", w: "1fr", render: (f) => <span className="mono">{f.ficha_type}</span> },
    { label: "Situação", w: "1.2fr", render: (f) =>
      <Tag tone={FICHA_STATUS[f.status]?.tone}>{FICHA_STATUS[f.status]?.label ?? f.status}</Tag> },
    { label: "Tentativas", w: "0.7fr", align: "right", render: (f) => <span className="mono">{fmtNumber(f.attempts)}</span> },
    { label: "Último erro", w: "2fr", render: (f) => f.last_error ?? "—" },
    { label: "Criada em", w: "1fr", render: (f) => fmtDateTime(f.created_at) },
    { label: "Aceita em", w: "1fr", render: (f) => fmtDateTime(f.accepted_at) },
    { label: "", w: "auto", align: "right", render: (f) => canResend(roles, f) && (
      <button type="button" aria-label={`Reenviar ficha ${f.id}`} style={secondaryButtonStyle}
        onClick={() => { setDone(null); setResending(f); }}>
        Reenviar
      </button>
    ) }
  ];

  return (
    <Frame right={
      <label style={selectLabel}>
        Competência
        <select value={competence ?? data?.competence ?? ""} style={inputStyle}
          onChange={(e) => { setCompetence(e.target.value); setPage(1); }}>
          {options.map((c) => <option key={c} value={c}>{competenceLabel(c)}</option>)}
        </select>
      </label>
    }>
      {query.isError && <p role="alert" style={alertStyle}>{productionError(query.error)}</p>}
      {done && <p role="status" style={statusStyle}>{done}</p>}
      {query.isPending && <p className="mono" style={loadingStyle}>carregando…</p>}

      {resending && (
        <SensitiveAction
          key={resending.id}
          title="Reenviar ficha recusada"
          description="A ficha volta para a fila e é enviada de novo ao PEC da cidade."
          requiresStepUp
          confirmLabel="Reenviar"
          run={async () => { await resendFicha(resending.id); }}
          onDone={() => {
            setResending(null);
            setDone("Ficha reenviada para a fila. A situação muda quando o PEC responder.");
            refresh();
          }}
          onCancel={() => setResending(null)}
          onGoToSecurity={onGoToSecurity}
          translateError={(err) => {
            if (productionErrorCode(err) !== "not_rejected") return null;
            refresh();
            return RESEND_STALE;
          }}
        />
      )}

      {data && (
        <>
          {banner && (
            <p role={banner.tone === "down" ? "alert" : "status"}
              style={{ ...bannerStyle, color: banner.tone === "down" ? "var(--down)" : "var(--warn)" }}>
              {banner.text}
            </p>
          )}
          <p style={deadlineStyle}>{deadlinePhrase(data.deadline_on, data.business_days_left)}</p>
          <KpiGrid min={140}>
            <StatTile label="Aceitas" value={data.counts.accepted} tone="ok" />
            <StatTile label="Recusadas" value={data.counts.rejected} tone={data.counts.rejected > 0 ? "down" : undefined} />
            <StatTile label="Pendentes" value={data.counts.pending} />
            <StatTile label="Enviando" value={data.counts.sending} />
            <StatTile label="Falharam" value={data.counts.failed} tone={data.counts.failed > 0 ? "down" : undefined} />
          </KpiGrid>

          <Panel title="Motivos de recusa" sub="agrupados pela mensagem do PEC">
            <DataTable<{ message: string; count: number }>
              cols={[
                { label: "Motivo", w: "3fr", render: (r) => r.message },
                { label: "Fichas", w: "0.7fr", align: "right", render: (r) => <span className="mono">{fmtNumber(r.count)}</span> }
              ]}
              rows={data.rejections}
              rowKey={(r) => r.message}
              empty="nenhuma recusa nesta competência"
            />
          </Panel>

          <Panel title="Fichas" sub={`competência ${competenceLabel(data.competence)} · página ${page}`}>
            <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
              <DataTable<LediFicha> cols={cols} rows={data.fichas} rowKey={(f) => f.id} empty="nenhuma ficha nesta página" />
              <div style={{ display: "flex", gap: 8, justifyContent: "flex-end" }}>
                <button type="button" disabled={page === 1} style={page === 1 ? disabledButtonStyle : secondaryButtonStyle}
                  onClick={() => goTo(page - 1)}>
                  Página anterior
                </button>
                <button type="button" disabled={!hasNextPage(page, data.fichas_total)}
                  style={hasNextPage(page, data.fichas_total) ? buttonStyle : disabledButtonStyle}
                  onClick={() => goTo(page + 1)}>
                  Próxima página
                </button>
              </div>
            </div>
          </Panel>
        </>
      )}
    </Frame>
  );
}

function Frame({ right, children }: { right?: ReactNode; children: ReactNode }) {
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Produção e-SUS" sub="envio da produção ao PEC da cidade" right={right} />
      {children}
    </div>
  );
}

const selectLabel = { display: "flex", flexDirection: "column" as const, gap: 4, fontSize: 12, color: "var(--ink2)", minWidth: 140 };
const alertStyle = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const statusStyle = { margin: 0, fontSize: 13, fontWeight: 600 };
const loadingStyle = { margin: 0, fontSize: 10.5, color: "var(--ink3)" };
const bannerStyle = { margin: 0, fontSize: 13, fontWeight: 600 };
const deadlineStyle = { margin: 0, fontSize: 12.5, color: "var(--ink2)" };
```

- [ ] **Step 4: Rode e veja passar**

Run: `npx vitest run src/modules/Production.test.tsx && npx tsc --noEmit`
Expected: PASS. Se "prazo vencido e alerta crítico" achar mais de um `alert`, confira que o erro da query só aparece com `query.isError` (aqui a query teve sucesso).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/Production.tsx src/modules/Production.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the e-SUS production screen with step-up resend

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Consultar CADSUS na validação presencial

**Files:**
- Modify: `src/lib/attendance.ts` (`MESSAGES`), `src/modules/Attendance.tsx` (`Attendance` e `Counter`)
- Create: `src/modules/attendance/CadsusCheck.tsx`
- Test: `src/modules/attendance/CadsusCheck.test.tsx`, `src/modules/Attendance.test.tsx`

**Interfaces:**
- Consumes: `cadsusLookup`, `CadsusLookupResult`, `verifyCitizen(cpf, code, extra?)` (Task 2); `hasFeature` (Task 1); `attendanceError`.
- Produces:
  - `CadsusCheck({ cpf, code, confirmed, onConfirmedChange })` e `matchLabel(v: boolean | null): string` em `src/modules/attendance/CadsusCheck.tsx`;
  - `Counter({ cadsusOn }: { cadsusOn: boolean })`: com `cadsusOn`, a validação chama `verifyCitizen(cpf, code, { cadsus_confirmed })`; sem, `verifyCitizen(cpf, code)` como antes.

- [ ] **Step 1: Teste do componente**

```tsx
// src/modules/attendance/CadsusCheck.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, cadsusLookup: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { CadsusCheck, matchLabel } from "./CadsusCheck";

afterEach(cleanup);
const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function renderCheck(confirmed = false) {
  const onConfirmedChange = vi.fn();
  render(<CadsusCheck cpf="529.982.247-25" code="123456" confirmed={confirmed} onConfirmedChange={onConfirmedChange} />);
  return onConfirmedChange;
}

describe("CadsusCheck (contratos §5.4)", () => {
  beforeEach(() => m(api.cadsusLookup).mockReset());

  it("matchLabel: confere, diverge, sem dado", () => {
    expect(matchLabel(true)).toBe("confere");
    expect(matchLabel(false)).toBe("diverge do que o cidadão declarou");
    expect(matchLabel(null)).toBe("sem dado para comparar");
  });

  it("encontrado e tudo confere: CNS mascarado e a confirmação é do atendente", async () => {
    m(api.cadsusLookup).mockResolvedValue({ found: true, cns_masked: "7** **** **** 1234", birth_date_matches: true, sex_matches: true });
    const onChange = renderCheck();
    fireEvent.click(screen.getByRole("button", { name: "Consultar CADSUS" }));
    expect(await screen.findByText("7** **** **** 1234")).not.toBeNull();
    expect(api.cadsusLookup).toHaveBeenCalledWith("529.982.247-25", "123456");
    expect(screen.getByText("Data de nascimento: confere")).not.toBeNull();
    expect(screen.getByText("Sexo: confere")).not.toBeNull();
    expect(screen.queryByText(/Há divergência/)).toBeNull();
    const box = screen.getByLabelText("Gravar o CNS do CADSUS no cadastro") as HTMLInputElement;
    expect(box.checked).toBe(false);
    expect(onChange).toHaveBeenLastCalledWith(false);
    fireEvent.click(box);
    expect(onChange).toHaveBeenLastCalledWith(true);
  });

  it("divergência: aviso, e a confirmação não vem marcada", async () => {
    m(api.cadsusLookup).mockResolvedValue({ found: true, cns_masked: "7** **** **** 1234", birth_date_matches: false, sex_matches: null });
    renderCheck();
    fireEvent.click(screen.getByRole("button", { name: "Consultar CADSUS" }));
    expect(await screen.findByText("Data de nascimento: diverge do que o cidadão declarou")).not.toBeNull();
    expect(screen.getByText("Sexo: sem dado para comparar")).not.toBeNull();
    expect(screen.getByText("Há divergência com o que o cidadão declarou. Confira no documento antes de decidir.")).not.toBeNull();
    expect((screen.getByLabelText("Gravar o CNS do CADSUS no cadastro") as HTMLInputElement).checked).toBe(false);
  });

  it("não encontrado: sem CNS e sem caixa", async () => {
    m(api.cadsusLookup).mockResolvedValue({ found: false, cns_masked: null, birth_date_matches: null, sex_matches: null });
    renderCheck();
    fireEvent.click(screen.getByRole("button", { name: "Consultar CADSUS" }));
    expect(await screen.findByText("Não encontrado no CADSUS. Siga pela conferência do documento.")).not.toBeNull();
    expect(screen.queryByLabelText("Gravar o CNS do CADSUS no cadastro")).toBeNull();
  });

  it("indisponível ou desligado: mensagem e o balcão segue", async () => {
    m(api.cadsusLookup).mockRejectedValueOnce(new ApiError(503, { error: "cadsus_unavailable" }, "503"));
    renderCheck();
    fireEvent.click(screen.getByRole("button", { name: "Consultar CADSUS" }));
    expect(await screen.findByText("CADSUS indisponível agora — siga pela conferência do documento")).not.toBeNull();
    m(api.cadsusLookup).mockRejectedValueOnce(new ApiError(403, { error: "feature_disabled", feature: "cadsus_lookup" }, "403"));
    fireEvent.click(screen.getByRole("button", { name: "Consultar CADSUS" }));
    expect(await screen.findByText("a consulta ao CADSUS foi desligada para a cidade — siga pela conferência do documento")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Teste do balcão (`src/modules/Attendance.test.tsx`)**

No `vi.mock("../lib/api", ...)` do topo, troque

```tsx
    getMyProfessional: vi.fn(), listPendingErasures: vi.fn()
```

por

```tsx
    getMyProfessional: vi.fn(), listPendingErasures: vi.fn(), cadsusLookup: vi.fn()
```

E acrescente, no fim do arquivo:

```tsx
describe("Attendance — CADSUS no balcão (módulo 16)", () => {
  const withCadsus = (): api.SessionUser => ({ ...session("citizen_verifier"), features: [ "cadsus_lookup" ] });

  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.lookupCitizen, api.verifyCitizen, api.cadsusLookup, api.listActiveUnits, api.getMyProfessional ]) {
      mocked(fn).mockReset();
    }
    mocked(api.fetchCurrentSession).mockResolvedValue(withCadsus());
    mocked(api.listActiveUnits).mockResolvedValue([]);
    mocked(api.getMyProfessional).mockResolvedValue(null);
    mocked(api.lookupCitizen).mockResolvedValue(found);
    mocked(api.verifyCitizen).mockResolvedValue(undefined);
  });

  async function search() {
    fireEvent.change(await screen.findByLabelText("CPF do cidadão (validação)"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Código de validação"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar validação" }));
    await screen.findByText("(**) *****-5432");
    fireEvent.click(screen.getByLabelText("Conferi o documento com foto e o CPF confere"));
  }

  it("sem o interruptor na sessão, não há botão e a validação manda o corpo de antes", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session("citizen_verifier"));
    renderAttendance();
    await search();
    expect(screen.queryByRole("button", { name: "Consultar CADSUS" })).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Validar cadastro" }));
    await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456"));
  });

  it("consulta, confirma e envia cadsus_confirmed: true", async () => {
    mocked(api.cadsusLookup).mockResolvedValue({ found: true, cns_masked: "7** **** **** 1234", birth_date_matches: true, sex_matches: true });
    renderAttendance();
    await search();
    fireEvent.click(screen.getByRole("button", { name: "Consultar CADSUS" }));
    expect(await screen.findByText("7** **** **** 1234")).not.toBeNull();
    fireEvent.click(screen.getByLabelText("Gravar o CNS do CADSUS no cadastro"));
    fireEvent.click(screen.getByRole("button", { name: "Validar cadastro" }));
    await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456", { cadsus_confirmed: true }));
  });

  it("CADSUS fora do ar: o balcão segue pelo documento com cadsus_confirmed: false", async () => {
    mocked(api.cadsusLookup).mockRejectedValue(new ApiError(503, { error: "cadsus_unavailable" }, "503"));
    renderAttendance();
    await search();
    fireEvent.click(screen.getByRole("button", { name: "Consultar CADSUS" }));
    expect(await screen.findByText("CADSUS indisponível agora — siga pela conferência do documento")).not.toBeNull();
    const validate = screen.getByRole("button", { name: "Validar cadastro" }) as HTMLButtonElement;
    expect(validate.disabled).toBe(false);
    fireEvent.click(validate);
    await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456", { cadsus_confirmed: false }));
  });

  it("nova busca limpa a confirmação anterior", async () => {
    mocked(api.cadsusLookup).mockResolvedValue({ found: true, cns_masked: "7** **** **** 1234", birth_date_matches: true, sex_matches: true });
    renderAttendance();
    await search();
    fireEvent.click(screen.getByRole("button", { name: "Consultar CADSUS" }));
    fireEvent.click(await screen.findByLabelText("Gravar o CNS do CADSUS no cadastro"));
    fireEvent.click(screen.getByRole("button", { name: "Cancelar" }));
    await search();
    fireEvent.click(screen.getByRole("button", { name: "Validar cadastro" }));
    await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456", { cadsus_confirmed: false }));
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/modules/attendance/CadsusCheck.test.tsx src/modules/Attendance.test.tsx`
Expected: FAIL — `./CadsusCheck` não existe e o balcão não tem o botão.

- [ ] **Step 4: Mensagens (`src/lib/attendance.ts`)**

No `MESSAGES`, troque a última linha

```ts
  invalid_neighborhood: "bairro inválido — escolha outro da lista"
```

por

```ts
  invalid_neighborhood: "bairro inválido — escolha outro da lista",
  // Módulo 16 (contratos §5.4): o CADSUS nunca trava o balcão.
  cadsus_unavailable: "CADSUS indisponível agora — siga pela conferência do documento",
  feature_disabled: "a consulta ao CADSUS foi desligada para a cidade — siga pela conferência do documento",
  not_found: "validação não encontrada — busque o cidadão de novo"
```

- [ ] **Step 5: Componente (`src/modules/attendance/CadsusCheck.tsx`)**

```tsx
// src/modules/attendance/CadsusCheck.tsx
// Consulta ao CADSUS na validação presencial (módulo 16; ADR 0028; spec §7;
// contratos §5.4). Só aparece com `cadsus_lookup` ligado. Mostra o CNS
// mascarado e se nascimento e sexo conferem com o que o cidadão declarou;
// nunca nome, mãe ou endereço. Quem decide é o atendente: a caixa começa
// desmarcada, toda consulta nova a desmarca, e CADSUS fora do ar nunca trava
// o balcão (segue pelo documento).
import { useState } from "react";
import { cadsusLookup, type CadsusLookupResult } from "../../lib/api";
import { attendanceError } from "../../lib/attendance";
import { KeyValue } from "../../components/KeyValue";
import { disabledButtonStyle, secondaryButtonStyle } from "../../components/formStyles";

export function matchLabel(value: boolean | null): string {
  if (value === true) return "confere";
  if (value === false) return "diverge do que o cidadão declarou";
  return "sem dado para comparar";
}

export function CadsusCheck({ cpf, code, confirmed, onConfirmedChange }: {
  cpf: string; code: string; confirmed: boolean; onConfirmedChange(next: boolean): void;
}) {
  const [ result, setResult ] = useState<CadsusLookupResult | null>(null);
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);

  async function consult() {
    if (busy) return;
    setBusy(true); setError(null); setResult(null);
    onConfirmedChange(false);
    try {
      setResult(await cadsusLookup(cpf, code));
    } catch (err) {
      setError(attendanceError(err));
    } finally {
      setBusy(false);
    }
  }

  const diverges = !!result?.found && (result.birth_date_matches === false || result.sex_matches === false);

  return (
    <section aria-label="Consulta ao CADSUS" style={box}>
      <div>
        <button type="button" disabled={busy} onClick={() => void consult()} style={busy ? disabledButtonStyle : secondaryButtonStyle}>
          Consultar CADSUS
        </button>
      </div>
      {error && <p role="alert" style={alertStyle}>{error}</p>}
      {result && !result.found && <p style={text}>Não encontrado no CADSUS. Siga pela conferência do documento.</p>}
      {result?.found && (
        <>
          <KeyValue k="CNS (CADSUS)" v={result.cns_masked ?? "—"} />
          <p style={text}>Data de nascimento: {matchLabel(result.birth_date_matches)}</p>
          <p style={text}>Sexo: {matchLabel(result.sex_matches)}</p>
          {diverges && (
            <p role="status" style={warnStyle}>Há divergência com o que o cidadão declarou. Confira no documento antes de decidir.</p>
          )}
          <label style={checkLabel}>
            <input type="checkbox" checked={confirmed} onChange={(e) => onConfirmedChange(e.target.checked)} />
            Gravar o CNS do CADSUS no cadastro
          </label>
        </>
      )}
    </section>
  );
}

const box = { display: "flex", flexDirection: "column" as const, gap: 8, maxWidth: 420, border: "1px solid var(--rule)",
  borderRadius: 8, padding: "8px 12px" };
const text = { margin: 0, fontSize: 12.5, color: "var(--ink2)" };
const alertStyle = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const warnStyle = { margin: 0, fontSize: 12.5, color: "var(--warn)", fontWeight: 600 };
const checkLabel = { display: "flex", alignItems: "center", gap: 8, fontSize: 12, color: "var(--ink2)" };
```

O texto "Data de nascimento: …" fica num `<p>` só com nós de texto, para o `getByText` achar a frase inteira.

- [ ] **Step 6: Balcão (`src/modules/Attendance.tsx`)**

1. Acrescente os imports, depois de `import { ErasureRequests } from "./attendance/ErasureRequests";`:

```tsx
import { CadsusCheck } from "./attendance/CadsusCheck";
import { hasFeature } from "../lib/features";
```

2. Em `Attendance`, troque

```tsx
      {canVerify && <Counter />}
```

por

```tsx
      {canVerify && <Counter cadsusOn={hasFeature(user, "cadsus_lookup")} />}
```

3. Troque a assinatura e o começo do `Counter`:

```tsx
function Counter() {
  const [ state, setState ] = useState<CounterState>("form");
```

por

```tsx
// `cadsusOn` (módulo 16): a cidade tem `cadsus_lookup` ligado na sessão.
function Counter({ cadsusOn }: { cadsusOn: boolean }) {
  const [ state, setState ] = useState<CounterState>("form");
  const [ cadsusConfirmed, setCadsusConfirmed ] = useState(false);
```

4. Em `reset()`, troque

```tsx
    setState("form"); setCpf(""); setCode(""); setChecked(false); setFound(null); setError(null);
```

por

```tsx
    setState("form"); setCpf(""); setCode(""); setChecked(false); setFound(null); setError(null);
    setCadsusConfirmed(false);
```

5. Em `search()`, troque

```tsx
      setFound(result);
      setState("found");
```

por

```tsx
      setFound(result);
      setCadsusConfirmed(false);
      setState("found");
```

6. Em `validate()`, troque

```tsx
      await verifyCitizen(cpf, code);
```

por

```tsx
      if (cadsusOn) await verifyCitizen(cpf, code, { cadsus_confirmed: cadsusConfirmed });
      else await verifyCitizen(cpf, code);
```

7. No JSX do estado `found`, logo antes de

```tsx
            <label style={{ ...labelStyle, flexDirection: "row", alignItems: "center", gap: 8 }}>
              <input type="checkbox" checked={checked} onChange={(e) => setChecked(e.target.checked)} />
```

acrescente

```tsx
            {cadsusOn && (
              <CadsusCheck cpf={cpf} code={code} confirmed={cadsusConfirmed} onConfirmedChange={setCadsusConfirmed} />
            )}
```

- [ ] **Step 7: Rode e veja passar**

Run: `npx vitest run src/modules/attendance/CadsusCheck.test.tsx src/modules/Attendance.test.tsx && npx tsc --noEmit`
Expected: PASS, incluindo o teste antigo "busca, exige a caixa do documento e valida" (sessão sem `features`, chamada com dois argumentos).

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add src/lib/attendance.ts src/modules/attendance/CadsusCheck.tsx src/modules/attendance/CadsusCheck.test.tsx src/modules/Attendance.tsx src/modules/Attendance.test.tsx
/opt/homebrew/bin/git commit -m "feat: look up CADSUS on presencial verification when the switch is on

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Menu "e-SUS" por papel e por interruptor, e rotas

**Files:**
- Modify: `src/shell/modules.ts`, `src/App.tsx`
- Test: `src/shell/modules.test.ts`

**Interfaces:**
- Consumes: `hasFeature` (Task 1); `Integrations` (Task 4), `Cnes` (Task 6), `Production` (Task 8).
- Produces:
  - `ModuleId` ganha `"integrations" | "cnes" | "production"`; grupo `"e-SUS"` depois de `"Cidade"`;
  - `navGroupsFor(user: { operator: boolean; memberships?: { role: string }[]; features?: unknown } | null)`: Integrações e CNES só para `municipal_admin` que não é operador; Produção para `municipal_admin` ou `analyst`, não operador, com `ledi_export` na sessão; grupo sem item some.

- [ ] **Step 1: Ajuste e acrescente os testes (`src/shell/modules.test.ts`)**

No teste `it("operador não vê o grupo Conta ...")`, troque

```ts
      expect(groups.length).toBe(NAV_GROUPS.length - 6);
```

por

```ts
      expect(groups.some((g) => g.label === "e-SUS")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 7);
```

No teste `it("sem sessão, esconde Equipe, Atendimento e Cidade", ...)`, troque

```ts
      expect(groups.length).toBe(NAV_GROUPS.length - 5);
```

por

```ts
      expect(groups.some((g) => g.label === "e-SUS")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 6);
```

E acrescente, antes do último `});` do arquivo:

```ts
  describe("módulo 16 na navegação", () => {
    const user = (roles: string[], features?: unknown, operator = false) =>
      ({ operator, memberships: roles.map((role) => ({ role })), features });
    const ids = (u: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(u).flatMap((g) => g.items.map((i) => i.id));

    it("Integrações e CNES só para municipal_admin", () => {
      expect(ids(user([ "municipal_admin" ]))).toEqual(expect.arrayContaining([ "integrations", "cnes" ]));
      for (const role of [ "analyst", "viewer", "citizen_verifier", "health_professional", "campaign_manager" ]) {
        expect(ids(user([ role ], [ "ledi_export" ]))).not.toContain("integrations");
        expect(ids(user([ role ], [ "ledi_export" ]))).not.toContain("cnes");
      }
      expect(labelFor("integrations")).toBe("Integrações");
      expect(labelFor("cnes")).toBe("CNES");
    });

    it("Produção e-SUS para municipal_admin e analyst, só com ledi_export ligado", () => {
      expect(ids(user([ "municipal_admin" ], [ "ledi_export" ]))).toContain("production");
      expect(ids(user([ "analyst" ], [ "ledi_export" ]))).toContain("production");
      expect(ids(user([ "municipal_admin" ]))).not.toContain("production");
      expect(ids(user([ "municipal_admin" ], [ "cadsus_lookup" ]))).not.toContain("production");
      expect(ids(user([ "viewer" ], [ "ledi_export" ]))).not.toContain("production");
      expect(labelFor("production")).toBe("Produção e-SUS");
    });

    it("grupo e-SUS some quando não sobra item", () => {
      expect(navGroupsFor(user([ "analyst" ])).some((g) => g.label === "e-SUS")).toBe(false);
      expect(navGroupsFor(user([ "analyst" ], [ "ledi_export" ])).find((g) => g.label === "e-SUS")?.items.map((i) => i.id))
        .toEqual([ "production" ]);
    });

    it("operador nunca vê e-SUS, nem com papel e interruptor", () => {
      expect(ids(user([ "municipal_admin" ], [ "ledi_export" ], true))).not.toContain("integrations");
      expect(ids(user([ "municipal_admin" ], [ "ledi_export" ], true))).not.toContain("production");
    });

    it("features fora de formato não quebra o menu", () => {
      expect(ids(user([ "municipal_admin" ], "ledi_export"))).not.toContain("production");
      expect(ids(user([ "municipal_admin" ], null))).toContain("integrations");
    });
  });
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/shell/modules.test.ts`
Expected: FAIL — não há grupo "e-SUS" nem os ids novos.

- [ ] **Step 3: `src/shell/modules.ts`**

1. Acrescente no topo do arquivo, depois do comentário inicial:

```ts
import { hasFeature } from "../lib/features";
```

2. Troque

```ts
  | "professionals" | "my-profile" | "territory" | "campaigns" | "analytics";
```

por

```ts
  | "professionals" | "my-profile" | "territory" | "campaigns" | "analytics"
  | "integrations" | "cnes" | "production";
```

3. Troque

```ts
  { label: "Cidade", items: [
    { id: "territory", label: "Território", icon: "⌖" }
  ]},
```

por

```ts
  { label: "Cidade", items: [
    { id: "territory", label: "Território", icon: "⌖" }
  ]},
  { label: "e-SUS", items: [
    { id: "integrations", label: "Integrações", icon: "⇌" },
    { id: "cnes", label: "CNES", icon: "⌗" },
    { id: "production", label: "Produção e-SUS", icon: "⇪" }
  ]},
```

4. Troque a função `navGroupsFor` inteira (da linha `export function navGroupsFor(` até o `}` final do arquivo) por:

```ts
export function navGroupsFor(
  user: { operator: boolean; memberships?: { role: string }[]; features?: unknown } | null
): NavGroupDef[] {
  const roles = user?.memberships?.map((m) => m.role) ?? [];
  const isAdmin = roles.includes("municipal_admin");
  const canAttend = isAdmin || roles.includes("citizen_verifier") || roles.includes("health_professional");
  const isProfessional = roles.includes("health_professional");
  // Módulo 12: /campaigns é do campaign_manager; o municipal_admin sem o
  // papel entra para a chave de SMS da cidade (spec 2026-09-29 §7).
  const canCampaigns = isAdmin || roles.includes("campaign_manager");
  // Módulo 14 (ADR 0025, D6/D12): Analytics é do analyst e do municipal_admin
  // da cidade; o operador, com ou sem grant, nunca (a API responde 403).
  const canAnalytics = !user?.operator && (isAdmin || roles.includes("analyst"));
  // Módulo 16 (ADR 0028; contratos §1 e §5): Integrações e CNES são do
  // municipal_admin; Produção é também do analyst e só aparece com
  // `ledi_export` ligado na sessão. O operador nunca vê o grupo.
  const canIntegrations = !user?.operator && isAdmin;
  const canProduction = !user?.operator && (isAdmin || roles.includes("analyst")) && hasFeature(user, "ledi_export");
  return NAV_GROUPS.filter((group) => {
    if (group.label === "Conta") return !user?.operator;
    if (group.label === "Equipe") return isAdmin;
    // Módulo 11: /territory recusa (403 missing_role) quem não é municipal_admin.
    if (group.label === "Cidade") return isAdmin;
    if (group.label === "Comunicação") return canCampaigns;
    if (group.label === "Análise") return canAnalytics;
    if (group.label === "Atendimento") return canAttend;
    return true;
  }).map((group) => ({
    ...group,
    items: group.items.filter((item) => {
      // Módulo 10: "Meu perfil" é do profissional; sem sessão ainda, some
      // (a API responderia 404 no_profile para quem não é profissional).
      if (item.id === "my-profile") return isProfessional;
      if (item.id === "integrations" || item.id === "cnes") return canIntegrations;
      if (item.id === "production") return canProduction;
      return true;
    })
  })).filter((group) => group.items.length > 0);
}
```

- [ ] **Step 4: Rotas (`src/App.tsx`)**

Acrescente os imports, depois de `import { MyProfile } from "./modules/MyProfile";`:

```tsx
import { Integrations } from "./modules/Integrations";
import { Cnes } from "./modules/Cnes";
import { Production } from "./modules/Production";
```

E troque

```tsx
    case "analytics":      return <Analytics />;
```

por

```tsx
    case "analytics":      return <Analytics />;
    case "integrations":   return <Integrations onGoToSecurity={() => setActive("security")} />;
    case "cnes":           return <Cnes onGoToSecurity={() => setActive("security")} />;
    case "production":     return <Production onGoToSecurity={() => setActive("security")} />;
```

- [ ] **Step 5: Rode e veja passar**

Run: `npx vitest run src/shell/modules.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/shell/modules.ts src/shell/modules.test.ts src/App.tsx
/opt/homebrew/bin/git commit -m "feat: add the e-SUS menu group gated by role and city switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Suíte, build, revisão e prova no navegador

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd apps/dashboard/.claude/mod16 && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde. A CI roda os mesmos três; `dist/` está no `.gitignore`.

```bash
/opt/homebrew/bin/git status --short
/opt/homebrew/bin/git log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e a contagem `0`. Se o symlink aparecer, **não** o adicione.

- [ ] **Step 2: Nenhum segredo nem dado sem máscara no cliente**

```bash
cd apps/dashboard/.claude/mod16
grep -rn "password" src --include=*.ts --include=*.tsx | grep -v "\.test\." 
grep -rn "console\.\(log\|info\|debug\)" src/modules/Integrations.tsx src/modules/Cnes.tsx src/modules/Production.tsx src/modules/attendance/CadsusCheck.tsx
```

Expected: `password` só em `src/lib/api.ts` (corpo do `PUT`), `src/components/SensitiveAction.tsx` (tipo do campo) e `src/modules/Integrations.tsx` (nome e tipo do campo, `values.password`); nenhum `console.*` nas telas novas.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§3.1, §3.3, §5, §6.5, §7, §9), o ADR 0028 e o arquivo de contratos (§1 e §5). Pontos de atenção:
- `features` ausente = `[]`; chave desconhecida não quebra; `403 feature_disabled` nunca vira "seu papel" e relê a sessão uma vez só;
- toda escrita (credencial, CNES, reenvio) só pelo `SensitiveAction` com `requiresStepUp`; teste de conexão e consulta ao CADSUS sem step-up;
- senha só no corpo do `PUT`, como digitada, em campo `password`, fora de qualquer `queryKey`, log ou estado depois de salvar;
- nenhuma proposta do CNES aplicada sem seleção e confirmação; `skipped` dito; seleção podada na releitura;
- `deadline_on` por `fmtDay`; competência por `todayInCity()`; paginação fixa a competência;
- CADSUS: a caixa começa desmarcada, nova consulta e nova busca desmarcam, erro nunca trava o balcão; `cadsus_confirmed` só vai com o interruptor ligado;
- menu: Integrações/CNES só `municipal_admin`; Produção `municipal_admin`/`analyst` com `ledi_export`; operador nunca.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Rode o api da branch do módulo 16 (fundação + exportador) na porta **3033**, com a semente do módulo (spec §10: Curitiba com `ibge_code` 4106902, retrato CNES fictício, credencial `cadsus` simulada) e o PEC local da prova técnica. Depois rode o Vite do worktree apontando para ele:

```bash
cd apps/dashboard/.claude/mod16 && VITE_API_PROXY_TARGET=http://localhost:3033 npx vite --port 5181 --host 0.0.0.0
```

Abra `http://curitiba.localhost:5181/dashboard/`. O usuário faz o login; não digite senha nem TOTP. Confira com screenshot:
- como `admin@curitiba.demo`, antes de ligar nada: grupo "e-SUS" com Integrações e CNES, sem Produção; em Integrações, os dois interruptores "desligada" com o que falta em frase;
- no `maintenance` (`maintenance.localhost:5177`, login de dev), o operador preenche modo/PEC no `admin` e o mantenedor liga `ledi_export` e `cadsus_lookup` para Curitiba; o usuário recarrega o dashboard e Produção aparece;
- em Integrações, o usuário cadastra a credencial `ledi` gerada no PEC local (step-up), e "Testar conexão" mostra "conexão ok"; a funcionalidade passa a "ligada e funcionando";
- em CNES, as propostas da semente aparecem mascaradas; confirmar duas pede step-up e o resumo diz quantas foram aplicadas;
- uma ficha sintética enfileirada (comando do plano do exportador) aparece em Produção como aceita, com o prazo da competência;
- em Atendimento → Balcão, com um código gerado no wpda: "Consultar CADSUS" mostra o CNS mascarado da semente simulada, a caixa vem desmarcada, e validar com a caixa marcada grava `cadsus_confirmed`;
- o mantenedor desliga `ledi_export`; na tela de Produção aberta, o próximo carregamento mostra "desligado nesta cidade" e o item some do menu.

- [ ] **Step 5:** **Pare.** O merge do dashboard só vem depois do merge do `api` (fundação e exportador), e só com autorização explícita do usuário. A ordem de entrega é contracts → api-foundation → api-exporter → maintenance/admin → dashboard (contratos §7).

---

## Nota: incorporado ao contrato

As sete divergências levantadas na escrita deste plano foram aceitas e já estão no arquivo de contratos; o plano segue o texto atual:

1. **§5.4:** `POST /attendance/cadsus_lookup` recebe `{ "cpf", "code" }` (o par é identificado por `VerificationCodeMatch`, como no lookup).
2. **§5.2:** `local`/`cnes` = `{ name, cnes?, ine?, cbo?, cpf_masked?, cns_masked? }` (CPF/CNS sempre mascarados); `detail` é string ou `null`.
3. **§5.3:** a resposta traz `fichas_total`; a paginação usa `hasNextPage(page, fichas_total)` em vez de deduzir pela página cheia.
4. **§5.4:** interruptor desligado = 403 `feature_disabled`; ligado mas não utilizável, ou serviço fora = 503 `cadsus_unavailable`.
5. **§5.4:** `cadsus_confirmed` é opcional (ausente = `false`).
6. **§5.1 e §5.3:** o 200 das escritas da cidade (`PUT credentials/:kind`, `POST …/check`, `POST fichas/:id/resend`) é o objeto puro, sem envelope.
7. **§4.3:** o exemplo de prazo passou a `2026-11-16` (10º dia útil de novembro de 2026), o mesmo das fixtures.

## Self-review

- **Cobertura da spec:**
  - §3.1 interruptores na sessão, telas escondidas, `403 feature_disabled`: Tasks 1, 8 e 10;
  - §3.3 Integrações (estado, cadastrar/trocar com step-up, testar conexão, `GET` sem segredo): Tasks 2, 3 e 4; modo/IBGE/PEC só leitura: Task 4;
  - §5 CNES (propostas, divergências, confirmar item ou lote com step-up, mascarado): Tasks 2, 5 e 6;
  - §6.5 Produção (aceitas, recusadas com motivos, pendentes, falhas, prazo, alertas, reenviar): Tasks 2, 7 e 8; leitura do `analyst` (contratos §5.3): Tasks 7, 8 e 10;
  - §7 CADSUS na validação presencial (botão com o interruptor, divergência para o atendente decidir, indisponível segue pelo documento): Tasks 2 e 9;
  - §9 "Front" do dashboard (Integrações, CNES, Produção, CADSUS no balcão) e prova no navegador: Tasks 4, 6, 8, 9 e 11.
- **Placeholders:** nenhum. As Tasks 1, 2, 4, 9 e 10 alteram arquivos existentes por trechos exatos (de/para); as telas novas vêm inteiras.
- **Consistência de nomes:**
  - `sessionFeatures`, `hasFeature`, `featureDisabledKey`, `featureLabel`, `FEATURE_DISABLED_MESSAGE` saem da Task 1 e são usados nas Tasks 4, 7, 8, 9 e 10;
  - `getIntegrations`, `setIntegrationCredential`, `checkIntegrationCredential`, `getCnes`, `applyCnesProposals`, `getProduction`, `resendFicha`, `cadsusLookup`, `verifyCitizen(cpf, code, extra?)`, `FICHAS_PER_PAGE` e os tipos saem da Task 2 e são usados nas Tasks 3 a 9;
  - `INTEGRATIONS_KEY`, `recordModeLabel`, `credentialLabel`, `checkLabel`, `setSummary`, `checkSummary`, `missingPhrase`, `featureState`, `integrationsError` saem da Task 3 e são usados na Task 4;
  - `competenceLabel`, `competenceOptions` (Task 5) são usados nas Tasks 6 e 8; `CNES_KEY`, `describeSide`, `toggleOne`, `toggleAll`, `pruneSelection`, `applySummary` (Task 5) na Task 6;
  - `PRODUCTION_KEY`, `FICHA_STATUS`, `canReadProduction`, `canResend`, `deadlinePhrase`, `alertBanner`, `hasNextPage`, `RESEND_STALE`, `productionErrorCode`, `productionError` saem da Task 7 e são usados na Task 8;
  - fixtures: `credential`/`integrationsFixture` (Task 3), `proposal`/`cnesFixture` (Task 5), `ficha`/`productionFixture` (Task 7); `sessionWith`/`renderWithProviders` (existentes, módulo 12).
- **Review Focus:** cada um dos cinco itens tem teste na task dona:
  - 1: Tasks 1 e 8;
  - 2: Tasks 2 e 4;
  - 3: Tasks 5 e 6;
  - 4: Tasks 5, 7 e 8;
  - 5: Task 9.
