# Módulo 15 — Triagem direcionada por perfil (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Telas do módulo 15 no painel da cidade. O autor monta a oferta (`offer`) e as sugestões (`suggestions`) do protocolo num painel ao lado do JSON, com um construtor visual de condições, vê as perguntas no visual do wpda e testa um perfil no simulador. O `municipal_admin` pausa, ordena, restringe e define o período de cada protocolo na aba "Catálogo de triagens", com step-up; os outros papéis de protocolo só leem. O balcão de validação presencial passa a conferir data de nascimento e sexo no documento.

**Architecture:** O cliente HTTP novo vai para `src/lib/api.ts`, com os tipos do contrato. As regras ficam fora do React, em funções puras: modelo de condição e campos por contexto em `src/lib/condition.ts`, frase em português em `src/lib/conditionPhrase.ts`, leitura e escrita de `offer`/`suggestions` em `src/lib/offer.ts`, perfil em `src/lib/profile.ts` e regras do catálogo em `src/lib/triageCatalog.ts`. O `ConditionBuilder` é um componente só, usado na elegibilidade, nas sugestões e na restrição da cidade. O JSON do editor continua a fonte: o painel "Oferta e sugestões" lê a definição a cada render e devolve uma definição nova, no mesmo padrão do `AnalyticQuestions` do módulo 14. O step-up é sempre do `SensitiveAction`, que já existe.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("../lib/api")` e `vi.stubGlobal("fetch")`), Vite 5 (proxy de dev).

**Spec:** `docs/superpowers/specs/2026-10-05-module-15-triage-catalog-design.md` (§7 é deste plano; §4, §5.4 e §6.2 dão o contexto), `docs/adr/0027.md` e o arquivo de contratos `docs/superpowers/plans/2026-10-05-module-15-triage-catalog-contracts.md` (§1, §2 e §4 são deste plano). Os tipos da Task 1 copiam o contrato literalmente. O plano do api precisa estar **mergeado antes** do merge deste.

## Global Constraints

- Rotas usadas (sessão municipal, banco da cidade):
  - `GET /triage_catalog` → `{ offers: TriageOffer[] }`, para `protocol_author`, `protocol_reviewer` e `municipal_admin`;
  - `PUT /triage_catalog/:protocol_name` `{ enabled, position, restriction, available_from, available_until }` → `{ offer }`, só `municipal_admin`, com step-up;
  - `POST /authoring/protocols/simulate_offer` `{ definition, profile: { age, sex, neighborhood_id }, answers?, outcome? }` → `{ eligible, eligibility_text, suggestions: [{ protocol, matches }], errors }`, sem gravar nada;
  - `POST /attendance/lookup` (existente) devolve `citizen.profile` (formato 3.1 do contrato, ou `null`);
  - `POST /attendance/verifications` (existente) ganha `birth_date` e `sex` obrigatórios e `gender_identity` opcional (`null` = não informado).
- Uma entrada nova no proxy do Vite: `/triage_catalog`. `/authoring` e `/attendance` já existem.
- Step-up: a API responde 401 `{ error: "mfa_required" }` (`apps/api/app/controllers/concerns/mfa_step_up.rb:17`, contratos §4.2), que o `SensitiveAction` já trata.
- `simulate_offer` com definição que falha no gate responde 200 com `eligible: false`, `suggestions: []` e `errors`, nunca 422.
- Validação presencial: o lookup devolve o perfil do par do código em `citizen.profile`; recusas 422 `invalid_birth_date`, `invalid_sex`, `invalid_gender_identity`. `position` do catálogo é inteiro ≥ 1.
- Recusas do `PUT`: 404 `unknown_protocol`; 422 `invalid_restriction`, `invalid_period` e `invalid_position`. Nenhuma traz `message`, então a tela traduz pelo código.
- Variáveis por lugar (contrato §1):
  - `offer.eligibility`: `profile.age`, `profile.sex`;
  - `suggestions[].when`: `profile.age`, `profile.sex`, `outcome.tier`, `outcome.score`, `outcome.priority` e ids de passo do protocolo;
  - restrição do catálogo: `profile.age`, `profile.sex`, `citizen.neighborhood_id` (comparado com `in`).
- Operadores do construtor: idade e números `≥` (`gte`), `≤` (`lte`), entre (`all` de `gte` + `lte`) e igual; sexo, classificação, opções de lista e bairro "é um de" (`in`); resposta sim/não "é" (`eq` com `"true"`/`"false"`). NÃO por linha ou por grupo; grupos E (`all`) e OU (`any`).
- `profile.sex` ∈ `female` | `male`. `gender_identity` ∈ `cis_woman` | `cis_man` | `trans_woman` | `trans_man` | `travesti` | `non_binary` | `other`, ou `null`. Rótulos (contrato §2, literal): Mulher cis, Homem cis, Mulher trans, Homem trans, Travesti, Não binária, Outra; "Prefiro não informar" = `null`.
- `birth_date`: `YYYY-MM-DD`, não futura, idade ≤ 130, conferida no fuso da cidade (`todayInCity`).
- Schema `protocols-v1.4.0`: `offer.title` 1–60 caracteres, `offer.summary` 1–200, `offer.retake_after_days` inteiro de 1 a 3650, no máximo 10 `suggestions`, e `suggestions[].protocol` casa com `^[a-z][a-z0-9-]+$` (o mesmo padrão do `name`). Campo opcional vazio sai **ausente** do JSON, nunca `""` nem `null`. `offer` vazio sai inteiro.
- Contadores: últimos 30 dias, cada valor `null` quando entre 1 e 4. `null` aparece como "< 5" (o `SUPPRESSED_LABEL` que os painéis já usam), nunca como 0 nem "—".
- A definição do protocolo é `unknown` no dashboard (`src/lib/editor.ts`), e o `contracts` não publica tipos TS: `offer` e `suggestions` são tipados localmente em `src/lib/offer.ts`.
- O visual do wpda é reproduzido, nunca importado. Os tokens reais estão em `apps/wpda/src/theme/tokens.ts` e `global.css`, iguais aos do dashboard; `contracts/design-tokens/` é só um README.
- Interface tem ciclo próprio: use os componentes e estilos existentes (`Panel`, `DataTable`, `Tag`, `SegmentedControl`, `SensitiveAction`, `formStyles`), sem redesign.
- O `admin` consome os mesmos `/admin/api/*` e não muda neste módulo.
- Testes que dependem de "hoje" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(...)` em `beforeEach`, e `vi.useRealTimers()` em `afterEach`. Funções puras recebem `today` como argumento.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`.

## Ambiente de execução

- Crie o worktree a partir de `origin/main` e ligue o `node_modules` do checkout principal. `/.claude/` já está no `.gitignore` do dashboard.

  ```bash
  cd apps/dashboard
  /opt/homebrew/bin/git fetch origin
  /opt/homebrew/bin/git worktree add .claude/mod15 -b feat/mod-15-triage-catalog origin/main
  ln -s ../../node_modules .claude/mod15/node_modules
  ```

- Testes (script `test` = `vitest run`):

  ```bash
  cd apps/dashboard/.claude/mod15 && npx vitest run <arquivos>
  ```

- Checagem de tipos antes de cada commit (script `typecheck` = `tsc --noEmit`):

  ```bash
  cd apps/dashboard/.claude/mod15 && npx tsc --noEmit
  ```

- Base do ambiente de teste:
  - o `vitest.config.ts` já fixa `TZ=America/Sao_Paulo`;
  - não há `setupFiles` nem jest-dom (use `toBeTruthy`, `not.toBeNull()` e `toBeNull()`);
  - `globals: false`, então todo teste de componente chama `afterEach(cleanup)`.
- O `SensitiveAction` só mostra o botão de confirmar depois que o `AuthProvider` carregou a sessão. Nos testes, procure o botão de confirmar com `findByRole`, nunca com `getByRole`.
- O api do módulo 15 roda em dev na porta **3032**; a prova no navegador aponta o proxy para ela (Task 14).

## Review Focus

1. **Regra escrita à mão fora do subconjunto do construtor** (`gt`, mapa legado `{passo: valor}`, `not` duplo, grupo dentro de subgrupo). O painel não pode reescrever nem apagar a regra ao abrir: mostra "Regra avançada" com a frase, sem controles, e o JSON fica intacto. Testes:
   - Task 5, "regra avançada: só a frase, sem controles, e nada é emitido";
   - Task 7, "regra avançada no JSON: painel em leitura e o texto não muda".
2. **Linha incompleta durante a digitação** (valor vazio, "entre" com início ≥ fim, nenhuma opção marcada). Ela não pode entrar no JSON como árvore inválida, e também não pode sumir da tela porque o JSON voltou sem ela. Testes:
   - Task 3, "linhas incompletas não entram na árvore";
   - Task 5, "linha incompleta continua na tela e não emite nada".
3. **Restrição com bairro desativado depois.** O id que sobrou na regra aparece marcado como "(bairro inativo)", dá para desmarcar, e a opção some ao desmarcar; a frase diz "(bairro inativo)". Testes:
   - Task 4, "bairro inativo e bairro desconhecido";
   - Task 5, "bairro inativo marcado aparece e sai ao desmarcar".
4. **Contador suprimido.** `null` é "entre 1 e 4", não "zero" nem "sem dado": aparece "< 5", com a dica. Testes:
   - Task 10, "contador nulo é < 5, zero é 0";
   - Task 11, "contadores com supressão".
5. **Data de nascimento nas bordas.** Aniversário hoje conta o ano novo; 29 de fevereiro em ano comum; data futura e idade acima de 130 são recusadas antes da API; "hoje" é o dia da cidade. Testes:
   - Task 2, "idade na borda do aniversário" e "data de nascimento inválida";
   - Task 13, "data futura trava a validação".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `vite.config.ts`, `README.md`, `src/lib/api.ts` | proxy de `/triage_catalog`; tipos do contrato; cliente do catálogo e do simulador | 1 |
| `src/lib/profile.ts` | rótulos de sexo e identidade de gênero, idade, validação da data de nascimento, perfil em frase | 2 |
| `src/lib/condition.ts`, `src/test/conditionFixtures.ts` | campos por contexto, modelo de UI, árvore ↔ modelo, edição do modelo | 3 |
| `src/lib/conditionPhrase.ts` | qualquer árvore em frase em português | 4 |
| `src/modules/protocols/ConditionBuilder.tsx` | construtor visual (grupos, linhas, NÃO, frase, regra avançada) | 5 |
| `src/lib/offer.ts` | leitura e escrita de `offer` e `suggestions` na definição | 6 |
| `src/modules/protocolEditor/OfferPanel.tsx`, `src/modules/ProtocolEditor.tsx` | painel "Oferta e sugestões" ao lado do JSON | 7 |
| `src/modules/protocolEditor/wpdaLook.ts`, `QuestionPreview.tsx`, `ProtocolEditor.tsx` | perguntas no visual do wpda | 8 |
| `src/modules/protocolEditor/OfferSimulator.tsx`, `ProtocolEditor.tsx` | simulador de perfil | 9 |
| `src/lib/triageCatalog.ts`, `src/test/triageCatalogFixtures.ts` | papéis, situação, período, contador, formulário, payload, recusas | 10 |
| `src/modules/protocols/TriageCatalogTab.tsx`, `src/modules/Protocols.tsx` | aba "Catálogo de triagens" (leitura) | 11 |
| `src/modules/protocols/TriageOfferForm.tsx`, `TriageCatalogTab.tsx` | edição com step-up | 12 |
| `src/lib/api.ts`, `src/lib/attendance.ts`, `src/modules/attendance/ProfileCheck.tsx`, `src/modules/Attendance.tsx` | perfil conferido na validação presencial | 13 |
| — | suíte, build, revisão e prova no navegador | 14 |

---

### Task 1: Cliente do catálogo e do simulador, tipos do contrato e proxy

**Files:**
- Modify: `vite.config.ts`, `README.md`, `src/lib/api.ts` (fim do arquivo)
- Test: `src/lib/api.triageCatalog.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `ApiError` e `AUTHORING_BASE`, que já estão em `src/lib/api.ts`.
- Produces (em `src/lib/api.ts`):
  - tipos `ConditionTree = Record<string, unknown>`, `Sex = "female" | "male"`, `GenderIdentity` (os 7 valores), `CitizenProfile { birth_date; sex; gender_identity: GenderIdentity | null; profile_source: "declared" | "verified" }`;
  - `TriageOfferCounters { offered; started; completed; from_suggestion }` (cada um `number | null`), `TriageOffer` (item 4.1), `TriageOfferFields` (corpo 4.2);
  - `listTriageCatalog(): Promise<TriageOffer[]>`, `updateTriageOffer(protocolName, fields): Promise<TriageOffer>`;
  - `SimulateProfile { age: number; sex: Sex; neighborhood_id: string | null }`, `SimulateOutcome { tier?; score?; priority? }`, `SimulateOfferInput`, `SimulateOfferResult { eligible; eligibility_text: string | null; suggestions: { protocol; matches }[]; errors: string[] }`;
  - `simulateOffer(input): Promise<SimulateOfferResult>` — definição inválida volta 200 com `eligible: false`, `suggestions: []` e `errors` (nunca 422); qualquer erro HTTP é exceção.

- [ ] **Step 1: Escreva o teste do cliente**

```ts
// src/lib/api.triageCatalog.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import { ApiError, listTriageCatalog, simulateOffer, updateTriageOffer, type TriageOffer } from "./api";

afterEach(() => vi.unstubAllGlobals());

function stub(body: unknown, status = 200) {
  const fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
    new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fn);
  return fn;
}
const call = (fn: ReturnType<typeof stub>, i = 0) => fn.mock.calls[i] as [ string, RequestInit ];

const OFFER: TriageOffer = {
  protocol_name: "saude-do-idoso", title: "Saúde do idoso", active_version: 3,
  eligibility: { gte: [ "profile.age", 60 ] }, retake_after_days: 365, configured: true,
  enabled: true, position: 2, restriction: null, available_from: null, available_until: "2026-12-31",
  counters: { offered: 120, started: 40, completed: null, from_suggestion: 6 }
};
const FIELDS = { enabled: false, position: 2, restriction: null, available_from: null, available_until: null };

describe("cliente do catálogo de triagens", () => {
  it("lista desembrulha { offers }, manda o cookie e mantém o null dos contadores", async () => {
    const fn = stub({ offers: [ OFFER ] });
    const offers = await listTriageCatalog();
    expect(offers[0].title).toBe("Saúde do idoso");
    expect(offers[0].counters.completed).toBeNull();
    expect(call(fn)[0]).toBe("/triage_catalog");
    expect(call(fn)[1].credentials).toBe("include");
  });

  it("grava com PUT no nome escapado e desembrulha { offer }", async () => {
    const fn = stub({ offer: OFFER });
    expect((await updateTriageOffer("saude/idoso", FIELDS)).protocol_name).toBe("saude-do-idoso");
    expect(call(fn)[0]).toBe("/triage_catalog/saude%2Fidoso");
    expect(call(fn)[1].method).toBe("PUT");
    expect(JSON.parse(call(fn)[1].body as string)).toEqual(FIELDS);
  });

  it("recusa do PUT vira ApiError com o código no corpo", async () => {
    stub({ error: "invalid_period" }, 422);
    const err = await updateTriageOffer("saude-do-idoso", FIELDS).catch((e: unknown) => e);
    expect(err).toBeInstanceOf(ApiError);
    expect((err as ApiError).body).toEqual({ error: "invalid_period" });
  });
});

describe("simulador de oferta", () => {
  it("manda definição, perfil, respostas e resultado para /authoring/protocols/simulate_offer", async () => {
    const result = { eligible: true, eligibility_text: "idade ≥ 60", suggestions: [ { protocol: "x-y", matches: true } ], errors: [] };
    const fn = stub(result);
    const input = {
      definition: { name: "saude-do-idoso" }, profile: { age: 62, sex: "female" as const, neighborhood_id: null },
      answers: { q1: "true" }, outcome: { tier: "alta", score: 17 }
    };
    expect(await simulateOffer(input)).toEqual(result);
    expect(call(fn)[0]).toBe("/authoring/protocols/simulate_offer");
    expect(call(fn)[1].method).toBe("POST");
    expect(JSON.parse(call(fn)[1].body as string)).toEqual(input);
  });

  it("definição inválida vem como 200 com os erros do gate", async () => {
    const invalid = { eligible: false, eligibility_text: null, suggestions: [], errors: [ "offer.eligibility usa outcome.score" ] };
    stub(invalid);
    expect(await simulateOffer({ definition: {}, profile: { age: 30, sex: "male", neighborhood_id: null } })).toEqual(invalid);
  });

  it("erro HTTP é exceção", async () => {
    stub({ error: "forbidden" }, 403);
    await expect(simulateOffer({ definition: {}, profile: { age: 30, sex: "male", neighborhood_id: null } }))
      .rejects.toBeInstanceOf(ApiError);
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/api.triageCatalog.test.ts`
Expected: FAIL — `listTriageCatalog`, `updateTriageOffer` e `simulateOffer` não existem.

- [ ] **Step 3: Implemente o cliente (fim de `src/lib/api.ts`)**

```ts
// ─── Catálogo de triagens (módulo 15, ADR 0027; contratos §2 e §4) ───────────
// /triage_catalog tem prefixo próprio porque /protocols/:name já captura
// qualquer segmento. Os contadores vêm `null` quando ficam entre 1 e 4
// (supressão do ADR 0025): quem desenha nunca os trata como zero.
const TRIAGE_CATALOG_BASE = import.meta.env.VITE_TRIAGE_CATALOG_BASE || "/triage_catalog";

// Árvore da linguagem de condição (ADR 0009). A tela nunca a avalia: só a
// monta, a descreve e a manda.
export type ConditionTree = Record<string, unknown>;

export type Sex = "female" | "male";
export type GenderIdentity =
  | "cis_woman" | "cis_man" | "trans_woman" | "trans_man" | "travesti" | "non_binary" | "other";
export interface CitizenProfile {
  birth_date: string;
  sex: Sex;
  gender_identity: GenderIdentity | null;
  profile_source: "declared" | "verified";
}

export interface TriageOfferCounters {
  offered: number | null; started: number | null; completed: number | null; from_suggestion: number | null;
}
export interface TriageOffer {
  protocol_name: string;
  title: string;
  active_version: number;
  eligibility: ConditionTree | null;
  retake_after_days: number | null;
  // false = sem linha em triage_offers: enabled/position/restriction/período vêm null.
  configured: boolean;
  enabled: boolean | null;
  position: number | null;
  restriction: ConditionTree | null;
  available_from: string | null;
  available_until: string | null;
  counters: TriageOfferCounters;
}
export interface TriageOfferFields {
  enabled: boolean;
  position: number;
  restriction: ConditionTree | null;
  available_from: string | null;
  available_until: string | null;
}

export async function listTriageCatalog(): Promise<TriageOffer[]> {
  return (await jsonFetch<{ offers: TriageOffer[] }>(TRIAGE_CATALOG_BASE)).offers;
}

// Só municipal_admin, com step-up: quem trata 401 mfa_required é o SensitiveAction.
export async function updateTriageOffer(protocolName: string, fields: TriageOfferFields): Promise<TriageOffer> {
  const body = await jsonFetch<{ offer: TriageOffer }>(`${TRIAGE_CATALOG_BASE}/${encodeURIComponent(protocolName)}`, {
    method: "PUT", body: JSON.stringify(fields)
  });
  return body.offer;
}

export interface SimulateProfile { age: number; sex: Sex; neighborhood_id: string | null }
export interface SimulateOutcome { tier?: string; score?: number; priority?: number }
export interface SimulateOfferInput {
  definition: unknown;
  profile: SimulateProfile;
  answers?: Record<string, string>;
  outcome?: SimulateOutcome;
}
export interface SimulateOfferResult {
  eligible: boolean;
  // Só para conferência; a frase da tela é a do construtor (contratos §4.3).
  eligibility_text: string | null;
  suggestions: Array<{ protocol: string; matches: boolean }>;
  errors: string[];
}

// Não grava nada. Definição que falha no gate responde 200 com
// `eligible: false`, `suggestions: []` e os erros (contratos §4.3), nunca 422.
export async function simulateOffer(input: SimulateOfferInput): Promise<SimulateOfferResult> {
  return jsonFetch<SimulateOfferResult>(`${AUTHORING_BASE}/simulate_offer`, {
    method: "POST", body: JSON.stringify(input)
  });
}
```

- [ ] **Step 4: Proxy e README**

Em `vite.config.ts`, acrescente a linha do comentário depois de `//   /campaigns  → ...`:

```ts
//   /triage_catalog → catálogo de triagens da cidade (módulo 15; leitura para papéis de protocolo, escrita do municipal_admin).
```

e a entrada no `proxy`, depois de `"/campaigns": proxy(TARGET)` (ponha a vírgula na linha anterior):

```ts
      "/campaigns": proxy(TARGET),
      "/triage_catalog": proxy(TARGET)
```

Em `README.md`, troque `` `/territory` e `/campaigns` para `` por `` `/territory`, `/campaigns` e `/triage_catalog` para ``.

- [ ] **Step 5: Rode e veja passar**

Run: `npx vitest run src/lib/api.triageCatalog.test.ts && npx tsc --noEmit`
Expected: PASS, sem erro de tipo.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/lib/api.ts src/lib/api.triageCatalog.test.ts vite.config.ts README.md
/opt/homebrew/bin/git commit -m "feat: add triage catalog and offer simulator client

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Perfil — rótulos, idade e data de nascimento

**Files:**
- Create: `src/lib/profile.ts`
- Test: `src/lib/profile.test.ts`

**Interfaces:**
- Consumes: `CitizenProfile`, `GenderIdentity`, `Sex` (Task 1); `fmtDay` de `src/lib/audiencePhrase.ts`.
- Produces:
  - `SEX_OPTIONS: { value: Sex; label: string }[]` ("Feminino", "Masculino");
  - `GENDER_IDENTITY_OPTIONS: { value: GenderIdentity; label: string }[]`, `NO_GENDER_IDENTITY = "Prefiro não informar"`, `MAX_AGE = 130`;
  - `sexLabel(sex: string): string` (minúsculo, para frase), `genderIdentityLabel(value: string | null): string`;
  - `ageOn(birthDate: string, today: string): number | null`;
  - `birthDateProblem(value: string, today: string): string | null`;
  - `describeProfile(profile: CitizenProfile | null, today: string): string`.

- [ ] **Step 1: Escreva o teste**

```ts
// src/lib/profile.test.ts
import { describe, expect, it } from "vitest";
import {
  GENDER_IDENTITY_OPTIONS, SEX_OPTIONS, ageOn, birthDateProblem, describeProfile, genderIdentityLabel, sexLabel
} from "./profile";

const TODAY = "2026-10-05";

describe("rótulos do perfil", () => {
  it("sexo tem só os dois valores do contrato", () => {
    expect(SEX_OPTIONS.map((o) => o.value)).toEqual([ "female", "male" ]);
    expect(sexLabel("female")).toBe("feminino");
    expect(sexLabel("male")).toBe("masculino");
  });

  it("identidade de gênero usa os rótulos do contrato, e null é 'não informada'", () => {
    expect(GENDER_IDENTITY_OPTIONS.map((o) => o.label)).toEqual([
      "Mulher cis", "Homem cis", "Mulher trans", "Homem trans", "Travesti", "Não binária", "Outra"
    ]);
    expect(genderIdentityLabel("non_binary")).toBe("Não binária");
    expect(genderIdentityLabel(null)).toBe("não informada");
  });
});

describe("idade na borda do aniversário", () => {
  it("aniversário hoje já conta o ano novo; amanhã ainda não", () => {
    expect(ageOn("1966-10-05", TODAY)).toBe(60);
    expect(ageOn("1966-10-06", TODAY)).toBe(59);
  });

  it("29 de fevereiro faz aniversário em 1º de março no ano comum", () => {
    expect(ageOn("2000-02-29", "2025-02-28")).toBe(24);
    expect(ageOn("2000-02-29", "2025-03-01")).toBe(25);
  });

  it("data que não existe não tem idade", () => {
    expect(ageOn("2026-02-30", TODAY)).toBeNull();
    expect(ageOn("02/04/1963", TODAY)).toBeNull();
  });
});

describe("data de nascimento inválida", () => {
  it("vazia, inexistente, futura e acima de 130 anos são recusadas", () => {
    expect(birthDateProblem("", TODAY)).toBe("informe a data de nascimento");
    expect(birthDateProblem("2026-13-01", TODAY)).toBe("data de nascimento inválida");
    expect(birthDateProblem("2026-10-06", TODAY)).toBe("a data de nascimento não pode ser no futuro");
    expect(birthDateProblem("1895-10-04", TODAY)).toBe("idade acima de 130 anos — confira a data");
  });

  it("hoje e exatamente 130 anos são aceitos", () => {
    expect(birthDateProblem(TODAY, TODAY)).toBeNull();
    expect(birthDateProblem("1896-10-05", TODAY)).toBeNull();
  });
});

describe("perfil em frase", () => {
  it("mostra data sem deslocar o dia, idade, sexo e identidade", () => {
    expect(describeProfile({ birth_date: "1963-04-02", sex: "female", gender_identity: "cis_woman", profile_source: "declared" }, TODAY))
      .toBe("nascimento 02/04/1963 (63 anos) · sexo feminino · identidade de gênero Mulher cis");
  });

  it("um ano no singular, e sem perfil diz isso", () => {
    expect(describeProfile({ birth_date: "2025-10-01", sex: "male", gender_identity: null, profile_source: "declared" }, TODAY))
      .toBe("nascimento 01/10/2025 (1 ano) · sexo masculino · identidade de gênero não informada");
    expect(describeProfile(null, TODAY)).toBe("sem perfil declarado");
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/profile.test.ts`
Expected: FAIL — `./profile` não existe.

- [ ] **Step 3: Implemente**

```ts
// src/lib/profile.ts
// Perfil do par (módulo 15; ADR 0027; contratos §2): rótulos e regras de data
// usados pelo balcão de validação, pelo simulador e pelo construtor de
// condições. A elegibilidade olha só o sexo, nunca a identidade de gênero.
// Datas são strings YYYY-MM-DD comparadas como texto: nenhum fuso desloca o dia.
import type { CitizenProfile, GenderIdentity, Sex } from "./api";
import { fmtDay } from "./audiencePhrase";

export const SEX_OPTIONS: { value: Sex; label: string }[] = [
  { value: "female", label: "Feminino" },
  { value: "male", label: "Masculino" }
];

export const GENDER_IDENTITY_OPTIONS: { value: GenderIdentity; label: string }[] = [
  { value: "cis_woman", label: "Mulher cis" },
  { value: "cis_man", label: "Homem cis" },
  { value: "trans_woman", label: "Mulher trans" },
  { value: "trans_man", label: "Homem trans" },
  { value: "travesti", label: "Travesti" },
  { value: "non_binary", label: "Não binária" },
  { value: "other", label: "Outra" }
];

export const NO_GENDER_IDENTITY = "Prefiro não informar";
export const MAX_AGE = 130;

export function sexLabel(sex: string): string {
  return SEX_OPTIONS.find((o) => o.value === sex)?.label.toLowerCase() ?? sex;
}

export function genderIdentityLabel(value: string | null): string {
  if (value === null) return "não informada";
  return GENDER_IDENTITY_OPTIONS.find((o) => o.value === value)?.label ?? value;
}

const ISO_DAY = /^(\d{4})-(\d{2})-(\d{2})$/;

function dayParts(date: string): [ number, number, number ] | null {
  const m = ISO_DAY.exec(date);
  if (!m) return null;
  const [ y, mo, d ] = [ Number(m[1]), Number(m[2]), Number(m[3]) ];
  const probe = new Date(Date.UTC(y, mo - 1, d));
  if (probe.getUTCFullYear() !== y || probe.getUTCMonth() !== mo - 1 || probe.getUTCDate() !== d) return null;
  return [ y, mo, d ];
}

export function ageOn(birthDate: string, today: string): number | null {
  const b = dayParts(birthDate);
  const t = dayParts(today);
  if (!b || !t) return null;
  const beforeBirthday = t[1] < b[1] || (t[1] === b[1] && t[2] < b[2]);
  return t[0] - b[0] - (beforeBirthday ? 1 : 0);
}

export function birthDateProblem(value: string, today: string): string | null {
  if (!value) return "informe a data de nascimento";
  if (!dayParts(value)) return "data de nascimento inválida";
  if (value > today) return "a data de nascimento não pode ser no futuro";
  const age = ageOn(value, today);
  if (age === null || age > MAX_AGE) return `idade acima de ${MAX_AGE} anos — confira a data`;
  return null;
}

export function describeProfile(profile: CitizenProfile | null, today: string): string {
  if (!profile) return "sem perfil declarado";
  const age = ageOn(profile.birth_date, today);
  const ageText = age === null ? "" : ` (${age} ${age === 1 ? "ano" : "anos"})`;
  return `nascimento ${fmtDay(profile.birth_date)}${ageText} · sexo ${sexLabel(profile.sex)}` +
    ` · identidade de gênero ${genderIdentityLabel(profile.gender_identity)}`;
}
```

- [ ] **Step 4: Rode e veja passar**

Run: `npx vitest run src/lib/profile.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/profile.ts src/lib/profile.test.ts
/opt/homebrew/bin/git commit -m "feat: add citizen profile labels and birth date rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Modelo de condição — campos por contexto e árvore ↔ modelo

**Files:**
- Create: `src/lib/condition.ts`, `src/test/conditionFixtures.ts`
- Test: `src/lib/condition.test.ts`

**Interfaces:**
- Consumes: `ConditionTree`, `PanelNeighborhood` (`src/lib/api.ts`); `SEX_OPTIONS` (Task 2).
- Produces (em `src/lib/condition.ts`):
  - tipos `ConditionContext = "eligibility" | "suggestion" | "restriction"`, `FieldKind = "number" | "choice" | "boolean"`, `FieldOption { value; label; inactive? }`, `ConditionField { id; label; kind; group; unit?; unitOne?; min?; max?; options?; verb? }`;
  - tipos `ConditionRow` (união por `op`: `"gte" | "lte" | "eq"` com `number | null`; `"between"` com `[number | null, number | null]`; `"in"` com `string[]`; `"is"` com `string`), `RowOp`, `ConditionGroup { kind: "group"; key; mode: "all" | "any"; negated; children }`, `ConditionNode`, `ParsedCondition = { ok: true; root } | { ok: false }`;
  - constantes `RESERVED_PREFIXES`, `BOOLEAN_OPTIONS`, `OP_LABELS: Record<RowOp, string>`;
  - `fieldsFor(context, sources?: { definition?: unknown; neighborhoods?: PanelNeighborhood[] }): ConditionField[]`, `tiersOf(definition): string[]`;
  - `nextKey()`, `newGroup(mode)`, `emptyRoot()`, `opsFor(kind)`, `newRow(field, op?)`, `withOp(row, op, field)`, `rowProblem(row, field)`;
  - `toTree(root): ConditionTree | null`, `fromTree(tree: unknown, fields): ParsedCondition`;
  - `updateNode(root, key, fn)`, `removeNode(root, key)`, `appendTo(root, groupKey, child)`, `treeKey(tree: unknown): string`.
- Produces (em `src/test/conditionFixtures.ts`): `SUGGESTION_DEF` (protocolo com passos boolean, enum, integer e text e `scoring.weighted`), `NEIGHBORHOODS: PanelNeighborhood[]` (Xaxim e Boqueirão ativos, Centro inativo).

- [ ] **Step 1: Escreva as fixtures**

```ts
// src/test/conditionFixtures.ts
// Dados comuns aos testes do construtor de condições (módulo 15).
import type { PanelNeighborhood } from "../lib/api";

export const SUGGESTION_DEF = {
  name: "saude-mental", version: 1, start_step_id: "humor",
  steps: [
    { id: "humor", prompt: "Sentiu-se triste?", answer_type: "boolean", branches: { true: "freq", false: null } },
    { id: "freq", prompt: "Com que frequência?", answer_type: "enum", options: [ "nunca", "às vezes", "sempre" ] },
    { id: "dias", prompt: "Há quantos dias?", answer_type: "integer" },
    { id: "obs", prompt: "Observações", answer_type: "text" }
  ],
  scoring: { type: "weighted", thresholds: { baixa: 0, media: 8, alta: 15 } }
};

export const NEIGHBORHOODS: PanelNeighborhood[] = [
  { id: "n1", name: "Xaxim", active: true },
  { id: "n2", name: "Boqueirão", active: true },
  { id: "n3", name: "Centro", active: false }
];
```

- [ ] **Step 2: Escreva o teste**

```ts
// src/lib/condition.test.ts
import { describe, expect, it } from "vitest";
import {
  appendTo, emptyRoot, fieldsFor, fromTree, newGroup, newRow, removeNode, rowProblem, tiersOf, toTree, updateNode, withOp,
  type ConditionContext, type ConditionField, type ConditionGroup, type ConditionRow
} from "./condition";
import { NEIGHBORHOODS, SUGGESTION_DEF } from "../test/conditionFixtures";

const FIELDS: Record<ConditionContext, ConditionField[]> = {
  eligibility: fieldsFor("eligibility"),
  suggestion: fieldsFor("suggestion", { definition: SUGGESTION_DEF }),
  restriction: fieldsFor("restriction", { neighborhoods: NEIGHBORHOODS })
};
const field = (context: ConditionContext, id: string) => FIELDS[context].find((f) => f.id === id)!;
const AGE = "profile.age";
const range = (f: string, a: number, b: number) => ({ all: [ { gte: [ f, a ] }, { lte: [ f, b ] } ] });
const OLD_OR_BABY = { any: [ { lte: [ AGE, 2 ] }, { gte: [ AGE, 60 ] } ] };

describe("campos por contexto", () => {
  it("elegibilidade só tem idade e sexo", () => {
    expect(FIELDS.eligibility.map((f) => f.id)).toEqual([ "profile.age", "profile.sex" ]);
    expect(field("eligibility", "profile.sex").options).toEqual([
      { value: "female", label: "feminino" }, { value: "male", label: "masculino" }
    ]);
  });

  it("sugestão ganha as respostas (sem texto livre) e o resultado com as classificações do protocolo", () => {
    expect(FIELDS.suggestion.map((f) => f.id)).toEqual([
      "profile.age", "profile.sex", "humor", "freq", "dias", "outcome.tier", "outcome.score", "outcome.priority"
    ]);
    expect(field("suggestion", "humor")).toMatchObject({ kind: "boolean", label: "“Sentiu-se triste?”", verb: " é " });
    expect(field("suggestion", "freq").options?.map((o) => o.value)).toEqual([ "nunca", "às vezes", "sempre" ]);
    expect(field("suggestion", "dias").kind).toBe("number");
    expect(field("suggestion", "outcome.tier").options?.map((o) => o.value)).toEqual([ "baixa", "media", "alta" ]);
    expect(field("suggestion", "outcome.priority")).toMatchObject({ min: 1, max: 9 });
  });

  it("passo com prefixo reservado não vira campo", () => {
    const def = { steps: [ { id: "profile.idade", prompt: "x", answer_type: "integer" } ] };
    expect(fieldsFor("suggestion", { definition: def }).map((f) => f.id)).not.toContain("profile.idade");
  });

  it("classificações de tabela de decisão vêm das regras e do fallback, sem repetir", () => {
    const def = { scoring: { type: "decision_table", rules: [ { tier: "vermelha" }, { tier: "amarela" }, { tier: "vermelha" } ], fallback: { tier: "verde" } } };
    expect(tiersOf(def)).toEqual([ "vermelha", "amarela", "verde" ]);
    expect(tiersOf({})).toEqual([]);
  });

  it("restrição ganha o bairro, em ordem de nome, com o inativo marcado", () => {
    expect(field("restriction", "citizen.neighborhood_id").options).toEqual([
      { value: "n2", label: "Boqueirão" },
      { value: "n3", label: "Centro (bairro inativo)", inactive: true },
      { value: "n1", label: "Xaxim" }
    ]);
  });
});

// Árvores canônicas: as que o próprio construtor produz. Ida e volta tem de
// devolver exatamente a mesma árvore.
const CANONICAL: Array<[ ConditionContext, string, unknown ]> = [
  [ "eligibility", "idade a partir de", { gte: [ AGE, 60 ] } ],
  [ "eligibility", "idade até", { lte: [ AGE, 12 ] } ],
  [ "eligibility", "idade entre", range(AGE, 18, 59) ],
  [ "eligibility", "idade igual", range(AGE, 40, 40) ],
  [ "eligibility", "sexo, um valor", { in: [ "profile.sex", [ "female" ] ] } ],
  [ "eligibility", "sexo, dois valores", { in: [ "profile.sex", [ "female", "male" ] ] } ],
  [ "eligibility", "NÃO na linha", { not: { gte: [ AGE, 60 ] } } ],
  [ "eligibility", "NÃO no entre", { not: range(AGE, 18, 59) } ],
  [ "eligibility", "E", { all: [ { gte: [ AGE, 40 ] }, { in: [ "profile.sex", [ "female" ] ] } ] } ],
  [ "eligibility", "OU", OLD_OR_BABY ],
  [ "eligibility", "OU com uma linha", { any: [ { gte: [ AGE, 60 ] } ] } ],
  [ "eligibility", "NÃO no grupo", { not: OLD_OR_BABY } ],
  [ "eligibility", "NÃO na raiz com uma linha", { not: { all: [ { gte: [ AGE, 60 ] } ] } } ],
  [ "eligibility", "subgrupo", { all: [ { in: [ "profile.sex", [ "female" ] ] }, OLD_OR_BABY ] } ],
  [ "eligibility", "subgrupo negado", { all: [ { in: [ "profile.sex", [ "male" ] ] }, { not: OLD_OR_BABY } ] } ],
  [ "suggestion", "sim/não", { eq: [ "humor", "true" ] } ],
  [ "suggestion", "lista", { in: [ "freq", [ "às vezes", "sempre" ] ] } ],
  [ "suggestion", "número de resposta", { gte: [ "dias", 14 ] } ],
  [ "suggestion", "pontuação", { gte: [ "outcome.score", 15 ] } ],
  [ "suggestion", "classificação", { in: [ "outcome.tier", [ "alta" ] ] } ],
  [ "suggestion", "prioridade", { lte: [ "outcome.priority", 3 ] } ],
  [ "suggestion", "misto", { all: [ { eq: [ "humor", "true" ] }, { any: [ { gte: [ "outcome.score", 15 ] }, { in: [ "freq", [ "sempre" ] ] } ] } ] } ],
  [ "restriction", "bairro", { in: [ "citizen.neighborhood_id", [ "n1", "n2" ] ] } ],
  [ "restriction", "bairro inativo", { in: [ "citizen.neighborhood_id", [ "n3" ] ] } ],
  [ "restriction", "idade e bairro", { all: [ { gte: [ AGE, 60 ] }, { in: [ "citizen.neighborhood_id", [ "n2" ] ] } ] } ]
];

describe("ida e volta", () => {
  it.each(CANONICAL)("%s / %s: árvore → modelo → árvore é a mesma", (context, _name, tree) => {
    const parsed = fromTree(tree, FIELDS[context]);
    expect(parsed.ok).toBe(true);
    if (parsed.ok) expect(toTree(parsed.root)).toEqual(tree);
  });

  it("entre vira uma linha só, e igual vira a linha 'igual a'", () => {
    const between = fromTree(range(AGE, 18, 59), FIELDS.eligibility);
    const equal = fromTree(range(AGE, 40, 40), FIELDS.eligibility);
    expect(between.ok && between.root.children[0]).toMatchObject({ kind: "row", op: "between", value: [ 18, 59 ] });
    expect(equal.ok && equal.root.children[0]).toMatchObject({ kind: "row", op: "eq", value: 40 });
  });

  it("árvore ausente é a raiz vazia, e a raiz vazia volta como null", () => {
    const parsed = fromTree(undefined, FIELDS.eligibility);
    expect(parsed.ok && parsed.root.children).toEqual([]);
    expect(toTree(emptyRoot())).toBeNull();
  });
});

describe("formas aceitas que o construtor reescreve no formato dele", () => {
  it.each([
    [ "eq em campo de escolha vira in", { eq: [ "profile.sex", "female" ] }, { in: [ "profile.sex", [ "female" ] ] } ],
    [ "eq numérico vira o par gte/lte", { eq: [ AGE, 60 ] }, range(AGE, 60, 60) ],
    [ "all com uma linha na raiz vira a linha", { all: [ { gte: [ AGE, 18 ] } ] }, { gte: [ AGE, 18 ] } ]
  ])("%s", (_name, tree, canonical) => {
    const parsed = fromTree(tree, FIELDS.eligibility);
    expect(parsed.ok).toBe(true);
    if (parsed.ok) expect(toTree(parsed.root)).toEqual(canonical);
  });
});

describe("regra avançada (fora do subconjunto)", () => {
  it.each([
    [ "eligibility", "gt", { gt: [ AGE, 59 ] } ],
    [ "eligibility", "lt", { lt: [ AGE, 12 ] } ],
    [ "suggestion", "mapa legado", { humor: "true" } ],
    [ "eligibility", "NÃO duplo", { not: { not: { gte: [ AGE, 60 ] } } } ],
    [ "eligibility", "grupo dentro de subgrupo", { all: [ { any: [ { all: [ { gte: [ AGE, 1 ] }, { in: [ "profile.sex", [ "male" ] ] } ] } ] } ] } ],
    [ "eligibility", "campo de outro lugar", { gte: [ "outcome.score", 15 ] } ],
    [ "restriction", "resposta na restrição", { eq: [ "humor", "true" ] } ],
    [ "suggestion", "passo de texto livre", { eq: [ "obs", "dor" ] } ],
    [ "eligibility", "número como texto", { gte: [ AGE, "60" ] } ],
    [ "eligibility", "in vazio", { in: [ "profile.sex", [] ] } ],
    [ "eligibility", "grupo vazio", { all: [] } ],
    [ "eligibility", "operando curto", { gte: [ AGE ] } ],
    [ "eligibility", "não é objeto", "idade > 60" ]
  ] as Array<[ ConditionContext, string, unknown ]>)("%s / %s", (context, _name, tree) => {
    expect(fromTree(tree, FIELDS[context])).toEqual({ ok: false });
  });
});

describe("linhas incompletas não entram na árvore", () => {
  const age = field("eligibility", AGE);
  const sex = field("eligibility", "profile.sex");

  it("valor vazio, entre pela metade, entre invertido e nenhuma opção somem da árvore", () => {
    const root: ConditionGroup = { ...emptyRoot(), children: [
      newRow(age),
      { ...newRow(age, "between"), value: [ 60, null ] } as ConditionRow,
      { ...newRow(age, "between"), value: [ 60, 18 ] } as ConditionRow,
      newRow(sex)
    ] };
    expect(toTree(root)).toBeNull();
  });

  it("com uma linha completa e outra incompleta, a árvore é só a completa", () => {
    const root: ConditionGroup = { ...emptyRoot(), children: [ { ...newRow(age), value: 60 } as ConditionRow, newRow(sex) ] };
    expect(toTree(root)).toEqual({ gte: [ AGE, 60 ] });
  });

  it("subgrupo vazio some", () => {
    const root: ConditionGroup = { ...emptyRoot(), children: [ { ...newRow(age), value: 60 } as ConditionRow, newGroup("any") ] };
    expect(toTree(root)).toEqual({ gte: [ AGE, 60 ] });
  });

  it("o motivo de cada linha incompleta é dito", () => {
    expect(rowProblem(newRow(age), age)).toBe("informe o valor");
    expect(rowProblem({ ...newRow(age), value: 200 } as ConditionRow, age)).toBe("use um valor entre 0 e 130");
    expect(rowProblem({ ...newRow(age, "between"), value: [ 60, 18 ] } as ConditionRow, age)).toBe("o início precisa ser menor que o fim");
    expect(rowProblem(newRow(sex), sex)).toBe("marque ao menos uma opção");
    expect(rowProblem({ ...newRow(age), value: 60 } as ConditionRow, age)).toBeNull();
    expect(rowProblem(newRow(age), undefined)).toBe("campo indisponível neste lugar");
  });
});

describe("edição do modelo", () => {
  const age = field("eligibility", AGE);

  it("trocar de operador numérico guarda o número digitado", () => {
    const row = { ...newRow(age), value: 60 } as ConditionRow;
    expect(withOp(row, "between", age)).toMatchObject({ key: row.key, op: "between", value: [ 60, null ] });
    expect(withOp(row, "lte", age)).toMatchObject({ key: row.key, op: "lte", value: 60 });
  });

  it("acrescenta, altera e remove por chave, inclusive dentro de subgrupo", () => {
    const group = newGroup("any");
    let root = appendTo(emptyRoot(), "nada", newRow(age));
    expect(root.children).toHaveLength(0);
    root = appendTo({ ...emptyRoot(), children: [ group ] }, group.key, newRow(age));
    const inner = (root.children[0] as ConditionGroup).children[0];
    root = updateNode(root, inner.key, (n) => ({ ...(n as ConditionRow), value: 60 }) as ConditionRow);
    expect(toTree(root)).toEqual({ any: [ { gte: [ AGE, 60 ] } ] });
    root = removeNode(root, inner.key);
    expect(toTree(root)).toBeNull();
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/lib/condition.test.ts`
Expected: FAIL — `./condition` não existe.

- [ ] **Step 4: Implemente**

```ts
// src/lib/condition.ts
// Construtor de condições (módulo 15; ADR 0027 sobre a linguagem do ADR 0009;
// spec §7). Converte a árvore JSON num modelo de tela — grupos E/OU, linhas
// campo · operador · valor, NÃO por linha ou grupo — e de volta. Tudo puro: o
// componente só desenha o modelo e a API é quem avalia.
//
// Subconjunto do construtor (o resto é "regra avançada", mostrada em frase):
// - linha numérica: {gte:[v,n]}, {lte:[v,n]}; "entre" = {all:[{gte:[v,a]},{lte:[v,b]}]}
//   com a < b; "igual a" = o mesmo par com a = b, para não depender de `eq`
//   comparar número com texto no servidor;
// - linha de escolha: {in:[v,[...]]} (aceita {eq:[v,"x"]} e o reescreve como in);
// - linha de sim/não (resposta boolean): {eq:[passo,"true"|"false"]};
// - {not: linha} e {not: grupo}, uma vez só;
// - grupo {all|any:[...]}: linhas e, só na raiz, subgrupos de linhas.
// Uma raiz E com uma linha só sai como a linha sozinha ({gte:["profile.age",60]}).
import type { ConditionTree, PanelNeighborhood } from "./api";
import { SEX_OPTIONS } from "./profile";

export type ConditionContext = "eligibility" | "suggestion" | "restriction";
export type FieldKind = "number" | "choice" | "boolean";
export interface FieldOption { value: string; label: string; inactive?: boolean }
export interface ConditionField {
  id: string;
  label: string;
  kind: FieldKind;
  group: string;
  unit?: string;
  unitOne?: string;
  min?: number;
  max?: number;
  options?: FieldOption[];
  // Entre o rótulo e o valor na frase: " " para "sexo feminino", " é " para respostas.
  verb?: string;
}

export type ConditionRow =
  | { kind: "row"; key: string; field: string; op: "gte" | "lte" | "eq"; value: number | null; negated: boolean }
  | { kind: "row"; key: string; field: string; op: "between"; value: [ number | null, number | null ]; negated: boolean }
  | { kind: "row"; key: string; field: string; op: "in"; value: string[]; negated: boolean }
  | { kind: "row"; key: string; field: string; op: "is"; value: string; negated: boolean };
export type RowOp = ConditionRow["op"];
export interface ConditionGroup {
  kind: "group";
  key: string;
  mode: "all" | "any";
  negated: boolean;
  children: ConditionNode[];
}
export type ConditionNode = ConditionRow | ConditionGroup;
export type ParsedCondition = { ok: true; root: ConditionGroup } | { ok: false };

export const RESERVED_PREFIXES = [ "profile.", "outcome.", "citizen." ];
export const BOOLEAN_OPTIONS: FieldOption[] = [ { value: "true", label: "sim" }, { value: "false", label: "não" } ];
export const OP_LABELS: Record<RowOp, string> = {
  gte: "a partir de", lte: "até", between: "entre", eq: "igual a", in: "é um de", is: "é"
};

// ─── Campos por contexto ────────────────────────────────────────────────────

const AGE: ConditionField = {
  id: "profile.age", label: "idade", kind: "number", group: "Perfil", unit: " anos", unitOne: " ano", min: 0, max: 130
};
const SEX: ConditionField = {
  id: "profile.sex", label: "sexo", kind: "choice", group: "Perfil",
  options: SEX_OPTIONS.map((o) => ({ value: o.value, label: o.label.toLowerCase() }))
};

export interface FieldSources { definition?: unknown; neighborhoods?: PanelNeighborhood[] }

export function fieldsFor(context: ConditionContext, sources: FieldSources = {}): ConditionField[] {
  if (context === "eligibility") return [ AGE, SEX ];
  if (context === "restriction") return [ AGE, SEX, neighborhoodField(sources.neighborhoods ?? []) ];
  return [ AGE, SEX, ...stepFields(sources.definition), ...outcomeFields(sources.definition) ];
}

function neighborhoodField(list: PanelNeighborhood[]): ConditionField {
  const sorted = [ ...list ].sort((a, b) => a.name.localeCompare(b.name, "pt-BR"));
  return {
    id: "citizen.neighborhood_id", label: "bairro", kind: "choice", group: "Cidade",
    options: sorted.map((n) => (n.active
      ? { value: n.id, label: n.name }
      : { value: n.id, label: `${n.name} (bairro inativo)`, inactive: true }))
  };
}

type RawStep = { id?: unknown; prompt?: unknown; answer_type?: unknown; options?: unknown };

function rawSteps(definition: unknown): RawStep[] {
  const steps = definition && typeof definition === "object" ? (definition as { steps?: unknown }).steps : undefined;
  return Array.isArray(steps) ? steps.filter((s): s is RawStep => !!s && typeof s === "object") : [];
}

// Texto livre não entra: comparar texto digitado pelo cidadão não é regra clínica.
function stepFields(definition: unknown): ConditionField[] {
  return rawSteps(definition).flatMap((s): ConditionField[] => {
    const id = typeof s.id === "string" ? s.id : "";
    if (!id || RESERVED_PREFIXES.some((p) => id.startsWith(p))) return [];
    const label = `“${typeof s.prompt === "string" && s.prompt ? s.prompt : id}”`;
    if (s.answer_type === "boolean") {
      return [ { id, label, group: "Respostas", verb: " é ", kind: "boolean", options: BOOLEAN_OPTIONS } ];
    }
    if (s.answer_type === "enum") {
      const options = Array.isArray(s.options) ? s.options.filter((o): o is string => typeof o === "string") : [];
      return [ { id, label, group: "Respostas", verb: " é ", kind: "choice", options: options.map((o) => ({ value: o, label: o })) } ];
    }
    if (s.answer_type === "integer") return [ { id, label, group: "Respostas", kind: "number" } ];
    return [];
  });
}

export function tiersOf(definition: unknown): string[] {
  const scoring = definition && typeof definition === "object" ? (definition as { scoring?: unknown }).scoring : undefined;
  if (!scoring || typeof scoring !== "object") return [];
  const s = scoring as { thresholds?: unknown; rules?: unknown; fallback?: unknown };
  const out: string[] = [];
  if (s.thresholds && typeof s.thresholds === "object") out.push(...Object.keys(s.thresholds));
  if (Array.isArray(s.rules)) {
    for (const rule of s.rules) {
      const tier = rule && typeof rule === "object" ? (rule as { tier?: unknown }).tier : undefined;
      if (typeof tier === "string") out.push(tier);
    }
  }
  const fallback = s.fallback && typeof s.fallback === "object" ? (s.fallback as { tier?: unknown }).tier : undefined;
  if (typeof fallback === "string") out.push(fallback);
  return [ ...new Set(out) ];
}

function outcomeFields(definition: unknown): ConditionField[] {
  return [
    { id: "outcome.tier", label: "classificação", kind: "choice", group: "Resultado",
      options: tiersOf(definition).map((t) => ({ value: t, label: t })) },
    { id: "outcome.score", label: "pontuação", kind: "number", group: "Resultado" },
    { id: "outcome.priority", label: "prioridade", kind: "number", group: "Resultado", min: 1, max: 9 }
  ];
}

// ─── Modelo ─────────────────────────────────────────────────────────────────

let seq = 0;
export function nextKey(): string {
  seq += 1;
  return `k${seq}`;
}

export function newGroup(mode: "all" | "any"): ConditionGroup {
  return { kind: "group", key: nextKey(), mode, negated: false, children: [] };
}

export function emptyRoot(): ConditionGroup {
  return newGroup("all");
}

export function opsFor(kind: FieldKind): RowOp[] {
  if (kind === "number") return [ "gte", "lte", "between", "eq" ];
  if (kind === "choice") return [ "in" ];
  return [ "is" ];
}

export function newRow(field: ConditionField, op: RowOp = opsFor(field.kind)[0]): ConditionRow {
  const base = { kind: "row" as const, key: nextKey(), field: field.id, negated: false };
  switch (op) {
    case "between": return { ...base, op, value: [ null, null ] };
    case "in": return { ...base, op, value: [] };
    case "is": return { ...base, op, value: "true" };
    default: return { ...base, op, value: null };
  }
}

function firstNumber(row: ConditionRow): number | null {
  if (row.op === "between") return row.value[0];
  if (row.op === "gte" || row.op === "lte" || row.op === "eq") return row.value;
  return null;
}

export function withOp(row: ConditionRow, op: RowOp, field: ConditionField): ConditionRow {
  const next = { ...newRow(field, op), key: row.key, negated: row.negated };
  const n = firstNumber(row);
  if (next.op === "between") return { ...next, value: [ n, null ] };
  if (next.op === "gte" || next.op === "lte" || next.op === "eq") return { ...next, value: n };
  return next;
}

export function rowProblem(row: ConditionRow, field: ConditionField | undefined): string | null {
  if (!field) return "campo indisponível neste lugar";
  const numbers = row.op === "between" ? row.value
    : (row.op === "gte" || row.op === "lte" || row.op === "eq") ? [ row.value ] : [];
  if (numbers.some((n) => n === null)) return "informe o valor";
  const outOfRange = numbers.some((n) => n !== null &&
    ((field.min !== undefined && n < field.min) || (field.max !== undefined && n > field.max)));
  if (outOfRange) return `use um valor entre ${field.min ?? "…"} e ${field.max ?? "…"}`;
  if (row.op === "between" && (row.value[0] as number) >= (row.value[1] as number)) return "o início precisa ser menor que o fim";
  if (row.op === "in" && row.value.length === 0) return "marque ao menos uma opção";
  return null;
}

// ─── Modelo → árvore ────────────────────────────────────────────────────────

function range(field: string, a: number, b: number): ConditionTree {
  return { all: [ { gte: [ field, a ] }, { lte: [ field, b ] } ] };
}

// Linha incompleta não emite nada: o JSON nunca recebe uma árvore inválida, e
// a linha continua no modelo da tela até ser completada ou removida.
function rowTree(row: ConditionRow): ConditionTree | null {
  let tree: ConditionTree | null = null;
  switch (row.op) {
    case "gte":
    case "lte":
      tree = row.value === null ? null : { [row.op]: [ row.field, row.value ] };
      break;
    case "eq":
      tree = row.value === null ? null : range(row.field, row.value, row.value);
      break;
    case "between": {
      const [ a, b ] = row.value;
      tree = a === null || b === null || a >= b ? null : range(row.field, a, b);
      break;
    }
    case "in":
      tree = row.value.length === 0 ? null : { in: [ row.field, [ ...row.value ] ] };
      break;
    case "is":
      tree = { eq: [ row.field, row.value ] };
      break;
  }
  return tree && row.negated ? { not: tree } : tree;
}

function groupTree(group: ConditionGroup, isRoot: boolean): ConditionTree | null {
  const parts = group.children
    .map((c) => (c.kind === "row" ? rowTree(c) : groupTree(c, false)))
    .filter((t): t is ConditionTree => t !== null);
  if (parts.length === 0) return null;
  const bare = isRoot && group.mode === "all" && parts.length === 1 && !group.negated;
  const tree = bare ? parts[0] : { [group.mode]: parts };
  return group.negated ? { not: tree } : tree;
}

export function toTree(root: ConditionGroup): ConditionTree | null {
  return groupTree(root, true);
}

// ─── Árvore → modelo ────────────────────────────────────────────────────────

type Fields = Map<string, ConditionField>;

function single(node: unknown): { op: string; arg: unknown } | null {
  if (!node || typeof node !== "object" || Array.isArray(node)) return null;
  const keys = Object.keys(node);
  return keys.length === 1 ? { op: keys[0], arg: (node as Record<string, unknown>)[keys[0]] } : null;
}

const isNumber = (v: unknown): v is number => typeof v === "number" && Number.isFinite(v);

function operand(arg: unknown): [ string, unknown ] | null {
  return Array.isArray(arg) && arg.length === 2 && typeof arg[0] === "string" ? [ arg[0], arg[1] ] : null;
}

function parseRange(arg: unknown, fields: Fields): ConditionRow | null {
  if (!Array.isArray(arg) || arg.length !== 2) return null;
  const lo = single(arg[0]);
  const hi = single(arg[1]);
  if (lo?.op !== "gte" || hi?.op !== "lte") return null;
  const a = operand(lo.arg);
  const b = operand(hi.arg);
  if (!a || !b || a[0] !== b[0] || fields.get(a[0])?.kind !== "number" || !isNumber(a[1]) || !isNumber(b[1])) return null;
  const base = { kind: "row" as const, key: nextKey(), field: a[0], negated: false };
  if (a[1] === b[1]) return { ...base, op: "eq", value: a[1] };
  return a[1] < b[1] ? { ...base, op: "between", value: [ a[1], b[1] ] } : null;
}

function parseRow(node: unknown, fields: Fields): ConditionRow | null {
  const n = single(node);
  if (!n) return null;
  if (n.op === "not") {
    const inner = parseRow(n.arg, fields);
    return inner && !inner.negated ? { ...inner, negated: true } : null;
  }
  if (n.op === "all") return parseRange(n.arg, fields);
  const o = operand(n.arg);
  const field = o ? fields.get(o[0]) : undefined;
  if (!o || !field) return null;
  const base = { kind: "row" as const, key: nextKey(), field: field.id, negated: false };
  const value = o[1];
  if ((n.op === "gte" || n.op === "lte") && field.kind === "number" && isNumber(value)) {
    return { ...base, op: n.op as "gte" | "lte", value };
  }
  if (n.op === "eq" && field.kind === "number" && isNumber(value)) return { ...base, op: "eq", value };
  if (n.op === "eq" && field.kind === "boolean" && (value === "true" || value === "false")) return { ...base, op: "is", value };
  if (n.op === "eq" && field.kind === "choice" && typeof value === "string") return { ...base, op: "in", value: [ value ] };
  if (n.op === "in" && field.kind === "choice" && Array.isArray(value) && value.length > 0 &&
      value.every((v) => typeof v === "string")) {
    return { ...base, op: "in", value: [ ...(value as string[]) ] };
  }
  return null;
}

function parseGroup(node: unknown, fields: Fields, depth: number): ConditionGroup | null {
  const n = single(node);
  if (!n) return null;
  if (n.op === "not") {
    const group = parseGroup(n.arg, fields, depth);
    return group && !group.negated ? { ...group, negated: true } : null;
  }
  if ((n.op !== "all" && n.op !== "any") || !Array.isArray(n.arg) || n.arg.length === 0) return null;
  const children: ConditionNode[] = [];
  for (const child of n.arg) {
    const row = parseRow(child, fields);
    if (row) { children.push(row); continue; }
    const group = depth === 0 ? parseGroup(child, fields, 1) : null;
    if (!group) return null;
    children.push(group);
  }
  return { kind: "group", key: nextKey(), mode: n.op as "all" | "any", negated: false, children };
}

export function fromTree(tree: unknown, fields: ConditionField[]): ParsedCondition {
  if (tree === null || tree === undefined) return { ok: true, root: emptyRoot() };
  const byId: Fields = new Map(fields.map((f) => [ f.id, f ]));
  const row = parseRow(tree, byId);
  if (row) return { ok: true, root: { ...emptyRoot(), children: [ row ] } };
  const root = parseGroup(tree, byId, 0);
  return root ? { ok: true, root } : { ok: false };
}

// ─── Edição ─────────────────────────────────────────────────────────────────

export function updateNode(root: ConditionGroup, key: string, fn: (node: ConditionNode) => ConditionNode): ConditionGroup {
  if (root.key === key) return fn(root) as ConditionGroup;
  return {
    ...root,
    children: root.children.map((c) => (c.key === key ? fn(c) : c.kind === "group" ? updateNode(c, key, fn) : c))
  };
}

export function removeNode(root: ConditionGroup, key: string): ConditionGroup {
  return {
    ...root,
    children: root.children.filter((c) => c.key !== key).map((c) => (c.kind === "group" ? removeNode(c, key) : c))
  };
}

export function appendTo(root: ConditionGroup, groupKey: string, child: ConditionNode): ConditionGroup {
  return updateNode(root, groupKey, (g) => (g.kind === "group" ? { ...g, children: [ ...g.children, child ] } : g));
}

// Comparação de árvores: nós de chave única e listas têm ordem estável.
export function treeKey(tree: unknown): string {
  return tree === null || tree === undefined ? "" : JSON.stringify(tree);
}
```

- [ ] **Step 5: Rode e veja passar**

Run: `npx vitest run src/lib/condition.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/lib/condition.ts src/lib/condition.test.ts src/test/conditionFixtures.ts
/opt/homebrew/bin/git commit -m "feat: add condition model with tree round trip for the builder

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Condição em frase

**Files:**
- Create: `src/lib/conditionPhrase.ts`
- Test: `src/lib/conditionPhrase.test.ts`

**Interfaces:**
- Consumes: `ConditionField`, `fieldsFor` (Task 3); `joinPt` de `src/lib/audiencePhrase.ts`; fixtures da Task 3.
- Produces: `describeCondition(tree: unknown, fields: ConditionField[], emptyText: string): string`; `UNKNOWN_RULE = "regra não reconhecida"`. Descreve **qualquer** árvore da linguagem (inclusive `gt`, `lt` e o mapa legado), para servir também à "regra avançada" e à aba do catálogo.

- [ ] **Step 1: Escreva o teste**

```ts
// src/lib/conditionPhrase.test.ts
import { describe, expect, it } from "vitest";
import { fieldsFor } from "./condition";
import { UNKNOWN_RULE, describeCondition } from "./conditionPhrase";
import { NEIGHBORHOODS, SUGGESTION_DEF } from "../test/conditionFixtures";

const ELIG = fieldsFor("eligibility");
const SUGG = fieldsFor("suggestion", { definition: SUGGESTION_DEF });
const REST = fieldsFor("restriction", { neighborhoods: NEIGHBORHOODS });
const AGE = "profile.age";
const say = (tree: unknown, fields = ELIG) => describeCondition(tree, fields, "para todos");

describe("frase da condição", () => {
  it("sem condição usa o texto do lugar", () => {
    expect(say(null)).toBe("para todos");
    expect(describeCondition(undefined, REST, "nenhuma")).toBe("nenhuma");
  });

  it("idade com unidade, inclusive no singular", () => {
    expect(say({ gte: [ AGE, 60 ] })).toBe("idade a partir de 60 anos");
    expect(say({ lte: [ AGE, 1 ] })).toBe("idade até 1 ano");
    expect(say({ all: [ { gte: [ AGE, 18 ] }, { lte: [ AGE, 59 ] } ] })).toBe("idade entre 18 e 59 anos");
    expect(say({ all: [ { gte: [ AGE, 40 ] }, { lte: [ AGE, 40 ] } ] })).toBe("idade igual a 40 anos");
    expect(say({ eq: [ AGE, 60 ] })).toBe("idade igual a 60 anos");
  });

  it("sexo é um de, em português", () => {
    expect(say({ in: [ "profile.sex", [ "female" ] ] })).toBe("sexo feminino");
    expect(say({ in: [ "profile.sex", [ "female", "male" ] ] })).toBe("sexo feminino ou masculino");
    expect(say({ eq: [ "profile.sex", "male" ] })).toBe("sexo masculino");
  });

  it("NÃO, E, OU e subgrupo entre parênteses", () => {
    expect(say({ not: { gte: [ AGE, 60 ] } })).toBe("não (idade a partir de 60 anos)");
    expect(say({ all: [ { gte: [ AGE, 40 ] }, { in: [ "profile.sex", [ "female" ] ] } ] }))
      .toBe("idade a partir de 40 anos e sexo feminino");
    expect(say({ all: [ { in: [ "profile.sex", [ "female" ] ] }, { any: [ { lte: [ AGE, 2 ] }, { gte: [ AGE, 60 ] } ] } ] }))
      .toBe("sexo feminino e (idade até 2 anos ou idade a partir de 60 anos)");
    expect(say({ all: [ { in: [ "profile.sex", [ "female" ] ] }, { all: [ { gte: [ AGE, 18 ] }, { lte: [ AGE, 59 ] } ] } ] }))
      .toBe("sexo feminino e idade entre 18 e 59 anos");
  });

  it("respostas e resultado na sugestão", () => {
    expect(say({ eq: [ "humor", "true" ] }, SUGG)).toBe("“Sentiu-se triste?” é sim");
    expect(say({ in: [ "freq", [ "às vezes", "sempre" ] ] }, SUGG)).toBe("“Com que frequência?” é às vezes ou sempre");
    expect(say({ gte: [ "dias", 14 ] }, SUGG)).toBe("“Há quantos dias?” a partir de 14");
    expect(say({ gte: [ "outcome.score", 15 ] }, SUGG)).toBe("pontuação a partir de 15");
    expect(say({ in: [ "outcome.tier", [ "alta" ] ] }, SUGG)).toBe("classificação alta");
  });

  it("bairro inativo e bairro desconhecido", () => {
    expect(say({ in: [ "citizen.neighborhood_id", [ "n1", "n3" ] ] }, REST)).toBe("bairro Xaxim ou Centro (bairro inativo)");
    expect(say({ in: [ "citizen.neighborhood_id", [ "zz" ] ] }, REST)).toBe("bairro zz (fora da lista)");
  });

  it("regra avançada também vira frase", () => {
    expect(say({ gt: [ AGE, 59 ] })).toBe("idade acima de 59 anos");
    expect(say({ lt: [ AGE, 12 ] })).toBe("idade abaixo de 12 anos");
    expect(say({ humor: "true", freq: "sempre" }, SUGG)).toBe("“Sentiu-se triste?” é sim e “Com que frequência?” é sempre");
    expect(say({ not: { not: { gte: [ AGE, 60 ] } } })).toBe("não (não (idade a partir de 60 anos))");
    expect(say({ gte: [ "profile.peso", 80 ] })).toBe("profile.peso a partir de 80");
  });

  it("o que não é condição diz que não reconhece", () => {
    expect(say("idade > 60")).toBe(UNKNOWN_RULE);
    expect(say({})).toBe(UNKNOWN_RULE);
    expect(say({ all: [] })).toBe(UNKNOWN_RULE);
    expect(say({ gte: [ AGE ] })).toBe(UNKNOWN_RULE);
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/conditionPhrase.test.ts`
Expected: FAIL — `./conditionPhrase` não existe.

- [ ] **Step 3: Implemente**

```ts
// src/lib/conditionPhrase.ts
// Condição em frase em português (módulo 15; spec §7: "frase sempre
// visível"). Descreve qualquer árvore da linguagem do ADR 0009, não só o
// subconjunto do construtor: a mesma função serve à regra avançada e às
// colunas da aba do catálogo.
import { joinPt } from "./audiencePhrase";
import type { ConditionField } from "./condition";

export const UNKNOWN_RULE = "regra não reconhecida";

const OPERATORS = new Set([ "eq", "in", "gt", "lt", "gte", "lte", "all", "any", "not" ]);
const COMPARE: Record<string, string> = { gte: "a partir de", lte: "até", gt: "acima de", lt: "abaixo de" };
type Fields = Map<string, ConditionField>;

export function describeCondition(tree: unknown, fields: ConditionField[], emptyText: string): string {
  if (tree === null || tree === undefined) return emptyText;
  return phrase(tree, new Map(fields.map((f) => [ f.id, f ])));
}

function entries(node: unknown): Array<[ string, unknown ]> | null {
  if (!node || typeof node !== "object" || Array.isArray(node)) return null;
  return Object.entries(node as Record<string, unknown>);
}

function operand(arg: unknown): [ string, unknown ] | null {
  return Array.isArray(arg) && arg.length === 2 && typeof arg[0] === "string" ? [ arg[0], arg[1] ] : null;
}

function label(id: string, fields: Fields): string {
  return fields.get(id)?.label ?? id;
}

function withUnit(v: unknown, field?: ConditionField): string {
  if (typeof v !== "number") return String(v);
  const unit = v === 1 && field?.unitOne ? field.unitOne : field?.unit ?? "";
  return `${v}${unit}`;
}

function valueLabel(v: unknown, field?: ConditionField): string {
  const s = String(v);
  if (!field?.options) return s;
  return field.options.find((o) => o.value === s)?.label ?? `${s} (fora da lista)`;
}

function phrase(node: unknown, fields: Fields): string {
  const e = entries(node);
  if (!e || e.length === 0) return UNKNOWN_RULE;
  if (e.length !== 1 || !OPERATORS.has(e[0][0])) return legacy(e, fields);
  const [ op, arg ] = e[0];
  if (op in COMPARE) return compare(op, arg, fields);
  if (op === "eq") return equals(arg, fields);
  if (op === "in") return member(arg, fields);
  if (op === "not") return `não (${phrase(arg, fields)})`;
  if (op === "all") return rangePhrase(arg, fields) ?? list(arg, "e", fields);
  return list(arg, "ou", fields);
}

function compare(op: string, arg: unknown, fields: Fields): string {
  const o = operand(arg);
  if (!o) return UNKNOWN_RULE;
  return `${label(o[0], fields)} ${COMPARE[op]} ${withUnit(o[1], fields.get(o[0]))}`;
}

function equals(arg: unknown, fields: Fields): string {
  const o = operand(arg);
  if (!o) return UNKNOWN_RULE;
  const field = fields.get(o[0]);
  if (field?.kind === "number") return `${field.label} igual a ${withUnit(o[1], field)}`;
  return `${label(o[0], fields)}${field?.verb ?? " "}${valueLabel(o[1], field)}`;
}

function member(arg: unknown, fields: Fields): string {
  const o = operand(arg);
  if (!o || !Array.isArray(o[1]) || o[1].length === 0) return UNKNOWN_RULE;
  const field = fields.get(o[0]);
  return `${label(o[0], fields)}${field?.verb ?? " "}${joinPt(o[1].map((v) => valueLabel(v, field)), "ou")}`;
}

function rangePhrase(arg: unknown, fields: Fields): string | null {
  if (!Array.isArray(arg) || arg.length !== 2) return null;
  const lo = entries(arg[0]);
  const hi = entries(arg[1]);
  if (lo?.length !== 1 || hi?.length !== 1 || lo[0][0] !== "gte" || hi[0][0] !== "lte") return null;
  const a = operand(lo[0][1]);
  const b = operand(hi[0][1]);
  if (!a || !b || a[0] !== b[0] || typeof a[1] !== "number" || typeof b[1] !== "number") return null;
  const field = fields.get(a[0]);
  if (a[1] === b[1]) return `${label(a[0], fields)} igual a ${withUnit(a[1], field)}`;
  return `${label(a[0], fields)} entre ${a[1]} e ${withUnit(b[1], field)}`;
}

// Filho E/OU com mais de um item vai entre parênteses; o "entre" não.
function composite(node: unknown): boolean {
  const e = entries(node);
  if (!e || e.length !== 1) return false;
  const [ op, arg ] = e[0];
  if ((op !== "all" && op !== "any") || !Array.isArray(arg) || arg.length < 2) return false;
  return !(op === "all" && rangePhrase(arg, new Map()) !== null);
}

function list(arg: unknown, word: "e" | "ou", fields: Fields): string {
  if (!Array.isArray(arg) || arg.length === 0) return UNKNOWN_RULE;
  return arg.map((c) => (composite(c) ? `(${phrase(c, fields)})` : phrase(c, fields))).join(` ${word} `);
}

// Mapa legado {passo: valor} = E de igualdades (ADR 0009).
function legacy(e: Array<[ string, unknown ]>, fields: Fields): string {
  return e.map(([ id, v ]) => `${label(id, fields)}${fields.get(id)?.verb ?? " "}${valueLabel(v, fields.get(id))}`).join(" e ");
}
```

- [ ] **Step 4: Rode e veja passar**

Run: `npx vitest run src/lib/conditionPhrase.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/conditionPhrase.ts src/lib/conditionPhrase.test.ts
/opt/homebrew/bin/git commit -m "feat: describe any condition tree as a Portuguese sentence

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Construtor de condições (componente)

**Files:**
- Create: `src/modules/protocols/ConditionBuilder.tsx`
- Test: `src/modules/protocols/ConditionBuilder.test.tsx`

**Interfaces:**
- Consumes: tudo de `src/lib/condition.ts` (Task 3), `describeCondition` (Task 4), `inputStyle`/`secondaryButtonStyle` de `src/components/formStyles.ts`.
- Produces: `ConditionBuilder(props: ConditionBuilderProps)` com `ConditionBuilderProps { label: string; fields: ConditionField[]; value: unknown; emptyText: string; onChange(next: ConditionTree | null): void }`.
  - `value` é a árvore que está no lugar (JSON do editor ou formulário do catálogo); `onChange(null)` = tirar a condição;
  - acessibilidade que os testes e as telas seguintes usam: `fieldset` com `aria-label={label}`; grupo raiz `role="group"` "regras"; linha `role="group"` "condição N"; subgrupo `role="group"` "grupo N"; controles `aria-label` "campo", "operador", "valor", "de", "até", "combinar"; caixas "NÃO" (linha), "NÃO (negar tudo)" (raiz), "NÃO (negar o grupo)"; botões "+ condição", "+ grupo", "remover", "remover grupo";
  - texto "Em frase: …" sempre visível; com árvore fora do subconjunto, o aviso "Regra avançada: o construtor não edita esta regra. Altere no JSON." e nenhum controle.
- Regra de sincronia: o modelo local só é relido de `value` quando `value` muda para algo diferente do que o próprio construtor emitiu (`treeKey`), ou quando a lista de campos muda e a regra estava como avançada. Assim uma linha incompleta não some porque o JSON voltou sem ela.

- [ ] **Step 1: Escreva o teste**

```tsx
// src/modules/protocols/ConditionBuilder.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
import { useState } from "react";
import { ConditionBuilder } from "./ConditionBuilder";
import { fieldsFor, type ConditionField } from "../../lib/condition";
import { NEIGHBORHOODS } from "../../test/conditionFixtures";

afterEach(cleanup);

const ELIG = fieldsFor("eligibility");
const REST = fieldsFor("restriction", { neighborhoods: NEIGHBORHOODS });

function Harness({ initial, fields = ELIG, onTree }: { initial: unknown; fields?: ConditionField[]; onTree: (t: unknown) => void }) {
  const [ value, setValue ] = useState<unknown>(initial);
  return (
    <>
      <ConditionBuilder label="Elegibilidade" fields={fields} value={value} emptyText="para todos"
        onChange={(tree) => { setValue(tree); onTree(tree); }} />
      <button type="button" onClick={() => setValue({ gte: [ "profile.age", 18 ] })}>trocar por fora</button>
    </>
  );
}

const lastTree = (fn: ReturnType<typeof vi.fn>) => fn.mock.calls[fn.mock.calls.length - 1][0];
const row = (n: number) => screen.getByRole("group", { name: `condição ${n}` });

describe("ConditionBuilder", () => {
  it("vazio diz 'para todos'; idade a partir de 60 vira a árvore e a frase", () => {
    const onTree = vi.fn();
    render(<Harness initial={undefined} onTree={onTree} />);
    expect(screen.getByText("para todos")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "+ condição" }));
    fireEvent.change(within(row(1)).getByLabelText("valor"), { target: { value: "60" } });
    expect(lastTree(onTree)).toEqual({ gte: [ "profile.age", 60 ] });
    expect(screen.getByText("idade a partir de 60 anos")).not.toBeNull();
  });

  it("linha incompleta continua na tela e não emite nada", () => {
    const onTree = vi.fn();
    render(<Harness initial={undefined} onTree={onTree} />);
    fireEvent.click(screen.getByRole("button", { name: "+ condição" }));
    expect(lastTree(onTree)).toBeNull();
    expect(row(1)).not.toBeNull();
    expect(within(row(1)).getByText("informe o valor")).not.toBeNull();
    fireEvent.change(within(row(1)).getByLabelText("operador"), { target: { value: "between" } });
    fireEvent.change(within(row(1)).getByLabelText("de"), { target: { value: "60" } });
    fireEvent.change(within(row(1)).getByLabelText("até"), { target: { value: "18" } });
    expect(lastTree(onTree)).toBeNull();
    expect(within(row(1)).getByText("o início precisa ser menor que o fim")).not.toBeNull();
  });

  it("entre emite o all de gte e lte", () => {
    const onTree = vi.fn();
    render(<Harness initial={undefined} onTree={onTree} />);
    fireEvent.click(screen.getByRole("button", { name: "+ condição" }));
    fireEvent.change(within(row(1)).getByLabelText("operador"), { target: { value: "between" } });
    fireEvent.change(within(row(1)).getByLabelText("de"), { target: { value: "18" } });
    fireEvent.change(within(row(1)).getByLabelText("até"), { target: { value: "59" } });
    expect(lastTree(onTree)).toEqual({ all: [ { gte: [ "profile.age", 18 ] }, { lte: [ "profile.age", 59 ] } ] });
  });

  it("sexo é um de: marcar as duas opções", () => {
    const onTree = vi.fn();
    render(<Harness initial={undefined} onTree={onTree} />);
    fireEvent.click(screen.getByRole("button", { name: "+ condição" }));
    fireEvent.change(within(row(1)).getByLabelText("campo"), { target: { value: "profile.sex" } });
    fireEvent.click(within(row(1)).getByLabelText("feminino"));
    fireEvent.click(within(row(1)).getByLabelText("masculino"));
    expect(lastTree(onTree)).toEqual({ in: [ "profile.sex", [ "female", "male" ] ] });
    expect(screen.getByText("sexo feminino ou masculino")).not.toBeNull();
  });

  it("NÃO na linha e na raiz", () => {
    const onTree = vi.fn();
    render(<Harness initial={{ gte: [ "profile.age", 60 ] }} onTree={onTree} />);
    fireEvent.click(within(row(1)).getByLabelText("NÃO"));
    expect(lastTree(onTree)).toEqual({ not: { gte: [ "profile.age", 60 ] } });
    fireEvent.click(screen.getByLabelText("NÃO (negar tudo)"));
    expect(lastTree(onTree)).toEqual({ not: { all: [ { not: { gte: [ "profile.age", 60 ] } } ] } });
  });

  it("grupo OU dentro do E", () => {
    const onTree = vi.fn();
    render(<Harness initial={{ in: [ "profile.sex", [ "female" ] ] }} onTree={onTree} />);
    fireEvent.click(screen.getByRole("button", { name: "+ grupo" }));
    const group = screen.getByRole("group", { name: "grupo 2" });
    fireEvent.change(within(group).getByLabelText("valor"), { target: { value: "60" } });
    fireEvent.click(within(group).getByRole("button", { name: "+ condição" }));
    const second = within(group).getByRole("group", { name: "condição 2" });
    fireEvent.change(within(second).getByLabelText("operador"), { target: { value: "lte" } });
    fireEvent.change(within(second).getByLabelText("valor"), { target: { value: "2" } });
    expect(lastTree(onTree)).toEqual({ all: [
      { in: [ "profile.sex", [ "female" ] ] },
      { any: [ { gte: [ "profile.age", 60 ] }, { lte: [ "profile.age", 2 ] } ] }
    ] });
    expect(screen.getByText("sexo feminino e (idade a partir de 60 anos ou idade até 2 anos)")).not.toBeNull();
  });

  it("mudança vinda de fora reaparece nos controles", () => {
    render(<Harness initial={{ gte: [ "profile.age", 60 ] }} onTree={vi.fn()} />);
    fireEvent.click(screen.getByRole("button", { name: "trocar por fora" }));
    expect((within(row(1)).getByLabelText("valor") as HTMLInputElement).value).toBe("18");
  });

  it("remover a única linha emite null e volta a 'para todos'", () => {
    const onTree = vi.fn();
    render(<Harness initial={{ gte: [ "profile.age", 60 ] }} onTree={onTree} />);
    fireEvent.click(within(row(1)).getByRole("button", { name: "remover" }));
    expect(lastTree(onTree)).toBeNull();
    expect(screen.getByText("para todos")).not.toBeNull();
  });

  it("regra avançada: só a frase, sem controles, e nada é emitido", () => {
    const onTree = vi.fn();
    render(<Harness initial={{ gt: [ "profile.age", 59 ] }} onTree={onTree} />);
    expect(screen.getByText("Regra avançada: o construtor não edita esta regra. Altere no JSON.")).not.toBeNull();
    expect(screen.getByText("idade acima de 59 anos")).not.toBeNull();
    expect(screen.queryByRole("button", { name: "+ condição" })).toBeNull();
    expect(onTree).not.toHaveBeenCalled();
  });

  it("bairro inativo marcado aparece e sai ao desmarcar", () => {
    const onTree = vi.fn();
    render(<Harness fields={REST} initial={{ in: [ "citizen.neighborhood_id", [ "n3", "n1" ] ] }} onTree={onTree} />);
    const inactive = within(row(1)).getByLabelText("Centro (bairro inativo)") as HTMLInputElement;
    expect(inactive.checked).toBe(true);
    expect(screen.getByText("bairro Centro (bairro inativo) ou Xaxim")).not.toBeNull();
    fireEvent.click(inactive);
    expect(lastTree(onTree)).toEqual({ in: [ "citizen.neighborhood_id", [ "n1" ] ] });
    expect(within(row(1)).queryByLabelText("Centro (bairro inativo)")).toBeNull();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/protocols/ConditionBuilder.test.tsx`
Expected: FAIL — `./ConditionBuilder` não existe.

- [ ] **Step 3: Implemente**

```tsx
// src/modules/protocols/ConditionBuilder.tsx
// Construtor visual de condições (módulo 15; spec §7). Um componente só para
// a elegibilidade, as sugestões e a restrição da cidade: o que muda entre
// eles é a lista de campos (fieldsFor). A frase em português fica sempre à
// vista; árvore fora do subconjunto aparece como "regra avançada", só em frase.
import { useEffect, useRef, useState, type CSSProperties } from "react";
import type { ConditionTree } from "../../lib/api";
import {
  OP_LABELS, appendTo, fromTree, newGroup, newRow, opsFor, removeNode, rowProblem, toTree, treeKey, updateNode, withOp,
  type ConditionField, type ConditionGroup, type ConditionNode, type ConditionRow, type ParsedCondition, type RowOp
} from "../../lib/condition";
import { describeCondition } from "../../lib/conditionPhrase";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";

export interface ConditionBuilderProps {
  label: string;
  fields: ConditionField[];
  value: unknown;
  emptyText: string;
  onChange(next: ConditionTree | null): void;
}

type Fields = Map<string, ConditionField>;

export function ConditionBuilder({ label, fields, value, emptyText, onChange }: ConditionBuilderProps) {
  const [ parsed, setParsed ] = useState<ParsedCondition>(() => fromTree(value, fields));
  const emitted = useRef(treeKey(value));
  const fieldsKey = fields.map((f) => f.id).join("|");
  const seenFields = useRef(fieldsKey);

  // Relê só o que não veio daqui: o eco do próprio onChange não pode apagar
  // uma linha incompleta que ainda não tem árvore.
  useEffect(() => {
    const key = treeKey(value);
    const fieldsChanged = seenFields.current !== fieldsKey;
    seenFields.current = fieldsKey;
    if (key === emitted.current && !(fieldsChanged && !parsed.ok)) return;
    emitted.current = key;
    setParsed(fromTree(value, fields));
  }, [ value, fieldsKey ]);

  function emit(root: ConditionGroup) {
    setParsed({ ok: true, root });
    const tree = toTree(root);
    emitted.current = treeKey(tree);
    onChange(tree);
  }

  const byId: Fields = new Map(fields.map((f) => [ f.id, f ]));
  const sentence = describeCondition(parsed.ok ? toTree(parsed.root) : value, fields, emptyText);

  return (
    <fieldset aria-label={label} style={box}>
      <legend style={legend}>{label}</legend>
      <p style={phraseStyle}><span style={{ color: "var(--ink3)" }}>Em frase: </span>{sentence}</p>
      {!parsed.ok && <p style={hint}>Regra avançada: o construtor não edita esta regra. Altere no JSON.</p>}
      {parsed.ok && (
        <GroupEditor group={parsed.root} root={parsed.root} isRoot index={0} fields={fields} byId={byId} onRoot={emit} />
      )}
    </fieldset>
  );
}

interface EditorProps { root: ConditionGroup; fields: ConditionField[]; byId: Fields; onRoot(root: ConditionGroup): void }

function GroupEditor({ group, root, isRoot, index, fields, byId, onRoot }:
  EditorProps & { group: ConditionGroup; isRoot: boolean; index: number }) {
  const set = (fn: (g: ConditionGroup) => ConditionGroup) =>
    onRoot(updateNode(root, group.key, (n) => fn(n as ConditionGroup)));
  const first = fields[0];

  return (
    <div role="group" aria-label={isRoot ? "regras" : `grupo ${index}`} style={isRoot ? stack : subgroup}>
      <div style={line}>
        <select aria-label="combinar" value={group.mode} style={select}
          onChange={(e) => set((g) => ({ ...g, mode: e.target.value as "all" | "any" }))}>
          <option value="all">todas (E)</option>
          <option value="any">qualquer uma (OU)</option>
        </select>
        <label style={small}>
          <input type="checkbox" checked={group.negated} onChange={(e) => set((g) => ({ ...g, negated: e.target.checked }))} />
          {isRoot ? "NÃO (negar tudo)" : "NÃO (negar o grupo)"}
        </label>
        {!isRoot && (
          <button type="button" style={secondaryButtonStyle} onClick={() => onRoot(removeNode(root, group.key))}>remover grupo</button>
        )}
      </div>
      {group.children.map((child: ConditionNode, i) => (child.kind === "row"
        ? <RowEditor key={child.key} row={child} n={i + 1} root={root} fields={fields} byId={byId} onRoot={onRoot} />
        : <GroupEditor key={child.key} group={child} root={root} isRoot={false} index={i + 1} fields={fields} byId={byId} onRoot={onRoot} />))}
      <div style={line}>
        <button type="button" style={secondaryButtonStyle} disabled={!first}
          onClick={() => first && onRoot(appendTo(root, group.key, newRow(first)))}>+ condição</button>
        {isRoot && (
          <button type="button" style={secondaryButtonStyle} disabled={!first}
            onClick={() => first && onRoot(appendTo(root, group.key, { ...newGroup("any"), children: [ newRow(first) ] }))}>
            + grupo
          </button>
        )}
      </div>
    </div>
  );
}

function RowEditor({ row, n, root, fields, byId, onRoot }: EditorProps & { row: ConditionRow; n: number }) {
  const field = byId.get(row.field);
  const set = (next: ConditionRow) => onRoot(updateNode(root, row.key, () => next));
  const problem = rowProblem(row, field);
  const groups = [ ...new Set(fields.map((f) => f.group)) ];

  return (
    <div role="group" aria-label={`condição ${n}`} style={rowBox}>
      <select aria-label="campo" value={row.field} style={select}
        onChange={(e) => {
          const next = byId.get(e.target.value);
          if (next) set({ ...newRow(next), key: row.key, negated: row.negated });
        }}>
        {groups.map((g) => (
          <optgroup key={g} label={g}>
            {fields.filter((f) => f.group === g).map((f) => <option key={f.id} value={f.id}>{f.label}</option>)}
          </optgroup>
        ))}
      </select>
      {field && (
        <select aria-label="operador" value={row.op} style={select}
          onChange={(e) => set(withOp(row, e.target.value as RowOp, field))}>
          {opsFor(field.kind).map((op) => <option key={op} value={op}>{OP_LABELS[op]}</option>)}
        </select>
      )}
      {field && <ValueEditor row={row} field={field} onChange={set} />}
      <label style={small}>
        <input type="checkbox" checked={row.negated} onChange={(e) => set({ ...row, negated: e.target.checked })} />
        NÃO
      </label>
      <button type="button" style={secondaryButtonStyle} onClick={() => onRoot(removeNode(root, row.key))}>remover</button>
      {problem && <small style={hint}>{problem}</small>}
    </div>
  );
}

function numberOrNull(text: string): number | null {
  if (text.trim() === "") return null;
  const n = Number(text);
  return Number.isFinite(n) ? n : null;
}

function ValueEditor({ row, field, onChange }: { row: ConditionRow; field: ConditionField; onChange(next: ConditionRow): void }) {
  const unit = field.unit ? <span style={small}>{field.unit.trim()}</span> : null;

  if (row.op === "between") {
    return (
      <>
        <input type="number" aria-label="de" value={row.value[0] ?? ""} style={numberInput}
          onChange={(e) => onChange({ ...row, value: [ numberOrNull(e.target.value), row.value[1] ] })} />
        <span style={small}>e</span>
        <input type="number" aria-label="até" value={row.value[1] ?? ""} style={numberInput}
          onChange={(e) => onChange({ ...row, value: [ row.value[0], numberOrNull(e.target.value) ] })} />
        {unit}
      </>
    );
  }
  if (row.op === "gte" || row.op === "lte" || row.op === "eq") {
    return (
      <>
        <input type="number" aria-label="valor" value={row.value ?? ""} style={numberInput}
          onChange={(e) => onChange({ ...row, value: numberOrNull(e.target.value) })} />
        {unit}
      </>
    );
  }
  if (row.op === "is") {
    return (
      <select aria-label="valor" value={row.value} style={select} onChange={(e) => onChange({ ...row, value: e.target.value })}>
        {(field.options ?? []).map((o) => <option key={o.value} value={o.value}>{o.label}</option>)}
      </select>
    );
  }

  // Opção inativa só aparece se já estiver marcada (bairro desativado depois
  // da regra); valor que nem existe na lista aparece como "fora da lista".
  const known = field.options ?? [];
  const unknown = row.value.filter((v) => !known.some((o) => o.value === v)).map((v) => ({ value: v, label: `${v} (fora da lista)` }));
  const shown = [ ...known.filter((o) => !o.inactive || row.value.includes(o.value)), ...unknown ];
  if (shown.length === 0) return <small style={hint}>nenhuma opção disponível</small>;
  return (
    <span role="group" aria-label="valores" style={{ display: "inline-flex", gap: 8, flexWrap: "wrap" }}>
      {shown.map((o) => (
        <label key={o.value} style={small}>
          <input type="checkbox" checked={row.value.includes(o.value)}
            onChange={(e) => onChange({
              ...row, value: e.target.checked ? [ ...row.value, o.value ] : row.value.filter((v) => v !== o.value)
            })} />
          {o.label}
        </label>
      ))}
    </span>
  );
}

const box: CSSProperties = { border: "1px solid var(--rule)", borderRadius: 8, padding: "8px 12px", margin: 0, display: "flex", flexDirection: "column", gap: 8 };
const legend: CSSProperties = { fontSize: 13, fontWeight: 600 };
const phraseStyle: CSSProperties = { margin: 0, fontSize: 12.5 };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const stack: CSSProperties = { display: "flex", flexDirection: "column", gap: 8 };
const subgroup: CSSProperties = { ...stack, padding: 8, borderLeft: "3px solid var(--rule2)", background: "var(--sunken)", borderRadius: 6 };
const line: CSSProperties = { display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" };
const rowBox: CSSProperties = { ...line, paddingBottom: 6, borderBottom: "1px dashed var(--rule)" };
const small: CSSProperties = { fontSize: 12, color: "var(--ink2)", display: "inline-flex", gap: 4, alignItems: "center" };
const select: CSSProperties = { ...inputStyle, width: "auto", marginTop: 0, padding: 6, fontSize: 12.5 };
const numberInput: CSSProperties = { ...inputStyle, width: 80, marginTop: 0, padding: 6, fontSize: 12.5 };
```

- [ ] **Step 4: Rode e veja passar**

Run: `npx vitest run src/modules/protocols/ConditionBuilder.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/protocols/ConditionBuilder.tsx src/modules/protocols/ConditionBuilder.test.tsx
/opt/homebrew/bin/git commit -m "feat: add visual condition builder with live Portuguese sentence

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Oferta e sugestões na definição (regras puras)

**Files:**
- Create: `src/lib/offer.ts`
- Test: `src/lib/offer.test.ts`

**Interfaces:**
- Consumes: `ConditionTree` (Task 1).
- Produces:
  - tipos locais `OfferDraft { title: string; summary: string; eligibility: ConditionTree | null; retakeAfterDays: number | null }`, `SuggestionDraft { protocol: string; when: ConditionTree | null }`, `OfferRead = { ok: true; offer; suggestions } | { ok: false; reason: string }`;
  - constantes `TITLE_MAX = 60`, `SUMMARY_MAX = 200`, `RETAKE_MAX = 3650`, `SUGGESTIONS_MAX = 10`, `PROTOCOL_NAME_PATTERN = /^[a-z][a-z0-9-]+$/`, `RETAKE_PRESETS` ("sem intervalo" `null`, "6 meses" 180, "1 ano" 365);
  - `readOffer(definition): OfferRead`, `writeOffer(definition, offer): unknown`, `writeSuggestions(definition, list): unknown`, `protocolNameOf(definition): string | null`;
  - `parseRetake(text): { days: number | null; problem: string | null }`, `retakeLabel(days): string`, `suggestionProblem(s, ownName): string | null`.

- [ ] **Step 1: Escreva o teste**

```ts
// src/lib/offer.test.ts
import { describe, expect, it } from "vitest";
import {
  parseRetake, protocolNameOf, readOffer, retakeLabel, suggestionProblem, writeOffer, writeSuggestions
} from "./offer";

const BASE = { name: "saude-do-idoso", version: 2, start_step_id: "s1", steps: [ { id: "s1", prompt: "?", answer_type: "boolean" } ] };
const FULL = {
  ...BASE,
  offer: { title: "Saúde do idoso", summary: "Quedas e memória.", eligibility: { gte: [ "profile.age", 60 ] }, retake_after_days: 365 },
  suggestions: [ { protocol: "saude-mental", when: { gte: [ "outcome.score", 15 ] } } ]
};
const EMPTY_OFFER = { title: "", summary: "", eligibility: null, retakeAfterDays: null };

describe("leitura", () => {
  it("sem offer nem suggestions: tudo vazio", () => {
    expect(readOffer(BASE)).toEqual({ ok: true, offer: EMPTY_OFFER, suggestions: [] });
  });

  it("lê o bloco inteiro", () => {
    expect(readOffer(FULL)).toEqual({
      ok: true,
      offer: { title: "Saúde do idoso", summary: "Quedas e memória.", eligibility: { gte: [ "profile.age", 60 ] }, retakeAfterDays: 365 },
      suggestions: [ { protocol: "saude-mental", when: { gte: [ "outcome.score", 15 ] } } ]
    });
  });

  it("sugestão sem when ou sem protocol é lida como incompleta", () => {
    expect(readOffer({ ...BASE, suggestions: [ {} ] })).toMatchObject({ ok: true, suggestions: [ { protocol: "", when: null } ] });
  });

  it.each([
    [ "definição que não é objeto", [], "a definição precisa ser um objeto JSON" ],
    [ "offer que não é objeto", { ...BASE, offer: "sim" }, "“offer” não é um objeto: corrija no JSON" ],
    [ "título que não é texto", { ...BASE, offer: { title: 3 } }, "“offer.title” precisa ser texto: corrija no JSON" ],
    [ "resumo que não é texto", { ...BASE, offer: { summary: [] } }, "“offer.summary” precisa ser texto: corrija no JSON" ],
    [ "elegibilidade que não é objeto", { ...BASE, offer: { eligibility: "60+" } }, "“offer.eligibility” precisa ser uma condição: corrija no JSON" ],
    [ "intervalo quebrado", { ...BASE, offer: { retake_after_days: 1.5 } }, "“offer.retake_after_days” precisa ser um número inteiro de dias: corrija no JSON" ],
    [ "suggestions que não é lista", { ...BASE, suggestions: {} }, "“suggestions” não é uma lista: corrija no JSON" ],
    [ "item fora do formato", { ...BASE, suggestions: [ { protocol: 1 } ] }, "uma sugestão está fora do formato { protocol, when }: corrija no JSON" ]
  ])("%s → recusa com motivo", (_name, def, reason) => {
    expect(readOffer(def)).toEqual({ ok: false, reason });
  });
});

describe("escrita", () => {
  it("campo vazio sai ausente; offer vazio sai inteiro; o resto da definição fica", () => {
    const next = writeOffer(FULL, { ...EMPTY_OFFER, title: "Idoso" }) as Record<string, unknown>;
    expect(next.offer).toEqual({ title: "Idoso" });
    expect(next.steps).toBe(FULL.steps);
    expect("offer" in (writeOffer(FULL, EMPTY_OFFER) as Record<string, unknown>)).toBe(false);
  });

  it("ida e volta mantém o bloco", () => {
    const read = readOffer(FULL);
    if (!read.ok) throw new Error("leitura falhou");
    expect(writeSuggestions(writeOffer(FULL, read.offer), read.suggestions)).toEqual(FULL);
  });

  it("sugestão sem condição sai sem when; lista vazia tira a chave", () => {
    expect((writeSuggestions(BASE, [ { protocol: "x-y", when: null } ]) as Record<string, unknown>).suggestions)
      .toEqual([ { protocol: "x-y" } ]);
    expect("suggestions" in (writeSuggestions(FULL, []) as Record<string, unknown>)).toBe(false);
  });

  it("nome do protocolo", () => {
    expect(protocolNameOf(FULL)).toBe("saude-do-idoso");
    expect(protocolNameOf({})).toBeNull();
  });
});

describe("intervalo e sugestões", () => {
  it("intervalo: vazio é sem intervalo; só inteiros de 1 a 3650", () => {
    expect(parseRetake("")).toEqual({ days: null, problem: null });
    expect(parseRetake(" 365 ")).toEqual({ days: 365, problem: null });
    expect(parseRetake("0")).toEqual({ days: null, problem: "use de 1 a 3650 dias" });
    expect(parseRetake("3651")).toEqual({ days: null, problem: "use de 1 a 3650 dias" });
    expect(parseRetake("1,5")).toEqual({ days: null, problem: "use um número inteiro de dias" });
  });

  it("rótulo do intervalo usa os atalhos", () => {
    expect(retakeLabel(null)).toBe("sem intervalo: pode refazer quando quiser");
    expect(retakeLabel(180)).toBe("6 meses (180 dias)");
    expect(retakeLabel(365)).toBe("1 ano (365 dias)");
    expect(retakeLabel(1)).toBe("1 dia");
    expect(retakeLabel(90)).toBe("90 dias");
  });

  it("sugestão incompleta, com nome inválido ou para si mesma é apontada", () => {
    const when = { gte: [ "outcome.score", 15 ] };
    expect(suggestionProblem({ protocol: "", when }, "a-b")).toBe("escolha o protocolo sugerido");
    expect(suggestionProblem({ protocol: "Saude", when }, "a-b")).toBe("nome de protocolo inválido (minúsculas, números e hífen)");
    expect(suggestionProblem({ protocol: "a-b", when }, "a-b")).toBe("um protocolo não pode sugerir a si mesmo");
    expect(suggestionProblem({ protocol: "c-d", when: null }, "a-b")).toBe("defina quando sugerir");
    expect(suggestionProblem({ protocol: "c-d", when }, "a-b")).toBeNull();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/offer.test.ts`
Expected: FAIL — `./offer` não existe.

- [ ] **Step 3: Implemente**

```ts
// src/lib/offer.ts
// Bloco `offer` e lista `suggestions` do protocolo (módulo 15; schema
// protocols-v1.4.0; contratos §1). O JSON do editor é a fonte: estas funções
// leem a definição e devolvem uma definição nova, sem tocar no resto. A
// definição é `unknown` no dashboard (src/lib/editor.ts) e o contracts não
// publica tipos TS, então os tipos daqui são locais.
import type { ConditionTree } from "./api";

export interface OfferDraft {
  title: string;
  summary: string;
  eligibility: ConditionTree | null;
  retakeAfterDays: number | null;
}
export interface SuggestionDraft { protocol: string; when: ConditionTree | null }
export type OfferRead =
  | { ok: true; offer: OfferDraft; suggestions: SuggestionDraft[] }
  | { ok: false; reason: string };

export const TITLE_MAX = 60;
export const SUMMARY_MAX = 200;
export const RETAKE_MAX = 3650;
export const SUGGESTIONS_MAX = 10;
// O mesmo padrão do `name` do protocolo (contratos §1).
export const PROTOCOL_NAME_PATTERN = /^[a-z][a-z0-9-]+$/;
export const RETAKE_PRESETS: Array<{ label: string; days: number | null }> = [
  { label: "sem intervalo", days: null },
  { label: "6 meses", days: 180 },
  { label: "1 ano", days: 365 }
];

type Obj = Record<string, unknown>;
const isObj = (v: unknown): v is Obj => !!v && typeof v === "object" && !Array.isArray(v);
const fail = (reason: string): OfferRead => ({ ok: false, reason });

export function readOffer(definition: unknown): OfferRead {
  if (!isObj(definition)) return fail("a definição precisa ser um objeto JSON");
  const raw = definition.offer;
  if (raw !== undefined && !isObj(raw)) return fail("“offer” não é um objeto: corrija no JSON");
  const o: Obj = isObj(raw) ? raw : {};
  if (o.title !== undefined && typeof o.title !== "string") return fail("“offer.title” precisa ser texto: corrija no JSON");
  if (o.summary !== undefined && typeof o.summary !== "string") return fail("“offer.summary” precisa ser texto: corrija no JSON");
  if (o.eligibility !== undefined && !isObj(o.eligibility)) {
    return fail("“offer.eligibility” precisa ser uma condição: corrija no JSON");
  }
  if (o.retake_after_days !== undefined && !Number.isInteger(o.retake_after_days)) {
    return fail("“offer.retake_after_days” precisa ser um número inteiro de dias: corrija no JSON");
  }
  const list = definition.suggestions;
  if (list !== undefined && !Array.isArray(list)) return fail("“suggestions” não é uma lista: corrija no JSON");
  const items: unknown[] = Array.isArray(list) ? list : [];
  const malformed = items.some((s) => !isObj(s) ||
    (s.protocol !== undefined && typeof s.protocol !== "string") || (s.when !== undefined && !isObj(s.when)));
  if (malformed) return fail("uma sugestão está fora do formato { protocol, when }: corrija no JSON");

  return {
    ok: true,
    offer: {
      title: (o.title as string | undefined) ?? "",
      summary: (o.summary as string | undefined) ?? "",
      eligibility: (o.eligibility as ConditionTree | undefined) ?? null,
      retakeAfterDays: (o.retake_after_days as number | undefined) ?? null
    },
    suggestions: (items as Obj[]).map((s) => ({
      protocol: (s.protocol as string | undefined) ?? "",
      when: (s.when as ConditionTree | undefined) ?? null
    }))
  };
}

// Campo vazio sai ausente (o schema recusa "" e null); offer vazio sai inteiro.
export function writeOffer(definition: unknown, offer: OfferDraft): unknown {
  const next: Obj = { ...(definition as Obj) };
  const block: Obj = {};
  if (offer.title !== "") block.title = offer.title;
  if (offer.summary !== "") block.summary = offer.summary;
  if (offer.eligibility !== null) block.eligibility = offer.eligibility;
  if (offer.retakeAfterDays !== null) block.retake_after_days = offer.retakeAfterDays;
  if (Object.keys(block).length === 0) delete next.offer;
  else next.offer = block;
  return next;
}

// Sugestão sem condição vai sem `when`: o gate do api aponta o erro, e o
// painel mostra "defina quando sugerir" ao lado.
export function writeSuggestions(definition: unknown, list: SuggestionDraft[]): unknown {
  const next: Obj = { ...(definition as Obj) };
  if (list.length === 0) delete next.suggestions;
  else next.suggestions = list.map((s) => (s.when === null ? { protocol: s.protocol } : { protocol: s.protocol, when: s.when }));
  return next;
}

export function protocolNameOf(definition: unknown): string | null {
  return isObj(definition) && typeof definition.name === "string" ? definition.name : null;
}

export function parseRetake(text: string): { days: number | null; problem: string | null } {
  const t = text.trim();
  if (t === "") return { days: null, problem: null };
  if (!/^\d+$/.test(t)) return { days: null, problem: "use um número inteiro de dias" };
  const n = Number(t);
  if (n < 1 || n > RETAKE_MAX) return { days: null, problem: `use de 1 a ${RETAKE_MAX} dias` };
  return { days: n, problem: null };
}

export function retakeLabel(days: number | null): string {
  if (days === null) return "sem intervalo: pode refazer quando quiser";
  const preset = RETAKE_PRESETS.find((p) => p.days === days);
  if (preset) return `${preset.label} (${days} dias)`;
  return `${days} ${days === 1 ? "dia" : "dias"}`;
}

export function suggestionProblem(s: SuggestionDraft, ownName: string | null): string | null {
  if (s.protocol === "") return "escolha o protocolo sugerido";
  if (!PROTOCOL_NAME_PATTERN.test(s.protocol)) return "nome de protocolo inválido (minúsculas, números e hífen)";
  if (s.protocol === ownName) return "um protocolo não pode sugerir a si mesmo";
  if (s.when === null) return "defina quando sugerir";
  return null;
}
```

- [ ] **Step 4: Rode e veja passar**

Run: `npx vitest run src/lib/offer.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/offer.ts src/lib/offer.test.ts
/opt/homebrew/bin/git commit -m "feat: read and write protocol offer and suggestions blocks

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 7: Painel "Oferta e sugestões" no editor de protocolo

**Files:**
- Create: `src/modules/protocolEditor/OfferPanel.tsx`
- Modify: `src/modules/ProtocolEditor.tsx`
- Test: `src/modules/ProtocolEditor.test.tsx` (novo `describe`)

**Interfaces:**
- Consumes: `ConditionBuilder` (Task 5); `fieldsFor` (Task 3); tudo de `src/lib/offer.ts` (Task 6); `listAuthorProtocols` (existente) para a lista de nomes.
- Produces: `OfferPanel({ definition: unknown | null; protocolNames: string[]; onChange(next: unknown): void })`.
  - `definition === null` (JSON inválido) → só o aviso "Corrija o JSON para editar a oferta e as sugestões.";
  - `readOffer` recusou → o motivo em `role="alert"`, sem controles;
  - controles: "Título no catálogo", "Resumo", construtor "Quem pode fazer (elegibilidade)", grupo "Intervalo para refazer" com o campo "Intervalo para refazer (dias)" e os atalhos "sem intervalo", "6 meses", "1 ano"; cada sugestão é `role="group"` "sugestão N" com "Protocolo sugerido", construtor "Quando sugerir" e "remover sugestão"; botão "+ sugestão" (até 10).
- O JSON continua a fonte: cada edição chama `onChange(writeOffer(...))` ou `onChange(writeSuggestions(...))`, e o `ProtocolEditor` grava o texto formatado, como no `AnalyticQuestions`.

- [ ] **Step 1: Escreva os testes (fim de `src/modules/ProtocolEditor.test.tsx`)**

```tsx
describe("ProtocolEditor — Oferta e sugestões", () => {
  beforeEach(() => {
    vi.resetAllMocks();
    mocked(api.listAuthorProtocols).mockResolvedValue([
      { name: "saude-mental-aprofundada", version: "1", status: "active" },
      { name: "arbovirose", version: "3", status: "draft" }
    ]);
    mocked(api.gateProtocol).mockResolvedValue({ valid: true });
  });

  const offerBox = () => screen.getByRole("group", { name: "Quem pode fazer (elegibilidade)" });
  const json = () => JSON.parse(definitionBox().value);

  it("JSON com offer preenche o painel; digitar o título grava no JSON e apagar tira o bloco", () => {
    render(<ProtocolEditor />);
    typeDefinition({ ...DEF, offer: { title: "Arboviroses" } });
    const title = screen.getByLabelText("Título no catálogo") as HTMLInputElement;
    expect(title.value).toBe("Arboviroses");
    fireEvent.change(title, { target: { value: "Dengue e chikungunya" } });
    expect(json().offer).toEqual({ title: "Dengue e chikungunya" });
    fireEvent.change(title, { target: { value: "" } });
    expect("offer" in json()).toBe(false);
  });

  it("elegibilidade montada no construtor vai para o JSON, e a frase acompanha", () => {
    render(<ProtocolEditor />);
    typeDefinition(DEF);
    fireEvent.click(within(offerBox()).getByRole("button", { name: "+ condição" }));
    fireEvent.change(within(offerBox()).getByLabelText("valor"), { target: { value: "60" } });
    expect(json().offer).toEqual({ eligibility: { gte: [ "profile.age", 60 ] } });
    expect(within(offerBox()).getByText("idade a partir de 60 anos")).not.toBeNull();
  });

  it("editar o JSON atualiza o construtor", () => {
    render(<ProtocolEditor />);
    typeDefinition({ ...DEF, offer: { eligibility: { gte: [ "profile.age", 65 ] } } });
    expect((within(offerBox()).getByLabelText("valor") as HTMLInputElement).value).toBe("65");
    typeDefinition({ ...DEF, offer: { eligibility: { gte: [ "profile.age", 70 ] } } });
    expect((within(offerBox()).getByLabelText("valor") as HTMLInputElement).value).toBe("70");
  });

  it("regra avançada no JSON: painel em leitura e o texto não muda", () => {
    render(<ProtocolEditor />);
    const def = { ...DEF, offer: { eligibility: { gt: [ "profile.age", 59 ] } } };
    typeDefinition(def);
    expect(within(offerBox()).getByText("Regra avançada: o construtor não edita esta regra. Altere no JSON.")).not.toBeNull();
    expect(within(offerBox()).getByText("idade acima de 59 anos")).not.toBeNull();
    expect(definitionBox().value).toBe(JSON.stringify(def, null, 2));
  });

  it("atalho '1 ano' grava 365; valor inválido não grava; 'sem intervalo' tira a chave", () => {
    render(<ProtocolEditor />);
    typeDefinition(DEF);
    fireEvent.click(screen.getByRole("button", { name: "1 ano" }));
    expect(json().offer).toEqual({ retake_after_days: 365 });
    expect(screen.getByText("1 ano (365 dias)")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Intervalo para refazer (dias)"), { target: { value: "0" } });
    expect(screen.getByText("use de 1 a 3650 dias")).not.toBeNull();
    expect(json().offer).toEqual({ retake_after_days: 365 });
    fireEvent.click(screen.getByRole("button", { name: "sem intervalo" }));
    expect("offer" in json()).toBe(false);
  });

  it("sugestão: protocolo da cidade e condição sobre a pontuação; o próprio protocolo não é oferecido", async () => {
    render(<ProtocolEditor />);
    typeDefinition({ ...DEF, scoring: { type: "weighted", thresholds: { baixa: 0 } } });
    fireEvent.click(screen.getByRole("button", { name: "+ sugestão" }));
    const card = screen.getByRole("group", { name: "sugestão 1" });
    expect(within(card).getByText("escolha o protocolo sugerido")).not.toBeNull();
    const select = within(card).getByLabelText("Protocolo sugerido") as HTMLSelectElement;
    await waitFor(() => expect(within(select).queryByText("saude-mental-aprofundada")).not.toBeNull());
    expect(within(select).queryByText("arbovirose")).toBeNull();
    fireEvent.change(select, { target: { value: "saude-mental-aprofundada" } });
    const when = within(card).getByRole("group", { name: "Quando sugerir" });
    fireEvent.click(within(when).getByRole("button", { name: "+ condição" }));
    fireEvent.change(within(when).getByLabelText("campo"), { target: { value: "outcome.score" } });
    fireEvent.change(within(when).getByLabelText("valor"), { target: { value: "15" } });
    expect(json().suggestions).toEqual([ { protocol: "saude-mental-aprofundada", when: { gte: [ "outcome.score", 15 ] } } ]);
    expect(within(card).queryByText("defina quando sugerir")).toBeNull();
    fireEvent.click(within(card).getByRole("button", { name: "remover sugestão" }));
    expect("suggestions" in json()).toBe(false);
  });

  it("JSON inválido: o painel pede correção", () => {
    render(<ProtocolEditor />);
    fireEvent.change(definitionBox(), { target: { value: "{ quebrado" } });
    expect(screen.getByText("Corrija o JSON para editar a oferta e as sugestões.")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/ProtocolEditor.test.tsx`
Expected: FAIL — "Título no catálogo" não existe (os testes antigos do `describe` de Analytics continuam passando).

- [ ] **Step 3: Implemente o painel**

```tsx
// src/modules/protocolEditor/OfferPanel.tsx
// Painel "Oferta e sugestões" (módulo 15; spec §7), ao lado do JSON. Lê
// `offer` e `suggestions` da definição a cada render e devolve uma definição
// nova a cada edição: o JSON continua a fonte, e o que muda num lado aparece
// no outro. O conteúdo é parte da versão assinada (ADR 0016).
import { useEffect, useMemo, useState, type CSSProperties, type ReactNode } from "react";
import { ConditionBuilder } from "../protocols/ConditionBuilder";
import { fieldsFor } from "../../lib/condition";
import {
  RETAKE_PRESETS, SUGGESTIONS_MAX, SUMMARY_MAX, TITLE_MAX, parseRetake, protocolNameOf, readOffer, retakeLabel,
  suggestionProblem, writeOffer, writeSuggestions, type OfferDraft, type SuggestionDraft
} from "../../lib/offer";
import { disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";

export function OfferPanel({ definition, protocolNames, onChange }: {
  definition: unknown | null; protocolNames: string[]; onChange(next: unknown): void;
}) {
  if (definition === null) return <Section><p style={hint}>Corrija o JSON para editar a oferta e as sugestões.</p></Section>;
  const read = readOffer(definition);
  if (!read.ok) return <Section><p role="alert" style={alert}>{read.reason}</p></Section>;
  return (
    <OfferForm definition={definition} offer={read.offer} suggestions={read.suggestions}
      protocolNames={protocolNames} onChange={onChange} />
  );
}

function OfferForm({ definition, offer, suggestions, protocolNames, onChange }: {
  definition: unknown; offer: OfferDraft; suggestions: SuggestionDraft[]; protocolNames: string[]; onChange(next: unknown): void;
}) {
  const ownName = protocolNameOf(definition);
  const eligibilityFields = useMemo(() => fieldsFor("eligibility"), []);
  const suggestionFields = fieldsFor("suggestion", { definition });
  const choices = protocolNames.filter((n) => n !== ownName);
  const setOffer = (patch: Partial<OfferDraft>) => onChange(writeOffer(definition, { ...offer, ...patch }));
  const setSuggestions = (list: SuggestionDraft[]) => onChange(writeSuggestions(definition, list));
  const replace = (i: number, next: SuggestionDraft) => setSuggestions(suggestions.map((s, j) => (j === i ? next : s)));

  return (
    <Section>
      <label style={label}>
        Título no catálogo
        <input value={offer.title} maxLength={TITLE_MAX} style={inputStyle} onChange={(e) => setOffer({ title: e.target.value })} />
      </label>
      <small style={hint}>{offer.title.length}/{TITLE_MAX} · sem título, o cidadão vê o nome do protocolo</small>
      <label style={label}>
        Resumo
        <textarea value={offer.summary} maxLength={SUMMARY_MAX} style={{ ...inputStyle, minHeight: 56 }}
          onChange={(e) => setOffer({ summary: e.target.value })} />
      </label>
      <small style={hint}>{offer.summary.length}/{SUMMARY_MAX}</small>

      <ConditionBuilder label="Quem pode fazer (elegibilidade)" fields={eligibilityFields} value={offer.eligibility}
        emptyText="para todos" onChange={(tree) => setOffer({ eligibility: tree })} />

      <RetakeField days={offer.retakeAfterDays} onChange={(days) => setOffer({ retakeAfterDays: days })} />

      <h3 style={{ fontSize: 14, margin: "8px 0 0" }}>Sugestões ao fim da triagem</h3>
      <p style={hint}>Aparecem em “Recomendamos também” e ficam pendentes no catálogo do cidadão. Resultado urgente nunca sugere.</p>
      {suggestions.map((s, i) => {
        const problem = suggestionProblem(s, ownName);
        return (
          <div key={i} role="group" aria-label={`sugestão ${i + 1}`} style={card}>
            <label style={label}>
              Protocolo sugerido
              <select value={s.protocol} style={inputStyle} onChange={(e) => replace(i, { ...s, protocol: e.target.value })}>
                <option value="">escolha…</option>
                {choices.map((n) => <option key={n} value={n}>{n}</option>)}
                {s.protocol !== "" && !choices.includes(s.protocol) && (
                  <option value={s.protocol}>
                    {s.protocol} {s.protocol === ownName ? "(este protocolo)" : "(não existe nesta cidade)"}
                  </option>
                )}
              </select>
            </label>
            <ConditionBuilder label="Quando sugerir" fields={suggestionFields} value={s.when} emptyText="—"
              onChange={(tree) => replace(i, { ...s, when: tree })} />
            {problem && <small style={hint}>{problem}</small>}
            <div>
              <button type="button" style={secondaryButtonStyle}
                onClick={() => setSuggestions(suggestions.filter((_, j) => j !== i))}>remover sugestão</button>
            </div>
          </div>
        );
      })}
      <div>
        <button type="button" disabled={suggestions.length >= SUGGESTIONS_MAX}
          style={suggestions.length >= SUGGESTIONS_MAX ? disabledButtonStyle : secondaryButtonStyle}
          onClick={() => setSuggestions([ ...suggestions, { protocol: "", when: null } ])}>+ sugestão</button>
      </div>
    </Section>
  );
}

// O texto digitado fica local: "0" ou "1,5" mostram o motivo e não chegam ao
// JSON. Só um valor válido (ou o campo vazio) grava.
function RetakeField({ days, onChange }: { days: number | null; onChange(days: number | null): void }) {
  const [ text, setText ] = useState(days === null ? "" : String(days));
  useEffect(() => {
    if (parseRetake(text).days !== days) setText(days === null ? "" : String(days));
  }, [ days ]);
  const { problem } = parseRetake(text);

  function type(value: string) {
    setText(value);
    const parsed = parseRetake(value);
    if (!parsed.problem) onChange(parsed.days);
  }

  return (
    <div role="group" aria-label="Intervalo para refazer" style={{ display: "flex", flexDirection: "column", gap: 6 }}>
      <label style={label}>
        Intervalo para refazer (dias)
        <input inputMode="numeric" value={text} style={inputStyle} onChange={(e) => type(e.target.value)} />
      </label>
      <div style={{ display: "flex", gap: 8, flexWrap: "wrap" }}>
        {RETAKE_PRESETS.map((p) => (
          <button key={p.label} type="button" aria-pressed={p.days === days} style={secondaryButtonStyle}
            onClick={() => { setText(p.days === null ? "" : String(p.days)); onChange(p.days); }}>
            {p.label}
          </button>
        ))}
      </div>
      {problem ? <small role="alert" style={alert}>{problem}</small> : <small style={hint}>{retakeLabel(days)}</small>}
    </div>
  );
}

function Section({ children }: { children: ReactNode }) {
  return <section aria-label="Oferta e sugestões" style={{ display: "flex", flexDirection: "column", gap: 8 }}>{children}</section>;
}

const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12, color: "var(--down)" };
const card: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, padding: 10, border: "1px solid var(--rule)", borderRadius: 8 };
```

- [ ] **Step 4: Ligue o painel no editor**

Em `src/modules/ProtocolEditor.tsx`, acrescente o import depois do de `AnalyticQuestions`:

```tsx
import { OfferPanel } from "./protocolEditor/OfferPanel";
```

Depois de `const current = parseDefinition(text);`, acrescente:

```tsx
  // Nomes de protocolo da cidade para a sugestão (um nome por protocolo, sem a versão).
  const protocolNames = [ ...new Set(opts.map((o) => o.name)) ].sort();
```

Troque o começo da segunda coluna:

```tsx
      <section>
        <h2 style={{ fontSize: 16, margin: "0 0 8px" }}>Preview ao vivo</h2>
```

por:

```tsx
      <section>
        <h2 style={{ fontSize: 16, margin: "0 0 8px" }}>Oferta e sugestões</h2>
        <OfferPanel
          definition={current.ok ? current.value : null}
          protocolNames={protocolNames}
          onChange={(next) => setText(JSON.stringify(next, null, 2))}
        />
        <h2 style={{ fontSize: 16, margin: "16px 0 8px" }}>Preview ao vivo</h2>
```

- [ ] **Step 5: Rode e veja passar**

Run: `npx vitest run src/modules/ProtocolEditor.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes de Analytics continuam verdes: a caixa do JSON segue sendo o primeiro `textbox`).

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/modules/protocolEditor/OfferPanel.tsx src/modules/ProtocolEditor.tsx src/modules/ProtocolEditor.test.tsx
/opt/homebrew/bin/git commit -m "feat: edit protocol offer and suggestions beside the JSON

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Perguntas no visual do wpda

**Files:**
- Create: `src/modules/protocolEditor/wpdaLook.ts`, `src/modules/protocolEditor/QuestionPreview.tsx`
- Modify: `src/modules/ProtocolEditor.tsx`
- Test: `src/modules/protocolEditor/QuestionPreview.test.tsx`, `src/modules/ProtocolEditor.test.tsx` (um caso)

**Interfaces:**
- Consumes: nada de tasks anteriores além do `ProtocolEditor`.
- Produces:
  - `WPDA: Record<string, CSSProperties>` (medidas de `apps/wpda/src/modules/citizen/ui.tsx`, cores pelas variáveis CSS comuns);
  - `previewSteps(definition: unknown): PreviewStep[]` com `PreviewStep { id; prompt; answerType; options: string[] }` (boolean → `["Sim", "Não"]`, como o `Citizens::StepPayload` com `citizen.yes`/`citizen.no`);
  - `QuestionPreview({ definition: unknown | null })`: `section` "Como o cidadão vê", uma pergunta por vez, na ordem do JSON (sem seguir ramificações), botões "← pergunta anterior" e "próxima pergunta →".

- [ ] **Step 1: Escreva o teste**

```tsx
// src/modules/protocolEditor/QuestionPreview.test.tsx
import { afterEach, describe, expect, it } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { QuestionPreview, previewSteps } from "./QuestionPreview";

afterEach(cleanup);

const DEF = {
  name: "dor", version: 1, start_step_id: "febre",
  steps: [
    { id: "febre", prompt: "Teve febre?", answer_type: "boolean" },
    { id: "dias", prompt: "Há quantos dias?", answer_type: "integer" },
    { id: "onde", prompt: "Onde dói?", answer_type: "enum", options: [ "cabeça", "barriga" ] },
    { id: "obs", prompt: "Conte mais", answer_type: "text" }
  ]
};

describe("previewSteps", () => {
  it("sim/não vira Sim e Não; lista usa as opções; passo sem texto usa o id; lixo some", () => {
    expect(previewSteps({ steps: [ { id: "a", answer_type: "boolean" }, null, "x", { id: "b", prompt: "B?", answer_type: "enum", options: [ "1", 2 ] } ] }))
      .toEqual([
        { id: "a", prompt: "a", answerType: "boolean", options: [ "Sim", "Não" ] },
        { id: "b", prompt: "B?", answerType: "enum", options: [ "1" ] }
      ]);
    expect(previewSteps(null)).toEqual([]);
  });
});

describe("QuestionPreview", () => {
  it("primeira pergunta como no wpda: título, contagem, Sim e Não, sem Continuar nem Voltar", () => {
    render(<QuestionPreview definition={DEF} />);
    expect(screen.getByRole("heading", { name: "Teve febre?" })).not.toBeNull();
    expect(screen.getByText("Pergunta 1 de 4")).not.toBeNull();
    expect(screen.getByText("Sim")).not.toBeNull();
    expect(screen.getByText("Não")).not.toBeNull();
    expect(screen.queryByText("Continuar")).toBeNull();
    expect(screen.queryByText("Voltar")).toBeNull();
    expect((screen.getByRole("button", { name: "← pergunta anterior" }) as HTMLButtonElement).disabled).toBe(true);
    expect(screen.getByText("Em emergência, ligue 192.")).not.toBeNull();
  });

  it("número: campo, Continuar e Voltar", () => {
    render(<QuestionPreview definition={DEF} />);
    fireEvent.click(screen.getByRole("button", { name: "próxima pergunta →" }));
    expect(screen.getByRole("heading", { name: "Há quantos dias?" })).not.toBeNull();
    expect(screen.getByLabelText("campo do cidadão")).not.toBeNull();
    expect(screen.getByText("Continuar")).not.toBeNull();
    expect(screen.getByText("Voltar")).not.toBeNull();
  });

  it("lista mostra as opções; na última a próxima trava", () => {
    render(<QuestionPreview definition={DEF} />);
    fireEvent.click(screen.getByRole("button", { name: "próxima pergunta →" }));
    fireEvent.click(screen.getByRole("button", { name: "próxima pergunta →" }));
    expect(screen.getByText("cabeça")).not.toBeNull();
    expect(screen.getByText("barriga")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "próxima pergunta →" }));
    expect(screen.getByText("Pergunta 4 de 4")).not.toBeNull();
    expect((screen.getByRole("button", { name: "próxima pergunta →" }) as HTMLButtonElement).disabled).toBe(true);
  });

  it("pergunta removida no JSON: a posição recua para a última que existe", () => {
    const { rerender } = render(<QuestionPreview definition={DEF} />);
    fireEvent.click(screen.getByRole("button", { name: "próxima pergunta →" }));
    fireEvent.click(screen.getByRole("button", { name: "próxima pergunta →" }));
    rerender(<QuestionPreview definition={{ ...DEF, steps: DEF.steps.slice(0, 1) }} />);
    expect(screen.getByText("Pergunta 1 de 1")).not.toBeNull();
  });

  it("sem perguntas não desenha nada", () => {
    const { container } = render(<QuestionPreview definition={null} />);
    expect(container.innerHTML).toBe("");
  });
});
```

E, no fim do `describe("ProtocolEditor — Oferta e sugestões")` da Task 7:

```tsx
  it("as perguntas aparecem no visual do wpda", () => {
    render(<ProtocolEditor />);
    typeDefinition(DEF);
    const preview = screen.getByRole("region", { name: "Como o cidadão vê" });
    expect(within(preview).getByRole("heading", { name: "Teve febre?" })).not.toBeNull();
  });
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/protocolEditor/QuestionPreview.test.tsx src/modules/ProtocolEditor.test.tsx`
Expected: FAIL — `./QuestionPreview` não existe.

- [ ] **Step 3: Implemente**

```ts
// src/modules/protocolEditor/wpdaLook.ts
import type { CSSProperties } from "react";

// Medidas do canal do cidadão, copiadas de apps/wpda/src/modules/citizen/ui.tsx
// (Screen, BigButton, Field): texto de 18 px, alvos de 56 px, cantos de 12 px.
// As cores são as variáveis CSS que o dashboard e o wpda definem com os mesmos
// valores (theme/tokens.ts e theme/global.css dos dois apps). O
// contracts/design-tokens ainda é só um README; nada aqui importa código do wpda.
export const WPDA: Record<string, CSSProperties> = {
  screen: {
    maxWidth: 360, padding: 16, border: "1px solid var(--rule2)", borderRadius: 16, background: "var(--panel)",
    fontFamily: "system-ui, sans-serif", fontSize: 18, color: "var(--ink)"
  },
  brand: { fontSize: 14, color: "var(--ink3)" },
  title: { fontSize: 24, margin: "4px 0 16px" },
  counter: { margin: "0 0 8px", color: "var(--ink2)" },
  progress: { width: "100%", height: 8, marginBottom: 16 },
  button: { minHeight: 56, width: "100%", borderRadius: 12, fontSize: 18, fontWeight: 600, cursor: "default" },
  primary: { background: "var(--accent)", color: "#fff", border: "none" },
  secondary: { background: "transparent", color: "var(--ink)", border: "1px solid var(--rule2)" },
  field: { minHeight: 56, fontSize: 20, padding: "0 12px", borderRadius: 12, border: "1px solid var(--rule2)" },
  footer: { fontSize: 14, color: "var(--ink3)", textAlign: "center" }
};
```

```tsx
// src/modules/protocolEditor/QuestionPreview.tsx
// Pré-visualização das perguntas no visual do wpda (módulo 15; spec §7). Uma
// pergunta por vez, na ordem do JSON, sem seguir ramificações: o que se
// confere aqui é texto e opções. Nenhum botão do "celular" responde nada.
import { useState } from "react";
import { secondaryButtonStyle, disabledButtonStyle } from "../../components/formStyles";
import { WPDA } from "./wpdaLook";

export interface PreviewStep { id: string; prompt: string; answerType: string; options: string[] }

export function previewSteps(definition: unknown): PreviewStep[] {
  const steps = definition && typeof definition === "object" ? (definition as { steps?: unknown }).steps : undefined;
  if (!Array.isArray(steps)) return [];
  return steps
    .filter((s): s is Record<string, unknown> => !!s && typeof s === "object" && !Array.isArray(s))
    .map((s) => {
      const id = String(s.id ?? "");
      const answerType = typeof s.answer_type === "string" ? s.answer_type : "";
      const options = answerType === "boolean"
        ? [ "Sim", "Não" ]
        : Array.isArray(s.options) ? s.options.filter((o): o is string => typeof o === "string") : [];
      return { id, prompt: typeof s.prompt === "string" && s.prompt ? s.prompt : id, answerType, options };
    });
}

export function QuestionPreview({ definition }: { definition: unknown | null }) {
  const steps = previewSteps(definition);
  const [ index, setIndex ] = useState(0);
  if (steps.length === 0) return null;
  const i = Math.min(index, steps.length - 1);
  const step = steps[i];
  const choice = step.answerType === "boolean" || step.answerType === "enum";
  const typed = step.answerType === "integer" || step.answerType === "text";
  const nav = (disabled: boolean) => (disabled ? disabledButtonStyle : secondaryButtonStyle);

  return (
    <section aria-label="Como o cidadão vê" style={{ marginTop: 16, display: "flex", flexDirection: "column", gap: 8 }}>
      <h3 style={{ fontSize: 14, margin: 0 }}>Como o cidadão vê (wpda)</h3>
      <div style={WPDA.screen}>
        <strong style={WPDA.brand}>Rota Saúde</strong>
        <h4 style={WPDA.title}>{step.prompt}</h4>
        <p style={WPDA.counter}>Pergunta {i + 1} de {steps.length}</p>
        <progress value={i + 1} max={steps.length} style={WPDA.progress} />
        {choice && (
          <div style={{ display: "grid", gap: 12 }}>
            {step.options.map((o) => (
              <button key={o} type="button" tabIndex={-1} style={{ ...WPDA.button, ...WPDA.secondary }}>{o}</button>
            ))}
          </div>
        )}
        {typed && (
          <div style={{ display: "grid", gap: 6 }}>
            <span style={{ fontWeight: 600 }}>{step.prompt}</span>
            <input aria-label="campo do cidadão" readOnly tabIndex={-1} style={WPDA.field}
              inputMode={step.answerType === "integer" ? "numeric" : undefined} />
          </div>
        )}
        <div style={{ display: "grid", gap: 12, paddingTop: 16 }}>
          {!choice && <button type="button" tabIndex={-1} style={{ ...WPDA.button, ...WPDA.primary }}>Continuar</button>}
          {i > 0 && <button type="button" tabIndex={-1} style={{ ...WPDA.button, ...WPDA.secondary }}>Voltar</button>}
        </div>
        <p style={WPDA.footer}>Em emergência, ligue 192.</p>
      </div>
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={i === 0} style={nav(i === 0)} onClick={() => setIndex(i - 1)}>← pergunta anterior</button>
        <button type="button" disabled={i === steps.length - 1} style={nav(i === steps.length - 1)}
          onClick={() => setIndex(i + 1)}>próxima pergunta →</button>
      </div>
    </section>
  );
}
```

Em `src/modules/ProtocolEditor.tsx`, importe `import { QuestionPreview } from "./protocolEditor/QuestionPreview";` e, logo depois do bloco `{preview?.outcome && ( ... )}` (antes do `</section>` da segunda coluna), acrescente:

```tsx
        <QuestionPreview definition={current.ok ? current.value : null} />
```

- [ ] **Step 4: Rode e veja passar**

Run: `npx vitest run src/modules/protocolEditor/QuestionPreview.test.tsx src/modules/ProtocolEditor.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/protocolEditor/wpdaLook.ts src/modules/protocolEditor/QuestionPreview.tsx src/modules/protocolEditor/QuestionPreview.test.tsx src/modules/ProtocolEditor.tsx src/modules/ProtocolEditor.test.tsx
/opt/homebrew/bin/git commit -m "feat: preview protocol questions in the citizen web look

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Simulador de perfil

**Files:**
- Create: `src/modules/protocolEditor/OfferSimulator.tsx`
- Modify: `src/modules/ProtocolEditor.tsx`
- Test: `src/modules/protocolEditor/OfferSimulator.test.tsx`, `src/modules/ProtocolEditor.test.tsx` (mock e um caso)

**Interfaces:**
- Consumes: `simulateOffer`, `Sex`, `SimulateOfferResult` (Task 1); `SEX_OPTIONS`, `MAX_AGE` (Task 2); `tiersOf` (Task 3); `parseDefinition` de `src/lib/editor.ts`.
- Produces: `OfferSimulator({ definition: unknown | null; valid: boolean; answers: string })` — `answers` é o texto da caixa "Respostas" do editor. Campos "Idade", "Sexo", "Classificação", "Pontuação", "Prioridade"; botão "Simular"; resultado em `role="status"`.
- O perfil vai com `neighborhood_id: null`: a restrição da cidade não é do protocolo e não entra no simulador do editor (o texto da tela diz isso).

- [ ] **Step 1: Escreva o teste**

```tsx
// src/modules/protocolEditor/OfferSimulator.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, simulateOffer: vi.fn() };
});

import * as api from "../../lib/api";
import { OfferSimulator } from "./OfferSimulator";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

const DEF = {
  name: "saude-do-idoso", version: 1, start_step_id: "s1",
  steps: [ { id: "s1", prompt: "Caiu no último ano?", answer_type: "boolean" } ],
  scoring: { type: "weighted", thresholds: { baixa: 0, alta: 15 } },
  offer: { eligibility: { gte: [ "profile.age", 60 ] } },
  suggestions: [ { protocol: "saude-mental", when: { gte: [ "outcome.score", 15 ] } } ]
};
const OK = { eligible: true, eligibility_text: "idade ≥ 60", suggestions: [ { protocol: "saude-mental", matches: true } ], errors: [] };
const simulate = () => fireEvent.click(screen.getByRole("button", { name: "Simular" }));

describe("OfferSimulator", () => {
  beforeEach(() => mocked(api.simulateOffer).mockReset());

  it("manda o perfil e as respostas, sem resultado quando os campos dele estão vazios", async () => {
    mocked(api.simulateOffer).mockResolvedValue(OK);
    render(<OfferSimulator definition={DEF} valid answers={'{"s1":"true"}'} />);
    fireEvent.change(screen.getByLabelText("Idade"), { target: { value: "62" } });
    simulate();
    await waitFor(() => expect(api.simulateOffer).toHaveBeenCalledWith({
      definition: DEF, profile: { age: 62, sex: "female", neighborhood_id: null }, answers: { s1: "true" }
    }));
    expect(await screen.findByText("Elegível: a triagem aparece no catálogo deste perfil")).not.toBeNull();
    expect(screen.getByText("saude-mental: sugere")).not.toBeNull();
  });

  it("resultado preenchido vai como números, com a classificação do protocolo", async () => {
    mocked(api.simulateOffer).mockResolvedValue(OK);
    render(<OfferSimulator definition={DEF} valid answers="{}" />);
    fireEvent.change(screen.getByLabelText("Sexo"), { target: { value: "male" } });
    fireEvent.change(screen.getByLabelText("Classificação"), { target: { value: "alta" } });
    fireEvent.change(screen.getByLabelText("Pontuação"), { target: { value: "17" } });
    fireEvent.change(screen.getByLabelText("Prioridade"), { target: { value: "3" } });
    simulate();
    await waitFor(() => expect(api.simulateOffer).toHaveBeenCalledWith({
      definition: DEF, profile: { age: 62, sex: "male", neighborhood_id: null }, answers: {},
      outcome: { tier: "alta", score: 17, priority: 3 }
    }));
  });

  it("respostas com JSON inválido não chamam a API", () => {
    render(<OfferSimulator definition={DEF} valid answers="{" />);
    simulate();
    expect(screen.getByText("respostas: JSON inválido")).not.toBeNull();
    expect(api.simulateOffer).not.toHaveBeenCalled();
  });

  it("idade fora de 0 a 130 ou pontuação quebrada trava o botão", () => {
    render(<OfferSimulator definition={DEF} valid answers="{}" />);
    fireEvent.change(screen.getByLabelText("Idade"), { target: { value: "131" } });
    expect((screen.getByRole("button", { name: "Simular" }) as HTMLButtonElement).disabled).toBe(true);
    expect(screen.getByText("idade de 0 a 130; pontuação e prioridade, números inteiros")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Idade"), { target: { value: "40" } });
    fireEvent.change(screen.getByLabelText("Pontuação"), { target: { value: "1.5" } });
    expect((screen.getByRole("button", { name: "Simular" }) as HTMLButtonElement).disabled).toBe(true);
  });

  it("definição com erro trava e diz por quê", () => {
    render(<OfferSimulator definition={DEF} valid={false} answers="{}" />);
    expect((screen.getByRole("button", { name: "Simular" }) as HTMLButtonElement).disabled).toBe(true);
    expect(screen.getByText("Corrija os erros para simular.")).not.toBeNull();
  });

  it("não elegível e erros do gate", async () => {
    mocked(api.simulateOffer).mockResolvedValue({ eligible: false, eligibility_text: null, suggestions: [], errors: [ "suggestions[0].when usa citizen.neighborhood_id" ] });
    render(<OfferSimulator definition={DEF} valid answers="{}" />);
    simulate();
    expect(await screen.findByText("Não elegível: a triagem não aparece para este perfil")).not.toBeNull();
    expect(screen.getByText("suggestions[0].when usa citizen.neighborhood_id")).not.toBeNull();
  });

  it("falha de rede vira mensagem", async () => {
    mocked(api.simulateOffer).mockRejectedValue(new Error("rede"));
    render(<OfferSimulator definition={DEF} valid answers="{}" />);
    simulate();
    expect(await screen.findByText("não foi possível simular — tente de novo")).not.toBeNull();
  });
});
```

Em `src/modules/ProtocolEditor.test.tsx`, acrescente `simulateOffer: vi.fn()` à lista do `vi.mock("../lib/api", ...)` e, no `describe("ProtocolEditor — Oferta e sugestões")`:

```tsx
  it("o simulador usa a definição e as respostas do editor", async () => {
    mocked(api.simulateOffer).mockResolvedValue({ eligible: true, eligibility_text: null, suggestions: [], errors: [] });
    render(<ProtocolEditor />);
    typeDefinition(DEF);
    const button = screen.getByRole("button", { name: "Simular" }) as HTMLButtonElement;
    await waitFor(() => expect(button.disabled).toBe(false));
    fireEvent.click(button);
    await waitFor(() => expect(api.simulateOffer).toHaveBeenCalledWith({
      definition: DEF, profile: { age: 62, sex: "female", neighborhood_id: null }, answers: {}
    }));
  });
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/protocolEditor/OfferSimulator.test.tsx src/modules/ProtocolEditor.test.tsx`
Expected: FAIL — `./OfferSimulator` não existe.

- [ ] **Step 3: Implemente**

```tsx
// src/modules/protocolEditor/OfferSimulator.tsx
// Simulador de perfil (módulo 15; spec §7; contratos §4.3): confere a
// elegibilidade e as sugestões DESTA definição para um perfil, sem gravar
// nada. A restrição da cidade não entra — ela mora no Catálogo de triagens.
import { useState, type CSSProperties } from "react";
import { simulateOffer, type Sex, type SimulateOfferResult, type SimulateOutcome } from "../../lib/api";
import { parseDefinition } from "../../lib/editor";
import { tiersOf } from "../../lib/condition";
import { MAX_AGE, SEX_OPTIONS } from "../../lib/profile";
import { buttonStyle, disabledButtonStyle, inputStyle } from "../../components/formStyles";

const INTEGER = /^-?\d+$/;

export function OfferSimulator({ definition, valid, answers }: { definition: unknown | null; valid: boolean; answers: string }) {
  const [ age, setAge ] = useState("62");
  const [ sex, setSex ] = useState<Sex>("female");
  const [ tier, setTier ] = useState("");
  const [ score, setScore ] = useState("");
  const [ priority, setPriority ] = useState("");
  const [ result, setResult ] = useState<SimulateOfferResult | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);

  const ageOk = /^\d+$/.test(age) && Number(age) <= MAX_AGE;
  const optionalOk = [ score, priority ].every((v) => v.trim() === "" || INTEGER.test(v.trim()));
  const can = valid && definition !== null && ageOk && optionalOk && !busy;

  async function run() {
    if (!can) return;
    const parsed = parseDefinition(answers);
    if (!parsed.ok || !parsed.value || typeof parsed.value !== "object" || Array.isArray(parsed.value)) {
      setError("respostas: JSON inválido");
      return;
    }
    const outcome: SimulateOutcome = {
      ...(tier ? { tier } : {}),
      ...(score.trim() ? { score: Number(score) } : {}),
      ...(priority.trim() ? { priority: Number(priority) } : {})
    };
    setBusy(true); setError(null);
    try {
      setResult(await simulateOffer({
        definition, profile: { age: Number(age), sex, neighborhood_id: null },
        answers: parsed.value as Record<string, string>,
        ...(Object.keys(outcome).length > 0 ? { outcome } : {})
      }));
    } catch {
      setError("não foi possível simular — tente de novo");
    } finally {
      setBusy(false);
    }
  }

  return (
    <section aria-label="Simulador de perfil" style={{ marginTop: 16, display: "flex", flexDirection: "column", gap: 8 }}>
      <h3 style={{ fontSize: 14, margin: 0 }}>Simulador de perfil</h3>
      <p style={hint}>
        Confere a elegibilidade e as sugestões desta definição para um perfil, com as respostas acima. Não grava nada.
        A restrição da cidade não entra aqui: ela fica no Catálogo de triagens.
      </p>
      <div style={{ display: "flex", gap: 8, flexWrap: "wrap" }}>
        <label style={label}>Idade<input type="number" value={age} style={small} onChange={(e) => setAge(e.target.value)} /></label>
        <label style={label}>
          Sexo
          <select value={sex} style={small} onChange={(e) => setSex(e.target.value as Sex)}>
            {SEX_OPTIONS.map((o) => <option key={o.value} value={o.value}>{o.label}</option>)}
          </select>
        </label>
        <label style={label}>
          Classificação
          <select value={tier} style={small} onChange={(e) => setTier(e.target.value)}>
            <option value="">—</option>
            {tiersOf(definition).map((t) => <option key={t} value={t}>{t}</option>)}
          </select>
        </label>
        <label style={label}>Pontuação<input type="number" value={score} style={small} onChange={(e) => setScore(e.target.value)} /></label>
        <label style={label}>Prioridade<input type="number" value={priority} style={small} onChange={(e) => setPriority(e.target.value)} /></label>
      </div>
      {(!ageOk || !optionalOk) && (
        <small role="alert" style={alert}>idade de 0 a {MAX_AGE}; pontuação e prioridade, números inteiros</small>
      )}
      <div>
        <button type="button" disabled={!can} style={can ? buttonStyle : disabledButtonStyle} onClick={() => void run()}>Simular</button>
      </div>
      {!valid && <small style={hint}>Corrija os erros para simular.</small>}
      {error && <p role="alert" style={alert}>{error}</p>}
      {result && <SimulationResult result={result} />}
    </section>
  );
}

function SimulationResult({ result }: { result: SimulateOfferResult }) {
  return (
    <div role="status" style={{ display: "flex", flexDirection: "column", gap: 4, fontSize: 13 }}>
      <strong>
        {result.eligible
          ? "Elegível: a triagem aparece no catálogo deste perfil"
          : "Não elegível: a triagem não aparece para este perfil"}
      </strong>
      {result.eligibility_text && <small style={hint}>Regra conferida pelo servidor: {result.eligibility_text}</small>}
      {result.suggestions.length > 0 && (
        <ul style={{ margin: 0, paddingLeft: 18 }}>
          {result.suggestions.map((s) => <li key={s.protocol}>{s.protocol}: {s.matches ? "sugere" : "não sugere"}</li>)}
        </ul>
      )}
      {result.errors.length > 0 && (
        <ul role="alert" style={{ ...alert, paddingLeft: 18 }}>
          {result.errors.map((e, i) => <li key={i}>{e}</li>)}
        </ul>
      )}
    </div>
  );
}

const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const small: CSSProperties = { ...inputStyle, width: 110 };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12, color: "var(--down)" };
```

Em `src/modules/ProtocolEditor.tsx`, importe `import { OfferSimulator } from "./protocolEditor/OfferSimulator";` e acrescente, logo depois de `<QuestionPreview ... />`:

```tsx
        <OfferSimulator definition={current.ok ? current.value : null} valid={valid} answers={answers} />
```

- [ ] **Step 4: Rode e veja passar**

Run: `npx vitest run src/modules/protocolEditor/OfferSimulator.test.tsx src/modules/ProtocolEditor.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/protocolEditor/OfferSimulator.tsx src/modules/protocolEditor/OfferSimulator.test.tsx src/modules/ProtocolEditor.tsx src/modules/ProtocolEditor.test.tsx
/opt/homebrew/bin/git commit -m "feat: add profile simulator to the protocol editor

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Regras do catálogo — papéis, situação, período, contadores e formulário

**Files:**
- Create: `src/lib/triageCatalog.ts`, `src/test/triageCatalogFixtures.ts`
- Test: `src/lib/triageCatalog.test.ts`

**Interfaces:**
- Consumes: `ApiError`, `ConditionTree`, `TriageOffer`, `TriageOfferFields` (Task 1); `fmtDay` (`audiencePhrase`); `fmtNumber` (`format`); `SUPPRESSED_LABEL` (`smallCount`).
- Produces (em `src/lib/triageCatalog.ts`):
  - `TRIAGE_CATALOG_KEY = ["triageCatalog"]`, `CATALOG_READER_ROLES`, `COUNTER_HINT`;
  - `canReadCatalog(roles: string[]): boolean`, `canEditCatalog(roles: string[]): boolean`;
  - `offerState(o: TriageOffer, today: string): { label: string; tone: "ok" | "neutral" | "warn" | "info" }`;
  - `periodPhrase(from: string | null, until: string | null): string`, `fmtCounter(v: number | null): string`;
  - `OfferFormState { enabled: boolean; position: string; restriction: ConditionTree | null; from: string; until: string }`, `formFrom(o, all): OfferFormState`, `offerFormProblem(f): string | null`, `offerPayload(f): TriageOfferFields`;
  - `triageCatalogError(err: unknown): string | null` (para o `translateError` do `SensitiveAction`).
- Produces (em `src/test/triageCatalogFixtures.ts`): `catalogOffer(overrides?) → TriageOffer` (Saúde do idoso, configurada, posição 2, restrita ao bairro `n2`, até 31/12/2026, `completed: null`) e `RESPIRATORY: TriageOffer` (sem linha no catálogo, sem elegibilidade, contadores zerados).

- [ ] **Step 1: Escreva as fixtures**

```ts
// src/test/triageCatalogFixtures.ts
import type { TriageOffer } from "../lib/api";

export function catalogOffer(overrides: Partial<TriageOffer> = {}): TriageOffer {
  return {
    protocol_name: "saude-do-idoso", title: "Saúde do idoso", active_version: 3,
    eligibility: { gte: [ "profile.age", 60 ] }, retake_after_days: 365,
    configured: true, enabled: true, position: 2,
    restriction: { in: [ "citizen.neighborhood_id", [ "n2" ] ] },
    available_from: null, available_until: "2026-12-31",
    counters: { offered: 120, started: 40, completed: null, from_suggestion: 6 },
    ...overrides
  };
}

export const RESPIRATORY: TriageOffer = catalogOffer({
  protocol_name: "triage-respiratoria", title: "Sintomas respiratórios", active_version: 5,
  eligibility: null, retake_after_days: null, configured: false, enabled: null, position: null,
  restriction: null, available_from: null, available_until: null,
  counters: { offered: 0, started: 0, completed: 0, from_suggestion: 0 }
});
```

- [ ] **Step 2: Escreva o teste**

```ts
// src/lib/triageCatalog.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  canEditCatalog, canReadCatalog, fmtCounter, formFrom, offerFormProblem, offerPayload, offerState, periodPhrase,
  triageCatalogError
} from "./triageCatalog";
import { RESPIRATORY, catalogOffer } from "../test/triageCatalogFixtures";

const TODAY = "2026-10-05";

describe("papéis", () => {
  it("lê quem lê protocolos; edita só o municipal_admin", () => {
    expect(canReadCatalog([ "protocol_author" ])).toBe(true);
    expect(canReadCatalog([ "protocol_reviewer" ])).toBe(true);
    expect(canReadCatalog([ "citizen_verifier", "analyst" ])).toBe(false);
    expect(canEditCatalog([ "protocol_reviewer" ])).toBe(false);
    expect(canEditCatalog([ "municipal_admin" ])).toBe(true);
  });
});

describe("situação no catálogo (ADR 0027, regra de oferta)", () => {
  it.each([
    [ "sem linha e sem elegibilidade: oferecida como hoje", RESPIRATORY, "oferecida (sem configuração)", "ok" ],
    [ "sem linha e com elegibilidade: fora de oferta", catalogOffer({ configured: false, enabled: null, position: null }), "fora de oferta: configure", "warn" ],
    [ "pausada", catalogOffer({ enabled: false }), "pausada", "neutral" ],
    [ "antes do início", catalogOffer({ available_from: "2026-10-06" }), "agendada", "info" ],
    [ "depois do fim", catalogOffer({ available_until: "2026-10-04" }), "período encerrado", "warn" ],
    [ "no último dia ainda vale", catalogOffer({ available_until: TODAY }), "oferecida", "ok" ]
  ])("%s", (_name, offer, label, tone) => {
    expect(offerState(offer, TODAY)).toEqual({ label, tone });
  });
});

describe("período e contadores", () => {
  it("período sem deslocar o dia", () => {
    expect(periodPhrase(null, null)).toBe("sem período");
    expect(periodPhrase("2026-07-01", null)).toBe("a partir de 01/07/2026");
    expect(periodPhrase(null, "2026-12-31")).toBe("até 31/12/2026");
    expect(periodPhrase("2026-07-01", "2026-12-31")).toBe("de 01/07/2026 a 31/12/2026");
  });

  it("contador nulo é < 5, zero é 0", () => {
    expect(fmtCounter(null)).toBe("< 5");
    expect(fmtCounter(0)).toBe("0");
    expect(fmtCounter(1200)).toBe("1.200");
  });
});

describe("formulário", () => {
  it("linha configurada vem como está; datas nulas viram campo vazio", () => {
    expect(formFrom(catalogOffer(), [])).toEqual({
      enabled: true, position: "2", restriction: { in: [ "citizen.neighborhood_id", [ "n2" ] ] }, from: "", until: "2026-12-31"
    });
  });

  it("protocolo sem linha começa oferecido, na próxima posição livre", () => {
    expect(formFrom(RESPIRATORY, [ catalogOffer(), RESPIRATORY ])).toEqual({
      enabled: true, position: "3", restriction: null, from: "", until: ""
    });
    expect(formFrom(RESPIRATORY, [ RESPIRATORY ]).position).toBe("1");
  });

  it("ordem precisa ser inteiro a partir de 1; fim não pode vir antes do início", () => {
    const base = formFrom(catalogOffer(), []);
    expect(offerFormProblem(base)).toBeNull();
    expect(offerFormProblem({ ...base, position: "0" })).toBe("a ordem precisa ser um número inteiro a partir de 1");
    expect(offerFormProblem({ ...base, position: "2.5" })).toBe("a ordem precisa ser um número inteiro a partir de 1");
    expect(offerFormProblem({ ...base, from: "2026-12-01", until: "2026-11-01" })).toBe("o fim do período não pode ser antes do início");
    expect(offerFormProblem({ ...base, from: "2026-12-01", until: "2026-12-01" })).toBeNull();
  });

  it("payload: número na ordem e data vazia como null", () => {
    expect(offerPayload({ enabled: false, position: " 4 ", restriction: null, from: "", until: "2026-12-31" }))
      .toEqual({ enabled: false, position: 4, restriction: null, available_from: null, available_until: "2026-12-31" });
  });
});

describe("recusas da API", () => {
  it("traduz os códigos do PUT e deixa o resto para a mensagem padrão", () => {
    expect(triageCatalogError(new ApiError(422, { error: "invalid_restriction" }, "x")))
      .toBe("a restrição usa um campo que o catálogo não aceita ou está malformada");
    expect(triageCatalogError(new ApiError(422, { error: "invalid_period" }, "x"))).toBe("o fim do período não pode ser antes do início");
    expect(triageCatalogError(new ApiError(422, { error: "invalid_position" }, "x"))).toBe("a ordem precisa ser um número inteiro a partir de 1");
    expect(triageCatalogError(new ApiError(404, { error: "unknown_protocol" }, "x")))
      .toBe("este protocolo não existe mais nesta cidade — recarregue a lista");
    expect(triageCatalogError(new ApiError(422, { error: "outra_coisa" }, "x"))).toBeNull();
    expect(triageCatalogError(new ApiError(500, "erro", "x"))).toBeNull();
    expect(triageCatalogError(new Error("rede"))).toBeNull();
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/lib/triageCatalog.test.ts`
Expected: FAIL — `./triageCatalog` não existe.

- [ ] **Step 4: Implemente**

```ts
// src/lib/triageCatalog.ts
// Regras da aba "Catálogo de triagens" (módulo 15; ADR 0027; contratos §4.1 e
// §4.2). A situação repete a regra de oferta só para mostrar; quem decide o
// catálogo de cada cidadão é o Triages::Offer no api.
import { ApiError, type ConditionTree, type TriageOffer, type TriageOfferFields } from "./api";
import { fmtDay } from "./audiencePhrase";
import { fmtNumber } from "./format";
import { SUPPRESSED_LABEL } from "./smallCount";

export const TRIAGE_CATALOG_KEY = [ "triageCatalog" ] as const;
export const CATALOG_READER_ROLES = [ "protocol_author", "protocol_reviewer", "municipal_admin" ];
export const COUNTER_HINT = "Últimos 30 dias. Contagens de 1 a 4 aparecem como “< 5”, para não identificar ninguém.";

export function canReadCatalog(roles: string[]): boolean {
  return roles.some((r) => CATALOG_READER_ROLES.includes(r));
}

export function canEditCatalog(roles: string[]): boolean {
  return roles.includes("municipal_admin");
}

export type OfferTone = "ok" | "neutral" | "warn" | "info";

export function offerState(o: TriageOffer, today: string): { label: string; tone: OfferTone } {
  if (!o.configured) {
    return o.eligibility === null
      ? { label: "oferecida (sem configuração)", tone: "ok" }
      : { label: "fora de oferta: configure", tone: "warn" };
  }
  if (!o.enabled) return { label: "pausada", tone: "neutral" };
  if (o.available_from && today < o.available_from) return { label: "agendada", tone: "info" };
  if (o.available_until && today > o.available_until) return { label: "período encerrado", tone: "warn" };
  return { label: "oferecida", tone: "ok" };
}

export function periodPhrase(from: string | null, until: string | null): string {
  if (from && until) return `de ${fmtDay(from)} a ${fmtDay(until)}`;
  if (from) return `a partir de ${fmtDay(from)}`;
  if (until) return `até ${fmtDay(until)}`;
  return "sem período";
}

// null = entre 1 e 4 (ADR 0025). Nunca 0, nunca "—".
export function fmtCounter(v: number | null): string {
  return v === null ? SUPPRESSED_LABEL : fmtNumber(v);
}

export interface OfferFormState {
  enabled: boolean;
  position: string;
  restriction: ConditionTree | null;
  from: string;
  until: string;
}

export function formFrom(o: TriageOffer, all: TriageOffer[]): OfferFormState {
  if (o.configured) {
    return {
      enabled: o.enabled ?? true, position: String(o.position ?? 1), restriction: o.restriction,
      from: o.available_from ?? "", until: o.available_until ?? ""
    };
  }
  const last = Math.max(0, ...all.map((x) => x.position ?? 0));
  return { enabled: true, position: String(last + 1), restriction: null, from: "", until: "" };
}

export function offerFormProblem(f: OfferFormState): string | null {
  if (!/^\d+$/.test(f.position.trim()) || Number(f.position) < 1) return "a ordem precisa ser um número inteiro a partir de 1";
  if (f.from && f.until && f.until < f.from) return "o fim do período não pode ser antes do início";
  return null;
}

export function offerPayload(f: OfferFormState): TriageOfferFields {
  return {
    enabled: f.enabled, position: Number(f.position.trim()), restriction: f.restriction,
    available_from: f.from || null, available_until: f.until || null
  };
}

const MESSAGES: Record<string, string> = {
  invalid_restriction: "a restrição usa um campo que o catálogo não aceita ou está malformada",
  invalid_period: "o fim do período não pode ser antes do início",
  invalid_position: "a ordem precisa ser um número inteiro a partir de 1",
  unknown_protocol: "este protocolo não existe mais nesta cidade — recarregue a lista"
};

// Para o translateError do SensitiveAction: as recusas do PUT não trazem
// `message`, e sem isto a tela diria só "a API recusou a ação".
export function triageCatalogError(err: unknown): string | null {
  if (!(err instanceof ApiError) || !err.body || typeof err.body !== "object") return null;
  const code = (err.body as { error?: unknown }).error;
  return typeof code === "string" ? MESSAGES[code] ?? null : null;
}
```

- [ ] **Step 5: Rode e veja passar**

Run: `npx vitest run src/lib/triageCatalog.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/lib/triageCatalog.ts src/lib/triageCatalog.test.ts src/test/triageCatalogFixtures.ts
/opt/homebrew/bin/git commit -m "feat: add triage catalog rules for state, period, counters and form

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Aba "Catálogo de triagens" no módulo Protocolos (leitura)

**Files:**
- Create: `src/modules/protocols/TriageCatalogTab.tsx`
- Modify: `src/modules/Protocols.tsx`
- Test: `src/modules/protocols/TriageCatalogTab.test.tsx`, `src/modules/Protocols.test.tsx`

**Interfaces:**
- Consumes: `listTriageCatalog`, `listPanelNeighborhoods`, `TriageOffer` (Task 1 e existente); `fieldsFor` (Task 3); `describeCondition` (Task 4); tudo de `src/lib/triageCatalog.ts` (Task 10); `todayInCity` (`src/lib/campaigns.ts`); `PANEL_NEIGHBORHOODS_KEY` (`src/lib/neighborhoodFilter.ts`); `SegmentedControl`, `Panel`, `DataTable`, `Tag`, `Skeleton`, `ErrorState`; `sessionWith`/`renderWithProviders` de `src/test/campaignFixtures.tsx`; `NEIGHBORHOODS` (Task 3) e `catalogOffer`/`RESPIRATORY` (Task 10).
- Produces:
  - `TriageCatalogTab({ onNavigate?: (id: ModuleId) => void })` — `Panel` "Catálogo de triagens" com uma linha por protocolo ativo: Ordem, Protocolo (título, nome e versão), Elegibilidade (frase, "para todos"), Restrição da cidade (frase, "nenhuma"), Período, Situação (`Tag`), Contadores "oferecida · iniciada · concluída · de sugestão". `onNavigate` é usado na Task 12;
  - `Protocols` passa a ter as abas "Versões" e "Catálogo de triagens" (`SegmentedControl`), esta só para `canReadCatalog`. O conteúdo de hoje vira `ProtocolVersions`, sem mudança de comportamento.

- [ ] **Step 1: Escreva o teste da aba**

```tsx
// src/modules/protocols/TriageCatalogTab.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, screen } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return {
    ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(),
    listTriageCatalog: vi.fn(), listPanelNeighborhoods: vi.fn(), updateTriageOffer: vi.fn()
  };
});

import * as api from "../../lib/api";
import { TriageCatalogTab } from "./TriageCatalogTab";
import { renderWithProviders, sessionWith } from "../../test/campaignFixtures";
import { NEIGHBORHOODS } from "../../test/conditionFixtures";
import { RESPIRATORY, catalogOffer } from "../../test/triageCatalogFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-05T12:00:00-03:00"));
  for (const fn of [ api.fetchCurrentSession, api.stepUpMfa, api.listTriageCatalog, api.listPanelNeighborhoods, api.updateTriageOffer ]) {
    mocked(fn).mockReset();
  }
  mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "protocol_reviewer" ]));
  mocked(api.listPanelNeighborhoods).mockResolvedValue(NEIGHBORHOODS);
});
afterEach(() => { cleanup(); vi.useRealTimers(); });

describe("TriageCatalogTab — leitura", () => {
  it("uma linha por protocolo, com as frases, o período e a situação", async () => {
    mocked(api.listTriageCatalog).mockResolvedValue([ catalogOffer(), RESPIRATORY ]);
    renderWithProviders(<TriageCatalogTab />);
    expect(await screen.findByText("Saúde do idoso")).not.toBeNull();
    expect(screen.getByText("idade a partir de 60 anos")).not.toBeNull();
    expect(await screen.findByText("bairro Boqueirão")).not.toBeNull();
    expect(screen.getByText("até 31/12/2026")).not.toBeNull();
    expect(screen.getByText("oferecida")).not.toBeNull();
    expect(screen.getByText("Sintomas respiratórios")).not.toBeNull();
    expect(screen.getByText("para todos")).not.toBeNull();
    expect(screen.getByText("nenhuma")).not.toBeNull();
    expect(screen.getByText("sem período")).not.toBeNull();
    expect(screen.getByText("oferecida (sem configuração)")).not.toBeNull();
  });

  it("contadores com supressão", async () => {
    mocked(api.listTriageCatalog).mockResolvedValue([ catalogOffer(), RESPIRATORY ]);
    renderWithProviders(<TriageCatalogTab />);
    expect(await screen.findByText("120 · 40 · < 5 · 6")).not.toBeNull();
    expect(screen.getByText("0 · 0 · 0 · 0")).not.toBeNull();
    expect(screen.getByText("Últimos 30 dias. Contagens de 1 a 4 aparecem como “< 5”, para não identificar ninguém.")).not.toBeNull();
  });

  it("protocolo com elegibilidade e sem linha no catálogo aparece fora de oferta", async () => {
    mocked(api.listTriageCatalog).mockResolvedValue([ catalogOffer({ configured: false, enabled: null, position: null, restriction: null }) ]);
    renderWithProviders(<TriageCatalogTab />);
    expect(await screen.findByText("fora de oferta: configure")).not.toBeNull();
  });

  it("cidade sem protocolo em uso", async () => {
    mocked(api.listTriageCatalog).mockResolvedValue([]);
    renderWithProviders(<TriageCatalogTab />);
    expect(await screen.findByText("nenhum protocolo em uso nesta cidade")).not.toBeNull();
  });

  it("falha ao carregar mostra o erro com nova tentativa", async () => {
    mocked(api.listTriageCatalog).mockRejectedValue(new Error("rede"));
    renderWithProviders(<TriageCatalogTab />);
    expect(await screen.findByText("não foi possível carregar o catálogo")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Escreva os testes das abas em `src/modules/Protocols.test.tsx`**

Acrescente `listTriageCatalog: vi.fn(), listPanelNeighborhoods: vi.fn()` à lista do `vi.mock("../lib/api", ...)`, o import `import { catalogOffer } from "../test/triageCatalogFixtures";` e, no fim do arquivo:

```tsx
describe("Protocols — abas", () => {
  beforeEach(() => {
    mocked(api.listTriageCatalog).mockReset();
    mocked(api.listPanelNeighborhoods).mockReset();
    mocked(api.listTriageCatalog).mockResolvedValue([ catalogOffer() ]);
    mocked(api.listPanelNeighborhoods).mockResolvedValue([]);
  });

  it("quem lê protocolos vê a aba Catálogo de triagens e a lista do catálogo", async () => {
    stubReads([ row() ]);
    renderProtocols("protocol_reviewer");
    fireEvent.click(await screen.findByRole("tab", { name: "Catálogo de triagens" }));
    expect(await screen.findByText("Saúde do idoso")).not.toBeNull();
    fireEvent.click(screen.getByRole("tab", { name: "Versões" }));
    expect(await screen.findByRole("region", { name: "Protocolos & versões" })).not.toBeNull();
  });

  it("papel sem leitura de protocolo não vê as abas e nunca chama o catálogo", async () => {
    stubReads([ row() ]);
    renderProtocols("citizen_verifier");
    expect(await screen.findByRole("region", { name: "Protocolos & versões" })).not.toBeNull();
    expect(screen.queryByRole("tab", { name: "Catálogo de triagens" })).toBeNull();
    expect(api.listTriageCatalog).not.toHaveBeenCalled();
  });
});
```

(`renderProtocols`, `stubReads` e `row` são os helpers que o arquivo já tem.)

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/modules/protocols/TriageCatalogTab.test.tsx src/modules/Protocols.test.tsx`
Expected: FAIL — `./TriageCatalogTab` não existe e não há aba.

- [ ] **Step 4: Implemente a aba**

```tsx
// src/modules/protocols/TriageCatalogTab.tsx
// Aba "Catálogo de triagens" (módulo 15; spec §7; ADR 0027). Uma linha por
// protocolo com versão em uso. Leitura para os papéis de protocolo.
import { useQuery } from "@tanstack/react-query";
import { listPanelNeighborhoods, listTriageCatalog, type TriageOffer } from "../../lib/api";
import { useAuth } from "../../lib/auth";
import { fieldsFor } from "../../lib/condition";
import { describeCondition } from "../../lib/conditionPhrase";
import { PANEL_NEIGHBORHOODS_KEY } from "../../lib/neighborhoodFilter";
import { todayInCity } from "../../lib/campaigns";
import { COUNTER_HINT, TRIAGE_CATALOG_KEY, canEditCatalog, fmtCounter, offerState, periodPhrase } from "../../lib/triageCatalog";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { Skeleton } from "../../components/Skeleton";
import { ErrorState } from "../../components/ErrorState";
import type { ModuleId } from "../../shell/modules";

export function TriageCatalogTab(_props: { onNavigate?: (id: ModuleId) => void } = {}) {
  const auth = useAuth();
  const canEdit = canEditCatalog((auth.user?.memberships ?? []).map((m) => m.role));
  const catalog = useQuery({ queryKey: TRIAGE_CATALOG_KEY, queryFn: listTriageCatalog });
  const neighborhoods = useQuery({ queryKey: PANEL_NEIGHBORHOODS_KEY, queryFn: listPanelNeighborhoods });

  if (catalog.isLoading) return <Panel title="Catálogo de triagens"><Skeleton rows={4} /></Panel>;
  if (catalog.isError) return <ErrorState message="não foi possível carregar o catálogo" onRetry={() => void catalog.refetch()} />;

  const offers = catalog.data ?? [];
  const profileFields = fieldsFor("eligibility");
  const restrictionFields = fieldsFor("restriction", { neighborhoods: neighborhoods.data ?? [] });
  const today = todayInCity();

  return (
    <Panel title="Catálogo de triagens" sub={canEdit ? "clique numa linha para editar" : "somente leitura"}>
      <DataTable<TriageOffer>
        cols={[
          { label: "Ordem", w: "0.5fr", render: (o) => <span className="mono">{o.position ?? "—"}</span> },
          {
            label: "Protocolo", w: "1.6fr", render: (o) => (
              <span>
                <strong>{o.title}</strong>{" "}
                <span className="mono" style={{ color: "var(--ink3)" }}>{o.protocol_name} v{o.active_version}</span>
              </span>
            )
          },
          { label: "Elegibilidade", w: "1.6fr", render: (o) => describeCondition(o.eligibility, profileFields, "para todos") },
          { label: "Restrição da cidade", w: "1.6fr", render: (o) => describeCondition(o.restriction, restrictionFields, "nenhuma") },
          { label: "Período", w: "1fr", render: (o) => periodPhrase(o.available_from, o.available_until) },
          {
            label: "Situação", w: "1fr", render: (o) => {
              const state = offerState(o, today);
              return <Tag tone={state.tone}>{state.label}</Tag>;
            }
          },
          {
            label: "Oferecida · iniciada · concluída · de sugestão", w: "1.4fr", render: (o) => (
              <span className="mono" title={COUNTER_HINT}>
                {[ o.counters.offered, o.counters.started, o.counters.completed, o.counters.from_suggestion ].map(fmtCounter).join(" · ")}
              </span>
            )
          }
        ]}
        rows={offers}
        rowKey={(o) => o.protocol_name}
        empty="nenhum protocolo em uso nesta cidade"
      />
      <p style={{ margin: "8px 0 0", fontSize: 11, color: "var(--ink3)" }}>{COUNTER_HINT}</p>
    </Panel>
  );
}
```

- [ ] **Step 5: Abas em `src/modules/Protocols.tsx`**

1. Imports, junto dos outros:

```tsx
import { SegmentedControl } from "../shell/SegmentedControl";
import { canReadCatalog } from "../lib/triageCatalog";
import { TriageCatalogTab } from "./protocols/TriageCatalogTab";
```

2. Troque a assinatura

```tsx
export function Protocols({ onNavigate }: { onNavigate?: (id: ModuleId) => void } = {}) {
  const [ openId, setOpenId ] = useState<string | null>(null);
```

por

```tsx
type ProtocolsTab = "versions" | "catalog";
const TABS: { key: ProtocolsTab; label: string }[] = [
  { key: "versions", label: "Versões" },
  { key: "catalog", label: "Catálogo de triagens" }
];

// Módulo 15: a aba do catálogo só aparece para quem a API deixa ler
// (GET /triage_catalog); para os outros papéis, a tela é a de sempre.
export function Protocols({ onNavigate }: { onNavigate?: (id: ModuleId) => void } = {}) {
  const auth = useAuth();
  const showCatalog = canReadCatalog((auth.user?.memberships ?? []).map((m) => m.role));
  const [ tab, setTab ] = useState<ProtocolsTab>("versions");
  return (
    <Wrap>
      {showCatalog && <SegmentedControl options={TABS} value={tab} onChange={setTab} />}
      {showCatalog && tab === "catalog"
        ? <TriageCatalogTab onNavigate={onNavigate} />
        : <ProtocolVersions onNavigate={onNavigate} />}
    </Wrap>
  );
}

function ProtocolVersions({ onNavigate }: { onNavigate?: (id: ModuleId) => void }) {
  const [ openId, setOpenId ] = useState<string | null>(null);
```

3. Dentro de `ProtocolVersions` (só nela — é o corpo antigo de `Protocols`, até o `}` que fecha antes de `function DetailDrawer`), troque todo `<Wrap>` por `<Stack>` e todo `</Wrap>` por `</Stack>`. São as três saídas antecipadas (carregando, erro, sem dados) e o `return` principal.

4. Depois da função `Wrap` no fim do arquivo, acrescente:

```tsx
// O conteúdo de cada aba, sem o cabeçalho: o PageHeader fica uma vez só, no Wrap.
function Stack({ children }: { children: React.ReactNode }) {
  return <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>{children}</div>;
}
```

- [ ] **Step 6: Rode e veja passar**

Run: `npx vitest run src/modules/protocols/TriageCatalogTab.test.tsx src/modules/Protocols.test.tsx && npx tsc --noEmit`
Expected: PASS, inclusive os testes antigos de `Protocols.test.tsx` (a aba padrão é "Versões").

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/modules/protocols/TriageCatalogTab.tsx src/modules/protocols/TriageCatalogTab.test.tsx src/modules/Protocols.tsx src/modules/Protocols.test.tsx
/opt/homebrew/bin/git commit -m "feat: add triage catalog tab to the protocols module

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Edição do catálogo com step-up

**Files:**
- Create: `src/modules/protocols/TriageOfferForm.tsx`
- Modify: `src/modules/protocols/TriageCatalogTab.tsx` (arquivo inteiro)
- Test: `src/modules/protocols/TriageCatalogTab.test.tsx` (novo `describe`)

**Interfaces:**
- Consumes: `updateTriageOffer`, `listPanelNeighborhoods`, `TriageOffer` (Task 1); `ConditionBuilder` (Task 5); `fieldsFor` (Task 3); `describeCondition` (Task 4); `formFrom`, `offerFormProblem`, `offerPayload`, `triageCatalogError`, `TRIAGE_CATALOG_KEY` (Task 10); `SensitiveAction` (existente).
- Produces: `TriageOfferForm({ offer: TriageOffer; all: TriageOffer[]; onSaved(): void; onCancel(): void; onGoToSecurity?(): void })` — `section` "Editar {título}" com "Oferecer no catálogo", "Ordem no catálogo", construtor "Restrição da cidade", "Disponível a partir de", "Disponível até", botão "Salvar no catálogo…" que abre o `SensitiveAction` (`requiresStepUp`, título "Salvar {título} no catálogo"). A aba abre o formulário ao clicar numa linha, só para `municipal_admin`, mostra "Catálogo atualizado: {título}" e relê a lista.

- [ ] **Step 1: Escreva os testes (fim de `src/modules/protocols/TriageCatalogTab.test.tsx`)**

Acrescente `fireEvent`, `waitFor` e `within` ao import de `@testing-library/react` e `ApiError` ao de `../../lib/api` (`import { ApiError } from "../../lib/api";`).

```tsx
describe("TriageCatalogTab — edição", () => {
  const admin = () => mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "municipal_admin" ]));
  const open = async (title: string) => {
    fireEvent.click(await screen.findByText(title));
    return screen.getByRole("region", { name: `Editar ${title}` });
  };

  it("admin pausa e salva com a janela de step-up aberta; a lista é relida", async () => {
    admin();
    mocked(api.listTriageCatalog).mockResolvedValue([ catalogOffer(), RESPIRATORY ]);
    mocked(api.updateTriageOffer).mockResolvedValue(catalogOffer({ enabled: false }));
    renderWithProviders(<TriageCatalogTab />);
    const form = await open("Saúde do idoso");
    fireEvent.click(within(form).getByLabelText("Oferecer no catálogo"));
    fireEvent.click(within(form).getByRole("button", { name: "Salvar no catálogo…" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.updateTriageOffer).toHaveBeenCalledWith("saude-do-idoso", {
      enabled: false, position: 2, restriction: { in: [ "citizen.neighborhood_id", [ "n2" ] ] },
      available_from: null, available_until: "2026-12-31"
    }));
    expect(await screen.findByText("Catálogo atualizado: Saúde do idoso")).not.toBeNull();
    await waitFor(() => expect(api.listTriageCatalog).toHaveBeenCalledTimes(2));
  });

  it("protocolo sem linha começa na próxima posição e ganha restrição por bairro", async () => {
    admin();
    mocked(api.listTriageCatalog).mockResolvedValue([ catalogOffer(), RESPIRATORY ]);
    mocked(api.updateTriageOffer).mockResolvedValue(RESPIRATORY);
    renderWithProviders(<TriageCatalogTab />);
    const form = await open("Sintomas respiratórios");
    expect((within(form).getByLabelText("Ordem no catálogo") as HTMLInputElement).value).toBe("3");
    const restriction = within(form).getByRole("group", { name: "Restrição da cidade" });
    fireEvent.click(within(restriction).getByRole("button", { name: "+ condição" }));
    fireEvent.change(within(restriction).getByLabelText("campo"), { target: { value: "citizen.neighborhood_id" } });
    fireEvent.click(await within(restriction).findByLabelText("Xaxim"));
    expect(within(restriction).queryByLabelText("Centro (bairro inativo)")).toBeNull();
    fireEvent.click(within(form).getByRole("button", { name: "Salvar no catálogo…" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.updateTriageOffer).toHaveBeenCalledWith("triage-respiratoria", {
      enabled: true, position: 3, restriction: { in: [ "citizen.neighborhood_id", [ "n1" ] ] },
      available_from: null, available_until: null
    }));
  });

  it("período invertido trava o salvar", async () => {
    admin();
    mocked(api.listTriageCatalog).mockResolvedValue([ catalogOffer() ]);
    renderWithProviders(<TriageCatalogTab />);
    const form = await open("Saúde do idoso");
    fireEvent.change(within(form).getByLabelText("Disponível a partir de"), { target: { value: "2027-01-10" } });
    expect(within(form).getByText("o fim do período não pode ser antes do início")).not.toBeNull();
    expect((within(form).getByRole("button", { name: "Salvar no catálogo…" }) as HTMLButtonElement).disabled).toBe(true);
  });

  it("janela de step-up fechada pede o código antes de gravar", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "municipal_admin" ], { mfa_verified_at: null }));
    mocked(api.stepUpMfa).mockResolvedValue(undefined);
    mocked(api.listTriageCatalog).mockResolvedValue([ catalogOffer() ]);
    mocked(api.updateTriageOffer).mockResolvedValue(catalogOffer());
    renderWithProviders(<TriageCatalogTab />);
    const form = await open("Saúde do idoso");
    fireEvent.click(within(form).getByRole("button", { name: "Salvar no catálogo…" }));
    fireEvent.change(await screen.findByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.stepUpMfa).toHaveBeenCalledWith("123456"));
    await waitFor(() => expect(api.updateTriageOffer).toHaveBeenCalledTimes(1));
  });

  it("recusa do servidor é traduzida e o diálogo continua aberto", async () => {
    admin();
    mocked(api.listTriageCatalog).mockResolvedValue([ catalogOffer() ]);
    mocked(api.updateTriageOffer).mockRejectedValue(new ApiError(422, { error: "invalid_restriction" }, "x"));
    renderWithProviders(<TriageCatalogTab />);
    const form = await open("Saúde do idoso");
    fireEvent.click(within(form).getByRole("button", { name: "Salvar no catálogo…" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    expect(await screen.findByText("a restrição usa um campo que o catálogo não aceita ou está malformada")).not.toBeNull();
    expect(screen.getByRole("button", { name: "Confirmar" })).not.toBeNull();
  });

  it("revisor só lê: clicar na linha não abre edição", async () => {
    mocked(api.listTriageCatalog).mockResolvedValue([ catalogOffer() ]);
    renderWithProviders(<TriageCatalogTab />);
    fireEvent.click(await screen.findByText("Saúde do idoso"));
    expect(screen.queryByRole("button", { name: "Salvar no catálogo…" })).toBeNull();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/protocols/TriageCatalogTab.test.tsx`
Expected: FAIL — nenhuma linha abre o formulário.

- [ ] **Step 3: Implemente o formulário**

```tsx
// src/modules/protocols/TriageOfferForm.tsx
// Edição de uma linha do catálogo da cidade (módulo 15; ADR 0027; contratos
// §4.2): oferecer ou pausar, ordem, restrição e período. Só municipal_admin,
// sempre com step-up (SensitiveAction). A restrição soma com E à elegibilidade
// assinada: a cidade restringe, nunca amplia.
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { listPanelNeighborhoods, updateTriageOffer, type TriageOffer } from "../../lib/api";
import { fieldsFor } from "../../lib/condition";
import { describeCondition } from "../../lib/conditionPhrase";
import { PANEL_NEIGHBORHOODS_KEY } from "../../lib/neighborhoodFilter";
import { formFrom, offerFormProblem, offerPayload, triageCatalogError, type OfferFormState } from "../../lib/triageCatalog";
import { ConditionBuilder } from "./ConditionBuilder";
import { SensitiveAction } from "../../components/SensitiveAction";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";

export function TriageOfferForm({ offer, all, onSaved, onCancel, onGoToSecurity }: {
  offer: TriageOffer; all: TriageOffer[]; onSaved(): void; onCancel(): void; onGoToSecurity?(): void;
}) {
  const [ form, setForm ] = useState<OfferFormState>(() => formFrom(offer, all));
  const [ confirming, setConfirming ] = useState(false);
  const neighborhoods = useQuery({ queryKey: PANEL_NEIGHBORHOODS_KEY, queryFn: listPanelNeighborhoods });
  const restrictionFields = fieldsFor("restriction", { neighborhoods: neighborhoods.data ?? [] });
  const problem = offerFormProblem(form);
  // Mexer no formulário depois de abrir a confirmação fecha a confirmação:
  // o que se confirma é sempre o que está na tela.
  const set = (patch: Partial<OfferFormState>) => { setConfirming(false); setForm((f) => ({ ...f, ...patch })); };

  return (
    <section aria-label={`Editar ${offer.title}`} style={panel}>
      <strong>
        {offer.title} <span className="mono" style={{ fontSize: 11, color: "var(--ink3)" }}>{offer.protocol_name}</span>
      </strong>
      <p style={hint}>
        Elegibilidade assinada: {describeCondition(offer.eligibility, fieldsFor("eligibility"), "para todos")}.
        A restrição abaixo soma com E: só restringe, nunca amplia.
      </p>
      <label style={{ ...label, flexDirection: "row", alignItems: "center", gap: 8 }}>
        <input type="checkbox" checked={form.enabled} onChange={(e) => set({ enabled: e.target.checked })} />
        Oferecer no catálogo
      </label>
      <label style={label}>
        Ordem no catálogo
        <input inputMode="numeric" value={form.position} style={{ ...inputStyle, width: 120 }}
          onChange={(e) => set({ position: e.target.value })} />
      </label>
      <ConditionBuilder label="Restrição da cidade" fields={restrictionFields} value={form.restriction}
        emptyText="nenhuma" onChange={(tree) => set({ restriction: tree })} />
      <div style={{ display: "flex", gap: 8, flexWrap: "wrap" }}>
        <label style={label}>
          Disponível a partir de
          <input type="date" value={form.from} style={inputStyle} onChange={(e) => set({ from: e.target.value })} />
        </label>
        <label style={label}>
          Disponível até
          <input type="date" value={form.until} style={inputStyle} onChange={(e) => set({ until: e.target.value })} />
        </label>
      </div>
      {problem && <p role="alert" style={alert}>{problem}</p>}
      {!confirming && (
        <div style={{ display: "flex", gap: 8 }}>
          <button type="button" disabled={problem !== null} style={problem ? disabledButtonStyle : buttonStyle}
            onClick={() => setConfirming(true)}>Salvar no catálogo…</button>
          <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
        </div>
      )}
      {confirming && (
        <SensitiveAction
          title={`Salvar ${offer.title} no catálogo`}
          description="Vale para os próximos catálogos montados no wpda e fica registrado na auditoria."
          requiresStepUp
          run={async () => { await updateTriageOffer(offer.protocol_name, offerPayload(form)); }}
          translateError={triageCatalogError}
          onDone={onSaved}
          onCancel={() => setConfirming(false)}
          onGoToSecurity={onGoToSecurity}
        />
      )}
    </section>
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 16, marginTop: 12, border: "1px solid var(--rule)", borderRadius: 8 };
const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12, color: "var(--down)" };
```

- [ ] **Step 4: Substitua `src/modules/protocols/TriageCatalogTab.tsx` inteiro**

```tsx
// src/modules/protocols/TriageCatalogTab.tsx
// Aba "Catálogo de triagens" (módulo 15; spec §7; ADR 0027). Uma linha por
// protocolo com versão em uso. Leitura para os papéis de protocolo; o
// municipal_admin clica na linha para editar, com step-up.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { listPanelNeighborhoods, listTriageCatalog, type TriageOffer } from "../../lib/api";
import { useAuth } from "../../lib/auth";
import { fieldsFor } from "../../lib/condition";
import { describeCondition } from "../../lib/conditionPhrase";
import { PANEL_NEIGHBORHOODS_KEY } from "../../lib/neighborhoodFilter";
import { todayInCity } from "../../lib/campaigns";
import { COUNTER_HINT, TRIAGE_CATALOG_KEY, canEditCatalog, fmtCounter, offerState, periodPhrase } from "../../lib/triageCatalog";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { Skeleton } from "../../components/Skeleton";
import { ErrorState } from "../../components/ErrorState";
import type { ModuleId } from "../../shell/modules";
import { TriageOfferForm } from "./TriageOfferForm";

export function TriageCatalogTab({ onNavigate }: { onNavigate?: (id: ModuleId) => void } = {}) {
  const auth = useAuth();
  const queryClient = useQueryClient();
  const canEdit = canEditCatalog((auth.user?.memberships ?? []).map((m) => m.role));
  const catalog = useQuery({ queryKey: TRIAGE_CATALOG_KEY, queryFn: listTriageCatalog });
  const neighborhoods = useQuery({ queryKey: PANEL_NEIGHBORHOODS_KEY, queryFn: listPanelNeighborhoods });
  const [ editing, setEditing ] = useState<TriageOffer | null>(null);
  const [ done, setDone ] = useState<string | null>(null);

  if (catalog.isLoading) return <Panel title="Catálogo de triagens"><Skeleton rows={4} /></Panel>;
  if (catalog.isError) return <ErrorState message="não foi possível carregar o catálogo" onRetry={() => void catalog.refetch()} />;

  const offers = catalog.data ?? [];
  const profileFields = fieldsFor("eligibility");
  const restrictionFields = fieldsFor("restriction", { neighborhoods: neighborhoods.data ?? [] });
  const today = todayInCity();

  return (
    <Panel title="Catálogo de triagens" sub={canEdit ? "clique numa linha para editar" : "somente leitura"}>
      {done && <p role="status" style={{ margin: "0 0 8px", fontSize: 12.5 }}>{done}</p>}
      <DataTable<TriageOffer>
        cols={[
          { label: "Ordem", w: "0.5fr", render: (o) => <span className="mono">{o.position ?? "—"}</span> },
          {
            label: "Protocolo", w: "1.6fr", render: (o) => (
              <span>
                <strong>{o.title}</strong>{" "}
                <span className="mono" style={{ color: "var(--ink3)" }}>{o.protocol_name} v{o.active_version}</span>
              </span>
            )
          },
          { label: "Elegibilidade", w: "1.6fr", render: (o) => describeCondition(o.eligibility, profileFields, "para todos") },
          { label: "Restrição da cidade", w: "1.6fr", render: (o) => describeCondition(o.restriction, restrictionFields, "nenhuma") },
          { label: "Período", w: "1fr", render: (o) => periodPhrase(o.available_from, o.available_until) },
          {
            label: "Situação", w: "1fr", render: (o) => {
              const state = offerState(o, today);
              return <Tag tone={state.tone}>{state.label}</Tag>;
            }
          },
          {
            label: "Oferecida · iniciada · concluída · de sugestão", w: "1.4fr", render: (o) => (
              <span className="mono" title={COUNTER_HINT}>
                {[ o.counters.offered, o.counters.started, o.counters.completed, o.counters.from_suggestion ].map(fmtCounter).join(" · ")}
              </span>
            )
          }
        ]}
        rows={offers}
        rowKey={(o) => o.protocol_name}
        onRowClick={canEdit ? (o) => { setDone(null); setEditing(o); } : undefined}
        empty="nenhum protocolo em uso nesta cidade"
      />
      <p style={{ margin: "8px 0 0", fontSize: 11, color: "var(--ink3)" }}>{COUNTER_HINT}</p>
      {editing && (
        <TriageOfferForm
          key={editing.protocol_name}
          offer={editing}
          all={offers}
          onCancel={() => setEditing(null)}
          onGoToSecurity={() => onNavigate?.("security")}
          onSaved={() => {
            setDone(`Catálogo atualizado: ${editing.title}`);
            setEditing(null);
            void queryClient.invalidateQueries({ queryKey: TRIAGE_CATALOG_KEY });
          }}
        />
      )}
    </Panel>
  );
}
```

- [ ] **Step 5: Rode e veja passar**

Run: `npx vitest run src/modules/protocols/TriageCatalogTab.test.tsx src/modules/Protocols.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/modules/protocols/TriageOfferForm.tsx src/modules/protocols/TriageCatalogTab.tsx src/modules/protocols/TriageCatalogTab.test.tsx
/opt/homebrew/bin/git commit -m "feat: edit the city triage catalog with step-up

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Perfil conferido na validação presencial

**Files:**
- Create: `src/modules/attendance/ProfileCheck.tsx`, `src/lib/api.attendanceProfile.test.ts`
- Modify: `src/lib/api.ts` (`AttendanceCitizen`, `verifyCitizen`), `src/lib/attendance.ts` (`MESSAGES`), `src/modules/Attendance.tsx` (`Counter`)
- Test: `src/modules/attendance/ProfileCheck.test.tsx`, `src/modules/Attendance.test.tsx`

**Interfaces:**
- Consumes: `CitizenProfile`, `GenderIdentity`, `Sex` (Task 1); `SEX_OPTIONS`, `GENDER_IDENTITY_OPTIONS`, `NO_GENDER_IDENTITY`, `birthDateProblem`, `describeProfile` (Task 2); `todayInCity` (`src/lib/campaigns.ts`).
- Produces:
  - em `src/lib/api.ts`: `AttendanceCitizen.profile?: CitizenProfile | null`; `VerifiedProfile { birth_date: string; sex: Sex; gender_identity: GenderIdentity | null }`; `verifyCitizen(cpf, code, profile: VerifiedProfile)`;
  - em `ProfileCheck.tsx`: `ProfileCheckValue { birthDate: string; sex: Sex | ""; genderIdentity: GenderIdentity | "" }`, `initialProfileCheck(declared: CitizenProfile | null)`, `profileCheckProblem(value, today): string | null`, `ProfileCheck({ declared, value, today, onChange })` — `fieldset` "Perfil conferido no documento" com "Data de nascimento (documento)", "Sexo (documento)", "Identidade de gênero (opcional)".

- [ ] **Step 1: Escreva os testes**

```ts
// src/lib/api.attendanceProfile.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import { verifyCitizen } from "./api";

afterEach(() => vi.unstubAllGlobals());

describe("validação presencial com perfil", () => {
  it("manda data de nascimento, sexo e identidade conferidos junto do documento", async () => {
    const fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
      new Response(JSON.stringify({ verification: { id: "v1" } }), { status: 201, headers: { "Content-Type": "application/json" } }));
    vi.stubGlobal("fetch", fn);
    await verifyCitizen("529.982.247-25", "123456", { birth_date: "1963-04-02", sex: "female", gender_identity: null });
    const [ url, init ] = fn.mock.calls[0] as [ string, RequestInit ];
    expect(url).toBe("/attendance/verifications");
    expect(JSON.parse(init.body as string)).toEqual({
      cpf: "529.982.247-25", code: "123456", document_checked: true,
      birth_date: "1963-04-02", sex: "female", gender_identity: null
    });
  });
});
```

```tsx
// src/modules/attendance/ProfileCheck.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { ProfileCheck, initialProfileCheck, profileCheckProblem } from "./ProfileCheck";

afterEach(cleanup);
const TODAY = "2026-10-05";
const DECLARED = { birth_date: "1963-04-02", sex: "female" as const, gender_identity: "cis_woman" as const, profile_source: "declared" as const };

describe("ProfileCheck", () => {
  it("valor inicial vem do declarado; sem perfil, tudo vazio", () => {
    expect(initialProfileCheck(DECLARED)).toEqual({ birthDate: "1963-04-02", sex: "female", genderIdentity: "cis_woman" });
    expect(initialProfileCheck(null)).toEqual({ birthDate: "", sex: "", genderIdentity: "" });
  });

  it("problema: data primeiro, depois sexo; identidade é opcional", () => {
    expect(profileCheckProblem({ birthDate: "", sex: "", genderIdentity: "" }, TODAY)).toBe("informe a data de nascimento");
    expect(profileCheckProblem({ birthDate: "1990-01-01", sex: "", genderIdentity: "" }, TODAY)).toBe("informe o sexo do documento");
    expect(profileCheckProblem({ birthDate: "1990-01-01", sex: "male", genderIdentity: "" }, TODAY)).toBeNull();
  });

  it("mostra o declarado e devolve cada correção", () => {
    const onChange = vi.fn();
    render(<ProfileCheck declared={DECLARED} value={initialProfileCheck(DECLARED)} today={TODAY} onChange={onChange} />);
    expect(screen.getByText("Declarado pelo cidadão: nascimento 02/04/1963 (63 anos) · sexo feminino · identidade de gênero Mulher cis"))
      .not.toBeNull();
    fireEvent.change(screen.getByLabelText("Sexo (documento)"), { target: { value: "male" } });
    expect(onChange).toHaveBeenLastCalledWith({ birthDate: "1963-04-02", sex: "male", genderIdentity: "cis_woman" });
    fireEvent.change(screen.getByLabelText("Identidade de gênero (opcional)"), { target: { value: "" } });
    expect(onChange).toHaveBeenLastCalledWith({ birthDate: "1963-04-02", sex: "female", genderIdentity: "" });
  });

  it("perfil já conferido no posto é dito", () => {
    render(<ProfileCheck declared={{ ...DECLARED, profile_source: "verified" }} value={initialProfileCheck(DECLARED)} today={TODAY} onChange={vi.fn()} />);
    expect(screen.getByText(/já conferido no posto/)).not.toBeNull();
  });
});
```

Em `src/modules/Attendance.test.tsx`, troque o teste `it("busca, exige a caixa do documento e valida", ...)` inteiro por:

```tsx
  it("busca, exige a caixa do documento e o perfil conferido, e valida", async () => {
    mocked(api.lookupCitizen).mockResolvedValue(found);
    mocked(api.verifyCitizen).mockResolvedValue(undefined);
    renderAttendance();
    fireEvent.change(await screen.findByLabelText("CPF do cidadão (validação)"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Código de validação"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar validação" }));
    expect(await screen.findByText("(**) *****-5432")).not.toBeNull();
    expect(screen.getByText("triage-respiratoria")).not.toBeNull();
    const validate = screen.getByRole("button", { name: "Validar cadastro" }) as HTMLButtonElement;
    fireEvent.click(screen.getByLabelText("Conferi o documento com foto e o CPF confere"));
    expect(validate.disabled).toBe(true);
    fireEvent.change(screen.getByLabelText("Data de nascimento (documento)"), { target: { value: "1990-05-10" } });
    fireEvent.change(screen.getByLabelText("Sexo (documento)"), { target: { value: "male" } });
    expect(validate.disabled).toBe(false);
    fireEvent.click(validate);
    await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456",
      { birth_date: "1990-05-10", sex: "male", gender_identity: null }));
    expect(await screen.findByText("Cadastro validado")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Próximo atendimento" }));
    expect((screen.getByLabelText("CPF do cidadão (validação)") as HTMLInputElement).value).toBe("");
  });
```

E acrescente, no fim do arquivo:

```tsx
describe("Attendance — perfil conferido no documento", () => {
  const declared = {
    ...found,
    citizen: {
      ...found.citizen,
      profile: { birth_date: "1963-04-02", sex: "female" as const, gender_identity: "cis_woman" as const, profile_source: "declared" as const }
    }
  };

  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date("2026-10-05T23:30:00-03:00"));
    for (const fn of [ api.fetchCurrentSession, api.lookupCitizen, api.verifyCitizen, api.listActiveUnits, api.getMyProfessional ]) {
      mocked(fn).mockReset();
    }
    mocked(api.fetchCurrentSession).mockResolvedValue(session("citizen_verifier"));
    mocked(api.listActiveUnits).mockResolvedValue([]);
    mocked(api.getMyProfessional).mockResolvedValue(null);
    mocked(api.verifyCitizen).mockResolvedValue(undefined);
  });
  afterEach(() => vi.useRealTimers());

  async function openFound(result: typeof found) {
    mocked(api.lookupCitizen).mockResolvedValue(result);
    renderAttendance();
    fireEvent.change(await screen.findByLabelText("CPF do cidadão (validação)"), { target: { value: "52998224725" } });
    fireEvent.change(screen.getByLabelText("Código de validação"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar validação" }));
    await screen.findByText("(**) *****-5432");
    fireEvent.click(screen.getByLabelText("Conferi o documento com foto e o CPF confere"));
    return screen.getByRole("button", { name: "Validar cadastro" }) as HTMLButtonElement;
  }

  it("mostra o declarado, já preenche os campos e envia o conferido", async () => {
    const validate = await openFound(declared);
    expect(screen.getByText("Declarado pelo cidadão: nascimento 02/04/1963 (63 anos) · sexo feminino · identidade de gênero Mulher cis"))
      .not.toBeNull();
    expect((screen.getByLabelText("Data de nascimento (documento)") as HTMLInputElement).value).toBe("1963-04-02");
    fireEvent.click(validate);
    await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456",
      { birth_date: "1963-04-02", sex: "female", gender_identity: "cis_woman" }));
  });

  it("o atendente corrige o sexo e tira a identidade", async () => {
    const validate = await openFound(declared);
    fireEvent.change(screen.getByLabelText("Sexo (documento)"), { target: { value: "male" } });
    fireEvent.change(screen.getByLabelText("Identidade de gênero (opcional)"), { target: { value: "" } });
    fireEvent.click(validate);
    await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456",
      { birth_date: "1963-04-02", sex: "male", gender_identity: null }));
  });

  it("sem perfil declarado: diz isso e só valida com data e sexo", async () => {
    const validate = await openFound(found);
    expect(screen.getByText("Declarado pelo cidadão: sem perfil declarado")).not.toBeNull();
    expect(validate.disabled).toBe(true);
    expect(screen.getByText("informe a data de nascimento")).not.toBeNull();
  });

  it("data futura trava a validação (às 23h30, amanhã ainda é futuro)", async () => {
    const validate = await openFound(declared);
    fireEvent.change(screen.getByLabelText("Data de nascimento (documento)"), { target: { value: "2026-10-06" } });
    expect(screen.getByText("a data de nascimento não pode ser no futuro")).not.toBeNull();
    expect(validate.disabled).toBe(true);
    fireEvent.change(screen.getByLabelText("Data de nascimento (documento)"), { target: { value: "2026-10-05" } });
    expect(validate.disabled).toBe(false);
  });

  it("recusa do servidor pela data é traduzida", async () => {
    mocked(api.verifyCitizen).mockRejectedValue(new ApiError(422, { error: "invalid_birth_date" }, "x"));
    const validate = await openFound(declared);
    fireEvent.click(validate);
    expect(await screen.findByText("data de nascimento inválida — confira no documento")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/api.attendanceProfile.test.ts src/modules/attendance/ProfileCheck.test.tsx src/modules/Attendance.test.tsx`
Expected: FAIL — `./ProfileCheck` não existe e `verifyCitizen` não manda o perfil.

- [ ] **Step 3: Cliente (`src/lib/api.ts`)**

Troque

```ts
export interface AttendanceCitizen {
  id: string; cpf_masked: string; phone_masked: string; created_at: string;
  verification_level: "declared" | "verified";
}
```

por

```ts
export interface AttendanceCitizen {
  id: string; cpf_masked: string; phone_masked: string; created_at: string;
  verification_level: "declared" | "verified";
  // Módulo 15 (contratos §4.4): o perfil declarado do par, para o atendente
  // confirmar ou corrigir. Opcional porque uma API anterior omite a chave.
  profile?: CitizenProfile | null;
}
// O que o atendente conferiu no documento (spec §5.4).
export interface VerifiedProfile { birth_date: string; sex: Sex; gender_identity: GenderIdentity | null }
```

e troque

```ts
export async function verifyCitizen(cpf: string, code: string): Promise<void> {
  await jsonFetch<unknown>(`${ATTENDANCE_BASE}/verifications`, {
    method: "POST", body: JSON.stringify({ cpf, code, document_checked: true })
  });
}
```

por

```ts
export async function verifyCitizen(cpf: string, code: string, profile: VerifiedProfile): Promise<void> {
  await jsonFetch<unknown>(`${ATTENDANCE_BASE}/verifications`, {
    method: "POST", body: JSON.stringify({ cpf, code, document_checked: true, ...profile })
  });
}
```

Em `src/lib/attendance.ts`, acrescente ao `MESSAGES`, depois de `invalid_neighborhood`:

```ts
  invalid_neighborhood: "bairro inválido — escolha outro da lista",
  invalid_birth_date: "data de nascimento inválida — confira no documento",
  invalid_sex: "informe o sexo que consta no documento",
  invalid_gender_identity: "identidade de gênero inválida — escolha da lista"
```

(a linha de `invalid_neighborhood` já existe; só ganha a vírgula.)

- [ ] **Step 4: Componente**

```tsx
// src/modules/attendance/ProfileCheck.tsx
// Perfil conferido no documento na validação presencial (módulo 15; spec
// §5.4; contratos §4.4). Mostra o que o cidadão declarou e pede data de
// nascimento e sexo como estão no documento; a identidade de gênero é
// opcional e nunca entra na elegibilidade.
import type { CSSProperties } from "react";
import type { CitizenProfile, GenderIdentity, Sex } from "../../lib/api";
import { GENDER_IDENTITY_OPTIONS, NO_GENDER_IDENTITY, SEX_OPTIONS, birthDateProblem, describeProfile } from "../../lib/profile";
import { inputStyle } from "../../components/formStyles";

export interface ProfileCheckValue { birthDate: string; sex: Sex | ""; genderIdentity: GenderIdentity | "" }

export function initialProfileCheck(declared: CitizenProfile | null): ProfileCheckValue {
  return { birthDate: declared?.birth_date ?? "", sex: declared?.sex ?? "", genderIdentity: declared?.gender_identity ?? "" };
}

export function profileCheckProblem(value: ProfileCheckValue, today: string): string | null {
  return birthDateProblem(value.birthDate, today) ?? (value.sex === "" ? "informe o sexo do documento" : null);
}

export function ProfileCheck({ declared, value, today, onChange }: {
  declared: CitizenProfile | null; value: ProfileCheckValue; today: string; onChange(next: ProfileCheckValue): void;
}) {
  const problem = profileCheckProblem(value, today);
  return (
    <fieldset aria-label="Perfil conferido no documento" style={box}>
      <legend style={{ fontSize: 13, fontWeight: 600 }}>Perfil conferido no documento</legend>
      <p style={hint}>
        Declarado pelo cidadão: {describeProfile(declared, today)}
        {declared?.profile_source === "verified" ? " · já conferido no posto" : ""}
      </p>
      <label style={label}>
        Data de nascimento (documento)
        <input type="date" value={value.birthDate} max={today} style={inputStyle}
          onChange={(e) => onChange({ ...value, birthDate: e.target.value })} />
      </label>
      <label style={label}>
        Sexo (documento)
        <select value={value.sex} style={inputStyle} onChange={(e) => onChange({ ...value, sex: e.target.value as Sex | "" })}>
          <option value="">escolha…</option>
          {SEX_OPTIONS.map((o) => <option key={o.value} value={o.value}>{o.label}</option>)}
        </select>
      </label>
      <label style={label}>
        Identidade de gênero (opcional)
        <select value={value.genderIdentity} style={inputStyle}
          onChange={(e) => onChange({ ...value, genderIdentity: e.target.value as GenderIdentity | "" })}>
          <option value="">{NO_GENDER_IDENTITY}</option>
          {GENDER_IDENTITY_OPTIONS.map((o) => <option key={o.value} value={o.value}>{o.label}</option>)}
        </select>
      </label>
      {problem && <small style={hint}>{problem}</small>}
    </fieldset>
  );
}

const box: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, maxWidth: 360, border: "1px solid var(--rule)", borderRadius: 8, padding: "8px 12px", margin: 0 };
const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const hint: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
```

O texto "Declarado pelo cidadão: …" fica num `<p>` só com nós de texto, para o `getByText` do teste achar a frase inteira.

- [ ] **Step 5: Balcão (`src/modules/Attendance.tsx`, função `Counter`)**

1. No import de `../lib/api`, acrescente `type Sex` à lista. Acrescente os imports:

```tsx
import { todayInCity } from "../lib/campaigns";
import { ProfileCheck, initialProfileCheck, profileCheckProblem, type ProfileCheckValue } from "./attendance/ProfileCheck";
```

2. Depois de `const [ checked, setChecked ] = useState(false);` (dentro de `Counter`):

```tsx
  const [ profile, setProfile ] = useState<ProfileCheckValue>(() => initialProfileCheck(null));
```

3. Em `reset()`, troque

```tsx
    setState("form"); setCpf(""); setCode(""); setChecked(false); setFound(null); setError(null);
```

por

```tsx
    setState("form"); setCpf(""); setCode(""); setChecked(false); setFound(null); setError(null);
    setProfile(initialProfileCheck(null));
```

4. Em `search()`, troque `setFound(result);` por:

```tsx
      setFound(result);
      setProfile(initialProfileCheck(result.citizen.profile ?? null));
```

5. Troque o começo de `validate()` e a chamada:

```tsx
  async function validate() {
    if (busy || !checked) return;
```

por

```tsx
  // "Hoje" no fuso da cidade: às 23h30 de São Paulo, amanhã ainda é futuro.
  const profileProblem = profileCheckProblem(profile, todayInCity());

  async function validate() {
    if (busy || !checked || profileProblem) return;
```

e

```tsx
      await verifyCitizen(cpf, code);
```

por

```tsx
      await verifyCitizen(cpf, code, {
        birth_date: profile.birthDate, sex: profile.sex as Sex, gender_identity: profile.genderIdentity || null
      });
```

6. No JSX do estado `found`, logo antes de

```tsx
            <label style={{ ...labelStyle, flexDirection: "row", alignItems: "center", gap: 8 }}>
              <input type="checkbox" checked={checked} onChange={(e) => setChecked(e.target.checked)} />
```

acrescente

```tsx
            <ProfileCheck declared={found.citizen.profile ?? null} value={profile} today={todayInCity()} onChange={setProfile} />
```

7. No botão "Validar cadastro", troque `disabled={!checked || busy}` por `disabled={!checked || busy || profileProblem !== null}` e `style={(!checked || busy) ? disabledButtonStyle : buttonStyle}` por `style={(!checked || busy || profileProblem !== null) ? disabledButtonStyle : buttonStyle}`.

- [ ] **Step 6: Rode e veja passar**

Run: `npx vitest run src/lib/api.attendanceProfile.test.ts src/modules/attendance/ProfileCheck.test.tsx src/modules/Attendance.test.tsx && npx tsc --noEmit`
Expected: PASS. `grep -rn "verifyCitizen(" src --include=*.ts --include=*.tsx` mostra só chamadas com três argumentos (fora dos testes que usam o mock).

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/lib/api.ts src/lib/api.attendanceProfile.test.ts src/lib/attendance.ts src/modules/attendance/ProfileCheck.tsx src/modules/attendance/ProfileCheck.test.tsx src/modules/Attendance.tsx src/modules/Attendance.test.tsx
/opt/homebrew/bin/git commit -m "feat: confirm birth date and sex from the document on presencial verification

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Suíte, build, revisão e prova no navegador

- [ ] **Step 1: O `admin` não compartilha tipos com o dashboard**

Run (da raiz do monorepo):

```bash
grep -rn "dashboard/src\|@rota-saude/dashboard" apps/admin/src apps/admin/package.json apps/admin/tsconfig.json
```

Expected: nenhuma linha. O `admin` não muda neste módulo (spec, "Afeta").

- [ ] **Step 2: Suíte, tipos e build**

Run: `cd apps/dashboard/.claude/mod15 && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde. A CI roda os mesmos três; `dist/` está no `.gitignore`.

```bash
/opt/homebrew/bin/git status --short
/opt/homebrew/bin/git log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e a contagem `0`. Se o symlink aparecer, **não** o adicione.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§7 e §9.3), o ADR 0027 e o arquivo de contratos (§1, §2 e §4). Pontos de atenção:
- o JSON do editor continua a fonte: abrir o painel não reescreve a definição, e regra fora do subconjunto fica intacta;
- o construtor nunca emite árvore inválida (linha incompleta fica fora) e nunca usa variável fora do lugar (`fieldsFor` por contexto);
- campo opcional vazio sai ausente do JSON (`offer.title`, `summary`, `retake_after_days`, `offer` inteiro, `suggestions`, `when`);
- `PUT /triage_catalog` só pelo `SensitiveAction` com `requiresStepUp`, só para `municipal_admin`; os outros papéis nunca veem o formulário; quem não lê protocolos nunca chama `GET /triage_catalog`;
- contador `null` sempre "< 5";
- a identidade de gênero nunca entra em condição; nenhum dado de perfil vai para URL ou log (só corpo de POST);
- "hoje" (situação do catálogo, data de nascimento) é sempre `todayInCity()`;
- nenhum código do wpda importado; o visual vem de `wpdaLook.ts`.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Rode o api da branch do módulo 15 na porta **3032**, com a semente do módulo (spec §10: "Saúde do idoso", "Saúde mental" com sugestão para o aprofundamento, catálogo de Curitiba com o idoso restrito a dois bairros). Depois rode o Vite do worktree apontando para ele:

```bash
cd apps/dashboard/.claude/mod15 && VITE_API_PROXY_TARGET=http://localhost:3032 npx vite --port 5180 --host 0.0.0.0
```

Abra `http://curitiba.localhost:5180/dashboard/`. O usuário faz o login; não digite senha nem TOTP. Confira com screenshot:
- como autor da `SignatureCrew`, no Editor de protocolo:
  - carregar "Saúde do idoso": o painel mostra título, resumo, "idade a partir de 60 anos" e "1 ano (365 dias)";
  - trocar a idade para 65 no construtor e ver o JSON mudar; trocar no JSON e ver o construtor mudar;
  - "Como o cidadão vê" com as perguntas no visual do wpda;
  - simulador: 62 anos → elegível; 8 anos → não elegível. Em "Saúde mental", pontuação alta → o aprofundamento "sugere";
- como `admin@curitiba.demo`, em Protocolos → "Catálogo de triagens":
  - as frases de elegibilidade e restrição (dois bairros), o período e os contadores (com "< 5" onde couber);
  - pausar "Saúde mental — aprofundamento" pede step-up e a linha passa a "pausada";
- como `admin@curitiba.demo` (que tem `citizen_verifier` em dev), em Atendimento → Balcão, com um código gerado no wpda: o perfil declarado aparece, a data e o sexo vêm preenchidos e a validação manda os dois.

- [ ] **Step 5:** **Pare.** O merge do dashboard só vem depois do merge do api, e só com autorização explícita do usuário. A ordem de deploy é contracts → api → wpda e dashboard (spec §11).

---

## Nota: incorporado ao contrato

As cinco divergências levantadas na escrita deste plano foram aceitas e já estão no arquivo de contratos; o plano segue o texto atual:

1. **§4.2:** o step-up responde 401 `mfa_required` (não `step_up_required`).
2. **§4.4:** a validação com perfil recusa com 422 `invalid_birth_date`, `invalid_sex` e `invalid_gender_identity`; o lookup devolve o perfil do par do código em `citizen.profile`.
3. **§4.2:** `position` é inteiro ≥ 1.
4. **§4.3:** o simulador com definição inválida responde 200 com `eligible: false`, `suggestions: []` e `errors`, nunca 422 (o cliente não tem ramo de 422).
5. **§1:** "idade igual a N" é escrito como `all` de `gte` + `lte` com o mesmo N, porque o `eq` de hoje compara texto e `profile.age` é inteiro.

## Self-review

- **Cobertura da spec:**
  - §7, construtor de condições (grupos E/OU, linhas, NÃO por linha e por grupo, campos por contexto, frase sempre visível, regra avançada em leitura): Tasks 3, 4 e 5;
  - §7, editor de protocolo (painel "Oferta e sugestões" ao lado do JSON, lê e escreve `offer` e `suggestions`, edição nos dois sentidos): Tasks 6 e 7;
  - §7, pré-visualização no visual do wpda: Task 8; simulador de perfil (`simulate_offer`): Tasks 1 e 9;
  - §7, aba "Catálogo de triagens" (título, elegibilidade e restrição em frase, período, oferecida/pausada, ordem, edição com step-up só do `municipal_admin`): Tasks 10, 11 e 12;
  - §7, contadores com supressão: Tasks 10 e 11;
  - §5.4, validação presencial com data de nascimento e sexo do documento: Task 13;
  - §9.3 (ida e volta, frase, regra avançada, simulador, aba com step-up): Tasks 3, 4, 5, 9 e 12;
  - §9.5, prova no navegador: Task 14.
- **Placeholders:** nenhum. As Tasks 7, 8, 9, 11 e 13 alteram arquivos existentes por trechos exatos (de/para); a Task 12 substitui o arquivo da Task 11 inteiro.
- **Consistência de nomes:**
  - `ConditionTree`, `Sex`, `GenderIdentity`, `CitizenProfile`, `TriageOffer`, `TriageOfferFields`, `listTriageCatalog`, `updateTriageOffer`, `simulateOffer` saem da Task 1 e são usados nas Tasks 2, 3, 9, 10, 11, 12 e 13;
  - `fieldsFor`, `fromTree`, `toTree`, `treeKey`, `tiersOf`, `ConditionField` saem da Task 3 e são usados nas Tasks 4, 5, 7, 9, 11 e 12;
  - `describeCondition(tree, fields, emptyText)` sai da Task 4 e é usado nas Tasks 5, 11 e 12;
  - `ConditionBuilder({ label, fields, value, emptyText, onChange })` sai da Task 5 e é usado nas Tasks 7 e 12;
  - `readOffer`, `writeOffer`, `writeSuggestions`, `parseRetake`, `retakeLabel`, `suggestionProblem`, `protocolNameOf` saem da Task 6 e são usados na Task 7;
  - `TRIAGE_CATALOG_KEY`, `canReadCatalog`, `canEditCatalog`, `offerState`, `periodPhrase`, `fmtCounter`, `formFrom`, `offerFormProblem`, `offerPayload`, `triageCatalogError` saem da Task 10 e são usados nas Tasks 11 e 12;
  - `SEX_OPTIONS`, `MAX_AGE`, `birthDateProblem`, `describeProfile` saem da Task 2 e são usados nas Tasks 3, 9 e 13;
  - fixtures: `SUGGESTION_DEF`/`NEIGHBORHOODS` (Task 3), `catalogOffer`/`RESPIRATORY` (Task 10), `sessionWith`/`renderWithProviders` (existentes, módulo 12).
- **Review Focus:** cada um dos cinco itens tem teste na task dona:
  - 1: Tasks 5 e 7;
  - 2: Tasks 3 e 5;
  - 3: Tasks 4 e 5;
  - 4: Tasks 10 e 11;
  - 5: Tasks 2 e 13.
