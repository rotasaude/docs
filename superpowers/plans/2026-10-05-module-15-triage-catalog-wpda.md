# Módulo 15 — Triagem direcionada por perfil (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** No canal web do cidadão, depois de "Para quem é esta triagem?", a pessoa sem perfil informa data de nascimento e sexo (identidade de gênero opcional), vê o catálogo de triagens que o api oferece para ela ("Em andamento", "Sugeridas para você", "Disponíveis", "Feitas recentemente"), começa a triagem escolhida, recebe "Recomendamos também" no resultado e corrige o próprio perfil em "Meu perfil" enquanto ele não foi conferido no posto.

**Architecture:** Tipos e chamadas novas em `src/lib/citizenApi.ts`, com os campos novos normalizados na borda (ausente vira `null` ou `[]`), como já é feito para `neighborhood`, `attendance` e `reference_units`. Regras puras do perfil (máscara e leitura da data, idade, "hoje" no fuso da cidade, validação local, rótulos) em `src/lib/profile.ts`, e a data de calendário (`AAAA-MM-DD`, sem fuso) em `src/lib/format.ts`, testadas sem React. Um formulário único (`ProfileForm`) serve três lugares: pessoa nova na `PeopleStep`, perfil obrigatório e "Meu perfil" na `ProfileStep`. O catálogo é uma tela nova (`CatalogStep`) e o bloco de sugestões é um componente (`AlsoRecommended`) dentro da `ResultStep`. O `Flow` continua sendo a máquina de telas: ganha os estados `catalog` e `profile`, cria o par com perfil (`POST /citizen/people`) e começa a triagem por nome de protocolo. Quem decide o que é oferecido é sempre o api; o wpda só mostra.

**Tech Stack:** React 18 + TypeScript (strict, `noUnusedLocals`, `noUnusedParameters`), Vite 5, Vitest 2 + Testing Library com jest-dom e user-event. Sem TanStack Query nas telas do cidadão (estado local, como hoje). Nenhuma dependência nova.

**Spec:** `docs/superpowers/specs/2026-10-05-module-15-triage-catalog-design.md` (§8 "wpda", §6.1, §9.4, §9.5). ADR: `docs/adr/0027.md`. Contrato entre apps (fonte única dos formatos): `docs/superpowers/plans/2026-10-05-module-15-triage-catalog-contracts.md` (§0, §2, §3). Depende do plano do api do módulo 15 estar mergeado **antes** do merge deste.

## Global Constraints

- Rotas consumidas, exatamente nos formatos do contrato §3 (erro sempre `{ "error": "<reason>" }`):
  - `GET /citizen/people` → cada pessoa ganha `profile: { birth_date, sex, gender_identity, profile_source } | null`;
  - `POST /citizen/people` → corpo `{ cpf, consent_version, birth_date, sex, gender_identity, neighborhood_id? }`; 201 criou, 200 o par já existia (perfil **não** sobrescrito); 409 `consent_outdated`; 422 `invalid_cpf`, `too_many_people`, `invalid_birth_date`, `invalid_sex`, `invalid_gender_identity`, `invalid_neighborhood`; resposta `{ person }`;
  - `POST /citizen/people/:id/profile` → corpo `{ birth_date, sex, gender_identity }` (a chave `gender_identity` vai mesmo quando `null`); 404 `not_found`; 409 `profile_verified`; 422 dos três campos; resposta `{ person }`;
  - `GET /citizen/people/:id/catalog` → `{ in_progress, suggested, available, recent }` + `reference_units` (bairro atual do par, `[]` sem bairro) e `suggested[].source_title`; 404; 409 `profile_required`;
  - `POST /citizen/conversations` → o wpda manda **só** `{ citizen_id, protocol_name, consent_version }`; 409 `profile_required`, `not_offered`, `triage_in_progress`; 422 `protocol_name_required`; resposta igual à de hoje (`StartResult`);
  - `GET /citizen/triages/:id` → ganha `suggestions: [{ suggestion_id, protocol_name, title, summary }]`, `[]` em resultado urgente.
- Valores de perfil (contrato §2): `sex` ∈ `female | male`; `gender_identity` ∈ `cis_woman | cis_man | trans_woman | trans_man | travesti | non_binary | other` ou `null`; `birth_date` `AAAA-MM-DD`, não futura, idade ≤ 130. Rótulos: Feminino, Masculino; Mulher cis, Homem cis, Mulher trans, Homem trans, Travesti, Não binária, Outra; "Prefiro não informar" = `null`.
- **Perfil obrigatório** (spec §8.2, ADR 0027): sem perfil não há catálogo nem triagem. A elegibilidade é do api: o wpda nunca filtra, ordena ou esconde item do catálogo por conta própria.
- **Privacidade do perfil** (spec §5.5, ADR 0027 "Invariantes"): data de nascimento, idade, sexo e identidade de gênero nunca vão para URL, query string, `console.*` ou log. Nada de `<form>` nativo nas telas de perfil (um submit nativo GET poria os campos na URL); o campo de data tem `autoComplete="off"` (o celular é da família: preencher com o aniversário do dono estaria errado). Os corpos JSON são montados campo a campo, sem espalhar objetos.
- **Sugestão nunca começa sozinha** (ADR 0027): "Fazer agora" é um toque do cidadão; o bloco "Recomendamos também" não aparece com lista vazia.
- Datas de calendário do contrato (`birth_date`, `suggested_on`, `last_completed_on`, `next_available_on`) são formatadas por texto (`fmtCalendarDate`), **nunca** por `new Date(...)`: `new Date("2026-10-02")` é meia-noite UTC e vira 01/10 no fuso de Brasília. "Hoje", para validar a data de nascimento, é o dia no fuso da cidade (`cityTimeZone()`).
- Interface tem ciclo próprio (decisão do usuário): nada de redesign. Use `Screen`, `BigButton`, `Field`, `ErrorText` de `src/modules/citizen/ui.tsx` e o estilo do bloco `ReferenceUnits`; a única peça nova é `RadioGroup`, no mesmo estilo (texto ≥ 18 px, alvo ≥ 48 px).
- Testes que dependem de data fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(...)` em `beforeEach` (só `Date`: o polling da `ResultStep` usa `setTimeout` real). Mocks do cliente com `vi.spyOn(citizenApi, ...)`, como nos testes existentes.
- O WhatsApp está descontinuado; o wpda web é o único canal do cidadão.
- Nunca `git add -A` (o worktree tem symlink de `node_modules`); adicione arquivos pelo nome.
- Commits em inglês, Conventional Commits com tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

- Crie o worktree a partir de `origin/main` do wpda (topo atual `fbb505c`):

  ```bash
  cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda
  /opt/homebrew/bin/git fetch origin
  /opt/homebrew/bin/git worktree add .claude/mod15 -b feat/mod-15-triage-catalog origin/main
  cd .claude/mod15 && [ -e node_modules ] || ln -s ../../node_modules node_modules
  /opt/homebrew/bin/git status -sb && /opt/homebrew/bin/git log --oneline -1
  ```

  Esperado: `## feat/mod-15-triage-catalog...origin/main` sem mudanças. O `.gitignore` ignora `node_modules` (sem barra) e `/.claude/`; mesmo assim, nunca `git add -A`.

- Todos os comandos abaixo rodam em `apps/wpda/.claude/mod15`:

  ```bash
  npx vitest run <arquivos>    # um ou mais arquivos
  npm test                     # suíte inteira
  npx tsc --noEmit             # tipos, antes de cada commit
  ```

- Anote a contagem da suíte antes da Task 1 (`npm test`, linha `Tests  N passed`); nenhum teste que já existe pode cair sem que a task diga qual e por quê.

## Incorporado ao contrato

Duas propostas deste plano foram aceitas e já estão no contrato §3.4 (2026-10-05); o código abaixo as usa diretamente:

- **`reference_units` no catálogo:** `GET /citizen/people/:id/catalog` devolve as unidades de referência do bairro **atual** do par, no mesmo formato de `GET /citizen/triages/:id` (ADR 0023), `[]` sem bairro. O catálogo vazio mostra essas unidades (spec §8.7).
- **`suggested[].source_title`:** título da triagem que gerou a sugestão. A linha da sugestão é "Sugerida pelo resultado da triagem <source_title> de DD/MM/AAAA".

## Divergências entre a spec/contrato e o código (decididas neste plano)

1. **"Contato" da unidade** = nome, tipo e endereço, que é o que `ReferenceUnit` tem (não há telefone no modelo do módulo 11). Reaproveita o bloco `ReferenceUnits`.
2. **Interpretação do 200 em `POST /citizen/people`** (contrato §3.2): o par já existia. Se ele voltar **sem** `profile`, o wpda grava o perfil que a pessoa acabou de informar com `POST /citizen/people/:id/profile`; se voltar **com** perfil, segue para o catálogo sem sobrescrever (a correção é em "Meu perfil").
3. **Bairro de pessoa existente sem bairro.** Hoje ia junto do `POST /citizen/conversations`. Como o wpda novo manda só `citizen_id` + `protocol_name` na conversa, o bairro escolhido antes do catálogo vai por `POST /citizen/people/:id/neighborhood` (rota que já existe). Pessoa nova manda o bairro no `POST /citizen/people`. "Prefiro não informar" continua não mandando nada (a pergunta volta na próxima vez, como hoje).
4. **Ordem para CPF novo:** CPF → perfil → bairro → par criado. O perfil vem antes do bairro para que o 422 `invalid_neighborhood` continue sendo tratado onde já é hoje (a `PeopleStep` relê a lista e pergunta de novo), sem perder o perfil digitado.
5. **"Fazer outra triagem" no resultado** passa a abrir o catálogo da mesma pessoa (antes: a escolha de pessoa). "Voltar" no catálogo leva à escolha de pessoa.

## Review Focus

1. **Data de calendário deslocada pelo fuso:** `suggested_on: "2026-10-02"` ou `next_available_on: "2027-03-10"` lidos com `new Date(...)` aparecem um dia antes no fuso de Brasília. A pessoa esperaria ver exatamente a data do api. Teste: Task 2 (`fmtCalendarDate`) e Task 5 (texto na tela, com relógio fixo).
2. **"Hoje" perto da meia-noite e em outro fuso:** às 23h30 em Brasília o UTC já virou o dia; quem nasceu hoje tem que passar, quem "nasceu amanhã" tem que ser recusado, e a cidade de Manaus tem o próprio "hoje". Teste: Task 2 (`todayInCity`, `validateProfile`) e Task 3 (tela).
3. **Toque duplo em "Começar", "Continuar" ou "Fazer agora":** num celular lento, dois toques não podem abrir duas conversas (o segundo daria 409 `triage_in_progress` e uma mensagem falsa). Botões desabilitados até a resposta. Teste: Task 5 e Task 9.
4. **CPF que já existe no celular digitado como "outra pessoa":** o api responde 200 com o par existente; sem perfil, o wpda grava o perfil informado; com perfil, segue sem sobrescrever. A pessoa esperaria cair no catálogo, sem erro. Teste: Task 7.
5. **Api anterior a este deploy, ou campo opcional ausente:** `profile`, `suggestions`, `reference_units` e `summary` ausentes viram `null`/`[]`, e a tela nunca escreve "null" ou "undefined". Teste: Task 1 (normalização) e Task 5 (`summary: null`, catálogo sem `reference_units`).

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `src/lib/citizenApi.ts` | tipos `Sex`, `GenderIdentity`, `ProfileInput`, `Profile`, `Catalog*`, `TriageSuggestion`; `Person.profile`; `TriageSummary.suggestions`; `createPerson`, `setProfile`, `catalog`, `startTriage`; normalização; remoção de `start` (Task 7) | 1, 7 |
| `src/modules/citizen/ui.tsx` | mensagens dos códigos novos; `RadioGroup` | 1, 3 |
| `src/lib/format.ts` | `fmtCalendarDate` | 2 |
| `src/lib/profile.ts` | opções e rótulos, `maskDate`, `parseBirthDate`, `toMaskedDate`, `ageOn`, `todayInCity`, `validateProfile` | 2 |
| `src/modules/citizen/ProfileForm.tsx` | formulário de perfil (finalidade + termo, validação local) | 3 |
| `src/modules/citizen/PeopleStep.tsx` | CPF novo pede o perfil antes do bairro | 4 |
| `src/modules/citizen/CatalogStep.tsx` | catálogo em seções, vazio com unidade de referência | 5 |
| `src/modules/citizen/ProfileStep.tsx` | perfil obrigatório e "Meu perfil" (declared edita, verified só leitura) | 6 |
| `src/modules/citizen/Flow.tsx` | estados `catalog`/`profile`, criação do par, início por protocolo, recusas | 7, 8, 9 |
| `src/modules/citizen/AlsoRecommended.tsx`, `src/modules/citizen/ResultStep.tsx` | "Recomendamos também" | 9 |

---

### Task 1: Tipos, cliente da API e mensagens

**Files:**
- Modify: `src/lib/citizenApi.ts`
- Modify: `src/modules/citizen/ui.tsx` (mapa `MESSAGES`)
- Test: `src/lib/citizenApi.test.ts`, `src/modules/citizen/ui.test.tsx` (novo)

**Interfaces:**
- Produces (em `src/lib/citizenApi.ts`):
  - `type Sex = "female" | "male"`
  - `type GenderIdentity = "cis_woman" | "cis_man" | "trans_woman" | "trans_man" | "travesti" | "non_binary" | "other"`
  - `interface ProfileInput { birth_date: string; sex: Sex; gender_identity: GenderIdentity | null }`
  - `interface Profile extends ProfileInput { profile_source: "declared" | "verified" }`
  - `Person.profile?: Profile | null` (normalizado para `null` em `people()`, `createPerson()`, `setProfile()`)
  - `interface CatalogEntry { protocol_name: string; title: string; summary: string | null }`
  - `interface SuggestedEntry extends CatalogEntry { suggestion_id: string; source_triage_id: string; source_title: string; suggested_on: string }`
  - `interface RecentEntry extends CatalogEntry { last_completed_on: string; next_available_on: string }`
  - `interface InProgressEntry { conversation_id: string; protocol_name: string; title: string }`
  - `interface Catalog { in_progress: InProgressEntry | null; suggested: SuggestedEntry[]; available: CatalogEntry[]; recent: RecentEntry[]; reference_units: ReferenceUnit[] }`
  - `interface TriageSuggestion { suggestion_id: string; protocol_name: string; title: string; summary: string | null }`
  - `TriageSummary.suggestions?: TriageSuggestion[]` (normalizado para `[]` em `triage()` e `triages()`)
  - `citizenApi.createPerson(p: { cpf: string; consentVersion: string; profile: ProfileInput; neighborhoodId?: string }): Promise<Person>`
  - `citizenApi.setProfile(citizenId: string, profile: ProfileInput): Promise<Person>`
  - `citizenApi.catalog(citizenId: string): Promise<Catalog>`
  - `citizenApi.startTriage(p: { citizenId: string; protocolName: string; consentVersion: string }): Promise<StartResult>`
  - `citizenApi.start` continua existindo até a Task 7 (o `Flow` ainda o usa).
- Produces (em `ui.tsx`): `messageFor` conhece `invalid_birth_date`, `invalid_sex`, `invalid_gender_identity`, `profile_verified`, `profile_required`, `not_offered`, `triage_in_progress`, `protocol_name_required`.

- [ ] **Step 1: Escreva os testes do cliente**

Acrescente ao fim de `src/lib/citizenApi.test.ts` (o arquivo já tem `mockFetch`, `citizenApi` e `ApiError` importados):

```ts
describe("citizenApi — módulo 15 (perfil, catálogo, sugestões)", () => {
  const profile = { birth_date: "1963-04-02", sex: "female" as const, gender_identity: null };
  const person = {
    id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared", neighborhood: null,
    profile: { ...profile, profile_source: "declared" }
  };

  it("people normaliza profile ausente para null e preserva o presente", async () => {
    mockFetch(200, { people: [ { id: "p0", cpf_masked: "***.111.222-**", verification_level: "declared" }, person ] });
    const { people } = await citizenApi.people();
    expect(people[0].profile).toBeNull();
    expect(people[1].profile).toEqual({ ...profile, profile_source: "declared" });
  });

  it("createPerson manda CPF, termo e perfil, sem bairro quando não veio", async () => {
    const fn = mockFetch(201, { person });
    const result = await citizenApi.createPerson({ cpf: "529.982.247-25", consentVersion: "3", profile });
    const [url, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(url).toBe("/citizen/people");
    expect(init.method).toBe("POST");
    expect(JSON.parse(init.body as string)).toEqual({
      cpf: "529.982.247-25", consent_version: "3", birth_date: "1963-04-02", sex: "female", gender_identity: null
    });
    expect(result.id).toBe("p1");
    expect(result.profile?.profile_source).toBe("declared");
  });

  it("createPerson manda neighborhood_id quando veio", async () => {
    const fn = mockFetch(201, { person });
    await citizenApi.createPerson({ cpf: "529.982.247-25", consentVersion: "3", profile, neighborhoodId: "n1" });
    const body = JSON.parse((fn.mock.calls[0] as unknown as [string, RequestInit])[1].body as string);
    expect(body.neighborhood_id).toBe("n1");
  });

  it("createPerson (200, par que já existia) normaliza profile e neighborhood ausentes", async () => {
    mockFetch(200, { person: { id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared" } });
    const result = await citizenApi.createPerson({ cpf: "529.982.247-25", consentVersion: "3", profile });
    expect(result.profile).toBeNull();
    expect(result.neighborhood).toBeNull();
  });

  it("createPerson 409 consent_outdated vira ApiError", async () => {
    mockFetch(409, { error: "consent_outdated" });
    await expect(citizenApi.createPerson({ cpf: "529.982.247-25", consentVersion: "2", profile }))
      .rejects.toEqual(new ApiError(409, "consent_outdated"));
  });

  it("setProfile manda só os três campos, com gender_identity null explícito", async () => {
    const fn = mockFetch(200, { person });
    const result = await citizenApi.setProfile("p1", profile);
    const [url, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(url).toBe("/citizen/people/p1/profile");
    expect(init.method).toBe("POST");
    const body = JSON.parse(init.body as string);
    expect(body).toEqual({ birth_date: "1963-04-02", sex: "female", gender_identity: null });
    expect("gender_identity" in body).toBe(true);
    expect(result.id).toBe("p1");
  });

  it("setProfile 409 profile_verified vira ApiError", async () => {
    mockFetch(409, { error: "profile_verified" });
    await expect(citizenApi.setProfile("p1", profile)).rejects.toEqual(new ApiError(409, "profile_verified"));
  });

  it("nenhuma URL carrega dado de perfil", async () => {
    const fn = mockFetch(200, { person });
    await citizenApi.setProfile("p1", { birth_date: "1963-04-02", sex: "female", gender_identity: "trans_woman" });
    await citizenApi.createPerson({ cpf: "529.982.247-25", consentVersion: "3", profile });
    for (const call of fn.mock.calls) {
      expect(String((call as unknown[])[0])).not.toMatch(/1963|female|trans_woman|birth|sex|gender/);
    }
  });

  it("catalog faz GET em /people/:id/catalog e preserva o que veio", async () => {
    const full = {
      in_progress: { conversation_id: "c1", protocol_name: "triage-respiratoria", title: "Sintomas respiratórios" },
      suggested: [ { protocol_name: "saude-mental-aprofundada", title: "Saúde mental — aprofundamento", summary: "Mais perguntas.",
        suggestion_id: "s1", source_triage_id: "t0", source_title: "Saúde mental", suggested_on: "2026-10-02" } ],
      available: [ { protocol_name: "saude-do-idoso", title: "Saúde do idoso", summary: "Quedas, memória e medicamentos." } ],
      recent: [ { protocol_name: "saude-mental", title: "Saúde mental", summary: null,
        last_completed_on: "2026-10-02", next_available_on: "2027-04-02" } ],
      reference_units: [ { id: "u1", name: "UBS Batel", kind: "ubs",
        address: { street: null, number: null, complement: null, zip: null } } ]
    };
    const fn = mockFetch(200, full);
    expect(await citizenApi.catalog("p1")).toEqual(full);
    const [url, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(url).toBe("/citizen/people/p1/catalog");
    expect(init.method).toBe("GET");
  });

  it("catalog normaliza chaves ausentes (api anterior ou campo opcional)", async () => {
    mockFetch(200, { suggested: [ { protocol_name: "a", title: "A", suggestion_id: "s1", source_triage_id: "t0", source_title: "Saúde mental", suggested_on: "2026-10-02" } ] });
    expect(await citizenApi.catalog("p1")).toEqual({
      in_progress: null,
      suggested: [ { protocol_name: "a", title: "A", summary: null, suggestion_id: "s1", source_triage_id: "t0", source_title: "Saúde mental", suggested_on: "2026-10-02" } ],
      available: [], recent: [], reference_units: []
    });
  });

  it("catalog 409 profile_required vira ApiError", async () => {
    mockFetch(409, { error: "profile_required" });
    await expect(citizenApi.catalog("p1")).rejects.toEqual(new ApiError(409, "profile_required"));
  });

  it("startTriage manda citizen_id, protocol_name e a versão do termo, nada mais", async () => {
    const fn = mockFetch(201, { conversation_id: "c", citizen_id: "p1", resumed: false, step: {} });
    await citizenApi.startTriage({ citizenId: "p1", protocolName: "saude-do-idoso", consentVersion: "3" });
    const [url, init] = fn.mock.calls[0] as unknown as [string, RequestInit];
    expect(url).toBe("/citizen/conversations");
    expect(JSON.parse(init.body as string)).toEqual({ citizen_id: "p1", protocol_name: "saude-do-idoso", consent_version: "3" });
  });

  it("triage() sem suggestions normaliza para []; com a lista, normaliza summary ausente", async () => {
    const base = {
      id: "t1", status: "completed", tier: "alta", priority: 1, created_at: "2026-10-05T12:00:00Z",
      completed_at: "2026-10-05T12:05:00Z", report_url: null, consent_active: true, origin_phone_masked: null
    };
    mockFetch(200, base);
    expect((await citizenApi.triage("t1")).suggestions).toEqual([]);
    mockFetch(200, { ...base, suggestions: [ { suggestion_id: "s1", protocol_name: "b", title: "B" } ] });
    expect((await citizenApi.triage("t1")).suggestions).toEqual([ { suggestion_id: "s1", protocol_name: "b", title: "B", summary: null } ]);
  });
});
```

- [ ] **Step 2: Escreva o teste das mensagens**

```tsx
// src/modules/citizen/ui.test.tsx
import { describe, expect, it } from "vitest";
import { ApiError } from "../../lib/citizenApi";
import { messageFor } from "./ui";

// Módulo 15 (contrato §3): todo código novo que chega ao cidadão tem texto
// próprio, nunca o genérico.
describe("messageFor — módulo 15", () => {
  it.each([
    [ "invalid_birth_date", "Data de nascimento inválida. Confira dia, mês e ano." ],
    [ "invalid_sex", "Escolha o sexo." ],
    [ "invalid_gender_identity", "Escolha uma opção de identidade de gênero." ],
    [ "profile_verified", "Este perfil foi conferido no posto e só pode ser corrigido lá." ],
    [ "profile_required", "Antes, informe a data de nascimento e o sexo desta pessoa." ],
    [ "not_offered", "Esta triagem não está mais disponível para esta pessoa." ],
    [ "triage_in_progress", "Já existe uma triagem em andamento para esta pessoa. Continue a que está aberta." ],
    [ "protocol_name_required", "Escolha uma triagem para começar." ]
  ])("%s", (code, message) => {
    expect(messageFor(new ApiError(409, code))).toBe(message);
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/lib/citizenApi.test.ts src/modules/citizen/ui.test.tsx`
Expected: FAIL — `citizenApi.createPerson is not a function` (e similares) e as mensagens caem no genérico "Algo deu errado. Tente de novo.".

- [ ] **Step 4: Implemente os tipos e o cliente**

Em `src/lib/citizenApi.ts`, logo depois de `export interface ReferenceUnit { ... }`, acrescente:

```ts
// Perfil do par (ADR 0027; contrato do módulo 15 §2). Dado sensível: nunca em
// URL, query string ou console; os corpos abaixo são montados campo a campo.
export type Sex = "female" | "male";
export type GenderIdentity =
  "cis_woman" | "cis_man" | "trans_woman" | "trans_man" | "travesti" | "non_binary" | "other";

export interface ProfileInput {
  birth_date: string; // AAAA-MM-DD
  sex: Sex;
  gender_identity: GenderIdentity | null; // null = "Prefiro não informar"
}

export interface Profile extends ProfileInput {
  profile_source: "declared" | "verified";
}

// Catálogo de triagens de uma pessoa (contrato §3.4). O api decide o que é
// oferecido; o wpda só mostra.
export interface CatalogEntry { protocol_name: string; title: string; summary: string | null }
export interface SuggestedEntry extends CatalogEntry {
  suggestion_id: string;
  source_triage_id: string;
  source_title: string; // título da triagem que gerou a sugestão
  suggested_on: string; // AAAA-MM-DD
}
export interface RecentEntry extends CatalogEntry {
  last_completed_on: string; // AAAA-MM-DD
  next_available_on: string; // AAAA-MM-DD
}
export interface InProgressEntry { conversation_id: string; protocol_name: string; title: string }
export interface Catalog {
  in_progress: InProgressEntry | null;
  suggested: SuggestedEntry[];
  available: CatalogEntry[];
  recent: RecentEntry[];
  // Unidades do bairro atual do par ([] sem bairro). Ausente numa api anterior: [].
  reference_units: ReferenceUnit[];
}

// "Recomendamos também" (contrato §3.6): [] em resultado urgente.
export interface TriageSuggestion { suggestion_id: string; protocol_name: string; title: string; summary: string | null }
```

Na `interface Person`, depois de `neighborhood?: Neighborhood | null;`, acrescente:

```ts
  // Perfil (ADR 0027). Ausente numa api anterior: normalizado em normalizePerson.
  profile?: Profile | null;
```

Na `interface TriageSummary`, depois de `reference_units?: ReferenceUnit[];`, acrescente:

```ts
  // Sugestões nascidas desta triagem (módulo 15). Ausente numa api anterior:
  // normalizado para [] (ver normalizeTriage).
  suggestions?: TriageSuggestion[];
```

Em `normalizeTriage`, troque a última linha do objeto devolvido (`reference_units: t.reference_units ?? []`) por:

```ts
    reference_units: t.reference_units ?? [],
    suggestions: (t.suggestions ?? []).map(s => ({ ...s, summary: s.summary ?? null }))
```

Troque `normalizePerson` por:

```ts
function normalizePerson(p: Person): Person {
  return { ...p, neighborhood: p.neighborhood ?? null, profile: p.profile ?? null };
}

function withSummary<T extends CatalogEntry>(e: T): T {
  return { ...e, summary: e.summary ?? null };
}

function profileBody(profile: ProfileInput) {
  return { birth_date: profile.birth_date, sex: profile.sex, gender_identity: profile.gender_identity };
}
```

No objeto `citizenApi`, depois de `setNeighborhood: ...,`, acrescente:

```ts
  // Par novo já com perfil (contrato §0.1 e §3.2): confere o termo antes de
  // gravar o CPF. 200 = o par já existia (perfil não é sobrescrito).
  createPerson: async (p: { cpf: string; consentVersion: string; profile: ProfileInput; neighborhoodId?: string }) =>
    normalizePerson((await call<{ person: Person }>("POST", "/people", {
      cpf: p.cpf,
      consent_version: p.consentVersion,
      ...profileBody(p.profile),
      ...(p.neighborhoodId ? { neighborhood_id: p.neighborhoodId } : {})
    })).person),
  // Correção pelo cidadão enquanto declared (409 profile_verified depois do posto).
  setProfile: async (citizenId: string, profile: ProfileInput) =>
    normalizePerson((await call<{ person: Person }>(
      "POST", `/people/${encodeURIComponent(citizenId)}/profile`, profileBody(profile))).person),
  catalog: async (citizenId: string): Promise<Catalog> => {
    const d = await call<Partial<Catalog>>("GET", `/people/${encodeURIComponent(citizenId)}/catalog`);
    return {
      in_progress: d.in_progress ?? null,
      suggested: (d.suggested ?? []).map(withSummary),
      available: (d.available ?? []).map(withSummary),
      recent: (d.recent ?? []).map(withSummary),
      reference_units: d.reference_units ?? []
    };
  },
  // Início pela triagem escolhida (contrato §3.5). Mesmo protocolo em andamento = retoma.
  startTriage: (p: { citizenId: string; protocolName: string; consentVersion: string }) =>
    call<StartResult>("POST", "/conversations", {
      citizen_id: p.citizenId, protocol_name: p.protocolName, consent_version: p.consentVersion
    }),
```

- [ ] **Step 5: Implemente as mensagens**

Em `src/modules/citizen/ui.tsx`, dentro de `MESSAGES`, depois de `invalid_neighborhood: INVALID_NEIGHBORHOOD_MESSAGE`, troque a linha por:

```ts
  invalid_neighborhood: INVALID_NEIGHBORHOOD_MESSAGE,
  // Módulo 15 (contrato §3).
  invalid_birth_date: "Data de nascimento inválida. Confira dia, mês e ano.",
  invalid_sex: "Escolha o sexo.",
  invalid_gender_identity: "Escolha uma opção de identidade de gênero.",
  profile_verified: "Este perfil foi conferido no posto e só pode ser corrigido lá.",
  profile_required: "Antes, informe a data de nascimento e o sexo desta pessoa.",
  not_offered: "Esta triagem não está mais disponível para esta pessoa.",
  triage_in_progress: "Já existe uma triagem em andamento para esta pessoa. Continue a que está aberta.",
  protocol_name_required: "Escolha uma triagem para começar."
```

- [ ] **Step 6: Rode e veja passar; suíte e tipos**

Run: `npx vitest run src/lib/citizenApi.test.ts src/modules/citizen/ui.test.tsx && npm test && npx tsc --noEmit`
Expected: PASS, sem queda de teste existente.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/lib/citizenApi.ts src/lib/citizenApi.test.ts src/modules/citizen/ui.tsx src/modules/citizen/ui.test.tsx
/opt/homebrew/bin/git commit -m "feat: add citizen profile, triage catalog and suggestion api client" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras puras do perfil e data de calendário

**Files:**
- Create: `src/lib/profile.ts`
- Modify: `src/lib/format.ts`
- Test: `src/lib/profile.test.ts` (novo), `src/lib/format.test.ts`

**Interfaces:**
- Consumes: `Sex`, `GenderIdentity`, `ProfileInput` (Task 1); `cityDateFormat`, `cityTimeZone`, `setCityTimeZone` de `format.ts`; `onlyDigits` de `masks.ts`.
- Produces (em `src/lib/format.ts`): `fmtCalendarDate(iso: string | null | undefined): string` — `"2026-10-02"` → `"02/10/2026"`, inválido → `"—"`.
- Produces (em `src/lib/profile.ts`):
  - `MAX_AGE = 130`
  - `SEX_OPTIONS: readonly { value: Sex; label: string }[]`
  - `GENDER_IDENTITY_OPTIONS: readonly { value: GenderIdentity | null; label: string }[]` (primeiro item `null` = "Prefiro não informar")
  - `sexLabel(s: Sex): string`, `genderIdentityLabel(g: GenderIdentity | null): string` (`null` → "Não informado")
  - `maskDate(input: string): string` (`"02041963"` → `"02/04/1963"`)
  - `parseBirthDate(masked: string): string | null` (`"02/04/1963"` → `"1963-04-02"`)
  - `toMaskedDate(iso: string): string` (`"1963-04-02"` → `"02/04/1963"`, inválido → `""`)
  - `ageOn(birthIso: string, todayIso: string): number`
  - `todayInCity(now?: Date): string` (AAAA-MM-DD no fuso da cidade)
  - `type ProfileDraft = { birthDate: string; sex: Sex | null; genderIdentity: GenderIdentity | null }`
  - `type ProfileErrors = { birthDate?: string; sex?: string }`
  - `validateProfile(d: ProfileDraft, today: string): { ok: true; value: ProfileInput } | { ok: false; errors: ProfileErrors }`
  - mensagens exportadas: `BIRTH_DATE_INVALID`, `BIRTH_DATE_FUTURE`, `BIRTH_DATE_TOO_OLD`, `SEX_REQUIRED`

- [ ] **Step 1: Escreva os testes de `fmtCalendarDate`**

Em `src/lib/format.test.ts`, troque a linha de import de `./format` por:

```ts
import { cityTimeZone, fmtCalendarDate, fmtDate, fmtDateTime, setCityTimeZone } from "./format";
```

e acrescente ao fim do arquivo:

```ts
// Datas de calendário do módulo 15 (nascimento, sugerida em, próxima a partir
// de): sem hora nem fuso. new Date("2026-10-02") seria 01/10 em Brasília.
describe("fmtCalendarDate", () => {
  it("AAAA-MM-DD vira DD/MM/AAAA, sem deslocar o dia pelo fuso", () => {
    expect(fmtCalendarDate("2026-10-02")).toBe("02/10/2026");
    expect(fmtCalendarDate("2027-01-01")).toBe("01/01/2027");
  });
  it("vale em qualquer fuso de cidade", () => {
    setCityTimeZone("America/Manaus");
    expect(fmtCalendarDate("2026-10-02")).toBe("02/10/2026");
    setCityTimeZone(null);
  });
  it.each([ null, undefined, "", "2026-10-02T00:00:00Z", "02/10/2026", "2026-1-2" ])("%s vira —", (v) => {
    expect(fmtCalendarDate(v)).toBe("—");
  });
});
```

- [ ] **Step 2: Escreva os testes de `profile.ts`**

```ts
// src/lib/profile.test.ts
import { afterEach, describe, expect, it } from "vitest";
import {
  BIRTH_DATE_FUTURE, BIRTH_DATE_INVALID, BIRTH_DATE_TOO_OLD, GENDER_IDENTITY_OPTIONS, SEX_OPTIONS, SEX_REQUIRED,
  ageOn, genderIdentityLabel, maskDate, parseBirthDate, sexLabel, toMaskedDate, todayInCity, validateProfile
} from "./profile";
import { setCityTimeZone } from "./format";

afterEach(() => setCityTimeZone(null));

describe("opções e rótulos (contrato §2)", () => {
  it("sexo: os dois valores do contrato", () => {
    expect(SEX_OPTIONS).toEqual([ { value: "female", label: "Feminino" }, { value: "male", label: "Masculino" } ]);
    expect(sexLabel("female")).toBe("Feminino");
    expect(sexLabel("male")).toBe("Masculino");
  });
  it("identidade de gênero: 'Prefiro não informar' primeiro (null), depois os sete valores", () => {
    expect(GENDER_IDENTITY_OPTIONS).toEqual([
      { value: null, label: "Prefiro não informar" },
      { value: "cis_woman", label: "Mulher cis" },
      { value: "cis_man", label: "Homem cis" },
      { value: "trans_woman", label: "Mulher trans" },
      { value: "trans_man", label: "Homem trans" },
      { value: "travesti", label: "Travesti" },
      { value: "non_binary", label: "Não binária" },
      { value: "other", label: "Outra" }
    ]);
    expect(genderIdentityLabel(null)).toBe("Não informado");
    expect(genderIdentityLabel("non_binary")).toBe("Não binária");
  });
});

describe("maskDate", () => {
  it.each([
    [ "", "" ], [ "0", "0" ], [ "02", "02" ], [ "020", "02/0" ], [ "0204", "02/04" ], [ "02041", "02/04/1" ],
    [ "02041963", "02/04/1963" ], [ "020419639", "02/04/1963" ], [ "02/04/1963", "02/04/1963" ], [ "ab02cd", "02" ]
  ])("%s → %s", (input, out) => expect(maskDate(input)).toBe(out));
});

describe("parseBirthDate", () => {
  it("dia/mês/ano vira AAAA-MM-DD", () => expect(parseBirthDate("02/04/1963")).toBe("1963-04-02"));
  it("29/02 em ano bissexto existe", () => expect(parseBirthDate("29/02/2024")).toBe("2024-02-29"));
  it.each([ "", "02/04/63", "2/4/1963", "31/02/2000", "29/02/2025", "00/01/2000", "01/13/2000", "01/01/0063" ])(
    "recusa %s", (s) => expect(parseBirthDate(s)).toBeNull());
});

describe("toMaskedDate", () => {
  it("AAAA-MM-DD vira DD/MM/AAAA", () => expect(toMaskedDate("1963-04-02")).toBe("02/04/1963"));
  it("inválido vira vazio", () => expect(toMaskedDate("1963-4-2")).toBe(""));
});

describe("ageOn", () => {
  it("aniversário hoje conta o ano", () => expect(ageOn("1966-10-05", "2026-10-05")).toBe(60));
  it("véspera do aniversário ainda não", () => expect(ageOn("1966-10-06", "2026-10-05")).toBe(59));
  it("nascido em 29/02 faz aniversário em 01/03 no ano comum", () => {
    expect(ageOn("2000-02-29", "2025-02-28")).toBe(24);
    expect(ageOn("2000-02-29", "2025-03-01")).toBe(25);
  });
  it("nascido hoje tem 0", () => expect(ageOn("2026-10-05", "2026-10-05")).toBe(0));
});

describe("todayInCity", () => {
  it("23h30 em Brasília ainda é o mesmo dia (no UTC já virou)", () => {
    expect(todayInCity(new Date("2026-10-05T23:30:00-03:00"))).toBe("2026-10-05");
  });
  it("segue o fuso da cidade da sessão", () => {
    setCityTimeZone("America/Manaus");
    expect(todayInCity(new Date("2026-10-06T00:30:00-03:00"))).toBe("2026-10-05");
  });
});

describe("validateProfile", () => {
  const TODAY = "2026-10-05";

  it("válido: data ISO, sexo e identidade null", () => {
    expect(validateProfile({ birthDate: "02/04/1963", sex: "female", genderIdentity: null }, TODAY))
      .toEqual({ ok: true, value: { birth_date: "1963-04-02", sex: "female", gender_identity: null } });
  });
  it("identidade escolhida vai no valor do contrato", () => {
    const r = validateProfile({ birthDate: "02/04/1963", sex: "male", genderIdentity: "trans_man" }, TODAY);
    expect(r).toEqual({ ok: true, value: { birth_date: "1963-04-02", sex: "male", gender_identity: "trans_man" } });
  });
  it("vazio aponta os dois campos", () => {
    expect(validateProfile({ birthDate: "", sex: null, genderIdentity: null }, TODAY))
      .toEqual({ ok: false, errors: { birthDate: BIRTH_DATE_INVALID, sex: SEX_REQUIRED } });
  });
  it("data incompleta ou que não existe é inválida", () => {
    for (const d of [ "02/04/19", "31/02/2000" ]) {
      expect(validateProfile({ birthDate: d, sex: "female", genderIdentity: null }, TODAY))
        .toEqual({ ok: false, errors: { birthDate: BIRTH_DATE_INVALID } });
    }
  });
  it("nascido hoje passa; amanhã é futuro", () => {
    expect(validateProfile({ birthDate: "05/10/2026", sex: "female", genderIdentity: null }, TODAY).ok).toBe(true);
    expect(validateProfile({ birthDate: "06/10/2026", sex: "female", genderIdentity: null }, TODAY))
      .toEqual({ ok: false, errors: { birthDate: BIRTH_DATE_FUTURE } });
  });
  it("130 anos é o teto; 131 é recusado", () => {
    expect(validateProfile({ birthDate: "05/10/1896", sex: "female", genderIdentity: null }, TODAY).ok).toBe(true);
    expect(validateProfile({ birthDate: "06/10/1895", sex: "female", genderIdentity: null }, TODAY).ok).toBe(true);
    expect(validateProfile({ birthDate: "05/10/1895", sex: "female", genderIdentity: null }, TODAY))
      .toEqual({ ok: false, errors: { birthDate: BIRTH_DATE_TOO_OLD } });
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `npx vitest run src/lib/profile.test.ts src/lib/format.test.ts`
Expected: FAIL — `Failed to resolve import "./profile"` e `fmtCalendarDate is not a function`.

- [ ] **Step 4: Implemente `fmtCalendarDate`**

Acrescente ao fim de `src/lib/format.ts`:

```ts
// Data de calendário "AAAA-MM-DD" (nascimento, "sugerida em", "próxima a partir
// de"): sem hora nem fuso. Nunca passa por new Date, que leria meia-noite UTC e
// mostraria o dia anterior no fuso de Brasília.
export function fmtCalendarDate(iso: string | null | undefined): string {
  const m = /^(\d{4})-(\d{2})-(\d{2})$/.exec(iso ?? "");
  return m ? `${m[3]}/${m[2]}/${m[1]}` : "—";
}
```

- [ ] **Step 5: Implemente `profile.ts`**

```ts
// src/lib/profile.ts
// Regras de tela do perfil do par (ADR 0027; contrato do módulo 15 §2), sem
// React. O api valida de novo (Citizens::SetProfile → 422); aqui o cidadão vê
// o erro na hora. Nada daqui escreve em console ou URL.
import type { GenderIdentity, ProfileInput, Sex } from "./citizenApi";
import { cityDateFormat, fmtCalendarDate } from "./format";
import { onlyDigits } from "./masks";

export const MAX_AGE = 130;

export const SEX_OPTIONS: readonly { value: Sex; label: string }[] = [
  { value: "female", label: "Feminino" },
  { value: "male", label: "Masculino" }
];

// Valores do Cadastro Individual do e-SUS APS; null = "Prefiro não informar".
export const GENDER_IDENTITY_OPTIONS: readonly { value: GenderIdentity | null; label: string }[] = [
  { value: null, label: "Prefiro não informar" },
  { value: "cis_woman", label: "Mulher cis" },
  { value: "cis_man", label: "Homem cis" },
  { value: "trans_woman", label: "Mulher trans" },
  { value: "trans_man", label: "Homem trans" },
  { value: "travesti", label: "Travesti" },
  { value: "non_binary", label: "Não binária" },
  { value: "other", label: "Outra" }
];

export function sexLabel(s: Sex): string {
  return SEX_OPTIONS.find(o => o.value === s)?.label ?? "";
}

export function genderIdentityLabel(g: GenderIdentity | null): string {
  if (g === null) return "Não informado";
  return GENDER_IDENTITY_OPTIONS.find(o => o.value === g)?.label ?? "";
}

export const BIRTH_DATE_INVALID = "Digite a data como dia/mês/ano, por exemplo 02/04/1963.";
export const BIRTH_DATE_FUTURE = "A data de nascimento não pode ser depois de hoje.";
export const BIRTH_DATE_TOO_OLD = "Confira o ano de nascimento.";
export const SEX_REQUIRED = "Escolha o sexo.";

export function maskDate(input: string): string {
  const d = onlyDigits(input).slice(0, 8);
  if (d.length <= 2) return d;
  if (d.length <= 4) return `${d.slice(0, 2)}/${d.slice(2)}`;
  return `${d.slice(0, 2)}/${d.slice(2, 4)}/${d.slice(4)}`;
}

// "02/04/1963" → "1963-04-02"; data que não existe (31/02) → null. Anos de 0001
// a 0099 caem fora sozinhos: Date.UTC os lê como 1900–1999.
export function parseBirthDate(masked: string): string | null {
  const m = /^(\d{2})\/(\d{2})\/(\d{4})$/.exec(masked.trim());
  if (!m) return null;
  const [ , dd, mm, yyyy ] = m;
  const day = Number(dd), month = Number(mm), year = Number(yyyy);
  const d = new Date(Date.UTC(year, month - 1, day));
  if (d.getUTCFullYear() !== year || d.getUTCMonth() !== month - 1 || d.getUTCDate() !== day) return null;
  return `${yyyy}-${mm}-${dd}`;
}

export function toMaskedDate(iso: string): string {
  const out = fmtCalendarDate(iso);
  return out === "—" ? "" : out;
}

export function ageOn(birthIso: string, todayIso: string): number {
  const [ by, bm, bd ] = birthIso.split("-").map(Number);
  const [ ty, tm, td ] = todayIso.split("-").map(Number);
  const beforeBirthday = tm < bm || (tm === bm && td < bd);
  return ty - by - (beforeBirthday ? 1 : 0);
}

// Hoje no fuso da cidade (o da sessão do cidadão), em AAAA-MM-DD.
export function todayInCity(now: Date = new Date()): string {
  const parts = cityDateFormat({ year: "numeric", month: "2-digit", day: "2-digit" }).formatToParts(now);
  const get = (type: string) => parts.find(p => p.type === type)?.value ?? "";
  return `${get("year")}-${get("month")}-${get("day")}`;
}

export type ProfileDraft = { birthDate: string; sex: Sex | null; genderIdentity: GenderIdentity | null };
export type ProfileErrors = { birthDate?: string; sex?: string };

export function validateProfile(d: ProfileDraft, today: string):
  { ok: true; value: ProfileInput } | { ok: false; errors: ProfileErrors } {
  const errors: ProfileErrors = {};
  const iso = parseBirthDate(d.birthDate);
  if (!iso) errors.birthDate = BIRTH_DATE_INVALID;
  else if (iso > today) errors.birthDate = BIRTH_DATE_FUTURE;
  else if (ageOn(iso, today) > MAX_AGE) errors.birthDate = BIRTH_DATE_TOO_OLD;
  if (!d.sex) errors.sex = SEX_REQUIRED;
  if (!iso || !d.sex || errors.birthDate) return { ok: false, errors };
  return { ok: true, value: { birth_date: iso, sex: d.sex, gender_identity: d.genderIdentity } };
}
```

- [ ] **Step 6: Rode e veja passar; suíte e tipos**

Run: `npx vitest run src/lib/profile.test.ts src/lib/format.test.ts && npm test && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/lib/profile.ts src/lib/profile.test.ts src/lib/format.ts src/lib/format.test.ts
/opt/homebrew/bin/git commit -m "feat: add citizen profile validation and calendar date formatting" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Formulário de perfil

**Files:**
- Modify: `src/modules/citizen/ui.tsx` (componente `RadioGroup`)
- Create: `src/modules/citizen/ProfileForm.tsx`
- Test: `src/modules/citizen/ProfileForm.test.tsx` (novo)

**Interfaces:**
- Consumes: `ProfileInput`, `citizenApi.consentTerm` (Task 1); `SEX_OPTIONS`, `GENDER_IDENTITY_OPTIONS`, `maskDate`, `toMaskedDate`, `todayInCity`, `validateProfile`, `ProfileErrors` (Task 2).
- Produces:
  - `RadioGroup<T extends string | null>({ legend, options, value, onChange, error?, disabled? })` em `ui.tsx`.
  - `PROFILE_PURPOSE: string` e `ProfileForm({ title, who?, initial?, busy?, error?, submitLabel, onSubmit, onBack })` em `ProfileForm.tsx`, com `onSubmit: (profile: ProfileInput) => void` chamado **só** com dados válidos.

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/citizen/ProfileForm.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { PROFILE_PURPOSE, ProfileForm } from "./ProfileForm";
import { citizenApi } from "../../lib/citizenApi";
import { BIRTH_DATE_FUTURE, BIRTH_DATE_INVALID, BIRTH_DATE_TOO_OLD, SEX_REQUIRED } from "../../lib/profile";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-05T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

type Props = Parameters<typeof ProfileForm>[0];

function setup(props: Partial<Props> = {}) {
  const onSubmit = vi.fn();
  const onBack = vi.fn();
  const view = render(<ProfileForm title="Sobre esta pessoa" submitLabel="Continuar"
    onSubmit={onSubmit} onBack={onBack} {...props} />);
  return { onSubmit, onBack, ...view };
}

const birth = () => screen.getByLabelText("Data de nascimento");
const submit = () => userEvent.click(screen.getByRole("button", { name: "Continuar" }));

describe("ProfileForm", () => {
  it("sem nada preenchido: aponta os dois campos e não envia", async () => {
    const { onSubmit } = setup();
    await submit();
    expect(screen.getByText(BIRTH_DATE_INVALID)).toBeInTheDocument();
    expect(screen.getByText(SEX_REQUIRED)).toBeInTheDocument();
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it("mascara a data e envia; identidade de gênero começa em 'Prefiro não informar' (null)", async () => {
    const { onSubmit } = setup();
    await userEvent.type(birth(), "02041963");
    expect(birth()).toHaveValue("02/04/1963");
    expect(screen.getByRole("radio", { name: "Prefiro não informar" })).toBeChecked();
    await userEvent.click(screen.getByRole("radio", { name: "Feminino" }));
    await submit();
    expect(onSubmit).toHaveBeenCalledWith({ birth_date: "1963-04-02", sex: "female", gender_identity: null });
  });

  it("identidade de gênero escolhida vai com o valor do contrato", async () => {
    const { onSubmit } = setup();
    await userEvent.type(birth(), "02041963");
    await userEvent.click(screen.getByRole("radio", { name: "Feminino" }));
    await userEvent.click(screen.getByRole("radio", { name: "Mulher trans" }));
    await submit();
    expect(onSubmit).toHaveBeenCalledWith({ birth_date: "1963-04-02", sex: "female", gender_identity: "trans_woman" });
  });

  it("data futura, data que não existe e idade acima de 130 são recusadas na hora", async () => {
    const { onSubmit } = setup();
    await userEvent.click(screen.getByRole("radio", { name: "Masculino" }));
    await userEvent.type(birth(), "06102026");
    await submit();
    expect(screen.getByText(BIRTH_DATE_FUTURE)).toBeInTheDocument();
    await userEvent.clear(birth());
    await userEvent.type(birth(), "31022000");
    await submit();
    expect(screen.getByText(BIRTH_DATE_INVALID)).toBeInTheDocument();
    await userEvent.clear(birth());
    await userEvent.type(birth(), "05101895");
    await submit();
    expect(screen.getByText(BIRTH_DATE_TOO_OLD)).toBeInTheDocument();
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it("às 23h30 de Brasília, quem nasceu hoje passa", async () => {
    vi.setSystemTime(new Date("2026-10-05T23:30:00-03:00"));
    const { onSubmit } = setup();
    await userEvent.type(birth(), "05102026");
    await userEvent.click(screen.getByRole("radio", { name: "Feminino" }));
    await submit();
    expect(onSubmit).toHaveBeenCalledWith({ birth_date: "2026-10-05", sex: "female", gender_identity: null });
  });

  it("perfil existente vem preenchido", () => {
    setup({ initial: { birth_date: "1963-04-02", sex: "male", gender_identity: "cis_man" } });
    expect(birth()).toHaveValue("02/04/1963");
    expect(screen.getByRole("radio", { name: "Masculino" })).toBeChecked();
    expect(screen.getByRole("radio", { name: "Homem cis" })).toBeChecked();
  });

  it("mostra quem é, a finalidade e abre o termo sem sair da tela", async () => {
    const term = vi.spyOn(citizenApi, "consentTerm").mockResolvedValue({ version: "3", body: "Texto do termo" });
    setup({ who: "CPF ***.982.247-**" });
    expect(screen.getByText("CPF ***.982.247-**")).toBeInTheDocument();
    expect(screen.getByText(PROFILE_PURPOSE)).toBeInTheDocument();
    expect(term).not.toHaveBeenCalled();
    await userEvent.click(screen.getByRole("button", { name: "Ler o termo de consentimento" }));
    expect(await screen.findByText("Texto do termo")).toBeInTheDocument();
    expect(screen.getByRole("button", { name: "Fechar o termo" })).toHaveAttribute("aria-expanded", "true");
  });

  it("sem <form> nativo e sem preenchimento automático da data", () => {
    const { container } = setup();
    expect(container.querySelector("form")).toBeNull();
    expect(birth()).toHaveAttribute("autocomplete", "off");
  });

  it("erro vindo do api aparece; 'Voltar' não envia", async () => {
    const { onSubmit, onBack } = setup({ error: "Data de nascimento inválida. Confira dia, mês e ano." });
    expect(screen.getByText("Data de nascimento inválida. Confira dia, mês e ano.")).toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Voltar" }));
    expect(onBack).toHaveBeenCalled();
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it("ocupado: botões e opções desabilitados", () => {
    setup({ busy: true });
    expect(screen.getByRole("button", { name: "Continuar" })).toBeDisabled();
    expect(screen.getByRole("radio", { name: "Feminino" })).toBeDisabled();
  });

  it("cada opção é um alvo de 48 px com texto de 18 px", () => {
    setup();
    const label = screen.getByRole("radio", { name: "Feminino" }).closest("label") as HTMLElement;
    expect(label.style.minHeight).toBe("48px");
    expect(label.style.fontSize).toBe("18px");
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/ProfileForm.test.tsx`
Expected: FAIL — `Failed to resolve import "./ProfileForm"`.

- [ ] **Step 3: Implemente `RadioGroup` em `ui.tsx`**

Acrescente em `src/modules/citizen/ui.tsx`, depois do componente `Toggle`:

```tsx
// Escolha única: a opção inteira é o alvo de toque (48 px ou mais). Aceita
// null como valor ("Prefiro não informar").
export function RadioGroup<T extends string | null>({ legend, options, value, onChange, error, disabled }: {
  legend: string; options: readonly { value: T; label: string }[]; value: T | undefined;
  onChange: (value: T) => void; error?: string; disabled?: boolean;
}) {
  const name = useId();
  return (
    <fieldset style={{ border: "none", padding: 0, margin: "0 0 16px", display: "grid", gap: 8 }}>
      <legend style={{ fontWeight: 600, padding: 0, marginBottom: 6 }}>{legend}</legend>
      {options.map(o => (
        <label key={o.value ?? "none"}
          style={{ display: "flex", alignItems: "center", gap: 12, minHeight: 48, fontSize: 18, padding: "0 12px",
            borderRadius: 12, border: "1px solid var(--rule2, #ccc)", cursor: disabled ? "default" : "pointer",
            opacity: disabled ? 0.6 : 1 }}>
          <input type="radio" name={name} checked={value === o.value} disabled={disabled}
            onChange={() => onChange(o.value)} style={{ width: 24, height: 24, margin: 0, flexShrink: 0 }} />
          {o.label}
        </label>
      ))}
      {error && <ErrorText>{error}</ErrorText>}
    </fieldset>
  );
}
```

- [ ] **Step 4: Implemente `ProfileForm`**

```tsx
// src/modules/citizen/ProfileForm.tsx
// Perfil do par (spec 2026-10-05 §8.2; ADR 0027): data de nascimento, sexo e
// identidade de gênero opcional. Sem <form> nativo: um submit GET poria o
// perfil na URL. A validação local espelha os 422 do api.
import { useState } from "react";
import { citizenApi, type GenderIdentity, type ProfileInput, type Sex } from "../../lib/citizenApi";
import {
  GENDER_IDENTITY_OPTIONS, SEX_OPTIONS, maskDate, toMaskedDate, todayInCity, validateProfile, type ProfileErrors
} from "../../lib/profile";
import { BigButton, ErrorText, Field, RadioGroup, Screen, messageFor } from "./ui";

export const PROFILE_PURPOSE =
  "Usamos a data de nascimento e o sexo só para mostrar as triagens indicadas para esta pessoa. Você pode corrigir depois em \"Meu perfil\".";

const linkStyle = { minHeight: 48, padding: 0, background: "none", border: "none", textDecoration: "underline", fontSize: 18 } as const;

function ConsentTermLink() {
  const [open, setOpen] = useState(false);
  const [term, setTerm] = useState<string | null>(null);
  const [error, setError] = useState<string | null>(null);

  function toggle() {
    const next = !open;
    setOpen(next);
    if (next && term === null) {
      citizenApi.consentTerm().then(t => setTerm(t.body)).catch(e => setError(messageFor(e)));
    }
  }

  return (
    <div style={{ marginBottom: 16 }}>
      <button type="button" aria-expanded={open} onClick={toggle} style={linkStyle}>
        {open ? "Fechar o termo" : "Ler o termo de consentimento"}
      </button>
      {open && (error ? <ErrorText>{error}</ErrorText>
        : term === null ? <p>Carregando…</p>
        : <div style={{ whiteSpace: "pre-wrap", lineHeight: 1.5, fontSize: 18, padding: 12, borderRadius: 12,
            border: "1px solid var(--line, #eee)" }}>{term}</div>)}
    </div>
  );
}

export function ProfileForm({ title, who, initial, busy, error, submitLabel, onSubmit, onBack }: {
  title: string; who?: string; initial?: ProfileInput | null; busy?: boolean; error?: string | null;
  submitLabel: string; onSubmit: (profile: ProfileInput) => void; onBack: () => void;
}) {
  const [birthDate, setBirthDate] = useState(initial ? toMaskedDate(initial.birth_date) : "");
  const [sex, setSex] = useState<Sex | null>(initial?.sex ?? null);
  const [genderIdentity, setGenderIdentity] = useState<GenderIdentity | null>(initial?.gender_identity ?? null);
  const [errors, setErrors] = useState<ProfileErrors>({});

  function submit() {
    const result = validateProfile({ birthDate, sex, genderIdentity }, todayInCity());
    if (!result.ok) {
      setErrors(result.errors);
      return;
    }
    setErrors({});
    onSubmit(result.value);
  }

  return (
    <Screen title={title} footer={<>
      <BigButton disabled={busy} onClick={submit}>{submitLabel}</BigButton>
      <BigButton variant="secondary" disabled={busy} onClick={onBack}>Voltar</BigButton>
    </>}>
      {who && <p>{who}</p>}
      <p>{PROFILE_PURPOSE}</p>
      <ConsentTermLink />
      <Field label="Data de nascimento" inputMode="numeric" autoComplete="off" placeholder="dd/mm/aaaa"
        hint="Dia, mês e ano. Exemplo: 02/04/1963." value={birthDate} error={errors.birthDate} disabled={busy}
        onChange={e => setBirthDate(maskDate(e.target.value))} />
      <RadioGroup legend="Sexo" options={SEX_OPTIONS} value={sex} onChange={setSex} error={errors.sex} disabled={busy} />
      <RadioGroup legend="Identidade de gênero (opcional)" options={GENDER_IDENTITY_OPTIONS} value={genderIdentity}
        onChange={setGenderIdentity} disabled={busy} />
      {error && <ErrorText>{error}</ErrorText>}
    </Screen>
  );
}
```

- [ ] **Step 5: Rode e veja passar; suíte e tipos**

Run: `npx vitest run src/modules/citizen/ProfileForm.test.tsx && npm test && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/ui.tsx src/modules/citizen/ProfileForm.tsx src/modules/citizen/ProfileForm.test.tsx
/opt/homebrew/bin/git commit -m "feat: add citizen profile form with local validation" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Pessoa nova informa o perfil antes do bairro

**Files:**
- Modify: `src/modules/citizen/PeopleStep.tsx`
- Test: `src/modules/citizen/PeopleStep.test.tsx`

**Interfaces:**
- Consumes: `ProfileForm` (Task 3), `ProfileInput` (Task 1).
- Produces: `export type Who = { citizenId: string } | { cpf: string; profile: ProfileInput }`; `PersonChoice = Who & { neighborhoodId?: string }` (mesmo nome de hoje). Para pessoa existente, `onChoose` continua recebendo `{ citizenId, neighborhoodId? }`. O `Flow` não muda nesta task: ele espalha a escolha em `citizenApi.start`, que ignora `profile` (o tsc aceita propriedade vinda de spread); a Task 7 troca isso.

- [ ] **Step 1: Ajuste os testes que digitam CPF novo e acrescente os novos**

Em `src/modules/citizen/PeopleStep.test.tsx`:

- troque a primeira linha por:

```ts
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
```

- troque `afterEach(() => vi.restoreAllMocks());` por:

```ts
beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-05T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const PROFILE = { birth_date: "1963-04-02", sex: "female", gender_identity: null };
```

- troque a função `typeNewCpf` por:

```ts
// CPF novo: o perfil vem antes do bairro (plano wpda do módulo 15, Divergência 4).
async function typeNewCpf() {
  await userEvent.type(await screen.findByLabelText("CPF de outra pessoa"), "52998224725");
  await userEvent.click(screen.getByRole("button", { name: "Continuar com este CPF" }));
  await userEvent.type(await screen.findByLabelText("Data de nascimento"), "02041963");
  await userEvent.click(screen.getByRole("radio", { name: "Feminino" }));
  await userEvent.click(screen.getByRole("button", { name: "Continuar" }));
}
```

- nas três expectativas de CPF novo, acrescente o perfil:
  - `toHaveBeenCalledWith({ cpf: "529.982.247-25", neighborhoodId: "n2" })` → `toHaveBeenCalledWith({ cpf: "529.982.247-25", profile: PROFILE, neighborhoodId: "n2" })`;
  - `expect(onChoose.mock.calls[0][0]).toEqual({ cpf: "529.982.247-25" });` → `expect(onChoose.mock.calls[0][0]).toEqual({ cpf: "529.982.247-25", profile: PROFILE });`;
  - no teste "lista com erro (api antiga ou rede)": `toHaveBeenCalledWith({ cpf: "529.982.247-25" })` → `toHaveBeenCalledWith({ cpf: "529.982.247-25", profile: PROFILE })`.

- acrescente ao fim do arquivo:

```ts
describe("PeopleStep — perfil de pessoa nova (módulo 15)", () => {
  it("CPF novo: pede data de nascimento e sexo antes do bairro, com o CPF digitado", async () => {
    const { onChoose } = setup([]);
    await userEvent.type(await screen.findByLabelText("CPF de outra pessoa"), "52998224725");
    await userEvent.click(screen.getByRole("button", { name: "Continuar com este CPF" }));

    expect(await screen.findByRole("heading", { name: "Sobre esta pessoa" })).toBeInTheDocument();
    expect(screen.getByText("CPF 529.982.247-25")).toBeInTheDocument();
    expect(screen.queryByRole("heading", { name: "Em que bairro esta pessoa mora?" })).not.toBeInTheDocument();
    expect(onChoose).not.toHaveBeenCalled();
  });

  it("CPF novo: perfil inválido não avança", async () => {
    const { onChoose } = setup([]);
    await userEvent.type(await screen.findByLabelText("CPF de outra pessoa"), "52998224725");
    await userEvent.click(screen.getByRole("button", { name: "Continuar com este CPF" }));
    await userEvent.type(await screen.findByLabelText("Data de nascimento"), "06102026");
    await userEvent.click(screen.getByRole("button", { name: "Continuar" }));

    expect(screen.getByText("A data de nascimento não pode ser depois de hoje.")).toBeInTheDocument();
    expect(screen.queryByRole("heading", { name: "Em que bairro esta pessoa mora?" })).not.toBeInTheDocument();
    expect(onChoose).not.toHaveBeenCalled();
  });

  it("CPF novo: 'Voltar' no perfil volta à lista sem chamar nada", async () => {
    const { onChoose } = setup([]);
    await userEvent.type(await screen.findByLabelText("CPF de outra pessoa"), "52998224725");
    await userEvent.click(screen.getByRole("button", { name: "Continuar com este CPF" }));
    await userEvent.click(await screen.findByRole("button", { name: "Voltar" }));

    expect(await screen.findByText("Para quem é esta triagem?")).toBeInTheDocument();
    expect(onChoose).not.toHaveBeenCalled();
  });

  it("cidade sem bairros: CPF novo começa logo depois do perfil", async () => {
    const { onChoose } = setup([], []);
    await typeNewCpf();
    await waitFor(() => expect(onChoose).toHaveBeenCalledWith({ cpf: "529.982.247-25", profile: PROFILE }));
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/PeopleStep.test.tsx`
Expected: FAIL — `Unable to find a label with the text of: Data de nascimento` (o CPF novo ainda vai direto ao bairro).

- [ ] **Step 3: Implemente**

Em `src/modules/citizen/PeopleStep.tsx`:

- troque os imports do topo por:

```tsx
import { useEffect, useRef, useState } from "react";
import { ApiError, citizenApi, type Neighborhood, type Person, type ProfileInput } from "../../lib/citizenApi";
import { isValidCpf, maskCpf } from "../../lib/masks";
import { NeighborhoodPicker } from "./NeighborhoodPicker";
import { ProfileForm } from "./ProfileForm";
import { BigButton, ErrorText, Field, INVALID_NEIGHBORHOOD_MESSAGE, Screen, messageFor } from "./ui";
import { PendingConfirmations } from "./PendingConfirmations";
```

- troque `export type Who = { citizenId: string } | { cpf: string };` por:

```tsx
// CPF novo leva o perfil junto: o par nasce com ele (contrato §3.2).
export type Who = { citizenId: string } | { cpf: string; profile: ProfileInput };
```

- troque o tipo `Mode` por:

```tsx
type Mode =
  | { at: "list" }
  | { at: "profile-for-new"; cpf: string }
  | { at: "pick-for-start"; who: Who; notice: string | null }
  | { at: "pick-for-change"; person: Person; notice: string | null };
```

- troque `submitNew` por:

```tsx
  // CPF novo: perfil primeiro, depois o bairro (o 422 de bairro continua
  // tratado em begin(), sem perder o perfil digitado).
  function submitNew() {
    if (!isValidCpf(cpf)) return setError("CPF inválido. Confira os números.");
    setError(null);
    setMode({ at: "profile-for-new", cpf });
  }
```

- logo antes de `if (mode.at === "pick-for-change") {`, acrescente:

```tsx
  if (mode.at === "profile-for-new") {
    const newCpf = mode.cpf;
    return <ProfileForm title="Sobre esta pessoa" who={`CPF ${newCpf}`} busy={busy} submitLabel="Continuar"
      onSubmit={profile => void ask({ cpf: newCpf, profile }, false)}
      onBack={() => setMode({ at: "list" })} />;
  }
```

- [ ] **Step 4: Rode e veja passar; suíte e tipos**

Run: `npx vitest run src/modules/citizen/PeopleStep.test.tsx src/modules/citizen/entry.test.tsx && npm test && npx tsc --noEmit`
Expected: PASS (o `entry.test.tsx` usa pessoa existente e não muda).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/PeopleStep.tsx src/modules/citizen/PeopleStep.test.tsx
/opt/homebrew/bin/git commit -m "feat: ask a new person's birth date and sex before the neighborhood" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Tela do catálogo

**Files:**
- Create: `src/modules/citizen/CatalogStep.tsx`
- Test: `src/modules/citizen/CatalogStep.test.tsx` (novo)

**Interfaces:**
- Consumes: `citizenApi.catalog`, `Catalog`, `ApiError` (Task 1); `fmtCalendarDate` (Task 2); `ReferenceUnits` (`src/modules/ReferenceUnits.tsx`, existente).
- Produces: `EMPTY_CATALOG_MESSAGE: string` e `CatalogStep({ citizenId, notice?, onStart, onProfileRequired, onProfile, onHistory, onBack })`, com `onStart: (protocolName: string) => Promise<string | null>` — `null` = o Flow já trocou de tela; texto = recusa a mostrar (a tela relê o catálogo).

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/citizen/CatalogStep.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, waitFor, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { CatalogStep, EMPTY_CATALOG_MESSAGE } from "./CatalogStep";
import { ApiError, citizenApi, type Catalog } from "../../lib/citizenApi";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-05T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const FULL: Catalog = {
  in_progress: null,
  suggested: [ { protocol_name: "saude-mental-aprofundada", title: "Saúde mental — aprofundamento",
    summary: "Mais perguntas sobre humor e sono.", suggestion_id: "s1", source_triage_id: "t0", source_title: "Saúde mental", suggested_on: "2026-10-02" } ],
  available: [
    { protocol_name: "saude-do-idoso", title: "Saúde do idoso", summary: "Avaliação anual de quedas, memória e medicamentos." },
    { protocol_name: "triage-respiratoria", title: "Sintomas respiratórios", summary: null }
  ],
  recent: [ { protocol_name: "saude-mental", title: "Saúde mental", summary: null,
    last_completed_on: "2026-10-02", next_available_on: "2027-03-10" } ],
  reference_units: []
};

const EMPTY: Catalog = { in_progress: null, suggested: [], available: [], recent: [], reference_units: [] };

function setup(catalog: Catalog | Error, onStartResult: string | null = null) {
  const load = vi.spyOn(citizenApi, "catalog");
  if (catalog instanceof Error) load.mockRejectedValue(catalog); else load.mockResolvedValue(catalog);
  const props = {
    onStart: vi.fn<(name: string) => Promise<string | null>>().mockResolvedValue(onStartResult),
    onProfileRequired: vi.fn(), onProfile: vi.fn(), onHistory: vi.fn(), onBack: vi.fn()
  };
  render(<CatalogStep citizenId="p1" {...props} />);
  return { load, ...props };
}

const region = (name: string) => screen.findByRole("region", { name });
const item = (title: string) => screen.getByText(title).closest("li") as HTMLElement;

describe("CatalogStep", () => {
  it("mostra as três seções na ordem do api, com a origem e as datas de calendário", async () => {
    const { load } = setup(FULL);
    const suggested = await region("Sugeridas para você");
    expect(load).toHaveBeenCalledWith("p1");
    expect(within(suggested).getByText("Saúde mental — aprofundamento")).toBeInTheDocument();
    expect(within(suggested).getByText("Sugerida pelo resultado da triagem Saúde mental de 02/10/2026")).toBeInTheDocument();

    const available = screen.getByRole("region", { name: "Disponíveis" });
    const titles = within(available).getAllByRole("listitem").map(li => li.querySelector("strong")?.textContent);
    expect(titles).toEqual([ "Saúde do idoso", "Sintomas respiratórios" ]);

    const recent = screen.getByRole("region", { name: "Feitas recentemente" });
    expect(within(recent).getByText("Feita em 02/10/2026")).toBeInTheDocument();
    expect(within(recent).getByText("Próxima a partir de 10/03/2027")).toBeInTheDocument();
    expect(within(recent).queryByRole("button")).not.toBeInTheDocument();
    expect(screen.queryByRole("region", { name: "Em andamento" })).not.toBeInTheDocument();
  });

  it("resumo ausente não vira texto", async () => {
    setup(FULL);
    await region("Disponíveis");
    expect(document.body.textContent).not.toMatch(/null|undefined/);
  });

  it("'Começar' numa disponível e numa sugerida chama onStart com o nome do protocolo", async () => {
    const { onStart } = setup(FULL);
    await region("Disponíveis");
    await userEvent.click(within(item("Saúde do idoso")).getByRole("button", { name: "Começar" }));
    expect(onStart).toHaveBeenLastCalledWith("saude-do-idoso");
    await userEvent.click(within(item("Saúde mental — aprofundamento")).getByRole("button", { name: "Começar" }));
    expect(onStart).toHaveBeenLastCalledWith("saude-mental-aprofundada");
  });

  it("'Em andamento' aparece primeiro, com 'Continuar'", async () => {
    const { onStart } = setup({ ...FULL,
      in_progress: { conversation_id: "c1", protocol_name: "triage-respiratoria", title: "Sintomas respiratórios" } });
    const inProgress = await region("Em andamento");
    await userEvent.click(within(inProgress).getByRole("button", { name: "Continuar" }));
    expect(onStart).toHaveBeenCalledWith("triage-respiratoria");
    const headings = screen.getAllByRole("heading", { level: 2 }).map(h => h.textContent);
    expect(headings[0]).toBe("Em andamento");
  });

  it("toque duplo em 'Começar' chama uma vez só", async () => {
    const { onStart } = setup(FULL);
    onStart.mockReturnValue(new Promise<string | null>(() => {}));
    await region("Disponíveis");
    const btn = within(item("Saúde do idoso")).getByRole("button", { name: "Começar" });
    await userEvent.click(btn);
    await userEvent.click(btn);
    expect(onStart).toHaveBeenCalledTimes(1);
    expect(btn).toBeDisabled();
  });

  it("recusa ao começar: mostra o aviso e relê o catálogo", async () => {
    const { load } = setup(FULL, "Esta triagem não está mais disponível para esta pessoa.");
    await region("Disponíveis");
    await userEvent.click(within(item("Saúde do idoso")).getByRole("button", { name: "Começar" }));
    expect(await screen.findByText("Esta triagem não está mais disponível para esta pessoa.")).toBeInTheDocument();
    await waitFor(() => expect(load).toHaveBeenCalledTimes(2));
  });

  it("aviso vindo do Flow aparece no topo", async () => {
    vi.spyOn(citizenApi, "catalog").mockResolvedValue(FULL);
    render(<CatalogStep citizenId="p1" notice="Perfil atualizado." onStart={vi.fn()} onProfileRequired={vi.fn()}
      onProfile={vi.fn()} onHistory={vi.fn()} onBack={vi.fn()} />);
    expect(await screen.findByRole("status")).toHaveTextContent("Perfil atualizado.");
  });

  it("catálogo vazio: mensagem neutra e a unidade de referência", async () => {
    setup({ ...EMPTY, reference_units: [ { id: "u1", name: "UBS Batel", kind: "ubs",
      address: { street: "Rua Padre Anchieta", number: "1500", complement: null, zip: "80730000" } } ] });
    expect(await screen.findByText(EMPTY_CATALOG_MESSAGE)).toBeInTheDocument();
    expect(screen.getByRole("heading", { name: "Sua unidade de referência" })).toBeInTheDocument();
    expect(screen.getByText("UBS Batel")).toBeInTheDocument();
  });

  it("catálogo vazio sem unidade (sem bairro ou api sem o campo): só a mensagem", async () => {
    setup(EMPTY);
    expect(await screen.findByText(EMPTY_CATALOG_MESSAGE)).toBeInTheDocument();
    expect(screen.queryByText(/unidade de referência|unidades de referência/)).not.toBeInTheDocument();
  });

  it("só 'Feitas recentemente' também é vazio para começar: mensagem e a lista das feitas", async () => {
    setup({ ...EMPTY, recent: FULL.recent });
    expect(await screen.findByText(EMPTY_CATALOG_MESSAGE)).toBeInTheDocument();
    expect(screen.getByRole("region", { name: "Feitas recentemente" })).toBeInTheDocument();
  });

  it("409 profile_required leva ao perfil", async () => {
    const { onProfileRequired } = setup(new ApiError(409, "profile_required"));
    await waitFor(() => expect(onProfileRequired).toHaveBeenCalledTimes(1));
    expect(screen.queryByRole("alert")).not.toBeInTheDocument();
  });

  it("outro erro mostra a mensagem", async () => {
    setup(new ApiError(404, "not_found"));
    expect(await screen.findByText("Algo deu errado. Tente de novo.")).toBeInTheDocument();
  });

  it("rodapé: Meu perfil, Minhas triagens e Voltar", async () => {
    const { onProfile, onHistory, onBack } = setup(FULL);
    await region("Disponíveis");
    await userEvent.click(screen.getByRole("button", { name: "Meu perfil" }));
    await userEvent.click(screen.getByRole("button", { name: "Minhas triagens" }));
    await userEvent.click(screen.getByRole("button", { name: "Voltar" }));
    expect(onProfile).toHaveBeenCalled();
    expect(onHistory).toHaveBeenCalled();
    expect(onBack).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/CatalogStep.test.tsx`
Expected: FAIL — `Failed to resolve import "./CatalogStep"`.

- [ ] **Step 3: Implemente**

```tsx
// src/modules/citizen/CatalogStep.tsx
// Catálogo de triagens de uma pessoa (spec 2026-10-05 §8.3 e §8.7; ADR 0027;
// contrato §3.4). Mostra só o que o api ofereceu, na ordem do api: a regra de
// oferta mora lá. Datas de calendário sem new Date (ver fmtCalendarDate).
import { useEffect, useRef, useState, type ReactNode } from "react";
import { ApiError, citizenApi, type Catalog } from "../../lib/citizenApi";
import { fmtCalendarDate } from "../../lib/format";
import { ReferenceUnits } from "../ReferenceUnits";
import { BigButton, ErrorText, Screen, messageFor } from "./ui";

export const EMPTY_CATALOG_MESSAGE =
  "Não há triagens disponíveis para esta pessoa agora. Se precisar de atendimento, procure uma unidade de saúde.";

function Section({ id, title, children }: { id: string; title: string; children: ReactNode }) {
  return (
    <section aria-labelledby={id} style={{ marginBottom: 24 }}>
      <h2 id={id} style={{ fontSize: 20, margin: "0 0 8px" }}>{title}</h2>
      <ul style={{ listStyle: "none", margin: 0, padding: 0, display: "grid", gap: 12 }}>{children}</ul>
    </section>
  );
}

function Item({ title, summary, lines = [], action }:
  { title: string; summary: string | null; lines?: string[]; action?: ReactNode }) {
  return (
    <li style={{ border: "1px solid var(--rule2, #ccc)", borderRadius: 12, padding: 12, display: "grid", gap: 8 }}>
      <strong style={{ fontSize: 18 }}>{title}</strong>
      {summary && <p style={{ margin: 0, fontSize: 18 }}>{summary}</p>}
      {lines.map(l => <p key={l} style={{ margin: 0, fontSize: 18, color: "var(--ink2, #555)" }}>{l}</p>)}
      {action}
    </li>
  );
}

export function CatalogStep({ citizenId, notice, onStart, onProfileRequired, onProfile, onHistory, onBack }: {
  citizenId: string; notice?: string | null;
  onStart: (protocolName: string) => Promise<string | null>;
  onProfileRequired: () => void; onProfile: () => void; onHistory: () => void; onBack: () => void;
}) {
  const [catalog, setCatalog] = useState<Catalog | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [message, setMessage] = useState<string | null>(notice ?? null);
  const [busy, setBusy] = useState(false);
  // Guarda síncrona: setBusy só vale no próximo render, e dois toques cabem antes dele.
  const inFlight = useRef(false);
  const alive = useRef(true);
  // O callback muda a cada render do Flow; a ref evita reler o catálogo por isso.
  const profileRequired = useRef(onProfileRequired);
  profileRequired.current = onProfileRequired;

  function load() {
    citizenApi.catalog(citizenId)
      .then(c => { if (alive.current) { setCatalog(c); setError(null); } })
      .catch(e => {
        if (!alive.current) return;
        if (e instanceof ApiError && e.code === "profile_required") profileRequired.current();
        else setError(messageFor(e));
      });
  }

  useEffect(() => {
    alive.current = true;
    load();
    return () => { alive.current = false; };
  }, [citizenId]);

  async function start(protocolName: string) {
    if (inFlight.current) return;
    inFlight.current = true;
    setBusy(true);
    setMessage(null);
    try {
      const refused = await onStart(protocolName);
      if (refused && alive.current) {
        setMessage(refused);
        load();
      }
    } finally {
      inFlight.current = false;
      if (alive.current) setBusy(false);
    }
  }

  const startButton = (protocolName: string, label = "Começar") =>
    <BigButton disabled={busy} onClick={() => void start(protocolName)}>{label}</BigButton>;

  const c = catalog;
  const nothingToStart = c !== null && c.in_progress === null && c.suggested.length === 0 && c.available.length === 0;

  return (
    <Screen title="Qual triagem fazer?" footer={<>
      <BigButton variant="secondary" onClick={onProfile}>Meu perfil</BigButton>
      <BigButton variant="secondary" onClick={onHistory}>Minhas triagens</BigButton>
      <BigButton variant="secondary" onClick={onBack}>Voltar</BigButton>
    </>}>
      {message && <p role="status">{message}</p>}
      {error && <ErrorText>{error}</ErrorText>}
      {c === null && !error && <p>Carregando…</p>}
      {c?.in_progress && (
        <Section id="catalog-in-progress" title="Em andamento">
          <Item title={c.in_progress.title} summary={null} action={startButton(c.in_progress.protocol_name, "Continuar")} />
        </Section>
      )}
      {c && c.suggested.length > 0 && (
        <Section id="catalog-suggested" title="Sugeridas para você">
          {c.suggested.map(s => (
            <Item key={s.suggestion_id} title={s.title} summary={s.summary}
              lines={[ `Sugerida pelo resultado da triagem ${s.source_title} de ${fmtCalendarDate(s.suggested_on)}` ]}
              action={startButton(s.protocol_name)} />
          ))}
        </Section>
      )}
      {c && c.available.length > 0 && (
        <Section id="catalog-available" title="Disponíveis">
          {c.available.map(a => (
            <Item key={a.protocol_name} title={a.title} summary={a.summary} action={startButton(a.protocol_name)} />
          ))}
        </Section>
      )}
      {c && nothingToStart && (
        <>
          <p>{EMPTY_CATALOG_MESSAGE}</p>
          <ReferenceUnits units={c.reference_units} />
        </>
      )}
      {c && c.recent.length > 0 && (
        <Section id="catalog-recent" title="Feitas recentemente">
          {c.recent.map(r => (
            <Item key={r.protocol_name} title={r.title} summary={r.summary}
              lines={[ `Feita em ${fmtCalendarDate(r.last_completed_on)}`,
                `Próxima a partir de ${fmtCalendarDate(r.next_available_on)}` ]} />
          ))}
        </Section>
      )}
    </Screen>
  );
}
```

- [ ] **Step 4: Rode e veja passar; suíte e tipos**

Run: `npx vitest run src/modules/citizen/CatalogStep.test.tsx && npm test && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/CatalogStep.tsx src/modules/citizen/CatalogStep.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the citizen triage catalog screen" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Perfil obrigatório e "Meu perfil"

**Files:**
- Create: `src/modules/citizen/ProfileStep.tsx`
- Test: `src/modules/citizen/ProfileStep.test.tsx` (novo)

**Interfaces:**
- Consumes: `citizenApi.people`, `citizenApi.setProfile`, `Person`, `ProfileInput`, `ApiError` (Task 1); `fmtCalendarDate`, `sexLabel`, `genderIdentityLabel` (Task 2); `ProfileForm` (Task 3).
- Produces: `VERIFIED_PROFILE_TEXT: string` e `ProfileStep({ citizenId, required, onSaved, onBack })`. `required: true` = vindo de `profile_required` (título "Sobre esta pessoa", botão "Continuar"); `false` = "Meu perfil" (botão "Salvar"). `onSaved()` é chamado depois do 200.

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/citizen/ProfileStep.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { ProfileStep, VERIFIED_PROFILE_TEXT } from "./ProfileStep";
import { ApiError, citizenApi, type Person } from "../../lib/citizenApi";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-05T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const declared: Person = {
  id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared", neighborhood: null,
  profile: { birth_date: "1963-04-02", sex: "female", gender_identity: null, profile_source: "declared" }
};
const verified: Person = {
  ...declared, verification_level: "verified",
  profile: { birth_date: "1963-04-02", sex: "female", gender_identity: "cis_woman", profile_source: "verified" }
};
const noProfile: Person = { ...declared, profile: null };

function setup(person: Person, required = false) {
  vi.spyOn(citizenApi, "people").mockResolvedValue({ people: [ person ] });
  const onSaved = vi.fn();
  const onBack = vi.fn();
  render(<ProfileStep citizenId="p1" required={required} onSaved={onSaved} onBack={onBack} />);
  return { onSaved, onBack };
}

describe("ProfileStep", () => {
  it("declared: 'Meu perfil' vem preenchido, salva e avisa o Flow", async () => {
    const set = vi.spyOn(citizenApi, "setProfile").mockResolvedValue(declared);
    const { onSaved } = setup(declared);
    expect(await screen.findByRole("heading", { name: "Meu perfil" })).toBeInTheDocument();
    expect(screen.getByText("CPF ***.982.247-**")).toBeInTheDocument();
    const birth = screen.getByLabelText("Data de nascimento");
    expect(birth).toHaveValue("02/04/1963");
    await userEvent.clear(birth);
    await userEvent.type(birth, "03041963");
    await userEvent.click(screen.getByRole("button", { name: "Salvar" }));
    await waitFor(() => expect(onSaved).toHaveBeenCalledTimes(1));
    expect(set).toHaveBeenCalledWith("p1", { birth_date: "1963-04-03", sex: "female", gender_identity: null });
  });

  it("sem perfil (obrigatório): 'Sobre esta pessoa', campos vazios e 'Continuar'", async () => {
    const set = vi.spyOn(citizenApi, "setProfile").mockResolvedValue(declared);
    const { onSaved } = setup(noProfile, true);
    expect(await screen.findByRole("heading", { name: "Sobre esta pessoa" })).toBeInTheDocument();
    expect(screen.getByLabelText("Data de nascimento")).toHaveValue("");
    await userEvent.type(screen.getByLabelText("Data de nascimento"), "02041963");
    await userEvent.click(screen.getByRole("radio", { name: "Feminino" }));
    await userEvent.click(screen.getByRole("button", { name: "Continuar" }));
    await waitFor(() => expect(onSaved).toHaveBeenCalled());
    expect(set).toHaveBeenCalledWith("p1", { birth_date: "1963-04-02", sex: "female", gender_identity: null });
  });

  it("verified: mostra o perfil conferido, sem campos nem 'Salvar'", async () => {
    const set = vi.spyOn(citizenApi, "setProfile");
    setup(verified);
    expect(await screen.findByText(VERIFIED_PROFILE_TEXT)).toBeInTheDocument();
    expect(screen.getByText("02/04/1963")).toBeInTheDocument();
    expect(screen.getByText("Feminino")).toBeInTheDocument();
    expect(screen.getByText("Mulher cis")).toBeInTheDocument();
    expect(screen.queryByLabelText("Data de nascimento")).not.toBeInTheDocument();
    expect(screen.queryByRole("radio")).not.toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Salvar" })).not.toBeInTheDocument();
    expect(set).not.toHaveBeenCalled();
  });

  it("409 profile_verified (conferido no posto entre a leitura e o envio): relê e mostra só leitura com aviso", async () => {
    vi.spyOn(citizenApi, "people")
      .mockResolvedValueOnce({ people: [ declared ] })
      .mockResolvedValue({ people: [ verified ] });
    vi.spyOn(citizenApi, "setProfile").mockRejectedValue(new ApiError(409, "profile_verified"));
    const onSaved = vi.fn();
    render(<ProfileStep citizenId="p1" required={false} onSaved={onSaved} onBack={vi.fn()} />);
    await userEvent.click(await screen.findByRole("button", { name: "Salvar" }));
    expect(await screen.findByText(VERIFIED_PROFILE_TEXT)).toBeInTheDocument();
    expect(screen.getByText("Este perfil foi conferido no posto e só pode ser corrigido lá.")).toBeInTheDocument();
    expect(onSaved).not.toHaveBeenCalled();
  });

  it("422 do api aparece no formulário, sem sair da tela", async () => {
    vi.spyOn(citizenApi, "setProfile").mockRejectedValue(new ApiError(422, "invalid_birth_date"));
    const { onSaved } = setup(declared);
    await userEvent.click(await screen.findByRole("button", { name: "Salvar" }));
    expect(await screen.findByText("Data de nascimento inválida. Confira dia, mês e ano.")).toBeInTheDocument();
    expect(screen.getByLabelText("Data de nascimento")).toBeInTheDocument();
    expect(onSaved).not.toHaveBeenCalled();
  });

  it("toque duplo em 'Salvar' grava uma vez só", async () => {
    let finish!: (p: Person) => void;
    const set = vi.spyOn(citizenApi, "setProfile").mockReturnValue(new Promise(r => { finish = r; }));
    setup(declared);
    const btn = await screen.findByRole("button", { name: "Salvar" });
    await userEvent.click(btn);
    await userEvent.click(btn);
    finish(declared);
    expect(set).toHaveBeenCalledTimes(1);
  });

  it("pessoa fora da sessão: mensagem e 'Voltar'", async () => {
    vi.spyOn(citizenApi, "people").mockResolvedValue({ people: [] });
    const onBack = vi.fn();
    render(<ProfileStep citizenId="p1" required={false} onSaved={vi.fn()} onBack={onBack} />);
    expect(await screen.findByText("Algo deu errado. Tente de novo.")).toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Voltar" }));
    expect(onBack).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/ProfileStep.test.tsx`
Expected: FAIL — `Failed to resolve import "./ProfileStep"`.

- [ ] **Step 3: Implemente**

```tsx
// src/modules/citizen/ProfileStep.tsx
// Perfil obrigatório antes do catálogo e "Meu perfil" (spec 2026-10-05 §8.2 e
// §8.6; ADR 0027). declared: o cidadão corrige. verified: conferido no posto,
// só leitura; daí em diante só muda lá (409 profile_verified).
import { useEffect, useRef, useState } from "react";
import { ApiError, citizenApi, type Person, type ProfileInput } from "../../lib/citizenApi";
import { fmtCalendarDate } from "../../lib/format";
import { genderIdentityLabel, sexLabel } from "../../lib/profile";
import { ProfileForm } from "./ProfileForm";
import { BigButton, ErrorText, Screen, messageFor } from "./ui";

export const VERIFIED_PROFILE_TEXT =
  "Conferido no posto. Para corrigir, procure uma unidade de saúde com um documento com foto.";

export function ProfileStep({ citizenId, required, onSaved, onBack }:
  { citizenId: string; required: boolean; onSaved: () => void; onBack: () => void }) {
  const [person, setPerson] = useState<Person | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [busy, setBusy] = useState(false);
  const inFlight = useRef(false);

  function apply(people: Person[]) {
    const p = people.find(x => x.id === citizenId);
    if (p) setPerson(p);
    else setError(messageFor(new ApiError(404, "not_found")));
  }

  useEffect(() => {
    let alive = true;
    citizenApi.people()
      .then(r => { if (alive) apply(r.people); })
      .catch(e => { if (alive) setError(messageFor(e)); });
    return () => { alive = false; };
  }, [citizenId]);

  async function save(input: ProfileInput) {
    if (inFlight.current) return;
    inFlight.current = true;
    setBusy(true);
    setError(null);
    try {
      await citizenApi.setProfile(citizenId, input);
      onSaved();
    } catch (e) {
      // Conferido no posto entre a leitura e o envio: relê e mostra só leitura.
      if (e instanceof ApiError && e.code === "profile_verified") {
        await citizenApi.people().then(r => apply(r.people)).catch(() => undefined);
      }
      setError(messageFor(e));
    } finally {
      inFlight.current = false;
      setBusy(false);
    }
  }

  const title = required ? "Sobre esta pessoa" : "Meu perfil";
  const back = <BigButton variant="secondary" onClick={onBack}>Voltar</BigButton>;

  if (!person) {
    return <Screen title={title} footer={back}>{error ? <ErrorText>{error}</ErrorText> : <p>Carregando…</p>}</Screen>;
  }

  const profile = person.profile ?? null;
  if (profile && profile.profile_source === "verified") {
    return (
      <Screen title="Meu perfil" footer={back}>
        <p>CPF {person.cpf_masked}</p>
        {error && <ErrorText>{error}</ErrorText>}
        <dl style={{ display: "grid", gap: 4, margin: "0 0 16px", fontSize: 18 }}>
          <dt style={{ fontWeight: 600 }}>Data de nascimento</dt>
          <dd style={{ margin: "0 0 8px" }}>{fmtCalendarDate(profile.birth_date)}</dd>
          <dt style={{ fontWeight: 600 }}>Sexo</dt>
          <dd style={{ margin: "0 0 8px" }}>{sexLabel(profile.sex)}</dd>
          <dt style={{ fontWeight: 600 }}>Identidade de gênero</dt>
          <dd style={{ margin: 0 }}>{genderIdentityLabel(profile.gender_identity)}</dd>
        </dl>
        <p style={{ padding: 12, borderRadius: 12, border: "1px solid var(--line, #eee)" }}>{VERIFIED_PROFILE_TEXT}</p>
      </Screen>
    );
  }

  return <ProfileForm key={person.id} title={title} who={`CPF ${person.cpf_masked}`} initial={profile}
    busy={busy} error={error} submitLabel={required ? "Continuar" : "Salvar"}
    onSubmit={input => void save(input)} onBack={onBack} />;
}
```

- [ ] **Step 4: Rode e veja passar; suíte e tipos**

Run: `npx vitest run src/modules/citizen/ProfileStep.test.tsx && npm test && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/ProfileStep.tsx src/modules/citizen/ProfileStep.test.tsx
/opt/homebrew/bin/git commit -m "feat: let citizens fill and correct a declared profile" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Fluxo pessoa → perfil → catálogo → triagem escolhida

**Files:**
- Modify: `src/modules/citizen/Flow.tsx`
- Modify: `src/lib/citizenApi.ts` (remove `start`)
- Test: `src/modules/citizen/Flow.catalog.test.tsx` (novo), `src/modules/citizen/Flow.test.tsx`, `src/lib/citizenApi.test.ts`

**Interfaces:**
- Consumes: `createPerson`, `setProfile`, `setNeighborhood`, `catalog`, `startTriage` (Task 1); `PersonChoice` com `profile` (Task 4); `CatalogStep` (Task 5); `ProfileStep` (Task 6).
- Produces (no `Flow`):
  - estados `{ at: "catalog"; consentVersion: string; citizenId: string; notice: string | null }` e `{ at: "profile"; consentVersion: string; citizenId: string; required: boolean }`;
  - `startTriage(consentVersion: string, citizenId: string, protocolName: string): Promise<string | null>` (nesta task, qualquer erro devolve `messageFor(e)`; a Task 8 trata as recusas).
- `citizenApi.start` deixa de existir.

- [ ] **Step 1: Escreva os testes do fluxo**

```tsx
// src/modules/citizen/Flow.catalog.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Flow } from "./Flow";
import { citizenApi, ApiError, type Catalog, type Person, type Step } from "../../lib/citizenApi";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-05T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); window.history.replaceState(null, "", "/"); });

const PROFILE_IN = { birth_date: "1963-04-02", sex: "female" as const, gender_identity: null };
const avo: Person = {
  id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared", neighborhood: null,
  profile: { ...PROFILE_IN, profile_source: "declared" }
};
const semPerfil: Person = { ...avo, profile: null };

const CATALOG: Catalog = {
  in_progress: null, suggested: [],
  available: [ { protocol_name: "saude-do-idoso", title: "Saúde do idoso", summary: "Avaliação anual de quedas, memória e medicamentos." } ],
  recent: [], reference_units: []
};

const step: Step = {
  triage_id: "t1", step_id: "quedas", prompt: "Você caiu nos últimos 12 meses?", answer_type: "boolean",
  options: [ { id: "true", title: "Sim" }, { id: "false", title: "Não" } ], index: 1, total: 3, can_undo: false
};

function signedIn(people: Person[]) {
  vi.spyOn(citizenApi, "currentSession").mockResolvedValue({ phone_masked: "(**) *****-5432" });
  vi.spyOn(citizenApi, "consentTerm").mockResolvedValue({ version: "1", body: "Termo" });
  vi.spyOn(citizenApi, "people").mockResolvedValue({ people });
  vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue([]);
  vi.spyOn(citizenApi, "appointments").mockResolvedValue({ appointments: [] });
  vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [], unread_count: 0 });
}

async function choosePerson() {
  render(<Flow />);
  await userEvent.click(await screen.findByRole("button", { name: "Concordo" }));
  await userEvent.click(await screen.findByRole("button", { name: "CPF ***.982.247-**" }));
}

async function typeNewCpfAndProfile() {
  render(<Flow />);
  await userEvent.click(await screen.findByRole("button", { name: "Concordo" }));
  await userEvent.type(await screen.findByLabelText("CPF de outra pessoa"), "52998224725");
  await userEvent.click(screen.getByRole("button", { name: "Continuar com este CPF" }));
  await fillProfile();
  await userEvent.click(screen.getByRole("button", { name: "Continuar" }));
}

async function fillProfile() {
  await userEvent.type(await screen.findByLabelText("Data de nascimento"), "02041963");
  await userEvent.click(screen.getByRole("radio", { name: "Feminino" }));
}

const findItem = async (title: string) => (await screen.findByText(title)).closest("li") as HTMLElement;

describe("Flow — catálogo e início por protocolo", () => {
  it("pessoa com perfil: catálogo e início pela triagem escolhida", async () => {
    signedIn([ avo ]);
    const catalog = vi.spyOn(citizenApi, "catalog").mockResolvedValue(CATALOG);
    const start = vi.spyOn(citizenApi, "startTriage")
      .mockResolvedValue({ conversation_id: "c1", citizen_id: "p1", resumed: false, step });
    await choosePerson();

    await userEvent.click(within(await findItem("Saúde do idoso")).getByRole("button", { name: "Começar" }));

    expect(await screen.findByText("Você caiu nos últimos 12 meses?")).toBeInTheDocument();
    expect(catalog).toHaveBeenCalledWith("p1");
    expect(start).toHaveBeenCalledWith({ citizenId: "p1", protocolName: "saude-do-idoso", consentVersion: "1" });
  });

  it("pessoa sem perfil: o catálogo pede o perfil (409) e, salvo, o catálogo aparece", async () => {
    signedIn([ semPerfil ]);
    const catalog = vi.spyOn(citizenApi, "catalog")
      .mockRejectedValueOnce(new ApiError(409, "profile_required")).mockResolvedValue(CATALOG);
    const set = vi.spyOn(citizenApi, "setProfile").mockResolvedValue(avo);
    await choosePerson();

    expect(await screen.findByRole("heading", { name: "Sobre esta pessoa" })).toBeInTheDocument();
    await fillProfile();
    await userEvent.click(screen.getByRole("button", { name: "Continuar" }));

    expect(await screen.findByRole("region", { name: "Disponíveis" })).toBeInTheDocument();
    expect(set).toHaveBeenCalledWith("p1", PROFILE_IN);
    expect(catalog).toHaveBeenCalledTimes(2);
  });

  it("perfil obrigatório: 'Voltar' volta à escolha de pessoa", async () => {
    signedIn([ semPerfil ]);
    vi.spyOn(citizenApi, "catalog").mockRejectedValue(new ApiError(409, "profile_required"));
    await choosePerson();
    await screen.findByRole("heading", { name: "Sobre esta pessoa" });
    await userEvent.click(screen.getByRole("button", { name: "Voltar" }));
    expect(await screen.findByText("Para quem é esta triagem?")).toBeInTheDocument();
  });

  it("CPF novo: o par nasce com o perfil (POST /citizen/people) e o catálogo é o dele", async () => {
    signedIn([]);
    const create = vi.spyOn(citizenApi, "createPerson").mockResolvedValue({ ...avo, id: "p9" });
    const set = vi.spyOn(citizenApi, "setProfile");
    const catalog = vi.spyOn(citizenApi, "catalog").mockResolvedValue(CATALOG);
    await typeNewCpfAndProfile();

    expect(await screen.findByRole("region", { name: "Disponíveis" })).toBeInTheDocument();
    expect(create).toHaveBeenCalledWith({ cpf: "529.982.247-25", consentVersion: "1", profile: PROFILE_IN });
    expect("neighborhoodId" in (create.mock.calls[0][0] as object)).toBe(false);
    expect(set).not.toHaveBeenCalled();
    expect(catalog).toHaveBeenCalledWith("p9");
  });

  it("CPF que já existia sem perfil (200): grava o perfil informado e segue", async () => {
    signedIn([]);
    vi.spyOn(citizenApi, "createPerson").mockResolvedValue({ ...semPerfil, id: "p9" });
    const set = vi.spyOn(citizenApi, "setProfile").mockResolvedValue({ ...avo, id: "p9" });
    vi.spyOn(citizenApi, "catalog").mockResolvedValue(CATALOG);
    await typeNewCpfAndProfile();

    expect(await screen.findByRole("region", { name: "Disponíveis" })).toBeInTheDocument();
    expect(set).toHaveBeenCalledWith("p9", PROFILE_IN);
  });

  it("CPF que já existia com perfil (200): não sobrescreve e segue", async () => {
    signedIn([]);
    vi.spyOn(citizenApi, "createPerson").mockResolvedValue({ ...avo, id: "p9" });
    const set = vi.spyOn(citizenApi, "setProfile");
    vi.spyOn(citizenApi, "catalog").mockResolvedValue(CATALOG);
    await typeNewCpfAndProfile();

    expect(await screen.findByRole("region", { name: "Disponíveis" })).toBeInTheDocument();
    expect(set).not.toHaveBeenCalled();
  });

  it("CPF novo com termo vencido (409 consent_outdated): volta ao termo", async () => {
    signedIn([]);
    vi.spyOn(citizenApi, "createPerson").mockRejectedValue(new ApiError(409, "consent_outdated"));
    await typeNewCpfAndProfile();
    expect(await screen.findByRole("button", { name: "Concordo" })).toBeInTheDocument();
  });

  it("'Meu perfil' no catálogo: corrige e volta ao catálogo com aviso", async () => {
    signedIn([ avo ]);
    vi.spyOn(citizenApi, "catalog").mockResolvedValue(CATALOG);
    const set = vi.spyOn(citizenApi, "setProfile").mockResolvedValue(avo);
    await choosePerson();

    await userEvent.click(await screen.findByRole("button", { name: "Meu perfil" }));
    const birth = await screen.findByLabelText("Data de nascimento");
    await userEvent.clear(birth);
    await userEvent.type(birth, "03041963");
    await userEvent.click(screen.getByRole("button", { name: "Salvar" }));

    expect(await screen.findByRole("status")).toHaveTextContent("Perfil atualizado.");
    expect(set).toHaveBeenCalledWith("p1", { ...PROFILE_IN, birth_date: "1963-04-03" });
  });

  it("'Em andamento': 'Continuar' retoma pela triagem aberta", async () => {
    signedIn([ avo ]);
    vi.spyOn(citizenApi, "catalog").mockResolvedValue({ ...CATALOG,
      in_progress: { conversation_id: "c1", protocol_name: "triage-respiratoria", title: "Sintomas respiratórios" } });
    const start = vi.spyOn(citizenApi, "startTriage")
      .mockResolvedValue({ conversation_id: "c1", citizen_id: "p1", resumed: true, step });
    await choosePerson();

    await userEvent.click(within(await screen.findByRole("region", { name: "Em andamento" }))
      .getByRole("button", { name: "Continuar" }));

    expect(await screen.findByText("Você caiu nos últimos 12 meses?")).toBeInTheDocument();
    expect(start).toHaveBeenCalledWith({ citizenId: "p1", protocolName: "triage-respiratoria", consentVersion: "1" });
  });

  it("'Voltar' no catálogo volta à escolha de pessoa", async () => {
    signedIn([ avo ]);
    vi.spyOn(citizenApi, "catalog").mockResolvedValue(CATALOG);
    await choosePerson();
    await screen.findByRole("region", { name: "Disponíveis" });
    await userEvent.click(screen.getByRole("button", { name: "Voltar" }));
    expect(await screen.findByText("Para quem é esta triagem?")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Ajuste `Flow.test.tsx` ao início por protocolo**

Em `src/modules/citizen/Flow.test.tsx`:

- na função `reachQuestionStep`, troque o bloco `vi.spyOn(citizenApi, "start").mockResolvedValue({ ... });` por:

```ts
  vi.spyOn(citizenApi, "catalog").mockResolvedValue({
    in_progress: null, suggested: [], recent: [], reference_units: [],
    available: [ { protocol_name: "triage-respiratoria", title: "Sintomas respiratórios", summary: null } ]
  });
  vi.spyOn(citizenApi, "startTriage").mockResolvedValue({
    conversation_id: "c1", citizen_id: "p1", resumed: false, step: boolStep
  });
```

  e, logo depois de `await userEvent.click(await screen.findByRole("button", { name: "CPF ***.982.247-**" }));`, acrescente:

```ts
  await userEvent.click(await screen.findByRole("button", { name: "Começar" }));
```

- troque o teste inteiro `it("422 invalid_neighborhood ao começar: sem erro genérico, pede o bairro de novo e segue", ...)` por:

```ts
  // Módulo 15: o bairro de quem já existe vai por POST /citizen/people/:id/neighborhood
  // antes do catálogo (plano wpda, Divergência 3).
  it("422 invalid_neighborhood ao gravar o bairro: sem erro genérico, pede o bairro de novo e segue ao catálogo", async () => {
    vi.spyOn(citizenApi, "currentSession").mockResolvedValue({ phone_masked: "(**) *****-5432" });
    vi.spyOn(citizenApi, "consentTerm").mockResolvedValue({ version: "1", body: "Termo" });
    vi.spyOn(citizenApi, "people").mockResolvedValue({
      people: [{ id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared", neighborhood: null }]
    });
    vi.spyOn(citizenApi, "neighborhoods")
      .mockResolvedValueOnce([ { id: "n1", name: "Batel" }, { id: "n2", name: "São Francisco" } ])
      .mockResolvedValueOnce([ { id: "n2", name: "São Francisco" } ]);
    const set = vi.spyOn(citizenApi, "setNeighborhood")
      .mockRejectedValueOnce(new ApiError(422, "invalid_neighborhood"))
      .mockResolvedValue({});
    vi.spyOn(citizenApi, "catalog").mockResolvedValue({
      in_progress: null, suggested: [], recent: [], reference_units: [],
      available: [ { protocol_name: "triage-respiratoria", title: "Sintomas respiratórios", summary: null } ]
    });

    render(<Flow />);
    await userEvent.click(await screen.findByRole("button", { name: "Concordo" }));
    await userEvent.click(await screen.findByRole("button", { name: "CPF ***.982.247-**" }));
    await userEvent.click(await screen.findByRole("button", { name: "Batel" }));

    expect(await screen.findByText("Esse bairro não está mais na lista. Escolha de novo.")).toBeInTheDocument();
    expect(screen.queryByText("Algo deu errado. Tente de novo.")).not.toBeInTheDocument();
    expect(set).toHaveBeenNthCalledWith(1, "p1", "n1");

    await userEvent.click(screen.getByRole("button", { name: "São Francisco" }));
    expect(await screen.findByRole("region", { name: "Disponíveis" })).toBeInTheDocument();
    expect(set).toHaveBeenNthCalledWith(2, "p1", "n2");
  });
```

- [ ] **Step 3: Ajuste `citizenApi.test.ts` à remoção de `start`**

Em `src/lib/citizenApi.test.ts`, apague os três testes que chamam `citizenApi.start` (o início agora é `startTriage`, testado na Task 1):
- `it("start manda citizen_id ou cpf e a versão do termo", ...)`;
- `it("start manda neighborhood_id quando veio", ...)`;
- `it("start sem bairro não manda a chave neighborhood_id", ...)`.

- [ ] **Step 4: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/Flow.catalog.test.tsx src/modules/citizen/Flow.test.tsx`
Expected: FAIL — depois de escolher a pessoa o `Flow` ainda chama `citizenApi.start` (sem mock nos testes novos), e não existe região "Disponíveis".

- [ ] **Step 5: Implemente no `Flow`**

Em `src/modules/citizen/Flow.tsx`:

- acrescente aos imports, depois de `import { PeopleStep, ... } from "./PeopleStep";`:

```tsx
import { CatalogStep } from "./CatalogStep";
import { ProfileStep } from "./ProfileStep";
```

- no tipo `State`, depois de `| { at: "people"; consentVersion: string }`, acrescente:

```tsx
  | { at: "catalog"; consentVersion: string; citizenId: string; notice: string | null }
  | { at: "profile"; consentVersion: string; citizenId: string; required: boolean }
```

- troque a função `choose` inteira por:

```tsx
  // "Para quem é?" (spec 2026-10-05 §8): pessoa nova nasce com o perfil
  // (POST /citizen/people); quem já existe grava o bairro escolhido, se houver.
  // Os dois seguem para o catálogo, que pede o perfil se faltar (409).
  async function choose(consentVersion: string, choice: PersonChoice): Promise<ChooseOutcome> {
    setError(null);
    try {
      let citizenId: string;
      if ("cpf" in choice) {
        const person = await citizenApi.createPerson({
          cpf: choice.cpf, consentVersion, profile: choice.profile,
          ...(choice.neighborhoodId ? { neighborhoodId: choice.neighborhoodId } : {})
        });
        // 200 = o par já existia (contrato §3.2): sem perfil, grava o informado agora.
        if (!person.profile) await citizenApi.setProfile(person.id, choice.profile);
        citizenId = person.id;
      } else {
        if (choice.neighborhoodId) await citizenApi.setNeighborhood(choice.citizenId, choice.neighborhoodId);
        citizenId = choice.citizenId;
      }
      setState({ at: "catalog", consentVersion, citizenId, notice: null });
    } catch (e) {
      if (e instanceof ApiError && (e.code === "consent_outdated" || e.code === "no_consent")) {
        setState({ at: "consent" });
        return "done";
      }
      // O bairro saiu da lista entre a leitura e o envio: a PeopleStep relê e
      // pergunta de novo (spec 2026-09-28 §6), sem o erro genérico.
      if (e instanceof ApiError && e.code === "invalid_neighborhood") return "invalid_neighborhood";
      setError(messageFor(e));
    }
    return "done";
  }

  // Começa (ou retoma) a triagem escolhida no catálogo ou sugerida no
  // resultado. null = trocou de tela; texto = recusa para a tela mostrar.
  async function startTriage(consentVersion: string, citizenId: string, protocolName: string): Promise<string | null> {
    try {
      const r = await citizenApi.startTriage({ citizenId, protocolName, consentVersion });
      setState({ at: "question", consentVersion, conversationId: r.conversation_id, citizenId: r.citizen_id, step: r.step });
      return null;
    } catch (e) {
      return messageFor(e);
    }
  }
```

- no `switch`, logo depois do `case "people": ... break;`, acrescente:

```tsx
    case "catalog":
      view = <CatalogStep key={state.citizenId} citizenId={state.citizenId} notice={state.notice}
        onStart={name => startTriage(state.consentVersion, state.citizenId, name)}
        onProfileRequired={() => setState({ at: "profile", consentVersion: state.consentVersion, citizenId: state.citizenId, required: true })}
        onProfile={() => setState({ at: "profile", consentVersion: state.consentVersion, citizenId: state.citizenId, required: false })}
        onHistory={() => setState({ at: "history", consentVersion: state.consentVersion, citizenId: state.citizenId })}
        onBack={() => setState({ at: "people", consentVersion: state.consentVersion })} />;
      break;
    case "profile":
      view = <ProfileStep key={`${state.citizenId}-${state.required}`} citizenId={state.citizenId} required={state.required}
        onSaved={() => setState({ at: "catalog", consentVersion: state.consentVersion, citizenId: state.citizenId,
          notice: state.required ? null : "Perfil atualizado." })}
        onBack={() => setState(state.required
          ? { at: "people", consentVersion: state.consentVersion }
          : { at: "catalog", consentVersion: state.consentVersion, citizenId: state.citizenId, notice: null })} />;
      break;
```

- [ ] **Step 6: Remova `citizenApi.start`**

Em `src/lib/citizenApi.ts`, apague o método:

```ts
  start: (p: { citizenId?: string; cpf?: string; neighborhoodId?: string; consentVersion: string }) =>
    call<StartResult>("POST", "/conversations", {
      ...(p.citizenId ? { citizen_id: p.citizenId } : { cpf: p.cpf }),
      ...(p.neighborhoodId ? { neighborhood_id: p.neighborhoodId } : {}),
      consent_version: p.consentVersion
    }),
```

Confira que nada mais o usa:

Run: `grep -rn "citizenApi.start(\|\"start\")" src`
Expected: nenhuma linha.

- [ ] **Step 7: Rode e veja passar; suíte e tipos**

Run: `npx vitest run src/modules/citizen/Flow.catalog.test.tsx src/modules/citizen/Flow.test.tsx src/lib/citizenApi.test.ts && npm test && npx tsc --noEmit`
Expected: PASS. Contagem da suíte: −3 (testes de `start` apagados) + os novos.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/Flow.tsx src/modules/citizen/Flow.catalog.test.tsx \
  src/modules/citizen/Flow.test.tsx src/lib/citizenApi.ts src/lib/citizenApi.test.ts
/opt/homebrew/bin/git commit -m "feat: route citizens through profile and catalog before starting a chosen triage" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Recusas ao começar e guarda de privacidade do perfil

**Files:**
- Modify: `src/modules/citizen/Flow.tsx` (`startTriage`)
- Test: `src/modules/citizen/Flow.catalog.test.tsx`

**Interfaces:**
- Consumes: `startTriage` do `Flow` (Task 7), mensagens (Task 1).
- Produces: `startTriage` passa a mandar `profile_required` para o perfil obrigatório e `consent_outdated`/`no_consent` para o termo (devolve `null` nesses casos); `not_offered`, `triage_in_progress` e o resto continuam voltando como texto para a tela.

- [ ] **Step 1: Escreva os testes**

Acrescente ao fim de `src/modules/citizen/Flow.catalog.test.tsx` (usa os helpers do topo do arquivo; acrescente `waitFor` ao import de `@testing-library/react`):

```tsx
describe("Flow — recusas ao começar", () => {
  it.each([
    [ "not_offered", "Esta triagem não está mais disponível para esta pessoa." ],
    [ "triage_in_progress", "Já existe uma triagem em andamento para esta pessoa. Continue a que está aberta." ]
  ])("409 %s: fica no catálogo relido, com o aviso", async (code, message) => {
    signedIn([ avo ]);
    const catalog = vi.spyOn(citizenApi, "catalog").mockResolvedValue(CATALOG);
    vi.spyOn(citizenApi, "startTriage").mockRejectedValue(new ApiError(409, code));
    await choosePerson();

    await userEvent.click(within(await findItem("Saúde do idoso")).getByRole("button", { name: "Começar" }));

    expect(await screen.findByText(message)).toBeInTheDocument();
    await waitFor(() => expect(catalog).toHaveBeenCalledTimes(2));
    expect(screen.queryByText("Algo deu errado. Tente de novo.")).not.toBeInTheDocument();
  });

  it("409 profile_required ao começar: vai para o perfil obrigatório", async () => {
    signedIn([ avo ]);
    vi.spyOn(citizenApi, "catalog").mockResolvedValue(CATALOG);
    vi.spyOn(citizenApi, "startTriage").mockRejectedValue(new ApiError(409, "profile_required"));
    await choosePerson();

    await userEvent.click(within(await findItem("Saúde do idoso")).getByRole("button", { name: "Começar" }));

    expect(await screen.findByRole("heading", { name: "Sobre esta pessoa" })).toBeInTheDocument();
  });

  it.each([ "consent_outdated", "no_consent" ])("409 %s ao começar: volta ao termo", async (code) => {
    signedIn([ avo ]);
    vi.spyOn(citizenApi, "catalog").mockResolvedValue(CATALOG);
    vi.spyOn(citizenApi, "startTriage").mockRejectedValue(new ApiError(409, code));
    await choosePerson();

    await userEvent.click(within(await findItem("Saúde do idoso")).getByRole("button", { name: "Começar" }));

    expect(await screen.findByRole("button", { name: "Concordo" })).toBeInTheDocument();
  });
});

describe("Flow — privacidade do perfil (ADR 0027, invariantes)", () => {
  it("nenhum dado de perfil vai para a URL ou para o console, nem no erro do api", async () => {
    const methods = [ "log", "info", "warn", "error", "debug" ] as const;
    const spies = methods.map(m => vi.spyOn(console, m).mockImplementation(() => {}));
    signedIn([]);
    vi.spyOn(citizenApi, "createPerson")
      .mockRejectedValueOnce(new ApiError(422, "invalid_birth_date"))
      .mockResolvedValue({ ...avo, id: "p9" });
    vi.spyOn(citizenApi, "catalog").mockResolvedValue(CATALOG);

    render(<Flow />);
    await userEvent.click(await screen.findByRole("button", { name: "Concordo" }));
    await userEvent.type(await screen.findByLabelText("CPF de outra pessoa"), "52998224725");
    await userEvent.click(screen.getByRole("button", { name: "Continuar com este CPF" }));
    await fillProfile();
    await userEvent.click(screen.getByRole("radio", { name: "Mulher trans" }));
    await userEvent.click(screen.getByRole("button", { name: "Continuar" }));
    expect(await screen.findByText("Data de nascimento inválida. Confira dia, mês e ano.")).toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Continuar" }));
    expect(await screen.findByRole("region", { name: "Disponíveis" })).toBeInTheDocument();

    const leaked = /1963|02\/04|female|trans_woman|Feminino|Mulher trans/;
    expect(window.location.href).not.toMatch(leaked);
    for (const spy of spies) for (const call of spy.mock.calls) expect(JSON.stringify(call)).not.toMatch(leaked);
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/Flow.catalog.test.tsx`
Expected: FAIL nos casos `profile_required`, `consent_outdated` e `no_consent` (a tela mostra a mensagem em vez de trocar). Os casos `not_offered` e `triage_in_progress` e a guarda de privacidade já passam (mensagens da Task 1, corpo montado campo a campo); ficam como guarda.

- [ ] **Step 3: Implemente**

Em `src/modules/citizen/Flow.tsx`, troque o `catch` de `startTriage` por:

```tsx
    } catch (e) {
      if (e instanceof ApiError) {
        if (e.code === "consent_outdated" || e.code === "no_consent") {
          setState({ at: "consent" });
          return null;
        }
        if (e.code === "profile_required") {
          setState({ at: "profile", consentVersion, citizenId, required: true });
          return null;
        }
      }
      // not_offered, triage_in_progress e o resto: a tela mostra e relê o catálogo.
      return messageFor(e);
    }
```

- [ ] **Step 4: Rode e veja passar; suíte e tipos**

Run: `npx vitest run src/modules/citizen/Flow.catalog.test.tsx && npm test && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/Flow.tsx src/modules/citizen/Flow.catalog.test.tsx
/opt/homebrew/bin/git commit -m "feat: send citizens to the profile or the consent term when a start is refused" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: "Recomendamos também" no resultado

**Files:**
- Create: `src/modules/citizen/AlsoRecommended.tsx`
- Modify: `src/modules/citizen/ResultStep.tsx`, `src/modules/citizen/Flow.tsx`
- Test: `src/modules/citizen/ResultStep.test.tsx`, `src/modules/citizen/Flow.catalog.test.tsx`

**Interfaces:**
- Consumes: `TriageSummary.suggestions`, `TriageSuggestion` (Task 1); `startTriage` do `Flow` (Tasks 7 e 8).
- Produces:
  - `LATER_NOTE: string` e `AlsoRecommended({ suggestions, later, onLater, onStart })`, com `later: ReadonlySet<string>` (ids de sugestão escondidos), `onLater: (suggestionId: string) => void`, `onStart: (protocolName: string) => Promise<string | null>`. Lista vazia → `null`.
  - `ResultStep({ triageId, onAgain, onHistory, onStartSuggestion })`, `onStartSuggestion: (protocolName: string) => Promise<string | null>` (obrigatório).
  - No `Flow`, "Fazer outra triagem" (`onAgain`) abre o catálogo da mesma pessoa.

- [ ] **Step 1: Ajuste as renderizações existentes da `ResultStep`**

O prop novo é obrigatório. Acrescente-o às renderizações que já existem em `src/modules/citizen/ResultStep.test.tsx`:

```bash
perl -pi -e 's/(onHistory=\{(?:vi\.fn\(\)|onHistory)\}) \/>/$1 onStartSuggestion={vi.fn()} \/>/g' src/modules/citizen/ResultStep.test.tsx
grep -c "onStartSuggestion" src/modules/citizen/ResultStep.test.tsx
```

Expected: `6` (todas as `render(<ResultStep ... />)` do arquivo).

- [ ] **Step 2: Escreva os testes da `ResultStep`**

Acrescente ao fim de `src/modules/citizen/ResultStep.test.tsx`:

```tsx
describe("ResultStep — Recomendamos também (módulo 15)", () => {
  const suggestions = [
    { suggestion_id: "s1", protocol_name: "saude-mental-aprofundada", title: "Saúde mental — aprofundamento",
      summary: "Mais perguntas sobre humor e sono." },
    { suggestion_id: "s2", protocol_name: "saude-do-idoso", title: "Saúde do idoso", summary: null }
  ];

  it("mostra as sugestões enquanto o relatório é preparado; 'Fazer agora' chama com o nome do protocolo", async () => {
    vi.spyOn(citizenApi, "triage").mockResolvedValue({ ...summary(null), suggestions });
    const onStartSuggestion = vi.fn<(name: string) => Promise<string | null>>().mockResolvedValue(null);
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} onStartSuggestion={onStartSuggestion} />);

    const block = (await screen.findByRole("heading", { name: "Recomendamos também" })).closest("section")!;
    expect(within(block).getByText("Mais perguntas sobre humor e sono.")).toBeInTheDocument();
    expect(within(block).queryByText(/null|undefined/)).not.toBeInTheDocument();
    const item = within(block).getByText("Saúde mental — aprofundamento").closest("li")!;
    await userEvent.click(within(item).getByRole("button", { name: "Fazer agora" }));
    expect(onStartSuggestion).toHaveBeenCalledWith("saude-mental-aprofundada");
  });

  it("lista vazia (resultado urgente ou nada sugerido): bloco ausente", async () => {
    const triage = vi.spyOn(citizenApi, "triage").mockResolvedValue({ ...summary(null), suggestions: [] });
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} onStartSuggestion={vi.fn()} />);
    await waitFor(() => expect(triage).toHaveBeenCalled());
    await act(async () => { await Promise.resolve(); });
    expect(screen.queryByText("Recomendamos também")).not.toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Fazer agora" })).not.toBeInTheDocument();
  });

  it("api anterior sem o campo: bloco ausente", async () => {
    const triage = vi.spyOn(citizenApi, "triage").mockResolvedValue(summary(null));
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} onStartSuggestion={vi.fn()} />);
    await waitFor(() => expect(triage).toHaveBeenCalled());
    await act(async () => { await Promise.resolve(); });
    expect(screen.queryByText("Recomendamos também")).not.toBeInTheDocument();
  });

  it("'Depois' esconde a sugestão; sem nenhuma, fica só a nota de onde encontrá-las", async () => {
    vi.spyOn(citizenApi, "triage").mockResolvedValue({ ...summary(null), suggestions: [ suggestions[0] ] });
    const onStartSuggestion = vi.fn();
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} onStartSuggestion={onStartSuggestion} />);
    await userEvent.click(await screen.findByRole("button", { name: "Depois" }));
    expect(screen.queryByRole("heading", { name: "Recomendamos também" })).not.toBeInTheDocument();
    expect(screen.getByText(LATER_NOTE)).toBeInTheDocument();
    expect(onStartSuggestion).not.toHaveBeenCalled();
  });

  it("'Depois' continua valendo quando o relatório fica pronto", async () => {
    vi.spyOn(citizenApi, "triage")
      .mockResolvedValueOnce({ ...summary(null), suggestions: [ suggestions[0] ] })
      .mockResolvedValue({ ...summary("http://curitiba.localhost/wpda/?token=abc"), suggestions: [ suggestions[0] ] });
    stubReport();
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} onStartSuggestion={vi.fn()} />);
    await userEvent.click(await screen.findByRole("button", { name: "Depois" }));
    expect(await screen.findByText("Procure atendimento hoje", undefined, { timeout: 3000 })).toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Fazer agora" })).not.toBeInTheDocument();
    expect(screen.getByText(LATER_NOTE)).toBeInTheDocument();
  });

  it("recusa ao começar aparece no bloco", async () => {
    vi.spyOn(citizenApi, "triage").mockResolvedValue({ ...summary(null), suggestions: [ suggestions[0] ] });
    const onStartSuggestion = vi.fn().mockResolvedValue("Esta triagem não está mais disponível para esta pessoa.");
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} onStartSuggestion={onStartSuggestion} />);
    await userEvent.click(await screen.findByRole("button", { name: "Fazer agora" }));
    expect(await screen.findByText("Esta triagem não está mais disponível para esta pessoa.")).toBeInTheDocument();
  });

  it("toque duplo em 'Fazer agora' chama uma vez só", async () => {
    vi.spyOn(citizenApi, "triage").mockResolvedValue({ ...summary(null), suggestions: [ suggestions[0] ] });
    const onStartSuggestion = vi.fn().mockReturnValue(new Promise(() => {}));
    render(<ResultStep triageId="t1" onAgain={vi.fn()} onHistory={vi.fn()} onStartSuggestion={onStartSuggestion} />);
    const btn = await screen.findByRole("button", { name: "Fazer agora" });
    await userEvent.click(btn);
    await userEvent.click(btn);
    expect(onStartSuggestion).toHaveBeenCalledTimes(1);
    expect(btn).toBeDisabled();
  });
});
```

e acrescente ao topo do arquivo, depois de `import { ResultStep } from "./ResultStep";`:

```tsx
import { LATER_NOTE } from "./AlsoRecommended";
```

- [ ] **Step 3: Escreva o teste do fluxo**

Acrescente ao fim de `src/modules/citizen/Flow.catalog.test.tsx`:

```tsx
describe("Flow — resultado com sugestão", () => {
  async function reachResult() {
    signedIn([ avo ]);
    vi.spyOn(citizenApi, "catalog").mockResolvedValue(CATALOG);
    const start = vi.spyOn(citizenApi, "startTriage")
      .mockResolvedValueOnce({ conversation_id: "c1", citizen_id: "p1", resumed: false, step })
      .mockResolvedValue({ conversation_id: "c2", citizen_id: "p1", resumed: false,
        step: { ...step, triage_id: "t2", step_id: "humor", prompt: "Como está o seu humor?" } });
    vi.spyOn(citizenApi, "answer").mockResolvedValue({ status: "completed", triage_id: "t1" });
    vi.spyOn(citizenApi, "triage").mockResolvedValue({
      id: "t1", status: "completed", tier: "media", priority: 2, created_at: "2026-10-05T12:00:00Z",
      completed_at: "2026-10-05T12:05:00Z", report_url: null, consent_active: true, origin_phone_masked: null,
      attendance: null, check_in_available: false, reference_units: [],
      suggestions: [ { suggestion_id: "s1", protocol_name: "saude-mental-aprofundada",
        title: "Saúde mental — aprofundamento", summary: null } ]
    });
    await choosePerson();
    await userEvent.click(within(await findItem("Saúde do idoso")).getByRole("button", { name: "Começar" }));
    await userEvent.click(await screen.findByRole("button", { name: "Não" }));
    await screen.findByRole("heading", { name: "Recomendamos também" });
    return { start };
  }

  it("'Fazer agora' começa a triagem sugerida para a mesma pessoa", async () => {
    const { start } = await reachResult();
    await userEvent.click(screen.getByRole("button", { name: "Fazer agora" }));
    expect(await screen.findByText("Como está o seu humor?")).toBeInTheDocument();
    expect(start).toHaveBeenLastCalledWith({ citizenId: "p1", protocolName: "saude-mental-aprofundada", consentVersion: "1" });
  });

  it("'Fazer outra triagem' abre o catálogo da mesma pessoa", async () => {
    await reachResult();
    await userEvent.click(screen.getByRole("button", { name: "Fazer outra triagem" }));
    expect(await screen.findByRole("heading", { name: "Qual triagem fazer?" })).toBeInTheDocument();
  });
});
```

- [ ] **Step 4: Rode e veja falhar**

Run: `npx vitest run src/modules/citizen/ResultStep.test.tsx src/modules/citizen/Flow.catalog.test.tsx`
Expected: FAIL — `Failed to resolve import "./AlsoRecommended"`.

- [ ] **Step 5: Implemente `AlsoRecommended`**

```tsx
// src/modules/citizen/AlsoRecommended.tsx
// "Recomendamos também" (spec 2026-10-05 §8.5; ADR 0027): sugestões nascidas
// desta triagem. Nunca começa sozinha. "Depois" só esconde aqui: a sugestão
// continua em "Sugeridas para você" no catálogo. Lista vazia (resultado
// urgente, nada sugerido, api anterior) = bloco ausente.
import { useRef, useState } from "react";
import type { TriageSuggestion } from "../../lib/citizenApi";
import { BigButton, ErrorText } from "./ui";

export const LATER_NOTE = "As sugestões ficam em \"Sugeridas para você\", na escolha da triagem.";

export function AlsoRecommended({ suggestions, later, onLater, onStart }: {
  suggestions: TriageSuggestion[]; later: ReadonlySet<string>; onLater: (suggestionId: string) => void;
  onStart: (protocolName: string) => Promise<string | null>;
}) {
  const [error, setError] = useState<string | null>(null);
  const [busy, setBusy] = useState(false);
  // Guarda síncrona: setBusy só vale no próximo render.
  const inFlight = useRef(false);

  if (suggestions.length === 0) return null;
  const visible = suggestions.filter(s => !later.has(s.suggestion_id));
  if (visible.length === 0) return <p role="status" style={{ fontSize: 18, margin: "0 0 24px" }}>{LATER_NOTE}</p>;

  async function start(protocolName: string) {
    if (inFlight.current) return;
    inFlight.current = true;
    setBusy(true);
    setError(null);
    try {
      const refused = await onStart(protocolName);
      if (refused) setError(refused);
    } finally {
      inFlight.current = false;
      setBusy(false);
    }
  }

  return (
    <section aria-labelledby="also-recommended-title"
      style={{ margin: "0 0 24px", padding: 12, borderRadius: 12, border: "1px solid var(--line, #eee)" }}>
      <h2 id="also-recommended-title" style={{ fontSize: 20, margin: "0 0 8px" }}>Recomendamos também</h2>
      <ul style={{ listStyle: "none", margin: 0, padding: 0, display: "grid", gap: 16 }}>
        {visible.map(s => (
          <li key={s.suggestion_id} style={{ display: "grid", gap: 8 }}>
            <strong style={{ fontSize: 18 }}>{s.title}</strong>
            {s.summary && <p style={{ margin: 0, fontSize: 18, color: "var(--ink2, #555)" }}>{s.summary}</p>}
            <BigButton disabled={busy} onClick={() => void start(s.protocol_name)}>Fazer agora</BigButton>
            <BigButton variant="secondary" disabled={busy} onClick={() => onLater(s.suggestion_id)}>Depois</BigButton>
          </li>
        ))}
      </ul>
      {error && <ErrorText>{error}</ErrorText>}
    </section>
  );
}
```

- [ ] **Step 6: Ligue o bloco na `ResultStep`**

Em `src/modules/citizen/ResultStep.tsx`:

- troque `import { citizenApi, type ReferenceUnit } from "../../lib/citizenApi";` por:

```tsx
import { citizenApi, type ReferenceUnit, type TriageSuggestion } from "../../lib/citizenApi";
import { AlsoRecommended } from "./AlsoRecommended";
```

- troque a assinatura e os estados do componente por:

```tsx
export function ResultStep({ triageId, onAgain, onHistory, onStartSuggestion }: {
  triageId: string; onAgain: () => void; onHistory: () => void;
  onStartSuggestion: (protocolName: string) => Promise<string | null>;
}) {
  const [token, setToken] = useState<string | null>(null);
  const [gaveUp, setGaveUp] = useState(false);
  const [units, setUnits] = useState<ReferenceUnit[]>([]);
  const [suggestions, setSuggestions] = useState<TriageSuggestion[]>([]);
  // "Depois" mora aqui: o bloco muda de lugar quando o relatório fica pronto.
  const [later, setLater] = useState<ReadonlySet<string>>(new Set());
```

- em `poll`, troque `if (alive) setUnits(t.reference_units ?? []);` por:

```tsx
        if (alive) {
          setUnits(t.reference_units ?? []);
          setSuggestions(t.suggestions ?? []);
        }
```

- logo antes de `const actions = (`, acrescente:

```tsx
  const recommended = <AlsoRecommended suggestions={suggestions} later={later}
    onLater={id => setLater(prev => new Set(prev).add(id))} onStart={onStartSuggestion} />;
```

- troque `return <Report token={token}><ReferenceUnits units={units} />{actions}</Report>;` por:

```tsx
    return <Report token={token}><ReferenceUnits units={units} />{recommended}{actions}</Report>;
```

- no `return` do "Triagem concluída", troque `<ReferenceUnits units={units} />` por:

```tsx
      <ReferenceUnits units={units} />
      {recommended}
```

- [ ] **Step 7: Ligue no `Flow`**

Em `src/modules/citizen/Flow.tsx`, troque o `case "result":` inteiro por:

```tsx
    case "result":
      view = <ResultStep triageId={state.triageId}
        onAgain={() => setState({ at: "catalog", consentVersion: state.consentVersion, citizenId: state.citizenId, notice: null })}
        onHistory={() => setState({ at: "history", consentVersion: state.consentVersion, citizenId: state.citizenId })}
        onStartSuggestion={name => startTriage(state.consentVersion, state.citizenId, name)} />;
      break;
```

- [ ] **Step 8: Rode e veja passar; suíte e tipos**

Run: `npx vitest run src/modules/citizen/ResultStep.test.tsx src/modules/citizen/Flow.catalog.test.tsx && npm test && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 9: Commit**

```bash
/opt/homebrew/bin/git add src/modules/citizen/AlsoRecommended.tsx src/modules/citizen/ResultStep.tsx \
  src/modules/citizen/ResultStep.test.tsx src/modules/citizen/Flow.tsx src/modules/citizen/Flow.catalog.test.tsx
/opt/homebrew/bin/git commit -m "feat: show suggested follow-up triages on the citizen result" \
  -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Revisão e prova no navegador

**Files:** nenhum, a não ser que a prova ache um bug (nesse caso, teste + correção + commit `fix:` no wpda; se o bug for do api, avise a sessão dona do api em vez de mexer lá).

- [ ] **Step 1: Revisão do branch.** Um subagente revisor lê `/opt/homebrew/bin/git diff origin/main..HEAD` do wpda contra a spec §8, o contrato §2–§3 e este plano, com atenção a:
  - nenhum campo além dos do contrato nos corpos (`POST /citizen/people`, `/profile`, `/conversations`); os corpos montados campo a campo;
  - nenhum dado de perfil em URL, query string ou `console.*`: `/opt/homebrew/bin/git diff origin/main..HEAD -- src | grep -n "console\.\|location\|history\.\(push\|replace\)State\|<form"` só pode mostrar linhas de teste;
  - nenhuma data de calendário passando por `new Date(` (`grep -n "new Date(" src/modules/citizen/CatalogStep.tsx src/modules/citizen/ProfileStep.tsx src/lib/profile.ts` só pode mostrar o `Date.UTC` de `parseBirthDate` e o `todayInCity`);
  - o wpda não filtra nem reordena o catálogo; "Recomendamos também" nunca aparece com lista vazia;
  - todo `button`/`label` novo com `fontSize` ≥ 18 e alvo ≥ 48 px;
  - `vi.setSystemTime` em `beforeEach` em todo teste novo que depende de data;
  - commits só com arquivos pelo nome (`/opt/homebrew/bin/git show --stat origin/main..HEAD` sem `node_modules` nem `.superpowers/`).
- [ ] **Step 2: Confira o contrato com o api.** Com o api do módulo 15 rodando em `:3032`, as respostas de `GET /citizen/people`, `GET /citizen/people/:id/catalog`, `POST /citizen/people`, `POST /citizen/people/:id/profile` e `GET /citizen/triages/:id` batem com o contrato §3, inclusive `reference_units` e `suggested[].source_title` no catálogo. Se divergir, pare e avise; não adapte o cliente a outro formato sem decisão.
- [ ] **Step 3: Prova no navegador (Curitiba, spec §9.5).** Com o api do branch do módulo 15 em `:3032` (migração de cidade aplicada e a semente da spec §10: "Saúde do idoso" com `gte profile.age 60`, "Saúde mental" sugerindo o aprofundamento por pontuação, família no mesmo celular com avó e neto) e o Vite deste worktree:

  ```bash
  VITE_API_PROXY_TARGET=http://localhost:3032 npx vite --port 5188
  ```

  Abra `http://curitiba.localhost:5188/wpda/`. O login do cidadão (celular + código do log do api) é feito pelo usuário; não digite o código. Depois, com screenshot de cada passo:
  1. termo → "Para quem é esta triagem?" → a avó (62): se não tiver perfil, "Sobre esta pessoa" aparece antes do catálogo; o catálogo mostra "Saúde do idoso" em "Disponíveis";
  2. "Voltar" → o neto (8): o catálogo **não** mostra "Saúde do idoso"; "Meu perfil" mostra os dados dele, editáveis;
  3. CPF novo: CPF → perfil (data futura recusada na hora) → bairro → catálogo; a URL continua `/wpda/` em todas as telas;
  4. "Saúde mental" com respostas de pontuação alta → resultado com "Recomendamos também" → "Depois" → "Fazer outra triagem" → o aprofundamento está em "Sugeridas para você" com "Sugerida pelo resultado da triagem Saúde mental de <hoje>";
  5. "Começar" na sugerida → a triagem do aprofundamento abre; ao fim, a anterior aparece em "Feitas recentemente" com "Próxima a partir de";
  6. a cidade pausa o aprofundamento (dashboard do módulo 15, `municipal_admin` com step-up; sem o dashboard mergeado, peça à sessão dona do api para pausar) com uma nova sugestão pendente → recarregar o catálogo: a sugestão some (expirou);
  7. um par com perfil `verified` (validação presencial na semente ou pelo dashboard): "Meu perfil" mostra "Conferido no posto", sem campos.
- [ ] **Step 4: Suíte final.** `npm test` e `npx tsc --noEmit` no worktree; registre a contagem no relatório (antes → depois).
- [ ] **Step 5: Pare.** O merge do wpda vem depois do merge do api (spec §11), e só com autorização explícita do usuário. Não faça push.

---

## Self-review

- **Cobertura da spec:** §8.1 (celular + código → "para quem é?", como hoje) → sem mudança, coberto pelos testes existentes; §8.2 (perfil obrigatório: data, sexo, identidade opcional com "Prefiro não informar", finalidade e link do termo) → Tasks 3, 4, 6 e 7; §8.3 (catálogo: sugeridas com a origem via `source_title`, disponíveis, feitas recentemente com "próxima a partir de") → Task 5, mais "Em andamento" do contrato §0.2; §8.4 (triagem como hoje) → Task 7 (início por `protocol_name`); §8.5 (resultado + "Recomendamos também", Fazer agora / Depois, nunca com urgente) → Task 9; §8.6 ("Meu perfil": declared corrige, verified "conferido no posto") → Task 6, com o 409 `profile_verified`; §8.7 (catálogo vazio com a unidade de referência) → Task 5, com `reference_units` do catálogo (contrato §3.4); §5.5 (nada de perfil em URL/log) → Tasks 1, 3 e 8; contrato §3.5 (`not_offered`, `triage_in_progress`, `profile_required`) → Tasks 7 e 8; §9.4 (perfil obrigatório, três seções, "Recomendamos também" ausente em urgente, correção e bloqueio quando verified, relógio fixo) → Tasks 3–9; §9.5 (prova no navegador) → Task 10.
- **Placeholders:** nenhum passo sem código; os ajustes em arquivos existentes dizem a linha âncora. A porta do api na prova é 3032 (api do módulo 15).
- **Tipos:** `ProfileInput`/`Profile`/`Catalog`/`TriageSuggestion` (Task 1) são os usados nas Tasks 2–9; `PersonChoice` com `{ cpf, profile }` (Task 4) é o que o `choose` da Task 7 lê; `CatalogStep.onStart` e `ResultStep.onStartSuggestion` têm o mesmo tipo `(protocolName: string) => Promise<string | null>` que `startTriage` do `Flow` devolve (Tasks 5, 7, 9); os estados `catalog` (`notice: string | null`) e `profile` (`required: boolean`) nascem na Task 7 e são os que as Tasks 8 e 9 usam; `citizenApi.start` some na Task 7, depois que o `Flow` passa a usar `startTriage`.
- **Review Focus:** os cinco itens têm teste na task dona (Tasks 1, 2, 3, 5, 7 e 9).
