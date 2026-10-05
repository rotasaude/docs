# Módulo 16 — Interruptores por cidade (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Aba **Funcionalidades** no detalhe da cidade do console de manutenção (F-16.1): a lista `features` da cidade (chave, descrição, ligada, utilizável, o que falta, quem mudou e quando), o liga/desliga por `setCityFeature` com confirmação, e o modo de prontuário (`recordMode`) e o código IBGE do perfil da cidade (`profile.ibgeCode`) só para leitura.

**Architecture:** Uma aba nova, montada por um componente próprio (`FeaturesTab`, no molde de `ProtocolsTab` e `AnalyticsTab`), que só consulta quando a aba é aberta. Uma consulta GraphQL própria (`CityFeatures`: `recordMode`, `profile { ibgeCode }`, `features`), **fora** do `CityHeader`, e uma mutation (`SetCityFeature`). As regras de exibição (rótulos do modo, do que falta, dos estados, dos erros e a frase depois do ato) ficam em `src/lib/features.ts`. A confirmação é o `ConfirmButton` que já existe (dois cliques). Depois do ato a tela diz o estado que o api devolveu e relê a lista.

**Por que consulta própria (ordem de deploy):** a validação do GraphQL recusa o **documento inteiro** quando um campo não existe. Um build novo do maintenance contra um api sem o módulo 16 derrubaria qualquer consulta que pedisse `features`/`recordMode`, inclusive o topo da ficha. Com consulta própria, só a aba Funcionalidades mostra o aviso. Cidade inalcançável não derruba a aba: `usable`/`missing` degradam (`false`/`["city_unreachable"]`) e o liga/desliga segue; só `profile` (anulável, banco da cidade) vem nulo com o erro.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL Codegen (client preset, `src/gql/` gerado e commitado), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/.claude/ciclo2/superpowers/specs/2026-10-05-module-16-record-mode-and-export-design.md` (§3.1, §8 "Front", §9 e §10 F-16.1 são deste plano) e `docs/.claude/ciclo2/adr/0028.md`. O schema GraphQL está fixado em `docs/.claude/ciclo2/superpowers/plans/2026-10-05-module-16-record-mode-contracts.md` §3 (catálogo de pré-requisitos no §2), o mesmo arquivo que o plano `api-foundation` usa. O api da fundação precisa estar **na branch dele e rodando** antes da Task 2 (o `schema.graphql` é extraído do api) e **mergeado antes** do merge deste.

## Global Constraints

- Schema (contratos §3, literal):
  ```graphql
  type CityFeature { key: String!, description: String!, enabled: Boolean!,
                     usable: Boolean!, missing: [String!]!,
                     changedAt: ISO8601DateTime, changedBy: String }
  # em City:
  features: [CityFeature!]!
  recordMode: String!          # off | integrated | record (só leitura aqui)
  # o IBGE continua em City.profile.ibgeCode (existente)
  # mutation (padrão das mutations existentes, BaseMutation#audited):
  setCityFeature(citySlug: String!, key: String!, enabled: Boolean!):
    { ok: Boolean!, errors: [UserError!]!, feature: CityFeature }
  ```
  Erros de regra: `unknown_city`, `unknown_feature` (em `UserError { path, message }`; a tela traduz a mensagem quando ela vem como esse código e mostra a mensagem do api nos demais casos).
- Cidade inalcançável (contratos §3): `enabled`, `changedAt`, `changedBy` vêm da plataforma; `usable`/`missing` degradam para `false`/`["city_unreachable"]`; o liga/desliga continua funcionando.
- Pré-requisitos (`missing`, contratos §2 e §3): `city_unreachable`, `record_mode_off`, `pec_url_missing`, `ibge_code_missing`, `credential_missing:ledi`, `credential_unauthorized:ledi`, `credential_missing:cadsus`, `credential_unauthorized:cadsus`. Valor desconhecido (de `missing`, `recordMode` ou `errors`) aparece **cru**, nunca some e nunca vira `undefined`.
- "Ligada" não é "utilizável" (spec §3.1): três estados na tela — desligada; ligada e utilizável; ligada, falta pré-requisito. Ligar **não** exige pré-requisito: a tela liga e mostra o que falta.
- Só o maintenance escreve interruptor (spec §2, decisão 8). Modo de prontuário, código IBGE e endereço do PEC são do operador no `admin` (spec §3.2): aqui só leitura, sem campo de edição. O código IBGE tem **uma** fonte: `City.profile.ibgeCode` (banco da cidade, o mesmo da aba Perfil; contratos §3). Quando o perfil não pôde ser lido, a tela diz "indisponível (<código>)", nunca "não preenchido".
- Liga/desliga com confirmação: o `ConfirmButton` existente (o primeiro clique só troca o rótulo; o segundo chama a mutation). Sem TOTP: o contrato não tem `code` em `setCityFeature`.
- Nenhum campo novo entra no `CityHeader`. A aba só consulta quando é aberta (`staleTime: Infinity`, botão "atualizar"), como as outras.
- **Ordem de deploy: api → maintenance** (spec §9). O README ganha o aviso, como o das abas Protocolos e Analytics.
- Interface tem ciclo próprio: componentes e estilo existentes (`Panel`, `DataTable`, `Tag`, `Button`, `ConfirmButton`, `ErrorState`, `EmptyState`), sem redesign.
- Nunca escreva à mão em arquivo gerado (`schema.graphql`, `src/gql/*`): eles saem do api em execução e do codegen.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`.
- O módulo 15 corre em paralelo em outras branches; este plano parte de `origin/main` do maintenance e não depende dele.

## Ambiente de execução

- Crie o worktree a partir de `origin/main` e ligue o `node_modules` do checkout principal. O `.gitignore` do maintenance ignora `node_modules` (sem barra, então também o symlink), mas **não** ignora `.claude/`: o checkout principal lista `.claude/` como não rastreado. É esperado; não adicione.

  ```bash
  cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
  /opt/homebrew/bin/git fetch origin
  /opt/homebrew/bin/git log --oneline -1 origin/main
  /opt/homebrew/bin/git worktree add .claude/mod16 -b feat/mod-16-record-mode origin/main
  ln -s ../../node_modules .claude/mod16/node_modules
  ```

  Expected: `origin/main` em `baf3219 fix: explain the missing analytics fields when the old api rejects the date variable` (ou mais novo; se houver commit novo em `src/screens/CityDetail.tsx`, confira os trechos que a Task 3 troca antes de aplicar).

- Todos os caminhos de arquivo abaixo são relativos a `apps/maintenance/.claude/mod16` (e os comandos `cd` usam o caminho a partir da raiz do monorepo, `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`).
- Testes (script `test` = `vitest run`; o `vitest.config.ts` exclui `e2e/**`):

  ```bash
  cd apps/maintenance/.claude/mod16 && npx vitest run <arquivos>
  ```

- Tipos (`typecheck` = `tsc --noEmit`) e codegen:

  ```bash
  cd apps/maintenance/.claude/mod16 && npx tsc --noEmit
  cd apps/maintenance/.claude/mod16 && npm run codegen
  ```

- Base do ambiente de teste: `environment: "jsdom"`, `globals: false` (todo teste de componente chama `afterEach(cleanup)`), sem jest-dom (use `not.toBeNull()`, `toBeNull()`, `.textContent`). O `vitest.config.ts` não fixa `TZ`: a data de "Última mudança" sai de `fmtWhen` (`src/lib/analytics.ts`), que já formata em `America/Sao_Paulo`.
- **`npm run schema:pull` não serve aqui.** O script (`scripts/pull-schema.sh`) sobe três níveis a partir de `scripts/` — no worktree isso cai em `apps/maintenance/.claude`, não na raiz — e escreve no `schema.graphql` do checkout principal. A Task 2 extrai o SDL pelo `docker compose exec` direto, com destino no worktree.
- Login de dev (Task 4): `http://maintenance.localhost:5177`, nunca `localhost:5177` (o 403 de Origin vira a frase genérica). A conta é o `Maintainer` `dev@local` semeado pelo `db:seed`; o usuário faz o login.

## Review Focus

1. **Maintenance novo contra api antigo.** A validação recusa o documento que pede `features`/`recordMode`, com `errors` e sem `data`. A aba Funcionalidades explica ("o api do módulo 16 precisa subir antes do maintenance"); o topo e as outras abas seguem, e o `CityHeader` nunca pede esses campos. Testes: Task 2, "api antiga sem os campos: a aba explica a ordem de deploy"; Task 3, "api antiga: só a aba Funcionalidades falha; o topo e as outras abas seguem".
2. **Ligar sem os pré-requisitos.** O mantenedor liga `ledi_export` numa cidade sem PEC nem credencial: o api aceita (contratos §1: ligada ≠ utilizável), e a tela precisa dizer que ficou ligada **mas não utilizável** e o que falta, em vez de sugerir que a exportação já funciona. Teste: Task 2, "um clique só não muda nada; o segundo liga, e a tela diz o que o api devolveu".
3. **Clique acidental.** Um clique em "ligar"/"desligar" não pode mudar nada: só o segundo, no botão já em "confirmar: …", chama a mutation. Teste: Task 2, mesmo caso do item 2 (conta zero chamadas depois do primeiro clique).
4. **Cidade inalcançável.** O mantenedor precisa conseguir desligar um interruptor de cidade cujo banco caiu. `missing` chega `["city_unreachable"]` e a tela diz isso (não "falta credencial"); o IBGE aparece "indisponível (CITY_UNREACHABLE)", não "não preenchido"; o liga/desliga funciona. Testes: Task 2, "cidade inalcançável: features degrada, o IBGE diz indisponível e o liga/desliga continua"; e, para recusas que ainda derrubam, "erro de campo em features (cidade arquivada) mostra código e mensagem, sem botões" e "recusa do envelope (payload nulo) mostra código e motivo".
5. **Valor que a tela não conhece** (api mais novo: modo `hybrid`, pré-requisito `terminology_missing:sigtap`, mensagem de erro `rate_limited`) e cidade sem IBGE: aparece cru ou "não preenchido", nunca `undefined`/`null` nem linha vazia. Testes: Task 1, "modo desconhecido aparece cru…", "pré-requisito desconhecido (api mais novo) aparece cru", "traduz os erros do contrato e deixa cru o desconhecido"; Task 2, "cidade sem IBGE e em modo off: diz o que falta, sem 'null'".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `src/lib/features.ts`, `src/lib/features.test.ts` | rótulos do modo, do IBGE, dos pré-requisitos (inclusive `city_unreachable`), dos estados, dos erros; frase depois do ato | 1 |
| `schema.graphql`, `src/gql/*` (gerados) | tipos novos do api | 2 |
| `src/screens/FeaturesTab.tsx`, `src/screens/FeaturesTab.test.tsx` | consulta `CityFeatures`, mutation `SetCityFeature`, a aba | 2 |
| `src/screens/CityDetail.tsx`, `src/screens/CityDetail.test.tsx` | aba "Funcionalidades", só consultada quando aberta | 3 |
| `README.md` | aba nova e ordem de deploy | 3 |
| — | suíte, codegen:check, build, revisão e prova no navegador | 4 |

---

### Task 1: Regras de exibição dos interruptores

**Files:**
- Create: `src/lib/features.ts`
- Test: `src/lib/features.test.ts`

**Interfaces:**
- Consumes: `Tone` de `src/theme/tokens.ts`.
- Produces (em `src/lib/features.ts`):
  ```ts
  export function recordModeText(mode: string | null | undefined): string;   // "off" → "desligado (off)"; desconhecido → cru; ausente → "—"
  export function ibgeCodeText(code: string | null | undefined): string;     // ausente → "não preenchido"
  export function missingText(code: string): string;                         // pré-requisito (e city_unreachable) → frase; desconhecido → cru
  export function missingSummary(missing: string[]): string;                 // "a; b" ou "nada"
  export type FeatureView = { enabled: boolean; usable: boolean; missing: string[] };
  export function featureStatus(feature: FeatureView): { label: string; tone: Tone };
  export type UserErrorView = { path?: string | null; message: string };
  export function setFeatureErrorText(errors: UserErrorView[]): string;      // UserError de setCityFeature; código do contrato traduzido
  export function doneText(feature: { key: string } & FeatureView): string;  // frase depois do ato, do estado devolvido
  ```

- [ ] **Step 1: Escreva o teste que falha**

Crie `src/lib/features.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import {
  doneText, featureStatus, ibgeCodeText, missingSummary, missingText, recordModeText, setFeatureErrorText
} from "./features";

describe("recordModeText", () => {
  it("traduz os três modos", () => {
    expect(recordModeText("off")).toBe("desligado (off)");
    expect(recordModeText("integrated")).toBe("integrado ao PEC da cidade (integrated)");
    expect(recordModeText("record")).toBe("prontuário no Rota Saúde (record)");
  });

  it("modo desconhecido aparece cru, e ausente vira travessão", () => {
    expect(recordModeText("hybrid")).toBe("hybrid");
    expect(recordModeText(null)).toBe("—");
    expect(recordModeText(undefined)).toBe("—");
  });
});

describe("ibgeCodeText", () => {
  it("mostra o código ou diz que falta", () => {
    expect(ibgeCodeText("4106902")).toBe("4106902");
    expect(ibgeCodeText(null)).toBe("não preenchido");
    expect(ibgeCodeText(undefined)).toBe("não preenchido");
  });
});

describe("missingText e missingSummary", () => {
  it("traduz cada pré-requisito do catálogo", () => {
    expect(missingText("record_mode_off")).toBe("modo de prontuário desligado");
    expect(missingText("pec_url_missing")).toBe("endereço do PEC não preenchido");
    expect(missingText("ibge_code_missing")).toBe("código IBGE não preenchido");
    expect(missingText("credential_missing:ledi")).toBe("credencial LEDI não cadastrada");
    expect(missingText("credential_unauthorized:ledi")).toBe("credencial LEDI recusada no último teste");
    expect(missingText("credential_missing:cadsus")).toBe("credencial CADSUS não cadastrada");
    expect(missingText("credential_unauthorized:cadsus")).toBe("credencial CADSUS recusada no último teste");
  });

  it("cidade inalcançável vira frase própria, não 'falta credencial'", () => {
    expect(missingText("city_unreachable")).toBe("banco da cidade inalcançável — não deu para conferir");
  });

  it("pré-requisito desconhecido (api mais novo) aparece cru", () => {
    expect(missingText("terminology_missing:sigtap")).toBe("terminology_missing:sigtap");
  });

  it("junta a lista; vazia é 'nada'", () => {
    expect(missingSummary([])).toBe("nada");
    expect(missingSummary([ "pec_url_missing", "credential_missing:ledi" ]))
      .toBe("endereço do PEC não preenchido; credencial LEDI não cadastrada");
  });
});

describe("featureStatus", () => {
  it("desligada, ligada e utilizável, ligada sem pré-requisito", () => {
    expect(featureStatus({ enabled: false, usable: false, missing: [ "pec_url_missing" ] }))
      .toEqual({ label: "desligada", tone: "neutral" });
    expect(featureStatus({ enabled: true, usable: true, missing: [] }))
      .toEqual({ label: "ligada e utilizável", tone: "ok" });
    expect(featureStatus({ enabled: true, usable: false, missing: [ "pec_url_missing" ] }))
      .toEqual({ label: "ligada, falta pré-requisito", tone: "warn" });
  });
});

describe("setFeatureErrorText", () => {
  it("traduz os erros do contrato e deixa cru o desconhecido", () => {
    expect(setFeatureErrorText([ { path: "key", message: "unknown_feature" } ])).toBe("funcionalidade desconhecida pelo api");
    expect(setFeatureErrorText([ { path: "citySlug", message: "unknown_city" } ])).toBe("cidade inexistente");
    expect(setFeatureErrorText([ { path: null, message: "rate_limited" } ])).toBe("rate_limited");
  });

  it("mensagem já em português passa como veio, e várias se juntam", () => {
    expect(setFeatureErrorText([
      { path: "citySlug", message: "cidade inexistente" }, { path: "key", message: "funcionalidade desconhecida" }
    ])).toBe("cidade inexistente; funcionalidade desconhecida");
  });

  it("lista vazia com ok=false vira a frase genérica", () => {
    expect(setFeatureErrorText([])).toBe("não foi possível concluir — tente de novo");
  });
});

describe("doneText", () => {
  it("diz o estado devolvido pelo api, inclusive ligada sem poder usar", () => {
    expect(doneText({ key: "ledi_export", enabled: false, usable: false, missing: [] }))
      .toBe("ledi_export desligada.");
    expect(doneText({ key: "cadsus_lookup", enabled: true, usable: true, missing: [] }))
      .toBe("cadsus_lookup ligada e utilizável.");
    expect(doneText({ key: "ledi_export", enabled: true, usable: false, missing: [ "credential_missing:ledi" ] }))
      .toBe("ledi_export ligada, mas ainda não utilizável — falta: credencial LEDI não cadastrada.");
  });
});
```

- [ ] **Step 2: Rode e confirme que falha**

Run: `cd apps/maintenance/.claude/mod16 && npx vitest run src/lib/features.test.ts`
Expected: FAIL — `Failed to resolve import "./features" from "src/lib/features.test.ts"`.

- [ ] **Step 3: Implemente**

Crie `src/lib/features.ts`:

```ts
import type { Tone } from "../theme/tokens";

// Regras de exibição da aba Funcionalidades (módulo 16, ADR 0028; contratos
// §2 e §3). Quem decide o que está ligado e o que falta é o api: aqui só se
// traduz o que ele devolveu. Valor desconhecido (api mais novo que a tela)
// aparece cru — nunca "undefined", nunca some.

const RECORD_MODE_LABELS: Record<string, string> = {
  off: "desligado (off)",
  integrated: "integrado ao PEC da cidade (integrated)",
  record: "prontuário no Rota Saúde (record)"
};

export function recordModeText(mode: string | null | undefined): string {
  if (!mode) return "—";
  return RECORD_MODE_LABELS[mode] ?? mode;
}

export function ibgeCodeText(code: string | null | undefined): string {
  return code ? code : "não preenchido";
}

// Pré-requisitos do catálogo (contratos §2).
const MISSING_LABELS: Record<string, string> = {
  record_mode_off: "modo de prontuário desligado",
  pec_url_missing: "endereço do PEC não preenchido",
  ibge_code_missing: "código IBGE não preenchido",
  "credential_missing:ledi": "credencial LEDI não cadastrada",
  "credential_unauthorized:ledi": "credencial LEDI recusada no último teste",
  "credential_missing:cadsus": "credencial CADSUS não cadastrada",
  "credential_unauthorized:cadsus": "credencial CADSUS recusada no último teste",
  // O api não conseguiu ler o banco da cidade: usable vem false sem que se
  // saiba o que de fato falta (contratos §3). O liga/desliga continua.
  city_unreachable: "banco da cidade inalcançável — não deu para conferir"
};

export function missingText(code: string): string {
  return MISSING_LABELS[code] ?? code;
}

export type FeatureView = { enabled: boolean; usable: boolean; missing: string[] };

// Três estados, nunca dois: "ligada" não quer dizer "utilizável" (contratos §1).
export function featureStatus(feature: FeatureView): { label: string; tone: Tone } {
  if (!feature.enabled) return { label: "desligada", tone: "neutral" };
  if (feature.usable) return { label: "ligada e utilizável", tone: "ok" };
  return { label: "ligada, falta pré-requisito", tone: "warn" };
}

export function missingSummary(missing: string[]): string {
  return missing.length === 0 ? "nada" : missing.map(missingText).join("; ");
}

// Erros de regra de setCityFeature (contratos §3): UserError { path, message },
// como as outras mutations. Se a mensagem vier como o código do contrato
// (unknown_city, unknown_feature), traduz; senão mostra a mensagem do api.
const SET_ERROR_LABELS: Record<string, string> = {
  unknown_city: "cidade inexistente",
  unknown_feature: "funcionalidade desconhecida pelo api"
};

export type UserErrorView = { path?: string | null; message: string };

export function setFeatureErrorText(errors: UserErrorView[]): string {
  if (errors.length === 0) return "não foi possível concluir — tente de novo";
  return errors.map((e) => SET_ERROR_LABELS[e.message] ?? e.message).join("; ");
}

// Frase depois do ato: diz o estado que o api devolveu, não o pedido.
export function doneText(feature: { key: string } & FeatureView): string {
  if (!feature.enabled) return `${feature.key} desligada.`;
  if (feature.usable) return `${feature.key} ligada e utilizável.`;
  return `${feature.key} ligada, mas ainda não utilizável — falta: ${missingSummary(feature.missing)}.`;
}
```

- [ ] **Step 4: Rode e confirme que passa**

Run: `cd apps/maintenance/.claude/mod16 && npx vitest run src/lib/features.test.ts`
Expected: PASS, 12 testes.

- [ ] **Step 5: Tipos e commit**

Run: `cd apps/maintenance/.claude/mod16 && npx tsc --noEmit`
Expected: sem saída.

```bash
cd apps/maintenance/.claude/mod16
/opt/homebrew/bin/git add src/lib/features.ts src/lib/features.test.ts
/opt/homebrew/bin/git status --short
/opt/homebrew/bin/git commit -m "feat: add display rules for the maintenance feature switches

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Expected: `status --short` antes do commit lista só os dois arquivos.

---

### Task 2: Schema, codegen e `FeaturesTab`

**Files:**
- Create: `src/screens/FeaturesTab.tsx`
- Test: `src/screens/FeaturesTab.test.tsx`
- Gerados: `schema.graphql`, `src/gql/*`

**Interfaces:**
- Consumes: tudo de `src/lib/features.ts` (Task 1); `fmtWhen` de `src/lib/analytics.ts`; `gql` de `src/lib/api.ts`; `graphql` de `src/gql`; `GraphQLRefusal` de `src/lib/errors.ts`; `Panel`, `Button`, `ConfirmButton`, `Tag`, `DataTable`, `EmptyState`, `ErrorState` de `src/components/`. Do api: `City.features`, `City.recordMode`, `City.profile.ibgeCode` (existente) e `Mutation.setCityFeature` (contratos §3).
- Produces: `export function FeaturesTab({ slug }: { slug: string }): JSX.Element`. Operações GraphQL com nomes fixos, que os testes da Task 3 procuram: `query CityFeatures` e `mutation SetCityFeature`. Chave de cache: `[ "city", slug, "features" ]`.
- **Depende do plano `api-foundation`** ter os campos e a mutation no `Maintenance::Schema` na branch em execução.

- [ ] **Step 1: Extraia o schema do api do módulo 16**

O api da fundação roda na worktree dele (`apps/api/.claude/mod16`, com o `config/master.key` copiado, como diz o plano do api); o container monta `apps/api` em `/rails`. Da raiz do monorepo:

```bash
# api do módulo 16 ainda na worktree dele:
docker compose exec -T -w /rails/.claude/mod16 api bin/rails runner 'puts Maintenance::Schema.to_definition' \
  > apps/maintenance/.claude/mod16/schema.graphql
# (se o api do módulo 16 já estiver mergeado em main, tire o `-w /rails/.claude/mod16`)

grep -n -E "^type CityFeature|^type SetCityFeaturePayload|  features: \[CityFeature!\]!|  recordMode: String!|  ibgeCode: String|  setCityFeature\(" \
  apps/maintenance/.claude/mod16/schema.graphql
sed -n '/^type CityFeature {/,/^}/p;/^type SetCityFeaturePayload {/,/^}/p' apps/maintenance/.claude/mod16/schema.graphql
```

Expected: as linhas de `CityFeature`, `SetCityFeaturePayload`, `features`, `recordMode`, `setCityFeature(` e **uma só** de `ibgeCode: String` (a de `CityProfile`, que já existia; `City` não ganha `ibgeCode`). Os dois tipos impressos têm exatamente os campos do contrato §3 — `CityFeature { changedAt: ISO8601DateTime, changedBy: String, description: String!, enabled: Boolean!, key: String!, missing: [String!]!, usable: Boolean! }` e `SetCityFeaturePayload { errors: [UserError!]!, feature: CityFeature, ok: Boolean! }` (o SDL ordena os campos por nome).

Se faltar algum, **pare e reporte**: não escreva à mão em arquivo gerado. Se aparecerem com nome ou tipo diferente do contrato §3 (em especial `errors` que não seja `[UserError!]!`, ou um `ibgeCode` novo em `City`), pare e reporte também: o contrato é a fonte, e um dos dois planos precisa mudar antes de seguir.

Depois confira que só o schema mudou:

```bash
cd apps/maintenance/.claude/mod16 && /opt/homebrew/bin/git diff --stat
```

Expected: só `schema.graphql`. Se outras partes do SDL mudarem (o api da fundação partiu de um `origin/main` com algo que o maintenance ainda não tem), reporte as linhas antes de seguir.

- [ ] **Step 2: Escreva o teste que falha**

Crie `src/screens/FeaturesTab.test.tsx`:

```tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";
import { FeaturesTab } from "./FeaturesTab";

afterEach(cleanup);

function reply(status: number, body: unknown) {
  return new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } });
}
function bodyOf(call: unknown[]): { query: string; variables: Record<string, unknown> } {
  return JSON.parse((call[1] as RequestInit).body as string);
}
function calls(fetchMock: ReturnType<typeof vi.fn>, operation: string) {
  return fetchMock.mock.calls.filter((call) => bodyOf(call).query.includes(operation));
}

type Feature = {
  key: string; description: string; enabled: boolean; usable: boolean; missing: string[];
  changedAt: string | null; changedBy: string | null;
};
const LEDI_OFF: Feature = {
  key: "ledi_export", description: "Exportação LEDI para o PEC da cidade", enabled: false, usable: false,
  missing: [ "pec_url_missing", "credential_missing:ledi" ], changedAt: null, changedBy: null
};
const CADSUS_ON: Feature = {
  key: "cadsus_lookup", description: "Consulta ao CADSUS na validação presencial", enabled: true, usable: true,
  missing: [], changedAt: "2026-10-05T13:00:00Z", changedBy: "dev@local"
};

function featuresReply(features: Feature[], extra: Record<string, unknown> = {}) {
  return reply(200, {
    data: { city: { slug: "sp", recordMode: "integrated", profile: { ibgeCode: "3550308" }, features, ...extra } }
  });
}

function renderTab() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  }
  return render(<FeaturesTab slug="sp" />, { wrapper });
}

describe("FeaturesTab", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  afterEach(() => vi.unstubAllGlobals());

  it("lista chave, descrição, estado, o que falta e quem mudou; modo e IBGE só leitura", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ LEDI_OFF, CADSUS_ON ])));
    renderTab();

    expect(await screen.findByText("ledi_export")).not.toBeNull();
    expect(screen.getByText("Exportação LEDI para o PEC da cidade")).not.toBeNull();
    expect(screen.getByText("desligada")).not.toBeNull();
    expect(screen.getByText("endereço do PEC não preenchido; credencial LEDI não cadastrada")).not.toBeNull();
    expect(screen.getByText("nunca alterada")).not.toBeNull();

    expect(screen.getByText("cadsus_lookup")).not.toBeNull();
    expect(screen.getByText("ligada e utilizável")).not.toBeNull();
    expect(screen.getByText("nada")).not.toBeNull();
    expect(screen.getByText(/05\/10\/2026, 10:00 — dev@local/)).not.toBeNull();

    expect(screen.getByText("integrado ao PEC da cidade (integrated)")).not.toBeNull();
    expect(screen.getByText("3550308")).not.toBeNull();
    // Modo e IBGE não têm controle de edição aqui.
    expect(screen.queryByRole("textbox")).toBeNull();
    expect(screen.queryByRole("combobox")).toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("cidade sem IBGE e em modo off: diz o que falta, sem 'null'", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(
      featuresReply([ LEDI_OFF ], { recordMode: "off", profile: { ibgeCode: null } })
    ));
    renderTab();

    expect(await screen.findByText("desligado (off)")).not.toBeNull();
    expect(screen.getByText("não preenchido")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("um clique só não muda nada; o segundo liga, e a tela diz o que o api devolveu", async () => {
    const user = userEvent.setup();
    const turnedOn: Feature = {
      ...LEDI_OFF, enabled: true, changedAt: "2026-10-05T18:00:00Z", changedBy: "dev@local"
    };
    let listed = [ LEDI_OFF, CADSUS_ON ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ turnedOn, CADSUS_ON ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: turnedOn } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();
    await screen.findByText("ledi_export");

    await user.click(screen.getByRole("button", { name: "ligar" }));
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(0);

    await user.click(screen.getByRole("button", { name: "confirmar: ligar ledi_export" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "ledi_export", enabled: true });

    // Ligar sem os pré-requisitos é permitido; a tela não esconde o que falta.
    expect((await screen.findByRole("status")).textContent).toBe(
      "ledi_export ligada, mas ainda não utilizável — falta: endereço do PEC não preenchido; credencial LEDI não cadastrada."
    );
    // A lista foi relida: o estado novo aparece na tabela.
    expect(await screen.findByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(calls(fetchMock, "query CityFeatures")).toHaveLength(2);
  });

  it("desligar manda enabled=false", async () => {
    const user = userEvent.setup();
    const turnedOff: Feature = { ...CADSUS_ON, enabled: false, usable: false };
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: turnedOff } } }));
      }
      return Promise.resolve(featuresReply([ CADSUS_ON ]));
    });
    renderTab();
    await screen.findByText("cadsus_lookup");

    await user.click(screen.getByRole("button", { name: "desligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: desligar cadsus_lookup" }));

    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables.enabled).toBe(false);
    expect((await screen.findByRole("status")).textContent).toBe("cadsus_lookup desligada.");
  });

  it("erro de regra (unknown_feature) aparece traduzido e não relê a lista", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        return Promise.resolve(reply(200, {
          data: { setCityFeature: { ok: false, errors: [ { path: "key", message: "unknown_feature" } ], feature: null } }
        }));
      }
      return Promise.resolve(featuresReply([ LEDI_OFF ]));
    });
    renderTab();
    await screen.findByText("ledi_export");

    await user.click(screen.getByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar ledi_export" }));

    expect((await screen.findByRole("alert")).textContent).toBe("funcionalidade desconhecida pelo api");
    expect(screen.queryByRole("status")).toBeNull();
    expect(calls(fetchMock, "query CityFeatures")).toHaveLength(1);
  });

  it("recusa do envelope (payload nulo) mostra código e motivo", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        return Promise.resolve(reply(200, {
          data: { setCityFeature: null },
          errors: [ {
            message: "cidade fora do escopo do token", path: [ "setCityFeature" ],
            extensions: { code: "CITY_OUT_OF_SCOPE" }
          } ]
        }));
      }
      return Promise.resolve(featuresReply([ LEDI_OFF ]));
    });
    renderTab();
    await screen.findByText("ledi_export");

    await user.click(screen.getByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar ledi_export" }));

    expect((await screen.findByRole("alert")).textContent).toBe("CITY_OUT_OF_SCOPE — cidade fora do escopo do token");
  });

  it("cidade inalcançável: features degrada, o IBGE diz indisponível e o liga/desliga continua", async () => {
    const user = userEvent.setup();
    const unreachable: Feature = { ...LEDI_OFF, usable: false, missing: [ "city_unreachable" ] };
    const turnedOn: Feature = { ...unreachable, enabled: true, changedAt: "2026-10-05T18:00:00Z", changedBy: "dev@local" };
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: turnedOn } } }));
      }
      return Promise.resolve(reply(200, {
        data: { city: { slug: "sp", recordMode: "record", profile: null, features: [ unreachable ] } },
        errors: [ {
          message: "banco da cidade inacessível", path: [ "city", "profile" ],
          extensions: { code: "CITY_UNREACHABLE" }
        } ]
      }));
    });
    renderTab();

    expect(await screen.findByText("banco da cidade inalcançável — não deu para conferir")).not.toBeNull();
    expect(screen.getByText("indisponível (CITY_UNREACHABLE)")).not.toBeNull();
    expect(screen.getByText("prontuário no Rota Saúde (record)")).not.toBeNull();
    // "não preenchido" afirmaria que a cidade não tem IBGE — e não se sabe.
    expect(screen.queryByText("não preenchido")).toBeNull();

    await user.click(screen.getByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar ledi_export" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect((await screen.findByRole("status")).textContent).toBe(
      "ledi_export ligada, mas ainda não utilizável — falta: banco da cidade inalcançável — não deu para conferir."
    );
  });

  it("erro de campo em features (cidade arquivada) mostra código e mensagem, sem botões", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(reply(200, {
      data: { city: null },
      errors: [ {
        message: "cidade arquivada", path: [ "city", "features" ],
        extensions: { code: "CITY_ARCHIVED" }
      } ]
    })));
    renderTab();

    expect((await screen.findByRole("alert")).textContent).toBe("CITY_ARCHIVED — cidade arquivada");
    expect(screen.queryByRole("button", { name: "ligar" })).toBeNull();
  });

  it("api antiga sem os campos: a aba explica a ordem de deploy", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(reply(200, {
      errors: [ {
        message: "Field 'recordMode' doesn't exist on type 'City'",
        extensions: { code: "undefinedField", typeName: "City", fieldName: "recordMode" }
      } ]
    })));
    renderTab();

    expect((await screen.findByRole("alert")).textContent)
      .toMatch(/o api do módulo 16 precisa subir antes do maintenance/);
  });
});
```

- [ ] **Step 3: Rode e confirme que falha**

Run: `cd apps/maintenance/.claude/mod16 && npx vitest run src/screens/FeaturesTab.test.tsx`
Expected: FAIL — `Failed to resolve import "./FeaturesTab" from "src/screens/FeaturesTab.test.tsx"`.

- [ ] **Step 4: Implemente a aba**

Crie `src/screens/FeaturesTab.tsx`:

```tsx
import { useState, type CSSProperties } from "react";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { gql } from "../lib/api";
import { graphql } from "../gql";
import { GraphQLRefusal } from "../lib/errors";
import { fmtWhen } from "../lib/analytics";
import {
  doneText, featureStatus, ibgeCodeText, missingSummary, recordModeText, setFeatureErrorText
} from "../lib/features";
import { Panel } from "../components/Panel";
import { Button } from "../components/Button";
import { ConfirmButton } from "../components/ConfirmButton";
import { Tag } from "../components/Tag";
import { DataTable } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { ErrorState } from "../components/ErrorState";

// Aba Funcionalidades (módulo 16, F-16.1; ADR 0028; contratos §3): os
// interruptores da cidade e o liga/desliga. Só o maintenance escreve
// interruptor. Modo de prontuário e código IBGE são do operador (console
// admin) e aqui são só leitura; o IBGE é o do perfil da cidade
// (`profile.ibgeCode`, fonte única — contratos §3).
//
// Consulta PRÓPRIA, fora do CityHeader: contra um api sem o módulo 16, a
// validação recusa o documento inteiro, e só esta aba pode cair. Cidade
// inalcançável NÃO derruba a aba: `features` degrada (`usable: false`,
// `missing: ["city_unreachable"]`) e o liga/desliga continua; só `profile`
// (anulável) vem nulo com o erro no caminho ["city", "profile"].
const CityFeaturesQuery = graphql(`
  query CityFeatures($slug: String!) {
    city(slug: $slug) {
      slug
      recordMode
      profile { ibgeCode }
      features { key description enabled usable missing changedAt changedBy }
    }
  }
`);

const SetCityFeatureMutation = graphql(`
  mutation SetCityFeature($citySlug: String!, $key: String!, $enabled: Boolean!) {
    setCityFeature(citySlug: $citySlug, key: $key, enabled: $enabled) {
      ok
      errors { path message }
      feature { key description enabled usable missing changedAt changedBy }
    }
  }
`);

const dlStyle: CSSProperties = {
  display: "grid",
  gridTemplateColumns: "max-content 1fr",
  columnGap: 12,
  rowGap: 6,
  margin: 0,
  fontSize: 12.5
};

// Códigos da validação do graphql-ruby que um api sem o módulo 16 devolve.
const OLD_API_CODES = new Set([ "undefinedField" ]);

function requestErrorText(err: unknown): string {
  if (err instanceof GraphQLRefusal && OLD_API_CODES.has(err.code)) {
    return "esta API ainda não tem os interruptores — o api do módulo 16 precisa subir antes do maintenance";
  }
  return err instanceof Error ? err.message : "erro inesperado";
}

function fieldError(fieldErrors: GraphQLRefusal[], field: string): GraphQLRefusal | undefined {
  return fieldErrors.find((refusal) => {
    const path = refusal.path ?? [];
    return path[0] === "city" && (path.length === 1 || path[1] === field);
  });
}

type Feature = {
  key: string; description: string; enabled: boolean; usable: boolean; missing: string[];
  changedAt?: string | null; changedBy?: string | null;
};

export function FeaturesTab({ slug }: { slug: string }) {
  const queryClient = useQueryClient();
  const queryKey = [ "city", slug, "features" ];
  const query = useQuery({ queryKey, queryFn: () => gql(CityFeaturesQuery, { slug }), staleTime: Infinity });

  const [ done, setDone ] = useState<string | null>(null);
  const [ failure, setFailure ] = useState<string | null>(null);

  const mutation = useMutation({
    mutationFn: (vars: { key: string; enabled: boolean }) =>
      gql(SetCityFeatureMutation, { citySlug: slug, key: vars.key, enabled: vars.enabled }),
    onMutate: () => { setDone(null); setFailure(null); },
    onSuccess: ({ data, fieldErrors }) => {
      const payload = data?.setCityFeature ?? null;
      if (payload === null) {
        // Recusa do envelope (CITY_OUT_OF_SCOPE, CITY_BUDGET_EXCEEDED…):
        // `data.setCityFeature` nulo e o motivo em `errors`.
        const refusal = fieldErrors[0];
        setFailure(refusal ? `${refusal.code} — ${refusal.message}` : setFeatureErrorText([]));
        return;
      }
      if (!payload.ok) {
        setFailure(setFeatureErrorText(payload.errors));
        return;
      }
      if (payload.feature) setDone(doneText(payload.feature));
      void queryClient.invalidateQueries({ queryKey });
    },
    onError: (err) => setFailure(err instanceof Error ? err.message : "erro inesperado")
  });

  const city = query.data?.data?.city ?? null;
  const err = query.data ? fieldError(query.data.fieldErrors, "features") : undefined;
  const profileErr = query.data ? fieldError(query.data.fieldErrors, "profile") : undefined;
  const ibge = profileErr ? `indisponível (${profileErr.code})` : ibgeCodeText(city?.profile?.ibgeCode);

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
      <div>
        <Button onClick={() => void query.refetch()} busy={query.isFetching}>atualizar</Button>
      </div>

      {query.isPending && <p>carregando…</p>}
      {query.isError && <ErrorState message={requestErrorText(query.error)} />}
      {err && <ErrorState message={`${err.code} — ${err.message}`} />}

      {city && (
        <>
          <Panel title="Prontuário e exportação">
            <dl style={dlStyle}>
              <dt>Modo de prontuário</dt><dd>{recordModeText(city.recordMode)}</dd>
              <dt>Código IBGE</dt><dd>{ibge}</dd>
            </dl>
            <p style={{ margin: 0, fontSize: 11.5, color: "var(--ink3)" }}>
              Modo, código IBGE e endereço do PEC são do operador, no console de plataforma (admin).
            </p>
          </Panel>

          <Panel title="Funcionalidades">
            {done && <p role="status" style={{ margin: 0, fontSize: 12.5, color: "var(--ok)" }}>{done}</p>}
            {failure && <ErrorState message={failure} />}
            {city.features.length === 0 ? <EmptyState message="nenhuma funcionalidade no catálogo" /> : (
              <DataTable<Feature>
                columns={[
                  { key: "key", label: "Chave" },
                  { key: "description", label: "Descrição" },
                  {
                    key: "state", label: "Estado",
                    render: (f) => {
                      const s = featureStatus(f);
                      return <Tag tone={s.tone}>{s.label}</Tag>;
                    }
                  },
                  { key: "missing", label: "O que falta", render: (f) => missingSummary(f.missing) },
                  {
                    key: "changed", label: "Última mudança",
                    render: (f) => f.changedAt ? `${fmtWhen(f.changedAt)} — ${f.changedBy ?? "—"}` : "nunca alterada"
                  },
                  {
                    key: "action", label: "",
                    render: (f) => (
                      <ConfirmButton
                        label={f.enabled ? "desligar" : "ligar"}
                        confirmLabel={f.enabled ? `confirmar: desligar ${f.key}` : `confirmar: ligar ${f.key}`}
                        busy={mutation.isPending && mutation.variables?.key === f.key}
                        disabled={mutation.isPending}
                        onConfirm={() => mutation.mutate({ key: f.key, enabled: !f.enabled })}
                      />
                    )
                  }
                ]}
                rows={city.features}
                rowKey={(f) => f.key}
              />
            )}
          </Panel>
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 5: Gere os tipos**

Run: `cd apps/maintenance/.claude/mod16 && npm run codegen`
Expected: `[SUCCESS] Generate outputs`; `src/gql/gql.ts` e `src/gql/graphql.ts` ganham `CityFeaturesQuery` e `SetCityFeatureMutation`.

Sem este passo, o `tsc` acusa `Argument of type 'unknown' is not assignable to parameter of type 'TypedDocumentNode<…>'` nas duas chamadas de `gql`: o `graphql()` só conhece um documento depois do codegen.

- [ ] **Step 6: Rode e confirme que passa**

Run: `cd apps/maintenance/.claude/mod16 && npx vitest run src/screens/FeaturesTab.test.tsx && npx tsc --noEmit`
Expected: PASS, 9 testes; `tsc` sem saída.

- [ ] **Step 7: Commit**

```bash
cd apps/maintenance/.claude/mod16
/opt/homebrew/bin/git add schema.graphql src/gql/gql.ts src/gql/graphql.ts src/gql/index.ts \
  src/screens/FeaturesTab.tsx src/screens/FeaturesTab.test.tsx
/opt/homebrew/bin/git status --short
/opt/homebrew/bin/git commit -m "feat: add the feature switches tab for the maintenance city detail

Lists the city's feature switches with what each one still misses, and
turns them on or off with setCityFeature behind a two-click confirmation.
Record mode and the city profile IBGE code are read-only here.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Expected: `status --short` antes do commit sem nada fora da lista (se `src/gql/index.ts` não mudou, o `git add` dele é inofensivo).

---

### Task 3: A aba no detalhe da cidade e o README

**Files:**
- Modify: `src/screens/CityDetail.tsx` (import, comentário do topo, `TabKey`, `TABS`, montagem da aba)
- Modify: `src/screens/CityDetail.test.tsx` (listas de operações do primeiro teste; caso novo de api antiga)
- Modify: `README.md`

**Interfaces:**
- Consumes: `FeaturesTab({ slug })` e a operação `query CityFeatures` (Task 2).
- Produces: aba `"features"` com rótulo **"Funcionalidades"** no detalhe da cidade.

- [ ] **Step 1: Escreva o teste que falha**

Em `src/screens/CityDetail.test.tsx`, nas **duas** listas de operações do primeiro teste ("carrega só o topo…"), troque

```ts
"CityAnalyticsStatus", "CityAnalyticsIndicators" ]) {
```

por

```ts
"CityAnalyticsStatus", "CityAnalyticsIndicators", "CityFeatures" ]) {
```

(duas ocorrências; use substituição de todas). E insira, logo antes do caso `it("'atualizar' numa aba refaz só a consulta daquela aba", async () => {`:

```tsx
  it("api antiga: só a aba Funcionalidades falha; o topo e as outras abas seguem", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation((_url, init) => {
      const body = bodyOf([ _url, init ]);
      if (body.query.includes("query CityHeader")) return Promise.resolve(HEADER_REPLY.clone());
      if (body.query.includes("query CityFeatures")) {
        return Promise.resolve(reply(200, {
          errors: [ { message: "Field 'recordMode' doesn't exist on type 'City'", extensions: { code: "undefinedField" } } ]
        }));
      }
      if (body.query.includes("query CityProfile")) {
        return Promise.resolve(
          reply(200, { data: { city: { slug: "sp", consentTermVersion: "v3", profile: { name: "São Paulo", uf: "SP", ibgeCode: "3550308" } } } })
        );
      }
      return Promise.resolve(reply(200, { data: { city: { slug: "sp" } } }));
    });

    renderDetail();
    await screen.findByText("São Paulo");

    // O topo nunca pede interruptor, modo nem IBGE de plataforma.
    expect(bodyOf(operationCalls(fetchMock, "CityHeader")[0]).query).not.toMatch(/features|recordMode|ibgeCode/);

    await user.click(screen.getByRole("button", { name: "Funcionalidades" }));
    expect((await screen.findByRole("alert")).textContent)
      .toMatch(/o api do módulo 16 precisa subir antes do maintenance/);
    expect(operationCalls(fetchMock, "CityFeatures")).toHaveLength(1);

    // O topo continua, e a consulta dele não se repetiu.
    expect(screen.getByText("+55 11 90000-0000")).not.toBeNull();
    expect(operationCalls(fetchMock, "CityHeader")).toHaveLength(1);

    await user.click(screen.getByRole("button", { name: "Perfil" }));
    expect(await screen.findByText("3550308")).not.toBeNull();
    expect(screen.queryByRole("alert")).toBeNull();
  });
```

- [ ] **Step 2: Rode e confirme que falha**

Run: `cd apps/maintenance/.claude/mod16 && npx vitest run src/screens/CityDetail.test.tsx`
Expected: FAIL só no caso novo — `Unable to find an accessible element with the role "button" and name "Funcionalidades"`.

- [ ] **Step 3: Monte a aba**

Em `src/screens/CityDetail.tsx`:

1. Troque

```tsx
import { AnalyticsTab } from "./AnalyticsTab";
```

por

```tsx
import { AnalyticsTab } from "./AnalyticsTab";
import { FeaturesTab } from "./FeaturesTab";
```

2. No comentário do topo, troque

```tsx
// topo. Por isso nenhum campo de Analytics entra no CityHeader.
```

por

```tsx
// topo. Por isso nenhum campo de Analytics entra no CityHeader. A aba
// Funcionalidades (módulo 16) segue o mesmo molde (`FeaturesTab`): nenhum
// campo de interruptor, modo ou IBGE de plataforma entra no CityHeader.
```

3. Troque

```tsx
type TabKey = "profile" | "protocols" | "alertRecipients" | "accounts" | "counts" | "operations" | "analytics";
```

por

```tsx
type TabKey = "profile" | "protocols" | "alertRecipients" | "accounts" | "counts" | "operations" | "analytics" | "features";
```

4. Troque

```tsx
  { key: "analytics", label: "Analytics" }
];
```

por

```tsx
  { key: "analytics", label: "Analytics" },
  { key: "features", label: "Funcionalidades" }
];
```

5. Troque

```tsx
          {activeTab === "analytics" && <AnalyticsTab slug={slug} />}
```

por

```tsx
          {activeTab === "analytics" && <AnalyticsTab slug={slug} />}

          {activeTab === "features" && <FeaturesTab slug={slug} />}
```

- [ ] **Step 4: Rode e confirme que passa**

Run: `cd apps/maintenance/.claude/mod16 && npx vitest run src/screens/CityDetail.test.tsx && npx tsc --noEmit`
Expected: PASS (todos os casos do arquivo); `tsc` sem saída.

- [ ] **Step 5: README**

Em `README.md`:

1. Na tabela de telas, troque

```markdown
Contas, Contagens, Operação e Analytics. Cada aba
```

por

```markdown
Contas, Contagens, Operação, Analytics e Funcionalidades. Cada aba
```

2. Insira, logo antes da linha `## Subir em dev`:

```markdown
### Funcionalidades

A aba **Funcionalidades** do detalhe de cidade (módulo 16, ADR 0028) lista os
interruptores de funcionalidade da cidade (`features`): chave, descrição,
estado (desligada; ligada e utilizável; ligada, falta pré-requisito), o que
falta e a última mudança (quando e por qual mantenedor). Ligar e desligar
(`setCityFeature`) pede dois cliques: o primeiro só troca o rótulo do botão
para "confirmar: …". Ligar não exige os pré-requisitos: a funcionalidade fica
ligada, e a cidade só a usa quando o que falta for resolvido (credencial no
dashboard; modo, código IBGE e endereço do PEC no console admin). Depois do
ato, a tela diz o estado que o api devolveu.

O modo de prontuário (`recordMode`) e o código IBGE (`profile.ibgeCode`, o
mesmo da aba Perfil) aparecem só para leitura: quem os muda é o operador, no
admin. Com o banco da cidade inalcançável, a aba continua: o que falta aparece
como "banco da cidade inalcançável", o IBGE como "indisponível", e o
liga/desliga segue funcionando.

> **Ordem de deploy:** o `api` sobe **antes** do maintenance. Contra um api
> sem os campos do módulo 16, a validação recusa a consulta da aba
> Funcionalidades, e só ela mostra o aviso; o topo e as outras abas seguem.
```

- [ ] **Step 6: Commit**

```bash
cd apps/maintenance/.claude/mod16
/opt/homebrew/bin/git add src/screens/CityDetail.tsx src/screens/CityDetail.test.tsx README.md
/opt/homebrew/bin/git status --short
/opt/homebrew/bin/git commit -m "feat: add the feature switches tab to the maintenance city detail

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–3 na branch `feat/mod-16-record-mode`; o api da fundação rodando com a semente.
- Produces: branch pronta para merge (depois do merge do api).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `cd apps/maintenance/.claude/mod16 && npx vitest run && npx tsc --noEmit && npm run codegen:check && npm run build`
Expected: tudo verde (a CI, `.github/workflows/ci.yml`, roda os quatro). Na base `baf3219` são 19 arquivos de teste; com este plano, **21 arquivos** (`features.test.ts` e `FeaturesTab.test.tsx` novos). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd apps/maintenance/.claude/mod16 && /opt/homebrew/bin/git status --short
```

Expected: vazio. O symlink `node_modules` não aparece porque é ignorado. Se aparecer, **não** o adicione.

- [ ] **Step 2: O topo não pede campo novo**

```bash
cd apps/maintenance/.claude/mod16 && sed -n '/const CityHeaderQuery/,/`);/p' src/screens/CityDetail.tsx | grep -n -E "features|recordMode|ibgeCode" ; echo "exit $?"
```

Expected: nenhuma linha e `exit 1` (o `grep` não achou nada dentro do `CityHeaderQuery`; o IBGE desta aba vem de `profile`, que o topo também não pede).

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§3.1, §8, §9), o ADR 0028 e os contratos §2–§3. Pontos de atenção:
- nenhum campo novo dentro do `CityHeader` nem de outra consulta existente;
- `recordMode` e `profile.ibgeCode` sem controle de edição; nenhum `ibgeCode` lido de outro lugar;
- cidade inalcançável: `city_unreachable` com frase própria, IBGE "indisponível", liga/desliga disponível;
- liga/desliga só no segundo clique; `enabled` enviado é o inverso do estado lido;
- depois do ato, a frase vem do `feature` **devolvido** (nunca do pedido), e a lista é relida;
- "ligada" × "utilizável" nunca se confundem; o que falta aparece mesmo com a funcionalidade desligada;
- `schema.graphql` e `src/gql/*` iguais ao que o api em execução produz (`codegen:check` verde);
- nenhum `git add -A` no histórico (`git log --stat origin/main..HEAD` sem `node_modules`).

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api da fundação rodando com a semente, na porta que o plano do api definir (no módulo 11 foi `:3031`; o padrão do proxy é `:3030`). O proxy do Vite troca o Host para `maintenance-api.localhost` e injeta o Origin `http://maintenance.localhost:5177`; por isso o Vite do worktree sobe **na 5177** (pare antes o serviço `maintenance` do compose, que serve o checkout principal). Da raiz do monorepo:

```bash
docker compose stop maintenance
cd apps/maintenance/.claude/mod16 && VITE_API_PROXY_TARGET=http://localhost:<porta do api da fundação> npx vite --port 5177 --host 0.0.0.0
```

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor; não digite senha nem TOTP. Confira com screenshot:
- Cidades → Curitiba → aba **Funcionalidades**: o quadro "Prontuário e exportação" com o modo e o código IBGE (o mesmo da aba Perfil), e a tabela com `ledi_export` e `cadsus_lookup`, cada uma com a descrição do catálogo, o estado e o que falta;
- "ligar" em `ledi_export` → o botão vira "confirmar: ligar ledi_export"; o segundo clique liga; a frase diz se ficou utilizável ou o que falta; a coluna "Última mudança" mostra a hora e `dev@local`;
- "desligar" → "confirmar: desligar ledi_export" → volta a "desligada";
- a trilha da plataforma (tela **Auditoria**) mostra as duas mudanças;
- as outras abas continuam abrindo.

Depois: `docker compose start maintenance`. Deixe os interruptores da cidade de dev como estavam antes da prova.

- [ ] **Step 5:** **Pare.** O merge do maintenance só vem depois do merge do api da fundação, e só com autorização explícita do usuário (merge, push e fechamento de card, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance: publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-16-record-mode
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove .claude/mod16
/opt/homebrew/bin/git branch -d feat/mod-16-record-mode
```

Na produção, publicar o maintenance novo **antes** do api novo não derruba a ficha, só a aba Funcionalidades, e ela diz o motivo. O maintenance não roda em produção (spec §1), então a ordem vale para staging.

---

## Incorporado ao contrato

Três divergências levantadas ao escrever este plano foram resolvidas no contrato (`…-contracts.md` §3) e já estão no código das Tasks 1–3:

1. **`setCityFeature.errors` é `[UserError!]!`** (`{ path, message }`), o padrão de toda mutation da API de manutenção (`BaseMutation#audited`). A mutation consulta `errors { path message }`; `setFeatureErrorText` traduz a mensagem quando ela vem como `unknown_city`/`unknown_feature` e mostra a do api nos demais casos.
2. **Código IBGE com fonte única: `City.profile.ibgeCode`** (banco da cidade, o mesmo da aba Perfil). Não nasce `City.ibgeCode`. A aba lê `profile { ibgeCode }`; perfil ilegível aparece "indisponível (<código>)".
3. **Cidade inalcançável não derruba a aba.** `usable`/`missing` degradam para `false`/`["city_unreachable"]`; `enabled`, `changedAt`, `changedBy` e o liga/desliga seguem da plataforma. `missingText("city_unreachable")` tem frase própria.

Nenhuma divergência em aberto.

## Self-review

- **Cobertura da spec:** §3.1 (mutation `setCityFeature`, "ligada" × "utilizável", `missing` para a tela, inclusive a degradação `city_unreachable` do contrato §3) → Tasks 1 e 2; §8 "Front: maintenance (interruptores)" → Tasks 2 e 3; §8 "Prova no navegador: ligar interruptor no maintenance" → Task 4, Step 4; §9 ordem de deploy (maintenance depois do api) → README (Task 3), consulta própria (Tasks 2 e 3) e Task 4, Step 5; §10 F-16.1 (parte do maintenance) → Tasks 1–3; contrato §3 (`recordMode` e `profile.ibgeCode` só leitura; `UserError` com `unknown_city`/`unknown_feature`) → Tasks 1 e 2. A auditoria `city.feature_changed` é do api; a prova a confere na tela Auditoria (Task 4).
- **Placeholders:** nenhum; todo passo de código tem o código. A única variável é a porta do api da fundação no Step 4 da Task 4, que o plano do api define.
- **Tipos:** `FeatureView`/`doneText` (Task 1) aceitam o `CityFeature` gerado (`enabled`, `usable`, `missing`, `key`); `recordModeText`/`ibgeCodeText` aceitam `string | null | undefined` (o `profile.ibgeCode` gerado é `Maybe<string>`); `setFeatureErrorText(UserErrorView[])` recebe o `errors: Array<UserError>` gerado (`path?: Maybe<string>`, `message: string`); `FeaturesTab({ slug })` é o que a Task 3 monta; os nomes `CityFeatures` e `SetCityFeature` são os mesmos nos testes das Tasks 2 e 3.
- **Review Focus:** as cinco linhas têm teste citado pelo nome nas Tasks 1, 2 e 3.
- **Código conferido:** o código das Tasks 1–3 foi aplicado numa cópia descartável de `origin/main` do maintenance (`baf3219`) em 2026-10-05, com o `schema.graphql` editado **só nessa cópia** conforme o contrato §3 (o api da fundação ainda não existe): codegen, `tsc --noEmit`, vitest 21 arquivos / 188 testes, `codegen:check` e `npm run build`, tudo verde. Na execução real, o schema vem do api (Task 2, Step 1).
