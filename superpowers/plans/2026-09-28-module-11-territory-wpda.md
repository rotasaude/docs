# Módulo 11 — Território (wpda, fatia 3) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** No canal web do cidadão, a pessoa informa o bairro na escolha "Para quem é esta triagem?" (ou prefere não informar), pode trocá-lo depois, e vê o bloco "Sua unidade de referência" no resultado da triagem, na área logada.

**Architecture:** Tipos e chamadas novas em `src/lib/citizenApi.ts`, com os campos novos normalizados na borda (ausente vira `null` ou `[]`), como já é feito para `attendance` e agendamentos. Regras puras (busca sem acento, endereço, rótulo do tipo de unidade) em `src/lib/territory.ts`, testadas sem React. Um componente de escolha (`NeighborhoodPicker`) usado pela `PeopleStep` em dois modos (antes de começar e "Trocar bairro"), e um bloco `ReferenceUnits` mostrado só na área logada (`ResultStep`), sempre a partir de `GET /citizen/triages/:id`. O relatório público (`/r/:token`, componente `Report`) **não** mostra a unidade de referência (decisão do usuário, 2026-09-28).

**Tech Stack:** React 18 + TypeScript (strict, `noUnusedLocals`), Vite 5, Vitest 2 + Testing Library com jest-dom e user-event. Sem TanStack Query nas telas do cidadão (estado local, como hoje). Nenhuma dependência nova.

**Spec:** `docs/superpowers/specs/2026-09-28-module-11-territory-design.md` (§4.1 "Cidadão", §4.2, §6, §7.2 "wpda"). ADR: `docs/adr/0023.md`. Depende do plano do api (`docs/superpowers/plans/2026-09-28-module-11-territory-api.md`, fatia 3) estar mergeado **antes** do merge deste.

## Global Constraints

- Rotas consumidas (spec §4.1), sem campo além destes:
  - `GET /citizen/neighborhoods` → bairros ativos, por nome, `{ neighborhoods: [{id, name}] }` (envelope confirmado com o api);
  - `GET /citizen/people` → cada pessoa ganha `neighborhood: {id, name} | null`;
  - `POST /citizen/conversations` → aceita `neighborhood_id` opcional; bairro inválido = 422 `invalid_neighborhood` (antes de criar o CPF);
  - `POST /citizen/people/:id/neighborhood` → `{ neighborhood_id | null }`; CPF fora da sessão = 404; bairro inativo ou inexistente = 422 `invalid_neighborhood`;
  - `GET /citizen/triages/:id` → `reference_units: [{id, name, kind, address: {street, number, complement, zip}}]`; `address` sempre vem, mas cada campo pode ser `null` (contrato confirmado com o api).
- **Relatório público sem unidade de referência:** `GET /r/:token` e o componente `Report` não leem nem mostram `reference_units` (decisão do usuário, 2026-09-28). O bloco só aparece na área logada, a partir de `GET /citizen/triages/:id`.
- `GET /citizen/triages` (lista do histórico) não traz `reference_units` no contrato; a `HistoryStep` não mostra o bloco.
- "Prefiro não informar" é sempre permitido: no começo, **não** manda `neighborhood_id`; na troca, manda `neighborhood_id: null`.
- O bairro só é mandado com o CPF, depois do consentimento (a tela de pessoas já vem depois do termo; ADR 0017).
- A unidade de referência **informa, nunca restringe** (ADR 0023). Lista vazia = bloco ausente; várias = todas.
- `kind` da unidade: `ubs | upa | hospital | other` (`HealthUnit::KINDS` no api).
- Cidade sem bairros (sem semente, spec §9 "Sem semente, nada quebra"): a pergunta não aparece e a triagem segue sem bairro.
- wpda: texto interativo com pelo menos 18 px e alvos com pelo menos 48 px (usar `BigButton`, `Field`, `Screen` de `ui.tsx`).
- Testes que dependem de data fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(...)` em `beforeEach` (só `Date`: o polling da `ResultStep` usa `setTimeout` real).
- O WhatsApp está descontinuado; o wpda web é o único canal do cidadão.
- Nunca `git add -A` (o worktree tem symlink de `node_modules`); adicione arquivos pelo nome.
- Commits em inglês, Conventional Commits com tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

- Worktree já criado: `apps/wpda/.claude/mod11`, branch `feat/mod-11-territory`, a partir de `origin/main` (`bb42097`). Confira antes de começar:

  ```bash
  cd apps/wpda/.claude/mod11 && /opt/homebrew/bin/git status -sb && /opt/homebrew/bin/git log --oneline -1
  ```

  Esperado: `## feat/mod-11-territory...origin/main` sem mudanças, topo `bb42097`.

- `node_modules` do app principal, por symlink (o `.gitignore` já ignora `node_modules` sem barra, então o symlink não aparece no `git status`; mesmo assim, nunca `git add -A`):

  ```bash
  cd apps/wpda/.claude/mod11 && [ -e node_modules ] || ln -s ../../node_modules node_modules
  ```

- Testes (um arquivo ou a suíte inteira):

  ```bash
  cd apps/wpda/.claude/mod11 && npx vitest run <arquivos>
  cd apps/wpda/.claude/mod11 && npm test
  ```

- Checagem de tipos antes de cada commit:

  ```bash
  cd apps/wpda/.claude/mod11 && npx tsc --noEmit
  ```

- Anote a contagem da suíte antes da Task 1 (`npm test`, linha `Tests  N passed`); nenhum teste que já existe pode cair.

## Review Focus

1. **Cidade sem bairros ou api antiga:** `GET /citizen/neighborhoods` devolve `[]`, 404 ou falha de rede. A pessoa esperaria entrar na triagem como antes, sem pergunta nem erro, e sem "Trocar bairro" na tela. Teste: Task 3 ("cidade sem bairros" e "lista com erro").
2. **Bairro desativado entre abrir a lista e tocar nele:** o `POST /citizen/conversations` volta 422 `invalid_neighborhood`. A pessoa esperaria ver a lista de novo, já sem aquele bairro, com um aviso, e não o erro genérico. Se a lista voltar vazia, a triagem segue sem bairro. Teste: Task 3 ("422 relê a lista" e "422 com a lista vazia"), e Task 3 em `Flow.test.tsx`.
3. **Endereço incompleto:** unidade só com CEP, com número sem logradouro, com campos em branco (`"  "`), com CEP de 7 dígitos, ou todos os campos `null`. A pessoa esperaria ler só o que existe, nunca `undefined`, `null` ou `CEP ` solto. Teste: Task 1 (`formatAddress`) e Task 2 (varredura do texto do bloco).
4. **Busca digitada sem acento ou em maiúsculas:** "sao francisco" ou "BATEL" encontram "São Francisco" e "Batel". Teste: Task 1 (`filterNeighborhoods`) e Task 3 (tela).
5. **Toque duplo no bairro:** num celular lento, dois toques não podem abrir duas conversas. Os botões ficam desabilitados enquanto o envio não volta. Teste: Task 3 ("toque duplo chama uma vez só").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `src/lib/citizenApi.ts` | tipos `Neighborhood`, `UnitAddress`, `ReferenceUnit`; `Person.neighborhood`; `TriageSummary.reference_units`; `neighborhoods()`, `setNeighborhood()`, `start({ neighborhoodId })`; normalização | 1 |
| `src/lib/territory.ts` | `filterNeighborhoods`, `formatAddress`, `formatZip`, `unitKindLabel` | 1 |
| `src/modules/ReferenceUnits.tsx` | bloco "Sua unidade de referência" | 2 |
| `src/modules/citizen/ResultStep.tsx` | mostra o bloco (com e sem relatório pronto) | 2 |
| `src/modules/Report.test.tsx` | prova que o relatório público não mostra o bloco | 2 |
| `src/modules/citizen/NeighborhoodPicker.tsx` | lista com busca + "Prefiro não informar" | 3 |
| `src/modules/citizen/PeopleStep.tsx`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/ui.tsx` | pergunta antes de começar; 422; mensagem | 3 |
| `src/modules/citizen/PeopleStep.tsx` | "Trocar bairro" | 4 |

---

### Task 1: Tipos, cliente da API e regras puras

**Files:**
- Modify: `src/lib/citizenApi.ts`
- Create: `src/lib/territory.ts`
- Test: `src/lib/territory.test.ts` (novo), `src/lib/citizenApi.test.ts`

**Interfaces:**
- Produces (em `src/lib/citizenApi.ts`):
  - `interface Neighborhood { id: string; name: string }`
  - `interface UnitAddress { street: string | null; number: string | null; complement: string | null; zip: string | null }`
  - `interface ReferenceUnit { id: string; name: string; kind: string; address: UnitAddress }`
  - `Person.neighborhood?: Neighborhood | null` (normalizado para `null` em `people()`)
  - `TriageSummary.reference_units?: ReferenceUnit[]` (normalizado para `[]` em `triage()` e `triages()`)
  - `citizenApi.neighborhoods(): Promise<Neighborhood[]>`
  - `citizenApi.setNeighborhood(citizenId: string, neighborhoodId: string | null): Promise<unknown>`
  - `citizenApi.start({ citizenId?, cpf?, neighborhoodId?, consentVersion })` — manda `neighborhood_id` só quando `neighborhoodId` veio.
- Produces (em `src/lib/territory.ts`): `filterNeighborhoods(list: Neighborhood[], query: string): Neighborhood[]`, `formatZip(zip): string | null`, `formatAddress(a: UnitAddress | null | undefined): string | null`, `unitKindLabel(kind): string | null`.

- [ ] **Step 1: Escreva os testes das regras puras**

```ts
// src/lib/territory.test.ts
import { describe, expect, it } from "vitest";
import { filterNeighborhoods, formatAddress, formatZip, unitKindLabel } from "./territory";

const list = [
  { id: "n1", name: "Batel" },
  { id: "n2", name: "São Francisco" },
  { id: "n3", name: "Santa Felicidade" }
];

describe("filterNeighborhoods", () => {
  it("busca vazia devolve a lista inteira, na ordem da API", () => {
    expect(filterNeighborhoods(list, "")).toEqual(list);
    expect(filterNeighborhoods(list, "   ")).toEqual(list);
  });
  it("ignora acento e maiúsculas", () => {
    expect(filterNeighborhoods(list, "sao francisco").map(n => n.id)).toEqual([ "n2" ]);
    expect(filterNeighborhoods(list, "BATEL").map(n => n.id)).toEqual([ "n1" ]);
  });
  it("casa pedaço do nome", () => {
    expect(filterNeighborhoods(list, "fel").map(n => n.id)).toEqual([ "n3" ]);
  });
  it("sem resultado devolve lista vazia", () => {
    expect(filterNeighborhoods(list, "xyz")).toEqual([]);
  });
});

describe("formatZip", () => {
  it("8 dígitos viram 00000-000", () => expect(formatZip("80730000")).toBe("80730-000"));
  it("aceita já formatado", () => expect(formatZip("80730-000")).toBe("80730-000"));
  it.each([ null, undefined, "", "8073000", "807300001" ])("recusa %s", (zip) => expect(formatZip(zip)).toBeNull());
});

describe("formatAddress", () => {
  it("endereço completo", () => {
    expect(formatAddress({ street: "Rua Padre Anchieta", number: "1500", complement: "Térreo", zip: "80730000" }))
      .toBe("Rua Padre Anchieta, 1500 — Térreo · CEP 80730-000");
  });
  it("sem número nem complemento", () => {
    expect(formatAddress({ street: "Rua Padre Anchieta", number: null, complement: null, zip: "80730000" }))
      .toBe("Rua Padre Anchieta · CEP 80730-000");
  });
  it("sem CEP", () => {
    expect(formatAddress({ street: "Rua Padre Anchieta", number: "1500", complement: null, zip: null }))
      .toBe("Rua Padre Anchieta, 1500");
  });
  it("só CEP", () => {
    expect(formatAddress({ street: null, number: null, complement: null, zip: "80730000" })).toBe("CEP 80730-000");
  });
  it("número e complemento sem logradouro não dizem nada sozinhos", () => {
    expect(formatAddress({ street: null, number: "1500", complement: "Sala 2", zip: null })).toBeNull();
  });
  it("campos em branco contam como ausentes", () => {
    expect(formatAddress({ street: "  ", number: " ", complement: "", zip: "  " })).toBeNull();
  });
  it("CEP mal formado é omitido, o resto fica", () => {
    expect(formatAddress({ street: "Rua A", number: "1", complement: null, zip: "8073000" })).toBe("Rua A, 1");
  });
  it.each([ null, undefined ])("address %s vira null", (a) => expect(formatAddress(a)).toBeNull());
  it("nunca escreve undefined nem null, com qualquer combinação de campos ausentes", () => {
    const values = [ null, undefined, "", "x" ] as const;
    for (const street of values) for (const number of values) for (const complement of values) for (const zip of [ null, "80730000" ]) {
      const out = formatAddress({ street, number, complement, zip } as never) ?? "";
      expect(out).not.toMatch(/undefined|null/);
    }
  });
});

describe("unitKindLabel", () => {
  it.each([ [ "ubs", "UBS" ], [ "upa", "UPA" ], [ "hospital", "Hospital" ], [ "other", "Unidade de saúde" ] ])(
    "%s → %s", (kind, label) => expect(unitKindLabel(kind)).toBe(label));
  it("tipo desconhecido ou ausente não mostra rótulo", () => {
    expect(unitKindLabel("clinica_nova")).toBeNull();
    expect(unitKindLabel(null)).toBeNull();
    expect(unitKindLabel(undefined)).toBeNull();
  });
});
```

- [ ] **Step 2: Escreva os testes do cliente**

Acrescente ao fim do `describe("citizenApi", ...)` em `src/lib/citizenApi.test.ts`:

```ts
  it("people normaliza neighborhood ausente para null e preserva o presente", async () => {
    mockFetch(200, { people: [
      { id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared" },
      { id: "p2", cpf_masked: "***.111.222-**", verification_level: "declared", neighborhood: { id: "n1", name: "Batel" } }
    ] });
    const { people } = await citizenApi.people();
    expect(people[0].neighborhood).toBeNull();
    expect(people[1].neighborhood).toEqual({ id: "n1", name: "Batel" });
  });

  it("neighborhoods faz GET /citizen/neighborhoods e desembrulha { neighborhoods }", async () => {
    const fn = mockFetch(200, { neighborhoods: [ { id: "n1", name: "Batel" } ] });
    expect(await citizenApi.neighborhoods()).toEqual([ { id: "n1", name: "Batel" } ]);
    const [url, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(url).toBe("/citizen/neighborhoods");
    expect(init.method).toBe("GET");
  });

  it("start manda neighborhood_id quando veio", async () => {
    const fn = mockFetch(201, { conversation_id: "c", citizen_id: "p", resumed: false, step: {} });
    await citizenApi.start({ cpf: "529.982.247-25", neighborhoodId: "n1", consentVersion: "1" });
    const init = (fn.mock.calls[0] as unknown as [string, RequestInit])[1];
    expect(JSON.parse(init.body as string)).toEqual({ cpf: "529.982.247-25", neighborhood_id: "n1", consent_version: "1" });
  });

  it("start sem bairro não manda a chave neighborhood_id", async () => {
    const fn = mockFetch(201, { conversation_id: "c", citizen_id: "p", resumed: false, step: {} });
    await citizenApi.start({ citizenId: "p1", consentVersion: "1" });
    const body = JSON.parse((fn.mock.calls[0] as unknown as [string, RequestInit])[1].body as string);
    expect("neighborhood_id" in body).toBe(false);
  });

  it("setNeighborhood manda POST com o id ou null", async () => {
    const fn = mockFetch(200, {});
    await citizenApi.setNeighborhood("p1", null);
    const [url, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(url).toBe("/citizen/people/p1/neighborhood");
    expect(init.method).toBe("POST");
    expect(JSON.parse(init.body as string)).toEqual({ neighborhood_id: null });
  });

  it("setNeighborhood com bairro inválido rejeita com invalid_neighborhood", async () => {
    mockFetch(422, { error: "invalid_neighborhood" });
    await expect(citizenApi.setNeighborhood("p1", "n9")).rejects.toEqual(new ApiError(422, "invalid_neighborhood"));
  });

  it("triage() sem reference_units normaliza para [] e preserva a lista quando vem", async () => {
    const base = {
      id: "t1", status: "completed", tier: "alta", priority: 1, created_at: "2026-09-28T12:00:00Z",
      completed_at: "2026-09-28T12:05:00Z", report_url: null, consent_active: true, origin_phone_masked: null
    };
    mockFetch(200, base);
    expect((await citizenApi.triage("t1")).reference_units).toEqual([]);

    const units = [ { id: "u1", name: "UBS Batel", kind: "ubs",
      address: { street: "Rua Padre Anchieta", number: null, complement: null, zip: null } } ];
    mockFetch(200, { ...base, reference_units: units });
    expect((await citizenApi.triage("t1")).reference_units).toEqual(units);
  });
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/lib/territory.test.ts src/lib/citizenApi.test.ts`
Expected: FAIL (`./territory` não existe; `neighborhoods`/`setNeighborhood` não são funções; `reference_units` `undefined`).

- [ ] **Step 4: Implemente o cliente**

Em `src/lib/citizenApi.ts`, depois de `export interface Option ...`:

```ts
// Território (spec 2026-09-28 §4.1; ADR 0023).
export interface Neighborhood { id: string; name: string }

// Endereço em texto da unidade: o objeto sempre vem, cada campo pode ser null.
export interface UnitAddress {
  street: string | null;
  number: string | null;
  complement: string | null;
  zip: string | null;
}

export interface ReferenceUnit {
  id: string;
  name: string;
  kind: string;
  address: UnitAddress;
}
```

Em `Person`, depois de `verified_at?`:

```ts
  // Bairro declarado (ADR 0023). Ausente numa api anterior: normalizado em people().
  neighborhood?: Neighborhood | null;
```

Em `TriageSummary`, depois de `check_in_available?`:

```ts
  // Unidades ativas que cobrem o bairro da triagem. Ausente numa api anterior:
  // normalizado para [] (ver normalizeTriage).
  reference_units?: ReferenceUnit[];
```

Em `normalizeTriage`, no objeto devolvido, depois de `check_in_available: ...`:

```ts
    check_in_available: t.check_in_available ?? false,
    reference_units: t.reference_units ?? []
```

(troque a linha `check_in_available: t.check_in_available ?? false` pelas duas acima.)

Depois de `normalizeAppointment`:

```ts
function normalizePerson(p: Person): Person {
  return { ...p, neighborhood: p.neighborhood ?? null };
}
```

No objeto `citizenApi`, troque `people` e `start`:

```ts
  people: async () => {
    const data = await call<{ people: Person[] }>("GET", "/people");
    return { people: data.people.map(normalizePerson) };
  },
  // Bairros ativos da cidade, por nome (spec §4.1).
  neighborhoods: async () => (await call<{ neighborhoods: Neighborhood[] }>("GET", "/neighborhoods")).neighborhoods,
  // null = "Prefiro não informar" (tira o bairro). Troca não muda triagens antigas.
  setNeighborhood: (citizenId: string, neighborhoodId: string | null) =>
    call<unknown>("POST", `/people/${encodeURIComponent(citizenId)}/neighborhood`, { neighborhood_id: neighborhoodId }),
  start: (p: { citizenId?: string; cpf?: string; neighborhoodId?: string; consentVersion: string }) =>
    call<StartResult>("POST", "/conversations", {
      ...(p.citizenId ? { citizen_id: p.citizenId } : { cpf: p.cpf }),
      ...(p.neighborhoodId ? { neighborhood_id: p.neighborhoodId } : {}),
      consent_version: p.consentVersion
    }),
```

- [ ] **Step 5: Implemente as regras puras**

```ts
// src/lib/territory.ts
// Regras de tela do território (spec 2026-09-28 §6; ADR 0023), sem React.
import type { Neighborhood, UnitAddress } from "./citizenApi";

const KIND_LABEL: Record<string, string> = {
  ubs: "UBS",
  upa: "UPA",
  hospital: "Hospital",
  other: "Unidade de saúde"
};

export function unitKindLabel(kind: string | null | undefined): string | null {
  return kind ? (KIND_LABEL[kind] ?? null) : null;
}

// "São Francisco" e "sao francisco" são a mesma busca.
function fold(s: string): string {
  return s.normalize("NFD").replace(/\p{Diacritic}/gu, "").toLowerCase().trim();
}

export function filterNeighborhoods(list: Neighborhood[], query: string): Neighborhood[] {
  const q = fold(query);
  if (q === "") return list;
  return list.filter(n => fold(n.name).includes(q));
}

function clean(s: string | null | undefined): string | null {
  const t = (s ?? "").trim();
  return t === "" ? null : t;
}

export function formatZip(zip: string | null | undefined): string | null {
  const d = (zip ?? "").replace(/\D/g, "");
  return d.length === 8 ? `${d.slice(0, 5)}-${d.slice(5)}` : null;
}

// "Rua X, 123 — Sala 2 · CEP 80000-000". Número e complemento sem logradouro
// não dizem nada sozinhos e somem; nada ausente vira texto.
export function formatAddress(a: UnitAddress | null | undefined): string | null {
  if (!a) return null;
  const street = clean(a.street);
  const number = clean(a.number);
  const complement = clean(a.complement);
  const zip = formatZip(a.zip);

  let line: string | null = null;
  if (street) {
    line = number ? `${street}, ${number}` : street;
    if (complement) line = `${line} — ${complement}`;
  }
  const parts = [ line, zip ? `CEP ${zip}` : null ].filter((p): p is string => p !== null);
  return parts.length > 0 ? parts.join(" · ") : null;
}
```

- [ ] **Step 6: Rode os testes e os tipos**

Run: `npx vitest run src/lib/territory.test.ts src/lib/citizenApi.test.ts && npx tsc --noEmit`
Expected: PASS, sem erro de tipo.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/lib/citizenApi.ts src/lib/citizenApi.test.ts src/lib/territory.ts src/lib/territory.test.ts
/opt/homebrew/bin/git commit -m "feat: add neighborhood and reference unit types to the citizen api client" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Bloco "Sua unidade de referência" (só na área logada)

**Files:**
- Create: `src/modules/ReferenceUnits.tsx`
- Modify: `src/modules/citizen/ResultStep.tsx`
- Test: `src/modules/ReferenceUnits.test.tsx` (novo), `src/modules/citizen/ResultStep.test.tsx`, `src/modules/Report.test.tsx`

**Interfaces:**
- Consumes (Task 1): `ReferenceUnit`, `TriageSummary.reference_units`, `formatAddress`, `unitKindLabel`.
- Produces: `ReferenceUnits({ units }: { units: ReferenceUnit[] | null | undefined })` — devolve `null` com lista vazia ou ausente.

**Comportamento:**
- título "Sua unidade de referência" com 1 unidade; "Suas unidades de referência" com 2 ou mais (todas listadas, na ordem da API, que já vem por nome);
- cada unidade: nome, rótulo do tipo (se conhecido) e a linha de endereço (se houver);
- rodapé: "Atende o bairro informado. Você pode procurar qualquer unidade de saúde." (ADR 0023: informa, nunca restringe);
- a `ResultStep` mostra o bloco **sempre** a partir de `reference_units` de `GET /citizen/triages/:id`: enquanto o relatório é preparado, quando desiste de esperar e também depois que o relatório fica pronto (abaixo do `Report`, antes dos botões);
- o `Report` (link público `/r/:token`) **não muda**: não lê nem mostra `reference_units`, mesmo que a API mande o campo (decisão do usuário, 2026-09-28). `src/lib/report.ts` e `src/modules/Report.tsx` não são tocados.

- [ ] **Step 1: Escreva os testes do componente**

```tsx
// src/modules/ReferenceUnits.test.tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { ReferenceUnits } from "./ReferenceUnits";
import type { ReferenceUnit } from "../lib/citizenApi";

const NO_ADDRESS = { street: null, number: null, complement: null, zip: null };
const ubs: ReferenceUnit = {
  id: "u1", name: "UBS Batel", kind: "ubs",
  address: { street: "Rua Padre Anchieta", number: "1500", complement: null, zip: "80730000" }
};
const upa: ReferenceUnit = { id: "u2", name: "UPA Matriz", kind: "upa", address: NO_ADDRESS };

describe("ReferenceUnits", () => {
  it.each([ [ "lista vazia", [] ], [ "null", null ], [ "undefined", undefined ] ])("%s: bloco ausente", (_label, units) => {
    const { container } = render(<ReferenceUnits units={units as ReferenceUnit[] | null | undefined} />);
    expect(container).toBeEmptyDOMElement();
  });

  it("uma unidade: título no singular, nome, tipo e endereço", () => {
    render(<ReferenceUnits units={[ ubs ]} />);
    expect(screen.getByRole("heading", { name: "Sua unidade de referência" })).toBeInTheDocument();
    expect(screen.getByText("UBS Batel")).toBeInTheDocument();
    expect(screen.getByText("UBS")).toBeInTheDocument();
    expect(screen.getByText("Rua Padre Anchieta, 1500 · CEP 80730-000")).toBeInTheDocument();
    expect(screen.getByText(/Você pode procurar qualquer unidade de saúde/)).toBeInTheDocument();
  });

  it("duas unidades: título no plural e as duas; a sem endereço fica sem linha de endereço", () => {
    const { container } = render(<ReferenceUnits units={[ ubs, upa ]} />);
    expect(screen.getByRole("heading", { name: "Suas unidades de referência" })).toBeInTheDocument();
    expect(screen.getAllByRole("listitem")).toHaveLength(2);
    expect(screen.getByText("UPA Matriz")).toBeInTheDocument();
    expect(screen.getByText("UPA")).toBeInTheDocument();
    expect(container.textContent).not.toMatch(/undefined|null|CEP\s*$/);
  });

  it("endereço parcial e tipo desconhecido: mostra só o que existe", () => {
    const { container } = render(<ReferenceUnits units={[
      { id: "u3", name: "Clínica da Família", kind: "clinica_nova",
        address: { street: null, number: "10", complement: null, zip: "80730000" } }
    ]} />);
    expect(screen.getByText("Clínica da Família")).toBeInTheDocument();
    expect(screen.getByText("CEP 80730-000")).toBeInTheDocument();
    expect(container.textContent).not.toMatch(/undefined|null|clinica_nova/);
  });
});
```

- [ ] **Step 2: Escreva os testes da `ResultStep` e do `Report` público**

Acrescente ao fim de `src/modules/citizen/ResultStep.test.tsx`:

```tsx
describe("ResultStep — unidade de referência", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date("2026-09-28T10:00:00-03:00"));
  });
  afterEach(() => vi.useRealTimers());

  const units = [
    { id: "u1", name: "UBS Batel", kind: "ubs",
      address: { street: "Rua Padre Anchieta", number: "1500", complement: null, zip: "80730000" } },
    { id: "u2", name: "UPA Matriz", kind: "upa", address: { street: null, number: null, complement: null, zip: null } }
  ];

  it("enquanto o relatório não fica pronto, mostra a unidade vinda da triagem", async () => {
    vi.spyOn(citizenApi, "triage").mockResolvedValue({ ...summary(null), reference_units: [ units[0] ] });
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} />);
    expect(await screen.findByRole("heading", { name: "Sua unidade de referência" })).toBeInTheDocument();
    expect(screen.getByText("UBS Batel")).toBeInTheDocument();
  });

  it("com o relatório pronto, continua mostrando as unidades da triagem (duas), uma vez só", async () => {
    vi.spyOn(citizenApi, "triage").mockResolvedValue({
      ...summary("http://curitiba.localhost/wpda/?token=abc"), reference_units: units
    });
    stubReport();
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} />);

    expect(await screen.findByText("Procure atendimento hoje")).toBeInTheDocument();
    expect(screen.getAllByRole("heading", { name: "Suas unidades de referência" })).toHaveLength(1);
    expect(screen.getByText("UBS Batel")).toBeInTheDocument();
    expect(screen.getByText("UPA Matriz")).toBeInTheDocument();
  });

  it("triagem sem unidade de referência: bloco ausente", async () => {
    const triage = vi.spyOn(citizenApi, "triage").mockResolvedValue({ ...summary(null), reference_units: [] });
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} />);
    await waitFor(() => expect(triage).toHaveBeenCalled());
    expect(screen.queryByText(/unidade de referência|unidades de referência/)).not.toBeInTheDocument();
  });
});
```

(troque os imports do topo por `import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";` e `import { render, screen, waitFor } from "@testing-library/react";`.)

Acrescente ao fim do `describe("Report", ...)` em `src/modules/Report.test.tsx` (guarda da decisão: o link público nunca mostra a unidade de referência):

```tsx
  it("link público não mostra a unidade de referência, mesmo que a API mande reference_units", async () => {
    stubFetch(json(200, { ...frozen, reference_units: [
      { id: "u1", name: "UBS Batel", kind: "ubs",
        address: { street: "Rua Padre Anchieta", number: "1500", complement: null, zip: "80730000" } }
    ] }));
    render(<Report token="abc" />);

    await screen.findByText("Seu resultado");
    expect(screen.queryByText(/unidade de referência|unidades de referência/)).not.toBeInTheDocument();
    expect(screen.queryByText("UBS Batel")).not.toBeInTheDocument();
    expect(screen.queryByText(/Rua Padre Anchieta/)).not.toBeInTheDocument();
  });
```

(As datas de `frozen` são literais formatados, não dependem de "agora"; o `describe` do `Report` já não fixa o relógio e o teste novo também não precisa.)

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/modules/ReferenceUnits.test.tsx src/modules/citizen/ResultStep.test.tsx src/modules/Report.test.tsx`
Expected: FAIL em `ReferenceUnits` (arquivo não existe) e nos casos novos da `ResultStep`. O teste novo do `Report` já passa (é guarda de regressão: prova que ninguém ligou o bloco no link público).

- [ ] **Step 4: Implemente o componente**

```tsx
// src/modules/ReferenceUnits.tsx
// Bloco "Sua unidade de referência" (spec 2026-09-28 §6; ADR 0023): as
// unidades ativas que atendem o bairro da triagem. Informa, nunca restringe.
// Só na área logada: o relatório público (/r/:token) não o mostra.
// Lista vazia (sem bairro, bairro sem cobertura ou api antiga) = sem bloco.
import type { ReferenceUnit } from "../lib/citizenApi";
import { formatAddress, unitKindLabel } from "../lib/territory";

export function ReferenceUnits({ units }: { units: ReferenceUnit[] | null | undefined }) {
  if (!units || units.length === 0) return null;
  const title = units.length === 1 ? "Sua unidade de referência" : "Suas unidades de referência";
  return (
    <section aria-label={title}
      style={{ margin: "0 0 24px", padding: 12, borderRadius: 12, border: "1px solid var(--line, #eee)" }}>
      <h2 style={{ fontSize: 16, margin: "0 0 8px" }}>{title}</h2>
      <ul style={{ listStyle: "none", margin: 0, padding: 0, display: "grid", gap: 12 }}>
        {units.map(u => {
          const kind = unitKindLabel(u.kind);
          const address = formatAddress(u.address);
          return (
            <li key={u.id}>
              <strong>{u.name}</strong>
              {kind && <span style={{ marginLeft: 8, fontSize: 13, color: "var(--ink2, #555)" }}>{kind}</span>}
              {address && <div style={{ fontSize: 14, color: "var(--ink2, #555)" }}>{address}</div>}
            </li>
          );
        })}
      </ul>
      <p style={{ fontSize: 13, color: "var(--ink3, #888)", margin: "8px 0 0" }}>
        Atende o bairro informado. Você pode procurar qualquer unidade de saúde.
      </p>
    </section>
  );
}
```

- [ ] **Step 5: Use o bloco na `ResultStep`**

Em `src/modules/citizen/ResultStep.tsx`:
- troque o import de `citizenApi` por `import { citizenApi, type ReferenceUnit } from "../../lib/citizenApi";` e importe `import { ReferenceUnits } from "../ReferenceUnits";`;
- depois de `const [gaveUp, setGaveUp] = useState(false);`: `const [units, setUnits] = useState<ReferenceUnit[]>([]);`
- dentro de `poll`, logo depois de `const t = await citizenApi.triage(triageId);`: `if (alive) setUnits(t.reference_units ?? []);` (roda em toda consulta, inclusive na que acha o link, então o bloco já está preenchido quando o relatório aparece);
- troque a constante `actions` e os dois `return` por:

```tsx
  const actions = (
    <div style={{ display: "grid", gap: 12, padding: 16, maxWidth: 520, margin: "0 auto" }}>
      <BigButton onClick={onAgain}>Fazer outra triagem</BigButton>
      <BigButton variant="secondary" onClick={onHistory}>Minhas triagens</BigButton>
    </div>
  );

  // Área logada: a unidade de referência vem sempre de GET /citizen/triages/:id.
  // O Report (link público) não a mostra.
  if (token) {
    return <>
      <Report token={token} />
      <div style={{ padding: "0 16px", maxWidth: 520, margin: "0 auto" }}><ReferenceUnits units={units} /></div>
      {actions}
    </>;
  }
  return (
    <Screen title="Triagem concluída" footer={actions}>
      <p>{gaveUp ? "Seu resultado ainda está sendo preparado. Veja em \"Minhas triagens\" daqui a pouco." : "Preparando seu resultado…"}</p>
      <ReferenceUnits units={units} />
    </Screen>
  );
```

- [ ] **Step 6: Rode os testes e os tipos**

Run: `npx vitest run src/modules/ReferenceUnits.test.tsx src/modules/citizen/ResultStep.test.tsx src/modules/Report.test.tsx && npx tsc --noEmit`
Expected: PASS; os testes antigos de `Report` e `ResultStep` continuam verdes.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/modules/ReferenceUnits.tsx src/modules/ReferenceUnits.test.tsx src/modules/citizen/ResultStep.tsx src/modules/citizen/ResultStep.test.tsx src/modules/Report.test.tsx
/opt/homebrew/bin/git commit -m "feat: show the reference health unit in the signed-in triage result" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Bairro na escolha da pessoa

**Files:**
- Create: `src/modules/citizen/NeighborhoodPicker.tsx`
- Modify: `src/modules/citizen/PeopleStep.tsx`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/ui.tsx`
- Test: `src/modules/citizen/PeopleStep.test.tsx` (novo), `src/modules/citizen/entry.test.tsx`, `src/modules/citizen/Flow.test.tsx`

**Interfaces:**
- Consumes (Task 1): `Neighborhood`, `Person.neighborhood`, `citizenApi.neighborhoods()`, `citizenApi.start({ neighborhoodId })`, `filterNeighborhoods`.
- Produces:
  - `NeighborhoodPicker({ title, neighborhoods, notice?, busy?, onPick(id: string | null), onBack() })`;
  - `type Who = { citizenId: string } | { cpf: string }`, `type PersonChoice = Who & { neighborhoodId?: string }`, `type ChooseOutcome = "done" | "invalid_neighborhood"` (exportados de `PeopleStep.tsx`);
  - `PeopleStep({ onChoose: (c: PersonChoice) => Promise<ChooseOutcome>; onHistory })`.

**Comportamento (spec §6):**
1. A `PeopleStep` busca a lista de bairros ao montar. Lista vazia ou com erro = a cidade não tem bairros: nenhuma pergunta e nenhum "Trocar bairro".
2. **CPF novo:** CPF válido + "Continuar com este CPF" → tela "Em que bairro esta pessoa mora?" com busca, um botão por bairro e "Prefiro não informar". Tocar num bairro chama `onChoose({ cpf, neighborhoodId })`; "Prefiro não informar" chama `onChoose({ cpf })`. "Voltar" volta à lista de pessoas.
3. **Pessoa existente sem bairro:** tocar no CPF abre a mesma pergunta, uma vez, antes de começar. **Com bairro:** começa direto, sem mandar bairro (a troca é pela rota própria, Task 4).
4. Cada pessoa mostra "Bairro: X" ou "Bairro não informado" (quando a cidade tem bairros).
5. **422 `invalid_neighborhood`:** `Flow.choose` devolve `"invalid_neighborhood"` sem mostrar erro. A `PeopleStep` relê a lista e reabre a pergunta com o aviso "Esse bairro não está mais na lista. Escolha de novo.". Se a lista voltar vazia, começa sem bairro.
6. Enquanto o envio não volta, os botões da pergunta ficam desabilitados (toque duplo não abre duas conversas).

- [ ] **Step 1: Escreva os testes da `PeopleStep`**

```tsx
// src/modules/citizen/PeopleStep.test.tsx
import { describe, it, expect, vi, afterEach } from "vitest";
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { PeopleStep, type ChooseOutcome } from "./PeopleStep";
import { citizenApi, type Person } from "../../lib/citizenApi";

afterEach(() => vi.restoreAllMocks());

const LIST = [
  { id: "n1", name: "Batel" },
  { id: "n2", name: "São Francisco" },
  { id: "n3", name: "Santa Felicidade" }
];
const semBairro: Person = { id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared", neighborhood: null };
const comBairro: Person = { ...semBairro, neighborhood: { id: "n1", name: "Batel" } };

function setup(people: Person[], list: typeof LIST | Error = LIST) {
  vi.spyOn(citizenApi, "people").mockResolvedValue({ people });
  const neighborhoods = list instanceof Error
    ? vi.spyOn(citizenApi, "neighborhoods").mockRejectedValue(list)
    : vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue(list);
  const onChoose = vi.fn<(c: unknown) => Promise<ChooseOutcome>>().mockResolvedValue("done");
  render(<PeopleStep onChoose={onChoose} onHistory={vi.fn()} />);
  return { onChoose, neighborhoods };
}

async function typeNewCpf() {
  await userEvent.type(await screen.findByLabelText("CPF de outra pessoa"), "52998224725");
  await userEvent.click(screen.getByRole("button", { name: "Continuar com este CPF" }));
}

describe("PeopleStep — bairro antes de começar", () => {
  it("CPF novo: pergunta o bairro, busca sem acento e manda o escolhido junto com o CPF", async () => {
    const { onChoose } = setup([]);
    await typeNewCpf();

    expect(await screen.findByRole("heading", { name: "Em que bairro esta pessoa mora?" })).toBeInTheDocument();
    expect(onChoose).not.toHaveBeenCalled();

    await userEvent.type(screen.getByLabelText("Buscar bairro"), "sao");
    expect(screen.queryByRole("button", { name: "Batel" })).not.toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "São Francisco" }));

    await waitFor(() => expect(onChoose).toHaveBeenCalledWith({ cpf: "529.982.247-25", neighborhoodId: "n2" }));
  });

  it("CPF novo: 'Prefiro não informar' começa sem bairro", async () => {
    const { onChoose } = setup([]);
    await typeNewCpf();
    await userEvent.click(await screen.findByRole("button", { name: "Prefiro não informar" }));

    await waitFor(() => expect(onChoose).toHaveBeenCalledTimes(1));
    expect(onChoose.mock.calls[0][0]).toEqual({ cpf: "529.982.247-25" });
    expect("neighborhoodId" in (onChoose.mock.calls[0][0] as object)).toBe(false);
  });

  it("busca sem resultado avisa", async () => {
    setup([]);
    await typeNewCpf();
    await userEvent.type(await screen.findByLabelText("Buscar bairro"), "xyz");
    expect(screen.getByText("Nenhum bairro encontrado com esse nome.")).toBeInTheDocument();
  });

  it("'Voltar' na pergunta volta para a lista de pessoas", async () => {
    const { onChoose } = setup([]);
    await typeNewCpf();
    await userEvent.click(await screen.findByRole("button", { name: "Voltar" }));
    expect(await screen.findByText("Para quem é esta triagem?")).toBeInTheDocument();
    expect(onChoose).not.toHaveBeenCalled();
  });

  it("pessoa existente sem bairro: pergunta uma vez antes de começar", async () => {
    const { onChoose } = setup([ semBairro ]);
    expect(await screen.findByText("Bairro não informado")).toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "CPF ***.982.247-**" }));
    await userEvent.click(await screen.findByRole("button", { name: "Batel" }));

    await waitFor(() => expect(onChoose).toHaveBeenCalledWith({ citizenId: "p1", neighborhoodId: "n1" }));
    expect(onChoose).toHaveBeenCalledTimes(1);
  });

  it("pessoa existente com bairro: começa direto, sem mandar bairro", async () => {
    const { onChoose } = setup([ comBairro ]);
    expect(await screen.findByText("Bairro: Batel")).toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "CPF ***.982.247-**" }));

    await waitFor(() => expect(onChoose).toHaveBeenCalledWith({ citizenId: "p1" }));
    expect(screen.queryByRole("heading", { name: "Em que bairro esta pessoa mora?" })).not.toBeInTheDocument();
  });

  it("cidade sem bairros: não pergunta nem oferece troca", async () => {
    const { onChoose } = setup([ semBairro ], []);
    await userEvent.click(await screen.findByRole("button", { name: "CPF ***.982.247-**" }));
    await waitFor(() => expect(onChoose).toHaveBeenCalledWith({ citizenId: "p1" }));
    expect(screen.queryByText("Bairro não informado")).not.toBeInTheDocument();
    expect(screen.queryByRole("button", { name: /Trocar bairro/ })).not.toBeInTheDocument();
  });

  it("lista com erro (api antiga ou rede): segue sem perguntar", async () => {
    const { onChoose } = setup([], new Error("offline"));
    await typeNewCpf();
    await waitFor(() => expect(onChoose).toHaveBeenCalledWith({ cpf: "529.982.247-25" }));
  });

  it("422 invalid_neighborhood: relê a lista, avisa e pede nova escolha", async () => {
    const { onChoose, neighborhoods } = setup([ semBairro ]);
    neighborhoods.mockResolvedValueOnce(LIST).mockResolvedValueOnce(LIST.filter(n => n.id !== "n1"));
    onChoose.mockResolvedValueOnce("invalid_neighborhood").mockResolvedValue("done");

    await userEvent.click(await screen.findByRole("button", { name: "CPF ***.982.247-**" }));
    await userEvent.click(await screen.findByRole("button", { name: "Batel" }));

    expect(await screen.findByText("Esse bairro não está mais na lista. Escolha de novo.")).toBeInTheDocument();
    expect(neighborhoods).toHaveBeenCalledTimes(2);
    expect(screen.queryByRole("button", { name: "Batel" })).not.toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "São Francisco" }));
    await waitFor(() => expect(onChoose).toHaveBeenLastCalledWith({ citizenId: "p1", neighborhoodId: "n2" }));
  });

  it("422 com a lista agora vazia: começa sem bairro", async () => {
    const { onChoose, neighborhoods } = setup([ semBairro ]);
    neighborhoods.mockResolvedValueOnce(LIST).mockResolvedValueOnce([]);
    onChoose.mockResolvedValueOnce("invalid_neighborhood").mockResolvedValue("done");

    await userEvent.click(await screen.findByRole("button", { name: "CPF ***.982.247-**" }));
    await userEvent.click(await screen.findByRole("button", { name: "Batel" }));

    await waitFor(() => expect(onChoose).toHaveBeenLastCalledWith({ citizenId: "p1" }));
    expect(onChoose).toHaveBeenCalledTimes(2);
  });

  it("toque duplo no bairro chama uma vez só", async () => {
    const { onChoose } = setup([ semBairro ]);
    onChoose.mockReturnValue(new Promise<ChooseOutcome>(() => {}));
    await userEvent.click(await screen.findByRole("button", { name: "CPF ***.982.247-**" }));
    const batel = await screen.findByRole("button", { name: "Batel" });
    await userEvent.click(batel);
    await userEvent.click(batel);

    expect(onChoose).toHaveBeenCalledTimes(1);
    expect(batel).toBeDisabled();
  });
});
```

- [ ] **Step 2: Ajuste os testes antigos que montam a `PeopleStep`**

`src/modules/citizen/entry.test.tsx`, no teste `"escolhe uma pessoa da lista ou valida o CPF novo"` (a escolha passou a ser assíncrona, porque espera a lista de bairros):
- logo depois do `vi.spyOn(citizenApi, "people")...`, acrescente `vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue([]);`;
- troque `const onChoose = vi.fn();` por `const onChoose = vi.fn().mockResolvedValue("done");`;
- troque `expect(onChoose).toHaveBeenCalledWith({ citizenId: "p1" });` por `await waitFor(() => expect(onChoose).toHaveBeenCalledWith({ citizenId: "p1" }));`;
- acrescente `waitFor` ao import de `@testing-library/react`.

`src/modules/citizen/Flow.test.tsx`: em `reachQuestionStep` e no teste `"issue do check-in é memoizado..."`, depois do `vi.spyOn(citizenApi, "people")...`, acrescente `vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue([]);` (sem bairros na cidade, o fluxo antigo não muda).

Acrescente ao `describe("Flow", ...)`:

```tsx
  it("422 invalid_neighborhood ao começar: sem erro genérico, pede o bairro de novo e segue", async () => {
    vi.spyOn(citizenApi, "currentSession").mockResolvedValue({ phone_masked: "(**) *****-5432" });
    vi.spyOn(citizenApi, "consentTerm").mockResolvedValue({ version: "1", body: "Termo" });
    vi.spyOn(citizenApi, "people").mockResolvedValue({
      people: [{ id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared", neighborhood: null }]
    });
    vi.spyOn(citizenApi, "neighborhoods")
      .mockResolvedValueOnce([ { id: "n1", name: "Batel" }, { id: "n2", name: "São Francisco" } ])
      .mockResolvedValueOnce([ { id: "n2", name: "São Francisco" } ]);
    const start = vi.spyOn(citizenApi, "start")
      .mockRejectedValueOnce(new ApiError(422, "invalid_neighborhood"))
      .mockResolvedValue({ conversation_id: "c1", citizen_id: "p1", resumed: false, step: boolStep });

    render(<Flow />);
    await userEvent.click(await screen.findByRole("button", { name: "Concordo" }));
    await userEvent.click(await screen.findByRole("button", { name: "CPF ***.982.247-**" }));
    await userEvent.click(await screen.findByRole("button", { name: "Batel" }));

    expect(await screen.findByText("Esse bairro não está mais na lista. Escolha de novo.")).toBeInTheDocument();
    expect(screen.queryByText("Algo deu errado. Tente de novo.")).not.toBeInTheDocument();
    expect(start).toHaveBeenNthCalledWith(1, { citizenId: "p1", neighborhoodId: "n1", consentVersion: "1" });

    await userEvent.click(screen.getByRole("button", { name: "São Francisco" }));
    expect(await screen.findByText("Você está com tosse?")).toBeInTheDocument();
    expect(start).toHaveBeenNthCalledWith(2, { citizenId: "p1", neighborhoodId: "n2", consentVersion: "1" });
  });
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/PeopleStep.test.tsx src/modules/citizen/entry.test.tsx src/modules/citizen/Flow.test.tsx`
Expected: FAIL (a pergunta não existe; `ChooseOutcome` não é exportado; o 422 mostra o erro genérico).

- [ ] **Step 4: Implemente o `NeighborhoodPicker`**

```tsx
// src/modules/citizen/NeighborhoodPicker.tsx
// Escolha do bairro (spec 2026-09-28 §6; ADR 0023): bairros ativos da cidade
// com busca, e "Prefiro não informar". Tocar num bairro já escolhe.
import { useState } from "react";
import type { Neighborhood } from "../../lib/citizenApi";
import { filterNeighborhoods } from "../../lib/territory";
import { BigButton, ErrorText, Field, Screen } from "./ui";

export function NeighborhoodPicker({ title, neighborhoods, notice, busy, onPick, onBack }:
  { title: string; neighborhoods: Neighborhood[]; notice?: string | null; busy?: boolean;
    onPick: (neighborhoodId: string | null) => void; onBack: () => void }) {
  const [query, setQuery] = useState("");
  const shown = filterNeighborhoods(neighborhoods, query);

  return (
    <Screen title={title} footer={<>
      <BigButton variant="secondary" disabled={busy} onClick={() => onPick(null)}>Prefiro não informar</BigButton>
      <BigButton variant="secondary" disabled={busy} onClick={onBack}>Voltar</BigButton>
    </>}>
      <p>Usamos o bairro para indicar a unidade de saúde que atende você. Você pode trocar depois.</p>
      {notice && <ErrorText>{notice}</ErrorText>}
      <Field label="Buscar bairro" value={query} autoComplete="off" onChange={e => setQuery(e.target.value)} />
      {shown.length === 0 && <p>Nenhum bairro encontrado com esse nome.</p>}
      <div style={{ display: "grid", gap: 8 }}>
        {shown.map(n => (
          <BigButton key={n.id} variant="secondary" disabled={busy} onClick={() => onPick(n.id)}>{n.name}</BigButton>
        ))}
      </div>
    </Screen>
  );
}
```

- [ ] **Step 5: Reescreva a `PeopleStep`**

```tsx
// src/modules/citizen/PeopleStep.tsx
import { useEffect, useRef, useState } from "react";
import { citizenApi, type Neighborhood, type Person } from "../../lib/citizenApi";
import { isValidCpf, maskCpf } from "../../lib/masks";
import { NeighborhoodPicker } from "./NeighborhoodPicker";
import { BigButton, ErrorText, Field, Screen, messageFor } from "./ui";

export type Who = { citizenId: string } | { cpf: string };
export type PersonChoice = Who & { neighborhoodId?: string };
// "invalid_neighborhood": o bairro saiu da lista entre a leitura e o envio (422);
// "done": o Flow cuidou do resto (pergunta, termo ou erro).
export type ChooseOutcome = "done" | "invalid_neighborhood";

type Mode =
  | { at: "list" }
  | { at: "pick-for-start"; who: Who; notice: string | null };

export const STALE_NEIGHBORHOOD = "Esse bairro não está mais na lista. Escolha de novo.";

const linkStyle = { minHeight: 48, background: "none", border: "none", textDecoration: "underline", fontSize: 18 } as const;

export function PeopleStep({ onChoose, onHistory }:
  { onChoose: (c: PersonChoice) => Promise<ChooseOutcome>; onHistory: (citizenId: string) => void }) {
  const [people, setPeople] = useState<Person[] | null>(null);
  const [neighborhoods, setNeighborhoods] = useState<Neighborhood[]>([]);
  const listRef = useRef<Promise<Neighborhood[]> | null>(null);
  const [mode, setMode] = useState<Mode>({ at: "list" });
  const [cpf, setCpf] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [busy, setBusy] = useState(false);

  function loadPeople() {
    citizenApi.people().then(r => setPeople(r.people)).catch(e => setError(messageFor(e)));
  }

  // Lista vazia ou com erro = a cidade não tem bairros (sem semente, spec §9):
  // não pergunta, e a triagem segue sem bairro.
  function reloadNeighborhoods(): Promise<Neighborhood[]> {
    const p = citizenApi.neighborhoods().catch(() => [] as Neighborhood[]);
    listRef.current = p;
    p.then(setNeighborhoods);
    return p;
  }

  function currentNeighborhoods(): Promise<Neighborhood[]> {
    return listRef.current ?? reloadNeighborhoods();
  }

  useEffect(() => {
    loadPeople();
    void reloadNeighborhoods();
  }, []);

  async function begin(who: Who, neighborhoodId: string | null) {
    setBusy(true);
    try {
      const outcome = await onChoose(neighborhoodId ? { ...who, neighborhoodId } : who);
      if (outcome !== "invalid_neighborhood") return;
      const list = await reloadNeighborhoods();
      if (list.length === 0) {
        setMode({ at: "list" });
        await onChoose(who);
        return;
      }
      setMode({ at: "pick-for-start", who, notice: STALE_NEIGHBORHOOD });
    } finally {
      setBusy(false);
    }
  }

  // Pergunta o bairro uma vez, antes de começar, a quem ainda não tem; quem
  // já tem começa direto (a troca é pelo "Trocar bairro").
  async function ask(who: Who, hasNeighborhood: boolean) {
    const list = await currentNeighborhoods();
    if (hasNeighborhood || list.length === 0) return begin(who, null);
    setMode({ at: "pick-for-start", who, notice: null });
  }

  function submitNew() {
    if (!isValidCpf(cpf)) return setError("CPF inválido. Confira os números.");
    setError(null);
    void ask({ cpf }, false);
  }

  if (mode.at === "pick-for-start") {
    const who = mode.who;
    return <NeighborhoodPicker title="Em que bairro esta pessoa mora?" neighborhoods={neighborhoods}
      notice={mode.notice} busy={busy}
      onPick={id => void begin(who, id)} onBack={() => setMode({ at: "list" })} />;
  }

  const cityHasNeighborhoods = neighborhoods.length > 0;
  return (
    <Screen title="Para quem é esta triagem?">
      {people === null && !error && <p>Carregando…</p>}
      <div style={{ display: "grid", gap: 12, marginBottom: 24 }}>
        {people?.map(p => (
          <div key={p.id} style={{ display: "grid", gap: 8 }}>
            <BigButton variant="secondary" disabled={busy} onClick={() => void ask({ citizenId: p.id }, Boolean(p.neighborhood))}>
              CPF {p.cpf_masked}
            </BigButton>
            {(cityHasNeighborhoods || p.neighborhood) &&
              <p style={{ margin: 0 }}>{p.neighborhood ? `Bairro: ${p.neighborhood.name}` : "Bairro não informado"}</p>}
            <button type="button" onClick={() => onHistory(p.id)} style={linkStyle}>
              Ver triagens de {p.cpf_masked}
            </button>
          </div>
        ))}
      </div>
      <Field label="CPF de outra pessoa" inputMode="numeric" value={cpf}
        onChange={e => setCpf(maskCpf(e.target.value))} />
      {error && <ErrorText>{error}</ErrorText>}
      <BigButton disabled={busy} onClick={submitNew}>Continuar com este CPF</BigButton>
    </Screen>
  );
}
```

- [ ] **Step 6: Ajuste o `Flow` e a mensagem**

`src/modules/citizen/Flow.tsx`:
- troque o import por `import { PeopleStep, type ChooseOutcome, type PersonChoice } from "./PeopleStep";`;
- troque a função `choose` inteira por:

```tsx
  async function choose(consentVersion: string, choice: PersonChoice): Promise<ChooseOutcome> {
    setError(null);
    try {
      const r = await citizenApi.start({ ...choice, consentVersion });
      setState({ at: "question", consentVersion, conversationId: r.conversation_id, citizenId: r.citizen_id, step: r.step });
    } catch (e) {
      if (e instanceof ApiError && (e.code === "consent_outdated" || e.code === "no_consent")) {
        setState({ at: "consent" });
        return "done";
      }
      // O bairro saiu da lista entre a leitura e o envio: a PeopleStep relê e
      // pergunta de novo (spec §6), sem o erro genérico.
      if (e instanceof ApiError && e.code === "invalid_neighborhood") return "invalid_neighborhood";
      setError(messageFor(e));
    }
    return "done";
  }
```

(O uso `onChoose={c => choose(state.consentVersion, c)}` não muda.)

`src/modules/citizen/ui.tsx`, em `MESSAGES`, depois de `appointment_not_eligible`:

```ts
  appointment_not_eligible: "Este agendamento não está disponível para check-in.",
  invalid_neighborhood: "Esse bairro não está mais na lista. Escolha de novo."
```

- [ ] **Step 7: Rode a suíte inteira e os tipos**

Run: `npm test && npx tsc --noEmit`
Expected: PASS; a contagem é a anotada no começo mais os testes novos.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/NeighborhoodPicker.tsx src/modules/citizen/PeopleStep.tsx src/modules/citizen/PeopleStep.test.tsx src/modules/citizen/Flow.tsx src/modules/citizen/Flow.test.tsx src/modules/citizen/ui.tsx src/modules/citizen/entry.test.tsx
/opt/homebrew/bin/git commit -m "feat: ask the citizen's neighborhood before the first triage" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: "Trocar bairro"

**Files:**
- Modify: `src/modules/citizen/PeopleStep.tsx`
- Test: `src/modules/citizen/PeopleStep.test.tsx`

**Interfaces:**
- Consumes (Task 1): `citizenApi.setNeighborhood(citizenId, neighborhoodId | null)`. (Task 3): `NeighborhoodPicker`, `Mode`, `reloadNeighborhoods`, `loadPeople`, `STALE_NEIGHBORHOOD`, `linkStyle`.
- Produces: botão "Trocar bairro de *CPF mascarado*" ao lado de cada pessoa, só quando a cidade tem bairros.

**Comportamento (spec §4.1, §6):**
1. "Trocar bairro de ***.982.247-**" abre a pergunta com o título "Bairro de ***.982.247-**".
2. Tocar num bairro manda `POST /citizen/people/:id/neighborhood { neighborhood_id }`; "Prefiro não informar" manda `null` (tira o bairro).
3. Sucesso: volta à lista, relê as pessoas e mostra "Bairro atualizado.".
4. 422 `invalid_neighborhood`: relê a lista e mostra o aviso de bairro fora da lista, na mesma pergunta.
5. Outro erro (404 de CPF fora da sessão, rede): mostra a mensagem de `messageFor` na pergunta.
6. Trocar não muda triagens antigas (regra do api; a tela não promete nada sobre elas).

- [ ] **Step 1: Escreva os testes**

Acrescente a `src/modules/citizen/PeopleStep.test.tsx` (troque o import de `citizenApi` por `import { ApiError, citizenApi, type Person } from "../../lib/citizenApi";`):

```tsx
describe("PeopleStep — Trocar bairro", () => {
  it("troca o bairro, relê as pessoas e confirma", async () => {
    const people = vi.spyOn(citizenApi, "people")
      .mockResolvedValueOnce({ people: [ comBairro ] })
      .mockResolvedValue({ people: [ { ...comBairro, neighborhood: { id: "n2", name: "São Francisco" } } ] });
    vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue(LIST);
    const set = vi.spyOn(citizenApi, "setNeighborhood").mockResolvedValue({});
    const onChoose = vi.fn().mockResolvedValue("done");
    render(<PeopleStep onChoose={onChoose} onHistory={vi.fn()} />);

    await userEvent.click(await screen.findByRole("button", { name: "Trocar bairro de ***.982.247-**" }));
    expect(await screen.findByRole("heading", { name: "Bairro de ***.982.247-**" })).toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "São Francisco" }));

    expect(await screen.findByText("Bairro atualizado.")).toBeInTheDocument();
    expect(await screen.findByText("Bairro: São Francisco")).toBeInTheDocument();
    expect(set).toHaveBeenCalledWith("p1", "n2");
    expect(people).toHaveBeenCalledTimes(2);
    expect(onChoose).not.toHaveBeenCalled();
  });

  it("'Prefiro não informar' na troca tira o bairro (manda null)", async () => {
    vi.spyOn(citizenApi, "people")
      .mockResolvedValueOnce({ people: [ comBairro ] })
      .mockResolvedValue({ people: [ semBairro ] });
    vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue(LIST);
    const set = vi.spyOn(citizenApi, "setNeighborhood").mockResolvedValue({});
    render(<PeopleStep onChoose={vi.fn().mockResolvedValue("done")} onHistory={vi.fn()} />);

    await userEvent.click(await screen.findByRole("button", { name: "Trocar bairro de ***.982.247-**" }));
    await userEvent.click(await screen.findByRole("button", { name: "Prefiro não informar" }));

    expect(await screen.findByText("Bairro não informado")).toBeInTheDocument();
    expect(set).toHaveBeenCalledWith("p1", null);
  });

  it("422 invalid_neighborhood na troca: relê a lista e avisa, sem sair da pergunta", async () => {
    vi.spyOn(citizenApi, "people").mockResolvedValue({ people: [ comBairro ] });
    const neighborhoods = vi.spyOn(citizenApi, "neighborhoods")
      .mockResolvedValueOnce(LIST).mockResolvedValue(LIST.filter(n => n.id !== "n2"));
    vi.spyOn(citizenApi, "setNeighborhood").mockRejectedValue(new ApiError(422, "invalid_neighborhood"));
    render(<PeopleStep onChoose={vi.fn().mockResolvedValue("done")} onHistory={vi.fn()} />);

    await userEvent.click(await screen.findByRole("button", { name: "Trocar bairro de ***.982.247-**" }));
    await userEvent.click(await screen.findByRole("button", { name: "São Francisco" }));

    expect(await screen.findByText("Esse bairro não está mais na lista. Escolha de novo.")).toBeInTheDocument();
    expect(screen.getByRole("heading", { name: "Bairro de ***.982.247-**" })).toBeInTheDocument();
    expect(neighborhoods).toHaveBeenCalledTimes(2);
    await waitFor(() => expect(screen.queryByRole("button", { name: "São Francisco" })).not.toBeInTheDocument());
  });

  it("404 (CPF fora da sessão): mostra a mensagem, sem sair da pergunta", async () => {
    vi.spyOn(citizenApi, "people").mockResolvedValue({ people: [ comBairro ] });
    vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue(LIST);
    vi.spyOn(citizenApi, "setNeighborhood").mockRejectedValue(new ApiError(404, "not_found"));
    render(<PeopleStep onChoose={vi.fn().mockResolvedValue("done")} onHistory={vi.fn()} />);

    await userEvent.click(await screen.findByRole("button", { name: "Trocar bairro de ***.982.247-**" }));
    await userEvent.click(await screen.findByRole("button", { name: "Batel" }));

    expect(await screen.findByText("Algo deu errado. Tente de novo.")).toBeInTheDocument();
    expect(screen.getByRole("heading", { name: "Bairro de ***.982.247-**" })).toBeInTheDocument();
  });

  it("'Voltar' na troca não grava nada", async () => {
    vi.spyOn(citizenApi, "people").mockResolvedValue({ people: [ comBairro ] });
    vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue(LIST);
    const set = vi.spyOn(citizenApi, "setNeighborhood");
    render(<PeopleStep onChoose={vi.fn().mockResolvedValue("done")} onHistory={vi.fn()} />);

    await userEvent.click(await screen.findByRole("button", { name: "Trocar bairro de ***.982.247-**" }));
    await userEvent.click(await screen.findByRole("button", { name: "Voltar" }));

    expect(await screen.findByText("Para quem é esta triagem?")).toBeInTheDocument();
    expect(set).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/PeopleStep.test.tsx`
Expected: FAIL (não há botão "Trocar bairro de …").

- [ ] **Step 3: Implemente**

Em `src/modules/citizen/PeopleStep.tsx`:

- troque o import da API por `import { ApiError, citizenApi, type Neighborhood, type Person } from "../../lib/citizenApi";`;
- acrescente o modo novo a `Mode`:

```tsx
type Mode =
  | { at: "list" }
  | { at: "pick-for-start"; who: Who; notice: string | null }
  | { at: "pick-for-change"; person: Person; notice: string | null };
```

- depois de `const [busy, setBusy] = useState(false);`: `const [saved, setSaved] = useState<string | null>(null);`
- depois de `submitNew`, acrescente:

```tsx
  // "Trocar bairro" (spec §4.1): null tira o bairro. Não muda triagens antigas.
  async function change(person: Person, neighborhoodId: string | null) {
    setBusy(true);
    try {
      await citizenApi.setNeighborhood(person.id, neighborhoodId);
      setMode({ at: "list" });
      setSaved("Bairro atualizado.");
      loadPeople();
    } catch (e) {
      if (e instanceof ApiError && e.code === "invalid_neighborhood") {
        await reloadNeighborhoods();
        setMode({ at: "pick-for-change", person, notice: STALE_NEIGHBORHOOD });
      } else {
        setMode({ at: "pick-for-change", person, notice: messageFor(e) });
      }
    } finally {
      setBusy(false);
    }
  }
```

- depois do `if (mode.at === "pick-for-start") { ... }`, acrescente:

```tsx
  if (mode.at === "pick-for-change") {
    const person = mode.person;
    return <NeighborhoodPicker title={`Bairro de ${person.cpf_masked}`} neighborhoods={neighborhoods}
      notice={mode.notice} busy={busy}
      onPick={id => void change(person, id)} onBack={() => setMode({ at: "list" })} />;
  }
```

- na lista, logo depois de `{people === null && !error && <p>Carregando…</p>}`: `{saved && <p role="status">{saved}</p>}`
- em cada pessoa, entre a linha do bairro e "Ver triagens de …":

```tsx
            {cityHasNeighborhoods &&
              <button type="button" disabled={busy} style={linkStyle}
                onClick={() => { setSaved(null); setMode({ at: "pick-for-change", person: p, notice: null }); }}>
                Trocar bairro de {p.cpf_masked}
              </button>}
```

- em `ask`, antes do `const list = ...`, acrescente `setSaved(null);` (o aviso some quando a pessoa segue para a triagem).

- [ ] **Step 4: Rode a suíte inteira e os tipos**

Run: `npm test && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/PeopleStep.tsx src/modules/citizen/PeopleStep.test.tsx
/opt/homebrew/bin/git commit -m "feat: let the citizen change or clear a person's neighborhood" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Revisão e prova no navegador

**Files:** nenhum, a não ser que a prova ache um bug (nesse caso, teste + correção + commit `fix:` no wpda; se o bug for do api, avise a sessão dona do api em vez de mexer lá).

- [ ] **Step 1: Revisão do branch.** Um subagente revisor lê `/opt/homebrew/bin/git diff origin/main..HEAD` do wpda contra a spec §4.1 e §6, com atenção a:
  - nenhum campo além dos da spec §4.1 (`neighborhood`, `neighborhood_id`, `reference_units` com `id, name, kind, address {street, number, complement, zip}`);
  - `src/lib/report.ts` e `src/modules/Report.tsx` intocados (`git diff origin/main..HEAD -- src/lib/report.ts src/modules/Report.tsx` vazio): o link público não mostra a unidade de referência;
  - "Prefiro não informar" no começo **não** manda `neighborhood_id`; na troca manda `null`;
  - nenhum `undefined`/`null` possível no texto do bloco de referência;
  - `vi.setSystemTime` em todo teste novo com datas;
  - commits só com arquivos pelo nome (`/opt/homebrew/bin/git show --stat HEAD~3..HEAD` sem `node_modules`).
- [ ] **Step 2: Confira o contrato com o api.** Com o api da fatia 3 rodando, `GET /citizen/neighborhoods` devolve `{ neighborhoods: [...] }` e `GET /citizen/triages/:id` devolve `reference_units` com `address` como objeto de 4 campos (confirmados pelo usuário em 2026-09-28). Se divergir, pare e avise; não adapte o cliente a outro formato sem decisão.
- [ ] **Step 3: Prova no navegador (Curitiba).** Com o api do worktree da fatia 3 em `:3031` (migração e `city:territory:seed[curitiba]` aplicados) e o Vite deste worktree:

  ```bash
  cd apps/wpda/.claude/mod11 && VITE_API_PROXY_TARGET=http://localhost:3031 npx vite --port 5186
  ```

  Abra `http://curitiba.localhost:5186/wpda/`. O login do cidadão (celular + código do log do api, `grep "[otp]"`) é feito pelo usuário; não digite o código. Depois, com screenshot de cada passo:
  1. CPF novo → a pergunta do bairro aparece; buscar "sao" sem acento acha o bairro com acento; escolher um bairro **com** cobertura; responder a triagem; no resultado, o bloco "Sua unidade de referência" aparece com nome, tipo e endereço;
  2. abrir o link público do relatório (`?token=`) → **sem** o bloco (decisão do usuário: só na área logada);
  3. voltar, "Trocar bairro" para um bairro **sem** cobertura → "Bairro atualizado."; nova triagem → **sem** bloco no resultado (a troca não muda a triagem do passo 1, que continua com o bairro antigo; isso é provado pelos specs do api, porque a área logada não tem tela que reabra o resultado de uma triagem antiga);
  4. "Trocar bairro" → "Prefiro não informar" → a pessoa mostra "Bairro não informado"; ao começar outra triagem, a pergunta aparece de novo.
- [ ] **Step 4: Suíte final.** `npm test` e `npx tsc --noEmit` no worktree; registre a contagem no relatório (antes → depois).
- [ ] **Step 5: Pare.** O merge do wpda vem depois do merge do api, e só com autorização explícita do usuário. Não faça push.

---

## Self-review

- **Cobertura da spec:** §6 "CPF novo escolhe o bairro com busca e 'Prefiro não informar'" → Task 3; "pessoa existente sem bairro, uma vez antes de começar" → Task 3; "`neighborhood_id` no `POST /citizen/conversations`" → Tasks 1 e 3; 422 `invalid_neighborhood` → Tasks 3 e 4; "Trocar bairro" com `POST /citizen/people/:id/neighborhood` aceitando `null` → Tasks 1 e 4; bloco "Sua unidade de referência" só na área logada (resultado), a partir de `GET /citizen/triages/:id`, 0 = ausente, várias = todas, e ausente no relatório público por decisão do usuário (2026-09-28) → Task 2; §7.2 wpda (CPF novo, pessoa existente, prefiro não informar, troca, 0/1/2 unidades) → Tasks 2, 3 e 4; prova no navegador (§7.2) → Task 5.
- **Placeholders:** nenhum passo sem código; os ajustes em arquivos existentes dizem a linha âncora.
- **Tipos:** `Who`, `PersonChoice`, `ChooseOutcome` ("done" | "invalid_neighborhood"), `Neighborhood`, `UnitAddress`, `ReferenceUnit`, `STALE_NEIGHBORHOOD` e `linkStyle` são os mesmos nas Tasks 1, 3 e 4; `citizenApi.start` recebe `neighborhoodId` (camelCase) e manda `neighborhood_id`.
- **Review Focus:** os cinco itens têm teste na task dona (Tasks 1, 2 e 3).
