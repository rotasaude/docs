# Módulo 11 — Território (dashboard, fatias 2, 4 e 5) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Telas do módulo 11 no dashboard. O `municipal_admin` cadastra bairros e a cobertura, e informa o endereço da unidade com consulta de CEP no ViaCEP. O desfecho "encaminhado" já vem com a unidade de referência escolhida. Os 5 painéis com cidadão ganham, para todo papel que os lê, o seletor de bairro, e o que vier suprimido aparece como "< 5".

**Architecture:** Cliente HTTP novo em `src/lib/api.ts` para `/territory/*`, com o endereço nas chamadas de unidade e `reference_unit_ids` na fila. As regras puras ficam fora do React: nome de bairro, ViaCEP, endereço e contagem suprimida, em `src/lib/territory.ts`, `viacep.ts`, `unitAddress.ts` e `smallCount.ts`. Um módulo novo (`Territory`) e um formulário de unidade extraído (`UnitForm`). O filtro de bairro mora na URL (`?bairro=`), e um hook (`useNeighborhoodFilter`) o põe na chave e nos parâmetros das 5 consultas. A lista do seletor vem de `GET /admin/api/neighborhoods`, aberta a todo papel que lê os painéis.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom), Vite 5 (proxy de dev), `fetch` + `AbortController` do navegador para o ViaCEP.

**Spec:** `docs/superpowers/specs/2026-09-28-module-11-territory-design.md` (§4 contratos, §5 dashboard, §9 fatias 2, 4 e 5) e `docs/adr/0023.md`. O plano do api (`docs/superpowers/plans/2026-09-28-module-11-territory-api.md`, escrito em paralelo) precisa estar **mergeado antes** do merge deste.

## Global Constraints

- Rotas de território: todas sob `/territory`, só `municipal_admin`; outro papel recebe 403 `missing_role`. Precisa de **uma** entrada nova no proxy do Vite.
- Recusas do território: `name_taken` (422), `blank_name` (422), `inactive_unit` (422), `inactive_neighborhood` (422), `not_found` (404). Unidade: `invalid_zip` (422), `invalid_neighborhood` (422).
- Sem step-up em território nem endereço (D8).
- CEP: o navegador chama `https://viacep.com.br/ws/<cep>/json/` com timeout de **5 s** (`AbortController`). `{erro: true}`, erro de rede e timeout mostram o aviso "não foi possível consultar o CEP", e os campos continuam livres. O bairro do CEP é só sugestão. A pré-seleção vale quando o nome bate, sem diferenciar maiúsculas e acentos. O `api` nunca chama serviço de CEP.
- CSP: hoje não há nenhuma (conferido em `index.html`, `nginx.conf`, `Dockerfile`, `docker-compose.yml` e `apps/api/config`). Se passar a existir, ela precisa de `connect-src https://viacep.com.br`.
- Desfecho "encaminhado": vem escolhida a primeira de `reference_unit_ids` **por nome**. As outras de referência sobem ao topo com a etiqueta "referência". Nunca restringe. A pré-seleção **nunca** escolhe a própria unidade do atendimento (decisão do usuário, 2026-09-28). O api já a exclui de `reference_unit_ids`, e o front a filtra de novo, por defesa.
- Contratos confirmados com o plano do api:
  - `GET /admin/api/neighborhoods` responde `{ neighborhoods: [...] }` **sem** o envelope `data`;
  - nos 5 painéis, o eco do filtro vem em `data.filter = { neighborhood: {id, name} | "none" | null }`, ao lado de `data.scope`;
  - as escritas de `/territory` respondem `{ neighborhood }` (a tela não depende disso) e a lista responde `{ neighborhoods }`;
  - `reference_unit_ids` vem em cada linha da fila (`waiting` e `in_care`), já sem a própria unidade e por nome;
  - a taxa de conclusão suprimida vem com `tone: "neutral"`;
  - o endereço aceita CEP com hífen e grava só dígitos; só as chaves enviadas mudam (o front manda sempre as cinco).
  - Demais: `reference_unit_ids` vem nas linhas de `/attendance/units/:id/queue`; o endereço vem em campos soltos, e `null` apaga o campo; a lista de amostra suprimida vem como `null`; o KPI com valor suprimido vem com `delta: null`.
- Painéis (Visão geral, Classificação, Triagens, Relatórios, Conversas): o seletor vale para **todo papel** que lê os painéis (decisão do usuário, 2026-09-28). As opções são "Todos", "Sem bairro" e os bairros, com os inativos marcados. A escolha fica na URL como `?bairro=` e vai à API como `neighborhood_id=<uuid>` ou `neighborhood_id=none`.
- Lista do seletor: `GET /admin/api/neighborhoods` → `{ neighborhoods: [{ id, name, active }] }`, só leitura. Não é `/territory`, que continua só do `municipal_admin` (tela Território e formulário de unidade).
- Supressão: com o filtro ligado, a API troca de 1 a 4 por `{ suppressed: true }`, e 0 e ≥ 5 vêm como número. Vale para KPI, contagem por categoria, ponto de série, percentual e média. Suprimido aparece como `< 5` com dica. As listas de amostra (`sampleTriages`, linhas de relatório) somem quando o total está suprimido.
- Cache: já é por usuário (`useSessionQueryClient`). A chave leva o bairro e **nunca** o `userId`.
- Testes que dependem de "hoje" fixam o relógio com `vi.setSystemTime` em `beforeEach`. Nenhum teste deste plano depende de data. Se algum passar a depender, a regra vale.
- Nunca `git add -A`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`.

## Ambiente de execução

- O worktree já existe: `apps/dashboard/.claude/mod11`, branch `feat/mod-11-territory`, a partir de `origin/main` `2c67cbe`. Falta o `node_modules`. Crie o symlink para o do checkout principal:

  ```bash
  cd apps/dashboard && ln -s ../../node_modules .claude/mod11/node_modules
  ```

- Testes:

  ```bash
  cd apps/dashboard/.claude/mod11 && npx vitest run <arquivos>
  ```

- Checagem de tipos antes de cada commit:

  ```bash
  cd apps/dashboard/.claude/mod11 && npx tsc --noEmit
  ```

- O `vitest.config.ts` já fixa `TZ=America/Sao_Paulo`. Não há `setupFiles`, e os testes usam `toBeTruthy`/`not.toBeNull()` (sem jest-dom).

## Review Focus

1. **Resposta atrasada do ViaCEP depois que o admin corrigiu o CEP.** A resposta do CEP antigo chega depois da do novo e não pode sobrescrever logradouro nem bairro. Teste: Task 5, "resposta de um CEP já trocado é descartada".
2. **Admin escolheu o bairro à mão e depois digitou o CEP.** A sugestão do ViaCEP não troca uma escolha já feita. Ela só preenche um bairro vazio. Teste: Task 5, "não troca o bairro já escolhido".
3. **Link com `?bairro=` de bairro que não existe mais, de outra cidade ou com lixo.** O painel volta para "Todos" em vez de ficar no 422 `invalid_neighborhood`. Teste: Task 9, "bairro desconhecido na URL volta para Todos", e o caso de lixo em `neighborhoodFilter.test.ts`.
4. **"Atendido e liberado" ou "Retorno" num atendimento com unidade de referência.** A pré-seleção do encaminhamento não pode vazar como `referral_unit_id` em outro desfecho. Teste: Task 6, "pré-seleção não vaza para outro desfecho".
5. **Cobertura de bairro que ainda lista uma unidade desativada depois da carga.** Salvar não pode falhar com `inactive_unit`. A tela avisa que essa unidade sai da cobertura e manda só as ativas. Teste: Task 4, "unidade desativada na cobertura sai ao salvar, com aviso".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `vite.config.ts` | proxy de `/territory` | 1 |
| `src/lib/api.ts` | `/territory/*`; endereço em `createUnit`/`updateUnit`/`HealthUnit`; `QueueRow.reference_unit_ids` | 1 |
| `src/lib/territory.ts` | `normalizeName`, `sortByName`, `matchNeighborhood`, `validateNeighborhoodName`, `territoryError`, `NEIGHBORHOODS_KEY`, `SOURCE_LABEL` | 1 |
| `src/lib/viacep.ts` | `lookupCep` com timeout de 5 s | 2 |
| `src/lib/unitAddress.ts` | máscara de CEP, campos ↔ payload, `formatAddress` | 2 |
| `nginx.conf` | comentário: CSP futura precisa liberar o ViaCEP | 2 |
| `src/shell/modules.ts`, `src/App.tsx`, `README.md` | menu "Cidade → Território" só para `municipal_admin` | 3 |
| `src/modules/Territory.tsx` | tabela, busca, criar/renomear, (des)ativar, cobertura | 4 |
| `src/modules/attendance/UnitForm.tsx`, `Units.tsx`, `src/lib/attendance.ts` | endereço + CEP no formulário de unidade | 5 |
| `src/modules/attendance/UnitQueue.tsx`, `src/lib/attendance.ts` | pré-seleção da referência no desfecho | 6 |
| `src/lib/types.ts`, `src/lib/smallCount.ts`, `src/components/Count.tsx`, `StatTile.tsx`, `Sparkline.tsx`, `BarMini.tsx`, `src/modules/Triages.tsx`, `Reports.tsx` | tipos e exibição de suprimido (1ª metade) | 7 |
| `src/lib/types.ts`, `src/lib/classification.ts`, `src/modules/Classification.tsx`, `Conversations.tsx`, `Overview.tsx` | exibição de suprimido (2ª metade) | 8 |
| `src/lib/api.ts` (`listPanelNeighborhoods`), `src/lib/neighborhoodFilter.ts`, `src/components/NeighborhoodPicker.tsx`, `PageHeader.tsx`, `src/hooks/use{Overview,Classification,Triages,Reports,Conversations}.ts`, os 5 painéis, `Territory.tsx` (invalida a lista do seletor), 3 testes antigos que contavam `fetch` | filtro na URL para todo papel, seletor e chave de consulta | 9 |
| — | conferência do `admin`, suíte inteira, revisão, prova no navegador | 10 |

---

### Task 1: Cliente de território, endereço da unidade e regras de nome

**Files:**
- Modify: `vite.config.ts`, `src/lib/api.ts`
- Create: `src/lib/territory.ts`
- Test: `src/lib/territory.test.ts`

**Interfaces:**
- Produces (em `src/lib/api.ts`):
  - tipos `UnitAddress { address_street; address_number; address_complement; address_zip; neighborhood_id }` (todos `string | null`), `NeighborhoodUnit { id; name; active }`, `Neighborhood { id; name; active; source: "seed" | "manual"; units: NeighborhoodUnit[] }`;
  - `HealthUnit` passa a `extends Partial<UnitAddress>`;
  - `QueueRow.reference_unit_ids?: string[]`;
  - `createUnit(name, kind, address: UnitAddress)`, `updateUnit(id, name, kind, address: UnitAddress)`;
  - `listNeighborhoods(): Promise<Neighborhood[]>`, `createNeighborhood(name): Promise<void>`, `renameNeighborhood(id, name): Promise<void>`, `setNeighborhoodActive(id, active): Promise<void>`, `replaceCoverage(id, healthUnitIds: string[]): Promise<void>`.
- Produces (em `src/lib/territory.ts`): `NEIGHBORHOODS_KEY = ["territoryNeighborhoods"]`, `NAME_MAX = 120`, `SOURCE_LABEL`, `normalizeName(s) → string`, `sortByName<T extends {name}>(rows) → T[]`, `matchNeighborhood(name, list) → Neighborhood | null`, `validateNeighborhoodName(name) → string | null`, `territoryError(err) → string`.

- [ ] **Step 1: Escreva os testes**

```ts
// src/lib/territory.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import {
  ApiError, createUnit, listNeighborhoods, replaceCoverage, setNeighborhoodActive, type Neighborhood
} from "./api";
import { matchNeighborhood, normalizeName, sortByName, territoryError, validateNeighborhoodName } from "./territory";

afterEach(() => vi.unstubAllGlobals());

const n = (id: string, name: string, active = true): Neighborhood => ({ id, name, active, source: "seed", units: [] });

describe("normalizeName", () => {
  it("ignora acento, maiúscula e espaço sobrando", () => {
    expect(normalizeName("São Brás")).toBe("sao bras");
    expect(normalizeName("  ÁGUA   Verde ")).toBe("agua verde");
  });
});

describe("sortByName", () => {
  it("ordena em pt-BR sem mexer na lista original", () => {
    const list = [ n("1", "Centro"), n("2", "Água Verde"), n("3", "Batel") ];
    expect(sortByName(list).map((x) => x.name)).toEqual([ "Água Verde", "Batel", "Centro" ]);
    expect(list[0].name).toBe("Centro");
  });
});

describe("matchNeighborhood", () => {
  const list = [ n("1", "São Francisco"), n("2", "Batel", false) ];
  it("casa sem diferenciar maiúsculas e acentos", () => {
    expect(matchNeighborhood("SAO FRANCISCO", list)?.id).toBe("1");
  });
  it("nunca sugere bairro inativo", () => {
    expect(matchNeighborhood("Batel", list)).toBeNull();
  });
  it("vazio ou sem par: null", () => {
    expect(matchNeighborhood("", list)).toBeNull();
    expect(matchNeighborhood(undefined, list)).toBeNull();
    expect(matchNeighborhood("Rebouças", list)).toBeNull();
  });
});

describe("validateNeighborhoodName", () => {
  it("recusa vazio e nome com mais de 120 caracteres", () => {
    expect(validateNeighborhoodName("   ")).toBe("informe o nome do bairro");
    expect(validateNeighborhoodName("a".repeat(121))).toBe("nome com mais de 120 caracteres");
    expect(validateNeighborhoodName(" Batel ")).toBeNull();
  });
});

describe("territoryError", () => {
  const err = (status: number, error: string) => new ApiError(status, { error }, "x");
  it("traduz as recusas nomeadas", () => {
    expect(territoryError(err(422, "name_taken"))).toBe("já existe um bairro com este nome");
    expect(territoryError(err(422, "blank_name"))).toBe("informe o nome do bairro");
    expect(territoryError(err(422, "inactive_unit"))).toBe("há unidade desativada ou inexistente na cobertura — recarregue e tente de novo");
    expect(territoryError(err(422, "inactive_neighborhood"))).toBe("bairro desativado não recebe cobertura — reative antes");
    expect(territoryError(err(404, "not_found"))).toBe("bairro não encontrado — recarregue a lista");
    expect(territoryError(err(403, "missing_role"))).toBe("seu papel não permite esta ação");
  });
  it("401 e desconhecido", () => {
    expect(territoryError(new ApiError(401, {}, "x"))).toBe("sessão expirada — entre de novo");
    expect(territoryError(new Error("rede"))).toBe("não foi possível concluir — tente de novo");
  });
});

describe("cliente /territory e endereço da unidade", () => {
  function stub(body: unknown) {
    const fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
      new Response(JSON.stringify(body), { status: 200, headers: { "Content-Type": "application/json" } }));
    vi.stubGlobal("fetch", fn);
    return fn;
  }
  const call = (fn: ReturnType<typeof stub>, i = 0) => fn.mock.calls[i] as [ string, RequestInit ];

  it("lista os bairros de /territory/neighborhoods", async () => {
    const fn = stub({ neighborhoods: [ n("1", "Centro") ] });
    expect((await listNeighborhoods()).map((x) => x.name)).toEqual([ "Centro" ]);
    expect(call(fn)[0]).toBe("/territory/neighborhoods");
  });

  it("desativa e reativa por rotas próprias", async () => {
    const fn = stub({});
    await setNeighborhoodActive("n1", false);
    await setNeighborhoodActive("n1", true);
    expect(call(fn, 0)[0]).toBe("/territory/neighborhoods/n1/deactivate");
    expect(call(fn, 1)[0]).toBe("/territory/neighborhoods/n1/activate");
    expect(call(fn, 0)[1].method).toBe("POST");
  });

  it("substitui a cobertura com health_unit_ids", async () => {
    const fn = stub({});
    await replaceCoverage("n1", [ "u1", "u2" ]);
    expect(call(fn)[0]).toBe("/territory/neighborhoods/n1/coverage");
    expect(JSON.parse(String(call(fn)[1].body))).toEqual({ health_unit_ids: [ "u1", "u2" ] });
  });

  it("createUnit manda o endereço achatado, com null onde não há valor", async () => {
    const fn = stub({ unit: { id: "u1", name: "UBS", kind: "ubs", active: true } });
    await createUnit("UBS", "ubs", { address_street: "Rua A", address_number: "1", address_complement: null,
      address_zip: "80010000", neighborhood_id: "n1" });
    expect(JSON.parse(String(call(fn)[1].body))).toEqual({ name: "UBS", kind: "ubs", address_street: "Rua A",
      address_number: "1", address_complement: null, address_zip: "80010000", neighborhood_id: "n1" });
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/territory.test.ts`
Expected: FAIL (`./territory` inexistente; `listNeighborhoods` não exportado).

- [ ] **Step 3: Proxy**

Em `vite.config.ts`, no comentário do topo, depois da linha de `/attendance`:

```ts
//   /professionals → profissionais (módulo 10).
//   /territory  → bairros e cobertura (módulo 11, só municipal_admin).
```

E na lista de `proxy`, depois de `"/professionals"` (note a vírgula na linha anterior):

```ts
      "/professionals": proxy(TARGET),
      "/territory": proxy(TARGET)
```

- [ ] **Step 4: Cliente da API**

Em `src/lib/api.ts`, substitua o bloco `// ─── Atendimento: unidades de saúde (Task 6)` até o fim de `updateUnit` por:

```ts
// ─── Atendimento: unidades de saúde (Task 6) ─────────────────────────────────

// Endereço da unidade (módulo 11, ADR 0023; spec 2026-09-28 §4.1). Opcional em
// tudo: uma API anterior ao módulo 11 omite as chaves. Na escrita vai sempre o
// conjunto inteiro, e null apaga o campo.
export interface UnitAddress {
  address_street: string | null;
  address_number: string | null;
  address_complement: string | null;
  address_zip: string | null;
  neighborhood_id: string | null;
}

export interface HealthUnit extends Partial<UnitAddress> { id: string; name: string; kind: string }
export interface HealthUnitRow extends HealthUnit { active: boolean }

export async function listActiveUnits(): Promise<HealthUnit[]> {
  const payload = await jsonFetch<{ units: HealthUnit[] }>(`${ATTENDANCE_BASE}/units`);
  return payload.units;
}

export async function listAllUnits(): Promise<HealthUnitRow[]> {
  const payload = await jsonFetch<{ units: HealthUnitRow[] }>(`${ATTENDANCE_BASE}/units/all`);
  return payload.units;
}

export async function createUnit(name: string, kind: string, address: UnitAddress): Promise<HealthUnitRow> {
  const payload = await jsonFetch<{ unit: HealthUnitRow }>(`${ATTENDANCE_BASE}/units`, {
    method: "POST", body: JSON.stringify({ name, kind, ...address })
  });
  return payload.unit;
}

export async function updateUnit(id: string, name: string, kind: string, address: UnitAddress): Promise<HealthUnitRow> {
  const payload = await jsonFetch<{ unit: HealthUnitRow }>(`${ATTENDANCE_BASE}/units/${encodeURIComponent(id)}`, {
    method: "POST", body: JSON.stringify({ name, kind, ...address })
  });
  return payload.unit;
}
```

Na interface `QueueRow` (a do atendimento, em `api.ts`), acrescente depois de `called_by_name`:

```ts
  // Módulo 11 (spec 2026-09-28 §4.1): unidades ativas que cobrem o bairro da
  // triagem deste atendimento (sem triagem, o bairro atual do cidadão).
  // Opcional: a API anterior ao módulo 11 não manda.
  reference_unit_ids?: string[];
```

No fim do arquivo:

```ts
// ─── Território (módulo 11, ADR 0023; spec 2026-09-28 §4.1) ─────────────────
// Tudo sob /territory, só municipal_admin — uma entrada no proxy de dev. As
// escritas devolvem o bairro, mas a tela relê a lista: nada aqui depende do
// corpo da resposta.
const TERRITORY_BASE = import.meta.env.VITE_TERRITORY_BASE || "/territory";

export interface NeighborhoodUnit { id: string; name: string; active: boolean }
export interface Neighborhood {
  id: string;
  name: string;
  active: boolean;
  source: "seed" | "manual";
  units: NeighborhoodUnit[];
}

function neighborhoodPath(id: string, action?: string): string {
  return `${TERRITORY_BASE}/neighborhoods/${encodeURIComponent(id)}${action ? `/${action}` : ""}`;
}

export async function listNeighborhoods(): Promise<Neighborhood[]> {
  return (await jsonFetch<{ neighborhoods: Neighborhood[] }>(`${TERRITORY_BASE}/neighborhoods`)).neighborhoods;
}

export async function createNeighborhood(name: string): Promise<void> {
  await jsonFetch<unknown>(`${TERRITORY_BASE}/neighborhoods`, { method: "POST", body: JSON.stringify({ name }) });
}

export async function renameNeighborhood(id: string, name: string): Promise<void> {
  await jsonFetch<unknown>(neighborhoodPath(id), { method: "POST", body: JSON.stringify({ name }) });
}

export async function setNeighborhoodActive(id: string, active: boolean): Promise<void> {
  await jsonFetch<unknown>(neighborhoodPath(id, active ? "activate" : "deactivate"), { method: "POST", body: "{}" });
}

export async function replaceCoverage(id: string, healthUnitIds: string[]): Promise<void> {
  await jsonFetch<unknown>(neighborhoodPath(id, "coverage"), {
    method: "POST", body: JSON.stringify({ health_unit_ids: healthUnitIds })
  });
}
```

`Units.tsx` chama `createUnit`/`updateUnit` com a assinatura antiga e só compila de novo na Task 5. Para o `tsc` passar agora, troque as duas chamadas em `src/modules/attendance/Units.tsx` por:

```tsx
        await updateUnit(form.id, form.name, form.kind, EMPTY_ADDRESS);
```

```tsx
        await createUnit(form.name, form.kind, EMPTY_ADDRESS);
```

com, no topo do arquivo, a constante temporária (a Task 5 a remove):

```tsx
import type { UnitAddress } from "../../lib/api";
// Temporário (Task 1): a Task 5 troca pelo endereço do formulário.
const EMPTY_ADDRESS: UnitAddress = { address_street: null, address_number: null, address_complement: null, address_zip: null, neighborhood_id: null };
```

Em `src/modules/attendance/Units.test.tsx`, os dois `toHaveBeenCalledWith` passam a esperar esse quarto argumento:

```tsx
    await waitFor(() => expect(api.createUnit).toHaveBeenCalledWith("Hospital Sul", "hospital", expect.objectContaining({ address_zip: null })));
```

```tsx
    await waitFor(() => expect(api.updateUnit).toHaveBeenCalledWith("u1", "UBS Centro Novo", "ubs", expect.objectContaining({ address_zip: null })));
```

Confira com `grep -rn "createUnit\|updateUnit" src` que não há outro chamador.

- [ ] **Step 5: Regras de território**

```ts
// src/lib/territory.ts
// Regras do território sem React (módulo 11, ADR 0023; spec 2026-09-28 §5).
import { ApiError, type Neighborhood } from "./api";

// Uma chave só para a lista de bairros: tela Território, formulário de
// unidade e seletor dos painéis leem e invalidam o mesmo cache.
export const NEIGHBORHOODS_KEY = [ "territoryNeighborhoods" ] as const;
export const NAME_MAX = 120;
export const SOURCE_LABEL: Record<Neighborhood["source"], string> = { seed: "semente", manual: "manual" };

// "São Brás" e "sao bras" são o mesmo bairro, na busca e no CEP (D6).
export function normalizeName(s: string): string {
  return s.normalize("NFD").replace(/[̀-ͯ]/g, "").toLowerCase().trim().replace(/\s+/g, " ");
}

export function sortByName<T extends { name: string }>(rows: T[]): T[] {
  return [ ...rows ].sort((a, b) => a.name.localeCompare(b.name, "pt-BR"));
}

// Bairro vindo do ViaCEP → bairro ATIVO da cidade com o mesmo nome. Só
// sugere: quem decide é o admin (D6), e bairro inativo não entra em escolha
// nova (ADR 0023, invariantes).
export function matchNeighborhood(name: string | null | undefined, list: Neighborhood[]): Neighborhood | null {
  const key = normalizeName(name ?? "");
  if (!key) return null;
  return list.find((n) => n.active && normalizeName(n.name) === key) ?? null;
}

export function validateNeighborhoodName(name: string): string | null {
  const trimmed = name.trim();
  if (!trimmed) return "informe o nome do bairro";
  if (trimmed.length > NAME_MAX) return `nome com mais de ${NAME_MAX} caracteres`;
  return null;
}

const GENERIC = "não foi possível concluir — tente de novo";
const MESSAGES: Record<string, string> = {
  name_taken: "já existe um bairro com este nome",
  blank_name: "informe o nome do bairro",
  inactive_unit: "há unidade desativada ou inexistente na cobertura — recarregue e tente de novo",
  inactive_neighborhood: "bairro desativado não recebe cobertura — reative antes",
  not_found: "bairro não encontrado — recarregue a lista",
  missing_role: "seu papel não permite esta ação",
  forbidden: "seu papel não permite esta ação"
};

export function territoryError(err: unknown): string {
  if (!(err instanceof ApiError)) return GENERIC;
  if (err.status === 401) return "sessão expirada — entre de novo";
  const code = (err.body as { error?: string } | undefined)?.error;
  return (code && MESSAGES[code]) || GENERIC;
}
```

- [ ] **Step 6: Rode, cheque tipos e faça commit**

Run: `npx vitest run src/lib/territory.test.ts src/modules/attendance/Units.test.tsx && npx tsc --noEmit`
Expected: PASS.

```bash
/opt/homebrew/bin/git add vite.config.ts src/lib/api.ts src/lib/territory.ts src/lib/territory.test.ts src/modules/attendance/Units.tsx src/modules/attendance/Units.test.tsx
/opt/homebrew/bin/git commit -m "feat: add territory API client and neighborhood name rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 2: Consulta de CEP no ViaCEP e regras de endereço

**Files:**
- Create: `src/lib/viacep.ts`, `src/lib/unitAddress.ts`
- Modify: `nginx.conf`
- Test: `src/lib/viacep.test.ts`, `src/lib/unitAddress.test.ts`

**Interfaces:**
- Consumes: `UnitAddress` (Task 1), `onlyDigits` (`src/lib/attendance.ts`).
- Produces (em `src/lib/viacep.ts`): `VIACEP_TIMEOUT_MS = 5000`, `type CepResult = { ok: true; street: string; neighborhood: string } | { ok: false }`, `lookupCep(cep: string, timeoutMs = VIACEP_TIMEOUT_MS): Promise<CepResult>`, que nunca rejeita.
- Produces (em `src/lib/unitAddress.ts`): `AddressFields { street; number; complement; zip; neighborhoodId }` (strings), `EMPTY_ADDRESS_FIELDS`, `EMPTY_ADDRESS: UnitAddress`, `maskCep(input) → "00000-000"`, `zipError(zip) → string | null`, `addressFieldsFrom(unit: Partial<UnitAddress>) → AddressFields`, `addressPayload(fields) → UnitAddress`, `formatAddress(unit, neighborhoodName?) → string`.

- [ ] **Step 1: Escreva os testes**

```ts
// src/lib/viacep.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import { VIACEP_TIMEOUT_MS, lookupCep } from "./viacep";

afterEach(() => { vi.useRealTimers(); vi.unstubAllGlobals(); });

const respond = (body: unknown, status = 200) =>
  vi.fn(async (_url: string, _init?: RequestInit) => new Response(JSON.stringify(body), { status }));

describe("lookupCep", () => {
  it("consulta o ViaCEP direto do navegador, sem cookie, e devolve logradouro e bairro", async () => {
    const fetchMock = respond({ cep: "80010-000", logradouro: "Rua XV de Novembro", bairro: "Centro", localidade: "Curitiba", uf: "PR" });
    vi.stubGlobal("fetch", fetchMock);
    await expect(lookupCep("80010-000")).resolves.toEqual({ ok: true, street: "Rua XV de Novembro", neighborhood: "Centro" });
    const [ url, init ] = fetchMock.mock.calls[0] as [ string, RequestInit ];
    expect(url).toBe("https://viacep.com.br/ws/80010000/json/");
    expect(init.credentials).toBe("omit");
    expect(init.signal).toBeInstanceOf(AbortSignal);
  });

  it.each([ [ { erro: true } ], [ { erro: "true" } ] ])("CEP inexistente (%o) é falha", async (body) => {
    vi.stubGlobal("fetch", respond(body));
    await expect(lookupCep("99999999")).resolves.toEqual({ ok: false });
  });

  it("HTTP fora de 2xx é falha", async () => {
    vi.stubGlobal("fetch", respond({}, 400));
    await expect(lookupCep("80010000")).resolves.toEqual({ ok: false });
  });

  it("erro de rede é falha, sem rejeitar", async () => {
    vi.stubGlobal("fetch", vi.fn(async () => { throw new TypeError("Failed to fetch"); }));
    await expect(lookupCep("80010000")).resolves.toEqual({ ok: false });
  });

  it("corpo que não é JSON é falha", async () => {
    vi.stubGlobal("fetch", vi.fn(async () => new Response("<html>", { status: 200 })));
    await expect(lookupCep("80010000")).resolves.toEqual({ ok: false });
  });

  it("CEP sem 8 dígitos nem chama a rede", async () => {
    const fetchMock = respond({});
    vi.stubGlobal("fetch", fetchMock);
    await expect(lookupCep("8001")).resolves.toEqual({ ok: false });
    expect(fetchMock).not.toHaveBeenCalled();
  });

  it("desiste em 5 s: aborta e devolve falha", async () => {
    vi.useFakeTimers();
    vi.stubGlobal("fetch", vi.fn((_url: string, init: RequestInit) => new Promise<Response>((_resolve, reject) => {
      init.signal?.addEventListener("abort", () => reject(new DOMException("aborted", "AbortError")));
    })));
    let settled = false;
    const result = lookupCep("80010000").then((r) => { settled = true; return r; });
    await vi.advanceTimersByTimeAsync(VIACEP_TIMEOUT_MS - 1);
    expect(settled).toBe(false);
    await vi.advanceTimersByTimeAsync(1);
    await expect(result).resolves.toEqual({ ok: false });
    expect(VIACEP_TIMEOUT_MS).toBe(5000);
  });
});
```

```ts
// src/lib/unitAddress.test.ts
import { describe, expect, it } from "vitest";
import { EMPTY_ADDRESS, addressFieldsFrom, addressPayload, formatAddress, maskCep, zipError } from "./unitAddress";

describe("maskCep", () => {
  it("põe o hífen e corta em 8 dígitos", () => {
    expect(maskCep("80010000")).toBe("80010-000");
    expect(maskCep("80.010-0009")).toBe("80010-000");
    expect(maskCep("8001")).toBe("8001");
  });
});

describe("zipError", () => {
  it("vazio ou 8 dígitos passam; o resto não", () => {
    expect(zipError("")).toBeNull();
    expect(zipError("80010-000")).toBeNull();
    expect(zipError("8001")).toBe("CEP precisa ter 8 dígitos");
  });
});

describe("addressPayload", () => {
  it("apara, troca vazio por null e manda o CEP só com dígitos", () => {
    expect(addressPayload({ street: " Rua A ", number: "10", complement: "  ", zip: "80010-000", neighborhoodId: "n1" }))
      .toEqual({ address_street: "Rua A", address_number: "10", address_complement: null, address_zip: "80010000", neighborhood_id: "n1" });
  });
  it("formulário vazio vira EMPTY_ADDRESS", () => {
    expect(addressPayload({ street: "", number: "", complement: "", zip: "", neighborhoodId: "" })).toEqual(EMPTY_ADDRESS);
  });
});

describe("addressFieldsFrom", () => {
  it("unidade sem endereço (API antiga) vira campos vazios", () => {
    expect(addressFieldsFrom({})).toEqual({ street: "", number: "", complement: "", zip: "", neighborhoodId: "" });
  });
  it("CEP volta mascarado", () => {
    expect(addressFieldsFrom({ address_zip: "80010000" }).zip).toBe("80010-000");
  });
});

describe("formatAddress", () => {
  it("junta logradouro, número, complemento, bairro e CEP", () => {
    expect(formatAddress({ address_street: "Rua A", address_number: "1", address_complement: "sala 2", address_zip: "80010000" }, "Centro"))
      .toBe("Rua A, 1 — sala 2 · Centro · 80010-000");
    expect(formatAddress({ address_street: "Rua A", address_number: "1" }, "Centro")).toBe("Rua A, 1 · Centro");
  });
  it("sem nada: travessão", () => {
    expect(formatAddress({}, null)).toBe("—");
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/viacep.test.ts src/lib/unitAddress.test.ts`
Expected: FAIL (módulos inexistentes).

- [ ] **Step 3: ViaCEP**

```ts
// src/lib/viacep.ts
// Consulta de CEP direto do navegador (módulo 11, ADR 0023, D6). O api nunca
// chama serviço de CEP. Falhar é normal e barato: {erro: true}, rede, HTTP
// fora de 2xx, corpo estranho ou timeout viram { ok: false }, e a tela deixa
// o endereço para o preenchimento à mão. Nunca rejeita.
export const VIACEP_TIMEOUT_MS = 5000;

export type CepResult = { ok: true; street: string; neighborhood: string } | { ok: false };

export async function lookupCep(cep: string, timeoutMs: number = VIACEP_TIMEOUT_MS): Promise<CepResult> {
  const digits = cep.replace(/\D/g, "");
  if (digits.length !== 8) return { ok: false };
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  try {
    // credentials: "omit": o cookie da cidade não tem o que fazer num terceiro.
    // Sem cabeçalho próprio, para não disparar preflight de CORS.
    const res = await fetch(`https://viacep.com.br/ws/${digits}/json/`, { signal: controller.signal, credentials: "omit" });
    if (!res.ok) return { ok: false };
    const body = (await res.json()) as { erro?: unknown; logradouro?: unknown; bairro?: unknown };
    if (body.erro === true || body.erro === "true") return { ok: false };
    return {
      ok: true,
      street: typeof body.logradouro === "string" ? body.logradouro : "",
      neighborhood: typeof body.bairro === "string" ? body.bairro : ""
    };
  } catch {
    return { ok: false };
  } finally {
    clearTimeout(timer);
  }
}
```

- [ ] **Step 4: Endereço**

```ts
// src/lib/unitAddress.ts
// Endereço da unidade (módulo 11, spec 2026-09-28 §3.3): texto livre +
// CEP de 8 dígitos + bairro da lista da cidade. Os campos da tela são
// strings; o payload troca vazio por null (null apaga na API).
import type { UnitAddress } from "./api";
import { onlyDigits } from "./attendance";

export interface AddressFields { street: string; number: string; complement: string; zip: string; neighborhoodId: string }

export const EMPTY_ADDRESS_FIELDS: AddressFields = { street: "", number: "", complement: "", zip: "", neighborhoodId: "" };
export const EMPTY_ADDRESS: UnitAddress = {
  address_street: null, address_number: null, address_complement: null, address_zip: null, neighborhood_id: null
};

export function maskCep(input: string): string {
  const d = onlyDigits(input).slice(0, 8);
  return d.length > 5 ? `${d.slice(0, 5)}-${d.slice(5)}` : d;
}

export function zipError(zip: string): string | null {
  const d = onlyDigits(zip);
  return d.length === 0 || d.length === 8 ? null : "CEP precisa ter 8 dígitos";
}

export function addressFieldsFrom(unit: Partial<UnitAddress>): AddressFields {
  return {
    street: unit.address_street ?? "",
    number: unit.address_number ?? "",
    complement: unit.address_complement ?? "",
    zip: unit.address_zip ? maskCep(unit.address_zip) : "",
    neighborhoodId: unit.neighborhood_id ?? ""
  };
}

const orNull = (s: string): string | null => (s.trim() === "" ? null : s.trim());

export function addressPayload(fields: AddressFields): UnitAddress {
  const zip = onlyDigits(fields.zip);
  return {
    address_street: orNull(fields.street),
    address_number: orNull(fields.number),
    address_complement: orNull(fields.complement),
    address_zip: zip === "" ? null : zip,
    neighborhood_id: fields.neighborhoodId || null
  };
}

export function formatAddress(unit: Partial<UnitAddress>, neighborhoodName?: string | null): string {
  const street = [ unit.address_street, unit.address_number ].filter(Boolean).join(", ");
  const line = [ street, unit.address_complement ].filter(Boolean).join(" — ");
  const parts = [ line, neighborhoodName ?? "", unit.address_zip ? maskCep(unit.address_zip) : "" ].filter((p) => p.trim() !== "");
  return parts.length > 0 ? parts.join(" · ") : "—";
}
```

- [ ] **Step 5: Confira que não há CSP e deixe o aviso**

Run (da raiz do monorepo):

```bash
grep -rniI "content-security\|content_security\|connect-src" apps/dashboard/.claude/mod11/index.html apps/dashboard/.claude/mod11/nginx.conf apps/dashboard/.claude/mod11/Dockerfile docker-compose.yml apps/api/config
```

Expected: nenhuma linha (o levantamento deste plano não achou CSP). Se aparecer alguma, pare. Acrescente `https://viacep.com.br` ao `connect-src` dessa política, no mesmo commit, e registre o arquivo no relatório da task.

Sem CSP, deixe o aviso para quem criar uma. Em `nginx.conf`, logo depois de `gzip_types ...;`:

```nginx
  # Módulo 11 (ADR 0023): o formulário de unidade consulta o CEP direto do
  # navegador em https://viacep.com.br. Se um dia houver Content-Security-Policy
  # aqui, o connect-src precisa liberar esse host, senão a consulta cai sempre
  # em "não foi possível consultar o CEP".
```

- [ ] **Step 6: Rode, cheque tipos e faça commit**

Run: `npx vitest run src/lib/viacep.test.ts src/lib/unitAddress.test.ts && npx tsc --noEmit`
Expected: PASS.

```bash
/opt/homebrew/bin/git add src/lib/viacep.ts src/lib/viacep.test.ts src/lib/unitAddress.ts src/lib/unitAddress.test.ts nginx.conf
/opt/homebrew/bin/git commit -m "feat: add ViaCEP lookup and unit address helpers

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 3: Navegação — "Cidade → Território" só para `municipal_admin`

**Files:**
- Modify: `src/shell/modules.ts`, `src/App.tsx`, `README.md`
- Create: `src/modules/Territory.tsx` (mínimo; a Task 4 o preenche)
- Test: `src/shell/modules.test.ts`

**Interfaces:**
- Produces: `ModuleId` ganha `"territory"`. `NAV_GROUPS` ganha o grupo `"Cidade"` com `{ id: "territory", label: "Território" }`, visível só para `municipal_admin`. `App` renderiza `<Territory />`.

- [ ] **Step 1: Escreva os testes**

Em `src/shell/modules.test.ts`, os dois testes de contagem mudam, porque o grupo novo também some para operador e para quem ainda não tem sessão:

```ts
    it("operador não vê o grupo Conta (segurança da conta é de usuário de cidade)", () => {
      const groups = navGroupsFor({ operator: true });
      expect(groups.some((g) => g.label === "Conta")).toBe(false);
      expect(groups.some((g) => g.label === "Equipe")).toBe(false);
      expect(groups.some((g) => g.label === "Atendimento")).toBe(false);
      expect(groups.some((g) => g.label === "Cidade")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 4);
    });

    it("sem sessão, esconde Equipe, Atendimento e Cidade", () => {
      const groups = navGroupsFor(null);
      expect(groups.some((g) => g.label === "Conta")).toBe(true);
      expect(groups.some((g) => g.label === "Equipe")).toBe(false);
      expect(groups.some((g) => g.label === "Atendimento")).toBe(false);
      expect(groups.some((g) => g.label === "Cidade")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 3);
    });
```

E acrescente, no fim do `describe("modules")`:

```ts
  describe("módulo 11 na navegação", () => {
    const user = (roles: string[]) => ({ operator: false, memberships: roles.map((role) => ({ role })) });
    const ids = (u: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(u).flatMap((g) => g.items.map((i) => i.id));

    it("Território só para municipal_admin", () => {
      expect(ids(user([ "municipal_admin" ]))).toContain("territory");
      for (const role of [ "viewer", "citizen_verifier", "health_professional", "protocol_publisher" ]) {
        expect(ids(user([ role ]))).not.toContain("territory");
      }
      expect(labelFor("territory")).toBe("Território");
    });
  });
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/shell/modules.test.ts`
Expected: FAIL (`"territory"` não é `ModuleId`; contagens erradas).

- [ ] **Step 3: Implemente**

Em `src/shell/modules.ts`:
- no `ModuleId`, troque `| "professionals" | "my-profile";` por `| "professionals" | "my-profile" | "territory";`;
- em `NAV_GROUPS`, entre o grupo `"Equipe"` e o `"Conta"`:

```ts
  { label: "Cidade", items: [
    { id: "territory", label: "Território", icon: "⌖" }
  ]},
```

- em `navGroupsFor`, no `filter`, depois da linha do `"Equipe"`:

```ts
    // Módulo 11: /territory recusa (403 missing_role) quem não é municipal_admin.
    if (group.label === "Cidade") return isAdmin;
```

Crie `src/modules/Territory.tsx` mínimo:

```tsx
// src/modules/Territory.tsx — preenchido na Task 4.
import { Placeholder } from "./Placeholder";

export function Territory() {
  return <Placeholder title="Território" />;
}
```

Em `src/App.tsx`, importe `import { Territory } from "./modules/Territory";` e acrescente ao `switch`, depois de `my-profile`:

```tsx
    case "territory":      return <Territory />;
```

Em `README.md`, na tabela "Módulos", entre as linhas de Equipe e Conta:

```markdown
| Cidade | Território: bairros, cobertura das unidades (ADR 0023) | `municipal_admin` |
```

E, na frase do proxy, troque `` `/protocols`, `/mfa` e `/attendance` `` por `` `/protocols`, `/mfa`, `/attendance`, `/professionals` e `/territory` ``.

- [ ] **Step 4: Rode, cheque tipos e faça commit**

Run: `npx vitest run src/shell && npx tsc --noEmit`
Expected: PASS.

```bash
/opt/homebrew/bin/git add src/shell/modules.ts src/shell/modules.test.ts src/App.tsx src/modules/Territory.tsx README.md
/opt/homebrew/bin/git commit -m "feat: add territory to the dashboard navigation for city admins

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 4: Tela Território — bairros e cobertura

**Files:**
- Modify: `src/modules/Territory.tsx`
- Test: `src/modules/Territory.test.tsx`

**Interfaces:**
- Consumes: `listNeighborhoods`, `createNeighborhood`, `renameNeighborhood`, `setNeighborhoodActive`, `replaceCoverage`, `listActiveUnits` (Task 1). Também `NEIGHBORHOODS_KEY`, `SOURCE_LABEL`, `normalizeName`, `sortByName`, `validateNeighborhoodName` e `territoryError`.
- Produces: `Territory()`. A lista vive em `NEIGHBORHOODS_KEY` e as unidades ativas em `["activeUnits"]` (a mesma chave do Atendimento). Toda escrita invalida `NEIGHBORHOODS_KEY`.

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/Territory.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, listNeighborhoods: vi.fn(), createNeighborhood: vi.fn(), renameNeighborhood: vi.fn(),
    setNeighborhoodActive: vi.fn(), replaceCoverage: vi.fn(), listActiveUnits: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError, type Neighborhood } from "../lib/api";
import { Territory } from "./Territory";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

const centro: Neighborhood = { id: "n1", name: "Centro", active: true, source: "seed",
  units: [ { id: "u1", name: "UBS Centro", active: true }, { id: "u9", name: "UPA Velha", active: false } ] };
const saoBraz: Neighborhood = { id: "n2", name: "São Braz", active: true, source: "manual", units: [] };
const batel: Neighborhood = { id: "n3", name: "Batel", active: false, source: "seed", units: [] };

function renderIt() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  render(<Territory />, {
    wrapper: ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>
  });
}

const rowOf = (name: string) => within(screen.getByText(name).closest("[role=row]") as HTMLElement);

describe("Territory", () => {
  beforeEach(() => {
    for (const fn of [ api.listNeighborhoods, api.createNeighborhood, api.renameNeighborhood, api.setNeighborhoodActive,
      api.replaceCoverage, api.listActiveUnits ]) mocked(fn).mockReset();
    mocked(api.listNeighborhoods).mockResolvedValue([ centro, saoBraz, batel ]);
    mocked(api.listActiveUnits).mockResolvedValue([
      { id: "u1", name: "UBS Centro", kind: "ubs" }, { id: "u2", name: "UPA Norte", kind: "upa" }
    ]);
  });

  it("lista por nome, com origem, estado e o número de unidades ativas", async () => {
    renderIt();
    await screen.findByText("Centro");
    const rows = screen.getAllByRole("row").slice(1).map((r) => r.textContent ?? "");
    expect(rows[0]).toMatch(/^Batel/);
    expect(rows[1]).toMatch(/^Centro/);
    expect(rows[2]).toMatch(/^São Braz/);
    expect(rowOf("Centro").getByText("semente")).toBeTruthy();
    expect(rowOf("Centro").getByText("ativo")).toBeTruthy();
    expect(rowOf("Centro").getByText("1")).toBeTruthy();
    expect(rowOf("São Braz").getByText("manual")).toBeTruthy();
    expect(rowOf("Batel").getByText("inativo")).toBeTruthy();
  });

  it("busca sem diferenciar maiúsculas e acentos", async () => {
    renderIt();
    await screen.findByText("Centro");
    fireEvent.change(screen.getByLabelText("Buscar bairro"), { target: { value: "SAO" } });
    expect(screen.getByText("São Braz")).toBeTruthy();
    expect(screen.queryByText("Centro")).toBeNull();
  });

  it("cria bairro com o nome aparado e relê a lista", async () => {
    mocked(api.createNeighborhood).mockResolvedValue(undefined);
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Novo bairro" }));
    fireEvent.change(screen.getByLabelText("Nome do bairro"), { target: { value: "  Rebouças " } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
    await waitFor(() => expect(api.createNeighborhood).toHaveBeenCalledWith("Rebouças"));
    await waitFor(() => expect(api.listNeighborhoods).toHaveBeenCalledTimes(2));
    expect(screen.queryByRole("dialog")).toBeNull();
  });

  it("nome vazio é recusado na tela, sem chamar a API", async () => {
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Novo bairro" }));
    fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
    expect((await screen.findByRole("alert")).textContent).toBe("informe o nome do bairro");
    expect(api.createNeighborhood).not.toHaveBeenCalled();
  });

  it("name_taken aparece traduzido e o diálogo continua aberto", async () => {
    mocked(api.createNeighborhood).mockRejectedValue(new ApiError(422, { error: "name_taken" }, "x"));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Novo bairro" }));
    fireEvent.change(screen.getByLabelText("Nome do bairro"), { target: { value: "centro" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
    expect(await screen.findByText("já existe um bairro com este nome")).toBeTruthy();
    expect(screen.getByRole("dialog")).toBeTruthy();
  });

  it("renomeia a partir do nome atual", async () => {
    mocked(api.renameNeighborhood).mockResolvedValue(undefined);
    renderIt();
    await screen.findByText("Centro");
    fireEvent.click(rowOf("Centro").getByRole("button", { name: "Renomear" }));
    const input = screen.getByLabelText("Nome do bairro") as HTMLInputElement;
    expect(input.value).toBe("Centro");
    fireEvent.change(input, { target: { value: "Centro Histórico" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
    await waitFor(() => expect(api.renameNeighborhood).toHaveBeenCalledWith("n1", "Centro Histórico"));
  });

  it("desativa o ativo e reativa o inativo", async () => {
    mocked(api.setNeighborhoodActive).mockResolvedValue(undefined);
    renderIt();
    await screen.findByText("Centro");
    fireEvent.click(rowOf("Centro").getByRole("button", { name: "Desativar" }));
    await waitFor(() => expect(api.setNeighborhoodActive).toHaveBeenCalledWith("n1", false));
    // Uma ação por vez: espera a primeira terminar antes do segundo clique.
    await waitFor(() => expect((rowOf("Centro").getByRole("button", { name: "Desativar" }) as HTMLButtonElement).disabled).toBe(false));
    fireEvent.click(rowOf("Batel").getByRole("button", { name: "Reativar" }));
    await waitFor(() => expect(api.setNeighborhoodActive).toHaveBeenCalledWith("n3", true));
  });

  it("bairro inativo não oferece cobertura", async () => {
    renderIt();
    await screen.findByText("Batel");
    expect(rowOf("Batel").queryByRole("button", { name: "Cobertura" })).toBeNull();
    expect(rowOf("Centro").getByRole("button", { name: "Cobertura" })).toBeTruthy();
  });

  it("cobertura: caixas das unidades ativas, marcadas pelo que já cobre, e substitui o conjunto", async () => {
    mocked(api.replaceCoverage).mockResolvedValue(undefined);
    renderIt();
    await screen.findByText("Centro");
    fireEvent.click(rowOf("Centro").getByRole("button", { name: "Cobertura" }));
    const ubs = await screen.findByLabelText("UBS Centro") as HTMLInputElement;
    const upa = screen.getByLabelText("UPA Norte") as HTMLInputElement;
    expect(ubs.checked).toBe(true);
    expect(upa.checked).toBe(false);
    fireEvent.click(upa);
    fireEvent.click(screen.getByRole("button", { name: "Salvar cobertura" }));
    await waitFor(() => expect(api.replaceCoverage).toHaveBeenCalledWith("n1", [ "u1", "u2" ]));
  });

  it("unidade desativada na cobertura sai ao salvar, com aviso", async () => {
    mocked(api.replaceCoverage).mockResolvedValue(undefined);
    renderIt();
    await screen.findByText("Centro");
    fireEvent.click(rowOf("Centro").getByRole("button", { name: "Cobertura" }));
    expect(await screen.findByText(/UPA Velha/)).toBeTruthy();
    expect(screen.getByText(/saem da cobertura ao salvar/)).toBeTruthy();
    fireEvent.click(screen.getByRole("button", { name: "Salvar cobertura" }));
    await waitFor(() => expect(api.replaceCoverage).toHaveBeenCalledWith("n1", [ "u1" ]));
  });

  it("inactive_unit ao salvar a cobertura aparece traduzido", async () => {
    mocked(api.replaceCoverage).mockRejectedValue(new ApiError(422, { error: "inactive_unit" }, "x"));
    renderIt();
    await screen.findByText("Centro");
    fireEvent.click(rowOf("Centro").getByRole("button", { name: "Cobertura" }));
    await screen.findByLabelText("UBS Centro");
    fireEvent.click(screen.getByRole("button", { name: "Salvar cobertura" }));
    expect(await screen.findByText("há unidade desativada ou inexistente na cobertura — recarregue e tente de novo")).toBeTruthy();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/Territory.test.tsx`
Expected: FAIL (ainda é o placeholder).

- [ ] **Step 3: Implemente a tela**

```tsx
// src/modules/Territory.tsx
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import {
  createNeighborhood, listActiveUnits, listNeighborhoods, renameNeighborhood, replaceCoverage, setNeighborhoodActive,
  type Neighborhood
} from "../lib/api";
import {
  NEIGHBORHOODS_KEY, SOURCE_LABEL, normalizeName, sortByName, territoryError, validateNeighborhoodName
} from "../lib/territory";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../components/formStyles";

// Território (módulo 11; ADR 0023; spec 2026-09-28 §5): só municipal_admin,
// sem step-up (D8). Bairro não se apaga: desativa. A cobertura diz quais
// unidades ATIVAS atendem o bairro, e é dela que sai a unidade de referência
// do cidadão.
type Editing = { kind: "create" } | { kind: "rename"; neighborhood: Neighborhood };

export function Territory() {
  const queryClient = useQueryClient();
  const list = useQuery({ queryKey: NEIGHBORHOODS_KEY, queryFn: listNeighborhoods });
  const [ search, setSearch ] = useState("");
  const [ editing, setEditing ] = useState<Editing | null>(null);
  const [ covering, setCovering ] = useState<Neighborhood | null>(null);
  const [ busyId, setBusyId ] = useState<string | null>(null);
  const [ error, setError ] = useState<string | null>(null);

  function refresh() {
    void queryClient.invalidateQueries({ queryKey: NEIGHBORHOODS_KEY });
  }

  async function toggleActive(n: Neighborhood) {
    if (busyId) return;
    setBusyId(n.id); setError(null);
    try {
      await setNeighborhoodActive(n.id, !n.active);
      refresh();
    } catch (err) {
      setError(territoryError(err));
    } finally {
      setBusyId(null);
    }
  }

  const key = normalizeName(search);
  const rows = sortByName(list.data ?? []).filter((n) => !key || normalizeName(n.name).includes(key));

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Território" sub="bairros · cobertura" />

      {editing && (
        <NameDialog
          key={editing.kind === "rename" ? editing.neighborhood.id : "new"}
          editing={editing}
          onCancel={() => setEditing(null)}
          onDone={() => { setEditing(null); refresh(); }}
        />
      )}

      {covering && (
        <CoverageEditor
          key={covering.id}
          neighborhood={covering}
          onCancel={() => setCovering(null)}
          onDone={() => { setCovering(null); refresh(); }}
        />
      )}

      <Panel title="Bairros" right={
        <button type="button" style={buttonStyle} onClick={() => { setCovering(null); setEditing({ kind: "create" }); }}>
          Novo bairro
        </button>
      }>
        <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
          {error && <p role="alert" style={alertStyle}>{error}</p>}
          {list.isError && <p role="alert" style={alertStyle}>{territoryError(list.error)}</p>}
          <label style={{ ...labelStyle, maxWidth: 320 }}>
            Buscar bairro
            <input value={search} onChange={(e) => setSearch(e.target.value)} style={inputStyle} />
          </label>
          {list.isPending ? (
            <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando…</p>
          ) : (
            <DataTable<Neighborhood>
              cols={[
                { label: "Nome", w: "2fr", render: (n) => n.name },
                { label: "Origem", w: "1fr", render: (n) => <Tag>{SOURCE_LABEL[n.source] ?? n.source}</Tag> },
                { label: "Estado", w: "1fr", render: (n) => <Tag tone={n.active ? "ok" : undefined}>{n.active ? "ativo" : "inativo"}</Tag> },
                // Só as ativas contam: é o que entra na unidade de referência.
                { label: "Unidades", w: "1fr", align: "right", render: (n) =>
                  <span className="mono">{n.units.filter((u) => u.active).length}</span> },
                { label: "", w: "auto", align: "right", render: (n) => (
                  <div style={{ display: "flex", gap: 8, justifyContent: "flex-end" }}>
                    <button type="button" style={secondaryButtonStyle}
                      onClick={() => { setCovering(null); setEditing({ kind: "rename", neighborhood: n }); }}>
                      Renomear
                    </button>
                    {n.active && (
                      <button type="button" style={secondaryButtonStyle} onClick={() => { setEditing(null); setCovering(n); }}>
                        Cobertura
                      </button>
                    )}
                    <button type="button" disabled={busyId === n.id}
                      style={busyId === n.id ? disabledButtonStyle : secondaryButtonStyle}
                      onClick={() => void toggleActive(n)}>
                      {n.active ? "Desativar" : "Reativar"}
                    </button>
                  </div>
                ) }
              ]}
              rows={rows}
              rowKey={(n) => n.id}
              empty={key ? "nenhum bairro com esse nome" : "nenhum bairro cadastrado — carregue a semente da cidade ou crie o primeiro"}
            />
          )}
        </div>
      </Panel>
    </div>
  );
}

function NameDialog({ editing, onCancel, onDone }: { editing: Editing; onCancel(): void; onDone(): void }) {
  const [ name, setName ] = useState(editing.kind === "rename" ? editing.neighborhood.name : "");
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const title = editing.kind === "rename" ? `Renomear ${editing.neighborhood.name}` : "Novo bairro";

  async function save() {
    if (busy) return;
    const invalid = validateNeighborhoodName(name);
    if (invalid) { setError(invalid); return; }
    setBusy(true); setError(null);
    try {
      if (editing.kind === "rename") await renameNeighborhood(editing.neighborhood.id, name.trim());
      else await createNeighborhood(name.trim());
      onDone();
    } catch (err) {
      setError(territoryError(err));
      setBusy(false);
    }
  }

  return (
    <section role="dialog" aria-label={title} style={dialogStyle}>
      <strong>{title}</strong>
      {error && <p role="alert" style={alertStyle}>{error}</p>}
      <label style={labelStyle}>
        Nome do bairro
        <input value={name} onChange={(e) => setName(e.target.value)} style={inputStyle} />
      </label>
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={busy} onClick={() => void save()} style={busy ? disabledButtonStyle : buttonStyle}>Salvar</button>
        <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Cancelar</button>
      </div>
    </section>
  );
}

function CoverageEditor({ neighborhood, onCancel, onDone }: { neighborhood: Neighborhood; onCancel(): void; onDone(): void }) {
  const units = useQuery({ queryKey: [ "activeUnits" ], queryFn: listActiveUnits });
  const [ selected, setSelected ] = useState<Set<string>>(
    () => new Set(neighborhood.units.filter((u) => u.active).map((u) => u.id)));
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  // Unidade desativada depois de entrar na cobertura: a API recusaria
  // (inactive_unit) se ela fosse junto, então sai — e a tela diz isso.
  const leaving = neighborhood.units.filter((u) => !u.active);
  const title = `Cobertura de ${neighborhood.name}`;

  function toggle(id: string) {
    setSelected((prev) => {
      const next = new Set(prev);
      if (next.has(id)) next.delete(id); else next.add(id);
      return next;
    });
  }

  async function save() {
    if (busy || !units.data) return;
    setBusy(true); setError(null);
    try {
      await replaceCoverage(neighborhood.id, sortByName(units.data).filter((u) => selected.has(u.id)).map((u) => u.id));
      onDone();
    } catch (err) {
      setError(territoryError(err));
      setBusy(false);
    }
  }

  return (
    <section role="dialog" aria-label={title} style={dialogStyle}>
      <strong>{title}</strong>
      <p className="mono" style={{ margin: 0, fontSize: 11, color: "var(--ink3)" }}>
        unidades ativas que atendem o bairro — a unidade de referência do cidadão sai daqui
      </p>
      {error && <p role="alert" style={alertStyle}>{error}</p>}
      {units.isError && <p role="alert" style={alertStyle}>{territoryError(units.error)}</p>}
      {units.isPending ? (
        <p className="mono" style={{ margin: 0, fontSize: 10.5, color: "var(--ink3)" }}>carregando…</p>
      ) : (units.data ?? []).length === 0 ? (
        <p style={{ margin: 0, fontSize: 12.5 }}>nenhuma unidade ativa na cidade</p>
      ) : (
        <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
          {sortByName(units.data ?? []).map((u) => (
            <label key={u.id} style={{ display: "flex", gap: 8, alignItems: "center", fontSize: 12.5 }}>
              <input type="checkbox" checked={selected.has(u.id)} onChange={() => toggle(u.id)} />
              {u.name}
            </label>
          ))}
        </div>
      )}
      {leaving.length > 0 && (
        <p role="note" style={{ margin: 0, fontSize: 12, color: "var(--warn)" }}>
          {`Unidades desativadas nesta cobertura saem da cobertura ao salvar: ${leaving.map((u) => u.name).join(", ")}`}
        </p>
      )}
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={busy || !units.data} onClick={() => void save()}
          style={busy || !units.data ? disabledButtonStyle : buttonStyle}>
          Salvar cobertura
        </button>
        <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Cancelar</button>
      </div>
    </section>
  );
}

const labelStyle = { display: "flex", flexDirection: "column" as const, gap: 4, fontSize: 12, color: "var(--ink2)" };
const alertStyle = { margin: 0, fontSize: 12.5, color: "var(--down)" };
const dialogStyle = {
  display: "flex", flexDirection: "column" as const, gap: 10, padding: 16, maxWidth: 480,
  border: "1px solid var(--rule)", borderRadius: 8, background: "var(--panel)"
};
```

Confira em `src/theme/tokens.ts` que `var(--warn)` existe (o `Overview` já o usa). Se o `DataTable` não marcar cada linha com `role="row"`, ajuste o `rowOf` do teste ao seletor de linha que ele usa. O `Reports.test.tsx` já conta `getAllByRole("row")`, então o papel existe.

- [ ] **Step 4: Rode, cheque tipos e faça commit**

Run: `npx vitest run src/modules/Territory.test.tsx && npx tsc --noEmit`
Expected: PASS.

```bash
/opt/homebrew/bin/git add src/modules/Territory.tsx src/modules/Territory.test.tsx
/opt/homebrew/bin/git commit -m "feat: manage neighborhoods and unit coverage in the territory screen

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 5: Endereço com CEP no formulário de unidade

**Files:**
- Create: `src/modules/attendance/UnitForm.tsx`
- Modify: `src/modules/attendance/Units.tsx`, `src/lib/attendance.ts`
- Test: `src/modules/attendance/UnitForm.test.tsx`, `src/modules/attendance/Units.test.tsx`, `src/lib/attendance.test.ts`

**Interfaces:**
- Consumes: `lookupCep`, `VIACEP_TIMEOUT_MS` (Task 2). `maskCep`, `zipError`, `addressFieldsFrom`, `addressPayload`, `formatAddress`, `EMPTY_ADDRESS_FIELDS` e `EMPTY_ADDRESS` (Task 2). `matchNeighborhood`, `sortByName` e `NEIGHBORHOODS_KEY` (Task 1). `listNeighborhoods`, `createUnit` e `updateUnit` (Task 1).
- Produces: `UnitFormValue = AddressFields & { name: string; kind: string }` e o componente `UnitForm({ initial, neighborhoods, busy, cepTimeoutMs?, onSave(value), onCancel() })`. Em `attendance.ts`, as mensagens `invalid_zip` e `invalid_neighborhood`.

- [ ] **Step 1: Escreva os testes do formulário**

```tsx
// src/modules/attendance/UnitForm.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import type { Neighborhood } from "../../lib/api";
import { UnitForm, type UnitFormValue } from "./UnitForm";

afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

const neighborhoods: Neighborhood[] = [
  { id: "n1", name: "Centro", active: true, source: "seed", units: [] },
  { id: "n2", name: "São Francisco", active: true, source: "seed", units: [] },
  { id: "n3", name: "Batel", active: false, source: "seed", units: [] }
];
const blank: UnitFormValue = { name: "UBS Nova", kind: "ubs", street: "", number: "", complement: "", zip: "", neighborhoodId: "" };
const FAIL = /não foi possível consultar o CEP/;

function renderForm(props: Partial<Parameters<typeof UnitForm>[0]> = {}) {
  const onSave = vi.fn();
  render(<UnitForm initial={blank} neighborhoods={neighborhoods} busy={false} onSave={onSave} onCancel={vi.fn()} {...props} />);
  return { onSave };
}

function stubViaCep(bairro: string, logradouro = "Rua XV de Novembro") {
  const fn = vi.fn(async (_url: string, _init?: RequestInit) =>
    new Response(JSON.stringify({ cep: "80010-000", logradouro, bairro, localidade: "Curitiba", uf: "PR" }), { status: 200 }));
  vi.stubGlobal("fetch", fn);
  return fn;
}

const typeCep = (value: string) => fireEvent.change(screen.getByLabelText("CEP"), { target: { value } });
const field = (label: string) => screen.getByLabelText(label) as HTMLInputElement;
const bairro = () => screen.getByLabelText("Bairro") as HTMLSelectElement;

describe("UnitForm — CEP", () => {
  it("sucesso: mascara, preenche o logradouro, mostra a sugestão e pré-seleciona o bairro", async () => {
    const fetchMock = stubViaCep("Centro");
    renderForm();
    typeCep("80010000");
    expect(field("CEP").value).toBe("80010-000");
    expect(await screen.findByText("bairro segundo o CEP: Centro")).toBeTruthy();
    expect(field("Logradouro").value).toBe("Rua XV de Novembro");
    expect(bairro().value).toBe("n1");
    expect((fetchMock.mock.calls[0] as [ string, RequestInit ])[0]).toBe("https://viacep.com.br/ws/80010000/json/");
  });

  it("casa o bairro sem diferenciar maiúsculas e acentos", async () => {
    stubViaCep("SAO FRANCISCO");
    renderForm();
    typeCep("80010000");
    await screen.findByText("bairro segundo o CEP: SAO FRANCISCO");
    expect(bairro().value).toBe("n2");
  });

  it("bairro inativo não é pré-selecionado, mas a sugestão aparece", async () => {
    stubViaCep("Batel");
    renderForm();
    typeCep("80010000");
    await screen.findByText("bairro segundo o CEP: Batel");
    expect(bairro().value).toBe("");
  });

  it("não troca o bairro já escolhido", async () => {
    stubViaCep("Centro");
    renderForm();
    fireEvent.change(bairro(), { target: { value: "n2" } });
    typeCep("80010000");
    await screen.findByText("bairro segundo o CEP: Centro");
    expect(bairro().value).toBe("n2");
  });

  it("{erro: true}: aviso e campos livres", async () => {
    vi.stubGlobal("fetch", vi.fn(async () => new Response(JSON.stringify({ erro: true }), { status: 200 })));
    renderForm();
    typeCep("99999999");
    expect(await screen.findByText(FAIL)).toBeTruthy();
    fireEvent.change(field("Logradouro"), { target: { value: "Rua sem CEP" } });
    expect(field("Logradouro").value).toBe("Rua sem CEP");
  });

  it("falha de rede: aviso", async () => {
    vi.stubGlobal("fetch", vi.fn(async () => { throw new TypeError("Failed to fetch"); }));
    renderForm();
    typeCep("80010000");
    expect(await screen.findByText(FAIL)).toBeTruthy();
  });

  it("timeout: aviso", async () => {
    vi.stubGlobal("fetch", vi.fn((_url: string, init: RequestInit) => new Promise<Response>((_resolve, reject) => {
      init.signal?.addEventListener("abort", () => reject(new DOMException("aborted", "AbortError")));
    })));
    renderForm({ cepTimeoutMs: 30 });
    typeCep("80010000");
    expect(await screen.findByText(FAIL)).toBeTruthy();
  });

  it("antes de 8 dígitos não consulta", () => {
    const fetchMock = stubViaCep("Centro");
    renderForm();
    typeCep("8001");
    expect(fetchMock).not.toHaveBeenCalled();
  });

  it("resposta de um CEP já trocado é descartada", async () => {
    vi.stubGlobal("fetch", vi.fn(async (url: string) => {
      const slow = url.includes("80010000");
      if (slow) await new Promise((r) => setTimeout(r, 50));
      return new Response(JSON.stringify(slow
        ? { logradouro: "Rua Velha", bairro: "Centro" }
        : { logradouro: "Rua Nova", bairro: "São Francisco" }), { status: 200 });
    }));
    renderForm();
    typeCep("80010000");
    typeCep("80020000");
    await screen.findByText("bairro segundo o CEP: São Francisco");
    await new Promise((r) => setTimeout(r, 80));
    expect(screen.queryByText("bairro segundo o CEP: Centro")).toBeNull();
    expect(field("Logradouro").value).toBe("Rua Nova");
    expect(bairro().value).toBe("n2");
  });
});

describe("UnitForm — salvar", () => {
  it("CEP incompleto bloqueia, sem chamar onSave", () => {
    stubViaCep("Centro");
    const { onSave } = renderForm();
    typeCep("8001");
    fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
    expect(screen.getByRole("alert").textContent).toBe("CEP precisa ter 8 dígitos");
    expect(onSave).not.toHaveBeenCalled();
  });

  it("entrega o valor do formulário", async () => {
    stubViaCep("Centro");
    const { onSave } = renderForm();
    typeCep("80010000");
    await screen.findByText("bairro segundo o CEP: Centro");
    fireEvent.change(field("Número"), { target: { value: "100" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
    expect(onSave).toHaveBeenCalledWith(expect.objectContaining({
      name: "UBS Nova", zip: "80010-000", street: "Rua XV de Novembro", number: "100", neighborhoodId: "n1"
    }));
  });

  it("mostra o bairro atual mesmo se ele estiver inativo, marcado", () => {
    renderForm({ initial: { ...blank, neighborhoodId: "n3" } });
    expect(bairro().value).toBe("n3");
    expect(Array.from(bairro().options).map((o) => o.textContent)).toContain("Batel (inativo)");
  });
});
```

Acrescente a `src/lib/attendance.test.ts`, dentro do `describe("attendance helpers")`:

```ts
  it("traduz os erros do endereço da unidade (módulo 11)", () => {
    expect(attendanceError(new ApiError(422, { error: "invalid_zip" }, "x"))).toBe("CEP precisa ter 8 dígitos");
    expect(attendanceError(new ApiError(422, { error: "invalid_neighborhood" }, "x"))).toBe("bairro inválido — escolha outro da lista");
  });
```

- [ ] **Step 2: Ajuste os testes de `Units`**

Em `src/modules/attendance/Units.test.tsx`:
- no `vi.mock`, acrescente `listNeighborhoods: vi.fn()` ao objeto devolvido;
- troque `afterEach(cleanup);` por `afterEach(() => { cleanup(); vi.unstubAllGlobals(); });`;
- troque `const rows = [...]` por:

```tsx
const rows = [
  { id: "u1", name: "UBS Centro", kind: "ubs", active: true, address_street: "Rua A", address_number: "1",
    address_complement: null, address_zip: null, neighborhood_id: "n1" },
  { id: "u2", name: "UPA Norte", kind: "upa", active: false }
];
const centro = { id: "n1", name: "Centro", active: true, source: "seed" as const, units: [] };
```

- no `beforeEach`, inclua `api.listNeighborhoods` no laço de `mockReset` e acrescente `mocked(api.listNeighborhoods).mockResolvedValue([ centro ]);`;
- em "cria uma unidade com nome e tipo", a expectativa passa a ser:

```tsx
    await waitFor(() => expect(api.createUnit).toHaveBeenCalledWith("Hospital Sul", "hospital", EMPTY_ADDRESS));
```

- em "edita uma unidade", a edição preserva o endereço que a unidade já tinha:

```tsx
    await waitFor(() => expect(api.updateUnit).toHaveBeenCalledWith("u1", "UBS Centro Novo", "ubs", {
      address_street: "Rua A", address_number: "1", address_complement: null, address_zip: null, neighborhood_id: "n1"
    }));
```

- importe `import { EMPTY_ADDRESS } from "../../lib/unitAddress";`;
- acrescente dois testes:

```tsx
  it("mostra o endereço com o nome do bairro", async () => {
    renderUnits();
    expect(await screen.findByText("Rua A, 1 · Centro")).not.toBeNull();
  });

  it("cria uma unidade com endereço e o bairro sugerido pelo CEP", async () => {
    vi.stubGlobal("fetch", vi.fn(async () =>
      new Response(JSON.stringify({ logradouro: "Rua XV de Novembro", bairro: "Centro" }), { status: 200 })));
    mocked(api.createUnit).mockResolvedValue({ id: "u3", name: "UBS XV", kind: "ubs", active: true });
    renderUnits();
    await screen.findByText("Rua A, 1 · Centro");
    fireEvent.click(screen.getByRole("button", { name: "Nova unidade" }));
    fireEvent.change(screen.getByLabelText("Nome"), { target: { value: "UBS XV" } });
    fireEvent.change(screen.getByLabelText("CEP"), { target: { value: "80010000" } });
    await screen.findByText("bairro segundo o CEP: Centro");
    fireEvent.change(screen.getByLabelText("Número"), { target: { value: "100" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
    await waitFor(() => expect(api.createUnit).toHaveBeenCalledWith("UBS XV", "ubs", {
      address_street: "Rua XV de Novembro", address_number: "100", address_complement: null,
      address_zip: "80010000", neighborhood_id: "n1"
    }));
  });
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/modules/attendance/UnitForm.test.tsx src/modules/attendance/Units.test.tsx src/lib/attendance.test.ts`
Expected: FAIL (`UnitForm` inexistente, mensagens novas ausentes).

- [ ] **Step 4: Mensagens**

Em `src/lib/attendance.ts`, no fim do objeto `MESSAGES`:

```ts
  invalid_zip: "CEP precisa ter 8 dígitos",
  invalid_neighborhood: "bairro inválido — escolha outro da lista"
```

(com vírgula depois de `missing_role`).

- [ ] **Step 5: Formulário**

```tsx
// src/modules/attendance/UnitForm.tsx
import { useRef, useState } from "react";
import type { Neighborhood } from "../../lib/api";
import { UNIT_KINDS, onlyDigits } from "../../lib/attendance";
import { matchNeighborhood, sortByName } from "../../lib/territory";
import { maskCep, zipError, type AddressFields } from "../../lib/unitAddress";
import { VIACEP_TIMEOUT_MS, lookupCep } from "../../lib/viacep";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";

// Formulário da unidade (módulo 09 + endereço do módulo 11; spec 2026-09-28
// §5, D6). Com 8 dígitos de CEP, o navegador consulta o ViaCEP. O logradouro
// vem preenchido, e o bairro do CEP é só sugestão. Ele pré-seleciona um
// bairro ATIVO de mesmo nome apenas se o admin ainda não escolheu nenhum.
// Qualquer falha deixa os campos livres.
export interface UnitFormValue extends AddressFields { name: string; kind: string }

interface Props {
  initial: UnitFormValue;
  neighborhoods: Neighborhood[];
  busy: boolean;
  cepTimeoutMs?: number;
  onSave(value: UnitFormValue): void;
  onCancel(): void;
}

type CepState = { kind: "idle" } | { kind: "loading" } | { kind: "found"; neighborhood: string } | { kind: "failed" };

export function UnitForm({ initial, neighborhoods, busy, cepTimeoutMs = VIACEP_TIMEOUT_MS, onSave, onCancel }: Props) {
  const [ value, setValue ] = useState<UnitFormValue>(initial);
  const [ cep, setCep ] = useState<CepState>({ kind: "idle" });
  const [ error, setError ] = useState<string | null>(null);
  // Cada CEP digitado ganha um número; resposta de número velho é descartada
  // (o admin corrigiu o CEP antes de a primeira consulta voltar).
  const seq = useRef(0);
  // A lista pode chegar depois do clique: a resposta do ViaCEP casa com a mais nova.
  const neighborhoodsRef = useRef(neighborhoods);
  neighborhoodsRef.current = neighborhoods;

  const set = (patch: Partial<UnitFormValue>) => setValue((v) => ({ ...v, ...patch }));

  // Escolha nova só entre ativos; o bairro atual aparece mesmo se inativo,
  // marcado, para a edição não apagá-lo em silêncio.
  const options = sortByName(neighborhoods.filter((n) => n.active || n.id === value.neighborhoodId));

  async function onZip(raw: string) {
    const zip = maskCep(raw);
    set({ zip });
    const request = ++seq.current;
    if (onlyDigits(zip).length !== 8) { setCep({ kind: "idle" }); return; }
    setCep({ kind: "loading" });
    const result = await lookupCep(zip, cepTimeoutMs);
    if (request !== seq.current) return;
    if (!result.ok) { setCep({ kind: "failed" }); return; }
    setCep({ kind: "found", neighborhood: result.neighborhood });
    const match = matchNeighborhood(result.neighborhood, neighborhoodsRef.current);
    setValue((v) => ({
      ...v,
      street: result.street || v.street,
      neighborhoodId: v.neighborhoodId || match?.id || ""
    }));
  }

  function save() {
    if (busy || !value.name.trim()) return;
    const invalidZip = zipError(value.zip);
    if (invalidZip) { setError(invalidZip); return; }
    setError(null);
    onSave(value);
  }

  const blocked = busy || !value.name.trim();

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 10, maxWidth: 420 }}>
      {error && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{error}</p>}
      <label style={labelStyle}>
        Nome
        <input value={value.name} onChange={(e) => set({ name: e.target.value })} style={inputStyle} />
      </label>
      <label style={labelStyle}>
        Tipo
        <select value={value.kind} onChange={(e) => set({ kind: e.target.value })} style={inputStyle}>
          {UNIT_KINDS.map((k) => <option key={k.value} value={k.value}>{k.label}</option>)}
        </select>
      </label>
      <label style={labelStyle}>
        CEP
        <input value={value.zip} onChange={(e) => void onZip(e.target.value)} style={inputStyle}
          inputMode="numeric" placeholder="00000-000" />
      </label>
      {cep.kind === "loading" && <p className="mono" style={hintStyle}>consultando o CEP…</p>}
      {cep.kind === "failed" && (
        <p role="status" style={{ ...hintStyle, color: "var(--warn)" }}>não foi possível consultar o CEP — preencha o endereço à mão</p>
      )}
      {cep.kind === "found" && cep.neighborhood && (
        <p role="status" style={hintStyle}>{`bairro segundo o CEP: ${cep.neighborhood}`}</p>
      )}
      <label style={labelStyle}>
        Logradouro
        <input value={value.street} onChange={(e) => set({ street: e.target.value })} style={inputStyle} />
      </label>
      <div style={{ display: "flex", gap: 8 }}>
        <label style={{ ...labelStyle, width: 110 }}>
          Número
          <input value={value.number} onChange={(e) => set({ number: e.target.value })} style={inputStyle} />
        </label>
        <label style={{ ...labelStyle, flex: 1 }}>
          Complemento
          <input value={value.complement} onChange={(e) => set({ complement: e.target.value })} style={inputStyle} />
        </label>
      </div>
      <label style={labelStyle}>
        Bairro
        <select value={value.neighborhoodId} onChange={(e) => set({ neighborhoodId: e.target.value })} style={inputStyle}>
          <option value="">—</option>
          {options.map((n) => <option key={n.id} value={n.id}>{n.active ? n.name : `${n.name} (inativo)`}</option>)}
        </select>
      </label>
      <div style={{ display: "flex", gap: 8 }}>
        <button type="button" disabled={blocked} onClick={save} style={blocked ? disabledButtonStyle : buttonStyle}>Salvar</button>
        <button type="button" disabled={busy} onClick={onCancel} style={secondaryButtonStyle}>Cancelar</button>
      </div>
    </div>
  );
}

const labelStyle = { display: "flex", flexDirection: "column" as const, gap: 4, fontSize: 12, color: "var(--ink2)" };
const hintStyle = { margin: 0, fontSize: 11.5, color: "var(--ink3)" };
```

- [ ] **Step 6: `Units` usa o formulário**

Substitua `src/modules/attendance/Units.tsx` inteiro. Isso também remove a constante temporária da Task 1:

```tsx
import { useEffect, useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { createUnit, listAllUnits, listNeighborhoods, setUnitActive, updateUnit, type HealthUnitRow } from "../../lib/api";
import { UNIT_KINDS, attendanceError } from "../../lib/attendance";
import { NEIGHBORHOODS_KEY } from "../../lib/territory";
import { EMPTY_ADDRESS_FIELDS, addressFieldsFrom, addressPayload, formatAddress } from "../../lib/unitAddress";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { buttonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { UnitForm, type UnitFormValue } from "./UnitForm";

// Units (Task 6) — cadastro de unidades de saúde, só para municipal_admin.
// Lista + criar/editar (UnitForm, com/sem id) + desativar/reativar. Desde o
// módulo 11 a unidade tem endereço (CEP pelo ViaCEP) e o bairro onde fica.
// Criar ou reativar muda quem entra em "unidades ativas" (UnitPicker,
// destino de encaminhamento em UnitQueue): invalida a query `activeUnits`
// para essas telas recarregarem (card dashboard#4, item 1 — Task 7).
const KIND_LABEL: Record<string, string> = Object.fromEntries(UNIT_KINDS.map((k) => [ k.value, k.label ]));

interface Editing { id: string | null; initial: UnitFormValue }

export function Units() {
  const queryClient = useQueryClient();
  const neighborhoods = useQuery({ queryKey: NEIGHBORHOODS_KEY, queryFn: listNeighborhoods });
  const [ rows, setRows ] = useState<HealthUnitRow[] | null>(null);
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const [ editing, setEditing ] = useState<Editing | null>(null);

  async function load() {
    try {
      setRows(await listAllUnits());
    } catch (err) {
      setError(attendanceError(err));
    }
  }

  useEffect(() => { void load(); }, []);

  async function toggleActive(row: HealthUnitRow) {
    if (busy) return;
    setBusy(true); setError(null);
    try {
      await setUnitActive(row.id, !row.active);
      if (!row.active) void queryClient.invalidateQueries({ queryKey: [ "activeUnits" ] });
      await load();
    } catch (err) {
      setError(attendanceError(err));
    } finally {
      setBusy(false);
    }
  }

  async function save(value: UnitFormValue) {
    if (busy || !editing) return;
    setBusy(true); setError(null);
    try {
      const address = addressPayload(value);
      if (editing.id) {
        await updateUnit(editing.id, value.name, value.kind, address);
      } else {
        await createUnit(value.name, value.kind, address);
        void queryClient.invalidateQueries({ queryKey: [ "activeUnits" ] });
      }
      setEditing(null);
      await load();
    } catch (err) {
      setError(attendanceError(err));
    } finally {
      setBusy(false);
    }
  }

  const nameOf = new Map((neighborhoods.data ?? []).map((n) => [ n.id, n.name ]));

  return (
    <Panel
      title="Unidades"
      right={!editing && (
        <button type="button" style={buttonStyle}
          onClick={() => setEditing({ id: null, initial: { name: "", kind: "ubs", ...EMPTY_ADDRESS_FIELDS } })}>
          Nova unidade
        </button>
      )}
    >
      <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
        {error && <p role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>{error}</p>}

        {editing && (
          <UnitForm
            key={editing.id ?? "new"}
            initial={editing.initial}
            neighborhoods={neighborhoods.data ?? []}
            busy={busy}
            onSave={(value) => void save(value)}
            onCancel={() => setEditing(null)}
          />
        )}

        {rows && (
          <DataTable<HealthUnitRow>
            cols={[
              { label: "Nome", w: "2fr", render: (r) => r.name },
              { label: "Tipo", w: "1fr", render: (r) => KIND_LABEL[r.kind] ?? r.kind },
              { label: "Endereço", w: "3fr", render: (r) =>
                formatAddress(r, r.neighborhood_id ? nameOf.get(r.neighborhood_id) : null) },
              { label: "Situação", w: "1fr", render: (r) => <Tag tone={r.active ? "ok" : undefined}>{r.active ? "ativa" : "inativa"}</Tag> },
              {
                label: "", w: "auto", align: "right", render: (r) => (
                  <div style={{ display: "flex", gap: 8, justifyContent: "flex-end" }}>
                    <button type="button" style={secondaryButtonStyle}
                      onClick={() => setEditing({ id: r.id, initial: { name: r.name, kind: r.kind, ...addressFieldsFrom(r) } })}>
                      Editar
                    </button>
                    <button type="button" style={secondaryButtonStyle} disabled={busy} onClick={() => void toggleActive(r)}>
                      {r.active ? "Desativar" : "Reativar"}
                    </button>
                  </div>
                )
              }
            ]}
            rows={rows}
            rowKey={(r) => r.id}
            empty="nenhuma unidade cadastrada"
          />
        )}
      </div>
    </Panel>
  );
}
```

- [ ] **Step 7: Rode, cheque tipos e faça commit**

Run: `npx vitest run src/modules/attendance src/lib/attendance.test.ts && npx tsc --noEmit`
Expected: PASS, sem quebra nos testes de `Attendance`/`UnitPicker`/`CheckIn`. Se algum teste de `Attendance.test.tsx` renderizar `Units` sem mockar `listNeighborhoods`, acrescente o mock com `[]`. O comportamento que ele prova não muda.

```bash
/opt/homebrew/bin/git add src/modules/attendance/UnitForm.tsx src/modules/attendance/UnitForm.test.tsx src/modules/attendance/Units.tsx src/modules/attendance/Units.test.tsx src/lib/attendance.ts src/lib/attendance.test.ts
/opt/homebrew/bin/git commit -m "feat: add address with CEP lookup to the health unit form

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Se o passo anterior mexeu em `src/modules/Attendance.test.tsx`, inclua esse arquivo no `git add`.

### Task 6: Desfecho "encaminhado" com a unidade de referência

**Files:**
- Modify: `src/lib/attendance.ts`, `src/modules/attendance/UnitQueue.tsx`
- Test: `src/lib/attendance.test.ts`, `src/modules/attendance/UnitQueue.test.tsx`

**Interfaces:**
- Consumes: `QueueRow.reference_unit_ids?` (Task 1).
- Produces: `splitReferenceUnits<T extends { id; name }>(units: T[], referenceIds: string[] | undefined, currentUnitId: string) → { referenceUnits: T[]; otherUnits: T[] }`. As de referência vêm por nome, sem a própria unidade do atendimento, e as outras na ordem recebida. A própria unidade fica entre as outras, sem etiqueta.

- [ ] **Step 1: Escreva os testes**

Em `src/lib/attendance.test.ts` (importe `splitReferenceUnits`):

```ts
describe("splitReferenceUnits", () => {
  const units = [ { id: "u1", name: "UBS Centro" }, { id: "u2", name: "UPA Norte" }, { id: "u3", name: "Hospital Sul" } ];

  it("sobe as de referência, por nome, e mantém a ordem das outras", () => {
    const { referenceUnits, otherUnits } = splitReferenceUnits(units, [ "u2", "u3" ], "u1");
    expect(referenceUnits.map((u) => u.id)).toEqual([ "u3", "u2" ]);
    expect(otherUnits.map((u) => u.id)).toEqual([ "u1" ]);
  });

  it("id de referência fora das unidades ativas é ignorado; sem ids, nada muda", () => {
    expect(splitReferenceUnits(units, [ "sumiu" ], "u1").referenceUnits).toEqual([]);
    expect(splitReferenceUnits(units, undefined, "u1").otherUnits).toEqual(units);
  });

  it("a própria unidade do atendimento nunca é referência, mesmo se vier na lista", () => {
    const { referenceUnits, otherUnits } = splitReferenceUnits(units, [ "u1", "u2" ], "u1");
    expect(referenceUnits.map((u) => u.id)).toEqual([ "u2" ]);
    expect(otherUnits.map((u) => u.id)).toEqual([ "u1", "u3" ]);
  });
});
```

Em `src/modules/attendance/UnitQueue.test.tsx`, troque o `renderQueue` para aceitar as unidades:

```tsx
function renderQueue(props: {
  canCare: boolean; careBlocked?: string | null; onClinicalRefused?(): void; units?: api.HealthUnit[]
}) {
  const { units = [ unit, otherUnit ], ...rest } = props;
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  render(<UnitQueue unit={unit} units={units} {...rest} />, { wrapper });
  return { client };
}
```

e acrescente, dentro de `describe("profissional (canCare)")`:

```tsx
    describe("unidade de referência (módulo 11)", () => {
      const hospital = { id: "u3", name: "Hospital Sul", kind: "hospital" };
      const withRefs = (ids: string[]) => mocked(api.listUnitQueue).mockResolvedValue({
        waiting, in_care: [ { ...inCare[0], reference_unit_ids: ids } ]
      });

      it("pré-seleciona a primeira de referência por nome e as sobe com a etiqueta", async () => {
        withRefs([ "u2", "u3" ]);
        renderQueue({ canCare: true, units: [ unit, otherUnit, hospital ] });
        fireEvent.click(await screen.findByRole("button", { name: "Encerrar" }));
        fireEvent.change(screen.getByLabelText("Desfecho"), { target: { value: "referred" } });
        const select = screen.getByLabelText("Unidade de destino") as HTMLSelectElement;
        expect(Array.from(select.options).map((o) => o.textContent)).toEqual([
          "—", "Hospital Sul · referência", "UPA Norte · referência", "UBS Centro"
        ]);
        expect(select.value).toBe("u3");
        expect(screen.getByText("Gera pedido de agendamento na Hospital Sul")).not.toBeNull();
      });

      it("confirma com a pré-seleção sem o profissional mexer", async () => {
        withRefs([ "u2" ]);
        mocked(api.closeAttendance).mockResolvedValue({ attendance: { id: "a3" },
          appointmentRequest: { id: "r1", kind: "referral", target_unit_name: "UPA Norte", status: "open" } });
        renderQueue({ canCare: true });
        fireEvent.click(await screen.findByRole("button", { name: "Encerrar" }));
        fireEvent.change(screen.getByLabelText("Desfecho"), { target: { value: "referred" } });
        fireEvent.click(screen.getByRole("button", { name: "Confirmar encerramento" }));
        await waitFor(() => expect(api.closeAttendance).toHaveBeenCalledWith("a3", "referred", "u2", undefined));
      });

      it("o profissional troca para '—' e a pré-seleção não volta", async () => {
        withRefs([ "u2" ]);
        renderQueue({ canCare: true });
        fireEvent.click(await screen.findByRole("button", { name: "Encerrar" }));
        fireEvent.change(screen.getByLabelText("Desfecho"), { target: { value: "referred" } });
        fireEvent.change(screen.getByLabelText("Unidade de destino"), { target: { value: "" } });
        expect((screen.getByLabelText("Unidade de destino") as HTMLSelectElement).value).toBe("");
        expect((screen.getByRole("button", { name: "Confirmar encerramento" }) as HTMLButtonElement).disabled).toBe(true);
      });

      it("pré-seleção não vaza para outro desfecho", async () => {
        withRefs([ "u2" ]);
        mocked(api.closeAttendance).mockResolvedValue({ attendance: { id: "a3" }, appointmentRequest: null });
        renderQueue({ canCare: true });
        fireEvent.click(await screen.findByRole("button", { name: "Encerrar" }));
        fireEvent.click(screen.getByRole("button", { name: "Confirmar encerramento" }));
        await waitFor(() => expect(api.closeAttendance).toHaveBeenCalledWith("a3", "discharged", undefined, undefined));
      });

      it("referência só = a própria unidade: nada pré-selecionado e sem etiqueta", async () => {
        withRefs([ "u1" ]);
        renderQueue({ canCare: true });
        fireEvent.click(await screen.findByRole("button", { name: "Encerrar" }));
        fireEvent.change(screen.getByLabelText("Desfecho"), { target: { value: "referred" } });
        const select = screen.getByLabelText("Unidade de destino") as HTMLSelectElement;
        expect(select.value).toBe("");
        expect(Array.from(select.options).map((o) => o.textContent)).toEqual([ "—", "UBS Centro", "UPA Norte" ]);
        expect(screen.queryByText(/Gera pedido de agendamento/)).toBeNull();
      });

      it("referência que não está entre as ativas é ignorada", async () => {
        withRefs([ "desativada" ]);
        renderQueue({ canCare: true });
        fireEvent.click(await screen.findByRole("button", { name: "Encerrar" }));
        fireEvent.change(screen.getByLabelText("Desfecho"), { target: { value: "referred" } });
        const select = screen.getByLabelText("Unidade de destino") as HTMLSelectElement;
        expect(select.value).toBe("");
        expect(Array.from(select.options).map((o) => o.textContent)).toEqual([ "—", "UBS Centro", "UPA Norte" ]);
      });
    });
```

Se o arquivo não tiver `import * as api` com o tipo `HealthUnit` acessível, use `import type { HealthUnit } from "../../lib/api"` e tipe `units?: HealthUnit[]`.

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/attendance.test.ts src/modules/attendance/UnitQueue.test.tsx`
Expected: FAIL nos casos novos.

- [ ] **Step 3: Implemente**

Em `src/lib/attendance.ts`, depois de `currentUnitKey`:

```ts
// Módulo 11 (ADR 0023, D4): as unidades de referência do bairro da triagem
// sobem para o topo do destino do encaminhamento, por nome. Id que não está
// entre as ativas (unidade desativada depois da triagem) é ignorado: nunca se
// sugere unidade que a API recusaria (invalid_unit). A própria unidade do
// atendimento nunca é referência (decisão do usuário, 2026-09-28): o api já a
// exclui, e aqui ela sai de novo, por defesa.
export function splitReferenceUnits<T extends { id: string; name: string }>(
  units: T[], referenceIds: string[] | undefined, currentUnitId: string
): { referenceUnits: T[]; otherUnits: T[] } {
  const ids = new Set(referenceIds ?? []);
  ids.delete(currentUnitId);
  return {
    referenceUnits: units.filter((u) => ids.has(u.id)).sort((a, b) => a.name.localeCompare(b.name, "pt-BR")),
    otherUnits: units.filter((u) => !ids.has(u.id))
  };
}
```

Em `src/modules/attendance/UnitQueue.tsx`:
- importe `splitReferenceUnits` de `../../lib/attendance` (junto de `ATTENDANCE_REFETCH_MS` e `attendanceError`);
- em `ClosePanel`, troque `const [ referralUnitId, setReferralUnitId ] = useState("");` por:

```tsx
  // Módulo 11 (D4): a primeira unidade de referência, por nome, já vem
  // escolhida; o profissional troca ou volta para "—". null = ainda não
  // mexeu, e aí vale a sugestão (que pode chegar depois, com `units`).
  const [ referralChoice, setReferralChoice ] = useState<string | null>(null);
  const { referenceUnits, otherUnits } = splitReferenceUnits(units, row.reference_unit_ids, unit.id);
  const referralUnitId = referralChoice ?? referenceUnits[0]?.id ?? "";
```

- em `confirm`, a unidade só vai quando o desfecho é "encaminhado":

```tsx
      const result = await closeAttendance(row.id, outcome,
        outcome === "referred" ? referralUnitId || undefined : undefined, note || undefined);
```

- o `<select>` da "Unidade de destino" passa a:

```tsx
            <select value={referralUnitId} onChange={(e) => setReferralChoice(e.target.value)} style={inputStyle}>
              <option value="">—</option>
              {referenceUnits.map((u) => <option key={u.id} value={u.id}>{`${u.name} · referência`}</option>)}
              {otherUnits.map((u) => <option key={u.id} value={u.id}>{u.name}</option>)}
            </select>
```

`referralIncomplete` e `targetUnitName` continuam lendo `referralUnitId`, que agora é derivado.

- [ ] **Step 4: Rode, cheque tipos e faça commit**

Run: `npx vitest run src/lib/attendance.test.ts src/modules/attendance/UnitQueue.test.tsx && npx tsc --noEmit`
Expected: PASS, com os testes antigos da fila intactos (linha sem `reference_unit_ids` = comportamento de hoje).

```bash
/opt/homebrew/bin/git add src/lib/attendance.ts src/lib/attendance.test.ts src/modules/attendance/UnitQueue.tsx src/modules/attendance/UnitQueue.test.tsx
/opt/homebrew/bin/git commit -m "feat: preselect the reference unit when closing an attendance as referred

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 7: Contagem suprimida — primitivos, Triagens e Relatórios

**Files:**
- Modify: `src/lib/types.ts`, `src/components/StatTile.tsx`, `src/components/Sparkline.tsx`, `src/components/BarMini.tsx`, `src/modules/Triages.tsx`, `src/modules/Reports.tsx`
- Create: `src/lib/smallCount.ts`, `src/components/Count.tsx`
- Test: `src/lib/smallCount.test.ts`, `src/components/StatTile.test.tsx`, `src/modules/panels.test.tsx`, `src/modules/Reports.test.tsx`

**Interfaces:**
- Produces (em `types.ts`): `Suppressed { suppressed: true }`, `SmallCount = number | Suppressed`, `NeighborhoodFilterEcho = { id; name } | "none" | null`, `Filtered { filter?: { neighborhood: NeighborhoodFilterEcho } }`. `TriagesData` e `ReportsData` passam a usar `SmallCount`, e `ReportsData.reports` fica opcional e anulável.
- Produces (em `smallCount.ts`): `SUPPRESSED_LABEL = "< 5"`, `SUPPRESSED_HINT`, `isSuppressed(v): v is Suppressed`, `fmtCount(v, unit?: "%")`, `numberOrNull(v)`, `chartSeries(values) → (number | null)[]` e `whenAllCounted(rows) → rows com count:number | null`.
- Produces (em `Count.tsx`): `<Count value unit? />`, `<CountList items />` (lista com `aria-label="contagens"`) e `<SegmentsOrList segments render />`.
- `StatTile.value` aceita `SmallCount`, e `StatTile.spark` aceita `SmallCount[]`. `Sparkline`/`BarMini` aceitam `(number | null)[]`, e null vira lacuna.

- [ ] **Step 1: Escreva os testes**

```ts
// src/lib/smallCount.test.ts
import { describe, expect, it } from "vitest";
import { SUPPRESSED_LABEL, chartSeries, fmtCount, isSuppressed, numberOrNull, whenAllCounted } from "./smallCount";

const s = { suppressed: true as const };

describe("smallCount", () => {
  it("reconhece o suprimido e só ele", () => {
    expect(isSuppressed(s)).toBe(true);
    expect(isSuppressed(0)).toBe(false);
    expect(isSuppressed({ suppressed: false })).toBe(false);
    expect(isSuppressed(null)).toBe(false);
  });

  it("formata: suprimido é '< 5'; 0 continua 0", () => {
    expect(fmtCount(s)).toBe(SUPPRESSED_LABEL);
    expect(SUPPRESSED_LABEL).toBe("< 5");
    expect(fmtCount(0)).toBe("0");
    expect(fmtCount(1234)).toBe("1.234");
    expect(fmtCount(75, "%")).toBe("75%");
    expect(fmtCount(s, "%")).toBe("< 5");
  });

  it("série para gráfico: suprimido vira lacuna, nunca 0", () => {
    expect(chartSeries([ 0, s, 7 ])).toEqual([ 0, null, 7 ]);
    expect(numberOrNull(s)).toBeNull();
  });

  it("whenAllCounted: null se alguma categoria está suprimida", () => {
    expect(whenAllCounted([ { label: "a", count: 1 }, { label: "b", count: 0 } ])).toHaveLength(2);
    expect(whenAllCounted([ { label: "a", count: s } ])).toBeNull();
  });
});
```

```tsx
// src/components/StatTile.test.tsx
import { afterEach, describe, expect, it } from "vitest";
import { cleanup, render, screen } from "@testing-library/react";
import { StatTile } from "./StatTile";
import { SUPPRESSED_HINT } from "../lib/smallCount";

afterEach(cleanup);

describe("StatTile", () => {
  it("suprimido: '< 5' com a dica, sem unidade nem delta", () => {
    render(<StatTile label="Taxa" value={{ suppressed: true }} unit="%" delta="+2" tone="ok" />);
    const value = screen.getByText("< 5");
    expect(value.getAttribute("title")).toBe(SUPPRESSED_HINT);
    expect(screen.queryByText("%")).toBeNull();
    expect(screen.queryByText("+2")).toBeNull();
  });

  it("número segue como antes", () => {
    render(<StatTile label="Iniciadas" value={1234} delta="+2" />);
    expect(screen.getByText("1.234")).toBeTruthy();
    expect(screen.getByText("+2")).toBeTruthy();
  });
});
```

Em `src/modules/panels.test.tsx`, importe `within` de `@testing-library/react`, `Suppressed` de `../lib/types` e `SUPPRESSED_HINT` de `../lib/smallCount`, e acrescente em `describe("Triages (F-05.8)")`:

```tsx
  it("com bairro: contagem e taxa suprimidas aparecem como '< 5' com a dica", async () => {
    const s: Suppressed = { suppressed: true };
    stubRoutes({ "/triages": { ...triages, started: s, completed: 0, completionRate: s, series: [ 0, s, 7 ],
      byProtocol: [ { version: "resp · 2", count: s, share: s, status: "active" } ] } });
    renderPanel(<Triages />);

    const row = await kpiRow();
    expect(row.getAllByText("< 5")).toHaveLength(2);
    expect(row.getAllByTitle(SUPPRESSED_HINT)).toHaveLength(2);
    expect(row.getByText("0")).toBeTruthy();
    expect(row.queryByText("%")).toBeNull();
    expect(within(screen.getByRole("region", { name: "Por protocolo / versão" })).getAllByText("< 5")).toHaveLength(2);
  });
```

Em `src/modules/Reports.test.tsx`:

```tsx
  it("com bairro e total suprimido: a lista some e o total aparece como '< 5'", async () => {
    stubFetch(200, { data: { total: { suppressed: true } }, as_of: "2026-09-26T12:00:00Z" });
    renderReports();

    expect(await screen.findByText("lista oculta")).not.toBeNull();
    expect(screen.getByText("< 5")).not.toBeNull();
    expect(screen.queryByRole("table")).toBeNull();
  });
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/lib/smallCount.test.ts src/components/StatTile.test.tsx src/modules/panels.test.tsx src/modules/Reports.test.tsx`
Expected: FAIL (módulos inexistentes; painéis quebram com objeto no lugar de número).

- [ ] **Step 3: Tipos**

Em `src/lib/types.ts`, logo depois do comentário do topo:

```ts
// Módulo 11 (spec 2026-09-28 §4.3): com o filtro de bairro ligado, a API
// troca todo número de 1 a 4 por { suppressed: true } — KPI, contagem por
// categoria, ponto de série, percentual e média sobre total de 1 a 4. 0 e
// ≥ 5 continuam números. Sem filtro, nada muda.
export interface Suppressed { suppressed: true }
export type SmallCount = number | Suppressed;

// Eco do filtro aplicado (spec §4.1): null = sem filtro; "none" = sem bairro.
export type NeighborhoodFilterEcho = { id: string; name: string } | "none" | null;
export interface Filtered { filter?: { neighborhood: NeighborhoodFilterEcho } }
```

Troque `TriagesData` e `ReportsData` por:

```ts
export interface TriagesData extends Filtered {
  series: SmallCount[];
  started: SmallCount;
  completed: SmallCount;
  completionRate: SmallCount;
  byProtocol: Array<{ version: string; count: SmallCount; share: SmallCount; status: string }>;
}
```

```ts
export interface ReportsData extends Filtered {
  // null = oculta pela supressão (contrato confirmado); ausente também é
  // tratado como oculta, por tolerância. [] = nenhum no período.
  reports?: ReportRow[] | null;
  total: SmallCount;
}
```

- [ ] **Step 4: Regras e componentes**

```ts
// src/lib/smallCount.ts
// Exibição de contagem suprimida (módulo 11; ADR 0023, "Supressão"). A regra
// é da API; aqui só se mostra "< 5" e nunca se trata suprimido como zero.
import type { SmallCount, Suppressed } from "./types";
import { fmtNumber, fmtPercent } from "./format";

export const SUPPRESSED_LABEL = "< 5";
export const SUPPRESSED_HINT =
  "Com um bairro escolhido, números de 1 a 4 aparecem como “< 5” para não identificar ninguém.";

export function isSuppressed(v: unknown): v is Suppressed {
  return typeof v === "object" && v !== null && (v as { suppressed?: unknown }).suppressed === true;
}

export function fmtCount(v: SmallCount | null | undefined, unit?: "%"): string {
  if (isSuppressed(v)) return SUPPRESSED_LABEL;
  return unit === "%" ? fmtPercent(v) : fmtNumber(v);
}

export function numberOrNull(v: SmallCount | null | undefined): number | null {
  return typeof v === "number" ? v : null;
}

// Ponto suprimido vira lacuna no gráfico: desenhar 0 seria afirmar um número.
export function chartSeries(values: SmallCount[]): (number | null)[] {
  return values.map(numberOrNull);
}

// Barra empilhada e funil calculam proporção sobre o total; com qualquer
// categoria suprimida não há total, então quem chama mostra a lista.
export function whenAllCounted<T extends { count: SmallCount }>(rows: T[]): Array<T & { count: number }> | null {
  return rows.every((r) => typeof r.count === "number") ? (rows as Array<T & { count: number }>) : null;
}
```

```tsx
// src/components/Count.tsx
// Número de painel que pode vir suprimido (módulo 11): "< 5" com a dica.
import type { ReactNode } from "react";
import type { SmallCount } from "../lib/types";
import { SUPPRESSED_HINT, SUPPRESSED_LABEL, fmtCount, isSuppressed, whenAllCounted } from "../lib/smallCount";

export function Count({ value, unit }: { value: SmallCount | null | undefined; unit?: "%" }) {
  if (isSuppressed(value)) return <span title={SUPPRESSED_HINT}>{SUPPRESSED_LABEL}</span>;
  return <>{fmtCount(value, unit)}</>;
}

export function CountList({ items }: { items: Array<{ label: string; count: SmallCount }> }) {
  return (
    <ul aria-label="contagens" style={{ listStyle: "none", margin: 0, padding: 0, display: "flex", flexDirection: "column", gap: 6 }}>
      {items.map((item, i) => (
        <li key={`${item.label}-${i}`} className="mono" style={{ display: "flex", justifyContent: "space-between", gap: 12, fontSize: 12 }}>
          <span style={{ color: "var(--ink2)" }}>{item.label}</span>
          <span><Count value={item.count} /></span>
        </li>
      ))}
    </ul>
  );
}

// Gráfico de proporção quando todas as categorias têm número; lista quando não.
export function SegmentsOrList<T extends { label: string; count: SmallCount }>(
  { segments, render }: { segments: T[]; render(counted: Array<T & { count: number }>): ReactNode }
) {
  const counted = whenAllCounted(segments);
  return counted ? <>{render(counted)}</> : <CountList items={segments} />;
}
```

`src/components/StatTile.tsx`, com as mudanças marcadas:

```tsx
// StatTile — KPI. Valor mono 27px, rótulo 11.5, source badge, sparkline opcional.
// Módulo 11: valor suprimido aparece como "< 5" com a dica, sem unidade nem
// delta. O api já manda delta null com valor suprimido (contrato
// confirmado); esconder aqui é defesa, porque o delta o revelaria.

import { fmtNumber, fmtPercent } from "../lib/format";
import { SUPPRESSED_HINT, SUPPRESSED_LABEL, chartSeries, isSuppressed } from "../lib/smallCount";
import type { SmallCount, Suppressed } from "../lib/types";
import { toneColor, type Tone } from "../theme/tokens";
import { SourceBadge } from "./SourceBadge";
import { Sparkline } from "./Sparkline";

interface Props {
  label: string;
  value: number | string | Suppressed | null | undefined;
  unit?: string;
  delta?: string | null;
  tone?: Tone | string;
  spark?: SmallCount[];
  source?: "live" | "proj";
  hint?: string;
}

export function StatTile({ label, value, unit, delta, tone, spark, source, hint }: Props) {
  const { fg } = toneColor(tone);
  const suppressed = isSuppressed(value);
  const formatted = isSuppressed(value)
    ? SUPPRESSED_LABEL
    : typeof value === "number"
      ? (unit === "%" ? fmtPercent(value, "") : fmtNumber(value))
      : value ?? "—";
```

No JSX:
- o `<span className="mono">` do valor ganha `title={suppressed ? SUPPRESSED_HINT : undefined}`;
- `{unit && (` passa a `{unit && !suppressed && (`;
- `{delta && (` passa a `{delta && !suppressed && (`;
- o sparkline passa a `<Sparkline data={chartSeries(spark)} color={tone ? fg : "var(--accent)"} h={26} />`.

Em `src/components/Sparkline.tsx` e `src/components/BarMini.tsx`, troque `data: number[];` por:

```ts
  // null = ponto suprimido (módulo 11): lacuna, nunca zero.
  data: (number | null)[];
```

O Recharts já desenha `null` como lacuna na área e sem barra no gráfico de barras, e os demais chamadores passam `number[]`, que continua válido.

- [ ] **Step 5: Triagens e Relatórios**

Em `src/modules/Triages.tsx`:
- troque `import { fmtNumber } from "../lib/format";` por:

```tsx
import { Count } from "../components/Count";
import { chartSeries, numberOrNull } from "../lib/smallCount";
```

- depois de `const d = data.data;`:

```tsx
  // Taxa suprimida não tem tom: não há número para julgar.
  const rate = numberOrNull(d.completionRate);
  const rateTone = rate === null ? undefined : rate >= 70 ? "ok" : rate >= 40 ? "warn" : "down";
```

- a StatTile da taxa passa a `<StatTile label="Taxa de conclusão" value={d.completionRate} unit="%" tone={rateTone} source="live" />`;
- `<BarMini data={d.series} h={120} />` passa a `<BarMini data={chartSeries(d.series)} h={120} />`;
- as colunas "Contagem" e "Share":

```tsx
            { label: "Contagem", w: "1fr", align: "right", render: (r) => <span className="mono"><Count value={r.count} /></span> },
            { label: "Share", w: "1fr", align: "right", render: (r) => <span className="mono"><Count value={r.share} unit="%" /></span> }
```

Em `src/modules/Reports.tsx`, importe `import { Count } from "../components/Count";` e troque o conteúdo do `Panel` por:

```tsx
      <Panel title="Relatórios" sub="report_snapshots (metadados)" asOf={data.as_of}>
        {d.reports == null ? (
          // Spec 2026-09-28 §4.3: com bairro e total de 1 a 4, a API não manda as linhas.
          <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
            <EmptyState title="lista oculta" sub="menos de 5 relatórios com este bairro — as linhas não aparecem para não identificar ninguém" />
            <p className="mono" style={{ margin: 0, fontSize: 11, color: "var(--ink3)", textAlign: "center" }}>
              relatórios no período: <Count value={d.total} />
            </p>
          </div>
        ) : (
          <DataTable
            cols={[
              { label: "Data", w: "2fr", render: (r) => <span className="mono">{fmtDateTime(r.createdAt)}</span> },
              { label: "Tier", w: "1fr", render: (r) => <Tag tone={tierTone(r.tier)}>{r.tier ?? "—"}</Tag> },
              { label: "Protocolo", w: "2fr", render: (r) => <span className="mono">{r.protocol}</span> },
              { label: "Expiração", w: "1fr", render: (r) => <Tag tone={r.live ? "ok" : "neutral"}>{r.live ? "ativo" : "expirado"}</Tag> }
            ]}
            rows={d.reports}
            rowKey={(r: ReportRow) => r.id}
            empty="nenhum relatório no período"
          />
        )}
      </Panel>
```

- [ ] **Step 6: Rode, cheque tipos e faça commit**

Run: `npx vitest run src/lib/smallCount.test.ts src/components src/modules/panels.test.tsx src/modules/Reports.test.tsx src/modules/Overview.test.tsx && npx tsc --noEmit`
Expected: PASS. O `Overview.test.tsx` entra porque usa `StatTile` e `Sparkline`.

```bash
/opt/homebrew/bin/git add src/lib/types.ts src/lib/smallCount.ts src/lib/smallCount.test.ts src/components/Count.tsx src/components/StatTile.tsx src/components/StatTile.test.tsx src/components/Sparkline.tsx src/components/BarMini.tsx src/modules/Triages.tsx src/modules/Reports.tsx src/modules/panels.test.tsx src/modules/Reports.test.tsx
/opt/homebrew/bin/git commit -m "feat: render suppressed small counts in triages and reports panels

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 8: Contagem suprimida — Classificação, Conversas e Visão geral

**Files:**
- Modify: `src/lib/types.ts`, `src/lib/classification.ts`, `src/modules/Classification.tsx`, `src/modules/Conversations.tsx`, `src/modules/Overview.tsx`
- Test: `src/modules/Classification.test.tsx`, `src/modules/Conversations.test.tsx`, `src/modules/Overview.test.tsx`

**Interfaces:**
- Consumes: `SmallCount`, `Suppressed`, `Filtered`, `Count`, `SegmentsOrList` e `SUPPRESSED_HINT` (Task 7).
- Produces: `ToneSegment.count: SmallCount`. `OverviewKpi.value/spark`, `ConversationsData`, `ClassificationData` passam a usar `SmallCount`. O tipo novo `SampleTriage` e `ClassificationData.sampleTriages?: SampleTriage[] | null`, onde null significa "oculta". `normalizeClassification` preserva o "oculta".

- [ ] **Step 1: Escreva os testes**

Em `src/modules/Classification.test.tsx`:

```tsx
describe("Classification — filtro de bairro (módulo 11)", () => {
  it("contagem suprimida vira '< 5', a distribuição vira lista e a amostra some", async () => {
    vi.stubGlobal("ResizeObserver", class { observe() {} unobserve() {} disconnect() {} });
    const s = { suppressed: true };
    const filtered = {
      ...classification,
      tiers: [ { key: "alta", label: "alta", count: s, tone: "down" }, { key: "baixa", label: "baixa", count: 0, tone: "info" } ],
      urgent: s,
      urgentTrend: [ 0, s ],
      byProtocol: [ { protocol: "resp · 1", counts: { alta: s, baixa: 0 } } ],
      byMode: [ { mode: "weighted", label: "weighted", count: s, share: s } ],
      sampleTriages: undefined
    };
    vi.stubGlobal("fetch", vi.fn(async () => new Response(
      JSON.stringify({ data: filtered, as_of: "2026-09-27T12:00:00Z" }),
      { status: 200, headers: { "Content-Type": "application/json" } }
    )));
    renderClassification();

    const row = await kpiRow();
    expect(row.getAllByText("< 5")).toHaveLength(2);
    expect(within(screen.getByRole("region", { name: "Distribuição de tier" })).getByRole("list", { name: "contagens" })).toBeTruthy();
    expect(within(screen.getByRole("region", { name: "Tier por protocolo" })).getByText("< 5")).toBeTruthy();
    expect(within(screen.getByRole("region", { name: "Modo de scoring" })).getByRole("list", { name: "contagens" })).toBeTruthy();
    expect(within(screen.getByRole("region", { name: "Amostra de inspeção" })).getByText("amostra oculta")).toBeTruthy();
  });

  it("amostra vazia (não oculta) continua 'sem amostras'", async () => {
    vi.stubGlobal("ResizeObserver", class { observe() {} unobserve() {} disconnect() {} });
    vi.stubGlobal("fetch", vi.fn(async () => new Response(
      JSON.stringify({ data: { ...classification, sampleTriages: [] }, as_of: "2026-09-27T12:00:00Z" }),
      { status: 200, headers: { "Content-Type": "application/json" } }
    )));
    renderClassification();
    expect(await within(await screen.findByRole("region", { name: "Amostra de inspeção" })).findByText("sem amostras")).toBeTruthy();
  });
});
```

Em `src/modules/Conversations.test.tsx` (importe `type Suppressed` de `../lib/types` e `SUPPRESSED_HINT` de `../lib/smallCount`):

```tsx
  it("com bairro: suprimido vira '< 5' e o funil com categoria suprimida vira lista", async () => {
    const s: Suppressed = { suppressed: true };
    stubFetch(200, envelope(data({
      live: s,
      funnel: [
        { key: "greeting", label: "greeting", count: 0, tone: "neutral" },
        { key: "awaiting_consent", label: "awaiting_consent", count: s, tone: "info" }
      ],
      abandonRate: s,
      liveActive: { awaiting: s, inProgress: 7 }
    })));
    renderConversations();

    await screen.findByText("Conversas ativas agora");
    expect(screen.getAllByText("< 5").length).toBeGreaterThanOrEqual(4);
    expect(screen.getAllByTitle(SUPPRESSED_HINT).length).toBeGreaterThanOrEqual(4);
    expect(within(screen.getByRole("region", { name: "Funil FSM" })).getByRole("list", { name: "contagens" })).toBeTruthy();
    expect(screen.getByText("7")).toBeTruthy();
  });
```

(importe `within` de `@testing-library/react` se o arquivo ainda não importa.)

Em `src/modules/Overview.test.tsx`:

```tsx
  it("com bairro: KPI suprimido mostra '< 5' sem o delta; jobs seguem como número", async () => {
    routes({ "/overview": { kpis: [
      { ...kpi("urgent", "Casos urgentes", 0), value: { suppressed: true }, delta: "+2" },
      kpi("failed", "Jobs falhados abertos", 3)
    ] } });
    renderPanel(<Overview />);

    const row = await kpiRow();
    expect(row.getByText("< 5")).toBeTruthy();
    expect(row.queryByText("+2")).toBeNull();
    expect(row.getByText("3")).toBeTruthy();
  });
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/Classification.test.tsx src/modules/Conversations.test.tsx src/modules/Overview.test.tsx`
Expected: FAIL nos casos novos.

- [ ] **Step 3: Tipos**

Em `src/lib/types.ts`:

```ts
export interface ToneSegment {
  key?: string;
  label: string;
  count: SmallCount;
  tone?: string;
}

export interface OverviewKpi {
  id: string;
  label: string;
  value: SmallCount;
  unit: string;
  delta: string | null;
  tone: string;
  spark: SmallCount[];
  source: "live" | "proj";
}
export interface OverviewData extends Filtered { kpis: OverviewKpi[] }
```

```ts
export interface ConversationsData extends Filtered {
  live: SmallCount;
  funnel: ToneSegment[];
  exits: ToneSegment[];
  abandonRate: SmallCount | null;
  avgToCompleteMin: SmallCount | null;
  liveActive: { awaiting: SmallCount; inProgress: SmallCount };
}
```

```ts
export interface SampleTriage {
  id: string;
  tier: string | null;
  priority: number | null;
  urgent: boolean;
  mode: string | null;
  protocol: string;
  at: string | null;
}

export interface ClassificationData extends Filtered {
  tiers: ToneSegment[];
  tierKeys: string[];
  urgent: SmallCount;
  urgentMaxPriority: number;
  urgentTrend: SmallCount[];
  byProtocol: Array<{ protocol: string; counts: Record<string, SmallCount> }>;
  byMode: Array<{ mode: string; label: string; count: SmallCount; share: SmallCount }>;
  // null = oculta pela supressão (módulo 11, contrato confirmado); ausente
  // também é tratado como oculta, por tolerância. [] = nenhuma no período.
  sampleTriages?: SampleTriage[] | null;
}
```

Os tipos `Suppressed`/`SmallCount`/`Filtered` precisam vir **antes** de `ToneSegment` no arquivo. A Task 7 já os pôs logo depois do comentário do topo.

- [ ] **Step 4: Normalizador da Classificação**

Em `src/lib/classification.ts`:

```ts
import type { ClassificationData, SampleTriage, SmallCount } from "./types";

type Legacy = Partial<Omit<ClassificationData, "byProtocol" | "sampleTriages">> & {
  priorityTrue?: number;
  priorityTrend?: number[];
  byProtocol?: Array<{ protocol: string; counts?: Record<string, SmallCount>; [tier: string]: unknown }>;
  sampleTriages?: Array<Omit<SampleTriage, "priority" | "urgent"> & {
    priority: number | boolean | null;
    urgent?: boolean;
  }> | null;
};
```

e, no retorno de `normalizeClassification`, troque o `sampleTriages:` por:

```ts
    // Módulo 11: sem a chave (ou null), a amostra está OCULTA pela
    // supressão — não é o mesmo que vazia.
    sampleTriages: raw.sampleTriages == null ? null : raw.sampleTriages.map((s) => ({
      ...s,
      priority: typeof s.priority === "number" ? s.priority : null,
      urgent: s.urgent ?? s.priority === true
    }))
```

- [ ] **Step 5: Painéis**

`src/modules/Classification.tsx`:
- troque `import { fmtNumber, fmtTime } from "../lib/format";` por `import { fmtTime } from "../lib/format";` e acrescente `import { Count, SegmentsOrList } from "../components/Count";`;
- a "Distribuição de tier":

```tsx
        {d.tiers.every((t) => t.count === 0) ? (
          <EmptyState title="sem triagens classificadas" />
        ) : (
          <SegmentsOrList segments={d.tiers} render={(tiers) => <StackedBar segments={tiers} />} />
        )}
```

- a célula do pivô: `render: (r: ClassificationData["byProtocol"][number]) => <span className="mono"><Count value={r.counts[tier] ?? 0} /></span>`;
- o "Modo de scoring", no ramo com dados:

```tsx
          <SegmentsOrList
            segments={d.byMode.map((m) => ({ label: m.label, count: m.count, tone: m.mode === "weighted" ? "info" : "accent" }))}
            render={(modes) => <StackedBar segments={modes} />}
          />
```

- a "Amostra de inspeção": envolva o `DataTable` existente:

```tsx
        {d.sampleTriages == null ? (
          <EmptyState title="amostra oculta" sub="menos de 5 triagens com este bairro — a amostra não aparece para não identificar ninguém" />
        ) : (
          <DataTable
            cols={[ /* as mesmas colunas de hoje */ ]}
            rows={d.sampleTriages}
            rowKey={(r) => r.id}
            onRowClick={(r) => setTrailOf(r.id)}
            empty="sem amostras"
          />
        )}
```

Mantenha as colunas exatamente como estão. O comentário acima só marca onde elas ficam.

Substitua `src/modules/Conversations.tsx` inteiro:

```tsx
// ConversationsView (§4.2) — FSM e funil.

import { useConversations } from "../hooks/useConversations";
import { Panel } from "../components/Panel";
import { PageHeader } from "../components/PageHeader";
import { KpiGrid } from "../components/KpiGrid";
import { StatTile } from "../components/StatTile";
import { Funnel } from "../components/Funnel";
import { StackedBar } from "../components/StackedBar";
import { Skeleton } from "../components/Skeleton";
import { ErrorState } from "../components/ErrorState";
import { EmptyState } from "../components/EmptyState";
import { Count, SegmentsOrList } from "../components/Count";
import { KpiSkeleton } from "./Overview";

export function Conversations() {
  const { data, isLoading, isError, error, refetch } = useConversations();

  if (isLoading) {
    return (
      <Wrap>
        <KpiGrid><KpiSkeleton /><KpiSkeleton /><KpiSkeleton /></KpiGrid>
        <Panel title="Funil FSM"><Skeleton rows={4} /></Panel>
      </Wrap>
    );
  }
  if (isError) return <Wrap><ErrorState message={(error as Error)?.message || "Erro"} onRetry={() => refetch()} /></Wrap>;
  if (!data) return <Wrap><EmptyState title="sem dados" /></Wrap>;

  const d = data.data;
  return (
    <Wrap>
      <KpiGrid asOf={data.as_of}>
        <StatTile label="Conversas ativas agora" value={d.live} tone="info" source="live" />
        <StatTile label="Taxa de abandono" value={d.abandonRate ?? "—"} unit={d.abandonRate !== null ? "%" : ""} source="live" />
        <StatTile label="Tempo médio até conclusão" value={d.avgToCompleteMin ?? "—"} unit={d.avgToCompleteMin !== null ? "min" : ""} source="live" />
      </KpiGrid>

      <Panel title="Funil FSM" sub="por estado da conversa" asOf={data.as_of}>
        {d.funnel.every((f) => f.count === 0) ? (
          <EmptyState title="sem conversas no período" />
        ) : (
          <SegmentsOrList segments={d.funnel} render={(steps) => <Funnel steps={steps} />} />
        )}
      </Panel>

      <Panel title="Saídas" sub="desfecho das conversas iniciadas no período" asOf={data.as_of}>
        {d.exits.every((e) => e.count === 0) ? (
          <EmptyState title="nenhuma saída registrada" />
        ) : (
          <SegmentsOrList segments={d.exits} render={(exits) => <StackedBar segments={exits} />} />
        )}
      </Panel>

      <Panel title="Vivas agora" sub="awaiting_consent · consented" asOf={data.as_of}>
        <div className="mono" style={{ display: "flex", gap: 24, fontSize: 12 }}>
          <span>aguardando: <strong><Count value={d.liveActive.awaiting} /></strong></span>
          <span>em curso: <strong><Count value={d.liveActive.inProgress} /></strong></span>
          {d.abandonRate !== null && <span>abandono: <strong><Count value={d.abandonRate} unit="%" /></strong></span>}
        </div>
      </Panel>
    </Wrap>
  );
}

function Wrap({ children }: { children: React.ReactNode }) {
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Conversas" sub="conversations · fsm" />
      {children}
    </div>
  );
}
```

Em `src/modules/Overview.tsx` (painel "Ingestão & triagem"), acrescente `import { SegmentsOrList } from "../components/Count";` e troque:

```tsx
      ) : conversations.data && conversations.data.data.funnel.some((f) => f.count > 0) ? (
```

por

```tsx
      ) : conversations.data && conversations.data.data.funnel.some((f) => f.count !== 0) ? (
```

e `<Funnel steps={conversations.data.data.funnel} />` por:

```tsx
            <SegmentsOrList segments={conversations.data.data.funnel} render={(steps) => <Funnel steps={steps} />} />
```

- [ ] **Step 6: Rode a suíte inteira, cheque tipos e faça commit**

Run: `npx vitest run && npx tsc --noEmit`
Expected: PASS, sem cair teste antigo, inclusive o de tolerância ao contrato antigo da Classificação.

```bash
/opt/homebrew/bin/git add src/lib/types.ts src/lib/classification.ts src/modules/Classification.tsx src/modules/Classification.test.tsx src/modules/Conversations.tsx src/modules/Conversations.test.tsx src/modules/Overview.tsx src/modules/Overview.test.tsx
/opt/homebrew/bin/git commit -m "feat: render suppressed small counts in classification, conversations and overview

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 9: Seletor de bairro nos 5 painéis, pela URL

**Files:**
- Create: `src/lib/neighborhoodFilter.ts`, `src/components/NeighborhoodPicker.tsx`
- Modify: `src/lib/api.ts`, `src/components/PageHeader.tsx`, `src/hooks/useOverview.ts`, `src/hooks/useClassification.ts`, `src/hooks/useTriages.ts`, `src/hooks/useReports.ts`, `src/hooks/useConversations.ts`, `src/modules/Overview.tsx`, `src/modules/Classification.tsx`, `src/modules/Triages.tsx`, `src/modules/Reports.tsx`, `src/modules/Conversations.tsx`, `src/modules/Territory.tsx`, `README.md`
- Modify (testes antigos que contavam chamadas de `fetch`): `src/modules/panels.test.tsx`, `src/modules/Reports.test.tsx`, `src/modules/Conversations.test.tsx`
- Test: `src/lib/neighborhoodFilter.test.ts`, `src/components/NeighborhoodPicker.test.tsx`, `src/modules/neighborhoodPanels.test.tsx`

**Interfaces:**
- Consumes: `sortByName` (Task 1), `SUPPRESSED_HINT` (Task 7), `GET /admin/api/neighborhoods` do plano do api (decisão do usuário, 2026-09-28): `{ neighborhoods: [{ id, name, active }] }`, só leitura, para **todo papel** que lê os painéis.
- Produces:
  - em `api.ts`: `PanelNeighborhood { id; name; active }` e `listPanelNeighborhoods(): Promise<PanelNeighborhood[]>`;
  - em `neighborhoodFilter.ts`: `type NeighborhoodFilter = string | null` (uuid, `"none"` ou null para "Todos"), `NONE = "none"`, `PANEL_NEIGHBORHOODS_KEY = ["panelNeighborhoods"]`, `readNeighborhoodParam(search?)`, `writeNeighborhoodParam(value)`, `useNeighborhoodParam()`, `useNeighborhoodFilter()` e `neighborhoodParams(filter) → { neighborhood_id?: string }`;
  - `<NeighborhoodPicker />`;
  - `PageHeader` ganha `right?: ReactNode`.
- Chaves de consulta: `[ "<painel>", period, municipalityId, neighborhood ]`, sem `userId`, porque o cache já é por usuário.

- [ ] **Step 1: Escreva os testes da regra**

```ts
// src/lib/neighborhoodFilter.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import { act, renderHook } from "@testing-library/react";
import { listPanelNeighborhoods } from "./api";
import {
  NONE, neighborhoodParams, readNeighborhoodParam, useNeighborhoodParam, writeNeighborhoodParam
} from "./neighborhoodFilter";

const CENTRO = "11111111-1111-4111-8111-111111111111";

afterEach(() => { window.history.replaceState(null, "", "/"); vi.unstubAllGlobals(); });

describe("readNeighborhoodParam", () => {
  it("aceita uuid e none", () => {
    expect(readNeighborhoodParam(`?bairro=${CENTRO}`)).toBe(CENTRO);
    expect(readNeighborhoodParam("?bairro=none")).toBe(NONE);
  });
  it("lixo ou ausente vira Todos (null), nunca um 422", () => {
    expect(readNeighborhoodParam("?bairro=centro")).toBeNull();
    expect(readNeighborhoodParam("?bairro=")).toBeNull();
    expect(readNeighborhoodParam("")).toBeNull();
  });
});

describe("writeNeighborhoodParam", () => {
  it("grava e apaga só o bairro, sem mexer no resto da URL", () => {
    window.history.replaceState(null, "", "/dashboard/?x=1");
    writeNeighborhoodParam(CENTRO);
    expect(window.location.pathname).toBe("/dashboard/");
    expect(new URLSearchParams(window.location.search).get("x")).toBe("1");
    expect(new URLSearchParams(window.location.search).get("bairro")).toBe(CENTRO);
    writeNeighborhoodParam(null);
    expect(window.location.search).toBe("?x=1");
  });

  it("avisa quem lê pelo hook", () => {
    const { result } = renderHook(() => useNeighborhoodParam());
    expect(result.current).toBeNull();
    act(() => writeNeighborhoodParam(NONE));
    expect(result.current).toBe(NONE);
  });
});

describe("neighborhoodParams", () => {
  it("Todos não manda o parâmetro", () => {
    expect(neighborhoodParams(null)).toEqual({ neighborhood_id: undefined });
    expect(neighborhoodParams(NONE)).toEqual({ neighborhood_id: "none" });
  });
});

describe("listPanelNeighborhoods", () => {
  const stub = (body: unknown) => {
    const fn = vi.fn(async (_input: RequestInfo | URL) =>
      new Response(JSON.stringify(body), { status: 200, headers: { "Content-Type": "application/json" } }));
    vi.stubGlobal("fetch", fn);
    return fn;
  };

  it("lê GET /admin/api/neighborhoods no formato combinado", async () => {
    const fn = stub({ neighborhoods: [ { id: CENTRO, name: "Centro", active: true } ] });
    expect(await listPanelNeighborhoods()).toEqual([ { id: CENTRO, name: "Centro", active: true } ]);
    expect(new URL(String(fn.mock.calls[0][0])).pathname).toBe("/admin/api/neighborhoods");
  });

  it("tolera o envelope { data, as_of } dos outros painéis e resposta sem a chave", async () => {
    stub({ data: { neighborhoods: [ { id: CENTRO, name: "Centro", active: true } ] }, as_of: "x" });
    expect(await listPanelNeighborhoods()).toHaveLength(1);
    stub({ data: { live: 0 } });
    expect(await listPanelNeighborhoods()).toEqual([]);
  });
});
```

- [ ] **Step 2: Escreva os testes do seletor e dos painéis**

```tsx
// src/components/NeighborhoodPicker.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, listPanelNeighborhoods: vi.fn() };
});

import * as api from "../lib/api";
import { SUPPRESSED_HINT } from "../lib/smallCount";
import { NeighborhoodPicker } from "./NeighborhoodPicker";

const CENTRO = "11111111-1111-4111-8111-111111111111";
const BATEL = "22222222-2222-4222-8222-222222222222";
const AGUA = "33333333-3333-4333-8333-333333333333";
const SUMIU = "99999999-9999-4999-8999-999999999999";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function renderIt() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  render(<QueryClientProvider client={client}><NeighborhoodPicker /></QueryClientProvider>);
}
const picker = () => screen.getByLabelText("Bairro") as HTMLSelectElement;
const ready = () => waitFor(() => expect(picker().disabled).toBe(false));

beforeEach(() => {
  mocked(api.listPanelNeighborhoods).mockReset().mockResolvedValue([
    { id: CENTRO, name: "Centro", active: true },
    { id: BATEL, name: "Batel", active: false },
    { id: AGUA, name: "Água Verde", active: true }
  ]);
});
afterEach(() => { cleanup(); window.history.replaceState(null, "", "/"); });

describe("NeighborhoodPicker", () => {
  it("Todos, Sem bairro e os bairros por nome, com os inativos marcados", async () => {
    renderIt();
    await ready();
    expect(Array.from(picker().options).map((o) => o.textContent)).toEqual([
      "Todos", "Sem bairro", "Água Verde", "Batel (inativo)", "Centro"
    ]);
  });

  it("escolher grava ?bairro= e mostra a regra do '< 5'; Todos apaga", async () => {
    renderIt();
    await ready();
    fireEvent.change(picker(), { target: { value: CENTRO } });
    expect(new URLSearchParams(window.location.search).get("bairro")).toBe(CENTRO);
    expect(screen.getByText(SUPPRESSED_HINT)).toBeTruthy();
    fireEvent.change(picker(), { target: { value: "" } });
    expect(window.location.search).toBe("");
    expect(screen.queryByText(SUPPRESSED_HINT)).toBeNull();
  });

  it("?bairro=none vem selecionado como Sem bairro", async () => {
    window.history.replaceState(null, "", "/dashboard/?bairro=none");
    renderIt();
    await ready();
    expect(picker().value).toBe("none");
  });

  it("bairro desconhecido na URL volta para Todos", async () => {
    window.history.replaceState(null, "", `/dashboard/?bairro=${SUMIU}`);
    renderIt();
    await waitFor(() => expect(window.location.search).toBe(""));
    expect(picker().value).toBe("");
  });

  it("lista que não carrega: o seletor continua com Todos e Sem bairro, e a URL não é apagada", async () => {
    mocked(api.listPanelNeighborhoods).mockRejectedValue(new api.ApiError(500, "", "x"));
    window.history.replaceState(null, "", `/dashboard/?bairro=${CENTRO}`);
    renderIt();
    expect(await screen.findByText("não foi possível carregar os bairros")).toBeTruthy();
    expect(Array.from(picker().options).map((o) => o.textContent)).toEqual([ "Todos", "Sem bairro" ]);
    expect(new URLSearchParams(window.location.search).get("bairro")).toBe(CENTRO);
  });
});
```

```tsx
// src/modules/neighborhoodPanels.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import type { ReactElement } from "react";
import { renderPanel, stubResizeObserver, stubRoutes } from "../test/panelHarness";
import { Overview } from "./Overview";
import { Classification } from "./Classification";
import { Triages } from "./Triages";
import { Reports } from "./Reports";
import { Conversations } from "./Conversations";

// Módulo 11 (spec 2026-09-28 §5; decisão do usuário 2026-09-28): os 5
// painéis com cidadão levam o bairro da URL como neighborhood_id, para
// qualquer papel, e a chave de consulta muda com ele. A lista do seletor
// vem de GET /admin/api/neighborhoods.

const CENTRO = "11111111-1111-4111-8111-111111111111";
const SUMIU = "99999999-9999-4999-8999-999999999999";

const ROUTES = {
  "/neighborhoods": { neighborhoods: [ { id: CENTRO, name: "Centro", active: true } ] },
  "/overview": { kpis: [ { id: "done", label: "Triagens concluídas", value: 9, unit: "", delta: null, tone: "ok", spark: [], source: "live" } ] },
  "/classification": { tiers: [], tierKeys: [], urgent: 0, urgentMaxPriority: 1, urgentTrend: [], byProtocol: [], byMode: [], sampleTriages: [] },
  "/triages": { series: [], started: 0, completed: 0, completionRate: 0, byProtocol: [] },
  "/reports": { reports: [], total: 0 },
  "/conversations": { live: 0, funnel: [], exits: [], abandonRate: null, avgToCompleteMin: null, liveActive: { awaiting: 0, inProgress: 0 } },
  "/ingestion": { inboundSeries: [], inboundTotal: 0, ack: [], purge: { pending: 0, oldestH: 0, ttlH: 2160, overTtl: false } },
  "/queues": { queues: [], oldestPendingS: 0, failedExecutions: [], recurring: [] },
  "/events": { total: 0, retentionMonths: 12, replayAnchor: null, byType: [], stream: [], filters: [] },
  "/health": { projections: [], recurring: [], driftOverall: null }
};

function neighborhoodIds(fetchMock: ReturnType<typeof stubRoutes>, path: string): (string | null)[] {
  return fetchMock.mock.calls
    .map((c) => new URL(String((c as unknown[])[0])))
    .filter((u) => u.pathname === `/admin/api${path}`)
    .map((u) => u.searchParams.get("neighborhood_id"));
}

beforeEach(() => stubResizeObserver());
afterEach(() => { cleanup(); vi.unstubAllGlobals(); window.history.replaceState(null, "", "/"); });

const PANELS: Array<[ string, () => ReactElement ]> = [
  [ "/overview", () => <Overview /> ],
  [ "/classification", () => <Classification /> ],
  [ "/triages", () => <Triages /> ],
  [ "/reports", () => <Reports /> ],
  [ "/conversations", () => <Conversations /> ]
];

describe("filtro de bairro nos painéis", () => {
  it.each(PANELS)("%s leva o bairro da URL como neighborhood_id e mostra o seletor", async (path, element) => {
    window.history.replaceState(null, "", `/dashboard/?bairro=${CENTRO}`);
    const fetchMock = stubRoutes(ROUTES);
    renderPanel(element());
    await waitFor(() => expect(neighborhoodIds(fetchMock, path)).toContain(CENTRO));
    expect(await screen.findByLabelText("Bairro")).toBeTruthy();
  });

  it("escolher 'Sem bairro' refaz a busca com neighborhood_id=none", async () => {
    const fetchMock = stubRoutes(ROUTES);
    renderPanel(<Triages />);
    const picker = await screen.findByLabelText("Bairro") as HTMLSelectElement;
    await waitFor(() => expect(picker.disabled).toBe(false));
    fireEvent.change(picker, { target: { value: "none" } });
    await waitFor(() => expect(neighborhoodIds(fetchMock, "/triages")).toContain("none"));
    expect(new URLSearchParams(window.location.search).get("bairro")).toBe("none");
  });

  it("Visão geral: filas, saúde, eventos e ingestão não levam o bairro", async () => {
    window.history.replaceState(null, "", `/dashboard/?bairro=${CENTRO}`);
    const fetchMock = stubRoutes(ROUTES);
    renderPanel(<Overview />);
    await waitFor(() => expect(neighborhoodIds(fetchMock, "/overview")).toContain(CENTRO));
    for (const path of [ "/queues", "/health", "/events", "/ingestion", "/neighborhoods" ]) {
      expect(neighborhoodIds(fetchMock, path).every((id) => id === null)).toBe(true);
    }
  });

  it("bairro desconhecido na URL volta para Todos", async () => {
    window.history.replaceState(null, "", `/dashboard/?bairro=${SUMIU}`);
    const fetchMock = stubRoutes(ROUTES);
    renderPanel(<Triages />);
    await waitFor(() => expect(window.location.search).toBe(""));
    await waitFor(() => expect(neighborhoodIds(fetchMock, "/triages").at(-1)).toBeNull());
  });
});
```

- [ ] **Step 3: Ajuste os testes antigos que contavam `fetch`**

O seletor agora faz a sua própria chamada (`/admin/api/neighborhoods`) em todo painel. Por isso deixa de valer "a primeira chamada é a do painel" e "duas chamadas no total".
- `src/modules/panels.test.tsx`, Triages: troque `new URL(String((fetchMock.mock.calls[0] as unknown[])[0]))` por:

```tsx
    const url = fetchMock.mock.calls.map((c) => new URL(String((c as unknown[])[0]))).find((u) => u.pathname === "/admin/api/triages")!;
    expect(url.searchParams.get("period")).toBe("7d");
```

- `src/modules/Reports.test.tsx` e `src/modules/Conversations.test.tsx`: acrescente no topo o auxiliar

```tsx
const callsTo = (fetchMock: ReturnType<typeof vi.fn>, pathname: string) =>
  fetchMock.mock.calls.map((c) => new URL(String((c as unknown[])[0]))).filter((u) => u.pathname === pathname);
```

  e troque, nos testes "busca GET /admin/api/…", a leitura de `calls[0]` por `const url = callsTo(fetchMock, "/admin/api/reports")[0];` (ou `/admin/api/conversations`), mantendo as expectativas de `pathname` e `period`. Nos testes "500 … 'tentar novamente' refaz a busca", troque `expect(fetchMock).toHaveBeenCalledTimes(2)` por `expect(callsTo(fetchMock, "/admin/api/reports")).toHaveLength(2)` (ou `/admin/api/conversations`).

  O `stubFetch` desses arquivos devolve o mesmo corpo para qualquer URL. `listPanelNeighborhoods` lê esse corpo como lista vazia (Step 4), então o seletor aparece só com "Todos" e "Sem bairro" e não quebra nada.

- [ ] **Step 4: Rode e veja falhar**

Run: `npx vitest run src/lib/neighborhoodFilter.test.ts src/components/NeighborhoodPicker.test.tsx src/modules/neighborhoodPanels.test.tsx`
Expected: FAIL (módulos inexistentes).

- [ ] **Step 5: Cliente da lista e regra do filtro**

Em `src/lib/api.ts`, logo depois de `adminFetch`:

```ts
// Lista de bairros para o seletor dos painéis (módulo 11; decisão do usuário
// 2026-09-28): GET /admin/api/neighborhoods, só leitura, para todo papel que
// lê os painéis (não é /territory, que é só do municipal_admin). O contrato
// combinado é { neighborhoods: [...] }; o envelope { data } dos outros
// painéis também é aceito (o api confirmou que responde SEM envelope, mas
// tolerar custa uma linha), e sem a chave a lista é vazia.
export interface PanelNeighborhood { id: string; name: string; active: boolean }

export async function listPanelNeighborhoods(): Promise<PanelNeighborhood[]> {
  const body = await jsonFetch<{ neighborhoods?: PanelNeighborhood[]; data?: { neighborhoods?: PanelNeighborhood[] } }>(
    new URL(`${BASE}/neighborhoods`, window.location.origin).toString());
  return body?.data?.neighborhoods ?? body?.neighborhoods ?? [];
}
```

```ts
// src/lib/neighborhoodFilter.ts
// Filtro de bairro dos painéis com cidadão (módulo 11; spec 2026-09-28 §5).
// Vale para todo papel que lê os painéis. Mora na URL (?bairro=<uuid> ou
// ?bairro=none) para sobreviver a recarregar a página e valer nos cinco
// painéis; "Todos" = sem o parâmetro. O cache do TanStack já é por usuário
// (useSessionQueryClient): a chave leva o bairro, nunca o usuário.
import { useSyncExternalStore } from "react";

export type NeighborhoodFilter = string | null;
export const NONE = "none";
export const PANEL_NEIGHBORHOODS_KEY = [ "panelNeighborhoods" ] as const;

const PARAM = "bairro";
const EVENT = "rotasaude:bairro";
const UUID = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

// Lixo na URL vira "Todos": mandar para a API seria um 422 invalid_neighborhood.
export function readNeighborhoodParam(search: string = window.location.search): NeighborhoodFilter {
  const raw = new URLSearchParams(search).get(PARAM)?.trim() ?? "";
  return raw === NONE || UUID.test(raw) ? raw : null;
}

// replaceState: trocar o bairro não é navegação; o "voltar" do navegador
// continua saindo da página.
export function writeNeighborhoodParam(value: NeighborhoodFilter): void {
  const url = new URL(window.location.href);
  if (value) url.searchParams.set(PARAM, value);
  else url.searchParams.delete(PARAM);
  window.history.replaceState(window.history.state, "", url.toString());
  window.dispatchEvent(new Event(EVENT));
}

function subscribe(onChange: () => void): () => void {
  window.addEventListener(EVENT, onChange);
  window.addEventListener("popstate", onChange);
  return () => {
    window.removeEventListener(EVENT, onChange);
    window.removeEventListener("popstate", onChange);
  };
}

export function useNeighborhoodParam(): NeighborhoodFilter {
  return useSyncExternalStore(subscribe, () => readNeighborhoodParam());
}

// Nome próprio para o que os hooks consomem: se um dia o filtro depender de
// mais que a URL, muda só aqui.
export function useNeighborhoodFilter(): NeighborhoodFilter {
  return useNeighborhoodParam();
}

export function neighborhoodParams(filter: NeighborhoodFilter): Record<string, string | undefined> {
  return { neighborhood_id: filter ?? undefined };
}
```

Em `src/modules/Territory.tsx`, o `refresh()` passa a invalidar também a lista do seletor, para um bairro criado ou renomeado aparecer nos painéis sem esperar o `staleTime`:

```tsx
  function refresh() {
    void queryClient.invalidateQueries({ queryKey: NEIGHBORHOODS_KEY });
    void queryClient.invalidateQueries({ queryKey: PANEL_NEIGHBORHOODS_KEY });
  }
```

com `import { PANEL_NEIGHBORHOODS_KEY } from "../lib/neighborhoodFilter";`.

- [ ] **Step 6: Seletor e cabeçalho**

```tsx
// src/components/NeighborhoodPicker.tsx
import { useEffect } from "react";
import { useQuery } from "@tanstack/react-query";
import { listPanelNeighborhoods } from "../lib/api";
import { NONE, PANEL_NEIGHBORHOODS_KEY, useNeighborhoodParam, writeNeighborhoodParam } from "../lib/neighborhoodFilter";
import { sortByName } from "../lib/territory";
import { SUPPRESSED_HINT } from "../lib/smallCount";
import { inputStyle } from "./formStyles";

// Seletor de bairro dos painéis com cidadão (módulo 11; spec 2026-09-28 §5),
// para todo papel que lê os painéis. Inativos continuam na lista (histórico e
// filtro, ADR 0023), marcados.
export function NeighborhoodPicker() {
  const value = useNeighborhoodParam();
  const list = useQuery({ queryKey: PANEL_NEIGHBORHOODS_KEY, queryFn: listPanelNeighborhoods });

  // Link velho, bairro de outra cidade: sem isto o painel ficaria no 422.
  // Só com a lista carregada — lista que falhou não prova que o bairro sumiu.
  const unknown = !!value && value !== NONE && list.isSuccess && !list.data.some((n) => n.id === value);
  useEffect(() => { if (unknown) writeNeighborhoodParam(null); }, [ unknown ]);

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 4, minWidth: 220, maxWidth: 320 }}>
      <label style={{ display: "flex", flexDirection: "column", gap: 2, fontSize: 12, color: "var(--ink2)" }}>
        Bairro
        <select
          value={unknown ? "" : value ?? ""}
          disabled={list.isPending}
          onChange={(e) => writeNeighborhoodParam(e.target.value || null)}
          style={inputStyle}
        >
          <option value="">Todos</option>
          <option value={NONE}>Sem bairro</option>
          {sortByName(list.data ?? []).map((n) => (
            <option key={n.id} value={n.id}>{n.active ? n.name : `${n.name} (inativo)`}</option>
          ))}
        </select>
      </label>
      {list.isError && <span style={{ fontSize: 11, color: "var(--down)" }}>não foi possível carregar os bairros</span>}
      {value && !unknown && <span style={{ fontSize: 11, color: "var(--ink3)" }}>{SUPPRESSED_HINT}</span>}
    </div>
  );
}
```

O aviso de falha não usa `role="alert"`: o seletor aparece em todo painel, e um alerta ali confundiria os testes e leitores de tela que procuram o erro do próprio painel.

`src/components/PageHeader.tsx`:

```tsx
// Cabeçalho de página: título 20px + slug mono + slot à direita (seletor de bairro).
import type { ReactNode } from "react";

interface Props {
  title: string;
  sub: string;
  right?: ReactNode;
}

export function PageHeader({ title, sub, right }: Props) {
  return (
    <div style={{ display: "flex", alignItems: "baseline", gap: 10, marginBottom: 4, flexWrap: "wrap" }}>
      <h1 style={{ margin: 0, fontSize: 20, fontWeight: 700, color: "var(--ink)" }}>{title}</h1>
      <span
        className="mono"
        style={{
          fontSize: 10.5,
          color: "var(--ink3)",
          textTransform: "uppercase",
          letterSpacing: 0.6
        }}
      >
        {sub}
      </span>
      {right && <div style={{ marginLeft: "auto" }}>{right}</div>}
    </div>
  );
}
```

- [ ] **Step 7: Hooks com o bairro na chave e no pedido**

`src/hooks/useTriages.ts`:

```ts
import { useQuery } from "@tanstack/react-query";
import { adminFetch } from "../lib/api";
import { scopeParams, useScope } from "../lib/scope";
import { neighborhoodParams, useNeighborhoodFilter } from "../lib/neighborhoodFilter";
import type { TriagesData } from "../lib/types";

export function useTriages() {
  const scope = useScope();
  const neighborhood = useNeighborhoodFilter();
  return useQuery({
    queryKey: [ "triages", scope.period, scope.municipalityId, neighborhood ],
    queryFn: () => adminFetch<TriagesData>("/triages", { ...scopeParams(scope), ...neighborhoodParams(neighborhood) }),
    staleTime: 30_000
  });
}
```

Os outros quatro seguem a mesma forma. Muda só o nome, o caminho e o tipo:

```ts
// src/hooks/useOverview.ts
import { useQuery } from "@tanstack/react-query";
import { adminFetch } from "../lib/api";
import { scopeParams, useScope } from "../lib/scope";
import { neighborhoodParams, useNeighborhoodFilter } from "../lib/neighborhoodFilter";
import type { OverviewData } from "../lib/types";

export function useOverview() {
  const scope = useScope();
  const neighborhood = useNeighborhoodFilter();
  return useQuery({
    queryKey: [ "overview", scope.period, scope.municipalityId, neighborhood ],
    queryFn: () => adminFetch<OverviewData>("/overview", { ...scopeParams(scope), ...neighborhoodParams(neighborhood) }),
    staleTime: 30_000
  });
}
```

```ts
// src/hooks/useClassification.ts
import { useQuery } from "@tanstack/react-query";
import { adminFetch } from "../lib/api";
import { scopeParams, useScope } from "../lib/scope";
import { neighborhoodParams, useNeighborhoodFilter } from "../lib/neighborhoodFilter";
import type { ClassificationData } from "../lib/types";

export function useClassification() {
  const scope = useScope();
  const neighborhood = useNeighborhoodFilter();
  return useQuery({
    queryKey: [ "classification", scope.period, scope.municipalityId, neighborhood ],
    queryFn: () => adminFetch<ClassificationData>("/classification", { ...scopeParams(scope), ...neighborhoodParams(neighborhood) }),
    staleTime: 30_000
  });
}
```

```ts
// src/hooks/useReports.ts
import { useQuery } from "@tanstack/react-query";
import { adminFetch } from "../lib/api";
import { scopeParams, useScope } from "../lib/scope";
import { neighborhoodParams, useNeighborhoodFilter } from "../lib/neighborhoodFilter";
import type { ReportsData } from "../lib/types";

export function useReports() {
  const scope = useScope();
  const neighborhood = useNeighborhoodFilter();
  return useQuery({
    queryKey: [ "reports", scope.period, scope.municipalityId, neighborhood ],
    queryFn: () => adminFetch<ReportsData>("/reports", { ...scopeParams(scope), ...neighborhoodParams(neighborhood) }),
    staleTime: 30_000
  });
}
```

```ts
// src/hooks/useConversations.ts
import { useQuery } from "@tanstack/react-query";
import { adminFetch } from "../lib/api";
import { scopeParams, useScope } from "../lib/scope";
import { neighborhoodParams, useNeighborhoodFilter } from "../lib/neighborhoodFilter";
import type { ConversationsData } from "../lib/types";

export function useConversations() {
  const scope = useScope();
  const neighborhood = useNeighborhoodFilter();
  return useQuery({
    queryKey: [ "conversations", scope.period, scope.municipalityId, neighborhood ],
    queryFn: () => adminFetch<ConversationsData>("/conversations", { ...scopeParams(scope), ...neighborhoodParams(neighborhood) }),
    staleTime: 30_000
  });
}
```

`useConversations` também alimenta o funil do painel "Ingestão & triagem" da Visão geral, que passa a seguir o mesmo bairro. `useIngestion`, `useQueues`, `useHealth` e `useEvents` **não** mudam: não têm cidadão (spec §4.3).

- [ ] **Step 8: O seletor nas cinco telas**

Importe `import { NeighborhoodPicker } from "../components/NeighborhoodPicker";` em cada uma e passe `right`:
- `src/modules/Overview.tsx`: `<PageHeader title="Visão geral" sub="overview" right={<NeighborhoodPicker />} />`;
- `src/modules/Classification.tsx` (no `Wrap`): `<PageHeader title="Classificação" sub="classification · scoring" right={<NeighborhoodPicker />} />`;
- `src/modules/Triages.tsx` (no `Wrap`): `<PageHeader title="Triagens" sub="triages" right={<NeighborhoodPicker />} />`;
- `src/modules/Reports.tsx` (no `Wrap`): `<PageHeader title="Relatórios" sub="reports" right={<NeighborhoodPicker />} />`;
- `src/modules/Conversations.tsx` (no `Wrap`): `<PageHeader title="Conversas" sub="conversations · fsm" right={<NeighborhoodPicker />} />`.

O seletor fica no `Wrap`, e o `Wrap` também envolve os estados de carregando e de erro. Um 422 ou 500 com bairro escolhido deixa trocar para "Todos" sem recarregar a página.

- [ ] **Step 9: README**

Em `README.md`, logo depois da frase "O que a API recusaria (403) fica fora do menu. ...":

```markdown
**Filtro de bairro** (módulo 11, ADR 0023): Visão geral, Conversas, Triagens,
Classificação e Relatórios têm o seletor "Bairro", para todo papel que lê os
painéis (a lista vem de `/admin/api/neighborhoods`). A escolha fica na URL
(`?bairro=<id>` ou `?bairro=none`). Com bairro escolhido, a API troca contagem
de 1 a 4 por `{ suppressed: true }`, e a tela mostra "< 5".
```

- [ ] **Step 10: Rode a suíte inteira, cheque tipos e faça commit**

Run: `npx vitest run && npx tsc --noEmit`
Expected: PASS. Os testes antigos dos painéis passam a ver o seletor: ele busca `/admin/api/neighborhoods`, que o `stubRoutes` responde com 404 (lista vazia, aviso de falha) e o `stubFetch` com o corpo do painel (lista vazia). Nenhuma expectativa antiga de conteúdo muda; só as de contagem de chamadas, ajustadas no Step 3.

```bash
/opt/homebrew/bin/git add src/lib/api.ts src/lib/neighborhoodFilter.ts src/lib/neighborhoodFilter.test.ts src/components/NeighborhoodPicker.tsx src/components/NeighborhoodPicker.test.tsx src/components/PageHeader.tsx src/hooks/useOverview.ts src/hooks/useClassification.ts src/hooks/useTriages.ts src/hooks/useReports.ts src/hooks/useConversations.ts src/modules/Overview.tsx src/modules/Classification.tsx src/modules/Triages.tsx src/modules/Reports.tsx src/modules/Conversations.tsx src/modules/Territory.tsx src/modules/neighborhoodPanels.test.tsx src/modules/panels.test.tsx src/modules/Reports.test.tsx src/modules/Conversations.test.tsx README.md
/opt/homebrew/bin/git commit -m "feat: filter citizen panels by neighborhood through the URL

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 10: Conferência do `admin`, suíte, revisão e prova no navegador

- [ ] **Step 1: O console `admin` não compartilha tipos com o dashboard**

Run (da raiz do monorepo):

```bash
grep -rn "dashboard/src\|@rota-saude/dashboard" apps/admin/src apps/admin/package.json apps/admin/tsconfig.json
```

Expected: nenhuma linha. O `admin` tem os próprios tipos (`apps/admin/src/lib/types.ts`) e não envia `neighborhood_id`, então nunca recebe `{ suppressed: true }` (spec §4.3). O teste de contrato dele fica no api. Se aparecer alguma linha, rode também `cd apps/admin && npx tsc --noEmit && npx vitest run` e trate a quebra antes de seguir. Este plano não muda nada no `admin`.

- [ ] **Step 2: Suíte, tipos e build**

Run: `cd apps/dashboard/.claude/mod11 && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde. A CI (`.github/workflows/ci.yml`) roda os mesmos três. `dist/` está no `.gitignore`.

Confira também que nenhum arquivo do plano ficou fora de commit:

```bash
/opt/homebrew/bin/git status --short
```

Expected: vazio. O symlink `node_modules` não aparece porque é ignorado. Se aparecer, **não** o adicione.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§4 e §5) e o ADR 0023, com atenção a:
- nenhum caminho trata `{ suppressed: true }` como 0 (gráfico, tom, soma, `every(=== 0)`);
- o ViaCEP é chamado só do navegador, sem cookie (`credentials: "omit"`), com timeout de 5 s, e toda falha cai no aviso;
- `reference_unit_ids` só vira `referral_unit_id` no desfecho "encaminhado";
- a chave de consulta dos 5 painéis leva o bairro e não leva `userId`;
- nenhum `git add -A` no histórico da branch (`git log --stat origin/main..HEAD` sem `node_modules`).

- [ ] **Step 4: Prova no navegador**

Com o api da branch do módulo 11 rodando a partir do worktree dele em `:3031` (semente de território aplicada em Curitiba), rode o Vite do worktree apontando para ele:

```bash
cd apps/dashboard/.claude/mod11 && VITE_API_PROXY_TARGET=http://localhost:3031 npx vite --port 5179 --host 0.0.0.0
```

Abra `http://curitiba.localhost:5179/dashboard/`. O usuário faz o login como `admin@curitiba.demo`. Não digite senha nem TOTP. Confira com screenshot:
- Cidade → Território: a lista da semente, a busca "sao" achando bairro com acento, criar "Bairro Teste", renomear, desativar e reativar, e a cobertura com caixas;
- Atendimento → Unidades → Editar: com o CEP `80010000` vêm o logradouro e "bairro segundo o CEP: Centro", com Centro pré-selecionado. Um CEP inexistente (`99999999`) mostra "não foi possível consultar o CEP";
- Painéis: em Triagens, o seletor "Bairro" grava `?bairro=` na URL, recarregar a página mantém a escolha e um valor pequeno aparece como "< 5" com a dica;
- depois que o wpda (plano próprio) gerar uma triagem com bairro coberto, um profissional encerra com "Encaminhado" e o destino vem pré-escolhido com "· referência".

- [ ] **Step 5:** **Pare.** O merge do dashboard só vem depois do merge do api, e só com autorização explícita do usuário.

---

## Self-review

- **Cobertura da spec:**
  - §5 Território: Tasks 3 e 4.
  - Formulário de unidade com CEP: Tasks 2 e 5.
  - CSP: Task 2, Step 5.
  - Desfecho "encaminhado": Task 6.
  - Seletor nos 5 painéis, para todo papel, com a lista de `/admin/api/neighborhoods` e `?bairro=`: Task 9.
  - Supressão, "< 5", listas de amostra e tipos: Tasks 7 e 8.
  - Cache por usuário sem `userId` na chave: Task 9 e o revisor da Task 10.
  - §7.2 dashboard: tela Território (Task 4); ViaCEP simulado com sucesso, `{erro:true}`, rede e timeout (Tasks 2 e 5); pré-seleção (Task 6); seletor e "< 5" (Tasks 7 a 9).
  - Proxy: Task 1.
  - `admin`: Task 10, Step 1.
- **Placeholders:** o único código "a preencher" é o `Territory` mínimo da Task 3, substituído na Task 4. O comentário `/* as mesmas colunas de hoje */` da Task 8 marca onde ficam colunas que não mudam.
- **Consistência de nomes:**
  - `NEIGHBORHOODS_KEY`, `listNeighborhoods`, `Neighborhood` e `sortByName` aparecem iguais nas Tasks 1, 4, 5 e 9.
  - `addressPayload`, `addressFieldsFrom`, `EMPTY_ADDRESS` e `EMPTY_ADDRESS_FIELDS` estão definidos na Task 2 e usados na Task 5.
  - `SmallCount`, `Suppressed`, `SUPPRESSED_HINT`, `Count` e `SegmentsOrList` estão definidos na Task 7 e usados nas Tasks 8 e 9.
  - `splitReferenceUnits` está definido e usado na Task 6.
  - `useNeighborhoodFilter`, `neighborhoodParams`, `listPanelNeighborhoods` e `PANEL_NEIGHBORHOODS_KEY` estão definidos e usados na Task 9.
  - O 3º argumento de `splitReferenceUnits` (a unidade atual, `unit.id`) aparece nos três lugares: teste da lib, teste da fila e `ClosePanel`.
- **Review Focus:** os cinco casos têm teste na task dona: 1 e 2 na Task 5, 3 na Task 9, 4 na Task 6 e 5 na Task 4.
